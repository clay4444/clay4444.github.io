---
title: Spring注解驱动开发之组件注册
tags:
  - Spring注解驱动开发
abbrlink: 9c6cea3b
date: 2018-05-03 11:59:22
---

#### 使用基于配置文件的注册

1. 新建Person类

```java
public class Person {

    private String name;
    private Integer age;

    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public Integer getAge() {
        return age;
    }
    public void setAge(Integer age) {
        this.age = age;
    }

    public Person(String name, Integer age) {
        super();
        this.name = name;
        this.age = age;
    }
    public Person() {
        super();
        // TODO Auto-generated constructor stub
    }
    @Override
    public String toString() {
        return "Person [name=" + name + ", age=" + age + ", nickName=" + nickName + "]";
    }
}
```

1. resource资源目录下新建beans.xml

```xml
<bean id="person" class="org.clay.bean.Person" >
    <property name="age" value="1"></property>
    <property name="name" value="zhangsan"></property>
</bean>
```

1. 从spring容器中取出自动装配好的bean

```java
public class MainTest {

    @SuppressWarnings("resource")
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans.xml");
        Person bean = (Person) applicationContext.getBean("person");
        System.out.println(bean);
    }
}
```

<br/>

#### 使用@configuration和@Bean给容器中注册组件

1. 新建MainConfig (配置文件类)

```java
//配置类==配置文件
@Configuration  //告诉Spring这是一个配置类
public class MainConfig {

    //给容器中注册一个Bean;类型为返回值的类型，id默认是用方法名作为id
    @Bean("person")
    public Person person01(){
        return new Person("lisi", 20);
    }
}
```

1. 通过AnnotationConfigApplicationContext获取bean

```java
public class MainTest {

    @SuppressWarnings("resource")
    public static void main(String[] args) {

        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfig.class);
        Person bean = applicationContext.getBean(Person.class);
        System.out.println(bean);

        String[] namesForType = applicationContext.getBeanNamesForType(Person.class);
        for (String name : namesForType) {
            System.out.println(name);//打印beanName
        }
    }
}
```

<br/>

#### ComponentScan 自动扫描规则

1. 基于xml文件的方式

```xml
<!-- 包扫描、只要标注了@Controller、@Service、@Repository，@Component -->
<context:component-scan base-package="org.clay" use-default-filters="false">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"></context:include-filter>
</context:component-scan>
<bean id="person" class="org.clay.bean.Person" scope="prototype">
    <property name="age" value="1"></property>
    <property name="name" value="zhangsan"></property>
</bean>
```

1. 基于注解的方式

```java
//配置类==配置文件
@Configuration  //告诉Spring这是一个配置类
@ComponentScans(
    value = {
        @ComponentScan(value="org.clay",includeFilters = {
            @Filter(type=FilterType.ANNOTATION,classes={Controller.class}),
            @Filter(type=FilterType.ASSIGNABLE_TYPE,classes={BookService.class}),
            @Filter(type=FilterType.CUSTOM,classes={MyTypeFilter.class})
        },useDefaultFilters = false)	
    }
)
//@ComponentScan  value:指定要扫描的包
//excludeFilters = Filter[] ：指定扫描的时候按照什么规则排除那些组件
//includeFilters = Filter[] ：指定扫描的时候只需要包含哪些组件
//FilterType.ANNOTATION：按照注解
//FilterType.ASSIGNABLE_TYPE：按照给定的类型；
//FilterType.ASPECTJ：使用ASPECTJ表达式
//FilterType.REGEX：使用正则指定
//FilterType.CUSTOM：使用自定义规则
public class MainConfig {

    //给容器中注册一个Bean;类型为返回值的类型，id默认是用方法名作为id
    @Bean("person")
    public Person person01(){
        return new Person("lisi", 20);
    }
}
```

<br/>

==使用自定义规则的方式进行过滤==

查看Filter源码

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({})
@interface Filter {

    /**
		 * The type of filter to use.
		 * <p>Default is {@link FilterType#ANNOTATION}.
		 * @see #classes
		 * @see #pattern
		 */
    FilterType type() default FilterType.ANNOTATION;

    /**
		 * Alias for {@link #classes}.
		 * @see #classes
		 */
    @AliasFor("classes")
    Class<?>[] value() default {};
}
```

点击FilterType  查看Customer类型

```java
public enum FilterType {
    /** Filter candidates using a given custom
	 * {@link org.springframework.core.type.filter.TypeFilter} implementation.
	 */
    CUSTOM
}
```

所以：实现org.springframework.core.type.filter.TypeFilter即可。

<br/>

1. 自定义过滤规则

```java
public class MyTypeFilter implements TypeFilter {

    /**
	 * metadataReader：读取到的当前正在扫描的类的信息
	 * metadataReaderFactory:可以获取到其他任何类信息的
	 */
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
        throws IOException {
        // TODO Auto-generated method stub
        //获取当前类注解的信息
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
        //获取当前正在扫描的类的类信息
        ClassMetadata classMetadata = metadataReader.getClassMetadata();
        //获取当前类资源（类的路径）
        Resource resource = metadataReader.getResource();

        String className = classMetadata.getClassName();
        System.out.println("--->"+className);
        if(className.contains("er")){
            return true;
        }
        return false;
    }
}
```

<br/>

#### Scope设置作用域单实例的懒加载

```java
@Configuration
public class MainConfig2 {

