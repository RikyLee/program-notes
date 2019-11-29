# Kubernetes

## Kubernetes基本概念

### Pod

Pod是一组紧密关联的容器集合，它们共享PID、IPC、Network和UTS namespace，是Kubernetes调度的基本单位。Pod的设计理念是支持多个容器在一个Pod中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。

### Node

Node是Pod真正运行的主机，可以物理机，也可以是虚拟机。为了管理Pod，每个Node节点上至少要运行container runtime（比如docker或者rkt）、kubelet和kube-proxy服务。

### Namespace

Namespace是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。常见的pods, services, replication controllers和deployments等都是属于某一个namespace的（默认是default），而node, persistentVolumes等则不属于任何namespace。

### Service

Service是应用服务的抽象，通过labels为应用提供负载均衡和服务发现。匹配labels的Pod IP和端口列表组成endpoints，由kube-proxy负责将服务IP负载均衡到这些endpoints上。

每个Service都会自动分配一个cluster IP（仅在集群内部可访问的虚拟地址）和DNS名，其他容器可以通过该地址或DNS来访问服务，而不需要了解后端容器的运行。

### Label

Label是识别Kubernetes对象的标签，以key/value的方式附加到对象上（key最长不能超过63字节，value可以为空，也可以是不超过253字节的字符串）。

Label不提供唯一性，并且实际上经常是很多对象（如Pods）都使用相同的label来标志具体的应用。

Label定义好后其他对象可以使用Label Selector来选择一组相同label的对象（比如ReplicaSet和Service用label来选择一组Pod）。Label Selector支持以下几种方式：

- 等式，如app=nginx和env!=production
- 集合，如env in (production, qa)
- 多个label（它们之间是AND关系），如app=nginx,env=test

### Annotations

Annotations是key/value形式附加于对象的注解。不同于Labels用于标志和选择对象，Annotations则是用来记录一些附加信息，用来辅助应用部署、安全策略以及调度策略等。比如deployment使用annotations来记录rolling update的状态。

## Kubernetes架构

Kubernetes 借鉴了 Borg 的设计理念，比如 Pod、Service、Labels 和单 Pod 单 IP 等。Kubernetes 的整体架构跟 Borg 非常像，如下图所示:
![architecture](https://kubernetes.feisky.xyz/zh/architecture/images/architecture.png)

Kubernetes主要由以下几个核心组件组成：

- etcd保存了整个集群的状态
- apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制
- controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等
- scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上
- kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理
- Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）
- kube-proxy负责为Service提供cluster内部的服务发现和负载均衡

![components](https://kubernetes.feisky.xyz/zh/components/images/components.png)

除了核心组件，还有一些推荐的Add-ons:

- kube-dns负责为整个集群提供DNS服务
- Ingress Controller为服务提供外网入口
- Heapster提供资源监控
- Dashboard提供GUI
- Federation提供跨可用区的集群
- Fluentd-elasticsearch提供集群日志采集、存储与查询

## 组件通信

Kubernetes 多组件之间的通信原理为

- apiserver 负责 etcd 存储的所有操作，且只有 apiserver 才直接操作 etcd 集群
- apiserver 对内（集群中的其他组件）和对外（用户）提供统一的 REST API，其他组件均通过apiserver 进行通信
  - controller manager、scheduler、kube-proxy 和 kubelet 等均通过 apiserver watch API 监测资源变化情况，并对资源作相应的操作
  - 所有需要更新资源状态的操作均通过 apiserver 的 REST API 进行
- apiserver 也会直接调用 kubelet API（如 logs, exec, attach 等），默认不校验 kubelet证书，但可以通过 --kubelet-certificate-authority 开启（而 GKE 通过 SSH 隧道保护它们之- 间的通信）

比如典型的创建 Pod 的流程为:

![workflow](https://kubernetes.feisky.xyz/zh/components/images/workflow.png)

1. 用户通过 REST API 创建一个 Pod
2. apiserver 将其写入 etcd
3. scheduluer 检测到未绑定 Node 的 Pod，开始调度并更新 Pod 的 Node 绑定
4. kubelet 检测到有新的 Pod 调度过来，通过 container runtime 运行该 Pod
5. kubelet 通过 container runtime 取到 Pod 状态，并更新到 apiserver 中

## 端口号

![components](https://kubernetes.feisky.xyz/zh/components/images/ports.png)

### Master node(s)

| Protocol | Direction | Port Range | Purpose                          |
| -------- | --------- | ---------- | -------------------------------- |
| TCP      | Inbound   | 6443*      | Kubernetes API server            |
| TCP      | Inbound   | 8080       | Kubernetes API insecure server   |
| TCP      | Inbound   | 2379-2380  | etcd server client API           |
| TCP      | Inbound   | 10250      | Kubelet API                      |
| TCP      | Inbound   | 10251      | kube-scheduler healthz           |
| TCP      | Inbound   | 10252      | kube-controller-manager healthz  |
| TCP      | Inbound   | 10253      | cloud-controller-manager healthz |
| TCP      | Inbound   | 10255      | Read-only Kubelet API            |
| TCP      | Inbound   | 10256      | kube-proxy healthz               |

### Worker node(s)

| Protocol | Direction | Port Range  | Purpose               |
| -------- | --------- | ----------- | --------------------- |
| TCP      | Inbound   | 4194        | Kubelet cAdvisor      |
| TCP      | Inbound   | 10248       | Kubelet healthz       |
| TCP      | Inbound   | 10249       | kube-proxy metrics    |
| TCP      | Inbound   | 10250       | Kubelet API           |
| TCP      | Inbound   | 10255       | Read-only Kubelet API |
| TCP      | Inbound   | 10256       | kube-proxy healthz    |
| TCP      | Inbound   | 30000-32767 | NodePort Services**   |

## 分层架构

Kubernetes设计理念和功能其实就是一个类似Linux的分层架构，如下图所示:
![分层架构](https://kubernetes.feisky.xyz/zh/architecture/images/14937095836427.jpg)

- 核心层：Kubernetes最核心的功能，对外提供API构建高层的应用，对内提供插件式应用执行环境
- 应用层：部署（无状态应用、有状态应用、批处理任务、集群应用等）和路由（服务发现、DNS解析等）
- 管理层：系统度量（如基础设施、容器和网络的度量），自动化（如自动扩展、动态Provision等）以及策- 略管理（RBAC、Quota、PSP、NetworkPolicy等）
- 接口层：kubectl命令行工具、客户端SDK以及集群联邦
- 生态系统：在接口层之上的庞大容器集群管理调度的生态系统，可以划分为两个范畴:
  - Kubernetes外部：日志、监控、配置管理、CI、CD、Workflow、FaaS、OTS应用、ChatOps等
  - Kubernetes内部：CRI、CNI、CVI、镜像仓库、Cloud Provider、集群自身的配置和管理等