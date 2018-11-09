## kubernetes v1.11.3 手动安装

说明： 本教程是对[《Kubernetes v1.8.x 全手動苦工安裝教學》](https://kairen.github.io/2017/10/27/kubernetes/deploy/manual-v1.8/)或者[《Kubernetes 1.8.x 全手动安装教程》](https://www.kubernetes.org.cn/3096.html)的修改补充，因为感觉原文的安装有点混乱，所以这里特意整理了一下，其中有些文字直接使用原文的，望见谅。

### 环境准备

本次安装的版本为：
	
	kubernetes v1.11.3
	etcd v3.2.9
	CNI v0.7.1
	calico v3.1
	docker v18.06.0-ce

预先准备信息

本教程的节点信息如下：

| IP address  |  Role | hostname  |
| ------------ | ------------ | ------------ |
| 10.61.0.91  | master,etcd node  | vmnode1  |
| 10.61.0.92  |  node,etcd node|  vmnode2 |
| 10.61.0.93  | node,etcd node  |  vmnode3 |
| 10.61.0.94 |  node |  vmnode4 |
| 10.61.0.95 |  node |   vmnode5|

Ingress Vip（安装ingress server虚拟ip，没被使用的ip）:  10.61.0.96

说明：

	1.集群使用的网段为10.96.0.0/12，原文中是设置的这个网段，它是一个虚网段，所以我们不需要配置物理网卡上配置这个网段的ip，既然是虚网段，我们还是按照原作者选择的10.96.0.0/12的网段，并且使用其中的10.96.0.10作为集群ip。
	2.10.61.0.0/24这个网段是我的物理网段，如果这个网段需要根据实际情况做出相应的调整。
	3.etcd集群我这选3个节点，原文中选用一个节点，这个可靠性不高。

首先安装前要确认以下几项都准备好了：

1.节点需要彼此互通，并且vmnode1能够免密码登陆其他节点。

2.所有防火墙与selinux关闭，如Centos:

	systemctl stop firewalld && systemctl disable firewalld
	setenforce 0
	vim /etc/selinux/config
	SELINUX=disabled(/etc/selinux/config中此项变为disabled)

3.所有节点需要设定/etc/host解析到所有主机。(这里如果节点比较少，可以使用这种方式，如果节点比较多，可以考虑使用dnsmasq，或者bind)

	[root@vmnode1 ~]# cat /etc/hosts
	127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
	::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
	10.61.0.91 vmnode1
	10.61.0.92 vmnode2
	10.61.0.93 vmnode3
	10.61.0.94 vmnode4
	10.61.0.95 vmnode5

4.所有节点都安装docker引擎(以下步骤在所有节点上执行)：

	[root@vmnode1 ~]# yum install epel-release
	[root@vmnode1 ~]# yum install -y yum-utils device-mapper-persistent-data lvm2
	[root@vmnode1 ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
	[root@vmnode1 ~]# yum install docker-ce -y
	[root@vmnode1 ~]# systemctl enable docker && systemctl restart docker
	[root@vmnode1 ~]# docker version
	Client:
	 Version:      18.06.1-ce
	 API version:  1.37
	 Go version:   go1.9.5
	 Git commit:   9ee9f40
	 Built:        Thu Apr 26 07:20:16 2018
	 OS/Arch:      linux/amd64
	 Experimental: false
	 Orchestrator: swarm
	Server:
	 Engine:
	  Version:      18.06.1-ce
	  API version:  1.37 (minimum version 1.12)
	  Go version:   go1.9.5
	  Git commit:   9ee9f40
	  Built:        Thu Apr 26 07:23:58 2018
	  OS/Arch:      linux/amd64
	  Experimental: false


5.所有节点需要设定/etc/sysctl.d/k8s.conf的系统参数。

	[root@vmnode1 ~]# cat  /etc/sysctl.d/k8s.conf 
	net.ipv4.ip_forward = 1
	net.bridge.bridge-nf-call-ip6tables = 1
	net.bridge.bridge-nf-call-iptables = 1

6.所有节点必须保持时区和时间一致，否则etcd集群会出现时钟漂移，这里有两种方法，一种是本地安装一个ntp server，另一种是直接使用网上的ntp server，我这里就选择第二种（以下步骤在所有节点上进行），添加最后一行到/etc/crontab（首先确定ntpdate是否安装）：

首先更改时区：

	[root@vmnode1 ~]# date
	Sun Jan  7 20:41:06 EST 2018
	[root@vmnode1 ~]# rm -rf /etc/localtime 
	[root@vmnode1 ~]# ln -sv /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
	‘/etc/localtime’ -> ‘/usr/share/zoneinfo/Asia/Shanghai’
	[root@vmnode1 ~]# date
	Mon Jan  8 09:41:58 CST 2018

然后添加定时时间同步任务：

	[root@vmnode1 ~]# cat /etc/crontab
	SHELL=/bin/bash
	PATH=/sbin:/bin:/usr/sbin:/usr/bin
	MAILTO=root

	# For details see man 4 crontabs

	# Example of job definition:
	# .---------------- minute (0 - 59)
	# |  .------------- hour (0 - 23)
	# |  |  .---------- day of month (1 - 31)
	# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
	# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
	# |  |  |  |  |
	# *  *  *  *  * user-name  command to be executed

	*/2 * * * *  root  ntpdate cn.ntp.org.cn &> /dev/null

7.关闭所有节点上swap，否则kubelet服务无法启动。

* 关闭系统swap:

	[root@vmnode1 ~]# swapoff -a

* 如果/etc/fstab挂载了swap需要注释掉该行

8.安装ipvsadm:

在kubernetes v1.11.0 以后可以使用ipvs,这里我们使用ipvs，而不使用iptables.在每个节点上安装ipvsadm。

	[root@vmnode1 ~]# yum install ipvsadm -y

8.在vmnode1上安装cfssl工具，这将会用来建立 TLS certificates。

	[root@vmnode1 ~]# export CFSSL_URL="https://pkg.cfssl.org/R1.2"
	[root@vmnode1 ~]# wget "${CFSSL_URL}/cfssl_linux-amd64" -O /usr/local/bin/cfssl
	[root@vmnode1 ~]# wget "${CFSSL_URL}/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson
	[root@vmnode1 ~]# chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
	
8.准备需要用到的配置文件

	[root@vmnode1 tmp]# cd /opt
	[root@vmnode1 opt]# git clone https://github.com/happy2048/k8s-manual
	[root@vmnode1 ~]# echo "export K8S_CONFIG_FILES=/opt/k8s-manual/v1.11.3" >> ~/.bashrc
	[root@vmnode1 ~]# echo  "export KUBE_APISERVER=https://10.61.0.91:6443" >>  ~/.bashrc
	[root@vmnode1 ~]# echo  "export MYIP=10.61.0.91" >>  ~/.bashrc
	[root@vmnode1 ~]# source ~/.bashrc
	[root@vmnode1 ~]# echo $K8S_CONFIG_FILES
	/opt/k8s_config_files
	
### Etcd安装

在开始安装 Kubernetes 之前，需要先将一些必要系统创建完成，其中 Etcd 就是 Kubernetes 最重要的一环，Kubernetes 会将大部分信息储存于 Etcd 上，来提供给其他节点索取，以确保整个集群运作与沟通正常。

1.使用yum安装etcd（该步骤所有etcd节点上进行，这里我只展示在vmnode1的操作，其他etcd节点执行一样）

	[root@vmnode1 ~]# yum install epel-release -y
	[root@vmnode1 ~]# yum install etcd -y
	[root@vmnode1 ~]# mkdir /etc/etcd/ssl && chown etcd:etcd  /etc/etcd/ssl
	
2.建集群CA与Certificates（以下步骤在vmnode1进行)

2.1 建立/etc/etcd/ssl文件夹，然后进入目录完成以下操作。

	[root@vmnode1 etcd]#  cd /etc/etcd/ssl
	
2.2 复制ca-config.json与etcd-ca-csr.json文件到本目录下，并产生 CA 密钥（此步骤在vmnode1上进行）：

	[root@vmnode1 ssl]# cp $K8S_CONFIG_FILES/pki/ca-config.json  $K8S_CONFIG_FILES/pki/etcd-ca-csr.json ./
	[root@vmnode1 ssl]#  cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare etcd-ca
	2018/01/08 10:30:32 [INFO] generating a new CA key and certificate from CSR
	2018/01/08 10:30:32 [INFO] generate received request
	2018/01/08 10:30:32 [INFO] received CSR
	2018/01/08 10:30:32 [INFO] generating key: rsa-2048
	2018/01/08 10:30:32 [INFO] encoded CSR
	2018/01/08 10:30:32 [INFO] signed certificate with serial number 343991803789219532784889668850923759971181192623
	[root@vmnode1 ssl]# ls
	ca-config.json  etcd-ca.csr  etcd-ca-csr.json  etcd-ca-key.pem  etcd-ca.pem
	
ps: 本步骤中的两个json文件无需做更改,其中etcd-ca-csr.json中names字段的值解释如下：

	"C": country
	"L": locality or municipality (such as city or town name)
	"O": organisation
	"OU": organisational unit, such as the department responsible for owning the key; it can also be used for a "Doing Business As" (DBS) name
	"ST": the state or province
	
2.3 复制etcd-csr.json文件到本目录下，并产生 kube-apiserver certificate 证书(本步骤不能直接运行命令，需要对etcd-csr.json做相应的修改，文件修改说明如下)（此步骤在vmnode1上进行）：

	[root@vmnode1 ssl]# cp $K8S_CONFIG_FILES/pki/etcd-csr.json  ./
	[root@vmnode1 ssl]# cfssl gencert \
	-ca=etcd-ca.pem \
	-ca-key=etcd-ca-key.pem \
	-config=ca-config.json \
	-hostname=127.0.0.1,10.61.0.91,10.61.0.92,10.61.0.93 \
	-profile=kubernetes \
	etcd-csr.json | cfssljson -bare etcd

修改属主：

	[root@vmnode1 ~]# chown -R etcd:etcd /etc/etcd/ssl

ps: hostname选项需要将所有地etcd节点都写进去，否则生成的证书致使有的节点无法使用。

2.4 将生成的相关认证文件复制到所有etcd节点（vmnode1,vmnode2,vmnode3），使用如下脚本执行：

	[root@vmnode1 ssl]# cat /tmp/copy.sh
	#!/bin/bash
	for i in vmnode2 vmnode3;do
		ssh $i mkdir -pv /etc/etcd
		scp -r /etc/etcd/ssl $i:/etc/etcd
		ssh $i chown etcd:etcd -R /etc/etcd/ssl
	done

2.5 这里我选择静态文件配置etcd，当然也可以像原文那样选择直接在命令行配置，配置文件的地址为/etc/etcd/etcd.conf（配置文件的修改在所有的etcd节点上进行,下面是在vmnode1进行的，不同的节点配置文件的信息有点不同，具体如下）:

	[root@vmnode1 ssl]# cp $K8S_CONFIG_FILES/config/etcd.conf  /etc/etcd/etcd.conf
	[root@vmnode1 ssl]# scp /etc/etcd/etcd.conf vmnode2:/etc/etcd
	[root@vmnode1 ssl]# scp /etc/etcd/etcd.conf vmnode3:/etc/etcd
	[root@vmnode1 ssl]# cat /etc/etcd/etcd.conf
	#[Member]
	#ETCD_CORS=""
	ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
	#ETCD_WAL_DIR=""
	ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
	ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,https://0.0.0.0:2379"
	#ETCD_MAX_SNAPSHOTS="5"
	#ETCD_MAX_WALS="5"
	ETCD_NAME="etcd-node1"
	#ETCD_SNAPSHOT_COUNT="100000"
	#ETCD_HEARTBEAT_INTERVAL="100"
	#ETCD_ELECTION_TIMEOUT="1000"
	#ETCD_QUOTA_BACKEND_BYTES="0"
	#
	#[Clustering]
	ETCD_INITIAL_ADVERTISE_PEER_URLS="https://vmnode1:2380"
	ETCD_ADVERTISE_CLIENT_URLS="https://vmnode1:2379"
	#ETCD_DISCOVERY=""
	#ETCD_DISCOVERY_FALLBACK="proxy"
	#ETCD_DISCOVERY_PROXY=""
	#ETCD_DISCOVERY_SRV=""
	ETCD_INITIAL_CLUSTER="etcd-node1=https://vmnode1:2380,etcd-node2=https://vmnode2:2380,etcd-node3=https://vmnode3:2380"
	ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-1"
	ETCD_INITIAL_CLUSTER_STATE="new"
	#ETCD_STRICT_RECONFIG_CHECK="true"
	#ETCD_ENABLE_V2="true"
	#
	#[Proxy]
	#ETCD_PROXY="off"
	#ETCD_PROXY_FAILURE_WAIT="5000"
	#ETCD_PROXY_REFRESH_INTERVAL="30000"
	#ETCD_PROXY_DIAL_TIMEOUT="1000"
	#ETCD_PROXY_WRITE_TIMEOUT="5000"
	#ETCD_PROXY_READ_TIMEOUT="0"
	#
	#[Security]
	ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
	ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
	ETCD_CLIENT_CERT_AUTH="true"
	ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
	ETCD_AUTO_TLS="true"
	ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
	ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
	ETCD_PEER_CLIENT_CERT_AUTH="true"
	ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
	ETCD_PEER_AUTO_TLS="true"
	#
	#[Logging]
	#ETCD_DEBUG="false"
	#ETCD_LOG_PACKAGE_LEVELS=""
	#ETCD_LOG_OUTPUT="default"
	#
	#[Unsafe]
	#ETCD_FORCE_NEW_CLUSTER="false"
	#
	#[Version]
	#ETCD_VERSION="false"
	#ETCD_AUTO_COMPACTION_RETENTION="0"
	#
	#[Profiling]
	#ETCD_ENABLE_PPROF="false"
	#ETCD_METRICS="basic"
	#
	#[Auth]
	#ETCD_AUTH_TOKEN="simple"

配置文件说明：

	1.ETCD_DATA_DIR如果这个目录在还没启动etcd时就已经存在了，说明你曾经装过etcd，请把所有节点的/var/lib/etcd下的所有文件都删除，保留/var/lib/etcd为空目录，否则etcd将不会启动成功
	2.ETCD_INITIAL_ADVERTISE_PEER_URLS和ETCD_ADVERTISE_CLIENT_URLS需要写本etcd节点的ip或者主机名,保证其他节点能够解析，举例来说,如果我这个节点是vmnode1,那么我这就写https://vmnode1:2380 ，如果我这个节点是vmnode2，那么我就写https://vmnode2:2380
	3.ETCD_NAME需要给每个节点取一个不同的名字，例如vmnode1这台机器，我取名为etcd-node1，vmnode2我取名为etcd-node2
	4.ETCD_INITIAL_CLUSTER需要写入所有的节点
	5.ETCD_INITIAL_CLUSTER_STATE首次运行写"new"，运行成功后修改为existing,再重启etcd服务

2.6 启动etcd服务（本步骤在所有节点上执行，并且需要同时启动，不能是第一个节点启动成功后再启动第二个etcd节点，建议开多个终端，分别执行，否则etcd服务可能执行不成功）

	[root@vmnode1 ~]# systemctl enable etcd
	[root@vmnode1 ~]# systemctl start etcd
	[root@kuber-node2 ~]# systemctl status  etcd

如果服务没有启动,使用journalctl -xe查看，如果出现以下错误信息，需要修改/etc/etcd/ssl下所有文件的属主为etcd：

	Jan 08 12:13:47 vmnode1 etcd[17420]: open /etc/etcd/ssl/etcd-key.pem: permission denied
	Jan 08 12:13:47 vmnode1 systemd[1]: etcd.service: main process exited, code=exited, status=1/FAILURE
	Jan 08 12:13:47 vmnode1 systemd[1]: Failed to start Etcd Server.

2.7 检查etcd集群健康状态，在任意一个etcd节点上执行如下步骤（我这是在vmnode1上执行，但是访问的是kuber-node2节点）：

	[root@vmnode1 ~]# etcdctl --cert-file=/etc/etcd/ssl/etcd.pem  \
	--key-file=/etc/etcd/ssl/etcd-key.pem \
	--ca-file=/etc/etcd/ssl/etcd-ca.pem \
	--endpoint https://vmnode2:2379 cluster-health

显示信息如下：

	member 14b69ef4fd7851b4 is healthy: got healthy result from https://vmnode2:2379
	member 773eb256b3c437b8 is healthy: got healthy result from https://vmnode1:2379
	member ed5a8029cadfb1e3 is healthy: got healthy result from https://vmnode3:2379

### 准备CNI组建

下载CNI二进制文件：

	[root@vmnode1 ~]# export CNI_URL=https://github.com/containernetworking/plugins/releases/download
	[root@vmnode1 ~]# mkdir -p /opt/cni/bin && cd /opt/cni/bin
	[root@vmnode1 ~]# wget -qO- "${CNI_URL}/v0.7.1/cni-plugins-amd64-v0.7.1.tgz" | tar -zx

复制二进制文件到其他节点：

	[root@vmnode1 ~]# cat /tmp/copy-cni.sh
	#!/bin/bash
	for i in {vmnode2,vmnode3,vmnode4,vmnode5};do
		scp -r /opt/cni ${i}:/opt
	done

###下载kubernetes二进制文件：

1 在vmnode1上下载二进制文件：

	[root@vmnode1 ~]# export KUBE_URL=https://storage.googleapis.com/kubernetes-release/release/v1.11.3/bin/linux/amd64
	[root@vmnode1 ~]# wget ${KUBE_URL}/kubelet -O /usr/local/bin/kubelet
	[root@vmnode1 ~]# chmod +x /usr/local/bin/kubelet
	[root@vmnode1 ~]# wget ${KUBE_URL}/kubectl -O /usr/local/bin/kubectl

ps: 在centos7上安装kubernetes使用ipvs模式至少需要v1.11.1版本。

2 复制二进制文件到除vmnode1的所有节点：

	[root@vmnode1 ~]# cat /tmp/copy.sh
	#!/bin/bash
	for i in {vmnode2,vmnode3,vmnode4,vmnode5};do
		scp /usr/local/bin/kubelet ${i}:/usr/local/bin
	done

### Kubernetes Master

Master 是 Kubernetes 的大总管，主要创建apiserver、Controller manager与Scheduler来组件管理所有 Node。本步骤将下载 Kubernetes 并安装至 vmnode1上，然后产生相关 TLS Cert 与 CA 密钥，提供给集群组件认证使用。

1 创建集群 CA 与 Certificates

在这部分，将会需要生成 client 与 server 的各组件 certificates，并且替 Kubernetes admin user 生成 client 证书。

1.1 创建pki文件夹，然后进入目录完成以下操作。

	[root@vmnode1 k8s]# mkdir -p /etc/kubernetes/pki && cd /etc/kubernetes/pki
	[root@vmnode1 pki]# export KUBE_APISERVER="https://10.61.0.91:6443"

复制ca-config.json与ca-csr.json文件到当前目录，并生成 CA 密钥：

	[root@vmnode1 pki]# cp $K8S_CONFIG_FILES/pki/ca-csr.json  $K8S_CONFIG_FILES/pki/ca-config.json   ./
	[root@vmnode1 pki]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca

1.2 API server certificate

复制apiserver-csr.json文件，并生成 kube-apiserver certificate 证书：

	[root@vmnode1 pki]# cp $K8S_CONFIG_FILES/pki/apiserver-csr.json   ./
	[root@vmnode1 pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-hostname=10.96.0.1,10.61.0.91,127.0.0.1,kubernetes.default,vmnode1 \
	-profile=kubernetes \
	apiserver-csr.json | cfssljson -bare apiserver

1.3 Front proxy certificate

复制front-proxy-ca-csr.json文件，并生成 Front proxy CA 密钥，Front proxy 主要是用在 API aggregator 上:

	[root@vmnode1 pki]# cp $K8S_CONFIG_FILES/pki/front-proxy-ca-csr.json   ./
	[root@vmnode1 pki]#  cfssl gencert \
	-initca front-proxy-ca-csr.json | cfssljson -bare front-proxy-ca

复制front-proxy-client-csr.json文件，并生成 front-proxy-client 证书：

	[root@vmnode1 pki]# cp $K8S_CONFIG_FILES/pki/front-proxy-client-csr.json ./
	[root@vmnode1 pki]# cfssl gencert \
	-ca=front-proxy-ca.pem \
	-ca-key=front-proxy-ca-key.pem \
	-config=ca-config.json \
	-profile=kubernetes \
	front-proxy-client-csr.json | cfssljson -bare front-proxy-client

1.4 Bootstrap Token

由于通过手动创建 CA 方式太过繁杂，只适合少量机器，因为每次签证时都需要绑定 Node IP，随机器增加会带来很多困扰，因此这边使用 TLS Bootstrapping 方式进行授权，由 apiserver 自动给符合条件的 Node 发送证书来授权加入集群。

主要做法是 kubelet 启动时，向 kube-apiserver 传送 TLS Bootstrapping 请求，而 kube-apiserver 验证 kubelet 请求的 token 是否与设定的一样，若一样就自动产生 kubelet 证书与密钥。具体作法可以参考 TLS bootstrapping。

首先建立一个变量来产生BOOTSTRAP_TOKEN，并建立 bootstrap.conf 的 kubeconfig 文件：

	[root@vmnode1 pki]# export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
	[root@vmnode1 pki]# cat <<EOF > /etc/kubernetes/token.csv
	${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
	EOF

	# bootstrap set-cluster
	[root@vmnode1 pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../bootstrap.conf

	# bootstrap set-credentials
	[root@vmnode1 pki]# kubectl config set-credentials kubelet-bootstrap \
	--token=${BOOTSTRAP_TOKEN} \
	--kubeconfig=../bootstrap.conf

	# bootstrap set-context
	[root@vmnode1 pki]# kubectl config set-context default \
	--cluster=kubernetes \
	--user=kubelet-bootstrap \
	--kubeconfig=../bootstrap.conf

	# bootstrap set default context
	[root@vmnode1 pki]# kubectl config use-context default --kubeconfig=../bootstrap.conf

1.5 Admin certificate

复制admin-csr.json文件，并生成 admin certificate 证书：

	[root@vmnode1 pki]# cp $K8S_CONFIG_FILES/pki/admin-csr.json ./
	[root@vmnode1 pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-profile=kubernetes \
	admin-csr.json | cfssljson -bare admin

接着通过以下指令生成名称为 admin.conf 的 kubeconfig 文件：

	# admin set-cluster
	[root@vmnode1 pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../admin.conf

	# admin set-credentials
	[root@vmnode1 pki]# kubectl config set-credentials kubernetes-admin \
	--client-certificate=admin.pem \
	--client-key=admin-key.pem \
	--embed-certs=true \
	--kubeconfig=../admin.conf

	# admin set-context
	[root@vmnode1 pki]# kubectl config set-context kubernetes-admin@kubernetes \
	--cluster=kubernetes \
	--user=kubernetes-admin \
	--kubeconfig=../admin.conf

	# admin set default context
	[root@vmnode1 pki]# kubectl config use-context kubernetes-admin@kubernetes \
	--kubeconfig=../admin.conf

1.6 Controller manager certificate

复制manager-csr.json文件，并生成 kube-controller-manager certificate 证书：

	[root@vmnode1 pki]# cp $K8S_CONFIG_FILES/pki/manager-csr.json ./
	[root@vmnode1 pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-profile=kubernetes \
	manager-csr.json | cfssljson -bare controller-manager
接着通过以下指令生成名称为controller-manager.conf的 kubeconfig 文件：

	# controller-manager set-cluster
	[root@vmnode1 pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../controller-manager.conf

	# controller-manager set-credentials
	[root@vmnode1 pki]# kubectl config set-credentials system:kube-controller-manager \
	--client-certificate=controller-manager.pem \
	--client-key=controller-manager-key.pem \
	--embed-certs=true \
	--kubeconfig=../controller-manager.conf

	# controller-manager set-context
	[root@vmnode1 pki]# kubectl config set-context system:kube-controller-manager@kubernetes \
	--cluster=kubernetes \
	--user=system:kube-controller-manager \
	--kubeconfig=../controller-manager.conf

	# controller-manager set default context
	[root@vmnode1 pki]# kubectl config use-context system:kube-controller-manager@kubernetes \
	--kubeconfig=../controller-manager.conf

1.7 Scheduler certificate

复制scheduler-csr.json文件，并生成 kube-scheduler certificate 证书：

	[root@vmnode1 pki]# cp $K8S_CONFIG_FILES/pki/scheduler-csr.json  ./
	[root@vmnode1 pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-profile=kubernetes \
	scheduler-csr.json | cfssljson -bare scheduler
接着通过以下指令生成名称为 scheduler.conf 的 kubeconfig 文件：

	# scheduler set-cluster
	[root@vmnode1 pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../scheduler.conf

	# scheduler set-credentials
	[root@vmnode1 pki]# kubectl config set-credentials system:kube-scheduler \
	--client-certificate=scheduler.pem \
	--client-key=scheduler-key.pem \
	--embed-certs=true \
	--kubeconfig=../scheduler.conf

	# scheduler set-context
	[root@vmnode1 pki]# kubectl config set-context system:kube-scheduler@kubernetes \
	--cluster=kubernetes \
	--user=system:kube-scheduler \
	--kubeconfig=../scheduler.conf

	# scheduler set default context
	[root@vmnode1 pki]# kubectl config use-context system:kube-scheduler@kubernetes \
	--kubeconfig=../scheduler.conf

1.8 Kubelet master certificate

复制kubelet-csr.json文件，并生成 master node certificate 证书：

	[root@vmnode1 pki]# cp $K8S_CONFIG_FILES/pki/kubelet-csr.json ./

把NODE字符串替换为master主机名(我这是vmnode1):

	[root@vmnode1 pki]# sed -i 's@\${NODE_NAME}@vmnode1@g' pki/kubelet-csr.json
	[root@vmnode1 pki]# cfssl gencert \
	-ca=ca.pem \
	-ca-key=ca-key.pem \
	-config=ca-config.json \
	-hostname=${HOSTNAME},10.61.0.91 \
	-profile=kubernetes \
	kubelet-csr.json | cfssljson -bare kubelet

此处的-hostname需要根据实际情况做调整。

接着通过以下指令生成名称为 kubelet.conf 的 kubeconfig 文件：

	# kubelet set-cluster
	[root@vmnode1 pki]# kubectl config set-cluster kubernetes \
	--certificate-authority=ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=../kubelet.conf

	# kubelet set-credentials
	[root@vmnode1 pki]# kubectl config set-credentials system:node:${HOSTNAME} \
	--client-certificate=kubelet.pem \
	--client-key=kubelet-key.pem \
	--embed-certs=true \
	--kubeconfig=../kubelet.conf

	# kubelet set-context
	[root@vmnode1 pki]# kubectl config set-context system:node:${HOSTNAME}@kubernetes \
	--cluster=kubernetes \
	--user=system:node:${HOSTNAME} \
	--kubeconfig=../kubelet.conf


	# kubelet set default context
	[root@vmnode1 pki]# kubectl config use-context system:node:${HOSTNAME}@kubernetes \
	--kubeconfig=../kubelet.conf

1.9 Service account key

Service account 不是通过 CA 进行认证，因此不要通过 CA 来做 Service account key 的检查，这边建立一组 Private 与 Public 密钥提供给 Service account key 使用：

	[root@vmnode1 pki]# openssl genrsa -out sa.key 2048
	[root@vmnode1 pki]# openssl rsa -in sa.key -pubout -out sa.pub

确认/etc/kubernetes与/etc/kubernetes/pki有以下文件：

	[root@vmnode1 pki]# ls -l 
	total 140
	-rw-r--r-- 1 root root 1025 Jan  8 15:45 admin.csr
	-rw-r--r-- 1 root root  176 Jan  8 15:44 admin-csr.json
	-rw------- 1 root root 1679 Jan  8 15:45 admin-key.pem
	-rw-r--r-- 1 root root 1440 Jan  8 15:45 admin.pem
	-rw-r--r-- 1 root root 1029 Jan  8 15:19 apiserver.csr
	-rw-r--r-- 1 root root  181 Jan  8 15:17 apiserver-csr.json
	-rw------- 1 root root 1675 Jan  8 15:19 apiserver-key.pem
	-rw-r--r-- 1 root root 1533 Jan  8 15:19 apiserver.pem
	-rw-r--r-- 1 root root  278 Jan  8 14:59 ca-config.json
	-rw-r--r-- 1 root root 1025 Jan  8 15:00 ca.csr
	-rw-r--r-- 1 root root  177 Jan  8 14:59 ca-csr.json
	-rw------- 1 root root 1675 Jan  8 15:00 ca-key.pem
	-rw-r--r-- 1 root root 1407 Jan  8 15:00 ca.pem
	-rw-r--r-- 1 root root 1082 Jan  8 15:57 controller-manager.csr
	-rw------- 1 root root 1679 Jan  8 15:57 controller-manager-key.pem
	-rw-r--r-- 1 root root 1497 Jan  8 15:57 controller-manager.pem
	-rw-r--r-- 1 root root  891 Jan  8 15:25 front-proxy-ca.csr
	-rw-r--r-- 1 root root   66 Jan  8 15:24 front-proxy-ca-csr.json
	-rw------- 1 root root 1679 Jan  8 15:25 front-proxy-ca-key.pem
	-rw-r--r-- 1 root root 1143 Jan  8 15:25 front-proxy-ca.pem
	-rw-r--r-- 1 root root  903 Jan  8 15:29 front-proxy-client.csr
	-rw-r--r-- 1 root root   74 Jan  8 15:27 front-proxy-client-csr.json
	-rw------- 1 root root 1671 Jan  8 15:29 front-proxy-client-key.pem
	-rw-r--r-- 1 root root 1188 Jan  8 15:29 front-proxy-client.pem
	-rw-r--r-- 1 root root 1050 Jan  8 16:30 kubelet.csr
	-rw-r--r-- 1 root root  193 Jan  8 16:26 kubelet-csr.json
	-rw------- 1 root root 1679 Jan  8 16:30 kubelet-key.pem
	-rw-r--r-- 1 root root 1509 Jan  8 16:30 kubelet.pem
	-rw-r--r-- 1 root root  217 Jan  8 15:56 manager-csr.json
	-rw-r--r-- 1 root root 1679 Jan  8 16:40 sa.key
	-rw-r--r-- 1 root root  451 Jan  8 16:40 sa.pub
	-rw-r--r-- 1 root root 1058 Jan  8 16:14 scheduler.csr
	-rw-r--r-- 1 root root  199 Jan  8 16:12 scheduler-csr.json
	-rw------- 1 root root 1679 Jan  8 16:14 scheduler-key.pem
	-rw-r--r-- 1 root root 1472 Jan  8 16:14 scheduler.pem


### 安装 Kubernetes 核心组件

1 首先复制 Kubernetes 核心组件 YAML 文件，这边我们不透过 Binary 方案来创建 Master 核心组件，而是利用 Kubernetes Static Pod 来创建，因此需下载所有核心组件的Static Pod文件到/etc/kubernetes/manifests目录：

	[root@vmnode1 pki]# mkdir -p /etc/kubernetes/manifests && cd /etc/kubernetes/manifests
	[root@vmnode1 manifests]# cp -ar $K8S_CONFIG_FILES/master/manifests  /etc/kubernetes

2 修改/etc/kubernetes/manifests/apiserver.yml配置文件，修改信息如下：

	1.--advertise-address，ip需要根据实际情况做改动
	2. --etcd-servers，etcd每个节点的访问url都应写入

我这里做如下修改：

	[root@vmnode1 manifests]# sed -i 's@\${API_SERVER_IP}@10.61.0.91@g' apiserver.yml
	[root@vmnode1 manifests]# export ETCD_SERVER_ENDPOINTS="https://vmnode1:2379,https://vmnode2:2379,https://vmnode3:2379"
	[root@vmnode1 manifests]# sed -i 's@\${ETCD_SERVER_ENDPOINTS}@ETCD_SERVER_ENDPOINTS@g' apiserver.yml

3 生成一个用来加密 Etcd 的 Key：

	[root@vmnode1 manifests]# head -c 32 /dev/urandom | base64
	mwYib16D3v9fpBqOstsfHlgpniZzPjWvPcRGz2iGDtk=

4 在/etc/kubernetes/目录下，创建encryption.yml的加密 YAML 文件：

	[root@vmnode1 manifests]# cp  $K8S_CONFIG_FILES/master/encryption.yml  /etc/kubernetes
	[root@vmnode1 manifests]# sed -i 's@secret:\(.*\)@secret: mwYib16D3v9fpBqOstsfHlgpniZzPjWvPcRGz2iGDtk=@g' /etc/kubernetes/encryption.yml

5 在/etc/kubernetes/目录下，创建audit-policy.yml的进阶审核策略 YAML 文件：

	[root@vmnode1 manifests]# cp  $K8S_CONFIG_FILES/master/audit-policy.yml  /etc/kubernetes

6 下载kubelet.service相关文件来管理 kubelet：

	[root@vmnode1 manifests]# mkdir -p /etc/systemd/system/kubelet.service.d
	[root@vmnode1 manifests]# mkdir -p /var/log/kubernetes
	[root@vmnode1 manifests]# cp $K8S_CONFIG_FILES/master/10-kubelet.conf  /etc/systemd/system/kubelet.service.d/
	[root@vmnode1 manifests]# cp $K8S_CONFIG_FILES/master/kubelet.service /lib/systemd/system/
	[root@vmnode1 manifests]# cp -ar $K8S_CONFIG_FILES/master/kubelet /var/lib

7 修改配置文件/etc/systemd/system/kubelet.service.d/10-kubelet.conf，修改信息如下：

	1.KUBELET_POD_CONTAINER,如果拉取不到该容器，可以使用如下的根容器：
	registry.access.redhat.com/rhel7/pod-infrastructure:latest


8 启动 kubelet 服务:

	[root@vmnode1 manifests]#  systemctl enable kubelet.service && systemctl start kubelet.service

ps: 如果启动失败，使用journalctl -xe查看产生的错误信息。

9 其他说明：

	1.如果使用systemctl status kubelet,看见如下信息，不要惊慌，因为apiserver还没启动：
	Failed to list *v1.Pod: Get https://10.61.0.91:6443/api/v1/pods?..
	2.查看监听的端口是否在监听：
	[root@vmnode1 manifests]# ss -tunlp | grep kube
	tcp    LISTEN     0      128    127.0.0.1:10248                 *:*                   users:(("kubelet",pid=6216,fd=20))
	tcp    LISTEN     0      128    127.0.0.1:10251                 *:*                   users:(("kube-scheduler",pid=6703,fd=3))
	tcp    LISTEN     0      128    127.0.0.1:10252                 *:*                   users:(("kube-controller",pid=6815,fd=3))
	tcp    LISTEN     0      128      :::10250                :::*                   users:(("kubelet",pid=6216,fd=23))
	tcp    LISTEN     0      128      :::6443                 :::*                   users:(("kube-apiserver",pid=6893,fd=164))
	tcp    LISTEN     0      128      :::10255                :::*                   users:(("kubelet",pid=6216,fd=21))
	tcp    LISTEN     0      128      :::4194                 :::*                   users:(("kubelet",pid=6216,fd=8))


10 完成后，复制 admin kubeconfig 文件，并通过简单指令验证：

	[root@vmnode1 manifests]# cp /etc/kubernetes/admin.conf ~/.kube/config
	[root@vmnode1 manifests]# kubectl get cs
	NAME                 STATUS    MESSAGE              ERROR
	controller-manager   Healthy   ok
	scheduler            Healthy   ok                   
	etcd-0               Healthy   {"health": "true"}   
	etcd-2               Healthy   {"health": "true"}   
	etcd-1               Healthy   {"health": "true"}   

	[root@vmnode1 manifests]# kubectl get nodes
	NAME           STATUS    ROLES     AGE       VERSION
	vmnode1   Ready     master    6m        v1.8.6

	[root@vmnode1 manifests]# kubectl -n kube-system get po
	NAME                                   READY     STATUS    RESTARTS   AGE
	kube-apiserver-vmnode1            1/1       Running   0          5m
	kube-controller-manager-vmnode1   1/1       Running   0          5m
	kube-scheduler-vmnode1            1/1       Running   0          5m

ps: 如果执行kubectl get cs出现如下错误信息：

	The connection to the server localhost:8080 was refused – did you specify the right host or port?

可能的原因是忘了执行如下命令，可以再执行一次：

	[root@vmnode1 manifests]# cp /etc/kubernetes/admin.conf ~/.kube/config

确认服务能够执行 logs 等指令：

	[root@vmnode1 manifests]# kubectl -n kube-system logs -f kube-scheduler-vmnode1
	Error from server (Forbidden): Forbidden (user=kube-apiserver, verb=get, resource=nodes, subresource=proxy) ( pods/log kube-scheduler-vmnode1)

ps: 这边会发现出现 403 Forbidden 问题，这是因为 kube-apiserver user 并没有 nodes 的资源权限，属于正常。

由于上述权限问题，我们必需创建一个 apiserver-to-kubelet-rbac.yml 来定义权限，以供我们执行 logs、exec 等指令：

	[root@vmnode1 manifests]# cd /etc/kubernetes/
	[root@vmnode1 kubernetes]# cp $K8S_CONFIG_FILES/master/apiserver-to-kubelet-rbac.yml  ./
	[root@vmnode1 kubernetes]# kubectl apply -f apiserver-to-kubelet-rbac.yml 
	clusterrole "system:kube-apiserver-to-kubelet" created
	clusterrolebinding "system:kube-apiserver" created
	[root@vmnode1 kubernetes]# kubectl -n kube-system logs -f kube-scheduler-vmnode1

### Kubernetes Node

Node 是主要执行容器实例的节点，可视为工作节点。在这步骤我们会下载 Kubernetes binary 文件，并创建 node 的 certificate 来提供给节点注册认证用。Kubernetes 使用Node Authorizer来提供Authorization mode，这种授权模式会替 Kubelet 生成 API request。

在开始前，我们先在vmnode1将需要的 ca 与 cert,以及二进制文件复制到 Node 节点上：

	[root@vmnode1 kubernetes]# cat   /tmp/copy-kube.sh
	#!/bin/bash
	for NODE in {vmnode2,vmnode3,vmnode4,vmnode5}; do
		ssh ${NODE} "mkdir -p /etc/kubernetes/pki/"
		ssh ${NODE} "mkdir -p /etc/kubernetes/manifests"
		ssh ${NODE} "mkdir -p /var/lib/kubelet"
		ssh ${NODE} "mkdir -p /var/log/kubernetes"
		scp /usr/local/bin/kubelet ${NODE}:/usr/local/bin
		for FILE in pki/ca.pem pki/ca-key.pem bootstrap.conf; do
		  scp /etc/kubernetes/${FILE} ${NODE}:/etc/kubernetes/${FILE}
		done
	done

1 设置node节点的配置文件,使用脚本将配置文件传到各个node节点上：

	[root@vmnode1 kubernetes]# cat  /tmp/copy-kube-conf.sh
	#!/bin/bash
	for NODE in {vmnode2,vmnode3,vmnode4,vmnode5}; do
		ssh ${NODE} "mkdir -p  /etc/systemd/system/kubelet.service.d"
		scp $K8S_CONFIG_FILES/node/10-kubelet.conf ${NODE}:/etc/systemd/system/kubelet.service.d
		scp $K8S_CONFIG_FILES/node/kubelet.service  ${NODE}:/lib/systemd/system
		scp $K8S_CONFIG_FILES/node/kubelet ${NODE}:/var/lib
	done

2 授权 Kubernetes Node

当所有节点都完成后，在master节点，因为我们采用 TLS Bootstrapping，所需要创建一个 ClusterRoleBinding：

	[root@vmnode1 kubernetes]# kubectl create clusterrolebinding kubelet-bootstrap \
	--clusterrole=system:node-bootstrapper \
	--user=kubelet-bootstrap

3 启动各节点的kubelet服务，使用如下脚本进行：

	[root@vmnode1 kubernetes]# cat  /tmp/copy-kube-start.sh
	#!/bin/bash
	for NODE in {vmnode2,vmnode3,vmnode4,vmnode5}; do
		ssh ${NODE} "systemctl start kubelet"
		ssh ${NODE} "systemctl enable kubelet"
	done

ps: 原文中是先启动各个节点的服务再授权，我经过测试，发现如果在未授权的情况下,各节点的kubelet服务根本无法启动，错误报告如下：

	error: failed to run Kubelet: cannot create certificate signing request: certificatesigningrequests.certificates.k8s.io is forbidden:

只有先授权，才能启动各节点的kubelet服务。

4.检查节点是否成功加入

在vmnode1通过简单指令验证，会看到节点处于pending：

	[root@vmnode1 kubernetes]# kubectl get csr
	NAME                                                   AGE       REQUESTOR           CONDITION
	node-csr-9ocxrCiSF6hjpIWXjB2qibfprN6i2YYcE6ttnL2lXKM   2m        kubelet-bootstrap   Pending
	node-csr-IhB-ttp82G_z5Sgh2oyCs1ygRjgj52X2qoOBVJmFpIE   9s        kubelet-bootstrap   Pending
	node-csr-lsrD6n3t5gboFimV28ayVZdzPf_eoV702oMQTreRKUQ   1m        kubelet-bootstrap   Pending
	node-csr-nm4TaOggBqgaWg98WOkaBh44X-Hn-N0AhWUr_VUQtBQ   51s       kubelet-bootstrap   Pending

通过 kubectl 来允许节点加入集群：

	[root@vmnode1 kubernetes]# kubectl get csr | awk '/Pending/ {print $1}' | xargs kubectl certificate approve

	[root@vmnode1 kubernetes]# kubectl get csr
	NAME                                                   AGE       REQUESTOR           CONDITION
	node-csr-9ocxrCiSF6hjpIWXjB2qibfprN6i2YYcE6ttnL2lXKM   3m        kubelet-bootstrap   Approved,Issued
	node-csr-IhB-ttp82G_z5Sgh2oyCs1ygRjgj52X2qoOBVJmFpIE   1m        kubelet-bootstrap   Approved,Issued
	node-csr-lsrD6n3t5gboFimV28ayVZdzPf_eoV702oMQTreRKUQ   2m        kubelet-bootstrap   Approved,Issued
	node-csr-nm4TaOggBqgaWg98WOkaBh44X-Hn-N0AhWUr_VUQtBQ   1m        kubelet-bootstrap   Approved,Issued

	[root@vmnode1 kubernetes]# kubectl get nodes
	NAME           STATUS    ROLES     AGE       VERSION
	vmnode1    Ready        master    50m       v1.11.3
	vmnode2    NotReady     node      51s       v1.11.3
	vmnode3    NotReady     node      51s       v1.11.3
	vmnode4    NotReady     node      51s       v1.11.3
	vmnode5    NotReady     node      51s       v1.11.3

ps: 

* 如果节点加入不成功需要重新加入，得先把/var/lib/kubeletes/下的文件删除，然后重启服务即可。
* 节点出现NotReady状态是属于正常的，因为我们还没安装，kube-proxy和网络calico。

最后，禁止容器调度到vmnode1,使用如下操作：

	[root@vmnode1 kubernetes]# kubectl taint nodes vmnode1:NoSchedule

### Kubernetes Core Addons 部署

当完成上面所有步骤后，接着我们需要安装一些插件，而这些有部分是非常重要跟好用的，如Kube-dns与Kube-proxy等。

**1 Kube-proxy addon**

kube-proxy 是实现 Kubernetes Service 资源功能的关键组件，这个组件会通过 DaemonSet 在每台节点上执行，然后监听 API Server 的 Service 与 Endpoint 资源对象的事件，并依据资源预期状态通过 iptables 或 ipvs 来实现网络转发，而本次安装采用 ipvs。

使用如下步骤进行安装：

	[root@vmnode1 ~]# cp -ar $K8S_CONFIG_FILES/addons  /etc/kubernetes
	[root@vmnode1 ~]# cd /etc/kubernetes/addons
	[root@vmnode1 addons]# export KUBE_APISERVER=https://10.61.0.91:6443
	[root@vmnode1 addons]# sed -i "s/\${KUBE_APISERVER}/${KUBE_APISERVER}/g" kube-proxy/kube-proxy-cm.yml
	[root@vmnode1 addons]# kubectl apply -f kube-proxy
	[root@vmnode1 addons]# kubectl get po -n kube-system -l k8s-app=kube-proxy
	NAME               READY     STATUS    RESTARTS   AGE
	kube-proxy-jczp5   1/1       Running   0          5m
	kube-proxy-lv7wp   1/1       Running   1          5m
	kube-proxy-s5dbv   1/1       Running   0          5m
	kube-proxy-z4dbc   1/1       Running   0          5m

检查log看看是否使用了ipvs:

	[root@vmnode1 addons]# kubectl -n kube-system logs -f kube-proxy-jczp5
	I0709 08:41:48.220815       1 feature_gate.go:230] feature gates: &{map[SupportIPVSProxyMode:true]}
	I0709 08:41:48.231009       1 server_others.go:183] Using ipvs Proxier.

ps: kubernetes版本至少在v1.11.1,否则在centos上出现如下问题：

	Jun 25 20:50:00 VM_3_4_centos kube-proxy[3828]: E0625 20:50:00.312569    3828 ipset.go:156] Failed to make sure ip set: &{{KUBE-LOOP-BACK hash:ip,port,ip inet 1024 65536 0-65535 Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose} map[] 0xc42073e1d0} exist, error: error creating ipset KUBE-LOOP-BACK, error: exit status 2

若有安装 ipvsadm 的话，可以通过以下指令查看 proxy 规则：

	[root@vmnode1 addons]# ipvsadm -ln
	IP Virtual Server version 1.2.1 (size=4096)
	Prot LocalAddress:Port Scheduler Flags
	  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
	TCP  172.17.0.1:30699 rr
	  -> 10.244.0.4:53                Masq    1      0          0         
	TCP  172.17.0.1:31862 rr
	  -> 10.244.0.4:9153              Masq    1      0          0         

**2 CoreDNS**

本节将通过 CoreDNS 取代 Kube DNS 作为集群服务发现组件，由于 Kubernetes 需要让 Pod 与 Pod 之间能够互相沟通，然而要能够沟通需要知道彼此的 IP 才行，而这种做法通常是通过 Kubernetes API 来取得达到，但是 Pod IP 会因为生命周期变化而改变，因此这种做法无法弹性使用，且还会增加 API Server 负担，基于此问题 Kubernetes 提供了 DNS 服务来作为查询，让 Pod 能够以 Service 名称作为域名来查询 IP 地址，因此用户就再不需要关切实际 Pod IP，而 DNS 也会根据 Pod 变化更新资源纪录(Record resources)。

CoreDNS 是由 CNCF 维护的开源 DNS 项目，该项目前身是 SkyDNS，其采用了 Caddy 的一部分来开发服务器框架，使其能够建构一套快速灵活的 DNS，而 CoreDNS 每个功能都可以被实作成一个插件的中间件，如 Log、Cache、Kubernetes 等功能，甚至能够将源纪录储存至 Redis、Etcd 中。

在vmnode1通过 kubectl 部署CoreDNS：

	[root@vmnode1 addons]# kubectl apply -f coredns
	[root@vmnode1 addons]# kubectl -n kube-system get po -l k8s-app=kube-dns
	NAME                      READY     STATUS    RESTARTS   AGE
	coredns-8dcd46458-6gjw8   1/1       Pending   0          1m
	coredns-8dcd46458-nftd9   1/1       Pending   0          1m

这边会发现 Pod 处于Pending状态，这是由于 Kubernetes 的集群网络没有建立，因此所有节点会处于NotReady状态，而这也导致 Kubernetes Scheduler 无法替 Pod 找到适合节点而处于Pending，为了解决这个问题，下节将说明与建立 Kubernetes 集群网络。

### Kubernetes集群网路

Kubernetes 在默认情况下与 Docker 的网络有所不同。在 Kubernetes 中有四个问题是需要被解决的，分别为：

* 高耦合的容器到容器沟通：通过 Pods 与 Localhost 的沟通来解决。
* Pod 到 Pod 的沟通：通过实现网络模型来解决。
* Pod 到 Service 沟通：由 Services object 结合 kube-proxy 解决。
* 外部到 Service 沟通：一样由 Services object 结合 kube-proxy 解决。

而 Kubernetes 对于任何网络的实现都需要满足以下基本要求(除非是有意调整的网络分段策略)：

* 所有容器能够在没有 NAT 的情况下与其他容器沟通。
* 所有节点能够在没有 NAT 情况下与所有容器沟通(反之亦然)。
* 容器看到的 IP 与其他人看到的 IP 是一样的。

庆幸的是 Kubernetes 已经有非常多种的网络模型以网络插件(Network Plugins)方式被实现，因此可以选用满足自己需求的网络功能来使用。另外 Kubernetes 中的网络插件有以下两种形式：

* CNI plugins：以 appc/CNI 标准规范所实现的网络，详细可以阅读 CNI Specification。
* Kubenet plugin：使用 CNI plugins 的 bridge 与 host-local 来实现基本的 cbr0。这通常被用在公有云服务上的 Kubernetes 集群网络。

网络部署与设定：

从上述了解 Kubernetes 有多种网络能够选择，这里选择了 Calico 作为集群网络的使用。Calico 是一款纯 Layer 3 的网络，其好处是它整合了各种云原生平台(Docker、Mesos 与 OpenStack 等)，且 Calico 不采用 vSwitch，而是在每个 Kubernetes 节点使用 vRouter 功能，并通过 Linux Kernel 既有的 L3 forwarding 功能，而当数据中心复杂度增加时，Calico 也可以利用 BGP route reflector 来达成。

使用如下命令进行部署操作：

	[root@vmnode1 addons]# sed -i 's@\${CLUSTER_NETWORK}@10.244.0.0/16@g' cni/calico/v3.1/calico.yaml
	[root@vmnode1 addons]# kubectl -f cni/calico/v3.1
	[root@vmnode1 addons]# kubectl -n kube-system get po -l k8s-app=calico-node
	NAME                READY     STATUS    RESTARTS   AGE
	calico-node-2qv82   2/2       Running   2          2m
	calico-node-4jk7x   2/2       Running   0          2m
	calico-node-gc6ft   2/2       Running   0          2m
	calico-node-vq86d   2/2       Running   0          2m

确认 calico-node 都正常运作后，通过 kubectl exec 进入 calicoctl pod 来检查功能是否正常：

	[root@vmnode1 addons]# kubectl exec -ti -n kube-system calicoctl -- calicoctl get profiles -o wide
	NAME                LABELS   
	kns.default         map[]    
	kns.external-dns    map[]    
	kns.ingress-nginx   map[]    
	kns.jenkins         map[]    
	kns.kube-public     map[]    
	kns.kube-system     map[]    
	kns.monitoring      map[]

	[root@vmnode1 addons]# kubectl exec -ti -n kube-system calicoctl -- calicoctl get node -o wide
	NAME    ASN         IPV4            IPV6   
	vmnode2   (unknown)   10.61.0.92/24          
	vmnode3   (unknown)   10.61.0.93/24          
	vmnode4   (unknown)   10.61.0.94/24          
	vmnode5   (unknown)   10.61.0.95/24          

完成后，通过检查节点是否不再是NotReady，以及Pod是否不再处于Pending：

	[root@vmnode1 addons]# kubectl get no
	NAME      STATUS    ROLES     AGE       VERSION
	vmnode1     Ready     master    20m       v1.11.3
	vmnode2     Ready     <none>    20m       v1.11.3
	vmnode3     Ready     <none>    20m       v1.11.3
	vmnode4     Ready     <none>    20m       v1.11.3

	[root@vmnode1 addons]# kubectl -n kube-system get po -l k8s-app=kube-dns 
	NAME                      READY     STATUS    RESTARTS   AGE
	coredns-8dcd46458-6gjw8   1/1       Running   0          20m
	coredns-8dcd46458-nftd9   1/1       Running   0          20m

这样基本的组建已经安装完成了。

### Kubernetes Extra Addons 部署

本节说明如何部署一些官方常用的额外 Addons，如 Dashboard、Metrics Server 与 Ingress Controller 等等。

**1 Ingress Controller**

Ingress 是 Kubernetes 中的一个抽象资源，其功能是通过 Web Server 的 Virtual Host 概念以域名(Domain Name)方式转发到内部 Service，这避免了使用 Service 中的 NodePort 与 LoadBalancer 类型所带来的限制(如 Port 数量上限)，而实现 Ingress 功能则是通过 Ingress Controller 来达成，它会负责监听 Kubernetes API 中的 Ingress 与 Service 资源对象，并在发生资源变化时，依据资源预期的结果来设定 Web Server。另外 Ingress Controller 有许多实现可以选择：

* Ingress NGINX: Kubernetes 官方维护的项目，也是本次安装使用的 Controller。
* F5 BIG-IP Controller: F5 所开发的 Controller，它能够让管理员通过 CLI 或 API 从 Kubernetes 与 OpenShift 管理 F5 BIG-IP 设备。
* Ingress Kong: 著名的开源 API Gateway 项目所维护的 Kubernetes Ingress Controller。
* Traefik: 是一套开源的 HTTP 反向代理与负载平衡器，而它也支持了 Ingress。
* Voyager: 一套以 HAProxy 为底的 Ingress Controller。

在vmnode1上执行如下操作：

	[root@vmnode1 addons]# export INGRESS_VIP=10.61.0.96
	[root@vmnode1 addons]# sed -i "s/\${INGRESS_VIP}/${INGRESS_VIP}/g" ingress-controller/ingress-controller-svc.yml
	[root@vmnode1 addons]# kubectl create ns ingress-nginx
	[root@vmnode1 addons]# kubectl apply -f ingress-controller
	[root@vmnode1 addons]# kubectl -n ingress-nginx get po,svc
	NAME                                           READY     STATUS    RESTARTS   AGE
	pod/default-http-backend-846b65fb5f-g64mj      1/1       Running   0          19h
	pod/nginx-ingress-controller-5db8d65fb-5x4rt   1/1       Running   0          19h

	NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
	service/default-http-backend   ClusterIP      10.104.62.229    <none>        80/TCP         19h
	service/ingress-nginx          LoadBalancer   10.104.127.204   10.61.0.96    80:32569/TCP   19h

然后以如下的方式访问，得到的结果如下。

	[root@vmnode1 addons]# curl http://10.61.0.96
	default backend - 404

当确认上面步骤都没问题后，就可以通过kubeclt建立简单NGINX来测试功能：

	[root@vmnode1 addons]# cp -ar $K8S_CONFIG_FILES/apps /etc/kubernetes
	[root@vmnode1 addons]# kubectl apply -f /etc/kubernetes/apps/nginx
	[root@virt2 addons]# kubectl get po,svc,ing
	NAME                                READY     STATUS    RESTARTS   AGE
	pod/demo-jenkins-7886db64c4-jt869   1/1       Running   0          6h
	pod/nginx-966857787-whshs           1/1       Running   0          18h

	NAME                         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
	service/demo-jenkins         LoadBalancer   10.109.74.79     <pending>     8080:32620/TCP   6h
	service/demo-jenkins-agent   ClusterIP      10.99.97.105     <none>        50000/TCP        6h
	service/kubernetes           ClusterIP      10.96.0.1        <none>        443/TCP          1d
	service/nginx                ClusterIP      10.107.225.162   <none>        80/TCP           19h

	NAME                               HOSTS             ADDRESS      PORTS     AGE
	ingress.extensions/nginx-ingress   nginx.k8s.local   10.61.0.96   80        19h

完成后通过cURL工具来测试功能是否正常：

	root@vmnode1 addons]# curl 10.61.0.96 -H 'Host: nginx.k8s.local'
	<!DOCTYPE html>
	<html>
	<head>
	<title>Welcome to nginx!</title>
	...
	
	# 测试其他 domain name 是是否回应 404
	$ curl 10.61.0.96 -H 'Host: nginx1.k8s.local'
	default backend - 404

虽然 Ingress 能够让我们通过域名方式存取 Kubernetes 内部服务，但是若域名无法被测试机器解析的话，将会显示default backend – 404结果，而这经常发生在内部自建环境上，虽然可以通过修改主机/etc/hosts来描述，但并不弹性，因此下节将说明如何建立一个 External DNS 与 DNS 服务器来提供自动解析 Ingress 域名。

**2 External DNS**

External DNS 是 Kubernetes 小区的孵化项目，被用于定期同步 Kubernetes Service 与 Ingress 资源，并依据资源内容来自动设定公有云 DNS 服务的资源纪录(Record resources)。而由于部署不是公有云环境，因此需要通过 CoreDNS 提供一个内部 DNS 服务器，再由 ExternalDNS 与这个 CoreDNS 做串接。

首先在vmnode1执行下述指令来建立CoreDNS Server，并检查是否部署正常：

	[root@vmnode1 addons]# export DNS_VIP=10.61.0.96
	[root@vmnode1 addons]# sed -i "s/\${DNS_VIP}/${DNS_VIP}/g" external-dns/coredns/coredns-svc-tcp.yml
	[root@vmnode1 addons]# sed -i "s/\${DNS_VIP}/${DNS_VIP}/g" external-dns/coredns/coredns-svc-udp.yml
	[root@vmnode1 addons]# kubectl create -f external-dns/coredns/
	[root@vmnode1 addons]# kubectl -n external-dns get po,svc
	NAME                                READY     STATUS    RESTARTS   AGE
	pod/coredns-54bcfcbd5b-svf8t        1/1       Running   0          19h
	pod/coredns-etcd-6c9c68fd76-94227   1/1       Running   0          19h
	pod/external-dns-86f67f6df8-p84wj   1/1       Running   0          19h

	NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                       AGE
	service/coredns-etcd   ClusterIP      10.102.8.45     <none>        2379/TCP,2380/TCP             19h
	service/coredns-tcp    LoadBalancer   10.102.61.139   10.61.0.96    53:30699/TCP,9153:31862/TCP   19h
	service/coredns-udp    LoadBalancer   10.98.108.183   10.61.0.96    53:32263/UDP                  19h

ps: 默认域名为：k8s.local，修改可以文件中的coredns-cm.yml来改变。

完成后，通过dig 工具来检查是否DNS是否正常：

	[root@vmnode1 addons]# dig @10.61.0.96 SOA nginx.k8s.local +noall +answer +time=2 +tries=1

	; <<>> DiG 9.9.4-RedHat-9.9.4-29.el7 <<>> @10.61.0.96 SOA nginx.k8s.local +noall +answer +time=2 +tries=1
	; (1 server found)
	;; global options: +cmd
	k8s.local.		300	IN	SOA	ns.dns.k8s.local. hostmaster.k8s.local. 1541496044 7200 1800 86400 30

接着部署ExternalDNS来与CoreDNS同步资源纪录:

	[root@vmnode1 addons]# kubectl apply -f external-dns/external-dns
	[root@vmnode1 addons]# kubectl -n external-dns get po -l k8s-app=external-dns
	NAME                            READY     STATUS    RESTARTS   AGE
	external-dns-86f67f6df8-p84wj   1/1       Running   0          19h

完成后，通过dig 与nslookup工具检查上节测试Ingress的NGINX服务：

	[root@vmnode1 addons]# dig @10.61.0.96 A nginx.k8s.local +noall +answer +time=2 +tries=1
	...
	; (1 server found)
	;; global options: +cmd
	nginx.k8s.local.    300    IN    A    10.61.0.96

	[root@vmnode1 addons]# nslookup nginx.k8s.local
	Server:        10.61.0.96
	Address:    10.61.0.96#53

	** server can't find nginx.k8s.local: NXDOMAIN
	
这时会无法通过 nslookup 解析域名，这是因为测试机器并没有使用这个dns服务器(10.61.0.96)，需要将这个dns加入到/etc/resolve.conf，就能成功了。

再次通过nslookup检查，会发现可以解析了，这时也就能通过cURL来测试结果：

	[root@virt2 addons]# dig @10.61.0.96 A nginx.k8s.local +noall +answer +time=2 +tries=1

	; <<>> DiG 9.9.4-RedHat-9.9.4-29.el7 <<>> @10.61.0.96 A nginx.k8s.local +noall +answer +time=2 +tries=1
	; (1 server found)
	;; global options: +cmd
	nginx.k8s.local.	300	IN	A	10.61.0.96
	[root@virt2 addons]# nslookup nginx.k8s.local
	Server:		10.61.0.202
	Address:	10.61.0.202#53

	Name:	nginx.k8s.local
	Address: 10.61.0.96

**3 Dashboard addon**

Dashboard 是 Kubernetes 官方开发的 Web-based 仪表板，目的是提升管理 Kubernetes 集群资源便利性，并以资源可视化方式，来让人更直觉的看到整个集群资源状态。

在vmnode1通过 kubeclt 执行下面指令来建立 Dashboard 至 Kubernetes，并检查是否正确部署：

	[root@vmnode1 addons]# kubectl apply -f dashboard
	[root@virt2 addons]# kubectl -n kube-system get po,svc -l k8s-app=kubernetes-dashboard
	NAME                                       READY     STATUS    RESTARTS   AGE
	pod/kubernetes-dashboard-6948bdb78-cvtwn   1/1       Running   0          19h

	NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
	service/kubernetes-dashboard   ClusterIP   10.105.176.95   <none>        443/TCP   19h

	
在这边会额外建立名称为anonymous-dashboard-proxy的 Cluster Role(Binding) 来让system:anonymous这个匿名用户能够通过 API Server 来 proxy 到 Kubernetes Dashboard，而这个 RBAC 规则仅能够存取services/proxy资源，以及https: kubernetes-dashboard:资源名称。

因此我们能够在完成后，通过以下连结来进入 Kubernetes Dashboard：

	https://10.61.0.91:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

在 1.7 版本以后的 Dashboard 将不再提供所有权限，因此需要建立一个 service account 来绑定 cluster-admin role：

	[root@vmnode1 addons]# kubectl -n kube-system create sa dashboard
	
	[root@vmnode1 addons]# kubectl create clusterrolebinding dashboard --clusterrole cluster-admin --serviceaccount=kube-system:dashboard
	[root@vmnode1 addons]# SECRET=$(kubectl -n kube-system get sa dashboard -o yaml | awk '/dashboard-token/ {print $3}')

 	[root@vmnode1 addons]# kubectl -n kube-system describe secrets ${SECRET} | awk '/token:/{print $2}'
	
把token贴到Kubernetes dashboard。

**4 Prometheus**

由于 Heapster 将要被移弃，因此这边选用 Prometheus 作为第三方的集群监控方案。而本次安装采用 CoreOS 开发的 Prometheus Operator 用于管理在 Kubernetes 上的 Prometheus 集群与资源。

首先在vmnode1执行下述指令来部署所有Prometheus需要的组件：

	[root@vmnode1 addons]# kubectl apply -f prometheus/
	[root@vmnode1 addons]# kubectl apply -f prometheus/operator/
	[root@vmnode1 addons]# kubectl apply -f prometheus/alertmanater/
	[root@vmnode1 addons]# kubectl apply -f prometheus/node-exporter/
	[root@vmnode1 addons]# kubectl apply -f prometheus/kube-state-metrics/
	[root@vmnode1 addons]# kubectl apply -f prometheus/grafana/
	[root@vmnode1 addons]# kubectl apply -f prometheus/kube-service-discovery/
	[root@vmnode1 addons]# kubectl apply -f prometheus/prometheus/
	[root@vmnode1 addons]# kubectl apply -f prometheus/servicemonitor/

ps: 必须确保上一个部署执行成功后，才能执行下一个部署，否则可能导致下一个部署不能成功。

检查所有部署是否成功了：

	[root@vmnode1 addons]# kubectl -n monitoring get po,svc,ing
	NAME                                      READY     STATUS    RESTARTS   AGE
	pod/alertmanager-main-0                   2/2       Running   0          19h
	pod/alertmanager-main-1                   2/2       Running   0          19h
	pod/alertmanager-main-2                   2/2       Running   0          19h
	pod/grafana-6d495c46d5-k2l2h              1/1       Running   0          19h
	pod/kube-state-metrics-59d76f964d-pjhhp   4/4       Running   0          19h
	pod/node-exporter-hhjs8                   2/2       Running   2          19h
	pod/node-exporter-kbv2s                   2/2       Running   0          19h
	pod/node-exporter-twz9m                   2/2       Running   0          19h
	pod/node-exporter-v4ccv                   2/2       Running   0          19h
	pod/prometheus-k8s-0                      3/3       Running   2          19h
	pod/prometheus-k8s-1                      3/3       Running   1          19h
	pod/prometheus-operator-9ffd6bdd9-c9x9x   1/1       Running   0          19h

	NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
	service/alertmanager-main       ClusterIP   10.99.240.16    <none>        9093/TCP            19h
	service/alertmanager-operated   ClusterIP   None            <none>        9093/TCP,6783/TCP   19h
	service/grafana                 ClusterIP   10.106.8.244    <none>        3000/TCP            19h
	service/kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP   19h
	service/node-exporter           ClusterIP   None            <none>        9100/TCP            19h
	service/prometheus-k8s          ClusterIP   10.109.117.19   <none>        9090/TCP            19h
	service/prometheus-operated     ClusterIP   None            <none>        9090/TCP            19h
	service/prometheus-operator     ClusterIP   10.110.56.134   <none>        8080/TCP            19h

	NAME                                HOSTS                             ADDRESS      PORTS     AGE
	ingress.extensions/grafana-ing      grafana.monitoring.k8s.local      10.61.0.96   80        19h
	ingress.extensions/prometheus-ing   prometheus.monitoring.k8s.local   10.61.0.96   80        19h

确认没问题后，通过浏览器查看 prometheus.monitoring.k8s.local 与 grafana.monitoring.k8s.local 是否正常，若没问题就可以看到如下图所示结果。

**5 Metrics Server**

Metrics Server 是实现了资源 Metrics API 的组件，其目标是取代 Heapster 作为 Pod 与 Node 提供资源的 Usage metrics，该组件会从每个 Kubernetes 节点上的 Kubelet 所公开的 Summary API 中收集 Metrics。

首先在vmnode1测试一下 kubectl top 指令：

	[root@vmnode1 addons]# kubectl top node
	error: metrics not available yet

发现 top 指令无法取得 Metrics，这表示 Kubernetes 集群没有安装 Heapster 或是 Metrics Server 来提供 Metrics API 给 top 指令取得资源使用量。

由于上述问题，我们要在k8s-m1节点通过 kubectl 部署 Metrics Server 组件来解决：

	[root@vmnode1 addons]# kubectl create -f metric-server/
	[root@vmnode1 addons]# kubectl -n kube-system get po -l k8s-app=metrics-server
	NAME                              READY     STATUS    RESTARTS   AGE
	metrics-server-86bd9d7667-l74mh   1/1       Running   0          5h

完成后，等待一点时间（约30s – 2m）收集指标，再次执行kubectl top指令查看：

	[root@vmnode1 addons]# kubectl top node
	NAME      CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%   
	vmnode2     452m         2%        4617Mi          14%       
	vmnode3     233m         1%        2349Mi          3%        
	vmnode4     11860m       74%       4813Mi          7%        
	vmnode5     12204m       76%       4245Mi          6%  

而这时若有使用HPA的话，就能够正确抓到Pod的CPU与Memory使用量了。

**6 Helm Tiller Server**

Helm 是 Kubernetes Chart 的管理工具，Kubernetes Chart 是一套预先组态的 Kubernetes 资源。其中Tiller Server主要负责接收来至 Client 的指令，并通过 kube-apiserver 与 Kubernetes 集群做沟通，根据 Chart 定义的内容，来产生与管理各种对应 API 对象的 Kubernetes 部署文件(又称为 Release)。

首先在vmnode1安装Helm工具：

	[root@vmnode1 addons]# wget -qO- https://kubernetes-helm.storage.googleapis.com/helm-v2.9.1-linux-amd64.tar.gz | tar -zx
	[root@vmnode1 addons]# mv linux-amd64/helm /usr/local/bin/

另外在所有node节点安装socat：

	[root@vmnode1 addons]# yum install socat -y

接着初始化Helm（这边会安装Tiller Server）：

	[root@vmnode1 addons]# kubectl -n kube-system create sa tiller
	[root@vmnode1 addons]# kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
	[root@vmnode1 addons]# helm init --service-account tiller
	[root@vmnode1 addons]# kubectl -n kube-system get po -l app=helm
	NAME                            READY     STATUS    RESTARTS   AGE
	tiller-deploy-759cb9df9-cg7l5   1/1       Running   0          19h
	[root@vmnode1 addons]# helm version
	Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
	Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}

测试Helm功能:

这边部署简单Jenkins来进行功能测试:

	[root@vmnode1 addons]# cp -ar $K8S_CONFIG_FILES/helm  /etc/kubernetes
	[root@vmnode1 addons]# helm install --name demo --set Persistence.Enabled=false -f ../helm/charts/jenkins/values.yaml
	[root@vmnode1 addons]#  kubectl get po,svc  -l app=demo-jenkins
	NAME                                READY     STATUS    RESTARTS   AGE
	pod/demo-jenkins-7886db64c4-jt869   1/1       Running   0          6h

	NAME                         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
	service/demo-jenkins         LoadBalancer   10.109.74.79   <pending>     8080:32620/TCP   6h
	service/demo-jenkins-agent   ClusterIP      10.99.97.105   <none>        50000/TCP        6h

当服务都正常运作时，就可以通过浏览器查看http://10.61.0.91:32620页面。

测试完成后，就可以通过以下指令来删除 Release：

	[root@vmnode1 addons]#  helm ls
	NAME	REVISION	UPDATED                 	STATUS  	CHART         	NAMESPACE
	demo	1       	Tue Nov  6 10:58:16 2018	DEPLOYED	jenkins-0.21.0	default  
	You have mail in /var/spool/mail/root
	[root@vmnode1 addons]# helm delete demo --purge

### 简单部署 Nginx 服务

Kubernetes 可以选择使用指令直接创建应用程序与服务，或者撰写 YAML 与 JSON 档案来描述部署应用程序的配置，以下将创建一个简单的 Nginx 服务：

	[root@vmnode1 addons]# kubectl run nginx --image=nginx --port=80
	[root@vmnode1 addons]# kubectl expose deploy nginx --port=80 --type=LoadBalancer --external-ip=10.61.0.160
	[root@vmnode1 addons]# kubectl get svc,po
	NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
	svc/kubernetes   ClusterIP      10.96.0.1        <none>        443/TCP        4h
	svc/nginx        LoadBalancer   10.103.165.116   10.61.0.160   80:30367/TCP   1m

	NAME                        READY     STATUS    RESTARTS   AGE
	po/nginx-7cbc4b4d9c-h8gqn   1/1       Running   0          1m