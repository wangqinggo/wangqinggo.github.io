[..](./../basis-standard/index.md)
# tcp基础

## TCP 三次握手的性能提升

- 调整SYN报文重传次数

- 调整SYN半连接队列长度

- 调整SYN+ACK报文重传次数

- 调整accept队列长度

- 绕过三次握手

## TCP 四次挥手的性能提升

- 调整FIN报文重传次数

- 调整FIN_WAIT2状态的时间

- 调整孤儿连接的上限个数

- 调整time_wait状态的上限个数

- 复用time_wait状态的连接

## TCP 数据传输的性能提升

- 扩大窗口大小

- 调整发送缓冲区范围

- 调整接受缓冲区范围

- 接受缓冲区动态调整

- 调整内存范围

## socket选项

- SO_LINGER:设置延迟关闭的时间，等待套接字发送缓冲区中的数据发送完成

- SO_RCVBUF:接收缓冲区大小

- SO_SNDBUF:发送缓冲区大小

- SO_KEEPALIVE:发送“保持活动”包

- SO_REUSEADDR:允许套接口和一个已在使用中的地址捆绑

- TCP_NODELAY:禁止发送合并的Nagle算法

## [通信过程](https://blog.csdn.net/zxy987872674/article/details/52653101)

## [**tcp状态机**](https://blog.csdn.net/xy010902100449/article/details/48274635)