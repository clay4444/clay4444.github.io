---
title: NIO系列之非阻塞式IO
categories:
  - 多线程编程
tags:
  - NIO
abbrlink: 84f5b3e
date: 2017-09-08 17:03:49
---

### 阻塞和非阻塞

​	传统的 IO 流都是阻塞式的。也就是说，当一个线程调用 read() 或 write()时，该线程被阻塞，直到有一些数据被读取或写入，该线程在此期间不能执行其他任务。因此，在完成网络通信进行 IO 操作时，由于线程会阻塞，所以服务器端必须为每个客户端都提供一个独立的线程进行处理，当服务器端需要处理大量客户端时，性能急剧下降。

{% asset_img clipboard.png %}

​	Java NIO 是非阻塞模式的。当线程从某通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。线程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出通道。因此， NIO 可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。

<br/>

### 选择器（ Selector）

选择器（ Selector） 是 SelectableChannle 对象的多路复用器， Selector 可以同时监控多个 SelectableChannel 的 IO 状况，也就是说，利用 Selector可使一个单独的线程管理多个 Channel。 **Selector 是非阻塞 IO 的核心**。

SelectableChannle 的结构如下图：

{% asset_img clipboard1.png %}

<br/>

#### 1. 选择器（ Selector）的应用

创建Selector：通过调用Selector.open() 方法创建一个Selector

```java
//4. 获取选择器
Selector selector = Selector.open();
```

向选择器注册通道，

~~~java
//创建一个Socket套接字
Socket socket = new Socket(InetAddress.getByName("127.0.0.1"),9898);

//获取SocketChannel
SocketChannel channel = socket.getChannel();

//创建选择器
Selector selector = Selector.open();

//将SocketChannel切换到非阻塞模式
channel.configureBlocking(false);

//向Selector注册channel
SelectionKey key = channel.register(selector,SelectionKey.OP_READ);
~~~

当调用 register(Selector sel, int ops) 将通道注册选择器时，选择器对通道的监听事件，需要通过第二个参数 ops 指定。

可以监听的事件类型（ 可使用 SelectionKey 的四个常量表示）：

- 读 : SelectionKey.OP_READ （ 1）
- 写 : SelectionKey.OP_WRITE （ 4）
- 连接 : SelectionKey.OP_CONNECT （ 8）
- 接收 : SelectionKey.OP_ACCEPT （ 16）

若注册时不止监听一个事件，则可以使用“位或”操作符连接。

