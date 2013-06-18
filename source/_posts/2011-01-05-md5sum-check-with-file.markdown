---
comments: true
date: 2011-01-05 21:58:32
layout: post
title: md5sum的--check用法
wordpress_id: 124
categories:  [Linux]
tags:  [md5sum]

---

项目开发过程中，有时会批量更新替换一系列其他人提供的文件，文件提供者会给出各文件的路径和md5，在将文件替换后，也需要对md5进行校验。此时可以使用md5sum的--check参数进行批量验证：

用法是

{%codeblock md5sum_check lang:bash %}
md5sum --check ~/data_md5
{%endcodeblock %}

其中data_md5文件的内容如下：

{%codeblock md5sum_data lang:bash %}

MD5值[空格][空格]文件1路径
MD5值[空格][空格]文件2路径
...

{%endcodeblock %}

需要注意的是，每一行中的MD5值和路径之间必须是两个空格，否则md5sum会报文件不存在之类的错误。

检测的输出结果的格式如下：

{%codeblock md5sum_data lang:bash %}

文件1路径： OK
文件2路径： FAILED

{%endcodeblock %}

其中OK表示验证通过（文件md5和文件中给定的md5一直），FAILED表示验证失败
