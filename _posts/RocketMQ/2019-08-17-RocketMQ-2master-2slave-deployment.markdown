---
title: RocketMQ 2Master-2Slave Deployment
layout: post
date:   2019-08-17
author: "bing5tui3"
tags: [java, rocketMQ]
categories: [rocketMQ]
excerpt_separator: <!--more-->
---

本文介绍如何部署双主双从模式的**RocketMQ**集群

<!--more-->

#### 物理环境准备

首先需要4台机器，我这边虚拟了选用了4台**Linux**作为部署**RocketMQ**的机器。

操作系统我选择了**ArchLinux**，因为其内核较小，运行时占用资源较少。

安装过程可参考[ArchLinux installation]({% post_url 2019-08-11-Arch-linux-installation %})。

我这边4台服务器的静态IP设置为： 

- 10.0.1.21 / 24
- 10.0.1.22 / 24
- 10.0.1.23 / 24
- 10.0.1.24 / 24

接下来设置4台服务器的`/etc/hosts`文件内容为：
（注意，每台机器都将这么配置）

```
10.0.1.21 rocketmq-nameserver1
10.0.1.22 rocketmq-nameserver2
10.0.1.23 rocketmq-nameserver3
10.0.1.24 rocketmq-nameserver4
10.0.1.21 rocketmq-master1
10.0.1.22 rocketmq-master2
10.0.1.23 rocketmq-master1-slave
10.0.1.24 rocketmq-master2-slave
```

也是就说：
1. 4台服务器我们都作为nameserver
2. 地址为21、22的两台机器我们作为broker的master节点，地址为23、24的两台机器作为broker的slave节点。

架构图如下：

![image01](/assets/posts/rocketmq/rocketmq-2master-2slave-deployment/1.png) 

#### RocketMQ安装

**RocketMQ**的安装可以参考之前的 [[单节点部署]({% post_url 2019-06-29-RocketMQ-one-master-no-slave-deployment %})]

注意：这次我将RocketMQ安装在了`/opt/rocketmq`目录下而非之前文章里的`/opt/apache-rocketmq`目录。

还有：不要忘了日志配置的部分！

#### 双主双从节点设置

我们到RocketMQ的安装目录下的`conf`目录，进行双主双从的配置文件编写：

``` bash
cd /opt/rocketmq/rocketmq-4.5.1/conf
```

我们看一下RocketMQ默认提供的配置目录结构：

```
.
+ -- 2m-2s-async/
+ -- 2m-2s-sync/
|  + -- broker-a.properties
|  + -- broker-a-s.properties
|  + -- broker-b.properties
|  + -- broker-b-s.properties
+ -- 2m-noslave/
+ -- dledger/
+ -- broker.conf
+ -- logback_broker.xml
+ -- logback_namesrv.xml
+ -- logback_tools.xml
+ -- plain_acl.yml
+ -- tools.yml
```

从目录名就可以看出，双主双从模式的配置在`2m-2s-async`和`2m-2s-sync`下。

这里我们不得不解释一下**主从同步**和**主从异步**两个模式的区别：

- **同步模式**： 当消息到达broker的`master`节点后，broker并不直接返回给producer一个`ack`消息，而是将数据同步写入到该broker对应的`slave`节点后再返回给producer一个`ack`消息。
- **异步模式**： 相对于同步模式来说，异步模式只要broker的`master`节点确认接收到消息后，马上返回给producer一个`ack`消息。

所以我们这边选择`2m-2s-sync`目录进入，可以看到里面有4个配置文件：

- broker-a.properties 对应`rocketmq-master1`节点的配置
- broker-a-s.properties 对应`rocketmq-master1-slave`节点的配置
- broker-b.properties 对应`rocketmq-master2`节点的配置
- broker-b-s.properties 对应`rocketmq-master2-slave`节点的配置


##### [rocketmq-master1]节点的配置

`broker-a.properties`的文件配置我们修改成如下：

``` bash
#所属集群名字
brokerClusterName=rocketmq-cluster-test
#broker 名字，注意此处不同的配置文件填写的不一样
brokerName=broker-1
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer 地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876;rocketmq-nameserver2:9876;rocketmq-nameserver2:9876
#在发送消息时，自动创建服务器不存在的 topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建 Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4 点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog 每个文件的大小默认 1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue 每个文件默认存 30W 条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/opt/rocketmq/rocketmq-4.5.1/store
#commitLog 存储路径
storePathCommitLog=/opt/rocketmq/rocketmq-4.5.1/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/opt/rocketmq/rocketmq-4.5.1/store/consumequeue
#消息索引存储路径
storePathIndex=/opt/rocketmq/rocketmq-4.5.1/store/index
#checkpoint 文件存储路径
storeCheckpoint=/opt/rocketmq/rocketmq-4.5.1/store/checkpoint
#abort 文件存储路径
abortFile=/opt/rocketmq/rocketmq-4.5.1/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制 Master
#- SYNC_MASTER 同步双写 Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```

