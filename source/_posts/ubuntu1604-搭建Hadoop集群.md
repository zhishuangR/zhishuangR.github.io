---
title: ubuntu1604 搭建Hadoop集群
top: false
cover: false
toc: true
mathjax: false
date: 2020-04-30 16:28:34
password:
summary:
categories: 大数据
img:
keywords: hadoop 集群
tags:
	- hadoop
	- 集群
---

## 系统环境

| 名称   | 版本                                                         |
| ------ | ------------------------------------------------------------ |
| Ununtu | 1604                                                         |
| JDK    | [JDK8](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html) |
| Hadoop | [3.1.3](https://www.apache.org/dyn/closer.cgi/hadoop/common/hadoop-3.1.3/hadoop-3.1.3.tar.gz) |

下载JDK8和Hadoop3.1.3，放在ubuntu里Downloads文件夹下

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430165326.png)

## 基本配置

### 设置静态IP

[ubuntu1604 设置静态IP](https://zhishuang.tk/2020/04/30/ubuntu1604-she-zhi-jing-tai-ip/)

### 修改hostname

执行命令`sudo gedit /etc/hostname`,修改hostname为master

### 配置主机映射

执行命令`ifconfig`获取IP地址

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200504220459.png)

执行命令`sudo gedit /etc/hosts`,添加映射关系，注意IP地址换成自己的

```
192.168.79.129	master
```

### 关闭防火墙

执行命名`sudo ufw disable`关闭防火墙

### 配置SSH

>Hadoop名称节点(NameNode)需要启动集群中所有机器的Hadoop守护进程，这个过 程需要通过SSH登录来实现。Hadoop并没有提供SSH输入密码登录的形式，因此，为 了能够顺利登录每台机器，需要将所有机器配置为名称节点可以无密码登录它们。

#### 配置SSH的无密码登录

安装openssh-server( 通常Linux系统会默认安装openssh的客户端软件openssh-client)，所以需要自己安装一下服务端。

```
sudo apt-get install openssh-server
```

在终端执行命令`ssh-keygen -t rsa`，然后一直回车，生成密钥。

执行命令`ssh-copy-id localhost`，在提示信息中输入`yes`，然后输入本机密码

执行命令`ssh localhost`，出现登录时间，说明ssh配置成功

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200504234742.png)

## 配置JAVA环境

### 创建文件夹

```bash
sudo mkdir /usr/java
```

在/usr目录下创建文件夹java，我们将把JDK8解压到这个文件夹中

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430170954.png)

### 解压JDK

```bash
sudo tar -zxvf ~/Downloads/jdk-8u251-linux-x64.tar.gz -C /usr/java
```

解压jdk8到/usr/java目录下

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430171614.png)

### 配置环境变量

执行`sudo gedit /etc/profile`命令打开配置文件，在末尾添加以下几行文字，注意自己的jdk版本号。

```
#set java env
export JAVA_HOME=/usr/java/jdk1.8.0_251
export JRE_HOME=${JAVA_HOME}/jre    
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib    
export PATH=${JAVA_HOME}/bin:$PATH
```

执行`source /etc/profile`命令让配置文件生效

执行`java -version`命令验证是否配置成功

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430173142.png)

## 配置Hadoop环境

### 解压Hadoop

将我们下载的Hadoop解压到 /usr/local/ 中

```
sudo tar -zxvf ~/Downloads/hadoop-3.1.3.tar.gz -C /usr/local
```

利用`cd /usr/local/`命令切换操作空间，将文件夹名改为hadoop

```
sudo mv ./hadoop-3.1.3/ ./hadoop
```

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430174115.png)

### 配置Hadoop环境变量

执行`sudo gedit /etc/profile`命令打开配置文件,添加下面的语句

```
#set Hadoop env
HADOOP_HOME=/usr/local/hadoop
export PATH=${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:$PATH
```

执行`source /etc/profile`命令让配置文件生效

### 修改Hadoop配置文件

执行`sudo gedit hadoop-env.sh`命令打开配置文件，添加语句`export JAVA_HOME=/usr/java/jdk1.8.0_251`

执行`hadoop version`命令验证是否配置成功

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430183459.png)

到这儿 我们Hadoop的单机模式配置完成

## Hadoop伪分布式安装配置

>Hadoop可以在单节点上以伪分布式的方式运行，Hadoop进程以分离的 Java 进程来运行，节点既作为 NameNode也作为DataNode， 同时，读取的是 HDFS 中的文件
>
>Hadoop的配置文件位于$HADOOP_HOME/etc/hadoop/中，伪分布式需要修改2个配置文件 core-site.xml 和 hdfs-site.xml
>
>Hadoop的配置文件是xml格式，每个配置以声明property的name和value的方式来实现

### hadoop 目录说明

