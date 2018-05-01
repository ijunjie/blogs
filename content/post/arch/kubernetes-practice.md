---
title: "一个 Kubernetes 微服务应用示例"
date: 2018-04-30
draft: false
tags:
- kubernetes
categories:
- arch
---



[Rinor Maloku ](https://rinormaloku.com/me/) 在 [Kubernetes and everything else in Practice](https://rinormaloku.com/kubernetes-everything-else-practice/) 一文中全面、细致地讲述了一个微服务应用的开发、构建，并部署到 Kubernetes 的过程。全文洋洋洒洒分为十一篇，并配有精美的动态演示图，实为入门 Kubernetes 的良心之作。如果链接打不开，可以访问 DockOne 提供的中文版：[三小时学会Kubernetes：容器编排详细指南](http://www.dockone.io/article/5132)。

本文根据实际操作情况，对代码和部署做了一些调整：

- 使用多节点集群代替 minikube；
- Service 类型使用 NodePort 代替 LoadBalancer；
- 调整前端代码请求 webapp 的 URL 并使用 nginx 解决跨域问题；
- 调整了副本数量和部署策略。

修改后的源码：[sentiment-analyse](https://github.com/ijunjie/sentiment-analyse)




## TL; DR

**下载代码库**

```shell
git clone https://github.com/ijunjie/sentiment-analyse.git
cd sentiment-analyse/resource-manifests
```


**部署到 Kubernetes 环境**

```shell
kubectl create -f sa-logic-deployment.yaml
kubectl create -f service-sa-logic.yaml

kubectl create -f sa-web-app-deployment.yaml
kubectl create -f service-sa-web-app-lb.yaml

kubectl create -f sa-frontend-deployment-fixed.yaml
kubectl create -f service-sa-frontend-lb.yaml
```

**访问应用**

查看 NodePort：

```shell
kubectl describe service sa-frontend-lb
```

使用浏览器访问集群节点的 NodePort 即可。




## 应用简介

该应用以一个句子作为输入，可以分析计算出句子的情感值。

该应用由三个微服务组成：

- SA-Frontend：前端应用，使用ReactJS构建，部署在Nginx；
- SA-WebApp：Java Web 应用，处理来自前端的请求，使用 Spring boot 构建，使用内嵌tomcat的jar包部署。
- SA-Logic：Python应用， 使用 Flask 构建，处理分析逻辑。





## Kubernetes 环境搭建

在这里推荐一个离线搭建 Kubernetes 集群的工具： [kubekit](https://github.com/Orientsoft/kubekit) ，可以在不联网的环境下快速搭建一套三个节点的 Kubernetes 集群环境。可以说是非常良心了。需要注意的是，三个虚拟机要关闭防火墙、selinux等以避免一些不必要的麻烦。



注意事项：三个节点要关闭 firewalld, selinux 以避免一些不必要的麻烦。



## 服务发现

sa-frontend/src/App.js 中，访问 webapp 的URL不带域且增加了/api前缀： `/api/sentiment` . 

- 开发环境下，这个 URL 应改为 webapp 的实际 endpoint,  有完整的域且不带/api前缀, 如 `http://localhost:8080/sentiment`.
- 生产环境下可以通过 nginx 配置，将`/api/sentiment` 请求重写为 `/sentiment`， 并转发到 webapp 的地址。 Kubernetes 环境中使用 Service 名即可。



在nginx 配置中, 使用 webapp 的 Service 名做服务发现 `http://sa-web-app-lb/` ：

```nginx
    location /api/ {
        rewrite ^.+api/?(.*)$ /$1 break;
        include uwsgi_params;
        proxy_pass http://sa-web-app-lb/;
    }
```




在 resource-manifests/sa-web-app-deployment.yaml 中也使用了 logic 的服务地址：


```yaml
env:
  - name: SA_LOGIC_API_URL
    value: "http://sa-logic"
```



跨域问题相关参考资料：

- [https://segmentfault.com/a/1190000010792260](https://segmentfault.com/a/1190000010792260)
- [https://segmentfault.com/a/1190000011796903](https://segmentfault.com/a/1190000011796903)

