---
title: pdd暑期实习一面之死锁设计
date: 2024-04-01 20:10:33
tags:
- 暑期实习
- JUC
- DeadLock
- Java
---

> 在 pdd 一面的时候，面试官让我设计一个100%死锁的场景，要求是不能使用 Thread.sleep() 和 while。刚听完我内心直接崩溃，因为之前从来没有主动去设计一个死锁，往往都是要避免死锁。因此我觉得十分有必要记录一下，这道题从反向考察了对多线程的理解，太神了（但是我还是想不明白为什么我校招和别人社招的问题一模一样）。

要想回答出这个问题，得先了解在 JDK1.5 的时候引入的两个 JUC 工具类：`CountDownLatch` 和 `CyclicBarrier`。

# CountDownLatch

CountDownLatch 又称门闩或闭锁，它允许一个线程或多个线程一直等待，直到其他线程执行完毕之后再执行。就好比课代表们收作业，课代表们必须等到同学们交齐了自己负责这一科的作业之后才能把作业统一交给老师。课代表可以是一个也可以是多个，也就对应了这里的一个或多个线程，交作业的同学就是这里被等待的线程。

它的底层是通过一个计数器来实现的，这个计数器的初始化值为交作业同学的数量，也就是被等待线程的数量。当 CountDownLatch 为0的时候就说明其他线程已经全部执行完毕，这个时候在 CountDownLatch 上等待的线程就可以继续执行了，也就是说课代表们可以把作业统一交给老师了。

CountDownLatch 的常用方法如下：

1. `public CountDownLatch(int count)`
	CountDownLatch 的构造函数，接收一个 int 型的变量，表示被等待的线程的数量，也就是需要交作业的同学的数量。
2. `public void await()`
	使当前线程进入同步队列等待其他线程执行完毕，当 CountDownLatch 的值为0的时候，这些等待线程就会被唤醒。
3. `public boolean await(long timeout, TimeUnit unit)`
	带超时时间的 await。
4. `public void countDown()`
	使计数器的值减一，一个被等待线程如果想要告诉其他线程自己执行完毕，就调用这个方法。
5. `public long getCount()`
	获取当前 CountDownLatch 的值，也就是计数器的值。

## Demo

下面写一个 Demo，在这个 Demo 中，主线程需要等待所有的子线程完成自己的任务之后才能继续执行自己的任务。

``` java
public class CountDownLatchDemo1 {
  private static final int THREAD_NUM = 5;
  
  public static void main(String[] args) throws InterruptedException {
  	CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUM);
    for (int i = 0; i < THREAD_NUM; i++) {
      Thread t = new Thread(() -> {
        System.out.println(Thread.currentThread().getName() + " 开始吃炒饭");
        try {
          Thread.sleep(1000);  // 模拟吃炒饭的时间
        } catch (Exception e) {
          throw new RuntimeException(e);
        }
        System.out.println(Thread.currentThread().getName() + " 吃完了");
        countDownLatch.countDown();
      });
      t.start();
    }
    
    countDownLatch.await();  // 主线程等待其他线程吃完了再擦桌子
    System.out.println("主线程开始擦桌子");
  }
}
```

在上面这个 Demo 中，只有一个主线程在 CountDownLatch 上等待。当然，也可由多个线程在 CountDownLatch 上等待。设想另一个场景，一桌客人正在吃饭，但他们吃完后，需要几个服务员一起来收拾桌子。在这个场景中，客人就是被等待的线程，服务员就是等待的线程，并且可以有多个线程一起等待。下面是这个场景的实现：

``` java
public class CountDownLatchDemo2 {
  private static final int GUEST_NUM = 5;  // 5个客人
  private static final int CLEANER_NUM = 3;  // 3个服务员
  
  public static void main(String[] args) throws InterruptedException {
    final CountDownLatch countDownLatch = new CountDownLatch(GUEST_NUM);
    
    for (int i = 0; i < CLEANER_NUM; i++) {
      Thread t = new Thread(() -> {
        System.out.println(Thread.currentThread().getName() + " 等待中");
        try {
          countDownLatch.await();
        } catch (Exception e) {
          throw new RuntimeException(e);
        }
        System.out.println(Thread.currentThread().getName() + " 开始收拾桌子");
      });
      t.start();
    }
    
    for (int i = 0; i < GUEST_NUM; i++) {
      Thread t = new Thread(() -> {
        System.out.println(Thread.currentThread().getName() + " 正在吃饭");
        try {
          Thread.sleep(1000);  // 模拟吃饭
        } catch (Exception e) {
          throw new RuntimeException(e);
        }
        System.out.println(Thread.currentThread().getName() + " 吃完了");
        countDownLatch.countDown();
      });
      t.start();
    }
  }
}
```



