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
