title: ZooKeeper Watcher 和 AsyncCallback 的区别与实现
date: 2016-08-22 22:15:08
tags: [Zookeeper]
categories: Zookeeper
---

## 前言
初学 Zookeeper 会发现客户端有两种回调方式： Watcher 和 AsyncCallback，而 Zookeeper 的使用是离不开这两种方式的，搞清楚它们之间的区别与实现显得尤为重要。本文将围绕下面几个方面展开

- Watcher 和 AsyncCallback 的区别
- Watcher 的回调实现
- AsyncCallback 的回调实现
- IO 与事件处理

## Watcher 和 AsyncCallback 的区别
我们先通过一个例子来感受一下：

    zooKeeper.getData(root, new Watcher() {
                public void process(WatchedEvent event) {

                }
            }, new AsyncCallback.DataCallback() {
                public void processResult(int rc, String path, Object ctx, byte[] data, Stat stat) {

                }
            }, null);

可以看到，`getData`方法可以同时设置两个回调：Watcher 和 AsyncCallback，同样是回调，它们的区别是什么呢？要解决这个问题，我们就得从这两个接口的功能入手。

- `Watcher`：`Watcher`是用于监听节点，session 状态的，比如`getData`对数据节点`a`设置了`watcher`，那么当`a`的数据内容发生改变时，客户端会收到`NodeDataChanged`通知，然后进行`watcher`的回调。
- `AsyncCallback`:`AsyncCallback`是在以异步方式使用 ZooKeeper API 时，用于处理返回结果的。例如：`getData`同步调用的版本是：`byte[] getData(String path, boolean watch,Stat stat)`，异步调用的版本是：`void getData(String path,Watcher watcher,AsyncCallback.DataCallback cb,Object ctx)`，可以看到，前者是直接返回获取的结果，后者是通过`AsyncCallback`回调处理结果的。

## Watcher
Watcher 主要是通过`ClientWatchManager`进行管理的。
### Watcher 的类型
`ClientWatchManager`中有四种`Watcher`

- `defaultWatcher`：创建`Zookeeper`连接时传入的`Watcher`，用于监听 session 状态
- `dataWatches`：存放`getData`传入的`Watcher`
- `existWatches`：存放`exists`传入的`Watcher`，如果节点已存在，则`Watcher`会被添加到`dataWatches`
- `childWatches`：存放`getChildren`传入的`Watcher`

从代码上可以发现，监听器是存在`HashMap`中的，`key`是节点名称`path`，`value`是`Set<Watcher>`

    private final Map<String, Set<Watcher>> dataWatches =
            new HashMap<String, Set<Watcher>>();
    private final Map<String, Set<Watcher>> existWatches =
            new HashMap<String, Set<Watcher>>();
    private final Map<String, Set<Watcher>> childWatches =
            new HashMap<String, Set<Watcher>>();

    private volatile Watcher defaultWatcher;

### 通知的状态类型与事件类型
在`Watcher`接口中，已经定义了所有的状态类型和事件类型

- KeeperState.Disconnected(0)

    此时客户端处于断开连接状态，和ZK集群都没有建立连接。

    - EventType.None(-1)

        触发条件：一般是在与服务器断开连接的时候，客户端会收到这个事件。
- KeeperState. SyncConnected(3)

    此时客户端处于连接状态
    - EventType.None(-1)

        触发条件：客户端与服务器成功建立会话之后，会收到这个通知。
    - EventType. NodeCreated (1)

        触发条件：所关注的节点被创建。
    - EventType. NodeDeleted (2)

        触发条件：所关注的节点被删除。
    - EventType. NodeDataChanged (3)

        触发条件：所关注的节点的内容有更新。注意，这个地方说的内容是指数据的版本`号dataVersion`。因此，即使使用相同的数据内容来更新，还是会收到这个事件通知的。无论如何，调用了更新接口，就一定会更新`dataVersion`的。
    - EventType. NodeChildrenChanged (4)

        触发条件：所关注的节点的子节点有变化。这里说的变化是指子节点的个数和组成，具体到子节点内容的变化是不会通知的。

- KeeperState. AuthFailed(4)

    认证失败

    - EventType.None(-1)

- KeeperState. Expired(-112)

    session 超时

    - EventType.None(-1)

