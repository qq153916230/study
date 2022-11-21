> 参考来源：
>
> 康师傅：https://www.bilibili.com/video/BV1iq4y1u7vj?p=134
>
> 爱编程的大李子：https://blog.csdn.net/LXYDSF/article/details/126338994

# 一、数据库服务器的优化步骤

在数据库调优中，我们的目标就是 **响应时间更快，吞吐量更大**。利用宏观的监控工具和微观的日志分析可以帮我们快速找到调优的思路和方式

整个流程划分成了 **观察（Show status）** 和 **行动（Action）** 两个部分。字母 S 的部分代表观察（会使用相应的分析工具），字母 A 代表的部分是行动（对应分析可以采取的行动）

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/59.png)

我们可以通过观察了解数据库整体的运行状态,通过性能分析工具可以让我们了解执行慢的SQL都有哪些，查看具体的SQL执行计划，甚至是SQL执行中的每一步的成本代价， 这样才能定位问题所在，找到了问题，再采取相应的行动。

如果我们发现执行SQL时存在不规则延迟或卡顿的时候，就可以采用分析工具帮我们定位有问题的SQL，这三种分析工具你可以理解是SQL调优的三个步骤：**慢查询**、**EXPLAIN** 和**SHOW**
**PROFILING**。

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/60.png)

可以看到数据库调优的步骤中越往金字塔尖走，其成本越高，效果越差，因此我们在数据库调优的过程中，要重点把握金字塔底部的 sql 及索引调优，数据库表结构调优，系统配置参数调优等软件层面的调优

## 1. 查看系统性能参数

可以使用 **SHOW STATUS** 语句查询一些数据库服务器的**性能参数和使用频率**。其语法如下：

```mysql
SHOW [GLOBAL][SESSION] STATUES LIKE '参数';
```

一些常用的性能参数如下：

- **Connections**：MySQL服务器的连接次数。
- **Uptime**：MySQL服务器工作时间。
- **Slow_queries**：慢查询的次数。慢查询次数参数可以结合慢查询日志找出慢查询语句，然后针对慢查询语句进行 **表结构优化** 或者**查询语句优化**
- **Innodb_rows_read**：Select查询返回的行数
- **Innodb_rows_inserted**：执行INSERT操作插入的行数
- **Innodb_rows_updated**：执行UPDATE操作更新的行数
- **Innodb_rows_deleted**：执行DELETE操作删除的行数
- **Com_select**：查询操作的次数。
- **Com_insert**：插入操作的次数。对于批量插入的 INSERT 操作，只累加一次。
- **Com_update**：更新操作的次数。
- **Com_delete**：删除操作的次数。

## 2. 统计 SQL 的查询成本：last_query_cost

一条 SQL查询语句在执行前需要确定查询执行计划，如果存在多种执行计划的话，MySQL 会计算每个执行计划所需要的成本，从中选择**成本最小**的一个作为最终执行的执行计划。

如果我们想要查看某条SQL语句的查询成本，可以在执行完这条SQL语句之后，通过查看当前会话中的 **last_query_cost** 变量值来得到当前查询的成本。它通常也是我们**评价一个查询的执行效率**的一个常用指标。这个查询成本对应的是**SQL语句所需要读取的页的数量**。

查询 last_query_cost 对于比较开销是非常有用的，特别是我们有好几种查询方式可选的时候

> SQL查询是一个动态的过程，从页加载的角度，我们可以得到以下两点结论：
>
> 1. **位置决定效率**：如果页就在数据库**缓冲池**中，那么效率是最高的，否则还需要从**内存**或者**磁盘**中进行读取，当然针对单个页的读取来说，如果页存在于内存中，会比在磁盘中读取效率高很多。即 **数据库缓冲池>内存>磁盘**
> 2. **批量决定效率**：如果我们从磁盘中单一页进行随机读，那么效率是很低的（差不多10ms），而采用顺序读取的方式，批量对页进行读取，平均一页的读取效率就会提升很多，甚至要快于单个页面在内存中的随机读取。**即顺序读取>大于随机读取**
>
> 所以说，遇到 I/O 并不用担心，方法找对了，效率还是很高的。我们首先要考虑数据存放的位置，如果是经常使用的数据就要尽量放到缓冲池中，其次我们可以充分利用磁盘的吞吐能力，一次性批量读取数据，这样单个页的读取效率也就得到了提升。
>
> 注：缓冲池和查询缓存并不是一个东西

