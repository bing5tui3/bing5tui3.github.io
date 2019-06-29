---
title: RocketMQ environment build
layout: post
date:   2019-06-29
author: "bing5tui3"
tags: [java, rocketMQ]
categories: [rocketMQ]
excerpt_separator: <!--more-->
---

本文记录如何在Linux上搭建一个单点的RocketMQ环境。

<!--more-->

#### RocketMQ构建

我选择从**`Github`**克隆源码进行构建，因为之后学习也需要对着源码进行。 

当然也可以选择从**`RocketMQ`**门户下载源码的**zip**包。

我这边使用的版本是**4.5.1**，官网目前的Release版本应该是**4.4.0**（20190629）。

克隆源码 （我这边的环境是WSL）：

```bash
mkdir /mnt/c/Code # windows下的"C:\Code\"目录
cd /mnt/c/Code
git clone https://github.com/apache/rocketmq.git
```

克隆可能需要一段时间，完成后进入目录进行构建：

```bash
cd rocketmq
mvn -Prelease-all -DskipTests clean install -U
```

构建也需要一段时间（这里需要安装好**`maven`**构建工具）。

构建完成后，我们进入`./distribution/target/apache-rocketmq`目录下：

```bash
cd distribution/target/apache-rocketmq
ls -la
```
在这里我们可以找到构建好的**`RocketMQ`**的**tar**包和**zip**包。

到这里为止，**`RocketMQ`**的构建就完成了，当然你也可以直接下载构建好的压缩包（官网有链接）。

#### RocketMQ部署


