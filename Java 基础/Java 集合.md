# Java集合

## 集合框架总览

![image-20210105190533863](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210105190533863.png)

1. 两个**遍历接口**，Iterator/ListIterator，后者是前者的优化版，支持在任意一个位置进行前后双向遍历
2. Collection/Map，前者存储一系列**对象**，后者存储一系列**key-value对**
3. 四种具体的集合类型：Map、(Set、List、Queue)
   1. Map:<key, value>
   2. Set:**不可重复**，**无序**
   3. List:**可重复**，**有序**按插入顺序排列
   4. Queue:**队列**，特性和List相同，只能从**队尾、队头**操作
4. 两个工具类Collections/Arrays
5. 不同场景下选择不同的集合类型，没有最佳的结合



## Iterator、Iterable、ListIterator

### Iterator、Iterable区别

区别主要在于

- Iterable中提供了Iterator接口，因此实现了Iterable接口的类依旧可以使用iterator遍历元素
- JDK1.8中, Iterable提供了forEach的**语法糖**，本质仍是Iterator遍历

原因：

- Iterator的保留可以让子类去实现自己的迭代器
- Iterable接口更加关注于forEach增强语法

### ListIterator

继承Iterator接口，在遍历**List**集合时可以从**任意索引下标**开始遍历，支持**双向遍历**

![image-20210105194119494](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210105194119494.png)



## Map、Collection

![image-20210106191925028](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210106191925028.png)

Map:

- AbstractMap: 为子类提供一些**通用的API实现**，所有具体的Map如HashMap都继承它
- SortedMap: 可以对<key, value>按自己定义的规则进行**排序**，具体实现如TreeMap

Collection:提供了所有集合类的通用方法

- add/addAll
- remove/removeAll
- contains/containsAlll
- size()/isEmpty()



### Map

![image-20210106192957186](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210106192957186.png)

Map的设计理念：定位元素的时间复杂度优化至O(1)

#### HashMap

将key的哈希值转换为数组的索引下标，**确定存放位置**，查找时根据key的哈希值转换得到的下标，去数组中对应位置查找。

![image-20210107191949470](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210107191949470.png)

- 底层数组+链表+红黑树
- **非线程安全**
- 发生哈希冲突时，将相同地址的元素连成一条**链表**，链表长度大于8且数组长度大于64时转换为**红黑树**

主要内部功能：

1.根据key计算哈希值

本质上分为三步：取key.hashCode(), 高位运算, 取模运算。

![image-20210107192950907](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210107192950907.png)

2.put方法

![image-20210107193158249](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210107193158249.png)

3.扩容机制

用一个扩大了两倍的数组代替原来的数组，tranfer()将原有的Entry数组的元素采用头插法拷贝置新Entry数组中，JDK 1.7中原先在同一条链上的元素扩容后仍在同一条链上，JDK 1.8则不一定在同一条链上。



#### LinkedHashMap

在HashMap的基础上添加了一条双向链表，默认存储各个**元素的插入顺序**。也可以是实现LRU（最近最少使用）缓存淘汰策略，因此也可以设置顺序按照**元素的访问次序**

![image-20210107195529260](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210107195529260.png)

- 底层维护了一条双向链表，继承HashMap，因此也是**线程不安全的**
- 可实现LRU淘汰策略，1.设置accessOrder为true 2.重写removeEldestEntry方法，定义在容量达到threshold后淘汰链头元素的逻辑



#### TreeMap

SortedMap的子类，元素有序。基于**红黑树**实现。默认按照key排序，也可以传入Comparator自定义排序规则。

![image-20210107200225437](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210107200225437.png)

- 底层红黑树，操作的时间复杂度恒为O(logn)
- 线程不安全
- 可排序

排序前提：

- 默认排序需要key实现了**Comparable**接口
- 自定义排序需要在初始化TreeMap时传入Comparator，key不需要实现Comparable接口



#### WeakHashMap

Entry中的键在每次GC中都会被回收。内部维护了一个引用队列queue，包含了所有被GC掉的键，在expungeStaleEntries()中遍历queue，在WeekHashMap中删除对应key的Entry。

![image-20210107200818512](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210107200818512.png)

