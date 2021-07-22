# IO多路复用

### 1.1 阻塞与非阻塞

阻塞和非阻塞描述的是程序在等待调用结果（消息，返回值）时的状态。

- **阻塞**：阻塞调用是指调用方发出request的线程因为某种原因（如：等待系统资源）被服务方挂起，当服务方得到response后就唤醒挂起线程，并将response返回给调用方。

- **非阻塞**：非阻塞调用是指调用方发出request的线程在没有等到结果时不会被挂起，直到得到response后才返回。

阻塞和非阻塞最大的区别就是看调用方线程是否会被挂起。

### 1.2 同步和异步

同步和异步描述的是消息通信的机制。

- 同步：当一个request发送出去以后，会得到一个response，这整个过程就是一个同步调用的过程。哪怕response为空，或者response的返回特别快，但是针对这一次请求而言就是一个同步的调用。

- 异步：当一个request发送出去以后，没有得到想要的response，而是通过后面的callback、状态或者通知的方式获得结果。可以这么理解，对于异步请求分两步：

```
1. 调用方发送request没有返回对应的response
2. 服务提供方将response处理完成以后通过callback的方式通知调用方。
```

​	异步请求有一个最典型的特点：需要callback、状态或者通知的方式来告知调用方结果。

同步和异步最大的区别就是看调用方是如何观测获取返回结果的。

### 1.3 用户空间与内核空间

现在操作系统都是采用虚拟存储器，那么对32位操作系统而言，它的寻址空间（虚拟存储空间）为4G（2的32次方）。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核（kernel），保证内核的安全，操心系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。针对linux操作系统而言，将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF），供内核使用，称为内核空间，而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF），供各个进程使用，称为用户空间。

### 1.4 进程切换

为了控制进程的执行，内核必须有能力挂起正在CPU上运行的进程，并恢复以前挂起的某个进程的执行。这种行为被称为进程切换。因此可以说，任何进程都是在操作系统内核的支持下运行的，是与内核紧密相关的。

从一个进程的运行转到另一个进程上运行，这个过程中经过下面这些变化：
 \1. 保存处理机上下文，包括程序计数器和其他寄存器。
 \2. 更新PCB信息。
 \3. 把进程的PCB移入相应的队列，如就绪、在某事件阻塞等队列。
 \4. 选择另一个进程执行，并更新其PCB。
 \5. 更新内存管理的数据结构。
 \6. 恢复处理机上下文。

