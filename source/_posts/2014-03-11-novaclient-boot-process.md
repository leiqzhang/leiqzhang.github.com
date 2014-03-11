# novaclient's boot process

## novaclient

novaclient是用于在CLI模式下与NOVA进行交互的工具， 其将NOVA的功能封装为多个命令，执行命令时的过程均为构造相应的REST请求和NOVA标准和扩展的REST API进行交互。

novaclient的可执行文件为nova, 可通过nova help命令来查看其支持的所有命令。每个命令执行时可通过添加--debug选项来将其执行过程中和NOVA及相关Project的REST API进行交互的过程均输出出来，便于分析。

nova的boot命令主要用于创建一个新的虚拟机(Instance), 其支持的命令行参数很多，这里以几个常见的命令行参数来调用boot，并就其执行流程进行分析。

## 环境信息

为了使得novaclient可以获得keystone的授权，一般会在环境中配置如下的RC文件，其中会包含admin tenant的相应信息。

本例的RC文件内容如下：

```
#!/bin/bash

export OS_AUTH_URL=http://{CONTROLLER_IP}:5000/v2.0 #这里的controller_ip是controller节点的IP

# With the addition of Keystone we have standardized on the term **tenant**
# as the entity that owns the resources.
export OS_TENANT_ID=507667477fd840e1a81c1ca7f6f54f69
export OS_TENANT_NAME="admin"

# In addition to the owning entity (tenant), openstack stores the entity
# performing the action as the **user**.
export OS_USERNAME="admin"

# With Keystone you pass the keystone password.
export OS_PASSWORD="admin"

```

## 命令和期望结果

针对如下命令的执行过程进行分析：

```
nova --debug boot --flavor m1.tiny --image cirros --swap 100 test2

```

其中m1.tiny flavor的swap大小配置为0，通过命令行指定其swap大小为100M，超过了flavor的配置，故此命令期望的结果是NOVA会返回错误。

## 执行流程分析

nova可执行文件的执行入口是novaclient.shell:main。其主要执行步骤如下：

* 首先进行Authorization
* 根据子命令名称(boot)，调用相应的方法
	* 构造启动参数
	* 

### Authorization

首先会进行authorization。从debug的输出也可以看到，在authorization时，会首先根据RC文件中的配置信息，调用novaclient.v1_1.client:authenticate,并最终向keystone发出如上请求获取token。

```
REQ: curl -i http://{CONTROLLER_IP}/v2.0/tokens -X POST -H "Content-Type: application/json" -H "Accept: application/json" -H "User-Agent: python-novaclient" -d '{"auth": {"passwordCredentials": {"username": "admin", "password": "admin"}, "tenantId": "507667477fd840e1a81c1ca7f6f54f69"}}'
```

