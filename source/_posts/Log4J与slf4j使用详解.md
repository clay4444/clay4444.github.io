---
title: Log4J与slf4j使用详解
tags:
  - 日志框架
categories:
  - Java基础
abbrlink: 9b145405
date: 2018-03-27 14:19:54
---

### 一般 工程 中使用log4J

依赖

```xml
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```

<br/>

然后，在src/main/java目录（包的根目录即classpath）新建log4j.properties文件

```properties
log4j.rootLogger=INFO,console
log4j.additivity.org.apache=true
#console
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.Threshold=INFO
log4j.appender.console.ImmediateFlush=true
log4j.appender.console.Target=System.out
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```

<br/>

最后，新建Main.java文件

```java
package com.xmyself.log4j;
import org.apache.log4j.Logger;
public class Main {
  public static void main(String[] args) {
    new Test().test();
  }
}
class Test {
  final Logger log = Logger.getLogger(Test.class);
  public void test() {
    log.info("hello this is log4j info log");
  }
}
```



<br/>

### spring mvc工程 中 使用log4J

```xml
<context-param>
    <param-name>log4jConfigLocation</param-name>
    <param-value>classpath:/conf/log4j.properties</param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>
</listener>
```



<br/>

### 普通web工程中使用log4J

没有了spring提供的listener加载log4j.properties，我们要怎么加载这个文件呢？同样，把log4j.properties放在src/main/resources的conf目录，用servlet加载

```java
public class Log4jServlet extends HttpServlet {
  private static final long serialVersionUID = 1L;

  public void init(ServletConfig config) throws ServletException {
    String prefix = this.getClass().getClassLoader().getResource("/").getPath();
    String path = config.getInitParameter("log4j-path");
    PropertyConfigurator.configure(prefix + path);
  }
  public void doGet(HttpServletRequest req, HttpServletResponse res) throws IOException, ServletException {}
  public void doPost(HttpServletRequest req, HttpServletResponse res) throws IOException, ServletException {}
  public void destroy() {}
}
```

<br/>

编辑web.xml，添加

```xml
<servlet>
  <servlet-name>log4j</servlet-name>
  <servlet-class>com.xmyself.log4j.Log4jServlet</servlet-class>
  <init-param>
    <param-name>log4j-path</param-name>
    <param-value>conf/log4j.properties</param-value>
  </init-param>
  <load-on-startup>1</load-on-startup>
</servlet>
```

看着是不是和spring mvc的很像，甚至你也想到了，普通java工程没有指定log4j.properties的路径，那说明log4j的jar包一定有一个默认的路径。另外，建议，log4j的配置放在第一个，因为后续加载其他组件就要开始使用日志记录了。

<br/>

### log4j.properties内容

1. 日志对象（logger）
2. 输出位置（appender）
3. 输出样式（layout）

<br/>

等级：ALL、DEBUG、INFO、WARN、ERROR、FATAL、OFF，

分等级是为了设置日志输出的门槛，只有等级等于或高于这个门槛的日志才有机会输出

<br/>

#### logger

日志实例，就是代码里实例化的Logger对象

```properties
log4j.rootLogger=LEVEL,appenderName1,appenderName2,...
#表示不会在父logger的appender里输出，默认true
log4j.additivity.org.apache=false;
```

这是全局logger的配置，LEVEL用来设定日志等级，appenderName定义日志输出器，示例中的“console”就是一个日志输出器

下面给出一个更清晰的例子，配置“ com.demo.test ”包下所有类中实例化的Logger对象。

```properties
log4j.logger.com.demo.test=DEBUG,test
log4j.additivity.com.demo.test=false
```



<br/>

#### appender

日志输出器，指定logger的输出位置

```properties
log4j.appender.appenderName=className
```

appender有5种选择

org.apache.log4j.ConsoleAppender（控制台）

org.apache.log4j.FileAppender（文件）

org.apache.log4j.DailyRollingFileAppender（每天产生一个日志文件）

org.apache.log4j.RollingFileAppender（文件大小到达指定尺寸的时候产生一个新的文件）

org.apache.log4j.WriterAppender（将日志信息以流格式发送到任意指定的地方）

<br/>

每种appender都有若干配置项，下面逐一介绍：

**ConsoleAppender**（常用）

Threshold=WARN：指定日志信息的最低输出级别，默认DEBUG

ImmediateFlush=true：表示所有消息都会被立即输出，设为false则不输出，默认值是true

Target=System.err：默认值是System.out

<br/>

FileAppender：

Threshold=WARN：指定日志信息的最低输出级别，默认DEBUG

ImmediateFlush=true：表示所有消息都会被立即输出，设为false则不输出，默认true

Append=false：true表示消息增加到指定文件中，false则将消息覆盖指定的文件内容，默认true

