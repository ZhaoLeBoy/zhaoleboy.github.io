---
layout:     post
title:      "Nginx+Keepalived 双机主备，双机热备"
subtitle:   "Nginx+Keepalived 双机主备，双机热备"
date:        2020-02-18
author:     "ZhaoLe"
header-img: "img/nginx/post-bg.png"
catalog: true
tags:
    - Keepalived Nginx
---


### 一.环境配置

|  名称    | 版本号  |
| --- | --- |
| CentOS | 7.x  |
| keepalived | 2.0.2 |
| Nginx | 1.16.0 |

### 二.安装Nginx
 [Nginx的安装](!http://jinlipool.com/2020/02/17/nginx)

### 三.安装Keepalived
 [Keepalived的安装](!http://jinlipool.com/2020/02/17/keepalived)

### 四.双机主备

![1](/img/nginx/nginx_keepalived/1.png)

一台机器作为master，然后用一台同样配置的机器作为备用机器，使用虚拟ip(vip)作为对外的访问，当master出现异常时候，keepalived会将虚拟ip绑定在备用节点上，当master修复后，虚拟ip还是会继续回到master节点上

#### 主节点配置
1. 首先修改master节点上的配置文件，我选择用`192.168.252.131`作为主节点
>ip addr #查看当前机器的网卡名称，配置文件中需要
![bfdad922a729ef883c515096af0cf279](/img/nginx/nginx_keepalived/2.png)

2. 修改keepalived的配置文件`/etc/keepalived/keepalived.conf`

```shell
global_defs {
    #路由ID ，保证全局唯一
   router_id keep_131
}

vrrp_instance VI_1 {
    #表示这台机器是主机(MASTER)还是备用机BACKUP
    state MASTER
    #网卡名称 ens33
    interface ens33
    #保证主备节点一致
    virtual_router_id 51
    #权重，master一般高于backup
    priority 100
    #主备间同步检查时间间隔 单位：秒
    advert_int 1
    #认证权限密码 防止非法节点进入
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 虚拟IP 可以是多个
    virtual_ipaddress {
        #主备机的vip要保持一致
        192.168.252.101
    }
}
```

#### 备用节点配置
1. 修改备用节点上的配置文件，我选择用`192.168.252.132`作为备用节点

>ip addr #查看当前机器的网卡名称，配置文件中需要

![3](/img/nginx/nginx_keepalived/3.png)

2. 修改keepalived的配置文件`/etc/keepalived/keepalived.conf`

```shell
global_defs {
    #路由ID ，保证全局唯一
   router_id keep_132
}

vrrp_instance VI_1 {
    #表示这台机器是主机(MASTER)还是备用机BACKUP
    state BACKUP
    #网卡名称 ens33
    interface ens33
    #保证主备节点一致
    virtual_router_id 51
    #权重，master一般高于backup
    priority 80
    #主备间同步检查时间间隔 单位：秒
    advert_int 1
    #认证权限密码 防止非法节点进入
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 虚拟IP 可以是多个
    virtual_ipaddress {
        #主备机的vip要保持一致
        192.168.252.101
    }
}
```

#### 编写脚本用来启动nginx

主备模式下，能保证当主节点出现宕机，备用节点是能及时启动。但是还有一种情况,就是当这个节点里面的nginx出现了问题，比如出现宕机，这个时候备用节点并不会被激活，VIP也不会绑定到备用节点上的.
为了保证nginx能持续提供服务,需要keepalived定时去检测nginx是否健康,如果出现问题应该及时启动,只有nginx启动不了的情况下才去启动备用节点.
实现这个功能 需要通过shell脚本,我把这个脚本放在了`/etc/keepalived`文件夹中.它是由keepalived的配置文件进行调用,方便在一起方便调用(具体位置看个人只要路径别出错)

>check_nginx.sh

```shell
 #!/bin/bash
 A=`ps -C nginx --no-header|wc -l`
 
 if [ $A -eq 0 ];then
    /usr/local/nginx/sbin/nginx
    sleep 3
    if [ `ps -C nginx --no-header|wc -l` -eq 0 ];then
        killall keepalived
    fi
 fi
```

然后先修改主节点的配置文件
>vi /etc/keepalived/keepalived.conf

```shell
global_defs {
   router_id keep_131
}

#新增部分
vrrp_script check_nginx {
   script "/etc/keepalived/check_nginx.sh"  
   #每2s运行一次脚本
   interval 2
   #成功执行一次脚本权重加10 否则-10
   weight 10
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    #新增部分 触发检查脚本
    track_script {
        check_nginx  #追踪nginx脚本
    }
    virtual_ipaddress {
        192.168.252.101
    }
}
```
配置完之后重新启动下keepalived就可以了，测试的话可以手动关闭nginx进程，你会发现nginx是关闭不了的。。等你关闭了keepalived调用脚本又把nginx给重新启动了。


### 五.双机热备

![4](/img/nginx/nginx_keepalived/4.png)

主备模式有个最大问题就是浪费资源，如果master节点一直没问题，那备用机就一直空置在那。所以又衍生出双机热备模式。
两台机器互为主备，这种模式需要两个虚拟IP。图上的DNS轮循是在云服务器上的，由一个域名绑定多个ip，然后通过dns进行负载均衡

如果上面双机主备设置成功，双机热备配置是很简单的。只需要修改主备节点的配置文件`keepalived.conf`

#### 主节点

```shell
#双主热备 再原有的配置后面继续追加即可
vrrp_instance VI_2 {
    state BACKUP
    interface ens33
    #保证主备节点(每组)一致
    virtual_router_id 52
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.252.102
    }
}
```

#### 备节点

```shell
vrrp_instance VI_2 {
    state MASTER
    interface ens33
    #保证主备节点一致
    virtual_router_id 52
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.252.102
    }
}
```

#### 验证结果

修改完主备节点上的keepalived的配置文件后重启就可以了

* **1 .当两个节点都正常工作**
 
![5](/img/nginx/nginx_keepalived/5.png)
![6](/img/nginx/nginx_keepalived/6.png)

![5](/img/nginx/nginx_keepalived/7.png)
![6](/img/nginx/nginx_keepalived/8.png)


* **2. 关闭131节点**

![5](/img/nginx/nginx_keepalived/9.png)
![6](/img/nginx/nginx_keepalived/10.png)

![5](/img/nginx/nginx_keepalived/11.png)
![6](/img/nginx/nginx_keepalived/12.png)

关闭了131的keepalived后，原本跟131绑定的VIP已经自动跟132进行绑定了。当用户请求进来，无论DNS怎么轮循，两个VIP都对应到132这个节点，直到131节点恢复
