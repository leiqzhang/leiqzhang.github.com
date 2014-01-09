---
comments: true
date: 2014-01-09 22:40:31
layout: post
title: 'NOVA API Service Initialization Process'
categories:  [OpenStack]
tags:  [Nova, WSGI, Paste, Deploy]

---

# NOVA-API服务启动流程 #

**前提**

1. 对Nova的整体结构已经有所理解
2. 基于stable/havana分支
3. 基于Redhat的RDO库进行的环境安装，基于CentOS 6.4

**内容**

1. openstack-nova-api服务启动流程
2. Paste、Deploy、WSGI等相关知识

## 执行结果 ##

启动API服务时的命令为：

```
service openstack-nova-api start
```

启动成功后，查看系统进程，发现实际执行结果为一个nova-api父进程，同时其有三个nova-api子进程：

	
```
nova 3438 S 0:07 /usr/bin/python /usr/bin/nova-api --logfile /var/log/nova/api.log
nova 3446 S 0:00 \_ /usr/bin/python /usr/bin/nova-api --logfile /var/log/nova/api.log
nova 3447 S 0:00 \_ /usr/bin/python /usr/bin/nova-api --logfile /var/log/nova/api.log
nova 3448 S 0:00 \_ /usr/bin/python /usr/bin/nova-api --logfile /var/log/nova/api.log
```

查看监听的端口，发现父进程nova-api同时监听了三个端口：

```
tcp        0      0 0.0.0.0:8773                0.0.0.0:*                   LISTEN      3438/python         
tcp        0      0 0.0.0.0:8774                0.0.0.0:*                   LISTEN      3438/python         
tcp        0      0 0.0.0.0:8775                0.0.0.0:*                   LISTEN      3438/python
```

## 代码路径 ##

### 主要路径 ###

服务启动脚本为/etc/init.d/openstack-nova-api，查看此脚本发现在启动服务时实际执行的是/usr/bin/nova-api,而/usr/bin/nova-api是一个python脚本，实际执行的是nova.cmd.api.main

nova.cmd.api.main的主要代码如下：

```python
    launcher = service.process_launcher()
    for api in CONF.enabled_apis:
        should_use_ssl = api in CONF.enabled_ssl_apis
        if api == 'ec2':
            server = service.WSGIService(api, use_ssl=should_use_ssl,
                                         max_url_len=16384)
        else:
            server = service.WSGIService(api, use_ssl=should_use_ssl)
        launcher.launch_service(server, workers=server.workers or 1)
    launcher.wait()
```

首先生成一个nova.common.service.Processlauncher类型的launcher, 接着会根据配置中的enabled_apis列表值，依次生成相应的WSGIService:

```
server = service.WSGIService(...)
```

然后通过launcher的launch_service方法，将上述的WSGIService和workers数量作为参数传入。

在luanch_service方法中，会根据传入的workers参数fork出相应个数的子进程，并在各个子进程中调用对应WSGIService的start方法，开始对外服务。

### 涉及的配置 ###

根据上述的主要执行路径，查看配置文件/etc/nova/nova.conf中的相关配置如下：

```
	# a list of APIs to enable by default (list value)
	#enabled_apis=ec2,osapi_compute,metadata          
```

以及后面依次的ec2、osapi_compute和metadata相应的如下几项配置：

```
	# IP address for EC2 API to listen (string value)
	#ec2_listen=0.0.0.0

	# port for ec2 api to listen (integer value)
	#ec2_listen_port=8773

	# Number of workers for EC2 API service (integer value)
	#ec2_workers=<None>
```

由此可见，在启动nova-api服务时，会同时启动ec2、osapi_compute和metadata三个API服务，各服务均有相应的监听地址和监听端口以及workers数量配置。

结合执行结果中看到的端口监听中看到的三个端口均为父进程进行监听，应该是父进程会对请求进行转发，由于nova-api是无状态的，当具有多个worker时（也就会被launcher派生出多个子进程），父进程应该还会有LB的逻辑，这一点可后续再深究和验证。

从日志文件/var/log/nova/api.log中，也可以对上述过程进行验证：

```
	--
	2014-01-08 18:41:18.091 3438 INFO nova.wsgi [-] ec2 listening on 0.0.0.0:8773
	2014-01-08 18:41:18.091 3438 INFO nova.openstack.common.service [-] Starting 1 workers
	2014-01-08 18:41:18.093 3438 INFO nova.openstack.common.service [-] Started child 3446
	--
	2014-01-08 18:41:18.793 3438 INFO nova.wsgi [-] osapi_compute listening on 0.0.0.0:8774
	2014-01-08 18:41:18.793 3438 INFO nova.openstack.common.service [-] Starting 1 workers
	2014-01-08 18:41:18.795 3438 INFO nova.openstack.common.service [-] Started child 3447
	--
	2014-01-08 18:41:18.802 3438 INFO nova.wsgi [-] metadata listening on 0.0.0.0:8775
	2014-01-08 18:41:18.802 3438 INFO nova.openstack.common.service [-] Starting 1 workers
	2014-01-08 18:41:18.804 3438 INFO nova.openstack.common.service [-] Started child 3448
```

### WSGIService ###

所谓的WSGI，可参考**参考资料**中的内容。

简单来说就是分为Server和APP两层，Server负责接受Request，设置相应的Environment，并转发到相应的APP进行处理，并返回Response。

如前所述，在服务启动过程中，主要涉及WSGIService的初始化(__init__)和启动(start)方法，下面分别进行分析。

**__init__方法**

方法主体如下：

