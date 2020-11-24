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





## 查看安装信息

查看安装列表



查看命令帮助



## innobackupex命令

常用选项

|      |      |
| ---- | ---- |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |

常用选项2

|      |      |
| ---- | ---- |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |

****







恢复数据的步骤：

1. 停止数据库服务
2. 清空数据库目录
3. 准备恢复数据
4. 拷贝数据库
5. 修改数据库目录的所有者和组用户为mysql
6. 启动服务
7. 查看数据库

innobackupex 完全备份与恢复   完全备份命令格式 ？ ]# innobackupex --user root --password A...qqq321 /allbak --no-timestamp ]# ls /allbak  ]# scp -r /allbak/ root@192.168.4.51:/opt/     完全恢复命令格式?   恢复数据的步骤：如下  1 停止数据库服务  2 清空数据库目录  3 准备恢复数据  4 拷贝数据库  5 修改数据库目录的所有者和组用户为mysql  6 启动服务  7 查看数据库

  71 systemctl stop mysqld  

 73 rm -rf /var/lib/mysql/* 

  76 cat /opt/allbak/xtrabackup_checkpoints

​    77 innobackupex --apply-log /opt/allbak/

   78 cat /opt/allbak/xtrabackup_checkpoints    

79 innobackupex --copy-back /opt/allbak/  

   82 chown -R mysql:mysql /var/lib/mysql  

83 systemctl start mysqld  

  84 mysql -uroot -pA...qqq321