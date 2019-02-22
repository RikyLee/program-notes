## 主机型虚拟化：

- 虚拟机管理器直接安装在硬件平台上（esxi）
- 宿主机安装有自己的操作系统，在宿主机操作系统上在安装虚拟机管理软件（vmware workstation）

## 容器级虚拟化

- chroot
- linux 的内核namespaces原生支持

| namespace |            系统调用参数|             隔离内容 |                   内核版本| 
| :-------- :           | :-----:       | :----: | :----: |
|UTS                    |CLONE_NEWUTS   |          主机名和域名        |        2.6.19|
|Mount                  |CLONE_NEWNS    |         挂载点（文件系统）     |     2.4.19|
|IPC                    |CLONE_NEWIPC   |         信号量、消息队列、共享内存  ||2.6.19|
|PID                    |CLONE_NEWPID   |         进程编号                |    2.6.24|
|User                   |CLONE_NEWUSER  |        用户和用户组            |    3.8   |
|NetWork                |CLONE_NEWNET   |         网络设备、网络栈、端口等  |  2.6.29|

- Control Groups （CGroups）控制资源分派
	
## 容器编排

- machine+swarm+compose
- mesos+marathon
- kubernetes->k8s

## docker

- docker daemon
- docker client
- docker registry 提供用户访问鉴权、镜像存储、镜像索引
	registry 每个镜像的一系列版本存储在一个repo中，通过repo+tag唯一区分一个镜像


- 依赖环境
64 bits CPU
linux Kernel 3.10+
linux cgroups and namespaces

可选yum源
CentOS 7 extras repo  版本比较旧
Docker-CE      所有版本，需安装yum源

- docker 程序环境
	环境配置文件：
		/etc/sysconfig/docker-network
		/etc/sysconfig/docker-storage
		/etc/sysconfig/docker
	Unit File:
		/usr/lib/systemd/system/docker.service
	Docker Registry配置文件：
		/etc/containers/registries.conf
	
	docker-ce
		配置文件：/ect/docker/daemon.json
	
- Docker镜像加速
	docker cn
	阿里云加速器
	中国科技大学
	｛
		"registry-mirrors":[""]
	｝

- 进入容器
docker exec -it container-name /bin/sh

docker image 含有启动容器所需的文件系统及其内容，因此，其用于创建并启动docker容器
	采用分层构建机制，最底层为bootfs，其次为rootfs
		bootfs：用于系统引导的文件系统，包括bootloader和kernel，容器启动之后会被卸载以节约内存资源
		rootfs：位于bootfs之上，表现为docker容器的根文件系统，docker中rootfs被内核挂载为只读，然后通过联合挂载的形式挂载一个可写层

Aufs Advanced multi-layered unification filesystem 高级多层统一文件系统
- 用于Linux文件系统实现联合挂载
- aufs是之前UnionFS的重新实现，2006年优Junjiro Okajima开发
- Docker最早使用aufs作为容器文件系统层，目前仍作为存储后端之一来支持
- aufs的竞争产品是overlayfs，自3.18版本之后合并到linux的内核
- docker 的分层镜像，除了aufs，docker还支持btrfs，devicemapper和vfs
	- 在Ubuntu系统，docker默认使用aufs，而在centos 7上，用的是devicemapper
	


Docker Registry分类

- Registry用于保存docker镜像，包括镜像的层次结构和元数据
- 用户可以自己Registry，也可以使用官方的Docker Hub
- 分类
	- Sponsor Registry 第三方的registry，供客户和Docker社区使用
	- Mirror Registry 第三方的registry，只供客户使用
	- Vendor Registry 由发布Docker镜像的供应商提供的registry
	- Private Registry 通过设有防火墙和额外的安全层的私有实体通过的registry
	
Registry （repository and index） index维护用户信息、镜像索引信息等


镜像的导入导出

docker save -o filename  IMAGE [IMAGE]

docker load -i filename

docker容器虚拟网络

