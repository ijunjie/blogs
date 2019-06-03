---
title: "Docker 拾遗"
date: 2018-05-19
draft: false
tags:
- arch
- docker
categories:
- programming
---


## 一、镜像的分层结构

- 可写容器层。容器启动时，一个可写层被加载到镜像的顶部。所有对容器的改动，只会发生在容器层。
- 叠加文件系统，如果不同层中有一个相同路径的文件，比如 /a，上层的 /a 会覆盖下层的 /a，也就是说用户只能访问到上层中的文件 /a。在容器层中，用户看到的是一个叠加之后的文件系统。
- Copy-On-Write, 只有当需要修改时才复制一份数据，这种特性被称作 Copy-on-Write。容器层保存的是镜像变化的部分，不会对镜像本身进行任何修改。
	- 添加文件：在容器中创建文件时，新文件被添加到容器层中。
	- 读取文件：在容器中读取某个文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后打开并读入内存。
	- 修改文件：在容器中修改已存在的文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后修改之。
	- 删除文件：在容器中删除文件时，Docker 也是从上往下依次在镜像层中查找此文件。找到后，会在容器层中记录下此删除操作。

## 二、容器的资源限额

### CPU 限额

默认设置下，所有容器可以平等地使用 host CPU 资源并且没有限制。通过 cpu share 可以设置容器使用 CPU 的优先级。

- 通过 -c 或 --cpu-shares 设置容器使用 CPU 的权重，默认为 1024

如在 host 中启动了两个容器：

```
docker run --name "container_A" -c 1024 ubuntu

docker run --name "container_B" -c 512 ubuntu
```

container_A 的 cpu share 1024，是 container_B 的两倍。当两个容器都需要 CPU 资源时，container_A 可以得到的 CPU 是 container_B 的两倍。

需要特别注意的是，这种按权重分配 CPU 只会发生在 CPU 资源紧张的情况下。如果 container_A 处于空闲状态，这时，为了充分利用 CPU 资源，container_B 也可以分配到全部可用的 CPU。

### 内存限额

容器可使用的内存包括两部分：物理内存和 swap, 默认都为 -1

- `-m` 或 `--memory`：设置内存的使用限额，例如 100M, 2G。
- `--memory-swap`：设置 内存+swap 的使用限额。 --memory-swap 默认为 momory 的两倍。

例如 `-m 200M --memory-swap=300M`



### Block IO 限额

默认情况下，所有容器能平等地读写磁盘。

#### 优先级设置

- 可以通过设置 --blkio-weight 参数来改变容器 block IO 的优先级。`--blkio-weight` 设置的是相对权重值，**默认为 500**. 

设置container_A 读写磁盘的带宽是 container_B 的两倍：

```
docker run -it --name container_A --blkio-weight 600 ubuntu   
	
docker run -it --name container_B --blkio-weight 300 ubuntu
```
#### 具体设置

限制 bps 和 iops. bps 是 byte per second，每秒读写的数据量。iops 是 io per second，每秒 IO 的次数。可通过以下参数控制容器的 bps 和 iops：

- --device-read-bps，限制读某个设备的 bps。
- --device-write-bps，限制写某个设备的 bps。
- --device-read-iops，限制读某个设备的 iops。
- --device-write-iops，限制写某个设备的 iops。

## 三、容器限额实验

progrium/stress 镜像的使用

### 测试 CPU 限额

1. 启动 container_A, `cpu_share` 设置为 1024
	
	```
	docker run --name container_A -it -c 1024 progrium/stress --cpu 1
	```
	`--cpu` 用来设置工作线程的数量。因为当前 host 只有 1 颗 CPU，所以一个工作线程就能将 CPU 压满。
	
2. 启动 container_B, `cpu_share` 设置为 512：

	```
	docker run --name container_B -it -c 512 progrium/stress --cpu 1
	```
	
	在 host 中执行 `top`, 可以看到两个容器分别使用了 66.3% 和 33.3% 左右。
	
4. 此时暂停 container_A, 可以观察到 container_B 在 container_A 空闲的情况下能够用满整颗 CPU.


### 测试内存限额

```
docker run -it -m 200M --memory-swap=300M progrium/stress --vm 1 --vm-bytes 280M
```

`--vm 1`：启动 1 个内存工作线程。

`--vm-bytes 280M`：每个线程分配 280M 内存。

因为 280M 在可分配的范围（300M）内，所以工作线程能够正常工作，其过程是：

1. 分配 280M 内存。

2. 释放 280M 内存。

3. 再分配 280M 内存。

4. 再释放 280M 内存。

一直循环......

### 测试 Block IO 限额

限制容器写 `/dev/sda` 的速率为 30 MB/s

```
docker run -it --device-write-bps /dev/sda:30MB ubuntu
```

通过 dd 测试在容器中写磁盘的速度。因为容器的文件系统是在 host /dev/sda 上的，在容器中写文件相当于对 host /dev/sda 进行写操作。另外，oflag=direct 指定用 direct IO 方式写文件，这样 --device-write-bps 才能生效。

```
docker run -it --device-write-bps /dev/sda:30MB ubuntu
```

```
time dd if=/dev/zero of=test.out bs=1M count=800 oflag=direct
```

结果表明 bps 没有超过 30 MB/s 的限速。

## 四、单机容器网络

### bridge

Docker 安装时会创建一个 命名为 docker0 的 linux bridge。如果不指定 `--network`，创建的容器默认都会挂到 docker0 上。

使用 `docker network create --driver bridge mynet` 创建自定义网络。使用 `brctl show` 查看发现新增一个网桥。

创建网络时可以指定 `--subnet` 和 `--gateway` 参数。


可以通过 `--ip` 指定容器的 IP，前提是网络使用 `--subnet`创建。


为 container 增加一个某个网络的网卡：

`docker netwrok connect <net-name> <container-id>`

### DNS Server

从 Docker 1.10 版本开始，docker daemon 实现了一个内嵌的 DNS server，使容器可以直接通过**容器名**通信。方法很简单，只要在启动时用 `--name` 为容器命名就可以了。

**只能在 user-defined 网络中使用。也就是说，默认的 bridge 网络是无法使用 DNS 的。**

### joined 容器

joined 容器非常特别，它可以使两个或多个容器共享一个网络栈，共享网卡和配置信息，joined 容器之间可以通过 127.0.0.1 直接通信。例如，创建一个 httpd 容器，名字为 web1：

```
docker run -d -it --name=web1 httpd
```

然后创建 `busybox` 容器并通过 `--network=container:web1` 指定 jointed 容器为 web1. 格式为 `--network=container:<container-name>`

```
docker run -it --network=container:web1 busybox
```

busybox 和 web1 的网卡 mac 地址与 IP 完全一样，它们共享了相同的网络栈。

