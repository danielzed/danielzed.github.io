[effective-modern-c++](/assets/ebook/Effective_Modern_C++.pdf)

这本书主要是针对c++11的一些新特性的最佳实践

# type deduction 类型推导

1. 理解template type的deduction

- Case 1: ParamType is a Reference or Pointer, but not a Universal Reference: template 的T会保留类型和const，但是不会保留&和*。另外，如果paramType上写了const，那么T也不会保留const。其他都与直觉符合

- Case 2: ParamType is a Universal Reference：universal reference声明为T&&。 如果argument传入左值，那么T和paramType会被推导成lvalue reference，这也是唯一能推导出reference的一种情况。如果传入右值，那么就是右值的universal reference。

- Case 3: ParamType is Neither a Pointer nor a Reference：在推导T,paramType的时候，传入argument的reference，const，volatile都会被去掉。因为实际上param是argument的一个拷贝，所以类似const这些属性也不重要，不会被传入模板。那么同理，针对const type const *的argument情况，T和paramType会是const type *的类型推导，即指针指向的const被丢弃了，因为指针被拷贝了。但是指针指向地址的const保留，因为拷贝后的指针与argument指针，指向同一个地址。

2. 理解auto的类型推导

auto基本和template的type deduction一致。尤其是上面case2的例子
```
auto&& uref1 = x; #x是左值，auto是universal ref，所以uref1的推导type是int&。
```
唯一区别在于
```
auto a = {1}；#会将a推断成std::initializer_list，而不是int
auto a = {1,2,3}；#也可以正常编译
f({1,2,3})；  就无法编译通过，无法给T和paramType类型推导
```

3. 理解decltype

- decltype在推断类型的时候，基本跟我们想的一致，就是数据的类型
- 当用在模板中时，可以配合auto提供的后置返回类型声明一起使用，后置一个decltype(op)，从而动态返回不同数据类型。
- universal reference是可以同时支持左值和右值的引用类型，需要配合std::forward进行类型转换。
- c++14还有很多高深的用法，比如decltype(auto),c++11暂时不提供。同时c++11不提供自动类型推导，针对超过1个statement的lambda或者func。c++14提供。

4. 知道如何view deduced type

一般通过三种阶段查看deduced 的type，包括IDE，compile以及runtime阶段。

重点将runtime阶段，我们知道typeid或者说std::type_info.name是可以提供类型信息的，但在某些复杂场景，其会丢失比如const或者ref之类的信息。可以使用Boost中提供的工具

# auto

auto是用于类型推断的标识符，除了方便之外，也可以避免手动类型声明带来的一些正确性或者性能问题。同时，auto遵循前面讲到的类型推断原则，但是可能不是用户想要的，如何获得想要的类型，也是个问题。

5. type声明优先使用auto

auto声明类型的好处在于
- 编译时避免默认初始化，如果声明了auto，必须显式初始化才能知道类型
- 更直观，不用写很复杂的类型。
- 避免一些问题，比如函数返回的是long，但显式声明的是unsigned，就会导致精度丢失。以及，map类型返回的迭代器元素里，key是不可变的，是const类型，如果显示声明没有包含const，就会进行转型，同时会拿到临时的拷贝对象而不是原始对象。

6. 当auto推断的类型不是想要的，使用显式type initializer

在某些情况，比如vector<bool>的[]获取的元素类型的auto使用上，会发现auto推导出的类型不是bool，这是因为c++的vector<bool>内部是packed形式，bitmap实现的。

这里涉及了一个概念，proxy class，用于在设计和暴露接口之间起着过渡作用，比如shared_ptr智能指针也算。std::vector<bool>::reference也是。proxy class往往是希望不为用户感知的，但是仔细查看文档或者header文件都可以看到。

针对这种情况，也可以使用auto，同时使用static_cast进行转型，即可使auto推导出正确类型。

# moving to modern c++
这一章主要讲modernc++，即11和14版本带来的新特性，包括智能指针，move，func，lambda，concurrency。如何使用

7. 创建对象时区分()和{}

{}比起()有很多好处，
- 更统一，比起(),=可能有些地方能用，有些地方不能用，但是{}的初始化到处都可以用
- 避免隐式的narrow conversion，比如int a = {1.0 + 2.0}会编译失败，需要显示的将double类型转成int类型，避免丢失精度
- 可以避免c++常见的，定义变量，却变成了定义函数。比如Widget w();会被编译器当做定义一个函数，而不是调用Widget的默认构造函数。

