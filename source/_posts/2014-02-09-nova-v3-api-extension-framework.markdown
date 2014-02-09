---
comments: true
date: 2014-02-09 22:55:31
layout: post
title: 'NOVA V3 API Extension Framework'
categories:  [OpenStack]
tags:  [Nova, Stevedore]

---

# NOVA V3 API Extension Framework #

**背景**

1. 基于stable/havana分支
3. 基于CentOS 6.4，以Redhat的RDO库进行的环境安装

**内容**

1. V2扩展机制存在的问题
2. Nova API V3中Plugin的实现机制和现状
3. 总结

## V2的扩展机制存在的问题 ##

参考V3 API framework的Spec，当前V2 API存在的问题如下：

The development of the Nova v3 API will give us the opportunity to rework the extension framework. The current framework suffers from:

* extension specific code required to exist in core code for specific extensions to work
	* eg nova/api/openstack/compute/servers.py:Controller:create where there are hard coded references to parameters specific to extensions to pass kwargs to the server create call
* issues around efficiency with extensions each doing their own db lookup loops
	* https://bugs.launchpad.net/nova/+bug/1160487
* Can't have CamelCase class names for extension classes due to extension loading mechanism

这里主要展开说明前两点。

<!--more-->

### Extension需要涉及Core Code的变动 ###

第一个问题主要指当前的extension机制中，增加一个contrib后，往往需要在所谓的“core code”中添加必要的参数处理来将extension中引入的参数传入到内部的API中。**需要注意的是**这里的core code指的是nova.api.openstack.compute.*,即仅仅指的是API层次中核心资源相关的API Handlers，并非指的是各个Module内部的API/RPCAPI->Manager->Driver的代码层次结构中的代码。

以当前对核心资源“虚拟机”的create方法扩展来看，在api.openstack.compute.servers.py:create中存在大量类似如下的逻辑：

```
		# optional openstack extensions:
        key_name = None
        if self.ext_mgr.is_loaded('os-keypairs'):
            key_name = server_dict.get('key_name')

        user_data = None
        if self.ext_mgr.is_loaded('os-user-data'):
            user_data = server_dict.get('user_data')
        self._validate_user_data(user_data)
```

如上代码中的os-keypairs和os-user-data就是两个extensions。当添加这俩extension后，就需要在servers.py的create方法中进行检测和参数处理，以便最终将获取到的key_name和user_data等参数传递到内部的compute_api:create方法中。

### 性能问题 ###

关于第二个问题，在上文的bugs链接中已经描述的比较清楚，以servers为例，问题主要是指在GET等方法中，各个extension的处理逻辑中往往需要往返回的server信息中追加extension相关的信息，如disk_config扩展需要追加存储在DB中的disk config相关的信息等，故在各个extension中均会分别进行数据库查询操作，从而导致性能问题。

不过从目前HAVANA的代码来看，此问题已经通过一种方法缓解了，但从commit来看，和V3 API并不属于同一系列的提交。以servers为例，此问题现在的解决办法是，在core API处理时，就将可能需要从db中读取出来的servers相关的信息全部一次性读取出来并放入Cache中的Instance对象，在各个extension处理时只需要从Cache中取出所需的属性即可。在core API中一次性读出的信息除了instances表中一些extension添加的字段外，还会读出关联的instance_metadata、instance_system_metadata等多个表的信息。具体可参考nova.db.sqlalchemy.api:instance_get等相关的方法实现。


## Nova API V3中Plugin的实现机制和现状 ##

### 实现机制 ###

API V3中的Plugin的实现机制利用的是和Ceilometer一样的Stevedore，此模块主要是对python setuptools中的Entry Points机制进行了封装。

### Setuptools Entry Points ###

Python setuptools的Entry Points机制主要用途有两个：Automatic Script Creation和Dynamic Discovery of Services and Plugins。

Automatic Script Creation主要功能是封装了平台的差异，在不同平台上生成Python Package的入口脚本。如RDO库中的/usr/bin目录下的nova-api、nova-scheduler等入口脚本均是通过此机制生成的。

Dynamic Discovery of Services and Plugins主要是指本Package通过Entry Point声明自身的“扩展点”，然后第三方Package或者本Package的各个Module可以声明对此“扩展点”的实现。在本Package中，可以通过某种方式获取相应“扩展点”当前所有的实现，并进行相应的处理。

Entry Point的声明和注册均是在各个Package的setup.cfg配置文件或者setup.py:setup method中完成的。以当前的HAVANA为例，在nova的setup.cfg中可以看到有如下配置：

```
	[entry_points]
	#...

	console_scripts =
    	nova-all = nova.cmd.all:main
    	nova-api = nova.cmd.api:main
		#....
	nova.api.v3.extensions =
    	admin_actions = nova.api.openstack.compute.plugins.v3.admin_actions:AdminActions
    	admin_password = nova.api.openstack.compute.plugins.v3.admin_password:AdminPassword
    	agents = nova.api.openstack.compute.plugins.v3.agents:Agents
		#...
	nova.api.v3.extensions.server.create =
    	availability_zone = nova.api.openstack.compute.plugins.v3.availability_zone:AvailabilityZone
    	block_device_mapping = nova.api.openstack.compute.plugins.v3.block_device_mapping:BlockDeviceMapping
		#...
	#...
```

