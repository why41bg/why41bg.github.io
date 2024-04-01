---
title: Java设计模式之工厂模式
date: 2024-04-01 02:10:25
tags:
- java
- GoF设计模式
categories:
- GoF设计模式
---

# 工厂模式

工厂模式是一种创建型设计模式，提供了一种针对接口，而不是实现的编码方式。工厂模式的核心目的在于将对象的实例化过程与对象的使用解耦，将实际实现类的实例化从应用程序代码中移除。在工厂模式下，客户端代码只需要关心对象的接口，而无需关心具体的实例化过程。这样就实现了解耦，提高了程序的灵活性，可扩展性，可维护性。

下面是一个工厂模式的 demo：

首先提供一个计算机的抽象类，代码如下：

``` java
public abstract class Computer {
  
  public abstract String getCPU();
  
  public abstract String getRAM();
  
  public abstract String getROM();
  
  @Override
  public String toString() {
    return "CPU="+this.getCPU()+" RAM="+this.getRAM()+" ROM="+this.getROM();
  }
}
```

这个抽象类定义了一个计算机的抽象，并且提供了三个抽象方法，来获取计算机的CPU，RAM和ROM。基于这个抽象类，可以定义两个实际的类，个人PC和服务器，下面是这两个子类：

``` java
public class PC extends Computer {
  private String CPU;
  private String RAM;
  private String ROM;
  
  public PC(String cpu, String ram, String rom) {
    this.CPU = cpu;
    this.RAM = ram;
    this.ROM = rom;
  }
  
  @Override
  public String getCPU() {
    return this.CPU;
  }
  
  @Override
  public String getRAM() {
    return this.RAM;
  }
  
  @Override
  public String getROM() {
    return this.ROM;
  }
}
```

``` java
public class Server extends Computer {
  private String CPU;
  private String RAM;
  private String ROM;
  
  public Server(String cpu, String ram, String rom) {
    this.CPU = cpu;
    this.RAM = ram;
    this.ROM = rom;
  }
  
  @Override
  public String getCPU() {
    return this.CPU;
  }
  
  @Override
  public String getRAM() {
    return this.RAM;
  }
  
  @Override
  public String getROM() {
    return this.ROM;
  }
}
```

接下来就是重点了，实现一个工厂类，将对象的创建过程向外界屏蔽，外界从这个工厂类来获取实际实现类的实例化对象，工厂类如下：

``` java
public class ComputerFactory {
  public static Computer getComputer(String type, String cpu, String ram, String rom) {
    if ("PC".equals(type)) return new PC(cpu, ram, rom);
    else if ("Server".equals(type)) return new Server(cpu, ram, rom);
    else return null;
  }
}
```

进一步的修改是，可以对这个工厂类采用单例模式进行设计，比如采用双重锁检验的形式实现工厂单例类。进一步修改如下：

``` java
public class ComputerFactory {
  private static ComputerFactory instance;
  
  private ComputerFactory() {};
  
  public static ComputerFactory getInstance() {
    if (instance == null) {
      synchronized(ComputerFactory.class) {
        if (instance == null) {
          instance = new ComputerFactory();
        }
      }
    }
    return instance;
  }
  
  public Computer getComputer(String type, String cpu, String ram, String rom) {
    if ("PC".equals(type)) return new PC(cpu, ram, rom);
    else if ("Server".equals(type)) return new Server(cpu, ram, rom);
    else return null;
  }
  
}
```

又或者说可以使用一个静态内部辅助类来实现工厂单例类，实现如下：

``` java
public class ComputerFactory {
  
  private ComputerFactory() {}
  
  private static class ComputerFactoryHelper {
    private static ComputerFactory INSTANCE = new ComputerFactory();
  }
  
  public static getInstance() {
    return ComputerFactoryHelper.INSTANCE;
  }
  
  public Computer getComputer(String type, String cpu, String ram, String rom) {
    if ("PC".equals(type)) return new PC(cpu, ram, rom);
    else if ("Server".equals(type)) return new Server(cpu, ram, rom);
    else return null;
  }
  
}
```

# 总结

这篇博客写于处女面被pdd击穿第二天，真是百感交集。。还是想不通为什么我校招和别人社招的问题一模一样，真是哎哎了。
