> 参考来源：
>
> 康师傅：https://www.bilibili.com/video/BV1iq4y1u7vj?p=136
>
> 爱编程的大李子：https://blog.csdn.net/LXYDSF/article/details/126338994

# 一、EXPLAIN 概述

**定位了查询慢的SQL之后，我们就可以使用 EXPLAIN 或 DESCRIBE 工具做针对性的分析查询语句**。DESCRIBE 语句的使用方法与 EXPLAIN 语句是一样的，并且分析结果也是一样的。

MySQL 中有专门负责优化 SELECT 语句的优化器模块，主要功能：通过计算分析系统中收集到的统计信息，为客户端请求的 Query 提供它认为最优的**执行计划**（他认为最优的数据检索方式，但不见得是 DBA 认为是最优的，这部分最耗费时间)。

这个执行计划展示了接下来具体执行查询的方式，比如多表连接的顺序是什么，对于每个表采用什么访问方法来具体执行查询等等。 MySQL 为我们提供了 EXPLAIN 语句来帮助我们查看某个查询语句的具体执行计划，看懂 EXPLAIN 语句的各个输出项，可以有针对性的提升我们查询语句的性能。

## 1. 能做什么？

- 表的读取顺序
- 数据读取操作的操作类型
- 哪些索引可以使用
- **哪些索引被实际使用**
- 表之间的引用
- **每张表有多少行被优化器查询**

## 2. 官网介绍

- 5.7版本：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html
- 8.0版本：https://dev.mysql.com/doc/refman/8.0/en/explain-output.html

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/63.png)

## 3. 版本情况

- MySQL 5.6.3 以前只能 **EXPLAIN SELECT** ；MYSQL 5.6.3 以后就可以 **EXPLAIN SELECT，UPDATE，DELETE**
- 在5.7以前的版本中，想要显示 **partitions** 需要使用 **explain partitions** 命令；想要显示 **filtered** 需要使用 **explain extended** 命令。在5.7版本后，默认explain 直接显示 partitions 和 filtered 中的信息

注意：**EXPLAIN 仅仅是查看执行计划，不会真实的执行 sql**

# 二、EXPLAIN 使用

## 1. 基本语法

EXPLAIN 或 DESCRIBE语句的语法形式如下：

```mysql
EXPLAIN SELECT select_options
或者
DESCRIBE SELECT select_options
```

如果我们想看看某个查询的执行计划的话，可以在具体的查询语句前边加一个EXPLAIN ，就像这样：

```mysql
EXPLAIN SELECT 1;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/64.png)

输出的上述信息就是所谓的**执行计划**。在这个执行计划的辅助下，我们需要知道应该怎样改进自己的查询语句以使查询执行起来更高效。其实除了以 **SELECT** 开头的查询语句，其余的 **DELETE**、 **INSERT**、**REPLACE** 以及 **UPDATE** 语句等都可以加上 **EXPLAIN** ，用来查看这些语句的执行计划，只是平时我们对 **SELECT** 语句更感兴趣。

注意：执行 EXPLAIN 时并没有真正的执行该后面的语句，因此可以安全的查看执行计划。

## 2. EXPLAIN 各列作用

| 列名              | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| **id**            | 在一个大的查询语句中每个 SELECT 关键字都对应一个**唯一的 id** |
| **select_type**   | SELECT关键字对应的那个查询的类型                             |
| **table**         | 表名                                                         |
| **partitions**    | 匹配的分区信息                                               |
| **type**          | 针对单表的访问方法                                           |
| **possible_keys** | 可能用到的索引                                               |
| **key**           | 实际上使用的索引                                             |
| **key_len**       | 实际使用到的索引长度                                         |
| **ref**           | 当使用索引列等值查询时，与索引列进行等值匹配的对象信息       |
| **rows**          | 预估的需要读取的记录条数                                     |
| **filtered**      | 某个表经过搜索条件过滤后剩余记录条数的百分比                 |
| **Extra**         | 一些额外的信息                                               |



### ① table

不论我们的查询语句有多复杂，里边儿包含了多少个表 ，到最后也是需要对每个表进行单表访问的，所以 MySQL 规定 EXPLAIN 语句输出的每条记录都对应着某个单表的访问方法，该条记录的 **table 列代表着该表的表名**（有时不是真实的表名字，可能是简称）。

```mysql
EXPLAIN SELECT * FROM s1 INNER JOIN s2;
```

如下图，**一张表对应一个记录**。

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/65.png)



注：临时表也会有对应的记录，比如我们使用UNION时就会出现临时表

```mysql
EXPLAIN SELECT * FROM s1 UNION SELECT * FROM s2;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/66.png)

