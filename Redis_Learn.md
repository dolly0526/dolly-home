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

## 数据类型 ##
常见操作命令: [Http://redisdoc.com/](Http://redisdoc.com/)

### key ###
![](https://i.imgur.com/WwImq8h.jpg)

### String ###
- string是redis最基本的类型，你可以理解成与Memcached一模一样的类型，一个key对应一个value。
- string类型是二进制安全的。意思是redis的string可以包含任何数据。比如jpg图片或者序列化的对象 。
- string类型是Redis最基本的数据类型，一个redis中字符串value最多可以是512M
- 常用  
![](https://i.imgur.com/VPVLZWm.jpg)  
![](https://i.imgur.com/DJWiQ4B.jpg)

### List ###
- Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素导列表的头部（左边）或者尾部（右边）。
- 它的底层实际是个链表
- 常用  
![](https://i.imgur.com/un6ATYk.jpg)  
![](https://i.imgur.com/odsWdx1.jpg)

### Set ###
- Redis的Set是string类型的无序集合。它是通过HashTable实现实现的
- 常用  
![](https://i.imgur.com/HHvHtTO.jpg)

### Hash ###
- Redis hash 是一个键值对集合
- Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象
- 类似Java里面的Map<String, Object>
- 常用  
![](https://i.imgur.com/xUwMmYD.jpg)

### Zset ###
- sorted set：有序集合
- Redis zset 和 set 一样也是string类型元素的集合, 且不允许重复的成员, 不同的是每个元素都会关联一个double类型的分数。
- Redis正是通过分数来为集合中的成员进行从小到大的排序。zset的成员是唯一的, 但分数(score)却可以重复。
- 常用  
![](https://i.imgur.com/00DvzXj.jpg)  
![](https://i.imgur.com/CwXFRId.jpg)

## 配置文件 ##
redis.conf
### LIMITS ###
1. Maxmemory-policy  
（1）volatile-lru：使用LRU算法移除key，只对设置了过期时间的键  
（2）allkeys-lru：使用LRU算法移除key  
（3）volatile-random：在过期集合中移除随机的key，只对设置了过期时间的键  
（4）allkeys-random：移除随机的key  
（5）volatile-ttl：移除那些TTL值最小的key，即那些最近要过期的key  
（6）noeviction (默认)：不进行移除。针对写操作，只是返回错误信息
2. Maxmemory-samples  
![](https://i.imgur.com/waEZyDh.png)

## P6面试题 ##
![](https://i.imgur.com/XaNeGcO.png)  
![](https://i.imgur.com/d35Cc4I.png)