---
title: Spring注解驱动开发之生命周期-属性赋值-自动装配
tags:
  - Spring注解驱动开发
abbrlink: f1cf47b3
date: 2019-07-07 15:13:13
---

### 生命周期

#### 指定初始化和销毁方法

```java
/**
 * bean的生命周期：
 * 		bean创建---初始化----销毁的过程
 * 容器管理bean的生命周期；
 * 我们可以自定义初始化和销毁方法；容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法
 * 
 * 构造（对象创建）
 * 		单实例：在容器启动的时候创建对象
 * 		多实例：在每次获取的时候创建对象
 * 
 * BeanPostProcessor.postProcessBeforeInitialization
 * 初始化：
 * 		对象创建完成，并赋值好，调用初始化方法。。。
 * BeanPostProcessor.postProcessAfterInitialization
 * 销毁：
 * 		单实例：容器关闭的时候
 * 		多实例：容器不会管理这个bean；容器不会调用销毁方法；
 * 
 * 
 * 遍历得到容器中所有的BeanPostProcessor；挨个执行beforeInitialization，
 * 一但返回null，跳出for循环，不会执行后面的BeanPostProcessor.postProcessorsBeforeInitialization
 * 
 * BeanPostProcessor原理 下面是IOC的源码
 * populateBean(beanName, mbd, instanceWrapper);给bean进行属性赋值
 * initializeBean
 * {
 * applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
 * invokeInitMethods(beanName, wrappedBean, mbd);执行自定义初始化
 * applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
 *}
 * 
 *
 * 1）、指定初始化和销毁方法；
 * 		通过@Bean指定init-method和destroy-method；
 * 2）、通过让Bean实现InitializingBean（定义初始化逻辑），
 * 				DisposableBean（定义销毁逻辑）;
 * 3）、可以使用JSR250；
 * 		@PostConstruct：在bean创建完成并且属性赋值完成；来执行初始化方法
 * 		@PreDestroy：在容器销毁bean之前通知我们进行清理工作
 * 4）、BeanPostProcessor【interface】：bean的后置处理器；
 * 		在bean初始化前后进行一些处理工作；
 * 		postProcessBeforeInitialization:在初始化之前工作
 * 		postProcessAfterInitialization:在初始化之后工作
 * 
 * Spring底层对 BeanPostProcessor 的使用；
 * 		bean赋值，注入其他组件，@Autowired，生命周期注解功能，@Async,xxx BeanPostProcessor;
 * 
 */
@ComponentScan("com.atguigu.bean")
@Configuration
public class MainConfigOfLifeCycle {
	
	//@Scope("prototype")
	@Bean(initMethod="init",destroyMethod="detory")
	public Car car(){
		return new Car();
	}
}
```

Car.java

```java
@Component
public class Car {
   
   public Car(){
      System.out.println("car constructor...");
   }
   
   public void init(){
      System.out.println("car ... init...");
   }
   
   public void detory(){
      System.out.println("car ... detory...");
   }
}
```

</br>

#### InitializingBean和DisposableBean

```java
@Component
public class Cat implements InitializingBean,DisposableBean {
   
   public Cat(){
      System.out.println("cat constructor...");
   }

   @Override
   public void destroy() throws Exception {
      // TODO Auto-generated method stub
      System.out.println("cat...destroy...");
   }

   @Override
   public void afterPropertiesSet() throws Exception {
      // TODO Auto-generated method stub
      System.out.println("cat...afterPropertiesSet...");
   }
}
```

</br>

#### @PostConstruct&@PreDestroy

这个ApplicationContextAware的功能就是为我们注入applicationContext的，但是这个功能是怎么实现的呢？就是ApplicationContextProcessor 做的，它会判断我们的我们的类是否实现了ApplicationContextAware接口，如果是就调用相应的方法给这个类注入值，注入值的方式就是把这个bean强转成ApplicationContextAware，然后调用setApplicationContext方法；

```JAVA
@Component
public class Dog implements ApplicationContextAware {
   
   //@Autowired
   private ApplicationContext applicationContext;
   
   public Dog(){
      System.out.println("dog constructor...");
   }
   
   //对象创建并赋值之后调用
   @PostConstruct
   public void init(){
      System.out.println("Dog....@PostConstruct...");
   }
   
   //容器移除对象之前
   @PreDestroy
   public void detory(){
      System.out.println("Dog....@PreDestroy...");
   }

   @Override
   public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
      // TODO Auto-generated method stub
      this.applicationContext = applicationContext;
   }
}
```

