

# Ceph

## Ceph概述

### 分布式存储

分布式是指一种独特的系统架构，它由一组网络进行通信、为了完成共同的任务而协调工作的计算机节点组成

分布式系统是为了用廉价的、普通的机器完成单个计算机无法完成的计算、存储任务

其目的就是利用更多的机器，处理更多的数据

### 常用分布式文件系统

Lustre	太老了

Hadoop	特定行业（大数据等）

FastDFS	市场太少

Ceph	主流

GlusterFS	市场太少									

### 什么是Ceph

- ceph是一个分布式文件系统
- 具有高扩展、高可用、高性能的特点
- ceph可以提供对象存储、块存储、文件系统存储
- ceph可以提供PB级别的存储空间（PB->TB->GB)
  - 1024G*1024G=1048576G
- 软件定义存储（software defined storage）作为存储行业的一大发展趋势，已经越来越受到市场的认可

![image-20201103101753641](E:\fc-learn\Linux-learn\Cluster\image-20201103101753641.png)

- ceph-mon（过半原则）

mon服务默认过半原则

（例：两台互为备份，一台坏了之后相当于1/2正常工作，没有过半，所以说要做最低的备份，要至少3个mon服务，相当于坏了一个的情况下，还有2/3的服务可以使用，过半了）

- ceph数据默认3副本

一份数据，ceph会做3副本（相当于一份源+2份复制），如果一个硬盘损坏，那么此时ceph会在其他硬盘继续做备份以保证一份数据永远是3份，除非极端情况硬盘只剩下不到3块了

### ceph组件

- OSDs

存储设备

- Monitors

集群监控组件

- RadosGateway（RGW）

对象存储网关

- MDSs

存放文件系统的元数据（对象存储和块存储不需要该组件）

- client

ceph客户端

![image-20201103104841254](E:\fc-learn\Linux-learn\Cluster\image-20201103104841254.png)

## 实验环境

### 实现功能

准备四台虚拟机，其三台作为存储集群节点，一台安装为客户端，实现如下功能：

- 创建1台客户端虚拟机
- 创建3台存储集群虚拟机
- 配置主机名、IP地址、YUM源
- 修改所有主机的主机名
- 配置无密码SSH连接
- 配置NTP时间同步
- 创建虚拟机磁盘

### 实验方案

使用4台虚拟机，1台客户端、3台存储集群服务器，拓扑结构如图-1所示。

![image-20201103105514061](E:\fc-learn\Linux-learn\Cluster\image-20201103105514061.png)

Ceph组件架构如图-2所示。

![image-20201103105554148](E:\fc-learn\Linux-learn\Cluster\image-20201103105554148.png)      

  Ceph会对数据进行切割处理，如图-3所示。      

![img](E:\fc-learn\Linux-learn\Cluster\image003.png)

图-3

​      

Ceph随机读写数据的思路，如图-4所示

![img](E:\fc-learn\Linux-learn\Cluster\image004.png)

​                                                   图4

Ceph集群结构如图-5所示。

![img](E:\fc-learn\Linux-learn\Cluster\image005.png)

### 实验步骤

实现此案例需要按照如下步骤进行。

#### 步骤一：安装前准备

##### 1.为所有节点配置yum源服务器

把四台虚拟机全部关机；每台虚拟机都添加一个光驱；

做如下相同操作:

右击虚拟机,选【设置】---【添加】---【CD|DVD驱动器】--【完成】；

点击刚刚新建的光盘[CD|DVD],勾选使用ISO映像文件--[浏览]；

找到自己真机的ceph10.iso加载即可。

添加磁盘：

除了客户端，所有3台ceph服务器都添加2块20G磁盘。

启动所有虚拟机后，查看磁盘情况:

易错点：看清楚哪个是系统光盘，哪个是ceph光盘，光盘是否有顺序错乱！

```shell
[root@client ~]# lsblk
[root@node1 ~]# lsblk
[root@node2 ~]# lsblk
[root@node3 ~]# lsblk
```

##### 2.所有主机设置防火墙和SElinux

```shell
[root@client ~]# firewall-cmd --set-default-zone=trusted
[root@client ~]# sed -i '/SELINUX/s/enforcing/permissive/' /etc/selinux/config
[root@client ~]# setenforce 0

[root@node1 ~]# firewall-cmd --set-default-zone=trusted
[root@node1 ~]# sed -i '/SELINUX/s/enforcing/permissive/' /etc/selinux/config
[root@node1 ~]# setenforce 0

[root@node2 ~]# firewall-cmd --set-default-zone=trusted
[root@node2 ~]# sed -i '/SELINUX/s/enforcing/permissive/' /etc/selinux/config
[root@node2 ~]# setenforce 0

[root@node3 ~]# firewall-cmd --set-default-zone=trusted
[root@node3 ~]# sed -i '/SELINUX/s/enforcing/permissive/' /etc/selinux/config
[root@node3 ~]# setenforce 0
```

