---
title: Channel
date: 2017-06-14 22:59:47
tags: [java,io]
categories: io
---
**本文内容主要摘录自：《JAVA NIO》。**

Channel的类图如下：
![Channel-class][1]

# 通道基础

1. 从 Channel 接口引申出的其他接口都是面向字节的子接口，包括 WritableByteChannel和ReadableByteChannel.这也正好支持了我们之前所学的：通道只能在字节缓冲区上操作。
2. SelectableChannel和InterruptibleChannel

## 打开通道
+ ``SocketChannel sc = SocketChannel.open( );``
+ ``ServerSocketChannel ssc = ServerSocketChannel.open( );``
+ ``DatagramChannel dc = DatagramChannel.open( );``
+ RandomAccessFile、 FileInputStream 或 FileOutputStream对象上调用 getChannel( )方法来获取
```java
RandomAccessFile raf = new RandomAccessFile ("somefile", "r");
FileChannel fc = raf.getChannel( );
```

## 使用通道

1. 通道可以是单向的，也可以是双向的，由底层的打开方式决定。
2. ByteChannel 接口本身并不定义新的 API 方法，它是一种用来聚集它自己以一个新名称继承的多个接口的便捷接口。
3. **通道可以以阻塞（blocking）或非阻塞（nonblocking）模式运行。**非阻塞模式的通道永远不会让调用的线程休眠。请求操作要么立即完成，要么返回一个结果表明未进行任何操作。
    **只有面向流的（stream-oriented）的通道，如 sockets 和 pipes 才能使用非阻塞模式。**

## 关闭通道

1. **调用通道的close( )方法时，可能会导致在通道关闭底层I/O服务的过程中线程暂时阻塞,哪怕该通道处于非阻塞模式。**
    通道关闭时的阻塞行为（如果有的话）是高度取决于操作系统或者文件系统的。**在一个通道上多次调用close( )方法是没有坏处的**，但是如果第一个线程在close( )方法中阻塞，那么在它完成关闭通道之前，任何其他调用close( )方法都会阻塞。后续在该已关闭的通道上调用close( )不会产生任何操作，只会立即返回。
2. 通道引入了一些与**关闭和中断有关的新行为**。
    如果一个通道实现 InterruptibleChannel接口，它的行为以下述语义为准：**如果一个线程在一个通道上被阻塞并且同时被中断（由调用该被阻塞线程的 interrupt( )方法的另一个线程中断），那么该通道将被关闭，该被阻塞线程也会产生一个 ClosedByInterruptException 异常。**
3. **请不要将在 Channels 上休眠的中断线程同在 Selectors上休眠的中断线程混淆。前者会关闭通道，而后者则不会。**
    不过，如果您的线程在 Selector 上休眠时被中断，那它的 interrupt status 会被设置。假设那个线程接着又访问一个Channel，则该通道会被关闭。
4. 仅仅因为休眠在其上的线程被中断就关闭通道，这看起来似乎过于苛刻了。
    不过这却是 NIO架构师们所做出的明确的设计决定。经验表明，想要在所有的操作系统上一致而可靠地处理被中断的 I/O操作是不可能的。 java.nio 包中强制使用此行为来避免因操作系统独特性而导致的困境，因为该困境对 I/O 区域而言是极其危险的。**这也是为增强健壮性（robustness）而采用的一种经典的权衡。**
5. **可中断的通道也是可以异步关闭的。**
    实现 InterruptibleChannel 接口的通道可以在任何时候被关闭，即使有另一个被阻塞的线程在等待该通道上的一个 I/O 操作完成。当一个通道被关闭时，休眠在该通道上的所有线程都将被唤醒并接收到一个 AsynchronousCloseException异常。接着通道就被关闭并将不再可用。

# Scanner/Gather

通道提供了一种被称为 Scatter/Gather的重要新功能（有时也被称为矢量 I/O）。 

