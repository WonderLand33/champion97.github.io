---
title: RocketMQ集群部署笔记
date: 2019-12-09 17:41:38
tags:
---



## RocketMQ集群(2m-2s-sync)部署笔记

#### 背景介绍

随着时间推移和数据的累积，老项目中的ActiveMQ的缺点越来越明显了。主要表现在：队列增多、消息堆积、吞吐量下降甚至出现了消息丢失等一系列问题；结合这几天对消息中间件的调研最终决定使用RocketMQ（集群部署）替代ActiveMQ（迁移工作笔记后续补充）。



#### 使用RocketMQ的原因

- 灵活的分布式横向扩展部署架构，低延迟、高可靠
- 海量消息堆积能力
- 提供丰富的消息拉取模式，支持严格顺序消息、事务消息、定时消息等，提供各种消息过滤器机制，例如Tag和SQL
- 功能丰富的Dashboard，用于配置、指标监控
- 开源社区活跃，成熟（经过双十一考验）

- 等等....

*官网介绍：https://rocketmq.apache.org/docs/motivation*



#### RocketMQ的几种角色



![Q8jLC9.png](https://s2.ax1x.com/2019/12/05/Q8jLC9.png)



- **Producer**：消息发布的角色，支持分布式集群方式部署。Producer通过MQ的负载均衡模块选择相应的Broker集群队列进行消息投递，投递的过程支持快速失败并且低延迟。
- **Consumer**：消息消费的角色，支持分布式集群方式部署。支持以push推，pull拉两种模式对消息进行消费。同时也支持集群方式和广播方式的消费，它提供实时消息订阅机制，可以满足大多数用户的需求。
- **NameServer**：NameServer 是一个非常简单的Topic路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。主要包括两个功能：Broker管理，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；路由信息管理，每个NameServer将保存关于 Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer,Consumer仍然可以动态感知Broker的路由的信息。
- **BrokerServer**：Broker主要负责消息的存储、投递和查询以及服务高可用保证。



#### 几种集群搭建模式



##### 1.单Master模式

这种方式风险较大，一旦Broker重启或者宕机时，会导致整个服务不可用。不建议线上环境使用,可以用于本地测试。


##### 2.多Master模式

一个集群无Slave，全是Master，例如2个Master或者3个Master，这种模式的优缺点如下：

- 优点：配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢），性能最高；
- 缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到影响。
  

##### 3.多Master多Slave模式-异步复制

每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟（毫秒级），这种模式的优缺点如下：

- 优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，同时Master宕机后，消费者仍然可以从Slave消费，而且此过程对应用透明，不需要人工干预，性能同多Master模式几乎一样；
- 缺点：Master宕机，磁盘损坏情况下会丢失少量消息。



##### 4.多Master多Slave模式-同步双写

每个Master配置一个Slave，有多对Master-Slave，HA采用同步双写方式，即只有主备都写成功，才向应用返回成功，这种模式的优缺点如下：

- 优点：数据与服务都无单点故障，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高；
- 缺点：性能比异步复制模式略低（大约低10%左右），发送单个消息的RT会略高，且目前版本在主节点宕机后，备机不能自动切换为主机。



#### 2m-2s-sync模式搭建记录

我们采用的是2m-2s-sync()模式进行集群（也就是2个Master和2个Slave-异步复制的模式）。双主双从一般有两种模式：异步复制和同步双写。同步双写就是，当生产端生产消息后，主节点和从节点都收到消息并把消息都同时写入到本地后，才会回复消息。而异步复制是指，当主节点将数据保存到本地后，直接返回成功的消息，不关系从节点是否写入成功。因此通过比较，我们可以看出，异步复制效率比较高，而同步双写更具有高可用性，数据也变得更加可靠。



###### 环境准备

1.  Apache RocketMQ4.6.0 安装包

   下载地址：https://rocketmq.apache.org/dowloading/releases/，目前有官方有7个大版本，我下载的最新版本 **4.6.0**。

   **4.6.x** 对JRE版本要求(需要有Java运行环境)：

   | **Client** | **Broker** | **NameServer** |
   | ---------- | ---------- | -------------- |
   | \>=1.6     | \>=1.8     | \>=1.8         |

   

2. 2台Linux服务器

   | 主机  | IP地址        | 角色                                        | 模式         |
   | ----- | ------------- | ------------------------------------------- | ------------ |
   | A机器 | 192.168.1.240 | NameServer、broker-a-master、broker-b-slave | Master/Slave |
   | B机器 | 192.168.1.241 | NameServer、broker-b-master、broker-a-slave | Master/Slave |



3.分别A、B两台机器中下载解压RocketMQ文件(二进制包)

```bash
$ mkdir rocketmq && cd rocketmq
$ wget http://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.6.0/rocketmq-all-4.6.0-bin-release.zip
$ unzip rocketmq-all-4.6.0-bin-release.zip
$ mv rocketmq-all-4.6.0-bin-release 4.6.0
$ pwd
/home/data/application/rocketmq/4.6.0
$ ll
总用量 48
drwxr-xr-x 2 root root  4096 11月 20 11:04 benchmark
drwxr-xr-x 3 root root  4096 8月  19 15:31 bin
drwxr-xr-x 6 root root  4096 8月   6 16:23 conf
drwxr-xr-x 2 root root  4096 11月 20 11:04 lib
-rw-r--r-- 1 root root 17336 8月   6 16:23 LICENSE
-rw-r--r-- 1 root root  1338 8月   6 16:23 NOTICE
-rw-r--r-- 1 root root  4225 11月  1 16:54 README.md
```



###### 集群配置

1.在A机器上配置MasterA/SlaveB

```bash
[root@A 2m-2s-async]# vim broker-a.properties
```

```properties
# 集群名称
brokerClusterName=rocketmq_cluster
brokerName=broker-a
# 主服务器必须为0
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
autoCreateTopicEnable=true
listenPort=10911
storePathRootDir=$ROCKETMQ_HOME/srote/a
# nameServer地址，分号分割
namesrvAddr=192.168.1.240:9876;192.168.1.241:9876
```

```bash
[root@A 2m-2s-async]# vim broker-b-s.properties
```

```properties
# 集群名称
brokerClusterName=rocketmq_cluster
brokerName=broker-b
# 从服务器大于0
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
autoCreateTopicEnable=true
listenPort=10950
storePathRootDir=$ROCKETMQ_HOME/srote/b
# nameServer地址，分号分割
namesrvAddr=192.168.1.240:9876;192.168.1.241:9876
```



2.在B机器上配置MasterB/SlaveA

```bash
[root@B 2m-2s-async]# vim broker-b.properties
```

```properties
# 集群名称
brokerClusterName=rocketmq_cluster
brokerName=broker-b
# 主服务器必须为0
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
autoCreateTopicEnable=true
listenPort=10950
storePathRootDir=$ROCKETMQ_HOME/srote/b
# nameServer地址，分号分割
namesrvAddr=192.168.1.240:9876;192.168.1.241:9876
```

```bash
[root@B 2m-2s-async]# vim broker-a-s.properties
```

```properties
# 集群名称
brokerClusterName=rocketmq_cluster
brokerName=broker-a
# 从服务器大于0
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
autoCreateTopicEnable=true
listenPort=10911
storePathRootDir=$ROCKETMQ_HOME/srote/a
# nameServer地址，分号分割
namesrvAddr=192.168.1.240:9876;192.168.1.241:9876
```



3.启动NameServer集群

```bash
### 在A机器启动 NameServer
[root@A 4.6.0]# nohup sh bin/mqnamesrv &
### 验证 NameServer 是否启动成功
[root@A 4.6.0]# tail -f nohup.out
The Name Server boot success...

### 在B机器启动 NameServer
[root@B 4.6.0]# nohup sh bin/mqnamesrv &
### 验证 NameServer 是否启动成功
[root@B 4.6.0]# tail -f nohup.out
The Name Server boot success...
```

P.S. 如果出现`Cannot allocate memory` 异常信息，则需要修改一下runserver.sh和runbroker.sh的合适的jvm参数就可以了。



4.启动Broker集群

- 在A机器启动Broker

```bash
### 在A机器启动Master、Slave
[root@A 4.6.0]# nohup sh bin/mqbroker -c $ROCKETMQ_HOME/conf/2m-2s-async/broker-a.properties &
[root@A 4.6.0]# nohup sh bin/mqbroker -c $ROCKETMQ_HOME/conf/2m-2s-async/broker-b-s.properties &
```



- 在B机器启动Broker

```bash
### 在B机器启动Master、Slave
[root@B 4.6.0]# nohup sh bin/mqbroker -c $ROCKETMQ_HOME/conf/2m-2s-async/broker-b.properties &
[root@B 4.6.0]# nohup sh bin/mqbroker -c $ROCKETMQ_HOME/conf/2m-2s-async/broker-a-s.properties &
```



- `$ROCKETMQ_HOME`指的RocketMQ安装目录，需要用户自己设置此环境变量。

  ```bash
  vim /etc/profile
  ### 在尾部加上
  export ROCKETMQ_HOME=/home/data/application/rocketmq/4.6.0
  ### 重启环境生效
  source /etc/profile
  ```



我在A机器启动Broker的时候出现了异常信息：

```java
[main] ERROR RocketmqCommon - Failed to obtain the host name
java.net.UnknownHostException: testing: testing: 域名解析暂时失败
	at java.net.InetAddress.getLocalHost(InetAddress.java:1506) ~[na:1.8.0_221]
	at org.apache.rocketmq.common.BrokerConfig.localHostName(BrokerConfig.java:189) [rocketmq-common-4.6.0.jar:4.6.0]
	at org.apache.rocketmq.common.BrokerConfig.<init>(BrokerConfig.java:38) [rocketmq-common-4.6.0.jar:4.6.0]
	at org.apache.rocketmq.broker.BrokerStartup.createBrokerController(BrokerStartup.java:110) [rocketmq-broker-4.6.0.jar:4.6.0]
	at org.apache.rocketmq.broker.BrokerStartup.main(BrokerStartup.java:58) [rocketmq-broker-4.6.0.jar:4.6.0]
Caused by: java.net.UnknownHostException: testing: 域名解析暂时失败
	at java.net.Inet6AddressImpl.lookupAllHostAddr(Native Method) ~[na:1.8.0_221]
	at java.net.InetAddress$2.lookupAllHostAddr(InetAddress.java:929) ~[na:1.8.0_221]
	at java.net.InetAddress.getAddressesFromNameService(InetAddress.java:1324) ~[na:1.8.0_221]
	at java.net.InetAddress.getLocalHost(InetAddress.java:1501) ~[na:1.8.0_221]
	... 4 common frames omitted
```

由错误信息得知，这是由于Broker启动的时候找不到主机名（我的主机名叫testing）对应的IP地址造成的。可以通过修改 /etc/hosts 文件，增加一个主机名解析就可以了。

```bash
$ echo '127.0.0.1 testing' >> /etc/hosts
```



当看到所以broker启动成功即完成。



**`broker.properties`的详细配置信息：**

```
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a
#0表示Master，>0表示Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许Broker自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许Broker自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨4点
deleteWhen=04
#文件保留时间，默认48小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker的角色
#-ASYNC_MASTER异步复制Master
#-SYNC_MASTER同步双写Master
#-SLAVE
brokerRole=ASYNC_MASTER
#刷盘方式
#-ASYNC_FLUSH异步刷盘
#-SYNC_FLUSH同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```



查看NameServer、Broker服务是否启动

```bash
$ jps
32515 BrokerStartup
32618 BrokerStartup
32764 Jps
31613 NamesrvStartup
```



关闭所有服务

```bash
$ sh bin/mqshutdown broker
$ sh bin/mqshutdown namesrv
```



###### 安装管理面板(rocketmq-console-ng)



1.下载源代码编译

```bash
git clone https://github.com/apache/rocketmq-externals/tree/master/rocketmq-console
```

或者

```bash
 wget https://codeload.github.com/apache/rocketmq-externals/zip/master -O rocketmq-externals.zip
```

然后进入 rocketmq-console 模块



2.修改部分参数并编译

```bash
$ vim src/main/resources/application.properties
```

```properties
rocketmq.config.namesrvAddr=192.168.1.240:9876;192.168.1.241:9876
```



3.使用maven编译打包并启动

```
$ mvn clean package -Dmaven.test.skip=true
$ java -jar target/rocketmq-console-ng-1.0.1.jar
```



###### 使用mqadmin查看集群情况

```
$ export NAMESRV_ADDR="192.168.1.240:9876;192.168.5.241:9876"
$ sh mqadmin clusterlist -n 192.168.1.240:9876

```



###### 遇到的问题及处理办法

https://github.com/apache/rocketmq/issues/504

https://github.com/apache/rocketmq/issues/568

https://github.com/Shellbye/Shellbye.github.io/issues/69



###### 相关资料

https://rocketmq.apache.org

https://github.com/apache/rocketmq/tree/release-4.6.0/docs/cn

https://github.com/apache/rocketmq-spring