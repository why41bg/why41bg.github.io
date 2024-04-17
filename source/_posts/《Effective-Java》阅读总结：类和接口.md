---
title: 《Effective-Java》阅读总结：类和接口
date: 2024-04-15 17:20:52
tags:
- 《Effective Java》
- 类和接口的设计指导原则
categories:
- 《Effective Java》
---

> **“尽信书不如无书”**
>
> 此篇博客主要阐述在设计类和接口时的一些指导原则，这些指导原则来自于《Effective Java》第三版的第15条到25条。以下的内容都是我在阅读此书之后，并尝试去结合一些 Java 类库源码得到的实际感受。受限于开发经验，其中的一些内容可能稍有偏差或者不全面，我会不断学习然后加以补充。
>
> 此系列博客会随着我的阅读进度不断完善，也是监督我学习进度的一种方式啦 :)

# 使类和成员的可访问性最小化

## 对封装思想的进一步理解

当谈到面向对象中的封装思想的时候，我相信大多数人和我的理解差不多，简单来说就是将对象的属性和行为封装到一起。但是封装更深层次的含义应该还包括**信息隐藏**，即一个好的设计，对外部来说是看不见其内部那些与如何使用无关的细节的，组件之间只需要知道如何通信足矣。

信息隐藏之所以很重要，我认为主要是因为其保证了组件的安全性和实现了系统的解耦。因为组件内部对外部不可见，这就避免了外部有意或无意地接触到了组件内部的细节，从而导致一些潜在的安全性问题。另外，组件之间因为相互隐藏了细节，联系并不紧密，组件之间并不会相互产生影响。对于大型系统来说，无论是开发阶段还是维护迭代阶段，都大大降低了成本。

## 如何控制可访问性？

使用访问修饰符 `private`，`protected` 和 `public` 来控制类及其成员的可访问性。只需要时刻牢记一个简单的规则：**尽可能地使每个类或者成员不被外界访问**。这也就是说对于一个类的每个实例变量（类的非静态成员属性，后文中的成员变量和实例域也是这个意思）来说，都应该使用 `private` 进行修饰，只有当同一个包下的其他类需要访问其成员变量的时候，才需要将其访问扩大到包级。但是如果经常发生这种情况，就应该去考虑是否有另一种分解方案，能够得到耦合度更低的类。

对“尽可能地使每个类或者成员不被外界访问”这条简单规则进一步的细化是：**公有类的实例域决不能是公有的，对静态域来说同样如此，除非是这个静态域是一个 `final` 修饰的常量（也就是由大写字母组成，下划线隔开的那些类变量）**。对于这条规则，很容易出错的一个场景是当类有一个私有的指向可变对象 final 域，例如数组，当这种域被方法返回时，就违背这一规则。因为数组中的对象是可变的，这也就是意味着外部调用者可以随意修改这个数组当中的内容，即使它尝试将自己隐藏在类中。

修正这个问题有两种方法。

1. 使公有数组变成私有的，并增加一个公有的不可变列表。下面是书中给出的示例代码：
	```java
	private static final Thing[] PRIVATE_VALUES = {...};
	public static final List<Thing> VALUES = Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
	```

	

2. 使数组私有的，并添加一个公有方法，返回私有数组的拷贝
	```java
	private static final Thing[] PRIVATE_VALUES = {...};
	public static final Thing[] values() {
	  return PRIVATE_VALUES.clone();
	}
	```

## 小结

**在仔细设计了一个最小的公有 API 之后，应该防止把任何散乱的类、接口或者成员变成 API 的一部分，除了公有静态 final 域的特殊情形之外（此时它们充当常量），公有类都不应该包含公有域，并且要确保公有静态 final 域所引用的对象都是不可变的。**



# 要在公有类而非公有域中使用访问方法

## 说人话版

公有类永远都不应该暴露可变的域，而暴露不可变的类通常来说也存在一定的问题。其实面向对象也已经告诉我们了，对于公有类的数据域应该设置为私有的，然后为其提供公有的访问方法（getter），而对于可变的数据域，还可以考虑提供公有的修改方法（setter）。

##  鲜活的反例

`java.awt` 包中的 `Point` 类和 `Dimension` 类直接暴露了其数据域，它们是不值得效仿的例子。例如在 JDK1.8 中 `Point` 类源码如下：

```java
public class Point extends Point2D implements java.io.Serializable {

  public int x;

  public int y;
  
  // ...
}
```

`Dimension` 类源码如下：

```java
public class Dimension extends Dimension2D implements java.io.Serializable {

  public int width;

  public int height;
  
  // ...
}
```

在这两个类中，它们的数据域都直接暴露给了外部。这在我们的代码中应该是要被我们所坚决杜绝的！！



# 使可变性最小化

