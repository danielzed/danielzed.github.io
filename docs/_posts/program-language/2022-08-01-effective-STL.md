主要介绍容器，algorithm，func概念及使用技巧。

# container
1. 慎重选择container

几个类别
- 标准STL序列化容器：vector，string，list，deque
- 标准STL关联容器：map，multimap，set，multiset
- 非标准STL序列化容器：slist，rope(string增强版)
- 非标准STL关联容器：hash_map,hash_multimap,hash_set,hash_multiset
- 标准非STL容器：array，bitset，queue，stack，priority_queue

选择正确的容器，需要根据使用的场景需求，包括很多问题：
- 是否需要多个insert elem之间保持事务语义
- 是否需要和c的内存layout适配
- 是否需要支持，即使删除某些elem，获得的ref和iter也不会失效。

2. Beware the illusion of container-independent code.

很难写出container-independent代码，因为不同容器的特性差距很大，支持什么类型迭代器，是否C 内存layout兼容，是node-based还是整块内存，删除元素或者插入，是否使得已经获取到的iterator失效。

当接口要兼容所有可能传入的容器时，内部使用容器就会很受限。

相反，可以在设计接口的时候，将使用什么容器封装在接口里面，这样可以根据需求选择不同容器共同完成任务。

3. Make copying cheap and correct for objects in containers.

STL的容器，在push或者front读的时候，都会调用copy func，所以如果一个object是copy expensive的就要注意了。

另外，将derived object拷贝给base object，会调用copy constructor构造一个base object。这种情况应该使用指针，更高效，也不会出错。使用智能指针可以解决内存的管理问题。

4. Call empty instead of checking size() against zero

因为empty肯定是常数时间可以完成的，但size，在某些情况，比如list的splice接口，是需要n时间计算size的。

5. Prefer range member functions to their single-element counterparts

container的范围方法一般都比single-element加一层循环更有效
- 减少函数调用
- 内部减少频繁的移动和拷贝
- 直接指导比如insert之后的总元素数量，申请一次内存就可以容纳

range-func包括：
- range-construct
- range-erase
- range-insert
- range-assignment

6. Be alert for C++'s most vexing parse

c++很容易将一些函数调用，构造函数初始化解析成某个func的定义，需要注意。

7. When using containers of newed pointers, remember to delete the pointers before the container is destroyed.

使用for_each比for-loop要好，可以exception-safe，同时更简洁。

同时如果在new和delete之间出现异常，此时可以用smart-pointer解决。

8. Never create containers of auto_ptrs.

容器中不能使用auto_ptr，甚至编译不过。这是因为很多STL的algorithm，比如quick_sort，里面是会有copy的，此时如果auto_ptr执行copy会出现问题。

9. Choose carefully among erasing options.	


13. Prefer vector and string to dynamically allocated arrays.

比起内置数组，prefer vector和string。
- vector有内置的配套的algorithm，array也有，但是vector用起来更方便
- iterator，begin，end等操作
- 内存管理
- 内部元素的资源回收。

14. Use reserve to avoid unnecessary reallocations.

使用reserver可以避免插入元素造成的内存扩展realloc，以及元素拷贝，iterator invalidate的工作。

15. Be aware of variations in string implementations.


后续的不看了，STL的使用，更多的看源码即可。

























