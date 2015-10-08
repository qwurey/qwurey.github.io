---
layout: post
title: "Postgres-XL集群的搭建"
date: 2015-10-08 19:32:26 +0800
comments: true
categories: pgxl
tags: [pgxl, postgres-xl]
styles: [data-table]
---


> “如果你惟一的工具是一把锤子，你往往会把一切问题看成钉子” —— 无名


#### **集群规划**
<br>

建立5个虚拟机构成的集群，虚拟机的os均为centos6.5，依次命名为cnode1，cnode2，cnode3，cnode4，cnode5，其中cnode1为gtm，其余4个节点均为coordinator和datanode。


hostname | ip | function
-------- | ----
cnode1 | 172.16.15.131 | gtm
cnode2 | 172.16.15.132 | coordinator & datanode
cnode3 | 172.16.15.133 | coordinator & datanode
cnode4 | 172.16.15.134 | coordinator & datanode
cnode5 | 172.16.15.135 | coordinator & datanode

<br>
#### **建立用户**
<br>

每个节点(`cnode1~cnode5`)都建立用户`postgres`，并且建立`.ssh`目录，并配置相应的权限：

```
useradd postgres
passwd postgres
su - postgres
mkdir ~/.ssh
chmod 700 ~/.ssh
```

<br>
#### **ssh免密码登录**
<br>

在cnode1节点配置：

```
su - postgres
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

将刚生成的认证文件拷贝到cnode2到cnode5中，使得cnode1可以免密码登录cnode2~cnode5的任意一个节点：

```
scp ~/.ssh/authorized_keys postgres@cnode2:~/.ssh/
```


<br>
#### **安装postgres-xl**
<br>

每个节点安装编译pgxl所需的依赖包：

```
yum install -y flex bison readline-devel zlib-devel openjade docbook-style-dsssl 
```

每个节点安装postgres-xl和pgxc_ctl，我这里使用的版本是：`postgres-xl-v9.2-src.tar.gz`：

```
tar -zxvf postgres-xl-v9.2-src.tar.gz
cd postgres-xl
./configure --prefix=/home/postgres/pgxl/
make
make install
cd contrib/
make
make install
```

每个节点编辑环境变量：

```
vim ~/.bashrc
```
添加如下内容：

```
export PGHOME=/home/postgres/pgxl
export PGUSER=postgres
export LD_LIBRARY_PATH=$PGHOME/lib:$LD_LIBRARY_PATH
export PATH=$PGHOME/bin:$PATH
```

使之生效：

```
source ~/.bashrc
```


<br>
#### **配置集群**
<br>

关闭防火墙或者放开相应的端口，这里我直接关闭了每个虚拟机的防火墙，并且重启它们：

```
chkconfig iptables off   --重启后生效
```

选择cnode1(gtm)，进入/home/pgxc/postgres-xl/contrib/pgxc_ctl目录中，执行pgxc_ctl命令来生成配置集群的模板文件：

```
./pgxc_ctl				---会提示Error说没有配置文件，忽略即可
PGXC prepare			---执行该命令将会生成一份配置文件模板
PGXC ^C
```

根据自己的集群配置该文件：

```
vim ~/pgxc_ctl/pgxc_ctl.conf
```

我这里简单的进行了如下的配置：

```
#user and path
pgxcInstallDir=$HOME/pgxl
pgxcOwner=postgres
pgxcUser=$pgxcOwner

#gtm
gtmName=gtm
gtmMasterServer=cnode1
gtmMasterPort=20001
gtmMasterDir=$HOME/pgxl/nodes/gtm

#---- Configuration ---
gtmExtraConfig=none                     # Will be added gtm.conf for both Master and Slave (done at initilization only)
gtmMasterSpecificExtraConfig=none       # Will be added to Master's gtm.conf (done at initialization only)


gtmSlave=n

#gtm proxy
gtmProxy=n
#gtmProxyDir=$HOME/pgxl/nodes/gtm_pxy
#gtmProxyNames=(gtm_pxy1 gtm_pxy2 gtm_pxy3 gtm_pxy4)
#gtmProxyServers=(cnode2 cnode3 cnode4 cnode5)
#gtmProxyPorts=(20001 20001 20001 20001)
#gtmProxyDirs=($gtmProxyDir $gtmProxyDir $gtmProxyDir $gtmProxyDir)
#gtmPxyExtraConfig=none
#gtmPxySpecificExtraConfig=(none none none none)

#coordinator
coordMasterDir=$HOME/pgxl/nodes/coord
coordNames=(coord1 coord2 coord3 coord4)
coordPorts=(20004 20004 20004 20004)
poolerPorts=(20010 20010 20010 20010)
coordPgHbaEntries=(0.0.0.0/0)
coordMasterServers=(cnode2 cnode3 cnode4 cnode5)
coordMasterDirs=($coordMasterDir $coordMasterDir $coordMasterDir $coordMasterDir)
coordMaxWALsernder=0
coordMaxWALSenders=($coordMaxWALsernder $coordMaxWALsernder $coordMaxWALsernder $coordMaxWALsernder)
coordSlave=n
coordSpecificExtraConfig=(none none none none)
coordSpecificExtraPgHba=(none none none none)


#datanode
datanodeNames=(datanode1 datanode2 datanode3 datanode4)
datanodePorts=(20008 20008 20008 20008)
datanodePoolerPorts=(20012 20012 20012 20012)
datanodePgHbaEntries=(0.0.0.0/0)
datanodeMasterServers=(cnode2 cnode3 cnode4 cnode5)
datanodeMasterDir=$HOME/pgxl/nodes/dn_master
datanodeMasterDirs=($datanodeMasterDir $datanodeMasterDir $datanodeMasterDir $datanodeMasterDir)
datanodeMaxWalSender=0
datanodeMaxWALSenders=($datanodeMaxWalSender $datanodeMaxWalSender $datanodeMaxWalSender $datanodeMaxWalSender)
datanodeSlave=n
primaryDatanode=datanode1

```

以上各配置项的具体含义可以参考：http://files.postgres-xl.org/documentation/pgxc_ctl.html


<br>
#### **初始化+启动集群**
<br>

在cnode1（gtm）执行命令，这个命令会初始化（配置）各个节点，并且启动相应的进程：

```
pgxc_ctl -c pgxc_ctl.conf init all    --初始化完成后会自动启动
```

可以在每个节点上查看对应的进程：

```
ps -ef | grep gtm
ps -ef | grep postgres
```


<br>
#### **测试集群**
<br>

可以在任意节点上输入以下命令，查看除gtm以外的所有节点的配置情况：

```
select * from pgxc_node; 
```

简单的测试，在cnode2节点上创建一个数据库test，并创建一个表test，插入数据：

```
psql -p 20004 -U postgres 
CREATE DATABASE test;
\c test
CREATE TABLE test(id int);
INSERT INTO test VALUES(1);
SELECT * FROM test;
\q
```

在cnode3节点上查看刚刚在cnode2节点的更改：

```
psql -p 20004 -U postgres  
\c test
SELECT * FROM test;
\q
```

发现有刚刚插入的数据，验证成功！



