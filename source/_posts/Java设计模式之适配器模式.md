---
title: Java设计模式之适配器模式
date: 2024-04-01 15:38:20
tags:
- java
- GoF设计模式
categories:
- GoF设计模式
---

# 适配器模式

## 思想

适配器模式是一种结构型设计模式，主要是为了使两个原本不能一起工作的接口能够一起使用。一个最直观的例子就是手机电源适配器，它能将外部提供的电压转换为手机充电时的工作电压。适配器模式就是基于这种思想，实现方案有两种：一种是基于继承的类适配器，另一种是对象适配器。

## Demo

假设有一个 `Socket` 对象，能够产生一定的电压 `Volt`，电压默认为 120 V，现在使用适配器模式对这个 `Socket` 进行修改，使其能够根据外部需求进行相应的适配。

`Volt`

```java
public class Volt {
  private int defaultVoltNum;
  
  public Volt(int vn) {
    this.defaultVoltNum = vn;
  }
  
  public int getVoltNum() {
    return this.defaultVoltNum;
  }
  
  public boolean setVoltNum(int vn) {
    this.defaultVoltNum = vn;
    return true;
  }
}
```

`Socket`

``` java
public class Socket {
  public Socket() {}
  
  public Volt getVolt() {
    return new Volt(120);
  }
}
```

到目前为止，这个 `Socket` 只能产生 120 V 的 `Volt` ，下面使用适配器模式对其进行改造，使其能够根据外部需求产生不同值的 `Volt`。因此，首先还需要一个适配器规则接口。

`SocketAdapter`

``` java
public interface SocketAdapter {
  public Volt get120Volt();
  
  public Volt get40Volt();
  
  public Volt get3Volt();
}
```



### 类适配器

类适配器的思想是基于继承对原始类进行修改，如下：

`SocketClassAdapterImpl`

``` java
public class SocketClassAdapterImpl extends Socket implements SocketAdapter{
  @Override
  public Volt get120Volt() {
    return getVolt();
  }
  
  @Override
  public Volt get40Volt() {
    Volt v = getVolt();  // 120
    return convertVolt(v, 3);
  }
  
  @Override
  public Volt get3Volt() {
    Volt v = getVolt();  // 120
    return convertVolt(v, 40);
  }
  
  private Volt convertVolt(Volt v, int num) {
    return new Volt(v.getVolt() / num);  // 120 / num
  }
}
```

在这个类适配器中，每次方法调用底层都是通过一个 `convertVolt` 函数创建了一个 `Volt` 对象，然后返回的这个新的对象。类适配器简单来说就是对被适配类使用继承进行扩展，从而使得内部更加灵活。



### 对象适配器

对象适配器模式的思想是在适配器类中创建一个被适配类，然后对这个被适配类进行适配。

`SocketObjectAdapterImpl`

``` java
public class SocketObjectAdapterImpl {
  private Socket socket = new Socket();
  
  public SocketObjectAdapterImpl() {}
  
  @Override
  public Volt get120Volt() {
    return socket.getVolt();
  }
  
  @Override
  public Volt get40Volt() {
    Volt v = socket.getVolt();  // 120
    return convertVolt(v, 3);
  }
  
  @Override
  public Volt get3Volt() {
    Volt v = socket.getVolt();  // 120
    return convertVolt(v, 40);
  }
  
  private Volt convertVolt(Volt v, int num) {
    return new Volt(v.getVolt() / num);  // 120 / num
  }
}
```



## 总结

适配器模式主要用于解决两个接口不兼容的问题，解决思想主要是对被适配类进行相应的修改，实现方式有两种：**一种是基于继承的类适配器，另一种是对被适配类实例进行修改的对象适配器**。

