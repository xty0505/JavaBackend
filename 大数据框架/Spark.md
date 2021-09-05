# Spark

## 提交流程

### Yarn-Client

Yarn-Client 模式 Driver 运行在 **Client 端**，能够观察 Driver 端的执行和日志。

```bash
$SPARK_HOME/bin/spark-submit 
	--master yarn 
	--deploy-mode client 
	--class org.apache.spark.examples.SparkPi $SPARK_HOME/examples/jars/spark-examples*.jar
```

在 Yarn-Client 模式下， Spark 应用的执行总体流程如下。

![image-20210309182343182](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210309182343182.png)

1. Client 端提交 Spark 应用，提交时首先找到主类，然后进入主类的 main 函数执行应用。此时一般会创建 SparkSession 和 ResourceManager 进行通信。
2. ResourceManager 创建一个 ApplicationMaster，该 ApplicationMaster 向 ResourceManager 请求资源( CPU、内存、磁盘、网络)，ResourceManager 返回资源列表。
3. ApplicationMaster 申请资源 Container 分配给 NodeManager 并启动 Executor。
4. Executor 反向注册到 Driver 端，Dirver 将 Spark 应用划分为多个 Stage 和 Task，然后把 Task 提交给 Executor 端执行，Executor 返回执行进度和状态，Job 完成后返回结果。

### Yarn-Cluster

Yarn-Cluster 模式下 Diver 运行在 **ApplicationMaster**的容器中，因此无法在 Client 端查看任务执行情况，只能在 Yarn 上查看日志。

```bash
$SPARK_HOME/bin/spark-submit 
	--master yarn 
	--deploy-mode cluster 
	--class org.apache.spark.examples.SparkPi $SPARK_HOME/examples/jars/spark-examples*.jar
```

![image-20210309184509143](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210309184509143.png)

1. Client 端提交 Spark 应用，提交时首先找到主类，然后进入主类的 main 函数执行应用。此时一般会创建 SparkSession 和 ResourceManager 进行通信。
2. ResourceManager 创建一个 ApplicationMaster，该 ApplicationMaster 向 ResourceManager 请求资源( CPU、内存、磁盘、网络)，ResourceManager 返回资源列表。此时 **Driver 也运行在 ApplicationMaster 上**。
3. ApplicationMaster 申请资源 Container 分配给 NodeManager 并启动 Executor。Executor 反向注册到 Driver 端，Driver 端将 Task 提交给 Executor，并监控任务执行情况， Executor 端执行任务，将执行进度返回给 Driver。
4. 执行完成后 Driver 将结果返回给 Client。

### Standalone

Spark 自己实现的资源调度框架，主要组件有 Client，Master 和 Worker。Driver 既可以运行在 Master 节点上，也可以运行在本地 Client 端。

- spark-shell 提交任务时运行在 Master 上
- spark-submit 提交任务时运行在 Client上

![image-20210309190555814](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210309190555814.png)

1. Driver 端提交 Spark 应用，初始化 SparkSession (SparkContext )，包括 DAGScheduler 和 TaskScheduler。TaskScheduler 注册到 Master 节点并申请资源。
2. Master 根据资源申请要求和 Worker 的心跳报告决定在哪个 Worker 上分配资源并启动 Executor。所有 Executor 反向注册到 Driver 上，然后开始执行 Spark 应用中的逻辑代码。
3. 遇到一个执行算子产生一个 Job，DAGScheduler 将 Job 划分为多个 Stage，每个 Stage 即一个 TaskSet，TaskScheduler 将 TaskSet 分配给相应的 Worker 执行。
4. Executor 执行任务，执行完成将结果返回给 Driver，Driver 向 Master 注销资源。

详细版图解如下。![image-20210309191907624](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210309191907624.png)

## 调度策略

Spark 的任务调度主要通过 TaskScheduler 实现，内部维护一个任务队列。

Spark 有两种可调度的实体

- Pool：调度池，可以有子 Pool，Spark 中 rootPool 默认是一个无名的 Pool
- TaskSetManager：封装了 TaskSet

主要有两种调度策略 

### FIFO

Job 的优先级越高越先执行，优先级相同时 Stage 越靠前越先执行。

对于同一个 Job 的 Stage，其依赖的 Stage 执行完之前，DAGScheduler 都不会将它提交到调度队列中，因此若有两个无入度的 Stage，则先调度 Stage Id 小的。

```scala
private[spark] class FIFOSchedulingAlgorithm extends SchedulingAlgorithm {
  override def comparator(s1: Schedulable, s2: Schedulable): Boolean = {
    val priority1 = s1.priority
    val priority2 = s2.priority
    var res = math.signum(priority1 - priority2)
    if (res == 0) {
      val stageId1 = s1.stageId
      val stageId2 = s2.stageId
      res = math.signum(stageId1 - stageId2)
    }
    if (res < 0) {
      true
    } else {
      false
    }
  }
}
```

