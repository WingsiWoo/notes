# 面试-JVM

## JVM整体结构

1. ==类加载子系统==
2. ==运行时数据区（内存结构）==
3. ==本地方法接口==
4. ==执行引擎==



## 什么是JVM内存结构？

1. ==程序计数器==

   线程私有的一块很小的内存空间，作为当前线程的行号指示器，记录当前正在执行的线程的指令地址

2. ==虚拟机栈==

   线程私有，每个方法执行时都会创建一个栈帧，用于存储局部变量表、操作数、动态链接和方法返回值等信息，如果线程请求的栈深度超过了虚拟机允许的最大深度时，就会抛`StackOverFlowError`

3. ==本地方法栈==

   线程私有，用于保存`native`方法的信息，JVM不会在虚拟机栈中为本地方法创建栈帧，而是简单的动态链接并直接调用该方法

   > HotSpot中不区分虚拟机栈和本地方法栈（没有本地方法栈）

4. ==Java堆==

   所有线程共享，几乎所有的对象实例和数组都要在堆上分配内存，垃圾回收的主要对象

5. ==方法区==

   存放已被加载的类信息、常量、静态变量、即时编译器编译后的代码数据，也就是永久代。JDK8中不再存在方法区，取而代之的是元数据区，原来的方法区被分为两部分：

   - 加载的类信息（在元数据区中）
   - 运行时常量池（在堆中）



## 什么是JVM内存模型（JMM）？

由于主存与CPU的速度差异十分大，所以在传统的计算机内存架构中会引入高速缓存来作为主存和CPU之间的缓冲，一般都有多级缓存，分为L1、L2、L3缓存。因为这些缓存的存在，**提升了数据的访问性能，也减轻了数据总线上数据传输的压力**。

但是同时也带来了以下问题：

1. ==缓存一致性问题==

   在多CPU的系统中，每个CPU内核都拥有自己的高速缓存，它们共享同一主存。当多个CPU的操作都涉及同一块主存区域的时候，就可能会导致不同CPU的高速缓存内的数据不一致问题

2. ==处理器优化和指令重排序==

   为了进一步提升CPU的执行效率，使得处理器内部的运算单元能够最大化被充分利用，处理器会对输入的指令进行优化，指令重排是处理器优化的其中一种。重排序可以分为三种类型：

   - **编译器优化的重排序（指令重排）**

     编译器在不改变单线程环境下程序运算结果的前提下，可以重新安排语句的执行顺序

   - **指令集并行的重排序**

     采用指令级并行技术（如流水线技术）来将多条指令重叠执行，如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序

   - **内存系统的重排序**

     由于处理器使用缓存和读写缓冲区，这使得加载和存储操作看上去可能是在乱序执行

   > Java并发的三个问题：可见性问题、原子性问题、有序性问题，实际上就是缓存一致性（➡️可见性问题）、处理器优化（➡️原子性问题）、指令重排（➡️有序性问题）造成的



为了解决上面的问题，规范内存的读写操作，JMM定义了一组排序规则来**保证线程的可见性**。这一组规则就是**`Happens-Before`**，JMM 规定，要想保证 B 操作能够看到 A 操作的结果(无论它们是否在同一个线程)，那么 A 和 B 之间必须满足 **`Happens-Before`** ：

1. ==单线程规则==

   一个线程中的每个动作都`happens-before`于该线程的后续动作

2. ==监视器锁定规则==

   监视器的解锁动作一定`happens-before`于该监视器的后续锁定操作

3. ==volatile变量规则==

   对volatile字段的写入操作`happens-before`于对其的读取操作

4. ==线程start规则==

   线程start()方法的执行`happens-before`该启动线程内的任何执行动作

5. ==线程join规则==

   一个线程内的所有动作都`happens-before`于该线程join()方法返回

6. ==传递性==

   如果A `happens-before` B，B `happens-before` C，那么A `happens-before` C

> 即JMM描述的是多线程对共享内存修改后彼此之间的可见性，另外还确保了正确的Java代码可以在不同体系结构的处理器上正确运行



## 内存屏障

|  屏障种类  |          伪代码          |                          说明                          |
| :--------: | :----------------------: | :----------------------------------------------------: |
|  LoadLoad  |  Load1; Barrier; Load2;  |                Load1在Load2之前读取完成                |
| StoreStore | Store1; Barrier; Store2; | Store1在Store2之前写入完成，并刷入主存（所有线程可见） |
| LoadStore  | Load1; Barrier; Store2;  |               Load1在Store2之前读取完成                |
| StoreLoad  |  Store1 Barrier; Load2;  |         Store1在Load2之前写入完成（刷入内存）          |

`StoreLoad`是一个“全能型”的屏障，它同时具有其他3个屏障的效果。现代的多数处理器大多支持该屏障（其他类型的屏障不一定被所有处理器支持）。执行该屏障开销会很昂贵，因为当前处理器通常要把写缓冲区中的数据全部刷新到内存中。



## 结合内存模型讲一下volatile实现原理

### 内存模型

- 操作系统内存模型

  指令都是由CPU执行的，而一般的指令都是需要读写数据的，这样子就涉及到访问主存。但是访问主存的速度和CPU执行速度差得很大，I/O操作严重拖低了CPU的执行效率。因此为了解决这个问题，在CPU和主存之间增加了高速缓存，以此来减小主存和CPU之间的速度差异。高速缓存中存储着对主存数据的copy，CPU对高速缓存中的数据进行操作，执行结束后把高速缓存中更新完成的数据刷回主存

- Java内存模型

  主要思想跟操作系统的内存模型差不多，其规定所有变量的值都在主存中，每个线程都有自己的工作内存（相当于CPU高速缓存），线程对变量的操作都必须在工作内存中完成，是不能直接对主存进行操作的，并且每个线程不能访问其他线程的工作内存。



### 多线程的主要特性

#### 原子性

如i++的操作，实际上不止一步：

1. `read`：从主存中读取数据i，以便后续`load`操作
2. `load`：把主存中读取到的数据写入当前线程的工作内存
3. `use`：从工作内存中读取数据并进行+1操作
4. `assign`：把更新之后的数据写入工作内存
5. `store`：把工作内存中的数据刷入主存，以便后续`write`操作
6. `write`：把工作内存中更新后的变量值赋值给主存中的变量

#### 可见性

可见性即每个线程的工作内存中的数据对其他线程不可见。`volatile`只能解决这个问题。

#### 有序性

在满足`happens-before`原则的前提下，指令可能会进行重排序



### volatile实现原理

#### 禁止指令重排序

`volatile`通过内存屏障禁止指令重排序，主要遵循以下三个规则：

1. 当第二个操作是`volatile`写时，不管第一个操作是什么，都不能重排序。这个规则确保==`volatile`写之前的操作不会被编译器重排序到`volatile`写之后==。➡️ `storestore`、`loadstore`屏障
2. 当第一个操作是`volatile`读时，不管第二个操作是什么，都不能重排序。这个规则确保==`volatile`读之后的操作不会被编译器重排序到`volatile`读之前==。➡️ `loadstore`、`loadload`屏障
3. 当第一个操作是`volatile`写，第二个操作是`volatile`读时，不能重排序。

`volatile`内存屏障插入策略：

- 写操作**前**插入`storestore`屏障

  别人写完自己才能写

- 写操作**后**插入`storeload`屏障

  自己写完别人才能读

- 读操作**后**插入`loadload`屏障

  自己读完别人才能读

- 读操作**后**插入`loadstore`屏障

  自己读完别人才能写



#### 保证数据可见性

`volatile`的作用是保证数据的可见性，使用`volatile`修饰的变量在修改后会立刻刷入主存，保证其他线程能第一时间发现缓存行失效，并到主存中重新读取数据。

假设下面一段代码：

```java
while(run) {
	// 死循环，直到run为false
}
```

假设线程1执行上述死循环代码，线程2通过把`run`设置为`false`来控制死循环的结束。

- 如果`run`不用`volatile`修饰，那么线程2修改`run`后如果没有及时把数据刷回主存，那么线程1就不会及时退出循环；
- 但是如果使用`volatile`修饰，那么就会强制立即把数据刷回主存，线程1在下次循环再次读取自己工作内存中的`run`时，就能及时发现自己对`run`的缓存失效了，到主存中重新读取数据，从而及时退出循环。



