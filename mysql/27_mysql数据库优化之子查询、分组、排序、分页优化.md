# 一、子查询优化

MySQL 从 4.1 版本开始支持子查询，使用子查询可以进行 SELECT 语句的嵌套查询，即一个 SELECT 查询的结果作为另一个 SELECT 语句的条件。子查询可以一次性完成很多逻辑上需要多个步骤才能完成的操作 。

子查询是 MySQL 的一项重要的功能，可以帮助我们通过一个 SQL 语句**实现比较复杂的查询**。但是，子查询的**执行效率不高**。 通常我们可以将其**优化**成一个**连接查询**

**原因：**

- 执行子查询时，MySQL 需要为内层查询语句的查询结果**建立**一个**临时表** ，然后外层查询语句从临时表中查询记录。查询完毕后，再**撤销**这些临时表 。这样会**消耗**过多的 CPU 和 IO **资源**，**产生**大量的**慢查询**。

- 子查询的结果集存储的**临时表**，不论是内存临时表还是磁盘临时表都 **不会存在索引** ，所以查询性能会受到一定的影响。

- 对于返回**结果集比较大**的子查询，其对查询**性能的影响**也就越大。

在 MySQL 中，可以使用**连接**（JOIN）查询来**替代子查询**。 连接查询 不需要建立临时表，其 速度比子查询要快，如果查询中**使用索引**的话，性能就会更好。

举例1：查询学生表中是班长的学生信息

- 使用子查询

```mysql
#创建班级表中班长的索引
CREATE INDEX idx_monitor ON class(monitor);

#查询班长的信息
EXPLAIN SELECT * FROM student stu1
    WHERE stu1.`stuno` IN (
    SELECT monitor
    FROM class c
    WHERE monitor IS NOT NULL
);
```

- 推荐：使用多表查询

```mysql
EXPLAIN SELECT stu1.* FROM student stu1 JOIN class c 
ON stu1.`stuno` = c.`monitor`
WHERE c.`monitor` IS NOT NULL;
```

举例2：取所有不为班长的同学

- 不推荐

```mysql
#查询不为班长的学生信息
EXPLAIN SELECT SQL_NO_CACHE a.* 
FROM student a 
WHERE  a.stuno  NOT  IN (
            SELECT monitor FROM class b 
            WHERE monitor IS NOT NULL);
```

- 推荐

```mysql
# 转换成左连接查询
EXPLAIN SELECT SQL_NO_CACHE a.*
FROM  student a LEFT OUTER JOIN class b 
ON a.stuno =b.monitor
WHERE b.monitor IS NULL;
```

> 尽量**不要使用 NOT IN** 或者 **NOT EXISTS**，用 **LEFT JOIN xxx ON xx WHERE xx IS NULL** 替代

# 二、ORDER BY 排序优化

**问题：在 WHERE 条件字段上加索引，但是为什么在 ORDER BY 字段上还要加索引呢？**

在 MySQL 中，支持两种**排序方式**，分别是 **FileSort** 和 **Index** 排序。

- Index 排序中，**索引**可以保证数据的**有序性**，就不需要再进行排序，效率更高。
- FileSort 排序则一般在 **内存中** 进行排序，**占用 CPU 较多**。如果待排序的**结果较大**，会产生**临时文件** I/O 到磁盘进行排序的情况，效率低。

**优化建议：**

1. SQL 中，可以在 WHERE 子句和 ORDER BY 子句中使用索引，目的是在 WHERE 子句中 **避免全表扫描**，在 ORDER BY 子句 **避免使用 FileSort 排序**。当然，某些情况下全表扫描，或者 FileSort 排序不一定比索引慢。但总的来说，我们还是要避免，以提高查询效率。
2. 尽量使用 Index 完成 ORDER BY 排序。如果 **WHERE** 和 **ORDER BY** 后面是相同的列就使用单索引列；如果**不同**就使用**联合索引**。
3. 无法使用 Index 时，需要对 FileSort 方式进行调优。

所有的排序都是在**条件过滤之后**才执行的。所以，如果**条件过滤大部分数据**的话，剩下几百几千条数据进行排序其实并不是很消耗性能，即使**索引**优化了排序，但实际**提升性能很有限**。

## 1. filesort 算法

排序的字段若不在索引列上，则 filesort 会有两种算法：**双路排序** 和 **单路排序**

