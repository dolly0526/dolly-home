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
