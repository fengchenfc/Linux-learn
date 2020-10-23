# VPN
## vpn概述

**Virtual Private Network**  虚拟专用网络

- 在<u>公司网络上</u>建立<u>专用私用网络</u>进行加密通讯
- 多用于为集团公司的各地子公司建立连接
- 连接完成后，各地区的子宫给弄死可以像局域网一样通讯
- 在企业网络中有广泛应用
- **可以科学上网！！！**

目前主流的VPN技术

- GRE
- PPTP
- L2TP+IPSec
- SSL

## GRE技术

### GRE模块

**缺点：**

- **缺少加密机制**
- **仅Linux可用**

ip_gre		#Linux内核模块

modprobe  ip_gre		#加载模板

lsmod | grep ip_gre		#显示模块列表

modinfo	ip_gre		#查看模块信息

### 搭建方式

#### Client

步骤：

启用GRE模块

加载模块ip_gre

查看模块信息

创建隧道

启用隧道（类似于设置网卡up）

为VPN配置隧道IP地址

```shell
[root@web1 ~]# modprobe ip_gre		//加载gre内核模块功能

[root@web1 ~]# lsmod | grep ip_gre		//查询是否开启
ip_gre                 22707  0 
ip_tunnel              25163  1 ip_gre
gre                    13144  1 ip_gre

[root@web1 ~]# ip tunnel add tun0 mode gre remote 192.168.2.200 local 192.168.2.100		//创建名为tun0，使用gre技术，与2.200连接，自身ip是2.100的隧道

[root@web1 ~]# ip link show		//查看是否成功

[root@web1 ~]# ip addr add 10.10.10.10/24 peer 10.10.10.5/24 dev tun0		//在tun0隧道中使用私有ip地址。本机配置IP10.10,远程主机（隧道对面）IP是10.5
[root@web1 ~]# ip addr show		//查看

[root@web1 ~]# ip link set tun0 up		//激活以上配置
```

删除隧道方法

```shell
ip tunnel del tun0		//删除已配隧道方法（如果配错的话）
```

#### proxy

步骤：

启用GRE模块

加载模块ip_gre

查看模块信息

创建隧道

启用隧道（类似于设置网卡up）

为VPN配置隧道IP地址

测试连通性

```shell
[root@web2 ~]# modprobe ip_gre		//加载gre内核模块功能
[root@web2 ~]# lsmod | grep ip_gre		//查询是否开启
[root@web2 ~]#ip tunnel add tun0 mode gre remote 192.168.2.100 local 192.168.2.200		//创建隧道
[root@web2 ~]#ip link show		
[root@web2 ~]#ip addr add 10.10.10.5/24 peer 10.10.10.10/24 dev tun0		//配置隧道IP

[root@web2 ~]#ip link set tun0 up		//激活隧道配置
[root@web2 ~]#ip addr s		
[root@web2 ~]#ping 10.10.10.5
[root@web2 ~]#ping 10.10.10.10
```

至此，两台主机实现点到点的隧道通讯，实验完成。

## PPTP VPN

### PPTP VPN 概述

- PPTP(Point to Point Tunneling Protocol)

- 支持密码身份验证
- 支持MPPE(Microsoft Point-to-Point Encryption)加密

- 支持windows

### 搭建方式









## L2TP+IPSec