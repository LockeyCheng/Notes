**FullNAT:** 除了DR/NAT/TUNNEL之外IPVS下的新的包转发模式，解决了DR/NAT/TUNNEL中的一些缺点（如不能跨vlan或者跨vlan成本太高，服务搭建较复杂，不易运维等）。

**主要规则如下:**

lip(local ip address), cip(client ip address), vip(virtual ip address), rip(realserver ip adress)

 IPVS 转换cip-vip 到/来自 lip-rip,这里的lip和rip都是IDC 内部ip地址,所以LVS负载均衡器和真实主机可以不在同一个vlan中, 而且真实主机只需要接入内网。

**FULLNAT实现如下功能：**

1.数据包从外部进来的时候，目标ip更换为realserver ip，源ip更换为内网local ip；

2.数据包发送出去的时候，目标ip更换为client ip，源ip更换为vip；
性能：和NAT比，正常转发性能下降<10%；

**SYNPROXY: 抵御同步泛滥（也叫拒绝服务攻击DDoS）攻击**

基于SYN cookie

Linux kernel 2.6.32 IPVS下的FullNAT和SYNPROXY程序代码 是由阿里的吴家明和360的陈建以及淘宝朱顺明,在阿里章文嵩的一些建议下写成的。程序代码的写成也受到了源NAT和SYNPROXY 思想的影响。


FullNAT和 SYNPROXY支持被吴家明加到了keepalived/ipvsadm

## 编译
### 1. LVS Kernel

**1.1 从redhat获取kernel rpm**

	 wget ftp://ftp.redhat.com/pub/redhat/linux/enterprise/6Server/en/os/SRPMS/kernel-2.6.32-220.23.1.el6.src.rpm

**1.2从rpm获取kernel源码**

	 rpm -ivh kernel-2.6.32-220.23.1.el6.src.rpm
	 cd ~/rpms/SPECS
	 rpmbuild -bp kernel.spec
注意：执行rpmbuild -bp kernel.spec命令的时候可能会提示一些依赖需要解决，可以通过yum源或者下载相应的rpm包来解决依赖问题。然后还有一个问题：过程中会停下来，这时候需要执行以下命令的一条或者两条用来生成随机数（新开终端）：


	rngd -r /dev/hwrandom
	rngd -r /dev/urandom
如果没有rngd 命令：

	yum whatprovides *\rngd
	yum install rng-tools-2-13.el6_2.x86_64 -y

执行以上步骤之后我们就可以在~/rpms/BUILD中看到内核源码了

**1.3 添加lvs补丁**

	cd ~/rpms/BUILD/kernel-2.6.32-220.23.1.el6/linux-2.6.32-220.23.1.el6.x86_64/
	cp lvs-2.6.32-220.23.1.el6.patch ./
	patch -p1<lvs-2.6.32-220.23.1.el6.patch;
	//需要注意补丁在lvs-fullnat-synproxy.tar.gz文件中，解压后即可看到，不要问我这个压缩文件在哪里，下面
[lvs-fullnat-synproxy.tar.gz](http://kb.linuxvirtualserver.org/images/a/a5/Lvs-fullnat-synproxy.tar.gz)

1.4 打完补丁后我们来进行**编译安装**（虚拟机的话尽量内存给的大一点，因为比较费时间，物理机的话可以多开几个线程同时编译）

	 make #物理机的话可以加参数 -j16 ，表示同时开启16个线程进行编译;编译时间较长，耐心等待或者打局游戏再来看
	 make modules_install;#安装模块
	 make install;
执行完以上步骤之后修改/boot/grub/grub.conf文件中第一个出现的default值为0，然后reboot，主机起来后使用uname -r查看结果是否为kernel-2.6.32，如果是的就可以去**3. LVS 工具的安装这一步了**

### 2. RealServer Kernel (TOA)

获取内核rpm包然后执行与1.1和1.2相同的动作;
然后添加 toa 补丁

	 cd ~/rpms/BUILD/kernel-2.6.32-220.23.1.el6/linux-2.6.32-220.23.1.el6.x86_64/
	 cp toa-2.6.32-220.23.1.el6.patch ./
	 patch -p1<toa-2.6.32-220.23.1.el6.patch; //补丁在lvs-fullnat-synproxy.tar.gz文件中

**编译安装**

	make #物理机的话可以加参数 -j16 ，表示同时开启16个线程进行编译;
	make modules_install;#安装模块
	make install;

### 3. LVS 工具的安装 (keepalived/ipvsadm/quaage)

	 tar xzf lvs-tools.tar.gz
	 cd lvs-fullnat-synproxy
	 tar xzf lvs-tools.tar.gz
lvs-tools.tar.gz在lvs-fullnat-synproxy.tar.gz文件中，解压lvs-tools.tar.gz会生成tools目录，进入目录就可以找到keepalived/ipvsadm/quaage

**3.1 安装keepalived**

	 cd ~/lvs-fullnat-synproxy/keepalived;
	 ./configure --with-kernel-dir="/lib/modules/`uname -r`/build";#注意uname -r被反引号包裹着
	 make;
	 make install;

安装完成之后因为配置文件和启动脚本都不在一般目录下，所以需要做软链接

	ln -s /usr/local/etc/rc.d/init.d/keepalived /etc/init.d/
	ln -s /usr/local/etc/sysconfig/keepalived /etc/sysconfig/
	ln -s /usr/local/etc/keepalived /etc/
	ln -s /usr/local/sbin/keepalived /usr/sbin
**3.2 安装ipvsadm**

	 cd ~/lvs-fullnat-synproxy/tools/ipvsadm;
	 make;
	 make install;
安装完成之后可以使用ipvsadm --help | grep fullnat查看支持

我这边安装过程出现了问题：undefined refenrence \***，删除了一些包之后就OK了，至于具体要删除什么包要看keepalived 和ipvsadm安装过程中的提示，我本机删除的包如下：

	  yum remove libnl
	  yum remove libnl-devel

还有人安装的时候提示 keepalived 某些目录下缺少一些关联文件，将缺失的文件拷贝过去之后安装顺利进行。

**3.3 安装quaage**

	 cd ~/lvs-fullnat-synproxy/tools/quagga;
	 ./configure --disable-ripd --disable-ripngd --disable-bgpd --disable-watchquagga --disable-doc  --enable-user=root --enable-vty-group=root --enable-group=root --enable-zebra --localstatedir=/var/run/quagga

	 make
	 make install
本文虽然基本属于翻译内容，但是操作都经过我本人验证。附原文地址http://kb.linuxvirtualserver.org/wiki/IPVS_FULLNAT_and_SYNPROXY
