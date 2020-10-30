# 集群

## 集群简介

### 什么是集群

- 通过高速网络将很多服务器集中在一起

> 提供同一种服务，在客户端看来就像只有一个服务器

- 可以在付出较低成本的情况下获得在性能、可靠性、灵活性方面的相对较高的收益
- **任务调度**是集群系统中的**核心技术**

### 集群的目的

- 提高性能

如计算机密集型应用，如天气预报、核试验模拟

- 降低成本

相对百万美元级的超级计算机，价格便宜

- 提高可扩展性

只要增加集群节点即可

- 增强可靠性

多个节点完成相同功能，避免单点失败

### 集群分类

- 高性能计算集群HPC

通过以集群开发的并行应用程序，解决复杂的科学问题

- 负载均衡（LB）集群

客户端负载在计算机集群中尽可能**平均分摊**

- 高可用（HA）集群

避免**单点故障**，当一个系统发生故障时，可以快速迁移

--------

# LVS

## LVS概述

### LVS项目介绍

- Linux虚拟服务器（LVS）是章文嵩在国防科技大学就读博士期间创建的

- LVS可以实现高可用、可伸缩的Web、mail、Cache和Media等网络服务
- 最终目标是利用Linux操作系统和LVS集群软件实现一个高可用、高性能、低成本的服务器应用集群

### LVS集群组成

- 前端：**负载均衡层**

由一台或多台负载调度器构成

- 中间：**服务器群组层**

由一组实际运行应用服务的服务器组成

- 底端：**数据共享存储层**

提供共享存储空间的存储区域

![](E:\fc-learn\Linux-learn\Cluster\chat_img_465324_16040221226066.jpg)

### LVS术语

- Director Server:调度服务器

将负载分发到Real Server的服务器

- Real Server:真实服务器

真正提供应用服务的服务器

- VIP:虚拟IP地址

公布给用户访问的虚拟IP地址

- DIP:调度器连接后端节点服务器的IP地址
- RIP真实IP地址

集群节点上使用的IP地址

![](E:\fc-learn\Linux-learn\Cluster\chat_img_465324_16040221087372.jpg)

### LVS工作模式

- NAT模式
- DR模式
- TUN模式

![image-20201030100808748](E:\fc-learn\Linux-learn\Cluster\image-20201030100808748.png)

### 负载均衡调度算法

LVS目前实现了10种调度算法

常用调度算法有四种

- 轮询（Round Robin）
- 加权轮询（Weighted Round Robin）
- 最少连接（Least Connections）
- 加权最少连接（Weigted Least Connections）

其他调度算法

- 源地址散列（Source Hashing）
- 目标地址散列（Destination Hashing）
- 基于局部性的最少连接
- 带复制的基于局部性最少连接
- 最短的期望的延迟
- 最少队列调度

## ipvsadm命令用法

安装ipvsadm软件包，使用ipvsadm命令，实现如下功能

- 使用命令添加基于TCP一些的集群服务
- 在集群中添加若干台后端真实服务器
- 实现同一客户端访问，调度器分配固定服务器
- 会使用ipvsadm实现规则的增、删、改
- 保存ipvsadm规则



常用ipvsadm命令语法格式如表-1及表-2所示。

![img](E:\fc-learn\Linux-learn\Cluster\table001.png)



![img](E:\fc-learn\Linux-learn\Cluster\table002.png)

#### 步骤一：使用命令增删改LVS集群规则

##### 1.创建LVS虚拟集群服务器（算法位加权轮询：wrr）

```shell
[root@proxy ~]# yum -y install ipvsadm
[root@proxy ~]# ipvsadm -A -t 192.168.4.5:80 -s wrr
[root@proxy ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.4.5:80 wrr
```

##### 2.为集群添加哎若干real server

