---
title: 关于HttpSession的线程安全问题
categories:
  - 多线程编程
tags:
  - 并发编程实战读书笔记
abbrlink: c2c02cf
date: 2018-02-25 11:02:31
---

### 背景

最近在读《Java并发编程实战》，其中1.4节提到了Java web的线程安全问题时有如下一段话：

<br/>

即使你可以确保每次只有一个线程调用某个Servlet，但在构建网页应用程序时仍然必须注意线程安全性。Servlet 通常会访问与其他Servlet 共享的信息，例如应用程序中的对象〈这些对象保存在ServletContext 中）或者会话中的对象〈这些对象保存在每个客户端的HttpSession中）。当一个Servlet 访问可在多个Servlet 或者请求中共享的对象时，必续正确地协同这些对象的访问，因为多个请求可能在不同的线程中同时访问这些对象。Servlet 和JSP ，以及在ServletContext 和HttpSession 等容器中保存的Servlet 过滤器和对象等，都必须是线程安全的。

<br/>

Servlet, JSP, Servlet filter 以及保存在 ServletContext、HttpSession 中的对象必须是线程安全的。含义有两点：

- Servlet, JSP, Servlet filter 必须是线程安全的(JSP的本质其实就是servlet)；
- 保存在ServletContext、HttpSession中的对象必须是线程安全的；

<br/>

servlet和servelt filter必须是线程安全的，这个一般是不存在什么问题的，只要我们的servlet和servlet filter中没有实例属性或者实例属性是”不可变对象“就基本没有问题。但是保存在ServletContext和HttpSession中的对象必须是线程安全的，这一点似乎一直被我们忽略掉了。在Java web项目中，我们经常要将一个登录的用户保存在HttpSession中，而这个User对象就是像下面定义的一样的一个Java bean:

~~~java
public class User {
  private int id;
  private String userName;
  private String password;
  // ... ...

  public int getId() {
    return id;
  }
  public void setId(int id) {
    this.id = id;
  }
  public String getUserName() {
    return userName;
  }
  public void setUserName(String userName) {
    this.userName = userName;
  }
  public String getPassword() {
    return password;
  }
  public void setPassword(String password) {
    this.password = password;
  }
}
~~~

<br/>

### 源码分析

下面分析一下为什么将一个这样的Java对象保存在HttpSession中是有问题的，至少在线程安全方面不严谨的，可能会出现并发问题。

Tomcat8.0中HttpSession的源码在org.apache.catalina.session.StandardSession.java文件中，源码如下(截取我们需要的部分)：

~~~java
public class StandardSession implements HttpSession, Session, Serializable {
  // ----------------------------------------------------- Instance Variables
  /**
     * The collection of user data attributes associated with this Session.
     */
  protected Map<String, Object> attributes = new ConcurrentHashMap<>();

  /**
     * Return the object bound with the specified name in this session, or
     * <code>null</code> if no object is bound with that name.
     *
     * @param name Name of the attribute to be returned
     *
     * @exception IllegalStateException if this method is called on an
     *  invalidated session
     */
  @Override
  public Object  getAttribute(String name) {

    if (!isValidInternal())
      throw new IllegalStateException
      (sm.getString("standardSession.getAttribute.ise"));

    if (name == null) return null;

    return (attributes.get(name));

  }

  /**
     * Bind an object to this session, using the specified name.  If an object
     * of the same name is already bound to this session, the object is
     * replaced.
     * <p>
     * After this method executes, and if the object implements
     * <code>HttpSessionBindingListener</code>, the container calls
     * <code>valueBound()</code> on the object.
     *
     * @param name Name to which the object is bound, cannot be null
     * @param value Object to be bound, cannot be null
     * @param notify whether to notify session listeners
     * @exception IllegalArgumentException if an attempt is made to add a
     *  non-serializable object in an environment marked distributable.
     * @exception IllegalStateException if this method is called on an
     *  invalidated session
     */

