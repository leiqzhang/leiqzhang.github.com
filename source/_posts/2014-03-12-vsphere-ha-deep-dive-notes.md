# [Notes] VMWare vSphere 5.1 Clustering Deepdive - HA


## HA介绍

### HA

云计算或虚拟化环境中的高可用性（High Available/HA)的目标是针对运行在VM中应用的。典型的场景是将多个Host组成一个HA集群，当其中的VM所在的主机故障的时候，可以自动的在集群中其它Host上将VM重新拉起。

更进一步，除了VM所在主机故障的情形外，还可以在VM中Guest OS乃至特定应用的状态发生故障时，触发HA。当然，对于GuestOS相关的监控则会依赖于VMWare Tools。

### 和MSCS对比

MSCS集群方案的目标之一也是提供应用的高可用性，虚拟化场景下的HA和MSCS的区别主要有：

* MSCS只保证在一个节点故障的时候，将其上没得APP迁移到其它节点上，至于服务在此之后是否正常运转，则要求运行在其上面的APP有特定设计
* 相比HA,MSCS使用和配置起来更为复杂

### 5.x的HA新特点

__5.0版本__
 
* 新的HA Agent：FDM
* 不再依赖DNS
* 支持管理网分区的情况
* 引入Primary 和 Slave Node的概念
* 强化隔离验证：避免管理网完全故障时的误报
* 引入基于存储的datastore心跳
* 强化Admission Control Polices

__5.1版本__
 
* 进一步强化和增强Admission Control Policies
* 强化对PDL(Permanent Device Loss)的处理
* 支持Guest OS休眠模式
* 在VMWare Tools SDK中引入APP监控SDK
* FDM VIB自动加入Auto-Deploy image profile
* 支持通过配置延迟触发isolation response（具体含义在后文中解释）

## HA架构

[[TODO HA涉及的组件和架构图]]

其主要的组件包括：

* FDM
* HOSTD
* vCenter

### FDM

__职责__

1. 向集群中其它host传播信息：host resource info；vm state；HA properties
2. 处理心跳机制
3. vm placement
4. vm restarts
5. logging （单独的fdm.log日志）

__自身的可靠性__

1. watchdog
2. resilient to network interruptions and "all paths down" (APD) conditions
3. 在网络故障时，各主机间的通信可自动使用其它的通信路径（支持冗余路径）

### HOSTD Agent

__职责__

many of the tasks we take for granted like powering on vm

__和FDM的关系__

* FDM直接和HOSTD和vCenter通信，使得HA可更快的响应power-on request，降低VM的启动时间
* FDM依赖HOSTD获取host上注册的vm信息，管理VM也是通过HOSTD APIs
* 如果HOSTD失效，则FDM也会失效

### vCenter

__职责__