```shell
[root@proxy ~]# ipvsadm -a -t 192.168.4.5:80 -r 192.168.2.100 
[root@proxy ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.4.5:80 wrr
  -> 192.168.2.100:80             router    1      0          0
[root@proxy ~]# ipvsadm -a -t 192.168.4.5:80 -r 192.168.2.200 -m -w 2
[root@proxy ~]# ipvsadm -a -t 192.168.4.5:80 -r 192.168.2.201 -m -w 3
[root@proxy ~]# ipvsadm -a -t 192.168.4.5:80 -r 192.168.2.202 -m -w 4
```

##### 3.修改集群服务器设置

（修改调度器算法，将加权轮询修改为轮询）

```shell
[root@proxy ~]# ipvsadm -E -t 192.168.4.5:80 -s rr
[root@proxy ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.4.5:80 rr
  -> 192.168.2.100:80             router    1      0          0         
  -> 192.168.2.200:80             masq      2      0          0         
  -> 192.168.2.201:80             masq      2      0          0         
  -> 192.168.2.202:80             masq      1      0          0
```

##### 4.修改read server

使用-g选项，将模式改为DR模式

```shell
[root@proxy ~]# ipvsadm -e -t 192.168.4.5:80 -r 192.168.2.202 -g
```

##### 5.查看LVS状态

```shell
[root@proxy ~]# ipvsadm -Ln
```

##### 6.创建另一个集群

算法为最少连接算法；

使用-m选项，设置工作模式为NAT模式

```shell
[root@proxy ~]# ipvsadm -A -t 192.168.4.5:3306 -s lc
[root@proxy ~]# ipvsadm -a -t 192.168.4.5:3306 -r 192.168.2.100 -m
[root@proxy ~]# ipvsadm -a -t 192.168.4.5:3306 -r 192.168.2.200 -m
```

------------

## 部署LVS-NAT集群

使用LVS实现NAT模式的集群调度服务器，为用户提供Web服务：

- 集群对外公网IP地址为192.168.4.5
- 调度器内网IP地址为192.168.2.5
- 真实Web服务器地址分别为192.168.2.100、192.168.2.200
- 使用加权轮询调度算法，真实服务器权重任意

### 实验拓扑结构及主机配置

> 10月30日 11：40课，cluster 讲解为什么配网关，从新听一听！

**proxy**: <u>eth0:192.168.4.10/24</u>

**web1**: <u>eth1:192.168.2.100/24</u>

​			<u>gateway:192.168.2.5</u>

​			<u>如果有4网段IP地址，需要临时关闭</u>

**web2**: <u>eth1:192.168.2.200/24</u>

​			<u>gateway:192.168.2.5</u>

​			<u>如果有4网段IP地址，需要临时关闭</u>

**client**: <u>eth0: 192.168.4.10/24</u>

使用4台虚拟机

1台作为Director调度器

2台作为Real Server

1台客户端

拓扑结构如图

**!!注意：web1和web2必须配置网关地址!!**

![image-20201030115330764](E:\fc-learn\Linux-learn\Cluster\image-20201030115330764.png)

### 步骤

#### 步骤一： 配置基础环境

##### 1.设置web服务器

```shell
[root@web1 ~]# yum -y install httpd        #安装软件
[root@web1 ~]# echo "192.168.2.100" > /var/www/html/index.html    #创建网页文件
[root@web1 ~]# firewall-cmd --set-default-zone=trusted            #设置防火墙
[root@web1 ~]# setenforce  0
[root@web1 ~]# sed -i  '/SELINUX/s/enforcing/permissive/'  /etc/selinux/config  

[root@web2 ~]# yum -y install httpd        #安装软件
[root@web2 ~]# echo "192.168.2.200" > /var/www/html/index.html    #创建网页文件
[root@web2 ~]# firewall-cmd --set-default-zone=trusted            #设置防火墙
[root@web2 ~]# setenforce  0
[root@web2 ~]# sed -i  '/SELINUX/s/enforcing/permissive/'  /etc/selinux/config
```

##### 2.启动web服务器软件并验证

```shell
[root@web1 ~]# systemctl restart httpd
[root@web2 ~]# systemctl restart httpd

[root@proxy ~]# curl http://192.168.2.100
[root@proxy ~]# curl http://192.168.2.200
#使用proxy主机测试是否可以访问web1和web2
```

