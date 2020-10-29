# Ansible 1

## Ansible基础

### 简介

#### 工具历程

- Ansible首次发布于2012年
- 作者Michael DeHaan（他也是Cobber的作者）
- 在2015年被红帽收购

#### 功能及特点

Ansilbe是一款**自动化运维**工具，基于python开发：

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

> ansible默认有个all组，这个组包含所有主机

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

-------

## Ansible ad-hoc

Ansible ad-hoc是一种通过命令行批量管理的方式，命令基本格式如下：

格式：**ansible 主机集合 -m 模块名 -a "参数"**

> ansible键入命令后：
>
> 得到的回复是粉色（紫色），说明执行成功，但是不专业等；
>
> 得到的回复是黄色的内容，说明此命令确实执行了操作。成功；
>
> 得到的恢复时绿色的内容，说明此命令完成，但是没操作（没干活本身就成功的）

### Ansible ad-hoc应用一

#### 应用要求

- 测试主机列表中的主机是否可以ping通
- 查看被管理主机的服务器信息（如时间、版本、内存等）
- 学习ansible-doc命令的用法
- 测试command与shell模块的区别
- 使用script模块在远程主机执行脚本（装软件包、启服务）

#### 步骤

##### 步骤一：测试环境

###### 1.查看主机列表

```shell
[root@control ~]# cd  ~/ansible          #非常重要
[root@control ansible]# ansible  all  --list-hosts           #查看所有主机列表
```

- --list-hosts是ansible这个命令的固定选项，如同ls -a一样（-a是ls命令的固定选项）

- #英语词汇：list（列表，清单）、host（主机、主办、主人）

###### 2.测试远程主机是否能ping通

- 当需要远程多个主机或者多个组时，中间使用逗号分隔！！！

```shell
[root@control ansible]# ansible  node1  -m  ping          	#调用ping模块
[root@control ansible]# ansible  node1,webserver  -m  ping
```

- 常见报错（有问题可以参考，没问题可以忽略）

```shell
node1 | UNREACHABLE! => {
"changed": false, 
"msg": "Failed to connect to the host via ssh: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).", 
"unreachable": true
}
```

问题分析：
英语词汇：Failed（失败），connect（连接），to（到），host（主机），via（通过）
permission（权限），denied（被拒绝）
Failed to connect to host via ssh（通过ssh远程连接到主机失败）
Permission denied（因为无法连接，所以报错说权限被拒绝）
解决办法：手动ssh其他主机（如node1），看看是否可以实现免密码登录。
           Ansible的原理是基于ssh远程管理，如果无法实现免密码登录，后面的实验无法成功！
           如何实现免密码登录，可以参考案例上面的命令，或者第一阶段知识。

**提示：该模块虽然叫ping，但是它不会发送任何ICMP协议的ping数据包，控制端主机仅仅是ssh远程被管理端主机，检查其是否有python环境，能顺利远程并且有Python环境就会返回正确的提示信息，否则报错。**

![image-20201026141346728](E:\fc-learn\Linux-learn\image-20201026141346728.png)

###### 3.快速入门

模块就是脚本（多数为Python脚本），多数脚本都支持参数，默认模块为command。

```shell
[root@control ansible]# ansible  node1  -m  command  -a   "uptime"     #查看CPU负载
[root@control ansible]# ansible  node1  -m command -a  "uname -r"      #查看内核版本
[root@control ansible]# ansible  node1   -a   "ip a s"                  #查看网卡信息
[root@control ansible]# ansible  all   -a   "date"                      #查看时间
```

![image-20201026141611858](E:\fc-learn\Linux-learn\image-20201026141611858.png)



**通过ansible-doc获取帮助**

```shell
[root@control ansible]# ansible-doc  -l      #列出所有模块
[root@control ansible]# ansible-doc -l | grep yum      #在所有模块中过滤关键词
[root@control ansible]# ansible-doc yum        #查看模块帮助
```

###### 4.SHELL模块

