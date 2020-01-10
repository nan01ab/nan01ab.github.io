---
layout: page
title: Some Optimizations of Linux Network Stack
tags: [Operating System, Network]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Some Optimizations of Linux Network Stack

### 0x00 SO_REUSEPORT

 SO_REUSEPORT是Linux网络栈中一个很有效果，有应用得很广泛的一个应用，而且能够达成的效果非常好。SO_REUSEPORT基本的思路是改变在一个端口只能被一个端口监听(TCP)/recvfrom(UDP)的情况，提高性能。之前有个不少的研究就是为了改进Linux网络栈的Accept等的性能。mTCP是一个用户态TCP协议栈的实现，在Linux使用了SO_REUSEPORT之后，两者的性能差距明显缩小。Linux中相关的代码一部分如下，基本的思路是在使用了SO_REUSEPORT的选择之后，原来端口对应到的Socket会添加一个sk_reuseport_cb的字段，这个字段中保存一个socket的array，在处理通信的时候，根据源地址端口和目的地址端口计算 or 使用出一个hash值，来决定这个array那一个socket来实际处理。这个除了使用hash之外，还可以自己使用bpf来控制(通过这个功能想到一个有意思的玩法，有时间试试看)，

```c
// /linux/net/core/sock_reuseport.c
253  /**
254	 *  reuseport_select_sock - Select a socket from an SO_REUSEPORT group.
255	 *  @sk: First socket in the group.
256	 *  @hash: When no BPF filter is available, use this hash to select.
257	 *  @skb: skb to run through BPF filter.
258	 *  @hdr_len: BPF filter expects skb data pointer at payload data.  If
259	 *    the skb does not yet point at the payload, this parameter represents
260	 *    how far the pointer needs to advance to reach the payload.
261	 *  Returns a socket that should receive the packet (or NULL on error).
262	 */
263	struct sock *reuseport_select_sock(struct sock *sk,
264					   u32 hash,
265					   struct sk_buff *skb,
266					   int hdr_len)
267	{
268		struct sock_reuseport *reuse;
269		struct bpf_prog *prog;
270		struct sock *sk2 = NULL;
271		u16 socks;
272	
273		rcu_read_lock();
274		reuse = rcu_dereference(sk->sk_reuseport_cb);
275	
276		/* if memory allocation failed or add call is not yet complete */
277		if (!reuse)
278			goto out;
279	
280		prog = rcu_dereference(reuse->prog);
281		socks = READ_ONCE(reuse->num_socks);
282		if (likely(socks)) {
283			/* paired with smp_wmb() in reuseport_add_sock() */
284			smp_rmb();
285	
286			if (!prog || !skb)
287				goto select_by_hash;
288	
289			if (prog->type == BPF_PROG_TYPE_SK_REUSEPORT)
290				sk2 = bpf_run_sk_reuseport(reuse, sk, prog, skb, hash);
291			else
292				sk2 = run_bpf_filter(reuse, socks, prog, skb, hdr_len);
293	
294	select_by_hash:
295			/* no bpf or invalid bpf result: fall back to hash usage */
296			if (!sk2)
297				sk2 = reuse->socks[reciprocal_scale(hash, socks)];
298		}
299	
300	out:
301		rcu_read_unlock();
302		return sk2;
303	}
```

sock_reuseport结果并不复杂，就是为何来socket数据的一些信息已经可选的一个bpf_prog。在IPv4-UDP的udp4_lib_lookup2处理函数中，/linux/net/ipv4/udp.c中的逻辑就是在设置了sk_reuseport，之后时间处理的会是sk_reuseport中sockets中选出来的一个，这样看上去原来的socket就类似一个proxy的功能。这样看的话就类似于和直接协议proxy查询将数据转发到后面的几个backend类似，只不过这里直接将其放到网络栈中，这样是不是很多在现在负载均衡中使用的一些方式可以用在这里，比如一致性hash?。这里的hash值的每次启动会初始化一个随机数，提高安全性。在TCP Listen处理的时候，基本的逻辑是一样的。

