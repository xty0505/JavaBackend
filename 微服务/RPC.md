# RPC 概念

远程过程调用 （RPC）核心要解决的问题是如何调用另一个主机上的方法，就像在本地调用一样。如下图

![image-20210817154917649](../pic/image-20210817154917649.png)

那么需要考虑的问题有两个：

1. 序列化和反序列化。请求端将对象序列化成二进制序列，服务端将二进制序列反序列化成对象，保证两方都能正确处理和发送数据。
2. 网络传输。RPC 需要可靠传输，因此一般建立在 TCP 之上的。



## Thrift RPC

### Thrift 简介

Thrift 是一套包含序列化功能和支持服务通信的 RPC 框架，主要包含三大部分:**代码生成**、**序列化框架**、**RPC框架**，大致相当于protoc + protobuffer + gRPC，并且支持大量语言，保证常用功能在跨语言间功能一致。

#### 特点

- 基于二进制的高性能序列化和反序列化框架
- 基于 NIO 的网络通信
- 相对简单的服务调用模型
- 使用 IDL 支持跨语言调用

#### 架构

![image-20210817155715690](../pic/image-20210817155715690.png)

thrift 的网络层次

```bash
  +-------------------------------------------+
  | Server                                    |
  | (single-threaded, event-driven etc)       |
  +-------------------------------------------+
  | Processor                                 |
  | (compiler generated)                      |
  +-------------------------------------------+
  | Protocol                                  |
  | (JSON, compact etc)                       |
  +-------------------------------------------+
  | Transport                                 |
  | (raw TCP, HTTP etc)                       |
  +-------------------------------------------+
```

- Transport：传输层提供网络 IO 读写的抽象接口，将底层 IO 读写和 Thrfit 其他组件解耦。

Transport 提供以下接口：

```
open
close
read
write
flush
```

对于 Server 端，还提供了 ServerTransport 接口用于监听和接收客户端的请求

```
open
listen
accept
close
```



- Protocol：

Procotol 定义编码格式，实现序列化和反序列化。主要提供的协议有 binary/comact/json。

流式处理，无需消息被完整接收就能开始解码，减少了消息处理的时延。



- Processor：

Processor 从 protocol 中读取方法名和参数，调用 Server 上的实现，将结果写到 protocol。封装了读取输入流、写入输出流的能力。



### IDL 和代码生成

Thrift 采用IDL（Interface Definition Language）来定义通用的服务接口，然后通过Thrift提供的编译器，可以将服务接口编译成不同语言编写的代码，通过这个方式来实现跨语言的功能。

### Thrift 序列化协议

Thrift 序列化协议主要有 binary/compact/json。下面主要介绍 binary 和 compact。二进制编码通常使用 TLV 编码实现，即 Tag，Length，Value组成的结构体，几乎可以描述任意数据类型。TLV 的 Value 也可以是 TLV，这种嵌套的特性可以用来包装协议的实现。bianry 和 compact 都是用 TLV 变种实现的，两者不同在于对整数类型的处理。

#### binary 序列化

##### 简单数据类型为定长编码：类型标识+编号+数据值

![image-20210817172539815](../pic/image-20210817172539815.png)

| 简单数据类型 |               |                |                                   |
| ------------ | ------------- | -------------- | --------------------------------- |
| 数据类型     | 类型标识(8位) | field_id(16位) | field_value                       |
| bool         | 2             |                | 一个字节的值（true：1，false：0） |
| byte         | 3             |                | 一个字节值                        |
| double       | 4             |                | 八个字节值                        |
| i16          | 6             |                | 两个字节值                        |
| i32          | 8             |                | 四个字节值                        |
| i64          | 10            |                | 八个字节值                        |

##### 复合数据类型

复合数据类型的 field_value 不同：

list 和 set：类型标识（8）+成员个数（32）+成员值

![image-20210817172955582](../pic/image-20210817172955582.png)

map：k类型（8）+v类型（8）+成员个数（32）+kv值

![image-20210817173039499](../pic/image-20210817173039499.png)

| 复合数据类型 |                 |                  |                                           |
| ------------ | --------------- | ---------------- | ----------------------------------------- |
| 数据类型     | 类型标识（8位） | field_id（16位） | field_value                               |
| string       | 11              |                  | size(32) + field_value                    |
| struct       | 12              |                  | 嵌套数据+一个字节停止符（0）              |
| map          | 13              |                  | k类型（8）+v类型（8）+成员个数（32）+kv值 |
| set          | 14              |                  | 类型标识（8）+成员个数（32）+成员值       |
| list         | 15              |                  | 类型标识（8）+成员个数（32）+成员值       |

#### 

#### compact 序列化

compact 序列化方式不同于 binary 的地方在于整数类型采用 zigzag 和 varint 压缩编码实现。

1. varint 编码：不定长无符号整数编码，每个字节按低位到高位，我们只使用低7位，最高的一位作为一个标志位（msb）

   - 1：下一个字节也是该数字的一部分
   - 0：下一个字节不是该数字的一部分

   因此小数字占有字节更少，大数字占有字节更多，但大部分序列化的数字都是小数字，所以整体压缩效果明显。比如300(i32)，Binary序列化下需要4个字节，采用varint编码只需要两个字节：**1**010 1100 **0**000 0010。

