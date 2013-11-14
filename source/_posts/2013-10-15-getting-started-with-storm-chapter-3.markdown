---
layout: post
title: "[译]【Storm入门指南】第三章 Topologies"
date: 2013-10-15 17:42
comments: true
categories: Storm
published: true
toc: true
---

> 特别注明：本文翻译自 [Getting started with Storm](http://shop.oreilly.com/product/0636920024835.do) 第三章，以作学习交流之用，非盈利性质。如需转载，请以超链接形式标明文章原始出处和作者信息及版权声明。


你将从本章学习到：如何在一个 Storm topology 的不同组件间传递 tuple，以及如何将 topology 部署到一个 Storm 集群。

# 3.1 流分组

当设计一个 topology 的时候，你需要做的最重要的一件事是定义如何在不同组件间交换数据，换言之就是 bolt 间如何消费数据流。流分组明确了哪些流被每个 bolt 消费以及被如何消费。

> 一个节点可以发射不止一个数据流。流分组允许我们选择接收哪个流。

<!-- more -->

正如[第二章](http://javanlu.github.io/blog/2013/10/14/getting-started-with-storm-chapter-2/)所示，当 topology 定义时设置流分组：

```java
...
	builder.setBolt("word-normalizer", new WordNormalizer()).shuffleGrouping("word-reader");
...
```

上面的代码块展示的是，在 topology 构建中设置一个 bolt，使用随机分组分发数据。一个流分组通常将源组件ID作为一个参数，以及另一个可选的取决于流分组类型的参数。

> 也许在``InputDeclarer``中有超过一个数据源，每个源可以使用不同的流分组方式。

## <strong>3.1.1 随机分组</strong>

随机分组(Shuffle Grouping)是使用最普遍的分组方式。它只需一个参数（源组件），随机派发数据源里面的 tuple，保证每个 bolt 接收到的 tuple 数目大致相同。

随机分组在进行比如数学计算等原子操作中非常有用。然而，如果操作符不能被随机分发，比如[第二章](http://javanlu.github.io/blog/2013/10/15/getting-started-with-storm-chapter-2/)中需要统计单词频率的例子，你需要考虑使用其他分组方式。

## <strong>3.1.2 按字段分组</strong>

按字段分组(Field Grouping)允许你按照一个或多个 tuple 中的字段来控制 tuple 如何被发送到 bolt。它保证同样的数值集合或字段组合被发送到同一个 bolt。回到单词统计的例子，如果你按照``word``字段分组，那么``word-normalizer`` bolt 将发送 指定单词的tuple 到相同的 ``word-counter`` bolt 实例。

```java
...
	builder.setBolt("word-counter", new WordCounter(),2)
	.fieldsGrouping("word-normalizer", new Fields("word"));
...
```

> 按字段分组中的所有字段必须存在源字段声明中。

## <strong>3.1.3 广播分组</strong>

广播分组(All Grouping)给所有用来接收的 bolt 发送每一个tuple的拷贝。这种分组用来向 bolt 发送信号。比如，如果需要刷新缓存，你可以发送一个 ``refresh cache``信号给所有的 bolt。在单词统计的例子中，你可以使用广播分组来增加清楚统计缓存的功能。

```java
public void execute(Tuple input) {
	String str = null;
	try{
		if(input.getSourceStreamId().equals("signals")){
			str = input.getStringByField("action");
			if("refreshCache".equals(str))
				counters.clear();
		}
	}catch (IllegalArgumentException e) {
	//Do nothing
	}
	...
}
```
添加一个``if``检查源数据流。Storm具有命名流的能力（如果你不发送tuple到一个已命名的流，流是``default``）；这是个极好的标识源 tuple 的途径，比如我们想用来标识信号。

在 topology 定义中，添加另一个流到``word-counter`` bolt 中，这样使得``signals-spout``能够发送 tuple 到前者的每一个 bolt 实例。

```java
builder.setBolt("word-counter", new WordCounter(),2)
.fieldsGrouping("word-normalizer", new Fields("word"))
.allGrouping("signals-spout","signals");
```

``signals-spout``的实现代码可以从[Github](https://github.com/storm-book/examples-ch03-topologies/tree/master/src/main/java/countword)中找到。

## <strong>3.1.4 自定义分组</strong>

你可以创建自己定义的分组方式，只需实现``back
type.storm.grouping.CustomStreamGrouping``接口。这给了你决定哪个 bolt 接收某个 tuple 的能力。

让我们修改单词统计的例子，将 tuple 按照单词的首个字母相同的方式进行分组，这样首字母相同的单词将被同一个 bolt 接收。

```java

public class ModuleGrouping implements CustomStreamGrouping, Serializable{
	int numTasks = 0;
	@Override
	public List<Integer> chooseTasks(List<Object> values) {
		List<Integer> boltIds = new ArrayList();
		if(values.size()>0){
			String str = values.get(0).toString();
			if(str.isEmpty())
			boltIds.add(0);
		}else{
			boltIds.add(str.charAt(0) % numTasks);
		}
		return boltIds;
	}
	@Override
	public void prepare(TopologyContext context, Fields outFields,
	List<Integer> targetTasks) {
		numTasks = targetTasks.size();
	}
}

```

上述是``CustomStreamGrouping``的一个简单实现。我们用单词首字母的数值与任务节点数目来取模，来决定哪个 bolt 接收 tuple。

为了在示例中使用该分组，只需改变一下``word-normalizer``的分组方式：

```java
builder.setBolt("word-normalizer", new WordNormalizer())
.customGrouping("word-reader", new ModuleGrouping());
```

## <strong>3.1.5 直接分组</strong>

直接分组(Direct Grouping)是一种特殊的分组方式，用这种分组意味着消息的发送者决定哪个组件用来接收tuple。如前所示，我们使用直接分组来实现根据单词首字母来决定接收者是谁。为了使用直接分组，在``WordNormalizer`` bolt 中使用 ``emitDirect`` 方法来代替 ``emit``。

```java
public void execute(Tuple input) {
	...
	for(String word : words){
	if(!word.isEmpty()){
		...
		collector.emitDirect(getWordCountIndex(word),new Values(word));
		}
	}
	// Acknowledge the tuple
	collector.ack(input);
}

public Integer getWordCountIndex(String word) {
	word = word.trim().toUpperCase();
	if(word.isEmpty())
		return 0;
	else
		return word.charAt(0) % numCounterTasks;
}
```

在``prepare``方法中获取目标任务的数目：

```java
public void prepare(Map stormConf, TopologyContext context,OutputCollector collector) {
	this.collector = collector;
	this.numCounterTasks = context.getComponentTasks("word-counter");
}
```

然后，在 topology 定义中明确数据流将被直接分组：
 
```java
builder.setBolt("word-counter", new WordCounter(),2)
.directGrouping("word-normalizer");
```

## <strong>3.1.6 全局分组</strong>

全局分组(Global Grouping)将所有源实例产生的 tuple 发送到一个目标实例中（更具体点，分配给ID值最低的那个任务）。

## <strong>3.1.7 不分组</strong>

在写作本书的时候（Storm 版本是 0.7.1），不分组(None Grouping)与使用随机分组具有一样的效果。换句话说，使用这种分组方式，不用关心数据流是如何分组的。

> 译者注：有一点不同的是 Storm 会把这个 bolt 放到这个 bolt 的订阅者同一个线程里面去执行。


# 3.2 LocalCluster vs. StormSubmitter

到目前为止，我们使用``LocalCluster``工具类在本机运行 topology。在本机运行 Storm 组件，允许你可以很容易地运行和调试不同的 topology。但是，如果你想将 topology 提交到一个运行中的 Storm 集群呢？ Storm 一个有趣的特性就是可以很容易地发送 topology 到真实的集群中运行。你需要将``LocalCluster``改成``StormSubmitter``，并实现``submitTopology``方法来负责发送 topology 到集群中。

如下是改变的代码：

```java
/LocalCluster cluster = new LocalCluster();
//cluster.submitTopology("Count-Word-Topology-With-Refresh-Cache", conf,
builder.createTopology());
StormSubmitter.submitTopology("Count-Word-Topology-With-Refresh-Cache", conf,
builder.createTopology());
//Thread.sleep(1000);
//cluster.shutdown();
```

> 当你使用``StormSubmitter``，你无法像在``LocalCluster``中一样通过代码控制集群。

接下来，打包代码到Jar包中，它将在运行 Storm 客户端命令时提交 topology。由于使用了Maven，你所需要做的仅仅是去源码目录下运行如下指令：

```sh
	mvn package
```

一旦生成了Jar包，使用``storm jar``指令提交 topology。语法是：``storm jar allmycode.jar org.me.MyTopology arg1 arg2 arg3``。

在本例中，在源码的工程目录下运行：

```sh
storm jar target/Topologies-0.0.1-SNAPSHOT.jar countword.TopologyMain src/main/
resources/words.txt
```
使用上述指令，你已经将 topology 发布到集群中了。

停止或杀掉它，使用：

```sh
storm kill Count-Word-Topology-With-Refresh-Cache
```

> topology的名字必须是唯一的。

# 3.3 DRPC Topologies

有一个特别的 topology，称为 分布式远程方法调用(DRPC)，是Storm 分布式执行的远程方法调用(RPC)[如图3.1所示]。Storm 提供了一些工具来使用 DRPC。第一个是 DRCP 服务器，它在客户端和 Strom topology 之间建立连接，当作 topology spout 的源。它接收执行的功能和参数。在功能运行的每一个数据片上，分配一个请求 ID 在topology结构中来标识 RPC 请求。当 topology 执行到最后一个 bolt，它发射 RPC 的请求 ID 和关联的结构，这样帮助DRPC服务器将结果返回给正确的客户端。

![图3.1](/images/post/storm/chapter3/fig3_1.png)

> 一台DRPC服务器能执行很多功能。每个功能用唯一的名字来标识。

Storm 提供的第二个工具是``LinearDRPCTopologyBuilder``，它是一个用来帮助建立 DRPC topology 的抽象类。生成的 topology 创建 ``DRPCSpouts``——用来连接到DRPC服务器并发射数据到 topology 中的其他部分，包装最后一个 bolt 返回的结果。所有 bolt 被添加到``LinearDRPCTopologyBuilder``依次序执行。

创建一个加法处理过程来展示这种类型的 topology 的用法。这是一个简单的例子，但是涉及到的概念可被扩展来完成复杂的分布式数学操作。

bolt 代码中的输出声明如下：

```java
public void declareOutputFields(OutputFieldsDeclarer declarer) {
	declarer.declare(new Fields("id","result"));
}
```

由于 topology 中仅有一个 bolt，它必须 发射 RPC ID 和结果。

``execute``方法负责执行加法操作：

```java
public void execute(Tuple input) {
	String[] numbers = input.getString(1).split("\\+");
	Integer added = 0;
	if(numbers.length<2){
		throw new InvalidParameterException("Should be at least 2 numbers");
	}
	for(String num : numbers){
		added += Integer.parseInt(num);
	}
	collector.emit(new Values(input.getValue(0),added));
}
```

将 added bolt 加入 topology 的定义中：

```java
public static void main(String[] args) {
	LocalDRPC drpc = new LocalDRPC();
	LinearDRPCTopologyBuilder builder = new LinearDRPCTopologyBuilder("add");
	builder.addBolt(new AdderBolt(),2);
	Config conf = new Config();
	conf.setDebug(true);
	LocalCluster cluster = new LocalCluster();
	cluster.submitTopology("drpc-adder-topology", conf, builder.createLocalTopology(drpc));
	String result = drpc.execute("add", "1+-1");
	checkResult(result,0);
	result = drpc.execute("add", "1+1+5+10");
	checkResult(result,17);
	cluster.shutdown();
	drpc.shutdown();
}
```

创建一个``LocalDRPC``对象在本地运行DRPC服务器。接下来，创建一个 topology builder 然后将 bolt 添加到这个 topology。执行DRPC对象上的``execute``方法来测试这个 topology。

> 使用``DRPCClient``类连接到一个远程DRPC服务器。DRPC服务器暴露了 Thrift API，可采用不同语言调用。本地或远程运行DRPC服务器是同样的API。
> 将``createRemoteTopology``方法替代``createLocalTopology``，使用 Storm 配置中的 DRPC 配置信息来提交 topology 到 Storm 集群中。