<u>command模块，不支持**管道、重定向、&、>**等命令；也就是说**command不支持bash的特性**</u>

<u>**所有需要调用shell的功能command都无法使用**</u>

> bash有哪些特性可以参考Shell课程第一天的PPT，如管道和重定向等功能，但是shell模块可以支持。

不可以使用shell模块执行**交互命令**，如vim、tio等。

```shell
[root@control ansible]# ansible test -m command -a "ps | wc -l"      #报错
[root@control ansible]# ansible test -m command -a  "ls &"        #报错

[root@control ansible]# ansible test -m shell -a  "ps aux | wc -l"       #进程数量
[root@control ansible]# ansible test -m shell -a  "who"                   #登陆信息
[root@control ansible]# ansible test -m shell -a  "touch /tmp/txt.txt"  
#使用shell模块创建文件会有Warning警告提示，正常！！！
```

总结shell与command：

shell比command支持更广，能用command的地方必能用shell

shell运行时要比command占用资源稍高一些

###### 5.script模块

script模块会把-a后面的脚本拷贝到被管理端主机，然后执行这个脚本。

示例：

编写脚本安装httpd服务，到被控制端的主机验证

```shell
[root@control ansible]# vim  ~/ansible/test.sh  
#!/bin/bash
dnf -y install httpd
echo "test script" > /var/www/html/index.html
systemctl start httpd
[root@control ansible]# ansible  test  -m script  -a  "./test.sh"    
#test是主机组的名称，-m调用script模块，-a后面的./test.sh是上面创建脚本的相对路径和文件名
#./是当前目录的意思，在当前目录下有个脚本叫test.sh
```

![image-20201026152519339](E:\fc-learn\Linux-learn\image-20201026152519339.png)

验证（ssh到被控制端主机）

```shell
[root@node1 ~]# rpm -q  httpd		#查看是否安装http
[root@node1 ~]# systemctl  status  httpd	#查看httpd服务状态
[root@node1 ~]# ss -nulpt | grep httpd 	#查看端口状态

[root@node1 ~]# curl http://localhost		#访问本地网页
test script		#本地页面内容
```



### Ansible ad-hoc应用二

很多ansible模块（不是所有）都具有**幂等性**的特征。

幂等性：任意次执行所产生的影响均与一次执行的影响相同。

#### 应用要求

- 远程目标主机新建文件和目录、修改文件或目录的权限
- 在远程目标主机创建链接文件
- 删除远程目标主机上的文件或目录
- 将控制端本地的文件拷贝到被管理端
- 从被管理端下载文件到本地
- 修改远程目标主机上的文件内容

#### 步骤

##### 步骤一：file模块

file模块可以创建/删除文件、目录、链接；修改权限与属性等（ansible-doc file）

###### 创建文档

```shell
[root@control ansible]# ansible  test  -m  file  -a  "path=/tmp/file.txt state=touch"     
#远程test组中所有主机，新建文件，path后面指定要创建的文件或目录的名称
#state=touch是创建文件，state=directory是创建目录
```

验证： 到node1主机，使用ls /tmp/file.txt看看文件是否被创建成功 

<u>**!!注意！不要用此模块再执行进行幂等性验证！因为：**</u>

<u>**touch作用：**</u>

<u>**1.创建文本**</u>

<u>**2.修改文件时间**</u>

<u>**所以，上述操作流程无法验证文件幂等性!!**</u>

![image-20201026161826227](E:\fc-learn\Linux-learn\image-20201026161826227.png)

1. 控制端主机**ssh远程**node1，将**file.py**脚本拷贝到node1

2. 在node1主机**执行./file.py**  path=/tmp/file.txt  state=touch

   ​	备注：file.py脚本必须有参数才可以执行，否则脚本不知道要干嘛

   ​			    file.py脚本的参数是path=/tmp/file.txt	state=touch

3. 控制端退出远程连接

----

###### 常见错误

1. 值错了

