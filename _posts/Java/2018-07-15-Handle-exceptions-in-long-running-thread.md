---
title:  "Handle exceptions in long-runing threads"
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
	at ...
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

但是这样子线程就意外死亡了。

#### 使用守护线程来恢复意外死亡的线程

为了保证一个线程的长时间运行，我们可以尝试在`run()`方法内抓取异常，然后进行处理。并且设置一个`UncaughtExceptionHandler`的实现，为线程运行过程中的非受查异常兜底。但是一旦`UncaughtExceptionHandler`开始执行异常处理，就说明线程要因为异常而死亡了。所以在下面的代码中，为了能够使线程在系统运行过程中一直运作，我添加了一个守护线程，用于观察用户线程是否死亡。

首先使一个工作任务处理类`Worker`：

~~~ java
class Worker implements Runnable, Thread.UncaughtExceptionHandler {

    private String content;

    // worker线程的状态
    private volatile boolean isWorking = false;

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public boolean isWorking() {
        return isWorking;
    }

    public void setWorking(boolean working) {
        isWorking = working;
    }

    public void run() {
        // 开始的时候置状态为working
        isWorking = true;
        // 设置未捕获异常处理器为自己
        Thread.currentThread().setUncaughtExceptionHandler(this);

        while (true) {
            try {
                // working
                doWork();

                Thread.sleep(1000);

            } catch (Throwable t) {
                // handle error
                doError(t);
            }
        }
    }

    public void doWork() throws Exception {
        System.out.println(content);
        throw new Exception("An exception thrown deliberately");
    }

    public void doError(Throwable t) {
        System.out.println("Handling error..." + t.getMessage());
        throw new RuntimeException("An unchecked exception thrown deliberately");
    }

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println(e.getMessage());
        // 当未捕获异常处理器抓到异常时，将working状态置为false
        isWorking = false;
    }
}
~~~

然后写一个`WorkerManager`类，里面有个`watch()`方法，一旦`worker`不工作了，则将`worker`恢复:

~~~ java
class WorkerManager {

    List<Worker> workers;

    ExecutorService executorService = Executors.newCachedThreadPool();

    public WorkerManager(List<Worker> workers) {
        this.workers = workers;
    }

    public void start() {
        
        // 每个worker开始工作
        for (Runnable worker : workers) {
            executorService.execute(worker);
        }

        // 开始监视
        watch();

    }

    // 监视方法
    public void watch() {
        Thread t = new Thread(() -> {
                while (true) {

                    // 如果线程数量小于worker数量
                    if (((ThreadPoolExecutor) executorService).getActiveCount() < workers.size()) {
                        
                        System.out.println("There are some workers not working...");

                        for (Worker worker : workers) {
                            if (!worker.isWorking()) {
                                executorService.execute(worker);
                            }
                        }
                    }

                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {

                    }
                }
            }
        );
        // 设置为守护线程
        t.setDaemon(true);

        t.start();
    }

    public void shutdown() throws InterruptedException {
        Thread.sleep(10000);
        executorService.shutdown();
    }

}
~~~

`main`方法：

~~~ java
public static void main(String[] args) throws InterruptedException {

    Worker w = new Worker();
    w.setContent("my name is WORKER!");
    List<Worker> workers = new ArrayList<>();
    workers.add(w);

    WorkerManager manager = new WorkerManager(workers);
    manager.start();
    manager.shutdown();
}
~~~

运行后结果如下：

~~~ console
my name is WORKER!
Handling error...An exception thrown deliberately
An unchecked exception thrown deliberately
There are some workers not working...
my name is WORKER!
Handling error...An exception thrown deliberately
An unchecked exception thrown deliberately
There are some workers not working...
my name is WORKER!
Handling error...An exception thrown deliberately
An unchecked exception thrown deliberately
There are some workers not working...
my name is WORKER!
Handling error...An exception thrown deliberately
An unchecked exception thrown deliberately
There are some workers not working...
~~~

我们可以看到，在`shutdown()`之前，`Worker w` 一次次地从异常中被恢复。


#### 结论
经过上述讨论，我们可以看到在处理异步线程时，异常处理往往非常头痛，一旦处理不好则会造成线程泄露。

从我个人的实践来看，要写一个与系统生命周期相同的用户线程，我们可以在线程中抓取所有可能的异常，处理能够处理的异常，生吞剩下的不方便处理的异常；或者可以在线程中只抓取我们想要的异常，而将剩下的异常漏给线程类的异常处理器，并辅以守护线程监视用户线程的健康状况，一旦用户线程因异常意外死亡时，守护线程能够尽快将用户线程恢复过来。