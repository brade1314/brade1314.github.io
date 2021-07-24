---
layout:       post
title:        "netty源码解析"
subtitle:     "ServerBootstrap"
date:         2021-07-06 13:00:00
author:       "Brade"
header-style: text
header-mask:  0.3
catalog:      true
tags:
    - netty
    - 多线程
    - 源码
    - ServerBootstrap
---

# ServerBootstrap 源码解析
本文中 `netty` 采用的版本为 `4.1.66.Final-SNAPSHOT`。`ServerBootstrap` 的`类图`如下所示：
> ![img.png](/img/netty/ServerBootstrap.png)

## AbstractBootstrap
抽象类，实现了`Cloneable`接口。
> 构造器
```java
    AbstractBootstrap(AbstractBootstrap<B, C> bootstrap) {
        // 事件循环池 eventLoopGroup
        group = bootstrap.group;
        // channel 工厂
        channelFactory = bootstrap.channelFactory;
        // channel 处理器
        handler = bootstrap.handler;
        // socket 本地地址
        localAddress = bootstrap.localAddress;
        // channel 属性列表，LinkedHashMap
        synchronized (bootstrap.options) {
            options.putAll(bootstrap.options);
        }
        // Bootstrap 参数表
        attrs.putAll(bootstrap.attrs);
    }
```
> `bind` 方法实际实现：
```java
    /**
     * 注册 channel 绑定操作实际功能实现方法
     * @param localAddress
     * @return
     */
    private ChannelFuture doBind(final SocketAddress localAddress) {
        // 注册 channel 到 eventLoopGroup
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }
        // 注册成功
        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            // 此时我们就知道注册完成并且成功了。
            ChannelPromise promise = channel.newPromise();
            // 直接调用绑定方法
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            // 增加监听器来判断是否注册成功
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }

    /**
     * 创建 channel 并注册 到 eventLoopGroup 具体实现方法
     * @return
     */
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            // 初始化 channel ，实际调用的子类 ServerBootstrap.init(channel)
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        // 调用 eventLoopGroup的注册方法
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }

    abstract void init(Channel channel) throws Exception;

    /**
     * channel 绑定端口并添加监听
     * @param regFuture
     * @param channel
     * @param localAddress
     * @param promise
     */
    private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    // 调用Channel接口的bind方法，最终实现在AbstractChannel的AbstractUnsafe.bind
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }
```
```java
    static final class PendingRegistrationPromise extends DefaultChannelPromise {

        // Is set to the correct EventExecutor once the registration was successful. Otherwise it will
        // stay null and so the GlobalEventExecutor.INSTANCE will be used for notifications.
        private volatile boolean registered;

        PendingRegistrationPromise(Channel channel) {
            super(channel);
        }

        void registered() {
            registered = true;
        }

        @Override
        protected EventExecutor executor() {
            if (registered) {
                // If the registration was a success executor is set.
                //
                // See https://github.com/netty/netty/issues/2586
                return super.executor();
            }
            // The registration failed so we can only use the GlobalEventExecutor as last resort to notify.
            return GlobalEventExecutor.INSTANCE;
        }
    }
```
> 创建 `channel` 实例，在boss:
```java
    /**
     * 用于创建  Channel 实例，在子类 ServerBootstrap中调用
     * @param channelClass
     * @return
     */
    public B channel(Class<? extends C> channelClass) {
        return channelFactory(new ReflectiveChannelFactory<C>(
                ObjectUtil.checkNotNull(channelClass, "channelClass")
        ));
    }
```

> 设置 `ServerSocketChannel` 的属性和参数列表
```java
    /**
     * 设置 ServerSocketChannel 属性
     * @param option
     * @param value
     * @param <T>
     * @return
     */
    public <T> B option(ChannelOption<T> option, T value) {
        ObjectUtil.checkNotNull(option, "option");
        synchronized (options) {
            if (value == null) {
                options.remove(option);
            } else {
                options.put(option, value);
            }
        }
        return self();
    }
```
```java
/**
     * 设置 ServerSocketChannel 参数
     * @param key
     * @param value
     * @param <T>
     * @return
     */
    public <T> B attr(AttributeKey<T> key, T value) {
        ObjectUtil.checkNotNull(key, "key");
        if (value == null) {
            attrs.remove(key);
        } else {
            attrs.put(key, value);
        }
        return self();
    }
```

