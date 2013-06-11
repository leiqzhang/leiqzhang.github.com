---
comments: true
date: 2010-11-12 14:44:38
layout: post
slug: wget-mirror
title: wget下载整个网站的镜像
wordpress_id: 32
categories:
- Linux/Unix
tags: [wget]

---

<span style="background-color: #ffff00;">$ wget --mirror -p --convert-links -P ./LOCAL-DIR WEBSITE-URL</span>





	
  * –mirror : 下载镜像

	
  * -p : 下载gif等显示html必须的资源

	
  * –convert-links : 下载后，将文档中所有的链接进行转换，以支持本地查看

	
  * -P ./LOCAL-DIR : 需要保存下载文档的目录


