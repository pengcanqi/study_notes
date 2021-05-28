

## 缓存行

为了高效地存取缓存, 不是简单随意地将单条数据写入缓存的. 缓存是由缓存行组成的, 典型的一行是64字节. 读者可以通过下面的shell命令,查看机器的缓存行是多大

```
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
```

CPU存取缓存都是按行为最小单位操作的. . 一个Java long型占8字节, 所以从一条缓存行上你可以获取到8个long型变量. 所以如果你访问一个long型数组, 当有一个long被加载到cache中, 你将无消耗地加载了另外7个. 所以你可以非常快地按行遍历数组.但是如果按列遍历数组的话，速率可能就会分比较慢。

## 伪共享

当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。如下图中的一个例子：

```java
class foo {
    long x;
    long y; 
};

foo f = new foo();

/* The two following functions are running concurrently: */

void inc_x()
{
    int s = 0;
    for (int i = 0; i < 1000000; ++i)
       ++f.x;
}

void inc_y()
{
    for (int i = 0; i < 1000000; ++i)
        ++f.y;
}
```

在上述代码中，假设inc_x的执行时间是t，串行调用inc_x和inc_y，则需要的时间为2t。如果并行调用inc_x和inc_y，如果没有了解缓存行，我们可能会认为耗时远小于t，而实际的结果则是远远大于2t的。假设cpu0执行函数inc_x，cpu1执行函数inc_y，由于变量x和变量y位于同一个缓存行中，cpu0每次执行函数inc_x，会让cpu1中对应的缓存行失效，cpu1执行函数inc_y，会让cpu0中对应的缓存行失效，相当于每次的读写都要经过主存而没有经过缓存，所以会导致耗时很高。那么要如何避免伪共享呢？避免伪共享的主要思路就是让不相干的变量不要出现在同一个缓存行中。在上述例子中，我们有如下解决方法

- x和y变量之间加七个 long 类型，这样x和y就位于不同的缓存行了
- 在使用java8提供的注解@sun.misc.Contended，在JVM启动参数加上`-XX:-RestrictContended`，@sun.misc.Contended 是 Java 8 新增的一个注解，对某字段加上该注解则表示该字段会单独占用一个**缓存行**（Cache Line）

## Cache写策略

### Cache 写

当系统将数据写入缓存时，它也必须在某个时刻将该缓存写入后备存储。写后备存储的时间是由写策略控制的。有两种基本的写策略:

- write through:同步写缓存和后备存储。
- write back（也称作延迟写）:最初，只写缓存。直到修改后的缓存内容要被另一个缓存行替换出去时，才写入后备存储。

回写缓存实现起来更复杂，因为它需要跟踪自己的哪些位置已经修改，并将它们标记为脏的，以便以后写入后备存储。只有当这些位置的数据从缓存中被逐出时，它们才被写回后备存储，这种特性称为惰性写。由于这个原因，一个read miss操作在write back策略中，通常需要两次内存访问：

- 将脏数据从缓存写回后备存储
- 从后备存储中加载read miss操作需要的数据到缓存

### Write Miss写缺失处理方式

对于写操作，存在待写入的数据不在缓存中的情况，这时有两种处理方式：	

- Write allocate：将写入位置读入缓存，然后采用write-hit（缓存命中写入）操作。写缺失操作与读缺失操作类似。
- No-write allocate：不将写入位置读入缓存，而是直接将数据写入存储。这种方式下，只有读操作会被缓存。

无论是Write-through还是Write-back都可以使用写缺失的两种方式之一。只是通常Write-back采用Write allocate方式，而Write-through采用No-write allocate方式；因为多次写入同一缓存时，Write allocate配合Write-back可以提升性能；而对于Write-through则没有帮助。下图为处理流程：



​																							*Write back with Write Allocate*

