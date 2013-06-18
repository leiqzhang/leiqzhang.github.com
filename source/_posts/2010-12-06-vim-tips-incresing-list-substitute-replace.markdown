---
comments: true
date: 2010-12-06 17:37:57
layout: post
title: VIM中通过substitute/replace快速生成一系列连续的IP地址
wordpress_id: 119
categories: [VIM]
tags:  [eval, submatch]

---

今天有这么一个操作，需要快速添加一个IP段，如220.181.50.206～220.181.50.254，我需要快速生成该段中一个所有IP地址的列表，以进行IP地址的导入。

在VIM中可以这样：

首先输入第一行:

[code]

220.181.50.206

[/code]

然后yy复制第一行，并48p复制出49行"220.181.50.206", 然后执行以下命令即可

[code]

：%s/50.\zs\(\d\+\)\ze$/\=eval(submatch(1)+line(".")-1)/g

[/code]

该命令是指对于所有匹配的部分（ip最后一个段），加上当前行号减一，这样就生成了一个从206到254的连续ip地址列表。

Just Enjoy It！
