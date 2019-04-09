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

> ThreadLoacl解决的问题是多线程状态下，单个线程的变量共享。

认清线程同步和ThreadLocal的区别

1. 线程同步是让多个线程共享一个变量
2. ThreadLocal是为每一个线程创建一个自己所有的并且独立于其他线程的副本。

看完源码之后我总结出的几点：
1. ThreadLocals是一个ThreadLocalMap类型的，它是属于Thread类。就是用来和ThreadLocal来关联，初始值为null。
2. ThreadMap是ThreadLocal的内部类，通过操作Entity来存储数据。
3. ThreadLocal是以线程为单个作用域进行数据共享。

先来从ThreadLocal看起：可以看到ThreadLocalMap是ThreadLocal的一个子类。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409154129483.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

打开ThreadLocalMap你会看见里面还嵌套了一个内部类，那就是Entity，它的功能就是以ThreadLocal为键来存储一个值。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409154455400.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

它从父类中拿到ThreadLocal，虽说它的属于ThreadLocalMap，但是它的最终父类为ThreadLocal。因此传入的Key就是当前的ThreadLocal。并且继承于弱引用，GC就可能被清理（关于这部分的GC在以后再详细解说）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409154614289.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

可以从ThreadLocalMap的构造方法中看出它给子类的Entity间接的把Thread Local传了进去，这种是创建一个最初的ThredLocalMap。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409155110571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

如果父类映射允许继承的话，就可以创建一个给定的父映射的自映射

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409155535399.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

采用开放定址法，因此key的散列值和数组索引并不完全对应，先取一个探测数（key），如果key是要找的元素就返回否则就return getEntityAfterMiss.

![在这里插入图片描述](https://img-blog.csdnimg.cn/201904091557188.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

来到getEntityAfterMiss，在没有找到的时候用它的哈希槽。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409160514911.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

根据负载因子来进行rehash

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409160910425.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

ThreadLocalMap就看到这，现在进入ThreadLoacl，先来看看get方法，根据当前线程获得ThreadLocal，然后获取Entity根据ThreadLocal，并且从Entity取存的value。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409161037389.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

getmap方法是根据Thread和ThreadLocalMap关联来获取的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409161217264.png)

如果get操作时发现根据currentThread不能获取到ThreadLocalMap的话就返回一个setInitialValue，然后进入到setInitalValue，先进行对ThreadLocalMap进行判空操作，如果ThreadLocalMap不为空那就设置初值，如果为空的话那就先创建一个ThreadLocalMap

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409173228494.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

创建一个新的ThreadLocalMap，也就是说每一个线程刚开始使用ThreadLocal的时候都要先在这里创建一个新的ThreadLocalMap并且赋值给Thread里的threadLocals。因此就可以说threadLocals就是在Thread中对ThreadLocalMap的一个引用。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019040917453440.png)

set操作逻辑也和上边大致相似，很简单。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409174932828.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

remove方法，是删除当前线程的ThreadLocalMap。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409175013844.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

这个是创建一个子映射根据传入的父映射

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409175643660.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)


下边就是应用的实例

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