FIFO 模式下 rootPool 直接管理 TaskSetManager

![image-20210309195938638](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210309195938638.png)

### Fair

哪个任务最需要执行就调度谁。

- 饥饿任务优先
- 都饥饿，则资源占用比小的优先
- 都不饥饿，权重大的优先
- 否则任务名字典排序靠前的优先

```scala
private[spark] class FairSchedulingAlgorithm extends SchedulingAlgorithm {
    override def comparator(s1: Schedulable, s2: Schedulable): Boolean = {
    //最小共享,可以理解为执行需要的最小资源即CPU核数，其他相同时，所需最小核数小的优先调度
    val minShare1 = s1.minShare
    val minShare2 = s2.minShare
    //运行的任务的数量
    val runningTasks1 = s1.runningTasks
    val runningTasks2 = s2.runningTasks
    //是否有处于挨饿状态的任务，看可分配的核数是否少于任务数，如果资源不够用，那么处于挨饿状态
    val s1Needy = runningTasks1 < minShare1
    val s2Needy = runningTasks2 < minShare2

     //最小资源占用比例，这里可以理解为偏向任务较轻的   
    val minShareRatio1 = runningTasks1.toDouble / math.max(minShare1, 1.0).toDouble
    val minShareRatio2 = runningTasks2.toDouble / math.max(minShare2, 1.0).toDouble

     //权重，任务数相同，权重高的优先
    val taskToWeightRatio1 = runningTasks1.toDouble / s1.weight.toDouble
    val taskToWeightRatio2 = runningTasks2.toDouble / s2.weight.toDouble
    var compare: Int = 0

     //挨饿的优先
    if (s1Needy && !s2Needy) {
        return true
    } else if (!s1Needy && s2Needy) {
        return false
    } else if (s1Needy && s2Needy) {
        //都处于挨饿状态则，需要资源占用比小 的优先
        compare = minShareRatio1.compareTo(minShareRatio2)
    } else {
        //都不挨饿，则比较权重比，比例低的优先
        compare = taskToWeightRatio1.compareTo(taskToWeightRatio2)
   }

  if (compare < 0) {
        true
   } else if (compare > 0) {
    false
   } else {
  //如果都一样，那么比较名字,按照字母顺序比较，不考虑长度，所以名字比较重要
    s1.name < s2.name
  }
 }
}
```

Fair 模式下 rootPool 管理用户定义的 Pool，在下一层才是 TaskSetManager。

![image-20210309200107534](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210309200107534.png)

## MR 与 Spark 的对比

- 代码抽象：MR 抽象层次低；Spark 通过 RDD 统一抽象
- 灵活性：只提供 Map 和 Reduce 两个编程抽象，不够灵活；Spark 通过 RDD 转换算子和行动算子，实现了许多基本操作
- 复杂程度：一个 Job 只有 Map 和Reduce 两个阶段，有些任务可能需要多个 Job 才能完成；Spark 的 Job 可以包含多个 RDD 转换操作，生成多个 Stage，一个 Stage 中也可以包含多个 Map 操作。
- 迭代能力：MR 多个 Job 直接需要中间结果落盘；Spark 使用内存存储数据，内存不足时才溢写到磁盘。
- 连续性：ReduceTask 需要等待所有 MapTask 完成后才能进行；Spark 分区相同的转换可以在一个 Task 中以流水线的形式执行，只有分区不同的转换需要 Shuffle 操作
- 时效性：MR 较慢只适合批处理；Spark 可以支持处理流数据

## RDD

RDD 弹性分布式数据集，Spark 底层的分布式存储数据结构，所有 Spark API 都是在 RDD 上操作的。

- 分布在多个节点上的不可变、已分区的集合对象
- 弹性：不保存数据，保存数据转换的元数据信息，即血缘关系 lineage 形成的 DAG。如果中间某个节点出错，只需通过 DAG 重新计算
- 可以控制存储级别（内存，磁盘等）进行重用
- 遇到行动算子才真正启动计算过程

### RDD 的依赖

RDD 的转换算子分为两种，区分两种依赖的依据就是是否发生了 shuffle 操作

- 窄依赖：RDD 各分区不依赖于父 RDD 的其他分区，能够独立计算结果
- 宽依赖：RDD 个分区依赖于父 RDD 的多个分区，如 reduceByKey 

Spark 的 Stage 阶段等于 RDD 的宽依赖数量 + 1

## 内存管理

Driver 端的内存管理较为简单，主要讨论的是 Executor 结点的内存管理。

### 堆内和堆外

