﻿---
title: JAVA IO
date: 2017-06-04 14:14:39
tags: [IO,JAVA]
categories: technology
---
# 计算机IO
本部分内容主要参考自《JAVA NIO》的第一章简介。
## 缓冲区操作
下图简单描述了数据从外部磁盘向运行中的进程的内存区域移动的过程。
进程使用 read( )系统调用，要求其缓冲区被填满。内核随即向磁盘控制硬件发出命令，要求其从磁盘读取数据。磁盘控制器把数据直接写入内核内存缓冲区，**这一步通过 DMA 完成，无需主 CPU 协助**。一旦磁盘控制器把缓冲区装满，内核即把数据从内核空间的临时缓冲区拷贝到进程执行 read()调用时指定的缓冲区。
  ![缓冲区操作][1]  
为什么不让磁盘控制器直接将磁盘上数据直接copy到用户缓冲区呢？

+ 硬件无法直接访问用户空间
+ 像磁盘这样基于块存储的硬件设备操作的是固定大小的数据块，而用户进程请求的可能是任意大小的或非对齐的数据块。在数据往来于用户空间与存储设备的过程中，内核负责
数据的分解、再组合工作，因此充当着中间人的角色。 
## 发散和汇聚
根据发散／汇聚的概念，进程只需一个系统调用，就能把一连串缓冲区地址传递给操作系统。然后，内核就可以顺序填充或排干多个缓冲区，读的时候就把数据发散到多个用户空间缓冲区，写的时候再从多个缓冲区把数据汇聚起来（如下图所示）。
![发散和汇聚][2]
优点：

+ 这样用户进程就不必多次执行系统调用（那样做可能代价不菲）
+ 内核也可以优化数据的处理过程，因为它已掌握待传输数据的全部信息。
+ 如果系统配有多个 CPU，甚至可以同时填充或排干多个缓冲区。
+ 多个缓冲区可能方便一些特定协议的编程（自己加的）。
## 虚拟内存
现代操作系统都支持虚拟内存，CPU为了更好的支持虚拟内存也加入了MMU（内存管理单元）。在寻址时，需要把虚拟地址经过MMU计算得到真实的物理地址，然后再去寻址。
虚拟内存的好处：

+ 一个以上的虚拟地址可指向同一个物理内存地址。
+ 虚拟内存空间可大于实际可用的硬件内存。
## 内存页面调度
**为了支持虚拟内存，需要把内存存储到硬盘上，现代操作系统是通过内存页面调度实现的。对于采用分页技术的现代操作系统而言，这也是数据在磁盘与物理内存之间往来的唯一方式。**
下面的步骤说明了内存页面调度的过程：

+ 当 CPU 引用某内存地址时，MMU负责确定该地址所在页（往往通过对地址值进行移位或屏蔽位操作实现），并将虚拟页号转换为物理页号（这一步由硬件完成，速度极快）。如果当前不存在与该虚拟页形成有效映射的物理内存页，MMU 会向 CPU 提交一个页错误。
+ 页错误随即产生一个陷阱（类似于系统调用），把控制权移交给内核，附带导致错误的虚拟地址信息，然后内核采取步骤验证页的有效性。内核会安排页面调入操作，把缺失的页内容读回物理内存。这往往导致别的页被移出物理内存，好给新来的页让地方。在这种情况下，如果待移出的页已经被碰过了（自创建或上次页面调入以来，内容已发生改变），还必须首先执行页面调出，把页内容拷贝到磁盘上的分页区。
+ 如果所要求的地址不是有效的虚拟内存地址（不属于正在执行的进程的任何一个内存段），则该页不能通过验证，段错误随即产生。于是，控制权转交给内核的另一部分，通常导致的结果就是进程被强令关闭。
一旦出错的页通过了验证，MMU 随即更新，建立新的虚拟到物理的映射（如有必要，中断被移出页的映射），用户进程得以继续。造成页错误的用户进程对此不会有丝毫察觉，一切都在不知不觉中进行。
## 文件IO
**虚拟内存通过页面调度可以调度内存页和硬盘页，现代操作系统对文件IO的操作也是基于页面调度实现的。**
采用分页技术的操作系统执行 I/O 的全过程可总结为以下几步：

