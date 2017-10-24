
![这里写图片描述](http://img.blog.csdn.net/20171024190413335?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Hadoop的创始人Doug Cutting解释Hadoop的得名 ：“这个名字是我孩子给一个棕黄色的大象玩具命名的。我的命名标准就是简短，容易发音和拼写，没有太多的意义，并且不会被用于别处。小孩子恰恰是这方面的高手”，我只能说大神们果然任性！

### Hadoop是什么？

**开源的、分布式存储、分布式计算的平台**。Hadoop原本来自于谷歌一款名为MapReduce的编程模型包。谷歌的MapReduce框架可以把一个应用程序分解为许多并行计算指令，跨大量的计算节点运行非常巨大的数据集。使用该框架的一个典型例子就是在网络数据上运行的搜索算法。Hadoop 最初只与网页索引有关，迅速发展成为分析大数据的领先平台。

### 核心架构
![这里写图片描述](http://img.blog.csdn.net/20171024191237164?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### Hadoop优点：

	高可靠性。Hadoop按位存储和处理数据的能力值得人们信赖。

	高扩展性。Hadoop是在可用的计算机集簇间分配数据并完成计算任务的，这些集簇可以方便地扩展到数以千计的节点中。

	高效性。Hadoop能够在节点之间动态地移动数据，并保证各个节点的动态平衡，因此处理速度非常快。

	高容错性。Hadoop能够自动保存数据的多个副本，并且能够自动将失败的任务重新分配。


### 应用场景：

	Web 搜索，即数据挖掘
	大数据的并行计算
	3-D建模与渲染，气象预报，科学计算等

Hadoop简短介绍就到这，接下来进入实战环节：

	在典型的 HA 集群中，通常有两台不同的机器充当 NN。在任何时间，只有一台机器处于Active 状态；另一台机器是处于 Standby 状态。Active NN 负责集群中所有客户端的操作；而 Standby NN 主要用于备用，它主要维持足够的状态，如果必要，可以提供快速的故障恢复。为了让 Standby NN 的状态和 Active NN 保持同步，即元数据保持一致，它们都将会和JournalNodes 守护进程通信。当 Active NN 执行任何有关命名空间的修改，它需要持久化到一半以上的 JournalNodes 上(通过 edits log 持久化存储)，而 Standby NN 负责观察 edits log的变化，它能够读取从 JNs 中读取 edits 信息，并更新其内部的命名空间。一旦 Active NN出现故障，Standby NN 将会保证从 JNs 中读出了全部的 Edits，然后切换成 Active 状态。Standby NN 读取全部的 edits 可确保发生故障转移之前，是和 Active NN 拥有完全同步的命名空间状态。为了提供快速的故障恢复，Standby NN 也需要保存集群中各个文件块的存储位置。为了实现这个，集群中所有的 Database 将配置好 Active NN 和 Standby NN 的位置，并向它们发送块文件所在的位置及心跳，如下图所示：
![这里写图片描述](http://img.blog.csdn.net/20171024195819633?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

	在任何时候，集群中只有一个NN处于Active 状态是极其重要的。否则，在两个Active NN的状态下 NameSpace 状态将会出现分歧，这将会导致数据的丢失及其它不正确的结果。为了保证这种情况不会发生，在任何时间，JNs 只允许一个 NN 充当 writer。在故障恢复期间，将要变成 Active 状态的 NN 将取得 writer 的角色，并阻止另外一个NN 继续处于Active状态。
**实验主机环境与安装配置列表：**

OS均为: Redhat enterprise release 6.5 x86_64bit，均安装jdk
Selinux关闭，防火墙关闭并且做好解析

| IP | HOSTNAME | ROLE && SOFTWARES TO INSTALL |
| ------------- |:-------------|: -----|
| 192.168.0.109 | cobbler1 | hadoop、NFS服务器，NameNode、DFSZKFailoverController、ResourceManager |
| 192.168.0.126 | cobbler2| zookeeper，JournalNode、QuorumPeerMain、DataNode、NodeManager |
| 192.168.0.x | cobbler3| zookeeper，JournalNode、QuorumPeerMain、DataNode、NodeManager |
| 192.168.0.x | cobbler4| zookeeper，JournalNode、QuorumPeerMain、DataNode、NodeManager |
| 192.168.0.138| cobbler5| hadoop，NameNode、DFSZKFailoverController、ResourceManager |
本文涉及安装包已分享到百度网盘
链接: https://pan.baidu.com/s/1bpLHsBX 密码: 6ib9

### 1. 新建一个普通用户用来进行实验（所有主机都做）

	useradd -u 1000 hadoop#所有主机用户相同
	passwd hadoop#设置密码，免密认证要用
	ssh-keygen#生成秘钥
	ssh-copy-id  192.168.0.109#做免密，此步在cobbler1上做了其他主机挂载NFS共享后也生效

### 2. 配置NFS共享（cobbler1做NFS服务器）
NFS共享的好处就在于一台主机做了文件配置其他主机都不用做了，但是一些命令还是需要单独执行，如要使Java环境变量生效命令

**如果系统中没有NFS服务的话执行以下安装即可：**

	yum install nfs-utils -y

**配置共享目录**
![这里写图片描述](http://img.blog.csdn.net/20171024203122429?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

配置/etc/exports文件内容如下，配置完成启动rpcbind以及nfs：

	/home/hadoop *(rw,anonuid=1000,anongid=1000)

选一台主机先做一下连接测试：

![这里写图片描述](http://img.blog.csdn.net/20171024204127382?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到主机上此目录下的所有文件都可以访问，并且可以无密码ssh登录，perfect！

### 3. 安装配置Java环境

	tar -zxvf  jdk-7u79-linux-x64.tar
	ln -s jdk1.7.0_79 jdk#为了好记做了软连接

[hadoop@cobbler1 ~]$ vim .bash_profile

在PATH后面添加以下内容：

![这里写图片描述](http://img.blog.csdn.net/20171024204721155?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

[hadoop@cobbler1 ~]$  source .bash_profile#文件配置虽然同步了，但是此步需要在所有主机上执行

查看java环境变量是否生效：

![这里写图片描述](http://img.blog.csdn.net/20171024204906940?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 4. 安装配置ZOOKEEPER

	tar -zxvf zookeeper-3.4.9.tar

	[hadoop@cobbler1 ~]$ cd zookeeper-3.4.9/
	[hadoop@cobbler1 zookeeper-3.4.9]$ cp conf/zoo_sample.cfg conf/zoo.cfg#拷贝配置文件
	[hadoop@cobbler1 zookeeper-3.4.9]$ vim conf/zoo.cfg #编辑配置文件

在最后添加如下内容：

	server.1=192.168.0.126:2888:3888
	server.2=192.168.0.167:2888:3888
	server.3=192.168.0.169:2888:3888

因为配置中默认的数据目录为：

	dataDir=/tmp/zookeeper

所以需要创建目录，然后在这个目录下执行一个命令（三台zookeeper主机都做，ID要不同）用来标志每一个zookeeper的id（cobbler2对应1，cobbler3对应2，cobbler4对应3）

![这里写图片描述](http://img.blog.csdn.net/20171024211333266?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

然后启动zookeeper并且查看各自的状态：

![这里写图片描述](http://img.blog.csdn.net/20171024211552830?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20171024211604832?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20171024211614821?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


使用bin/zkCli.sh -server 127.0.0.1:2181命令连接zookeeper进行测试：

![这里写图片描述](http://img.blog.csdn.net/20171024211948625?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 5. 安装配置Hadoop

	tar -zxvf hadoop-2.7.3.tar

	ln -s  hadoop-2.7.3 hadoop

**编辑 slaves文件:**
[hadoop@cobbler1 hadoop]$ cat etc/hadoop/slaves

	192.168.0.126
	192.168.0.167
	192.168.0.169

**编辑 core-site.xml 文件:**

	<configuration>
	<!-- 指定 hdfs 的 namenode 为 masters （名称可自定义）-->
	<property>
	<name>fs.defaultFS</name>
	<value>hdfs://masters</value>
	</property>
	<!-- 指定 zookeeper 集群主机地址 -->
	<property>
	<name>ha.zookeeper.quorum</name>
	<value>192.168.0.126:2181,192.167.0.169:2181,192.168.0.169:2181</value>
	</property>
	<!-- 指定客户端的最大尝试连接次数 -->
	<property>
	<name>ipc.client.connect.max.retries</name>
	<value>20</value>
	<description>
	Indicates the number of retries a clientwill make to establisha server connection.
	</description>
	</property>
	<property>
	<!-- 指定客户端的尝试连接间隔 -->
	<name>ipc.client.connect.retry.interval</name>
	<value>5000</value>
	<description>
	Indicates the number of milliseconds aclient will wait for before retrying to establish a server connection.
	</description>
	</property>
	</configuration>

**编辑 hdfs-site.xml 文件：**

	<configuration>
	<!-- 指定 hdfs 的 nameservices 为 masters，和 core-site.xml 文件中的设置保持一
	致 -->
	<!-- 指定节点数 -->
	<property>
	<name>dfs.replication</name>
	<value>3</value>
	</property>
	<property>
	<name>dfs.nameservices</name>
	<value>masters</value>
	</property>
	<!-- masters 下面有两个 namenode 节点，分别是 h1 和 h2 （名称可自定义）
	-->
	<property>
	<name>dfs.ha.namenodes.masters</name>
	<value>h1,h2</value>
	</property>
	<!-- 指定 h1 节点的 rpc 通信地址 -->
	<property>
	<name>dfs.namenode.rpc-address.masters.h1</name>
	<value>192.168.0.109:9000</value>
	</property>
	<!-- 指定 h1 节点的 http 通信地址 -->
	<property>
	<name>dfs.namenode.http-address.masters.h1</name>
	<value>192.168.0.109:50070</value>
	</property>
	<!-- 指定 h2 节点的 rpc 通信地址 -->
	<property>
	<name>dfs.namenode.rpc-address.masters.h2</name>
	<value>192.168.0.138:9000</value>
	</property>
	<!-- 指定 h2 节点的 http 通信地址 -->
	<property>
	<name>dfs.namenode.http-address.masters.h2</name>
	<value>192.168.0.138:50070</value>
	</property>
	<!-- 指定 NameNode 元数据在 JournalNode 上的存放位置 -->
	<property>
	<name>dfs.namenode.shared.edits.dir</name>
	<value>qjournal://192.168.0.126:8485;192.168.0.167:8485;192.168.0.169:8485/masters
	</value>
	</property>
	<!-- 指定 JournalNode 在本地磁盘存放数据的位置 -->
	<property>
	<name>dfs.journalnode.edits.dir</name>
	<value>/tmp/journaldata</value></property>
	<!-- 开启 NameNode 失败自动切换 -->
	<property>
	<name>dfs.ha.automatic-failover.enabled</name>
	<value>true</value>
	</property>
	<!-- 配置失败自动切换实现方式 -->
	<property>
	<name>dfs.client.failover.proxy.provider.masters</name>
	<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider
	</value>
	</property>
	<!-- 配置隔离机制方法，每个机制占用一行-->
	<property>
	<name>dfs.ha.fencing.methods</name>
	<value>
	sshfence
	shell(/bin/true)
	</value>
	</property>
	<!-- 使用 sshfence 隔离机制时需要 ssh 免密码 -->
	<property>
	<name>dfs.ha.fencing.ssh.private-key-files</name>
	<value>/home/hadoop/.ssh/id_rsa</value>
	</property>
	<!-- 配置 sshfence 隔离机制超时时间 -->
	<property>
	<name>dfs.ha.fencing.ssh.connect-timeout</name>
	<value>30000</value>
	</property>
	</configuration>


**在三个 DN 上依次启动 journalnode(第一次启动 hdfs 必须先启动 journalnode)**

	[hadoop@cobbler2 hadoop]$ sbin/hadoop-daemon.sh start journalnod
	Error: JAVA_HOME is not set and could not be found.
启动失败，是因为忘记了设置JAVA_HOME

[hadoop@cobbler1 hadoop]$ vim etc/hadoop/hadoop-env.sh

![这里写图片描述](http://img.blog.csdn.net/20171024213755578?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

然后再次启动
![这里写图片描述](http://img.blog.csdn.net/20171024213913028?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**格式化 HDFS 集群**

	$ bin/hdfs namenode -format
	Namenode 数据默认存放在/tmp，需要把数据拷贝到 h2
![这里写图片描述](http://img.blog.csdn.net/20171024214304511?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
	$ scp -r /tmp/hadoop-hadoop cobbler5:/tmp

**格式化 zookeeper （只需在 h1 上执行即可）**

	$ bin/hdfs zkfc -formatZK (注意大小写)
![这里写图片描述](http://img.blog.csdn.net/20171024214604433?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**启动 hdfs 集群（只需在 h1 上执行即可）**

	$ sbin/start-dfs.sh
![这里写图片描述](http://img.blog.csdn.net/20171024215039336?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**查看各节点状态**

![这里写图片描述](http://img.blog.csdn.net/20171024225435235?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20171024225327591?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**cobbler1处于active状态**
![这里写图片描述](http://img.blog.csdn.net/20171024225240988?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**cobbler5处于standby状态**
![这里写图片描述](http://img.blog.csdn.net/20171024225534047?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**测试故障自动切换**

![这里写图片描述](http://img.blog.csdn.net/20171024225857893?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20171024230044674?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**cobbler5变成 了 active状态**
![这里写图片描述](http://img.blog.csdn.net/20171024225810709?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

通过命令测试

	[hadoop@cobbler2 zookeeper-3.4.9]$ bin/zkCli.sh -server 127.0.0.1:2181

![这里写图片描述](http://img.blog.csdn.net/20171024230726706?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


启动 h1 上的 namenode，此时为 standby 状态

	[hadoop@cobbler1 hadoop]$ sbin/hadoop-daemon.sh start namenode

![这里写图片描述](http://img.blog.csdn.net/20171024230921675?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


**停掉cobbler5上的NameNode**

	[hadoop@cobbler5 tmp]$ jps
	3946 Jps
	3680 NameNode
	3788 DFSZKFailoverController
	[hadoop@cobbler5 tmp]$ kill -9 3680
	[hadoop@cobbler5 tmp]$


**cobbler1变成 了 master**
![这里写图片描述](http://img.blog.csdn.net/20171024231344742?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

至此hdfs的高可用完成!