```c
// linux/include/net/sock_reuseport.h
13	struct sock_reuseport {
14		struct rcu_head		rcu;
15	
16		u16			max_socks;	/* length of socks */
17		u16			num_socks;	/* elements in socks */
18		/* The last synq overflow event timestamp of this
19		 * reuse->socks[] group.
20		 */
21		unsigned int		synq_overflow_ts;
22		/* ID stays the same even after the size of socks[] grows. */
23		unsigned int		reuseport_id;
24		bool			bind_inany;
25		struct bpf_prog __rcu	*prog;		/* optional BPF sock selector */
26		struct sock		*socks[0];	/* array of sock pointers */
27	};

// /linux/net/ipv4/udp.c
442     if (sk->sk_reuseport) {
443					hash = udp_ehashfn(net, daddr, hnum,
444							   saddr, sport);
445					result = reuseport_select_sock(sk, hash, skb,
446								sizeof(struct udphdr));
447					if (result)
448						return result;
449				}

// /linux/net/ipv4/udp.c
412	static u32 udp_ehashfn(const struct net *net, const __be32 laddr,
413			       const __u16 lport, const __be32 faddr,
414			       const __be16 fport)
415	{
416		static u32 udp_ehash_secret __read_mostly;
417	
418		net_get_random_once(&udp_ehash_secret, sizeof(udp_ehash_secret));
419	
420		return __inet_ehashfn(laddr, lport, faddr, fport,
421				      udp_ehash_secret + net_hash_mix(net));
422	}
```

 使用了SO_REUSEPORT的效果可以查看网上的一些信息。

### 0x01 Lockless Listener

  Lockless Listener/无锁监听是Linux TCP协议实现中一个很有意思的优化。这个优化要比这里总结的其它优化方式要复杂不少，但是这个非常值得一看。Linux中TCP相关的socket会保存到一个叫做inet_hashinfo的结构结构中，这个结构的定义如下，ehash中保存了已经完成握手的socket，bhash中保存bind之后的socket，而listening_hash中保存的是处于监听状态的socket，另外一个lhash2是最近的Linux版本添加的用于加速listening查找的一个优化。以基于IPv4的TCP为例，接受到一个数据包之后，Kernel中会进入到一个tcp_v4_rcv的函数来处理，这个函数通过__inet_lookup_skb来查找对应的socket，这个函数的基本逻辑是先查询ehash，在没有结果的情况下查询listening_hash。接下来的逻辑中，kernel会根据这个socket的类型来处理。如果socket处于监听的状态，这个数据包也是一个合法的数据包，则转入处理握手的逻辑，如果这个是一个普通的socket，则转入普通的数据包处理的逻辑。而这里处理过程都会多这个socket中的一个锁进行上锁操作。这个优化就涉及到监听的情况下，消除这个加锁的操作。

```c
struct inet_hashinfo {
	/* This is for sockets with full identity only.  Sockets here will
	 * always be without wildcards and will have the following invariant:
	 *          TCP_ESTABLISHED <= sk->sk_state < TCP_CLOSE
	 */
	struct inet_ehash_bucket	*ehash;
	spinlock_t			*ehash_locks;
	unsigned int			ehash_mask;
	unsigned int			ehash_locks_mask;
	/* Ok, let's try this, I give up, we do need a local binding
	 * TCP hash as well as the others for fast bind/connect.
	 */
	struct kmem_cache		*bind_bucket_cachep;
	struct inet_bind_hashbucket	*bhash;
	unsigned int			bhash_size;
  
	/* The 2nd listener table hashed by local port and address */
	unsigned int			lhash2_mask;
	struct inet_listen_hashbucket	*lhash2;
	/* All the above members are written once at bootup and
	 * never written again _or_ are predominantly read-access.
	 *
	 * Now align to a new cache line as all the following members
	 * might be often dirty.
	 */
	/* All sockets in TCP_LISTEN state will be in listening_hash.
	 * This is the only table where wildcard'd TCP sockets can
	 * exist.  listening_hash is only hashed by local port number.
	 * If lhash2 is initialized, the same socket will also be hashed
	 * to lhash2 by port and address.
	 */
	struct inet_listen_hashbucket	listening_hash[INET_LHTABLE_SIZE] ____cacheline_aligned_in_smp;
};
```

 这里原来的kernel实现会有涉及到两个队列，很多时候就称之为半连接队列(syns queue)和连接队列(accept queue)。前者是那些收到第一个SYN包回复了一个SYN+ACK包的没有完全建立的连接；后者是三次握手完成已经建立的连接，这些连接之后就可以被accept操作处理。这个优化之前的版本中会有一个listen sock结果来表示listen socket的一些信息，而在现在的版本中这个结构已经没有了。现在的处理逻辑是在第一个SYN处理完成之后，设置sock结构的状态为TCP_NEW_SYN_RECV，并将这个sock结构添加到全局的ehash table中。这个sock结构同时会保存志向其listen sock结构的指针。现在Linux TCP中实现的一个TCP状态的定义如下，

