# RHCE

## 环境说明：

- 真机（RHEL8.0）：server1.net0.example.com 172.25.0.254/24

预设 root 口令为 tedu

提供 RHEL8 软件源 http://server1.net0.example.com/rhel8/BaseOS

提供 RHEL8 软件源 http://server1.net0.example.com/rhel8/AppStream

提供 DNS 服务，为区域 net0.example.com 中相关站点提供解析

提供 NTP 网络时间服务；提供 NFS 文件服务，共享/rhome/ldapuser0 目录

- 虚拟机 control（RHEL8.0） —— 控制机：

control.net0.example.com 172.25.254.100/24

预设授权 sudo 用户 alice，密码为 Asimov

预设 IP 地址 172.25.254.100/24，已做好主机名映射

- 虚拟机 node1~node5（RHEL8.0） —— 受管机：

node[1-5].net0.example.com 172.25.254.10[1-5]/24

预设授权 sudo 用户 alice，密码为 Asimov

预设 IP 地址 172.25.254.101~105/24，已做好主机名映射

- 其他信息：

除非另有指定，否则所有工作都需要归属在控制机的 /home/alice/ansible/ 目录下





## 01.安装和配置  ansible  环境
1）安装所需软件包
2）在/home/alice/ansible/inventory 文件中填写主机清单,要求:
node1 属于 test01 主机组
node2 属于 test02 主机组
node3 和 node4 属于 web 主机组
node5 属于 test05 主机组
web 组属于 webtest 主机组
3)  在/home/alice/ansible 目录中创建 ansible.cfg,满足以下需求:
主机清单文件为/home/alice/ansible/inventory
playbook 中角色位置/home/alice/ansible/roles

```bash
[root@server1 ~]# ssh alice@control
#备注：control主机的yum源是提前配置OK的

[alice@control ~]$ pwd                           #确认自己的位置
/home/alice
[alice@control ~]$ sudo grep alice /etc/sudoers
alice	ALL=(ALL) 	NOPASSWD:ALL
#用户名  所有主机（root） 不需要密码:所有命令 

[alice@control ~]$ sudo yum -y install ansible      #安装软件包

[alice@control ~]$ mkdir -p /home/alice/ansible/roles

[alice@control ~]$ cd /home/alice/ansible/
#写主配置文件，要参考/etc/ansible/ansible.cfg
[alice@control ansible]$ vim ansible.cfg           #创建主配置文件
[defaults]
inventory=inventory		#主机清单配置文件
remote_user=alice		#以什么用户远程被管理主机
roles_path=roles		#角色存放路径
[privilege_escalation]	#特权升级
become=True				#alice没有特权，是否需要提升权限
become_method=sudo		#怎么用普通用户执行一些管理员的命令
become_user=root		#以什么用户的权限去执行
become_ask_pass=False	#执行sudo命令提权时是否需要输入密码

[alice@control ansible]$ vim inventory       #创建主机清单文件
[test01]		#定义主机组
node1			#定义组中的具体主机
[test02]
node2
[web]
node3
node4
[test05]
node5
[webtest:children]	 #嵌套组（children为关键字）
web					#嵌套组可以在组中包含其他组
[alice@control ansible]$ ansible all -m ping        #测试一下环境
[alice@control ansible]# ansible  all  --list-hosts	#查看所有主机列表
```



## 02.创建和运行  Ansible  临时命令
1）编写脚本 /home/alice/ansible/adhoc.sh，用来为所有受管机配置 2 个 yum 仓库。

- 仓库 1：
  名称为 BASE，描述为 software base
  URL 为 http://study.lab0.example.com/rhel8/BaseOS
  GPG 签名启用，GPG 秘钥 URL 为 http://study.lab0.example.com/rhel8/RPM-GPG-KEY-redhat-release
  仓库为启用状态

- 仓库 2:
  名称为 STREAM，描述为 software stream
  URL 为 http://study.lab0.example.com/rhel8/AppStream
  GPG 签名启用，GPG 秘钥 URL 为 http://study.lab0.example.com/rhel8/RPM-GPG-KEY-redhat-release
  仓库为启用状态

