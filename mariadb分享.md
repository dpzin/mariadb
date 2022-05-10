# mariadb 分享



![2015918143640534.png](https://img.jbzj.com/file_images/article/201509/2015918143640534.png)



## 背景

数据库可以说是伴随我们开发者整个职业生涯，整个数据库的知识体系是非常庞大。除了我们日常进行模型设计和使用还包括：监控，故障管理，容量管理，性能优化，安全保障，自动部署，集群管理等等。

此次分享打算从以下几个方面进行介绍

        1. mariadb的背景和产品介绍
        2. 如何搭建mariadb server的集群服务
        3. 和mysql比较
        4. 基础篇（基础架构，日志系统，事务，索引，锁）
        5. 设计篇  (类型选择，索引设计)
        6. 工具篇（mysqldump, XtrBackup,mysqlslap,mysqlbinlog）



## mariadb产生的背景

mysql创始人Widenius早期以10亿美金的价格将自己的公司Monty Program AB卖给了SUN，此后，随着SUN被甲骨文收购，MySQL的所有权也落入Oracle的手中。

Widenius离开sun公司之后，决定fork一条分支并且以自己女儿的名字maria命名。

![image-20220426102810788](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220426102810788.png)

优势:mariaDB迭代版本相对比较快，新的feature和bugfix响应会比较快。会参考好的开源产品的功能，比如

引入perconaServer的XtraDB引擎。

劣势：使用文档并不完善，只对基础功能和对自己特有的功能进行了介绍。其他的文档要参考mysql管方的，以及如果使用XtraDB的存储引擎 需要参考perconaServer的文档。

## mariadb产品介绍

| 组件                    | 功能描述                            |
| ----------------------- | ----------------------------------- |
| mariadb server          | 数据库，社区版                      |
| mariaDB Gerlera cluster | mariaDB集群同步的方案，只支持InnoDB |
| mariaDB Enterprise Server | 数据库，企业版                                  |
| mariaDB maxscale          | 数据库代理，故障转移，可用来管理数据库服务器集群    |
| mariaDB columnStrore      | 列存储执行引擎                                    |
| MariaDB Xpand             | 分布式SQL引擎，高可用性，容错，写扩展和水平向外扩展 |

![image-20220507163837502](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220507163837502.png)





## 安装

官方setup：curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash

yum安装：sudo yum install -y MariaDB-server galera-4 MariaDB-client

启动： systemctl start mariadb

初始用户密码：ALTER USER root@localhost IDENTIFIED VIA mysql_native_password USING PASSWORD("Huayun@123");



## 集群搭建

MariaDB Galera Cluster is a [virtually synchronous](https://mariadb.com/kb/en/about-galera-replication/) multi-primary cluster for MariaDB. It is available on Linux only, and only supports the [InnoDB](https://mariadb.com/kb/en/innodb/) storage engine (although there is experimental support for [MyISAM](https://mariadb.com/kb/en/myisam/) and, from [MariaDB 10.6](https://mariadb.com/kb/en/what-is-mariadb-106/), [Aria](https://mariadb.com/kb/en/aria/). See the [wsrep_replicate_myisam](https://mariadb.com/kb/en/galera-cluster-system-variables/#wsrep_replicate_myisam) system variable, or, from [MariaDB 10.6](https://mariadb.com/kb/en/what-is-mariadb-106/), the [wsrep_mode](https://mariadb.com/kb/en/galera-cluster-system-variables/#wsrep_mode) system variable).







## oracle mysql 对比

### 使用用户

使用MySQL的有Facebook、Github、YouTube、Twitter、PayPal、诺基亚、Spotify、Netflix等。

使用MariaDB的有Redhat、DBS、Suse、Ubuntu、1＆1、Ingenico等。

###  相同点

mariadb和mysql大部分功能都是相同的，他们之间也有很好的兼容性。mariadb可以直接使用mysql的驱动程序。

### 不同点

mariadb支持的更多的存储引擎：

![image-20220426110134715](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220426110134715.png)

功能比对：

![image-20220426105632043](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220426105632043.png)



![image-20220426105700822](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220426105700822.png)

官方提供的功能比较：https://mariadb.com/kb/zh-cn/mariadb-mariadbmysql/  

官方提供的性能基准测试比较： https://mariadb.com/resources/blog/mariadb-5-3-optimizer-benchmark/

## 基础篇

### 基础架构

####  架构图



![image-20220505101013470](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220505101013470.png)



#### 各个组件

**连接器**：show processlist 查看当前数据库连接



**查询缓存**：总体来说弊大于利，mysql8.0以后不支持缓存，mariadb支持   show VARIABLES where VARIABLE_name like '%query%'



**分析器**：词法分析，语法分析



**优化器**:  执行计划，索引选择



**执行器**：调用存储引擎的接口或者数据



#### 案例一

​     查询如果存在索引的情况下



![image-20220507165244533](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220507165244533.png)



#### 案例二

​     更新一条数据

![image-20220507170527356](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220507170527356.png)



### 事务

提到事务，你肯定会想到 ACID（Atomicity、Consistency、Isolation、Durability，即原子性、一致性、隔离性、持久性）

#### 隔离级别

1.  读未提交 （read uncommitted）
2.  读已提交 （read committed）
3.  可重复读（repeatable read）
4.  串行化 （serializable ）

![image-20220507180215166](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220507180215166.png)





### Innodb索引模型

#### 基础

InnoDB 使用了 B+ 树索引模型，所以数据都是存储在 B+ 树中的

CREATE TABLE `T` (
  `id` int(11) NOT NULL,
  `k` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `k` (`k`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

![image-20220509104349408](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220509104349408.png)



主键索引：如果语句是select * from T where ID = 500 ,即主键查询方式

普通索引：如果语句是select * from T where k=5 ，即普通索引查询方式，则需要先搜索k索引树，得到ID的值为500，再到ID索引树搜索一次。这个过程称为回表。

#### 优化手段

##### 覆盖索引：

​     索引覆盖了查询字段

##### 最左前缀原则

​    like 'xxx%'  这种是可以走索引查询

##### 索引下推

   联合索引 (network_id,subnet_id)  

​               where  network_id = 'xxx %' and subnet_id = "123" 

  没有索引下推的情况下： 根据联合索引找符合network_id的状况，再回表找到所有的对应记录，再比对subnet_id的值。

  索引下推的情况下：直接用索引来判断。



### 锁

#### 全局锁

让整个库处于只读状态

命令：flush tables with read lock

使用场景：全库逻辑备份

#### 表级锁

表级锁有两种：一种是表锁，一种是元数据锁

**表锁**

​       lock table ... read/write

**元数据锁**

​      DML语句会给每个表加上MDL读锁，DDL会给每个表加上MDL写锁。

不停机情况下安全的增加字段：

ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 

#### 行级锁

行锁是在需要的时候才加上的，事务结束时才会释放。

![image-20220509154604705](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220509154604705.png)

死锁和死锁检测

![image-20220509155118916](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220509155118916.png)

当出现死锁以后，有两种解决策略：

   1. 设置超时时间 innodb_lock_wait_timeout。超时之后回滚 默认是50s

   2. 死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务。让其他事务继续执行

      参数innodb_deadlock_detect = on。弊端：需要消耗大量CPU资源



## 设计篇

### 类型选择
#### 数字类型

![img](https://s4.51cto.com/images/blog/202108/27/9f3047ed0b7e2dfeadc6b07ed5ee1a3c.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

1.  FLOAT和DOUBLE不推荐使用，不是高精度类型，计算可能会有误差。
2. 账户金额相关设计时，可以考虑使用整型类型和单位的方式，而不是DECIMAL类型 这样性能更好。



#### 字符串类型

![image-20220509191619036](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220509191619036.png)

1.  推荐使用字符集UTF8MB4，可以存储emoji表情。
2.  默认字符的排序规则，ci代表大小写不敏感



![image-20220509191933490](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220509191933490.png)



#### 日期类型

![image-20220509192836350](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220509192836350.png)



1.   timestamp 优势：存储空间小，附带时区属性  劣势：千年虫问题；时区转换计算，性能不如datetime。
2.   timestamp 有千年虫的问题，



#### JSON类型

1.  适合存储修改比较少的，相对静态的数据
2.  官方提供了很多查询的函数，也可以利用函数索引，针对json格式里面的某一个Key值构建索引。



#### 表结构设计

1.  自增主键和uuid主键选择。 自增主键优点：1. 插入是顺序的，有利于范围查询 。2. 占用空间小。避免很多页分裂的情况。  缺点：1. 高并发和分布式场景下会有性能问题。2. 数据不安全。   建议： 核心业务表使用UUID ，非核心业务表可以选择使用自增ID，对性能有要求的数据可以选择业务+排序的UUID

   ![image-20220510100024853](C:\Users\fei.gao\AppData\Roaming\Typora\typora-user-images\image-20220510100024853.png)

   



## 备份

### 逻辑备份

mysqldump

案例：

​     备份单个数据库：mysqldump db_name > 备份文件.sql

​     恢复或者加载数据库：mysql db_name < 备份文件.sql

### 物理备份

[Mariabackup](https://mariadb.com/kb/en/mariabackup/)

mysqlhotcopy



### 日志系统

innodb : redo log ，bin log , undo log , relay log

#### 系统问题

show VARIABLES like '%flush_log%'   系统发生崩坏的时候，可能会导致1s的数据丢失。


