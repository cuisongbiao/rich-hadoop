Hadoop 3.1.x集群安装和配置

1. Hadoop 3.1.x集群搭建
1.1 集群简介
Hadoop集群具体来说包含两个集群：HDFS集群和YARN集群，两者逻辑上分离，但物理上常在一起.
HDFS集群：负责海量数据的存储，集群中的角色主要有 NameNode/DataNode
YARN集群：负责海量数据运算时的资源调度，集群中的角色主要有 ResourceManager/NodeManager

本集群搭建案例，以4节点为例进行搭建，角色分配如下：

hadoop-node01 (192.168.51.155)  NameNode，ResourceManager，SecondaryNameNode，JobHistoryServer

hadoop-node02 (192.168.51.156)  DataNode，NodeManager

hadoop-node03 (192.168.51.157)  DataNode，NodeManager

注：hadoop集群以实际为准；

1.2 服务器准备
本案例使用虚拟机服务器来搭建Hadoop集群，所用软件及版本：

1.3 同步时间
sudo yum install ntp ntpdate ntp-doc -y
sudo systemctl restart ntpd
sudo systemctl enable ntpd

1.4 设置主机名
hostnamectl set-hostname hadoop-node01
hostnamectl set-hostname hadoop-node02
hostnamectl set-hostname hadoop-node03

1.5 配置内网域名映射：
vim /etc/hosts
192.168.51.155   hadoop-node01
192.168.51.156   hadoop-node02
192.168.51.157   hadoop-node03


1.6 配置ssh免密登陆:
ssh-keygen 
ssh-copy-id 192.168.51.155
ssh-copy-id 192.168.51.156
ssh-copy-id 192.168.51.157

1.9 关闭防火墙和selinux

#disable selinux
sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
setenforce 0

#stop firewalld
systemctl stop firewalld  && systemctl disable firewalld

1.10  jdk环境安装
OSS下载jdk包到安装目录：/opt/
解压安装包：
unzip jdk.zip
配置环境变量 
vim /etc/profile

export JAVA_HOME=/opt/jdk
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH

运行如下命令刷新环境变量:
source /etc/profile

2. Hadoop安装部署
OSS下载hadoop包到安装目录：/opt/
unzip hadoop.zip

配置环境变量:

vi /etc/profile
#在配置文件最后一行添加如下配置
 
#HADOOP
export HADOOP_HOME=/opt/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/lib

运行如下命令刷新环境变量:
source /etc/profile

进行测试是否成功：
[root@hadoop-node01 ~]# hadoop version
Hadoop 3.1.4
Source code repository https://github.com/apache/hadoop.git -r 1e877761e8dadd71effef30e592368f7fe66a61b
Compiled by gabota on 2020-07-21T08:05Z
Compiled with protoc 2.5.0
From source with checksum 38405c63945c88fdf7a6fe391494799b
This command was run using /opt/hadoop/share/hadoop/common/hadoop-common-3.1.4.jar

3. 配置Hadoop
3.1 创建存储目录：
(1)在/data目录下创建目录
mkdir -p /data/hadoop /data/hadoop/tmp /data/hadoop/var /data/hadoop/dfs /data/hadoop/dfs/name /data/hadoop/dfs/data /data/hadoop/yarn/local /data/hadoop/yarn/log

3.2 下载安装所需资源包
下载路径：oss://rich-ops/hadoop_3.1.x_install/

3.3 添加datanode节点（以实际规划为准）：
cd /opt/hadoop/etc/hadoop
vi  workers 
hadoop-node02
hadoop-node03

3.4 YARN调优
  Hadoop YARN同时支持内存和CPU两种资源的调度。
  wget http://public-repo-1.hortonworks.com/HDP/tools/2.1.1.0/hdp_manual_install_rpm_helper_files-2.1.1.385.tar.gz 
  可以使用脚本yarn-utils.py 来计算上面的值，进行调优，例如：32核，64G，2块硬盘