```bash
[alice@control ansible]$ ansible-doc -l | grep yum
yum                                                    Manages packages with the `yum' package manager  
yum_repository                                         Add or remove YUM repositories 

#参考ansible-doc yum_repository【EXAMPLES】
[alice@control ansible]$ vim adhoc.sh
#!/bin/bash
ansible all -m yum_repository -a 'name=BASE description="software base" baseurl="http://study.lab0.example.com/rhel8/BaseOS" gpgcheck=yes gpgkey="http://study.lab0.example.com/rhel8/RPM-GPG-KEY-redhat-release" enabled=yes'
ansible all -m yum_repository -a 'name=STREAM description="software stream" baseurl="http://study.lab0.example.com/rhel8/AppStream" gpgcheck=yes gpgkey="http://study.lab0.example.com/rhel8/RPM-GPG-KEY-redhat-release" enabled=yes'

#name参数：必须参数，用于指定要操作的唯一的仓库ID，也就是”.repo”配置文件中每个仓库对应的”中括号”内的源标识。
#description参数：此参数用于设置仓库的注释信息，也就是”.repo”配置文件中每个仓库对应的”name字段”对应的内容
[alice@control ansible]$ chmod +x adhoc.sh
[alice@control ansible]$ ./adhoc.sh
#测试：可以去node1-node5任意安装一个软件包测试

#yum_repository常用参数解析
#name参数：必须参数，用于指定要操作的唯一的仓库ID，也就是”.repo”配置文件中每个仓库对应的”中括号”内的源标识。
#description参数：此参数用于设置仓库的注释信息，也就是”.repo”配置文件中每个仓库对应的”name字段”对应的内容
#file参数：此参数用于设置仓库的配置文件名称,也就是.repo的前缀
#enabled参数：此参数用于设置是否激活对应的 yum 源
#gpgcheck参数：此参数用于设置是否开启 rpm 包验证功能
#gpgkey参数：当 gpgcheck 参数设置为 yes 时，需要使用此参数指定验证包所需的公钥。
#state参数：默认值为 present，当值设置为 absent 时，表示删除对应的 yum 源。
```



## 03.编写剧本远程安装软件
创建名为/home/alice/ansible/tools.yml 的 playbook，能够实现以下目的：
1）将 php 和 tftp 软件包安装到 test01、test02 和 web 主机组中的主机上
2）将 RPM Development Tools 软件包组安装到 test01 主机组中的主机上
3）将 test01 主机组中的主机上所有软件包升级到最新版本

```bash
#参考ansible-doc yum【EXAMPLES】
[alice@control ansible]$ vi tools.yml
---
- hosts: test01,test02,web 
  tasks:
    - name: 将 php 和 tftp 软件包安装到 test01、test02 和 web 主机组中的主机上
      yum:
        name:
          - php
          - tftp
        state: present
- hosts: test01
  tasks:
    - name: 将 RPM Development Tools 软件包组安装到 test01 主机组中的主机上
      yum:
        name: "@RPM Development Tools"
        state: present
    - name: 将 test01 主机组中的主机上所有软件包升级到最新版本
      yum:
        name: '*'
        state: latest			#最新的

[alice@control ansible]$ ansible-playbook tools.yml
#验证：ssh远程node1
# rpm -qa |grep php
# yum grouplist   看有没有installed group的软件组包
```





## 04.安装并使用系统角色
安装 RHEL 角色软件包，并创建剧本 /home/alice/ansible/timesync.yml，满足以下要求：
1）在所有受管理节点运行
2）使用 timesync 角色
3）配置该角色,使用时间服务器 172.25.254.250,并启用 iburst 参数

```bash
[alice@control ansible]$ sudo yum list |grep roles
#软件包记不住，可以grep过滤
[alice@control ansible]$ sudo yum -y install rhel-system-roles
#安装软件包（题目有要求）
[alice@control ansible]$ sudo rpm -ql rhel-system-roles
#查看软件安装到哪里
[alice@control roles]$ cp -r /usr/share/ansible/roles/rhel-system-roles.timesync/  /home/alice/ansible/roles/
#根据考试题目第一题，要求：所有角色必须放在/home/alice/ansible/roles目录

#使用角色之前需要看帮助（README.md,找Example Playbook)
[alice@control ansible]$ vim roles/rhel-system-roles.timesync/README.md 