##### 3.所有主机挂载ceph光盘和系统光盘

易错点：【不能照抄、不能照抄、不能照抄】，需要根据实际情况操作，

案例假设是将系统光盘挂载到/media目录，将ceph光盘挂载到/ceph目录。

```shell
[root@client ~]# umount /dev/sr0
[root@client ~]# umount /dev/sr1             #未挂载的话会报错
[root@client ~]# mkdir  /ceph
[root@client ~]# vim  /etc/fstab
    /dev/sr0    /ceph     iso9660   defaults   0  0       #需要根据实际情况挂载（看清楚哪个是ceph光盘，该挂载到哪个目录）
    /dev/sr1    /media    iso9660   defaults   0  0       #需要根据实际情况挂载（看清楚那额是系统光盘，该挂载到哪个目录）
#切记不能照抄，切记！！切记！！切记！！切记！！
[root@client ~]# mount -a
[root@client ~]# lsblk

[root@node1 ~]# umount /dev/sr0
[root@node1 ~]# umount /dev/sr1
[root@node1 ~]# mkdir /ceph
[root@node1 ~]# vim /etc/fstab
    /dev/sr0    /ceph     iso9660   defaults   0  0      #需要根据实际情况挂载（看清楚哪个是ceph光盘，该挂载到哪个目录）
    /dev/sr1    /media    iso9660   defaults   0  0      #需要根据实际情况挂载（看清楚那额是系统光盘，该挂载到哪个目录）
#切记不能照抄，切记！！切记！！切记！！切记！！
[root@node1 ~]# mount -a

[root@node2 ~]# umount /dev/sr0
[root@node2 ~]# umount /dev/sr1
[root@node2 ~]# mkdir /ceph
[root@node2 ~]# vim /etc/fstab 
    /dev/sr0    /ceph     iso9660   defaults   0  0       #需要根据实际情况挂载（看清楚哪个是ceph光盘，该挂载到哪个目录）
    /dev/sr1    /media    iso9660   defaults   0  0       #需要根据实际情况挂载（看清楚那额是系统光盘，该挂载到哪个目录）
#切记不能照抄，切记！！切记！！切记！！切记！！
[root@node2 ~]# mount -a

[root@node3 ~]# umount /dev/sr0
[root@node3 ~]# umount /dev/sr1
[root@node3 ~]# mkdir /ceph
[root@node3 ~]# vim /etc/fstab
    /dev/sr0    /ceph     iso9660   defaults   0  0        #需要根据实际情况挂载（看清楚哪个是ceph光盘，该挂载到哪个目录）
    /dev/sr1    /media    iso9660   defaults   0  0        #需要根据实际情况挂载（看清楚那额是系统光盘，该挂载到哪个目录）
#切记不能照抄，切记！！切记！！切记！！切记！！
[root@node3 ~]# mount -a
```

##### 4.配置无密码连接

(包括自己远程自己也不需要密码)，在node1操作

```shell
[root@node1 ~]# ssh-keygen   -f /root/.ssh/id_rsa    -N ''
#-f后面跟密钥的文件名称（希望创建密钥到哪个文件）
#-N ''代表不给密钥配置密钥（不能给密钥配置密码）
[root@node1 ~]# for i in 10  11  12  13
 do
     ssh-copy-id  192.168.4.$i
 done
#通过ssh-copy-id将密钥传递给192.168.4.10、192.168.4.11、192.168.4.12、192.168.4.13
```

##### 5.修改/etc/hosts并同步到所有主机

注意：/etc/hosts解析的域名要与本机主机名一致！！！！

```shell
 [root@node1 ~]# vim /etc/hosts     #修改文件，手动添加如下内容（不要删除文件原有内容）
... ...
192.168.4.10  client
192.168.4.11     node1
192.168.4.12     node2
192.168.4.13     node3
```

提示：/etc/hosts解析的域名必须与本机主机名一致！！！  

将/etc/hosts文件拷贝给所有其他主机（client、node1、node2、node3）

```shell
[root@node1 ~]# for i in client node1  node2  node3
do
scp  /etc/hosts   $i:/etc/
done
```

##### 6.修改所有节点都需要配置yum源，并同步到所有主机

