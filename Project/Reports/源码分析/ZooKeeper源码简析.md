

## ZooKeeper 源码简析

严贝 2019K8009937001

本文为对 ZooKeeper 项目源码的学习简析报告，其中将重点关注 Leader 选举机制的实现。

分析使用的版本号为 ZooKeeper3.6.3

源码仓库：https://github.com/apache/zookeeper

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image1.png?raw=true" style="zoom: 60%;" />

### 一、ZooKeeper简介

#### 1.1 什么是 ZooKeeper ？

ZooKeeper 诞生于 Yahoo，为了解决内部系统的分布式单点问题，Yahoo 需要建立一个类似系统来进行分布式协调。而其内部系统都以动物命名，用于管理它们的系统 ZooKeeper 因此得名。

具体来说，ZooKeeper 是一个在分布式系统中协作多个任务的工具。它提供适用于分布式应用程序的高性能协调服务，目标是将那些复杂且容易出错的分布式一致性服务封装起来，构成一个高效可靠的原语集，并以一系列简单易用的接口提供给用户使用。它的出现大大降低了分布式应用程序的开发难度和工作量，让程序员可以专注于分布式架构的设计。

ZooKeeper 在很多开源项目中都有应用，如Apache Flink、Spark等。

#### 1.2 ZooKeeper 的特点

ZooKeepe 提供的分布式协调服务具有如下的特点：

**（1）顺序性**

从同一个客户端发送的请求，最终将会严格按照其发送顺序进入 Zookeeper 中，先进先出。

**（2）原子性**

所有请求的响应结果在整个分布式集群中所有机器上的应用情况是一致的。

**（3）单一性**

客户端连接到任意 Zookeeper 服务器上所看到的服务端数据模型都是一致的。

**（4）可靠性**

 一旦一次更改请求被应用，更改的结果就会被持久化，直至被下一次更改覆盖。

**（5）实时性**

当某个请求成功处理后，客户端能够立即获取服务端的最新数据状态。

#### 1.3 ZooKeeper 的应用场景

ZooKeeper 提供的典型协调服务如下：

**（1）命名服务**

Zookeeper 可以实现一套分布式全局唯一 ID 的分配机制，在分布式环境中，为上层应用提供全局唯一的名字。

**（2）配置管理**

ZooKeeper 可以实现多个分布式程序的集中化的配置管理，在每次配置信息发生变化时向所有相关程序发出通知，从而使它们获取最新的配置信息应用。

**（3）集群管理**

ZooKeeper 可以监测集群中所有机器的上/下线情况，并在集群中选举 Leader。

**（4）分布式锁**

ZooKeeper 可以在分布式程序中实现分布式锁，支持多客户端互斥地对共享资源进行访问，保证数据的一致性。

### 二、主要功能分析与建模

#### 2.1 数据模型

集群通过在 ZooKeeper 里存取数据来进行消息的传递。ZooKeeper 的数据模型类似于允许文件也成为目录的文件系统，其数据节点构成一个具有层级关系的树状结构，节点的路径始终表示为规范的、绝对的、斜杠分隔的路径。每个数据节点叫作 Znode，其中保存自己的数据内容以及一系列属性信息，可以具有子节点。ZooKeeper 对自身的定位不是一个通用数据库，而是管理协调数据，因此每个 Znode 存储的数据大小必须小于1M。

