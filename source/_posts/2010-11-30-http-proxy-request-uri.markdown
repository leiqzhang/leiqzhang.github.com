---
comments: true
date: 2010-11-30 21:27:24
layout: post
title: HTTP请求头中Request-URI的取值和代理服务器proxy的关系
wordpress_id: 112
categories: [web_dev]
tags:  [http, proxy]

---

一个合法的HTTP URL的格式如下：

[code]http_URL = "http:" "//" host [ ":" port ] [ abs_path [ "?" query ]][/code]

当访问一个网站时，http的请求头中会包含一个Request-URI字段，该字段可以有四种类型的取值：

[code]Request-URI    = "*" | absoluteURI | abs_path | authority[/code]

其中星号"*"表示客户端请求的是服务器本身，而并没有请求特定的资源；authority格式只用于http协议的CONNECT方法。这两种本文不进行讨论，下面主要看另外两种

当请求是发送给代理服务器时，必须使用absoluteURI(绝对URI)格式。代理服务器需要转发该请求或者直接从本地的cache给出响应。需要注意的，代理服务器可能会将该
请求转发给另一个代理服务器，也可能会按照absoluteURI直接发送给相应的服务器。如我们需要通过代理服务器访问被墙掉的Facebook时，可以看到请求头如下：


![](http://leivli.duapp.com/wp-content/uploads/2010/11/13.png)


其它情况下，http请求头中的RequestURI将会使用一个资源相对地址abs_path,并加上一个Host头，供web服务器通过Host和RequestURI生成相应的URI，当我们输入的
url没有abs_path时，则传递的RequestURI中的abs_path将自动设置为'/'。如我们直接访问我们的百度时，请求头如下：


![](http://leivli.duapp.com/wp-content/uploads/2010/11/21.png)




结论：




当通过代理服务器访问某个URL时，发送的RequestURI必须时一个绝对URI，否则将发送一个相对资源地址，即http_url中的abs_path段(及后面的query)
