---
title: Java并发包
date: 2020-11-23 11:34:41
keywords: Java,并发包
categories: 
- Java
tags:
- Java
description: Java并发包、阻塞队列、线程池
---
#### 并发包锁

1. ##### 乐观锁

  乐观锁采用乐观的思想处理数据，在每次读取数据时都认为别人不会修改该数据，所以不会上锁，但在更新时会判断在此期间别人有没有更新该数据，通常采用在写时先读出当前版本号然后加锁的方法。

  Java中通过`CAS(Compare And Swap, 比较和交换)`操作实现的，CAS是一种原子更新操作，在对数据操作之前首先会比较当前值跟传入的值是否一样，如果一样则更新，否则不执行更新操作。

2. #####  悲观锁

  悲观锁采用悲观思想处理数据，在每次读取数据时都认为别人会修改数据，所以每次在读写数据时都会上锁，这样别人想读写这个数据时就会阻塞、等待直到拿到锁。

  Java中基于`AQS(Abstract Queued Synchronized, 抽象的队列同步器)`架构实现。AQS定义了一套多线程访问共享资源的同步框架，许多同步类的实现都依赖于它，例如常用的Synchronized、ReentrantLock、Semaphore、CountDownLatch等。该框架下的锁会尝试以CAS乐观锁去获取锁，如果获取不到，则会转为悲观锁(如RetreenLock)。

3. ##### 自旋锁

  自旋锁认为：如果持有锁的线程能在很短的时间内释放锁资源，那么那些等待竞争锁的线程就不需要做内核态和用户态之间的切换进入阻塞、挂起状态，只需等一等(也叫做自旋)，在等待持有锁的线程释放锁后即可立即获取锁，这样就避免了用户线程在内核状态的切换上导致的锁时间消耗。

  线程在自旋时会占用CPU，在线册灰姑娘长时间自旋获取不到锁时，将会产生CPU的浪费，甚至有时线程永远无法获取锁而导致CPU资源被永久占用，所以需要设定一个自旋等待的最大时间。在线程执行的时间超过自旋等待的最大时间后，线程会退出自旋模式并释放其持有的锁。

4. ##### 可重入锁

  可重入锁也叫做递归锁，指在同一线程中，在外层函数获取到该锁之后，内层的递归函数仍然可以继续获取该锁。

5. ##### 公平锁与非公平锁

  - 公平锁(Fair Lock)：指在分配锁前检查是否有线程在排队等待获取该锁，优先将锁分配给排队时间最长的线程。
  - 非公平锁(Nonfair Lock)：指在分配锁时不考虑线程等待的情况，直接尝试获取锁，在获取不到锁时再排队到队尾等待。

6. ##### 共享锁和独占锁

  Java并发包提供的加锁模式分为独占锁和共享锁：
  - 独占锁：也叫互斥锁，每次只允许一个线程持有该所，ReentrantLock为独占锁的实现。
  - 共享锁：允许多个线程同时获取该锁，并发访问共享资。ReentrantReadWriteLock中的读锁为共享锁的实现。

7. ##### 分段锁

  分段锁是一种思想，用于将数据分段并在每个分段上都单独加锁，把锁进一步细粒度化，以提高并发效率。ConcurrentHashMap在内部就是使用分段锁实现的。

8. ##### 重量级锁、轻量级锁和偏向锁
  
  锁的状态一共有四种：无锁、偏向锁、轻量级锁和重量级锁。

  - 重量级锁：是基于操作系统的互斥量(Mutex Lock)而实现的锁，会导致进程在用户态和内核态之间切换，相对开销较大。

    synchronized在内部基于监视器锁(Monitor)实现，监视器锁基于底层操作系统的Mutex Lock实现，因此属于重量级锁。所以synchronized的运行效率不高。

  - 轻量级锁：相对于重量级锁而言的。轻量级锁的核心设计是在没有多线程竞争的前提下，减少重量级锁的使用以提高系统性能。轻量级锁适用于线程交替执行同步代码块的情况(即互斥操作)，如果同一时刻有多个线程访问同一个锁，则将会导致轻量级锁膨胀为重量级锁。
  - 偏向锁：用于在某个线程获取某个锁之后，消除这个线程锁重入的开销，看起来似乎是这个线程得到了该锁的偏向(偏袒)。

    偏向锁的主要目的是在同一个线程多次获取某个锁的情况下尽量减少轻量级锁的执行路径，因为轻量级锁的获取及释放需要多次CAS原子操作，而偏向锁只需要在切换ThreadID时执行一次CAS原子操作，因此可以提高锁的运行效率。

9.  ##### synchronized 和 ReentrantLock不同点

  - ReentrantLock显式获取和释放锁；synchronized隐式获取和释放锁。为了避免程序出现异常而无法正常释放锁，在使用ReentrantLock时必须在finally控制块中释放锁。
  - ReentrantLock可响应中断、可轮回，为处理锁提供了更多的灵活性。
  - ReentrantLock是API级别的，synchronized是JVM级别的。
  - ReentrantLock可以定义公平锁。
  - ReentrantLock通过Condition可以绑定多个条件。
  - 二者底层实现不一样：synchronized是同步阻塞，采用的是悲观并发策略；Lock是同步非阻塞，采用的是乐观并发策略。
  - 可以通过Lock知道有么有成功获取锁，而synchronized无法做到
  - Lock可以通过分别定义读写锁提高多线程读的效率。


#### 阻塞队列

阻塞队列：可阻塞插入和可阻塞移除元素的队列