```c
// /include/net/request_sock.h
/** struct listen_sock - listen state
 *
 * @max_qlen_log - log_2 of maximal queued SYNs/REQUESTs
 */
struct listen_sock {
	u8			max_qlen_log;
	u8			synflood_warned;
	/* 2 bytes hole, try to use */
	int			qlen;
	int			qlen_young;
	int			clock_hand;
	u32			hash_rnd;
	u32			nr_table_entries;
	struct request_sock	*syn_table[0];
};

// /include/net/tcp_states.h
enum {
	TCP_ESTABLISHED = 1,
	TCP_SYN_SENT,
	TCP_SYN_RECV,
	TCP_FIN_WAIT1,
	TCP_FIN_WAIT2,
	TCP_TIME_WAIT,
	TCP_CLOSE,
	TCP_CLOSE_WAIT,
	TCP_LAST_ACK,
	TCP_LISTEN,
	TCP_CLOSING,	/* Now a valid state */
	TCP_NEW_SYN_RECV,
	TCP_MAX_STATES	/* Leave at the end! */
};
```

 在接受到三次握手的地三个包的时候，从全局的ehash table中取出这个半连接状态的socket。然后通过其构造完成握手的socket，添加到其listen socket的accept queue中，完成握手的全部过程。这样这里的处理过程就不需要对listen socke加其中的一个锁了。实现这里的话，加上syncookies，fastopen等一些优化，使得这里的代码十分复杂，判断的分支很多。

### 0x02 SO_INCOMING_CPU

 从之前的一些研究的Paper中的优化策略来看，将一个socket的操作都放在痛一个CPU核心之上可以获得性能提升。Linux中也支持了这个操作，从[1]中的Patch来看，在个优化主要就是早setsocket上面添加了一个Flag。以udp的数据处理为例，在后面的数据包查找处理的socket时候，会有一个compute_score函数来计算一个数值，根据这个数据选择最佳的一个处理socket，基本的逻辑如下。如果设置incoming_cpu的CPU与目前运行的CPU核心是同一个，则会增加这个score。

```c
// linux/net/ipv4/udp.c
407 if (sk->sk_incoming_cpu == raw_smp_processor_id())
408			score++;
```

然后在后面的udp4_lib_lookup2函数中处理的时候，查找每个候选的socket的时候，通过一个badness参数达到了选择最大值的效果，在if (sk->sk_incoming_cpu == raw_smp_processor_id())为真的情况下，对于的socket就会有最大的score。从这里的代码看，配合线程设置CPU亲和性和使用前面提到的SO_REUSEPORT会有更好的效果。

```c
// linux/net/ipv4/udp.c
425	static struct sock *udp4_lib_lookup2(struct net *net,
426					     __be32 saddr, __be16 sport,
427					     __be32 daddr, unsigned int hnum,
428					     int dif, int sdif, bool exact_dif,
429					     struct udp_hslot *hslot2,
430					     struct sk_buff *skb)
431	{
432		struct sock *sk, *result;
433		int score, badness;
434		u32 hash = 0;
435	
436		result = NULL;
437		badness = 0;
438		udp_portaddr_for_each_entry_rcu(sk, &hslot2->head) {
439			score = compute_score(sk, net, saddr, sport,
440					      daddr, hnum, dif, sdif, exact_dif);
441			if (score > badness) {
442				if (sk->sk_reuseport) {
443					hash = udp_ehashfn(net, daddr, hnum,
444							   saddr, sport);
445					result = reuseport_select_sock(sk, hash, skb,
446								sizeof(struct udphdr));
447					if (result)
448						return result;
449				}
450				badness = score;
451				result = sk;
452			}
453		}
454		return result;
455	}
```

### 0x03 Zero Copy

 Zero Copy同样也是一些网络栈优化Papper中经常提到的优化手段。这个思路同样被Linux采用，在一些情况下，特别是发送的数据比较大的情况下，会减少很多内存拷贝的时候，提高性能。Linux Zero Copy的特性使用起来比较复杂。同样地，为了使用这个特性，想要setsocket设置一个flag[2]。在send数据的时候同样需要添加一个标记zero copy的flag，MSG_ZEROCOPY。但是只是这样是不够的，网络栈中zero copy优化要处理的一个问题就是buffer重用的问题。这里也一样，如果在数据发送完成之前，应用复用了这个bufer，然后写入了新的数据，这样就可能导致传输了不一致的数据。所以需要后面条件如下的功能的代码，用于查看传输是否已经完成了，

