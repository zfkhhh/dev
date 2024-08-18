redis热点key问题分两个：如何找到热点key，如何处理热点key

一、如何发现热点key
1. redis slot的qps监控，对redis集群的每个slot做流量监控，粒度太粗
2. redis proxy的监控，在redis proxy中对key的访问做一个计数(redis6.0推出了集群proxy)
3. redis4.0支持每个节点上基于lfu进行key热点机制，客户端命令rediscli -hotkeys可以获得key访问次数，定时查询这个信息就可以获得热点key
4. 对客户端的改造，在客户端做key访问计数，这个就需要公司基建
5. 业务埋点，服务端对key访问计数

二、处理热点key
1. 缓存预热：实际上部分热点key是可以通过业务可预测的，提前缓存好这部分数据
2. 二级缓存（本地缓存）：每次获得热点key时在本地内存中缓存一份，当缓存过期再重新从redis获取，查询数据先查本地缓存，查不到再查redis，再查不到查数据库
2.1 本地缓存最大的问题就是缓存一致性，需要watch数据的变更（redis watcher）
3. 热点key拆分到redis集群不同节点，redis集群是分布式的，可以降低压力
4. 限流：对于超过单秒qps压力的部分请求，通过限流算法进行限流