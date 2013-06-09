---
comments: true
date: 2010-11-15 17:17:35
layout: post
slug: php_input_post
title: php://input和$_POST
wordpress_id: 50
categories:
- PHP
tags: [php, post, soap]

---

一个soap_server的实现框架代码如下：

[php]
$wsdlFile = realpath(dirname(__FILE__).'/../wsdl/service.wsdl');
$actor = new soapServer($wsdlFile); //初始化soapServer
$actor->setClass('ServiceInterface');//导入方法
$actor->handle(file_get_contents('php://input')); //注意：这里使用了php://input
[/php]

**一个soap调用的数据是通过POST的方式发送给服务器的，那么上面的处理中为什么要使用php://input,是否可以使用$_POST？
**

一个典型的soap调用框架代码如下：

[php]
$client = new SoapClient(SOAP_SERVICEURL,array('trace'=>true));
//使用trace选项，可以保证后面的__getLastRequest、__getLastResponse等方法可以读取到相应的请求/响应
$ret = $client->getServiceName('serviceStr');
echo htmlspecialchars($client->__getLastRequestHeaders());
 echo "<br/>";
 echo htmlspecialchars($client->__getLastRequest());
 echo "<br/>";
 var_dump($ret);
 echo "<br/>";
 echo htmlspecialchars($client->__getLastResponseHeaders());
 echo "<br/>";
 echo htmlspecialchars($client->__getLastResponse());
 echo "<br/>";
[/php]

以下是运行这段client后的输出：

[code]
POST **** HTTP/1.1 Host: [*****] Connection: Keep-Alive User-Agent: PHP-SOAP/5.2.10 Content-Type: text/xml;
charset=utf-8 SOAPAction: "urn:*******#getServiceName" Content-Length: ***

<?xml version="1.0" encoding="UTF-8"?> <SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
xmlns:ns1="urn:****" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/" SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
<SOAP-ENV:Body><ns1:getServiceName><servicename xsi:type="xsd:string">serviceStr</servicename></ns1:getServiceName></SOAP-ENV:Body>
</SOAP-ENV:Envelope>

NULL

HTTP/1.1 200 OK Date: Mon, 15 Nov 2010 08:19:16 GMT Server: Apache/2.0.63 (Unix) PHP/5.2.10
X-Powered-By: PHP/5.2.10 Set-Cookie: *****; path=/; domain=*** Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0 Pragma: no-cache Content-Length: 521
Keep-Alive: timeout=15, max=100 Connection: Keep-Alive Content-Type: text/xml; charset=utf-8

<?xml version="1.0" encoding="UTF-8"?> <SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
xmlns:ns1="urn:AmsInterfaceswsdl" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema"
xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/" SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"><SOAP-ENV:Body>
<ns1:getServiceNameResponse><return xsi:nil="true"/></ns1:getServiceNameResponse></SOAP-ENV:Body></SOAP-ENV:Envelope>
[/code]

由请求header可以看到，soap虽然使用的post的方式传输请求串，但是传递的数据是一个xml，content-type也是“text/xml”，而对于php://input,[官方文档](http://php.net/manual/en/wrappers.php.php)给的是这样的说明:


php://input allows you to read raw data from the request body. In case of POST requests, it preferrable to [$HTTP_RAW_POST_DATA](http://www.php.net/manual/en/reserved.variables.httprawpostdata.php) as it does not depend on special php.ini directives. Moreover, for those cases where [$HTTP_RAW_POST_DATA](http://www.php.net/manual/en/reserved.variables.httprawpostdata.php) is not populated by default, it is a potentially less memory intensive alternative to activating [always_populate_raw_post_data](http://www.php.net/manual/en/ini.core.php#ini.always-populate-raw-post-data). php://input is not available with _enctype="multipart/form-data"_.





> **Note**:  A stream opened with php://input can only be read once; the stream does not support seek operations. However, depending on the SAPI implementation, it may be possible to open another php://input stream and restart reading. This is only possible if the request body data has been saved. Typically, this is the case for POST requests, but not other request methods, such as PUT or PROPFIND. 





也就是说php://input可以用来读取http post的raw request data。对于一般的表单post，http头中设置的contentType是'application/x-form-www-urlencoded'(无文件上传)或'multipart/form-data'(一般用于文件上传),这两种情况下，post的数据会被设置到$_POST中（第一种情况下，php://input也会被设置，第二种情况下，php://input则不会被设置），而对于text/xml这种类型，post数据并不会被设置到$_POST中，具体的post、get和php://input的实验可参考这篇[博文](http://tuzwu.javaeye.com/blog/798978)。

这里只摘抄其测试结论：

1，Coentent-Type仅在取值为application/x-www-data-urlencoded和multipart/form-data两种情况下，PHP才会将http请求数据包中相应的数据填入全局变量$_POST
2，PHP不能识别的Content-Type类型的时候，会将http请求包中相应的数据填入变量$HTTP_RAW_POST_DATA
3,  只有Coentent-Type不为multipart/form-data的时候，PHP不会将http请求数据包中的相应数据填入php://input，否则其它情况都会。填入的长度，由Coentent-Length指定。
4，只有Content-Type为application/x-www-data-urlencoded时，php://input数据才跟$_POST数据相一致。
5，php://input数据总是跟$HTTP_RAW_POST_DATA相同，但是php://input比$HTTP_RAW_POST_DATA更凑效，且不需要特殊设置php.ini
6，PHP会将PATH字段的query_path部分，填入全局变量$_GET。通常情况下，GET方法提交的http请求，body为空。
