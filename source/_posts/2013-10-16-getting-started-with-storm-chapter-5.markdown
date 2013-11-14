---
layout: post
title: "[译]【Storm入门指南】第五章 Bolts"
date: 2013-10-16 21:38
comments: true
categories: Storm 
published: true
toc: true
---

> 特别注明：本文翻译自 [Getting started with Storm](http://shop.oreilly.com/product/0636920024835.do) 第五章，以作学习交流之用，非盈利性质。如需转载，请以超链接形式标明文章原始出处和作者信息及版权声明。


如你所见，bolt 是Storm集群的关键组件。本章，你将了解到一个 bolt 的生命周期、一些 bolt 设计策略以及几个演示如何实现它的例子。

# 5.1 Bolt 生命周期

Bolt 是一个将 tuple 作为输入、产出多个 tuple 作为输出的组件。实现一个 bolt，通常需要实现``IRichBolt``接口。Bolt 在客户端服务器中被创建，序列化到 topology 中，然后被提交到集群的管理节点上。集群加载工作者，将 bolt 反序列化、调用 bolt 的``prepare``方法，然后开始处理 tuple。

<!-- more -->

> 自定义一个 bolt，你应该在构造方法中设置参数并将它们保存为实例变量中，这样当提交 bolt 到集群时它们会被序列化。

# 5.2 Bolt 结构

Bolt 有如下方法：

- ``declareOutputFields(OutputFieldsDeclarer declarer)``
  ：定义该 bolt 的输出模式。

- ``prepare(java.util.Map stormConf, TopologyContext context, OutputCollector col
lector)``
  ：在 bolt 开始处理 tuple 钱被调用

- ``execute(Tuple input)``
  ：处理一个单独的输入 tuple

- ``cleanup()``
  ：当 bolt 将要关闭时被调用

看看下面这个将句子分解成单词的 bolt 例子：

```java
class SplitSentence implements IRichBolt {
	private OutputCollector collector;
	
	public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
		this.collector = collector;
	}

	public void execute(Tuple tuple) {
		String sentence = tuple.getString(0);
		for(String word: sentence.split(" ")) {
			collector.emit(new Values(word));
		}
	}

	public void cleanup() {
	}
	
	public void declareOutputFields(OutputFieldsDeclarer declarer) {
		declarer.declare(new Fields("word"));
	}
}
```

如你所见，该 bolt 非常浅显直白。值得提到的是，上述例子没有消息保证机制。这意味着，如果 bolt 由于某些原因丢弃了一条消息，无论是宕机或者是编码上故意丢弃，生成该条消息的 spout 将从不会被通知到，或者中间其他任何 bolt 或 spout。

在许多情形下，你希望在整个 topology 中保证消息的处理。

# 5.3 可靠 Bolt vs. 不可靠 Bolt

如以前提到的，Storm 保证一条由 spout 发送的消息将被所有的 bolt 完全处理。这个设计上的考量，意味着你需要决定你的 bolt 是否需要保证消息。 

一个 topology 是一棵节点树，在其中 消息（tuple）沿着一条或多条分支传递。每个节点将执行``ack(tuple)``或者``fail(tuple)``。这样，Storm 就能知道一条消息何时失败了，并通知到发送该消息的一个或若干个 spout。由于一个 Storm topology 运行在一个高度并行化的环境中，持续追踪原始 spout 实例的最佳方法是在消息 tuple 中保存原始 spout 的引用。这项技术被称为锚定（Anchoring）。改变前例中的``SplitSentence`` bolt，让它可以保证消息的处理。

```java
class SplitSentence implements IRichBolt {
	private OutputCollector collector;
	
	public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
		this.collector = collector;
	}
	
	public void execute(Tuple tuple) {
		String sentence = tuple.getString(0);
		for(String word: sentence.split(" ")) {
			collector.emit(tuple, new Values(word));
		}
		collector.ack(tuple);
	}

	public void cleanup() {
	}

	public void declareOutputFields(OutputFieldsDeclarer declarer) {
		declarer.declare(new Fields("word"));
	}
}
```

锚定发生的地方是在``collector.emit()``。如前所述，传递 tuple 使得 Storm 可以追踪原始 spout。``collector.ack(tuple)``和``collector.fail(tuple)``将每条消息发生的状况告知 spout。当在 topology 树中每条消息都被处理后，Storm 将这个从 spout 而来的 tuple 视为 完全处理。一个被认为失败的 tuple，是指在给定超时时间内未被完全处理的消息。默认时间是 30 秒。

> 可以通过改变``Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS``来设置 topology 的超时配置。

当然，spout 也需要相应地关注消息失败、重试或丢弃的情形。

> 每个你处理的 tuple 都要被应答或者宣告失败。Storm 使用内存来追踪每个 tuple。所以，如果你不进行 ack/fail，任务可能会用光内存。

# 5.4 多流

一个 bolt 可以使用 ``emit(streamId, tuple)``向多个流发送 tuple。用``streamId``来标识流。然后，你可以在``TopologyBuilder``中决定订阅哪个流。

# 5.5 多锚定

为了让一个 bolt 连接或聚合流，你需要将 tuple 缓存在内存中。为了这种情形下的消息担保，你需要将流锚定到不知一个 tuple 上。通过调用包含tuple列表的``emit``方法来实现。

```java
...
List<Tuple> anchors = new ArrayList<Tuple>();
anchors.add(tuple1);
anchors.add(tuple2);
_collector.emit(anchors, values);
...
```

通过这种方式，任何时候一个 bolt 被应答或失败，它将通知树。并且由于流被锚定到不止一个 tuple，所有参与的 spout 都会被通知到。

# 5.6 使用 IBasicBolt 来自动 Ack

你可能注意到，许多情形下你需要消息担保。为了让事情更简单，Storm 提供了另一个 bolt 接口 ``IBasicBolt``。该接口封装了一种模式，即在执行``execute``方法后调用``ack``方法。该接口的一种实现类``BaseBasicBolt``，被用来做自动应答。

```java
class SplitSentence extends BaseBasicBolt {
	public void execute(Tuple tuple, BasicOutputCollector collector) {
		String sentence = tuple.getString(0);
		for(String word: sentence.split(" ")) {
			collector.emit(new Values(word));
		}
	}

	public void declareOutputFields(OutputFieldsDeclarer declarer) {
		declarer.declare(new Fields("word"));
	}
}
```

> 被发射到``BasicOutputCollector``的 tuple 被自动锚定到输入 tuple。


  


