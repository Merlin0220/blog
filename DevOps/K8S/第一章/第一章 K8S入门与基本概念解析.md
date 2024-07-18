# 为什么需要k8s
最开始，使用的部署方式是手动部署。但是该部署方式存在着人工时间长、环境不隔离、进程之间产生资源争抢等问题。
![[Pasted image 20240714234258.png]]
后来，使用了虚拟化部署，把不同服务部署到了不同虚拟机中，虽然解决了环境不隔离问题，但是**占用的资源过多**。
![[Pasted image 20240714233925.png]]
之后，随着Docker的流行，容器化部署开始普及。容器化部署直接使用宿主机的操作系统，占用资源较少。
![[Pasted image 20240714234241.png]]
同时，k8s是对容器化部署做管理，提供了以下特性：
* 自我修复：在容器出现错误时（如代码编写不当等），如果重启可以解决问题（比如内存溢出），k8s提供了容器自动重启的功能。
* 弹性伸缩：字面意思，自动提高/减少服务数。
* 自动部署和回滚
* 服务发现和负载均衡：k8s自带了服务发现和负载均衡的方案。
* 机密和配置管理：用于配置动态化（类似配置中心）。
* 存储编排：提供磁盘映射功能。
* 批处理：批量的处理方案。

# k8s集群架构及组件
## 整体架构图
![[Pasted image 20240715232933.png]]

## 相关组件
### 控制面板组件（Master节点）
* kube-apiserver：接口服务，基于REST风格开放k8s接口的服务。、
* kube-controller-manager：负责运行控制器进程，包括：
	* 节点控制器（Node Controller）：负责在节点出现故障时进行通知和相应。
	* 任务控制器（Job Controller）：监测代表一次性任务的Job对象
	* 一级端点分片控制器、服务账号控制器等等
* cloud-controller-manager：云控制器管理器，主要管理第三方云平台提供的控制器。
* kube-scheduler：调度器，负责将Pod基于一定算法，将其调用到更合适的节点上。
* etcd：k8s数据库（键值型分布式数据库），基于Raft算法实现了自主的集群高可用。旧版本基于内存，新版本持久化存储。
![[Pasted image 20240715234910.png]]
### 节点组件
* kubelet：负责Pod的生命周期、存储等。
* kube-proxy：网络代理、负责Service的服务发现、负载均衡（4层）。
* container-runtime：容器运行时环境，如docker、containerd、CRI-O等、
![[Pasted image 20240715235535.png]]
### 附加组件
kube-dns：为整个集群提供DNS服务。
ingress-controller：为服务提供外网入口。
Prometheus：监控。
Federation：提供跨可用区的集群。

## 分层架构
![[Pasted image 20240716000000.png]]
* 生态系统：基于K8S创建出的其他应用服务。
* 接口层：K8S的接口。
* 管理层：系统度量、自动化以及策略管理等。
* 应用层：部署和路由。
* 核心层：对外提供API构建高层的应用，对内提供插件式应用执行环境。

# K8S资源和对象
资源可以理解为Java的类，对象就是从类创建出来的对象。
K8S中的所有内容都被抽象为“资源”，如 Pod、Service、Node等都是资源。对象就是资源的实例，是持久化的实体，如某个具体的Pod。
k8s有多种类别的资源，kubectl可以通过配置文件来创建对象，配置文件可以理解为描述对象属性的文件，常用YAML格式。
## 规约和状态
* 规约（Spec）：规约描述了对象的期望状态——希望对象所具有的特征。当创建k8s对象时，必须提供对象的规约以及一些基本信息（如名称等）。
* 状态（Status）：对象的实际状态，由k8s自己维护。k8s会通过一系列的控制器对对象进行管理，尽可能的让状态与规约重合。
## 资源的分类
具体可以分为：元数据、集群和命名空间三种级别，其中元空间在集群之上，集群内的所有资源都可以访问。
![[Pasted image 20240716225245.png]]
（图里的元空间指的就是元数据）
### 元数据型
* HPA：Horizontal Pod Autoscaler，负责Pod的弹性伸缩。可以根据CPU使用率或自定义指标（metrics）自动对Pod进行缩容扩容。
* Pod Template：关于Pod的定义，被包含在其他的k8s对象中（如Deployment、StatefulSet、DaemonSet等控制器），控制器通过 Pod Template 信息来创建Pod。
* LimitRange：用于限制Pod对资源的使用，如内存等等。
### 集群型
* Namespace：集群下的命名空间。
* Node：节点资源
* ClusterRole：即节点上的角色组，用于权限的控制。
* ClusterRoleBinding：将角色组和用户进行联系，用于权限的控制。
### 命名空间型（重点）
#### Pod
##### 基本概念
Pod可以理解为“容器组”，是k8s中最小的可部署单元。一个Pod内的容器共享存储资源和一个唯一的网络IP地址，以及一些确定容器该如何运行的选项。
Pod大部分情况下只运行一个容器，即“one-container-per-pod”的方式，此时可以将Pod视为运行容器的wrapper，k8s通过Pod管理容器，而不是直接管理容器。
如果多个容器是紧耦合的（需要资源共享等），那么也可以将多个容器编排在一起运行在同一个Pod中。
Pod底层存在一个pause容器，其他容器之前共享的资源实际上都是通过请求pause容器实现的：
![[Pasted image 20240716233821.png]]
![[Pasted image 20240716233910.png]]
##### 副本数
副本（repicas）指的是一个Pod可以被复制成多份，每一份都可以被称之为一个“副本”，这些副本除了一些描述的信息（Pod Name, uid等）不一样外，其他信息相同，提供一样的功能。
副本数包含在Pod的“控制器”（Deployment）中，当集群中该Pod数量和副本数指定的不一致时，k8s会采取一些策略去满足副本数配置的要求。
##### 控制器
K8S是容器资源管理和调度平台，容器跑在Pod里，Pod是K8S里最小的单元。所以，这些Pod作为一个个单元我们肯定需要去**操作它的状态和生命周期**。那么如何操作？这里就需要用到控制器了。
控制器分为四种：无状态服务控制器、有状态服务控制器、守护进程控制器以及任务控制器。
###### 无状态服务（Nginx等）控制器
* RC：ReplicationController，帮助我们动态更新Pod副本数。与Pod深度绑定，目前已经废弃。
* RS：ReplicaSet，与RC做的事情相同，但可以通过**Selector**选择对哪些Pod生效。但只支持扩容和缩容功能。
* Deployment：最常用的无状态服务控制器，对RS做了进一步封装。Deployment提供以下功能：
	* 创建 ReplicaSet / Pod：Deployment底层对Pod的扩容缩容还是通过RS实现的，因此Deployment提供了自动创建ReplicaSet和Pod的功能。
	* 滚动升级/回滚：类似于灰度发布，当Pod进行升级后，Deployment会自动将Pod及其副本一个个进行升级，每次升级会自动新建一个Pod和对应的RS，同时原先的Pod和RS不会被删除，用于回滚。
	* 平滑扩容和缩容。
	* 暂停与回复：当一段时间内有多次升级时，我们不希望每次修改Deployment都自动进行滚动升级，因此可以暂停Deployment的功能，并在修改好之后进行恢复。
