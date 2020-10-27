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

##### 常见报错

（有问题可以参考，没问题可以忽略）

```shell
node1 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: alice@node1: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).",
    "unreachable": true
}
问题分析：
英语词汇：Failed（失败），connect（连接），to（到），host（主机），via（通过）
permission（权限），denied（被拒绝）
Failed to connect to host via ssh alice@node1（通过ssh使用alice远程连接到主机失败）
Permission denied（因为无法连接，所以报错说权限被拒绝）
解决办法：手动ssh alice@主机名（如node1），看看是否可以实现免密码登录。
          Ansible的原理是基于ssh远程管理，如果无法实现alice免密码登录，则实验会失败！
        如何实现免密码登录，可以参考案例上面的命令，或者第一阶段相关知识。
```

##### 3.修改inventory主机清单配置文件（参考即可，不需要操作）

如果个别主机的账户不同，该如何处理呢？

如果有些主机需要使用密码远程呢？如果有些主机的SSH端口不是22呢？

```shell
[root@control ~]# cat  ~/ansible/inventory
[test]                    
node1           ansible_ssh_port=端口号       #自定义远程SSH端口
[proxy]
node2           ansible_ssh_user=用户名    #自定义远程连接的账户名
[webserver]
node[3:4]       ansible_ssh_pass=密码       #自定义远程连接的密码
[database]
node5
[cluster:children]                
webserver
database
```

--------

## Ansible Playbook

### Playbook概述

- Ansible ad-hoc可以通过命令行形式远程管理其他主机
  - 适合执行一些**临时性简单任务**

- Ansible Playbook中文名称叫剧本
  - 将经常需要执行的任务写入一个文件(剧本)
  - 剧本中可以包含多个任务
  - 剧本写后，我们随时根据剧本，执行相关的任务命令
  - playbook剧本要求按照**YAML格式**编写
  - 适合执行周期性**经常执行的复杂任务**

### YAML简介

#### YAML是什么

- YAML是一个可读性高、用来表达数据序列的格式语言
- YAML：YAML Ain't a Markup Language
- YAML以数据为中心，重点描述数据的关系和结构

#### YAML的格式要求

- "#"代表注释，一般第一行为三个横杠（---）
- **键值（key/value）对使用":"表示，数组使用"-"表示，"-"后面有空格**
- key和value之间
- 使用":"分隔
- ":"后面**必须有空格**
- 一般缩进由两个或以上空格组成
- 相同层级的缩进**必须对齐**，缩进代表层级关系
- **全文不可以使用tab键**
- 区分大小写
- 扩展名为yml或者yaml
- 跨行数据需要使用>或者|，其中|会保留换行符