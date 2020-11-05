

# HAproxy服务器

##### 负载均衡小总结

性能：lvs > haproxy > nginx    #从运行和效率来讲

功能：nginx > haproxy > lvs	#

软件功能：小	--->    性能：高

> nginx：web，地址重写，https，用户代理，健康检查

## HAproxy概述

和Nginx一摸一样：代理

LVS(NAT) ,web1和web2必须配网关

但是，王凯老师讲的nginx代理实验，没配网关却通了，为什么？

因为client把数据交给proxy后，就没下文了，真实原理是proxy去找web端请求数据，然后得到后，自己再交给client

在lvs中，原理是proxy 是作路由转发，所以必须加网关，而不是自身拿到数据后再去后端请求web服务

### HAproxy简介

- 它是免费、快速并且可靠的一种解决方案

- 适用于哪些负载特大的web站点，这些站点通常又需要会话保持或七层处理

- 提供高可用性、负载均衡及基于TCP和HTTP应用的代理

### 负载均衡分类

- **二层负载均衡（mac）**

  一般是用虚拟mac地址方式，外部对虚拟MAC地址请求，负载均衡接收后分配后端实际的MAC地址响应。

- **三层负载均衡（ip）**

  一般采用虚拟IP地址方式，外部对虚拟的ip地址请求，负载均衡接收后分配后端实际的IP地址响应。

- **四层负载均衡（tcp）**

  在三层负载均衡的基础上，用ip+port接收请求，再转发到对应的机器。

- **七层负载均衡（http）**

  根据虚拟的 url 或 IP 或 主机名接收请求，再转向相应的处理服务器。

### 四层负载与七层负载的区别

**总结：从上面的对比看来四层负载与七层负载最大的区别就是效率（性能）与功能的区别。**

**四层负载架构设计比较简单，无需解析具体的消息内容，在网络吞吐量及处理能力上会相对比较高；**

**七层负载均衡的优势则体现在功能多，控制灵活强大。在具体业务架构设计时，使用七层负载或者四层负载还得根据具体的情况综合考虑。**

四层用得最多得是LVS ，七层用的最多的是ngin

四层的传输速度会比较快，LVS的工作模式比较多，性能好，吞吐量大，安全性低

七层的功能比较多，可以用location匹配，rewrite地址重写，安全性高

同时，nginx还有上传、压缩的功能，

可以定义地址实，地址实里面写上我们的IP+Port，再用反向代理代理过去就可以了。

### 衡量负责均衡器性能的因素

- Session rate  会话率

每秒钟产生的会话数

- Session concurrency  并发会话数

服务器处理会话的时间越长，并发会话数越多

- Data rate  数据速率

以 MB/s或Mbps衡量

### HAproxy 工作模式

- mode http

客户端请求被深度分析后再发往服务器

- mode tcp

4层调度，不检查第七层信息

- mode health

仅做健康状态检查，已经不建议使用

## 配置HAproxy负载均衡集群

### 实现功能

准备4台Linux服务器，两台做Web服务器，1台安装HAProxy，1台做客户端，实现如下功能：

- 客户端访问HAProxy，HAProxy分发请求到后端Real Server
- 开启HAProxy监控页面，及时查看调度器状态
- 设置HAProxy为开机启动

### 实现方案

使用4台虚拟机，1台作为HAProxy调度器、2台作为Real Server、1台作为客户端，拓扑结构如图-3所示，具体配置如表-3所示。

![image-20201102171602651](E:\fc-learn\Linux-learn\Cluster\image-20201102171602651.png)

![img](E:\fc-learn\Linux-learn\Cluster\table003.png)

为什么Haproxy的实验不需要开启路由，不需要给web服务器配置网关？ 

Hapoxy是代理服务器（帮你干活的人或物就是你的代理），通讯流程如图-4所示。

![img](E:\fc-learn\Linux-learn\Cluster\image004.png)

### 实验步骤

#### 准备工作

实现此案例需要按照如下步骤进行。

web1配置本地真实IP地址（不能照抄网卡名）。

```shell
[root@web1 ~]# nmcli connection modify eth1 ipv4.method manual \
ipv4.addresses 192.168.2.100/24 connection.autoconnect yes
[root@web1 ~]# nmcli connection up eth1
```

Web2配置本地真实IP地址（不能照抄网卡名）。

```shell
[root@web2 ~]# nmcli connection modify eth1 ipv4.method manual \
ipv4.addresses 192.168.2.200/24 connection.autoconnect yes
[root@web2 ~]# nmcli connection up eth1
```

