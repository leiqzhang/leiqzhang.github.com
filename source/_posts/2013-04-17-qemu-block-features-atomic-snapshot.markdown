---
comments: true
date: 2013-04-17 11:38:31
layout: post
slug: atomic snapshot
title: 'QEMU Block Fetures: Atomic Snapshot'
categories:
- QEMU
- Virtualization
tags: [qemu, kvm, libvirt]

---

### SnapshotsMultipleDevices

1. ref: [QEMU-Feature-SnapshotsMultipleDevices](http://wiki.qemu.org/Features/SnapshotsMultipleDevices#Atomic_Snapshots_of_Multiple_Devices)

2. QMP Steps
	* [CMD] qemu [...] -qmp tcp:localhost:4444,server
	    * Start QMP on a TCP socket, so that telnet can be used
	    * -qmp conflicts with "-monitor"
	* [CMD] telnet localhost 4444
	    * Run telnet
	    * Should see the QMP's greeting banner:
		<pre><code>{"QMP":{"version": {"qemu": {"micro": 50, "minor": 13, "major": 0}, "package": ""}, "capabilities": []}} </code></pre>
	* [QMP_INPUT] 
	    * Make QMP enter command mode:
		<pre><code>{"execute": "qmp_capabilities"}</code></pre>
	* [QMP_INPUT]
	    * Query the device info:
		<pre><code>{"execute":"query-blockstats"} </code></pre>
		or
		<pre><code>{ "execute": "human-monitor-command", "arguments": { "command-line": "info block" } } </code></pre>
	    * Get the device info from the output
	* [QMP_input]
	    * Atomic snapshot:
		<pre><code>
		{ "execute": "transaction",
		     "arguments": { 
			     "actions": [
				 { 'type': 'blockdev-snapshot-sync', 'data' : { "device": "drive0",
					 "snapshot-file": "/pkgs/imgs/win2003.snap",
					 "format": "qcow2" } },
				 { 'type': 'blockdev-snapshot-sync', 'data' : { "device": "drive1",
					 "snapshot-file": "/pkgs/imgs/data.snap",
					 "format": "qcow2" } } 
			     ] 
		     } 
		}</code></pre>
	    * View Result, will find the current **device/backing_file/backing_file_depth** have already changed:
		<pre><code>{ "execute": "human-monitor-command", "arguments": { "command-line": "info block" } } </code></pre>
	    * CMD Args:
		* type: the operation to perform. The only supported value is "**blockdev-snapshot-sync**".
		* data: a dict which contents depand on the value of "type"
		* if type == "blockdev-snapshot-sync":
		    * device: device name to snapshot ( can get from the **info** block command )
		    * snapshto-file: name of new image file
		    * format: format of new image ( support "qed", "qcow2", default "qcow2" )
		    * mode: whether and how QEMU should create the snapshot file ( NewImageMode, optional, default "absolute-paths" ) ( Can set to "existing", which means ~~do the **internal snapshot**~~ use the file as snap )

3. Libvirt Support
    1 Code
	* Interface **virDomainSnapshotCreateXML** has a input arg **flags**, and can be set with **VIR_DOMAIN_SNAPSHOT_CREATE_ATOMIC** 
	* With **VIR_DOMAIN_SNAPSHOT_CREATE_ATOMIC** libvirt guarantees that this command will not alter any disks unless the entire set of changes can be done atomically, making failure recovery simpler (note that it is still possible to fail after disks have changed, but only in the much rarer cases of running out of memory or disk space).
    2 Steps
	* Create domain
	    * [LIBVIRT-INPUT] <pre><code>virsh create win2003.xml</code></pre>
	    * disks info:
		<pre><code>
		    	<disk type='file' device='disk'>
				<driver name='qemu' type='raw' cache='none'/>
				<source file='/pkgs/imgs/win2003.img'/>
				<target dev='vda' bus='virtio'/>
		    	</disk>
		     	<disk type='file' device='disk'>
				<driver name='qemu' type='qcow2' cache='none'/>
				<source file='/pkgs/imgs/data.qcow2'/>
				<target dev='vdb' bus='virtio'/>
		    	</disk>
		</code></pre>
	* Create Snapshot XML
		<pre><code>
			<domainsnapshot>
			  <description>Test Libvirt Snapshot-Create</description>
			  <disks>
			    <disk name='/pkgs/imgs/win2003.img'>
			      <source file='/pkgs/imgs/w.snap'/>
			    </disk>
			    <disk name='/pkgs/imgs/data.qcow2'>
			      <source file='/pkgs/imgs/d.snap'/>
			    </disk>
			  </disks>
			</domainsnapshot> 
		</code></pre>
	* Create Atomic Snapshot
	    * [LIBVIRT-INPUT] <pre><code>virsh snapshot-create 3 --xmlfile ./snap.xml  --disk-only **--atomic**</code></pre>
	    * [LIBVIRT-OUTPUT] <pre><code>Domain snapshot 1366295688 created from './snap.xml'</code></pre>
	    * Reult Disk Info:
		<pre><code>
			    <disk type='file' device='disk'>
			      <driver name='qemu' type='qcow2' cache='none'/>
			      <source file='/pkgs/imgs/w.snap'/>
			      <target dev='vda' bus='virtio'/>
			      <alias name='virtio-disk0'/>
			      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
			    </disk>
			    <disk type='file' device='disk'>
			      <driver name='qemu' type='qcow2' cache='none'/>
			      <source file='/pkgs/imgs/d.snap'/>
			      <target dev='vdb' bus='virtio'/>
			      <alias name='virtio-disk1'/>
			      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
			    </disk>
		</code></pre>
	
