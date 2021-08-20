# Redis

## 简介

Redis 是一个灵活的高性能 key-value 数据库，被广泛应用于缓存。另外，Redis 除了做缓存之外，Redis 也经常用来做**分布式锁**，甚至是**消息队列**。

Redis 提供了多种数据类型来支持不同的业务场景。Redis 还支持事务 、持久化、Lua 脚本、多种集群方案。

Redis 主要由有两个程序组成：

- Redis 客户端 redis-cli
- Redis 服务器 redis-server

客户端、服务器可以位于同一台计算机或两台不同的计算机中。

### 特点

Redis 比其他 key-value 缓存产品有以下特点：

- 高性能：Redis 将所有数据都存储在内存中，可以在入门级 Linux 机器中每秒写（SET）11 万次，读（GET）8.1 万次。Redis 支持 Pipelining 命令，可一次发送多条命令来提高吞吐率，减少通信延迟。
- 持久化：Redis 支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载到内存使用。Redis 提供了两种持久化的方式，分别是 RDB（Redis DataBase）和 AOF（Append Only File）。
- 原子操作：Redis 不仅支持简单的 key-value 类型的数据，同时还提供 list，set，zset，hash 等数据结构的存储。而且对不同数据类型的处理都是原子性的。
- 主从复制：Redis 支持简单快速的主从复制，配置文件中只要一行来设置它。

### 与 Memcached 对比

共同点：

- 都是基于内存的数据库，一般应用于缓存
- 都有过期策略
- 两者性能都很高

区别：

- Redis 支持更丰富复杂的数据类型，即支持更多应用场景。而 Memcached 只支持最简单的 k/v 数据类型。
- Redis 支持数据的**持久化**。Memcached 把数据全部存在内存。
- Redis 有灾难恢复机制，因为可以把缓存的数据持久化在磁盘。
- Redis 在内存不够时可以把不用的数据落盘。Memcached 内存耗尽后会报异常。
- Redis 原生支持集群模式。Memcached 没有原生的集群模式。
- Redis 使用单线程的多路 IO 复用模型（Redis 6.0 后引入多线程 IO）。Memcached 是多线程，非阻塞 IO 复用的网络模型。
- Redis 支持发布订阅模型、Lua 脚本、事务等功能，而 Memcached 不支持。并且，Redis 支持更多的编程语言。
- Memcached过期数据的删除策略只用了惰性删除，而 Redis 同时使用了惰性删除与定期删除。

### 使用场景

- 计数器：Redis 作为内存数据库读写性能很高，很适合存储频繁读写的计数量。
- 缓存：将热点数据存放至内存中，设置内存的最大使用量和淘汰策略保证缓存的命中率。
- 消息队列：使用 list 数据类型，通过 lpush 和 rpop 写入和读取消息。
- 分布式锁：SETNX 命令实现分布式锁，或者 RedLock 分布式锁实现。
- 其他：set 实现交集、并集，从而实现共同好友等功能； zset 可以实现有序性操作，从而实现排行榜等功能。

## 数据结构和对象

### 数据结构

#### SDS

Redis 对 string 的底层实现为 **SDS**。

```c
struct sdshdr{
	int len;
	int free; // 记录 buf 数组中为使用的字节数
	char buf[];
}
```

遵循了 C 语言的以 '\0' 为字符串结尾的惯例，以便重用 C 字符串函数库。

SDS 在动态修改字符串时（增长，缩短）实现了**空间预分配**和**惰性释放**的优化策略，满足 Redis 作为内存数据库快速响应且数据频繁修改的需求。

C 语言字符串和 SDS 之间区别在于：

- strelen 复杂度：C 语言为 O(n)，SDS 为O(1)
- 是否安全：C 语言修改字符串长度时不安全，可能造成缓存区溢出。SDS 首先检查 free 是否满足扩充要求，不满足则会先扩充 SDS 的空间。
- 重分配内存：C 语言修改字符串 N 次必定执行 N 次内存重分配。SDS 的内存分配策略为
  - 如果 SDS 的大小< 10M ，则分配 free 空间大小等于 len，总长度即 2*len+1。
  - SDS 大小> 10M，则分配 1MB 的free。
- 保存数据类型：C 语言只能保存文本数据。SDS 可以保存文本或者二进制数据。

#### 链表

链表的底层实现为**双向链表**，结构如下，并有以下的特点

![image-20210401204632318](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210401204632318.png)

- 双端无环队列：获取链头和链尾结点 O(1)。
- 维护 len ：获取长度为 O(1)。
- 多态：使用 void* 指针保存 dup/free/match ，可以使 list 实现多种类型。

#### 字典

字典使用**哈希表**作为底层实现，哈希表中有多个哈希结点

![image-20210321151036339](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210321151036339.png)

- type 和 privdata 属性针对不同类型的键值对，为创建**多态**字典而设置。
- ht：包含两个 dictht，只有在 rehash 时才会使用 ht[1]。
- rehashidx：记录当前 rehash 的进度，没在 rehash 时为 -1 。



发送**冲突**时采用头插法将新节点插入到对应下标的链表头，随着表中元素不断增多，为了使负载因子保持在一个合理的范围(服务器正在执行 BGSAVE/BGREWRITEAOF 命令时 loadFactor >=5，平时 loadFactor >=1)，会执行 rehash。

