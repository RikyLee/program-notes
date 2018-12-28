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
