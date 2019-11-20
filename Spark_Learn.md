# Spark原理和源码 #
2019/11/17 18:18:24 

## 入门 ##
1. 小故事  
![](https://i.imgur.com/uGo3gu9.png)
2. WordCount
![](https://i.imgur.com/pKP1hx2.png)  
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
2. SparkSubmit类
 ```
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
 ```