---
title: FileChannel
date: 2017-06-12 14:21:58
tags: [IO]
---
FileChannel主要进行文件IO的操作，下面主要分析write()的整个过程，read()和write()是相同的。
# 概要　
write()/read()时:

+ 如果传入的是DirectBuffer，则不需要进行数据复制，最后通过write系统调用完成IO。
+ 如果传入的是非DirectBuffer，则先从本线程的DirectBuffer池中得到一个满足（size能容纳所有数据）的Buffer,之后进行数据复制，并使用DirectBuffer参与write系统调用完成IO。

**对于第二种情况，数据复制会引起效率。如果业务代码再不重复利用所传入的非DirectBuffer，则会增加GC频率。
第二种情况，线程里有一个DirectBuffer池，使得DirectBuffer可以重复利用。这样不仅可以减小DirectBuffer的新建和释放开销，而且可以减小GC频率。这给我们以借鉴，我们在编写业务代码时页应该这样处理。**

下面分析具体的源码。
# 源码
sun.nio.ch.FileChannelImpl.write()
```java
    public int write(ByteBuffer src) throws IOException {
        ensureOpen();
        if (!writable)
            throw new NonWritableChannelException();
        synchronized (positionLock) {
            int n = 0;
            int ti = -1;
            Object traceContext = IoTrace.fileWriteBegin(path);
            try {
                begin();
                ti = threads.add();
                if (!isOpen())
                    return 0;
                do {
                    //这里写
                    n = IOUtil.write(fd, src, -1, nd);
                } while ((n == IOStatus.INTERRUPTED) && isOpen());
                return IOStatus.normalize(n);
            } finally {
                threads.remove(ti);
                end(n > 0);
                IoTrace.fileWriteEnd(traceContext, n > 0 ? n : 0);
                assert IOStatus.check(n);
            }
        }
    }
```
sun.nio.ch.IOUtil.write()
```java
  static int write(FileDescriptor fd, ByteBuffer src, long position,
                     NativeDispatcher nd)
        throws IOException
    {
        //如果是DirectBuffer
        if (src instanceof DirectBuffer)
            return writeFromNativeBuffer(fd, src, position, nd);

        // Substitute a native buffer
        int pos = src.position();
        int lim = src.limit();
        assert (pos <= lim);
        int rem = (pos <= lim ? lim - pos : 0);
        //从本线程所缓存的临时DirectBuffer池中得到一个size满足的Buffer（如果没有符合的，new一个）
        ByteBuffer bb = Util.getTemporaryDirectBuffer(rem);
        try {
            //数据copy
            bb.put(src);
            bb.flip();
            // Do not update src until we see how many bytes were written
            src.position(pos);
            //真正的写
            int n = writeFromNativeBuffer(fd, bb, position, nd);
            if (n > 0) {
                // now update src
                src.position(pos + n);
            }
            return n;
        } finally {
            //将本次使用的DirectBuffer重新还给DirectBuffer池
            Util.offerFirstTemporaryDirectBuffer(bb);
        }
    }
```
接上面，sun.nio.ch.Util.getTemporaryDirectBuffer()。
**下面代码说明了每个线程维持了一个DirectBuffer池，当使用的是Heap类型的Buffer进行IO时，需要先从池中得到一个DirectBuffer,之后还有进行数据复制等，并使用DirectBuffer参与系统调用完成IO。**
```java
    /**
     * Returns a temporary buffer of at least the given size
     */
    static ByteBuffer getTemporaryDirectBuffer(int size) {
        BufferCache cache = bufferCache.get();
        ByteBuffer buf = cache.get(size);
        if (buf != null) {
            return buf;
        } else {
            // No suitable buffer in the cache so we need to allocate a new
            // one. To avoid the cache growing then we remove the first
            // buffer from the cache and free it.
            if (!cache.isEmpty()) {
                buf = cache.removeFirst();
                free(buf);
            }
            return ByteBuffer.allocateDirect(size);
        }
    }
```

sun.nio.ch.IOUtil.writeFromNativeBuffer()
```java
 private static int writeFromNativeBuffer(FileDescriptor fd, ByteBuffer bb,
                                             long position, NativeDispatcher nd)
        throws IOException
    {
        int pos = bb.position();
        int lim = bb.limit();
        assert (pos <= lim);
        int rem = (pos <= lim ? lim - pos : 0);

        int written = 0;
        if (rem == 0)
            return 0;
        if (position != -1) {
           //不从Buffer头开始写
           //sun.nio.ch.FileDispatcherImpl.writeFromNativeBuffer()
            written = nd.pwrite(fd,
                                ((DirectBuffer)bb).address() + pos,
                                rem, position);
        } else {
           //从Buffer头开始
           //sun.nio.ch.FileDispatcherImpl.writeFromNativeBuffer()
            written = nd.write(fd, ((DirectBuffer)bb).address() + pos, rem);
        }
        if (written > 0)
            bb.position(pos + written);
        return written;
    }
```
sun.nio.ch.FileDispatcherImpl.writeFromNativeBuffer()
```java
    int write(FileDescriptor fd, long address, int len) throws IOException {
        //native调用
        //源码在openjdk/jdk/src/solaris/native/sun/nio/ch/FileDispatcherImpl.c里面
        return write0(fd, address, len);
    }
```
接上面，native调用：
```c
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_write0(JNIEnv *env, jclass clazz,
                              jobject fdo, jlong address, jint len)
{
    jint fd = fdval(env, fdo);
    void *buf = (void *)jlong_to_ptr(address);
    //write的原型是
　　//extern ssize_t write (int __fd, const void *__buf, size_t __n) __wur;
    return convertReturnVal(env, write(fd, buf, len), JNI_FALSE);
}
```
**所以最后，通过系统调用完成IO.操作系统调用接口的原型是：**
```c
/* Write N bytes of BUF to FD.  Return the number written, or -1.

   This function is a cancellation point and therefore not marked with
   __THROW.  */
extern ssize_t write (int __fd, const void *__buf, size_t __n) __wur;
```


