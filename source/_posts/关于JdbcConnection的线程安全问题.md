---
title: 关于JdbcConnection的线程安全问题
categories:
  - 多线程编程
tags:
  - 并发编程实战读书笔记
abbrlink: 92f9baae
date: 2018-02-25 11:36:50
---



### 背景

最近在读《Java并发编程实战》，书中曾多次讨论Servlet，HttpSession，ServletContext，RMI，JdbcConnection的线程安全问题，无奈本人愚钝，始终有些疑惑，所以多方查找资料下，终于明白了一些。

<br/>

### StackOverFlow上的问题说起

 Is java.sql.Connection thread safe?

To rephrase the question: should I avoid sharing instances of classes which implement `java.sql.Connection` between different threads?

我应该避免共享在不同线程之间实现java.sql.connection的类的实例吗？

<br/>

高票的回答：

If the JDBC driver is spec-compliant, then technically yes, the object is thread-safe, but you should avoid sharing connections between threads, since the activity on the connection will mean that only one thread will be able to do anything at a time.

如果jdbc驱动程序符合规范，那么从技术上讲，对象是线程安全的，但是您应该避免在线程之间共享连接，因为连接上的活动意味着一次只有一个线程可以执行任何操作。

<br/>

You should use a connection pool (like [Apache Commons DBCP](https://commons.apache.org/proper/commons-dbcp/)) to ensure that each thread gets its own connection.

你应该使用连接池（如apache commons dbcp）来确保每个线程都获得自己的连接。

<br/>

### 疑问一：Connection实例是线程安全的吗?

即一个connection实例,在多线程环境中是否可以确保数据操作是安全的?

~~~java
private static Connection connection;  
~~~

​    上述代码,设计会不会有问题? 一个Connection实例,即对应底层一个TCP链接,有些开发者可能考虑到"性能",就将代码写成上述样式,最终一个application中所有的DB操作,使用一个connection.确实减少了与DB的TCP链接个数,但是这真的安全吗??真的提升了性能了吗??

<br/>

似乎一个初学者都不会这些写,因为没有sample是这么展示的.但是往往会在每个方法调用时都新建connection,然后close();

当一个开发者考虑到性能的时候,可能会写出上面的代码..

<br/>

​    Connection不是线程安全的,它在多线程环境中使用时,会导致数据操作的错乱,特别是有事务的情况 ，connection.commit()方法就是提交事务,你可以想象,在多线程环境中,线程A开启了事务,然后线程B却意外的commit,这该是个多么纠结的情况.

<br/>

 如果在没有事务的情况下,且仅仅是简单的SQL Select操作,似乎在不会出现数据错乱,一切看起来比较安全.但是这并非意味着提升了性能,JDBC的各种实现中,connection的各种操作都进行了同步:

~~~java
##PreparedStatement  
public int executeUpdate(){  
    synchronized(connection){  
        //do      
    }  
}  
~~~

同时只有当DB操作完成之后,方法调用才会返回. 尽管connection实例的个数唯一,但是在并发环境中,connection实例上锁竞争是必然的,反而没有提升性能(并发量).通常情况下,一个Connection实例只会在一个线程中使用,使用完之后close().

<br/>

  TCP链接的创建开支是昂贵的,当然DB server所能承载的TCP并发连接数也是有限制的.因此每次调用都创建一个Connection,这是不现实的;所以才有了数据库连接池的出现.

<br/>

​    数据库连接池中保持了一定数量的connection实例,当需要DB操作的时候"borrow"一个出来,使用结束之后"return"到连接池中,多线程环境中,连接池中的connection实例交替性的被多个线程使用.

<br/>



### 疑问二：在使用dataSourcePool的情况下,一个线程中所有的DB操作使用的是同一个connection吗?

 比如线程A,依次调用了2个方法,每个方法都进行了一次select操作,那么这两个select操作是使用同一个connection吗?

<br/>

​    第一感觉就是: dataSourcePool本身是否使用了threadLocal来保存线程与connection实例的引用关系;如果使用了threadLocal,那么一个线程多次从pool中获取是同一个connection,直到线程消亡或者调用向pool归还资源..

<br/>

如果在spring环境中(或者其他ORM框架中),这个问题需要分2种情况:事务与非事务.

<br/>

 在**非事务场景下（同时也没有使用spring的事务管理器）**,一切都很简单,每一次调用,都是从pool中取出一个connection实例,调用完毕之后归还资源,因此多次调用,应该是不同的connection实例.

~~~java
public List<Object> select(String sql){  
  Connection connection = pool.getConnection();  
  //do  
  pool.release(connection);//归还资源  
}  
~~~

<br/>

​    在**使用事务**的场景下（或者使用spring事务管理器：TransactionManager，AOP等）,情况就有所不同,开启一个新事务的同时,就会冲pool中获取一个connection实例,并将transaction和connection互为绑定,即此transaction中只会使用此connection,此connection此时只会在一个transaction中使用;因此,在此事务中,无论操作了多少次DB,事实上只会是一个connection实例,直到事务提交或者回滚,当事务提交或者回滚时,将会解除transaction与connection的绑定关系,同时将connection归还到pool中.

~~~java
//开启事务  
public TransactionHolder transaction(){  
  Connection connection = pool.getConnection();  
  connection.setAutoCommit(false);  
  return new TransactionHolder(connection);  
}  

//执行sql  
public boolean insert(String sql,TransactionHolder holder){  
  Connection connection = holder.getConnection();  
  //do  
  try{  
    //doInsert  
    return true;  
  }catch(Exception e){  
    holder.setRollback(true);  
  }  
  return false;  
}  

//提交事务  
public void commit(TransactionHolder holder){  
  Connection connection = holder.getConnection();  
  connection.commit();  
  holder.unbind();//解除绑定  
  pool.release(connection);//归还资源   
}  
~~~

