[..](./../basis-standard/index.md)
# guava

## eventbus

- 学习观察者模式实现

- 反射获取对象的方法用了cache

- 分发Dispatcher支持多种策略(如同步、异步、一个线程一个队列)

## cache

- 过期数据维护，没有采用线程

- 内部会维护两个队列 accessQueue,writeQueue 用于记录缓存顺序

- 利用 GC 来回收内存

## concurrent

### 令牌桶算法RateLimiter

- 延时计算令牌，而不是采用线程

- 通过控制令牌生成速度来达到限流，与之相对的是漏桶算法通过令牌总量控制

### Listenable系列的Executors、Future

### 增强Future

#### Futures

#### SettableFuture

#### CollectionFuture

#### TrustedListenableFutureTask