- 继承AbstractMap，底层数组+链表，线程不安全
- 弱键，随时可能被GC，不能保证某次访问元素一定存在
- 通常作为缓存使用，**短暂访问，只访问一次**



#### Hashtable

底层数组+链表，线程安全，并发环境下性能十分低效，因为它所有的方法都加上了synchronized关键字。



### Collection

![image-20210107201303791](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210107201303791.png)



### Set

Set集合中任意两个元素不会出现o1.equals(o2),且set中至多只有一个NULL元素

- 存入**可变元素**时，后续变化可能导致set中出现两个相等的元素，因此一般不存var进入Set中
- Set实际运用于判重



#### AbstractSet

抽象类，将Set中的相同行为定义在这里，避免子类**包含大量重复代码**

所有的Set有相同的hashCode()和equals()方法，抽象类重写这两个方法后，子类就不用关注了



#### SortedSet

在Set的基础上扩展了**排序**的行为。

![image-20210108164534974](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210108164534974.png)



#### HashSet

底层借助HashMap实现，key是存入的对象，value为PRESENT静态常量

![image-20210108165043125](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210108165043125.png)

只需要关注key，屏蔽了HashMap的value

![image-20210108165140022](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210108165140022.png)

- 底层数组+链表+红黑树
- 线程不安全
- 存入HashSet中的对象状态**最好不应该发生变化**



#### LinkedHashSet

底层使用LinkedHashMap，即HashMap+双向链表，双向链表中保存了元素的插入顺序

![image-20210108165731433](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210108165731433.png)

- 线程不安全
- 继承HashSet，HashSet默认使用HashMap对象，但LinkedHashMap调用父类构造函数时采用LinkedHashMap



#### TreeSet

基于TreeMap实现，底层数组+红黑树，元素有序

![image-20210108170038359](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210108170038359.png)

与TreeMap类似，可以采用默认排序（key实现Comparable接口），或者自定义排序（Comparator）

- TreeSet的操作底层都是对TreeMap的操作，红黑树操作的平均复杂度O(logn)
- 线程不安全
- 通常应用于不重复的元素自定义排序
- 判断重复不是equals方法，而是compareTo方法是否返回0



### List

元素可重复、有序，按照插入顺序排列，可以根据索引随机读取



#### AbstractList/AbstractSequentialList

抽象类，内部实现List共同的逻辑。AbstractSequentialList继承AbstractList，限制**只能顺序访问**，无法随机访问，LinkedList继承AbstractSequentialList



#### Vector

- 过时，由于每个方法都加上synchronized修饰，性能低下
- 线程安全的场景下，使用ArrayList，并发环境下，使用 CopyOnWriteArrayList 或者 Collections.synchronizedLsist()



#### Stack

- 继承Vector，同样过时
- LIFO，push/pop/peek
- 替代使用Deque接口下的ArrayDeque实现



#### ArrayList

底层数组，线程不安全，查询O(1)，在数组中间或头部插入O(n)

![image-20210108172216141](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210108172216141.png)

- 随机访问，访问元素效率高，频繁插入删除效率低
- 首次扩容长度为**10**
- 第二次之后的扩容，数组长度扩展至原来的1.5倍



#### LinkedList

底层双向链表，链表的内存地址不连续，删除、插入只需要操作指针，不需要移动元素

![image-20210108172751031](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210108172751031.png)

- 无扩容机制，插入和删除元素效率较高，适用于频繁操作元素的场景
- 查找效率较低，查找某个index做了优化，若index大于size/2，则从头开始找，否则从尾部开始找
- 实现了Deque，可以用作双端队列



### Queue

两种不同类型的队列实现：单向队列（AbstractQueue），双端队列（Deque）

Queue接口中提供了两套增加、删除的API，对应失败时两种不同的处理策略

|                | 插入方法 | 删除方法 | 查找方法 |
| -------------- | -------- | -------- | -------- |
| 抛出异常       | add      | remove   | get      |
| 返回失败默认值 | offer    | poll     | peek     |



#### LinkedList

同上



#### ArrayDeque

底层数组，无界，JDK 1.8下默认最小容量为8，JDK 11中为16

