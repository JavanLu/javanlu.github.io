---
layout: post
title: "[译]【Storm入门指南】附录A 安装Storm客户端"
date: 2013-10-29 16:02
comments: true
categories: Storm
published: true
---

Storm 客户端允许你使用指令来管理 topology 提交到集群中。遵循如下步骤来安装 Storm 客户端：

- 1. 从 [Storm 站点](http://storm-project.net/downloads.html)下载最新稳定版 Storm；

- 2. 一旦下载了，将其解压到``/usr/local/bin/storm``；

- 3. 接着，添加storm PATH，以便可以运行 storm 指令，无需输入全路径。如果你使用``/usr/local/bin/storm``目录，使用``export PATH=$PATH:usr/local/bin/storm``来设置；

<!-- more -->

- 4. 最后，需要创建 Storm 本地设置用来告诉 nimbus host 是哪台机器。创建 ``~./storm/storm.yaml``，输入``nimbus.host:"our nimbus address"``

现在，你可以在 Storm 集群中管理 topology 了。


> Storm 客户端包含运行Storm集群的所有必需指令，但是还需要安装其他工具以及配置一些参数。请参见[附录B](http://javanlu.github.io/blog/2013/10/16/getting-started-with-storm-appendix-b/)。

为了管理集群中的 toology，你拥有一组非常简单和实用的指令来提交、杀死、禁止、重新激活以及再平衡topology。

jar指令负责执行topology，并通过主函数中的 ``StormSubmitter``对象将它提交到集群中。

```sh
storm jar path-to-topology-jar class-with-the-main arg1 arg2 argN
```

``path-to-topology-jar``是包含了topology代码以及所有类库的编译jar 所在的完全路径。``class-with-the-main``是``StormSubmitter``被执行的主函数，其他参数是主函数方法参数。

Storm 具有 挂起或禁止正在运行的topology、冻结topology spout的能力。当冻结 topology时，已发射的tuple会被处理，但是 spout 的 nextTuple不会被再调用。

禁止一个topology，运行

```sh
storm deactive topology-name
```

如果想重新激活一个被终止的topology，运行：

```sh
storm active topology-name
```

如果想销毁一个topology，可以使用 kill 指令。它会以一种安全的方式销毁topology。首先终止topology，冻结持续的 topology 消息，允许topology处理完正在进行的工作。

杀死一个topology，运行：

```sh
storm kill topology-name
```

> 可以通过在运行杀死指令时增加 -w [time-in-sec] 来改变销毁topology时的冻结持续时间。

再平衡允许你可以将任务再发布到整个集群的工作节点中。当你的任务不平衡时，这是个可派上用场的强大的指令。比如，如果你往正在运行的集群中增加了一些节点的时候。再平衡指令会冻结topology中的消息，将它们再发布到工作节点，然后Storm重新激活该topology。

为了再平衡一个topology，运行：

```sh
storm rebalance topology-name
```

如果想配置冻结的持续时间，使用 -w 参数设置：

```sh
storm rebalance topology-name -w toher-time
```

> 指令的详细解释，请参加：[https://github.com/nathanmarz/storm/wiki/Command-line-client](https://github.com/nathanmarz/storm/wiki/Command-line-client) 。