File=D:/logs/logging.log4j：指定消息输出到logging.log4j文件

<br/>

**DailyRollingFileAppender**（常用）

Threshold=WARN：指定日志信息的最低输出级别，默认DEBUG

ImmediateFlush=true：表示所有消息都会被立即输出，设为false则不输出，默认true

Append=false：true表示消息增加到指定文件中，false则将消息覆盖指定的文件内容，默认true

File=D:/logs/logging.log4j：指定当前消息输出到logging.log4j文件

DatePattern='.'yyyy-MM：每月滚动一次日志文件，即每月产生一个新的日志文件。当前月的日志文件名为logging.log4j，前一个月的日志文件名为logging.log4j.yyyy-MM

另外，也可以指定按周、天、时、分等来滚动日志文件，对应的格式如下：

1)'.'yyyy-MM：每月

2)'.'yyyy-ww：每周

3)'.'yyyy-MM-dd：每天

4)'.'yyyy-MM-dd-a：每天两次

5)'.'yyyy-MM-dd-HH：每小时

6)'.'yyyy-MM-dd-HH-mm：每分钟

<br/>



RollingFileAppender

Threshold=WARN：指定日志信息的最低输出级别，默认DEBUG

ImmediateFlush=true：表示所有消息都会被立即输出，设为false则不输出，默认true

Append=false：true表示消息增加到指定文件中，false则将消息覆盖指定的文件内容，默认true

File=D:/logs/logging.log4j：指定消息输出到logging.log4j文件

MaxFileSize=100KB：后缀可以是KB,MB或者GB。在日志文件到达该大小时，将会自动滚动，即将原来的内容移到logging.log4j.1文件

MaxBackupIndex=2：指定可以产生的滚动文件的最大数，例如，设为2则可以产生logging.log4j.1，logging.log4j.2两个滚动文件和一个logging.log4j文件



<br/>

#### layout

指定logger输出内容及格式

layout有4种选择：

1. org.apache.log4j.HTMLLayout（以HTML表格形式布局）
2. org.apache.log4j.PatternLayout（可以灵活地指定布局模式）
3. org.apache.log4j.SimpleLayout（包含日志信息的级别和信息字符串）
4. org.apache.log4j.TTCCLayout（包含日志产生的时间、线程、类别等信息）

<br/>

layout也有配置项，下面具体介绍

HTMLLayout

LocationInfo=true：输出java文件名称和行号，默认false

Title=My Logging： 默认值是Log4J Log Messages

<br/>

**PatternLayout**（最常用的配置）

设置格式的参数说明如下：

%p：输出日志信息的优先级，即DEBUG，INFO，WARN，ERROR，FATAL

%d：输出日志时间点的日期或时间，默认格式为ISO8601，可以指定格式如：%d{yyyy/MM/dd HH:mm:ss,SSS}

%r：输出自应用程序启动到输出该log信息耗费的毫秒数

%t：输出产生该日志事件的线程名

%l：输出日志事件的发生位置，相当于%c.%M(%F:%L)的组合，包括类全名、方法、文件名以及在代码中的行数

%c：输出日志信息所属的类目，通常就是类全名

%M：输出产生日志信息的方法名

%F：输出日志消息产生时所在的文件名

%L：输出代码中的行号

%m：输出代码中指定的具体日志信息

%n：输出一个回车换行符，Windows平台为"rn"，Unix平台为"n"

%x：输出和当前线程相关联的NDC(嵌套诊断环境)

%%：输出一个"%"字符

<br/>



### log4j完整配置示例

log4j.rootLogger=DEBUG,console,dailyFile,rollingFile,logFile

log4j.additivity.org.apache=true

<br/>

控制台console日志输出器：

```properties
# 控制台(console)
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.Threshold=DEBUG
log4j.appender.console.ImmediateFlush=true
log4j.appender.console.Target=System.err
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```

<br/>

文件logFile日志输出器：

```properties
# 日志文件(logFile)
log4j.appender.logFile=org.apache.log4j.FileAppender
log4j.appender.logFile.Threshold=DEBUG
log4j.appender.logFile.ImmediateFlush=true
log4j.appender.logFile.Append=true
log4j.appender.logFile.File=D:/logs/log.log4j
log4j.appender.logFile.layout=org.apache.log4j.PatternLayout
log4j.appender.logFile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```

<br/>

滚动文件rollingFile日志输出器：

```properties
# 滚动文件(rollingFile)
log4j.appender.rollingFile=org.apache.log4j.RollingFileAppender
log4j.appender.rollingFile.Threshold=DEBUG
log4j.appender.rollingFile.ImmediateFlush=true
log4j.appender.rollingFile.Append=true
log4j.appender.rollingFile.File=D:/logs/log.log4j
log4j.appender.rollingFile.MaxFileSize=200KB
log4j.appender.rollingFile.MaxBackupIndex=50
log4j.appender.rollingFile.layout=org.apache.log4j.PatternLayout
log4j.appender.rollingFile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```