</br>

#### BeanPostProcessor-后置处理器

注意这个就不是实现初始化和销毁方法了，而是在初始化之前和初始化之后做一些工作，需要和上面的区分开；

```java
/**
 * 后置处理器：初始化前后进行处理工作
 * 将后置处理器加入到容器中
 */
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

   @Override
   public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      // TODO Auto-generated method stub
      System.out.println("postProcessBeforeInitialization..."+beanName+"=>"+bean);
      return bean;
   }

   @Override
   public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
      // TODO Auto-generated method stub
      System.out.println("postProcessAfterInitialization..."+beanName+"=>"+bean);
      return bean;
   }
}
```

</br>

#### BeanPostProcessor原理

源码从IOC 容器的 refresh() 开始

遍历得到容器中所有的BeanPostProcessor；挨个执行beforeInitialization，

一但返回null，跳出for循环，不会执行后面的 BeanPostProcessor.postProcessorsBeforeInitialization

先执行 populateBean(beanName, mbd, instanceWrapper);给bean进行属性赋值

然后执行initializeBean方法

这个方法会先调用 applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);

然后调用 invokeInitMethods(beanName, wrappedBean, mbd);执行自定义初始化

最后执行applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
</br>

</br>

#### BeanPostProcessor在Spring底层的使用

ApplicationContextAware 是用ApplicationContextProcessor 实现的；

Dog的 @PostConstruct 和 @PreDestroy 注解可以实现在生命周期中被调用也是用InitDestroyAnnotationBeanPostProcessor 实现的；

@Autowired 注解也是用AutowiredAnnotationBeanPostProcessor 实现的，这么厉害。。。。

</br>

</br>

### 属性赋值

#### @Value赋值

```java
public class Person {
   
   //使用@Value赋值；
   //1、基本数值
   //2、可以写SpEL； #{}
   //3、可以写${}；取出配置文件【properties】中的值（在运行环境变量里面的值）
   
   @Value("张三")
   private String name;
   @Value("#{20-2}")
   private Integer age;
   
   @Value("${person.nickName}")
   private String nickName;

   public String getNickName() {
      return nickName;
   }
   public void setNickName(String nickName) {
      this.nickName = nickName;
   }
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
   }
   @Override
   public String toString() {
      return "Person [name=" + name + ", age=" + age + ", nickName=" + nickName + "]";
   }
}
```

</br>

#### @PropertySource加载外部配置文件

```java
//使用@PropertySource读取外部配置文件中的k/v保存到运行的环境变量中;加载完外部的配置文件以后使用${}取出配置文件的值
@PropertySource(value={"classpath:/person.properties"})
@Configuration
public class MainConfigOfPropertyValues {
   
   @Bean
   public Person person(){
      return new Person();
   }
}
```

</br>

```java
public class IOCTest_PropertyValue {
   AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfigOfPropertyValues.class);
   @Test
   public void test01(){
      printBeans(applicationContext);
      System.out.println("=============");
      
      Person person = (Person) applicationContext.getBean("person");
      System.out.println(person);

      ConfigurableEnvironment environment = applicationContext.getEnvironment();
      //因为配置文件中的值会放在环境变量中，所以可以直接从环境变量中获取出来；
      String property = environment.getProperty("person.nickName");
      System.out.println(property);
      applicationContext.close();
   }
   
   private void printBeans(AnnotationConfigApplicationContext applicationContext){
      String[] definitionNames = applicationContext.getBeanDefinitionNames();
      for (String name : definitionNames) {
         System.out.println(name);
      }
   }
}
```

</br>

</br>

### 自动装配

#### @Autowired&@Qualifier&@Primary

