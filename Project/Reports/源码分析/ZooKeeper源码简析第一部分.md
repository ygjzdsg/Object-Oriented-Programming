## ZooKeeper 源码简析

严贝 2019K8009937001

本文为对 ZooKeeper 项目源码的学习简析报告，其中将重点关注 Leader 选举机制的实现。

分析使用的版本号为 ZooKeeper3.6.3

源码仓库：https://github.com/apache/zookeeper

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image1.png?raw=true" alt="ZooKeeper 的应用场景" style="zoom: 67%;" />

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

> <img src="https://images2018.cnblogs.com/blog/733013/201806/733013-20180627120322365-702119801.png" alt="img" style="zoom: 67%;" />
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

> <img src="https://cwiki.apache.org/confluence/download/attachments/24193436/service.png?version=1&modificationDate=1295027310000&api=v2" alt="img" style="zoom: 80%;" />
>
> 图片来自https://cwiki.apache.org/confluence/display/ZOOKEEPER/ProjectDescription

**集群角色：**

ZooKeeper 的服务器集群中分为 Leader、Follower 和 Observer 三种角色，其中 Follower 和 Observer 可以统称为 Learner。

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

考虑 ZooKeeper 中客户端向服务端发起一个简单写请求的过程，简要构建流程图如下：

<img src="C:\Users\86187\Desktop\未命名文件(2).png" alt="未命名文件(2)" style="zoom: 80%;" />

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

Leader 选举是保证分布式数据一致性的关键所在。当 Zookeeper 集群中任意一台服务器初始化启动或在运行期间无法和 Leader 保持链接时，需要进行 Leader 选举。

 **1. 更新状态**

 若为运行期间的 Leader 选举，所有 Server 的状态更新为 LOOKING。

**2. Server 投票**

每个 Server 首先投自己作为 Leader，投票中包含所选举的服务器 myid、Zxid 以及 epoch，将这个投票发送给集群中其他机器。

其中 epoch 用来判断多个投票是否在同一轮选举周期中，它是一个自增序列，每进入新一轮的投票都会加1。

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

Leader 将需要同步的数据发送给 Follower，并在完成同步后通知 Follower。

**4. 继续服务**

 Follower 收到更新完成消息后，继续接受客户端的请求提供服务。

**（3）写请求处理**

**1. 接收写请求**

客户端向链接的 Server 发出写请求。

**2. 转发写请求**

若连接的 Server 为 Follower 或者 Observer，将写请求转发给 Leader 决议；若为 Leader，直接进行决议。

**3. 发起投票**

Leader 接收到请求后发送 proposal 给 Follower 发起投票，Follower 开始写入，如果成功则将成功投票结果反馈给 Leader。

**4. 统计投票**

Leader 汇总投票结果，如果有一半以上的 Follower 写入成功，则 Leader 可以写入，开始写入。同时，Leader 将写入操作发送通知给所有 Follower，完成写入同步。

**5. 返回请求**

与客户端相连的 Server 将请求结果返回给客户端。

> <img src="https://img2018.cnblogs.com/blog/1383365/201811/1383365-20181119135821298-1900058559.png" alt="img" style="zoom:67%;" />
>
> 图片来自https://www.cnblogs.com/wuzhenzhao/p/9983231.html

下一部分中将对 Leader 选举功能进行更深入的分析。

### 

