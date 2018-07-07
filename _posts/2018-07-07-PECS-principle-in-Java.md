---
title:  "PECS principle in Java"
layout: post
date:   2018-07-07
author: "bing5tui3"
categories: java
excerpt_separator: <!--more-->
---

工作上碰到一个需要异步处理任务的功能，所以想基于队列来实现，由守护线程来从队列中消费消息。 
由于队列容器内的对象需要有一些公共机制，以使异步处理模块能够不关心具体的实现，这就需要使用到泛型队列了。
本文记录一下泛型队列的PECS原则的实践。

<!--more-->

#### PECS

Producer Extends Consumer Super

#### 一个基于基类的容器例子

`? extends BaseClass` 规定了一个上界，表示一个容器类能够装载的对象的类可以是 `BaseClass` 或者 `BaseClass` 的子类。 

我们先创建一个基类 `BaseClass` 和一个子类 `SubClass`，然后试着向容器添加一些内容（*这里用了ArrayList*）：

~~~ java
// 基类
class BaseClass {}

// 子类
class SubClass extend BaseClass {}

//
public class Main {

    public static void main(String[] args) {
		
        // 一个基于BaseClass的ArrayList的对象添加   --  no problem
        BaseClass base = new BaseClass();
        List<BaseClass> listNormal = new ArrayList<>();
        listNormal.add(base);

        // listNormal也能添加SubClass的实例
        SubClass sub = new SubClass();
        listNormal.add(sub);

        // 编译无问题，但是运行时如果listNormal.get(0)并不是一个SubClass的实例，则会产生ClassCastException
        SubClass subRes = (SubClass) listNormal.get(0);

    }
}
~~~

如代码中注释的，基于 `BaseClass` 的容器虽然能够容下所有 `BaseClass` 和 `BaseClass` 子类的实例，但是在获取的时候如果里面的类型并非你所想，则在进行类型转换的时候会出现 `ClassCastException`。

所以在这种情况下，假设 `methodA` 会产生 `AClass extends BaseClass` 的实例，并且使用容器 `arrayA : ArrayList<BaseClass>` ，`methodB` 会产生 `BClass extends BaseClass` 的实例，并且使用 `arrayB : ArrayList<BaseClass>`。
如果 `methodA` 不小心将它产生的实例放入了 `arrayB` 中，那么消费 `arrayB` 中元素的某个方法将有可能出现错误，因为那个方法将有可能编写类型转换的代码： `BClass b = (BClass) arrayB.get(n)`。

可能有些疑问的就是，为什么不给 `methodA` 的容器定义为 `arrayA : ArrayList<AClass>`，而给 `methodB` 的容器定义为 `arrayB : ArrayList<BClass>` ，这样即使 `methdodA` 想要将其产生的实例放入 `arrayB` 也无法实现。

我们知道很多时候我们可能会要编写一些公共的代码，由这套公共代码驱动完成一套业务流程，而代码中可能需要适配不同的子业务流程，这些子业务流程的总流程都相同，只是某些地方各有不同，而这些不同的部分可能就是之前的 `methodA`，`methodB`，并且这些不同的方法可能都会用到一套公用的容器，这套容器又在公共的代码框架中维护，所以就会产生容器由外部代码统一维护，而不同的实现方法只是获取和使用，无法自行定义。

#### 简单的异步处理类

下面我们写一个简单的异步处理类，它维护一个简单的任务列表和一个与列表对应的任务队列，每个任务由一个线程守护，从队列中读取消息实例，然后交给任务处理其处理。

###### 首先是一个任务类的基类，需要有一个实例域是任务名，后续用于读取后根据任务名分配对应的任务处理器：
~~~ java
// BaseTaskModel 抽象类，不能直接被实体化，每个任务要求实现自己的TaskModel
public abstarct class BaseTaskModel {
    
    private String taskName;

    public String getTaskName() { return this.taskName; }

    public void setTaskName(String taskName) { this.taskName = taskName; }

}
~~~

###### 任务A的任务类 `ATaskModel.java`：
~~~ java
// ATaskModel
public class ATaskModel extends BaseTaskModel {
    // 任务A的任务类携带消息messageA
    private MessageA messageA;

