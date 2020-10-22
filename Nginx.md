## 虚拟机准备工作
四台虚拟机
> 最小化安装
>
> > proxy   两块网卡 192.168.2.5（vmnet2）  192.168.4.5（vmnet4）
> > web1	192.168.2.100（vmnet2）
> > web2	192.168.2.200（vmnet2）
> > client	     192.168.4.10（vmnet2）

> 配置YUM仓库，安装基础软件
>
> > vim		//vim编辑器
> > base-completion		//tab补全



- 新系统基础常用软件包

bash-completion	//支持tab键的工具包

vim	//vim文本编辑器

net-tools	//网络相关工具，比如ifconfig命令

psmisc	//支持killall命令的软件包

-----

## NGINX
### Web服务器对比
- Unix和Linux平台下
  - Apache、Nginx、Tengine、Lighttpd
  - Tomcat、IBM WebSphere、Jboss
- Windows平台下
  - 微软公司的IIS(Internet Information Server)
#### 反向代理与正向代理概念

两者的区别在于代理的对象不一样：

**正向代理**代理的对象是客户端，**反向代理**代理的对象是服务端

- 正向代理

​       有时网站服务器搭建好了客户会因为距离较远而访问效果不好，出现这种情况就可以用varnish工具在距离客户比较近的地区搭建缓存服务器，然后客户访问缓存服务器即可，缓存服务器会从原始站点获得数据并缓存，随着被访问与被缓存的数据越来越多，客户的访问速度就可以加快了，但由于这种方式不是一般企业可以做到，通常有需求时可以去购买cdn （内容分发网络）服务。

- 反向代理

​       有时网站服务器搭建好了客户会因为距离较远而访问效果不好，出现这种情况就可以用varnish工具在距离客户比较近的地区搭建缓存服务器，然后客户访问缓存服务器即可，缓存服务器会从原始站点获得数据并缓存，随着被访问与被缓存的数据越来越多，客户的访问速度就可以加快了，但由于这种方式不是一般企业可以做到，通常有需求时可以去购买cdn （内容分发网络）服务。

**********
### Nginx简介

- Nginx("engine x")
  -	是俄罗斯人编写的十分轻量级的HTTP服务器
  - 是一个高性能的HTTP和反向代理服务器，同时也是一个IMAP/POP3/SMTP代理服务器
  - 官方网站：http://nginx.org/

### Nginx环境搭建

1. 传输软件源码包（lnmp_soft.tar.gz），拷贝到root家目录下并原地释放
2. 释放lnmp_soft目录中的nginx-1.17.6.tar.gz
3. 安装gcc、make工具；安装nginx搭配工具

```bash
yum -y install gcc  //安装编译工具
yum -y install pcre-devel  //安装让nginx支持正则的软件包
yum -y install openssl-devel  //让nginx搭建基于ssl（安全加密）
```

4. 编译安装,并开启服务

```bash
./configure  --with-http_ssl_module  //配置，添加安全模块
make   //编译
make install   //安装
/usr/local/nginx/sbin/nginx   //开启服务		

./configure --help | grep http_ssl   //另外，可以用该方式搜索与http_ssl有关的模块，--help可以看配置nginx时的配置选项与模块

cd /usr/local/nginx     //进入nginx安装目录
ls    //查看
```

### nginx进程管理

```bash
/usr/local/nginx/sbin/nginx  -V  //查看版本已经添加的功能模块
/usr/local/nginx/sbin/nginx   //开启
/usr/local/nginx/sbin/nginx  -s stop  //关闭
/usr/local/nginx/sbin/nginx  -s reload   //重新加载配置文件，服务必须是开启状态
ps aux | grep nginx    //查看服务相关进程
netstat-untlp | grep nginx		//查看服务占用端口信息
```

### nginx中的主要目录及文件

/usr/local/nginx 	//服务所在位置
conf 	//存放配置文件
html 	//存放网页文件
sbin 	//存放主程序
logs 	//存放日志

nginx/conf/nginx.conf.default	//配置文件的备份，在重新做实验或者配置文件被破坏需要还原是可以用该文件拷贝覆盖nginx.conf


> tips：快速寻找一对括号的位置
vim中，在一个大括号/其他括号上，按%（shift + 5），可以直接跳转到匹配的相对括号

