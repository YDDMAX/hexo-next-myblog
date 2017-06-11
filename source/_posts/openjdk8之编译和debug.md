---
title: openjdk8֮�����debug
date: 2017-06-12 00:26:56
tags: [openjdk,jvm]
categories: jvm
---
ϵͳ����Ϊubuntu 16.04��uname -a:
```bash
Linux ddy-Aspire-V5-573G 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:33:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```
�ڱ����У�Ҫ�����openjdk�汾Ϊ:[openjdk-8u40-src-b25-10_feb_2015][1]��
�����˱���[openjdk-8-src-b132-03_mar_2014][2]������ʧ�ܡ�����˵,��Ϊubuntu16.04���£����Ǹð汾��JDK���ϣ�����ʧ�ܡ�

����˵�������debug���̡�

# make�汾
OpenJDK8����ʹ��"config && make"���빹��������ʹ��Ant��ALT_ *�������������ù�����
������ҪGNU make 3.81����µİ汾
# ��װ����JDK
��ʹ�õ�����JDK��[jdk-7u76-linux-x64][3]��
```bash
java version "1.6.0_45"
Java(TM) SE Runtime Environment (build 1.6.0_45-b06)
Java HotSpot(TM) 64-Bit Server VM (build 20.45-b01, mixed mode)
```

# ��װ���빤����⣺

��װgcc��g++��make��  
``sudo apt-get install build-essential``
��װXRender  
``sudo apt-get install libxrender-dev``  
``sudo apt-get install xorg-dev`` 
��װalsa  
``sudo apt-get install libasound2-dev``
Cups  
``sudo apt-get install libcups2-dev`` 
��װ����Ĺ��߰�  
``sudo apt-get install gawk zip libxtst-dev libxi-dev libxt-dev``
 
 
# ��������ű�
--with-boot-jdk��ָ������JDK����Ŀ¼���Է�������װ��JDKӰ�죨��������ǰ��װ��JDK8����������JAVA_HOMEָ��JDK8����
--with-target-bits��ָ������64λϵͳ��JDK��

Ϊ���Խ���Դ����ԣ���ָ����������������
--with-debug-level=slowdebug��ָ�������������ĵ�����Ϣ��
--enable-debug-symbols ZIP_DEBUGINFO_FILES=0�����ɵ��Եķ�����Ϣ�����Ҳ�ѹ����
��openjdkĿ¼���½�build.sh����������:
```bash
cd openjdk  
bash ./configure --with-target-bits=64 --with-boot-jdk=/usr/java/jdk1.7.0_80/ --with-debug-level=slowdebug --enable-debug-symbols ZIP_DEBUGINFO_FILES=0  
make all ZIP_DEBUGINFO_FILES=0  
```

# ����
ִ��``./build.sh``
��������������ģ�
![openjdk8-compile-success.png][4]
# ��GDB�����Ƿ���debug
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
[openjdk֮���뾭�����ֵ�����][5]
[openjdk7�ı����debug][6]
������Ҫ�ο���[ubuntu14.04 ����openjdk7][7]
debug��Ҫ�ο���[CentOS�ϱ���OpenJDK8Դ�� �Լ� ��eclipse�ϵ���HotSpot�����Դ��][8]


  [1]: https://pan.baidu.com/s/1mhLHkc4
  [2]: https://pan.baidu.com/s/1jI1cGNc
  [3]: https://pan.baidu.com/s/1hr6qkOO
  [4]: http://oqxil93b6.bkt.clouddn.com/images/openjdk/opjdk-8u40-compile-success.png
  [5]: https://yddmax.github.io/2017/06/12/openjdk%E4%B9%8B%E7%BC%96%E8%AF%91%E7%BB%8F%E5%B8%B8%E5%87%BA%E7%8E%B0%E7%9A%84%E9%97%AE%E9%A2%98/
  [6]: https://yddmax.github.io/2017/06/11/openjdk7%E4%B9%8B%E7%BC%96%E8%AF%91%E5%92%8Cdebug/
  [7]: https://ayonel.me/index.php/2017/01/05/compile_openjdk/
  [8]: http://blog.csdn.net/tjiyu/article/details/53725247