# mysql下载

MySQL的4大版本

- MySQL Community Server 社区版本，开源免费，自由下载，但不提供官方技术支持，适用于大多数普通用户。
- MySQL Enterprise Edition 企业版本，需付费，不能在线下载，可以试用30天。提供了更多的功能和更完备的技术支持，更适合于对数据库的功能和可靠性要求较高的企业客户。
- MySQL Cluster 集群版，开源免费。用于架设集群服务器，可将几个MySQL Server封装成一个Server。需要在社区版或企业版的基础上使用。
- MySQL Cluster CGE 高级集群版，需付费。

## 1.下载地址

官网：https://www.mysql.com

## 2.打开官网，点击DOWNLOADS

然后，点击 MySQL Community(GPL) Downloads

![](D:\mine\study\mysql\pic\9.png)

## 3.点击 MySQL Community Server

![](D:\mine\study\mysql\pic\10.png)

## 4.在General Availability(GA) Releases中选择适合的版本

Windows平台下提供两种安装文件：MySQL二进制分发版（.msi安装文件）和免安装版（.zip压缩文件）。一般来讲，应当使用二进制分发版，因为该版本提供了图形化的安装向导过程，比其他的分发版使用起来要简单，不再需要其他工具启动就可以运行MySQL。
这里在Windows 系统下推荐下载 MSI安装程序；点击 Go to Download Page 进行下载即可

![](D:\mine\study\mysql\pic\11.png)

![](D:\mine\study\mysql\pic\12.png)

Windows下的MySQL8.0安装有两种安装程序

- mysql-installer-web-community-8.0.26.0.msi 下载程序大小：2.4M；安装时需要联网安装组件。
- mysql-installer-community-8.0.26.0.msi 下载程序大小：450.7M；安装时离线安装即可。推荐。

如果安装MySQL5.7版本的话，选择 Archives ，接着选择MySQL5.7的相应版本即可。这里下载最近
期的MySQL5.7.34版本。

![](D:\mine\study\mysql\pic\13.png)

![](D:\mine\study\mysql\pic\14.png)

# MySQL8.0 版本的安装

MySQL下载完成后，找到下载文件，双击进行安装，具体操作步骤如下。
步骤1：双击下载的mysql-installer-community-8.0.26.0.msi文件，打开安装向导。
步骤2：打开“Choosing a Setup Type”（选择安装类型）窗口，在其中列出了5种安装类型，分别是Developer Default（默认安装类型）、Server only（仅作为服务器）、Client only（仅作为客户端）、Full（完全安装）、Custom（自定义安装）。这里选择“Custom（自定义安装）”类型按钮，单击“Next(下一步)”按钮。

![](D:\mine\study\mysql\pic\15.png)

步骤3：打开“Select Products” （选择产品）窗口，可以定制需要安装的产品清单。例如，选择“MySQL Server 8.0.26-X64”后，单击“→”添加按钮，即可选择安装MySQL服务器，如图所示。采用通用的方法，可以添加其他你需要安装的产品。

![](D:\mine\study\mysql\pic\16.png)

此时如果直接“Next”（下一步），则产品的安装路径是默认的。如果想要自定义安装目录，则可以选中对应的产品，然后在下面会出现“Advanced Options”（高级选项）的超链接。

![](D:\mine\study\mysql\pic\17.png)

单击“Advanced Options”（高级选项）则会弹出安装目录的选择窗口，如图所示，此时你可以分别设置MySQL的服务程序安装目录和数据存储目录。如果不设置，默认分别在C盘的Program Files目录和ProgramData目录（这是一个隐藏目录）。如果自定义安装目录，请避免“中文”目录。另外，建议服务目录和数据目录分开存放。

![](D:\mine\study\mysql\pic\18.png)

步骤4：在上一步选择好要安装的产品之后，单击“Next”（下一步）进入确认窗口，如图所示。单击“Execute”（执行）按钮开始安装。

![](D:\mine\study\mysql\pic\19.png)

步骤5：安装完成后在“Status”（状态）列表下将显示“Complete”（安装完成），如图所示。

![](D:\mine\study\mysql\pic\20.png)

# 配置MySQL8.0

MySQL安装之后，需要对服务器进行配置。具体的配置步骤如下。
步骤1：在上一个小节的最后一步，单击“Next”（下一步）按钮，就可以进入产品配置窗口。

![](D:\mine\study\mysql\pic\21.png)

步骤2：单击“Next”（下一步）按钮，进入MySQL服务器类型配置窗口，如图所示。端口号一般选择默认端口号3306。

