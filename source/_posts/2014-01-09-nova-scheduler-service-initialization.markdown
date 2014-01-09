---
comments: true
date: 2014-01-09 22:40:31
layout: post
title: 'NOVA Scheduler Service Initialization Process'
categories:  [OpenStack]
tags:  [Nova, scheduler, RabbitMQ]

---

# NOVA-SCHEDULER服务启动流程 #

**前提**

1. 对Nova的整体结构已经有所理解
2. 基于stable/havana分支
3. 基于Redhat的RDO库进行的环境安装，基于CentOS 6.4
4. 主机名为controller

**内容**

1. openstack-nova-scheduler服务启动流程
2. MessageQueue的相关知识及在scheduler服务启动过程中的涉及的行为

## 执行结果 ##

根据之前对Noah系统结构的理解，在scheduler启动过程中，会和MessageQueue交互，创建相应的Exchange和Consumer。


启动前的MessageQueue的状态：

```
	#rabbitmqctl list_exchanges name type durable internal
	Listing exchanges ...
		direct	true	false
	amq.direct	direct	true	false
	amq.fanout	fanout	true	false
	amq.headers	headers	true	false
	amq.match	headers	true	false
	amq.rabbitmq.log	topic	true	false
	amq.rabbitmq.trace	topic	true	false
	amq.topic	topic	true	false
	...done.

	#rabbitmqctl list_bindings source_name source_kind destination_name destination_kind routing_key
	Listing bindings ...
	...done.

	#rabbitmqctl list_connections pid name  port host peer_port peer_host state channels protocol
	Listing connections ...
	...done.

	#rabbitmqctl list_channels pid connection name number consumer_count
	Listing channels ...
	...done.

	#rabbitmqctl list_consumers
	Listing consumers ...
	...done.
```

由上可知，在Scheduler启动前，只有RabbitMQ-Server默认创建的一些exchange，而binding、connection、channel和consumer均为空。

现在启动Scheduler服务：

```
service openstack-nova-scheduler start
```

启动成功后，再次查看MQ的状态如下：


```
	#rabbitmqctl list_exchanges name type durable internal
	Listing exchanges ...
		direct	true	false
	amq.direct	direct	true	false
	amq.fanout	fanout	true	false
	amq.headers	headers	true	false
	amq.match	headers	true	false
	amq.rabbitmq.log	topic	true	false
	amq.rabbitmq.trace	topic	true	false
	amq.topic	topic	true	false
	nova	topic	false	false
	scheduler_fanout	fanout	false	false
	...done.

	#rabbitmqctl list_bindings source_name source_kind destination_name destination_kind routing_key
	Listing bindings ...
		exchange	scheduler	queue	scheduler
		exchange	scheduler.controller	queue	scheduler.controller
		exchange	scheduler_fanout_132fbd38ac304ffb9adb93c09656e769	queue	scheduler_fanout_132fbd38ac304ffb9adb93c09656e769
	nova	exchange	scheduler	queue	scheduler
	nova	exchange	scheduler.controller	queue	scheduler.controller
	scheduler_fanout	exchange	scheduler_fanout_132fbd38ac304ffb9adb93c09656e769	queue	scheduler
	...done.

	#rabbitmqctl list_connections pid name  port host peer_port peer_host state channels protocol
	Listing connections ...
	<rabbit@controller.2.1156.0>	192.168.251.11:56077 -> 192.168.251.11:5672	5672	192.168.251.11	56077	192.168.251.11	running	{0,8,0}
	...done.

	#rabbitmqctl list_channels pid connection name number consumer_count
	Listing channels ...
	<rabbit@controller.2.1161.0>	<rabbit@controller.2.1156.0>	192.168.251.11:56077 -> 192.168.251.11:5672 (1)	1	3
	...done.

	#rabbitmqctl list_consumers
	Listing consumers ...
	scheduler	<rabbit@controller.2.1161.0>	1	true
	scheduler.controller	<rabbit@controller.2.1161.0>	2	true
	scheduler_fanout_132fbd38ac304ffb9adb93c09656e769	<rabbit@controller.2.1161.0>	3	true
	...done.

   	#rabbitmqctl list_queues name pid owner_pid consumers status
	Listing queues ...
	scheduler	<rabbit@controller.2.1162.0>		1	running
	scheduler.controller	<rabbit@controller.2.1163.0>		1	running
	scheduler_fanout_132fbd38ac304ffb9adb93c09656e769	<rabbit@controller.2.1164.0>		1	running
	...done.
```

