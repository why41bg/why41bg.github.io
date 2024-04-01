---
title: Java设计模式之单例模式
date: 2024-04-01 00:05:23
tags:
- java
- GoF设计模式
categories:
- GoF设计模式
---

# 单例模式

单例模式如同它的名字一样，限制了采用单例模式的类在 JVM 中只能有一个实例对象存在。常用于日志，驱动对象，缓存和线程池。单例模式属于 GoF 中提出的创建型，结构型和行为型中的创建型。

单例类需要满足三个要求，分别是：

1. 私有化构造方法，防止其他类实例化单例类；

2. 单例类的私有静态变量，是该类的唯一实例；

3. 提供一个公共的静态方法来返回该类的实例，这是外部获取单例类实例的全局访问点。

下面逐一说明单例模式的8种实现。

### 一. 饿汉式初始化

使用 Eager 实现，单例类的实例在类加载的时候就初始化了（类变量，与类加载的知识有关）。这种方式的缺点是，即使应用程序没有使用它，它也会被创建。下面是 demo：

```java
public class EagerInitSingleton {
  private static EagerInitSingleton instance = new EagerInitSingleton();

  private EagerInitSingleton() {}

  public static EagerInitSingleton getInstance() {
    return instance;
  }
}
```

在 Eager 这种实现方式中，单例类的私有静态变量就是该类的唯一实例，并且在类加载的时候就初始化了。构造方法也被私有化了，防止其他类实例化单例类。最后暴露了一个接口，向外部提供单例类的实例化对象。

### 二. 静态块初始化

静态块初始化和饿汉式初始化很类似，区别在于在静态块初始化中，实例对象是在静态代码块中创建的，因此可以提供额外的异常处理。下面是 demo：

```java
public class StaticBlockInitSingleton {
  private static StaticBlockInitSingleton instance;

  private StaticBlockInitSingleton() {}

  static {
    try {
      instance = new StaticBlockInitSingleton();
    } catch(Exception e) {
      throw new RuntimeException("异常抛出。。");
    }
  }

  public static StaticBlockInitSingleton getInstance() {
    return instance;
  }
}
```

### 三. 懒汉式初始化

懒汉式初始化的思想是延迟初始化，在外部请求获取实例对象的时候才去创建，而不是在类加载的时候就创建。因此，实例化对象的操作要放到外部访问点中去，下面是 demo：

```java
public class LazyInitSingleton {
  private static LazyInitSingleton instance;

  private LazyInitSingleton() {}

  public static LazyInitSingleton getInstance() {
    if (instance != null) {
      instance = new LazyInitSingleton();
    }     
  }
}
```

这个 demo 只适用于单线程，在多线程环境下会出现并发问题。设想这么一个场景，线程 A 在判断实例对象不存在之后，进入了 if 代码块，还没来得及为静态变量赋实例类，这时发生了线程调度，线程 B 上处理机也发现实例对象不存在，然后创建了实例对象并返回，回到线程 A，线程 A 会继续实例化对象，然后返回。这个时候 A 和 B 获取到的就是两个不同的实例对象。

### 四. 基于同步锁保证线程安全的懒汉式初始化

创建线程安全的单例类最简单的方法其实就是为全局访问点加上一个同步锁，来限制一次只能有一个线程来尝试实例化对象。下面是 demo：

``` java
public class ThreadSafeLazyInitSingleton {
  private static ThreadSafeLazyInitSingleton instance;
  
  private ThreadSafeLazyInitSingleton() {}
  
  public static synchronized ThreadSafeLazyInitSingleton getInstance() {
    if (instance == null) {
      instance = new ThreadSafeLazyInitSingleton();
    }
    return instance;
  }
}
```