这是因为 Union 是取表的并集，需要建临时表进行去重，因此会有三条记录。可以看到第三条记录的 **Extra** 就标识了它是一张临时表哦。**临时表 id 是 Null**。



### ② id

**在查询语句中每出现一个 SELECT 关键字，MySQL 就会为它分配一个唯一的id** ，代表着一次查询。这个 id 就是 **EXPLAIN** 语句查询结果的第一列。所以上面 `table` 的第一个查询两行数据的 id 都为 1，而**第二查询有两个 SELECT 则一条 id 为 1，一条 id 为 2**

特例：

```mysql
EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key2 FROM s2 WHERE common_field = 'a');
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/67.png)

两个记录的 id 都是 1 

这是因为优化器会对上面的 sql 语句进行优化，将其转换为多表连接，而不是子查询。因为子查询其实是一种嵌套查询的情况，其时间复杂度是 O(n^m)，其中 m 是嵌套的层数，而多表查询的时间复杂度是 O(n*m)

> **小结：**
>
> 1. id如果相同，可以认为是一组，从上往下顺序执行
> 2. 在所有组中，id值越大，优先级越高，越先执行
> 3. 关注点：id 号每个号码，表示一趟独立的查询, 一个 sql 的查询趟数越少越好（子查询和连接查询的效率问题）



### ③ select_type

一条大的查询语句里边可以包含若干个 SELECT 关键字，**每个 SELECT 关键字代表着一个小的查询语句**，而每 SELECT 关键字的 FROM 子句中都可以包含若干张表(这些表用来做连接查询)，**每一张表都对应着执行计划输出中的一条记录**，对于在同一个 SELECT 关键字中的表来说，它们的id值是相同的。

MysSQL 为每一个 SELECT 关键字代表的小查询都定义了一个称之为 select_type 的属性，意思是我们只要知道了某个小查询的 select_type 属性，就知道了这个小查询在整个大查询中扮演了一个什么角色，我们看一下 select_type 都能取哪些值，请看官方文档：

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/68.png)



- **SIMPLE**

  查询语句中不包含 **UNION** 或者 **子查询** 的查询都算作是 **SIMPLE** 类型，join 连接查询也是 SIMPLE 类型

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/69.png)

- **Union 联合查询**。其左边的查询是 **Primary**，右边的查询类型是 **Union**，去重的临时表查询类型是： **Union Result**

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/70.png)

  - 对于包含 **UNION** 或者 **UNION ALL** 的大查询来说，它是由几个小查询组成的，其中除了最左边的那个查询的 **select_type** 值就是 **PRIMARY**，其余的小查询的 **select_type** 值就是 **UNION**
  - **MySQL** 选择使用临时表来完成 **UNION** 查询的去重工作，针对该临时表的查询的 **select_type** 就是 **UNION RESULT**
  - 对应子查询的大查询来说，子查询是外边的那个是 **PRIMARY**

- **SUBQUERY**：不会被优化成多表连接的子查询

  如果包含子查询的查询语句不能够转为多表连接的形式(也就是不会被优化器进行自动的优化)，并且该子查询是不相关的子查询

  该子查询的第一个 **SELECT** 关键字代表的那个查询的 **select_type** 就是 **SUBQUERY**。也就是外层查询是 **Primary**，内层查询是  **SUBQUERY**

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/71.png)

  如果子查询不能被转换为多表连接的形式，并且该子查询是相关子查询。

  比如下面的查询在内部子查询使用了外部的表。则该子查询的第一个 **SELECT** 关键字代表的那个查询的 **select_type** 就是**DEPENDENT SUBQUERY**。 外层查询是 **Primary**，内层查询是 **DEPENDENT SUBQUERY**

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/72.png)

  需要注意的是 **DEPENDENT SUBQUERY** 的查询语句可能会被执行多次，因为内层查询依赖于外层的查询，因此可能会是外层传一个值，内层就执行一次的模式。

- 包含 **UNION** 或者 **UNION ALL** 的**子查询**

  在包含 **Union** 或者 **Union All** 的子查询 sql 中，如果各个小查询都依赖于外查询，那么除了最左边的小查询外，各个小查询的类型都是 **DEPENDENT UNION**

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/73.png)

  外查询是 **Primary**，最左边的子查询是 **DEPENDENT SUBQUERY**，后面的子查询是 **DEPENDENT UNION**，临时去重表的类型是 **Union Result**。这里大家可能要困惑，第一个子查询中也没有看到依赖 s1 啊。这其实也是优化器会在执行时进行优化，将 **IN** 改成 **Exist**，并且把外部的表移到内部去。

- **关于派生表的子查询**

  对于包含 **派生表** 的查询，该派生表对应的子查询的 **select_type** 就是 **DERIVED**

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/74.png)

- **子查询的物化后与外层连接查询**

  当优化器在执行子查询时选择把子查询优化成为一张**物化表**，与外层查询进行连接查询时。

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/75.png)

  从下往上看，子查询的查询类型是 **MATERIALIZED**；物化过程是基于 id 为 2 的查询结果表进行的，其 table 是 **subquery 2**，查询类型是 **SIMPLE**，而外层也相当于是与固定的直接值进行查询，其类型也是 **SIMPLE**

### ④ partitions (可略)

代表分区表中的命中情况，非分区表，该项为 **NULL**。一般情况下我们的查询语句的执行计划的 **partitions** 列的值都是 **NULL**

官方文档：https://dev.mysql.com/doc/refman/8.0/en/alter-table-partition-operations.html

如果想详细了解，可以如下方式测试。创建分区表：

```mysql
-- 创建分区表，
-- 按照id分区，id<100 p0分区，其他p1分区
CREATE TABLE user_partitions (
    id INT auto_increment,
    NAME VARCHAR(12),PRIMARY KEY(id))
    PARTITION BY RANGE(id)(
    PARTITION p0 VALUES less than(100),
    PARTITION p1 VALUES less than MAXVALUE
 );