**使可变性最小化简单来说就是在设计一个类的时候，要朝着将其设计为不可变类的方向努力**。因此以下谈论的核心都是如何将类设计成一个不可变类。像 Java 核心类库中的 `String`、`BigInteger`、`BigDecimal` 和基本类型的包装类都是不可变类。努力将类设计成不可变类需要遵循五条准则：

1. 不要提供任何会修改对象状态的方法（setter）
2. 保证类不会被扩展
3. 声明所有的域都是 final 的
4. 声明所有的域都是私有的
5. 确保对于任何可变组件的互斥访问

## 保证类不会被扩展

保证类不被扩展一个最简单的方法就是将类声明为 final 的。例如 `java.lang` 包下的 `String` 类：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence
```

另一个保证类不会被扩展的更加灵活的方式是，**让类的所有构造器都变成私有的或者包级私有的，并添加公有的静态工厂（这里的静态工厂不等同于设计模式中工厂的概念）来代替公有的构造器**。下面是书上提供的一个例子：

```java
// Immutable class with static factories instead of constructors
public class Complex {
  private final double re;
  private final double im;
  
  private Complex(double re, double im) {
    this.re = re;
    this.im = im;
  }
  
  public static Complex valueOf(double re, double im) {
    return new Complex(re, im);
  }
}
```

静态工厂方法通常都有一些惯例名称：`from`、`of`、`valueOf`、`instance`、`getInstance`、`create`、`newInstance`、`getType`、`newType`、`type`。

## 声明所有的域都是私有的

使域私有同时也控制了域的可访问性。一方面可以防止客户端获得访问被域引用的可变对象的权限，防止客户端直接修改它们。另一方面由于对外部隐藏了内部的细节，没有对外部做出承诺，因此可以在以后的版本中根据需要改变内部的表示方法。

## 确保对于任何可变组件的互斥访问

说人话就是如果类具有可变对象的域，则必须确保该类的客户端无法获得指向这些对象的引用。并且，永远不要用客户端提供的对象引用来初始化这样的域（客户端提供的，那客户端必然能够对这个对象进行控制），也不要从任何访问方法中返回**该对象**的引用，但是可以返回一个新的对象（**保护性拷贝技术**）。

下面以一个例子来说明：

```java
// Immutable complex number class
public final class Complex {
  private final doubel re;
  private final double im;
  
  public Complex(double re, double im) {
    this.re = re;
    this.im = im;
  }
  
  public double realPart() {
    return this.re;
  }
  
  public double imaginaryPart() {
    return this.im;
  }
  
  public Complex plus(Complex c) {
    return new Complex(this.re + c.re, this.im + c.im);
  }
  
  public Complex minus(Complex c) {
    return new Complex(this.re - c.re, this.im - c.re);
  }
  
  public Complex times(Complex c) {
    return new Complex(this.re * c.re - this.im * c.im,
                      this.re * c.im + this.im * c.re);
  }
  
	public Complex dividedBy(Complex c) {
    double tmp = c.re * c.re + c.im * c.im;
    return new Complex((this.re * c.re + this.im * c.im) / tmp,
                      (this.im * c.re - this.re * c.im) / tmp);
  }
  
