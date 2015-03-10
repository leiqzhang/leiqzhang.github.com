# XEN Hypervisor信息查看方法介绍

声明：

本博客欢迎转发，但请保留原作者信息!

博客地址：http://openstack.wiaapp.cn

博客地址：http://www.leiqzhang.com

新浪微博：[@leivli](http://www.weibo.com/leivli "leiqzhang")

内容系本人学习、研究和总结，如有雷同，实属荣幸！

## Hypervisor信息查看的必要性

在基于XEN 的虚拟化环境中，Hypervisor 直接运行于硬件之上，所有虚拟机（包括Dom0 和DomU）均作为Domain 运行于Hypervisor 之上。各Domain 的VCPU、Memory 等计算资源的分配、各Domain之间的内存共享和信息交互、主机硬件的识别、管理和在各Domain之间的共享使用均需Hypervisor 的参与。

XEN 提供了一些工具和启动选项的支持，以支持对Hypervisor 中与上述信息相关的数据的显示、收集和分析。这些数据对于理解XEN 虚拟化原理、虚拟化半虚驱动的性能调优等均有很大的指导意义。比如可以使用xenstrace工具在一段时间内，抓取Domain通过Hypercall 或 VMExit 陷入到Hypervisor 的记录，分析虚拟化的实现机制，并可对通过xenalyze工具对这些记录进行汇总分析。

## 常用的信息查看方法

### 虚拟机内部确定其使用的虚拟化方式

在虚拟机内部可以通过dmesg 命令查看启动日志中与xen 相关的信息，可以初步判断出虚拟机所使用的虚拟化方式，比如是PVHVM还是PVH等。关于PVHVM、PV、PVH等虚拟化方式的区别可以参考[1]。

```
#在DomU内部运行
dmesg | grep -i xen 
```
如果是HVM，则在输出中会有类似如下的信息：
```
Hypervisor detected: Xen HVM
```
如果是PVHVM，则在输出中会有类似如下的信息，即内核启动过程中可以通过pvops机制检测到其处于虚拟化环境中，并自动使用pv timer
```
Xen: using vcpuop timer interface
```
Brendan Gregg 提供的Xen Feature Detection 的工具，即是根据这些信息来在虚拟机内部判断虚拟机所使用的虚拟化类型的([2])。

### 确定硬件辅助虚拟化特性的支持情况

Intel/AMD提供了一系列的硬件辅助虚拟化特性，如VT-x、VT-d等等，其中一些特性是需要在BIOS中手动进行开启的。在Dom0中可以通过如下命令查看XEN Hypervisor 的启动日志，确定这些特性是否被XEN 正常的启用起来。

比如查看VT-d 是否支持，可通过如下命令：
```
xl dmesg | grep -i "vt-d"
```
这里的xl 命令是XEN 在4.x 的某个版本后引入的，运行于Dom0 的管理工具，其代替了之前版本的XM 命令。

### 查看主机的NUMA 拓扑

在Dom0中可以通过如下两个命令均可以查看主机的NUMA拓扑
```
xl info -n
xenpm get-cpu-topology
```

### 查看/调整各虚拟机的vcpu 和pcpu 的亲和性关系

通过在Dom0上运行如下命令可以查看本主机上所有虚拟机的vcpu 列表，各vcpu 当前正在哪个pcpu 上运行，以及各vcpu 可以被调度到哪些pcpu 上运行（即亲和性）。
```
xl vcpu-list
```
通过在Dom0上运行如下命令可以调整各DomU的vcpu 与pcpu 的亲和性：
```
xl vcpu-pin {domain-id} {vcpu} {pcpu_list}
#比如: xl vcpu-pin 2 0 1-3
```
对于Dom0的vcpu 与 pcpu亲和性设置，一种方式是通过在xen 启动参数中增加选项来指定，此种方式下，Dom0启动后，不能再通过上述方式对其亲和性进行动态调整，另一种方式是在xen 启动参数中不指定如上选项，并在启动后通过上述方式进行调整：
```
#启动参数中指定，如下选项会使Dom0仅使用4个vcpu，且pin 到pcpu 0-3上
dom0_max_vcpus=4 dom0_vcpus_pin
``` 

### 查看Hypervisor 的内部数据结构

通过查看Hypervisor 的内部数据结构，可以获取各Domain 的内存在不同Numa Node 上的分配情况，各物理设备中断号与Domain 的eventchannel 的关联关系等等，是一个很强大的工具。

可以通过在Dom0中运行如下命令的组合，查看所支持查看的各种内部数据结构：
```
xl debug-keys h
xl dmesg
```

类似的输出信息如下：
```
(XEN) 'h' pressed -> showing installed handlers
(XEN)  key '%' (ascii '25') => Trap to xendbg
(XEN)  key '0' (ascii '30') => dump Dom0 registers
(XEN)  key 'C' (ascii '43') => trigger a crashdump
(XEN)  key 'H' (ascii '48') => dump heap info
(XEN)  key 'N' (ascii '4e') => NMI statistics
(XEN)  key 'Q' (ascii '51') => dump PCI devices
(XEN)  key 'R' (ascii '52') => reboot machine
(XEN)  key 'a' (ascii '61') => dump timer queues
(XEN)  key 'c' (ascii '63') => dump cx structures
(XEN)  key 'd' (ascii '64') => dump registers
(XEN)  key 'e' (ascii '65') => dump evtchn info
(XEN)  key 'h' (ascii '68') => show this message
(XEN)  key 'i' (ascii '69') => dump interrupt bindings
(XEN)  key 'm' (ascii '6d') => memory info
(XEN)  key 'n' (ascii '6e') => trigger an NMI
(XEN)  key 'q' (ascii '71') => dump domain (and guest debug) info
(XEN)  key 'r' (ascii '72') => dump run queues
(XEN)  key 't' (ascii '74') => display multi-cpu clock info
(XEN)  key 'u' (ascii '75') => dump numa info
(XEN)  key 'v' (ascii '76') => dump Intel's VMCS
(XEN)  key 'z' (ascii '7a') => print ioapic info
```

比如，如果要查看各Domain 的内存在不同Numa Node 上的分配情况，则可以通过如下命令组合查看：
```
xl debug-keys u
xl dmesg
```

如果Domain 数量比较多时，由于xl dmesg buf 的大小限制，通过上述方法可能看不到完整的信息输出，此时可以通过本文后面提到的方法配置串口，通过串口连接到Hypervisor（连按三次Ctrl+A），然后按上述debug-keys 对应的键，就可以直接通过串口读取到完整的信息。

### 查看虚拟机前后端半虚驱动的特性协商情况

对于使用半虚驱动的DomU，为支持前后端驱动的前向兼容，需要使用xenstore 来在前后端之间对支持的特性进行协商。在Dom0中通过如下命令查看xenstore 可以看到这些前后端驱动协商的结果，确定某些高级特性是否被正常启用。
```
xenstore-ls
```
如上命令的输出是一个树形结构，各Domain 均对应一个树节点，如果使用了半虚驱动，在此Domain 节点中会有vbd 和vif 两个子节点，分别对应半虚的blk 和net 驱动，这两个子节点下会包含前后端通过xenstore 进行协商和通信所交换的数据。

### 抓取虚拟机陷入到Hypervisor的所有记录

可以在虚拟机中运行要分析的应用，然后在Dom 0中运行如下命令抓取一段时间此主机上所有虚拟机陷入到Hypervisor 的记录：
```
xentrace -D -e all -T 15 /home/trace.log
# -T： 抓取的时间
# -e： 要抓取的pcpu mask，比如可以指定-e 0xf0，表示仅抓取发生在pcpu 5-8上的记录
```

如上命令的输出文件/home/trace.log 是一个二进制文件，可以通过xenalyze 工具对其进行后分析。xenalyze 工具常用的两种分析方式为根据trace.log分别获取summary 汇总信息或所有记录的dump 信息。

```
#获取summary 汇总信息
xenalyze --cpu-hz=2.7G --summary /home/trace.log > /home/trace.summary
#获取dump 信息
xenalyze --cpu-hz=2.7G -a /home/trace.log > /home/trace.dump

#如上的--cpu-hz是xenalyze用于计算各操作耗时的，需要根据/proc/cpuinfo中的值进行指定
```

在汇总信息中，会包含各Domain 处于各种状态的时间比例分布，以及此Domain各vcpu 处于各种状态的时间比例、主要退出原因的汇总统计等等;在dumap 信息中，会包含各Domain 所执行的所有hypercall 类型，所有VMExit的原因等信息，同时也会包含XEN Hypervisor 代码中包含的TRACE信息

### 通过串口连接Hypervisor

当XEN 或Dom0 内核启动失败时，以及通过debug-keys 查看Hypervisor 的内部数据结构时，通过串口连接Hypervisor 来获取信息是一种常用的方法。

一般来说，直接到机房连接到服务器的串口上进行查看是不方便的。目前各服务器往往会有带外的支持IPMI重定向的串口，可以通过配置可以通过另一台服务器通过IPMI连接到要调试的服务器的串口上，并获取到串口中的信息。

```
#被调试服务器，XEN增加启动参数：
loglvl=all guest_loglvl=all com1=38400,8n1 console=com1
## 即指定XEN 使用串口com1， 同时设置串口的速度为38400 b/s， 8n1的意思为配置串口为 8 databits, no parity and 1 stopbit

#被调试服务器，启动kernel的vmlinuz命令，增加参数：
console=hvc0 earlyprintk=xen 
## 这里指定hvc0 是指Xen console。这里执行console=hvc0 是让Dom0 内核的输出也通过Xen 打印到串口。故最终效果是，Dom0 内核和XEN Hypervisor 的输出均会通过com1 串口输出。 之后可以通过连按三次Ctrl+A来使得串口在Dom0 和Hypervisor 之间进行切换

#查看信息的服务器，配置IPMI重定向
ipmitool -I lanplus -H 192.100.99.67 -U root -P root sol activate
> [SOL Session operational.  Use ~? for help]
## ipmitool -I lanplus -H <目标节点带外IP> -U <带外用户名> -P <带外密码> sol activate
```

## 参考资料

1. [1] [XEN不同的虚拟化方式](http://www.brendangregg.com/blog/2014-05-07/what-color-is-your-xen.html "What Color Is Your Xen?")
1. [2] [Xen Feature Detection Tools](http://www.brendangregg.com/blog/2014-05-09/xen-feature-detection.html "Xen Feature Detection Tools")
1. [3] [xl tool manpage](http://manpages.ubuntu.com/manpages/raring/man1/xl.1.html "xl tool manual")
1. [4] [xen debugging tools: kdb/gdb/debug-keys/trace etc](http://zhigang.org/wiki/XenDebugging "Xen Debugging")
1. [5] [Xen Wiki: Xen Serial Console Config](http://wiki.xen.org/wiki/Xen_Serial_Console "Xen Serial Console")
