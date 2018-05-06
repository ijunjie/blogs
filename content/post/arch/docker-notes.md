
# Docker 学习笔记整理

本篇是 [每天5分钟玩转 Docker 容器技术](http://www.cnblogs.com/CloudMan6/) 的学习笔记。

## 共用 Host 的 Kernel

所有容器共用 Host 的 kernel. 如果容器对 kernel 版本有要求，则不适合使用 Docker。

## 容器的 Copy-On-Write 特性

容器启动时，一个可写层被加载到镜像的顶部。所有对容器的改动，只会发生在容器层。

**只有容器层是可写的，容器层下面的所有镜像层都是只读的。**

镜像层数量可能会很多，所有镜像层会联合在一起组成一个统一的文件系统。如果不同层中有一个相同路径的文件，比如 /a，上层的 /a 会覆盖下层的 /a，也就是说用户只能访问到上层中的文件 /a。在容器层中，用户看到的是一个叠加之后的文件系统。

- 添加文件：在容器中创建文件时，新文件被添加到容器层中。
- 读取文件：在容器中读取某个文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后打开并读入内存。
- 修改文件：在容器中修改已存在的文件时，Docker 会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后修改之。
- 删除文件：在容器中删除文件时，Docker 也是从上往下依次在镜像层中查找此文件。找到后，会在容器层中记录下此删除操作。

只有当需要修改时才复制一份数据，这种特性被称作 Copy-on-Write。可见，容器层保存的是镜像变化的部分，不会对镜像本身进行任何修改。

容器层记录对镜像的修改，所有镜像层都是只读的，不会被容器修改，所以镜像可以被多个容器共享。

## 镜像构建方式

两种构建镜像的方式：

- docker commit
	```
	docker commit <current_container_name> <new_image_name>
	```
- dockerfile. 底层实际也是通过 docker commit 一层一层构建新镜像的。

## 镜像构建历史

`docker history <image_name>` 会显示镜像的构建历史，也就是 Dockerfile 的执行过程。

## Dockerfile 调试

如果第三层镜像构建失败，可以手动启动一个第二层的容器查看失败原因。

```
docker run -it <intermediate_container_id>
```

## Dockerfile 指令

- WORKDIR
	**指定容器内工作目录，如果 WORKDIR 不存在，Docker 会自动创建。**
	如 `WORKDIR /app` 则后面指令涉及容器内路径时，相对于`/app`
- ENV
	指定环境变量，后面指令可以引用。
	如 `ENV MYSQL_HOST "10.153.46.32"`,后续指令以 `$MYSQL_HOST`获取。

## 指令格式：Shell & Exec

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

**推荐用法：** ENTRYPOINT 使用 Exec 格式。如上面例子中改为 `ENTRYPOINT ["/bin/sh", "-c", "echo Hello, $name"]`. 

## ENTRYPOINT & CMD

建议 CMD 只用于为 ENTRYPOINT 提供默认参数。

```
ENTRYPOINT ["/bin/echo", "Hello"]  
CMD ["world"]
```

此时，可以在启动容器时传入参数，覆盖 CMD 提供的内容。

`docker run -it [image]` 输出 `Hello world`;
`docker run -it [image] Foo Bar Haha` 输出 `Hello Foo Bar Haha`;

**`ENTRYPOINT command param1 param2` 会忽略任何 CMD 或 docker run 提供的参数。**

## 镜像的分发

- 提供 Dockerfile 给他人自行构建
- 上传到 registry 供别人下载

## docker attach & docker exec

attach 直接进入容器 启动命令 的终端，不会启动新的进程

exec 则是在容器中打开新的终端，并且可以启动新的进程。

`docker exec -it <container> bash|sh` 是执行 exec 最常用的方式。

`docker logs -f` 也比较常用。


## 服务类容器和工具类容器

按用途容器大致可分为两类：服务类容器和工具类的容器。


1. 服务类容器以 daemon 的形式运行，对外提供服务。比如 web server，数据库等。通过 -d 以后台方式启动这类容器是非常合适的。如果要排查问题，可以通过 exec -it 进入容器。

2. 工具类容器通常给能我们提供一个临时的工作环境，通常以 run -it 方式运行。


## 重命名容器可执行docker rename

## 容器启动和停止

- docker stop
- docker kill
- docker start, 保留容器的第一次启动时的所有参数
- docker restart, 相当于依次执行 docker stop 和docker start.

可以先创建容器，稍后再启动。

- docker create 创建的容器处于 Created 状态。
- docker start 将以后台方式启动容器。 docker run 命令实际上是 docker create 和 docker start 的组合。

## 重启策略

启动容器时，可以设置重启策略。`--restart=always` 意味着无论容器因何种原因退出（包括正常退出），就立即重启。该参数的形式还可以是 `--restart=on-failure:3`，意思是如果启动进程退出代码非0，则重启容器，最多重启3次。

## 暂停容器

有时我们只是希望暂时让容器暂停工作一段时间，比如要对容器的文件系统打个快照，或者 dcoker host 需要使用 CPU，这时可以执行 `docker pause`. 处于暂停状态的容器不会占用 CPU 资源，直到通过 docker unpause 恢复运行。

## 删除容器

docker rm 一次可以指定多个容器，如果希望批量删除所有已经退出的容器，可以执行如下命令：

```
docker rm -v $(docker ps -aq -f status=exited)
```

## 限额

- 内存限额，默认为 -1， 指定了 --memory 则 --memory-swap 默认为 momory 的两倍
- CPU 限额，默认为 1024
- Block IO 限额，默认为 500


## 容器内存限额

避免某个容器因占用太多资源而影响其他容器乃至整个 host 的性能。

容器可使用的内存包括两部分：物理内存和 swap。 Docker 通过下面两组参数来控制容器内存的使用量。

- `-m` 或 `--memory`：设置内存的使用限额，例如 100M, 2G。

- `--memory-swap`：设置 内存+swap 的使用限额。

如 

```
docker run -m 200M --memory-swap=300M ubuntu
```
（注意以 --memory-swap=300M 的 =）其含义是允许该容器最多使用 200M 的内存和 100M 的 swap。默认情况下，上面两组参数为 -1，即对容器内存和 swap 的使用没有限制。

如果在启动容器时只指定 -m 而不指定 --memory-swap，那么 **--memory-swap 默认为 -m 的两倍**，比如：

```
docker run -it -m 200M ubuntu
```

容器最多使用 200M 物理内存和 200M swap。

## progrium/stress 镜像

该镜像可用于对容器执行压力测试。执行如下命令：

```
docker run -it -m 200M --memory-swap=300M progrium/stress --vm 1 --vm-bytes 280M
```

--vm 1：启动 1 个内存工作线程。

--vm-bytes 280M：每个线程分配 280M 内存。

因为 280M 在可分配的范围（300M）内，所以工作线程能够正常工作，其过程是：

1. 分配 280M 内存。

2. 释放 280M 内存。

3. 再分配 280M 内存。

4. 再释放 280M 内存。

一直循环......

### 容器 CPU 限额 

**默认设置下，所有容器可以平等地使用 host CPU 资源并且没有限制。**

Docker 可以通过 -c 或 --cpu-shares 设置容器使用 CPU 的权重。如果不指定，**默认值为 1024**。

与内存限额不同，通过 -c 设置的 cpu share 并不是 CPU 资源的绝对数量，而是一个相对的权重值。某个容器最终能分配到的 CPU 资源取决于它的 cpu share 占所有容器 cpu share 总和的比例。

换句话说：通过 cpu share 可以设置容器使用 CPU 的优先级。

比如在 host 中启动了两个容器：

```
docker run --name "container_A" -c 1024 ubuntu

docker run --name "container_B" -c 512 ubuntu
```

container_A 的 cpu share 1024，是 container_B 的两倍。当两个容器都需要 CPU 资源时，container_A 可以得到的 CPU 是 container_B 的两倍。

需要特别注意的是，这种按权重分配 CPU 只会发生在 CPU 资源紧张的情况下。如果 container_A 处于空闲状态，这时，为了充分利用 CPU 资源，container_B 也可以分配到全部可用的 CPU。

**实验**：

1. 启动 container_A, cpu_share 为 1024
	
	```
	docker run --name container_A -it -c 1024 progrium/stress --cpu 1
	```
	--cpu 用来设置工作线程的数量。因为当前 host 只有 1 颗 CPU，所以一个工作线程就能将 CPU 压满。
2. 启动 container_B, cpu_share 为 512：
	```
	docker run --name container_B -it -c 512 progrium/stress --cpu 1
	```
3. 在 host 中执行 top, 可以看到两个容器对 CPU 的使用情况。可以看到分别使用了 66.3% 和 33.3% 左右。
4. 此时暂停 container_A, 可以观察到 container_B 在 container_A 空闲的情况下能够用满整颗 CPU.


## 容器 Block IO 限额

默认情况下，所有容器能平等地读写磁盘，可以通过设置 --blkio-weight 参数来改变容器 block IO 的优先级。--blkio-weight 设置的是相对权重值，**默认为 500**.

在下面的例子中，container_A 读写磁盘的带宽是 container_B 的两倍。

```
docker run -it --name container_A --blkio-weight 600 ubuntu   

docker run -it --name container_B --blkio-weight 300 ubuntu
```

## 限制 bps 和 iops

bps 是 byte per second，每秒读写的数据量。iops 是 io per second，每秒 IO 的次数。

可通过以下参数控制容器的 bps 和 iops：
- --device-read-bps，限制读某个设备的 bps。
- --device-write-bps，限制写某个设备的 bps。
- --device-read-iops，限制读某个设备的 iops。
- --device-write-iops，限制写某个设备的 iops。


## cgroup

cgroup 实现资源限额， namespace 实现资源隔离。

cgroup 全称 Control Group。Linux 操作系统通过 cgroup 可以设置进程使用 CPU、内存 和 IO 资源的限额。--cpu-shares、-m、--device-write-bps 实际上就是在配置 cgroup.

在 /sys/fs/cgroup/cpu/docker/<container_id>/ 下的文件中记录了容器资源限额对应到 cgroup 的配置。

## namespace

在每个容器中，我们都可以看到文件系统，网卡等资源，这些资源看上去是容器自己的。拿网卡来说，每个容器都会认为自己有一块独立的网卡，即使 host 上只有一块物理网卡。这种方式非常好，它使得容器更像一个独立的计算机。

Linux 实现这种方式的技术是 namespace. namespace 管理着 host 中全局唯一的资源，并可以让每个容器都觉得只有自己在使用它。换句话说，namespace 实现了容器间资源的隔离。

**Linux 使用了六种 namespace，分别对应六种资源：Mount、UTS、IPC、PID、Network 和 User**.

## Mount namespace

Mount namespace 让容器看上去拥有整个文件系统。

容器有自己的 / 目录，可以执行 mount 和 umount 命令。当然我们知道这些操作只在当前容器中生效，不会影响到 host 和其他容器。

## UTS namespace

简单的说，UTS namespace 让容器有自己的 hostname。 **默认情况下，容器的 hostname 是它的短ID**，可以通过 -h 或 --hostname 参数设置。

## IPC namesapce

IPC namespace 让容器拥有自己的共享内存和信号量（semaphore）来实现进程间通信，而不会与 host 和其他容器的 IPC 混在一起。

## PID namespace

容器在 host 中以进程的形式运行。例如当前 host 中运行了的容器，可以使用 `ps axf` 查看容器进程。所有容器的进程都挂在 dockerd 进程下。容器中进程的 PID 不同于 host 中对应进程的 PID，容器中 PID=1 的进程当然也不是 host 的 init 进程。也就是说：容器拥有自己独立的一套 PID，这就是 PID namespace 提供的功能。

## Network namespace

Network namespace 让容器拥有自己独立的网卡、IP、路由等资源。我们会在后面网络章节详细讨论。

## User namespace

User namespace 让容器能够管理自己的用户，host 不能看到容器中创建的用户。

## 单一 Host 下 Docker 的网络