由上可知，在scheduler启动过程中，对于MQ的操作包括：

* 创建了名称为nova，类型为topic的exchange
* 创建了名称为scheduler_fanout，类型为fanout的exchange
* 创建了名称为scheduler、scheduler.controller和scheduler_fanout_xx的三个queue
* 创建了一个具有一个channel的connection
* 创建了名称为scheduler、schueler.controller和scheduler_fanout_xx的三个consumer
* 创建了6个bindings，分别将上面的三个新建的queue和上述的三个新建的exchange和RabbitMQ默认的名称为空的exchange进行了binding

故可以得知，Scheduler启动后，相应的MQ结构如下：

```
	Exchange(nova,topic) <---<routing_key:scheduler>---Queue(scheduler)--->Consumer(scheduler)

	Exchange(nova,topic) <---<routing_key:scheduler.controller>---Queue(scheduler.controller)--->Consumer(scheduler.controller)

	Exchange(scheduler_fanout,fanout) <---<routing_key:scheduler>---Queue(scheduler_fanout_xxx)--->Consumer(scheduler_fanout_xxx)

```

对比RabbitMQ和OpenStack的概念，其中名称为nova的Exchange为Topic Exchange，支持时类似于MSG的Unicast。名称为scheduler_fanout_xx的Exchange为Fanout Exchange，支持的是类似于MSG的Broadcast。

名称分别为scheduler和scheduler.controller的Consumer均为Topic Consumer，用于从连接的Queue中接收相应topic的msg，这里的topic就是指RabbitMQ中的Queue的"Routing Key"。

故Scheduler服务启动时，就会创建一个Topic Exchange，并会初始化两个Queue连接此Exchange和两个Topic Consumer，两个Consumer分别接受的topic分别是“scheduler”和"scheduler.{hostname}"。

结合OpenStack的文档，~~当进行rpc.cast调用时，实际是使用scheduler这个Queue发送消息，当进行rpc.call调用时，实际是使用scheduler.{hostname}这个Queue发送消息。~~ （注: 使用哪个队列跟使用rpc.cast和rpc.call无关系，只跟调用这俩rpc时传入的topic有关系。 这俩rpc调用的区别仅仅在于是否需要后续的双向通信）

