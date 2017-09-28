### 主从复制用途以及条件

	实时灾备，用于故障切换
	读写分离，提供查询服务
	备份，避免影响业务

### 主从部署的必要条件

	主库开启binlog日志（设置log-bin参数）
	主从server-id不同
	从库服务器能连同主库

### 主从复制原理

	从库生成两个线程，一个i/o线程，一个SQL线程；

	i/o线程去请求主库的binlog，并且得到的binlog日志写道relay log(中继日志)文件中，

	主库会生成一个log dump线程，用来给从库的i/o线程传binlog；

	SQL线程，会读取中继日志文件，并解析成具体的操作执行，这样主从的操作就一致了，而最终的数据也就一直了。


### 主从复制存在的问题以及解决办法

	主库宕机之后，数据可能会丢失
	从库只有一个sql Thread，主库写压力大，复制很可能延时


解决方法：

	半同步复制解决--解决数据丢失的问题
	并行复制--解决从库复制延时的问题

### 1. mysql 的 AB主从复制

实验环境:

	rhel6.5.x86_64
	master : 172.25.5.2
	slave1 : 172.25.5.3
	mysql5.1.71


注: mysql 数据库的版本,两个数据库版本要相同,或者 slave 比 master 版本高!

**master配置**

	vim /etc/my.cnf

	[mysqld]
	datadir=/var/lib/mysql
	socket=/var/lib/mysql/mysql.sock
	user=mysql
	# Disabling symbolic-links is recommended to prevent assorted security risks
	symbolic-links=0
	log-bin=mysql-bin ###启动二进制日志系统
	binlog-do-db=lockey ####二进制需要同步的数据库名,如果需要同步多个库,例如要再同步halo库,再添加一行“binlog-do-db=halo”,以此类推
	server-id=1 ####必须为 1 到 232–1 之间的一个正整数值
	binlog-ignore-db=mysql ####禁止同步 mysql 数据库


	[mysqld_safe]
	log-error=/var/log/mysqld.log
	pid-file=/var/run/mysqld/mysqld.pid

配置完成后重新启动mysqld服务然后连接到数据库

创建同步帐户,并给予权限

	mysql> grant replication slave on *.* to ha@'172.25.5.%' identified by 'halo';
	Query OK, 0 rows affected (0.00 sec)

	mysql> Flush privileges;
	Query OK, 0 rows affected (0.00 sec)

	mysql> show master status;
	+------------------+----------+--------------+------------------+
	| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
	+------------------+----------+--------------+------------------+
	| mysql-bin.000002 |      326 | lockey       | mysql            |
	+------------------+----------+--------------+------------------+
	1 row in set (0.00 sec)##记录 File 和 Position 的值,配置slave时会用到。

**slave 配置**

vim /etc/my.cnf

只需要在[mysqld]下面添加server-id即可

	server-id=2
	#从服务器 ID 号,不要和主 ID 相同,如果设置多个从服务器,每个从服务器必
	须有一个唯一的 server-id 值,必须与主服务器的以及其它从服务器的不相同。
	可以认为 server-id 值类似于 IP 地址:这些 ID 值能唯一识别复制服务器群集
	中的每个服务器实例。

配置完成后重新启动mysqld服务然后连接到数据库

	mysql> show slave status;
	Empty set (0.00 sec)

	mysql> change master to master_host='172.25.5.2',master_user='ha',master_password='halo',master_log_file='mysql-bin.000002',master_log_pos=326;
	Query OK, 0 rows affected (0.05 sec)

	mysql> slave start;
	Query OK, 0 rows affected (0.00 sec)

	mysql> show slave status\G;
	*************************** 1. row ***************************
	               Slave_IO_State: Waiting for master to send event
	                  Master_Host: 172.25.5.2######
	                  Master_User: ha##########
	                  Master_Port: 3306
	                Connect_Retry: 60
	              Master_Log_File: mysql-bin.000002###
	          Read_Master_Log_Pos: 326#####
	               Relay_Log_File: mysqld-relay-bin.000002
	                Relay_Log_Pos: 251
	        Relay_Master_Log_File: mysql-bin.000002#####
	             Slave_IO_Running: Yes##########
	            Slave_SQL_Running: Yes##########
	              Replicate_Do_DB:
	          Replicate_Ignore_DB:
	           Replicate_Do_Table:
	       Replicate_Ignore_Table:
	      Replicate_Wild_Do_Table:
	  Replicate_Wild_Ignore_Table:
	                   Last_Errno: 0
	                   Last_Error:
	                 Skip_Counter: 0
	          Exec_Master_Log_Pos: 326#####
	              Relay_Log_Space: 407
	              Until_Condition: None
	               Until_Log_File:
	                Until_Log_Pos: 0
	           Master_SSL_Allowed: No
	           Master_SSL_CA_File:
	           Master_SSL_CA_Path:
	              Master_SSL_Cert:
	            Master_SSL_Cipher:
	               Master_SSL_Key:
	        Seconds_Behind_Master: 0
	Master_SSL_Verify_Server_Cert: No
	                Last_IO_Errno: 0
	                Last_IO_Error:
	               Last_SQL_Errno: 0
	               Last_SQL_Error:
	1 row in set (0.00 sec)