```shell
node1 | FAILED! => {
   … …
    "changed": false,
    "msg": "value of state must be one of: absent, directory, file, hard, link, touch, got: touc"
}
英语词汇：value（值），must（必须），be（是），of（…的），one（一个）
value of state must be one of:【state的值必须是后面给出的其中一个值】
解决办法：检查state的值是否有字母错误,上面报错例子中输入的是touc，不是touch。
```

2.	变量错了
```shell
node1 | FAILED! => {
   … …
   "msg": "Unsupported parameters for (file) module: nmae Supported parameters include: _diff_peek, _original_basename, access_time, 
access_time_format, attributes, backup, content, delimiter, directory_mode,
 follow, force, group, mode, modification_time, modification_time_format, owner,
 path, recurse, regexp, remote_src, selevel, serole, setype, seuser, src, state,
 unsafe_writes"
}
英语词汇：unsupported（不支持的），parameters（参数），supported（支持的）include(包括)
问题分析：file模块不支持nmae这个参数，它支持的参数包括哪些，后面有提示.
解决办法：检查模块的参数是否有字母错误，上面错误案例将name错写为nmae。
```

###### 创建目录

```shell
[root@control ansible]# ansible  test  -m  file  \
-a  "path=/tmp/mydir state=directory"       
#远程test组中所有主机，创建目录，path后面指定要创建的文件或目录的名称
## 验证：到node1主机，使用ls /tmp/看看tmp目录下是否有mydir子目录
```

执行结果，得到黄色的输出结果，内容代表执行并创建成功

（关键词：CHANGED）

```shell
node1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": true,
    "gid": 0,
    "group": "root",
    "mode": "0755",
    "owner": "root",
    "path": "/tmp/kongmi.dir",
    "secontext": "unconfined_u:object_r:user_tmp_t:s0",
    "size": 6,
    "state": "directory",
    "uid": 0
}
```

此时，再次反复执行同命令（创建同名目录），得到绿色的输出结果

**<u>!!此指令可验证文件幂等性!!</u>**

(关键词:SUCCESS)，证明此次指令执行输入后结果：未执行但达到指令目的

```shell
node1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "gid": 0,
    "group": "root",
    "mode": "0755",
    "owner": "root",
    "path": "/tmp/kongmi.dir",
    "secontext": "unconfined_u:object_r:user_tmp_t:s0",
    "size": 6,
    "state": "directory",
    "uid": 0
}
```

###### 修改文件或目录权限

```shell
[root@control ansible]# ansible  test  -m  file \
-a  "path=/tmp/file.txt owner=sshd group=adm mode=0777"  
#修改文件或目录权限，path后面指定要修改的文件名或目录名称，owner后面指定用户，group后面指定组，mode后面指定要修改的权限（0777中第一个0代表的是无特殊权限，如SUID、SGID等）
## 验证：到node1主机，使用ls -l /tmp/file.txt查看文件的详细信息是否正确
```

> 补充内容：在权限管理中，mode除了经典ugo三个参数以
>
> 前面还有一位可以表达特殊权限
>
> suid：4		sgid：2		sticky：1
>
> 例：chmod	0777	/opt/test

###### 删除目录/文件

```shell
[root@control ansible]# ansible test -m file -a "path=/tmp/mydir state=absent"
#state=absent代表删除（删除目录）
[root@control ansible]# ansible test -m file -a "path=/tmp/file.txt state=absent"
# state=absent代表删除（删除文件）
```

###### 创建链接

```shell
[root@control ansible]# ansible test -m file \
-a "src=/etc/hosts  path=/tmp/host.txt state=link"  
#给/etc/hosts文件创建一个链接文件/tmp/host.txt（src指定源文件，path是软链接文件名）
#相当于执行命令 ln -s  /etc/hosts  /tmp/host.txt
## 验证：到node1主机使用ls -l  /tmp/hosts查看文件是否为软链接
```

-----

##### 步骤二：copy模块

copy模块可以将文件拷贝到远程主机 (ansible-doc copy)。

