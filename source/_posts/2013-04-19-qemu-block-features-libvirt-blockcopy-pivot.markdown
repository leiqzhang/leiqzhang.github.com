---
comments: true
date: 2013-04-19 11:30:31
layout: post
title: 'QEMU Block Fetures: libvirt blockcopy switch to dest when complete'
categories:  [Virtualization]
tags:  [QEMU, libvirt, blockcopy]

---

## libvirt blockcopy switch to dest when complete

1. create vm
{% codeblock create vm %}
virsh create /pkgs/imgs/win2003.xml
{% endcodeblock %}
2. start blockcopy
{% codeblock start blockcopy %}
virsh blockcopy win2003 /pkgs/imgs/win2003.img /pkgs/imgs/t322.img --pivot --wait
{% endcodeblock %}
3. query blockcopy info
<!-- more -->
{% codeblock query blockcopy info %}
virsh blockjob --info win2003 /pkgs/imgs/win2003.img
Block Copy: [ 17 %]
{% endcodeblock %}
4. dumpxml after job complete
{% codeblock dumpxml %}
virsh dumpxml win2003
{% endcodeblock %}

{% codeblock xmlinfo lang:xml %}
<disk type='file' device='disk'>
  <driver name='qemu' type='raw' cache='none'/>
  <source file='/pkgs/imgs/t322.img'/>
   <target dev='vda' bus='virtio'/>
   <alias name='virtio-disk0'/>
   <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
</disk>
{% endcodeblock %}
