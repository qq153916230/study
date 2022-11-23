> 参考来源：
>
> 康师傅：https://www.bilibili.com/video/BV1iq4y1u7vj?p=141
>
> 爱编程的大李子：https://blog.csdn.net/LXYDSF/article/details/126606855

# 一、优化概述

都有哪些纬度可以进行数据库调优？简言之：

- 索引失效、没有充分利用所以——**索引建立**
- 关联查询太多 JOIN（设计缺陷或不得已的需求）——**SQL 优化**
- 服务器调优及各个参数设置（缓冲、 线程数）——**调整 my.cnf**
- 数据过多——**分库分表**

关于数据库调优的知识点非常分散，不同 DBMS，不同的公司，不同的职位，不同的项目遇到的问题都不尽相同。

虽然 SQL 查询优化的技术很多，但是大体方向上完全可以分为 **物理查询优化** 和 **逻辑查询优化** 两大块。

- 物理查询优化是通过 **索引** 和 **表连接方式** 等技术来进行优化，这里重点需要掌握索引的使用
- 逻辑查询优化就是通过 SQL **等价变换** 提升查询效率，直白一点来讲就是，换一种执行效率更高的查询写法

# 二、数据准备

学员表插50万条， 班级表插1万条。

## 1. 建表

```mysql
# 班级表
CREATE TABLE `class` (
`id` INT(11) NOT NULL AUTO_INCREMENT,
`className` VARCHAR(30) DEFAULT NULL,
`address` VARCHAR(40) DEFAULT NULL,
`monitor` INT NULL ,
PRIMARY KEY (`id`)
) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

# 学员表
CREATE TABLE `student` (
`id` INT(11) NOT NULL AUTO_INCREMENT,
`stuno` INT NOT NULL ,
`name` VARCHAR(20) DEFAULT NULL,
`age` INT(3) DEFAULT NULL,
`classId` INT(11) DEFAULT NULL,
PRIMARY KEY (`id`)
# CONSTRAINT `fk_class_id` FOREIGN KEY (`classId`) REFERENCES `t_class` (`id`)
) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

## 2. 设置参数

```mysql
# 命令开启：允许创建函数设置：
set global log_bin_trust_function_creators=1;   
# 不加global只是当前窗口有效。
```

## 3. 创建函数

```mysql
# 随机产生字符串
DELIMITER //
CREATE FUNCTION rand_string(n INT) RETURNS VARCHAR(255)
BEGIN  
DECLARE chars_str VARCHAR(100) DEFAULT
'abcdefghijklmnopqrstuvwxyzABCDEFJHIJKLMNOPQRSTUVWXYZ';
DECLARE return_str VARCHAR(255) DEFAULT '';
DECLARE i INT DEFAULT 0;
WHILE i < n DO 
SET return_str =CONCAT(return_str,SUBSTRING(chars_str,FLOOR(1+RAND()*52),1)); 
SET i = i + 1;
END WHILE;
RETURN return_str;
END //
DELIMITER ;

# 假如要删除
# drop function rand_string;
```

随机产生班级编号

```mysql
# 用于随机产生多少到多少的编号
DELIMITER //
CREATE FUNCTION rand_num (from_num INT ,to_num INT) RETURNS INT(11)
BEGIN 
DECLARE i INT DEFAULT 0; 
SET i = FLOOR(from_num +RAND()*(to_num - from_num+1))  ;
RETURN i; 
END //
DELIMITER ;

# 假如要删除
# drop function rand_num;
```

## 4. 创建存储过程

```mysql
# 创建往stu表中插入数据的存储过程
DELIMITER //
CREATE PROCEDURE insert_stu(  START INT , max_num INT )
BEGIN 
	DECLARE i INT DEFAULT 0; 
	SET autocommit = 0;   #设置手动提交事务
	REPEAT  #循环
	SET i = i + 1;  #赋值
	INSERT INTO student (stuno, name ,age ,classId ) VALUES
	((START+i),rand_string(6),rand_num(1,50),rand_num(1,1000)); 
	UNTIL i = max_num 
	END REPEAT; 
	COMMIT;  #提交事务
END //
DELIMITER ;

