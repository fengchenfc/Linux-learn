# Ansible 3

## Ansible进阶

### ansible应用

- 熟悉firewalld和template模块的使用
- 熟悉error处理机制
- 熟悉handlers任务
- 熟悉when条件判断
- 熟悉block任务块
- 熟悉loop循环的使用方法

#### firewalld模块

使用firewalld模块可以配置防火墙策略

```shell
[root@control ~]#  vim ~/ansible/firewall.yml
---
- hosts: test                           #hosts定义需要远程的主机
  tasks:                                 #tasks定义需要执行哪些任务
    - name: install firewalld.         #name为第一个任务定义描述信息
      yum:                               #第一个任务调用yum模块安装软件
        name: firewalld                 #需要安装的软件名称为firewalld
        state: present                  #state等于present代表安装软件
    - name: run firewalld.             #定义第二个任务的描述信息
      service:                          #第二个任务调用service模块启动服务
        name: firewalld                #启动的服务名称为firewalld
        state: started                 #state等于started代表启动服务
        enabled: yes                    #enabled等于yes是设置服务为开机自启动
    - name: set firewalld rule.       #第三个任务的描述信息
      firewalld:                        #第三个任务调用firewalld模块设置防火墙规则
        port: 80/tcp                    #在防火墙规则中添加一个放行tcp，80端口的规则
        permanent: yes                  #permaenent 是设置永久规则
        immediate: yes                  #immediate 是让规则立刻生效
        state: enabled                  #state等于enabled是添加防火墙规则
#最终：在默认zone中添加一条放行80端口的规则
```

#### **template模块**

copy模块可以将一个文件拷贝给远程主机，但是如果希望每个拷贝的文件内容都不一样呢？如何给所有web主机拷贝index.html内容是各自的IP地址？      

Ansible可以利用Jinja2模板引擎读取变量，之前在playbook中调用变量，也是Jinja2的功能，Jinja2模块的表达式包含在分隔符"{{    }}"内。

> copy=直男 ；   
>
> template=渣男；

这里，我们给webserver主机拷贝首页，要求每个主机内容不同。

```shell
[root@control ansible]# mkdir ~/ansible/template

[root@control ansible]# vim ~/ansible/template/index.html
Welcome to {{ansible_hostname}} on {{ ansible_eth0.ipv4.address }}. 
#注意网卡名称根据实际情况填写，不可以完全照抄，不知道网卡名可以通过ip a s查询！
#{{ansible_hostname}}和{{ ansible_eth0.ipv4.address }}是ansible自动的facts变量。     
```



编写Playbook将网页模板文件拷贝到远程主机。

```shell
[root@control ansible]# vim ~/ansible/template.yml
---
- hosts: webserver
  tasks:
    - name: use template copy index.html to webserver.
      template:
        src: ~/ansible/template/index.html
        dest: /tmp/index.html
#hosts定义需要远程的目标主机是谁；tasks定义需要执行的任务是什么
#- name定义任务的描述信息；任务需要调用的模块是template模块
#template模块需要两个参数，src指定需要拷贝的源文件，dest指定需要拷贝的目标位置
#src: ~/ansible/template/index.html是上面创建的文件,文件中包含变量
#dest: /tmp/index.html拷贝到目标主机放在/tmp目录下
```



---------

## 高级语法

### error处理机制

默认ansible在遇到error会立刻停止playbook，使用ignore_errors可以忽略错误，继续后续的任务。   

如果一个剧本里面有20个任务，执行到第3个时失败，则不再往下执行。     

下面这个这个Playbook在执行时会意外中断。

```shell
[root@control ansible]# vim ~/ansible/error.yml
---
- hosts: test
  tasks:
    - name: start a service that does not exist.
      service:
        name: hehe         #注意：没有这个服务（启动一个不存在的服务）                                       
        state: started
    - name: touch a file.
      file:
        path: /tmp/service.txt
        state: touch
```