### nginx配置解析
#### 配置文件结构

全局配置

```bash
http {
		…………
		server {
				…………
				location / {
						…………
				}
		}
}
```

### 用户认证

###### 案例1：为nginx增加网站认证功能

> 通常情况下网站搭建好之后，只要知道ip或者域名，那么任何用户都可以访问该网站，如果仅仅想让某些用户访问就可以使用该功能

1. 首先打开主配置文件，在http框内，server框外（42行参考位置）添加

```bash
vim /usr/local/nginx/conf/nginx.conf

auth_basic  "password:";   //提示信息，用户登录网站时看到的
auth_basic_user_file  "/usr/local/nginx/pass";  //存放用户名密码的文件路径
```

2. 安装网站工具包，支持htpasswd命令

```bash
yum -y install httpd-tools  //安装网站工具包，支持htpasswd命令
htpasswd -c  /usr/local/nginx/pass  tom  //创建网站的用户与密码文件，第1个用户名为tom（可以自定义），之后输入2次密码
cat /usr/local/nginx/pass   //查看pass文件
```

3. 重加载配置，并使用浏览器测试结果

> 如果要反复测试该狗狗你能，需每次操作前清空浏览器的历史记录

```bash
/usr/local/nginx/sbin/nginx -s reload  //如果nginx服务已经开启，就重加载配置文件
/usr/local/nginx/sbin/nginx  //如果nginx服务没开启，就开启使用火狐浏览器打开192.168.2.5发现已经需要用户名和密码认证
htpasswd  /usr/local/nginx/pass abc  //追加新账户，无需c选项
如果要反复测试该功能，需要清空浏览器的历史记录
```



### nginx虚拟主机

通常使用一台服务器开启一个nginx服务就可以开启一个网站，但是如果公司需要很多不同域名的网站，而每个网站的业务量不大时，不必购买多台服务器，使用一台服务器利用虚拟主机技术既可以实现。

- 基于域名的虚拟主机（最常用）
- 基于端口的虚拟主机
- 基于IP的虚拟主机



> 回顾虚拟主机
> 在httpd中配置：
>
> ```bash
> <virtualhost *:80>
> 	Servername www.a.com
> 	Documentroot /var/www/html
> </ virtualhost>
> ```



在nginx中配置(基本框架)
```bash
http {	
	server {
	listen :80;    //监听端口号
	server_name www.a.com;  //网站域名
	root html;    //网站页面文件存储的位置
	index index.html   //默认页面
    }
}
```



正式配置

1. 修改主配置文件，在34~39行开始添加以下内容

```bash
34     server {
35         listen 80;   //监听端口
36         server_name www.b.com;   //域名
37         root b;  //网页文件存放地
38         index index.html;  //默认页文件名
39      }
```

2. 准备测试页面

```bash
cd  /usr/local/nginx
mkdir b  //创建b网站的页面存放目录
echo "proxy_nginx_web_b~~~~~"  > b/index.html  //b网站内容
echo "proxy_nginx_web_a~~~~~"  > html/index.html  //a网站内容

vim /etc/hosts  添加域名与ip的映射关系
192.168.2.5 www.a.com www.b.com www.c.com

curl www.b.com			测试b网站
curl www.a.com  		测试a网站
```

> 如果在windows环境测试域名访问网站，需要修改hosts文件，添加一样的内容（hosts文件的权限需要设置为完全控制）
> C:\Windows\System32\drivers\etc\hosts
> 设置hosts的权限要先右键---属性---安全---编辑---users---完全控制勾选，然后右键该文件---打开方式---选择文本方式打开在最后一行下面写入与之前linux一样的内容
> 192.168.2.5  www.a.com  www.b.com  www.c.com

然后使用火狐浏览器访问www.a.com或者www.b.com

### 基于ssl技术的安全网站

#### 加密算法

- 对称加密，适合单机环境
  - 加密和解密是相同密码
  - 应用场景：RAR、ZIP压缩加密等

- 非对称加密，适合网络数据传递
  - 加密和解密分别使用公钥和私钥
  - 证书（包含公钥)，可以哟个你来加密
  - 私钥，可以用来解密
  - 应用场景：https、ssh

