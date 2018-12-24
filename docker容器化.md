主机型虚拟化：
	1、虚拟机管理器直接安装在硬件平台上（esxi）
	2、宿主机安装有自己的操作系统，在宿主机操作系统上在安装虚拟机管理软件（vmware workstation）

容器级虚拟化

	chroot

	linux 的内核namespaces原生支持

		namespace              系统调用参数             隔离内容                    内核版本
		UTS                    CLONE_NEWUTS             主机名和域名                2.6.19
		Mount                  CLONE_NEWNS              挂载点（文件系统）          2.4.19
		IPC                    CLONE_NEWIPC             信号量、消息队列、共享内存  2.6.19
		PID                    CLONE_NEWPID             进程编号                    2.6.24
		User                   CLONE_NEWUSER            用户和用户组                3.8   
		NetWork                CLONE_NEWNET             网络设备、网络栈、端口等    2.6.29

	Control Groups （CGroups）控制资源分派
	
    LXC linux container
	
	Docker  lxc的二次封装发行版
	
容器编排

machine+swarm+compose
mesos+marathon
kubernetes->k8s