![](D:\mine\study\mysql\pic\22.png)

其中，“Config Type”选项用于设置服务器的类型。单击该选项右侧的下三角按钮，即可查看3个选项，如图所示。

![](D:\mine\study\mysql\pic\23.png)

- Development Machine（开发机器） ：该选项代表典型个人用桌面工作站。此时机器上需要运行多个应用程序，那么MySQL服务器将占用最少的系统资源。
- Server Machine（服务器） ：该选项代表服务器，MySQL服务器可以同其他服务器应用程序一起运行，例如Web服务器等。MySQL服务器配置成适当比例的系统资源。
- Dedicated Machine（专用服务器） ：该选项代表只运行MySQL服务的服务器。MySQL服务器配置成使用所有可用系统资源。

步骤3：单击“Next”（下一步）按钮，打开设置授权方式窗口。其中，上面的选项是MySQL8.0提供的新的授权方式，采用SHA256基础的密码加密方法；下面的选项是传统授权方法（保留5.x版本兼容性）。

![](D:\mine\study\mysql\pic\24.png)

步骤4：单击“Next”（下一步）按钮，打开设置服务器root超级管理员的密码窗口，如图所示，需要输入两次同样的登录密码。也可以通过“Add User”添加其他用户，添加其他用户时，需要指定用户名、允许该用户名在哪台/哪些主机上登录，还可以指定用户角色等。此处暂不添加用户，用户管理在康师傅MySQL高级特性篇中讲解。

![](D:\mine\study\mysql\pic\25.png)

步骤5：单击“Next”（下一步）按钮，打开设置服务器名称窗口，如图所示。该服务名会出现在Windows服务列表中，也可以在命令行窗口中使用该服务名进行启动和停止服务。本书将服务名设置为“MySQL80”。如果希望开机自启动服务，也可以勾选“Start the MySQL Server at System Startup”选项（推荐）。
下面是选择以什么方式运行服务？可以选择“Standard System Account”(标准系统用户)或者“Custom User”(自定义用户)中的一个。这里推荐前者。

![](D:\mine\study\mysql\pic\26.png)

步骤6：单击“Next”（下一步）按钮，打开确认设置服务器窗口，单击“Execute”（执行）按钮。

![](D:\mine\study\mysql\pic\27.png)

步骤7：完成配置，如图所示。单击“Finish”（完成）按钮，即可完成服务器的配置。

![](D:\mine\study\mysql\pic\28.png)

步骤8：如果还有其他产品需要配置，可以选择其他产品，然后继续配置。如果没有，直接选择“Next”（下一步），直接完成整个安装和配置过程。

![](D:\mine\study\mysql\pic\29.png)

步骤9：结束安装和配置。

![](D:\mine\study\mysql\pic\30.png)

配置MySQL8.0 环境变量

如果不配置MySQL环境变量，就不能在命令行直接输入MySQL登录命令。下面说如何配置MySQL的环境变量：

步骤1：在桌面上右击【此电脑】图标，在弹出的快捷菜单中选择【属性】菜单命令。 步骤2：打开【系统】窗口，单击【高级系统设置】链接。 

步骤3：打开【系统属性】对话框，选择【高级】选项卡，然后单击【环境变量】按钮。 步骤4：打开【环境变量】对话框，在系统变量列表中选择path变量。 

步骤5：单击【编辑】按钮，在【编辑环境变量】对话框中，将MySQL应用程序的bin目录（C:\Program Files\MySQL\MySQL Server 8.0\bin）添加到变量值中，用分号将其与其他路径分隔开。 

步骤6：添加完成之后，单击【确定】按钮，这样就完成了配置path变量的操作，然后就可以直接输入MySQL命令来登录
数据库了。

# MySQL5.7 版本的安装、配置

安装

此版本的安装过程与上述过程除了版本号不同之外，其它环节都是相同的。所以这里省略了MySQL5.7.34
版本的安装截图。

配置

配置环节与MySQL8.0版本确有细微不同。大部分情况下直接选择“Next”即可，不影响整理使用。
这里配置MySQL5.7时，重点强调：与前面安装好的MySQL8.0不能使用相同的端口号。

# 安装失败问题

MySQL的安装和配置是一件非常简单的事，但是在操作过程中也可能出现问题，特别是初学者。

## 问题1：无法打开MySQL8.0软件安装包或者安装过程中失败，如何解决？

在运行MySQL8.0软件安装包之前，用户需要确保系统中已经安装了.Net Framework相关软件，如果缺少此软件，将不能正常地安装MySQL8.0软件。

