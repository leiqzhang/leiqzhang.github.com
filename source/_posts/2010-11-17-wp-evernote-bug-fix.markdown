---
comments: true
date: 2010-11-17 00:27:56
layout: post
slug: wp-evernote-bug-fix
title: WP Evernote Site Memory插件1.1.1一个小问题的修正
wordpress_id: 72
categories:
- WordPress
tags: [ervernote, site memory, wordpress]

---

Evernote是一个十分好用的笔记软件，针对wordpress有相应的WP Evernote Site Memory插件，安装完并进行相应的配置后，可以在每一篇博文开头或结尾有个小的evernote图标，点击后可以发送文章内容到读者自己的evernote笔记本中，十分方便。

在site memory的配置中有如下选项：

![](file:///C:/Users/ADMINI%7E1/AppData/Local/Temp/moz-screenshot.png)![](file:///C:/Users/ADMINI%7E1/AppData/Local/Temp/moz-screenshot-1.png)![设置ContentID](http://leivli.duapp.com/wp-content/uploads/2010/11/11.png)即可以设定evernote获取页面哪一部分作为笔记内容，留空则可以使用id为“post-ID"的div的内容，也即相应博文的内容。

但昨天装了1.1.1后，发现点击某篇博文下的evernote图标后，发送到笔记的内容是整个页面的内容而非单独那篇博文的内容，刚才简单追踪了一下，发现是该插件附带js的一个小bug：生成的按钮中调用clip方法时传递postID的属性名是"contentID",而之后却使用属性名"contentId"进行读取,这样导致Evernote认为没有传递合适的postID，从而就抓取了整个页面的内容。

修改起来也十分简单，只需要将插件中所带的js文件（”plugins/wp-evernote-site-memory/js/noteit.js和noteit.min.js“）中所有的"contentId"替换为"contentID"即可。
