---
comments: true
date: 2013-04-17 11:38:31
layout: post
slug: qemu-block-features
title: 'QEMU Block Features: Image Formats'
categories:
- QEMU
- Virtualization
tags: [qemu, kvm, libvirt]

---

### 镜像格式

1. 支持的格式

     >  Supported formats: vvfat **vpc** **vmdk** vdi sheepdog sheepdog sheepdog **raw** host_cdrom host_floppy host_device file **qed** **qcow2** qcow parallels nbd nbd nbd dmg tftp ftps ftp https http cow cloop bochs blkverify blkdebug 
2. 镜像创建
	* CMD: qemu-img
	* USAGE: qemu-img create [-q] [-f fmt] [-o options] filename [size]
3. qemu启动虚拟机
	* ARGS： -drive file=/path/to/img,if=none,format=**FORMAT**,cache=none,id=drive0
4. libvirt配置
	<pre><code>
		<disk type="file" device="disk">
		    <driver name="qemu" type="raw"/> 
		    <!-- avaliable types when use the name "qemu": raw, bochs, qcow2 and qed -->
		    <!-- optional attributes: cache, io, ioeventfd, error_policy, event_idx, copy_on_read -->
		    <!-- cache attr: control the cache mechanism, possiable values are "default", "none", "writethrough", "writeback", "directsync" ( pass host page cache ) and "unsafe". -->
		    <!-- io attr: "thread" or "native" -->
		    <!-- ioeventfd attr: "on" or "off" -->
		    <!-- error_policy attr: "stop","report","ignore", and "enospace" -->
		    <!-- event_idx attr: "on" or "off", if "on", will reduce the number of interrupts and exits for the guest -->
		    <!-- copy_on_read attr: "on" or "off" -->
		    <!-- @see http://libvirt.org/formatdomain.html#elementsDevices -->
		    <source file="/path/to/img"/>
		    <target dev="vda" bus="virtio"/>  
		    <!-- avaliable bus: ide, scsi, virtio, usb, sata, etc. if ommited, the bus type is inferred from the style of the device name (e.g. "sda" => "scsi"). -->
		</disk>
	</code></pre>