Scatter/Gather是一个简单却强大的概念，它是指在多个缓冲区上实现一个简单的 I/O 操作。
**对于一个 write 操作而言**，数据是从几个缓冲区按顺序抽取（称为 gather）并沿着通道发送的。缓冲区本身并不需要具备这种 gather 的能力（通常它们也没有此能力）。该 gather过程的效果就好比全部缓冲区的内容被连结起来，并在发送数据前存放到一个大的缓冲区中。
**对于 read 操作而言**，从通道读取的数据会按顺序被散布（称为 scatter）到多个缓冲区，将每个缓冲区填满直至通道中的数63据或者缓冲区的最大空间被消耗完。大多数现代操作系统都支持本地矢量 I/O（native vectored I/O）。当您在一个通道上请求一个Scatter/Gather操作时，该请求会被翻译为适当的本地调用来直接填充或抽取缓冲区。这是一个很大的进步，因为减少或避免了缓冲区拷贝和系统调用。 

Scatter:
```java
public interface ScatteringByteChannel
extends ReadableByteChannel
{
public long read (ByteBuffer [] dsts)
throws IOException;
public long read (ByteBuffer [] dsts, int offset, int length)
throws IOException;
}
```
Gather:
```java
public interface GatheringByteChannel
extends WritableByteChannel
{
public long write(ByteBuffer[] srcs)
throws IOException;
public long write(ByteBuffer[] srcs, int offset, int length)
throws IOException;
}
```
使用得当的话， **Scatter/Gather 会是一个极其强大的工具。它允许您委托操作系统来完成辛苦活：将读取到的数据分开存放到多个存储桶（bucket）或者将不同的数据区块合并成一个整体。这是一个巨大的成就，因为操作系统已经被高度优化来完成此类工作了。它节省了您来回移动数据的工作，也就<font color="red">避免了缓冲区拷贝(为啥？)</font>和减少了您需要编写、调试的代码数量。**

# 文件通道

## 基本操作

![Filechannel-class][2]
 
FileChannel原型，如下：
```java
public abstract class FileChannel
    extends AbstractChannel
    implements ByteChannel, GatheringByteChannel, ScatteringByteChannel
{
// This is a partial API listing
public abstract long position( )
public abstract void position (long newPosition)
public abstract int read (ByteBuffer dst)
public abstract int read (ByteBuffer dst, long position)
public abstract int write (ByteBuffer src)
public abstract int write (ByteBuffer src, long position)
public abstract long size( )
public abstract void truncate (long size)
public abstract void force (boolean metaData)
}
```
    
1. 文件通道总是阻塞式的，因此不能被置于非阻塞模式。
    现代操作系统都有复杂的缓存和预取机制，使得本地磁盘I/O操作延迟很少。网络文件系统一般而言延迟会多些，不过却也因该优化而受益。 **面向流的 I/O的非阻塞范例对于面向文件的操作并无多大意义，这是由文件I/O本质上的不同性质造成的。**
2. FileChannel 对象是线程安全（thread-safe）的。
3. File IO的比较
    ![file-io-opeartion-compare][3]
4. 文件空洞
    当磁盘上一个文件的分配空间小于它的文件大小时会出现“文件空洞”。对于内容稀疏的文件，大多数现代文件系统只为实际写入的数据分配磁盘空间（更准确地说，只为那些写入数据的文件系统页分配空间）。假如数据被写入到文件中非连续的位置上，这将导致文件出现在逻辑上不包含数据的区域（即“空洞”）。**如果该文件被顺序读取的话，所有空洞都会被“0”填充但不占用磁盘空间。**
5. truncate()
    当需要减少一个文件的 size 时， truncate( )方法会砍掉您所指定的新 size 值之外的所有数据。
    如果当前 size 大于新 size，超出新 size 的所有字节都会被悄悄地丢弃。
    如果提供的新 size 值大于或等于当前的文件 size 值，该文件不会被修改。
   **这两种情况下， truncate( )都会产生副作用：文件的position 会被设置为所提供的新 size 值。**

## 文件锁定

