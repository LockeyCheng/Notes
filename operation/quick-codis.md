### 1. 安装go环境

[root@lockey128-vm4 ~]# tar -zxvf go1.5.1.linux-amd64.tar.gz -C /usr/local/

**设置go环境变量**

[root@lockey128-vm4 ~]#  vim /etc/profile#最后添加以下三行

	export GOROOT=/usr/local/go
	export PATH=$GOROOT/bin:$PATH
	export GOPATH=/home/user/go


 [root@lockey128-vm4 ~]#  source /etc/profile
[root@lockey128-vm4 ~]#  go version

	go version go1.5.1 linux/amd64
**测试一个go程序**

[root@lockey128-vm4 ~]#  vim halo.go
[root@lockey128-vm4 ~]#  go run halo.go
  ![867  clear](http://img.blog.csdn.net/20171019012651432?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 2. 安装Codis

	[root@lockey128-vm4 ~]# $ mkdir -p $GOPATH/src/github.com/CodisLabs
	[root@lockey128-vm4 ~]# unzip codis-release3.2.zip
	[root@lockey128-vm4 ~]# mv codis-release3.2 $GOPATH/src/github.com/CodisLabs/codis

**执行make命令**

 [root@lockey128-vm4 ~]#  cd $GOPATH/src/github.com/CodisLabs/codis

[root@lockey128-vm4 codis]# make

**等待少许时间make结束后，执行快速启动**

	[root@lockey128-vm4 codis]#  ./admin/codis-dashboard-admin.sh start

	[root@lockey128-vm4 codis]#  ./admin/codis-proxy-admin.sh start

	[root@lockey128-vm4 codis]#  ./admin/codis-server-admin.sh start

	[root@lockey128-vm4 codis]#  ./admin/codis-fe-admin.sh start
**查看各进程是否正常：**


![这里写图片描述](http://img.blog.csdn.net/20171019013530237?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**查看服务端口：**

![这里写图片描述](http://img.blog.csdn.net/20171019013925854?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**访问测试：**

**Dashboard 	192.168.1.9:18080**

![这里写图片描述](http://img.blog.csdn.net/20171019013554104?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**Proxy 192.168.1.9:19000 **

![这里写图片描述](http://img.blog.csdn.net/20171019013602212?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20171019013609777?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20171019013617701?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


**添加组：**
需要填写组号、ip:port->New Group->Add Server

![这里写图片描述](http://img.blog.csdn.net/20171019014216066?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


**通过fe初始化slot**

    新增的集群 slot 状态是 offline，因此我们需要对它进行初始化（将 1024 个 slot 分配到各个 group），而初始化最快的方法可通过 fe 提供的 rebalance all slots 按钮来做，如下图所示，点击此按钮，我们即快速完成了一个集群的搭建。

![这里写图片描述](http://img.blog.csdn.net/20171019014529195?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