![[Pasted image 20240717224406.png]]
###### 有状态服务（Redis等）控制器
StatefulSet专门用于有状态服务，在k8s v1.5版本以上支持。该控制器提供以下功能：
* 稳定的持久化存储，不会因Pod的删除而删除。
* 稳定的网络标志。
* 有序部署，有序扩展：即Pod是有顺序的（从0到N-1），在部署或者扩展时，之前的Pod必须都是Running或Ready状态，基于init contrainers实现。
* 有序收缩，有序删除。
StatefulSet由两部分组成：Headless Service（DNS服务）和VolumeClaimTemplate（存储卷服务），分别对应了网络管理和存储管理。其中Headless Service需要在StatefulSet之前创建好。
StatefulSet中每个Pod的DNS格式为{statefulSetName}-{0...N-1}.{serviceName}.{namespace}.svc.cluster.local
* serviceName为Headless Service的名字。
* 0...N-1为Pod的序号。
* namespace为服务所在的namespace名，Headless Service和StatefulSet必须在相同的namespace
![[Pasted image 20240717233107.png]]
###### 守护进程控制器
为匹配上的Node都生成一个守护进程，常用来部署一些集群的日志、监控等应用。
![[Pasted image 20240718001309.png]]
###### 任务控制器
Job：一次性任务，运行完成后Pod销毁，不再重新启动新容器。
CronJob：在Job基础上加上了定时功能，由linux crontab实现。

#### 服务发现
k8s对于服务发现共有两种方案：Ingress和Service，分别解决了南北流量和东西流量的问题，具体架构如下所示：
![[Pasted image 20240718233938.png]]
举例：如下图所示由product服务和order服务，分别创建对应的Service，其中product服务的Service监听真实的28088端口并将请求转发到Pod的8088端口（即商品服务的虚拟端口），order服务的Service监听28099端口并将请求转发到Pod的8099端口。
如果product服务和order服务想要互相访问，直接使用serviceName作为域名，监听的端口作为端口发送请求即可，比如product服务请求order服务对应地址为 order-svc:28099：
![[Pasted image 20240718233855.png]]

#### 存储与配置
* Volume：即数据卷，用于共享Pod中容器使用的数据，用于数据的持久化，比如数据库数据。
* CSI：Container Storage Interface，是k8s、mesos、docker等联合制定的一个行业标准接口规范，旨在将任意存储系统暴露给容器化应用程序（便于第三方插件集成）。CSI声明了Volume Plugin必须实现的接口。

#### 特殊类型配置
* ConfigMap：ConfigMap 是一种 API 对象，用来将非机密性的数据保存到键值对中。使用时，Pod 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。ConfigMap可以简单理解为一个配置中心。ConfigMap 并不提供保密或者加密功能。
* Secret：作用与ConfigMap相同，它解决了密码、token、密钥等敏感数据的配置问题。使用 Secret 意味着你不需要在应用程序代码中包含机密数据。
* DownwardAPI：用于让Pod里运行的容器可以直接获取到这个Pod对象本身的一些信息，它提供了两种方式：
	* 环境变量：用于单个变量，可以将Pod信息和容器信息直接注入容器内部。
	* Volume挂载：将Pod信息生成为文件，直接挂载到容器内部去。

#### 权限相关
* Role：一组权限的集合，例如包含列出Pod权限及列出Deployment权限，与ClusterRole不同的是，Role仅用于给某个Namespcae中的资源进行鉴权（粒度为命名空间级，比ClusterRole小）。
* RoleBinding：作用与ClusterRoleBinding相同，只不过作用在命名空间上。