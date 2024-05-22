mvcc: 基于版本控制的乐观锁

mysql中可重复读隔离级别，为解决不可重复读问题，快照读就是基于mvcc实现，给每次改动添加版本，版本与事务时间戳关联，读操作只读在事务开始前的快照；存在缺点，不能解决更新丢失问题


ETCD中的mvcc实现：分为treeIndex和boltDB,boltDB是一个kv数据库，key是全局版本号revision,value是key value等字段组成的结构体，treeIndex是内存中b+树，对key版本索引管理

treeIndex的树节点keyIndex结构体里有generation数组，保存key的所有版本信息，vesion结构体记录了事务开始时的主版本

写过程：
1. 查找key的版本keyIndex,事务里的删改操作，会基于主版本号递增版本号
2. 事务结束，将最后版本号与key的映射计入treeIndex
3. 构造mvcc的keyvalue结构体，启动backend协程异步写入boltdb并同步更新buffer

其中，删除操作是追加空的generation结构体，并在生成的BoltDB key版本号加一个标识t表示已删除；当查询时，查询treeIndex版本号小于删除版本号，直接返空

读过程：
1. 读操作底层统一调用的range方法
2. 查treeIndex中key的keyIndex结构体，遍历generation查找版本号,如果没传版本号查最新版本
3. 通过版本号，优先查buffer数据，查不到再查boltdb中的value