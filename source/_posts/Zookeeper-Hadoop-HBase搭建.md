---
title: Zookeeper+Hadoop+HBase搭建
top: false
cover: false
toc: true
mathjax: false
date: 2020-05-07 21:54:26
password:
summary:
categories: 大数据
img:
keywords: hadoop zookeeper hbase 大数据
tags:
	- hadoop
	- zookeeper
	- hbase
	- 大数据
---

[Hadoop搭建](https://zhishuang.tk/2020/04/30/ubuntu1604-da-jian-hadoop-ji-qun/)

[Zookeeper+Hadoopda搭建（Hadoop HA）](https://zhishuang.tk/2020/05/06/hadoop-ha-da-jian/)

这儿实现在Hadoop HA的基础上搭建Hbase

## 安装Hbase

将安装文件`hbase-2.2.4-bin.tar.gz`到`/usr/local`并重命名为`hbase`

```bash
tar -zxvf hbase-2.2.4-bin.tar.gz -C /usr/local
cd /usr/local
mv -r hbase-2.2.4 hbase
```

## 配置Hbase

**环境变量**，执行`sudo gedit /etc/profile`打开配置文件，添加如下内容

```bash
#set hbase env
export HBASE_HOME=/usr/local/hbase
export PATH=$HBASE_HOME/bin:$PATH
```

记得所有主机都要配置，执行`source /etc/profile`使配置生效

**hbase-env.sh**

```bash
export JAVA_HOME=/usr/java/jdk1.8.0_251
export HADOOP_HOME=/usr/local/hadoop
export HBASE_HOME=/usr/local/hbase
#关闭自身zookeeper，采用外部的zookeeper
export HBASE_MANAGES_ZK=false
```

**hbase-site.xml**

```xml
<configuration>
	<!-- hadoop集群名称 -->
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://mycluster/hbase</value>
    </property>
    <property>
        <name>hbase.zookeeper.quorum</name>
        <value>master,slave1,slave2</value>
    </property>
    <property>
        <name>hbase.zookeeper.property.clientPort</name>
        <value>2181</value>
    </property>
    <!--  是否是完全分布式 -->
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <!--  完全分布式式必须为false  -->
    <property>
        <name>hbase.unsafe.stream.capability.enforce</name>
        <value>false</value>
    </property>
    <!--  指定缓存文件存储的路径 -->
    <property>
        <name>hbase.tmp.dir</name>
        <value>/usr/local/hadoop/tmp</value>
    </property>
    <!--  指定Zookeeper数据存储的路径  -->
    <property>
    	<name>hbase.zookeeper.property.dataDir</name>
    	<value>/usr/local/zookeeper/zkData</value>
    </property>
</configuration>
```

**regionservers**

```bash
master
slave1
slave2
```

**配置Hmaster高可用**

为了保证HBase集群的高可靠性，HBase支持多Backup Master 设置。当Active Master挂掉后，Backup Master可以自动接管整个HBase的集群。该配置极其简单：在 $HBASE_HOME/conf/目录下新增文件配置backup-masters，在其内添加要用做Backup Master的节点hostname。
		执行`gedit /usr/local/hbase/conf/backup-masters`新建并打开文件，添加`slave2`
		没设置backup-masters之前启动hbase， 只有一台有启动了HMaster进程，设置之后，重新启动整个集群，我们会发现，在backup-masters清单上的主机，都启动了HMaster进程

**分发hbase给其他主机**

```bash
scp -r /usr/local/hbase/ slave1:/usr/local/
scp -r /usr/local/hbase/ slave1:/usr/local/
```

## 时间同步

执行`sudo apt-get install ntp`安装ntp

配置ntp，执行`gedit /etc/ntp.conf`打开配置文件

* master端配置

```bash
# /etc/ntp.conf, configuration for ntpd; see ntp.conf(5) for help
# 时间差异文件
driftfile /var/lib/ntp/ntp.drift

# 分析统计信息
#statsdir /var/log/ntpstats/

statistics loopstats peerstats clockstats
filegen loopstats file loopstats type day enable
filegen peerstats file peerstats type day enable
filegen clockstats file clockstats type day enable

# 上层ntp server.
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 2.ubuntu.pool.ntp.org iburst
pool 3.ubuntu.pool.ntp.org iburst

# Use Ubuntu's ntp server as a fallback.
pool ntp.ubuntu.com

# 不允许来自公网上ipv4和ipv6客户端的访问
restrict -4 default kod notrap nomodify nopeer noquery limited
restrict -6 default kod notrap nomodify nopeer noquery limited

# 让NTP Server和其自身保持同步，如果在/etc/ntp.conf中定义的server都不可用时，将使用local时间作为ntp服务提供给ntp客户端.
restrict 127.0.0.1
restrict ::1

# Needed for adding pool entries
restrict source notrap nomodify noquery

# 允许这个网段的对时请求.
restrict 192.168.79.0 mask 255.255.255.0 nomodify 

# If you want to provide time to your local subnet, change the next line.
# (Again, the address is an example only.)
#broadcast 192.168.123.255

# If you want to listen to time broadcasts on your local subnet, de-comment the
# next lines.  Please do this only if you trust everybody on the network!
#disable auth
#broadcastclient

#Changes recquired to use pps synchonisation as explained in documentation:
#http://www.ntp.org/ntpfaq/NTP-s-config-adv.htm#AEN3918

#server 127.127.8.1 mode 135 prefer    # Meinberg GPS167 with PPS
#fudge 127.127.8.1 time1 0.0042        # relative to PPS for my hardware

#server 127.127.22.1                   # ATOM(PPS)
#fudge 127.127.22.1 flag3 1            # enable PPS API
```

* slave端配置

```bash
# /etc/ntp.conf, configuration for ntpd; see ntp.conf(5) for help
# 时间差异文件
driftfile /var/lib/ntp/ntp.drift

# 分析统计信息
#statsdir /var/log/ntpstats/

statistics loopstats peerstats clockstats
filegen loopstats file loopstats type day enable
filegen peerstats file peerstats type day enable
filegen clockstats file clockstats type day enable

# 上层ntp server.
# pool 0.ubuntu.pool.ntp.org iburst
# pool 1.ubuntu.pool.ntp.org iburst
# pool 2.ubuntu.pool.ntp.org iburst
# pool 3.ubuntu.pool.ntp.org iburst
server 192.168.79.129
# Use Ubuntu's ntp server as a fallback.
# pool ntp.ubuntu.com

# 不允许来自公网上ipv4和ipv6客户端的访问
restrict -4 default kod notrap nomodify nopeer noquery limited
restrict -6 default kod notrap nomodify nopeer noquery limited

# 让NTP Server和其自身保持同步，如果在/etc/ntp.conf中定义的server都不可用时，将使用local时间作为ntp服务提供给ntp客户端.
restrict 127.0.0.1
restrict ::1

# Needed for adding pool entries
restrict source notrap nomodify noquery

# 允许这个网段的对时请求.
# restrict 192.168.79.0 mask 255.255.255.0 nomodify 

# If you want to provide time to your local subnet, change the next line.
# (Again, the address is an example only.)
#broadcast 192.168.123.255

# If you want to listen to time broadcasts on your local subnet, de-comment the
# next lines.  Please do this only if you trust everybody on the network!
#disable auth
#broadcastclient

#Changes recquired to use pps synchonisation as explained in documentation:
#http://www.ntp.org/ntpfaq/NTP-s-config-adv.htm#AEN3918

#server 127.127.8.1 mode 135 prefer    # Meinberg GPS167 with PPS
#fudge 127.127.8.1 time1 0.0042        # relative to PPS for my hardware

#server 127.127.22.1                   # ATOM(PPS)
#fudge 127.127.22.1 flag3 1            # enable PPS API
```



查看ntp的时间服务是否启动：`ps -aux | grep ntp`

执行 `service ntp restart`，重启ntp服务

执行`ntpq -p`查看配置

这个命令可以列出目前我们的 NTP 与相关的上层 NTP 的状态，上头的几个字段的意义为：

* remote: 它指的就是本地机器所连接的远程NTP服务器；
* refid: 它指的是给远程服务器提供时间同步的服务器；
* st: 远程服务器的层级别（stratum）. 由于NTP是层型结构,有顶端的服务器,多层的Relay Server再到客户端。所以服务器从高到低级别可以设定为1-16. 为了减缓负荷和网络堵塞,原则上应该避免直接连接到级别为1的服务器的；
* when: 几秒钟前曾经做过时间同步化更新的动作；
* poll: 本地机和远程服务器多少时间进行一次同步(单位为秒).在一开始运行NTP的时候这个poll值会比较小,那样和服务器同步的频率也就增加了,可以尽快调整到正确的时间范围.之后poll值会逐渐增大,同步的频率也就会相应减小；
* reach: 已经向上层 NTP 服务器要求更新的次数；
* delay: 网络传输过程当中延迟的时间，单位为 10^(-6) 秒；
* offset: 时间补偿的结果，单位与 10^(-3) 秒；
* jitter: Linux 系统时间与 BIOS 硬件时间的差异时间， 单为 10^(-6) 秒。简单地说这个数值的绝对值越小我们和服务器的时间就越精确；
* *: 它告诉我们远端的服务器已经被确认为我们的主NTP Server,我们系统的时间将由这台机器所提供；
* +: 它将作为辅助的NTP Server和带有号的服务器一起为我们提供同步服务. 当号服务器不可用时它就可以接管；
* -: 远程服务器被clustering algorithm认为是不合格的NTP Server；
* x: 远程服务器不可用

## 启动HBase

首先启动Hadoop-HA集群，再执行`start-hbase.start`启动HBase，执行jps命令，可以看到：

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200509141401.png)

从`http://master:16010/`可查看HBase集群信息

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200509141902.png)

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
   	echo "======================== start hbase ========================== "
   	ssh master "source /etc/profile;start-hbase.sh"
;;
"stop")
	echo "======================== stop hbase ========================== "
  	ssh master "source /etc/profile;stop-hbase.sh"
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

