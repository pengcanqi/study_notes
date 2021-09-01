

# 							**《RocketMQ原理与应用》**

# 			**软件应用开发中心-后台应用开发组-彭璨麒**





# 一、RocketMQ网络层设计

## **rocketmq部署架构**

![部署架构](../study_note_image/rocketmq/部署架构.png)

涉及到的网络交互有很多，例如消息拉取、路由信息获取、消息发送、心跳检测.......

------

## IO模型选择

write操作的流程：

​	write操作不是要等到对方收到消息后才会返回，而是只把消息写到本地的write缓冲区冲，然后由操作系统把缓冲区中的数据发送到对端。**如果write缓冲区满，则就要等到缓冲区有空闲空间**，这个就是write操作IO操作的真正耗时之处。

read操作的流程：

​	read操作不是直接从目标机器拉取数据，只是把消息从本地的read缓冲区取出而已。**但如果read缓冲区为空，read操作就要等待read缓冲区中有数据**，这个也就是read操作IO操作的真正耗时之处。

![读io](../study_note_image/rocketmq/读io.png)

复用：从本质上来说，复用就是为了解决**有限资源和过多使用者的不平衡问题**，从而实现最大的利用率，处理更多的问题。

**Reactor模式**

![reactor](../study_note_image/rocketmq/reactor.png)

多Reactor多线程模型

- mainReactor负责监听ServerSocket，用来处理新连接的建立，通常单线程就可以处理，将建立的SocketChannel指定注册给subReactor
- subReactor(它的个数一般是和CPU个数等同)维护自己的selector,基于mainReactor注册的socketChannel多路分离IO读写事件，读写网络数据，对业务处理的功能，将其扔给worker线程池来完成。

## 设计概要

![网络层设计](../study_note_image/rocketmq/网络层设计.png)

## 网络层协议传输对象

RemotingCommand是网络层协议传输对象，核心字段如下：

- code，命令号
- opaque，请求唯一标识
- extFields，附加信息
- body，请求体

## 长度域解码器

```java
public class NettyDecoder extends LengthFieldBasedFrameDecoder {
    ......
}
```

NettyDecoder是NettyRemotingAbstract的入站解码器。把入站事件解析成RemotingCommand。而我们知道rocketmq的消息是不定长的，所以理所当然能知道RemotingCommand也是不定长的，所以我们的入站解码器是继承自LengthFieldBasedFrameDecoder，长度域解码器。

## 协议处理器映射表

processorTable是协议处理器映射表，根据RemotingCommand中的code找到对应的处理器，交由处理器对应的处理线程处理请求。

## 返回结果映射表

responseTable是返回结果映射表，当一次请求需要server返回结果时，会先在responseTable中维护一个opaque和responseFuture的映射，然后在server响应后，根据opaque找到responseFuture中的response，这样client就能获取到本次请求的响应了。

## 请求命令与回复命令

RemotingCommand又分为请求命令和回复命令两种类型：

```java
public enum RemotingCommandType {
    // 请求命令
    REQUEST_COMMAND,
    // 回复命令
    RESPONSE_COMMAND;
}
```

典型的交互如下所示：

![请求命令和回复命令](../study_note_image/rocketmq/请求命令和回复命令.png)

请求命令处理流程：

- 根据命令号，在处理器表中找到处理器
- 处理器处理事件
- 执行处理回调，写回响应（可选项）

回复命令处理流程：

- 根据回复命令的的opaque，在responseTable找到对应的responseFuture
- 将RemotingCommand中的reponse设置到responseFuture中
- 唤醒正在等待的responseFuture响应的线程
- 执行回调，处理response数据

## 网络层核心定时任务

scanResponseTable是NettyRemotingAbstract的核心定时任务之一，随着NettyRemotingAbstract启动而启动，每1s执行一次。扫描 responseTable 表，将过期的 responseFuture 移除。

## 同步调用与回执

![同步调用](../study_note_image/rocketmq/同步调用.png)

## 异步调用与回执

![异步请求](../study_note_image/rocketmq/异步请求.png)

## 单向与回执

![单向调用](../study_note_image/rocketmq/单向调用.png)

# 二、消息生产者

## MQClientInstance

MQClientInstance对象在RocketMQ中非常重量级，所有通信请求都会优先由它来处理，包括拉取请求、开线程定时刷新本地缓存数据、消费负载、心跳检测等等，因此针对一个MQ集群，一般是创建一个MQClientInstance实例。所以RocketMQ在底层维护了一个Map来缓存该实例（Key为客户端clientId，Value为客户端实例）。

