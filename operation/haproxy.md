HAProxy提供高可用性、负载均衡以及基于 TCP 和 HTTP 应用的代理服务。

HAProxy的特点是：

	1、支持两种代理模式：TCP（四层）和HTTP（七层），支持虚拟主机；
	2、能够补充Nginx的一些缺点比如Session的保持，Cookie的引导等工作
	3、支持url检测后端的服务器出问题的检测会有很好的帮助。
	4、更多的负载均衡策略比如：动态加权轮循(Dynamic Round Robin)，加权源地址哈希(Weighted Source Hash)，加权URL哈希和加权参数哈希(Weighted Parameter Hash)已经实现
	5、单纯从效率上来讲HAProxy更会比Nginx有更出色的负载均衡速度。
	6、HAProxy可以对Mysql进行负载均衡，对后端的DB节点进行检测和负载均衡。
	7、支持负载均衡算法：Round-robin（轮循）、Weight-round-robin（带权轮循）、source（原地址保持）、RI（请求URL）、rdp-cookie（根据cookie）
	8、不能做Web服务器即Cache。

实验环境：系统 rhel6.5.x86_64 关闭防火墙

实验主机三台：

    172.6.6.20 haproxy
    172.6.6.30 web30 apache
    172.6.6.6 web6 apache

### 1. 安装haproxy

    [root@rhel6-vm2 ~]# rpmbuild -tb haproxy-1.4.23.tar.gz
 #如果提示没有rpmbuild命令就需要先安装rpm-build：

	 yum install rpm-build -y

    [root@rhel6-vm2 ~]# rpm -ivh /root/rpmbuild/RPMS/x86_64/haproxy-1.4.23-1.x86_64.rpm

### 2. 配置haproxy
[root@rhel6-vm2 haproxy]# cat haproxy.cfg

	# this config needs haproxy-1.1.28 or haproxy-1.2.1

	global
		log 127.0.0.1 local0 #指定日志设备
		#log 127.0.0.1 local1 notice
		log loghost local0 info #指定日志类型，还有 err warning debug
		maxconn 65535 #并发最大连接数量
		chroot /usr/share/haproxy #jail 目录
		uid 99 #用户
		gid 99 #组
		daemon #后台运行
		#debug
		#quiet

	defaults
		log	global
		mode http #默认使用 http 的 7 层模式 tcp: 4 层
		option httplog #http 日志格式
		option dontlognull #禁用空链接日志
		retries 3 #重试 3 次失败认为服务器不可用
		option redispatch #当 client 连接到挂掉的机器时，重新分配到健康的主机
		maxconn 65535
		contimeout 5000 #连接超时
		clitimeout 50000 #客户端超时
		srvtimeout 50000 #服务器端超时stats uri /status #haproxy 监控页面

	listen	www.lockey.com *:80#监听的实例名称，地址和端口
		balance	roundrobin#负载均衡算法
		server	web1 172.6.6.30:80 cookie app1inst1 check inter 2000 rise 2 fall 5
		server	web2 172.6.6.6:80 cookie app1inst2 check inter 2000 rise 2 fall 5

### 3. 基本项测试

	# mkdir /usr/share/haproxy
	# /etc/init.d/haproxy start
测试负载for i in {1..20};do curl localhost;done
![这里写图片描述](http://img.blog.csdn.net/20170919214845679?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

访问 haproxy 监控页面：http://172.6.6.20/status
![这里写图片描述](http://img.blog.csdn.net/20170919220651845?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 4.为监控页面添加认证信息

    listen stats_auth 172.6.6.20:80
            stats enable
            stats uri /status #监控页面地址
            stats auth admin:lockey #管理帐号和密码
            stats refresh 5s #刷新频率
![这里写图片描述](http://img.blog.csdn.net/20170919221114231?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 5. haproxy 日志：
vim /etc/rsyslog.conf

	$ModLoad imudp #接受 haproxy 日志
	$UDPServerRun 514
	local0.* /var/log/haproxy.log #日志文件位置

/etc/init.d/rsyslog restart

### 6. haproxy+keepalived
haproxy可以实现负载均衡，但是不能够进行健康检查，比如一个server出现故障，haproxy仍然会把请求转发给故障的server服务器，这样就会导致请求的无效性。keepalive 软件可以进行健康检查，而且能同时实现haproxy 的高可用性，解决haproxy 单点故障的问题。

要实现haproxy+keepalived就需要额外再多加一个主机（172.6.6.10，haproxy+keepalived）形成主从模式master-backup

master(172.6.6.20)配置：
cat /etc/keepalived/keepalived.conf

	! Configuration File for keepalived
	vrrp_script check_haproxy {#检测脚本的配置
	        script "/opt/check_haproxy.sh"
	        interval 2
	        weight 2
	        }
	global_defs {
	   notification_email {
	     lockey@123.com
	   }
	   notification_email_from Alexandre.Cassen@firewall.loc
	   smtp_server 127.0.0.1
	   smtp_connect_timeout 30
	   router_id LVS_DEVEL
	}

	vrrp_instance VI_1 {
	    state MASTER#节点模式为主节点
	    interface eth0
	    virtual_router_id 51
	    priority 100
	    advert_int 1
	    authentication {
	 auth_type PASS
	        auth_pass lockey
	    }
	    virtual_ipaddress {
	        172.6.6.99
	    }
	   track_script {
	        check_haproxy
	        }
	}
backup(172.6.6.10)配置：
cat /etc/keepalived/keepalived.conf

	! Configuration File for keepalived
	vrrp_script check_haproxy {
	        script "/opt/check_haproxy.sh"
	        interval 2
	        weight 2
	        }
	global_defs {
	   notification_email {
	     lockey@123.com
	   }
	   notification_email_from Alexandre.Cassen@firewall.loc
	   smtp_server 127.0.0.1
	   smtp_connect_timeout 30
	   router_id LVS_DEVEL
	}

	vrrp_instance VI_1 {
	    state BACKUP#节点模式为备份节点
	    interface eth0
	    virtual_router_id 51
	    priority 80#节点优先级较MASTER低
	    advert_int 1
	    authentication {
	 auth_type PASS
	        auth_pass lockey
	    }
	    virtual_ipaddress {
	        172.6.6.99#虚拟ip
	    }
	   track_script {
	        check_haproxy#跟随检测脚本
	        }
	}

关于以上配置中涉及到的检测脚本(两台keepalived主机都需要)：
[root@rhel6-vm2 opt]# cat check_haproxy.sh

	#!/bin/bash
	/etc/init.d/haproxy status &> /dev/null || /etc/init.d/haproxy restart &> /dev/null
	if [ $? -ne 0 ];
	then
		/etc/init.d/keepalived stop &> /dev/null
	fi
测试描述：

在外部主机的浏览器中访问172.6.6.99(virtual ip)，会分别得到来自172.6.6.30和172.6.6.6的apache服务返回的页面。起初虚拟ip在keepalived MASTER节点上，如果我们停掉了主控节点的keepalived服务，那么虚拟ip就会转移到备份节点（BACKUP）上，这称之为ip漂移。只要两个控制节点有一个是正常运行的，那么服务就不会中断。
