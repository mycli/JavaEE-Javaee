# JavaEE-Javaee
## 深圳黑马JavaEE第104期基础班+就业班+高手课件
# 企业级分布式数据库架构实践



## 概述cmL46679910

大型网站的系统架构并不是从一开始就具备高性能、高可用、高伸缩等特性的。小型创业公司通常不需要设计非常复杂的系统架构，能将基本的业务跑起来就行。

随着用户和业务量的增加，系统架构需要根据具体情况重新设计，这其中就包括容易出现瓶颈的数据库服务器。

针对数据库架构的优化要根据业务特点、以及实施的成本逐步进行，通常不会一上来就设计出一个非常复杂的架构。cmL46679910

数据库的优化可以从这些方面进行

- SQL 优化
- 索引优化
- 缓存系统
- 主从复制、读写分离
- 垂直拆分
- 水平拆分

![aef2eeed69ca3cbc241eab9c52b6497.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66816421a52f4839a5e6f3abf2d5a68b~tplv-k3u1fbpfcp-watermark.image?)


## 数据库扩展思想

- 热备

  > 在数据库服务器运行过程中对数据进行备份操作。相对的是冷备，冷备份需要停机操作。

- 多活

  > 多个数据库服务器，保证高可用，避免单点故障。

- 故障切换

  > 当一台数据库服务器出现异常，自动切换到其他数据库服务器继续提供服务。

- 读写分离

  > 数据库的读写操作分发到不同的服务器，提高数据处理能力。

- 负载均衡

  > 负载均衡一般是建立的读写分离的基础之上，将读写操作根据情况，合理的分摊到数据库服务器，提高并发能力，同时避免过载。





## 主从复制

### 概述

主从复制，是通过部署多台数据库服务器，这些数据库之间有主从关系。其中，主数据库用于提供服务，从数据库中的数据和主数据库是保持一致的，这样做的好处是能够热备份，同时，可以在此基础上扩展读写分离的架构。

主从复制的架构一般分为：一主一从、一主多从、主主复制、级联复制、主主与级联复制的结合。



<img src=".\images\企业级分布式数据库架构实践\image-20201110105534346.png" alt="image-20201110105534346" style="zoom:50%;" />



这是一个典型的一主多从读写分离架构，应用系统写数据时，会写入到 master  节点，然后再由 master 节点将数据复制到 slave 节点中。这个架构仍有不足，例如 master 节点存在单点、数据的复制存在延迟。



### 单主单从cmL46679910

演示单主单从的主从复制



1、在 Docker 中创建两个数据库容器

```shell
# 创建数据库容器挂载目录
mkdir -p /usr/local/mysql/3306/conf
mkdir -p /usr/local/mysql/3306/data
mkdir -p /usr/local/mysql/3307/conf
mkdir -p /usr/local/mysql/3307/data

# 创建一个临时容器
docker run -d --name temp -e MYSQL_ROOT_PASSWORD=123456 mysql:5.5

# 复制配置文件
docker cp temp:/etc/mysql/my.cnf /usr/local/mysql/3306/conf/my.cnf
docker cp temp:/etc/mysql/my.cnf /usr/local/mysql/3307/conf/my.cnf

# 删除临时容器
docker rm -f temp

##################################

# 创建容器
docker run -d --name master3306 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -v /usr/local/mysql/3306/conf/my.cnf:/etc/mysql/my.cnf -v /usr/local/mysql/3306/data/:/var/lib/mysql/ mysql:5.5 --character-set-server=utf8
docker run -d --name slave3307 -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123456 -v /usr/local/mysql/3307/conf/my.cnf:/etc/mysql/my.cnf -v /usr/local/mysql/3307/data/:/var/lib/mysql/ mysql:5.5 --character-set-server=utf8
```



2、修改配置文件，添加内容

`3306/conf/my.cnf`

```properties
server-id=1
log-bin=binlog
binlog-do-db=company
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
binlog-ignore-db=performance_schema
binlog-format=STATEMENT
```



`3307/conf/my.cnf`

```properties
server-id=2
relay-log=relaylog
replicate-do-db=company
replicate-ignore-db=mysql
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
```



3、重启 master 和 slave

```shell
docker restart master3306
docker restart slave3307
```

