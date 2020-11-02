# keepalived热备

## keepalived概述

- 调度器出现单点故障，如何解决？
  - keeplived实现了高可用集群
  - keepalived最初是为LVS设计的
    - 专门监控各服务器节点的状态
- keepalived后来加入了VRRP功能，防止单点故障



1.VRRP（自动配IP）   验证 ip a s

2.自动配置LVS规则		ipvsadm -Ln

3.健康检查		把web1关服务，再查ipvsadm -Ln

## keepalived运行原理

- keepalived检测每个服务器节点状态

- 服务器节点异常或工作出现故障，keepalived将故障节点从集群系统中剔除

- 故障节点恢复后，keepalived再将其加入到集群系统中

- 所有工作自动完成，无需人工干预



## 部署keepalived高可用服务器

### 实现功能

准备三台Linux服务器，两台做Web服务器，并部署Keepalived高可用软件，一台作为客户端主机，实现如下功能：

- 使用Keepalived实现web服务器的高可用
- Web服务器IP地址分别为192.168.4.100和192.168.4.200
- Web服务器的浮动VIP地址为192.168.4.80
- 客户端通过访问VIP地址访问Web页面

### 方案

使用3台虚拟机，2台作为Web服务器，并部署Keepalived、1台作为客户端，拓扑结构如图-1所示，主机配置如表-1所示。

![image-20201102151128161](E:\fc-learn\Linux-learn\Cluster\image-20201102151128161.png)

![image-20201102151144449](E:\fc-learn\Linux-learn\Cluster\image-20201102151144449.png)

### 步骤

实现此案例需要按照如下步骤进行。

#### 步骤一：配置网络环境

（如果在前面课程已经完成该配置，可以忽略此步骤）      

##### 1.设置Web1服务器网络参数、配置Web服务

（不能照抄网卡名称）

```shell
[root@web1 ~]# nmcli connection modify eth0 ipv4.method manual ipv4.addresses 192.168.4.100/24 connection.autoconnect yes
[root@web1 ~]# nmcli connection up eth0

[root@web1 ~]# yum -y install httpd        #安装软件
[root@web1 ~]# echo "192.168.4.100" > /var/www/html/index.html    #创建网页文件
[root@web1 ~]# systemctl restart httpd        #启动服务器
```

##### 2.设置web2服务器网络参数、配置web服务

（不能照抄网卡名称）

```shell
[root@web2 ~]# nmcli connection modify eth0 ipv4.method manual ipv4.addresses 192.168.4.200/24 connection.autoconnect yes
[root@web2 ~]# nmcli connection up eth0
[root@web2 ~]# yum -y install httpd        #安装软件
[root@web2 ~]# echo "192.168.4.200" > /var/www/html/index.html    #创建网页文件
[root@web2 ~]# systemctl restart httpd        #启动服务器
```

##### 3.配置proxy主机的网络参数

备注：这个实验，我们使用proxy当作客户端主机，网卡名称不能照抄

```shell
[root@proxy ~]# nmcli connection modify eth0 ipv4.method manual ipv4.addresses 192.168.4.5/24 connection.autoconnect yes
[root@proxy ~]# nmcli connection up eth0
```

#### 步骤二：安装Keepalived软件

注意：两台Web服务器做相同的操作

```shell
[root@web1 ~]# yum install -y keepalived
[root@web2 ~]# yum install -y keepalived 
```

#### 步骤三：部署keepalived服务

##### 1.修改web1服务器keepalived配置文件

```shell
[root@web1 ~]# vim /etc/keepalived/keepalived.conf
global_defs {
  router_id  web1           #12行，设置路由ID号（实验需要修改）
    vrrp_iptables           #13行，清除防火墙的拦截规则（实验需要修改，手动添加该行）
}
vrrp_instance VI_1 {
  state MASTER              #21行，主服务器为MASTER（备服务器需要修改为BACKUP）
  interface eth0            #22行，VIP配在哪个网卡（实验需要修改，不能照抄网卡名）
  virtual_router_id 51      #23行，主备服务器VRID号必须一致
  priority 100              #24行，服务器优先级,优先级高优先获取VIP
  advert_int 1
  authentication {
    auth_type pass
    auth_pass 1111                       
  }
  virtual_ipaddress {       #30~32行，谁是主服务器谁获得该VIP（实验需要修改）
192.168.4.80/24 
}    
}
```

