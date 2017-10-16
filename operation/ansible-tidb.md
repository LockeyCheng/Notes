一般环境中TiDB 集群：

![这里写图片描述](http://img.blog.csdn.net/20171017012703023?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**本实验主机环境(rhel7.2 x86_64bit)**

	192.168.199.218 中控机  lockey218
	192.168.199.126 PD，TiDB lockey126
	192.168.199.130 TiKV1 lockey130
	192.168.199.128 TiKV2 lockey128
	192.168.199.224 TiKV3 lockey224

**注意:**

1.在开始进行集群配置之前需要为各主机添加用来部署集群的用户（需具有sudo权限），并且做好ssh免密连接工作（个主机之间都需要做免密）：

	ssh-keygen -t rsa
	cd .ssh
	cat id_rsa.pub >> authorized_keys
	chmod 644 authorized_keys
	ssh-copy-id -i ~/.ssh/id_rsa.pub 192.168.1.100

2.机器之间互通网络。部署时关闭防火墙和 iptables，部署完成后再开启。

3.所有机器的时间和时区设置一致，有 NTP 服务可以同步正确时间。
### 1. 在中控机上安装 Ansible

	yum install epel-release
	yum update
	yum install ansible
![这里写图片描述](http://img.blog.csdn.net/20171017005126895?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 2. 中控机下载 TiDB-Ansible

下载地址https://github.com/pingcap/tidb-ansible

将下载下来的文件解压缩，默认的文件夹名称为 tidb-ansible-master。该文件夹包含用 TiDB-Ansible 来部署 TiDB 集群所需要的所有文件。

[root@lockey152-218 tidb-ansible-master]# tree -L 1

	.
	├── ansible.cfg
	├── bootstrap.yml
	├── cloud
	├── conf
	├── deploy.yml
	├── DOC.md
	├── downloads
	├── fact_files
	├── fetch_logfile.yml
	├── filter_plugins
	├── group_vars
	├── inventory.ini
	├── library
	├── LICENSE
	├── local_prepare.yml
	├── README.md
	├── resources
	├── retry_files
	├── roles
	├── rolling_update.yml
	├── scripts
	├── start_spark.yml
	├── start.yml
	├── stop_spark.yml
	├── stop.yml
	├── templates
	├── unsafe_cleanup_data.yml
	├── unsafe_cleanup.yml
	├── unsafe_restart.yml
	├── y
	└── y.pub

	12 directories, 19 files
### 3. 编辑 tidb-ansible-master 文件夹中的 inventory.ini 文件来配置集群节点：

	# TiDB Cluster Part
	[tidb_servers]
	192.168.199.126

	[tikv_servers]
	192.168.199.224
	192.168.199.128
	192.168.199.130

	[pd_servers]
	192.168.199.126

	[spark_master]

	[spark_slaves]

	# Monitoring Part
	[monitoring_servers]
	192.168.199.126

	[grafana_servers]
	192.168.199.126

	[monitored_servers:children]
	tidb_servers
	tikv_servers
	pd_servers
	spark_master
	spark_slaves

	## Binlog Part
	[pump_servers:children]
	tidb_servers

	[cistern_servers]

	[drainer_servers]

	[pd_servers:vars]
	# location_labels = ["zone","rack","host"]

	## Global variables
	[all:vars]
	deploy_dir = /home/tidb/deploy

	## Connection
	# ssh via root:
	# ansible_user = root
	# ansible_become = true
	# ansible_become_user = tidb

	# ssh via normal user
	ansible_user = halo

	cluster_name = test-cluster

	# misc
	enable_elk = False
	enable_firewalld = False
	enable_ntpd = True
	machine_benchmark = True
	set_hostname = False
	tidb_version = latest
	use_systemd = False

	# binlog trigger
	enable_binlog = False


### 4. 连网下载 TiDB、TiKV 和 PD binaries：

	ansible-playbook local_prepare.yml
	若是虚拟机，则此过程大约持续近十分钟，请耐心等耐

![这里写图片描述](http://img.blog.csdn.net/20171017010633515?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20171017010655290?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 5. 初始化目标机器的系统环境，修改内核参数：

	ansible-playbook bootstrap.yml -k -K
	如果连接至托管节点需要密码，需添加 -k（小写）参数。这同样适用于其他 playbooks。
	如果 sudo 到 root 权限需要密码，需添加 -K（大写）参数。
	此过程耗时较长且容易出现各种问题

安装前请确保主机的内存和磁盘容量都比较大，否则可能会出现因为磁盘不够用导致的失败
![这里写图片描述](http://img.blog.csdn.net/20171017010744758?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

写入速度慢导致的失败
![这里写图片描述](http://img.blog.csdn.net/20171017010724566?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

不知为什么的主机不可达导致的失败
![这里写图片描述](http://img.blog.csdn.net/20171017010908717?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 6. 部署 TiDB 集群：

	ansible-playbook deploy.yml -k

![这里写图片描述](http://img.blog.csdn.net/20171017011207213?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

各节点主机中ntp服务未安装导致的错误
**解决：yum install -y ntp.x86_64**

	systemctl start ntpd#启动服务

	ntpstat#查看时间同步
![这里写图片描述](http://img.blog.csdn.net/20171017011501613?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

最大文件描述符太小导致的错误：
![这里写图片描述](http://img.blog.csdn.net/20171017011633072?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

解决：echo 100000 > /proc/sys/fs/file-max

### 7. 启动 TiDB 集群

启动 TiDB 集群：

	ansible-playbook start.yml -k

使用 MySQL 客户端连接至 TiDB 集群：

	mysql -u root -h 192.168.199.126 -P 4000

TiDB 基本操作的基本操作与mysql类似，可以自己进行增删改查的练习

**打开浏览器，访问以下监控平台：**

地址：http://192.168.199.126:3000， 默认帐户和密码为：admin@admin。
![这里写图片描述](http://img.blog.csdn.net/20171017012115947?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**重要监控指标详解**

**PD**

        Storage Capacity : TiDB 集群总可用数据库空间大小

        Current Storage Size : TiDB 集群目前已用数据库空间大小

        Store Status -- up store : TiKV 正常节点数量

        Store Status -- down store : TiKV 异常节点数量

        如果大于 0，证明有节点不正常

        Store Status -- offline store : 手动执行下线操作 TiKV 节点数量

        Store Status -- Tombstone store : 下线成功的 TiKV 节点数量

        Current storage usage : TiKV 集群存储空间占用率

        超过 80% 应考虑添加 TiKV 节点

        99% completed_cmds_duration_seconds : 99% pd-server 请求完成时间

        小于 5ms

        average completed_cmds_duration_seconds : pd-server 请求平均完成时间

        小于 50ms

        leader balance ratio : leader ratio 最大的节点与最小的节点的差

        均衡状况下一般小于 5%，节点重启时会比较大

        region balance ratio : region ratio 最大的节点与最小的节点的差

        均衡状况下一般小于 5%，新增/下线节点时会比较大

**TiDB**

        handle_requests_duration_seconds : 请求 PD 获取 TSO 响应时间

        小于 100ms

        tidb server QPS : 集群的请求量

        connection count : 从业务服务器连接到数据库的连接数

        和业务相关。但是如果连接数发生跳变，需要查明原因。比如突然掉为 0，可以检查网络是否中断； 如果突然上涨，需要检查业务。

        statement count : 单位时间内不同类型语句执行的数目

        Query Duration 99th percentile : 99% 的 query 时间

**TiKV**

        99% & 99.99% scheduler command duration : 99% & 99.99% 命令执行的时间

      99% 小于 50ms；99.99% 小于 100ms

        95% & 99% storage async_request duration : 95% & 99% Raft 命令执行时间

        95% 小于 50ms；99% 小于 100ms

        server report failure message : 发送失败或者收到了错误的 message

        如果出现了大量的 unreachadble 的消息，表明系统网络出现了问题。如果有 store not match 这样的错误， 表明收到了不属于这个集群发过来的消息

        Vote : Raft vote 的频率

        通常这个值只会在发生 split 的时候有变动，如果长时间出现了 vote 偏高的情况，证明系统出现了严重的问题， 有一些节点无法工作了

        95% & 99% coprocessor request duration : 95% & 99% coprocessor 执行时间

        和业务相关，但通常不会出现持续高位的值

        Pending task : 累积的任务数量

        除了 pd worker，其他任何偏高都属于异常

        stall : RocksDB Stall 时间

        大于 0，表明 RocksDB 忙不过来，需要注意 IO 和 CPU 了

        channel full : channel 满了，表明线程太忙无法处理

        如果大于 0，表明线程已经没法处理了

        95% send_message_duration_seconds : 95% 发送消息的时间

        小于 50ms

        leader/region : 每个 TiKV 的 leader/region 数量

### 8. 集群销毁

停用集群：

	ansible-playbook stop.yml -k

销毁集群：

	ansible-playbook unsafe_cleanup.yml -k