> <img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/iamge2.png?raw=true" style="zoom: 60%;" />
>
> 图片来自(https://www.cnblogs.com/f-ck-need-u/p/9233249.html)

根据存活时间，Znode 可以划分为持久节点和临时节点，以及顺序持久节点和顺序临时节点，节点类型在创建时就被确定下来，并且不能改变。

**持久节点**的存活时间不依赖于客户端会话，只有客户端在显式执行删除节点操作时，节点才消失。

**临时节点**的存活时间依赖于客户端会话，当会话结束，临时节点就会被自动删除，并且不能拥有子节点。

**顺序持久节点和顺序临时节点**在持久节点和临时节点的基础上，在路径末尾附加了一个10位的单调递增计数器，同一个父节点下的计数器都是唯一的。

每个 Znode 的统计结构由以下部分组成:

| Field          | Description                                           |
| -------------- | ----------------------------------------------------- |
| czxid          | 此 Znode 创建时的 zxid（事务id）                      |
| mzxid          | 最近一次修改此 Znode 的 zxid                          |
| pzxid          | 最近一次修改此 Znode 的子节点的 zxid                  |
| ctime          | 创建此 Znode 的时间（以毫秒为单位）                   |
| mtime          | 最近一次修改此 Znode 的时间（以毫秒为单位）           |
| version        | 此 Znode 的修改次数                                   |
| cversion       | 此 Znode 子节点的修改次数                             |
| aversion       | 此 Znode 的 ACL 修改次数                              |
| ephemeralOwner | 若此 Znode 为临时节点，则为其所有者的会话 ID；否则为0 |
| dataLength     | 此 Znode 的数据字段长度                               |
| numChildren    | 此 Znode 的子节点总数                                 |

#### 2.2 集群角色、状态

ZooKeeper 分为服务器集群和客户端，客户端通过与服务器集群中的某一台机器建立 TCP 连接来使用服务。

>  <img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/iamge3.png?raw=true" style="zoom: 60%;" />
>
> 图片来自https://cwiki.apache.org/confluence/display/ZOOKEEPER/ProjectDescription

**集群角色：**

ZooKeeper 的服务器集群中分为 **Leader**、**Follower** 和 **Observer** 三种角色，其中 Follower 和 Observer 可以统称为 Learner。

**（1）Leader**

Leader 在整个服务器集群中只有一个，负责投票的发起和决议和更新系统状态。它既可以为客户端提供读服务，又可以为客户端提供写服务。

**（2）Follower**

Follower 在集群中拥有投票权，可以参与 Leader 的选举过程和写操作的过程。它只能为客户端提供读服务，接收到的所有写请求都会转发给 Leader 决议。

**（3）Observer**

Ovserver 在集群中没有投票权。和 Follower 一样，它只能为客户端提供读服务，接收到的所有写请求都会转发给 Leader 决议。

**集群状态：**

服务器具有四种状态：

**（1）LOOKING**

表示当前集群中没有 Leader，需要进入 Leader 选举。

**（2）FOLLOWING**

表示当前服务器是 Follower 角色

**（3）LEADING**

表示当前服务器是 Leader 角色

**（4）OBSERVING**

表示当前服务器是 Observer 角色

#### 2.3 需求建模

ZooKeeper 的使用分为单机模式与集群模式两种，由于本文主要关注集群模式的实现，后文均默认以集群模式工作。

考虑 ZooKeeper 中客户端向服务端发起一个事务请求，即写请求的过程，简要构建流程图如下：

 <img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/iamge4.png?raw=true" style="zoom: 60%;" />

其中，主要模块的需求建模如下：

```
需求建模
[用例名称]
    Leader选举
[用例描述]
    服务端初始化启动或当前Leader服务器崩溃时选举出集群中的Leader
[约束和限制]
    至少需要2台服务器
```

```
需求建模
[用例名称]
    数据同步
[用例描述]
    Leader选举完成后，根据Leader的最新状态同步所有Server
    当新的服务器节点加入时对其进行同步
[约束和限制]
    保证集群中存在过半的机器与Leader的数据状态保持一致
```

```
需求建模
[用例名称]
    服务端集群处理写请求
[用例描述]
    Leader接收到写请求直接决议
    Follower和Observer接收到写请求后转发给Leader决议
```

#### 2.3 主要功能分析

##### **（1）Leader 选举**

Leader 选举是保证分布式数据一致性的关键所在。当 Zookeeper 集群初始化启动或在运行期间 Leader 出现断开链接等情况时，需要进行 Leader 选举。

 **1. 更新状态**

 若为运行期间的 Leader 选举，所有 Server 的状态更新为 LOOKING。

**2. Server 投票**

每个 Server 首先投自己作为 Leader，投票中包含所选举的服务器 myid、Zxid 和 epoch，将这个投票发送给集群中其他机器。

其中 epoch 为一个自增序列，表示已经进行的 Leader 选举轮数，每进行一轮选举则加一。

**3. 更新投票**

Server 每收到一个其他投票，首先判断投票的有效性，如检查是否是本轮投票、是否来自 LOOKING 状态的服务器。若有效，将收到的投票和自己的原始投票进行比较，选择 Zxid 更大的 Server 作为 Leader，如果 Zxid 相同，选择 myid 更大的。每轮比较完成后将更新后的选票再次向集群中所有机器发送。

**4. 统计投票**

每次投票后，服务器都会统计自己收到的投票信息，判断是否有超过一半的机器与自己的选票相同，如果有，则Leader 产生，否则继续下一轮投票。

**5. 再次更新状态**

确定了 Leader 后，每个服务器会更新自己的状态，如果是 Follower，那么就变更为 FOLLOWING，如果是Leader，就变更为LEADING。

##### **（2）数据同步**

Leader 和 Follower之间需要进行状态数据同步，确保整个系统状态一致。

**1. 收集状态**

Leader 发送同步消息给 Follower，Follower 将自己数据中最大的Zxid发送给Leader。

**2. 确定同步点**

 Leader 根据 Follower 发送过来的 Zxid 与自身 Zxid 判断确定同步点，

**3. 广播通知**

Leader 将需要同步的数据发送给 Follower 和 Observer ，并在完成同步后通知它们。

**4. 继续服务**

 Follower 收到更新完成消息后，继续接受客户端的请求提供服务。

**（3）事务请求处理**

**1. 接收事务请求**

客户端向链接的 Server 发出事务请求。

**2. 转发写请求**

若连接的 Server 为 Follower 或者 Observer，将写请求转发给 Leader 决议；若为 Leader，直接进行决议。

**3. 发起投票**

当 Leader 接收到事务请求后，为其赋 64 位全局自增事务 id，即 Zxid，并将这条请求封装为一个 Proposal （提案）分发给所有的 Follower。Follower 接收到 Proposal 后将其写入磁盘，如果成功则向 Leader 反馈一个 Ack。

**4. 统计投票**

Leader 汇总 Follower 的反馈结果，如果接收到超过半数 Follower 的 Ack，则 Leader 开始执行提交该请求，同时向这些 Follower 发送 Commit 命令。当 Follower 收到 Commit 后，提交该请求，同步数据。

Observer 不参与上述3和4的投票过程，但需要同步 Leader 的数据以保证数据一致性。

**5. 返回请求**

与客户端相连的 Server 将请求结果返回给客户端。

>  <img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image5.png?raw=true" style="zoom: 60%;" />
>
> 图片来自https://www.cnblogs.com/wuzhenzhao/p/9983231.html

### 三、核心流程设计分析

这一部分将对 Leader 选举机制的进行进一步的分析。

#### 3.1 Zab 协议

Zookeeper 通过 **Zab**（Zookeeper Atomic Broadcast）协议来保证分布式事务的最终一致性，ZooKeeper 的集群角色模型以及 Leader 选举机制都是基于这个协议构建的。为了更深入理解 Leader 选举机制，对 Zab 协议进行一个简要的介绍。

Zab 协议是为 Zookeeper 专门设计的一种支持崩溃恢复的原子广播协议 ，是 Zookeeper 保证数据一致性的核心算法。它包含两种基本模式，分别是**崩溃恢复**和**消息广播**。

**（1）模式切换**　

正常工作过程中，ZooKeeper 在这两种模式下不停切换。

当整个集群启动，或者当 Leader 出现网络中断、崩溃等情况时，亦或是集群中不存在超过半数的服务器与Leader保存正常通信时，Zab 协议就会进入崩溃恢复模式。在崩溃恢复模式下，整个集群发起选举产生新的 Leader。当选举完成，且集群中有超过一半的 Follower 和 Leader 完成数据同步后，Zab 协议就会退出崩溃恢复模式。

退出崩溃恢复模式后，Zab 协议进入消息广播模式，集群开始接收事务请求正常工作。如果 Leader 正常工作时，启动了一台新的服务器加入集群，则这个服务器直接进入数据恢复模式，和 Leader 进行数据同步。同步完成后这个服务器加入集群一起正常对外提供非事务请求的处理。如果再出现上述情况，则切换回崩溃恢复模式。

**（2）消息广播**　　

消息广播模式下，ZooKeeper 为客户端正常提供事务、非事务请求的处理，核心是涉及数据一致性的事务请求处理，处理流程可见第一部分中的主要功能分析。

**（3）崩溃恢复**　　

崩溃恢复模式下，ZooKeeper 需要发起新一轮的 Leader 选举以正常工作，其中要确保在任何情况下都保持服务器间的数据一致性。

考虑崩溃恢复的两种可能影响数据一致性的异常情况：

1. 事务请求处理过程中，Leader 在收到了超过一半 Follower 的 Ack，在本地执行提交了该请求，但只向部分 Follower 发送了 Commit 命令时崩溃，则剩余 Follower 还未收到并执行 Commit，但客户端可能已经收到了请求处理成功的反馈。
2. 事务请求处理过程中，Leader 在接收到消息请求生成了 Proposal，还未向 Follower 发送时崩溃，则若选举出新的 Leader，这条请求被跳过，但原先的 Leader 转为 Follower 后还保留着这条请求的 proposal。

针对以上情况，Zab 协议规定 Leader 选举机制必须满足：

1. 已经被处理的消息不能丢失，即确保已经被 Leader 提交的 Proposal 必须最终会被所有的 Follower 服务器提交。
2. 被丢弃的消息不能再次出现，即确保丢弃已经被 Leader 提出的但是没有被提交的 Proposal。

#### 3.2 Leader 选举源码分析

之前的内容主要对选举机制的理论进行了分析，这一节将从具体代码着手分析 Leader 选举机制的实现细节。

在集群模式下，Server 为 QuorumPeer 类的对象。QuorumPeer 类继承自 ZooKeeperThread，是一个线程。

```java
public class QuorumPeer extends ZooKeeperThread implements QuorumStats.Provider 
```

**（1）Server 启动**

集群中的一个 Server 启动时，会调用 QuorumPeer.start 方法。

首先，Server 会调用 loadDataBase() 从本地文件中恢复数据至内存数据库对象 ZKDatabase，获取最新的 zxid。之后，调用 startServerCnxnFactory() 开启一个线程来接收客户端的请求，但由于此时初始化还未完成，还不能正常接收请求进行工作。

```java
public synchronized void start() {
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
        }
        //载入本地数据
        loadDataBase();
        //启动Server
        startServerCnxnFactory();
        try {
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
            System.out.println(e);
        }
        //初始化Leader选举算法
        startLeaderElection();
        //监控JVM暂停的时间信息
        startJvmPauseMonitor();
        //重写run方法
        super.start();
    }
```

**Leader 选举算法初始化**

完成加载数据、启动 Server 后，进入 Leader 选举算法初始化入口 startLeaderElection()。

判断当前 Server 的状态，如果为LOOKING，则创建选票，且初始化选票为自己。选票由myid、最近一次记录提交的Zxid和当前epoch组成。

```java
public synchronized void startLeaderElection() {
        try {
            //若当前状态为LOOKING，则创建一个新的投票
            if (getPeerState() == ServerState.LOOKING) {
                currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
            }
        } catch (IOException e) {
            RuntimeException re = new RuntimeException(e.getMessage());
            re.setStackTrace(e.getStackTrace());
            throw re;
        }
        //创建选举算法
        this.electionAlg = createElectionAlgorithm(electionType);
}
```

**创建选举算法**

Server 的选票初始化完成后，由 createElectionAlgorithm 方法创建选举算法。

选举算法有3种，但 ZooKeeper 已经不再支持第一种和第二种，默认创建第三种 FastLeaderElection 算法。

在创建选举算法时，每个 Server 会启动一个 QuorumCnxManager 对象，它负责各台服务器之间底层Leader选举过程中的网络通信。启动后通过它检查当前是否已经存在选举，若存在，则直接停止当前的选举；若不存在，则初始化并启动 QuorumCnxManager 的监听器 Listener，用于监听 electionPort 端口。之后初始化并启动 FastLeaderElection 选举算法。

```java
protected Election createElectionAlgorithm(int electionAlgorithm) {
        Election le = null;
        
        //TODO: use a factory rather than a switch
        switch (electionAlgorithm) {
        case 1:
            throw new UnsupportedOperationException("Election Algorithm 1 is not supported.");
        case 2:
            throw new UnsupportedOperationException("Election Algorithm 2 is not supported.");
        case 3:
            //初始化选举过程中各服务器之间的网络通信
            QuorumCnxManager qcm = createCnxnManager();
            QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
            //如果当前已经存在选举，则停止选举
            if (oldQcm != null) {
                LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
             //停止选举
                oldQcm.halt();
            }
            //初始化监听器
            QuorumCnxManager.Listener listener = qcm.listener;
            if (listener != null) {
            //启动监听器
                listener.start();
            //创建FastLeaderElection算法
                FastLeaderElection fle = new FastLeaderElection(this, qcm);
            //启动新建的选举算法
                fle.start();
                le = fle;
            } else {
                LOG.error("Null listener when initializing cnx manager");
            }
            break;
        default:
            assert false;
        }
        return le;
    }

```

在 FastLeaderElection 选举算法的启动过程初始化了意向 Leader 和发送队列、接收队列，并创建了信息对象。

```java
public class FastLeaderElection implements Election {
    ... 
    public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager) {
        this.stop = false;
        this.manager = manager;
        starter(self, manager);
    }
    
    private void starter(QuorumPeer self, QuorumCnxManager manager) {
        this.self = self;
        //意向Leader的myid
        proposedLeader = -1;
        //意向Leader的Zxid
        proposedZxid = -1;
        //创建发送队列
        sendqueue = new LinkedBlockingQueue<ToSend>();
        //创建接收队列
        recvqueue = new LinkedBlockingQueue<Notification>();
        //创建信息对象
        this.messenger = new Messenger(manager);
    }
}
```

在构建 messenger 的过程中，创建两个投票的发送和接收线程。

```java
protected class Messenger {
        ...
        Messenger(QuorumCnxManager manager) {
            //创建投票发送线程
            this.ws = new WorkerSender(manager);
            this.wsThread = new Thread(this.ws, "WorkerSender[myid=" + self.getId() + "]");
            this.wsThread.setDaemon(true);
            //创建投票接收线程
            this.wr = new WorkerReceiver(manager);
            this.wrThread = new Thread(this.wr, "WorkerReceiver[myid=" + self.getId() + "]");
            this.wrThread.setDaemon(true);
        }
}
```

**（2）选举过程**

在完成了上述一系列的 Server 启动以及 Leader 选举算法初始化过程后，返回 QuorumPeer.start 方法中的 super.start()，开始重写 run 方法，开启 Server 运行的主线程。

由于相关代码的篇幅较大，之后的代码都进行删减，只保留一些核心逻辑。

在 run 方法中，Server 根据当前所处的不同状态选择相应的操作：LOOKING 状态下，Server 进入 Leader 选举过程，且在 LOOKING 状态一直阻塞，直至选举出 Leader；其余状态下，Server 根据选举后自身的角色进行正常的集群工作，OBSERVING 状态 Observer 监听Leader的数据同步，FOLLOWING 状态 Follower 在 Leader 的领导下处理客户端请求，LEADING 状态下 Leader 领导集群工作。

```java
public void run() {
        ...
        try {
            while (running) {
                //根据服务器当前状态选择相应的操作
                switch (getPeerState()) {
                // LOOKING状态下，进入选举
                case LOOKING:                       
                    LOG.info("LOOKING");
                        ...
                    try {
                        ...
                //根据相应的选举算法进行领导选举
                   setCurrentVote(makeLEStrategy().lookForLeader());
                    }
                    ...
                    break;
                case OBSERVING:
                    ...             
                case FOLLOWING:
                    ...
                case LEADING:
                    ...
                }
            }
        } 
        ...
    }
```

LOOKING 状态下，选举进行，Server 通过 FastLeaderElection 中 lookForLeader() 进行选举，不断更新自己的选票，与其他 Server 间相互发送、接收选票，直至有超过一半的 Server 选出了相同的 Leader，选举结束。根据选举结果确定自身的状态、角色，开始正常工作。

选举过程中的具体逻辑可以见下方代码中的注释。

```java
public Vote lookForLeader() throws InterruptedException {
        ...
        try {
            //初始化recvset用于存储收到的选票
            Map<Long, Vote> recvset = new HashMap<Long, Vote>();
            //初始化outofelection用于选举结果
            Map<Long, Vote> outofelection = new HashMap<Long, Vote>();
            ...
            synchronized (this) {
            //更新逻辑时钟，用来判断是否在同一轮选举周期
                logicalclock.incrementAndGet();
            //初始化选票数据，即更新myid，Zxid，epoch
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }
            LOG.info(
                "New election. My id = {}, proposed zxid=0x{}",
                self.getId(),
                Long.toHexString(proposedZxid));
            //异步发送选举消息，先投自己一票
            sendNotifications();
            SyncedLearnerTracker voteSet;
            //在LOOKING状态不断循环，根据投票信息进行leader选举，直至选出一个Leader
            while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
                //从recvqueue中获取其他节点Server的选票
                Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);
                // 如果没有收到选票
                if (n == null) {
                //判断自己是否成功发出了选票，如果发出了则持续向外继续发送选票
                    if (manager.haveDelivered()) {
                        sendNotifications();
                    } else {
                //如果未成功发出，尝试重新与集群其他服务器连接
                        manager.connectAll();
                    }
                    ...
                //收到选票后，判断是否属于本集群
                } else if (validVoter(n.sid) && validVoter(n.leader)) {
                //判断接收到的选票投票者的状态
                    switch (n.state) {
                //LOOKING状态说明也在参与选举        
                    case LOOKING:
                //忽略Zxid=-1的选票
                        ...
                        // 判断epoch 是否大于本机的逻辑时钟，如果大于，说明是新一轮选举，重新加入选举
                        if (n.electionEpoch > logicalclock.get()) {
                            // 更新本地逻辑时钟
                            logicalclock.set(n.electionEpoch);
                            // 清空接收队列
                            recvset.clear();
                            //将接收到的选票信息和自身进行比较
                            //如果接收到的选票优先级更高，更新自身选票与其一致
                            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                            } else {
                            //否则投给自己
                                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                            }
                            //向外发送自己更新后的选票
                            sendNotifications();
                        //若epoch小于本机的逻辑时钟，忽略这张选票
                        } else if (n.electionEpoch < logicalclock.get()) {
                            ...
                            break;
                        //若epoch等于本机的逻辑时钟，说明处于同一轮选举中
                        //比较自己目前的意向Leader和对方的选票优先级，更新为其中优先级更高的
                        } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        //向外发送自己更新后的选票
                            sendNotifications();
                        }
                        ...
                        
                        //将收到的选票存入recvset 
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                        //获取recvset内的所有选票，判断已经收到的选票结果能否结束这一轮选举，即是否有超过一半的Server和自己选择了相同的Leader
                        voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));
                        //如果能结束
                        if (voteSet.hasAllQuorums()) {
                        //继续等待最后finalizeWait，看是否有人修改自己的选票
                            while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
                        //如果有，且修改后选票优先级更高，会影响选举结果，则放入选票队列中继续选举
                                if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                    recvqueue.put(n);
                                    break;
                                }
                            }
                        //如果这段时间内未收到新选票，则投票结束
                            if (n == null) {
                                //根据选出的Leader确定自己的角色，更新相应状态
                                setPeerState(proposedLeader, voteSet);
                                Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                                leaveInstance(endVote);
                                //返回选出的Leader
                                return endVote;
                            }
                        }
                        break;
                    //OBSERVING状态下不参与选举
                    case OBSERVING:
                        LOG.debug("Notification from observer: {}", n.sid);
                        break;
                    //FOLLOWING/LEADING状态下参与选举
                    case FOLLOWING:
                    case LEADING:
                    //如果对方的Epoch和本地时钟逻辑相等，则储存对方选票至recvset
                        if (n.electionEpoch == logicalclock.get()) {
                            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                            voteSet = getVoteTracker(recvset, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                    //判断recvset内选票能否结束选举确定自身的角色，可以则确定相应角色，结束选举
                            if (voteSet.hasAllQuorums() && checkLeader(recvset, n.leader, n.electionEpoch)) {
                                setPeerState(n.leader, voteSet);
                                Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }

                    //否则将其选票储存至outofelection，即不再更新的选票集合
                        outofelection.put(n.sid, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state)); 
                    //判断outofelection集合中的选票能否确定选举结果
                        voteSet = getVoteTracker(outofelection, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                    //若可以则确定相应角色，返回Leader
                        if (voteSet.hasAllQuorums() && checkLeader(outofelection, n.leader, n.electionEpoch)) {
                            synchronized (this) {
                                logicalclock.set(n.electionEpoch);
                                setPeerState(n.leader, voteSet);
                            }
                            Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    //否则继续接受选票进行选举
                        break;
                    default:
                        ...
    }
```

选举过程中，选票优先级的判断方法为，优先选择 epoch 更高的选票；如果 epoch 相同，优先选择 Zxid 更高的选票；如果 Zxid 相同，优先选择 myid 更高的选票。

```java
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
        LOG.debug(
            "id: {}, proposed id: {}, zxid: 0x{}, proposed zxid: 0x{}",
            newId,
            curId,
            Long.toHexString(newZxid),
            Long.toHexString(curZxid));

        if (self.getQuorumVerifier().getWeight(newId) == 0) {
            return false;
        }
　　　　/*如果以下三种情况之一成立，则返回true:
　　　　* 1-选票中epoch更高
　　　　* 2-选票中epoch与当前epoch相同，但新zxid更高
　　　　* 3-选票中epoch与当前epoch相同，新zxid与当前zxid相同，但服务器id更高
　　　　*/
        return ((newEpoch > curEpoch)
                || ((newEpoch == curEpoch)
                    && ((newZxid > curZxid)
                        || ((newZxid == curZxid)
                            && (newId > curId)))));
    }
```

#### **3.3 主要类图、时序图**

**（1）主要类间关系图**

**3.2**中分析的选举机制源码涉及的主要类图如下

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image6.png?raw=true" style="zoom: 60%;" />

**（2）简要时序图**

整个选举机制的时序大致可以分为 Server 启动过程、选举过程以及接收、发出选票的网络通信过程。

**3.2**中重点关注的是选举过程的实现以及必要的启动、通信过程，其简要时序图如下：

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image8.png?raw=true" style="zoom: 60%;" />



### 四、高级设计意图分析

这一部分将分析 ZooKeeper 的 Leader 选举机制实现过程中蕴含的高级设计意图，包括体现的设计原则以及使用到的设计模式。

#### 4.1 设计原则

ZooKeeper 作为一个大型开源项目，其建立、重构的过程中对所有设计原则肯定都有所运用。下面只简要分析 Leader 选举机制实现过程中体现的几个主要设计原则。

**（1）合成复用原则**

> 合成复用原则：尽量使用组合/聚合达到复用而非继承。

由**3.3**中 Leader 选举机制涉及到的主要类间关系图可以看出，当一个类要使用另一个类时，都主要使用了组合关系，而较少使用继承关系来达到复用的目的，这是符合合成复用原则的良好设计。

**（2）开闭原则**

> 开闭原则：面向扩展开放，面向修改关闭。

由下图可知，ZooKeeper 引入了 LearnerZooKeeperServer 抽象类来更好的扩展 FollowerZooKeeperServer 和 ObserverZooKeeperServer 类，这是符合开闭原则的良好设计。

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image9.png?raw=true" style="zoom: 60%;" />



**（3）单一职责原则**

> 单一职责原则：每一个类应该只做一件事情。

由下图可知，QuorumCnxManager 中设置了不同的内部类来完成不同的功能，这是符合单一职责原则的良好设计。

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image10.png?raw=true" style="zoom: 60%;" />

#### 4.2 设计模式

和**4.1**中提到设计原则时所说的一样，ZooKeeper 项目中也必定使用到很多设计模式。这里选取 ZooKeeper 中最有代表性的观察者模式和 Java 中最常用的工厂模式进行分析。

##### **（1）观察者模式**

**1. 观察者模式简介**

观察者模式也称发布-订阅模式，是一种行为型模式。它定义了一种一对多的依赖关系，让多个观察者对象同时监听某一个主题对象。当主题对象的状态发生变化时，会通知所有的观察者对象，使他们能够自动更新自己。

观察者模式下分为四种角色：

**Subject**：抽象被观察者。它把所有对观察者对象的引用保存在一个集合里，每个主题都可以有任意数量的观察者，抽象主题提供一个接口，可以增加和删除观察者对象。
**ConcreteSubject**：具体被观察者。它将有关状态存入具体观察者对象，在具体主题的内部状态发生改变时，给所有登记过的观察者发送通知。
**Observer**：抽象观察者。它定义了一个更新接口，使得在得到主题更改通知时能够更新自己。
**ConcrereObserver**：具体观察者。它实现抽象观察者定义的更新接口。

具体结构图如下：

> <img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image12.png?raw=true" style="zoom: 60%;" />
>
> 图片来自《大话设计模式》

**2. ZooKeeper 应用场景**

ZooKeeper 中观察者模式最典型的应用场景是在事务处理的 Watcher 监听机制中 。

ZooKeeper 中客户端可以通过监听器 Watcher，也就是上面所说的观察者，对数据节点 Znode 的变化进行监听。当需要监听某个数据节点时，客户端会在获取该数据的同时注册一个 Watcher。一旦这个节点发生数据变更、节点删除、子节点状态变更等变化，集群服务端就会向客户端发送通知。客户端收到通知响应，就会查找相应的 Watcher 并回调。

需要注意的是，ZooKeeper 的监听机制比较特别的一点是它的监听是“一次性”的。当监听的数据节点发生变化时，客户端只会收到一次通知。如果后续这个节点再次发生变化，之前设置 Watcher 的客户端不会再次收到消息。但是可以通过循环监听达到永久监听效果。

Watcher实现的具体机制如下：

客户端对数据节点进行事务操作 getData / exists / getChildren 时可以注册绑定 Watcher。

以 exists 为例：

```Java
public Stat exists(final String path, Watcher watcher) throws KeeperException, InterruptedException {
        final String clientPath = path;
        PathUtils.validatePath(clientPath);
    
        WatchRegistration wcb = null;
        //注册Watcher，与相应路径的节点绑定
        if (watcher != null) {
            wcb = new ExistsWatchRegistration(watcher, clientPath);
        }
        ...
    }
```

客户端类中也提供了对某个节点相应增、删 Watcher 的方法。

```Java
public void removeWatches(
        String path,
        Watcher watcher,
        WatcherType watcherType,
        boolean local) throws InterruptedException, KeeperException {
        ...
}
public void removeAllWatches(
        String path,
        WatcherType watcherType,
        boolean local) throws InterruptedException, KeeperException {
        ...
}
public void addWatch(String basePath, Watcher watcher, AddWatchMode mode)
            throws KeeperException, InterruptedException {
       ...
}
...
```

当被监听的数据节点进行createNode /deleteNode /setData等操作发生变化时，会触发监听器 Watcher，向客户端发送通知。

以setData为例：

```Java
public Stat setData(String path, byte[] data, int version, long zxid, long time) throws KeeperException.NoNodeException {
        Stat s = new Stat();
        DataNode n = nodes.get(path);
        ...
        updateWriteStat(path, dataBytes);
        //触发Watcher
        dataWatches.triggerWatch(path, EventType.NodeDataChanged);
        return s;
 }
```

Watcher 接口定义了回调方法 process 。

```Java
public interface Watcher {
    ...
    void process(WatchedEvent event);
}
```

客户端收到通知事件后会调用 processEvent 方法处理。

```Java
private void processEvent(Object event) {
            try {
                if (event instanceof WatcherSetEventPair) {
                    // each watcher will process the event
                    WatcherSetEventPair pair = (WatcherSetEventPair) event;
                    //得到符合触发机制的所有Watcher列表
                    for (Watcher watcher : pair.watchers) {
                        //循环回调所有的Watcher
                        try {
                            watcher.process(pair.event);
                        } catch (Throwable t) {
                            LOG.error("Error while calling watcher ", t);
                        }
                    }
                } else if
                    ...
            }

    }
```

以上对监听机制的分析省略了其中的很多通信过程，可能看起来有些零散。下面通过一个 ZooKeeper 应用过程中具体监听例子的实现来有一个更直观的认识。

> ```java
> public class WatcherDemo {
>     public static void main(String[] args) throws IOException, InterruptedException, KeeperException {
>         final CountDownLatch countDownLatch=new CountDownLatch(1);
>         //创建新客户端
>         final ZooKeeper zooKeeper=
>                  new ZooKeeper("192.168.254.135:2181," +
>                          "192.168.254.136:2181,192.168.254.137:2181",
>                         4000, new Watcher() {
>                     @Override
>                     public void process(WatchedEvent event) {
>                         System.out.println("默认事件： "+event.getType());
>                         if(Event.KeeperState.SyncConnected==event.getState()){
>                             //如果收到了服务端的响应事件，连接成功
>                             countDownLatch.countDown();
>                         }
>                     }
>                 });
>         countDownLatch.await();
>         //创建节点"/zk-wuzz"
>         zooKeeper.create("/zk-wuzz","1".getBytes(),
>                 ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT);
>         //通过exists在节点"/zk-wuzz"上注册绑定Watcher
>         Stat stat=zooKeeper.exists("/zk-wuzz", new Watcher() {
>             @Override
>             public void process(WatchedEvent event) {
>                 System.out.println(event.getType()+"->"+event.getPath());
>                 try {
>                     //再一次去绑定
>                     zooKeeper.exists(event.getPath(),true);
>                 } catch (KeeperException e) {
>                     e.printStackTrace();
>                 } catch (InterruptedException e) {
>                     e.printStackTrace();
>                 }
>             }
>         });
>         //通过setdata触发Watcher
>         stat=zooKeeper.setData("/zk-wuzz","2".getBytes(),stat.getVersion());
> 
>         Thread.sleep(1000);
>         //删除节点
>         zooKeeper.delete("/zk-wuzz",stat.getVersion());
> 
>         System.in.read();
>     }
> }
> ```
>
> 以上代码例子来自https://www.cnblogs.com/wuzhenzhao/p/9994450.html

##### （2）工厂模式

**1. 工厂模式简介**

工厂模式是一种创建型模式。它提供了一种创建对象的最佳方式，使得在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。

工厂模式下分为四种角色：

**Creator**：抽象工厂。它声明了工厂方法，通常返回一个 Product 类型的实例对象。
**ConcreteCreator**：具体工厂。它重定义了抽象工厂中的工厂方法，返回一个具体的 Product 实例。
**Product**：抽象产品。它定义了工厂创建对象的父类或拥有的共同接口。
**ConcrereObserver**：具体产品。它具体实现了抽象产品中定义的对象。

具体结构图如下：

> <img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image13.png?raw=true" style="zoom: 60%;" />
>
> 图片来自《大话设计模式》

**2. ZooKeeper 应用场景**

ZooKeeper 中客户端与服务端的网络通信类 ServerCnxnFactory 的实现就是典型的工厂模式应用。ServerCnxn 代表一个客户端与服务端的连接。ServerCnxnFactory 则是一个抽象工厂类，用于管理客户端与服务端的链接。它有两种实现， 一个是 NIOServerCnxnFactory，使用Java原生NIO实现；一个是 NettyServerCnxnFactory，提供Netty通信方式。在初始化时，默认为 NIOServerCnxnFactory 实现。

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image11.png?raw=true" style="zoom: 60%;" />

### 五、结语

作为一个 Java 初学者，本文对 ZooKeeper 的源码分析还有很多不足，在分析过程中省去了很多细节，只重点关注了核心代码框架的实现，希望这学期结束之后能抽出时间再将 ZooKeeper 内的通信机制仔细阅读一下。

在此感谢一个学期以来老师和助教们的帮助！
