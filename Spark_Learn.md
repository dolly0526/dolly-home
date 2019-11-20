# Spark原理和源码 #
2019/11/17 18:18:24 

## 入门 ##
1. 小故事  
![](https://i.imgur.com/uGo3gu9.png)
2. WordCount
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

## 部署流程 ##
1. YARN调度流程  
![](https://i.imgur.com/7kWlpbG.png)
2. 演示指令  
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
3. YARN部署Spark流程图  
![](https://i.imgur.com/dOqCRik.png)
4. spark-submit源码解析
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
5. ApplicationMaster源码解析
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
6. 
7. YARN部署Spark流程图, 源码解析  
![](https://i.imgur.com/JOMFF8q.png)