#### 适用场景

1. 像上面通过`run`变量控制循环这种对数据要求不太高的，即这一次没读到最新数据没关系，只要下一次读能及时发现缓存行失效去主存读取最新数据就行了的，适用`volatile`

2. 而对于像i++这种复杂的操作，由于`volatile`并不保证原子性，可能会导致最终i的结果并不正确。因此需要通过原子类或者加锁来实现并发安全

   > 假设最开始i=0，线程1执行到use操作（已经从工作内存中读取了数据），突然进来了一个线程2一气呵成的完成了i++的操作，并把最新数据i=1刷入主存。由于线程1已经从工作内存中读取过数据了，不会再去读取，自然也不会发现自己的缓存行失效了，仍然会对旧数据i=0进行自增操作。这样就导致i自增了两次，可最终结果却是只+1
   >
   > 原子类通过CAS保证原子性，volatile保证可见性



## 方法区的实现

在JDK7及以前是永久代，在JDK8起是元空间

### 永久代

永久代与堆中的老年代在物理地址上是连续的，二者之中任何一个满了都会触发Full GC

JDK7时部分数据不再存储在永久代：

- 字符串常量池`StringTable`移动到Java堆

  程序运行过程中会创建大量字符串，其中很多可能只使用几次，所以`StringTable`实际上是经常需要回收的，而永久代只有在`Full GC`时才会被回收，导致无用的字符串常量长期霸占永久代内存，导致永久代内存空间不足

- 符号引用移动到本地方法栈

- 静态变量移动到Java堆

> 由于方法区主要存放类的相关信息，所以对于动态生成类的情况很容易导致永久代内存不足，引发java.lang.OutOfMemoryError:PermGen Space

### 元空间

元空间不在虚拟机中，而是在**本地内存**中，一般情况下元空间大小仅受本地内存大小影响（这样就大大降低了Full GC的触发概率）

> 抛出异常为java.lang.OutOfMemoryError

转换原因：

- 方法区用于存储类的相关信息，难以确定具体的大小，因此如何合理的设置永久代大小也比较困难。因为老年代与永久代是相连的，永久代设置过小容易永久代溢出，过大容易老年代溢出（都会触发`Full GC`）
- 字符串常量池存放在永久代中，回收效率低，容易永久代内存溢出
- 永久代给GC带来不必要的复杂性



## 垃圾回收机制

### GC算法

#### 根搜索算法（可达性算法）

从GC ROOT节点开始，寻找对应的引用节点，即从GC ROOT可达的节点仍然被引用，无需回收；从GC ROOT不可达的节点已经不再被引用，需要回收。

Java中可以作为GC ROOT的对象：

- 虚拟机栈中引用的对象（本地变量表）
- 方法区中静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中引用的对象（Native对象）

> 基本所有的GC算法都引用根搜索算法来判断对象是否需要回收



#### 标记-清除算法

1. 使用根搜索算法对可达节点（存活对象）进行标记
2. 标记完成后，再扫描整个空间中未被标记的对象进行直接回收

标记-清除算法直接回收不存活的对象，会造成内存碎片。适用于存活对象比较多的情况



#### 标记-复制算法

复制算法把内存划分为两个区间，使用此算法时，所有动态分配的对象都只能分配在其中一个区间（活动区间），另外一个区间（空闲区间）则是空闲的

1. 使用根搜索算法搜索存活对象，把存活对象复制到空闲区间
2. 扫描完毕后，把活动区间中的所有对象全部进行回收
3. 此时空闲区间和活动区间身份反转

标记-复制算法效率很高，不会产生内存碎片，但是浪费了50%的内存空间，适用于存活对象比较少的情况（否则一直把存活对象复制来复制去）

> 对新生代内存的回收（Minor GC）主要采用标记-复制算法



#### 标记-整理算法

有一个指针指向空闲区和非空闲区的分界线

1. 使用根搜索算法对存活对象进行标记
2. 把存活对象向一边移动，覆盖回收对象
3. 更新空闲边界指针

效率不高，但是没有像复制算法的50%内存浪费，也不会产生内存碎片

> 对老年代的回收（Major GC）主要采用标记-整理算法



#### 分代收集算法

堆被划分为两个空间，新生代和老年代，其中新生代又分为Eden区、From区、To区（默认占比为8:1:1）

1. 对象优先在Eden区分配内存（如果是超大对象，可能会直接在老年区分配内存，因此要避免超大对象的创建，因为如果这个超大对象只使用一次，理论上应该尽快回收，但由于其在的老年代很少GC，所以可能要等很久才会被回收），当Eden区内存不足时，会触发一次**`Minor GC`（采用复制算法）**

   - 把Eden区中无用的对象清理掉
   - 把Eden区中存活的对象移动到From区

2. **Eden区满**，再次触发`Minor GC`

   - 把Eden区和From区中无用的对象清理掉
   - 把Eden区和From区中存活的对象移动到To区
   - From区和To区角色对换（即当前的To区变为下一次GC的From区）

3. 对象每在From区和To区之间移动一次，年龄就会+1，到达一定年龄（默认为15）后就会被移动到老年代

4. 正常来说，老年代中的对象比较稳定，因此很少GC

5. 触发**`Major GC`的情况（采用标记-整理算法）**

   - `Minor GC`后进入老年代的对象所需内存大于老年代的可用内存
   - 老年代空间不足/永久代空间不足（JDK7前）
   - 超大对象所需内存大于老年代可用内存
   - 调用`System.gc()`，系统建议执行`Major GC`，但不一定执行

   > `Major GC（Full GC）`的回收对象为新生代、老年代、永久代



### GC收集器

每一个GC收集器都存在`Stop The World`的问题，即JVM为了要进行GC而暂停应用程序的执行，除GC所需的线程外，其他线程都进入阻塞状态，直到GC完成。GC收集器的优劣是通过其减少`Stop The World`的程度来评价的



#### Serial（-XX:+UseSerialGC）

Serial收集器是JVM中最基本、最早的收集器，在JDK1.3之前是JVM**新生代收集器**的唯一选择。顾名思义，Serial收集器是一个串行收集器，其关于GC的操作执行时（包括标记）都需要Stop The World，暂停所有的用户线程，直到回收结束。

- 使用复制算法
- 只有一个GC线程
- 对于限定单个CPU的环境，由于没有线程交互的开销，专心做垃圾收集，所以相对其他的收集器是最高效的



#### SerialOld（-XX:+UseSerialGC）

是Serial收集器的**老年代收集器**版本

- 使用标记-整理算法
- 在JDK1.5及之前与`ParallelScavenge`收集器搭配使用
- 作为CMS收集器的后备方案，如果CMS出现`Concurrent Mode Failure`，则SerialOld将作为后备收集器



#### ParNew（-XX:+UseParNewGC）

其实就是Serial收集器的多线程版本，有多个GC线程并发进行回收。除了Serial收集器外，只有它能与CMS收集器配合工作

- 使用复制算法
- 是许多运行在Server模式下的JVM首选**新生代收集器**
- 单CPU情况下，效率远低于Serial收集器



#### ParallelScavenge（-XX:+UseParallelGC）

吞吐量优先收集器，也是一个**新生代收集器**

- 使用复制算法
- 所谓吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值，即`吞吐量 = 运行用户代码时间 / （运行用户代码时间 + 垃圾收集时间）`



#### ParallelOld（-XX:+UseParallelOldGC）

JDK1.6之后才开始提供，在此之前，`ParallelScavenge`只能选择SerialOld来作为其**老年代的收集器**，这严重拖累了`ParallelScavenge`整体的速度。`ParallelOld`的出现，提高了`ParallelScavenge`的效率

- 是并行收集器，使用标记-整理算法
- 在注重吞吐量与CPU数量大于1的情况下，都可以优先考虑`ParallelScavenge + ParalleloOld`收集器。



#### 🌟CMS（-XX:+UseConcMarkSweepGC）

