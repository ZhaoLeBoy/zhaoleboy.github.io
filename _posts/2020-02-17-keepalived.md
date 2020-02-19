---
layout:     post
title:      "Keepalived基本安装"
subtitle:   "Keepalived基本安装"
date:        2020-02-17
author:     "ZhaoLe"
header-img: "img/nginx/post-bg.png"
catalog: true
tags:
    - Keepalived
---

#### 一.介绍
首先来一段keepalived介绍
>Keepalived的作用是检测服务器的状态，如果有一台web服务器宕机，或工作出现故障，Keepalived将检测到，并将有故障的服务器从系统中剔除，同时使用其他服务器代替该服务器的工作，当服务器工作正常后Keepalived自动将服务器加入到服务器群中，这些工作全部自动完成，不需要人工干涉，需要人工做的只是修复故障的服务器。


#### 二.环境

|  名称    | 版本号  |
| --- | --- |
| CentOS | 7.x  |
| keepalived | 2.0.2 |
  

#### 三.keepalived安装

1. 安装相关依赖
>yum -y install libnl libnl-devel 相关依赖

2. 先将本地的keepalived传入到虚拟服务器中并且解压
>tar -zvxf keepalived-2.0.20.tar.gz

3. 进入解压后的文件夹，先进行配置
> ./configure --prefix=/usr/local/keepalived --sysconf=/etc
    `sysconf 是keepalived的核心配置文件所在位置，这个位置是固定在etc下`

4. 编译和安装
> make & make install

5. keepalived的启动文件和配置文件是在两个地方
> 核心配置文件 /etc/keepalived/keepalived.conf \
> 启动文件 /usr/local/keepalived/sbin/keepalived

一般来说安装Keepalived伴随这集群来的，具体根据当前是什么架构再进行配置,例如：

**1. [Nginx+Keepalived 双机主备，双机热备](!http://jinlipool.com/2020/02/18/keepalived_nginx)**

**2. [LVS(DR模式)+Keepalived(主备)+Nginx](!http://jinlipool.com/2020/02/18/lvs_keepalived_nginx)**

#### 四. 将Keepalived作为系统服务启动
1. 先回到解压后`keepalived-2.0.20`文件夹中。在进入里面的`keepalived`文件夹
> cp ./etc/init.d/keepalived /etc/init.d/ \
> cp ./etc/sysconfig/keepalived /etc/sysconfig/ \
> 如果有提示是否覆盖，直接覆盖就可以了 \

2. 刷新，使新放入系统的命令生效
>systemctl daemon-reload

3. keepalived相关操作
> systemctl start keepalived.service \
> systemctl stop keepalived.service \
> systemctl restart keepalived.service 

4. 最后查看是否启动成功
>ps -ef \| grep keepalived