  public void setAttribute(String name, Object value, boolean notify) {

    // Name cannot be null
    if (name == null)
      throw new IllegalArgumentException
      (sm.getString("standardSession.setAttribute.namenull"));

    // Null value is the same as removeAttribute()
    if (value == null) {
      removeAttribute(name);
      return;
    }
    // ... ...
    // Replace or add this attribute
    Object unbound = attributes.put(name, value);
    // ... ...
  }
  /**
     * Release all object references, and initialize instance variables, in
     * preparation for reuse of this object.
     */
  @Override
  public void recycle() {
    // Reset the instance variables associated with this Session
    attributes.clear();
    // ... ...
  }
  /**
     * Write a serialized version of this session object to the specified
     * object output stream.
     * <p>
     * <b>IMPLEMENTATION NOTE</b>:  The owning Manager will not be stored
     * in the serialized representation of this Session.  After calling
     * <code>readObject()</code>, you must set the associated Manager
     * explicitly.
     * <p>
     * <b>IMPLEMENTATION NOTE</b>:  Any attribute that is not Serializable
     * will be unbound from the session, with appropriate actions if it
     * implements HttpSessionBindingListener.  If you do not want any such
     * attributes, be sure the <code>distributable</code> property of the
     * associated Manager is set to <code>true</code>.
     *
     * @param stream The output stream to write to
     *
     * @exception IOException if an input/output error occurs
     */
  protected void doWriteObject(ObjectOutputStream stream) throws IOException {
    // ... ...
    // Accumulate the names of serializable and non-serializable attributes
    String keys[] = keys();
    ArrayList<String> saveNames = new ArrayList<>();
    ArrayList<Object> saveValues = new ArrayList<>();
    for (int i = 0; i < keys.length; i++) {
      Object value = attributes.get(keys[i]);
      if (value == null)
        continue;
      else if ( (value instanceof Serializable)
               && (!exclude(keys[i]) )) {
        saveNames.add(keys[i]);
        saveValues.add(value);
      } else {
        removeAttributeInternal(keys[i], true);
      }
    }

    // Serialize the attribute count and the Serializable attributes
    int n = saveNames.size();
    stream.writeObject(Integer.valueOf(n));
    for (int i = 0; i < n; i++) {
      stream.writeObject(saveNames.get(i));
      try {
        stream.writeObject(saveValues.get(i));  
        // ... ...            
      } catch (NotSerializableException e) {
        // ... ...               
      }
    }
  }
}
~~~

我们看到每一个独立的HttpSession中保存的所有属性，是存储在一个独立的ConcurrentHashMap中的：

~~~
protected Map<String, Object> attributes = new ConcurrentHashMap<>();
~~~

所以我可以看到 HttpSession.getAttribute()， HttpSession.setAttribute() 等等方法就都是线程安全的。

<br/>

另外如果我们要将一个对象保存在HttpSession中时，那么该对象应该是可序列化的。不然在进行HttpSession的持久化时，就会被抛弃了，无法恢复了：

~~~java
else if ( (value instanceof Serializable)
         && (!exclude(keys[i]) )) {
  saveNames.add(keys[i]);
  saveValues.add(value);
} else {
  removeAttributeInternal(keys[i], true);
}
~~~

所以从源码的分析，我们得出了下面的结论：

- HttpSession.getAttribute()， HttpSession.setAttribute() 等等方法都是线程安全的；
- 要保存在HttpSession中对象应该是序列化的；

<br/>虽然getAttribute，setAttribute是线程安全的了，那么下面的代码就是线程安全的吗？

~~~java
session.setAttribute("user", user);
User user = (User)session.getAttribute("user", user);
~~~

不是线程安全的！因为User对象不是线程安全的，假如有一个线程执行下面的操作：

~~~~java
User user = (User)session.getAttribute("user", user);
user.setName("xxx");
~~~~

<br/>

那么显然就会存在并发问题。因为会出现：有多个线程访问同一个对象 user, 并且至少有一个线程在修改该对象。但是在通常情况下，我们的Java web程序都是这么写的，为什么又没有出现问题呢？原因是：在web中 ”多个线程访问同一个对象 user, 并且至少有一个线程在修改该对象“ 这样的情况极少出现；因为我们使用HttpSession的目的是在内存中暂时保存信息，便于快速访问，所以我们一般不会进行下面的操作：

~~~java
User user = (User)session.getAttribute("user", user);
user.setName("xxx");
~~~

我们一般是只使用对从HttpSession中的对象使用get方法来获得信息，一般不会对”从HttpSession中获得的对象“调用set方法来修改它；而是直接调用 setAttribute来进行设置或者替换成一个新的。

<br/>

### 结论

所以结论是：如果你能保证不会对”从HttpSession中获得的对象“调用set方法来修改它，那么保存在HttpSession中的对象可以不是线程安全的(因为他是”事实不可变对象“，并且ConcurrentHashMap保证了它是被”安全发布的“)；但是如果你不能保证这一点，那么你必须要实现”保存在HttpSession中的对象必须是线程安全“。不然的话，就存在并发问题。

<br/>

使Java bean线程安全的最简单方法，就是在所有的get/set方法都加上synchronized。

