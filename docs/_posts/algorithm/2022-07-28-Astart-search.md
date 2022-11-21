[A*算法知乎介绍](https://zhuanlan.zhihu.com/p/54510444)

A*算法是用来寻找start到end的最短路径的，那么几个算法对比如下：

- BFS：按照距离一层一层遍历。问题在于遍历了很多不必要的点，同时无法处理那些路径长度不为1的情况。

- dijkstra算法：使用优先队列存储所有可达节点，选择最近的，更新剩余节点路径，更新优先队列重排序。再选择最近的，不断循环