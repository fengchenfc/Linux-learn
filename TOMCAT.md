## 概述

目前主流的网站，

使用Java语言写的网站，相匹配的服务为tomcat

使用php语言写的网站，相匹配的服务为nginx

#### java

- 一种跨平台、面向对象的程序设计语言
- java技术具有卓越的
  - 通用性
  - 高效性
  - **平台移植性**
  - 安全性

- java体系
  - java SE（标准版）
  - java EE（企业版）

#### JDK简介

JDK（Java Development Kit）是Sun针对Java开发者推出的

**Java语言的软件开发工具包**

JDK是整个Java的核心

- 包括了Java运行环境
- Java工具（如编译、排错、打包等工具）
- Java基础的类库



JRE（Java Runtime Environment，Java运行环境）

**JRE是JDK的子集**

JRE包括

- Java虚拟机（jvm）
- Java核心类库和支持文件
- 不包含开发工具（JDK）--编译器、调试器和其他工具

-----------

## tomcat的安装

<u>**tomcat的运行需要依赖Java，需先安装jdk工具包**</u>

配置Java环境并拷贝tomcat程序到指定目录（该目录可自定义）

```shell
yum -y install java-1.8.0-openjdk    //首先安装java环境软件包
cd  ~/lnmp_soft
tar -xf apache-tomcat-8.0.30.tar.gz  //在lnmp_soft目录下释放tomcat软件包
cp -r apache-tomcat-8.0.30 /usr/local/tomcat
cd /usr/local/tomcat/   
```

------

## tomcat主配置文件分析

#### tomcat目录介绍

bin	//存放tomcat主要程序

logs	//存放日志

conf	//存放配置文件

work	//存放动态网站（页面）被解析时的临时文件

webapps	//存放网站页面，类nginx的html

#### 服务的启停

bin/shutdown.sh		//关闭tomcat服务

bin/startup		//开启tomcat服务

若想重启服务，先关闭在开启。



#### 服务的状态查看

运用ss/netstat,此处不能检索tomcat，而应该检索java

```shell
[root@web1 tomcat]# bin/startup.sh 	//开启服务

Using CATALINA_BASE:   /usr/local/tomcat
Using CATALINA_HOME:   /usr/local/tomcat
Using CATALINA_TMPDIR: /usr/local/tomcat/temp
Using JRE_HOME:        /usr
Using CLASSPATH:       /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar
Tomcat started.

[root@web1 tomcat]# ss -nutlp | grep java   //tomcat状态查看
tcp    LISTEN     0      1      ::ffff:127.0.0.1:8005                 :::*                   users:(("java",pid=1491,fd=67))
tcp    LISTEN     0      100      :::8009                 :::*                   users:(("java",pid=1491,fd=51))
tcp    LISTEN     0      100      :::8080                 :::*                   users:(("java",pid=1491,fd=46))
```

可以看到，有三个端口

- ：8005  

  //跟软件的安全和关闭有关

- ：8080 

  //访问页面的端口（因为tomcat是阿帕奇发明的，httpd使用80端口，所以tomcat默认网页端口是8080，避免与自家服务冲突）

- ：8009

  //tomcat工程师作内部测试用的，对于我们无用，但是此端口异常时，tomcat也无法正常工作

  

<u>在开启tomcat服务后，8005端口不一定可以打开。</u>

<u>开启需要依赖足够的随机数，否则在开启tomcat服务后，8005端口无法打开。</u>

> dev/random		//此文件记录服务器中产生的随机数，数量不够多的话，开启tomcat服务之后此端口无法打开
>
> dev/urandom	//此文件会疯狂的产生随机数

#### 解决8005端口无法打开的方法

解决的的方法有三种：

1. **（不推荐）**随便在Linux系统操作操作（cd/ls/mv/cp/等待），时间不定；
2. 制作伪random文件，软连接unrandom

```shell
# mv /dev/random /dev/random.bak	//备份/dev/random文件
# ln -s /dev/urandom /dev/random	//制作uradom的软连接random
# ls  /dev/random		
/dev/random

yum -y install psmisc	//安装killall工具
killall java	//杀死服务
ss -nutlp | grep java	//查看当前状态
[root@web1 tomcat]# bin/startup.sh 	//开启服务
ss -nutlp | grep java	//再次查看8005端口是否开启
```

