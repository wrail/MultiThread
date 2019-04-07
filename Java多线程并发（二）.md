## Synchronized关键字

> 这个关键字主要是用于线程同步，通过加锁来实现互斥操作。

Synchronized的使用有两种方式：

一种是在代码块中使用，另一种就是在方法上使用。

### 在代码块上使用 

```

public class TestSynchronized {

    public static void main(String[] args) {
        new TestSynchronized().init();

    }

    class TellName {

        //如果要同步整个代码方法就给方法前加synchronized（同步互斥）
        public void printName(String name) {

        String lock = "";
        int len = name.length();
        synchronized (lock) {
//      synchronized (this) {
               for (int i = 0; i < len; i++) {
                    System.out.print(name.charAt(i));
                }
                System.out.println();
            }
        }
    }

    public void init() {
        final TellName tellName = new TellName();

        while (true) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    tellName.printName("zhangsan");

                }
            }).start();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    tellName.printName("lisi");
                }
            }).start();
        }
    }

}




```

这段程序是让两个线程重复进行字符打印（把一个姓名拆分为一个一个的字符）在不加锁的情况下，可以在下图看到在下边第四行就出错了，发现并不安全。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406141002425.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

然后我们加上锁如上边的代码块加上锁后就不会出现这种明面上的错误了


![在这里插入图片描述](https://img-blog.csdnimg.cn/2019040614140247.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

> 注意的是使用Synchronized代码块传入的必须是**同一个对象**（因为只有访问同一个对象才会出现互斥问题），如果我传个name，就会出错（打印结果出错），因此我定义了一个字符串是属于这个类的，是共有的，因此就可以直接写为this。

### 在方法上

```
 public static synchronized void print(String name) {

        Integer len = name.length();
        for (int i = 0; i < len; i++) {
            System.out.print(name.charAt(i));
        }
        System.out.println();
    }

```

只需要加一个synchronized关键字就可以了。

> synchronized使用很简单，下来我们又有问题了，如果我们两个线程一个使用代码块级别的synchronized另外一个使用方法级别的，它们还会同步吗？

> 经测试过不加static的方法级别synchornized可以同步，加static不会同步，因此以加static为例。

```

public class TestSynchronized {

    public static void main(String[] args) {
        new TestSynchronized().init();

    }

    static class TellName {

        //如果要同步整个代码方法就给方法前加synchronized（同步互斥）
        public void printName(String name) {

            int len = name.length();
            synchronized (this) {
                for (int i = 0; i < len; i++) {
                    System.out.print(name.charAt(i));
                }
                System.out.println();
            }
        }

        public static synchronized void print(String name) {

            Integer len = name.length();
            for (int i = 0; i < len; i++) {
                System.out.print(name.charAt(i));
            }
            System.out.println();
        }
    }

    public void init() {
        final TellName tellName = new TellName();

        while (true) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    tellName.printName("zhangsan");

                }
            }).start();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    tellName.print("lisi");
                }
            }).start();
        }
    }


}




```

然后输出打印结果很明显出现了我们不想看到的结果。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190406175010337.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

> 那这个问题改如何解决呢？

把synchronized里的this换为类.class。

```
 ...
 synchronized (TellName.class) {
 ...
  
```


为什么必须这样才可以呢？因为静态方法不能实例化对象，因此在参数中的this就对它不起作用，然而我们想到编译的class文件也可以看作一个对象，把它传进去就可以对这个类通用了。

> 下边就用一个实例来总结线程同步互斥

子线程循环10次，主线程循环5次，就这样循环20次

```
 public class MyThreadTest {

    public static void main(String[] args) {
        new MyThreadTest().init();
    }

    public void init() {

        final DoExchange doExchange = new DoExchange();

        new Thread(
                new Runnable() {
                    @Override
                    public void run() {
                        for (int i = 0; i < 20; i++) {
                            doExchange.sub(i);
                        }
                    }
                }
        ).start;
        new Thread(
                new Runnable() {
                    @Override
                    public void run() {
                        for (int i = 0; i < 20; i++) {
                            doExchange.main(i);
                        }
                    }
                }
        ).start();

    }

    class DoExchange {
         private boolean flag = true;//true：子线程运行

        public synchronized void sub(int i) {

            while (!flag) {   //while要优于if，防止伪唤醒
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            for (int j = 0; j < 10; j++) {
                System.out.println("sub" + i + "---->" + j);
            }
            flag = false;
            this.notify();

        }

        public synchronized void main(int i) {

            while (flag){
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            for (int j = 0; j < 5; j++) {
                System.out.println("main" + i + "---->" + j);
            }
            flag = true;
            this.notify();
        }


    }


}


```

在这里再强调一下run和start的区别：

* run是进入运行状态

    > 多个线程同时run的话会出现相互阻塞，有一个是run，其余的调用start的话，run会先执行，然后调用start的再进行竞争。

* start是进入就绪状态

    > 就绪不等于运行。

> 在这里需要考察的东西第一步先要线程互斥(synchronzied)，第二部要线程同步(flag+wait+notify).



> 在用到共同数据或者同步锁或相同类型的加密解密算法的若干方法一般都设计在一个类上。比如我上边将主线程和子线程打印放在同一个类中。

