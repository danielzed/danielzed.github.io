[文章链接](https://dzone.com/articles/understanding-reactor-pattern-thread-based-and-eve)

大规模请求处理上，一般有两种方式
- 每个请求一个线程去处理，因为进程的生成，上下文的切换都很重型，占用内存多而且慢，所以使用线程。同时，请求在各自线程处理，彼此不会串扰。但是，因为IO，网络的阻塞，大量线程处于等待状态，另外，线程的stack本身也是大量内存开销。

- reactor：即event-driven模式处理，包括一个bounded的queue存放请求，一个生产者，多个消费者。生产者，即网络线程，在accept socket之后，将事件加入epoll类似的轮询，并注册相应的可读，可写函数，等到linux系统判断相应事件触发，即会通知，此时调用消费者取出事件进行IO。 这种模式可以避免C10K问题。