##### 3.配置网关

web1和web2的网关设置为192.168.2.5

如果有4网段的IP，则临时将该网卡关闭

> `nmcli con down 网卡名称`

```shell
[root@web1 ~]# nmcli connection modify ens33 \
ipv4.method manual ipv4.gateway 192.168.2.5
#备注:网卡名称不能照抄,需要自己查看下2.100的网卡名称
[root@web1 ~]# nmcli connection up ens33
[root@web1 ~]# ip route show                #查看默认网关
default via 192.168.2.5 dev ens33          #提示：这里default后面的IP就是默认网关
#英语词汇：default（默认，预设值）
… …
[root@web2 ~]# nmcli connection modify ens33 \
ipv4.method manual ipv4.gateway 192.168.2.5
#备注:网卡名称不能照抄,需要自己查看下2.200的网卡名称
[root@web2 ~]# nmcli connection up ens33
[root@web2 ~]# ip route show        #查看默认网关，default后面的IP就是默认网关
```

##### 为社么需要配置网关？

![image-20201030121929221](E:\fc-learn\Linux-learn\Cluster\image-20201030121929221.png)

为了方便下面所有的IP都采用简写,如4.10代表192.168.4.10，2.100代表192.168.2.100。

​      

英语词汇：source（src）代表源地址，destination（dest或dst）代表目标地址。

​      

LVS采用的是路由器的NAT通讯原理！通讯流程如下：

​      

1.客户端发送请求数据包(src:4.10,dst:4.5)

​      

2.数据包被发送给LVS调度器，调度器做NAT地址转换（外网转内网，内网转外网）

​      

a)数据包被修改为src:4.10,dst:2.100(dst也有可能被修改为2.200，随机的）

​      

b)LVS调度器把数据包转发给后端真正的web服务器（2.100）

​      

3.web1收到数据包开始回应数据（rsc:2.100，dst:4.10）

​      

备注：谁访问就给谁回复数据，因为src是4.10，所以应该给4.10回应数据！

​      

但是，自己是2.100，对方是4.10，跨网段默认无法通讯，如何解决？？？

​      

Web1和web2都需要设置默认网关（也就是192.168.2.5）

​      

4）web1想发送数据给4.10但是又无法与其通讯，所以数据包被交给默认网关

​      

5）LVS调度器（软路由）收到后端web发送过来的数据后，再次做NAT地址转换

​      

a)数据包被修改为src:4.5,dst:4.10

​      

b)LVS调度器把数据包转发给客户端主机

​      

6）客户端接收网页数据内容

​      

注意：客户端访问的是4.5，最后是4.5给客户端回复的网页数据！！！！

#### 步骤二：部署调度器

##### 1.确认调度器的路由转发功能

(如果已经开启，可以忽略)

lvs在Linux内核已内嵌，但是需要安装ipvsadm工具用来给lvs命令

```shell
[root@proxy ~]# echo 1 > /proc/sys/net/ipv4/ip_forward     #开启路由转发，临时有效
[root@proxy ~]# cat /proc/sys/net/ipv4/ip_forward          #查看效果
1
[root@proxy ~]# echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
#修改配置文件，设置永久规则，英语词汇：forward（转寄，转发，发送，向前）
```

##### 2.创建集群服务器（ipvsadm）

```shell
[root@proxy ~]# ipvsadm -a -t 192.168.4.5:80 -r 192.168.2.100 -w 1 -m
[root@proxy ~]# ipvsadm -a -t 192.168.4.5:80 -r 192.168.2.200 -w 1 -m
#-a(add)往虚拟服务器集群中添加后端真实服务器IP,指定往-t 192.168.4.5:80这个集群中添加
#-r(real)后面跟后端真实服务器的IP和端口，这里不写端口默认是80端口
#-w(weight)指定服务器的权重，权重越大被访问的次数越多，英语词汇：weight（重量，分量）
#-m指定集群工作模式为NAT模式，如果是-g则代表使用DR模式，-i代表TUN模式
```

##### 4.查看规则列表

L是list查看，n是number数字格式显示