- 使用**数组**实现双端队列，作为队列使用效率高于LinkedList，下标移动代价小于指针移动
- 避免频繁扩容，导致元素的移动和复制



#### PriorityQueue

底层数组，元素有序，默认按照元素本身排序（实现Comparable接口），可以自定义排序规则（实现Comparator），因此PriorityQueue不允许存储NULL元素

- 基于**优先级堆**实现的优先级队列，堆使用数组维护
- 适用于元素需要按照优先级处理的场景



### 集合总结

| 集合          | 插入、删除时间复杂度 | 查询    | 底层数据结构              | 线程安全 |
| ------------- | -------------------- | ------- | ------------------------- | -------- |
| HashSet       | O(1)                 | O(1)    | 数组+链表+红黑树(HashMap) | 否       |
| LinkedHashSet | O(1)                 | O(1)    | 数组+链表+红黑树+双向链表 | 否       |
| TreeSet       | O(logn)              | O(logn) | 数组+红黑树               | 否       |
| Vector        | O(n)                 | O(1)    | 数组                      | 是       |
| Stack         | O(n)                 | O(1)    | 数组                      | 是       |
| ArrayList     | O(n)                 | O(1)    | 数组                      | 否       |
| LinkedList    | O(1)                 | O(n)    | 链表                      | 否       |
| ArrayDeque    | O(n)                 | O(1)    | 数组                      | 否       |
| PriorityQueue | O(logn)              | O(logn) | 数组                      | 否       |

## 集合中的设计模式

### 迭代器模式

![image-20210221142641705](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210221142641705.png)

Collection 继承了 Iterable 接口，其中的 iterator() 方法能够产生一个 Iterator 对象，通过这个对象就可以迭代遍历 Collection 中的元素。

从 JDK 1.5 之后可以使用 foreach 方法来遍历实现了 Iterable 接口的集合对象。

### 适配器模式

java.util.Arrays#asList() 可以把数组类型转换为 List 类型。

```java
@SafeVarargs
public static <T> List<T> asList(T... a)
```

应该注意的是 asList() 的参数为泛型的变长参数，不能使用基本类型数组作为参数，只能使用相应的包装类型数组。

```java
Integer[] arr = {1, 2, 3};
List list = Arrays.asList(arr);
```

也可以使用以下方式调用 asList()：

```java
List list = Arrays.asList(1, 2, 3);
```

## 源码分析

以下源码分析未做特殊说明都是基于JDK1.8

### ArrayList

1.概览

基于数组实现，支持快速随机访问。以下是ArrayList实现的接口。

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

数组初始大小默认为10。

```java
private static final int DEFAULT_CAPACITY = 10;
```

![image-20210221143326175](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210221143326175.png)

2.扩容

- 添加元素时 ensureCapacityInternal() 方法保证容量足够。

- 容量不够时 grow() 扩容。新容量的大小为 `oldCapacity + (oldCapacity >> 1)`，即 oldCapacity+oldCapacity/2。其中 oldCapacity >> 1 需要取整，所以新容量大约是旧容量的 1.5 倍左右。

- 扩容时需要调用 `Arrays.copyOf()` 复制原数组，开销较大。因此创建时最好指定大概容量大小，避免频繁扩容。

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

3.删除元素

需要调用 `System.arraycopy()` 将index+1 后面的元素都复制到 index 位置上，时间复杂度为O(n)。

```java
public E remove(int index) {
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}
```

4.序列化

ArrayList 基于数组实现，并且具有动态扩容特性，因此保存元素的数组不一定都会被使用，那么就没必要全部进行序列化。

保存元素的数组 elementData 使用 transient 修饰，该关键字声明数组默认不会被序列化。

```java
transient Object[] elementData; // non-private to simplify nested class access
```

ArrayList 实现了 writeObject() 和 readObject() 来控制只序列化数组中有元素填充那部分内容。

```java
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    elementData = EMPTY_ELEMENTDATA;

    // Read in size, and any hidden stuff
    s.defaultReadObject();

    // Read in capacity
    s.readInt(); // ignored

    if (size > 0) {
        // be like clone(), allocate array based upon size not capacity
        ensureCapacityInternal(size);

        Object[] a = elementData;
        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            a[i] = s.readObject();
        }
    }
}
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```

