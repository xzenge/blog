---
title: NIO
date: 2020-01-29 20:52:57
tags:
    - Java
    - IO
categories:
    - Technology
---

## Buffer

负责数据的存取。缓冲区及时数组。用于存储不同数据类型的数据。（boolean除外）
通过allocate()获取缓冲区
put()，get()
缓冲区中的核心属性
capacity:容量，表示缓冲区中最大存储数据的容量。一旦申明不能改变。
limit:界限，表示缓冲区中可以操作数据的大小。limit后端的数据不能进行读写。
position:位置，表示缓冲区中正在操作的位置。
0<=mark<=posiotion<=limit<=capacity
flip() 切换读取数据模式
rewind() 可重复读
clear()清空缓冲区，数据并没有被清空
mark:标记，表示记录当前position的位置。可以通过reset()恢复到mark的位置。
hasRemaining()：缓冲区中是否还有剩余的数据
<!-- more -->
### 实现类

- ByteBuffer
- CharBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer

### 直接缓冲区&非直接缓冲区

可以通过isDirect()判断是否为直接缓冲区&非直接缓冲区

- 直接缓冲区(物理内存映射文件)

  通过allocateDirec()方法分配直接缓冲区，将缓冲区建立在物理内存中。
  可以提高执行效率。
  
  物理磁盘-内核地址空间-物理内存映射文件-用户地址空间-应用程序

- 非直接缓冲区

  通过allocatie()方法分配缓冲区，将缓冲区建立在JVM的内存中。
  
  磁盘-内核地址空间-用户地址空间-应用程序

## Channel

负责缓冲区中的数据传输。Channel本身不存储数据，因此需要配合缓冲区进行传输。 
针对支持通道的泪提供了getChannel()方法：
本地IO：
FileInputStream/FileOutputStream
RandomAccessFile
网路IO：
Socket
SeverSocket
DatagramSocket

在JDK1.7中的NIO.2针对各个通道提供了静态方法open()
咋JDK1.7中的NIO.2的Files工具类的newByteChannel()

### 实现类

- FileChannel
- SocketChannel
- ServerSocketChannel
- DatagramChannel

### 通道之间的数据传输

- transferFrom()
- transferTo()

### 分散与聚集

- 分散(Scatter)

  分散读取(Scattering Reads)：将通道中的数据分散到多个缓冲区中

- 聚集(Gather)

  聚集写入(Gathering Writes)：将多个缓冲区中的数据聚集到通道中

### 字符集(Charest)

- 编码

  字符串 -> 字节数组

- 解码

  字节数组 -> 字符串

## Selector