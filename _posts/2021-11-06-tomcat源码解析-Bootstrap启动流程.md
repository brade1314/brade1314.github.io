---
layout:       post
title:        "tomcat源码解析之Bootstrap启动流程"
date:         2021-11-06 17:33:00
author:       "Brade"
header-style: text
header-mask:  0.3
catalog:      true
tags:
    - tomcat
    - Java
    - 源码
---

# tomcat 源码解析
本文中 `tomcat` 采用的版本为 `10.0.12`。

## 1. `bootstrap` 启动
- 1.1 静态块获取 `catalina.base` 变量值 ;
- 1.2 调用 `Bootstrap.main()` , 里面再调用 `init()` , `setAwait(true)` ,`load()` , `start()` .

## 2. 具体启动流程
- 2.1 `init()`  
> 初始化类加载器 `ClassLoader` , 指定为 `commonLoader` , 加载 `catalina.home` 下 `lib` 包中所有 `jar` 文件 ; 
> `catalinaLoader` , `sharedLoader` 指向 `commonLoader` ; 初始化 `catalinaDaemon` 为 `Catalina` .
    
- 2.2 `setAwait(true)`
> 调用 `Catalina.setAwait()` 方法 , 设置 `await` 状态为 `true` .

- 2.3 `load()`
> 调用 `Catalina.load()` 方法 , 初始化 `Naming` , 解析 `server.xml` 为 `Server` 对象 , 主要逻辑 `Catalina.createStartDigester()` ;
> 通过模板方法 `LifecycleBase.initInternal()` , 陆续调用 `StandardServer` , `StandardService` , `StandardEngine` , `StandardThreadExecutor` , `MapperListener` , `Connector` 的 `initInternal()` 方法 ;
> 最后调用 `NioEndpoint.initServerSocket()` , 使用 `NIO` 的 `ServerSocketChannel.bind` 方法绑定端口地址 .

- 2.4 `start()` 
> 调用 `Catalina.start()` 方法 ,
> 再通过模板方法 `LifecycleBase.startInternal()` 流程和上面的 `load()` 方法类似 , 
> 最后调用 `NioEndpoint.serverSocketAccept()` , 使用 `NIO` 的 `ServerSocketChannel.accept()` 接受连接 .