```shell
[root@control ansible]# echo AAA > ~/a3.txt                   #新建测试文件
[root@control ansible]# ansible test -m copy -a "src=~/a3.txt dest=/root/"
#把管理端本机的a3.txt文件，拷贝到test组中所有主机的/root/目录
#src代表源文件，dest代表目标文件
## 验证：到node1主机使用ls /root/a3.txt查看是否有该文件
```

![image-20201026172109314](E:\fc-learn\Linux-learn\image-20201026172109314.png)

##### 步骤三：fetch模块

可以将其他主机的文件拷贝到本地(ansible-doc fetch)。

fetch模块与copy类似，但是作用相反。

```shell
[root@control ansible]# ansible test -m fetch -a "src=/etc/hostname   dest=~/"
#将远程test组中所有主机的hostname文件下载到本地家目录
#src代表源文件，dest代表目标文件
[root@control ansible]# ls  ~/          #使用ls查看下是否下载成功
#不能下载目录，如果需要下载目录，可以先打包后再下载
```

![image-20201026172257324](E:\fc-learn\Linux-learn\image-20201026172257324.png)

如果使用fetch模块，从多台被控制端主机拷贝同名文件到控制机，不会覆盖，会在控制端主机目的目录下，分别创建以被控制主机命名的目录，其中包含源

```shell
[root@control ansible]# ansible node1,node2,node3 -m fetch -a "src=/etc/hosts dest=~/"
node1 | CHANGED => {
    "changed": true,
    "checksum": "7335999eb54c15c67566186bdfc46f64e0d5a1aa",
    "dest": "/root/root/node1/etc/hosts",
    "md5sum": "54fb6627dbaa37721048e4549db3224d",
    "remote_checksum": "7335999eb54c15c67566186bdfc46f64e0d5a1aa",
    "remote_md5sum": null
}
node2 | CHANGED => {
    "changed": true,
    "checksum": "7335999eb54c15c67566186bdfc46f64e0d5a1aa",
    "dest": "/root/root/node2/etc/hosts",
    "md5sum": "54fb6627dbaa37721048e4549db3224d",
    "remote_checksum": "7335999eb54c15c67566186bdfc46f64e0d5a1aa",
    "remote_md5sum": null
}
node3 | CHANGED => {
    "changed": true,
    "checksum": "7335999eb54c15c67566186bdfc46f64e0d5a1aa",
    "dest": "/root/root/node3/etc/hosts",
    "md5sum": "54fb6627dbaa37721048e4549db3224d",
    "remote_checksum": "7335999eb54c15c67566186bdfc46f64e0d5a1aa",
    "remote_md5sum": null
}

[root@control ansible]# ls ~
a3.txt  anaconda-ks.cfg  ansible  ansible_soft  ansible_soft.tar.gz  keypass.sh  node1  node2  node3
```

##### 步骤四：lineinfile | replace 模块

在修改单个文件的单行内容时可以使用lineinfile模块(ansible-doc lineinfile)

lineinfile会替换一整行

replace可以替换关键词(ansible-doc replace)

本模块支持简单的修改，复杂的修改推荐还是用shell脚本

> 一般执行lineinfile，添加的内容会在目标文件中的最后面一行
>
> 如果需要插入到某指定位置，加入参数insert（插入），befor（在..之前）after（在..之后）
>
> lineinfile用法

```shell
[root@control ansible]# ansible test -m lineinfile  \
-a "path=/etc/issue line='hello world'"
#在/etc/issue文件中添加一行内容hello world，默认添加到最后，line后面跟的是需要添加的文件内容
## 验证：到node1主机执行命令cat /etc/issue查看文件内容是否正确

[root@control ansible]# ansible test -m lineinfile \
-a "path=/etc/issue line='hello world'"
#基于幂等原则，重复执行，不会创建多行内容

[root@control ansible]# ansible test -m lineinfile \
-a "path=/etc/issue line='insert' insertafter='Kernel'"
#将line后面的内容插入到/etc/issue文件中Kernel行的后面
#英语词汇：insert（插入），after（在…后面）
## 验证：到node1主机执行命令cat /etc/issue查看文件内容是否正确
```

