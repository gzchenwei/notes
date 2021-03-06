### RHCS

#### 基础

集群是由2个或者多个计算机节点组成的一起执行任务的系统，主要的集群类型有4种

* 存储型
* 高可用型
* 负载均衡型
* 高性能型

##### 组件

RHCS：REAHAT CLUSTER SUITE （集群套件），是软件组件的集成，可以通过不同的配置进行部署来满足对性能、高可用、负载均衡等的需要

* 群集基础结构

​        提供节点群集方式的基本功能：配置文件管理、成员资格管理、锁管理、安全管理

* 高可用性服务管理

​        提供在某个节点不可操作时，服务从一个群集节点到另外一个节点的故障切换

* 群集管理工具

​        设置、配置和管理redhat集群的配置和管理工具，这些工具和群集基础结构组件、高可用和服务管理组件、存储组件一起使用

* LVS

   提供IP负载均衡

##### 高可用型集群

coremail主要应用的场景为高可用集群，以下会主要介绍高可用集群

在一个节点停止运作时将服务切换到另外一个节点，提供服务的持续可用性，高可用型集群必须维护数据的完整性

配置高可用群集需要用到的组件及概念

* 集群管理器（CMAN)
  * 通过监控群集节点的信息来跟踪成员资格
  * 群集成员资格发生变化时，通知其他基础结构组件


* 资源组管理器(rgmanager)

  监控service的状态以及制定故障切换操作


* 集群配置管理工具
  * luci:服务器组件，运行在一台机器上并通过ricci进行通讯
  * ricci: 运行在每一台节点上与luci通讯


* Failover Domain

  当一个节点挂了，会自动转移资源到Failover Domain中的其他节点，在Faiover Domain中定义节点的优先级，优先级数字越低，优先级越高

* Quorum

  资源的转移是有条件的，RHCS提供了仲裁的方法Quorum(法定票数),每个可用节点进行投票，当投票数大于总票数的一半，即为集群可用

* fence设备

  * 在节点挂掉的情况下，转移之前，使用fence隔离挂掉的节点，否则资源无法转移，主要是为了防止脑裂
  * 所谓脑裂，就是由于节点之间的心跳信息无法传达，双方都认为对方挂掉，自己需要接管资源，如果发生脑裂，可能导致双方都接管数据，从而造成数据损坏
  * 通过服务器或者存储本身的硬件管理接口或者外部电源管理设备，对服务器或者存储发出硬件管理指令

#### 安装及配置

##### 安装

```
yum install cman rgmanager luci ricci
```

##### 配置集群

安装好luci后，通过luci的管理界面进行集群配置，url https://ip:8084，登录用户名为系统用户

* 创建集群，添加节点

   创建集群，并将节点添加到集群，最后会生成配置文件/etc/cluster/cluster.conf

  ![rhcs-1](../图片/rhcs-1.png)

  

* 添加fence设备

  测试使用了dell idrac，生产环境根据实际选择

  ![rhcs-3](../图片/rhcs-3.png)

* 添加failover domains

  定义failover domains 定义节点之间的优先级以及切换的规则

  ![rhcs-4](../图片/rhcs-4.png)

* 添加资源

  后端高可用需要的资源主要是

  * 共享存储（测试使用了nfs)
  * VIP
  * 启动脚本

  ![rhcs-5](../图片/rhcs-5.png)

  ![rhcs-6](../图片/rhcs-6.png)

  ![rhcs-7](../图片/rhcs-7.png)![rhcs-8](../图片/rhcs-8.png)

* 添加资源组

  资源需要添加到资源组，资源组里面的资源加载是有顺序的，这个顺序是有默认优先级的

  优先级为 磁盘 > ip > 脚本

* 启动资源组

  ​	

#### 注意事项

* 关闭防火墙
* 设置好/etc/hosts，添加每个节点
* 关闭selinux

#### 常用命令

* clustat 

  查看集群的状态

* cman_tool status

  查看集群的详细信息

* cman_tool nodes -a

​       查看节点信息

#### 常用日志

日志文件的路径： /var/log/cluster

* cman日志

corosync.log

* rgmanager日志

rgmanager.log

* fence日志

fenced.log

### 参考资料

https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/5/html/cluster_suite_overview/s1-service-management-overview-cso

http://blog.51cto.com/jiayimeng/1877007

pcs resource defaults resource-stickiness=100
