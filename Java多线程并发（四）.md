# 基础知识

## 多级缓存

> CPU的处理速度比我们普通I/O速度快的多，怎么更好的，更有效的使用CPU呢？

最开始是一个快速的单级缓存，然后不断发展到多级缓存

因为CPU缓冲区的读取速度比直接访问内存快的多，因此可以在一定程度上缓解这种关系，但是CPU缓冲区一般容量都特别小，那它凭什么可以缓解cpu和内存之间的关系。

### 缓存命中的根据 

> 缓冲区可以缓解CPU和内存之间的关系有什么依据

1. 时间局限性

如果一个数据被访问，它那可能在邻近的实现内被访问到。比如循环访问等。

2. 空间局限性

如果一个数据被访问，那它的临近空间可能在接下俩一段时间被访问到。比如一个数组处于一段连续的内存空间，有了前一个的地址，访问后边的地址速度更快。

> 如果是单个缓存我们不需要考虑它的一致性问题，因为它就是标准，但是如果是多级缓存的话该怎么保存缓存之间的一致性。

### 缓存一致性

> 在多级缓存中各个缓存都有自己的状态，缓存有以下几种状态

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190416154324352.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

可以看到缓存中分为以下四个状态

1. M（Modified）：在缓存中改了数据而导致和内存中的数据不一致了。

2. S（Shared）：表示这个数据是和内存一致的，并且可以共享给其他cpu。

3. E（Exclusive）：表示独享，只被缓存在cpu中。

4. I（Invalid）：表示作废的数据，比如在Share中的数据被修改后，
那就广播为状态I，表示此数据已经作废了。

并且在缓存一致性中有四个操作

1. local read：读取本地缓存的数据

2. local write：把数据写进本地缓存

3. remote read：读取内存中的数据

4. remote write： 把数据写进内存

这些状态之间的读写关系如上图中的箭头方向。

状态之间的转换

M：

1. 当数据被写入内存的时候，为了让其他核共享数据，状态转变为S（remote read）

2. 当写入到内存的时候，如果其他核修改了数据，就变为I（作废）（remote write）

3.要是在缓存中读取（local read）或修改（local write）不变化 

S：

1. 如果localread那就不变

2. 如果修改数据（loacl write）的话，那就转为M（修改），在其他核里变为I（作废）。

3. 如果是存到本地页不会改变状态

4. 如果是在本地修改数据（loacal write），那所有关于此数据的Cache Line不能使用了，状态就变为I（作废）。

E：

1. 从CacheLine中读取数据就不改变

2. 别人需要读取，并写入内存并变为S（remote read）。

3. 如果修改（local write）的话变为M  

4. 在内存中被修改的话（reomte write）就变为I

I：

1.（laocal read） 如果其他核中没有这个数据的话就从内存中取，并且状态变为E，如果在其他Cache中有这个数据的话（也就是这个数据的状态为S或E），那就从内存中读取，并且将这些缓存行的数据变为S。如果这个数据的状态是M，那就把它写到内存再进行读取，这两个缓存行的状态就变为S。

2. 从内存中取数据并且在Cache中修改，状态变为M。如果其他Cache中有这份数据并且状态为M，那就要将数据更新到内存中。如果其他数据有这份数据并且状态不为M，那就将这些转换为I。

3. 既然是作废的状态那就remote操作就不能执行了。


## JMM模型

> 开篇：JVM（Java Virtual Machine）和JMM（Java Memory Model）有什么区别？

可以从含义看到JVM和JMM是一个包含的关系，在JVM中采用的JMM将多个线程分来，可以实现多线程并发的局域化。我们都知道在JVM中处于共享的区域是堆和方法区，因此可以看到数据都是从这些共享区拿来的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190416150420412.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

JMM中每一个线程都有自己的工作内存，它里边的数据都是从主内存（Main Memory）中Copy来的，也就是说它里边存的是主内存的副本，如果对副本进行修改的话，就必须再Copy到主内存，在线程间的通信也是实现的这一原理。如果A线程要和B线程通信，那就是A线程的数据经过Main Memory 然后再到B线程的数据副本的过程。一切的工作都离不开主内存。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019041615102017.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyNjA1OTY4,size_16,color_FFFFFF,t_70)

正如上图所示，在Heap中存放了好多的对象实例，在方法区中存了一些互相分割的线程栈（这些线程栈都是线程私有的），这些都是提供给JMM使用的。Thread可以经过高速缓冲区和主存进行双向的数据交互。这就是在多线程情况下采取的JMM。










