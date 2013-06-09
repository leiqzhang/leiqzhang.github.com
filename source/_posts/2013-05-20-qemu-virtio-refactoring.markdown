---
comments: true
date: 2013-05-20 20:40:31
layout: post
slug: qemu-virtio-refactoring
title: 'QEMU Fetures: Virtio Refactoring'
categories:
- QEMU
- Virtualization
tags: [qemu, kvm, libvirt, scsi, virtio]

---

### Background

在研究QEMU的virtio-scsi时，发现了如下问题: 1.5的qemu相比1.2多了几种virtio及scsi相关的设备

1. qemu-1.2

    > **CMD:** qemu-kvm -device ? 2>&1 | grep "virtio\|scsi"

    > **OUTPUT:** 
    
    > - name "scsi-hd", bus SCSI, desc "virtual SCSI disk"
    > - name "virtio-blk-pci", bus PCI, alias "virtio-blk"
    > - name "virtio-9p-pci", bus PCI
    > - name "scsi-block", bus SCSI, desc "SCSI block device passthrough"
    > - name "scsi-generic", bus SCSI, desc "pass through generic scsi device (/dev/sg*)"
    > - name "scsi-disk", bus SCSI, desc "virtual SCSI disk or CD-ROM (legacy)"
    > - name "virtserialport", bus virtio-serial-bus
    > - name "scsi-cd", bus SCSI, desc "virtual SCSI CD-ROM"
    > - name "virtconsole", bus virtio-serial-bus
    > - name "virtio-serial-pci", bus PCI, alias "virtio-serial"
    > - name "am53c974", bus PCI, desc "AMD Am53c974 PCscsi-PCI SCSI adapter"
    > - name "virtio-net-pci", bus PCI, alias "virtio-net"
    > - name "virtio-balloon-pci", bus PCI, alias "virtio-balloon"
    > - name "virtio-scsi-pci", bus PCI

2. qemu-1.5

    > **CMD:** ./x86_64-softmmu/qemu-system-x86_64  -device ? 2>&1 | grep "virtio\|scsi"

    > **OUTPUT:** 
    
    > - name "scsi-hd", bus SCSI, desc "virtual SCSI disk"
    > - name "virtio-blk-pci", bus PCI, alias "virtio-blk"
    > - name "vhost-scsi-pci", bus PCI
    > - name "scsi-block", bus SCSI, desc "SCSI block device passthrough"
    > - name "virtio-serial-device", bus virtio-bus
    > - name "virtio-scsi-common", bus virtio-bus
    > - name "scsi-generic", bus SCSI, desc "pass through generic scsi device (/dev/sg*)"
    > - name "vhost-scsi", bus virtio-bus
    > - name "scsi-disk", bus SCSI, desc "virtual SCSI disk or CD-ROM (legacy)"
    > - name "virtserialport", bus virtio-serial-bus
    > - name "scsi-cd", bus SCSI, desc "virtual SCSI CD-ROM"
    > - name "virtio-net-device", bus virtio-bus
    > - name "virtconsole", bus virtio-serial-bus
    > - name "virtio-serial-pci", bus PCI, alias "virtio-serial"
    > - name "virtio-balloon-device", bus virtio-bus
    > - name "virtio-rng-pci", bus PCI
    > - name "pvscsi", bus PCI
    > - name "am53c974", bus PCI, desc "AMD Am53c974 PCscsi-PCI SCSI adapter"
    > - name "virtio-balloon-pci", bus PCI, alias "virtio-balloon"
    > - name "virtio-scsi-pci", bus PCI
    > - name "virtio-blk-device", bus virtio-bus
    > - name "virtio-scsi-device", bus virtio-bus
    > - name "virtio-net-pci", bus PCI, alias "virtio-net"


### Questions

1. What is virtio-bus?
2. Which device can provide virtio-bus?
3. How to use virtio-scsi-device or virtio-blk-device?
4. What is virtio-scsi-common?

### Root Cause: Virtio Refactoring

1. Ref: http://wiki.qemu.org/Features/virtio-refactoring
2. Details:

    > Actually QEMU has a lot of virtio devices, which are not QOM compliant; they can be used with two different transports:
    > 
    > PCI: with eg: -device virtio-blk-pci.
    > S390: with eg: -device virtio-blk-s390.
    > The new implementation is as follows:
    > 
    > A new bus is introduced: virtio-bus which is an abstract bus.
    
    > For each transport a specific implementation of virtio-bus is created. (eg: virtio-pci-bus for virtio-pci).
    
    > Each virtio transport is built with a virtio-bus that the virtio devices connect to.
    > VirtIODevice becomes a device which is abstract and must be connected on a virtio-bus.
    
    > Each virtio device are created (eg: virtio-blk) extends VirtioDevice and then connects on a virtio-bus.
    
    > As we discussed, we must keep the backward compatibility. So the re-factored device must have the same properties and the same behavior. We should have only one way to create the device, so the transport/virtio-bus/virtio-device layout is only internal to QEMU for S390 and PCI device.

### Answers:

* __Q:__ What is virtio-bus?
    > __A:__ An abstract bus for virtio device

* __Q:__ Which device can provide virtio-bus?
    > __A:__ Every virtio related bus, such as virtio-pci-bus, virtio-scsi-bus, etc.

* __Q:__ How to use virtio-scsi-device or virtio-blk-device?
    > __A:__ Both of them are **VirtIODevice** mentioned above. There is no way to use them explicit

* __Q:__ What is virtio-scsi-common?
    > __A:__ Maybe not so related to this subject. It is used for new device supporting the tcm_vhost

    > __Ref:__ http://lists.gnu.org/archive/html/qemu-devel/2013-01/msg05821.html

    > __Summary:__

    > Ok, so here is my attempt at a vhost-scsi device.  I'm creating an entirely separate device, with the common parts of virtio-scsi and vhost-scsi (actually little more than the initialization) grouped into a VirtIOSCSICommon type.  The device is used simply like "-device vhost-scsi-pci,wwpn=WWPN", with all configuration done in configfs beforehand.

### The Change of Qtree

1. CMD
    > qemu-kvm -enable-kvm -name win2003 -M pc-0.15 -m 1024 -boot c -drive file=/pkgs/imgs/win7.img,if=none,format=raw,cache=none,id=drive0 -monitor stdio -vga qxl -vnc 186.100.8.144:0 -nodefaults  -device virtio-scsi-pci,id=scsi -device scsi-hd,drive=drive0 -device piix3-usb-uhci,id=usb,bus=pci.0,addr=0x1.0x2 -device usb-tablet,id=input0

2. Qtree of Qemu 1.2

```
    dev: virtio-scsi-pci, id "scsi"
        ioeventfd = on
        ...
        bus: scsi.0
            type SCSI
                dev: scsi-hd, id ""
                    drive = drive0 
```           

3. Qtree of Qemu 1.3

```
	    dev: virtio-scsi-pci, id "scsi"
         ioeventfd = on
         ...
         bus: scsi.0
           type virtio-pci-bus
           dev: virtio-scsi-device, id ""
             num_queues = 1
             max_sectors = 65535
             cmd_per_lun = 128
             bus: scsi.0
               type SCSI
               dev: scsi-hd, id ""
                 drive = drive0
```

