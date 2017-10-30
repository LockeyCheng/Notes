因为GFW的关系，分享一下k8s所有组件的安装包给没有VPN的伙伴们： https://pan.baidu.com/s/1i5vMXDV

本文将介绍使用二进制部署 kubernetes 集群的所有步骤，同时开启集群的TLS安全认证；

**实验环境明细：**

关闭所有节点的SELinux以及防火墙
三台主机做好本地解析

Master：192.168.1.103

	kubectl
	kube-apiserver
	kube-scheduler
	kube-controller-manager

Node：192.168.1.103、192.168.1.106、192.168.1.107

	etcd,docker,Flanneld
	kubelet和kube-proxy
，将安装包下载后，解压进入以下目录然后执行以下命令(master和节点选各自需要的组件进行拷贝)

    cp -rp ./{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/

![这里写图片描述](http://img.blog.csdn.net/20171031032614854?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### 1. 创建TLS证书和秘钥
**二进制源码包安装 CFSSL**



	wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
	chmod +x cfssl_linux-amd64
	mv cfssl_linux-amd64 /usr/local/bin/cfssl

	wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
	chmod +x cfssljson_linux-amd64
	mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

	wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
	chmod +x cfssl-certinfo_linux-amd64
	mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

**创建 CA (Certificate Authority)**


	cd /etc/kubernetes/ssl
	cfssl print-defaults config > config.json
	cfssl print-defaults csr > csr.json

根据config.json文件的格式创建如下的ca-config.json文件

cat ca-config.json

	{
	  "signing": {
	    "default": {
	      "expiry": "87600h"
	    },
	    "profiles": {
	      "kubernetes": {
	        "usages": [
	            "signing",
	            "key encipherment",
	            "server auth",
	            "client auth"
	        ],
	        "expiry": "87600h"
	      }
	    }
	  }
	}


**创建 CA 证书签名请求**

创建 ca-csr.json 文件，内容如下：

	{
	  "CN": "kubernetes",
	  "key": {
	    "algo": "rsa",
	    "size": 2048
	  },
	  "names": [
	    {
	      "C": "CN",
	      "ST": "BeiJing",
	      "L": "BeiJing",
	      "O": "k8s",
	      "OU": "System"
	    }
	  ]
	}

**生成 CA 证书和私钥**

	$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca

**创建 kubernetes 证书**

[root@lockey103 ssl]# cat kubernetes-csr.json

	{
	    "CN": "kubernetes",
	    "hosts": [
	      "127.0.0.1",
	      "192.168.1.103",
	      "192.168.1.106",
	      "192.168.1.107",
	      "10.254.0.1",
	      "kubernetes",
	      "kubernetes.default",
	      "kubernetes.default.svc",
	      "kubernetes.default.svc.cluster",
	      "kubernetes.default.svc.cluster.local"
	    ],
	    "key": {
	        "algo": "rsa",
	        "size": 2048
	    },
	    "names": [
	        {
	            "C": "CN",
	            "ST": "BeiJing",
	            "L": "BeiJing",
	            "O": "k8s",
	            "OU": "System"
	        }
	    ]
	}


**生成 kubernetes 证书和私钥**

	$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes


**创建 admin 证书**

cat admin-csr.json

	{
	  "CN": "admin",
	  "hosts": [],
	  "key": {
	    "algo": "rsa",
	    "size": 2048
	  },
	  "names": [
	    {
	      "C": "CN",
	      "ST": "BeiJing",
	      "L": "BeiJing",
	      "O": "system:masters",
	      "OU": "System"
	    }
	  ]
	}

**生成 admin 证书和私钥**

	$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

**创建 kube-proxy 证书**

cat kube-proxy-csr.json

	{
	  "CN": "system:kube-proxy",
	  "hosts": [],
	  "key": {
	    "algo": "rsa",
	    "size": 2048
	  },
	  "names": [
	    {
	      "C": "CN",
	      "ST": "BeiJing",
	      "L": "BeiJing",
	      "O": "k8s",
	      "OU": "System"
	    }
	  ]
	}


	$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy

**校验证书**

以 kubernetes 证书为例

使用 opsnssl 命令

	$ openssl x509  -noout -text -in  kubernetes.pem

或者使用 cfssl-certinfo 命令

	$ cfssl-certinfo -cert kubernetes.pem

**分发证书**

scp -r /etc/kubernetes/ssl/*.pem 192.168.1.106/107:/etc/kubernetes/ssl/

### 2.创建 kubectl kubeconfig 文件

以下操作只需要在master节点上执行，生成的*.kubeconfig文件可以直接拷贝到node节点的/etc/kubernetes目录下。

	export KUBE_APISERVER="https://192.168.1.103:6443"
	# 设置集群参数
	kubectl config set-cluster kubernetes \
	  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
	  --embed-certs=true \
	  --server=${KUBE_APISERVER}
	# 设置客户端认证参数
	kubectl config set-credentials admin \
	  --client-certificate=/etc/kubernetes/ssl/admin.pem \
	  --embed-certs=true \
	  --client-key=/etc/kubernetes/ssl/admin-key.pem
	# 设置上下文参数
	kubectl config set-context kubernetes \
	  --cluster=kubernetes \
	  --user=admin
	# 设置默认上下文
	kubectl config use-context kubernetes
![这里写图片描述](http://img.blog.csdn.net/20171031032907742?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


**创建 TLS Bootstrapping Token**

Token可以是任意的包涵128 bit的字符串，可以使用安全的随机数发生器生成。

	export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
	cat > token.csv <<EOF
	${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
	EOF

注意：在进行后续操作前请检查 token.csv 文件，确认其中的 ${BOOTSTRAP_TOKEN} 环境变量已经被真实的值替换。



	# 设置集群参数
	kubectl config set-cluster kubernetes \
	  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
	  --embed-certs=true \
	  --server=${KUBE_APISERVER} \
	  --kubeconfig=bootstrap.kubeconfig

	# 设置客户端认证参数
	kubectl config set-credentials kubelet-bootstrap \
	  --token=${BOOTSTRAP_TOKEN} \
	  --kubeconfig=bootstrap.kubeconfig

	# 设置上下文参数
	kubectl config set-context default \
	  --cluster=kubernetes \
	  --user=kubelet-bootstrap \
	  --kubeconfig=bootstrap.kubeconfig

	# 设置默认上下文
	kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

--embed-certs 为 true 时表示将 certificate-authority 证书写入到生成的 bootstrap.kubeconfig 文件中；
设置客户端认证参数时没有指定秘钥和证书，后续由 kube-apiserver 自动生成；

**创建 kube-proxy kubeconfig 文件**


	# 设置集群参数
	kubectl config set-cluster kubernetes \
	  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
	  --embed-certs=true \
	  --server=${KUBE_APISERVER} \
	  --kubeconfig=kube-proxy.kubeconfig
	# 设置客户端认证参数
	kubectl config set-credentials kube-proxy \
	  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
	  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
	  --embed-certs=true \
	  --kubeconfig=kube-proxy.kubeconfig
	# 设置上下文参数
	kubectl config set-context default \
	  --cluster=kubernetes \
	  --user=kube-proxy \
	  --kubeconfig=kube-proxy.kubeconfig
	# 设置默认上下文
	kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

**分发 kubeconfig 文件**

将两个 kubeconfig 文件分发到所有 Node 机器的 /etc/kubernetes/ 目录

	scp bootstrap.kubeconfig kube-proxy.kubeconfig 192.168.1.106/107:/etc/kubernetes/

### 3.创建高可用 etcd 集群（三节点同步执行）

**下载二进制文件**

	wget https://github.com/coreos/etcd/releases/download/v3.1.5/etcd-v3.1.5-linux-amd64.tar.gz
	tar -xvf etcd-v3.1.5-linux-amd64.tar.gz
	mv etcd-v3.1.5-linux-amd64/etcd* /usr/local/bin

**创建 etcd 的 systemd unit 文件**

	注意替换IP地址为你自己的etcd集群的主机IP。
[root@lockey103 bin]# cat /etc/systemd/system/etcd.service

	[Unit]
	Description=Etcd Server
	After=network.target
	After=network-online.target
	Wants=network-online.target
	Documentation=https://github.com/coreos

	[Service]
	Type=notify
	WorkingDirectory=/var/lib/etcd/
	EnvironmentFile=-/etc/etcd/etcd.conf
	ExecStart=/usr/local/bin/etcd \
	  --name ${ETCD_NAME} \
	  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
	  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
	  --peer-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
	  --peer-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
	  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
	  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
	  --initial-advertise-peer-urls ${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
	  --listen-peer-urls ${ETCD_LISTEN_PEER_URLS} \
	  --listen-client-urls ${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
	  --advertise-client-urls ${ETCD_ADVERTISE_CLIENT_URLS} \
	  --initial-cluster-token ${ETCD_INITIAL_CLUSTER_TOKEN} \
	  --initial-cluster infra1=https://192.168.1.103:2380,infra2=https://192.168.1.106:2380,infra3=https://192.168.1.107:2380 \
	  --initial-cluster-state new \
	  --data-dir=${ETCD_DATA_DIR}
	Restart=on-failure
	RestartSec=5
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target


指定 etcd 的工作目录为 /var/lib/etcd，数据目录为 /var/lib/etcd，需在启动服务前创建这两个目录；


**环境变量配置文件/etc/etcd/etcd.conf**

cat /etc/etcd/etcd.conf（各节点对应替换ip）

	ETCD_NAME=infra1
	ETCD_DATA_DIR="/var/lib/etcd"
	ETCD_LISTEN_PEER_URLS="https://192.168.1.103:2380"
	ETCD_LISTEN_CLIENT_URLS="https://192.168.1.103:2379"

	#[cluster]
	ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.103:2380"
	ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
	ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.103:2379"

三节点同时启动 etcd 服务


	systemctl daemon-reload
	systemctl start etcd

**验证服务**
![这里写图片描述](http://img.blog.csdn.net/20171031034158597?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### 4.部署master节点

**4.1 配置和启动 kube-apiserver**


serivce配置文件/usr/lib/systemd/system/kube-apiserver.service内容：

	[Unit]
	Description=Kubernetes API Service
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes
	After=network.target
	After=etcd.service

	[Service]
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/apiserver
	ExecStart=/usr/local/bin/kube-apiserver \
	        $KUBE_LOGTOSTDERR \
	        $KUBE_LOG_LEVEL \
	        $KUBE_ETCD_SERVERS \
	        $KUBE_API_ADDRESS \
	        $KUBE_API_PORT \
	        $KUBELET_PORT \
	        $KUBE_ALLOW_PRIV \
	        $KUBE_SERVICE_ADDRESSES \
	        $KUBE_ADMISSION_CONTROL \
	        $KUBE_API_ARGS
	Restart=on-failure
	Type=notify
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
/etc/kubernetes/config文件的内容为：

	KUBE_LOGTOSTDERR="--logtostderr=true"

	# journal message level, 0 is debug
	KUBE_LOG_LEVEL="--v=0"
	#
	# # Should this cluster be allowed to run privileged docker containers
	KUBE_ALLOW_PRIV="--allow-privileged=true"
	#
	# # How the controller-manager, scheduler, and proxy find the apiserver

	KUBE_MASTER="--master=http://192.168.1.103:8080"


apiserver配置文件/etc/kubernetes/apiserver内容为：

	## The address on the local server to listen to.
	##KUBE_API_ADDRESS="--insecure-bind-address=sz-pg-oam-docker-test-001.tendcloud.com"
	KUBE_API_ADDRESS="--advertise-address=192.168.1.103 --bind-address=192.168.1.103 --insecure-bind-address=192.168.1.103"
	##
	### The port on the local server to listen on.
	#KUBE_API_PORT="--port=8080"
	##
	### Port minions listen on
	#KUBELET_PORT="--kubelet-port=10250"
	##
	### Comma separated list of nodes in the etcd cluster
	KUBE_ETCD_SERVERS="--etcd-servers=https://192.168.1.103:2379,https://192.168.1.106:2379,https://192.168.1.107:2379"
	##
	### Address range to use for services
	KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
	##
	### default admission control policies
	KUBE_ADMISSION_CONTROL="--admission-control=ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota"
	##
	### Add your own!
	KUBE_API_ARGS="--authorization-mode=RBAC --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true --experimental-bootstrap-token-auth --token-auth-file=/etc/kubernetes/token.csv --service-node-port-range=30000-32767 --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem --etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem --enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/lib/audit.log --event-ttl=1h"

**启动kube-apiserver**

	systemctl daemon-reload

	systemctl start kube-apiserver

**验证**
![这里写图片描述](http://img.blog.csdn.net/20171031035300853?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**4.2 配置和启动 kube-controller-manager**

创建 kube-controller-manager的serivce配置文件

cat /usr/lib/systemd/system/kube-controller-manager.service

	[Unit]
	Description=Kubernetes Controller Manager
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes

	[Service]
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/controller-manager
	ExecStart=/root/local/bin/kube-controller-manager \
	        $KUBE_LOGTOSTDERR \
	        $KUBE_LOG_LEVEL \
	        $KUBE_MASTER \
	        $KUBE_CONTROLLER_MANAGER_ARGS
	Restart=on-failure
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target

配置文件/etc/kubernetes/controller-manager。

	KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 --service-cluster-ip-range=10.254.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem --root-ca-file=/etc/kubernetes/ssl/ca.pem --leader-elect=true"
	[root@lockey103 bin]# cat /etc/kubernetes/controller-manager
	KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 --service-cluster-ip-range=10.254.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem --root-ca-file=/etc/kubernetes/ssl/ca.pem --leader-elect=true"


**启动 kube-controller-manager**

	systemctl daemon-reload

	systemctl start kube-controller-manager

**4.3 配置和启动 kube-scheduler**

创建 kube-scheduler的serivce配置文件

文件路径/usr/lib/systemd/system/kube-scheduler.service

	[Unit]
	Description=Kubernetes Scheduler Plugin
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes

	[Service]
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/scheduler
	ExecStart=/root/local/bin/kube-scheduler \
	            $KUBE_LOGTOSTDERR \
	            $KUBE_LOG_LEVEL \
	            $KUBE_MASTER \
	            $KUBE_SCHEDULER_ARGS
	Restart=on-failure
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target

配置文件/etc/kubernetes/scheduler

	###
	# kubernetes scheduler config

	# default config should be adequate

	# Add your own!
	KUBE_SCHEDULER_ARGS="--leader-elect=true --address=127.0.0.1"


启动 kube-scheduler

	systemctl daemon-reload

	systemctl start kube-scheduler


**验证 master 节点功能**
![这里写图片描述](http://img.blog.csdn.net/20171031035700496?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### 5. 部署node节点(各节点同步，并且对应修改ip)

**5.1 配置Flanneld**

	yum install flannel -y

service配置文件/usr/lib/systemd/system/flanneld.service

	[Unit]
	Description=Flanneld overlay address etcd agent
	After=network.target
	After=network-online.target
	Wants=network-online.target
	After=etcd.service
	Before=docker.service

	[Service]
	Type=notify
	EnvironmentFile=/etc/sysconfig/flanneld
	EnvironmentFile=-/etc/sysconfig/docker-network
	ExecStart=/usr/bin/flanneld-start \
	  -etcd-endpoints=${ETCD_ENDPOINTS} \
	  -etcd-prefix=${ETCD_PREFIX} \
	  $FLANNEL_OPTIONS
	ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
	Restart=on-failure

	[Install]
	WantedBy=multi-user.target
	RequiredBy=docker.service


/etc/sysconfig/flanneld配置文件

	ETCD_ENDPOINTS="https://192.168.1.103:2379,https://192.168.1.106:2379,https://192.168.1.107:2379"

	# etcd config key.  This is the configuration key that flannel queries
	# # For address range assignment
	ETCD_PREFIX="/kube-centos/network"
	#
	# # Any additional options that you want to pass
	FLANNEL_OPTIONS="-etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem -etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem"


**在etcd中创建网络配置**

执行下面的命令为docker分配IP地址段:
![这里写图片描述](http://img.blog.csdn.net/20171031040150624?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


**配置Docker**

如果你不是使用yum安装的flanneld，那么需要下载flannel github release中的tar包，解压后会获得一个mk-docker-opts.sh文件。

这个文件是用来Generate Docker daemon options based on flannel env file。

执行./mk-docker-opts.sh -i将会生成如下两个文件环境变量文件。

/run/flannel/subnet.env

	FLANNEL_NETWORK=172.30.0.0/16
	FLANNEL_SUBNET=172.30.46.1/24
	FLANNEL_MTU=1450
	FLANNEL_IPMASQ=false
/run/docker_opts.env

	DOCKER_OPT_BIP="--bip=172.30.46.1/24"
	DOCKER_OPT_IPMASQ="--ip-masq=true"
	DOCKER_OPT_MTU="--mtu=1450"
设置docker0网桥的IP地址

	source /run/flannel/subnet.env
	ifconfig docker0 $FLANNEL_SUBNET

![这里写图片描述](http://img.blog.csdn.net/20171031040336521?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


同时在 docker 的配置文件 docker.service 中增加环境变量配置：

	EnvironmentFile=-/run/flannel/docker
	EnvironmentFile=-/run/docker_opts.env
	EnvironmentFile=-/run/flannel/subnet.env

启动flannel

	systemctl daemon-reload
	systemctl start flanneld
现在查询etcd中的内容可以看到：
![这里写图片描述](http://img.blog.csdn.net/20171031040521185?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 6. 为节点安装和配置 kubelet(各节点同步执行)

cd /etc/kubernetes

	kubectl create clusterrolebinding kubelet-bootstrap \
	  --clusterrole=system:node-bootstrapper \
	  --user=kubelet-bootstrap


**创建 kubelet 的service配置文件**

文件位置/usr/lib/systemd/system/kubelet.service

	[Unit]
	Description=Kubernetes Kubelet Server
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes
	After=docker.service
	Requires=docker.service

	[Service]
	WorkingDirectory=/var/lib/kubelet
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/kubelet
	ExecStart=/usr/local/bin/kubelet \
	            $KUBE_LOGTOSTDERR \
	            $KUBE_LOG_LEVEL \
	            $KUBELET_API_SERVER \
	            $KUBELET_ADDRESS \
	            $KUBELET_PORT \
	            $KUBELET_HOSTNAME \
	            $KUBE_ALLOW_PRIV \
	            $KUBELET_POD_INFRA_CONTAINER \
	            $KUBELET_ARGS
	Restart=on-failure

	[Install]
	WantedBy=multi-user.target

kubelet的配置文件/etc/kubernetes/kubelet。其中的IP地址更改为你的每台node节点的IP地址。

注意：/var/lib/kubelet需要手动创建。

	KUBELET_ADDRESS="--address=192.168.1.103"
	#
	## The port for the info server to serve on
	#KUBELET_PORT="--port=10250"
	#
	## You may leave this blank to use the actual hostname
	KUBELET_HOSTNAME="--hostname-override=192.168.1.103"
	#
	## location of the api-server
	KUBELET_API_SERVER="--api-servers=http://192.168.1.103:8080"
	#
	## pod infrastructure container
	KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=sz-pg-oam-docker-hub-001.tendcloud.com/library/pod-infrastructure:rhel7"
	#
	## Add your own!
	KUBELET_ARGS="--cgroup-driver=systemd --cluster-dns=10.254.0.2 --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --require-kubeconfig --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local --hairpin-mode promiscuous-bridge --serialize-image-pulls=false"

**启动kublet**

	systemctl daemon-reload

	systemctl start kubelet

**通过 kublet 的 TLS 证书请求**

![这里写图片描述](http://img.blog.csdn.net/20171031040848436?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20171031040913916?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 7. 为节点配置 kube-proxy(各节点同步执行)
**创建 kube-proxy 的service配置文件**

文件路径/usr/lib/systemd/system/kube-proxy.service

	[Unit]
	Description=Kubernetes Kube-Proxy Server
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes
	After=network.target

	[Service]
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/proxy
	ExecStart=/root/local/bin/kube-proxy \
	        $KUBE_LOGTOSTDERR \
	        $KUBE_LOG_LEVEL \
	        $KUBE_MASTER \
	        $KUBE_PROXY_ARGS
	Restart=on-failure
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target

**kube-proxy配置文件/etc/kubernetes/proxy**

	# Add your own!
	KUBE_PROXY_ARGS="--bind-address=192.168.1.103 --hostname-override=192.168.1.103 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"


**启动 kube-proxy**

	systemctl daemon-reload

	systemctl start kube-proxy
	systemctl status kube-proxy
![这里写图片描述](http://img.blog.csdn.net/20171031041155866?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG9ja2V5MjM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**验证测试**

	[root@lockey103 ssl]# kubectl run nginx --replicas=2 --labels="run=load-balancer-example" --image=nginx:latest

	[root@lockey103 ssl]# kubectl expose deployment nginx --type=NodePort --name=example-service

	[root@lockey103 ssl]# kubectl describe svc example-service