```shell
[root@node1 ~]# vim /etc/yum.repos.d/ceph.repo    #新建YUM源配置文件，内容如下
[mon]
name=mon
baseurl=file:///ceph/MON
gpgcheck=0
[osd]
name=osd
baseurl=file:///ceph/OSD
gpgcheck=0
[tools]
name=tools
baseurl=file:///ceph/Tools
gpgcheck=0
[root@node1 ~]# yum clean all               #清空缓存
[root@node1 ~]# yum repolist                #验证YUM源软件数量
源标识            源名称                    状态
Dvd                redhat                    9,911
Mon                mon                        41
Osd                osd                        28
Tools            tools                    33
repolist: 10,013
[root@node1 ~]# for i in  client  node1  node2  node3
do
scp  /etc/yum.repos.d/ceph.repo   $i:/etc/yum.repos.d/
done
```

##### 7.给所有节点安装ceph相关软件包

```shell
[root@node1 ceph-cluster]# for i in node1 node2 node3
do
    ssh  $i "yum -y install ceph-mon ceph-osd ceph-mds ceph-radosgw"
done 
```

##### 8.Client主机配置NTP服务器

```shell
[root@client ~]# yum -y install chrony
[root@client ~]# vim /etc/chrony.conf
    allow 192.168.4.0/24        #修改26行
    local stratum 10            #修改29行(去注释即可)
[root@client ~]# systemctl restart chronyd
```

##### 9.node1 node2 node3 修改NTP客户端配置

```shell
[root@node1 ~]# vim /etc/chrony.conf
server 192.168.4.10   iburst              #配置文件第二行，手动添加一行新内容
[root@node1 ~]# systemctl restart chronyd
[root@node1 ~]# chronyc sources -v        #查看同步结果，应该是^*
[root@node2 ~]# vim /etc/chrony.conf
server 192.168.4.10   iburst              #配置文件第二行，手动添加一行新内容
[root@node2 ~]# systemctl restart chronyd
[root@node2 ~]# chronyc sources -v            #查看同步结果，应该是^*

[root@node3 ~]# vim /etc/chrony.conf
server 192.168.4.10   iburst              #配置文件第二行，手动添加一行新内容
[root@node3 ~]# systemctl restart chronyd
[root@node3 ~]# chronyc sources -v       #查看同步结果，应该是^*
```

## 部署ceph集群

### 实现功能

沿用实验环境，部署Ceph集群服务器，实现以下目标：

- 安装部署工具ceph-deploy
- 创建ceph集群
- 准备日志磁盘分区
- 创建OSD存储空间
- 查看ceph状态，验证

### 实现步骤

#### 步骤一：安装部署软件ceph-deploy

##### 1.在node1安装部署工具，学习工具的语法格式

```shell
[root@node1 ~]#  yum -y install ceph-deploy
[root@node1 ~]#  ceph-deploy  --help
[root@node1 ~]#  ceph-deploy mon --help
```

##### 2.创建目录

目录名称可以任意，推荐与案例一致

```shell
[root@node1 ~]#  mkdir ceph-cluster
[root@node1 ~]#  cd ceph-cluster/
```

#### 步骤二：部署ceph集群

##### 1.创建Ceph集群配置,在ceph-cluster目录下生成Ceph配置文件（ceph.conf）。

<u>**在ceph.conf配置文件中定义monitor主机是谁。**</u>

```shell
[root@node1 ceph-cluster]# ceph-deploy new node1 node2 node3
```

##### 2.初始化所有节点的mon服务，也就是启动mon服务

拷贝当前目录的配置文件到所有节点的/etc/ceph/目录并启动mon服务。

```shell
[root@node1 ceph-cluster]# ceph-deploy mon create-initial
#配置文件ceph.conf中有三个mon的IP，ceph-deploy脚本知道自己应该远程谁
```

##### 3.在每个node主机查看自己的服务

**注意每台主机服务名称不同**

```shell
[root@node1 ceph-cluster]# systemctl status ceph-mon@node1
[root@node2 ~]# systemctl status ceph-mon@node2
[root@node3 ~]# systemctl status ceph-mon@node3
#备注:管理员可以自己启动（start）、重启（restart）、关闭（stop），查看状态（status）.
#提醒:这些服务在30分钟只能启动3次,超过就报错. 
#StartLimitInterval=30min
#StartLimitBurst=3
#在这个文件中有定义/usr/lib/systemd/system/ceph-mon@.service
#如果修改该文件，需要执行命令# systemctl  daemon-reload重新加载配置
```

##### 4.查看ceph集群状态(现在状态应该是health HEALTH ERR)

```shell
[root@node1 ceph-cluster]# ceph -s
```

##### 常见错误及解决方法

（非必要操作，有错误可以参考）：

如果提示如下错误信息：（如何无法修复说明环境准备有问题，需要重置所有虚拟机）

```shell
[node1][ERROR ] admin_socket: exception getting command descriptions: [Error 2] No such file or directory
```

解决方案如下

**!!仅在node1操作!!**

