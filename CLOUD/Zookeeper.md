# Zookeeper

## 概述

- zookeeper是什么

开源的分布式应用程序协调服务

- zookeeper能做什么

用来保证数据在集群间的事务一致性

- zookeeper应用场景

   - 集群分布式锁

  - 集群统一命名服务

  - 分布式协调服务







4、配置 hdfs-site.xml

```bash
[root@hadoop1 ~]# vim /usr/local/hadoop/etc/hadoop/hdfs-site.xml
<configuration>
    <property>
        <name>dfs.nameservices</name>
        <value>mycluster</value>
    </property>
#定义组名mycluster
    <property>
        <name>dfs.ha.namenodes.mycluster</name>
        <value>nn1,nn2</value>
    </property>
#定义两个角色:nn1,nn2,在mycluster组中
    <property>
        <name>dfs.namenode.rpc-address.mycluster.nn1</name>
        <value>hadoop1:8020</value>
    </property>
#定义组名角色nn1，对应hadoop1:8020
    <property>
        <name>dfs.namenode.rpc-address.mycluster.nn2</name>
        <value>hadoop2:8020</value>
    </property>
##定义组名角色nn2，对应hadoop2:8020
    <property>
        <name>dfs.namenode.http-address.mycluster.nn1</name>
        <value>hadoop1:50070</value>
    </property>
#定义组名角色nn1，对应Hadoop1：50070
    <property>
        <name>dfs.namenode.http-address.mycluster.nn2</name>
        <value>hadoop2:50070</value>
    </property>
#定义组名角色nn2，对应Hadoop2：50070
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://node-0001:8485;node-0002:8485;node-0003:8485/mycluster</value>
    </property>
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/var/hadoop/journal</value>
    </property>
    <property>
        <name>dfs.client.failover.proxy.provider.mycluster</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
    <property> 
        <name>dfs.ha.fencing.methods</name>
        <value>sshfence</value>
    </property>
#定义高可用免密登录方式
    <property>
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/root/.ssh/id_rsa</value>
    </property>
#指定高可用私钥路径
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
    <property>
        <name>dfs.hosts.exclude</name>
        <value>/usr/local/hadoop/etc/hadoop/exclude</value>
    </property>
</configuration>
```



6、配置 yare-site.xml

```bash
[root@hadoop1 ~]# vim /usr/local/hadoop/etc/hadoop/yarn-site.xml
<configuration>
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
# 激活HA配置
    <property>
        <name>yarn.resourcemanager.recovery.enabled</name>
        <value>true</value>
    </property>
# 管理节点状态自动恢复
    <property>
        <name>yarn.resourcemanager.store.class</name>
        <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
    </property>
# 数据状态保持介质
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <value>node-0001:2181,node-0002:2181,node-0003:2181</value>
    </property>
#配置zookeeper地址
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>yarn-ha</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>
#定义rm1，rm2两个角色
    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>hadoop1</value>
    </property>
#定义rm1对应主机hadoop1
    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>hadoop2</value>
    </property>
#定义rm2对应主机hadoop2
<!-- Site specific YARN configuration properties -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```