这种基于同步锁来保证线程安全的方法会降低一定的性能，可优化的点就在于不管实例化对象是否存在，线程都需要互斥地去尝试获取实例对象，而如果实例化对象已经存在，线程去获取实例化对象是可以不用互斥进行的。因此，可以缩小锁的范围，只有实例化对象不存在的时候才加锁，这样也就保证了在实例对象不存在的情况下，只能有一个线程去初始化实例对象。而实例化对象如果存在，那么线程去获取实例化对象就无需互斥进行。

``` java
public class ThreadSafeLazyInitSingleton {
  private static ThreadSafeLazyInitSingleton instance;
  
  private ThreadSafeLazyInitSingleton() {}
  
  public static ThreadSafeLazyInitSingleton getInstance() {
    if (instance == null) {
      synchronized(ThreadSafeLazyInitSingleton.class) {
        if (instance == null) {
          instance = new ThreadSafeLazyInitSingleton();
        }
      }
    }
    return instance;
  }
}
```

第一次检验是为了保证在实例对象不存在的情况下，才让线程去互斥的尝试初始化实例对象。第二次检验是为了防止一个线程 A 在通过第一次检验之后，发生了线程调度，另一个线程 B 也能通过第一次检验并初始化对象，A 重回处理机后又去覆盖了实例化对象，导致 A 和 B 拿到的不是同一个对象。

### 五. Bill Pugh 实现

Bill Pugh 是一位计算机科学家，他提出使用静态内部辅助类来实现单例模式，即将初始化对象的工作交给一个静态的内部类。这是目前使用最广泛的一种实现单例模式的方式，因为它不需要同步。下面是 demo：

``` java
public class BillPughSingleton {
  private BillPughSingleton() {}
  
  private static class SingletonHelper {
    private static final BillPughSingleton INSTANCE = new BillPughSingleton();
  }
  
  public static BillPughSingleton getInstance() {
    return SingletonHelper.INSTANCE;
  }
}
```

需要注意的是：这个静态内部类不会睡着单例类的加载而加载，而是在 getInstance 这个方法调用的时候才会被加载到内存。

### 六. 反射破坏破坏单例模式

反射破坏单例模式的原因在于：单例模式要求私有化构造函数，而通过反射可以获取私有化的构造函数。下面是反射破坏单例模式的一个 demo：

``` java
import java.lang.reflect.Constructor;

public class ReflectionSingletonTest {

    public static void main(String[] args) {
        EagerInitializedSingleton instanceOne = EagerInitializedSingleton.getInstance();
        EagerInitializedSingleton instanceTwo = null;
        try {
            Constructor[] constructors = EagerInitializedSingleton.class.getDeclaredConstructors();
            for (Constructor constructor : constructors) {
                // This code will destroy the singleton pattern
                constructor.setAccessible(true);
                instanceTwo = (EagerInitializedSingleton) constructor.newInstance();
                break;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println(instanceOne.hashCode());
        System.out.println(instanceTwo.hashCode());
    }

}
```

博主对于反射不太了解，因此这里这个 demo 来源于[这里](https://www.digitalocean.com/community/tutorials/java-singleton-design-pattern-best-practices-examples#6-using-reflection-to-destroy-singleton-pattern)。反射知识待补充！！

### 七. 枚举实现单例模式

为了防止反射对于单例模式的破坏，可以使用枚举来实现单例模式。原因在于 Java 会确保在程序中枚举类只会被实例化一次。枚举实现的缺点在于不够灵活，不支持懒加载。下面是 demo：

``` java
public enum EnumSingleton {
  INSTANCE;
  
  public static void getInstance() {
    // something else
  }
}
```

### 八. 序列化与单例模式

有时候在分布式系统中，需要对一个单例类实现序列化，以便将其存储在文件系统中。但是在反序列化的时候会创建一个新的实例对象，这样就破坏了单例模式。针对这个问题的解决方案，只需要实现一个 readResolve 方法，在这个方法当中调用全局接入点并返回。

# 总结

此篇博客写于2024年4月1日凌晨1.44，仅以此纪念被某多多击穿的处女面。。