###### 1.检查命令是否是在ceph-cluster目录下执行

如果确认是在该目录下执行的create-initial命令，依然报错，可以使用如下方式修复。

```shell
[root@node1 ceph-cluster]# vim ceph.conf      #文件最后追加以下内容
public_network = 192.168.4.0/24
```

###### 2.修改后重新推送配置文件

```shell
[root@node1 ceph-cluster]# ceph-deploy --overwrite-conf config push node1 node2 node3
[root@node1 ceph-cluster]# ceph-deploy --overwrite-conf mon  create-initial
```

###### 3.如果还出错

可能是准备实验环境时配置的域名解析和主机名不一致

#### 步骤三：创建OSD

##### 1.初始化清空磁盘数据（仅node1操作即可）

初始化磁盘，将所有磁盘分区格式设置为GPT格式（根据实际情况填写磁盘名称）

```shell
[root@node1 ceph-cluster]# ceph-deploy disk  zap  node1:sdb   node1:sdc   
[root@node1 ceph-cluster]# ceph-deploy disk  zap  node2:sdb   node2:sdc
[root@node1 ceph-cluster]# ceph-deploy disk  zap  node3:sdb   node3:sdc  
#相当于ssh 远程node1，在node1执行parted /dev/sdb  mktable  gpt
#其他主机都是一样的操作
#ceph-deploy是个脚本，这个脚本会自动ssh远程自动创建gpt分区
```

思考题？

```shell
# vim test.sh
#!/bin/bash
case $1 in
user)
     useradd -u 1000 $2;;
disk)
     parted  /dev/$2  mktable  gpt;;
esac
# chmod +x test.sh
# ./test.sh  user  jerry
# ./test.sh  disk  sdc
```

执行上面的脚本没有指定账户UID，为什么会自动创建一个UID为1000的用户？

执行上面的脚本没有指定磁盘分区表类型，为什么创建的分区表类型为gpt类型？

上面的脚本如果执行时不给位置变量的参数为怎么样？

##### 2.创建OSD存储空间

仅node1操作即可

重要：很多同学在这里会出错！将主机名、设备名称输入错误！！！

远程所有node主机，创建分区，格式化磁盘，挂载磁盘，启动osd服务共享磁盘。

```shell
[root@node1 ceph-cluster]# ceph-deploy osd create node1:sdb  node1:sdc
#每个磁盘都会被自动分成两个分区；一个固定5G大小；一个为剩余所有容量
#5G分区为Journal日志缓存；剩余所有空间为数据盘。
[root@node1 ceph-cluster]# ceph-deploy osd create node2:sdb  node2:sdc
[root@node1 ceph-cluster]# ceph-deploy osd create node3:sdb  node3:sdc
```

提醒：ceph-deploy是个脚本，脚本会自动创建分区、格式化、挂载！

怎么验证分区了？怎么验证格式化？怎么验证挂载了？

```shell
[root@node1 ~]# df -Th
[root@node2 ~]# df -Th
[root@node3 ~]# df -Th
```

思考题：请问lsblk和df命令的区别？

##### 3.在三台不同的主机查看OSD服务状态

可以开启、关闭、重启服务

```shell
[root@node1 ~]# systemctl status ceph-osd@0
[root@node2 ~]# systemctl status ceph-osd@2
[root@node3 ~]# systemctl status ceph-osd@4
#备注:管理员可以自己启动（start）、重启（restart）、关闭（stop），查看状态（status）.
#提醒:这些服务在30分钟只能启动3次,超过就报错.
#StartLimitInterval=30min
#StartLimitBurst=3
#在这个文件中有定义/usr/lib/systemd/system/ceph-osd@.service
#如果修改该文件，需要执行命令# systemctl  daemon-reload重新加载配置
```

##### 常见错误及解决方法

（非必须操作）

使用osd create创建OSD存储空间时，如提示下面的错误提示：

[ceph_deploy][ERROR ] RuntimeError: bootstrap-osd keyring not found; run 'gatherkeys'

可以使用如下命令修复文件，重新配置ceph的密钥文件：

```shell
[root@node1 ceph-cluster]#  ceph-deploy gatherkeys node1 node2 node3 
```

#### 步骤四：验证测试

##### 1.查看集群状态

```shell
[root@node1 ~]#  ceph  -s
[root@node1 ~]#  ceph   osd   tree
```

##### 2.常见错误

如果查看状态包含如下信息：

```shell
health: HEALTH_WARN
        clock skew detected on  node2, node3… 
```

clock skew表示时间不同步，解决办法：请先将所有主机的时间都使用NTP时间同步！！！

Ceph要求所有主机时差不能超过0.05s，否则就会提示WARN。

如果状态还是失败，可以尝试执行如下命令，重启所有ceph服务：