OVS (Open VSwitch) SDN (Software Defined Network)

Overlay Network 叠加网络 隧道转发 报文封装

docker 默认bridge 是一个nat桥  安装bridge-utils 使用brctl 查看

docker 四种网络模型
- Closed container 只有Loopback interface 不能实现网络通讯
- Bridged container 桥接网络
- Joined container 联盟式网络，多个容器隔离User、Pid、Mount，但是共享UTC、IPC、NET
- Open container  开放式网络，直接使用物理机网卡

/ect/docker/daemon.json

bip: 修改桥接网络默认的IP地址
hosts:["tcp://0.0.0.0:2375","unix:///var/run/docker.sock"] 设置之后远程可操作本机docker


打开宿主机上的核心网络转发，可以实现本机docker上的不同bridge的通信


docker 容器存储卷

写时复制（COW）：修改一个已经存在的文件，文件会被从只读层复制到读写层，只读文件仍然存在，被读写层的文件副本隐藏

关闭并重启容器，数据不受影响；但删除Docker容器，则其更改全部丢失。
存在问题：
	- 存储于联合文件系统中，不易于宿主机访问
	- 容器间数据共享不便
	- 容器删除其数据会丢失

解决方案: volume(卷)
	- 卷是容器上的一个活多个“目录”，此类目录可绕过联合文件系统，与宿主机上的某目录“绑定（关联）”

Docker两种类型的卷，每种类型都在容器中存在一个挂载点，但其在宿主机上的位置有所不同
- Bind-mount Volume: a volume that points a user-specified location on the host file system

- Docker-managed volume: the Docker daemon creates managed volumes in a portion of the host's file system that's owned by Docker 存放路径为`/var/lib/docker/volumes/${id}/${volume path}`

在容器中使用Volumes

在docker run 命令使用-v选项即可使用Volume
- Docker-managed Volume `docker run -it --name bbox1 -v /data busybox`,查看bbox1容器的卷、卷标识符及挂载的主机目录 `docker inspect -f {{.Mounts}} bbox1`
- Bind-mount Volume `docker run -it -v HOSTERDIR:VOLUMEDIR --name bbox2 busybox` 查看bbox1容器的卷、卷标识符及挂载的主机目录 `docker inspect -f {{.Mounts}} bbox2`

共享Volumes
- 多个容器使用同一个主机目录
	```
	docker run -it --name c1 -v /docker/volumes/v1:/data busybox
	docker run -it --name c2 -v /docker/volumes/v1:/data busybox
	```
- 复制使用其他容器的卷,在docker run命令使用 --volumes-from container
	```
	docker run -it --name bbox1 -v /docker/volumes/v1:/data busybox
	docker run -it --name bbox2 --volumes-from bbox1 busybox
	```

镜像的制作
- 基于容器
- Dockerfile

Dockerfile Format

- Format
 - #Comment #号开头表示注释
 - INSTRUCTION arguments  指令不区分大小写 第一个指令必须`FROM` 基础镜像

 .dockeringore 每行表示一个文件，可以通配符，包含的文件会被忽略，不会打入镜像中

 Dockerfile instructions  dockerfile 指令

 - FROM指令是最重的一个且必须为Dockerfile文件开篇的第一个非注释行，用于为映像文件构建过程指定基准镜像，后续的指令运行于此基准镜像所提供的运行环境，默认情况下，docker build会在docker主机上查找指定的镜像文件，在其不存在是，则会从Docker Hub Registry上拉取所需的镜像文件，如果找不到指定的镜像文件，docker build会返回一个错误信息

	```
	FROM <repository>[:<tag>]
	FROM <repository>@<digest>

	<repositort>: 指定作为base image的名称
	<tag>: base image的标签，为可选项，省略时默认为
	```

- MAINTAINER(depreacted) 用于让Dockerfile制作者提供本人的详细信息。不限制出现的位置，但推荐放在FROM指令之后