## 使用场景

1. 将一个任务交给多个线程执行，等待这些线程全部执行完毕后，再由主线程来汇总。
2. 实现最大并行性。让多个线程处于同一个起跑线开始运行，await 就相当于线程向外界宣布自己准备好了。等待所有线程准备好之后，也就是 CountDownLatch 的值为0的时候。这些在 CountDownLatch 上等待的线程就能够同时开始执行。



# CyclicBarrier

CyclicBarrier 字面意思是循环栅栏，循环就是可重复使用的意思。它可以使一定数量的线程全部等待彼此到达共同的屏障点。它的作用就好比它的名字，设置一个栅栏，当一个线程到达栅栏的时候，就通知其他的线程，当指定数量的线程到达栅栏后，就放开栅栏让这些线程继续执行。

CyclicBarrier 常用方法如下：

1. `public CyclicBarrier(int parties)`
	CyclicBarrier 的构造函数，指定需要等待的线程数，当给定数量的线程到达后，放开栅栏。
2. `public CyclicBarrier(int parties, Runnable barrierAction)`
	CyclicBarrier 第二种构造函数，注册一个回调函数，在栅栏放开之前，由最后一个到达的线程执行回调函数。
3. `public int await()
	当一个线程调用了 await 方法，就是在向 CyclicBarrier 声名自己已经到达了栅栏，正在等待其他线程到达。
4. `public int await(long timeout, TimeUnit unit)`
	带超时的 await 方法。
5. `public void reset()`
	将屏障初始化，这也是它与 CountDownLatch 不同之处。
6. `public int getNumberWaitting()`
	返回当前正在栅栏处等待的线程数。
7. `public boolean isBroken()`
	阻塞的线程是否被中断。

## Demo

下面以一个聚会吃饭的场景设计一个 Demo，当所有的人都到场后才能够开始吃饭。

``` java
public class CyclicBarrierDemo {
  private static final int THREAD_NUM = 10;
  
  public static void main(String[] args) throws InterruptedException {
    final CyclicBarrier cyclicBarrier = new CyclicBarrier(THREAD_NUM);
    for (int i = 0; i < THREAD_NUM; i++) {
			Thread t = new Thread(() -> {
        try {
          Thread.sleep(1000);  // 模拟前往聚会
        } catch (Exception e) {
          throw new RuntimeException(e);
        }
        System.out.println(Thread.currentThread().getName() + " 等待其他人ing");
        try {
          cyclicBarrier.await();
        } catch (Exception e) {
          throw new RuntimeException(e);
        }

        System.out.println(Thread.currentThread().getName() + " 开动啦！");
      });
      t.start();
    }
  }
}
```

## 使用场景

~~暂时没感觉到这俩有啥区别，可能是因为我太菜了~~

目前的感觉就是简单的需求用 CountDownLatch，复杂的场景使用 CyclicBarrier。



# CountDownLatch与CyclicBarrier对比

1. CountDownLatch 的计数器只能使用一次，而 CyclicBarrier 提供了 reset 方法，可以重复使用，适合更复杂的场景，从 CyclicBarrier 的名字也能看出来 —— 循环栅栏。
2. CyclicBarrier 还提供了一些其他有用的方法，例如判断当前线程是否被中断的 isBroken 方法。



# 100%死锁设计

```java
import java.util.concurrent.CountDownLatch;

public class DeadLockDemo {
    public static void main(String[] args) {
        Object lock1 = new Object();
        Object lock2 = new Object();

        final CountDownLatch count = new CountDownLatch(2);

        Thread t1 = new Thread(() -> {
            synchronized(lock1) {
                System.out.println("t1 get lock1");
                count.countDown();
                try {
                    count.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
                System.out.println("t1 死锁");
                synchronized(lock2) {
                    System.out.println("t1!!!");
                }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized(lock2) {
                System.out.println("t2 get lock1");
                count.countDown();
                try {
                    count.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
                System.out.println("t2 死锁");
                synchronized(lock1) {
                    System.out.println("t2!!!");
                }
            }
        });

        t1.start();
        t2.start();
    }
}
```

在这个100%死锁的 Demo 中，使用 CountDownLatch 让两个线程到达同一起跑线，并且手中各自持有了一把锁，同时起跑去争夺对方手里的锁。同理也可以使用 CyclicBarrier 来实现。
