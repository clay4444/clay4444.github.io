---
title: Java的反射与动态代理
categories:
  - Java基础
abbrlink: 13325b7b
date: 2017-12-17 19:45:58
---

## Java Reflection

- Reflection（反射）是被视为动态语言的关键，反射机制允许程序在执行期借助于Reflection API取得任何类的内部信息，并能直接操作任意对象的内部属性及方法
- Java反射机制提供的功能

1. 在运行时判断任意一个对象所属的类
2. 在运行时构造任意一个类的对象
3. 在运行时判断任意一个类所具有的成员变量和方法
4. 在运行时调用任意一个对象的成员变量和方法
5. 生成动态代理

- 反射相关的主要API：

1. java.lang.Class:代表一个类
2. java.lang.reflect.Method:代表类的方法 (这是一个类了，不是一个方法)
3. java.lang.reflect.Field:代表类的成员变量
4. java.lang.reflect.Constructor:代表类的构造方法

## Class 类

- 在Object类中定义了以下的方法，此方法将被所有子类继承：

```
public final Class getClass()
```

- 以上的方法返回值的类型是一个Class类，此类是Java反射的源头，实际上所谓反射从程序的运行结果来看也很好理解，即：可以通过对象反射求出类的名称。
- 有了反射，可以通过反射创建一个类的对象，并调用其中的结构

```java
@Test
public void test2() throws Exception{
  Class clazz = Person.class;

  //		Class clazz1 = String.class;

  //1.创建clazz对应的运行时类Person类的对象
  Person p = (Person)clazz.newInstance();
  System.out.println(p);
  //2.通过反射调用运行时类的指定的属性
  //2.1
  Field f1 = clazz.getField("name");
  f1.set(p,"LiuDeHua");
  System.out.println(p);
  //2.2
  Field f2 = clazz.getDeclaredField("age");
  f2.setAccessible(true);
  f2.set(p, 20);
  System.out.println(p);

  //3.通过反射调用运行时类的指定的方法
  Method m1 = clazz.getMethod("show");
  m1.invoke(p);

  Method m2 = clazz.getMethod("display",String.class);
  m2.invoke(p,"CHN");

}
```

- java.lang.Class:是反射的源头。
- 我们创建了一个类，通过编译（javac.exe）,生成对应的.class文件。之后我们使用java.exe加载（JVM的类加载器完成的）
- 此.class文件，此.class文件加载到内存以后，就是一个运行时类，存在在缓存区。那么这个运行时类本身就是一个Class的实例！
- 1.每一个运行时类只加载一次！
- 2.有了Class的实例以后，我们才可以进行如下的操作：
- ----1）*创建对应的运行时类的对象
- ----2）获取对应的运行时类的完整结构（属性、方法、构造器、内部类、父类、所在的包、异常、注解、...）
- ----3）*调用对应的运行时类的指定的结构(属性、方法、构造器)
- ----4）反射的应用：动态代理

## 如何获取Class的实例（3种）

- 1.调用运行时类本身的.class属性

```java
Class clazz1 = Person.class;
System.out.println(clazz1.getName());

Class clazz2 = String.class;
System.out.println(clazz2.getName());
```

- 通过运行时类的对象获取

```java
Person p = new Person();
Class clazz3 = p.getClass();
System.out.println(clazz3.getName());
```

- 3.通过Class的静态方法获取.通过此方式，体会一下，反射的动态性。

```java
String className = "com.atguigu.java.Person";
		
Class clazz4 = Class.forName(className);
clazz4.newInstance();
System.out.println(clazz4.getName());
```

- 4.（了解）通过类的加载器

```java
ClassLoader classLoader = this.getClass().getClassLoader();
Class clazz5 = classLoader.loadClass(className);
System.out.println(clazz5.getName());

System.out.println(clazz1 == clazz3);//true             三个相等说明Person类就加载一次。
System.out.println(clazz1 == clazz4);//true
System.out.println(clazz1 == clazz5);//true
```

## 类的加载过程

