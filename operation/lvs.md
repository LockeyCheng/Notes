### LVS
权威发布网址http://www.linuxvirtualserver.org/zh/lvs1-4.html

LVS是 Linux Virtual Server 的简称，也就是Linux虚拟服务器。LVS 是一个实现负载均衡集群的开源软件项目，LVS架构从逻辑上可分为调度层、Server集群层和共享存储。

### LVS的组成

一般来说，LVS集群采用三层结构，三层主要组成部分为：

    1 负载调度器（load balancer），它是整个集群对外面的前端机，负责将客户的请求发送到一组服务器上执行，而客户认为服务是来自一个IP地址（我们可称之为虚拟IP地址）上的。
    2 服务器池（server pool），是一组真正执行客户请求的服务器，执行的服务有WEB、MAIL、FTP和DNS等。
    3 共享存储（shared storage），它为服务器池提供一个共享的存储区，这样很容易使得服务器池拥有相同的内容，提供相同的服务。

### lvs工作原理

可以参考图lvs-elementary-diagram.png

1. 当用户向负载均衡调度器（Director Server）发起请求，调度器将请求发往至内核空间
2. PREROUTING链首先会接收到用户请求，判断目标IP确定是本机IP，将数据包发往INPUT链
3. IPVS是工作在INPUT链上的，当用户请求到达INPUT时，IPVS会将用户请求和自己已定义好的集群服务进行比对，如果用户请求的就是定义的集群服务，那么此时IPVS会强行修改数据包里的目标IP地址及端口，并将新的数据包发往POSTROUTING链
4. POSTROUTING链接收数据包后发现目标IP地址刚好是自己的后端服务器，那么此时通过选路，将数据包最终发送给后端的服务器

### LVS相关术语

1. DS：Director Server。指的是前端负载均衡器节点。
2. RS：Real Server。后端真实的工作服务器。
3. VIP：向外部直接面向用户请求，作为用户请求的目标的IP地址。
4. DIP：Director Server IP，主要用于和内部主机通讯的IP地址。
5. RIP：Real Server IP，后端服务器的IP地址。
6. CIP：Client IP，访问客户端的IP地址。

#### 针对不同的网络服务需求和服务器配置，IPVS调度器实现了如下八种负载调度算法：

    1 轮叫（Round Robin）
    调度器通过"轮叫"调度算法将外部请求按顺序轮流分配到集群中的真实服务器上，它均等地对待每一台服务器，而不管服务器上实际的连接数和系统负载。

    2 加权轮叫（Weighted Round Robin）
    调度器通过"加权轮叫"调度算法根据真实服务器的不同处理能力来调度访问请求。这样可以保证处理能力强的服务器处理更多的访问流量。调度器可以自动问询真实服务器的负载情况，并动态地调整其权值。

    3 最少链接（Least Connections）
    调度器通过"最少连接"调度算法动态地将网络请求调度到已建立的链接数最少的服务器上。如果集群系统的真实服务器具有相近的系统性能，采用"最小连接"调度算法可以较好地均衡负载。

    4 加权最少链接（Weighted Least Connections）
    在集群系统中的服务器性能差异较大的情况下，调度器采用"加权最少链接"调度算法优化负载均衡性能，具有较高权值的服务器将承受较大比例的活动连接负载。调度器可以自动问询真实服务器的负载情况，并动态地调整其权值。

    5 基于局部性的最少链接（Locality-Based Least Connections）
    "基于局部性的最少链接" 调度算法是针对目标IP地址的负载均衡，目前主要用于Cache集群系统。该算法根据请求的目标IP地址找出该目标IP地址最近使用的服务器，若该服务器 是可用的且没有超载，将请求发送到该服务器；若服务器不存在，或者该服务器超载且有服务器处于一半的工作负载，则用"最少链接"的原则选出一个可用的服务 器，将请求发送到该服务器。

    6 带复制的基于局部性最少链接（Locality-Based Least Connections with Replication）
    "带复制的基于局部性最少链接"调度算法也是针对目标IP地址的负载均衡，目前主要用于Cache集群系统。它与LBLC算法的不同之处是它要维护从一个 目标IP地址到一组服务器的映射，而LBLC算法维护从一个目标IP地址到一台服务器的映射。该算法根据请求的目标IP地址找出该目标IP地址对应的服务 器组，按"最小连接"原则从服务器组中选出一台服务器，若服务器没有超载，将请求发送到该服务器，若服务器超载；则按"最小连接"原则从这个集群中选出一 台服务器，将该服务器加入到服务器组中，将请求发送到该服务器。同时，当该服务器组有一段时间没有被修改，将最忙的服务器从服务器组中删除，以降低复制的 程度。

    7目标地址散列（Destination Hashing）
    "目标地址散列"调度算法根据请求的目标IP地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。

    8 源地址散列（Source Hashing）
    "源地址散列"调度算法根据请求的源IP地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。

