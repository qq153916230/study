# 一、概述

子查询指一个查询语句嵌套在另一个查询语句内部的查询

SQL 中子查询的使用大大增强了 SELECT 查询的能力，因为很多时候查询需要从结果集中获取数据，或者
需要从同一个表中先计算得出一个数据结果，然后与这个数据结果（可能是某个标量，也可能是某个集
合）进行比较。

```mysql
# 查询谁的年龄比张三高

# 方式一：分步查询
SELECT age
FROM user
WHERE name = '张三';
# 假设查询出的结果为：25

# 利用上面查询的结果进行下一步查询
SELECT name,age
FROM user
WHERE age > 25;

#方式二：自连接
SELECT u2.name,u2.age
FROM user u1,user u2
WHERE u1.name = '张三'
AND u1.age < u2.age

# 方式三：利用子查询
SELECT name,age
FROM user
WHERE age > (
                SELECT age
                FROM user
                WHERE name = '张三'
            );
```

#  二、子查询的基本使用

- 子查询的基本语法结构：![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/45.png)
- 子查询（内查询）在主查询之前一次执行完成。
- 子查询的结果被主查询（外查询）使用 。

**注意事项**

- 子查询要包含在括号内
- 将子查询放在比较条件的右侧
- 单行操作符对应单行子查询，多行操作符对应多行子查询

# 三、子查询的分类

**分类方式1：**

按内查询的结果返回一条还是多条记录，将子查询分为 **单行子查询** 、 **多行子查询** 。

**分类方式2：**

我们按内查询是否被执行多次，将子查询划分为 **相关(或关联)子查询** 和 **不相关(或非关联)子查询** 。

子查询从数据表中查询了数据结果，如果这个数据结果只执行一次，然后这个数据结果作为主查询的条件进行执行，那么这样的子查询叫做**不相关子查询**。

同样，如果子查询需要执行多次，即采用循环的方式，先从外部查询开始，每次都传入子查询进行查询，然后再将结果反馈给外部，这种嵌套的执行方式就称为**相关子查询**。

## 1.单行子查询

### 1.1 单行比较操作符

```mysql
# 示例：=、<>、>、>=、<、<=
SELECT name,age
FROM user
WHERE age > (
                SELECT age
                FROM user
                WHERE name = '张三'
            );
            
# 利用单行函数：min()...
SELECT name, age
FROM user
WHERE age = 
		(SELECT MIN(age)
		FROM user);

# 成对比较
SELECT name, age
FROM user
WHERE (name, age) IN
				(SELECT name, age
				FROM user
				WHERE age IN (18,25));

```

### 1.2 HAVING 中的子查询

- 首先执行子查询。
- 向主查询中的HAVING 子句返回结果。

```mysql
# 查询 班级和班级最小年纪比1班中最小年纪大的有哪些
SELECT class_id, MIN(age)
FROM user
GROUP BY class_id
HAVING MIN(age) >
            (SELECT MIN(age)
            FROM user
            WHERE class_id = 1);
```

### 1.3 CASE中的子查询

```mysql
# 显式员工的employee_id,last_name和location。其中，若员工department_id与location_id为1800的department_id相同，则location为’Canada’，其余则为’USA’。
SELECT employee_id, last_name,
        (CASE department_id
        WHEN
            (SELECT department_id FROM departments
            WHERE location_id = 1800)
        THEN 'Canada' ELSE 'USA' END) location
FROM employees;
```

## 2. 多行子查询

- 也称为集合比较子查询
- 内查询返回多行
- 使用多行比较操作符

### 2.1 多行比较操作符

| **操作符** | **含义**                                                     |
| ---------- | ------------------------------------------------------------ |
| IN         | 等于列表中的**任意一个**                                     |
| ANY        | 需要和单行比较操作符一起使用，和子查询返回的**某一个**值比较 |
| ALL        | 需要和单行比较操作符一起使用，和子查询返回的**所有**值比较   |
| SOME       | 实际上是ANY的别名，作用相同，一般常使用ANY                   |

```mysql
# 查询平均年龄最低的班级
#方式1：
SELECT class_id, age
FROM class
GROUP BY class_id
HAVING AVG(age) = (
    SELECT MIN(avg_age)
    FROM (
        SELECT AVG(age) avg_age
        FROM class
        GROUP BY class_id
        ) class_avg_age
    )

#方式2：
SELECT class_id, age
FROM class
GROUP BY class_id
HAVING AVG(age) <= ALL (
    SELECT AVG(age) avg_age
    FROM class
    GROUP BY class_id
    )
```

