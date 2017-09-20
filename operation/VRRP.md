### 1. VRRP简介
用一张图说明VRRP出现的原因：

![这里写图片描述](http://img.blog.csdn.net/20170921000746155?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**VRRP**（Virtual Router Redundancy Protocol，**虚拟路由器冗余协议**）将可以承担网关功能的路由器加入到备份组中，形成一台虚拟路由器，由VRRP的选举机制决定哪台路由器承担转发任务，局域网内的主机只需将虚拟路由器配置为缺省网关。
![这里写图片描述](http://img.blog.csdn.net/20170921000809160?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

VRRP是一种容错协议，在提高可靠性的同时，简化了主机的配置。在具有多播或广播能力的局域网（如以太网）中，借助VRRP能在某台路由器出现故障时仍然提供高可靠的缺省链路，有效避免单一链路发生故障后网络中断的问题，而无需修改动态路由协议、路由发现协议等配置信息

**相关术语**

-  虚拟路由器：由一个Master路由器和多个Backup路由器组成。主机将虚拟路由器当作默认网关。

- VRID：虚拟路由器的标识。有相同VRID的一组路由器构成一个虚拟路由器。

- Master路由器：虚拟路由器中承担报文转发任务的路由器。

- Backup路由器：Master路由器出现故障时，能够代替Master路由器工作的路由器。

- 虚拟IP地址：虚拟路由器的IP地址。一个虚拟路由器可以拥有一个或多个IP地址。

- IP地址拥有者：接口IP地址与虚拟IP地址相同的路由器被称为IP地址拥有者。

-  虚拟MAC地址：一个虚拟路由器拥有一个虚拟MAC地址。虚拟MAC地址的格式为00-00-5E-00-01-{VRID}。通常情况下，虚拟路由器回应ARP请求使用的是虚拟MAC地址，只有虚拟路由器做特殊配置的时候，才回应接口的真实MAC地址。

- 优先级：VRRP根据优先级来确定虚拟路由器中每台路由器的地位。

- 非抢占方式：如果Backup路由器工作在非抢占方式下，则只要Master路由器没有出现故障，Backup路由器即使随后被配置了更高的优先级也不会成为Master路由器。

-  抢占方式：如果Backup路由器工作在抢占方式下，当它收到VRRP报文后，会将自己的优先级与通告报文中的优先级进行比较。如果自己的优先级比当前的Master路由器的优先级高，就会主动抢占成为Master路由器；否则，将保持Backup状态。

**VRRP定时器**

VRRP定时器分为两种：VRRP通告报文间隔时间定时器和VRRP抢占延迟时间定时器。

1. VRRP通告报文时间间隔定时器

	VRRP备份组中的Master路由器会定时发送VRRP通告报文，通知备份组内的路由器自己工作正常。

	用户可以通过设置VRRP定时器来调整Master路由器发送VRRP通告报文的时间间隔。如果Backup路由器在等待了3个间隔时间后，依然没有收到VRRP通告报文，则认为自己是Master路由器，并对外发送VRRP通告报文，重新进行Master路由器的选举。

2. VRRP抢占延迟时间定时器

	为了避免备份组内的成员频繁进行主备状态转换，让Backup路由器有足够的时间搜集必要的信息（如路由信息），Backup路由器接收到优先级低于本地优先级的通告报文后，不会立即抢占成为Master，而是等待一定时间——抢占延迟时间后，才会对外发送VRRP通告报文取代原来的Master路由器

### 2. VRRP备份组简介

VRRP将局域网内的一组路由器划分在一起，称为一个备份组。备份组由一个Master路由器和多个Backup路由器组成，功能上相当于一台虚拟路由器。

**2.1 VRRP备份组具有以下特点：**

- 虚拟路由器具有IP地址，称为虚拟IP地址。局域网内的主机仅需要知道这个虚拟路由器的IP地址，并将其设置为缺省路由的下一跳地址。

- 网络内的主机通过这个虚拟路由器与外部网络进行通信。

- 备份组内的路由器根据优先级，选举出Master路由器，承担网关功能。其他路由器作为Backup路由器，当Master路由器发生故障时，取代Master继续履行网关职责，从而保证网络内的主机不间断地与外部网络进行通信。

**2.2 备份组中路由器的优先级**

	VRRP根据优先级来确定备份组中每台路由器的角色（Master路由器或Backup路由器）。优先级越高，则越有可能成为Master路由器。

	VRRP优先级的取值范围为0到255（数值越大表明优先级越高），可配置的范围是1到254，优先级0为系统保留给特殊用途来使用，255则是系统保留给IP地址拥有者。当路由器为IP地址拥有者时，其优先级始终为255。因此，当备份组内存在IP地址拥有者时，只要其工作正常，则为Master路由器。

**2.3 备份组中路由器的工作方式**

备份组中的路由器具有以下两种工作方式：

- 非抢占方式：如果备份组中的路由器工作在非抢占方式下，则只要Master路由器没有出现故障，Backup路由器即使随后被配置了更高的优先级也不会成为Master路由器。

- 抢占方式：如果备份组中的路由器工作在抢占方式下，它一旦发现自己的优先级比当前的Master路由器的优先级高，就会对外发送VRRP通告报文。导致备份组内路由器重新选举Master路由器，并最终取代原有的Master路由器。相应地，原来的Master路由器将会变成Backup路由器。

**2. 4 备份组中路由器的认证方式**

为了防止非法用户构造报文攻击备份组，VRRP通过在VRRP报文中增加认证字的方式，验证接收到的VRRP报文。VRRP提供了两种认证方式：

- simple：简单字符认证。发送VRRP报文的路由器将认证字填入到VRRP报文中，而收到VRRP报文的路由器会将收到的VRRP报文中的认证字和本地配置的认证字进行比较。如果认证字相同，则认为接收到的报文是真实、合法的VRRP报文；否则认为接收到的报文是一个非法报文。

- md5：MD5认证。发送VRRP报文的路由器利用认证字和MD5算法对VRRP报文进行摘要运算，运算结果保存在Authentication Header（认证头）中。收到VRRP报文的路由器会利用认证字和MD5算法进行同样的运算，并将运算结果与认证头的内容进行比较。如果相同，则认为接收到的报文是真实、合法的VRRP报文；否则认为接收到的报文是一个非法报文。

在一个安全的网络中，用户也可以不设置认证方式

### 3.VRRP工作过程

(1) 路由器使能VRRP功能后，会根据优先级确定自己在备份组中的角色。优先级高的路由器成为Master路由器，优先级低的成为Backup路由器。Master路由器定期发送VRRP通告报文，通知备份组内的其他路由器自己工作正常；Backup路由器则启动定时器等待通告报文的到来。

(2) 在抢占方式下，当Backup路由器收到VRRP通告报文后，会将自己的优先级与通告报文中的优先级进行比较。如果大于通告报文中的优先级，则成为Master路由器；否则将保持Backup状态。

(3) 在非抢占方式下，只要Master路由器没有出现故障，备份组中的路由器始终保持Master或Backup状态，Backup路由器即使随后被配置了更高的优先级也不会成为Master路由器。

(4) 如果Backup路由器的定时器超时后仍未收到Master路由器发送来的VRRP通告报文，则认为Master路由器已经无法正常工作，此时Backup路由器会认为自己是Master路由器，并对外发送VRRP通告报文。备份组内的路由器根据优先级选举出Master路由器，承担报文的转发功能。

### 4 VRRP应用（以基于IPv4的VRRP为例）

**4.1 主备备份**

	主备备份方式表示转发任务仅由Master路由器承担。当Master路由器出现故障时，才会从其他Backup路由器选举出一个接替工作。主备备份方式仅需要一个备份组，不同路由器在该备份组中拥有不同优先级，优先级最高的路由器将成为Master路由器。
![这里写图片描述](http://img.blog.csdn.net/20170921001830760?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

	初始情况下，Router A为Master路由器并承担转发任务，Router B和Router C是Backup路由器且都处于就绪监听状态。如果Router A发生故障，则备份组内处于Backup状态的Router B和Router C路由器将根据优先级选出一个新的Master路由器，这个新Master路由器继续向网络内的主机提供路由服务

**4.2 负载分担**

	在路由器的一个接口上可以创建多个备份组，使得该路由器可以在一个备份组中作为Master路由器，在其他的备份组中作为Backup路由器。

	负载分担方式是指多台路由器同时承担业务，因此负载分担方式需要两个或者两个以上的备份组，每个备份组都包括一个Master路由器和若干个Backup路由器，各备份组的Master路由器各不相同
![这里写图片描述](http://img.blog.csdn.net/20170921001957100?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
同一台路由器同时加入多个VRRP备份组，在不同备份组中有不同的优先级。

图中有三个备份组存在：

- 备份组1：对应虚拟路由器1。Router A作为Master路由器，Router B和Router C作为Backup路由器。

- 备份组2：对应虚拟路由器2。Router B作为Master路由器，Router A和Router C作为Backup路由器。

- 备份组3：对应虚拟路由器3。Router C作为Master路由器，Router A和Router B作为Backup路由器。

为了实现业务流量在Router A、Router B和Router C之间进行负载分担，需要将局域网内的主机的缺省网关分别设置为虚拟路由器1、2和3。在配置优先级时，需要确保三个备份组中各路由器的VRRP优先级形成交叉对应。
