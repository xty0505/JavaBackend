# 线程池

池化技术相比大家已经屡见不鲜了，线程池、数据库连接池、Http 连接池等等都是对这个思想的应用。池化技术的思想主要是为了**减少每次获取资源的消耗，提高对资源的利用率。**

线程池提供限制和管理线程资源的功能，还维护了一些基本统计信息，如已完成任务的数量。

- 降低资源消耗。通过重复利用已创建的线程降低创建和销毁带来的消耗
- 提高响应速度。任务到达时不用等待创建线程
- 提高线程的可管理性。统一分配，调度，监控线程池中的线程资源

## Executor 框架

Executor 框架是 Java5 之后引进的，在 Java 5 之后，通过 Executor 来启动线程比使用 Thread 的 start 方法更好，除了更易管理，效率更好（用线程池实现，节约开销）外。

Executor 框架不仅包括了线程池的管理，还提供了线程工厂、队列以及拒绝策略等，Executor 框架让并发编程变得更加简单。

![image-20210302135103303](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210302135103303.png)

### Runnable/Callable

实现 Runnable/Callable 接口的类才能够被 `ThreadPoolExecutor` 或 `ScheduledThreadPoolExecutor` 执行

### Future

FutureTask 是 Future 的实现类，可以用来异步获取线程执行结果

实现 Runnable/Callable 接口的类提交给 `ThreadPoolExecutor` 或 `ScheduledThreadPoolExecutor` 执行（调用 submit() 方法返回一个 FutureTask 对象）

### Executor

核心接口 Executor， 以及继承它的 ExecutorService

![image-20210302134709322](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210302134709322.png)

## ThreadPoolExecutor

```java
    /**
     * 用给定的初始参数创建一个新的ThreadPoolExecutor。
     */
    public ThreadPoolExecutor(int corePoolSize,//线程池的核心线程数量
                              int maximumPoolSize,//线程池的最大线程数
                              long keepAliveTime,//当线程数大于核心线程数时，多余的空闲线程存活的最长时间
                              TimeUnit unit,//时间单位
                              BlockingQueue<Runnable> workQueue,//任务队列，用来储存等待执行任务的队列
                              ThreadFactory threadFactory,//线程工厂，用来创建线程，一般默认即可
                              RejectedExecutionHandler handler//拒绝策略，当提交的任务过多而不能及时处理时，我们可以定制策略来处理任务
                               ) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

最重要的参数

- corePoolSize：核心线程数定义了线程池中最小可以同时运行的线程数量
- maximumPoolSize：当队列中存放的任务达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数。
- workQueue：当新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在队列中。

其他重要参数

- keepAliveTime：当线程池中的线程数量大于 `corePoolSize` 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 `keepAliveTime`才会被回收销毁；
- unit：keepAliveTime 的单位
- threadFactory：executor 创建新线程的时候会用到。
- handler：拒绝策略，当任务数量超过最大线程数时，采用的策略

![image-20210302135804951](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210302135804951.png)

拒绝策略定义

- ThreadPoolExecutor.AbortPolicy：抛出 `RejectedExecutionException`来拒绝新任务的处理。
- ThreadPoolExecutor.CallerRunsPolicy：调用执行自己的线程运行任务，也就是直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务，如果执行程序已关闭，则会丢弃该任务。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果您的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略。
- ThreadPoolExecutor.DiscardPolicy：不处理新任务，丢弃
- ThreadPoolExecutor.DiscardOldestPolicy：丢弃最早的未处理任务请求。

### execute 执行原理

![image-20210302150534189](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210302150534189.png)

### Runable v.s. Callable

- Runable 接口不会返回结果，Callable 可以返回结果
- Runable 不会抛出抽查异常， Callable 可以抛出

Executors 可以实现 Runnable 和 Callable 对象之间的相互转换

### execute() v.s. submit()

- execute() 用于提交不需要返回值的任务，无法判断任务是否被线程池成功执行
- submit() 方法用于提交需要返回值的任务。线程池会返回一个 `Future` 类型的对象，通过这个 `Future` 对象可以判断任务是否执行成功 ，并且可以通过 `Future` 的 `get()`方法来获取返回值，`get()`方法会阻塞当前线程直到任务完成，而使用 `get（long timeout，TimeUnit unit）`方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

### shutdown() v.s. shutdownNow()

- shutdown()：关闭线程池，线程池的 SHUTDOWN。线程池不再接受新任务，但是**队列中的任务必须执行完毕**
- shutdonwNow()：关闭线程池，线程的状态变成 STOP。线程池会终止当前正在运行的任务，并停止处理排队的任务并返回正在等待的任务列表

### isTerminated() v.s. isShutdown()

- isShutDown：当调用 shutdown() 方法返回为 true
- isTerminated：当调用 shutdown() 方法后，并且提交的任务完成后返回true

## Executors

### FixedThreadPool

```java
   /**
     * 创建一个可重用固定数量线程的线程池
     */
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