CMS是一个**老年代收集器**，全称 `Concurrent Low Pause Collector`，是JDK1.4后期开始引用的新GC收集器，在JDK1.5、1.6中得到了进一步的改进。

- 使用标记-清理算法
- 用两次短暂的`Stop The World`来代替串行或并行标记整理算法时候的长暂停
- 是一个对于响应时间的重要性需求大于吞吐量需求的收集器，适用于要求服务器响应速度高的情况

执行过程如下：

1. ==初始标记（STW initial mark）——第一次短暂暂停==

   需要`Stop The World`，扫描**与GC ROOT直接关联的对象**，并作标记

2. ==并发标记（Concurrent marking）==

   在初始标记的基础上继续向下追溯标记，该过程无需STW，用户线程和GC线程并发执行

3. ==并发预清理（Concurrent precleaning）==

   查找并发标记阶段进入老年代的对象，通过这一阶段，减少下一阶段——重新标记阶段的工作量，因为下一阶段需要STW，要尽可能缩短它执行的时间

4. ==重新标记（STW remark）——第二次短暂暂停==

   需要`Stop The World`，**从GC ROOT开始查找并标记并发标记阶段遗漏的对象**（在并发标记阶段结束后对象状态更新导致的遗漏），**并处理对象关联**。这个阶段耗时要比初始标记长，并且这个阶段可以并行标记

5. ==并发清理（Concurrent sweeping）==

   应用线程和GC清除线程可以并发执行

6. ==并发重置（Concurrent reset）==

   重置CMS收集器的数据结构，以便下一次垃圾回收，这一阶段也是并发的

缺点：

1. ==使用标记-清除算法进行清理，导致内存中会产生内存碎片。==

   CMS收集器做了一个小优化，把未分配的空间汇总成一个列表，当JVM需要分配内存空间时，从这个列表寻找可用空间即可，加快了分配内存的速度。但是由于内存碎片的存在，可能无法找到足够大的连续的内存空间，导致引发Full GC

2. ==需要更多的CPU资源==

   CMS收集器的大部分阶段都是GC线程和用户线程并发执行的，这样就需要占用更多的CPU资源，牺牲一部分吞吐量

3. ==需要更大的堆空间==

   由于CMS标记阶段应用程序仍然并发执行，就可能会存在需要分配内存空间的问题。为了保障CMS执行期间应用程序可以正常分配内存空间，就需要预留一部分内存空间，即还没有用完就开始进行GC。

4. ==无法处理浮动垃圾==

   在并发清理的过程中也可能会产生垃圾，但是由于本次GC并没有对其标记过，所以这些垃圾只能等到下一次GC的时候处理，这种垃圾就叫浮动垃圾

   > CMS默认在老年代空间使用`68%`时候启动垃圾回收。可以通过`-XX:CMSinitiatingOccupancyFraction=n`来设置这个阀值。
   >
   > 如果这个阈值设置的太大，就会导致CMS触发太晚，可能会引发内存无法分配成功的问题，抛出`Concurrent Mode Failure`，这是CMS独有的错误。发生这个错误的时候可以把阈值调小或通过`-XX:+UseCMSCompactAtFullCollection`参数开启空间碎片整理。（不整的话就会退化为SerialOld收集器）



#### 🌟GarbageFirst（G1）

- 负责**年轻代和老年代的GC**（`Young GC`和`Old GC`）

- 保留了分代概念

- 可控的停顿时间，可以通过 `-XX:MaxGCPauseMillis=200` 指定期望的停顿时间

  - 使用停顿预测模型来满足用户指定的停顿时间目标，并基于这个时间目标来选择进行垃圾回收的区块数量（只是尽量满足，不是绝对的）
  - G1采用增量回收的方式，每次回收一些region，而不是整堆回收

- G1收集器把堆划分为一个个大小相等的小块`region`，每一块的内存是连续的，G1在标记阶段结束之后就知道哪些`region`基本上是垃圾，存活对象极少，会优先回收这些区块，因为从这些区块能很快释放得到较大的连续内存空间。这也是被称为`GarbageFirst`的原因

  > 这种方式也在一定程度上优化了内存碎片化的问题
  >
  > 同时这些块也可以充当Eden、Survivor、Old 三种角色，不是固定的，使得内存可以更加灵活使用
  >
  > 从整体来看是基于标记-整理算法实现的，但是从局部（region之间）来看是基于复制算法实现的

- Full GC的时候仍然需要STW，所以应该尽量避免Full GC

执行阶段：

1. ==初始标记（Initial Marking）==

   与CMS收集器的第一个阶段很相似，只标记与GC ROOT**直接关联**的对象，**需要STW**

2. ==并发标记（Concurrent Marking）==

   从GC ROOT开始对堆中对象进行可达性分析，可以与用户线程并发执行

3. ==最终标记（Final Marking）==

   标记并发标记期间遗漏的对象，**需要STW**，可以并行标记（多个标记线程同时执行）

4. ==筛选回收（Live Data Counting and Evacuation）==

   对Region的回收价值和成本进行计算和排序，根据用户所期望的GC停顿时间来制定回收计划并按照计划进行回收，**需要STW**

   > 这个阶段实际上也是可以与用户线程并发执行的，但是由于回收的执行时间已经是用户可控制的，为了提高回收效率，G1选择了STW执行



### 为什么需要两次标记（可达性算法标记的对象一定会被回收吗）

如果对象在可达性分析中没有与GC ROOT的引用链的话，就会被第一次标记并进行一次筛选，筛选的条件是对象是否有重写`finalize()`方法并调用，如果没有的话就没有什么后顾之忧了，可以进行回收；

有的话就会把这个对象放入`F-Queue`的队列中，虚拟机会触发一个低优先级的`finalize`线程去执行。虚拟机不会保证`F-Queue`队列内的对象的`finalize`方法一定能执行完毕，如果`finalize()`执行缓慢或者发生了死锁，造成`F-Queue`一直等待，引起内存系统的崩溃。

GC会对`F-Queue`中的`finalize`执行完毕的对象进行第二次标记，被第二次标记的对象就会被移入等待回收的集合中等待回收，这时才成为真正的垃圾

JDK并不推荐使用`finalize()`方法进行资源回收，在JDK9中，已经将 `Object.finalize()` 标记为 `deprecated`：

- GC并不保证`finalize()`一定会执行/一定能执行完毕
- 如果`finalize()`方法使用不当，可能会引起死锁，导致内存系统崩溃
- 建议使用`try-with-resouces`或者`try- finally`机制对资源进行回收



### Safe Point

只有线程运行到安全点时，才能Stop The World：

- 循环的末尾（避免大循环导致线程长时间无法暂停）
- 方法临返回前 / 调用方法的call指令后
- 可能抛异常的位置



### Minor GC与Major GC？

- Minor GC
  - 只收集新生代
  - 触发条件：Eden区满
- Major GC / Full GC
  - 收集整个堆和方法区
  - 触发条件：
    - 通过Minor GC后进入老年代的大小大于老年代的可用内存
    - 老年代空间不够分配新的内存
    - 永久代空间内存不足（1.7及以前才有），1.8之后用元空间取代了永久代，减少了Full GC的频率
    - 由Eden区、From区复制到To区中，To区内存不足，这时则把对象移到老年代，若老年代仍然内存不足，则触发Major GC
    - 调用System.gc()，系统建议执行Full GC，但不保证一定会执行



### System.gc()

首先要明确的一点是，程序调用了`System.gc()`，代表显式的触发`Full GC`，但是系统不保证一定会执行。该方法只有在`justRanFinalization = true`的时候才会执行，而这个属性在调用`runFinalization()`的时候会被修改为`true`。

所以如果想要保证`System.gc()`一定会触发`Full GC`，就要与`runFinalization()`方法搭配使用

```java
System.gc();
runtime.runFinalizationSync();
System.gc();
```

> 不过不建议这样做，因为JVM有自己的GC策略，不建议手动GC



### 空间分配担保原则

JVM的分配内存机制有三大原则和空间分配担保机制：

1. 优先分配到Eden区

2. 大对象直接进入老年代

3. 长期存活的对象分配到老年代

