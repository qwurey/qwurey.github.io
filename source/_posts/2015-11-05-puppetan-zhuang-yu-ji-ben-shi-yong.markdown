---
layout: post
title: "puppet安装与基本使用"
date: 2015-11-05 16:01:14 +0800
comments: true
categories: [tool]
tags: [puppet]
---

> “Talk is cheap. Show me the code” —— Linux



#### 一、puppet基本介绍

puppet是一个开源的软件自动化配置和部署工具。它是典型C／S架构，由puppet master和若干puppet client组成。

#### 二、安装过程

##### 1. 设置hosts：

```
172.16.14.150	master.hp	# puppet master
172.16.14.151	slave1.hp	# puppet client
```

##### 2. 为了方便，进行前期准备：


(1). 关闭SELinu	
(2). 关闭防火墙

##### 3. 安装puppet官方源：

```
 wget http://yum.puppetlabs.com/el/6/products/x86_64/puppetlabs-release-6-7.noarch.rpm
 rpm -ivh puppetlabs-release-6-7.noarch.rpm
 yum update
```

##### 4. puppet master的安装与配置：

安装puppet server：

```
yum -y install puppet-server
```

启动puppet server：

```
service puppetmaster start
```

查看证书，由于puppet默认将server本机当成一个client，所以已经将本机管理起来，自动签发了一个证书：

```vim
[urey@master ~]$ sudo puppet cert list --all
+ "master.hp" (SHA256) 89:94:59:7C:85:F2:03:27:82:72:09:63:F4:86:C3:A4:2E:15:EE:7F:A5:31:C1:57:91:29:E4:9A:1B:87:F5:9B (alt names: "DNS:master.hp", "DNS:puppet", "DNS:puppet.hp")
```

##### 5. puppet client的安装与配置：

安装puppet client：

```
yum -y install puppet
```

启动puppet client：

```
service puppet start
```

##### 6. puppet server与puppet client交互：

由于server还没有给client发证书，所以还不能与server通信，但是需要发出请求，一旦server签收，就可以正常通信了。在client运行：

```
puppet agent --no-daemonize --onetime --verbose --debug --server=master.hp
```

在server端查看，`＋` 号代表已经签收，可以看到，目前`slave1.hp`还没有被签收：

```
[urey@master ~]$ sudo puppet cert list --all
  "slave1.hp" (SHA256) F7:D4:2D:2C:6D:63:A8:78:13:CE:A7:0A:2A:4E:0F:E0:9E:D7:71:BC:0F:3F:52:68:21:B4:17:4B:B7:08:FC:6E
+ "master.hp" (SHA256) 89:94:59:7C:85:F2:03:27:82:72:09:63:F4:86:C3:A4:2E:15:EE:7F:A5:31:C1:57:91:29:E4:9A:1B:87:F5:9B (alt names: "DNS:master.hp", "DNS:puppet", "DNS:puppet.hp")
```

server通过 `sign` 签收：

```
puppet cert --sign slave1.hp
```

再次查看：

```
puppet cert list --all
[urey@master ~]$ sudo puppet cert list --all
+ "master.hp" (SHA256) 89:94:59:7C:85:F2:03:27:82:72:09:63:F4:86:C3:A4:2E:15:EE:7F:A5:31:C1:57:91:29:E4:9A:1B:87:F5:9B (alt names: "DNS:master.hp", "DNS:puppet", "DNS:puppet.hp")
+ "slave1.hp" (SHA256) 1A:23:D6:D4:31:92:AF:7D:F4:CA:A3:83:0A:68:DF:24:11:DB:0A:D5:6C:A8:0A:3C:EE:6C:9F:E0:62:B4:D3:B0
```

这样，client和server可以正常通信了。

如果不想这么麻烦，每次server查看有哪些client发出请求，再一一签收的话，server可以通过编辑配置文件来自动完成签收。

##### 1. 关闭server，修改配置文件，使得server添加自动签发证书：

编辑 /etc/puppet/puppet.conf 文件， 在[main]段内加入 autosign = true，server = master.com