### materialize 方法
`ClientWatchManager`只有一个方法，那就是`materialize`，它根据事件类型`type`和`path`返回监听该节点的特定类型的`Watcher`。

    public Set<Watcher> materialize(Watcher.Event.KeeperState state,
        Watcher.Event.EventType type, String path);

核心逻辑如下：

1. `type == None`:返回所有`Watcher`，也就是说所有的`Watcher`都会被触发。如果`disableAutoWatchReset == true`且当前`state != SyncConnected`，那么还会清空`Watcher`，意味着移除所有在节点上的`Watcher`。
2. `type == NodeDataChanged | NodeCreated`:返回监听`path`节点的`dataWatches & existWatches`
3. `type == NodeChildrenChanged`:返回监听`path`节点的`childWatches`
4. `type == NodeDeleted`:返回监听`path`节点的`dataWatches | childWatches`

每次返回都会从`HashMap`中移除节点对应的`Watcher`，例如：`addTo(dataWatches.remove(clientPath), result);`，这就是为什么`Watcher`是一次性的原因（`defaultWatcher`除外）。值得注意的是，由于使用的是`HashSet`存储`Watcher`，重复添加同一个实例的`Watcher`也只会被触发一次。

## AsyncCallback
Zookeeper 的`exists`,`getData`,`getChildren`方法都有异步的版本，它们与同步方法的区别仅仅在于是否等待响应，底层发送都是通过`sendThread`异步发送的。下面我们用一幅图来说明：

![][1]
上面的图展示了同步/异步调用`getData`的流程，其他方法也是类似的。

## IO 与事件处理
Zookeeper 客户端会启动两个常驻线程

- `SendThread`：负责 IO 操作，包括发送，接受响应，发送 ping 等。
- `EventThread`：负责处理事件，执行回调函数。

![][2]
### readResponse
`readResponse`是`SendThread`处理响应的核心函数，核心逻辑如下：

1. 接受服务器的响应，并反序列化出`ReplyHeader`： 有一个单独的线程`SendThread`，负责接收服务器端的响应。假设接受到的服务器传递过来的字节流是`incomingBuffer`，那么就将这个`incomingBuffer`反序列化为`ReplyHeader`。
2. 判断响应类型：判断`ReplyHeader`是`Watcher`响应还是`AsyncCallback`响应：`ReplyHeader.getXid()`存储了响应类型。

   1. 如果是`Watcher`类型响应：从`ReplyHeader`中创建`WatchedEvent`，`WatchedEvent`里面存储了节点的路径，然后去`WatcherManager`中找到和这个节点相关联的所有`Watcher`，将他们写入到`EventThread`的`waitingEvents`中。
   2. 如果是`AsyncCallback`类型响应：从`ReplyHeader`中读取`response`，这个`response`描述了是`Exists，setData，getData，getChildren，create.....`中的哪一个异步回调。从`pendingQueue`中拿到`Packet`，`Packet`中的`cb`存储了`AsyncCallback`，也就是异步 API 的结果回调。最后将`Packet`写入到`EventThread`的`waitingEvents`中。

### processEvent
`processEvent`是`EventThread`处理事件核心函数，核心逻辑如下：

1. 如果`event instanceof WatcherSetEventPair`，取出`pair`中的`Watchers`，逐个调用`watcher.process(pair.event)`
2. 否则`event`为`AsyncCallback`，根据`p.response`判断为哪种响应类型，执行响应的回调`processResult`。

可见，`Watcher`和`AsyncCallback`都是由`EventThread`处理的，通过`processEvent`进行区分处理。

## 总结
Zookeeper 客户端中`Watcher`和`AsyncCallback`都是异步回调的方式，但它们回调的时机是不一样的，前者是由服务器发送事件触发客户端回调，后者是在执行了请求后得到响应后客户端主动触发的。它们的共同点在于都需要在获取了服务器响应之后，由`SendThread`写入`EventThread`的`waitingEvents`中，然后由`EventThread`逐个从事件队列中获取并处理。

*参考资料*
[ZooKeeper个人笔记客户端watcher和AsycCallback回调][3]
[【ZooKeeper Notes 13】ZooKeeper Watcher的事件通知类型][4]


  [1]: http://7xjtfr.com1.z0.glb.clouddn.com/async.png
  [2]: http://7xjtfr.com1.z0.glb.clouddn.com/event.png
  [3]: http://www.cnblogs.com/francisYoung/p/5225703.html
  [4]: http://nileader.blog.51cto.com/1381108/954670