3. 安装rng-tools，此工具用来产生随机数

```shell
yum -y install rng-tools	
systemctl start rngd  //开启服务
```

然后杀死java服务，重新开启tomcat，检查状态查询是否开启8005端口。



## 测试服务器

在浏览器输入

http://192.168.2.100:8080

查看测试页面是否正确显示，显示则服务正常



## 创建测试页

tomcat默认网站页面存放位置：

/usr/local/tomcat/webapps/ROOT/test.jsp

```shell
[root@web1 ~]# vim  /usr/local/tomcat/webapps/ROOT/test.jsp
<html>
<body>
<center>
Now time is: <%=new java.util.Date()%>            //显示服务器当前时间
</center>
</body>
</html>
```



## tomcat虚拟主机

回顾http创建虚拟主机：写多个VirtualHost

```shell
<VirtualHost *:80>
	Servername www.a.com
	Documentroot /var/www/html
</virtualhost>
```

回顾nginx创建虚拟主机：写多个Server

```shell
http {
	server {
	listen :80;
	server_name www.a.com;
	root html;
	index index.html;
	}
}
```

tomcat创建虚拟主机：写多个Host

```shell
<Host name=www.a.com appbase=webapps>
</Host>
```

### tomcat主配置文件解析

主配置文件位置

/usr/local/tomcat/conf/server.xml

**！！！tomcat主配置文件中即使参数编写错误，重启服务时候有时候也不报错，显示成功开启，但配置文件却是错的！**

> 修改配置文件后，重启服务，使用ss检查服务是否正常开启。
>
> 若不是，说明配置文件书写有错误。

虚拟主机配置文件

> (初始配置文件123行-139行)

<Host 开头

> autoDeploy		//自动更新网站功能，方便网站开发工程师测试
>
> unpackWARs		//自动解war包（类tar包）

\</Host> 结尾

----------

##### 扩展知识：war包

war包，类tar包

创建war包的工具：Java-1.8.0-openjdk-devel

压缩命令：jar -cf  压缩包名 压缩的文件  存放路径

示例：将var下的log文件，创建war包，移动到tomcat虚拟主机的网页目录下，会发现war包已被释放

```shell
yum -y install java-1.8.0-openjdk-devel
[root@web1 tomcat]# cd /
[root@web1 /]# jar -cf xyz.war /var/log
[root@web1 /]# mv xyz.war /usr/local/tomcat/webb/
[root@web1 /]# cd /usr/local/tomcat
[root@web1 tomcat]# ls webb
ROOT  xyz  xyz.war
```

--------

**虚拟主机Host应在Engine框架内，而不应该在外面！**

```shell
123       <Host name="localhost"  appBase="webapps"
124             unpackWARs="true" autoDeploy="true">
125 
126         <!-- SingleSignOn valve, share authentication between web applications
127              Documentation at: /docs/config/valve.html -->
128         <!--
129         <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
130         -->
131 
132         <!-- Access log processes all example.
133              Documentation at: /docs/config/valve.html
134              Note: The pattern used is equivalent to using pattern="common" -->
135         <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
136                prefix="localhost_access_log" suffix=".txt"
137                pattern="%h %l %u %t &quot;%r&quot; %s %b" />
138 
139       </Host>
140     </Engine>
141   </Service>
```

### 创建虚拟tmocat虚拟主机

#### 修改server.xml主配置文档，添加Host虚拟主机

name：输入域名信息

appbase：网站的页面存放目录

```shell
<Host name="www.b.com" appBase="webb"	
      unpackWARs="true" autoDeploy="true">
</Host>

mkdir -p webb/ROOT
echo "webb/ROOT/index.html" > webb/ROOT/index.html
ping www.b.com
bin/shutdown.sh 
bin/startup.sh 
curl www.b.com:8080
```

#### 修改server.xml主配置文件，给host添加context

```shell
<Context path=""  docBase=""  />
```

##### docBase

1. 当docBase为空时，访问页面会直接在webb中，而不是webb/ROOT中

```shell
<Host name="www.b.com" appBase="webb"
      unpackWARs="true" autoDeploy="true">
    <Context path="" docBase="" />
</Host>

echo " web b /test.html" > webb/test.html  //创建测试页
```

重启服务，在浏览器中访问www.b.com:8080/test.html 测试

