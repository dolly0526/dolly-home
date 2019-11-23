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
![](https://i.imgur.com/m0Kpxr6.jpg)

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

- 参考: [跳表和红黑树和二叉树](https://blog.csdn.net/lusic01/article/details/92001898)

- sorted set：有序集合
- Redis zset 和 set 一样也是string类型元素的集合, 且不允许重复的成员, 不同的是每个元素都会关联一个double类型的分数。
- Redis正是通过分数来为集合中的成员进行从小到大的排序。zset的成员是唯一的, 但分数(score)却可以重复。
- 常用  
![](https://i.imgur.com/00DvzXj.jpg)  
![](https://i.imgur.com/CwXFRId.jpg)

## 配置文件 ##
redis.conf
### LIMITS ###
1. Maxclients  
![](https://i.imgur.com/vmMmtJT.png)
2. Maxmemory  
![](https://i.imgur.com/A0Fx1HZ.png)
3. Maxmemory-policy  
![](https://i.imgur.com/AdeTcl1.png)
4. Maxmemory-samples  
![](https://i.imgur.com/waEZyDh.png)

### 常用配置 ###
参数说明
redis.conf 配置项说明如下：
1. Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
 - daemonize no
2. 当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid文件，可以通过pidfile指定
 - pidfile /var/run/redis.pid
3. 指定Redis监听端口，默认端口为6379，作者在自己的一篇博文中解释了为什么选用6379作为默认端口，因为6379在手机按键上MERZ对应的号码，而MERZ取自意大利歌女Alessia Merz的名字  
 - port 6379
4. 绑定的主机地址
 - bind 127.0.0.1
5. 当客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
 - timeout 300
6. 指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，默认为verbose
 - loglevel verbose
7. 日志记录方式，默认为标准输出，如果配置Redis为守护进程方式运行，而这里又配置为日志记录方式为标准输出，则日志将会发送给/dev/null
 - logfile stdout
8. 设置数据库的数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id
 - databases 16
9. 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
 - save [seconds] [changes]  
 - Redis默认配置文件中提供了三个条件：  
    save 900 1  
    save 300 10  
    save 60 10000  
    分别表示900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改。
10. 指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，可以关闭该选项，但会导致数据库文件变的巨大
 - rdbcompression yes
11. 指定本地数据库文件名，默认值为dump.rdb
 - dbfilename dump.rdb
12. 指定本地数据库存放目录
 - dir ./
13. 设置当本机为slav服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步
 - slaveof <masterip> <masterport>
14. 当master服务设置了密码保护时，slav服务连接master的密码
 - masterauth <master-password>
15. 设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过AUTH <password>命令提供密码，默认关闭
 - requirepass foobared
16. 设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息
 - maxclients 128
17. 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，Value会存放在swap区
 - maxmemory <bytes>
18. 指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为 redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no
 - appendonly no
19. 指定更新日志文件名，默认为appendonly.aof
 -  appendfilename appendonly.aof
20. 指定更新日志条件，共有3个可选值： 
 - no：表示等操作系统进行数据缓存同步到磁盘（快） 
 - always：表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全） 
 - everysec：表示每秒同步一次（折衷，默认值）, appendfsync everysec
21. 指定是否启用虚拟内存机制，默认值为no，简单的介绍一下，VM机制将数据分页存放，由Redis将访问量较少的页即冷数据swap到磁盘上，访问多的页面由磁盘自动换出到内存中（在后面的文章我会仔细分析Redis的VM机制）
 -  vm-enabled no
22. 虚拟内存文件路径，默认值为/tmp/redis.swap，不可多个Redis实例共享
 -  vm-swap-file /tmp/redis.swap
23. 将所有大于vm-max-memory的数据存入虚拟内存,无论vm-max-memory设置多小,所有索引数据都是内存存储的(Redis的索引数据 就是keys),也就是说,当vm-max-memory设置为0的时候,其实是所有value都存在于磁盘。默认值为0
 -  vm-max-memory 0
24. Redis swap文件分成了很多的page，一个对象可以保存在多个page上面，但一个page上不能被多个对象共享，vm-page-size是要根据存储的 数据大小来设定的，作者建议如果存储很多小对象，page大小最好设置为32或者64bytes；如果存储很大大对象，则可以使用更大的page，如果不 确定，就使用默认值
 -  vm-page-size 32
25. 设置swap文件中的page数量，由于页表（一种表示页面空闲或使用的bitmap）是在放在内存中的，，在磁盘上每8个pages将消耗1byte的内存。
 -  vm-pages 134217728
26. 设置访问swap文件的线程数,最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟。默认值为4
 -  vm-max-threads 4
27. 设置在向客户端应答时，是否把较小的包合并为一个包发送，默认为开启
 - glueoutputbuf yes
28. 指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法
 - hash-max-zipmap-entries 64
 - hash-max-zipmap-value 512
29. 指定是否激活重置哈希，默认为开启（后面在介绍Redis的哈希算法时具体介绍）
 - activerehashing yes
30. 指定包含其它的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各个实例又拥有自己的特定配置文件
 - include /path/to/local.conf

## 持久化 ##
### RDB（Redis DataBase） ###
1. 是什么: 在指定的时间间隔内将内存中的数据集**快照**写入磁盘，也就是行话讲的Snapshot快照，它恢复时是将快照文件直接读到**内存**里
 - Redis会单独创建（fork）一个**子进程**来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方式要比AOF方式更加的高效。RDB的缺点是最后一次持久化后的数据可能丢失。
2. Fork: fork的作用是复制一个与当前进程一样的进程。新进程的所有数据（变量、环境变量、程序计数器等）数值都和原进程一致，但是是一个全新的进程，并作为原进程的子进程
3. Rdb保存的是dump.rdb文件
4. 配置位置  
![](https://i.imgur.com/BXtesxG.jpg)
5. 优势:  
a. 适合大规模的数据恢复  
b. 对数据完整性和一致性要求不高
6. 劣势:  
a. 在一定间隔时间做一次备份，所以如果redis意外down掉的话，就会丢失最后一次快照后的所有修改  
b. Fork的时候，内存中的数据被克隆了一份，大致2倍的膨胀性需要考虑
7. 总结  
![](https://i.imgur.com/pP5bYTm.png)

### AOF（Append Only File） ###
1. 是什么: 以**日志**的形式来记录每个写操作，将Redis执行过的所有写指令记录下来(读操作不记录)，只许追加文件但不可以改写文件，redis启动之初会读取该文件重新构建数据，换言之，redis重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作
2. AOF保存的是**appendonly.aof**文件
3. 配置位置  
![](https://i.imgur.com/zjHGpui.jpg)
4. 优势:  
	a. 每修改同步：appendfsync always		同步持久化 每次发生数据变更会被立即记录到磁盘  性能较差但数据完整性比较好  
	b. 每秒同步：appendfsync everysec	异步操作，每秒记录   如果一秒内宕机，有数据丢失  
	c. 不同步：appendfsync no	从不同步
5. 劣势:  
a. 相同数据集的数据而言aof文件要远大于RDB文件，恢复速度慢于RDB  
b. AOF运行效率要慢于rdb,每秒同步策略效率较好，不同步效率和RDB相同
6. 总结  
![](https://i.imgur.com/nknop9M.png)

#### Rewrite ####
1. 是什么: AOF采用文件追加方式，文件会越来越大, 为避免出现此种情况，新增了重写机制, 当AOF文件的大小超过所设定的**阈值**时，Redis就会启动aof文件的内容压缩，只保留可以恢复数据的最小指令集.可以使用命令bgrewriteaof
2. 重写原理: aof文件持续增长而过大时，会**fork**出一条新进程来将文件重写(也是先写临时文件最后再rename)，遍历新进程的内存中数据，每条记录有一条的Set语句。重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件，这点和快照有点类似
3. 触发机制: Redis会记录上次重写时的aof大小，默认配置是当aof文件大小是上次rewrite后大小的一倍且文件大于64M时触发

### 总结 ###
1. RDB持久化方式能够在指定的时间间隔能对你的数据进行快照存储
2. AOF持久化方式记录每次对服务器写的操作,当服务器重启的时候会重新执行这些命令来恢复原始的数据,AOF命令以redis协议追加保存每次写的操作到文件末尾. Redis还能对AOF文件进行后台重写,使得AOF文件的体积不至于过大
3. 只做缓存：如果你只希望你的数据在服务器运行的时候存在,你也可以不使用任何持久化方式.
4. 同时开启两种持久化方式
 - 在这种情况下,当redis重启的时候会优先载入AOF文件来恢复原始的数据, 因为在通常情况下AOF文件保存的数据集要比RDB文件保存的数据集要完整.
 - RDB的数据不实时，同时使用两者时服务器重启也只会找AOF文件。那要不要只使用AOF呢？作者建议不要，因为RDB更适合用于备份数据库(AOF在不断变化不好备份)，快速重启，而且不会有AOF可能潜在的bug，留着作为一个万一的手段。
5. 性能建议  
![](https://i.imgur.com/rOtDt7M.png)

## 事务 ##
1. 是什么: 可以一次执行多个命令，本质是一组命令的集合。一个事务中的所有命令都会序列化，按顺序地串行化执行而不会被其它命令插入，不许加塞
2. 能干嘛: 一个队列中，一次性、顺序性、排他性的执行一系列命令
3. 3阶段
 - 开启：以MULTI开始一个事务
 - 入队：将多个命令入队到事务中，接到这些命令并不会立即执行，而是放到等待执行的事务队列里面
 - 执行：由EXEC命令触发事务
4. 3特性
 - 单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
 - 没有隔离级别的概念：队列中的命令没有提交之前都不会实际的被执行，因为事务提交前任何指令都不会被实际执行，也就不存在”事务内的查询要看到事务里的更新，在事务外查询不能看到”这个让人万分头痛的问题
 - 不保证原子性：redis同一个事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚

## 主从复制 ##
1. 是什么: 行话, 也就是我们所说的主从复制，主机数据更新后根据配置和策略，自动同步到备机的master/slaver机制，Master以写为主，Slave以读为主
2. 能干嘛: 读写分离, 容灾恢复
3. 常用3招
 - 一主二仆: 一个Master两个Slave
 - 薪火相传: 上一个Slave可以是下一个slave的Master，Slave同样可以接收其他slaves的连接和同步请求，那么该slave作为了链条中下一个的master, 可以有效减轻master的写压力
 - 反客为主: 使当前数据库停止与其他数据库的同步，转成主数据库(SLAVEOF no one)
4. 复制原理
 - Slave启动成功连接到master后会发送一个sync命令
 - Master接到命令启动后台的存盘进程，同时收集所有接收到的用于修改数据集命令，在后台进程执行完毕之后，master将传送整个数据文件到slave, 以完成一次完全同步
 - 全量复制：而slave服务在接收到数据库文件数据后，将其存盘并加载到内存中
 - 增量复制：Master继续将新的所有收集到的修改命令依次传给slave, 完成同步
 - 但是只要是重新连接master, 一次完全同步（全量复制）将被自动执行
5. **哨兵模式(sentinel)**
 - 是什么: 反客为主的自动版，能够后台监控主机是否故障，如果故障了根据投票数自动将从库转换为主库
 - 原有的master挂了, 投票新选, 之后重新主从继续开工  
![](https://i.imgur.com/2bMUcp5.jpg)
 - 问题：如果之前的master重启回来，会不会双master冲突？不会, 会成为slave  
![](https://i.imgur.com/kDlhrIB.jpg)
 - 一组sentinel能同时监控多个Master
6. 复制的缺点: 由于所有的写操作都是先在Master上操作，然后同步更新到Slave上，所以从Master同步到Slave机器有一定的延迟，当系统很繁忙的时候，延迟问题会更加严重，Slave机器数量的增加也会使这个问题更加严重。(复制延时)

## P6面试题 ##
![](https://i.imgur.com/XaNeGcO.png)  
![](https://i.imgur.com/d35Cc4I.png)