```
		self.name = name
        self.manager = self._get_manager()
        self.loader = loader or wsgi.Loader()
        self.app = self.loader.load_app(name)
        self.host = getattr(CONF, '%s_listen' % name, "0.0.0.0")
        self.port = getattr(CONF, '%s_listen_port' % name, 0)
        self.workers = getattr(CONF, '%s_workers' % name, None)
        self.use_ssl = use_ssl
        self.server = wsgi.Server(name,
                                  self.app,
                                  host=self.host,
                                  port=self.port,
                                  use_ssl=self.use_ssl,
                                  max_url_len=max_url_len)
        # Pull back actual port used
        self.port = self.server.port
        self.backdoor_port = None
```

其中name就是前文提到的api_name,也即ec2、osapi_compute或metadata。

第2行的_get_manager方法，会从配置中寻找{self.name}_manager对应的类，并加载进来。关于OpenStack Nova中划分的Manager的角色和作用可参考 [这里](http://docs.openstack.org/developer/nova/devref/services.html)

查看nova配置文件(/etc/nova/nova.conf)，对于ec2、osapi_compute和metadata，默认情况下只有metadata_manager:

```
	# OpenStack metadata service manager (string value)
	#metadata_manager=nova.api.manager.MetadataManager 	
```

后面的逻辑，首先会通过wsgiLoader来load_app，这里涉及到Paste和Deploy的相关处理。参考最后一节。

最后会使用相应的参数初始化一个wsgiServer, 此Server对应的app就是上面使用wsgiLoader加载进来的APP。

**start方法**

主要逻辑是调用了上述wsgiServer的start方法，开始对外服务。对于metadata API来说，还需要其manager介入在start前后进行相应的处理。

```
		if self.manager:
            self.manager.init_host()
            self.manager.pre_start_hook()
        if self.backdoor_port is not None:
            self.manager.backdoor_port = self.backdoor_port
        self.server.start()
        if self.manager:
            self.manager.post_start_hook()
```

###Paste & Deploy###

关于Paste和Deploy可查看**参考资料**部分中相应的内容。

简单来说，Paste主要提供的是WSGI middleware，各middleware对WSGI的server来说是一个APP，但对于WSGI的APP来说是一个Server，且可以将多个middleware叠加，从而可实现类似Linux中pipeline的效果。在pipeline中，可以实现诸如压缩、解压缩、加密、解密、鉴权等各种预处理和后处理操作。Deploy是根据paste配置文件来发现和加载WSGI APP和Middleware的模块。

这里针对上节中的wsgiLoader进行详细分析

```
        self.loader = loader or wsgi.Loader()
        self.app = self.loader.load_app(name)
```

第一行初始化Loader，期间会根据配置项api_paste_config来locate到相应的paste配置文件，默认是/etc/nova/api-paste.ini。 

第二行的load_app(name),最终会使用Deploy模块加载name对应的APP，这里的name就是上述的ec2、osapi_compute或metadata。

在/etc/nova/api-paste.ini中，针对上述的三种name，均会有对应的[composite:{name}] section，并根据Paste的规则，配置相应的pipeline。其中osapi_compute的部分默认配置如下：

```
	[composite:osapi_compute]
	use = call:nova.api.openstack.urlmap:urlmap_factory
	/: oscomputeversions
	/v1.1: openstack_compute_api_v2
	/v2: openstack_compute_api_v2
	/v3: openstack_compute_api_v3

	[composite:openstack_compute_api_v2]
	use = call:nova.api.auth:pipeline_factory
	noauth = faultwrap sizelimit noauth ratelimit osapi_compute_app_v2
	keystone = faultwrap sizelimit authtoken keystonecontext ratelimit osapi_compute_app_v2
	keystone_nolimit = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v2

	[composite:openstack_compute_api_v3]
	use = call:nova.api.auth:pipeline_factory
	noauth = faultwrap sizelimit noauth_v3 ratelimit_v3 osapi_compute_app_v3
	keystone = faultwrap sizelimit authtoken keystonecontext ratelimit_v3 osapi_compute_app_v3
	keystone_nolimit = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v3

	[filter:faultwrap]
	paste.filter_factory = nova.api.openstack:FaultWrapper.factory

	[filter:noauth]
	...
```

在配置OpenStack环境时，我们一般均会修改此paste配置文件，增加或修改如下authtoken的filter的内容。如上述composite:osapi_compute的内容，各API的keystone argument中指定的filter列表中均包含了下面的keystonecontext和authtoken。这样就将keystone的认证过程通过filter的方式附加到了API请求处理的pipeline中了。

```
	[filter:keystonecontext]
	paste.filter_factory = nova.api.auth:NovaKeystoneContext.factory

	[filter:authtoken]
	paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
	auth_host = controller
	auth_port = 35357
	auth_protocol = http
	auth_uri = http://controller:5000/v2.0
	admin_user = nova
	admin_tenant_name = service
	admin_password = nova
    ...
```

## 参考资料 ##

* [Nova的Service、Manager和Driver](http://docs.openstack.org/developer/nova/devref/services.html)
* [Python Paste: WSGI middle and pipeline](http://en.wikipedia.org/wiki/Python_Paste)
* [Python Deploy: Sample Project's Explannation](http://docs.pylonsproject.org/projects/pyramid/en/1.1-branch/narr/startup.html)
* [Python Deploy: Offical Document](http://pythonpaste.org/deploy/)
* [Python Deploy: Related Document](http://blog.csdn.net/sonicatnoc/article/details/6539716)
* [Python WebOb](http://docs.webob.org)
