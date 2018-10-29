---
title: MTProxy for Telegram on CentOS
layout: post
date:   2018-10-28
author: "bing5tui3"
tags: [proxy]
categories: [proxy]
excerpt_separator: <!--more-->
---

**`Telegram`**在今年6月更新后在**`iOS`**上就无法连接了，即使挂了科学上网软件，貌似还有部分连接无法被代理软件捉取。无奈只能用**TG**官方的**`MTProxy`**来为**`Telegram`**代理了。

**`Github`**:**https://github.com/TelegramMessenger/MTProxy**

本文记录了**`MTProxy`**服务器的搭建过程。

<!--more-->

#### 构建

首先需要安装一些依赖包，用于后续的构建过程。

```bash
yum install openssl-devel zlib-devel
yum groupinstall "Development Tools"
```

创建一个目录`/opt/mtproxy`：
```bash
mkdir -p /opt/mtproxy && cd /opt/mtproxy
```

克隆源码：
```bash
git clone https://github.com/TelegramMessenger/MTProxy
cd MTProxy
```

接下来进行编译构建，构建后的二进制文件在`./objs/bin/mtproto-proxy`：
```bash
make
```

#### 配置和运行

首先，从Telegram官方获取一个密钥`proxy-secret`，用来连接Telegram的服务器：
```bash
curl -s https://core.telegram.org/getProxySecret -o proxy-secret
```

获取Telegram的服务器地址配置`proxy-multi.conf`，因为偶尔该配置会变动，官方建议每天更新：
```bash
curl -s https://core.telegram.org/getProxyConfig -o proxy-multi.conf
```

生成一个密钥供该服务器给客户端使用：
```bash
head -c 16 /dev/urandom | xxd -ps
```
生成后的密钥请保存下来，之后在服务器运行和客户端配置的时候都需要使用。

如此之后我们将拥有三个关键文件（`pwd = /opt/mtproxy/MTProxy/`：
1. 二进制可执行文件： `./objs/bin/mtproto-proxy`
2. 与Telegram服务器连接所需要的密钥： `./proxy-secret`
3. Telegram服务器的地址列表配置文件： `./proxy-multi.conf`

运行`mtproto-proxy`：
```bash
./objs/bin/mtproto-proxy -u nobody -p 8888 -H 9527 -S <secret> --aes-pwd proxy-secret proxy-multi.conf -M 1
```

其中：
- `nobody`是用户名
- `9527`是该代理服务器供客户端连接的端口号
- `8888`是本地端口，该端口只开在`loopback`回路上，可以获取一些统计信息：`wget localhost:8888/stats`
- `<secret>`是前面生成的密钥，可以设置多个密钥：`-S <secret1> -S <secret2>`
- `1`是worker数量，一般自用的化只需要设置成1就行
- 如果购买的服务器是在内网的，可以通过添加`--nat-info <privateIP>:<publicIP>`将其暴露

#### systemd配置

创建systemd的配置文件：
```bash
vim /etc/systemd/system/MTProxy.service
```

编辑配置文件：
```bash
[Unit]
Description=MTProxy
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/mtproxy/MTProxy
ExecStart=/opt/mtproxy/MTProxy/objs/bin/mtproto-proxy -u nobody -p 8888 -H 9527 -S <secret> -P <proxy tag> <other params>
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

重载配置文件：
```bash
systemctl daemon-reload
```

启动服务：
```bash
systemctl restart MTProxy
```

查看服务状态：
```bash
systemctl status MTProxy
```

设置成开机自动启动：
```bash
systemctl enable MTProxy
```

#### 结语

Enjoy!