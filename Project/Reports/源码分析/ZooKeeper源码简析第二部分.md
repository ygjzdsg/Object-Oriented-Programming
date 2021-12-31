## ZooKeeper 源码简析

严贝 2019K8009937001

本文为对 ZooKeeper 项目源码的学习简析报告，其中将重点关注 Leader 选举机制的实现。

分析使用的版本号为 ZooKeeper3.6.3

源码仓库：https://github.com/apache/zookeeper

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image1.png?raw=true" alt="ZooKeeper 的应用场景" style="zoom: 60%;" />

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

在创建选举算法时，每个 Server 会启动一个 QuorumCnxManager 对象，它负责各台服务器之间底层Leader选举过程中的网络通信。启动后通过它检查当前是否已经存在选举，若存在，则直接停止当前的选举；若不存在，则初始化并启动QuorumCnxManager 的监听器 Listener，用于监听 electionPort 端口。之后初始化并启动 FastLeaderElection 选举算法。

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

在 run 方法中，Server 根据当前所处的不同状态选择相应的操作：LOOKING 状态下，Server 进入 Leader 选举过程，且在 LOOKING 状态一直循环，直至选举出 Leader；其余状态下，Server 根据选举后自身的角色进行正常的集群工作，OBSERVING 状态 Observer 监听Leader的数据同步，FOLLOWING 状态 Follower 在 Leader 的领导下处理客户端请求，LEADING 状态下 Leader 领导集群工作。

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

3.2中分析的选举机制源码涉及的主要类图如下

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image6.png?raw=true" alt="image6" style="zoom: 60%;" />

整个选举机制的时序大致可以分为 Server 启动过程、选举过程以及接收、发出选票的网络通信过程。

3.2中重点关注的是选举过程的实现以及必要的启动、通信过程，其简要时序图如下：

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image8.png?raw=true" style="zoom: 60%;" />



