VMWare vSphere 5.1 Clustering Deepdive - HA


## HA概念

HA概念

HA和MSCS的区别
[ ]MSCS要求APP有特定设计
[ ]MSCS复杂
[ ]HA的APP Monitor依赖VMWare Tools


## HA特点

几大优势，需要结合这几点来比对版本

5.0 
FDM
不依赖DNS
支持管理网分区
强化分区验证：避免管理网故障时的误报
datastore心跳
强化Admission Control Polices -- HA增强
5.1 
Admission Control Policies： slot size
支持retrieve a list of the vms span multiple slots
强化对PDL的处理
支持Guest OS sleep mode
在Tools SDK中引入APP监控SDK
FDM VIB自动加入Auto-Deploy image profile
支持通过配置delay isolationresponse

## HA架构

HA涉及的组件和架构图

### FDM

职责：
1. 向集群中其它host传播如下信息
host resource info
vm state
HA properties
2. 处理心跳机制
3. vm placement
4. vm restarts
5. logging

FDM自身的可靠性：
1. watchdog
2. resilient to network interruptions and "all paths down" (APD) conditions
3. 在网络故障时，Inter-host commu 自动使用其它的通信路径（支持冗余路径）

去除对DNS的依赖

fdm.log

### HOSTD Agent

职责： many of the tasks we take for granted like powering n vm

FDM直接和HOSTD和vCenter通信，尅是的HA更快的响应power-on request，降低VM的启动时间

FDM依赖HOSTD获取host上注册的vm信息，管理VM也是通过HOSTD APIs。

如果HOSTD失效，则FDM也会失效

### vCenter

职责：
1. 部署和配置HA Agents（支持Parallel部署）
2. 通知集群配置变更(到选举出的master），变更包括调整高级设置、添加新host等等
3. protection of vms and unprotection. (比如vm开关机、ESX 断连--unprotect）

在HA处理故障时，vCenter并不参与，但HA依赖vCenter来配置或监控集群

vCenter故障时，cluster的配置变更不可用。

vCenter包含的信息：
1. 保护的vm列表
2. 集群配置
3. vm-to-host兼容性信息
4. host membership

vCenter及其依赖的服务，在HA时应该具有高优先级

## 基本概念

Master/Slave agents
除了网络分区外，cluster中只有一个master HA agent
master负责监控其负责的VM的健康状态，并且在其失败时进行重启
slave负责转发信息到master，以及在master的指导下重启VM
Heartbeating
Isolated vs Network partitioned
VM Protection

### Master Agent

master 宣称对VM的职责，是通过 taking "ownership" of　放置vm配置文件的datastore的方式来做的。 （疑问：cluster重叠的情况？） 
=> 建议不要存在重叠的情况
锁定的文件，是放置在各集群各自的文件夹下的

Election

master由一系列的HA Agents来进行选举，只要这些agents无能和master互通，具体情形包括：
1. 初始配置
2. master所在主机
故障
网络分区或隔离
从vCenter断开
进入维护或standby莫斯
HA被重配置

选举大概15s，UDP

选举过程中不能处理故障，这些故障需要在选举完毕后，由master进行处理

master：连接datastore数目最多且具有最大的Managed Object Id

slaves： 会通过管理平面，和master建立安全、加密的TCP连接(SSL)，在有master的情况下，slaves相互之间不会通信，除非重新进行选举

master会试图获取datastore的ownership：
自己可以直接访问到的
可以通过slave中转请求访问到的
自己后续发现的
周期性重试之前没有获取成功的
方式： lock protectedlist 文件

/<root of datastore>/.vSphere-HA/<cluster-specific-dir>/protectedlist

protectedlist文件内容：被HA保护的VM列表，以及vm cpu reservation and memory overhead

master会在集群中VM所使用的所有datastores中分布此文件

master异常场景
1. master fail：lock会expire，新master会relock
2. master isolation: master会release lock (问题：如何知道自己isolation了）
3. fail while isolation：restart of vms will be delayed until a new master has been elected
slave通过心跳判断master是否异常（点对点心跳） (问题，如果master到datastore断了？）

master也会负责监控slave状态且向vCenter上报slave host的状态，如果slave故障了，master会决定哪些VM需要重启，且决定其placement

### Slaves

职责： 监控host上运行的VM状态，并通知master状态变化；监控到master的心跳；到master不可达后，发起并参与选举；发送心跳到master（点对点）

Files for both slave and master

A communication mechanism

Remote Files： poweron file => 1. tracking vm power-on state 2. slaves inform the master if it is isolated from mgt network

Local Files: clusterconfig; vmmetadata/compatlist; fdm.cfg; hostlist(IP,hostname,mac) 

### Heartbeating

Network Heartbeating
slave => master
master => all slaves
Point to Point
every one second

slave无法收到master的心跳时，会试图判断自己是否为isolated (如何判断？）

Datastore Heartbeating
only used in case the master has lost network connectivity with the slaves
used to validate whether a host has failed or is merely isolated/network partitioned 
Isolation: through "poweron" file

host failed (no datastore hb) => restart the failed host's vms
host isolated/np => only take action when it is appropriate to. (只有vm被down or power down/shutdown by a triggered isolation resp) (后续章节内容，TODO）

默认HA选择两个hb datastore， 选择尽可能多主机均可访问的datastore，可通过配置调整hb的数量。在NFS和VMFS中优选VMFS datastore。也可以自行选择。

How works?
VMFS: heartbeat region (只要有打开的文件，则会自动更新心跳。HA会在datastore上创建特定的per-host file来用于打开）
NFS： 各主机每隔5s更新自己的hb文件
如果之前用于hb的datastore不可用或者被移除了，HA会检测到并重新选择新的datastore用于心跳

Isolated versus Partitioned
Administrator's perspective:
partitioned: 俩主机可以操作但是通过管理网络和对方不通
Isolated： 主机不能在管理网络上observe任何HA管理traffic，而且不能ping通配置的isolation address
FDM perspective当FDM无法和master通信，就会选举新的master如果网络分区了，a master election will occur so that a host failure or network isolation with this partition will result in appropriate action on the impacted vms每一个网络分区中，均会有自己的master，分区恢复后，任何一个master都可以作为新的master一个master可以负责另一个网络分区的VM，VM状态的监控通过datastore communication mechanism
在HA架构中，一个主机是否被分区了， is determinated by the master reporting the condition。比如分区发生后，各自分区的master均会上报vCenter当前哪些主机被分区了，vCenter会进行展示。master仅根据管理网不通可以确定host是partition或isolate，但不能区分。一个host只有它自己通过datastore通知master自己隔离了，master才可以上报此主机是isolated （问题：host和datastore断连了咋办？）
俩视图可能会有不一致的情况，比如host到存储也断了，管理网和master也不通了，则从HA看，此host是故障了，但从Administrato看此host是隔离了。


[[TODO: Table]]

HA基于host的state来触发相应的action。
  Host Failed： will initiate restart of vms
  Host Isolated: might initiate restart
    A virtual machine will only be shutdown or powered off when the isolated host knows there is a master out there that has taken ownership for the vm or when the isolated host loses access to the home datastore of the vm
    在主机对于其上面VM所在的datastore无Access能力时（APD)，HA会触发isolation response来保证原VM被关机并安全重启，避免脑裂 （问题：HA咋触发？）（问题：Isolation Response）


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