```

查询 id 大于200（200>100，p1分区）的记录，查看执行计划，partitions 是 p1，符合我们的分区规则

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/76.png)

### ⑤ type ☆

执行计划的一条记录就代表着MysQL对某个表的 **执行查询时的访问方法**，又称"访问类型”，其中的 **type** 列就表明了这个访问方法是啥，是较为重要的一个指标。比如，看到 **type** 列的值是 **ref**，表明 **MySQL** 即将使用 **ref** 访问方法来执行对 **s1** 表的查询。

完整的访问方法如下（根据顺序**越往后查询效率越低**）： **system** 、**const**、**eq_ref** 、**ref** 、**fulltext**、**ref_or_null**、**index_merge**、**unique_subquery**、**index_subquery**、 **range**、**index**、**ALL**。

- **system**

  当**表中只有一条记录**，并且该表中存**储引擎统计数据是精确的**，比如 MYISAM，Memory，那么其访问方法就是 **System**。这种方式几乎是性能最高的，当然我们**几乎用不上**。

- **Const**

  当我们根据**主键**或者**唯一的二级索引**，与**常数**进行等值匹配时，对单表的访问方法就是 **const**。

  比如对**主键**与**常数**匹配，进行**等值查询**

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE id = 10005;
  ```

  对 **Unique** 标识的**唯一二级索引** key2 与**常数**匹配，进行**等值查询**。

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key2 = 10066;
  ```

- **eq_ref**

  在进行 **连接查询** 时，如果 **被驱动表** 是通过 **主键** 或者 **唯一二级索引** 等值匹配的方式进行查询的，那么**被驱动表**的访问方式是 **eq_ref**

  ```mysql
  EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id;
  ```

  查询结果 **s2** 的 type 为 **eq_ref**

- **ref**

  当使用普通的**二级索引**与**常量**进行**等值匹配**时，type 是 **ref**。

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';
  ```