<br/>

定期滚动文件dailyFile日志输出器

```properties
# 定期滚动日志文件(dailyFile)
log4j.appender.dailyFile=org.apache.log4j.DailyRollingFileAppender
log4j.appender.dailyFile.Threshold=DEBUG
log4j.appender.dailyFile.ImmediateFlush=true
log4j.appender.dailyFile.Append=true
log4j.appender.dailyFile.File=D:/logs/log.log4j
log4j.appender.dailyFile.DatePattern='.'yyyy-MM-dd
log4j.appender.dailyFile.layout=org.apache.log4j.PatternLayout
log4j.appender.dailyFile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```

<br/>



### log4j局部日志配置

以上介绍的配置都是全局的，整个工程的代码使用同一套配置，意味着所有的日志都输出在了相同的地方，你无法直接了当的去看数据库访问日志、用户登录日志、操作日志，它们都混在一起，因此，需要为包甚至是类配置单独的日志输出，下面给出一个例子，为“com.demo.test”包指定日志输出器“test”，“com.demo.test”包下所有类的日志都将输出到/log/test.log文件

```properties
log4j.logger.com.demo.test=DEBUG,test
log4j.appender.test=org.apache.log4j.FileAppender
log4j.appender.test.File=/log/test.log
log4j.appender.test.layout=org.apache.log4j.PatternLayout
log4j.appender.test.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```

<br/>

也可以让同一个类输出不同的日志，为达到这个目的，需要在这个类中实例化两个logger

private static Log logger1 = LogFactory.getLog("myTest1");

private static Log logger2 = LogFactory.getLog("myTest2");

<br/>

然后分别配置：

```properties
log4j.logger.myTest1= DEBUG,test1
log4j.additivity.myTest1=false
log4j.appender.test1=org.apache.log4j.FileAppender
log4j.appender.test1.File=/log/test1.log
log4j.appender.test1.layout=org.apache.log4j.PatternLayout
log4j.appender.test1.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```

```properties
log4j.logger.myTest2=DEBUG,test2
log4j.appender.test2=org.apache.log4j.FileAppender
log4j.appender.test2.File=/log/test2.log
log4j.appender.test2.layout=org.apache.log4j.PatternLayout
log4j.appender.test2.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%p] %m%n
```



<br/>

### slf4j与log4j联合使用

slf4j是什么？slf4j只是定义了一组日志接口，但并未提供任何实现，既然这样，为什么要用slf4j呢？log4j不是已经满足要求了吗？

　　是的，log4j满足了要求，但是，日志框架并不只有log4j一个，你喜欢用log4j，有的人可能更喜欢logback，有的人甚至用jdk自带的日志框架，这种情况下，如果你要依赖别人的jar，整个系统就用了两个日志框架，如果你依赖10个jar，每个jar用的日志框架都不同，岂不是一个工程用了10个日志框架，那就乱了！

　　如果你的代码使用slf4j的接口，具体日志实现框架你喜欢用log4j，其他人的代码也用slf4j的接口，具体实现未知，那你依赖其他人jar包时，整个工程就只会用到log4j日志框架，这是一种典型的门面模式应用，与jvm思想相同，我们面向slf4j写日志代码，slf4j处理具体日志实现框架之间的差异，正如我们面向jvm写java代码，jvm处理操作系统之间的差异，结果就是，一处编写，到处运行。况且，现在越来越多的开源工具都在用slf4j了。

那么，怎么用slf4j呢？

　　首先，得弄到slf4j的jar包，maven依赖如下，log4j配置过程完全不变

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.21</version>
</dependency>
```

然后，弄到slf4j与log4j的关联jar包，通过这个东西，将对slf4j接口的调用转换为对log4j的调用，不同的日志实现框架，这个转换工具不同

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.21</version>
</dependency>
```

当然了，slf4j-log4j12这个包肯定依赖了slf4j和log4j，所以使用slf4j+log4j的组合只要配置上面这一个依赖就够了

<br/>

最后，代码里声明logger要改一下，原来使用log4j是这样的：

```java
import org.apache.log4j.Logger;
class Test {
  final Logger log = Logger.getLogger(Test.class);
  public void test() {
    log.info("hello this is log4j info log");
  }
}
```

现在要改成这样：

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
class Test {
  Logger log = LoggerFactory.getLogger(Test.class);
  public void test() {
    log.info("hello, my name is {}", "chengyi");
  }
}
```

依赖的Logger变了，而且，slf4j的api还能使用占位符，很方便。