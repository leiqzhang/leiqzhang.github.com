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

## 从Glance Image启动虚拟机

从Glance Image启动虚拟机时，Nova的Create API会通过如下调用路径来将Image的Properties存入Nova数据库的instance_system_metadata表：

```
#1. nova.compute.api:create
#2. nova.compute.api:_create_instance
#3. nova.compute.api:_provision_instances
#4. nova.compute.api:create_db_entry_for_new_instance
#5. nova.compute.api:_populate_instance_for_create

	# Store image properties so we can use them later
    # (for notifications, etc).  Only store what we can.
    if not instance.obj_attr_is_set('system_metadata'):
        instance.system_metadata = {}
    # Make sure we have the dict form that we need for instance_update.
    instance['system_metadata'] = utils.instance_sys_meta(instance)

    #核心处在这一行
    system_meta = utils.get_system_metadata_from_image(
        image, instance_type)

    # In case we couldn't find any suitable base_image
    system_meta.setdefault('image_base_image_ref', instance['image_ref'])

    instance['system_metadata'].update(system_meta)

#6. nova.compute.api.utils:get_system_metadata_from_image

def get_system_metadata_from_image(image_meta, flavor=None):
    system_meta = {}
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

    return system_meta

```