```java
/**
 * 自动装配;
 *        Spring利用依赖注入（DI），完成对IOC容器中中各个组件的依赖关系赋值；
 * 
 * 1）、@Autowired：自动注入：
 *        1）、默认优先按照类型去容器中找对应的组件:applicationContext.getBean(BookDao.class);找到就赋值
 *        2）、如果找到多个相同类型的组件，再将属性的名称作为组件的id去容器中查找
 *                       applicationContext.getBean("bookDao")
 *        3）、@Qualifier("bookDao")：使用@Qualifier指定需要装配的组件的id，而不是使用默认的属性名
 *        4）、自动装配默认一定要将属性赋值好，没有就会报错；
 *           可以使用@Autowired(required=false);
 *        5）、@Primary：让Spring进行自动装配的时候，默认使用首选的bean；
 *              也可以继续使用 @Qualifier 指定需要装配的bean的名字
 *        BookService{
 *           @Autowired
 *           BookDao  bookDao;
 *        }
 * 
 * 2）、Spring还支持使用@Resource(JSR250)和@Inject(JSR330)[java规范的注解]
 *        @Resource:
 *           可以和@Autowired一样实现自动装配功能；默认是按照组件名称进行装配的；
 *           没有能支持@Primary功能，没有支持@Autowired（reqiured=false）;
 *        @Inject:
 *           需要导入javax.inject的包，和Autowired的功能一样。没有required=false的功能；
 *  @Autowired:Spring定义的； @Resource、@Inject都是java规范
 *     
 * AutowiredAnnotationBeanPostProcessor:解析完成自动装配功能；       
 * 
 * 3）、 @Autowired:构造器，参数，方法，属性；都是从容器中获取参数组件的值
 *        1）、[标注在方法位置]：@Bean+方法参数；参数从容器中获取;默认不写@Autowired效果是一样的；都能自动装配
 *        2）、[标在构造器上]：如果组件只有一个有参构造器，这个有参构造器的@Autowired可以省略，参数位置的组件还是可以自动从容器中获取
 *        3）、放在参数位置：
 * 
 * 4）、自定义组件想要使用Spring容器底层的一些组件（ApplicationContext，BeanFactory，xxx）；
 *        自定义组件实现xxxAware；在创建对象的时候，会调用接口规定的方法注入相关组件；Aware；
 *        把Spring底层一些组件注入到自定义的Bean中；
 *        xxxAware：功能使用xxxProcessor；
 *           ApplicationContextAware==》ApplicationContextAwareProcessor；
 *
 */
@Configuration
@ComponentScan({"com.clay.service","com.clay.dao",
   "com.clay.controller","com.clay.bean"})
public class MainConifgOfAutowired {
   
   @Primary
   @Bean("bookDao2")
   public BookDao bookDao(){
      BookDao bookDao = new BookDao();
      bookDao.setLable("2");
      return bookDao;
   }
   
   /**
    * @Bean标注的方法创建对象的时候，方法参数的值从容器中获取
    * @param car
    * @return
    */
   @Bean
   public Color color(Car car){
      Color color = new Color();
      color.setCar(car);
      return color;
   }
}
```

</br>

```java
//名字默认是类名首字母小写
@Repository
public class BookDao {
   
   private String lable = "1";

   public String getLable() {
      return lable;
   }

   public void setLable(String lable) {
      this.lable = lable;
   }

   @Override
   public String toString() {
      return "BookDao [lable=" + lable + "]";
   }
}
```

</br>

```java
@Service
public class BookService {
   
   @Qualifier("bookDao")
   @Autowired(required=false)
   private BookDao bookDao;
   
   public void print(){
      System.out.println(bookDao);
   }

   @Override
   public String toString() {
      return "BookService [bookDao=" + bookDao + "]";
   }
}
```

</br>

```java
public class IOCTest_Autowired {
   
   @Test
   public void test01(){
      AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConifgOfAutowired.class);
      
      BookService bookService = applicationContext.getBean(BookService.class);
      System.out.println(bookService);
      
      //BookDao bean = applicationContext.getBean(BookDao.class);
      //System.out.println(bean);
      
      Boss boss = applicationContext.getBean(Boss.class);
      System.out.println(boss);
      Car car = applicationContext.getBean(Car.class);
      System.out.println(car);
      
      Color color = applicationContext.getBean(Color.class);
      System.out.println(color);
      System.out.println(applicationContext);
      applicationContext.close();
   }
}
```

</br>

#### @Resource&@Inject

```java
@Service
public class BookService {
   
   //@Resource(name="bookDao2")
   @Inject
   private BookDao bookDao;
   
   public void print(){
      System.out.println(bookDao);
   }

   @Override
   public String toString() {
      return "BookService [bookDao=" + bookDao + "]";
   }
}
```

</br>

#### 方法、构造器位置的自动装配

针对 @Autowired 注解

