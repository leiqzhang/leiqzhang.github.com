---
comments: true
date: 2013-06-27 22:40:31
layout: post
title: '将多个linear device mapper虚拟磁盘改造为同一磁盘的多个分区'
categories:  [Virtualization]
tags:  [device-mapper, mbr, gpt, fdisk, parted]

---
## 问题

在虚拟化场景下，历史上存在如下虚拟磁盘的管理模式：

在一个LUN上通过linear device mapper的方式制作出若干个DM设备，这些DM设备之间在LUN上并不一定完全相邻，但第一个DM设备必然是从LUN的0扇区开始划分。将这些DM设备以RAW格式的虚拟镜像(卷)挂载给VM使用。

{%codeblock structure%}
               ╔════════════════╗
               ║      VM        ║
               ╚═══════╬════════╝
                       |
                       |
        +--------------+------------+
        |                           |
 ╔══════╬═══════╤══════════╤════════╬═══════╗
 ║  DM          │   GAP    │   DM           ║
 ╚══════════════╧══════════╧════════════════╝
{%endcodeblock%}

为了便于管理，现在需要将这些LUN按照一个单独卷进行管理。如果直接将此LUN挂载给VM,则只能识别到第一个DM设备对应的虚拟磁盘，后面的DM设别将不会被识别。

本文主要讨论如何在不损坏数据且高效的解决此问题，达到透明的管理模式切换，即将LUN挂载给VM后，VM还是可以识别到和之前一样数量的磁盘。

## 问题原因及解决思路

由于之前是通过DM方式划分的LUN，并将不同的DM设备分别进行的文件系统格式化，故在每个DM设备起始位置的MBR或GPT中只会包含有此DM设备相关的分区信息。

当把LUN整体挂载给VM时，VM只能通过此LUN起始位置处的MBR或GPT信息来识别磁盘分区信息，由于LUN起始位置的MBR就是第一个DM设备的MBR，故只能识别到第一个DM设备。

解决此问题的思路也很简单，即将此LUN上除第一个DM外的DM设备的分区信息添加到第一个DM的MBR/GPT中包含的分区表中即可。


## 场景构造

<!-- more -->

简单起见，假设LUN被划分为两个DM设备，每个DM设备均使用MBR方式，且均只有一个分区。第一个DM设备会作为系统镜像（windows系统），第二个设备会作为数据盘。


### 准备系统镜像和数据盘

使用如下命令创建相应大小的RAW镜像：

{%codeblock qemu-img lang:linux%}
qemu-img create -f raw /data/os.img 8G
qemu-img create -f raw /data/data.img 3G
{%endcodeblock%}

挂载windows2003 ISO，使用qemu启动VM，安装操作系统到os.img。使用os.img启动VM且挂载data.img为数据盘，对数据盘进行格式化并写入一些数据以作数据完整性验证。

### 挂载LUN

在IPSAN上划分一个20G大小的LUN，并在Host上通过iscsi-initiator挂载此LUN到本地

### 划分DM

首先从LUN起始位置划分8G大小的DM设备：

{%codeblock dm lang:linux%}
echo -e "0 16777216 linear /dev/sdf 0" | dmsetup create vol-win2003
{%endcodeblock%}

然后和第一个DM相隔4个扇区（模拟俩DM不相邻的情况）创建3G大小的DM设备：

{%codeblock dm lang:linux%}
echo -e "0 6291456 linear /dev/sdf 16777220" | dmsetup create vol-data
{%endcodeblock%}

### 导入OS和Data镜像数据

将之前制作的OS和Data镜像数据分别导入到两个DM设备中：

{%codeblock dm lang:linux%}
dd if=/data/os.img of=/dev/mapper/vol-win2003 bs=1G conv=sync
dd if=/data/data.img of=/dev/mapper/vol-data bs=1G conv=sync
{%endcodeblock%}

## 问题解决

首先通过fdisk命令查看两个DM设备的分区表情况：

{%codeblock fdisk lang:linux%}
	fdisk -l /dev/mapper/vol-win2003
	
	>>>
	Device 	Boot  Start   End      Blocks   Id  System
	（…）   *     63    16755794      (…)    7  HPFS/NTFS/exFAT
	<<<
	
	fdisk -l /dev/mapper/vol-data
	
	>>>
	Device 	Boot  Start   End      Blocks   Id  System
	（…）       	 63    6281414      (…)     7  HPFS/NTFS/exFAT
	<<<
	
{%endcodeblock%}

假设LUN在Host上对应的设备为/dev/sdf,此时通过fdisk查看/dev/sdf的分区情况会和/dev/mapper/vol-win2003一致。

接着操作/dev/sdf分区表，为其新增个分区，系统类型为HPFS/NTFS/exFAT,起始扇区为 1677216 (8G对应的扇区) + 4 (gap) + 63 (start of vol-data) = 16777283, 结束扇区为 1677216+4+6281414 = 23058634.

{%codeblock fdisk lang:linux%}

	fdisk /dev/sdf
	
	> a     # Add partition; input the start (16777283) 
				# and end (23058634)
				
	> t  		# Change the partition's system id with 7
				# eg. HPFS/NTFS/exFAT
				
	> w		# Write changes
	
	fdisk -l /dev/sdf
	
	>>>
	Device 		Boot      Start        End      Blocks   Id  System
	/dev/sdf1   *          63    16755794     8377866    7  HPFS/NTFS/exFAT
	/dev/sdf2        16777283    23058634     3140676    7  HPFS/NTFS/exFAT
	<<<

{%endcodeblock%}


## 结果

直接将/dev/sdf作为RAW镜像挂载给VM启动后，发现VM可以识别到系统盘和数据盘，大小、数据均和之前的镜像一致，从而证明了此方式的有效性。


## 进一步工作

1. 对于GPT分区，需要使用parted而非fdisk进行分区表更改
2. 对于一个DM设备上具有多个分区的情况，特别是具有主分区和扩展分区或者所有DM设备数量超过MBR主分区最大数量（3个）时，需要自行规划将DM设备及其内部的分区分别设置为相应地主分区或扩展分区
3. 将上述步骤脚本化