```shell
[root@node1 ~]#  systemctl restart ceph.target
```



## 创建ceph块存储

### 实现功能

使用Ceph集群的块存储功能，实现以下目标：

- 创建块存储镜像
- 客户端映射镜像
- 删除镜像

### 实现步骤

#### 步骤一：创建镜像

##### 1.）查看存储池，默认存储池名称为rbd

```shell
[root@node1 ~]# ceph osd lspools
0 rbd,
#查看结果显示，共享池的名称为rbd，这个共享池的编号为0，英语词汇：pool（池塘、水塘）
```

##### 2.创建镜像、查看镜像

```shell
[root@node1 ~]# rbd create demo-image --image-feature  layering --size 10G
#创建demo-image镜像，这里的demo-image创建的镜像名称，名称可以为任意字符。
#size可以指定镜像大小
[root@node1 ~]# rbd create rbd/jacob  --image-feature  layering --size 10G
#在rbd池中创建名称为jacob的镜像（rbd/jacob），镜像名称可以任意
```

\#--image-feature参数指定我们创建的镜像有哪些功能，layering是开启COW功能。

\#提示：ceph镜像支持很多功能，但很多是操作系统不支持的，我们只开启layering。

```shell
[root@node1 ~]# rbd list      #列出所有镜像
[root@node1 ~]# rbd info demo-image     #查看demo-image这个镜像的详细信息
rbd image 'demo-image':
    size 10240 MB in 2560 objects
    order 22 (4096 kB objects)
    block_name_prefix: rbd_data.d3aa2ae8944a
    format: 2
    features: layering
```

#### 步骤二：动态调整

##### 1.扩容容量

```shell
[root@node1 ~]# rbd resize --size 15G jacob             
#调整jacob镜像的大小，jacob是镜像的名称，size指定扩容到15G
[root@node1 ~]# rbd info jacob
```

##### 2.缩小容量

```shell
[root@node1 ~]# rbd resize --size 7G jacob --allow-shrink
#英文词汇：allow（允许），shrink（缩小）
[root@node1 ~]# rbd info jacob
#查看jacob这个镜像的详细信息（jacob是前面创建的镜像）
```

#### 步骤三：通过KRBD访问

Linux内核可用直接访问Ceph块存储，KVM可用借助于librbd访问Ceph块存储。

客户端访问结构如图-6所示。

![img](E:\fc-learn\Linux-learn\Cluster\image006.png)

##### 1.客户端通过KRBD访问

```shell
#客户端需要安装ceph-common软件包
#拷贝配置文件（否则不知道集群在哪）
#拷贝连接密钥（否则无连接权限）
[root@client ~]# yum -y  install ceph-common
[root@client ~]# scp 192.168.4.11:/etc/ceph/ceph.conf  /etc/ceph/
[root@client ~]# scp 192.168.4.11:/etc/ceph/ceph.client.admin.keyring \
/etc/ceph/
[root@client ~]# rbd map  jacob          #客户端访问映射服务器的jacob共享镜像
[root@client ~]#  lsblk                   #查看结果（会多一块磁盘）
[root@client ~]# rbd showmapped          #查看磁盘名和共享镜像名称的对应关系
id pool image snap device    
0  rbd  jacob -    /dev/rbd0
```

##### 2.客户端格式化，挂载分区

```shell
[root@client ~]# mkfs.xfs /dev/rbd0                     #格式化，格式为xfs
[root@client ~]# mount /dev/rbd0 /mnt/                  #挂载（可以挂载到任意目录）
[root@client ~]# echo "test" > /mnt/test.txt           #写入数据
```

#### 步骤四：删除镜像

##### 1.客户端撤销磁盘映射

```shell
[root@client ~]# umount /mnt                      #卸载
[root@client ~]# rbd showmapped                  #查看磁盘名和共享镜像名称的对应关系
id pool image        snap device    
0  rbd  jacob        -    /dev/rbd0
[root@client ~]# rbd unmap /dev/rbd0            #撤销磁盘映射
```



## ceph操作思路（知识总结）

### 一、准备工作：

IP，主机名，hosts解析，ssh密钥，时间同步，yum源，防火墙，selinux

### 二、部署ceph：

1.安装软件

```shell
 ceph-deploy(脚本)
  ceph-mon  ceph-osd  ceph-mds  ceph-radosgw(集群)
```

2.修改配置启动服务mon

```shell
  mkdir  目录；cd 目录
  ceph-deploy  new  node1   node2   node3  (生成配置文件)
  ceph-deploy  mon  create-initial  (拷贝配置文件并启动mon服务)
```

3.启动osd服务共享硬盘

