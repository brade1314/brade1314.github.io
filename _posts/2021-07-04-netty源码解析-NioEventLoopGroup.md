---
layout:       post
title:        "netty源码解析"
subtitle:     "NioEventLoopGroup"
date:         2021-07-04 16:30:00
author:       "Brade"
header-style: text
header-mask:  0.3
catalog:      true
tags:
    - netty
    - 多线程
    - 源码
    - NioEventLoopGroup
---

# NioEventLoopGroup 源码解析
本文中 `netty` 采用的版本为 `4.1.66.Final-SNAPSHOT`。`NioEventLoopGroup` 的`类图`如下所示：
> ![img.png](/img/netty/NioEventLoopGroup.png)

## Executor
JDK中定义的接口，线程执行器，主要用来控制线程的启动、执行和关闭，可以简化并发编程的操作，只定义了一个方法。
```java
    void execute(Runnable command);
```

## ExecutorService
JDK中定义的接口，扩展了`Executor`的功能，提供了生命周期管理的方法，返回 Future 对象。
> ![img.png](/img/netty/ExecutorService.png)

## ScheduledExecutorService
JDK中定义的接口，扩展了`ExecutorService`，用于线程任务的定时或者延期执行。
> ![img.png](/img/netty/ScheduledExecutorService.png)

## EventExecutorGroup
接口，继承了`ScheduledExecutorService` 和 `Iterator`，因此具备了线程和iterator最基本的方法，同时定义了关闭线程、释放线程资源的`优雅`方法。
> ![img.png](/img/netty/EventExecutorGroup.png)

## EventLoopGroup
接口，定义了注册`channel`到`EventLoop`方法，注册完成后会收到通知，返回的是`ChannelFuture`。
```java
public interface EventLoopGroup extends EventExecutorGroup {
    /**
     * Return the next {@link EventLoop} to use
     */
    @Override
    EventLoop next();

    /**
     * Register a {@link Channel} with this {@link EventLoop}. The returned {@link ChannelFuture}
     * will get notified once the registration was complete.
     */
    ChannelFuture register(Channel channel);

    /**
     * Register a {@link Channel} with this {@link EventLoop} using a {@link ChannelFuture}. The passed
     * {@link ChannelFuture} will get notified once the registration was complete and also will get returned.
     */
    ChannelFuture register(ChannelPromise promise);

    /**
     * Register a {@link Channel} with this {@link EventLoop}. The passed {@link ChannelFuture}
     * will get notified once the registration was complete and also will get returned.
     *
     * @deprecated Use {@link #register(ChannelPromise)} instead.
     */
    @Deprecated
    ChannelFuture register(Channel channel, ChannelPromise promise);
}
```


## AbstractEventExecutorGroup
抽象类，`EventExecutorGroup` 的默认实现类，对线程任务的定时和延期执行方法具体实现。
```java
public abstract class AbstractEventExecutorGroup implements EventExecutorGroup {
    @Override
    public Future<?> submit(Runnable task) {
        return next().submit(task);
    }

    @Override
    public <T> Future<T> submit(Runnable task, T result) {
        return next().submit(task, result);
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        return next().submit(task);
    }

    @Override
    public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
        return next().schedule(command, delay, unit);
    }

    @Override
    public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {
        return next().schedule(callable, delay, unit);
    }

    @Override
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
        return next().scheduleAtFixedRate(command, initialDelay, period, unit);
    }

    @Override
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
        return next().scheduleWithFixedDelay(command, initialDelay, delay, unit);
    }

    @Override
    public Future<?> shutdownGracefully() {
        return shutdownGracefully(DEFAULT_SHUTDOWN_QUIET_PERIOD, DEFAULT_SHUTDOWN_TIMEOUT, TimeUnit.SECONDS);
    }

    @Override
    public <T> List<java.util.concurrent.Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
            throws InterruptedException {
        return next().invokeAll(tasks);
    }

    @Override
    public <T> List<java.util.concurrent.Future<T>> invokeAll(
            Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException {
        return next().invokeAll(tasks, timeout, unit);
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException {
        return next().invokeAny(tasks);
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        return next().invokeAny(tasks, timeout, unit);
    }

    @Override
    public void execute(Runnable command) {
        next().execute(command);
    }
}
```

## MultithreadEventExecutorGroup
抽象类，多线程事件循环执行器池，继承了`AbstractEventExecutorGroup`，维护了一个`EventExecutor`数组，管理多线程，定义了一些 `EventExecutor` 数组的操作方法。
> ![img.png](/img/netty/MultithreadEventExecutorGroup.png)
> 
> 主要在这个构造器中进行功能实现，调用子类`NioEventLoopGroup`中的`newChild`方法。
```java
    /**
     * Create a new instance.
     *
     * @param nThreads          the number of threads that will be used by this instance.
     * @param executor          the Executor to use, or {@code null} if the default should be used.
     * @param chooserFactory    the {@link EventExecutorChooserFactory} to use.
     * @param args              arguments which will passed to each {@link #newChild(Executor, Object...)} call
     */
    protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        // 检查线程数量>0
        checkPositive(nThreads, "nThreads");

        //  如果执行器为空，则创建一个
        if (executor == null) {
            // 线程优先策略，采用默认优先级5，最小优先级1，最大优先级10
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        // 执行器数组大小为给定的线程数量
        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                // 调用子类创建执行器的方法，创建多线程
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                // 创建失败，则释放执行器数组内的线程资源
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }
        /**
         * 选择器，根据执行器数据长度确定如何挑选，默认实现为DefaultEventExecutorChooserFactory
         * 采用策略者模式：如果是2的指数倍, 返回 PowerOfTwoEventExecutorChooser，否则返回 GenericEventExecutorChooser
         */
        chooser = chooserFactory.newChooser(children);

        // 创建线程停止监听器
        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        // 为每个执行器添加线程停止监听
        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        // 只读事件实例化，返回指定集合的不可修改集合，这种方法允许模块为用户提供对内部集合的“只读”访问
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```