```shell
[root@proxy ~]# ipvsadm -Ln
```

此时检查防火墙及色Linux设置，确保不会影响实验

#### 步骤三：客户端测试

客户端client主机使用curl命令反复连接http://192.168.4.5，查看访问的页面是否会轮询到不同的后端真实服务器。

可能出现的问题解决方法：

1.web服务是否启动，防火墙是否设置；

2.webIP 配置是否正确，网关是否正确，能否ping通；

3.web有没有4网段的IP（不能有）

4.proxy添加的LVS规则是否正确

5.proxy防火墙是否设置

```shell
[root@client ~]# curl 192.168.4.5
192.168.2.100
[root@client ~]# curl 192.168.4.5
192.168.2.200
[root@client ~]# curl 192.168.4.5
192.168.2.100
[root@client ~]# curl 192.168.4.5
192.168.2.200
```

###### 此操作未实现高可用

至此，如web1或web2故障，lvs不会动态感知到，依然会照常转发，导致在关闭一台web服务器服务时候，此状态下测试时一条成功一条失败，暂时未达到高可用。

Nginx的web集群有动态感知，可以识别无法访问的web服务器

## 部署LVS-DR集群

使用LVS实现DR模式的集群调度服务器，为用户提供Web服务：

- 客户端IP地址为192.168.4.10
- LVS调度器VIP地址为192.168.4.15
- LVS调度器DIP地址设置为192.168.4.5
- 真实Web服务器地址分别为192.168.4.100、192.168.4.200
- 使用加权轮询调度算法，权重可以任意

说明：      

CIP是客户端的IP地址；     

VIP是对客户端提供服务的IP地址；   

RIP是后端服务器的真实IP地址；  

DIP是调度器与后端服务器通信的IP地址(VIP必须配置在虚拟接口)

### 方案

使用4台虚拟机，1台作为客户端、1台作为Director调度器、2台作为Real Server，拓扑结构如图-3所示。实验拓扑结构主机配置细节如表-4所示

![image-20201030155100190](E:\fc-learn\Linux-learn\Cluster\image-20201030155100190.png)

![img](E:\fc-learn\Linux-learn\Cluster\table004.png)

为什么本实验中web1和web2要采用4网段IP？如图-4所示。

![img](E:\fc-learn\Linux-learn\Cluster\image004.png)

LVS NAT实验请求数据包从LVS调度器进，web的相应数据包也从LVS调度器出，那么LVS调度器就需要承载所有数据的压力，会成为整个集群的瓶颈！！      

本实验LVS DR模式的核心需求是希望web1和web2可以不走调度器返回数据！      

但是如如-3所示，如果web1和web2采用2.100和2.200这样2网段的IP，又不希望给4.10回复数据走LVS调度器（也就是不给web1和web2配置默认网关为2.5），最后是无法跨网段通讯的！！！    

怎么办？核心需求是希望web1和web2可以直接返回数据给客户端！！！     

想让web1和web2可以直接返回数据给客户端，可以给web1和web2配置4网段IP，如图-5所示

![img](E:\fc-learn\Linux-learn\Cluster\image005.png)

这样就可以了吗？答案是否定的！！！网络中的基本原则是A访问B，必须是B返回数据给A，现在4.10访问4.5，最终4.100给4.10返回网页数据，所有数据包都会被丢弃！！！    

那怎么办呢？地址欺骗！

![img](E:\fc-learn\Linux-learn\Cluster\image006.png)

如图-6所示，我们给web1和web2再额外添加一个伪装的IP地址，这个IP地址因为是用来做地址欺骗用的，假的就是假的，不能暴露（必须配置在lo本地回环网卡上面）。     

lo网卡上面默认配置的IP是127.0.0.1。      

如果你家里有非法的1000W，你会天天出去跟别人说你有1000W吗?

### 步骤

实现此案例需要按照如下步骤进行。   

说明：  

CIP是客户端的IP地址；   

VIP是对客户端提供服务的IP地址（本案例为192.168.4.15）；   

VIP必须配置在虚拟接口（目的是防止地址冲突）；    