Executor 作为一个 JVM 进程，内存管理是建立在 JVM 内存管理之上的。Spark 对 JVM 堆内空间进行了更详细的分配，以充分利用内存。同时引入堆外内存，进一步优化了内存的使用。

**堆内内存**

可以通过 `--executor-memory` 或者 `spark.executor.memory` 参数配置。

- 存储内存：缓存 RDD 和 广播变量数据占用的内存。
- 执行内存：任务在执行 shuffle 过程中占用的内存。
- 其他内存：Spark 内部的对象实例，用户程序中定义的对象实例等。

![image-20210318110009248](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318110009248.png)

**堆外内存**

堆外内存存储经过序列化的二进制数据。使用 JDK Unsafe API 可以直接操作系统堆外内存，减少了不必要的内存开销以及频繁的 GC，提升了处理性能。堆外内存可以被精确申请和释放，相比堆内内存来说降低了管理难度。

默认不启用，通过 `spark.memory.offHeap.enabled` 参数启用，`spark.memory.offHeap.size` 参数设定堆外空间大小。

堆外内存的划分除了没有 other 空间，其他和堆内一致。

![image-20210318110205546](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318110205546.png)

### 统一内存管理

以上堆外和堆内内存管理的图示代表了 Spark 的静态内存管理。但是如果设置不合理，很容易导致存储内存或者执行内存任意一方中有大量内存未使用，而另一方被占满的情况。因此 Spark 退出了新的内存管理机制。

Spark 1.6 后引入统一内存管理机制，存储内存和执行内存可以共享一块空间，可以动态占用对方的空闲区域。

**堆内**

![image-20210318111032178](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318111032178.png)

**堆外**

![image-20210318111110131](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318111110131.png)

动态占用机制

- 设定基本的内存区域 `spark.storage.storageFraction`
- 双方内存都不足时，溢写至磁盘。
- 若己方空间不足可以借用对方的空间
  - 执行内存被占用后，可以让存储内存占用的部分转存储到磁盘，要回可用空间。
  - 存储内存被占用后，无法让执行内存被淘汰，只能等待释放。

![image-20210318111529300](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318111529300.png)

## Shuffle

Shuffle 过程主要分为两个阶段：shuffle write (map)，shuffle read (reduce)

- shuffle write：按照分区将 map 产生的中间结果保存在内存中，内存不够时溢写到磁盘
- shuffle read：若未设置分区数则 reduce 时的分区数等于父 Stage 中最后一个 RDD 的分区数，从内存或磁盘中读取中间结果进行 reduce

### HashShuffleManager

Spark 1.6 以前，一直使用 HashShuffle。

#### 未优化

每个 ShuffleMapTask 会为每个 ReduceTask 创建一个缓冲区，将数据经过 Partitioner 划分到不同的缓冲区，缓冲区达到阈值后溢写到磁盘上。所有 ShuffleMapTask 执行完后，ReduceTask 将对应的数据文件拉取过来，然后执行聚合逻辑。

![image-20210318123718819](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318123718819.png)

缺点：每个 ShuffleMapTask 都可能会产生对应 ReduceTask 数量的文件，如果节点数较多且每个切点上的 ShuffleMapTask 数量也很多，会导致大量的小文件。ReduceTask 在拉取数据时就会有更多网络和磁盘 I/O 开销。

#### 优化

Executor 上每个核才会产生对应 ReduceTask 数量的文件，这样在同一个核内的 ShuffleMapTask 可以复用已经创建好的缓冲区和对应文件。

![image-20210318124714449](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318124714449.png)

### SortShuffleManager

运行机制主要有两种，一种为普通运行机制，另一种为 bypass 机制。当 shuffle read task 数量小于 `spark.shuffle.sort.bypassMergeThreshold` 参数时(默认200)，启用 bypass

#### 普通机制

根据不同的 shuffle 算子选择不同的数据结构

- reduceByKey 等聚合类：Map
- join 等普通类：Array

当内存中的数据超出某个阈值后，溢写到磁盘上。溢写之前，先根据 key 对内存中的数据进行**排序**，排序过后会分批写入磁盘。

一个 ShuffleMapTask 可能会发生多次磁盘溢写，因此会产生多个临时文件。最后会将所有的临时磁盘文件都进行合并，此外还会单独写一个**索引文件**，用来告诉下游 ReduceTask 各分区数据在文件中的起始位置和结束位置。

![image-20210318133644197](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318133644197.png)

#### bypass

bypass 机制需要 shuffle 算子不能是排序类的算子，它在往内存缓冲区中写数据时按照 key 的 hash 值写入对应缓冲区，并溢写到对应的文件。

![image-20210318134024173](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318134024173.png)

相比普通机制的好处在于不会执行排序操作。

