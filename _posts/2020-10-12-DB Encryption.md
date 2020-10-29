---
layout: page
title: Azure SQL Database Always Encrypted
tags: [Database, Security]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Azure SQL Database Always Encrypted

### 0x00 基本内容

 这篇Paper讲的是Azure SQL Database的Always Encrypted(AE，第二个版本)加密功能，其可以实现加密的数据从client出去之后就一直是加密，并利用enclave running within a trusted execution environment可以提供在加密上面的一些查询功能的支持。在AEv1时，AEv1使用的方式是对要加密的列使用的 deterministic encryption的方式，即同样的明文在同样的密钥加密之后的密文是一样的，这样会有一些的安全问题。 deterministic encryption的优点是可以很简单就支持点查询，以及equi-joins, 和 equality-based grouping基于等值的查询操作。而且AEv1在切换加密密钥的情况下也很麻烦，

```
turning on encryption for the first time (initial encryption) and rotating encryption keys both require a roundtrip to client systems possessing the encryption key(s), which can be prohibitively expensive. For terabyte large databases, this roundtrip can result in latencies as long as a week for ini- tial encryption and key rotation....
```

 为了优化v1的一些限制，AEv2应运而生。v2使用了trusted execution environment (TEE)，这个TEE作为一个安全区域，可以基于Windows的Virtualization-based Security (VBS) 来实现，也可以基于Intel SGX功能来实现。TEE可以视为一个安全的环境，所以可以在TEE中对数据进行解密操作，这样就不仅仅支持相等的比较操作，只支持deterministic encryption的限制。这样也会有一些问题：1. 怎么证明这个TEE环境是可靠的，需要证明enclave中执行的程序的可信任性，以及在各种环境下面可以正常工作；2. 在这个安全的TEE环境中如何和不安全的SQL Server进行交互；3. 如何在这样的架构中进行devops，如何运维，如何排查问题等。AE的总体设计上，

* AE使用两层key的方式去加密数据。数据部分使用对称加密，使用AES加密算法。在table中会有一列的column encryption key (CEK)，长32byte。这个CEK使用第二层的key来加密，称之为column master key (CMK)，保存在其它的地方。SQL Server可以通过table元信息中带有的URL信息引用到这个key，但是不能直接获取到合格key。这个CMK支持多种类型的key provider(s)。

  ```
  CREATE COLUMN MASTER KEY MyCMK WITH ( 
    KEY_STORE_PROVIDER_NAME = N'AZURE_KEY_VAULT_PROVIDER',
    KEY_PATH = N'https://vault.azure.net/...', 
    ENCLAVE_COMPUTATIONS (SIGNATURE = 0x6FCF...))
    
  CREATE COLUMN ENCRYPTION KEY MyCEK
    WITH VALUES (COLUMN_MASTER_KEY = MyCMK,
    ALGORITHM = 'RSA_OAEP', ENCRYPTED_VALUE = 0x0170...)
    
  CREATE TABLE T(id int,
    value int ENCRYPTED WITH (COLUMN_ENCRYPTION_KEY = MyCEK,
    ENCRYPTION_TYPE = Randomized,
    ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'))
  ```

  通过上面这样的建表语句，可以指定其使用AE的一些信息。

* AEv2可以支持Deterministic (DET)的加密方式和Randomized (RND)方式。DET使用AES CBC模式，使用加密数据的SHA作为初始化向量IV；而END使用AES的Cipher Block Chaining (CBC) 模式，使用随机的IV。RND一般会更加安全，所以一些v2的功能只支持了END。在enclave-enabled的情况下，可以支持 equality, range 和 pattern matching queries (LIKE predicate)等操作。在加密之外，每个加密的value还会包含一个HMAC，用来保证数据的完整性(data in-tegrity)。用于避免一些恶意的client添加进去比如 random byte作为加密数据。

### 0x01 基本设计

SQL ServervAlways Encrypted的基本架构如下图。在client端进行加密解密的操作，这些操作有数据库driver进行，对于应用是透明的。SQL Server中另外添加了一个新的api sp_describe_parameter_encryption，这个api输出包含了加密的一些信息，比如加密之后的CEK以及CMK的元信息等，

