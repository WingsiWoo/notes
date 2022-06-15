# 面试-Spring

## 生命周期/IOC容器加载过程

1. ==实例化==（Instantiation）
2. ==属性赋值==（Populate）
3. ==初始化==（Initialization）
   1. 初始化前：
      1. 检查 Aware 相关接口并设置相关依赖(BeanNameAware、BeanClassLoaderAware、BeanFactoryAware)
      2. BeanPostProcessor 前置处理
   2. 初始化中：
      1. 若实现 InitializingBean 接口，调用 afterPropertiesSet() 方法
      2. 若配置自定义的 init-method方法，则执行
   3. 初始化后：
      1. BeanPostProceesor 后置处理
4. ==销毁==（Destruction）
   1. 若实现 DisposableBean 接口，则执行 destory()方法
   2. 若配置自定义的 detory-method 方法，则执行



## 什么是控制反转

IOC是一种解耦的设计思想，它借助第三方实现具有依赖关系的对象之间的解耦。如对象A依赖对象B，没有引入IOC之前，对象A就需要自己去创建对象B（主动）；但是引入IOC之后，对象A向容器请求对象B即可，对象B的创建由容器来进行管理（被动），由主动转换为被动，实现了控制权反转

优点：

1. 实现了上层类对下层类的控制

   假设有一个类`Framework`和一个类`Car`，其中`Car`依赖`Framework`

   ```java
   public class Car {
   	private Framework = new Framework(int size);
   }
   
   public class Framework {
   	private int size;
   	public Framework(int size) {
   		this.size = size;
   	}
   }
   ```

   `Car`通过调用`Framework`的构造方法主动创建`Framework`，如果`Framework`的构造方法进行了修改，需要传递别的参数，那么`Car`也要跟着修改。在一个大的项目中可能会有多层的依赖/继承关系，一个底层类的修改可能引起很多个上层类的修改

   引入IOC后，假设`Car`采用构造器注入方式，直接把`Framework`对象作为`Car`构造器的参数，由于`Car`没有主动调用`Framework`的构造器，因此当`Framework`的构造器发生变化时Car的代码无需修改

2. 可实现热插拔

   当一个接口有多个实现类的时候，可以通过修改`xml`或者`@Resource`注解指定实现类类名的方式来指定使用哪个实现类，而无需修改代码

3. 便于单例的管理

   对于像`Service`、`Dao`这种类，实际上只需要一个实例对象就够了。如果是自己主动创建的话，如果该`Service`再多个地方被引用，就会被多次创建。`Spring IOC`默认使用单例模式管理对象，在创建对象之前会先到容器中查找是否已经创建过该对象，如果已经创建过就直接返回，保证了对象的单例性。



## 循环依赖解决过程

A依赖B，B依赖A，Spring是怎么解决循环依赖的？

单例对象的创建分为三个步骤：

1. `createBeanInstance`：调用对象的构造器实例化对象
2. `populateBean`：填充属性
3. `InitializeBean`：初始化`Bean`

整体步骤：

1. A查询缓存后发现没有创建过，先进行A的实例化，并把其提前暴露到三级缓存`singletonFactories`中

2. 然后进行A的属性填充，发现其依赖B，由于三级缓存中都没有B的存在，因此转去创建B

3. 进行B的实例化，并将其提前暴露到三级缓存`singletonFactories`中

4. 然后进行B的属性填充，发现其依赖A，先到一级缓存`singletonObjects`中查找，找不到的话到二级缓存`earlySingletonObjects`中查找，还找不到的话到三级缓存

5. 发现三级缓存`singletonFactories`中有A的存在，因此B拿到A的提前引用，顺利走完后续创建步骤（创建完成后存放到一级缓存中）

   > 从三级缓存中获取都是`ObjectFactory`对象，而通过它创建的实例对象每次可能都不一样的，因此需要升级到二级缓存
   >
   > **那三级缓存中为什么不能直接存放实例对象而是`ObjectFactory`对象呢？**
   >
   > 这是因为如果想进行增强，直接使用实例对象是行不通的。如果B发现A存在AOP代理的话，那么B依赖的应该是代理对象，而不是原始的`ObjectFactory`，B会调用`ObjectFactory.getSingleton()`方法获取到代理对象后，把A从三级缓存中移除，放入二级缓存