[alice@control ansible]# vi timesync.yml     #写剧本做题
---
- hosts: all
  vars:						#设置 NTD 服务器变量
    timesync_ntp_servers:
      - hostname: 172.25.254.250
        iburst: yes
  roles:		#调用角色
    - rhel-system-roles.timesync	
#验证：ssh远程node1，查看chrony配置文件
[alice@node1 ~]# head /etc/chrony.conf
```



## 05.通过 galaxy  安装角色
创建剧本 /home/alice/ansible/roles/down.yml，用来从以下 URL 下载角色，并安装到
/home/alice/ansible/roles 目录下：
http://study.lab0.example.com/roles/haproxy.tar 此角色名为 haproxy
http://study.lab0.example.com/roles/myphp.tar 此角色名为 myphp

```bash
[alice@control ansible]$ vi roles/down.yml
- src: http://study.lab0.example.com/roles/haproxy.tar
  name: haproxy
- src: http://study.lab0.example.com/roles/myphp.tar
  name: myphp
[alice@control ansible]$ ansible-galaxy install -r roles/down.yml
#ansible-galaxy默认是去官网下载角色，通过-r指定上面一步创建的文件名，到特定的链接下载role
#验证：ls roles
#多了两个角色（haproxy，myphp）
```



## 06.创建及使用自定义角色
根据下列要求,在/home/alice/ansible/roles 中创建名为 httpd 的角色:
1）安装 httpd 软件，并能够开机自动运行
2）开启防火墙，并允许 httpd 通过
3）使用模板 index.html.j2，用来创建/var/www/html/index.html 网页，内容如下（其中，HOSTNAME 是受管理节点的完全域名，IPADDRESS 是 IP 地址）：
Welcome to HOSTNAME on IPADDRESS
然后创建剧本 /home/alice/ansible/myrole.yml，为 webtest 主机组启用 httpd 角色。

```bash
[alice@control ansible]$ ansible-galaxy init roles/httpd
#创建一个名称为httpd的角色，在roles目录下创建
#验证：[alice@control ansible]$ ls roles/httpd/

#参考ansible-doc yum
ansible-doc service
ansible-doc firewalld
[alice@control ansible]$ vim roles/httpd/tasks/main.yml   //编写任务
- name: install httpd
  yum:
    name: httpd
    state: present
#调用的模块是yum，安装的软件名称是httpd，present代表安装
- name: running httpd
  service:
    name: httpd
    state: started
    enabled: yes
#调用的模块是service，启动的服务名称是httpd，started代表启动服务
#enabled: yes代表设置开机自启
- name: running firewalld
  service:
    name: firewalld
    state: started
    enabled: yes
- firewalld:
    service: http
    permanent: yes
    immediate: yes
    state: enabled
#调用的模块是firewalld，state:enabled是添加防火墙规则
#运行http服务的访问（放行，不要拦截）
#permanent:yes代表永久规则，immediate: yes代表立刻生效
- template:
    src: index.html.j2
    dest: /var/www/html/index.html
#调用template模块，将index.html.j2文件拷贝到被管理主机
#放到被管理主机的/var/www/html/index.html

#参考：ansible node3 -m setup | less（找facts变量）
[alice@control ansible]$ vi roles/httpd/templates/index.html.j2
Welcome to {{ansible_facts.fqdn}} on {{ansible_facts.eth0.ipv4.address}}
#IP这块一定不要用最开始那个ansible_all..........那个，考试容易出错

[alice@control ansible]$ vi myrole.yml       #编写剧本调用role角色
---
- hosts: webtest
  roles:
    - httpd
[alice@control ansible]$ ansible-playbook myrole.yml
#测试：
[alice@control ansible]$ curl 172.25.254.103
[alice@control ansible]$ curl 172.25.254.104
```





## 07.使用 之前 通过 galaxy  下载的角色
创建剧本 /home/alice/ansible/web.yml，满足下列需求：
1）该剧本中包含一个 play，可以在 test05 主机组运行 haproxy 角色（此角色已经配置
好网站的负载均衡服务）
2）多次访问 http://node5.net0.example.com 可以输出不同主机的欢迎页面
3）该剧本中包含另一个 play，可以在 webtest 主机组运行 myphp 角色（此角色已经配置
好网站的 php 页面）
4）多次访问 http://node5.net0.example.com/index.php 也输出不同主机的欢迎页面

```bash
#角色是前面第五题已经下载好的haproxy和myphp
#参考cat roles/httpd/tasks/main.yml（参考之前写的防火墙规则）

