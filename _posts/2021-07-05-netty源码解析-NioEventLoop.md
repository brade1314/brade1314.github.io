---
layout:       post
title:        "netty源码解析之NioEventLoop"
subtitle:     "NioEventLoop"
date:         2021-07-05 13:30:00
author:       "Brade"
header-style: text
header-mask:  0.3
catalog:      true
tags:
    - netty
    - 多线程
    - 源码
    - NioEventLoop
---

# NioEventLoop 源码解析
本文中 `netty` 采用的版本为 `4.1.66.Final-SNAPSHOT`。`NioEventLoop` 的`类图`如下所示：
> ![img.png](/img/netty/NioEventLoop.png)
> 最顶层跟`NioEventLoopGroup`一样，都是`Executor` ==> `ExecutorService`

## EventExecutor
接口，继承了`EventExecutorGroup`，是一个特殊的`EventExecutorGroup`，新定义了一些查看`Thread`是否在事件循环中执行。
```java
public interface EventExecutor extends EventExecutorGroup {

    /**
     * Returns a reference to itself.
     */
    @Override
    EventExecutor next();

    /**
     * 返回当前 EventExecutor 的父级EventExecutorGroup
     * Return the {@link EventExecutorGroup} which is the parent of this {@link EventExecutor},
     */
    EventExecutorGroup parent();

    /**
     * 当前线程是否在 EventLoop
     * Calls {@link #inEventLoop(Thread)} with {@link Thread#currentThread()} as argument
     */
    boolean inEventLoop();

    /**
     * 给出的指定线程是否在 EventLoop
     * Return {@code true} if the given {@link Thread} is executed in the event loop,
     * {@code false} otherwise.
     */
    boolean inEventLoop(Thread thread);

    /**
     * Return a new {@link Promise}.
     */
    <V> Promise<V> newPromise();

    /**
     * Create a new {@link ProgressivePromise}.
     */
    <V> ProgressivePromise<V> newProgressivePromise();

    /**
     * Create a new {@link Future} which is marked as succeeded already. So {@link Future#isSuccess()}
     * will return {@code true}. All {@link FutureListener} added to it will be notified directly. Also
     * every call of blocking methods will just return without blocking.
     */
    <V> Future<V> newSucceededFuture(V result);

    /**
     * Create a new {@link Future} which is marked as failed already. So {@link Future#isSuccess()}
     * will return {@code false}. All {@link FutureListener} added to it will be notified directly. Also
     * every call of blocking methods will just return without blocking.
     */
    <V> Future<V> newFailedFuture(Throwable cause);
}
```
## OrderedEventExecutor
接口，只继承了 `EventExecutor`，未做任何处理，空白接口。

## AbstractExecutorService
`JDK`中定义普通类，是`ExecutorService`的默认实现类。此类说明：该类使用`newTaskFor` 返回的`RunnableFuture` 实现`submit`、`invokeAny`
和`invokeAll` 方法，默认为该包中提供的`FutureTask` 类。例如，`submit(Runnable)` 的实现创建了一个关联的 `RunnableFuture`，它被执行并返回。
子类可以覆盖 `newTaskFor` 方法以返回除 `FutureTask` 之外的 `RunnableFuture` 实现。
```java
 protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
```

## AbstractEventExecutor 
抽象类，继承`AbstractExecutorService` 然后实现了`EventExecutor`中定义的和继承自`EventExecutorGroup`的接口。

## AbstractScheduledEventExecutor
抽象类，继承`AbstractEventExecutor`，扩展了支持调度的`AbstractEventExecutor`。
> 初始化了一个长度为11的调度队列`DefaultPriorityQueue`，队列是一个`ScheduledFutureTask`数组。
```java
    PriorityQueue<ScheduledFutureTask<?>> scheduledTaskQueue() {
        if (scheduledTaskQueue == null) {
            scheduledTaskQueue = new DefaultPriorityQueue<ScheduledFutureTask<?>>(
                    SCHEDULED_FUTURE_TASK_COMPARATOR,
                    // Use same initial capacity as java.util.PriorityQueue
                    11);
        }
        return scheduledTaskQueue;
    }
```
> 队列长度增长算法如下：当队列长度小于`64`时，增长为原来的`2倍 + 2`，大于等于是增长为原来的`1.5倍`。

