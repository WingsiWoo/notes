# Spring源码

## @Transactional

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
	//alias means 别名，即value和transactionManager实际上是同一个属性
  //这两个属性用于多数据源时指定所使用的事务管理器
   @AliasFor("transactionManager")
   String value() default "";
   @AliasFor("value")
   String transactionManager() default "";
	//事务的传播行为
   Propagation propagation() default Propagation.REQUIRED;
	//事务的隔离机制
   Isolation isolation() default Isolation.DEFAULT;
	//事务超时时间
   int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
	//是否可读
   boolean readOnly() default false;
	//回滚的异常，默认只能回滚RuntimeException，如果要回滚非RuntimeException，就要设置该属性为Exception.class
   Class<? extends Throwable>[] rollbackFor() default {};

   String[] rollbackForClassName() default {};

   Class<? extends Throwable>[] noRollbackFor() default {};

   String[] noRollbackForClassName() default {};

}
```

> 事务的传播行为：
>
> REQUIRED（默认）：
>
> - 若方法A调用时，上下文中存在事务，则加入当前事务
> - 若方法A调用时，上下文中没有事务，则新建一个事务
> - 当在方法A调用方法B时，方法A、B将使用同一个事务
> - 如果方法B发生异常需要回滚时，将回滚整个事务
>
> REQUIRED_NEW ：
>
> - 对于方法A与B，无论方法调用时是否存在事务，都会开启一个新的事务
> - 且如果方法调用时存在事务，当前事务将被延缓
> - 如果方法B发生异常，将不会导致方法A的回滚
>
> NESTED ：
>
> - 与REQUIRED_NEW类似，但支持JDBC，不支持JPA或Hibernate
>
> SUPPORTS ：
>
> - 如果方法调用时，上下文中已经开启了事务，就使用这个事务
> - 如果方法调用时，上下文中没有事务，就不使用事务
>
> NOT_SUPPORTS ：
>
> - 以非事务的方式运行，如果存在事务，在方法调用到结束阶段事务都将被挂起
>
> NEVER ：
>
> - 以非事务的方式运行，如果存在事务，则将抛出异常
>
> MANDATORY ：
>
> - 强制方法在事务中执行，如果存在事务，则加入当前事务
> - 如果不存在事务，则将抛出异常



## 单例模式和原型模式的区别

> Spring官方文档中给出的bean的scope有五种：singleton、prototype、request、session、global session，其中前两个最基本（源码中对BeanDefinition的定义中也只有这两种）其他的都是扩展出来的

1. 单例是同一个对象，而原型是拷贝的对象（属性一模一样但实际上不是同一个对象）
2. 单例模式多次调用hashCode是相同的，但是原型模式多次调用hashCode是不同的（验证时要注意要在两个类分别注入，因为一个类中注入的时候只调用一次）
3. Spring中bean默认是单例的（为了提高性能、减少新生成实例的消耗和jvmGC），但是单例如果是在有状态的话在**并发环境下线程不安全**（并发环境下修改单例bean要考虑线程安全）

**单例bean的优势**

- 减少系统新创建实例消耗的资源
- 减少jvm垃圾回收
- 可以从缓存中快速获取到bean



## IOC

![img](https://images0.cnblogs.com/blog/400827/201409/172219470349285.x-png)

BeanFactory作为最顶层的接口定义了IOC容器的基本功能规范，Spring定义了多层次的接口，但最终的默认实现类是DefaultListableBeanFactory

> 既然只有一个默认实现类，那么为什么要定义这么多层次的接口？
>
> 主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。例如 ListableBeanFactory 接口表示这些 Bean 是可列表的，而 HierarchicalBeanFactory 表示的是这些 Bean 是有继承关系的，也就是每个Bean 有可能有父 Bean。AutowireCapableBeanFactory 接口定义 Bean 的自动装配规则。这四个接口共同定义了 Bean 的集合、Bean 之间的关系、以及 Bean 行为.

```java
/**
 * BeanFactory中有一个特殊的常量，是对FactoryBean的转义定义
 * 因为如果直接使用bean的名字检索FactoryBean得到的对象是工厂生成的对象，而如果要得     	* 到工厂本身，就需要在beanName前面加上&前缀进行转义
 */
