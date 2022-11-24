> 参考来源：
>
> 康师傅：https://www.bilibili.com/video/BV1iq4y1u7vj?p=144
>
> 爱编程的大李子：https://blog.csdn.net/LXYDSF/article/details/126606855

# 一、数据准备

```mysql
# 创建Type表
CREATE TABLE IF NOT EXISTS `type` (
`id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
`card` INT(10) UNSIGNED NOT NULL,
PRIMARY KEY (`id`)
);

# 创建book表
CREATE TABLE IF NOT EXISTS `book` (
`bookid` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
`card` INT(10) UNSIGNED NOT NULL,
PRIMARY KEY (`bookid`)
);

# 在type表中执行20次如下数据，插入20条数据。
INSERT INTO `TYPE`(card) VALUES(FLOOR(1 + RAND() * 20));
INSERT INTO `TYPE`(card) VALUES(FLOOR(1 + RAND() * 20));
# ...

# 同样的，在book表中插入20条数据
INSERT INTO `book`(card) VALUES(FLOOR(1 + RAND() * 20));
INSERT INTO `book`(card) VALUES(FLOOR(1 + RAND() * 20));
# ...
```

# 二、外连接与内连接的比较

## 1. 采用左外连接

我们知道**多表查询**分为**外连接**和**内连接**，而**外连接**又分为**左外连接**，**右外连接**和满外连接。其中外连接中，**左外连接与右外连接可以通过交换表来相互改造**，其原理也是类似的，而满外连接无非是二者的一个综合，因此外连接我们**只介绍左外连接**的优化即可。

**1.1下面开始 EXPLAIN 分析，当没有使用索引时，可以看到是全表扫描~**

```mysql
EXPLAIN SELECT SQL_NO_CACHE * FROM `type` LEFT JOIN book ON type.card = book.card;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/86.png)

在上面的查询 sql 中，type表是驱动表，book表是被驱动表。在执行查询时，会先查找驱动表中符合条件的数据，再根据驱动表查询到的数据在被驱动表中根据匹配条件查找对应的数据。因此被驱动表嵌套查询的次数是 **20*20=400** 次。实际上，由于我们总是需要在被驱动表中进行查询，优化器帮我们已经做了优化，上面的查询结果中可以看到，使用了 **join buffer**，将数据缓存起来，提高检索的速度。

**1.2 为了提高外连接的性能，我们添加下索引**

```mysql
CREATE INDEX Y ON book(card); #【被驱动表】，可以避免全表扫描

EXPLAIN SELECT SQL_NO_CACHE * FROM `type` 
LEFT JOIN book ON type.card = book.card;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/87.png)

对于外层表来说，虽然其查询仍然是全表扫描，但是因为是左外连接，LEFT JOIN左边的表的数据无论是否满足条件都会保留，因此全表扫描也是不赖的。另外可以看到第二行的 type 变为了 ref，rows 也变成了1，优化比较明显。这是由左连接特性决定的。LEFT JOIN 条件用于确定如何从右表搜索行，左边一定都有，所以 **右边是我们的关键点，一定需要建立索引**

**1.3 我们当然也可以给type表建立索引。**

```mysql
CREATE INDEX X ON `type`(card); #【驱动表】，无法避免全表扫描
# ALTER TABLE `type` ADD INDEX X (card);

EXPLAIN SELECT SQL_NO_CACHE * FROM `type` LEFT JOIN book ON type.card = book.card;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/88.png)

> 注意，外连接的关联条件中，两个关联字段的类型、字符集一定要保持一致，否则索引会失效哦。

**1.4 删除索引Y，我们继续查询**