##### 2.修改web2服务器Keepalived配置文件

```shell
[root@web2 ~]# vim /etc/keepalived/keepalived.conf
global_defs {
  router_id  web2           #12行，设置路由ID号（实验需要修改）
  vrrp_iptables             #13行，清除防火墙的拦截规则（实验需要修改，手动添加该行） 
}
vrrp_instance VI_1 {
  state BACKUP              #21行，备服务器为BACKUP（实验需要修改）
  interface eth0            #22行，VIP配在哪个网卡（实验需要修改，不能照抄网卡名）
  virtual_router_id 51      #23行，主辅VRID号必须一致
  priority 50               #24行，服务器优先级（实验需要修改）
  advert_int 1
  authentication {
     auth_type pass
     auth_pass 1111                   
  }
  virtual_ipaddress {       #30~32行，谁是主服务器谁配置VIP（实验需要修改）
192.168.4.80/24 
 }   
}
```

##### 3.启动服务

```shell
[root@web1 ~]# systemctl start keepalived
[root@web2 ~]# systemctl start keepalived
```

##### 4.配置防火墙和SELinux

```shell
[root@web1 ~]# firewall-cmd --set-default-zone=trusted
[root@web1 ~]# sed -i  '/SELINUX/s/enforcing/permissive/' /etc/selinux/config
[root@web1 ~]# setenforce 0

[root@web2 ~]# firewall-cmd --set-default-zone=trusted
[root@web2 ~]# sed -i  '/SELINUX/s/enforcing/permissive/' /etc/selinux/config
[root@web2 ~]# setenforce 0
```

#### 步骤四：测试

##### 1.登录两台Web服务器查看VIP信息

```shell
[root@web1 ~]# ip addr show
[root@web2 ~]# ip addr show
```

##### 2.客户端访问

客户端使用curl命令连接http://192.168.4.80，查看Web页面；

给Web1关机，客户端再次访问http://192.168.4.80，验证是否可以正常访问服务。

-------

## 部署keepalived+LVS服务器

### 实现功能

使用Keepalived为LVS调度器提供高可用功能，防止调度器单点故障，为用户提供Web服务：   

- LVS1调度器真实IP地址为192.168.4.5
- LVS2调度器真实IP地址为192.168.4.6
- 服务器VIP地址设置为192.168.4.15
- 真实Web服务器地址分别为192.168.4.100、192.168.4.200
- 使用加权轮询调度算法，真实web服务器权重不同

### 实现方案

使用5台虚拟机，1台作为客户端主机、2台作为LVS调度器、2台作为Real Server，实验拓扑环境结构如图-2所示，基础环境配置如表-2所示。

![img](E:\fc-learn\Linux-learn\Cluster\image002.png)

![img](E:\fc-learn\Linux-learn\Cluster\table002.png)

### 实现步骤

实现此案例需要按照如下步骤进行。

#### 步骤一：配置网络环境

##### 1.关闭服务

（把案例1中给web1和web2安装的keepalived关闭）

**警告：请先将案例1中web1和web2的keepalived关闭！！！**

```shell
[root@web1 ~]# systemctl  stop   keepalived
[root@web2 ~]# systemctl  stop   keepalived
```

##### 2.设置Web1服务器的网络参数

（不能照抄网卡名称）

```shell
[root@web1 ~]# nmcli connection modify eth0 ipv4.method manual \
ipv4.addresses 192.168.4.100/24 connection.autoconnect yes
[root@web1 ~]# nmcli connection up eth0
```

接下来给web1配置VIP地址

注意：这里的子网掩码必须是32（也就是全255），网络地址与IP地址一样，广播地址与IP地址也一样。