```java
// 客户端实例管理器
public class MQClientManager {
    private final static InternalLogger log = ClientLogger.getLog();
    private static MQClientManager instance = new MQClientManager();
    private AtomicInteger factoryIndexGenerator = new AtomicInteger();
    // 管理客户端实例
    private ConcurrentMap<String/* clientId */, MQClientInstance> factoryTable =
        new ConcurrentHashMap<String, MQClientInstance>();
}
```

### clientId生成

```java
public class ClientConfig {
    public String buildMQClientId() {
        StringBuilder sb = new StringBuilder();
        sb.append(this.getClientIP());

        sb.append("@");
        sb.append(this.getInstanceName());
        if (!UtilAll.isBlank(this.unitName)) {
            sb.append("@");
            sb.append(this.unitName);
        }

        return sb.toString();
    }
}
```

如果未指定instanceName，会采用默认的instanceName “DEFAULT”，则clientId为 ip@DEFAULT

### 生产者/消费者注册

![MqClientInstance](../study_note_image/rocketmq/MqClientInstance.png)

MqClientInstance的配置由的ClientConfig类维护，MqClientInstance含有一个ClientConfig的引用clientConfig。所以MqClientInstance一般对应了一个MQ集群。多个生产者消费者线程都可以注册到同一个MqClientInstance中，MqClientInstance中有维护关于消费者和生产者的ConcurrentHashMap，其中key是group的名称。

### RocketMQ 多个实例消息配置

如果项目中需要同时消费多个不同集群的消息。注意一定要配置不同的instanceName，否则就算配置了不同的nameServer，但是由于生成的clientId是相同的，消费者启动时，拿到的MqClientInstance是同一个，永远都是第一个启动的MqClientInstance，导致配置的不同的nameServer其实并不会生效。

```java
@Bean("defaultMQProducer")
    public DefaultMQProducer defaultMQProducer() {
        if (StringUtils.isEmpty(nameServer)) {
            logger.error("nameServer can not be empty");
            return null;
        }
        DefaultMQProducer producer = new TraceDefaultMQProducer();
        producer.setNamesrvAddr(nameServer);
        producer.setProducerGroup(weatherInfoCycleProducerGroup);
        producer.setInstanceName(weatherInfoCycleInstanceName);
        try {
            producer.start();
        } catch (MQClientException e) {
            logger.error("producer start error", e);
        }
        return producer;
}
```

```
参考链接 https://blog.csdn.net/doctor_who2004/article/details/83120396
```

## 自动创建主题

**消费者向不存在主题发送消息**

![自动创建主题](../study_note_image/rocketmq/自动创建主题.png)

自动创建主题流程如上图所示。

当producer尝试向一个不存在的topic发送消息时，发送程序会报错：**No route info of this topic**。但是broker有个配置项**autoCreateTopicEnable**，当开启时，producer可以向一个不存在的topic发送消息，broker在收到消息后会自动创建主题。如上图所示，当autoCreateTopicEnable关闭时，无论从哪里都获取不到test/TWB102这个主题的路由信息，就会抛出No route info of this topic的异常。

------

**为什么生产上不建议自动创建主题**

为了消息发送的高可用，希望新创建的Topic在集群中的每台Broker上创建对应的队列，避免Broker的单节点故障。而自动创建主题，需要broker收到了这个不存在的主题的消息，才会自动创建。那另一个broker要是没收到，那相当于这个topic就不会在这个broker上创建。

疑问：那为啥不收到了一个topic创建请求做一下同步？

我个人的见解：同步这个得nameserver来做了，但是nameserver的设计又是轻量级的，是ap的，由它来做这个操作不符合设计初衷。

------

## 消息重投

### 同步消息重投

![同步消息重投](../study_note_image/rocketmq/同步消息重投.png)

```java
private int retryTimesWhenSendFailed = 2;
private boolean retryAnotherBrokerWhenNotStoreOK = false;
```

同步消息重投由上述两个变量控制，重投逻辑如上图所示，简单来说就是for循环里发送消息，for循环的循环次数就是最大重试次数，发送失败了就重试，发送成功了就返回发送结果。

同步发送失败重投次数，默认为2，因此生产者会最多尝试发送retryTimesWhenSendFailed +  1次。**不会选择上次失败的broker，尝试向其他broker发送，最大程度保证消息不丢**。超过重投次数，抛出异常，由客户端保证消息不丢。

### 异步消息重投

![异步消息重投](../study_note_image/rocketmq/异步消息重投.png)

