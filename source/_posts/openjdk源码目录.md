---
title: openjdk源码目录
date: 2017-06-05 20:53:40
tags: [JDK,java]
categories: java
---
本文主要参考自：[openjdk源码结构][1]
# 源码下载
openjdk-7的源码：[openjdk7源码][2]
openjdk-8的源码：[openJDK8源码][3]
# 目录说明
openjdk的源码的目录结构如下图所示：
![此处输入图片的描述][4]

各个目录的说明如下：
**—— corba：不流行的多语言、分布式通讯接口 
—— hotspot：Java 虚拟机 
—— jaxp：XML 处理 
—— jaxws：一组 XML web services 的 Java API 
—— jdk：java 开发工具包 
—— —— 针对操作系统的部分 
—— —— share：与平台无关的实现 
—— langtools：Java 语言工具 
—— nashorn：JVM 上的 JavaScript 运行时**

为了让大家易于理解，有所简化了结构。


## Corba

全称 Common Object Request Broker Architecture，通用对象请求代理架构，是基于 对象-服务 机制设计得。

与 JavaBean、COM 等是同种范畴。

目前，通用的远程过程调用协议是 SOAP（Simple Object Access Protocol，简单对象访问协议），消息格式是 XML-RPC（存在 Json-RPC）。 
另外，Apache Thrift 提供了多语言 C/S 通讯支持； 不少语言也内置了跨语言调用或对分布式环境友好，比如： lua 可以与 c 代码互调用，Go 可以调用 C 代码，erlang 在本地操作与分布式环境下的操作方法一样等。
**Corba和RMI类似，都能实现RPC。但是RMI只是针对JAVA环境的，Corba是支持多语言的方案**

## Hotspot

全称 Java HotSpot Performance Engine，是 Java 虚拟机的一个实现，包含了服务器版和桌面应用程序版。利用 JIT 及自适应优化技术（自动查找性能热点并进行动态优化）来提高性能。

使用 java -version 可以查看 Hotspot 的版本。

从 Java 1.3 起为默认虚拟机，现由 Oracle 维护并发布。

其他 java 虚拟机：

+ JRockit：专注于服务器端，曾经号称是“世界上速度最快的 Java 虚拟机”，现归于 Oracle 旗下。
+ J9：IBM 设计的 Java 虚拟机。
+ Harmony：Apache 的顶级项目之一，从 2011 年 11 月 6 日起入驻 Apache 的 Java 项目。虽然其能够兼容 jdk，但由于 JCP （Java Community Process）仅仅允许授权给 Harmony 一个带有限制条件的TCK（Technology Compatibility Kit），即仅仅能使用在 J2SE ,而不是所有Java实现上（包括 J2ME 和 J2EE），导致了 Apache 组织与 Oracle 的决裂。Harmony 是 Android 虚拟机 Dalvik 的前身。
+ Dalvik 并不是 Java 虚拟机，它执行的是 dex 文件而不是 class 文件，使用的也是寄存器架构而不是栈架构 。

## jaxp

全称 Java API for XML Processing，处理 XML 的Java API，是 Java XML 程序设计的应用程序接口之一，它提供解析和验证XML文档的能力。 
jaxp 提供了处理 xml 文件的三种接口：

+ DOM 接口（文档对象模型解析），位于 \openjdk\jaxp\src\org\w3c\dom
+ SAX 接口（xml 简单 api 解析），位于 \openjdk\jaxp\src\org\xml\sax
+ StAX 接口（xml 流 api），位于 \openjdk\jaxp\src\javax\xml
+ 除了解析接口，JAXP还提供了XSLT接口用来对XML文档进行数据和结构的转换。

## JaxWS

全称 Java API for Web Services，JAX-WS 允许开发者选择 RPC-oriented（面向 RPC） 或者 message-oriented（消息通信，erlang 使用的就是消息通信，不过 Java 内存模型是内存共享）来实现自己的web services。

通过 Web Services 提供的环境，可以实现 Java 与其他编程语言的交互（事实上就是 thrift 所做的，任何一种语言都可以通过 Web Services 实现与其他语言的通信，客户端用一种语言，服务器端可以用其他语言）。

## LangTools

Java 语言支持工具

## JDK

全称 Java Development Kit。

**share**

**classes** 目录里的是 Java 的实现，**native** 目录里的是 C++ 的实现，两部分基本对应。这两个目录里的结构与 java 的包也是对应，各个部分的用途另外再讲。

back、instrument、javavm、npt、transport 几个部分是实现 java 的基础部分，都是 C++ 代码，在这里从最底层理解 java，往后这些内容也会详讲。

sample 和 demo 目录有以下示例，区别在于 demo 目录是 针对 applets 的。

## Nashorn

Nashorn 项目的目的是基于 Java 在 JVM 上实现一个轻量级高性能的 JavaScript 运行环境。基于 JSR-223 协议，Java 程序员可在 Java 程序中嵌入 JavaScript 代码。 
该项目使用了 JSR-229 里描述的新连接机制（从 Java7起开始使用的连接机制）：新的字节码（invokedynamic）以及新的基于方法句柄（method handle）的连接机制。通过接口注入（interface injection）在运行时修改类也是 JSR-229 里的内容。


  [1]: http://blog.csdn.net/socrj/article/details/43916221
  [2]: http://pan.baidu.com/s/1qYAtBK4
  [3]: http://download.csdn.net/download/socrj/8454221
  [4]: http://oqxil93b6.bkt.clouddn.com/images/openjdk%E6%BA%90%E7%A0%81%E7%BB%93%E6%9E%84openjdk-source-structure.png
