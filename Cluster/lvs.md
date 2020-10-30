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

##### 2.创建集群服务器

```shell
[root@proxy ~]# yum -y install ipvsadm
[root@proxy ~]# ipvsadm -A -t 192.168.4.5:80 -s wrr
# -A(add)是创建添加虚拟服务器集群
# -t(tcp)后面指定集群VIP的地址和端口，协议是tcp协议
# -s后面指定调度算法，如rr（轮询）、wrr（加权轮询）、lc（最少连接）、wlc（加权最少连接）等等
```

##### 3.添加真实服务器

ipvsadm如环境配错，可使用`ipvsadm -C`清空所有配置

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