2. 再次修改配置文件，在docBase=""中加入bbb

访问页面为webb/bbb/index.html

```shell
<Host name="www.b.com" appBase="webb"
      unpackWARs="true" autoDeploy="true">
    <Context path="" docBase="bbb" />
</Host>

bin/shutdown.sh
bin/start.sh
ss -nuplt | grep java
```

此时重启服务会发现，服务根本没起来

原因：网页目录webb中，根本没有创建bbb目录

```shell
[root@web1 tomcat]# mkdir webb/bbb
[root@web1 tomcat]# echo " webb/bbb/index.html" > webb/bbb/index.html
```

重启服务，用ss查看端口启动情况，测试www.b.com:8080

3. 再次修改配置文件，在docBase="bbb"中，变更路径为绝对路径/b,

访问页面为/b/index.html

然后创建绝对路径目录及网页默认页内容，实现网站页面可以自由定义存放路径

```shell
<Host name="www.b.com" appBase="webb"
      unpackWARs="true" autoDeploy="true">
    <Context path="" docBase="/b" />
</Host>

[root@web1 tomcat]# mkdir /b
[root@web1 tomcat]# echo "/b/index.html" > /b/index.html
```

重启服务，测试www.b.com:8080，得到/b/index.html文件的网页内容

##### path

1. 原有配置文件中，继续在path后加/abc

```shell
<Host name="www.b.com" appBase="webb"
      unpackWARs="true" autoDeploy="true">
    <Context path="/abc" docBase="/b" />
</Host>
```

测试www.b.com:8080/abc  可以看到，访问到/b下的页面内容

此时测试www.b.com:8080  可以看到，展示的页面为配置中的appBase指定的页面

2. 继续在原有配置文件中，修改docBase指定的路径为相对路径

```shell
<Host name="www.b.com" appBase="webb"
      unpackWARs="true" autoDeploy="true">
    <Context path="/abc" docBase="bbb" />
</Host>
```

访问www.b.com:8080/abc  结果显示webb/bbb下的页面内容

访问www.b.com:8080    结果显示最原始的页面

**总结：**

**此操作可以隐藏真实目录，提高网页安全性**

**可以根据实际需求，配置文件中写入多个context，只要没有冲突，都可以实现**

案例：实现下述需求

访问www.b.com:8080   看到的是/usr/local/tomcat/abc/a 中的内容

访问www.b.com:8080/abc/  看到的是/var/www/html 中的内容

```shell
<Host name="www.b.com" appBase="abc"
      unpackWARs="true" autoDeploy="true">
<Context path="" docBase="a" />
<Context path="/abc" docBase="/var/www/html" />
```

## SSL加密网站

tomcat中，作加密效果后，全部网站均加密；配一次保全部

nginx中，配置是针对单个网站作配置；配一个生效一个

> tomcat中 ，加密的端口为：8443
>
> 在nginx中，加密端口为：443

1. 修改配置文件，添加内容

在配置文件中，已预写好ssl配置内容，寻找8443端口框架内容，去掉注释，在尾端添加密钥对的文件

> keystoreFile="/usr/local/tomcat/keystore" keystorePass="123456"

```shell
 <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" keystoreFile="/usr/local/tomcat/keystore" keystorePass="123456"/>
```

2. 使用命令keytool 创建密钥对文件(注意与配置文件中的密钥对配置一致)

此命令可使用--help查看参数使用方式

```shell
[root@web1 tomcat]#  keytool -genkeypair -alias tomcat -keyalg RSA -keystore /usr/local/tomcat/keystore
```

-genkeypiar		//创建密钥对

-alias		//别名

-keyalg		//使用什么算法，RSA为一种非对称算法

-keystore	//是密钥对文件的存储路径

3. 重启服务，使用ss查看端口启动情况

4. 浏览器测试 https://www.a.com:8443 

   或命令行 curl -k http:www.a.com:8443/

## 日志

位置: /usr/local/tomcat/logs		//按时间创建

默认tomcat虚拟主机中，只有默认网站有日志记录

未配置名称的话，日志名为：localhost.2020-10-21.log

其他网站需要作相应配置才有日志记录

#### 配置文件中的日志部分分析

参考默认主机的参数,可修改命名，显得更专业点哈哈哈哈哈

prefix：日志名字

suffix：日志后缀

```shell
<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
```





