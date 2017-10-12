## 一、认识 TiDB
### 1. TiDB 简介

TiDB 是 PingCAP 公司基于 Google Spanner / F1 论文实现的开源分布式 NewSQL 数据库。

TiDB 具备如下 NewSQL 核心特性：

    SQL支持 （TiDB 是 MySQL 兼容的）
    水平线性弹性扩展
    分布式事务
    跨数据中心数据强一致性保证
    故障自恢复的高可用

TiDB 的设计目标是 100% 的 OLTP 场景和 80% 的 OLAP 场景。

TiDB 对业务没有任何侵入性，能优雅的替换传统的数据库中间件、数据库分库分表等 Sharding 方案。同时它也让开发运维人员不用关注数据库 Scale 的细节问题，专注于业务开发，极大的提升研发的生产力。

### 2. TiDB 整体架构

要深入了解 TiDB 的水平扩展和高可用特点，首先需要了解 TiDB 的整体架构。
![这里写图片描述](http://img.blog.csdn.net/20171012231137365?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
TiDB 集群主要分为三个组件：

**TiDB Server**

TiDB Server 负责接收 SQL 请求，处理 SQL 相关的逻辑，并通过 PD 找到存储计算所需数据的 TiKV 地址，与 TiKV 交互获取数据，最终返回结果。 TiDB Server 是无状态的，其本身并不存储数据，只负责计算，可以无限水平扩展，可以通过负载均衡组件（如LVS、HAProxy 或 F5）对外提供统一的接入地址。

**PD Server**

Placement Driver (简称 PD) 是整个集群的管理模块，其主要工作有三个： 一是存储集群的元信息（某个 Key 存储在哪个 TiKV 节点）；二是对 TiKV 集群进行调度和负载均衡（如数据的迁移、Raft group leader 的迁移等）；三是分配全局唯一且递增的事务 ID。

PD 是一个集群，需要部署奇数个节点，一般线上推荐至少部署 3 个节点。

**TiKV Server**

TiKV Server 负责存储数据，从外部看 TiKV 是一个分布式的提供事务的 Key-Value 存储引擎。存储数据的基本单位是 Region，每个 Region 负责存储一个 Key Range （从 StartKey 到 EndKey 的左闭右开区间）的数据，每个 TiKV 节点会负责多个 Region 。TiKV 使用 Raft 协议做复制，保持数据的一致性和容灾。副本以 Region 为单位进行管理，不同节点上的多个 Region 构成一个 Raft Group，互为副本。数据在多个 TiKV 之间的负载均衡由 PD 调度，这里也是以 Region 为单位进行调度。

### 3. 核心特性

**水平扩展**

无限水平扩展是 TiDB 的一大特点，这里说的水平扩展包括两方面：计算能力和存储能力。TiDB Server 负责处理 SQL 请求，随着业务的增长，可以简单的添加 TiDB Server 节点，提高整体的处理能力，提供更高的吞吐。TiKV 负责存储数据，随着数据量的增长，可以部署更多的 TiKV Server 节点解决数据 Scale 的问题。PD 会在 TiKV 节点之间以 Region 为单位做调度，将部分数据迁移到新加的节点上。所以在业务的早期，可以只部署少量的服务实例（推荐至少部署 3 个 TiKV， 3 个 PD，2 个 TiDB），随着业务量的增长，按照需求添加 TiKV 或者 TiDB 实例。

**高可用**

高可用是 TiDB 的另一大特点，TiDB/TiKV/PD 这三个组件都能容忍部分实例失效，不影响整个集群的可用性。下面分别说明这三个组件的可用性、单个实例失效后的后果以及如何恢复。

    TiDB

    TiDB 是无状态的，推荐至少部署两个实例，前端通过负载均衡组件对外提供服务。当单个实例失效时，会影响正在这个实例上进行的 Session，从应用的角度看，会出现单次请求失败的情况，重新连接后即可继续获得服务。单个实例失效后，可以重启这个实例或者部署一个新的实例。

    PD

    PD 是一个集群，通过 Raft 协议保持数据的一致性，单个实例失效时，如果这个实例不是 Raft 的 leader，那么服务完全不受影响；如果这个实例是 Raft 的 leader，会重新选出新的 Raft leader，自动恢复服务。PD 在选举的过程中无法对外提供服务，这个时间大约是3秒钟。推荐至少部署三个 PD 实例，单个实例失效后，重启这个实例或者添加新的实例。

    TiKV

    TiKV 是一个集群，通过 Raft 协议保持数据的一致性（副本数量可配置，默认保存三副本），并通过 PD 做负载均衡调度。单个节点失效时，会影响这个节点上存储的所有 Region。对于 Region 中的 Leader 结点，会中断服务，等待重新选举；对于 Region 中的 Follower 节点，不会影响服务。当某个 TiKV 节点失效，并且在一段时间内（默认 10 分钟）无法恢复，PD 会将其上的数据迁移到其他的 TiKV 节点上。

## 二、为zabbix配置TiDB数据库
本文配置测试环境为rhel7.2 x86_64bit，且已经安装zabbix-server并且配置过mariadb数据库
### 1. TiDB 安装前系统配置与检查

文件系统 	TiDB 部署环境推荐使用 ext4 文件系统
Swap 空间 	TiDB 部署推荐关闭 Swap 空间
Disk Block Size 	设置系统磁盘 Block 大小为 4096

selinux处于disabled状态
防火墙关闭：请查看 TiDB 所需端口在各个节点之间是否能正常访问

### 2. 下载官方 Binary

 \# 下载压缩包

	wget http://download.pingcap.org/tidb-latest-linux-amd64.tar.gz
	wget http://download.pingcap.org/tidb-latest-linux-amd64.sha256

 \# 检查文件完整性，返回 ok 则正确

	sha256sum -c tidb-latest-linux-amd64.sha256

 \# 解开压缩包(在所有tidb主机中都要有二进制包)

	tar -xzf tidb-latest-linux-amd64.tar.gz
	cd tidb-latest-linux-amd64

### 3. 功能性测试部署

这里我将使用四个节点，部署一个 PD，三个 TiKV，以及一个 TiDB，各个节点以及所运行服务信息如下：
| hostname | IP| 安装的系统服务 |
| ------------- |:-------------| :-----|
| lockey41 | 192.168.0.41 | PD1, TiDB,zabbix-server,web |
| lockey151 | 192.168.0.151 | TiKV1 |
| lockey152 | 192.168.0.152 | TiKV2 |
| lockey153 | 192.168.0.153 | TiKV3 |

按如下步骤依次启动 PD 集群，TiKV 集群以及 TiDB：

**注意：以下启动各个应用程序组件实例的时候，请选择后台启动，避免前台失效后程序自动退出。**

	cd tidb-latest-linux-amd64

**步骤一. 在lockey41 启动 PD：**

	./bin/pd-server --name=pd1 --data-dir=pd1 --client-urls="http://192.168.0.41:2379" --peer-urls="http://192.168.0.41:2380" --initial-cluster="pd1=http://192.168.0.41:2380" --log-file=pd.log &


**步骤二. 在 lockey151，lockey152，lockey153 启动 TiKV：**

lockey151

	./bin/tikv-server --pd="192.168.0.41:2379" --addr="192.168.0.151:20160" --data-dir=tikv1 --log-file=tikv.log &
lockey152

	./bin/tikv-server --pd="192.168.0.41:2379" --addr="192.168.0.152:20160" --data-dir=tikv1 --log-file=tikv.log &
lockey153

	./bin/tikv-server --pd="192.168.0.41:2379" --addr="192.168.0.151:20160" --data-dir=tikv1 --log-file=tikv.log &

**步骤三. 在 lockey41 启动 TiDB：**

	./bin/tidb-server

对于各服务的启动状态和数据库端口可以使用以下命令检查：

	ps -aux | grep server
	netstat -antlp | grep 4000

**步骤四. 使用 MySQL 客户端连接 TiDB：**
![这里写图片描述](http://img.blog.csdn.net/20171012234848872?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
这一步如果OK的话基本的数据库配置就成功了，接下来需要往zabbix应用中进行集成；如果这一步出现问题请先检查防火墙（systemctl status firewalld）是否关闭以及各服务是否正常启动

	[root@lockey41 zabbix-server-mysql-3.4.2]# mysql -h 127.0.0.1 -P 4000 -ulockey -plockey23
	Welcome to the MariaDB monitor.  Commands end with ; or \g.
	Your MySQL connection id is 6
	Server version: 5.7.1-TiDB-0.9.0 MySQL Community Server (Apache License 2.0)

	Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

	Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
	MySQL [test]> CREATE DATABASE zabbix CHARSET 'utf8';
	Query OK, 0 rows affected (0.07 sec)

	MySQL [test]> GRANT ALL ON zabbix.* TO 'lockey'@'192.168.%.%' IDENTIFIED BY 'lockey23';
	Query OK, 1 row affected (0.01 sec)

	MySQL [(none)]> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| INFORMATION_SCHEMA |
	| zabbix             |
	+--------------------+
	2 rows in set (0.00 sec)

**记得往数据库中导入数据**

	[root@lockey41 ~]# cd /usr/share/doc/zabbix-server-mysql-3.4.2/
	[root@lockey41 zabbix-server-mysql-3.4.2]# tar -zxvf create.sql.gz
	[root@lockey41 zabbix-server-mysql-3.4.2]# mysql -h 192.168.0.41 -P 4000 -ulockey -plockey23 zabbix<create.sql

### 4. 将TiDB集成到zabbix中去

**修改之前的zabbix-server配置文件**

vim /etc/zabbix/zabbix_server.conf

	DBPort=4000##特别注意这一行

修改zabbix的web（php）配置

	[root@lockey41 web]# pwd
	/etc/zabbix/web
	[root@lockey41 web]# vim zabbix.conf.php

![这里写图片描述](http://img.blog.csdn.net/20171012234832727?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
如果上边的配置没有修改则会出现以下错误：

	Database error      Error connecting to database: Can't connect to local MySQL server through socket
![这里写图片描述](http://img.blog.csdn.net/20171012234926899?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上边步骤如果都OK将可以正常登录（初始用户名密码Admin/zabbix）
![这里写图片描述](http://img.blog.csdn.net/20171012235128700?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
登录成功首页如下展示：
![这里写图片描述](http://img.blog.csdn.net/20171012235321993?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