```java
//默认加在ioc容器中的组件，容器启动会调用无参构造器创建对象，再进行初始化赋值等操作
@Component
public class Boss {

   private Car car;

   //构造器要用的组件，都是从容器中获取
   @Autowired  //可以省略
   public Boss(Car car){
      this.car = car;
      System.out.println("Boss...有参构造器");
   }

   public Car getCar() {
      return car;
   }

   //@Autowired 
   //标注在方法，Spring容器创建当前对象，就会调用方法，完成赋值；
   //方法使用的参数，自定义类型的值从ioc容器中获取
   public void setCar(Car car) {
      this.car = car;
   }

   @Override
   public String toString() {
      return "Boss [car=" + car + "]";
   }
}
```

</br>

#### Aware注入Spring底层组件&原理

```java
@Component
public class Red implements ApplicationContextAware,BeanNameAware,EmbeddedValueResolverAware {
   
   private ApplicationContext applicationContext;

   @Override
   public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
      System.out.println("传入的ioc："+applicationContext);
      this.applicationContext = applicationContext;
   }

   @Override
   public void setBeanName(String name) {
      System.out.println("当前bean的名字："+name);
   }

   @Override
   public void setEmbeddedValueResolver(StringValueResolver resolver) {
      String resolveStringValue = resolver.resolveStringValue("你好 ${os.name} 我是 #{20*18}");
      System.out.println("解析的字符串："+resolveStringValue);
   }
}
```

</br>

#### @Profile根据环境注册bean

```java
/**
 * Profile：
 *        Spring为我们提供的可以根据当前环境，动态的激活和切换一系列组件的功能；
 * 
 * 开发环境、测试环境、生产环境；
 * 数据源：(/A)(/B)(/C)；
 *
 * @Profile：指定组件在哪个环境的情况下才能被注册到容器中，不指定，任何环境下都能注册这个组件
 * 
 * 1）、加了环境标识的bean，只有这个环境被激活的时候才能注册到容器中。默认是default环境
 * 2）、写在配置类上，只有是指定的环境的时候，整个配置类里面的所有配置才能开始生效
 * 3）、没有标注环境标识的bean在，任何环境下都是加载的；
 */

@PropertySource("classpath:/dbconfig.properties")
@Configuration
public class MainConfigOfProfile implements EmbeddedValueResolverAware{
   
   @Value("${db.user}")
   private String user;
   
   private StringValueResolver valueResolver;
   
   private String  driverClass;
   
   
   @Bean
   public Yellow yellow(){
      return new Yellow();
   }
   
   @Profile("test")
   @Bean("testDataSource")
   public DataSource dataSourceTest(@Value("${db.password}")String pwd) throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser(user);
      dataSource.setPassword(pwd);
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/test");
      dataSource.setDriverClass(driverClass);
      return dataSource;
   }
   
   
   @Profile("dev")
   @Bean("devDataSource")
   public DataSource dataSourceDev(@Value("${db.password}")String pwd) throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser(user);
      dataSource.setPassword(pwd);
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/ssm_crud");
      dataSource.setDriverClass(driverClass);
      return dataSource;
   }
   
   @Profile("prod")
   @Bean("prodDataSource")
   public DataSource dataSourceProd(@Value("${db.password}")String pwd) throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser(user);
      dataSource.setPassword(pwd);
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/scw_0515");
      
      dataSource.setDriverClass(driverClass);
      return dataSource;
   }

   @Override
   public void setEmbeddedValueResolver(StringValueResolver resolver) {
      this.valueResolver = resolver;
      driverClass = valueResolver.resolveStringValue("${db.driverClass}");
   }
}
```

</br>

```java
public class IOCTest_Profile {
   
   //1、使用命令行动态参数: 在虚拟机参数位置加载 -Dspring.profiles.active=test
   //2、代码的方式激活某种环境；
   @Test
   public void test01(){
      AnnotationConfigApplicationContext applicationContext = 
            new AnnotationConfigApplicationContext();
      //1、创建一个applicationContext
      //2、设置需要激活的环境
      applicationContext.getEnvironment().setActiveProfiles("dev");
      //3、注册主配置类
      applicationContext.register(MainConfigOfProfile.class);
      //4、启动刷新容器
      applicationContext.refresh();

      String[] namesForType = applicationContext.getBeanNamesForType(DataSource.class);
      for (String string : namesForType) {
         System.out.println(string);
      }
      
      Yellow bean = applicationContext.getBean(Yellow.class);
      System.out.println(bean);
      applicationContext.close();
   }
}
```