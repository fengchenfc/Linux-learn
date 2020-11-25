# percona软件

## 常用的mysql备份工具

- 物理备份缺点
  - 跨平台性差
  - 备份时间长、冗余备份、浪费存储空间
- mysqldump备份缺点
  - 效率较低、备份和还原速度慢、锁表
  - 备份过程中、数据插入和更新操作被阻塞

## XtraBackup工具

- 一款强大的在线热备份工具
  - 备份过程中不锁库表、适合生产环境
  - 由专业组之percona提供（改进MySQL分支）
- 主要含两个组件
  - -xtrabackup: C程序，支持lnnoDB/XtraDB
  - innobackupex：以Perl脚本封装xtrabackup，还支持MySAM

## 安装软件包

- yum安装自动解决依赖

`– percona-xtrabackup-24-2.4.7-1.el7.x86_64.rpm`

```bash
]# rpm -ivh libev-4.15-1.el6.rf.x86_64.rpm 
]# yum -y  install percona-xtrabackup-24-2.4.7-1.el7.x86_64.rpm
```

## 查看安装信息

- 查看安装列表

```bash
[root@host62 ~]# rpm -ql percona-xtrabackup-24
/usr/bin/innobackupex  	//备份innodb 、xtrdb、myisam引擎的表
/usr/bin/xbcloud
/usr/bin/xbcloud_osenv
/usr/bin/xbcrypt
/usr/bin/xbstream
/usr/bin/xtrabackup     	//备份innodb 、xtrdb引擎的表
```

- 查看命令帮助

```bash
[root@host62 ~]# rpm -ql percona-xtrabackup-24
[root@host62 ~]# innobackupex  --help		//常用选项
[root@host62 ~]# man  innobackupex    	//详细帮助
```

# innobackupex备份与恢复

## innobackupex命令

### 常用选项

| **--host**         | **主机名**                             |
| ------------------ | -------------------------------------- |
| **--user**         | **用户名**                             |
| **--port**         | **端口号**                             |
| **--password**     | **密   码**                            |
| **--databases**    | **数据库名**                           |
| **--no-timestamp** | **不用日期命名备份文件存储的子目录名** |

> --databases="库名"      #一个库
>
> --databases="库1  库2"      #多个库
>
> --databases="库1.表"      #一张表

### 常用选项2

| **常用选项**                     | **含 义**                                        |
| -------------------------------- | ------------------------------------------------ |
| **--apply-log**                  | **准备恢复数据**                                 |
| **--copy-back**                  | **拷贝数据**                                     |
| **--incremental** **目录名**     | **增量备份**                                     |
| **--incremental-basedir=目录名** | **增量备份时，指定上一次备份数据存储的目录名**   |
| **--incremental-dir=目录名**     | **准备恢复数据时，指定增量备份数据存储的目录名** |
| **--export**                     | **导出表信息**                                   |
| **import**                       | **导入表空间**                                   |

****

## 完全备份与恢复

恢复数据的步骤：

1. 停止数据库服务
2. 清空数据库目录
3. 准备恢复数据
4. 拷贝数据库
5. 修改数据库目录的所有者和组用户为mysql
6. 启动服务
7. 查看数据库

### 完全备份

命令

```bash
#innobackupex    --user  用户名    --password  密码  备份目录名  --no-timestamp 
```

备份所有数据到/allbak目录

```bash
]# innobackupex --user  root  --password 123qqq...A  /allbak  --no-timestamp
```

### 完全恢复

命令

```bash
#innobackupex    --apply-log   目录名  //准备恢复数据 
#innobackupex    --copy-back  目录名  //恢复数据 
```

使用备份文件恢复数据

```bash
]# systemctl  stop  mysqld
]# rm -rf  /var/lib/mysql/*                            // 恢复时要求目录为空
]# cat /opt/allbak/xtrabackup_checkpoints     
]# innobackupex   --apply-log  /allbak        // 准备恢复数据
]# cat /opt/allbak/xtrabackup_checkpoints     
]# innobackupex  --copy-back  /allbak        // 恢复数据
]# chown  -R  mysql:mysql  /var/lib/mysql
]# systemctl  start  mysqld
```

### 恢复单张表

#### 操作步骤

具体操作如下：

- 删除表空间
- 导出表信息
- 拷贝表信息文件到数据库目录下
- 修改表信息文件的所有者及组用户为mysql
- 导入表空间
- 删除数据库目录下的表信息文件
- 查看表记录

#### 相关命令

```bash
mysql> alter  table  库名.表名  discard  tablespace;  
//删除表空间
]# innobackupex --apply-log --export   数据完全备份目录  
//导出表信息
]# cp  数据完全备份目录/数据库名目录/表名.{ibd,cfg,exp}  数据库目录/库名目录/    
//拷贝表信息文件
]# chown mysql:mysql   数据库目录/库名   
//修改所有者/组
mysql> alter  table  库名.表名   import  tablespace;  
//导入表空间
mysql> select  * from   库名.表名;  
//查看表记录
]#  rm  -rf   数据库目录/库名/表名.{cfg,exp}   
//删除表信息文件
```

## 增量备份与恢复

### 增量备份

- 增量备份时，必须先有一次备份,通常是完全备份

- 周一完全备份 ， 周二~周日增量备份

```bash
#innobackupex --user root --password   密码    /fullbak    --no-timestamp 	//完全备份
#innobackupex --user root --password   密码   --incremental  /new1dir    \
   --incremental-basedir=/fullbak   --no-timestamp  			             //增量备份
#innobackupex --user root --password   密码     --incremental  /new2dir    \
   --incremental-basedir=/new1dir   --no-timestamp  			             //增量备份
```

扩展：如果要做差异备份的话，每次在`--incremental-basedir=/...`写需要对比的源备份文件即可~

### 增量恢复

停服务

清空本机数据库目录

准备恢复数据   apply-log

合并数据

拷贝数据

更改目录属性（修改文件的所有者和组）

启动服务

```bash
# systemctl  stop mysqld
# rm  -rf  /var/lib/mysql/*
# innobackupex  --apply-log  --redo-only  /fullbak  	   //恢复完全备份数据
# innobackupex  --apply-log --redo-only  /fullbak   --incremental-dir=/new1dir 
//恢复增量数据
# innobackupex  --apply-log --redo-only  /fullbak   --incremental-dir=/new2dir 
//恢复增量数据
# innobackupex  --copy-back    /fullbak   		           //拷贝文件
# chown -R  mysql:mysql  /var/lib/mysql/
# systemctl  start mysqld 
```

