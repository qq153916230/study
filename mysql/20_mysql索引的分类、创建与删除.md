> 参考来源：
>
> 康师傅：https://www.bilibili.com/video/BV1iq4y1u7vj?p=128
>
> 爱编程的大李子：https://blog.csdn.net/LXYDSF/article/details/126247744

# 一、索引的声明与使用

## 1. 索引的分类

MySQL 的索引包括普通索引、唯一性索引、全文索引、单列索引、多列索引和空间索引等。

- 从 **功能逻辑** 按照上说，索引主要有 4 种：**普通索引、唯一索引、主键索引、全文索引**。
- 按照 **物理实现方式** ，索引可以分为 2 种：**聚簇索引**和**非聚簇索引**。
- 按照 **作用字段个数** 进行划分，分成**单列索引**和**联合索引**。
- **前缀索引**：通过截取字段的前面一部分内容建立索引

## 2. 创建索引

MySQL支持多种方法在单个或多个列上创建索引：在创建表的定义语句**CREATE TABLE**中指定索引列，使用 **ALTER TABLE** 语句在存在的表上创建索引，或者使用**CREATE INDEX** 语句在已存在的表上添加索引。

### 2.1 创建表的时候创建索引

使用CREATE TABLE创建表时，除了可以定义列的数据类型外，还可以定义主键约束、外键约束或者唯一性约束，而不论创建哪种约束，在定义约束的同时相当于在指定列上创建了一个索引。

- **隐式的索引创建**：

  ```mysql
  # 1.隐式的添加索引(在添加有主键约束、唯一性约束或者外键约束的字段会自动的创建索引)
  CREATE TABLE dept(
      dept_id INT PRIMARY KEY AUTO_INCREMENT,# 创建主键索引
      dept_name VARCHAR(20)
  );
  CREATE TABLE emp(
      emp_id INT PRIMARY KEY AUTO_INCREMENT,# 主键索引
      emp_name VARCHAR(20) UNIQUE,# 唯一索引
      dept_id INT,
      CONSTRAINT emp_dept_id_fk FOREIGN KEY(dept_id) REFERENCES dept(dept_id)
  ); # 外键索引
  ```

- **显式的索引创建**的话，基本语法格式如下，共有七种情况~

  ```mysql
  CREATE TABLE table_name [col_name data_type]
  [UNIQUE | FULLTEXT | SPATIAL] [INDEX | KEY] [index_name] (col_name [length]) [ASC | DESC]
  ```

  - **UNIQUE**、 **FULLTEXT** 和 **SPATIAL** 为可选参数，分别表示唯一索引、全文索引和空间索引;
  - **INDEX**与**KEY** 为同义词，两者的作用相同，用来指定创建索引;
  - **index_name** 指定索引的名称，为可选参数，如果不指定，那么 MySQL 默认 col_name 为索引名;
  - **col_name**为需要创建索引的字段列，该列必须从数据表中定义的多个列中选择;
  - **length** 为可选参数，表示索引的长度，只有字符串类型的字段才能指定索引长度;
  - **ASC** 或 **DESC** 指定升序或者降序的索引值存储。
  - 特例：主键索引使用主键约束的方式来创建。
    

#### 2.1.1 创建普通索引

```mysql
# 创建普通的索引
CREATE TABLE user(
    id INT ,
    name VARCHAR(100),
    # 声明索引
    INDEX idx_name(name)
);
```

#### 2.1.2 创建唯一索引

```mysql
# ②创建唯一索引
CREATE TABLE user (
  	id INT,
    name VARCHAR(100),
    email VARCHAR(100),
    # 声明索引
  	UNIQUE INDEX uk_idx_email (email)
);
```

#### 2.1.3 创建主键索引

```mysql
# 主键索引

create table user(
    id int primary key, # 通过定义主键约束的方式定义主键索引
    name varchar(100)
) ;
```

#### 2.1.4 创建组合索引

```mysql
# 创建联合索引
create table user(
    id INT,
    name VARCHAR(100),
    age TINYINT (3),
    index mul_name_age(name,age)	
)
```

### 2.2 在已经存在的表上创建索引

在已经存在的表中创建索引可以使用 ALTER TABLE 语句或者 CREATE INDEX 语句。

#### 2.2.1 使用 ALTER TABLE 语句创建索引

```mysql
ALTER TABLE table_name ADD [UNIQUE | FULLTEXT | SPATIAL] [INDEX | KEY]
[index_name] (col_name[length],...) [ASC | DESC]
```