有关 FileChannel 实现的文件锁定模型的一个重要注意项是：**锁的对象是文件而不是通道或线程，这意味着文件锁不适用于判优同一台 Java 虚拟机上的多个线程发起的访问。**
如果一个线程在某个文件上获得了一个独占锁，然后第二个线程利用一个单独打开的通道来请求该文件的独占锁，那么第二个线程的请求会被批准。
但如果这两个线程运行在不同的 Java 虚拟机上，那么第二个线程会阻塞，因为锁最终是由操作系统或文件系统来判优的并且几乎总是在进程级而非线程级上判优。锁都是与一个文件关联的，而不是与单个的文件句柄或通道关联。锁与文件关联，而不是与通道关联。**我们使用锁来判优外部进程，而不是判优同一个 Java 虚拟机上的线程。**

## 内存映射文件

1. 因为不需要做明确的系统调用，那会很消耗时间。
    更重要的是，操作系统的虚拟内存可以自动缓存内存页（memory page）。这些页是用系统内存来缓存的，所以不会消耗 Java 虚拟机内存堆（memory heap）。
2. 与文件锁的范围机制不一样，映射文件的范围不应超过文件的实际大小。
    **如果您请求一个超出文件大小的映射，文件会被增大以匹配映射的大小(文件空洞)。**即使您请求的是一个只读映射，map( )方法也会尝试这样做并且大多数情况下都会抛出一个 IOException 异常，因为底层的文件不能被修改。该行为同之前讨论的文件“空洞”的行为是一致的。
3. MapMode.READ_ONLY 
4. MapMode.READ_WRITE 
5. MapMode.PRIVATE(写时拷贝)
    + 您通过 put( )方法所做的任何修改都会导致产生一个私有的数据拷贝，并且该拷贝中的数据只有MappedByteBuffer 实例可以看到。该过程不会对底层文件做任何修改，而且一旦缓冲区被施以垃圾收集动作（garbage collected），那些修改都会丢失。
    + **尽管写时拷贝的映射可以防止底层文件被修改，您也必须以 read/write 权限来打开文件**以建立 MapMode.PRIVATE 映射。只有这样，返回的MappedByteBuffer 对象才能允许使用 put( )方法。
    + 选择使用 MapMode.PRIVATE 模式并不会导致您的缓冲区看不到通过其他方式对文件所做的修改。
        对文件某个区域的修改在使用 MapMode.PRIVATE模式的缓冲区中都能反映出来，除非该缓冲区已经修改了文件上的同一个区域。如果缓冲区还没对某个页做出修改，那么这个页就会反映被映射文件的相应位置上的内容。一旦某个页因为写操作而被拷贝，之后就将使用该拷贝页，并且不能被其他缓冲区或文件更新所修改。
    + 关闭相关联的 FileChannel 不会破坏映射，只有丢弃缓冲区对象本身才会破坏该映射。
        **NIO设计师们之所以做这样的决定是因为当关闭通道时破坏映射会引起安全问题，而解决该安全问题又会导致性能问题。** ***如果您确实需要知道一个映射是什么时候被破坏的，他们建议使用虚引用（phantomreferences，参见java.lang.ref.PhantomReference）和一个 cleanup 线程。***不过有此需要的概率是微乎其微的。
6. load()
    在一个映射缓冲区上调用 load()方法会是一个**代价高**的操作，因为它会导致大量的页调入（page-in），具体数量取决于文件中被映射区域的实际大小。
   **然而， load( )方法返回并不能保证文件就会完全常驻内存，这是由于请求页面调入（demandpaging）是动态的。**具体结果会因某些因素而有所差异，这些因素包括：操作系统、文件系统，可用 Java 虚拟机内存，最大 Java 虚拟机内存，垃圾收集器实现过程等等。请小心使用 load( )方法，它可能会导致您不希望出现的结果。该方法的主要作用是为提前加载文件埋单，以便后续的访问速度可以尽可能的快。
7. isLoaded()
     isLoaded( )方法来判断一个被映射的文件是否完全常驻内存了。**不过，这也是不能保证的。**同样地，返回 false 值并不一定意味着访问缓冲区将很慢或者该文件并未完全常驻内存。 isLoaded()方法的返回值只是一个暗示，由于垃圾收集的异步性质、底层操作系统以及运行系统的动态性等因素，**想要在任意时刻准确判断全部映射页的状态是不可能的。**