当前尚不知晓在服务初始化过程中创建的fanout Exchange、Queue和Consumer的具体作用，从如下[参考资料](http://www.gossamer-threads.com/lists/openstack/dev/20622)中了解到其主要作用是：

* 发送原子性和非关键性的信息到所有的workers
* 计算节点定期发送容量等信息到scheduler

这一点待后续进一步确定。

以上初始化过程中未涉及到Publisher的创建，结合OpenStack的文档，Publisher是在真正发送消息时创建的。实际上在发送消息和处理消息过程中会涉及到创建Direct Exchange、Topic Publisher、Direct Publisher和Direct Consumer等MQ相关组件，这些的详细过程会在后续分析。

## 代码路径 ##

### 主要路径 ###

服务启动脚本为/etc/init.d/openstack-nova-scheduler，查看此脚本发现在启动服务时实际执行的是/usr/bin/nova-scheduler,而/usr/bin/nova-scheduler是一个python脚本，实际执行的是nova.cmd.scheduler.main

nova.cmd.scheduler.main的主要代码如下：

```
    server = service.Service.create(binary='nova-scheduler',
                                    topic=CONF.scheduler_topic)
    service.serve(server)
    service.wait()
```

第一行中的CONF.scheduler_topic，是从配置文件/etc/nova/nova.conf中读取的，默认是"scheduler"。

第二行的serve方法如下：

```
	global _launcher
    if _launcher:
        raise RuntimeError(_('serve() can only be called once'))

    _launcher = service.launch(server, workers=workers)
```

在最后一行调用的launch方法中，会在workers为None时，调用ServiceLauncher.launch_service来运行service，否则会调用ProcessLauncher.launch_service来运行service:

```
	def launch(service, workers=None):
    if workers:
        launcher = ProcessLauncher()
        launcher.launch_service(service, workers=workers)
    else:
        launcher = ServiceLauncher()
        launcher.launch_service(service)
    return launcher
```

其中ProcessLauncher是fork出workers数量的进程，ServiceLauncher是生成一个线程(greenthread），此线程的入口是先后调用相应service的start和wait方法，进而完成服务的初始化。

在Scheduler中，使用的是ServiceLauncher。


### Service.create ###

Service.create的主要逻辑是获取配置的"scheduler_manager"对应的类作为manager，加上配置中的report_internal、periodic_enable、periodic_fuzzy_delay等参数，初始化一个Service Object。

在Service的__init__方法中，主要是根据上述参数进行了属性初始化。

### Service.start ###

这里主要关注start方法中关于MessageQueue和ServiceGroup的部分：

```
		self.conn = rpc.create_connection(new=True)
        LOG.debug(_("Creating Consumer connection for Service %s") %
                  self.topic)

        rpc_dispatcher = self.manager.create_rpc_dispatcher(self.backdoor_port)

        # Share this same connection for these Consumers
        self.conn.create_consumer(self.topic, rpc_dispatcher, fanout=False)

        node_topic = '%s.%s' % (self.topic, self.host)
        self.conn.create_consumer(node_topic, rpc_dispatcher, fanout=False)

        self.conn.create_consumer(self.topic, rpc_dispatcher, fanout=True)

        # Consume from all consumers in a thread
        self.conn.consume_in_thread()

        self.manager.post_start_hook()

        LOG.debug(_("Join ServiceGroup membership for this service %s")
                  % self.topic)
        # Add service to the ServiceGroup membership group.
        self.servicegroup_api.join(self.host, self.topic, self)
```

由上可知，第一行首先创建了一个新的Connection，Connection是nova.openstack.common.amqp.ConnectionContext对象，针对create_connection方法，当前由多个driver进行了实现，其中RabbitMQ对应的是nova.rpc.impl_kombu，也即配置文件中配置的rpc_backend。

接着由manager创建了一个rpc_dispatcher,此对象负责在接收到消息后进行处理，细节在后续分析。

接着分别调用create_consumer了三次，在此方法中会创建出上节提到的各个Exchange、Queue、Bindings和Consumers：

```
	# 前两个create_consumer调用, 最终会调用TopicConsumer.__init__:

	# Default options
    options = {'durable': conf.amqp_durable_queues,
               'queue_arguments': _get_queue_arguments(conf),
               'auto_delete': conf.amqp_auto_delete,
               'exclusive': False}
    options.update(kwargs)
	#这一行决定了exchange_name, 默认会从配置文件中读取control_exchange配置（值为#openstack），但是在nova.config中通过		
	#rpc.set_defaults(control_exchange='nova')将其设置为了"nova"
    exchange_name = exchange_name or rpc_amqp.get_control_exchange(conf)     
	#exchange的type是topic
    exchange = kombu.entity.Exchange(name=exchange_name,
                                     type='topic',                                         durable=options['durable'],
                                     auto_delete=options['auto_delete'])
    #此调用会创建出Consumer、Binding和Queue
    super(TopicConsumer, self).__init__(channel,
                                        callback,
                                        tag,
                                        name=name or topic,
                                        exchange=exchange,
                                        routing_key=topic,
                                        **options)

	# 第三个create_consumer调用，最终会调用FanoutConsumer.__init__:

	unique = uuid.uuid4().hex
	# 这也就是上一节看到的scheduler_fanout_xx
    exchange_name = '%s_fanout' % topic
    queue_name = '%s_fanout_%s' % (topic, unique)

    # Default options
    options = {'durable': False,
              'queue_arguments': _get_queue_arguments(conf),                 'auto_delete': True,
               'exclusive': False}
    options.update(kwargs)
    exchange = kombu.entity.Exchange(name=exchange_name, type='fanout',
                                     durable=options['durable'],
                                     auto_delete=options['auto_delete'])
    super(FanoutConsumer, self).__init__(channel, callback, tag,
                                         name=queue_name,
                                         exchange=exchange,
                                         routing_key=topic,
                                         **options)
```

接着调用的consume_in_thread最终会调用到queue的consume方法，从队列中取出消息并进行处理。

最后调用的join方法来加入到相应的ServiceGroup。关于ServiceGroup，主要用途是为了管理Group中各个节点的liveness状态的，具体实现目前默认是通过db，也可以通过ZooKeeper实现。对于Scheduler来说，需要通过ServiceGroup感知到所有Compute Node的存活状态，以支持自身的任务调度。

## 参考资料 ##

* [Nova RPC: cast && call](http://docs.openstack.org/developer/nova/devref/rpc.html)
* [Rabbitmqctl Manual](https://www.rabbitmq.com/man/rabbitmqctl.1.man.html)
* [Consumers and Queues created during Scheduler service init](http://www.gossamer-threads.com/lists/openstack/dev/20622)
* [ServiceGroup](http://docs.openstack.org/trunk/config-reference/content/configuring-compute-service-groups.html)