![Write-back_with_write-allocation.png](https://github.com/pengcanqi/study_note_image/blob/main/%E5%A4%9A%E7%BA%BF%E7%A8%8B/Write-back_with_write-allocation.png?raw=true)

​																						*Write through with NO-Write Allocate*

![Write-through_with_no-write-allocation.png](https://github.com/pengcanqi/study_note_image/blob/main/%E5%A4%9A%E7%BA%BF%E7%A8%8B/Write-through_with_no-write-allocation.png?raw=true)



## 缓存一致性

在计算机体系结构中，缓存一致性是指最终存储在多个本地缓存中的共享资源数据的一致性。

在下图中，考虑两个客户端都有上一次读取的特定内存块的缓存副本。假设底部的客户端更新/更改了该内存块，顶部的客户端可能会留下一个无效的内存缓存，而没有任何更改通知。缓存一致性旨在通过维护多个缓存中的数据值的一致性视图来管理此类冲突。

![Cache_Coherency_Generic.png](https://github.com/pengcanqi/study_note_image/blob/main/%E5%A4%9A%E7%BA%BF%E7%A8%8B/Cache_Coherency_Generic.png?raw=true)

### 概述

在一个共享内存多处理器系统中，每个处理器都有一个单独的缓存内存，共享数据可能有多个副本。当数据的一个副本发生更改时，其他副本必须对此变更做出响应。缓存一致性是确保共享操作数(数据)值的变化及时地在整个系统中传播的规则。

缓存一致性要求如下:

- 写传播

  对任何缓存中数据的更改都必须传播到对等缓存中的其他副本。

- 事务串行化

  对单个内存位置的读/写必须被所有处理器以相同的顺序看到。

### 定义

缓存一致性定义了对单个地址位置的读写行为。

在多处理器系统中，考虑一个以上的处理器缓存了内存位置x的一个副本。要实现缓存一致性，必须满足以下条件:

- 处理器P对位置X进行先写后读操作，在写和读之间没有其他处理器写位置X，则P应读到它上一次写操作写入X的值；
- 处理器P对位置X进行写操作，然后处理器Q对位置X进行读操作，如果在写和读之间没有其他处理器对位置X进行写操作，且读与写间隔足够长的时间，则Q应读取到P上次写操作写入X的值；

上述条件满足了缓存一致性所需的写传播标准。但是，它们是不够的，因为它们不满足事务序列化条件。为了更好地说明这一点，考虑以下示例:

- 一个多处理器系统由四个处理器组成——P1、P2、P3和P4，它们都包含共享变量S的缓存副本，该变量S的初始值为0。处理器P1将S的值(在它的缓存副本中)修改为10，然后处理器P2将S的值在它自己的缓存副本中修改为20。如果我们确保只写传播，那么P3和P4肯定会看到P1和P2对S所做的更改。然而,P3可能看到的操作顺序是p1->p2,因此返回20。另一方面P4可能看到的顺序是P2->P1，因此返回10。


因此，为了满足事务串行化，从而实现缓存一致性，必须满足以下条件以及本节中提到的前两个条件:

```
对相同位置的写入必须进行排序。换句话说，如果位置X从任意两个处理器接收到两个不同的值A和B，按照这个顺序，处理器永远不能把位置X读成B然后再把它读成A
```

## MESI

现代计算机都是多核cpu，cpu需要和内存交互，但内存相对cpu的速度实在太慢，于是cpu和内存之间还有cache层，每个cpu都有属于自己的cache，cache由cache line组成，每个cache line 64位（根据不同架构，也可能是32位或128位），每个cache line知道自己对应什么范围的物理内存地址，当cpu需要读取某一个内存地址的值时，它会把内存地址传递给一级cache，一级cache会检查它是否有这个内存地址对应的cache line。如果没有，它会以cache line为单位从内存加载数据，是的，一次加载整个cache line，这是基于这样一个假设：内存访问倾向于本地化（localized），如果我们当前需要某个地址的数据，那么很可能我们马上要访问它的邻近地址。

![最原始的cpu架构.jpg](https://github.com/pengcanqi/study_note_image/blob/main/%E5%A4%9A%E7%BA%BF%E7%A8%8B/%E6%9C%80%E5%8E%9F%E5%A7%8B%E7%9A%84cpu%E6%9E%B6%E6%9E%84.jpg?raw=true)

由于每个cpu独立工作，那就会有一个显著的问题：多个cache与内存之间的数据同步该怎么做？缓存一致性协议就是要解决这个问题，协议有多种，可以分为两类：“窥探（snooping）”协议和“基于目录的（directory-based）”协议，本文所讲述的MESI协议属于一种“窥探协议“。

### 窥探协议的基本思想

所有cache与内存，cache与cache（是的，cache之间也会有数据传输）之间的传输都发生在一条共享的总线上，而所有的cpu都能看到这条总线，同一个指令周期中，只有一个cache可以读写内存，所有的内存访问都要经过仲裁（arbitrate）。

窥探协议的思想是，cahce不但与内存通信时和总线打交道，而且它会不停地窥探总线上发生的数据交换，跟踪其他cache在做什么。所以当一个cache代表它所属的cpu去读写内存时，其它cpu都会得到通知，它们以此来使自己的cache保持同步。

### MESI协议的工作方式

”MESI“该名称来自4个状态的首字母的缩写，协议中最重要的内容有两部分：cache line的状态以及消息通知机制。

**cache line的状态**有4个：

- Invalid，表明该cache line已失效，它要么已经不在cache中，要么它的内容已经过时。处于该状态下的cache line等同于它从来没被加载到cache中。
- Shared，表明该cache line是内存中某一段数据的拷贝，处于该状态下的cache line只能被cpu读取，不能写入，因为此时还没有独占。不同cpu的cache line都可以拥有这段内存数据的拷贝。
- Exclusive，和 Shared 状态一样，表明该cache line是内存中某一段数据的拷贝。区别在于，该cache line独占该内存地址，其他处理器的cache line不能同时持有它，如果其他处理器原本也持有同一cache line，那么它会马上变成“Invalid”状态。
- Modified，表明该cache line已经被修改，cache line只有处于Exclusive状态才能被修改。此外，已修改cache line如果被丢弃或标记为Invalid，那么先要把它的内容回写到内存中。

我们发现，cpu有读取数据的动作，有独占的动作，有独占后更新数据的动作，有更新数据之后回写内存的动作，根据”窥探协议“的规范，每个动作都需要通知到其他cpu，于是有以下的**消息机制**：

- Read，cpu发起读取数据请求，请求中包含需要读取的数据地址。
- Read Response，作为Read消息的响应，该消息可能是内存响应的，也可能是某cpu响应的(比如该地址在某cpu cache Line中为Modified状态，则该cpu必须返回该地址的最新数据)。
- Invalidate，cpu发起”我要独占一个cache line，其他cpu请失效对应的cache line“的消息，消息中包含了内存地址，所有的其它cpu需要将对应cache line置为Invalid状态。
- Invalidate ACK，收到Invalidate消息的cpu在将对应cache line置为Invalid后，返回Invalid ACK。
- Read Invalidate，相当于Read消息+Invalidate消息，即取得数据并且独占它，将收到一个Read Response和所有其它cpu的Invalidate ACK。
- Write back，写回消息，即将状态为Modified的cache line写回到内存，通常在该行将被替换时使用。现代cpu cache基本都采用”写回(Write Back)”而非”直写(Write Through)”的方式。

结合cache line状态以及消息机制，我们来看看cpu之间是如何协作的。为了简化，假设我们有个四核cpu系统，每个cpu只有一个cache line，每个cache line大小为1个字节，内存地址空间一共两个字节的数据，地址分别为0x0和0x8，有如下操作序列:

![cache line时序变化图.jpg](https://github.com/pengcanqi/study_note_image/blob/main/%E5%A4%9A%E7%BA%BF%E7%A8%8B/cache%20line%E6%97%B6%E5%BA%8F%E5%8F%98%E5%8C%96%E5%9B%BE.jpg?raw=true)

1. 初始状态，4个cpu的cache line都为Invalid状态（黑色表示Invalid）。
2. cpu0发送Read消息，加载0x0的数据，数据从内存返回，cache line状态变为Shared。
3. cpu3发送Read消息，加载0x0的数据，数据从内存返回，cache line状态变为Shared。
4. cpu0发送Read消息，加载0x8的数据，导致cache line被替换，由于之前状态为Shared，即与内存中数据一致，可直接覆盖，而无需回写。
5. cpu2发送Read Invalidate消息，从内存返回最新数据，cpu3返回Invalidate ACK，并将状态变为Invalid，cpu2获得独占权，状态变为Exclusive。
6. cpu2修改cache line中的数据，cache line状态为Modified，同时内存中0x0的数据过期。
7. cpu1 对地址0x0的数据执行原子(atomic)递增操作，发出Read Invalidate消息，cpu2将返回Read Response(而不是内存)，包含最新数据，并返回Invalidate ACK，同时cache line状态变为Invalid。最后cpu1获得独占权，cache line状态变为Modified，数据为递增后的数据，而内存中的数据仍然为过期状态。
8. cpu1 加载0x8的数据，此时cache line将被替换，由于之前状态为Modified，因此需要先执行写回操作，此时内存中0x0的数据得以更新。

这就是缓存一致性协议，一个状态机，仅此而已。因为该协议的存在，每个cpu就可以放心操作属于自己的cache，而不需要担心本地cache中的数据会不会已经被其他cpu修改了之类的烦心事。但到目前为止，cpu并不满足，觉得在缓存一致性协议的框架下工作性能不够高，但这并不是协议的问题，协议本身逻辑很严谨，没毛病（同类协议之间的优劣那是另外一回事）。性能差在哪里？假如某数据存在于其他cpu的cache中，那自己每次需要修改数据时，都需要发送Read Invalidate消息，除了等待最新数据的返回，还需要等待其他cpu的Invalidate ACK才能继续执行其他指令，这是一种同步行为，cpu可忍不了。

### 性能优化之旅——Store Buffer

当cpu需要的数据在其他cpu的cache内时，需要请求，并且等待响应，这显然是一个同步行为，优化的方案也很明显，采用异步。思路大概是在cpu和cache之间加一个store buffer，cpu可以先将数据写到store buffer，同时给其他cpu发送消息，然后继续做其它事情，等到收到其它cpu发过来的响应消息，再将数据从store buffer移到cache line。

![增加storebuffer.jpg](https://github.com/pengcanqi/study_note_image/blob/main/%E5%A4%9A%E7%BA%BF%E7%A8%8B/%E5%A2%9E%E5%8A%A0storebuffer.jpg?raw=true)

该方案听起来不错！但逻辑上有漏洞，需要细化，我们来看几个漏洞。比如有如下代码:

```java
a = 1;
b = a + 1;
assert(b == 2);
```

初始状态下，假设a，b值都为0，并且a存在cpu1的cache line中(Shared状态)，可能出现如下操作序列:

![cache line时序变化图.jpg](https://github.com/pengcanqi/study_note_image/blob/main/%E5%A4%9A%E7%BA%BF%E7%A8%8B/cache%20line%E6%97%B6%E5%BA%8F%E5%8F%98%E5%8C%96%E5%9B%BE.jpg?raw=true)

1. cpu0 要写入a，将a=1写入store buffer，并发出Read Invalidate消息，继续其他指令。
2. cpu1 收到Read Invalidate，返回Read Response(包含a=0的cache line)和Invalidate ACK，cpu0 收到Read Response，更新cache line(a=0)。
3. cpu0 开始执行b=a+1，此时cache line中还没有加载b，于是发出Read Invalidate消息，从内存加载b=0，同时cache line中已有a=0，于是得到b=1，状态为Modified状态。
4. cpu0 得到 b=1，断言失败。
5. cpu0 将store buffer中的a=1推送到cache line，然而为时已晚。

造成这个问题的根源在于对同一个cpu存在对a的两份拷贝，一份在cache，一份在store buffer，而cpu计算b=a+1时，a和b的值都来自cache。仿佛代码的执行顺序变成了这个样子：

```java
b = a + 1;
a = 1;
assert(b == 2);
```

这个问题需要优化。

> 思考题：以上操作序列只是其中一种可能，具体响应（也就是操作2）什么时候到来是不确定的，那么假如响应在cpu0计算“b=a+1”之后到来会发生什么？

### 性能优化之旅——Store **Forwarding**

store buffer可能导致破坏程序顺序的问题，硬件工程师在store buffer的基础上，又实现了”store forwarding”技术: cpu可以直接从store buffer中加载数据，即支持将cpu存入store buffer的数据传递(forwarding)给后续的加载操作，而不经由cache。

![Store Forwarding.jpg](https://github.com/pengcanqi/study_note_image/blob/main/%E5%A4%9A%E7%BA%BF%E7%A8%8B/Store%20Forwarding.jpg?raw=true)

虽然现在解决了同一个cpu读写数据的问题，但还是有漏洞，来看看并发程序:

```java
void foo() {
    a = 1;
    b = 1;
}
void bar() {
    while (b == 0) continue;
    assert(a == 1)
}
```

初始状态下，假设a，b值都为0，a存在于cpu1的cache中，b存在于cpu0的cache中，均为Exclusive状态，cpu0执行foo函数，cpu1执行bar函数，上面代码的预期断言为真。那么来看下执行序列:

1. cpu1执行while(b == 0)，由于cpu1的Cache中没有b，发出Read b消息
2. cpu0执行a=1，由于cpu0的cache中没有a，因此它将a(当前值1)写入到store buffer并发出Read Invalidate a消息
3. cpu0执行b=1，由于b已经存在在cache中，且为Exclusive状态，因此可直接执行写入
4. cpu0收到Read b消息，将cache中的b(当前值1)返回给cpu1，将b写回到内存，并将cache Line状态改为Shared
5. cpu1收到包含b的cache line，结束while (b == 0)循环
6. cpu1执行assert(a == 1)，由于此时cpu1 cache line中的a仍然为0并且有效(Exclusive)，断言失败
7. cpu1收到Read Invalidate a消息，返回包含a的cache line，并将本地包含a的cache line置为Invalid，然而已经为时已晚。
8. cpu0收到cpu1传过来的cache line，然后将store buffer中的a(当前值1)刷新到cache line

出现这个问题的原因在于cpu不知道a, b之间的数据依赖，cpu0对a的写入需要和其他cpu通信，因此有延迟，而对b的写入直接修改本地cache就行，因此b比a先在cache中生效，导致cpu1读到b=1时，a还存在于store buffer中。从代码的角度来看，foo函数似乎变成了这个样子：

```java
void foo() {
    b = 1;
    a = 1;
}
```

foo函数的代码，即使是store forwarding也阻止不了它被cpu“重排”，虽然这并没有影响foo函数的正确性，但会影响到所有依赖foo函数赋值顺序的线程。看来还要继续优化！

### 性能优化之旅——写屏障指令

到目前为止，我们发现了”指令重排“的其中一个本质[[1\]](https://zhuanlan.zhihu.com/p/125549632#ref_1)，cpu为了优化指令的执行效率，引入了store buffer（forwarding），而又因此导致了指令执行顺序的变化。要保证这种顺序一致性，靠硬件是优化不了了，需要在软件层面支持，没错，cpu提供了写屏障（write memory barrier）指令，Linux操作系统将写屏障指令封装成了smp_wmb()函数，cpu执行smp_mb()的思路是，会先把当前store buffer中的数据刷到cache之后，再执行屏障后的“写入操作”，该思路有两种实现方式: 一是简单地刷store buffer，但如果此时远程cache line没有返回，则需要等待，二是将当前store buffer中的条目打标，然后将屏障后的“写入操作”也写到store buffer中，cpu继续干其他的事，当被打标的条目全部刷到cache line，之后再刷后面的条目（具体实现非本文目标），以第二种实现逻辑为例，我们看看以下代码执行过程：

```java
void foo() {
    a = 1;
    smp_wmb()
    b = 1;
}
void bar() {
    while (b == 0) continue;
    assert(a == 1)
}
```

1. cpu1执行while(b == 0)，由于cpu1的cache中没有b，发出Read b消息。
2. cpu0执行a=1，由于cpu0的cache中没有a，因此它将a(当前值1)写入到store buffer并发出Read Invalidate a消息。
3. cpu0看到smp_wmb()内存屏障，它会标记当前store buffer中的所有条目(即a=1被标记)。
4. cpu0执行b=1，尽管b已经存在在cache中(Exclusive)，但是由于store buffer中还存在被标记的条目，因此b不能直接写入，只能先写入store buffer中。
5. cpu0收到Read b消息，将cache中的b(当前值0)返回给cpu1，将b写回到内存，并将cache line状态改为Shared。
6. cpu1收到包含b的cache line，继续while (b == 0)循环。
7. cpu1收到Read Invalidate a消息，返回包含a的cache line，并将本地的cache line置为Invalid。
8. cpu0收到cpu1传过来的包含a的cache line，然后将store buffer中的a(当前值1)刷新到cache line，并且将cache line状态置为Modified。
9. 由于cpu0的store buffer中被标记的条目已经全部刷新到cache，此时cpu0可以尝试将store buffer中的b=1刷新到cache，但是由于包含B的cache line已经不是Exclusive而是Shared，因此需要先发Invalidate b消息。
10. cpu1收到Invalidate b消息，将包含b的cache line置为Invalid，返回Invalidate ACK。
11. cpu1继续执行while(b == 0)，此时b已经不在cache中，因此发出Read消息。
12. cpu0收到Invalidate ACK，将store buffer中的b=1写入Cache。
13. cpu0收到Read消息，返回包含b新值的cache line。
14. cpu1收到包含b的cache line，可以继续执行while(b == 0)，终止循环，然后执行assert(a == 1)，此时a不在其cache中，因此发出Read消息。
15. cpu0收到Read消息，返回包含a新值的cache line。
16. cpu1收到包含a的cache line，断言为真。

### 性能优化之旅——**Invalid Queue**

引入了store buffer，再辅以store forwarding，写屏障，看起来好像可以自洽了，然而还有一个问题没有考虑: store buffer的大小是有限的，所有的写入操作发生cache missing（数据不再本地）都会使用store buffer，特别是出现内存屏障时，后续的所有写入操作(不管是否cache missing)都会挤压在store buffer中(直到store buffer中屏障前的条目处理完)，因此store buffer很容易会满，当store buffer满了之后，cpu还是会卡在等对应的Invalidate ACK以处理store buffer中的条目。因此还是要回到Invalidate ACK中来，Invalidate ACK耗时的主要原因是cpu要先将对应的cache line置为Invalid后再返回Invalidate ACK，一个很忙的cpu可能会导致其它cpu都在等它回Invalidate ACK。解决思路还是化同步为异步: cpu不必要处理了cache line之后才回Invalidate ACK，而是可以先将Invalid消息放到某个请求队列Invalid Queue，然后就返回Invalidate ACK。CPU可以后续再处理Invalid Queue中的消息，大幅度降低Invalidate ACK响应时间。此时的CPU Cache结构图如下：

![Invalid Queue.jpg](https://github.com/pengcanqi/study_note_image/blob/main/%E5%A4%9A%E7%BA%BF%E7%A8%8B/Invalid%20Queue.jpg?raw=true)

加入了invalid queue之后，cpu在处理任何cache line的MSEI状态前，都必须先看invalid queue中是否有该cache line的Invalid消息没有处理。另外，它也再一次破坏了内存的一致性。请看代码：

```java
void foo() {
    a = 1;
    smp_wmb()
    b = 1;
}
void bar() {
    while (b == 0) continue;
    assert(a == 1)
}
```

仍然假设a, b的初始值为0，a在cpu0，cpu1中均为Shared状态，b在cpu0独占(Exclusive状态)，cpu0执行foo，cpu1执行bar:

1. cpu0执行a=1，由于其有包含a的cache line，将a写入store buffer，并发出Invalidate a消息。
2. cpu1执行while(b == 0)，它没有b的cache，发出Read b消息。
3. cpu1收到cpu0的Invalidate a消息，将其放入Invalidate Queue，返回Invalidate ACK。
4. cpu0收到Invalidate ACK，将store buffer中的a=1刷新到cache line，标记为Modified。
5. cpu0看到smp_wmb()内存屏障，但是由于其store buffer为空，因此它可以直接跳过该语句。
6. cpu0执行b=1，由于其cache独占b，因此直接执行写入，cache line标记为Modified。
7. cpu0收到cpu1发的Read b消息，将包含b的cache line写回内存并返回该cache line，本地的cache line标记为Shared。
8. cpu1收到包含b(当前值1)的cache line，结束while循环。
9. cpu1执行assert(a == 1)，由于其本地有包含a旧值的cache line，读到a初始值0，断言失败。
10. cpu1这时才处理Invalid Queue中的消息，将包含a旧值的cache line置为Invalid。

问题在于第9步中cpu1在读取a的cache line时，没有先处理Invalid Queue中该cache line的Invalid操作，怎么办？其实cpu还提供了**读屏障指令**，Linux将其封装成smp_rmb()函数，将该函数插入到bar函数中，就像这样：

```java
void foo() {
    a = 1;
    smp_wmb()
    b = 1;
}
void bar() {
    while (b == 0) continue;
    smp_rmb()
    assert(a == 1)
}
```

和smp_wmb()类似，cpu执行smp_rmb()的时，会先把当前invalidate queue中的数据处理掉之后，再执行屏障后的“读取操作”，具体操作序列不再赘述。



