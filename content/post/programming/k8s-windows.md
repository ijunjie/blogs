---
title: "Windows 配置 docker 和 k8s 开发环境"
date: 2018-12-03
draft: false
tags:
- k8s
- docker
categories:
- programming
---


Docker for Windows 内置一个单节点 Kubernetes 集群，可以用于本地测试开发。



## 1. 安装 docker for windows 最新版



前提条件：打开 BIOS 中的虚拟化选项。



注意，安装 docker for windows 使用 hyper-v 做虚拟化，会导致 virtualbox 和 vmware 不可用。



## 2. 配置 docker 环境

配合 IDE 的docker 插件使用时，需要打开docker host，勾选 General 选项 ”Expose daemon on 2375” 即可。



## 3. 配置 k8s

在 Settings 中，配置 Kubernetes 选项。



在安装过程中，Docker 也为我们安装了 kubectl 控制命令：



```shell
$ kubectl get namespaces
$ kubectl get posts --namespace kube-system
```



## 4. 安装 dashboard



```shell
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```



验证：



```shell
kubectl get deployments --namespace kube-system
```



```shell
kubectl get services --namespace kube-system
```



在 Dashboard 启动完毕后，可以使用 kubectl 提供的 Proxy 服务来访问该面板：



```shell
kubectl proxy
```

访问：

`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`



如果不能访问，需要编辑 `kubectl -n kube-system edit service kubernetes-dashboard` 将 `type: ClusterIP`  改为 `type: NodePort` 并保存，再次访问即可。



显示界面后，可以先点“跳过”，进入到 dashboard.





Docker 同样为我们提供了简单的应用示范，可以直接使用如下的 Docker Compose 配置文件:



```
version: '3.3'

services:
  web:
    build: web
    image: dockerdemos/lab-web
    volumes:
     - "./web/static:/static"
    ports:
     - "80:80"

  words:
    build: words
    image: dockerdemos/lab-words
    deploy:
      replicas: 5
      endpoint_mode: dnsrr
      resources:
        limits:
          memory: 16M
        reservations:
          memory: 16M

  db:
    build: db
    image: dockerdemos/lab-db
```



然后使用 stack 命令创建应用栈：



```
docker stack deploy --compose-file stack.yml demo
```





应用栈创建完毕后，可以使用 kubectl 查看创建的 Pods:

```
kubectl get pods
```



也可以来查看部署的集群与服务：



```
kubectl get deployments
kubectl get services
```



最后我们还可以用 stack 与 kubectl 命令来删除应用：



```
docker stack remove demo
kubectl delete deployment kubernetes-dashboard --namespace kube-system
```





参考：

https://segmentfault.com/a/1190000012850396