[alice@control ansible]$ vim web.yml 
---
- hosts: webtest
  roles:
    - myphp
#远程webtest主机(node3,node4)，调用myphp角色，自动配置php动态网页
- hosts: test05
  roles:
    - haproxy
#远程test05(node5)，调用haproxy角色，自动配置负载均衡调度器
  tasks:
    - name: running firewalld
      service:
        name: firewalld
        state: started
        enabled: yes
    - firewalld:
        service: http
        permanent: yes
        immediate: yes
        state: enabled
#调用firewalld设置防火墙规则（如第6题）
[alice@control ansible]$ ansible-playbook web.yml    #执行剧本
#验证：
[alice@control ansible]$ curl http://node5                   #访问静态
[alice@control ansible]$ curl http://node5/index.php  		#访问动态
```



## 08.编写剧本远程管理逻辑卷
创建剧本 /home/alice/ansible/lvm.yml，用来为所有受管机完成以下部署：
1）在卷组 search 中创建名为 mylv 的逻辑卷，大小为 1000MiB
2）使用 ext4 文件系统格式化该逻辑卷
3）如果无法创建要求的大小，应显示错误信息 insufficient free space，并改为 500MiB
4）如果卷组 search 不存在，应显示错误信息 VG not found
5）不需要挂载逻辑卷

```bash
#参考：ansble node1 -m setup | less
#ansible-doc  lvol （逻辑卷） 
[alice@control ansible]$ vi lvm.yml
---
- hosts: all
  tasks:
    - name: display a error info
      debug:
        msg: "VG not found"		#报错信息
      when: "'search' not in ansible_lvm.vgs"	
      failed_when: "'search' not in ansible_lvm.vgs"	   
#调用debug模块，显示报错信息，msg：后面是报错信息的内容
#debug模块可以显示变量的值，可以辅助排错，通过msg可以显示变量的值
#when定义什么时候执行debug模块，条件是在ansible_lvm.vgs变量里面找不到search卷组
#failed_when定义什么情况是剧本失败，条件是在ansible_lvm.vgs变量里面找不到search卷组
#找不到search这个卷组，就立刻停止执行剧本，不在往下执行后面的任务
#当’ failed_when’关键字对应的条件成立时，’ failed_when’会将对应的任务的执行状态设置为失败，以停止playbook的运行
    - block:				#先通过block尝试创建1000M的逻辑卷
        - lvol:				#使用的是lvol模块，vg指定使用哪个卷组创建逻辑卷
            vg: search		#使用的是search这个卷组
            lv: mylv		#lv指定创建的卷组叫什么名字，这里名称为mylv
            size: 1000		#size指定逻辑卷大小（1000m或者500m）
      rescue:				#如果前面block创建1000m无法成功，那么才执行rescue后面的任务
        - debug:
            msg: "insufficient free space"
        - lvol:			#调用相同的lvol模块，创建500m的逻辑卷
            vg: search
            lv: mylv
            size: 500
      always:				#最后不管创建多大的逻辑卷，都要格式化
        - filesystem:		#使用的是filesytem模块，格式化的格式是ext4，dev后面写对谁格式化
            fstype: ext4		
            dev: /dev/search/mylv
            force: yes
