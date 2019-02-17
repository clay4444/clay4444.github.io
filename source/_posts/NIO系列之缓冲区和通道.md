---
title: NIO系列之缓冲区和通道
categories:
  - 多线程编程
tags:
  - NIO
abbrlink: 65ebcdd2
date: 2017-09-04 15:59:15
---

### 基本概念

#### 1.简介：

Java NIO（ New IO） 是从Java 1.4版本开始引入的一个新的IO API，可以替代标准的Java IO API。NIO与原来的IO有同样的作用和目的，但是使用的方式完全不同， NIO支持面向缓冲区的、基于通道的IO操作。 NIO将以更加高效的方式进行文件的读写操作。

<br/>

#### 2.与IO的主要区别

{% asset_img clipboard.png %}

**只要是IO，肯定就是完成数据的传输**。

<br/>

普通IO:
{% asset_img clipboard1.png %}

直接面向的就是传输的数据的管道中的数据的流动。

**单向的**。

<br/>

NIO：

**NIO的通道也可以理解为传输数据的流，但是不可以看成水流，可以看成铁路，铁路要完成运输，必须要依靠火车，所以这个通道只是一个从源地点到目的地点的连接。通道本身没有任何的数据，要想传输数据，必须依靠缓冲区，缓冲区就可以理解为火车，一个地点上车，装满之后，到一个地点下车，所以它可以是双向的**。

<br/>

#### 3.通道（ Channel）与缓冲区（ Buffer）

Java NIO系统的核心在于：通道(Channel)和缓冲区(Buffer)。通道表示打开到 IO 设备(例如：文件、

套接字)的连接。若需要使用 NIO 系统，需要获取用于连接 IO 设备的通道以及用于容纳数据的缓冲

区。然后操作缓冲区，对数据进行处理。

简而言之， **Channel 负责传输， Buffer 负责存储**

<br/>

<br/>

### 缓冲区（ Buffer）

缓冲区（ Buffer） ：一个用于特定基本数据类型的容器。由 java.nio 包定义的，所有缓冲区

都是 Buffer 抽象类的子类

<br/>

Java NIO 中的 Buffer 主要用于与 NIO 通道进行交互，数据是从通道读入缓冲区，从缓冲区写

入通道中的。

<br/>

Buffer 就像一个数组，可以保存多个相同类型的数据。根据数据类型不同(boolean 除外) ，有以下 Buffer 常用子类：

~~~java
ByteBuffer
CharBuffer
ShortBuffer
IntBuffer
LongBuffer
FloatBuffer
DoubleBuffer
~~~

上述 Buffer 类 他们都采用相似的方法进行管理数据，只是各自管理的数据类型不同而已。都是通过如下方法获取一个 Buffer对象：

~~~java
static XxxBuffer allocate(int capacity) //创建一个容量为 capacity 的 XxxBuffer 对象
~~~

<br/>

#### 1.缓冲区的基本属性

**容量** (capacity) ： 表示 Buffer 最大数据容量，缓冲区容量不能为负，并且创建后不能更改。

**限制 **(limit)： 第一个不应该读取或写入的数据的索引，即位于 limit 后的数据不可读写。缓冲区的限制不能为负，并且不能大于其容量。

**位置** (position)： 下一个要读取或写入的数据的索引。缓冲区的位置不能为负，并且不能大于其限制

**标记** (mark)与**重置** (reset)： 标记是一个索引，通过 Buffer 中的 mark() 方法指定 Buffer 中一个特定的 position，之后可以通过调用 reset() 方法恢复到这个 position.

标记、 位置、 限制、 容量遵守以下不变式： 0 <= mark <= position <= limit <= capacity

{% asset_img clipboard2.png %}

<br/>

#### 2.Buffer 的常用方法

{% asset_img clipboard3.png %}

<br/>

#### 3.缓冲区的数据操作

获取 Buffer 中的数据：Buffer 所有子类提供了两个用于数据操作的方法： get()与 put() 方法：

~~~java
get() //读取单个字节
get(byte[] dst)//批量读取多个字节到 dst 中
get(int index)//读取指定索引位置的字节(不会移动 position)
~~~

放入数据到 Buffer 中

~~~java
put(byte b)//将给定单个字节写入缓冲区的当前位置
put(byte[] src)//将 src 中的字节写入缓冲区的当前位置
put(int index, byte b)//将指定字节写入缓冲区的索引位置(不会移动 position)
~~~

<br/>

#### 4.直接与非直接缓冲区

