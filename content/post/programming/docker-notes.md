---
title: "Docker 基础知识整理"
date: 2018-05-19
draft: false
tags:
- arch
- docker
categories:
- programming
---


[每天5分钟玩转 Docker 容器技术](http://www.cnblogs.com/CloudMan6/) 的学习笔记。


## 一、容器生态系统

### 容器核心技术

- 容器规范：OCI 发布的 runtime spec 和 image format spec
- 容器runtime：主流的三种 
	- lxc，Linux 老牌容器 runtime
	- runc，Docker 开发的
	- rkt，CoreOS 开发的
- 容器管理工具：
	- lxc：lxd， 
	- runc：docker engine，包括 deamon 和 cli
	- rkt：rkt cli
- 容器定义工具：
	- docker image & docker file
	- ACI，与 docker image 类似，CoreOS 开发
- Registries
	- Docker registry
	- Docker hub
	- Quay.io
- 容器 OS：
	- coreos
	- atomic
	- ubuntu core
		
### 容器平台技术：

- 容器编排引擎：
	- kubernetes
	- docker swarm
	- mesos + marathon
- 容器管理平台：
	- Rancher
	- ContainerShip
- 基于容器的PaaS：
	- Deis
	- Flynn
	- Dokku

### 容器支持技术：

- 容器网络
	- docker network
	- flannel
	- weave
	- calico
- 服务发现：
	- etcd
	- consul
	- zookeeper
- 监控
	- docker ps/top/stats
	- docker stats API
	- sysdig
	- aAdvisor/Heapster
	- Weave Scope
- 数据管理
	- Flocker 数据管理工具
- 日志管理
	- docker logs
	- logsout
- 安全性
	- OpenSCAP 能够对容器镜像进行扫描，发现潜在的漏洞

## 二、容器架构

### 容器与虚拟机：

- 更小体积
- 启动速度快
- 开销小
- 更容易迁移

### Docker 核心组件：

- Client
- Docker daemon
- Image
- Registry
- Container

## 三、镜像

### base 镜像

所有容器共用 Host 的 kernel. 如果容器对 kernel 版本有要求，则不适合使用 Docker。

### 分层结构

- 可写容器层
- 叠加文件系统
- Copy-On-Write

### 构建镜像：

- `docker commit <current_container_name> <new_image_name>`
- dockerfile. 底层实际也是通过 docker commit 一层一层构建的。
	- 调试 Dockerfile：例如，第三层镜像构建失败，可以手动启动一个第二层的容器查看失败原因。
	- `docker history <image_name>` 会显示镜像的构建历史，也就是 Dockerfile 的执行过程。

### 镜像的缓存特性

Docker 会缓存已有镜像的镜像层，构建新镜像时，如果某镜像层已经存在，就直接使用，无需重新创建。

### Dockerfile 常用指令

- FROM
- MAINTAINER
- COPY
- ADD 自动解压 tar, zip, tgz,xz 等
- ENV, 如 `ENV MYSQL_HOST "10.153.46.32"`,后续指令以 `$MYSQL_HOST`获取
- EXPOSE
- VOLUME
- WORKDIR, 指定容器内工作目录，如果 WORKDIR 不存在，Docker 会自动创建。如 `WORKDIR /app` 则后面指令涉及容器内路径时，相对于`/app`
- RUN
- CMD
- ENTRYPOINT

## Shell 和 Exec 格式的命令

- Shell 格式的命令会调用 `/bin/sh -c <command>`, command会被shell解析。
	```
	ENV name Foo Bar
	ENTRYPOINT echo "Hello, $name"
	```
	启动容器输出 `Hello, Foo Bar`. （注意 ENV 的用法 ENV name Foo Bar 可以写成 ENV name "Foo Bar"）

- Exec 格式的命令会直接执行 command，不会解析。如上面例子，使用Exec：
	```
	ENV name Foo Bar  
	ENTRYPOINT ["/bin/echo", "Hello, $name"]
	```
	输出 `Hello, $name`

### ENTRYPOINT & CMD
	
```
ENTRYPOINT ["/bin/echo", "Hello"]  
CMD ["world"]
```

此时，可以在启动容器时传入参数，覆盖 CMD 提供的内容。

`docker run -it [image]` 输出 `Hello world`;
`docker run -it [image] Foo Bar Haha` 输出 `Hello Foo Bar Haha`;

**`ENTRYPOINT command param1 param2` 会忽略任何 CMD 或 docker run 提供的参数。**

### 镜像的命名

一个特定镜像的名字由两部分组成：repository 和 tag:

`[repository]:[tag]`


如果执行 docker build 时没有指定 tag，会使用默认值 `latest`. 其效果相当于：

`docker build -t ubuntu-with-vi:latest`

### 镜像的分发

- 提供 Dockerfile 给他人自行构建
- 上传到 registry 供别人下载
	- 公共 Registry
	- 私有 Registry

## 四、容器

### 服务类容器和工具类容器

- 服务类容器以 daemon 的形式运行，对外提供服务。比如 web server，数据库等。通过 `-d` 以后台方式启动这类容器是非常合适的。如果要排查问题，可以通过 `exec -it` 进入容器。
- 工具类容器通常给能我们提供一个临时的工作环境，通常以 `run -it` 方式运行。


### docker 容器操作

- attach 直接进入容器 启动命令 的终端，不会启动新的进程
- exec 则是在容器中打开新的终端，并且可以启动新的进程。`docker exec -it <container> bash|sh` 是执行 exec 最常用的方式。
- `docker logs -f` 也比较常用。
- 重命名容器, docker rename
- docker stop
- docker kill
- docker start, 保留容器的第一次启动时的所有参数
- docker restart, 相当于依次执行 docker stop 和docker start.
- docker create 创建的容器处于 Created 状态。可以先创建容器，稍后再启动。
- docker start 将以后台方式启动容器。 docker run 命令实际上是 docker create 和 docker start 的组合。
- `--restart=always` 意味着无论容器因何种原因退出（包括正常退出），就立即重启。该参数的形式还可以是 `--restart=on-failure:3`，意思是如果启动进程退出代码非0，则重启容器，最多重启3次。
- 有时我们只是希望暂时让容器暂停工作一段时间，比如要对容器的文件系统打个快照，或者 dcoker host 需要使用 CPU，这时可以执行 `docker pause`. 处于暂停状态的容器不会占用 CPU 资源，直到通过 docker unpause 恢复运行。
- docker rm 一次可以指定多个容器，如果希望批量删除所有已经退出的容器，可以执行如下命令：`docker rm -v $(docker ps -aq -f status=exited)`

### 限额

- CPU 限额，默认设置下，所有容器可以平等地使用 host CPU 资源并且没有限制。通过 cpu share 可以设置容器使用 CPU 的优先级。通过 `-c` 或 `--cpu-shares` 设置容器使用 CPU 的权重，默认为 1024

- 内存限额，容器可使用的内存包括两部分：物理内存和 swap, 默认都为 -1
	- `-m` 或 `--memory`：设置内存的使用限额，例如 100M, 2G。
	- `--memory-swap`：设置 内存+swap 的使用限额。 --memory-swap 默认为 momory 的两倍

	例如 `-m 200M --memory-swap=300M`

- Block IO 限额，默认情况下，所有容器能平等地读写磁盘
	- 可以通过设置 --blkio-weight 参数来改变容器 block IO 的优先级。`--blkio-weight` 设置的是相对权重值，默认为 500. 
	- 限制 bps 和 iops. bps 是 byte per second，每秒读写的数据量。iops 是 io per second，每秒 IO 的次数。可通过以下参数控制容器的 bps 和 iops：
		- --device-read-bps，限制读某个设备的 bps。
		- --device-write-bps，限制写某个设备的 bps。
		- --device-read-iops，限制读某个设备的 iops。
		- --device-write-iops，限制写某个设备的 iops。



### 容器底层原理

#### cgroup

cgroup 实现资源限额， namespace 实现资源隔离。

cgroup 全称 Control Group。Linux 操作系统通过 cgroup 可以设置进程使用 CPU、内存 和 IO 资源的限额。--cpu-shares、-m、--device-write-bps 实际上就是在配置 cgroup.

在 /sys/fs/cgroup/cpu/docker/<container_id>/ 下的文件中记录了容器资源限额对应到 cgroup 的配置。

#### namespace

在每个容器中，我们都可以看到文件系统，网卡等资源，这些资源看上去是容器自己的。拿网卡来说，每个容器都会认为自己有一块独立的网卡，即使 host 上只有一块物理网卡。这种方式非常好，它使得容器更像一个独立的计算机。

Linux 实现这种方式的技术是 namespace. namespace 管理着 host 中全局唯一的资源，并可以让每个容器都觉得只有自己在使用它。换句话说，namespace 实现了容器间资源的隔离。

Linux 使用了六种 namespace，分别对应六种资源：Mount、UTS、IPC、PID、Network 和 User.

- Mount namespace 让容器看上去拥有整个文件系统。容器有自己的 / 目录，可以执行 mount 和 umount 命令。当然我们知道这些操作只在当前容器中生效，不会影响到 host 和其他容器。
- UTS namespace 让容器有自己的 hostname。 **默认情况下，容器的 hostname 是它的短ID**，可以通过 -h 或 --hostname 参数设置。
- IPC namesapce 让容器拥有自己的共享内存和信号量（semaphore）来实现进程间通信，而不会与 host 和其他容器的 IPC 混在一起。
- PID namespace 容器在 host 中以进程的形式运行。例如当前 host 中运行了的容器，可以使用 `ps axf` 查看容器进程。所有容器的进程都挂在 dockerd 进程下。容器中进程的 PID 不同于 host 中对应进程的 PID，容器中 PID=1 的进程当然也不是 host 的 init 进程。也就是说：容器拥有自己独立的一套 PID，这就是 PID namespace 提供的功能。
- Network namespace 让容器拥有自己独立的网卡、IP、路由等资源。我们会在后面网络章节详细讨论。
- User namespace 让容器能够管理自己的用户，host 不能看到容器中创建的用户。

## 五、容器网络

- none，挂在这个网络下的容器除了 lo，没有其他任何网卡。容器创建时，可以通过 --network=none 指定使用 none 网络。比如某个容器的唯一用途是生成随机密码，就可以放到 none 网络中避免密码被窃取
- host，容器的网络配置与 host 完全一样。可以通过 --network=host 指定使用 host 网络。直接使用 Docker host 的网络最大的好处就是性能，如果容器对网络传输效率有较高要求，则可以选择 host 网络。当然不便之处就是牺牲一些灵活性，比如要考虑端口冲突问题，Docker host 上已经使用的端口就不能再用了。
- bridge，默认网络

### bridge

Docker 安装时会创建一个 命名为 docker0 的 linux bridge。如果不指定 `--network`，创建的容器默认都会挂到 docker0 上。

使用 `docker network create --driver bridge mynet` 创建自定义网络。使用 `brctl show` 查看发现新增一个网桥。

创建网络时可以指定 `--subnet` 和 `--gateway` 参数。


### 指定容器 IP

可以通过--ip指定容器的 IP，前提是网络使用 --subnet 创建。


### 互通性

为 container 增加一个某个网络的网卡：

`docker netwrok connect <net-name> <container-id>`

### 通信方式

- IP 通信
- Docker DNS Server
- joined 容器

#### IP 方式

两个容器要能通信，必须要有属于同一个网络的网卡。
满足这个条件后，容器就可以通过 IP 交互了。具体做法是在容器创建时通过 --network 指定相应的网络，或者通过 docker network connect 将现有容器加入到指定网络。

#### Docker DNS Server

通过 IP 访问容器虽然满足了通信的需求，但还是不够灵活。因为我们在部署应用之前可能无法确定 IP，部署之后再指定要访问的 IP 会比较麻烦。对于这个问题，可以通过 docker 自带的 DNS 服务解决。

从 Docker 1.10 版本开始，docker daemon 实现了一个内嵌的 DNS server，使容器可以直接通过“容器名”通信。方法很简单，只要在启动时用 --name 为容器命名就可以了。

**使用 docker DNS 有个限制：只能在 user-defined 网络中使用。也就是说，默认的 bridge 网络是无法使用 DNS 的。**

#### joined 容器

joined 容器是另一种实现容器间通信的方式。

joined 容器非常特别，它可以使两个或多个容器共享一个网络栈，共享网卡和配置信息，joined 容器之间可以通过 127.0.0.1 直接通信。请看下面的例子：

先创建一个 httpd 容器，名字为 web1。

`docker run -d -it --name=web1 httpd`

然后创建 busybox 容器并通过 --network=container:web1 指定 jointed 容器为 web1. 格式为 `--network=container:<container-name>`

busybox 和 web1 的网卡 mac 地址与 IP 完全一样，它们共享了相同的网络栈。

### 容器与外部网络通信

容器默认就能访问容器网络以外的网络环境。通过NAT实现。

### 外部网络访问容器

`docker run -d -p 80 httpd` 只有 80 时，host 端口随机分配。通过 `docker port <container-id>`查看。

除了映射动态端口，也可在 -p 中指定映射到 host 某个特定端口，例如可将 80 端口映射到 host 的 8080 端口：`-p 8080:80`.

每一个映射的端口，host 都会启动一个 docker-proxy 进程来处理访问容器的流量。

外部访问容器过程，例如 httpd 容器 host 端口绑定 `32723:80` 时：

1. docker-proxy 监听 host 的 32773 端口。

2. 当 curl 访问 host的 32773 时，docker-proxy 转发给容器的80端口。

3. httpd 容器响应请求并返回结果。


## 六、Docker 存储

Docker 为容器提供了两种存放数据的资源：

- 由 storage driver 管理的镜像层和容器层。

- Data Volume.


### Docker storage driver

正是 storage driver 实现了多层数据的堆叠并为用户提供一个单一的合并之后的统一视图。	

Docker 支持多种 storage driver，有 AUFS、Device Mapper、Btrfs、OverlayFS、VFS 和 ZFS。它们都能实现分层的架构，同时又有各自的特性。对于 Docker 用户来说，具体选择使用哪个 storage driver 是一个难题，因为：

没有哪个 driver 能够适应所有的场景。

driver 本身在快速发展和迭代。

不过 Docker 官方给出了一个简单的答案：

**优先使用 Linux 发行版默认的 storage driver**.

Docker 安装时会根据当前系统的配置选择默认的 driver。默认 driver 具有最好的稳定性，因为默认 driver 在发行版上经过了严格的测试。

运行docker info查看 Ubuntu 的默认 driver.


### Docker Volume

Data Volume 本质上是 Docker Host 文件系统中的目录或文件，能够直接被 mount 到容器的文件系统中。

Data Volume 有以下特点：

- Data Volume 是目录或文件，而非没有格式化的磁盘（块设备）。

- 容器可以读写 volume 中的数据。

- volume 数据可以被永久的保存，即使使用它的容器已经销毁。

好，现在我们有数据层（镜像层和容器层）和 volume 都可以用来存放数据，具体使用的时候要怎样选择呢？考虑下面几个场景：

- Database 软件 vs Database 数据

- Web 应用 vs 应用产生的日志

- 数据分析软件 vs input/output 数据

- Apache Server vs 静态 HTML 文件

相信大家会做出这样的选择：

前者放在数据层中。因为这部分内容是无状态的，应该作为镜像的一部分。

后者放在 Data Volume 中。这是需要持久化的数据，并且应该与镜像分开存放。

还有个大家可能会关心的问题：如何设置 voluem 的容量？

因为 volume 实际上是 docker host 文件系统的一部分，所以 volume 的容量取决于文件系统当前未使用的空间，**目前还没有方法设置 volume 的容量。**

### bind mount

在具体的使用上，docker 提供了两种类型的 volume：bind mount 和 docker managed volume。

bind mount 是将 host 上已存在的目录或文件 mount 到容器。

通过 -v 将其 mount 到 httpd 容器, -v 的格式为 `<host path>:<container path>`. 于 /usr/local/apache2/htdocs 已经存在，原有数据会被隐藏起来，取而代之的是 host $HOME/htdocs/ 中的数据，这与 linux mount 命令的行为是一致的。

bind mount 时还可以指定数据的读写权限，**默认是可读可写**，可指定为只读：`-v ~/htdocs:/usr/local/apache2/htdocs:ro`.

除了目录到目录，还可以文件到文件。只需要向容器添加文件，不希望覆盖整个目录。**使用单一文件有一点要注意：host 中的源文件必须要存在，不然会当作一个新目录 bind mount 给容器。**

bind mount 的使用直观高效，易于理解，但它也有不足的地方：**bind mount 需要指定 host 文件系统的特定路径，这就限制了容器的可移植性，当需要将容器迁移到其他 host，而该 host 没有要 mount 的数据或者数据不在相同的路径时，操作会失败。**

### Docker managed volume

docker managed volume 与 bind mount 在使用上的最大区别是不需要指定 mount 源，指明 mount point 就行了。

## 共享数据

- docker cp
- 将数据放到 bind mount 中
- volume container
	- docker create， 因为 volume container 的作用只是提供数据，它本身不需要处于运行状态。
	- 其他容器可以通过 --volumes-from 使用 volume container
	- data-packed volume container
