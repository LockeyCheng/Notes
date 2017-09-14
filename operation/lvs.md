### LVS

LVS是 Linux Virtual Server 的简称，也就是Linux虚拟服务器。LVS 是一个实现负载均衡集群的开源软件项目，LVS架构从逻辑上可分为调度层、Server集群层和共享存储。

### LVS的组成
LVS 由2部分程序组成，包括 ipvs 和 ipvsadm。

1. ipvs(ip virtual server)：一段代码工作在内核空间，叫ipvs，是真正生效实现调度的代码。
2. ipvsadm：另外一段是工作在用户空间，叫ipvsadm，负责为ipvs内核框架编写规则，定义谁是集群服务，而谁是后端真实的服务器(Real Server)

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

下边是LVS的三种工作模式

### LVS的NAT模式
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

缺陷：对Director Server压力会比较大，请求和响应都需经过director server


1.实验环境

三台主机，一台作为 director（双ip 192.168.1.6和172.6.6.8）
两台作为 real server，ip分别为172.6.6.6和172.6.6.7，并且需要把两个 real server 的内网网关设置为 director 的内网 ip(172.6.6.8)

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

1.实验环境

Director节点：  (eth0 172.6.6.8  vip eth0:0 172.6.6.9)

Real server1： (eth0 172.6.6.6 vip lo:0 172.6.6.9)

Real server2： (eth0 172.6.6.7 vip lo:0 172.6.6.9)

2.安装和配置

两个 real server 上都安装 nginx 服务，使用curl -I locahost命令测试服务正常运行。

Director 上安装 ipvsadm

    [root@lockey ~]# yum install -y ipvsadm

3.Director 上编辑 nat 实现脚本然后运行配置

    #! /bin/bash
    echo 1 > /proc/sys/net/ipv4/ip_forward
    ipv=/sbin/ipvsadm
    vip=172.6.6.9
    rs1=172.6.6.6
    rs2=172.6.6.7
    ifconfig eth0:0 down
    ifconfig eth0:0 $vip broadcast $vip netmask 255.255.255.255 up
    route add -host $vip dev eth0:0
    $ipv -C
    $ipv -A -t $vip:80 -s wrr
    $ipv -a -t $vip:80 -r $rs1:80 -g -w 3
    $ipv -a -t $vip:80 -r $rs2:80 -g -w 1

4.在2台real server 上配置并运行脚本：

    #! /bin/bash
    vip=172.6.6.9
    ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up
    route add -host $vip lo:0
    echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
    echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce

5.在Director主机上通过浏览器测试2台real server的web内容http://172.6.6.9，测试结果与NAT的结果类似


### LVS 与 keepalive

LVS可以实现负载均衡，但是不能够进行健康检查，比如一个real server出现故障，LVS 仍然会把请求转发给故障的real server服务器，这样就会导致请求的无效性。keepalive 软件可以进行健康检查，而且能同时实现 LVS 的高可用性，解决 LVS 单点故障的问题。
