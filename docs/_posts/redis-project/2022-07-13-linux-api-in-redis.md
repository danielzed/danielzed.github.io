## 内存相关

1. strchr(char*,char)：返回某个char在字符串中出现的指针位置。没有返回null


## 网络相关
1. getaddrinfo(char*node,char* service, addrinfo* hints, addrinfo **res):node和service指定了一个地址和服务，返回的addrinfo可以用于bind和connect中。内部使用gethostbyname以及getservbyname实现。
2. htonl,htons,ntohl,ntohs
- 主机序与网络序的转换，h代表host，n代表network，s代表short，l代表long
3. getpeername(fd, sockaddr*, sockaddrlen) 获取对端地址。
- ss_family字段包含几种类型，AF_INET,AF_INET6,AF_UNIX。


## 事件相关
1. epoll_ctl支持多种操作，EPOLL_CTL_ADD添加事件fd到epfd的监听.ee传入监听的事件，比如EPOLLIN,EPOLLOUT

## 多线程
1. pthread_mutex_lock 加锁
2. pthread_cond_signal 唤醒信号量
3. pthread_mutex_unlock 解锁