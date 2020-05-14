---
title: Hadoop HA 搭建
top: false
cover: false
toc: true
mathjax: false
date: 2020-05-06 20:30:35
password:
summary:
tags:
categories:
img:
keywords:
---

所谓HA（High Available）是Hadoop2.0中引入来解决Hadoop1.0中单点故障问题的一种机制。HA严格来说应该分成各个组件的HA机制：HDFS的HA和YARN的HA。

## HDFS-HA集群配置

### 规划集群

| master            | slave1                  | slave2                   |
| ----------------- | ----------------------- | ------------------------ |
| NameNode (active) | NameNode (standby)      |                          |
| JournalNode       | JournalNode             | JournalNode              |
| DataNode          | DataNode                | DataNode                 |
| ZK                | ZK                      | ZK                       |
|                   | ResourceManager(active) | ResourceManager(standby) |
| NodeManager       | NodeManager             | NodeManager              |

### 配置Zookeeper集群

在master、slave1和slave2三个节点上部署Zookeeper

### 安装zookeeper

解压zookeeper，并重命名为zookeeper

```
tar -zxvf apache-zookeeper-3.5.7-bin.tar.gz -C /usr/local/
cd /usr/local
mv apache-zookeeper-3.5.7-bin/ zookeeper
```

### 配置zookeeper

* 在`/usr/local/zookeeper/`这个目录下创建zkData

```
cd /usr/local/zookeeper
mkdir zkData
```

* 重命名`/usr/local/zookeeper/conf`这个目录下的zoo_sample.cfg为zoo.cfg

```
cd /usr/local/zookeeper/conf
mv zoo_sample.cfg zoo.cfg
```

* 执行`sudo gedit zoo.cfg`打开配置文件，配置如下：

```
# 修改原有dataDir值如下
dataDir=/usr/local/zookeeper/zkData
#######################cluster##########################
server.1=master:2888:3888
server.2=slave1:2888:3888
server.3=slave2:2888:3888
```

Server.A=B:C:D。

A是一个数字，表示这个是第几号服务器；

B是这个服务器的IP地址；

C是这个服务器与集群中的Leader服务器交换信息的端口；

D是万一集群中的Leader服务器挂了，需要一个端口来重新进行选举，选出一个新的Leader，而这个端口就是用来执行选举时服务器相互通信的端口。

集群模式下配置一个文件myid，这个文件在dataDir目录下，这个文件里面有一个数据就是A的值，Zookeeper启动时读取此文件，拿到里面的数据与zoo.cfg里面的配置信息比较从而判断到底是哪个server。

* 在/usr/local/zookeeper/zkData目录下创建并打开文件myid

```
gedit myid
```

在myid文件中增加数据：1

* 拷贝配置好的zookeeper到其他机器上,并分别修改myid文件中内容为2、3

```
scp -r /usr/local/zookeeper/ slave1:/usr/local/
scp -r /usr/local/zookeeper/ slave2:/usr/local/
```

* 配置zookeeper环境变量

执行`sudo gedit /etc/profile`打开配置文件，加入下列内容

```
export ZOOKEEPER_HOME=/usr/local/zookeeper
export PATH=$ZOOKEEPER_HOME/bin:$PATH
```

执行`source /etc/profile`命令让配置文件生效

* 拷贝`/etc/profile`文件到其他机器

```
scp /etc/profile slave1:/etc
```



### 启动zookeeper

分别启动zookeeper,在三台机器上执行`zkServer.sh start`命令启动zookeeper,再执行`jps`命令可以看到有`QuorumPeerMain`

## 配置HDFS-HA集群

* 配置core-site.xml：

```
<configuration>
	<!-- 把两个NameNode的地址组装成一个集群mycluster -->
	<property>
		<name>fs.defaultFS</name>
        <value>hdfs://mycluster</value>
	</property>

	<!-- 指定hadoop运行时产生文件的存储目录 -->
	<property>
		<name>hadoop.tmp.dir</name>
		<value>/usr/local/hadoop/tmp</value>
	</property>
	<!-- 在网页界面访问数据使用的用户名。默认值是一个不真实存在的用户，此用户权限很小 -->
	<property>
        <name>hadoop.http.staticuser.user</name>
        <value>ubuntu</value>
	</property> 
	<!--Ha功能，需要一组zk地址，用逗号分隔。被ZKFailoverController使用于自动失效备援failover-->
	<property>
   		<name>ha.zookeeper.quorum</name>
  	    <value>master:2181,slave1:2181,slave2:2181</value>
 	</property>
</configuration>
```

