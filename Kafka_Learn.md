# Kafka原理 #
2019/11/17 18:20:19 

## 概述 ##
0. 参考资料
 - [Kafka简介](https://www.cnblogs.com/BYRans/p/6054930.html)
 - [Kafka【入门】就这一篇!](https://mp.weixin.qq.com/s/opAYVXIJoy4tCWaPcX5u6g)
1. 基础架构  
![](https://i.imgur.com/IeQLJ1V.png)  
 - Producer：消息生产者，就是向kafka broker发消息的客户端；
 - Consumer：消息消费者，向kafka broker取消息的客户端；
 - Consumer Group（CG）：消费者组，由多个consumer组成。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个消费者消费；消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。
 - Broker：一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个topic。
 - Topic：可以理解为一个队列，生产者和消费者面向的都是一个topic；
 - Partition：为了实现扩展性，一个非常大的topic可以分布到多个broker（即服务器）上，一个topic可以分为多个partition，每个partition是一个有序的队列；
 - Replica：副本，为保证集群中的某个节点发生故障时，该节点上的partition数据不丢失，且kafka仍然能够继续工作，kafka提供了副本机制，一个topic的每个分区都有若干个副本，一个leader和若干个follower
 - leader：每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是leader
 - follower：每个分区多个副本中的“从”，实时从leader中同步数据，保持和leader数据的同步。leader发生故障时，某个follower会成为新的follower

## 架构深入 ##
0. 参考资料
 - [Kafka消费组(consumer group)](https://www.cnblogs.com/huxi2b/p/6223228.html)
 - [什么是kafka同步生产者，什么是kafka异步生产者？](https://www.cnblogs.com/zlslch/p/6764742.html)
 - [kafka源码解析之十六生产者流程(客户端如何向topic发送数据)](https://blog.csdn.net/wl044090432/article/details/51124779)
 - [Kafka设计解析（八）- Exactly Once语义与事务机制原理](https://www.cnblogs.com/jasongj/p/7912348.html)
 - [Kafka的零拷贝技术](https://www.jianshu.com/p/835ec2d4c170)
 - [Kafka之Producer端](https://www.jianshu.com/u/81c228fd747c)
 - [直击Kafka的心脏——控制器](https://blog.csdn.net/u013256816/article/details/80865540)
1. 工作流程及文件存储机制  
 - Kafka中消息是以topic进行分类的，生产者生产消息，消费者消费消息，都是面向topic的。  
![](https://i.imgur.com/AItkADk.png)
 - topic是逻辑上的概念，而partition是物理上的概念，每个partition对应于一个log文件，该log文件中存储的就是producer生产的数据。Producer生产的数据会被不断追加到该log文件末端，且每条数据都有自己的offset。消费者组中的每个消费者，都会实时记录自己消费到了哪个offset，以便出错恢复时，从上次的位置继续消费。  
![](https://i.imgur.com/cOOpfHH.png)
 - 由于生产者生产的消息会不断追加到log文件末尾，为防止log文件过大导致数据定位效率低下，Kafka采取了分片和索引机制，将每个partition分为多个segment。每个segment对应两个文件——“.index”文件和“.log”文件。这些文件位于一个文件夹下，该文件夹的命名规则为：topic名称+分区序号。例如，first这个topic有三个分区，则其对应的文件夹为first-0,first-1,first-2。  

 ```
00000000000000000000.index
00000000000000000000.log
00000000000000170410.index
00000000000000170410.log
00000000000000239430.index
00000000000000239430.log
 ```
 - index和log文件以当前segment的第一条消息的offset命名。下图为index文件和log文件的结构示意图。  
![](https://i.imgur.com/HsaM2FF.png)
 - “.index”文件存储大量的索引信息，“.log”文件存储大量的数据，索引文件中的元数据指向对应数据文件中message的物理偏移地址。
2. 生产者
 - ack应答机制  
对于某些不太重要的数据，对数据的可靠性要求不是很高，能够容忍数据的少量丢失，所以没必要等ISR中的follower全部接收成功。  
所以Kafka为用户提供了三种可靠性级别，用户根据对可靠性和延迟的要求进行权衡，选择以下的配置。  
acks参数配置：  
acks：  
0：producer不等待broker的ack，这一操作提供了一个最低的延迟，broker一接收到还没有写入磁盘就已经返回，当broker故障时有可能丢失数据；  
1：producer等待broker的ack，partition的leader落盘成功后返回ack，如果在follower同步成功之前leader故障，那么将会丢失数据；  
-1（all）：producer等待broker的ack，partition的leader和follower全部落盘成功后才返回ack。但是如果在follower同步完成后，broker发送ack之前，leader发生故障，那么会造成数据重复。
3. Exactly Once: 在0.11版本之后，Kafka引入了幂等性机制（idempotent），配合acks = -1时的at least once语义，实现了producer到broker的exactly once语义。使用时，只需将enable.idempotence属性设置为true，kafka自动将acks属性设为-1。

## P6面试题 ##
![](https://i.imgur.com/DSpBRe8.png)