---
layout: post
title: "手动编译Hadoop的Eclipse Plugin"
date: 2013-09-24 10:32
comments: true
categories: Hadoop Tools
---

在学习Hadoop的过程中，很多童鞋都希望在IDE环境中将样例跑起来，比如在
Eclipse IDE中。但是Hadoop的Eclipse Plugin对Eclipse的版本要求很严格，经常会出现各种莫名的错误，而且最新的Hadoop中不再提供Eclipse Plugin的JAR，
那该怎么办呢？
<!--more-->
所谓求人不如求己，Hadoop提供了Eclipse Plugin的源码，我们可以结合自已的Eclipse版本手动编译生成plugin的JAR。这里以``${HADOOP_HOME}``来表示您电脑上hadoop的安装目录。


### 1. 编辑${HADOOP_HOME}/src/contrib/下的build-contrib.xml文件

```xml
	<project name="hadoopbuildcontrib" xmlns:ivy="antlib:org.apache.ivy.ant">
	<!--定义hadoop版本 和 Eclipse 安装目录-->
	<property name="version" value="1.2.1"/>
	<property name="eclipse.home" value="/opt/eclipse"/>
```
### 2. 编辑${HADOOP_HOME}/src/contrib/eclipse-plugin/下的build.xml文件

#### 2.1 添加hadoop-jars path，并同时加入到classpath中：
```xml
<path id="hadoop-jars">
    <fileset dir="${hadoop.root}/">
        <include name="hadoop-*.jar"/>
    </fileset>
</path>
<!-- Override classpath to include Eclipse SDK jars -->
<path id="classpath">
    <pathelement location="${build.classes}"/>
    <pathelement location="${hadoop.root}/build/classes"/>
    <path refid="eclipse-sdk-jars"/>
    <!-- 将 hadoop-jars 添加到这里 -->
    <path refid="hadoop-jars"/>
</path>
```
#### 2.2 设置includeantruntime=on，防止compile时报warning：
```xml
<target name="compile" depends="init, ivy-retrieve-common" unless="skip.contrib">
	<echo message="contrib: ${name}"/>
	<javac
     encoding="${build.encoding}"
     srcdir="${src.dir}"
     includes="**/*.java"
     destdir="${build.classes}"
     debug="${javac.debug}"
     deprecation="${javac.deprecation}" 
     includeantruntime="on"> <!-- 设置includeantruntime=on,防止compile报warning -->
     	<classpath refid="classpath"/>
    </javac>
</target>
```
#### 2.3 添加将要打包到plugin中的第三方包列表：
```xml	
<!-- Override jar target to specify manifest -->
<target name="jar" depends="compile" unless="skip.contrib">
    <mkdir dir="${build.dir}/lib"/>
    
    <!-- 这里主要修改的是file中的值，注意路径一定要正确 -->
    <copy file="${hadoop.root}/hadoop-core-${version}.jar"
          tofile="${build.dir}/lib/hadoop-core.jar" verbose="true"/>
    <copy file="${hadoop.root}/lib/commons-cli-1.2.jar"
          todir="${build.dir}/lib" verbose="true"/>
    <copy file="${hadoop.root}/lib/commons-lang-2.4.jar"
          todir="${build.dir}/lib" verbose="true"/>
    <copy file="${hadoop.root}/lib/commons-configuration-1.6.jar"
          todir="${build.dir}/lib" verbose="true"/>
    <copy file="${hadoop.root}/lib/jackson-mapper-asl-1.8.8.jar"
          todir="${build.dir}/lib" verbose="true"/>
    <copy file="${hadoop.root}/lib/jackson-core-asl-1.8.8.jar"
          todir="${build.dir}/lib" verbose="true"/>
    <copy file="${hadoop.root}/lib/commons-httpclient-3.0.1.jar"
          todir="${build.dir}/lib" verbose="true"/>
     
   <jar
      jarfile="${build.dir}/hadoop-${name}-${version}.jar"
      manifest="${root}/META-INF/MANIFEST.MF">
      <fileset dir="${build.dir}" includes="classes/ lib/"/>
      <fileset dir="${root}" includes="resources/ plugin.xml"/>
    </jar>
</target>
```
### 3. 修改${HADOOP_HOME}/src/contrib/eclipse-plugin/META-INF/ 的 MANIFEST.MF文件

