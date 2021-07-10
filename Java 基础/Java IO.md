# Java I/O

## IO流

### 分类

特点：

- 顺序读写
- 字节数组：读写数据本质上都是对字节数字进行读取和写出操作。



根据**处理数据的基本单位**的不同：

- 字符流：文本
- 字节流：音频，视频，图片等

|        | 字节流       | 字符流 |
| ------ | ------------ | ------ |
| 输入流 | InputStream  | Reader |
| 输出流 | OutputStream | Writer |

![image-20201222155702988](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20201222155702988.png)



字节流和字符流之间的转换：

- **输入流**：字节流 => 字符流 (InputStreamReader)
- **输出流**：字符流 => 字节流 (OutputStreamWriter)

在存储设备上数据都是以字节存储的，不论输入到内存，或者从内存输出到硬盘，都是以字节为单位进行的。



### 节点流和处理流

- 节点流：**真正传输数据**的对象，用于向特定的一个地方（节点）读写数据。例如FileInputStream
- 处理流：对**节点流的封装**，本质上利用节点流读写数据，外层的处理流提供额外的功能。处理流的基类都是以Filter开头。



## 核心类File

### 简介

File类指向操作系统中的文件和目录，通过该类只能访问文件和目录，无法访问内容。内部提供三类操作：

- 访问文件属性：绝对路径、相对路径、文件名……
- 文件检测：是否文件、是否目录、文件是否存在、rwx权限……
- 文件操作：创建文件、目录、删除文件……



### 字节流对象（Stream）

![image-20201222160528221](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20201222160528221.png)

#### InputStream

字节输入流的抽象基类，重要方法

| 重要方法                                    | 功能                                        |
| ------------------------------------------- | ------------------------------------------- |
| public abstract int read()                  | 从输入流中读取下一个字节，读到尾部返回-1    |
| public int read(byte b[])                   | 从输入流读取长度为b.length个字节，并放入b中 |
| public int read(byte b[], int off, int len) | 从输入流中读取指定范围的字节，放入b中       |
| public void close()                         | 关闭输入流并释放资源                        |

##### ByteArrayInputStream

内部维护一个buf字节数组缓存区，pos指向读取下一个字节的下标位置，内部还维护了一个count属性

![image-20201222161751892](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20201222161751892.png)

##### FileInputStream

文件输入流，通常对文件的拷贝、移动等操作

##### PipedInputStream

管道输入流，与PipedOutputStream成对出现，可以实现多线程中的**管道通信**

##### ObjectInputStream

对象输入流，用于对象的反序列化。

##### PushBackInputStream

一个处理流，内部维护一个**buf缓存数组**

- 在读入字节的过程中可以将**字节数据回退给缓存区**，下次可以直接从缓存区中读取之前的字节数。即允许多次读取输入流的字节数据

使用场景：**对数据进行分类规整**，例如对大文件中的数字和字母进行分类，传统的输入流需要读两遍，而PushBackInputStream第一读时对分类进行标记，回退至缓冲区后根据标记唤醒不同线程读取数据。

##### BufferedInputStream

缓冲流，一种处理流，内部维护**buffer缓冲区**用于缓存所有读入的字节，当缓冲区满时，才会将所有字节发送给客户端读取。

##### DataInputStream

数据输入流，一种处理流，对节点流进行封装，能在内部将字节数据转换为对应的Java基本数据类型。

##### SequenceInputStream

将多个输入流看做为一个输入流依次读取

##### 总结

- ByteArrayInputStream、FileInputStream是两种基本的**节点流**，分别从**字节数组**和**文件**中读取数据
- DataInputStream、BufferedInputStream、PushBackInputStream都是处理流，封装**节点流**并进行增强
- PipedInputStream用于多线程通信，可以与其他线程公用一个管道
- ObectInputStream用于对象的反序列化

#### OutputStream

| 重要方法                                    | 功能                                             |
| ------------------------------------------- | ------------------------------------------------ |
| public abstract int write(int b)            | 将制定的字节写到输出流，写入的字节是参数b的低8位 |
| public int write(byte b[])                  | 将制定字节数组b写到输出流                        |
| public int read(byte b[], int off, int len) | 将字节数组b的指定范围写到输出流                  |
| public void flush()                         | 刷新此输出流，强制写出所有缓冲的字节到指定位置   |
| public void close()                         | 关闭输出流，释放资源                             |

##### 总结

- 大多数类和InputStream对应
- PrintStream在OutputStream基础之上提供了增强的功能，**可以方便的输出各种类型的数据**的格式化表示，且从不抛出IOException，原理在于**写出时将各个数据类型的数据统一转换为String类型**。

### 字符流对象（Reader/Writer)

![image-20201229180514744](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20201229180514744.png)

#### Reader

| 方法                                                    | 方法功能                         |
| ------------------------------------------------------- | -------------------------------- |
| public int read(java.nio.CharBuffer target)             | 将读入的字符存入指定的字符缓存区 |
| public int read()                                       | 读取一个字符                     |
| public int read(char ;buf[])                            | 读取字符放入整个字符数组中       |
| abstract public int read(char cbuf[], int off, int len) | 将字符读入字符数组的指定范围中   |

##### CharArrayReader/StringReader

节点流，分别读入到字符数组和字符串中

