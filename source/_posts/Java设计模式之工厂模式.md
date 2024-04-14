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

## 设计思想

工厂模式是一种创建型设计模式，提供了一种针对接口，而不是实现的编码方式。工厂模式的核心目的在于将对象的实例化过程与对象的使用解耦，将实际实现类的实例化从应用程序代码中移除。在工厂模式下，客户端代码只需要关心对象的接口，而无需关心具体的实例化过程，从而实现了解耦。

工厂模式通常的实现方式是通过提供一个创建对象的接口，让客户端在运行的过程中自己决定实例化哪一个工厂类（在工厂中创建并返回的类），工厂模式使工厂类的创建延迟到了客户端运行时。

## Demo

Demo 的背景是模拟各种奖品的发放，这里暂时有三种奖品，分别是优惠券，实物商品和第三方爱奇艺会员兑换卡。如果不设计一个接口来统一这些奖品的发放，那么就有可能因为程序员的不同而导致出现各种各样的商品接口，例如下面这张表：

| 序号 | 类型                   | 接口                                                         |
| ---- | ---------------------- | ------------------------------------------------------------ |
| 1    | 优惠券                 | `CouponResult sendCoupon(String uId, String couponNumber, String uuid)` |
| 2    | 实物商品               | `Boolean deliverGoods(DeliverReq req)`                       |
| 3    | 第三方爱奇艺会员兑换卡 | `void grantToken(String bindMobileNumber, String cardId)`    |

如果想要发放奖品，那么就需要去了解这些接口的入参和出参，如果有很多奖品，那么这会导致很高的维护成本。一个很好的解决方案就是定义一个统一的发放奖品的接口（接口隔离），所有的奖品⽆论是实物、虚拟还是第三⽅，都需要通过我们的程序实现此接⼝进⾏处理，以保证最终⼊参出参的统⼀性。定义的接口如下：

`ICommodity`

``` java
public interface ICommodity {

    void sendCommodity(String uId, String commodityId, String bizId, Map<String, String> extMap) throws Exception;

}
```

下面就分别实现三种奖品的逻辑，这三种奖品都需要实现这个上面这个接口：

`CardCommodityService`

```java
public class CardCommodityService implements ICommodity {
  
  public void sendCommodity(String uId, String commodityId, String bizId, Map<String, String> extMap) throws Exception {
    // 发放爱奇艺会员兑换卡的业务逻辑
  }
}
```

`GoodsCommodityService`

```java
public class GoodsCommodityService implements ICommodity {
  
  public void sendCommodity(String uId, String CommodityId, String bizId, Map<String, String> extMap) throws Exception {
    // 发放实物商品的业务逻辑
  }
}
```

`CouponCommodityService`

```java
public class CouponCommodityService implements ICommodity {
  
  public void sendCommodity(String uId, String CommodityId, String bizId, Map<String, String> extMap) throws Exception {
    // 发放优惠券的业务逻辑
  }
}
```

最后再创建一个工厂，向外提供工厂类，后续其他奖品的增加只需要扩展这个工厂，无需修改已有的业务逻辑，满足开闭原则（对扩展开放，对修改封闭）。工厂的业务逻辑如下：

`StoreFactory`

```java
public class StoreFactory {

  public ICommodity getCommodityService(Integer commodityType) {
    if (null == commodityType) return null;
    if (1 == commodityType) return new CouponCommodityService();
    if (2 == commodityType) return new GoodsCommodityService();
    if (3 == commodityType) return new CardCommodityService();
    // 其他的奖品发放逻辑可在这里继续扩展
    throw new RuntimeException("不存在的商品服务类型");
  }

}
```



# 总结

工厂模式的**优点**如下：

1. 避免创建者与具体的产品逻辑耦合；
2. 满⾜单⼀职责，每⼀个业务逻辑实现都在所属⾃⼰的类中完成；
3. 满⾜开闭原则，⽆需更改使⽤调⽤⽅就可以在程序中引入新的产品类型。

**缺点**也很明显，如果有⾮常多的奖品类型，那么实现的子类会迅速扩张。但是这个缺点使⽤其他的模式进⾏优化。

这篇博客写于处女面被pdd击穿第二天，真是百感交集。。还是想不通为什么我校招和别人社招的问题一模一样，真是哎哎了。



# 更新

- 2024.4.14 更换了一个更加合适的 Demo。