```shell
[root@web1 ~]# cd /etc/sysconfig/network-scripts/
[root@web1 ~]# cp ifcfg-lo  ifcfg-lo:0
[root@web1 ~]# vim ifcfg-lo:0
DEVICE=lo:0
#设备名称
IPADDR=192.168.4.15
#IP地址
NETMASK=255.255.255.255
#子网掩码
NETWORK=192.168.4.15
#网络地址
BROADCAST=192.168.4.15
#广播地址
ONBOOT=yes
#开机是否激活该网卡
NAME=lo:0
#网卡名称
```

注意：这里因为web1也配置与调度器一样的VIP地址，默认肯定会出现地址冲突。

写入下面这四行的主要目的就是访问192.168.4.15的数据包，只有调度器会响应，其他主机都不做任何响应。

```shell
[root@web1 ~]# vim /etc/sysctl.conf
#手动写入如下4行内容，英语词汇：ignore（忽略、忽视），announce（宣告、广播通知）
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
#arp_ignore(防止进站冲突)
#arp_announce(防出站冲突)
[root@web1 ~]# sysctl  -p                #刷新，让配置文件立刻生效
```

重启网络服务

```shell
[root@web1 ~]# systemctl restart network      #重启网络服务
[root@web1 ~]# ip  a   s                     #查看IP地址
```

##### 3.设置web2服务器的网络参数

参考web1

##### 4.配置proxy主机的网络参数

**不配置VIP，VIP由keepalvied自动配置**

把前面课程创建VIP的网卡配置文件直接删除  

备注：不能照抄网卡名称。

```shell
[root@proxy ~]# nmcli connection modify eth0 ipv4.method manual \
ipv4.addresses 192.168.4.5/24 connection.autoconnect yes
[root@proxy ~]# nmcli connection up eth0
```

##### 5.配置proxy2主机的网络参数

##### **不配置VIP，VIP由keepalvied自动配置**

注意：按照前面的课程环境，默认没有该虚拟机，需要重新建一台虚拟机proxy2。

备注：不能照抄网卡名称。

配置方式同prxoy

#### 步骤二：配置后台web服务

##### 1.安装软件，自定义web页面

web1和web2主机

```shell
[root@web1 ~]# yum -y install httpd
[root@web1 ~]# echo "192.168.4.100" > /var/www/html/index.html
[root@web2 ~]# yum -y install httpd
[root@web2 ~]# echo "192.168.4.200" > /var/www/html/index.html
```

##### 2.启动web服务器软件

web1和web2主机

```shell
[root@web1 ~]# systemctl start httpd ; systemctl enable httpd
[root@web2 ~]# systemctl start httpd ; systemctl enable httpd
```

#### 步骤三：调度器安装keepalived与ipvsadm软件

**注意：两台LVS调度器执行相同的操作**

**（如何已经安装软件，可忽略此步骤）。**

安装软件

```shell
[root@proxy ~]# yum install -y keepalived
[root@proxy ~]# systemctl enable keepalived
[root@proxy ~]# yum install -y ipvsadm
[root@proxy ~]# ipvsadm -C

[root@proxy2 ~]# yum install -y keepalived
[root@proxy2 ~]# systemctl enable keepalived
[root@proxy2 ~]# yum install -y ipvsadm
[root@proxy2 ~]# ipvsadm -C
```

#### 步骤四:部署keepalived实现LVS-DR模式调度器的高可用

##### 1.LVS1调度器设置Keepalived，并启动服务

（在192.168.4.5主机操作）