### Hadoop 和 Spark 的 shuffle 对比

- shuffle 过程排序次数不同：MR 总共发生 3 次排序，第一次排序发生在 map 阶段，根据 key 对溢写到磁盘上数据进行排序；第二次排序发生在 map 阶段，通过 combiner 对溢出的小文件进行归并排序；第三次排序 reduce 阶段将不同 map 端的数据拉取同一个分区后进行归并排序。Spark 只有在使用 SortShuffleManager 且为普通模式下才会发生一次排序。
- shuffle 划分逻辑不同：MR 每个Job 就需要 shuffle 一次。而 Spark 只有在产生宽依赖的时候才会进行划分。
- shuffle fetch 后数据存放位置：MR 存储在磁盘上。Spark 首先会将数据存储在 reduce 端的内存缓冲区，当内存使用到达一定阈值时溢写到磁盘上。



# SparkSQL

SparkSQL 是使用 Spark 处理结构化数据的一个模块，将Spark SQL转换成RDD，然后提交到集群中去运行。

## DataFrame 和 DataSet

SparkSQL 提供了一个编程抽象叫做DataFrame并且作为分布式SQL查询引擎的作用。

相比于Spark RDD API，Spark SQL包含了对结构化数据和在其上运算的更多信息，Spark SQL使用这些信息进行了额外的优化，使对结构化数据的操作更加高效和方便。

有多种方式去使用Spark SQL，包括SQL、DataFrames API和Datasets API。但无论是哪种API或者是编程语言，它们都是基于同样的**执行引擎**，因此你可以在不同的API之间随意切换，它们各有各的特点，看你喜欢那种风格。

### DataFrame

DataFrame是一种以RDD为基础的分布式数据集，类似于传统数据库的二维表格，DataFrame带有Schema元信息，即DataFrame所表示的二维表数据集的每一列都带有名称和类型，但底层做了更多的优化。DataFrame可以从很多数据源构建，比如：已经存在的RDD、结构化文件、外部数据库、Hive表。

![image-20210904153543440](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210904153543440.png)

- RDD是分布式的对象集合，无法知道对象的详细信息；DataFrame 是分布式 Row 对象的集合，有对象的原信息（列名和列类型）

### DataSet

- 与 DataFrame 相比拥有完全相同的成员函数，只是每行的数据类型不同
- DataFrame 每一行的类型是Row，可以看做是DataSet[Row]





# 面试题

## 数据倾斜问题

1. 调整并行度：shuffle 默认使用 HashPartitioner ，如果并行度设置不合理，可能某个节点分配到了大量的 key。

   ![Image](C:\Users\aasus\AppData\Local\Temp\Image.png)

2. 自定义 Partitioner

3. 通过Spark的Broadcast机制，将Reduce侧Join转化为Map侧Join，避免Shuffle从而完全消除Shuffle带来的数据倾斜

   ![image-20210904151528280](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210904151528280.png)

4. 为倾斜的 key 增加随机前/后缀

## Spark 速度更快的原因

- 消除了冗余的磁盘读写：MR 每次 shuffle 操作必须写到磁盘，而 Spark 的 shuffle 首先会保存在内存中，内存不够才会溢写到磁盘上， 且可以将 Job 中可以复用的计算逻辑缓存到内存，以便迭代使用
- 消除冗余的 MR 阶段：MR 成对出现，一个计算任务可能需要多个 MR 才能实现，也意味着多个 shuffle，多次磁盘 I/O 。而 Spark 基于 RDD 提供了丰富的算子，可以尽量避免 shuffle

## Spark 作业提交流程

### Standalone

1. `spark-submit` 提交代码，执行 `new SparkContext()`，在 SparkContext 里构造 `DAGScheduler` 和 `TaskScheduler`。
2. TaskScheduler 会通过后台的一个进程，连接 Master，向 Master 注册 Application。
3. Master 接收到 Application 请求后，会使用相应的资源调度算法，在 Worker 上为这个 Application 启动多个 Executer。
4. Executor 启动后，会自己反向注册到 TaskScheduler 中。 所有 Executor 都注册到 Driver 上之后，SparkContext 结束初始化，接下来往下执行我们自己的代码。
5. 每执行到一个 Action，就会创建一个 Job。Job 会提交给 DAGScheduler。
6. DAGScheduler 会将 Job 划分为多个 stage，然后每个 stage 创建一个 TaskSet。
7. TaskScheduler 会把每一个 TaskSet 里的 Task，提交到 Executor 上执行。
8. Executor 上有线程池，每接收到一个 Task，就用 TaskRunner 封装，然后从线程池里取出一个线程执行这个 task。(TaskRunner 将我们编写的代码，拷贝，反序列化，执行 Task，每个 Task 执行 RDD 里的一个 partition)

