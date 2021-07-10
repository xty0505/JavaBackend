# Java 锁分类

![image-20210109172037571](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109172037571.png)

- 是否对资源加锁：乐观锁，悲观锁
- 资源已锁定，线程是否阻塞：自旋锁
- 多线程并发访问资源：无锁，偏向锁，轻量级锁，重量级锁
- 公平性：公平锁，非公平锁
- 锁是否可重复获取：可重入锁，不可重入锁
- 多个线程是否能获取同一个锁：共享锁，独占锁



## 乐观/悲观锁

乐观锁悲观锁是一种设计思想，并不是真实存在的锁

- 悲观锁：持有数据时总是锁住资源，synchronized,ReentrantLock等排它锁都是悲观锁
- 乐观锁：不上锁，实现方案：版本号机制，CAS。juc.atomic原子变量类使用了CAS乐观锁的思想实现

### 乐观锁的实现机制

#### 版本号机制

在数据上加一个version字段，当执行写操作且成功时，version=version+1，线程在更新数据时，读取数据的同时读取version值，提交更新时比较version

#### CAS

基于硬件实现，不需要进入内核，无切换线程开销

- 需要读写的内存地址 V
- 旧的预期值 A
- 拟写入的新值B
- 当且仅当内存地址 V 的值= A ，才将 V 的值改为 B ，否则什么都不做

### 乐观锁的缺点

- ABA问题，JDK 1.5后的AtomicStampedReference可以避免，或者给值V加上一个表示修改次数的标记符
- 自旋循环开销大



## 自旋锁

如果资源以已经被占有，线程可以一直循环等待资源释放，即自旋，另一种方式将自己阻塞起来，等待被唤醒，即互斥锁

### 优缺点

- 锁的竞争不激烈，自旋锁占用 CPU 的开销小于线程上下文切换的开销
- 锁的竞争比较激烈会导致 CPU 一直被自旋锁占用



## synchronized

synchronized有四种状态：无锁，偏向锁，轻量级锁，重量级锁

### Java对象头

Hotspot对象头包含两部分数据

- Mark Word：默认存储对象的HashCode、分代年龄、锁标志位信息。会随着对象的状态复用自己的存储空间
- Class Point：类元数据指针，JVM根据这个指针来判断对象属于那个类

![image-20210109181333367](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109181333367.png)

### Monitor

JVM 通过进入和退出 Monitor 对象实现同步，任何一个对象都有一个monitor与之关联

- monitorenter
- monitorexit

synchronized通过对象内部的监视器锁（monitor）实现，本质依赖于底层操作系统的 Mutex Lock 实现，而操作系统实现线程之间的切换需要从用户态转换到内核态，开销很大，因此synchronized是重量级锁

### 锁的分类

![image-20210109182307585](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109182307585.png)

#### 无锁

![image-20210109182525378](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109182525378.png)

无锁状态下 Mark Word 保存了对象的 hashcode 值和分代年龄用于 GC。

#### 偏向锁

![image-20210109182634880](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109182634880.png)

偏向锁是在多线程无竞争的情况下，同一线程多次获得资源，如果每次都是加轻量级锁，仍需要执行 CAS 操作。这时候可将锁优化为偏向锁，通过 CAS 操作将 Mark Word 中一部分修改为当前线程的 ID。

线程每次进入和退出同步代码块，只需要比较 Mark Word 中是否存储了当前线程的 ID，

- 如果存在则表示当前线程持有偏向锁，无需额外花费 CAS 操作加锁解锁。
- 如果不存在说明另一个线程尝试竞争偏向锁， 通过 CAS 修改 Mark Word 中原来线程 ID 为当前线程 ID。若 CAS 成功，则当前线程获得偏向锁。

**释放过程**

如果另外一个线程尝试竞争偏向锁失败，则开始撤销偏向锁。需要等待原持有偏向锁的线程到达全局安全点（在这个时间点上没有字节码正在执行），暂停该线程并检查其状态。

- 如果原持有偏向锁的线程不处于活跃状态或者已经退出同步代码块，则线程释放锁。将对象头设置为无锁状态。
- 如果原持有偏向锁的线程处于同步代码快内，则升级为轻量级锁。

#### 轻量级锁

![image-20210109184141802](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109184141802.png)

轻量级锁使用 CAS 操作避免使用 Monitor 对象加锁，从而提升效率。

**加锁过程**

首先会在当前线程的栈帧中创建一个名为锁记录（Lock Record）的空间，一部分用于存储 Mark Word 的拷贝（Displaced Mark Word），另一部分 owner 指针用于指向对象的 Mark Word 。

![image-20210317202039197](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210317202039197.png)

加锁时使用 CAS 操作尝试将对象头的 Mark Word 更新为该线程锁记录的指针。

- 如果更新成功，则说明该线程成功获得轻量级锁，对象的 Mark Word 状态变成('00')。

![image-20210317203841741](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210317203841741.png)

- 如果更新失败，检查 Mark Word 是否指向当前线程的锁记录
  - 如果是这说明当前线程已经拥有了该对象的锁，那么可以直接进入同步代码快
  - 否则说明锁被其他线程占有，当前线程会尝试**自旋**一定次数来获取锁，如果仍然失败，那么轻量级锁就要升级为重量级锁，对象的 Mark Word 指向 Monitor 对象，当前线程进入 EntryList。

**解锁过程**

- CAS 尝试用锁记录中的 Displaced Mark Word 中的数据替换对象的 Mark Word。
  - 成功则释放轻量级锁
  - 否则唤醒阻塞在重量级锁等待队列上的线程

#### 重量级锁

![image-20210109184159862](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210109184159862.png)

当轻量级锁加锁时 CAS 不成功，说明多个线程之间产生了竞争，此时锁升级为重量级锁，

![image-20210317204750675](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210317204750675.png)

#### 其他优化

1.**自适应自旋锁**

重量级锁切换线程需要进入内核态完成，效率不高。往往共享数据的锁定状态只会持续很短一段时间，采用自旋（CPU 执行空循环）的方式等待一会儿就能获取到锁，从而**避免线程切换的开销**。

在轻量级锁获取的过程中，线程会自旋 CAS 一定次数，如果自旋的时间过程，会导致 CPU 资源白白浪费。

JDK 1.6 后引入了自适应自旋的优化方式，能够根据自旋获取锁的历史记录来自适应地调整自旋次数，避免过长的自旋浪费 CPU 资源。

2.**锁消除**

编译器在运行时消除一些被检测到不可能存在竞争的同步块。

```java
public String concatString(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append("a");
    sb.append("b");
    sb.append("c");
    return sb.toString();
}
```

3.**锁粗化**

JVM 检测到一串零碎的操作都对同一个对象加锁，会把锁同步的范围粗化到整个操作序列的外部。避免多次加锁解锁。

##  公平性

### 公平锁

线程获取锁的顺序按照线程加锁的顺序来分配，即FIFO

### 非公平锁

获取锁的抢占机制，新线程加锁时资源刚好释放锁，它就可能立即获得锁，而不管在它之前就阻塞的线程，因此可能导致一些线程永远获得不了锁



## ReentrantLock

可重入锁，互斥锁，具有与synchronized相同的方法和监视器锁的语义，但是有更多的扩展功能

- 可以实现公平、非公平锁
- 可中断
- 条件变量



### 独占锁

同一时刻只能被一个线程拥有，synchronized，JUC中的Lock

### 共享锁

能够被多个线程持有锁，如果某个线程对资源加上共享锁后，其他线程也只能加共享锁。获得共享锁的线程只能**读数据**，不能修改数据