1. 给 ht[1] 分配内存空间，如果是扩充则为大于等于2倍 ht[0].used 的最小2的次幂，减小为一倍。
2. 重新计算所有元素在 ht[1] 的下标并插入到指定位置。
3. 交换并释放 ht[1]

渐进式 rehash，避免在 rehash 过程中服务不可用，通过 rehashidx 变量表示 rehash 的进度。

- rehashidx=0: rehash 正式开始
- rehashidx=0~ht[0].used: 代表 ht[0] 中 rehashidx 之前的哈希表节点都已经复制完毕。
- rehashidx=-1: rehash 完成。

渐进式 rehash 的过程中如果有插入操作，都是在 ht[1] 上进行，避免数据不一致。

#### 跳跃表

跳表的数据结构定义如下:

```c
#define ZSKIPLIST_MAXLEVEL 32
#define ZSKIPLIST_P 0.25

typedef struct zskiplistNode {
    robj *obj;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span; //当前指针跨越了多少节点，用于计算排名
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

![image-20210321135526261](..\pic\image-20210321135526261.png)

每个节点都一个 level 数组，表示了该节点存在于哪几层。每次生成一个节点时，会随机决定该节点的层数，为了保证一个良好的查找性能，保证

- 每个节点都有第一层指针
- 每个节点如果在 i 层有指针，则在 i+1 层有指针的概率为 p
- 节点最大层数不允许超过 MaxLevel

redis 将相关参数设置为：

```properties
p = 1/4
MaxLevel = 32
```

层数越高，概率越低，满足幂次定律，节点层数为一层的概率为 (1-p)，二层为 p(1-p)，....，因此平均每个节点的层数为：

![image-20210401213405040](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210401213405040.png)

当 p=1/4 时，每个节点平均只有 1.33 个指针，空间开销 O(n)

#### 整数集合

整数集合 (intset) 用于保存整数值的集合，可以保存 `int16_t` 、 `int32_t` 或者 `int64_t` 的整数值，并且没有重复。

其数据结构如下：

```c
typedef struct intset {

    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```

![image-20210401214001630](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210401214001630.png)

- encoding: 表示了 contents 数组的类型
- contents: 整数据集合的底层实现，从小到大排序且无重复。

每次向 intset 插入新元素时，如果新元素的类型大于原有元素的类型，需要对整个数组进行**升级操作**，即把所有元素的类型都升级为最高类型。引起升级的元素要么大于原有所有元素要么小于，因此插入的位置要么为开头要么为末尾。升级操作可以保证内存被充分利用，不会引起**内存浪费**。

intset 不支持降级操作。

#### 压缩列表

Redis 为了节约内存开发了 ziplist 由一系列特殊编码的连续内存块组成的顺序型数据结构。每个节点可以保存为一个字节数组或一个整数值。

![image-20210401215008052](D:\Typora\后端面试知识点\image-20210401215008052.png)

| 属性      | 类型       | 长度     | 用途                                                         |
| :-------- | :--------- | :------- | :----------------------------------------------------------- |
| `zlbytes` | `uint32_t` | `4` 字节 | 记录整个压缩列表占用的内存字节数：在对压缩列表进行内存重分配， 或者计算 `zlend` 的位置时使用。 |
| `zltail`  | `uint32_t` | `4` 字节 | 记录压缩列表表尾节点距离压缩列表的起始地址有多少字节： 通过这个偏移量，程序无须遍历整个压缩列表就可以确定表尾节点的地址。 |
| `zllen`   | `uint16_t` | `2` 字节 | 记录了压缩列表包含的节点数量： 当这个属性的值小于 `UINT16_MAX` （`65535`）时， 这个属性的值就是压缩列表包含节点的数量； 当这个值等于 `UINT16_MAX` 时， 节点的真实数量需要遍历整个压缩列表才能计算得出。 |
| `entryX`  | 列表节点   | 不定     | 压缩列表包含的各个节点，节点的长度由节点保存的内容决定。     |
| `zlend`   | `uint8_t`  | `1` 字节 | 特殊值 `0xFF` （十进制 `255` ），用于标记压缩列表的末端。    |

每个 ziplist 节点的结构如下：

![image-20210401215235662](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210401215235662.png)

- previous_entry_length: 记录了该节点前一个节点的长度
- encoding: 节点保存数据的类型和长度
- content: 保存节点的值，可以为一个字节数组或者整数。

![image-20210401215709502](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210401215709502.png)

### 对象

####  对象类型和编码

```c
typedef struct redisObject{
	unsigned type:4;
	unsigned encoding:4;
	void *ptr;
	// ...
}robj;
```

![image-20210401220511411](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210401220511411.png)

- type: 表示对象的类型，可以为 5 中基本类型对象中的一种。
- encoding: 表示对象使用什么数据结构作为对象的底层实现。
- ptr: 指向底层数据结构的指针。

| 类型           | 编码                        | 对象                                                 |
| :------------- | :-------------------------- | :--------------------------------------------------- |
| `REDIS_STRING` | `REDIS_ENCODING_INT`        | 使用整数值实现的字符串对象。                         |
| `REDIS_STRING` | `REDIS_ENCODING_EMBSTR`     | 使用 `embstr` 编码的简单动态字符串实现的字符串对象。 |
| `REDIS_STRING` | `REDIS_ENCODING_RAW`        | 使用简单动态字符串实现的字符串对象。                 |
| `REDIS_LIST`   | `REDIS_ENCODING_ZIPLIST`    | 使用压缩列表实现的列表对象。                         |
| `REDIS_LIST`   | `REDIS_ENCODING_LINKEDLIST` | 使用双端链表实现的列表对象。                         |
| `REDIS_HASH`   | `REDIS_ENCODING_ZIPLIST`    | 使用压缩列表实现的哈希对象。                         |
| `REDIS_HASH`   | `REDIS_ENCODING_HT`         | 使用字典实现的哈希对象。                             |
| `REDIS_SET`    | `REDIS_ENCODING_INTSET`     | 使用整数集合实现的集合对象。                         |
| `REDIS_SET`    | `REDIS_ENCODING_HT`         | 使用字典实现的集合对象。                             |
| `REDIS_ZSET`   | `REDIS_ENCODING_ZIPLIST`    | 使用压缩列表实现的有序集合对象。                     |
| `REDIS_ZSET`   | `REDIS_ENCODING_SKIPLIST`   | 使用跳跃表和字典实现的有序集合对象。                 |



Redis 支持五种对象。

- 字符串（string）
- 哈希（hash）
- 列表（list）
- 集合（set）
- 有序集合（sorted set）

#### string

string 数据结构是简单的 key-value 类型。虽然 Redis 是用 C 语言写的，但是 Redis 并没有使用 C 的字符串表示，而是自己构建了一种 **简单动态字符串**（simple dynamic string，**SDS**）。相比于 C 的原生字符串，Redis 的 SDS 不光可以保存文本数据还可以保存二进制数据，并且**获取字符串长度复杂度为 O(1)**（C 字符串为 O(N)）,除此之外，Redis 的 SDS API 是安全的，不会造成缓冲区溢出。

常用命令：`set,get,strlen,exists,dect,incr,setex` 等等。

一般常用在需要计数的场景，比如用户的访问次数、热点文章的点赞转发数量等等。

#### hash

hash 类似于 JDK1.8 前的 HashMap，内部实现也差不多(数组 + 链表)。不过，Redis 的 hash 做了更多优化。另外，hash 是一个 string 类型的 field 和 value 的映射表，**特别适合用于存储对象**，后续操作的时候，你可以直接仅仅修改这个对象中的某个字段的值。 比如我们可以 hash 数据结构来存储用户信息，商品信息等等。

常用命令：`hset,hmset,hexists,hget,hgetall,hkeys,hvals` 等。

应用于系统中对象数据的存储。

hash 的底层实现为 **ziplist**/**dict**。

当 hash 表只包含少量键值对，并且每个键值对的键和值要么是小整数要么是短字符串，此时会使用 ziplist 实现 hash 表。当满足以下某个条件时，ziplist 转换为 hashtable。

```properties
hash-max-ziplist-entries 512 # 数据个数超过 512
hash-max-ziplist-value 64 # 插入的任意一个数据长度超过 64 字节
```

#### list

**list** 即是 **链表**。链表是一种非常常见的数据结构，特点是易于数据元素的插入和删除并且且可以灵活调整链表长度，但是链表的随机访问困难。许多高级编程语言都内置了链表的实现比如 Java 中的 **LinkedList**，但是 C 语言并没有实现链表，所以 Redis 实现了自己的链表数据结构。Redis 的 list 的实现为一个 **双向链表**，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。

常用命令: `rpush,lpop,lpush,rpop,lrange、llen` 等。

应用于发布与订阅或者说消息队列、慢查询。

list 底层实现为 **双向链表** + **ziplist**。list 元素较少且每个元素大小较小时使用 ziplist，当满足以下某一个条件时，ziplist 转换为链表。

```properties
list-max-ziplist-entries 512 # 数据个数超过 512
list-max-ziplist-value 64 # 插入的任意一个数据长度超过 64
```

#### set

set 类似于 Java 中的 `HashSet` 。Redis 中的 set 类型是一种无序集合，集合中的元素没有先后顺序。当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并且 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的。可以基于 set 轻易实现交集、并集、差集的操作。比如：你可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。Redis 可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。这个过程也就是求交集的过程。

常用命令：`sadd,spop,smembers,sismember,scard,sinterstore,sunion` 等。

应用场景：需要存放的数据不能重复以及需要获取多个数据源交集和并集等场景。

set 的底层实现为 **dict**/**intset**。当集合只包含整数值元素时，并且集合元素数量不多时，Redis 就会采用 intset 作为 set 的底层实现。

```c
typedef struct intset{
	uint32_t encoding; // 编码方式
	uint32_t length; 
	int8_t contents[]; // 保存元素的数组
}intset;
```

intset 中插入新的元素类型大于原有元素类型时，需要进行升级。将 contents 数据中所有元素升级为新插入元素的类型。



#### sorted set

和 set 相比，sorted set 增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，还可以通过 score 的范围来获取元素的列表。有点像是 Java 中 HashMap 和 TreeSet 的结合体。

常用命令：`zadd,zcard,zscore,zrange,zrevrange,zrem` 等。

 需要对数据根据某个权重进行排序的场景。比如在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息。

Redis 中 sorted set 每插入一个元素都需要指定数据本身和对应的一个 score。根据 score 进行排序，当 score 相同时，按照数据本身的字典序排序。

sorted set 的一些命令在基础的 skiplist 中是不支持的：

- zrevrank: 根据数据查询其排名。
- zscroe: 根据数据查询其 score。
- zrevrange: 根据一个排名范围，查询在这个范围内的数据。

为了满足以上的命令，Redis 对skiplist 做了扩展，使得其能够根据排名快速查询到数据，或者根据 score 查到数据后，能够很容易获取排名，因此实际上，Redis sorted set 的底层实现为：

- 数据较少时由一个 **ziplist** 实现
- 数据多的时候由 zet 实现：**一个 dict + 一个 skiplist 实现**。

![image-20210401220804832](..\pic\image-20210401220804832.png)

对于 **ziplist**，它是由很多数据项组成的一大块连续内存。查找只能顺序查找，优点在于节省内存。当插入数据时满足以下两个条件之一，sorted set 就会转变为 zset。

```properties
zset-max-ziplist-entries 128 # 数据个数超过 128
zset-max-ziplist-value 64 # 插入的任意一个数据长度超过 64
```

Redis 使用 skiplist 而不用平衡树的原因在于

- 内存占用更少，平衡树每个节点上的指针数量一般大于 skiplist（在 p=1/4 时平均每个节点有 1.33 个指针）。
- 范围查询效果和平衡树一样好，zrange，zrevrange 的操作设计遍历 skiplist，这样的操作更够使缓存局部性发挥作用。
- 实现比较简单，查询操作平均 O(logn)，插入操作 O(logn)。

#### 内存管理

Redis 通过**引用计数**实现内存回收机制，具体为 redisObject 的 refcount 属性记录。

通过**共享对象**减少内存占用，初始化服务器时会创建 0-9999 一万个字符串对象用于共享。Redis 只对整数字符串对象进行共享。



## 单线程模型

Redis 基于 Reactor 模式设计开发了自己的一套高效的事件处理模型（Netty 的线程模型也是基于 Reactor 模式），这套事件处理模型对应的是 Redis 中的文件事件处理器（file event handler）。由于文件事件处理器（file event handler）是单线程方式运行的，所以我们一般都说 Redis 是单线程模型。

### 文件事件处理器

Redis 通过 **IO 多路复用技术**来监听来自客户端的大量连接（或者说是监听多个 Socket），它会将感兴趣的事件及类型(读、写）注册到内核中并监听每个事件是否发生。**I/O 多路复用技术的使用让 Redis 不需要额外创建多余的线程来监听客户端的大量连接，降低了资源的消耗**（和 NIO 中的 `Selector` 组件很像）。

当多个 socket 准备好执行连接应答、写入、读取、关闭等操作时，I/O 多路复用程序将就绪的 socket 放入一个队列里，通过队列同步地发送给文件事件分派器。文件事件分派器根据 socket 事件类型调用相应的事件处理器，完成客户端的请求。优先会处理读 socket。

文件事件处理器包含 4 个部分：

- 多个 socket （客户端连接）
- IO 多路复用程序 
- 文件事件分派器（将 Socket 关联到相应的事件处理器）
- 事件处理器

![image-20210225183106655](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210225183106655.png)

### 时间事件处理器

Redis 有一些操作需要在给定的时间执行，如**定时事件**，**周期性事件**。典型的时间时间为 serverCron ，用于定期对 Redis 服务器资源和状态进行检查和调整，如删除过期键，尝试进行持久化操作等等。

Redis 将所有时间事件存放在一个无序链表中，通过遍历整个链表找到已到达的时间事件，并调用相应的事件处理器。

### 事件调度与执行

Redis 服务器需要不断的监听 socket 来得到就绪的文件事件从而进行处理，但是**不能一直阻塞**在那，否则一些时间事件无法再规定的时间内执行，因此监听时间应该根据**距离现在最近的时间事件**来决定。

```python
def aeProcessEvents():
    # 获取到达时间离当前时间最接近的时间事件
    time_event = aeSearchNearestTimer()
    # 计算最接近的时间事件距离到达还有多少毫秒
    remaind_ms = time_event.when - unix_ts_now()
    # 如果事件已到达，那么 remaind_ms 的值可能为负数，将它设为 0
    if remaind_ms < 0:
        remaind_ms = 0
    # 根据 remaind_ms 的值，创建 timeval
    timeval = create_timeval_with_ms(remaind_ms)
    # 阻塞并等待文件事件产生，最大阻塞时间由传入的 timeval 决定
    aeApiPoll(timeval)
    # 处理所有已产生的文件事件
    procesFileEvents()
    # 处理所有已到达的时间事件
    processTimeEvents()
```

将 aeProcessEvents 函数置于一个循环中，基本构成了 Redis 服务器的主函数。

```python
def main():
    # 初始化服务器
    init_server()
    # 一直处理事件，直到服务器关闭为止
    while server_is_not_shutdown():
        aeProcessEvents()
    # 服务器关闭，执行清理操作
    clean_server()
```

![image-20210315155513212](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210315155513212.png)

### 为何不使用多线程

Redis 4.0 开始就加入了对多线程的支持，但是主要是针对一些大键值对的删除操作的命令，使用这些命令就会使用主处理之外的其他线程来“异步处理”。总的来说 6.0 之前基本都是单线程。

原因在于：

- 单线程实现简单且维护也容易
- Redis 的性能瓶颈不在CPU，主要在内存和网络。
- 多线程可能会产生死锁等并发问题，还带来了多线程切换的开销，影响性能。

Redis 6.0 后引入多线程的原因是为了**提高网络 IO 读写性能**。虽然，Redis6.0 引入了多线程，但是 Redis 的多线程只是在网络数据的读写这类耗时操作上使用了， 执行命令仍然是单线程顺序执行。因此不需要担心线程安全问题。

## 缓存

### 过期时间

一般情况下，我们设置保存的缓存数据的时候都会设置一个过期时间，以免大量缓存占用内存导致 OOM。此外在一些业务场景下，如短信验证码只在一分钟内有效。

可以通过 `setex` 设置过期时间，也可以通过 `expire`。 

### 过期字典

Redis 通过一个叫做过期字典（可以看作是hash表）来保存数据过期的时间。过期字典的键指向 Redis 数据库中的某个 key (键)，过期字典的值是一个 long long 类型的整数，这个整数保存了 key 所指向的数据库键的过期时间（毫秒精度的 UNIX 时间戳）。

![image-20210225201903587](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210225201903587.png)

过期字典存储在 redisDb 这个结构体中

```c
typedef struct redisDb {
    ...
    
    dict *dict;     //数据库键空间,保存着数据库中所有键值对
    dict *expires   // 过期字典,保存着键的过期时间
    ...
} redisDb;

```

### 缓存更新策略

1.先删缓存，再更新数据库

并发读写场景下可能会产生缓存脏数据，并且脏数据会一直等到缓存过期或发起新的更新操作才会结束。

![image-20210316105301446](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210316105301446.png)

2.先更新数据库，在删缓存

目前业界最常用的方案，同样不够完美，但问题发生的概率很小。并发读写场景下仍然会引发缓存脏数据。

![image-20210316105917342](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210316105917342.png)

但是出现这个 case 的概率很低，需要满足 4 个条件

- 读操作读缓存失效
- 同时存在并发写操作
- 写操作比读操作块
- 读操作早于写操作进入数据库，晚于写操作更新缓存

因此说发生问题的概率很低，就算发生了，也有缓存过期时间兜底。缓存本身就是在可用性和一致性之间的一个 tradeoff。

3.先更新数据库，再更新缓存

理论上来说这种方式比先更新数据库再删缓存有更高的读性能，因为它在更新数据库的时候就更新了缓存。同样是这个原因，写性能也比较低。在并发更新的情况下，可能出现缓存脏数据。

![image-20210316111036713](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210316111036713.png)

### 过期删除策略

- 惰性删除：只在获取该 key 时在对数据进行过期检查。这样对减少了 CPU 的计算负担，但是可能会造成太多过期的 key 没有被删除。
- 定期删除：每个一段时间抽取一批 key 执行删除过期 key 操作 。Redis 底层会通过限制删除操作执行的时长和频率来减少删除操作对 CPU 时间的影响。

定期删除对内存更加友好，惰性删除对 CPU 更加友好。两者各有千秋，所以 Redis 采用的是 **定期删除+惰性/懒汉式删除** 。但是，仅仅通过给 key 设置过期时间还是有问题的。因为还是可能存在定期删除和惰性删除漏掉了很多过期 key 的情况。这样就导致大量过期 key 堆积在内存里，然后就Out of memory了。需要内存淘汰策略解决。

### 内存淘汰策略

Redis 提供 6 种数据淘汰策略：

1. **volatile-lru（least recently used）**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
3. **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. **allkeys-lru（least recently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-eviction**：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧！

4.0 版本后增加以下两种：

1. **volatile-lfu（least frequently used）**：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
2. **allkeys-lfu（least frequently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

作为内存数据库，出于对性能和内存消耗的考虑， Redis 的淘汰算法实际实现上并非针对所有 key，而是抽样一小部分并从中选择出被淘汰的 key。

### 缓存穿透

缓存穿透说简单点就是大量请求的 key 根本不存在于缓存中，导致请求直接到了数据库上，根本没有经过缓存这一层。举个例子：某个黑客故意制造我们缓存中不存在的 key 发起大量请求，导致大量请求落到数据库。

解决方法：

1.缓存无效 key

如果缓存和数据库都查不到某个 key 的数据就写一个到 Redis 中去并设置过期时间，具体命令如下： `SET key value EX 10086` 。这种方式可以解决请求的 key 变化不频繁的情况，如果黑客恶意攻击，每次构建不同的请求 key，会导致 Redis 中缓存大量无效的 key 。很明显，这种方案并不能从根本上解决此问题。如果非要用这种方式来解决穿透问题的话，尽量将无效的 key 的过期时间设置短一点比如 1 分钟。

2.布隆过滤器

布隆过滤器是一个非常神奇的数据结构，通过它我们可以非常方便地判断一个给定数据是否存在于海量数据中。

把所有可能存在的请求的值都存放在布隆过滤器中，当用户请求过来，先判断用户发来的请求的值是否存在于布隆过滤器中。不存在的话，直接返回请求参数错误信息给客户端，存在的话才会走下面的流程。

但是，需要注意的是布隆过滤器可能会存在**误判**的情况。总结来说就是： **布隆过滤器说某个元素存在，小概率会误判。布隆过滤器说某个元素不在，那么这个元素一定不在。**

元素加入布隆过滤器：

- 根据哈希函数计算元素对应哈希值
- 根据哈希值把数组对应下标置1

判断元素是否在布隆过滤器：

- 计算哈希值
- 根据哈希值判断数组所有对应位置是否都为 1，如果都为 1 则说明值存在与布隆过滤器中，否则不存在。

不同的字符串有可能得到相同的哈希值，因此可能出现误判。

3.设置参数校验

对于一些恶意发送不合法 key 的请求，可以通过参数校验将一些不合法的请求直接拒绝。

### 缓存雪崩

缓存雪崩描述的就是这样一个简单的场景：**缓存在同一时间大面积的失效，后面的请求都直接落到了数据库上，造成数据库短时间内承受大量请求。** 

缓存失效的原因可能有

- Redis 服务不可用，宕机
- 热点缓存过期等

相应解决方法

- 针对服务不可用：采用 Redis 集群，避免单点故障；限流，避免处理大量请求
- 针对热点失效：调整设置缓存失效时间，比如设置随机失效时间。

### 缓存击穿

和缓存雪崩的场景有点类似，不同之处在于缓存击穿指的是一个热点 key 突然失效导致大量的请求直接发送到数据库。

解决方案

- 调整热点 key 的过期时间
- 互斥锁，在 key 失效时先加锁，然后从数据库中查询，更新缓存，再释放锁，其他请求阻塞。

## 持久化机制

很多时候我们需要持久化数据也就是将内存中的数据写入到硬盘里面，大部分原因是为了之后重用数据（比如重启机器、机器故障之后恢复数据），或者是为了防止系统故障而将数据备份到一个远程位置。

Redis 不同于 Memcached 的很重要一点就是，Redis 支持持久化，而且支持两种不同的持久化操作。**Redis 的一种持久化方式叫快照（snapshotting，RDB），另一种方式是只追加文件（append-only file, AOF）**。

### RDB

Redis 可以通过创建快照来获得存储在内存里面的数据在某个时间点上的副本。Redis 创建快照之后，可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（Redis 主从结构，主要用来提高 Redis 性能），还可以将快照留在原地以便重启服务器的时候使用。

快照持久化是 Redis 默认采用的持久化方式，在 Redis.conf 配置文件中默认有此下配置：

```bash
save 900 1           #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 300 10          #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 60 10000        #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
```

两种命令可以用于创建 RDB 文件

- SAVE: 阻塞
- BGSAVE: 子进程中创建 RDB，非阻塞。执行 BGSAVE 时禁止执行 SAVE/BGSAVE，避免产生竞争条件。也不能同时执行 BGASAVE 和 BGREWRITEAOF ，因为它们都涉及大量的磁盘写入操作，同时执行影响服务器性能。

### AOF

与快照持久化相比，AOF 持久化 的**实时性**更好，因此已成为主流的持久化方案。

开启 AOF 持久化后每执行一条会更改 Redis 中的数据的命令，Redis 就会将该命令写入 aof_buffer 缓冲区的末尾，等待后续写入磁盘。AOF 文件的保存位置和 RDB 文件的位置相同，都是通过 dir 参数设置的，默认的文件名是 appendonly.aof。

Redis 在结束每个时间循环之前，都会考虑是否将 aof_buffer 写入和保存到 AOF 文件中。

AOF 的三种方式为：

- appendfsync always  每次有数据修改发生时都会写入AOF文件,这样会严重降低Redis的速度
- appendfsync everysec  每秒钟同步一次，显示地将多个写命令同步到硬盘
- appendfsync no        让操作系统决定何时进行同步

AOF 文件载入时需要创建一个伪客户端（Redis 只能在客户端中执行命令），不断执行 AOF 中的命令，还原出服务器的状态。

**AOF 重写**是为了解决 AOF 文件体积膨胀的问题，实现原理为重新读取 Redis 服务器键空间的状态，用一条命令去记录每个键值对，代替之前可能存在的多条命令，从而减少 AOF 文件的体积。

重写通过 BGREWRITEAOF 开启子进程重写，避免阻塞主进程。并且为了在不加锁的情况下保证数据的安全性，子进程持有服务器状态的副本，因此可能会出现数据不一致的问题。

为了解决这个问题，Redis 设置了一个 **AOF 重写缓冲区**，在客户端执行命令时，如果重写正在执行，Redis 会把命令同时追加到 AOF 缓冲区和 AOF 重写缓冲区，在重写完成后，会把重写缓冲区中的内容写入新的 AOF 文件中，这样新的 AOF 文件就和服务器状态保持一致，然后替换掉旧的 AOF 文件。

## 事务

Redis 可以通过 **MULTI，EXEC，DISCARD 和 WATCH** 等命令来实现事务(transaction)功能。

```bash
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```

Redis 事务不支持 roll back，因此不满足原子性，且不满足持久性。

## 复制

通过使用 slaveof host port 命令来让一个服务器成为另一个服务器的从服务器。

一个从服务器只能有一个主服务器，并且**不支持主主复制**。

### 连接过程

1. Master 创建 RDB 文件，发送给 Slave，并在发送期间使用缓冲区记录执行的写命令。RDB 快照文件发送完毕后，发送缓冲区中的写命令。
2. Slave 丢弃所有旧数据，载入主服务器发送的 RDB 文件，之后接受写命令。
3. 主服务器每执行一次写命令，就向从服务器发送相同的写命令。

### 主从连

随着负载不断上升，主服务器可能无法很快地更新所有从服务器，或者重新连接和重新同步从服务器将导致系统超载。为了解决这个问题，可以创建一个中间层来分担主服务器的复制工作。中间层的服务器是最上层服务器的从服务器，又是最下层服务器的主服务器。

![image-20210315160059014](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210315160059014.png)

## 分片

分片是将数据划分多个部分的方法，可以将原来一台 Redis 中的数据存储到多台机器里面。

假设有 4 个 Redis 实例 R0，R1，R2，R3，还有很多表示用户的键 user:1，user:2，... ，有不同的方式来选择一个指定的键存储在哪个实例中。

- 最简单的方式是范围分片，例如用户 id 从 0~1000 的存储到实例 R0 中，用户 id 从 1001~2000 的存储到实例 R1 中，等等。但是这样需要维护一张映射范围表，维护操作代价很高。
- 还有一种方式是哈希分片，使用 CRC32 哈希函数将键转换为一个数字，再对实例数量求模就能知道应该存储的实例。

根据执行分片的位置，可以分为三种分片方式：

- 客户端分片：客户端使用一致性哈希等算法决定键应当分布到哪个节点。
- 代理分片：将客户端请求发送到代理上，由代理转发请求到正确的节点上。
- 服务器分片：Redis Cluster



### 常见应用

#### 分布式限流器 RRateLimiter

本质是令牌桶算法，RRateLimiter 设置了 Token 生成速度如1秒100个。调用接口前先去从 RRateLimiter 中拿 Token， 如果获取失败则拒绝请求。

```Java
Config config = new Config();
config.useSingleServer().setTimeout(1000000).setAddress("redis://127.0.0.1:6379");

RedissonClient redisson = Redisson.create(config);
RRateLimiter rateLimiter = redisson.getRateLimiter("myRateLimiter");
// 初始化
// 最大流速 = 每1秒钟产生1个令牌
rateLimiter.trySetRate(RateType.OVERALL, 1, 1, RateIntervalUnit.SECONDS);
return rateLimiter;
```



1. 创建限流器

   把速率和模式放到 hash 中

   ```pseudocode
   redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);
   redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);
   return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);
   ```

   

2. 获取 Token 

   ```pseudocode
   -- 速率
   local rate = redis.call("hget", KEYS[1], "rate")
   -- 时间区间(ms)
   local interval = redis.call("hget", KEYS[1], "interval")
   local type = redis.call("hget", KEYS[1], "type")
   assert(rate ~= false and interval ~= false and type ~= false, "RateLimiter is not initialized")
   
   -- {name}:value 分析后面的代码，这个key记录的是当前令牌桶中的令牌数
   local valueName = KEYS[2]
   
   -- {name}:permits 这个key是一个zset，记录了请求的令牌数，score则为请求的时间戳
   local permitsName = KEYS[4]
   
   -- 单机限流才会用到，集群模式不用关注
   if type == "1" then
       valueName = KEYS[3]
       permitsName = KEYS[5]
   end
   
   -- 原版本有bug(https://github.com/redisson/redisson/issues/3197)，最新版将这行代码提前了
   -- rate为1 arg1这里是 请求的令牌数量(默认是1)。rate必须比请求的令牌数大
   assert(tonumber(rate) >= tonumber(ARGV[1]), "Requested permits amount could not exceed defined rate")
   
   -- 第一次执行这里应该是null，会进到else分支
   -- 第二次执行到这里由于else分支中已经放了valueName的值进去，所以第二次会进if分支
   local currentValue = redis.call("get", valueName)
   if currentValue ~= false then
       -- 从第一次设的zset中取数据，范围是0 ~ (第二次请求时间戳 - 令牌生产的时间)
       -- 可以看到，如果第二次请求时间距离第一次请求时间很短(小于令牌产生的时间)，那么这个差值将小于上一次请求的时间，取出来的将会是空列表。反之，能取出之前的请求信息
       -- 这里作者将这个取出来的数据命名为expiredValues，可认为指的是过期的数据
       local expiredValues = redis.call("zrangebyscore", permitsName, 0, tonumber(ARGV[2]) - interval)
       local released = 0
       -- lua迭代器，遍历expiredValues，如果有值，那么released等于之前所有请求的令牌数之和，表示应该释放多少令牌
       for i, v in ipairs(expiredValues) do
           local random, permits = struct.unpack("fI", v)
           released = released + permits
       end
   
       -- 没有过期请求的话，released还是0，这个if不会进，有过期请求才会进
       if released > 0 then
           -- 移除zset中所有元素，重置周期
           redis.call("zrem", permitsName, unpack(expiredValues))
           currentValue = tonumber(currentValue) + released
           redis.call("set", valueName, currentValue)
       end
   
       -- 这里简单分析下上面这段代码:
       -- 1. 只有超过了1个令牌生产周期后的请求，expiredValues才会有值。
       -- 2. 以rate为3举例，如果之前发生了两个请求那么现在released为2，currentValue为1 + 2 = 3
       -- 以此可以看到，redisson的令牌桶放令牌操作是通过请求时间窗来做的，如果距离上一个请求的时间已经超过了一个令牌生产周期时间，那么令牌桶中的令牌应该得到重置，表示生产rate数量的令牌。
   
       -- 如果当前令牌数 ＜ 请求的令牌数
       if tonumber(currentValue) < tonumber(ARGV[1]) then
           -- 从zset中找到距离当前时间最近的那个请求，也就是上一次放进去的请求信息
           local nearest = redis.call('zrangebyscore', permitsName, '(' .. (tonumber(ARGV[2]) - interval), tonumber(ARGV[2]), 'withscores', 'limit', 0, 1); 
           local random, permits = struct.unpack("fI", nearest[1])
           -- 返回 上一次请求的时间戳 - (当前时间戳 - 令牌生成的时间间隔) 这个值表示还需要多久才能生产出足够的令牌
           return tonumber(nearest[2]) - (tonumber(ARGV[2]) - interval)
       else
           -- 如果当前令牌数 ≥ 请求的令牌数，表示令牌够多，更新zset
           redis.call("zadd", permitsName, ARGV[2], struct.pack("fI", ARGV[3], ARGV[1]))
           -- valueName存的是当前总令牌数，-1表示取走一个
           redis.call("decrby", valueName, ARGV[1])
           return nil
       end
   else
       -- set一个key-value数据 记录当前限流器的令牌数
       redis.call("set", valueName, rate)
       -- 建了一个以当前限流器名称相关的zset，并存入 以score为当前时间戳，以lua格式化字符串{当前时间戳为种子的随机数、请求的令牌数}为value的值。
       -- struct.pack第一个参数表示格式字符串，f是浮点数、I是长整数。所以这个格式字符串表示的是把一个浮点数和长整数拼起来的结构体。我的理解就是往zset里记录了最后一次请求的时间戳和请求的令牌数
       redis.call("zadd", permitsName, ARGV[2], struct.pack("fI", ARGV[3], ARGV[1]))
       -- 从总共的令牌数 减去 请求的令牌数。
       redis.call("decrby", valueName, ARGV[1])
       return nil
   end
   ```

   redisson 用一个 string 来保存当前剩余 Token 的数量。

   redisson 用 zset 来记录请求的信息，value 为一次请求的令牌数，score 为该次请求的时间戳。通过比较 score 来判断当前请求距离上一个请求有没有超过一个 令牌生成周期。

   - 如果超过了，则算出 zset 中当前时间戳-Token 生成周期以前的所有请求都过期，重新补充这些请求数量的 Token。

   然后比较 Token 的数量和请求的数量

   - 如果够则 Token - 请求量，返回
   - 否则返回到一下个 Token 生成还需要多少时间，返回时间

   

   ```java
   private void tryAcquireAsync(long permits, RPromise<Boolean> promise, long timeoutInMillis) {
       long s = System.currentTimeMillis();
       RFuture<Long> future = tryAcquireAsync(RedisCommands.EVAL_LONG, permits);
       future.onComplete((delay, e) -> {
           if (e != null) {
               promise.tryFailure(e);
               return;
           }
           
           if (delay == null) {
               //delay就是lua返回的 还需要多久才会有令牌
               promise.trySuccess(true);
               return;
           }
           
           //没有手动设置超时时间的逻辑
           if (timeoutInMillis == -1) {
               //延迟delay时间后重新执行一次拿令牌的动作
               commandExecutor.getConnectionManager().getGroup().schedule(() -> {
                   tryAcquireAsync(permits, promise, timeoutInMillis);
               }, delay, TimeUnit.MILLISECONDS);
               return;
           }
           
           //el 请求redis拿令牌的耗时
           long el = System.currentTimeMillis() - s;
           //如果设置了超时时间，那么应该减去拿令牌的耗时
           long remains = timeoutInMillis - el;
           if (remains <= 0) {
               //如果那令牌的时间比设置的超时时间还要大的话直接就false了
               promise.trySuccess(false);
               return;
           }
           //比如设置的的超时时间为1s，delay为1500ms，那么1s后告知失败
           if (remains < delay) {
               commandExecutor.getConnectionManager().getGroup().schedule(() -> {
                   promise.trySuccess(false);
               }, remains, TimeUnit.MILLISECONDS);
           } else {
               long start = System.currentTimeMillis();
               commandExecutor.getConnectionManager().getGroup().schedule(() -> {
                   //因为这里是异步的，所以真正再次拿令牌之前再检查一下过去了多久时间。如果过去的时间比设置的超时时间大的话，直接false
                   long elapsed = System.currentTimeMillis() - start;
                   if (remains <= elapsed) {
                       promise.trySuccess(false);
                       return;
                   }
                   //再次拿令牌
                   tryAcquireAsync(permits, promise, remains - elapsed);
               }, delay, TimeUnit.MILLISECONDS);
           }
       });
   }
   
   ```

   redis 返回给 java delay 字段。

   - delay 为 null，表示成果获得令牌
   - delay 有值， 则通过 delay 时间后通过异步线程拿令牌。非公平的。