## ServerBootstrap
普通类，继承了 [AbstractBootstrap](#AbstractBootstrap) ，扩展了一些方法和属性。
> 其中最常用的 `group` 方法：
```java
    /**
     * 为父级（接受者）和子级（客户端）设置 EventLoopGroup。
     * 这些EventLoopGroup 用于处理 ServerChannel 的所有事件和 IO 和 Channel
     */
    public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
        // 调用父类的 group 方法
        super.group(parentGroup);
        if (this.childGroup != null) {
            throw new IllegalStateException("childGroup set already");
        }
        this.childGroup = ObjectUtil.checkNotNull(childGroup, "childGroup");
        return this;
    }
```
> 设置 `SocketChannel` 的属性和参数列表：
```java
/**
     * 设置 SocketChannel 属性
     * @param childOption
     * @param value
     * @param <T>
     * @return
     */
    public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value) {
        ObjectUtil.checkNotNull(childOption, "childOption");
        synchronized (childOptions) {
            if (value == null) {
                childOptions.remove(childOption);
            } else {
                childOptions.put(childOption, value);
            }
        }
        return this;
    }
```
```java
/**
     * 设置 SocketChannel 参数
     * @param childKey
     * @param value
     * @param <T>
     * @return
     */
    public <T> ServerBootstrap childAttr(AttributeKey<T> childKey, T value) {
        ObjectUtil.checkNotNull(childKey, "childKey");
        if (value == null) {
            childAttrs.remove(childKey);
        } else {
            childAttrs.put(childKey, value);
        }
        return this;
    }
```
> 重写了父类的 `AbstractBootstrap` 的 `init()` 方法：
```java
    /**
     * 重写父类的初始化 channel 方法，被父类 initAndRegister() 方法中调用
     * @param channel
     */
    @Override
    void init(Channel channel) {
        // 设置channel 属性
        setChannelOptions(channel, newOptionsArray(), logger);
        // 设置channel 参数
        setAttributes(channel, newAttributesArray());

        // 管道
        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions = newOptionsArray(childOptions);
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs = newAttributesArray(childAttrs);

        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                // 为每个一个 ServerSocketChannel 管道增加处理器
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        // 为每个一个 SocketChannel 管道增加处理器
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```
> 设置 `SocketChannel` 的 `ServerBootstrapAcceptor`:
```java
    private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter {

        private final EventLoopGroup childGroup;
        private final ChannelHandler childHandler;
        private final Entry<ChannelOption<?>, Object>[] childOptions;
        private final Entry<AttributeKey<?>, Object>[] childAttrs;
        private final Runnable enableAutoReadTask;

        ServerBootstrapAcceptor(
                final Channel channel, EventLoopGroup childGroup, ChannelHandler childHandler,
                Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
            this.childGroup = childGroup;
            this.childHandler = childHandler;
            this.childOptions = childOptions;
            this.childAttrs = childAttrs;

            // Task which is scheduled to re-enable auto-read.
            // It's important to create this Runnable before we try to submit it as otherwise the URLClassLoader may
            // not be able to load the class because of the file limit it already reached.
            //
            // See https://github.com/netty/netty/issues/1328
            enableAutoReadTask = new Runnable() {
                @Override
                public void run() {
                    channel.config().setAutoRead(true);
                }
            };
        }

        @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg;

            // 为管道增加处理器
            child.pipeline().addLast(childHandler);

            // 设置属性和参数
            setChannelOptions(child, childOptions, logger);
            setAttributes(child, childAttrs);

            try {
                // 注册channel
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }

        private static void forceClose(Channel child, Throwable t) {
            child.unsafe().closeForcibly();
            logger.warn("Failed to register an accepted channel: {}", child, t);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            final ChannelConfig config = ctx.channel().config();
            if (config.isAutoRead()) {
                // stop accept new connections for 1 second to allow the channel to recover
                // See https://github.com/netty/netty/issues/1328
                config.setAutoRead(false);
                ctx.channel().eventLoop().schedule(enableAutoReadTask, 1, TimeUnit.SECONDS);
            }
            // still let the exceptionCaught event flow through the pipeline to give the user
            // a chance to do something with it
            ctx.fireExceptionCaught(cause);
        }
    }
```