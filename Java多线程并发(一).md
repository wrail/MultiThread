# Java多线程入门

> 什么是线程和进程，它们之间有什么关系？

线程：是程序中单独的顺序控制流，只能分配给程序资源和环境

进程：是执行中的程序，一个进程最少包含一个线程，也可以包含多个线程

单线程：程序中只有一个线程，主方法就是一个线程

多线程：是一个程序中运行多个任务，目的是为更好的使用CPU资源吗，也可以说是多线程索要的资源比单线程多。

## 线程的实现

> 一些简单的线程创建演示很简单就不演示了，在后边分析功能的时候再演示代码

1. 继承Thread

原理就是使用子类来覆盖run方法，父类里实现类Runable接口，也就是间接的实现Runable。

2. 实现Runable接口

这个相比于上边的它更为直接，它直接实现Runable接口，实现它里边的run方法。

3. 既继承Thread，又实现Runable。

> 那如果在一个Thread中同时实现了Runable和重写了Thread的run，那会执行哪一个呢？

```
//下面的例子，会执行Thread的run方法
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("Runable");
            }
        }) {
            public void run() {
                for (int i = 0; i < 100; i++)
                    System.out.println(i);
            }
        }.start();

```

会执行重写Thread的那个run方法。原因很简单，稍微分析一下源码就可以发现，Thread会首先找自己类下的run方法，如果是重写的就直接运行，如果没有重写那么，那它本类中的run就不执行，那它就会区找Runable中的run了。

> 线程通过start方法启动线程。

> 多线程不一定比单线程快，还有可能拖慢程序的进度。那为什么还要多线程下载呢？因为多线程可以利用更多资源，因此下载的更快。

说这些都是热身，接下来深入源码看看多线程的一些常用方法，和启动，初始化等。。可以很好的进行入门。

先从Thread开始

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406092533501.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

上边是Thread的所有构造方法，无参的构造就仅仅分配一个Thread对象，值都是默认值。第二个传入Runable就是给分配一个Thread对象并重写run方法，第三个传入一个Runable和AccessControlContext，AccessControlContext它可以控制当前线程对资源访问的决策。ThreadGroup是一个安全管理者（也就是说它就是Thread的一个集合）。传入String就仅仅是初始化一个Thread对象，并给它指定名字。
后边这几个也可以跟着前面的推出它的意思。

然后进入init方法，它需要一些初始化的信息。Runable，ThreadGroup，name，和stackSize（对栈的分配）等

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406094328565.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)


然后再看看start方法，来解决为什么我们在调用start方法的时候总是单线程执行。看到下图就会明白。并且将它加入到ThreadGroup中（因为初始化线程的时候ThreadGroup不是必须的）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406094816176.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

看完start也就要看run方法了，run的实现很简单，但它就是多线程的入口。这个run方法是从Runable继承过来的，判断条件是当前线程如果不为空，就执行，如果为空，就什么都不做。

因此我们在写Thread的时候需要重写它的run方法才能执行，否则就没反应。如果你没重写，它就target.run调用Runable接口的实现，如果你实现了就调用，没实现的话还是没作用。这也正是为什么重写的run方法优先于Runable对象实现的run方法。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406095124330.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

还有inrerrupt，中断线程，如果是正自我中断会调用checkAccess确定当前运行线程是否具有修改线程的权限，否则就中断。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406100052695.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

这些方法很多，就不一一列举。

接下来进入Runable，Runable就仅仅规定了一个规范。它里面仅仅只有一个run的抽象方法。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406101039367.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

## 传统定时器的使用

> 前几天在看Redis设计与实现的时候，它内部就又很多的定时器，因此说明定时器还是很重要的。

话不多说就进入正题，下边是一个简单的定时器的使用。实现很简单，就是实例化一个定时器，然后调用时间表函数为任务设置定时事件。


```

public class TimerTest {

    public static void main(String[] args) {

        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                System.out.println("Timer Comming!");
            }
        },1000,2000);

        while(true){
            System.out.println(System.currentTimeMillis());
            try {
                Thread.sleep(1000);//保证每次打印时间都是1s为间隔
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }

}


```

然后分析运行结果可以得出：第一次定时任务时长是1s，然后后来就都是两秒。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406104154393.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

```
 new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                System.out.println("Timer Comming!");
            }
        },1000);

```
这个定时器只能使用一次，相当于炸弹炸完后就炸不了了。

> 既然传两个参数可以有单个规律的触发，但我想要第一次次相隔1s，下一次2s，再下一次3s等等进行循环触发，这样就不能使用前面的方法了。

因此还要对代码进行改进,或者要设置flag为全局的就挪出来为静态的

```

public class TraditionalTimerTest {

    public static void main(String[] args) {

        class MyTimerTask extends TimerTask {
            Integer flag = 0;
            Integer time = 0;

            @Override
            public void run() {

                flag++;
                System.out.println("Timer Comming!");
                if (flag%2==0){
                    time = 2000;
                }
                else
                    time = 3000;
                new Timer().schedule(new MyTimerTask(),time);
            }
        }
        new Timer().schedule(new MyTimerTask(),3000);
        while (true) {
            System.out.println(System.currentTimeMillis());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }

}


```

这样就可以根据flag的值可以自行根据需求设定，这样就变的灵活多了，这种设计方法是很重要的，思想比代码重要。当然定时器的使用方式很多，这就不多说了。