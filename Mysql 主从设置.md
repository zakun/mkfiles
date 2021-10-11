## MySQL中怎么设置主从同步

MySQL中怎么设置主从同步，相信很多没有经验的人对此束手无策，为此本文总结了问题出现的原因和解决方法，通过这篇文章希望你能解决这个问题。

### 1. 配置主库my.ini
	port=3306
	datadir="C:/Program Files/MySQL/MySQL Server 5.0/Data/"
	server-id=1
	log-bin=mysql-bin.log
	log-slave-updates
 
### 2. 配置从库my2.ini
	port=3307
	datadir="C:/Program Files/MySQL/MySQL Server 5.0/Data2/"
	server-id=2
	log-bin=mysql-bin.log
	#启用从库日志，这样可以进行链式复制
	log-slave-updates
	#从库是否只读，0表示可读写，1表示只读
	read-only=1
	#只复制某个表
	#replicate-do-table=tablename
	#只复制某些表（可用匹配符）
	#replicate-wild-do-table=tablename%
	#只复制某个库（如果对多个数据库做同步，那么可以用多行来表示。）
	replicate-do-db = backup
	#只复制某些库
	#replicte-wild-do-db=dbname%
	#不复制某个表
	#replicate-ignore-table=tablename
	#不复制某些表
	#replicate-wild-ignore-table=tablename%
	#不复制某个库（如果忽略多个数据库的同步，那么可以用多行表示。）
	replicate-ignore-db=mysql
	#复制完的sql语句是否立即从中继日志中清除，1表示立即清除
	relay-log-purge = 1
	#从服务器主机，用于show slave hosts生成从库清单
	report-host=slave-1
	#即不管发生什么错误，镜像处理工作也继续进行
	slave-skip-errors=all
	#每经过n次日志写操作就把日志文件写入硬盘一次(对日志信息进行一次同步)。n=1是最安全的做法，但效率最低。
	#默认设置是n=0，意思是由操作系统来负责二进制日志文件的同步工作。
	sync_binlog=1
 
### 3. 设置主库
#### 3.1 启动主库：
`mysqld-nt --defaults-file=my.ini`

#### 3.2 创建用户
连接到主库中，创建复制用户
	C:/>mysql -uroot -ppassword -P3306
	mysql> grant replication slave on *.* to  identified by '123456';
	Query OK, 0 rows affected (0.00 sec)
 
#### 3.3 锁住主库
锁住主库的table，以便备份数据文件到从库进行初始化
	>flush tables with read lock;
	Query OK, 0 rows affected (0.00 sec)
#### 3.4 记录二进制日志文件名和位置
显示主库状态，注意记下当前二进制日志文件名和position

mysql>show master status;
+-----------------------+-----------+-------------------+------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+----------------------+------------+-------------------+------------------------+
| mysql-bin.000001 |      98 |  backup      |    mysql        |
+----------------------+------------+-------------------+--------------------------+
1 row in set (0.00 sec)

#### 3.5 同步初始数据
复制主库初始数据到从库, 将C:/Program Files/MySQL/MySQL Server 5.0/Data/下的内容打包复制到C:/Program Files/MySQL/MySQL Server 5.0/Data2/下，执行从库的初始化。当然，初始化也可以使用mysqldump来完成。
 
### 4. 设置从库
另外开启一个cmd，启动从库
`mysqld-nt --defaults-file=my2.ini`
#### 4.1 连接到从库进行配置
	C:/>mysql -uroot -ppassword -P3307
	mysql> CHANGE MASTER TO
	    -> MASTER_HOST='localhost',
	    -> MASTER_USER='backup',
	    -> MASTER_PASSWORD='backup',
	    -> MASTER_LOG_FILE='mysql-bin.000001',
	    -> MASTER_LOG_POS=98;
	Query OK, 0 rows affected (0.01 sec)
注意到这里`master_log_file`和`master_log_pos`就是前面`show master status`的结果。
#### 4.2 启动复制进程
	mysql> start slave;
	Query OK, 0 rows affected (0.00 sec)
#### 4.3 解锁
至此配置基本完成，在主库解开table的锁定
	mysql> unlock tables;
	Query OK, 0 rows affected (0.00 sec)