#### Hash值

- 信息摘要，数据校验
  - md5（命令：md5sum）
  - SHA256
  - SHA512
  - 应用场景：数据完整性校验

使用md5工具计算nginx配置文件随机校验值，以后可以根据该校验值判断文件是否被篡改

```bash
[root@proxy nginx]# md5sum conf/nginx.conf

3a0b1f2d0a5734fe3200a48703bafed2  conf/nginx.conf
```

###### 案例：搭建基于ssl技术的安全网站

1. 修改主配置文件中，被注释的HTTPS server区域

> 在第103行左右，找安全网站的虚拟主机的配置
>
> 去掉注释（可以使用:103,120s/#//） 

```bash
103     server {
104         listen       443 ssl;
105         server_name  www.c.com;
106
107         ssl_certificate      cert.pem;   //证书文件
108         ssl_certificate_key  cert.key;    //私钥文件
109
110         ssl_session_cache    shared:SSL:1m; 
111         ssl_session_timeout  5m;
112
113         ssl_ciphers  HIGH:!aNULL:!MD5;
114         ssl_prefer_server_ciphers  on;
115
116         location / {
117             root   c; 
118             index  index.html index.htm;
119         }
120     }
```

2. 创建私钥与证书

```bash\
cd /usr/local/nginx/conf
openssl genrsa > cert.key  创建私钥
openssl req -new -x509 -key cert.key > cert.pem  根据刚刚创建的私钥，再创建证书(包含了公钥),生成过程会询问诸如你在哪个国家之类的问题，可以随意回答，但要走完全过程
Country Name (2 letter code) [XX]:dc			国家名称
State or Province Name (full name) []:dc		省份名称
Locality Name (eg, city) [Default City]:dc        城市名称
Organization Name (eg, company) [Default Company Ltd]:dc    公司名称
Organizational Unit Name (eg, section) []:dc     部门名称
Common Name (eg, your name or your server's hostname) []:dc    服务器名称
Email Address []:dc@dc.com     邮件地址
```

3. 测试

```bash
[root@proxy nginx]# sbin/nginx -s reload   重新加载配置
```

使用火狐浏览器访问https://www.c.com/   看到提示---高级---接收风险
或者使用curl -k https://www.c.com/ 用命令行的方式访问，使用-k选项可以忽略风险提示直接看网页内容
netstat -ntulp | grep nginx   //查看nginx可以看到443端口已经开放



----------

### LNMP

#### 什么是LNMP
- -L:Linux操作系统
- -N:Nginx网站服务软件
- -M:MySQL、MariaDB数据库
- -P:网站开发语言（PHP、Perl、Python）

  - php：目前一个比较主流的网站开发语言



![image-20201017105526509](C:\Users\Administrator\Desktop\Linux-note\image-20201017105526509.png)



![image-20201017110539418](C:\Users\Administrator\Desktop\Linux-note\image-20201017110539418.png)



可以理解为：php-fpm就是fastCGI

php-fpm：指挥官，指挥fastCGI来为nginx指路，来找php解释器

fastCGI：中转，中介，帮nginx找到php解释器

#### 网站类型

- 静态网站
  -在不同环境下，浏览器的内容不会发生变化 
- 动态网站
  -在不同环境下，浏览网站的内容有可能会发生变化，该种类网站显示内容效果更好，会让用户体验更好，更适合目前市场环境

> 比如百度，搜索天气，不同地区显示不同地区的天气情况，这就是动态网站。
>



#### 部署LNMP环境

##### 1. 准备nginx

```bash
rm -rf /usr/local/nginx/   //删除原有环境（cd到用户家目录~，然后使用此命令，可删除源码编译安装的软件
netstat -ntulp | grep :80   //查询80端口的占用情况
killall nginx   //如果nginx还在运行，可以杀掉
cd lnmp_soft/nginx-1.17.6/
./configure --help | grep http_ssl  //查询模块名称
./configure --with-http_ssl_module  //配置
make   //编译
make install   //安装
```

##### 2. 安装其他相关软件包（PHP、MariaDB）

