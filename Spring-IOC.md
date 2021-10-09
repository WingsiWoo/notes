# Spring-IOC

## IOC（Inversion of Control）

Spring 的 IoC 设计支持以下功能：

- 依赖注入
- 依赖检查
- 自动装配
- 支持集合
- 指定初始化方法和销毁方法
- 支持回调某些方法（但是需要实现 Spring 接口，略有侵入）



### 容器

容器管理着`Bean`的生命周期，控制着`Bean`的依赖注入。

`Spring`设置了两个接口用以表示容器：

- `BeanFactory`

  其实就是个`HashMap`，`key`为`BeanName`，`value`为`Bean`的实例，通常只提供注册（`put`到`map`里），获取（`get`）功能，是一个“**低级容器**”

  ```java
  /** Map from bean name to merged RootBeanDefinition. */
  private final Map<String, RootBeanDefinition> mergedBeanDefinitions = new ConcurrentHashMap<>(256);
  
  /** Names of beans that have already been created at least once. */
  private final Set<String> alreadyCreated = Collections.newSetFromMap(new ConcurrentHashMap<>(256));
  ```

- `ApplicationContext`

  `ApplicationContext` 可以称之为 **“高级容器”**。因为他比 `BeanFactory` 多了更多的功能。他继承了多个接口。因此具备了更多的功能。例如资源的获取，支持多种消息（例如 JSP tag 的支持），对 `BeanFactory` 多了工具级别的支持等待。所以你看他的名字，已经不是 `BeanFactory` 之类的工厂了，而是 “应用上下文”， 代表着整个大容器的所有功能。该接口定义了一个 `refresh` 方法，此方法是所有阅读 `Spring` 源码的人的最熟悉的方法，用于刷新整个容器，即重新加载/刷新所有的 `bean`。

