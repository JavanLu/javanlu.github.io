---
layout: post
title: "[译]【Storm入门指南】第七章 在 Storm 中使用非 JVM 语言"
date: 2013-10-17 10:09
comments: true
categories: Storm
published: true
toc: true
---

> 特别注明：本文翻译自 [Getting started with Storm](http://shop.oreilly.com/product/0636920024835.do) 第七章，以作学习交流之用，非盈利性质。如需转载，请以超链接形式标明文章原始出处和作者信息及版权声明。

有时候你希望使用非JVM语言来实现一个 Storm工程，也许是你使用其他语言感觉更舒服，或者是你想使用其他语言的开发的库。

Storm 是用 Java 实现的，你之前所看到的本书那些示例 spout 和 bolt 也是用Java来编码的。所以能用能用 Python、Ruby 或者 JavaScript 来编写呢？答案是肯定的。通过使用**多语言协议**（*multilang protocol*） 来实现这种可能。

<!-- more -->

多语言协议是 Storm 实现的一个特殊协议，使用标准输入输出作为进程间通信通道来完spout和bolt的处理工作。消息被编码成JSON或者简单的文本通过该通道进行传递。

让我们来看一下一个简单的使用非JVM语言编写的 spout 和 bolt 的实例。有一个用来产生1到10000之间随机数的 spout 和一个过滤质数的 bolt，它们都是用PHP编写的。

> 本例中，检查质数的方式很幼稚。有很多更好的实现，但是它们太复杂，不在本例讨论的范围内。

有一个针对Storm 的 PHP DLS 官方实现。在本章，我们将展示我们的实现。首先，定义 topology。

```java
...
TopologyBuilder builder = new TopologyBuilder();
builder.setSpout("numbers-generator", new NumberGeneratorSpout(1, 10000));
builder.setBolt("prime-numbers-filter", new
PrimeNumbersFilterBolt()).shuffleGrouping("numbers-generator");
StormTopology topology = builder.createTopology();
...
```

> 有一种方式来明确非JVM语言的 topology。由于 Storm topology 是 Thrift 架构，且 Nimbus 是一个 Thrift 守护进程，因此你可以使用任何你想用的语言来创建和提交 topology。但有关讨论超出了本书的范围。

没有任何新东西。来看看``NumbersGeneratorSpout``的实现。

```java
public class NumberGeneratorSpout extends ShellSpout implements IRichSpout {
	public NumberGeneratorSpout(Integer from, Integer to) {
		super("php", "-f", "NumberGeneratorSpout.php", from.toString(), to.toString());
	}

	public void declareOutputFields(OutputFieldsDeclarer declarer) {
		declarer.declare(new Fields("number"));
	}

	public Map<String, Object> getComponentConfiguration() {
		return null;
	}
}
```

你可能注意到，这个 spout 继承了 ``ShellSpout``。这是一个来自 Storm 的特殊类，可以帮助运行和控制用其他语言编写的 spout。在本例中，它告诉 Storm 如何执行 PHP 脚本。

``NumberGeneratorSpout``的 PHP 脚本发射 tuple到标准输出，读取标准输入来处理应答和失败。

在浏览``NumberGeneratorSpout.php``脚本实现之前，我们来看看更多多语言协议的工作细节。

Spout 产生从``from``到``to``的序列号，然后传递给构造函数。

接着，来看看``PrimeNumbersFilterBolt``。这个方法实现了之前提到的接口。它告诉 Storm 如何执行 PHP 脚本。Storm 提供了特殊的类``ShellBolt``来实现该功能，你需要做的唯一的事情是声明怎样运行脚本和发射字段。

```java
public class PrimeNumbersFilterBolt extends ShellBolt implements IRichBolt {
	public PrimeNumbersFilterBolt() {
		super("php", "-f", "PrimeNumbersFilterBolt.php");
	}

	public void declareOutputFields(OutputFieldsDeclarer declarer) {
		declarer.declare(new Fields("number"));
	}
}
```
在构造函数中，只需要告诉 Storm 如何运行 PHP 脚本。这等于执行如下脚本命令：

```sh
php -f PrimeNumbersFilterBolt.php
```

``PrimeNumbersFilterBolt``的 PHP 脚本从标准输入读取 tuple，处理，发射，应答或失败到标准输出。在浏览``PrimeNumbersFilterBolt.php``脚本之前，让我们来看看多语言协议是如何工作的。


# 7.1 详解多语言协议

该协议依赖标准输入输出作为线程间通信的信道。一个脚本需要按照如下步骤来完成工作：

- 1. 初始握手。

- 2. 开始循环。

- 3. 读写 tuple。

让我们详细查看每一步，以及如何使用PHP脚本去实现。

## <strong>7.1.1 初始握手</strong>

Storm 为了控制处理（启动或结束），需要知道正在运行脚本的进程 ID。根据多语言协议，当进程启动时做的第一件事就是发送一个包含了 Storm 配置、topology 上下文以及 PID 目录的 JSON 对象到标准输入。这个对象有点像如下代码块：

```json
{
"conf": {
	"topology.message.timeout.secs": 3,
	// etc
	},
	"context": {
		"task->component": {
			"1": "example-spout",
			"2": "__acker",
			"3": "example-bolt"
		},
		"taskid": 3
	},
	"pidDir": "..."
}
```

进程必须在指定的``pidDir``目录下创建一个以进程ID命名的空文件，将 PID 以 JSON 对象的方式写到标准输出中。

```json
{"pid": 1234}
```

比如，当接收到``/tmp/exmaple/``且脚本的 PID 是``123``，你应该在``/tmp/example/123``下创建一个空文件并且打印``{"pid":123}``和``end``到标准输出。Storm 就是这样在关闭进程的时候来持续跟踪 PID 和终止进程的。让我们来看看 PHP 是如何做到的：

```php
$config = json_decode(read_msg(), true);
$heartbeatdir = $config['pidDir'];
$pid = getmypid();
fclose(fopen("$heartbeatdir/$pid", "w"));
storm_send(["pid"=>$pid]);
flush();
```

你已经建立了``read_msg``功能从标准输入来读取消息。多语言协议允许消息可以被编码成JSON的单行或多行。当 Storm 发送单条单词``end``时，一条消息就完成了。

```php
function read_msg() {
	$msg = "";
	while(true) {
		$l = fgets(STDIN);
		$line = substr($l,0,-1);
		if($line=="end") {
			break;
		}
		$msg = "$msg$line\n";
	}
	return substr($msg, 0, -1);
}

function storm_send($json) {
	write_line(json_encode($json));
	write_line("end");
}

function write_line($line) {
	echo("$line\n");
}
```

> 调用``flush``是非常重要的。可能存在一个直到字符数累计到一定数量才会被刷新的缓冲区。这意味着你的脚本可能会被一直挂起，等待来自Storm的输入。它永远不会被接收，因为Storm在等待来自你脚本的输出。因此确保当你的脚本有输出时及时刷新是重要的。


## <strong>7.1.2 开始循环并读写 Tuple</strong>

这是最重要的一步，在这里所有工作被完成。该步的实现取决于你在开发一个 spout 或是 bolt。

在 spout 情形下，你需要开始发射 tuple。在 bolt 情形下，循环读取tuple、处理并发射它们、ack 或 fail。

让我们看看 spout 发射数字的实现。

```php
	$from = intval($argv[1]);
	$to = intval($argv[2]);
	while(true) {
		$msg = read_msg();
		$cmd = json_decode($msg, true);
		if ($cmd['command']=='next') {
			if ($from<$to) {
				storm_emit(array("$from"));
				$task_ids = read_msg();
				$from++;
			} else {
				sleep(1);
			}
		}
		storm_sync();
	}
```

从命令行中获取``from``和``to``参数，然后开始循环。每当从 Storm 获取到一个``next``消息，意味着准备开始发射一个新的 tuple。

当发射完所有的数字并且没有更多的 tuple 需要发送的话，休眠一段时间。

为了保证脚本已经为准备好下一个 tuple，在发射下一个之前等待``sync``指令。使用``read_msg()``读取指令并按照 JSON 格式编码。

在本例中，bolt 有所不同。

```php
while(true) {
	$msg = read_msg();
	$tuple = json_decode($msg, true, 512, JSON_BIGINT_AS_STRING);
	if (!empty($tuple["id"])) {
		if (isPrime($tuple["tuple"][0])) {
			storm_emit(array($tuple["tuple"][0]));
		}
		storm_ack($tuple["id"]);
	}
}
```
循环从标准输入读取 tuple。获取到消息后，使用 JSON 格式解码。如果是一个 tuple，处理它并检查是否是一个质数。如果是质数，将数字发射出去；否则忽略。

不论如何，都对 tuple 作应答。

> 在	``json_decode``中使用``JSON_BIGINT_AS_STRING``参数，是为了解决 Java 和 PHP 之间的转换问题。Java 发送某些很大的数字，然而这些数字在 PHP 解析时会丢失精度，这就有问题了。为了解决这个问题，告诉 PHP 把大数字解析成字符串，并且在JSON消息中禁止对该字符串使用双引号。PHP 5.4.0 或者更高版本，需要这个参数才能正常工作。


``emit``、``ack``、``fail``和``log``的格式如下：

**Emit**

```json
{
	"command": "emit",
	"tuple": ["foo", "bar"]
}
```

tuple 数组中是发射的值。

**Ack**

```json
{
	"command": "ack",
	"id": 123456789
}
```

id 是要处理的 tuple 的 ID。

**Fail**

```json
{
	"command": "fail",
	"id": 123456789
}
```

和 Ack 一样，id 是要处理的 tuple 的 ID。

**Log**

```json
{
	"command": "log",
	"msg": "some message to be logged by storm."
}
```

下面给出所有的 PHP 脚本。

Spout:

```php
<?php
function read_msg() {
	$msg = "";
	while(true) {
		$l = fgets(STDIN);
		$line = substr($l,0,-1);
		if ($line=="end") {
			break;
		}
		$msg = "$msg$line\n";
	}
	return substr($msg, 0, -1);
}

function write_line($line) {
	echo("$line\n");
}

function storm_emit($tuple) {
	$msg = array("command" => "emit", "tuple" => $tuple);
	storm_send($msg);
}

function storm_send($json) {
	write_line(json_encode($json));
	write_line("end");
}

function storm_sync() {
	storm_send(array("command" => "sync"));
}

function storm_log($msg) {
	$msg = array("command" => "log", "msg" => $msg);
	storm_send($msg);
	flush();
}

$config = json_decode(read_msg(), true);
$heartbeatdir = $config['pidDir'];

$pid = getmypid();
fclose(fopen("$heartbeatdir/$pid", "w"));
storm_send(["pid"=>$pid]);
flush();

$from = intval($argv[1]);
$to = intval($argv[2]);

while(true) {
	$msg = read_msg();
	$cmd = json_decode($msg, true);
	if ($cmd['command']=='next') {
		if ($from<$to) {
			storm_emit(array("$from"));
			$task_ids = read_msg();
			$from++;
		} else {
			sleep(1);
		}
	}
	storm_sync();
}
?>
```

Bolt:

```php
<?php
function isPrime($number) {
	if ($number < 2) {
		return false;
	}
	if ($number==2) {
		return true;
	}
	for ($i=2; $i<=$number-1; $i++) {
		if ($number % $i == 0) {
			return false;
		}
	}
	return true;
}

function read_msg() {
	$msg = "";
	while(true) {
		$l = fgets(STDIN);
		$line = substr($l,0,-1);
		if ($line=="end") {
			break;
		}
		$msg = "$msg$line\n";
		}
	return substr($msg, 0, -1);
}

function write_line($line) {
	echo("$line\n");
}

function storm_emit($tuple) {
	$msg = array("command" => "emit", "tuple" => $tuple);
	storm_send($msg);
}

function storm_send($json) {
	write_line(json_encode($json));
	write_line("end");
}

function storm_ack($id) {
	storm_send(["command"=>"ack", "id"=>"$id"]);
}

function storm_log($msg) {
	$msg = array("command" => "log", "msg" => "$msg");
	storm_send($msg);
}

$config = json_decode(read_msg(), true);
$heartbeatdir = $config['pidDir'];

$pid = getmypid();
fclose(fopen("$heartbeatdir/$pid", "w"));
storm_send(["pid"=>$pid]);
flush();

while(true) {
	$msg = read_msg();
	$tuple = json_decode($msg, true, 512, JSON_BIGINT_AS_STRING);
	if (!empty($tuple["id"])) {
		if (isPrime($tuple["tuple"][0])) {
			storm_emit(array($tuple["tuple"][0]));
		}
		storm_ack($tuple["id"]);
	}
}
?>
```

> 务必将上面所有脚本放到工程的``multilang/resources``目录下。这个目录会被包含在JAR包中发送到工作者。如果你没有将脚本放置到该目录，Storm 将会出错且不会运行它们。


