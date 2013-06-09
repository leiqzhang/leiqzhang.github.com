---
comments: true
date: 2011-04-02 00:38:37
layout: post
slug: install-mac-os-snow-leopard-on-vmware
title: vmware上安装Mac OS Snow Leopard
wordpress_id: 143
categories:
- virtualization
- vmware
tags: [mac, vmware]

---

今天在vmware上安装了Mac OS Snow Leopard，记录过程以作备忘和参考。

**安装环境：**

Windows 7 + VmWare 7.1.2

**安装过程：**

1. 下载Hazard修改的Snow Leopard系统镜像，种子地址：http://thepiratebay.org/torrent/5203531/Snow_Leopard_10.6.1-10.6.2_Intel_AMD_made_by_Hazard

2. 下载一个事先定制好的一个vmware虚拟磁盘（VMDK）和引导iso：dariwn_snow.iso，地址：http://download607.mediafire.com/6ce95acc9glg/dhbxnndmznw/Snowy_Vmware_files.tbz2

3. 通过vmware打开step 2中下载的vmdk虚拟机，编辑其参数，将dariwn_snow.iso挂载到cd中，并开启此虚拟机

4. 出现boot menu后，通过vmware插入step 1中下载的系统镜像（连接），然后根据boot menu提示按c启动安装

5. 安装完毕在提示需要重启的时候，通过vmware将dariwn_snow.iso重新加载到cd中，然后强制重启虚拟机

6. 进入苹果系统

7. 通过vmware连接cd room，将dariwn_snow.iso挂载到系统中，桌面会出现vmware tools图标，点击安装vmware tools

8.安装后会提示重启，直接点击重启虚拟机不能重启，需要手动强制重启。重启进入后，会在系统桌面看到vmware share doc的文件夹，点击进入时会提示找不到此路径，这是因为vmware还未设置共享文件夹的原因。通过vmware添加共享文件夹后即可使用

9. 从http://downloads.sourceforge.net/project/vmsvga2/Display/VMsvga2_v1.2.2_Common_Installer.pkg?use_mirror=garr下载Mac显示驱动，放在共享文件夹中，通过Mac安装，安装后重启，在系统设置中既可以选择更多的分辨率了

**参考：**

http://www.ihackintosh.com/2009/12/install-snow-leopard-in-vmware-7-windows-edition/

http://www.sysprobs.com/increase-screen-resolution-wide-screen-support-mac-os-virtual-machine-vmware-player-workstation