```java
    @Override
    public boolean offer(T e) {
        if (e.priorityQueueIndex(this) != INDEX_NOT_IN_QUEUE) {
            throw new IllegalArgumentException("e.priorityQueueIndex(): " + e.priorityQueueIndex(this) +
                    " (expected: " + INDEX_NOT_IN_QUEUE + ") + e: " + e);
        }

        // Check that the array capacity is enough to hold values by doubling capacity.
        if (size >= queue.length) {
            // Use a policy which allows for a 0 initial capacity. Same policy as JDK's priority queue, double when
            // "small", then grow by 50% when "large".
            queue = Arrays.copyOf(queue, queue.length + ((queue.length < 64) ?
                                                         (queue.length + 2) :
                                                         (queue.length >>> 1)));
        }

        bubbleUp(size++, e);
        return true;
    }
```

> 具体调度实现在这个方法：

```java
    private <V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
        // 当前线程如果在EventLoop中，直接放到任务队列
        if (inEventLoop()) {
            scheduleFromEventLoop(task);
        } else {
            final long deadlineNanos = task.deadlineNanos();
            // 当前线程的不在EventLoop，调度任务也未过期，直接执行
            // beforeScheduledTaskSubmitted 和 afterScheduledTaskSubmitted 在NioEventLoop/EpollEventLoop中重写 
            if (beforeScheduledTaskSubmitted(deadlineNanos)) {
                execute(task);
            } else {
                // 懒加载执行，具体功能在子类SingleThreadEventExecutor 中的 private void execute(Runnable task, boolean immediate) 方法实现
                lazyExecute(task);
                // 调度任务已经提交
                if (afterScheduledTaskSubmitted(deadlineNanos)) {
                // 执行唤醒任务
                    execute(WAKEUP_TASK);
                }
            }
        }

        return task;
    }
```

