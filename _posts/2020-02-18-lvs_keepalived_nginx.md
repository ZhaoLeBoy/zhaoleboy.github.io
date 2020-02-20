---
layout:     post
title:      "LVS(DR模式)+Keepalived(主备)+Nginx"
subtitle:   "LVS(DR模式)+Keepalived(主备)+Nginx"
date:        2020-02-18
author:     "ZhaoLe"
header-img: "img/nginx/post-bg.png"
catalog: true
tags:
    - Keepalived Nginx
---


#### 一.环境配置
  

|  名称    | 版本号  |
| --- | --- |
| CentOS | 7.x  |
| keepalived | 2.0.2 |
| Nginx | 1.16.0 |
  
使用LVS(DR模式)，Keepalived主从模式进行搭建，具体配置如下

![1](/img/nginx/lvs_nginx_keepalived/1.png)

### 二.安装Nginx
现在两台RS(Real Server)上安装Nginx.

[Nginx基本安装](http://jinlipool.com/2020/02/17/nginx/)

### 三.配置网卡规则和路由
安装完Nginx后紧接着对两台RS进行配置。由于是基于LVS的DR模式

#### 配置lo网卡

>cd /etc/sysconf/network-scripts/ \
>cp ifcfg-lo ifcfg-lo:v1

```shell
#名称
DEVICE=lo:v1
#虚拟ip
IPADDR=192.168.1.100
```

#### 配置网卡规则(抑制ARP)


继续在两台RS(Real Server)上进行配置

**配置arp抑制规则**
>vi /etc/sysctl.conf

```shell
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
```

**刷新网卡**
>sysctl -p

**添加路由，并且保存到自启动文件**
>echo "route add -host 192.168.252.100 dev lo:v1" >> /etc/rc.local 


### 四.安装&配置keepalived

分别在`161`，`162`两台机器上安装和配置Keepalived

#### 安装Keepalived
[Keepalived基本安装](http://jinlipool.com/2020/02/17/keepalived)

#### 配置Keepalived.conf

安装完毕后，分别对主从RS中的Keepalived进行配置。进入`/etc/keepalived/`修改配置文件`keepalived.conf`

**主节点配置**
```shell
global_defs {
   router_id LVS_161
}
#配置主节点
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 61
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100
    }
}

#LVS配置
virtual_server 192.168.1.100 80 {
    #健康检查的时间
    delay_loop 6
    #负载均衡算法
    lb_algo rr
    #LVS模式
    lb_kind DR
    persistence_timeout 5
    protocol TCP

    real_server 192.168.1.151 80 {
        weight 1
        TCP_CHECK {
          #检查80接口
          connect_port 80
          #超时时间 2
          connect_timeout 2
          #重试次数 2
          nb_get_retry 2
          #间隔时间 3s
          delay_before_retry 3
        }
    }

    real_server 192.168.1.152 80 {
       ...同上151配置
    }
}
```

**备用节点配置**
```shell
global_defs {
   router_id LVS_161
}
#配置主节点
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 61
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100
    }
}

#LVS配置
vrrp_instance VI_1 {
  #同主节点配置
}
```

#### 重启keepalived即可

重启下keepalived就可以生效了
>systemctl restart keepalived.service


### 五.安装&使用ipvsadm

在两台部署Keepalived的主备节点上进行操作

#### 安装ipvadmin
>yum install ipvsadm

ipvsadm 是Linux虚拟服务器的管理命令。是用于设置、维护和检查Linux内核中虚拟服务器列表的命令

#### 添加新的虚拟服务
>-A 新增一个虚拟服务，
>-t 表明这是一个tcp服务 
>-s 调度模式 rr 轮循
>ipvsadm -A -t 192.168.1.100:80 -s rr

#### 添加真实服务器

在内核虚拟服务器表的一条记录里添加新的真实服务器记录。也就是在一个虚拟服务器中增加一台新的真实服务器。

在主服务器添加如下命令
>-a 真实服务
> -t 表明是个tcp服务 
> -r 真实服务地址 
> -g dr模式 可以不写，默认的
>ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.151:80 -g 
>ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.152:80 -g

#### 测试

为了方便测试，需要将持久化时间调低
>-E 编辑 \
>-p 持久化 \
ipvsadm -E -t 192.168.1.100:80 -s rr -p 5 \
ipvsadm --set 1 1 1


第一次访问
![2](/img/nginx/lvs_nginx_keepalived/2.png)

隔个1s再刷一次，说明负载均衡没问题，轮循的方式。
![3](/img/nginx/lvs_nginx_keepalived/3.png)

关闭`151`的nginx，无论怎么刷新，只会显示`152`的nginx
![4](/img/nginx/lvs_nginx_keepalived/4.png)
![5](/img/nginx/lvs_nginx_keepalived/5.png)


最后，重新开启151的Nnginx,并且关闭主节点的keepalived

进入到备用节点，使用`ip addr`查看ip信息，可以看到VIP已经漂移到备用节点
![6](/img/nginx/lvs_nginx_keepalived/6.png)
页面是能正常访问的
![7](/img/nginx/lvs_nginx_keepalived/7.png)
等重新启动主节点的keepalived，VIP还是会漂移回主节点上的。




### 基本扫盲

**1. 什么是环回接口**
[简单理解Linux的Loopback接口](https://www.cnblogs.com/EasonJim/p/9996162.html)

**2. 什么是ARP**
ARP(Address Resolution Protocol) 是一种解决地址问题的协议,以目标IP地址为线索，用来定位下一个应该接受数据分包的网络设备对应的MAC地址。只适用于IPV4网络。

**3. 为什么将VIP配置在lo网卡**
首先是为了让RS能够处理地址是VIP的数据，在lo上配置vip能够接收到相关报并将结果返回给客户端。如果将VIP设置到出口网卡，会导致客户端的ARP request出现混乱。可能会影响到整个负载均衡

**4. 为什么要抑制ARP**
用户发起请求，在请求前会发一个arp广播包。由于DIP和SIP都在同一个物理网络中，并且都设置了VIP。为了让整个请求只会发送到DIP上。必须让其他SIP不能响应这个请求发出的ARP请求。

**5. arp规则说明**
* arp-ignore:ARP响应级别（处理请求）
0：主要本机配置IP，就能响应请求。
1：请求的目标地址到达对应的网络接口，才会响应请求。

* arp-announce:ARP通告行为（返回响应）**
0：本机上任何网络接口都向外通告，所有的网卡都能接受到通告。
1：尽可能避免本网卡与不匹配的目标进行通告。
2：只在本网卡通告。