RIP是后端服务器的真实IP地址（本案例为192.168.4.100和192.168.4.200）；    

DIP是调度器与后端服务器通信的IP地址（本案例为192.168.4.5）。

#### 步骤一：配置实验网络环境

##### 1.设置proxy服务器的VIP和DIP

**!!注意~为了防止冲突，VIP必须要配置在网卡的虚拟接口，网卡名称不能照抄!! !!**

```shell
[root@proxy ~]# cd /etc/sysconfig/network-scripts/
[root@proxy ~]# cp ifcfg-eth0  ifcfg-eth0:0
[root@proxy ~]# vim ifcfg-eth0:0
TYPE=Ethernet
#网卡类型为：以太网卡
BOOTPROTO=none
#none手动配置IP，或者dhcp自动配置IP
NAME=eth0:0
#网卡名称
DEVICE=eth0:0
#设备名称
ONBOOT=yes
#开机时是否自动激活该网卡
IPADDR=192.168.4.15
#IP地址
PREFIX=24
#子网掩码
[root@proxy ~]# systemctl restart network        #重启网络服务
[root@proxy ~]# ip  a  s             #会看到一个网卡下面有两个IP地址
```

##### 2.设置web1服务器网络参数

不能照抄网卡名称哦~~

```shell
[root@web1 ~]# nmcli connection modify eth0 ipv4.method manual \
ipv4.addresses 192.168.4.100/24 connection.autoconnect yes
[root@web1 ~]# nmcli connection up eth0
```

接下来给web1配置VIP地址

**注意！！！**

**这里的子网掩码必须是32 （也就是全255）**

**网络地址与IP地址一样**

**广播地址与IP地址也一样**

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
#开机是否激活本网卡
NAME=lo:0
#网卡名称
```

防止地址冲突的问题：      

这里因为web1也配置与调度器一样的VIP地址，默认肯定会出现地址冲突；      

sysctl.conf文件写入这下面四行的主要目的就是访192.168.4.15的数据包，只有调度器会响应，其他主机都不做任何响应，这样防止地址冲突的问题

```shell
[root@web1 ~]# vim /etc/sysctl.conf
#文件末尾手动写入如下4行内容,英语词汇：ignore（忽略、忽视），announce（宣告、广播通知）
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
#当有arp广播问谁是192.168.4.15时，本机忽略该ARP广播，不做任何回应（防止进站冲突）
#本机不要向外宣告自己的lo回环地址是192.168.4.15（防止出站冲突）
[root@web1 ~]# sysctl -p
```

**<u>上述代码中，</u>**

**<u>参数"1"表示，忽略；</u>**

**<u>`net.ipv4.conf.all.arp_ignore = 1`表示忽略arp广播寻找4.15的IP，不做回应</u>**

**<u>参数"2"表示，打死也不说！！！</u>**

**<u>`net.ipv4.conf.lo.arp_announce = 2`表示禁止向外宣告自己有4.15的IP</u>**

重启网络服务

```shell
[root@web1 ~]# systemctl restart network        #重启网络服务
[root@web1 ~]# ip  a   s           #会看到一个网卡下面有两个IP地址
```

常见错误：如果重启网络后未正确配置lo:0，有可能是NetworkManager和network服务有冲突，关闭NetworkManager后重启network即可。（非必须的操作）

```shell
[root@web1 ~]# systemctl stop NetworkManager
[root@web1 ~]# systemctl restart network
```

##### 3.设置web2服务器网络参数

不能照抄网卡名称哦~~·

```shell
[root@web2 ~]# nmcli connection modify eth0 ipv4.method manual \
ipv4.addresses 192.168.4.200/24 connection.autoconnect yes
[root@web2 ~]# nmcli connection up eth0
```

接下来给web2配置VIP地址      

注意：这里的子网掩码必须是32（也就是全255），网络地址与IP地址一样，广播地址与IP地址也一样。

```shell
[root@web2 ~]# cd /etc/sysconfig/network-scripts/
[root@web2 ~]# cp ifcfg-lo  ifcfg-lo:0
[root@web2 ~]# vim ifcfg-lo:0
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