retryTimesWhenSendAsyncFailed:异步发送失败重试次数，**异步重试不会选择其他broker，仅在同一个broker上做重试，不保证消息不丢**。

## Queue队列选择策略

**默认投递方式：基于`Queue队列`轮询算法投递**

```java
/**
 * 默认选择队列的方法
 * 参数1：上次失败的BrokerName（可以为null）
 * 返回值：当前主题下的 一个 队列
 */
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    if (lastBrokerName == null) {
        return selectOneMessageQueue();
    } else {
        for (int i = 0; i < this.messageQueueList.size(); i++) {
            int index = this.sendWhichQueue.getAndIncrement();
            int pos = Math.abs(index) % this.messageQueueList.size();
            if (pos < 0)
                pos = 0;
            MessageQueue mq = this.messageQueueList.get(pos);
            if (!mq.getBrokerName().equals(lastBrokerName)) {
                return mq;
            }
        }
        return selectOneMessageQueue();
    }
}
```

默认情况下，采用了最简单的轮询算法，这种算法有个很好的特性就是，保证每一个`Queue队列`的消息投递数量尽可能均匀。

当消息重投时，会选择与lastBrokerName不同的brokerName中的`Queue队列`。

------

**默认投递方式的增强：基于`Queue队列`轮询算法和`消息投递延迟最小`的策略投递**

默认的投递方式比较简单，但是也暴露了一个问题，就是有些`Queue队列`可能由于自身数量积压等原因，可能在投递的过程比较长，对于这样的`Queue队列`会影响后续投递的效果。 基于这种现象，RocketMQ在每发送一个MQ消息后，都会统计一下消息投递的`时间延迟`，根据这个`时间延迟`，可以知道往哪些`Queue队列`投递的速度快。 在这种场景下，会优先使用`消息投递延迟最小`的策略，如果没有生效，再使用`Queue队列轮询`的方式。

## 延时消息

![延时消息](../study_note_image/rocketmq/延时消息.png)

定时消息是指消息发送到broker后，不会立即被消费，等待特定时间投递给真正的topic。 broker有配置项messageDelayLevel，messageDelayLevel是broker的属性，不属于某个topic。默认的延时等级和延时时间对照关系如下：

| 1    | 2    | 3    | 4    | 5    | 6    | 7    | 8    | 9    | 10   | 11   | 12   | 13   | 14   | 15   | 16   | 17   | 18   |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 1s   | 5s   | 10s  | 30s  | 1m   | 2m   | 3m   | 4m   | 5m   | 6m   | 7m   | 8m   | 9m   | 10m  | 20m  | 30m  | 1h   | 2h   |

定时消息会暂存在名为SCHEDULE_TOPIC_XXXX的topic中，并根据delayTimeLevel存入特定的queue，而延时等级和队列ID的对应关系为：`queueId = delayLevel - 1`，即一个queue只存相同延迟的消息，保证具有相同发送延迟的消息能够顺序消费。broker会调度地消费SCHEDULE_TOPIC_XXXX，将消息写入真实的topic。

负责调度地消费SCHEDULE_TOPIC_XXXX的服务为**ScheduleMessageService**，具体的调度功能由内部维护的定时器Timer负责，它负责从SHHEDULE_TOPIC_XXXX对应的queue中，取出消息，分发到真实topic对应的queue中。

ScheduleMessageService里还维护了两个重要的map：

- 延时等级和延时时间映射
- 延时队列和对应消费位点映射（定期持久化到磁盘）

## 消息投递流程

![消息发送](../study_note_image/rocketmq/消息发送.png)

无论是哪种消息投递方式，都基本遵循如下流程：

1.根据topic获取到路由信息

2.选择队列

3.发送消息

# 三、消息消费者

## 消费重试

### 消费消息重试

![消费重试](../study_note_image/rocketmq/消费重试.png)

RocketMQ会为每个消费组都设置一个Topic名称为“%RETRY%+consumerGroup”的重试队列（**这里需要注意的是，这个Topic的重试队列是针对消费组，而不是针对每个Topic设置的**），用于暂时保存因为各种异常而导致Consumer端无法消费的消息。考虑到异常恢复起来需要一些时间，会为重试队列设置多个重试级别，每个重试级别都有与之对应的重新投递延时，重试次数越多投递延时就越大。RocketMQ对于重试消息的处理是先保存至Topic名称为“SCHEDULE_TOPIC_XXXX”的延迟队列中，后台定时任务按照对应的时间进行Delay后重新保存至“%RETRY%+consumerGroup”的重试队列中，当达到最大重试次数后，把消息保存到死信队列中（**%DLQ%**）

