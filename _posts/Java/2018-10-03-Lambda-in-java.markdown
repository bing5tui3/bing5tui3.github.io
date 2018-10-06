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

在Web后端开发中，我们可能会经常编写一些功能，而这些功能可能会依赖一下守护线程来维护其健康性。
比如一个类包含了一个容器，该容器会在程序运行过程中一直累积元素（比如缓存某些数据），然后通过守护线程定期清理以保持其容量保持在可控范围内。

&nbsp;

以下是我第一次入门lambda表达式的场景，其实就是从最简单的Runnable接口进行了第一次的尝试。

***场景案例：***

我们有个`UserService`，其为我们提供`User`数据，然后我们还有个`CacheSerivce`，其缓存最近**10s**所读取的用户信息。当我们想要获取一个User数据时，调用`CacheService`获取，如果`CacheService`的缓存中有，则返回缓存中的数据，如果没有则调用`UserService`从持久化数据中读取。

**UML图示如下：**

![image01](/assets/posts/java/lambda-in-java/1.png) 

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
private void cleanCache() {
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

而现在函数式编程中想要做的事情是什么呢，其实就是通过接受参数动态控制方法体内逻辑的执行，可以说是向方法中传递行为，类似于下面的伪代码：

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

如此，我们可以说勉强实现了向一个方法中传递行为。但实际上又并不是真的传递了行为，而是传递了一个包含行为的实例。

**原理图如下：**

![image02](/assets/posts/java/lambda-in-java/2.png)

***我们希望控制`HelloService中`的`sayHello`方法的执行，那么只能让`HelloService`接收一个接口的实现类实例（这里是`Hello`接口类），然后调用接口方法（这里是`sayHello`），我们通过给`Hello`接口编写不同的实现（如`HelloWorld`，`HelloEarth`，`HelloUniverse`），来控制`HelloService`中的`sayHello`执行。这其实也就是基于Java的多态性，也体现了Java中针对接口编程的实践原则。***

&nbsp;

**然而**，我们又希望能够真正的传递行为，像下面的伪代码这样：

~~~ java
public void sayHello(someAction) {
    someAction();
}
~~~

lambda表达式就做了这么一件事，将一段表达式通过内联的方式赋值在一个变量上。

我们希望不用再写一个类，而将一段代码绑定在一个变量上（Function as value），就像下面这样（先不管Function是个什么东西）：

~~~ 
Function someAction = public void sayHello() {
    System.out.println("Hello World!");
}
~~~

Java中可以做到吗？ lambda表达式就做了这么一件事，将一段表达式通过内联的方式赋值在一个变量上。

那么这段代码中有哪些东西是没用的呢？


**首先，public肯定是没用的了，因为这样绑定后，这段代码就是一个局部变量了，和类中的概念就没关系了。我们把`public`去掉：**

~~~ 
Function someAction = void sayHello() {
    System.out.println("Hello World!");
}
~~~

**接下来，方法的名称`sayHello`还有用吗？肯定也没用了，因为这段代码的名称就由这个变量`someAction`来标明了，去掉`sayHello`：**

~~~
Function someAction = void () {
    System.out.println("Hello World!");
}
~~~

**我们再看这个返回类型`void`是否还有必要？ Java编译器已经足够聪明，根据代码来判断返回类型是什么，所以这里返回类型也不需要了，去掉：**

~~~
Function someAction = () {
    System.out.println("Hello World!");
}
~~~

**至此我们的lambda表达式进化的就差不多了，只不过还剩下一点微小的差别：**

~~~
Function someAction = () -> {
    System.out.println("Hello World!");
}
~~~

**OK, 这样我们的lambda表达式就进化完全了，由于这里块内的代码只有一行，我们还可以省略花括号（如果你的代码是多行的，那么花括号还是必需的）：**

~~~ java
Function someAction = () -> System.out.println("Hello World!");
~~~

至此，`someAction`表达的就是一段代码，这个代码接收0个参数，然后打印"Hello World!"，并返回`void`。

由于Java是强类型的语言，所以我们接下来要来看这个lambda表达是的类型是什么？**也就是这个`Function`是什么？**

根据我们一开始的构想，我们希望这段代码能够传递到方法内，然后在方法内执行调用：

~~~ java
public void sayHello(() -> System.out.println("Hello World!")) {
    ...
}
~~~

那么Java是怎么实现的呢？Java可没有新增一个类型叫`Function`，而是希望lambda表达式能够兼容老版本的代码。

Java中复用了原来的“接口类型”作为lambda表达式的类型，回过头来看我们原来的`HelloService`的`sayHello`方法：

~~~ java
public void sayHello(Hello hello) {
    hello.sayHello();
}
~~~

然后再看`Hello`接口：

~~~ java
public interface Hello {
    void sayHello();
}
~~~

显然`Hello`接口中有个待实现的方法：接收0个参数，然后执行一段代码，并返回`void`。

是不是和前面的`someAction`非常搭？someAction就像这个接口的一个实现。那么我们是不是可以像匿名内部类一样通过lambda表达式为一个接口做实现呢？像下面这样：

~~~ java
Hello hello = () -> System.out.println("Hello World!");
~~~

然而Java中，lambda表达式的类型确确实实就是这么用的，其最终可以用一个接口类型来绑定：

~~~ java
public static void main(String[] args) {
    // 初始化service
    HelloService helloService = new HelloService();
    // 初始化我们自定义的实现
    HelloWorld helloWorld = new HelloWorld();
    // 用我们自定义的实现去调用service
    helloService.sayHello(helloWorld);

    System.out.println("========== separate line ==========");
    // 用接口类型绑定lambda表达式
    Hello hello = () -> System.out.println("Hello World From Lambda!");
    // 调用service测试
    helloService.sayHello(hello);
}
~~~

输出结果如下：

~~~
Hello world!
========== separate line ==========
Hello World From Lambda!
~~~

是不是特别像一个简化版的匿名内部类：

~~~ java
// lambda写法
Hello hello = () -> System.out.println("Hello World!");
// 匿名内部类写法
Hello hello = new Hello() {
    @Override
    public void sayHello() {
        System.out.println("Hello World!");
    }
};
~~~

至此我们通过一个不带参数、无返回类型方法来理解lambda表达式的语法，希望对于理解lambda表达式有帮助。

接下来我们看看lambda表达式在不同场景的语法、lambda表达式和匿名内部类的区别以及lambda表达式的一些局限。

---

#### 再谈Lambda表达式

之前我们用了一个`Hello`接口的例子来阐述lambda表达式的语法：

~~~ java
Hello hello = () -> System.out.println("Hello World!");
~~~

但是仔细看这段代码，是不是觉得有那么些许问题？Java编译器是如何将这段lambda表达式和接口中的`sayHello`抽象方法关联在一起的？

之前的推断描述是：

1. ()：表示接收0个参数，或者表达为方法不带参数。
2. ->：指明方法块。 无实际意义，估计是装饰语法？
3. System.out.println("Hello World!")：为代码块实际的逻辑，并且编译器能够判断返回类型为void。

由于这三项条件符合`Hello`接口中的`sayHello`方法声明，所以能够这样绑定。

如果`Hello`接口改成这样呢：

~~~ java
public interface Hello {
    void sayHello();
    void sayGoodBye();
}
~~~

多出一个`sayGoodBye`的抽象方法，除了方法名称不一样以外，其他声明都是一样的！那这样的接口能够通过lambda表达式来实现吗？

很遗憾，确实不能，因为Java编译器确实无法知道你所对应的lambda表达式是哪个方法的实现。

那么我们再看，把`sayGoodBye`改成带参数的方法呢：

~~~ java
public interface Hello {
    void sayHello();
    void sayGoodBye(String content);
}
~~~

真是非常遗憾，这样的接口类，也无法用lambda表达式来实现。

***因为我们知道，一个接口如果要被实现，则它的所有抽象方法都必须被实现，而Java又需要兼容这些老的机制，所以即使新版本中新增了lambda表达式语法，也无法脱离这个老版本的规则。***

***然而我们又知道，lambda表达式受其语法本身限制，又只能针对接口中的某一个方法进行实现，所以对于接口中有两个抽象方法的，都无法使用lambda表达式来实现。***（lambda本身的出现就是为了满足函数式编程的范式，而将接口中所有的抽象方法都实现的规则是来自面向对象编程范式，所以lambda的出现，其本身的目的就不是来实现一个接口的，而实现一个接口则是lambda表达式的手段而已）

所以lambda表达式只能绑定至那些**只有一个**抽象方法的接口类型上，而Java中还对这种接口给了一个特定的注解：`@FunctionalInterface`，顾名思义就是“函数式编程接口”。

那我们修改一下`Hello`接口，将`sayGoodBye`方法提供一个默认实现，这样这个`Hello`接口又是一个能供通过lambda表达式来实现的函数式编程接口了：

~~~ java
@FunctionalInterface
public interface Hello {
    void sayHello();
    // 提供一个默认实现
    default void sayGoodBye(String content) {
        System.out.println("Hello " + content + "!");
    }
}
~~~

**那么写到这里，我们可以看到lambda表达式和匿名内部类的一个最主要的区别：**

- 匿名内部类能够为任意接口提供实现，无论该接口有多少个抽象方法，而lambda表达式不能，只能为只有一个抽象方法的接口提供实现

---

**讲了这么多，我们回到lambda语法，来总结一下lambda中针对不同情况的写法：**

- 无参数的抽象方法： **`() -> { code block }`**  或者 **`() -> code line`**
~~~ java
public interface NoParameter {
    void Method();
}
~~~
~~~ java
NoParameter noParam = () -> System.out.println("no parameter");
~~~

- 带一个参数的抽象方法： **`(x) -> { code block }`** 可以简写成 **`x -> { code block }`** 或者 **`(x) -> code line`**
~~~ java
public interface OneParameter {
    void Method(Object obj);
} 
~~~
~~~ java
OneParameter oneParam = o -> System.out.println(o.getClass());
~~~


- 带多个参数的抽象方法： **`(x1, ..., xn) -> { code block }`** 或者 **`(x1, ..., xn) -> code line`**
~~~ java
public interface MultiParameters {
    void Method(Object o1, Object o2);
}
~~~
~~~ java
MultiParameters multiParams = (o1, o2) -> 
    System.out.println(o1.getClass() + " | " + o2.getClass());
~~~

- 返回类型由代码块中的返回类型来决定，如果代码块中存在多行，则需要显示使用关键字return。
~~~ java
public interface WithReturnValue {
    Integer MethodSum(Integer... ints);
}
~~~
~~~ java
WithReturnValue withRetValue = ints -> {
    int sum = 0;
    for (int i : ints) {
        sum += i;
    }
    return sum;
} 
~~~

---

#### 后话

至此，我觉得Java中lambda表达式的语法以及语法的理解方法都已经介绍完成了。

当然函数式编程的内容还有很多很多，后面有机会的话还会继续分享。