- 可阻塞插入：当队列满时，线程向队列插入元素将会被阻塞，直到队列有空闲位置可用。

- 可阻塞移除：当队列为空时，线程从队列获取元素时将会被阻塞，直到队列中有新数据入队。

> 使用场景一般是在“生产者-消费者”模式中。JUC中，线程池本质上就是个“生产者-消费者”模式的实现。

1. ##### 阻塞队列
    JUC提供了7种适合不同场景的阻塞队列(均实现`BlockingQueue`接口):

    1. **`ArrayBlockingQueue`**：基于数组实现的有界队列阻塞队列
    2. **`LinkedBolockingQueue`**：基于链表实现的有界阻塞队列
    3. **`PriorityBlockingQueue`**：支持按优先级排序的无界阻塞队列
    4. **`DelayQueue`**：优先级队列实现的无界阻塞队列
    5. **`SynchronousQueue`**：不存储元素的阻塞队列
    6. **`LinkedTransferQueue`**：基于链表实现的无界阻塞队列
    7. **`LinkedBolckingDeque`**：基于链表实现的双向无界阻塞队列

    添加方法如下：
    - **`add(e)`**：队列满时抛出异常
    - **`offer(e)`**：队列满时不会阻塞，返回false，成功返回true
    - **`put(e)`**：队列满时会被阻塞直到队列有空闲位置可用
    - **`offer(e, time, unit)`**：队列满时线程会阻塞一段时间，超出指定时间还未成功直接退出
    
    移除方法如下：
    - **`remove(e)`**：队列为空抛出异常
    - **`poll(e)`**：队列为空不会阻塞，返回null
    - **`take(e)`**：队列为空会阻塞到队列有数据可用时
    - **`poll(time, unit)`**：队列为空线程会阻塞一段时间，超过指定时间仍无数据可用直接返回null

2. ##### ArrayBolckingQueue

    `ArrayBolckingQueue`在构造时指定容量，后面不能改变。适用于“生产”和“消费”速度比较稳定且基本匹配的情况下。
    全局使用独占锁`ReentrantLock`，可以使用公平/非公平策略。只能有一个线程进行入队或出列，高并发下可能会成为性能瓶颈。

3. ##### ArrayBolckingQueue

    `LinkedBlockingQueue`可以不指定容量，默认使用`Integer.MAX_VALUE`，入队和出列分别使用`ReentrantLock`，只使用非公平策略，并发性能比`ArrayBolckingQueue`好。

#### 线程池

1. ##### 线程创建和销毁步骤：

    1. 创建Java线程实例。线程是一个对象实例，会在堆种分配内存，创建线程需要时间和内存。
    2. JVM为线程创建其私有资源，虚拟机栈和程序计数器。
    3. 执行start方法启动线程，操作系统为线程创建对应的内核线程，线程处于就绪状态。内核线程属于操作系统资源，创建也需要时间和内存。
    4. 线程被操作系统CPU调度器选中后，线程开始运行任务。
    5. 线程在运行过程中会被CPU不断切换运行。
    6. 线程运行完毕，Java线程被垃圾回收器回收。

2. ##### 线程池`ThreadPoolExecutor`参数

    - **`corePoolSize`**：核心线程数

        线程池刚创建时，线程数默认为0

    - **`maximumPoolSize`**：最大线程数
    - **`workQueue`**：任务队列，缓存已经提交但尚未被执行的任务
        
        `workQueue`是一个阻塞队列，可选如下：
        - **`ArrayBlockingQueue`**
        - **`LinkedBlockingQueue`**：线程池工厂`Executors`中`newFixedThreadPool`使用了此队列
        - **`SynchronousQueue`**：不存储元素的队列，每个插入操作须等另一个线程调用移除操作，否则一直处于阻塞。线程池工厂`Executors`中`newCachedThreadPool`使用了此队列
        - **`PriorityBlockingQueue`**：具有优先级的无限阻塞队列，无界队列，按元素权重出队，如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行
   
    - **`keepAliveTime`**：空闲线程的存活时间

        默认情况下只回收非核心线程，可设置核心线程回收
        ```java
        allowCoreThreadTimeOut(true);
        ```

    - **`unit`**：`keepAliveTime`的单位(TimeUnit)
    - **`threadFactory`**：线程工厂(用于指定如何创建一个线程)
       
        默认是`Executors`中的`DefaultThreadFactory`，线程格式为`pool-{线程池id}-thread-{线程id}`。可以自定义线程工厂类设置名字，方便故障定位。

    - **`handler`**：拒绝策略(工作队列已满且线程池中线程已达上限时的处理策略)

        - **`AbortPolicy`**：默认策略，无法处理新任务时抛出异常
        - **`DiscardPolicy`**：新任务忽略不执行，丢弃
        - **`DiscardOldestPolicy`**：抛弃任务队列中等待最久的任务，将新任务添加到队列中
        - **`CallerRunPolicy`**：新任务使用调用者所在的线程来执行任务
  
        > 也可以实现`RejectedExecutionHandler`接口自定义策略

    ```mermaid
    graph TB
        start(提交任务) --> coreSize{工作线程数<br/>小于核心线程数}

        coreSize --否--> queue{任务队列未满}

        coreSize --是--> newThread(新建线程<br/>并执行任务)

        newThread --> over(任务结束)

        queue --否--> workSize{工作线程数<br/>小于最大线程数}

        queue --是--> putQueue(任务入队<br/>空闲线程执行)

        putQueue --> over

        workSize --是--> newThread2(新建线程<br/>并执行任务)

        workSize --> handler(拒绝策略)

        newThread2 --> over

    ```