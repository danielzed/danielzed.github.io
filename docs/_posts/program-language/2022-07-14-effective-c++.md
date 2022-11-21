[effective-c/c++](/assets/ebook/Effective_Modern_C++.pdf)




# constructor,destructor
5. 

类中的constructor，copy constructor，copy assignment，destructor4个方法，如果本身未定义，编译器会自动生成。

其中copy assignment以及copy constructor，会将入参的所有非static成员拷贝给左值对象。如果其中包含引用类型，则必须自定义copy assignmenet，否则编译器会报错。

6. 不提供copy func的方式

某些场景，不能提供copy constructor或者copy assignment，比如某些现实中的东西，是不存在两份一样的。这时，通过主动定义copy func保证不会自动生成默认，同时设置访问权限为private，保证用户不会调用。但这样，friend 类还是可以调用，通过不提供实现的方式，在link阶段报错。
c++应该是有delete标识符吧，这是c++11带来的特性

7. virtual destructor

c++有一个特性，当base class的pointer，指向一个derived class的object的时候，这个pointer在release的话,如果base class的destructor是实现了的，那么就会出问题。独属于derived class的对象就不会被释放。除非base class的destructor是virtual的。

但是方法定义为virtual，本身是会有内存开销的，会有一个vptr的指针占用。所以virtual不能无限制的使用。一般来说，class中只要有一个virtual的func，就会把destrcutor定义为virtual。

如果一个class，想作为base class被别人继承的话，一般也是定义destructor为virtual。

8. dont emit exception in destructor

因为destructor中如果抛出异常，程序没有办法处理，比如局部变量的对象，在退出程序块的时候自动释放，此时很难说抛出异常会如何进行处理。

常见的解决方式包括，退出程序abort，打印日志swallow exception，或者释放本身由资源对象的调用者传入cb完成，这样可以将destructor的责任交给资源使用者。

9. destructor和constructor中不要调用virtual函数

原因在于，base class的dest/cons中如果调用了virtual函数，我们希望的可能是调用derived版本的对应func，但实际上，因为derived对象的const还未完成，或者dtor已经完成，导致其derived的member是无效的，所以对应func也无法执行，所以会执行base 版本的func。

建议方法是，在derived class的constructor中，传给base class的construtor对应参数，完成初始化。

10. 一个传统，copy assignment要返回一个对象引用

类似x=y=z=15的情况，=号也就是assignment是右结合的，z=15的结果被赋值给了y，原理就是z=15的返回值是z的引用。这是一个约定俗成

11. 注意copy assignement中，左值和右值是同一个对象的情况

在一般的copy assignment中，我们可能会先将左值的资源进行释放，然后将右值的资源赋值申请给左值。但在两者相同对象的场景，资源会被释放无法使用。另外，如果资源重新申请过程中出现异常，资源也会丢失。所以需要保证，在左值资源申请出来之前，旧的资源不会被释放。

12. copy func要拷贝所有成员

一方面，在class增加成员后，copy func要注意适配。另一方面，在derived class中，注意copy func要调用base class的copy func，否则base class的成员不会被拷贝。

另外，还有个约定，copy constructor不能调用copy assignment，即使代码很多重复。

# 资源管理

资源管理问题常见的是内存泄漏，除此之外还有很多资源，包括文件句柄，网络连接，数据库连接，mutex等。资源管理需要在不用的时候进行释放，但是结合异常处理，复杂的逻辑路径就会产生很多问题

13. 使用对象来管理资源

资源永远不要手动释放，因为异常处理，代码迭代总是会使得架构变化。依赖对象的destructor方法完成，

- unique_ptr指针，当指针超出可用域时，自动delete指向的对象。unique_ptr只能有一个ref，不能拷贝，如果拷贝，src就会变成null。那么在结合container使用就会有问题，因为不能拷贝

- shard_ptr，引用计数指针，可以拷贝，当引用计数变为0时释放对象。

但这两种方式也不是最佳，因为开发者还是可能会忘记使用智能指针，最好的方式是返回得到的就是智能指针。

14. RAII对象拷贝时要注意

RAII对象，即constructor中申请资源，destructor中释放资源的对象。在其拷贝时，一般有如下几种情况
- 不支持，针对某些只应该有一个的对象，比如LOCK
- 拷贝，内部资源深度拷贝，自己管理自己的
- 拷贝，内部资源使用shared_ptr，通过引用计数管理。shared_ptr可以设置deleter，在释放时执行诸如unlock的定制释放func
- 转移资源所有权，旧的对象不再持有资源，类似unique_ptr

15. 在RAII中提供对raw资源的访问

比如RAII对象，我们拿到shard_ptr的指针，但是某个函数入参需要其*指针，针对这样的问题，我们有两种方式
- 显式：提供get接口，获得内部资源，比如shard_ptr.get()
- 隐式：class中提供一个operator resource-name()的方法获取其中资源，但隐式转换很容易出问题。

16. new和delete的格式要一致

主要讲的是，new和delete在针对单个对象和array的时候，要采取一致的方式。带[]或者不带[]

