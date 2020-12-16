# RPM打包

## RPM与源码包

- RPM包
  - 使用便利
  - 更新速度慢
- 源码包
  - 使用繁琐
  - 更新速度快

## 应用场景

- 官方未提供RPM包
- 官方RPM无法自定义
- 大量源码包，希望提供统一的软件管理机制

## 软件打包

打包方案：

准备源码软件，安装rpm-build软件包，编写SPEC配置文件，编译RPM软件包

### 步骤一：安装rpm-build软件

1. 安装rpm-build软件包

```shell
[root@web1 ~]# yum -y install  rpm-build
```

2. 生成rpmbuild目录结构

```shell
[root@web1 ~]# rpmbuild -ba nginx.spec  //会报错，没有文件或目录
[root@web1 ~]# ls /root/rpmbuild        //自动生成的目录结构
BUILD  BUILDROOT  RPMS  SOURCES  SPECS  SRPMS
```

3. 准备工作，将源码软件复制到SOURCES目录

```shell
[root@web1 ~]# cp nginx-1.12.2.tar.gz /root/rpmbuild/SOURCES/
```

4. 创建并修改SPEC配置文件

```shell
[root@web1 ~]# vim /root/rpmbuild/SPECS/nginx.spec 
Name:nginx                                   #源码包软件名称
Version:1.12.2                               #源码包软件的版本号
Release:    10                               #制作的RPM包版本号
Summary: Nginx is a web server software.     #RPM软件的概述    
License:GPL                                  #软件的协议
URL:    www.test.com                         #网址
Source0:nginx-1.12.2.tar.gz                  #源码包文件的全称

#BuildRequires:                              #制作RPM时的依赖关系
#Requires:                                   #安装RPM时的依赖关系
%description
nginx [engine x] is an HTTP and reverse proxy server.  #软件的详细描述

%post
useradd nginx                        #非必需操作：安装后脚本(创建账户)

%prep
%setup -q                            #自动解压源码包，并cd进入目录

%build
./configure
make %{?_smp_mflags}

%install
make install DESTDIR=%{buildroot}

%files
%doc
/usr/local/nginx/*                    #对哪些文件与目录打包

%changelog
```

### 步骤二：使用配置文件创建RPM包

1. 安装依赖包

```shell
[root@web1 ~]# yum -y install  gcc  pcre-devel openssl-devel
```

2. rpmbuild创建RPM软件包

```shell
[root@web1 ~]# rpmbuild -ba /root/rpmbuild/SPECS/nginx.spec
[root@web1 ~]# ls /root/rpmbuild/RPMS/x86_64/nginx-1.12.2-10.x86_64.rpm
```

### 步骤三：安装软件

```shell
[root@web1 ~]# yum install /root/rpmbuild/RPMS/x86_64/nginx-1.12.2-10.x86_64.rpm 
[root@web1 ~]# rpm -qa |grep nginx
[root@web1 ~]# ls /usr/local/nginx/
```