预备知识：

      arp_ignore:定义接收到ARP请求时的响应级别      
          0：默认，只用本地配置的有响应地址都给予响应      
          1：仅仅在目标IP是本地地址，并且是配置在请求进来的接口上的时候才给予响应(仅在请求的目标地址配置请求到达的接口上的时候，才给予响应)      
      arp_announce：定义将自己的地址向外通告时的级别      
          0：默认，表示使用配置在任何接口的任何地址向外通告      
          1：试图仅向目标网络通告与其网络匹配的地址      
          2：仅向与本地接口上地址匹配的网络进行通告
下边是LVS的四种工作模式

### LVS 的 NAT 模式

Virtual Server via Network Address Translation（VS/NAT）
通过网络地址转换，调度器重写请求报文的目标地址，根据预设的调度算法，将请求分派给后端的真实服务器；真实服务器的响应报文通过调度器时，报文的源地址被重写，再返回给客户，完成整个负载调度过程。

LVS/NAT原理

1. 当用户请求到达Director Server，此时请求的数据报文会先到内核空间的PREROUTING链。 此时报文的源IP为CIP，目标IP为VIP
2. PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链
3. IPVS比对数据包请求的服务是否为集群服务，若是，修改数据包的目标IP地址为后端服务器IP，然后将数据包发至POSTROUTING链。 此时报文的源IP为CIP，目标IP为RIP
4. POSTROUTING链通过选路，将数据包发送给Real Server
5. Real Server比对发现目标为自己的IP，开始构建响应报文发回给Director Server。 此时报文的源IP为RIP，目标IP为CIP
6. Director Server在响应客户端前，此时会将源IP地址修改为自己的VIP地址，然后响应给客户端。 此时报文的源IP为VIP，目标IP为CIP

LVS-NAT模型的特性

1. RS应该使用私有地址，RS的网关必须指向DIP
2. DIP和RIP必须在同一个网段内
3. 请求和响应报文都需要经过Director Server，高负载场景中，Director Server易成为性能瓶颈
4. 支持端口映射
5. RS可以使用任意操作系统

缺陷：伸缩能力有限， 当服务器结点数目升到20时，对Director Server压力会比较大，调度器本身有可能成为系统的新瓶颈，因为在VS/NAT中请求和响应报文都需要通过负载调度器


1.实验环境

三台主机：

    director（双ip 192.168.1.6和172.6.6.8）

    两台real server，ip分别为172.6.6.6和172.6.6.7，并且需要把两个 real server 的内网网关设置为 director 的内网 ip(172.6.6.8)

操作补充：

    添加ip的方法示例：

    (1). ip addr add  172.6.6.8/24 dev eth0

    (2). ifconfig eth0 172.6.6.8 netmask 255.255.255.0

    添加网关的方法：

    route add default gw 172.6.6.8

2、安装和配置

两个 real server 上都安装 nginx 服务，使用curl -I locahost命令测试服务正常运行。

Director 上安装 ipvsadm

    [root@lockey ~]# yum install -y ipvsadm

Director 上编辑 nat 实现脚本然后运行配置

    #! /bin/bash
    # director服务器上开启路由转发功能:
    echo 1 > /proc/sys/net/ipv4/ip_forward
    # 关闭 icmp 的重定向
    echo 0 > /proc/sys/net/ipv4/conf/all/send_redirects
    echo 0 > /proc/sys/net/ipv4/conf/default/send_redirects
    echo 0 > /proc/sys/net/ipv4/conf/eno16777736/send_redirects
    echo 0 > /proc/sys/net/ipv4/conf/virbr0/send_redirects
    echo 0 > /proc/sys/net/ipv4/conf/virbr0-nic/send_redirects
    # director设置 nat 防火墙
    iptables -t nat -F
    iptables -t nat -X
    iptables -t nat -A POSTROUTING -s 172.6.0.0/24 -j MASQUERADE
    # director设置 ipvsadm
    IPVSADM='/sbin/ipvsadm'
    $IPVSADM -C
    $IPVSADM -A -t 192.168.1.6:80 -s wrr
    $IPVSADM -a -t 192.168.1.6:80 -r 172.6.6.6:80 -m -w 1
    $IPVSADM -a -t 192.168.1.6:80 -r 172.6.6.7:80 -m -w 1