```bash
yum -y install mariadb mariadb-server mariadb-devel  
//安装数据库的客户端、服务端、依赖包
yum -y install php   //安装php解释器程序
yum -y install php-mysql  //安装php与数据关联的软件包
yum -y install php-fpm   //安装让nginx具有动态网站解析能
力的软件包
systemctl  restart  mariadb   //开启数据库
systemctl  restart  php-fpm	//开启php-fpm
netstat -ntulp | grep :3306    //查看数据库端口
netstat -ntulp | grep :9000		//查看php-fpm端口
```

数据库端口：3306

php-fpm端口：9000

##### 3. 测试效果（目前还无法让nginx解析动态网站）

```bash
cd  /root/lnmp_soft/php_scripts
cat test.php  //查看文件，有普通的网站语言(html)，也有php语言
cp test.php /usr/local/nginx/html/   //拷贝动态网站页面到nginx的html目录下，使用火狐浏览器访问192.168.2.5/test.php可以看到下载文件的提示
```

##### 4. 分析没有看到动态网站的原因：Nginx默认只支持静态页面
>用户如果访问静态页面
>
>>nginx会直接将该页面传递给用户的主机
>>
>>用户再使用浏览器解析该页面内容



>用户如果访问动态页面（如php语言编写的test.php页面）
>
>> nginx要先将该页面扔给后台的php-fpm
>>
>> 该程序就会利用php解释器翻译php语言编写的网站页面
>>
>> 然后发给nginx程序
>>
>> nginx再转发给用户
>>
>> 最终用户才能看到动态页面

##### 5. Nginx实现动静分离

为了让nginx支持动态网站的解析，要先修改配置文件

> /usr/local/nginx/conf/nginx.conf
>
> 默认第65-71行去掉注释，69行不用去，才可以实现动态页面解析

```bash
65        location ~ \.php$ {   //~是使用正则表达式，匹配以.php结尾
66           root           html;
67           fastcgi_pass   127.0.0.1:9000;   
//一旦用户访问了.php结尾的文件，就让nginx找后台的php-fpm（端口号9000）
68           fastcgi_index  index.php;
69         #   fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi  
  _script_name;
70           include        fastcgi.conf;   //这里需要修改名称
71         }
```

```bash
[root@proxy nginx]# sbin/nginx -s reload   //重新加载nginx配置文件
```

再使用火狐浏览器访问192.168.2.5/test.php ,就可以看具体页面内容了

**在nginx配置文件中，~表示使用正则表达式**

##### 6. 再测试调用数据库的网页

首先拷贝页面

```bash
cd ~/lnmp_soft/php_scripts/
cp mysql.php /usr/local/nginx/html/
```

再用火狐浏览器访问 http://192.168.2.5/mysql.php



##### 测试数据库内容发生变化之后网页是否变化

mysql		登录数据库，手工创建dc账户

```bash
MariaDB [(none)]> create user dc@localhost identified by '123';
```


再用刷新http://192.168.2.5/mysql.php页面，会看到dc账户



##### 进一步了解php-fpm服务：
php-fpm 是fastCGI的进程管理器
fastCGI  快速公共网关接口，可以用来关联网站服务与解释器

了解php相关配置文件，目前无需配置

```bash
vim /etc/php-fpm.d/www.conf   //php有关的配置文件
listen = 127.0.0.1:9000    //此处配置决定了php-fpm服务针对什么ip与什么端口
pm.start_servers = 5    //一上来开启的进程数量，pstree可以查看php-fpm的进程数量，如果修改了需要重启php-fpm服务
pm.max_children = 50   //开启的fastCGI进程最大数量
```


**********

### 地址重写
#### 基础知识
- 什么是地址重写
  - 获得一个来访的URL请求，然后改写成服务器可以处理的另一个URL的过程
  > （访问a.html，显示b.html)
  > www.abcd1234.com	www.abc.com
- 地址重写的好处
  - 缩短RUL，隐藏实际路径提高安全性
  - 易于用户记忆和键入
  - 易于被搜索引擎收录
#### 地址重写语法格式 rewrite语法
- rewrite基本语句
  - rewrite	**regex**	replacement	**flag**
> ​		rewrite	旧地址(支持正则)	新地址	[选项]
> ​								/a.html			/b.html

- 旧地值支持正则表达式，以防止页面误跳转