replace用法

```shell
[root@control ansible]# ansible test -m replace \
-a "path=/etc/issue.net regexp=Kernel replace=Ocean"
#将node1主机中/etc/issue.net文件全文所有的Kernel替换为Ocean
#regexp后面是需要替换的旧内容；replace后面是需要替换的新内容
# 验证：到node1主机执行命令cat /etc/issue.net查看文件内容是否正确
```

-----

### Ansible ad-hoc应用三

#### 应用要求

- 远程目标主机创建、删除系统账户；设置系统账户属性、修改账户密码
- 为目标主机创建、删除yum源配置文件；远程目标主机安装、卸载软件包
- 使用service模块管理远程主机的服务
- 创建、删除逻辑卷

#### 步骤

##### 步骤一：user模块

user模块可以实现Linux系统账户管理(ansible-doc user)

###### 最基本的用户创建

```shell
[root@control ansible]# ansible test -m user -a "name=tuser1"
#远程test组中的所有主机并创建系统账户tuser1，默认state的值为present，代表创建用户
## 验证：到node1主机执行命令id  tuser1查看是否有该用户
```

![image-20201027093751256](E:\fc-learn\Linux-learn\image-20201027093751256.png)

1. 控制端主机**ssh远程**node1，将**user.py**脚本拷贝到node1

2. 在node1主机**执行./user.py**  name=tuser1

   备注：相当于执行useradd tuser1（tuser1是useradd的参数，没有这个参数useradd不知道创建用户）

3. 控制端退出ssh远程连接



###### 进阶创建用户，指定信息（UID、组等）

```shell
[root@control ansible]# ansible test -m user -a \
"name=tuser2 uid=1010 group=adm groups=daemon,root home=/home/tuser2"
#创建账户并设置对应的账户属性，uid指定用户ID号，group指定用户属于哪个基本组
#groups指定用户属于哪些附加组，home指定用户的家目录
## 验证： 到node1主机执行命令id tuser2查看是否有该用户
```



###### 高级创建，指定用户密码并用hash加密

```shell
[root@control ansible]# ansible test -m user \
-a "name=tuser1 password={{'abc'| password_hash('sha512')}}"
#修改账户密码，用户名是tuser1，密码是abc，密码经过sha512加密
```

###### 删除用户

```shell
[root@control ansible]# ansible test -m user \
-a "name=tuser1 state=absent"
#删除账户tuser1，state=absent代表删除账户的意思，name指定要删除的用户名是什么
#账户的家目录不会被删除，相当于执行userdel tuser1
[root@control ansible]# ansible test -m user \
-a "name=tuser2 state=absent remove=true"
#删除tuser2账户同时删除家目录、邮箱，相当于执行userdel  -r  tuser2
```

##### 步骤二：yum_repository模块

使用yum_repository可以创建或修改yum源配置文件

（ansible-doc yum_repository）

```shell
[root@control ansible]# ansible test -m yum_repository \
-a "name=myyum description=hello baseurl=ftp://192.168.4.254/centos gpgcheck=no"
#新建一个yum源配置文件/etc/yum.repos.d/myyum.repo
#yum源文件名为myyum，该文件的内容如下：
[myyum]
baseurl = ftp://192.168.4.254/centos
gpgcheck = 0
name = hello
## 验证：到node1主机ls /etc/yum.repos.d/查看该目录下是否有新的yum文件

[root@control ansible]# ansible test -m yum_repository \
-a "name=myyum description=test baseurl=ftp://192.168.4.254/centos gpgcheck=yes gpgkey=…"
#修改yum源文件内容

[root@control ansible]# ansible test -m yum_repository -a "name=myyum state=absent"
#删除yum源文件myyum
```

##### 步骤三：yum模块

使用yum模块可以安装、卸载、升级软件包（ansible-doc yum）  

