---
title: 使用kubeadm安装Kubernetes v1.10
key: 20180827
tags: k8s
---

## 1. 基础环境

采用CentOS7.4 minimual，docker 1.13，kubeadm 1.10.0，etcd 3.0， k8s 1.10.0
我们这里选用三个节点搭建一个k8s集群，其中一个master，两个node。
<!--more-->

1.配置好各节点/etc/hosts文件

	192.168.0.111 master
	192.168.0.112 node1
	192.168.0.113 node2

2.关闭系统防火墙

	iptables -F
	systemctl stop firewalld
	systemctl disable firewalld

3.关闭SElinux

	vim /etc/selinux/config
	SELINUX=disabled

重启服务器，通过命令查看修改是否成功

	[devops@master ~]$ getenforce
	Disabled

4.关闭swap

	swapoff -a 
	或者在/etc/fstab删除swap相关配置

5.配置系统内核参数使流过网桥的流量也进入iptables/netfilter框架中，在/etc/sysctl.conf中添加以下配置：

	net.bridge.bridge-nf-call-iptables = 1
	net.bridge.bridge-nf-call-ip6tables = 1
	sysctl -p

## 2. 安装kubeadm和相关工具包
### 2.1 首先配置阿里K8S YUM源

	cat <<EOF > /etc/yum.repos.d/kubernetes.repo
	[kubernetes]
	name=Kubernetes
	baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
	enabled=1
	gpgcheck=0
	EOF
	yum -y install epel-release
	yum clean all
	yum makecache

### 2.2 安装kubeadm和相关工具包
	
	yum -y install docker 
	yum -y install kubectl-1.10.4-0.x86_64
	yum -y install kubelet-1.10.4-0.x86_64
	yum -y install kubeadm-1.10.4-0.x86_64
	yum -y install kubernetes-cni.x86_64

> docker在1.13版本以后分为docker-ce和docker-ee，本环境安装的是docker1.13

### 2.3 启动Docker与kubelet服务

	systemctl enable docker && systemctl start docker
	systemctl enable kubelet && systemctl start kubelet

> 提示：此时kubelet的服务运行状态是异常的，因为缺少主配置文件kubelet.conf。但可以暂不处理，因为在完成Master节点的初始化后才会生成这个配置文件。

## 3. 利用kubeadm安装k8s
### 3.1 配置镜像加速器
 
因为无法直接访问gcr.io下载镜像，所以需要配置一个国内的容器镜像加速器，配置一个阿里云的加速器：