8. force()
该方法会强制将映射缓冲区上的更改应用到永久磁盘存储器上。当用 MappedByteBuffer 对象来更新一个文件，**您应该总是使用 MappedByteBuffer.force( ),而非 FileChannel.force( )，因为通道对象可能不清楚通过映射缓冲区做出的文件的全部更改。**
**MappedByteBuffer 没有不更新文件元数据的选项——元数据总是会同时被更新的。**
**如果映射是以 MapMode.READ_ONLY 或 MAP_MODE.PRIVATE 模式建立的，那么调用 force()方法将不起任何作用，因为永远不会有更改需要应用到磁盘上（但是这样做也是没有害处的）。**

## Channel-to-Channel传输

1. transferTo( )和 transferFrom( )方法允许将一个通道交叉连接到另一个通道，而不需要通过一个中间缓冲区来传递数据。
2. **只有 FileChannel 类有这两个方法，因此 channel-to-channel 传输中通道之一必须是 FileChannel。您不能在 socket 通道之间直接传输数据**，不过 socket 通道实现WritableByteChannel 和 ReadableByteChannel接口，因此文件的内容可以用 transferTo( )方法传输给一个 socket 通道，或者也可以用 transferFrom( )方法将数据从一个 socket通道直接读取到一个文件中。
3. **直接的通道传输不会更新与某个 FileChannel 关联的 position 值。**
4.  + 对于传输数据来源是一个文件的transferTo()方法，如果position+count的值大于文件的size值，传输会在文件尾的位置终止。
    + 假如传输的目的地是一个非阻塞模式的socket通道，那么当发送队列（sendqueue）满了之后传输就可能终止，并且如果输出队列（output queue）已满的话可能不会发送任何数据。
    + 类似地，对于 transferFrom( )方法：如果来源 src是另外一个FileChannel并且已经到达文件尾，那么传输将提早终止；如果来源 src 是一个非阻塞 socket通道，只有当前处于队列中的数据才会被传输（可能没有数据）。**由于网络数据传输的非确定性，阻塞模式的socket 也可能会执行部分传输，这取决于操作系统。许多通道实现都是提供它们当前队列中已有的数据而不是等待您请求的全部数据都准备好。**

# Socket通道

![socket-channel-class][4]

1. 使用Channel没有为每个 socket连接使用一个线程的必要了，也避免了管理大量线程所需的上下文交换总开销。
2. 请注意 DatagramChannel 和SocketChannel实现定义读和写功能的接口而ServerSocketChannel不实现。ServerSocketChannel 负责监听传入的连接和创建新的 SocketChannel 对象，它本身从不传输数据。
3. **虽然每个 socket 通道（在 java.nio.channels 包中）都有一个关联的 java.net socket 对象，却并非所有的 socket 都有一个关联的通道。**如果您用传统方式（直接实例化）创建了一个Socket 对象，它就不会有关联的 SocketChannel并且它的 getChannel( )方法将总是返回 null。

## 非阻塞模式

 Socket 通道可以工作在阻塞和非阻塞模式下，并且可以在运行过程中动态切换。
SelectableChannel:
```java
public abstract class SelectableChannel
extends AbstractChannel
implements Channel
{
// This is a partial API listing
public abstract void configureBlocking (boolean block)
throws IOException;
public abstract boolean isBlocking( );99
public abstract Object blockingLock( );
}
```
1. 要把一个 socket 通道置于非阻塞模式，我们要依靠所有 socket 通道类的公有超级类：SelectableChannel。
    **非阻塞 I/O 和可选择性是紧密相连的，那也正是管理阻塞模式的 API 代码要在SelectableChannel 超级类中定义的原因。**
2. blockingLock( )
    该方法会返回一个非透明的对象引用。返回的对象是通道实现修改阻塞模式时内部使用的。只有拥有此对象的锁的线程才能更改通道的阻塞模式。

## ServerSocketChannel

