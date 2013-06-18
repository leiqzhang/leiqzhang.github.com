---
comments: true
date: 2013-04-19 15:00:31
layout: post
title: 'QEMU Block Fetures: live snapshot blockcopy'
categories:  [Virtualization]
tags:  [QEMU, libvirt, blockcopy]

---

## Live Block Copy

1. Ref: [QEMU-Feature-SnapshotsMultipleDevices-LiveBlockCopy](http://wiki.qemu.org/Features/SnapshotsMultipleDevices#Application_to_live_block_copy)
2. Ref: [more blockcopy usages](https://bugzilla.redhat.com/show_bug.cgi?id=888426)
3. Process: snapshot; block-stream; drive-mirror; pivot (optional)
4. Libvirt Supprot
<!-- more -->

{% codeblock start a vm %}
virsh create /root/vm/livecd/livecd.xml 
Domain livecd created from /root/vm/livecd/livecd.xml
{% endcodeblock %}

{% codeblock start drive mirror %}
virsh blockcopy livecd /root/vm/livecd/data.img /root/a.img --wait 
Now in mirroring phase

virsh blockjob --info livecd /root/vm/livecd/data.img 
Block Copy: [100 %]
{% endcodeblock %}

{% codeblock stop drive mirror %}
virsh blockjob --abort livecd /root/vm/livecd/data.img
{% endcodeblock %}

{% codeblock have to add reuse flag if the target already exists %}
virsh blockcopy livecd /root/vm/livecd/data.img /root/a.img --wait
error: unsupported configuration: external destination file for disk vda already exists and is not a block device: /root/a.img

virsh blockcopy livecd /root/vm/livecd/data.img /root/a.img --wait --reuse-external
Now in mirroring phase
{% endcodeblock %}

{% codeblock just copy, don't go to mirroring phase %}
virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/a.img --wait --finish
Successfully copied
{% endcodeblock %}

{% codeblock can not run an in flight blockcopy if drive is in mirroring phase %}
virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/a.img --wait --async --reuse-external
Now in mirroring phase

virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/b.img --finish --wait
error: block copy still active: disk 'vda' already in active block copy job

virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/b.img --finish --wait --async
error: block copy still active: disk 'vda' already in active block copy job
{% endcodeblock %}

{% codeblock can not migrate if drive is in blockcopy %}
virsh migrate --live --verbose livecd qemu+ssh://186.100.8.136/system
error: Requested operation is not valid: domain has an active block job
{% endcodeblock %}

{% codeblock switch to the dest when blockcopy complete %}
virsh blockcopy livecd /pkgs/imgs/a.img /root/a2.img --pivot --wait
Successfully copied
{% endcodeblock %}

{% codeblock after complete lang:xml %}
<disk type='file' device='disk'>
    <driver name='qemu' type='raw' cache='none'/>
    <source file='/root/a2.img'/>
    <target dev='vda' bus='virtio'/>
    <alias name='virtio-disk0'/>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
</disk>
{% endcodeblock %}
