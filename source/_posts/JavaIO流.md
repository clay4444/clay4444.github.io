---
title: JavaIO流
tags:
  - JavaIO流系列
abbrlink: c51233f6
date: 2017-11-09 11:18:18
---

## 1：概述

## 1.1 输入和输出 – 数据源和目标媒介

**往里读往外写 **

**输入流可以理解为向内存输入，输出流可以理解为从内存输出**

​	Java的IO包主要关注的是从原始数据源的读取以及输出原始数据到目标媒介。以下是最典型的数据源和目标媒介：

<br/>

- 文件
- 管道
- 网络连接
- 内存缓存
- System.in, System.out, System.error(注：Java标准输入、输出、错误输出)

### 1.2 流

​	在Java IO中，流是一个核心的概念。流从概念上来说是一个连续的数据流。你既可以从流中读取数据，也可以往流中写数据。流与数据源或者数据流向的媒介相关联。在Java IO中流既可以是字节流(以字节为单位进行读写)，也可以是字符流(以字符为单位进行读写)。

### 1.3 类InputStream, OutputStream, Reader 和Writer

一个程序需要InputStream或者Reader从数据源读取数据，需要OutputStream或者Writer将数据写入到目标媒介中。以下的图说明了这一点：

{% asset_img 无标题1.png %}

InputStream和Reader与数据源相关联，OutputStream和writer与目标媒介相关联。

### 1.4 Java IO的用途和特征

- 文件访问
- 网络访问
- 内存缓存访问
- 线程内部通信(管道)
- 缓冲
- 过滤
- 解析
- 读写文本 (Readers / Writers)
- 读写基本类型数据 (long, int etc.)
- 读写对象

<br>

<br/>

## 2：文件

### 2.1 **通过 **Java IO读文件

​	如果你需要在不同端之间读取文件，你可以根据该文件是二进制文件还是文本文件来选择使用FileInputStream或者FileReader。这两个类允许你从文件开始到文件末尾一次读取一个字节或者字符，或者将读取到的字节写入到字节数组或者字符数组。你不必一次性读取整个文件，相反你可以按顺序地读取文件中的字节和字符。

​	如果你需要跳跃式地读取文件其中的某些部分，可以使用RandomAccessFile。

### 2.2 **通过**Java IO写文件

​	如果你需要在不同端之间进行文件的写入，你可以根据你要写入的数据是二进制型数据还是字符型数据选用FileOutputStream或者FileWriter。你可以一次写入一个字节或者字符到文件中，也可以直接写入一个字节数组或者字符数据。数据按照写入的顺序存储在文件当中。

### 2.3 **通过**Java IO随机存取文件

​	正如我所提到的，你可以通过RandomAccessFile对文件进行随机存取。

​	随机存取并不意味着你可以在真正随机的位置进行读写操作，它只是意味着你可以跳过文件中某些部分进行操作，并且支持同时读写，不要求特定的存取顺序。这使得RandomAccessFile可以覆盖一个文件的某些部分、或者追加内容到它的末尾、或者删除它的某些内容，当然它也可以从文件的任何位置开始读取文件。

### 2.4 **文件和目录信息的获取**

​	有时候你可能需要读取文件的信息而不是文件的内容，举个例子，如果你需要知道文件的大小和文件的属性。对于目录来说也是一样的，比如你需要获取某个目录下的文件列表。通过File类可以获取文件和目录的信息。

<br>

<br/>

## 3：管道

- Java IO中的管道为运行在同一个JVM中的两个线程提供了通信的能力。所以管道也可以作为数据源以及目标媒介。
- 你不能利用管道与不同的JVM中的线程通信(不同的进程)。在概念上，Java的管道不同于Unix/Linux系统中的管道。在Unix/Linux中，运行在不同地址空间的两个进程可以通过管道通信。在Java中，通信的双方应该是运行在同一进程中的不同线程。

<br/>

### 3.1 通过Java IO创建管道

​	可以通过Java IO中的PipedOutputStream和PipedInputStream创建管道。一个PipedInputStream流应该和一个PipedOutputStream流相关联。一个线程通过PipedOutputStream写入的数据可以被另一个线程通过相关联的PipedInputStream读取出来。

### 3.1 Java IO管道示例

这是一个如何将PipedInputStream和PipedOutputStream关联起来的简单例子：

~~~java
import java.io.IOException;
import java.io.PipedInputStream;
import java.io.PipedOutputStream;

public class PipeExample {

  public static void main(String[] args) throws Exception{

    final PipedOutputStream output = new PipedOutputStream();
    final PipedInputStream input = new PipedInputStream(output);

    Thread thread1 = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          output.write("Hello World pipe".getBytes());
        }catch (IOException e){
          System.out.println(e);
        }
      }
    });

    Thread thread2 = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          int data = input.read();
          while(data != -1){
            System.out.println((char)data);
            data = input.read();
          }
        }catch (IOException e){
          System.out.println(e);
        }
      }
    });

    thread1.start();
    thread2.start();
  }
}
~~~

