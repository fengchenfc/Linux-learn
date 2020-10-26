# Ansible

## Ansible基础

### 简介

#### 工具历程

- Ansible首次发布于2012年
- 作者Michael DeHaan（他也是Cobber的作者）
- 在2015年被红帽收购

#### 功能及特点

Ansilbe时一款**自动化运维**工具，基于python开发：

- 批量系统配置
- 批量程序部署
- 批量运行命令功能
- 批量修改服务器密码
- 批量安装软件包
- 批量修改配置

#### Ansible原理

控制端主机自带很多模块（模块就是脚本）；

ansible通过ssh远程被管理主机，将控制端的模块（脚本）或命令传输到被管理主机；

在被管理端主机执行模块（脚本）或命令，执行不同的模块或命令可以实现不同的功能；

最后ansible退出ssh远程。

绝大多数模块（脚本）都需要参数才能执行成功！！！类似于shell脚本的位置变量！

![image-20201026095100076](E:\fc-learn\Linux-learn\image-20201026095100076.png)

#### Ansible产品特色

![image-20201026101712076](E:\fc-learn\Linux-learn\image-20201026101712076.png)

### ansible环境部署

#### 主机列表

| 主机名  | IP地址        | 角色                   |
| ------- | ------------- | ---------------------- |
| control | 192.168.4.253 | 控制节点（manager）    |
| node1   | 192.168.4.11  | 被控制节点（test）     |
| node2   | 192.168.4.12  | 被控制节点（proxy）    |
| node3   | 192.168.4.13  | 被控制节点（web1）     |
| node4   | 192.168.4.14  | 被控制节点（web2）     |
| node5   | 192.168.4.15  | 被控制节点（database） |

#### 一、准备基础环境

在控制节点：

- 域名解析
- 配置SSH密钥（ansible是基于ssh实现远程控制）
- 安装ansible

##### 1）Control控制节点

###### 域名解析

修改/etc/hosts，在文件中手动添加如下内容，修改该文件的目的是做域名解析。

```shell
[root@control ~]# vim  /etc/hosts		#修改文件，手动添加如下内容（不要删除文件原来的内容）
192.168.4.253	control	
192.168.4.11	node1	
192.168.4.12	node2	
192.168.4.13	node3	
192.168.4.14	node4	
192.168.4.15	node5

[root@control ~]# ping  node1      #验证是否成功，依次ping
```

###### 配置SSH密钥

实现免密码登录（非常重要）

Ansible是基于SSH远程的原理实现远程控制，如果控制端主机无法免密登录被管理端主机，后续的所有试验都会失败！！

#拷贝密钥到远程主机
#提示：拷贝密钥到远程主机时需要输入对方电脑的账户密码才可以！！
#拷贝密钥到node1就需要输入node1对应账户的密码，拷贝密钥到node2就需要输入node2对应的密码

```shell
[root@control ~]#  ssh-keygen        #生成ssh密钥
写脚本拷贝密钥到各node主机/直接在命令行敲脚本实现
[root@control ~]#  for i in node1 node2 node3 node4 node5
do
ssh-copy-id   $i 
done			//脚本内容

[root@control ~]# ssh  node1      #使用ssh命令依次远程所有主机都可以免密码登录
[root@node1 ~]# exit          #退出ssh远程登录 
```

###### 2)部署Ansible软件

（仅Control主机操作，软件包在ansible_soft目录）。

```shell
[root@control ~]# tar -xf   ansible_soft.tar.gz
[root@control ~]# cd ansible_soft
[root@control ansible_soft]# ls
ansible-2.8.5-2.el8.noarch.rpm     python3-bcrypt-3.1.6-2.el8.1.x86_64.rpm  python3-pynacl-1.3.0-5.el8.x86_64.rpm
libsodium-1.0.18-2.el8.x86_64.rpm  python3-paramiko-2.4.3-1.el8.noarch.rpm  sshpass-1.06-9.el8.x86_64.rpm
[root@control ansible_soft]# dnf  -y  install   *	//六个包，全部安装
```



被控制节点要求：

- Ansible默认通过SSH协议管理机器
- 被管理主机要开启SSH服务，并允许控制主机登录
- 被管理主机需要安装有Python



#### 二、修改配置文件

##### 主配置文件说明

主配置文件ansible.cfg（主配置文件内容可参考/etc/ansible/ansible.cfg）

**ansible配置文件查找顺序**

<u>首先</u>检测ANSIBLE_CONFIG变量定义的配置文件（默认没有这个变量）

<u>其次</u>检查当前目录下的./ansible.cfg文件

<u>再次</u>检查当前用户家目录下~/ansible.cfg文件

<u>最后</u>检查/etc/ansible/ansible.cfg文件

##### 1) 修改主配置文件

```shell
[root@control ~]# mkdir  ~/ansible
[root@control ~]# vim  ~/ansible/ansible.cfg
[defaults]
inventory = ~/ansible/inventory            
#主机清单配置文件（inventory可以是任意文件名），英语词汇：inventory（清单、财产清单）
#forks = 5          #ssh并发数量
#ask_pass = True        #使用密钥还是密码远程，True代表使用密码
#host_key_checking = False      #是否校验密钥（第一次ssh时是否提示yes/no）
```

##### 2) 修改主机清单文件

##### （清单文件名必须与主配置文件inventory定义的一致）。

```shell
[root@control ~]# vim  ~/ansible/inventory
[test]        #定义主机组（组名称任意）
node1         #定义组中的具体主机，组中包括一台主机node1
[proxy]         #定义主机组（组名称任意），英语词汇：proxy（代理人，委托人）
node2          #proxy组中包括一台主机node2
[webserver]
node[3:4]        #这里的node[3:4]等同于node3和node4
[database]
node5
[cluster:children]      #嵌套组（children为关键字），不需要也可以不创建嵌套组
webserver            #嵌套组可以在组中包含其他组
database
```