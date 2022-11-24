# 切片集群模式
## 概念和设计
切片集群主要是用来，启动多个server均分整个集群的负载。每个server按照hash打散的槽位slot，分配各自处理的kv。作用在于，如果一台redis-server内存无限扩大，进行RDB的时候会进行fork完成持久化，此时对fork内存修改机会cow同步拷贝，从而引起阻塞。

但也引入了问题，主要逻辑涉及集群server信息变化时，槽位的重新分配以及数据的迁移。client如何知道想要的数据在哪台server？

集群redis是正常redis的降级，设计多个键值比如set的union和intersection没有实现。同时只有数据库0，不支持kv在集群间计算时移动。

集群节点间使用gossip协议传播消息和事件，类似病毒传播。

### 安全写入
集群节点间使用异步冗余备份（master和slave之间）。

可能丢失数据，因为是异步，不在IO路径上。主节点处理完客户端的写并返回之后，才会将写入同步给slave节点，如果同步之前挂掉则数据丢失。

另外，如果网络分区，导致主节点和从节点分区A,B，分区B从节点failover成master，分区A主节点接收的写入，在分区合并之后，可能丢失。

### 可用性
对于大量网络分区时的可用性，redis不很合适。一对主从节点，只要有一个留在分区内，该部分数据保持可用。

## 使用
cluster create命令创建集群
或者cluster meet手动建立实例间连接。再使用cluster addslots指定每个实例的哈希槽个数。

可以使用key hash tags，即{}包起用于计算hash slot的key部分。从而可以实现一些跨key操作。

## 内存结构体
集群间消息类型
```
#define CLUSTERMSG_TYPE_PING 0          /* Ping */
#define CLUSTERMSG_TYPE_PONG 1          /* Pong (reply to Ping) */
#define CLUSTERMSG_TYPE_MEET 2          /* Meet "let's join" message */
#define CLUSTERMSG_TYPE_FAIL 3          /* Mark node xxx as failing */
#define CLUSTERMSG_TYPE_PUBLISH 4       /* Pub/Sub Publish propagation */
#define CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST 5 /* May I failover? */
#define CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK 6     /* Yes, you have my vote */
#define CLUSTERMSG_TYPE_UPDATE 7        /* Another node slots configuration */
#define CLUSTERMSG_TYPE_MFSTART 8       /* Pause clients for manual failover */
#define CLUSTERMSG_TYPE_MODULE 9        /* Module cluster API message. */
#define CLUSTERMSG_TYPE_COUNT 10        /* Total number of message types. */
```

集群节点结构体
```
typedef struct clusterNode {
    mstime_t ctime; /* Node object creation time. */
    char name[CLUSTER_NAMELEN]; /* Node name, hex string, sha1-size */
    int flags;      /* CLUSTER_NODE_... */
    uint64_t configEpoch; /* Last configEpoch observed for this node */
    unsigned char slots[CLUSTER_SLOTS/8]; /* slots handled by this node */
    int numslots;   /* Number of slots handled by this node */
    int numslaves;  /* Number of slave nodes, if this is a master */
    struct clusterNode **slaves; /* pointers to slave nodes */
    struct clusterNode *slaveof; /* pointer to the master node. Note that it
                                    may be NULL even if the node is a slave
                                    if we don't have the master node in our
                                    tables. */
    mstime_t ping_sent;      /* Unix time we sent latest ping */
    mstime_t pong_received;  /* Unix time we received the pong */
    mstime_t data_received;  /* Unix time we received any data */
    mstime_t fail_time;      /* Unix time when FAIL flag was set */
    mstime_t voted_time;     /* Last time we voted for a slave of this master */
    mstime_t repl_offset_time;  /* Unix time we received offset for this node */
    mstime_t orphaned_time;     /* Starting time of orphaned master condition */
    long long repl_offset;      /* Last known repl offset for this node. */
    char ip[NET_IP_STR_LEN];  /* Latest known IP address of this node */
    int port;                   /* Latest known clients port of this node */
    int cport;                  /* Latest known cluster port of this node. */
    clusterLink *link;          /* TCP/IP link with this node */
    list *fail_reports;         /* List of nodes signaling this as failing */
} clusterNode;
```