- 选项：redirect	#加入此选项后，访问a地址跳转b地址后，地址栏变成a页面。否则只发生页面变化，地址栏不变



#### 地址重写格式总结

rewrite 旧地值 新地址 [选项];

last		不再读其他rewrite

break		不再读其他语句，结束请求

redirect		临时重定向

permanent		永久重定向



##### 应用方法

- redirect 临时重定向(http状态码302)
- permanent 永久重定向(http状态码301)

打开nginx主配置文件，在42行添加
rewrite  /a.html  /b.html  redirect; 
然后开启或者重加载nginx服务
curl 192.168.2.5/a.html		//此时看到的页面是b



rewrite  /a.html  /b.html  permanent;
sbin/nginx -s reload   //重加载nginx服务
curl 192.168.2.5/a.html 	//此时看到的页面也是b，说明redirect与permanent效果在客户机看来是一样的，但是状态码不一样，对于搜索引擎来说更关心301的



- last 不再读其他rewrite
- break 不再读其他语句，结束请求



[root@proxy nginx]# echo nginx_web_c~ > html/c.html  //准备c页面
打开主配置文件，在42行修改
rewrite /a.html /b.html;
rewrite /b.html /c.html;
sbin/nginx -s reload   //重加载nginx服务
使用火狐访问192.168.2.5/a.html 看到的是c.页面

rewrite /a.html /b.html last;   //然后再修改配置添加last可以实现不继续跳转
sbin/nginx -s reload   //重加载nginx服务
使用火狐访问192.168.2.5/a.html 看到的是b.html

再次按下图修改配置文件，删除原有rewrite语句，如果使用last无法阻止第
二个location中的rewrite语句，而break可以

![image-20201019195010278](C:\Users\Administrator\Desktop\Linux-note\image-20201019195010278.png)

开启或者重加载服务
使用火狐访问192.168.2.5/a.html 看到的是b.html

#### 案例：地址重写

###### 案例1：相同网站内的页面跳转，地址栏不变

打开nginx著配置文件，在server{} (<u>默认配置42行</u>) 添加

```bash 
rewrite	^/a.html$ 	/b.html;	

sbin/nginx -s reload   重加载配置文件
```

> 当访问a页面时可以看到b页面的内容,此处的a.html匹配可以使用正则没有添加^和$之前，可以匹配/xyz/a.html 或 /a.htmlxyz添加了正则符号之后匹配的更精准，只针对/a.html进行重写。

使用浏览器访问192.168.2.5/a.html ,结果显示b.html内容

###### 案例2： 相同网站内的页面跳转，地址栏发生变化

打开nginx著配置文件，在server{} (<u>默认配置42行</u>) 添加

```bash
rewrite  /a.html$  /b.html  redirect;   //添加了redirect（重定向）选项

sbin/nginx -s reload   重加载配置文件
```

使用火狐浏览器访问192.168.2.5/a.html

可以看到页面换成b.html的同时，浏览器地址栏也发生了变化

###### 案例3：不同网站的地址跳转

打开nginx主配置文件，server{} (<u>默认配置42行</u>) 添加

```bash
rewrite  /  http://www.tmooc.cn;   //访问老网站，跳转到新的
sbin/nginx -s reload   //重加载配置文件
```

使用火狐浏览器访问192.168.2.5/a.html 会跳到tmooc.cn

###### 案例4：不同网站的相同页面的地址跳转

打开nginx主配置文件，server{} (<u>默认配置42行</u>) 添加

```bash
rewrite  ^/(.*)$  http://www.tmooc.cn/$1; 

sbin/nginx -s reload   重加载配置文件
```

> 访问老网站的某个页面时，跳转到新网站对应的相同页面。
>
> 前面使用正则表达式匹配用户输入的任意页面，并保存起来（小括号在正则中的效果是保留，相当于保存复制）
>
> 后面使用$1将之前保存的页面地址粘贴到新网站


使用火狐浏览器访问192.168.2.5/a.html

访问192.168.2.5/子页面a，重定向至www.tmooc.cn/子页面a



>在rewrite语法中，正则表达式无变化

>但是正则表达式中前者使用（）保留的内容，后者写入不再使用sed中的方式