+ 确定请求的数据分布在文件系统的哪些页（磁盘扇区组）。磁盘上的文件内容和元数
据可能跨越多个文件系统页，而且这些页可能也不连续。
+ 在内核空间分配足够数量的内存页，以容纳得到确定的文件系统页。
+ 在内存页与磁盘上的文件系统页之间建立映射。
+ 为每一个内存页产生页错误。
+ 虚拟内存系统俘获页错误，安排页面调入，从磁盘上读取页内容，使页有效。
+ 一旦页面调入操作完成，文件系统即对原始数据进行解析，取得所需文件内容或属性信息。
### 内存映射文件
**为了在内核空间的文件系统页与用户空间的内存区之间移动数据，``一次以上的拷贝操作几乎总是免不了的``。这是因为，在文件系统页与用户缓冲区之间往往没有一一对应关系。**但是，还有一种大多数操作系统都支持的特殊类型的 I/O 操作，允许用户进程最大限度地利用面向页的系统 I/O 特性，并完全摒弃缓冲区拷贝。这就是内存映射 I/O。
**内存映射文件将内存页全部映射到文件的硬盘块上，存在一一映射关系。使用内存映射文件时，用户态是没有缓冲区的，只存在映射到硬盘页的内存页面缓冲区，所以是真正的ZeroCopy。而不使用内存映射文件的IO，因为直接内存和内存页缓冲区没有映射关系，所以使使用直接内存作为缓冲区也是需要最少一次copy的。**``（那文件IO时使用直接内存的好处是什么？）``
### 文件锁定
JVM实现的是进程之间的锁定，一个进程之间的多个线程之间是不锁定的。
支持共享锁、独占锁等
支持锁定文件的部分区域，粒度支持到字节。
### 流IO
流的传输一般（也不必然如此）比块设备慢，经常用于间歇性输入。多数操作系统允许把流置于非块模式，这样，进程可以查看流上是否有输入，即便当时没有也不影响它干别的。这样一种能力使得进程可以在有输入的时候进行处理，输入流闲置的时候执行其他功能。
处理流IO需要依赖操作系统的通知机制：

+ Selector通知已就绪，可以进行相关IO操作。
+ AIO通知IO操作已完成，可以处理相关业务逻辑了。

# JAVA IO 发展和总结
BIO 抽象工作良好，适应用途广泛。但是当**移动大量数据**时，这些 I/O 类可伸缩性不强，也没有提供当今大多数操作系统普遍具备的常用 I/O 功能，如**文件锁定**、**非块 I/O**、**就绪性选择**和**内存映射**。这些特性对实现可伸缩性是至关重要的，对保持与非 Java 应用程序的正常交互也可以说是必不可少的，尤其是在企业应用层面，而传统的 Java I/O 机制却没有模拟这些通用 I/O 服务。操作系统与 Java 基于流的 I/O模型有些不匹配。操作系统要移动的是大块数据（缓冲区），这往往是在硬件直接存储器存取（DMA）的协助下完成的。**BIO 类喜欢操作小块数据——单个字节、几行文本。**

+ BIO
  JDK版本：1.1
  时间：
  特点：
1. 同步阻塞
2. 面向流（字节流和字符流）
3. 不支持操作系统支持的很多IO概念和特性
4. read()操作返回所读的字节数，write操作没有返回值
5. read是SeekAble的（支持skip），write不是Seekable的。
+ NIO.1
JDK版本：1.4
  时间：2002
  特点：
1. 针对现代操作系统重新抽象和封装实现了Channel和Buffer
2. 网络IO支持配置是否阻塞，文件IO只能是阻塞的
3. 网络IO支持多路复用（异步），使用多路复用时，Channel必须是非阻塞模式，而且阻塞模式不能再被修改
4. 网络IO和文件IO都是可中断的。文件IO过程中被中断时，JVM会关闭Channel。
5. 支持直接Buffer、支持内存映射文件
6. 支持文件锁定，但是是进程级别的锁定
7. 支持Pipe
8. read操作返回所读的字节数，write操作返回所写的字节数
9. Channel是全双工的。虽然RandomAccessFile也可以是全双工的，但是Channel这种封装方式更好
+ NIO.2（AIO）
  JDK版本：1.7
  时间：2011
  特点：
1. 文件IO和网络IO是异步的（当然是非阻塞的），异步形式是future和callback。
2. Watch Service
3. 文件IO支持更丰富的API




# BIO
# NIO.1
# NIO.2
## 异步
### Watch Service
### IO

## 更丰富的文件操作


  [1]: http://oqxil93b6.bkt.clouddn.com/images/JAVA%20IO/buffer-io.png
  [2]: http://oqxil93b6.bkt.clouddn.com/images/JAVA%20IO/scatter-gather.png