- **ref_or_null**

  当使用普通的**二级索引**进行**等值匹配**时，当索引值可以是 **Null** 时，type 是 **ref_or_null**。

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key1 IS NULL;
  ```

- **index_merge**

  当进行单表访问时，如果**多个查询字段分别建立了单列索引**，使用 **OR** 连接，其访问类型是 **index_merge**。同时还可以看到 key 这一字段，是使用了**多个索引**

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/77.png)

  如果使用 **AND** 连接，则引用类型为 **ref**，这是因为用 AND 连接两个查询时，实际上只使用了 key1 的索引

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' AND key3 = 'a';
  ```

- **unique_subquery**

  针对一些包含 **IN** 的子查询的查询语句中，如果**优化器决定将 IN 子查询优化为 EXIST 子查询**，而且**子查询可以使用主键进行等值匹配**的话，那么该**子查询**执行计划的 type 就是 **unique_subquery**

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key2 IN (SELECT id FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
  ```

- **range**

  如果使用 **索引** 获取某些 **范围区间** 的记录，那么就可能使用到 **range** 访问方法

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 IN ('a', 'b', 'c');
  EXPLAIN SELECT * FROM s1 WHERE key1 > 'a' AND key1 < 'b';
  ```

- **index**

  当我们可以使用**索引覆盖**，但是**需要扫描的全部的索引记录**时，该表的访问方式就是 **index**。

  索引覆盖：比如 sql 语句中，key_part2 ，key_part3 都属于联合索引 idx_key_part(key_part1, key_part2, key_part3) 的一部分，在查找数据时可以用上这个联合索引，而不用进行回表操作，这种情况即索引覆盖
  ```mysql
  EXPLAIN SELECT key_part2 FROM s1 WHERE key_part3 = 'a';
  ```

- **ALL**

  全表扫描 `ALL`，应尽量避免全表扫描

> 小结
>
> 结果值从最好到最坏依次是： system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL 其中比较重要的几个提取出来（前4个）
>
> SQL 性能优化的目标：至少要达到 range 级别，要求是 ref 级别，最好是 consts 级别（阿里巴巴开发手册要求）

### ⑥ possible_keys 和 key

在 EXPLAIN 语句输出的执行计划中, **possible_keys** 列表示在某个查询语句中，对某个表执行单表查询时**可能用到的索引**有哪些。一般查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询使用。**key** 列表示**实际用到的索引**有哪些，如果为 **NULL**，则**没有使用索引**。

对应优化器来说，可以选择的 **possible_keys** 越少越好，因为选项越多，进行过滤花的时间也就对应更多。另外，优化器会对各个索引进行查询的效率进行评估，以此来选择实际使用的 **key**。而且由于优化器会对 sql 进行优化，完全**可能会出现 possible_keys 是 null，但是 key 不为 null** 的情况

### ⑦  key_len ☆

**实际使用的索引的长度，单位是字节**。可以帮助你检查是否充分利用了索引，**主要针对联合索引**具有一定的参考，**对同一索引来说，key_len 值越大越好**

- **int 类型**的索引**占 4 个字节**
- 字符类型的索引根据**字段长度**和**编码类型**计算。比如：类型是 varchar(100)，100 个字符，utf-8 每个字符占 3 个字节，共 300 个字节，加上变长列表 2 个字节，**一共 102 字节**
- 索引允许为 **Null** 占用 **1 个字节**

> 练习：**key_len的长度计算公式**：
>
> varchar(10)变长字段且允许NULL = 10 \*( character set：utf8=3, gbk=2, latin1=1) + 1(NULL)+2(变长字段)
> varchar(10)变长字段且不允许NULL = 10 \* ( character set：utf8=3 ,gbk=2, latin1=1) +2(变长字段)
> char(10)固定字段且允许NULL = 10 \*( character set：utf8=3, gbk=2, latin1=1) +1(NULL)
> char(10)固定字段且不允许NULL = 10 \* ( character set：utf8=3,gbk=2,latin1=1)

### ⑧ ref

当**索引列进行等值查询**时，与**索引列匹配的对象信息**。

- 比如只是一个**常数**或者是**某个列**，其 ref 是 **const**

```mysql
EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';
```

- 当进行**多表连接查询**时，对被驱动表 s2 执行的查询引用了**db.s1.id**(库.表.字段)字段进行等值查询，则 ref 是 **db.s1.id**

```mysql
EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id;
```

- 当**连接条件使用函数**时，其 **ref** 就是 **func**

```mysql
 EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s2.key1 = UPPER(s1.key1);
```

### ⑨ rows ☆

预估的**需要读取的记录条目数**，条目数**越小越好**。这是因为值越小，加载I/O的页数就越少

### ⑩ filtered

**经过搜索条件后过滤剩下的记录所占的百分比，百分比越高越好**。比如同样 rows 是 40，如果 filter 是 100，则是从 40 条记录里进行查找，如果 filter 是 10，则是从 400 条记录里进行查找，相比较而言当然是前者的效率更高哦。

- 如果执行的是**单表扫描**，那么计算时需要估计除了**对应搜索条件外的其他搜索条件满足的记录有多少条**，例如：

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 > 'z' AND common_field = 'a';
  ```

  结果是 10，表示有 347 条记录满足 key1 > ‘z’ 的条件，这 347 条记录的 10% 满足 common_field = ‘a’ 条件。

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/78.png)

- 实际上，对于**单表查询**，这个字段没有太大的意义，我们更加**关注连接查询时的 filtered 值**，它决定了**被驱动表要执行的次数**。

  ```mysql
  EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key1 
  WHERE s1.common_field = 'a';
  ```

  结果如下。在标明驱动表 s1 提供给被驱动表的记录数是 9895 条，其中 989.5 条满足过滤条件s1.key1 = s2.key1，那么被驱动表需要执行 990 次查询。

  ![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/79.png)

  > filtered=(最终查询结果/rows列数据)*100%，越大表示过滤后的数据，越是最终结果。
  > 相比较filtered越小，减少了数据再次过滤的性能

### ⑪ extra ☆

顾名思义，**Extra** 列是用来说明一些额外信息的，包含**不适合在其他列中显示但十分重要的额外信息**。我们可以通过这些额外信息来**更准确的理解MySQL到底将如何执行给定的查询语句**。MySQL提供的额外信息有好几十个，下面挑选几个比较重要的额外信息：

#### - No tables used

当查询语句的**没有 FROM 子句**时将会提示该额外信息

```mysql
EXPLAIN SELECT 1;
```

#### - Impossible WHERE

当查询条件**永远不可能满足，查不到数据时**会出现该信息。

```mysql
EXPLAIN SELECT * FROM s1 WHERE 1 != 1;
```

#### - Using where

当**没有使用索引**，普通的 **where** 查询即**全表扫描**时，会出现该信息

```mysql
EXPLAIN SELECT * FROM s1 WHERE common_field = 'a';
```

使用**索引查询**，则默默使用索引，显示为 `NULL` 什么额外信息也没有。

如果**使用索引加普通 where**，那还是 **using where**

```mysql
EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' AND common_field = 'a';
```

#### - No matching min/max row

当查询语句中有 **MIN**、**MAX** 等聚合函数，但是并**没有符合 where 条件的搜索记录**时，会提供额外信息 **No matching min/max row**（表中根本没有满足 where 条件的字句，找 min、max 没有意义）

```mysql
EXPLAIN SELECT MIN(key1) FROM s1 WHERE key1 = 'fhhfe55 乱写的条件，表里面找不到数据';
```

#### - Select tables optimized away

**查询已经优化后的表**。当查询语句中有 **MIN**、**MAX** 等聚合函数，**有符合 where 条件**的搜索记录时

```mysql
EXPLAIN SELECT MIN(key1) FROM s1 WHERE key1 = 'abc';
```

#### - Using index

在使用覆盖索引的情况提示。所谓**覆盖索引**，就是**索引中覆盖了需要查询的所有字段，不需要再使用聚簇索引进行回表查找**。比如下面的例子，使用 key1 作为查找条件，该字段建立了索引，B+ 树可以查找到 key1 字段和主键，因此下面只查找 key1 字段就不用进行回表操作，这是非常好的情况。
```mysql
EXPLAIN SELECT key1 FROM s1 WHERE key1 = 'a';
```

#### - Using index condition

搜索列中**虽然出现了索引列**，但是**不能够使用索引**，这种情况是比较坑的~

比如下面的查询**虽然出现了索引列作为查询条件，但是还是需要进行回表查找**，回表操作是一个随机 I/O，比较耗时。

```mysql
EXPLAIN SELECT * FROM s1 WHERE key1 > 'z' AND key1 LIKE '%a';
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/80.png)

