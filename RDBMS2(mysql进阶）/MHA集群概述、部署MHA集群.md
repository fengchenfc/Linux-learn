# MHA集群概述

## MHA介绍
### MHA简介

- MHA（Master High Availability）

由日本DeNA狗狗尼斯youshimaton开发

是一套优秀的实现MySQL高可用的解决方案

数据库的自动故障切换操作能做到0~30秒之内完成

MHA能确保在故障切换过程中最大限度保证数据的一致性，以达到真正意义上的高可用

### MHA组成

- MHA Manager（管理节点）

管理所有数据库服务器

可以单独部署在一台独立的机器上

也可以部署在某台数据库服务器上

- MHA Node（数据节点）

存储数据的MySQL服务器

运行在每台MySQL服务器上 



## MHA工作过程

1. 由manager定时探测集群中的master节点
2. 当master故障时，manager自动给你将拥有最新数据的slave提升为新的master



- MHA集群架构

![](E:\fc-learn\Linux-learn\RDBMS2(mysql进阶）\MHA集群概述、部署MHA集群.assets\LINUXNSD_V01RDBMS2DAY04_008.png)



# 部署MHA集群

## IP规划

|      |      |      |      |
| ---- | ---- | ---- | ---- |
|      |      |      |      |
|      |      |      |      |
|      |      |      |      |
|      |      |      |      |
|      |      |      |      |
|      |      |      |      |

## 拓扑图

![img](E:\fc-learn\Linux-learn\RDBMS2(mysql进阶）\MHA集群概述、部署MHA集群.assets\LINUXNSD_V01RDBMS2DAY04_012.png)

