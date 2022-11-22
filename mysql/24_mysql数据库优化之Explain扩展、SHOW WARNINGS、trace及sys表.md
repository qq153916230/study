> 参考来源：
>
> 康师傅：https://www.bilibili.com/video/BV1iq4y1u7vj?p=139
>
> 爱编程的大李子：https://blog.csdn.net/LXYDSF/article/details/126338994

# 一、EXPLAIN 四种输出格式

EXPLAIN可以输出四种格式： **传统格式** ，**JSON格式**， **TREE格式** 以及 **可视化输出**。用户可以根据需要选择适用于自己的格式。

## 1. 传统格式

**传统格式**简单明了，输出是一个表格形式，概要说明查询计划。

```mysql
# EXPLAIN + sql语句
EXPLAIN SELECT * FROM table;
```

## 2. JSON 格式

第1种格式中介绍的 **EXPLAIN** 语句输出中缺少了一个衡量执行计划好坏的重要属性–成本。 而JSON格式是四种格式里面输出**信息最详尽**的格式，里面包含了执行的成本信息。

传统格式与json格式的各个字段存在如下表所示的对应关系(mysql5.7官方文档)。

| Column        | JSON Name     | Meaning                                        |
| ------------- | ------------- | ---------------------------------------------- |
| id            | select_id     | The SELECT identifier                          |
| select_ type  | None          | The SELECT type                                |
| table         | table_name    | The table for the output row                   |
| partitions    | partitions    | The matching partitions                        |
| type          | access_type   | The join type                                  |
| possible_keys | possible_keys | The possible indexes to choose                 |
| key           | key           | The index actually chosen                      |
| key_len       | key_length    | The length of the chosen key                   |
| ref           | ref           | The columns compared to the index              |
| rows          | rows          | Estimate of rows to be examined                |
| filtered      | filtered      | Percentage of rows filtered by table condition |
| Extra         | None          | Additional information                         |

**JSON格式：在EXPLAIN单词和真正的查询语句中间加上 FORMAT=JSON**

```mysql
EXPLAIN FORMAT=JSON SELECT s1.key1, s2.key1 FROM s1 LEFT JOIN s2 ON s1.key1 = s2.key1 WHERE s2.common_field IS NOT NULL;
```

结果如下，可以看到 json 格式的信息量会更加丰富。尤其是**成本信息**，是用于衡量一个执行计划的好坏的重要指标

```mysql
mysql> EXPLAIN FORMAT=JSON SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key2 WHERE s1.common_field ='a'\G;
*************************** 1. row ***************************
EXPLAIN: {
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "1360.07"
    },
    "nested_loop": [
      {
        "table": {
          "table_name": "s1",
          "access_type": "ALL",
          "possible_keys": [
            "idx_key1"
          ],
          "rows_examined_per_scan": 9895,
          "rows_produced_per_join": 989,
          "filtered": "10.00",
          "cost_info": {
            "read_cost": "914.80",
            "eval_cost": "98.95",
            "prefix_cost": "1013.75",
            "data_read_per_join": "1M"
          },
          "used_columns": [
            "id",
            "key1",
            "key2",
            "key3",
            "key_part1",
            "key_part2",
            "key_part3",
            "common_field"
          ],
          "attached_condition": "((`atguigudb1`.`s1`.`common_field` = 'a') and (`atguigudb1`.`s1`.`key1` is not null))"
        }
      },
      {
        "table": {
          "table_name": "s2",
          "access_type": "eq_ref",
          "possible_keys": [
            "idx_key2"
          ],
          "key": "idx_key2",
          "used_key_parts": [
            "key2"
          ],
          "key_length": "5",
          "ref": [
            "atguigudb1.s1.key1"
          ],
          "rows_examined_per_scan": 1,
          "rows_produced_per_join": 989,
          "filtered": "100.00",
          "index_condition": "(cast(`atguigudb1`.`s1`.`key1` as double) = cast(`atguigudb1`.`s2`.`key2` as double))",
          "cost_info": {
            "read_cost": "247.38",
            "eval_cost": "98.95",
            "prefix_cost": "1360.08",
            "data_read_per_join": "1M"
          },
          "used_columns": [
            "id",
            "key1",
            "key2",
            "key3",
            "key_part1",
            "key_part2",
            "key_part3",
            "common_field"
          ]
        }
      }
    ]
  }
}
1 row in set, 2 warnings (0.00 sec)
```