![](D:\mine\study\mysql\pic\31.png)

解决方案：到这个地址https://www.microsoft.com/en-us/download/details.aspx?id=42642下载Microsoft.NET Framework 4.5并安装后，再去安装MySQL。

另外，还要确保Windows Installer正常安装。windows上安装mysql8.0需要操作系统提前已安装好Microsoft Visual C++ 2015-2019。

![](D:\mine\study\mysql\pic\32.png)

![](D:\mine\study\mysql\pic\33.png)

解决方案同样是，提前到微软官网https://support.microsoft.com/en-us/topic/the-latest-supported-visual-c-downloads-2647da03-1eea-4433-9aff-95f26a218cc0，下载相应的环境。

## 问题2：卸载重装MySQL失败？

该问题通常是因为MySQL卸载时，没有完全清除相关信息导致的。
解决办法是，把以前的安装目录删除。如果之前安装并未单独指定过服务安装目录，则默认安装目录是“C:\Program Files\MySQL”，彻底删除该目录。同时删除MySQL的Data目录，如果之前安装并未单独指定过数据目录，则默认安装目录是“C:\ProgramData\MySQL”，该目录一般为隐藏目录。删除后，重新安装即可。

## 问题3：如何在Windows系统删除之前的未卸载干净的MySQL服务列表？

操作方法如下，在系统“搜索框”中输入“cmd”，按“Enter”（回车）键确认，弹出命令提示符界面。然后输入“sc delete MySQL服务名”,按“Enter”（回车）键，就能彻底删除残余的MySQL服务了。

# MySQL的登录

服务的启动与停止

MySQL安装完毕之后，需要启动服务器进程，不然客户端无法连接数据库。
在前面的配置过程中，已经将MySQL安装为Windows服务，并且勾选当Windows启动、停止时，MySQL也自动启动、停止。

方式1：使用图形界面工具

步骤1：打开windows服务

- 方式1：计算机（点击鼠标右键）→ 管理（点击）→ 服务和应用程序（点击）→ 服务（点击）
- 方式2：控制面板（点击）→ 系统和安全（点击）→ 管理工具（点击）→ 服务（点击）
- 方式3：任务栏（点击鼠标右键）→ 启动任务管理器（点击）→ 服务（点击）
- 方式4：单击【开始】菜单，在搜索框中输入“services.msc”，按Enter键确认

步骤2：找到MySQL80（点击鼠标右键）→ 启动或停止（点击）

![](D:\mine\study\mysql\pic\34.png)

方式2：使用命令行工具

启动 MySQL 服务命令：

```
net start MySQL服务名
```

停止 MySQL 服务命令：

```
net stop MySQL服务名
```

![](D:\mine\study\mysql\pic\35.png)

说明：
1. start和stop后面的服务名应与之前配置时指定的服务名一致。
2. 如果当你输入命令后，提示“拒绝服务”，请以 系统管理员身份 打开命令提示符界面重新尝试。

# 自带客户端的登录与退出

当MySQL服务启动完成后，便可以通过客户端来登录MySQL数据库。注意：确认服务是开启的。
登录方式1：MySQL自带客户端
开始菜单 → 所有程序 → MySQL → MySQL 8.0 Command Line Client

![](D:\mine\study\mysql\pic\36.png)

说明：仅限于root用户

登录方式2：windows命令行

格式：

mysql -h 主机名 -P 端口号 -u 用户名 -p密码

举例：

```mysql
mysql -h localhost -P 3306 -uroot -p123456 # 这里root用户的密码是123456
```

![](D:\mine\study\mysql\pic\37.png)

注意：

（1）-p与密码之间不能有空格，其他参数名与参数值之间可以有空格也可以没有空格。如：

```mysql
mysql -hlocalhost -P3306 -uroot -p123456
```

（2）密码建议在下一行输入，保证安全

```mysql
mysql -h localhost -P 3306 -u root -p
Enter password:****
```

（3）客户端和服务器在同一台机器上，所以输入localhost或者IP地址127.0.0.1。同时，因为是连接本
机： -hlocalhost就可以省略，如果端口号没有修改：-P3306也可以省略

简写成：

```mysql
mysql -u root -p
Enter password:****
```

连接成功后，有关于MySQL Server服务版本的信息，还有第几次连接的id标识。
也可以在命令行通过以下方式获取MySQL Server服务版本的信息：

```mysql
c:\> mysql -V

c:\> mysql --version
```

