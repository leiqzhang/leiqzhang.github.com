---
comments: true
date: 2013-08-07 22:40:31
layout: post
title: '虚拟磁盘的空间回收: Virtual Disk UNMAP/Shrink'
categories:  [Virtualization]
tags:  [unmap, trim, shrink, sparse, scsi]

---

## 问题

在虚拟化场景下，瘦分配(Thin-provisioning)磁盘应用场景非常广泛。目前主流的虚拟磁盘镜像格式，如Dynamic VHD、Sparse Raw、Qcow2、VMDK等均只具有随着虚拟机读写而动态增长的能力，一般来说是按需每次分配一个固定大小的块，如VHD的块是2M为单位。

当Guest OS删除了文件，已经分配的空间在虚拟磁盘上可否进行空间回收呢？

空间回收主要包括两步，一是获取到可以回收的空间，二是在虚拟磁盘文件中对相应空间进行回收。

根据获取所需回收空间时虚拟磁盘的IO情况来讲，回收主要有在线回收和离线回收，在线和离线的区别在于虚拟磁盘是否在有Guest持续写IO的情况下进行空间回收。也即离线回收不单单指虚拟机关机情形下的回收，也包括虚拟磁盘只有读IO的情况（如虚拟磁盘为某个ROW快照的父镜像）。

就虚拟磁盘文件来说，空间回收的结果可以是如下两种

1. 在Host上实际占用的空间减少
2. 在Host上实际占用的空间减少，但是后续Guest的写入可以复用之前需要回收的空间，从而使得虚拟磁盘文件不会随着Guest的写入立即分配新块

下文首先分别对离线和在线可回收空间获取方法进行讨论，然后对虚拟磁盘文件的空间回收方式进行讨论，最后以Qemu当前的实现为例进行说明。

<!-- more -->


## 离线可回收空间获取方法

离线情况下获取可回收空间，需要首先挂载虚拟磁盘到Dom0，然后进行如下几种方式之一的操作：

1. 读取虚拟磁盘上Guest文件系统信息，找到已经被删除的文件，在已删除文件区域填零作为标记
2. 直接在虚拟磁盘中生成一个能生成的极限大小的全零文件

第一种需要感知具体的文件系统格式，如对于NTFS来说可以利用开源的ntfs-3g库，但总归是需要较为复杂的程序逻辑；第二种实现起来简单，但是此方法在标记完成后会将虚拟磁盘文件撑到其规格容量大小。


## 在线可回收空间获取方法 -- 全零气球文件

在线情况下，不能使用离线情况下的第一种方案，因为此方案实际是分为两步，第一步是向OS查询已删除空间，第二步是填零，这两步并非原子操作，由于虚拟磁盘有持续些IO，可能在第一步后OS又将此已删除空间分配了出去，那么接下来第二步的填零操作就会破坏文件系统的数据，当此时OS分配的空间是用于文件系统元数据的话，可能还会造成文件系统的损坏。

