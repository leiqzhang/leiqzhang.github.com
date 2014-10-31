# DevOps相关服务搭建过程 #

## 介绍 ##

* 简化日常开发环境的安装和配置工作

## 原则 ##

* 自动化：内部部件的部署要可自动化；尽量对用户透明，让用户使用环境所需进行的操作最少
* 组件化：内部各部件要组件化，便于灵活部署和迁移

## 功能 ##

* 通过提供NTP、Yum Repo、Puppet、DNS、DHCP等服务，实现节点（需要预定义安装一定软件包）启动后，自动完成IP分配、域名设置、本地Repo设置等配置任务，并可通过Puppet实现常用软件包的安装和配置，目前包括：docker/git/python/vim/gcc
* 提供Docker Registry服务，并提供若干自定义镜像


支撑云目前提供如下服务：

* NTP Server服务
* DNS Server服务
* DHCP Server服务
* PUPPET Master服务
* YUM Repo服务
* Docker Private Registry服务
* GIT Repo服务

当前支撑云部署图如下：

**TODO**

为能够和支撑云配合使用，计算节点需要使用定制化的ISO进行安装，目前支持的定制化ISO包括：

* CentOS 7最小化安装
* ifconfig包
* 不安装NetworkManager
* 安装并开机启动ntp
* 安装并开机启动puppet
* 安装git

## 交互流程 ##

一个节点安装完毕ISO到加入到DevOps环境的典型交互流程如下：

** TODO **

## 使用指南 ##

** TODO **

## 部署指南 ##

### 前提 ###

* 支撑云的节点安装了操作系统
* 操作系统至少需要CentOS 7
* 节点配置好网络且可正常访问Git Server
* 节点上已经安装了git和puppet软件包
* 已经有可供搭建Yum Repo源的软件包
* 已经有可用的Docker Registry，且上面已经包含了部署支撑云所需的各镜像

关于搭建Docker Registry和制作Docker镜像，请参考相关文章

### 部署形态 ###

支撑云涉及的服务，以Docker镜像提供的均可以选择部署到虚拟机或物理机上，各服务的具体部署建议参考部署步骤

### 部署步骤 ###

#### Yum Repo ####

Yum Repo服务需要部署在软件仓库所在节点，假设软件仓库目录为/home/repo，Registry地址为10.10.10.10，则部署过程为：

<pre>
docker pull 10.10.10.10:5000/devops/httpd:2.4.6
docker run -d -p 80:80 -v /home:/var/www/html 10.10.10.10:5000/devops/httpd:2.4.6
</pre>

此时访问http://{$server_ip}/repo，就可以看到repo服务已经可以正常访问了

#### 初始化服务器环境 ####

在部署NTP服务、DNS服务和PuppetMaster服务的节点上，需要首先执行此步骤，其主要完成的工作包括：
* 设置节点使用本地Yum源
* 安装并部署docker
* 开启puppetagent服务
* 安装并配置ntp服务

假设GIT Server的IP地址为10.10.10.11，通过如下命令完成此步骤：

<pre>
#下载初始化环境脚本
git clone http://10.10.10.11/devops/puppet-local
#配置
cd puppet-local && sh init.sh
</pre>

#### 部署NTP服务 ####

**建议**：为避免虚拟机时钟偏移，NTP服务最好部署在物理机上

<pre>
git clone http://10.10.10.11/devops/dockerfiles
cd dockerfiles/ntpd-with-local-clock
sh deploy.sh
</pre>

#### 部署DNS服务器 ####

<pre>
git clone http://10.10.10.11/devops/dockerfiles
cd dockerfiles/bind/
</pre>

根据实际情况，传递如下环境变量并执行deploy.sh:

* DOMAIN_NAME: 使用的域
* MASTERDNS_ADDR: DNS服务地址
* PUPPET_ADDR： PUPPET Master地址
* POOL_ADDR：NTP服务器地址
* REPO_ADDR：YUM仓库服务地址
* DHCP_ADDR：DHCP服务器地址
* REGISTRY_ADDR：Docker Registry服务地址
* GITSERVER_ADDR：GIT服务器地址