下面这个Playbook在执行时因为忽略了错误（针对某一个任务），不会被中断

```shell
[root@control ansible]# vim ~/ansible/error.yml
---
- hosts: test
  tasks:
    - name: start a service that does not exist.
      service:
        name: hehe
        state: started
      ignore_errors: true       #针对某一个任务忽略错误(ignore_errors是关键词)                          
    - name: touch a file.
      file:
        path: /tmp/service.txt
        state: touch
```

下面这个Playbook在执行时因为忽略了错误，不会被中断

```shell
[root@control ansible]# cat ~/ansible/error.yml
---
- hosts: test
  ignore_errors: true      #针对playbook全局忽略错误                             
  tasks:
    - name: start a service that does not exist.
      service:
        name: hehe
        state: started
    - name: touch a file.
      file:
        path: /tmp/service.txt
        state: touch
```

### handlers

可以通过handlers定义一组任务

仅当某个任务触发(notify)handlers时才执行相应的任务

如果有多个notify触发执行handlers任务，也仅执行一次

**仅当任务的执行状态为changed时handlers任务才执行**

handlers任务在所有其他任务都执行后才执行。

### when条件判断

when可以定义判断条件，条件为真时才执行某个任务

常见条件操作符有：==、!=、>、>=、<、<=

多个条件可以使用and(并且)或or（或者）分割

**when表达式中调用变量不要使用{{  }}**

下面编写Playbook，远程主机剩余内存不足700M则关闭NetworkManager服务

```shell
[root@control ansible]# vim ~/ansible/when_1.yml
---
- hosts: test
  tasks:
    - name: check memory size.
      service:
        name: NetworkManager
        state: stopped
      when: ansible_memfree_mb < 700
#被管理端主机剩余内存不足700M则关闭NetworkManager服务(也可以关闭别的不需要的服务)
#ansible_memfree_mb这个是ansible自带的facts变量,代表剩余内存的容量。
```

下面再编写一个Playbook，判断操作系统是RedHat8则创建测试文件。YAML的语法格式中>支持多行输入，但不保留换行符

```shell
[root@control ansible]# vim ~/ansible/when_2.yml
---
- hosts: test
  tasks:
    - name: touch a file
      file:
        path: /tmp/when.txt
        state: touch
      when:  >
        ansible_distribution == "RedHat"
                and
        ansible_distribution_major_version == "8"
#判断操作系统是RedHat8则创建测试文件
#YAML的语法格式中>支持多行输入，但不保留换行符（计算机会认为实际是一行内容）
#ansible_distribution和ansible_distribution_major_version都是自带的facts变量
#可以使用setup模块查看这些变量
```

### block任务块

如果我们需要当条件满足时执行N个任务,我们可以给N个任务后面都加when判断(但是很麻烦),

此时可以使用block定义一个任务块,当条件满足时执行整个任务块.

任务块就是把一组任务合并为一个任务组，使用block语句可以将多个任务合并为一个任务组

```shell
[root@control ansible]# vim ~/ansible/block_1.yml
---
- hosts: test
  tasks:
    - name: define a group of tasks.
      block:                                          #block是关键词，定义任务组
        - name: install httpd                       #任务组中的第一个任务
          yum:                                        #调用yum模块安装httpd软件包
            name: httpd
            state: present
        - name: start httpd                          #任务组中的第二个任务
          service:                                    #调用service模块启动httpd服务
            name: httpd
            state: started
      when: ansible_distribution == "RedHat"       #仅当条件满足再执行任务组
#注意:when和block是对齐的,他们在一个级别,当条件满足时要执行的是任务组（不是某一个任务）
#判断条件是看远程的目标主机使用的Linux发行版本是否是RedHat.
```

对于block任务块

我们可以使用rescue语句定义在block任务执行失败时要执行的其他任务

还可以使用always语句定义无论block任务是否成功，都要执行的任务。



下面编写一个包含rescue和always的示例

