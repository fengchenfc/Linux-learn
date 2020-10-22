# Varnish代理服务器

## 基本概念

有时网站服务器搭建好了客户会因为距离较远而访问效果不好，

出现这种情况就可以用varnish工具在距离客户比较近的地区搭建缓存服务器，然后客户访问缓存服务器即可.

缓存服务器会从原始站点获得数据并缓存，

随着被访问与被缓存的数据越来越多,

客户的访问速度就可以加快了，但由于这种方式不是一般企业可以做到，

通常有需求时可以去购买cdn （内容分发网络）服务。

## Varnish服务器

- Varnish一款高性能且开源的反向代理服务器
- Varnish具有性能高、速度快、管理方便等诸多优点

## 部署Varnish

1. 安装varnish服务

```shell
首先使用web1主机开启httpd服务
[root@web1 ~]# ss -ntulp | grep :80    //可以先查询80端口是否被占用
killall nginx    //如果nginx占用就杀掉
[root@web1 ~]# systemctl start httpd   //仅仅启动httpd
再使用proxy测试web1的页面
[root@proxy ~]# curl 192.168.2.100
[root@proxy ~]# ss -ntulp | grep :80
[root@proxy ~]# killall nginx  //如果有服务占用80端口就杀死
[root@proxy ~]#cd  ~/ lnmp_soft
[root@proxy lnmp_soft]# tar -xf varnish-5.2.1.tar.gz
[root@proxy lnmp_soft]# cd varnish-5.2.1/
yum -y install gcc readline-devel pcre-devel python-docutils  //安装varnish所需依赖包
./configure
make
make install
useradd -s /sbin/nologin varnish    //创建varnish所需账户
```

2. 修改配置

```shell
cp etc/example.vcl /usr/local/etc/default.vcl  //拷贝配置文件
vim /usr/local/etc/default.vcl  //修改配置文件17、18行
    .host = "192.168.2.100";   //原始服务器的ip
    .port = "80";					//原始服务器的端口号
varnishd  -f  /usr/local/etc/default.vcl   //指定配置文件路径并启动varnish服务
```

3. 测试效果

测试访问proxy的页面，可以看到原始服务器web1的内容

```shell
[root@proxy varnish-5.2.1]# curl 192.168.2.5  
```



