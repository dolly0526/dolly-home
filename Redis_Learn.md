# Redis技术内幕 #
2019/11/17 1:59:46  

## NoSQL ##
### ACID ###
![](https://i.imgur.com/pV1Jj1W.png)

### CAP ###
1. CAP
 - **C**onsistency（强一致性）: 在分布式系统中的所有数据备份，在同一时刻是否同样的值。（等同于所有节点访问同一份最新的数据副本）
 - **A**vailability（可用性）: 在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求。（对数据更新具备高可用性）
 - **P**artition tolerance（分区容错性）: 以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。
2. CAP的3进2
![](https://i.imgur.com/t7zPyd2.png)  
![](https://i.imgur.com/Pz7F5oM.png)  
![](https://i.imgur.com/3RJbXRF.png)
3. 经典CAP图  
![](https://i.imgur.com/PvIpnzF.png)  
![](https://i.imgur.com/8SJFLyH.jpg)

### BASE ###
![](https://i.imgur.com/pG9tVSx.png)

## Redis入门 ##
1. 入门概述
 - 是什么: Redis: **RE**mote **DI**ctionary **S**erver (远程字典服务器)
 - 是完全开源免费的，用C语言编写的，遵守BSD协议，是一个高性能的(key/value)分布式内存数据库，基于内存运行并支持持久化的NoSQL数据库，是当前最热门的NoSql数据库之一, 也被人们称为数据结构服务器
 - Redis 与其他 key - value 缓存产品有以下三个特点:  
a. Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用  
b. Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储  
c. Redis支持数据的备份，即master-slave模式的数据备份
2. 能干嘛  
![](https://i.imgur.com/pVl1j4f.png)

## Redis数据类型 ##
