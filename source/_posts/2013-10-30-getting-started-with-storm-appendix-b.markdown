---
layout: post
title: "[译]【Storm入门指南】附录B 安装Storm集群"
date: 2013-10-30 15:38
comments: true
categories: Storm
published: true
---

如果想创建一个 Storm 集群，有如下两种途径：

- 使用 [storm-deploy](https://github.com/nathanmarz/storm-deploy)在 Amazon EC2创建一个集群，如第六章所示

- 手动安装Storm（本节的内容）


手动安装Storm，需要安装如下软件：

<!-- more -->

- Zookeeper 集群，参见[管理手册](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html)

- Java 6.0

- Python 2.6.6

- Unzip 指令

> 上述所有要安装的软件，是 Nimbus 和监控程序所需要的。

当安装了上述软件后，请安装本地库。

安装 ZeroMQ，运行：

```sh
wget http://download.zeromq.org/historic/zeromq-2.1.7.tar.gz
tar -xzf zeromq-2.1.7.tar.gz
cd zeromq-2.1.7
./configure
make
sudo make install
```

安装JZMQ，运行：

```sh
git clone https://github.com/nathanmarz/jzmq.git
cd jzmq
./autogen.sh
./configure
make
sudo make install
```

一旦安装了本地库，下载稳定版的Storm并解压。

修改配置文件，增加你的Storm 集群配置。（所有默认配置可以在Storm 的 defaults.yaml 文件中看到）

编辑 ``conf/storm.yaml``文件来修改storm 集群配置：

```
storm.zookeeper.servers:
	- "zookeeper addres 1"
	- "zookeeper addres 2"
	- "zookeeper addres N"

storm.local.dir: "a local directory"

nimbus.host: "Numbus host addres"

supervisor.slots.ports:
	- supervisor slot port 1
	- supervisor slot port 2
	- supervisor slot port N
```

配置解释如下：

- storm.zookeeper.servers
 ：zookeeper 服务器的地址

- storm.local.dir
 ：Storm 存储内部处理数据的本地目录
  