---
layout: page
title: New Syscalls of Linux 4.x
tags: [Operating System]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## New Syscalls of Linux 4.x

### 引言

  Linux 4.19 就要发布了，按照套路，4.19应该是Linux 4.x的最后一个版本。Linux 4.0发布在2015年4月(那时候还是大二的小懵懂)。这几年中Linux改变了那些东西呢。这篇是总结了一下新增的syscalls的，其实syscall与新特性大部分时候都没啥关系，这里只是为了好玩。



### Syscalls

####  copy_file_range

```c
ssize_t copy_file_range(int fd_in, loff_t *off_in,
                        int fd_out, loff_t *off_out,
                        size_t len, unsigned int flags);
```

  Linux 4.5中添加的，主要是为了减少不必要的数据拷贝。比如从一个文件的数据拷贝到另外一个文件是，可以不经过user space。



####  mlock2

```c
int mlock(const void *addr, size_t len);
int mlock2(const void *addr, size_t len, int flags);
int munlock(const void *addr, size_t len);
```

  Linux 4.4 添加的，mlock的强化版本，主要就是添加了一个flags的参数。主要功能就是将一个process的page不要被sweep到磁盘上面。



####  pkey_alloc,  pkey_free,  pkey_mprotect

```c
int pkey_alloc(unsigned long flags, unsigned long access_rights);
int pkey_free(int pkey);
int pkey_mprotect(void *addr, size_t len, int prot, int pkey);
```

 这个主要是对page权限的改进。

```
Memory Protection Keys provide a mechanism for changing protections without requiring modification of the page tables on every permission change.
```



####  preadv2, pwritev2

```c
ssize_t preadv(int fd, const struct iovec *iov, int iovcnt,
                      off_t offset);

ssize_t pwritev(int fd, const struct iovec *iov, int iovcnt,
                       off_t offset);

ssize_t preadv2(int fd, const struct iovec *iov, int iovcnt,
                       off_t offset, int flags);

ssize_t pwritev2(int fd, const struct iovec *iov, int iovcnt,
                        off_t offset, int flags);
```

preadv, pwritev的添加了一个flags参数的版本。



####  statx

```c
int statx(int dirfd, const char *pathname, int flags,
                 unsigned int mask, struct statx *statxbuf);
```

stat的强化版本。

```
To access a file's status, no permissions are required on the file itself, but in the case of statx() with a pathname, execute (search) permission is required on all of the directories in pathname that lead to the file.
```



####  userfaultfd

```c
int userfaultfd(int flags);
```

  上面的几个都很无聊，这个比较有意思一点，在Linux 4.3 添加。通常情况下，page fault是有kernel处理的，这个syscall通过文件的形式给user space处理page fault提供了一些功能。[2]中有一些说明。



>

### New Syscalls of Linux 3.x

  顺便看一下3.x的吧，emmmm。



#### bpf

```c
int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

 Berkeley Packet Filters相关的一个syscall，挺有用的。



#### execveat

```c
int execveat(int dirfd, const char *pathname,
	char *const argv[], char *const envp[], int flags);
```

 execve(2)的强化版本。



#### finit_module

```c
int finit_module(int fd, const char *param_values, int flags);
```

Load一个ELF image到kernel space，类似init_module。



#### getrandom

```c
ssize_t getrandom(void *buf, size_t buflen, unsigned int flags);
```

借助内核生成随机数，比直接读区/dev/random更加好用。



#### kcmp

```c
int kcmp(pid_t pid1, pid_t pid2, int type,
                unsigned long idx1, unsigned long idx2);
```

检查2个进程的共享资源的信息。



#### kexec_file_load

```c
long kexec_file_load(int kernel_fd, int initrd_fd,
                           unsigned long cmdline_len, const char *cmdline,
                           unsigned long flags);
```

 kexec_file_load用来用于在reboot之后使用新的kernel。



#### membarrier

```c
int membarrier(int cmd, int flags);
```

 提供一个类似内存屏障的功能.



#### memfd_create

```c
int memfd_create(const char *name, unsigned int flags);
```

 将内存当作文章。



#### process_vm_readv, process_vm_writev

```c
ssize_t process_vm_readv(pid_t pid,
                                const struct iovec *local_iov,
                                unsigned long liovcnt,
                                const struct iovec *remote_iov,
                                unsigned long riovcnt,
                                unsigned long flags);

ssize_t process_vm_writev(pid_t pid,
                                 const struct iovec *local_iov,
                                 unsigned long liovcnt,
                                 const struct iovec *remote_iov,
                                 unsigned long riovcnt,
                                 unsigned long flags);
```

对vm space做一些操作。



#### renameat2

```c
int renameat2(int olddirfd, const char *oldpath,
                     int newdirfd, const char *newpath, unsigned int flags);
```

 renameat添加了flags参数.



#### sched_setattr, sched_getattr

```c
int sched_setattr(pid_t pid, struct sched_attr *attr,
                         unsigned int flags);

int sched_getattr(pid_t pid, struct sched_attr *attr,
                         unsigned int size, unsigned int flags);
```

scheduling policy相关的syscalls。



#### seccomp

```c
 int seccomp(unsigned int operation, unsigned int flags, void *args);
```

Secure Computing (seccomp)，终于看到一个有趣的一点的了，这个syscall在一些地方很有用，最大的一个用处就是，终于看到一个有趣的一点的了，这个syscall在一些地方很有用，最大的一个用处就是可以对调用syscall做一些限制。



#### sendmmsg

```c
 int sendmmsg(int sockfd, struct mmsghdr *msgvec, unsigned int vlen,
                    int flags);
```

 这个也是一个比较有用的syscall，主要就是将多个syscall合在一起，平摊(amortize)syscall的开支。

 

#### setns

```c
int setns(int fd, int nstype);
```

 Linux namespace 直接相关的syscall，Docker之类的都要使用的东西。Linux 3.0的时候就添加了。



## 参考 

1. http://man7.org/linux/man-pages/man2/syscall.2.html
2. http://xiaogr.com/?p=96, Look Into Userfaultfd