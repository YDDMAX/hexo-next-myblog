---
title: openjdk7之编译和debug
date: 2017-06-11 23:53:55
tags: [openjdk,jvm]
categories: jvm
---
为了更好的学习JDK、HotSpot等源码，需要能debug JDK、HotSpot等源码。本文主要讲述，怎么编译openjdk并debug相关源码。
在本文中，要编译的openjdk:[openjdk-7u40-fcs-src-b43-26_aug_2013.zip][1]
系统环境为ubuntu 16.04，uname -a:
```bash
Linux ddy-Aspire-V5-573G 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:33:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```
# 编译
1. 下载源代码
openjdk的源码可以通过hg方式下载。
也可以从此处下载：[openjdk源码][2]
2. 安装引导JDK
因为JDK中有很多代码是Java自身实现的，所以还需要一个已经安装在本机上可用的JDK，叫做“Bootstrap JDK”。我所选用的Bootstarp JDK是JDK1.6.0_45。
```bash
  java version "1.6.0_45"
  Java(TM) SE Runtime Environment (build 1.6.0_45-b06)
  Java HotSpot(TM) Server VM (build 20.45-b01, mixed mode)  
```
  JDK1.6.0_45下载地址：[jdk1.6.0_45.tar.gz][3]
3. 安装编译前的依赖环境
安装gcc、g++、make等  
``sudo apt-get install build-essential``
安装ant 1.7以上  
``sudo apt-get install ant``  
安装XRender  
``sudo apt-get install libxrender-dev``  
``sudo apt-get install xorg-dev`` 
安装alsa  
``sudo apt-get install libasound2-dev``
Cups  
``sudo apt-get install libcups2-dev`` 
安装零碎的工具包  
``sudo apt-get install gawk zip libxtst-dev libxi-dev libxt-dev``
4. 配置编译脚本
将你的openjdk解压后，并进入该文件夹。比如我的是在/home/ddy/openjdk-compile/openjdk-7u40-fcs-b43-26/openjdk
下。新建一个build.sh，并添加如下内容：

```bash
export LANG=C
#将一下两项设置为你的BootstrapJDK安装目录
export ALT_BOOTDIR=/home/ddy/jdk1.6.0_45
export ALT_JDK_IMPORT_PATH=/home/ddy/jdk1.6.0_45
#允许自动下载依赖包
export ALLOW_DOWNLOADS=true

#使用预编译头文件，以提升便以速度
export USE_PRECOMPILED_HEADER=true

#要编译的内容，我只选择了LANGTOOLS、HOTSPOT以及JDK
export BUILD_LANGTOOLS=true
export BUILD_JAXP=false
export BUILD_JAXWS=false
export BUILD_CORBA=false
export BUILD_HOSTPOT=true
export BUILD_JDK=true

#要编译的版本
export SKIP_DEBUG_BUILD=false
export SKIP_FASTDEBUG_BUILD=true
export DEBUG_NAME=debug

#避免javaws和浏览器Java插件等的build
BUILD_DEPLOY=false

#不build安装包
BUILD_INSTALL=false

#包含全部的调试信息
export  ENABLE_FULL_DEBUG_SYMBOLS=1
#调试信息是否压缩，如果配置为１,libjvm.debuginfo会被压缩成libjvm.diz,将不能被debug。
export  ZIP_DEBUGINFO_FILES=0
#用于编译线程数
export  HOTSPOT_BUILD_JOBS=3

#设置存放编译结果的目录
#export ALT_OUTPUTDIR=/home/ddy/openjdk/7/build

unset CLASSPATH
unset JAVA_HOME
make sanity
DEBUG_BINARIES=true make 2>&1
```

5.开始编译
在openjdk目录下，运行build.sh


```bash
chmod +x build.sh
./build.sh
```


最后编译耗时将近2分钟。编译完成输出如下信息：
![compile-success][4]
此时openjdk就编译完成了，编译的输出在``/home/ddy/openjdk-compile/openjdk-7u40-fcs-b43-26/openjdk/build/``下。
进入``/home/ddy/openjdk-compile/openjdk-7u40-fcs-b43-26/openjdk/build/linux-amd64-debug/j2re-image/bin
n``，执行
``./java -version``
输出的java版本信息将是带着你的机器用户名，我的输出是：
```bash
openjdk version "1.7.0-internal-debug"
OpenJDK Runtime Environment (build 1.7.0-internal-debug-ddy_2017_06_10_22_30-b00)
OpenJDK 64-Bit Server VM (build 24.0-b56-jvmg, mixed mode)
```
# debug
编译完成了之后，就可以对JDK源码和HotSpot源码等进行debug了。
## JDK
首先是JDK源码，在build目录下编译生成的jdk里面的jar包都是可编译的了，直接把eclipse的JDK或者JRE换成编译成功的JDK或者JRE即可。
## HotSpot
注意，如果不能进入断点，出现以下类似信息：
      ``Missing separate debuginfo for/root/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/lib/amd64/server/libjvm.so``
是因为在编译时因为编译配置项不正确而没有生成调试的符号信息，或生成后被压缩为"libjvm.diz"了，所以无法找到。如果是因为没有编译时没有生成调试信息，需要修改编译配置并重新编译。对于被压缩的情况，可以去到"libjvm.so"所在目录

+ 然后解压：unzip libjvm.diz                
+ 解压出来：libjvm.debuginfo