![img](https://user-gold-cdn.xitu.io/2018/10/12/16665e117c5994e6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



### getBean

![img](https://user-gold-cdn.xitu.io/2018/10/12/16665e117cf84589?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

从图中可以看出，getBean 的操作都是在低级容器里操作的。其中有个递归操作，这个是什么意思呢？

假设 ： 当 `Bean_A` 依赖着 `Bean_B`，而这个 `Bean_A` 在加载的时候，其配置的 `ref = “Bean_B”` 在解析的时候只是一个占位符，被放入了 `Bean_A` 的属性集合中，当调用 `getBean` 时，需要真正 `Bean_B` 注入到 `Bean_A` 内部时，就需要从容器中获取这个 `Bean_B`，因此产生了递归。

为什么不是在加载的时候，就直接注入呢？因为加载的顺序不同，很可能 `Bean_A` 依赖的 `Bean_B` 还没有加载好，也就无法从容器中获取，你不能要求用户把 `Bean` 的加载顺序排列好，这是不人道的。

所以，`Spring` 将其分为了 2 个步骤：

1. 加载所有的 `Bean` 配置成 `BeanDefinition` 到容器中，如果 Bean 有依赖关系，则使用占位符暂时代替。
2. 然后，在调用 `getBean` 的时候，进行真正的依赖注入，即如果碰到了属性是 `ref` 的（占位符），那么就从容器里获取这个 `Bean`，然后注入到实例中 —— 称之为依赖注入。

可以看到，**依赖注入IOC实际上，只需要 “低级容器” 就可以实现。**



### 循环依赖

<img src="https://img2018.cnblogs.com/blog/1162587/201901/1162587-20190108224120891-308387799.png" alt="img" style="zoom:50%;" />

1. 构造器方式注入**（无法解决）**

   即A类所依赖的B类作为构造器参数传入，BC，AC之间同理。

   Spring会将正在创建的`bean`放在一个“当前创建bean”的池中，如果在创建`bean`的时候发现自己已经在这个池中就会抛出`BeanCurrentlyInCreationException`异常表示发生了循环依赖，创建完毕的`bean`将从该池清除。

   > 首先创建A类，把其放入池中，然后发现A依赖B，因此要去创建B，把B放入池中
   >
   > 创建B过程中发现B依赖C，创建C，把C放入池中
   >
   > 创建C过程中发现C依赖A，但是A已经在池中，因此说明发生了循环依赖，抛出异常

2. `Setter`方式注入-单例（默认）

   `Spring`先是用构造实例化`Bean`对象 ，此时`Spring`会将这个实例化结束的对象放到一个`Map`中，并且`Spring`提供了获取这个未设置属性的实例化对象引用的方法。

   > 先实例化ABC并放入*Map*中，然后初始化A的属性，发现是依赖B，因此会去*Map*中取出B的单例对象。
   >
   > 因此不会出现异常

3. `Setter`方式注入-原型**（无法解决）**

   `scope="prototype"` 意思是 每次请求都会创建一个实例对象。两者的区别是：有状态的`bean`都使用 `prototype` 作用域，无状态的一般都使用`singleton`单例作用域。

   如果是原型模式会报错：对于“`prototype`”作用域`Bean`，`Spring`容器无法完成依赖注入，因为“`prototype`”作用域的`Bean`，`Spring`容器不进行缓存，因此无法提前暴露一个创建中的`Bean`。



### 三级缓存

| 等级（不做强制说明） | 名称                  | 说明             | 作用                                                         |
| -------------------- | --------------------- | ---------------- | ------------------------------------------------------------ |
| 一级                 | singletonObjects      | 可以理解为单例池 | 存放初始化后的单例对象，也就是完成的bean对象                 |
| 二级                 | earlySingletonObjects | 早期单例对象缓存 | 存放实例化，未完成初始化的单例对象（未完成属性注入的对象），也是用来解决性能问题 |
| 三级                 | singletonFactories    | 单例工厂缓存     | 存放ObjectFactory对象，存放的是工厂对象，也是用来解决aop的问题 |

```java
// DefaultSingletonBeanRegistry
// 一级缓存：存放的是已经完成实例化，属性填充和初始化步骤的单例bean实例
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// 二级缓存：存储已经执行默认构造函数但未给属性赋值的对象，主要解决bean之间循环依赖的问题
// 存放的是提前暴露的单例bean实例，可能是代理对象，也可能是未经代理的原对象，但都还没有完成初始化的步骤
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

// 三级缓存：存储创建单例bean对象实例的工厂对象实例，一个工厂对象实例创建一种类型bean(必须先有工厂才能创建bean)
// 存放的是ObjectFactory的匿名内部类实例，一个获取提前暴露的单例bean引用的回调getEarlyBeanReference()，此处画重点，因为可以在这里进行代理对象的创建，也即AOP切面的织入。
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

> 一个Bean的创建，经历了 singletonFactories -> earlySingletonObjects -> singletonObjects的三个过程。三级缓存和二级缓存在创建bean的时候，即AbstractAutowireCapableBeanFactory.doCreateBean()方法中设置
>
> 一级缓存在获取Bean，即AbstractBeanFactory.doGetBean()方法中设置
>
> 单例对象先实例化存放于singletonFactories中，之后又存放于earlySingletonObjects中，最后完成后存放于singletonObjects中



#### 放入缓存的条件

```java
// Bean是单例 && 允许循环依赖 && 当前bean正在创建中
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      isSingletonCurrentlyInCreation(beanName));
```



#### 放入三级缓存

doGetBean()方法调用getEarlyBeanReference()获取到objectFactory对象，再把该对象作为参数传入到addSingletonFactory()方法放入三级缓存（beanName为key，objectFactory为value）

> getEarlyBeanReference()方法返回早期对象，用于将来升级到二级缓存: 要么返回原Object，要么返回执行了AbstractAutoProxyCreator.getEarlyBeanReference获取加强后的Object代理对象。

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
   Assert.notNull(singletonFactory, "Singleton factory must not be null");
   synchronized (this.singletonObjects) {
      // 一级缓存中有该bean的话，即容器中已经有创建完成的bean了，不用再放入三级缓存了
      if (!this.singletonObjects.containsKey(beanName)) {
         // 把一个ObjectFactory放入三级缓存，key为beanName
         this.singletonFactories.put(beanName, singletonFactory);
        // 如果二级缓存中存在该bean，就移除掉（即如果是未创建完成的话（二/三级缓存）就从头开始，创建已经完成（一级缓存）的话就直接使用）
				// 如果key不存在，remove方法不会报NPE的
         this.earlySingletonObjects.remove(beanName);
         this.registeredSingletons.add(beanName);
      }
   }
}
```



#### 放入二级缓存

doGetBean()方法调用getSingleton()方法移除三级缓存中的bean，放入到二级缓存中

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   // Quick check for existing instance without full singleton lock
   // 存有已经实例化的单例对象的map-singletonObjects，使用ConcurrentMap保证线程安全
   Object singletonObject = this.singletonObjects.get(beanName);
   // 使用contains查看map中是否存在该对象（一级缓存中没有该bean）
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      // 二级缓存中的早期对象
      singletonObject = this.earlySingletonObjects.get(beanName);
      // 二级缓存中也没有该bean
      if (singletonObject == null && allowEarlyReference) {
         // 上锁保证线程安全，两次判空保证一级缓存中获取到的bean真的为空
         synchronized (this.singletonObjects) {
            // Consistent creation of early reference within full singleton lock
            singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
               singletonObject = this.earlySingletonObjects.get(beanName);
               if (singletonObject == null) {
                  // 从三级缓存中获取工厂对象，用于之后创建早期对象放入二级缓存
                  ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                  if (singletonFactory != null) {
                     // 从前面的设置的ObjectFactory中，即getEarlyBeanReference方法中，获取放入二级缓存的早期对象
                     singletonObject = singletonFactory.getObject();
                     // 放入二级缓存
                     this.earlySingletonObjects.put(beanName, singletonObject);
                     // 从三级缓存中移除
                     this.singletonFactories.remove(beanName);
                  }
               }
            }
         }
      }
   }
   // 如果一级缓存/二级缓存中已经有bean了，就直接使用就可以了
   return singletonObject;
}
```



#### 放入一级缓存

```java
/**
	* 和放入二级缓存那里的getSingleton实际上调用的是同一个方法
	* 只不过第二个参数allowEarlyReference在上面那一次调用直接设置为true
	* 而本次调用的allowEarlyReference参数通过调用createBean方法获得
 	*/
