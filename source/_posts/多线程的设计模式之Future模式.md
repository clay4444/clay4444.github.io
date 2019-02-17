---
title: 多线程的设计模式之Future模式
categories:
  - 设计模式
abbrlink: 6d530e97
date: 2017-07-24 13:32:34
---

## Future模式

- 时序图

{% asset_img Future模式设计思想.png %}

- Future模式有点类似于商品订单，比如在网购时，当看中某一件商品时，就可以提交订单，当订单处理完成后，在家里等待商品送货上门即可，或者说更形象的我们发送Ajax请求的时候，页面是异步的进行后台处理，用户无需一直等待请求的结果，可以继续浏览或操作其他内容
- FutureData是对RealData的包装，是对真实数据的一个代理，封装了获取真实数据的等待过程。它们都实现了共同的接口，所以，针对客户端程序组是没有区别的；
- 客户端在调用的方法中，单独启用一个线程来完成真实数据的组织，这对调用客户端的main函数式封闭的；
- 因为在FutureData中的notifyAll和wait函数，主程序会等待组装完成后再会继续主进程，也就是如果没有组装完成，main函数会一直等待。
- 在FutureClient中单独启动一个线程获取RealData，所以是不能直接把它返回的，只能把它作为为一个成员变量赋给FutureData的一个成员变量，然后用FutureData封装获取真实数据的等待过程，通过在FutureData中的一个isReady变量和wait()和notify()方法来控制主线程获取真实数据的过程，即如果没有获取完成，一直等待，如果获取完成了，直接返回
- 实例代码：

```java
public interface Data {

  String getRequest();
}
```

```java
public class FutureClient {

  public Data request(final String queryStr){
    //1 我想要一个代理对象（Data接口的实现类）先返回给发送请求的客户端，告诉他请求已经接收到，可以做其他的事情
    final FutureData futureData = new FutureData();
    //2 启动一个新的线程，去加载真实的数据，传递给这个代理对象
    new Thread(new Runnable() {
      @Override
      public void run() {
        //3 这个新的线程可以去慢慢的加载真实对象，然后传递给代理对象
        RealData realData = new RealData(queryStr);
        futureData.setRealData(realData);
      }
    }).start();

    return futureData;
  }
}
```

~~~~java
public class FutureData implements Data{

  private RealData realData ;

  private boolean isReady = false;

  public synchronized void setRealData(RealData realData) {
    //如果已经装载完毕了，就直接返回
    if(isReady){
      return;
    }
    //如果没装载，进行装载真实对象
    this.realData = realData;
    isReady = true;
    //进行通知
    notify();
  }

  @Override
  public synchronized String getRequest() {
    //如果没装载好 程序就一直处于阻塞状态
    while(!isReady){
      try {
        wait();
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
    //装载好直接获取数据即可
    return this.realData.getRequest();
  }
}
public class RealData implements Data{

  private String result ;

  public RealData (String queryStr){
    System.out.println("根据" + queryStr + "进行查询，这是一个很耗时的操作..");
    try {
      Thread.sleep(5000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println("操作完毕，获取结果");
    result = "查询结果";
  }

  @Override
  public String getRequest() {
    return result;
  }

}
public class Main {

  public static void main(String[] args) throws InterruptedException {

    FutureClient fc = new FutureClient();
    Data data = fc.request("请求参数");
    System.out.println("请求发送成功!");
    System.out.println("做其他的事情...");

    String result = data.getRequest();
    System.out.println(result);

  }
}
~~~~

