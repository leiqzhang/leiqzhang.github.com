---
comments: true
date: 2013-06-13 20:40:31
layout: post
title: 'HTTP access control (CORS) of Web.py'
categories:  [Virtualization]
tags:  [oVirt, CORS, OPTIONS, lighttpd, webpy]

---

## 问题描述

由于想使用gevent，当前请求结构为browser->lighttpd(portal)->web.py，其中lighttpd和web.py运行在同一台服务器上并监听不同的端口。portal上一些请求会直接发送给web.py，即web.py直接用作了REST的Gateway。

但当使用FF/Chrome在通过和web.py不同的domain向其发起POST请求返回一个JSON数据后，web.py收到的请求Method为__OPTIONS__而非__POST__，由于OPTIONS非web.py支持的Method,故会返回405，Method Not Found. Chrome/FF的Console中会有如下形式的错误：

{% codeblock %}
XMLHttpRequest Cannot load http://server:webpy_port/path. Origin is not allowed by Access-Control-Allow-Origin
{% endcodeblock %}

<!-- more -->

## 解决办法

由于portal的端口和web.py的端口不同，此问题实际上是__CORS__问题(Cross-site HTTP requests)，解决办法无非由如下几种：

1. portal上使用JSONP等通用的解决跨域问题的措施
2. web.py可正常对OPTIONS method进行响应，并按照HTTP规范返回相应的信息
3. 使用一个web容器将web.py包含起来，使用fastcgi方式调用
4. 将lighttpd完全作为一个反向代理，根据url转发请求到web.py，web.py不对外提供服务


第一种方法不加详述，可参考https://developers.google.com/web-toolkit/doc/2.1/tutorial/Xsite?hl=en#design

第二种方法需要修改web.py，基本的修改方法如下，主要是在request方法中对OPTIONS进行处理，并返回包含Access-Control-*的header：
{% codeblock  web/application.py.diff lang:diff %}
@@ -201,6 +201,12 @@
         if 'HTTP_CONTENT_TYPE' in env:
             env['CONTENT_TYPE'] = env.pop('HTTP_CONTENT_TYPE')
 
+        if method in ["OPTIONS"]:
+            data = ''
+            response = web.storage()
+            response.status = 200
+            response.headers = {"Access-Control-Allow-Methods":"POST", "Access-Control-Allow-Origin":"*"}
+            return response
         if method in ["POST", "PUT"]:
             data = data or ''
             import StringIO
{% endcodeblock %}

第三种方法不加详述，可google lighttpd+fastcgi+web.py，使用此方法的缺点是无法使用gevent的高性能请求队列了。

第四种方法需要修改lighttpd的配置文件，主要是启用mod_proxy模块，并在proxy模块的配置中转发请求：
{% codeblock conf.d/proxy.conf lang:conf %}
server.modules += ( "mod_proxy" )

$HTTP["url"] =~ ".*/webpy_api/.*" {
    proxy.server  = ( "" =>
        (( "host" => "0.0.0.0", "port" => webpy_port ))
    )
}
{% endcodeblock %} 

{% codeblock Append_after_lighttpd.conf lang:conf %}
include "conf.d/proxy.conf"
{% endcodeblock %}

同时需要将portal上直接发送到webpy的请求均发送到lighttpd上，这样就没有跨域的问题了

## TODO

webpy如何配置以只接受来自本机lighttpd的请求而拒绝来自外部的请求

## Ref

1. CORS: https://developer.mozilla.org/zh-CN/docs/HTTP/Access_control_CORS