sharedInstance = getSingleton(beanName, () -> {
   try {
      // 二三级缓存处理
      // 调用getSingleton方法之前先执行createBean方法获得参数（allowEarlyReference）
      return createBean(beanName, mbd, args);
   }
```



#### 三级缓存的作用

Spring bean生命周期的三个阶段：

1. 实例化
2. 属性装配

3. 初始化
4. 生成aop对象

![image-20210510210859145](/Users/wingsiwoo/Library/Application Support/typora-user-images/image-20210510210859145.png)

在初次创建bean对象的过程中，当完成第一阶段-bean实例化（`createBeanInstance()`）后不会直接进入第二阶段-属性装配，而是先检查是否允许提前暴露。

> 这也是构造器注入方式的循环依赖无法解决的原因，因为其在实例化阶段就已经需要B，而此时A还没放入缓存中，尝试创建B的时候无法在缓存中找到A

如果允许则通过将对应的工厂对象`singletonFactory`加入到三级缓存`singletonFactories`（`addSingletonFactory()`），使得外界能够提前获得该bean的未完成引用—将早期引用暴露，之后再进行属性装配（`populateBean()`）。

1. 首先创建A，完成A的实例化后，把其对应的工厂对象放入三级缓存（此时三级缓存`singletonFactories`中存有A的工厂对象）

2. 进行`bean A`的属性装配，发现A依赖于B，调用`getSingleton()`方法尝试从缓存中获取B，但B尚未创建过，因此会走跟1一样的流程去创建B

3. 实例化B后并将对应工厂对象放入三级缓存后，进行`bean B`的属性装配，调用`getSingleton()`方法尝试从缓存中获取A

4. 发现三级缓存`singletonFactories`中存有A对应的工厂对象，因此成功获得`bean A`的引用，完成了`bean B`的属性装填工作

5. 完成`bean B`的属性装填工作后，继续向下走。如果存在aop代理，则依赖的应该是代理对象，而不是原始的bean/bean工厂，因此要属性装填工作完成后要调用`getSingleton()`方法生成代理对象，并放入二级缓存，然后将其对应的工厂对象从三级缓存中移除。

   > 也就是说出现循环依赖时，才会执行工厂的`getObject`生成(获取)早期依赖，这个时候就需要给它挪个窝了，因为真正暴露的不是工厂，而是对象，所以需要使用一个新的缓存（二级缓存`earlySingletonObjects`）保存暴露的早期对象，同时移除提前暴露的工厂（从一级缓存中移除），也不需要在多重循环依赖时每次去执行`getObject`(虽然个人觉得不会出问题，因为代理对象不会重复生成

6. 如果bean B有aop代理的话，就对其进行代理，并把代理完成的对象放入一级缓存`singletonObjects`，从二级缓存中移除，至此，`bean B`创建完成。

7. 解决完`bean B`的创建后，bean A就可以继续完成属性装填工作，后续流程相同。



Q：二级缓存就可以解决循环依赖问题，那么为什么`Spring`还要采用三级缓存？

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