4. 空间分配担保机制

   在发生`Minor GC`之前，虚拟机会检查老年代最大可用的连续内存空间是否大于新生代所有对象的总空间（是否具有足够可用空间）

   - 如果大于，说明有足够空间可用，则此次`Minor GC`是安全的
   - 如果小于，则虚拟机会查看`HandlePromotionFailure`参数是否允许担保失败
     - `HandlePromotionFailure = true`，则会继续检查老年代最大可用连续空间是否大于历次晋升到老年代的对象的平均大小。如果大于，则尝试进行一次`Minor GC`，但是这次`Minor GC`仍然是有风险的；如果小于，则进行一次`Full GC`
     - `HandlePromotionFailure = false`，说明不允许`Minor GC`失败的情况产生，直接进行一次`Full GC`



### 为什么说Minor GC是不安全/有风险的？

这是因为新生代采用复制算法进行GC，如果大量对象在Minor GC后仍然存活，那么这些对象就全部都会复制进S区，然后S区是很小的（Eden:From:To=8:1:1），当S区空间不足时就需要把无法容纳的对象放入老年代。

但是前提是老年代有足够的空间能够容纳对象，这时候就需要进行空间分配担保，由于有多少对象能在Minor GC之后存活是不可知的，因此只能通过过去的垃圾回收后晋升到老年代对象大小的平均值作为参考，来评估这次Minor GC的风险。

通过开启空间分配担保，虽然无法完全避免Minor GC的风险，但是在一定程度上降低了Full GC的执行频率



### 四种引用

1. ==强引用==

   是最常见的普通对象引用，比如new一个对象创建的就是强引用。在强引用面前，即使内存不足，JVM宁愿抛出OOM也不会通过回收强引用对象来解决内存不足问题即只要有强引用指向一个对象，那么就认为该对象是存活的。

   > 对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为null，就意味着此对象可以被垃圾收集了。但要注意的是，并不是赋值为null后就立马被垃圾回收，具体的回收时机还是要看垃圾收集策略的。

2. ==软引用==

   只有当内存不足时，JVM才会在抛出OOM之前清理软引用指向的对象。JVM会尽可能优先回收长时间闲置不用的软引用指向的对象，对那些刚构建的或者刚使用过的软引用指向的对象尽可能的保留。

   > 基于软引用的这些特性，**软引用可以用来实现很多内存敏感点的缓存场景**，即如果内存还有空闲，可以暂时缓存一些业务场景所需的数据，当内存不足时就可以清理掉，等后面再需要时，可以重新获取并再次缓存。这样就确保在使用缓存提升性能的同时，不会导致耗尽内存。

   ```java
   // 软引用通常和一个引用队列联合使用
   SoftReference<List<Foo>> ref = new SoftReference<List<Foo>>(new LinkedList<Foo>());
   // 程序中可以添加要被软引用指向的对象进入队列中
   List<Foo> list = ref.get();
   // 因为不知道是否已被回收，因此软引用使用前必须进行判空
   if (list != null)
   {
       list.add(foo);
   }
   ```

3. ==弱引用==

   弱引用指向的对象是十分临近`finalize`状态的情况，当弱引用被清除时，就符合`finalize`的条件了。

   > 弱引用与软引用最大的区别就是弱引用比软引用的生命周期更短暂。垃圾回收器会扫描它所管辖的内存区域的过程中，只要发现弱引用的对象，不管内存空间是否有空闲，都会回收它。

   ```java
   Object obj = new Object();
   WeakReference<Object> wf = new WeakReference<Object>(obj);
   obj = null;
   // 可能会返回null
   wf.get();
   // 返回是否被垃圾回收器标记为即将回收的垃圾
   wf.isEnQueued();
   ```

4. ==幻想引用/虚引用==

   如果一个对象仅持有虚引用，就相当于没有任何引用一样，随时都可能会被回收。不能通过虚引用访问对象，它仅仅提供了一种确保对象`finalize`后做某些事情（如`Post-Mortem`清理机制）的机制，也有人利用虚引用监控对象的创建和销毁

   > 如今的Java平台，开始采用`java.lang.ref.Cleaner` 代替`finalize`。`Cleaner` 的实现使用了幻象引用。这是一种常见的`post-mortem`清理机制。这个`Cleaner` 的操作都是独立的，有自己的运行线程，避免意外死锁的问题。

   ```java
   Object obj = new Object();
   PhantomReference<Object> pf = new PhantomReference<Object>(obj);
   obj=null;
   // 永远返回null
   pf.get();
   // 返回是否从内存中已经删除
   pf.isEnQueued();　　
   ```

> 除了虚引用，其他引用都可以访问到所指的对象。**利用软引用和弱引用，我们可以将访问到的对象，重新指向强引用，也就是人为的改变了对象的可达性状态**。所以对于软引用、弱引用之类，垃圾收集器可能会存在**二次确认**的问题，以确保处于弱引用状态的对象没有改变为强引用。



## 类加载

类的加载过程必须按照下述顺序按部就班的开始（指的是开始顺序，这些阶段通常是互相交叉的混合进行，会在一个阶段执行的过程中调用、激活另一个阶段）。而**解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持`Java`语言的运行时绑定特性（又称动态绑定）**

1. ==加载==

   1. 通过类的全限定类名获取该类的二进制流
   2. 将该二进制流的静态存储结构转为方法区的运行时数据结构
   3. 在堆中为该类生成一个`class`对象，作为方法区中该类的数据的访问入口

2. ==连接==

   1. ==验证==：验证该`class`文件中的字节流信息是否符合JVM要求，不会威胁到JVM的安全

   2. ==准备==：为该`class`对象的静态变量分配内存，初始化其初始值（零值）

      - 对**基本数据类型**来说，对于类变量（static）和全局变量，如果不显式地对其赋值而直接使用，则系统会为其赋予默认的零值，而对于局部变量来说，在使用前必须显式地为其赋值，否则编译时不通过。
      - 对于同时被`static`和`final`修饰的常量，必须在声明的时候就为其显式地赋值，否则编译时不通过；而只被**final**修饰的常量在使用前必须为其显式地赋值，系统不会为其赋予默认零值。
      - 对于**引用数据类型**来说，如数组引用、对象引用等，如果没有对其进行显式地赋值而直接使用，系统都会为其赋予默认的零值，即`null`。
      - 如果在**数组**初始化时没有对数组中的各元素赋值，那么其中的元素将根据对应的数据类型而被赋予默认的零值。

      > 这些内存都在方法区中进行分配，具体的赋值在初始化阶段完成
      >
      > 这里不包含用`final`修饰的`static`变量，因为==`final`在编译时就会分配内存了，准备阶段会显式初始化==

   3. ==解析==：该阶段主要完成符号引用转化为直接引用

      > **符号引用**就是一组符号来描述目标，可以是任何字面量；
      >
      > **直接引用**就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

3. ==初始化==：为类的静态变量赋予正确的初始值，即代码中显式初始化的值。在Java中对类变量进行初始值设定有两种方式：

   - 声明类变量时指定初始值
   - 使用静态代码块为类变量指定初始值

4. ==使用==

5. ==卸载==

   条件：

   1. 该类的所有实例对象都已经被回收（即Java堆中不存在任何该类的实例）

   2. 加载该类的`ClassLoader`已经被回收（因为JVM始终保持对类加载器的引用，而类加载器保持着对其加载的类的`class`对象的引用，所以如果类加载器没有被回收，那么该类的`class`对象始终是可达的）

      > 这也是自定义`ClassLoader`存在的意义，因为系统的`ClassLoader`永远是可达的，那么由它们加载的类永远不会被卸载

   3. 该类的`class`对象在任何地方都没有被引用，无法通过反射访问该类

   > 总结来说就是三个不可达：实例对象不可达、类加载器不可达、`class`不可达

   类的卸载其实就是在方法区中清空该类的信息



### 类初始化的时机

对类的使用，我们分为两种形式，一种是类的主动使用，一种是类的被动使用。只有当一个类被首次主动使用的时候，才会被初始化，否则不会被初始化。

**类的主动使用：**

1. 使用new关键字实例化对象的时候

