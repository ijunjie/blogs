---
title: "Docker Machine"
date: 2018-06-13
draft: false
tags:
- docker
categories:
- arch
---

关于 Docker Machine 的一些实践总结。

Docker Machine 用于解决多主机环境部署 docker 的效率和一致性问题。 

创建 machine 要求能够无密码登录远程主机，所以需要先通过 ssh-copy-id 将 ssh key 拷贝到目标机器。

往普通的 Linux 中部署 docker，可以使用 generic driver.
 

创建完 machine 后，到 `/etc/systemd/system/docker.service.d/` 查看配置(笔者环境为 CentOS 7.5.1804, Docker 18.05.0-ce)，确认 docker-deamon 已经被 docker-machine 配置为接收远程访问。目标机器的 hostname 也会被设置。


docker-machine upgrade 可以批量更新 machine 的 docker 到最新版本，如 `docker-machine upgrade`.

**stop/start/restart 是对 machine 的操作系统操作，而 不是 stop/start/restart docker daemon**.

docker-machine scp 可以在不同 machine 之间拷贝文件，比如：

`docker-machine scp host1:/tmp/a host2:/tmp/b`

创建 docker-machine 时，由于网络问题经常安装失败，需要多尝试几次。

参考：[http://www.cnblogs.com/CloudMan6/p/7248188.html](http://www.cnblogs.com/CloudMan6/p/7248188.html)