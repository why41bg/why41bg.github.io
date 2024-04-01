---
title: Java设计模式之策略模式
date: 2024-04-01 11:56:49
tags:
- java
- GoF设计模式
categories:
- GoF设计模式
---

# 策略模式

## 思想

策略设计模式是行为型设计模式之一。当我们为特定任务设计了多种算法，让客户端在运行时决定实际使用的算法时，就可以使用策略模式，让客户端将使用的算法作为参数传递。策略模式的一个最直观的示例就是 **Collections.sort()** 方法，这个方法接收一个 **Comparator** 参数，根据这个接口具体的实现来在运行时决定实际采用的排序算法。

策略模式和状态模式非常相似，它们之间的主要区别在于：在状态模式下，上下文中使用了变量来存储状态，外部的任务实现依靠这些内部状态。而在策略模式下，上下文中并不存储策略，而是将策略作为参数，由外部调用者在运行时传入。

## Demo

接下来采用一个购物车的 demo 来演示策略模式的具体应用。假设有一个购物车，购物车中有多个商品，还提供了一个支付的功能，用于一次性支付购物车中的商品，在支付的时候可以采用两种支付方式，信用卡支付或者 paypal 支付，具体使用哪种支付方式由用户决定。

因此，首先定义一个支付策略的接口，所有的支付实现类都需要实现这个接口，接口如下：

`IPayMethod`

``` java
public interface IPayMethod {
  public boolean pay(int amount);
}
```

接下来实现两种支付方式对应的类，分别是 paypal 支付和信用卡：

`PaypalStrategy`

``` java
public class PaypalStrategy implements IPayMethod {
  private String emailId;
  private String passwd;
  
  public PaypalStrategy(String emailId, String passwd) {
    this.emailId = emailId;
    this.passwd = passwd;
  }
  
  @Override
  public boolean pay(int amount) {
    System.out.println(amount + " paid, according to Paypal.");
    return true;
  }
}
```

`CreditCardStrategy`

``` java
public class CreditCardStrategy implements IPayMethod {
  private String cardId;
  private String passwd;
  // something else
  
  public CreditCardStrategy(String cardId, String passwd) {
    this.cardId = cardId;
    this.passwd = passwd;
  }
  
  @Override
  public boolean pay(int amount) {
    System.out.println(amount + " paid, according to CreditCard.");
    return true;
  }
}
```

OK，我们现在实现了两个类分别代表两种支付方式，接下来实现商品与购物车。首先是商品：

`Item`

``` java
public class Item {
  private String upcCode;  // 商品条形码
  private Integer price;  // 商品单价，单位：分
  
  public Item(String code, int price) {
    this.upcCode = code;
    this.price = price;
  }
  
  public String getUpcCode() {
    return this.upcCode;
  }
  
  public int getPrice() {
    return this.price;
  }
}
```

接下来是购物车类，在购物车类需要提供一个方法用于支付整个购物车，依据策略模式的思想，具体的支付方式应该作为参数传入，因此：

`ShoppingCart`

``` java
public class ShoppingCart {
  private List<Item> items;  // 以一个动态数组来存储购物车中的商品
  
  public ShoppingCart() {
    this.items = new ArrayList<Item>();
  }
  
  public int getTotalPrice() {
    int sum = 0;
    for (Item item : items) {
      sum += item.getPrice();
    }
    return sum;
  }
  
  public boolean pay(IPayMethod payStrategy) {
    return payStrategy.pay(this.getTotalPrice());
  }
}
```

## 总结

**策略模式的核心体现就在于购物车的 `pay` 方法中，具体的支付方式作为参数传递，并且没有以变量的形式存储在类上下文中，这也是策略模式与状态模式的主要区别。**