## 3. 定位执行慢的 SQL：慢查询日志

MySQL的慢查询日志，用来记录在MySQL中**响应时间超过阀值**的语句，具体指运行时间超过**long_query_time**值的SQL，则会被记录到慢查询日志中。long_query_time的默认值为10，意思是运行10秒以上(不含10秒)的语句，认为是超出了我们的最大忍耐时间值。

它的主要作用是，帮助我们发现那些执行时间特别长的SQL查询，并且有针对性地进行优化,从而提高系统的整体效率。当我们的数据库服务器发生阻塞、运行变慢的时候，检查一下慢查询日志，找到那些慢查询，对解决问题很有帮助。比如一条sq|执行超过5秒钟， 我们就算慢SQL，希望能收集超过5秒的sql，结合explain进行全面分析。

默认情况下，MySQL数据库没有开启慢查询日志，需要我们手动来设置这个参数。如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。

慢查询日志支持将日志记录写入文件。

## 4. 开启慢查询日志

开启 **slow_query_log**

```mysql
# 查看慢查询日志是否开启，以及日志的位置
show variables like '%slow_query_log%';

# 修改慢查询日志状态为开启，注意这里要加 global，因为它是全局系统变量，否则会报错。
set global slow_query_log='ON';
```

修改 **long_query_time** 阈值

```mysql
# 查看慢查询的时间阈值
show variables like '%long_query_time%';

# 设置慢查询的时间阈值
# 设置global的方式对当前session的long_query_time失效。对新连接的客户端有效，所以可以一并执行下列语句
set global long_query_time = 1;
set long_query_time = 1;
```

上面的设置是临时的，如果想要永久设置，则需按照下面的步骤：

修改my.cnf文件，[mysqld]下增加或修改参数long_query_time 、slow_query_log和
slow_ query_log_file后，然后重启MySQL服务器。

```
[mysqld]
slow_query_log=ON # 开启慢查询日志的开关
slow_query_log_file=/var/lib/mysql/xxx-slow.log # 慢查询日志的目录和文件名信息
long_query_time=3 # 设置慢查询的阀值为3秒，超出此设定值的SQL即被记录到慢查询日志
log_output=FILE
```

> 补充说明：
>
> 在Mysql中，除了上述变量，控制慢查询日志的还有另外一个变量**min_examined_row_limit** 。这个变量的意思是，查询**扫描过的最少记录数**。这个变量和查询执行时间，共同组成了判别一个查询是否慢查询的条件。如果查询扫描过的记录数大于等于这个变量的值，并且查询执行时间超过 **long_query_time** 的值，那么这个查询就被记录到慢查询日志中。反之，则不被记录到慢查询日志中。另外，**min_examined_row_limit** 默认是 0，我们也一般不会去修改它。
>
> 当这个值为默认值0时，与 long_query_time=10合在一起，表示只要查询的执行时间超过10秒钟，哪怕一个记录也没有扫描过，都要被记录到慢查询日志中。你也可以根据需要，通过修改"my.ini"文件，来修改查询时长，或者通过SET指令，用SQL语句修改**min_examined_row_limit** 的值。

## 5. 慢查询日志分析工具：Mysqldumpslow

在生产环境中，如果要手工分析日志，查找、分析 SQL，显然是个体力活，MySQL 提供了日志分析工具 **mysqldumpslow**。

> 注意:
> 1.该工具并不是 MySQL 内置的，不要在 MySQL 下执行，可以直接在根目录或者其他位置执行
> 2.该工具只有 Linux 下才是开箱可用的，实际上生产中mysql数据库一般也是部署在linux环境中的。如果是windows环境下，可以参考博客：
> https://www.cnblogs.com/-mrl/p/15770811.html

```mysql
# 查询慢查询日志所在目录
SHOW VARIABLES LIKE '%slow%';   
```

通过 `mysqldumpslow --help` 可以查看慢查询日志命令帮助

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/61.png)

- -a: 不将数字抽象成N，字符串抽象成S
- -s: 是表示按照何种方式排序：
  - c: 访问次数
  - l: 锁定时间
  - r: 返回记录
  - t: 查询时间
  - al:平均锁定时间
  - ar:平均返回记录数
  - at:平均查询时间 （默认方式）
  - ac:平均查询次数
