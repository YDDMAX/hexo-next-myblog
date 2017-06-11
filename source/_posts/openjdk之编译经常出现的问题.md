---
title: openjdk之编译经常出现的问题
date: 2017-06-12 00:41:19
tags: [openjdk,jvm]
categories: jvm
---

openjdk编译过程中，因为系统环境和openjdk版本等问题，会出现各种问题。本文主要列出两个经常出现的问题及解决办法。
# time is more than 10 years from present
```java
Error: time is more than 10 years from present: 1136059200000
Java.lang.RuntimeException: time is more than 10 years from present: 1136059200000
  at build.tools.generatecurrencydata.GenerateCurrencyData.makeSpecialCaseEntry(GenerateCurrencyData.java:285)
  at build.tools.generatecurrencydata.GenerateCurrencyData.buildMainAndSpecialCaseTables(GenerateCurrencyData.java:225)
  at build.tools.generatecurrencydata.GenerateCurrencyData.main(GenerateCurrencyData.java:154)

```
解决办法:

修改CurrencyData.properties（路径：jdk/src/share/classes/java/util/CurrencyData.properties）

修改108行
``AZ=AZM;2009-12-31-20-00-00;AZN``
修改381行
``MZ=MZM;2009-06-30-22-00-00;MZN``
修改443行
``RO=ROL;2009-06-30-21-00-00;RON``
修改535行
``TR=TRL;2009-12-31-22-00-00;TRY``
修改561行
``VE=VEB;2009-01-01-04-00-00;VEF``

# OS is not supported: Linux … 4.0.0-1-amd64 …
openjdk在编译时检查linux的内核版本，之前的检查代码没有检查4.x版本(那个时候还没有这个版本的内核)，导致出错。我们只需要在对应的检查代码里加上即可。

在文件hotspot/make/linux/Makefile中，修改如下：
```
-SUPPORTED_OS_VERSION = 2.4% 2.5% 2.6% 3%
+SUPPORTED_OS_VERSION = 2.4% 2.5% 2.6% 3% 4%
```