4、在 master 上创建用户并授权

```sql
-- 创建用户
create user 'slave'@'172.17.0.3' identified by '123456';

-- 授权
grant all privileges on *.* to 'slave'@'172.17.0.3' with grant option;

-- 刷新权限
flush privileges;
```



5、查看 master 状态信息

```sql
show master status;
```



5、在 slave 上更新 master 的信息

```sql
-- 关闭 slave
stop slave;

-- 更新 master 的信息
change master to master_host='172.17.0.2',master_user='slave',master_password='123456',master_log_file='binlog.000001',master_log_pos=440;

-- 开启 slave
start slave;

-- 查看 slave 信息
show slave status\G;
```

<img src=".\images\企业级分布式数据库架构实践\image-20201110112056644.png" alt="image-20201110112056644" style="zoom:50%;" />



IO Thread 和 SQL Thread 都处于正常运行状态，表示 slave 已准备就绪。



7、测试

- 在 master 中创建数据库、数据表、插入数据，slave 会同步
- 但在 slave 中插入数据，不会同步到 master 中，因为现在是单向的。



### 主从复制的原理

在 MySQL 中有两种复制机制，分别是异步复制和半同步复制，默认情况下采用的是异步复制。



异步复制



流程

1. 应用程序提交事务到 master
2. master 收到事务的提交请求后，会更新内部的 binlog 日志，接着让 mysql 引擎执行事务操作，并返回给客户端执行结果。同时，master 中还有一个事件监听，监听着 binlog 日志的变化，一旦发生更新，则会触发 dump 线程。
3. dump 线程被触发后，通知 slave 中的 I/O 线程。
4. slave 中的 I/O 线程收到通知后，会从 relay-log.info 文件中获取 binlog 日志文件和 pos 位置信息，并发送给 master 的 dump 线程。
5. master 的 dump 线程收到信息后，会将日志文件的 pos 位置之后的所有日志都同步给 slave 的 I/O 线程。
6. slave 的 I/O 线程接受到同步过来的日志后，会将日志记录到 relaybin 中继日志中。
7. SQL 线程监听到 relaybin 发生更改，会将新的日志读取出来并重做数据【异步操作】
8. slave 将日志记录到 relaybin 之后，会返回一个 ACK 消息给 master。



对于这一系列的操作，可以发现 master 写入完 binlog 后，会立即提交事务并向客户端返回响应，而对于数据的同步，则是以异步的形式完成的。



这种方式的优点的响应速度快，缺点是可能会造成数据不一致（存在延迟）。





半同步复制



半同步复制和异步复制的工作流程大体相同，不同的是，master 在写入 binlog 日志之后，不会立即通过引擎提交事务，而是暂时挂起等待，等 slave 响应的 ACK 之后，再唤醒等待，继续执行。



半同步复制的优点是保证数据的一致性，但性能会降低。



注意：半同步复制时如果等待没有被唤醒，它不会一直等待下去，默认的等待时间是10秒钟，但该时间是可以配置的。10秒钟过后，会自动唤醒并提交事务，同时自动的切换到异步复制，待 slave 节点恢复后又会重新切换到半同步复制。



### 主从复制相关文件介绍

master 节点

执行过的更新都记录在这些日志文件中。



slave 节点


### 异步复制演示cmL46679910

关闭 slave 节点，向 master 发送数据更改，能够立即得到响应，说明 master 不会等待 slave 的 ACK 确认。



### 半同步复制演示

配置

1、进入 master 容器，安装半同步复制插件。

```
install plugin rpl_semi_sync_master soname 'semisync_master.so';
```



2、查看插件信息

```
show plugins;
```


3、启用插件并设置等待超时时间

```
set global rpl_semi_sync_master_enabled=1; #1：启用，0：禁止
set global rpl_semi_sync_master_timeout=10000; #单位是毫秒
```



插件启用成功后，master 会打印相应的日志信息

<img src=".\images\企业级分布式数据库架构实践\image-20201110144059010.png" alt="image-20201110144059010" style="zoom:50%;" />



4、进入 slave 容器安装半同步插件

```sql
install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
```



5、启用 slave 的半同步插件

```sql
set global rpl_semi_sync_slave_enabled=1; #1：启用，0：禁止
```