>sed方式为(a)(b)(c)---\1\2\3\

>此处方式为(a)(b)(c)---$1 $2 $3

###### 案例5：根据用户的情况决定跳转到什么样的页面

```bash
cd  /usr/local/nginx/html
mkdir  firefox   //创建火狐浏览器专属文件的目录
echo "proxy_nginx_firefox~" > firefox/test.html  //准备火狐专用页面
cd ..
echo "proxy_nginx_other~"  > html/test.html  //准备普通浏览器使用的页面
vim conf/nginx.conf  //删除原有地址重写的配置，然后在第46行添加
if ($http_user_agent ~* firefox) {   
//如果用户的浏览器使用了火狐，就执行下面的rewrite任务，~代表匹配正则，*是不区分大小写，$http_user_agent是nginx的内置变量，存储了用户的信息，比如用的什么浏览器
rewrite ^/(.*)$  /firefox/$1;     //就跳转到火狐专用目录的页面
}
sbin/nginx -s reload   重加载配置文件
```

分别使用火狐浏览器与其他浏览器访问192.168.2.5/test.html

可以得到两个不同页面内容则成功

--------------

### Nginx实现网站代理功能

一台服务器的能力是有限的，如果客户访问量比较大，可以利用nginx的代理功能组建集群，集群中的服务器越多集群整体性能就越强。

#### 集群中服务器的准备工作，这里使用web1和web2



1. 在web1与web2安装常用工具

```bash
yum -y install vim
yum -y install bash-completion    //tab键补全软件包
yum -y install net-tools   //网络工具软件包，支持ifconfig等命令
```

在web1与web2安装网站服务

```bash
yum -y install httpd
echo web1 > /var/www/html/index.html   //这里如果是web2主机要改成echo  web2
systemctl start httpd
systemctl stop firewalld
```

使用proxy测试web1与web2

```bash
[root@proxy nginx]# curl 192.168.2.100
web1
[root@proxy nginx]# curl 192.168.2.200
web2
```

2. 在proxy主机添加创建集群配置

首先是34~37行

```bash
upstream web {		  //创建nginx集群，名称是web
     server 192.168.2.100:80;     //这里是集群中的服务器ip与端口
     server 192.168.2.200:80;     //第二台集群主机
  }
```

然后在47行

```bash
47         proxy_pass http://web;   //调用web集群

sbin/nginx -s reload   //重加载nginx服务
```

使用火狐访问192.168.2.5  不断刷新(或者使用curl 192.168.2.5)

看到的页面内容是web1与web2之间切换


#### 集群优化

在集群中的主机ip与端口后面可以加入以下参数，可以实现不同效果

###### 1. 任务量的分配，性能较强的服务器可以多分配工作量

```bash
server 192.168.2.100:80 weight=2;  //权重，值越大，工作量越大
```

####### 2. 健康检查功能

```bash
server 192.168.2.100:80 max_fails=2 fail_timeout=30;  
//检测2次.如果失败，则认为集群中的服务器故障,认为集群中的服务器故障之
后等待30秒才会再次链接
```

###### 3. 集群主机需要维护时

```bash
server 192.168.2.100:80  down;   
//加down标记，使集群服务器暂时不参与集群的任务轮询
```

###### 4. 相同客户机访问相同服务器

```bash
  upstream web {		
	 ip_hash  //相同客户机访问相同服务器，让一个客户机访问集群时锁定一个后台服务器，避免重复登陆的问题
     server 192.168.2.100:80;   
     server 192.168.2.200:80;  
  }
```

### 使用Nginx实现其他业务的集群

> 通常情况下nginx是搭建网站的工具，还可以组建网站集群.
>
> 但如果后端的集群服务器跑的不是网站业务
>
> 就可以利用--with-stream模块创建非网站业务的集群。

1. 准备工作

在proxy主机：停止nginx服务，并删除nginx
回到家目录下的lnmp_soft的nginx目录重新编译安装nginx，将上述两个模块都添加上。

其中--with-stream是四层代理模块可以让nginx组建其他业务(非web)的集群，另外一个模块可以查看网站后台数据（后面的实验需要）。