1. 部署和配置HA Agents（支持并发部署）
2. 通知集群配置变更到选举出的master，变更包括调整高级设置、添加新host等等
3. 启用和停止虚拟机的保护 (场景：比如vm开关机、master所在ESX断连等场景）

__和HA的关系__

需要注意的是，在HA处理故障时，并不需要vCenter的参与，但HA依赖vCenter来配置或监控集群。在vCenter故障时，HA Cluster的配置变更不可用。

HA依赖vCenter提供的信息包括：

1. 保护的vm列表
2. 集群配置
3. vm-to-host兼容性信息
4. host membership

vCenter及提供其依赖的服务的VM，需要在HA时具有高优先级来进行恢复

## 基本概念

HA涉及到的基本概念包括：

* Master/Slave agents
	* 除了网络分区的场景外，cluster中只有一个master HA agent
	* master负责监控其负责的VM的健康状态，并且在其失败时进行重启
	* slave负责转发信息到master，以及在master的指导下重启VM
* Heartbeating
* Isolated vs Network partitioned
* VM Protection

### Master Agent

__职责__

* 跟踪监控自己负责的VM，并在需要HA的条件下触发相应的行为。
* 负责监控slave状态且向vCenter上报slave host的状态，如果slave故障了，Master会决定哪些VM需要重启，且决定其placement

__Master的确定__

Master是由Cluster中的HA Agents来进行选举出来的，可触发选举的场景包括如下几种：

* 初始配置Cluster
* 当之前的Master所在主机
	* 故障
	* 网络分区或隔离
	* 从vCenter断开
	* 进入维护或standby模式
	* HA被重配置

满足如下条件的HA Agent会被选举为Master：

* 连接datastore数目最多
* 具有最大的Managed Object Id

选举通过UDP协议，大概时间为15s。选举过程中不能处理故障，这些故障需要在选举完毕后，由Master进行处理。

在选举出Master后，其余Agent就是Slave。各Slave会通过管理平面，和Master建立安全、加密的TCP连接(SSL)，在有Master的情况下，Slaves相互之间不会通信，除非重新进行选举。

__Master的工作原理__

Master will claim responsibility for a vm by taking "ownership" of the datastore on which the virtual machine's configuration file is stored。 

所谓获取"ownership"，是指在相应datastore中如下位置建立protectedlist文件，并进行lock:

```
/<root of datastore>/.vSphere-HA/<cluster-specific-dir>/protectedlist
```

其中"protectedlist"文件的内容包括：

* 被HA保护的VM列表
* VM的 cpu reservation and memory overhead

Master会在集群中VM所使用的各个datastores中放置此文件

另外，Master会试图获取满足如下任一条件的datastore的ownership：

* 自己可以直接访问到的
* 可以通过slave中转请求访问到的
* 自己后续发现的
* 对于之前没有获取成功的datastore会周期性进行重试

__Master异常__

如上节所述，Master会lock datastore，如果Master故障或者隔离的情况下，如何处理呢？ 可分为如下几种异常场景进行分析：

* master fail：lock会expire，新master会relock
* master isolation: master会release lock，至于如何判断自己处于隔离状态，可参考[[TODO]]）
* fail while isolation：restart of vms will be delayed until a new master has been elected

而Slave通过到Master点对点的心跳判断Master是否异常来决定是否重新选举新的Master。

### Slaves

__职责__

* 监控host上运行的VM状态，并通知master状态变化
* 监控到master的心跳
* 到master不可达后，发起并参与选举；
* 发送心跳到master

### Files for both slave and master

Master和Slave不仅使用文件来存储状态数据，也会使用文件（Remote File）作为一种通信机制。

__Remote Files__

放置在共享存储上的文件主要是poweron文件，其有俩作用：

* tracking vm power-on state
* Slaves会通过此文件来标记其是否处于和管理网隔离的状态，Master会通过此文件获知此信息

__Local Files__

Master/Slave均会放置一些文件在本地存储上，主要包括：

* clusterconfig
* vmmetadata/compatlist
* fdm.cfg(logging cfg)
* hostlist(IP,hostname,mac) 

### Heartbeating

在5.x的HA中有两种心跳：

* 网络心跳
* 存储心跳

__网络心跳__

网络心跳均为点到点的,非广播或组播，心跳周期为1秒，会在如下部件间存在：
* slave到master
* master到所有的slaves

__存储心跳__

存储心跳，也叫Datastore Heartbeating，仅会在master和Slave之间管理网不通时，才会用于进一步判断主机是否已经故障或者仅仅是隔离/分区。

默认HA会选择两个用作存储心跳的datastore，且会优先选择集群中能被尽可能多主机访问的datastore，另外相对NFS datastore，会优先选择VMFS datastore。

VMFS的datastore，存储心跳是通过VNFS的heartbeat region机制来实现的。只要一个Host打开了VMFS datastore上的文件，就会自动在heartbeat region中有心跳。HA会在VMFS datastore上创建特定的per-host file来由各个Host打开。

对于NFS的datastore，由个主机负责每隔5s中更新本主机对应的心跳文件。

如果之前用于hb的datastore不可用或者被移除了，HA会检测到并重新选择新的datastore用于心跳

 
Isolation: through "poweron" file

host failed (no datastore hb) => restart the failed host's vms
host isolated/np => only take action when it is appropriate to. (只有vm被down or power down/shutdown by a triggered isolation resp) (后续章节内容，TODO）


### Isolated versus Partitioned

[[TODO: 配图]]
[[TODO: isolation address]]

需要注意的是，从上层Admisitrator和从HA的FDM的视角来看，可能会看到不同的故障场景。

Administrator's perspective:
partitioned: 俩主机可以操作但是通过管理网络和对方不通
Isolated： 主机不能在管理网络上observe任何HA管理traffic，而且不能ping通配置的isolation address



FDM perspective当FDM无法和master通信，就会选举新的master如果网络分区了，a master election will occur so that a host failure or network isolation with this partition will result in appropriate action on the impacted vms每一个网络分区中，均会有自己的master，分区恢复后，任何一个master都可以作为新的master一个master可以负责另一个网络分区的VM，VM状态的监控通过datastore communication mechanism
在HA架构中，一个主机是否被分区了， is determinated by the master reporting the condition。比如分区发生后，各自分区的master均会上报vCenter当前哪些主机被分区了，vCenter会进行展示。master仅根据管理网不通可以确定host是partition或isolate，但不能区分。一个host只有它自己通过datastore通知master自己隔离了，master才可以上报此主机是isolated （问题：host和datastore断连了咋办？）
俩视图可能会有不一致的情况，比如host到存储也断了，管理网和master也不通了，则从HA看，此host是故障了，但从Administrato看此host是隔离了。

HA基于host的state来触发相应的action。
[ ]Host Failed： will initiate restart of vms
[ ]Host Isolated: might initiate restart
A virtual machine will only be shutdown or powered off when the isolated host knows there is a master out there that has taken ownership for the vm or when the isolated host loses access to the home datastore of the vm
[ ]在主机对于其上面VM所在的datastore无Access能力时（APD)，HA会触发isolation response来保证原VM被关机并安全重启，避免脑裂 （问题：HA咋触发？）（问题：Isolation Response）


### VM Protection

虚拟机状态变化后，vCenter会通知master来启用或禁用此VM的HA保护。 启用或禁用保护的操作，master需要保证将虚拟机状态更新到disk上后才可以返回成功


## CH4 Restarting VMs

HA响应VM故障的场景：
Failed Host
Isolated Host
Failed guest os

### Restart Priority and Order
Agent VM
FT secondary vm
priority of high
priority of medium
priority of low

### Restart Retries
可配置 （0 2 6 14 30m -> 5次重试）

### 场景：Failed Host

Failed Slave
T0-fail；T3s-master 监控datastore hb持续15s；T10s-host被宣布为不可达，master开始ping host的管理网，持续5s；T15s-如果host没有配置datastore hb，则host被宣布为dead；T18s-如果host配置了datastore hb，主机被宣布为dead

处理： (5.1) master会从protectedlist中，也可以向vCenter获取vm列表，进行重启。 后者是避免在网络分区情况下，其中一个master那分区中没有足够资源拉起vm，5.1允许多个master同时试图重启同一台VM
Failed master
T0-fail; T10s-Master election process initiated; T25s-新选举出的master读取protectedlist; T35s-新master对于protectedlist中未运行的vm发起重启


Isolation Resp and Detection

Isolation Resp: HA在主机lost its conn with the network and the remaining nodes in the cluster 后所会采取的行为。
Power Off 、 Leave Power on、Shut down
如果选择的是关机，但是否会触发，还依赖于host判断是否有own vm所在datastore的master可以restart此vm 
如果触发了resp，会首先将vm重poweron文件中移除，并在poweredoff文件夹下建立相应的vm file
选择关机的一种场景：vm可以访问vm网络但是不能访问它的存储，如果它维持开机，则会出现俩相同IP的vm（问题：为什么？）

Isolation DetectionIsolation Slave：
5.1：
期间有任何状态变化，均不会再触发isolation response。


如果Isolation Resp被触发了：

Isolation Master：


Restarting VM
Placement； DRS Relation

### Split-Brain

### Permanent Device Loss
