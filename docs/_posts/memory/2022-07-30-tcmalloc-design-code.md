[知乎介绍tcmalloc](https://zhuanlan.zhihu.com/p/29216091)
vmem论文
tcmalloc是google开发的，thread-cache malloc内存池，针对线程缓存访问做了很多优化。

特点
- 快速，per thread提供object的cache，大多数分配都是lockless的，减少多线程竞争
- 灵活的内存分配，释放的内存被重用或者释放回obs
- 元数据开销更小

组件：
- front-end：提供per-thread和per-cpu的object cache
- middle-end：提供free-list以及transfer-cache，free-list管理从back-end申请的page span，管理pagespan是通过一个三层radix树，也是主要的管理结构。
- backend：从os中申请page。