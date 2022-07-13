[TOC]

# redis项目-源码阅读
## 项目背景
redis是单机kv存储系统，功能上提供hash，set,zset,list等多种数据类型，支持事务，订阅发布等高阶功能。同时，采用全内存获得高性能。
redis是cs结构，支持各种语言的api，同时实现了redis-cli发送命令。

## 问题
### 网络线程,结构体的内部实现？
### 内存结构体中的objec概念是什么作用,是为了方便内存管理么？
### 内存管理?


## 设计
### linux基础
pthread的信号量，mutex，pipe2操作

### 网络
#### connection
主要代码在connection.c,以及anet.c中。anet只要是将socket的操作封装起来,有很多linux系统使用的api，之后可以深入了解。
#### client
主要代码在networking.c中，其中关键结构体client，包含链接，网络buf，处理请求相关内容等。

在redis-server的请求处理中，会以如下流程创建client，main->initServer中根据server.ipfd注册accept-handler->acceptTcpHandler->acceptCommonHandler。

而网络到请求处理的接口，则在client注册的readHandler函数，readQueryFromClient->processInputBuffer->processCommandAndResetClient->processCommand针对不同语义命令，不同server状态进行不同处理.
按流程梳理时可以写个时序图！！
### 请求处理
#### asyncEpoll
redis实现了一套aeEventLoop的事件轮询机制,内部支持4种实现，select，kqueue，evport，epoll。

event-driven architecture很好的解耦了消息的发送者和处理者，同时避免轮询带来的开销。
[aeEventLoop组件](./resources/aeEventLoop组件.eddx)

#### 事件
事件类型，包含读，写，时间，屏障事件
- 屏障事件：当前时间的读写事件要按照先处理可写，在处理可读的顺序来进行。正常先可读，在可写。那么可读处理完的命令，一次循环就能reply。如果reply之前需要进行些处理，就要barrier，这样就在下次epoll才reply

#### 3种轮询请求模式
IO多路复用，单线程内处理多个请求
##### select
```
int select (int __nfds, fd_set *__readfds, fd_set *__writefds, fd_set *__exceptfds, struct timeval *__timeout)
```
问题在于，监听的文件描述符上限有限制，1024，其次，判断哪些fd就绪需要遍历fd_set。
##### poll
```
int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout);
```
fd上限超过1024，但是仍然需要遍历fd_set
##### epoll

#### reactor响应模式
三种角色，reactor，accepter，handler。其中ae即为reactor的角色，定义了FileEvent的结构，以及event的事件驱动逻辑。

### 内存数据结构
数据结构操作通过db进行管理，db间是彼此隔离的，db数默认16个，通过redis命令select进行切换。
#### string类型
dict结构体，内部通过hash_table实现，开链法解决冲突，将新增元素放在链表开头。
同时使用主备两个hash_table，完成rehash动态扩缩容。

#### set类型
同样是dict结构体，hash_table实现，插入前会判断元素是否存在。

#### zset类型 
有序集合，每个元素会伴随一个score作为排序依据。
代码主要在t_zset.c中，内部有两种实现，一种LISTPACK，另外一种SKIPLIST，保存在zset结构体中.

#### hash类型
内部通过hash_table实现

#### dict类型
hash表采用开链法解决冲突，当负载因子大于一定程度时(1,5)会采取rehash，降低每个桶上的元素链表长度，这个过程叫做rehash。

rehash采用渐进式，双dicttable，从而在rehash过程中不影响正常业务.渐进rehash有两种渐进维度，一种是指定rehash n个桶_dictRehashStep，一种是rehash tms时间dictRehashMilliseconds。

iterator功能，提供fingerprint hash计算dict状态作为iterator的一种快照版本

scan功能
- 通过cursor完成读续传功能，遍历整个dict的数据，返回0时代表遍历完成。保证全部返回，不保证只返回一次
- 这个功能用在hash，set,zset,string等类型的scan命令中。
- 其中cursor需要翻转，+1，再翻转，保证在高位进行+1，原因是什么？ 以及为什么不保证只返回一次呢，什么时候同一个kv返回多次？