```shell
[root@proxy ~]# vim /etc/keepalived/keepalived.conf
global_defs {
  router_id  lvs1          #12行，设置路由ID号(实验需要修改)
  vrrp_iptables            #13行，清除防火墙的拦截规则（实验需要修改，手动添加）   
}
vrrp_instance VI_1 {
  state MASTER             #21行，主服务器为MASTER
  interface eth0           #22行，定义网络接口（不能照抄网卡名）
  virtual_router_id 51     #23行，主辅VRID号必须一致
  priority 100             #24行，服务器优先级
  advert_int 1
  authentication {
    auth_type pass
    auth_pass 1111                       
  }
  virtual_ipaddress {      #30~32行，配置VIP（实验需要修改）
192.168.4.15/24 
 }   
}
virtual_server 192.168.4.15 80 {        #设置ipvsadm的VIP规则（实验需要修改）
  delay_loop 6                          #默认健康检查延迟6秒
  lb_algo rr                            #设置LVS调度算法为RR
  lb_kind DR                            #设置LVS的模式为DR（实验需要修改）
  #persistence_timeout 50               #（实验需要删除）
#注意persistence_timeout的作用是保持连接
#开启后，客户端在一定时间内(50秒)始终访问相同服务器
  protocol TCP                          #TCP协议
  real_server 192.168.4.100 80 {        #设置后端web服务器真实IP（实验需要修改）
    weight 1                            #设置权重为1
    TCP_CHECK {                         #对后台real_server做健康检查（实验需要修改）
    connect_timeout 3                   #健康检查的超时时间3秒
    nb_get_retry 3                      #健康检查的重试次数3次
        delay_before_retry 3            #健康检查的间隔时间3秒
    }
  }
 real_server 192.168.4.200 80 {         #设置后端web服务器真实IP（实验需要修改）
    weight 2                            #设置权重为2
    TCP_CHECK {                         #对后台real_server做健康检查（实验需要修改）
         connect_timeout 3              #健康检查的超时时间3秒
    nb_get_retry 3                      #健康检查的重试次数3次
    delay_before_retry 3                #健康检查的间隔时间3秒
    }
  }
}
[root@proxy1 ~]# systemctl start keepalived
[root@proxy1 ~]# ipvsadm -Ln           #查看LVS规则
[root@proxy1 ~]# ip a  s               #查看VIP配置
```

##### 2.lvs2调度器设置Keepalived

（在192.168.4.6主机操作）

```shell
[root@proxy2 ~]# vim /etc/keepalived/keepalived.conf
global_defs {
   router_id  lvs2                     #12行，设置路由ID号（实验需要修改）
 vrrp_iptables                       #13行，清除防火墙的拦截规则（实验需要修改，手动添加）   

}
vrrp_instance VI_1 {
  state BACKUP                          #21行，从服务器为BACKUP（实验需要修改）
  interface eth0                        #22行，定义网络接口（不能照抄网卡名）
  virtual_router_id 51                  #23行，主辅VRID号必须一致
  priority 50                           #24行，服务器优先级（实验需要修改）
  advert_int 1
  authentication {
    auth_type pass
    auth_pass 1111 
  }
  virtual_ipaddress {                   #30~32行，设置VIP（实验需要修改）
192.168.4.15/24  
}  
}
virtual_server 192.168.4.15 80 {        #自动设置LVS规则（实验需要修改）
  delay_loop 6
  lb_algo  rr                           #设置LVS调度算法为RR
  lb_kind DR                            #设置LVS的模式为DR（实验需要修改）
 # persistence_timeout 50               #（实验需要删除该行）
#注意persistence_timeout的作用是保持连接
#开启后，客户端在一定时间内(50秒)始终访问相同服务器
  protocol TCP                          #TCP协议
  real_server 192.168.4.100 80 {        #设置后端web服务器的真实IP（实验需要修改）
    weight 1                            #设置权重为1
    TCP_CHECK {                         #对后台real_server做健康检查（实验需要修改）
      connect_timeout 3                   #健康检查的超时时间3秒
    nb_get_retry 3                      #健康检查的重试次数3次
    delay_before_retry 3                #健康检查的间隔时间3秒
    }
  }
 real_server 192.168.4.200 80 {          #设置后端web服务器的真实IP（实验需要修改）
    weight 1                             #设置权重为1，权重可以根据需要修改
    TCP_CHECK {                          #对后台real_server做健康检查（实验需要修改）
      connect_timeout 3                    #健康检查的超时时间3秒
    nb_get_retry 3                       #健康检查的重试次数3次
    delay_before_retry 3                 #健康检查的间隔时间3秒
    }
  }
[root@proxy2 ~]# systemctl start keepalived
[root@proxy2 ~]# ipvsadm -Ln             #查看LVS规则
[root@proxy2 ~]# ip  a   s               #查看VIP设置
```

### 步骤五：客户端测试

客户端使用curl命令反复连接http://192.168.4.15，查看访问的页面是否会轮询到不同的后端真实服务器。