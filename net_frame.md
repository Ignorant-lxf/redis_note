# 网络模型

**Redis 为什么快？**
- C 语言实现
- 纯内存 I/O 
- I/O 多路复用
- 单线程模型 (处理客户端请求是单线程) (指的是 网络模型)

**Redis 为啥选择单线程？**
- 对于 DB 来讲，大部分不是 CPU 密集型，而是 I/O 密集型。Redis 是完全的纯内存操作，执行速度是非常快的。Redis 真正的性能瓶颈在于网络 I/O
- 避免上下文切换带来的消耗
- 避免同步机制的开销
- 代码简单容易维护

客户端向 Redis 发起请求命令的工作原理:

- Redis 服务器启动，开启主线程事件循环，注册 acceptTcpHandler 连接应答处理器到用户配置的监听端口对应的文件描述符，等待新连接到来
- 客户端和服务端建立网络连接
- acceptTcpHandler 被调用，将 readQueryFromClient 绑定到新连接对应的文件描述符上，并初始化一个 client 绑定这个客户端连接
- 客户端发送请求命令，除非读就绪事件，主线程调用 readQueryFromClient 读取 socket 里面客户端发过来的命令到 client-querybuf
- 调用 processInputBuffer，根据 Redis 协议解析命令，最后调用 processCommand 执行命令
- 根据解析的命令类型，去调用对应的执行器。 最后调用 addReply 函数族的一系列函数将响应数据写入到对应 client 的写出缓冲区，client-buf 或者 client-reply。
  - client-buf 是首选，固定大小 16 KB，如果响应数据过大，写入 client-reply，是链表结构，理论上数据可以无限大
- 在事件循环机制，遍历 client-pending-write 队列，把缓冲数据写回客户端

**refer to https://segmentfault.com/a/1190000039223696**