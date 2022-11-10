[康师傅]: https://www.bilibili.com/video/BV1iq4y1u7vj?p=111
[爱编程的大李子]: https://blog.csdn.net/LXYDSF/article/details/125755327

# 一、SQL执行流程图

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/47.png)

- **查询缓存**

  Server 如果在查询缓存中发现了这条 SQL 语句，就会直接将结果返回给客户端;如果没 有，就进入到解析器阶段。需要说明的是，因为查询缓存往往效率不高，所以在 MySQL 8.0 之后就抛弃 了这个功能。

- **解析器**

  在解析器中对 SQL 语句进行语法分析、语义分析。语句不对就会收到错误提醒，语句正确，则会生成一个语法树。

- **优化器**

  可以分为 `逻辑查询` 优化阶段和 `物理查询` 优化阶段

  经过了解析器，MySQL 就知道你要做什么了。在开始执行之前，还要先经过优化器的处理，比如是根据 **全表检索** ，还是根据 **索引检索** 等。**一条查询可以有很多种执行方式，最后都返回相同的结果。优化器的作用就是找到这其中最好的执行计划**。

  截止到现在，还没有真正去读写真实的表，仅仅只是产出了一个执行计划。于是就进入了 `执行器阶段` 。

- **执行器**

  在执行之前需要判断该用户是否具备权限 。如果没有，就会返回权限错误。

  如果有权限，就打开表继续执行SQL并返回结果。在 MySQL 8.0 以前的版本，如果设置了查询缓存，这时会将查询结果进行缓存。打开表的时候，执行器就会根据表的引擎定义，调用存储引擎API对表进行的读写。存储引擎API只是抽象接口，下面还有个**存储引擎层**，具体实现还是要看表选择的存储引擎。

SQL 语句在 MySQL 中的流程是：`SQL 语句 → 查询缓存 → 解析器 → 优化器 → 执行器`。

# 二、查看SQL 执行所使用的资源

## 1. 确认profiling是否开启

```mysql
SELECT @@profiling;
# 或
SHOW VARIABLES LIKE 'profiling';
# Profiling功能由MySQL会话变量：profiling控制。默认是OFF（关闭状态）。

# profiling = 0代表关闭，我们需要把profiling打开，即设置为1；
SET profiling = 1;
```

## 2. 查看profiles

```mysql
#查询所有sql语句的分析概览
SHOW PROFILES;

# 展示最新的sql执行过程
SHOW PROFILE;

# 根据 Query_ID 查看某一次sql执行的分析过程
SHOW PROFILE FOR QUERY Query_ID;
```

## 3. PROFILE的详细语法：

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/48.png)