>修改配置文件之前，先看一下hadoop下的目录：
>
>- bin：hadoop最基本的管理脚本和使用脚本所在目录，这些脚本是sbin目录下管理脚本的基础实现，用户可以直接使用这些脚本管理和使用hadoop
>- etc：配置文件存放的目录，包括core-site.xml,hdfs-site.xml,mapred-site.xml等从hadoop1.x继承而来的配置文件和yarn-site.xml等hadoop2.x新增的配置文件
>- include：对外提供的编程库头文件（具体动态库和静态库在lib目录中，这些头文件军事用c++定义的，通常用于c++程序访问hdfs或者编写mapreduce程序）
>- Lib：该目录包含了hadoop对外提供的才变成动态库和静态库，与include目录中的头文件结合使用
>- libexec：各个服务对应的shell配置文件所在目录，可用于配置日志输出目录、启动参数等信息
>- sbin：hadoop管理脚本所在目录，主要包含hdfs和yarn中各类服务的启动、关闭脚本
>- share：hadoop各个模块编译后的jar包所在目录。

### 修改配置文件	

在 `/usr/local/hadoop/etc/hadoop/`目录下，执行命令`sudo gedit core-site.xml`，修改配置文件core-site.xml，内容如下：

```
<configuration> 
 <property>
      <name>hadoop.tmp.dir</name> 
      <value>file:/usr/local/hadoop/tmp</value>
      <description>Abase for other temporary directories.</description>
 </property> 
 <property>
   <name>fs.defaultFS</name>
   <value>hdfs://localhost:9000</value> 
  </property>
</configuration>
```

- hadoop.tmp.dir表示存放临时数据的目录，即包括NameNode的数据，也包 括DataNode的数据。该路径任意指定，只要实际存在该文件夹即可
- name为fs.defaultFS的值，表示hdfs路径的逻辑名称

在 `/usr/local/hadoop/etc/hadoop/`目录下，执行命令`sudo gedit hdfs-site.xml`，修改配置文件hdfs-site.xml，内容如下：

```
<configuration> 
   <property>
       <name>dfs.replication</name>
       <value>1</value> 
   </property> 
   <property>
       <name>dfs.namenode.name.dir</name>
       <value>file:/usr/local/hadoop/tmp/dfs/name</value> 
    </property>
   <property>
       <name>dfs.datanode.data.dir</name>         
       <value>file:/usr/local/hadoop/tmp/dfs/data</value>
   </property>
</configuration>
```

- dfs.replication表示副本的数量，伪分布式要设置为1
- dfs.namenode.name.dir表示本地磁盘目录，是存储fsimage文件的地方
- dfs.datanode.data.dir表示本地磁盘目录，HDFS数据存放block的地方

### 启动HDFS	

第一次启动时需要执行 `hdfs namenode - format`格式化节点

执行`start-all.sh`启动HDFS及相关服务，执行`jps`命令，出现以下6个线程说配置成功：

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200504235929.png)

进一步确认，可以浏览器访问`localhost:9870`和`localhost:8088`

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200505000709.png)

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200505000753.png)

## 配置Hadoop集群

先从master克隆两个虚拟机，分别命名为slave1和slave2。配置方案如下：

|      | master                | slave1                         | slave2                          |
| ---- | --------------------- | ------------------------------ | ------------------------------- |
| HDFS | NameNode<br/>DataNode | SecondaryNameNode<br/>DataNode | DataNode                        |
| YARN | NodeManager           | NodeManager                    | ResourceManager<br/>NodeManager |

### IP，主机名，主机映射配置

#### slave1和slave2配置

执行`sudo gedit /etc/network/interfaces`命令打开网络配置文件，修改slave1的IP为192.168.79.130，修改slave2的IP为192.168.79.131

执行`sudo gedit /etc/hostname`命令，修改slave1的hostname为slave1，修改slave2的hostname为slave2

执行`sudo gedit /etc/hosts`配置主机映射，添加下列语句：

```
#192.168.79.129	master#这条已经存在，不用添加
192.168.79.130	slave1
192.168.79.131	slave2
```

注意，上面的配置需要分别在slave1和slave2上执行，配置结束之后执行`reboot`重启主机

#### master配置

执行`sudo gedit /etc/hosts`配置主机映射，添加下列语句

```
192.168.79.130	slave1
192.168.79.131	slave2
```

### 免密配置

免密配置需要保证三台主机之间都能互相免密登录，**三台主机**都需要**执行**下面的命令

```
ssh-keygen -t rsa    //前面配置过则不需要
ssh-copy-id master    //将公钥文件发送给包括自身在内的3台服务器
ssh-copy-id slave1
ssh-copy-id slave2

ssh master    //连接其他服务器看还是否需要密码，三台服务器之间都不需要则配置成功
ssh slave1
ssh slave2
```

### 配置Hadoop配置文件

**在master中配置好后再分发给slave1和slave2**

执行`cd /usr/local/hadoop/etc/hadoop/`进入配置文件所在的文件夹

执行`sudo gedit core-site.xml`打开配置文件，配置如下：