```c
// 设置flag
if (setsockopt(fd, SOL_SOCKET, SO_ZEROCOPY, &one, sizeof(one)))
        error(1, errno, "setsockopt zerocopy");
// 发送数据
ret = send(fd, buf, sizeof(buf), MSG_ZEROCOPY);
// 查询完成情况
pfd.fd = fd;
pfd.events = 0;
if (poll(&pfd, 1, -1) != 1 || pfd.revents & POLLERR == 0)
        error(1, errno, "poll");
ret = recvmsg(fd, &msg, MSG_ERRQUEUE);
if (ret == -1)
        error(1, errno, "recvmsg");
read_notification(msg);
```

  在内核的实现中，这里知道要处理的数据就是如何在不拷贝buffer的情况下，将这些数据发送出去。Kernel中在处理这里的数据的时候，会一路处理到下面的函数。get_user_pages_fast会将用户态的Pages映射到内核的空间，并会将这些Pages Pin在内存中，避免换出。

```c
/**
 * get_user_pages_fast() - pin user pages in memory
 * @start:	starting user address
 * @nr_pages:	number of pages from start to pin
 * @gup_flags:	flags modifying pin behaviour
 * @pages:	array that receives pointers to the pages pinned.
 *		Should be at least nr_pages long.
 *
 * Attempt to pin user pages in memory without taking mm->mmap_sem.
 * If not successful, it will fall back to taking the lock and
 * calling get_user_pages().
 *
 * Returns number of pages pinned. This may be fewer than the number
 * requested. If nr_pages is 0 or negative, returns 0. If no pages
 * were pinned, returns -errno.
 */
int get_user_pages_fast(unsigned long start, int nr_pages,
			unsigned int gup_flags, struct page **pages)
{
	//...
}
```

### 0x04 SOCKMAP

 SockMap的功能是为了解决将高效地数据从一个地方发送到另外一个地方的问题。这个功能已经有了不少的应用，比如一些Service Mesh的程序中。SockMap主要利用ePBF的一些功能。主要是下面的bpf-helper中的一个API。bpf_sk_redirect_map用于将一个socket的packet重定向到另外的socket中去。

```
int bpf_sk_redirect_map(struct bpf_map *map, u32 key, u64 flags)
              Description
                     Redirect the packet to the socket referenced by map (of
                     type BPF_MAP_TYPE_SOCKMAP) at index key. Both ingress
                     and egress interfaces can be used for redirection. The
                     BPF_F_INGRESS value in flags is used to make the
                     distinction (ingress path is selected if the flag is
                     present, egress path otherwise). This is the only flag
                     supported for now.
              Return SK_PASS on success, or SK_DROP on error.
```

[5]中给出了一个简单的利用SocketMap实现简单echo服务器的例子，一般的情况下，echo服务器的实现就是读区client的出入然后重新发送，这里就是有两次的kernel-userspace的数据拷贝。基本逻辑如下echo-naive。而使用SockMap则可以省去两次的数据拷贝操作。下面例子中是idx=0将其重定向到输入的socket。

```c
// echo-naive
while (1) {
		int n = recv(cd, buf, sizeof(buf), 0);
		if (n < 0) {
			//...
		}
		if (n == 0) {
			/...
		}

		int m = send(cd, buf, n, MSG_NOSIGNAL);
		if (m < 0) {
			//...
		}
		if (m == 0) {
			 break;
		}
}

// ehco-sockmap
SEC("prog_parser")
int _prog_parser(struct __sk_buff *skb) {
	return skb->len;
}

SEC("prog_verdict")
int _prog_verdict(struct __sk_buff *skb) {
	uint32_t idx = 0;
	return bpf_sk_redirect_map(skb, &sock_map, idx, 0);
}
```

### 0x05 Several Options

* SO_BUSY_POLL，socket使用轮询的模式，在使用高速网卡，一些workload下面会有更好的效果，特别是将低延迟方面。

