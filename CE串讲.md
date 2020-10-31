# CE考试

考试环境虚拟机同ansible： control+node1~5

所有机器IP 之类的已经配好 不用动

收卷后老师会把node1~5还原

可以自己提前还原，测试做题结果



## 1

1-1

控制机上，yum 已经配好

用 sudo yum -y install ansible



1-2

mkdir  ....../ansible

cd  ...../ansible	下	后面的运行的剧本啥的也应该在ansible目录下



1-3

配完了之后用ansile  all --list-hosts 检查下

有时间的话一个一个试试

还有就是查查能不能控制   **allsible all  -m ping**    这条看到底是不是配错了

记得创建 roles 目录 提前

---------

## 2

勤快使用ansible-doc  <u>xxxx</u>    里面参数向

 =  必有参数  

\- 非必要



检查 ansible all -a "yum -y install vim"

或 ansible all -a "yum repolist"



## 3

3-2 

host：

tasks：

​	-yum：

​        name： @RPM Development tools   

**记住 这个@符号，装包组用的！还有装包的包组名称带RPM，别省略**



--------

## 4  再看看 再问问

yum list "\*roles*"        查角色包 叫啥  如果忘了的话

rpm  -ql  

ansible-galaxy list    查看当前目录有多少角色（能用的） 检查是否完成安装

4-2 & 4-3

打开 角色目录下的 README  看看 这个角色干嘛的   可以打开后搜索关键词example

set nu 显示行号

复制列子贴到timesync.yml（74-84 行）   在vim中直接输入 73,85w timesync.yml 可以直接复制到文件



## 5





## 6

   思路

在main.yml  中

\-  yum : name = install httpd 

\-  template:				后面最好加个force=yes 

\-  service:

\-  firewalld: 





！！！  

在 ~/.vimrc   创建个VIM文本，就这样的 然后以后按TAB 就直接是2 个空格了

set ai 

set ts=2	



## 7

7-4 

意思就是test05 是proxy  

webtest 是两台后端服务器

第2题和第六题 都 配好环境了 基础的



思路

vim web.yml

-name : myphp

hosts:webtest

roles:

  -myphp

-name: haproxy

hosts:test05

rules:

-haproxy





## 9 







## 10

所有清单主机：  hosts： all

本体需要应用到if判断

任务： template 或copy    （template更方便更简单）

-name:xxx

 hosts:all

 tasks:

-te:src=i

ssue.j2 (自己弄)   dest









---------

[alice@control ansible]$ cat newissue.yml
\- name: xxx
  hosts: all
  tasks:
    \#- template: src=issue.j2 dest=/etc/issue force=yes
    \- copy:
        content: |
          {% if 'test01' in  group_names %}
          test01
          {% elif 'test02' in  group_names  %}
          test02
          {% elif 'web' in group_names  %}
          Webserver
          {% endif %}
        dest: /etc/issue
        force: yes

------------

[alice@control ansible]$ cat webdev.yml
\- name: deploy webdir
  hosts: test01
  tasks:
    \- yum: name=httpd state=present
    \- yum: name=policycoreutils-python-utils state=present
    \- group: name=webdev state=present
    \- file: path=/webdev group=webdev state=directory mode="2775" setype=httpd_sys_content_t
    \- file: path=/var/www/html/webdev src=/webdev state=link force=yes  setype=httpd_sys_content_t
    \- copy: content="It's works!" dest=/webdev/index.html  setype=httpd_sys_content_t force=yes
    \- service: name=httpd state=restarted enabled=yes
    \- firewalld: service=http state=enabled permanent=yes immediate=yes

------------

