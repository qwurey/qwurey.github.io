---
layout: post
title: "Postgres-XL介绍"
date: 2015-10-08 19:32:26 +0800
comments: true
categories: pgxl
tags: [pgxl, postgres-xl]
styles: [data-table]
---


> “沉默即赞同” —— 无名


#### **什么是Postgres-XL**
<br>

XL的意思是：eXtensible Lattice，可以扩展的格子，即将PostgreSQL应用在多机器上的分布式数据库的形象化表达。

Postgres-XL 是一个完全满足ACID的、开源的、可方便进行水平扩展的、多租户安全的、基于PostgreSQL的数据库解决方案。

Postgres-XL 可非常灵活的应付各种负载，比如：

* OLAP（通过MPP并行化）
* OLTP
* OLAP & OLTP
* 操作数据存储
* Key-value存储，包括JSON格式


不同的应用场景：

* 支持商业智能应用（数据仓库&数据集市），因为PGXL支持MPP(Massively Parallel Processing)
* Web2.0，数据库扩容的解决方案
* 遗留系统的数据库扩容的解决方案
* 新应用，可以先使用PostgreSQL，之后随着数据库变大使用PGXL扩容


PGXL底层为PostgreSQL，这意味着它支持所有支持PostgresSQL类型的驱动，包括： JDBC, ODBC, OLE DB, Python, Ruby, perl DBI, Tcl, and Erlang.


<br>
#### **PostgreSQL与Postgres-XL**
<br>
1994年，Postgre95发布，开源。
1996年，PostgreSQL继承了Postgre95，发布。
2010年，Postgres-XC发布。
2012年，前PGXC核心开发者创建StormDB公司，进行了一些改进，包括对MPP并行化的性能改进和多租户安全。
2013年，TransLattice收购了StormDB。
2014年，将项目开源，命名为Postgres-XL。

<br>
#### **Postgres-XC与Postgres-XL**
<br>

PGXL的架构师和开发者	很多都是以前做PGXC的，PGXL的部分代码是从PGXC移植过来的。

比起功能性，PGXL更强调稳定性, 正确性和性能. 

PGXL增加了一些重要的性能提升，比如MPP和replan avoidance on the data nodes，这些都是PGXC没有的。

PGXC目前集中在OLTP的业务上面，PGXL则更加灵活，可以应用于很多不同种类的业务上，比如可以用在大数据处理领域，除此，在多租户的环境中，PGXL也更加安全。

PGXL的社区非常开放。

<br>
#### **PGXL架构基本知识**
<br>

PGXL是一系列PostgreSQL数据库的集群，在上层看来就像使用一个数据库一样。根据设计方案的不同，每张表可以是replicated或是distributed的形式。

PGXL有三个主要组件，分别是GTM，Coordinator和Datanode。

* GTM(Gloable Transaction Manager)负责提供事务的ACID属性；
* Datanode负责存储表的数据和本地执行由Coordinator派发的SQL任务；
* Coordinator负责处理每个来自Application的SQL任务，并且决定由哪个Datanode执行，然后将任务计划派发给相应的Datanode，根据需要收集结果返还给Application；

GTM通常由一台独立的服务器承担，因为GTM需要处理来自所有Coordinator和Datanode的事务请求。为了将Coordinator和Datanode上进程的请求和响应聚集到一台机器上，可以配置GTM-Proxy。GTM-Proxy会减少GTM的负载，同时会帮助处理GTM失效的情况。即便如此，GTM还是可能会发生单点失效问题，这时可以配置一个GTM-Standby节点作为GTM的备用节点。

每台机器最好同时配置一个Coordinator和一个Datanode，这样既不用担心二者的负载均衡，而且可以降低网络流量。

<br>
#### **如何实现High Availability**
<br>

可以对每个节点增加slave，就类似PostgreSQL的streaming replication一样。

GTM可以有一个GTM Standby。

针对自动的failover，目前可以使用Corosync/Pacemaker，虽然它们现在还不是核心项目。

<br>
#### **PGXL的license**
<br>

PGXL和PostgreSQL使用相同的LICENSE，截止到2015年，使用的还是Mozilla Public License.