登录[https://cr.console.aliyun.com/](https://cr.console.aliyun.com/)，在页面中找到并点击镜像加速按钮，即可看到属于自己的专属加速链接，选择Centos版本后即可看到配置方法。

	1. 安装／升级Docker客户端
	推荐安装1.10.0以上版本的Docker客户端，参考文档 docker-ce
	
	2. 配置镜像加速器
	针对Docker客户端版本大于 1.10.0 的用户
	
	您可以通过修改daemon配置文件/etc/docker/daemon.json来使用加速器
	sudo mkdir -p /etc/docker
	sudo tee /etc/docker/daemon.json <<-'EOF'
	{
	  "registry-mirrors": ["https://j20ri28s.mirror.aliyuncs.com"]
	}
	EOF
	sudo systemctl daemon-reload
	sudo systemctl restart docker

### 3.2 下载K8S相关镜像

解决完加速器的问题之后，开始下载k8s相关镜像，下载后将镜像名改为k8s.gcr.io/开头的名字，以便kubeadm识别使用。

	#!/bin/bash
	images=(kube-proxy-amd64:v1.10.0 kube-scheduler-amd64:v1.10.0 kube-controller-manager-amd64:v1.10.0 kube-apiserver-amd64:v1.10.0
	etcd-amd64:3.1.12 pause-amd64:3.1 kubernetes-dashboard-amd64:v1.8.3 k8s-dns-sidecar-amd64:1.14.8 k8s-dns-kube-dns-amd64:1.14.8
	k8s-dns-dnsmasq-nanny-amd64:1.14.8)
	for imageName in ${images[@]} ; do
	  docker pull keveon/$imageName
	  docker tag keveon/$imageName k8s.gcr.io/$imageName
	  docker rmi keveon/$imageName
	done

上面的shell脚本主要做了3件事，下载各种需要用到的容器镜像、重新打标记为符合k8s命令规范的版本名称、清除旧的容器镜像。

> 提示：镜像版本一定要和kubeadm安装的版本一致，否则会出现time out问题。 

### 3.3 初始化安装K8S Master

执行上述shell脚本，等待下载完成后，执行kubeadm init

	[root@k8smaster ~]# kubeadm init --kubernetes-version=v1.10.0 --pod-network-cidr=10.244.0.0/16
	[init] Using Kubernetes version: v1.10.0
	[init] Using Authorization modes: [Node RBAC]
	[preflight] Running pre-flight checks.
	    [WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
	    [WARNING FileExisting-crictl]: crictl not found in system path
	Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
	[preflight] Starting the kubelet service
	[certificates] Generated ca certificate and key.
	[certificates] Generated apiserver certificate and key.
	[certificates] apiserver serving cert is signed for DNS names [k8smaster kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.100.202]
	[certificates] Generated apiserver-kubelet-client certificate and key.
	[certificates] Generated etcd/ca certificate and key.
	[certificates] Generated etcd/server certificate and key.
	[certificates] etcd/server serving cert is signed for DNS names [localhost] and IPs [127.0.0.1]
	[certificates] Generated etcd/peer certificate and key.
	[certificates] etcd/peer serving cert is signed for DNS names [k8smaster] and IPs [10.0.100.202]
	[certificates] Generated etcd/healthcheck-client certificate and key.
	[certificates] Generated apiserver-etcd-client certificate and key.
	[certificates] Generated sa key and public key.
	[certificates] Generated front-proxy-ca certificate and key.
	[certificates] Generated front-proxy-client certificate and key.
	[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
	[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
	[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
	[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
	[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
	[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
	[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
	[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
	[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
	[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
	[init] This might take a minute or longer if the control plane images have to be pulled.
	[apiclient] All control plane components are healthy after 21.001790 seconds
	[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
	[markmaster] Will mark node k8smaster as master by adding a label and a taint
	[markmaster] Master k8smaster tainted and labelled with key/value: node-role.kubernetes.io/master=""
	[bootstraptoken] Using token: thczis.64adx0imeuhu23xv
	[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
	[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
	[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
	[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
	[addons] Applied essential addon: kube-dns
	[addons] Applied essential addon: kube-proxy
	Your Kubernetes master has initialized successfully!
	To start using your cluster, you need to run the following as a regular user:
	  mkdir -p $HOME/.kube
	  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	  sudo chown $(id -u):$(id -g) $HOME/.kube/config
	You should now deploy a pod network to the cluster.
	Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
	  https://kubernetes.io/docs/concepts/cluster-administration/addons/
	You can now join any number of machines by running the following on each node
	as root:
	  kubeadm join 192.168.0.111:6443 --token hcymqj.6mdpqlgu7ghi4wiv --discovery-token-ca-cert-hash sha256:d23cd67bd64076077a6a301ee6b7a67244479e5766f41f0de3b60a1df59f111a

上面的命令大约需要1分钟的过程，期间可以观察下tail -f /var/log/message日志文件的输出，掌握该配置过程和进度。
 
> 提示：选项–kubernetes-version=v1.10.0是必须的，否则会因为访问google网站被墙而无法执行命令。这里使用v1.10.0版本，刚才前面也说到了下载的容器镜像版本必须与K8S版本一致否则会出现time out。

### 3.4 配置kubectl认证信息

对于非root用户

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

对于root用户

	export KUBECONFIG=/etc/kubernetes/admin.conf
	也可以直接放到~/.bash_profile
	echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile

### 3.5 安装flannel网络

	mkdir -p /etc/cni/net.d/
	cat <<EOF> /etc/cni/net.d/10-flannel.conf
	{
	“name”: “cbr0”,
	“type”: “flannel”,
	“delegate”: {
	“isDefaultGateway”: true
	}
	}
	EOF
	mkdir /usr/share/oci-umount/oci-umount.d -p
	mkdir /run/flannel/
	cat <<EOF> /run/flannel/subnet.env
	FLANNEL_NETWORK=10.244.0.0/16
	FLANNEL_SUBNET=10.244.1.0/24
	FLANNEL_MTU=1450
	FLANNEL_IPMASQ=true
	EOF
	kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml

### 3.6 让node1、node2加入集群
查看token

	[devops@master ~]$  kubeadm token create --print-join-command
	kubeadm join 192.168.0.111:6443 --token hcymqj.6mdpqlgu7ghi4wiv --discovery-token-ca-cert-hash sha256:d23cd67bd64076077a6a301ee6b7a67244479e5766f41f0de3b60a1df59f111a

在node1和node2节点上分别执行上面输出的kubeadm join命令，加入集群：

	[root@k8snode1 ~]# kubeadm join 192.168.0.111:6443 --token hcymqj.6mdpqlgu7ghi4wiv --discovery-token-ca-cert-hash sha256:d23cd67bd64076077a6a301ee6b7a67244479e5766f41f0de3b60a1df59f111a
	[preflight] Running pre-flight checks.
	    [WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
	    [WARNING FileExisting-crictl]: crictl not found in system path
	Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
	[discovery] Trying to connect to API Server "192.168.0.111:6443"
	[discovery] Created cluster-info discovery client, requesting info from "https://192.168.0.111:6443"
	[discovery] Requesting info from "https://192.168.0.111:6443" again to validate TLS against the pinned public key
	[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.0.111:6443"
	[discovery] Successfully established connection with API Server "192.168.0.111:6443"
	This node has joined the cluster:
	* Certificate signing request was sent to master and a response
	  was received.
	* The Kubelet was informed of the new secure connection details.
	Run 'kubectl get nodes' on the master to see this node join the cluster.

默认情况下，Master节点不参与工作负载，也可以使用该命令设置

	[devops@master ~]$ kubectl cordon master

但如果希望安装出一个All-In-One的k8s环境，则可以执行以下命令，让Master节点也成为一个Node节点：

    kubectl taint nodes --all node-role.kubernetes.io/master-

### 3.7 验证K8S Master是否搭建成功
查看节点状态

	[devops@master ~]$ kubectl get nodes
	NAME      STATUS                     ROLES     AGE       VERSION
	master    Ready,SchedulingDisabled   master    75d       v1.10.4
	node1     Ready                      <none>    75d       v1.10.4
	node2     Ready                      <none>    75d       v1.10.4

查看pods状态

	[devops@master ~]$ kubectl get pods --all-namespaces
	NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE
	kube-system   etcd-master                             1/1       Running   6          33d
	kube-system   heapster-84b8d84f96-2m72v               1/1       Running   6          45d
	kube-system   kube-apiserver-master                   1/1       Running   5          11d
	kube-system   kube-controller-manager-master          1/1       Running   6          11d
	kube-system   kube-dns-86f4d74b45-84pdr               3/3       Running   21         75d
	kube-system   kube-flannel-ds-6x897                   1/1       Running   10         75d
	kube-system   kube-flannel-ds-94fdh                   1/1       Running   11         75d
	kube-system   kube-flannel-ds-p44gh                   1/1       Running   17         75d
	kube-system   kube-proxy-bq95b                        1/1       Running   3          11d
	kube-system   kube-proxy-cmv85                        1/1       Running   3          11d
	kube-system   kube-proxy-xzfgg                        1/1       Running   4          11d
	kube-system   kube-scheduler-master                   1/1       Running   6          11d
	kube-system   kubernetes-dashboard-7d5dcdb6d9-b28w4   1/1       Running   0          5d
	kube-system   monitoring-grafana-7c77b7dd76-nm66s     1/1       Running   6          45d
	kube-system   monitoring-influxdb-779c4854d6-rx252    1/1       Running   6          48d

查看K8S集群状态

	[devops@master ~]$ kubectl get cs
	NAME                 STATUS    MESSAGE              ERROR
	controller-manager   Healthy   ok                   
	scheduler            Healthy   ok                   
	etcd-0               Healthy   {"health": "true"}  

## FAQ:
1.kube init失败怎么办？ 查找原因，并使用如下命令进行回退

	sudo kubeadm reset
	sudo docker stop $(docker ps |grep k8s_ | awk '{print $1}')
	sudo docker rm   $(docker ps |grep k8s_ | awk '{print $1}')
	sudo rm -rf /var/lib/kubelet/

2.gcr.io被墙怎么办？ 为Docker 设置 http,https socks5 设置代理

	mkdir -p /etc/systemd/system/docker.service.d
	touch /etc/systemd/system/docker.service.d/http-proxy.conf

编辑http-proxy.conf文件, 添加
	[Service]
	Environment="HTTPS_PROXY=socks5://127.0.0.1:1080/" "NO_PROXY=localhost,127.0.0.1,docker.io,j20ri28s.mirror.aliyuncs.com,*.aliyuncs.com,*.mirror.aliyuncs.com,registry.docker-cn.com,hub.c.163.com,hub-auth.c.163.com,"

更新docker配置并重启

	[root@docker devops]# systemctl daemon-reload
	[root@docker devops]# systemctl show --property=Environment docker
	Environment=GOTRACEBACK=crash DOCKER_HTTP_HOST_COMPAT=1 PATH=/usr/libexec/docker:/usr/bin:/usr/sbin HTTPS_PROXY=socks5://127.0.0.1:1080/ NO_PROXY=localhost,127.0.0.1,docker
	
	[root@docker devops]# systemctl restart docker

3. 如果没有代理而且也不能访问谷歌，怎么办？
 
可以采用二进制文件安装，获取链接[https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG.md](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG.md "k8s release binary")

也可以自己编译k8s images镜像，编译方法参考[https://mazengxie.github.io/2018/08/28/k8s-make.html](https://mazengxie.github.io/2018/08/28/k8s-make.html "kubernetes的编译与打包")