如:
<pre>
#使用默认值
sh deploy.sh
#自定义
DOMAIN_NAME=example.com sh deploy.sh #备注：暂不支持
</pre>

#### 部署DHCP服务器 ####

**注意**：由于dhcpd在启动时，会验证配置给其的IP池和本地网卡是否在同一个网段，故暂时未找到办法让dhcpd服务运行于Docker的方法

<pre>
git clone http://10.10.10.11/devops/puppet-local
cd puppet-local && sh init-dhcp.sh
</pre>

#### 部署PuppetMaster ####

<pre>
git clone http://10.10.10.11/devops/dockerfiles
cd dockerfiles/puppetmaster/
sh deploy.sh
</pre>


## 开发指南 ##

### 相关Repo ###

## 遗留问题 ##

* 使用systemd管理以docker形式部署的服务
* 对于DNS服务的部署脚本，需要强化支持根据环境变量生成对应的配置文件和zone文件
* 提供Yum Repo服务的部署脚本支持
* DHCP服务器中MAC和Host映射关系的添加，支持通过命令行或者界面方式进行管理
* 搭建DSWare环境，用于统一存储各容器运行所需使用的数据目录和文件，以支持容器的高可靠性
* 考虑统一使用puppet来代替deploy.sh
* 考虑引入Mcolletive，实现Puppet的推送模式
* DHCP服务器与vagrant结合，实现其它网段DHCP服务器的快速部署
* 无论是Puppet的Pull/Push模式，调研对节点进行分组配置的能力


# 经验&教训&改进点 #

## 经验&教训 ##

### Puppet ###
* Puppet的Master/Agent之间依赖SSL通信，必须保证Master和Agent之间的时间同步
* Puppet的Master识别Agent并为Agent建立SSL证书依赖的是Agent的FQDN，Agent的FQDN是通过Facter收集的，如果配置了DNS服务器，Facter会从DNS服务器上获取本机的FQDN

### DHCP ###
* DHCP服务器使用DDNS和DNS服务器交互时，也要保证DHCP服务器和DNS服务器的时间同步
* DHCP服务运行时，如果报"no free leases"错误，请检查/var/lib/dhcpd/dhcpd.leases文件的权限（dhcpd用户是否有写权限），同时注意配置selinux规则
* CentOS 7自带的DHCP客户端dhclient工具，自身并不具备接收DHCP Server推送过来的hostname并设置到本机的能力

### OS ###
* CentOS默认的NetworkManager一些行为尚未摸透，开启此服务会影响获取主机fqdn的几个命令的执行逻辑，建议关闭此服务

### Docker ###
* 对于低于1.3的Docker，如果要使用Host上的目录作为数据卷给容器使用，且容器会对此数据卷进行写操作，则需要在启动容器前，首先通过chron命令设置好这些主机目录的SELinux规则
* 容器执行时，最好通过tail命令等将容器运行的主服务的日志文件重定向到标准输入输出，这样就可以通过docker logs看到后台运行容器的日志，可借助dockerize工具更方便做到这一点

## 改进点 ##

* 目前Docker使用环境变量来定义容器运行时的行为，当环境变量很多时，并不方便
* Docker容器内服务的日志如何收集和存储，目前社区建议的使用统一的数据volume作为log目录并不太理想
   * 配置容器内运行的服务或应用，使其log到标准输入和输出，对于仅支持输出到文件的，可参考dockerize工具和http://blog.froese.org/2014/05/15/docker-logspout-and-nginx/
   * 方法一：使用systemd管理容器，使用systemd的journal来收集和重定向日志
   * 方法二：使用logspout来统一收集和重定向日志：https://github.com/progrium/logspout

# 相关资源 #

* 部署PuppetMaster： https://docs.puppetlabs.com/guides/passenger.html
* Dockerize工具：http://jasonwilder.com/blog/2014/10/13/a-simple-way-to-dockerize-applications/
* DDNS：http://www.semicomplete.com/articles/dynamic-dns-with-dhcp/
* BIND配置：http://www.unixmen.com/dns-server-installation-step-by-step-using-centos-6-3/
* Puppet：https://www.digitalocean.com/community/tutorials/how-to-install-puppet-to-manage-your-server-infrastructure
