- [Redis](#redis)
  - [集成方案](#集成方案)
    - [主从复制模式](#主从复制模式)
      - [工作机制](#工作机制)
      - [优点](#优点)
      - [缺点](#缺点)
    - [Sentinel（哨兵）模式](#sentinel哨兵模式)
      - [工作机制](#工作机制-1)
      - [优点](#优点-1)
      - [缺点](#缺点-1)
    - [Cluster模式](#cluster模式)
      - [工作机制](#工作机制-2)
      - [优点](#优点-2)
      - [缺点](#缺点-2)
      - [Cluster模式集群节点最小配置6个节点(3主3从，因为需要半数以上)，其中主节点提供读写操作，从节点作为备用节点，不提供请求，只作为故障转移使用](#cluster模式集群节点最小配置6个节点3主3从因为需要半数以上其中主节点提供读写操作从节点作为备用节点不提供请求只作为故障转移使用)
  - [redis持久化](#redis持久化)
    - [触发时机](#触发时机)
    - [RDB](#rdb)
    - [AOF](#aof)
    - [混合持久化](#混合持久化)
    - [持久化方案的建议](#持久化方案的建议)
  - [待定](#待定)
  - [淘汰策略](#淘汰策略)
  
## Redis

### 集成方案

#### 主从复制模式

##### 工作机制

- slave启动后，向master发送SYNC命令，master接收到SYNC命令后通过bgsave保存快照（即上文所介绍的RDB持久化），并使用缓冲区记录保存快照这段时间内执行的写命令

- master将保存的快照文件发送给slave，并继续记录执行的写命令

- slave接收到快照文件后，加载快照文件，载入数据

- master快照发送完后开始向slave发送缓冲区的写命令，slave接收命令并执行，完成复制初始化

- 此后master每次执行一个写命令都会同步发送给slave，保持master与slave之间数据的一致性

##### 优点

- master能自动将数据同步到slave，可以进行读写分离，分担master的读压力

- master、slave之间的同步是以非阻塞的方式进行的，同步期间，客户端仍然可以提交查询或更新请求

##### 缺点

- 不具备自动容错与恢复功能，master或slave的宕机都可能导致客户端请求失败，需要等待机器重启或手动切换客户端IP才能恢复

- master宕机，如果宕机前数据没有同步完，则切换IP后会存在数据不一致的问题

- 难以支持在线扩容，Redis的容量受限于单机配置

#### Sentinel（哨兵）模式

##### 工作机制

在配置文件中通过sentinel monitor <master-name> <ip> <redis-port> <quorum>来定位master的IP、端口，一个哨兵可以监控多个master数据库，只需要提供多个该配置项即可。哨兵启动后，会与要监控的master建立两条连接：

- 一条连接用来订阅master的sentinel:hello频道与获取其他监控该master的哨兵节点信息

- 另一条连接定期向master发送INFO等命令获取master本身的信息

- 与master建立连接后，哨兵会执行三个操作：

- 定期（一般10s一次，当master被标记为主观下线时，改为1s一次）向master和slave发送INFO命令

- 定期向master和slave的sentinel:hello频道发送自己的信息

- 定期（1s一次）向master、slave和其他哨兵发送PING命令

- 发送INFO命令可以获取当前数据库的相关信息从而实现新节点的自动发现。所以说哨兵只需要配置master数据库信息就可以自动发现其slave信息。获取到slave信息后，哨兵也会与slave建立两条连接执行监控。通过INFO命令，哨兵可以获取主从数据库的最新信息，并进行相应的操作，比如角色变更等。

- 接下来哨兵向主从数据库的sentinel:hello频道发送信息与同样监控这些数据库的哨兵共享自己的信息，发送内容为哨兵的ip端口、运行id、配置版本、master名字、master的ip端口还有master的配置版本。这些信息有以下用处：

- 其他哨兵可以通过该信息判断发送者是否是新发现的哨兵，如果是的话会创建一个到该哨兵的连接用于发送PING命令。

- 其他哨兵通过该信息可以判断master的版本，如果该版本高于直接记录的版本，将会更新

- 当实现了自动发现slave和其他哨兵节点后，哨兵就可以通过定期发送PING命令定时监控这些数据库和节点有没有停止服务。

- 如果被PING的数据库或者节点超时（通过 sentinel down-after-milliseconds master-name milliseconds 配置）未回复，哨兵认为其主观下线（sdown，s就是Subjectively —— 主观地）。

- 如果下线的是master，哨兵会向其它哨兵发送命令询问它们是否也认为该master主观下线，如果达到一定数目（即配置文件中的quorum）投票，哨兵会认为该master已经客观下线（odown，o就是Objectively —— 客观地），并选举领头的哨兵节点对主从系统发起故障恢复。

- 若没有足够的sentinel进程同意master下线，master的客观下线状态会被移除，若master重新向sentinel进程发送的PING命令返回有效回复，master的主观下线状态就会被移除。

- 哨兵认为master客观下线后，故障恢复的操作需要由选举的领头哨兵来执行，选举采用Raft算法：

  - 发现master下线的哨兵节点（我们称他为A）向每个哨兵发送命令，要求对方选自己为领头哨兵
  - 如果目标哨兵节点没有选过其他人，则会同意选举A为领头哨兵
  - 如果有超过一半的哨兵同意选举A为领头，则A当选
  - 如果有多个哨兵节点同时参选领头，此时有可能存在一轮投票无竞选者胜出，此时每个参选的节点等待一个随机时间后再次发起参选请求，进行下一轮投票竞选，直至选举出领头哨兵
- 选出领头哨兵后，领头者开始对系统进行故障恢复，从出现故障的master的从数据库中挑选一个来当选新的master,选择规则如下：
  - 所有在线的slave中选择优先级最高的，优先级可以通过slave-priority配置
  - 如果有多个最高优先级的slave，则选取复制偏移量最大（即复制越完整）的当选
  - 如果以上条件都一样，选取id最小的slave

挑选出需要继任的slave后，领头哨兵向该数据库发送命令使其升格为master，然后再向其他slave发送命令接受新的master，最后更新数据。将已经停止的旧的master更新为新的master的从数据库，使其恢复服务后以slave的身份继续运行。

##### 优点

- 哨兵模式基于主从复制模式，所以主从复制模式有的优点，哨兵模式也有

- 哨兵模式下，master挂掉可以自动进行切换，系统可用性更高

##### 缺点

- 同样也继承了主从模式难以在线扩容的缺点，Redis的容量受限于单机配置

- 需要额外的资源来启动sentinel进程，实现相对复杂一点，同时slave节点作为备份节点不提供服务

#### Cluster模式

##### 工作机制

- 在Redis的每个节点上，都有一个插槽（slot），取值范围为0-16383

- 当我们存取key的时候，Redis会根据CRC16的算法得出一个结果，然后把结果对16384求余数，这样每个key都会对应一个编号在0-16383之间的哈希槽，通过这个值，去找到对应的插槽所对应的节点，然后直接自动跳转到这个对应的节点上进行存取操作

- 为了保证高可用，Cluster模式也引入主从复制模式，一个主节点对应一个或者多个从节点，当主节点宕机的时候，就会启用从节点

- 当其它主节点ping一个主节点A时，如果半数以上的主节点与A通信超时，那么认为主节点A宕机了。如果主节点A和它的从节点都宕机了，那么该集群就无法再提供服务了

##### 优点

- 无中心架构，数据按照slot分布在多个节点

- 集群中的每个节点都是平等的关系，每个节点都保存各自的数据和整个集群的状态。每个节点都和其他所有节点连接，而且这些连接保持活跃，这样就保证了我们只需要连接集群中的任意一个节点，就可以获取到其他节点的数据

- 可线性扩展到1000多个节点，节点可动态添加或删除

- 能够实现自动故障转移，节点之间通过gossip协议交换状态信息，用投票机制完成slave到master的角色转换

##### 缺点

- 客户端实现复杂，驱动要求实现Smart Client，缓存slots mapping信息并及时更新，提高了开发难度。目前仅JedisCluster相对成熟，异常处理还不完善，比如常见的“max redirect exception”

- 节点会因为某些原因发生阻塞（阻塞时间大于 cluster-node-timeout）被判断下线，这种failover是没有必要的

- 数据通过异步复制，不保证数据的强一致性

- slave充当“冷备”，不能缓解读压力

- 批量操作限制，目前只支持具有相同slot值的key执行批量操作，对mset、mget、sunion等操作支持不友好

- key事务操作支持有线，只支持多key在同一节点的事务操作，多key分布不同节点时无法使用事务功能

- 不支持多数据库空间，单机redis可以支持16个db，集群模式下只能使用一个，即db 0 Redis

- Cluster模式不建议使用pipeline和multi-keys操作，减少max redirect产生的场景

##### Cluster模式集群节点最小配置6个节点(3主3从，因为需要半数以上)，其中主节点提供读写操作，从节点作为备用节点，不提供请求，只作为故障转移使用

### [redis持久化](https://www.cnblogs.com/richiewlq/p/12191261.html)

#### 触发时机

- 手动触发

- 自动触发

#### RDB

- 主要配置

  - save 900 1 # 在900s内存在至少一次写操作

  - save "" # 禁用RBD持久化

  - stop-writes-on-bgsave-error yes # 当备份进程出错时主进程是否停止写入操作

  - rdbcompression no  # 是否压缩rdb文件 推荐no 相对于硬盘成本cpu资源更贵

- 与AOF相比，RDB文件相对较小，恢复数据比较快

- 服务器宕机，RBD方式会丢失掉上一次RDB持久化后的数据

- 使用bgsave fork子进程时会耗费内存

#### AOF

- 主要配置

  - appendonly no # 默认关闭AOF，若要开启将no改为yes

  - appendfilename "appendonly.aof" # append文件的名字

  - appendfsync everysec # 每隔一秒将缓存区内容写入文件 默认开启的写入方式

  - auto-aof-rewrite-percentage 100 # 当AOF文件大小的增长率大于该配置项时自动开启重写（这里指超过原大小的100%）。

  - auto-aof-rewrite-min-size 64mb # 当AOF文件大小大于该配置项时自动开启重写

- AOF只是追加文件，对服务器性能影响较小，速度比RDB快，消耗内存也少，同时可读性高

- 生成的文件相对较大，即使通过AOF重写，仍然会比较大

- 恢复数据的速度比RDB慢

#### 混合持久化

- 4.0版开始支持

- 兼容性差，一旦开启了混合持久化，在4.0之前的版本都不识别该持久化文件

- aof-use-rdb-preamble yes # 开启混合持久化

#### 持久化方案的建议

- 如果Redis只是用来做缓存服务器，比如数据库查询数据后缓存，那可以不用考虑持久化，因为缓存服务失效还能再从数据库获取恢复

- 如果你要想提供很高的数据保障性，那么建议你同时使用两种持久化方式。如果你可以接受灾难带来的几分钟的数据丢失，那么可以仅使用RDB

- 通常的设计思路是利用主从复制机制来弥补持久化时性能上的影响。即Master上RDB、AOF都不做，保证Master的读写性能，而Slave上则同时开启RDB和AOF（或4.0以上版本的混合持久化方式）来进行持久化，保证数据的安全性

### 待定

- 集群扩容缩容

- 监控分析

- Redis为什么快

- 应用使用redis注意

- 参数解读

- 运维经验

  - [优酷蓝鲸近千节点的 Redis 集群运维经验总结](https://www.infoq.cn/article/2016/08/youku-Redis-nosql?utm_source=related_read&utm_medium=article)

  - [Redis 内存使用优化与存储](https://www.infoq.cn/article/tq-redis-memory-usage-optimization-storage?utm_source=related_read&utm_medium=article)

  - [阿里云 Redis 开发规范](https://www.infoq.cn/article/K7dB5AFKI9mr5Ugbs_px?utm_source=related_read&utm_medium=article)

### 淘汰策略

- allkeys-lru：根据 LRU 算法删除键，不管数据有没有设置超时属性，直到腾出足够空间为止

- allkeys-random：随机删除所有键，直到腾出足够空间为止

- volatile-lru(默认): 超过最大内存后，在过期键中使用 lru 算法进行 key 的剔除，保证不过期数据不被删除，但是可能会出现 OOM 问题

- volatile-random: 随机删除过期键，直到腾出足够空间为止

- volatile-ttl：根据键值对象的 ttl 属性，删除最近将要过期数据。如果没有，回退到 noeviction 策略

- noeviction：不会剔除任何数据，拒绝所有写入操作并返回客户端错误信息"(error) OOM command not allowed when used memory"，此时 Redis 只响应读操作