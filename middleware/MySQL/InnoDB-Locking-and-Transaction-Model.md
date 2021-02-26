[..](./../../middleware/index.md)
- [锁](#锁)
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
- [InnoDB Transaction Model](#innodb-transaction-model)
  - [Transaction Isolation Levels](#transaction-isolation-levels)
    - [InnoDB支持《SQL:1992标准》中定义的四个事务隔离级别:](#innodb支持sql1992标准中定义的四个事务隔离级别)
    - [InnoDB对每个事务隔离级别使用不同的锁策略](#innodb对每个事务隔离级别使用不同的锁策略)
    - [`REPEATABLE READ`](#repeatable-read)
    - [`READ COMMITTED`](#read-committed)
    - [`READ UNCOMMITTED`](#read-uncommitted)
    - [`SERIALIZABLE`](#serializable)
  - [autocommit, Commit, and Rollback](#autocommit-commit-and-rollback)
      - [Grouping DML Operations with Transactions](#grouping-dml-operations-with-transactions)
      - [Transactions in Client-Side Languages](#transactions-in-client-side-languages)
    - [Consistent Nonlocking Reads(非锁定一致性读)](#consistent-nonlocking-reads非锁定一致性读)
      - [一致性读不支持某些DDL（drop、alter）语句](#一致性读不支持某些ddldropalter语句)
      - [对于没有指定 `FOR UPDATE` 或者 `LOCK IN SHARE MODE` 的各种查询, 其行为有所不同, 如 `INSERT INTO ... SELECT`, `UPDATE ... (SELECT)`, 以及 `CREATE TABLE ... SELECT`:](#对于没有指定-for-update-或者-lock-in-share-mode-的各种查询-其行为有所不同-如-insert-into--select-update--select-以及-create-table--select)

  
## 锁

> 锁是计算机协调多个进程或线程并发访问某一资源的机制。
>
> 在数据库中，处理传统的计算资源（如CPU、RAM、I/O等）的竞争外，数据也是一种共享资源。如何保证数据并发访问的一致性、有效性是所有数据库必须解决的一个问题；如果加锁过度，会极大的降低并发处理能力。

### InnoDB 锁

参考：[15.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

> InnoDB实现标准的`行级锁`，分为 共享锁和排它锁。

#### Shared and Exclusive Locks

- [共享锁](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_shared_lock)：一种锁，允许其他[事务](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_transaction)（可以提交或回滚的原子单位）读取锁定的对象，并获取该对象上的其它共享锁，但不能对其写入；允许持有该锁的事务读取一行。
- [互斥锁](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_exclusive_lock)：一种锁，可以防止其它事务锁定同一行。根据事物隔离级别，这种锁可能会阻止其它事务写入同一行，或者也可能阻止其它事务读取同一行。默认的InnoDB隔离级别`可重复读`通过允许事务读取具有`互斥锁`的行来实现更高的并发性，这种技术也称`一致读取`。允许持有该锁的事务 update or delete a row。

##### 假设事务T1有row1的S锁，然后另一个事务T2也对row1进行获得锁请求？

- T2请求S锁，会被`立即授予`，结果就是T1、T2共同持有row1的S锁；
- T2请求X锁，不能被立即授予；

> 如果事务T1对row1持有X锁，则不能立即批准另一个事务T2对row1任何类型锁的请求；T2必须等待事务T1释放row1的锁定。

#### Intention Locks

> InnoDB支持多种粒度锁定，允许行锁和表锁并存。
>
> 例如：LOCK TABLES ... WRITE之类语句对指定表采用X锁。
>
> 为了支持多个粒度级别的锁，InnoDB使用I（意向）锁。
>
> 意向锁是表级锁，表示事务稍后对表中的行需要那种类型的锁（S | X）。

##### 两种类型的I锁

- IS（意向共享锁）：表示事务打算对`表`中的各行设置共享锁；
- XS（意向互斥锁）：表示事务打算对`表`中的各行设置排它锁。

例如： `SELECT ... FOR SHARE`可设置IS， `SELECT ... FOR UPDATE`设置IX

##### 意向锁协议

1. 当事务要获取表中某行`共享锁（S锁）之前`，必须首先获取该表`意向共享锁(IS)或者更强的锁`。
2. 当事务要获取表中某行的`排它锁（X锁）之前`，必须首先获取该表`意向排它锁（IX）`。

##### 表级锁兼容汇总

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

#### Record Locks（记录锁，作用于索引）

> 记录锁，是对索引的锁定；例如t WHERE c1 = 10 FOR UPDATE中选择c1， 防止任何其他事务插入，更新或删除t.c1值为10的行。
>
> 记录锁始终锁定索引记录，即使没有定义索引的表也是如此；对于没有这种情况，InnoDB 会创建一个隐藏的聚集索引并将该索引用于记录锁定。

#### Gap Locks（间隙锁）
> 间隙锁定是对索引记录之间的间隙的锁定。
>
> 例如： `SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;`防止其它事务将值15插到t.c1中，无论该列中是否已有这样的值，因为`该范围中所有值之间的间隙是锁定的`。

- 间隙锁可能跨越单个索引值、多个索引值、甚至为空；

- 间隙锁是性能和并发性之间权衡的一部分，并且在某些事务隔离级别而非其他级别中使用。

- 对于使用唯一索引锁定的唯一行的语句，不需要间隙锁；搜索条件包含多列唯一索引的情况，会发生间隙锁。例如：下面id唯一，就未使用间隙锁：

  ```
  SELECT * FROM child WHERE id = 100;
  ```

  - `如果id不是索引列或者索引不唯一，则该语句会锁定前面的间隙`。

- gap lock不需要用于那些使用**唯一索引**锁住行来查找唯一行的语句(这不包括一个情况，那就是当查询条件包含了**复合唯一索引**的一部分时，gap lock确实会存在)。
- **gap lock的意义只在于阻止区间被插入，因此是可以共存的**。`一个事务获取的gap lock不会阻止另一个事务获取同一个gap的gap lock。`共享和排它的gap lock是没有区别的。它们相互不冲突，且功能相同。
- gap lock可以被显式禁止。当在`READ COMMITTED`等级下时，gap lock被禁止用于索引查找，而只用于外检约束检查和重复键检查。

#### Next-Key Locks（邻键锁）

> 是一个索引锁和索引记录之前的间隙锁的组合；

- InnoDB执行行级锁的锁定方式是：当它单条或者范围查找索引时，会在遇到的索引记录上设置X或S锁；因此， `行级锁实际就是索引记录锁。`索引记录上的Next-Key锁同样也会影响该索引前面的`间隙`。也就是说，**一个next-key锁 = 索引上的记录锁 + 锁住前面间隙的gap lock**。如果一个事务在记录R上的某个索引持有了S或X锁，则另一个事务不能立即在R的对应索引前面`插入`新的记录。
- 假设有一个索引有值10、11、13、20。则其上的Next-Key Locks 包含有如下的区间，圆括号表示开，方括号表示闭：

```
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```

对于最后一个区间，Next-Key锁施加一个`无穷大`的伪记录上，从而会锁住大于最大值20的间隙。这个伪记录具有大于所有实际存在于索引的值。`无穷大`记录不是真实存在的，因此，实际上它上面的Next-Key锁只锁住最大值20之后的间隙。

- 默认情况下，InnoDB事务隔离级别默认是 [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) ；在此级别下，InnoDB使用Next-Key进行单条或范围查找索引来**防止幻读**。

#### 综上三锁

> record lock、gap lock、next-key lock，都是`作用在索引`上的。假设有记录1,3,5,7，则5上的记录锁会锁住5，5上的gap lock会锁住(3,5)，5上的next-key lock会锁住(3,5]。

#### Insert Intention Locks

> 插入意向锁是一种在行插入之前通过`INSERT`操作设置的 `gap lock`；目的是为了插入的性能和正确。
>
> `这种锁表示要以这样的一种方式插入：如果多个事务插入到相同的索引间隙中，如果它们不在间隙中的相同位置插入，则无需等待其他事务（不阻塞）`。
>
> 例如：有索引4和7之间的间隙，有两个事务分别想插入5、6，在获得插入行的X锁之前，每个锁都使用Insert Intention Locks来锁定4和7之间的间隙，但不互相阻塞，因为行是无冲突的。

#### AUTO-INC Locks

> 是一种特殊的表级锁，由插入到具体的 `AUTO_INCREMENT列`的表中的事务获取。
>
> 最简单的情况，如果一个事务正在向表中插入值，则任何其它事务都必须等待自己在该表中进行插入，以便第一个事务插入的行接收连续的主键值。
>
>  [`innodb_autoinc_lock_mode`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode) 配置可以控制用于自增锁算法；它让你可以选择如何在可预测的自增序列与插入操作的最大并发性之间进行权衡。

#### Predicate Locks for Spatial Indexes

> 空间索引的谓语锁；
>
> InnoDB支持对包含空间列的列进行SPATIAL索引。

- 为了处理与 SPATIAL 索引有关的操作的锁定，Next-Key锁定不能很好地支持 REPEATABLE READ 或 SERIALIZABLE 事务隔离级别。 多维数据中没有绝对排序概念，因此不清楚哪个是邻键。
- 为了支持具有 SPATIAL 索引的表的隔离级别，InnoDB使用Predicate锁。空间索引包含最小外接矩形值，因此 InnoDB 通过在用于查询的 MBR 值上设置谓词锁来强制对索引进行一致性读。其他事务不能插入或修改与查询条件匹配的行。

##  InnoDB Transaction Model

参考：[InnoDB Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html) [14.7 InnoDB的锁和事务模型](https://github.com/cncounter/translation/blob/master/tiemao_2020/44_innodb-storage-engine/14.7_innodb-locking-transaction-model.md)

> InnoDB的事务模型(transaction model), 目标是将多版本数据库(multi-versioning database)的最佳属性与传统的两阶段锁定(two-phase locking)相结合。 默认情况下, InnoDB使用行级锁, 并以非锁定一致性读(nonlocking consistent read)的方式来执行查询, 类似Oracle数据库。 InnoDB中的锁信息, 以节省空间的方式存储, 因此不需要锁升级(lock escalation)。 支持多个用户锁定InnoDB表中的每一行, 或者任意多行, 都不会让InnoDB的内存耗尽。

###  Transaction Isolation Levels

> 事务隔离(Transaction isolation)是数据库的基础特征。 隔离(Isolation)就是`ACID`中的`I`； 隔离级别是一个可配置项, 用于在多个事务进行同时并发修改和并发查询时, 调节性能、可靠性(reliability)、一致性(consistency)和可重复性(reproducibility)之间的平衡。

#### InnoDB支持《SQL:1992标准》中定义的四个事务隔离级别:

- `READ UNCOMMITTED`(读未提交),
- `READ COMMITTED`(读已提交),
- `REPEATABLE READ`(可重复读), InnoDB 默认的隔离级别。
- `SERIALIZABLE`(串行化)。

用户可以改变当前会话的隔离级别(控制自己会话的可见性), 也可以更改后续所有连接的隔离级别, 使用 [`SET TRANSACTION`](https://dev.mysql.com/doc/refman/5.7/en/set-transaction.html) 语句即可。 要设置服务器的默认隔离级别, 可在命令行或配置文件中使用 `--transaction-isolation` 选项。 设置隔离级别的详细信息, 请参见 [Section 13.3.6, “SET TRANSACTION Statement”](https://dev.mysql.com/doc/refman/5.7/en/set-transaction.html)。

#### InnoDB对每个事务隔离级别使用不同的锁策略

- 可使用默认的 `REPEATABLE READ` 级别来实现一致性, 比如 `ACID` 规范很重要的关键数据处理。
- 在批处理报表之类的场景下, 可以使用 `READ COMMITTED` 甚至 `READ UNCOMMITTED` 来放宽一致性约束, 这时候精确的一致性和可重复的结果, 相对来说不如降低锁的开销重要。
- `SERIALIZABLE` 比 `REPEATABLE READ` 的限制更严格, 主要用于特殊情况, 例如 `XA` 事务, 或者对并发和死锁问题进行故障诊断和排查等场景。

#### `REPEATABLE READ`

> InnoDB的默认隔离级别。 可重复读隔离级别, 同一事务中的一致性读, 使用第一次读取时创建的快照。 这意味着, 在同一事务中执行多个普通的 `SELECT`语句(nonlocking), 则这些 `SELECT` 语句之间彼此是能保证一致性的。 详情请查看 [14.7.2.3 非锁定一致性读](https://github.com/cncounter/translation/blob/master/tiemao_2020/44_innodb-storage-engine/14.7_innodb-locking-transaction-model.md#14.7.2.3)。
>
> 对于`UPDATE`语句, `DELETE`语句, 以及锁定读(locking read, 即 `SELECT ... FOR UPDATE` 或 `SELECT ... LOCK IN SHARE MODE`语句), 根据过滤条件是否使用了唯一索引, 还是使用范围条件来确定使用的锁:

- 对于使用了唯一索引的唯一查询条件, InnoDB只会锁定查找到的索引记录, 而不锁定前面的间隙。
- 对于其它查询条件, InnoDB 会锁定扫描到的索引范围, 通过gap locks或Next-Key locks来阻止其它会话在这个范围中插入新值。 

#### `READ COMMITTED`

> 在【读已提交】隔离级别下, 即使在同一事务中, 每次一致性读都会设置和读取自己的新快照。 有关一致性读的信息, 请参考 [14.7.2.3 非锁定一致性读](https://github.com/cncounter/translation/blob/master/tiemao_2020/44_innodb-storage-engine/14.7_innodb-locking-transaction-model.md#14.7.2.3)。

- 对于锁定读(`SELECT ... FOR UPDATE` 或者 `SELECT ... LOCK IN SHARE MODE`), `UPDATE` 语句和 `DELETE` 语句, InnoDB这时候仅锁定索引记录, 而不锁定它们之间的间隙, 因此, 其它事务可以在锁定记录旁边插入新记录。 这时候间隙锁仅用于外键约束检查和重复键检查。
- 由于禁用了间隙锁, 有可能会产生幻读问题(phantom problem), 因为其它会话可能会在间隙中插入新行。 有关幻读的信息, 请参考 [14.7.4 幻影行](https://github.com/cncounter/translation/blob/master/tiemao_2020/44_innodb-storage-engine/14.7_innodb-locking-transaction-model.md#14.7.4)
- `READ COMMITTED` 隔离级别仅支持基于行的bin-log。 如果将 `READ COMMITTED` 与 `binlog_format=MIXED` 一起使用, 则服务器会自动切换到基于行的bin-log。

- 使用 `READ COMMITTED` 还会有其他效果:

  - 对于`UPDATE` 和 `DELETE`语句, InnoDB仅持有 更新或删除行的锁。 MySQL评估完 `WHERE` 条件后, 会释放不匹配行的记录锁。这大大降低了死锁的可能性, 但还是有可能会发生。
  - 对于 `UPDATE` 语句, 如果某行已被锁定, 则InnoDB会执行半一致读(“semi-consistent” read), 将最新的提交版本返给MySQL, 让MySQL确定该行是否符合 `UPDATE` 的`WHERE`条件。 如果该行匹配(表示需要更新), 则MySQL再次读取该行, 这一次 InnoDB 要么锁定它, 要么就等待上面的锁先释放。

  示例：

  ```
  CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
  INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
  COMMIT;
  ```

  这种情况下, 因为没有索引, 所以查询和索引扫描时, 会使用隐藏的聚集索引来作为记录锁。

  | session A                                               | Session B                       |
  | ------------------------------------------------------- | ------------------------------- |
  | START TRANSACTION;<br />UPDATE t SET b = 5 WHERE b = 3; |                                 |
  |                                                         | UPDATE t SET b = 4 WHERE b = 2; |

  InnoDB 执行 `UPDATE` 时, 会为其读取到的每一行先设置一个排他锁(exclusive lock), 然后再确定是否需要对其进行修改。 如果 InnoDB不需要修改, 则会释放该行的锁。 否则, InnoDB将保留这个行锁直到事务结束。 这会影响事务的处理过程, 如下所示。

假如使用默认的 `REPEATABLE READ` 隔离级别时, 第一个 `UPDATE` 会先在其扫描读取到的每一行上设置X锁, 并且不会释放任何一个:

```
# Session A 加锁过程
x-lock(1,2); retain x-lock
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); retain x-lock
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); retain x-lock
```

因为第一个 `UPDATE` 在所有行上都保留了锁, 第二个 `UPDATE` 尝试获取任何一个锁时都会立即阻塞, 直到第一个`UPDATE`提交或回滚之后才能继续执行:

```
x-lock(1,2); block and wait for first UPDATE to commit or roll back
```

如果使用 `READ COMMITTED` 隔离级别, 则第一个 `UPDATE` 会在扫描读取到的每一行上获取X锁, 然后释放不需要修改行上的X锁:

```
# Session A 释放锁过程
x-lock(1,2); unlock(1,2)
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); unlock(3,2)
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); unlock(5,2)
```

对于第二个`UPDATE`, InnoDB会执行半一致读(“semi-consistent” read), 将最新的提交版本返给MySQL, 让MySQL确定该行是否符合 `UPDATE` 的 `WHERE`条件:

```
# Session BA 加锁过程
x-lock(1,2); update(1,2) to (1,4); retain x-lock
x-lock(2,3); unlock(2,3)
x-lock(3,2); update(3,2) to (3,4); retain x-lock
x-lock(4,3); unlock(4,3)
x-lock(5,2); update(5,2) to (5,4); retain x-lock
```

`但是, 如果 WHERE 条件中包括了索引列, 并且 InnoDB 使用了这个索引, 则获取和保留record locks时只考虑索引列`。 在下面的示例中, 第一个 `UPDATE` 在所有 `b = 2` 的行上获取并保留一个X锁。 第二个 `UPDATE` 尝试获取同一记录上的X锁时会`阻塞`, 因为也使用了 b 这列上面定义的索引。

```
CREATE TABLE t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2,3),(2,2,4);
COMMIT;

# Session A
START TRANSACTION;
UPDATE t SET b = 3 WHERE b = 2 AND c = 3;

# Session B
UPDATE t SET b = 4 WHERE b = 2 AND c = 4;
```

使用 `READ COMMITTED` 隔离级别, 与设置 `innodb_locks_unsafe_for_binlog` 选项的效果基本一样【该选项已废弃】, 但也有一些不同:

- `innodb_locks_unsafe_for_binlog` 是一个全局设置, 会影响所有会话, 而`隔离级别既可以对所有会话进行全局设置, 也可以对每个会话单独设置。`
- `innodb_locks_unsafe_for_binlog` 只能在服务器启动时设置, 而隔离级别可以在启动时设置, 也可以在运行过程中更改。

#### `READ UNCOMMITTED`

> 【读未提交】隔离级别下, `SELECT`语句以非锁定的方式执行, 但可能会用到某一行的早期版本。 所以使用此隔离级别时, 不能保证读取的一致性, 这种现象称为脏读(dirty read)。 其他情况下, 此隔离级别类似于 `READ COMMITTED`。

#### `SERIALIZABLE`

> 【串行化】这个隔离级别类似于 `REPEATABLE READ`;

-  如果禁用了 `autocommit`, 则 InnoDB 会隐式地将所有普通的 `SELECT` 语句转换为 `SELECT ... LOCK IN SHARE MODE`。
-  如果启用了自动提交(`autocommit`), 则 `SELECT` 就单独在一个事务中。 因此被认为是只读的, 如果以一致性非锁定读取方式执行, 不需要阻塞其他事务就可以实现串行化。

**如果要强制普通的 `SELECT` 语句在其他事务修改选定行时进行阻塞等待, 请禁用 `autocommit`。**

| 隔离级别         | 脏读（Dirty Read） | 不可重复读（NonRepeatable Read） | 幻读（Phantom Read） |
| :---: | :---: | :---: | :--: |
| Read uncommitted | 可能 | 可能 | 可能 |
| Read committed   | 不可能 | 可能 | 可能 |
| Repeatable read  | 不可能 | 不可能 | 可能 |
| Serializable     | 不可能 | 不可能 | 不可能 |

### autocommit, Commit, and Rollback

> 在InnoDB中, 所有用户活动都在事务中执行。 如果启用了自动提交模式(`autocommit`), 则每条SQL语句都会自己形成一个事务。

- MySQL中的每个新会话连接, 默认都是自动提交模式, 一个SQL语句如果没有产生错误, 则会在其执行完后自动提交。 如果某条SQL语句返回错误, 则根据具体的错误来决定是提交还是回滚。 详情请参考 [Section 14.22.4, “InnoDB Error Handling”](https://dev.mysql.com/doc/refman/5.7/en/innodb-error-handling.html)。
- 启用了自动提交的会话, 也可以执行多语句事务, 通过显式的 `START TRANSACTION` 或者 `BEGIN` 语句开始, 然后以 `COMMIT` 或者 `ROLLBACK` 语句结束。 具体情况请参考 [Section 13.3.1, “START TRANSACTION, COMMIT, and ROLLBACK Statements”](https://dev.mysql.com/doc/refman/5.7/en/commit.html)。
- 如果通过 `SET autocommit = 0` 禁用了自动提交模式, 则该会话会始终有一个打开的事务。 `COMMIT` 或者 `ROLLBACK` 语句则会结束当前事务并开启一个新的事务。
- 在禁用了自动提交模式的会话中, 在没有明确提交事务的情况下, 如果连接断开或者会话结束, 则MySQL会执行事务回滚。
- 有些语句会隐式地结束事务, 效果类似于在这种语句之前自动增加了一条 `COMMIT` 语句。 详细信息请参考[Section 13.3.3, “Statements That Cause an Implicit Commit”](https://dev.mysql.com/doc/refman/5.7/en/implicit-commit.html)。
- **`COMMIT` 表示需要将当前事务所做的更改进行持久化(permanent), 并对其他会话可见。 而 `ROLLBACK` 语句则取消当前事务中的修改。 `COMMIT` 和 `ROLLBACK` 都会释放所有在当前事务期间设置的 InnoDB 锁。**

##### Grouping DML Operations with Transactions

> DML(Data Manipulation Language,S、I、U、D、M等)操作分组和事务；
>
> MySQL数据库的客户端连接, 默认开启自动提交模式, 每个SQL语句执行完都会自动提交。 用过其他数据库系统的用户, 可能对这种操作模式不太习惯, 因为他们更常用的方式, 是执行一连串的DML语句, 然后再一起提交, 或者一起回滚。

想要使用多语句事务:

1. 以通过 `SET autocommit = 0` 语句关闭自动提交模式, 并在适当的时机以 `COMMIT` 或者 `ROLLBACK` 结束事务。
2. 处于自动提交状态, 可以通过 `START TRANSACTION` 开启一个事务, 并以 `COMMIT` 或者 `ROLLBACK` 结束。

##### Transactions in Client-Side Languages

> 在MySQL客户端API中, 例如 PHP, Perl DBI, JDBC, ODBC, 或者标准C调用接口, 可以将事务控制语句(如`COMMIT`)当做字符串发送给MySQL服务器, 就像普通的SQL语句(`SELECT` 和 `INSERT`)一样。 某些API还单独提供了提交事务和回滚的函数/方法。

#### Consistent Nonlocking Reads(非锁定一致性读)

> 一致性读(consistent read), 意味着 InnoDB 通过多版本技术, 为一个查询呈现出某个时间点上的数据库快照。
>
> 查询能看到这个时间点之前所有已提交事务的更改, 而看不到这个时间点之后新开的事务、或者未提交的事务所做的更改。
>
> 例外是查询可以看到同一事务中前面执行的语句所做的更改。 这种例外会引起一些异常: 如果更新了表中的某些行, 则 `SELECT` 将看到该行被更新之后的最新版本, 但其他的行可能看到的还是旧版本。 如果其他会话也更新了这张表, 则这种异常意味着我们可能会看到某种并不存在的状态。

- 如果是默认的 `REPEATABLE READ` 隔离级别, 则同一事务中的所有一致读, 都会读取该事务中第一次读取时所创建的快照。 可以提交当前事务, 并在此之后执行新的查询语句来获取最新的数据快照。
- 使用 `READ COMMITTED` 隔离级别时, 事务中的每次一致性读, 都会设置并读取自己的新快照。
- 在 `READ COMMITTED` 和 `REPEATABLE READ` 隔离级别下, 一致性读是InnoDB 处理 `SELECT` 语句的默认模式。 一致性读不会在读取的表上设置任何锁, 所以在读取时, 其它会话可以自由对这些表执行修改。
- 使用默认的 `REPEATABLE READ` 隔离级别, 执行普通的 `SELECT` 一致性读时, InnoDB 会为当前事务指定一个时刻, 根据这个时刻来确定事务中的所有查询可以看到哪些数据。 如果在这个给定的时刻之后, 另一个事务删除了一行数据并提交, 那么当前事务则看不到这一行已被删除。插入和更新操作的处理方式也是一样的。


> Note
>
> 数据库状态的快照适用于事务中的 `SELECT` 语句, 而不一定适用于DML语句(增删改)。 如果插入或修改一些行, 稍后再提交事务（T0）, 则另一个并发的 `REPEATABLE READ` 事务（T1）中的 `DELETE` 或者 `UPDATE` 语句可能会影响准备提交(just-committed)的这些行, 即使（T1）会话可能无法读取看到它们。 如果某个事务确实更新或删除已经被另一个事务提交的行, 则这些更改对于当前事务而言来说会变得可见。

```
# 场景1
SELECT COUNT(c1) FROM t1 WHERE c1 = 'xyz';
-- Returns 0: 没有匹配的行. (查不到)
DELETE FROM t1 WHERE c1 = 'xyz';
-- 删除了被另一个事务提交的某些行... (但确实会删除)

# 场景2
SELECT COUNT(c2) FROM t1 WHERE c2 = 'abc';
-- Returns 0: 没有匹配的行. (查不到)
UPDATE t1 SET c2 = 'cba' WHERE c2 = 'abc';
-- Affects 10 rows: 比如, 另一个事务 tx n2 刚刚提交了 10 行 'abc' 值. (但确实更新了...)
SELECT COUNT(c2) FROM t1 WHERE c2 = 'cba';
-- Returns 10: 本事务 txn1 这时候可以看到刚刚更新的行.
```

- 我们可以通过提交事务, 然后执行另一个 `SELECT` 或者 `START TRANSACTION WITH CONSISTENT SNAPSHOT` 语句来推进时间点。这被称为**多版本并发控制(multi-versioned concurrency control)**。

```
             Session A              Session B

           SET autocommit=0;      SET autocommit=0;
时间序
|          SELECT * FROM t;
|          empty set
|                                 INSERT INTO t VALUES (1, 2);
|
v          SELECT * FROM t;
           empty set
                                  COMMIT;

           SELECT * FROM t;
           empty set

           COMMIT;

           SELECT * FROM t;
           ---------------------
           |    1    |    2    |
```

如果要查看数据库的最新状态(freshest), 可以使用 `READ COMMITTED`隔离级别, 或者使用锁定读取:

```
SELECT * FROM t LOCK IN SHARE MODE;
```

- 使用 `READ COMMITTED` 隔离级别时, 事务中的每次一致性读都会设置并读取自己的新快照。 带有 `LOCK IN SHARE MODE` 的 SELECT 语句, 会发生`锁定读`: `SELECT` 可能会被阻塞, 直到包含最新行的事务结束为止。

##### 一致性读不支持某些DDL（drop、alter）语句

1. 一致性读不能在 `DROP TABLE` 时生效, 因为MySQL无法使用已删除的表, 而且 InnoDB 已经销毁了这张表。
2. 一致性读不能在 `ALTER TABLE` 操作时生效, 因为这个操作会创建原始表的临时副本,并在构建临时副本之后删除原始表。 在事务中重新执行一致性读时, 新表中的数据行是不可见的, 因为事务在获取快照时这些行还不存在。 这种情况下, 会返回错误信息: `ER_TABLE_DEF_CHANGED`, “Table definition has changed, please retry transaction”.

##### 对于没有指定 `FOR UPDATE` 或者 `LOCK IN SHARE MODE` 的各种查询, 其行为有所不同, 如 `INSERT INTO ... SELECT`, `UPDATE ... (SELECT)`, 以及 `CREATE TABLE ... SELECT`:

- 默认情况下, InnoDB 在这些语句中使用更强的锁, 而 `SELECT` 部分的行为类似于 `READ COMMITTED`, 即使在同一事务中, 每次一致性读都会设置并读取自己的新快照。

- 要在这种情况下执行非锁定读取, 请启用 `innodb_locks_unsafe_for_binlog` 选项, 并将事务隔离级别设置为`READ UNCOMMITTED`, `READ COMMITTED`, 或者 `REPEATABLE READ`, 以避免在读取数据行时上锁。