或登录后，通过以下方式查看当前版本信息：

```mysql
mysql> select version();

#退出登录
exit 或 quit

#查看所有的数据库
show databases;
```

- “information_schema”是 MySQL 系统自带的数据库，主要保存 MySQL 数据库服务器的系统信息，比如数据库的名称、数据表的名称、字段名称、存取权限、数据文件 所在的文件夹和系统使用的文件夹，等等
- “performance_schema”是 MySQL 系统自带的数据库，可以用来监控 MySQL 的各类性能指标。
- “sys”数据库是 MySQL 系统自带的数据库，主要作用是以一种更容易被理解的方式展示 MySQL 数据库服务器的各类性能指标，帮助系统管理员和开发人员监控 MySQL 的技术性能。
- “mysql”数据库保存了 MySQL 数据库服务器运行时需要的系统信息，比如数据文件夹、当前使用的字符集、约束检查信息，等等

添加一条记录报错

```mysql
mysql> insert into student values(1,'张三');
ERROR 1366 (HY000): Incorrect string value: '\xD5\xC5\xC8\xFD' for column 'name' at row 1

mysql> insert into student values(2,'李四');
ERROR 1366 (HY000): Incorrect string value: '\xC0\xEE\xCB\xC4' for column 'name' at
row 1

show create table 表名称\G

#查看student表的详细创建信息
show create table student\G

#结果如下
*************************** 1. row ***************************
Table: student
Create Table: CREATE TABLE `student` (
`id` int(11) DEFAULT NULL,
`name` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)
```

上面的结果显示student的表格的默认字符集是“latin1”不支持中文。

查看数据库的创建信息

```mysql
show create database 数据库名\G
#查看atguigudb数据库的详细创建信息
show create database atguigudb\G

#结果如下
*************************** 1. row ***************************
Database: atguigudb
Create Database: CREATE DATABASE `atguigudb` /*!40100 DEFAULT CHARACTER SET latin1 */
1 row in set (0.00 sec)
```

上面的结果显示atguigudb数据库也不支持中文，字符集默认是latin1。

步骤1：查看编码命令

```mysql
show variables like 'character_%';
show variables like 'collation_%';
```

步骤2：修改mysql的数据目录下的my.ini配置文件

[mysql] #大概在63行左右，在其下添加
...
default-character-set=utf8 #默认字符集
[mysqld] # 大概在76行左右，在其下添加
...
character-set-server=utf8
collation-server=utf8_general_ci

注意：建议修改配置文件使用notepad++等高级文本编辑器，使用记事本等软件打开修改后可能会
导致文件编码修改为“含BOM头”的编码，从而服务重启失败。

步骤3：重启服务
步骤4：查看编码命令

```mysql
show variables like 'character_%';
show variables like 'collation_%';
```

![](D:\mine\study\mysql\pic\38.png)

![](D:\mine\study\mysql\pic\39.png)

如果是以上配置就说明对了。接着我们就可以新创建数据库、新创建数据表，接着添加包含中文的
数据了。

MySQL8.0中
在MySQL 8.0版本之前，默认字符集为latin1，utf8字符集指向的是utf8mb3。网站开发人员在数据库设计
的时候往往会将编码修改为utf8字符集。如果遗忘修改默认的编码，就会出现乱码的问题。从MySQL 8.0
开始，数据库的默认编码改为 utf8mb4 ，从而避免了上述的乱码问题。

# MySQL图形化管理工具

工具1. MySQL Workbench

工具2. Navicat

工具3. SQLyog

工具4：dbeaver

可能出现连接问题：

有些图形界面工具，特别是旧版本的图形界面工具，在连接MySQL8时出现“Authentication plugin 'caching_sha2_password' cannot be loaded”错误。

![](D:\mine\study\mysql\pic\40.png)

出现这个原因是MySQL8之前的版本中加密规则是mysql_native_password，而在MySQL8之后，加密规则是caching_sha2_password。解决问题方法有两种，

第一种是升级图形界面工具版本，第二种是把MySQL8用户登录密码加密规则还原成mysql_native_password。

第二种解决方案如下，用命令行登录MySQL数据库之后，执行如下命令修改用户密码加密规则并更新用户密码，这里修改用户名为“root@localhost”的用户密码规则为“mysql_native_password”，密码值为“123456”，如图所示。

```mysql
#使用mysql数据库
USE mysql;
```

```mysql
#修改'root'@'localhost'用户的密码规则和密码
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'abc123';
```

```mysql
#刷新权限
FLUSH PRIVILEGES;
```