    public MessageA getMessageA() { return this.messageA; }

    public void setMessageA(MessageA messageA) { this.messageA = messageA};

}
~~~

###### 任务A的任务类中携带的消息类 `MessageA.java`：
~~~ java
// MessageA
public class MessageA {}
~~~

###### 任务B的任务类 `BTaskModel.java` ：
~~~ java
// BTaskModel
public class BTaskModel extends BaseTaskModel {
    // 任务B的任务类携带消息messageB
    private MessageB messageB;

    public MessageB getMessageB() { return this.messageB; }

    public void setMessageB(MessageB messageB) { this.messageB = messageB; }

}
~~~

###### 任务B的任务类中携带的消息类 `MessageB.java`：
~~~ java
// MessageB
public class MessageB {}
~~~

###### 任务处理器接口 `TaskHandler.java`
~~~ java
public interface TaskHandler {

    // 处理任务
    void handleTask(BaseTaskModel taskModel);
    // 处理异常
    void handleException(BaseTaskModel taskModel);

}
~~~

###### 任务A的任务处理器类 `ATaskHandler.java`：
~~~ java
public class ATaskHandler implements TaskHandler {

    public void handleTask(BaseTaskModel taskModel) { 
        // 强制转换成A的任务实例
        ATaskModel a = (ATaskModel) taskModel;
        // do something 
    }

    public void handleException(BaseTaskModel taskModel) {
        // do something
    }

}
~~~

###### 任务B的任务处理器类 `BTaskHandler.java`：
~~~ java
public class BTaskHandler implements TaskHandler {

    public void handleTask(BaseTaskModel taskModel) { 
        // 强制转换成B的任务实例
        BTaskModel b = (BTaskModel) taskModel;
        // do something 
    }

    public void handleException(BaseTaskModel taskModel) {
        // do something
    }

}
~~~

###### 异步任务处理类 `AsynchronousTaskProcesser.java`
~~~ java
public class AsynchronousTaskProcesser {

    // 任务列表  任务名 -> 任务处理器
    private Map<String, TaskHandler> taskNameMap = new HashMap<>();
    // 任务队列  任务名 -> 队列
    private Map<String, LinkedBlockingQueue<BaseTaskModel taskModel>> taskQueueMap = new HashMap<>();

    private ExecutorService executorService;

    private TaskHandler ATaskHandler = new ATaskHandler();

    private TaskHandler BTaskHandler = new BTaskHandler();

    // 提供一个公有的初始化方法
    public void init() {

        // 初始化任务列表
        taskNameMap.put("taskA", ATaskHandler);
        taskNameMap.put("taskB", BTaskHandler);
        // 初始化任务队列
        taskQueueMap.put("taskA", new LinkedBlockingQueue<>());
        taskQueueMap.put("taskB", new LinkedBlockingQueue<>());

        // 根据队列数量设置线程池线程
        executorService = Executors.newFixedThreadPool(taskQueueMap.size());

        for (Map.Entry<String, LinkedBlockingQUeue<BaseTaskModel> entry : taskQueueMap.entrySet()) {
            executorService.submit(
                () -> {
                    String taskName = entry.getKey();

                    TaskHandler handler = taskMap.get(taskName);

                    LinkedBlockingQueue<BaseTaskModel> queue = entry.getValue();

                    while (true) {
                        BaseTaskModel taskModel = null;
                        try {

                            taskModel = queue.take();
                            // 取出任务实例后交给任务处理器处理任务
                            handler.handleTask(taskModel);

                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } catch (Exception e) {
                            e.printStackTrace();

                            // 错误发生后交给任务处理器处理错误
                            if (taskModel != null) {
                                handler.handleException(taskModel);
                            }
                        }
                    }
                }
            );
        }

    }

    // 提供一个添加任务消息的公有方法
    public boolean add(BaseTaskModel taskModel) {
        if (taskModel != null) {
            LinkedBlockingQueue<BaseTaskModel> queue = taskQueueMap.get(taskModel.getTaskName());
            if (queue != null) {
                return queue.offer(taskModel);
            }
        }
        return false;
    }

}
~~~

#### <? extends BaseClass>


#### ? super BaseClass


#### 实现一个泛型的队列