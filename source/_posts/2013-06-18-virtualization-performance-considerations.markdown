---
comments: true
date: 2013-06-18 20:40:31
layout: post
slug: virtualization-perf
title: 'Virtualization performance considerations'
categories:
- Virtualization
- Performance
tags: [NUMA, DBPM, ELI]

---

今天看了云计算大会2013上EMC/Vmware的一片关于虚拟化性能的文章，结合之前看过的一些材料对其进行了汇总和总结。主要站在全云计算堆栈的整体视角上看各个组件或特性对性能的影响。

需要注意的是性能会包括好多指标，这些指标往往是互相矛盾的，如实时性（低延迟）和CPU利用率、电源利用率等等。本文也会结合之前vmware的文档归纳下如何尽可能地提高虚拟化场景下的实时性。


## CPU & Memory

1. NUMA

	__Respect boundaries, rightsize, use like topologies.__

	Hypervisor的调度器需要尽可能将同一个VM的所有vCPU调度到同一个NUMA节点上，同时此VM相关的内存需要分配到此节点的本地物理内存中。

	Vmware ESX 5.0及之后的版本支持一种叫做__vNUMA__的特性，它将Host的NUMA特征暴露给了Guest OS，从而使得Guest OS可以根据NUMA特征进行更高性能的调度。

2. CPU相关的一些指标和新特性

	* Processor Clock Frequency
	* Dual or Quad Socket
	* Hyper-threading (=Enabled)
	* Support __EPT__(Intel) or __RVI__(AMD), eg. Hwmmu
	
3. BIOS相关

	主要是指BIOS中对电源管理方式的设置。
	
	<!--more-->
	
	目前主流的BIOS电源管理支持__DBPM__(demand-based power management), 主要是指BIOS可以选择暴露部分或全部的P State到OS，由OS来根据负载情况管理电源的使用。如Dell 11G服务器支持如下几种电源管理方式：
	1. Static Max Performance：完全禁用掉DBPM
	2. OS Control：完全由OS控制
	3. Active Power Controller: Dellsystem DBPM, OS部分控制
	4. Custom
	
	对于虚拟化场景下的性能来说，如果Hypersior有能力，则最好是将电源管理设置为OS Control，交由Hypersior来管理。当然，对于实时性要求很高的应用，则令当别论，会在后面单独一节详述。
	
4. Memory

	内存方面主要需要关注的是内存的实际速度，同时需要记住如下一句话：
	
	__Memory is Cheap Performance__


## Network


网络方面当前的了解程度还不够，这里只是汇总下Vmware的观点

1. Use Distributed Virtual Switches
2. Route based on phy NIC
3. NetIOC
4. Gb vs 10Gb
5. Jumbo Frames for 10Gb (especially NFS, iSCSI)


## Storage

存储方面主要需要考虑如下几方面

1. The Protocol War

	即使用NFS、iSCSI还是FC
	
2. 存储使用方式

	RDM（Passthru)或者集群文件系统(VMFS等)上得VirtualDisk(VMDK, VHD, etc)
	
3. 存储类型： 是否瘦分配

4. Array Integration & Configuration

	即Vmware一直推崇的VAAI，即存储卸载。当前在VAAI的基础上进一步演进，已经退出vVol、vFlash、vSSD等概念，并正在进行实现。
	
## Guest

主要是应用层面的考虑了，和非虚拟化场景下的一般考虑因素基本一样。

1. Right sizing

2. Use a Modern OS

3. Install VMware Tools (vEthernet Adapters/vSCSI Controller)

	对于其余的Hypervisor，主要是指需要在Guest中安装相应的半虚驱动(ParDriver),如Xen下的PVDriver、KVM下的Virtio等等

4. VM Disk Layout

5. Limit Snapshots Use

	无论是COW快照还是ROW快照，均会影响到VM的IO性能。
 
6. Disk Alignment

## 实时性考虑

这里讨论的场景是需要尽可能降低IO延迟，而可以容忍高的CPU利用率和电源损耗。

基本上需要考虑的方面就是本文上面几节中列出的各个因素，并加以权衡。

1. BIOS

    当启用__DBPM__时，不可避免的CPU需要参与到电源管理中去，必然会增加IO处理的延迟，故可以考虑将电源管理设置为“Static High”模式。

    对于更新一些的CPU(Intel Nehalem class), CPU还提供了C-states和更高级的C1E两种电源管理选项，为了追求低延迟，此俩选项也许禁用。

2. NUMA

    参考上文中的描述，主要依赖于Hypervisor的NUMA感知的调度能力，并提供的类似于__vNUMA__的能力。

3. Guest

    首先是使用更新版本的Guest OS。

4. 物理网卡

    当前的NIC一般均会默认启用一些尽可能减少对CPU中断次数的技术，以避免CPU被频繁中断，这些技术即__interrupt moderation__或者__interrupt throttling__，它们会对部分中断进行合并后再中断CPU。

    如果考虑低延迟，则需要通过ethtool等工具将上面的特性关闭掉。当然关闭此特性会影响到LRO(Large Receive Offloads)特性。

5. 虚拟网卡

    VMware是建议使用其最新版本的VMXNET3.

6. VM配置

    如果应用是计算密集型的则给VM配置更多地vCPU，如果是存储密集型的，给VM配置更多的内存。当然需要避免vCPU和内存的overcommit。

    如果Hypersior支持，可设置不允许Hypersior在VM的vcpu空闲时将其调度出去。

7. 其它

    考虑开发支持Poll机制的设备驱动，对于IO的执行不依赖于设备返回中断而是直接通过poll方式等待IO的完成。因为设备中断返回到VM中，会引发VM的几次退出。

    PS，关于如何避免中断引起VM的退出，QEMU社区已经有了方案（ELI/PV-EOI），可参考：
    * 原始论文：http://www.mulix.org/pubs/eli/eli.pdf
    * http://blog.csdn.net/luo_brian/article/details/8686717
    * http://blog.csdn.net/luo_brian/article/details/8744025

    再PS，在下一代的Intel Haswell处理器中，已经通过硬件实现了所谓的vAPIC，可有效避免中断引起的VM退出。故QEMU社区的软件方案未能进入到Kernel中，期待一下Intel的新处理器中的此新特性吧。


## References

1. https://www.emcworldonline.com/2013/connect/sessionDetail.ww?SESSION_ID=1496
2. http://en.community.dell.com/techcenter/power-cooling/w/wiki/best-practices-in-power-management.aspx
3. http://www.vmware.com/files/pdf/techpaper/VMW-Tuning-Latency-Sensitive-Workloads.pdf