- 当程序主动使用某个类时，如果该类还未被加载到内存中，则系统会通过如下三个步骤来对该类进行初始化。
- 1. 类的加载，将类的class文件读入内存，并为之创建一个java.lang.Class对象。此过程由类加载器完成
- 1. 类的连接，将类的二进制数据合并到JRE中
- 1. 类的初始化，JVM负责对类进行初始化
- ClassLoader
- 类加载器是用来把类(class)装载进内存的。JVM规范定义了两种类型的类加载器：启动类加载器(bootstrap)和用户自定义加载器(user-defined class loader)。 JVM在运行时会产生3个类加载器组成的初始化加载器层次结构 ，如下图所示：
- 引导类加载器：用C++编写的，是JVM自带的类加载器，负责Java平台核心库，用来加载核心类库。该加载器无法直接获取
- 扩展类加载器：负责jre/lib/ext目录下的jar包或 –D java.ext.dirs 指定目录下的jar包装入工作库
- 系统类加载器：负责java –classpath 或 –D java.class.path所指的目录下的类与jar包装入工作 ，是最常用的加载器

```java
//关于类的加载器：ClassLoader
@Test
public void test5() throws Exception{
  ClassLoader loader1 = ClassLoader.getSystemClassLoader();
  System.out.println(loader1);

  ClassLoader loader2 = loader1.getParent();
  System.out.println(loader2);

  ClassLoader loader3 = loader2.getParent();          引导类加载器加载，是无法获取到的
    System.out.println(loader3);

  Class clazz1 = Person.class;
  ClassLoader loader4 = clazz1.getClassLoader();     系统类加载器
    System.out.println(loader4);

  String className = "java.lang.String";
  Class clazz2 = Class.forName(className);
  ClassLoader loader5 = clazz2.getClassLoader();      核心类库由引导类加载器加载，是无法获取到的
    System.out.println(loader5);

  //掌握如下
  //法一：
  ClassLoader loader = this.getClass().getClassLoader();
  InputStream is = loader.getResourceAsStream("com\\atguigu\\java\\jdbc.properties");
  //法二：
  //	FileInputStream is = new FileInputStream(new File("jdbc1.properties"));

  Properties pros = new Properties();
  pros.load(is);
  String name = pros.getProperty("user");
  System.out.println(name);

  String password = pros.getProperty("password");
  System.out.println(password);

}
```

## 创建类对象并获取类的完整结构

- 有了Class对象，能做什么？
- 创建类的对象：调用Class对象的newInstance()方法
  - 要 求：
- ----------1）类必须有一个无参数的构造器。
- ----------2）类的构造器的访问权限需要足够。
- 难道没有无参的构造器就不能创建对象了吗？
- 不是！只要在操作的时候明确的调用类中的构造方法，并将参数传递进去之后，才可以实例化操作。步骤如下

1. 通过Class类的getDeclaredConstructor(Class … parameterTypes)取得本类的指定形参类型的构造器
2. 向构造器的形参中传递一个对象数组进去，里面包含了构造器中所需的各个参数。 以上是反射机制应用最多的地方。

```java
@Test
public void test1() throws Exception{
  String className = "com.atguigu.java.Person";
  Class clazz = Class.forName(className);
  //创建对应的运行时类的对象。使用newInstance()，实际上就是调用了运行时类的空参的构造器。
  //要想能够创建成功：①要求对应的运行时类要有空参的构造器。②构造器的权限修饰符的权限要足够。
  Object obj = clazz.newInstance();
  Person p = (Person)obj;
  System.out.println(p);
}
```

~~~java
//调用指定的构造器,创建运行时类的对象
@Test
public void test3() throws Exception{
  String className = "com.atguigu.java.Person";
  Class clazz = Class.forName(className);

  Constructor cons = clazz.getDeclaredConstructor(String.class,int.class);
  cons.setAccessible(true);
  Person p = (Person)cons.newInstance("罗伟",20);
  System.out.println(p);
}
~~~

## 通过反射调用类的完整结构

### 获取对应的运行时类的属性

1. getFields():只能获取到运行时类中及其父类中声明为public的属性