# handlers:
#   - filesystem:		#使用的是filesytem模块，格式化的格式是ext4，dev后面写对谁格式化
#       fstype: ext4		
#       dev: /dev/search/mylv
#       force=yes
[alice@control ansible]$ ansible-playbook /home/alice/ansible/lvm.yml
[alice@control ansible]$ ansible all -a 'lvs'
```



## 09.根据模板部署主机文件
1）从 http://study.lab0.example.com/materials/newhosts.j2 下载模板文件
2）完成该模板，用来生成新主机清单（主机的显示顺序没有要求），结构如下
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
172.25.254.101 node1.lab0.example.com node1
172.25.254.102 node2.lab0.example.com node2
172.25.254.103 node3.lab0.example.com node3
172.25.254.104 node4.lab0.example.com node4
172.25.254.105 node5.lab0.example.com node5
3）创建剧本 /home/alice/ansible/newhosts.yml，它将使用上述模板在 test01 主机组的主
机上生成文件/etc/newhosts

```bash
#参考：ansible node1 -m setup |less （查变量名）
#下载文件
#修改文件（变量)
#写剧本，拷贝文件到所有主机（template）
[alice@control ansible]$ sudo yum -y install wget 
#装包
[alice@control ansible]$ wget http://study.lab0.example.com/materials/newhosts.j2
#下载考题给的文件
[alice@control ansible]$ cat newhosts.j2
#看一看
[alice@control ansible]$ vim newhosts.j2        #修改文件方法一
#直接复制考题的内容，到文件中
[alice@control ansible]$ vim newhosts.j2         #修改文件方法二
#在源文件下面，手写如下内容
{% for i in groups.all %}    
{{hostvars[i].ansible_eth0.ipv4.address}}  {{hostvars[i].ansible_fqdn}}  {{hostvars[i].ansible_hostname}} 
{% endfor %} 
#groups是ansible自带的变量（魔法变量）被管理的组，all是所有主机
#groups.all代表，被管理组中的所有主机
#hostvars是ansible自带的变量（魔法变量），他可以提取某个主机的变量信息，如hostvars[node1]

[alice@control ansible]$ vi newhosts.yml     #编写剧本拷贝上面准备的文件
---
- hosts: all
- hosts: test01
  tasks:
    - template:
        src: newhosts.j2
        dest: /etc/newhosts
#注意事项：先远程所有主机，获取所有主机的变量
#然后再远程test01,拷贝模版文件newhosts.j2到对方主机
#模版文件里通过for循环逐一写入所有主机的IP，域名和主机名

[alice@control ansible]$ ansible-playbook /home/alice/ansible/newhosts.yml

[alice@node1 ~]$ cat /etc/newhosts
```



## 10.编写剧本修改远程文件内容
创建剧本 /home/alice/ansible/newissue.yml，满足下列要求：
1）在所有清单主机上运行，替换/etc/issue 的内容
2）对于 test01 主机组中的主机，/etc/issue 文件内容为 test01
3）对于 test02 主机组中的主机，/etc/issue 文件内容为 test02
4）对于 web 主机组中的主机，/etc/issue 文件内容为 Webserver

```bash
[alice@control ansible]$ vim inventory    #修改主机清单文件，添加如下内容：
[test01:vars]
xx=test01
[test02:vars]
xx=test02
[web:vars]
xx=Webserver
 #给test01组创建变量,变量名称为xx（名称任意），变量的值为test01
 #其他所有组类似
 #参考：ansible-doc copy
[alice@control ansible]$ vi  newissue.yml        #编写剧本
---
- hosts: all
  tasks:
    - copy:
        content: "{{xx}}"
        dest: /etc/issue
[alice@control ansible]$ ansible-playbook newissue.yml
 #执行剧本
 #验证：
[alice@control ansible]$ ansible all -a "cat /etc/issue"

[alice@control ansible]$ vim newissue.yml
- name: deploy /etc/issue
  hosts: all
  tasks:
   - copy:
     content: 
       {% if "test01" in group_names %} #如果所在组包括 dev
       test01
       {% elif "test02" in group_names %} #如果所在组包括 test
       test02
       {% elif "web" in group_names %} #如果所在组包括 prod
       Webserver
       {% endif %}
       dest: /etc/issue #复制到指定目标文件
[alice@control ansible]$ ansible-playbook newissue.yml
```



## 11.编写剧本部署远程  Web  目录
创建剧本 /home/alice/ansible/webdev.yml，满足下列要求：
1）在 test01 主机组运行
2）创建目录/webdev，属于 webdev 组，常规权限为 rwxrwxr-x，具有 SetGID 特殊权限
3）使用符号链接/var/www/html/webdev 链接到/webdev 目录
4）创建文件/webdev/index.html，内容是 It's works!
5）查看 test01 主机组的 web 页面 http://node1/webdev/ 将显示 It's works!

```bash
#参考：ansible-doc group
#ansible-doc file
[alice@control ansible]$ vi webdev.yml    #编写剧本
---
- hosts: test01
  tasks:
    - group:
        name: webdev
        state: present
