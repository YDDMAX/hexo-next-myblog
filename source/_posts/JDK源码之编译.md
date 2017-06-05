---
title: JDK源码之编译 
date: 2017-06-05 21:19:58
tags: [JDK,java]
categories: java
---
本文主要说明如何编译组成rt.jar的源码，不涉及JVM的编译。
# rt.jar
rt.jar也就是运行时相关的class类，安装的jre里面的rt.jar是不能被debug的，为了对JDK的源码进行debug跟踪，需要重新编译rt.jar。
rt.jar的全部源码在openjdk的jdk目录下，具体的路径是openjdk\jdk\src\，源码根据操作系统的不同和是否共享分成几个目录。
**实际上rt.jar因为操作系统的不同，所包含的class也会有所不同，这导致在一些特殊特性上JAVA不再是“一次编译，到处执行”。**
**比如，在java7中，linux/unix下的jre支持``com.sun.java.swing.plaf.gtk.GTKLookAndFeel``**,但是windows就不支持。
操作系统安装的JDK里面的src.zip不包含**sun.*包**，***sun.\***包在openjdk源码里面,具体的路径是：**jdk\src\share\classes**。

openjdk的目录说明和源码下载参见：[openjdk目录][1]

# 编译
本文编译的是openjdk-7里面jdk目录下的源码。

1. 新建java project
2. 将openjdk源码copy到project中。除了复制share目录下的共享源码，还需要复制具体操作系统下的源码。
3. 编译导出jar。





参考资料：[怎么对jdk核心包进行跟踪调试，并查看调试中的变量值][2]


  [1]: https://yddmax.github.io/2017/06/05/openjdk%E6%BA%90%E7%A0%81%E7%9B%AE%E5%BD%95/
  [2]: http://bijian1013.iteye.com/blog/2302520