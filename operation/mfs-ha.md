
### MooseFS简介

MooseFS(即Moose File System，简称MFS)是一个具有容错性的网络分布式文件系统，它将数据分散存放在多个物理服务器或单独磁盘或分区上，确保一份数据有多个备份副本，对于访问MFS的客户端或者用户来说，整个分布式网络文件系统集群看起来就像一个资源一样，也就是说呈现给用户的是一个统一的资源。MooseFS就相当于UNIX的文件系统（类似ext3、ext4、nfs），它是一个分层的目录树结构。
MFS存储支持POSIX标准的文件属性（权限，最后访问和修改时间），支持特殊的文件，如块设备，字符设备，管道、套接字、链接文件（符合链接、硬链接）；
MFS支持FUSE(用户空间文件系统Filesystem in Userspace，简称FUSE），客户端挂载后可以作为一个普通的Unix文件系统使用MooseFS。
MFS可支持文件自动备份的功能，提高可用性和高扩展性。MogileFS不支持对一个文件内部的随机或顺序读写，因此只适合做一部分应用，如图片服务，静态HTML服务、
文件服务器等，这些应用在文件写入后基本上不需要对文件进行修改，但是可以生成一个新的文件覆盖原有文件

本文将一步步来介绍mfs的基本用法以及如何实现高可用

**实验主机环境(redhat 6.5 x86_64bit)**

| ip | hostname | softwares to install |
| ------------- |:-------------| -----|
| 192.168.1.8 | cobbler1 | mfs-master cgi-server keepalived|
| 192.168.1.9 | cobbler2 | mfs-master cgi-server keepalived |
| 192.168.1.10 | cobbler3 | chunkserver |
| 192.168.1.11 | cobbler4 | chunkserver |
| 192.168.1.12 | cobbler5 | mfs-client |


**实验步骤**

	1.cobbler1，cobbler2安装测试mfs-master 和 cgi-server
	2.cobbler3，cobbler4安装测试chunkserver
	3.cobbler5安装测试mfs-client
	4.1.cobbler1，cobbler2安装测试keepalived
	5.cobbler1，cobbler2配置mfs高可用

### 1.cobbler1，cobbler2安装测试mfs-master 和 cgi-server(两台主机操作相同)

安装包本人已经上传到[百度云盘](https://pan.baidu.com/s/1gfD14f5)，用到的rpm包如下：
![这里写图片描述](http://img.blog.csdn.net/20171023221612210?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


	yum install moosefs-cgi-3.0.80-1.x86_64.rpm moosefs-cgiserv-3.0.80-1.x86_64.rpm moosefs-master-3.0.80-1.x86_64.rpm -y

启动服务：

	mfsmaster start
	mfscgiserv start

**做好本地解析**
/etc/hosts#cobbler1

	192.168.1.8 mfsmaster
![这里写图片描述](http://img.blog.csdn.net/20171023222134985?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

/etc/hosts#cobbler2

	192.168.1.9 mfsmaster
![这里写图片描述](http://img.blog.csdn.net/20171023222304406?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

	此时进入/var/lib/mfs 可以看到 moosefs 所产生的数据:
	.mfsmaster.lock 文件记录正在运行的 mfsmaster 的主进程
	metadata.mfs, metadata.mfs.back MooseFS 文件系统的元数据 metadata 的镜像
	changelog.*.mfs 是 MooseFS 文件系统元数据的改变日志(每一个小时合并到 metadata.mfs中一次)
	Metadata 文件的大小是取决于文件数的多少(而不是他们的大小)。changelog 日志的大小是取决于每小时操作的数目,但是这个时间长度(默认是按小时)是可配置的

### 2.cobbler3，cobbler4安装测试chunkserver（两台主机同步）

![这里写图片描述](http://img.blog.csdn.net/20171023222629677?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

	yum install moosefs-chunkserver-3.0.80-1.x86_64.rpm -y#软件安装
	mkdir /mnt/chunk123#创建存储目录，两台主机目录要区分开
	chown mfs.mfs /mnt/chunk123/ -R#修改目录所有者
	编辑配置文件
	[root@cobbler3 ~]# sed -n '/#/!p' /etc/mfs/mfshdd.cfg
	/mnt/chunk123

	mfschunkserver start#启动服务

**做好解析之后访问测试：**
/etc/hosts#两台都做，可以指定mfsmaster为192.168.1.8或者192.168.1.9都可以，我这里都指向了192.168.1.8

	192.168.1.8 mfsmaster cobbler1

**页面访问测试是否加入**
![这里写图片描述](http://img.blog.csdn.net/20171023223851353?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 3.cobbler5安装测试mfs-client

![这里写图片描述](http://img.blog.csdn.net/20171023223937675?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

	yum install moosefs-client-3.0.80-1.x86_64.rpm -y

	vim /etc/mfs/mfsmount.cfg#定义客户端默认挂载
	/mnt/mfs

	vim /etc/hosts#做解析
	192.168.1.8 mfsmaster cobbler1


	mfsmount

![这里写图片描述](http://img.blog.csdn.net/20171023232216443?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
	cd /mnt/mfs/

	[root@cobbler5 mfs]# ls#以下列出的为手动创建的目录，用来做数据测试的
	test1  test2

	[root@cobbler5 mfs]# pwd
	/mnt/mfs

	mfssetgoal -r 1 test1#设置此目录下数据的备份数为1
	mfssetgoal -r 2 test2#设置此目录下数据的备份数为2

	cp /etc/passwd test1
	cp /etc/passwd test2

![这里写图片描述](http://img.blog.csdn.net/20171023224553898?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

根据上图的结果可知cobbler4上存了两份数据，分别来自test1和test2，我们去看一下：
![这里写图片描述](http://img.blog.csdn.net/20171023225008905?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20171023225107658?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 4. cobbler1，cobbler2安装测试keepalived

	[root@cobbler1 ~]# wget http://www.keepalived.org/software/keepalived-1.3.5.tar.gz
	[root@cobbler1 ~]# tar -zxf keepalived-1.3.5.tar.gz
	[root@cobbler1 ~]# cd keepalived-1.3.5
	[root@cobbler1 ~]# ./configure --prefix=/usr/local/keepalived --with-init=SYSV
	[root@cobbler1 ~]# make && make install
	[root@cobbler1 ~]# ln -s /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
	[root@cobbler1 ~]# chmod +x /etc/init.d/keepalived
	[root@cobbler1 ~]# ln -s /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
	[root@cobbler1 ~]# ln -s /usr/local/keepalived/etc/keepalived/ /etc/
	[root@cobbler1 ~]# ln -s /usr/local/keepalived/sbin/keepalived /usr/sbin/
	[root@cobbler1 ~]# scp -r /usr/local/keepalived/ cobbler2:/usr/local/

**因为安装目录都scp到cobbler2上了，所以cobbler2上只需要执行以下操作即可：**

		[root@cobbler2 ~]# ln -s /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
		[root@cobbler2 ~]# chmod +x /etc/init.d/keepalived
		[root@cobbler2 ~]# ln -s /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
		[root@cobbler2 ~]# ln -s /usr/local/keepalived/etc/keepalived/ /etc/
		[root@cobbler2 ~]# ln -s /usr/local/keepalived/sbin/keepalived /usr/sbin/

启动服务看是否正常：#最好1和2都测试一下

	[root@cobbler1 ~]# /etc/init.d/keepalived start
	Starting keepalived:
	[root@cobbler1 ~]# ps -aux | grep keep
	Warning: bad syntax, perhaps a bogus '-'? See /usr/share/doc/procps-3.2.8/FAQ
	root      15261  0.0  0.0  37344   988 ?        Ss   23:19   0:00 keepalived -D
	root      15263  0.0  0.1  37344  1952 ?        S    23:19   0:00 keepalived -D
	root      15264  0.0  0.1  37344  1388 ?        S    23:19   0:01 keepalived -D
	root      20677  0.0  0.0 103272   880 pts/0    S+   23:55   0:00 grep keep

### 5.cobbler1，cobbler2配置mfs高可用

**cobbler1上**

[root@cobbler1 ~]# cat /etc/keepalived/keepalived.conf

	! Configuration File for keepalived
	global_defs {
	  notification_email {
	    root@localhost
	    }

	notification_email_from keepalived@localhost
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	router_id MFS_HA_MASTER#作为mfs高可用的MASTER
	}

	vrrp_script chk_mfs {                          
	  script "/etc/keepalived/ifalived_mfsmaster.sh"#不断执行的服务检测脚本
	  interval 2
	  weight 2
	}

	vrrp_instance VI_1 {
	  state MASTER#MASTER身份
	  interface eth1
	  virtual_router_id 51
	  priority 100#优先级，初始一定要比BACKUP高
	  advert_int 1
	  authentication {
	    auth_type PASS
	    auth_pass 1111
	}
	  track_script {
	    chk_mfs#上面定义的脚本调用
	}
	virtual_ipaddress {
	    192.168.1.99/24
	}
	notify_master "/etc/keepalived/mfs-data-sync.sh "#当它成为master之后要执行的脚本，自己可以定义
	}

**cobbler2和cobbler1的大体配置一致，需要修改三处**
![这里写图片描述](http://img.blog.csdn.net/20171023230514171?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


**服务监控脚本（cobbler1和cobbler2一样，要有执行权限）：**
[root@cobbler1 ~]# cat /etc/keepalived/ifalived_mfsmaster.sh

	#!/bin/bash
	STATUS=`ps -C mfsmaster --no-header | wc -l`
	if [ $STATUS -eq 0 ]
	then
	mfsmaster start
	sleep 3
	   if [ `ps -C mfsmaster --no-header | wc -l ` -eq 0 ]
		then
	      	killall -9 mfscgiserv
	      	killall -9 keepalived
	   fi
	fi

配置结束，因为到这里高可用算是搭建起来了，后面将通过VIP来进行master切换，所以修改以下解析（五台主机都做）

	/etc/hosts
	192.168.1.99 mfsmater

启动所有服务之后会看到VIP在keepalived 的 master主机上
![这里写图片描述](http://img.blog.csdn.net/20171023230941249?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

访问页面测试一下：
![这里写图片描述](http://img.blog.csdn.net/20171023231054355?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

因为在mfsmaster切换的时候可能会出现数据不同步的问题，所以我们需要写个数据同步的脚本来做这件事情：

[root@cobbler2 ~]# cat /etc/keepalived/mfs-data-sync.sh

	#!/bin/bash
	STATUS=`ip addr|grep 192.168.1.99|awk -F" " '{print $2}'|cut -d"/" -f1`
	if [ $STATUS == 192.168.1.99 ];then
	   mfsmaster stop
	   /bin/rm -f /var/lib/mfs/*
	   /usr/bin/rsync -e "ssh -p22" -avpgolr 192.168.1.8:/var/lib/mfs/* /var/lib/mfs/
	   /usr/sbin/mfsmetarestore -m
	   mfsmaster -ai
	   sleep 3
	   echo "this server has become the master of MFS"
	   if [ $STATUS != 192.168.1.99 ];then
	   echo "this server is still MFS's slave"
	   fi
	fi

然后将这个脚本路径加到刚才keepalived配置的notify_master参数后面，这样每次进行master切换之后都会执行数据同步，如：

	notify_master "/etc/keepalived/mfs-data-sync.sh "#当它成为master之后要执行的数据同步脚本

后面就可以通过在一边终止掉keepalived进程来进行故障模拟然后观察vip的漂移以及这个过程的文件系统可访问性测试高可用。