[root@hadoop-node01 opt]# cd hdp_manual_install_rpm_helper_files-2.1.1.385/scripts/
[root@hadoop-node01 scripts]# ls
directories.sh  hdp-configuration-utils.py  usersAndGroups.sh  yarn-utils.py
[root@hadoop-node01 scripts]# python yarn-utils.py -c 32 -m 64 -d 2 -k False
 Using cores=32 memory=64GB disks=2 hbase=False
 Profile: cores=32 memory=57344MB reserved=8GB usableMem=56GB disks=2
 Num Container=4
 Container Ram=14336MB
 Used Ram=56GB
 Unused Ram=8GB
 yarn.scheduler.minimum-allocation-mb=14336
 yarn.scheduler.maximum-allocation-mb=57344
 yarn.nodemanager.resource.memory-mb=57344
 mapreduce.map.memory.mb=14336
 mapreduce.map.java.opts=-Xmx11468m
 mapreduce.reduce.memory.mb=14336
 mapreduce.reduce.java.opts=-Xmx11468m
 yarn.app.mapreduce.am.resource.mb=14336
 yarn.app.mapreduce.am.command-opts=-Xmx11468m
 mapreduce.task.io.sort.mb=5734

根据以上参数，对应优化调整yarn-site.xml配置。

3.5 同步hadoop安装包及配置信息
使用scp命令将hadoop-node01下的目录复制到各个从节点的相应位置上

scp -r /opt/jdk hadoop-node02:/opt/jdk
scp -r /opt/hadoop hadoop-node02:/opt/hadoop
scp -r /etc/profile hadoop-node02:/etc/
 
scp -r /opt/jdk hadoop-node03:/opt/jdk
scp -r /opt/hadoop hadoop-node03:/opt/hadoop
scp -r /etc/profile hadoop-node03:/etc/

在从节点上分别运行下述命令刷新环境变量：
source /etc/profile


4 启动集群

4.1 格式化节点
在hadoop-node01中运行下述命令，格式化节点：
hdfs namenode -format

运行之后不报错，并在倒数第五六行有successfully即为格式化节点成功。

4.2 运行以下命令，启动hadoop集群的服务
cd /opt/hadoop/sbin/
启动所有
start-all.sh 
关闭所有
stop-all.sh

启动JobHistoryServer：
$ $HADOOP_HOME/bin/mapred --daemon start historyserver

注：hadoop集群中单个服务的启动/关闭命令：
Hadoop集群中单个服务的启动：
To start a Hadoop cluster you will need to start both the HDFS and YARN cluster.

The first time you bring up HDFS, it must be formatted. Format a new distributed filesystem as hdfs:

[hdfs]$ $HADOOP_HOME/bin/hdfs namenode -format <cluster_name>
Start the HDFS NameNode:

[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon start namenode
Start a HDFS DataNode:

[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon start datanode
If etc/hadoop/workers and ssh trusted access is configured (see Single Node Setup), all of the HDFS processes can be started with a utility script. As hdfs:

[hdfs]$ $HADOOP_HOME/sbin/start-dfs.sh
Start the  ResourceManager:

[yarn]$ $HADOOP_HOME/bin/yarn --daemon start resourcemanager
Start a NodeManager:

[yarn]$ $HADOOP_HOME/bin/yarn --daemon start nodemanager


Start the MapReduce JobHistory Server:

[mapred]$ $HADOOP_HOME/bin/mapred --daemon start historyserver

Hadoop集群中单个服务的关闭：
Stop the NameNode:

[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon stop namenode
Run a script to stop a DataNode as hdfs:

[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon stop datanode

Stop the ResourceManager:

[yarn]$ $HADOOP_HOME/bin/yarn --daemon stop resourcemanager
Stop a NodeManager:

[yarn]$ $HADOOP_HOME/bin/yarn --daemon stop nodemanager

Stop the MapReduce JobHistory Server:

[mapred]$ $HADOOP_HOME/bin/mapred --daemon stop historyserver

注：注意启动顺序，要最后启动JobHistoryServer

4.3 在hadoop节点上输入jps命令查看启动的服务:

[root@hadoop-node01 sbin]# jps
20305 ResourceManager
28161 Jps
19718 NameNode
20025 SecondaryNameNode
20151 JobHistoryServer

[root@hadoop-node02 data]# jps
90309 NodeManager
97796 Jps
90141 DataNode

4.4 访问WEB UI页面
##HDFS
http://192.168.51.155:50070/

##YARN
http://192.168.51.155:8088/cluster
##JobHistoryServer
http://192.168.51.155:19888


