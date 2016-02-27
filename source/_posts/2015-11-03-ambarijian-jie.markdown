---
layout: post
title: "Ambari简介"
date: 2015-11-03 19:03:57 +0800
comments: true
categories: ambari
tags: [ambari]
styles: [data-table]
---

> “计算机没什么用。他们只会告诉你答案。” —— 巴勃罗·毕加索，画家


#### **一、Apache Ambari**
<br>
Apache Ambari项目通过提供配置部署、管理、监控功能，目的在使Hadoop的管理更简单。它提供了一个直观的,易于使用的web UI和其背后的一套RESTful api。

```
Ambari enables System Administrators to:

Provision a Hadoop Cluster
Ambari provides a step-by-step wizard for installing Hadoop services across any number of hosts.
Ambari handles configuration of Hadoop services for the cluster.
Manage a Hadoop Cluster
Ambari provides central management for starting, stopping, and reconfiguring Hadoop services across the entire cluster.
Monitor a Hadoop Cluster
Ambari provides a dashboard for monitoring health and status of the Hadoop cluster.
Ambari leverages Ambari Metrics System for metrics collection.
Ambari leverages Ambari Alert Framework for system alerting and will notify you when your attention is needed (e.g., a node goes down, remaining disk space is low, etc).
Ambari enables Application Developers and System Integrators to:

Easily integrate Hadoop provisioning, management, and monitoring capabilities to their own applications with the Ambari REST APIs.
```

#### **二、Ambari设计目标**
<br>

1. Platform Independence：系统需要有平台无关，支持大多数的操作系统。
2. Pluggable Components：可增添的组件
3. Version Management & Upgrade：支持组件的多版本管理和升级
4. Extensibility：可扩展性
5. Failure Recovery：错误恢复
6. Security：安全性
7. Error Trace：错误追踪，给用户足够的细节处理错误
8. Near Real-Time and Intermediate Feedback for Operations：对于长时间的操作，给用户展示中间过程，提供更好的体验


详细介绍可以参看官网的：Ambari-Architecture.pdf


#### **三、Ambari术语理解**
<br>

1. Hadoop Stack：Ambari被认为是用来快速部署、管理、监控Hadoop生态系统各个组件的工具，或者将来可以定义为一个管理大数据平台（不仅仅限于Hadoop生态系统），所以有若干个支持的组件，形象化的成为Hadoop Stack（Hadoop栈 或 Hadoop组件栈）。
2. Service：Hadoo Stack中提供的一个栈（服务），比如：Hive、Spark等等，一个Service可以包含若干个Component，比如：HDFS Service包含NameNode、Secondary NameNode（SNameNode）、DataNode。当然，Service也可能只是一个client library，比如：Pig不包含任何的daemon services，只包含一个client library。
3. Component：上面已经介绍。这里要说明的是一个Component可能跨多个节点，比如：DataNode。
4. Node\Host：集群中的一个机器。
5. Node-Component：特定机器上的特定组件，比如：一台机器上的DataNode。
6. Operation：一个操作。一个操作可能包含若干个Action（动作）。
7. Task：发送给Node执行的任务，一个Action可能包含若干个Task。
8. Stage：一系列Task的集合。
9. Action：一个动作包含一个Task或多个Task。每个动作都有一个action id。若不特别说明，Stage和Action一一对应。比如：HDFS rebalancing，HBase smoke test等等。
10. Stage Plan：一个Operation包含很多个Task，需要在不同的节点上执行，这就需要一个调用顺序。可以并行执行的tasks是一个Stage Plan，即每一个Stage Plan时流水线工作。
11. Manifest：用来定义一个Task，它会被发送到节点上去执行，因此一个manifest必须是可序列化的。Manifest当然必须可以被永久存储在磁盘上用来recovery或者record。
12. Role：可以对应一个组件，比如：NameNode或者DataNode等等，也可以对应一个Action，比如：HDFS rebalancing或者HBase smoke test等等。

#### **四、Ambari架构**
<br>

Ambari由Ambari server和若干个Ambari agent组成。类似于master-slave架构。server通过给agent发送命令控制agent的配置、安装和管理。agent定期汇报heartbeat。server还暴露出api供外接访问，server将系统的数据存在默认的postgres db中。

**Ambari整体架构图：**

<img src="http://7xlhna.com1.z0.glb.clouddn.com/ambari.png" width = "600" height = "400" alt="Ambari-arch" align=center />
<br>

**Ambari Server的架构：**
<img src="http://7xlhna.com1.z0.glb.clouddn.com/ambari-server.png" width = "600" height = "400" alt="ambari-server" align=center />

<br>
**Ambari Agent的架构：**

<img src="http://7xlhna.com1.z0.glb.clouddn.com/ambari-agent.png" width = "600" height = "400" alt="ambari-agent" align=center />

<br>
#### **五、Some Use cases**

通过几个实例来了解server和agent如何交互以及server内部各组件工作的流程：

**I 以给现有cluster增加一个HBase service为例**
HBase的master和slaves将会部署在现有cluster的若干nodes上
<br>


1. API请求调用到达Coordinator的API handler。

2. API handler构建steps用来开始一个service，steps包含：检查prerequisites，顺序安装服务的各个Components，重新配置Nagios server增加对新服务的监视，这里是对HBase server的监视。

3. Coordinator与Dependency Tracker通信确定prerequisites：HBase需要HDFS和ZooKeeper Component，需要Nagios Server提供的对HBase的监视client，和这些required component的required state。至此，Coordinator获得了它想要知道的全部信息，最后它将这些component的desired state写进DB。

4. 在上一步，Coordinator还可能获得API提供的其它信息。这种情况这里不再讨论。
5. Coordinator将components list以及它们的desired state发送给Stage Planner。Stage Planner返回每个node应该执行的操作序列。Stage Planner还会通过Manifest Generator生成manifest。
6. Coordinator将`this ordered list of stages`发送给Action Manager，使用的是同一个request id。
7. Action Manager更新FSM中的每个node的state。
8. Action Manger对每个操作构建一个action id，并写入Plan，将这些操作集合放入队列，等待发送给相应的agent。
9. Heartbeat Handler收集agent报上来的信息，将action的response发送给Action Manager，Action Manger更新FSM。当所有nodes执行完毕对应的action后，这个action才宣告执行完毕；当一个Stage的所有actions都执行完毕后，这个Stage才宣告执行完毕。

**II**

* a1

  * 1
 
  * 2
  
* b2


<br>
参考资料：

1. http://ambari.apache.org/








