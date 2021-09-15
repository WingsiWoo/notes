# JVM

## Java技术体系

- JDK（Java Development Kit）：Java程序设计语言、Java虚拟机、Java类库

  > JDK是用于支持Java程序开发的最小环境

- JRE（Java Runtime Environment）：Java类库API中的Java SE API子集、Java虚拟机

  > JRE是支持Java程序运行的标准环境



## 虚拟机

就是一台虚拟的计算机，它是一款软件，用来执行一系列虚拟计算机指令。

**系统虚拟机**：如Visual Box、VMWare，它们完全是对物理计算机的仿真，提供了一个可运行完整操作系统的软件平台

**程序虚拟机**：如JVM，它专门为执行单个计算机程序而设计，在JVM中执行的指令称为字节码指令

> - Java虚拟机是一台执行字节码的虚拟计算机，它拥有独立的运行机制，其运行的字节码文件不一定由Java语言编译而成
>
> - JVM平台的各种语言可以共享JVM带来的跨平台性、优秀的垃圾回收机制、可靠的即时编译器
>
> - 所有的Java程序都运行在JVM内部



## Java——跨平台的语言

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsdwtpdz18j30op0ee0wd.jpg" alt="image-20200727121755076" style="zoom:67%;" />

1. 一次编写，到处运行
2. 提供了一种相对安全的内存管理和访问机制，避免了绝大部分内存泄漏和指针越界问题
3. 实现了热点代码检测和运行时编译
4. 有一套完整的应用接口程序



## JVM——跨语言的平台

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsdwtu08rrj30sa0crgqj.jpg" alt="image-20200727121725188" style="zoom:67%;" />

1. 一次编译，到处运行
2. 自动内存管理
3. 自动垃圾回收功能
4. JVM是运行在操作系统之上的，它与硬件没有直接交互

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsdwtycv5rj30j30hgtf0.jpg" alt="image-20210709173035561" style="zoom:67%;" />

上图主要针对HotSpot（目前市面上使用最广泛的虚拟机），它采用解释器与即时编译器并存的架构。



## JVM概述

### Java代码执行流程

![image-20210709181126780](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwu1k7ytj30kg0g20xy.jpg)

> JIT编译器把热点代码缓存起来，是性能高的关键



#### 解释器与JIT编译器

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsdwu69atej312x0f57eh.jpg" alt="image-20210711112417208" style="zoom:80%;" />

> 解释器是一行一行地将字节码解析成机器码，解释到哪就执行到哪，狭义地说，就是for循环100次，你就要将循环体中的代码逐行解释执行100次。当程序需要迅速启动和执行时，解释器可以首先发挥作用，省去编译的时间，立即执行。
>
> ​																	==输入的代码 -> [ 解释器 解释执行 ] -> 执行结果==
>
> JIT编译器以方法为单位，将热点代码的字节码一次性转为机器码，并在本地缓存起来的工具。避免了部分代码被解释器逐行解释执行的效率问题。但：**编译所需时间肯定要比解释长，另外code catch的内存是有限的，所以全部采用即时编译是不实际的**
>
> ​												==输入的代码 -> [ 编译器 编译 ] -> 编译后的代码 -> [ 执行 ] -> 执行结果==

解释器需要逐行解释字节码，中途基本没有暂停，但是如果遇到for循环之类的，可能会一直重复解释同一段代码（类似于步行）

JIT编译器先寻找热点代码，再对热点代码进行编译，这一段编译时间会比解释器要长，形成一段时间的暂停。JIT编译器把编译完成后的指令放在缓存中，运行速度很快（类似于等待公交车需要一段时间，但是一旦上车就会比步行要快）

> 说JIT比解释快，其实说的是**执行编译后的代码**比**解释器解释执行**要快，编译这个动作实际上是要比解释慢的。
>
> 对于只执行一次的代码，一般采用解释执行；对于存在循环的代码，一般采用JIT编译（所以一般所说的热点代码，就是指被多次调用的方法、被多次执行的循环体）



### JVM的架构模型

> Java编译器输入的指令流基本上是一种基于栈的指令集架构

#### **基于栈式架构的特点**

1. 设计和实现更简单，适用于资源受限的系统
2. 避开了寄存器的分配难题：使用零地址指令方式分配
3. 指令流中的指令大部分是零地址指令，其执行过程依赖于操作栈。指令集更小（但是指令多），编译器容易实现
4. 不需要硬件支持，可移植性更好，更好地实现**跨平台**
5. 缺点：性能下降，实现同样的功能需要更多的指令

> 对栈来说不存在垃圾回收问题

#### **基于寄存器架构的特点**

1. 典型的应用是x86的二进制指令集
2. 指令集架构完全依赖硬件，可移植性差
3. 性能优秀、执行更高效
4. 花费更少的指令去完成一项操作
5. 在大部分情况下，基于寄存器架构的指令集往往以一地址指令、二地址指令和三地址指令为主，而基于栈式架构的指令集一般以零地址指令为主

> ```java
> public static void main(String[] args) {
>     int i = 2;
>     int j = 3;
>     int k = i + j;
> }
> // 反编译-进入到out包下需要反编译的class文件所在的目录，然后输入命令javap -v xxx.class
> ```
>
> ![image-20210711105418881](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwuai7qaj30bd08nt9x.jpg)



### JVM的生命周期

#### **虚拟机的启动**

JVM的启动是通过引导类加载器（`bootstrap class loader`）创建一个初始类（`initial class`）来完成的，这个类是由虚拟机的具体实现指定的

#### **虚拟机的执行**

- 一个运行中的`JVM`有着一个清晰的任务：执行`Java`程序
- 程序开始执行时他才运行，程序结束时他就停止
- 执行一个所谓的`Java`程序时，真正在执行的是一个叫做`JVM`的进程

#### **虚拟机的退出**

- 程序正常执行结束
- 程序在执行过程中遇到了异常或错误而异常终止
- 由于操作系统出现错误而导致`JVM`进程终止
- 某线程调用`Runtime`类或`System`类的`exit`方法，或`Runtime`类的`halt`方法，并且`java`安全管理器也允许这次`exit/halt`操作



### 三大虚拟机

#### HotSpot

- 从`JDK1.3`起，`HotSpot`就成为了默认虚拟机
- `HotSpot`指的就是其热点代码探测技术
  - 通过计数器找到最具编译价值的代码，触发即时编译或栈上替换
  - 通过编译器与解释器协同工作，在最优化的程序响应时间（取决于解释器）与最佳执行性能（取决于编译器）中取得平衡

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsdwue60n3j30j30hgtf0.jpg" alt="image-20210709173035561" style="zoom:67%;" />



#### JRockit

- 专注于服务器端应用：它可以不太关注程序启动速度，因此`JRockit`内部不包含解析器实现，全部代码都靠即时编译器编译后执行
- 是目前最快的`JVM`
- 全面的`Java`运行时解决方案组合
  - `JRockit`面向延迟敏感性应用的解决方案`JRockit Real Time`提供以毫秒或微秒级的JVM响应时间，适合财务、军事指挥、电信网络的需要
  - `MissionControl`服务套件，它是一组以极低的开销来监控、管理和分析生产环境中的应用程序的工具



#### J9

- 服务器端、桌面应用、嵌入式等多用途`VM`



## 内存结构

![image-20210711152358198](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwuiwrquj30ks0owdzh.jpg)

> 类加载器子系统`ClassLoader SubSystem`
>
> 1. 加载`Loading`
>    - 引导类加载器-`BootStrap ClassLoader`
>    - 扩展类加载器-`Extension ClassLoader`
>    - 应用/系统累加载器-`Applicayion ClassLoader`
> 2. 链接`Linking`
>    - 验证-`Verify`
>    - 准备-`Prepare`
>    - 解析-`Resolve`
> 3. 初始化`Initialization`
>
>  
>
> 运行时数据区`Runtime Data Area`
>
> - PC程序计数器：每个线程有一个
> - 虚拟机栈：每个线程维护一个栈，其中栈中的每个元素称为`Stack Frame`-栈帧
> - 本地方法栈`Native Method Stack`
> - 堆区`Heap Area`：堆区是共享的，是内存中最大的一块
> - 方法区`Method Area`：只有`HotSpot`才有
>
>  
>
> 执行引擎`Execution Engine`
>
> 1. 解释器`Interperter`
> 2. `JIT`即时编译器
>    - 单词代码生成器➡️代码优化器➡️目标代码生成器
>    - 分析器
> 3. 垃圾回收器`Garbage Collection`
>
>  
>
> 本地方法接口`Native Method Interface`
>
> 本地方法库`Native Method Library`



### 虚拟机类加载机制

> `JVM`把描述类的数据从`Class`文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用`的`Java类型，这个过程被称为虚拟机的类加载机制
>
> 与那些在编译时需要进行连接的语言不同，在`Java`语言里，类型的加载、连接和初始化过程都是在程序运行期间完成的，这种策略让`Java`语言进行提前编译会面临额外的困难，也会让类加载时稍微增加一些性能开销，但是却为Java应用提供了极高的扩展性和灵活性，`Java`可以动态扩展的语言特性就是依赖运行期动态加载和动态连接这个特点实现的

![image-20210711153301587](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwunn3wtj30kn09ctd2.jpg)



- 类加载器子系统负责从文件系统或者网络中加载`class`文件，class`文件`在文件开头有特定的文件标识（`CAFE BABE`，又称魔数）

- `ClassLoader`只负责`class`文件的加载，至于其是否可以运行，由`Execetion Engine`决定

- 加载的类信息存放于一块称为方法区的内存空间。除了类的信息外，方法区中还会存放运行时常量池信息，可能还包括字符串字面量和数字常量（这部分常量信息是`Class`文件中常量池部分的内存映射）

  > ![image-20210711153809254](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwupdk6uj30j304adg5.jpg)
  >
  > 由类的字节码文件生成该类的二进制流，通过字节流将该类的类信息放在方法区，最后在内存中生成该类的Class对象放在堆中，这个Class对象作为访问方法区数据的入口。
  >
  > 类被加载之后，Class文件中的常量池会被复制一份到方法区，成为运行时常量池



![image-20210711154413769](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwuuta92j30fp0crq5e.jpg)

1. `class file`存在于本地硬盘上，可以理解为设计师画在纸上的模板，而最终这个模板在执行的时候是要加载到JVM当中来的，然后根据这个模板文件实例化对象
2. `class file`加载到`JVM`中，被称为`DNA`元数据模板，放在方法区
3. 在`.class`文件→`JVM`→最终成为元数据模板，此过程中需要类加载器`ClassLoader`作为一个运输的角色



#### 类加载流程

![image-20210711155042588](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwuyc07bj30gd0ci77s.jpg)

![image-20210711212018389](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwv14hodj30kr07sgp5.jpg)

> 类的加载过程必须按照上图按部就班的开始（指的是开始顺序，这些阶段通常是互相交叉的混合进行，会在一个阶段执行的过程中调用、激活另一个阶段）。而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持`Java`语言的运行时绑定特性（又称动态绑定）





#### 加载-Loading

1. 通过一个类的全限定名获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构

3. 在内存中生成一个代表这个类的`java.lang.Class`对象，作为方法区这个类的各种数据的访问入口



#### 链接-Linking

##### 验证Verify

- 目的在于确保`Class`文件的字节流中包含信息符合当前虚拟机的要求，确保被加载类的正确性，不会危害虚拟机自身安全
- 主要包括四种验证：文件格式验证、元数据验证、字节码验证、符号引用验证

##### 准备Prepare

- ==为类变量分配内存并且设置该类变量的默认初始值，即零值==

  > 这里不包含用`final`修饰的`static`，因为==`final`在编译时就会分配了，准备阶段会显式初始化==
  >
  > 这里不会为实例变量分配初始化，类变量会分配在方法区中，而实例变量是会随着对象一起分配到Java堆中

##### 解析Resolve

- 将常量池内的符号引用转换为直接引用的过程

  > 符号引用就是一组符号来描述所引用的目标；直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄

- 实际上，解析操作往往会伴随着`JVM`在执行完初始化之后再执行

- 解析动作主要针对类或接口、字段、类方法、接口方法、方法类型等



#### 初始化-Initialization

- **初始化阶段就是执行类构造器方法`<clinit>()`的过程**

  > - 此方法不需定义，是`javac`编译器自动收集类中的**所有类变量的赋值动作和静态代码块中的语句**合并而来
  >
  >
  > - `<clinit>()`不同于类的构造器（构造器是虚拟机视角下的`<init>()`）
  >
  > ![image-20210711165946651](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwv4s7hoj30hv081dgm.jpg)
  >
  > - `clinit`只有在类里定义了静态变量/静态代码块时才会出现
  > - 接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口和类一样都会自动生成`<clinit>()`方法。但接口与类不一样的是，其不需要先执行父接口的`clinit`方法，因为只有当父接口中定义的变量被使用时，父类接口才会被初始化。此外，接口的实现类在初始化时也不会执行接口的`clinit`方法

