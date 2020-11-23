---
title: Java并发包
date: 2020-11-23 11:34:41
categories: 
- Java
tags:
- Java
- description: Java并发包、阻塞队列、线程池
---
#### 并发包锁

1. ##### synchronized同步锁

    `synchronized`同步锁(隐式锁、独占排他锁)问题：

    1. 不能控制阻塞，不能灵活控制锁的释放，使用`synchronized`同步锁,线程会在三种情况下释放锁:

        1. 线程执行完同步代码块/方法，释放锁
        2. 线程执行时发生异常，此时JVM会让线程自动释放锁
        3. 在同步代码块/方法中，锁对象执行了`wait()`方法，线程释放锁

    2. 在读多写少的场景中，效率低下。当多个线程都只进行读操作时，所有线程都只能同步进行，只能有一个读线程可以进行操作，其他线程只能被阻塞而无法进行操作

2. ##### ReentrantLock重入锁(独占锁)

    > java.util.concurrent包下有重入锁`ReentrantLock`和读写锁`ReentrantReadWriteLock`、`StampedLock`(Java8新的读写锁)

    重入锁的底层实现使用`AbstractQueuedSynchronizer`。
    - 公平锁：多个线程按照申请锁的顺序，先来后到的原则获得锁。
    - 非公平锁：多个线程可能并不是按照申请锁的顺序，允许新线程插队
  
    一般情况下，使用公平锁在多线程访问时，在线程的调度上面的开销比较大，所以总体吞吐量会比较低。

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

