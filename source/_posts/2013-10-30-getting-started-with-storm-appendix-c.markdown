---
layout: post
title: "[译]【Storm入门指南】附录C 安装真实示例"
date: 2013-10-30 15:39
comments: true
categories: Storm
published: true
---

首先，从GitHub仓库克隆该示例：

```
> git clone git://github.com/storm-book/examples-ch06-real-life-app.git
```

*src/main*：包含Topology源代码

*src/test*：Topology的测试

*webapps directory h*：包含 Topology 相关的 Node.js WebApp

<!-- more -->

![图1](/images/post/storm/appendix_c_1.png)


# 安装 Redis

安装Redis非常简单：

1. 从Redis官方下载最新的稳定版本；

2. 提取文件；

3. 运行 make，接着``make install``。

这将编译Redis，将可执行文件放置到PATH目录下，这样就可以开始使用Redis了。

你可以从Redis官网得到关于命令和设计方面的文档。

# 安装 Node.js

安装 Node.js 简单易懂。从[http://www.nodejs.org/#download](http://www.nodejs.org/#download)下载最新的Node.js。

提取文件，运行 ``./configure``、``make``、``make install``。

你可以从官网获取更多信息。

# 编译和测试

为了编译示例，需要在你的机器上开启``redis-server``。

```
>nohup redis-server &
```

之后，运行``mvn``指令来编译和测试应用。

```sh
>mvn package
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 32.163s
[INFO] Finished at: Sun Jun 17 18:55:10 GMT-03:00 2012
[INFO] Final Memory: 9M/81M
[INFO] ------------------------------------------------------------------------
>
```

# 运行 Topology

当Redis服务器启动，并且编译顺利完成后，可以在``LocalCluster``中运行 topology。

```
>java -jar target/storm-analytics-0.0.1-jar-with-dependencies.jar
```

当Topology运行后，运行下面的指令来启动 Node.js Web 应用：

```
>node webapp/app.js
```

# 玩转示例

在浏览器输入 [http://localhost:3000/](http://localhost:3000/)，开始示例之旅吧。
