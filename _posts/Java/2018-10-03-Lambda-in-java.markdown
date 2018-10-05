---
title: Lambda in Java
layout: post
date:   2018-07-15
author: "bing5tui3"
tags: [java]
categories: [java]
excerpt_separator: <!--more-->
---

在Java的面向对象编程中，所有的代码都必须放在类中实现。这是非常麻烦的一件事，因为编写一个类的语法相当的严格，而且类的放置也有一定的讲究。Java 8之后，引入了lambda表达式，标志着Java也半只脚进入了函数式编程的时代。 本文记录了我对Lambda语法的一些理解。

<!--more-->

---

#### 使用Lambda表达式的理由

- 开启函数式编程（闭包）
- 可读性更高和更加简洁的代码
- 并行编程的可能
- 更加方便使用的API

---

#### 初入Lamdba表达式

以下是我第一次入门lambda表达式的场景，其实就是从最简单的Runnable接口进行了第一次的尝试。

在Web后端开发中，我们可能会经常编写一些功能，而这些功能可能会依赖一下守护线程来维护其健康性。
比如一个类包含了一个容器，该容器会在程序运行过程中一直累积元素（比如缓存某些数据），然后通过守护线程定期清理以保持其容量保持在可控范围内。

假设我们有个`CacheService`，其缓存最近**10s**所读取的用户信息。

**我们的用户类`User`如下：**

~~~ java
package com.bing5tui3.lambda;

public class User {
	// 用户id
    private int id;
    // 用户名称
    private String name;

    /** 常规的一些getters & setters */
}
~~~

**模拟一个用户的Repository类`UserService`（从持久化层获取数据），初始化10个样例用户（user0 - user9）：**

~~~ java
package com.bing5tui3.lambda;

import java.util.HashMap;
import java.util.Map;

public class UserService {

    private Map<Integer, User> userRepo = new HashMap<>();

    public UserService() {
        for (int i = 0; i < 10; i++) {
            User user = new User();
            user.setId(i);
            user.setName("user:" + i);
            userRepo.put(i, user);
        }
    }

    public User getUser(int id) {
    	System.out.println("user got from repo");
        return userRepo.containsKey(id) ? userRepo.get(id) : null;
    }
}
~~~

**我们的缓存类`CacheService`：**

~~~ java
package com.bing5tui3.lambda;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class CacheService {

    private Map<Integer, User> usercache;

    // 模拟从持久化层获取用户数据
    private UserService userService = new UserService();

    public CacheService() {
        usercache = new ConcurrentHashMap<>();
    }

    // 模拟缓存的情况，如果缓存里有则取缓存的，否则从持久化层获取
    public User getUser(int id) {

        if (usercache.containsKey(id)) {
            System.out.println("user got from cache");
            return usercache.get(id);
        } else {
            User user = userService.getUser(id);
            if (user != null) {
                usercache.put(user.getId(), user);
                return user;
            }
        }

        return null;
    }
}
~~~

**编写一个测试入口：**

~~~ java
package com.bing5tui3.lambda;

public class Application {

    public static void main(String[] args) {
        CacheService cacheService = new CacheService();
        User user = cacheService.getUser(1);
        System.out.println("User name is : [" + user.getName() + "]");
        user = cacheService.getUser(1);
        System.out.println("User name is : [" + user.getName() + "]");
    }
}
~~~
可以看到获取两次一样的用户后，第二次就会从缓存中拿取：

~~~
user got from repo
User name is : [user:1]
user got from cache
User name is : [user:1]
~~~

这里只是一个功能模拟的场景，如果说持久化的用户数据量非常大，则这个`CacheService`缓存的数据量也会越来越大直到内存溢出。

那么我们就需要想办法定期清理缓存，最直接的方法就是找一个守护线程，每**10s**清空这个缓存`Map`，在这里我们在	`CacheService`中增加一个`cleanCache()`的方法，然后让其在实例初始化后启动一个守护线程：

~~~ java
package com.bing5tui3.lambda;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class CacheService {

    private Map<Integer, User> usercache;

    // 模拟从持久化层获取用户数据
    private UserService userService = new UserService();

    public CacheService() {
        usercache = new ConcurrentHashMap<>();

        // 实例被初始化后即刻启动守护线程进行缓存的维护工作
        cleanCache();
    }

    // 模拟缓存的情况，如果缓存里有则取缓存的，否则从持久化层获取
    public User getUser(int id) {

        if (usercache.containsKey(id)) {
            System.out.println("user got from cache");
            return usercache.get(id);
        } else {
            User user = userService.getUser(id);
            if (user != null) {
                usercache.put(user.getId(), user);
                return user;
            }
        }

        return null;
    }

    private void cleanCache() {
        Thread thread = new Thread(
                /** 清理工作 */
        );
        thread.setDaemon(true);  // 设置为守护线程或者叫后台线程
        thread.start();
    }
}
~~~

接下来我们就需要在`Thread`中填充我们的任务执行代码。

首先我们需要编写一个清理的方法`clean()`，其功能就是每10s清理一次缓存（这里使用暴力清理`Map`接口的`clear()`）：

~~~ java
private void clean() {
    System.out.println("cleaner starts...");
    while (true) {
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("cleaning cache...");
        usercache.clear();
        System.out.println("finished cleaning cache...");
    }
}
~~~

