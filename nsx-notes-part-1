# NSX Notes

*   当前网络虚拟化存在的问题
*   NSX功能架构
*   NSX的部署和配置
*   NSX的逻辑服务和作用
*   NSX的逻辑服务与vSphere已有组件的关系
*   NSX+vSphere，提供的能力总结
*   与FusionNetwork的对比

## 当前网络虚拟化存在的问题

*   大二层需求
*   ...
*   趋势：SDN

## NSX功能架构

 ![NSX_solution_overview](D:\workspace\ALL\技术归档\Network\NSX\Notes\NSX_solution_overview.png)

### 功能组件说明

*   组件形态
    *   NSX Manager、NSX Controller、NSX Edge均为虚拟机形态存在
    *   NSX Controller对应的虚拟机会组成一个集群（奇数个）
    *   每个分布式逻辑路由器会有一个对应的Control VM
    *   数据层的NSX vSwitch基于VDS，并通过在各ESXi节点上部署VXLAN、DLR、DFW等内核模块提供的vxlan、分布式路由和分布式防火墙的底层能力支持，提供SDN的数据平面能力
*   NSX的典型应用场景

 ![nsx_solution_env](D:\workspace\ALL\技术归档\Network\NSX\Notes\nsx_solution_env.png)

*   各功能组件的作用

    *   NSX Manager为NSX的管理面，与一个vCenter是1:1的关系，其负责：

        *   安装部署阶段：部署NSX Controller集群、各ESXi主机安装部署VXLAN、DLR、DFW等内核模块、各ESXi主机上安装部署控制面所需的用户Agent
        *   业务管理阶段：部署配置NSX Edge服务网关以及相应的网络服务，如LB、FW、NAT等等
    *   NSX Controller集群为NSX的控制面，负责：

        *   管理Hypervisor中的交换和路由模块
        *   Controller集群管理整个NSX域所有的虚拟网络组件
        *   集群节点数目为奇数个，不同虚拟网络组件的Master分散在不同的集群节点上
        *   针对VXLAN，NSX Controller的“软件定义“管理机制，可以不依赖于物理网络提供IGMP Snooping或PIM Routing能力，同时支持”ARP suppression"，降低ARP flooding发生的概率
    *   NSX Edge，提供各种各样的网络服务：
        *   路由：虚拟的逻辑网络与实际的物理网络之间的路由
        *   NAT
        *   Firewall：stateful firewall，南北向流量
        *   LB
        *   L2/L3 VPNs
        *   DHCP, DNS
    *   NSX VDS和相应内核模块，为NSX的数据面，负责实际的数据传输：
         ![esxi_host_user_kernel_components](D:\workspace\ALL\技术归档\Network\NSX\Notes\esxi_host_user_kernel_components.png)

## NSX的部署和配置

### 部署整体流程

 ![deploy_seq](D:\workspace\ALL\技术归档\Network\NSX\Notes\deploy_seq.png)

*   第一步：通过模板部署NSX Manager虚拟机

*   第二步：登录NSX Manager提供的Portal，注册vCenter信息，重启vCenter。之后登录vSphere web client，可以看到主页上会新增一个入口“Networking & Security"，通过此入口可以进入NSX管理主页

*   第三步：进入NSX管理主页的”安装“页面，首先部署三个”NSX控制器节点“

