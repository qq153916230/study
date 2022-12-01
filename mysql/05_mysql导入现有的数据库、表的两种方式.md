# 1.CMD命令行

**1.1 source**

```mysql
# source 文件的全路径名
source d:\xxxdb.sql;
```

**1.2 LOAD DATA INFILE**

**LOAD DATA INFILE** 语句用于高速地从一个文本文件中读取行，并写入一个表中。文件名称必须为一个文字字符串。
LOAD DATA INFILE 是  **SELECT ... INTO OUTFILE** 的相对语句。把表的数据备份到文件使用SELECT ... INTO OUTFILE，从备份文件恢复表数据，使用 LOAD DATA INFILE。

标准示例：

```mysql
LOAD DATA LOCAL INFILE 'data.txt' INTO TABLE tbl_name 
FIELDS TERMINATED BY ',' 
OPTIONALLY ENCLOSED BY '"' 
LINES TERMINATED BY '\n'
```

链接：https://www.jianshu.com/p/bcafd8f3ad8e

# 2.基于具体的图形化界面的工具

以SQLyog为例：

在SQLyog中 选择 “工具” -- “执行sql脚本” -- 选中xxx.sql即可。