##  3.相关子查询

### 3.1相关子查询执行流程

如果子查询的执行依赖于外部查询，通常情况下都是因为子查询中的表用到了外部的表，并进行了条件关联，因此每执行一次外部查询，子查询都要重新计算一次，这样的子查询就称之为 **关联子查询** 。

![](https://raw.githubusercontent.com/qq153916230/study/main/mysql/pic/46.png)

说明：**子查询中使用主查询中的列**

### 3.2 在 FROM 中使用子查询

```mysql
# 查询员工中工资大于本部门平均工资的员工
SELECT last_name,salary,e1.department_id
FROM employees e1,(SELECT department_id,AVG(salary) dept_avg_sal FROM employees GROUP
BY department_id) e2
WHERE e1.`department_id` = e2.department_id
AND e2.dept_avg_sal < e1.`salary`;

```

> from型的子查询：子查询是作为from的一部分，子查询要用()引起来，并且要给这个子查询取别名， 把它当成一张“临时的虚拟的表”来使用。

### 3.3 在WHERE 中使用子查询

```mysql
# 查询工作小于3份的员工信息
SELECT e.employee_id, last_name,e.job_id
FROM employees e
WHERE 3 < (SELECT COUNT(*)
            FROM job_history
            WHERE employee_id = e.employee_id);
```

### 3.4 EXISTS 与 NOT EXISTS关键字

- 关联子查询通常也会和 EXISTS操作符一起来使用，用来检查在子查询中是否存在满足条件的行。
- **如果在子查询中不存在满足条件的行**：
  - 条件返回 FALSE
  - 继续在子查询中查找
- **如果在子查询中存在满足条件的行**：
  - 不在子查询中继续查找
  - 条件返回 TRUE
- NOT EXISTS关键字表示如果不存在某种条件，则返回TRUE，否则返回FALSE。

```mysql
# 查询公司管理者的信息

# 方式一：
SELECT employee_id, last_name, job_id, department_id
FROM employees e1
WHERE EXISTS ( SELECT *
                FROM employees e2
                WHERE e2.manager_id = e1.employee_id);
# 方式二：
SELECT employee_id,last_name,job_id,department_id
FROM employees
WHERE employee_id IN (
                    SELECT DISTINCT manager_id
                    FROM employees);
                    
# 方式三：自连接
SELECT DISTINCT e1.employee_id, e1.last_name, e1.job_id, e1.department_id
FROM employees e1 JOIN employees e2
WHERE e1.employee_id = e2.manager_id;

```

```mysql
# 查询没有员工的部门信息
SELECT department_id, department_name
FROM departments d
WHERE NOT EXISTS (SELECT 'X'
                    FROM employees
                    WHERE department_id = d.department_id);
```

> EXISTS 与 NOT EXISTS关键字的子查询中 SELECT 后面随便写都可以

### 3.5 相关更新

```mysql
# 使用相关子查询依据一个表中的数据更新另一个表的数据。
UPDATE table1 alias1
SET column = (SELECT expression
                FROM table2 alias2
                WHERE alias1.column = alias2.column);
```

```mysql
# 在employees中增加一个department_name字段，数据为员工对应的部门名称
# 1）
ALTER TABLE employees
ADD(department_name VARCHAR2(14));
# 2）
UPDATE employees e
SET department_name = (SELECT department_name
                        FROM departments d
                        WHERE e.department_id = d.department_id);
```

### 3.6 相关删除

```mysql
# 删除表employees中，其与emp_history表皆有的数据
DELETE FROM employees e
WHERE employee_id in (SELECT employee_id
                        FROM emp_history
                        WHERE employee_id = e.employee_id);
```

# 四、思考

自连接和子查询可以实现相同的功能，那么用哪个更好？

康师傅推荐：自连接方式好！

题目中可以使用子查询，也可以使用自连接。一般情况建议你使用自连接，因为在许多 DBMS 的处理过
程中，对于自连接的处理速度要比子查询快得多。

可以这样理解：子查询实际上是通过未知表进行查询后的条件判断，而自连接是通过已知的自身数据表
进行条件判断，因此在大部分 DBMS 中都对自连接处理进行了优化。