**注意：slave上要开启read_only，不然可能会因为不小心的操作导致数据不一致以及主从复制出现问题！！！**

**测试数据同步**

**在master上进行数据库操作**

	mysql> use lockey;
	Database changed
	mysql> create table users(username varchar(20) not null,password varchar(20) not null);
	Query OK, 0 rows affected (0.02 sec)

	mysql> insert into users values('lockey','root');
	Query OK, 1 row affected (0.00 sec)

	mysql> insert into users values('lockey1','root1');
	Query OK, 1 row affected (0.00 sec)

**在slave上边验证**

	mysql> use lockey;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A

	Database changed
	mysql> show tables;
	+------------------+
	| Tables_in_lockey |
	+------------------+
	| users            |
	+------------------+
	1 row in set (0.00 sec)

	mysql> select * from users;
	+----------+----------+
	| username | password |
	+----------+----------+
	| lockey   | root     |
	| lockey1  | root1    |
	+----------+----------+
	2 rows in set (0.00 sec)

两边数据是一致的，表示主从复制成功！！！

注意：如果开启主从复制之前两边的数据不一致的话需要拷贝二进制日志文件，要确保开启主从复制之前两边数据是一致的

### 2. Mysql-5.7.19 基于GTID的主从复制

实验环境与上例基本一致，只是Mysql的版本换成了5.7，可以从官网下载以下格式的安装包然后提取文件进行安装：

mysql-5.7.19-1.el6.x86_64.rpm-bundle.tar
tar -xf mysql-5.7.19-1.el6.x86_64.rpm-bundle.tar

	yum install mysql-community-common-5.7.19-1.el6.x86_64.rpm mysql-community-libs-5.7.19-1.el6.x86_64.rpm mysql-community-client-5.7.19-1.el6.x86_64.rpm mysql-community-server-5.7.19-1.el6.x86_64.rpm mysql-community-libs-compat-5.7.19-1.el6.x86_64.rpm -y

如果是在前面的两台主机上继续接着实验，请先卸载前面安装的mysql并且删除数据目录，再执行上边的yum install命令，安装完成之后进行以下配置：

**master && slave**
前一步的配置不变，接着添加以下内容即可：

vim /etc/my.cnf

	gtid-mode=on
	enforce-gtid-consistency=1

并且开启两台主机的mysql服务

	/etc/init.d/mysqld start

