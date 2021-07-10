# Socket

## I/O模型

一个Socket上的输入操作通常包括两个阶段：

- 等待数据从网络中到达，将数据复制到内核的某个缓冲区中。
- 把数据从内核缓冲区复制到应用进程缓冲区。

Unix下五种I/O模型：

- BIO
- NIO
- I/O复用(select和poll)
- 信号驱动式I/O(SIGIO)
- AIO

### BIO

应用进程被阻塞，直到数据从内核缓冲区中复制到进程缓冲区才恢复。

下图中recvfrom()用于接受从Socket传来的数据。

![image-20210218115904503](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210218115904503.png)

### NIO

应用进程执行系统调用后，内核返回一个错误吗。进程可以继续执行，不断地执行系统调用来获取I/O是否完成，如果完成就将数据复制到进程缓冲区中。这种方式成为**轮询**

![image-20210218120153664](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210218120153664.png)

### I/O复用

使用select或者poll等待数据，可以等待多个Socket连接完成数据传输，当某个Socket连接可用后，再通过系统调用recvfrom把数据从内核复制到进程中。

使单个进程能够处理多个I/O事件。相比多进程和多线程技术，I/O复用可以减少进行线程创建和切换的开销。

![image-20210218121643779](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210218121643779.png)

### 信号驱动I/O

进程使用sigaction系统调用，内核立即返回，进程不阻塞可以继续执行。内核在数据到达后发送SIGIO信号，进程收到后系统调用recvfrom复制数据。

![image-20210218172842739](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210218172842739.png)

### AIO

进程执行aio_read系统调用，内核立即返回，进程不阻塞继续执行。

内核会在 IO 所有操作完成后发送信号通知进程 IO 结束。

AIO 与信号驱动IO的区别在于AIO的信号是通知进程I/O完成，而信号驱动IO的信号是通知进程内核已经准备好，可以开始I/O即复制数据

### 五大I/O模型比较

- 同步I/O：将数据从内核复制至进程(第二阶段)会阻塞。
- 异步I/O：第二阶段进程不会阻塞。

同步I/O包含前四个I/O模型，它们的区别主要在等待数据到来(第一阶段)是否阻塞。

![image-20210218173630055](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210218173630055.png)

## select、poll、epoll

Linux 网络 IO 中多路复用 I/O 的具体实现，**本质上都是同步 I/O**，都需要在数据就绪后自己负责读写。

### select

```c
int select(int maxfdp1,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout)
```

- maxfdp1：监听的 socket 描述符个数
- readset：该集合内的 socket 可读时返回
- writeset：该集合内的 socket 可写时返回
- exceptset：该集合内的 socket 异常时返回
- timeout：超时时间

select 调用后监听所有它负责的 socket，当某个或多个 socket 连接可用时，他就会返回，如果都不可用 select 进程就会阻塞，直到 socket 可用或者超时。select 返回后就可以通过遍历 fdset 找到就绪的描述符。

![image-20210310122548114](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210310122548114.png)

select 几乎在所有平台上支持，有良好的跨平台性。

缺点在于

1. 单线程能够监视的 fd 数量存在限制，Linux 上一般为 1024。
2. 需要维护一个 fdset 数据结构，使得在用户空间和内核空间中传递 fdset 开销较大。
3. 每次有 socket 就绪时，都要遍历一遍 fdset，O(n) 的时间复杂度时间开销大。

### poll

```c
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

poll 的实现基本和 select 类似，只是描述 fd 集合的方式不同。poll 使用 pollfd 结构，使用链表维护 socket 描述符，因此不限制 fd 的个数。

缺点也和 select 一样。

### epoll

epoll 使用 epfd (epoll 文件描述符) 管理多个 socket 描述符，不限制个数。同时只在调用 epoll_ctl() 时向内核注册新的描述符或者是改变某个描述符的状态，只需要将描述符拷贝一次。

epoll 监听的 socket 就绪时，以回调函数的方式将就绪 socket 加入就绪链表，唤醒 epoll_wait() 去判断就绪链表是否为空。相比 select 和 poll，无需遍历 socket 集合，时间复杂度 O(1)

![image-20210310122524376](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210310122524376.png)

#### 操作模式

- LT( level trigger)：默认模式。当 epoll_wait 检测到 socket 就绪时就通知用户进程，用户进程可以不立即处理。下次调用 epoll_wait 时再次产生通知。同时支持阻塞和非阻塞读写
- ET( edge trigger)：当 epoll_wait 检测到 socket 就绪时就通知用户进程，用户进程**必须**立即处理。下次调用 epoll_wait 时不会产生通知。避免了 epoll 事件被重复触发，效率比 LT 高。只支持非阻塞读写，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

#### epoll 函数

##### epoll_create()

创建一个 epfd，创建后占用一个 fd 值，必须调用 close() 关闭，否则可能导致 fd 耗尽。

```c
int epoll_create(int size);
```

size 设置 epfd 支持的 fd 数量上限

##### epoll_ctl()

用于控制 epfd 上的事件，可以注册、修改、删除事件。

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

- epfd：epoll 文件描述符
- op：操作的选项
  - EPOLL_CTL_ADD：注册目标 socket 描述符到 epfd
  - EPOLL_CTL_MOD：修改
  - EPOLL_CTL_DEL：删除
- fd：目标 socket 描述符
- event：事件

##### epoll_wait()

监听 socket 是否就绪。

```c
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
```

- events：监听的事件集合
- maxevents：事件个数，不能超过 epfd 的size
- timeout：超时时间

### epoll 优点

- epoll 获取就绪 socket 描述符的时间复杂度为 O(1)
- 采用内存映射技术，用户空间和内核空间共享一块物理内存，里面保存了 epoll 监听的 socket 描述符，无需复制。
- epoll 基于监听事件的通知方式，只关注就绪的 socket 描述符。