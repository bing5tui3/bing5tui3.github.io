---
title: Java Concurrency In Practice - (1)
layout: post
date:   2019-02-23
author: "bing5tui3"
tags: [java, concurrency]
categories: [java]
excerpt_separator: <!--more-->
---

最近在学习《Java并发编程实战》，觉得还是非常开阔眼界的。  
因此，特意想为Java并发相关的内容做一个知识的整理。  
本篇为第一篇，整理一下**`synchronized`**关键字的相关要点。

<!--more-->

#### synchronized关键字

`synchronized`关键字是Java中主要的同步机制，它提供了一种独占的加锁方式。

##### 内置锁
`synchronized`第一个特点是内置，其锁的机制由Java语言本身的实现所提供。 以下引用摘自《Java并发编程实战》：
> **每个**Java**对象**都可以用作一个实现同步的锁，这些锁被称为内置锁（Intrinsic Lock）或监视器锁（Monitor Lock）。   
> Java提供了内置锁来支持原子性：同步代码块（Synchronized Block）。同步代码块包括两个部分：  
> 1. 一个作为锁的对象引用。
> 2. 一个作为由这个锁保护的代码块。  

> 以关键字`synchronized`来修饰的方法就是一种横跨整个方法体的同步代码块，其中该同步代码块的锁就是方法调用所在的对象。静态的	`synchronized`方法以Class对象作为锁。

归纳一下`synchronized`关键字作为内置锁的时候一些语法使用上的特点：
1. `synchronized`关键字接受一个对象作为锁的对象。
2. `synchronized`关键字修饰成员方法的时候直接以当前实例（`this`）作为锁的对象。
3. `synchronized`关键字修饰静态方法的时候使用当前类的对象作为锁的对象。

`synchronized`实现的锁是通过对象来锁的，只要使用相同的对象，`synchronized`所保护的代码就能实现同步。   

##### 重入

`synchronized`内置锁的第二个比较重要的特点是**重入**。以下内容摘自《Java并发编程实战》：

> 我们知道当某个线程请求一个由其他线程占有的`synchronized`锁时，发出请求的线程就会阻塞。
然而，由于内置锁是可重入的，因此如果某个线程试图获得一个已经由它自己持有的锁，那么这个请求就会成功。
**“重入”**意味着获得锁的操作的粒度是**“线程”**，而不是**“调用”**。


---
---

#### 内置锁示例

接下来的例子都是使用`synchronized`关键字来保证`count`变量在并发修改情况下的原子性。  
可以从选用的例子中看到，`synchronized`的关键字本质使用一个对象来作为锁，任何其他线程想要进入被`synchronized`所保护代码块中都要获取该对象的内置锁。（例子逻辑较为简单，不做过多说明了）

**1. 将任何一个Java对象作为锁的对象，这里是`synchronized(o)`**：

~~~ java
public class SynchronizedClass1 implements SynchronizedClass {

    private int count = 0;

    private Object o = new Object();

    @Override
    public void increase() {
        // synchronized，对对象o进行加锁
        synchronized (o) {
            count ++;
            System.out.println(Thread.currentThread().getName() + ": " + count);
        }
    }
}
~~~

---

**2. 将当前实例作为锁的对象, 这里是`synchronized(this)`**：

~~~ java
public class SynchronizedClass2 implements SynchronizedClass {

    private int count = 0;

    @Override
    public void increase() {
        // synchronized，对当前实例对象进行加锁
        synchronized (this) {
            count ++;
            System.out.println(Thread.currentThread().getName() + ": " + count);
        }
    }
}
~~~

---

**3. `synchronized`修饰的实例方法**：

~~~ java
public class SynchronizedClass3 implements SynchronizedClass{

    private int count = 0;

    @Override
    public synchronized /* 对当前对象this进行上锁, 相当于synchronized(this) */ void increase() {
        count ++;
        System.out.println(Thread.currentThread().getName() + ": " + count);
    }
}
~~~

---

**4. 静态方法使用当前`Class`对象作为锁的对象**：

~~~ java
public class SynchronizedClass4 {

    private static int count = 0;

    public static void increase() {
        // 静态方法可以通过对自己的Class类对象上锁
        synchronized (SynchronizedClass4.class) {
            count ++;
            System.out.println(Thread.currentThread().getName() + ": " + count);
        }
    }
}
~~~

---

**5. 静态方法对一个静态成员对象进行上锁（实际上还是找一个已经存在的对象进行上锁）**：

~~~ java
public class SynchronizedClass5 {

    private static int count = 0;

    private static Object o = new Object();

    public static void increase() {
        // 可以对已存在的对象进行上锁
        synchronized (o) {
            count ++;
            System.out.println(Thread.currentThread().getName() + ": " + count);
        }
    }
}
~~~

