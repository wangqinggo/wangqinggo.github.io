[..](./../../middleware/index.md)

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
