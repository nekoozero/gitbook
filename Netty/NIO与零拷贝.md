# NIO与零拷贝

> 时间：2020/8/11

## 预备知识

关于 IO 内存映射：

设备通过控制总线、数据总线、状态总线与 CPU 相连。控制总线传送控制信号，在传统的操作中，都是通过读写设备寄存器的值来实现。但是这样耗费了 CPU 时钟。而且每次取值都要读取设备寄存器，造成了效率的低下。在现代操作系统中，引用了 IO 内存映射，即把寄存器的值映射到主存。对设备寄存器的操作转换为对主存的操作，这样极大的提高了效率。

关于 DMA

传统的处理方法为：当设备接收到数据，向 CPU 报告中断。CPU 处理中断，CPU 把数据从设备的寄存器数据读到内存。

在现代的操作系统中引入了 DMA 设备，设备接收到数据时，把数据放至 DMA 内存，再向 CPU 产生中断。CPU 再通过 DMA 读取数据，这样节省了大量的 CPU 时间。

## 正文

零拷贝时服务器网络编程的关键，任何性能优化都离不开。常用的零拷贝有 **mmap** 和 **sendFile**，它们在使用场景上也是有着不同之处的。

目标：

1. 避免操作系统内核缓冲区之间进行数据拷贝操作。
2. 避免操作系统内核和用户应用程序地址空间这两者之间进行数据拷贝操作。
3. 用户应用程序可以避开操作系统直接访问硬件存储。
4. 数据传输尽量让 DMA 来做。
5. 避免不必要的系统调用上下文切换。
6. 需要拷贝的数可以先被缓存起来。
7. 对数据进行处理尽量让硬件来做。

一般的 IO 和网络编程：

```java
File file = new File("index.html");
RandomAccessFile raf = new RandomAccessFile(file, "rw");

byte[] arr = new byte[(int) file.length()];
raf.read(arr);

Socket socket = new ServerSocket(8080).accept();
socket.getOutputStream().write(arr);
```

当调用 read 方法读取 index.html 的内容。