    //默认是单实例的
    /**
	 * @Scope:调整作用域
	 * prototype：多实例的：ioc容器启动并不会去调用方法创建对象放在容器中。
	 * 					 每次获取的时候才会调用方法创建对象；
	 * singleton：单实例的（默认值）：ioc容器启动会调用方法创建对象放到ioc容器中。
	 * 			以后每次获取就是直接从容器（map.get()）中拿，
	 * request：同一次请求创建一个实例
	 * session：同一个session创建一个实例
	 * 
	 * 懒加载：
	 * 		单实例bean：默认在容器启动的时候创建对象；
	 * 		懒加载：容器启动不创建对象。第一次使用(获取)Bean创建对象，并初始化；
	 */
    @Scope("singleton")
    @Lazy
    @Bean("person")
    public Person person(){
        System.out.println("给容器中添加Person....");
        return new Person("张三", 25);
    }
}
```

测试

```java
@Test
public void test02(){
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfig2.class);

    System.out.println("ioc容器创建完成....");
    Object bean = applicationContext.getBean("person");
    Object bean2 = applicationContext.getBean("person");
    System.out.println(bean == bean2);
}
```

打印结果

```java
true		//且只打印一次给容器中添加Person....
```

<br/>

#### 按照一定的条件注册bean

1. 在要注册到容器中的bean上添加@Conditional注解

```java
@Configuration
public class MainConfig2 {
    /**
	 * @Conditional({Condition}) ： 按照一定的条件进行判断，满足条件给容器中注册bean
	 * 
	 * 如果系统是windows，给容器中注册("bill")
	 * 如果是linux系统，给容器中注册("linus")
	 */
    @Conditional({WindowsCondition.class})
    @Bean("bill")
    public Person person01(){
        return new Person("Bill Gates",62);
    }

    @Conditional(LinuxCondition.class)
    @Bean("linus")
    public Person person02(){
        return new Person("linus", 48);
    }
}
```

1. 编写类实现Condition接口

```java
//判断是否linux系统
public class LinuxCondition implements Condition {

    /**
	 * ConditionContext：判断条件能使用的上下文（环境）
	 * AnnotatedTypeMetadata：注释信息
	 */
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // TODO是否linux系统
        //1、能获取到ioc使用的beanfactory
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        //2、获取类加载器
        ClassLoader classLoader = context.getClassLoader();
        //3、获取当前环境信息
        Environment environment = context.getEnvironment();
        //4、获取到bean定义的注册类
        BeanDefinitionRegistry registry = context.getRegistry();

        String property = environment.getProperty("os.name");

        //可以判断容器中的bean注册情况，也可以给容器中注册bean
        boolean definition = registry.containsBeanDefinition("person");
        if(property.contains("linux")){
            return true;
        }
        return false;
    }
}
```

1. 测试

```java
@Test
public void test03(){
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfig2.class);
    ConfigurableEnvironment environment = applicationContext.getEnvironment();
    //动态获取环境变量的值；Windows 10
    String property = environment.getProperty("os.name");
    System.out.println(property);
    for (String name : namesForType) {
        System.out.println(name);//没有linus
    }
}
```

<br/>

#### @import注解快速导入一个组件

1. 新建Color，Red类

```java
public class Color {
}
```

```java
public class Red {
}
```

1. 编写MyImportSelector实现ImportSelector接口

```java
//自定义逻辑返回需要导入的组件
public class MyImportSelector implements ImportSelector {

    //返回值，就是到导入到容器中的组件全类名
    //AnnotationMetadata:当前标注@Import注解的类的所有注解信息
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // TODO Auto-generated method stub
        //importingClassMetadata
        //方法不要返回null值
        return new String[]{"com.atguigu.bean.Blue","com.atguigu.bean.Yellow"};
    }
}
```

1. 编写MyImportBeanDefinitionRegistrar实现ImportBeanDefinitionRegistrar接口

==spring-boot 大量使用到此种方式==

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    /**
	 * AnnotationMetadata：当前类的注解信息
	 * BeanDefinitionRegistry:BeanDefinition注册类；
	 * 		把所有需要添加到容器中的bean；调用
	 * 		BeanDefinitionRegistry.registerBeanDefinition手工注册进来
	 */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        boolean definition = registry.containsBeanDefinition("com.atguigu.bean.Red");
        boolean definition2 = registry.containsBeanDefinition("com.atguigu.bean.Blue");
        if(definition && definition2){
            //指定Bean定义信息；（Bean的类型，Bean。。。）
            RootBeanDefinition beanDefinition = new RootBeanDefinition(RainBow.class);
            //注册一个Bean，指定bean名
            registry.registerBeanDefinition("rainBow", beanDefinition);
        }
    }
}
```

1. @import 注解导入

```java
@Import({Color.class,Red.class,MyImportSelector.class,MyImportBeanDefinitionRegistrar.class})
//@Import导入组件，id默认是组件的全类名
public class MainConfig2 {
    /**
	 * 给容器中注册组件；
	 * 1）、包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]
	 * 2）、@Bean[导入的第三方包里面的组件]
	 * 3）、@Import[快速给容器中导入一个组件]
	 * 		1）、@Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
	 * 		2）、ImportSelector:返回需要导入的组件的全类名数组；
	 * 		3）、ImportBeanDefinitionRegistrar:手动注册bean到容器中
	 * 4）、使用Spring提供的 FactoryBean（工厂Bean）;
	 * 		1）、默认获取到的是工厂bean调用getObject创建的对象
	 * 		2）、要获取工厂Bean本身，我们需要给id前面加一个&
	 * 			&colorFactoryBean
	 */
    @Bean
    public ColorFactoryBean colorFactoryBean(){
        return new ColorFactoryBean();
    }
}
```

1. 测试

```java
private void printBeans(AnnotationConfigApplicationContext applicationContext){
    String[] definitionNames = applicationContext.getBeanDefinitionNames();
    for (String name : definitionNames) {
        System.out.println(name);
    }
}

@Test
public void testImport(){
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfig2.class);
    printBeans(applicationContext);
    Blue bean = applicationContext.getBean(Blue.class);
    System.out.println(bean);//导入成功
}
```

