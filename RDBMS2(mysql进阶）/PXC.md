# PXC

> 配置PXC集群，具体步骤如下：
>
> 1. 安装软件（所有集群主机都要装）
>
> 2. 修改配置文件
>
>    mysqld.cnf  wsrep.cnf
>
> 3. 在任意一台服务器做集群初始化配置(且只做一次)
>
> 4. 在另外两台服务器启动mysql服务
>
> 5. 在任意一台服务器，查看集群状态
>
> 6. 客户端连接集群中的任意一台主机，访问呢数据库服务
>
> 7. 测试高可用功能

## PXC概述

### PXC介绍

- Percona XtraDB Cluster （简称PXC）

基于Galera的MySQL高可用集群解决方案

### PXC特点



### 相应端口



### 主机角色



## 部署PXC

### 安装软件

- 软件介绍

percona-xtrabackup-24-2.4.13-11.el7.x86_64.rpm       #在线热备程序

qpress-1.1-14.11.x86_64.rpm       #递归压缩程序

Perconna-XtraDB-Cluster-server-57-5.7.25-31.35.1.el7.x86_64.rpm        #集群服务程序