集群状态
```
typedef struct clusterState {
    clusterNode *myself;  /* This node */
    uint64_t currentEpoch;
    int state;            /* CLUSTER_OK, CLUSTER_FAIL, ... */
    int size;             /* Num of master nodes with at least one slot */
    dict *nodes;          /* Hash table of name -> clusterNode structures */
    dict *nodes_black_list; /* Nodes we don't re-add for a few seconds. */
    clusterNode *migrating_slots_to[CLUSTER_SLOTS];
    clusterNode *importing_slots_from[CLUSTER_SLOTS];
    clusterNode *slots[CLUSTER_SLOTS];
    uint64_t slots_keys_count[CLUSTER_SLOTS];
    rax *slots_to_keys;
    /* The following fields are used to take the slave state on elections. */
    mstime_t failover_auth_time; /* Time of previous or next election. */
    int failover_auth_count;    /* Number of votes received so far. */
    int failover_auth_sent;     /* True if we already asked for votes. */
    int failover_auth_rank;     /* This slave rank for current auth request. */
    uint64_t failover_auth_epoch; /* Epoch of the current election. */
    int cant_failover_reason;   /* Why a slave is currently not able to
                                   failover. See the CANT_FAILOVER_* macros. */
    /* Manual failover state in common. */
    mstime_t mf_end;            /* Manual failover time limit (ms unixtime).
                                   It is zero if there is no MF in progress. */
    /* Manual failover state of master. */
    clusterNode *mf_slave;      /* Slave performing the manual failover. */
    /* Manual failover state of slave. */
    long long mf_master_offset; /* Master offset the slave needs to start MF
                                   or zero if still not received. */
    int mf_can_start;           /* If non-zero signal that the manual failover
                                   can start requesting masters vote. */
    /* The following fields are used by masters to take state on elections. */
    uint64_t lastVoteEpoch;     /* Epoch of the last vote granted. */
    int todo_before_sleep; /* Things to do in clusterBeforeSleep(). */
    /* Messages received and sent by type. */
    long long stats_bus_messages_sent[CLUSTERMSG_TYPE_COUNT];
    long long stats_bus_messages_received[CLUSTERMSG_TYPE_COUNT];
    long long stats_pfail_nodes;    /* Number of nodes in PFAIL status,
                                       excluding nodes without address. */
} clusterState;
```
## 关键流程和算法
### 入口main
主IO线程在processCommand中，如果开启了集群模式，并且请求不是master发来的，那么根据key所在槽位，返回失败或者moved消息。
### getNodeByQuery
如果一条命令中多条key怎么办？返回CLUSTER_REDIR_CROSS_SLOT错误。
如果多条key在一个slot，但是有key正在迁移或迁入怎么办？CLUSTER_REDIR_UNSTABLE
如果有槽位没有node处理？CLUSTER_REDIR_DOWN_UNBOUND
key的处理node不是myself？CLUSTER_REDIR_ASK或者CLUSTER_REDIR_MOVED
MOVED和ASK的区别，MOVED是槽位之后所有的请求永久转到新节点。ASK则是一次性的MOVED，不建议缓存。

如果集群已经down了？CLUSTER_REDIR_DOWN_STATE
myself处理这个key？CLUSTER_REDIR_NONE

集群down如何判断？判断server.cluster->state状态

如果判断正在migrate或者import，同时当前server不包含所有的key，那么返回CLUSTER_REDIR_ASK通知client重定向。
readonly-command可以直接返回myself。

### 重定向机制
当client发送到一台server，无法处理对应key的槽位时，返回MOVED信息指示key所在的server地址。
### slave副本逻辑

### resharding 碎片重整--集群在线重配置

### gossip
节点C刚加入集群，可能只认识节点B，那么先和B建立连接。B会在和其他节点A的gossip消息中，介绍C。从而A也会主动连接C。

## 失效检测
1. PFAIL：点到点网络不通
2. FAIL：PFAIL在gossip的基础上，达成了意见一致。

### epoch
1. cluster epoch:只发生在备升主的情况会更新epoch
2. configuration epoch：

### 从节点选举和提升
1. 时机：从节点认为主节点FAIL时.等待FAIL消息在集群中gossip出去。随机延时避免同时竞选分票。
2. 动作：增加currentEpoch值，要求其他主节点投票。
clusterHandleSlaveFailover 判断repl_offset的rank。增加自己currentEpoch，发起投票,类型为CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST,发给所有master和slave，但只有master应该回复，判断自己是否能failover自己的master。

一致性协议的示意图

### 备份迁移算法
提供更多的从节点，并且在某些主节点挂掉，从节点升主的配对中，将从节点迁移过去作为备份。
1. 触发条件：检测到由主节点没有备份