String FACTORY_BEAN_PREFIX = "&";
```

Spring IoC容器对Bean定义资源的载入是从refresh()函数开始的，refresh()是一个模板方法，refresh()方法的作用是：在创建IoC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh之后使用的是新建立起来的IoC容器。refresh的作用类似于对IoC容器的重启，在新建立好的容器中对容器进行初始化，对Bean定义资源进行载入

![img](https://gitee.com/cbaj/blogImag/raw/master/img/bean%E5%88%9B%E5%BB%BA%E7%9A%84%E6%97%B6%E5%BA%8F%E5%9B%BE.png?imageslim)



## ClassPathXmlApplicationContext

### 构造器

当调用到ClassPathXmlApplicationContext的构造器时，无论传入的参数如何，都会一级级向下⬇️

```java
public ClassPathXmlApplicationContext(
      String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
      throws BeansException {

   super(parent);
   setConfigLocations(configLocations);
   if (refresh) {
      refresh();
   }
}
```

其中调用super(parent)的目的就是根据ClassPathXmlApplicationContext的继承关系一级级向上构建其上下文环境

![image-20210311213912030](/Users/wingsiwoo/Library/Application Support/typora-user-images/image-20210311213912030.png)



### refresh()

上述环境准备好后，接下来就要进行整个加载流程的核心方法refresh()

refresh()方法担任了整个Spring容器初始化和加载的所有逻辑，包括BeanFactory的初始化、post-processor的注册以及调用、bean的初始化、事件发布等

|              方法名               |                           方法作用                           |
| :-------------------------------: | :----------------------------------------------------------: |
|         prepareRefresh()          | 为刷新准备上下文，主要设置状态量(是否关闭、是否激活)、记录启动时间、初始化属性资源占位符、校验必填属性是否配置以及初始化用于存储早期应用事件的容器. |
|     obtainFreshBeanFactory()      | 主要用于获取一个新的BeanFactory，如果BeanFactory已存在，则将其销毁并重建，默认重建的BeanFactory为AbstractRefreshableApplicationContext;此外，此方法委托其子类从XML中或者基于注解的类中加载BeanDefinition |
|       prepareBeanFactory()        | 配置BeanFatory使其具有一个上下文的的标准特征，如上下文的类加载器、后处理程序(post-processors,如设置各种感知接口等) |
|     postProcessBeanFactory()      | 在应用上下文内部的BeanFactory初始化结束后对其进行修改，载所有的BeanDefinition已被加载但还没有实例化bean, 此刻可以注册一些特殊的BeanPostProcessors,如web应用会注册一些ServletContextAwareProcessor等 |
| invokeBeanFactoryPostProcessors() | 调用注册在上下文中的BeanFactoryPostProcessor，如果给与顺序则按照属性调用，**并且一定在单例对象实例化之前调用** |
|   registerBeanPostProcessors()    | 实例化并调用所有注册的BeanPostProcessor，如果有显式的顺序则按照顺序调用；**一定在任意的bean实例化之前调用** |
|        initMessageSource()        |                            国际化                            |
| initApplicationEventMulticaster() | 初始化ApplicationEventMulticaster，如果上下文没有定义则默认使用SimpleApplicationEventMulticaster,此类主要用于广播ApplicationEvent，onRefresh()在一些特定的上下文子类中初始化特定的bean， 如在Webapp的上下文中初始化主题资源 |
|        registerListeners()        | 添加实现了ApplicationListener的bean作为监听器，它不影响非bean的监听器；它还会使用多播器发布早期的ApplicationEvent，finishBeanFactoryInitialization() 实例化所有非延迟加载的单例，完成bean factory的初始化工作 |
|          finishRefresh()          | 完成上下文的刷新工作，调用LifecycleProcessor的onFresh()方法以及发布ContextRefreshedEvent事件 |
|        resetCommonCaches()        | 重置Spring公共的缓存，比较典型的如： ReflectionUtils, ResolvableType以及CachedIntrospectionResults的缓存 |

如果refresh()方法产生异常，会在catch快中摧毁已经创建的bean避免资源浪费，并且重置prepareRefresh()中更新的active标识 ，最后再抛出异常给调用者



#### obtainFreshBeanFactory()

该方法首先调用`refreshBeanFactory()`方法，`refreshBeanFactory()`方法的默认实现由`AbstractRefreshableApplicationContext`来提供

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
  //先检查原先是否已经存在BeanFactory，如果有就销毁关闭
   if (hasBeanFactory()) {
      destroyBeans();
      closeBeanFactory();
   }
   try {
     //建立默认的BeanFactory-DefaultListableBeanFactory
      DefaultListableBeanFactory beanFactory = createBeanFactory();
      beanFactory.setSerializationId(getId());
     //简单配置，是否允许对BeanDefinition进行重写、是否允许循环引用
      customizeBeanFactory(beanFactory);
     //BeanDefinition的加载过程
      loadBeanDefinitions(beanFactory);
      this.beanFactory = beanFactory;
   }
   catch (IOException ex) {
      throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
   }
}
```

 loadBeanDefinitions(beanFactory)有多种实现，以下主要分析AbstractXmlApplicationContext（基于XML的加载实现）的实现

```java
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   // Create a new XmlBeanDefinitionReader for the given BeanFactory.
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

   // Configure the bean definition reader with this context's
   // resource loading environment.
   beanDefinitionReader.setEnvironment(this.getEnvironment());
   beanDefinitionReader.setResourceLoader(this);
   beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

   // Allow a subclass to provide custom initialization of the reader,
   // then proceed with actually loading the bean definitions.
   initBeanDefinitionReader(beanDefinitionReader);
   loadBeanDefinitions(beanDefinitionReader);
}
```

> 1. XmlBeanDefinitionReader对象用于读取xml的bean定义，实际上它会将对于xml的处理委托给BeanDefinitionDocumentReader接口，这个对象的构造器需要传入BeanDefinitionRegistry对象，即上述代码中的beanFactory
> 2. 之后按顺序为环境、当前上下文、实体处理赋值
> 3. 14行主要是对beanDefinitionReader进行配置，为后面其加载BeanDefinition作准备（默认为空实现）
> 4. 15行是BeanDefinition的主要加载过程，它使用了第4行创建的BeanDefinitionReader进行加载。这个方法仅仅负责加载或者注册BeanDefinition（实际上是由beanDefinitionReader执行的），而生命周期是通过beanFactory的refreshbeanFactory()方法管理

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
   Assert.notNull(encodedResource, "EncodedResource must not be null");
   if (logger.isTraceEnabled()) {
      logger.trace("Loading XML bean definitions from " + encodedResource);
   }
	//在解析Resources之前，Spring会将此Resouce存储于当前线程的局部变量resourcesCurrentlyBeingLoaded中（是ThreadLocal<Set<EncodedResource>>类型），解析完成后会把该Resource移除。这里是得到本线程的Set集合
   Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	//如果向存储的Set集合中加入本Resource失败，说明集合中已经存在该Resource，不可重复解析
   if (!currentResources.add(encodedResource)) {
      throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
   }

   try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
      ......
      return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
   }
   finally {
     //解析完成之后移除Resource
      currentResources.remove(encodedResource);
      if (currentResources.isEmpty()) {
         this.resourcesCurrentlyBeingLoaded.remove();
      }
   }
}
```

> 6行注释的目的：
>
> 1. 利用Set集合保证存储的Resource唯一，不会重复解析
> 2. 利用ThreadLocal保证线程安全。因为Set不是线程安全的，在多线程情况下，仍然有可能出现Resource重复解析的情况。传统的解决方式是用synchronized加锁，但是这样会导致效率低下，因此Spring采用ThreadLocal，保证每个线程只能访问自己的Set

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
      throws BeanDefinitionStoreException {

   try {
     //将resource解析为Document对象
      Document doc = doLoadDocument(inputSource, resource);
     //对Document对象进行提取与BeanDefinition的注册
      int count = registerBeanDefinitions(doc, resource);
   }
  
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
  //创建一个BeanDefinitionDocumentReader对象，此对象是一个BeanDefinition注册到容器的一个SPI（Service Provider Interface），它的默认实现为DefaultBeanDefinitionDocumentReader,它是通过反射获取到的实例（BeanUtils.instantiateClass）
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
  //获取对此Document解析之前已经注册的BeanDefinition的数量，因为此方法要返回的是本次注册的数量
		int countBefore = getRegistry().getBeanDefinitionCount();
  //解析和注册，具体的实现在DefaultBeanDefinitionDocumentReader
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
}  
```