但是{}也存在问题，在涉及到std::initializer_list入参的构造函数时，会优先走这个构造函数。类似用法比如vector v{10,20};是构造一个vector，有两个元素10,20.

8. perfer nullptr to 0 and NULL

因为0和NULL都不是指针类型，都是int类型，所以在某些func的入参解析上会有问题。另外模板类型的推导上，也会有问题。nullptr也不是指针类型，但把它当做指针类型使用不会出现这种不一致。

9. prefer alias声明而不是typedef

两者语法如下
```
type real_type alias_type;
using alias_type = real_type;
```
表达模板的不同：
```
template<typename T> // MyAllocList<T>
using MyAllocList = std::list<T, MyAlloc<T>>;
MyAllocList<Widget> lw; // client code

template<typename T> // MyAllocList<T>::type
struct MyAllocList {
typedef std::list<T, MyAlloc<T>> type;
}; 
MyAllocList<Widget>::type lw; // client code
```
显然，alias using语法的更直观。

10. prefer scoped enum to unscoped enum

永远使用enum class的枚举类型定义而不是enum。
- enum class定义，可以限制枚举变量的可见域，只有enum::var才能引用，内部才会重名。
- enum class变量编译期会做类型检查，不会implicitly的type conversion成int或者flog

11. prefer deleted func to private undefined func

deleted比起private undefined func有几个好处
- deleted的方法谁都调用不了，private的方法，friend和derived class还能调用
- 针对隐式转换场景，比如func接收int，但是传入float，此时会自动转型。如果想避免这种情况，可以将对应func的float版本声明为deleted。
- 针对模板方法，某些类型场景下，显式deleted，可以禁止模板传入这种类型

12. override的func要声明override

override的好处，可以在编译时检查应该override的dericed class的func，是否真的override了base class的func，即他们的参数，函数名，const以及&或者&&(这个功能，是用来标识，返回值是左值还是右值的，从而避免拷贝)是否相同。

13. prefer const_iterator to iterator

iterator的本质就是读，而不是修改，所以一般const_iterator更实用。c++11提供了cbegin()和cend()接口快速获取const_iterator

14. func如果不抛出异常，声明noexcept

noexcept声明不抛出异常，会对性能进行优化。但是不要强行让func不抛出异常。一般destructor或者delete都是隐式noexcept的，如果抛出异常，undefined behavior。

15. constexpr whenever possible

- constexpr修饰变量的时候，是说它的值可以在编译期确定。和const不同，const的值可能要runtime才能得到，但是不能改变。
- constexpr修饰func的时候，是说如果func的入参都是constexpr，那么func返回值也是一个constexpr，编译期确定，可以传给array作为长度参数的那种。如果入参不是constexpr，那么就是一个普通函数。

16. 让const member func 线程安全

cosnt member func有几个特性
- 不能修改class成员
- 返回值是const的
- 不能调用非const的member func

mutable member成员提供特性
- 可以在const member func中修改

那么问题来了，如果在const func中执行了写操作，client代码认为const func是不用外部同步的，那么就会出现data race，导致数据不一致。所以const func应该thread safe

17. 理解特殊member func的生成

c++11的默认函数，增加了与copy func对应的两个，是move func。move func如果用户提供了任意一个实现，那么另一个就不会产生。如果提供了move func的实现，copy func不会产生，反之亦然。move func传入右值，返回左值引用。

# 智能指针

raw-pointer有很多问题，不知道指向的是object还是array，不知道如何释放，是delete还是有其他释放函数。不知道指针是否是dangle，是否被释放过。

智能指针解决了这个问题，有4种类型，auto_ptr是c++98的遗留，可以由unique_ptr替代。shared_ptr和weak_ptr。引用次数上有区别。

18. 使用unique_ptr完成独占资源的管理

- unique_ptr与raw_ptr比较接近，没有mem或者计算上的额外开销(除非指定deleter)。
- unique_ptr是对资源进行独占，没有copy，只能move。赋值之后，旧的指针变为nullptr。很适合作为工厂方法的返回值。同时可以在工厂方法内指针deleter完成定制化清理。
- unique_ptr针对object和object[]是不同的定义，所以不会混淆。
- c++11不支持raw-ptr和智能指针的implicitly转义，必须显式声明。

19. 使用shared_ptr完成共享资源的管理