6. B创建完成后，A也可以继续走后续创建步骤了

![image-20220109142654878](https://tva1.sinaimg.cn/large/008i3skNgy1gy7ekg7t6jj30kd08i760.jpg)



### Q：二级缓存就可以解决循环依赖问题，那么为什么`Spring`还要采用三级缓存？

A：初始的`Spring`是没有解决循环引用问题的，设计的原则是**bean实例化->属性设置->初始化->生成aop对象**。

如果不采用三级缓存的话，生成`aop`对象的这个阶段就要提前到实例化的阶段执行，也就是说所有的`bean`在创建过程中都会先生成代理对象再设置属性等其他工作。

但是这样做的话，就与`Spring`的`aop`的设计原则相驳：`aop`的实现需要与`bean`的正常生命周期分离。因此，`Spring`选择再增加一级缓存，当发生循环依赖时，就触发代理类的生成，并把代理类存入一级缓存（`singletonObjects`）

> 去掉三级缓存之后，Bean 直接创建 `earlySingletonObjects`， 看着好像也可以。
>
> 如果有代理的时候，在 `earlySingletonObjects` 直接放代理对象就行了。
>
> 但是会导致一个问题：在实例化阶段就得执行后置处理器，判断有 `AnnotationAwareAspectJAutoProxyCreator` 并创建代理对象。
>
> 这么一想，是不是会对 Bean 的生命周期有影响。
>
> 同样，先创建 `singletonFactory` 的好处就是：在真正需要实例化的时候，再使用 `singletonFactory.getObject()` 获取 `Bean` 或者 `Bean` 的代理。相当于是延迟实例化。



## AOP

### AOP实现方式

- 静态代理：指使用AOP框架提供的命令进行编译，从而在编译期就生成代理类，因此又称为编译时增强
  - 编译时编织（特殊编译器实现）
  - 类加载时编织（特殊的类加载器实现）
- 动态代理：在运行时在内存中临时生成AOP代理，因此又称为运行时增强
  - JDK动态代理：**通过反射来接收被代理的类**，并且**要求被代理的类必须实现一个接口**，核心是`InvocationHandler`接口和`Proxy`类
  - CGLIB动态代理：如果目标类没有实现接口，`Spring AOP`就会选择CGLIB来实现代理。CGLIB是一个代码生成的类库，它**通过继承的方式来接收被代理的类**（因此**要求目标类必须是可继承的**，如果被标记为`final`，就不能采用CGLIB）



### Spring AOP与AspectJ

|        Spring AOP        |                    AspectJ                    |
| :----------------------: | :-------------------------------------------: |
|       基于动态代理       |                 基于静态代理                  |
| 仅支持方法级别的PointCut | 提供了完全的AOP支持，还支持属性级别的PointCut |



## 使用到的设计模式

### 工厂模式

- `BeanFactory`：延迟注入（需要使用的时候才注入），程序启动速度快，可能会引发NPE
- `ApplicationContext`：立即注入（程序启动时就把对象立即创建并注入），程序启动速度较慢，但可以在项目启动的时候就能发现配置错误



### 单例模式

Spring中bean的作用域（`scope`）默认为`singleton`，即交给Spring容器管理对象默认都是单例对象（通过`ConcurrentHashMap`单例注册表实现）

其他的作用域：

- `prototype`原型：每次都会创建一个新的实例对象
- `request`：每一次HTTP请求都会创建一个新的对象，该bean仅在当前`HTTP Request`有效
- `session`：每一次HTTP请求都会创建一个新的对象，该bean仅在当前`HTTP Session`有效



### 代理模式

即`Spring AOP`，`Spring AOP`采用动态代理模式，如果需要被代理的目标类有实现接口，就采用JDK动态代理（反射）；如果没有就采用CGLIB动态代理，生成一个目标类的子类作为代理类

![img](https://img-blog.csdnimg.cn/20190716234116351.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoYW84MjE=,size_16,color_FFFFFF,t_70)



### 模板方法

> 所谓模板，就是在父类中定义主要流程，而把一些个性化的步骤延迟到子类中去实现，父类始终控制着整个流程的主动权，子类只是辅助父类实现某些可定制的步骤。 

如`JdbcTemplate`、`HibernateTemplate`等数据库操作模版。

一般模版方法采用继承的方式实现，但是Spring采用`Callback`的方式实现：

如`StatementCallback`接口内定义了一个方法`doInStatement()`（钩子方法），在这里就无需考虑具体的实现，而是交由数据库模版其内实现子类来实现。

如`JdbcTemplate`中的`update()`方法就定义了一个实现类`UpdateStatementCallback`，其实现的`doInStatement()`的内容是JDBC更新数据库的操作，然后把该实现类传给`execute`方法进行回调，`execute`方法再调用`doInStatement()`方法对数据库进行操作

**为什么采用回调方式？**

对数据库有多种操作方式，如果全部都在父类中定义为抽象方法，那么继承的子类就必须要全部实现，这样就导致子类非常臃肿。采用回调方式的话，子类就可以只定制某个功能的方法，定义内部类实现回调函数，然后把该内部类作为参数传给`execute`方法，由`execute`方法调用回调函数对数据库参数。



### 观察者模式

![image-20220110133713089](https://tva1.sinaimg.cn/large/008i3skNgy1gy8ir0avhxj30fs09rglu.jpg)

- 是一种对象间的一对多关系
- 当目标发送改变（发布事件），监听者（观察者就可以收到）
- 监听者如何处理，目标无需干涉，它们之间的关系是松耦合的

Spring的事件驱动模型由三部分组成：

- **事件**：`ApplicationEvent`，继承自JDK的`EventObject`，所有事件将继承它，并通过`source`得到事件源。
- **事件发布者**：`ApplicationEventPublisher`及`ApplicationEventMulticaster`接口，使用这个接口，我们的`Service`就拥有了发布事件的能力。
- **事件订阅者**：`ApplicationListener`，继承自JDK的`EventListener`，所有监听器将继承它。



### 适配器模式

`SpringMVC`中的适配器`HandlerAdapter`，根据`Handler`规则执行不同的`handler`

实现过程：

1. `DispatcherServlet`根据`HandlerMapping`返回的`Handler`，向`HandlerAdapter`请求`handler`
2. `HandlerAdapter`根据`handler`规则找到对应的`handler`（即`Controller`）并执行
3. 执行完毕后`Handler`会向`HandlerAdapter`返回一个`ModelAndView`
4. 最后由`HandlerAdapter`向`DispatchServelet`返回一个`ModelAndView`。



### 装饰者模式

装饰者设计模式可以动态地给对象增加些额外的属性或行为（例：配置`DataSource`）



### 策略者模式

Spring框架的资源访问`Resource`接口。该接口提供了更强的资源访问能力，Spring 框架本身大量使用了 `Resource` 接口来访问底层资源。

Spring 为 `Resource` 接口提供了如下实现类：

- `UrlResource`： 访问网络资源的实现类。
- `ClassPathResource`： 访问类加载路径里资源的实现类。
- `FileSystemResource`： 访问文件系统里资源的实现类。
- `ServletContextResource`： 访问相对于 `ServletContext` 路径里的资源的实现类.
- `InputStreamResource`： 访问输入流资源的实现类。
- `ByteArrayResource`： 访问字节数组资源的实现类。

这些 `Resource` 实现类，针对不同的的底层资源，提供了相应的资源访问逻辑，并提供便捷的包装，以利于客户端程序的资源访问。



## SpringMVC

### 流程

1. 客户端发送请求，由`DispatcherServlet`接收
2. `DispatcherServlet`根据请求信息调用`HandlerMapping`，解析出请求的目标`Handler`
3. `HandlerAdapter`根据`handler`规则，找到并执行目标`Handler`（即`Controller`）
4. `Handler`执行完后，会返回一个`ModelAndView`给`HandlerAdapter`
5. `ViewResolver`负责解析视图`View`，根据逻辑`View`寻找实际`View`
6. `DispatcherServlet`把返回的`Model`传给`View`以渲染视图
7. 把`View`返回给客户端



### @Controller注解的作用

该注解负责标记控制器`Controller`，`Spring MVC`会扫描带`@Controller`注解的类中带`@xxxMapping`注解的方法，并为这些方法生成对应的处理器对象，给`HandlerMapping`解析

> `@RestController=@ResponseBody+@Controller`