s1 表的 "cost_info"部分：

```json
"cost_info": {
	"read_cost": "914.80",
    "eval_cost": "98.95",
    "prefix_cost": "1013.75",
    "data_read_per_join": "1M"
}
```

- **read_cost** 是由下边这两部分组成的：

  - **IO** 成本

  - 检测 **rows × (1 - filter)** 条记录的 **CPU** 成本

> **rows** 和 **filter** 都是我们前边介绍执行计划的输出列，在 **JSON 格式**的执行计划中，**rows** 相当于**rows_examined_per_scan**，**filtered** 名称不变

- **eval_cost** 是这样计算的：
  - 检测 **rows** × **filter** 条记录的成本。

- **prefix_cost** 就是单独查询 s1 表的成本，也就是：**read_cost** + **eval_cost**

- **data_read_per_join** 表示在此次查询中需要读取的数据量。

对于 s2 表的 “cost_info” 部分是这样的：

```json
"cost_info": {
	"read_cost": "247.38",
    "eval_cost": "98.95",
    "prefix_cost": "1360.08",
    "data_read_per_join": "1M"
}
```

由于 s2 表是被驱动表，所以可能被读取多次，这里的 **read_cost** 和 **eval_cost** 是访问多次 s2 表后累加起来的值，大家主要关注里边儿的 **prefix_cost** 的值代表的是整个连接查询预计的成本，也就是单次查询 s1 表和多次查询 s2 表后的成本的和，也就是：
```
247.38 + 98.95 + 1013.75 = 1360.08
```

## 3. TREE 格式

TREE 格式是 8.0.16 版本之后引入的新格式，主要根据查询的**各个部分之间的关系**和**各部分的执行顺序**来描述**如何查询**。

```mysql
mysql> EXPLAIN FORMAT=TREE SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key2 WHERE s1.common_field ='a'\G;
*************************** 1. row ***************************
EXPLAIN: -> Nested loop inner join  (cost=1360.08 rows=990)
    -> Filter: ((s1.common_field = 'a') and (s1.key1 is not null))  (cost=1013.75 rows=990)
        -> Table scan on s1  (cost=1013.75 rows=9895)
    -> Single-row index lookup on s2 using idx_key2 (key2=s1.key1), with index condition: (cast(s1.key1 as double) = cast(s2.key2 as double))  (cost=0.25 rows=1)

1 row in set, 1 warning (0.00 sec)
```

## 4. 可视化输出

可视化输出，可以通过 **MySQL Workbench** 可视化查看 MySQL 的执行计划。通过点击 Workbench 的**放大镜图标**，即可生成可视化的查询计划

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/83.png)

上图按**从左到右的连接顺序**显示表。红色框表示 **全表扫描** ，而绿色框表示**使用索引查找对于每个表显示使用的索引**。还要注意的是，每个表格的**框上方是每个表访问所发现的行数的估计值**以及**访问该表的成本**

# 二、SHOW WARNINGS 的使用

**可以显示数据库真正执行的 SQL** ，因为有时候 MySQL 执行引擎会对我们的 SQL 进行优化~

先使用 **Explain**，我们写的 sql 按道理是使用 **s1 作为驱动表**，**s2 作为被驱动表**

```mysql
EXPLAIN SELECT s1.key1, s2.key1 FROM s1 LEFT JOIN s2 ON s1.key1 = s2.key1 WHERE s2.common_field IS NOT NULL;
```

但是 **执行结果把 s2 作为了驱动表，s1 作为了被驱动表**

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/84.png)

紧接着使用 **SHOW WARNINGS** ，原来执行引擎将 **LEFT JOIN** 优化成了 **INNER JOIN**

```mysql
mysql> SHOW WARNINGS\G;
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `atguigudb1`.`s1`.`key1` AS `key1`,`atguigudb1`.`s2`.`key1` AS `key1` 
from `atguigudb1`.`s1` 
join `atguigudb1`.`s2` 
where ((`atguigudb1`.`s1`.`key1` = `atguigudb1`.`s2`.`key1`) 
and (`atguigudb1`.`s2`.`common_field` is not null))
1 row in set (0.00 sec)
```

上面 **message** 中显示的是**数据库优化、重写后真正执行的查询语句**。果然它帮我们做了优化

再举一个例子：下面是一个 **子查询 SQL**，**应该对应着两个不同的 id**