- new做的事情包括申请内存，调用constructor
- delete做的事情包括调用destrcutor，释放内存
针对数组情况需要new []和delete []。array关键是要知道数组size，delete的时候如果指定了[]，那么在调用destructor之前会检查size，再做释放。

同时，针对typedef的类型，也是一样。如果将array类型typedef成其他，还是要用delete[]释放。所以建议array类型不要typedef，或者用vector

17. 智能指针用的时候，必须单独使用

因为智能指针的new raw指针，以及赋值给智能指针，是两个步骤，如果和其他流程在一起，可能由于其他流程的异常，导致只执行了new，没有执行赋值，从而导致资源泄漏

# designs and declarations

18. 让接口容易用对，不容易用错

在item13的基础上，当时client需要手动将raw ptr转成shard_ptr，这样client还是可能用错接口。所以接口可以直接返回shard_ptr，同时可以shard_ptr初始化时传入deleter，避免client不知道怎么delete。

智能指针的开销，更多的内存，以及多线程访问时的同步。

19. class的设计要像原生type设计一样

针对class的func，使用场景，要考虑周全。从整体的类型结构设计上，以及具体的资源管理，构造，释放上。

20. 用const的reference代替value类型传给func

- reference内部使用指针，c++的func传参，都是传的value，就是形参会调用copy constructor生成实参。使勇reference可以减少constructor以及destructor，以及copy的开销
- 使用const，避免func内部修改参数对client的影响
- copy constructor会导致类型被slice掉，比如client传入的是derived class的对象，参数要求是一个base class的value，那么中间就会进行copy constructor，就会导致对象的virtual func调用出现问题，想要执行derived class的func，实际执行了base class的virtual func

21. 返回结果，不要用reference

即使ref可以减少constructor的调用，但是在func 的返回结果上，做不到，使用reference只会更差。

22. data member要private权限

data memeber不要直接设为public或者protected，
- 统一api，所有class的调用都是func
- 细粒度授权，某些data member不是所有人都应该访问
- 代码迭代，如果member为public就没有改的空间了，设置为func，内部实现可以不断变化演进。

23. 偏好使用non-member以及non-friend的func

对于data member来说，class中能访问他的member-func以及friend-func越多，则其封装性越差，暴露给外面的接口就越多。这点不是很理解

24. 如果一个方法，参数需要type conversion，那么最好不用member func

因为如果是member-func，与类的耦合更严重了。除此之外，会导致member-func的入参在转型时存在问题。这里的原因没看懂，例子是2*ration与ration*2，后者编译通过，前者失败，在ration中提供operator*的时候。

25. swap方法，在某些场景下很没必要

这个也没有很看懂，太长了
- 可以针对template自定义swap方法，要保证不会抛出异常，因为两个相同对象交换，可能会丢数据
- 如果自定义swap，命名空间需要指定

# implementation

26. 尽可能迟的构建对象

由于异常处理和流程控制上的变化，有些对象可能完全不会用到。尽可能在用到之前去构建这些对象，避免无效开销。同时，等到所有构建对象的条件都准备好，再去构建，而不是null ctor然后assignment。同时针对循环场景，需要比较ctor和dtor与assignment的开销差异，选择在loop内还是外声明对象。

27. minimize casting

cast是类型转换的一种，在现有类型基础上，增加了漏洞的可能，也增加了类型使用的灵活性

- old-style ： (T)name或者T(name)
- new-style: 通过限制cast的种类，从而限制使用者的能力，减少问题
	- const_cast 去除变量的const属性
	- static_cast 强制隐式的类型转换，比如int到double
	- dynamic_cast 安全的向下类型转换
	- reinterrept_cast low_level层面的类型转换，较少使用

尽可能少使用转型，尤其dynamic_cast。对性能有影响。大多数转型可以通过base class声明virtual func，或者保存derived class的指针避免。

28. func避免返回handle(比如引用，指针)

- 返回handle，会造成外界对内部成员有读写权限，这可能不是封装想要的结果
- 即使给返回的handle const修饰，外界可能拿到这样的引用并保存了下来，但是对象随时可能释放，那么内部member的handle随时可能无效，保存的值就变成了dangle的无效值

29. 针对异常也能正确处理的函数

异常是不可能避免的，比如内存申请失败，api调用问题。要保证不会出现资源泄漏以及成员变量值dangle无效。

常用的方式是copy and swap，生成一个旧值拷贝，针对拷贝修改，然后交换。同时使用智能指针，RAII封装实现资源管理。

30. 使用内联函数的注意事项

- 内联函数也是有开销的，会使得程序过大
- 只针对哪些频繁调用的简单函数使用内联函数。诸如ctor和dtor，看起来很简单，但实际里面会做很多初始化，异常处理的工作，不能用内联函数
- 内联函数是编译时替换，所以定义要在header文件里。

31. 尽量依赖declaration而不是defination，减少文件间依赖

为了避免对一个class的修改，所有class都重新编译，尽可能彼此间减少依赖，只依赖对方对外暴露的接口，而不是实现。

比如头文件里有class的private成员，这些其实属于实现，不需要对外暴露。那如果这些private成员的实现改了，所有引用了头文件的class都要重新编译。

