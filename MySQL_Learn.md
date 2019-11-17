# MySQL技术内幕 #
2019/11/16 15:03:00  

## 概述 ##
![](https://i.imgur.com/OyfH6U2.png)

## 逻辑架构 ##
### 总体概览 ###
和其它数据库相比，MySQL有点与众不同，它的架构可以在多种不同场景中应用并发挥良好作用。主要体现在存储引擎的架构上，插件式的存储引擎架构将查询处理和其它的系统任务以及数据的存储提取相分离。这种架构可以根据业务的需求和实际需要选择合适的存储引擎。  
 - 图解  
![](https://i.imgur.com/XoDWv93.png)  
 - 解释  
a. **连接层**:  
最上层是一些客户端和连接服务，包含本地sock通信和大多数基于客户端/服务端工具实现的类似于tcp/ip的通信。主要完成一些类似于连接处理、授权认证、及相关的安全方案。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于SSL的安全链接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。  
b.**服务层**:  
第二层架构主要完成大多少的核心服务功能，如SQL接口，并完成缓存的查询，SQL的分析和优化及部分内置函数的执行。所有跨存储引擎的功能也在这一层实现，如过程、函数等。在该层，服务器会解析查询并创建相应的内部解析树，并对其完成相应的优化如确定查询表的顺序，是否利用索引等，最后生成相应的执行操作。如果是select语句，服务器还会查询内部的缓存。如果缓存空间足够大，这样在解决大量读操作的环境中能够很好的提升系统的性能。  
c. **引擎层**:  
存储引擎层，存储引擎真正的负责了MySQL中数据的存储和提取，服务器通过API与存储引擎进行通信。不同的存储引擎具有的功能不同，这样我们可以根据自己的实际需要进行选取。后面介绍MyISAM和InnoDB  
d. **存储层**:  
数据存储层，主要是将数据存储在运行于裸设备的文件系统之上，并完成与存储引擎的交互。

### 查询说明 ###
- 首先，mysql的查询流程大致是：mysql客户端通过协议与mysql服务器建连接，发送查询语句，先检查查询缓存，如果命中，直接返回结果，否则进行语句解析
- 有一系列预处理，比如检查语句是否写正确了，然后是查询优化（比如是否使用索引扫描，如果是一个不可能的条件，则提前终止），生成查询计划，然后查询引擎启动，开始执行查询，从底层存储引擎调用API获取数据，最后返回给客户端。怎么存数据、怎么取数据，都与存储引擎有关。
- 然后，mysql默认使用的BTREE索引，并且一个大方向是，无论怎么折腾sql，至少在目前来说，mysql最多只用到表中的一个索引。

## 存储引擎 ##
### 查看命令 ###
![](https://i.imgur.com/JGliX4R.png)

### MyISAM和InnoDB ###
![](https://i.imgur.com/RQ00KMk.png)

### 阿里优化 ###
![](https://i.imgur.com/6oQHZRH.png)

## SQL变慢原因 ##
1. 查询语句写的烂
2. 索引失效(单值/复合)
3. 关联查询太多join(设计缺陷或不得已的需求)
4. 服务器调优及各个参数设置(缓冲/线程数等)

## SQL机读顺序 ##
![](https://i.imgur.com/YpXhCsH.png)  
![](https://i.imgur.com/uyilXT7.png)

## 七种JOIN ##
![](https://i.imgur.com/rpMpwfO.png)  
**注意:**  
MySQL不支持FULL OUTER JOIN, 需用UNION  
![](https://i.imgur.com/TkPcyVb.png)  
![](https://i.imgur.com/GeYUhS7.png)

## 索引 ##
### 是什么 ###
 - 官方定义: 索引(Index)是帮助MySQL高效获取数据的**数据结构**  
![](https://i.imgur.com/BioF5H9.png)
 - 简单理解: 排好序的快速查找数据结构(where和order by均受影响)  
a. 详解  
![](https://i.imgur.com/KMc61gX.png)  
b. 除数据本身之外, 数据库还维护着一个满足特定查找算法的数据结构, 这些数据结构以某种方式指向数据, 这样就可以在这些数据结构的基础上实现高级查找算法, 这种数据结构就是索引
 - 一般来说索引本身也很大, 不可能全部存储在内存中, 因此索引往往以索引文件的形式存储在磁盘上
 - 我们平常所说的索引，如果没有特别指明，都是指B+树结构组织的索引。其中聚集索引，次要索引，覆盖索引，复合索引，前缀索引，唯一索引默认都是使用B+树索引，统称索引。当然，除了B+树这种类型的索引之外，还有哈稀索引(hash index)等。

### 优势 ###
1. 类似大学图书馆建书目索引，提高数据检索的效率，降低数据库的IO成本
2. 通过索引列对数据进行排序，降低数据排序的成本，降低了CPU的消耗

### 劣势 ###
1. 实际上索引也是一张表，该表保存了主键与索引字段，并指向实体表的记录，所以索引列也是要占用空间的
2. 虽然索引大大提高了查询速度，同时却会降低更新表的速度，如对表进行INSERT、UPDATE和DELETE。因为更新表时，MySQL不仅要保存数据，还要保存一下索引文件每次更新添加了索引列的字段，都会调整因为更新所带来的键值变化后的索引信息
3. 索引只是提高效率的一个因素，如果你的MySQL有大数据量的表，就需要花时间研究建立最优秀的索引，或优化查询语句

### 分类 ###
1. **单值索引**: 即一个索引只包含单个列，一个表可以有多个单列索引
2. **唯一索引**: 索引列的值必须唯一，但允许有空值
3. **复合索引**: 即一个索引包含多个列
4. 基本语法
 - 创建:   
a. CREATE  [UNIQUE ] INDEX indexName ON mytable(columnname(length));   
b. 如果是CHAR，VARCHAR类型，length可以小于字段实际长度；
如果是BLOB和TEXT类型，必须指定length。  
c. ALTER mytable ADD  [UNIQUE ]  INDEX [indexName] ON (columnname(length)) 
 - 删除: DROP INDEX [indexName] ON mytable; 
 - 查看: SHOW INDEX FROM table_name;
 - 使用ALTER命令:   
a. ALTER TABLE tbl_name ADD PRIMARY KEY (column_list): 该语句添加一个主键，这意味着索引值必须是唯一的，且不能为NULL。  
b. ALTER TABLE tbl_name ADD UNIQUE index_name(column_list): 这条语句创建索引的值必须是唯一的（除了NULL外，-NULL可能会出现多次）。  
c. ALTER TABLE tbl_name ADD INDEX index_name (column_list): 添加普通索引，索引值可出现多次。  
d. ALTER TABLE tbl_name ADD FULLTEXT index_name (column_list):该语句指定了索引为 FULLTEXT，用于全文索引。

### 结构 ###
0. 参考资料: 
 - [BTree和B+Tree详解](https://www.cnblogs.com/vianzhang/p/7922426.html)
 - [MySQL索引原理及BTree（B-/+Tree）结构详解](https://blog.csdn.net/u013967628/article/details/84305511)
 - [记一次腾讯面试：有了二叉查找树、平衡树（AVL）为啥还需要红黑树？](https://zhuanlan.zhihu.com/p/72505589)
1. B+Tree索引
 - 图解检索原理  
![](https://i.imgur.com/372TIhV.png)
 - 初始化介绍  
一颗b+树，浅蓝色的块我们称之为一个磁盘块，可以看到每个磁盘块包含几个数据项（深蓝色所示）和指针（黄色所示），  
如磁盘块1包含数据项17和35，包含指针P1、P2、P3，  
P1表示小于17的磁盘块，P2表示在17和35之间的磁盘块，P3表示大于35的磁盘块。  
真实的数据存在于叶子节点即3、5、9、10、13、15、28、29、36、60、75、79、90、99。  
**非叶子节点只不存储真实的数据，只存储指引搜索方向的数据项**，如17、35并不真实存在于数据表中。
 - 查找过程  
如果要查找数据项29，那么首先会把磁盘块1由磁盘加载到内存，此时发生一次IO，在内存中用二分查找确定29在17和35之间，锁定磁盘块1的P2指针，内存时间因为非常短（相比磁盘的IO）可以忽略不计，通过磁盘块1的P2指针的磁盘地址把磁盘块3由磁盘加载到内存，发生第二次IO，29在26和30之间，锁定磁盘块3的P2指针，通过指针加载磁盘块8到内存，发生第三次IO，同时内存中做二分查找找到29，结束查询，总计三次IO。
 - 真实的情况是，3层的b+树可以表示上百万的数据，如果上百万的数据查找只需要三次IO，性能提高将是巨大的，如果没有索引，每个数据项都要发生一次IO，那么总共需要百万次的IO，显然成本非常非常高。
2. 其他: Hash索引, Full-Text全文索引, R-Tree索引

### 适合建索引 ###
1. 主键自动建立唯一索引
2. 频繁作为查询条件的字段应该创建索引
3. 查询中与其它表关联的字段，外键关系建立索引
4. 频繁更新的字段不适合创建索引: 因为每次更新不单单是更新了记录还会更新索引，加重了IO负担
5. Where条件里用不到的字段不创建索引
6. 单键/组合索引的选择问题，who？(在高并发下倾向创建组合索引)
7. 查询中排序的字段，排序字段若通过索引去访问将大大提高排序速度
8. 查询中统计或者分组字段

### 不适合建索引 ###
1. 表记录太少
2. 经常增删改的表 (Why: 提高了查询速度，同时却会降低更新表的速度，如对表进行INSERT、UPDATE和DELETE。因为更新表时，MySQL不仅要保存数据，还要保存一下索引文件)
3. 数据重复且分布平均的表字段，因此应该只为最经常查询和最经常排序的数据列建立索引。注意，如果某个数据列包含许多重复的内容，为它建立索引就没有太大的实际效果。

## 性能分析 ##
### MySql Query Optimizer ###
![](https://i.imgur.com/59EZuqs.png)

### 常见瓶颈 ###
1. CPU：CPU在饱和的时候一般发生在数据装入内存或从磁盘上读取数据时候
2. IO：磁盘I/O瓶颈发生在装入数据远大于内存容量的时候
3. 服务器硬件的性能瓶颈：top, free, iostat和vmstat来查看系统的性能状态

### EXPLAIN ###
1. 是什么 (查看执行计划)
 - 使用**EXPLAIN**关键字可以模拟优化器执行SQL查询语句，从而知道MySQL是如何处理你的SQL语句的。分析你的查询语句或是表结构的性能瓶颈
 - 官网介绍  
![](https://i.imgur.com/bq7Euaq.png)
2. 能干嘛  
 - 表的读取顺序
 - 数据读取操作的操作类型
 - 哪些索引可以使用
 - 哪些索引被实际使用
 - 表之间的引用
 - 每张表有多少行被优化器查询
3. 怎么玩
 - EXPLAIN + SQL语句
 - 执行计划包含的信息  
![](https://i.imgur.com/12Fi215.png)
4. 各字段解释
 - **id**: select查询的序列号,包含一组数字，表示查询中执行select子句或操作表的顺序, 有三种情况:   
a. id相同，执行顺序**由上至下**  
![](https://i.imgur.com/wjgAqeL.png)  
b. id不同，如果是子查询，id的序号会**递增，id值越大优先级越高**，越先被执行  
![](https://i.imgur.com/epw8PyD.png)  
c. id相同又不同，**同时存在**  
![](https://i.imgur.com/j52vTaq.png)
 - select_type  
a. 有哪些  
![](https://i.imgur.com/HfzUhLM.png)  
b. 查询的类型，主要是用于区别普通查询、联合查询、子查询等的复杂查询  
![](https://i.imgur.com/ve60rcD.png)
 - table: 显示这一行的数据是关于哪张表的
 - **type**  
a. 有哪些  
![](https://i.imgur.com/N0eWFKj.png)  
b. 访问类型排列  
![](https://i.imgur.com/BYNpkda.png)  
c. 显示查询使用了何种类型，从最好到最差依次是：system > const > eq_ref > ref > range > index > ALL  
![](https://i.imgur.com/3oh8MIv.png)
 - possible_keys: 显示可能应用在这张表中的索引，一个或多个。  
注: 查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询实际使用
 - **key**: 实际使用的索引。如果为NULL，则没有使用索引  
注: 查询中若使用了**覆盖索引**，则该索引仅出现在key列表中  
![](https://i.imgur.com/otZPxRh.png)
 - key_len: 表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精确性的情况下，长度越短越好  
注: key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的  
![](https://i.imgur.com/E5b8Enx.png)
 - ref: 显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值  
![](https://i.imgur.com/WIt0yQg.png)
 - **rows**: 根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数  
![](https://i.imgur.com/agDGhUj.png)
 - **Extra**: 包含不适合在其他列中显示但十分重要的额外信息  
a. **Using filesort**: 说明mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。MySQL中无法利用索引完成的排序操作称为“文件排序”  
![](https://i.imgur.com/cQSdvdA.png)  
b. **Using temporary**: 使了用临时表保存中间结果,MySQL在对查询结果排序时使用临时表。常见于排序 order by 和分组查询 group by。  
![](https://i.imgur.com/hsvhEZg.png)  
c. **Using index**: 表示相应的select操作中使用了覆盖索引(Covering Index)，避免访问了表的数据行，效率不错！如果同时出现using where，表明索引被用来执行索引键值的查找; 如果没有同时出现using where，表明索引用来读取数据而非执行查找动作。  
![](https://i.imgur.com/XHj0pBc.png)  
**覆盖索引(Covering Index)**   
![](https://i.imgur.com/t8pyPSg.png)
d. Using where: 表明使用了where过滤   
e. Using join buffer: 使用了连接缓存
f. Impossible where: where子句的值总是false，不能用来获取任何元组  
![](https://i.imgur.com/LPdRSDD.png)  
g. Select tables optimized away: 在没有GROUP BY子句的情况下，基于索引优化MIN/MAX操作或者对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化。  
h. Distinct: 优化distinct操作，在找到第一匹配的元组后即停止找同样值的动作
5. 热身Case  
![](https://i.imgur.com/zVVaw1U.png)

### 索引优化 ###