```JAVA
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

<br/>

#### 2.SelectionKey

SelectionKey： 表示 SelectableChannel 和 Selector 之间的注册关系。每次向选择器注册通道时就会选择一个事件(选择键)。 选择键包含两个表示为整数值的操作集。操作集的每一位都表示该键的通道所支持的一类可选择操作。

{% asset_img clipboard2.png %}

<br/>

#### 3.Selector 的常用方法

{% asset_img clipboard3.png %}

<br/>

#### 4. SocketChannel

Java NIO中的SocketChannel是一个连接到TCP网络套接字的通道。

操作步骤：

- 打开 SocketChannel
- 读写数据
- 关闭 SocketChannel

Java NIO中的 ServerSocketChannel 是一个可以监听新进来的TCP连接的通道，就像标准IO中的ServerSocket一样。

<br/>

#### 5.DatagramChannel

Java NIO中的DatagramChannel是一个能收发UDP包的通道。

操作步骤：

- 打开 DatagramChannel
- 接收/发送数据

<br/>

<br/>

### 管道 (Pipe)

Java NIO 管道是2个线程之间的单向数据连接。Pipe有一个source通道和一个sink通道。数据会被写到sink通道，从source通道读取。

{% asset_img clipboard4.png %}

<br/>

#### 1.向管道写数据

~~~java
String str = "测试数据";

//创建管道
Pipe pipe = Pipe.open();

//向管道写输入
Pipe.SinkChannel sinkChannel = pipe.sink();

//通道 SinkChannel 的write() 方法写数据
ByteBuffer buf = ByteBuffer.allocate(1024);
buf.clear();
buf.put(str.getBytes());
buf.flip();

while(buf.hasRemaining()){
	sinkChannel.write(buf);
}
~~~

<br/>

#### 2.从管道读取数据

~~~java
//从管道读取数据
Pipe.SourceChannel sourceChannel = pipe.source();

//调用SourceChannel的read()方法读取数据
ByteBuffer buf = ByteBuffer.allocate(1024);
sourceChannel.read(buf);
~~~



### 使用 NIO 完成网络通信的三个核心：

1. 通道（Channel）：负责连接

- java.nio.channels.Channel 接口：
- |--SelectableChannel
- |----SocketChannel
- |----ServerSocketChannel
- |----DatagramChannel
- |----Pipe.SinkChannel
- |----Pipe.SourceChannel

1. 缓冲区（Buffer）：负责数据的存取
2. 选择器（Selector）：是 SelectableChannel 的多路复用器。用于监控 SelectableChannel 的 IO 状况

#### 1.TCP

~~~java
public class TestNonBlockingNIO {

  //客户端
  @Test
  public void client() throws IOException{
    //1. 获取通道
    SocketChannel sChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 9898));

    //2. 切换非阻塞模式
    sChannel.configureBlocking(false);

    //3. 分配指定大小的缓冲区
    ByteBuffer buf = ByteBuffer.allocate(1024);

    //4. 发送数据给服务端
    Scanner scan = new Scanner(System.in);

    while(scan.hasNext()){
      String str = scan.next();
      buf.put((new Date().toString() + "\n" + str).getBytes());
      buf.flip();
      sChannel.write(buf);
      buf.clear();
    }

    //5. 关闭通道
    sChannel.close();
  }

  //服务端
  @Test
  public void server() throws IOException{
    //1. 获取通道
    ServerSocketChannel ssChannel = ServerSocketChannel.open();

    //2. 切换非阻塞模式
    ssChannel.configureBlocking(false);

    //3. 绑定连接
    ssChannel.bind(new InetSocketAddress(9898));

    //4. 获取选择器
    Selector selector = Selector.open();

    //5. 将通道注册到选择器上, 并且指定“监听接收事件”
    ssChannel.register(selector, SelectionKey.OP_ACCEPT);

    //6. 轮询式的获取选择器上已经“准备就绪”的事件
    while(selector.select() > 0){

      //7. 获取当前选择器中所有注册的“选择键(已就绪的监听事件)”
      Iterator<SelectionKey> it = selector.selectedKeys().iterator();

      while(it.hasNext()){
        //8. 获取准备“就绪”的是事件
        SelectionKey sk = it.next();

        //9. 判断具体是什么事件准备就绪
        if(sk.isAcceptable()){
          //10. 若“接收就绪”，获取客户端连接
          SocketChannel sChannel = ssChannel.accept();

          //11. 切换非阻塞模式
          sChannel.configureBlocking(false);

          //12. 将该通道注册到选择器上
          sChannel.register(selector, SelectionKey.OP_READ);
        }else if(sk.isReadable()){
          //13. 获取当前选择器上“读就绪”状态的通道
          SocketChannel sChannel = (SocketChannel) sk.channel();

          //14. 读取数据
          ByteBuffer buf = ByteBuffer.allocate(1024);

          int len = 0;
          while((len = sChannel.read(buf)) > 0 ){
            buf.flip();
            System.out.println(new String(buf.array(), 0, len));
            buf.clear();
          }
        }

        //15. 取消选择键 SelectionKey
        it.remove();
      }
    }
  }
}
~~~

#### 2.UDP

~~~java
public class TestNonBlockingNIO2 {

  @Test
  public void send() throws IOException{
    DatagramChannel dc = DatagramChannel.open();

    dc.configureBlocking(false);

    ByteBuffer buf = ByteBuffer.allocate(1024);

    Scanner scan = new Scanner(System.in);

    while(scan.hasNext()){
      String str = scan.next();
      buf.put((new Date().toString() + ":\n" + str).getBytes());
      buf.flip();
      dc.send(buf, new InetSocketAddress("127.0.0.1", 9898));
      buf.clear();
    }

    dc.close();
  }

  @Test
  public void receive() throws IOException{
    DatagramChannel dc = DatagramChannel.open();

    dc.configureBlocking(false);

    dc.bind(new InetSocketAddress(9898));

    Selector selector = Selector.open();

    dc.register(selector, SelectionKey.OP_READ);

    while(selector.select() > 0){
      Iterator<SelectionKey> it = selector.selectedKeys().iterator();

      while(it.hasNext()){
        SelectionKey sk = it.next();

        if(sk.isReadable()){
          ByteBuffer buf = ByteBuffer.allocate(1024);

          dc.receive(buf);
          buf.flip();
          System.out.println(new String(buf.array(), 0, buf.limit()));
          buf.clear();
        }
      }

      it.remove();
    }
  }

}
~~~

