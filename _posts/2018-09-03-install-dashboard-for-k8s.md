---
title: 安装和使用dashboard
key: 20180903
tags: k8s
---

## 1. 安装dashboard

为了提供更丰富的用户体验，Kubernetes 还开发了一个基于 Web 的 Dashboard，用户可以用 Kubernetes Dashboard 部署容器化的应用、监控应用的状态、执行故障排查任务以及管理 Kubernetes 各种资源。
<!--more-->
在 Kubernetes Dashboard 中可以查看集群中应用的运行状态，也能够创建和修改各种 Kubernetes 资源，比如 Deployment、Job、DaemonSet 等。用户可以 Scale Up/Down Deployment、执行 Rolling Update、重启某个 Pod 或者通过向导部署新的应用。Dashboard 能显示集群中各种资源的状态以及日志信息。

可以说，Kubernetes Dashboard 提供了 kubectl 的绝大部分功能，大家可以根据情况进行选择。Kubernetes 默认没有部署 Dashboard，可通过如下命令安装：

	kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

Dashboard 会在 kube-system namespace 中创建自己的 Deployment 和 Service。

官方文档建议dashboard的访问方式是通过 kubectl proxy：

	kubectl proxy --address='192.168.0.111' --port=8086 --accept-hosts='^*$'

> 按官方文档建议的方式安装完dashboard后，使用kubectl proxy代理的方式来访问webUI。使用这个代理的方式访问就会导致登录无响应的问题。

发现登录无响应，所以我又采用kubernetes-dashboard 服务暴露了 NodePort，使用 https://NodeIP:nodePort 地址访问 dashboard。修改yaml,添加NodePort

	kind: Service
	apiVersion: v1
	metadata:
	  labels:
	    k8s-app: kubernetes-dashboard
	  name: kubernetes-dashboard
	  namespace: kube-system
	spec:
	  # 添加Service的type为NodePort
	  type: NodePort
	  ports:
	    - port: 443
	      targetPort: 8443
	      # 添加映射到虚拟机的端口,k8s只支持30000以上的端口
	      nodePort: 30001
	  selector:
	    k8s-app: kubernetes-dashboard

然后应用到dashboard：

	kubectl apply -f kubernetes-dashboard.yaml

## 2. 使用dashboard
dashboard登录地址为https://192.168.0.111:30008

推荐使用火狐浏览器firefox并为该网站添加例外。
![](/images/2018-09-03/https-cert.jpg)
如果选择使用谷歌浏览器话，需要关掉chrome证书检查

登录认证有两种方式：
![](/images/2018-09-03/login.jpg)

① token直接认证

利用如下命令获取token

	[root@master dashboard]# kubectl -n kube-system describe $(kubectl -n kube-system get secret -o name | grep namespace) | grep token
	Name:         namespace-controller-token-7c7w2
	Type:  kubernetes.io/service-account-token
	token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJuYW1lc3BhY2UtY29udHJvbGxlci10b2tlbi03Yzd3MiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJuYW1lc3BhY2UtY29udHJvbGxlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImJkMTIwNDU3LTZlYjMtMTFlOC1hNWI4LTAwMGMyOWI1Nzk2NiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTpuYW1lc3BhY2UtY29udHJvbGxlciJ9.YIHNOFqyoJXczz5myTDGpDp6nNTeUI1OBOCiMxmsu66iitWICIihiXDeVb3rpPsTAaRt5YQl5uYcn0gv1erFOBaAepz-G6E8Ke2A-cXA_QZ4wq4phAIhnXqX8JjKVA8qf6sAb5nBY1bXcCitQorthT2Kcz3TPUeqfxzIvYv0UBsRUrxI2hBHNxB_hB3AF7WiIQi3IIH54VaQ0OokgYrMebDdaxv1OBhM0nODb4ZZ05qpwLrnIb4ims1JfIX9re2YfLnGkEk0zvqtV1DTvseRt1OCbenqU_pdXFOsEVnGatVHb-Ln7qMJn2vL5zIdh4fx_OTdLIa5Hq7uSniHQ_kN

② 通过Kubeconfig文件认证

只需要在kubeadm生成的admin.conf文件末尾加上刚刚获取的token就好了。

	- name: kubernetes-admin
	  user:
	    client-certificate-data: xxxxxxxx
	    client-key-data: xxxxxx
	    token: "在这里加上token"

