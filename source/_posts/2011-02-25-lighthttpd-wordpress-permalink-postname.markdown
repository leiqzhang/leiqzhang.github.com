---
comments: true
date: 2011-02-25 20:24:13
layout: post
title: lighthttpd实现WordPress永久链接(Permalink)
wordpress_id: 129
categories:  [WordPress]
tags:  [lighthttpd, permalink, rewrite]

---

WordPress的永久链接(permalink)可支持自定义的链接结构,如本博客的url格式为http://leivli.duapp.com/%postname%,其中postname表示文章的别名。

站点迁移到lighthttpd环境后，需要进行一些配置才可以支持:

[code]

$HTTP["host"] =~ "leivli\.(.*)" {
url.rewrite-once = (
 "^/(wp-admin\/.*)/?$" => "/$1",
 "^/([_0-9a-zA-Z-]+)/?$" => "/index.php?name=$1",
 )
}

[/code]

原理其实就是将/postname形式的url重新改为index.php?name=postname的形式，但需要特殊处理管理入口(/wp-admin)，否则会认为wp-admin也是一个postname