![OSSokcetIO.png](http://www.qxnekoo.cn:8888/images/2020/08/11/OSSokcetIO.png)

1. read 调用导致用户态到内核态的一次变化，同时，第一次复制开始：DMA（Direct Memory Access,直接内存读取，即不使用 CPU 拷贝数据到内存，而是 DMA 引擎传输数据到内存，用于解放 CPU）。引擎从磁盘读取 index.html 文件，并将数据放入到内核缓冲区。

2. 发生第二次数据拷贝，即：将内核缓冲区的数据拷贝到用户缓冲区，同时，发生了一次用内核态到用户态的上下文切换。

3. 发生第三次数据拷贝，我们调用 write 方法，系统将用户缓冲区的数据拷贝到 Socket 缓冲区。此时，又发生了一次用户态到内核态的上下文切换。

4. 第四次拷贝，数据异步的从 Socket 缓冲区，使用 DMA 引擎拷贝到网络协议引擎。这一段不需要上下文切换。

5. write 方法返回，再次从内核态切换到用户态。

   ![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yNzI3MTktOWI4MDBmNjJhOWMwZTQ3ZC5QTkc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy81NDQvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

### mmap 优化

mmap 通过**内存映射**，将文件映射到内核缓冲区，同时，**用户空间可以共享内核空间的数据**，在进行网络传输时，就可以减少内核空间到用户空间的拷贝次数。

![mmapIO.png](http://www.qxnekoo.cn:8888/images/2020/08/11/mmapIO.png)

user buffer 和 kernel buffer 共享 index.html。如果想把硬盘中的 index.html 传输到网络中，再也不用拷贝到用户空间，再从用户空间拷贝到 Socket 缓冲区。内核缓冲区拷贝到 Socket 缓冲区即可，就减少了一次内存拷贝。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yNzI3MTktYzk1NWM2MDA5NTY0N2Q2ZS5QTkc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy81NTAvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

#### 隐藏的陷阱

当程序 map 了一个文件，但是文件被另一个进程截断（truncate）时，write 系统调用会因为访问非法地址而被 **SIGBUS** 信号终止。这个信号默认会杀死你的进程并产生一个 coredump，服务器可能会被终止。

结局方案：

1. 为 SIGBUS 信号建立信号处理程序

   遇到 SIGBUS 信号时，信号处理程序简单地返回，write 系统调用在被中断之前会返回已经写入的字节数，并且 errno 会被设置成 success，这个并没有解决问题的实质核心。

2. 使用**文件租借锁**

   一般使用这种方法，在文件描述符上使用租借锁，我们为文件向内核申请一个租借锁，当其它进程想要截断这个文件时，内核会先我们发送一个实时的 RT_SIGNAL_LEASE 信号，告诉我们内核正在破坏加持在文件上的读写锁。这样在程序访问非法内存并且被 SIGBUS 杀死之前，write 系统调用会被中断。write 会返回已经写入的字节数，并且置 errno 为 success。

## sendFile

Linux 2.1 提供了 sendFile 函数，其基本原理如下，数据根本不经过用户态，直接从内核缓冲区进入到 Socket Buffer，同时，由于和用户态完全无关，就减少了一次上下文切换。

[![sendFile2-1.png](http://www.qxnekoo.cn:8888/images/2020/08/11/sendFile2-1.png)](http://www.qxnekoo.cn:8888/image/T5kc)

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yNzI3MTktNWM0OWFlYmM4NTA4NTcyNi5QTkc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy82MjYvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

在进行 sendFile 系统调用时，数据被 DMA 引擎从文件复制到内核缓冲区，然后调用 write 方法时，从内核缓冲区进入到 Socket，这是，是没有上下文切换的，因为在一个用户空间。最后数据从 Socket 缓冲区进入到协议栈。

此时数据经历了 3 次拷贝，3 次上下文切换。

在 Linux 2.4 版本中，做了一些修改，避免了从内核缓冲区拷贝到 Socket buffer 的操作，直接拷贝到协议栈，从而再一次减少了数据拷贝。

![sendFile2-4.png](http://www.qxnekoo.cn:8888/images/2020/08/11/sendFile2-4.png)

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yNzI3MTktODQ2MWNjNDE0MWM4ZGQ0NS5QTkc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy84MzMvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

现在 index.html 从文件进入到网络协议栈，只需两次拷贝：第一次使用 DMA 引擎从文件拷贝到内核缓冲区，第二次从内核缓冲区将数据拷贝到网络协议栈；内核缓冲区只会拷贝一些 offset 和 length 信息到 SocketBuffer，基本无消耗。

 零拷贝不仅仅带来更少的数据复制，还能带来其他的性能优势，例如更少的上下文切换，更少的 CPU 缓存伪共享以及无 CPU 校验和计算。



## 区别

1. mmap 适合小数据量读写，sendFile 适合大文件传输。
2. mmap 需要 4 次上下文切换，3 次数据拷贝；sendFile 需要 3 次上下文切换，最少 2 次数据拷贝。
3. sendFile 可以利用 DMA 方式，减少 CPU 拷贝，mmap 则不能（必须从内核拷贝到 Socket 缓冲区）。

在这个选择上：rocketMQ 在消费消息时，使用 mmap。kafka 使用了 sendFile。



1. java NIO

   - MappedByteBuffer

     是 NIO 基于内存映射（mmap）这种零拷贝方式提供的一种实现，意思是把一个文件从 position 位置开始的 size 大小的区域映射为内存映射文件。这样添加地址映射，而不进行拷贝。

   - DirectByteBuffer

     DirectByteBuffer 的对象引用位于 java 内存模型的堆里面，JVM 可以对 DirectByteBuffer 的对象进行内存分配和回收管理，是 MappedByteBuffer 的具体实现类，同样具有零拷贝技术。

   - FileChannel

     定义了 transferFrom() 和 transferTo() 两个抽象方法，它通过在通道和通道之间建立连接实现数据传输的。

     Linux 2.4 版本，socket 缓冲区做了调整，DMA 带收集功能。

     - 从 DMA 拷贝至内核缓冲区
     - 将数据的位置和长度信息的描述父增加值内核空间（socket 缓冲区）
     - DMA 将数据从内核拷贝至协议引擎

2. Netty

   Netty 中的零拷贝和上面提到的操作系统层面上的零拷贝不太一样, 我们所说的 Netty 零拷贝完全是基于（Java 层面）用户态的。

   （1）Netty 通过 DefaultFileRegion 类对 FileChannel 的 tranferTo() 方法进行包装，相当于是间接的通过 java 进行零拷贝。

   （2）我们的数据传输一般都是通过 TCP/IP 协议实现的，在实际应用中，很有可能**一条完整的消息被分割为多个数据包进行网络传输，而单个的数据包对你而言是没有意义的，只有当这些数据包组成一条完整的消息时你才能做出正确的处理**，而 Netty 可以通过零拷贝的方式将这些数据包组合成一条完整的消息供你来使用。

   此时零拷贝的作用范围仅在用户空间中。那 Netty 是如何实现的呢？为此我们就要找到 Netty 进行数据传输的接口，这个接口一定包含了可以实现零拷贝的功能，这个接口就是 ChannelBuffer。

   **既然有接口肯定就有实现类，一个最主要的实现类是 CompositeChannelBuffer，这个类的主要作用是将多个 ChannelBuffer 组成一个虚拟的 ChannelBuffer 来进行操作**

   **为什么说是虚拟的呢，因为 CompositeChannelBuffer 并没有将多个 ChannelBuffer 真正的组合起来，而只是保存了他们的引用，这样就避免了数据的拷贝，实现了 Zero Copy。**

   （3）ByteBuf 可以通过 wrap 操作把字节数组、ByteBuf、ByteBuffer 包装成一个 ByteBuf 对象, 进而避免了拷贝操作

   （4）ByteBuf 支持 slice 操作, 因此可以将 ByteBuf 分解为多个共享同一个存储区域的 ByteBuf，避免了内存的拷贝。

MappedByteBuffer、HeapByteBuffer、DirectByteBuffer。

sendFile 实现的零拷贝：transferTo、transferFrom

