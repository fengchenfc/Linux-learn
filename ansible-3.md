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