```java
Class clazz = Person.class;
Field[] fields = clazz.getFields();
for(int i = 0;i < fields.length;i++){
  System.out.println(fields[i]);
}
```

2. getDeclaredFields():获取运行时类本身声明的所有的属性

```java
Field[] fields1 = clazz.getDeclaredFields();
for(Field f : fields1){
  System.out.println(f.getName());
}
```

3. 获取属性的权限修饰符 变量类型 变量名

```java
@Test
public void test2(){
  Class clazz = Person.class;
  Field[] fields1 = clazz.getDeclaredFields();
  for(Field f : fields1){
    //1.获取每个属性的权限修饰符
    int i = f.getModifiers();
    String str1 = Modifier.toString(i);
    System.out.print(str1 + " ");
    //2.获取属性的类型
    Class type = f.getType();
    System.out.print(type.getName() + " ");
    //3.获取属性名
    System.out.print(f.getName());

    System.out.println();
  }
}
```

4. 调用运行时类中指定的属性

- //getField(String fieldName):获取运行时类中声明为public的指定属性名为fieldName的属性
- //getDeclaredField(String fieldName):获取运行时类中指定的名为fieldName的属性

```java
@Test
public void test3() throws Exception{
  Class clazz = Person.class;
  //1.获取指定的属性

  Field name = clazz.getField("name");
  //2.创建运行时类的对象 
  Person p = (Person)clazz.newInstance();
  System.out.println(p);
  //3.将运行时类的指定的属性赋值
  name.set(p,"Jerry");
  System.out.println(p);
  System.out.println("%"+name.get(p));

  System.out.println();
  Field age = clazz.getDeclaredField("age");
  //由于属性权限修饰符的限制，为了保证可以给属性赋值，需要在操作前使得此属性可被操作。
  age.setAccessible(true);
  age.set(p,10);
  System.out.println(p);

  //	Field id = clazz.getField("id");

}
```

### 获取运行时类的方法

1. getMethods():获取运行时类及其父类中所有的声明为public的方法

```java
Class clazz = Person.class;
Method[] m1 = clazz.getMethods();
for(Method m : m1){
  System.out.println(m);
}
```

2. getDeclaredMethods():获取运行时类本身声明的所有的方法

```java
Method[] m2 = clazz.getDeclaredMethods();
for(Method m : m2){
  System.out.println(m);
}
```

3. 注解 权限修饰符 返回值类型 方法名 形参列表 异常

```java
@Test
public void test2(){
  Class clazz = Person.class;

  Method[] m2 = clazz.getDeclaredMethods();
  for(Method m : m2){
    //1.注解
    Annotation[] ann = m.getAnnotations();
    for(Annotation a : ann){
      System.out.println(a);
    }

    //2.权限修饰符
    String str = Modifier.toString(m.getModifiers());
    System.out.print(str + " ");
    //3.返回值类型
    Class returnType = m.getReturnType();
    System.out.print(returnType.getName() + " ");
    //4.方法名
    System.out.print(m.getName() + " ");

    //5.形参列表
    System.out.print("(");
    Class[] params = m.getParameterTypes();
    for(int i = 0;i < params.length;i++){
      System.out.print(params[i].getName() + " args-" + i + " ");
    }
    System.out.print(")");

    //6.异常类型
    Class[] exps = m.getExceptionTypes();
    if(exps.length != 0){
      System.out.print("throws ");
    }
    for(int i = 0;i < exps.length;i++){
      System.out.print(exps[i].getName() + " ");
    }
    System.out.println();
  }
}
```

4. getMethod(String methodName,Class ... params):获取运行时类中声明为public的指定的方法

```java
Class clazz = Person.class;
Method m1 = clazz.getMethod("show");
Person p = (Person)clazz.newInstance();
```

5. 调用指定的方法：Object invoke(Object obj,Object ... obj)

```java
Object returnVal = m1.invoke(p);//我是一个人
System.out.println(returnVal);//null

Method m2 = clazz.getMethod("toString");
Object returnVal1 = m2.invoke(p);
System.out.println(returnVal1);//Person [name=null, age=0]
```

6. 对于运行时类中静态方法的调用