其中的"console_scripts"是一个特殊的entry_point，触发的就是上述的Automatic Script Creation，在执行setup.py install时，就会根据这里的配置在/usr/bin目录下生成相应内容的nova-all和nova-api等文件。

后面的nova.api.v3.extensions和nova.api.v3.extensions.server.create就是此Package支持的“扩展点”,其Value中对应的list就是此扩展点对应的实现。


### Python stevedore ###

Python setuptools的pkg_resources包中，有多种utils方法，可以获取指定entry_point对应的所有的实现。Stevedore就是对这些方法进行了封装，并对外提供了几种扩展机制，具体可参考其官方网站。这里只简单说明下几种：

* DriverManager: 对应的是Driver形式的扩展机制，即使一个EntryPoint有多个实现，系统只能使用其中一个
* HookManager：对应的是Hook机制，会遍历EntryPoint的所有实现，并进行invoke
* EnabledExtensionManager：在load所有实现时，增加check_func的filter机制

在ExtensionManager中，比较重要的一个方法是map(func, *args, *kwds),此方法会遍历所有的extensions，并依次调用func方法。


### NOVA中的现状 ###

NOVA中APIv3相关的代码在nova.api.openstack.compute.pulgins.v3层次下。下面对V3 API Extension Framework的工作机制做简单的分析：

** v3 API入口 **

在etc.nova.api-paste.ini中，通过配置定义了/v3的URL入口，并最终指向的app_factory是nova.api.openstack.compute:APIRouterV3.factory。

** 加载所有Extensions **

在APIRouteV3中，会通过初始化self.api_extension_manager为stevedore.enabled.EnabledExtensionManager来加载所有声明为nova.api.v3.extensions的Extensions，然后会对各extensions进行相应的map调用来进行类似V2 API中的资源和Action的扩展，并会检测配置的Core API是否全部成功load：

```
    API_EXTENSION_NAMESPACE = 'nova.api.v3.extensions'
    
    # 初始化api_extension_manager并load所有的API_EXTENSION_NAMESPACE对应的extensions。
    # 如前文所述，这里load就是nova项目中setup.cfg中配置的此namespace对应的所有extensions。当然如果有第三方的应用也声明了此namespace的extension，这里也会一并load进来。
    # 其中的check_func为check_load_extension,其所作的工作主要包括：
    # a) 检测是否为相应的V3APIExtensionBase类型
    # b) extension的黑白名单过滤和验证
    self.api_extension_manager = stevedore.enabled.EnabledExtensionManager(
            namespace=self.API_EXTENSION_NAMESPACE,
            check_func=_check_load_extension,
            invoke_on_load=True,
            invoke_kwds={"extension_info": self.loaded_extension_info})
            
    #…

    # NOTE(cyeoh) Core API support is rewritten as extensions
    # but conceptually still have core
    if list(self.api_extension_manager):
        # 对所有的extension依次进行_register_resources和_register_controllers的方法调用
        self.api_extension_manager.map(self._register_resources,
                                           mapper=mapper)
            self.api_extension_manager.map(self._register_controllers)

        #检测配置的Core Extension是否全部成功加载
        missing_core_extensions = self.get_missing_core_extensions(
            self.loaded_extension_info.get_extensions().keys())
        if not self.init_only and missing_core_extensions:
            LOG.critical(_("Missing core API extensions: %s"),
                         missing_core_extensions)
            raise exception.CoreAPIMissing(
                missing_apis=missing_core_extensions)
```

其中上面代码中的_register_resources和_register_controllers会分别调用extension的get_resources和get_controller_extensions方法，其所作的工作分别对应于V2 API中在contrib中所作的新增资源和扩展现有资源以添加action。这里就不详细展开。

这里需要注意的是，V2 API中会区分Core API和Extension API。在V3 API中，所有的API均作为extension的形式提供。如V2 API中的servers是Core API，在V3 API中也是一个普通的extension，此extension会在get_resources方法中返回servers资源对象，而get_controller_extensions方法会返回空list：

```
    #nova.api.openstack.compute.pulgins.v3.servers:Servers
    
    class Servers(extensions.V3APIExtensionBase):
    """Servers."""

    name = "Servers"
    alias = "servers"
    namespace = "http://docs.openstack.org/compute/core/servers/v3"
    version = 1

    def get_resources(self):
        member_actions = {'action': 'POST'}
        collection_actions = {'detail': 'GET'}
        resources = [
            extensions.ResourceExtension(
                'servers',
                ServersController(extension_info=self.extension_info),
                member_name='server', collection_actions=collection_actions,
                member_actions=member_actions)]

        return resources

    def get_controller_extensions(self):
        return []

```

而当disk_config等需要对servers资源扩展action时，其所需做的就是在get_controller_extensions中返回对servers资源的扩展：