注：本例忽略了流的关闭。请在处理流的过程中，务必保证关闭流，或者使用jdk7引入的try-resources代替显示地调用close方法的方式。

​	你也可以使用两个管道共有的connect()方法使之相关联。PipedInputStream和PipedOutputStream都拥有一个可以互相关联的connect()方法。

### 3.2 管道和线程

​	请记得，当使用两个相关联的管道流时，务必将它们分配给不同的线程。read()方法和write()方法调用时会导致流阻塞，这意味着如果你尝试在一个线程中同时进行读和写，可能会导致线程死锁。

### 3.3 管道的替代

​	除了管道之外，一个JVM中不同线程之间还有许多通信的方式。实际上，线程在大多数情况下会传递完整的对象信息而非原始的字节数据。但是，如果你需要在线程之间传递字节数据，Java IO的管道是一个不错的选择。

<br>

<br/>

## 4：网络

​	当两个进程之间建立了网络连接之后，他们通信的方式如同操作文件一样：利用InputStream读取数据，利用OutputStream写入数据。换句话来说，**Java网络API用来在不同进程之间建立网络连接，而Java IO则用来在建立了连接之后的进程之间交换数据**

<br/>

​	基本上意味着如果你有一份能够对文件进行写入某些数据的代码，那么这些数据也可以很容易地写入到网络连接中去。你所需要做的仅仅只是在代码中利用OutputStream替代FileOutputStream进行数据的写入。因为FileInputStream是InputStream的子类，所以这么做并没有什么问题。

<br/><br/>

## 5. 流

### 5.1  **InputStream**

​	java.io.InputStream类是所有Java IO输入流的基类。如果你正在开发一个从流中读取数据的组件，请尝试用InputStream替代任何它的子类(比如FileInputStream)进行开发。这么做能够让你的代码兼容任何类型而非某种确定类型的输入流。

<br/>

​	然而仅仅依靠InputStream并不总是可行。如果你需要将读过的数据推回到流中，你必须使用PushbackInputStream，这意味着你的流变量只能是这个类型，否则在代码中就不能调用PushbackInputStream的unread()方法。

<br/>

​	通常使用输入流中的read()方法读取数据。read()方法返回一个整数，代表了读取到的字节的内容(译者注：0 ~ 255)。当达到流末尾没有更多数据可以读取的时候，read()方法返回-1。

<br/>

这是一个简单的示例：

~~~java
InputStream input = new FileInputStream("c:\\data\\input-file.txt");
int data = input.read(); 
while(data != -1){
  data = input.read();
}
~~~

### 5.2  **OutputStream**

​	java.io.OutputStream是Java IO中所有输出流的基类。如果你正在开发一个能够将数据写入流中的组件，请尝试使用OutputStream替代它的所有子类。

<br/>

这是一个简单的示例：

~~~java
OutputStream output = new FileOutputStream("c:\\data\\output-file.txt");
output.write("Hello World".getBytes());
output.close();
~~~

### 5.3  **组合流**

​	你可以将流整合起来以便实现更高级的输入和输出操作。比如，一次读取一个字节是很慢的，所以可以从磁盘中一次读取一大块数据，然后从读到的数据块中获取字节。为了实现缓冲，可以把InputStream包装到BufferedInputStream中。代码示例：

~~~java
InputStream input = new BufferedInputStream(new FileInputStream("c:\\data\\input-file.txt"));
~~~

​	缓冲同样可以应用到OutputStream中。你可以实现将大块数据批量地写入到磁盘(或者相应的流)中，这个功能由BufferedOutputStream实现。

<br/>