```mysql
# 删除索引
DROP INDEX Y ON book;

EXPLAIN SELECT SQL_NO_CACHE * FROM `type` LEFT JOIN book ON type.card = book.card;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/89.png)

book表使用`join buffer`，再次验证了**左外连接左边的表是驱动表**，**右边的表是被驱动表**，**后面我们将与内连接在这一点进行对比**。

> 左外连接左表是驱动表右表是被驱动表，**右外连接**和此**相反**，**内连接**则是按照**数据量**的大小，数据量**少的是驱动表**，**多的是被驱动表**

## 2. 采用内连接

**2.1 删除现有的索引，换成 inner join(MySQL自动选择驱动表)**

```mysql
drop index X on type;
drop index Y on book;# (如果已经删除了可以不用再执行该操作)
EXPLAIN SELECT SQL_NO_CACHE * FROM type INNER JOIN book ON type.card=book.card;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/90.png)

**2.2 为book表添加索引优化**

```mysql
ALTER TABLE book ADD INDEX Y (card);

EXPLAIN  SELECT SQL_NO_CACHE * FROM  type INNER JOIN book ON type.card=book.card;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/91.png)

**2.3 向type表中再增加20条数据，为type表增加索引优化，观察情况**

```mysql
# 再向type表中插入20条数据，此时type:40条数据，book:20条数据 (过程省略)
ALTER TABLE type ADD INDEX X (card);

EXPLAIN SELECT SQL_NO_CACHE * FROM type INNER JOIN book ON type.card=book.card;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/92.png)

**2.4 接着，删除被驱动表的索引**

```mysql
DROP INDEX X ON `type`;

EXPLAIN SELECT SQL_NO_CACHE * FROM type INNER JOIN book ON type.card=book.card;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/93.png)

**被驱动表**进行了**反转**。这是因为**内连接优化器**可以**决定**（被）驱动表。在只有**一个表存在索引**的情况下，会选择**存在索引的表**作为**被驱动表**(因为被驱动表查询次数更多，建立索引以后可以**避免全表扫描**)

**2.5 给驱动表再加上索引，观察结果**

```mysql
ALTER TABLE `type` ADD INDEX X (card);

EXPLAIN SELECT SQL_NO_CACHE * FROM type INNER JOIN book ON type.card=book.card;
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/94.png)

被驱动表又进行了反转~

> 结论：
>
> **内连接**中会优先使用**有索引**的表作为**右表(被驱动表)**，否则会选择**大表作为右表(被驱动表)**，也就是**小表驱动大表**

# 三、JOIN 语句原理

join 方式连接多表，本质就是各个表之间数据的循环匹配。MySQL 5.5 版本之前，MySQL 只支持一种表间关联方式，就是嵌套循环。如果关联表的数据量很大，则 join 关联的执行时间会非常漫长。在 MySQL 5.5 以后的版本中，MySQL 通过引入 BNLJ 算法来优化嵌套执行。

## 1. 驱动表和被驱动表

**驱动表**就是**主表**，**被驱动表**就是**从表**、非驱动表。

- **对于内连接来说:**

```mysql
SELECT * FROM A JOIN B ON ...
```

A 并不一定就是驱动表，**优化器**会根据你的查询语句做**优化**，决定先查哪张表。先查询的哪张表就是驱动表，反之就是被驱动表。通过 explain 关键字可以查看。

- **对于外连接来说：**

```mysql
SELECT * FROM A LEFT JOIN B ON ...
# 或
SELECT * FROM B RIGHT JOIN A ON ...
```

通常，大家会认为 A 就是驱动表，B 就是被驱动表。但也未必。测试如下：