大部分的配置都如注释所描述的，其中几个需要特别强调一下：
1. `brokerRole`设置成同步
2. `flushDiskType`设置成异步刷盘
3. `namesrvAddr`将前面的nameserver1、2、3、4都配置进去
4. 存储路径配置按实际的配

##### [rocketmq-master1-slave]节点的配置

`broker-a-s.properties`的文件配置我们修改成如下（只列出和`broker-a.properties`中不同的地方）：

```bash
...
brokerId=1
...
brokerRole=SLAVE
...
```

这里：
1. `brokerId`设置为1，表示是第一个从节点
2. `brokerRole`设置为**`SLAVE`**表示是从节点

##### [rocketmq-master2]节点的配置

`broker-b.properties`的文件配置我们修改成如下（只列出和`broker-a.properties`中不同的地方）：

```bash
...
brokerName=broker-2
...
```

这里：
只需要修改`brokerName`为broker-2即可，其他的配置和`rocketmq-master1`的是一样的。

##### [rocketmq-master2-slave]节点的配置

`broker-b-s.properties`的文件配置我们修改成如下（只列出和`broker-b.properties`中不同的地方）：

```bash
...
brokerId=1
...
brokerRole=SLAVE
...
```

#### RocketMQ集群启动

##### NameServer进程启动

首先我们4台服务器都启动**nameserver**进程：

```bash
cd /opt/rocketmq/rocketmq-4.5.1/bin
nohup sh mqnamesrv &
```

##### Broker进程启动

4台服务器都进入如下目录：

```bash
cd /opt/rocketmq/rocketmq-4.5.1/bin
```

分节点进行**broker**的启动（注意配置文件的对应关系）：

1. [rocketmq-master1]节点启动
```bash
nohup sh mqbroker -c /opt/rocketmq/rocketmq-4.5.1/conf/2m-2s-sync/broker-a.properties  > /dev/null 2>&1 &
```

2. [rocketmq-master2]节点启动
```bash
nohup sh mqbroker -c /opt/rocketmq/rocketmq-4.5.1/conf/2m-2s-sync/broker-b.properties  > /dev/null 2>&1 &
```

3. [rocketmq-master1-slave]节点启动
```bash
nohup sh mqbroker -c /opt/rocketmq/rocketmq-4.5.1/conf/2m-2s-sync/broker-a-s.properties  > /dev/null 2>&1 &
```

4. [rocketmq-master2-slave]节点启动
```bash
nohup sh mqbroker -c /opt/rocketmq/rocketmq-4.5.1/conf/2m-2s-sync/broker-b-s.properties  > /dev/null 2>&1 &
```

完成启动后可以通过`jps`检查进程启动状况。


#### systemd服务编写

为了后续管理方便，此小节给出**`systemd`**服务的编写例子。

**broker**的`systemd`服务文件`/etc/systemd/system/broker.service`：

```bash
[Unit]
Description=Broker Service
#Requires=namesrv.service
[Service]
Type=simple
Environment="JAVA_HOME=/opt/java/jdk1.8.0_221/"
ExecStart=sh /opt/rocketmq/rocketmq-4.5.1/bin/mqbroker -c /opt/rocketmq/rocketmq-4.5.1/conf/2m-2s-sync/[broker-xxx.properties]  > /dev/null 2>&1 &
ExecStop=sh /opt/rocketmq/rocketmq-4.5.1/bin/mqshutdown broker
[Install]
WantedBy=multi-user.target
```

注意其中的`broker-xxx.properties`替换为每台服务器自己对应的配置文件。


**nameserver**的`systemd`服务文件`/etc/systemd/system/namesrv.service`：

```bash
[Unit]
Description=Nameserver Service
[Service]
Type=simple
Environment="JAVA_HOME=/opt/java/jdk1.8.0_221/"
ExecStart=sh /opt/rocketmq/rocketmq-4.5.1/bin/mqnamesrv &
ExecStop=sh /opt/rocketmq/rocketmq-4.5.1/bin/mqshutdown namesrv
[Install]
WantedBy=multi-user.target
```

加载配置，设置开机自动启动，并启动服务：

```bash
# 重新加载配置
systemctl daemon-relaod
# 开启开机自动启动
systemctl enable namesrv
systemctl enable broker
# 启动nameserver和broker的服务
systemctl start namesrv
systemctl start broker
```