```mysql
EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key2 FROM s2 WHERE common_field = 'a');
```

但是真正执行后，**对应着竟然是相同的id**

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/85.png)

我们使用 **SHOW WARNINGS\G;** 进行分析，发现执行**引擎将其优化成了 多表连接查询的方式**

```mysql
mysql> SHOW WARNINGS\G;
*************************** 1. row ***************************
  Level: Warning
   Code: 1739
Message: Cannot use ref access on index 'idx_key1' due to type or collation conversion on field 'key1'
*************************** 2. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `atguigudb1`.`s1`.`id` AS `id`,`atguigudb1`.`s1`.`key1` AS `key1`,`atguigudb1`.`s1`.`key2` AS `key2`,`atguigudb1`.`s1`.`key3` AS `key3`,`atguigudb1`.`s1`.`key_part1` AS `key_part1`,`atguigudb1`.`s1`.`key_part2` AS `key_part2`,`atguigudb1`.`s1`.`key_part3` AS `key_part3`,`atguigudb1`.`s1`.`common_field` AS `common_field` 
from `atguigudb1`.`s2` 
join `atguigudb1`.`s1` 
where ((`atguigudb1`.`s2`.`common_field` = 'a') 
and (cast(`atguigudb1`.`s1`.`key1` as double) = cast(`atguigudb1`.`s2`.`key2` as double)))
2 rows in set (0.00 sec)
```

# 三、分析优化器执行计划：trace

**OPTIMIZE_TRACE** 是 mysql 5.6 中引入的一个**跟踪工具**，它可以**跟踪优化器做出的各种决策**，比如**访问表的方法**，各种开销计算，**各种转换**，结果会被记录到 **information_schema.optimizer_trace** 中。

此功能**默认关闭**。开启 trace，并设置格式为 **JSON**，同时**设置 trace 最大能够使用的内存大小**，避免解析过程中因为**默认内存过小而不能够完整展示**。命令如下：
```mysql
SET optimizer_trace="enabled=on",end_markers_in_json=on;

set optimizer_trace_max_mem_size=1000000;
```

开启后，可分析如下语句：

- SELECT
- INSERT
- REPLACE
- UPDATE
- DELETE
- EXPLAIN
- SET
- DECLARE
- CASE
- IF
- RETURN
- CALL

首先执行如下 SQL 语句

```mysql
select * from student where id < 10;
```

然后查询 **information_schema.optimizer_trace** 就可以知道 MySQL 是如何执行 SQL 的

```mysql
select * from information_schema.optimizer_trace\G;
```

结果如下：

