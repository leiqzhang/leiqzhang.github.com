# OpenStack Operations Guide Notes


## CH1 Provisioning and Deployment

主要关注的是OS infrastructure的自动化部署和配置

### 自动化部署(Automated Deployment)

所谓自动化部署，是指在少量的人工操作（如服务器上架、mac/ip规划、开机等）后，不再需要人工介入，自动化对操作系统进行安装和配置。

对于操作系统的自动化部署来说，一般会首先通过PXE/TFTP的网络启动，然后通过kickstart/preseed等自动配置技术或者直接使用镜像部署的方式来进行。

在进行自动化部署之前，必须对一些关键点进行提前规划和考虑。

* 磁盘分区和RAID：生产环境，一定要考虑RAID
* 网络配置：保证可PXE启动，不使用bond，不VLAN

### 自动化配置

Puppet

### 远程管理

Out-of-band access

IPMI


## CH2 Cloud Controller Design

假设： 单独的Controller节点

### Hardware Considerations

Controller上的一些服务是可以虚拟化部署的，如MQ

*TODO* 还有什么？

假设：所有服务都非虚拟化部署

*问题* 根据可预见的规模来决定是否虚拟化部署, 后续扩容怎么办？

考虑如下几点：

* 同时运行的虚拟机数量：数据库大小；数据库水平扩展（虚拟机状态上报、调度）
* 同时运行的计算节点数量：消息队列
* 访问API的用户数量：CPU
* 访问Dashboard的用户数量：CPU
* 同时运行nova-api服务的数量：size the controller with a core per service
* 一个虚拟机运行的时间
* 是否使用外部权限系统:网络；CPU

### Separation of Services

* glance-* 运行于swfit-proxy 服务器
* 独立的数据库服务器
* 每个服务均虚拟化部署
* 使用LB

nova-compute、swfit-proxy和swift-object不建议虚拟化部署

### Database

重要依赖。建议cluster database来具有容错能力

### MQ

重要依赖。建议Cluster MQ

### API

是否要提供EC2兼容API？ 

可提供多个API server （LB）

### Extensions

### Scheduler

可提供多个scheduler （不依赖LB）

### Images

* glance-api
* glance-registry: 维护镜像元数据信息，依赖数据库

Backend的选择

### Dashboard

httpd

### Authentication and Authorization

Keystone; Policy

Identity Service Backends:

* In-memory kv存储
* SQL数据库
* PAM
* LDAP

### Network Considerations

保证带宽

尤其是Imaging Service在controller时

10GB NIC / bond 2 10GB NICs

## Scaling

### The Starting Point

估算cloud的核心数量和规格：虚拟机最大数量 (超配比例 x cores/virtual cores per instance)，所需ephemeral存储容量(flavor disk size * number instance)

还需要考虑oS本身服务所占用资源

还需考虑实际的业务模型，对于实际触发的操作类型的影响，进而对controller等的load

一般来说，8U8G 可制成一个机架的计算节点

### Adding Controller Nodes

对于只依赖于内部MQ的，可以直接添加服务，如nova-scheduler等等

对于API服务，需要依赖LB （software：Pound or HAProxy)

对于DashBoard，需要考虑websocket和session的共享

对于nova-api, glance-api等，本身可通过配置来使用多进程模式（*TODO*）

DB/MQ的LB等，需要单独考虑

### Segregating Your Cloud

cells，regions， zones，host aggregates

#### Cells and Regions

#### AZ and Host Aggregates

### Scalable Hardware

#### Hardware Procurement

CPU同型号，支持迁移

#### Capacity Planning

Monitor resource usage

#### Burn-in Testing

Hardware failure


## CH4 Compute Nodes

### CPU Choice

VT / Cores

### Hypervisor Choice

### Instance Storage Solutions

* Off compute node storage -- shared file system
* On compute node storage -- shared file system
* On compute node storage -- non-shared file system

Live Migration (non-share/share)

Shared Filesystems:
* NFS
* GlusterFS
* MooseFS
* Lustre

### Overcommitting

默认超配比例： CPU 16， RAM 1.5

### Logging

Condiser： 中心logging server； 解析和分析系统 (logstash)

### Networking

see CH6

## CH5 Storage Decisions

Persistent Storage

### Storage Concepts

* Object Storage
* Block Storage
	* 直接访问存储块设备
	* 基于文件
* File-level Storage： 没有直接呈现，但ephemeral实际就是

### Chossing Storage Back-ends

List of Questions

...

各种开源及以cinder-driver提供的商业存储后端的对比

* Swift
* Ceph： cow(fast boot-from-volume); 统一对象和块存储；支持keystone
* Gluster： UFO （分布式文件系统+对象存储）
* LVM：不支持副本
* Sheepdog

### Notes on OpenStack Object Storage

## Network Design

#### Management Network

* 避免虚拟机流量影响系统管理和监控
* 可考虑对openstack各组件内部通信配置单独的网络（VLAN）
	
#### Public Addressing Options

* Fixed/Float IP
	
#### IP Address Planning

可分解为一些列的sections:

* subnet router
* control services public interfaces
* object storage cluster internal comm
* compute and storage commu
* out-of-band remote mgt
* in-band remote mgt
* spare space for future growth

### Network Topology

OS 提供了不同的NetworkManager，会对网络拓扑有不同的需求：

* Flat
* FlatDHCP
* VlanManager
* FlatDHCP Multi-host HA

(*TODO*) 是不是因为版本原因，未列出GRE Tunel

VLAN/Multi-NIC (*TODO*)/Mutli-host and Single-host Networking

### Services for Networking

NTP/DNS(dnsmasq)

## CH7 Example Architecture

选型考虑

选择的各个组件及其原因

* ...
* RabbitMQ: only option which supports features such as Compute Cells ? (*TODO*) => trunk上的op文档也是这么说
