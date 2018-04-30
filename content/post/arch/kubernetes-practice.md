---
title: "一个完整的 Kubernetes 微服务应用示例"
date: 2018-04-30
draft: false
tags:
- kubernetes
categories:
- arch
---



这是一个比较完整的基于 Kubernetes 的微服务开发、构建、部署和编排示例。


## TL; DR

如果已有 Kubernetes 环境，想快速部署并访问应用，可以按如下步骤，使用我做好的部署配置和容器镜像。

**下载代码库**

```shell
git clone https://github.com/ijunjie/sentiment-analyse.git
cd resource-manifests
```


**部署到 Kubernetes 环境**

```shell
kubectl create -f sa-logic-deployment.yaml
kubectl create -f service-sa-logic.yaml

kubectl create -f sa-web-app-deployment.yaml
kubectl create -f service-sa-web-app-lb.yaml

kubectl create -f sa-frontend-deployment-green.yaml
kubectl create -f service-sa-frontend-lb.yaml
```

**访问应用**

通过集群节点的 nodePort 访问 frontend service.



## 参考资料

本文主要的参考了 [Kubernetes and everything else in Practice](https://rinormaloku.com/kubernetes-everything-else-practice/)，但在一些细节方面根据实际情况做了调整。修改后的源码 [sentiment-analyse](https://github.com/ijunjie/sentiment-analyse)  和 [容器镜像](https://hub.docker.com/u/wangjunjie/) . 主要不同之处：

- 没有使用 minikube, 而是使用 kubekit 部署的多节点集群环境。
- 编排时的 Service 类型使用 NodePort 代替 LoadBalance.
- 原文中前端代码调用 web app 的 URL 是写死的，最后还要改成  LoadBalancer 地址并重新构建，`fetch('http://192.168.99.100:31691/sentiment'...`, 在代码中仍然硬编码 URL. 本文中尝试使用 nginx 配置来解决这个服务发现问题。
- 副本数量和部署策略做了调整。





## 应用简介

该应用提供 UI 界面，以一个句子作为输入，可以分析计算出句子的情感值。

这个应用程序由三个微服务组成：

- SA-Frontend：前端应用，使用ReactJS构建，部署在Nginx；
- SA-WebApp：Java Web 应用，处理来自前端的请求，使用 Spring boot 构建，使用内嵌tomcat的jar包部署。
- SA-Logic：Python应用， 使用 Flask 构建，处理分析逻辑。





## Kubernetes 环境搭建

[kubekit](https://github.com/Orientsoft/kubekit)， 一个离线环境搭建 Kubernetes 集群环境的利器，可以说是良心之作。按照说明可以快速搭建一个小型的 Kubernetes 集群环境。

注意事项：三个节点要关闭 firewalld, selinux 以避免一些不必要的麻烦。



## 服务发现

前端 sentiment-analyse/blob/master/sa-frontend/src/App.js 中，访问 webapp 的URL不带域且增加了/api前缀： `/api/sentiment`. 

开发环境下，这个 URL 应改为 webapp 的实际 endpoint,  有完整的域且不带/api前缀, 如 `http://localhost:8080/sentiment`.

生产环境下可以通过 nginx 配置，将`/api/sentiment` 请求重写为 `/sentiment`， 并转发到 webapp 的地址。 Kubernetes环境下，利用服务发现，使用 Service 名即可。




App.js中访问 webapp: 

```javascript
analyzeSentence() {
        fetch('/api/sentiment', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({sentence: this.textField.getValue()})
        })
            .then(response => response.json())
            .then(data => this.setState(data));
    }
```

nginx 配置, 使用 webapp 的 Service 名做服务发现：

```nginx
    location /api/ {
        rewrite ^.+api/?(.*)$ /$1 break;
        include uwsgi_params;
        proxy_pass http://sa-web-app-lb/;
    }
```



参考 :

- [https://segmentfault.com/a/1190000010792260](https://segmentfault.com/a/1190000010792260)
- [https://segmentfault.com/a/1190000011796903](https://segmentfault.com/a/1190000011796903)





在 sentiment-analyse/resource-manifests/sa-web-app-deployment.yaml 中也使用了 logic 的服务地址：


```yaml
env:
  - name: SA_LOGIC_API_URL
    value: "http://sa-logic"
```