## 3. 通过rbac控制用户的权限

创建iot的用户的yaml文件：

	kind: RoleBinding
	apiVersion: rbac.authorization.k8s.io/v1beta1
	metadata:
	  namespace: iot # This only grants permissions within the "development" namespace.
	subjects:
	- kind: User
	  name: system:serviceaccount:iot:iot
	  apiGroup: rbac.authorization.k8s.io
	roleRef:
	  kind: ClusterRole
	  name: admin
	  apiGroup: rbac.authorization.k8s.io
	
	
	apiVersion: v1
	kind: ServiceAccount
	metadata:
	  name: iot
	  namespace: iot
	  labels:
	    kubernetes.io/cluster-service: "true"
	    addonmanager.kubernetes.io/mode: Reconcil

应用yaml文件并检验执行结果：

	[root@master k8s-user]# kubectl apply -f iot.yaml
	[root@master k8s-user]# kubectl -n iot describe $(kubectl -n iot get secret -o name | grep iot-token) | grep token
	Name:         iot-token-xrqdr
	Type:  kubernetes.io/service-account-token
	token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJpb3QiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoiaW90LXRva2VuLXhycWRyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImlvdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjhhYWFkZTNiLTg1YTctMTFlOC05MzM5LTAwMGMyOWI1Nzk2NiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDppb3Q6aW90In0.jXg61yJUooBaknbbMZaJDexZoM6OaDNS5zM2vZhKg7Hou3Kl9GqKugsCvawbw20TbZK5AYo03O-j1_Bf1rUOAup0TKWUQrS2UodBbxpFVKTLroBjHb-RqLxvoDrrmMoBGACUguhjxqWFG1b-lAjCyLosKycMSDnzW6LZmd7zEwp0HL4co5qfN9osZO5l7L3TBOwB9ij5-hVjD0qDm85Q6_--1FhUNdcimRdK9GdqvX0PYD4aOy3AEUyadYEfq2hiVYPbTNB8TFQqI51JQT-IBAix9JiTocdrGbiSIOkRo6vL4e5znXXUtAvBuERomdO7-oI1e0oDSj7M1lbIZOpjIw

我们新创建的用户，只能访问iot命名空间的，从而设置了用户的权限。

如果想在任意一个serviceaccount用户里面都能显示所有的命名空间，方便切换，可以使用如下yaml文件：

	kind: ClusterRole
	apiVersion: rbac.authorization.k8s.io/v1beta1
	metadata:
	  name: default-reader
	rules:
	  - apiGroups: [""]
	    resources:
	      - namespaces
	    verbs: ["get", "watch", "list"]
	  - nonResourceURLs: ["*"]
	    verbs: ["get", "watch", "list"]
	
	---
	
	kind: ClusterRoleBinding
	apiVersion: rbac.authorization.k8s.io/v1beta1
	metadata:
	  name: default-reader-binding
	subjects:
	  - kind: Group
	    name: system:authenticated
	roleRef:
	  kind: ClusterRole
	  name: default-reader
	  apiGroup: rbac.authorization.k8s.io

![](/images/2018-09-03/all-ns.jpg)
## FAQ：

1. 在dashboard界面登录没有反应？

	切换成NodePort方式登录，kube-proxy出现该问题的原因，我还未定位。

2. 浏览器不能访问？
     
	https证书问题，可以使用firefox为登录地址添加例外关掉chrome证书检查，然后登录

3. token过期时间太短？

	 Dashboard的Token失效时间可以通过 token-ttl 参数来设置，修改创建Dashboard的yaml文件，并重新创建即可。

		ports:
		- containerPort: 8443
		  protocol: TCP
		args:
		  - --auto-generate-certificates
		  - --token-ttl=43200

4.  不显示内存和CPU

	k8s默认是采用heapster组件来收集集群的内存和CPU使用情况，所以请确认是否安装了heapster组件。Heapster 本身是一个 Kubernetes 应用，部署方法很简单，运行如下命令：

		git clone https://github.com/kubernetes/heapster.git
		kubectl apply -f heapster/deploy/kube-config/influxdb/
		kubectl apply -f heapster/deploy/kube-config/rbac/heapster-rbac.yaml

	![](/images/2018-09-03/cpu-mem.jpg)