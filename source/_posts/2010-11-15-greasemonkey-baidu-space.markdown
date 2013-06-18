---
comments: true
date: 2010-11-15 01:12:11
layout: post
title: 百度空间插入代码段和HTML代码片段的GreaseMonkey脚本
wordpress_id: 40
categories:  [web_dev]
tags:  [greasemonkey, javascript, css]

---

现在百度空间写日志时，在IE/firefox下的文本编辑器是不能直接查看和编辑日志的HTML代码的，应该是基于安全考虑。目前在chrome下，百度空间的兼容性不好，写新日志或者编辑日志时，直接是提供的一个文本框，可以直接敲入HTML代码（当然了，写入<script>是会被过滤掉的)。

如果空间可以支持直接插入代码片段（语法高亮、缩进等等），则空间用来写技术文档就更方便了。最近刚好发现VIM的TOhtml命令非常强大，可以直接把编辑器中文本按照其颜色、缩进等生成相应的HTML片段，如果空间可以直接插入一段HTML代码的话，代码片段的插入也就不成问题了。

之前在[雪候鸟](http://www.laruence.com/)的博客上看到了[东方时尚约车脚本](http://www.laruence.com/2010/03/31/1377.html)相关的内容，联想我的需求，参考其东方时尚约车脚本写了一个简单的Greasemonkey脚本，主要在以下两方面对空间的编辑器进行了扩展：
1. 支持查看当前日志的HTML代码，以便进行简单的修改，和粘贴来自VIM的代码片段
2. 支持直接插入一段来自VIM的代码段，该代码段可以具有一定的样式，如常见的有边框包围，字体更清晰等等

**安装方法:**
1. 需要firefox
2. firefox需要安装greaseMonkey插件, 插件地址: [https://addons.mozilla.org/en-US/firefox/addon/748](https://addons.mozilla.org/en-US/firefox/addon/748)
3. 用firefox浏览本页, 并[点此](http://userscripts.org/scripts/show/78912)下载安装

**使用方法：**

安装后打开自己的空间，“新建文章”或者“编辑”已发布的日志时，在标题的右侧会出现“插入代码段“和”HTML视图“两个按钮，如下图：

![](http://leivli.duapp.com/wp-content/uploads/1.png)

其中”**插入代码段**“可以直接插入一段代码段，如我们VIM中TOhtml一个php脚本文件的结果，然后复制到”代码段“中：

![](http://leivli.duapp.com/wp-content/uploads/2.png)

以下是结果：


1 <?php
2 **$**file **=** file("harddisk.txt");
3 **foreach**(**$**file **as** **$**line){
4  **$**line **=** trim(**$**line);
5  **$**s **=** preg_match_all("/(?:\d)+[GTM]/i",**$**line,**$**matches,PREG_PATTERN_ORDER);
6  **if**(**!****$**s){
7  echo "error parse.. "**.****$**line**.**"\n";
8  }
9  **else** **if**(**$**s **>** 1){
10  echo "multi match .. "**.****$**line**.**": \n";
11  **foreach**(**$**matches[0] **as** **$**match){
12  echo "**$**match\n";
13  }
14  }
15  **else**{
16  echo "find.."**.****$**line**.**": \n";
17  echo **$**matches[0][0]**.**"\n";
18  }
19 }
20 ?>



代码段插入后，双击代码段可以进行**编辑**。

”**HTML视图**“则显示当前日志的HTML代码，可以直接编辑：

![](http://leivli.duapp.com/wp-content/uploads/3.png)
该对话框上面的”更新“按钮可以将改动写回空间的编辑器，而”加载“则是可以再次加载空间编辑器中的内容
