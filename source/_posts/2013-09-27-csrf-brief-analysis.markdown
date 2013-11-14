---
layout: post
title: "浅析CSRF"
date: 2013-09-27 10:37
comments: true
categories: CSRF Security

---

在进行WEB开发的时候，时常听到资深的开发人员说要注意防范CSRF、XSS攻击，但是很多时候人们连什么是CSRF都不知道。本文将粗略地向大家介绍下CSRF的原理和防御手段。



## 一、<strong>CSRF是什么</strong>

首先，来看看CSRF是什么。依据Wikipedia的解释，含义如下：

{% blockquote Wikipedia http://en.wikipedia.org/wiki/Cross-site_request_forgery  Cross-site request forgery<sup>[1]</sup> %}
CSRF（Cross-site request forgery）跨站请求伪造，也被称成为点击攻击或者会话劫持，通常缩写为CSRF或者XSRF，是一种恶意利用终端用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。XSS利用站点内的信任用户，而CSRF则通过伪装来自受信任用户的请求来利用受信任的网站。
{% endblockquote %}

通俗来讲，CSRF攻击通过在授权用户访问的页面中包含链接或者脚本的方式工作。让我们通过两个例子来对这段晦涩的定义做出解释。

<!--more-->

1. 一个网站用户 Alice 可能正在浏览聊天论坛，而同时另一个用户 Eve 也在此论坛中发布了一个具有 Alice 银行链接的图片消息。设想一下，Eve 编写了一个在 Alice 的银行站点上进行取款的form提交的链接，并将此链接作为图片tag。如果 Alice 的银行在cookie中保存他的授权信息，并且此cookie没有过期，那么当 Alice 的浏览器尝试装载图片时将提交这个取款form和他的cookie，这样在没经 Alice 同意的情况下便授权了这次事务。

2. 2009年黑客利用Gmail的一个CSRF漏洞，成功获取了好莱坞明星Vanessa Hudgens的独家艳照<sup>[2]</sup>。其攻击过程非常简单，给该明星的gmail账户发了一封email，标题是某大导演邀请你来看看这个电影，里面有个图片(该图片实际是Gmail转发的URL)，结果她登录Gmail，打开邮件就默默无闻的中招了，所有邮件被转发到黑客的账号。

通过上面两个例子，可以总结出**CSRF攻击的常见特性**：

 - 依赖使用用户标识的网站；
 - 利用网站对用户标识的信任；
 - 欺骗用户的浏览器发送HTTP请求给目标站点；
 - 依靠具有[side effect](http://en.wikipedia.org/wiki/HTTP#Safe_methods)<sup>[3]</sup>的HTTP请求。

 
**CSRF攻击步骤**如下：

 1. 受害者必须在同一浏览器窗口（即使不是同一tab）内访问并登陆目标站点；
 2. 利用用户有效的Session cookie，从而利用受害者的身份进行恶意操作。 

## 二、<strong>技术分析</strong>

大家可能对CSRF是什么还是一头雾水，那么用前面提到的的银行CSRF攻击的例子从代码层面来展示CSRF攻击是怎么做到的。

### 1. 回合Ⅰ

*银行网站A*：它以GET请求来完成银行转账的操作，格式如： ``http://bank.example.com/transfer.do?money=${val}&to=${BankId}``，其中``${val}``表示金额，``${BankId}``表示收款方的银行卡。

*危险网站B*：它里面有一段HTML的代码如下：
```html
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <title>csrf attack</title>
</head>
<body>
  <img src="http://bank.example.com/transfer.do?moeny=100&to=1"/>
</body>
</html>
```

你登录了银行网站A，然后访问了危险网站B。啊哈，你账户里面的钱被窃取了100元。见鬼了，为什么会这样？这里银行使用GET请求更新资源，在你登录了银行网站A的前提下，访问网站B上的``<img>``就能以你的身份（浏览器中你的银行网站A的Cookie中的信息）以GET的方式发起转账操作，请求了银行的服务器。由于有合法的身份标识，银行网站服务器收到请求后立即进行转账操作。

### 2. 回合Ⅱ

好吧，银行决定采用POST方式来发送请求，后台只接受POST请求来进行更新资源的操作，前台采用``Form``表单的方式进行POST请求。

```html
<form action="transfer.do" method="POST">
　　ToBankId: <input type="text" name="to" />
　　Money: <input type="text" name="money" />
　　<input type="submit" value="Transfer" />
</form>

```

这样感觉安全了，但是攻击者也做出相应的变化，将网站B的页面改成如下内容：

```html
<html>
<head>
	<script type="text/javascript">
		function steal(){
	    	iframe = document.frames["steal"];
	　　    iframe.document.Submit("transfer");
	　　}
	</script>
</head>
<body onload="steal()">
	<iframe name="steal" display="none">
		<form method="POST" name="transfer"　action="http://www.myBank.com/transfer.do">
　　　　		<input type="hidden" name="to" value="1"/>
    		<input type="text" name="money" value="100"/>
　　　　	</form>
　　	</iframe>
</body>
</html>
```

如果用户仍是继续上面的操作，发现钱同样没有了，因为危险网站B暗地里又以你的身份发送了POST请求到银行。

[《Web安全测试之跨站请求伪造(CSRF)篇》](http://netsecurity.51cto.com/art/200811/97281_1.htm)<sup>[4]</sup> 和 [《深入解析跨站请求伪造漏洞：实例讲解》](http://netsecurity.51cto.com/art/200812/102925_1.htm)<sup>[5]</sup> 中还有更多的实例讲解。

## <strong>三、原理剖析</strong>
归根结底，**<u>CSRF攻击是源于WEB的隐式身份验证机制</u>**。**<u>WEB的身份验证机制虽然可以保证一个请求是来自于某个用户的浏览器，但却无法保证该请求是用户批准发送的</u>**。 

{% blockquote 51CTO.COM http://netsecurity.51cto.com/art/200812/102951_1.htm 深入解析跨站请求伪造漏洞：原理剖析<sup>[6]</sup> %}

例如，假如Alice在浏览目标站点T，那么站点T便会给Alice的浏览器一个cookie，用于存放一个伪随机数作为会话标识符sid，以跟踪她的会话。该站点会要求Alice进行登录，当她输入有效的用户名和口令时，该站点会记录这样一个事实：Alice已经登录到会话sid。当Alice发送一个请求到站点T时，她的浏览器就会自动地发送包含sid的会话cookie。之后，站点T就会使用站点的会话记录来识别该会话是否来自Alice。

现在，假设Alice访问了一个恶意站点M，该站点提供的内容中的JavaScript代码或者图像标签会导致Alice的浏览器向站点T发送一个HTTP请求。由于该请求是发给站点T的，所以Alice的浏览器自动地给该请求附上与站点T对应的该会话cookie的sid。站点T看到该请求时，它就能通过该cookie的推断出：该请求来自Alice，所以站点T就会对Alice的帐户执行所请求的操作。这样，CSRF攻击就能得逞了。

其他大多数Web认证机制也面临同样的问题。例如，HTTP BasicAuth机制会要求Alice告诉浏览器她在站点T上的用户名和口令，于是浏览器将用户名和口令附加到之后发给站点T的请求中。当然，站点T也可能使用客户端SSL证书，但这也面临同样的问题，因为浏览器也会将证书附加到发给站点T的请求中。类似的，如果站点T通过IP地址来验证Alice的身份的话，照样面临CSRF攻击的威胁。
{% endblockquote %}

总之，只要身份认证是隐式进行的，就会存在CSRF攻击的危险，因为浏览器发出请求这一动作未必是受用户的指使。原则上，这种威胁可以通过对每个发送至该站点的请求都要求用户进行显式的、不可欺骗的动作（诸如重新输入用户名和口令）来消除，但实际上这会导致严重的易用性问题。


## <strong>四、如何防御</strong>

那么如何防御呢？大体来说，可以从服务端、客户端和安全设备这三个点来进行防御。


### 服务端防御

目前业界服务端防御CSRF攻击主要有三种策略<sup>[7]</sup>：验证``HTTP Referer``字段，在请求地址中添加token并验证，在HTTP头中自定义属性并验证。下面分别简述之。

#### 1. 验证HTTP Referer字段

根据HTTP协议，在HTTP头中有一个字段叫Referer，它记录了该HTTP请求的来源地址。在通常情况下，访问一个安全受限页面的请求必须来自于同一个网站。比如某银行的转账是通过用户访问``http://bank.test/test?page=10&userID=101&money=10000`` 页面完成，用户必须先登录``bank.test``，然后通过点击页面上的按钮来触发转账事件。当用户提交请求时，该转账请求的Referer值就会是转账按钮所在页面的URL（本例中，通常是以``bank.test``域名开头的地址）。而如果攻击者要对银行网站实施CSRF攻击，他只能在自己的网站构造请求，当用户通过攻击者的网站发送请求到银行时，该请求的Referer是指向攻击者的网站。因此，要防御CSRF攻击，银行网站只需要对于每一个转账请求验证其Referer值，如果是以``bank.test``开头的域名，则说明该请求是来自银行网站自己的请求，是合法的。如果Referer是其他网站的话，就有可能是CSRF攻击，则拒绝该请求。


#### 2. 在请求地址中添加token并验证

CSRF攻击之所以能够成功，是因为攻击者可以伪造用户的请求，该请求中所有的用户验证信息都存在于Cookie中，因此攻击者可以在不知道这些验证信息的情况下直接利用用户自己的Cookie来通过安全验证。由此可知，抵御CSRF攻击的关键在于：在请求中放入攻击者所不能伪造的信息，并且该信息不存在于Cookie之中。鉴于此，系统开发者可以在HTTP请求中以参数的形式加入一个随机产生的token，并在服务器端建立一个拦截器来验证这个token，如果请求中没有token或者token内容不正确，则认为可能是CSRF攻击而拒绝该请求。

#### 3. 在HTTP头中自定义属性并验证

自定义属性的方法也是使用token并进行验证，和前一种方法不同的是，这里并不是把token以参数的形式置于HTTP请求之中，而是把它放到HTTP头中自定义的属性里。通过``XMLHttpRequest``这个类，可以一次性给所有该类请求加上csrftoken这个HTTP头属性，并把token值放入其中。这样解决了前一种方法在请求中加入token的不便，同时，通过这个类请求的地址不会被记录到浏览器的地址栏，也不用担心token会通过Referer泄露到其他网站。

### 客户端

对于普通用户来说，都学习并具备网络安全知识以防御网络攻击是不现实的。但若用户养成良好的上网习惯，则能够很大程度上减少CSRF攻击的危害。例如，用户上网时，不要轻易点击网络论坛、聊天室、即时通讯工具或电子邮件中出现的链接或者图片；及时退出长时间不使用的已登录账户，尤其是系统管理员，应尽量在登出系统的情况下点击未知链接和图片。除此之外，用户还需要在连接互联网的计算机上安装合适的安全防护软件，并及时更新软件厂商发布的特征库，以保持安全软件对最新攻击的实时跟踪。

另外，现在有一些浏览器插件可以进行简单的防御工作，但是有一些局限性。

### 安全设备

CSRF攻击的本质是攻击者伪造了合法的身份，对系统进行访问。如果能够识别出访问者的伪造身份，也就能识别CSRF攻击。研究发现，有些厂商的安全产品能基于硬件层面对HTTP头部的Referer字段内容进行检查来快速准确的识别CSRF攻击。但是这种方式代价昂贵，一般并不会采用这种防御方案。

一般而言，最常用的就是采用服务端防御策略。解决办法就是在Form表单加一个hidden field，里面是服务端生成的足够随机数的一个Token，使得黑客猜不到也无法仿照Token。 

## <strong>五、具体实现</strong>

《浅谈CSRF攻击方式》<sup>[8]</sup>给出了一些常见的服务端防御实现，我也将给出一种在 SpringMVC3+Velocity 的框架下防御CSRF攻击的解决方案，详见[《一种在SpirngMVC3中防御CSRF攻击的实现》](http://javanlu.github.io/blog/2013/09/29/springmvc-csrf-defense/)<sup>[9]</sup>。


>### 参考资料
>[1] CSRF Wikipedia 定义：[http://en.wikipedia.org/wiki/Cross-site_request_forgery](http://en.wikipedia.org/wiki/Cross-site_request_forgery) 
>
>[2] Spring MVC防御CSRF、XSS和SQL注入攻击：[http://www.cnblogs.com/Mainz/archive/2012/11/01/2749874.html](http://www.cnblogs.com/Mainz/archive/2012/11/01/2749874.html)
>
>[3] side effect：[http://en.wikipedia.org/wiki/HTTP#Safe_methods](http://en.wikipedia.org/wiki/HTTP#Safe_methods)
>
>[4] Web安全测试之跨站请求伪造(CSRF)篇：[(http://netsecurity.51cto.com/art/200811/97281_1.htm](http://netsecurity.51cto.com/art/200811/97281_1.htm)
>
>[5] 深入解析跨站请求伪造漏洞：实例讲解：[http://netsecurity.51cto.com/art/200812/102925_1.htm](http://netsecurity.51cto.com/art/200812/102925_1.htm)
>
>[6] 深入解析跨站请求伪造漏洞：原理剖析：[http://netsecurity.51cto.com/art/200812/102951_1.htm](http://netsecurity.51cto.com/art/200812/102951_1.htm)
>
>[7] CSRF的攻击与防御：[http://www.h3c.com.cn/About_H3C/Company_Publication/IP_Lh/2012/04/Home/Catalog/201208/751467_30008_0.htm](http://www.h3c.com.cn/About_H3C/Company_Publication/IP_Lh/2012/04/Home/Catalog/201208/751467_30008_0.htm)
>
>[8] 浅谈CSRF攻击方式：[http://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html](http://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html)
>
>[9] 一种在SpirngMVC3中防御CSRF攻击的实现：[http://javanlu.github.io/blog/2013/09/29/springmvc-csrf-defense/](http://javanlu.github.io/blog/2013/09/29/springmvc-csrf-defense/)




