- [Oracle执行计划](#oracle执行计划)
  - [SQL执行过程（步骤）](#sql执行过程步骤)
    - [语法分析](#语法分析)
    - [视图转换](#视图转换)
    - [表达式转换](#表达式转换)
    - [选择优化器](#选择优化器)
      - [Cost Based Optimizer（CBO）（优先考虑）](#cost-based-optimizercbo优先考虑)
      - [Rule Based Optimizer（RBO）（逐渐不支持）](#rule-based-optimizerrbo逐渐不支持)
    - [选择连接方式，ORACLE有三种连接方式，对多表连接ORACLE可选择适当的连接方式，有四种关联方式](#选择连接方式oracle有三种连接方式对多表连接oracle可选择适当的连接方式有四种关联方式)
      - [排序合并连接 （Sort Merge Join（SMJ））](#排序合并连接-sort-merge-joinsmj)
      - [内部链接过程](#内部链接过程)
          - [使用建议](#使用建议)
        - [嵌套循环(Nested Loops(NL))](#嵌套循环nested-loopsnl)
          - [内部连接过程](#内部连接过程)
          - [使用建议](#使用建议-1)
        - [哈希连接（Hash Join）](#哈希连接hash-join)
          - [内部连接过程](#内部连接过程-1)
          - [使用建议](#使用建议-2)
        - [笛卡尔积（Cartesian product）](#笛卡尔积cartesian-product)
          - [内部连接过程](#内部连接过程-2)
          - [出现原因](#出现原因)
    - [选择连接顺序](#选择连接顺序)
    - [选择数据的搜索路径](#选择数据的搜索路径)
    - [运行"执行计划"](#运行执行计划)
  - [数据存取](#数据存取)
    - [全表扫描（Full Table Scan）](#全表扫描full-table-scan)
      - [优势](#优势)
      - [使用建议](#使用建议-3)
    - [通过ROWID的表存取](#通过rowid的表存取)
    - [索引扫描](#索引扫描)
      - [步骤](#步骤)
      - [注意点](#注意点)
      - [索引类型](#索引类型)
        - [B Tree索引（找资料配合理解）](#b-tree索引找资料配合理解)
        - [位图索引](#位图索引)
        - [HASH索引（找资料 图文）](#hash索引找资料-图文)
        - [反转键索引](#反转键索引)
        - [函数索引](#函数索引)
        - [用于分区表](#用于分区表)
      - [索引唯一扫描](#索引唯一扫描)
      - [索引范围扫描（常见场景）](#索引范围扫描常见场景)
      - [索引全扫描(类似索引快速扫描)](#索引全扫描类似索引快速扫描)
      - [索引快速扫描](#索引快速扫描)
      - [索引跳跃式扫描](#索引跳跃式扫描)
  - [实践操作](#实践操作)
    - [准备工作](#准备工作)
    - [查看表格](#查看表格)
    - [操作方式](#操作方式)
      - [sort](#sort)
      - [filter](#filter)
      - [view](#view)
    - [常见问题](#常见问题)
  - [参考资料](#参考资料)
# Oracle执行计划

- rowid:Rowid的概念：rowid是一个伪列，既然是伪列，那么这个列就不是用户定义，而是系统自己给加上的。 对每个表都有一个rowid的伪列，但是表中并不物理存储ROWID列的值。不过你可以像使用其它列那样使用它，但是不能删除改列，也不能对该列的值进行 修改、插入。一旦一行数据插入数据库，则rowid在该行的生命周期内是唯一的，即即使该行产生行迁移，行的rowid也不会改变。

- Recursive Sql:Recursive SQL概念：有时为了执行用户发出的一个sql语句，Oracle必须执行一些额外的语句，我们将这些额外的语句称之为''recursive calls''或''recursive SQL statements''.如当一个DDL语句发出后，ORACLE总是隐含的发出一些recursive SQL语句，来修改数据字典信息，以便用户可以成功的执行该DDL语句。当需要的数据字典信息没有在共享内存中时，经常会发生Recursive calls，这些Recursive calls会将数据字典信息从硬盘读入内存中。用户不比关心这些recursive SQL语句的执行情况，在需要的时候，ORACLE会自动的在内部执行这些语句。当然DML语句与SELECT都可能引起recursive SQL.简单的说，我们可以将触发器视为recursive SQL.

- Row Source : Row Source（行源）：用在查询中，由上一操作返回的符合条件的行的集合，即可以是表的全部行数据的集合；也可以是表的部分行数据的集合；也可以为对上2个row source进行连接操作（如join连接）后得到的行数据集合。

- Predicate（谓词）：一个查询中的WHERE限制条件

- Driving Table（驱动表）：该表又称为外层表（OUTER TABLE）。这个概念用于嵌套与HASH连接中。如果该row source返回较多的行数据，则对所有的后续操作有负面影响。注意此处虽然翻译为驱动表，但实际上翻译为驱动行源（driving row source）更为确切。一般说来，是应用查询的限制条件后，返回较少行源的表作为驱动表，所以如果一个大表在WHERE条件有有限制条件（如等值限 制），则该大表作为驱动表也是合适的，所以并不是只有较小的表可以作为驱动表，正确说法应该为应用查询的限制条件后，返回较少行源的表作为驱动表。在执行 计划中，应该为靠上的那个row source，后面会给出具体说明。在我们后面的描述中，一般将该表称为连接操作的row source 1.

- Probed Table（被探查表）：该表又称为内层表（INNER TABLE）。在我们从驱动表中得到具体一行的数据后，在该表中寻找符合连接条件的行。所以该表应当为大表（实际上应该为返回较大row source的表）且相应的列上应该有索引。在我们后面的描述中，一般将该表称为连接操作的row source 2.

  > Driving Table和Probed Table 见嵌套循环和哈希连接

- concatenated index（组合索引）：由多个列构成的索引，如create index idx_emp on emp（col1， col2， col3， ……），则我们称idx_emp索引为组合索引。在组合索引中有一个重要的概念：引导列（leading column），在上面的例子中，col1列为引导列。当我们进行查询时可以使用“where col1 = ？ ”，也可以使用“where col1 = ？ and col2 = ？”，这样的限制条件都会使用索引，但是“where col2 = ？ ”查询就不会使用该索引。所以限制条件中包含先导列时，该限制条件才会使用该组合索引。

- selectivity（可选择性）：比较一下列中唯一键的数量和表中的行数，就可以判断该列的可选择性。 如果该列的“唯一键的数量/表中的行数”的比值越接近1，则该列的可选择性越高，该列就越适合创建索引，同样索引的可选择性也越高。在可选择性高的列上进 行查询时，返回的数据就较少，比较适合使用索引查询。

## SQL执行过程（步骤）

###  语法分析

- 分析语句的语法是否符合规范，衡量语句中各表达式的意义

- 检查语句中涉及的所有数据库对象是否存在，且用户有相应的权限

### 视图转换

  > 将涉及视图的查询语句转换为相应的对基表查询语句

###  表达式转换

> 将复杂的 SQL 表达式转换为较简单的等效连接表达式

### 选择优化器

> 不同的优化器一般产生不同的“执行计划”

#### Cost Based Optimizer（CBO）（优先考虑）

- 依赖于数据表中数据的统计分布n

- 计算各种可能执行计划的代价，从中选用cost最低的方案

- 星型连接排列查询，哈希连接查询，函数索引和并行查询依赖于CBO

- SQL语句FROM子句后面的表的个数不宜太多，以免造成分析消耗太大

- 如果实在是要访问很多表，则最好使用 ORDER 提示，强制固定的访问顺序

- 对数据经常有增、删、改的表最好定期对表和索引进行分析

#### Rule Based Optimizer（RBO）（逐渐不支持）

- 根据SQL书写规则产生执行计划

- 对数据不敏感,执行计划不改变

- 对程序员写SQL要求比较苛刻

- 可以通过加rule提示，强制使用RBO

### 选择连接方式，ORACLE有三种连接方式，对多表连接ORACLE可选择适当的连接方式，有四种关联方式

#### 排序合并连接 （Sort Merge Join（SMJ））

#### 内部链接过程

- 生成row source1需要的数据，然后对这些数据按照连接操作关联列进行排序

- 生成row source2需要的数据，然后对这些数据按照与sort source1对应的连接操作关联列进行排序

- 最后两边已排序的行被放在一起执行合并操作，即将2个row source按照连接条件连接起来

###### 使用建议

- 合并两个row source的过程是串行的，但是可以并行访问这两个row source

- 如果row source已经在连接关联列上已经排序，则可以大大提高这种连接操作的连接速度

- 如果2个row source都已经预先排序，则这种连接方法的效率也是蛮高的

- SMJ经常不是一个特别有效的连接方法

##### 嵌套循环(Nested Loops(NL))

###### 内部连接过程

- 2层嵌套循环

- 需要用row source1中的每一行，去匹配row source2中的所有行

###### 使用建议

- 通常应该将小表或返回较小 row source的表作为驱动表（用于外层循环）

- 保持row source1尽可能的小与高效的访问row source2是影响这个连接效率的关键问题

- 可以先返回已经连接的行，而不必等待所有的连接操作处理完才返回数据，这可以实现快速的响应时间

##### 哈希连接（Hash Join）

###### 内部连接过程

- 较小的row source被用来构建hash table与bitmap

- 第2个row source被用来被hansed，并与第一个row source生成的hash table进行匹配

###### 使用建议

- 当被构建的hash table与bitmap能被容纳在内存中时，这种连接方式的效率极高

- 要使哈希连接有效，需要设置HASH_JOIN_ENABLED=TRUE，缺省情况下该参数为TRUE

- 设置 hash_area_size参数，以使哈希连接高效运行

##### 笛卡尔积（Cartesian product）

###### 内部连接过程

- 当两个row source做连接，但是它们之间没有关联条件时使用

- 笛卡尔乘积是一个表的每一行依次与另一个表中的所有行匹配

###### 出现原因

- 通常是代码疏漏造成

- SQL重写也可能出现这个

### 选择连接顺序

>对多表连接ORACLE选择哪一对表先连接，选择这两表中哪个表做为源数据表

### 选择数据的搜索路径

> 根据以上条件选择合适的数据搜索路径，如是选用全表搜索还是利用索引或是其他的方式

### 运行"执行计划"

## 数据存取

### 全表扫描（Full Table Scan）

#### 优势

- 一次I/O能读取多块数据块，减少了I/O总次数，提高了系统的吞吐量

- 每个数据块只

- 被读一次

#### 使用建议

- 在较大的表上不建议使用全表扫描

- 除非取出数据的比较多，超过总量的5% ~10%

- 使用并行查询功能

### 通过ROWID的表存取

- 通过ROWID来存取数据是Oracle存取单行数据的最快方法

- 不会用到多块读操作，一次I/O只能读取一个数据块

### 索引扫描

#### 步骤

- 扫描索引得到对应的rowid值

- 通过找到的rowid从表中读出具体的数据

#### 注意点

- 每步都是单独的一次I/O,但索引绝大多数都已经CACHE到内存中(逻辑I/O)

- 如果查询的数据能全在索引中找到，就可以避免进行第2步操作

- 如果sql语句中对索引列进行排序，在执行计划中不需要再对索引列进行排序

- 如果表比较大，则其数据不可能全在内存中，所以很有可能是物理I

- 如果对大表进行索引扫描，取出的数据如果大于总量的5%~ 10%，使用索引扫描会效率很低

- 类型不匹配或者非等于操作，可能导致索引用不上

#### 索引类型

##### B Tree索引（找资料配合理解）

- 叶子节点带有相应行的rowid

- 获取rowid需要可能需要访问多个数据块

##### 位图索引

- 适用于列的值个数较少的情况

- 仅适合于少量更新的情况

##### HASH索引（找资料 图文）

- 不适合选择性低的数据，会造成定位某一条记录时浪费多次表数据的访问

- 可能会浪费比较多的空间

- 只需要根据查询的值进行一次Hash函数计算，就可以得到ROWID的位置

##### 反转键索引

- 将索引键的值反转过来进行索引

- 用于索引列的前缀重复较多的情况，如学号

##### 函数索引

> 用于需要对列使用函数操作的情况

##### 用于分区表

- 分区索引

- 本地和全局索引

#### 索引唯一扫描

> 唯一索引查找一个数值->返回单个ROWID->获取对应的数据

- 常见场景

  - 需要存在UNIQUE 或PRIMARY KEY 约束

  - 配合TABLE ACCESS BY ROWID使用

#### 索引范围扫描（常见场景）

- 用于使用一个索引存取多行数据

- 在唯一索引列上使用了range操作符（>.. >=.）

- 在组合索引上，只使用部分列进行查询，导致查询出多行

- 对非唯一索引列上进行的任何查询

#### 索引全扫描(类似索引快速扫描)

- 查询出的数据都必须从索引中可以直接得到

- 不能使用多块读功能，不能使用并行读入

- 数据是以排序顺序被返回

- 索引全扫描可能并不高效

#### 索引快速扫描

- 查询出的数据都必须从索引中可以直接得到

- 可以使用多块读功能，可以使用并行读入

- 数据不是以排序顺序被返回

#### 索引跳跃式扫描

- 从Oracle9i开始才有的特性

- 允许优化器使用组合索引，即便索引的前导列没有出现在WHERE子句中

- 索引跳跃式扫描比全索引扫描要快的多

## 实践操作

### 准备工作

- AUTOTRACE

  ```

  SET LINE 32767

  SET PAGES 0

  SET FEEDBACK OFF

  SET AUTOTRACE ON

  -- SET AUTOTRACE TRACEONLY 

  alter session set nls_date_format='yyyy-mm-dd hh24:mi:ss';

  ```

- explain plan

  - explain plan for ...

  - select * from table(dbms_xplan.display);

### 查看表格

- 次序: 从上到下，从右到左

- 显示的值为估算值，结果不一样的sql估算值没有可比性

### 操作方式

#### sort

> 如果row source已经排好序，则排序就不会执行，例如使用full index scan的情况

>

> • SORT UNIQUE occurs if a user specifies a DISTINCT clause or if an operation requires unique values for the next step.

>

> • SORT AGGREGATE does not actually involve a sort. It is used when aggregates are being computed across the whole set of rows.

>

> • SORT GROUP BY is used when aggregates are being computed for different groups in the data. The sort is required to separate the rows into different groups.

>

> • SORT JOIN happens during a sort-merge join if the rows need to be sorted by the join key.

>

> • SORT ORDER BY is required when the statement specifies an ORDER BY that cannot be satisfied by one of the indexes.

- order by clauses

- group by

- sort merge join

#### filter

> 用于谓词过滤数据或者连接

#### view

### 常见问题

> 列类型不一致导致不走索引扫描

## 参考资料

- [Interpreting Explain Plan](http://www.akadia.com/services/ora_interpreting_explain_plan.html)
- [深入研究B树索引](http://space.itpub.net/index.php?uid/9842/action/spacelist/php/1)
- [Oracle IO问题解析](http://www.google.com.hk/search?q=Oracle+IO问题解析&btnG=搜索&sitesearch=www.HelloDBA.com)
- [oracle逻辑IO的秘密](http://www.google.com.hk/search?q=oracle逻辑IO的秘密&btnG=搜索&sitesearch=www.HelloDBA.com)
- [Performance Tuning Guide.pdf](http://docs.oracle.com/cd/B19306_01/server.102/b14211.pdf)