```java
public abstract class ServerSocketChannel
extends AbstractSelectableChannel
{
public static ServerSocketChannel open( ) throws IOException
public abstract ServerSocket socket( );
public abstract SocketChannel accept( ) throws IOException;
public final int validOps( )
}
```
1. 用静态的 open( )工厂方法创建一个新的 ServerSocketChannel 对象，将会返回与一个未绑定的java.net.ServerSocket 关联的通道。该对等 ServerSocket可以通过在返回的ServerSocketChannel上调用socket()方法来获取。作为ServerSocketChannel 的对等体被创建的 ServerSocket对象依赖通道实现。这些socket关联的SocketImpl能识别通道。通道不能被封装在随意的 socket 对象外面
2. accept
    + **如果您选择在 ServerSocket 上调用 accept( )方法**
        那么它会同任何其他的 ServerSocket 表现一样的行为：总是阻塞并返回一个 java.net.Socket 对象。
    + 如果您选择在 ServerSocketChannel 上调用 accept( )
        方法则会返回 SocketChannel 类型的对象，返回的对象能够在非阻塞模式下运行。
    + 如果在非阻塞模式下的ServerSocketChannel上调用
        当没有传入连接在等待时， ServerSocketChannel.accept( )会立即返回 null。

## SocketChannel

SocketChannel原型：
```java
public abstract class SocketChannel
extends AbstractSelectableChannel
implements ByteChannel, ScatteringByteChannel, GatheringByteChannel
{
// This is a partial API listing
public static SocketChannel open( ) throws IOException
public static SocketChannel open (InetSocketAddress remote)
throws IOException
public abstract Socket socket( );
public abstract boolean connect (SocketAddress remote)
throws IOException;103
public abstract boolean isConnectionPending( );
public abstract boolean finishConnect( ) throws IOException;
public abstract boolean isConnected( );
public final int validOps( )
}
```
1. connect()
    + 阻塞模式下：
        在Socket对象上调用和通过在阻塞模式的SocketChannel上调用时相同：线程在连接建立好或超时过期之前都将保持阻塞。
    + 非阻塞模式的SocketChannel上调用
        线程不阻塞，如果返回值是true，说明连接立即建立了（这可能是本地环回连接）；如果连接不能立即建立， connect( )方法会返回 false 且并发地继续连接建立过程。
2. **在 SocketChannel 上并没有一种connect()方法可以让您指定超时（timeout）值，编程时一般使用非阻塞模式的SocketChannel.connect()建立异步连接。**
3. isConnectPending()
4. **finishConnect()**
    + connect( )方法尚未被调用。那么将产生 NoConnectionPendingException 异常。
    + 连接建立过程正在进行，尚未完成。那么什么都不会发生， finishConnect( )方法会立即返回false 值。
    + 在非阻塞模式下调用 connect()方法之后，SocketChannel又被切换回了阻塞模式。那么如果有必要的话，调用线程会阻塞直到连接建立完成， finishConnect( )方法接着就会返回 true值。
    + 在初次调用 connect( )或最后一次调用finishConnect()之后，连接建立过程已经完成。那么SocketChannel对象的内部状态将被更新到已连接状态， finishConnect( )方法会返回 true值，然后 SocketChannel对象就可以被用来传输数据了。
    + 连接已经建立。那么什么都不会发生，finishConnect()方法会返回true值。假如某个SocketChannel上当前正由一个并发连接， isConnectPending( )方法就会返回 true 值。
5. isConnected( )
    不阻塞
6. 当通道处于中间的连接等待（connection-pending）状态时，您只可以调用 finishConnect( )、isConnectPending( )或isConnected( )方法。
7. **如果尝试异步连接失败，那么下次调用finishConnect()方法会产生一个适当的经检查的异常以指出问题的性质。通道然后就会被关闭并将不能被连接或再次使用.**
8. **connect( )和 finishConnect()方法是互相同步的，并且只要其中一个操作正在进行，任何读或写的方法调用都会阻塞,即使是在非阻塞模式下。如果此情形下您有疑问或不能承受一个读或写操作在某个通道上阻塞，请用isConnected()方法测试一下连接状态。**

## DatagramChannel

