---
comments: true
date: 2013-04-17 11:40:31
layout: post
slug: configure-qemu-kvm-env-in-f18
title: '新的F18环境下KVM环境搭建'
categories:
- QEMU
- Virtualization
tags: [qemu, kvm, libvirt]

---

# 新的F18环境下KVM环境搭建

1. 更改hostname，从localhost改为有意义的名字
    > hostnamectl set-hostname DESIRED_HOST_NAME

2. 安装git: yum install git
3. clone KVM project
    > git clone http://path/to/qemu.git

4. 编译
    1. ./configure '--enable-kvm' '--target-list=x86_64-softmmu' '--enable-virtio-blk-data-plane' --disable-slirp
    2. 根据prefix的结果yum install缺失的包，几个主要的包：
  * data-plane: libaio-devel
	* 根据configure的结果和已有环境的结果进行比对，对于未正常启用的安装相应的包
		* SDL support： yum install SDL-devel
		* curses support：yum install ncurses-devel
		* curl support：yum install libcurl-devel
		* VNC
		    * VNC TLS support：gnutls-devel
		    * VNC SASL support：cyrus-sasl-devel
		    * VNC JPEG support：libjpeg-turbo-devel
		    * VNC PNG support：libpng-devel
		    * VNC WS support：gnutls-devel
		* Documention: pod2man/makeinfo, yum install texinfo
		* fdt support:  libfdt-devel
		* uuid support: libuuid-devel
		* spice support: yum install spice-protocol spice-server spice-server-devel
		* seccomp support: libseccomp-devel

5. 防火墙
    1. systemctl stop firewalld.service; 
    2. systemctl disable firewalld.service

6. libvirt：
    1. virsh: libvirt-client
    2. libvirtd: libvirt-daemon
    3. libvirtd start related error
	* virsh list
		* error: Failed to reconnect to the hypervisorerror: no valid connectionerror: Failed to connect socket to '/var/run/libvirt/libvirt-sock': No such file or directory
			* libvirtd -l ; 并观察错误输出，可找到所缺失的so文件，进而找到缺失的rpm包
			* yum install libvirt-daemon-driver-*
		* error: libvirtd start error: Cannot read CA certificate '/etc/pki/CA/cacert.pem': No such file or directory
			* configuration:  /etc/libvirt/libvirtd.conf; uncomment the line: "listen_tls=0"