```shell
  ceph-deploy  disk  zap   主机名:磁盘名  ...  ...
  ceph-deploy  osd  create  主机名:磁盘   ...  ...
```

### 三、使用Ceph的思路

1.块共享

```shell
  服务器: rbd  create  创建一个共享镜像
  客户端: 安装cpeh-common;  cp 配置文件和密钥
          rbd  map  |  rbd  unmap
```

附加知识（如何删除某个OSD，下面的假设是删除osd.4）

 ```shell
ceph osd tree
ceph osd out osd.4
ceph osd tree
ceph -s
ceph osd crush remove osd.4
ceph auth del osd.4
ceph -s
ceph osd rm osd.4
最后要找到对应的主机，umount把osd.4对应的磁盘卸载
 ```

## 块存储应用案例

ceph三副本和快照不是一个概念

### 实现功能

延续上面的实验内容，演示块存储的应用案例，实现以下功能：

- 创建镜像快照
- 使用快照还原数据
- 使用快照克隆镜像
- 删除快照

### 实现步骤

#### 步骤一：创建镜像快照

##### 1.查看镜像快照

默认所有镜像都没有快照

```shell
[root@node1 ~]# rbd snap ls jacob
#查看某个镜像有没有快照，jacob是镜像的名称，ls是list查看
```

##### 2.给镜像创建快照

```shell
[root@node1 ~]# rbd snap create jacob --snap jacob-snap1
#为jacob镜像创建快照，--snap指定快照名称，快照名称为jacob-snap1，快照名称可以任意
[root@node1 ~]# rbd snap ls jacob
SNAPID NAME            SIZE 
     4 jacob-snap1 15360 MB
```

##### 3.删除客户端写入的测试文件

```shell
[root@client ~]# rm  -rf   /mnt/test.txt
[root@client ~]# umount  /mnt  
```

##### 4.还原快照

```shell
[root@node1 ~]# rbd snap rollback jacob --snap jacob-snap1
# rollback是回滚的意思，使用jacob-snap1快照回滚数据，对jacob镜像进行回滚数据
[root@client ~]# mount /dev/rbd0 /mnt/		#客户端重新挂载分区
[root@client ~]# ls  /mnt       	#查看数据是否被恢复
```

#### 步骤二：创建快照克隆

##### 1.克隆快照

```shell
[root@node1 ~]#  rbd snap protect jacob --snap jacob-snap1	#保护快照
#jacob是镜像名称，jacob-snap1是前面创建的快照（被保护的快照，不可以被删除）
[root@node1 ~]#  rbd snap rm jacob --snap jacob-snap1    #删除被保护的快照，会失败
[root@node1 ~]#  rbd clone \
jacob --snap jacob-snap1 jacob-clone --image-feature layering
#使用jacob镜像的快照jacob-snap1克隆一个新的名称为jacob-clone的镜像
#新镜像名称可以任意
```

##### 2.查看克隆镜像与父镜像快照的关系

```shell
[root@node1 ~]#  rbd info jacob-clone
rbd image 'jacob-clone':
	size 15360 MB in 3840 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.d3f53d1b58ba
	format: 2
	features: layering
	flags: 
	parent: rbd/jacob@jacob-snap1
#克隆镜像的很多数据都来自于快照链(相当于文件的软链接的概念)
#如果希望克隆镜像可以独立工作，就需要将父快照中的数据，全部拷贝一份，但比较耗时！！！
[root@node1 ~]#  rbd flatten jacob-clone		#让新克隆的镜像与快照脱离关系
[root@node1 ~]#  rbd info jacob-clone			#查看镜像信息
rbd image 'jadob-clone':
	size 15360 MB in 3840 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.d3f53d1b58ba
	format: 2
	features: layering
	flags: 
#注意，父快照信息没了！
```

##### 3.删除快照

```shell
3）删除快照
[root@node1 ~]#  rbd snap unprotect jacob --snap jacob-snap1     #取消快照保护
[root@node1 ~]#  rbd snap rm jacob --snap jacob-snap1    		#可以删除快照
```

----------

## ceph文件系统

块设备仅允许同时一个客户端访问，无法实现多人同时使用块设备

等于说无法实现同时访问。

### 实现功能

延续前面的实验，实现Ceph文件系统的功能。具体实现有以下功能：

- 部署MDSs节点
- 创建Ceph文件系统
- 客户端挂载文件系统

### 实现方案

前面的块共享，仅允许同时一个客户端访问，无法实现多人同时使用块设备。

而Ceph的文件系统共享则允许多人同时使用。

下面假设使用虚拟机node3，部署MDS节点。

主机的主机名及对应的IP地址如表-1所示。   

表－1 主机名称及对应IP地址表

![image-20201104140443236](E:\fc-learn\Linux-learn\Cluster\image-20201104140443236.png)