#调用group组，创建webdev
    - file:
        path: /webdev
        group: webdev
        mode: '2775'
        state: directory
#调用file模块创建目录，设置权限，所属组
    - file:
        src: /webdev
        dest: /var/www/html/webdev
        state: link
#调用file模块，src后面是源文件或目录，dest后面是软链接
#state的值是link，表示创建软链接
    - copy:
          content: "It's works!"
          dest: /webdev/index.html
#调用copy模块，创建一个index.html文件，内容是：It's works!
    - shell: setenforce 0
#调用shell模块执行setenforce 0关闭selinux
#参考前面题目写的防火墙规则，复制过来
#参考 cat roles/httpd/tasks/main.yml
    - name: run firewalld
      service:
        name: firewalld
        state: started
        enabled: yes
    - firewalld:
            service: http
            permanent: yes
            immediate: yes
            state: enabled
    - service:
            name: httpd
            state: started
            enabled: yes

#确保服务是启动
[alice@control ansible]$ ansible-playbook webdev.yml
#执行剧本
#测试：在真机使用firefox http://node1/webdev
```



## 12.编写剧本为受管机生成硬件报告
创建名为/home/alice/ansible/hardware.yml 的 playbook，满足下列要求：
1）使所有受管理节点从以下 URL 下载文件：
http://study.lab0.example.com/materials/hardware.empty
2）并用来生成以下硬件报告信息，存储在各自的/root/hardware.txt 文件中
清单主机名称
以 MB 表示的总内存大小
BIOS 版本
硬盘 vda 的大小
硬盘 vdb 的大小
其中，文件的每一行含有一个 key=value 对，如果项目不存在，则显示 NONE。

```bash
#使用真机浏览器访问一下http://study.lab0.example.com/materials/hardware.empty  
#看看大致的内容
#原文件内容如下：
hostname=inventoryhostname
mem=memory_in_MB
bios=BIOS_version
vdasize=disk_vda_size
vdbsize=disk_vdb_siz
#参考：ansible-doc get_url
#ansible-doc replace
#ansible  node1  -m setup  | less
[alice@control ansible]$ vim  hardware.yml    #编写剧本
---
- hosts: all
  ignore_errors: yes
  tasks:
    - get_url:
        url: http://study.lab0.example.com/materials/hardware.empty
        dest: /root/hardware.txt
    - replace:
        path: /root/hardware.txt
        regexp: inventoryhostname
        replace: "{{ansible_hostname}}"
    - replace:
        path: /root/hardware.txt
        regexp: memory_in_MB
        replace: "{{ansible_memtotal_mb}}"
    - replace:
        path: /root/hardware.txt
        regexp: BIOS_version
        replace: "{{ansible_bios_version}}"
    - replace:
        path: /root/hardware.txt
        regexp: disk_vda_size
        replace: "{{ansible_devices.vda.size}}"
    - replace:
        path: /root/hardware.txt
        regexp: disk_vdb_size
        replace: "{{ansible_devices.vdb.size if ansible_devices.vdb.size is defined else 'NONE'}}"