5.Fail-Fast

modCount 用来记录 ArrayList 结构发生变化的次数。结构发生变化是指添加或者删除至少一个元素的所有操作，或者是调整内部数组的大小，仅仅只是设置元素的值不算结构发生变化。

在进行序列化或者迭代等操作时，需要比较操作前后 modCount 是否改变，如果改变了需要抛出 ConcurrentModificationException。

iterator 对象初始化时就将集合当前 modCount 赋值给 expectedModeCount，因此当遍历期间发生元素数量改变的情况会导致 modCount 和 expectedModCount 不等。

### CopyOnWriteArrayList

1.读写分离

- 写操作在复制的新数组上进行，读操作在原始数组中进行。
- 写操作需要加锁，防止并发写入时数据丢失。
- 写操作之后需要把原始数组指向新的复制数组。

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

final void setArray(Object[] a) {
    array = a;
}
```

2.使用场景

CopyOnWriteArrayList 在写操作的同时允许读操作，大大提高了读操作的性能，因此很适合读多写少的应用场景。

但是 CopyOnWriteArrayList 有其缺陷：

- 内存占用：在写操作时需要复制一个新的数组，使得内存占用为原来的两倍左右；
- 数据不一致：读操作不能读取实时性的数据，因为部分写操作的数据还未同步到读数组中。

所以 CopyOnWriteArrayList 不适合内存敏感以及对实时性要求很高的场景。

### LinkedList

1.概览

基于双链表实现，使用Node存储链表节点信息。

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
}
```

每个链表存储了头尾指针。

```java
transient Node<E> first;
transient Node<E> last;
```



2.与 ArrayList 的比较

ArrayList 基于动态数组实现，LinkedList 基于双向链表实现。ArrayList 和 LinkedList 的区别可以归结为数组和链表的区别：

- 数组支持随机访问，但插入删除的代价很高，需要移动大量元素；
- 链表不支持随机访问，但插入删除只需要改变指针。

### HashMap

以下源码以JDK1.7为主。

1.存储结构

Entry 类型的数组，初始大小默认为16。数组中每个位置为一个桶，一个桶存放一个链表。同一个链表中存放哈希值和散列桶长度取模运算结果相同的 Entry。

```java
transient Entry[] table;
```

Entry 存储键值对，包含四个字段。

```java
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    int hash;

    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }

    public final K getKey() {
        return key;
    }

    public final V getValue() {
        return value;
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry e = (Map.Entry)o;
        Object k1 = getKey();
        Object k2 = e.getKey();
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
            Object v1 = getValue();
            Object v2 = e.getValue();
            if (v1 == v2 || (v1 != null && v1.equals(v2)))
                return true;
        }
        return false;
    }

    public final int hashCode() {
        return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());
    }

    public final String toString() {
        return getKey() + "=" + getValue();
    }
}
```

2.拉链法原理

```java
HashMap<String, String> map = new HashMap<>();
map.put("K1", "V1");
map.put("K2", "V2");
map.put("K3", "V3");
```

- 新建一个 HashMap，默认大小为 16；
- 插入 <K1,V1> 键值对，先计算 K1 的 hashCode 为 115，使用除留余数法得到所在的桶下标 115%16=3。
- 插入 <K2,V2> 键值对，先计算 K2 的 hashCode 为 118，使用除留余数法得到所在的桶下标 118%16=6。
- 插入 <K3,V3> 键值对，先计算 K3 的 hashCode 为 118，使用除留余数法得到所在的桶下标 118%16=6，采用**头插法**插在 <K2,V2> 前面。

3.put操作

- 首先计算 hash 值，然后找到桶下标。
- 遍历链表。如果已经存在键为 key 的 Entry，则更新这个 key 的 value 为新插入的值
- 不存在的话插入新的 Entry

```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    // 键为 null 单独处理
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    // 确定桶下标
    int i = indexFor(hash, table.length);
    // 先找出是否已经存在键为 key 的键值对，如果存在的话就更新这个键值对的值为 value
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    // 插入新键值对
    addEntry(hash, key, value, i);
    return null;
}
```