```
MAINTAINER <author's detail> 约定使用作者的名称及邮件地址

MAINTAINER "RikyLi <rikylee719@gmail.com>"
```

- LABEL添加镜像的元数据,可以包含多个label，也可以一行写多个label
```
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```

- COPY 用于从Docker主机复制文件到创建的新映像文件,建议dest使用绝对路径，否则COPY指定则以WORKDIR为其起始路径
```
COPY <src>... <dest>
COPY ["<src>"... "<dest>"]

<src> 必须是build上下文中的路径，不能使其父目录中的文件
<src> 是目录，则其内部文件或子目录会被递归复制，但<src>目录本身不会被复制
指定了多个<src>,或者使用了通配符，则<dest>必须是一个目录，且必须以/结尾
<dest>不存在，会被自动创建，包括其父目录路径
```
- ADD 类似COPY指令，ADD支持使用TAR文件和URL路径
```
ADD <src>... <dest>
ADD ["<src>",..."<dest>"]

<src>为URL路径且<dest>不以/结尾，则<src>指定文件将被下载并直接创建为<dest>,<dest>以/结尾，泽文件名URL指定的文件将被直接下载并保存为<dest>/<filename>
如果<src>是本地文件系统上的tar文件，则将被展开为一个目录，类似于“tar -x”，然而通过URL获取的tar文件将不会被自动展开
<src>有多个，或使用了通配符，则<dest>必须是一个以/结尾的目录；如果<dest>不以/结尾，这其被视为一个普通文件，<src>的内容将被直接写入到<dest>
```

- WORKDIR 为Dockerfile中所有的RUN，CMD，ENTRYPOINT，COPY，ADD指定工作目录,可以输指定多次，影响之后的指令

```
WORKDIR /usr/local/

ADD jdk ./java/
等价于  ADD jdk /usr/local/java/
```

- VOLUME 用于在image中创建一个挂载点，以挂载Docker host上的卷或其他容器上的卷

```
VOLUME <mountpoint>
VOLUME [" <mountpoint>"]

如果挂载点目录路径下此前有文件存在，docker run命令会在卷挂载完成后将此前的所有文件复制到新挂载的卷中
```

- EXPOSE 用于为容器打开指定要监听的端口以实现与外部通信
```
EXPOSE <port>[/<protocol>] <port>[/<protocol>] <port>[/<protocol>]

默认为TCP，可指定tcp/udp二者之一
```

- ENV 为镜像定义所系的环境变量，可以被Dockerfile中位于其后的指令所调用 格式为$variable_name或${variable_name}

```
ENV <key> <value>

ENV <key>=<value> ... 如果value中包含空格可以用\转义，也可以加引号标识，另外反斜线也可用于续行
```

- RUN 用于指定docker build过程中运行的程勋，可以是任何命令
```
RUN <command> 启动的进程为shell的子进程，默认使用 '/bin/sh -c' 启动
RUN ["<exectable>","<param1>","<param2>"]

```

- CMD 为容器指定默认要运行的程序，其运行结束之后，容器也将终止，可以被docker run 的命令行选项覆盖，可以存在多个CMD命令，但是只有最后一个生效

```
CMD <command> 启动的进程为shell的子进程，默认使用 '/bin/sh -c' 启动
CMD ["<exectable>","<param1>","<param2>"] 启动的进程的进程号为1，系统创建，不是shell子进程

CMD ["<param1>","<param2>"]  用于为ENTRYPOINT指令提供默认参数
```

- ENTRYPOINT 类似CMD指令的功能，用于容器指定默认运行的程序，从而使得容器像是一个单独的可执行程序，ENTRYPOINT启动的程序不会被docker run命令行指定的参数覆盖，而且这些命令行参数会被当作参数传递给ENRYPOINT指定的程序，在docker run 使用--entrypoint选修的参数可覆盖ENRYPOINT指定的程序

```
ENTRYPOINT <command>
ENTRYPOINT ["<exectable>","<param1>","<param2>"]

```