* 配置hdfs-site.xml

```
<configuration>
	<!-- 冗余度 -->
	<property>
  		<name>dfs.replication</name>
  		<value>3</value>
	</property>
	<!-- 暂不配置 
	<property>
  		<name>dfs.namenode.secondary.http-address</name>
  		<value>slave1:9869</value>
	</property>
	-->
	<!-- 完全分布式集群名称 -->
	<property>
		<name>dfs.nameservices</name>
		<value>mycluster</value>
	</property>

	<!-- 集群中NameNode节点都有哪些 -->
	<property>
		<name>dfs.ha.namenodes.mycluster</name>
		<value>nn1,nn2</value>
	</property>

	<!-- nn1的RPC通信地址 -->
	<property>
		<name>dfs.namenode.rpc-address.mycluster.nn1</name>
		<value>master:8020</value>
	</property>

	<!-- nn2的RPC通信地址 -->
	<property>
		<name>dfs.namenode.rpc-address.mycluster.nn2</name>
		<value>slave1:8020</value>
	</property>

	<!-- nn1的http通信地址 -->
	<property>
		<name>dfs.namenode.http-address.mycluster.nn1</name>
		<value>master:9870</value>
	</property>

	<!-- nn2的http通信地址 -->
	<property>
		<name>dfs.namenode.http-address.mycluster.nn2</name>
		<value>slave1:9870</value>
	</property>

	<!-- 指定NameNode元数据在JournalNode上的存放位置 -->
	<property>
		<name>dfs.namenode.shared.edits.dir</name>
	<value>qjournal://master:8485;slave1:8485;slave2:8485/mycluster</value>
	</property>

	<!-- 配置隔离机制，即同一时刻只能有一台服务器对外响应 -->
	<property>
		<name>dfs.ha.fencing.methods</name>
		<value>sshfence</value>
	</property>

	<!-- 使用隔离机制时需要ssh无秘钥登录-->
	<property>
		<name>dfs.ha.fencing.ssh.private-key-files</name>
		<value>/home/ubuntu/.ssh/id_rsa</value>
	</property>

	<!-- 声明journalnode服务器存储目录-->
	<property>
		<name>dfs.journalnode.edits.dir</name>
		<value>/usr/local/hadoop/tmp/jn</value>
	</property>

	<!-- 关闭权限检查-->
	<property>
		<name>dfs.permissions.enable</name>
		<value>false</value>
	</property>
	
	<!-- 是否开启自动故障转移-->
	<property>
   		<name>dfs.ha.automatic-failover.enabled</name>
   		<value>true</value>
	</property>
	
	<!--访问代理类：client，mycluster，active配置失败自动切换实现方式-->
	<property>
  		<name>dfs.client.failover.proxy.provider.mycluster</name>
	<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	</property>
</configuration>
```

* 配置mapred-site.xml

```
<configuration>
   <property>
       <name>mapreduce.framework.name</name>
       <value>yarn</value>
   </property>
</configuration>
```

配置yarn-site.xml

```
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <!--启用resourcemanager ha-->
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
 
    <!--声明两台resourcemanager的地址-->
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>cluster-yarn1</value>
    </property>

    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>

    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>slave1</value>
    </property>

    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>slave2</value>
    </property>
 
    <!--指定zookeeper集群的地址--> 
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <value>master:2181,slave1:2181,slave2:2181</value>
    </property>

    <!--启用自动恢复--> 
    <property>
        <name>yarn.resourcemanager.recovery.enabled</name>
        <value>true</value>
    </property>
 
    <!--指定resourcemanager的状态信息存储在zookeeper集群--> 
    <property>
	<name>yarn.resourcemanager.store.class</name>
<value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
    </property>

</configuration>
```

* 拷贝配置好的Hadoop文件到其他机器上