```java
Method m3 = clazz.getMethod("info");
m3.invoke(Person.class);
```

7. getDeclaredMethod(String methodName,Class ... params):获取运行时类中声明了的指定的方法

```java
Method m4 = clazz.getDeclaredMethod("display",String.class,Integer.class);
m4.setAccessible(true);
Object value = m4.invoke(p,"CHN",10);//我的国籍是：CHN
System.out.println(value);//10
```

### 获取运行时类的构造器

- 创建对应的运行时类的对象。使用newInstance()，实际上就是调用了运行时类的空参的构造器。
- 要想能够创建成功：①要求对应的运行时类要有空参的构造器。②构造器的权限修饰符的权限要足够。

```java
@Test
public void test1() throws Exception{
  String className = "com.atguigu.java.Person";
  Class clazz = Class.forName(className);
  //创建对应的运行时类的对象。使用newInstance()，实际上就是调用了运行时类的空参的构造器。
  //要想能够创建成功：①要求对应的运行时类要有空参的构造器。②构造器的权限修饰符的权限要足够。
  Object obj = clazz.newInstance();
  Person p = (Person)obj;
  System.out.println(p);
}
```

```java
@Test
public void test2() throws ClassNotFoundException{
  String className = "com.atguigu.java.Person";
  Class clazz = Class.forName(className);

  Constructor[] cons = clazz.getDeclaredConstructors();
  for(Constructor c : cons){
    System.out.println(c);
  }
}
```

- 调用指定的构造器,创建运行时类的对象

```java
@Test
public void test3() throws Exception{
  String className = "com.atguigu.java.Person";
  Class clazz = Class.forName(className);

  Constructor cons = clazz.getDeclaredConstructor(String.class,int.class);
  cons.setAccessible(true);
  Person p = (Person)cons.newInstance("罗伟",20);
  System.out.println(p);
}
```

### 获取其他

1. 获取运行时类的父类

```java
@Test
public void test1(){
  Class clazz = Person.class;
  Class superClass = clazz.getSuperclass();
  System.out.println(superClass);
}
```

2. 获取带泛型的父类

```java
@Test
public void test2(){
  Class clazz = Person.class;
  Type type1 = clazz.getGenericSuperclass();
  System.out.println(type1);
}
```

3. 获取父类的泛型

```java
public void test3(){
  Class clazz = Person.class;
  Type type1 = clazz.getGenericSuperclass();

  ParameterizedType param = (ParameterizedType)type1;
  Type[] ars = param.getActualTypeArguments();

  System.out.println(((Class)ars[0]).getName());
}
```

4. 获取实现的接口

~~~java
@Test 
public void test4(){
  Class clazz = Person.class; 
  Class[] interfaces = clazz.getInterfaces(); 
  for(Class i : interfaces){ 
    System.out.println(i); 
  } 
}
~~~

5. 获取所在的包

```java
@Test
public void test5(){
  Class clazz = Person.class;
  Package pack = clazz.getPackage();
  System.out.println(pack);
}
```

6. 获取注解

```java
@Test
public void test6(){
  Class clazz = Person.class;
  Annotation[] anns = clazz.getAnnotations();
  for(Annotation a : anns){
    System.out.println(a);
  }
}
```

- 静态代理的特征是代理类和目标对象的类都是在编译期间确定下来，不利于程序的扩展。同时，每一个代理类只能为一个接口服务，这样一来程序开发中必然产生过多的代理。
- 最好可以通过一个代理类完成全部的代理功能
- 动态代理是指 **客户通过代理类来调用其它对象的方法，并且是在程序运行时根据需要动态创建目标类的代理对象 **。
- 代理设计模式的原理: **使用一个代理将对象（InvocationHandler）包装起来, 然后用该代理对象取代原始对象. 任何对原始对象的调用都要通过代理. 代理对象决定是否以及何时将方法调用转到原始对象上**

