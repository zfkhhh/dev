1. http的keepalive是应用层（用户态）实现，是http长链接
2. tcp的keepalive是运输层（内核态）实现，是tcp的保活机制

一、http的keepalive
1. 在http1.0默认关闭，1.1后默认打开；http1.0需要在请求头加上connection:keep-alive;http1.1关闭需要在请求头加上connection:close
2. http基于tcp，http短连接就是每次客户端通讯都需要经历建立tcp连接，发送http请求，服务端响应，释放连接
3. 短连接一次连接只能请求一次资源，没有发挥tcp长连接的作用，http的keepalive实现了http长连接，使用一个tcp连接发送接收多次http请求响应
4. 为了避免http长连接浪费资源，提供了keepalive-timeout参数设置长连接的超时时间
5. keepalive客户端服务端都可以设置；当一方没有开启keepalive，会由服务端来关闭tcp连接
5.1 为什么由服务端来关闭？
5.1.1 只请求-响应模型里，不重用连接的时机在服务端，服务端通过请求头得到客户端的keepalive配置，也能找到服务端的配置
5.1.2 因为服务端关闭的话只需要调用一次close(),其他由内核处理，只需要一次syscall；如果由客户端关闭，服务端需要完成最后的响应后将socket加入readable队列，epoll等待事件，再调一次read()才能收到客户端的关闭信息，需要两次syscall
6. keepalive因为由服务端关闭tcp连接，容易造成tcp大量处于time_wait状态

二、tcp的keepalive
1. tcp的keepalive是保活机制，用来探测另一方的tcp连接是否存活；通过SO_KEEPALIVE选项开启
2. 当tcp连接一直没有数据交互，保活定时器超时，会发送保活报文keep-alive
2.1 当另一方接收到保活报文会响应数据keep-alive ack，保活定时器会重置
2.2 当另一方无响应，连续多次请求达到保活探测次数后，会判断该tcp连接死亡，释放连接
3. tcp的keepalive默认是2小时，一般由应用层设计心跳服务