```mysql
# 给用户表的name字段添加索引
ALTER TABLE user ADD INDEX idx_name(name);
```

#### 2.2.2 使用 CREATE INDEX 创建索引

```mysql
CREATE [UNIQUE | FULLTEXT | SPATIAL] INDEX index_name
ON table_name (col_name[length],...) [ASC | DESC]
```

```mysql
# 给用户表的name字段添加索引
CREATE INDEX idx_name ON user(name);
```

## 3. 查看索引

通过命令查看索引有没有创建成功

```mysql
# 方式1：
SHOW CREATE TABLE book;

# 方式2：
SHOW INDEX FROM book;
```

性能分析工具：EXPLAIN，查看索引是否正在使用

```mysql
EXPLAIN SELECT * from user where user_name = '张三';
```

## 4. 删除索引

MySQL中删除索引使用 **ALTER TABLE** 或 **DROP INDEX** 语句，两者可实现相同的功能，DROP INDEX语句在内部被映射到一个ALTER TABLE语句中

```mysql
# ALTER TABLE删除索引
ALTER TABLE table_name DROP INDEX index_name;

# DROP INDEX删除索引
DROP INDEX index_name ON table_name;
```

# 二、MySQL 8.0 索引新特性

## 1. 支持降序索引

降序索引以降序存储键值。虽然在语法上，从MySQL 4版本开始就已经支持降序索引的语法了，但实际上该DESC定义是被忽略的，直到MySQL 8.x版本才开始真正支持降序索引(仅限于InnoDB存储引擎)。

MySQL在**8.0版本之前创建的仍然是升序索引，使用时进行反向扫描，这大大降低了数据库的效率**。在某些场景下，降序索引意义重大。例如，如果一个查询，需要对多个列进行排序，且顺序要求不一致,那么使用降序索引将会避免数据库使用额外的文件排序操作，从而提高性能。

## 2. 隐藏索引（invisible indexes）

在 MySQL 5.7 版本及之前，只能通过显式的方式删除索引。此时，如果发现删除索引后出现错误，又只能通过显式创建索引的方式将删除的索引创建回来。如果数据表中的数据量非常大，或者数据表本身比较大，这种操作就会消耗系统过多的资源，操作成本非常高。

从MySQL 8.x 开始支持 **隐藏索引(invisible indexes)**，只需要将待删除的索引设置为隐藏索引，使查询优化器不再使用这个索引（即使使用 force index（强制使用索引），优化器也不会使用该索引）， 确认将索引设置为隐藏索引后系统不受任何响应，就可以彻底删除索引。**这种通过先将索引设置为隐藏索引，再删除索引的方式就是软删除。**

- 在 MySQL 中创建隐藏索引通过 SQL 语句 **INVISIBLE** 来实现，其语法形式如下：

  ```mysql
  CREATE TABLE tablename(
      propname1 type1[CONSTRAINT1],
      propname2 type2[CONSTRAINT2],
      ......
      propnamen typen,
      INDEX [indexname](propname1 [(length)]) INVISIBLE
  );
  ```

  上述语句比普通索引多了一个关键字 **INVISIBLE**，用来标记索引为不可见索引。

- **在已经存在的表上创建**

  ```mysql
  CREATE INDEX indexname
  ON tablename(propname[(length)]) INVISIBLE;
  ```

- **通过 ALTER TABLE 语句创建**

  ```mysql
  ALTER TABLE tablename
  ADD INDEX indexname (propname [(length)]) INVISIBLE;
  ```

- **切换索引可见状态**

  已存在的索引可通过如下语句切换可见状态：
  ```mysql
   ALTER TABLE tablename ALTER INDEX index_name INVISIBLE; #切换成隐藏索引 
   ALTER TABLE tablename ALTER INDEX index_name VISIBLE; #切换成非隐藏索引
  ```

> **注意：**当索引被隐藏时，它的内容仍然是和正常索引一样实时更新的。如果一个索引需要长期被隐藏，那么可以将其删除，因为索引的存在会影响插入、更新和删除的性能。

使隐藏索引对查询优化器可见(了解)

在 MySQL 8.x 版本中，为索引提供了一种新的测试方式，可以通过查询优化器的一个开关 （use_invisible_indexes）来打开某个设置，使隐藏索引对查询优化器可见。如果 use_invisible_indexes 设置为 off（默认），优化器会忽略隐藏索引。如果设置为 on，即使隐藏索引不可见，优化器在生成执行计划时仍会考虑使用隐藏索引。