HashMap 允许插入键为 null 的键值对。但是因为无法调用 null 的 hashCode() 方法，也就无法确定该键值对的桶下标，只能通过强制指定一个桶下标来存放。HashMap 使用第 0 个桶存放键为 null 的键值对。

```java
private V putForNullKey(V value) {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}
```

头插法插入新的 Entry。

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    // 头插法，链表头部指向新的键值对
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

4.确定桶下标

很多操作都需要先确定一个键值对所在的桶下标。

```java
int hash = hash(key);
int i = indexFor(hash, table.length);
```

计算 hash 值

```java
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

```java
public final int hashCode() {
    return Objects.hashCode(key) ^ Objects.hashCode(value);
}
```

取模

令 x = 1<<4，即 x 为 2 的 4 次方，它具有以下性质：

```
x   : 00010000
x-1 : 00001111
```

令一个数 y 与 x-1 做与运算，可以去除 y 位级表示的第 4 位以上数：

```
y       : 10110010
x-1     : 00001111
y&(x-1) : 00000010
```

这个性质和 y 对 x 取模效果是一样的：

```
y   : 10110010
x   : 00010000
y%x : 00000010
```

我们知道，位运算的代价比求模运算小的多，因此在进行这种计算时用位运算的话能带来更高的性能。

确定桶下标的最后一步是将 key 的 hash 值对桶个数取模：hash%capacity，如果能保证 capacity 为 2 的 n 次方，那么就可以将这个操作转换为**与运算**。

```java
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

5.扩容

设 HashMap 的 table 长度为 M，需要存储的键值对数量为 N，如果哈希函数满足均匀性的要求，那么每条链表的长度大约为 N/M，因此查找的复杂度为 O(N/M)。

为了让查找的成本降低，应该使 N/M 尽可能小，因此需要保证 M 尽可能大，也就是说 table 要尽可能大。HashMap 采用动态扩容来根据当前的 N 值来调整 M 值，使得空间效率和时间效率都能得到保证。

和扩容相关的参数主要有：capacity、size、threshold 和 load_factor。

- capacity：table 的容量大小，默认为 16。需要注意的是 capacity 必须保证为 2 的 n 次方。
- size：实际大小。
- threshold：size 的临界值，当 size 大于等于 threshold 就必须进行扩容操作。
- load_factor：装载因子，table 能使用的比例，`threshold = (int)(capacity* loadFactor)`。

```java
static final int DEFAULT_INITIAL_CAPACITY = 16;

static final int MAXIMUM_CAPACITY = 1 << 30;

static final float DEFAULT_LOAD_FACTOR = 0.75f;

transient Entry[] table;

transient int size;

int threshold;

final float loadFactor;

transient int modCount;
```

扩容使用 resize() 实现，需要注意的是，扩容操作同样需要把 oldTable 的所有键值对重新插入 newTable 中，因此这一步是很费时的。

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable);
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}

void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```

扩容后需要重新计算桶下标，在前面提到，HashMap 使用 hash%capacity 来确定桶下标。HashMap capacity 为 **2 的 n 次方**这一特点能够极大降低重新计算桶下标操作的复杂度。

```
a % b = a&(b-1) (当 b=2^n 时)
```

假设原数组长度 capacity 为 16，扩容之后 new capacity 为 32：

```
capacity     : 00010000
new capacity : 00100000
```

对于一个 Key，它的哈希值 hash 在第 5 位：

- 为 0，那么 xxx0xxxx%00100000 = xxx0xxxx&00011111，桶位置和原来一致；
- 为 1，xxx1xxxx%00100000 = xxx1xxxx&00011111，桶位置是原位置 + 16。

6.计算数组容量

HashMap 构造函数允许用户传入的容量不是 2 的 n 次方，因为它可以自动地将传入的容量转换为 2 的 n 次方。

先考虑如何求一个数的掩码，对于 10010000，它的掩码为 11111111，可以使用以下方法得到：

```java
mask |= mask >> 1    11011000
mask |= mask >> 2    11111110
mask |= mask >> 4    11111111
```

mask+1 是大于原始数字的最小的 2 的 n 次方。

```java
num     10010000
mask+1 100000000
```

以下是 HashMap 中计算数组容量的代码：

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

7.链表转红黑树

从JDK1.8开始，链表长度大于等于 8 时会将链表转换为红黑树，使查询时间复杂度由 O(1) 变成 O(logn)。

链表树化的条件

- 链表长度大于 8
- 哈希数组长度大于 64

如果链表长度条件满足，但哈希数组不满足，在 `treeifyBin` 时会进行扩容操作，因为 table 容量较小时节点更容易产生冲突，因此优先扩容，而不是立即树化。

同样在扩容时可能会将红黑树拆分为两棵树，或者红黑树太小（小于 等于6 ）时退化成链表。

### ConcurrentHashMap

#### 1.7

1.存储结构

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
}
```

