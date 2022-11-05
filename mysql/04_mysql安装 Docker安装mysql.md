> 参考来源：
>
> https://juejin.cn/post/6872210647287857165
>
> https://segmentfault.com/a/1190000039147983

### 1.  拉取镜像 / 查看镜像

```shell
docker pull mysql
docker images
```



### 2. 创建测试容器实例并启动

```shell
docker run -p 3306:3306 --name mysqltest -e MYSQL_ROOT_PASSWORD=root -d mysql
```



### 3. 进入 MySql容器 (可选)

```shell
docker exec -it mysqltest bash
```



### 4.创建本地路径并挂载 Docker 内数据

```shell
mkdir -p /mine/mysql/conf
mkdir -p /mine/mysql/data
docker cp mysqltest:/etc/mysql/my.cnf /mine/mysql/conf
docker cp mysqltest:/var/lib/mysql/ /mine/mysql/data
```



### 5. 停止并删除测试容器，创建新的 docker 容器并启动

```shell
docker stop mysqltest
docker rm mysqltest

docker run -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=root -v /mine/mysql/conf/my.cnf:/etc/mysql/my.cnf -v /mine/mysql/data/mysql:/var/lib/mysql -d mysql
```



### **报错**

sqlyog / navicat 连接 mysql **报错**：**不支持 caching_sha_password 加密方式**

1.进入容器

```shell
docker exec -it mysql bash
```

2.登录 Mysql

```shell
mysql -uroot -p
```

3.查看并选择数据库

```shell
show databases;
use mysql;
```

4.修改加密方式并退出 Mysql 和 Mysql 容器

```shell
select host,user,plugin from user;
alter user 'root'@'%' identified with mysql_native_password by 'root';
exit;
exit;
```