然后要将任务代码绑定到`Thread`上，在Java 8之前，我们在这里最常用的方法就是**匿名内部类**：

~~~ java
private void cleanCache() {
    Thread thread = new Thread(
            new Runnable() {
                @Override
                public void run() {
                    clean();
                }
            }
    );
    thread.setDaemon(true);
    thread.start();
}
~~~

修改测试入口：

~~~ java
public static void main(String[] args) throws InterruptedException {
    CacheService cacheService = new CacheService();
    User user = cacheService.getUser(1);
    System.out.println("User name is : [" + user.getName() + "]");
    user = cacheService.getUser(1);
    System.out.println("User name is : [" + user.getName() + "]");

    Thread.sleep(15000);
    user = cacheService.getUser(1);
    System.out.println("User name is : [" + user.getName() + "]");
}
~~~

可以看到在有守护线程清理的情况下，第三次获取用户信息时，是从`UserService`中获取的：

~~~
user got from repo
User name is : [user:1]
user got from cache
User name is : [user:1]
cleaner starts...
cleaning cache...
finished cleaning cache...
user got from repo
User name is : [user:1]
~~~

那么这里就可以看到通过匿名内部类的方式，这个`Thread`中的`Runnable`构造写了很多的冗余代码：比如声明啊，注解啊等。而这些代码又是重复性的，也就是如果我再有一个守护线程，可能这些声明还是得再写一遍。那么这个时候lambda表达式就来了：

~~~ java
public void cleanCache() {
    Thread thread = new Thread(
            () -> clean()
    );
    thread.setDaemon(true);
    thread.start();
}
~~~
可以看到，通过lambda表达式，我们省去了方法的声明，省去了访问修饰符，返回值的声明和名字等。 

当然我们还可以用方法引用的语法（Method Reference）使得代码更加简洁：

~~~ java
private void cleanCache() {
    // this::clean就是一个方法引用，指向当前实例的clean()方法。相当于一个方法指针
    Thread thread = new Thread(this::clean);
    thread.setDaemon(true);
    thread.start();
}
~~~

是不是发现lambda表达式将一些重复性的代码、影响阅读体验的代码省略后，代码逻辑一下子就清晰了很多？

这也就是我接触lambda表达式的第一个应用，当然我当时写的时候类中是有很多守护线程的，Runnable的构造也是相当的多，到后来就非常影响阅读体验了。

接下来我们就细节的聊一下如何理解Java的Lambda语法。

---

#### Lambda语法如何理解？

那么lambda语法如何理解呢？我们用一些例子来慢慢讲。

我们写一个`HelloService`类，其有一方法`sayHello()`，我们看到普通的方法调用需要**`new`**一个`HelloService`的实例，然后通过该实例调用其`sayHello()`方法：

~~~ java
package com.bing5tui3.lambda;

public class HelloService {

    public void sayHello(String content) {
        System.out.println("hello " + content + "!");
    }

}


class Main {
    public static void main(String[] args) {
        HelloService hello = new HelloService();
        hello.sayHello("world");
    }
}
~~~

但是这样的写法有个问题，`sayHello`只能是一个固定实现，任何调用`sayHello`的实例，都将输出类似"hello,blabla...!"的玩意儿，我们只能控制这个"blabla"是什么，而不能控制方法的内部执行逻辑。

而现在函数式编程中想要做的事情是什么呢，其实就是通过接受参数动态控制方法体内逻辑的执行，类似于下面的伪代码：

~~~ java
public void sayHello(___) {
    ________;
}
~~~

那么在lamdba语法出现之前，Java是怎么做的呢？

通过接口！

我们写一个`Hello`接口：

~~~ java
package com.bing5tui3.lambda;

public interface Hello {
    void sayHello();
}
~~~

然后我们修改`HelloService`的实现，让`sayHello`从接收字符串，改成接收一个`Hello`的接口实例，然后调用接口中的`sayHello`方法：

~~~ java
package com.bing5tui3.lambda;

public class HelloService {

    public void sayHello(Hello hello) {
        hello.sayHello();
    }

}
~~~

这样是不是我们既控制了参数，又控制了方法执行？

`HelloService`中，本来`sayHello`是固定的执行逻辑（`System.out::print`)，现在它将执行方法的权利让渡给了`Hello`接口。

那么如何控制参数呢？我们通过编写`Hello`接口的不同实现类啊！

那么如何控制方法执行呢？我们通过编写`sayHello()`方法的不同实现啊！

那这里假设我就想输出"Hello world!"该如何实现呢？

编写一个`HelloWorld`类，实现`Hello`接口和其接口方法`sayHello()`：

~~~ java
package com.bing5tui3.lambda;

public class HelloWorld implements Hello {
    @Override
    public void sayHello() {
        System.out.println("Hello world!");
    }
}
~~~

然后修改主程序：

~~~ java
public static void main(String[] args) {
    // 初始化service
    HelloService helloService = new HelloService();
    // 初始化我们自定义的实现
    HelloWorld helloWorld = new HelloWorld();
    // 用我们自定义的实现去调用service
    helloService.sayHello(helloWorld);
}
~~~

其结果就是我们通过自己实现了一个`Hello`接口的`HelloWorld`类，并将其实例传入`helloService`中，控制了`helloService`里的`sayHello`方法。