ConcurrentHashMap 和 HashMap 实现上类似，最主要的差别是 ConcurrentHashMap 采用了分段锁（Segment），每个分段锁维护着几个桶（HashEntry），多个线程可以同时访问不同分段锁上的桶，从而使其并发度更高（并发度就是 Segment 的个数）。

Segment 继承自 ReentrantLock。

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {

    private static final long serialVersionUID = 2249069246763182397L;

    static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

    transient volatile HashEntry<K,V>[] table;

    transient int count;

    transient int modCount;

    transient int threshold;

    final float loadFactor;
}
```

```java
final Segment<K,V>[] segments;
```

默认并发级别为 16，即默认创建 16 个 Segment。

```java
static final int DEFAULT_CONCURRENCY_LEVEL = 16;
```

2.size 操作

每个 Segment 维护了一个 count 变量来统计该 Segment 中的键值对个数。执行 size() 时需要遍历所有的 Segment 并累加 count。

- 首先尝试不加锁，如果连续两次不加锁获得结果一致，那就认为该结果正确。
- 尝试次数使用 RETRIES_BEFORE_LOCK 定义，该值为 2，retries 初始值为 -1，因此尝试次数为 3。
- 超过尝试次数机会对每个 Segment 加锁

```java
/**
 * Number of unsynchronized retries in size and containsValue
 * methods before resorting to locking. This is used to avoid
 * unbounded retries if tables undergo continuous modification
 * which would make it impossible to obtain an accurate result.
 */
static final int RETRIES_BEFORE_LOCK = 2;

public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            // 超过尝试次数，则对每个 Segment 加锁
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            // 连续两次得到的结果一致，则认为这个结果是正确的
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```

#### 1.8

JDK 1.7 使用分段锁机制来实现并发更新操作，核心类为 Segment，它继承自重入锁 ReentrantLock，并发度与 Segment 数量相等。

JDK 1.8 使用了 CAS 操作来支持更高的并发度，在 CAS 操作失败时使用内置锁 synchronized。底层数据结果为 volatile Node 数组+链表+红黑树。

1.数据结构

Node 数组用于存储哈希表节点，被 volatile 修饰

```java
transient volatile Node<K,V>[] table;
```

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
}
```

2.put 操作

1. 懒汉式初始化 table，若未初始化则调用 initTable()
2. 如果没有 hash 冲突就直接 CAS 插入
3. 如果还在扩容就先进行扩容操作
4. 如果存在 hash 冲突，就加锁来保证线程安全。如果该 hash 表该下标下是链表，则遍历更新或者插入到尾部，若是红黑树，则按照红黑树的方法更新或者插入。
5. 判断是否需要红黑树化
6. addCount()，检查是否需要扩容

3.get 操作

1. 计算 hash 值，定位到 table 对应下标，如果是首节点直接返回 val
2. 如果遇到扩容，调用表示该桶扩容已经完成的节点 ForwardingNode 的find 方法查找该 key，匹配就返回
3. 继续遍历结点，匹配就返回 val 否则返回 null

4.size 操作

通过遍历 baseCount 和 counterCell 得到 size 大小、

ConcurrentHashMap 的并发特性使它的 size 无法达到精准，因为在调用 size() 的同时可能有线程正在进行 put() 操作，对 baseCount 和 counterCell 进行 CAS 赋值。



### LinkedHashMap

1.存储结构

继承自 HashMap，因此具有和 HashMap 一样的快速查找特性。

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
```

内部维护了一个双向链表，用来维护插入顺序或者 LRU 顺序。

```java
/**
 * The head (eldest) of the doubly linked list.
 */
