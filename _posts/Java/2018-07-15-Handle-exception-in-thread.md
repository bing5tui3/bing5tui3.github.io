---
title:  "Handle exception in thread"
layout: post
date:   2018-07-15
author: "bing5tui3"
tags: [java]
categories: [java]
excerpt_separator: <!--more-->
---

软件开发过程中，可能常常会需要一些线程来处理一些任务，这些线程不能死亡，需要一直监听着某些事件的发生。如果线程死亡，那么可能会导致该处理的事项没有得到处理。本文记录一些线程开发过程中的异常处理方法。

<!--more-->

#### 线程异常
我们知道异常类都派生至 `Throwable`，其子类又主要分为 `Error` 类和 `Exception` 类。 `Exception` 类又派生出子类 `RuntimeException` 和 其他异常类。

* 所谓的非受查异常指的就是派生于 `Error` 类和 `RuntimeException` 类的异常类。
* 所谓的受查异常指的就是除了非受查异常以外的其他异常类。

线程的 `run()` 方法不能抛出任何受查异常，但是非受查异常会导致线程中止。在这种情况下，线程就死亡了。

所以我们如果要保证一个长时间不间断运行的线程时，一定要非常小心的处理异常问题。 

一般情况下，我们有两种处理方式：

* `run()` 方法中抓住所有异常，并对所有异常进行处理，这就难免需要生吞一些异常。
* `run()` 方法只抓取与业务逻辑有关的异常，让其他异常由异常处理器处理，线程死亡后恢复线程。

#### 异常由线程自己处理
异常由线程自己处理指的就是线程抓住所有可能的异常，然后自己处理，并保证线程不会意外死亡。

第一个例子：
该任务每秒打印一次`count`，每5秒抛出一个`Exception`, 但是线程不中断：

~~~ java
class Task implements Runnable {

    @Override
    public void run() {
        int i = 0;
        while (true) {
            i ++;
            System.out.println(i);
            try {
                Thread.sleep(1000);
                if (i % 5 == 0) {
                    // 每5次抛出一个异常
                    throw new Exception("An exception thrown deliberately");
                }
            } catch (Throwable e) {
            	// 抓住异常打印日志
                System.out.println(e.getMessage());
            }
        }
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(new Task());
        executorService.shutdown();
    }
}
~~~

在这个例子中，模拟线程产生异常，通过抓取`Throwable`类的异常，即可将所有可能的异常都`catch`住。 该例子中只是对错误进行了日志打印，其实就是所谓的生吞了异常。

运行结果：

~~~ console
1
2
3
4
5
An exception thrown deliberately
6
7
8
9
10
An exception thrown deliberately
11
12
13
14
15
An exception thrown deliberately
~~~

我们可以加一个方法`handleException`来处理异常，避免生吞异常（这里为了简便，还是只打印日志）：

~~~ java
class Task implements Runnable {

    @Override
    public void run() {
        int i = 0;
        while (true) {
            i ++;
            System.out.println(i);
            try {
                Thread.sleep(1000);
                if (i % 5 == 0) {
                	// 每5次抛出一个异常
                    throw new Exception("An exception thrown deliberately");
                }
            } catch (Throwable e) {
            	// 将异常交给自定义的处理方法
                handleException(e);
            }
        }
    }

    // 处理异常的方法
    public void handleException(Throwable e) {
        System.out.println(e.getMessage());

        // do something else;
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(new Task());
        executorService.shutdown();
    }
}
~~~

但是这样就有一个问题，如果在`handleException`方法中又抛出异常了怎么办：

~~~ java
public void handleException(Throwable e) {
    System.out.println(e.getMessage());
    // 丢出一个非受查异常
    throw new RuntimeException("An unchecked exception thrown deliberately");
}
~~~

我们用抛出异常的方法替换之前的`handleException`，执行一下看下结果：

~~~ console
1
2
3
4
5
An exception thrown deliberately
Exception in thread "pool-1-thread-1" java.lang.RuntimeException: An unchecked exception thrown deliberately
	at xyz.ruidegozaru.thread.test.Task.handleException(ThreadTest.java:59)
	at xyz.ruidegozaru.thread.test.Task.run(ThreadTest.java:51)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
~~~
发现线程因为非受查异常而中止了。 除非将该异常生吞，否则如果还要保持线程的正常运作，就不免产生无限处理异常的逻辑问题：

~~~ java
try {
    // do some business work
} catch (Throwable e1) {
    try {
        handleException(e1);
    } catch (Throwable e2) {
    	try {
    	    handleAnotherException(e2);
    	} catch (Throwable e3) {
    	    // ... 除非生吞异常一次，否则必须无限处理异常
    	}
    }
}
~~~

从目前的例子来看，如果异常由线程自己处理，则在处理过程中，一定会要生吞一次异常，否则会陷入无限处理异常的逻辑当中。


#### 异常由【线程类】的未捕获异常处理器处理

线程类中有个`UncaughtExceptionHandler`接口，我们可以通过实现这个接口来创建我们自己的未捕获异常处理器：

~~~ java
class ExceptionHandler implements Thread.UncaughtExceptionHandler {

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println(e.getMessage());
    }
    
}
~~~

这样我们可以为我们的线程设置这个未捕获异常处理器：

~~~ java
class Task implements Runnable {

    @Override
    public void run() {
        int i = 0;

        // 设置异常处理器
        Thread.currentThread().setUncaughtExceptionHandler(new ExceptionHandler());

        while (true) {
            i ++;
            System.out.println(i);
            try {
                Thread.sleep(1000);
                if (i % 5 == 0) {
                    throw new Exception("An exception thrown deliberately");
                }
            } catch (Throwable e) {
                handleException(e);
            }

        }
    }

    public void handleException(Throwable e) {
        System.out.println(e.getMessage());
        throw new RuntimeException("An unchecked exception thrown deliberately");
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        // 如果这里用submit的话，则需要在future实例中获得异常
        executorService.execute(new Task());
        executorService.shutdown();
    }
}

class ExceptionHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println(e.getMessage());
    }
}
~~~

这样，`handleException`里面抛出来的错误就会在`ExceptionHandler`里面被抓住（这里是打印message）。 运行结果如下：

~~~ console
1
2
3
4
5
An exception thrown deliberately
An unchecked exception thrown deliberately
~~~
