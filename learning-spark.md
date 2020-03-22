# Spark原理和源码 #
2019/11/17

## 参考资料

- [岑玉海 - Spark源码系列](https://www.cnblogs.com/cenyuhai/tag/Spark源码系列/)
- [岑玉海 - Spark](https://www.cnblogs.com/cenyuhai/tag/Spark/) 
- [有态度的HBase/Spark/BigData - Spark](http://hbasefly.com/category/spark/)

## Spark Core ##
### 小故事  

![](https://i.imgur.com/uGo3gu9.png)

### RDD

0. 参考资料

- [Spark源码系列（二）RDD详解](https://www.cnblogs.com/cenyuhai/p/3779125.html)

2. RDD的理解  
![](https://i.imgur.com/7QDYWjB.png)
3. RDD的特性  
RDD表示只读的分区的数据集，对RDD进行改动，只能通过RDD的转换操作，由一个RDD得到一个新的RDD，新的RDD包含了从其他RDD衍生所必需的信息。RDDs之间存在依赖，RDD的执行是按照血缘关系延时计算的。如果血缘关系较长，可以通过持久化RDD来切断血缘关系。
 - **分区**: RDD逻辑上是分区的，每个分区的数据是抽象存在的，计算的时候会通过一个compute函数得到每个分区的数据。如果RDD是通过已有的文件系统构建，则compute函数是读取指定文件系统中的数据，如果RDD是通过其他RDD转换而来，则compute函数是执行转换逻辑将其他RDD的数据进行转换。
 - **只读**: RDD是只读的，要想改变RDD中的数据，只能在现有的RDD基础上创建新的RDD。由一个RDD转换到另一个RDD，可以通过丰富的操作算子实现，不再像MapReduce那样只能写map和reduce了. RDD的操作算子包括两类，一类叫做transformations，它是用来将RDD进行转化，构建RDD的血缘关系；另一类叫做actions，它是用来触发RDD的计算，得到RDD的相关计算结果或者将RDD保存的文件系统中。下图是RDD所支持的操作算子列表。
 - **依赖**: RDDs通过操作算子进行转换，转换得到的新RDD包含了从其他RDDs衍生所必需的信息，RDDs之间维护着这种血缘关系，也称之为依赖。如下图所示，依赖包括两种，一种是窄依赖，RDDs之间分区是一一对应的，另一种是宽依赖，下游RDD的每个分区与上游RDD(也称之为父RDD)的每个分区都有关，是多对多的关系。
 - **缓存**: 如果在应用程序中多次使用同一个RDD，可以将该RDD缓存起来，该RDD只有在第一次计算的时候会根据血缘关系得到分区的数据，在后续其他地方用到该RDD的时候，会直接从缓存处取而不用再根据血缘关系计算，这样就加速后期的重用。如下图所示，RDD-1经过一系列的转换后得到RDD-n并保存到hdfs，RDD-1在这一过程中会有个中间结果，如果将其缓存到内存，那么在随后的RDD-1转换到RDD-m这一过程中，就不会计算其之前的RDD-0了。
 - **CheckPoint**: 虽然RDD的血缘关系天然地可以实现容错，当RDD的某个分区数据失败或丢失，可以通过血缘关系重建。但是对于长时间迭代型应用来说，随着迭代的进行，RDDs之间的血缘关系会越来越长，一旦在后续迭代过程中出错，则需要通过非常长的血缘关系去重建，势必影响性能。为此，RDD支持checkpoint将数据保存到持久化的存储中，这样就可以切断之前的血缘关系，因为checkpoint后的RDD不需要知道它的父RDDs了，它可以从checkpoint处拿到数据。
### groupByKey和reduceByKey

 - 参考: [reduceByKey和groupByKey区别与用法](https://blog.csdn.net/weixin_41804049/article/details/80373741)
 - groupByKey: groupByKey 也是对每个 key 进行操作，但只生成一个 sequence。

```scala
  /**
   * Group the values for each key in the RDD into a single sequence. Allows controlling the
   * partitioning of the resulting key-value pair RDD by passing a Partitioner.
   * The ordering of elements within each group is not guaranteed, and may even differ
   * each time the resulting RDD is evaluated.
   *
   * @note This operation may be very expensive. If you are grouping in order to perform an
   * aggregation (such as a sum or average) over each key, using `PairRDDFunctions.aggregateByKey`
   * or `PairRDDFunctions.reduceByKey` will provide much better performance.
   *
   * @note As currently implemented, groupByKey must be able to hold all the key-value pairs for any
   * key in memory. If a key has too many values, it can result in an [[OutOfMemoryError]].
   */
  def groupByKey(partitioner: Partitioner): RDD[(K, Iterable[V])] = self.withScope {
    // groupByKey shouldn't use map side combine because map side combine does not
    // reduce the amount of data shuffled and requires all map side data be inserted
    // into a hash table, leading to more objects in the old gen.
    val createCombiner = (v: V) => CompactBuffer(v)
    val mergeValue = (buf: CompactBuffer[V], v: V) => buf += v
    val mergeCombiners = (c1: CompactBuffer[V], c2: CompactBuffer[V]) => c1 ++= c2
    val bufs = combineByKeyWithClassTag[CompactBuffer[V]](
	  // dolly: mapSideCombine设置为false, 在map端不会合并
      createCombiner, mergeValue, mergeCombiners, partitioner, mapSideCombine = false)
    bufs.asInstanceOf[RDD[(K, Iterable[V])]]
  }
```
 - reduceByKey: 在一个(K,V)的 RDD 上调用，返回一个(K,V)的 RDD，使用指定的 reduce 函数，将相同
key 的值聚合到一起，reduce 任务的个数可以通过第二个可选的参数来设置。

```scala
  /**
   * Merge the values for each key using an associative and commutative reduce function. This will
   * also perform the merging locally on each mapper before sending results to a reducer, similarly
   * to a "combiner" in MapReduce.
   */
  def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)] = self.withScope {
	// dolly: mapSideCombine默认为true, 在map端会预先合并
    combineByKeyWithClassTag[V]((v: V) => v, func, func, partitioner)
  }
```
 - 区别:  
a. reduceByKey：按照 key 进行聚合，在 shuffle 之前有 combine（预聚合）操作，返回结果
是 RDD[k,v].  
b. groupByKey：按照 key 进行分组，直接进行 shuffle。  
c. 开发指导：reduceByKey 比 groupByKey，建议使用。但是需要注意是否会影响业务逻辑。

## 部署模式 ##
1. 通用运行流程概述
 - 不论 Spark 以何种模式进行部署，任务提交后，都会先启动 Driver 进程，随后 Driver 进程向集群管理器注册应用程序，之后集群管理器根据此任务的配置文件分配 Executor 并启动，当 Driver 所需的资源全部满足后，Driver 开始执行 main 函数，Spark 查询为懒执行，当执行到 action 算子时开始反向推算，根据宽依赖进行 stage 的划分，随后每一个 stage 对应一个 taskset，taskset 中有多个 task，根据本地化原则，task 会被分发到指定的 Executor 去执行，在任务执行的过程中，Executor 也会不断与 Driver 进行通信，报告任务运行情况。
 - 图解  
![](https://i.imgur.com/LHVvkbv.png)
2. 用户在提交任务给 Spark 处理时，以下两个参数共同决定了 Spark 的运行方式。
 - master MASTER_URL：决定了 Spark 任务提交给哪种集群处理。
 - deploy-mode DEPLOY_MODE：决定了 Driver 的运行方式，可选值为Client 或者 Cluster。

### YARN-Cluster模式 ###

1. YARN调度流程  
![](https://i.imgur.com/7kWlpbG.png)
2. 任务提交流程
 - 在 YARN Cluster 模式下，任务提交后会和 ResourceManager 通讯申请启动ApplicationMaster，随后 ResourceManager 分配 container，在合适的 NodeManager上启动 ApplicationMaster，此时的ApplicationMaster 就是 Driver。Driver 启动后向 ResourceManager 申请 Executor 内存，ResourceManager 接到ApplicationMaster 的资源申请后会分配 container，然后在合适的 NodeManager 上启动 Executor 进程，Executor 进程启动后会向 Driver 反向注册，Executor 全部注册完成后 Driver 开始执行 main 函数，之后执行到 Action 算子时，触发一个 job，并根据宽依赖开始划分 stage，每个 stage 生成对应的 taskSet，之后将 task 分发到各个Executor上执行。
 - 图解  
![](https://i.imgur.com/EkOBQuJ.png)
 - 提交一个 Spark 应用程序，首先通过 Client 向 ResourceManager 请求启动一个Application，同时检查是否有足够的资源满足 Application 的需求，如果资源条件满足，则准备 ApplicationMaster 的启动上下文，交给 ResourceManager，并循环监控Application 状态。当提交的资源队列中有资源时，ResourceManager 会在某个 NodeManager 上启动 ApplicationMaster 进程，ApplicationMaster 会单独启动 Driver 后台线程，当Driver 启动后，ApplicationMaster 会通过本地的 RPC 连接 Driver，并开始向ResourceManager 申请 Container 资源运行 Executor 进程（一个 Executor 对应与一个Container），当 ResourceManager 返回 Container 资源，ApplicationMaster 则在对应的 Container 上启动 Executor。Driver 线程主要是初始化 SparkContext 对象，准备运行所需的上下文，然后一方面保持与 ApplicationMaster 的 RPC 连接，通过 ApplicationMaster 申请资源，另一方面根据用户业务逻辑开始调度任务，将任务下发到已有的空闲 Executor 上。当 ResourceManager 向 ApplicationMaster 返 回 Container 资 源 时 ，ApplicationMaster 就尝试在对应的 Container 上启动 Executor 进程，Executor 进程起来后，会向 Driver 反向注册，注册成功后保持与 Driver 的心跳，同时等待 Driver分发任务，当分发的任务执行完毕后，将任务状态上报给 Driver。从上述时序图可知，Client 只负责提交 Application 并监控 Application 的状态。对于 Spark 的任务调度主要是集中在两个方面: **资源申请和任务分发**，其主要是通过 ApplicationMaster、Driver 以及 Executor 之间来完成。
 - 时序图  
![](https://i.imgur.com/ifQMoIn.png)
3. 演示指令


 ```
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--num-executors 2 \
--master yarn \
--deploy-mode cluster \
./examples/jars/spark-examples_2.11-2.1.1.jar \
100

bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--num-executors 2 \
--master yarn \
--deploy-mode client \
./examples/jars/spark-examples_2.11-2.1.1.jar \
100
 ```
### spark-submit源码解析

0. 参考资料

- [Spark源码系列（一）spark-submit提交作业过程](https://www.cnblogs.com/cenyuhai/p/3775687.html)
- [Spark源码系列（七）Spark on yarn具体实现](https://www.cnblogs.com/cenyuhai/p/3834894.html)

1. Spark源码中特殊的类
 - Backend: 后台
 - rpcEnv:  RPC
 - amEndpoint: 终端
 - RpcEndpointAddress: 终端地址
2. SparkSubmit源码

 - 我们在spark-submit时, 会按某一种规则输入编写命令, SparkSubmit类主要是把参数和命令进行封装, 通过反射加载执行任务的类, 并向RM提交任务, 之后的事情交给YARN处理.


 ```
(1) SparkSubmit
    
    // 启动进程
    -- main
    
        // 封装参数
        -- new SparkSubmitArguments
        
        // 提交
        -- submit
        
            // 准备提交环境
            -- prepareSubmitEnvironment
            
                // Cluster
                -- childMainClass = "org.apache.spark.deploy.yarn.Client"
                // Client
                -- childMainClass = args.mainClass (SparkPi)
            
            -- doRunMain (runMain)
            
                // 反射加载类
                -- Utils.classForName(childMainClass)
                // 查找main方法
                -- mainClass.getMethod("main", new Array[String](0).getClass)
                // 调用main方法
                -- mainMethod.invoke
             
// org.apache.spark.deploy.yarn.Client   
(2) Client

    -- main
    
        -- new ClientArguments(argStrings)
        
        -- new Client
        
            -- yarnClient = YarnClient.createYarnClient
        
        -- client.run
                
                -- submitApplication
                
                    // 封装指令 command = bin/java org.apache.spark.deploy.yarn.ApplicationMaster (Cluster)
                    // command = bin/java org.apache.spark.deploy.yarn.ExecutorLauncher  (client)
                    -- createContainerLaunchContext

                    
                    -- createApplicationSubmissionContext
                
                    // 向Yarn提交应用，提交指令
                    -- yarnClient.submitApplication(appContext)
 ```
3. ApplicationMaster源码

 - ApplicationMaster类先会创建AM类, 然后加载用户的类的main方法, 之后AM就作为该任务的Driver; 启动Driver线程, 向YARN申请资源, 之后分配资源给NM, 起多个Executor后台线程
 ```
1) ApplicationMaster
    
    // 启动进程
    -- main
    
        -- new ApplicationMasterArguments(args)
        
        // 创建应用管理器对象
        -- new ApplicationMaster(amArgs, new YarnRMClient)
        
        // 运行
        -- master.run
        
            // Cluster
            -- runDriver
            
                // 启动用户应用
                -- startUserApplication
                
                    // 获取用户应用的类的main方法
                    -- userClassLoader.loadClass(args.userClass)
      .getMethod("main", classOf[Array[String]])
      
                    // 启动Driver线程，执行用户类的main方法，
                    -- new Thread().start()
                    
                // 注册AM
                -- registerAM
                
                    // 获取yarn资源
                    -- client.register
                    
                    // 分配资源
                    -- allocator.allocateResources()
                    
                        -- handleAllocatedContainers
                        
                            -- runAllocatedContainers
                            
                                -- new ExecutorRunnable().run
                                
                                    -- startContainer
                                    
                                        // command = bin/java org.apache.spark.executor.CoarseGrainedExecutorBackend
                                        -- prepareCommand
 ```
4. CoarseGrainedExecutorBackend源码

 - CoarseGrainedExecutorBackend类会起一个线程, 主要实现反向注册和接受返回信息
 ```
1) CoarseGrainedExecutorBackend
    
    -- main
    
        -- run
        
            -- onStart
            	
				//反向注册
                -- ref.ask[Boolean](RegisterExecutor...
            
			//接收向driver注册executor的返回消息
            -- receive
            
                --  case RegisteredExecutor
					
					//Executor准确来说, 是该类的一个属性
                    -- new Executor
 ```
### 总结

 - YARN部署Spark流程图  
![](https://i.imgur.com/dOqCRik.png)
 - 源码级图解  
![](https://i.imgur.com/qQZDWNz.png)

## 任务调度机制 ##

0. 参考资料

- [Spark源码系列（三）作业运行过程](https://www.cnblogs.com/cenyuhai/p/3784602.html)

1. WordCount
 - 图解  
![](https://i.imgur.com/pKP1hx2.png)
 - 代码实现
 ```
sc.textFile("file:///app/software/spark/README.md")
	.flatMap(_.split(" "))
	.map((_, 1))
	.reduceByKey(_+_)
	.sortBy(_._2, false)
	.foreach(println)
 ```
 - 任务调度图解  
![](https://i.imgur.com/tQmlpz4.png)
2. 任务调度概述  
 - 当Driver起来后，Driver则会根据用户程序逻辑准备任务，并根据Executor资源情况逐步分发任务。在详细阐述任务调度前，首先说明下Spark里的几个概念。一个Spark应用程序包括Job、Stage以及Task三个概念：  
a. Job是以Action方法为界，遇到一个Action方法则触发一个Job；  
b. Stage是Job的子集，以RDD宽依赖(即**Shuffle**)为界，遇到Shuffle做一次划分；  
c. Task是Stage的子集，以并行度(分区数)来衡量，分区数是多少，则有多少个task。  
Spark的任务调度总体来说分两路进行，一路是Stage级的调度，一路是Task级的调度
 - 图解  
![](https://i.imgur.com/5eF7hXj.png)
 - Spark RDD通过其Transactions操作，形成了RDD血缘关系图，即DAG，最后通过Action的调用，触发Job并调度执行。DAGScheduler负责Stage级的调度，主要是将job切分成若干Stages，并将每个Stage打包成TaskSet交给TaskScheduler调度。TaskScheduler负责Task级的调度，将DAGScheduler给过来的TaskSet按照指定的调度策略分发到Executor上执行，调度过程中SchedulerBackend负责提供可用资源，其中SchedulerBackend有多种实现，分别对接不同的资源管理系统。
 - 图解  
![](https://i.imgur.com/APWvzg9.png)  
![](https://i.imgur.com/I66TTPy.png)
 - Driver初始化SparkContext过程中，会分别初始化DAGScheduler、TaskScheduler、SchedulerBackend以及HeartbeatReceiver，并启动SchedulerBackend以及HeartbeatReceiver。SchedulerBackend通过ApplicationMaster申请资源，并不断从TaskScheduler中拿到合适的Task分发到Executor执行。HeartbeatReceiver负责接收Executor的心跳信息，监控Executor的存活状况，并通知到TaskScheduler。

### Stage级调度 ###
1. Spark的任务调度是从DAG切割开始，主要是由DAGScheduler来完成。当遇到一个Action操作后就会触发一个Job的计算，并交给DAGScheduler来提交，下图是涉及到Job提交的相关方法调用流程图。  
![](https://i.imgur.com/wsij769.png)
2. Job由最终的RDD和Action方法封装而成，SparkContext将Job交给DAGScheduler提交，它会根据RDD的血缘关系构成的DAG进行切分，将一个Job划分为若干Stages，具体划分策略是，由最终的RDD不断通过依赖回溯判断父依赖是否是宽依赖，即以Shuffle为界，划分Stage，窄依赖的RDD之间被划分到同一个Stage中，可以进行pipeline式的计算，如上图紫色流程部分。划分的Stages分两类，一类叫做ResultStage，为DAG最下游的Stage，由Action方法决定，另一类叫做ShuffleMapStage，为下游Stage准备数据，下面看一个简单的例子WordCount。  
![](https://i.imgur.com/il8Px3j.png)  
Job由saveAsTextFile触发，该Job由RDD-3和saveAsTextFile方法组成，根据RDD之间的依赖关系从RDD-3开始回溯搜索，直到没有依赖的RDD-0，在回溯搜索过程中，RDD-3依赖RDD-2，并且是宽依赖，所以在RDD-2和RDD-3之间划分Stage，RDD-3被划到最后一个Stage，即ResultStage中，RDD-2依赖RDD-1，RDD-1依赖RDD-0，这些依赖都是窄依赖，所以将RDD-0、RDD-1和RDD-2划分到同一个Stage，即ShuffleMapStage中，实际执行的时候，数据记录会一气呵成地执行RDD-0到RDD-2的转化。不难看出，其本质上是一个深度优先搜索算法。
3. 一个Stage是否被提交，需要判断它的父Stage是否执行，只有在父Stage执行完毕才能提交当前Stage，如果一个Stage没有父Stage，那么从该Stage开始提交。Stage提交时会将Task信息（分区信息以及方法等）序列化并被打包成TaskSet交给TaskScheduler，一个Partition对应一个Task，另一方面TaskScheduler会监控Stage的运行状态，只有Executor丢失或者Task由于Fetch失败才需要重新提交失败的Stage以调度运行失败的任务，其他类型的Task失败会在TaskScheduler的调度过程中重试。

### Task级调度 ###
1. Spark Task的调度是由TaskScheduler来完成，由前文可知，DAGScheduler将Stage打包到TaskSet交给TaskScheduler，TaskScheduler会将TaskSet封装为TaskSetManager加入到调度队列中，TaskSetManager结构如下图所示。  
![](https://i.imgur.com/w8eIsA9.png)
2. TaskSetManager负责监控管理同一个Stage中的Tasks，TaskScheduler就是以TaskSetManager为单元来调度任务。  
![](https://i.imgur.com/lDtIWTe.png)
3. 前面也提到，TaskScheduler初始化后会启动SchedulerBackend，它负责跟外界打交道，接收Executor的注册信息，并维护Executor的状态，所以说SchedulerBackend是管“粮食”的，同时它在启动后会定期地去“询问”TaskScheduler有没有任务要运行，也就是说，它会定期地“问”TaskScheduler“我有这么余量，你要不要啊”，TaskScheduler在SchedulerBackend“问”它的时候，会从调度队列中按照指定的调度策略选择TaskSetManager去调度运行，大致方法调用流程如下图所示：  
![](https://i.imgur.com/c2bl5ES.png)
4. 图中，将TaskSetManager加入rootPool调度池中之后，调用SchedulerBackend的riviveOffers方法给driverEndpoint发送ReviveOffer消息；driverEndpoint收到ReviveOffer消息后调用makeOffers方法，过滤出活跃状态的Executor（这些Executor都是任务启动时反向注册到Driver的Executor），然后将Executor封装成WorkerOffer对象；准备好计算资源（WorkerOffer）后，taskScheduler基于这些资源调用resourceOffer在Executor上分配task。

## 通讯架构 ##

1. 通信架构概述  
   ![](https://i.imgur.com/bvI2uKK.png)  
   ![](https://i.imgur.com/RDAgrtV.png)
2. 通讯架构解析

 - 图解
   ![](https://i.imgur.com/6CT8Gnb.png)
 - 解析  
   (1) RpcEndpoint：RPC 端点，Spark 针对每个节点（Client/Master/Worker）都称之为一个 Rpc 端点，且都实现 RpcEndpoint 接口，内部根据不同端点的需求，设计不同的消息和不同的业务处理，如果需要发送（询问）则调用 Dispatcher；  
   (2) RpcEnv：RPC 上下文环境，每个 RPC 端点运行时依赖的上下文环境称为RpcEnv；  
   (3) Dispatcher：消息分发器，针对于 RPC 端点需要发送消息或者从远程 RPC接收到的消息，分发至对应的指令收件箱/发件箱。如果指令接收方是自己则存入收件箱，如果指令接收方不是自己，则放入发件箱；  
   (4) Inbox： 指 令 消 息 收 件 箱 ， 一 个 本 地 RpcEndpoint 对 应 一 个 收 件 箱 ，Dispatcher 在 每 次 向 Inbox 存 入 消 息 时 ， 都 将 对 应 EndpointData 加 入 内 部ReceiverQueue 中 ， 另 外 Dispatcher 创 建 时 会 启 动 一 个 单 独 线 程 进 行 轮 询ReceiverQueue，进行收件箱消息消费；  
   (5) RpcEndpointRef：RpcEndpointRef 是对远程 RpcEndpoint 的一个引用。当我们需要向一个具体的 RpcEndpoint 发送消息时，一般我们需要获取到该 RpcEndpoint的引用，然后通过该应用发送消息。  
   (6) OutBox： 指 令 消 息 发 件 箱 ， 对 于 当 前 RpcEndpoint 来 说 ， 一 个 目 标RpcEndpoint 对应一个发件箱，如果向多个目标 RpcEndpoint 发送信息，则有多个OutBox。当消息放入 Outbox后，紧接着通过 TransportClient 将消息发送出去。消息放入发件箱以及发送过程是在同一个线程中进行；  
   (7) RpcAddress：表示远程的 RpcEndpointRef 的地址，Host + Port。  
   (8) TransportClient ： Netty 通 信 客 户 端 ， 一 个 OutBox 对 应 一 个 TransportClient，TransportClient 不断轮询 OutBox，根据 OutBox 消息的 receiver 信息，请求对应的远程TransportServer；  
   (9) TransportServer ： Netty 通 信 服 务 端 ， 一 个 RpcEndpoint 对 应 一 个TransportServer，接受远程消息后调用 Dispatcher 分发消息至对应收发件箱；

## Shuffle解析 ##

0. 参考资料

- 《尚硅谷大数据技术之Spark内核解析》
- [Spark源码系列（六）Shuffle的过程解析](https://www.cnblogs.com/cenyuhai/p/3826227.html)

1. Spark-2.x源码

```scala
package org.apache.spark.shuffle.sort.SortShuffleManager;

  /** Get a writer for a given partition. Called on executors by map tasks. */
  // dolly: 写到磁盘
  override def getWriter[K, V](
      handle: ShuffleHandle,
      mapId: Int,
      context: TaskContext): ShuffleWriter[K, V] = {
    numMapsForShuffle.putIfAbsent(
      handle.shuffleId, handle.asInstanceOf[BaseShuffleHandle[_, _, _]].numMaps)
    val env = SparkEnv.get
    // dolly: 模式匹配选择一种Shuffle方式
    handle match {
      case unsafeShuffleHandle: SerializedShuffleHandle[K @unchecked, V @unchecked] =>
        new UnsafeShuffleWriter(
          env.blockManager,
          shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
          context.taskMemoryManager(),
          unsafeShuffleHandle,
          mapId,
          context,
          env.conf)
      case bypassMergeSortHandle: BypassMergeSortShuffleHandle[K @unchecked, V @unchecked] =>
        new BypassMergeSortShuffleWriter(
          env.blockManager,
          shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
          bypassMergeSortHandle,
          mapId,
          context,
          env.conf)
      case other: BaseShuffleHandle[K @unchecked, V @unchecked, _] =>
        new SortShuffleWriter(shuffleBlockResolver, other, mapId, context)
    }
  }

  /**
   * Register a shuffle with the manager and obtain a handle for it to pass to tasks.
   */
  override def registerShuffle[K, V, C](
      shuffleId: Int,
      numMaps: Int,
      dependency: ShuffleDependency[K, V, C]): ShuffleHandle = {
    if (SortShuffleWriter.shouldBypassMergeSort(SparkEnv.get.conf, dependency)) {
      // If there are fewer than spark.shuffle.sort.bypassMergeThreshold partitions and we don't
      // need map-side aggregation, then write numPartitions files directly and just concatenate
      // them at the end. This avoids doing serialization and deserialization twice to merge
      // together the spilled files, which would happen with the normal code path. The downside is
      // having multiple files open at a time and thus more memory allocated to buffers.
      new BypassMergeSortShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
      // dolly: 继续往里看, 一般有聚合的操作都不会走这个条件
    } else if (SortShuffleManager.canUseSerializedShuffle(dependency)) {
      // Otherwise, try to buffer map outputs in a serialized form, since this is more efficient:
      new SerializedShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else {
      // Otherwise, buffer map outputs in a deserialized form:
      new BaseShuffleHandle(shuffleId, numMaps, dependency)
    }
  }

private[spark] object SortShuffleWriter {
  def shouldBypassMergeSort(conf: SparkConf, dep: ShuffleDependency[_, _, _]): Boolean = {
    // We cannot bypass sorting if we need to do map-side aggregation.
    // dolly: 如果采用了map端预聚合, 则不能采用bypass方式
    if (dep.mapSideCombine) {
      require(dep.aggregator.isDefined, "Map-side combine without Aggregator specified!")
      false
    } else {
      // dolly: spark.shuffle.sort.bypassMergeThreshold 默认为200
      val bypassMergeThreshold: Int = conf.getInt("spark.shuffle.sort.bypassMergeThreshold", 200)
      // dolly: shuffle map task 数量小于 spark.shuffle.sort.bypassMergeThreshold 参数值
      dep.partitioner.numPartitions <= bypassMergeThreshold
    }
  }
}
```

### 数据倾斜

0. 参考资料

- 《尚硅谷大数据技术之Spark性能调优与故障处理》

- [Spark性能优化之道——解决Spark数据倾斜（Data Skew）的N种姿势](https://www.cnblogs.com/cssdongl/p/6594298.html)

## 内存管理

- 《尚硅谷大数据技术之Spark内核解析》
- [Spark源码系列（五）分布式缓存](https://www.cnblogs.com/cenyuhai/p/3808774.html)

### OOM

0. 参考资料

- [Spark面对OOM问题的解决方法及优化总结](https://blog.csdn.net/yhb315279058/article/details/51035631)

## Spark SQL

- [SparkSQL – 从0到1认识Catalyst](http://hbasefly.com/2017/03/01/sparksql-catalyst/)
- [Spark源码系列（九）Spark SQL初体验之解析过程详解](https://www.cnblogs.com/cenyuhai/p/4133319.html)

### Join

- [SparkSQL – 有必要坐下来聊聊Join](http://hbasefly.com/2017/03/19/sparksql-basic-join/)

## Spark Streaming ##

- [Spark源码系列（八）Spark Streaming实例分析](https://www.cnblogs.com/cenyuhai/p/3841000.html)

### Spark Streaming + Kafka ###

- [Spark踩坑记——Spark Streaming+Kafka](https://www.cnblogs.com/xlturing/p/6246538.html)

