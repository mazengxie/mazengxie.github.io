---
title: 构建镜像与Kubernetes 部署应用
key: 20180830
tags: docker
---

## 1. 基础环境
我们可以通过Dockerfile文件构建镜像，然后运行在kubernetes上，即利用kubernetes对容器的网络地址、生命周期等进行编排。我们的基础环境如下，
<!--more-->

	集群各节点得IP地址
	192.168.0.111 master
	192.168.0.112 node1
	192.168.0.113 node2
	192.168.0.114 docker仓库

software  | version
------------ | -------------
linux | centos7. 4 mini
kubernetes | v1.10
docker | 1.13
go | v1.9
java | 1.8

## 2. 构建镜像
### 2.1 创建Dockerfile
我们以java程序的jar包为例，创建Dockerfile文件如下

	FROM openjdk:8-alpine
	ENV TZ "Asia/Shanghai"
	RUN export LANG="zh_CH.UTF-8"
	ARG jar_path
	ADD $jar_path/*.jar /opt/
	WORKDIR /opt/
	ENV JAVA_OPTS "-Dspring.profiles.active=k8s_test -server -Xms128m -Xmx256m -XX:MaxMetaspaceSize=128m"
	ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /opt/*.jar"]

Dockerfile文件很简单，只需要8行就可以构建出生产级的java镜像。我们逐行来看一下实现的功能：

①执行 FROM，把openjdk:8-alpine 作为 base 镜像，当然我们也可以在一个纯净的centos镜像上，自己安装java软件也是可以的。

②ENV TZ，ENV指令用以定义镜像的环境变量，docker默认是utc时间，我们这儿设置为上海时区。

③RUN export LANG，RUN功能为运行指定的命令，例如RUN yum install vim，那么生成的镜像里面则会自动安装上vim编辑器。我们设置字符编码是utf-8的，以便可以显示中文。

④ARG jar_path，ARG定义构建镜像时需要的参数、用户可以在构建期间通过docker build --build-arg “varname”=“value”，将其传递给构建器、如果指定了dockerfile中没有定义的参数，则发出警告，提示构建参数未被使用。使用ARG可以增加dockerfile文件的复用性。

⑤ ADD $jar_path/*.jar /opt/， ADD指令的功能是将主机构建环境（上下文）目录中的文件和目录。我们的构建环境下的目录结构：

	[root@docker iot_images]# ls
	build.sh  dockerfile  eureka  yihao01-permission-management-6810

通过docker build --build-arg命令，就可以指定把eureka，还是yihao01-permission-management-6810目录下的jar包复制到容器中。

⑥WORKDIR /opt/， WORKDIR为后续的ENTRYPOINT指令配置工作目录/opt。

⑦ENV JAVA_OPTS，这里设置JAVA_OPTS的原因是java1.8配合docker有点问题。docker设置了内存限制，旧版本的Java并不能识别这个限制，目前java1.10已经解决。

⑧ENTRYPOINT，执行启动java的命令。

### 2.2 构建镜像
下面我们运行 docker build 命令构建镜像，并对照dockerfile文件查看每一步的细节。

	[root@docker iot_images]# docker build --build-arg jar_path=yihao01-permission-management-6810 -t 192.168.0.114:5000/oeasy/yihao01-permission-management-6810:1.0.0 -f dockerfile .
	Sending build context to Docker daemon 111.6 MB
	Step 1/8 : FROM 192.168.0.114:5000/oeasy/openjdk:8-alpine
	 ---> b91a9b712b0a
	Step 2/8 : ENV TZ "Asia/Shanghai"
	 ---> Using cache
	 ---> 6c0760debdd6
	Step 3/8 : RUN export LANG="zh_CH.UTF-8"
	 ---> Using cache
	 ---> 20446bbd6990
	Step 4/8 : ARG jar_path
	 ---> Using cache
	 ---> 4623a2b54189
	Step 5/8 : ADD $jar_path/*.jar /opt/
	 ---> Using cache
	 ---> 46cf26b006d1
	Step 6/8 : WORKDIR /opt/
	 ---> Using cache
	 ---> 756df3fc22c6
	Step 7/8 : ENV JAVA_OPTS "-Dspring.profiles.active=k8s_test -server -Xms128m -Xmx256m -XX:MaxMetaspaceSize=128m"
	 ---> Using cache
	 ---> 9c96fb8a50cb
	Step 8/8 : ENTRYPOINT sh -c java $JAVA_OPTS -jar /opt/*.jar
	 ---> Using cache
	 ---> 4b46ce12f23c
	Successfully built 4b46ce12f23c

运行 docker build 命令，-t 将新镜像命名为192.168.0.114:5000/oeasy/yihao01-permission-management-6810:1.0.0，其中192.168.0.114:5000是docker镜像仓库的地址，1.0.0是版本号。

命令末尾的“ .” 指明 build context 为当前目录。Docker 默认会从 build context 中查找 Dockerfile 文件，我们也可以通过 -f 参数指定 Dockerfile 的位置。

> 提示：Docker 将 build context 中的所有文件发送给 Docker daemon。build context 为镜像构建提供所需要的文件或目录。Dockerfile 中的 ADD、COPY 等命令可以将 build context 中的文件添加到镜像。此例中，build context 为当前目录 ，该目录下的所有文件和子目录都会被发送给 Docker daemon。
> 
>所以， 使用 build context 就得小心了，不要将多余文件放到 build context，特别不要把 /、/usr 作为 build context，否则构建过程会相当缓慢甚至失败。

编译成功，镜像的ID为4b46ce12f23c，然后将镜像push到仓库

	[root@docker iot_images]# docker push 192.168.0.114:5000/oeasy/yihao01-permission-management-6810:1.0.0
	The push refers to a repository [192.168.0.114:5000/oeasy/yihao01-permission-management-6810]
	6d40760f0cab: Layer already exists 
	09dbf8d3eae2: Layer already exists 
	c5c6d3823c72: Layer already exists 
	f592f00280f8: Layer already exists 
	5c843e862cab: Layer already exists 
	8ade71a1bece: Layer already exists 
	01bca7cb4826: Layer already exists 
	a8cc3712c14a: Layer already exists 
	cd7100a72410: Layer already exists 
	1.0.0: digest: sha256:0fa7930a9a5bb33a977a7ab5d9c7d06c112b7cd35ff1a36250a9af68159b8adc size: 2198

### 2.3 利用脚本构建和上传
当然我们也可以把上面的步骤写成shell脚本，目录结构和编译脚本如下:

	[root@docker iot_images]# ls
	build.sh  dockerfile  eureka  yihao01-permission-management-6810

	[root@docker iot_images]# cat build.sh 
	#/bin/bash
	name=$1
	version=$2
	images=$name":"$version
	
	docker build --build-arg jar_path=$name -t 192.168.0.114:5000/oeasy/$images -f dockerfile .
	docker push 192.168.0.114:5000/oeasy/$images
	
	[root@docker iot_images]sh build.sh yihao01-permission-management-6810 1.0.0

## 3. 利用kubernetes编排容器
我们已经了解到，Kubernetes 通过各种 Controller 来管理 Pod 的生命周期。为了满足不同业务场景，Kubernetes 开发了 Deployment、ReplicaSet、DaemonSet、StatefuleSet、Job 等多种 Controller。

### 3.1 利用Deployment创建应用

我们首先介绍最常用的 Deployment。Kubernetes 支持两种方式创建资源：

① 用 kubectl 命令直接创建，比如：

	kubectl run pm --image=192.168.0.114:5000/oeasy/yihao01-permission-management-6810:1.0.0 --replicas=2

在命令行中通过参数指定资源的属性。

② 通过配置文件和 kubectl apply 创建，首先创建配置文件pm.yaml：

	apiVersion: apps/v1beta1
	kind: Deployment
	metadata:
	  name: pm
	spec:
	  replicas: 1
	  template:
	    metadata:
	      labels:
	        run: pm
	    spec:
	      containers:
	      - name: pm
	        image: 192.168.0.114:5000/oeasy/yihao01-permission-management-6810:1.0.0
	        ports:
	        - containerPort: 6810 

我们逐行来看一下实现的功能。

① apiVersion 是当前配置格式的版本。

② kind 是要创建的资源类型，这里是 Deployment。

③ metadata 是该资源的元数据，name 是必需的元数据项。

④ spec 部分是该 Deployment 的规格说明。

⑤ replicas 指明副本数量，默认为 1。

⑥ template 定义 Pod 的模板，这是配置文件的重要部分。

⑦ metadata 定义 Pod 的元数据，至少要定义一个 label。label 的 key 和 value 可以任意指定。

⑧ spec 描述 Pod 的规格，此部分定义 Pod 中每一个容器的属性，name 和 image 是必需的。containerPort是容器需要暴露的端口，容器需要对外提供接口或者服务的时候，需要指定。

下面对这两种方式进行比较。

基于配置文件的方式 | 基于命令的方式 
------------ | -------------
配置文件描述了 What，即应用最终要达到的状态 | 简单直观快捷，上手快
配置文件提供了创建资源的模板，能够重复部署  |  --
可以像管理代码一样管理部署 |  --
适合正式的、跨环境的、规模化部署 | 适合临时测试或实验
方式要求熟悉配置文件的语法，有一定难度 | 简单

### 3.2 创建和查看Deployment

在master节点上创建pm.yaml文件，利用apply命令部署

	[devops@master ~]$ kubectl apply -f pm.yaml

> 使用kubectl -h命令可以查看帮助。

创建完毕后可以通过get -f 命令，查看为该yaml文件分配的资源，判断是否创建成果。

	[devops@master ~]$ kubectl get -f pm.yaml 
	NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
	pm        1         1         1            1           21d

	NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
	pm-svc    ClusterIP   10.108.165.235   <none>        6810/TCP   21d

当然你也可以单独查看deployment、pod的具体信息等。

	[devops@master ~]$ kubectl -n iot get pods -o wide | grep pm
	pm-75d9b84cbb-nmgqc           1/1       Running   4          21d       10.244.2.119   node2
