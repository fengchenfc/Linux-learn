## Session & cookies

### Session

- 存储在服务器端，保存用户名、登录状态等信息
- **保存在服务器**

### cookies

- 由服务器下发给客户端，保存在客户端的一个文件里。

  保存的内容包括：SessionID
  
- **保存在客户端**

不同网站下发的cookies有效期由网站决策，根据服务的性质决定

> 淘宝等网站，登录过后比较久的一段时间都不用再输入用户信息
>
> 银行等app，比较短的时间都要重新输入

----------

### memcached概述

#### 数据存储对比

- 性能

CPU缓存>内存>磁盘>数据库

- 价格

CPU缓存>内存>磁盘>数据库

<u>综上考量，session存储在内存最优</u>

#### memcached简介

- mamcached是高性能的分布式缓存服务器

用来集中缓存数据库查询结果，减少数据库访问次数，以提高动态web应用的响应速度

- 官方网站：http://memcached.org/

- **macache即是一个服务，又是一个数据库，服务提供数据库！**

-------

### 构建web集群

通过Nginx调度器负载后端两台Web服务器，实现以下目标：

1. 部署Nginx为前台调度服务器
2. 调度算法设置为轮询
3. 后端为两台LNMP服务器
4. 部署测试页面，查看PHP本地的Session信息



![image-20201020153833930](C:\Users\Administrator\Desktop\Linux-note\image-20201020153833930.png)

#### 操作流程

> 准备Nginx调度器一台，web服务器两台

部署nginx调度器：默认配置，无额外模块

部署后端lnmp主机（动静分离）

```bash
yum -y install pcre-devel openssl-devel	//让nginx支持正则、加密
yum -y install php php-mysql php-fpm
yum -y install mariadb mariadb-devel mariadb-server
```

部署测试页面 

使用客户端访问测试页面

> 排错：
> 1. 是否关闭httpd服务
> 2. 防火墙
> 3. php-fpm服务
> 4. 如果删除nginx，重新安装需要先杀死nginx进程，然后重新开启服务
> 5. 缺少依赖包   pcre-devel   openssl-devel
> 6. 配置文件是否书写错误
> 7. 不要给nginx调度器配置lnmp动静分离

**服务器中存储SessionID的位置**

```shell
ls /var/lib/php/session
```

### 构建memcached服务

要求先快速搭建好一台memcached服务器，并对memcached进行简单的增、删、改、查操作：

- 安装memcached软件，并启动服务
- 使用telnet测试memcached服务
- 对memcached进行增、删、改、查等操作

#### 安装memcached

在nginx代理服务器上安装

```shell
yum -y install memcached

sytemctl start memcached
```

memcached 该软件无法通过传统方式管理配置，需要用php语言管理；

所以这里通过其他工具测试此工具(telnet)

可以创建、覆盖、删除数据

```shell
yum -y install telnet    //类似ssh,这里可以用来测试memcached

[root@proxy nginx]# telnet 127.0.0.1 11211    //访问本机11211端口，此端口为memcached服务端口

Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
  
ERROR		//敲回车显示此结果

set abc 0 100 3		//创建或者覆盖abc变量，0是不压缩数据，存储时间100秒生命周期，存储3个字符
123
STORED
get abc		//获取变量abc的值
VALUE abc 0 3
123
END


```

set：可以创建，也可以覆盖

add：只能创建，不能覆盖

replace：覆盖现有变量的数据

get：获取变量的值

delete：删除变量

flush_all：清空所有数据

quit：退出

### PHP实现session共享

实现session会话共享：

- 配置PHP使用memcached服务器共享Session信息
- 客户端访问两台不同的后端Web服务器时，Session 信息一致

#### 安装PHP的memcache扩展

在web服务主机上安装php-pecl-memcache

使web服务器可以使用memcached

```shell
yum -y install php-pecl-memcache
```

打开php-fpm配置，修改末尾配置，并重启服务使生效

```shell
php_value[session.save_handler] = memcache
php_value[session.save_path] = tcp://192.168.2.5:11211

systemctl restart php-fpm
```

使用浏览器测试，访问php测试登录页面，刷新可看到两web服务器轮询，输入用户名密码后，可一次性登录入跳转home页面即成功。