### 实现步骤

实现此案例需要按照如下步骤进行。

#### 1.添加一台虚拟机node3，要求如下

IP地址:192.168.4.13

主机名:node3

配置yum源（包括操作系统的源、ceph的源）

与Client主机同步时间

node1允许无密码远程node3

修改node1的/etc/hosts，并同步到所有node主机

#### 2.部署元数据服务器

登陆node3，安装ceph-mds软件包（如果前面课程已经安装，此步骤可以忽略）

```shell
[root@node3 ~]# yum -y install ceph-mds 
```

登陆node1部署节点操作

```shell
[root@node1 ~]# cd  /root/ceph-cluster
#该目录，是最早部署ceph集群时，创建的目录

[root@node1 ceph-cluster]# ceph-deploy mds create node3
#远程nod3，拷贝集群配置文件，启动mds服务
```

#### 3.创建存储池

备注：一个文件系统是由inode和block两部分组成，效果如图-1所示。

inode存储文件的描述信息（metadata元数据），block中存储真正的数据。

![img](E:\fc-learn\Linux-learn\Cluster\image001.png)

```shell
[root@node3 ~]# ceph osd pool create cephfs_data 64
#创建存储池，共享池的名称为cephfs_data，对应有64个PG
#共享池名称可以任意
[root@node3 ~]# ceph osd pool create cephfs_metadata 64
#创建存储池，共享池的名称为cephfs_metadata，对应有64个PG
```

PG拓扑图如下：
PG是一个逻辑概念，没有对应的物质形态，是为了方便管理OSD而设计的概念。

为了方便理解，可以把PG想象成为是目录，可以创建32个目录来存放OSD，也可以创建64个目录来存放OSD。
![img](E:\fc-learn\Linux-learn\Cluster\image002.png)

#### 4.创建ceph文件系统

```shell
[root@node3 ~]# ceph fs new myfs1 cephfs_metadata cephfs_data
#myfs1是名称，名称可以任意，注意，先写metadata池，再写data池
#fs是filesystem的缩写，filesystem中文是文件系统
#默认，只能创建1个文件系统，多余的会报错

[root@node3 ~]# ceph fs ls
name: myfs1, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```

#### 5.客户端挂载

客户端需要安装ceph-common，前面的课程已经安装

```shell
[root@client ~]# mount -t ceph 192.168.4.11:6789:/  /mnt  \
-o name=admin,secret=AQBTsdRapUxBKRAANXtteNUyoEmQHveb75bISg==
#注意:-t（type）指定文件系统类型，文件系统类型为ceph
#-o（option）指定mount挂载命令的选项，选项包括name账户名和secret密码
#192.168.4.11为MON节点的IP（不是MDS节点），6789是MON服务的端口号
#admin是用户名,secret后面是密钥
#密钥可以在/etc/ceph/ceph.client.admin.keyring中找到
```

拓扑图如下：

![img](http://tts.tmooc.cn/ttsPage/LINUX/NSDTN202001/CLUSTER/DAY04/CASE/01/index.files/image003.png)

思考题：

1)根分区有100G，如果mkdir /{a,b}，则/a和/b存储空间分别是多少？

2)请问lsblk和df命令的区别？

3)如果做NFS的实验，客户端不安装nfs-utils是否可以mount成功？

\##扩展知识：创建ceph用户，查看用户##

```shell
[root@node1 ~]# ceph auth get-or-create client.nb \
osd  'allow *' \
mds  'allow *' \
mon  'allow *'      > 文件名
# >是重定向导出，后面的文件名可以任意，没有文件会创建，有文件则会覆盖文件的内容
[root@node1 ~]# ceph auth list              #查看所有用户列表
```

-------

## 创建对象存储服务器

对象存储：也叫基于对象的存储。

### 实现功能

延续前面的实验，实现Ceph对象存储的功能。具体实现有以下功能：

- 安装部署Rados Gateway
- 启动RGW服务
- 设置RGW的前端服务与端口
- 客户端测试

### 实现步骤

#### 步骤一：部署对象存储服务器

##### 1.准备实验环境，如下要求

IP地址:192.168.4.13

主机名:node3

配置yum源（包括操作系统的源、ceph的源）

与Client主机同步时间

node1允许无密码远程node3

修改node1的/etc/hosts，并同步到所有node主机

##### 2.部署RGW软件包

```shell
[root@node3 ~]# yum -y install ceph-radosgw
```

##### 3.新建网关实例

拷贝配置文件，启动一个rgw服务

```shell
[root@node1 ~]# cd /root/ceph-cluster
[root@node1 ~]# ceph-deploy rgw create node3		#远程mode3启动rgw服务
```

登陆node3验证服务是否启动