```
[main]
    # The Puppet log directory.
    # The default value is '$vardir/log'.
    logdir = /var/log/puppet

    # Where Puppet PID files are kept.
    # The default value is '$vardir/run'.
    rundir = /var/run/puppet

    # Where SSL certificates are kept.
    # The default value is '$confdir/ssl'.
    ssldir = $vardir/ssl
    autosign = true
    server = master.hp

[agent]
    # The file in which puppetd stores a list of the classes
    # associated with the retrieved configuratiion.  Can be loaded in
    # the separate ``puppet`` executable using the ``--loadclasses``
    # option.
    # The default value is '$confdir/classes.txt'.
    classfile = $vardir/classes.txt

    # Where puppetd caches the local configuration.  An
    # extension indicating the cache format is added automatically.
    # The default value is '$confdir/localconfig'.
    localconfig = $vardir/localconfig
```

重启server。

##### 2. 关闭client，编辑 /etc/puppet/puppet.conf 文件，在[agent]段内加入 listen = true，server = master.com，为客户端指定puppet服务器,并开启Master的推送功能：

```
[agent]
    # The file in which puppetd stores a list of the classes
    # associated with the retrieved configuratiion.  Can be loaded in
    # the separate ``puppet`` executable using the ``--loadclasses``
    # option.
    # The default value is '$confdir/classes.txt'.
    classfile = $vardir/classes.txt

    # Where puppetd caches the local configuration.  An
    # extension indicating the cache format is added automatically.
    # The default value is '$confdir/localconfig'.
    localconfig = $vardir/localconfig
    listen = true
    server = master.hp
```

编辑 /etc/puppet/auth.conf 文件， 在 auth / 最下面加入以下语句：

```
path /run
method save
allow master.com
```

好了，这样，client请求server后，server会自动sign相应的client。

#### 三、puppet目录结构

##### 1. server端的目录结构：

```
[urey@master ~]$ ls /etc/puppet/
auth.conf  environments  fileserver.conf  manifests  modules  puppet.conf
```

server上代码的目录结构如下：

|– auth.conf # 访问认证文件

|– environments # 环境

|– fileserver.conf # 自带的文件服务器配置文件，用户控制客户端访问文件资源的权限

|– manifests

|– |– site.pp # puppet的主入口文件，主要用于全局配置

|– |– xx.pp # puppet各个模块汇总  如abc.com.pp

|– |– nodes.pp # 节点配置文件，用于配置客户端节点应用哪些具体的配置

|– modules # puppet的各个模块所在文件

|– puppet.conf # 整个puppet的主配置文件，主要由两部分组成：[main]和[agent]，定义了一些基础的文件目录

##### 2. client端的目录结构：

```
[urey@slave1 ~]$ ls /etc/puppet/
auth.conf  modules  puppet.conf
```

client上代码的目录结构如下：

|– auth.conf # 访问认证文件

|– modules # puppet的各个模块所在文件

|– puppet.conf # puppet主文件所在目录，主要由两部分组成：[main]和[agent]，定义了一些基础的文件目录


#### 四、User Cases

到这里，终于可以看一下puppet的强大了。

我们在server端编写一段code：

```
[urey@master ~]$ sudo vim /etc/puppet/manifests/site.pp
```


```
node default {
        file {
                "/tmp/helloworld.txt": content => "hello, world";
        }
}
```

在client端之行puppet，执行后查看对应目录下生成了相应的文件：

```
[urey@slave1 ~]$ puppet agent --test --server=master.hp
Info: Caching certificate for slave1.hp
Info: Caching certificate_revocation_list for ca
Info: Caching certificate for slave1.hp
Info: Retrieving pluginfacts
Notice: /File[/home/urey/.puppet/var/facts.d]/mode: mode changed '0775' to '0755'
Info: Retrieving plugin
Info: Caching catalog for slave1.hp
Info: Applying configuration version '1446715692'
Notice: /Stage[main]/Main/Node[default]/File[/tmp/helloworld.txt]/ensure: defined content as '{md5}e4d7f1b4ed2e42d15898f4b27b019da4'
Info: Creating state file /home/urey/.puppet/var/state/state.yaml
Notice: Finished catalog run in 0.02 seconds
[urey@slave1 ~]$ cat /tmp/helloworld.txt
[urey@slave1 ~]$ hello,world
```

puppet需要学习的还有很多，未完待续。