上面这种情况可以使用 **索引下推** (可以通过配置项进行配置)，**使用 WHERE key1 > ‘z’ 得到的结果先进行模糊匹配 key1 LIKE ‘%a’，然后再去回表**，就可以减少回表的次数了。

#### - Using join buffer

在**连接查询中**，当**被驱动表不能够有效利用索引**实现提升速度，数据库就**使用缓存来尽可能提升一些性能**。

```mysql
EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.common_field = s2.common_field;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/81.png)

#### - Not exists

当我们使用**左（外）连接时**，如果 **WHERE** 子句中包含**要求被驱动表的某个列等于 NULL 值**的搜索条件，而且**那个列又是不允许存储 NULL 值**的，那么在该表的执行计划的 **Extra** 列就会提示 **Not exists** 额外信息。类似上面的 Impossible WHERE 永远不会满足。

```mysql
# s2.id 为主键，不能为 NULL
EXPLAIN SELECT * FROM s1 LEFT JOIN s2 ON s1.key1 = s2.key1 WHERE s2.id IS NULL;
```

#### - Using intersect(…) 、 Using union(…) 和 Using sort_union(…)

如果执行计划的 **Extra** 列出现了 **Using intersect(...)** 提示，说明准备使用 **Intersect 索引合并**的方式执行查询，**括号中的...表示需要进行索引合并的索引名称**；

如果出现了 **Using union(...)** 提示，说明准备使用 **Union 索引合**并的方式执行查询；

如果出现了 **Using sort_union(...)** 提示，说明准备使用 **Sort-Union 索引合并**的方式执行查询
```mysql
EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key3 = 'a';
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/82.png)

