1. http和grpc都是应用层通信协议，http简单易用，适用场景简单的web程序；grpc适用复杂的微服务，需要高性能、跨语言支持的场景
2. 最大的区别在与数据序列化不同，grpc使用protocol buffer传输格式，而http一般使用json；而protocol的序列化、反序列化更紧凑高效，所以grpc的性能更好
3. grpc不仅支持http2,还支持多种传输协议如tcp,udp,websocket