默认的重试级别是：`3+consumeTimes`

也即第一次重试默认是10秒后重试。

```java
消息重试带来的问题：由于MQ的重试机制，难免会引起消息的重复消费问题。比如一个ConsumerGroup中有两个，Consumer1和Consumer2，以集群方式消费。假设一条消息发往ConsumerGroup，由Consumer1消费，但是由于Consumer1消费过慢导致超时，此时Broker会将消息发送给Consumer2去消费，这样就产生了重复消费问题。因此，使用MQ时应该对一些关键消息进行幂等去重的处理。
```



## 消费偏移量

### OffsetStore

![OffsetStore](../study_note_image/rocketmq/OffsetStore.png)

消费偏移量在consumer端由类`OffsetStore`维护，它是一个接口，有如下两个实现类；

- LocalFileOffsetStore
- RemoteBrokerOffsetStore

其中LocalFileOffsetStore是广播消费使用，消费位点由consumer自己持久化。

RemoteBrokerOffsetStore是集群消费使用，消费位点由broker持久化。

广播消费这里不过多阐述，接下来所有的描述都基于集群消费。

### 初始化消费偏移量

![初始化消费位点](../study_note_image/rocketmq/初始化消费位点.png)

在`consumer`拉取消息时，在最初准备`pullRequest`对象时，会加载消费信息的偏移量，方法为`RebalanceImpl#updateProcessQueueTableInRebalance`。其实现就是从broker端获取到消费位点信息。

### 持久化消费偏移量

![持久化消费位点](../study_note_image/rocketmq/持久化消费位点.png)

消息消费服务ConsumeMessageService在每次消费消息后，会调用updateOffset方法，更新RemoteBrokerOffsetStore中的本地消费位点缓存offsetTable

MQClientInstance在启动时，会启动一个定时任务，5秒一次，将RemoteBrokerOffsetStore中的本地消费位点缓存offsetTable信息，发送到broker端，broker在接受到`UPDATE_CONSUMER_OFFSET`请求后，将客户端传过来的位点信息保存到ConsumerOffsetManager中的offsetTable中。需要注意的是，这个时候位点信息依旧只是保存在内存中。而具体持久化到文件，是需要broker中的定时任务，定期持久化ConsumerOffsetManager中的位点信息到文件中。

## 本地消费快照

![消费者本地快照](../study_note_image/rocketmq/消费者本地快照.png)

消息拉取下来后都会存入本地消费快照当中。

## 消息消费与消息拉取

![消息拉取与消息消费](../study_note_image/rocketmq/消息拉取与消息消费.png)

在`MQClientInstance#start`方法中，会启动消息拉取的服务：`PullMessageService`，`PullMessageService`是`ServiceThread`的子类，启动该服务时会创建一个新的线程，我们直接来看`PullMessageService#run()`方法

```java
@Override
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            PullRequest pullRequest = this.pullRequestQueue.take();
            this.pullMessage(pullRequest);
        } catch (InterruptedException ignored) {
        } catch (Exception e) {
            log.error("Pull Message Service Run Method exception", e);
        }
    }

    log.info(this.getServiceName() + " service end");
}
```

在`PullMessageService#run()`方法中，该方法会从`pullRequestQueue`中获取一个`pullRequest`的操作，然后调用`this.pullMessage(pullRequest)`进行拉取操作。

在拉取操作中，会调用到`DefaultMQPushConsumerImpl`的pullMessage方法，进行以下操作：

- 计算下一次拉取位点信息，重新添加拉取请求
- 把拉取到的消息存入本地快照
- 添加消息消费请求

消息消费服务，会从消息请求队列中取出请求，进行消费操作。最终会调用MessageListener中用户自定义的回调方法。

## 并发消费

推模式

```java
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
    handle(msgs);
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
```

## 顺序消费

![顺序消费](../study_note_image/rocketmq/顺序消费.png)

顺序消费由ConsumeMessageOrderlyService实现，随着service启动，有一个定时任务也随之启动，定时任务调用lockMQPeriodically方法，通过reblanceImpl，向broker发送一个LOCK_BATCH_MQ请求，并更新本地快照ProcessQueue的分布式锁标记**锁定成功后，同一个consumerGroup，同时只有一个client可以对messageQueue进行消息拉取**。