```
The output of this call for a parameterized query contains: the encryption type information (CEK) for each parameter; if the evaluation of the query requires an enclave, the output also contains the set of CEKs required within the enclave. For each CEK above, the output contains the encrypted CEK and the CMK metadata.
```

但是这里SQL Server是不可信的，也就这个api的输出可能是不可信的。如果操作需要用到enclave computations，则SQL Sevrer需要请求attestation server，返回给client的时候需要包含attestation information(这个信息也会包含在上面api的输出之中)，这样这里的话SQL Server有一个不受信任的man-in-the-middle，即中间人的角色。使用这里需要另外的方式另外api的输出是可信的。

* Driver会是用前面的这些信息获取CMKs，解密CEKs，给SQL Server发送查询请求的时候，参数也都会是加密的。如果请求需要用到enclave，enclave需要的CEKs使用一个shared-secret based secure channel添加到enclave中。AE提供的data confidentiality、 data integrity 等的保证，但是不保证metadata confidentiality ，比如表结果，记录条数等是暴露的。另外这里是不保证semantic security，语义安全的。
* SQL Server和enclave交互，让enclave处理一些需要看到解密之后数据的操作。SQL Server大部分的操作是不需要看到具体的数据，对于需要看到的部分，这里使用一个模块称之为expression services (ES)。Enclave作为ES的一部分。比如对于一个select * from T where value = @v这样的查询，如果使用表扫描的方式来执行这个查询，则对于每一条记录，对于value = @v的操作都会在enclave中进行，而对SQL Server输出一个Boolean类型的结构。当然除了扫描的方式，还要处理的是在加密之后的数据上面添加index的问题。对于DET类似的加密，直接就是使用加密之后的数据作为key，使用B+ tree作为index结构。对于RND类型的加密，也是使用B+-Tree作为，但是使用enclave来进行比较，比较的结果输出给SQL Server。

![](/assets/png/ae-arch.png)

前面提到SQL Server给出的sp_describe_parameter_encryption的结果是不可信的，这里需要额外的方式验证。这里引入了attestation，用于检查enclave处于正常状态。Attestation主要分为两个部分，一个是enclave平台正常与否，另外一个是enclave里面运行的代码正常与否。以VBS enclave为例，这种情况下，认为hypervisor是可信的，而运作在其之上的host kernel是不可信的。这里会利用Windows的Host Guardian Service功能，HGS则使用Trusted Platform Module (TPM) 来监测host machine的状态。

```
 TPMs measure the boot sesquence of a host and the measurement is returned in the form of a log called the TCG log. In an offline step, the TCG log obtained from the machine hosting SQL is registered with the HGS service to be included in its white-list. For VBS enclaves, we only trust the hypervisor, but not the host kernel. Therefore, we are only interested in the measurement of the boot sequence until the hypervisor is loaded. 
```

验证VBS enclave的时候，SQL调用Windows，发送目前的 TCG log到HGS，HGS来验证其是否在white-list中。然后通过这些功能，获取到一个health-certificate，这个health-certificate由一个HGS signing key进行签名。还会保护一个由host hypervisor发出的signing key，称之为host signing key。然后是SQL请求Windows去检查enclave，这个操作enclave会报告其一些属性，包含auth ID等，这个ID关联到一个对enclave 中运行的binary签名的 signing key，其包含了一个binary的hash值，以及the enclave 和 the host hyper-visor的version numbers。这里实际上就是一个证明enclave可靠，又证明其中运行的binary可靠的一些方式，和获取后面会用来验证的一些信息。。这里 VBS enclave会在被load的时候创建一个RSA public/private key，前面enclave报告自身的信息也会包含public key的一个hash信息。Enclave和driver之间通过Diffie-Hellman (DH)密钥交换协议来建立shared secret。在调用 s p_describe_parameter_encryption之后，SQL返回这些信息，

```
(1) The host health certificate containing the host signing key.
(2) The enclave report signed by the host signing key.
(3) The enclave’s public key and DH public key, signed by the enclave’s public key. Since the client sends its DH-public key as input, at this point, it follows from the DH protocol that the enclave already holds the shared secret.
```