- 构造器方法中指令**按语句在源文件中出现的==顺序执行==**

  > ```java
  > public class ClassInitTest {
  >     private static int num = 10;
  >     static {
  >         num = 2;
  >         number = 3;
  >         // 如果在此处System.out.println(num);没有问题
  >         // 但如果是System.out.println(number);就会报错：Illegal forward reference-非法的前向引用
  >         // 因为虽然这个值已经有初值了，但由于顺序执行，还没有执行line12，即初始化阶段，而编译器不允许引用一个未初始化的值
  >     }
  > 
  >     private static int number = 20;
  > 
  >     public static void main(String[] args) {
  >         System.out.println("[num]" + ClassInitTest.num);
  >         System.out.println("[number]" + ClassInitTest.number);
  >     }
  > }
  > ```
  >
  > ![image-20210711170434641](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwv83u24j305r01lglf.jpg)
  >
  > `number`变量在`Linking`中的`Prepare`阶段中就把其赋值为0（零值）
  >
  > 然后在`Initialization`阶段，**顺序执行**，先在`static`代码块把其初始化为3，然后执行到`line9`，把其覆盖为20，因此最终`number`的值为20
  >
  > 同理，`num`在`Linking`的`Prepare`阶段被赋为零值，在`Initialization`阶段，先在`line2`被初始化为10，然后在`static`代码块中被覆盖为2，因此最终`num`的值为2
  >
  > ![image-20210711172049079](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwvbdseoj30a7051aav.jpg)

- 若该类具有父类，JVM会保证子类的`<clinit>()`执行前，父类的`<clinit>()`已执行完毕

- 虚拟机保证`<clinit>()`只执行一次

  > 由于静态代码块是在`clinit`里面执行的，所以静态代码块只执行一次

