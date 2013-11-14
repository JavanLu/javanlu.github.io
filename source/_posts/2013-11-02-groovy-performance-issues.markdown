---
layout: post
title: "Groovy性能问题"
date: 2013-11-02 12:23
comments: true
categories: Groovy Performance
---

>之前在TB，做一个实时监控引擎，使用了 Groovy 来实现动态规则加载。系统虽然在数据结构以及 JVM 层面进行了多次优化，性能也得到了显著提升。但是感觉机器的性能并未用到极致，还遇到过 PermGen 溢出的问题。隐约感觉是 Groovy 这个环节出了问题，但是苦于水平有限，对 Groovy 在实际运用中的认识不够深刻，迟迟未查到原因。后来，三哥根据毕玄大神的指点，优化了Grrovy 这一块，据说性能提升了三倍。好东西，当然要分享出来。以下是毕玄大神的原文，以飨众生。

Groovy 的动态性为一些 Java 的应用场景带来了便利（例如规则类的场景，因为规则可能会经常变化或不断的增加），因此有不少Java 应用会使用到 Groovy，但在使用 Groovy 时有几个点需要注意，否则很容易碰到一些异常问题，这些总结来源于实际排查的一些 Case。

<!-- more -->
 
# 正确使用 GroovyClassLoader

通常使用如下代码在Java 中执行 Groovy 脚本：

```java
GroovyClassLoader groovyLoader = new GroovyClassLoader();
Class<Script> groovyClass = (Class<Script>) groovyLoader.parseClass(groovyScript);
Script groovyScript = groovyClass.newInstance();
```

这个用法是比较简单的，但上面这短短的三行代码是还真挺容易犯错的，我见过的使用 Groovy 的应用场景一开始几乎都要在此犯下错误。
 
首先容易犯的第一个错是采用一个看似不错的优化：就是对所有的Groovy脚本采用一个全局的``GroovyClassLoader``，这种方式当应用用到了很多的Groovy Script时将出现问题，PermGen 中的class能被卸载的前提是``ClassLoader``中所有的class都没有地方引用，因此如果是共用同一个``GroovyClassLoader``的话就很容易导致脚本class无法被卸载，从而导致 PermGen 被用满。
 
第二个容易犯的错误是每次都执行``groovyLoader.parseClass(groovyScript)``，Groovy 为了保证每次执行的都是新的脚本内容，在这里会每次生成一个新名字的 Class 文件，代码如下：

```java
return parseClass(text, "script" + System.currentTimeMillis() + Math.abs(text.hashCode()) + ".groovy");
```

因此当对同一段脚本每次都执行这个方法时，会导致的现象就是装载的Class会越来越多，从而导致PermGen被用满。
 
因此通常在使用 Groovy 装载和执行脚本时，比较好的做法是：

- 每个 script 都 new 一个 GroovyClassLoader 来装载；

- 对于 parseClass 后生成的 Class 对象进行cache，key 为 groovyScript 脚本的md5值。
 
# 注意 CodeCache 的空间状况

对于大量使用Groovy的应用，尤其是 Groovy 脚本还会经常更新的应用，由于这些Groovy脚本在执行了很多次后都会被JVM编译为 native 进行优化，会占据一些 CodeCache 空间，而如果这样的脚本很多的话，可能会导致 CodeCache 被用满，而 CodeCache 一旦被用满，JVM 的 Compiler 就会被禁用，那性能下降的就不是一点点了。
 
# 自定义函数使用较多的 Groovy 脚本

如应用的Groovy脚本比较多的使用到自定义函数，有一个细节要特别注意，Groovy在查找需要调用的方法时，首先从其生成的 MetaClass 时找，当找不到时会抛出异常，在其自身捕捉到异常后然后会转为使用上下文ClassLoader进行装载，而悲催的是，自定义函数在编译时MetaClass里是不会有的，通常只能在上下文 ClassLoader 里找到，因此如自定义函数使用较多，会导致的就是Groovy在执行的过程会不断的抛出``MissingMethodExceptionNoStack``，如果这个异常抛出非常频繁的话，会导致 CPU 在 handle_exception 上被大量的消耗。这个问题在官方中目前没有解决办法，只能是自己继承groovy 的 MetaClassImpl，自行实现方法的查找，来避免异常的抛出。
 
最后还有一个小小的建议，如果一个处理过程中要执行很多的Groovy Script（例如几百个），最好把这些Groovy Script合成一个或少数几个Script，这样一方面会节省PermGen的空间占用，另一方面也会一定程度的提升执行程序。

