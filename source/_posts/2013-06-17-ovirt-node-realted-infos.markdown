---
comments: true
date: 2013-06-17 20:40:31
layout: post
title: 'Ovirt Node Related Infomation'
categories: 
- Virtualization
tags:  [oVirt, livecd, CIM]

---

## 简介

ovirt-node 是开源虚拟化管理平台 oVirt 的客户端部分，它是一个经过定制的最小化 fedora 系统，充当虚拟机管理器（Hypervisor）host 的角色，与管理端 overt-engine 组成一个虚拟化管理平台。ovirt-node 以其小巧，灵活的特点，可以很方便地进行开发与定制。

ovirt-node提供的镜像中包含libvirt/vdsm（Virtual Desktop Server Manager)、KVM等虚拟化服务，使用libvirt/vdsm管理KVM虚拟机。

ovirt-node提供的镜像所需资源很小，可以将更多的资源保留给VM使用。

综合来说，ovirt-node的特点包括：

* Dedicated hypervisor
* JEOS
* livecd
* Built on Fedora
* Firmware
* Small Footprint (<200MB)

## 优缺点

<!-- more-->

优点：

* Single Image
* Easy Upgrades
* No managing individual package updates
* Upgrade directly from management

不足：

* Lack of Customization
* No easy shell access
* More difficult to debug problems

## 架构

ovirt-node基本上是基于layered kickstarts，%post脚本可以用于处理默认的配置，同时提供有TUI界面来进行安装和安装后配置。

安装包：

* ovirt-node：配置ISO
* ovirt-node-tools: working with ISO
* ovirt-node-recipe: building the ISO
* ovirt-node-iso: wrap the ISO
* ovirt-node-plugin-*: 插件

关键技术：

* qemu-kvm
* libvirt
* spice
* device-mapper-multipath
* newt/snack
* Livecd-tools

## 配置持久化

默认其配置均是非持久化的。持久化的配置可以存放在/config，默认容量上限是8MB，同时ovirt-node提供了persist和unpersist命令来支持持久化操作

## 安装和配置

ovirt-node的部署方式：

* CD/DVD/Virtual CD
* Flash Memory (USB/SD Card)
* Network (PXE)

可以安装到disk上(HDD/Flash Disk)，可以自动部署或手动部署。其中自动部署通过自定义内核启动参数实现，手动部署使用TUI界面进行。

安装后可使用admin账户登录进行配置。

可以很方便的进行升级，可以自动进行（命令行）或通过TUI进行。升级通常指的就是从新image中启动：

* Update the PXE image
* Boot new CD/USB/SD
* In Place Upgrade
    * upload new image to running system
    * Trigger upgrade logic
    * Used by oVirt Engine

升级前会自动将原root备份到RootBackup，在升级失败后会进行回滚

## 插件机制

ovirt-node的目标是提供一个最小化的通用性的base image，然后不同的应用场景（如Openstack、ovirt等等）通过plugin提供需要的包然后基于base image制作出相应的镜像(使用ovirt-node-tools)。

目前官方的ovirt-node的镜像中包含有vdsm等，将来会移除出去

## 无状态运行

ovirt-node支持stateless模式，通过内核启动参数可以激活此模式，在此模式下会忽略所有的本地存储

## References

* http://www.ovirt.org/images/1/1e/Ovirt-node-2013-01-23.pdf
* http://www.ibm.com/developerworks/cn/linux/l-cn-ovirt/
