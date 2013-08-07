---
comments: true
date: 2013-06-18 20:40:31
layout: post
title: 'White background after boot menu of isolinux in ovirt-node'
categories:  [Virtualization]
tags:  [oVirt, isolinux, syslinux, vesamenu, vga]

---
## 问题

当通过llivecd启动系统进行第一次部署时，在BOOT MENU页面(isolinux)后，会首先出现一段时间的“白屏”现象，然后才可以进入到安装和配置界面

## 问题原因

确认是启动参数中缺少了vga=选项

可参考： https://wiki.archlinux.org/index.php/Syslinux#White_block_in_upper_left_corner_when_using_vesamenu

{% blockquote archlinux_wiki,White_block_in_upper_left_corner_when_using_vesamenu https://wiki.archlinux.org/index.php/Syslinux#White_block_in_upper_left_corner_when_using_vesamenu ref_detail %}
Problem: As of linux-3.0, the modesetting driver tries to keep the current contents of the screen after changing the resolution (at least it does so with my Intel, when having Syslinux in text mode). It seems that this goes wrong when combined with the vesamenu module in Syslinux (the white block is actually an attempt to keep the Syslinux menu, but the driver fails to capture the picture from vesa graphics mode).

If you have a custom resolution and a vesamenu with early modesetting, try to append the following in syslinux.cfg to remove the white block and continue in graphics mode:

APPEND root=/dev/sda6 ro 5 vga=current quiet splash
{% endblockquote %}


## 社区相关BUG及描述

<!-- more -->

相关BUG可参考：https://bugs.archlinux.org/task/26452

关于此BUG，一些开发者的回复如下：

{% blockquote Tasos,bug_comment_1 https://bugs.archlinux.org/task/26452 ref_detail %}
Now that I 've been thinking of it. I am using a custom syslinux.cfg file with resolution set at 1400x1050 pixels. How is it supposed for the modesetting driver to keep the current contents on the screen if 1400x1050 resolution is used in the first place and changed back and forth with the 640x480 resolution? That's probably why I see the white block (that block is actually 640x480 pixels). If I had a syslinux 640x480 resolution, I would probably not see a white block, but the syslinux menu. And that's why vga=current fixes it because it does not change resolution.So I think this is a modesetting bug rather than a syslinux one, __if not a bug at all__. I also wonder what will happen if I have two monitors connected with different native resolution, but in anyway its not in the scope of this bug report and neither it is for discussion.
{% endblockquote %}


{% blockquote Thomas Bachler,bug_comment_2 https://bugs.archlinux.org/task/26452 ref_detail %}
As of 3.0, the modesetting driver tries to keep the current contents of the screen after changing the resolution (at least it does so with my intel, when having syslinux in text mode). It seems that this goes wrong when combined with the vesamenu module in syslinux (the white block is actually an attempt to keep the syslinux menu, but the driver fails to capture the picture from vesa graphics mode).It is my understanding that this block goes away when you either scroll it away or clear the screen. As this is __likely a restriction in the graphics hardware__, I am tempted to close this as __"Won't fix"__.In any case, I will try to reproduce this (I do not use vesamenu currently, but the text-based syslinux menu).
{% endblockquote %}

即各位认为此不属于syslinux应该解决的BUG

## 跟进

按照上文描述，在启动参数后加了vga=current后，在经过了MENU步骤后，分辨率不会自动发生变更，VNC窗口一直显示的是MENU的背景图片，无法自动进行分辨率切换并进入系统。

但当在启动参数后面增加vga=0x317后功能正常，但不同的是进入系统后，VNC窗口的分辨率为1024*768

关于vga选项的取值及其含义可参考：http://wiki.antlinux.com/pmwiki.php?n=HowTos.VgaModes 

从上面资料可知0x317对应的分辨率是Kernel VESA的1024*768分辨率。

试了一下，当前litevirt默认的vga是0x111，当指定vga=0x111时和vga=current的现象一直，不会进入系统。只有vga指定了一个和当前vga不同的值方可正常进入，也无白屏问题，但是这样的话进入系统后分辨率就固定了。

这样会否影响安装后的系统的分辨率？经过安装完毕直接重启后验证，不会有影响。而且admin账户的管理portal中也没有可以设置分辨率的入口

## 结论

因为当前白屏问题只是在第一次安装时出现，且不影响安装后系统的使用，故可以考虑直接在启动参数中添加“vga=xxx”的参数，只需保证这里vga的取值不要和isolinux默认或者自定义的MENU界面分辨率相同即可（经实验，当前isolinux默认的分辨率是0x111）

## 另外

另外,有人提出Ubuntu针对此问题有相应的fix，但是fedora没有：http://thr3ads.net/syslinux/2011/02/1149712-not-sure-how-to-get-rid-of-white-screen-during-boot

文中提到的两个相关的commit如下： 

1. http://kernel.ubuntu.com/git?p=apw/ubuntu-natty.git;a=commit;h=c4a3d548b57f3c511218d7f1294a7426da5e2b4e
2. http://kernel.ubuntu.com/git?p=apw/ubuntu-natty.git;a=commit;h=8a5bd3f5f45544b4a92015f3e9f1e6998d927b03