​	缓冲只是通过流整合实现的其中一个效果。你可以把InputStream包装到PushbackInputStream中，之后可以将读取过的数据推回到流中重新读取，在解析过程中有时候这样做很方便。或者，你可以将两个InputStream整合成一个[SequenceInputStream](http://tutorials.jenkov.com/java-io/sequenceinputstream.html)。

<br/>

​	将不同的流整合到一个链中，可以实现更多种高级操作。通过编写包装了标准流的类，可以实现你想要的效果和过滤器。

<br/><br/>

##  6. Reader And Writer

### 6.1 **Reader**

​	Reader类是Java IO中所有Reader的基类。子类包括BufferedReader，PushbackReader，InputStreamReader，StringReader和其他Reader。

<br/>

这是一个简单的Java IO Reader的例子：

~~~java
Reader reader = new FileReader("c:\\data\\myfile.txt");
int data = reader.read();
while(data != -1){
  char dataChar = (char) data;
  data = reader.read();
}
~~~

​	请注意，InputStream的read()方法返回一个字节，意味着这个返回值的范围在0到255之间(当达到流末尾时，返回-1)，Reader的read()方法返回一个字符，意味着这个返回值的范围在0到65535之间(当达到流末尾时，同样返回-1)。这并不意味着Reade只会从数据源中一次读取2个字节，Reader会根据文本的编码，一次读取一个或者多个字节。

<br/>

​	你通常会使用Reader的子类，而不会直接使用Reader。Reader的子类包括InputStreamReader，CharArrayReader，FileReader等等。可以查看[Java IO概述](http://ifeve.com/java-io-3/)浏览完整的Reader表格。

### 6.2  **整合Reader与InputStream**

​	一个Reader可以和一个InputStream相结合。如果你有一个InputStream输入流，并且想从其中读取字符，可以把这个InputStream包装到InputStreamReader中。把InputStream传递到InputStreamReader的构造函数中：

~~~java
Reader reader = new InputStreamReader(inputStream);
~~~

在构造函数中可以指定解码方式。更多内容请参阅[InputStreamReader](http://tutorials.jenkov.com/java-io/inputstreamreader.html)。

### 6.3 **Writer**

​	Writer类是Java IO中所有Writer的基类。子类包括BufferedWriter和PrintWriter等等。这是一个Java IO Writer的例子：

~~~java
Writer writer = new FileWriter("c:\\data\\file-output.txt"); 
writer.write("Hello World Writer"); 
writer.close();
~~~

​	同样，你最好使用Writer的子类，不需要直接使用Writer，因为子类的实现更加明确，更能表现你的意图。常用子类包括OutputStreamWriter，CharArrayWriter，FileWriter等。Writer的write(int c)方法，会将传入参数的低16位写入到Writer中，忽略高16位的数据。

### 6.4  **整合Writer和OutputStream**

​	与Reader和InputStream类似，一个Writer可以和一个OutputStream相结合。把OutputStream包装到OutputStreamWriter中，所有写入到OutputStreamWriter的字符都将会传递给OutputStream。这是一个OutputStreamWriter的例子：

~~~java
Writer writer = new OutputStreamWriter(outputStream);
~~~

### 6.5 **整合Reader和Writer**

​	和字节流一样，Reader和Writer可以相互结合实现更多更有趣的IO，工作原理和把Reader与InputStream或者Writer与OutputStream相结合类似。举个栗子，可以通过将Reader包装到BufferedReader、Writer包装到BufferedWriter中实现缓冲。以下是例子：

~~~java
Reader reader = new BufferedReader(new FileReader(...));
Writer writer = new BufferedWriter(new FileWriter(...));
~~~

<br/><br/>

## 7. 异常处理

​	流与Reader和Writer在结束使用的时候，需要正确地关闭它们。通过调用close()方法可以达到这一点。不过这需要一些思考。请看下边的代码：

~~~java
InputStream input = new FileInputStream("c:\\data\\input-text.txt");
int data = input.read();
while(data != -1) {
  //do something with data...  
  doSomethingWithData(data);
  data = input.read();
}
input.close();
~~~

​	第一眼看这段代码时，可能觉得没什么问题。可是如果在调用doSomethingWithData()方法时出现了异常，会发生什么呢？没错，这个InputStream对象就不会被关闭。

<br/>

​	为了避免异常造成流无法被关闭，我们可以把代码重写成这样:

~~~java
InputStream input = null;
try{
  input = new FileInputStream("c:\\data\\input-text.txt");
  int data = input.read();
  while(data != -1) {
    //do something with data...  
    doSomethingWithData(data);
    data = input.read();
  }
}catch(IOException e){
  //do something with e... log, perhaps rethrow etc.
} finally {
  if(input != null)
    input.close();
}
~~~

​	注意到这里把InputStream的关闭代码放到了finally块中，无论在try-catch块中发生了什么，finally内的代码始终会被执行，所以这个InputStream总是会被关闭。

<br/>

​	但是如果close()方法抛出了异常，告诉你流已经被关闭过了呢？为了解决这个难题，你也需要把close()方法写在try-catch内部，就像这样：

~~~java
} finally {
  try{
    if(input != null)
      input.close();
  } catch(IOException e){
    //do something, or ignore.
  }
}
~~~

​	这段解决了InputStream(或者OutputStream)流关闭的问题的代码，确实是有一些不优雅，尽管能够正确处理异常。如果你的代码中重复地遍布了这段丑陋的异常处理代码，这不是很好的一个解决方案。如果一个匆忙的家伙贪图方便忽略了异常处理呢？

<br/>

​	此外，想象一下某个异常最先从doSomethingWithData方法内抛出。第一个catch会捕获到异常，然后在finally里程序会尝试关闭InputStream。但是如果还有异常从close()方法内抛出呢？这两个异常中得哪个异常应当往调用栈上传播呢？

<br/>

​	幸运的是，有一个办法能够解决这个问题。这个解决方案称作“异常处理模板”。创建一个正确关闭流的模板，能够在代码中做到一次编写，重复使用，既优雅又简单。

### 7.1 Java7中IO的异常处理

​	从Java7开始，一种新的被称作“try-with-resource”的异常处理机制被引入进来。这种机制旨在解决针对InputStream和OutputStream这类在使用完毕之后需要关闭的资源的异常处理。

​	详细请看本人的另外一篇博客：详解Java7中的Try-with-resource

<br/><br/>

## 8. InputStream

​	InputStream类是Java IO API中所有输入流的基类。InputStream子类包括FileInputStream，BufferedInputStream，PushbackInputStream等等

### 8.1 **Java InputStream例子** 

InputStream用于读取基于字节的数据，一次读取一个字节，这是一个InputStream的例子：

~~~java
InputStream inputstream = new FileInputStream("c:\\data\\input-text.txt");
int data = inputstream.read();
while(data != -1) {
  //do something with data...
  doSomethingWithData(data);   
  data = inputstream.read();
}
inputstream.close();
~~~

​	这个例子创建了FileInputStream实例。FileInputStream是InputStream的子类，所以可以把FileInputStream实例赋值给InputStream变量。

<br/>

从Java7开始，你可以使用“[try-with-resource](http://tutorials.jenkov.com/java-exception-handling/try-with-resources.html)”结构确保InputStream在结束使用之后关闭，

~~~java
try( InputStream inputstream = new FileInputStream("file.txt") ) {
  int data = inputstream.read();
  while(data != -1){
    System.out.print((char) data);
    data = inputstream.read();
  }
}
~~~

当执行线程退出try语句块的时候，InputStream变量会被关闭。

### 8.2 **read()**

read()方法返回从InputStream流内读取到的一个字节内容(0~255)，例子如下：

~~~java
int data = inputstream.read();
~~~

你可以把返回的int类型转化成char类型：

~~~java
char aChar = (char) data;
~~~

​	InputStream的子类可能会包含read()方法的替代方法。比如，DataInputStream允许你利用readBoolean()，readDouble()等方法读取Java基本类型变量int，long，float，double和boolean。

### 8.3 **流末尾**

​	如果read()方法返回-1，意味着程序已经读到了流的末尾，此时流内已经没有多余的数据可供读取了。-1是一个int类型，不是byte或者char类型，这是不一样的。

<br/>

​	当达到流末尾时，你就可以关闭流了。

### 8.4 **read(byte[])**

InputStream包含了2个从InputStream中读取数据并将数据存储到缓冲数组中的read()方法，他们分别是：

- int read(byte[])
- int read(byte, int offset, int length)

一次性读取一个字节数组的方式，比一次性读取一个字节的方式快的多，所以，尽可能使用这两个方法代替read()方法。

<br/>

read(byte[])方法会尝试读取与给定字节数组容量一样大的字节数，返回值说明了已经读取过的字节数。如果InputStream内可读的数据不足以填满字节数组，那么数组剩余的部分将包含本次读取之前的数据。记得检查有多少数据实际被写入到了字节数组中。

<br/>

read(byte, int offset, int length)方法同样将数据读取到字节数组中，不同的是，该方法从数组的offset位置开始，并且最多将length个字节写入到数组中。同样地，read(byte, int offset, int length)方法返回一个int变量，告诉你已经有多少字节已经被写入到字节数组中，所以请记得在读取数据前检查上一次调用read(byte, int offset, int length)的返回值。

<br/>

这两个方法都会在读取到达到流末尾时返回-1。

<br/>

这是一个使用InputStream的read(byte[])的例子：

~~~java
InputStream inputstream = new FileInputStream("c:\\data\\input-text.txt");
byte[] data = new byte[1024];
int bytesRead = inputstream.read(data);
while(bytesRead != -1) {
  doSomethingWithData(data, bytesRead);
  bytesRead = inputstream.read(data);
}
inputstream.close();
~~~

在代码中，首先创建了一个字节数组。然后声明一个叫做bytesRead的存储每次调用read(byte[])返回值的int变量，并且将第一次调用read(byte[])得到的返回值赋值给它。

<br/>

在while循环内部，把字节数组和已读取字节数作为参数传递给doSomethingWithData方法然后执行调用。在循环的末尾，再次将数据写入到字节数组中。

<br/>

你不需要想象出read(byte, int offset, int length)替代read(byte[])的场景，几乎可以在使用read(byte, int offset, int length)的任何地方使用read(byte[])。