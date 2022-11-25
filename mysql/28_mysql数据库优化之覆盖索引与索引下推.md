> 参考来源：
>
> 康师傅：https://www.bilibili.com/video/BV1iq4y1u7vj?p=147
>
> 爱编程的大李子：https://blog.csdn.net/LXYDSF/article/details/126606855

# 一、覆盖索引

**一个索引包含了满足查询结果的数据就叫做覆盖索引**。它包括在查询里的 SELECT、JOIN 和 WHERE 子句用到的**所有列**（即建索引的字段正好是覆盖查询条件中所涉及的字段）。

简单说就是， **索引列+主键** 包含 **SELECT 到 FROM 之间查询的列**。

**举例一：**

```mysql
# 删除之前的索引
DROP INDEX idx_age_stuno ON student;

# 创建 age,NAME 两个字段的联合索引
CREATE INDEX idx_age_name ON student (age,NAME);

# 不等号查找 导致索引失效
EXPLAIN SELECT * FROM student WHERE age <> 20;

# 覆盖索引下，上面的语句 不等号没有使索引失效
EXPLAIN SELECT age,NAME FROM student WHERE age <> 20;
```

> 注意：前面我们提到如果使用上<>就不会使用上索引了 并不是绝对的。我们讲解的关于 索引失效以及索引优化都是根据效率来决定的。对于二级索引来说：查询时间 = 二级索引计算时间 + 回表查询时间，由于我们使用的是覆盖索引，回表查询时间 = 0，索引优化器考虑到这一点就使用上 二级索引了

**举例二：**

```mysql
# LIKE 以 % 开头的模糊查询导致索引失效
EXPLAIN SELECT * FROM student WHERE NAME LIKE '%abc';

# 由于上面创建的索引 idx_age_name 中包含了，id, age, name 三个字段，所以下面使用上了索引
EXPLAIN SELECT id,age FROM student WHERE NAME LIKE '%abc';
```

**覆盖索引的利弊**

- **好处1：避免Innodb表进行索引的二次查询（回表）**

  对于 Innodb 来说，二级索引在叶子节点中所保存的是行的主键信息，如果是用二级索引查询数据，在查找到相应的键值后，还需通过主键进行二次查询才能获取我们真实所需要的数据。

  在覆盖索引中，二级索引的键值中可以获取所要的数据，避免了对主键的二次查询，减少了 IO 操作，提升了查询效率。

- **好处2：可以把随机 IO 变成顺序 IO 加快查询效率**

  由于覆盖索引是按键值的顺序存储的，对于 I/O 密集型的范围查找来说，对比随机从磁盘读取每一行的数据 I/O 要少的多，因此利用覆盖索引在访问时也可以把磁盘的随机读取的 I/O 转变成索引查找的顺序 I/O。

  由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。

- **弊端**

  **索引字段的维护** 总是有代价的。因此，在建立**冗余索引**来支持覆盖索引时就需要权衡考虑了。

> 使用**前缀索引就用不上覆盖索引**对查询性能的优化了，前缀索引中的数据并不完整，需要回表查询完整的数据。这也是你在选择是否使用前缀索引时需要考虑的一个因素。

# 二、索引条件下推

**Index Condition Pushdown(ICP)** 是 MySQL 5.6 中新特性，是一种在**存储引擎层使用索引过滤数据**的一种优化方式。ICP 可以**减少存储引擎访问基表(回表)的次数以及 MySQL 服务器访问存储引擎的次数**。

> 类似覆盖索引，**索引条件下推**使得类似于 % 开头的模糊查询**索引不失效**。索引中包含这个字段，但是没有使用到这个字段的索引(比如‘%a%’)，却可以使用这个字段在索引中进行条件过滤，从而**减少回表的记录条数**，这就是索引条件下推带来的性能优化。

ICP 的开启、关闭（默认开启）

默认情况下启用索引条件下推。可以通过设置系统变量 optimizer_switch 控制 index_condition_pushdown

```mysql
#关闭索引下推
SET optimizer_switch='index_condition_pushdown=off';

#打开索引下推
SET optimizer_switch='index_condition_pushdown=on';
```

当使用索引条件下推时，**EXPLAIN** 语句输出结果中 **Extra** 列内容显示为 **Using index condition**

**ICP 的使用条件**

1. 只能用于二级索引（secondary index）
2. explain 显示的执行计划中 type 值（join 类型）为 range 、 ref 、 eq_ref 或者 ref_or_null 。
3. 并非全部 where 条件都可以用 ICP 筛选，如果 where 条件的字段不在索引列中，还是要读取整表的记录到 server 端做 where 过滤。
4. ICP 可以用于 MyISAM 和 InnnoDB 存储引擎
5. MySQL 5.6 版本的不支持分区表的 ICP 功能，5.7 版本的开始支持。
6. 当 SQL 使用覆盖索引时，不支持 ICP 优化方法。
   