​	字节缓冲区要么是直接的，要么是非直接的。如果为直接字节缓冲区，则 Java 虚拟机会尽最大努力直接在此缓冲区上执行本机 I/O 操作。也就是说，在每次调用基础操作系统的一个本机 I/O 操作之前（或之后），虚拟机都会尽量避免将缓冲区的内容复制到中间缓冲区中（或从中间缓冲区中复制内容）。

​	 直接字节缓冲区可以通过调用此类的 allocateDirect() 工厂方法来创建。此方法返回的缓冲区进行分配和取消分配所需成本通常高于非直接缓冲区。直接缓冲区的内容可以驻留在常规的垃圾回收堆之外，因此，它们对应用程序的内存需求量造成的影响可能并不明显。所以，建议将直接缓冲区主要分配给那些易受基础系统的本机 I/O 操作影响的大型、持久的缓冲区。一般情况下，最好仅在直接缓冲区能在程序性能方面带来明显好处时分配它们。

​	直接字节缓冲区还可以通过 FileChannel 的 map() 方法 将文件区域直接映射到内存中来创建。该方法返回MappedByteBuffer 。 Java 平台的实现有助于通过 JNI 从本机代码创建直接字节缓冲区。如果以上这些缓冲区中的某个缓冲区实例指的是不可访问的内存区域，则试图访问该区域不会更改该缓冲区的内容，并且将会在访问期间或稍后的某个时间导致抛出不确定的异常。

​	字节缓冲区是直接缓冲区还是非直接缓冲区可通过调用其 isDirect() 方法来确定。提供此方法是为了能够在性能关键型代码中执行显式缓冲区管理。

**非直接缓冲区**

{% asset_img clipboard4.png %}

中间的copy过程完全是在浪费资源。

<br/>

**直接缓冲区**

{% asset_img clipboard5.png %}

不安全

直接在物理内存中开辟一块空间比在用户地址空间中开辟一块空间资源的消耗大得多。

当我们把数据写到物理内存中的时候，我们已将不能再管理这个数据， 数据什么时候能到物理磁盘完全由操作系统控制。

销毁将完全由垃圾回收机制控制。

<br/>

**代码**

~~~~java
public class TestBuffer {

  @Test
  public void test3(){
    //分配直接缓冲区
    ByteBuffer buf = ByteBuffer.allocateDirect(1024);

    System.out.println(buf.isDirect());
  }

  @Test
  public void test2(){
    String str = "abcde";

    ByteBuffer buf = ByteBuffer.allocate(1024);

    buf.put(str.getBytes());

    buf.flip();

    byte[] dst = new byte[buf.limit()];
    buf.get(dst, 0, 2);
    System.out.println(new String(dst, 0, 2));
    System.out.println(buf.position());

    //mark() : 标记
    buf.mark();

    buf.get(dst, 2, 2);
    System.out.println(new String(dst, 2, 2));
    System.out.println(buf.position());

    //reset() : 恢复到 mark 的位置
    buf.reset();
    System.out.println(buf.position());

    //判断缓冲区中是否还有剩余数据
    if(buf.hasRemaining()){

      //获取缓冲区中可以操作的数量
      System.out.println(buf.remaining());
    }
  }