查看ipvsadm设置的规则


    [root@lockey packs]# ipvsadm -ln
    IP Virtual Server version 1.2.1 (size=4096)
    Prot LocalAddress:Port Scheduler Flags
      -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
    TCP  192.168.1.6:80 wrr
      -> 172.6.6.6:80                 Masq    1      0          0         
      -> 172.6.6.7:80                 Masq    1      0          0         
    [root@lockey packs]#

3.测试
在Director主机上通过浏览器测试2台real server的web内容http://192.168.1.6

分别停掉172.6.6.6和172.6.6.7上的Nginx服务然后刷新页面可以看到获取到的数据是不一样的：当6关掉后获取的数据是7上的（7服务正常）；当7关掉后获取的数据是6上的（6服务正常）；当两台server都关掉的时候提示Unable to connect


### LVS的DR模式

Virtual Server via Direct Routing（VS/DR）
VS/DR通过改写请求报文的MAC地址，将请求发送到真实服务器，而真实服务器将响应直接返回给客户。同VS/TUN技术一样，VS/DR技术可极大地 提高集群系统的伸缩性。这种方法没有IP隧道的开销，对集群中的真实服务器也没有必须支持IP隧道协议的要求，但是要求调度器与真实服务器都有一块网卡连 在同一物理网段上。

1.实验环境配置

Director: (eth0 172.25.5.30 vip eth0:0 172.25.5.200) ipvsadm

server1(eth0 172.25.5.10 vip lo:0 172.25.5.200)nginx

server2(eth0 172.25.5.40 vip lo:0 172.25.5.200)nginx

2.运行配置

director配置脚本

    #!/bin/bash
    echo 1 >/proc/sys/net/ipv4/ip_forward
    # director服务器上开启路由转发功能:
    ipv=/sbin/ipvsadm
    vip=172.25.5.200
    rs1=172.25.5.10
    rs2=172.25.5.40
    ifconfig eth0:0 down
    ifconfig eth0:0 $vip broadcast $vip netmask 255.255.255.255 up
    route add -host $vip dev eth0:0
    $ipv -C
    $ipv -A -t $vip:80 -s wrr
    $ipv -a -t $vip:80 -r $rs1:80 -g -w 3
    $ipv -a -t $vip:80 -r $rs2:80 -g -w 1


server1和server2配置脚本

    #!/bin/bash
    vip=172.25.5.200
    ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up
    route add -host $vip lo:0
    echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
    echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce

3.在Director主机上通过浏览器测试2台real server的web内容http://172.6.6.9，测试结果与NAT的结果类似


### LVS 与 keepalive

LVS可以实现负载均衡，但是不能够进行健康检查，比如一个real server出现故障，LVS 仍然会把请求转发给故障的real server服务器，这样就会导致请求的无效性。keepalive 软件可以进行健康检查，而且能同时实现 LVS 的高可用性，解决 LVS 单点故障的问题。

1.实验环境配置，四台主机，两台director1，两台real server

director1:172.25.5.20(ipvsadm + keepalived) BACKUP
director2:172.25.5.30(ipvsadm + keepalived) MASTER

real server1:172.25.5.10(nginx)

real server2:172.25.5.40(nginx)

virtual ip 172.25.5.99

2.运行配置

real server1 和 server2的配置脚本

    #!/bin/bash
    vip=172.25.5.99
    ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up
    route add -host $vip lo:0
    echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
    echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce

keepalived1 和 keepalived2配置

    # vim /etc/keepalived/keepalived.conf

    vrrp_instance VI_1 {
        state BACKUP
        interface eth0
        virtual_router_id 51
        priority 90
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        virtual_ipaddress {
            172.25.5.99
        }
    }

    virtual_server 172.25.5.99 80 {
        delay_loop 6
        lb_algo rr
        lb_kind DR
        persistence_timeout 1
        protocol TCP

        real_server 172.25.5.10 80 {
            weight 1
            TCP_CHECK {
                    connect_timeout 10
                    nb_get_retry 2
                    delay_before_retry 3
                    connect_port 80
            }
        }
    	real_server 172.25.5.40 80 {
    		weight 1
    		TCP_CHECK {
    		        connect_timeout 10
    		        nb_get_retry 2
    		        delay_before_retry 3
    		        connect_port 80
    		}
    	    }
    }

