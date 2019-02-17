---
title: Java9之模块化和REPL工具
tags:
  - Java9系列
abbrlink: c640a1d5
date: 2018-03-11 22:17:36
---

jdk = jre + 开发工具集（例如Javac编译工具等）

jre = JVM + Java SE 标准类库

<br/>

### 新特性

java 9 提供了超过150项新功能特性，包括备受期待的模块化系统、可交互的 REPL 工具：jshell，JDK 编译工具，Java 公共 API 和私有代码，以及安全增强、扩展提升、性能管理改善等。可以说Java 9是一个庞大的系统工程，完全做了一个整体改变。

- 模块化系统（核心）
- jShell命令（核心）
- 多版本兼容jar包
- 接口的私有方法（重要）
- 钻石操作符的使用升级（重要）
- 语法改进：try语句（重要）
- 下划线使用限制（重要）
- String存储结构变更（一般）
- 便利的集合特性：of()（一般）
- 增强的Stream API（一般）
- 多分辨率图像 API（一般）
- 全新的HTTP客户端API（一般）
- Deprecated的相关API（一般）
- 智能Java编译工具
- 统一的JVM日志系统
- javadoc的HTML 5支持
- Javascript引擎升级：Nashorn
- java的动态编译器

<br/>

### 后续版本更迭

- 从Java 9 这个版本开始，Java 的计划发布周期是 6 个月，下一个 Java 的主版本将于 2018 年 3 月发布，命名为 Java 18.3，紧接着再过六个月将发布 Java 18.9
- 这意味着java的更新从传统的以特性驱动的发布周期，转变为以时间驱动的（6 个月为周期）发布模式，并逐步的将 Oracle JDK 原商业特性进行开源。
- 针对企业客户的需求，Oracle 将以三年为周期发布长期支持版本。

<br/>

**小步快跑，快速迭代**

<br/>

### 模块化系统

谈到 Java 9 大家往往第一个想到的就是 Jigsaw 项目。众所周知，Java 已经发展超过 20 年（95 年最初发布），Java 和相关生态在不断丰富的同时也越来越暴露出一些问题：

1. Java 运行环境的膨胀和臃肿。每次JVM启动的时候，至少会有30～60MB的内存加载，主要原因是JVM需要加载rt.jar，不管其中的类是否被classloader加载，第一步整个jar都会被JVM加载到内存当中去（而模块化可以根据模块的需要加载程序运行需要的class）
2. 当代码库越来越大，创建复杂，盘根错节的“意大利面条式代码”的几率呈指数级的增长。不同版本的类库交叉依赖导致让人头疼的问题，这些都阻碍了 Java 开发和运行效率的提升。
3. 很难真正地对代码进行封装, 而系统并没有对不同部分（也就是 JAR 文件）之间的依赖关系有个明确的概念。每一个公共类都可以被类路径之下任何其它的公共类所访问到，这样就会导致无意中使用了并不想被公开访问的 API。
4. 类路径本身也存在问题: 你怎么知晓所有需要的 JAR 都已经有了, 或者是不是会有重复的项呢?

<br/>

作为java 9 平台最大的一个特性，随着 Java 平台模块化系统的落地，开发人员无需再为不断膨胀的 Java 平台苦恼，例如，您可以使用 jlink 工具，根据需要定制运行时环境。这对于拥有大量镜像的容器应用场景或复杂依赖关系的大型应用等，都具有非常重要的意义。

<br/>

本质上讲，模块(module)的概念，其实就是**package外再裹一层**，也就是说，用**模块来管理各个package**，通过声明某个package暴露，不声明默认就是隐藏。因此，模块化使得代码组织上更安全，因为它可以指定哪些部分可以暴露，哪些部分隐藏。

<br/>

设计理念：**模块独立、化繁为简**

<br/>

实现目标：

1. 主要目的在于**减少内存的开销**
2. 只须必要模块，而非全部jdk模块，可简化各种类库和大型应用的开发和维护
3. 改进 Java SE 平台，使其可以适应不同大小的计算设备
4. 改进其安全性，可维护性，提高性能

<br/>

##### java9demo 模块

org.clay.bean包下

```java
public class Person {
  private String name;
  private int age;

  public Person() {
  }

  public Person(String name, int age) {
    this.name = name;
    this.age = age;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public int getAge() {
    return age;
  }

  public void setAge(int age) {
    this.age = age;
  }

  @Override
  public String toString() {
    return "Person{" +
      "name='" + name + '\'' +
      ", age=" + age +
      '}';
  }
}
```

org.clay.entity包下

```java
public class User {

  private String name;
  private int age;

  public User() {
  }

  public User(String name, int age) {
    this.name = name;
    this.age = age;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public int getAge() {
    return age;
  }

  public void setAge(int age) {
    this.age = age;
  }

  @Override
  public String toString() {
    return "User{" +
      "name='" + name + '\'' +
      ", age=" + age +
      '}';
  }
}
```

src下的module-info.java文件

```java
module java9demo {
  //package we export
  exports org.clay.bean;
}
```



<br>

##### javatest 模块

org.clay.java包下

```java
public class ModuleTest {

    private static final Logger LOGGER = Logger.getLogger("atguigu");

    public static void main(String[] args) {
      
        Person p = new Person("Tom",12);
        System.out.println(p);  //成功打印
//        User user = new User();//失败
        LOGGER.info("aaaaaa");
    }
}
```

src下的module-info.java文件

```java
module javatest {
  requires java9demo;//引用需要的模块。
}
```

<br>



### REPL工具，jShell命令

python 和 scala 都提供了REPL工具。

<br>

可直接执行以下代码

```java
System.out.println("hello jshell!");
```

<br/>

```java
int i = 10;
int j = 20;
int res = j + i;
```

<br/>

可以直接定义方法

```java
public void add(int i, int j){
  System.out.println(i+j);
}
```

还可以直接修改方法

```java
public void add(int i, int j){
  System.out.println(i + j + 10);
}
```

<br/>

默认情况下已经导入的包

util包

io包

net包

math包

<br>

查看之前做的操作

```java
/list
```

<br>

打开外部文件编辑add方法：

```java
/edit  add
```

<br>

执行外部java文件的方法:

```java
/open HelloWorld.java
```

<br>

没有受检查异常了

jshell会自动帮我们处理异常

以下代码正常执行

```
URL url = new URL("http://www.baidu.com");
```

<br>

退出

~~~java
/exit
~~~