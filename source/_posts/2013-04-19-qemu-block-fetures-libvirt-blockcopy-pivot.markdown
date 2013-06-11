---
comments: true
date: 2013-04-19 11:30:31
layout: post
slug: libvirt-blockcopy-pivot
title: 'QEMU Block Fetures: libvirt blockcopy switch to dest when complete'
categories:
- QEMU
- Virtualization
tags: [qemu, kvm, libvirt]

---

### libvirt blockcopy switch to dest when complete

1. create vm
    <pre><code>virsh create /pkgs/imgs/win2003.xml</code></pre>
2. start blockcopy
    <pre><code>virsh blockcopy win2003 /pkgs/imgs/win2003.img /pkgs/imgs/t322.img --pivot --wait</code></pre>
3. query blockcopy info
    <pre><code>virsh blockjob --info win2003 /pkgs/imgs/win2003.img</code></pre>
    <pre><code>Block Copy: [ 17 %]</code></pre>
4. dumpxml after job complete
    <pre><code>virsh dumpxml win2003</code></pre>
    <pre><code>
                     <disk type='file' device='disk'>
                          <driver name='qemu' type='raw' cache='none'/>
                          <source file='/pkgs/imgs/t322.img'/>
                           <target dev='vda' bus='virtio'/>
                           <alias name='virtio-disk0'/>
                           <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
                     </disk>
    </code></pre>
