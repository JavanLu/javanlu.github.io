---
layout: post
title: "Storm 单机版安装指南"
date: 2013-10-11 09:19
comments: true
categories: Storm
published: true
toc: true
---

最近要使用storm进行一些数据流处理。在Unbuntu系统工作，发现storm安装过程各种错误，遂写下本文。本文介绍安装单机版Storm的步骤，集群版只是配置稍微有点区别。

英文指南见： [https://github.com/nathanmarz/storm/wiki/Setting-up-a-Storm-cluster](https://github.com/nathanmarz/storm/wiki/Setting-up-a-Storm-cluster)

要想使用Storm集群，要安装如下工具：
Python、Java 6、Git、Zookeeper、ZeroMQ、JZMQ、Storm

Python、Java 和 Git 的安装这里就不赘述了。

<!-- more -->

# 一、安装 Zookeeper

```sh
wget http://mirror.nexcess.net/apache/zookeeper/zookeeper-3.3.6/zookeeper-3.3.6.tar.gz
tar -zxvf zookeeper-3.3.6.tar.gz 
cp -R zookeeper-3.3.6 /usr/local/
ln -s /usr/local/zookeeper-3.3.6/ /usr/local/zookeeper
vim /etc/profile

export ZOOKEEPER_HOME="/path/to/zookeeper"
export PATH=$PATH:$ZOOKEEPER_HOME/bin

cp /usr/local/zookeeper/conf/zoo_sample.cfg /usr/local/zookeeper/conf/zoo.cfg
mkdir /tmp/zookeeper
mkdir /var/log/zookeeper
```

# 二、安装ZeroMQ

```sh
wget http://download.zeromq.org/zeromq-2.2.0.tar.gz
tar zxf zeromq-2.2.0.tar.gz 
cd zeromq-2.2.0
./configure --with-pgm
make
make install
sudo ldconfig 
```

**在安装过程中常见错误有:**

```
./configure configure: error: Unable to find a working C++ compiler
autoreconf: failed to run glibtoolize: No such file or directory
autoreconf: libtoolize is needed because this package uses Libtool
```

在``./configure``阶段，会检查各种依赖信息，可能会提示一些包需要安装，缺什么装什么，常见的有：

```sh
sudo yum install uuid*
sudo yum install libtool
sudo yum install libuuid 
sudo yum install libuuid-devel
sudo yum install gcc
sudo yum install gcc-c++
sudo yum install make
```

``./configure --with-pgm``是官网推荐的方式，不然在安装 JZMQ 时会报错 ``cannot find file zmq.h``


# 三、安装JZMQ


```sh
git clone git://github.com/nathanmarz/jzmq.git
cd jzmq
./autogen.sh
./configure
make
make install
```

**常见错误有：**

```
autogen.sh: error: could not find autoreconf.  autoconf and automake are required to run autogen.sh.
```

说明缺少autoconf 和 automake，安装：

```
sudo yum autoconf
sudo yum automake
```

在执行 ``make`` 的时候可能会报错：

```
*** No rule to make target `classdist_noinst.stamp', needed by `org/zeromq/ZMQ.class'.  Stop.
```

解决此问题的方案如下：

```sh
cd jzmq
touch src/classdist_noinst.stamp
cd src/org/zeromq/
javac *.java
make
sudo make install
```

# 四、安装 Storm


去[http://storm-project.net/](http://storm-project.net/)下载 Storm

```sh
unzip storm-0.8.2.zip
mv storm-0.8.2 /usr/local/
ln -s /usr/local/storm-0.8.2 /usr/local/storm
vim /etc/profile

export STORM_HOME=/usr/local/storm-0.8.2
export PATH=$PATH:$STORM_HOME/bin
```

至此，安装完毕。让我们测试一下本地模式下的WordCount。



# 五、运行storm-starter中的WordCount

```
git clone https://github.com/nathanmarz/storm-starter.git
mvn -f m2_pom.xml package
cp m2_pom.xml ./pom.xml
mvn eclipse:eclipse
```

注： storm-starter 是一个maven工程，请自行先配置好Maven，不会的google。

将项目 import 到Eclipse中，运行 ``storm.starte.WordCountTopology``，
看到如下信息，表明运行成功

```sh
10811 [Thread-31] INFO  backtype.storm.daemon.task  - Emitting: split default ["apple"]
10811 [Thread-16] INFO  backtype.storm.daemon.executor  - Processing received message source: split:5, stream: default, id: {}, ["apple"]
10811 [Thread-16] INFO  backtype.storm.daemon.task  - Emitting: count default [apple, 53]
10811 [Thread-31] INFO  backtype.storm.daemon.task  - Emitting: split default ["a"]
```

**常见错误：**

-  **在执行mvn -f m2_pom.xml package 报错1：**


```
Exception in thread "main" java.lang.NoClassDefFoundError: clojure.core.protocols$seq_reduce
```

解决方案如下：这是由于使用了GJC JVM，而不是使用Oracle JDK，导致字节码不被识别，可以使用``java -version``看具体的JDK信息。需要将系统的默认JDK指定为Oracle JDK的，如下：

```
sudo update-alternatives --install /usr/bin/java java ${JAVA_HOME}/bin/java 300
sudo update-alternatives --install /usr/bin/javac javac ${JAVA_HOME}/bin/javac 300
sudo update-alternatives --config java
sudo update-alternatives --config javac
```

- **在执行mvn -f m2_pom.xml package 报错2：**

```
Failure to transfer org.twitter4j:twitter4j-core:2.2.6-SNAPSHOT..
```

原因是由于Storm-starter使用twitter4j这个仓库来下载twitter4j-core这两个包，而twitter4j已经被伟大的长城盾了。

解决方案：修改Storm-Starter的pom文件m2-pom.xml ，修改dependency中twitter4j-core 和 twitter4j-stream两个包的依赖版本。

```xml
<dependency>
    <groupId>org.twitter4j</groupId>
    <artifactId>twitter4j-core</artifactId>
    <version>[2.2,)</version>
</dependency>
<dependency>
    <groupId>org.twitter4j</groupId>
    <artifactId>twitter4j-stream</artifactId>
    <version>[2.2,)</version>
</dependency>
```

# 六、单机环境运行storm-starter

上面的运行没有用到配置的单机环境，只是一个storm的虚拟环境下的demo，下面是部署到单机环境的步骤：

- (1) 启动Zookeeper：``/usr/local/zookeeper/bin/zkServer.sh start``，由于是单机环境，所以不用额外配置的。

- (2) 配置storm:

```
 storm.zookeeper.servers:
     - "127.0.0.1"
 ##此处的端口号应该和zookeeper配置文件中的一致
 storm.zookeeper.port: 2181 
 nimbus.host: "127.0.0.1"
 storm.local.dir: "/tmp/storm"
 supervisor.slots.ports:
  - 6700
  - 6701
  - 6702
``` 
注意：在每一项的开始时要加空格，冒号后也必须要加空格，否则storm就不认识这个配置文件了。

- (3) 启动主节点： ``bin/storm nimbus``

- (4) 启动从节点：``bin/storm supervisor``

- (5) 执行WordCount: `` storm jar StormStarter.jar storm.starter.WordCountTopology``

- (6) 执行完毕，查看storm：``bin/storm ui``，在浏览器输入：``127.0.0.1:8080``，会看到执行任务的情况。


如果运行过程中出错：

```
java.lang.RuntimeException: 
org.apache.thrift7.transport.TTransportException: 
java.net.ConnectException: Connection refused
```

解决方案如下：

- 检查zookeeper,nimbus,surpervisor是否开启了
- 原因是storm客户端连不上Nimbus ，查看一下你配置的storm.yaml文件是否正确。
- 如果storm.yaml文件正确，查看端口号是否被占用。


到这里，在单机下配置和运行storm就算结束了，现在可以玩玩它了。