## SingleThreadEventExecutor
抽象类，继承了[AbstractScheduledEventExecutor](#AbstractScheduledEventExecutor)，实现了 [OrderedEventExecutor](#OrderedEventExecutor)单个线程中执行所有提交的任务。实际上是大部分实现都在这个类，跟`MultithreadEventExecutorGroup`对应，子类也是调用的它里面方法。
> 构造器主要是这个：
```java
 /**
     * Create a new instance
     *
     * @param parent            the {@link EventExecutorGroup} which is the parent of this instance and belongs to it
     * @param executor          the {@link Executor} which will be used for executing
     * @param addTaskWakesUp    {@code true} if and only if invocation of {@link #addTask(Runnable)} will wake up the
     *                          executor thread
     * @param maxPendingTasks   the maximum number of pending tasks before new tasks will be rejected.
     * @param rejectedHandler   the {@link RejectedExecutionHandler} to use.
     */
    protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, int maxPendingTasks,
                                        RejectedExecutionHandler rejectedHandler) {
        // 当前Executor的直属集合
        super(parent);
        // 是否增加到唤醒任务，增加后会唤醒执行线程
        this.addTaskWakesUp = addTaskWakesUp;
        // 允许的最大挂起任务数，默认16
        this.maxPendingTasks = Math.max(16, maxPendingTasks);
        // 当前使用的执行器
        this.executor = ThreadExecutorMap.apply(executor, this);
        // 任务队列
        taskQueue = newTaskQueue(this.maxPendingTasks);
        // 拒绝策略，当超出最大挂起任务后执行
        rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
    }
```
> 具体实现了父类的 [AbstractScheduledEventExecutor](#AbstractScheduledEventExecutor) `lazyExecute()` 方法：

```java
    @Override
    public void lazyExecute(Runnable task) {
        execute(ObjectUtil.checkNotNull(task, "task"), false);
    }

    private void execute(Runnable task, boolean immediate) {
        // 判断当前线程是否在EventLoop，其实就是判断正在执行的线程与EventLoop中的线程是不是同一个
        boolean inEventLoop = inEventLoop();
        // 将线程增加到执行队列
        addTask(task);
        // 不在eventLoop
        if (!inEventLoop) {
            // 开始启动线程
            startThread();
            // 线程已经停止
            if (isShutdown()) {
                boolean reject = false;
                try {
                    // 先从队列移除线程
                    if (removeTask(task)) {
                        // 拒绝状态设置为true
                        reject = true;
                    }
                } catch (UnsupportedOperationException e) {
                    // The task queue does not support removal so the best thing we can do is to just move on and
                    // hope we will be able to pick-up the task before its completely terminated.
                    // In worst case we will log on termination.
                }
                // 拒绝状态为true
                if (reject) {
                    // 执行拒绝方法：抛个异常 RejectedExecutionException
                    reject();
                }
            }
        }

        // 如果线程不需要唤醒且立刻执行：
        if (!addTaskWakesUp && immediate) {
            // 执行唤醒方法，参数为是否在inEventLoop
            wakeup(inEventLoop);
        }
    }
```

```java
    /**
     * 开始启动线程
     */
    private void startThread() {
        // 线程状态必须是 未启动：1
        if (state == ST_NOT_STARTED) {
            // SingleThreadEventExecutor 实例的状态字段state 从 ST_NOT_STARTED=1 设置为 ST_STARTED=2，即未启动==>启动
            // 调用的是compareAndSet方法，先比较再设值 ，这是一个原子方法，同时多线程状态效率高，采用分段set的方法
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                boolean success = false;
                try {
                    // 开始调用启动业务方法
                    doStartThread();
                    // 启动状态设置
                    success = true;
                } finally {
                    // 如果启动失败
                    if (!success) {
                        STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                    }
                }
            }
        }
    }
```

```java
    /**
     * 线程启动的具体业务方法
     */
    private void doStartThread() {
        // 线程对象必须为空
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                // 线程对象设值为当前线程
                thread = Thread.currentThread();
                // 如果打断状态为true，执行线程打断的方法
                if (interrupted) {
                    thread.interrupt();
                }
                // 设置一个执行是否成功状态字段
                boolean success = false;
                // 更新最近一次执行提交的任务的时间
                updateLastExecutionTime();
                try {
                    // run 方法在子类 NioEventLoop 中重写，实际上调用的是NioEventLoop.run()
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    // 死循环，检查线程状态，直到线程执行完毕
                    for (;;) {
                        int oldState = state;
                        // 如果当前线程状态为已经停止或者设置当前实例状态为停止状态设置成功，就中断死循环
                        // 当前线程状态设置为 ST_SHUTTING_DOWN = 3
                        if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                            break;
                        }
                    }

                    // Check if confirmShutdown() was called at the end of the loop.
                    // 检查是否在循环结束时调用了 confirmShutdown() 确认线程停止
                    if (success && gracefulShutdownStartTime == 0) {
                        if (logger.isErrorEnabled()) {
                            logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                    SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                    "be called before run() implementation terminates.");
                        }
                    }

                    try {
                        // Run all remaining tasks and shutdown hooks. At this point the event loop
                        // is in ST_SHUTTING_DOWN state still accepting tasks which is needed for
                        // graceful shutdown with quietPeriod.
                        // 运行任务队列中所有剩余的任务和关闭钩子。 此时，事件循环处于 ST_SHUTTING_DOWN 状态，仍在接受使用 quietPeriod 正常关闭所需的任务。
                        // 死循环，检查线程确实关闭
                        for (;;) {
                            if (confirmShutdown()) {
                                break;
                            }
                        }

                        // Now we want to make sure no more tasks can be added from this point. This is
                        // achieved by switching the state. Any new tasks beyond this point will be rejected.
                        // 现在我们要确保从这一点开始不能再添加任务。 这是通过切换状态来实现的。 任何超出此点的新任务都将被拒绝。
                        // 当前线程状态设置为 ST_SHUTTING_DOWN = 4
                        for (;;) {
                            int oldState = state;
                            if (oldState >= ST_SHUTDOWN || STATE_UPDATER.compareAndSet(
                                    SingleThreadEventExecutor.this, oldState, ST_SHUTDOWN)) {
                                break;
                            }
                        }

                        // We have the final set of tasks in the queue now, no more can be added, run all remaining.
                        // No need to loop here, this is the final pass.
                        // 队列中的任务已经完成了，不能再添加了，运行所有剩余的任务。这里不需要循环，这是最后的关卡。
                        confirmShutdown();
                    } finally {
                        try {
                            // 当前类不实现，在子类NioEventLoop中重写，其实就是关闭选择器 selector.close();
                            cleanup();
                        } finally {
                            // Lets remove all FastThreadLocals for the Thread as we are about to terminate and notify
                            // the future. The user may block on the future and once it unblocks the JVM may terminate
                            // and start unloading classes.
                            // See https://github.com/netty/netty/issues/6596.
                            // 移除线程的所有 FastThreadLocals。 用户可能会阻塞在future 中，一旦它解除阻塞，JVM 可能会终止并开始卸载类
                            FastThreadLocal.removeAll();

                            // 当前线程状态设置为 ST_TERMINATED=5
                            STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                            // 倒计时计数器开始工作
                            threadLock.countDown();
                            int numUserTasks = drainTasks();
                            if (numUserTasks > 0 && logger.isWarnEnabled()) {
                                logger.warn("An event executor terminated with " +
                                        "non-empty task queue (" + numUserTasks + ')');
                            }
                            terminationFuture.setSuccess(null);
                        }
                    }
                }
            }
        });
    }

    /**
     * 检查队列中的任务数，唤醒任务除外
     * @return
     */
    final int drainTasks() {
        int numTasks = 0;
        for (;;) {
            Runnable runnable = taskQueue.poll();
            if (runnable == null) {
                break;
            }
            // WAKEUP_TASK should be just discarded as these are added internally.
            // The important bit is that we not have any user tasks left.
            if (WAKEUP_TASK != runnable) {
                numTasks++;
            }
        }
        return numTasks;
    }
```

## SingleThreadEventLoop
抽象类，继承了 `SingleThreadEventExecutor` ,再实现了`EventLoop`里面的接口。
> 比如注册 `channel` 到 `selector` 上的接口

```java

    @Override
    public EventLoop next() {
        //调用的父类 AbstractEventExecutor中的 next()
        return (EventLoop) super.next();
    }
    
    @Override
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        // 调用Channel==>默认实现类，抽象类AbstractChannel中的 AbstractUnsafe.register()
        // ==> AbstractUnsafe.register0() ==> AbstractNioChannel.doRegister()
        // 从而把channel注册在了selector上
        // 其中的selector是调用的 NioEventLoop.unwrappedSelector()
        promise.channel().unsafe().register(this, promise);
        return promise;
    }
    
```

## NioEventLoop
普通类，继承了`SingleThreadEventLoop`，实现多路复用`selector`的注册方法。
> 构造器，`NioEventLoopGroup`  的 `newChild()` 进行了调用:

```java

    /**
     * 构造器，NioEventLoopGroup 的 newChild() 进行了调用
     * @param parent 
     * @param executor
     * @param selectorProvider
     * @param strategy
     * @param rejectedExecutionHandler
     * @param taskQueueFactory
     * @param tailTaskQueueFactory
     */
    NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
                 EventLoopTaskQueueFactory taskQueueFactory, EventLoopTaskQueueFactory tailTaskQueueFactory) {
        // 调用父类的构造器，单线程事件循环
        super(parent, executor, false, newTaskQueue(taskQueueFactory), newTaskQueue(tailTaskQueueFactory),
                rejectedExecutionHandler);
        this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
        this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
        final SelectorTuple selectorTuple = openSelector();
        this.selector = selectorTuple.selector;
        this.unwrappedSelector = selectorTuple.unwrappedSelector;
    }
    
```

> `run()方法`，在 [SingleThreadEventExecutor](#SingleThreadEventExecutor) 中的 `doStartThread()` 中进行调用

```java

    @Override
    protected void run() {
        int selectCnt = 0;
        // 死循环
        for (;;) {
            try {
                int strategy;
                try {
                    // 计算selector策略：如果任务队列不为空，返回 selectNow()，否则返回 SelectStrategy.SELECT
                    strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                    switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
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
                // 如果用于处理I/O操作的比例为100%
                if (ioRatio == 100) {
                    try {
                        if (strategy > 0) {
                            // 调用处理 processSelectedKeys策略方法
                            processSelectedKeys();
                        }
                    } finally {
                        // Ensure we always run tasks.
                        // 执行所有任务
                        ranTasks = runAllTasks();
                    }
                } else if (strategy > 0) {
                    final long ioStartTime = System.nanoTime();
                    try {
                        // 调用处理 processSelectedKeys策略方法
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        // 轮询任务队列中的所有任务，如果运行时间超过设置的超时时间则停止队列任务并返回
                        // 默认50，超时时间刚好 ioTime * 1，50 < ioRatio < 100 则越靠近100，处理任务时间越长
                        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                } else {
                    ranTasks = runAllTasks(0); // This will run the minimum number of tasks
                }

                // 任务执行完毕或者 strategy>0
                if (ranTasks || strategy > 0) {
                    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                selectCnt - 1, selector);
                    }
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case) 意外唤醒（异常情况）
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
                        // 关闭全部channel
                        closeAll();
                        // 确认全部线程关闭
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

> 打开创建 `selector`

```java

    /**
     * 打开选择器，返回选择器元组
     * @return SelectorTuple
     */
    private SelectorTuple openSelector() {
        final Selector unwrappedSelector;
        try {
            // 默认调用WindowsSelectorProvider.openSelector()
            // ==> new WindowsSelectorImpl(this)
            // 因为jdk是下载的win环境下的
            unwrappedSelector = provider.openSelector();
        } catch (IOException e) {
            throw new ChannelException("failed to open a new selector", e);
        }

        if (DISABLE_KEY_SET_OPTIMIZATION) {
            return new SelectorTuple(unwrappedSelector);
        }

        Object maybeSelectorImplClass = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    return Class.forName(
                            "sun.nio.ch.SelectorImpl",
                            false,
                            PlatformDependent.getSystemClassLoader());
                } catch (Throwable cause) {
                    return cause;
                }
            }
        });

        if (!(maybeSelectorImplClass instanceof Class) ||
            // ensure the current selector implementation is what we can instrument.
            !((Class<?>) maybeSelectorImplClass).isAssignableFrom(unwrappedSelector.getClass())) {
            if (maybeSelectorImplClass instanceof Throwable) {
                Throwable t = (Throwable) maybeSelectorImplClass;
                logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, t);
            }
            return new SelectorTuple(unwrappedSelector);
        }

        final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;
        final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

        Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
                    Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                    if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {
                        // Let us try to use sun.misc.Unsafe to replace the SelectionKeySet.
                        // This allows us to also do this in Java9+ without any extra flags.
                        long selectedKeysFieldOffset = PlatformDependent.objectFieldOffset(selectedKeysField);
                        long publicSelectedKeysFieldOffset =
                                PlatformDependent.objectFieldOffset(publicSelectedKeysField);

                        if (selectedKeysFieldOffset != -1 && publicSelectedKeysFieldOffset != -1) {
                            PlatformDependent.putObject(
                                    unwrappedSelector, selectedKeysFieldOffset, selectedKeySet);
                            PlatformDependent.putObject(
                                    unwrappedSelector, publicSelectedKeysFieldOffset, selectedKeySet);
                            return null;
                        }
                        // We could not retrieve the offset, lets try reflection as last-resort.
                    }

                    Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }
                    cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }

                    selectedKeysField.set(unwrappedSelector, selectedKeySet);
                    publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
                    return null;
                } catch (NoSuchFieldException e) {
                    return e;
                } catch (IllegalAccessException e) {
                    return e;
                }
            }
        });

        if (maybeException instanceof Exception) {
            selectedKeys = null;
            Exception e = (Exception) maybeException;
            logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, e);
            return new SelectorTuple(unwrappedSelector);
        }
        selectedKeys = selectedKeySet;
        logger.trace("instrumented a special java.util.Set into: {}", unwrappedSelector);
        return new SelectorTuple(unwrappedSelector,
                                 new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet));
    }
```

> `run`方法中调用了处理`SelectedKeys`的策略。

```java

    /**
     * 处理 SelectedKeys 策略
     */
    private void processSelectedKeys() {
        if (selectedKeys != null) {
            //优化过的selectedKeys采用数组，增加元素和扩容进行了优化，采用了数组，add操作永远都是O(1)的时间复杂度
            processSelectedKeysOptimized();
        } else {
            // 普通的selectedKeys是个set
            processSelectedKeysPlain(selector.selectedKeys());
        }
    }
    
```