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
2. Process: snapshot; block-stream; drive-mirror; pivot (optional)
3. Libvirt Supprot
    1. start a vm 
  <pre><code>
		virsh create /root/vm/livecd/livecd.xml 
		Domain livecd created from /root/vm/livecd/livecd.xml
	</code></pre>
    2. start drive mirroring
	<pre><code>virsh blockcopy livecd /root/vm/livecd/data.img /root/a.img --wait 
	    Now in mirroring phase</code></pre>
	<pre><code>virsh blockjob --info livecd /root/vm/livecd/data.img 
	    Block Copy: [100 %]</code></pre>
    3. stop drive mirroring
	<pre><code>virsh blockjob --abort livecd /root/vm/livecd/data.img</code></pre>
    4. have to add reuse flag if the target already exists
	<pre><code>virsh blockcopy livecd /root/vm/livecd/data.img /root/a.img --wait
	    error: unsupported configuration: external destination file for disk vda already exists and is not a block device: /root/a.img</code></pre>
	<pre><code>virsh blockcopy livecd /root/vm/livecd/data.img /root/a.img --wait --reuse-external
	    Now in mirroring phase</code></pre>
    5. just copy, don't go to mirroring phase
	<pre><code>virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/a.img --wait --finish
	    Successfully copied</code></pre>
    6. can not run an in flight blockcopy if drive is in mirroring phase
	<pre><code>virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/a.img --wait --async --reuse-external
	    Now in mirroring phase</code></pre>
	<pre><code>virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/b.img --finish --wait
	    error: block copy still active: disk 'vda' already in active block copy job</code></pre>
	<pre><code>virsh blockcopy livecd /root/vm/livecd/data.img /root/vm/livecd/b.img --finish --wait --async
	    error: block copy still active: disk 'vda' already in active block copy job</code></pre>
    7. can not migrate if drive is in blockcopy
	<pre><code>virsh migrate --live --verbose livecd qemu+ssh://186.100.8.136/system
	    error: Requested operation is not valid: domain has an active block job</code></pre>
    8. switch to the dest when blockcopy complete
	<pre><code>virsh blockcopy livecd /pkgs/imgs/a.img /root/a2.img --pivot --wait
	    Successfully copied</code></pre>
	<pre><code><disk type='file' device='disk'>
		       <driver name='qemu' type='raw' cache='none'/>
		       <source file='/root/a2.img'/>
			<target dev='vda' bus='virtio'/>
			<alias name='virtio-disk0'/>
			<address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
		  </disk></code></pre>