```bash
./configure --with-stream --with-http_stub_status_module
make
make install

sbin/nginx  -V   //检查安装的版本与模块
sbin/nginx  //开启服务
```

web1与web2的root密码要设置成一致
proxy的家目录下的.ssh目录中known_hosts文件,记录了都远程登录过哪些机器，可以先删除

2. 创建集群

打开nginx主配置文件，在16行左右(http上面)，添加以下内容

```bash
16 	stream {				//创建新业务
17     upstream backend {     //创建名叫backend的集群
18         server 192.168.2.100:22;  //集群中的主机使用22端口对外提供服务
19         server 192.168.2.200:22;
20  }
21     server {
22         listen 12345;    //监听端口号
23         proxy_pass backend;    //调用集群
24  }
25  }
```

写完后启动服务或者重新加载配置

先删除远程连接的主机记录，之后每次连接测试都先删除该文件

```bash
[root@proxy nginx]# rm -rf ~/.ssh/known_hosts  //删除远程连接的主机记录
[root@proxy nginx]# ssh 192.168.2.5 -p 12345  //使用12345端口号连接代理服务器
```


此处使用自己连接自己测试，如果使用client主机就远程连接192.168.4.5

-----

### 扩展：ss命令 

**查看系统中启动的端口信息**

- -a显示所有端口的信息

- -n以数字格式显示端口号

- -t显示TCP连接的端口

- -u显示UDP连接的端口

- -l显示服务正在监听的端口信息，如httpd启动后，会一直监听80端口

- -p显示监听端口的服务名称是什么（也就是程序名称）

<u>**注意：**</u>

<u>**在RHEL7系统中可以使用ss命令替代netstat命令，功能一样，选项一样。**</u>

----

### nginx优化

#### HTTP错误代码

- 常见错误代码表

| 返回码 | 描述                                                         |
| :----- | ------------------------------------------------------------ |
| 200    | 一切正常                                                     |
| 400    | 请求语法错误                                                 |
| 401    | 访问被拒绝（账户或密码错误）                                 |
| 403    | 资源不可用，通常由于服务器上文件或目录的权限设置导致         |
| 403    | 禁止访问：客户端的IP地址被拒绝                               |
| 404    | 无法找到指定位置的资源（Not Found）                          |
| 414    | 请求URI头部太长                                              |
| 500    | 服务器内部错误                                               |
| 502    | 服务器作为网关或者代理时，为了完成请求访问下一个服务器。但该服务器返回了非法的应答（Bad Gateway） |

- nginx在404错误上的优化处理

> 客户访问网站时，如果看到了不存在的页面会有404报错的英文提示，这种提示很不友好，可以通过自定义页面改善用户体验

首先修改配置文件 

```bash
error_page  404        /test.jpg;   
//如果客户访问了不存在的页面就显示test.jpg的内容
```

保存退出
找一张图片，内容随意，比如用中文标注"抱歉！您访问的页面不存在" 

然后保存成test.jpg格式

然后拷贝到proxy主机的/usr/local/nginx/html目录下
重新加载nginx配置
使用浏览器随意访问不存在的页面192.168.2.5/XXXX.html  

就可以看到之前那张图片的内容	

--------

#### Nginx查看服务器状态信息（非常重要的功能！）

- status模块
  - --with-http_stub_module 开启模块功能
  - 可以查看Nginx连接数等信息

> 安装nginx源码包时，在./configure 后添加 此模块安装

- status页面

修改配置文件nginx.conf
```bash
  location /status {   //当用户输入的地址后面跟了/status之后
        stub_status on;   //开启网站后台状态信息查看功能
}
        error_page  404        /test.jpg;

curl http://192.168.2.5/status   //查看测试，仅仅2.5可以看
```

避免信息泄露，可再次优化代码,使状态信息仅设定ip可访问，其他ip拒绝

```bash
location /status {   //当用户输入的地址后面跟了/status之后
        stub_status on;   //开启网站后台状态信息查看功能
	    allow 192.168.2.5;   //仅仅允许2.5查看
        deny all;   //拒绝其他所有主机
}
        error_page  404        /test.jpg;

curl http://192.168.2.5/status   //查看测试，仅仅2.5可以看
```

浏览器/curl访问页面