`corePoolSize` 和 `maximumPoolSize` 都被设置为传入的参数，即无救急线程

![image-20210302152913364](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210302152913364.png)

不推荐使用 FixedThreadPool 的原因

- maximumPoolSize 和 keepAliveTime 都变成无效参数
- 使用无界队列 LinkedBlockingQueue 导致任务可能无限堆积，最后 OOM

### SingleThreadExecutor

```java
   /**
     *返回只有一个线程的线程池
     */
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

![image-20210302153348902](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210302153348902.png)

与 FixedThreadPool 的问题相同，而且核心线程数为 1 导致队列快速堆积任务

### CachedThreadPool

```java
    /**
     * 创建一个线程池，根据需要创建新线程，但会在先前构建的线程可用时重用它。
     */
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

![image-20210302153707393](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210302153707393.png)

`CachedThreadPool`允许创建的线程数量为 Integer.MAX_VALUE ，可能会创建大量线程，从而导致 OOM。

## ScheduledThreadPoolExecutor

`ScheduledThreadPoolExecutor` 主要用来在给定的延迟后运行任务，或者定期执行任务。

`ScheduledThreadPoolExecutor` 使用的任务队列 `DelayQueue` 封装了一个 `PriorityQueue`，`PriorityQueue` 会对队列中的任务进行排序，执行所需时间短的放在前面先被执行(`ScheduledFutureTask` 的 `time` 变量小的先执行)，如果执行所需时间相同则先提交的任务将被先执行(`ScheduledFutureTask` 的 `squenceNumber` 变量小的先执行)。

### 与 Timer 的比较

`ScheduledThreadPoolExecutor` 和 `Timer` 的比较：

- `Timer` 对系统时钟的变化敏感，`ScheduledThreadPoolExecutor`不是；
- `Timer` 只有一个执行线程，因此长时间运行的任务可以延迟其他任务。 `ScheduledThreadPoolExecutor` 可以配置任意数量的线程。 此外，如果你想（通过提供 ThreadFactory），你可以完全控制创建的线程;
- 在 `TimerTask`  中抛出的运行时异常会杀死一个线程，从而导致 `Timer` 死机，即计划任务将不再运行。`ScheduledThreadExecutor` 不仅捕获运行时异常，还允许您在需要时处理它们（通过重写 `afterExecute` 方法`ThreadPoolExecutor`）。抛出异常的任务将被取消，但其他任务将继续运行。

### 运行机制

![image-20210302154442383](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210302154442383.png)

1. `scheduleAtFixedRate()` 方法或者 `scheduleWithFixedDelay()` 方法，会向 `ScheduledThreadPoolExecutor` 的 `DelayQueue` 添加一个实现了 `RunnableScheduledFuture` 接口的 `ScheduledFutureTask`
2. 线程池中线程 从 `DelayQueue` 中获取已经到期的 `ScheduledFutureTask` （time 大于等于当前系统时间的任务）
3. 执行这个 `ScheduledFutureTask` 并修改 time 为下次执行的时间
4. 重新添加该 `ScheduledFutureTask` 到延迟队列中

![image-20210302155333733](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210302155333733.png)

## 线程池大小确定

- 线程数量设置太小：如果同一时间有大量任务需要处理，导致队列中堆积过多任务，可能出现队列满载或者 OOM 的情况，CPU没有得到重复利用
- 线程数量太大：线程上下文切换频繁，增加线程执行时间，影响整体效率

简单且适用范围较广的方法论

- CPU 密集型任务（ N + 1 ）：这种任务消耗的主要是 CPU 资源，可以将线程数设置为 N（CPU 核心数）+1，比 CPU 核心数多出来的一个线程是为了防止线程偶发的缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。
- I/O 密集型任务（2N ）：这种任务应用起来，系统会用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 来处理，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 2N。



