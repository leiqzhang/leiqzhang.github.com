---
comments: true
date: 2013-05-06 16:56:31
layout: post
title: 'QEMU Contribute: steps of submit a patch'
categories:  [Virtualization]
tags:  [QEMU]

---

1. 邮件的收件人和抄送人
    * 收件人：qemu-devel@nongnu.org
    * 抄送人
        * 参考代码库中的MAINTAINERS文件
        * 使用scripts/getmaintainer.pl来获取此文件的主要提交者

2. 邮件的主题和内容的格式
<!-- more -->
    * 使用git format-patch来生成标准格式的patch
    * 需要带-s参数使得生成的PATCH中包含signed-off-by开头的行
    * 邮件主题使用此命令生成的Patch文件中的subject
    * 邮件正文使用此文盲生成的Patch文件中的body
    * 包含大量更改的Patch，需按代码逻辑拆分为多个patch，组成patch series
        * Patch series要求
            * 每部分的change均需保证可编译通过
            * 文档类的change靠前
        * 邮件要求
            * 最开始是一个cover leter，必须包含PATCH的goal和diffstat，尽量使得cover letter的内容有意义
      * 每部分change单独放到一封邮件中，并回复cover letter，标题为[PATCH v1 0/5]等形式
3. Patch要求
    * 符合QEMU的代码规范
    * 注释明确，使用英文，没有拼写错误
    * 需要基于当前git master生成Patch
    * 不要包含无关内容

4. 邮件发出后
    * 跟进社区的回复，并持续跟进
    * 如果是在社区review后再次发送的Patch，需在标题后加上v2、v3等后缀，依次类推
    * 当社区中有协助进行review的，在后续版本的PATCH中恰当使用"Reviewed-by" tag，使得reviewer方便区分review内容
    * 如果发现PATCH发出后没有回应，每隔一到两周，“全部回复”之前的Patch邮件，并在主题中带上”Ping“