- -t: 即为返回前面多少条的数据；
- -g: 后边搭配一个正则匹配模式，大小写不敏感的；

接下来我们可以找到慢查询日志的位置

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/62.png)

按照查询时间排序，查看前五条 SQL 语句：

```shell
mysqldumpslow -s t -t 5 /var/lib/mysql/hadoop102-slow.log 
```

结果为：

```
Reading mysql slow query log from /var/lib/mysql/hadoop102-slow.log
Count: 1  Time=283.29s (283s)  Lock=0.00s (0s)  Rows=0.0 (0), root[root]@hadoop102
  CALL insert_stu1(N,N)

Count: 1  Time=1.09s (1s)  Lock=0.00s (0s)  Rows=5.0 (5), root[root]@localhost
  SELECT * FROM student WHERE name = 'S'

Count: 1  Time=1.03s (1s)  Lock=0.00s (0s)  Rows=1.0 (1), root[root]@localhost
  SELECT * FROM student WHERE stuno = N

Died at /usr/bin/mysqldumpslow line 162, <> chunk 3.
```

可以看到上面 sql 中具体的数值类都被N代替，字符串都被使用 S 代替，如果想要显示真实的数据，可以加上参数 **-a**

```shell
mysqldumpslow -a -s t -t 5 /var/lib/mysql/hadoop102-slow.log 
```

结果为：

```
Reading mysql slow query log from /var/lib/mysql/hadoop102-slow.log
Count: 1  Time=283.29s (283s)  Lock=0.00s (0s)  Rows=0.0 (0), root[root]@hadoop102
  CALL insert_stu1(100001,4000000)

Count: 1  Time=1.09s (1s)  Lock=0.00s (0s)  Rows=5.0 (5), root[root]@localhost
  SELECT * FROM student WHERE name = 'ZfCwDz'

Count: 1  Time=1.03s (1s)  Lock=0.00s (0s)  Rows=1.0 (1), root[root]@localhost
  SELECT * FROM student WHERE stuno = 3455655

Died at /usr/bin/mysqldumpslow line 162, <> chunk 3.
```

工作中常用的一些查询：

```shell
#得到返回记录集最多的10个SQL
mysqldumpslow -s r -t 10 /var/lib/mysql/atguigu-slow.log

#得到访问次数最多的10个SQL
mysqldumpslow -s c -t 10 /var/lib/mysql/atguigu-slow.log

#得到按照时间排序的前10条里面含有左连接的查询语句
mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/atguigu-slow.log

#另外建议在使用这些命令时结合 | 和more 使用 ，否则有可能出现爆屏情况
mysqldumpslow -s r -t 10 /var/lib/mysql/atguigu-slow.log | more
```

## 6. 关闭慢查询日志

**永久性关闭：**

修改my.cnf或my.ini文件，把【mysqld】组下的slow_query_log值设置为OFF，修改保存后，再重启MySQL服务，即可生效。

```
#配置文件
[mysqld]
slow_query_log=OFF
```

或者，把slow_query_log一项注释掉 或 删除

```
[mysqld]
#slow_query_log =OFF
```

**临时性方式**

```mysql
# 关闭慢查询日志
SET GLOBAL slow_query_log=off;
```

## 7. 删除与恢复慢查询日志

```mysql
# 显示慢查询日志信息
SHOW VARIABLES LIKE '%slow_query_log%';
```

调优结束可以及时删除慢查询日志节省磁盘空间

```shell
rm xxx-slow.log
```

如果误删了，而且还没有了备份，可以使用下面的命令来重新恢复生成哟，执行完毕后会在数据目录下重新生成查询日志文件

```mysql
# 在mysql中打开慢查询日志
SET GLOBAL slow_query_log=ON;
```

```shell
# linux下恢复慢查询日志
mysqladmin -uroot -p flush-logs slow
```

> 提示
>
> 慢查询日志都是使用`mysqladmin -uroot -p flush-logs slow` 命令来删除重建的。使用时一定要注意，一旦执行了这个命令，慢查询日志都只存在于新的日志文件中，如果需要旧的查询日志，就必须事先备份。