而ConsumeMessageOrderlyService内部实际上就行由一个线程池去消费consumeReuqest，在consumeReuqest内部就是对msg进行消费的逻辑。每次对对应的msg消费时，需要通过MessageQueueLock锁定消息队列，**保证本地同时只有一个线程在消费对应MessageQueue的消息**。同时还要判断ProcessQueue本地快照中的分布式锁标志是否还有效，且还要获取lockConsume这把可重入锁，避免与负载均衡冲突，避免因负载均衡导致消息重新消费。

消费消息是从快照中取出来的，快照是一个blockingQueue的结构，能够保证先进先出。

同一个consumerGroup，同时只有一个client可以对messageQueue进行消息拉取。同时只有一个线程在消费对应MessageQueue的消息。这样就保证了rocketmq的分区顺序性。

### 并发消费重试和顺序消费重试

并发消费重试优先采用上面的重试方式，重投消息，重投失败才本地进行重试。

顺序消费重试采用阻塞本地消费进度，优先本地重试，本地重试失败再进行消息重投，避免因消息重试导致的消息顺序问题。

## 负载均衡

![负载均衡](../study_note_image/rocketmq/负载均衡.PNG)先对Topic下的消息消费队列、消费者Id排序，然后用消息队列分配策略算法（默认为：消息队列的平均分配算法），计算出待拉取的消息队列。这里的平均分配算法，类似于分页的算法，将所有MessageQueue排好序类似于记录，将所有消费端Consumer排好序类似页数，并求出每一页需要包含的平均size和每个页面记录的范围range，最后遍历整个range而计算出当前Consumer端应该分配到的记录（这里即为：MessageQueue)。

[![img](https://github.com/apache/rocketmq/raw/master/docs/cn/image/rocketmq_design_8.png)](https://github.com/apache/rocketmq/blob/master/docs/cn/image/rocketmq_design_8.png)

然后，调用updateProcessQueueTableInRebalance()方法，具体的做法是，先将分配到的消息队列集合（mqSet）与processQueueTable做一个过滤比对。

[![img](https://github.com/apache/rocketmq/raw/master/docs/cn/image/rocketmq_design_9.png)](https://github.com/apache/rocketmq/blob/master/docs/cn/image/rocketmq_design_9.png)

- 上图中processQueueTable标注的红色部分，表示与分配到的消息队列集合mqSet互不包含。将这些队列设置Dropped属性为true，然后查看这些队列是否可以移除出processQueueTable缓存变量，这里具体执行removeUnnecessaryMessageQueue()方法，即每隔1s 查看是否可以获取当前消费处理队列的锁，拿到的话返回true。如果等待1s后，仍然拿不到当前消费处理队列的锁则返回false。如果返回true，则从processQueueTable缓存变量中移除对应的Entry；
- 上图中processQueueTable的绿色部分，表示与分配到的消息队列集合mqSet的交集。判断该ProcessQueue是否已经过期了，在Pull模式的不用管，如果是Push模式的，设置Dropped属性为true，并且调用removeUnnecessaryMessageQueue()方法，像上面一样尝试移除Entry，并持久化消费位点信息；

最后，为过滤后的消息队列集合（mqSet）中的每个MessageQueue创建一个ProcessQueue对象并存入RebalanceImpl的processQueueTable队列中（其中调用RebalanceImpl实例的computePullFromWhere(MessageQueue mq)方法获取该MessageQueue对象的下一个进度消费值offset，随后填充至接下来要创建的pullRequest对象属性中），并创建拉取请求对象—pullRequest添加到拉取列表—pullRequestList中，最后执行dispatchPullRequest()方法，将Pull消息的请求对象PullRequest依次放入PullMessageService服务线程的阻塞队列pullRequestQueue中，待该服务线程取出后向Broker端发起Pull消息的请求。

**并发消费负载均衡带来的问题：**

ConsumeRequest 是一个继承了 Runnable 的类，它是消息消费核心逻辑的实现类，submitConsumeRequest 方法将 ConsumeRequest 放入 消费线程池中执行消息消费，**从它的 run 方法中可看出，如果在执行消息消费逻辑中有节点加入，重平衡后该队列被分配给其它节点进行消费了，此时的队列被丢弃，则不提交消息消费进度，因为之前已经消费了，此时就会造成消息重复消费的情况。**

```
参考：https://cloud.tencent.com/developer/article/1521811?from=article.detail.1483408
```

## 订阅关系一致

正确的订阅关系

![正确的订阅关系](../study_note_image/rocketmq/正确的订阅关系.png)

错误的订阅关系

![错误的订阅关系](../study_note_image/rocketmq/错误的订阅关系.png)

```
错误订阅关系导致负载均衡出现问题：https://github.com/apache/rocketmq/issues/351
```