- 可以通过interface类和handler类解耦的形式解决。
- 或者将成员对象定义为指针，而不是对象，这样不需要对方的完整定义也可以declaration

# 继承和面向对象设计
c++的继承支持多个继承，同时支持public，private，protected继承，另外函数也包括正常函数，虚函数，纯虚函数。

32. public继承代表is-a的关系

public继承，即代表所有baseclass能做的事情，derivedclass都能做，这需要很严格的检查，类似长方形和正方形都是不满足的，因为他们有些性质是不一样的。尽量使用has-a的方式来表达类的关系。

33. 不要隐藏public继承来的那些func

就是说，derived中不要通过定义与base中相同名字的函数，使得base中func无法找到(即使函数签名不同也无法找到)。此时需要在derived中使用using主动声明才可以。

34. 接口继承和实现继承的区别

- 纯虚函数，是希望子类只继承接口
- 虚函数，是希望子类继承接口，同时默认实现
- 非虚函数，则是继承实现，并且不允许改变

35. 考虑虚函数的替代

除了虚函数之外，还有很多更灵活的方式，比如策略模式，传入算子完成策略的定制，从而根据不同场景调用不同实现。

或者模板方法，根据不同模板参数，执行不同策略。

36. 永远不要重定义继承来的非虚函数

37. 永远不要修改集成来的默认参数值

这两个都是会造成，用base pointer操作时的一些异常，不一致问题。如果需要重载，那么就用virtual方法。

38. 推荐使用has-a的模型来构建class关系

39. 明智的使用私有继承

私有继承就是没有继承接口，但是继承了实现。类似has-a的成员关系。无法使用向上转型。

那么比起成员，什么时候使用private继承呢？必须用的时候，比如要访问protected成员，或者virtual方法。

40. 明智的使用多重继承

这东西，没必要用吧，什么设计会用到多重继承，一定是垃圾设计。

# template和generic program

41. 理解隐式接口和编译时多态

使用模板方法实现类似go的那种duck-type的功能。就是，你具有某种功能，比如swap或者calculate，就可以传到这个func的template中，我func就可以执行。

42. 理解typename的两种含义

模板方法中，class和typename标识符基本一致，但是某些场景下不一致，叫做nested dependent type names。

43. 在template base class中，如何访问names

44. 模板方法在消除代码重复的同时，也会带来代码重复

因为模板方法，其实是编译时做事情，会根据模板的不同，生成多个方法定义。有些非类型模板，比如本应通过入参传进来的，却通过模板参数传进来，就会造成方法定义冗余很多份。

45. 使用member func template接收所有compatiable的type

46. 当需要type conversion时，在模板内部定义非成员方法

47. 使用trait类型，标识type中的信息

c++STL中，各种数据结构支持迭代器方法，迭代器共包含5中，input_iter,output_iter,forward_iter,backward_iter,random_access_iter。分别input,output,forward支持+1的迭代，backward支持正负的迭代，random_access支持+n的迭代。在某些模板方法比如advance，前进n步的实现中，需要根据iter支持不同类型的迭代，不同实现。这就需要在编译时或者运行时知道iter的具体类型。

方式就是，通过在类中定义一个trait class，然后在advance中获取类的trait信息，传给doAdvance方法，doAdvance实现多态方法，根据不同iter类型，实现不同的迭代方式

typeid关键字

48. be aware of template metaprogram

item47中介绍的两种方式，一种使用typeid在运行时根据不同iter类型不同实现，一种使用编译时多态，生成多个方法，对应iter的实现。第一种方式因为编译器的保证（某些代码即使不会走到，也要有效）可能编译失败。

metaprogram即使用代码，在编译器生成代码，可以使运行时减少开销。

比如fabonacci数列，使用recursive template，生成n个模板，在编译期间保存对应的值。

template metaprogram也叫做TMP，有着很多应用，但比较高深。

# customize new and delete
new,delete,new[],delete[],new-handler(申请内存失败的处理函数)。以及多线程时，因为heap和new-handler都是共享空间，会造成的一些问题。

c++比起java的优势在于内存精细的管理，减少time和space的开销。

49. new-handler的行为

c++支持set_new_handler的方式，将一个无入参，无结果的func传入作为new失败的handler。可以在其中实现异常通知，日志，释放内存的做法

当new失败找不到内存是，会不断的调用new_handler直到内存申请成功，或者程序异常退出。

50. 理解replace new和delete的时机

在某些场景下，替换new和delete，使用其他版本是有意义的。比如
- 定位内存问题，内存重复delete或者内存泄漏，在new的时候list中插入元素，delete的时候释放元素，从而知道哪里有问题
- 定位内存越界，在new的地址前后加上magic，在越界时会导致magic信息丢失，从而定位问题
- 内存使用统计
- 更好的性能

注意自己实现new和delete是很简单的，但同时也很可能出现问题。常见的包括alignment，以及需要重复调用new_handler等。可以参考开源内存管理器

51. 当重写new和delete的时候，adhere to convention

应该考虑require zero bytes，循环malloc获取内存，循环new-handler处理释放内存情况。

52. 当你替换new的时候，也要替换delete





