所以在线情况下只能使用类似于离线第二种方案的方案，即利用OS的API在Guest内部生成极限大小的全零气球文件，以此文件所占的空间来标记可回收空间。对于NTFS来说，Mircosoft的SDelete工具可以做到这一点(http://www.microsoft.com/technet/sysinternals/Security/SDelete.mspx).

当然此方法也会导致虚拟磁盘文件首先撑到规格大小，且在后续回收时需要VM一定时间的暂停。


## 在线可回收空间获取方法 -- 感知文件删除


在虚拟化场景下，如需Virtual Device Driver感知到文件的删除，只能两方面努力：

1. 使得Guest OS的文件系统在文件删除时可以下发通知到Driver层
2. 在Guest OS中的驱动或半虚驱动（PVDriver、VirtioDriver、VMware Tools）中对文件系统的通知进行处理，并进行相应地处理

### 文件系统下发删除通知

当前NTFS已经支持在删除文件时下发TRIM指令到Driver, 可以通过“fsutil behavior query DisableDeleteNotify“命令查询是否开启了TRIM；

EXT4在mount时可以指定-o discard选项，则删除文件后，EXT4会自动下发discard指令到Driver（ioctl BLKDISCARD）；EXT3等文件系统可以手动执行fstrim来使得文件系统对于已经删除的空间下发Discard。

### Driver层的处理

当前ATA-8 ACS-2协议以及SCSI协议均已经对存储设备的空间回收提供了相应的协议支持，在ATA中此命令称为TRIM，在SCSI中此命令称为UNMAP。

提到TRIM，可能更多的会联想到SSD，其实当前IDE也已经支持此命令了，SCSI的UNMAP一般为Thinly-provisioned storage提供支持，所有走SCSI协议的设备理论上均可以接受UNMAP命令。

当虚拟机内部的虚拟设备是支持UNMAP/TRIM的半虚设备时，如virtio-scsi，需要半虚驱动在接收到文件系统的删除通知后将其转换为相应协议中的UNMAP或TRIM命令；如果非半虚驱动，如IDE，则Guest OS中自带或者安装的原生驱动会负责处理文件系统删除命令到协议命令的转换；如果虚拟机内部的半虚设备或者非半虚设备所采用的协议不支持空间回收，如virtio-blk、xen的pvblk等等，是不能支持空间回收的。

### 结论

结合上述的协议和文件系统支持，在虚拟化场景下，可以使得Guest OS使用如上开启TRIM的NTFS、开启DISCARD的EXT4或支持fstrim的文件系统，然后模拟IDE、SSD或者SCSI设备到Guest中，并对半虚设备使用支持命令转换的半虚驱动，是实现空间回收的必要条件。


## 虚拟磁盘文件空间回收方法

对于支持稀疏文件空间回收的系统来说，如Linux 2.6.38 + Ext4/ocfs2等，可以通过fallocate系统调用的FALLOC_FL_PUNCH_HOLE flag来对文件内部指定区域进行punch hole，进而达到空间回收的目的。

对于Qcow2等虚拟镜像文件，可以对回收块对应的L Table和Refcount Table进行标记，从而使得此空间后续可进行复用。

## QEMU下在线空间回收的实现

这里得在线空间回收是指感知文件删除方式。

条件：

1. 启动时对于相应的drive指定discard=on选项，如-drive file=/path/to/file.qcow2,format=qcow2,if=none,cache=none,discard=on
2. 使用Default PC类型的Machine，即指定-m PC选项。对于-m pc-0.15来说，模拟的ide设备不支持trim指令
3. 对于Windows 7 + NTFS来说，可使用IDE、AHCI等driver
4. 对于Linux + EXT4来说，可使用IDE、AHCI、virtio-scsi等driver

实现：

QEMU的block层接受到Guest OS通过Driver转发下来的TRIM或UNMAP命令后，最终会执行到相应的虚拟磁盘文件对应的driver中，对于qcow2来说，执行的操作就是标记Table，对于Raw来说，执行的操作是fallocate，对于设备来说，执行的操作是ioctl(BLKDISCARD)

Windows限制：

当前Windows下virtio-scsi还无法支持空间回收(qemu 1.5)，主要原因是windows下的virtio scsi驱动并未对NTFS下发的TRIM指令进行捕获并翻译为SCSI的UNMAP指令。考虑到Windows得驱动栈，应该需要实现一个filter driver。当前社区正在进行此项工作，预计不久就会予以支持。


## 参考资料

* Fallocate： http://man7.org/linux/man-pages/man2/fallocate.2.html
* BLKDISCARD: http://man7.org/linux/man-pages/man8/blkdiscard.8.html
* AHCI & IDE： http://communities.intel.com/thread/8934
* QEMU Discard Option：http://lists.nongnu.org/archive/html/qemu-devel/2013-03/msg02851.html
* TRIM实例：http://dustymabe.com/2013/06/11/recover-space-from-vm-disk-images-by-using-discardfstrim/
* TRIM实例2：http://dustymabe.com/2013/06/21/guest-discardfstrim-on-thin-lvs/
* 一种利用loop设备+ext4的TRIM方式：http://www.outflux.net/blog/archives/2012/02/15/discard-hole-punching-and-trim/