客户端会检查由HGS signing key签名的health-certificate，从HGS的提供的接口中获取HGS signing key；检查由host signing key签名的enclave report的数据；检查encalve bianry通过specially provisioned signing key签名encalve binary，另外还会检查其version number，但是这个版本信息证明维护的paper中说的比较简单；检查enclave public key，enclave DH public key由the enclave public key签名。也就是说这里通过TPM来保证hypervisor是可信的。TPM来防止hypervisor被恶意修改。Host Guardian Service (HGS)使用了TPM来检查host machine的健康与否，在这篇paper之外，azure的文档中还记录了其可以使用“主机密钥”的方式。HGS通过HGS signing key签名host signing key等的信息，这样client就能检查host signing key等的信息是否是合法的。host signing key又会签名enclave提供的信息，来证明enclave提供的信息是合法的。。另外在SQL Sever上面的一些改进：

* Metadata and Type System。SQL中每一列的数据会有一个类型，SQL Server这里对type system进行了一些改进，支持记录每一列加密的相关信息。另外对类型推导也进行了一些改进，加入了新一个步骤，称之为encryption type deduction[1]。

*  Expression Services。ES的一部分运行在enclave中，这部分可以看到加密数据解密之后的明文。需要在ES中执行的expression都会被编译为ES objects，其class称之为CEsComp，保存在一个plan cache中。运行的时候，会生成一个可执行的CEsComp，另外CEsExec对象会暴露一个Eval接口，来执行这个编译之后的表达式。CEsExec可以执行查询中所有的标量的操作，比如evaluating filter predicates, computing hashes for hash join probing和 checking join equality等。如果这个ES运行在enclave中的时候，它需要向OS申请一些资源，但是OS是不可靠的，enclave runtime不支持OS提供的一些资源管理的方式。为了解决这个问题，在(1) reimplement ES, (2) port ES with SQL OS, 和 (3) port ES without SQL OS三种方案中，这里选择了port ES to run inside the enclave的方式，其既可以避免reimplement ES需要重新实现的很多功能，有可以避免改动SQL OS这样一个大的组件。这里实现一个small SQL OS layer，称之为enclave SQL OS。ES会expression将其编译为两个版本的binary，一个在enclave外运行，一个在enclave之内。这里加入了TMEval指令，用于invokes an enclave computation。这里涉及了不少这个系统内部的东西，paper中描述的只能大概了解这里做了些啥。

  ![](/assets/png/ae-es.png)

* Indexing，Paper中提到了一个SQL Server recovery要处理的一个地方：加密的index需要enclave中的keys，这些keys是在client发送查询请求的时候带上来的。如果一个client从来没有运行过涉及到这个加密列的index，这样就获取不到相关的keys，从而block这个recovery的操作。为了解决这个问题，这里使用的思路是defer，利用了之前SQL Server的deferred transactions的功能。在后面client的时候发送了这个key，这个deferred transactions会被resolved。但是这个deferred transactions可能持有lock，从而对后面运行的事务造成影响，这里使用之前onstant-time recovery (CTR)功能来解决这个问题，CTR会持久化保存数据的不同的版本，

  ```
   when the database recovers from a crash, clients only get access to the latest committed version (with all locks released), while uncommitted versions are cleaned in the background. In the presence of an encrypted index, the database is fully avail- able for clients, but the version cleaner that performs index traversals could potentially not find keys in the enclave, in which case it keeps retrying.
  ```

   这样就只会是block来一些清理的工作。但是如果client一致不提供的话，也会对数据库的运行造成影响，比如不能执行 log truncation的操作。这里加入了一个强制resolution的逻辑，会skipping recovery of index pages，同时标记 the index as invalid in the metadata。但是在clustered index 这样可能造成data loss，这里就不支持clustered index。

这里还是涉及到这个系统内部的信息，思路还是可以看一下。

### 0x02 评估

 这里具体可以参考[1].

## 参考

1. Azure SQL Database Always Encrypted, SIGMOD '20.