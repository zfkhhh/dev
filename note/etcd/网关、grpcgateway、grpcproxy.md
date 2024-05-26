etcd网关: tcp代理，将客户端请求代理到etcd集群，每个etcd部署的机器都会部署网关，网关监听本地地址
grpc gateway: etcdv3使用grpc作为通讯协议，有的语言不支持grpc，etcd通过grpc gateway对外提供http接口，内部将http请求转化grpc请求
grpc proxy:
1. 基于L7层的无状态的gRPC的etcd反向代理服务； 
2. 如果有多个客户端watcher,可以合并为链接到etcd的单个watcher（s-watcher）；
3. lease有续约机制，客户端通过定时刷新心跳来续约，proxy能将多个心跳合并成一个s-stream
4. 并且会缓存客户端的请求，防止频繁请求服务端