TCP/IP 这样面向流的的协议为了在包导向的互联网基础设施上维护流语义必然会产生巨大的开销，并且流隐喻不能适用所有的情形。**数据报的吞吐量要比流协议高很多， 并且数据报可以做很多流无法完成的事情。**

DatagramChannel的原型：
```java
public abstract class DatagramChannel
extends AbstractSelectableChannel
implements ByteChannel, ScatteringByteChannel, GatheringByteChannel
{
// This is a partial API listing
public static DatagramChannel open( ) throws IOException
public abstract DatagramSocket socket( );
public abstract DatagramChannel connect (SocketAddress remote)
throws IOException;
public abstract boolean isConnected( );
public abstract DatagramChannel disconnect( ) throws IOException;
public abstract SocketAddress receive (ByteBuffer dst)
throws IOException;107
public abstract int send (ByteBuffer src, SocketAddress target)
public abstract int read (ByteBuffer dst) throws IOException;
public abstract long read (ByteBuffer [] dsts) throws IOException;
public abstract long read (ByteBuffer [] dsts, int offset,
int length)
throws IOException;
public abstract int write (ByteBuffer src) throws IOException;
public abstract long write(ByteBuffer[] srcs) throws IOException;
public abstract long write(ByteBuffer[] srcs, int offset,
int length)
throws IOException;
}
```

1. DatagramChannel 对象既可以充当服务器（监听者）也可以充当客户端（发送者）。
    + 如果您希望新创建的通道负责监听，那么通道必须首先被绑定到一个端口或地址/端口组合上。
    + **不论通道是否绑定，所有发送的包都含有 DatagramChannel 的源地址（带端口号）。**
    + **未绑定的 DatagramChannel可以接收发送给它的端口的包，通常是来回应该通道之前发出的一个包。通道绑定了端口或者发送了数据包之后就有了相应端口，否则没有端口，也就不能接收数据。**
2. 与面向流的的 socket 不同， DatagramChannel 可以发送单独的数据报给不同的目的地址。同样， DatagramChannel 对象也可以接收来自任意地址的数据包。每个到达的数据报都含有关于它来自何处的信息（源地址）。
3. receive()
    receive( )方法将下次将传入的数据报的数据净荷复制到预备好的ByteBuffer中并返回一个SocketAddress对象以指出数据来源。
    + 如果通道处于阻塞模式， receive( )可能无限期地休眠直到有包到达。
    + 如果是非阻塞模式，当没有可接收的包时则会返回 null。
    + **假如您提供的 ByteBuffer 没有足够的剩余空间来存放您正在接收的数据包，没有被填充的字节都会被悄悄地丢弃**
4. send()
    如果 DatagramChannel 对象处于阻塞模式，调用线程可能会休眠直到数据报被加入传输队列。
    **发送数据报是一个全有或全无（all-or-nothing）的行为。**
    + **如果通道是非阻塞的，返回值要么是字节缓冲区的字节数，要么是“0”。**
    + 如果传输队列没有足够空间来承载整个数据报，那么什么内容都不会被发送。
5. connect()语义
    + 将 DatagramChannel 置于已连接的状态可以使除了它所“连接”到的地址之外的任何其他源地址的数据报被忽略。这是很有帮助的，因为不想要的包都已经被网络层丢弃了，从而避免了使用代码来接收、检查然后丢弃包的麻烦。
    + 当 DatagramChannel 已连接时，使用同样的令牌，您不可以发送包到除了指定给connect()方法的目的地址以外的任何其他地址。试图一定要这样做的话会导致一个 **SecurityException 异常**。
    + 已连接通道会发挥作用的使用场景之一是一个客户端/服务器模式、使用 UDP 通讯协议的实时游戏。每个客户端都只和同一台服务器进行会话而希望忽视任何其他来源地数据包。将客户端的DatagramChannel 实例置于已连接状态可以减少按包计算的总开销（因为不需要对每个包进行安全检查）和剔除来自欺骗玩家的假包。服务器可能也想要这样做，不过需要每个客户端都有一个DatagramChannel 对象。
    + **DatagramChannel 没有单独的 finishConnect( )方法。我们可以使用isConnected()方法来测试一个数据报通道的连
