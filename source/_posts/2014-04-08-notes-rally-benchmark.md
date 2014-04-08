# Notes: Benchmarking OpenStack at megascale: How we tested Mirantis OpenStack at SoftLayer

## 测试目的

测试一个OpenStack集群可承受的并发创建Ephemeral虚拟机的最大数量，作为此集群的一个性能基线。

测试指标：
1. 平均启动时间
2. 启动成功率
3. 最大并发请求规模

测试样本：
1. flavor： 1 vCPU， 64MB RAM， 2GB disk

## 测试环境

1. OpenStack HAVANA
2. CPU超分配： 1:32
3. 内存超分配： 1:1.5
4. 物理服务器： 4物理core（HT），16GB内存，250GB硬盘
5. controller： 128GB内存，SSD
6. controller和compute在不同机房，但之间延迟为40ms
7. Nova-network， FlatDHCP
8. 参数调整：调整SQLAlchemy
9. controller的数据库为MySQL+Galera主主，HAProxy LB， Corosycn/Pacemaker IP level HA
10. 测试引擎： Rally

## 测试中的一些调整和问题

1. 增加sqlalchemy的连接池
2. 增加haproxy的连接数
3. Pacemaker会有positive detection of haproxy，主要是因为haproxy连接池耗尽


## 测试结论

1. 支持并发请求未500
2. 并发请求为250时系统最稳定
3. 单集群支持75000虚拟机