* EPOLLEXCLUSIVE，epoll的惊群的问题常见在一个进程创建一个epoll的时候，然后fork出多个子进程来处理accept的，常见的例子就是nginx。nginx中使用过的一种处理方式是使用一个accept mutex。EPOLLEXCLUSIVE在epoll_event.events中设置，使用epoll_ctl传入到kernel中。Kernel的在处理时候，如果发现设置了这个flag，会采用一个exclusive处理的函数，Patch的部分如下。add_wait_queue、add_wait_queue_exclusive都是kernel中关于wait queue的API。两者的不同是前者会唤醒所有的等待相同事件的操作，而后者只会唤醒一个。

  ```c
  		if (epi->event.events & EPOLLEXCLUSIVE)
  			add_wait_queue_exclusive(whead, &pwq->wait);
  		else
  			add_wait_queue(whead, &pwq->wait);
  ```

### 0x06 Ring Buffer

  这部分的内容来自一个网络栈优化的开源项目[6]，目前好像这部分的代码没有和入Linux Kernel中。[6]中的ring queue是一个有意思的设计，实际上这个设计在其它的一些地方也使用到了，比如DPDK中。这个ring queue的结构定义如下主要的结构就是一个producer的结果和comsumer的结果，这里两个的结构类似但是是分开的。后面是ring字段就是ring queue的数据不符，是C语言中一种常见的编程方式。sp_enqueue、sc_dequeue分配表示是否只有一个生产者 or 消费者，如果是的话使用当个的情况可以使用效率更高的实现方式。

```c
struct ring_queue {
	int flags;		/* Flags supplied at creation */

	/* Ring producer status */
	struct prod {
		u32 watermark;	/* Maximum items before EDQUOT */
		u32 sp_enqueue;	/* True, if single producer */
		u32 size;	/* Size of ring */
		u32 mask;	/* Mask (size-1) of ring */
		u32 head;	/* Producer head */
		u32 tail;	/* Producer tail */
	} prod ____cacheline_aligned_in_smp;

	/* Ring consumer status */
	struct cons {
		u32 sc_dequeue;	/* True, if single consumer */
		u32 size;	/* Size of the ring */
		u32 mask;	/* Mask (size-1) of ring */
		u32 head;	/* Consumer head */
		u32 tail;	/* Consumer tail */
#ifdef CONFIG_LIB_RING_QUEUE_SPLIT_PROD_CONS
	} cons ____cacheline_aligned_in_smp;
#else
	} cons;
#endif

	/* Memory space of ring starts here.
	 * not volatile so need to be careful
	 * about compiler re-ordering */
	void *ring[0] ____cacheline_aligned_in_smp;
};
```

 这个设计的优点整个都是无锁的操作。而且另外一个特点就是可以支持多个的读者、写者同时读取 or 写入。这个同时操作的逻辑依赖于prod、cons结构中的head、tail字段。下面以可能多个写者的情况下的写入操作为例。操作一开始就是查询目前buffer中是否有足够的空间，在有足够空间的情况下，CAS设置prod.head，修改prod的head的位置。由于这里是并发操作，而且没有使用锁，所以这里会是常见的循环的模式，在失败之后重试。下面就是写入数据到buffer中。这里有一个特点，后面while (unlikely(READ_ONCE(r->prod.tail) != prod_head))这句的作用是另外的写者在写入数据的时候，可能这个写者有改变了prod.head，但是不能保证另外的写者的写入操作已经完成了，所以需要等到tail == head的时候，保证这个写者设置的head前面的数据都是写入完成的，没有空洞。这个中支持多个写者同时写入，有得爆炸buffer中没有空间的方法在 数据库中 WAL的写入设计中也可以看到类似的思路。在前面的写入都完成之后，再去设置WRITE_ONCE(r->prod.tail, prod_next)。读取操作的思路和这个差不多。对去只会有单个的写者 or 读者的情况，就不需要考虑写写冲突 or 读读冲突，而只要考虑读写重读。体现在代码中的不同的地方就是设置prod.head的时候不会是CAS的操作，也不会有while (unlikely(READ_ONCE(r->prod.tail) != prod_head))这里的逻辑。理解这里的逻辑就可以发现这里的设计十分巧妙。

