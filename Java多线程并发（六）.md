# 使用Lock和AQS实现自定义的互斥锁和重入锁
> 前言：在多线程并发中锁是非常重要的一部分，自己自定义实现锁有利于了解设计者思想，可以更好的学习，也有助于理解AQS的一些同步组件，如CountDownLatch，Semaphore，CyclicBarrier，ReentrantLock等。
## 使用Lock来实现锁
使用Lock实现锁很简单，只需要实现Lock接口，并对接口进行填补。
### 互斥锁
主要思想就是设置一个状态量，根据状态量来判断是否锁被占用，如果被占用就wait（），如果没被占用就使用。
#### 定义一个互斥锁
```
//自定义锁
public class Mylock1 implements Lock {

    //设置初值表示锁状态是开的
    private boolean isLock = false;

    //为了处理多线程，加上Synchronized
    @Override
    public synchronized void lock() {

        //使用while自旋判断，如果锁被占用那就wait（）
        while (isLock) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        //锁没被占用就自然执行到这里了，设置为锁被占用
        isLock = true;
    }

  
    @Override
    public synchronized void unlock() {
        //解锁并唤醒下一个执行加锁区域的线程
        isLock = false;
        notify();
    }
}

```
#### 使用这个互斥锁
```
public class UseLock1 {

    Mylock1 mylock1 = new Mylock1();
    private int value = 0;
    
    //使用锁来控制值的安全更新
    public int getNext() {
        mylock1.lock();
        value++;
        value++;
        mylock1.unlock();
        return value;
    }

    public static void main(String[] args) {

        UseLock1 lock1 = new UseLock1();
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                while (true) {
                    System.out.println(lock1.getNext());
                }
            }).start();

        }
    }
}
```
### 可重入锁
可重入锁的实现就是在互斥锁的基础上加上线程判断，并且使用一个state来记录锁的层数，以便于释放锁时的判断。
#### 定义一个可重入锁
```
public class MyLock2 implements Lock {
    //锁标志
    private boolean isLock = false;
    //那个线程享有这个锁
    ‘private Thread lockBy = null;
    //锁的层数
    private int lockCount = 0;

    @Override
    public synchronized void lock() {
        
        Thread currentThread = Thread.currentThread();
        //如果锁已经被使用了，但是不是当前线程，那就去等待
        while (isLock && lockBy != currentThread) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        //到这就说明，锁没被使用或者锁被使用了并且是当前的线程，那就lockCount++；
        isLock = true;
        //这一步是为了满足所有要求
        lockBy = currentThread;
        lockCount++;
    }


    //必须要加synchronized
    @Override
    public synchronized void unlock() {
         //如果是多层锁，每次减一层，直到没有锁
         //就设置isLock=false让其他线程可以用，设置lockBy=null（其实不设置也可以，为了代码规范）
        if (lockBy == Thread.currentThread()) {
            lockCount--;
            if (lockCount == 0) {
                notify();
                isLock = false;
                lockBy = null;
            }
        }

    }
}

```

#### 使用这个可重入锁
多层嵌套的使用锁（在单线程中测试），通过打印来看是不是可重入的锁，当然层也是没问题的（测试使用双层）
```
public class UserLock2 {

    private int value = 0;
    MyLock2 lock2 = new MyLock2();

    //里面嵌套这lockB
    public void lockA() {
        lock2.lock();
        System.out.println("LockA"+"第"+(++value)+"层");
        lockB();
        lock2.unlock();
        System.out.println("解锁第"+(value--)+"层");

    }
    //为了给lockA嵌套测试使用
    public void lockB() {
        lock2.lock();
        System.out.println("LockB"+"第"+(++value)+"层");
        lock2.unlock();
        System.out.println("解锁第"+(value--)+"层");

    }

    public static void main(String[] args) {
        UserLock2 userLock2 = new UserLock2();
        userLock2.lockA();

    }
}

```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190518232554246.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)
## 使用AQS实现锁
> 什么是AQS相信你看这个的时候就已经有过了解了吧，因此就不做过多的解释。主要说一些会用到的，使用AQS的方法是继承并重写自己需要的方法，state（int）AQS里的状态量，可以通过AQS的方法对它进行修改和获取，可以实现排他锁和共享锁。

JDK中的使用规范
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190518233500854.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

