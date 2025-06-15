+++
date = '2025-03-07T20:48:12+08:00'
draft = true
title = '使用kind和mirrod在macOS搭建k8s多node测试环境'
+++

# macOS上使用Docker Desktop, obstark, minikube运行k8s的局限性
- 对多node的支持较差或者根本不支持
- 受限于docker在macOS上的虚拟机实现，宿主机网络无法直接连通容器

# 什么是kind
kind 是一个使用 Docker 容器作为"节点"来运行本地 Kubernetes 集群的工具。它主要用于测试、开发和学习目的。kind 的主要优点是:

- 快速创建和销毁集群
- 支持多节点集群
- 轻量级且易于使用
- 支持 Kubernetes 的大部分功能

官方网站: https://kind.sigs.k8s.io/

# 什么是mirrod
mirrord 是一个强大的开发工具，它允许你在本地运行应用程序时，直接连接到 Kubernetes 集群中的网络和资源。主要特点包括：

- 无需修改代码即可访问集群资源
- 支持本地调试远程应用
- 可以拦截和重定向网络流量
- 支持多种编程语言

官方网站: https://mirrord.dev/

在这里，我们可以使用mirrord来让本地的进程访问到k8s集群网络，这个方案没有任何侵略性。


# 环境安装步骤
## 安装Docker Desktop或者OrbStack
1. Docker Desktop 安装：
   - 从 Docker 官网下载 Docker Desktop for Mac
   - 安装并启动 Docker Desktop

2. OrbStack 安装（替代方案）：
   - 从官网下载安装包
   - OrbStack 相比 Docker Desktop 更轻量且性能更好

## 安装kind
使用 Homebrew 安装：
```
brew install kind
```

## kind创建集群
kind集群配置文件kind-multi-node.yaml，这个配置文件声明了一个control-plane节点和三个worker节点
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: worker
  extraMounts: # node挂载目录
  - hostPath: /Users/kernal/storage/openebs/node1 # 宿主机目录
    containerPath: /root # node容器目录
- role: worker
  extraMounts:
    - hostPath: /Users/kernal/storage/openebs/node2
      containerPath: /root
- role: worker
  extraMounts:
  - hostPath: /Users/kernal/storage/openebs/node3
    containerPath: /root
- role: control-plane
  extraPortMappings: # nodePort要放开到宿主机端口，这里可以多填几个，因为后面发现不够再加需要删除并且重新创建集群
    - containerPort: 30000 # node容器内部的端口号
      hostPort: 30000 # 宿主机的端口号
      protocol: TCP # 网络协议
```

执行命令集群
```
kind create cluster --config kind-multi-node.yaml
```

## kind删除集群

```
kind kind delete cluster
```

## kubectl验证集群是否安装成功
```
kubectl get node

NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   10s   v1.32.2
kind-worker          Ready    <none>          10s   v1.32.2
kind-worker2         Ready    <none>          10s   v1.32.2
kind-worker3         Ready    <none>          10s   v1.32.2
```

## 安装mirrord
```
brew install metalbear-co/mirrord/mirrord
```
或者
```
curl -fsSL https://raw.githubusercontent.com/metalbear-co/mirrord/main/scripts/install.sh | bash
```

## mirrord用法
```
mirrord exec --target pod/app-pod-01 python main.py
```
或者不指定target，相当于直接把进程放到k8s集群中执行
```
mirrord exec python main.py
```

## 测试运行mirrod
我们先apply一个nginx.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```
这个nginx的svc为http://nginx-service.default.svc.cluster.local

直接在宿主机运行
```
curl http://nginx-service.default.svc.cluster.local
```
结果为curl: (56) Recv failure: Connection reset by peer/

使用mirrod
```
mirrord exec -- curl http://nginx-service.default.svc.cluster.local
```
结果
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

这样，macOS上多node且和宿主机网络互通的k8s开发环境就搭建完成了。