```shell
[root@control ansible]# vim ~/ansible/block_2.yml
---
- hosts: test
  tasks:
    - block:
        - name: touch a file test1.txt
          file:
            name: /tmp/test1.txt      #如果修改为/tmp/xyz/test1.txt就无法创建成功                        
            state: touch
      rescue:
        - name: touch a file test2.txt
          file:
            name: /tmp/test2.txt
            state: touch
      always:
        - name: touch a file test3.txt
          file:
            name: /tmp/test3.txt
            state: touch
#默认在/tmp/目录下创建test1.txt会成功，所以不执行rescue(创建test2.txt)
#如果我们把block中的任务改为创建/tmp/xyz/test1.txt（因为没有xyz目录所以会失败)
#当block默认任务失败时就执行rescue任务(创建test2.txt)
#但是不管block任务是否成功都会执行always任务(创建test3.txt)
```

### loop 循环

相同模块需要反复被执行的时候，使用loop循环可以避免重复代码。

编写playbook，循环创建目录

```shell
[root@control ansible]# vim ~/ansible/simple_loop.yml
---
- hosts: test
  tasks:
    - name: mkdir multi directory.
      file:
        path=/tmp/{{item}}       #注意，item是关键字，调用loop循环的值                                
        state=directory
      loop:                       #loop是关键词,定义循环的值,下面是具体的值
        - School
        - Legend
        - Life
#最终在/tmp目录下创建三个子目录.file模块被反复执行了三次。
#mkdir  /tmp/School;  mkdir  /tmp/Legend;   mkdir  /tmp/Life。
```

编写Playbook，循环创建用户并设置密码

```shell
[root@control ansible]# vim ~/ansible/complex_loop.yml
---
- hosts: test
  tasks:
    - name: create multi user.
      user:
        name: "{{item.iname}}"
        password: "{{item.ipass | password_hash('sha512')}}"
      loop:
        - { iname: 'term', ipass: '123456' }
        - { iname: 'amy' , ipass: '654321' }
#loop循环第一次调用user模块创建用户,user模块创建用户会读取loop里面的第一个值.
#loop第一个值里面有两个子值,iname和ipass
#创建用户item.iname就是loop第一个值里面的iname=term
#修改密码item.ipass就是loop第一个值里面的ipass=123456

#loop循环第二次调用user模块创建用户,user模块创建用户会读取loop里面的第二个值.
#loop第二个值里面有两个子值,iname和ipass
#创建用户item.iname就是loop第二个值里面的iname=amy
#修改密码item.ipass就是loop第二个值里面的ipass=654321
```

## ansible-vault 加密敏感数据

使用ansible-vault管理敏感数据

encrypt（加密）、decrypt（解密）、view（查看）

#### 加密敏感数据

```shell
[root@control ansible]# echo 123456 > data.txt               #新建测试文件
[root@control ansible]# ansible-vault encrypt data.txt      #加密文件
[root@control ansible]# cat data.txt

```

#### 查看加密文件（解开密码的）

```shell
[root@control ansible]# ansible-vault view data.txt         #查看加密文件
```

#### 修改密码

```shell
[root@control ansible]# ansible-vault rekey data.txt             #修改密码
Vault password: <旧密码>
New Vault password: <新密码>
Confirm New Vault password:<确认新密码>
```

#### 解密文件

```shell
[root@control ansible]# ansible-vault decrypt data.txt      #解密文件
[root@control ansible]# cat data.tx
```

#### 使用密码文件

加密、解密每次都输入密码很麻烦，可以将密码写入文件

```shell
[root@control ansible]# echo "I'm secret data" > data.txt       #需要加密的敏感数据
[root@control ansible]# echo 123456 > pass.txt                   #加密的密码
[root@control ansible]# ansible-vault  encrypt --vault-id=pass.txt  data.txt 
[root@control ansible]# cat data.txt
[root@control ansible]# ansible-vault decrypt --vault-id=pass.txt data.txt
[root@control ansible]# cat data.txt
```