主要是修改***Bundle-ClassPath***
```
	Bundle-ClassPath: classes/,lib/hadoop-core.jar,lib/jackson-core-asl-1.8.8.jar,
	lib/jackson-mapper-asl-1.8.8.jar, lib/commons-configuration-1.6.jar,
	lib/commons-lang-2.4.jar, lib/commons-httpclient-3.0.1.jar
```
### 4. 执行Ant命令
**注意：要吧目录切换到``${HADOOP_HOME}/src/contrib/eclipse-plugin/``下执行Ant命令。**

直接输入ant，结果提示我没有安装这个指令，这个时候，直接安装就行了：
```sh
	$ sudo apt-get install ant1.7
```
装完就行了，接着就是执行，结果build 失败了，而且我发现系统的jdk变成openjdk了，这让我很不爽啊。
```sh
	BUILD FAILED
	/opt/hadoop-1.2.1/src/contrib/eclipse-plugin/build.xml:68: Unable to find a javac compiler;
	com.sun.tools.javac.Main is not on the classpath.Perhaps JAVA_HOME does not point to the JDK.
	It is currently set to "/usr/lib/jvm/java-7-openjdk-amd64/jre"
```
怎么改呢？
```sh
	sudo update-alternatives --install /usr/bin/java java ${JAVA_HOME}/bin/java 300
	sudo update-alternatives --install /usr/bin/javac javac ${JAVA_HOME}/bin/javac 300
	sudo update-alternatives --config java
	sudo update-alternatives --config javac
```
``${JAVA_HOME}``要改成自己的JDK安装路径。如果你的系统中安装了其他的JDK，在执行最后两条指令的时候，系统会提示出来，然后作出你的选择就行了。
再Ant运行，成功，结果如下。
```sh
	Buildfile: build.xml
 
	check-contrib:
	 
	init:
	     [echo] contrib: eclipse-plugin
	 
	init-contrib:
	 
	ivy-download:
	      [get] Getting: http://repo2.maven.org/maven2/org/apache/ivy/ivy/2.1.0/ivy-2.1.0.jar
	      [get] To: /opt/hadoop-1.2.1/ivy/ivy-2.1.0.jar
	      [get] Not modified - so not downloaded
	 
	ivy-probe-antlib:
	 
	ivy-init-antlib:
	 
	ivy-init:
	[ivy:configure] :: Ivy 2.1.0 - 20090925235825 :: http://ant.apache.org/ivy/ ::
	[ivy:configure] :: loading settings :: file = /opt/hadoop-1.2.1/ivy/ivysettings.xml
	 
	ivy-resolve-common:
	 
	ivy-retrieve-common:
	[ivy:cachepath] DEPRECATED: 'ivy.conf.file' is deprecated, use 'ivy.settings.file' instead
	[ivy:cachepath] :: loading settings :: file = /opt/hadoop-1.2.1/ivy/ivysettings.xml
	 
	。。。。。。。。
	 
	BUILD SUCCESSFUL
	Total time: 14 seconds	
```
最后成功生成的``hadoop-eclipse-plugin-${version}.jar`` 在 ``${HADOOP_HOME}/build/contrib/eclipse-plugin``下。
```
	drwxr-xr-x 3 root root    4096 Aug  9 15:55 classes/
	drwxr-xr-x 2 root root    4096 Aug  9 15:13 examples/
	-rw-r--r-- 1 root root 5880495 Aug  9 15:55 hadoop-eclipse-plugin-1.2.1.jar
	drwxr-xr-x 2 root root    4096 Aug  9 15:55 lib/
	drwxr-xr-x 3 root root    4096 Aug  9 15:13 system/
	drwxr-xr-x 3 root root    4096 Aug  9 15:13 test/
```
把它拷贝到Eclipse下，就可以使用Hadoop的Plugin了。