- 双路排序（慢）

  **MySQL4.1 之前是使用双路排序**，字面意思就是两次扫描磁盘，最终得到数据， 读取行指针和 **order by 列**，对他们进行排序，然后扫描已经排序好的列表，按照列表中的值重新从列表中读取对应的数据输出

  从磁盘取排序字段，在 buffer 进行排序，再从 **磁盘取其他字段** 。

取一批数据，要对磁盘进行两次扫描，众所周知，IO 是很耗时的，所以在 MySQL4.1 之后，出现了第二种改进的算法，就是单路排序。

- 单路排序（快）

从磁盘读取查询需要的 **所有列** ，按照 order by 列在 buffer 对它们进行排序，然后扫描排序后的列表进行输出， 它的效率更快一些，避免了第二次读取数据。并且把随机 IO 变成了顺序 IO，但是它会使用更多的空间， 因为它把每一行都保存在内存中了。

**单路排序的问题**

- 在 sort_buffer 中，单路比多路要多占用很多空间，因为单路是把所有字段都取出，所以可能取出的数据的总大小超出了sort_buffer的容量，导致每次只能取 sort_buffer 容量大小的数据，进行排序（创建 temp 文件，多路合并），排完再取 sort_buffer 容量大小，再排…从而多次I/O。

- 单路本来想省一次 I/O 操作，反而导致了大量的 I/O 操作，反而得不偿失。
  

## 2. 优化策略

- **尝试提高 sort_buffer_size**

  不管用哪种算法，提高这个参数都会提高效率，要根据系统的能力去提高，因为这个参数是针对每个进程（connection）的 1M - 8M 之间调整。MySQL5.7，InnoDB 存储引擎默认值都是 1048576 字节，1MB。

  ```mysql
  SHOW VARIABLES LIKE '%sort_buffer_size%';
  ```

- **尝试提高 max_length_for_sort_data**

  提高这个参数，会增加改进算法的概率。

  ```mysql
  SHOW VARIABLES LIKE'%max_length_for_sort_data%';
  ```

  但是如果设的太高，数据总容量超出 sort_buffer_size 的概率就增大，明显症状是高的磁盘 I/O 活动和低的处理器使用率。如果需要返回的列的总长度大于 max_length_for_sort_data，使用双路算法，否则使用单路算法。1024-8192字节之间调整。

- Order by 时 **select * 是一个大忌**。最好只Query需要的字段。

  当 Query 的字段大小综合小于 max_length_for_sort_data，而且排序字段不是 TEXT|BLOG 类型时，会改进后的算法——单路排序，否则用老算法——多路排序。

  两种算法的数据都有可能超出 sort_buffer_size 的容量，超出之后，会创建 tmp 文件进行合并排序，导致多次 I/O，但是用单路排序算法的风险会更大一些，所以要提高 sort_buffer_size

# 三、GROUP BY 分组优化

- group by 使用索引的原则几乎跟 order by 一致 ，group by 即使没有过滤条件用到索引，也可以直接使用索引。
- group by 先排序再分组，遵照索引建的最佳左前缀法则
- 当无法使用索引列，增大 **max_length_for_sort_data** 和 **sort_buffer_size** 参数的设置
- where 效率高于 having，能写在 where 限定的条件就不要写在 having 中了
- 减少使用 order by，和业务沟通能不排序就不排序，或将排序放到程序端去做。Order by、group by、distinct 这些语句较为耗费 CPU，数据库的 CPU 资源是极其宝贵的。
- 包含了 order by、group by、distinct 这些查询的语句，where 条件过滤出来的结果集请保持在 1000 行以内，否则 SQL 会很慢。

# 四、分页查询优化

一般分页查询时，通过创建覆盖索引能够比较好地提高性能。一个常见有非常头疼的问题就是 **limit 2000000,10**，此时需要 MySQL 排序前 2000010 记录，仅仅返回 2000000-2000010 的记录，其他记录丢弃，查询排序的代价非常大。

- **优化思路一**

在索引上完成排序分页操作，最后根据主键关联回原表查询所需要的其他列内容。

```mysql
EXPLAIN SELECT * FROM student t,(SELECT id FROM student ORDER BY id LIMIT 2000000,10) a
WHERE t.id = a.id;
```

- **优化思路二**

该方案适用于主键自增的表，可以把 Limit 查询转换成某个位置的查询 。

```mysql
EXPLAIN SELECT * FROM student WHERE id > 2000000 LIMIT 10;
```