6、重启 slave 的 I/O Thread

```sql
stop slave io_thread;
start slave io_thread;
```



7、到目前w为止，半同步复制以及配置完成，可以在 master 中查看半同步复制的状态

```sql
-- 查询状态信息
show global status like "%sync%";

-- 查询参数信息
show global variables like '%sync%';
```

8、演示

模拟 slave 故障，主动关闭 slave 的 I/O 线程

```
stop slave io_thread;
```
搭建

1、创建两个 master 容器

```shell
# 创建数据库容器挂载目录
mkdir -p /usr/local/mysql/3308/conf
mkdir -p /usr/local/mysql/3308/data
mkdir -p /usr/local/mysql/3309/conf
mkdir -p /usr/local/mysql/3309/data

# 创建一个临时容器
docker run -d --name temp -e MYSQL_ROOT_PASSWORD=123456 mysql:5.5

# 复制配置文件
docker cp temp:/etc/mysql/my.cnf /usr/local/mysql/3308/conf/my.cnf
docker cp temp:/etc/mysql/my.cnf /usr/local/mysql/3309/conf/my.cnf

# 删除临时容器
docker rm -f temp

##################################

# 创建容器
docker run -d --name master3308 -p 3308:3306 -e MYSQL_ROOT_PASSWORD=123456 -v /usr/local/mysql/3308/conf/my.cnf:/etc/mysql/my.cnf -v /usr/local/mysql/3308/data/:/var/lib/mysql/ mysql:5.5 --character-set-server=utf8
docker run -d --name master3309 -p 3309:3306 -e MYSQL_ROOT_PASSWORD=123456 -v /usr/local/mysql/3309/conf/my.cnf:/etc/mysql/my.cnf -v /usr/local/mysql/3309/data/:/var/lib/mysql/ mysql:5.5 --character-set-server=utf8
```



2、修改配置文件，添加内容

`3308/conf/my.cnf`

```properties
server-id=1
log-bin=binlog
binlog-do-db=company
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
binlog-ignore-db=performance_schema
binlog-format=STATEMENT
relay-log=relaylog
```



`3309/conf/my.cnf`

```properties
server-id=2
log-bin=binlog
binlog-do-db=company
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
binlog-ignore-db=performance_schema
binlog-format=STATEMENT
relay-log=relaylog
```



3、重启两台 master 服务器

```
docker restart master3308
docker restart master3309
```



4、为两台 master 创建账户并授权

```sql
-- master3308
create user 'master'@'172.17.0.3' identified by '123456';
grant all privileges on *.* to 'master'@'172.17.0.3' with grant option;
flush privileges;

-- master3309
create user 'master'@'172.17.0.2' identified by '123456';
grant all privileges on *.* to 'master'@'172.17.0.2' with grant option;
flush privileges;
```


![bedf40859ce5fc66331f6ac439457a6.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66082ca30f45494aac17ae6358087fb8~tplv-k3u1fbpfcp-watermark.image?)

5、更新 master 信息

```sql
-- master3308
stop slave;
change master to master_host='172.17.0.3',master_user='master',master_password='123456',master_log_file='binlog.000001',master_log_pos=434;
start slave;
show slave status;

-- master3309
stop slave;
change master to master_host='172.17.0.2',master_user='master',master_password='123456',master_log_file='binlog.000001',master_log_pos=434;
start slave;
show slave status;
```



6、测试

- master3308 创建数据库和表
- master3309 插入数据

观察两台 master 是否能相互复制。



### 级联复制

要想提高读的并发能力，并且还要减少主从复制带来的性能损耗，可以采用级联复制的架构。

<img src=".\images\企业级分布式数据库架构实践\image-20201110154116342.png" alt="image-20201110154116342" style="zoom:50%;" />



级联复制的写请求入口仍然之后一个，但当 master 向 slave 进行复制时，对于 slave 可以分为多层，master 只需要向其中两台 slave 复制即可，然后再由这两台 slave 复制给更多的 slave。


需要看更多的资源加微信：
![f7a4195570f1388419dac8d7bfaa860](https://user-images.githubusercontent.com/41461298/155635002-6cc03361-1022-4d78-a447-311bf23aec5f.jpg)