transient LinkedHashMap.Entry<K,V> head;

/**
 * The tail (youngest) of the doubly linked list.
 */
transient LinkedHashMap.Entry<K,V> tail;
```

accessOrder 决定了顺序，默认为 false，此时维护的是插入顺序。

```java
final boolean accessOrder;
```

LinkedHashMap 最重要的是以下用于维护顺序的函数，它们会在 put、get 等方法中调用。

```java
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
```

2.afterNodeAccess()

当一个节点被访问时，如果 accessOrder 为 true，则会将该节点移到链表尾部。也就是说指定为 LRU 顺序之后，在每次访问一个节点时，会将这个节点移到链表尾部，保证链表尾部是最近访问的节点，那么链表首部就是最近最久未使用的节点。

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

3.afterNodeInsertion()

在 put 等操作之后执行，当 removeEldestEntry() 方法返回 true 时会移除最晚的节点，也就是链表首部节点 first。

evict 只有在构建 Map 的时候才为 false，在这里为 true。

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```

removeEldestEntry() 默认为 false，如果需要让它为 true，需要继承 LinkedHashMap 并且覆盖这个方法的实现，这在实现 LRU 的缓存中特别有用，通过移除最近最久未使用的节点，从而保证缓存空间足够，并且缓存的数据都是热点数据。

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

4.LRU 缓存

以下是使用 LinkedHashMap 实现的一个 LRU 缓存：

- 设定最大缓存空间 MAX_ENTRIES 为 3；
- 使用 LinkedHashMap 的构造函数将 accessOrder 设置为 true，开启 LRU 顺序；
- 覆盖 removeEldestEntry() 方法实现，在节点多于 MAX_ENTRIES 就会将最近最久未使用的数据移除。

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private static final int MAX_ENTRIES = 3;

    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > MAX_ENTRIES;
    }

    LRUCache() {
        super(MAX_ENTRIES, 0.75f, true);
    }
}
public static void main(String[] args) {
    LRUCache<Integer, String> cache = new LRUCache<>();
    cache.put(1, "a");
    cache.put(2, "b");
    cache.put(3, "c");
    cache.get(1);
    cache.put(4, "d");
    System.out.println(cache.keySet());
}
[3, 1, 4]
```

### WeakHashMap

1.存储结构

WeakHashMap 的 Entry 继承自 WeakReference，被 WeakReference 关联的对象在下一次垃圾回收时会被回收。

WeakHashMap 主要用来实现缓存，通过使用 WeakHashMap 来引用缓存对象，由 JVM 对这部分缓存进行回收。

```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V>
```

2.ConcurrentCache

Tomcat 中的 ConcurrentCache 使用了 WeakHashMap 来实现缓存功能。

ConcurrentCache 采取的是分代缓存：

- 经常使用的对象放入 eden 中，eden 使用 ConcurrentHashMap 实现，不用担心会被回收（伊甸园）；
- 不常用的对象放入 longterm，longterm 使用 WeakHashMap 实现，这些老对象会被垃圾收集器回收。
- 当调用 get() 方法时，会先从 eden 区获取，如果没有找到的话再到 longterm 获取，当从 longterm 获取到就把对象放入 eden 中，从而保证经常被访问的节点不容易被回收。
- 当调用 put() 方法时，如果 eden 的大小超过了 size，那么就将 eden 中的所有对象都放入 longterm 中，利用虚拟机回收掉一部分不经常使用的对象。

```java
public final class ConcurrentCache<K, V> {

    private final int size;

    private final Map<K, V> eden;

    private final Map<K, V> longterm;

    public ConcurrentCache(int size) {
        this.size = size;
        this.eden = new ConcurrentHashMap<>(size);
        this.longterm = new WeakHashMap<>(size);
    }

    public V get(K k) {
        V v = this.eden.get(k);
        if (v == null) {
            v = this.longterm.get(k);
            if (v != null)
                this.eden.put(k, v);
        }
        return v;
    }

    public void put(K k, V v) {
        if (this.eden.size() >= size) {
            this.longterm.putAll(this.eden);
            this.eden.clear();
        }
        this.eden.put(k, v);
    }
}
```