注：**总而言之就是很耗资源**，具体的可以参考这篇文章：[进程切换](https://link.juejin.cn/?target=http%3A%2F%2Fguojing.me%2Flinux-kernel-architecture%2Fposts%2Fprocess-switch%2F)

### 1.5 进程的阻塞

正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语(Block)，使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。`当进程进入阻塞状态，是不占用CPU资源的`。

### 1.6 文件描述符fd

文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。

文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。

### 1.7 缓存 I/O

缓存 I/O 又被称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O。在 Linux 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

**缓存 I/O 的缺点：**
 数据在传输过程中需要在应用程序地址空间和内核进行多次数据拷贝操作，这些数据拷贝操作所带来的 CPU 以及内存开销是非常大的。

## 二、为何需要IO复用

### 2.1 复用的概念

复用技术 multiplexing 并不是新技术而是一种设计思想，在通信和硬件设计中存在**频分复用、时分复用、波分复用、码分复用**等。

从本质上来说，复用就是为了解决**有限资源和过多使用者的不平衡问题**，从而实现最大的利用率，处理更多的问题。

### 2.2 理解IO复用

**I/O的含义：**在计算机领域常说的IO包括**磁盘 IO 和网络 IO**，我们所说的IO复用主要是指网络 IO ，在Linux中一切皆文件，因此网络IO也经常用文件描述符 FD 来表示。

**复用的含义：**那么这些文件描述符 FD 要复用什么呢？在网络场景中复用的就是任务处理线程，所以简单理解就是**多个IO共用1个处理线程**。

**IO复用的可行性：**

write操作的流程：write操作不是要等到对方收到消息后才会返回，而是只把消息写到本地的write缓冲区冲，然后由操作系统把缓冲区中的数据发送到对端。如果write缓冲区满，则write缓冲区就要等到缓冲区有空间空间，这个就是write操作IO操作的真正耗时之处。

read操作的流程：read操作不是直接从目标机器拉取数据，只是把消息从本地的read缓冲区取出而已。但如果read缓冲区为空，read操作就要等待read缓冲区中有数据，这个也就是read操作IO操作的真正耗时之处。

IO请求的基本操作包括read和write，由于网络交互的本质性，必然存在等待，换言之就是整个网络连接中FD的读写是交替出现的，时而可读可写，时而空闲，所以IO复用是可用实现的。

综上认为：**IO复用技术就是协调多个可释放资源的FD交替共享任务处理线程完成通信任务，实现多个fd对应1个任务处理线程的复用场景**。

### 2.3 同步阻塞（BIO）

服务端采用单线程，当accept一个请求后，在recv或send调用阻塞时，将无法accept其他请求（必须等上一个请求处recv或send完），`无法处理并发`

```c
// 伪代码描述
while(1) {
  // accept阻塞
  client_fd = accept(listen_fd)
  fds.append(client_fd)
  for (fd in fds) {
    // recv阻塞（会影响上面的accept）
    if (recv(fd)) {
      // logic
    }
  }  
}
```

这种模式只能阻塞处理单次请求，如果处理多次请求，会因为阻塞导致响应慢。当然我们可以采用多线程技术，多个线程去处理，就能同时处理多个请求了，但是随着并发量变高，线程上下文切换的耗时，及线程本身所占的空间也越来越大，最终导致无法维护高并发的系统。

### 2.4 同步非阻塞（NIO）

​	服务器端当accept一个请求后，加入fds集合，每次轮询一遍fds集合recv(非阻塞)数据，没有数据则立即返回错误，`每次轮询所有fd（包括没有发生读写事件的fd）会很浪费cpu`

```c
// 对文件描述符设置非阻塞模式
setNonblocking(listen_fd)
// 伪代码描述
while(1) {
  // accept非阻塞（cpu一直忙轮询）
  client_fd = accept(listen_fd)
  if (client_fd != null) {
    // 有人连接
    fds.append(client_fd)
  } else {
    // 无人连接
  }  
  for (fd in fds) {
    // recv非阻塞
    setNonblocking(client_fd)
    // recv 为非阻塞命令
    if (len = recv(fd) && len > 0) {
      // 有读写数据
      // logic
    } else {
       无读写数据
    }
  }  
}
```

### 2.5 IO多路复用

​	服务器端采用单线程通过select/epoll等系统调用获取fd列表，遍历有事件的fd进行accept/recv/send，使其能`支持更多的并发连接请求`

```c
fds = [listen_fd]
// 伪代码描述
while(1) {
  // 通过内核获取有读写事件发生的fd，只要有一个则返回，无则阻塞
  // 整个过程只在调用select、poll、epoll这些调用的时候才会阻塞，accept/recv是不会阻塞
  for (fd in select(fds)) {
    if (fd == listen_fd) {
        client_fd = accept(listen_fd)
        fds.append(client_fd)
    } elseif (len = recv(fd) && len != -1) { 
      // logic
    }
  }  
}
```




C10K问题

## 三、多路复用API

### 3.1 select

#### 3.1.1 api介绍

```c
#include <sys/select.h>
#include <sys/time.h>

#define FD_SETSIZE 1024
#define NFDBITS (8 * sizeof(unsigned long))
#define __FDSET_LONGS (FD_SETSIZE/NFDBITS)

// 数据结构 (bitmap)
typedef struct {
    unsigned long fds_bits[__FDSET_LONGS];
} fd_set;

// 返回值就绪描述符的数目
int select(
    int max_fd, // fd_set的最高有效位 例如 max_fd为5， fd_set 为 01011 0000 0000,则解析到01011就不会再解析了，合理设置max_fd避免的无效遍历fd_set
    fd_set *readset, //读集合 例如fd_set为 00011，则说明我们对fd为4， 5的文件描述符的读事件感兴趣
    fd_set *writeset, //写集合
    fd_set *exceptset, //异常集合
    struct timeval *timeout //超时时间
)

FD_ZERO(int fd, fd_set* fds)   // 清空集合，即将fd_set的bit所有为置0
FD_SET(int fd, fd_set* fds)    // 将给定的描述符加入集合，dbs_bits[fd] = 1;
FD_ISSET(int fd, fd_set* fds)  // 判断指定描述符是否在集合中 dbs_bits[fd] == 1?
FD_CLR(int fd, fd_set* fds)    // 将给定的描述符从文件中删除  dbs_bits[fd] == 0
```

#### 3.1.2 两个1024

1. select中存放文件描述符的数组大小FD_SETSIZE为1024
2. 进程的文件描述符上限默认是1024，**正是因为这个原因，select设计时才把数组大小设计为1024**

#### 3.1.3 使用伪代码

```c
int main() {
  fd_set read_fs, write_fs;
  struct timeval timeout;
  int max = 0;  // 用于记录最大的fd，在轮询中时刻更新即可
    
  // 建立连接，初始化最大的fd
  for(int i = 0; i < 5; i++){
      fds[i] = accept();
      if(fds[i] > max){
          max = fds[i];
      }
  }

  // 初始化比特位
  FD_ZERO(&read_fs);
  FD_ZERO(&write_fs);

  int nfds = 0; // 记录就绪的事件，可以减少遍历的次数
  while (1) {
    // 每次select都需要重新设置fd_set
    for(int i = 0; i < 5; i++){
        FD_SET(fds[i], &read_fs);
        FD_SET(fds[i], &write_fs);
    }
    // 阻塞获取
    // 每次需要把fd从用户态拷贝到内核态
    nfds = select(max + 1, &read_fd, &write_fd, NULL, &timeout);
    // 每次需要遍历所有fd，判断有无读写事件发生
    for (int i = 0; i <= max && nfds; ++i) {
      if (FD_ISSET(i, &read_fd)) {
        --nfds;
        // 这里处理read事件
      }
      if (FD_ISSET(i, &write_fd)) {
         --nfds;
        // 这里处理write事件
      }
    }
  }
```

#### 3.1.4 select缺点

- fd_set最大大小支支持1024，无法实现更大的并发
- O(N)的复杂度遍历fd_set查看读写状况
- 每次调用select都要把1024大小的fd_set在内核态和用户态之间进行拷贝，消耗较大
- 每次完成调用后，都需要重新对fd_set进行设置，操作冗余

### 3.2 poll

是select的优化版本，改进了fs_set最大只支持1024的问题，传入的感兴趣集合为数组实现。其他问题相较于select并无改进。

### 3.3 epoll

#### 3.3.1 api介绍

```c
//用户数据载体
typedef union epoll_data {
   void    *ptr;
   int      fd;
   uint32_t u32;
   uint64_t u64;
} epoll_data_t;

//fd装载入内核的载体
 struct epoll_event {
     uint32_t     events;    /* Epoll events */
     epoll_data_t data;      /* User data variable */
 };

// 数据结构
// 每一个epoll对象都有一个独立的eventpoll结构体
// 用于存放通过epoll_ctl方法向epoll对象中添加进来的事件
// epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可
struct eventpoll {
    /*红黑树的根节点，这颗树中存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root  rbr;
    /*双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件*/
    struct list_head rdlist;
};
 
// 创建epoll，size为它能管理的文件描述符数量，返回值为epoll的文件描述符
int epoll_create(int size); 
/**
 * epfd epoll的文件描述符
 * op 操作类型 EPOLL_CTL_ADD:注册新的fd到epfd、EPOLL_CTL_MOD:修改意见注册的fd的监听事件，EPOLL_CTL_DEL:从epfd中删除一个fd
 * fd 需要监听的文件描述符，议案指socket_fd
 * event 感兴趣的事件 EPOLLIN(读) 、 EPOOLOUT(写)、EPOLLET（将EPOLL设为边缘触发模式）.......
 */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
/**
 * epfd epoll的文件描述符
 * event 指向一个epoll_event结构的数组，当函数返回时，会把就绪的状态的数据拷贝到数组中
 * maxevents 数组的最高有效位
 * timeoiut 0 非阻塞 >0 超时阻塞 -1 阻塞调用
 * 返回值 本次就绪的fd个数
 */
int epoll_wait(int epfd, struct epoll_event *events,
                 int maxevents, int timeout);
```

#### 3.3.2 使用伪代码

```c
int main(int argc, char* argv[])
{
   /*
   * 在这里进行一些初始化的操作，
   * 比如初始化数据和socket等。
   */

    // 内核中创建ep对象
    epfd=epoll_create(256);
    // 需要监听的socket放到ep中
    epoll_ctl(epfd,EPOLL_CTL_ADD,listenfd,&ev);
 
    while(1) {
      // 阻塞获取
      nfds = epoll_wait(epfd,events,20,0);
      for(i=0;i<nfds;++i) {
          if(events[i].data.fd==listenfd) {
              // 这里处理accept事件
              connfd = accept(listenfd);
              // 接收新连接写到内核对象中
              epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev);
          } else if (events[i].events&EPOLLIN) {
              // 这里处理read事件
              read(sockfd, BUF, MAXLINE);
              //读完后准备写
              epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);
          } else if(events[i].events&EPOLLOUT) {
              // 这里处理write事件
              write(sockfd, BUF, n);
              //写完后准备读
              epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);
          }
      }
    }
    return 0;
}
```

#### 3.3.3 工作模式

epoll有EPOLLLT和EPOLLET两种触发模式，**LT是默认的模式**（水平触发），ET是“高速”模式（边缘触发）。

LT模式下，只要这个fd还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作

ET模式下，它只会提示一次，直到下次再有数据流入之前都不会再提示了，无论fd中是否还有数据可读。所以在ET模式下，read一个fd的时候一定要把它的buffer读完。

 **LT的读写操作**

LT的读操作，有read事件就读，读多读少都行，反正下次还可以继续读。但是写操作，一般来说，写缓冲区都很难被填满，肯能导致write事件一直是就绪的状态，加入这个时候对fd的读事件感兴趣，则一直会通知读事件。所以需要确保没有要发送的数据的时候，不要注册读事件，不然会可能会一直提醒可写。

**ET的读写操作**

fd可读则返回可读事件，若开发者没有把所有数据读取完毕，epoll不会再次通知read事件，也就是说如果没有全部读取所有数据，那么导致epoll不会再通知该socket的read事件。若发送缓冲区未满，epoll通知write事件，直到开发者填满发送缓冲区，epoll才会在下次发送缓冲区由满变成未满时通知write事件，所以其实并不会向LT一样频繁通知write事件。ET模式下只有socket的状态发生变化时才会通知，也就是读取缓冲区由无数据到有数据时通知read事件，发送缓冲区由满变成未满通知write事件。

**LT可写频繁提醒如何解决**

写完数据赶紧移除读事件，除非是遇到了errno为EAGAIN（在文件fd处于非阻塞模式下，缓冲区已满，请稍后再试），才注册写事件，等待缓冲区可写再继续写。

### 3.3 epoll的优势

- 对fd数量没有限制(当然这个在poll也被解决了)
- 抛弃了bitmap数组实现了新的结构来存储多种事件类型
- 无需重复拷贝fd 随用随加 随弃随删
- 采用事件驱动避免轮询查看可读写事件
- 红黑树查找就绪socket，比select遍历效率高

### 3.4 推荐阅读

https://juejin.cn/post/6844904006301843464#heading-18   从硬件入手深入理解epoll 的本质