[mockito官方文档,很清晰](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html#1)
1. mockito的用处
- verify接口验证某个mock对象，是否调用了某个函数。此时不需要提供mock func的定义，还是走的原始函数。可以指定校验方法出现的次数verify(obj, times).func()
- 定义方法行为：when(mockobj.func()).thenReturn。当mock对象调用某个func时，不实际调用，而是直接定义返回值。可以通过mock(class名)产生mock obj,也可以在obj声明前加上@Mock的注解。

	注解成员变量初始化（包括@Mock,@Spy,@InjectMocks），需要调用MockitoAnnotations.initMocks初始化，或者在junit测试类前，添加@RunWith(MockitoJUnitRunner.class)。
	
	存根连续调用：thenReturn中注册多个值，代表按照顺序，每次调用返回不同的结果。
	
- 校验func调用顺序：使用InOrder对象

- spy：mock的升级版，针对注册了存根的方法，则调用存根返回结果。没有存根的，则直接调用read func定义。也就是局部mock。

	使用的时候注意，spy对象when注册存根的时候，无法起作用。需要doReturn完成。
	
- mockito的用例开发框架：given-when-then

2. mockito的局限：
- 不能mock静态方法
- 不能mock 构造器
- 不能mock equals和hashCode方法