防止地址冲突的问题： 

这里因为web1也配置与调度器一样的VIP地址，默认肯定会出现地址冲突；

sysctl.conf文件写入这下面四行的主要目的就是访问192.168.4.15的数据包，只有调度器会响应，其他主机都不做任何响应，这样防止地址冲突的问题。

```shell
[root@web2 ~]# vim /etc/sysctl.conf
#手动写入如下4行内容，英语词汇：ignore（忽略、忽视），announce（宣告、广播通知）
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
#当有arp广播问谁是192.168.4.15时，本机忽略该ARP广播，不做任何回应（防止进站冲突）
#本机不要向外宣告自己的lo回环地址是192.168.4.15（防止出站冲突）
[root@web2 ~]# sysctl -p
```

重启网络服务

```shell
[root@web2 ~]# systemctl restart network        #重启网络服务
[root@web2 ~]# ip a  s                 #会看到一个网卡下面有两个IP地址
```

#### 步骤二：proxy调度器安装软件并部署LVS-DR模式调度器

##### 1.安装软件（如果已经安装，此步骤可以忽略）

##### 2.清理之前实验的规则，创建新的集群服务器规则

```shell
[root@proxy ~]# ipvsadm -C                                #清空所有规则
[root@proxy ~]# ipvsadm -A -t 192.168.4.15:80 -s wrr
## -A(add)是创建添加虚拟服务器集群
# -t(tcp)后面指定集群VIP的地址和端口，协议是tcp协议
# -s后面指定调度算法，如rr（轮询）、wrr（加权轮询）、lc（最少连接）、wlc（加权最少连接）等等
```

##### 3.添加真实服务器(-g参数设置LVS工作模式为DR模式，-w设置权重)

```shell
[root@proxy ~]# ipvsadm -a -t 192.168.4.15:80 -r 192.168.4.100 -g -w 1
[root@proxy ~]# ipvsadm -a -t 192.168.4.15:80 -r 192.168.4.200 -g -w 1
#-a(add)往虚拟服务器集群中添加后端真实服务器IP,指定往-t 192.168.4.15:80这个集群中添加
#-r(real)后面跟后端真实服务器的IP和端口，这里不写端口默认是80端口
#-w(weight)指定服务器的权重，权重越大被访问的次数越多，英语词汇：weight（重量，分量）
#-m指定集群工作模式为NAT模式，如果是-g则代表使用DR模式，-i代表TUN模式
```

##### 4.查看规则列表

（L代表list查看规则，n代表number数字格式显示

```shell
[root@proxy ~]# ipvsadm -Ln
TCP  192.168.4.15:80 wrr
  -> 192.168.4.100:80             Route   1      0          0         
  -> 192.168.4.200:80             Route   1      0          0
```

#### 步骤三：客户端测试

客户端使用curl命令反复连接http://192.168.4.15，查看访问的页面是否会轮询到不同的后端真实服务器。  

注意：本实验不可以在proxy主机（LVS调度器）使用curl访问网页验证！！！    

为什么？请思考图-7示意图。

![image-20201030180800843](E:\fc-learn\Linux-learn\Cluster\image-20201030180800843.png)



### 扩展知识：默认LVS不带健康检查功能，需要自己手动编写动态检测脚本，实现该功能：(参考脚本如下，仅供参考)

此条呼应部署LVS-NAT步骤三的小标题~~（老师还没讲）

```shell
[root@proxy ~]# vim check.sh
#!/bin/bash
VIP=192.168.4.15:80
RIP1=192.168.4.100
RIP2=192.168.4.200
while :
do
   for IP in $RIP1 $RIP2
   do
           curl -s http://$IP &>/dev/null
if [ $? -eq 0 ];then
            ipvsadm -Ln |grep -q $IP || ipvsadm -a -t $VIP -r $IP
        else
             ipvsadm -Ln |grep -q $IP && ipvsadm -d -t $VIP -r $IP
        fi
   done
sleep 1
done
```

