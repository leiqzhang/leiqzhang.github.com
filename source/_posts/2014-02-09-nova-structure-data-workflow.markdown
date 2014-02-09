---
comments: true
date: 2014-02-09 23:40:31
layout: post
title: 'NOVA structure data workflow'
categories:  [OpenStack]
tags:  [Nova]

---

# NOVA代码总体结构和数据流向总结 #

**前提**

1. 对Nova的整体结构已经有所理解
2. 基于stable/havana分支
3. 基于Redhat的RDO库进行的环境安装，基于CentOS 6.4
4. 主机名为controller

**内容**

1. Nova代码总体组织结构
2. Nova对外提供的服务中，一般的执行路径

**目的**

1. 为后续新增代码时，提供所需遵循的代码层次结构提供参考

**注意**

尚不完整，需要进一步补充和完善

## Nova代码总体结构 ##

```
├── etc
│   └── nova - 相关配置文件
├── nova
│   ├── api - Nova 对外的 REST API
│   │   ├── ec2 - the Amazon EC2 API bindings
│   │   ├── metadata - Nova metadata Server API 
│   │   └── openstack - the OpenStack API
│   ├── cell - Nova Cell形似部署时的 Cell Service Component
│   ├── cert
│   ├── cloudpipe - VPN Server Management
│   ├── cmd - Component的Service启动脚本
│   ├── compute - Nova Compute Service Component
│   ├── conductor - Nova Conductor Service Component
│   ├── console - instance console library
│   ├── consoleauth - Nova ConsoleAuth Service Component
│   ├── db - database abstraction
│   ├── hacking
│   ├── image - the Image Service
│   ├── ipv6
│   ├── keymgr
│   ├── locale
│   ├── network - the Nova Network Service Component
│   │   ├── neutronv2 - the Neutron API
│   │   └── security_group
│   ├── objects - The Model Level
│   ├── objectstore - 基于本地文件实现的S3-like的Storage Server，主要用于测试
│   ├── openstack - ongoing effort to reuse Nova parts with other OpenStack parts.
│   │   └── common
│   ├── pci
│   ├── scheduler - Nova Scheduler Service Component
│   ├── servicegroup - Service Group Support
│   │   └── drivers  - Service Group Drivers: ZooKeeper，Database，Memcached
│   ├── spice： Spice libraries for accessing instances
│   ├── storage: Linux SCSI Related Misc Methods
│   ├── tests - Unit tests. Sub directories should mirror ./nova
│   ├── virt - Hypervisor abstractions
│   ├── vnc - VNC libraries for accessing instances
│   └── volume - the Volume Service
├── plugins - hypervisor host plugins. Mostly for XenServer.
```

就其中已经分析或了解到的部分进行下说明：

* Service Component：对应的就是一个个Nova服务，即nova-compute、nova-scheduler、nova-conductor，等等
* cmd： 用于启动相应的Nova Service, 主要是利用Service.Create来封装相应的Service Component。例如cmd.api和cmd.scheduler分别是启动nova-api和nova-scheduler服务的。RDO的相应的服务启动脚本最终都是调用的这里
* conductor： 引入的目的是为了避免compute直接对数据库的访问，并引入些跨计算节点的功能。从目前Havana代码来看，还有很多功能还是直接由compute来访问数据库的，比如关机等

## Service Component ##

每个Service Component基本都会包含由api、manager和rpcapi，api主要用于模块间的直接调用，rpcapi主要用于通过MQ进行远程的模块间调用，manager主要用于此模块的功能，具体执行时，又可能会有多个driver可选。

## Data Workflow ##


一般来说，一个对外的计算相关特性的具体执行路径为：

1. 首先由nova.api.openstack.compute来处理REST API相关逻辑
2. 调用Controller的计算服务，及nova.compute.api
3. 涉及到远程调用其它Service Component的，则会调用相应Component的rpcapi，通过MQ发送消息，涉及到本地调用其它Service Component的，则会调用其的api
4. 相应的Service Component的Manager会从MQ中接收到相应的消息或者被此Component的api调用，根据请求类型，使用相应的driver或者继续按3的过程和其它Component进行交互


## 参考资料 ##

* [Nova Internals](http://www.sandywalsh.com/2012/04/openstack-nova-internals-pt1-overview.html)
* [Hacking on Openstack Nova Souce Code](http://www.slideshare.net/lzyeval/hacking-on-openstacks-nova-source-code)
* [Service, Mananger 和 Driver](http://docs.openstack.org/developer/nova/devref/services.html)
