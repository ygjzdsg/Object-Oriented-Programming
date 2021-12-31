

## ZooKeeper 源码简析

严贝 2019K8009937001

本文为对 ZooKeeper 项目源码的学习简析报告，其中将重点关注 Leader 选举机制的实现。

分析使用的版本号为 ZooKeeper3.6.3

源码仓库：https://github.com/apache/zookeeper

<img src="https://github.com/ygjzdsg/Object-Oriented-Programming/blob/main/Project/Reports/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image/image1.png?raw=true" style="zoom: 60%;" />

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

**（1）观察者模式**

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

**（2）工厂模式**

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