接状态。**
    + **DatagramChannel 对象可以任意次数地进行连接或断开连接。每次连接都可以到一个不同的远程地址。
        调用 disconnect( )方法可以配置通道，以便它能再次接收来自安全管理器（如果已安装）所允许的任意远程地址的数据或发送数据到这些地址上。**
    + 当一个 DatagramChannel 处于已连接状态时，发送数据将不用提供目的地址而且接收时的源地址也是已知的。这意味着DatagramChannel已连接时可以使用常规的 read( )和 write( )方法，包括scatter/gather形式的读写来组合或分拆包的数据。
    + read( )方法返回读取字节的数量，如果通道处于非阻塞模式的话这个返回值可能是“0”。write( )方法的返回值同 send( )方法一致：**要么返回您的缓冲区中的字节数量，要么返回“0”**（如果由于通道处于非阻塞模式而导致数据报不能被发送）。**当通道不是已连接状态时调用 read( )或write( )方法，都将产生 NotYetConnectedException 异常。**

# 管道

Pipe 类创建一对提供环回机制的 Channel 对象。这两个通道的远端是连接起来的，以便任何写在 SinkChannel 对象上的数据都能出现在 SourceChannel 对象上。

![pipe-class][5]

Pipe原型：
```java
package java.nio.channels;
public abstract class Pipe
{
public static Pipe open( ) throws IOException
public abstract SourceChannel source( );
public abstract SinkChannel sink( );119
public static abstract class SourceChannel
extends AbstractSelectableChannel
implements ReadableByteChannel, ScatteringByteChannel
public static abstract class SinkChannel
extends AbstractSelectableChannel
implements WritableByteChannel, GatheringByteChannel
}
```

1. Pipe.SourceChannel（管道负责读的一端）， Pipe.SinkChannel（管道负责写的一端）。
    这两个通道实例是在 Pipe 对象创建的同时被创建的，可以通过在 Pipe 对象上分别调用 source( )和 sink( )方法来取回。
2. **SinkChannel 和 SourceChannel 都由 AbstractSelectableChannel 引申而来（所以也是从 SelectableChannel 引申而来），这意味着 pipe 通道可以同选择器一起使用.**
3. **管道可以被用来仅在同一个 Java虚拟机内部传输数据（不能在外部）。**虽然有更加有效率的方式来在线程之间传输数据，**但是使用管道的好处在于封装性。**
4. **生产者线程和用户线程都能被写道通用的Channel API中。根据给定的通道类型，相同的代码可以被用来写数据到一个文件、socket或管道。选择器可以被用来检查管道上的数据可用性，如同在socket通道上使用那样地简单。这样就可以允许单个用户线程使用一个Selector来从多个通道有效地收集数据，并可任意结合网络连接或本地工作线程使用。因此，这些对于可伸缩性、冗余度以及可复用性来说无疑都是意义重大的。**
5. **管道所能承载的数据量是依赖实现的（implementation-dependent）。唯一可保证的是写到SinkChannel中的字节都能按照同样的顺序在 SourceChannel 上重现。**

# 通道工具类

![channel-util-table][6]

**这些方法返回的包封 Channel 对象可能会也可能不会实现 InterruptibleChannel 接口，它们也可能不是从 SelectableChannel 引申而来。因此，可能无法将这些包封通道同 java.nio.channels包中定义的其他通道类型交换使用。**

  [1]: http://oqxil93b6.bkt.clouddn.com/images/IO/Channel-class.png
  [2]: http://oqxil93b6.bkt.clouddn.com/images/IO/FileChanel-class.png
  [3]: http://oqxil93b6.bkt.clouddn.com/images/IO/file-io-operation-compare.png
  [4]: http://oqxil93b6.bkt.clouddn.com/images/IO/Socket-Channel-class.png
  [5]: http://oqxil93b6.bkt.clouddn.com/images/IO/pipe-class.png
  [6]: http://oqxil93b6.bkt.clouddn.com/images/IO/channel-util-table.png