  @Test
  public void test1(){
    String str = "abcde";

    //1. 分配一个指定大小的缓冲区
    ByteBuffer buf = ByteBuffer.allocate(1024);

    System.out.println("-----------------allocate()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //2. 利用 put() 存入数据到缓冲区中
    buf.put(str.getBytes());

    System.out.println("-----------------put()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //3. 切换读取数据模式
    buf.flip();

    System.out.println("-----------------flip()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //4. 利用 get() 读取缓冲区中的数据
    byte[] dst = new byte[buf.limit()];
    buf.get(dst);
    System.out.println(new String(dst, 0, dst.length));

    System.out.println("-----------------get()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //5. rewind() : 可重复读
    buf.rewind();

    System.out.println("-----------------rewind()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //6. clear() : 清空缓冲区. 但是缓冲区中的数据依然存在，但是处于“被遗忘”状态
    buf.clear();

    System.out.println("-----------------clear()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    System.out.println((char)buf.get());	
  }
}
~~~~

<br/>

### 通道

#### 1.底层区别

通道（ Channel）：由 java.nio.channels 包定义的。 Channel 表示 IO 源与目标打开的连接。

Channel 类似于传统的“流”。只不过 Channel本身不能直接访问数据， Channel 只能与Buffer 进行交互。

{% asset_img clipboard6.png %}

原始的计算机，让CPU来控制进行的IO的读写，浪费了CPU的资源。

{% asset_img clipboard7.png %}

DMA：直接访问存储器存储。CPU对DMA控制器进行初始化之后，后续的数据传输工作，完全由DMA控制器管理。降低了CPU的负担，使得CPU的利用率提高。

传统的IO流数据传输就是这种方式。

问题：当一些大型的应用程序进行大量的数据传输的时候，DMA可能会造成总线冲突。最终也会影响系统的资源和性能。

{% asset_img clipboard8.png %}

通道：一个完全独立的处理器。专门用于IO操作的。附属于CPU，有自己的处理命令和传输方式。

面对大型的应用程序进行大量的数据传输的时候，性能会略高，因为CPU的利用率高。

<br/>

#### 2.主要实现

- FileChannel：用于读取、写入、映射和操作文件的通道。
- DatagramChannel：通过 UDP 读写网络中的数据通道。
- SocketChannel：通过 TCP 读写网络中的数据。
- ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel。

<br/>

#### 3.获取通道

获取通道的一种方式是对支持通道的对象调用getChannel() 方法。支持通道的类如下：

- FileInputStream
- FileOutputStream
- RandomAccessFile
- DatagramSocket
- Socket
- ServerSocket

获取通道的其他方式是使用 Files 类的静态方法 newByteChannel() 获取字节通道。或者通过通道的静态方法 open() 打开并返回指定通道。

<br/>

#### 4.通道的数据传输

将 Buffer 中数据写入 Channel例如：

~~~java
int bytesWritten = inChannel.write(buf);
~~~

从 Channel 读取数据到 Buffer 例如：

~~~~java
int bytesRead = inChannel.read(buf);
~~~~

<br/>

#### 5.分散(Scatter)和聚集(Gather)

分散读取（ Scattering Reads）是指从 Channel 中读取的数据“分散” 到多个 Buffer 中。

{% asset_img clipboard9.png %}

注意：按照缓冲区的顺序，从 Channel 中读取的数据依次将 Buffer 填满。

聚集写入（ Gathering Writes）是指将多个 Buffer 中的数据“聚集”到 Channel。

<br/>
{% asset_img clipboard10.png %}

注意：按照缓冲区的顺序，写入 position 和 limit 之间的数据到 Channel 。

<br/>

#### 6. transferFrom()

将数据从源通道传输到其他 Channel 中：

~~~java
RandomAccessFile fromFile = new RandomAccessFile("data/fromFile.txt","rw");
//获取FileChannel
FileChannel fromChannel = fromFile.getChannel();

RandomAccessFile toFile = new RandomAccessFile("data/toFile.txt","rw");
FileChannel toChannel = toFile.getChannel();

//定义传输位置
long position = 0L;

//最多传输的字节数
long count = fromChannel.size();

//将数据从源通道传输到另一个通道
toChannel.transferFrom(fromChannel,count,position);
~~~

<br/>

#### 7.transferTo()

~~~java
RandomAccessFile fromFile = new RandomAccessFile("data/fromFile.txt","rw");
//获取FileChannel
FileChannel fromChannel = fromFile.getChannel();

RandomAccessFile toFile = new RandomAccessFile("data/toFile.txt","rw");
FileChannel toChannel = toFile.getChannel();

//定义传输位置
long position = 0L;

//最多传输的字节数
long count = fromChannel.size();

//将数据从源通道传输到另一个通道
fromChannel.transferTo(fromChannel,count,position);
~~~

<br/>

#### 8.FileChannel 的常用方法

{% asset_img clipboard11.png %}

<br/>

#### 9.  编解码测试代码

- 编码：字符串 -> 字节数组
- 解码：字节数组 -> 字符串

~~~~java
public class TestChannel {

  //字符集
  @Test
  public void test6() throws IOException{
    Charset cs1 = Charset.forName("GBK");

    //获取编码器
    CharsetEncoder ce = cs1.newEncoder();

    //获取解码器
    CharsetDecoder cd = cs1.newDecoder();

    CharBuffer cBuf = CharBuffer.allocate(1024);
    cBuf.put("尚硅谷威武！");
    cBuf.flip();

    //编码
    ByteBuffer bBuf = ce.encode(cBuf);

    for (int i = 0; i < 12; i++) {
      System.out.println(bBuf.get());
    }

    //解码
    bBuf.flip();
    CharBuffer cBuf2 = cd.decode(bBuf);
    System.out.println(cBuf2.toString());

    System.out.println("------------------------------------------------------");

    Charset cs2 = Charset.forName("GBK");
    bBuf.flip();
    CharBuffer cBuf3 = cs2.decode(bBuf);
    System.out.println(cBuf3.toString());
  }

  @Test
  public void test5(){
    Map<String, Charset> map = Charset.availableCharsets();

    Set<Entry<String, Charset>> set = map.entrySet();

    for (Entry<String, Charset> entry : set) {
      System.out.println(entry.getKey() + "=" + entry.getValue());
    }
  }

  //分散和聚集
  @Test
  public void test4() throws IOException{
    RandomAccessFile raf1 = new RandomAccessFile("1.txt", "rw");

    //1. 获取通道
    FileChannel channel1 = raf1.getChannel();

    //2. 分配指定大小的缓冲区
    ByteBuffer buf1 = ByteBuffer.allocate(100);
    ByteBuffer buf2 = ByteBuffer.allocate(1024);

    //3. 分散读取
    ByteBuffer[] bufs = {buf1, buf2};
    channel1.read(bufs);

    for (ByteBuffer byteBuffer : bufs) {
      byteBuffer.flip();
    }

    System.out.println(new String(bufs[0].array(), 0, bufs[0].limit()));
    System.out.println("-----------------");
    System.out.println(new String(bufs[1].array(), 0, bufs[1].limit()));

    //4. 聚集写入
    RandomAccessFile raf2 = new RandomAccessFile("2.txt", "rw");
    FileChannel channel2 = raf2.getChannel();

    channel2.write(bufs);
  }

  //通道之间的数据传输(直接缓冲区)
  @Test
  public void test3() throws IOException{
    FileChannel inChannel = FileChannel.open(Paths.get("d:/1.mkv"), StandardOpenOption.READ);
    FileChannel outChannel = FileChannel.open(Paths.get("d:/2.mkv"), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);

    //		inChannel.transferTo(0, inChannel.size(), outChannel);
    outChannel.transferFrom(inChannel, 0, inChannel.size());

    inChannel.close();
    outChannel.close();
  }

  //使用直接缓冲区完成文件的复制(内存映射文件)
  @Test
  public void test2() throws IOException{//2127-1902-1777
    long start = System.currentTimeMillis();

    FileChannel inChannel = FileChannel.open(Paths.get("d:/1.mkv"), StandardOpenOption.READ);
    FileChannel outChannel = FileChannel.open(Paths.get("d:/2.mkv"), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);

    //内存映射文件
    MappedByteBuffer inMappedBuf = inChannel.map(MapMode.READ_ONLY, 0, inChannel.size());
    MappedByteBuffer outMappedBuf = outChannel.map(MapMode.READ_WRITE, 0, inChannel.size());

    //直接对缓冲区进行数据的读写操作
    byte[] dst = new byte[inMappedBuf.limit()];
    inMappedBuf.get(dst);
    outMappedBuf.put(dst);

    inChannel.close();
    outChannel.close();

    long end = System.currentTimeMillis();
    System.out.println("耗费时间为：" + (end - start));
  }

  //利用通道完成文件的复制（非直接缓冲区）
  @Test
  public void test1(){//10874-10953
    long start = System.currentTimeMillis();

    FileInputStream fis = null;
    FileOutputStream fos = null;
    //①获取通道
    FileChannel inChannel = null;
    FileChannel outChannel = null;
    try {
      fis = new FileInputStream("d:/1.mkv");
      fos = new FileOutputStream("d:/2.mkv");

      inChannel = fis.getChannel();
      outChannel = fos.getChannel();

      //②分配指定大小的缓冲区
      ByteBuffer buf = ByteBuffer.allocate(1024);

      //③将通道中的数据存入缓冲区中
      while(inChannel.read(buf) != -1){
        buf.flip(); //切换读取数据的模式
        //④将缓冲区中的数据写入通道中
        outChannel.write(buf);
        buf.clear(); //清空缓冲区
      }
    } catch (IOException e) {
      e.printStackTrace();
    } finally {
      if(outChannel != null){
        try {
          outChannel.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }

      if(inChannel != null){
        try {
          inChannel.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }

      if(fos != null){
        try {
          fos.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }

      if(fis != null){
        try {
          fis.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }

    long end = System.currentTimeMillis();
    System.out.println("耗费时间为：" + (end - start));

  }

}
~~~~