### 互斥锁
#### 自定义一个互斥锁
主要思路还是围绕着这个state变量，就和我们在前面自定义的状态变量相同作用。
```
public class MyLock3 implements Lock {
    
    //使用一个内部类继承AQS，实现方法后，这个内部类就可以看作为实现锁的帮助类
    private class Helper extends AbstractQueuedSynchronizer {
        
        //1.获取状态
        //2.拿到状态，CAS判断(双重检测)，并设置为独占锁
        @Override
        protected boolean tryAcquire(int arg) {

            int state = getState();
            //使用双重检测，第一步检测并不能有效的仅使得一个线程进入（在同步的时间可能进来很多线程）
            if (state == 0) {
                //第二步进行CAS进一步筛选，第一层锁可以减轻CAS负担
                if (compareAndSetState(0, arg)) {
                    //设置独占锁的拥有者
                    setExclusiveOwnerThread(Thread.currentThread());
                    //获取成功
                    return true;
                }
            }
            //如果state不为0，那就获取失败
            return false;
        }

        //1.先看是不是当前线程，如果不是就不能释放，抛出异常

        //2.如果是的话，判断state，如果state-1（arg）=0，那就可以释放

        @Override
        protected boolean tryRelease(int arg) {
            //锁的获取和释放需要一一对应，如果不是获取锁的线程来释放锁就抛异常
            if(Thread.currentThread() != getExclusiveOwnerThread()){
                throw new RuntimeException();
            }
            //arg决定释放锁的层数
            int state = getState() - arg;
            //更新状态值
            setState(state);
            //如果state为0，那就完全释放锁，供其他线程使用
            if(state == 0){
                setExclusiveOwnerThread(null);
                return true;
            }
            //如果没有完全释放还是返回false
            return false;

        }
        //ConditionObject不能在外部使用是AQS里的
        Condition newCondition() {
            return new ConditionObject();
        }
    }

    //实例化Helper类，供外部类使用
   private Helper helper = new Helper();

    @Override
    public void lock() {
        helper.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        helper.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return helper.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return helper.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        helper.release(1);
    }

    @Override
    public Condition newCondition() {
        return helper.newCondition();
    }

}

```
#### 使用这个互斥锁
```
public class UseLock3 {

    private int value;

    MyLock3 lock = new MyLock3();

    //下边的代码没加锁会出现线程安全性问题
    /*
     public int getNext() {
         try {
             Thread.sleep(100);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return value++;
     }
     */
     
    //使用锁来安全更新
    public int getNext() {
        lock.lock();
        try {
            Thread.sleep(100);
            return value++;
        } catch (InterruptedException e) {
            throw new RuntimeException();
        }finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {

        UseLock3 lock3 = new UseLock3();
    
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                while (true) {
                    System.out.println(Thread.currentThread() + " " + lock3.getNext());
                }
            }).start();
        }
    }
}

```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190518234936990.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)
### 可重入锁
可重入锁思想和前面使用Lock的思想大致相同就是换了个实现方式而已

#### 自定义一个可重入锁
```
public class MyLock4 implements Lock {


    private class Helper extends AbstractQueuedSynchronizer {


        //1.获取状态

        //2.拿到状态，CAS判断(双重检测)，并设置为独占锁

        //3.判断是不是当前线程，要是当前线程的话就可重入

        @Override
        protected boolean tryAcquire(int arg) {

            Thread t = Thread.currentThread();
            int state = getState();
            if (state == 0) {
                if (compareAndSetState(0, arg)) {
                    setExclusiveOwnerThread(Thread.currentThread());
                    return true;
                }
            }else if (t == getExclusiveOwnerThread()){
                setState(state+1);
                return true;
            }
            return false;
        }


        //1.先看是不是当前线程，如果不是就不能释放，抛出异常

        //2.如果是的话，判断state，如果state-1（arg）=0，那就可以释放
        
        @Override
        protected boolean tryRelease(int arg) {
            //锁的获取和释放需要一一对应，那么调用这个方法的线程，一定是当前线程
            if(Thread.currentThread() != getExclusiveOwnerThread()){
                throw new RuntimeException();
            }
            int state = getState() - arg;
            setState(state);
            if(state == 0){
                setExclusiveOwnerThread(null);
                return true;
            }
            return false;

        }
        //ConditionObject不能在外部使用是AQS里的
        Condition newCondition() {
            return new ConditionObject();
        }
    }

    Helper helper = new Helper();

    @Override
    public void lock() {
        helper.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        helper.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return helper.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return helper.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        helper.release(1);
    }

    @Override
    public Condition newCondition() {
        return helper.newCondition();
    }

}

```
#### 使用这个可重入锁
```
public class UseLock4 {

    private int value;

    MyLock4 lock = new MyLock4();

    public int getNext() {
        lock.lock();
        try {
            Thread.sleep(100);
            return value++;
        } catch (InterruptedException e) {
            throw new RuntimeException();
        } finally {
            lock.unlock();
        }

    }

    private void lockA() {
        lock.lock();
        System.out.println("lockA");
        lockB();
        lock.unlock();
    }

    private void lockB() {
        lock.lock();
        System.out.println("lockB");
        lock.unlock();
    }

    public static void main(String[] args) {

        UseLock4 lock3 = new UseLock4();
        lock3.lockA();
    }
}

```
> 这就是使用Lock和AQS实现简单的锁，实现锁不是目的，真实目的是可以更好更深入的理解其原理和设计思想，以及对应粒度的控制等。
