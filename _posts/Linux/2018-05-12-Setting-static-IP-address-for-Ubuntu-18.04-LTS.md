---
title:  "Setting static IP address for Ubuntu 18.04 LTS"
layout: post
date:   2018-05-12
author: "bing5tui3"
tags: [linux]
excerpt_separator: <!--more-->
---
虚拟机安装了18.04新版本的Ubuntu，但是让Cisco路由器分配静态IP的时候，无论使用 `hardware-address` 还是 `client-identifier` 都无法实现DHCP分配静态IP的问题，所以只能让系统自行设置静态IP。本文记录一下18.04下如何在系统内设置静态IP。
<!--more-->

18.04的静态IP目前我没找到如果通过dhcp的方式由dhcp服务器下发，只能通过机器自行设置：

编辑 `/etc/netplan/*.yaml`：
~~~ yaml
network:
    ethernets:
        enp0s3:
            addresses: [10.0.1.41/24]
            dhcp4: no
            optional: true
            gateway4: 10.0.1.1
            nameservers:
                addresses: [192.168.1.1]
    version: 2
    renderer: networkd
~~~
