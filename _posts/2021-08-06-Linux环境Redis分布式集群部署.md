---
layout:       post
title:        "Linux环境Redis分布式集群部署"
date:         2021-08-06 19:43:00
author:       "Brade"
header-style: text
header-mask:  0.3
catalog:      true
tags:
- Linux
- redis
- redis集群 
- centos7
---


# Linux环境Redis分布集群部署
> 本文中采用的`redids`版本为`6.0.0`，是稳定版，偶数的版本号表示稳定的版本。`Linux`是`centos7`.

### 下载安装 `redis`
1. 下载
> 直接从 [官网](http://download.redis.io) 进行下载上传到服务器或者直接命令下载。
> 这里采用的命令下载：
```shell
  wget http://download.redis.io/releases/redis-6.0.0.tar.gz && tar xzf redis-6.0.0.tar.gz
```
> 如果出现报错异常，一般是`gcc`版本太低导致，`redis6`以上版本，需要`gcc`版本`5.3`以上，执行以下命令升级：
```shell
  yum -y install centos-release-scl && yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils && scl enable devtoolset-9 bash
```

2. 编译
```shell
  cd redis-6.0.0/src && make
```

3. 启动
```shell
  ./redis-server ../redis.conf
```

### 分布式部署
> 本文中采用3台服务器做主节点进行部署，在`3`台服务器上分别下载安装`redis-6.0.0`。
1. 修改 `redis-6.0.0` 目录中的配置文件`redis.conf`。
```shell
  vim redis.conf
```
> 按`i`进入编辑模式，对下面几个参数开启并进行修改。
> +    port 6379
> +    cluster-enabled yes
> +    cluster-config-file nodes.conf
> +    cluster-node-timeout 5000
> +    appendonly yes
> +    bind 0.0.0.0
> 
> 修改完成后按`ESC`，输入`wq`进行保存。

2. 分别启动`3`台服务器中的`redis`。
```shell
  ./redis-server ../redis.conf
```
3. 配置集群节点。
```shell
  cd redis-6.0.0/src && ./redis-cli --cluster create 192.168.26.128:6379 192.168.26.129:6379 192.168.26.130:6379
```
> 命令执行后会打印出一份预想中的配置，如果你觉得没问题的话，就可以输入 `yes`。
> 最后可以得到如下信息：
```shell
  [OK] All 16384 slots covered
```
> 同时`utils/create-cluster` 目录下提供了一个简单的脚本，可以创建启动一个有`3`个主节点和`3`个从节点的`6`节点集群。

4. 开放端口。
> `6379`配置的是供外网访问的端口， `16379`是集群节点通讯的端口。第二个端口是用于集群总线，使用二进制节点到节点的通信通道(gossip 协议)。总线端口的偏移量是固定的，始终为`10000`。
```shell
  firewall-cmd --zone=public --add-port=6379/tcp --permanent && firewall-cmd --zone=public --add-port=16379/tcp --permanent && firewall-cmd --reload
```

5. 测试分布式集群是否成功。
> 先在`192.168.26.128`插入一条数据
```shell
  ./redis-6.0.0/src/redis-cli set test 123456
```
> 然后进入`192.168.26.128`和`192.168.26.130`，检查数据是否同步。
```shell
  ./redis-6.0.0/src/redis-cli get test
```
> 同时可以查询节点信息
```shell
  ./redis-6.0.0/src/redis-cli --cluster info 192.168.26.128:6379
```

### 配置从节点和分片（略）
