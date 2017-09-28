MySQL读写分离基本原理是让master数据库处理写操作，slave数据库处理读操作。一般情况下还有一个主从复制的过程，如果想要了解主从复制的话可以参阅我的上一篇博客[mysql主从复制、基于gtid的主从复制、并行复制、半同步 ](http://blog.csdn.net/Lockey23/article/details/78122111)。MySQL实现读写分离的主要目的是为了提高系统性能以及减小服务压力。

**MySQL读写分离能提高系统性能的原因在于：**

    物理服务器增加，机器处理能力提升。拿硬件换性能。

    主从只负责各自的读和写，极大程度缓解X锁和S锁争用。

    slave可以配置myiasm引擎，提升查询性能以及节约系统开销。

    master直接写是并发的，slave通过主库发送来的binlog恢复数据是异步。

    slave可以单独设置一些参数来提升其读的性能。

    增加冗余，提高可用性。

### 通过MySQLProxy实现读写分离

在msyql主从复制的基础上，我们需要安装MySQLProxy
在这里我们要知道MySQLProxy是通过一些lua脚本来实现的，所以就需要有lua的运行环境，如果系统没有自带的话就需要去安装一下

实验主机：

	master：rhel6.5.x86_64,ip:172.6.6.10
	slave：rhel6.5.x86_64,ip:172.6.6.20
	mysql-proxy：rhel6.5.x86_64,ip:172.6.6.30

下载安装包

https://downloads.mysql.com/archives/get/file/mysql-proxy-0.8.5-linux-el6-x86-64bit.tar.gz

解压之后进行以下操作：

	mv mysql-proxy-0.8.5-linux-el6-x86-64bit /usr/local/mysql-proxy
	cd /usr/local/mysql-proxy/
	mkdir lua
	mkdir log
	cp share/doc/mysql-proxy/rw-splitting.lua ./lua/
	cp share/doc/mysql-proxy/admin.lua ./lua/

然后启动mysql-proxy

	./mysql-proxy --daemon --log-level=debug --user=mysql --keepalive --log-file=/var/log/mysql-proxy.log --plugins="proxy" --proxy-backend-addresses="172.6.6.10" --proxy-read-only-backend-addresses="172.6.6.20" --proxy-lua-script="/usr/local/mysql-proxy/lua/rw-splitting.lua" --plugins=admin --admin-username="admin" --admin-password="admin" --admin-lua-script="/usr/local/mysql-proxy/lua/admin.lua"

 **查看监听端口**

	# lsof -i:4040
	    COMMAND     PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
	    mysql-pro 25722 mysql   10u  IPv4 762429      0t0  TCP *:yo-main (LISTEN)
	    # lsof -i:4041
	    COMMAND     PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
	    mysql-pro 25722 mysql   11u  IPv4 762432      0t0  TCP *:houston (LISTEN)

测试

保证mysqlproxy节点上可执行mysql 。通过复制同步帐号连接proxy

	# mysql -h 172.6.6.30 -uha -pLockey+123 --port=4040

可以看到同步的数据库信息

登录admin

    # mysql -h 172.6.6.30 -u admin -p --port=4041
    mysql> select * from backends;

可以看到master和slave状态

1）登录proxy节点，创建数据库dufu，并创建一张表t

创建完数据库及表后，主从节点上应该都可以看到

2）关闭同步，分别在master和slave上插入数据

	mysql> slave stop;


3）proxy上查看结果

	结果可以看到数据是从slave上读取的，并不是来自master节点上

直接从proxy上插入数据后再次查询

	结果显示查询数据没有变化，因为proxy上执行insert相当于写入到了master上，而查询的数据是从slave上读取的。

启用复制，proxy查询

	mysql>start slave;

查询到的结果是所有数据；说明此时master上的数据同步到了slave，并且在proxy查询到数据是slave数据库的数据。至此MySQLProxy实现了读写的分离。
