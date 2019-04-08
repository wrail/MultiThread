# 线程范围内共享

> 我们可以让全部线程共享一个变量或方法或对象，这个很easy，哪有没有想到过，每一个线程都有自己的内部共享，为什么会有这样的需求呢？因为每个进程都有自己不同的状态，有这样一个变量或者对象，不仅有利于上下文切换，也有利于数据的隔离。

## 手动撸一个简易的共享实例，从中悟出道理

> 要手动写一个共享实例，先要详细想怎么让不同的线程区别开来呢？使用什么结构来存储多个状态？

来看看下边的代码，再思考思考这个问题。

```
package com.company.MyThread;


import java.util.HashMap;
import java.util.Map;
import java.util.Random;

/**
 * 线程内共享
 */

public class ShareThread {
    
    public static final Map<Thread, Integer> threadData = new HashMap<Thread, Integer>();

    public static void main(String[] args) {

        for (int i = 0; i < 3; i++) {
            new Thread(
                    new Runnable() {
                        @Override
                        public void run() {
                            int data = new Random().nextInt();
                            threadData.put(Thread.currentThread(), data);
                            System.out.println("当前线程：" + Thread.currentThread().getName() + "共享data = " + data);
                            new A().get();
                            new B().get();
                        }
                    }
            ).start();
            System.out.println();
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    static class A {
        public void get() {
            System.out.println("当前线程：" + Thread.currentThread() + "A get data = " + threadData.get(Thread.currentThread()));
        }
    }

    static class B {
        public void get() {
            System.out.println("当前线程：" + Thread.currentThread() + "B get data = " + threadData.get(Thread.currentThread()));
        }
    }



}


```

看到上边的实例有没有发现是怎样实现这一结构的？

众所周知吗，每一个线程都是唯一的，因此采用currentThread来作为唯一标识

为了要存储多个状态的不同数据，我们采用了Map来实现这一功能，并且Map也是一个static final类型的。

> 就这样使用了一个全局不变的结构来存储线程状态。

根据人类比较懒的习惯，这总冗余的作风，可不是一个正经程序员干的事情，因此在JDK对此做了比较不错的实现，那就是ThreadLocal。

> 那有的人问那怎么才能存取一个数据，那是不是太low了，记住Java的面向对象，可以在Map中叠加自定义对象。

## 使用ThreadLocal实现线程范围内共享

> 下边就针对有人的想法，使用一个包装类型来实现线程范围内共享

在进行之前来先简单从源码中了解一下ThreadLocal，你会发现它的实现和我们前面的实现惊人的相似。

先看看这些介绍否则后边可能有点懵

> ThreadLocal类用于存储以线程为作用域的数据，线程之间数据隔离。 

> ThreadLocalMap类是ThreadLocal的静态内部类，通过操作Entry来存储数据。 

> Thread类，ThreadLocals是线程存放多个ThreadLocal实例的。。 


ThreadLoacl的get方法，很熟悉吧，它根据t（currentThread）来获取一个ThreadLoaclMap，如果这个mao非空，就说明当前线程由数据，因此根据this（也就是currentThread）来生成一个Entity，如果Entity不空的话，就返回。如果根据this（currentThread）筛选map为空（也就是map中没有当前线程的东西），那它就return setInitialValue（），来根据this（currentThread）给ThreadLocalMap中加一条当前线程的信息。

> ThreadLocal首先通过当前线程t得到threadLocals，然后把自身this作为key，从threadLocals中得到存储在线程本地的相关的数据

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190408223437439.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190408231744598.png)


> 通过一个弱引用维持关系

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190408225152535.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)


进入setInitialValue 看一看，可以看到它再次根据currentThread判断的是根据t来判断是否存在这样一个map，存在的话就初始化value位null，不存在就创建一个

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190408224612761.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

根据传入的currentThread来创建一个map加入到保存threadLocals（保存当前线程一些线程独有的信息）的集合里。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190408225435682.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)


> 一个 ThreadLocal对象只对应一个值对象 ,多个ThreadLocal对象和相应的值存在线程中类型为ThreadLocalMap类型的threadLocals变量中

说了这么多，心里都糊涂了吧，看看代码提提神！

```

public class ThreadLocal {

    public static void main(String[] args) {

        for (int i = 0; i < 3; i++) {
            new Thread(
                    new Runnable() {
                        @Override
                        public void run() {
                            int data = new Random().nextInt();
                            MyThreadScopeData instance = MyThreadScopeData.getThreadInstance();
                            instance.setName(Thread.currentThread().getName());
                            instance.setPassword("PSWD" + data);
                            new A().get();
                            new B().get();
                        }
                    }
            ).start();
            System.out.println();
        }
    }

    static class A {
        public void get() {

            MyThreadScopeData threadInstance = MyThreadScopeData.getThreadInstance();
            threadInstance.setName(threadInstance.getName());
            threadInstance.setPassword(threadInstance.getPassword());
            System.out.println("当前线程：" + Thread.currentThread() +
                    "A get name = " + threadInstance.getName() + "    " + "A get password = " + threadInstance.getPassword());
        }
    }

    static class B {
        public void get() {

            MyThreadScopeData threadInstance = MyThreadScopeData.getThreadInstance();
            threadInstance.setName(threadInstance.getName());
            threadInstance.setPassword(threadInstance.getPassword());
            System.out.println("当前线程：" + Thread.currentThread() +
                    "B get name = " + threadInstance.getName() + "    " + "B get password = " + threadInstance.getPassword());
        }
    }


}

class MyThreadScopeData {

    private static java.lang.ThreadLocal<MyThreadScopeData> map = new java.lang.ThreadLocal<MyThreadScopeData>();

    public static MyThreadScopeData getThreadInstance() {
        MyThreadScopeData instance = map.get();

        if (map.get() == null) {
            instance = new MyThreadScopeData();
            map.set(instance);
//            return instance;
        }
        return instance;
    }

//    static MyThreadScopeData instance = null;

    private String name;
    private String password;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}


```

 使用ThreadLocal在对象内，然后定义一些对象属性name，和password，并加上getter和setter，使用方法通过在MyThreadScopeData类中创建一个入口也就是getInstance，然后在线程中调用入口就ok了。