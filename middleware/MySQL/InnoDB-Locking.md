[..](./../../middleware/index.md)

- [InnoDB 锁](#innodb-锁)
  - [Shared and Exclusive Locks](#shared-and-exclusive-locks)
      - [假设事务T1有row1的S锁，然后另一个事务T2也对row1进行获得锁请求？](#假设事务t1有row1的s锁然后另一个事务t2也对row1进行获得锁请求)
  - [Intention Locks](#intention-locks)
    - [两种类型的I锁](#两种类型的i锁)
    - [意向锁协议](#意向锁协议)
    - [表级锁兼容汇总](#表级锁兼容汇总)
  - [Record Locks（记录锁，作用于索引）](#record-locks记录锁作用于索引)
  - [Gap Locks（间隙锁）](#gap-locks间隙锁)
  - [Next-Key Locks（邻键锁）](#next-key-locks邻键锁)
  - [综上三锁](#综上三锁)
  - [Insert Intention Locks](#insert-intention-locks)
  - [AUTO-INC Locks](#auto-inc-locks)
  - [Predicate Locks for Spatial Indexes](#predicate-locks-for-spatial-indexes)
  
## InnoDB 锁
> 锁是计算机协调多个进程或线程并发访问某一资源的机制。
>
> 在数据库中，处理传统的计算资源（如CPU、RAM、I/O等）的竞争外，数据也是一种共享资源。如何保证数据并发访问的一致性、有效性是所有数据库必须解决的一个问题；如果加锁过度，会极大的降低并发处理能力。

参考：[15.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

> InnoDB实现标准的`行级锁`，分为 共享锁和排它锁。

### Shared and Exclusive Locks

- [共享锁](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_shared_lock)：一种锁，允许其他[事务](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_transaction)（可以提交或回滚的原子单位）读取锁定的对象，并获取该对象上的其它共享锁，但不能对其写入；允许持有该锁的事务读取一行。
- [互斥锁](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_exclusive_lock)：一种锁，可以防止其它事务锁定同一行。根据事物隔离级别，这种锁可能会阻止其它事务写入同一行，或者也可能阻止其它事务读取同一行。默认的InnoDB隔离级别`可重复读`通过允许事务读取具有`互斥锁`的行来实现更高的并发性，这种技术也称`一致读取`。允许持有该锁的事务 update or delete a row。

##### 假设事务T1有row1的S锁，然后另一个事务T2也对row1进行获得锁请求？

- T2请求S锁，会被`立即授予`，结果就是T1、T2共同持有row1的S锁；
- T2请求X锁，不能被立即授予；

> 如果事务T1对row1持有X锁，则不能立即批准另一个事务T2对row1任何类型锁的请求；T2必须等待事务T1释放row1的锁定。

### Intention Locks

> InnoDB支持多种粒度锁定，允许行锁和表锁并存。
>
> 例如：LOCK TABLES ... WRITE之类语句对指定表采用X锁。
>
> 为了支持多个粒度级别的锁，InnoDB使用I（意向）锁。
>
> 意向锁是表级锁，表示事务稍后对表中的行需要那种类型的锁（S | X）。

#### 两种类型的I锁

- IS（意向共享锁）：表示事务打算对`表`中的各行设置共享锁；
- XS（意向互斥锁）：表示事务打算对`表`中的各行设置排它锁。

例如： `SELECT ... FOR SHARE`可设置IS， `SELECT ... FOR UPDATE`设置IX

#### 意向锁协议

1. 当事务要获取表中某行`共享锁（S锁）之前`，必须首先获取该表`意向共享锁(IS)或者更强的锁`。
2. 当事务要获取表中某行的`排它锁（X锁）之前`，必须首先获取该表`意向排它锁（IX）`。

#### 表级锁兼容汇总

> IX表锁不与S行锁兼容；行锁只与S、IS兼容。

|      |  X   |  IX  |  S   |  IS  |
| :--: | :--: | :--: | :--: | :--: |
|  X   | 冲突 | 冲突 | 冲突 | 冲突 |
|  IX  | 冲突 | 兼容 | 冲突 | 兼容 |
|  S   | 冲突 | 冲突 | 兼容 | 兼容 |
|  IS  | 冲突 | 兼容 | 兼容 | 兼容 |

> 如果一个锁与现有锁兼容，则将其授予请求的事务，反之冲突则不授予；
>
> 事务等待直到冲突的现有锁释放。
>
> 如过锁定请求与现有锁定发生冲突，并且由于可能导致死锁而无法授予，则会发生错误。

- 意向锁除了全表请求（例如：LOCK TABLES ... WRITE）外，不阻止任何其他内容。意图锁定的主要目的是表明某人正在锁定表中的行或要锁定表中的行。

### Record Locks（记录锁，作用于索引）

> 记录锁，是对索引的锁定；例如t WHERE c1 = 10 FOR UPDATE中选择c1， 防止任何其他事务插入，更新或删除t.c1值为10的行。
>
> 记录锁始终锁定索引记录，即使没有定义索引的表也是如此；对于没有这种情况，InnoDB 会创建一个隐藏的聚集索引并将该索引用于记录锁定。

### Gap Locks（间隙锁）
> 间隙锁定是对索引记录之间的间隙的锁定。
>
> 例如： `SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;`防止其它事务将值15插到t.c1中，无论该列中是否已有这样的值，因为`该范围中所有值之间的间隙是锁定的`。

- 间隙锁可能跨越单个索引值、多个索引值、甚至为空；

- 间隙锁是性能和并发性之间权衡的一部分，并且在某些事务隔离级别而非其他级别中使用。

- 对于使用唯一索引锁定的唯一行的语句，不需要间隙锁；搜索条件包含多列唯一索引的情况，会发生间隙锁。例如：下面id唯一，就未使用间隙锁：

  ```sql
  SELECT * FROM child WHERE id = 100;
  ```

  - `如果id不是索引列或者索引不唯一，则该语句会锁定前面的间隙`。

- gap lock不需要用于那些使用**唯一索引**锁住行来查找唯一行的语句(这不包括一个情况，那就是当查询条件包含了**复合唯一索引**的一部分时，gap lock确实会存在)。
- **gap lock的意义只在于阻止区间被插入，因此是可以共存的**。`一个事务获取的gap lock不会阻止另一个事务获取同一个gap的gap lock。`共享和排它的gap lock是没有区别的。它们相互不冲突，且功能相同。
- gap lock可以被显式禁止。当在`READ COMMITTED`等级下时，gap lock被禁止用于索引查找，而只用于外检约束检查和重复键检查。

### Next-Key Locks（邻键锁）

> 是一个索引锁和索引记录之前的间隙锁的组合；

- InnoDB执行行级锁的锁定方式是：当它单条或者范围查找索引时，会在遇到的索引记录上设置X或S锁；因此， `行级锁实际就是索引记录锁。`索引记录上的Next-Key锁同样也会影响该索引前面的`间隙`。也就是说，**一个next-key锁 = 索引上的记录锁 + 锁住前面间隙的gap lock**。如果一个事务在记录R上的某个索引持有了S或X锁，则另一个事务不能立即在R的对应索引前面`插入`新的记录。
- 假设有一个索引有值10、11、13、20。则其上的Next-Key Locks 包含有如下的区间，圆括号表示开，方括号表示闭：

```matlab
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```

对于最后一个区间，Next-Key锁施加一个`无穷大`的伪记录上，从而会锁住大于最大值20的间隙。这个伪记录具有大于所有实际存在于索引的值。`无穷大`记录不是真实存在的，因此，实际上它上面的Next-Key锁只锁住最大值20之后的间隙。

- 默认情况下，InnoDB事务隔离级别默认是 [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) ；在此级别下，InnoDB使用Next-Key进行单条或范围查找索引来**防止幻读**。

### 综上三锁

> record lock、gap lock、next-key lock，都是`作用在索引`上的。假设有记录1,3,5,7，则5上的记录锁会锁住5，5上的gap lock会锁住(3,5)，5上的next-key lock会锁住(3,5]。

### Insert Intention Locks

> 插入意向锁是一种在行插入之前通过`INSERT`操作设置的 `gap lock`；目的是为了插入的性能和正确。
>
> `这种锁表示要以这样的一种方式插入：如果多个事务插入到相同的索引间隙中，如果它们不在间隙中的相同位置插入，则无需等待其他事务（不阻塞）`。
>
> 例如：有索引4和7之间的间隙，有两个事务分别想插入5、6，在获得插入行的X锁之前，每个锁都使用Insert Intention Locks来锁定4和7之间的间隙，但不互相阻塞，因为行是无冲突的。

### AUTO-INC Locks

> 是一种特殊的表级锁，由插入到具体的 `AUTO_INCREMENT列`的表中的事务获取。
>
> 最简单的情况，如果一个事务正在向表中插入值，则任何其它事务都必须等待自己在该表中进行插入，以便第一个事务插入的行接收连续的主键值。
>
>  [`innodb_autoinc_lock_mode`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode) 配置可以控制用于自增锁算法；它让你可以选择如何在可预测的自增序列与插入操作的最大并发性之间进行权衡。

### Predicate Locks for Spatial Indexes

> 空间索引的谓语锁；
>
> InnoDB支持对包含空间列的列进行SPATIAL索引。

- 为了处理与 SPATIAL 索引有关的操作的锁定，Next-Key锁定不能很好地支持 REPEATABLE READ 或 SERIALIZABLE 事务隔离级别。 多维数据中没有绝对排序概念，因此不清楚哪个是邻键。
- 为了支持具有 SPATIAL 索引的表的隔离级别，InnoDB使用Predicate锁。空间索引包含最小外接矩形值，因此 InnoDB 通过在用于查询的 MBR 值上设置谓词锁来强制对索引进行一致性读。其他事务不能插入或修改与查询条件匹配的行。

