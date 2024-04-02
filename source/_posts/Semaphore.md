---
title: Semaphore
date: 2024-04-02 01:42:40
tags:
- JUC
- java
---

# Semaphore

Semaphore 是 JDK1.5 引入 JUC 的一个工具类，字面意为信号量（进程间通信的方式之一）。用于控制可以同时访问互斥资源的线程数。

Semaphore 的底层维护着一个指定数量的许可（或者称之为令牌），每次线程访问共享资源的时候都得去尝试获取许可，许可获取到之后才能继续执行接下来的任务。在共享资源访问结束后，还需要归还这份许可。不然许可的数量会越来越少，最后归为0。这个时候就没有许可可以提供给线程了。

## 常用方法

1. `public Semaphore(int premits)`
	Semaphore 的只带一个参数的构造函数，指定许可的数量。

2. `public Semaphore(int premits, boolean fair)`
	带两个参数的构造函数，第一个是指定许可的数量，第二个是一个公平策略，如果 true，保证按照线程请求顺序分配许可证，如果为 false，则以一种非确定的方式分配许可证。开启公平策略会消耗一定的性能。

3. `public void acquire()`
	拿到一个许可，会导致 Semaphore 的数量减一，如果 Semaphore 的数量为0就拿不到许可。

4. `public void acquire(int premits)`

	拿指定数量的许可。

5. `public void release()`

	归还一份许可，会导致 Semaphore 的数量加一。

6. `public void release(int premits)`

	归还指定数量的许可。

7. `public boolean tryAcquire()`

	尝试拿一份许可，如果拿不到，会立即返回 false。

8. `public boolean tryAcquire(long timeout, TimeUnit unit)`

	为尝试拿许可的操作设置一个操作时间，只有在超过超时时间之后才会返回 false。

9. `public boolean tryAcquire(int premits)`

	尝试拿指定数量的许可，拿不到立即返回 false。

10. `public boolean tryAcquire(int premits, long timeout, TimeUnit unit)`

	尝试拿指定数量的许可，并设置超时时间。

11. `public int availablePremits()`

	用于获取当前信号量中可用的许可数。

## Demo

在面试中，对于多线程的考核有一个很常见的题就是让三个线程A，B，C按顺序打印1到100。思路就是：为三个线程发放一个许可，只有拿到许可的线程才能够打印，让这份许可在三个线程之间按顺序流转即可。

``` java
public class OrderPrintDemo {
 
  private static final Semaphore s1 = new Semaphore(1);
  private static final Semaphore s2 = new Semaphore(0);
  private static final Semaphore s3 = new Semaphore(0);
  
  public static void orderPrint(String printer, Semaphore curr, Semaphore next) {
    for (int i = 1; i <= 100; i++) {
      try {
        curr.acquire();
        System.out.println("线程 " + printer + " 打印 " + i);
      	next.release();
      } catch(Exception e) {
        throw new RuntimeException(e);
      }
    }
  }
  
  public static void main(String[] args) {
		new Thread(() -> {
      orderPrint("A", s1, s2);
    }).start();
    
    new Thread(() -> {
      orderPrint("B", s2, s3);
    }).start();
    
    new Thread(() -> {
      orderPrint("C", s3, s1);
    }).start();
  }
}
```

在这个 demo 中，线程只有在拿到许可后，才能去打印，打印结束后就将许可转移给下一个线程。如果没有拿到线程，则阻塞等待。

对这个场景稍加改动，要求三个线程轮流打印1到100，每个数字只由一个线程打印一次。要求输出如下：

```
A: 1
B: 2
C: 3
A: 4
B: 5
...
```

下面是使用信号量解决这个问题的示例：

``` java
public class OrderPrintDemo2 {
    private static final Semaphore semaphoreA = new Semaphore(1);
    private static final Semaphore semaphoreB = new Semaphore(0);
    private static final Semaphore semaphoreC = new Semaphore(0);

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            for (int i = 1; i <= 100; i += 3) {
                try {
                    semaphoreA.acquire();
                    System.out.println("A: " + i);
                    semaphoreB.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 2; i <= 100; i += 3) {
                try {
                    semaphoreB.acquire();
                    System.out.println("B: " + i);
                    semaphoreC.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        });

        Thread t3 = new Thread(() -> {
            for (int i = 3; i <= 100; i += 3) {
                try {
                    semaphoreC.acquire();
                    System.out.println("C: " + i);
                    semaphoreA.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        });

        t1.start();
        t2.start();
        t3.start();
    }
}
```