```
<configuration> 
 <property>
      <name>hadoop.tmp.dir</name> 
      <value>file:/usr/local/hadoop/tmp</value>
      <description>Abase for other temporary directories.</description>
 </property>
<!-- 指定NameNode的地址 --> 
 <property>
   <name>fs.defaultFS</name>
   <value>hdfs://master:9000</value> 
 </property>
</configuration>
```

执行`sudo gedit hdfs-site.xml`打开配置文件，配置如下：

```
<configuration> 
   <!-- 指定冗余度 -->
   <property>
       <name>dfs.replication</name>
       <value>3</value> 
   </property>
   <!-- 指定NameNode数据的存储目录 --> 
   <property>
       <name>dfs.namenode.name.dir</name>
       <value>file:/usr/local/hadoop/tmp/dfs/name</value> 
   </property>
   <!-- 指定Datanode数据的存储目录 -->
   <property>
       <name>dfs.datanode.data.dir</name>         
       <value>file:/usr/local/hadoop/tmp/dfs/data</value>
   </property>
   <!-- 设置secondarynamenode -->
   <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>slave1:50090</value>
   </property>
</configuration>
```

执行`sudo gedit yarn-site.xml`打开配置文件，配置如下：

```
<configuration>
   <!--指定mapreduce走shuffle -->
   <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
   </property>
   <!-- 指定ResourceManager的地址-->
   <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>slave2</value>
   </property>
</configuration>
```

执行`sudo gedit mapred-site.xml`打开配置文件，配置如下：

```
<configuration>
<!-- 指定mapreduce运行在yarn上 -->
<property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
</property>
</configuration>
```

执行`sudo gedit workers`打开配置文件，配置如下：

```
master
slave1
slave2
```

将配置好的文件分发给slave1和slave2

```
scp -r /usr/local/hadoop/etc/hadoop/ slave1:/usr/local/hadoop/etc/
scp -r /usr/local/hadoop/etc/hadoop/ slave2:/usr/local/hadoop/etc/
```

### 启动集群

首次运行之前需要格式化namenode，如果首次运行失败后又重新进行了部分配置，运行之前最好也格式化namenode，格式化namenode之前需要删除hadoop目录下的data,logs以及/tmp目录下的hadoop开头的文件（在没有数据的情况下）

* 在master主机上执行`hadoop namenode -format`命令格式化namenode

* 在master主机上执行`start-dfs.sh`命令启动HDFS，如果正常启动：
	* 在master上执行`jps`命令可以看到有namenode，datanode
	* 在slave1上执行`jps`命令可以看到有secondarynamenode，datanode
	* 在slave2上执行`jps`命令可以看到有datanode
* 因为resourcemanager部署在slave2上，所以在slave2上执行`start-yarn.sh`命令启动YARN，成功则：
	* 在master上执行`jps`命令可以看到有namenode，datanode，nodemanager
	* 在slave1上执行`jps`命令可以看到有secondarynamenode，datanode，nodemanager
	* 在slave2上执行`jps`命令可以看到有datanode，resourcemanager，nodemanager

至此完成了集群的搭建

## 查看集群

打开`http://master:9870/`观测是否有三分datanode

![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200505191607.png)

## 测试集群



## 编写集群启动脚本

为了方便启动集群和关闭集群，编写一个脚本，在master机器上执行`mkdir bin`命令在用户目录下创建`bin`目录，执行`gedit mycluster`命令创建并打开文件`mycluster`,在文件中添加下面的内容：

```
#!/bin/bash
if [ $# -lt 1 ]
 then 
   echo "No Args Input Error!!!!!"
   exit
fi
case $1 in 
"start")
   echo "======================== start hdfs ========================== "
   ssh master "source /etc/profile;start-dfs.sh"
   echo "======================== start yarn ========================== "
   ssh slave2 "source /etc/profile;start-yarn.sh"
;;
"stop")
   echo "======================== stop yarn ========================== "
   ssh slave2 "source /etc/profile;stop-yarn.sh"
   echo "======================== stop hdfs ========================== "
   ssh master "source /etc/profile;stop-dfs.sh"
;;
*)
  echo "Input Args Error!!!!!"
;;
esac
```

在`bin`目录下执行`gedit myjps`命令创建并打开文件`myjps`,在文件中添加下面的内容：

```
#!/bin/bash
for i in master slave1 slave2
do
   echo "====================== $i JPS ======================="
   ssh $i "source /etc/profile;jps"
done
```

执行`chmod 777 mycluster`和`chmod 777 myjps`命令修改脚本权限，之后只需要执行`mycluster start`即可启动集群。

## 问题解决

由于JAVA，Hadoop等环境变量配置在/etc/profile文件中，导致每次新打开一个命令窗口都要重新输入 source /etc/profile 才能使jdk等配置文件生效：

解决方法：

打开~/.bashrc 文件

```
sudo gedit ~/.bashrc
```

加入下列语句

```
source /etc/profile
```