*   第四步：集群准备：

    *   ”主机准备“->安装：部署内核模块和Agent到各个ESXi节点上
    *   "主机准备"->配置VXLAN：在各节点上配置VXLAN对应的VTEP接口，及其使用的IP池等
    *   ”逻辑网络准备“->分段ID：配置VXLAN的SegmentID池
    *   ”逻辑网络准备"->传输区域：新建传输区域

    ![prepare_hosts](D:\workspace\ALL\技术归档\Network\NSX\prepare_hosts.png

*   第五步：按需部署NSX Edge网关和各种网络服务，包括传输区域、逻辑交换机、分布式逻辑路由器等等

### vSphere Web Client中的NSX管理界面

![nsx_in_web_client](D:\workspace\ALL\技术归档\Network\NSX\Notes\nsx_in_web_client.png)

### 部署过程中遇到的问题记录

#### 第四步：准备主机集群

* 现象：准备主机时总是报错，提示信息是vib module not installed

* 失败的尝试

  * 先尝试配置了NTP和DNS，没有解决问题
  * 然后按照官方文档，禁用了VUM，没有解决问题
  * Ref：http://pubs.vmware.com/NSX-62/index.jsp#com.vmware.nsx.install.doc/GUID-07ED3DD6-BF82-4097-8702-4587FA88CFE2.html
  * Ref：http://kb.vmware.com/kb/2053782

* 定位过程

  * 经搜索资料，得到NSX部署时相关的日志为
    * ESXi主机：/var/log/esxupdate.log
    * vCenter: /var/log/vmware/eam/eam.log
  * 从ESXi主机的日志中看到，实际错误是ESXi在访问https://{vcenter_ip}:443/eam/vib?id=xxx形式的url下载vib时超时

* 解决措施

  * 经过搜索网上资料，尝试在ESXi Host上均手动修改防火墙规则，增加对443端口outbound的放行，之后再次安装，部署成功

  * Ref： https://pubs.vmware.com/vsphere-51/index.jsp?topic=%2Fcom.vmware.vsphere.troubleshooting.doc%2FGUID-54AFB94B-75DE-4895-B1BF-EE498F92B717.html

  * firewall rule

     ![firewall_rules](D:\workspace\ALL\技术归档\Network\NSX\firewall_rules.png)

  #### 第五步：部署网络服务中的DLR

  * 现象：部署DLR后，DLR对应的虚拟机已经创建成功，但NSX会提示等待DLR初始化超时，最终会把DLR虚拟机关机和删除，部署失败
    * ESXi：使用的版本是(vmware -vl命令可以查到) ESXi 6.0.0 build-2159203
  * 解决措施：升级vSphere版本
    * 升级vCenter到6.2，使用的是VMware-vCenter-Server-Appliance-6.0.0.20000-3634791-patch-FP.iso升级包
      * 挂载上述ISO到vCenter虚拟机
      * 登录vCenter虚拟机，进入Command模式（非shell模式），执行software-package install --iso --acceptEulas命令
    * 升级ESXi，使用的是update-from-esxi6.0-6.0_update02.zip升级包
      * 登录到ESXi节点
      * 执行如下命令：esxcli software vib install -n esx-base -n vsan -n vsanhealth -d /path/to/update-from-esxi6.0-6.0_update02.zip
      * 增加-n vsan之类的参数，是为了避免vsan版本无法满足要求而不能升级，参考：https://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2145049

### 单站点和多站点（已有站点升级）部署建议

由于NSX Manager和vCenter是1:1的关系

故在中小数据中心中，典型的部署场景如下：

 ![single_vcenter](D:\workspace\ALL\技术归档\Network\NSX\Notes\single_vcenter.png)

在大型数据中心，或者数据中心已经有数个存量的vCenter时，可采用的部署场景如下：![dedicate_vcenters](D:\workspace\ALL\技术归档\Network\NSX\Notes\dedicate_vcenters.png)

## NSX逻辑服务和作用

### 传输区域

基本等同于Neutron中的”网络“，定义了VXLAN逻辑网络覆盖的vSphere集群范围。

传输区域可以横跨多个vSphere的VDS：

<TODO: img>

### 逻辑交换机

逻辑交换机属于某个特定的”传输区域“，用于提供”L2隔离的逻辑子网“。

逻辑交换机可以连接虚拟机或物理机，也可以连接到Edge网关：

*   可以通过逻辑交换机的”添加虚拟机“功能入口，将vSphere中现有虚拟机的虚拟网卡连接到此逻辑交换机上
*   不仅可以提供"virtual-to-virtual"的通信，还可以通过DLR的”VXLAN-to-VLAN bridging“能力提供"virtual-to-physical"的通信能力
*   可以通过逻辑交换机的”连接Edge“功能入口，将此逻辑交换机连接到已有的Edge上
    *   DLR的”uplink“也需要通过逻辑交换机才可以连接到Edge Gateway

对于BUM报文，逻辑交换机提供三种Replication Mode：

*   Braodcast：依赖物理网络支持VXLAN的广播，即IGMP Snooping和PIM Routing。BUM包被封装为组播包在物理网络上发送
*   Unicast：每个VTEP子网中的一台ESXi主机被设置为此子网的UTEP(Unicast Tunnel End Point)，代理本子网内BUM包的复制。BUM包被直接封装为“单播”包在物理网络上发送
*   Hybrid：每个VTEP子网中的一台ESXi主机被设置为此子网的MTEP(Multicast Tunnel End Point)，代理本子网内BUM包的组播，依赖物理网络支持IGMP Snooping，不需要PIM Routing。BUM包在同一个子网内被封装为组播包在物理网络上发送，跨子网使用“单播”包在物理网络上发送

### 分布式防火墙

提供L2-L4的**stateful**防火墙服务，一般用于”东西向“流量的防护，如前文所述，”南北向“流量的防护一般通过NSX Edge网关上的防火墙服务提供。

DFW同时提供了”spoofguard“和”traffic redirection"的能力，前者提供类似”IP/MAC绑定“的能力，避免VM发出源IP非法的报文；后者提供了接入第三方组件的能力，可以将报文重定向第三方的service vm中进行进一步的安全防护，典型场景为DPI