#### ziplist类型
双向链表结构，通过对bit的操作提高内存使用效率。主要体现在zlentry的encoding字段上，变长编码。支持插入操作？
- 连锁更新问题：prevlen变大，导致prevlenSize变大，导致后续元素prevlenSize连环变大。
- 过长list，遍历效率下降

##### 增强版实现quicklist
内部一个链表，每个元素是一个ziplist。控制每个ziplist大小和元素个数，从而避免性能开销


##### listpack
紧凑列表，通过变长字符串的编码减少内存开销

不再保存prevlen避免连锁更新。而反向遍历的功能，通过对entryLen的精巧设计来完成。

#### zset
实现采用了跳表+hash表完成，从而提供了O(logn)的范围查询和O(1)的单点查询。

#### radix-tree
redis的stream语义中，因为有这样一个特点，连续的消息往往key存在很长相似项，避免内存重复存储，使用前缀存储类型，比如radix-tree以及listpack


stream语义用到了radix-tree，基数树，也叫多叉树，举例前缀字典树
- trie-tree前缀树存在问题：很多相同的前缀以及很多不同的后缀，如果都按照一个char去构造node，那么内存会浪费。而radix-tree则是在此基础上的改进，如果一个点分叉之后，儿子是叶子节点，那么儿子就会合并成一个节点。
- 压缩节点只能指向一个合并子节点，非压缩节点可以指向多个单字符子节点
[radix-trie,几个insert的case图很好](https://iq.opengenus.org/radix-tree/)

### 高阶功能
#### 缓存替换策略
LRU的近似策略，减少内存开销

##### LFU策略

##### lazy-free
后台线程删除evict的kv。如何不影响前端业务.

惰性删除就是将redisObject的引用地方全部删除，但是内存(会评估释放开销，开销大的，比如很多元素的set的删除)放在后台来释放,也即bio线程

#### pubsub
代码主要在pubsub.c中.

subscribe订阅频道，订阅可以指定字符串，也可以指定正则表达式
订阅时将channel加到client的订阅hash_table中，同时将client加到server的订阅client-list中。
在流程publishCommand->pubsubPublishMessageAndPropagateToCluster->pubsubPublishMessage-> pubsubPublishMessageInternal中将发布的消息发到所有订阅的client上


#### 持久化
redis本身是全内存，可能会丢失数据，所以需要提供持久化方式保证数据持久性
##### RDB
RDB即redis DB，通过快照方式完成，快照触发有四种场景
- 自定义配置的快照规则
- 执行save或bgsave
- 执行flushall
- 执行主从复制

特点：
1. RDB生成二进制压缩文件，空间占用下
2. 主进程fork出子进程进行持久化，不影响正常业务处理
3. 恢复数据快？？
缺点：
1. 不保证数据完整性，快照之后的数据没有持久化
2. 父进程在fork子进程时，可能因为进程堆栈过大而阻塞。

```
rdbSaveBackground中fork子进程->rdbSave->startSaving->moduleFireServerEvent插入持久化事件到server,rdbSave->rdbSaveRio将内存数据转储为RDB格式，并发到指定的redis io-channel
```

rdb文件存储的kv类型，是如何恢复成内存状态的？包括expiretime，LFU， MODULE这些？

##### AOF
AOF即append only file，为写后日志,每条命令server成功执行完，发给AOF程序进行转码，存储,支持内存syncBuf。通过redis.conf的appendonly yes开关控制

AOF重写：针对AOF文件无限制膨胀，在后台子线程中定期压缩，方式为记录当前内存值类似快照。重写期间的一致性问题，引入了AOF重写缓存解决，将客户端重写期间的命令双发到现有的AOF文件以及重写缓存.

AOF可以配合RDB使用，打开方式aof-use-rdb-preamble yes
- 没有其他AOF或者RDB在运行时才可以执行AOF重写。如果RDB在运行，AOFrewrite会设置一个标记，在serverCron定时任务中,等待满足条件时触发重写.
- AOF文件包括三种文件，BASE，INCR,HIST。base即快照，可以根据文件内容恢复内存状态。快照支持两种,RDB和AOF-preamble。RDB类型快照，直接将内存db的kv dict持久化，而AOF-preamble则将相应的操作，比如set，sadd持久化，理论上来讲占用空间更大，不知道有什么好处。
- INCR代表增量修改文件，HIST则为历史，其中数据均无效。在AOFRW过程中，会将旧的BASE和INCR置为HIST，后台删除。
#### 事务
```
discard: 丢弃multi之后的所有命令
exec：执行MULTI之后的所有命令
multi：开启事务块
unwatch：丢弃watched keys
watch：监控给定的key
```
实现方式，即为在client中flags增加CLIENT_MULTI状态，在processCommand中判断是MULTI状态，则将输入的指令加入client的multiState结构体中的队列。
exec中则会判断watched-key是否被修改，然后将mstate中缓存的command依次拿出执行，此时client进入EXEC状态.

### 集群部署
main中针对clusterfd注册监听函数clusterAcceptHandler，其中建立与其他节点的连接。

主从备份
- 状态机实现，包含4个阶段，初始化阶段，建立连接阶段，主从握手阶段，备份类型以及执行阶段
- 除了包含共识协议选主的功能之外，还包含负载均衡打散的响应功能，对于某些key会转到某些client处理。

### 阻塞功能
- blpop等操作当list没有元素时，会阻塞住当前client，直到list插入元素通知可用。内部通过ready_keys列表完成通知。
- server内存保存所有阻塞的key，以及在这些key上blocked的相应client。

#### sentinel哨兵模式
在master宕机后，避免手动切master。建立哨兵与集群内server通信，确定master状态，从而及时切换master。通知其他slave master已经切换的方式，是通过发布订阅。

为避免单个哨兵的主观错误，比如网络分区，也可以建立多个哨兵进行投票。这里的共识机制如何实现？raft协议选主

TILT模式，只接受处理消息，不主动进行故障切换功能。

哨兵和其所监听的主节点间，通过pubsub通信,汇报哨兵信息

- failover故障切换

- 哨兵，是获得集群信息的可靠源。哨兵的集群和redis的集群有些不同？
- 节点down有两种，一种是主观down，即经过大多数quorum 哨兵认可的。一种是客观down，网络连接失败。
- 哨兵需要在配置文件中注册其监控的master节点，可以监控多个。replica节点可以自动发现，并且会持久化到配置文件中。	
#### gossip协议
- 去中心化集群节点信息维护
- 节点信息包括节点名称，IP，端口号等，通过ping pong协议通信。集群中数据分布情况，以slots形式标识

#### 请求重定向
当集群节点接收到某个请求，但数据在·其他节点时，需要将请求重定向。

getNodeByQuery获取能够执行某个请求的clusterNode。
- 可能的失败情形
1. CLUSTER_REDIR_CORSS_SLOT：请求跨多个hash slot，但是一定跨多个节点么？slot是对应节点么？
2. CLUSTER_REDIR_UNSTABLE：请求对应同一个slot中的多个key，但是slot不稳定，正在迁移。一般发生在resharding过程。
3. CLUSTER_REDIR_DOWN_UNBOUND：slot不绑定到任何节点

### bio
redis并不是个完全的单线程，有些耗时并且非IO的任务会放在后台来做。包括：
```
BIO_CLOSE_FILE
BIO_AOF_FSYNC
BIO_LAZY_FREE
```
将fd的close，aof的fsync以及资源的free放到后台任务链表，慢慢执行

### ioThread
redis并不是将所有IO操作完全的交给单个线程，也提供ioThread功能，共同处理主IO线程的请求。这部分请求不是对接到fd上由事件触发，而是由主IO线程将用户buffer转换得来的读或者写请求，然后进行writeToClient或者readQueryFromClient的处理.

注意，IO线程只是处理数据的接收和发送，并不处理命令本身？？

### module概念
动态链接模块，使用外部模块扩展redis的功能。可以在启动时加载，也可以通过MODULE LOAD命令加载，也可以在redis.conf中 loadmodule指定对应的so文件。

外部module需要满足接口标准，定义在redismodule.h中，包含各种redis IO命令以及管理命令。
- 实现native的数据类型
- 阻塞操作，只阻塞client，不阻塞server。

## 关键数据结构

### 网络连接

## 应用
### 分布式锁
set nx ex 当某个key不存在时插入，并制定超时时间。如果已经存在，则无法成功加锁。


## 算法
### pqsort部分快排
[部分快排](https://blog.csdn.net/guodongxiaren/article/details/45567291)