#通过get_url下载题目指定的文件
#再通过replace修改文件内容
#replace里面的path指定需要修改的文件名，regexp指定原来的旧内容（需要被替换的内容）
#replace里面的replace后面是需要提出为的新内容{新内容都是变量}
#验证：
#远程node1-node5，逐一查看/root/hardware.txt文件的内容
[alice@control ansible]$ ansible-playbook /home/alice/ansible/hardware.yml
[alice@control ansible]$ ansible all -a "cat /root/hardware.txt"
```



## 13.创建保险库文件
1）创建 ansible 保险库 /home/alice/ansible/passdb.yml，其中有 2 个变量：
pw_dev，值为 ab1234
pw_man，值为 cd5678
2) 加密和解密该库的密码是 pwd@1234 ,密码存在/home/alice/ansible/secret.txt 中

```bash
[alice@control ansible]$ vi passdb.yml
---
pw_dev: ab1234
pw_man: cd5678
#设置变量名和变量的值，：后面有空格
[alice@control ansible]$ vim secret.txt        #加密上面passdb.yml文件的密码
pwd@1234
#参考：ansible-vault --help
[alice@control ansible]$ ansible-vault encrypt --vault-id secret.txt passdb.yml
#ansible-vault是密码用的命令，encrypt代表加密，decrypt代表解密
#--vault-id后面跟的是密码文件，最后是对passdb.yml文件加密
```



## 14.编写剧本为受管机批量创建用户，要求使用保险库中的密码
从以下 URL 下载用户列表，保存到/home/alice/ansible 目录下：
http://study.lab0.example.com/materials/name_list.yml
创建剧本 /home/alice/ansible/users.yml 的 playbook，满足下列要求:
1）使用之前题目中的 passdb.yml 保险库文件
2）职位描述为 dev 的用户应在 test01、test02 主机组的受管机上创建，从 pw_dev 变量
分配密码，是补充组 devops 的成员
3）职位描述为 man 的用户应在 web 主机组的受管机上创建,从 pw_man 变量分配密码,是
补充组 opsmgr 的成员
4）该 playbook 可以使用之前题目创建的 secret.txt 密码文件运行

```bash
[alice@control ansible]$ wget http://study.lab0.example.com/materials/name_list.yml
[alice@control ansible]$ cat name_list.yml 
users:
  - name: tom
    job: dev
  - name: jerry
    job: man
#参考：ansible-doc user
#方法一：
[alice@control ansible]$ vi users.yml
---
- hosts: test01,test02
  vars_files:
    - name_list.yml
    - passdb.yml
  tasks:
    - group: name=devops
    - user: 
        name: "{{item.name}}" 
        groups: devops 
        password: "{{pw_dev | password_hash('sha512')}}"
      loop: "{{users}}"
      when: item.job == 'dev'
- hosts: web
  vars_files:
    - name_list.yml
    - passdb.yml
  tasks:
    - group: name=opsmgr
    - user: 
        name: "{{item.name}}" 
        groups: opsmgr
        password: "{{pw_man | password_hash('sha512')}}"
      loop: "{{users}}"
      when: item.job == 'man'
#编写一个剧目，远程test01和test02，创建devops组，创建用户tom，设置密码
#首先通过vars_files读取两个文件中的变量（name_list.yml和passdb.yml)
#使用group模块给test01,test02组创建固定的组，名称为devops
#使用user模块创建系统账户，name后面跟账户的名称，账户名称使用item调用loop循环
#item是关键词不可修改，下面loop对变量{{users}}循环，
#users中包含两组数据，一组是tom的数据，一组是jerry的数据
#本来user模块通过loop循环可以创建两个账户，因为user中包含两组数据
#但是，下面又通过when语句做条件判断，仅当item.job的值=='关键词'时才创建用户
#对于test01和test02组的主机，就可以判断仅当item.job的值为dev时才创建账户
#创建的账户名称为item.name，也就是users.name（系统账户名称）
#item.job也就是users.job（职位的描述）
#账户对应的密码指定读取passdb.yml中定义的变量pw_dev和pw_man
#第一个剧本编写好后，使用数字+yy复制，p粘贴，编写第二个剧目
#适当修改第二个剧目的内容即可

[alice@control ansible]$ ansible-playbook users.yml --vault-id=/home/alice/ansible/secret.txt 
```



## 15.重设保险库密码
1）从以下 URL 下载保险库文件到/home/alice/ansible 目录：
http://study.lab0.example.com/materials/topsec.yml
2）当前的库密码是 banana，新密码是 big_banana，请更新该库密码

```bash
[alice@control ansible]$ wget http://study.lab0.example.com/materials/topsec.yml

[alice@control ansible]$ ansible-vault rekey topsec.yml 
Vault password: banana
New Vault password: big_banana
Confirm New Vault password: big_banana
Rekey successful 
```



![在这里插入图片描述](https://img-blog.csdnimg.cn/20200724165835583.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDEzNjQ0Ng==,size_16,color_FFFFFF,t_70)

