---
title: Dynamic Proxy In Java
layout: post
date:   2018-07-15
author: "bing5tui3"
tags: [java]
categories: [java]
excerpt_separator: <!--more-->
---

最近对Java的动态代理产生了兴趣，所以花了点时间研究了一下，本文尽量做到讲解到位。

<!--more-->

#### 动态代理示例

首先我们记录一下一个Java类的动态代理代码如何编写，写完我们在一步步看其原理是怎样的。好的，下面我们就开始编写一个动态代理的例子。

假设我们是一家咖啡店叫星法克，客人可以通过电话订咖啡，订完咖啡我们可以派送将咖啡送到客人手上。所以我们的星法克有个接口，我们假设叫`Deliverable`，包含一个`deliver() - 派送`方法：

~~~ java
package com.bing5tui3.dynamic.proxy;

public interface Deliverable {
    // 派送
    void deliver();
}
~~~

我们星法克咖啡店实现了这个接口：

~~~ java
package com.bing5tui3.dynamic.proxy;

public class StarFxxcks implements Deliverable {
    @Override
    public void deliver() {
        System.out.println("coffee starts delivering...");
        System.out.println("coffee delivered");
    }
}
~~~

如果我们星法克直接派送，那么代码将是如下：

~~~ java
package com.bing5tui3.dynamic.proxy;

public class DeliveryCase {

    public static void main(String args[]) {
    	// 建立一家新的星法克店铺
        StarFxxcks starF$cks = new StarFxxcks();
        // 由星法克直接派送
        starFxxcks.deliver();
    }

}
~~~

这个时候我们星法克发现直接派送成本太高，不如让【饿了啊】公司帮忙派送，饿了啊公司是一家专门做外卖的公司，它的成立依赖于那些有派送功能的店（如果一家店铺没有派送功能，那么说明它的产品可能不适合被派送），饿了啊很乐意帮忙派送，但是要求每次派送收取3元人民币作为派送费。所以我们饿了啊的平台实现如下：

~~~ java
package com.bing5tui3.dynamic.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class ErrrLeAa implements InvocationHandler {

    private Object object;

    public ErrrLeAa(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 饿了啊收取3元派送费
        System.out.println("ErrrLeAa charges 3 RMB");

        Object result = method.invoke(object, args);

        return result;
    }

}
~~~
饿了啊接收一个Object的实例，该Object可以认为是一个店铺的实例，该过程类似于注册，我们将我们的星法克咖啡店注册到饿了啊平台。这下咖啡的派送功能就被饿了啊代理了，以后的派送都交给饿了啊平台了。
场景代码如下：

~~~ java
package com.bing5tui3.dynamic.proxy;

import java.lang.reflect.Proxy;

public class DeliveryCase {

    public static void main(String args[]) {
        // 建立一家新的星法克店铺
        StarFxxcks starFxxcks = new StarFxxcks();
        // 由星法克直接派送
        starFxxcks.deliver();

        System.out.println("-------------------- separate line --------------------");

        // 将星法克店铺注册到饿了啊平台
        ErrrLeAa startFxxcks_registered_on_ErrrLeAa = new ErrrLeAa(starFxxcks);
        // 生成一个饿了啊平台的可派送店铺
        Deliverable startFxxcksOnErrrLeAa = (Deliverable) Proxy.newProxyInstance(
                StarFxxcks.class.getClassLoader(),
                new Class[] { Deliverable.class },
                startFxxcks_registered_on_ErrrLeAa
        );
        // 由饿了啊平台进行派送
        startFxxcksOnErrrLeAa.deliver();
    }

}
~~~
输出结果如下：

~~~
coffee starts delivering...
coffee delivered
-------------------- separate line --------------------
ErrrLeAa charges 3 RMB
coffee starts delivering...
coffee delivered
~~~
可以看到饿了啊平台收取了3人民币后进行了派送。

假设又有一家咖啡店法金咖啡想要加入饿了啊平台，但是法金咖啡还有一个功能是希望饿了啊帮忙打广告，怎么办呢？
很遗憾的是，如果你的打广告方法不是接口方法，则饿了啊平台是无法获取的，这个时候饿了啊平台需要定义一个新的接口`Advertiseable`，内涵一个打广告的方法`advertise()`。并且告诉法金咖啡店，你们需要实现这个接口，之后我们饿了啊平台就能够帮你们打广告。

~~~ java
package com.bing5tui3.dynamic.proxy;

public interface Advertisable {
    void advertise();
}
~~~

我们的法金咖啡店实现两个功能，一个是派送，一个是打广告：

~~~ java
package com.bing5tui3.dynamic.proxy;

public class FxxckinCoffee implements Deliverable, Advertisable {
    @Override
    public void advertise() {
        System.out.println("FxxckinCoffee is advertising");
    }

    @Override
    public void deliver() {
        System.out.println("FxxckinCoffee is delivering coffee");
        System.out.println("FxxckinCoffee delivered");
    }
}
~~~

我们将法金咖啡店注册到饿了啊平台（修改之前的`main`方法）：

~~~ java
package com.bing5tui3.dynamic.proxy;

import java.lang.reflect.Proxy;

public class DeliveryCase {

    public static void main(String args[]) {
        // 建立一家新的星法克店铺
        StarFxxcks starFxxcks = new StarFxxcks();
        // 由星法克直接派送
        starFxxcks.deliver();

        System.out.println("-------------------- separate line --------------------");

        // 将星法克店铺注册到饿了啊平台
        ErrrLeAa startFxxcks_registered_on_ErrrLeAa = new ErrrLeAa(starFxxcks);
        // 生成一个饿了啊平台的可派送店铺
        Deliverable startFxxcksOnErrrLeAa = (Deliverable) Proxy.newProxyInstance(
                StarFxxcks.class.getClassLoader(),
                new Class[] { Deliverable.class },
                startFxxcks_registered_on_ErrrLeAa
        );
        // 由饿了啊平台进行派送
        startFxxcksOnErrrLeAa.deliver();

        System.out.println("-------------------- separate line --------------------");

        FxxckinCoffee fxxckinCoffee = new FxxckinCoffee();

        ErrrLeAa fxxckinCoffee_registered_on_ErrrLeAa = new ErrrLeAa(fxxckinCoffee);

        Object fxxckinCoffeeOnErrrLeAa = Proxy.newProxyInstance(
                FxxckinCoffee.class.getClassLoader(),
                new Class[] { Deliverable.class, Advertisable.class },
                fxxckinCoffee_registered_on_ErrrLeAa
        );
        // 法金咖啡店打广告
        ((Advertisable) fxxckinCoffeeOnErrrLeAa).advertise();
        // 法金咖啡店派送
        ((Deliverable) fxxckinCoffeeOnErrrLeAa).deliver();
    }
}
~~~
可以看到输出结果如下：
~~~
coffee starts delivering...
coffee delivered
-------------------- separate line --------------------
ErrrLeAa charges 3 RMB
coffee starts delivering...
coffee delivered
-------------------- separate line --------------------
ErrrLeAa charges 3 RMB
FxxckinCoffee is advertising
ErrrLeAa charges 3 RMB
FxxckinCoffee is delivering coffee
FxxckinCoffee delivered
~~~