---

**6. `synchronized`修饰的静态方法**：

~~~ java
public class SynchronizedClass6 {

    private static int count = 0;

    // 静态方法使用synchronized关键字相当于synchronized(SynchronizedClass6.class)
    public static synchronized void increase() {
        count ++;
        System.out.println(Thread.currentThread().getName() + ": " + count);
    }
}
~~~

---

**7. 锁位置在对象上的验证**

虽然`synchronized`关键字语法上接受一个对象的引用，但实际上的锁是在对象上实现的。

~~~ java
public class SynchronizedClassForRef {

    private int count = 0;
    // 仅有引用，而没有实际对象
    private /*volatile */ Object o = new Object();

    public void increase() {
        while (true) {
            // 每次自增count变量，都先获取对象o的锁
            synchronized (o) {
                count++;
                System.out.println(Thread.currentThread().getName() + ":" + count);
            }
        }
    }

    // 将o的引用指向null而非之前的对象
    public void changeObjectReference2Null() {
        o = null;
    }
}

class App {
    public static void main(String[] args) throws InterruptedException {

        SynchronizedClassForRef target = new SynchronizedClassForRef();

        Thread th1 = new Thread(() -> target.increase());
        th1.start();

        Thread.sleep(1000); // 先让th1线程运行一小段时间

        Thread th2 = new Thread(() -> target.changeObjectReference2Null());
        th2.start();  // 通过线程th2去清理o的引用，将o指向null

    }
}
~~~

最终运行时会在**线程th1**中报错`NullPointerException`，因为**线程th2**将锁的目标对象清理了：

~~~
...
Thread-0:247972
Thread-0:247973
Exception in thread "Thread-0" java.lang.NullPointerException
	at io.bing5tui3.concurrent.sync.SynchronizedClassForRef.increase(SynchronizedClassForRef.java:12)
	at io.bing5tui3.concurrent.sync.App.lambda$main$0(SynchronizedClassForRef.java:30)
	at java.lang.Thread.run(Thread.java:748)
~~~

---
---

#### 重入锁示例

**1. synchronized方法A调用synchronized方法B**：

当某线程执行`increase()`方法时，说明该线程已经获取`this`对象上的锁，如果该锁非重入的，则在`increase()`方法内再调用`decrease()`方法将产生死锁，因为`decrease()`方法的执行也依赖于`this`对象上的锁。  
事实上`increase()`方法将正常执行完毕（也符合我们的编程常识），因为`synchronized`使用的内置锁是可重入的。

~~~ java
public class SynchronizedClass7 {

    private int count = 0;

    public synchronized void increase() {
        count ++;
        System.out.println(Thread.currentThread().getName() + ": " + count);

        // 在increase()方法已经在this对象上获得锁的情况下继续调用需要在this对象上获得锁的decrease()方法
        // 可重入的情况下方法会执行完毕并成功返回
        // 非重入的情况下将会死锁
        this.decrease();
    }

    public synchronized void decrease() {
        count --;
        System.out.println(Thread.currentThread().getName() + ": " + count);
    }

    public static void main(String[] args) {
        SynchronizedClass7 target = new SynchronizedClass7();
        // 直接main线程调用
        target.increase();
    }

}
~~~

**2. 子类重写后的synchronized方法调用父类synchronized方法**：

另外一个可重入的例子就是重写后的子类同步方法调用父类同类方法。

~~~ java
class BaseSynchronizedClass {

    public synchronized void print() {
        System.out.println(Thread.currentThread().getName() + " " + this.getClass().getSuperclass());
    }

}

public class SynchronizedClass8 extends BaseSynchronizedClass {

    @Override
    public synchronized  void print() {
        System.out.println(Thread.currentThread().getName() + " " + this.getClass());
        // 如果锁是非重入的，则super.print()将会阻塞，并产生死锁
        super.print();
    }

    public static void main(String[] args) {
        SynchronizedClass8 target = new SynchronizedClass8();
        target.print();
    }
}
~~~

#### 结语

其实`synchronized`关键字不仅保证了其保护的代码块的执行**原子性**，还保证了代码块内部共享变量的**可见性**（可见性的知识点会在后续文章中整理）。  

`synchronized`关键字所获得的锁将由**`JVM`**来管理释放，当`synchronized`的同步代码块执行完毕，或者抛出异常，则锁将释放。

另外，`synchronized`修饰方法是一种粗粒度的上锁方式，若一个类中有多个方法被`synchronized`修饰，则在并发环境下，同一个对象上的这些方法将在整体上串行执行。所以在复杂一点的情况下，一般使用`synchronized`修饰代码块而不是直接修饰方法：

~~~ java
synchronized (someObj) {
    ... // 同步代码
}
~~~