---
comments: true
date: 2010-11-14 23:18:45
layout: post
slug: vim-force-overwrite
title: VIM overwrite read-only file
wordpress_id: 36
categories:
- vim
tags: [vim]

---

今天在使用vim编辑一个read-only的文件（权限为r-- r-- r--)，在保存时虽然提示只读不能保存，但是通过w!可以强制覆盖原文件内容，难道VIM可以绕过权限限制么？

后来发现是在使用w！时，vim会强制删除原文件，并新建一个新的同名文件，并写入新的内容，所以，使用w！强制保存时，就要求当前用户有删除原文件的权限，即：



	
  1. 该文件所在的目录必须具有w权限

	
  2. 如果所在目录设置了t标志，还需要当前文件的所有者为自己


其实这也可以从vim的帮助中看到：


*write-readonly*


When the 'cpoptions' option contains 'W', Vim will refuse to overwrite a

readonly file.  When 'W' is not present, ":w!" will overwrite a readonly file,

if the system allows it (the directory must be writable).