```
    #nova.api.openstack.compute.pulgins.v3.disk_config
class DiskConfig(extensions.V3APIExtensionBase):
    """Disk Management Extension."""

    name = "DiskConfig"
    alias = ALIAS
    namespace = XMLNS_DCF
    version = 1

    def get_controller_extensions(self):
        servers_extension = extensions.ControllerExtension(
            self, 'servers', ServerDiskConfigController())

        return [servers_extension]
        
    #...
```

但是V3 API中可以通过配置约定了其中部分的API是作为“Core Extension”的，对于Core Extension，如上面代码所示，会在初始化时检测是否均被正常加载。

** 避免各Extension之间的代码耦合 **

上述两个步骤，只是换了中方式完成了V2 API中的Extension功能，下面看V3 API中如何解决了前文所述的Extension需要涉及Core API代码改动的问题。

V3中，主要是利用了Stevedore避免了各Extension之间的代码耦合，从而避免了上述问题。

还是以当前对核心资源“虚拟机”的create方法扩展来看，在servers资源对应的Controller nova.api.openstack.compute.pulgins.v3.servers:ServersController中，将create作为了一个extension的namespace，并会在初始化时就load所有的extensions：

```
    EXTENSION_CREATE_NAMESPACE = 'nova.api.v3.extensions.server.create'

    #…
    # Look for implmentation of extension point of server creation
    self.create_extension_manager = \
      stevedore.enabled.EnabledExtensionManager(
          namespace=self.EXTENSION_CREATE_NAMESPACE,
          check_func=_check_load_extension('server_create'),
          invoke_on_load=True,
          invoke_kwds={"extension_info": self.extension_info},
          propagate_map_exceptions=True)
```

然后在create方法中，会通过map方法依次调用各个Extensions的server_create方法：

```
    #create method:
    if list(self.create_extension_manager):
        self.create_extension_manager.map(self._create_extension_point,
                                              server_dict, create_kwargs)
                                              

    #….
    def _create_extension_point(self, ext, server_dict, create_kwargs):
        handler = ext.obj
        LOG.debug(_("Running _create_extension_point for %s"), ext.obj)

        handler.server_create(server_dict, create_kwargs) 
                                              
```

在nova的setup.cfg中也配置了此namespace对应的所有的extensions，比如nova.api.openstack.compute.plugins.v3.disk_config:DiskConfig，其server_create方法如下所示，会更新ServersController:create方法中传入的create_kwargs字典

```
    def server_create(self, server_dict, create_kwargs):
        create_kwargs['auto_disk_config'] = server_dict.get(
            'auto_disk_config')
```

最终，ServersController:create方法会以如下方式调用内部API来创建VM，即直接使用被各个extensions更新过的create_kwargs，而不用感知其实际的内容。

```
            (instances, resv_id) = self.compute_api.create(context,
                            inst_type,
                            image_uuid,
                            display_name=name,
                            display_description=name,
                            metadata=server_dict.get('metadata', {}),
                            access_ip_v4=access_ip_v4,
                            access_ip_v6=access_ip_v6,
                            admin_password=password,
                            requested_networks=requested_networks,
                            **create_kwargs)
```

## 总结 ##

由上面的分析可知，如上所有的所谓的Extension，最终均依赖的还是一个内部的API，如果此API本身不具有扩展性，那么如上所有的Extension，均只能在此内部API支持的功能的基础上进行发挥。 比如文中提到的创建VM的内部create API，此API的参数是固定的：

```
    def create(self, context, instance_type,
               image_href, kernel_id=None, ramdisk_id=None,
               min_count=None, max_count=None,
               display_name=None, display_description=None,
               key_name=None, key_data=None, security_group=None,
               availability_zone=None, user_data=None, metadata=None,
               injected_files=None, admin_password=None,
               block_device_mapping=None, access_ip_v4=None,
               access_ip_v6=None, requested_networks=None, config_drive=None,
               auto_disk_config=None, scheduler_hints=None, legacy_bdm=True):
               
```

故上述各种Extension无论如何进行扩展，均不能做到使传入到内部底层的参数变多，V3 API只是使得调用此内部API时，不需要由单个所谓的Core Extension来做逻辑处理和参数封装了，而是通过Entry Point的机制巧妙的由各个Extension自行封装参数而已。

而且不论是V2还是V3 Extension机制，均是在Public API层做的文章，如果要实现一个从Public API层到底层libvirt/hypervisor的新功能，不可避免是需要新增内部API的。故这里的Extension和传统我们理解的Plugin根本不是一回事。

## 参考资料 ##

* [Nova API V3 Extension Framework](https://etherpad.openstack.org/p/NovaAPIExtensionFramework)
* [Nova API V3 Extension Framework BP](https://blueprints.launchpad.net/nova/+spec/v3-api-extension-framework)
* [Nova API V2 Extension Problem about db lookup loops](https://bugs.launchpad.net/nova/+bug/1160487)
* [Python Stevedore](http://stevedore.readthedocs.org/en/latest/)
* [Python SetupTools EntryPoints and Plugins](http://pythonhosted.org/setuptools/setuptools.html#dynamic-discovery-of-services-and-plugins)
* [Python Discovery and Resource Access using pkg_resources](http://pythonhosted.org/setuptools/pkg_resources.html)
