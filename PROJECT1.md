准备9台虚拟机环境

准备系统，修改单主机主机名，配置IP、网关；

client:      ip :192.168.4.10/24 

proxy:		ip:192.168.2.5/24		192.168.4.5/24

proxy2:		ip: 192.168.4.6/24		192.168.2.6/24

web1:		192.168.2.11/24

web2:		192.168.2.12/24

web3:		192.168.2.13/24

database:		192.168.2.21/24

nfs:		192.168.2.31/24

------

思路：

本着一个思路，只从一台主机去配置所有服务端

1.配ip  

2.以代理服务器proxy为主

配置域名解析，安装ansible工具，分发密钥

测试（ping）是否成功，testping.yml

安装各主机yum环境，yumready.yml

设置selinux

设置防火墙，允许80&443端口tcp ，selinux-firewall-set.yml

同步所有服务端域名解析 ，synchronous-hosts.yml

3.配好proxy上的nginx代理

解压tar包，安装配套工具，

roles里：安装nginx

file：nginx安装包，（是否要有ceph安装源？还是等到时候装到nfs里呢？）

> 监控cpu负载？ xx时间检查，超过xx给xx发邮件？
>
> nginx是不是应该用nginx用户，nologin解释器？
>
> nfs那块 做集群存储，需要做ntp时间同步！







接下来我们就用with_item属性
item相当于for循环里面的i
with_items取值，但不支持通配符

---------

在porxy上操作：

1.安装VIM、bash-completion，ansible

2.修改/etc/hosts ，使后续操作更方便

```bash
[root@proxy ansible]# vim /etc/hosts
192.168.2.5 proxy
192.168.2.6 proxy2
192.168.2.11 web1
192.168.2.12 web2
192.168.2.13 web3
192.168.2.21 database
192.168.2.31 nfs
```

3. 生成key，分发key

```bash
[root@proxy ansible]# ssh-keygen -f /root/.ssh/id_rsa -N ''
[root@proxy ansible]#  for i in proxy proxy2 web1 web2 web3 database nfs; do ssh-copy-id  $i; done
```

4.