```mysql
*************************** 1. row ***************************
 # 第1部分：查询语句
 QUERY: select * from student where id < 10
 # 第2部分：QUERY 字段对应语句的跟踪信息
 TRACE: {
 "steps": [
 {
   "join_preparation": {  # 预备工作
    "select#": 1,
    "steps": [
    {
      "expanded_query": "/* select#1 */ select `student`.`id` AS
`id`,`student`.`stuno` AS `stuno`,`student`.`name` AS `name`,`student`.`age` AS
`age`,`student`.`classId` AS `classId` from `student` where (`student`.`id` < 10)"
    }
   ] /* steps */
  } /* join_preparation */
 },
 {
   "join_optimization": {  # 进行优化
    "select#": 1,
    "steps": [
    {
      "condition_processing": {  # 条件处理
       "condition": "WHERE",
       "original_condition": "(`student`.`id` < 10)",
       "steps": [
       {
         "transformation": "equality_propagation",
         "resulting_condition": "(`student`.`id` < 10)"
       },
       {
         "transformation": "constant_propagation",
         "resulting_condition": "(`student`.`id` < 10)"
       },
       {
         "transformation": "trivial_condition_removal",
         "resulting_condition": "(`student`.`id` < 10)"
       }
] /* steps */
     } /* condition_processing */
    },
    {
      "substitute_generated_columns": {  # 替换生成的列
     } /* substitute_generated_columns */
    },
    {
      "table_dependencies": [   # 表的依赖关系
      {
        "table": "`student`",
        "row_may_be_null": false,
        "map_bit": 0,
        "depends_on_map_bits": [
       ] /* depends_on_map_bits */
      }
     ] /* table_dependencies */
    },
    {
      "ref_optimizer_key_uses": [   # 使用键
     ] /* ref_optimizer_key_uses */
    },
    {
      "rows_estimation": [   # 行判断
      {
        "table": "`student`",
        "range_analysis": {
         "table_scan": {
          "rows": 3973767,
          "cost": 408558
        } /* table_scan */,   # 扫描表
         "potential_range_indexes": [   # 潜在的范围索引
         {
           "index": "PRIMARY",
           "usable": true,
           "key_parts": [
            "id"
          ] /* key_parts */
         }
        ] /* potential_range_indexes */,
         "setup_range_conditions": [   # 设置范围条件
        ] /* setup_range_conditions */,
         "group_index_range": {
          "chosen": false,
          "cause": "not_group_by_or_distinct"
        } /* group_index_range */,
         "skip_scan_range": {
          "potential_skip_scan_indexes": [
          {
            "index": "PRIMARY",
            "usable": false,
            "cause": "query_references_nonkey_column"
          }
         ] /* potential_skip_scan_indexes */
        } /* skip_scan_range */,
         "analyzing_range_alternatives": {  # 分析范围选项
          "range_scan_alternatives": [
          {
"index": "PRIMARY",
            "ranges": [
             "id < 10"
           ] /* ranges */,
            "index_dives_for_eq_ranges": true,
            "rowid_ordered": true,
            "using_mrr": false,
            "index_only": false,
            "rows": 9,
            "cost": 1.91986,
            "chosen": true
          }
         ] /* range_scan_alternatives */,
          "analyzing_roworder_intersect": {
           "usable": false,
           "cause": "too_few_roworder_scans"
         } /* analyzing_roworder_intersect */
        } /* analyzing_range_alternatives */,
         "chosen_range_access_summary": {   # 选择范围访问摘要
          "range_access_plan": {
           "type": "range_scan",
           "index": "PRIMARY",
           "rows": 9,
           "ranges": [
            "id < 10"
          ] /* ranges */
         } /* range_access_plan */,
          "rows_for_plan": 9,
          "cost_for_plan": 1.91986,
          "chosen": true
        } /* chosen_range_access_summary */
       } /* range_analysis */
      }
     ] /* rows_estimation */
    },
    {
      "considered_execution_plans": [  # 考虑执行计划
      {
        "plan_prefix": [
       ] /* plan_prefix */,
        "table": "`student`",
        "best_access_path": {  # 最佳访问路径
         "considered_access_paths": [
         {
           "rows_to_scan": 9,
           "access_type": "range",
           "range_details": {
            "used_index": "PRIMARY"
          } /* range_details */,
           "resulting_rows": 9,
           "cost": 2.81986,
           "chosen": true
         }
        ] /* considered_access_paths */
       } /* best_access_path */,
        "condition_filtering_pct": 100,  # 行过滤百分比
        "rows_for_plan": 9,
        "cost_for_plan": 2.81986,
        "chosen": true
      }
     ] /* considered_execution_plans */
    },
    {
      "attaching_conditions_to_tables": {  # 将条件附加到表上
       "original_condition": "(`student`.`id` < 10)",
       "attached_conditions_computation": [
      ] /* attached_conditions_computation */,
       "attached_conditions_summary": [  # 附加条件概要
       {
         "table": "`student`",
         "attached": "(`student`.`id` < 10)"
       }
      ] /* attached_conditions_summary */
     } /* attaching_conditions_to_tables */
    },
    {
      "finalizing_table_conditions": [
      {
        "table": "`student`",
        "original_table_condition": "(`student`.`id` < 10)",
        "final_table_condition  ": "(`student`.`id` < 10)"
      }
     ] /* finalizing_table_conditions */
    },
    {
      "refine_plan": [  # 精简计划
      {
        "table": "`student`"
      }
     ] /* refine_plan */
    }
   ] /* steps */
  } /* join_optimization */
 },
 {
   "join_execution": {   # 执行
    "select#": 1,
    "steps": [
   ] /* steps */
  } /* join_execution */
 }
] /* steps */
}
# 第3部分：跟踪信息过长时，被截断的跟踪信息的字节数。
MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0  # 丢失的超出最大容量的字节
# 第4部分：执行跟踪语句的用户是否有查看对象的权限。当不具有权限时，该列信息为1且TRACE字段为空，一般在
调用带有 SQL SECURITY DEFINER 的视图或者是存储过程的情况下，会出现此问题。
INSUFFICIENT_PRIVILEGES: 0  # 缺失权限
1 row in set (0.00 sec)
```