##### PipedReader

用于多线程中的通信，从公用地管道中读取字符数据

##### BufferedReader

字符输入缓冲流，将字符读入后放入**缓存区**， 实现高效读取字符

##### InputStreamReader

转换流，可以实现将**字节流转换为字符流**

#### Writer

| 方法                                                      | 方法功能                          |
| --------------------------------------------------------- | --------------------------------- |
| public void write(char cbuf[])                            | 将cbuf字符数组写出到输出流        |
| abstract public void write(char cbuf[], int off, int len) | 将指定范围的cbuf数组写出到输出流  |
| public void write(String str)                             | 将str写出到输出流，本质是字符数组 |
| public void write(String str, int off, int len)           | 指定范围str                       |
| abstract public void flush()                              | 将缓冲区数据刷写至指定位置        |
| abstract public void close()                              | 关闭流，释放资源                  |

##### 总结

- CharArrayWriter/StringWriter：向字符数组、字符串中写入数据，StringWriter内部保存了StringBuffer对象，实现对象的动态增长
- PipedWriter：多线程
- BufferedWriter：缓存输出流，将写出的字符缓存起来，当缓存区满时flush刷写，减少IO次数
- PrintWriter：类似PrintStream
- OutputStreamWriter：**字符流转换为字节流**



### 字节流和字符流转换

- InputStreamReader：**字节流转换为字符流**，将字节数据转换为字符数据读入内存
- OutputStreamWriter：**字符流转换为字节流**，将字符数据转换为字节存入存储设备



## BIO/NIO/AIO 区别

### BIO(同步阻塞 I/O)

线程一直被阻塞直到I/O完成

- 面向流的IO
- 通道单向，输入输出各有一个通道

![image-20210105153542779](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210105153542779.png)

#### 痛点——阻塞：

1. BIO遇到IO阻塞后挂起线程，直到IO完成后才会唤醒线程，增加了切换线程的开销
2. 每个IO操作都需要对应的一个线程去专门处理，增加服务器压力

### NIO(同步非阻塞 I/O)

每隔一段时间就确认I/O是否完成，如果那个线程完成了就先处理那个线程

- **缓冲区**（Buffer)：用于真正存储数据
- 双向通道用于传输数据

![image-20210105153924781](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210105153924781.png)



#### 缓冲区( Buffer )

除了 boolean 类型，其他7种基本数据类型都有对应的Buffer

- 核心方法
  - put()
  - get()
- 重要属性
  - capacity：Buffer数组大小，一旦声明**无法改变**
  - limit：表示Buffer中可以操作数据的大小，limit之后的数据无法读写，limit<=capacity
  - position：当前操作位置的下表
  - mark：标记位置，reset()将置position为mark，实现**重复读取数据**
- 辅助方法
  - flip()：**读写模式切换**
  - rewind()：将position置为0，再次读取Buffer中的数据
  - clear()：清空Buffer

#### 通道(Channel)

通道可以**双向读写**

![image-20210105155720886](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210105155720886.png)

重要类都在java.nio.channels包下：

- 文件IO：FIleChannel，用于文件读写、操作
- TCP网络IO：SocketChannel（读写数据的TCP通道），ServerSocketChannel(监听客户端的连接)
- UDP网络IO：DatagramChannel（收发数据报的通道）

#### 选择器(Selector)

选择器是提升IO性能的灵魂之一，底层使用了**多路复用**机制，让选择器可以监听多个IO连接。即**选择器可以监听多个IO连接，而传统的BIO每个IO连接都需要一个线程去监听和处理**

![image-20210105162047152](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210105162047152.png)



![image-20210105162416054](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210105162416054.png)

多个IO连接可以注册到Selector上，当某个IO连接就绪时，就会通知该通道内的Selector，Selector在**下一次轮询**时会发现该IO连接已就绪，进而处理

NIO主要用于网络IO，重要方法：

- open()：打开一个选择器
- int select()：阻塞地等待就绪的通道
- int select(long timeout)： 带时限的等待
- int selectNow()：非阻塞的轮询就绪的通道

**选择键**，用于识别就绪的通道处于何种状态

Java中提供 4 种选择键：

- SelectionKey.OP_READ
- SelectionKey.OP_WRITE
- SelectionKey.OP_ACCEPT
- SelectionKey.OP_CONNECT

SelectionKey中包含的重要属性：

- channel：该选择键绑定的**通道**
- selector：轮询到该选择键的**选择器**
- readyOps：当前**就绪选择键的值**
- interestOps：该选择器对该通道**感兴趣的所有选择键**

#### IO 与 NIO 区别

- IO 基于流，即一个字节一个字节处理数据；NIO 基于块，以 Buffer 为一个整体进行处理。
- IO 是阻塞的，NIO 是非阻塞的

#### 优点与缺点

- 相比传统 BIO 模型，NIO 模型使用更少的线程就可以管理多个客户端的 Socket 连接，减少了进程上下文切换的开销
- 没有 I/O 任务的时候可以安排线程执行其他任务，能够充分利用线程资源
- 原生 NIO 包的使用比较复杂且维护成本比较大，一般使用 Netty 框架

### AIO(异步非阻塞 I/O)

每个线程I/O完成后通知主线程，然后进行后续的处理