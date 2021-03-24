---
title: "Windows 配置 docker 和 k8s 开发环境"
date: 2018-12-03
draft: true
tags:
- k8s
- docker
categories:
- programming
---









更新：请参考 <https://github.com/AliyunContainerService/k8s-for-docker-desktop> 



## 1. 安装 docker for windows



打开 BIOS 中的虚拟化选项。



注意，开启 hyper-v 可能会导致 virtualbox 和 vmware 不可用。



## 2. 配置 docker 环境

如需启用 docker host，打开 General  ”Expose daemon on 2375” 。



## 3. 配置 k8s

安装过程中会自动安装 kubectl

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