  // ...
}
```

在上面这个例子中可以看到，首先数据域 `re` 和 `im` 都已经被私有化和使用 `final`  进行了声明。另外这个类还使用了 `final` 保证了其无法被扩展。在将多个 `Complex` 类进行算数运算之后，返回的是一个新的实例，而不是修改这个实例。这里有一个小 tips 是：这种模式被称为函数的方法，因为这些方法返回了一个函数的结构，这些函数对操作数进行运算但并不修改它。与之相对应的是更常见的是过程的或者命令式的方法，使用这些方法时，将一个过程作用在它们的操作数上，会导致它的状态发生改变。更细一点的事，函数的方法的名称都是**介词（plus），而不是动词（add）**，这是为了强调该方法不会改变对象的值。Java 核心类库一个反面的例子是 `BigInteger` 类，它并没有遵守这样的命名习惯。

下面是 `BigInteger` 类中的 `add` 方法，但是按照命名习惯来说，应该命名为 `plus`：

```java
public BigInteger add(BigInteger val) {
  if (val.signum == 0)
      return this;
  if (signum == 0)
      return val;
  if (val.signum == signum)
      return new BigInteger(add(mag, val.mag), signum);

  int cmp = compareMagnitude(val);
  if (cmp == 0)
      return ZERO;
  int[] resultMag = (cmp > 0 ? subtract(mag, val.mag)
                     : subtract(val.mag, mag));
  resultMag = trustedStripLeadingZeroInts(resultMag);

  return new BigInteger(resultMag, cmp == signum ? 1 : -1);
}
```

## 不可变类的唯一缺点

不可变类的唯一缺点是，**对于每个不同的值都需要一个单独的对象**。也就是说，不可变对象的频繁创建所带来的额外开销有可能会成为系统的瓶颈。因此我们可以为其提供一个配套类来帮助不可变类完成那些多步骤多操作。在 Java 核心类库中，`String` 类就有它的配套类 `StringBuilder` 和 `StringBuffer`。

## 小结

在设计一个类的时候，要牢记没有方法能够对对象的状态产生外部可见的改变，同时又因为对象的不可变性，可以将一些开销昂贵的计算结果缓存在不可变类的内部的非 final 域中，不可变性保证了这些计算每次都能得到相同的结果。

对外部客户端来说，不可变对象本质上是线程安全的，因此不可变对象可以自由被共享，现有实例应该尽可能被鼓励重用。

**即使一个类不能被做成不可变的，仍然应该尽可能朝着使其可变性最小化的方向努力。除非有令人信服的理由，否则每个域都应该是 `private` 且 `final` 的。**



# 复合优先于继承

## 先谈谈继承

在以前，我一直认为继承是实现代码重用的最主要的途径，无论在什么场景，我都会毫不犹豫地考虑如何利用继承来实现代码重用。但其实只有在两种场景下继承才是相对安全的，一种是在一个包类使用继承，另一种是超类是专门被设计用来继承的。在其他任何场景下，继承都会导致代码的不安全和束缚代码的扩展能力。

跨包继承最主要的风险来源于继承对封装的核心，信息隐藏思想的破坏。具体就体现在子类对父类的方法覆盖上，如果父类中的可被覆盖的方法依赖于**自用性**（self use，即类中方法实现依赖于同一个类中的另一个方法），那么由于信息隐藏，对子类来说并不知道这一细节，在子类中进行方法覆盖的时候就会产生不安全问题。而如果要避免这种不安全问题，子类就需要知道父类中的一切实现细节，这就违背了信息隐藏的思想，另一方面是这些实现细节并不是父类做出的承诺，在未来可能会发生变化。并且，当超类在未来新增方法时，子类也需要进行相应的方法覆盖，否则也会产生不安全问题。

可见，在跨包场景下，使用继承是多么的不合理和不安全。

## 比继承更好的思想

复合的思想是，不扩展现有类，而是在新类中增加一个**私有域，引用现有类的一个实例**，这种设计方法就叫做复合。**新类中的每个实例方法都可以调用被包含的现有类实例中对应的方法，并返回它的结果**，这叫做转发。

通过复合 + 转发，使得新类不依赖于现有类的实现细节，从而也就不会产生不安全问题。下面是书中给出的一个例子，为 HashSet 添加一个功能，记录它自创建以来添加了多少个元素，这个功能的实现分为两部分，类本身和可重用的转发类：

```java
// Wrapper class - uses composition in place of inheritance
public class InstrumentedSet<E> extends ForwardingSet<E> {
  private int addCount = 0;
  
  public InstrumentedSet(Set<E> s) { super(s); }
  
  @Override
  public boolean add(E e) { return super.add(e); }
  
  @Override
  public boolean addAll(Collection<? extends E> c) { return super.addAll(c); }
  
  public int getAddCount() { return this.addCount; }
}
```

```java
// Reusable forwarding class
public class ForwardingSet<E> implements Set<E> {
  private final Set<E> s;
  
  public ForwardingSet(Set<E> s) { this.s = s; }
  
  public void clear() { s.clear(); }
  
  public boolean contains(Object o) { return s.contains(o); }
  
  public int size() { return s.size(); }
  
  public Iterator<E> iterator() { return s.iterator(); }
  
  public boolean add(E e) { return s.add(e); }
  
  public boolean addAll(Collection<? extends E> c) { return s.addAll(c); }
  
  // ...
}
```

本质上来说，这里的包装类可以将一个 `Set` 变为一个拥有计数功能的 `Set`，可以被用来包装任何一个 Set。每一个 `InstrumentedSet` 实例都把另一个 `Set` 实例包装起来了，因此它被称为包装类，这也正是**装饰者（Decorator）模式**。包装类唯一的缺点就是不适合用于回调框架。

## 小结

继承的功能强大，当也存在许多问题，由于继承违背了封装思想，因此在不适当的场景下使用会导致严重的安全问题。复合加转发的机制使用场景比继承宽松，包装类不仅比子类更加健壮，同时功能更加强大。



# 要么设计继承并提供文档说明，要么禁止继承

为了继承而设计的类，会受到一些实质性的限制，需要在文档中具体说明其实现细节，这同时又违背了文档的“说明做了什么，而不是描述如何做到”的目的。

对于那些并非为了安全地进行子类化而设计和编写文档的类，要禁止子类化。要么将类声明为 `final` ，要么将构造器私有化，并提供静态工厂来替代构造器。

总而言之，设计一个专门用来继承的类是一个很辛苦的工作。我从 Java 核心类库的发展历程中感受到的最重要的一点就是：宁愿什么也不做，也不要犯错。



# 接口优于抽象类