# 四、MySQL 监控分析视图 sys schema

关于 MySQL 的**性能监控和问题诊断**，我们一般都从 **performance.schema** 中去获取想要的数据，在 MySQL5.7.7 版本
中新增 **sys schema** ,它将 **performance_schema** 和 **information_schema** 中的数据以**更容易理解的方式总结归纳为"视**
**图”**，其目的就是为了**降低查询 performance_schema 的复杂度**，让 DBA 能够**快速的定位问题**。下面看看这些库中
都有哪些监控表和视图，掌握了这些,在我们开发和运维的过程中就起到了事半功倍的效果。

**Sys schema 视图摘要**

1. **主机相关**：以 **host_summary** 开头，主要**汇总了 IO 延迟的信息**。
2. **Innodb 相关**：以 **innodb** 开头，汇总了 **innodb buffer 信息**和**事务等待 innodb 锁的信息**。
3. **I/O 相关**：以 **IO** 开头，汇总了**等待 I/O、I/O 使用量**情况。
4. **内存**使用情况：以 **memory** 开头，从**主机、线程、事件**等角度**展示内存的使用情况**
5. **连接与会话**信息：**processlist** 和 **session** 相关视图，总结了**会话相关信息**。
6. **表**相关：以 **schema_table** 开头的视图，展示了**表的统计信息**。
7. **索引**信息：统计了**索引的使用情况**，包含**冗余索引**和**未使用的索引**情况。
8. **语句**相关：以 **statement** 开头，包含执行**全表扫描**、使用**临时表**、**排序**等的语句信息。
9. **用户**相关：以 **user** 开头的视图，统计了用户使用的**文件 I/O**、**执行语句统计信息**。
10. **等待事件**相关信息：以 **wait** 开头，**展示等待事件的延迟情况**。

**Sys schema视图使用场景**

1. **索引情况**

   ```mysql
   #1. 查询冗余索引
   select * from sys.schema_redundant_indexes;
   
   #2. 查询未使用过的索引
   select * from sys.schema_unused_indexes;
   
   #3. 查询索引的使用情况
   select index_name,rows_selected,rows_inserted,rows_updated,rows_deleted
   from sys.schema_index_statistics where table_schema='dbname';
   ```

2. **表相关**

   ```mysql
   # 1. 查询表的访问量
   select table_schema,table_name,sum(io_read_requests+io_write_requests) as io from sys.schema_table_statistics group by table_schema,table_name order by io desc;
   
   # 2. 查询占用bufferpool较多的表
   select object_schema,object_name,allocated,data
   from sys.innodb_buffer_stats_by_table order by allocated limit 10;
   
   # 3. 查看表的全表扫描情况
   select * from sys.statements_with_full_table_scans where db='dbname';
   ```

3. **语句相关**

   ```mysql
   #1. 监控SQL执行的频率
   select db,exec_count,query from sys.statement_analysis
   order by exec_count desc;
   
   #2. 监控使用了排序的SQL
   select db,exec_count,first_seen,last_seen,query
   from sys.statements_with_sorting limit 1;
   
   #3. 监控使用了临时表或者磁盘临时表的SQL
   select db,exec_count,tmp_tables,tmp_disk_tables,query
   from sys.statement_analysis where tmp_tables>0 or tmp_disk_tables >0
   order by (tmp_tables+tmp_disk_tables) desc;
   ```

4. **IO相关**

   ```mysql
   #查看消耗磁盘IO的文件
   select file,avg_read,avg_write,avg_read+avg_write as avg_io
   from sys.io_global_by_file_by_bytes order by avg_read  limit 10;
   ```

5. **InnoDB相关**

   ```mysql
   #行锁阻塞情况
   select * from sys.innodb_lock_waits;
   ```

> **风险提示**：
>
> 通过 sys 库去查询时，MySQL 会**消耗大量资源**去收集相关信息，严重的可能会导致业务请求被阻塞，从而引起故障。建议生产上**不要频繁的去查询 ** sys 或者 **performance_schema**、 **information_schema** 来完成监控、巡检等工作。

# 小结：

查询数据库中**最频繁的操作**，提高查询速度可以有效地提高 MySQL 数据库的性能。通过对查询语句的分析可以了解查询语句的执行情况，找出查询语句执行的瓶颈，从而优化查询语句！