proxy关闭keepalived服务，清理LVS规则，不能照抄网卡名。

```shell
[root@proxy ~]# systemctl stop keepalived
[root@proxy ~]# systemctl disable keepalived
[root@proxy ~]# ipvsadm -C

[root@proxy ~]# nmcli connection modify eth0 ipv4.method manual \
ipv4.addresses 192.168.4.5/24 connection.autoconnect yes
[root@proxy ~]# nmcli connection up eth0

[root@proxy ~]# nmcli connection modify eth1 ipv4.method manual \
ipv4.addresses 192.168.2.5/24 connection.autoconnect yes
[root@proxy ~]# nmcli connection up eth1
```

#### 步骤一：配置后端web服务器

##### 设置两台后端Web服务

（如果已经配置完成，可忽略此步骤）

```shell
[root@web1 ~]# yum -y install httpd
[root@web1 ~]# systemctl start httpd
[root@web1 ~]# echo "192.168.2.100" > /var/www/html/index.html

[root@web2 ~]# yum -y install httpd
[root@web2 ~]# systemctl start httpd
[root@web2 ~]# echo "192.168.2.200" > /var/www/html/index.html
```

#### 步骤二：部署HAproxy服务器

##### 1.配置网络，安装软件

```shell
[root@proxy ~]# yum -y install haproxy
```

##### 2.修改配置文件

```shell
[root@proxy ~]# vim /etc/haproxy/haproxy.cfg
global
 log 127.0.0.1 local2   ##[err warning info debug]
 pidfile /var/run/haproxy.pid ##haproxy的pid存放路径
 user haproxy
 group haproxy
 daemon                    ##以后台进程的方式启动服务
defaults
 mode http                ##默认的模式mode { tcp|http|health } 
option dontlognull      ##不记录健康检查的日志信息
 option httpclose        ##每次请求完毕后主动关闭http通道
 option httplog          ##日志类别http日志格式
 option redispatch      ##当某个服务器挂掉后强制定向到其他健康服务器
 timeout client 300000 ##客户端连接超时，默认毫秒，也可以加时间单位
 timeout server 300000 ##服务器连接超时
 maxconn  3000          ##最大连接数
 retries  3             ##3次连接失败就认为服务不可用，也可以通过后面设置
  
listen  websrv-rewrite 0.0.0.0:80          
   balance roundrobin
   server  web1 192.168.2.100:80 check inter 2000 rise 2 fall 5
   server  web2 192.168.2.200:80 check inter 2000 rise 2 fall 5
#定义集群,listen后面的名称任意，端口为80
#balance指定调度算法为轮询（不能用简写的rr）
#server指定后端真实服务器，web1和web2的名称可以任意
#check代表健康检查，inter设定健康检查的时间间隔，rise定义成功次数，fall定义失败次数
```

##### 3.启动服务器并设置开机启动

```shell
[root@proxy ~]# systemctl restart haproxy
[root@proxy ~]# systemctl enable haproxy
```

#### 步骤三：客户端验证

客户端配置与HAProxy相同网络的IP地址，并使用火狐浏览器访问http://192.168.4.5，测试调度器是否正常工作，客户端访问http://192.168.4.5:1080/stats测试状态监控页面是否正常。访问状态监控页的内容，参考图-5所示。

![img](E:\fc-learn\Linux-learn\Cluster\image005.png)

### 查看服务器访问状态

在nginx中，查看访问的状态信息等数据，还需要安装模块

`--with-http_stub_status_module`

在haproxy中，在配置文件中加入以下配置参数即可

```shell
listen stats *:1080        #监听端口
    stats refresh 30s             #统计页面自动刷新时间
    stats uri /stats              #统计页面url
    stats realm Haproxy Manager #进入管理解面查看状态信息
    stats auth admin:admin       #统计页面用户名和密码设置
```



## 备注：   

Queue队列数据的信息（当前队列数量，最大值，队列限制数量）；   

Session rate每秒会话率（当前值，最大值，限制数量）；

Sessions总会话量（当前值，最大值，总量，Lbtot: total number of times a server was selected选中一台服务器所用的总时间）；

Bytes（入站、出站流量）；

Denied（拒绝请求、拒绝回应）；

Errors（错误请求、错误连接、错误回应）；

Warnings（重新尝试警告retry、重新连接redispatches）；

Server(状态、最后检查的时间（多久前执行的最后一次检查）、权重、备份服务器数量、down机服务器数量、down机时长)。

附加思维导图，如图-6所示：

![img](E:\fc-learn\Linux-learn\Cluster\image006.png)