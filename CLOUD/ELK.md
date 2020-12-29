# ELK 日志分析系统

- Elasticsearch

负责日志检索和存储

- Logstash

负责日志的收集和分析、处理

- Kibana

负责日志的可视化

**ELK是一整套解决方案，是三个软件产品的首字母缩写**

**ELK三款软件都是开源软件**

**ELK归于Elastic.co公司名下**

```Mermaid
graph LR
subgraph web cluster
  subgraph web1
    H1[apache] --> F1(filebeat)
  end
  subgraph web2
    H2[apache] --> F2(filebeat)
  end
  subgraph web3
    H3[apache] --> F3(filebeat)
  end
  style web1 color:#ff0000,fill:#99ff99
  style web2 color:#ff0000,fill:#99ff99
  style web3 color:#ff0000,fill:#99ff99
end
F1 --> A1
F2 --> A1
F3 --> A1
subgraph Logstash
  A1(input) --> A2(filter) --> A3(output)
end
subgraph ES Cluster
  ES1(Elasticsearch)  
  ES2(Elasticsearch)
  ES3(Elasticsearch)
  ES4(Elasticsearch)
  ES5(Elasticsearch)
end
A3 --> ES1
A3 --> ES2
A3 --> ES3
A3 --> ES4
A3 --> ES5
ES1 --> K(kibana)
ES2 --> K
ES3 --> K
ES4 --> K
ES5 --> K
```

ELK组件在海量日志系统的运维中，可用于解决

- 分布式日志数据集中式查询和管理
- 系统监控，包含系统硬件和应用各个组件的监控
- 故障排查
- 安全信息和事件管理
- 报表功能



# Elasticsearch

Elasticsearch 基于Lucene的搜索服务器，提供了一个分布式多用户能力的全文搜索引擎，基于RHSTfulAPI的Web接口

Elastic search 用==java==开发的，使用apache许可条款的开源软件，是企业级搜索引擎。

设计用于云计算中，能达到实时搜索，稳定，可靠，快速。

安装使用方便

- 主要特点

实时分析

分布式实时文件存储，并将每一个字段都编入索引

文档导向，所有的对象全部是文档

高可用性、易扩展、支持集群（Cluster）、分片和复制（Shards和Replicas）

接口友好，支持JSON

- 缺点

没有典型意义的事务

是一种面向文档的数据库

没有提供授权和认证特性

- 相关概念

| Node     | 装有一个ES服务器的节点              |
| -------- | ----------------------------------- |
| Cluster  | 有多个Node组成的集群                |
| Document | 一个可被搜索的基础信息单元          |
| Index    | 拥有相似特征的文档的集合            |
| Type     | 一个索引中可以定义一种/多种类型     |
| Filed    | 是ES的最小单位，相当于数据的某一列  |
| Shards   | 索引的分片，每一个分片就是一个shard |
| Replicas | 索引的拷贝                          |

- 与关系型数据库对比

| Relational database    | Elasticsearch         |
| ---------------------- | --------------------- |
| Database               | Index                 |
| Table                  | Type                  |
| Row                    | Document              |
| Column                 | Field                 |
| Schema                 | Mapping               |
| Index                  | Everything is indexed |
| SQL                    | Query DSL             |
| SELECT * FROM table... | GET http://...        |
| UPDATE table SET       | PUT http://...        |

## Elasticsearch 安装

> ELK安装软件列表
>
> - CentOS7-1804.iso	系统软件仓库
> - elasticsearch-2.3.4.rpm
> - filebesat-1.2.3-x86_64.rpm
> - kibana-4.5.2-1.x86_64.rpm
> - logstash-2.3.4-1.noarch.rpm

