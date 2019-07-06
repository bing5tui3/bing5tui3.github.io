---
title: RocketMQ Simple Message Example
layout: post
date:   2019-07-06
author: "bing5tui3"
tags: [java, rocketMQ]
categories: [rocketMQ]
excerpt_separator: <!--more-->
---

本文介绍了如何使用RocketMQ进行发送消息和接受消息：

- 通过RocketMQ生产**可靠同步消息**，**可靠异步消息**和**单向消息**。
- 通过RocketMQ消费消息。

<!--more-->

#### 初始化本地开发环境

##### 添加依赖

首先通过自己熟悉的方式创建一个Demo项目。

在项目中添加依赖：

```
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.5.1</version>
</dependency>
```

注意这边的`version`需要和你实际部署的**RocketMQ**版本一致。

##### 基本配置

配置**Host**（我的**RocketMQ**部署地址是10.0.1.15）：
``` bash
# rocketMQ
10.0.1.15 rocketmq-broker-master
10.0.1.15 rocketmq-namesrv
```

项目中创建如下包结构：
```
- src
+ -- main
|  + -- java
|     + -- common     # 公共类，如常量类等
|     + -- consumer   # 消费者
|     + -- producer   # 生产者
——
```

在**common**包下，我们创建一个常量类，保存一些配置信息：

``` java
public class Constants {
    // rocketmq-namesrv 是rocketMQ NameServer的Ip地址，在我这边是10.0.1.15
    public static final String NAME_SERVER_ADDRESS = "rocketmq-namesrv:9876";
    // Topic
    public static final String TOPIC_TEST = "B5T3-TestTopic";
    // Tag
    public static final String TAG_TEST = "B5T3-TestTag";
}
```

这边我们配置了**NameServer**服务器地址和端口，**Topic**名称，**Tag**名称。


#### 编写一个简单的消费者

``` java
public class SimpleConsumer {
    public static void main(String[] args) throws Exception {
        // 通过给定的消费者组名称初始化消费者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("simple_consumer_group");
        // 设置NameServer地址
        consumer.setNamesrvAddr(Constants.NAME_SERVER_ADDRESS);
        // 订阅B5T3-TestTopic下的所有Tag消息
        consumer.subscribe(TOPIC_TEST, "*");
        // 注册一个并发监听器
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                // 简单打印消息体
                msgs.forEach(msg -> System.out.println(new String(msg.getBody())));
                // 返回成功
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 消费者启动
        consumer.start();

        System.out.println("Consumer started!");
    }
}
```

#### 编写三种类型的生产者

生产者有三种基本的类型：

- 同步发送消息的生产者
- 异步发送消息的生产者
- 单向发送消息的生产者（无需消费者响应）

##### 同步发送消息的生产者

在**producer**包下新建一个`SimpleSyncProducer`类：

``` java
public class SimpleSyncProducer {
    public static void main(String[] args) throws Exception {
        // 通过生产者组名称来初始化一个生产者
        DefaultMQProducer producer = new DefaultMQProducer("simple_sync_producer_group");
        // 设置NameServer地址
        producer.setNamesrvAddr(Constants.NAME_SERVER_ADDRESS);
        // 生产者启动
        producer.start();
        // 发送100条消息
        for (int i = 0; i < 100; i++) {
            // 消息实体组装
            Message message = new Message(
                    Constants.TOPIC_TEST, /* TOPIC */
                    Constants.TAG_TEST,   /* TAG */
                    ("Hello World from SyncProducer, message count: " + i)
                            .getBytes(RemotingHelper.DEFAULT_CHARSET)
            );
            // 同步发送直接调用send方法
            SendResult result = producer.send(message);
            // 打印返回
            System.out.println(result);
        }
        // 关闭生产者
        producer.shutdown();
    }
}
```

##### 异步发送消息的生产者

在**producer**包下新建一个`SimpleAsyncProducer`类：

``` java
public class SimpleAsyncProducer {
    public static void main(String[] args) throws Exception {
        // 通过生产者组名称初始化一个生产者
        DefaultMQProducer producer = new DefaultMQProducer("simple_async_producer_group");
        // 设置NameServer地址
        producer.setNamesrvAddr(Constants.NAME_SERVER_ADDRESS);
        // 生产者启动
        producer.start();
        // 失败不重试
        producer.setRetryTimesWhenSendAsyncFailed(0);
        // 发送100条消息
        for (int i = 0; i < 100; i++) {
            final int index = i;
            // 组装消息
            Message message = new Message(
                    Constants.TOPIC_TEST,
                    Constants.TAG_TEST,
                    "Order " + i,
                    ("Hello World From AsyncProducer " + i)
                            .getBytes(RemotingHelper.DEFAULT_CHARSET)
            );
            // 异步发送消息，设置一个callBack函数
            producer.send(message, new SendCallback() {
                /**
                 * 发送成功打印消息ID
                 * @param sendResult
                 */
                @Override
                public void onSuccess(SendResult sendResult) {
                    System.out.printf("%-10d OK %s %n", index, sendResult.getMsgId());
                }

                /**
                 * 发送失败打印错误堆栈
                 * @param e
                 */
                @Override
                public void onException(Throwable e) {
                    System.out.printf("%-10d Exception %s %n", index, e);
                    e.printStackTrace();
                }
            });
        }
        // 等待1秒
        Thread.sleep(1000);
        // 关闭生产者
        producer.shutdown();
    }
}
```

##### 单向发送消息的生产者

在**producer**包下新建一个`SimpleOnewayProducer`类：

``` java
public class SimpleOnewayProducer {
    public static void main(String[] args) throws Exception {
        // 通过生产者组名称初始化一个生产者
        DefaultMQProducer producer = new DefaultMQProducer("simple_oneway_producer_group");
        // 设置NameServer地址
        producer.setNamesrvAddr(Constants.NAME_SERVER_ADDRESS);
        // 生产者启动
        producer.start();
        // 发送100条消息
        for (int i = 0; i < 100; i++) {
            // 组装消息
            Message message = new Message(
                    Constants.TOPIC_TEST,
                    Constants.TAG_TEST,
                    ("Hello World From OnewayProducer " + i)
                            .getBytes(RemotingHelper.DEFAULT_CHARSET)
            );
            // 发送单向消息
            producer.sendOneway(message);
        }
        // 关闭生产者
        producer.shutdown();
    }
}
```

#### 测试

可以将生产者和消费者的`main`方法分别运行一下，看看打印的效果。

由于**RocketMQ**的消息在**Broker**端是持久化存储的，所以，无论先启动**Consumer**还是先启动**Producer**，消息都不会丢失，也就是最后**Consumer**都将消费到消息。