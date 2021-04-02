# Netty基本组件之NioEventLoop

[TOC]

## 一、EventLoopGroup介绍

​	NioEventLoopGroup是可以看做是NioEventLoop的集合，在netty的reactor的模型中，创建了两个NioEventLoopGroup。一个用于接收客户端的TCP连接（bossGroup）,另一个用于处理IO相关的读写操作,以及非IO相关的系统Task、定时任务Task等（workerGroup）。

类图如下:

![img](https://docimg6.docs.qq.com/image/B4Il37vIw4aLziZM3pGRHw?w=455&h=787)

由类图可以看到，NioEventLoopGroup其实相当于一个线程池，其主要的属性都在类MultithreadEventExecutorGroup中：

- children：实际机型任务的线程，即NioEventLoop，数组默认长度为2倍CPU核数
- chooser：线程选择器，当任务到来时，采用轮询均衡的策略（RoundRobin），由chooser选择由哪一个children去执行任务

关于Chooser，它有两个实现类；

- PowerOfTwoEventExecutorChooser：位运算实现轮训均衡策略
- GenericEventExecutorChooser：取余实现轮训均衡策略

## 二、NioEventLoop重要属性介绍

nioEventLoop中的重要属性（有些是继承自父类的）：

```
Queue<Runnable> taskQueue; //非IO任务队列
Thread thread; //执行NioEventLoop run方法的当前线程
PriorityQueue<ScheduledFutureTask<?>> scheduledTaskQueue; // 延时任务优先级队列
```

## 三、NioEventLoop的启动过程

​	执行NioEventLoop的父类SingleThreadEventExecutor的execute方法时候

```java
    private void execute(Runnable task, boolean immediate) {
        /**
         * 判断执行execute方法的线程是不是NioEventLoop对应的线程
         */
        boolean inEventLoop = inEventLoop();
        /**
         * 将待执行的task添加进taskQueue
         */
        addTask(task);
        if (!inEventLoop) {
            /**
             * 启动NioEventLoop
             */
            startThread();
            if (isShutdown()) {
                boolean reject = false;
                try {
                    if (removeTask(task)) {
                        reject = true;
                    }
                } catch (UnsupportedOperationException e) {
                    // The task queue does not support removal so the best thing we can do is to just move on and
                    // hope we will be able to pick-up the task before its completely terminated.
                    // In worst case we will log on termination.
                }
                if (reject) {
                    reject();
                }
            }
        }

        if (!addTaskWakesUp && immediate) {
            wakeup(inEventLoop);
        }
    }
```

由上可知，execute方法主要分为以下三个步骤：

1. 判断执行execute方法的线程是不是NioEventLoop对应的线程
2. 将待执行的task添加进taskQueue
3. 启动NioEventLoop

下面对这三个步骤进行详细分析

### 3.1 inEventLoop执行流程

inEventLoop方法具体逻辑如下：

```java
    @Override
    public boolean inEventLoop(Thread thread) {
        return thread == this.thread;
    }
```

​	判断执行execute方法的线程是不是NioEventLoop对应的线程，第一次执行这个方法时候this.thread值为null，所以返回值为false，会进入下面的if循环体，执行startThread（）启动NioEventLoop。

### 3.2 addTask执行流程

```java
    protected void addTask(Runnable task) {
        ObjectUtil.checkNotNull(task, "task");
        if (!offerTask(task)) {
            reject(task);
        }
    }
    
    final boolean offerTask(Runnable task) {
        if (isShutdown()) {
            reject();
        }
        return taskQueue.offer(task);
    }
```

​	addTask会将待执行的task添加进taskQueue，等待NioEventLoop执行时，再把task从taskQueue中取出来执行。

### 3.3 startThread执行流程

```java
    private void startThread() {
        if (state == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                boolean success = false;
                try {
                    doStartThread();
                    success = true;
                } finally {
                    if (!success) {
                        STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                    }
                }
            }
        }
    }

	private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                try {
                /**
                 * 赋值inEventLoop判断中的this.thread为当前线程
                 */
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }

                boolean success = false;
                updateLastExecutionTime();
                try {
                    /**
                     * 执行NioEventLoop的run方法
                     */
                    SingleThreadEventExecutor.this.run();
                    success = true;
                }
                .....
           }
      }
```

​	startThread主要分为以下两个步骤：

1. 赋值inEventLoop判断中的this.thread为当前线程

2. 执行执行NioEventLoop的run方法

   至此，NioEventLoop的run方法被执行，NioEventLoop才算正式被启动

## 四、NioEventLoop的执行过程

​	NioEventLoop的run方法，会执行I/O任务和非I/O任务：

- I/O任务

  即selectionKey中ready的事件，如accept、connect、read、write等，由processSelectedKeys方法触发。

- 非IO任务

  添加到taskQueue中的任务，如register0、bind0等任务，以及用户自定义的延时任务，由runAllTasks方法触发。

```java

    protected void run() {
        int selectCnt = 0;
        for (;;) {
            try {
                int strategy;
                try {
                    strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                    switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO
                    case SelectStrategy.SELECT:
                        /**
                         * 延时任务队列中最近一次的执行截止时间
                         */
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            /**
                             * 如果taskQueue中没有任务才执行select方法，否则优先执行taskQueue里面的任务
                             */
                            if (!hasTasks()) {
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            // This update is just to help block unnecessary selector wakeups
                            // so use of lazySet is ok (no race condition)
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                        // fall through
                    default:
                    }
                } catch (IOException e) {
                    // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                    // the selector and retry. https://github.com/netty/netty/issues/8566
                    rebuildSelector0();
                    selectCnt = 0;
                    handleLoopException(e);
                    continue;
                }

                selectCnt++;
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                boolean ranTasks;
                /**
                  * IO与非IO两种任务的执行时间比由变量ioRatio控制，默认为50，则表示允许非IO任务执行的时间与IO任务的执行时间相等。
                  */
                if (ioRatio == 100) {
                    try {
                        if (strategy > 0) {
                            processSelectedKeys();
                        }
                    } finally {
                        // Ensure we always run tasks.
                        ranTasks = runAllTasks();
                    }
                } else if (strategy > 0) {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        /**
                         * 确保每次都执行task，并根据IO操作的执行时间确认task执行的时间，默认是1：1
                         */
                        final long ioTime = System.nanoTime() - ioStartTime;
                        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                } else {
                    ranTasks = runAllTasks(0); // This will run the minimum number of tasks
                }

                if (ranTasks || strategy > 0) {
                    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                selectCnt - 1, selector);
                    }
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                    selectCnt = 0;
                }
            } catch (CancelledKeyException e) {
                // Harmless exception - log anyway
                if (logger.isDebugEnabled()) {
                    logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                            selector, e);
                }
            } catch (Error e) {
                throw (Error) e;
            } catch (Throwable t) {
                handleLoopException(t);
            } finally {
                // Always handle shutdown even if the loop processing threw an exception.
                try {
                    if (isShuttingDown()) {
                        closeAll();
                        if (confirmShutdown()) {
                            return;
                        }
                    }
                } catch (Error e) {
                    throw (Error) e;
                } catch (Throwable t) {
                    handleLoopException(t);
                }
            }
        }
    }
```

`NioEventLoop`执行过程大致可以分为如下三个步骤

1. 检测IO事件
2. 处理IO事件
3. 处理任务队列

下面详细介绍这三个步骤

### 4.1检测IO事件

核心代码如下：

```java
                    case SelectStrategy.SELECT:
                        /**
                         * 延时任务队列中最近一次的执行截止时间
                         */
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            /**
                             * 如果taskQueue中没有任务才执行select方法，否则优先执行taskQueue里面的任务
                             */
                            if (!hasTasks()) {
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            // This update is just to help block unnecessary selector wakeups
                            // so use of lazySet is ok (no race condition)
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                        // fall through
```

1.执行nextScheduledTaskDeadlineNanos方法，取scheduledTaskQueue中堆顶元素的执行时间，作为本次select操作的截止时间。nextScheduledTaskDeadlineNanos方法代码如下：

```java
    protected final long nextScheduledTaskDeadlineNanos() {
        ScheduledFutureTask<?> scheduledTask = peekScheduledTask();
        return scheduledTask != null ? scheduledTask.deadlineNanos() : -1;
    }

    final ScheduledFutureTask<?> peekScheduledTask() {
        Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
        return scheduledTaskQueue != null ? scheduledTaskQueue.peek() : null;
    }
```

2.执行hasTasks（）方法，判断scheduledTaskQueue中是否有延时任务，判断taskQueue中是否有非IO任务。如果有，则不执行select操作，在下面的代码中会通过执行runAllTasks（）方法优先执行任务队列里的任务。

3.执行select方法，select操作至多执行到延时队列中堆顶元素的执行时间为止，具体逻辑如下

```java
    private int select(long deadlineNanos) throws IOException {
        if (deadlineNanos == NONE) {
            return selector.select();
        }
        // Timeout will only be 0 if deadline is within 5 microsecs
        long timeoutMillis = deadlineToDelayNanos(deadlineNanos + 995000L) / 1000000L;
        return timeoutMillis <= 0 ? selector.selectNow() : selector.select(timeoutMillis);
    }
```

至此，检测IO事件至此结束

### 4.2处理IO事件

处理`IO`事件的过程是在`processSelectedKeys()`中完成

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

具体逻辑如下：

```java
    private void processSelectedKeysOptimized() {
        for (int i = 0; i < selectedKeys.size; ++i) {
            final SelectionKey k = selectedKeys.keys[i];
            // null out entry in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.keys[i] = null;

            /**
             * 取出io事件对应的channel
             */
            final Object a = k.attachment();

            /**
             * 处理该channel
             */
            if (a instanceof AbstractNioChannel) {
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }

            /**
             * 判断是否需要重建selectedKeys
             */
            if (needsToSelectAgain) {
                // null out entries in the array to allow to have it GC'ed once the Channel close
                // See https://github.com/netty/netty/issues/2363
                selectedKeys.reset(i + 1);

                selectAgain();
                i = -1;
            }
        }
    }
```

1.取出`IO`事件以及对应的`Netty Channel`类

```java
/**
 * 取出io事件对应的channel
 */
final Object a = k.attachment();
```

接下来分析一下这里的attachment()方法，我们回顾一下Channel注册的过程

```java
    @Override
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
    }
```

​	javaChannel()返回`Netty`类`SelectableChannel`对应的`JDK`底层`channel`对象

```java
protected SelectableChannel javaChannel() {
    return ch;
}
```

​	查看`SelectableChannel`的`register`方法，不难推断出`Netty`轮询注册机制其实是将`SelectableChannel`对象注册到`JDK`类`Selctor`对象上去，并且将`AbstractNioChannel`类作为一个`attachment`附属上，这样在`JDK`轮询出某条`SelectableChannel`有`IO`事件发生时，就可以直接取出`AbstractNioChannel`进行后续操作

2.执行processSelectedKey(k, (AbstractNioChannel) a)方法处理IO时间

3.判断是否需要重建`selector`

```java
if (needsToSelectAgain) {
    for (;;) {
        i++;
        if (selectedKeys[i] == null) {
            break;
        }
        selectedKeys[i] = null;
    }
    selectAgain();
    selectedKeys = this.selectedKeys.flip();
    i = -1;
}
```

什么时候`needsToSelectAgain`会重新被设置成`true`呢？

```
void cancel(SelectionKey key) {
    key.cancel();
    cancelledKeys ++;
    if (cancelledKeys >= CLEANUP_INTERVAL) {
        cancelledKeys = 0;
        needsToSelectAgain = true;
    }
}
```

继续查看`cancel`函数被调用的地方

```
protected void doDeregister() throws Exception {
    eventLoop().cancel(selectionKey());
}
```

在`Channel`从`selector`上移除时调用`cancel`函数将`key`取消，当被去掉的`key`到达`CLEANUP_INTERVAL`的时候设置`needsToSelectAgain`为`true`。每满`256`次就会进入到`if`的代码块，首先将`selectedKeys`的内部数组全部清空方便被`JVM`垃圾回收，然后重新调用`selectAgain`重新填装一下`selectionKey`。`Netty`这么做目的我想应该是每隔256次`channel`断线，重新清理一下`selectionKey`保证现存的`selectionKey`及时有效。

### 4.3处理任务队列

```java
/**
* 确保每次都执行task，并根据IO操作的执行时间确认task执行的时间，默认是1：1
*/
final long ioTime = System.nanoTime() - ioStartTime;
ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
```

ioRatio用来控制执行IO操作与非IO操作的时间比例，ioRatio默认是50，IO操作与非IO操作的默认执行时间是1:1。runAllTasks方法代码如下：

```java
protected boolean runAllTasks(long timeoutNanos) {
    /**
     * 把满足条件的延时队列scheduledTaskQueue中的task加入到非IO任务队列taskqueue中
     */
    fetchFromScheduledTaskQueue();
   /**
     * 从taskQueue中取出任务
     */
    Runnable task = pollTask();
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }

    /**
     * 计算本次runAllTasks的执行截止时间deadline
     */
    final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        /**
         * task.run()执行任务
         */
        safeExecute(task);

        runTasks ++;

        // Check timeout every 64 tasks because nanoTime() is relatively expensive.
        // XXX: Hard-coded value - will make it configurable if it is really a problem.
        /**
         * todo pengcanqi
         * 每执行64次task校验一下是否已经到达本次runAllTasks的截止时间，因为nanoTime（）的消耗很大
         */
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        /**
         * 如果已经没有task了则退出并记录本次任务执行结束的时间戳
         */
        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    /**
     * 执行SingleEventLoop中tailTask列表中的任务，但此task队列netty中暂无使用场景，是用于用户扩展功能的
     */
    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```

## 五、避免JDK Epoll空轮训的bug

关于这个bug大家可以在网上自行百度，这个bug导致的问题是导致while循环一直不间断死循环执行，CPU飙升至100%。那么Netty当中是如何避免这个bug的呢

```java
    // returns true if selectCnt should be reset
    private boolean unexpectedSelectorWakeup(int selectCnt) {
        if (Thread.interrupted()) {
            // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
            // As this is most likely a bug in the handler of the user or it's client library we will
            // also log it.
            //
            // See https://github.com/netty/netty/issues/2426
            if (logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely because " +
                        "Thread.currentThread().interrupt() was called. Use " +
                        "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
            }
            return true;
        }
        /**
         * 判断是否出现了空轮训bug，是否需要重建selector
         */
        if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
            // The selector returned prematurely many times in a row.
            // Rebuild the selector to work around the problem.
            logger.warn("Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                    selectCnt, selector);
            rebuildSelector();
            return true;
        }
        return false;
    }
```

SELECTOR_AUTO_REBUILD_THRESHOLD默认值是512，那么selectCnt值是从何而来呢？

在NioEventLoop的run循环体内，会对selectCnt进行++操作。但是又会在下面代码中给selectCnt赋值为0

```java
                if (ranTasks || strategy > 0) {
                    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                selectCnt - 1, selector);
                    }
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                    selectCnt = 0;
                }
```

runTask表示是否执行了taskQueue里的任务，strategy表示的是select操作返回值即就绪的selectedKey的数量。

综上，当没有执行taskQueue里的任务，select操作又没有找到对应的selectedKey，且这样的情况超过512次后，netty会重建selector避免jdk空轮训bug。