```
scp -r /usr/local/hadoop/etc/hadoop/ slave1:/usr/local/hadoop/etc/
scp -r /usr/local/hadoop/etc/hadoop/ slave2:/usr/local/hadoop/etc/
```

## 启动

### 启动zookeeper集群

分别在master，slave1，slave2上执行`zkServer.sh start`命名启动zk

### 启动journalnode

分别在master，slave1，slave2上执行`hdfs --daemon start journalnode`命令启动JN

运行jps命令检验，master，slave1，slave2上多了JournalNode进程

### 格式化namenode，并启动

在master上执行命令`hdfs namenode -format`格式化namenode

在master上执行命令`hdfs --daemon start namenode`命令格式化namenode

### 副节点同步主节点格式化

在slave1上执行命令`hdfs namenode -bootstrapStandby`命令同步主节点格式化

### 格式化ZKFC

在master上执行命令`hdfs zkfc -formatZK`格式化ZKFC，第一次启动时需要格式化

### 启动HDFS

在master上执行命令`start-dfs.sh`启动HDFS

### 启动YARN

在slave1上执行`start-yarn.sh`命令启动YARN，把namenode和resourcemanager分开是因为性能问题，因为他们都要占用大量资源，所以把他们分开了，他们分开了就要分别在不同的机器上启动

## 测试集群工作状态的一些指令 

* `hdfs dfsadmin -report` 查看hdfs的各节点状态信息

* `hdfs haadmin -getServiceState nn1` 获取一个namenode节点的HA状态
* `hdfs haadmin -transitionToActive nn1`将nn1切换为Active
* `hdfs haadmin -transitionToStandby nn1`将nn1切换为Standby
* `hdfs haadmin -failover nn1 nn2`主备切换
* `yarn rmadmin -getServiceState rm1`获取一个resourcemanager节点的HA状态
* `hdfs --daemon start namenode` 单独启动一个namenode进程
* `hdfs --daemon start zkfc` 单独启动一个zkfc进程

## 测试集群高可用

执行`hadoop fs -put /etc/profile /profile`上传一个文件到hdfs

执行`hadoop fs -ls /`可查看hdfs下的文件 ：

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200507183228.png)

执行`hdfs haadmin -getServiceState nn1`和``hdfs haadmin -getServiceState nn2`命令查master和slave1上NameNode的状态

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200507191108.png)

在master上执行`jps`命令查看NameNode进程ID，再执行`kill -p <pid of NameNode>`关掉Active状态的NameNode，再执行`jps`会发现没有NameNode了,最后执行`hdfs haadmin -getServiceState nn2`会发现slave1的NameNode已经是active状态了

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200507191904.png)

执行`hadoop fs -ls /`会发现仍然能访问到hdfs的文件

在master上执行`hdfs --daemon start namenode`重新启动NameNode，再执行`hdfs haadmin -getServiceState nn1`会发现master上NameNode的状态变为了standby

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200507192833.png)

执行`hdfs haadmin -failover nn2 nn1`会主备切换，nn1变为active，nn2变为standby.

## 群起脚本

```bash
#!/bin/bash
if [ $# -lt 1 ]
 then 
   echo "No Args Input Error!!!!!"
   exit
fi
case $1 in 
"start")
   	echo "======================== start zookeeper ========================== "
	for i in master slave1 slave2
	do
   		echo "========== $i zookeeper =========="
   		ssh $i "source /etc/profile;zkServer.sh start"
	done
	echo "======================== start hdfs ========================== "
  	ssh master "source /etc/profile;start-dfs.sh"
   	echo "======================== start yarn ========================== "
   	ssh slave1 "source /etc/profile;start-yarn.sh"
;;
"stop")
	echo "======================== stop yarn ========================== "
   	ssh slave1 "source /etc/profile;stop-yarn.sh"
   	echo "======================== stop hdfs ========================== "
  	ssh master "source /etc/profile;stop-dfs.sh"
  	echo "======================== stop zookeeper ========================== "
	for i in master slave1 slave2
	do
   		echo "========== $i zookeeper =========="
   		ssh $i "source /etc/profile;zkServer.sh stop"
	done
;;
*)
  	echo "Input Args Error!!!!!"
;;
esac
```

