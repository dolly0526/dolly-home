# HDFS原理和源码 #
2019/10/20 13:58:52 

## 文件块大小

![](https://i.imgur.com/aWdeQoZ.png)  
![](https://i.imgur.com/0KKqX3q.png)

## 元数据 ##
参考: [HDFS元数据管理机制](https://www.jianshu.com/p/e0c43541099d)

## 写流程 ##
![](https://i.imgur.com/RmbOqYp.png)  
1）客户端通过Distributed FileSystem模块向NameNode请求上传文件，NameNode检查目标文件是否已存在，父目录是否存在。  
2）NameNode返回是否可以上传。  
3）客户端请求第一个 Block上传到哪几个DataNode服务器上。  
4）NameNode返回3个DataNode节点，分别为dn1、dn2、dn3。  
5）客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。   
6）dn1、dn2、dn3逐级应答客户端。  
7）客户端开始往dn1上传第一个Block（先从磁盘读取数据放到一个本地内存缓存），以Packet为单位，dn1收到一个Packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列等待应答。  
8）当一个Block传输完成之后，客户端再次请求NameNode上传第二个Block的服务器。（重复执行3-7步）。

## 读流程 ##
![](https://i.imgur.com/0F0Qcce.png)  
1）客户端通过Distributed FileSystem向NameNode请求下载文件，NameNode通过查询元数据，找到文件块所在的DataNode地址。  
2）挑选一台DataNode（就近原则，然后随机）服务器，请求读取数据。  
3）DataNode开始传输数据给客户端（从磁盘里面读取数据输入流，以Packet为单位来做校验）。  
4）客户端以Packet为单位接收，先在本地缓存，然后写入目标文件。

# YARN原理 #

## 基本架构 ##
![](https://i.imgur.com/XE0iDMe.png)

## 工作机制 ##
![](https://i.imgur.com/87Gd0iZ.png)  
（1）MR程序提交到客户端所在的节点。  
（2）YarnRunner向ResourceManager申请一个Application。  
（3）RM将该应用程序的资源路径返回给YarnRunner。  
（4）该程序将运行所需资源提交到HDFS上。  
（5）程序资源提交完毕后，申请运行mrAppMaster。  
（6）RM将用户的请求初始化成一个Task。  
（7）其中一个NodeManager领取到Task任务。  
（8）该NodeManager创建容器Container，并产生MRAppmaster。  
（9）Container从HDFS上拷贝资源到本地。  
（10）MRAppmaster向RM 申请运行MapTask资源。  
（11）RM将运行MapTask任务分配给另外两个NodeManager，另两个NodeManager分别领取任务并创建容器。  
（12）MR向两个接收到任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动MapTask，MapTask对数据分区排序。  
（13）MrAppMaster等待所有MapTask运行完毕后，向RM申请容器，运行ReduceTask。  
（14）ReduceTask向MapTask获取相应分区的数据。  
（15）程序运行完毕后，MR会向RM申请注销自己。  

## 作业提交流程 ##
![](https://i.imgur.com/87Gd0iZ.png)  
- 作业提交  
第1步：Client调用job.waitForCompletion方法，向整个集群提交MapReduce作业。  
第2步：Client向RM申请一个作业id。  
第3步：RM给Client返回该job资源的提交路径和作业id。  
第4步：Client提交jar包、切片信息和配置文件到指定的资源提交路径。  
第5步：Client提交完资源后，向RM申请运行MrAppMaster。  
- 作业初始化  
第6步：当RM收到Client的请求后，将该job添加到容量调度器中。  
第7步：某一个空闲的NM领取到该Job。  
第8步：该NM创建Container，并产生MRAppmaster。  
第9步：下载Client提交的资源到本地。  
- 任务分配  
第10步：MrAppMaster向RM申请运行多个MapTask任务资源。  
第11步：RM将运行MapTask任务分配给另外两个NodeManager，另两个NodeManager分别领取任务并创建容器。  
- 任务运行  
第12步：MR向两个接收到任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动MapTask，MapTask对数据分区排序。  
第13步：MrAppMaster等待所有MapTask运行完毕后，向RM申请容器，运行ReduceTask。  
第14步：ReduceTask向MapTask获取相应分区的数据。  
第15步：程序运行完毕后，MR会向RM申请注销自己。  
- 进度和状态更新  
YARN中的任务将其进度和状态(包括counter)返回给应用管理器, 客户端每秒(通过mapreduce.client.progressmonitor.pollinterval设置)向应用管理器请求进度更新, 展示给用户。  
- 作业完成  
除了向应用管理器请求作业进度外, 客户端每5秒都会通过调用waitForCompletion()来检查作业是否完成。时间间隔可以通过mapreduce.client.completion.pollinterval来设置。作业完成之后, 应用管理器和Container会清理工作状态。作业的信息会被作业历史服务器存储以备之后用户核查。

## 资源调度器 ##
1. 先进先出调度器（FIFO）  
![](https://i.imgur.com/Aci7wm5.png)
2. 容量调度器（Capacity Scheduler）: Hadoop2.7.2**默认**    
![](https://i.imgur.com/Jpzc88A.png)
3. 公平调度器（Fair Scheduler）  
![](https://i.imgur.com/rBlqAQh.png)

## 推测执行 ##
- 作业完成时间取决于最慢的任务完成时间, 发现拖后腿的任务，比如某个任务运行速度远慢于任务平均速度。为拖后腿任务启动一个备份任务，同时运行。谁先运行完，则采用谁的结果。
- Spark的Master也可以配置推测执行
- 不能启用推测执行机制情况:   
（1）任务间存在严重的负载倾斜；  
（2）特殊任务，比如任务向数据库中写数据。  

## HDFS-HA原理 ##

### HA概述 ###

1. 所谓HA（High Available），即高可用（7*24小时不中断服务）  
2. 实现高可用最关键的策略是消除单点故障。HA严格来说应该分成各个组件的HA机制：HDFS的HA和YARN的HA。  
3. Hadoop2.0之前，在HDFS集群中NameNode存在单点故障（SPOF）  
4. NameNode主要在以下两个方面影响HDFS集群  

 - NameNode机器发生意外，如宕机，集群将无法使用，直到管理员重启
 - NameNode机器需要升级，包括软件、硬件升级，此时集群也将无法使用

5. HDFS-HA功能通过配置Active/Standby两个NameNodes实现在集群中对NameNode的热备来解决上述问题。如果出现故障，如机器崩溃或机器需要升级维护，这时可通过此种方式将NameNode很快的切换到另外一台机器。

### 工作要点 ###  

0. 本质上是**分布式锁**的应用

1. 元数据管理方式需要改变  

 - 内存中各自保存一份元数据；  
 - Edits日志只有Active状态的NameNode节点可以做写操作  
 - 两个NameNode都可以读取Edits  
 - 共享的Edits放在一个共享存储中管理（qjournal和NFS两个主流实现）  

2. 需要一个状态管理功能模块  

 - 实现了一个zkfailover，常驻在每一个namenode所在的节点，每一个zkfailover负责监控自己所在NameNode节点，利用zk进行状态标识，当需要进行状态切换时，由zkfailover来负责切换，切换时需要防止brain split现象的发生

3. 必须保证两个NameNode之间能够ssh无密码登录
4. 隔离（Fence），即同一时刻有且只有一个NameNode对外提供服务

### 自动故障转移机制 ### 

1. 前面学习了使用命令`hdfs haadmin -failover`手动进行故障转移，在该模式下，即使现役NameNode已经失效，系统也不会自动从现役NameNode转移到待机NameNode，下面学习如何配置部署HA自动进行故障转移。自动故障转移为HDFS部署增加了两个新组件：ZooKeeper和ZKFailoverController（ZKFC）进程，如图3-20所示。ZooKeeper是维护少量协调数据，通知客户端这些数据的改变和监视客户端故障的高可用服务。HA的自动故障转移依赖于ZooKeeper的以下功能：  

 - 故障检测：集群中的每个NameNode在ZooKeeper中维护了一个持久会话，如果机器崩溃，ZooKeeper中的会话将终止，ZooKeeper通知另一个NameNode需要触发故障转移。  
 - 现役NameNode选择：ZooKeeper提供了一个简单的机制用于唯一的选择一个节点为active状态。如果目前现役NameNode崩溃，另一个节点可能从ZooKeeper获得特殊的排外锁以表明它应该成为现役NameNode。  

2. ZKFC是自动故障转移中的另一个新组件，是ZooKeeper的客户端，也监视和管理NameNode的状态。每个运行NameNode的主机也运行了一个ZKFC进程，ZKFC负责：  

 - 健康监测：ZKFC使用一个健康检查命令定期地ping与之在相同主机的NameNode，只要该NameNode及时地回复健康状态，ZKFC认为该节点是健康的。如果该节点崩溃，冻结或进入不健康状态，健康监测器标识该节点为非健康的。  
 - ZooKeeper会话管理：当本地NameNode是健康的，ZKFC保持一个在ZooKeeper中打开的会话。如果本地NameNode处于active状态，ZKFC也保持一个特殊的znode锁，该锁使用了ZooKeeper对短暂节点的支持，如果会话终止，锁节点将自动删除。  
 - 基于ZooKeeper的选择：如果本地NameNode是健康的，且ZKFC发现没有其它的节点当前持有znode锁，它将为自己获取该锁。如果成功，则它已经赢得了选择，并负责运行故障转移进程以使它的本地NameNode为Active。故障转移进程与前面描述的手动故障转移相似，首先如果必要保护之前的现役NameNode，然后本地NameNode转换为Active状态。  

3. 图解  
   ![](https://i.imgur.com/KayKSGx.png)
4. 搭建HDFS-HA, 对照3中的图解, 可知每个组件的作用  
   ![](https://i.imgur.com/kctcQtS.png)
5. 补充说明

 - JournaNode是一个轻量级的元数据文件系统, 可以和其他组件混部 

> JournalNode machines - the machines on which you run the JournalNodes. The JournalNode daemon is relatively lightweight, so these daemons may reasonably be collocated on machines with other Hadoop daemons, for example NameNodes, the JobTracker, or the YARN ResourceManager. Note: There must be at least 3 JournalNode daemons, since edit log modifications must be written to a majority of JNs. This will allow the system to tolerate the failure of a single machine. You may also run more than 3 JournalNodes, but in order to actually increase the number of failures the system can tolerate, you should run an odd number of JNs, (i.e. 3, 5, 7, etc.). Note that when running with N JournalNodes, the system can tolerate at most (N - 1) / 2 failures and continue to function normally.

 - JournalNode文件夹下大部分是edits文件, 依赖Paxos算法保持全局数据的一致性; NameNode复制JournalNode中的edits文件
 - 如果杀掉ZKFC, 则对应的NameNode和ZK失联, 会被另一个NameNode杀掉
 - ZK中的hadoop-ha下有两个节点: ActiveStandbyElectorLock和ActiveBreadCrumb
   a. ActiveStandbyElectorLock是一个**临时**节点, 排他锁被活跃的那个NameNode占有  
   ![](https://i.imgur.com/EWSDugP.png)  
   b. ActiveBreadCrumb是一个**持久**节点, NameNode在变成活跃状态时会在此节点中注册信息, 在正常下线时会删除此节点; 若非正常下线, 则另一个节点会依据此节点的信息对此节点进行隔离(fencing), 防止脑裂  
   ![](https://i.imgur.com/A2pAaWK.png)  
   c. [hadoop HA机制](https://blog.csdn.net/liu812769634/article/details/53097268)  
   d. 非常正常状态下的主备更替过程  
   ![](https://i.imgur.com/wKlIsom.png)
 - 如果某个节点失联(比如断网), 则另一个节点不会变为活跃; HA机制下**宁愿宕机也不能脑裂**, 需要人工干预; 以下为日志  
   ![](https://i.imgur.com/71T5gOL.png)  
   ![](https://i.imgur.com/yZ1N7gh.png)

