---
title: "k8s搭建"
date: 2021-12-10T14:23:48+08:00
draft: true
---









**本文是以CentOS 8.2 64位 **

**CPU&内存 2核4 GiB 来部署的**

**单机部署**

### 1. 安装docker

[找到自己的系统](https://hub.docker.com/search?q=&type=edition&offering=community&sort=updated_at&order=desc) 

官网安装的话需要注册账号，选择个人的就可以直接用了

可以安装[官网安装步骤](https://hub.docker.com/search?q=&type=edition&offering=community&sort=updated_at&order=desc)安装

但centos8没必要那么麻烦

直接

```
dnf install docker-ce docker-ce-cli containerd.io
```

然后启动docker

```
 sudo systemctl start docker
```

接着确定docker安装成功

```
sudo docker run hello-world
```



### 2. 安装minikube

要想本地部署Kubernetes需要下载minikube

这部分官网讲的很好了，按照[官网安装](https://minikube.sigs.k8s.io/docs/start/)

当然启动一般会报错

minikube不能用root身份启动

su - root 切换到root

```shell
useradd testuser
passwd redhat
```

创建用户

创建docker组并将用户加入docker组，并激活组的更改

```shell
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

切换用户 su - testuser

```shell
minikube start --driver=docker
```

当然由于没有安装kubectl还会报错



### 3. 安装kubectl

完全按照[官网安装](https://kubernetes.io/zh/docs/tasks/tools/install-kubectl-linux/)就可

这个时候运行minikube start --driver=docker

```shell
minikube start --driver=docker
```

然后再运行

```
kubectl cluster-info
```

出现

```shell
Kubernetes control plane is running at https://测试ip
CoreDNS is running at https://测试ip:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

```

代表运行成功