## MultithreadEventLoopGroup
抽象类，多线程事件循环池，`EventLoopGroup` 实现的抽象基类，同时继承了`MultithreadEventExecutorGroup`，可同时处理多个线程的任务。
> - 设置了默认线程数为`cup核心 * 2`
``` java
static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }
```
> - 重写父类 `MultithreadEventExecutorGroup` 的默认线程工厂方法，父类方法线程正常优先级 `5`，重写后为最大优先级 `10`。
```java
    @Override
    protected ThreadFactory newDefaultThreadFactory() {
        return new DefaultThreadFactory(getClass(), Thread.MAX_PRIORITY);
    }
```
> - 实现接口`EventLoopGroup`定义的注册`channel`方法，`EventLoop`处理`channel`注册后的所有`I/O`操作
```java
    @Override
    public EventLoop next() {
        return (EventLoop) super.next();
    }
    @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }

    @Override
    public ChannelFuture register(ChannelPromise promise) {
            return next().register(promise);
    }
```

## NioEventLoopGroup
普通类，具体实现了`MultithreadEventExecutorGroup`中的`newChild`方法，最后调用`new NioEventLoop`创建`NioEventLoop`，同时增加了`setIoRatio` 和 `rebuildSelectors` 方法。
```java
    /**
     * 重写创建执行器方法，继承自父类MultithreadEventLoopGroup的父类MultithreadEventExecutorGroup中的方法
     * 无参的构造器NioEventLoopGroup()调用方式：
     * ==> 1.
     * ==> nThread为0，
     * ==> executor为null，
     * ==> 选择器策略为默认 DefaultSelectStrategyFactory.INSTANCE
     * ==> RejectedExecutionHandler为 reject()（即直接抛异常）
     * ==> 2.
     * ==> 先调用父类 MultithreadEventLoopGroup 构造器，线程数初始化线程数为 cpu核心 * 2
     * ==> 3.
     * ==> MultithreadEventLoopGroup 再调用父类 MultithreadEventExecutorGroup 构造器，new ThreadPerTaskExecutor，
     * ==> 4.
     * ==> 调用子类 NioEventLoopGroup的 newChild 方法
     *
     * @param executor
     * @param args
     * @return
     * @throws Exception
     */
    @Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        // 1 selector 工厂方法，根据系统属性java.nio.channels.spi.SelectorProvider ，调用反射获取
        // 2 ServiceLoader 采用spi获取，这里采用的方式和jdk中的selector策略一样
        // 3 采用默认的selectorProvider:DefaultSelectorProvider==> WindowsSelectorProvider==> WindowsSelectorImpl
        SelectorProvider selectorProvider = (SelectorProvider) args[0];
        // selector策略工厂方法
        SelectStrategyFactory selectStrategyFactory = (SelectStrategyFactory) args[1];
        // 执行器拒绝策略
        RejectedExecutionHandler rejectedExecutionHandler = (RejectedExecutionHandler) args[2];
        // 任务队列工厂方法
        EventLoopTaskQueueFactory taskQueueFactory = null;
        // 尾部任务队列工厂方法
        EventLoopTaskQueueFactory tailTaskQueueFactory = null;

        int argsLength = args.length;
        if (argsLength > 3) {
            taskQueueFactory = (EventLoopTaskQueueFactory) args[3];
        }
        if (argsLength > 4) {
            tailTaskQueueFactory = (EventLoopTaskQueueFactory) args[4];
        }
        return new NioEventLoop(this, executor, selectorProvider,
                selectStrategyFactory.newSelectStrategy(),
                rejectedExecutionHandler, taskQueueFactory, tailTaskQueueFactory);
    }
```
设置子事件循环中用于 I/O 的所需时间量的百分比，默认50，是在`NioEventLoop`中定义的。
```java
/**
     * Sets the percentage of the desired amount of time spent for I/O in the child event loops.  The default value is
     * {@code 50}, which means the event loop will try to spend the same amount of time for I/O as for non-I/O tasks.
     */
    public void setIoRatio(int ioRatio) {
        for (EventExecutor e: this) {
            ((NioEventLoop) e).setIoRatio(ioRatio);
        }
    }
```
解决 `Linux` 的`epoll`模式 100% CPU 错误问题。
```java
/**
     * Replaces the current {@link Selector}s of the child event loops with newly created {@link Selector}s to work
     * around the  infamous epoll 100% CPU bug.
     */
    public void rebuildSelectors() {
        for (EventExecutor e: this) {
            ((NioEventLoop) e).rebuildSelector();
        }
    }
```