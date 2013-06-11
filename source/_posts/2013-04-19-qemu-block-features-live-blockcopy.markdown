---
comments: true
date: 2013-04-19 15:00:31
layout: post
slug: live-snapshot-blockcopy
title: 'QEMU Block Fetures: live snapshot blockcopy'
categories:
- QEMU
- Virtualization
tags: [qemu, kvm, libvirt]

---

### Live Block Copy

1. Ref: [QEMU-Feature-SnapshotsMultipleDevices-LiveBlockCopy](http://wiki.qemu.org/Features/SnapshotsMultipleDevices#Application_to_live_block_copy)
2. Ref: [more blockcopy usages](https://bugzilla.redhat.com/show_bug.cgi?id=888426)
3. Process: snapshot; block-stream; drive-mirror; pivot (optional)
4. Libvirt Supprot

    1. start a vm 
        {% codeblock %}
    	virsh create /root/vm/livecd/livecd.xml 
    	Domain livecd created from /root/vm/livecd/livecd.xml
        {% endcodeblock %}

    2. start drive mirroring 
        {% codeblock %}
    	virsh blockcopy livecd /root/vm/livecd/data.img /root/a.img --wait 
    	Now in mirroring phase

    	virsh blockjob --info livecd /root/vm/livecd/data.img 
    	Block Copy: [100 %]
        {% endcodeblock %}
    3. stop drive mirroring
``` bash
    virsh blockjob --abort livecd /root/vm/livecd/data.img
```
    4. have to add reuse flag if the target already exists
``` bash
    virsh blockcopy livecd /root/vm/livecd/data.img /root/a.img --wait
    error: unsupported configuration: external destination file for disk vda already exists and is not a block device: /root/a.img
```
```	bash
    virsh blockcopy livecd /root/vm/livecd/data.img /root/a.img --wait --reuse-external
    Now in mirroring phase
```
    5. just copy, don't go to mirroring phase
``` bash
    virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/a.img --wait --finish
    Successfully copied
```
    6. can not run an in flight blockcopy if drive is in mirroring phase
``` bash
    virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/a.img --wait --async --reuse-external
    Now in mirroring phase
```
``` bash
    virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/b.img --finish --wait
    error: block copy still active: disk 'vda' already in active block copy job
```
``` bash
    virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/b.img --finish --wait --async
    error: block copy still active: disk 'vda' already in active block copy job
```
    7. can not migrate if drive is in blockcopy
``` bash
    virsh migrate --live --verbose livecd qemu+ssh://186.100.8.136/system
    error: Requested operation is not valid: domain has an active block job
```
    8. switch to the dest when blockcopy complete
``` bash
    virsh blockcopy livecd /pkgs/imgs/a.img /root/a2.img --pivot --wait
    Successfully copied
```
``` xml
    <disk type='file' device='disk'>
        <driver name='qemu' type='raw' cache='none'/>
        <source file='/root/a2.img'/>
        <target dev='vda' bus='virtio'/>
        <alias name='virtio-disk0'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </disk>
```