```java
//动态代理的使用，体会反射是动态语言的关键
interface Subject {
  void action();
}

// 被代理类
class RealSubject implements Subject {
  public void action() {
    System.out.println("我是被代理类，记得要执行我哦！么么~~");
  }
}

class MyInvocationHandler implements InvocationHandler {
  Object obj;// 实现了接口的被代理类的对象的声明

  // ①给被代理的对象实例化②返回一个代理类的对象
  public Object blind(Object obj) {
    this.obj = obj;
    return Proxy.newProxyInstance(obj.getClass().getClassLoader(), obj
                                  .getClass().getInterfaces(), this);
  }
  //当通过代理类的对象发起对被重写的方法的调用时，都会转换为对如下的invoke方法的调用
  @Override
  public Object invoke(Object proxy, Method method, Object[] args)
    throws Throwable {
    //method方法的返回值时returnVal
    Object returnVal = method.invoke(obj, args);
    return returnVal;
  }

}

public class TestProxy {
  public static void main(String[] args) {
    //1.被代理类的对象
    RealSubject real = new RealSubject();
    //2.创建一个实现了InvacationHandler接口的类的对象
    MyInvocationHandler handler = new MyInvocationHandler();
    //3.调用blind()方法，动态的返回一个同样实现了real所在类实现的接口Subject的代理类的对象。
    Object obj = handler.blind(real);
    Subject sub = (Subject)obj;//此时sub就是代理类的对象

    sub.action();//转到对InvacationHandler接口的实现类的invoke()方法的调用

  }
}
```

## 动态代理与AOP

- 原理是和动态代理一样的，只是在invoke方法调用的中上下各添加一个方法。

```java
interface Human {
  void info();

  void fly();
}

// 被代理类
class SuperMan implements Human {
  public void info() {
    System.out.println("我是超人！我怕谁！");
  }

  public void fly() {
    System.out.println("I believe I can fly!");
  }
}

class HumanUtil {
  public void method1() {
    System.out.println("=======方法一=======");
  }

  public void method2() {
    System.out.println("=======方法二=======");
  }
}

class MyInvocationHandler implements InvocationHandler {
  Object obj;// 被代理类对象的声明

  public void setObject(Object obj) {
    this.obj = obj;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args)
    throws Throwable {
    HumanUtil h = new HumanUtil();
    h.method1();

    Object returnVal = method.invoke(obj, args);

    h.method2();
    return returnVal;
  }
}

class MyProxy {
  // 动态的创建一个代理类的对象
  public static Object getProxyInstance(Object obj) {
    MyInvocationHandler handler = new MyInvocationHandler();
    handler.setObject(obj);

    return Proxy.newProxyInstance(obj.getClass().getClassLoader(), obj
                                  .getClass().getInterfaces(), handler);
  }
}

public class TestAOP {
  public static void main(String[] args) {
    SuperMan man = new SuperMan();//创建一个被代理类的对象
    Object obj = MyProxy.getProxyInstance(man);//返回一个代理类的对象
    Human hu = (Human)obj;
    hu.info();//通过代理类的对象调用重写的抽象方法

    System.out.println();

    hu.fly();

  }
}
```

## 写个总结

1. 其实动态代理和静态代理的道理是相同的，都是一个代理类和一个被代理类，让他们实现同一个接口，然后在代理类的内个要执行的方法中，去想办法实例化被代理类，只不过静态代理是编写的时候就指定了，动态代理是利用反射，在执行过程中，动态的判断要代理的类
2. 动态代理的技术点：

- ①代理类的实例化过程是利用Proxy.newProxyInstance动态的去获取被代理类实现的类加载器和实现的接口，以此利用反射来动态的创建一个符合条件(和被代理类实现相同的接口)的代理类。
- ②当利用多态执行代理类的相应方法的时候，会转化为调用实现InvocationHandler接口的内个类的invoke方法。会把调用的这个方法传到invoke方法中，调用invoke的方法的时候，传入被代理类的对象，那么执行的就是被代理类对象的相应方法，因为被代理类和代理类都实现了相同的接口
- ③handler就相当于一个回调方法，调用代理类的相关方法的时候，就转化为调用invoke方法，而且handler方法中，还有已经实例化的被代理类，调用invoke的方法的时候，传入被代理类的对象，就相当于做了代理。