# 面试-JUC并发

## synchronized

### 实现原理

`synchronized`同步代码块是通过`monitorenter`和`monitorexit`指令实现的，当程序运行到`monitorenter`指令时，尝试获取锁，也就是`monitor`，`monitor`存放在每个Java对象的对象头中，其内置一个计数器，若成功获取锁，则锁计数器+1。当运行到`monitorexit`指令时则锁计数器-1，表示释放锁。

对于`synchronized`方法，没有这两个指令，取而代之的是`ACC_SYNCHRONIZED`标识，该标识指明了该方法是一个同步方法，当JVM发现方法带有这个标识时，会自动对其采用同步操作



### synchronized与volatile的区别

1. ==本质区别==
   - `volatile`本质是告诉JVM当前在寄存器的量不可信，必须到主存中读取；
   - `synchronized`是锁定该资源，只允许当前线程访问，其他线程一直阻塞等待，需要拿到锁才能访问
2. ==作用范围==
   - `volatile`只能作用在变量
   - `synchronized`可以作用在类、方法、变量
3. ==保证的性质==
   - `volatile`只能保证**可见性**，不能保证原子性
   - `synchronized`可以保证**原子性和可见性**
4. ==线程阻塞==
   - `volatile`不会引起线程阻塞
   - `synchronized`会使拿不到锁的线程阻塞
5. ==编译器优化==
   - `volatile`标记的变量创建过程不会被编译器优化（禁止指令重排）
   - `synchronized`标记的变量可能会被编译器优化



### synchronized与lock的区别

1. ==作用范围==
   - `synchronized`可作用于变量、方法、类
   - `Lock`只能作用于代码块
2. ==获取和释放==
   - `synchronized`会自动获取和释放，无法得知自己是否获取到锁
   - `Lock`需要手动获取和释放，可以知道自己是否成功获取锁
3. ==中断==
   - `synchronized`是不可中断的，即阻塞等待的线程会一直阻塞等待直到获取到锁
   - 如果调用`lockInterruptibly()`方法获取锁，那么处于等待过程中的线程可以被中断
4. ==`Lock`不是Java语言内置的，`synchronized`是Java语言的关键字，因此是内置特性。`Lock`是一个类，通过这个类可以实现同步访问==



### synchronized与ReentrantLock的联系与区别

1. ==都是可重入锁==

   可重入锁即递归锁，当一个线程获取到锁后其执行到的代码需要再次获取这个锁，那么该线程的锁计数器会加1，然后可以直接执行，而无需重新获取锁。（释放锁时要锁计数器为0，即持有的所有锁都释放了才可以释放）

2. ==实现==

   - `sycnhronized`是Java的一个关键字，在JVM层面实现
   - `ReentrantLock`是`Lock`的一个实现类，在API层面实现

3. ==ReentrantLock增加的高级功能==

   - 等待可中断-`lockInterruptibly()`
   - 可以指定为公平锁/非公平锁，`synchronized`只能是非公平锁
   - `ReentrantLock`类线程对象可以注册在指定的`Condition`中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。 在使用`notify()/notifyAll()`方法进行通知时，被通知的线程是由 JVM 选择的

4. ==死锁问题==

   - `synchronized`是由JVM自动获取和释放锁的
   - `ReentrantLock`需要手动获取和释放锁，使用不当可能会造成死锁问题



### synchronized锁膨胀的过程

在锁对象的对象头里有一个threadId的字段，第一次访问时该字段为空，并且使该线程持有**偏向锁**，然后把该字段设为该线程的ID。

再次进入时会先判断threadId与当前线程ID是否一致，如果一致则直接使用即可，不一致的话说明该锁正在被其他线程持有，该锁升级为**轻量级锁**，当前线程通过自旋尝试获取锁

如果自旋一定次数后还不能获取到锁的话，就升级为**重量级锁**

> 重量级锁依赖于系统底层的同步函数实现，会涉及到操作系统内核状态的切换、进程上下文的切换，因此重量级锁开销很大



### JVM对synchronized的优化

1. ==锁膨胀==

2. ==锁消除==

   通过JIT编译去除不可能存在竞争的锁

3. ==锁粗化==

   扩大锁的范围，避免频繁的获取锁和释放锁（如把循环内的锁移动到循环外）

4. ==自旋锁与自适应自旋锁==

   自适应自旋锁即根据前一个持有锁的状态以及上一次在同个锁的自旋时间动态调整自旋次数



## 线程池

### 参数

1. `corePoolSize`：核心线程数

2. `maximumPoolSize`：最大线程数

3. `keepAliveTime`：非核心线程存活时间

4. `unit`：时间单位

5. `workQueue`：阻塞队列

6. `threadFactory`：负责新建线程的工厂

7. `handler`：拒绝策略（饱和策略）

   - `AbortPolicy`：默认的拒绝策略，直接跑出`RejectedExecutionException`，用户可捕获这个异常并自己编写异常处理逻辑

   - `DiscardPolicy`：抛弃策略，什么都不做，直接抛弃被拒绝的任务

   - `DiscardOldestPolicy`：抛弃最老策略，即抛弃阻塞队列的队首任务（最老任务）

     > 如果阻塞队列是一个优先队列，那么会导致抛弃最高优先级任务。因此该拒绝策略最好不要跟优先级队列放在一起使用

   - `CallerRunsPolicy`：在调用者线程中执行该任务。即把任务回退到调用线程池执行任务的主线程来执行任务，由于主线程要执行任务，因此主线程至少在一段时间内不能提交任务。



### 执行任务流程

1. 判断任务是否小于核心线程数
   - 如果小于则新建线程运行任务
   - 如果大于则进行判断2
2. 阻塞队列是否已满
   - 如果否则该任务进入阻塞队列等待执行
   - 如果是则进入3
3. 是否超过最大线程数
   - 如果是则执行拒绝策略
   - 如果不是则新建非核心线程运行任务



### 常用的Java线程池

|         线程池          |      阻塞队列       |
| :---------------------: | :-----------------: |
|   newFixedThreadPool    | LinkedBlockingQueue |
| newSingleThreadExecutor | LinkedBlockingQueue |
|   newCachedThreadPool   |  SynchronousQueue   |
| newScheduledThreadPool  |  DelayedWorkQueue   |

- `newFixedThreadPool`

  返回一个固定数量的线程池，阻塞队列为`LinkedBlockingQueue`，容量为`Integer.MAX_VALUE`，可以认为是无限长

  - 如果当前线程数 < 指定的线程数量，则新建线程执行任务
  - 如果当前线程数 >= 指定的线程数量，则任务进入阻塞队列等待
  - 线程执行完任务后如果没有接收到新任务，则直接死亡

- `newSingleThreadExecutor`

  返回只有一个线程的线程池，阻塞队列为`LinkedBlockingQueue`，容量为`Integer.MAX_VALUE`，可以认为是无限长（实际上就是只有一个线程的`newFixedThreadPool`）

- `newCachedThreadPool`

  返回一个可根据实际情况调整线程数的线程池，核心线程数为0，最大线程数为`Integer.MAX_VALUE`（相当于没有限制），阻塞队列为`SynchronousQueue`，容量为1（相当于一个点）

  - 如果有空闲的线程则直接执行任务
  - 如果没有空闲的线程，则新建一个线程执行
  - 空闲线程60s后会被回收

- `newScheduledThreadPool`

  创建一个可以指定线程的数量的线程池，但是这个线程池还带有延迟和周期性执行任务的功能，类似定时器。



## 什么是this引用逃逸？

https://www.cnblogs.com/jian0110/p/9369096.html