- 虚拟机必须保证一个类的`<clinit>()`方法在多线程下被同步加锁

  > ```java
  > public static void main(String[] args) {
  >         Runnable runnable = () -> {
  >             System.out.println("线程运行");
  >             DeadThread deadThread = new DeadThread();
  >             System.out.println("FINISH");
  >         };
  >         new Thread(runnable,"A").start();
  >         new Thread(runnable,"B").start();
  >     }
  > 
  > }
  > 
  > class  DeadThread {
  >     static {
  >         if(true) {
  >             System.out.println(Thread.currentThread().getName()+"类初始化");
  >             // 死循环使得首先进来的线程停留在这里，模拟初始化的过程
  >             while(true) {
  > 
  >             }
  >         }
  >     }
  > }
  > ```
  >
  > ![image-20210711182149025](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwveqeoqj307n02o0so.jpg)
  >
  > 可以看到两个线程都运行了，而线程A先抢占到锁，但由于死循环的存在，锁一直没有被释放，所以线程B一直无法进入初始化
  >
  > ![image-20210711182313156](https://tva1.sinaimg.cn/large/008i3skNgy1gsdwvhiz3rj306w044mx4.jpg)
  >
  > 把死循环去掉后，可以看到类初始化只打印了一次，验证了上面那点保证只执行一次



#### 双亲委派模型

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsdwvki61oj30uu0ggabt.jpg" alt="image-20210711184840403" style="zoom:80%;" />

如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是**把这个请求委派给父类加载器去完成**，因此**所有的加载请求最终都应该传送到最顶层的启动类加载器`Bootstrap ClassLoader`中**，只有当父加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己去完成加载

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 首先检查请求的类是否已经被加载过了
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                  	// 请求委派给父类加载器
                    c = parent.loadClass(name, false);
                } else {
                  	// 启动类加载器在Java中的表示是null，因此parent为null的时候就去请求Bootstrap ClassLoader
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
              	// 如果父类加载器抛出ClassNotFoundException，说明其无法完成加载请求
            }

            if (c == null) {
                // 父类加载器无法完成加载请求时，调用自己的findClass方法来完成类加载
                long t1 = System.nanoTime();
                c = findClass(name);
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

**优点**

- 避免类的重复加载
- 保证程序安全，防止核心`API`被恶意篡改

> 例如`java.lang.Object`，它存放于`rt.jar`中，由启动类加载器进行加载，在双亲委派模型下无论哪一个类加载器要加载这个类，最终都是启动类加载器进行加载，这就保证了`Object`类在程序的各种类加载器环境中都能保证是同一个类；
>
> 反之，如果没有使用双亲委派模型，用户自己编写了一个名为`java.lang.Object`类，并放在程序的`Classpath`中，系统中就可能会出现多个不同的`Object`类（实际上会报`java.lang.SecurityException`）

##### 沙箱安全机制

如果自定义一个`java.lang.String`类，内有一个`main`方法，执行其`main`方法会报如下错

![image-20210712102254425](https://tva1.sinaimg.cn/large/008i3skNgy1gsdycqy7chj30fg03adg3.jpg)

这是因为在加载自定义`String`类的时候会率先使用启动类加载器加载，而启动类加载器在加载过程中会先加载`JDK`自带的文件（`rt.jar`包下的`java/lang/String.class`），报错信息说没有`main`方法，是因为实际上加载的是`rt.jar`包中的`String`类，这样可以保证对`java`核心源代码的保护，这就是沙箱安全机制。



#### 类加载器的分类

```java
public class ClassLoaderTest {
    public static void main(String[] args) {
        // 获取系统类加载器
        ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
        System.out.println(systemClassLoader); // sun.misc.Launcher$AppClassLoader@18b4aac2
        // 获取其上层：扩展类加载器
        ClassLoader extClassLoader = systemClassLoader.getParent();
        System.out.println(extClassLoader); // sun.misc.Launcher$ExtClassLoader@5cad8086
        // 试图获取上层：获取不到引导类加载器
        ClassLoader bootStrapClassLoader = extClassLoader.getParent();
        System.out.println(bootStrapClassLoader);   // null
        // 用户自定义类默认使用系统类加载器
        System.out.println(ClassLoaderTest.class.getClassLoader()); // sun.misc.Launcher$AppClassLoader@18b4aac2
        // String类使用引导类加载器进行加载——Java的核心类库都是使用引导类加载器进行加载
        System.out.println(String.class.getClassLoader());  // null
    }
}
```

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsdwvo0ef0j30jb0fgn59.jpg" alt="image-20210711223127267" style="zoom:80%;" />

##### 启动类加载器（引导类加载器，Bootstrap ClassLoader）

- 使用`C/C++`实现，嵌套在`JVM`内部
- 用来加载`Java`的核心类库，用于提供`JVM`自身需要的类
- 并不继承自`java.lang.ClassLoader`，没有父加载器
- 出于安全考虑，`Bootstrap`启动类加载器只加载包名为`java、javax、sun`等开头的类



##### 扩展类加载器（Extension ClassLoader）

- Java语言编写，由`sun.misc.Launcher$ExtClassLoader`实现
- 派生于`ClassLoader`类，是`Launcher`的一个静态内部类
- 从`java.ext.dirs`系统属性所指定的目录中加载类库，或从`JDK`的安装目录的`jre/lib/ext`子目录（扩展目录）下加载类库



##### 应用程序类加载器（系统类加载器，AppClassLoader）

- `Java`语言编写，由`sun.misc.Launcher$AppClassLoader`实现
- 派生于`ClassLoader`类，是`Launcher`的一个静态内部类
- 负责加载环境变量`classpath`或系统属性`java.class.path`指定路径下的类库
- 该类是程序中默认的类加载器，一般来说，`Java`应用的类都是由它来完成加载的
- 通过`ClassLoader.getSystemClassLoader()`方法可以获取到该类加载器



#### 扩展

##### 是否为同一个类的判断

在`JVM`中表示两个`class`对象是否为同一个类存在两个必要条件：

1. 类的全限定类名必须一致
2. 加载这个类的`ClassLoader`（指实例对象）必须相同



##### 对类加载器的引用

`JVM`必须知道一个类型是由启动类加载器加载的还是由用户类加载器加载的。如果一个类型是由用户类加载器加载的，那么**`JVM`会将这个类加载器的一个引用作为类型信息的一部分保存在方法区中**。当解析一个类型到另一个类型的引用的时候，`JVM`需要保证这两个类型的类加载器是相同的。 



##### 类的主动使用和被动使用

- 创建类的实例

- 访问某个类或接口的静态变量，或者对该静态变量赋值

- 调用类的静态方法

- 反射（如`Class.forName()`）

- 初始化一个类的子类

- `Java`虚拟机启动时被标明为启动类的类

- `JDK7`开始提供的动态语言支持：

  `Java.lang.invoke.MethodHandle`实例的解析结果

  REF_getStatic、REF_putStatic、REF_invokeStatic句柄对应的类没有初始化，则初始化

除了以上的情况，其他使用`Java`类的方式都被看作是**对类的被动使用，会加载，但不会导致类的初始化**



### 运行时数据区

![image-20210712105453104](https://tva1.sinaimg.cn/large/008i3skNgy1gsdza0s1kdj30jp07sjzi.jpg)

> 线程间共享：方法区、堆区——随着虚拟机启动而创建，随着虚拟机退出而销毁
>
> 线程独享：程序计数器、本地方法栈、虚拟机栈——随着线程的开始和结束而创建和销毁

![image-20210716161113138](https://tva1.sinaimg.cn/large/008i3skNgy1gsiuwfnctwj30tw0e9my8.jpg)



#### 程序计数器

![image-20210712110008164](https://tva1.sinaimg.cn/large/008i3skNgy1gsdzfhd90qj303f06rjsh.jpg)

程序计数器（`Program Counter Register`）是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。在`JVM`的概念模型里，字节码解释器工作时就是通过改变程序计数器的值来选取下一条需要执行的字节码指令，**它是程序控制流的指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个程序计数器来完成。**

> 如果线程正在执行的是一个`Java`方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果是本地（`Native`）方法，这个计数器的值为空。
>
> 程序计数器的内存区域是唯一一个在规范中没有规定任何`OutOfMemoryError`情况的区域（也不需要`GC`），它是一块很小的内存空间，几乎可以忽略不计，也是运行速度最快的存储区域



##### PC值变化解析

```java
public static void main(String[] args) {
    int i = 10;
    int j = 20;
    int k = i+ j;

    String str = "abc";
    System.out.println(i);
    System.out.println(k);
}
```

![image-20210713101907881](https://tva1.sinaimg.cn/large/008i3skNgy1gsf3v60imej30a3090wf6.jpg)

> `10 ldc #2`表示到常量池`#2`位置取常量
>
> ![image-20210713102005448](https://tva1.sinaimg.cn/large/008i3skNgy1gsf3w4da65j305r00qdfn.jpg)
>
> ![image-20210713102022370](https://tva1.sinaimg.cn/large/008i3skNgy1gsf3wepfewj306z01smx0.jpg)
>
> 常量池内`#2`存储的是字符串的相关信息，然后指向常量池中的`#27`位置
>
> ![image-20210713102115229](https://tva1.sinaimg.cn/large/008i3skNgy1gsf3xc0rqbj304z02yq2t.jpg)
>
> `#27`位置存放的就是字符串`abc`，因此其实`10 ldc #2`走了两步，所以下一条指令程序计数器的值为10+2=12
>
> `14 getstatic #3`同理（#3->#28、#29->#34、#35、#36），转了三次，所以程序计数器的下一条指令位置为当前值+3

![image-20210713103517116](https://tva1.sinaimg.cn/large/008i3skNgy1gsf4bxdbetj30hr066ack.jpg)



##### 两个常见问题

1. 为什么使用PC寄存器记录当前线程的执行地址？

   因为`CPU`需要经常切换不同线程，而在切换到不同线程后需要知道当前线程执行到哪里，要从哪里开始继续执行。`JVM`的字节码解释器需要通过改变PC寄存器的值来明确下一条应该执行什么样的字节码指令

2. PC寄存器为什么会被设定为线程私有？

   **为了能够准确地记录各个线程正在执行的当前字节码指令地址，使得各个线程之间可以进行独立计算，从而不会出现相互干扰的情况。**

   由于CPU时间片轮限制，众多线程在**并发执行过程**中，任何一个确定的时刻，一个处理器或者多核处理器中的一个内核，只会执行某个线程中的一条指令。这样就必然会导致线程的中断和恢复，因此程序计数器记录的值便有利于线程的恢复。

> 并发：一个时间段内多个线程抢占`CPU`时间片，使得宏观上看起来像是多个线程在同步执行，但微观上只是多个线程快速切换，依次执行
>
> 并行：多个线程真正的同时执行



#### 虚拟机栈

![image-20210713110112465](https://tva1.sinaimg.cn/large/008i3skNgy1gsf52wgyrlj306i06qq5i.jpg)

与程序计数器一样，虚拟机栈也是线程私有的，它的生命周期与线程相同。**每个方法被执行的时候，JVM都会同步创建一个栈帧（`Stack Frame`）用于存储局部变量表、操作数栈、动态连接、方法出口等信息。每一个方法被调用直至执行完毕的过程，就对应着一个栈帧在虚拟机栈入栈和出栈的过程。**

> 栈是运行时的单位，堆事存储的单位
>
> 栈解决程序的运行问题，即程序如何执行，如何处理数据
>
> 堆解决数据的存储问题，即数据怎么放、放在哪里



- 不同线程中所包含的栈帧时不允许存在相互引用的，即不可能在一个栈帧之中引用另外一个线程的栈帧。

- 如果当前方法调用了其他方法，方法返回之际，当前栈帧会传回此方法的执行结果给钱一个栈帧，然后虚拟机会丢弃当前栈帧（出栈），使得前一个栈帧重新成为当前栈帧

- Java方法有两种返回函数的方式，一种是正常的函数返回（`return`），另一种是抛出异常，无论哪种方式都会导致栈帧被弹出

  > 如果方法都是一路`throw`上去的，那么异常就会一直被抛到`main`方法（最里面的栈帧成为了当前栈帧），如果`main`方法也是抛出异常，那么整个虚拟机就会挂掉（栈中最后一个栈帧`main`也弹出了，当前栈里面已经没有栈帧了）

```java
public static void main(String[] args) {
    StackFrameTest stackFrameTest = new StackFrameTest();
    stackFrameTest.method1();
}

public void method1() {
    System.out.println("method1 start...");
    method2();
    System.out.println("method1 end...");
    // 即使这里没有写return语句，字节码最后一条指令也是return。
  	// 因此所有方法都是return或抛异常返回的，这里只是省略了return语句，而不是没有return
}

public int method2() {
    System.out.println("method2 start...");
    int i = 40;
    method3();
    System.out.println("method2 end...");
    return i;
  	// 字节码最后一条指令为ireturn，意为返回int类型
}

public double method3() {
    System.out.println("method3 start...");
    double k = 20.20;
    System.out.println("method3 end...");
    return k;
  	// 字节码最后一条指令为dreturn，意为返回double类型
}
```

![image-20210713153623025](https://tva1.sinaimg.cn/large/008i3skNgy1gsfd1946ytj307d03ymx7.jpg)







`JVM`规范允许`Java`栈的大小是动态的或者是固定不变的： 

- 如果采用固定大小的`Java`虚拟机栈，那每一个线程的Java虚拟机栈容量可以在线程创建的时候独立选定。如果线程请求分配的栈容量超过`Java`虚拟机栈允许的最大容量，JVM将会抛出一个`StackOverflowError`异常
- 如果`JVM`可以动态扩展，并且在尝试扩展的时候无法申请到足够的内存，或者在创建新的线程时没有足够的内存去创建对应的虚拟机栈，那`JVM`将会抛出一个`OutOfMemoryError`异常



##### 栈帧的内部结构

![image-20210713154923162](https://tva1.sinaimg.cn/large/008i3skNgy1gsfder71wzj30ni0c0q8h.jpg)

###### 局部变量表-Local Variables

- 局部变量表存放了编译期可知的基本数据类型、引用类型和`returnAddress`类型

  > - 8种基本数据类型：byte、short、int、float、double、long、boolean、char
  > - `reference`类型并不等同于对象本身，它可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置
  > - `returnAddress`类型指向一条字节码指令的地址

- 局部变量表是建立在线程的虚拟机栈上，是线程的私有数据，不存在线程安全问题

- 局部变量表所需的内存空间是**在编译期间完成分配的**，当进入一个方法时，这个方法需要在栈帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。

  > 此处所说的大小指的是局部变量槽的数量，其真正占据多少内存空间，由虚拟机决定的每个变量槽所占的内存空间大小而决定
  >
  > 换言之，一个栈帧需要分配多少内存，并不会收到程序运行期变量数据的影响，而仅仅取决于程序源码和具体的虚拟机实现的栈内存布局形式

- 在栈帧中，与性能调优关系最密切的部分就是局部变量表。在方法执行时，虚拟机使用局部变量表完成方法的传递

- **局部变量表中的变量也是重要的垃圾回收根节点（根可达性分析），只要被局部变量表中直接或间接引用的对象都不会被回收**



- 局部变量表中，最基本的存储单元是==局部变量槽（`Slot`）==

  > 32位以内的类型只占用一个`slot`（包括`returnAddress`类型）
  >
  > 而64位的（`long`和`double`）需要占用两个slot`，`==注意⚠️：是基本数据类型才占两个`slot`，如果是包装类仍然只占一个`slot`==
  >
  > byte、short、char、boolean在存储钱被转换为`int`

  - `JVM`会为局部变量表中的每一个`Slot`都分配一个访问索引，通过这个索引可以成功访问到局部变量表中指定的局部变量值

  - 当一个实例方法被调用时，它的方法参数和方法体内部定义的局部变量将会**按照顺序被复制**到局部变量表中的每一个槽上

  - 如果需要访问局部变量表中占两个`slot`的局部变量值时，只需要使用前一个索引

  - 如果**当前帧是由构造方法或者实例方法创建的，那么该对象引用`this`（传递方法所属对象实例的引用）将会存放在`index`为0的`slot`处**，其余参数按照参数表顺序继续排列

    > ![image-20210714100334556](https://tva1.sinaimg.cn/large/008i3skNgy1gsg918lzcuj308p03874a.jpg)
    >
    > ![image-20210714100458161](https://tva1.sinaimg.cn/large/008i3skNgy1gsg92p78qoj30e704fq36.jpg)
    >
    > 从序号那一列可以看出，ll变量（`long`类型）占据1、2号`slot`，dou变量（`double`类型）占据3、4号`slot`，要使用这两个变量，只需要其起始索引即可
    >
    > ![image-20210714095533800](https://tva1.sinaimg.cn/large/008i3skNgy1gsg8swn9ccj30ex039mxg.jpg)
    >
    > ![image-20210714095542873](https://tva1.sinaimg.cn/large/008i3skNgy1gsg8t2ax4gj30cv0410sw.jpg)
    >
    > 也就是说非静态方法局部变量表的第一个`slot`一定是`this`，可以理解为一个从构造器传过来的隐参，所以静态方法不能使用`this.xxx`，因为其局部变量表里面根本就没有这个变量，并且每个方法只能引用本局部变量表里面的变量

    

  - 栈帧中的局部变量表中的槽位是**可以重复利用**的，如果一个局部变量过了其作用域，那么在其作用域之后申明新的局部变量就很有可能会复用过期局部变量的槽位，从而达到节省资源的目的

    > ```java
    > public void testReuseSlot() {
    >     int a = 0;
    >     {
    >         int b = 1;
    >         b = a + 1;
    >     }
    >     int c = a + 1;
    > }
    > ```
    >
    > 按理说该方法的局部变量表应该有this、a、b、c四个局部变量，但实际上只有3个
    >
    > ![image-20210714101746685](https://tva1.sinaimg.cn/large/008i3skNgy1gsg9g0oi4ej307p03laa3.jpg)
    >
    > 原因：
    >
    > ![image-20210714101802606](https://tva1.sinaimg.cn/large/008i3skNgy1gsg9gaxr6wj30c70580sw.jpg)
    >
    > 根据==起始PC+长度==的值判断变量的作用域：
    >
    > 1. b变量：4+4=8，意味着b变量的作用域只在大括号内有效
    > 2. this变量：0+13=13，跟字节码长度一样，意味着this变量在该方法内全局有效
    > 3. a变量：2+11=13
    > 4. c变量：12+1=13，变量c的起始PC为12>8，即其创建时变量b已经被销毁，因此变量c重复利用变量b的slot，因此这两个变量的slot的序号都是2

![image-20210714104221482](https://tva1.sinaimg.cn/large/008i3skNgy1gsga5lqn98j30pn0d3gnt.jpg)

> 因为不同于成员变量一样有一个准备阶段会默认赋零值，所以局部变量必须显式赋值



###### 操作数栈-Operand Stack

- 操作数栈的最大深度在编译时就被写入到Code属性的`max_stacks`数据项中

- 栈中的任何一个元素可以是任意的`Java`数据类型

  > `32bit`占用一个栈单位深度
  >
  > `64bit`占用两个栈单位深度

- 操作数栈中元素的数据类型必须与字节码指令的序列严格匹配

- 如果被调用的方法带有返回值，其返回值将会被压入当前栈帧的操作数栈中，并更新PC中下一条需要执行的字节码指令

- `Java`虚拟机的解释引擎是基于栈的执行引擎，这个栈指的就是操作数栈

- 一般来说，两个不同栈帧作为不同方法的虚拟机栈的元素，是完全相互独立的。但是在大多数虚拟机的实现会做一些优化，令下面栈帧的部分操作数栈与上面栈帧的部分局部变量表重叠在一起。

  > 这样做不仅节约了一些空间，还能在进行方法调用时就可以直接共用一部分数据，无需进行额外的参数复制传递了



**有关i++与++i的问题**

![image-20210715111202113](https://tva1.sinaimg.cn/large/008i3skNgy1gshgmsav8gj30e707xwfw.jpg)

```java
int i1 = 10;
i1++;

int i2 = 20;
++i2;
```

![image-20210714154858553](https://tva1.sinaimg.cn/large/008i3skNgy1gsgj0p815yj30520373yg.jpg)

> `0-bipush 10`:方法一开始PC计数器的值为0，把数10压入操作数栈
>
> *P.S:`bipush`指的是把`byte`型当作`int`型，是因为10在`byte`的范围内（-128~127，一个字节就能存储），但是存储在局部变量表的时候还是`int`型。如果超过这个范围，就会变为`sipush`（看为`short`型）*
>
> `2-istore_1`：把操作数栈顶（数10）出栈并放入局部变量表中索引为1的位置（该方法为非静态方法，第一个槽存放的是`this`）
>
> `3-iinc 1 by 1`:把局部变量表中索引为1的位置的数据+1
>
> 如果只是单纯的i`++`与`++i`，没有赋值给新的变量，在字节码角度实际上是一样的

```java
int i3 = 30;
int i4 = i3++;

int i5 = 40;
int i6 = ++i5;
```

![image-20210714154935123](https://tva1.sinaimg.cn/large/008i3skNgy1gsgj1b291vj304q052t8s.jpg)

> `15-iload_3`：取出局部变量表中索引为3的数据（也就是前面存储的30）并复制到操作数栈栈顶
>
> `j=i++`的过程：从局部变量表取出i ➡️ `i+1` ➡️ 把j放入局部变量表中
>
> `j=++i`的过程：把局部变量表中对应索引的数据`i+1` ➡️ 取出该索引位置的数据i ➡️ 把j放入局部变量表中

```java
int i7 = 50;
i7 = i7++;

int i8 = 60;
i8 = ++i8;
```

![image-20210714155014698](https://tva1.sinaimg.cn/large/008i3skNgy1gsgj1zm82wj305d053wek.jpg)

> `i=i++`的过程：从局部变量表取并复制到操作数栈栈顶`load` ➡️ `i+1` ➡️ 操作数栈栈顶出栈并放回局部变量表`store`
>
> `i=++i`的过程：`i+1` ➡️ 从局部变量表取并复制到操作数栈栈顶`load` ➡️ 操作数栈栈顶出栈并放回局部变量表`store`



###### 动态连接-Dynamic Linking

- 每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态链接

- `Class`文件的常量池中存有大量的符号引用，字节码中的方法调用指令就以常量池里指向方法的符号引用作为参数

  - 静态解析：这些符号引用一部分会在**类加载阶段（`Linking`阶段中的解析`Resolve`阶段）或者第一次使用的时候**就被转化为直接引用

    条件：当一个字节码文件被装载进JVM内部时，被调用的目标方法在编译期可知，并且运行期保持不变

  - 动态链接：另一部分将在**每一次运行期间**都转化为直接引用

    条件：被调用的方法在编译期无法确定下来

  > 符号引用就是一组符号来描述所引用的目标；直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄
  >
  > 早期绑定与晚期绑定的概念与静态/动态解析相似，只是绑定的范围更大一些，可以是字段、方法或类，而连接只针对方法
  >
  > 如：面向接口编程，传进来的参数是一个接口。公共父类，那么在编译的时候是不可能知道实际上使用的是这个接口的哪个实现类的，因此只能晚期绑定➡️如果一个编程语言具备多态特性，那么自然也就具备早期绑定和晚期绑定两种绑定方式

```java
int num = 10;

public void methodA() {
    System.out.println("A");
}

public void methodB() {
    System.out.println("B");
    methodA();
}
```

![image-20210714162635218](https://tva1.sinaimg.cn/large/008i3skNgy1gsgk3rf4e6j30cs06bq3n.jpg)

> `invokeVirtual #5`是对 `System.out.println()` 方法的引用
>
> `invokeVirtual #7`是对`methodA`的引用

![image-20210714163451888](https://tva1.sinaimg.cn/large/008i3skNgy1gsgkcdwkg0j30oj0d5gnu.jpg)



###### 方法出口/方法返回地址-Return Address

- 存放调用该方法的PC计数器的值

  > 方法退出之后，都必须返回到最初方法被调用时的位置，程序才能继续执行

  ![image-20210715094417483](https://tva1.sinaimg.cn/large/008i3skNgy1gshe3hqpaxj30lj085q5c.jpg)

- 方法退出方式

  - 正常调用完成——执行引擎遇到任意一个方法返回的字节码指令

  - 异常调用完成——在方法执行过程中遇到了异常，并且这个异常没有在方法体内得到妥善处理，也就是在本方法的异常表中没有搜索到匹配的异常处理器

    > 一个方法使用异常完成出口的方式退出，是不会给它的上层调用者提供任何返回值的
    >
    > 方法异常退出时，返回地址是要通过异常处理表来确定的，栈帧中就一般不会保存着部分信息



##### 方法调用

> Java强大的动态扩展能力的来源：Class文件的编译过程中不包含传统程序语言编译的连接步骤，一切方法调用在Class文件里面存储的都只是在常量池中的符号引用，而不是方法在实际运行时内存布局中的入口地址（也就是直接引用）

在`Java`语言中符合“编译期可知，运行期不可变”的主要有静态方法和私有方法两大类。前者与类型直接相关，后者在外部不可被访问，这两种方法个字的特点决定了它们都不可能通过继承或别的方式重写，因此都适合在类加载阶段进行解析（即静态连接）

方法调用字节码指令：

- `invokestatic`：调用静态方法
- `invokespecial`：调用实例构造器`<init>()`方法、私有方法和父类中的方法
- `invokevirtual`：调用锁哟噗的虚方法
- `invokeinterface`：调用接口方法，会在运行时再确定一个实现类
- `invokedynamic`：先在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法。

> 只要能被`invokestatic`和`invokespecial`指令调用的方法都可以在解析阶段中确定唯一的调用版本（即可以静态连接），符合这个条件的有：**静态方法、私有方法、实例构造器、父类方法**四种，再加上**用final修饰的方法**（尽管它仍然适用`invokevirtual`指令调用）。这五种方法调用可以静态连接，统称为非虚方法，其余的称为虚方法。



```java
// 假设有两个类Man、Woman，它们都继承Human类。类中有一个方法sayHello进行了重载，参数分别为Human、Man、Woman
Human man = new Man();
Human woman = new Woman();
sayHello(man);
sayHello(woman)
```

实际上4、5执行的都是参数为`human`的`sayHello`方法

> 上面代码中的Human称为变量的静态类型（Static Type），或者叫外观类型（Apparent Type）；Man、Woman称为变量的实际类型（Actual Type），或者叫运行时类型（Runtime Type）。
>
> 静态类型和实际类型在程序中都可能会发生变化，区别是静态类型的变化仅仅在使用时发生，变量本身的静态类型不会改变，并且最终的==静态类型是在编译期可知的==；而实际类型变化的结果在运行期才可确定，编译器在编译程序的时候并不知道一个对象的实际类型是什么。

==虚拟机在重载时时**通过参数的静态类型**而不是实际类型作为判断依据的==。由于静态类型在编译期可知，所以在编译阶段，`javac`编译器就根据参数的静态类型决定了实际上使用的是哪个重载版本的方法（即`sayHello(Human)`这个版本），并把这个方法的符号引用写到主调方法里的两条`invokevirtual`指令的参数中

而如果是如下代码，则会各自调用`Man`、`Woman`版本的`sayHello()`方法，因为这个强转在编译期是可知的

```java
sayHello((Man)human);
sayHello((Woman)human);
```



```java
static abstract class Human {
  protected abstract void sayHello();
}

static class Man extends Human {
  @Override
  protected void sayHello() {
    System.out.println("man sayhello");
  }
}

static class Woman extends Human {
  @Override
  protected void sayHello() {
    System.out.println("woman sayhello");
  }
}

public static void main(String[] args) {
  Human man = new Man();
  Human woman = new Woman();
  man.sayHello();
  woman.sayHello();
  man = new Woman();
  man.sayHello();
}
```

![image-20210715103309800](https://tva1.sinaimg.cn/large/008i3skNgy1gshfic8dloj30670260sk.jpg)

![image-20210715103405284](https://tva1.sinaimg.cn/large/008i3skNgy1gshfjapf7yj30dc09k3zz.jpg)

1-8行是准备动作，作用是建立`man`和`woman`的内存空间、调用`Man`和`Woman`的实例构造器，把这两个实例的引用存放在第1、2个局部变量表的`slot`中。

9、11行是分别把刚刚创建的两个对象的引用压入栈顶

10和12行的`invokevirtual`指令是确定调用方法版本的关键，其运行时解析过程如下：

1. 找到操作数栈顶的第一个元素所指向的对象的**实际类型**C
2. 如果在类型C中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验。如果通过则返回这个方法的直接引用，查找过程结束；不通过则返回`java.lang.IllegalAccessError`异常
3. 否则，按照继承关系从下往上依次对C的各个父类进行第二步的搜索和验证过程
4. 如果始终没有找到合适的方法，则抛出`java.lang.AbstractMethodError`异常

> 正是因为`invokevirtual`指令执行的第一步就是在运行期确定接受者的实际类型，所以两次调用汇总的`invokevirtual`指令并不是把常量池中方法的符号引用解析到直接引用上就结束了，还会根据方法接受者的实际类型来选择方法版本，这个过程就是`Java`语言中方法重写的本质。



#### 本地方法栈

![image-20210713110129463](https://tva1.sinaimg.cn/large/008i3skNgy1gsf5377fe6j303t06nwfk.jpg)

`Java`语言是不能直接操作硬件的，但是线程的运行就需要和硬件打交道，所以只能通过`Native`方法来调用`C/C++`来操作硬件

- 虚拟机栈用于管理`Java`方法的调用，而本地方法栈用于管理本地方法的调用
- 本地方法栈也是线程私有的
- 允许被实现成固定或者是可动态扩展的内存大小（与虚拟机栈类似）
- 它的具体做法是本地方法栈中登记本地方法，在执行引擎执行时加载本地方法库



#### 堆

![image-20210713110145929](https://tva1.sinaimg.cn/large/008i3skNgy1gsf53hh4j7j302u060mxz.jpg)

对于`Java`应用程序来说，`Java`堆是虚拟机所管理的内存中最大的一块，`Java`堆是被**所有线程共享**的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是==存放对象实例==，`Java`里几乎所有的对象实例都在这里分配内存。

- 堆可以处于物理上不连续的内存空间中，但在逻辑上它应该被视为连续的

- 如果从分配内存的角度看，所有线程共享的`Java`堆中可以划分出**多个线程私有的分配缓冲区**`TLAB（Thread Local Allocation Buffer）`（包含在Eden空间内），以提升对象分配时的效率，提升内存分配的吞吐量

  > 为什么有`TLAB`？
  >
  > - 堆区是线程共享区域，任何线程都可以访问到堆区中的共享数据
  > - 由于对象实例的创建在`JVM`中非常频繁，因此在并发环境下从堆区中划分内存空间是线程不安全的
  > - 为避免多个线程操作同一地址，需要使用加锁等机制，从而影响了分配速度
  >
  > 无论如何，都不会改变`Java`堆中存储内容的共性，无论是哪个区域，存储的都只能是对象的实例，把`Java`堆细分的目的只是为了更好的回收内存，或者更快的分配内存

- 堆的大小在`JVM`启动时就已经设定好了，可以通过以下两个参数来进行设置

  `-Xms`：用来设置堆区的起始内存

  `-Xmx`：用于设置堆区的最大内存

  > 一旦堆区中的内存大小超过`-Xmx`所指定的最大内存时，将会抛出`OutOfMemory`异常（这里所说的堆内存指的是新生代+老年代）
  >
  > 通常会将`-Xms`和`-Xmx`两个参数配置相同的值，其目的是为了能够在Java垃圾回收机制清理完堆区后不需要重新分配计算堆区的大小，从而提高性能



##### 堆内存划分

![image-20210715195448374](https://tva1.sinaimg.cn/large/008i3skNgy1gshvqrgmxlj30gd06gq3j.jpg)

> 新生代与老年代在堆结构占比默认为**1:2**
>
> 修改参数`-XX:NewRatio=3`，表示新生代占1，老年代占3，新生代占整个堆的1/4
>
> `Eden`区和两个`Suvivor`区的比例默认为8:1:1
>
> 修改参数`-XX:SuvivorRatio=8`，表示两个`Suvivor`区各占1，`Eden`区占8
>
> `-Xmn`设置新生代的空间的大小

- 几乎所有的对象都是在Eden区被new出来的
- 绝大部份的对象的销毁都在新生代进行，很少GC老年代，几乎不动永久代/元空间



##### 对象分配过程

1. `new`的对象尝试放入`Eden区`

2. 如果`Eden区`已满，则把`Eden区`仍然存活的对象复制入`S0区`

3. GC `Eden区`

4. `new`的对象放入`Eden区`

   ![image-20210716104706588](https://tva1.sinaimg.cn/large/008i3skNgy1gsilj5swlhj30ly08s76b.jpg)

5. 再次触发GC：把`S0区`和`Eden区`仍然存活的对象复制入`S1区`

6. GC `Eden区`和`S0区`

   ![image-20210716104720839](https://tva1.sinaimg.cn/large/008i3skNgy1gsiljemym3j30kn09adi1.jpg)

   7. 当`S区`中的对象年龄达到阈值（默认为15）时，则可以晋升到老年代

   > 该阈值可以通过参数`-XX:MaxTenuringThreshold=N`设置

   ![image-20210716104736791](https://tva1.sinaimg.cn/large/008i3skNgy1gsiljofazaj30p708xmzh.jpg)

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gshxoi7g8xj30u01lcwlu.jpg" alt="image-20210715210151910" style="zoom: 50%;" />

> ⚠️如果对象超大，新生代放不下，则直接放入老年代
>
> 如果本次`GC`时`Survivor`满了，则直接晋升到老年代

![image-20210716095337506](https://tva1.sinaimg.cn/large/008i3skNgy1gsijzi8fq2j30gl0aigo9.jpg)



##### 常用参数设置

|            参数            |                          作用                          |
| :------------------------: | :----------------------------------------------------: |
|   -XX:+PrintFlagsInitial   |               查看所有的参数的默认初始值               |
|    -XX:+PrintFlagsFinal    | 查看所有的参数的最终值（可能会存在修改，不再是初始值） |
|            -Xms            |         初始堆空间内存（默认为物理内存的1/64）         |
|            -Xmx            |         最大堆空间内存（默认为物理内存的1/64）         |
|            -Xmn            |           设置新生代的大小（初始值及最大值）           |
|        -XX:NewRatio        |            配置新生代与老年代在堆结构的占比            |
|     -XX:SurvivorRatio      |          设置新生代中Eden区和S0/S1空间的占比           |
|  -XX:MaxTenuringThreshold  |   设置新生代垃圾的最大年龄（晋升为老年代的年龄阈值）   |
|    -XX:+PrintGCDetails     |                  输出详细的GC处理日志                  |
|  -XX:+PrintGC/-verbose:gc  |                     打印GC简要信息                     |
| -XX:HandlePromotionFailure |                  是否设置空间分配担保                  |

> `jps`：查看当前运行中的进程
>
> `jinfo -flag SurvivorRatio 进程ID`：查看某个进程的`SurvivorRatio`参数



##### 内存分配策略

- 优先分配到`Eden区`

- 大对象（指的是大于`Eden区`可用内存）直接分配到老年代

  > 因此要尽量避免程序中出现过多的大对象，如果这些大对象只使用很短的时间，那么出发Major GC就得不偿失

- 长期存活的对象分配到老年代

- 动态对象年龄判断：如果`Survivor区`中相同年龄的所有对象大小的总和大于`Survivor`空间的一半，则年龄大于等于该年龄的对象可以直接进入老年代，无需等到`MaxTenuringThreshold`中要求的年龄



#### 方法区

![image-20210713110138395](https://tva1.sinaimg.cn/large/008i3skNgy1gsf53cw9bhj302x061757.jpg)

方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它==用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据==。



##### 栈、堆、方法区的交互关系

![image-20210716161631259](https://tva1.sinaimg.cn/large/008i3skNgy1gsiv1wq6ykj30b803fjrn.jpg)



##### 永久代与元空间

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsj1y9xijlj30s20cwt9y.jpg" alt="image-20210716201513326" style="zoom: 67%;" />

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsj1z80sn9j30ru0cldh2.jpg" alt="image-20210716201608036" style="zoom: 67%;" />

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsj1zy748bj30xb0cg0u6.jpg" alt="image-20210716201649801" style="zoom: 67%;" />

> 方法区是一种规范，可以理解为接口，而永久代和元空间相当于两个不同的实现类（只有`HotSpot`才有永久代）
>
> JDK7及以前为**永久代（`PermGen`）**，JDK8及以后为**元空间（`Metaspace`）**

###### 永久代

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsdwvsh46zj30lb0a8jrt.jpg" alt="img" style="zoom:80%;" />

==永久代与堆中的老年代是连续的（指物理地址上，永久代本身不在堆中）==，因此，老年代与永久代其中一个满了，都会触发`Full GC`

在Java7时，仍然有永久代，永久代也与堆中的老年代连续，但永久代中存储的部分数据已经开始转移到`Java Heap`或`Native Memory`中了，比如：

- 符号引用(`Symbols`)转移到了`Native Memory`

- 字符串常量池(`interned strings`)转移到了`Java Heap`

  > 永久代的回收效率很低，在`Full GC`时才会触发，而`Full GC`是老年代空间不足、永久代空间不足时才会触发。
  >
  > 这就导致`StringTable`回收效率不高，而开发中会有大量的字符串被创建，回收效率低，导致永久代内存不足。放到堆中就能及时回收内存

- 类的静态变量(`class statics`)转移到了`Java Heap`

由于方法区主要存储类的相关信息，所以对于动态生成类的情况比较容易出现永久代的内存溢出。最典型的场景就是，在 `jsp` 页面比较多的情况，容易出现永久代内存溢出，会报出`java.lang.OutOfMemoryError: PermGen space` 异常。

> 我们可以使用以下的命令，来显示指定永久代的大小：
>
> - `-XX:PremSize`：设置永久代的初始大小
> - `-XX:MaxPermSize`: 设置永久代的最大值 

###### 元空间

元空间的本质和永久代类似，都是对`JVM`规范中方法区的实现。不过元空间与永久代之间最大的区别在于：==元空间并不在虚拟机中（不再与堆连续），而是使用本地内存==。因此，默认情况下，==元空间的大小仅受本地内存限制==。

- 类型信息、字段、方法、常量保存在本地内存的元空间
- 字符串常量池、类的静态变量仍在堆中

`Class`对象存放在堆区，类的元数据存放在元空间。（`Class`对象是加载的最终产品；类的方法代码、变量名、方法名、访问权限等在元空间）

> 但可以通过以下参数来指定元空间的大小：
>
> - `-XX:MetaspaceSize`，初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时`GC`会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过`MaxMetaspaceSize`时，适当提高该值。
> - `-XX:MaxMetaspaceSize`，最大空间，默认是没有限制的。
>
> 除了上面两个指定大小的选项以外，还有两个与 `GC` 相关的属性：
>
> - `-XX:MinMetaspaceFreeRatio`，在GC之后，最小的`Metaspace`剩余空间容量的百分比，减少为分配空间所导致的垃圾收集
> - `-XX:MaxMetaspaceFreeRatio`，在GC之后，最大的`Metaspace`剩余空间容量的百分比，减少为释放空间所导致的垃圾收集
>
> 在使用`-XX:MaxMetaspaceSize`显示指定元空间的大小为一个比较小的值，接着在循环中动态加载过多的类，那么会报出"`java.lang.OutOfMemoryError: Metaspace`"异常。
>
> 在Java8时，仍然使用`PremSize或MaxPermSize`设置永久代的大小时，会被编译器忽略并且警告。

###### 转换原因

1. 字符串存在永久代中，容易出现性能问题和内存溢出
2. 类及方法的信息等比较难确定其大小，因此对于永久代的大小指定比较困难，太小容易出现永久代溢出，太大容易导致老年代溢出
3. 永久代会为`GC`带来不必要的复杂性，并且回收效率偏低（`Full GC`）



##### 方法区的内部结构

1. 类型信息：
   - 这个类型的完整有效名称（全限定类名）
   - 这个类型直接父类的完整有效名称（对于`interface`或是`java.lang.Object`，都没有父类）
   - 这个类型的修饰符（`public`、`abstract`、`final`的某个子集）
   - 这个类型直接接口的一个有序列表
2. 域信息（按声明顺序）：域名称、域类型、域修饰符（`public`、`private`、`protected`、`static`、`final`、`volatile`、`transient`的某个子集）
3. 方法信息（按声明顺序）：
   - 方法名称
   - 方法的返回类型
   - 方法参数的数量和类型（按顺序）
   - 方法的修饰符（`public`、`private`、`protected`、`static`、`final`、`synchronized`、`native`、`abstract`的一个子集）
   - 方法的字节码、操作数栈、局部变量表及大小（`abstract`和native`方法`除外）
   - 异常表（`abstract`和`native`方法除外）：每个异常处理的开始位置、结束位置、代码处理在程序计数器中的偏移地址、被捕获的异常类的常量池索引



```java
public static void main(String[] args) {
        Order order = null;
        order.hello();
        System.out.println(order.count);
    }

class Order {
    public static int count = 1;
    public static final int number = 2;

    public static void hello() {
        System.out.println("HELLO");
    }
}
```

![image-20210716180035326](https://tva1.sinaimg.cn/large/008i3skNgy1gsiy2764ucj30ac04x74c.jpg)

![image-20210716180343735](https://tva1.sinaimg.cn/large/008i3skNgy1gsiy5gbnvxj30ed06udfx.jpg)

单纯的静态变量`count`在`Prepare`阶段赋值为零，在`Initialization`阶段显式赋值为1

声明为`final`的常量`number`直接在`Prepare`阶段显式赋值为2



##### 运行时常量池

- 常量池表用于==存放编译期生成的各种字面量与符号引用==，这部分内容将在类加载后存放到方法区的运行时常量池中
- 在加载类和接口到虚拟机后，就会创建对应的运行时常量池
- `JVM`为每个已加载的类型（类或接口）都维护一个常量池。池中的数据项像数组项一样，是通过索引访问的
- 当创建类或接口的运行时常量池时，如果构造运行时常量池所需的内存空间超过了方法区所能提供的最大值，则`JVM`会抛`OutOfMemoryError`

> 运行时常量池相对于`Class`文件常量池的另外一个重要特征是具备**动态性**，`Java`语言并不要求常量一定只有编译期才能产生，也就是说，并非预置入`Class`文件中常量池的内容才能进入方法区运行时常量池，运行期间也可以将新的常量放入池中，这种特性被开发人员利用的比较多的是`String.intern()`



#### 直接内存

- 不是虚拟机运行时数据区的一部分，也不是`JVM`规范中定义的内存区域

- 直接内存是在`Java`堆外的、直接向系统申请的内存区间

- 来源于`NIO`，通过存在堆中的`DirectByteBuffer`操作`Native`内存

- 通常，访问直接内存的速度会优于`Java`堆，即读写性能高

  - 处于性能考虑，读写频繁的场合可能会使用直接内存

  - `Java` 的`NIO`库允许`Java`程序使用直接内存，用于数据缓冲区

    > ByteBuffer.allocateDirect(int capacity)



#### 对象创建过程

```java
Object o = new Object();
```

![image-20210717111407789](https://tva1.sinaimg.cn/large/008i3skNgy1gsjrxl4owlj30ja04dq2y.jpg)

`new #2`：`#2→#20`是常量池中的`Object`的`Class`对象

> 先判断是否已经加载了`Object`类，如果没有则要先使用类加载器把该类加载进来

`dup`：`duplicate`复制，把操作数栈中生成的变量的引用复制一份，使得有两个引用指向堆空间的实体

> 栈顶的引用主要用于赋值操作；下面的引用作为句柄，主要用于调用相关的方法

`invokespecial #1`：`#1→#2、#19`，分别指向常量池中的`Object`的`Class`和其`<init>()`方法和参数列表，该条指令的作用是执行`Object`的`<init>`方法（调用构造器）

`astore_1`：把生成的变量从操作数栈中取出来（栈顶），放入局部变量表中索引为1的位置。

> 索引为0的位置是`main`方法的参数`args`，局部变量表的长度为`locals=2`

![image-20210717112502418](https://tva1.sinaimg.cn/large/008i3skNgy1gsjs8yuv8dj30tn0amwfn.jpg)

![image-20210717112426632](https://tva1.sinaimg.cn/large/008i3skNgy1gsjs8chstqj30xw0ep401.jpg)

1. 判断对象对应的类是否加载、连接、初始化

   虚拟机遇到一条`new`指令，首先去检查这个指令的参数能否在`Metaspace`的常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已经被加载、连接、初始化（即判断类信息是否存在）。如果没有，则必须先执行类加载过程。

   > 在双亲委派模式下，使用当前类加载器以`ClassLoader`+包名+类名为`Key`进行查找对应的`Class`文件。如果没有找到文件，则抛出`ClassNotFoundException`；如果找到，则进行类加载，并生成对应的`Class`类对象

2. 为对象分配内存

   对象所需内存的大小在类加载完成后便可完全确定，从`Java`堆中划分一块内存给新对象就是为其分配内存。

   > 如果实例成员变量是引用变量，仅分配引用变量空间即可，即4字节

   - **指针碰撞**（`Bump The Pointer`）：如果`Java`堆中内存是**绝对规整**的，即所有被使用过的内存被放在一边，空闲的内存放在另一边，中间放置一个**指针作为分界点的指示器**，那所分配内存就仅仅是把那个指针向空闲空间方向挪动一段与对象大小相等的距离

   - **空闲列表**（`Free List`）：如果`Java`堆中的内存并**不是规整**的，已被使用的内存和空闲的内存相互交错在一起，那虚拟机必须维护一个列表，记录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录

     > `Java`堆是否规整由所采用的垃圾收集器是否带有空间压缩整理（`Compact`）能力决定

3. 处理并发安全

   - 对分配内存空间的动作进行同步处理——实际上虚拟机是采用`CAS`配上失败重试的方式保证更新操作的原子性

   - 把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在`Java`堆中预先分配一小块内存，称为本地线程分配缓冲（`Thread Local Allocation Buffer,TLAB`），哪个线程要分配内存，就在哪个线程的本地缓冲区中分配，只有本地缓冲区用完了，分配新的缓冲区时才需要同步锁定

     > 是否使用`TLAB`可以通过`-XX:+/-UseTLAB`来设定

4. 初始化分配到的空间

   内存分配完成后，虚拟机必须将分配到的内存空间（但不包括对象头）都初始化为零值，保证了对象的实例字段在`Java`代码中可以不赋初始值就直接使用，使程序能访问到这些字段的数据类型所对应的零值

   > 如果使用`TLAB`，可以把这一步提前到`TLAB`分配时顺便进行

5. 设置对象的对象头

   ==对象头存储对象的所属类（类的元数据信息）、对象的`HashCode`（实际上会延后到真正调用`Object::hashCode()`方法时才计算）、对象的`GC`分代年龄等信息。==

6. 执行`init`方法进行初始化

   从虚拟机来看，至此一个新的对象已经产生了；但是从`Java`程序来看，对象创建才刚刚开始——构造函数，即`Class`文件中的`<init>()`方法还没有执行（为对象进行显式赋值）。



#### 对象的内存布局

![image-20210717171135041](https://tva1.sinaimg.cn/large/008i3skNgy1gsk29k1m9pj31ai0ir40r.jpg)

```java
// Customer cust = new Customer();
public class Customer {
	int id = 1001;
	String name;
	Account acct;
	
	{
		name="匿名客户";
	}
	
	public Customer() {
		acct = new Account();
	}
}

class Account {

}
```

![image-20210720163256591](https://tva1.sinaimg.cn/large/008i3skNgy1gsni09wcraj311r0j8q6k.jpg)

> 类型指针指向当前对象属于哪个类——方法区中的`Class`类元信息
>
> `name`被赋了一个字符串常量，在JDK7以后转移到了堆空间
>
> `acct`引用了一个对象，指向堆空间中的`new Account()`实例，`Account`中也维护了一个类型指针指向方法区



#### 对象的访问定位

1. 句柄访问

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsnnz0hc9yj30vu0ibjst.jpg" alt="image-20210720195922158" style="zoom:67%;" />

   局部变量表内存放了对象引用`reference`，堆空间中开辟了一块区域为句柄池，每个句柄包含两个信息：到对象实例数据的指针（堆区中）和到对象类型数据的指针（方法区中）

   > `reference`中存储稳定句柄地址，对象被移动（垃圾收集时移动对象很普遍）时只改变句柄中实例数据指针即可，`reference`本身不需要被修改

2. 直接指针（HotSpot采用）

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsnoag4l7xj30ue0h2wfn.jpg" alt="image-20210720201021255" style="zoom:67%;" />

   局部变量表内存放了对象引用`reference`，直接指向堆空间中的对象实例，对象实例的对象头内有一个类型指针，指向方法区内该对象对应的类型数据

   > 句柄方式需要专门开辟一块内存空间存放句柄，并且需要多次寻址，性能相对较弱



#### 逃逸分析

> 在`Java`的编译体系中，一个`Java`的源代码文件变成 计算机可执行的机器指令的过程中，需要经过两段编译：
>
> 1. `.java`文件→`.class`文件：就是静态编译（`javac`、`ECJ`等前端编译器）
> 2. `.class`文件→机器指令：就是动态编译（`JIT`编译器）
>
> 随着`JIT`编译器的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会对内存分配策略造成一定的变化，所以并不是所有对象都一定分配到堆上。

逃逸分析的基本原理就是：分析对象动态作用域，当一个对象在方法里面被定义后，它可能被外部方法所引用，例如：

- 方法逃逸：作为调用参数传递到其他方法中
- 线程逃逸：赋值给可以在其他线程中访问的实例变量

##### 栈上分配

如果确定一个对象不会逃逸出线程之外，那让这个对象在栈上分配内存可以减轻垃圾收集子系统的压力：分配到栈上的对象会随着方法的结束，跟着栈帧出栈而销毁，无需`GC`。

==**栈上分配可以支持方法逃逸，但不能支持线程逃逸**==

##### 标量替换

- 标量`Scalar`：一个无法再分解成更小的数据的数据（如原始数据类型以及`reference`类型）
- 聚合量`Aggregate`：还能进一步分解的数据

在`JIT`阶段，如果经过逃逸分析，发现一个对象不会被外界访问的话，那么经过`JIT`优化，就会把这个对象拆解成若干个其中包含的若干个成员变量来代替。这个过程就是标量替换。

将对象拆分后，除了可以让对象的成员变量在栈上分配之外，还可以为后续进一步的优化手段创建条件。

==**标量替换实际上是栈上替换的一种特例，但它对逃逸程度的要求更高，它不允许对象逃逸出方法范围内**==

##### 同步消除

在动态编译同步块的时候，JIT编译器可以借助逃逸分析来判断同步块所使用的锁对象是否只能够被一个线程访问而没有被发布到其他线程。

如果同步块所使用的锁对象通过这种分析被证实只能够被一个线程访问，那么`JIT`编译器在编译这个同步块的时候就会取消对这部分代码的同步。这个取消同步的过程就叫同步消除，也叫锁消除。

##### 为什么逃逸分析不能在静态编译期进行？

`Java`有很多小方法、很多虚方法调用、难以静态分析。而逃逸分析恰恰需要在比较大块的代码上工作才比较有效：编译器要能够看到更多的代码，以便更准确的判断对象有没有逃逸。只保守的在小块代码上分析的话，很多时候都只能得到“对象逃逸了”的判断，就没啥效果了。

> 简单来说：就是`Java`的分离编译+动态加载



#### 扩展

1. 为什么会有“不使用的对象应手动赋值为`null`“的说法？

   因为局部变量槽的可复用性以及局部变量表是GC中重要的`GC Root`。

   一个变量能否被回收的根本原因：局部变量表中的slot是否还存有关于该变量的引用。如果一段代码已经离开了这个变量的作用域，即该变量的`slot`可以被重复利用，但是在此之后再也没有发生过任何对局部变量表的读写操作（没有定义新的变量），那么该变量就会一直占用该`slot`，所以作为`GC Roots`一部分的局部变量表仍然保持对它的关联，也就不会被`GC`。

   这种关联没有及时被打断，在大多数情况下没什么影响。但是如果后面的代码有一些耗时很长的操作，前面又定义了一堆占用很大内存实际上不会再被使用的变量，手动赋为`null`值还是有意义的。

   > 但由于即时编译的存在，赋值为null的操作很有可能会被当作无效操作消除掉。所以说到底这个手动赋值为null没啥必要

2. 为什么在`static`方法中不可以引用`super`、`this`？

   非静态方法的局部变量表的第一个`slot`一定是`this`，即用于传递方法所属对象实例的引用；而静态方法并没有，即该方法内根本不存在`this`这个变量，而每个方法只能使用自己的局部变量表内的变量，自然无法使用`this`。

   静态方法在类加载时就已经存在了，但是对象是在创建时才分配到堆中，也就是说静态方法的出现时机比`this`要早，在一个先存在的方法尝试去引用一个可能未存在的对象，显然是不符合逻辑的。



### 执行引擎 

#### 代码编译和执行的过程

<img src="/Users/wingsiwoo/Library/Application Support/typora-user-images/image-20210721161158888.png" alt="image-20210721161158888" style="zoom:80%;" />

> 橙色部分是前端编译器，与JVM无关
>
> 绿色部分是解释器
>
> 蓝色部分是JIT编译器

解释器：当JVM启动时会根据预定义的规范**对字节码采用逐行解释**的方式执行，将每条字节码文件中的内容“翻译”为对应平台的本地机器指令执行 

> ==当程序需要迅速启动和执行时，解释器可以首先发挥作用，省去编译的时间，立即运行==；
>
> 当程序运行环境中内存资源限制较大，可以使用解释执行节约内存

JIT（Just In Time Compiler）编译器：就是虚拟机**将源代码直接编译成**和本地机器平台相关的机器语言

> 优点：==当程序启动后，随着时间的推移==，编译器逐渐发挥作用，把越来越多的代码编译成本地代码，这样可以==减少解释器的中间损耗，获得更高的执行效率；==
>
> 当程序运行环境中内存资源限制较小，可以使用编译执行来提升效率



##### 编译模式的设置

![image-20210726151323173](https://tva1.sinaimg.cn/large/008i3skNgy1gsudfauhlyj30gx0d5q42.jpg)



#### JIT即时编译器

热点代码：在运行过程中会被即时编译器编译的目标，主要有两类：被多次调用的方法、被多次执行的循环体

> 对于这两种情况，编译的目标对象都是整个方法体，而不会是单独的循环体。
>
> 标准的即时编译方式：第一种情况，由于是依靠方法调用触发的编译，那编译器理所应当会以整个方法作为编译对象
>
> 栈上替换（`On Stack Replacement,OSR`）：编译发生在方法执行的过程中，方法的栈帧还在栈上，方法就被替换了。针对第二种情况，尽管编译动作是由循环体所触发的，热点只是方法的一部分，但编译器依然必须以整个方法作为编译对象，只是执行入口（从方法第几条字节码指令开始执行）会稍有不同，编译时会传入执行入口点字节码序号（`Byte Code Index,BCI`）。

##### 热点探测判定方式：

1. 基于采样的热点探测

   采用这种方法的虚拟机会周期性的检查各个线程的调用栈顶，如果发现某个方法经常出现在栈顶，那这个方法就是“热点方法”。

   > 好处：实现简单高效，还可以很容易的获取方法的调用关系（将调用堆栈展开即可）
   >
   > 缺点：很难精确的确认一个方法的热度，容易因为受到线程阻塞或别的外界因素的影响而扰乱热点探测

2. 基于计数器的热点探测（`HotSpot`采用）

   采用这种方法的虚拟机会为每个方法（甚至每个代码块）建立计数器，统计方法的执行次数，如果执行次数超过一定的阈值就认为它是“热点方法”。

   > 优点：统计结果更加精确严谨
   >
   > 缺点：实现起来相对麻烦（要为每个方法建立计数器），不能直接获取到方法的调用关系

   `HotSpot`为每个方法准备了两类计数器

   - ==方法调用计数器==：用于统计方法被调用的次数

     当一个方法被调用时，会先检查该方法是否存在被JIT编译过的版本，如果存在，则优先使用编译后的本地代码来执行；如果不存在，则将此方法的调用计数器+1，然后判断**方法调用计数器与回边计数器之和是否超过方法调用计数器的阈值**。如果超过，那么将会向即时编译器提交一个该方法的代码编译请求

     > 默认阈值在`Client`模式下是1500次，在`Server`模式下是10000次。
     >
     > - `-client`：指定`JVM`运行在`Client`模式下（64位版本设置了也会被忽略掉，仍然使用`server`模式），并使用C1（Client Compiler）编译器；会对字节码进行**简单和可靠的优化，耗时短**，以达到更快的编译速度
     > - `-server`：指定`JVM`运行在`Server`模式下（默认），并使用C2（Server Compiler）编译器；C2进行**耗时较长的优化，以及激进优化**，但优化的代码执行效率更高
     >
     > 可以通过`-XX:CompileThreshold`来设定阈值
     >
     > 默认设置下，方法调用计数器统计的是一个相对的执行频率，即**一段时间内方法被调用的次数**。当超过一定的时间限度，如果还没有达到阈值的话，那该方法的调用计数器值就会减少一半，称为方法调用计数器**热度的衰减**，这段时间就称为此方法统计的**半衰周期**，进行热度衰减的动作是在虚拟机进行垃圾收集时顺便进行的，可以通过`-XX:-UseCounterDecay`来关闭热度衰减（*这样的话只要系统运行时间足够长，到最后所有的代码都会被JIT编译*），还可以使用`-XX:CounterHalfLifeTime`设置半衰周期时间，单位是秒

     ![image-20210726150158759](https://tva1.sinaimg.cn/large/008i3skNgy1gsud3guudqj30mv0j5abi.jpg)

   - ==回边计数器==：用于统计循环体执行的循环次数

     在字节码中遇到控制流向后跳转的指令称为回边



#### 垃圾回收

##### 基本概念

- 垃圾是指**在运行程序中没有任何指针指向的对象**，这个对象就是需要被回收的垃圾

- 如果不及时对内存中的垃圾进行清理，那么这些垃圾对象所占用的内存空间会一直保留到应用程序结束，被保留的空间无法被其他对象使用，甚至可能导致内存溢出

- 程序计数器、虚拟机栈、本地方法栈三个区域随线程而生，随线程而灭，栈中的栈帧随着方法的进入和退出而有条不紊的执行着出栈和入栈操作，并且每个栈帧分配多少内存基本上是在类结构确定下来时就已知的，因此这几个区域的内存分配和回收都具备确定性。

  而**`Java`堆和方法区**这两个区域则具有不确定性，一个接口的多个实现类需要的内存可能会不一样，一个方法所执行的不同条件分支所需要的内存也可能不一样，因此只有处于运行期间才能知道，`GC`针对的也是这两部分区域。



##### 标记阶段——垃圾标记算法

###### 引用计数算法

为每个对象保留一个引用计数器，用于记录对象被引用的情况。有一个地方引用它，计数器值就加一；当引用失效时，计数器值就减一；任何时刻计数器为零的对象就是已经死亡的→垃圾。

- 优点：实现简单，垃圾对象便于辨识；判定效率高，回收没有延迟性
- 缺点：
  1. 需要单独的字段存储计数器，增加了存储空间的开销
  2. 每次赋值都需要更新计数器，增加了时间开销
  3. ==无法处理循环引用的情况==，如`A`引用`B`，`B`引用`A`，除此之外没有任何引用。实际上这两个对象已经不可能再被访问，但由于循环引用的存在，引用计数器值一直为1，导致这两个对象不会被视为垃圾，也就不会被回收。因为这个致命缺陷大部分垃圾回收器都没有使用这类算法

###### 可达性分析算法

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gswpz9nnk1j30mg0fg0te.jpg" alt="image-20210728155851478" style="zoom:80%;" />

通过一系列称为”`GC Roots`“的根对象作为起始节点集，从这些节点开始根据引用关系向下搜索，搜索过程所走过的路径称为引用链（`Reference Chain`），如果某个对象到`GC Roots`间没有任何引用链相连（即从`GC Roots`到这个对象不可达），则证明此对象已死亡→垃圾

> `GC Roots`：由于`Root`采用栈的方式存放变量和指针，所以如果一个指针保存了堆内存里面的对象，但是自己不存放在堆内存里面，那么它就是一个`Root`
>
> - 在虚拟机栈（栈中的局部变量表）中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等
> - 在方法区中类静态属性引用的对象，譬如`Java`类的引用类型静态变量
> - 在方法区中常量引用的对象，譬如字符串常量池里的引用（虽然`JDK7`后移到了堆里面，但是理论上仍然属于方法区）
> - 在本地方法栈中JNI（即`Native`方法）引用的对象
> - `JVM`内部的引用，如基本数据类型对应的Class对象、一些常驻的异常对象、系统类加载器
> - 所有被同步锁（`synchronized`关键字）持有的对象
> - 反映`JVM`内部情况的`JMXBean`、`JVMTI`中注册的回调、本地代码缓存等
>
> 除了这些固定的`GC Roots`集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象”临时性“加入，共同构成`GC Roots`集合。譬如分代收集和局部回收
>
> 如果只针对`Java`堆中的某一块区域进行垃圾回收，必须考虑到内存区域是`JVM`自己的实现细节，而不是孤立封闭的，这个区域的对象完全有可能被其他区域的对象所引用，这时候就需要一并将关联的区域对象也加入`GC Roots`集合，才能保证可达性分析的准确性

==**如果要使用可达性分析算法来判断内存是否可回收，那么分析工作就必须在一个能保障一致性的快照中进行。这也是GC进行时必须`STW`的一个重要原因。（否则无法保证分析结果的准确性）**==



##### 对象的finalization机制

- `Java`对象提供了对象终止机制来允许开发人员**提供对象被销毁之前的自定义处理逻辑**

- 当垃圾收集器要回收一个对象之前，会先调用这个对象的`finalize()`方法

- `finalize()`方法允许在子类中被重写，用于在对象被回收时进行**资源释放**

- `finalize()`方法的调用应该交给GC调用，即永远不要主动调用`finalize()`方法

  - `finalize()`方法可能会导致对象复活
  - `finalize()`方法的执行时间没有保障，它完全由`GC`线程决定。在极端情况下，若不发生`GC`，则`finalize()`方法将没有执行机会
  - 一个糟糕的`finalize()`方法会严重影响`GC`的性能

- 由于`finalize()`方法的存在，虚拟机中的对象一般处于三种可能的状态：

  1. **可触及的**：即从`GC Roots`可达，非垃圾，不需要回收

  2. **可复活的**：对象的所有引用都被释放，但是对象有可能在`finalize()`方法中复活

     > 复活：只要重新与引用链上的任何一个对象建立关联即可，如把自己（`this`关键字）赋值给某个类变量或者对象的成员变量

  3. **不可触及的**：对象的`finalize()`方法被调用，并且没有复活。不可触及的对象不可能被复活，因为`finalize()`方法只会被调用一次

  > `finalize`方法是对象逃脱死亡的最后机会，且只有不可触及状态的对象才被视为死亡，会被回收



##### 引用分类

![引用.png](https://i.loli.net/2019/04/11/5caf62e756c0d.png)

- 强引用是最普遍存在的，垃圾收集器永远不会收集被强引用的对象，宁愿出现`OOM`

- 软引用可以很好地解决`OOM`的问题，并且其特性很适合用来实现缓存

  ```java
  SoftReference<T> reference = new SoftReference<>(new T());
  ```

- 弱引用：当`JVM`进行垃圾回收时，无论内存是否充足，都会被回收

  ```java
  WeakReference<T> reference = new WeakReference<>(new T());
  ```

- 虚引用：其并不影响对象的生命周期，如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾收集器回收，但是如果引用的对象被回收了，那么会收到系统通知。

  虚引用必须和引用队列关联使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

  ```java
  // 引用队列
  ReferenceQueue<T> queue = new ReferenceQueue<>();
  PhantomReference<T> pr = new PhantomReference<>(new T(), queue);
  ```

  

##### 清除阶段——垃圾收集算法

###### 标记-清除算法(Mark-Sweep)

是最基础的收集算法，首先标记出所有需要收集的对象，标记完成后统一回收掉所有被标记的对象。

> 这里的清除不是真的置空，而是把想要清除的对象地址保存在空闲的地址列表里，下次有新对象需要加载时，判断垃圾的位置空间是否足够，如果够，就存放

缺点：

- **执行效率不稳定**，如果有大量对象需要被回收，那么此时就需要进行大量的标记、清除动作，即标记和清除这两个动作都受回收对象的数量影响
- **内存空间的碎片化问题**，标记、清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致当以后在程序运行过程中需要分配较大对象时，因为无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作
- 需要额外的内存空间，因为内存不规整，所以需要维护一个空闲列表

![mark-sweep.png](https://i.loli.net/2019/04/12/5caff6994cc9e.png)



###### 标记-复制算法(Mark-Copying)

把可用的内存空间分为两块，每次只使用其中一块，垃圾回收时把正在使用的这一块内存中仍然存活的对象复制到另一块内存中，然后清除正在使用的这块内存中的所有对象，并交换两个内存的角色（即当前不使用的那一块内存会成为下一轮中使用中的内存），最后完成垃圾回收。

> 对应的就是`Survivor`区，S0和S1区不断交换角色，他们之中的存活对象也在这两个区之间移动

优点：

- 没有标记和清除过程，实现简单，运行高效
- 复制到另一个区之后保证了空间的连续性，不会出现”碎片“问题

缺点：

- **空间浪费**，因为每次只能使用其中的一个区，另一个区的内存是不可用的

  > 有研究表明新生代中的对象有98%熬不过第一轮收集，因此并不需要按照1:1的比例来划分新生代的内存空间
  >
  > `Appel`式回收：把新生代划分为一块较大的`Eden`区和两块较小的`Survivor`区（`HotSpot`默认的比例为8:1:1，即只有10%的内存空间会被浪费），每次分配内存只使用`Eden`区和其中的一个`Survivor`区
  >
  > 逃生门：因为无法保证每次回收都只有不多于10%的对象存活（即一个`Survivor`区可以容纳下所有存活的对象），当`Survivor`空间不足时就需要依赖其他内存区域（大部分情况下是老年代）进行分配担保（`Handle Promotion`）

- 对于`G1`这种分拆成为大量`region`的`GC`，复制而不是移动，意味着`GC`需要维护`region`之间对象引用关系，不管是内存占用或者时间开销都不小

- 如果内存中存活对象很多，那么需要就需要复制大量的对象，效率就会大幅下降。（适用于垃圾较多的情况）

  > 所以在老年代中不适用该算法，因为老年代中大部分对象都是长期存活的，需要清除的对象较少

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsyzzybhxej30sl0gxgmm.jpg" alt="image-20210730151638003" style="zoom: 67%;" />



###### 标记-压缩/整理算法(Mark-Compact)

复制算法的高效性是建立在存活对象少、垃圾对象多的前提下的，并且需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，因此在老年代中不适用这种算法。标记-整理算法其中的标记过程与标记-清除算法一样，但后续步骤不是直接对可回收对象进行清理，而是**让所有存活的对象都向内存空间一端移动，然后直接清理掉边界以外的内存**。

> - 移动存活的对象并更新所有引用这些对象的地方将会是一种极为负重的操作，而且这种==对象移动操作必须全程暂停用户应用程序才能进行（`STW,Stop The World`）==。
>
> - ==弥散堆中的存活对象导致的空间碎片化问题只能依赖更为复杂的内存分配器和内存访问器来解决，增加了额外的负担，应用程序吞吐量会降低==
>
> - 从垃圾收集的停顿时间来看，不移动对象停顿时间会更短，甚至可以不需要停顿
> - 从整个程序的吞吐量来看，移动对象负担更低，吞吐量更大
> - 移动则内存回收时会更复杂，不移动则内存分配时会更复杂

优点：

- 消除了标记-清除算法当中，内存空间碎片化的问题。当给新对象分配内存时，`JVM`只需要持有一个内存的起始地址即可
- 消除了复制算法当中内存减半的高昂代价

缺点：

- 效率上低于复制算法
- 移动对象的同时，如果对象被其他对象引用，则还需要调整引用的地址
- 移动过程中，需要`STW`

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gsz1axn969j30kv0gyaav.jpg" alt="image-20210730160149318" style="zoom:67%;" />

|          |          标记-清除           |               标记-复制               |  标记-整理   |
| :------: | :--------------------------: | :-----------------------------------: | :----------: |
|   速度   |              中              |                 最快                  |     最慢     |
| 空间开销 | 少（但会导致空间碎片化问题） | 多，通常需要活对象的2倍大小（无碎片） | 少（无碎片） |
| 移动对象 |              ×               |                   √                   |      √       |

- 标记阶段的开销与存活对象的数量呈正比
- 清除阶段的开销与所管理区域的大小成正相关（线性遍历）
- 整理阶段的开销与存活对象的数据成正比（存活对象越多，需要整理的对象就越多）



##### 分代收集理论

分代收集名为理论，实质是一套符合大多数程序运行实际情况的经验法则，它建立在两个分代假说之上：

1. 弱分代假说（`Weak Generation Hypothesis`）：绝大多数对象都是朝生夕灭的
2. 强分代假说（`Strong Generation Hypothesis`）：熬过越多次垃圾收集过程的对象越难以消亡

因此，**垃圾收集器应该将`Java`堆划分出不同的区域，然后将回收对象依据其年龄（即熬过垃圾手机的次数）分配到不同的区域之中存储**。这样在年轻代中（存储的几乎都是朝生夕死的对象）每次回收时就可以只关注如何保留少量存活而不是标记大量将要回收的对象，就能以较低代价回收到大量的空间，对于老年代虚拟机可以使用较低的频率去回收这个区域，这就**同时兼顾了垃圾收集的时间开销和内存的空间有效利用**

> 频繁收集新生代，较少收集老年代，几乎不动方法区
>
> 新生代：区域相对老年代较小，对象生命周期短、存活率低、回收频繁（因此==适用标记-复制算法==，而该算法内存利用率不高的问题，可以通过HotSpot中两个Survivor区的设计得到缓解）
>
> 老年代：区域较大，对象生命周期长、存活率高、回收频率低（因此==一般采用标记-清除算法或标记-整理算法与其的混合实现==）

`JVM`在进行`GC`时，并非每次都对三个内存区域（新生代、老年代、方法区）一起回收，大部分时候回收的都是新生代。

针对`HotSpot VM`的实现，它里面的`GC`按照回收区域又分类两大类：

- ==部分收集（`Partial GC`）==：不是完整收集整个`Java`堆的垃圾收集

  - **新生代收集（`Minor GC/Young GC`）**：只是新生代的垃圾收集

    > - 当新生代空间不足时，就会触发`Minor GC`，这里的新生代满指的是`Eden`区满，`Survivor`区满不会触发`GC`
    > - 因为`Java`对象大多都具备朝生夕灭的特性，所以`Minor GC`非常频繁，一般回收速度也比较快
    > - `MinorGC`会引发`STW（Stop The World）`，暂停其他用户的线程，等`GC`结束后用户线程才恢复运行

  - **老年代收集（`Major GC/Old GC`）**：只是老年代的垃圾收集

    > - 出现了`Major GC`，经常会伴随至少一次的`Minor GC`（也就是在老年代空间不足时，会先尝试触发`Minor GC`。如果空间仍然不足，才会触发`Major GC`）
    > - `Major GC`的速度一般会比`Minor GC`慢10倍以上，`STW`的时间更长
    > - 如果`Major GC`后内存还不足，就会报`OOM`

  - **混合收集（`Mixed GC`）**：收集整个新生代以及部分老年代的垃圾收集

- ==整堆收集（`Full GC`）==：收集整个`Java`堆和方法区的垃圾收集

  > 触发`Full GC`执行的情况有以下五种：
  >
  > 1. 调用`System.gc()`时，系统建议执行`Full GC`，但是不一定执行
  > 2. 老年代空间不足
  > 3. 方法区空间不足
  > 4. 通过`Minor GC`后进入老年代的平均大小大于老年代的可用内存
  > 5. 由`Eden`区、`From`区向`To`区复制时，对象大小大于`To`区可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小



##### 增量收集算法

迄今为止，所有收集器**在根节点枚举（即分析`GC Roots`可达性）这一步时都是必须暂停用户线程的（`STW`）**，如果垃圾回收时间过长，应用程序会被挂起很久，将严重影响用户体验和系统稳定性，因此诞生了增量收集算法（`Incremental Collecting`）

基本思想：如果一次性将所有的垃圾进行处理，需要造成系统长时间的停顿，那么就可以让垃圾收集线程和应用程序线程交替执行，每次只收集一小片区域的内存空间。总的来说，增量收集算法的基础仍是传统的标记-清除和复制算法，其**通过对线程间冲突的妥善处理，允许垃圾收集线程以分阶段的方式完成标记、清理或复制工作**

缺点：虽然能减少系统的停顿时间，但因为**线程切换和上下文转换的消耗，会使得垃圾回收的总体成本上升，造成系统吞吐量下降**



##### 分区算法

一次回收的区域越大，所需时间就越长。为了更好地控制`GC`产生的停顿时间，将一块大的内存区域分隔成多个小块，根据目标的停顿时间每次合理的回收若干个小分区，从而减少一次`GC`产生的停顿。

每一个小区间都独立使用、独立回收，这种算法的好处是可以控制一次回收多个小区间

> 分代算法按照对象的生命周期长短划分成两个部分（年轻代与老年代），分区算法将整个堆空间划分成连续的不同小区间`region`



#### 垃圾回收器

##### System.gc()的理解

- 在默认情况下，通过`System.gc()`或者`Runtime.getRuntime().gc()`的调用，会显式触发`Full GC`，同时对老年代和新生代进行回收，尝试释放被丢弃对象占用的内存

- `System.gc()`调用附带一个免责声明，无法保证对垃圾收集器的调用（即无法保证调用的时间）

  > 如果只是单独调用`System.gc()`方法，只会提醒`JVM`的垃圾收集器执行`GC`，但不确定是否马上执行
  >
  > 如果再调用`System.runFinalization()`方法，就会强制调用使用引用的对象的`finalize()`方法



##### 内存溢出（OOM）与内存泄漏（Memory Leak）

**内存溢出**：==空闲内存不足，并且垃圾收集器无法提供更多内存==

> 在抛出`OOM`之前，通常垃圾收集器会被触发，尽其所能的去清理出空间（但如果像分配一个超大对象，直接超出堆内存，那么即使通过垃圾收集也无法解决问题，会直接抛出`OOM`）

1. JVM的堆内存设置不足（`-Xms`、`-Xmx`）
2. 代码中创建了大量大对象，并且长时间不能被垃圾收集器收集（存在被引用）

**内存泄漏**：==对象不会再被程序用到了，但是`GC`又不能回收他们==

> 实际情况某些不太好的实践会导致对象的生命周期变得很长甚至导致`OOM`，可以叫做宽泛意义上的内存泄漏
>
> 内存泄漏不会立刻引起程序崩溃，但是一旦发生内存泄露，程序中的可用内存就会被逐步蚕食，直到耗尽所有内存，最终抛出`OOM`，导致程序崩溃





#### String

##### String的基本特性

- `String`声明为`final`的，不可继承的

- `String`实现了`Serializable`接口，表示字符串是支持序列化的；`Comparable`接口，表示`String`可以比较大小

- `String`在`JDK8`及以前定义了`final char[] value`字符数组用于存储字符串数据，`JDK9`起改用`final byte[] value`字节数组

  > `char`占用两字节，而字符串大部分都是拉丁字符（只需要一字节就存的下），因此如果使用字符数组，在多数情况下会会有一半的空间被浪费
  >
  > 因此`JDK9`改用字节数组来存储字符串数据，并加上编码标识，用于解决`UTF-16`（占用两字节）数据的存储问题
  >
  > 与`String`相关的类`StringBuffer`、`StringBuilder`也会跟着修改

- `String`代表不可变的字符序列

  > 总结来说凡是对字符串进行修改的操作都是重新指定内存区域赋值

  - 当对字符串重新赋值时，需要**重新指定内存区域赋值**，不能使用原有的`value`进行赋值

    ```java
    String s1 = "abc";
    String s2 = "abc";
    // true--字面量定义方式，abc存储在字符串常量池中，s1和s2指向的是字符串常量池中的同一个（也是唯一一个）abc
    System.out.println(s1 == s2);
    s1 = "hello";
    // false--重新赋值，不会影响原有的abc，而是在另一块内存空间存放hello，并让s1指向它
    System.out.println(s1 == s2);
    // hello--s1指针因为重新赋值已经转为指向hello
    System.out.println(s1);
    // abc
    System.out.println(s2);
    ```

  - 当对现有的字符串进行连接操作时，也需要**重新指定内存区域赋值**，不能使用原有的`value`进行赋值

    ```java
    // 由于采用数组存储字符串数据，因此字符串一旦定义好之后就不能修改长度了，因此拼接操作实际上是重新赋值
    s2 += "def";
    String s3 = "abc";
    // abcdef--重新指定内存区域赋值，不会对原有的abc产生影响，字符串常量池里会有abc和abcdef
    System.out.println(s2);
    // false--s2因为连接操作重新指定了内存区域
    System.out.println(s2 == s3);
    ```

  - 当调用`String.replace()`方法修改指定字符或字符串时，也需要**重新指定内存区域赋值**，不能使用原有的`value`进行赋值

    ```java
    String s3 = "abc";
    String s4 = s3.replace("a","m");
    // abc
    System.out.println(s3);
    // mbc
    System.out.println(s4);
    //false--s4做了修改，重新重新指定内存区域赋值
    System.out.println(s3 == s4);
    ```

- 通过字面量的方式（即`String str = “abc”`）给一个字符串赋值，此时的字符串值声明在字符串常量池中



##### StringTable

- `String`的`String Pool`是一个固定大小的`HashTable`

- 在`JDK6`中`StringTable`是固定的，长度为1009，所以如果常量池中的字符串很多，就会造成`Hash`冲突严重，从而导致链表会很长，而链表长了后会直接造成的影响就是调用`String.intern()`时性能会大幅下降

- 在`JDK7`中，`StringTable`的默认长度为60013

  > 可以通过`-XX:StringTableSize`设置

- `JDK8`起，1009是可设置的最小值



##### String的内存分配

- 字符串常量池中不存放重复字符串

- `JDK7`起`StringTable`从永久代移到了堆中

  > 原因：
  >
  > 1. 永久代一般比较小（过多字符串容易导致`OOM`）
  > 2. 永久代回收频率低



##### 字符串拼接操作

1. ==常量与常量的拼接结果在常量池==，原理是编译期优化（直接就把其视为一个已经拼接好的字符串）

   ```java
   String s1 = "a" + "b" + "c";
   String s2 = "abc";
   // true
   System.out.println(s1 == s2);
   // true
   System.out.println(s1.equals(s2));
   ```

2. 常量池中不会存在相同内容的常量

3. ==如果其中有一个是变量，结果就在堆中（非常量池区域，相当于`new String()`）==，变量拼接的原理是`StringBuilder`（在`JDK5`之前使用的是`StringBuffer`）

   > 注意变量的定义：如果是`String str`则是变量，拼接时使用`StringBuilder`；但如果是`final str`则是引用常量，原理是编译期优化（用`final`修饰的字符串就是在编译期可知的）
   >
   > ```java
   > // s1 + s2在字节码中表现为
   > StringBuilder sb = new StringBuilder();	// 隐藏的定义
   > sb.append(s1);
   > sb.append(s2);
   > // 约等于new String(拼接后的字符串)-->底层调用了new String()
   > sb.toString();
   > ```

   ```java
   String s1 = "javaEE";
   String s2 = "hadoop";
   String s3 = "javaEEhadoop";
   String s4 = "javaEE" + "hadoop";
   String s5 = s1 + "hadoop";
   String s6 = "javaEE" + s2;
   String s7 = s1 + s2;
   
   // true--常量与常量的拼接
   s3 == s4
   // true
   s3 == s5
   
   // false--变量与常量的拼接
   s3 == s6
   // false
   s3 == s7
   // false
   s5 == s6
   // false--变量与变量的拼接
   s6 == s7
   ```

   ![image-20210726201329360](https://tva1.sinaimg.cn/large/008i3skNgy1gsum3jxc8bj30c902ymxc.jpg)

   > 即使用+拼接字符串时，实际上仍然是使用`StringBuilder.append()`方法
   >
   > 那为什么会说`StringBuilder`拼接效率要比+更高呢？
   >
   > 准确的说，是在循环的情况下`StringBuilder`的效率在更高，在非循环情况下+拼接效率更高。
   >
   > 这是因为==在循环的情况下，使用+拼接，每次循环都会创建新的`StringBuilder`、`String`对象==（并且最后拼接完成调用`StringBuilder.toString()`时又会`new String()`），这样，绝大部分字符串都是临时对象，不但浪费内存，还会影响`GC`效率
   >
   > 如果显式的使用`StringBuilder`进行拼接，它是一个可变对象，可以预分配缓冲区，这样，往`StringBuilder`中新增字符时，不会创建新的临时对象（即多次`append`自始至终都使用的是同一个`StringBuilder`）
   >
   > 在非循环情况下，使用+进行字符串常量的拼接在编译期就已经完成，而使用`StringBuilder`则在运行时才完成，因此在此情况下使用+拼接效率更高
   >
   > *`StringBuilder`使用：其底层也是采用`byte`数组存储字符串数据（`JDK8`及以前是`char`数组），因此如果拼接字符串超过数组可存储的范围后会进行扩容操作，因此在循环拼接时在创建`StringBuilder`对象时指定数组大小*

4. 如果拼接的结果调用`intern()`方法，先判断字符串常量池中是否存在值，如果存在，则返回常量池中该值的地址；如果不存在，则主动将常量池中还没有的字符串对象放入池中，并返回此对象地址

   ```java
   String s8 = s6.intern();
   // true--常量池中已有javaEEhadoop，因此直接返回常量池中该字符串的地址，也是s3指向的地址
   s3 == s8
   ```



##### intern()

当`intern()`方法被调用时，会通过`equals()`方法判断字符串常量池中是否已存在该字符串。如果已存在，则返回字符串常量池中对应的常量；如果不存在，则把该字符串放入字符串常量池中并返回其引用 

- 如果不是用字面量方式声明的`String`对象，可以使用`String`提供的`intern()`方法

- 如果在任意字符串上调用`String.intern()`，那么其返回结果所指向的那个类实例，必须和直接以常量形式出现的字符串实例完全相同

  ```java
  new String("a" + "b" + "c").intern() == "abc";
  ```

- 通俗的说，`intern()`方法就是确保字符串再内存里只有一份拷贝（在字符串常量池中是唯一的），这样可以节约内存空间，加快字符串操作任务的速度

> 即保证变量指向的是字符串常量池中的数据有两种方式：
>
> 1. 字面量定义：`String s = “abc”;`
> 2. 调用`intern()`方法：`String s = new String(“abc”).intern();`



##### Question

1. new String(“ab”)创建了几个对象？

   ==1或2个==。首先检索常量池中是否存在“abc”，如果不存在，则先在常量池中创建该字符串，然后执行new操作（在堆内存中创建Stirng对象并返回引用）→ 创建2个对象；如果存在，则只会在堆中创建String对象→创建1个对象

   ![image-20210727103437526](https://tva1.sinaimg.cn/large/008i3skNgy1gsvazk0tohj30cd02rgls.jpg)

2. String s = “abc”创建了几个对象？

   ==0或1个==。同样先判断常量池中是否存在，如果不存在，则在常量池中创建并返回引用→创建1个对象；如果存在，则直接返回指向常量池该字符串的引用→创建0个对象

> 在上述过程中检查常量池是否有相同Unicode的字符串常量时，使用的方法便是String中的intern()方法

3. String str = “abc” + “def”创建了几个对象？

   ==0或1个==。当多个字符串常量拼接时，在编译期会被优化成一个拼接完成的字符串，即其实际上与`String str = “abcdef”`是等效的，不会创建临时对象`abc`和`def`，减轻了垃圾收集器的压力。

   ![image-20210727103205022](https://tva1.sinaimg.cn/large/008i3skNgy1gsvawwt9pjj305g01k744.jpg)

4. String str = new String(“abc”) + new String(“def”)创建了几个对象？

   ==4~6个==。一个是创建的`StringBuilder`对象，1个是拼接完成后调用`StringBuilder.toString()`里面会`new String(“abcdef”)`。剩下的详看1。

   ![image-20210727104645683](https://tva1.sinaimg.cn/large/008i3skNgy1gsvbc6hqdxj30d908p758.jpg)

   > 注意：无论字符串常量池中是否有字符串`“abcdef”`，`StringBuilder.toString()`的调用在常量池中都不会创建该字符串对象（*因为是用`StringBuilder`内**存储字符串数据的数组**作为参数进行`new String()`的，而不是以字面量作为参数）*
   >
   > ![image-20210727104549165](https://tva1.sinaimg.cn/large/008i3skNgy1gsvbb7ijp8j30db06z0t9.jpg)

5. String str = “abc” + new String(“def”)创建了几个对象？

   ==3~5个==。+号拼接隐藏创建的`StringBuilder`对象，拼接完成后调用`StringBuilder.toString()`时的`new String()`。

6. ![image-20210727105859777](https://tva1.sinaimg.cn/large/008i3skNgy1gsvbox10b6j30pt09ogmq.jpg)

   Line1：在字符串常量池中创建常量1，在堆中创建字符串实例。s指向的是堆空间中的引用地址

   Line2：调用此方法前，字符串常量池中已经存在了1。调用的`intern`方法虽然会返回指向常量池中常量的引用，但是没有赋值给s，没有改变s的指向，所以s仍然指向堆空间（如果是`s = s.intern()`，结果就会是`true`）

   Line3：s2指向的是常量池中常量1的地址。

   Line4：一个指向堆空间中的字符串实例，一个指向常量池中的常量，因此结果为`false`

   ![image-20210727154547165](https://tva1.sinaimg.cn/large/008i3skNgy1gsvjzd3x08j30ds07iwew.jpg)

​		Line1：s3指向的是堆空间中创建的`String`对象，执行完该行代码后，字符串常量池中不存在”11“

​		Line2：在字符串常量池中创建一个引用指向堆空间中的字符串对象，常量池中仍然不存在”11“，但存在对堆空间中”11“的引用（因为改变的是常量池中的值而不是指针指向，因此无论是否有接受intern()返回值结果都会是`true`）

​		Line3：s4指向的是上一行代码执行后，在常量池中生成的指向”11“的地址

​		Line4：`JDK6`中，字符串常量池在永久代中，调用`intern`方法会在堆中和永久代的字符串常量池中各创建一个”11“对象

​					 `JDK7`后，字符串常量池移到了堆中，考虑到空间，因为堆中已经存放了拼接后的字符串对象，因此常量池中不再重复创建，而是创建一个指向堆空间中的该对象的地址。因此s4指向的地址与s3一致（都是堆空间中的字符串对象的地址），结果为`true`

> 如果把s3和s4的定义顺序反过来，结果就会变成`false`。这是因为s4定义时先把”11“放入了字符串常量池，s4调用`intern()`方法后发现常量池中已经有11，因此返回了对常量池中11的引用，但是因为s3没有接受`intern`方法的返回值，没有重新赋值，仍然指向堆中的字符串对象，因此结果为`false`（如果改为`s3 = s3.intern()`结果就是`true`了）

![image-20210727162552205](https://tva1.sinaimg.cn/large/008i3skNgy1gsvl5pkhlxj30cb06idg6.jpg)

 