```shell
[root@node3 ~]# ps aux |grep radosgw
ceph      4109  0.2  1.4 2289196 14972 ?       Ssl  22:53   0:00 /usr/bin/radosgw -f --cluster ceph --name client.rgw.node3 --setuser ceph --setgroup ceph
[root@node3 ~]# systemctl  status ceph-radosgw@\*
```

##### 4.修改服务器端口

登陆node3，RGW默认服务端口为7480，修改为8000或80更方便客户端记忆和使用

```shell
[root@node3 ~]#  vim  /etc/ceph/ceph.conf
[client.rgw.node3]
host = node3
rgw_frontends = "civetweb port=8000"
#node3为主机名
#civetweb是RGW内置的一个web服务
[root@node3 ~]# systemctl  restart ceph-radosgw@\*
```

#### 步骤二：客户端测试

（扩展选做实验）

##### 1.curl测试

```shell
[root@client ~]# curl  192.168.4.13:8000
```

##### 2.使用第三方软件访问

登录node3（RGW）创建账户

```shell
[root@node3 ~]#  radosgw-admin user create \
--uid="testuser" --display-name="First User"
… …
"keys": [
        {
            "user": "testuser",
            "access_key": "5E42OEGB1M95Y49IBG7B",
            "secret_key": "i8YtM8cs7QDCK3rTRopb0TTPBFJVXdEryRbeLGK6"
        }
    ],
[root@node5 ~]# radosgw-admin user info --uid=testuser
//testuser为用户名，access_key和secret_key是账户密钥
```

##### 3.客户端安装软件

（软件需要自己上网搜索下载）

```shell
[root@client ~]#  yum install s3cmd-2.0.1-1.el7.noarch.rpm
```

修改软件配置（注意，除了下面设置的内容，其他提示都默认回车）

```shell
[root@client ~]#  s3cmd --configure
Access Key: 5E42OEGB1M95Y49IBG7B
Secret Key: i8YtM8cs7QDCK3rTRopb0TTPBFJVXdEryRbeLGK6
S3 Endpoint [s3.amazonaws.com]: 192.168.4.13:8000
[%(bucket)s.s3.amazonaws.com]: %(bucket)s.192.168.4.13:8000
Use HTTPS protocol [Yes]: No
Test access with supplied credentials? [Y/n] n
Save settings? [y/N] y
#注意，其他提示都默认回车
```

##### 4.创建存储数据的bucket

（类似于存储数据的目录）

```shell
[root@client ~]# s3cmd ls
[root@client ~]# s3cmd mb s3://my_bucket
Bucket 's3://my_bucket/' created
[root@client ~]# s3cmd ls
2018-05-09 08:14 s3://my_bucket

[root@client ~]# s3cmd put /var/log/messages s3://my_bucket/log/

[root@client ~]# s3cmd ls s3://my_bucket
DIR s3://my_bucket/log/
[root@client ~]# s3cmd ls s3://my_bucket/log/
2018-05-09 08:19 309034 s3://my_bucket/log/messages 
```

##### 5.测试下载功能

```shell
[root@client ~]# s3cmd get s3://my_bucket/log/messages /tmp/
```

##### 6.测试删除功能

```shell
[root@client ~]# s3cmd del s3://my_bucket/log/messages
```

--------

# ceph 使用思路总结

### 一、准备工作：


IP，主机名，hosts解析，ssh密钥，时间同步，yum源，防火墙，selinux


### 二、部署ceph：


#### 1.安装软件


```
  ceph-deploy(脚本)
  ceph-mon  ceph-osd  ceph-mds  ceph-radosgw(集群)
```


#### 2.修改配置启动服务mon


```
  mkdir  目录；cd 目录
  ceph-deploy  new  node1   node2   node3  (生成配置文件)
  ceph-deploy  mon  create-initial  (启动服务)
```


#### 3.启动osd共享硬盘


```
  ceph-deploy  disk  zap   主机名:磁盘名  ...  ...
  ceph-deploy  osd  create  主机名:磁盘   ...  ...
```


### 三、使用Ceph思路:


#### 1.块共享


```
  服务器: rbd  create  创建一个共享镜像
  客户端: 安装cpeh-common;  cp 配置文件和密钥
          rbd  map  |  rbd  unmap
```


#### 2.文件系统共享(文件系统由inode和block)


服务器: 创建两个共享池(名称任意)

使用两个共享池合并一个文件系统


安装ceph-mds软件，并启动服务(ceph-deploy  mds  create  node3)


```
  客户端： mount  -t  MON的IP:6789:/   /挂载点   -o  name=用户名,secret=密码
```

#### 3.对象存储   

服务器启动一个radosgw即可(RGW)  

```
   ceph-deploy  rgw  create  node3
```