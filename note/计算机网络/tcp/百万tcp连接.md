一、处理tcp连接的方式
1. 通常tcp连接的处理，每个tcp连接创建会启动一个goroutine去处理，会开启一个读、写goroutine处理socket的读写事件
2. 一个tcp连接三个goroutine，goroutine最小2kb，再加上读写channel、tcp连接对象，在百万连接下，20G都算少的；
3. 普通的tcp连接的读写方式无法支持百万连接，可以通过epoll，一个goroutine中epoll监听所有tcp连接的事件，存在事件时epoll.wait返回事件列表
4. 但epoll处理百万tcp连接，如果事件较多或者事件处理较慢性能会有问题，可以采用多epoll，这里有两种方式启动一个线程池，采用多event loop的方式处理事件；采用reuseport多个listener sock
5. 多线程event loop用线程池优化了一个tcp连接创建读写goroutine的内存消耗
6. reuseport端口复用可以让多个线程或者goroutine同时listen一个端口，每个epoller处理一批连接


二、reuseport
1. 通常一个端口只能一个套接字socket，如果尝试监听已被监听的端口，bind会报错98 EADDRINUSE
2. 但是一个端口只能被一个进程监听吗？
2.1 如nginx，先bind一个套接字再fork子进程，这样就可以多个进程监听同一端口；通过unix domain socket将fd发送到另一个进程
2.2 端口复用本身是必须的，如组播广播；在linux，可以调用setsocket将so_reuseaddr和so_reuseport设置为1，开启地址复用、端口复用；在go中因为net.Listen将socket(),accept(),listen()封装到了一起，可以通过net.ListenConfig设置
3. 多进程监听同一个端口，采用抢占式接收处理数据