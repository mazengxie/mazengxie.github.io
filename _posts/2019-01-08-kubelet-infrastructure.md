---
title: kubelet架构分析
key: 20190108
tags: kubelet
---

kubelet是kubernetes集群中真正维护容器状态，具体“干活”的组件。每个节点上都运行一个 kubelet 服务进程，默认监听 10250 端口，接收并执行 master 发来的指令，管理 Pod 及 Pod 中的容器。
<!--more-->

## 1.主要功能
### 节点管理
节点管理主要是节点自注册和节点状态更新：
- Kubelet 可以通过设置启动参数 --register-node 来确定是否向 API Server 注册自己；
- 如果 Kubelet 没有选择自注册模式，则需要用户自己配置 Node 资源信息，同时需要告知 Kubelet 集群上的 API Server 的位置；
- Kubelet 在启动时通过 API Server 注册节点信息，并定时向 API Server 发送节点新消息，API Server 在接收到新消息后，将信息写入 etcd

###  pod管理
###  容器监控
###  健康检查



## 2.监听端口
kubelet 默认监听四个端口，分别为 10250 、10255、10248、4194。


```
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      3272/kubelet        
tcp6       0      0 :::10255                :::*                    LISTEN      3272/kubelet        
tcp6       0      0 :::4194                 :::*                    LISTEN      3272/kubelet        
tcp6       0      0 :::10250                :::*                    LISTEN      3272/kubelet
```


1. 10250（kubelet API）：kubelet server 与 apiserver 通信的端口，定期请求 apiserver 获取自己所应当处理的任务，通过该端口可以访问获取 node 资源以及状态。kubectl查看pod的日志和cmd命令，都是通过kubelet端口10250访问，如果本地没有开启10250端口的话：

查看日志
```
[devops@master cloudk8s]$ kubectl logs nginx-deployment-ddbc89dc5-7tkt5 
Error from server: Get https://192.168.0.116:10250/containerLogs/default/nginx-deployment-ddbc89dc5-7tkt5/nginx: dial tcp 192.168.0.116:10250: getsockopt: connection refused
```

执行cmd命令
  
```
[devops@master cloudk8s]$ kubectl exec -it nginx-deployment-ddbc89dc5-7tkt5 /bin/sh
Error from server: error dialing backend: dial tcp 192.168.0.116:10250: getsockopt: connection refused
```


2. 10248（健康检查端口)： kubelet 是否正常工作, 通过 kubelet 的启动参数 --healthz-port 和 --healthz-bind-address 来指定监听的地址和端口。


```
[root@node1 ~]# curl http://127.0.0.1:10248/healthz
ok
```

3. 4194（cAdvisor 监听）：kublet 通过该端口可以获取到该节点的环境信息以及 node 上运行的容器状态等内容，访问 http://localhost:4194 可以看到 cAdvisor 的管理界面,
通过 kubelet 的启动参数 --cadvisor-port 可以指定 启动的端口。

```
[root@node1 ~]# curl  http://127.0.0.1:4194/metrics
```


4. 10255 （readonly API）：提供了 pod 和 node 的信息，接口以只读形式暴露出去，访问该端口不需要认证和鉴权。
获取 pod 的接口，与 apiserver 的 
http://127.0.0.1:8080/api/v1/pods?fieldSelector=spec.nodeName=  接口类似


```
[root@node1 ~]# curl  http://127.0.0.1:10255/pods
```


节点信息接口,提供磁盘、网络、CPU、内存等信息


```
[root@node1 ~]# curl http://127.0.0.1:10255/spec/
```


