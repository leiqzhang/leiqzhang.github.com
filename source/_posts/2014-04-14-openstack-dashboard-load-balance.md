# DashBoard的LoadBalance

## 问题描述

OpenStack的DashBoard对外提供Portal和noVNCProxy两个服务，使用Django框架实现。其中noVNCProxy还涉及到提供websocket服务器的能力。要做到DashBoard的LoadBalance，有如下几个问题需要考虑。

第一，DashBoard涉及到用户登陆和session管理，当使用多个DashBoard后端做LoadBalance时，需要保持多DashBoard后端session数据的一致性。可以通过将DashBoard所在主机添加到一个memcache集群中，并配置Django使用此memcache来作为session后端来完成这一点。当然，也可以通过由LB软件提供session sticky能力来完成这一点。所谓的session sticky，是指LB需要在第一次接收到请求并分发到后端时，记录其session信息，在后续收到同一个session的请求到来时，将请求转发到相同的后端上。作为L7 Level的LB软件，HAProxy具备session sticky的能力。可通过类似如下Backend配置来开启session sticky：
```
backend dashboard-backend
    mode http
    balance     roundrobin
    cookie SERVERID insert indirect nocache
    server  server01 controller01:80 check cookie server01
    server  server02 controller02:80 check cookie server02
    server  server03 controller03:80 check cookie server03
```
第二，考虑websocket，对LB软件有一定的挑战。首先，LB软件作为反向代理，会对HTTP连接进行监控，并关闭长时间空闲的HTTP连接，而websocket就是作为长连接存在的，故需要LB软件避免错误的关闭websocket；其次，websocket连接的建立，是首先通过HTTP协议进行握手，然后“Upgrade”而来的，这就需要LB软件可以从HTTP通信中识别到websocket请求，从而正确的对其进行处理；再次，LB软件需要能够检测各个websocket后端的健康情况，以进行LoadBalance。HAProxy对websocket提供了相应的支持，对于第一个问题，可通过设置“option http-server-close”来做到，对于第二个问题，HAProxy可以自动检测HTTP的“Upgrade”请求，从而可以识别到websocket连接请求，对于第三个问题，HAProxy本身没有提供支持，不过可以结果业务实际，设置HAProxy的httpchk值，以实现各websocket后端的健康检查。可参考 http://blog.exceliance.fr/2012/11/07/websockets-load-balancing-with-haproxy/ 中提到的设置方式。

第三， DashBoard需要通过public平面提供，从安全角度考虑，需要以HTTPS方式提供服务。而VNCProxy涉及到Websocket，一方面从安全角度考虑，Websocket通信也要采用Secure  Websocket的方式，另一方面，由于VNCProxy页面是作为iframe嵌入到DashBoard的，DashBoard本身是HTTPS方式，也要求了VNCProxy页面以及其发起的Websocket连接必须是HTTPS和Secure Websocket的。下面一节对此问题的解决进行详细说明。

## SSL反向代理

上述问题需要LB软件可提供对HTTPS和Secure Websocket的支持。当前HAProxy的稳定版本(1.4)尚不直接提供对HTTPS的支持，故这里采用业界比较通用的方式，即作为Backend的DashBoard提供不加密的HTTP和Websocket服务，然后在HAProxy之前再部署一个SSL反向代理(SSL Reverse Proxy)，由此反向代理负责将从frontend接收到的HTTPS信息解密为HTTP信息并传递到backend，将backend发送的HTTP回应加密为HTTPS消息回复到frontend。反向代理提供的这种能力也就是所谓的SSL Termination。

对于SSL反向代理来说，还有一个需要注意的问题就是对Backend发送的301和302 Redirect响应信息的处理。由于作为Backend的DashBoard提供的是HTTP服务，当其发送Redirect响应时，Header中的Location是一个HTTP地址，此时浏览器就会直接跳转到Location地址上，从而会访问HTTP服务。故要么SSL反向代理具有检测并更改Redirect Location的能力，要么配置HTTP服务器对Redirect进行相应的处理。

结合实际的系统部署，SSL反向代理需要作为HA组件(PaceMaker)管理的对象，最好其本身可对外暴露为一个系统服务，以更好的和HA组件进行集成。如前所述，DashBoard实际对外会暴露Portal和noVNCProxy两个服务，这两个服务是独立的后端，需要监听各自不同的端口。

## SSL反向代理软件对比

当前可提供HTTPS的SSL Termination能力且流行的反向代理软件有Pound、Stunnel和Stud等。其中Stunnel未部署成功，对于Pound和Stud进行了实际部署验证，两者的功能对比如下：

```
	Pound	Stud
HTTPS的SSL Termination	支持	支持
Websocket的SSL Termination	不支持	支持
HTTP Redirection	透明支持	不支持
二次开发	不需要
1.	支持通过一个服务，同时对不同的后端端口进行反向代理
2.	CentOS官方有RPM包	需要
1.	不支持一个服务，同时对不同的后端端口进行反向代理
2.	CentOS官方无RPM包
```

由于Pound不支持Secure Websocket的SSL Termination，故选择Stud。对于上表提到的Stud缺失的功能，进行如下处理：

1.	HTTP Redirection可交由后端HTTPD服务器处理，通过配置其Redirect规则和Header的Edit动作，可在HTTPD侧避免直接到HTTP的Redirection
2.	需要二次开发的工作包括RPM包的封装，系统服务的封装，基于Stud不支持一个进程同时对不同后端端口反向代理的能力，这里需要将其封装为两个系统服务，分别代理portal和novncproxy

## 部署结构

综上，DashBoard的LoadBalance涉及的组件结构图如下所示：

__TODO__


## 具体配置

```
dashboard配合stud：
RedirectMatch permanent ^/$ https://186.100.11.229/dashboard/
Header edit Location ^http://(.*)$ https://$1
```

```
#---------------------------------------------------------------------
# horizon frontend which proxys to the backends
#---------------------------------------------------------------------
frontend  dashboard controller-public:80
    default_backend             dashboard-backend

frontend novncproxy controller-public:6080
    acl is_websocket hdr(Upgrade) -i WebSocket
    use_backend websocket-backend if is_websocket
	
	default_backend	novncproxy-backend


#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend dashboard-backend
    mode http
    balance     roundrobin
    cookie SERVERID insert indirect nocache
    server  server01 controller01:80 check cookie server01
    server  server02 controller02:80 check cookie server02
    server  server03 controller03:80 check cookie server03

backend websocket-backend
	mode http
	option forwardfor
	option http-server-close
	option forceclose
	no option httpclose
	
	#TODO: Add httpchk and rr support for websocket-backend
	server app1 controller01:6080 weight 1 maxconn 8192 check

backend novncproxy-backend
	mode http
	balance     roundrobin
    cookie SERVERID insert indirect nocache
    server  server01 controller01:6080 check cookie server01
    server  server02 controller02:6080 check cookie server02
    server  server03 controller03:6080 check cookie server03
```

