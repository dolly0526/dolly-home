# ZooKeeper原理 #
2019/11/7 20:38:53 

## 参考资料 ##
- 《尚硅谷大数据技术之ZooKeeper》
- 《从Paxos到ZooKeeper：分布式一致性原理与实践》

## ZNode ##
### 数据结构 ###
![](https://i.imgur.com/tdR4YUW.png)

### 客户端操作 ###
1. 常用命令  
![](https://i.imgur.com/JJhSrlk.png)
2. 查看节点状态
 ```
[zk: localhost:2181(CONNECTED) 17] stat /test
cZxid = 0x100000003
ctime = Wed Aug 29 00:03:23 CST 2019
mZxid = 0x100000011
mtime = Wed Aug 29 00:21:23 CST 2019
pZxid = 0x100000014
cversion = 9
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 1
 ```

### 节点特性 ### 
1. 节点类型  
![](https://i.imgur.com/9LbhklP.png)
2. Stat结构体  
1）czxid-创建节点的事务zxid  
每次修改ZooKeeper状态都会收到一个zxid形式的时间戳，也就是ZooKeeper事务ID  
事务ID是ZooKeeper中所有修改总的次序。每个修改都有唯一的zxid，如果zxid1小于zxid2，那么zxid1在zxid2之前发生  
2）ctime - znode被创建的毫秒数(从1970年开始)  
3）mzxid - znode最后更新的事务zxid  
4）mtime - znode最后修改的毫秒数(从1970年开始)  
5）pZxid-znode最后更新的子节点zxid  
6）cversion - znode子节点变化号，znode子节点修改次数  
7）dataversion - znode数据变化号  
8）aclVersion - znode访问控制列表的变化号  
9）ephemeralOwner- 如果是临时节点，这个是znode拥有者的session id。如果不是临时节点则是0  
10）dataLength- znode的数据长度  
11）numChildren - znode子节点数量
3. **dataVersion**用于实现乐观锁机制中的写入检验, 类似CAS操作, 在`PrepRequestProcessor`类中, 每次处理`setDataRequest`请求时, 会进行如下的版本检查:
 ```
    version = setDataRequest.getVersion();
    int currentVersion = nodeRecord.stat.getVersion();
    if (version != -1 && version != currentVersion) {
        throw new KeeperException.BadVersionException(path);
    }
    version = currentVersion + 1;
 ```
![](https://i.imgur.com/FXvZHzb.png)

## HDFS-HA原理 ##
### HA概述 ###
1. 所谓HA（High Available），即高可用（7*24小时不中断服务）  
2. 实现高可用最关键的策略是消除单点故障。HA严格来说应该分成各个组件的HA机制：HDFS的HA和YARN的HA。  
3. Hadoop2.0之前，在HDFS集群中NameNode存在单点故障（SPOF）  
4. NameNode主要在以下两个方面影响HDFS集群  
 - NameNode机器发生意外，如宕机，集群将无法使用，直到管理员重启
 - NameNode机器需要升级，包括软件、硬件升级，此时集群也将无法使用
5. HDFS-HA功能通过配置Active/Standby两个NameNodes实现在集群中对NameNode的热备来解决上述问题。如果出现故障，如机器崩溃或机器需要升级维护，这时可通过此种方式将NameNode很快的切换到另外一台机器。

### 工作要点 ###  
1. 元数据管理方式需要改变  
 - 内存中各自保存一份元数据；  
 - Edits日志只有Active状态的NameNode节点可以做写操作  
 - 两个NameNode都可以读取Edits  
 - 共享的Edits放在一个共享存储中管理（qjournal和NFS两个主流实现）  
2. 需要一个状态管理功能模块  
 - 实现了一个zkfailover，常驻在每一个namenode所在的节点，每一个zkfailover负责监控自己所在NameNode节点，利用zk进行状态标识，当需要进行状态切换时，由zkfailover来负责切换，切换时需要防止brain split现象的发生
3. 必须保证两个NameNode之间能够ssh无密码登录
4. 隔离（Fence），即同一时刻有且只有一个NameNode对外提供服务

### 自动故障转移机制 ### 
1. 前面学习了使用命令`hdfs haadmin -failover`手动进行故障转移，在该模式下，即使现役NameNode已经失效，系统也不会自动从现役NameNode转移到待机NameNode，下面学习如何配置部署HA自动进行故障转移。自动故障转移为HDFS部署增加了两个新组件：ZooKeeper和ZKFailoverController（ZKFC）进程，如图3-20所示。ZooKeeper是维护少量协调数据，通知客户端这些数据的改变和监视客户端故障的高可用服务。HA的自动故障转移依赖于ZooKeeper的以下功能：  
 - 故障检测：集群中的每个NameNode在ZooKeeper中维护了一个持久会话，如果机器崩溃，ZooKeeper中的会话将终止，ZooKeeper通知另一个NameNode需要触发故障转移。  
 - 现役NameNode选择：ZooKeeper提供了一个简单的机制用于唯一的选择一个节点为active状态。如果目前现役NameNode崩溃，另一个节点可能从ZooKeeper获得特殊的排外锁以表明它应该成为现役NameNode。  
2. ZKFC是自动故障转移中的另一个新组件，是ZooKeeper的客户端，也监视和管理NameNode的状态。每个运行NameNode的主机也运行了一个ZKFC进程，ZKFC负责：  
 - 健康监测：ZKFC使用一个健康检查命令定期地ping与之在相同主机的NameNode，只要该NameNode及时地回复健康状态，ZKFC认为该节点是健康的。如果该节点崩溃，冻结或进入不健康状态，健康监测器标识该节点为非健康的。  
 - ZooKeeper会话管理：当本地NameNode是健康的，ZKFC保持一个在ZooKeeper中打开的会话。如果本地NameNode处于active状态，ZKFC也保持一个特殊的znode锁，该锁使用了ZooKeeper对短暂节点的支持，如果会话终止，锁节点将自动删除。  
 - 基于ZooKeeper的选择：如果本地NameNode是健康的，且ZKFC发现没有其它的节点当前持有znode锁，它将为自己获取该锁。如果成功，则它已经赢得了选择，并负责运行故障转移进程以使它的本地NameNode为Active。故障转移进程与前面描述的手动故障转移相似，首先如果必要保护之前的现役NameNode，然后本地NameNode转换为Active状态。  
3. 图解  
![](https://i.imgur.com/KayKSGx.png)
4. 搭建HDFS-HA, 对照3中的图解, 可知每个组件的作用  
![](https://i.imgur.com/kctcQtS.png)
5. 补充说明
 - JournaNode是一个轻量级的元数据文件系统, 可以和其他组件混部 
> JournalNode machines - the machines on which you run the JournalNodes. The JournalNode daemon is relatively lightweight, so these daemons may reasonably be collocated on machines with other Hadoop daemons, for example NameNodes, the JobTracker, or the YARN ResourceManager. Note: There must be at least 3 JournalNode daemons, since edit log modifications must be written to a majority of JNs. This will allow the system to tolerate the failure of a single machine. You may also run more than 3 JournalNodes, but in order to actually increase the number of failures the system can tolerate, you should run an odd number of JNs, (i.e. 3, 5, 7, etc.). Note that when running with N JournalNodes, the system can tolerate at most (N - 1) / 2 failures and continue to function normally.
 - JournalNode文件夹下大部分是edits文件, 依赖Paxos算法保持全局数据的一致性; NameNode复制JournalNode中的edits文件
 - 如果杀掉ZKFC, 则对应的NameNode和ZK失联, 会被另一个NameNode杀掉
 - ZK中的hadoop-ha下有两个节点: ActiveStandbyElectorLock和ActiveBreadCrumb
a. ActiveStandbyElectorLock是一个**临时**节点, 排他锁被活跃的那个NameNode占有  
![](https://i.imgur.com/EWSDugP.png)  
b. ActiveBreadCrumb是一个**持久**节点, NameNode在变成活跃状态时会在此节点中注册信息, 在正常下线时会删除此节点; 若非正常下线, 则另一个节点会依据此节点的信息对此节点进行隔离(fencing), 防止脑裂  
![](https://i.imgur.com/A2pAaWK.png)  
c. [hadoop HA机制](https://blog.csdn.net/liu812769634/article/details/53097268)  
d. 非常正常状态下的主备更替过程  
![](https://i.imgur.com/wKlIsom.png)
 - 如果某个节点失联(比如断网), 则另一个节点不会变为活跃; HA机制下**宁愿宕机也不能脑裂**, 需要人工干预; 以下为日志  
![](https://i.imgur.com/71T5gOL.png)  
![](https://i.imgur.com/yZ1N7gh.png)

## ZooKeeper技术内幕 ##
### 特点 ###
![](https://i.imgur.com/5GH4Hi1.png)

### 监听器原理 ###
1. 图解  
![](https://i.imgur.com/EXxxyaF.png)
2. 源码实现
 ```
    /**
     * To create a ZooKeeper client object, the application needs to pass a
     * connection string containing a comma separated list of host:port pairs,
     * each corresponding to a ZooKeeper server.
     * <p>
     * Session establishment is asynchronous. This constructor will initiate
     * connection to the server and return immediately - potentially (usually)
     * before the session is fully established. The watcher argument specifies
     * the watcher that will be notified of any changes in state. This
     * notification can come at any point before or after the constructor call
     * has returned.
     * <p>
     * The instantiated ZooKeeper client object will pick an arbitrary server
     * from the connectString and attempt to connect to it. If establishment of
     * the connection fails, another server in the connect string will be tried
     * (the order is non-deterministic, as we random shuffle the list), until a
     * connection is established. The client will continue attempts until the
     * session is explicitly closed.
     * <p>
     * Added in 3.2.0: An optional "chroot" suffix may also be appended to the
     * connection string. This will run the client commands while interpreting
     * all paths relative to this root (similar to the unix chroot command).
     *
     * @param connectString
     *            comma separated host:port pairs, each corresponding to a zk
     *            server. e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002" If
     *            the optional chroot suffix is used the example would look
     *            like: "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002/app/a"
     *            where the client would be rooted at "/app/a" and all paths
     *            would be relative to this root - ie getting/setting/etc...
     *            "/foo/bar" would result in operations being run on
     *            "/app/a/foo/bar" (from the server perspective).
     * @param sessionTimeout
     *            session timeout in milliseconds
     * @param watcher
     *            a watcher object which will be notified of state changes, may
     *            also be notified for node events
     * @param canBeReadOnly
     *            (added in 3.4) whether the created client is allowed to go to
     *            read-only mode in case of partitioning. Read-only mode
     *            basically means that if the client can't find any majority
     *            servers but there's partitioned server it could reach, it
     *            connects to one in read-only mode, i.e. read requests are
     *            allowed while write requests are not. It continues seeking for
     *            majority in the background.
     *
     * @throws IOException
     *             in cases of network failure
     * @throws IllegalArgumentException
     *             if an invalid chroot path is specified
     */
    public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            boolean canBeReadOnly)
        throws IOException
    {
        LOG.info("Initiating client connection, connectString=" + connectString
                + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);

        watchManager.defaultWatcher = watcher;

        ConnectStringParser connectStringParser = new ConnectStringParser(
                connectString);
        HostProvider hostProvider = new StaticHostProvider(
                connectStringParser.getServerAddresses());
		//底层创建两个子线程
        cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
                hostProvider, sessionTimeout, this, watchManager,
                getClientCnxnSocket(), canBeReadOnly);
		//子线程开始运行
        cnxn.start();
    }

    /**
     * Creates a connection object. The actual network connect doesn't get
     * established until needed. The start() instance method must be called
     * subsequent to construction.
     *
     * @param chrootPath - the chroot of this client. Should be removed from this Class in ZOOKEEPER-838
     * @param hostProvider
     *                the list of ZooKeeper servers to connect to
     * @param sessionTimeout
     *                the timeout for connections.
     * @param zooKeeper
     *                the zookeeper object that this connection is related to.
     * @param watcher watcher for this connection
     * @param clientCnxnSocket
     *                the socket implementation used (e.g. NIO/Netty)
     * @param sessionId session id if re-establishing session
     * @param sessionPasswd session passwd if re-establishing session
     * @param canBeReadOnly
     *                whether the connection is allowed to go to read-only
     *                mode in case of partitioning
     * @throws IOException
     */
    public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
            ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
            long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
        this.zooKeeper = zooKeeper;
        this.watcher = watcher;
        this.sessionId = sessionId;
        this.sessionPasswd = sessionPasswd;
        this.sessionTimeout = sessionTimeout;
        this.hostProvider = hostProvider;
        this.chrootPath = chrootPath;

        connectTimeout = sessionTimeout / hostProvider.size();
        readTimeout = sessionTimeout * 2 / 3;
        readOnly = canBeReadOnly;
		//生成两个子线程
        sendThread = new SendThread(clientCnxnSocket);
        eventThread = new EventThread();
    }

    public void start() {
		//connect
        sendThread.start();
		//listener
        eventThread.start();
    }
 ```
3. 关于两个子线程  
![](https://i.imgur.com/g1Jf6D6.png)  
![](https://i.imgur.com/RXtMN9o.png)

### 一致性协议 ###
#### Paxos算法 ####
1. Paxos算法一种基于消息传递且具有高度容错特性的一致性算法。
 - 分布式系统中的节点通信存在两种模型：共享内存（Shared memory）和消息传递（Messages passing）。基于消息传递通信模型的分布式系统，不可避免的会发生以下错误：进程可能会慢、被杀死或者重启，消息可能会延迟、丢失、重复，在基础 Paxos 场景中，先不考虑可能出现消息篡改即拜占庭错误的情况。
 - Paxos 算法解决的问题是在一个可能发生上述异常的分布式系统中如何就某个值达成一致，保证不论发生以上任何异常，都不会破坏决议的一致性。
 - 图解  
![](https://i.imgur.com/nCxbCcc.png)
2. Paxos算法流程中的每条消息描述如下：
 - Prepare: Proposer生成全局唯一且递增的Proposal ID (可使用时间戳加Server ID)，向所有Acceptors发送Prepare请求，这里无需携带提案内容，只携带Proposal ID即可。
 - Promise: Acceptors收到Prepare请求后，做出“两个承诺，一个应答”。  
两个承诺：  
a. 不再接受Proposal ID小于等于（注意：这里是<=）当前请求的Prepare请求。  
b. 不再接受Proposal ID小于（注意：这里是<）当前请求的Propose请求。  
一个应答：  
c.	不违背以前做出的承诺下，回复已经Accept过的提案中Proposal ID最大的那个提案的Value和Proposal ID，没有则返回空值。
 - Propose: Proposer 收到多数Acceptors的Promise应答后，从应答中选择Proposal ID最大的提案的Value，作为本次要发起的提案。如果所有应答的提案Value均为空值，则可以自己随意决定提案Value。然后携带当前Proposal ID，向所有Acceptors发送Propose请求。
 - Accept: Acceptor收到Propose请求后，在不违背自己之前做出的承诺下，接受并持久化当前Proposal ID和提案Value。
 - Learn: Proposer收到多数Acceptors的Accept后，决议形成，将形成的决议发送给所有Learners。
3. [Paxos算法细节详解(一)--通过现实世界描述算法](https://www.cnblogs.com/endsock/p/3480093.html)
4. 下面我们针对上述描述做三种情况的推演举例：为了简化流程，我们这里不设置Learner。
 - 情况1:  
![](https://i.imgur.com/YUdrCFO.png)
 - 情况2:  
![](https://i.imgur.com/W6kHupp.png)
 - Paxos算法缺陷：在网络复杂的情况下，一个应用Paxos算法的分布式系统，可能很久无法收敛，甚至陷入活锁的情况。
 - 情况3:  
![](https://i.imgur.com/7aEZZGJ.png)
 - 造成这种情况的原因是系统中有一个以上的Proposer，多个Proposers相互争夺Acceptors，造成迟迟无法达成一致的情况。针对这种情况，一种改进的Paxos算法被提出：从系统中选出一个节点作为Leader，只有Leader能够发起提案。这样，一次Paxos流程中只有一个Proposer，不会出现活锁的情况，此时只会出现例子中第一种情况。

#### ZAB协议 ####
黑话: 没有leader选leader(崩溃恢复), 有leader就干活(正常读写)

##### 选举机制 #####
1. 半数机制：集群中半数以上机器存活，集群可用。所以Zookeeper适合安装**奇数台**服务器。
2. Zookeeper虽然在配置文件中并没有指定Master和Slave。但是，Zookeeper工作时，是有一个节点为Leader，其他则为Follower，Leader是通过内部的选举机制临时产生的。
3. 以一个简单的例子来说明整个选举的过程。
 - 假设有五台服务器组成的Zookeeper集群，它们的id从1-5，同时它们都是最新启动的，也就是没有历史数据，在存放数据量这一点上，都是一样的。假设这些服务器依序启动，来看看会发生什么
 - 如图所示  
![](https://i.imgur.com/Aq9kTLJ.png)
 - 服务器1启动，发起一次选举。服务器1投自己一票。此时服务器1票数一票，不够半数以上（3票），选举无法完成，服务器1状态保持为**LOOKING**；
 - 服务器2启动，再发起一次选举。服务器1和2分别投自己一票并交换选票信息：此时服务器1发现服务器2的ID比自己目前投票推举的（服务器1）大，更改选票为推举服务器2。此时服务器1票数0票，服务器2票数2票，没有半数以上结果，选举无法完成，服务器1，2状态保持**LOOKING**；
 - 服务器3启动，发起一次选举。此时服务器1和2都会更改选票为服务器3。此次投票结果：服务器1为0票，服务器2为0票，服务器3为3票。此时服务器3的票数已经超过半数，服务器3当选Leader。服务器1，2更改状态为**FOLLOWING**，服务器3更改状态为**LEADING**；
 - 服务器4启动，发起一次选举。此时服务器1，2，3已经不是LOOKING状态，不会更改选票信息。交换选票信息结果：服务器3为3票，服务器4为1票。此时服务器4服从多数，更改选票信息为服务器3，并更改状态为**FOLLOWING**；
 - 服务器5启动，同4一样当小弟。
 - 

##### 写数据流程 #####
![](https://i.imgur.com/8B1OdTp.png)

##### Observer #####
[ZooKeeper 增加Observer部署模式提高性能](http://https://www.cnblogs.com/hongdada/p/8117677.html)

#### Raft协议 ####
简单了解: [说一说那些我也不太懂的 Raft 协议](https://www.jianshu.com/p/aa77c8f4cb5c)

## 分布式锁 ##
[分布式锁之zookeeper](https://www.jianshu.com/p/bee78409f122)  
[实战 -- Zookeeper实现分布式锁](https://blog.csdn.net/wangaiheng/article/details/82259211)  
[分布式锁的实现方案](https://www.jianshu.com/p/d987273d6dfb)