state: present(安装)|absent(卸载)|latest(升级)。

```shell
[root@control ansible]# ansible test -m yum -a "name=unzip state=present"
#安装unzip软件包，state默认为present，也可以不写
## 验证：到node1主机执行命令rpm -q unzip查看是否有该软件

[root@control ansible]# ansible test -m yum -a "name=unzip state=latest"
#升级unzip软件包，软件名称可以是*，代表升级所有软件包

[root@control ansible]# ansible test -m yum -a "name=unzip state=absent"
#调用yum模块，卸载unzip软件包，state=absent代表卸载软件
## 验证：到node1主机执行命令rpm -q unzip查看该软件是否已经被卸载
```

##### 步骤四: service模块（ansible-doc service）

service为服务管理模块（启动、关闭、重启服务等）      

state:started|stopped|restarted，     

enabled:yes设置开机启动。

```shell
[root@control ansible]# ansible test -m yum -a "name=httpd"
#调用yum模块，安装httpd软件包
## 验证：到node1主机执行命令rpm -q httpd查看该软件是否被安装
[root@control ansible]# ansible test -m service -a "name=httpd state=started"
#调用service模块，启动httpd服务
## 验证：到node1主机执行命令systemctl  status  httpd查看服务状态
[root@control ansible]# ansible test -m service -a "name=httpd state=stopped"
#调用service模块，关闭httpd服务
## 验证：到node1主机执行命令systemctl  status  httpd查看服务状态
[root@control ansible]# ansible test -m service -a "name=httpd state=restarted"
#调用service模块，重启httpd服务
[root@control ansible]# ansible test -m service -a "name=httpd enabled=yes"
#调用service模块，设置httpd服务开机自启
```

##### 步骤五：逻辑卷相关模块

（ansible-doc lvg、ansible-doc lvol）

提示：做实验之前需要给对应的虚拟机添加额外磁盘，并创建磁盘2个分区

提示：可以使用前面学习过的parted或fdisk命令给磁盘创建分区   

提示：这里的磁盘名称仅供参考，不要照抄！！！     

###### lvg模块

创建、删除卷组(VG)，修改卷组大小，     

state:present(创建)|absent(删除)。

```shell
 [root@control ansible]# ansible test -m yum -a "name=lvm2"
#安装lvm2软件包，安装了lvm2软件后，才有pvcreate、vgcreate、lvcreate等命令
[root@control ansible]# ansible test -m lvg -a "vg=myvg pvs=/dev/sdb1"
#创建名称为myvg的卷组，该卷组由/dev/sdb1组成
#注意：这里的磁盘名称要根据实际情况填写
## 验证：到node1主机执行命令pvs和vgs查看是否有对应的PV和VG

[root@control ansible]# ansible test -m lvg -a "vg=myvg pvs=/dev/sdb1,/dev/sdb2"
#修改卷组大小，往卷组中添加一个设备/dev/sdb2
```

###### lvol模块

创建、删除逻辑卷(LV)，修改逻辑卷大小，      

state:present(创建)|absent(删除)。

```shell
[root@control ansible]# ansible test -m lvol -a "lv=mylv vg=myvg size=2G"
#使用myvg这个卷组创建一个名称为mylv的逻辑卷，大小为2G
## 验证：到node1主机执行命令lvs查看是否有对应的LV逻辑卷

[root@control ansible]# ansible test -m lvol -a "lv=mylv vg=myvg size=4G"
#修改LV逻辑卷大小

[root@control ansible]# ansible test -m lvol -a "lv=mylv vg=myvg size=3G force=yes"
#缩减LV逻辑卷大小

[root@control ansible]# ansible test -m lvol -a "lv=mylv vg=myvg state=absent force=yes"
#删除逻辑卷，force=yes是强制删除
[root@control ansible]# ansible test -m lvg -a "vg=myvg state=absent"
#删除卷组myvg
```

## 附加思维导图

![img](E:\fc-learn\Linux-learn\image010.png)