> 13行处采用反射获取实例的原因是为了后续方便扩展，使得自定义注册成为可能（通过定义documentReaderClass）

在doRegisterBeanDefinitions方法中会将document.getDocumentElement（root）中的每个beanDefinition都进行注册，而对于root的解析则交由BeanDefinitionParseDelegate对象在parseBeanDefinitions()中实现

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
  //判断root的namespace是否为“http://www.springframework.org/schema/beans”
   if (delegate.isDefaultNamespace(root)) {
     //是则获取子节点
      NodeList nl = root.getChildNodes();
      for (int i = 0; i < nl.getLength(); i++) {
         Node node = nl.item(i);
         if (node instanceof Element) {
            Element ele = (Element) node;
           //如果子节点的namespace也是“http://www.springframework.org/schema/beans”
            if (delegate.isDefaultNamespace(ele)) {
              //根据namespace调用对应方法进行解析
               parseDefaultElement(ele, delegate);
            }
            else {
              //用户自定义的方式去解析
               delegate.parseCustomElement(ele);
            }
         }
      }
   }
   else {
     //使用用户自定义的规则去解析元素
      delegate.parseCustomElement(root);
   }
}
```

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
  //把element解析为BeanDefinitionHolder
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
      //对holder进行装配
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 把holder持有的beanDefinition注册到上下文中，就是存放到一个map中，key为bean的名字，value为BeanDefinition
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			// 发送注册事件
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

> What is BeanDefinitionHolder？
>
> 其通过name和alias持有beanDefinition，最终注册的时候会将其持有的BeanDefinition注册到上下文中



### getBean

1. 转换beanName，根据传入的beanName在别名表aliasMap中查找，如果不为空（表示传入的beanName为别名）就要用查找到的value即真实的beanName替换
2. 通过getSingleton()方法检查bean是否已经创建过（在缓存中），如果是：对于普通的bean会直接返回；对于FactoryBean类型的则会创建对应的实例返回。如果不是：如果这个bean是正在创建的Prototype类型的，Spring无法处理该类型的循环依赖，抛出异常；非prototype类型的查看父类中是否有相关的bean的定义信息

![img](https://user-gold-cdn.xitu.io/2018/10/12/16665e117cf84589?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



## 核心类

### DefaultListableBeanFactory

DefaultListableBeanFactory是整个bean加载的核心部分，是Spring注册及加载bean的默认实现，其他的如XmlBeanFactory与其不同的地方只是使用了不同的读取器(XmlBeanDefinitionReader)，实现了个性化的BeanDefinitionReader读取

![image-20210318205759223](/Users/wingsiwoo/Library/Application Support/typora-user-images/image-20210318205759223.png)

XML配置文件读取的大致流程：

![image-20210318212405729](/Users/wingsiwoo/Library/Application Support/typora-user-images/image-20210318212405729.png)

1. 通过继承自AbstractBeanDefinitionReader的loadBeanDefinitions方法，利用其内的ResourceLoader把资源文件路径转换成Resource对象
2. 再通过XmlBeanDefinitionReader中的doLoadDocument方法，用DocumentLoader对Resource进行转换，把其转换为Document
3. 然后调用DefaultBeanDefinitionDocumentReader中的registerBeanDefinitions方法，利用BeanDefinitionParserDelegate对Document.getDocumentElement进行解析

Resource接口抽象了所有spring内部使用到的底层资源：File、URL、Classpath等。这个接口定义了3个判断当前资源的方法：isOpen()（是否处于打开状态）、isReadable()（是否可读）、exists()（是否存在）。使得可以对所有资源文件进行统一处理。



### BeanDefinition

![image-20210326232239672](/Users/wingsiwoo/Library/Application Support/typora-user-images/image-20210326232239672.png)

BeanDefinition是一个接口，提供了与<bean>元素标签拥有的class、scope、lazy-init等配置属性相应的beanClass、scope、lazyInit属性，是配置文件<bean>元素标签在容器中的内部表示形式。

其含有3个实现类：RootBeanDefinition、GenericBeanDefinition、ChildBeanDefinition，其中父<bean>或者没有父<bean>的使用RootBeanDefinition表示。

而AbstractBeanDefinition对Root~和Child～进行了抽象。

Spring会将BeanDefinition注册到BeanDefinitionRegistry中（这玩意就像是Spring的内存数据库，专门用于存储配置信息，主要用map进行保存）



![img](https://img2018.cnblogs.com/blog/1162587/201901/1162587-20190108223107433-1167042193.png)







## AOP(Aspect-Oriented Programming)

