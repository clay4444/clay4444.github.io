---
title: 深入探讨Java的故障保护异常处理
tags:
  - Java异常处理
abbrlink: 6bb61ef6
date: 2017-10-15 14:33:40
---

### 声明：

本文翻译自：http://tutorials.jenkov.com/java-exception-handling/fail-safe-exception-handling.html

### 正文：

对于异常处理代码的故障保护是势在必行的，其中最重要的原因就是：

<br/>

**在 try-catch-finally 块中引发的最后一个异常是将会在调用堆栈上传播的异常。所有较早的异常都将消失 **。

<br/>

如果程序执行过程中从catch或finally块中引发异常，则此异常可能会被隐藏。试图确定错误的原因时，会导致我们做出错误的判断。
<br/>

下面是非故障保护异常处理的经典示例：

```java
InputStream input = null;
try{
  input = new FileInputStream("myFile.txt");
  //do something with the stream

} catch(IOException e){
  throw new WrapperException(e);
} finally {
  try{
    input.close();
  } catch(IOException e){
    throw new WrapperException(e);
  }
}
```

​	如果fileinputstream构造函数抛出filenotfoundexception，你认为会发生什么？

<br/>

​	首先执行catch块。这个块只是重新抛出了wrapperexception包装的异常。

<br/>

​	第二个finally块将被执行，关闭输入流。但是，由于fileinputstream构造函数引发了filenotfoundexception，因此“输入”引用将为空。结果将是从finally块抛出的nullpointerexception。nullpointerexception不被finally块的catch（ioexception e）子句捕获，所以它被传播到调用堆栈中。从catch块中抛出的WrapperException 就会消失！

<br/>

​	处理这种情况的正确方法是检查在try块中分配的引用是否为null，然后调用它们上的任何方法。代码如下：

```java
InputStream input = null;
try{
  input = new FileInputStream("myFile.txt");
  //do something with the stream

} catch(IOException e){ //first catch block
  throw new WrapperException(e);
} finally {
  try{
    if(input != null) input.close();
  } catch(IOException e){  //second catch block
    throw new WrapperException(e);
  }
}
```

​	但即使这个异常处理也有问题。让我们假装文件存在，所以“输入”引用现在指向一个有效的文件输入流。让我们也假装处理输入流时抛出一个异常。该异常被捕获（ioexception e）块。然后重新包装向上抛，在包装的异常传播到调用栈之前，执行finally子句。如果input.close（）调用失败，并抛出一个ioexception，那么它被捕获，包装后再次抛出。然而，当从finally子句抛出包装的异常时，从第一个catch块抛出的包装异常再次被遗忘。它消失了。只有从第二个catch块抛出的包装异常被传播到调用栈中。

<br/>

​	正如你所看到的，安全的异常处理并不总是微不足道的。输入流处理示例甚至不是您可能遇到的最复杂的示例。jdbc中的事务有更多的错误可能性。尝试提交时发生异常，然后回滚，最后在尝试关闭连接时再次抛出异常，所有这些可能的异常都应该由异常处理代码来处理，因此所有抛出的异常都不能使得抛出的第一个异常消失。一种方法是确保抛出的最后一个异常包含所有以前抛出的异常。这样他们都可以找到调查错误原因的开发人员。这样他们都可以被调查错误原因的开发人员找到。这就是java 持久化框架，mr persister，实现事务异常处理的方式。