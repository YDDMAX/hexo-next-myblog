---
title: openjdk8之编译和debug.md
date: 2017-06-12 00:26:56
tags: [openjdk,jvm]
categories: jvm
---
系统环境为ubuntu 16.04，uname -a:
```bash
Linux ddy-Aspire-V5-573G 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:33:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```
在本文中，要编译的openjdk版本为:[openjdk-8u40-src-b25-10_feb_2015][1]。
尝试了编译[openjdk-8-src-b132-03_mar_2014][2]，但是失败。网上说,因为ubuntu16.04较新，但是该版本的JDK较老，所以失败。

下面说明编译过程。

# make版本
OpenJDK8可以使用"config && make"编译构建，不再使用Ant和ALT_ *环境变量来配置构建。
不过需要GNU make 3.81或更新的版本
# 安装引导JDK
我使用的引导JDK是[jdk-7u76-linux-x64][3]。
```bash
java version "1.6.0_45"
Java(TM) SE Runtime Environment (build 1.6.0_45-b06)
Java HotSpot(TM) 64-Bit Server VM (build 20.45-b01, mixed mode)
```

# 安装编译工具类库：

```bash
#安装gcc、g++、make等  
sudo apt-get install build-essential      
#安装ant 1.7以上  
sudo apt-get install ant  
#安装XRender  
sudo apt-get install libxrender-dev  
sudo apt-get install xorg-dev 
#安装alsa  
sudo apt-get install libasound2-dev
#Cups  
sudo apt-get install libcups2-dev 
#安装零碎的工具包  
sudo apt-get install gawk zip libxtst-dev libxi-dev libxt-dev
```  
# 建立编译脚本
--with-boot-jdk：指定引导JDK所在目录，以防其他安装的JDK影响（本机上以前安装了JDK8，并配置了JAVA_HOME指向JDK8）；
--with-target-bits：指定编译64位系统的JDK；

为可以进行源码调试，再指定下面三个参数：
--with-debug-level=slowdebug：指定可以生成最多的调试信息；
--enable-debug-symbols ZIP_DEBUGINFO_FILES=0：生成调试的符号信息，并且不压缩；
在openjdk目录下新建build.sh，内容如下:
```bash
cd openjdk  
bash ./configure --with-target-bits=64 --with-boot-jdk=/usr/java/jdk1.7.0_80/ --with-debug-level=slowdebug --enable-debug-symbols ZIP_DEBUGINFO_FILES=0  
make all ZIP_DEBUGINFO_FILES=0  
```
# 编译
执行``./build.sh``
编译完成是这样的：
![openjdk8-compile-success.png][4]
# 用GDB测试是否能debug
```bash
ddy@ddy-Aspire-V5-573G ~/openjdk-compile/openjdk-8u40-src-b25-10_feb_2015/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin $ ./java -version
openjdk version "1.8.0-internal-debug"
OpenJDK Runtime Environment (build 1.8.0-internal-debug-ddy_2017_06_11_23_26-b00)
OpenJDK 64-Bit Server VM (build 25.40-b25-debug, mixed mode)
ddy@ddy-Aspire-V5-573G ~/openjdk-compile/openjdk-8u40-src-b25-10_feb_2015/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin $ export CLASSPATH=.:/home/
ddy/java_src
ddy@ddy-Aspire-V5-573G ~/openjdk-compile/openjdk-8u40-src-b25-10_feb_2015/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin $ gdb --args java FileChann
elTest
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.04) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from java...done.
(gdb) break init.cpp:95
No source file named init.cpp.
Make breakpoint pending on future shared library load? (y or [n]) y
Breakpoint 1 (init.cpp:95) pending.
(gdb) run
Starting program: /home/ddy/openjdk-compile/openjdk-8u40-src-b25-10_feb_2015/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin/java FileChannelTest
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[New Thread 0x7ffff7fc8700 (LWP 9311)]
[Switching to Thread 0x7ffff7fc8700 (LWP 9311)]

Thread 2 "java" hit Breakpoint 1, init_globals ()
    at /home/ddy/openjdk-compile/openjdk-8u40-src-b25-10_feb_2015/openjdk/hotspot/src/share/vm/runtime/init.cpp:95
95	jint init_globals() {
(gdb) l
90	  chunkpool_init();
91	  perfMemory_init();
92	}
93	
94	
95	jint init_globals() {
96	  HandleMark hm;
97	  management_init();
98	  bytecodes_init();
99	  classLoader_init();
(gdb) quit
A debugging session is active.

	Inferior 1 [process 9307] will be killed.

Quit anyway? (y or n) y
ddy@ddy-Aspire-V5-573G ~/openjdk-compile/openjdk-8u40-src-b25-10_feb_2015/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin $ 

```


  [1]: https://pan.baidu.com/s/1mhLHkc4
  [2]: https://pan.baidu.com/s/1jI1cGNc
  [3]: https://pan.baidu.com/s/1hr6qkOO
  [4]: http://oqxil93b6.bkt.clouddn.com/images/openjdk/opjdk-8u40-compile-success.png
