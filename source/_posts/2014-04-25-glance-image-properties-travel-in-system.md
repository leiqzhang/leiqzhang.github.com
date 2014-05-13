---
comments: true
date: 2014-04-25 22:50:31
layout: post
title: 'Glance Image Properties Travel In OpenStack'
categories:  [OpenStack]
tags:  [Glance, Metadata, Cinder, Nova]

---

# Glance Image Properties在系统中的流转

## 背景

Glance提供了Image的发现、注册和获取等服务，Image除了默认的kernel_id、ramdisk_id、disk_format、container_format、min_ram、min_disk、base_image_ref等Properties外，还可以针对Image设置各种自定义的Properties。这些Properties主要用于标记Image的特征、使用时对环境的需求等等。

可以通过Glance提供的Update API对Image添加自定义的Properties，使用glanceclient时，相应的命令如下：

```
glance image-update {image_id} --property {key}={value}
```

由于这些Properties标识了Image对运行环境的一些需求，故当使用此Image创建虚拟机、卷以及后续对此虚拟机的迁移等操作均需要考虑到这些Properties，故OpenStack相关的Glance、Nova、Cinder等模块必然需要对这些Properties进行相应的存储和处理。本文基于Icehouse版本，对Image Properties在整个OpenStack系统中的流转过程进行分析。


## Glance

Glance对外提供的Image的Update API，支持对Image自定义Properties和更新。自定义的Properties会存储到数据库glance.image_properties表中，在API中当前不会对自定义Properties的取值合法性进行校验，完全根据请求参数设置。

## 从Glance Image创建Cinder Volume

从Glance Image创建Cinder Volume时，Cinder的Create API最终会通过cinder.volume.flows.manager.\create_volume:_capture_volume_image_metadata将Image部分默认Properties以及所有自定义的Properties保存到Cinder的volume_glance_metadata表中。

其中会保存下来的默认Properties由cinder.volume.flows.manager.\create_volume.IMAGE_ATTRIBURES决定，当前包含的内容为：

```
IMAGE_ATTRIBUTES = (
    'checksum',
    'container_format',
    'disk_format',
    'min_disk',
    'min_ram',
    'size',
)
```

Cinder对外提供的卷信息查询接口中，会以"volume_image_metadata" 形式将这些信息返回。

## Nova创建虚拟机

不论是从Glance Image直接创建虚拟机，还是从Volume创建虚拟机，Nova的Create API均会在如下调用路径中读取到相关的Image Properties：

```
#1. nova.compute.api.create
#2. nova.compute.api._create_instance

    if image_href:
        #对应于从Glance Image启动情况，直接从glance读取Image Properties
        image_id, boot_meta = self._get_image(context, image_href)
    else:
        image_id = None
        boot_meta = {}
        #对应于从Volume启动情况，此方法会读取volume对应的volume_image_metadata信息
        boot_meta['properties'] = \
            self._get_bdm_image_metadata(context,
                block_device_mapping, legacy_bdm)
```

之后，Nova会将这些Image Properties存入Nova数据库的instance_system_metadata表：

```
#3. nova.compute.api:_provision_instances
#4. nova.compute.api:create_db_entry_for_new_instance
#5. nova.compute.api:_populate_instance_for_create
#6. nova.compute.api.utils:get_system_metadata_from_image

def get_system_metadata_from_image(image_meta, flavor=None):
    system_meta = {}
    #这里的SM_IMAGE_PROP_PREFIX是"image_"
    prefix_format = SM_IMAGE_PROP_PREFIX + '%s'

    for key, value in image_meta.get('properties', {}).iteritems():
        new_value = unicode(value)[:255]
        system_meta[prefix_format % key] = new_value

    for key in SM_INHERITABLE_KEYS:
        value = image_meta.get(key)

        if key == 'min_disk' and flavor:
            if image_meta.get('disk_format') == 'vhd':
                value = flavor['root_gb']
            else:
                value = max(value, flavor['root_gb'])

        if value is None:
            continue

        system_meta[prefix_format % key] = value
    
    #最终所有的Properties以"image_xx"的形式保存到system_metadata中
    return system_meta

# PS. 可通过nova.compute.util:get_system_metadata_from_image方法从system_metadata中将image相关的属性重新取出.

```

然后，Image Properteis会一直传递到virt driver层，对于libvirt driver来说，会根据这些Properties生成相应的XML配置，并创建虚拟机。

## 总结

* Image的Properties包含默认和用户自定义两类。

* 从Image创建Volume时，Cinder中会通过volume_glance_metadata对Image Properties进行保存，并可通过QUERY API查询到这些信息

* 从Image/Volume创建虚拟机时，Nova中会通过Instance的system_metadata对Image Properties进行保存


## 遗留问题

* 存储在Nova中system_metadata的image Properties具体的使用场景还待进一步探究。

* 从Glance Image创建Cinder Volume后，此Volume再次Upload到Glance上成为Image时，新Image的Properties是否会继承volume的volume_glance_metadata

* Instance快照后保存在Glance上的Image，是否会继承相应的Properties
