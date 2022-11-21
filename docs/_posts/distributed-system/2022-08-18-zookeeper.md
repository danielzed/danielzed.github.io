zookeeper特点：
1. 简单的api，create，delete，读写，watch
2. 原子性，每个zk操作期间，都是全部成功，或者全部失败
3. 有序性。同一个client发出的请求，必然按照发出的顺序执行。是通过每个请求带的seqno实现。
4. fast：全内存数据库，通过磁盘的日志以及snapshot实现recover
5. 扩展性：读可以发到任意server上，写要统一发到leader上。适合多读少写场景。同时通过version number保证数据一致性。

常用场景：
1. 统一配置管理：将某些配置文件放在zk上，当更新时，server节点来取最新的配置。
2. 统一命名服务：dns，管理ip到节点名映射
3. 分布式锁：针对某个znode，多个client创建带序号的ephemeral znode，判断自己如果seqno最小则获得锁。否则监听小一号的状态改变，改变时被通知抢占锁。
4. 集群管理

实现：
1. znode类似文件系统，但是znode中所有节点都可以读写内容，也都可以有子节点。有4种znode类型，持久化-seq，持久化，短暂，短暂-seq
2. ephemeral znode。在client session保持期间存在，会话结束znode自动删除。


使用：
1. zkServer支持独立部署和集群部署。重点配置ticktime是timeunit，默认session timeout是2个ticktime。

2. 支持zkCli直连，也支持java和c的接口连接。

3. 副本模式下，需要配置集群中所有server。这时每个server需要两个端口。主要用于换主过程，因为换主过程中，两个主都需要tcp-port用于通信。 

4. client因为网络分区断链之后，建议不要新开handle去连接。因为zk的client library中实现了启发式的重连算法，解决herd effect。拿着这个handle等待server通知你超时再重连。

5. watch：一次性的，是getData，getChildren，exist的伴随配置。在收到watch事件之前，不会观察到data的变化，是线性关系的。

6. 会话的几种状态：connecting，connected，closed

7. 有两种接收watcher通知的方式，区别在于注册以及调用方式
- legacy：在zookeeper_init时注册，所有watch触发同一个函数，实现watcher_fn接口
- watcher object：在get类方法中注册，同时传入watchCtx上下文。所有get类方法，需要使用带有w前缀的版本。

8. 使用api
- zookeeper_init初始化，返回zhandle_t句柄。异步建立会话，收到ZOO_CONNECTED_STATE会话建立完成。
- zookeeper_close：关闭句柄会话。
- 操作包括几类：zoo_create，zoo_delete，zoo_exist，zoo_get，zoo_set，zoo_get_children以及zoo_multi事务提交多个op。a开头的版本代表异步，不带a就是同步。w开头的版本代表可注册watcher-func和ctx。