初次启动一般来说登录之后是不能进行数据库操作的，需要首先更改密码的，根据提示进行操作即可，密码如果没有在terminal中显示出来，可以在日志文件中过滤查找，是一串奇怪的字符，通过初始密码登录之后修改密码如下：
![这里写图片描述](http://img.blog.csdn.net/20170928204654020?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20170928204627335?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

	mysql> alter user root@localhost identified by 'Lockey+123';

**在master上进行以下操作：**

	##创建同步帐户,并给予权限
	mysql> grant replication slave on *.* to ha@'172.25.5.%' identified by 'Lockey+123';
	Query OK, 0 rows affected (0.00 sec)

	mysql> Flush privileges;
	Query OK, 0 rows affected (0.00 sec)

	mysql> show master status;
	+------------------+----------+--------------+------------------+
	| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
	+------------------+----------+--------------+------------------+
	| mysql-bin.000002 |      326 | lockey       | mysql            |
	+------------------+----------+--------------+------------------+
	1 row in set (0.00 sec)
![这里写图片描述](http://img.blog.csdn.net/20170928204850221?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**然后在slave上进行以下操作：**

	mysql> change master to master_host='172.25.5.2',master_user='ha',master_password='Lockey+123',master_auto_position=1;


**测试：**

在master上进行数据库操作，然后在mysql数据库中使用以下命令查看结果：

	mysql>  select * from gtid_executed;
	mysql>  show master status

![这里写图片描述](http://img.blog.csdn.net/20170928205312467?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
然后在slave上的mysql数据库中使用以下命令查看结果：

	mysql> select * from gtid_executed;

![这里写图片描述](http://img.blog.csdn.net/20170928205336810?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

重起master之后再次在mysql数据库中使用以下命令查看结果：

	mysql> select * from gtid_executed;

![这里写图片描述](http://img.blog.csdn.net/20170928205351532?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 3.MySQL 5.7并行复制

**此配置只在slave上进行**

在进行以下配置之前首先可以通过一条命令看一下结果（得到的结果条目很少）：

mysql> show processlist；

然后配置vim /etc/my.cnf
 #添加以下内容（前面的配置继续保留）：

	slave-parallel-type=LOGICAL_CLOCK
	slave-parallel-workers=16
	master_info_repository=TABLE
	relay_log_info_repository=TABLE
	relay_log_recovery=ON

	master_info_repository解释

	开启MTS功能后，务必将参数master_info_repostitory设置为TABLE，这样性能可以有50%~80%的提升。这是因为并行复制开启后对于元master.info这个文件的更新将会大幅提升，资源的竞争也会变大。在之前InnoSQL的版本中，添加了参数来控制刷新master.info这个文件的频率，甚至可以不刷新这个文件。因为刷新这个文件是没有必要的，即根据master-info.log这个文件恢复本身就是不可靠的。在MySQL 5.7中，Inside君推荐将master_info_repository设置为TABLE，来减小这部分的开销。

	slave_parallel_workers解释

	若将slave_parallel_workers设置为0，则MySQL 5.7退化为原单线程复制，但将slave_parallel_workers设置为1，则SQL线程功能转化为coordinator线程，但是只有1个worker线程进行回放，也是单线程复制。然而，这两种性能却又有一些的区别，因为多了一次coordinator线程的转发，因此slave_parallel_workers=1的性能反而比0还要差

重启mysql服务，然后再次执行：

mysql> show processlist；
![这里写图片描述](http://img.blog.csdn.net/20170928205416262?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
结果增加了16条，就是配置slave-parallel-workers=16的作用，也就是开启了并行复制的结果。

### 4. mysql半同步
**半同步复制简介**

　　何为半同步复制模式呢？在此我们先了解异步复制模式，这是MySQL的默认复制选项。异步复制即是master数据库把binlog日志发送给slave数据库，然后就没有了然后了。在此暴露一个问题，当slave服务器发生故障了，那么肯定会导致主从数据库服务器的数据不一致。

　　为了解决上面的问题，MySQL5.5引入一种叫做半同步复制模式。开启这种模式，可以保证slave数据库接收完master数据库发送过来的binlog日志并写入自己的中继日志中，然后反馈给master数据库，告知已经复制完毕。

　　开启这种模式后，当出现超时，主数据库将会自动转为异步复制模式，直到至少有一台从服务器接受到主数据库的binlog，并且反馈给主数据库。这时主数据库才会切换回半同步复制模式。

mysql> show variables like 'have_dynamic_loading';
**确保value为YES**

![这里写图片描述](http://img.blog.csdn.net/20170928205611213?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


**注意：**
半同步复制模式必须在主服务器和从服务器同时中开启，否则将会默认为异步复制模式。

半同步复制需要安装插件，而插件的位置如下：
![这里写图片描述](http://img.blog.csdn.net/20170928205813612?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**master上安装插件**

	mysql> install plugin rpl_semi_sync_master soname 'semisync_master.so';
	Query OK, 0 rows affected (0.02 sec)



	mysql> set global rpl_semi_sync_master_enabled=ON;
	Query OK, 0 rows affected (0.00 sec)


![这里写图片描述](http://img.blog.csdn.net/20170928210030950?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**slave上安装插件**

	mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
	Query OK, 0 rows affected (0.02 sec)

	mysql> set global rpl_semi_sync_slave_enabled=ON;
	Query OK, 0 rows affected (0.00 sec)

	mysql> show variables like '%semi%';
![这里写图片描述](http://img.blog.csdn.net/20170928210051064?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**测试：**

首先在master上查看以下参数：
![这里写图片描述](http://img.blog.csdn.net/20170928210558216?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
然后slave上关闭io_thread

	mysql> stop slave io_thread;
	Query OK, 0 rows affected (0.01 sec)

然后在master上执行数据库操作，比如插入等，结果就是操作会等待10s返回结果，这时候退回异步复制，slave上没有接收到数据，这时候我们去查看master上的相关状态：
![这里写图片描述](http://img.blog.csdn.net/20170928210418940?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
此时插入数据会有一个10s的timeout，所以会卡顿一会，有些参数的值也变了：
![这里写图片描述](http://img.blog.csdn.net/20170928210809977?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
	show status like '%rpl_semi_sync%';
会发现部分参数的值改变了（Rpl_semi_sync_master_clients
、Rpl_semi_sync_master_no_tx等）

然后我们开启io_thread再去查看数据库的变化，发现数据同步了

	mysql> start slave io_thread;
	Query OK, 0 rows affected (0.00 sec)
![这里写图片描述](http://img.blog.csdn.net/20170928210840271?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