2. 读取或者设置一个类的静态变量的时候（注意：被final修饰，已在编译期把结果放在常量池的静态变量除外）

   > 对于`final`修饰的类变量，如果该类变量的值在编译时就可以确定下来，那么这个类变量相当于“宏变量”，也可理解成“常量”。Java便一起去会在编译时直接把这个类变量出现的地方替换成它的值，因此即使程序使用该静态变量也不会导致该类的初始化。

3. 调用一个类的静态方法的时候

4. 使用`java.lang.reflect`包的方法对类进行反射的时候

5. 初始化一个类的子类的时候（初始化子类的时候会先初始化父类）

   > 但是这条规则不适用于接口：
   >
   > - 在初始化一个类时，并不会先初始化他实现的接口
   > - 在初始化一个接口时，并不会初始化他的父接口
   >
   > 因此，一个父接口并不会因为他的子接口或者实现类的初始化而初始化。**只有当程序首次首次使用特定接口的静态变量时，才会导致该接口的初始化。**

6. java虚拟机启动时，被表明为启动类的类（包含main方法的类）



**不会初始化的情况：**

1. 通过子类引用父类的静态变量，不会引起子类的初始化

2. 通过数组定义来引用类不会触发此类的初始化

   > 即 `A[] a = new A[20]` 不会引起A类的初始化

3. 常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化（即上面类的主动使用中第二点说到的）



### 初始化代码块

- ==构造初始化代码块==：没有`static`修饰，每次调用构造方法的时候都会执行。

  - 如果在创建对象时，有构造代码块会先执行构造代码块，然后再执行构造方法。

  - 构造代码块和显式初始化，地位相同，按照顺序执行

    ```java
    // 定义变量时显式初始化
    private int age = 23;
    // 构造代码块
    {
    	age = 50;
    }
    public static void main(String[] args) {
    	System.out.println(new Teacher().age);//50
    }
    
    
    // 构造代码块
    {
    	age = 50;
    }
    // 定义变量时显示初始化
    private int age = 23;
    
    public static void main(String[] args) {
    	System.out.println(new Teacher().age);//23
    }
    ```

- ==静态初始化代码块==：使用`static`修饰，只在初始化的时候执行一次

  - 当静态代码块和`main`方法属于一个类时，静态代码块比`main`方法执行早。

    > 因为访问`main`方法就会加载类，然后就会执行静态代码块

  - 静态代码块通常用于给类的成员变量进行初始化，不能对实例的成员变量初始化。

  - 静态代码块和显式初始化，地位相同，按照顺序执行

    ```java
    public class MyTest6 {
        public static void main(String[] args) {
            Singleton singleton = Singleton.getInstance();
          	// 1
            System.out.println("counter1:"+Singleton.counter1);
            // 2
            System.out.println("counter2:"+Singleton.counter2);
        }
    }
    class Singleton{
        public static int counter1;
        private static Singleton singleton = new Singleton();
    
        private Singleton(){
            counter1++;
            counter2++;
        }
      	// 在counter2++之后初始化为2
        public static int counter2=2;
    
        public static Singleton getInstance(){
            return singleton;
        }
    }
    ```

    1. 程序运行时首先加载带有`main`方法的`MyTest6`这个类。
    2. 然后执行`Singleton.getInstance();`，此时就会加载`Singleton`类
    3. 加载`Singleton`类的同时，会在类的准备阶段会类的成员变量分配内存并默认初始化，即`counter1=0`，`singleton=0`，`counter2=0`
    4. 接着为静态成员变量显式初始化，先执行`new Singleton();`创建`Singleton`对象时调用构造方法，执行完毕后，`counter1=1`，`counter2=1`
    5. 再对`counter2`显式初始化，`counter2=0`



### 运行时绑定/动态绑定

运行时绑定也叫动态绑定，它是一种调用对象方法的机制。

对象方法的执行过程：

1. 编译器查看对象声明类型和方法名，获得被调方法的候选方法

   > 由于重载和重写的存在，类中会有多个重名方法

2. 编译器查看参数列表，排除不符合对应参数类型的重载方法

3. 如果是`private`、`static`、`final`方法或者是构造器，编译器可以直接调用，这是**静态绑定**

   > 因为这些方法无法重写，所以编译的时候就能确定真正调用哪个方法

4. 否则采用**动态绑定**，JVM会优先调用对象的实际类型的方法，如果有对应方法就调用自己的，没有的话就到其父类中寻找，通俗来说就是优先使用`new`后面的类的方法。

   > 由于每次调用方法都要搜索，时间开销大，JVM预先为每个类建了一个**方法表**，其中列出了所有方法的签名和实际调用的方法
   >
   > 如`Father sample = new Son();`，并调用`sample`的方法，JVM就会搜索`Father`和`Son`的方法表

   ```java
   public class Father {
     public void method() {
     	System.out.println("父类方法，对象类型：" + this.getClass());
     }
   }
   
   public class Son extends Father {
       public static void main(String[] args) {
          Father sample = new Son();// 向上转型
          // 父类方法，对象类型：class samples.Son
          sample.method();
       }
   }
   ```

   > 使用子类对象`son`去调用方法`method`，由于在子类中没有重写该方法，所以会到父类中找到`method`方法并执行。如果子类中重写了`method`方法，则会调用子类的方法

   ```java
   public class Father {
       protected String name="父亲属性";
   
       public void method() {
          System.out.println("父类方法，对象类型：" + this.getClass());
       }
   }
   
   public class Son extends Father {
       protected String name="儿子属性";
   
       public void method() {
          System.out.println("子类方法，对象类型：" + this.getClass());
       }
   
       public static void main(String[] args) {
          Father sample = new Son();// 向上转型
          // 调用的成员：父亲属性
          System.out.println("调用的成员："+sample.name);
       }
   }
   ```

   > 需要明确的是，**动态绑定的范畴只是对象的方法**。由于`sample`声明为`Father`类型，所以`sample.name`得到的是父类的成员变量。如果想要获取子类的成员变量，就需要通过`get`方法



## 双亲委派机制

### 什么是双亲委派机制？

双亲委派机制即当一个类加载器收到加载请求时，不会先自己去尝试类加载，而是先委托父类进行加载，当父类无法进行加载时才交还给子类进行加载。因此所有的类加载请求都会被传递到`Bootstrap ClassLoader`



### 为什么要使用双亲委派机制？

1. ==避免类的重复加载==

   对于某一个类，无论是哪个类加载器要加载类，最终都会通过双亲委派机制由某一个固定的类加载器进行加载。当这个负责加载该类的类加载器发现自己已经加载过这个类了，就不会再次加载了，从而避免了类的重复加载