```mysql
CREATE TABLE a(f1 INT,f2 INT,INDEX(f1)) ENGINE=INNODB;
CREATE TABLE b(f1 INT,f2 INT) ENGINE=INNODB;

INSERT INTO a values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6);
INSERT INTO b values(3,3),(4,4),(5,5),(6,6),(7,7),(8,8);
# 测试1
EXPLAIN SELECT * FROM a LEFT JOIN b ON(a.f1=b.f1) WHERE (a.f2=b.f2);
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/95.png)

明明我们写的是 **a LEFT JOIN b**，但是我们执行 sql 查询时，却是**b作为了驱动表**，**a作为了被驱动表**。

实际上，**查询优化器会帮你把外连接改造为内连接，然后根据其优化策略选择驱动表与被驱动表**

## 2. Simple Nested-Loop Join（简单嵌套循环连接）

算法相当简单，从表 A 取出一条数据 1，遍历表 B，将匹配到的数据放到 result。以此类推，**驱动表 A 中的每一条记录与被动驱动表 B 的记录进行判断**：

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/96.png)

可以看到这种方式效率是非常低的，以上述表 A 数据 100 条，表 B 数据 1000 条，则 A*B=10 万次。开销统计如下：

| 开销统计         | SNLJ  |
| ---------------- | ----- |
| 外表扫描次数     | 1     |
| 内表扫描次数     | A     |
| 读取记录数       | A+B*A |
| JOIN 比较次数    | B*A   |
| 回表读取记录次数 | 0     |

当然 MySQL 肯定不会这么**粗暴**的进行表的连接，所以就出现了后面的两种其的优化算法。

另外，从**读取记录**数来看：A+B*A中，驱动表A**对性能的影响权重**更大。因此我们优化器会选择**小表驱动大表**。

## 3. Index Nested-Loop Join（索引嵌套循环连接）

Index Nested-Loop Join 其优化的思路主要是为了 **减少内层表数据的匹配次数**，所以要求**被驱动表**上必须 **有索引** 才行。通过外层表匹配条件直接与内层索引进行匹配，**避免**和内层表的**每条记录进行比较**，这样极**大地减少**了对内层表的匹配次数。

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/97.png)

驱动表中的每条记录通过被驱动表的**索引**进行访问，因为索引查询的成本是比较固定的，故 MySQL 优化器都倾向于使用有索引的表作为被驱动表（减少被驱动表的多次全表扫描）。

| 开销统计         | SNLJ  | INLJ                      |
| ---------------- | ----- | ------------------------- |
| 外表扫描次数     | 1     | 1                         |
| 内表扫描次数     | A     | 0                         |
| 读取记录数       | A+B*A | A+B（match）              |
| JOIN 比较次数    | B*A   | A*Index（Height）         |
| 回表读取记录次数 | 0     | B（match）（if possible） |

如果被驱动表加索引，效率是非常高的，如果索引**不是主键索引**，所以还得进行一次**回表**查询。相比，被驱动表的索引**是主键索引**，**效率会更高**

## 4. Block Nested-Loop Join（块嵌套循环连接）

如果存在索引，那么会使用 index 的方式进行 join，如果 join 的列没有索引，被驱动表要扫描的次数太多了。每次访问被驱动表，其表中的记录都会被**加载到内存**中，然后再从驱动表中取一条与其匹配，匹配结束后**清除内存**，然后再**从驱动表中加载一条记录**，然后把驱动表的记录**再加载到内存**匹配，这样**周而复始**，大大增加了 IO 次数。为了减少被驱动表的 IO 次数，就出现了 Block Nested-Loop Join 的方式

不再是**逐条**获取驱动表的数据，而是**一块一块**的获取，引入了 join buffer 缓冲区，将驱动表 join 相关的部分数据列（大小受 join buffer 的限制）**缓存到 join buffer 中**，然后**全表扫描**被驱动表，被驱动表的每一条记录一次性和 join buffer 中的所有驱动表记录进行匹配（内存中操作），将**简单嵌套循环**中的`多次比较`**合并**成`一次`，**降低了被动表的访问频率**。


```mysql
注意：
这里缓存的不只是关联表的列，select 后面的列也会缓存起来
在一个有 N 个 join 关联的 SQL 中会分配 N-1 个 join buffer。所以查询的时候尽量减少不必要的字段，可以 让 join buffer 中存放更多的列。
```

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/98.png)

| 开销统计          | SNLJ  | INLJ                   | BNLJ                                          |
| ----------------- | ----- | ---------------------- | --------------------------------------------- |
| 外表扫描次数:     | 1     | 1                      | 1                                             |
| 内表扫描次数:     | A     | 0                      | A * used_column_size / join_buffer_size+1     |
| 读取记录数:       | A+B*A | A+B(match)             | A+B* (A* used_column_size / join_buffer_size) |
| JOIN比较次数:     | B*A   | A* Index(Height)       | B * A                                         |
| 回表读取记录次数: | 0     | B(match) (if possible) | 0                                             |

参数设置：

- block_nested_loop

  通过 `show variables like '%optimizer_switch%'` 查看 `block_nested_loop` 状态。默认是开启的。

- join_buffer_size

  驱动表能不能一次加载完，要看 join buffer 能不能存储所有的数据，默认情况下 `join_buffer_size = 256K`。

  join *buffer* size 的最大值在 32 位系统可以申请 4G，而在 64 位操做系统下可以申请大于 4G 的 join_buffer空间（64 位 Windows 除外，其大值会被截断为 4GB并发出警告）。

**小结：**

1. 保证被驱动表的 JOIN 字段已经创建了索引（减少内层表的循环匹配次数）
2. 需要 JOIN 的字段，数据类型保持绝对一致。
3. LEFT JOIN 时，选择小表作为驱动表， 大表作为被驱动表 。减少外层循环的次数。
4. INNER JOIN 时，MySQL 会自动将小结果集的表选为驱动表 。选择相信 MySQL 优化策略。
5. 能够直接多表关联的尽量直接关联，不用子查询。(减少查询的趟数)
6. 不建议使用多层嵌套子查询，建议将子查询 SQL 拆开结合程序多次查询，或使用 JOIN 来代替子查询。
7. 衍生表建不了索引
8. 默认效率比较：INLJ > BNLJ > SNLJ
9. 正确理解小表驱动大表：大小不是指表中的记录数，而是永远用小结果集驱动大结果集（其本质就是减少外层循环的数据数量）。 比如A表有100条记录，B表有1000条记录，但是where条件过滤后，B表结果集只留下50个记录，A表结果集有80条记录，此时就可能是B表驱动A表。其实上面的例子还是不够准确，因为结果集的大小也不能粗略的用结果集的行数表示，而是表行数 * 每行大小。其实要理解你只需要结合Join Buffer就好了，因为表行数 * 每行大小越小，其占用内存越小,就可以在Join Buffer中尽量少的次数加载完了。

# 四、Hash Join

从 MySQL 8.0.20 版本开始将废弃 BNLJ，因为加入了 hash join 默认都会使用 hash join

- Nested Loop：

  对于被连接的数据子集较小的情况，Nested Loop 是个较好的选择。

- Hash Join 是做 **大数据集连接** 时的常用方法，优化器使用两个表中较小（相对较小）的表利用 join key 在内存中建立 **散列表**，然后扫描较大的表并探测散列表，找出与 Hash 表匹配的行。

  - 这种方式适用于较小的表完全可以放于内存中的情况，这样总成本就是访问两个表的成本之和

  - 在表很大的情况下并不能完全放入内存，这时优化器会将它分割成 若干不同的分区，不能放入内存的部分就把该分区写入磁盘的临时段，此时要求有较大的临时段从而尽量提高 I/O 的性能。

  - 它能够很好的工作于没有索引的大表和并行查询的环境中，并提供最好的性能。大多数人都说它是 Join 的重型升降机。Hash Join 只能应用于等值连接（如 WHERE A.COL1 = B.COL2），这是由 Hash 的特点决定的。

| 类型     | Nested Loop                                                  | Hash Join                                                    |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 使用条件 | 任何条件                                                     | 等值连接（=）                                                |
| 相关资源 | CPU、磁盘 I/O                                                | 内存、临时空间                                               |
| 特点     | 当有高选择性索引或进行限制性搜索时效率比较高，能够快速返回第一次的搜索结果 | 当缺乏索引或者索引条件模糊时，Hash Join 比 Nested Loop 有效。在数据仓库环境下，如果表的记录数多，效率高 |
| 缺点     | 当索引丢失或者查询条件限制不够时，效率很低；当表的记录数较多，效率低 | 为遍历哈希表，需要大量内存。第一次的结果返回较慢             |

