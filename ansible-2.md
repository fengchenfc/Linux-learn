# Absible 2

## sudo账号提权

### sudo概述

- sudo

  - superuser or anther do
  - 以超级管理员或其他人的身份执行命令

- 基本流程

管理员需要先授权（修改/etc/suduers文件）

普通用户以sudo的形式执行命令

可以通过sudo-l查看授权情况

- 修改/etc/sudoers的方法
1.	visudo（带语法检查，默认没有颜色提示）
2. vim /etc/sudoers（不带语法检查，默认有颜色提示）

- 授权格式

用户或组     主机列表=(提权身份)     [NOPASSWD]:命令列表      

**注意事项：命令需要写绝对路径，对组授权需要在组名称前面加%。**

```shell
[root@control ~]# cat  /etc/sudoers         #不要改，下面仅仅是语法格式的示例（例子）
… …
root           ALL=(ALL)       ALL
tom            ALL=(root)      /usr/bin/systemctl
%wheel         ALL=(ALL)       ALL
```

## 配置sudo权限

### 要求

- 给所有被管理主机创建系统账户
- 账户名称为alice，密码为123456
- 修改sudo配置，让alice可以执行任何管理命令

#### 步骤一：配置sudo提权

##### 1.远程所有被管理主机批量创建系统账户，账户名称为alice，密码为123456

```shell
[root@control ansible]# ansible all -m user -a "name=alice \
password={{'123456' | password_hash('sha512')}}"
```

##### 2.配置alice账户可以提权执行所有命令（control批量授权，node1主机验证）

使用lineinfile模块修改远程被管理端主机的/etc/sudoers文件，line=后面的内容是需要添加到文件最后的具体内容。    

等于是在/etc/sudoers文件末尾添加一行:alice  ALL=(ALL)  NOPASSWD:ALL

```shell
[root@control ansible]# ansible all -m lineinfile \
-a "path=/etc/sudoers line='alice  ALL=(ALL) NOPASSWD:ALL'"
[root@control ansible]# ssh alice@node1
```

验证:

可以在node1电脑上面使用alice用户执行sudo重启服务的命令看看是否成功。

```shell
[alice@node1 ansible]$ sudo systemctl restart sshd  #不需要输入密码
```

## 修改Ansible配置

### 要求

- 修改主配置文件
- 设置ansible远程被管理端主机账户为alice
- 设置ansible远程管理提权的方式为sudo
- 修改主机清单文件
- 修改主机清单配置文件，添加SSH参数

#### 步骤一：**配置普通用户远程管理其他主机**

##### 1.修改主配置文件，配置文件文件的内容可以参考/etc/ansible/ansible.cfg

```shell
[root@control ansible]# vim ~/ansible/ansible.cfg
[defaults]
inventory = ~/ansible/inventory
remote_user = alice      #以什么用户远程被管理主机（被管理端主机的用户名）
[privilege_escalation]
become = true            #alice没有特权，是否需要切换用户提升权限
become_method = sudo      #如何切换用户（比如用su就可以切换用户，这里是sudo）
become_user = root          #切换成什么用户（把alice提权为root账户）
become_ask_pass = no        #执行sudo命令提权时是否需要输入密码
```

##### 2.远程被管理端主机的alice用户，需要提前配置SSH密钥

```shell
[root@control ansible]# for i in node1  node2  node3  node4  node5
do
  ssh-copy-id    alice@$i
done
```

验证：

```shell
[root@control ansible]# ssh alice@node1            #依次远程所有主机看看是否需要密码
#注意：是远程登录node1，应该输入的是node1电脑上面alice账户的密码，control没有alice用户
[root@control ansible]# ansible all -m command -a  "who"              #测试效果
[root@control ansible]# ansible all -m command -a  "touch /test"     #测试效果
```

