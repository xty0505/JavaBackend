# 同步容器类

## Collections.synchronizedxxx

除了Java本身自带的线程安全容器(Vector、Stack、Hashtable)，还有一种通过Collections.synchronized封装线程不安全的容器而变成线程安全的容器。

例如

- Collections.synchronizedList
- Collections.synchronizedMap

线程安全类的复合操作也可能导致整体程序线程不安全，可以通过在客户端上加锁来避免，即在复合操作外面加上一把锁



## fail-fast

大部分集合内部使用iterator进行遍历，在循环中使用同步锁的开销很大，而iterator的创建时轻量级的，所以集合内如果有并发修改的操作，集合会**快速失败**（fail-fast）。集合在迭代过程中发现有修改时，会抛出**ConcurrentModificationException**异常。

- 多个线程并发修改集合会fail-fast
- 单个线程在遍历同时修改也会fail-fast

## fail-safe

安全失败机制，遍历时不在原集合上进行访问，而是复制原集合在拷贝的集合上遍历，不会抛出异常。

JUC下的容器都是fail-safe，例如CopyOnWriteArrayList

- 需要复制集合，开销大
- 无法保证读取到的数据是**原始集合**中的数据



## 操作系统中的并发工具

操作系统级别的并发容器，即数据结构的设计

### 信号量

semaphore，取值0或任意整数，0代表不需要唤醒，正值代表唤醒次数

- down：semaphore--
- up: semaphore++
- 原子操作



### 互斥量

mutex

- 两种状态：**解锁unlock**,**加锁locked**
- 线程进入互斥区域时检查mutex，加锁
- 退出时解锁



### Futexes

- fast user space mutex快速用户空间互斥
- 由**内核服务**和**用户库**组成
- 内核服务提供一个等待队列，允许多个进程在上面排队（不运行），避免陷入内核



### Pthreads中的互斥量

- mutex
- 条件变量condition，不满足条件的线程也会被阻塞



### 管程

### 消息传递

### 屏障

### 避免锁：读-复制-更新



## JUC

### ConcurrentHashMap

![image-20210109160855178](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109160855178.png)

- 采取分段锁实现，将数组分为一个个Segement，不会导致线程独占访问。在并发环境下可以实现更高的吞吐量，在单线程环境下只损失非常小的性能
- 弱一致性，不会抛出ConcurrentModificationException异常
- size/empty/containsKey等方法不返回**精确值**，只是参考，在并发环境下size/empty等用处很小，因为他们总是在变化



### ConcurrentMap

接口，继承Map接口，提供4个新的原子方法

#### ConcurrentNavigableMap

支持并发访问，并且允许其**视图**的并发访问，视图时集合中的一段数据序列

#### ConcurrentSkipListMap

线程安全的有序哈希表，底层跳表

#### ConcurrentSkipListSet

底层基于ConcurrentNavigableMap，线程安全的有序集合

#### CopyOnWriteArrayList

ArrayList的线程安全版本，所有可变操作如add/set都是重新创建了一个副本，每次并发写都会创建新的副本，因此存在一定开销，适合读多写少的场景

#### CopyOnWriteArraySet

与CopyOnWriteArrayList类似



## BlockingQueue

JDK 1.5添加的新工具类，阻塞队列，继承Queue并扩展了功能

- 队列为空获取元素时、队列满存储元素时都会发生阻塞
- 不允许添加null
- 一般用于生产者-消费者模型

#### LinkedBlockingQueue

底层链表

#### ArrayBlockingQueue

底层数组，默认不保证线程公平地访问队列

#### PriorityBlockingQueue

支持优先级的阻塞队列

#### DelayQueue

支持延时获取元素的无阻塞队列，元素只有延迟过期后才能使用

#### TransferQueue

接口，继承BlockingQueue，实现该接口的类在生产者-消费者模型中，生产者会一致阻塞直到它生产的元素被消费

#### LinkedTransferQueue

无界的基于链表的TransferQueue



## BlockingDeque

JDK 1.6提出，分别扩展Queue和BlockingQueue

![image-20210109164644816](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109164644816.png)

#### ConcurrentLinkedDeque

无界的双向链表并发队列

#### LinkedBlockingDeque

链表，双向阻塞队列



## 同步工具类

### Semaphore

- acquire 获取信号量
- release 释放信号量

### CountDownLatch

- await 等待
- countDown 减少计数器，直到 0 唤醒 await

### Future/FutureTask

![image-20210109165656320](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109165656320.png)

- Future对Runable/Callable任务的执行结果进行操作，可以get获取执行结果
- Future是接口，FutureTask实现了RunableFuture(继承Runable和Future接口)

### Barrier

阻塞一组线程直到某个事件发生

![image-20210109171134108](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109171134108.png)

- 线程到达barrier后调用await，所有线程到达之后barrier打开释放所有线程，重置以下次使用
- await超时后，所有await的线程抛出BrokenBarrierException
- 成功通过Barrier，await返回一个唯一索引，用来选举一个新的leader

### Exchanger

用于线程之间协作的工具类，数据交换

![image-20210109171433293](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109171433293.png)

- 提供一个同步点，在同步点成对的线程调用exchange反复交换数据