2. zigzag 编码：varint 解决了无符号编码问题，假设有符号数也直接采用varint编码，因为负数最高位是1，比如i32就都会使用5个字节了，反而使用更多字节，为了解决有符号负数问题，先采用 zigzag 编码将有符号数映射到无符号数上。

   ```bash
   以i32为例,设n为i32值原始值，m为映射无符号值
   
   1) if n >= 0, m=2*n
   
   2) if n < 0, m=2*|n| - 1
   
   3) 采用位运算实现: m=(n << 1) ^ (n >> 31)
   ```

compact 序列化和 binary 基本一致，只是在 i16，i32，i64 三种类型使用 zigzga + varint 编码实现，复合类型的长度只用 varint 编码。



#### 可改进的地方

- 类型+编号就占了 3 个字节，大部分数据就是几个字节
- 除了整形数据，其他数据是否能够压缩

1. Thrift 官方没有给出解决方案，grpc 的序列化框架 Protobuf 将类型和编号合并了
2. Thrift 官方提供了 ZlibTransport 在传输层进行压缩。也可以在业务层对大结构体的数据实现类似 snappy/lz4 等压缩。

### 

### Thrift 传输协议

thrift 存在多个版本，apache thrift 和 fb thrift。前者采用的传输协议是无头协议，后者是 THeaderProtocol（带头部协议），比较好实现异步。

#### apache thrift-BinaryProtocol 

分为严格模式（默认）和非严格模式。严格模式会带上版本 Version 信息，非严格模式没有版本。

通讯消息类型（type Id）主要有四种：

- CALL：值为1，请求
- REPLY：值为2，响应
- EXCEPTION：3， 异常
- ONEWAY：4，无返回值请求

严格模式：version（含typeId，4字节）+method name（4）+seq_id（4）+payload+结束标记

![image-20210819143415650](../pic/image-20210819143415650.png)

非严格模式：method name（4）+调用类型（1）+seq_id（4）+payload+结束标记

![image-20210819143948902](../pic/image-20210819143948902.png)



#### apache thrift-CompactProtocol

主要对 BinaryProtocol 进行压缩



#### fbthrift-THeaderTransport 和 bytedance-TTHeader 协议

随着微服务的发展，出现了许多服务治理的技术，如服务发现/调用链/机房调度隔离/权限控制等。需要 RPC 协议中添加相应信息，原先的传输协议无法支持。FB 搞了一套 fbthrift，协议部分新增了一个 THeaderProtocol（带头部的协议）。

Bytedance 基于 THeaderProtocol 开发了一套 TTHeader 协议。主要对一些字段进行裁剪，优化了一些解析性能。

THeaderTransport 协议编码：

```
 0 1 2 3 4 5 6 7 8 9 a b c d e f 0 1 2 3 4 5 6 7 8 9 a b c d e f
+----------------------------------------------------------------+
| 0|                          LENGTH                             |
+----------------------------------------------------------------+
| 0|       HEADER MAGIC          |            FLAGS              |
+----------------------------------------------------------------+
|                         SEQUENCE NUMBER                        |
+----------------------------------------------------------------+
| 0|     Header Size(/32)        | ...
+---------------------------------
                  Header is of variable size:

                   (and starts at offset 14)
+----------------------------------------------------------------+
|         PROTOCOL ID  (varint)  |   NUM TRANSFORMS (varint)     |
+----------------------------------------------------------------+
|      TRANSFORM 0 ID (varint)   |        TRANSFORM 0 DATA ...
+----------------------------------------------------------------+
|         ...                              ...                   |
+----------------------------------------------------------------+
|        INFO 0 ID (varint)      |       INFO 0  DATA ...
+----------------------------------------------------------------+
|         ...                              ...                   |
+----------------------------------------------------------------+
|                                                                |
|                              PAYLOAD                           |
|                                                                |
+----------------------------------------------------------------+
```

- LENGTH：4字节，数据包剩余部分的字节大小，不包括自身
- HEADER MAGIC：2字节，值为0x0FFF，用于表示协议为 THeaderTransport
- FLAGS：2字节，预留
- SEQUENCE NUMBER：4字节，数据包序列号
- HEADER SIZE：2字节，等于 Header 部分字节/4。从第14个字节开始算，直到PLAYLOAD
- PROTOCOL ID：0为 bianry，1为 compact
- NUM TRANSFORMS：表示 TRANSTORMS 数量
- TRANSFORM ID：压缩方式，具体值如下
- INFO ID：具体如下

Header 部分字节必须是4的倍数，不足用 0x00 对齐。

Transform IDs：

```bash
ZLIB_TRANSFORM 0x01 - No data for this.  Use zlib to (de)compress the
                      data.
SNAPPY_TRANSFORM  0x03  - No data for this.  Use snappy to (de)compress the
                      data.
```

Info IDs：

```bash
INFO_KEYVALUE 0x01 - varint32 number of headers.
                   - key/value pairs of varstrings (varint16 length plus
                     no-trailing-null string).
```