# 假如要删除
# drop PROCEDURE insert_stu;
```

创建往 class 表中插入数据的存储过程

```mysql
# 执行存储过程，往class表添加随机数据
DELIMITER //
CREATE PROCEDURE `insert_class`( max_num INT )
BEGIN 
	DECLARE i INT DEFAULT 0; 
	SET autocommit = 0;  
	REPEAT 
	SET i = i + 1; 
	INSERT INTO class ( classname,address,monitor ) VALUES
	(rand_string(8),rand_string(10),rand_num(1,100000)); 
	UNTIL i = max_num 
	END REPEAT; 
	COMMIT;
END //
DELIMITER ;

# 假如要删除
# drop PROCEDURE insert_class;
```

## 5. 调用存储过程

往 class 表添加1万条数据

```mysql
# 执行存储过程，往class表添加1万条数据 
CALL insert_class(10000);
```

往 stu 表添加50万条数据，这个时间会稍微有点长，看个人电脑性能

```mysql
# 执行存储过程，往stu表添加80万条数据 
CALL insert_stu(100000,800000);
```

查询下数据是否插入成功

```mysql
SELECT COUNT(*) FROM class;
SELECT COUNT(*) FROM student;
```

## 6. 删除某表上的索引

创建删除索引存储过程。这是为了方便我们的学习，因为我们在演示某个索引的效果时，可能需要删除其它索引，如果需要一个个手工删除，就太费劲了。

```mysql
DELIMITER //
CREATE  PROCEDURE `proc_drop_index`(dbname VARCHAR(200),tablename VARCHAR(200))
BEGIN
   DECLARE done INT DEFAULT 0;
   DECLARE ct INT DEFAULT 0;
   DECLARE _index VARCHAR(200) DEFAULT '';
   DECLARE _cur CURSOR FOR  SELECT  index_name  FROM
information_schema.STATISTICS  WHERE table_schema=dbname AND table_name=tablename AND
seq_in_index=1 AND  index_name <>'PRIMARY' ;
# 每个游标必须使用不同的declare continue handler for not found set done=1来控制游标的结束
   DECLARE  CONTINUE HANDLER FOR NOT FOUND set done=2 ;   
# 若没有数据返回,程序继续,并将变量done设为2
    OPEN _cur;
    FETCH _cur INTO _index;
    WHILE _index<>'' DO
       SET @str = CONCAT("drop index " , _index , " on " , tablename );
       PREPARE sql_str FROM @str ;
       EXECUTE sql_str;
       DEALLOCATE PREPARE sql_str;
       SET _index='';
       FETCH _cur INTO _index;
    END WHILE;
 CLOSE _cur;
END //
DELIMITER ;
```

执行存储过程

```mysql
# 传入要删除索引的 数据库名 和 表名
CALL proc_drop_index("dbname","tablename");
```

# 三、索引失效案例

MySQL 中**提高性能**的一个最有效的方式是对数据表**设计合理的索引**。索引提供了高效访问数据的方法，并且加快查询的速度，因此索引对查询的速度有着至关重要的影响。

- 使用索引可以 **快速地定位** 表中的某条记录，从而提高数据库查询的速度，提高数据库的性能。
- 如果查询时没有使用索引，查询语句就会 **扫描表中的所有记录**。在数据量大的情况下，这样查询的速度会很慢。

大多数情况下都（默认）采用 **B+树 **来构建索引。只是空间列类型的索引使用 **R-树**，并且 MEMORY 表还支持 **hash 索引**。

其实，用不用索引，最终都是优化器说了算。优化器是基于什么的优化器?基于 **cost 开销(CostBaseOptimizer)**，它不是基于规则(Rule-BasedOptimizer)，也不是基于语义。怎么样开销小就怎么来。另外，**SQL 语句是否使用索引，跟数据库版本、数据量、数据选择度都有关系**。

> 这里讲的是基于 B+树的索引失效案例，**了解 B+树可以更好的了解索引失效原理**

## 1. 全值匹配

**全值匹配**可以充分的利用 复合 (组合、联合) 索引，**建立几个复合索引**字段，WHERE 里面最好就**用上几个字段**。且按照顺序来用。否则会造成后面的**索引失效**。

比如建立索引 index(a,b,c)，如果 where 中只用了 a 和 c，那么只有 a 字段使用了索引，c 字段索引就失效了，因为 b 字段没用上。

## 2. 最左匹配原则

在 MySQL 建立联合索引时会遵守最佳左前缀匹配原则，即最左优先，在检索数据时从联合索引的最左边开始匹配。

同上面的 **全值匹配** 一样，查询时如果字段没有按照组合索引建立的顺序，优化器会执行优化，会调整查询条件的顺序。不过在开发过程中我们还是要保持良好的开发习惯。

结论：MySQL 可以为多个字段创建索引，一个索引可以包括 16 个字段，对于多列字段，**过滤条件要使用索引那必须按照索引建立时的顺序，依次满足，一旦跳过某个字段，索引后面的字段都无法使用**。如果查询条件中没有使用这些字段中的第一个字段时，多列索引不会被使用。

> **拓展:Alibaba《Java开发手册》**
>
> 索引文件具有 B-Tree 的最左前缀匹配特性，如果左边的值未确定，那么无法使用此索引。

## 3. 计算、函数、类型转换导致索引失效

SQL 语句中中 WHERE 后面使用 计算、函数、类型转换（自动或手动）导致索引失效

```mysql
# LEFT 函数导致索引失效
SELECT SQL_NO_CACHE * FROM student WHERE LEFT(student.name,3) = 'abc';

