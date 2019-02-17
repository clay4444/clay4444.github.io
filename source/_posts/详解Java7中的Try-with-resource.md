---
title: 详解Java7中的Try-with-resource
tags:
  - Java异常处理
abbrlink: a4b4dacb
date: 2017-10-31 17:21:38
---

### 声明

本文翻译自：http://tutorials.jenkov.com/java-exception-handling/try-with-resources.html

### 正文

​	在java 7中使用Try-with-resource是一种新的异常处理机制，可以更容易地正确关闭在try-catch块内使用的资源。

<br/>

​	这里是本文涵盖的主题列表：

#### 1. 古老的Try-Catch-Finally 资源管理方式

​	在java 7之前，管理需要明确关闭的资源的实现方式有些繁琐。

<br/>

​	看下面的方法读取文件并将其打印到system.out：

~~~java
private static void printFile() throws IOException {
  InputStream input = null;

  try {
    input = new FileInputStream("file.txt");//1

    int data = input.read();//2
    while(data != -1){
      System.out.print((char) data);
      data = input.read();
    }
  } finally {
    if(input != null){
      input.close();//3
    }
  }
}
~~~

​	用序号标记的代码是代码可以抛出异常的地方。正如你所看到的那样，这可能发生在try-block内的3个地方，在finally块内部有1个地方。

<br/>

​	无论是否从try块抛出异常，finally块总是被执行。也就是说，无论try块中发生了什么，inputstream都是会关闭的。或者，会试图关闭。如果关闭失败，inputstream的close（）方法可能会抛出一个异常。

<br/>

​	想象一下，在try块内部抛出一个异常。然后执行finally块。与此同时，从finally块中也抛出一个异常。你认为哪个异常会在调用堆栈中传播？

<br/>

​	即使从try块抛出的异常与我们的错误排查更相关，但是最后传播到调用堆栈中的却是从finally块抛出的异常。这严重带给了我们误导。

#### 2. Try-with-resources方式

​	在Java 7中，您可以使用try-with-resource结构编写上例中的代码：

~~~java
private static void printFileJava7() throws IOException {

  try(FileInputStream input = new FileInputStream("file.txt")) {

    int data = input.read();
    while(data != -1){
      System.out.print((char) data);
      data = input.read();
    }
  }
}
~~~

​	注意方法中的第一行：

```java
try(FileInputStream input = new FileInputStream("file.txt")) {
```

​	这就是Try-with-resources的结构。在try关键字后面的括号内声明fileinputstream变量。此外，文件输入流被实例化并被分配给该变量。

<br/>

​	当try块完成时，fileinputstream将被自动关闭。这是可以实现的，因为fileinputstream实现了java接口java.lang.autocloseable。所有实现这个接口的类都可以在try-with-resources结构中使用。

<br/>

​	如果从try-with-resources块内部抛出一个异常，并且关闭了fileinputstream（当调用close（）时），则try块内抛出的异常将被抛出到外部世界。fileinputstream关闭时引发的异常被抑制。这与本文第一个例子使用旧式的异常处理中发生的情况恰好相反。

#### 3. 使用多个资源的情况

​	您可以在try-with-resources块中使用多个资源，并将它们全部自动关闭。这里是一个例子：

~~~java
private static void printFileJava7() throws IOException {

  try(  FileInputStream     input         = new FileInputStream("file.txt");
      BufferedInputStream bufferedInput = new BufferedInputStream(input)
     ) {

    int data = bufferedInput.read();
    while(data != -1){
      System.out.print((char) data);
      data = bufferedInput.read();
    }
  }
}
~~~

​	此示例在try关键字后面的括号内创建了两个资源。一个文件输入流和一个缓冲输入流。这两个资源将在执行离开try块时自动关闭。

<br/>

​	资源将按照创建/排列在括号内的顺序相反的顺序关闭。首先bufferedinputstream将被关闭，然后fileinputstream。

#### 4. 自定义的可自动关闭的实现

​	try-with-resources构造不仅仅适用于java api的类。您也可以在自己的类中实现java.lang.autocloseable接口，并将其与try-with-resources结构一起使用。

<br/>

​	可自动切断的接口只有一个叫做close（）的方法。源码如下：

~~~java
public interface AutoClosable {
  public void close() throws Exception;
}
~~~

​	任何实现这个接口的类都可以和try-with-resources结构一起使用。这里是一个简单的示例实现：

~~~java
public class MyAutoClosable implements AutoCloseable {

  public void doIt() {
    System.out.println("MyAutoClosable doing it!");
  }

  @Override
  public void close() throws Exception {
    System.out.println("MyAutoClosable closed!");
  }
}
~~~

​	doit（）方法不是可自动切断的接口的一部分。它就在那里，因为我们希望能够做的不仅仅是关闭对象。

<br/>

​	这里是myautoclosable如何与try-with-resources结构一起使用的例子：

```java
private static void myAutoClosable() throws Exception {

  try(MyAutoClosable myAutoClosable = new MyAutoClosable()){
    myAutoClosable.doIt();
  }
}
```

​	这里是调用myautoclosable（）方法时输出到system.out的结果：

```java
MyAutoClosable doing it!
MyAutoClosable closed!
```

​	正如你所看到的，try-with-resources是确保在try-catch块内部使用的资源被正确关闭的一种非常强大的方式，不管这些资源是你自己的创建还是java的内置api。