```c
static inline int
__ring_queue_mp_do_enqueue(struct ring_queue *r, void * const *obj_table,
			 unsigned n, enum ring_queue_queue_behavior behavior)
{
	u32 prod_head, prod_next;
	u32 cons_tail, free_entries;
	const unsigned max = n;
	int success;
	unsigned i;
	u32 mask = r->prod.mask;
	int ret;

	/* move prod.head atomically */
	do {
		/* Reset n to the initial burst count */
		n = max;

		prod_head = READ_ONCE(r->prod.head);
		cons_tail = READ_ONCE(r->cons.tail);
		/* The subtraction is done between two unsigned 32bits value
		 * (the result is always modulo 32 bits even if we have
		 * prod_head > cons_tail). So 'free_entries' is always between 0
		 * and size(ring)-1. */
		free_entries = (mask + cons_tail - prod_head);

		/* check that we have enough room in ring */
		if (unlikely(n > free_entries)) {
			if (behavior == RING_QUEUE_FIXED) {
				return -ENOBUFS;
			} else {
				/* No free entry available */
				if (unlikely(free_entries == 0)) {
					return 0;
				}

				n = free_entries;
			}
		}

		prod_next = prod_head + n;
		success = (cmpxchg(&r->prod.head, prod_head,
				   prod_next) == prod_head);
	} while (unlikely(success == 0));
	/* smp_rmb() for cons.tail is implicit by cmpxchg */

	ENQUEUE_PTRS(); /* write entries in ring */
	smp_wmb(); /* matching dequeue LOADs */

	/* if we exceed the watermark */
	if (unlikely(((mask + 1) - free_entries + n) > r->prod.watermark)) {
		ret = (behavior == RING_QUEUE_FIXED) ? -EDQUOT :
				(int)(n | RING_QUEUE_QUOT_EXCEED);
	} else {
		ret = (behavior == RING_QUEUE_FIXED) ? 0 : n;
	}

	/* If there are other enqueues in progress that preceeded us,
	 * we need to wait for them to complete
	 */
	while (unlikely(READ_ONCE(r->prod.tail) != prod_head))
		cpu_relax();

	WRITE_ONCE(r->prod.tail, prod_next);
	return ret;
}
```

 [6]中还有另外的一个的设计，同于实现一个qmempool。qmempool的优化是Linux网络栈在处理网络数据的时候。如果网卡的速度为100Gbps基本的，分配内存也会是一个耗时操作。qmempool的基本思路是将多个分配操作放在一起执行，实现基于内核的slab分配器。alf_queue是用于实现qmempool的一个结构。其基本的结构如下。看上去要比前面的ring queue要简单一些。其设计也是有两个actor，ring接在meta数据的后面的设计也是类似的，同样也区分了单个、多个的操作。

```c
struct alf_actor {
	u32 head;
	u32 tail;
};

struct alf_queue {
	u32 size;
	u32 mask;
	u32 flags;
	struct alf_actor producer ____cacheline_aligned_in_smp;
	struct alf_actor consumer ____cacheline_aligned_in_smp;
	void *ring[0] ____cacheline_aligned_in_smp;
};
```

 下面是其可能有多个写者的情况下的写入操作，emmmm，其实和ring queue逻辑上没有什么区别。

```c
static inline int
alf_mp_enqueue(const u32 n;
	       struct alf_queue *q, void *ptr[n], const u32 n)
{
	u32 p_head, p_next, c_tail, space;

	/* Reserve part of the array for enqueue STORE/WRITE */
	do {
		p_head = READ_ONCE(q->producer.head);
		c_tail = READ_ONCE(q->consumer.tail);/* as smp_load_aquire */

		space = q->size + c_tail - p_head;
		if (n > space)
			return 0;

		p_next = p_head + n;
	}
	while (unlikely(cmpxchg(&q->producer.head, p_head, p_next) != p_head));
	/* The memory barrier of smp_load_acquire(&q->consumer.tail)
	 * is satisfied by cmpxchg implicit full memory barrier
	 */

	/* STORE the elems into the queue array */
	__helper_alf_enqueue_store(p_head, q, ptr, n);
	smp_wmb(); /* Write-Memory-Barrier matching dequeue LOADs */

	/* Wait for other concurrent preceding enqueues not yet done,
	 * this part make us none-wait-free and could be problematic
	 * in case of congestion with many CPUs
	 */
	while (unlikely(READ_ONCE(q->producer.tail) != p_head))
		cpu_relax();
	/* Mark this enq done and avail for consumption */
	WRITE_ONCE(q->producer.tail, p_next);

	return n;
}
```

## 参考

1. https://patchwork.ozlabs.org/patch/528071/.
2. https://www.kernel.org/doc/html/latest/networking/msg_zerocopy.html.
3. sendmsg copy avoidance with MSG_ZEROCOPY.
4. https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/.
5. https://blog.cloudflare.com/sockmap-tcp-splicing-of-the-future/.
6. https://github.com/netoptimizer/prototype-kernel.
7. http://man7.org/linux/man-pages/man7/bpf-helpers.7.html.