# 条件中有计算时致索引失效
SELECT SQL_NO_CACHE id, stuno, NAME FROM student WHERE stuno+1 = 900001; 

# 表中name字段为字符类型，但是条件写了 int 整形 name = 123 发生类型转换，相当于使用了隐形 函数，索引失效
SELECT SQL_NO_CACHE * FROM student WHERE name=123;
```

实际测试中 **字段为 int 类型**，而 WHERE 后面**条件写成字符串类型 '123' 并不会导致索引失效**

## 4. 范围条件右边的列索引失效

```mysql
# 对于组合索引 index(age, classId, name)，classId使用范围查找(<、<=、>、>= 和 between 等) 导致 name 字段索引失效
SELECT * FROM student WHERE student.age=30 AND student.classId>20 AND student.name = 'abc' ;
```

> **解释一下为什么范围查询会导致索引失效：**
>
> 因为根据范围查找筛选后的数据，无法保证范围查找后面的字段是有序的。
>
> 例如：a_b_c这个索引，你根据b范围查找>2的，在满足b>2的情况下，如b：3,4，c可能是5,3、因为c无序，那么c的索引便失效了

**改进：**

可以把要**范围查找的字段放在组合索引的最后一个**，例如把上面的 index(age, classId, name) 改成 index(age, name, classId)

## 5. 不等于（!= 或者 <>）索引失效

使用等于精确查找可以使用索引，但是使用不等于之后，筛选出来的数据太多，还不如直接全表扫描，少了索引查找和回表的过程。

## 6. IS NOT NULL、NOT  LIKE 无法使用索引

原理同上，IS NOT NULL、NOT  LIKE 搜索的数据量太多，索引失效

## 7. LIKE 以通配符 % 开头索引失效

在使用 LIKE 关键字进行查询的查询语句中，不确定开头是哪些字符，无法精确匹配，必须全表扫描

> 拓展：Alibaba《Java 开发手册》
>
> 【强制】页面搜索严禁左模糊或者全模糊，如果需要请走搜索引擎来解决。

## 8. OR 前后存在非索引的列，索引失效

OR 的含义就是两个只要满足一个即可，因此只要有条件列没有进行索引，就会进行全表扫描，因此索引的条件列也会失效。因为 OR前后一个使用索引，一个进行全表扫描，还没有直接进行全表扫描更快。

## 9. 数据库和表的字符集统一使用 utf8mb4/utf8mb3

统一使用 utf8mb4（5.5.3版本以上支持）兼容性更好，统一字符集可以避免由于字符集转换产生的乱码。不同的 字符集 进行比较前需要进行 转换 会造成索引失效。

## 10. JOIN 两个表的字段不一致，索引失效

类似于第三点，存在类型转换，导致索引失效



**建议：**

- 对于单列索引，尽量选择针对当前 query 过滤性更好的索引
- 在选择组合索引的时候，当前 query 中过滤性最好的字段在索引字段顺序中，位置越靠前越好
- 在选择组合索引的时候，尽量选择能够包含当前 query 中的 where 子句中更多字段的索引
- 在选择组合索引的时候，如果某个字段可能出现范围查询时，尽量把这个字段放在索引次序的最后面。

总之，书写 SQL 语句时，尽量避免造成索引失效的情况。
