### 系统部件选型

* 本地存储使用共享存储方案，ceph、nfs、iscsi，需确定
    * 对于ceph也要看它作为ephemeral后端，可否利用链接克隆能力：从代码看不是的，但是使用ceph可提供共享存储能力，作为glance后端时可高效download
    * 对于NFS来说，可修改nova配置，使得ephemeral的后端及base的路径均指向NFS，可共享存储，可链接克隆，作为glance后端时可高效download
    * 但Ephemeral的初衷是什么呢？ https://s3.amazonaws.com/aws001/trailhead/TrailHead_Choosing_the_Right_Data_Storage_Solutions.pdf
        * Cache.. App Level Replication..Stateless
* swift vs ceph
    * https://ceph.com/cloud/ceph-and-mirantis-openstack/
        * Ceph的问题
            * 对时钟偏移相当敏感
            * multisite support，刚刚引入Emperor作为异步同步机制，默认的同步复制机制使其不适合长距离的跨站点场景
            * block storage density, 当前是2:1 3:1左右，在引入Firefly
            * Swift API gap，并非100%支持Swift API
        * 部署时的一些问题[ ]通过udev automount rules来使得主机重启后，自动挂载OSD devices and Journals
            * ssh互信
            * ...
    * 可考虑选择Swift
* qpid / rabbitmq 选型

### 待确定的问题

* ceph可否做存储qos
* ceph同时对内和对外提供服务，隔离？
* S3是否需要单独的网络？
* 考虑容灾方案
* 考虑备份方案。现状是无VM级别备份能力。ephemeral盘只支持以glance镜像保存快照方式进行一定程度的支持；cinder提供了卷备份，但对于发放VM时，由于不支持链接克隆，即使可卸载到存储上利用复制卷来做，也会有发放速度限制

### 可靠性 & 集群相关

* 计算节点HA：ecavate接口，可一定程度支持HA，但应该是缺乏防脑裂处理的，社区
* 使用哪个HA软件
    * 比对：http://www.slideshare.net/kenhui65/openstack-ha
        * DBRD有split-brain问题
    * HAProxy 和 websocket
        * http://blog.exceliance.fr/2012/11/07/websockets-load-balancing-with-haproxy/
        * http://blog.silverbucket.net/post/31927044856/3-ways-to-configure-haproxy-for-websockets  ： 其它的一些websocket检测方式，三种
        * https://www.exratione.com/2012/07/proxying-websocket-traffic-for-nodejs-the-present-state-of-play/ ： haproxy对于ssl也有问题？
        * http://www.infoq.com/articles/Web-Sockets-Proxy-Servers
* ceph管理网对网络平面划分的需求
* ceph可否用于单机演示，对controller节点数目需求
* 扩容：ceph扩容，是否中断服务；ceph扩容时，短时间内不能发放业务？

### 扩容 & 可扩展性 & 业务迁移

* cell 和 region的应用场景及使用配置，region/cell如何做双活站点
* 跨rack、idc、site，大规模扩容方案
* dashboard的可扩展性（session，websocket）
* 使用哪个LB软件（和HA软件选型结合）
    * Load Balance FAQ: http://blog.exceliance.fr/loadbalancing-faq/
    * Load Balance Layer 7 session: http://blog.exceliance.fr/2012/03/29/load-balancing-affinity-persistence-sticky-sessions-what-you-need-to-know/

### 安全 & 网络

* spice、vnc agent的安全隔离
* 各node的防火墙配置、端口开放策略
* 安全证书等生成
* 网络平面隔离（在网卡不足时的虚拟vlan方案）
* LB时，是否需要俩VIP
* 安全：ceph的大磁盘时的安全冲击

### 升级 

* cell情况下的升级方案
* 现有OS能力下的升级方案

### 可维护性

* 系统监控
* 日志
* 配置变更和管理

### 性能

* keystone考虑，主要是在cell region等情况下的性能
* 需启用cell/region的规格（触发条件？），以及cell/region各自的规格数据
* 从性能方面考虑，LB什么情况下应该部署（跨机房，机架，具体规格？）
* cell方式的性能问题考虑
* ephemeral盘使用的是链接克隆形式，在超过8个时可能会有性能问题
* glance性能瓶颈
