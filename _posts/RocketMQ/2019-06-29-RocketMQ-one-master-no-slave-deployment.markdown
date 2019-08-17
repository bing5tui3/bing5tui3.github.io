---
title: RocketMQ One-Master-No-Slave Deployment
layout: post
date:   2019-06-29
author: "bing5tui3"
tags: [java, rocketMQ]
categories: [rocketMQ]
excerpt_separator: <!--more-->
---

本文记录如何在**Linux**上搭建一个单点的**RocketMQ**环境。

**RocketMQ**的门户网站：[https://rocketmq.apache.org/](https://rocketmq.apache.org/)

**RocketMQ**的GitHub：[https://github.com/apache/rocketmq/](https://github.com/apache/rocketmq/)

<!--more-->

#### *RocketMQ*构建

我选择从**`Github`**克隆源码进行构建，因为之后学习也需要对着源码进行。 

当然也可以选择从**`RocketMQ`**门户下载源码的**zip**包。

我这边使用的版本是**4.5.1**，官网目前的Release版本应该是**4.4.0**（20190629）。

##### 克隆源码

克隆源码 （我这边的环境是WSL，可以自行找目录存放源码）：

```bash
mkdir /mnt/c/Code # windows下的"C:\Code\"目录
cd /mnt/c/Code
git clone https://github.com/apache/rocketmq.git
```

##### 项目构建

源码克隆完成后进入目录进行构建：

```bash
cd /mnt/c/Code/rocketmq 
```

使用官网的命令进行构建：

``` bash
mvn -Prelease-all -DskipTests clean install -U
```

构建也需要一段时间。

构建完成后，我们进入`./distribution/target/apache-rocketmq`目录下：

```bash
cd distribution/target/apache-rocketmq
ls -la
```
在这里我们可以找到构建好的**`RocketMQ`**的**tar**包和**zip**包：

**rocketmq-4.5.1.tar.gz**

**rocketmq-4.5.1.zip**

到这里为止，**`RocketMQ`**的构建就完成了，当然你也可以直接下载构建好的压缩包。

#### *RocketMQ*部署

接下来我们将部署单一节点的**RocketMQ**，选择一台Linux机器，可以是虚拟机也可以是物理机。

所谓单点，即单个节点的**NameServer**以及单个节点的**Broker**，并且**Broker**只部署主节点。

*注：本文的后续步骤需要切换至root用户执行*

##### 上传并解压

先建立一个目录用于存放上传的RocketMQ压缩包：

```bash
mkdir ~/download
cd ~/download
```
在该目录下执行rz命令将**rocketmq-4.5.1.tar.gz**上传。

接下来创建RocketMQ的安装目录，将**RocketMQ**解压至安装目录：

```bash
mkdir /opt/apache-rocketmq
tar xzvf ~/download/rocketmq-4.5.1.tar.gz -C /opt/apache-rocketmq
cd /opt/apache-rocketmq/rocketmq-4.5.1/
```

解压完成之后，可以看到其目录结构为：
```
.
+ -- LICENSE
+ -- NOTICE
+ -- README.md
+ -- benchmark/
+ -- bin/
+ -- conf/
|  + -- 2m-2s-async/
|  + -- 2m-2s-sync/
|  + -- 2m-noslave/
|  + -- dledger/
|  + -- broker.conf
|  + -- logback_broker.xml
|  + -- logback_namesrv.xml
|  + -- logback_tools.xml
|  + -- plain_acl.yml
|  + -- tools.yml
+ -- lib/
```

##### 编写*Broker*配置文件

接下来，我们要编写**broker**的一个配置文件，由于我这边使用单点的方式部署，所以将配置文件的目录命名为*1m-noslave*。

在`conf`目录下建立一个`1m-noslave`的目录：

```bash
mkdir conf/1m-noslave
```

*1m-noslave*的含义是，一个主节点（master），无从节点（slave）。

创建一个配置文件：
```bash
vim /conf/1m-noslave/broker.properties
```
按照如下内容进行配置：
```bash
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker 名字，注意此处不同的配置文件填写的不一样
brokerName=broker-1
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer 地址，分号分割
namesrvAddr=rocketmq-nameserver:9876
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
storePathRootDir=/opt/apache-rocketmq/rocketmq-4.5.1/store
#commitLog 存储路径
storePathCommitLog=/opt/apache-rocketmq/rocketmq-4.5.1/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/opt/apache-rocketmq/rocketmq-4.5.1/store/consumequeue
#消息索引存储路径
storePathIndex=/opt/apache-rocketmq/rocketmq-4.5.1/store/index
#checkpoint 文件存储路径
storeCheckpoint=/opt/apache-rocketmq/rocketmq-4.5.1/store/checkpoint
#abort 文件存储路径
abortFile=/opt/apache-rocketmq/rocketmq-4.5.1/store/abort
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
brokerRole=ASYNC_MASTER
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

注意配置文件中的这些目录：

```
/opt/apache-rocketmq/rocketmq-4.5.1/store
/opt/apache-rocketmq/rocketmq-4.5.1/store/commitlog
/opt/apache-rocketmq/rocketmq-4.5.1/store/consumequeue
```

我们可以事先为其建立好目录：

```bash
mkdir -p /opt/apache-rocketmq/rocketmq-4.5.1/store/commitlog
mkdir -p /opt/apache-rocketmq/rocketmq-4.5.1/store/consumequeue
mkdir -p /opt/apache-rocketmq/rocketmq-4.5.1/store/index
```

##### 配置日志信息

接下来我们再为**RocketMQ**建立日志目录：

```bash
mkdir /opt/apache-rocketmq/rocketmq-4.5.1/logs
```
并修改`conf`目录下**logback**配置中的日志路径配置：

```bash
cd /opt/apache-rocketmq/rocketmq-4.5.1/conf
# 替换*.xml中所有"${user.home}"字符串为"/opt/apache-rocketmq/rocketmq-4.5.1"
sed -i 's#${user.home}#/opt/apache-rocketmq/rocketmq-4.5.1#g' *.xml
```

接下来我们在`/etc/hosts`中配置一下**nameserver**的DNS：
```
127.0.0.1 rocketmq-nameserver
```

OK，到此我们单点的**broker**和**nameserver**的启动前配置都准备好了。

#### *RocketMQ*的启动

完成了配置之后，我们可以进行**RocketMQ**的启动了。

**RocketMQ**的启动过程是要先启动**NameServer**，再启动**Broker**。

##### 启动*NameServer*

```bash
cd /opt/apache-rocketmq/rocketmq-4.5.1/bin
nohup sh mqnamesrv &
```

可以通过`jps`命令查看**NameServer**是否启动成功：
```bash
jps | grep Namesrv
```

如果出现类似如下信息则启动成功：
```
xxxxx NamesrvStartup
```

##### 启动*Broker*

```bash
cd /opt/apache-rocketmq/rocketmq-4.5.1/bin
# 通过-C 指定配置文件的路径，这个配置就是之前写的broker配置
nohup sh mqbroker -C /opt/apache-rocketmq/rocketmq-4.5.1/conf/1m-noslave/broker.properties  > /dev/null 2>&1 &
```

通过`jps`命令查看**Broker**是否启动成功：

```bash
jps | grep Broker
```

如果出现类似如下信息则启动成功：

```
xxxxx BrokerStartup
```

#### 结语

至此我们在单一节点上完成**RocketMQ**的配置和启动。