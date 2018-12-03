---
title: "Apache Kylin 单实例部署"
date: 2018-07-09T22:04:28+08:00
draft: false
tags:
- kylin
categories:
- programming
---


## 1. 部署方式概览



Apache Kylin ( 以下简称 Kylin ) 有四种部署方式（参考 [Kylin 的部署](http://www.cnblogs.com/brucexia/p/6221528.html)）:

- 单实例
- 集群
- 读写分离
- 多环境部署



### 1.1 Single Instance, 单实例部署



在 Hadoop 集群的客户机 （或 Hadoop 集群的一个节点）部署。建模人员通过 Web 登录进行建模和创建 Cube. 业务分析系统发送 SQL 到 Kylin, Kylin 查询 Cube 并返回结果。这种方式简单快捷，适合做试用或测试；并发请求较多时 (QPS &gt; 50), 单台 Kylin 节点将成为瓶颈。



### 1.2 Cluster, 集群方式



只需要增加 Kylin 的节点数，因为 Kylin 的元数据存储在 HBase 中，只需在 Kylin 中配置，让 Kylin 的每个节点都能访问同一个 Metadata 表就形成了 Kylin 集群 (kylin.metadata.url 的值相同)。Kylin 集群中只有一个 Kylin 实例运行任务引擎 (kylin.server.mode=all), 其他 Kylin 实例都是查询引擎模式 (kylin.server.mode=query)。 通常可以使用 LDAP 来管理用户权限。通常还需要负载均衡，如 lvs, nginx 等，负载均衡器可以使用 SSL 加密，安装防火墙。这种方式可以满足一般场景。



### 1.3 读写分离



Kylin 非常适合读写分离，原因是 Kylin 的工作负载有两种：

- Cube 的计算，调用 MapReduce 进行批量计算，而且延时很长的计算需要密集的 CPU 和 IO 资源。
- 在线的实时查询计算，即 Cube 计算结束后进行查询。都是只读操作，要求响应快，延迟低。



第一种 Cube 的计算会对集群带来很大负载，从而会影响在线的实时查询计算，所有需要做读写分离。如果你的环境，基本都是利用夜间执行Cube计算，白天上班时间进行查询分析，那么可以不用做读写分离。



在部署Kylin时，Hadoop配置项指向运算的集群，HBase的配置项指向单独部署的HBase集群。说白了，就是Hadoop和HBase集群的分离。



### 1.4 Staging 和 Production 多环境部署



首先进行开发环境测试，然后部署到 Staging 环境，最后没有问题后才会发布到 Production 生产环境。这样做可以避免不当设计导致的对生产环境的破坏。



使用这种方案的场景：假如一个新用户使用 Kylin, 可能他对 Cube 设计不是很熟悉，创建了一个非常不好的 Cube, 导致 Cube 计算时产生大量的不必要的运算，或者查询花费的时间很长，会对其他业务造成影响。我们不希望这个有问题的Cube能进入生产环境，那么就需要建立一个 Staging 环境，或者称为 QA 环境。



Kylin 提供了一个工具，几分钟就可以将一个 Cube 从 Staging 环境迁移到 Production 环境，不需要在新环境中重新 build. 因为在生产环境的 Cube 不允许修改，只能做增量的 build. 这样做保证了Staging 和 Production 的分离，保证发布到 Production 上的 Cube 都是经过评审过的，所以对 Production 环境不会造成不可预料的影响，从而保证了 Production 环境的稳定。



## 2. 单实例部署



在这里我们采用单实例部署方式。Kylin 配置要求：

- Hadoop: 2.7+
- Hive: 0.13 - 1.2.1+
- HBase: 1.1+
- Spark 2.1.1+
- JDK: 1.7+
- OS: Linux only, CentOS 6.5+ or Ubuntu 16.0.4+



### 2.1 Hadoop 集群



由于 Kylin 依赖于 Hadoop 集群，所以需要事先搭建一个 Hadoop 集群。Kylin 的官网推荐使用 [HDP sandbox](http://hortonworks.com/products/hortonworks-sandbox/), 有条件的个人用户可以下载 HDP 虚拟机镜像（VMware 镜像 15 G 左右，需要 4 核 8 G 配置 ）。



这里我们使用一个已有的 Hadoop 集群。软件清单如下：

- HADOOP-2.7.4
- ZOOKEEPER-3.4.10
- HIVE-2.1.1
- SPARK-2.2.0
- SQOOP-1.4.6
- HBASE-1.2.6



该集群已创建 root 和 hadoop 两个用户。



### 2.2 Hadoop 客户机



Kylin 可以安装到 Hadoop 集群的任意节点上，但官方推荐安装到 Hadoop 集群的客户机上。



> Kylin itself can be started in any node of the Hadoop cluster. For simplity, you can run it in the master node. But to get better stability, we suggest you to deploy it a pure Hadoop client node, on which the command lines like hive, hbase, Hadoop, hdfs already be installed and the client congfigurations (core-site.xml, hive-site.xml, hbase-site.xml, etc) are properly configured and will be automatically syned with other nodes. The Linux account that running Kylin has the permission to access the Hadoop cluster, including create/write HDFS folders, hive tables, hbase tables and submit MR jobs.



因此，需要准备一台 Hadoop 集群客户端机器。按文档要求，最小配置为 4 核 CPU, 16 G 内存，100 G 磁盘。这台机器要与 Hadoop 集群在网络上互通。挂载磁盘并格式化相关操作可以参考 [https://www.jdcloud.com/help/detail/515/isCatalog/1](https://www.jdcloud.com/help/detail/515/isCatalog/1).



### 2.3 配置 Hadoop 集群客户端



将这台机器改造为 Hadoop 集群客户端的思路比较简单，将已有 Hadoop 集群中的 jdk, Hadoop, hive, hbase 等组件复制过来并做一些相应的配置即可。



#### 2.3.1 主机映射



将 Hadoop 集群 Namenode 的 /etc/hosts 配置拷贝过来。



```
192.168.0.3 a2yGLbxU-Master1.jcloud.local a2yGLbxU-Master1
192.168.0.5 a2yGLbxU-Core1.jcloud.local a2yGLbxU-Core1
192.168.0.4 a2yGLbxU-Core2.jcloud.local a2yGLbxU-Core2
```



#### 2.3.2 新建用户



新建用户组和用户：



```bash
groupadd Hadoop
useradd -d /usr/Hadoop -g Hadoop -m Hadoop
```



设置密码：



```bash
passwd Hadoop
```



#### 2.3.3 配置 JDK, Hadoop, Hive, Hbase



拷贝 Hadoop 集群的 JDK 环境有利于保持版本一致性。

使用 scp -r 将 Hadoop 集群 master 节点的 jdk, Hadoop, Hive, HBase, Spark 等拷贝到客户机对应位置，并编辑 /etc/profile：



```
JAVA_HOME=/opt/jmr/jdk1.8.0_77
HADOOP_HOME=/data0/Hadoop-2.7.4
HBASE_HOME=/data0/hbase-1.2.6
HIVE_HOME=/data0/apache-hive-2.1.1-bin
SPARK_HOME=/data0/spark-2.2.0-bin-Hadoop2.7
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
PATH=$HADOOP_HOME/bin:$HBASE_HOME/bin:$HIVE_HOME/bin:$SPARK_HOME/bin:$JAVA_HOME/bin:$PATH
```



#### 2.3.4 验证 Hadoop 集群客户端



使用 `su - hadoop` 切换用户，验证 Hadoop 集群客户端是否 ready.



**Hadoop version**



```bash
hadoop version
```



输出：



```
Hadoop 2.7.4
Subversion https://shv@git-wip-us.apache.org/repos/asf/hadoop.git -r cd915e1e8d9d0131462a0b7301586c175728a282
Compiled by kshvachk on 2017-08-01T00:29Z
Compiled with protoc 2.5.0
From source with checksum 50b0468318b4ce9bd24dc467b7ce1148
This command was run using /data0/hadoop-2.7.4/share/hadoop/common/hadoop-common-2.7.4.jar
```



**Job**



运行一个示例 Job：



```bash
hadoop jar /data0/hadoop-2.7.4/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.4.jar pi 10 100
```



输出结果中包含 Pi 的近似值：



```
Estimated value of Pi is 3.14800000000000000000
```



**HDFS**



上传文件到 HDFS：



```bash
hadoop fs -mkdir /kylin_test
hadoop fs -put /data0/hadoop-2.7.4/etc/hadoop /kylin_test/input
```


以上命令将 /data0/hadoop-2.7.4/etc/hadoop 上传到 HDFS 的 /kylin_test/input.



执行运算：



```bash
hadoop jar /data0/hadoop-2.7.4/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.4.jar grep /kylin_test/input /kylin_test/output 'dfs[a-z.]+'
```



以上命令使用示例程序调用 MapReduce 分析上传到 HDFS 的文件包含的所有文本，统计正则表达式 `’dfs[a-z.]+’` 匹配到内容和出现的次数，结果保存到 HDFS 的 /kylin_test/output.



查看结果：



```bash
hadoop fs -cat /kylin_test/output/part-r-00000
```



**Hive**



Hiveserver2 是一个允许远程客户端针对 Hive 执行查询并获取结果的服务接口。它的默认监听端口是10000.



连接 Hiverserver：



```bash
beeline -u jdbc:hive2://192.168.0.3:10000 -n hadoop
```



192.168.0.3 是 Hadoop 集群的 master 节点 IP，也可以使用域名；`-n` 指定用户。

执行命令后进入交互式查询，可以输入 `SHOW TABLES;` 测试；使用 `!quit` 退出交互查询。



**Spark**



进入 Spark 安装位置，运行 Spark 自带的示例应用：



```bash
run-example SparkPi 10
```



输出结果中包含使用[蒙特卡罗算法](https://www.zhihu.com/question/20254139)计算得出的圆周率近似值：



```
18/07/09 17:46:59 INFO DAGScheduler: Job 0 finished: reduce at SparkPi.scala:38, took 0.519827 s
Pi is roughly 3.1395711395711396
```



### 2.4 安装 Kylin




#### 2.4.1 下载安装包



Kylin 的二进制安装包将近 300 M, 建议使用下载工具提前下载好。



下载页面：[http://kylin.apache.org/download/](http://kylin.apache.org/download/)



下载链接：[http://ftp.cuhk.edu.hk/pub/packages/apache.org/kylin/apache-kylin-2.4.0/apache-kylin-2.4.0-bin-hbase1x.tar.gz](http://ftp.cuhk.edu.hk/pub/packages/apache.org/kylin/apache-kylin-2.4.0/apache-kylin-2.4.0-bin-hbase1x.tar.gz)



下载完成后，上传到 Hadoop 集群客户端机器上。



#### 2.4.2 环境检查



解压安装包：



```bash
tar -zxvf apache-kylin-2.3.1-hbase1x-bin.tar.gz
cd apache-kylin-2.3.1-bin
export KYLIN_HOME=`pwd`
```



安装环境预检查：



```bash
sh $KYLIN_HOME/bin/check-env.sh
```



#### 2.4.3 启动 Kylin



```bash
sh $KYLIN_HOME/bin/kylin.sh start
```



启动后，可以查看运行日志： `$KYLIN_HOME/logs/kylin.log`



#### 2.4.4 访问 Kylin Web



使用浏览器访问 `http://hostname:7070/kylin`. 初始用户名/密码为 `ADMIN/KYLIN`.



#### 2.4.5 停止 Kylin



```bash
sh $KYLIN_HOME/bin/kylin.sh stop
```



## 3. 参考资料

- [使用Kylin构建企业大数据分析平台的4种部署方式](http://www.cnblogs.com/brucexia/p/6221528.html)
- [Linux分区、格式化和创建文件系统](https://www.jdcloud.com/help/detail/515/isCatalog/1)
- [Apache Kylin Installation Guide](http://kylin.apache.org/docs/install/index.html)
- [hadoop的client搭建-即集群外主机访问hadoop](https://www.cnblogs.com/pu20065226/p/8464867.html)
- [Bitnami Documentation For Hadoop](https://docs.bitnami.com/general/apps/hadoop/)
