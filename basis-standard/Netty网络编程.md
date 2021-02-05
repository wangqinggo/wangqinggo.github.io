# Netty网络编程

## Java NIO 类库

### 缓冲区Buffer

- limit
- position
- capacity
- mark

### 通道Channel

### 多路复用器Selector

## 关键概念

### Channel

### ChannelPipeline

### ChannelHandler

### inbound/outbound 事件

### EventLoop

### Future/Promise

### 时间轮算法

## 关键流程

- 服务端创建流程
- 客户端创建流程

## 故障定位

- 接收不到消息
- 内存泄漏
- 性能问题

## 常见问题

- 限流
  - 针对写数据速度(高低水位,高水位暂停写)
  - 针对读数据速度(流量整形,慢慢读)
- 统计
- 超时
- 优雅退出
- 内存管理
- 心跳/空闲

## Netty使用案例

- [深入浅出Netty](https://www.infoq.cn/article/netty-in-depth)
- [Netty使用案例 -发送队列积压导致内存泄漏（一）](https://blog.csdn.net/u013642886/article/details/86632752)
- [Netty使用案例 -发送队列积压导致内存泄漏（二）](https://blog.csdn.net/u013642886/article/details/86633786)