- shared_ptr大小是raw_ptr的两倍，每构造对象的一个shared_ptr，内部计数会+1，取消指向则会-1.如果为0时，调用deleter，可以指定deleter。
- 引用计数，weak计数以及deleter等信息，存放在control_block上，shared_ptr有指针指向它。control_block动态申请。
- std::move移动shared_ptr的时候，引用计数不会变化。

智能指针的control_block是有开销的，同时注意，不能针对raw-pointer创建多个shared_ptr，会导致control-block创建多份，从而每个control-block计数为0时都会delete。所以最好，
- shared_ptr的构建使用new表达式，而不是raw-ptr，避免别人用同样的raw-ptr去构建
- 或者，使用make_shared来创建shared_ptr，这样就会复用control-block
- 要分析共享资源逻辑是否成立，优先使用unique_ptr，因为unique_ptr可以转换成shared_ptr，反之则不行。
- 避免用shared_ptr来管理T[]，不支持。

20. 针对shared_ptr可能dangle的场景，使用weak_ptr

weak_ptr是shared_ptr的一种增强，在构建和删除的时候不会进行引用计数的加减。所以weak_ptr指向的指针，可能已经被shared_ptr删除过期，所以weak_ptr可以知道自己指向的object是否有效。

- 在一些指针互相指向的场景，比如树的parent和child，如果parent用unique_ptr指向child，那么child可以用raw-ptr指向parent。因为只要parent被释放之后，child就不会有人引用到，那么即使raw-ptr dangle也不会有问题。
- 在观察者模式中，subject主题会有observers的指针，但是subject又不应该影响observer的引用释放，这里应该用weak_ptr，如果指针无效则不需要通知对方。

21. 智能指针创建比起new，优先用make_shared，make_unique(c++14)

c++11版本的make_unique可以自己实现一套模板
```
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```

make接口的好处在于
- 只需要写一次类型，new再转成shared_ptr需要两次
- exception-safe，在method-arguments场景下，op顺序可能被打乱。new再转shared可能拆成两步，从而导致new了但是没有转shared，内存泄漏。尽量单独写一个statement
- 性能考虑，make只申请一次内存(object + ctrl-block),而new的方式，两次申请。

但new-shared也有好处
- 可以指定deleter
- weak_ptr的场景，只要object还有weak_ptr指向，那么ctrl-block就不能释放。因为make方式，ctrl-block和object是一起申请的，所以两者都不能释放。new-shared可以释放object的内存。

22. 使用pimpl idiom时，在impl文件中定义特殊member func

pimpl：pointer to implementation，将data member换成对impl实现类的pointer。把之前在primary class里的成员，放入impl类中封装。从而避免修改造成的连锁编译问题。

一般impl成员使用unique_ptr来表示，从而可以在外部对象创建时申请，外部对象释放时释放。同时注意，因为impl类是incomplete type，即只有声明，没有定义。

使用pimpl模式的时候，需要在类中定义copy func，dtor等避免compiler生成的会调用static_assert检查指针是否为incomplete type从而抛异常。

# 右值引用，move语义，perfect forward

23. 理解move和forward

- move并没有移动任何byte，只是cast成rvalue。如果传入的值带了const，往往会调用copy func，而不是move func，从而还是有拷贝发生。
- forward 有条件的cast成rvalue，当传入的value也是rvalue的时候。理论上forward可以完成move的任何功能。但是move本身很简洁，不用传模板T进来。

24. 区分universal ref和rvalue ref

- 当在变量声明或者模板方法中用到auto&&或者T&&，并且涉及到类型推导的时候（就是说type不是提前就知道了的），那么变量就是universal ref
- 如果类型提前知道了，那么type&&的格式就是rvalue ref
- universal ref是可变的，当传入lvalue，就是lvalue ref。传入rvalue，就是rvalue ref。

25. 在rvalue ref上使用move，universal ref上使用forward

- 如果在lvalue上使用了move，会把原来lvalue的值清空。在某些情况下不是预期。所以只应该针对rvalue使用move，针对可能是lvalue ref的情况，使用forward。

- 针对可能lvalue，可能rvalue的情况，也可以实现函数的两个实现，接收不同左值右值参数。但这样，随着参数的增加，需要的函数实现指数级上升。

- rvalue ref其实还是一个lvalue，当需要转换成rvalue的时候，需要move转换。

- 不要对func返回的local variable使用move或者forward，会影响return-value-optimization

26. 避免在universal ref上overload重载

