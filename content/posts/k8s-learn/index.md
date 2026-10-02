+++
date = '2026-09-23T10:41:13+08:00'
lastmod = '2026-10-02T23:14:30+08:00'
title = 'k8s 学习笔记'
weight = -1
categories = ['Ops']
tags = ['k8s', 'Minikube']
+++

## Minikube 启动及代理配置

在宿主机上以一个 docker 容器的形式启动 Minikube 集群

```sh
minikube start --driver=docker
```

Minikube 使用 192.168.49.0/24 网段，需在宿主机的代理配置中添加相应监听

对于代理配置，Minikube 现默认使用 containerd 作为内部的容器运行时，因此 --docker-env、在容器内部设置 /etc/systemd/system/docker.service.d/xxx.conf 等方式均不会生效，应在容器内部编辑 /etc/systemd/system/containerd.service.d/xxx.conf 文件

```sh
minikube ssh
sudo mkdir /etc/systemd/system/containerd.service.d
sudo vi /etc/systemd/system/containerd.service.d/http-proxy.conf
```

```ini
[Service]
Environment="HTTP_PROXY=http://192.168.49.1:10808"
Environment="HTTPS_PROXY=http://192.168.49.1:10808"
Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
```

```sh
sudo systemctl daemon-reload
sudo systemctl restart containerd
exit
```

随后在宿主机移除因网络错误而拉取失败的 kindnet Pod，令其自动重新拉取

```sh
minikube kubectl -- get pods -A
minikube kubectl -- delete pod kindnet-<podno> -n kube-system
```

## 架构

- Control Plane 控制面
- Node 节点

一个控制面/节点（也可以两者同时是）对应一个物理/虚拟机

> Minikube 默认情况下在宿主机上启动一个大 Docker 容器，兼任控制面和节点，在该容器内部使用 containerd 作为容器运行时，启动各组件容器

- Pod
- ReplicaSet
- Deployment

Pod 是一个原子性的（临时性的）容器运行的环境，ReplicaSet 维持 Pod 的副本数，（控制面负责在节点上调度 Pod，）Deployment 管理 ReplicaSet

- Service

Service 是一层抽象，使 k8s 中 Pod 的死亡、复制等不影响应用，提供外部流量公开、负载平衡和服务发现

## 常用命令

查看当前集群中的 Pod/Service/Deployment/ReplicaSet 等资源对象

```sh
kubectl get pods/services/deployments/rs
```

详细信息

```sh
kubectl describe pod/<pod_name>
```

在 Pod 内执行命令，k8s 保证在同一命名空间内 Pod 名唯一

```sh
kubectl exec <pod_name> -- <command>
```

创建一个名为 kubernetes-bootcamp 的 Deployment（默认配置为一个 Pod）

```sh
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
```

向外暴露服务

> 暴露 kubernetes-bootcamp 这个 Deployment 的 8080 端口，k8s 会随机分配一个宿主机端口

> 在 Minikube 环境下，通过 `minikube ip` 获取其内部网关 ip

> LoadBalancer、NodePort、ClusterIP 三种类型，前者都是后者的超集

```sh
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
```

使用 Pod/Service 标签来查询

```sh
kubectl get pods/services -l app=kubernetes-bootcamp
```

添加标签

```sh
kubectl label pods <pod_name> <key>=<value>
```

扩缩 Deployment 的副本数（相应 Service 应使用 LoadBalancer 类型）

```sh
kubectl scale deployments/kubernetes-bootcamp --replicas=4
```

设置镜像版本，这将触发 Deployment 的滚动更新

> 第一个 kubernetes-bootcamp 为 Deployment 名称，第二个为 Pod 中的容器名称

```sh
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2
```

回滚 Deployment

```sh
kubectl rollout undo deployments/kubernetes-bootcamp
```

## ......
