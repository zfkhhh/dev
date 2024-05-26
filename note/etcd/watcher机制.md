etcd watcher 可以监听key或者key前缀的键值变化

1. client获得事件的机制，etcd是轮询还是推送？

etcd v2采用轮询模式，基于http1.x协议，每个watcher都是一个tcp链接，client通过http1.1的长链接定时轮询watcher获得事件，如果有大量的watcher，server会消耗大量socket、内存资源

etcd v3采用的推送，基于grpc协议，grpc底层是http2，grpc双向流，http2多路复用解决了v2的问题
http2有两个概念：帧和流，消息分帧，每个流有对应的id，帧会标识属于哪个流，这样一个tcp链接就可以存在多个流，可以交错发送消息


2. etcd事件如何存储

v2基于滑动窗口存储历史版本，在内存使用一个环形数组存储消息，当消息数量大于数组会丢失，当服务端客户端间网络异常容易造成消息丢失

v3基于mvcc机制，将key的历史版本持久化到boltdb数据库

3. etcd事件推送机制，当网络问题等因素照成服务端堆积了大量消息，etcd会怎么做；当监听的key版本server端不存在了，该怎么处理

etcd消息推送问题拆解成三部分：syncwatcherGroup 已同步完成的watcher; unsyncwatcherGroup 未同步到最新数据变更版本的watcher; victimwatcherGroup 存在推送失败的watcher

3.1 当key发生put事件，put事务结束后会封装成event调用notify，会从syncwatcherGroup中拿到监听这个key并且版本号小于等于事件版本号的watcher，往watcher的channel中推送event，会有一个sendLoop
从channel读出事件，发生消息

3.2 因为网络问题导致读channel缓慢，当channel的buffer满了，会将wacher放到victimWatcherGroup, 会有victimLoop处理
victimLoop会遍历group，尝试推送消息，失败就再次加入group等待下次重试；成功推送，如果watcher的监听的最小版本小于等于server的当前版本，说明还有消息没发完，放到unsyncGroup
如果监听的消息大于server当前版本，说明最新消息已经推送完，放到syncGroup

3.3 历史消息推送机制；有一个syncLoop协程会遍历unsyncGroup，从其中拿一部分watcher,获得其中的最小版本，查询最小版本到server的最大版本号区间的所有版本数据，匹配key对应的watcher，推送消息
如果watcher监听的版本号小于db压缩的最小版本，会给client发生错误，客户端需要根据这个错误，重新获取最新版本号watcher


4. etcd事件匹配，在key发生数据变更，如何找到对应的watcher

当watcher创建时，会有watcherMap会将监听的key范围插入到区间树，区间树是在红黑树的基础上扩展了区间为元素，区间树保持了相同key范围的watcher集合
当key发生变化时，可以查到key所在区间的watcher集合