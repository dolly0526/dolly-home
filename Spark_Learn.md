# Spark原理和源码 #
2019/11/17 18:18:24 

## 入门 ##
1. 小故事  
![](https://i.imgur.com/uGo3gu9.png)
2. RDD的理解  
![](https://i.imgur.com/7QDYWjB.png)

## 部署模式 ##
1. 用户在提交任务给 Spark 处理时，以下两个参数共同决定了 Spark 的运行方式。
 - master MASTER_URL：决定了 Spark 任务提交给哪种集群处理。
 - deploy-mode DEPLOY_MODE：决定了 Driver 的运行方式，可选值为Client 或者 Cluster。
2. 通用运行流程概述
 - 不论 Spark 以何种模式进行部署，任务提交后，都会先启动 Driver 进程，随后 Driver 进程向集群管理器注册应用程序，之后集群管理器根据此任务的配置文件分配 Executor 并启动，当 Driver 所需的资源全部满足后，Driver 开始执行 main 函数，Spark 查询为懒执行，当执行到 action 算子时开始反向推算，根据宽依赖进行 stage 的划分，随后每一个 stage 对应一个 taskset，taskset 中有多个 task，根据本地化原则，task 会被分发到指定的 Executor 去执行，在任务执行的过程中，Executor 也会不断与 Driver 进行通信，报告任务运行情况。
 - 图解  
![](https://i.imgur.com/LHVvkbv.png)

### YARN-Cluster模式 ###
1. YARN调度流程  
![](https://i.imgur.com/7kWlpbG.png)
2. 任务提交流程
 - 在 YARN Cluster 模式下，任务提交后会和 ResourceManager 通讯申请启动ApplicationMaster，随后 ResourceManager 分配 container，在合适的 NodeManager上启动 ApplicationMaster，此时的 ApplicationMaster 就是 Driver。Driver 启动后向 ResourceManager 申请 Executor 内存，ResourceManager 接到ApplicationMaster 的资源申请后会分配 container，然后在合适的 NodeManager 上启动 Executor 进程，Executor 进程启动后会向 Driver 反向注册，Executor 全部注册完成后 Driver 开始执行 main 函数，之后执行到 Action 算子时，触发一个 job，并根据宽依赖开始划分 stage，每个 stage 生成对应的 taskSet，之后将 task 分发到各个Executor上执行。
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
4. Spark源码中特殊的类
 - Backend: 后台
 - rpcEnv:  RPC
 - amEndpoint: 终端
 - RpcEndpointAddress: 终端地址
5. spark-submit源码解析


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
6. ApplicationMaster源码解析
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
7. CoarseGrainedExecutorBackend源码解析
 ```
1) CoarseGrainedExecutorBackend
    
    -- main
    
        -- run
        
            -- onStart
            
                -- ref.ask[Boolean](RegisterExecutor
            
            -- receive
            
                --  case RegisteredExecutor
                    -- new Executor
 ```
8. YARN部署Spark流程图  
![](https://i.imgur.com/dOqCRik.png)  
![](https://i.imgur.com/JOMFF8q.png)

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

## 任务调度机制 ##
1. 任务调度概述  
 - 当Driver起来后，Driver则会根据用户程序逻辑准备任务，并根据Executor资源情况逐步分发任务。在详细阐述任务调度前，首先说明下Spark里的几个概念。一个Spark应用程序包括Job、Stage以及Task三个概念：  
a. Job是以Action方法为界，遇到一个Action方法则触发一个Job；  
b. Stage是Job的子集，以RDD宽依赖(即Shuffle)为界，遇到Shuffle做一次划分；  
c. Task是Stage的子集，以并行度(分区数)来衡量，分区数是多少，则有多少个task。  
Spark的任务调度总体来说分两路进行，一路是Stage级的调度，一路是Task级的调度
 - 图解  
![](https://i.imgur.com/5eF7hXj.png)
 - Spark RDD通过其Transactions操作，形成了RDD血缘关系图，即DAG，最后通过Action的调用，触发Job并调度执行。DAGScheduler负责Stage级的调度，主要是将job切分成若干Stages，并将每个Stage打包成TaskSet交给TaskScheduler调度。TaskScheduler负责Task级的调度，将DAGScheduler给过来的TaskSet按照指定的调度策略分发到Executor上执行，调度过程中SchedulerBackend负责提供可用资源，其中SchedulerBackend有多种实现，分别对接不同的资源管理系统。
 - 图解  
![](https://i.imgur.com/APWvzg9.png)  
![](https://i.imgur.com/I66TTPy.png)
 - Driver初始化SparkContext过程中，会分别初始化DAGScheduler、TaskScheduler、SchedulerBackend以及HeartbeatReceiver，并启动SchedulerBackend以及HeartbeatReceiver。SchedulerBackend通过ApplicationMaster申请资源，并不断从TaskScheduler中拿到合适的Task分发到Executor执行。HeartbeatReceiver负责接收Executor的心跳信息，监控Executor的存活状况，并通知到TaskScheduler。
3. WordCount
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