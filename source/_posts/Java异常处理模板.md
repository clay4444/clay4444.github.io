---
title: Java异常处理模板
tags:
  - Java异常处理
abbrlink: e4155742
date: 2017-10-23 15:28:47
---

### 声明：

本文翻译自：http://tutorials.jenkov.com/java-exception-handling/exception-handling-templates.html#static-template-methods

### 正文：

#### 1. 正确的异常处理方式：繁琐

- 注：阅读本文前，强烈建议首先阅读本人另外一篇博客：深入探讨Java的故障保护异常处理

  正确的异常处理代码可能是非常繁琐的编写。try-catch块也会使代码混乱，使其更难阅读。看下面的例子：

~~~java
  	Input       input            = null;
    IOException processException = null;
    try{
        input = new FileInputStream(fileName);

        //...process input stream...
    } catch (IOException e) {
        processException = e;
    } finally {
       if(input != null){
          try {
             input.close();
          } catch(IOException e){
             if(processException != null){
                throw new MyException(processException, e,
                  "Error message..." +
                  fileName);
             } else {
                throw new MyException(e,
                    "Error closing InputStream for file " +
                    fileName;
             }
          }
       }
       if(processException != null){
          throw new MyException(processException,
            "Error processing InputStream for file " +
                fileName;
    }
~~~

​	在这个例子中，没有任何异常会丢失。如果从try块中抛出一个异常，并且finally块中的input.close（）调用引发了另一个异常，那么这两个异常都会保留在myexception实例中，并传播到调用堆栈中。

<br/>

​	这就是处理输入流的处理需要的代码量，这样写不会丢失任何异常。但是实际上它甚至只能捕捉到ioexceptions。如果input.close（）调用也引发异常，则不会保留从try-block抛出的runtimeexception。是不是很丑？在这种情形下，你会记得每次处理一个输入流时写下所有的代码吗？

<br/>

​	幸运的是，有一个简单的设计模式，即模板方法，可以帮助您每次都能正确地处理异常，而无需在代码中查看或编写它。好吧，也许你将不得不写一次，但就是这样。

<br/>

​	你要做的是把所有的异常处理代码放在一个模板中。模板只是一个普通的类。这里是上述输入流异常处理的模板：

~~~java
public abstract class InputStreamProcessingTemplate {

    public void process(String fileName){
        IOException processException = null;
        InputStream input = null;
        try{
            input = new FileInputStream(fileName);

            doProcess(input);
        } catch (IOException e) {
            processException = e;
        } finally {
           if(input != null){
              try {
                 input.close();
              } catch(IOException e){
                 if(processException != null){
                    throw new MyException(processException, e,
                      "Error message..." +
                      fileName);
                 } else {
                    throw new MyException(e,
                        "Error closing InputStream for file " +
                        fileName;
                 }
              }
           }
           if(processException != null){
              throw new MyException(processException,
                "Error processing InputStream for file " +
                    fileName;
        }
    }

    //override this method in a subclass, to process the stream.
    public abstract void doProcess(InputStream input) throws IOException;
}
~~~

​	所有的异常处理都被封装在inputstreamprocessingtemplate类中。请注意process（）方法如何调用try-catch块内的doprocess（）方法。您将通过继承该模板来使用该模板，并覆盖doprocess（）方法。要做到这一点，代码可写成如下形式：

~~~java
new InputStreamProcessingTemplate(){
  public void doProcess(InputStream input) throws IOException{
    int inChar = input.read();
    while(inChar !- -1){
      //do something with the chars...
    }
  }
}.process("someFile.txt");
~~~

​	本示例创建inputstreamprocessingtemplate类的匿名子类，实例化子类的一个实例，并调用其process（）方法。

<br/>

​	这写起来简单很多，而且更易于阅读。只有对流的处理逻辑在代码中可见。编译器会检查你是否正确地扩展了inputstreamprocessingtemplate。在编写代码时，通常也会从ide的快捷键中获得更多的帮助，因为ide将识别doprocess（）和process（）方法。

<br/>

​	您现在可以在需要处理文件输入流的代码中的任何位置重新使用inputstreamprocessingtemplate。您可以轻松修改模板以适用于所有输入流而不仅仅是文件。

<br/>

​	您现在可以在需要处理文件输入流的代码中的任何位置重新使用inputstreamprocessingtemplate。您可以轻松修改模板以适用于所有输入流而不仅仅是文件。

#### 2. 使用接口而不是子类化

我们还可以使用接口的形式

~~~java
public interface InputStreamProcessor {
    public void process(InputStream input) throws IOException;
}


public class InputStreamProcessingTemplate {

    public void process(String fileName, InputStreamProcessor processor){
        IOException processException = null;
        InputStream input = null;
        try{
            input = new FileInputStream(fileName);

            processor.process(input);
        } catch (IOException e) {
            processException = e;
        } finally {
           if(input != null){
              try {
                 input.close();
              } catch(IOException e){
                 if(processException != null){
                    throw new MyException(processException, e,
                      "Error message..." +
                      fileName;
                 } else {
                    throw new MyException(e,
                        "Error closing InputStream for file " +
                        fileName);
                 }
              }
           }
           if(processException != null){
              throw new MyException(processException,
                "Error processing InputStream for file " +
                    fileName;
        }
    }
}
~~~

​	注意模板的process（）方法中的额外参数。这是从try块（processor.process（input））中调用的输入流处理器。使用方法如下：

~~~java
new InputStreamProcessingTemplate()
  .process("someFile.txt", new InputStreamProcessor(){
    public void process(InputStream input) throws IOException{
      int inChar = input.read();
      while(inChar !- -1){
        //do something with the chars...
      }
    }
  });
~~~

​	除了对inputstreamprocessingtemplate.process（）方法的调用现在更接近代码的顶部之外，它与以前的用法看起来没什么两样。但是这可能会更容易阅读。

#### 3. 静态模板方法

​	我们还可以将模板方法实现为静态方法。这样你就不需要在每次调用它的时候都将这个模板实例化为一个对象。这里是inputstreamprocessingtemplate的静态方法的实现方式：

~~~java
public class InputStreamProcessingTemplate {

    public static void process(String fileName,
    InputStreamProcessor processor){
        IOException processException = null;
        InputStream input = null;
        try{
            input = new FileInputStream(fileName);

            processor.process(input);
        } catch (IOException e) {
            processException = e;
        } finally {
           if(input != null){
              try {
                 input.close();
              } catch(IOException e){
                 if(processException != null){
                    throw new MyException(processException, e,
                      "Error message..." +
                      fileName);
                 } else {
                    throw new MyException(e,
                        "Error closing InputStream for file " +
                        fileName;
                 }
              }
           }
           if(processException != null){
              throw new MyException(processException,
                "Error processing InputStream for file " +
                    fileName;
        }
    }
}
~~~

​	这里只是把process（...）方法设置成了静态的。调用该方法的过程如下：

~~~java
 InputStreamProcessingTemplate.process("someFile.txt",
        new InputStreamProcessor(){
            public void process(InputStream input) throws IOException{
                int inChar = input.read();
                while(inChar !- -1){
                    //do something with the chars...
                }
            }
        });
~~~

​	注意调用模板的process（）静态方法的调用过程。

#### 4. 总结

​	异常处理模板是一个简单而强大的机制，可以提高代码的质量和可读性。它也提高了你的工作效率，因为你写的代码少得多，而且不用担心丢失异常。异常全部由模板处理。而且，如果在开发过程中稍后需要改进异常处理，则只有一个地方可以对其进行更改：异常处理模板。

​	模板方法设计模式可用于除异常处理之外的其他目的。输入流的迭代也可以被放入模板中。jdbc中结果集的迭代可以放入模板中。在jdbc中正确执行一个事务可以放入一个模板中。等等其他各种用途。