两台keepalived主机的大体配置如上，有一些配置细节需要修改

    keepalived1 (172.25.5.30):

    vrrp_instance VI_1 {
    	state MASTER#作为主控
    	priority 100#优先级高

    keepalived2 (172.25.5.20):
    vrrp_instance VI_1 {
    	state BACKUP#作为备控
    	priority 90#优先级较主控低

配置完成查看状态ipvsadm：

    [root@server3 ~]# ipvsadm -L --stats
    IP Virtual Server version 1.2.1 (size=4096)
    Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
      -> RemoteAddress:Port
    TCP  172.25.5.99:http                  114      704        0    61747        0
      -> 172.25.5.40:http                   53      313        0    24199        0


    [root@server2 ~]# ipvsadm -L --stats
    IP Virtual Server version 1.2.1 (size=4096)
    Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
      -> RemoteAddress:Port
    TCP  172.25.5.99:http                    0        0        0        0        0
      -> 172.25.5.40:http                    0        0        0        0        0


3.结果测试，需要在外部可访问主机进行测试（浏览器或者使用curl命令皆可）

    for i in {1..50}; do curl 172.25.5.99;sleep 1;done

stop nginx on 172.25.5.10 or 172.25.5.40 you will find the rusult be different

当系统运行起来之后虚拟ip首先会在主控节点上，如果我们停掉了主控节点的keepalived服务，那么虚拟ip就会转移到备份节点上，这称之为ip漂移。只要两个控制节点有一个是正常运行的，那么服务就不会中断。





### LVS TUN

Virtual Server via IP Tunneling（VS/TUN）
采用NAT技术时，由于请求和响应报文都必须经过调度器地址重写，当客户请求越来越多时，调度器的处理能力将成为瓶颈。为了解决这个问题，调度器把请求报 文通过IP隧道转发至真实服务器，而真实服务器将响应直接返回给客户，所以调度器只处理请求报文。由于一般网络服务应答比请求报文大许多，采用 VS/TUN技术后，集群系统的最大吞吐量可以提高10倍。

IP隧道(IP tunneling)是将一个IP报文封装在另一个IP报文的技术，这可以使得目标为一个IP地址的数据报文能被封装和转发到另一个IP地址。IP隧道技术亦称为IP封装技术(IP encapsulation)。IP隧道主要用于移动主机和虚拟私有网络(Virtual Private Network)，在其中隧道都是静态建立的，隧道一端有一个IP地址，另一端也有唯一的IP地址。它的连接调度和管理与VS/NAT中的一样，只是它的报文转发方法不同。调度器根据各个服务器的负载情况，动态地选择一台服务器，将请求报文封装在另一个IP报文中，再将封装后的IP报文转发给选出的服务器;服务器收到报文后，先将报文解封获得原来目标地址为 VIP 的报文，服务器发现VIP地址被配置在本地的IP隧道设备上，所以就处理这个请求，然后根据路由表将响应报文直接返回给客户。

1.环境配置

director: eth0: 172.25.5.30（ipvsadm）
vip(tunl0）: 172.25.5.100

real server1: eth0:172.25.5.10（Nginx）
tunl0(vip)  :172.25.5.100

real server2: eth0:172.25.5.20（Nginx）
tunl0(vip) :172.25.5.100

2.运行配置

director(172.25.5.30)配置

      yum install ipvsadm -y
      ifconfig tunl0 172.25.5.100 broadcast 172.25.5.100 netmask 255.255.255.0 up
      route add -host $VIP dev tunl0
      ipvsadm -A -t 172.25.5.100:80 -s rr
      ipvsadm -a -t 172.25.5.100:80 -r 172.25.5.10 -i
      ipvsadm -a -t 172.25.5.100:80 -r 172.25.5.20 -i


real server1 和 real server2（配置同下）

    ifconfig tunl0 172.25.5.100 netmask 255.255.255.0 broadcast 172.25.5.100 up
    route add -host 172.25.5.100 dev tunl0
     echo "1" >/proc/sys/net/ipv4/conf/tunl0/arp_ignore
     echo "2" >/proc/sys/net/ipv4/conf/tunl0/arp_announce
     echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
     echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce

3.测试，通过外部主机访问http://172.25.5.100查看页面呈现