2. ==保护核心类库的加载==

   Java主要支持4种类加载器：

   - `Bootstrap ClassLoader` 启动类加载器：主要负责加载Java核心类库
   - `Extention ClassLoader` 标准扩展类加载器：主要负责加载扩展类
   - `Application ClassLoader` 应用类加载器：主要负责加载当前应用`classpath`下的所有类
   - `User ClassLoader` 用户自定义类加载器：用户自定义的类加载器，可以加载指定路径下的`class`文件

   ![preview](https://pic4.zhimg.com/v2-eb6ffa2110335ebb79b864e14a23c48b_r.jpg)

   > 注意它们之间不是继承关系，而是组合关系（即子类中定义一个父类加载器的属性）

   双亲委派机制使得核心类永远都是由`Bootstrap ClassLoader`负责加载的，避免了核心API被恶意篡改引起严重问题



### 破坏双亲委派机制

> 自定义类加载器时如果不想打破双亲委派机制，就只重写ClassLoader类中的findClass方法即可，无法被父类加载器加载的类最终会通过这个方法被加载

自定义类加载器，继承`ClassLoader`类，重写`loadClass`方法和`findClass`方法（先尝试自己加载，如果不行再委托父加载器加载）

#### Tomcat

1. tomcat对用户类库与类加载器的规划：

   1. /common目录：类库可被Tomcat和所有的Web应用程序共同使用
   2. /server目录：类库可被Tomcat使用，对所有的Web应用程序不可见
   3. /shared目录：类库可被所有的Web应用程序使用，但是对Tomcat不可见
   4. /WebApp/WEB-INF目录：类库仅仅可以被此Web应用程序使用，对Tomcat和其他Web应用程序都不可见

   > 为了支持这套目录结构，并对目录里面的类库进行加载和隔离，Tomcat自定义了多个类加载器
   >
   > ![image-20220318144945238](https://tva1.sinaimg.cn/large/e6c9d24egy1h0e1d5e41xj20fz0eaq4o.jpg)
   >
   > 其中WebApp类加载器和Jsp类加载器通常会存在多个实例，每一个Web应用程序对应一个WebApp类加载器，每一个JSP文件对应一个JSP类加载器
   >
   > 从上图的委派关系可以看出，Common类加载器加载的类可以被Catalina类加载器和Shared类加载器使用，而Catalina类加载器和Shared类加载器能加载的类相互隔离；WebAppClassLoader可以使用SharedClassLoader加载到的类，但各个WebAppClassLoader实例之间相互隔离

2. Web应用默认的类加载顺序是（打破了双亲委派规则）：
   1. 先从JVM的BootStrapClassLoader中加载。
   2. 加载Web应用下`/WEB-INF/classes`中的类。
   3. 加载Web应用下`/WEB-INF/lib/*.jap`中的jar包中的类。
   4. 加载上面定义的System路径下面的类。
   5. 加载上面定义的Common路径下面的类。

3. 如果在配置文件中配置了``，那么就是遵循双亲委派规则，加载顺序如下：
   1. 先从JVM的BootStrapClassLoader中加载。
   2. 加载上面定义的System路径下面的类。
   3. 加载上面定义的Common路径下面的类。
   4. 加载Web应用下`/WEB-INF/classes`中的类。
   5. 加载Web应用下`/WEB-INF/lib/*.jap`中的jar包中的类。

Tomcat，应用的类加载器优先自行加载应用目录下的`class`，加载不了才委派给父加载器，3个目的：

1.  对于各个`webapp`中的`class`和`lib`，需要相互隔离，不能出现一个应用中加载的类库会影响另一个应用的情况，并且`lib`要可共享，避免浪费资源
2.  使用单独的`ClassLoader`加载tomcat自身的类库，避免破坏
3.  热部署（修改项目代码后无需重启tomcat就能让修改生效）



#### OSGi

OSGi上线了模块化热部署，为每个模块都自定义了类加载器，需要更换模块时，模块与类加载器一起更换。



## 如何理解“不使用的对象应该手动赋值为null”？

1. 局部变量表是GC中重要的`GC ROOT`

   JVM中的GC一般采用可达性分析算法，即从`GC ROOT`开始搜索可达（存活）的对象。创建的对象都会存放在局部变量表的一个槽中

2. 局部变量表的可复用性

   局部变量表中的槽是可以复用的，如果代码已经离开了某个变量的作用域，那么该变量的槽可以被重复利用，之后如果产生了新的变量，就会覆盖这个槽中原有的变量。但是如果之后不再有任何对局部变量表的读写操作，这个变量就会一直占用着这个槽，局部变量表也就会一直保持着对它的引用，GC标记的时候就会误以为该变量是存活的，并不会回收这个变量。

> 这种关联没有及时被打断，在大多数情况下没什么影响。但是如果后面的代码有一些耗时很长的操作，前面又定义了一堆占用很大内存实际上不会再被使用的变量，手动赋为`null`值还是有意义的。



## 为什么static方法中不可以引用super、this？

1. 初始化时机

   static方法在类加载的时候就已经存在了，但是对象是在创建时才分配到堆中。也就是说static方法的出现时机把super、this要早，一个先存在的方法尝试去引用一个现在可能还未创建的对象，自然是不符合逻辑的

2. 局部变量表

   一个方法只能引用自己局部变量表中的对象，普通方法的局部变量表的第一个槽中的变量一定是this，即用于传递方法所属对象实例的引用；但是static方法是直属于类的，其局部变量表内并没有this变量，所以也就不可以引用了



## OOM

#### 内存溢出与内存泄漏

- 内存溢出

  空闲内存不足，并且垃圾收集器无法提供更多内存（垃圾回收之后仍然内存不足）

  > 在抛出OOM之前，垃圾收集器通常会被触发，尝试解决内存不足的问题。如果垃圾回收后仍无法提供足够内存，就会抛出OOM

- 内存泄漏

  某些对象不会再被使用到，但是无法被GC回收

  > 内存泄漏不会立刻引起程序崩溃，而是逐渐占用内存，直到内存不足时抛出OOM

  1. **静态集合类引发内存泄漏**

     放入静态集合中的对象，即使显式赋值为null，也不会被回收，因为静态集合中仍然保存着对其的强引用。因此使用完后要记得把对象从集合中移除/清空集合

  2. **接收器、监听器注册没取消造成的内存泄漏**

  3. **连接没有及时关闭造成的内存泄漏**

  4. **单例引发的内存泄漏**

     单例的不当使用，如果创建的单例对象只需要使用一次，这个对象就会一直占用内存，并且无法回收

     ```java
     public class AppManager {
         private static AppManager instance;
         private Context context;
       
         private AppManager(Context context) {
           // ✕：如果传入的是Activity的Context，这个Context对应的Activity退出时，由于其Context被单例对象持有，其生命周期等于整个应用程序的生命周期，所以Activity对象无法正常被回收
           this.context = context;
           // √：Application 的生命周期就是整个应用的生命周期，所以这将没有任何问题。
           this.context = context.getApplicationContext();
         }
       
         public static AppManager getInstance(Context context) {
           if (instance == null) {
           	instance = new AppManager(context);
           }
           return instance;
           }
         }
     ```

  5. **匿名内部类/非静态内部类和异步线程引发的内存泄漏**

     ```java
     public class MainActivity extends AppCompatActivity {
       private static TestResource mResource = null;
       @Override
       protected void onCreate(Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
         setContentView(R.layout.activity_main);
         if(mManager == null){
           mManager = new TestResource();
         }
         //...
       }
       class TestResource {
         //...
       }
     }
     ```

     如上述代码，`TestResource`非静态内部类默认持有一个对外部类`MainActivity`的引用。

     >  因此非静态内部类可以访问外部类的成员变量

     而`MainActivity`又创建了一个静态的`TestResource`实例，这样就会导致该静态实例会一直持有对`MainActivity`的引用，导致`Activity`的内存资源不能正常回收。

     正确的做法为：将该内部类设为静态内部类（不会持有对外部类的引用）或将该内部类抽取出来封装成一个单例



### 排查思路

1. 修改JVM启动参数，直接增加内存

   > -Xmx、-Xms

2. 检查错误日志，查看OOM错误前是否有其他异常或错误

3. 对程序进行分析，寻找可能发生内存溢出/内存泄漏的问题

   - 对数据库查询中是否有一次性获取全部数据的操作

     如果数据库中数据量十分大，一次性全部读出来可能就会引发OOM。因此一般建议采用分页查询

   - 是否存在死循环或递归

   - 是否存在大循环重复创建对象实例

   - 检查静态集合对象是否有使用完后未清除的问题

     集合对象始终保持对所存储对象的引用，会使得这些对象无法回收

4. 使用内存查看工具动态查看内存情况



## 堆和栈的区别

1. 内存分配

   - 栈由系统自动分配内存，无需程序员对内存进行管理（自动出栈）
   - 堆由程序员自己申请，是动态分配的，需要手动释放内存（c中malloc申请内存后需要手动free，Java中JVM添加了GC机制自动回收内存）

2. 大小限制

   - 栈是向低地址扩展的数据结构，是一块连续的内存空间，其最大容量是系统预先规定好的，如果超过内存将抛出`StackOverflow`，空间较小

     > `-Xss <size>`调整规定的栈大小
     >
     > 经典`StackOverflow`：死循环递归
     >
     > 如果栈支持动态扩容，则满了之后抛出OOM；如果不支持动态扩容则抛出StackOverflow。HotSpot不支持动态扩容

   - 堆是向高地址扩展的数据结构，是使用链表相连的离散的内存空间，获得空间比较灵活，比较大，如果超过内存限制将抛出`OutOfMemory`

     > `-Xmx <size>` 堆最大值
     >
     > `-Xms <size>` 堆初始值
     >
     > `-Xmn <size>` 年轻代大小
     >
     > `-XXSurvivorRatio`年轻代中`Eden`区与`Survivor`区大小的比值

3. 存储内容

   - 虚拟机栈是线程私有的，存放局部变量表、方法参数等信息
   - 堆是线程共享的，几乎所有的对象都要在堆上分配内存空间，是垃圾回收的主要对象



## 常量池

1. ==Class文件常量池==

   class文件是一组以字节为单位的二进制数据流，在java代码的编译期间，我们编写的java文件就被编译为.class文件格式的二进制数据存放在磁盘中，其中就包括class文件常量池，用于存放编译器生成的各种**字面量**和**符号引用**

   - 字面量就是我们所说的常量概念，如文本字符串、被声明为final的常量值等。

   -  符号引用是一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可（它与直接引用区分一下，直接引用一般是指向方法区的本地指针，相对偏移量或是一个能间接定位到目标的句柄）。一般包括下面三类常量：

     - 类和接口的全限定名

     - 字段的名称和描述符

     - 方法的名称和描述符

2. ==运行时常量池==

   当类加载到内存中后，jvm就会将class常量池中的内容存放到运行时常量池中。

   运行时常量池相对于class常量池一大特征就是具有动态性，java规范并不要求常量只能在运行时才产生，也就是说运行时常量池的内容并不全部来自class常量池，在运行时可以通过代码生成常量并将其放入运行时常量池中，这种特性被用的最多的就是`String.intern()`。

   > 在解析阶段，会把符号引用替换为直接引用，解析的过程会去查询字符串常量池，也就是`StringTable`，以保证运行时常量池所引用的字符串与字符串常量池中是一致的

3. ==字符串常量池==

   字符串常量池是JVM所维护的一个字符串实例的引用表，在HotSpot VM中， 它是一个叫做`StringTable`的全局表。在字符串常量池中维护的是字符串实例的引用，底层C++实现就是一个`Hashtable`。这些被维护的引用所指的字符串实例，被称作”被驻留的字符串”或”`interned string`”或通常所说的”进入了字符串常量池的字符串”。

   > JDK7时字符串常量池从方法区移到了堆中，由于`String.intern()`发生了改变，因此`String Pool`中也可以存放放于堆内的字符串对象的引用

4. ==基本类型包装类对象常量池==

   java中基本类型的包装类的大部分都实现了常量池技术，这些类是 Byte,Short,Integer,Long,Character,Boolean,另外两种浮点数类型的包装类则没有实现。另外上面这5种整型的包装类也只是在对应值<=127时才可使用对象池，也即对象不负责创建和管理大于127的这些类的对象。

### 总结

1. 字符串常量池在VM中只有一份，存放的是字符串常量的引用值。
2. class常量池是在编译的时候每个class都有的，在编译阶段，存放的是常量的符号引用。
3. 运行时常量池是在类加载完成之后，将每个class常量池中的符号引用值转存到运行时常量池中，也就是说，每个class都有一个运行时常量池，类在解析之后，将符号引用替换成直接引用，与字符串常量池中的引用值保持一致

![img](https://img-blog.csdn.net/20171115215708642?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2FuZ2JpYW8wMDc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



## finally

### 代码执行结果分析

- 当`finally`块中有`return`语句时，将会覆盖函数中其他`return`语句

  ```java
  // return finally
  public String s() {
    try {
    	return "try";
    } catch (Exception e) {
    	return "catch";
    } finally {
    	return "finally";
    }
  }
  
  // return i=10
  public int main() {
      int i = 0;
      try {
          i = 1;
          return i++;
      } catch (Exception e) {
          i = 9;
      } finally {
          i = 10;
        	// 真正执行的return语句
          return i;
      }
  }
  ```

- 如果`try/catch`块中有`return`语句的时候，会先把返回值保存起来，然后再执行`finally`块。`finally`块执行完毕后再把之前保存的返回值返回

  - 如果返回值是基本数据类型，`finally`块中并不会改变到已经保存的返回数据，所以返回值仍然是`try/catch`的操作结果
  - 如果返回值是引用类型，`finally`块中如果改变了引用的属性，之前已经保存的返回的引用（句柄）并不会改变，但是通过其引用找到的属性值最终仍然是改变了的

  ```java
  // return i=1
  public int main() {
      int i = 0;
      try {
          i = 1;
        	// 如果是++i则返回2
          return i++;
      } catch (Exception e) {
          i = 9;
      } finally {
          i = 10;
       		// 打印出i=10，即i实际上在finally中已经改变为10，但是返回值之前已经保存过为1了
        	System.out.println("finally-i=" + i);
      }
    	return i;
  }
  
  // return a.change = "finally"
  public String main() {
      A a = new A();
      try {
        a.change = "try";
        return a;
      } catch (Exception e) {
        a.change = "catch";
      } finally {
        a.change = "finally";
      }
      return a;
  }
  ```

- 如下代码，其中的`return num()`实际上等效于

  ```java
  int sum = num();
  return sum;
  ```

  在`num()`方法执行时就打印了“num 代码块被执行！”这句话，然后再进入`finally`块，最后再`return`。因此`num`的语句比`finally`的语句要先打印

  ```java
  public static void main(String[] args) {
    System.out.println("main 代码块中的执行结果为：" + myMethod());
  }
  
  public static int myMethod() {
    try {
      System.out.println("try 代码块被执行！");
      return num();
    } finally {
      System.out.println("finally 代码块被执行！");
    }
  }
  
  public static int num() {
    System.out.println("num 代码块被执行！");
    return 2;
  }
  
  // try 代码块被执行！
  // num 代码块被执行！
  // finally 代码块被执行！
  // main 代码块中的执行结果为：2
  ```



### 出现在程序中的finally块一定会执行吗？

不一定：

1. 在程序执行到`try`块中的代码之前方法已经返回，就不会执行`finally`块
2. 程序执行到`try`块中，并且`try`块中调用了`System.exit(0)`方法，表示立即退出虚拟机
3. 如果程序运行`try`块时，线程突然死亡



### 排除上述情况，JVM是怎么保证finally块一定执行的？

编译成字节码指令时，`finally`中的内容会被复制一份，分别放到`try`后和`catch`后，这样就保证无论是`try`执行完毕还是因为异常进入到`catch`块中最后都会执行`finally`了（在`return`之前执行）



## 创建对象

### 创建对象的过程

1. ==检查该类是否能在class常量池中定位到对应的符号引用==

2. ==检查该类是否已经加载、连接过，如果没有先进行类的加载==

3. 然后对对象进行==内存分配==（类加载后即可得知所需内存大小）

   - 指针碰撞

     如果内存绝对规整，则只需要一个指针指向空闲与非空闲区域的分界，虚拟机只需要移动这个指针就可以了。指针碰撞是线程不安全的

   - 空闲列表

     如果内存不规整，虚拟机需要维护一个空闲列表来记录那些内存是可用的，分配内存的时候从空闲列表中取出一块可用内存即可。

   **分配内存需要考虑线程安全问题**：

   - 使用CAS操作保证操作的原子性
   - TLAB：给每个线程预先分配一个内存空间，每个线程只能在这个内存空间里分配内存，互不干扰（在这个雨分配的内存空间里是单线程环境）

4. ==初始化==

   - ==设置零值==(实例数据初始化)，保证对象即使没有赋初值也可以直接使用
   - ==对象头设置==（如何找到类元数据、哈希码、gc分代年龄）
   - ==执行init方法==（非静态块+初始化+构造方法）

   

### 创建对象的方法

1. ==使用new关键字==

   ```java
   A a = new A();
   ```

2. ==Class对象的newInstance()方法==

   ```java
   String className = "com.fdd.Test";
   // 动态加载类的class对象
   Class clasz = Class.forName(className);
   Test t = (Test) clasz.newInstance();
   ```

3. ==构造函数的newInstance()方法==

   ```java
   Constructor<Test> constructor;
   try {
   	constructor = Test.class.getConstructor();
   	Test t = constructor.newInstance();
   } catch (Exception){
   	e.printStackTrace();
   }
   ```

4. ==序列化==

   需要实现Serializable接口

   ```java
   String filePath = "sample.txt";//序列化的路径
   Test t1 = new Test();
   try {
     //t1开始序列化，把序列化的数据写入sample.txt中
     FileOutputStream fileOutputStream = new FileOutputStream(filePath);
     ObjectOutputStream outputStream = new ObjectOutputStream(fileOutputStream);
     outputStream.writeObject(t1);
     outputStream.flush();
     outputStream.close();
     //t2开始反序列化，通过sample.txt中序列化后的数据进行反序列化
     FileInputStream fileInputStream = new FileInputStream(filePath);
     ObjectInputStream inputStream = new ObjectInputStream(fileInputStream);
     Test t2 = (Test) inputStream.readObject();
     inputStream.close();
     System.out.println(t2.getName());
   } catch (Exception ee) {
   	ee.printStackTrace();
   }
   ```

5. ==clone方法==

   相当于创建一个副本

   ```java
   Test t1 = new Test("java的架构师技术栈");
   Test t2 = (Test) t1.clone();
   System.out.println(t2.getName());
   ```

6. ==Unsafe==

   Unsafe类使Java拥有了像C语言的指针一样操作内存空间的能力，同时也带来了指针的问题。过度的使用Unsafe类会使得出错的几率变大，因此Java官方并不建议使用的，官方文档也几乎没有。

   ```java
   private static Unsafe getUnsafe() {
     try {
       // 通过反射方式得到unsafe对象，不能直接创建Unsafe
       Field field = Unsafe.class.getDeclaredField("theUnsafe");
       field.setAccessible(true);
       Unsafe unsafe = (Unsafe) field.get(null);
       return unsafe;
     } catch (Exception e) {
       e.printStackTrace();
     }
     return null;
   }
   // 拿到getUnsafe返回的对象后，调用native方法创建对象
   Object event = unsafe.allocateInstance(Test.class);
   ```



## this引用溢出

```java
public class ThisEscape {
  	// 直接在监听器的构造方法中就开始注册监听器并使用
　　public ThisEscape(EventSource source) {
　　　　source.registerListener(new EventListener() {
　　　　　　public void onEvent(Event e) {
　　　　　　　　doSomething(e);
　　　　　　}
　　　　});
　　}
 
　　void doSomething(Event e) {
　　}
 
　　interface EventSource {
　　　　void registerListener(EventListener e);
　　}
 
　　interface EventListener {
　　　　void onEvent(Event e);
　　}
 
　　interface Event {
　　}
}
```

这将导致`this`逸出，所谓逸出，就是**在不该发布的时候发布了一个引用**。

在这个例子里面，当我们实例化`ThisEscape`对象时，会调用`source`的`registerListener`方法，这时便启动了一个监听线程，而且这个线程持有了`ThisEscape`对象（调用了对象的`doSomething`方法），但此时`ThisEscape`对象却没有实例化完成（还没有返回一个引用），所以我们说，此时造成了一个`this`引用逸出，即还没有完成的实例化`ThisEscape`对象的动作，却已经暴露了对象的引用。其他线程访问还没有构造好的对象，可能会造成意料不到的问题。

> 只有当构造函数返回时，`this`引用才应该从线程中逸出。构造函数可以将`this`引用保存到某个地方，只要其他线程不会在构造函数完成之前使用它。

正确构造过程⬇️

```java
public class SafeListener {
　　private final EventListener listener;
 
　　private SafeListener() {
　　　　listener = new EventListener() {
　　　　　　public void onEvent(Event e) {
　　　　　　　　doSomething(e);
　　　　　　}
　　　　};
　　}
 
　　public static SafeListener newInstance(EventSource source) {
    	// 监听器构造完成后再注册监听器并使用
　　　　SafeListener safe = new SafeListener();
　　　　source.registerListener(safe.listener);
　　　　return safe;
　　}
 
　　void doSomething(Event e) {
　　}
 
　　interface EventSource {
　　　　void registerListener(EventListener e);
　　}
 
　　interface EventListener {
　　　　void onEvent(Event e);
　　}
 
　　interface Event {
　　}
　}
```



## 对象一定都是在堆中分配内存的吗？

在编译期间，JIT会对代码做很多优化。其中有一部分优化的目的就是减少内存堆分配压力，其中一种重要的技术叫做**逃逸分析**。

逃逸分析的作用就是分析对象的引用范围，是否会逃逸出方法外，即其作用域是否只在方法内部（不作为返回值，不会被外部线程访问到），如果对象不会逃逸，则直接在栈中分配内存，这样子**对象就可以随着方法的运行自动出栈，无需GC**

使用逃逸分析，还可以做如下优化：

1. 上面提到的把==堆分配转化为栈分配==

2. ==同步省略==

   如果一个对象被发现只能被一个线程访问到，则省去同步操作

3. ==分离对象/标量替换==

   有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中

   ```java
   class P {
   	int a;
   	int b;
   	
   	public P (int a, int b) {
   		this.a = a;
   		this.b = b;
   	}
   }
   
   public void main() {
     // 该句编译后可能会被优化成int a = 1; int b = 2;
   	P p = new P(1, 2);
     System.out.print("a:" + a + "b:" + b);
   	// 后续代码不会使用到对象p
   }
   ```



## 对象的访问方式

```java
Person person = new Person();
```

- `Person person` 表示一个本地引用，存储在JVM栈的本地变量表中，表示一个`reference`类型数据；
- `new Person()`作为实例对象数据存储在堆中；
- 堆中还记录了`Object`类的类型信息（接口、方法、field、对象类型等）的地址，这些地址所执行的数据存储在方法区中；

访问方式：

1. ==通过句柄访问==

   JVM堆中会专门有一块区域用来作为句柄池，存储相关句柄所执行的实例数据地址（包括在堆中地址和在方法区中的地址）。这种实现方法由于用句柄表示地址，因此**十分稳定**。

   <img src="https://img2018.cnblogs.com/blog/1064427/201904/1064427-20190403100714903-2101877239.png" alt="img" style="zoom:50%;" />

2. ==通过直接指针访问==

   通过直接指针访问的方式中，`reference`中存储的就是对象在堆中的实际地址，在堆中存储的对象信息中包含了在方法区中的相应类型数据。这种方法最大的优势是**速度快**，在`HotSpot`虚拟机中用的就是这种方式。

   <img src="https://images0.cnblogs.com/blog/406312/201309/21174413-e7b4a7cdec984c2881a56ad776d54354.png" alt="img" style="zoom: 67%;" />

   

## JVM调优命令



## 堆和栈

### 为什么要区分堆和栈

1. 软件设计：堆代表数据，栈代表处理逻辑。把数据和逻辑分开，是一种隔离、模块化的软件设计思想
2. 共享内存实现线程通信：堆与栈的隔离，使得堆中的内容可以被多个栈共享（多个线程可以访问同一个对象）
3. 动态增长：栈因为运行时的需要，比如保存系统运行时的上下文，需要进行地址段的划分，而且因为其职能向上增长，所以存储内容的能力有限。而堆中的对象是可以根据需要动态增长的，这样在栈里记录一个指针指向堆中的地址就可以了



### 堆和栈的区别

1. 内存管理方式：堆的内存由程序员分配和释放（C中使用malloc分配内存，free释放内存；Java中使用new分配内存，JVM的GC机制自动回收内存）；栈的内存由OS自动分配和释放，无需手动控制

2. 地址增长方向：堆的内存地址由低到高；栈的内存地址由高到低

3. 空间大小：堆的大小远远大于栈的大小

4. 分配效率：堆是由C/C++提供的库函数来进行内存管理，效率较低，容易产生内存碎片；栈是由操作系统进行内存管理，在硬件级别对栈提供了支持：分配专门的寄存器存放栈的地址、压栈出栈由专门的指令执行，因此栈的内存分配效率很高

5. 存放内容：几乎所有的对象都要在堆上分配内存；栈存储局部变量表、操作数、动态链接和方法返回值等信息

   > 对象不一定全部都在堆上分配内存，如果经过逃逸分析后发现该对象并不会逃逸到方法外（作用域仅在方法内），则可能会把堆分配转化为栈分配