#### - Zero limit

当 **LIMIT** 子句的参数为 **0** 时，**表示不打算从表中读出任何记录**，将会提示该额外信息

```mysql
EXPLAIN SELECT * FROM s1 LIMIT 0;
```

#### - Using filesort

很多情况下**排序操作无法使用到索引**，只能在**内存中**（记录较少的时候）或者**磁盘中**（记录较多的时候）进行**排序**， MySQL 把这种在内存中或者磁盘上进行排序的方式统称为**文件排序**（英文名：**filesort**）。这种情况时比较悲壮的~

```mysql
EXPLAIN SELECT * FROM s1 ORDER BY common_field LIMIT 10;
```

#### - Using temporary

在许多查询的执行过程中，MySQL 可能会**借助临时表**来完成一些功能，比如**去重、排序**之类的，比如我们在执行许多包含 **DISTINCT**、**GROUP BY**、**UNION** 等子句的查询过程中，如果**不能有效利用索引**来完成查询，MySQL很有可能寻求**通过建立内部的临时表**来执行查询。如果查询中使用到了内部的临时表，在执行计划的 **Extra** 列将会显示 **Using temporary** 提示

```mysql
EXPLAIN SELECT DISTINCT common_field FROM s1;
EXPLAIN SELECT common_field, COUNT(*) AS amount FROM s1 GROUP BY common_field;
```

执行计划中出现 **Using temporary** 并不是一个好的征兆，因为建立与维护临时表要付出很大成本的，所以我们**最好能使用索引来替代掉使用临时表**。

# 三、小结

- EXPLAIN 不考虑各种 Cache，只考虑 SQL 本身
- EXPLAIN 不能显示 MySQL 在执行查询时所作的优化工作
- EXPLAIN 不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况部分统计
- 信息是估算的，并非精确值