- 如果一个func接收universal ref作为参数，那么最好不用重载，会造成解析错误，导致调用了错误的函数。

- perfect forward的ctor也很可能造成问题，因为对齐non-const的value传入来构造对象，本来应该调用copy func的，但是perfect forward的ctor，因为不用增加const属性，更容易被调用。

27. 熟悉下可能overload universal ref的情况

忽略

28. 理解ref collapse

- ref collapse出现在模板定义，auto，以及alias，typedef的场景
- 当出现ref to ref的情况，如果有一个ref 是lvalue，就会collapse成一个lvalue的ref

29. Assume that move operations are not present, not cheap, and not used.

move不是任何时候都比copy’好。有些object没有提供move方法，有些move更差，有些不能用move。

30. 熟悉完美转发可能失败的场景

忽略

# lambda表达式

介绍lambda匿名函数以及closure闭包

31. 避免默认capture mode

闭包的capture包括两种，by-ref和by-val。
- by-ref：&variable或者单独的&，即将变量引用导入闭包。不要使用默认的，避免引用了局部变量，然后闭包调用的时候变量超出lifetime
- by-value：=或者声明variable。即按value拷贝到闭包里。注意不要默认，也会因为拷贝指针，而指针被释放，导致dangle。同时static的不用拷贝，闭包可以直接看到。
- 针对对象的member，不能直接capture，而是要capture 对象this，传入闭包，然后使用对象调用member。

32. Use init capture to move objects into closures.

c++14才有的特性，在closure的capture，支持move，可以减少性能开销。

33. Use decltype on auto&& parameters to std::forward them.

忽略，也是c++14的。

34. Prefer lambdas to std::bind.

lambda表达性更好，更可读。bind可以用来实现move capture的功能。

# 并发api

35. 比起thread编程，prefer task编程

使用std::async可以把要执行的func作为task来看，不适合线程绑定。同时返回的future接口提供get方法可以阻塞获取接口，也可以获取执行过程中的异常。

而thread编程，在软件thread和硬件thread的并发之间很难调优，同时thread中的异常很难获取。以及一些thread-cache，上下文调度上，开销都比较大。而async的task编程，将这些都交给标准库完成，只提交task给thread-pool，thread间的调度，均衡由标准库完成。

36. Specify std::launch::async if asynchronicity is essential

async支持两种调度模式，async和deffered，
- async即调度到其他thread上异步执行。
- deffered即推迟，直到对future调用get或者wait时才同步执行，执行完才结束get或者wait方法。

默认的调度策略是async|deffered,即随机表现，无法预测。这样会造成很多问题，所以异步task要显式声明策略。

37. Make std::threads unjoinable on all paths

忽略

38. 后面的部分，由于对c++11特性不太熟悉，暂时跳过，看完rocksdb代码再回头来看

39. Consider void futures for one-shot event
communication

针对线程间需要通知的情况，比起condvar通知，可以使用promise和future的通知逻辑，类似go的channel。promise设置某个值，future的get或者wait是阻塞获取的。

因为condvar一般要配合锁来使用，修改共享变量

另外也有轮询死等判断flag的方式，效率更差。

40. 并发使用atomic，特殊内存使用volatile

- atomic：用在并发多线程访问一个变量中，可以避免编译器reorder以及machine的reorder，避免线程间通信的一些异常状态，比如标记位翻转了，但相应的操作因为reorder问题，还没有执行。并且对应变量的操作，比如Read-Modify-Write是原子的，不会在一个thread修改同时，另一个thread看到中间状态。

- volatile：针对特殊内存，需要声明volatile，比如文件mmap，或者外设的控制器bit对应内存。类似redundant load和deadly store，即反复从一个位置上读，或者反复向一个位置上写。正常来说，编译器会将这种情况进行优化，但在对特殊内存操作时，是不应该优化的，因为可能是在向网络或外设发消息或读消息。

# tweaks

41. Consider pass by value for copyable parameters that are cheap to move and always copied.

并不是任何时候，传ref都比传value更好。
- 传ref，针对左值，右值都可能的情况，需要定义两个版本。或者一个带template+universal ref的版本。会增加代码量
- 传value在c++11中，并不都是调用copy func，在入参是rvalue的情况下，是调用move func的，此时cost并不一定很大。

但也只是考虑，用value比ref，会有很多限制要求，大多数时候还是用ref

42. Consider emplacement instead of insertion

在向container中插入元素时，如果使用insert插入比如string lateral，那么会先构造一个tmp string























