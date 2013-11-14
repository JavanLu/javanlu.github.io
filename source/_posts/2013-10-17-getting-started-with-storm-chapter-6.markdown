---
layout: post
title: "[译]【Storm入门指南】第六章 真实示例"
date: 2013-10-17 08:56
comments: true
categories: Storm
published: true
toc: true
---

> 特别注明：本文翻译自 [Getting started with Storm](http://shop.oreilly.com/product/0636920024835.do) 第六章，以作学习交流之用，非盈利性质。如需转载，请以超链接形式标明文章原始出处和作者信息及版权声明。

本章将演示一个典型的网页分析方案，通常使用 Hadoop 批量作业来解决的问题。不像 Hadoop 的实现方案，基于 Storm 的解决方案实时刷新并呈现结果。

示例有三个主要部分（如图6.1所示）：

- 一个 Node.js 的web应用，用来测试系统

- 一个 Redis 服务器，用来持久化数据

- 一个 Storm topology，用于实时分布式数据处理

<!-- more -->

![图6.1](/images/post/storm/chapter6/fig6_1.png)

> 如果你想在通览本章的同时运行实例，你应该先阅读[附录C](http://javanlu.github.io/blog/2013/10/16/getting-started-with-storm-appendix-c/)。

# 6.1 Node.js Web应用

我们模拟一个简单的电商网站的三个网页：主页、产品页和产品统计页。该应用使用[Express Framework](http://expressjs.com)和[Socket.io Framework](http://socket.io)将更新推送到浏览器。本应用的目的是让你运行集群并看到结果。但这不是本书的重点，所以我们除了页面本身的描述，不会深入其他任何细节。

- 主页
  ：本页提供平台上所有可用商品的链接来简化导航。它从Redis服务器读取并列出所有的项目。本页的URL是 http://localhost:3000/ （如图6.2所示）
  
![图6.2](/images/post/storm/chapter6/fig6_2.png)

- 产品页
  ：商品页显示与特定商品相关的信息，比如价格、商品标题和分类。本页的URL是 http://localhost:3000/product/:id （如图6.3所示）

![图6.3](/images/post/storm/chapter6/fig6_3.png)

- 商品统计页
  ：本页显示Storm集群计算出的相关信息，这些信息在用户浏览网页的时候被收集到。可以总结为如下流程：浏览了本产品的用户同时浏览了 n 次同类目下的产品。本页的URL是 http://localhost:3000/product/:id/stats (如图6.4所示)

![图6.4](/images/post/storm/chapter6/fig6_4.png)


# 6.2 启动 Node.js Web应用

启动 Redis 服务器之后，在项目目录下使用如下命令来运行web应用：

```sh
node webapp/app.js
```

这个web应用将自动往 Redis 中生成一些示例产品供你使用。

# 6.3 Storm Topology

本系统中 Storm Topology 的目标是：在用户浏览网页的同事，实时更新产品统计信息。产品统计页展示关联了统计的类目列表，显示浏览同一类目下其他产品的用户数。这将帮助销售人员理解客户的需求。如图6.5所示，Topology 接收浏览日志、更新产品统计信息。

![图6.5](/images/post/storm/chapter6/fig6_5.png)

该Storm Topology由五个组成部分：一个用于供给的 spout 和四个用于完成工作的 bolt。

- *UsersNavigationSpout*
  ：从用户浏览队列读取信息并发送至 topology 

- *GetCategoryBolt*
  ：从 Redis 服务器读取产品信息并添加其类目到流中

- *UserHistoryBolt*
  ：读取用户之前浏览的产品，发射 产品:类目 对来更新下一步的计数次

- *ProductCategoriesCounterBolt* 
  ：持续跟踪特定目录下用户浏览某个商品的次数

- *NewsNotifierBolt*
  ：通知web应用立即更新用户接口

如下展示了如何创建topology（如图6.6所示）：

![图6.6](/images/post/storm/chapter6/fig6_6.png)

```java
package storm.analytics;
...

public class TopologyStarter {
       
        public static void main(String[] args) {
        Logger.getRootLogger().removeAllAppenders();

                TopologyBuilder builder = new TopologyBuilder();
        
        builder.setSpout("read-feed", new UsersNavigationSpout(), 3);
        
        builder.setBolt("get-categ", new GetCategoryBolt(), 3)
                                        .shuffleGrouping("read-feed");
        
        builder.setBolt("user-history", new UserHistoryBolt(), 5)
                                        .fieldsGrouping("get-categ", new Fields("user"));
        
        builder.setBolt("product-categ-counter", new ProductCategoriesCounterBolt(), 5)
                                        .fieldsGrouping("user-history", new Fields("product"));
        
        if(!testing)
                builder.setBolt("news-notifier", new NewsNotifierBolt(), 5)
                                        .shuffleGrouping("product-categ-counter");
        
        Config conf = new Config();
        conf.setDebug(true);

        conf.put("redis-host", REDIS_HOST);
        conf.put("redis-port", REDIS_PORT);
        conf.put("webserver", WEBSERVER);
        conf.put("download-time", DOWNLOAD_TIME);
        
        LocalCluster cluster = new LocalCluster();
        cluster.submitTopology("analytics", conf, builder.createTopology());
        }
	...
}

```

## <strong>6.3.1 UsersNavigationSpout</strong>

``UsersNavigationSpout``负责向topology输送浏览记录。每条浏览记录是一个某用户浏览某产品的引用。它们通过web应用保存在Redis 服务器上。后面你将看到更多这方面的细节。

使用[Jedis](https://github.com/xetorthio/jedis)从Redis 服务器读取记录。Jedis 是一个极小极简的Redis Java客户端。

> 只有相关部分展示在如下代码中。

```java
package storm.analytics;
public class UsersNavigationSpout extends BaseRichSpout {
	Jedis jedis;
	...
	@Override
	public void nextTuple() {
		String content = jedis.rpop("navigation");
		if(content==null || "nil".equals(content)) {
			try { Thread.sleep(300); } catch (InterruptedException e) {}
		} else {
			JSONObject obj=(JSONObject)JSONValue.parse(content);
			String user = obj.get("user").toString();
			String product = obj.get("product").toString();
			String type = obj.get("type").toString();
			HashMap<String, String> map = new HashMap<String, String>();
			map.put("product", product);
			NavigationEntry entry = new NavigationEntry(user, type, map);
			collector.emit(new Values(user, entry));
		}
	}

	@Override
	public void declareOutputFields(OutputFieldsDeclarer declarer) {
		declarer.declare(new Fields("user", "otherdata"));
	}
}

```

该 spout 首先调用``jedis.rpop("navigation")``来删除和返回 Redis 服务器中 navigation 列表中最右的元素。如果列表为空，休眠0.3秒来避免由于忙空等循环阻塞了服务器。如果找到一个元素，转化内容将其映射到``NavigationEntry``对象上，该对象是一个包含了如下内容的 POJO。

- 浏览的用户

- 用户浏览的页面类型

- 取决于页面类型的附加页面信息。“PRODUCT”页面类型拥有一个被浏览的产品ID项。

spout 调用 ``collector.emit(new Values(user, entry))``来发射包含上述信息的 tuple。tuple 中的内容是topology中下一个 bolt[``GetCategoryBolt``]的输入。


## <strong>6.3.2 GetCategoryBolt</strong>
 
这是个非常简单的 bolt。它主要职责是反序列化前一个 spout 发射的内容。如果是关于产品页面的信息，将使用``ProductReader``帮助类从Redis服务器加载产品信息。接着，对于输入的每个 tuple，发射一个具有更多产品特殊信息的新的 tuple。这些信息包括：

- 用户

- 产品

- 产品类目


```java
package storm.analytics;

public class GetCategoryBolt extends BaseBasicBolt {
	private ProductsReader reader;
	...
	@Override
	public void execute(Tuple input, BasicOutputCollector collector) {
		NavigationEntry entry = (NavigationEntry)input.getValue(1);
		if("PRODUCT".equals(entry.getPageType())){
			try {
				String product = (String)entry.getOtherData().get("product");
				// Call the items API to get item information
				Product itm = reader.readItem(product);
				if(itm ==null)
					return ;
				String categ = itm.getCategory();
				collector.emit(new Values(entry.getUserId(), product, categ));
			} catch (Exception ex) {
				System.err.println("Error processing PRODUCT tuple"+ ex);
				ex.printStackTrace();
			}
		}
	}
...
}
```

如前所述，使用 ``ProductReader``帮助类读取产品的特殊信息：

```java
package storm.analytics.utilities;
	...
	public class ProductsReader {
	...
	public Product readItem(String id) throws Exception{
		String content= jedis.get(id);
		if(content == null || ("nil".equals(content)))
			return null;
		Object obj=JSONValue.parse(content);
		JSONObject product=(JSONObject)obj;

		Product i= new Product((Long)product.get("id"),
								(String)product.get("title"),
								(Long)product.get("price"),
								(String)product.get("category"));
		return i;
	}
	...
}

```

## <strong>6.3.3 UserHistoryBolt</strong>

``UserHistoryBolt``是该应用的核心。它负责保持跟踪每个用户浏览的产品以及决定哪些是应该被增加的结果对。

你将使用 Redis 服务器来按用户存储产品的历史记录，出于性能原因，在本地维护一个存储的拷贝。你通过``getUserNavigationHistory(user)`` 和``addProductToHistory(user,prodKey)``方法分别来读写，以此隐藏了数据的访问细节。


```java
package storm.analytics;

...
public class UserHistoryBolt extends BaseRichBolt{
	@Override
	public void execute(Tuple input) {
		String user = input.getString(0);
		String prod1 = input.getString(1);
		String cat1 = input.getString(2);
		// Product key will have category information embedded.
		String prodKey = prod1+":"+cat1;
		Set<String> productsNavigated = getUserNavigationHistory(user);
		// If the user previously navigated this item -> ignore it
		if(!productsNavigated.contains(prodKey)) {
			// Otherwise update related items
			for (String other : productsNavigated) {
				String [] ot = other.split(":");
				String prod2 = ot[0];
				String cat2 = ot[1];
				collector.emit(new Values(prod1, cat2));
				collector.emit(new Values(prod2, cat1));
			}
			addProductToHistory(user, prodKey);
		}
	}
}
```

注意该bolt期望的输出是发射类目关系应该被增加的产品。

看一下源码。该bolt为每个用户保存产品浏览列表。相比于只包含产品信息，这个列表包含产品：类目 对。因为后续调用中需要类目信息，这会比每次从数据库获取信息具有更好的性能。这是可能的，因为产品只属于一个类目，并且在产品的生命周期中不会改变。

在读取用户之前浏览的产品列表（包括类目信息）之后，检查当前的产品是否之前被浏览过。如果是，忽略该次记录。如果这是用户第一次浏览该产品，遍历用户的历史浏览记录并使用``collector.emit(new Values(prod1, cat2))``方法发送一个包含当前正在被浏览的产品及历史浏览记录中所有产品的类目信息的tuple，使用``collector.emit(new Values(prod2, cat1))``发送包含其他产品及正在被浏览的产品所属类目的另一个元组。最后，添加产品及它的类目到集合中。

比如，用户 John 有如下浏览历史：

![图6.7](/images/post/storm/chapter6/fig6_7.png)

需要处理的浏览记录如下：

![图6.8](/images/post/storm/chapter6/fig6_8.png)

用户没有查看产品8，所以需要处理它。

所以发射的 tuple 将如下所示：

![图6.9](/images/post/storm/chapter6/fig6_9.png)

注意左边的产品和右边的类目之间的关系应该在同一个单元中被增加。

现在，我们来看看Bolt中的持久化工作。

```java
public class UserHistoryBolt extends BaseRichBolt{
	...
	private Set<String> getUserNavigationHistory(String user) {
		Set<String> userHistory = usersNavigatedItems.get(user);
		if(userHistory == null) {
			userHistory = jedis.smembers(buildKey(user));
			if(userHistory == null)
				userHistory = new HashSet<String>();
				usersNavigatedItems.put(user, userHistory);
		}
		return userHistory;
	}

	private void addProductToHistory(String user, String product) {
		Set<String> userHistory = getUserNavigationHistory(user);
		userHistory.add(product);
		jedis.sadd(buildKey(user), product);
	}
	...
}
```

``getUserNavigationHistory``方法返回用户浏览的产品列表。首先，调用``usersNavigatedItems.get(user)``来尝试从本地内存中获取用户的历史。如果没获取到，调用``jedis.smembers(buildKey(user))``方法从Redis服务器读取，并以``usersNavigatedItems``格式将其保存到内存中。

当用户浏览一个新的产品，调用``addProductToHistory``来更新内存和Redis服务器内容。

需要注意的是，既然bolt在内存中按用户保存信息，那么当并行化bolt时，在第一级对用户使用``FieldGrouping``是非常重要的，否则用户历史记录的不同拷贝将不同步。

## <strong>6.3.4 ProductCategoriesCounterBolt</strong>

``ProductCategoriesCounterBolt``类负责跟踪所有的产品-类目关系。它接收``UserHistoryBolt``反射的产品-类目对，更新计数器。

每个键值对出现次数的信息被保存在Redis服务器中。处于性能考虑，使用一个本地缓存来读写缓存。信息通过一个后台进程被发送到Redis。

对于输入，该bolt也会发送一个包含了更新的计数器的元组来供topology中下一个bolt——``NewsNotifierBolt``消费，它用于广播消息到最终用户来用于实时更新。

```java

public class ProductCategoriesCounterBolt extends BaseRichBolt {
	...
	@Override
	public void execute(Tuple input) {
		String product = input.getString(0);
		String categ = input.getString(1);
		int total = count(product, categ);
		collector.emit(new Values(product, categ, total));
	}

	...
	private int count(String product, String categ) {
		int count = getProductCategoryCount(categ, product);
		count ++;
		storeProductCategoryCount(categ, product, count);
		return count;
	}
	...
}
```

该bolt的持久化隐藏在``getProductCategoryCount``和``storeProductCategoryCount``方法。让我们深入看看：

```java

package storm.analytics;
...
public class ProductCategoriesCounterBolt extends BaseRichBolt {
	// ITEM:CATEGORY -> COUNT
	HashMap<String, Integer> counter = new HashMap<String, Integer>();
	
	// ITEM:CATEGORY -> COUNT
	HashMap<String, Integer> pendingToSave = new HashMap<String, Integer>();
	...
	
	public int getProductCategoryCount(String categ, String product) {
		Integer count = counter.get(buildLocalKey(categ, product));
		if(count == null) {
			String sCount = jedis.hget(buildRedisKey(product), categ);
			if(sCount == null || "nil".equals(sCount)) {
				count = 0;
			} else {
				count = Integer.valueOf(sCount);
			}
		}
		return count;
	}

	...
	private void storeProductCategoryCount(String categ, String product, int count) {
		String key = buildLocalKey(categ, product);
		counter.put(key , count);
		synchronized (pendingToSave) {
			pendingToSave.put(key, count);
		}
	}
...
}
```

``getProductCategoryCount``方法首先从内存缓存中查找计数。如果不可用，将从Redis服务器查找。

``storeProductCategoryCount``方法更新计数器缓存和``pendingToSave``缓存。该缓存由如下后台进程来持久化：

```java
package storm.analytics;

public class ProductCategoriesCounterBolt extends BaseRichBolt {
	...
	private void startDownloaderThread() {
		TimerTask t = new TimerTask() {
			@Override
			public void run() {
				HashMap<String, Integer> pendings;
				synchronized (pendingToSave) {
					pendings = pendingToSave;
					pendingToSave = new HashMap<String, Integer>();
				}
				for (String key : pendings.keySet()) {
					String[] keys = key.split(":");
					String product = keys[0];
					String categ = keys[1];
					Integer count = pendings.get(key);
					jedis.hset(buildRedisKey(product), categ, count.toString());
				}
			}
		};
		timer = new Timer("Item categories downloader");
		timer.scheduleAtFixedRate(t, downloadTime, downloadTime);
	}
	...
}
```

下载线程锁定``pendingToSave``，当它发送旧的buffer到Redis的同时建立新的缓冲区来供其他线程使用。该代码块每隔 downloadTime 毫秒运行一次并且可以通过 down-load topology 配置参数来配置。download-time 越长，执行Redis写的次数越少，因为连续的添加被一次写入。

再次牢记，正如在前边的bolt中一样，当分配资源到该bolt时，应用正确的域分组是非常重要的，在该例中根据产品分组。那是因为它按产品存储该信息的内存拷贝，并且如果一些缓存和buffer的拷贝存在，将出现不一致。

## <strong>6.3.5 NewsNotifierBolt</strong>

``NewsNotifierBolt``负责通知web应用统计数据的变化，为的是让用户可以看到实时的变化。使用 Apache HttpClient 的HTTP POST来发送通知，发送到topology中配置的web服务器的URL地址。POST体被编码成JSON。

该bolt在测试时将会被从 topology 中删除。

```java
package storm.analytics;
...
public class NewsNotifierBolt extends BaseRichBolt {
	...
	@Override
	public void execute(Tuple input) {
		String product = input.getString(0);
		String categ = input.getString(1);
		int visits = input.getInteger(2);
		String content = "{ \"product\": \""+product+"\", \"categ\":\""+categ+"\",
		\"visits\":"+visits+" }";
		HttpPost post = new HttpPost(webserver);
		try {
			post.setEntity(new StringEntity(content));
			HttpResponse response = client.execute(post);
			org.apache.http.util.EntityUtils.consume(response.getEntity());
		} catch (Exception e) {
			e.printStackTrace();
			reconnect();
		}
	}
	...
}
```

# 6.4 Redis 服务器

Redis是一个先进的用于持久化的内存键值存储系统。使用它存储如下信息：

- 服务于网页的产品信息；

- 用于供给Storm Topology 的用户浏览队列；

- 用于Topology从失败中恢复的 Storm Topology 中间数据；

- 用于存储预期结果的Storm Topology 结果。

## <strong>6.4.1 产品信息</strong>

Redis服务器存储产品，使用产品ID作为键，包含产品所有的信息的JSON对象作为值。

```sh
> redis-cli
redis 127.0.0.1:6379> get 15
"{\"title\":\"Kids smartphone cover\",\"category\":\"Covers\",\"price\":30,\"id\":
15}"
```

## <strong>6.4.2 用户浏览队列</strong>

用户浏览队列存储在一个名为导航的先进先出的Redis列表。当用户浏览一个产品页面的时候，服务器想列表的左边添加一条记录用来表明哪个用户浏览了哪个产品。Storm 集群持续从列表的最右端删除记录，来处理该信息。

```sh
redis 127.0.0.1:6379> llen navigation
(integer) 5
redis 127.0.0.1:6379> lrange navigation 0 4
1) "{\"user\":\"59c34159-0ecb-4ef3-a56b-99150346f8d5\",\"product\":\"1\",\"type\":
\"PRODUCT\"}"
2) "{\"user\":\"59c34159-0ecb-4ef3-a56b-99150346f8d5\",\"product\":\"1\",\"type\":
\"PRODUCT\"}"
3) "{\"user\":\"59c34159-0ecb-4ef3-a56b-99150346f8d5\",\"product\":\"2\",\"type\":
\"PRODUCT\"}"
4) "{\"user\":\"59c34159-0ecb-4ef3-a56b-99150346f8d5\",\"product\":\"3\",\"type\":
\"PRODUCT\"}"
5) "{\"user\":\"59c34159-0ecb-4ef3-a56b-99150346f8d5\",\"product\":\"5\",\"type\":
\"PRODUCT\"}"
```

## <strong>6.4.3 中间数据</strong>

集群需要为每个用户分开存储历史数据。为了实现这样，它在Redis服务器中保存一个集合，该集合中包含每个用户浏览过的所有的产品以及对应的类目。

```java
redis 127.0.0.1:6379> smembers history:59c34159-0ecb-4ef3-a56b-99150346f8d5
1) "1:Players"
2) "5:Cameras"
3) "2:Players"
4) "3:Cameras"
```

## <strong>6.4.4 结果</strong>

集群产生用户访问特定产品的有用数据并且将它们存储在命名为”procnt”的Redis Hash中：后边紧跟着产品ID。

```java
redis 127.0.0.1:6379> hgetall prodcnt:2
1) "Players"
2) "1"
3) "Cameras"
4) "2"
```

# 6.5  测试 Topology

为了测试topology，使用提供的 ``LocalCluster``和一个本地Redis服务器。将在初始化时填充产品数据库并且在Redis服务器上模拟浏览日志的插入。我们的断言将通过读取topology输出到Redis服务器来执行。测试用Java和Groovy编写。

![图6.10](/images/post/storm/chapter6/fig6_10.png)

## <strong>6.5.1 结果初始化</strong>

初始化包括三步：

1. **启动LocalCluster，提交Topology**。初始化在``AbstractAnalyticsTest``中实现，该类被所有测试继承。一个叫做topologyStarted的静态标志被用来避免当多个``AbstractAnalyticsTest``子类初始化时``AbstractAnalyticsTest``本身被初始化不止一次的情况。

注意sleep的目的是允许LocalCluster在尝试从中恢复结果之前正确的启动。

```java
public abstract class AbstractAnalyticsTest extends Assert {
	def jedis
	static topologyStarted = false
	static sync= new Object()
	
	private void reconnect() {
		jedis = new Jedis(TopologyStarter.REDIS_HOST, TopologyStarter.REDIS_PORT)
	}

	@Before
	public void startTopology(){
		synchronized(sync){
			reconnect()
			if(!topologyStarted){
				jedis.flushAll()
				populateProducts()
				TopologyStarter.testing = true
				TopologyStarter.main(null)
				topologyStarted = true
				sleep 1000
			}
		}
	}

	...
	public void populateProducts() {
		def testProducts = [
			[id: 0, title:"Dvd player with surround sound system",
			category:"Players", price: 100],
			[id: 1, title:"Full HD Bluray and DVD player",
			category:"Players", price:130],
			[id: 2, title:"Media player with USB 2.0 input",
			category:"Players", price:70],
			...
			[id: 21, title:"TV Wall mount bracket 50-55 Inches",
			category:"Mounts", price:80]
		]

		testProducts.each() { product ->
		def val =
		"{ \"title\": \"${product.title}\" , \"category\": \"${product.category}\"," +
		" \"price\": ${product.price}, \"id\": ${product.id} }"
		println val
		jedis.set(product.id.toString(), val.toString())
		}
	}
	...
}
```


2. **在AbstractAnalyticsTest类中实现一个叫做navigate的方法**。为了使不同的测试有一种来模拟用户导航页面行为的方式，该步在Redis服务器导航队列中插入导航项。

```java

public abstract class AbstractAnalyticsTest extendsAssert {
	...
	public voidnavigate(user,product) {
	
		String nav = "{\"user\": \"${user}\",\"product\": \"${product}\", \"type\":\"PRODUCT\"}".toString()
		println "Pushingnavigation: ${nav}"
		jedis.lpush('navigation',nav)
	
	}
	...
}
```

3. **在AbstractAnalyticsTest中提供一个叫做getProductCategory的方法来从Redis服务器中读取特定的关系**。不同的测试也需要对统计的结果进行断言来确保topology按预期的运行。

```java
public abstract class AbstractAnalyticsTest extends Assert {
	...
	public int getProductCategoryStats(String product, String categ) {
		String count = jedis.hget("prodcnt:${product}", categ)
		if(count == null || "nil".equals(count))
			return 0
		return Integer.valueOf(count)
	}
	...
}
```

## <strong>6.5.2 一个测试用例</strong>

在下面的片段中，将模拟用户'1'的一部分产品浏览记录，然后检查结果。注意在断言确定存储到Redis的结果之前，需要等待两秒钟。(需要记住的是``ProductCategoriesCounterBolt``包含一个计数器的内存拷贝并且在后台将他们发送至Redis)。

```java

package functional
class StatsTest extends AbstractAnalyticsTest {
	@Test
	public void testNoDuplication(){
		navigate("1", "0") // Players
		navigate("1", "1") // Players
		navigate("1", "2") // Players
		navigate("1", "3") // Cameras
		Thread.sleep(2000) // Give two seconds for the system to process the data.
		assertEquals 1, getProductCategoryStats("0", "Cameras")
		assertEquals 1, getProductCategoryStats("1", "Cameras")
		assertEquals 1, getProductCategoryStats("2", "Cameras")
		assertEquals 2, getProductCategoryStats("0", "Players")
		assertEquals 3, getProductCategoryStats("3", "Players")
	}
}
```

# 6.6 扩展性和可用性的说明

该解决方案的架构被精简用来在一章中可以描述之。由于这个原因，规避了一些对于该方案的高可用性和扩张性复杂性来说必须的复杂度。该架构主要有两个问题。

架构中的Redis服务器不仅仅是单点失败并且是一个瓶颈。你只能获取Redis服务器所能处理的数据量。通过使用切片，Redis层可以被扩展。并且，它的可用性可以通过使用一个主/从配置来改进，这需要topology和web应用资源都做出改变。

另一个缺点是以循环方式添加机器时，WEB应用并没有成比例地扩展。这是因为，当一些产品统计信息变化时，它需要被通知到，且通知搜有相应的浏览器。在这里，"通知浏览器"桥接通过Socket.io来实现。但是它需要监听器和通知器需要部署到同一台web服务器。这只有在共享``GET /product/:id/stats ``通信和``POST /news``通信、并且使用相同标准、确保引用相同产品的请求在同台服务器上结束的情况下才能做到。