最终的返回结果如下，返回结果中会包含token、tenant列表以及所有的endpoints等等(有省略）。

```
DEBUG (connectionpool:295) "POST /v2.0/tokens HTTP/1.1" 200 3302
RESP: [200] {'date': 'Tue, 11 Mar 2014 07:17:54 GMT', 'content-type': 'application/json', 'content-length': '3302', ''}
RESP BODY: {"access": {"token": {...}}, "serviceCatalog": [...], "user": {...}, "metadata": {...}}}
```

### 启动Instance

boot命令对应的方法为novaclient.v1_1.shell:do_boot，在此方法中会首先调用novaclient.v1_1.shell:_boot方法，来构造启动参数。


#### 构造启动参数(_boot 方法)

此方法负责初始化所有相关的启动参数，所支持的启动参数及其含义如下：

* name: instance name
* image: Image to boot with
* flavor: Flavor to boot onto
* meta: dict of arbitrary key/value metadata to store for this instance
* files: dict of files to overwrite on the instance upon boot. key: file name, value: file content
* reservation_id: a UUID for the set of servers being requested
* min_count
* max_count
* security_groups
* userdata: user data to pass to be exposed by the metadata server. file type object / string
* key_name: name of previously created keypair to inject into the instance
* availability_zone
* block_device_mapping
* block_device_mapping_v2
* nics: nics to be added to this instance
* scheduler_hints: arbitrary key-value pairs specified by the client to help boot an instance
* config_drive
* disk_config: control how the disk is partitioned when the server is created. AUTO / MANUAL
* others

对于本例执行的命令行，其涉及到的处理逻辑如下所述。

** 验证并获取Image和Flavor信息 **

首先根据参数验证给定的image和flavor是否合法，并获取image和flavor的详细信息。

对于Image的验证，是由novaclient.v1_1.images:ImageManager来负责的。在boot命令中--image可以给定image的uuid或名称。

对于给定uuid的情况，ImageManager:get会被调用，根据上一步骤keystone返回的glance的endpoint来构造类似"/images/{image_uuid}"的URL来访问glance验证此Image是否存在并获取Image的详细信息。

对于给定名称的情况，ImageManager.list会首先被调用来通过访问"/images"来获取image列表，然后从结果中确定给定Image的uuid，然后再通过ImageManager:get方法来获取Image的信息。 

对于flavor参数的验证和信息获取与之类似。

下面是获取给定Image信息时，涉及的API交互信息(有省略）：

```
REQ: curl -i http://{CONTROLLER_IP}:8774/v2/507667477fd840e1a81c1ca7f6f54f69/images -X GET -H "X-Auth-Project-Id: admin" -H "User-Agent: python-novaclient" -H "Accept: application/json" -H "X-Auth-Token: 1c8848a651a24e35ad74cea8eeba1cd3"

DEBUG (connectionpool:295) "GET /v2/507667477fd840e1a81c1ca7f6f54f69/images HTTP/1.1" 200 3724
RESP: [200] {'date': 'Tue, 11 Mar 2014 09:28:55 GMT', 'content-length': '3724', 'content-type': 'application/json', 'x-compute-request-id': 'req-fb8f0a9e-0caf-4e34-bbe6-abb0513ae2e2'}
RESP BODY: {"images": [..., {"id": "ad559bb4-31d5-469f-93b7-0a67546c4a96", "links": [{"href": "http://186.100.8.144:8774/v2/507667477fd840e1a81c1ca7f6f54f69/images/ad559bb4-31d5-469f-93b7-0a67546c4a96", "rel": "self"}, {"href": "http://186.100.8.144:8774/507667477fd840e1a81c1ca7f6f54f69/images/ad559bb4-31d5-469f-93b7-0a67546c4a96", "rel": "bookmark"}, {"href": "http://186.100.8.144:9292/507667477fd840e1a81c1ca7f6f54f69/images/ad559bb4-31d5-469f-93b7-0a67546c4a96", "type": "application/vnd.openstack.image", "rel": "alternate"}], "name": "cirros"}]}


REQ: curl -i http://{CONTROLLER_IP}:8774/v2/507667477fd840e1a81c1ca7f6f54f69/images/ad559bb4-31d5-469f-93b7-0a67546c4a96 -X GET -H "X-Auth-Project-Id: admin" -H "User-Agent: python-novaclient" -H "Accept: application/json" -H "X-Auth-Token: 1c8848a651a24e35ad74cea8eeba1cd3"

DEBUG (connectionpool:295) "GET /v2/507667477fd840e1a81c1ca7f6f54f69/images/ad559bb4-31d5-469f-93b7-0a67546c4a96 HTTP/1.1" 200 720
RESP: [200] {'date': 'Tue, 11 Mar 2014 09:28:55 GMT', 'content-length': '720', 'content-type': 'application/json', 'x-compute-request-id': 'req-c636e7d0-b74c-42d6-8635-e24076f9aeff'}
RESP BODY: {"image": {"status": "ACTIVE", "updated": "2014-02-26T08:18:39Z", "links": [...], "id": "ad559bb4-31d5-469f-93b7-0a67546c4a96", "OS-EXT-IMG-SIZE:size": 13147648, "name": "cirros", "created": "2014-02-26T08:03:29Z", "minDisk": 1, "progress": 100, "minRam": 512, "metadata": {}}}
```

** swap的处理 **

然后会处理block_device_mapping配置，本例中不涉及--block-device-mapping命令行参数，无影响。 

下面是处理bdm的代码，做参考：
```
    block_device_mapping = {}
    for bdm in args.block_device_mapping:
        device_name, mapping = bdm.split('=', 1)
        block_device_mapping[device_name] = mapping
```

本例中命令行添加了-swap参数，在后面调用的_parse_block_device_mapping_v2中会在启动参数中生成相应的block_device_mapping_v2信息。

打开_parse_block_device_mapping_v2方法，可以看到，其处理的命令行参数包括：
* --boot-volume
* --snapshot
* --block-device
* --ephemeral
* --swap

其中对于本例涉及的--swap，其处理逻辑如下：

```
    if args.swap:
        bdm_dict = {'source_type': 'blank', 'destination_type': 'local',
                    'boot_index': -1, 'delete_on_termination': True,
                    'guest_format': 'swap', 'volume_size': args.swap}
        bdm.append(bdm_dict)
```

即会生成相应的block_device_mapping_v2信息，并附加到启动参数中。

#### 设置ResourceUrl

do_boot方法,接着会调用novaclient.v1_1.servers.ServerManager:create方法，其接收的参数就是前文提到的各个启动参数。

由此方法的代码可知，如果启动参数包含block_device_mapping(_v2),则设置resource_url为/os-volumes_boot, 否则设置为/servers。并最终会用此resource_url为参数调用novaclient.v1_1.servers.ServerManager:_boot进行后续操作。

如前所述，在本例中，block_device_mapping_v2已经被设置，故resource_url为/os-volumes_boot

### 构造REST请求

在ServerManager:_boot方法中，其会构造相应的body，不同启动参数和body的内容对应关系如下：

* name: server.name
* image: getid => server.imageRef
* flavor: getid => server.flavorRef
* userdata: (read)+base64encode => server.user_data
* meta: server.metadata
* reservation_id: server.reservation_id
* key_name: server.key_name
* scheduler_hints: os:scheduler_hints
* config_drive: server.config_drive
* admin_pass: server.adminPass
* min_count: server.min_count
* max_count: server.max_count
* security_group: [{'name': sg} ... ] => server.security_group
* files: [{"path":filepath, “contents": base64encode(data)}, ...] => server.personality
* availability_zone: server.availability_zone
* block_device_mapping: server.block_device_mapping
* block_device_mapping_v2: server.block_device_mapping_v2
* nics: [{'uuid':get('net-id'), 'fixed_ip':get('v4-fixed-ip'),'port':get('port-id')},...]=> server.networks
* disk_config: server.OS-DCF:diskConfig

接着调用base.Manager:_create方法来发送POST请求并返回结果，对应如下debug信息：

```
REQ: curl -i http://{CONTROLLER_IP}:8774/v2/507667477fd840e1a81c1ca7f6f54f69/os-volumes_boot -X POST -H "X-Auth-Project-Id: admin" -H "User-Agent: python-novaclient" -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token: 7e1e6458f641473cb048c685e6795411" -d '{"server": {"name": "test2", "imageRef": "ad559bb4-31d5-469f-93b7-0a67546c4a96", "block_device_mapping_v2": [{"source_type": "image", "delete_on_termination": true, "boot_index": 0, "uuid": "ad559bb4-31d5-469f-93b7-0a67546c4a96", "destination_type": "local"}, {"guest_format": "swap", "boot_index": -1, "volume_size": "100", "source_type": "blank", "destination_type": "local", "delete_on_termination": true}], "flavorRef": "1", "max_count": 1, "min_count": 1}}'

DEBUG (connectionpool:295) "POST /v2/507667477fd840e1a81c1ca7f6f54f69/os-volumes_boot HTTP/1.1" 400 101
RESP: [400] {'date': 'Tue, 11 Mar 2014 13:35:35 GMT', 'content-length': '101', 'content-type': 'application/json; charset=UTF-8', 'x-compute-request-id': 'req-9628ca64-dc0f-4ce0-b4b8-9295c15f5995'}
RESP BODY: {"badRequest": {"message": "Swap drive requested is larger than instance type allows.", "code": 400}}

DEBUG (shell:740) Swap drive requested is larger than instance type allows. (HTTP 400) (Request-ID: req-9628ca64-dc0f-4ce0-b4b8-9295c15f5995)
```
如之前的期望结果，这里NOVA会返回相应的错误。


## 后续工作

对NOVA侧接收到此POST请求后，真正的内部处理逻辑进行进一步的分析。



