```
Active connections: 2 						
server accepts handled requests	   
 17 17 28 
Reading: 0 Writing: 1 Waiting: 1 
```

Active connections：当前活动的连接数量

accepts：已经接受客户打u你的连接总数量（累加，除非重启服务器）

handled：已经处理客户端的连接总数量（已完成三次握手）

> 一般与accepts一致，除非服务器限制了连接数量

requests：客户端发送的请求数量

Reading：当前服务器正在读取客户的请求数量

Writing：当前服务器正在写喜爱能感应信息的数量

Waiting：当前多少客户端在等待服务器的响应

----------------

#### 配置nginx缓存数据的功能

客户在访问服务器时，有可能随时需要重复的文件；

如果反复从服务器获取会造成资源浪费，还耽误时间；

可以通过配置将常用的或者比较大的文件在用户访问一次之后就缓存在客户机中，下次客户访问时不用再找服务器而是从本机获取。



**修改配置文件 **

在默认的location的下面添加一个新的location

```bash
location ~* \.(jpg|png|mp4|html)$ {   //当用户访问的是这几种类型的
文件时
expires  30d;  //都会缓存在客户机上30天
}
```

然后使用火狐浏览器，先清空历史记录

然后地址栏输入about:cache
查看disk文件的列表

找到被访问文件看最后倒数第2列信息显示多久超时



#### 解决客户端访问头部信息过长的问题

> 优化nginx数据包头缓存，支持更长的地址，默认情况下nginx无法支持长地址栏，会报414错误。

在配置文件中的默认虚拟主机的上面添加（http内，server外）条件

```bash
client_header_buffer_size 200k;	//存储地址栏等信息的空间大小是200k
large_client_header_buffers 4 200k;	//如果不够再给4个200k

[root@proxy nginx]# sbin/nginx -s reload	//重启服务

./buffer.sh		//执行测试脚本测试
```



#### 压力测试

- 常用压力测试工具
  - ab
    - -ab -c 并发数	-n  总请求数	URL
- 其他常见压力测试软件（需要额外下载）
  - -http_load、webbench、siege



总结：

1. nginx配置文件：进程数量，单进程支持的并发访问数量

2. 系统的配置文件：系统支持文件的打开量



准备测试环境

在测试虚拟机安装httpd-tools，以使用ab工具

使用ab工具作压力测试

```bash
yum -y install http-tools

ab -c 100 -n 100 http://192.168.2.5/
结果：成功
ab -c 1000 -n 1000 http://192.168.2.5/
结果：成功
ab -c 3000 -n 3000 http://192.168.2.5/
结果：失败
```

优化配置参数，改变开启的进程数量

```bash
worker_processes  1;		//开启的进程数量,默认为1

events {
	worker_connections 1024;	//一个进程最大支持并发量
}
```

```bash
worker_processes  2;		//变更开启的进程为2

events {
	worker_connections 20000;	//变更最大支持进程20000
}
```

变更进程开启参数后，相应的进程也会看到变化

```bash
ss -nutlp | grep nginx	//查看服务状态
/sbin/nginx -s reload
ss -nutlp  |  grep nginx	//再次查看服务状态
```

进行压力测试，测试端失败出现

**socket: Too many open files (24)**		//打开了太多文件

> Linux默认定义：同一时间，同一个文件最多被同时打开1024次

继续优化，在proxy虚拟机上修改最大文件开启限制（永久）

> 该配置文件分4列，分别如下：

> #用户或组	硬限制/软限制	需要限制的项目	限制的值

```bash
[root@proxy nginx]# ulimit -n	//查看本机支持的最大开启文件数量
1024		//结果

]# vim /etc/security/limits.conf	//配置文件

53 *      soft    nofile        100000	//去#，改后面参数
54 *      hard    nofile        100000  //去#，改后面参数

reboot	//重启虚拟机，使配置生效
```

> 临时修改文件访问限制，以下两个命令都需要输入才能生效，一般软硬限制数值一样
>
> ulimit -Hn 100000		//临时修改（设置硬限制，硬件上的）
>
> ulimit -Sn  100000		//临时修改（设置软限制，软件上的）

测试端作同样操作（因测试端机器也需要同时打开与服务端同数量的文件）

在客户端使用ab命令测试