如果在编译时，把配置信息修改如下，则不会出现不能上述问题。
```bash
#包含全部的调试信息
export  ENABLE_FULL_DEBUG_SYMBOLS=1
#调试信息是否压缩，如果配置为１,libjvm.debuginfo会被压缩成libjvm.diz,将不能被debug。
export  ZIP_DEBUGINFO_FILES=0
```


### 使用GDB
 参考：[CentOS上编译OpenJDK8源码 以及 在eclipse上调试HotSpot虚拟机源码][5]
 
### 使用eclipse
1. 生成要运行的JAVA类

首先在``/home/ddy/src/java-src``目录下建立要运行的FileChannelTest.java，这个类在写文件时调用了JDK的native方法，其代码如下：
```java
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
public class FileChannelTest {

   public static void main(String[] args) throws IOException {
            FileChannel channel=new RandomAccessFile("test.txt","rw").getChannel();
            ByteBuffer buffer=ByteBuffer.allocate(1000);
            channel.write(buffer);
        }
}
```
然后对其进行编译,运行:
```bash
ddy@ddy-Aspire-V5-573G ~ $ cd src/java-src/
ddy@ddy-Aspire-V5-573G ~/src/java-src $ pwd
/home/ddy/src/java-src
ddy@ddy-Aspire-V5-573G ~/src/java-src $ /home/ddy/openjdk-compile/openjdk-7u40-fcs-b43-26/openjdk/build/linux-amd64-debug/j2sdk-image/bin/javac FileChannelTest.java 
```

2. 下载eclipse,安装C/C++插件

到官网选择一个合适的eclipse下载，因为本人主要进行JAVA开发，所以下载的是j2EE版本，这个版本没有C/C++的功能。不过可以安装插件使其支持C/C++功能。"help -> Eclipse Maketplace",搜索"c++"找到Eclipse C++ IDE..安装。安装后，就可以转到C++开发视图界面了。

3.导入hotspot工程

File-> New -> Makefile Project With Existing Code 
在界面中:
      Project Name：openjdk（这个可以自己选择）
      Existing Code Location：/root/openjdk
      Toolchain：选Linux GCC，然后按Finish.     
4.配置源码调试     

+ 右键工程 -> Debug As -> Debug Configurations -> 右键左边的C/C++ Application -> New -> 进入Main选项卡；
在选项卡中:
      Project：openjdk（选择导入的openjdk工程）
      C/C++ Application：``/home/ddy/openjdk-compile/openjdk-7u40-fcs-b43-26/openjdk/build/linux-amd64-debug/j2sdk-image/bin/java``（编译生成的openjdk虚拟机入口）
      Disable auto build：因为不再在eclipse里面编译hotspot源码,所以这里选上它；
+ 然后切换到Arguments选项卡, 输入Java的参数, 这里填上 "FileChannelTest"也就是我们要执行的JAVA程序。
+ 然后切换到Environment选项卡, 添加变量：
``JAVA_HOME=/home/ddy/openjdk-compile/openjdk-7u40-fcs-b43-26/openjdk/build/linux-amd64-debug/j2sdk-image``（编译生成JDK所在目录）
``CLASSPATH=.:/home/ddy/src/java-src`` (FileChannelTest.java文件所在目录)
点击下面的Apply保存；
+ 断点Debug
　下面分别在源码上打两个断点：

     + init.cpp(/home/ddy/openjdk-compile/openjdk-7u40-fcs-b43-26/openjdk/hotspot/src/share/vm/runtime目录下)      95行
     + FileDispatchImpl.c(/home/ddy/openjdk-compile/openjdk-7u40-fcs-b43-26/openjdk/jdk/src/solaris/native/sun/nio/ch目录下)    107行

然后开始debug。
首先是第一个断点：
![init.cpp-95.png][6]

F8进行到下一个断电点：

![FileDispatcherImpl-write.c.png][7]

从上图可以看到,FileChannel.write()最后调用的是write()操作系统调用。
所以，大家现在可以随便debug　HotSpot的源码和JDK的native源码了。酷！

# 参考资料
[openjdk之编译经常出现的问题][8]
[openjdk8的编译和debug][9]
编译主要参考：[ubuntu14.04 编译openjdk7][10]
debug主要参考：[CentOS上编译OpenJDK8源码 以及 在eclipse上调试HotSpot虚拟机源码][11]


  [1]: https://pan.baidu.com/s/1bFVRwq
  [2]: https://pan.baidu.com/s/1qXMoWfM
  [3]: https://pan.baidu.com/s/1eStdxq6
  [4]: http://oqxil93b6.bkt.clouddn.com/images/openjdk/compile-success.png
  [5]: http://blog.csdn.net/tjiyu/article/details/53725247
  [6]: http://oqxil93b6.bkt.clouddn.com/images/openjdk/init.cpp-95.png
  [7]: http://oqxil93b6.bkt.clouddn.com/images/openjdk/FileDispatcherImpl-write.c.jpg
  [8]: https://yddmax.github.io/2017/06/12/openjdk%E4%B9%8B%E7%BC%96%E8%AF%91%E7%BB%8F%E5%B8%B8%E5%87%BA%E7%8E%B0%E7%9A%84%E9%97%AE%E9%A2%98/
  [9]: https://yddmax.github.io/2017/06/12/openjdk8%E4%B9%8B%E7%BC%96%E8%AF%91%E5%92%8Cdebug/
  [10]: https://ayonel.me/index.php/2017/01/05/compile_openjdk/
  [11]: http://blog.csdn.net/tjiyu/article/details/53725247