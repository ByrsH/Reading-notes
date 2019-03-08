## 学习攻略 | 如何才能学好并发编程？


并发编程领域抽象成三个核心问题：分工，同步和互斥。

**分工**：把一个功能分给不同的线程共同实现

**同步**：当某个条件不满足时，线程需要等待，当某个条件满足时，线程需要被唤醒执行。管程是解决并发问题的万能钥匙。

**互斥**：保证线程安全，同一时刻，只允许一个线程访问共享变量。可见性，有序性，原子性。核心技术是锁。

全景图：

![](https://img-blog.csdnimg.cn/20190304224159219.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)



## 01 | 可见性、原子性和有序性问题：并发编程Bug的源头

为了平衡CPU、内存和I/O设备三者的速度差异，计算机体系机构、操作系统、编译程序都做了优化，主要为：

1. CPU增加了缓存，以均衡与内存的速度差异；
2. 操作系统增加了进程、线程，已分时复用CPU，进而均衡CPU与I/O设备的速度差异；
3. 编译程序优化指令执行次序，使得缓存能够得到更加合理地利用；

### 源头之一：缓存导致的可见性问题

一个线程对共享变量的修改，另外一个线程能够立刻看到，我们称为**可见性**。

单核时代，不同线程操作同一个CPU里面的缓存，不存在可见性问题。

![](https://img-blog.csdnimg.cn/20190307103229356.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


多核时代，每颗CPU都有自己的缓存，不同的线程操作不同CPU的缓存时，对彼此之间就不具备可见性了。

![](https://img-blog.csdnimg.cn/20190307103253420.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 源头二： 线程切换带来的原子性问题

![](https://img-blog.csdnimg.cn/20190307103323603.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

任务切换的时机大多数是在时间片结束的时候。我们现在使用的高级程序语言一条语句往往对应多条CPU指令，例如语句：count += 1，至少需要三条CPU指令。

* 指令1：首先，需要把变量 count 从内存加载到CPU的寄存器；
* 指令2：之后，在寄存器中执行 +1 操作；
* 指令3：最后，将结果写入内存（缓存机制导致可能写入的是CPU缓存而不是内存）。

操作系统做任务切换，可以发生在任何一条CPU指令执行完。这就可能得到意想不到的结果。

![](https://img-blog.csdnimg.cn/20190307103402441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


我们把一个或者多个操作在CPU执行的过程中不被中断的特性成为**原子性**。CPU能够保证原子操作是CPU指令级别的，但不是高级语言的操作符，因此很多时候我们需要在高级语言层面保证操作的原子性。


### 源头之三：编译优化带来的有序性问题

**有序性**指的是程序按照代码的先后顺序执行。编译器为了优化性能，有时候会改变程序中语句的先后顺序。

双重检查创建单例对象：

    public class Singleton {
      static Singleton instance;
      static Singleton getInstance(){
        if (instance == null) {
          synchronized(Singleton.class) {
            if (instance == null)
              instance = new Singleton();
          }
        }
        return instance;
      }
    }


instance = new Singleton() 语句经过编译优化重排序后的CPU执行过程可能是：
	
1. 分配一块内存M；
2. 将M的地址赋值给instance实例；
3. 最后再内存M上初始化Singleton对象。

当线程A执行完指令2时，线程切换B线程调用getInstance方法，获得未初始化的Singleton对象，如果此时访问对象成员变量，那么就可能触发空指针异常。

![](https://img-blog.csdnimg.cn/20190307103518718.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 总结

CPU缓存导致了可见性问题，线程切换带来了原子性问题，编译优化带来了有序性问题。引入它们的目的是为了提高程序性能。



## 02 | Java内存模型：看Java如何解决可见性和有序性问题

可见行问题是由缓存导致的，有序性问题是由编译优化导致的，因此只要禁用缓存和编译优化就可以解决可见性和有序性问题。考虑到性能问题就需要根据实际情况按需禁用。

什么是Java内存模型：Java内存模型规范了 JVM 如何提供按需禁用缓存和编译优化的方法。具体来说，这些方法包括 volatile，synchronized 和 final三个关键字，以及六项 Happens-Before 规则。

### 使用 volatile 的困惑

Java在1.5版本时有对volatile语义进行了增强，也就是说1.5版本前程序允许结果和1.5版本及后面版本可能不一样。具体是一项 Happens-Before规则。

### Happens-Before 规则

Happens-Before表示前面一个操作的结果对后续操作是可见的。Happens-Before 规则约束了编译器的优化行为，虽允许编译器优化，但是要求编译器优化后一定遵循 Happens-Before 规则。

Happens-Before 规则中的六项规则是关于可见性的，具体如下：

#### 1.程序的顺序性规则

这条规则是指，在一个线程中，按照程序顺序，前面的操作 Happens-Before 于后续的任意操作。比如程序前面对某个变量的修改一定是对后续操作可见的。


#### 2.volatile变量规则

这条规则是指对一个volatile变量的写操作， Happens-Before 与后续对这个volatile变量的读操作。


#### 3.传递性

这条规则是指，如果 A Happens-Before B， 且 B Happens-Before C， 那么 A Happens-Before C。


#### 4.管程中锁的规则

这条规则是指，对一个锁的解锁 Happens-Before 于后续对这个锁的加锁。

管程是一种通用的同步原语，在Java中指的就是 synchronized, synchronized 是Java里对管程的实现。


    synchronized (this) { // 此处自动加锁
      // x 是共享变量, 初始值 =10
      if (this.x < 12) {
        this.x = 12;
      }  
    } // 此处自动解锁


#### 5.线程 start() 规则

这条规则是关于线程启动，指的是主线程A启动（调用start()方法）子线程B后，子线程B能够看到主线程在启动子线程B前的操作。

    
    Thread B = new Thread(()->{
      // 主线程调用 B.start() 之前
      // 所有对共享变量的修改，此处皆可见
      // 此例中，var==77
    });
    // 此处对共享变量 var 修改
    var = 77;
    // 主线程启动子线程
    B.start();


#### 6.线程join() 规则

这条规则是关于线程等待的，指的是主线程A等待子线程B完成（主线程A通过调用子线程B的join() 方法实现），当子线程B完成后（主线程A中join()方法返回），主线程能够看到子线程的对共享变量的操作。


    Thread B = new Thread(()->{
      // 此处对共享变量 var 修改
      var = 66;
    });
    // 例如此处对共享变量修改，
    // 则这个修改结果对线程 B 可见
    // 主线程启动子线程
    B.start();
    B.join()
    // 子线程所有对共享变量的修改
    // 在主线程调用 B.join() 之后皆可见
    // 此例中，var==66



### 被忽视的final

final 修饰变量时，初衷是告诉编译器：这个变量生而不变，可以可劲儿优化。

在1.5以后 Java 内存模型对final类型变量的重排序进行了约束。只要提供正确构造函数没有“逸出”，就不会出问题。

下面是“逸出” 举例：


    // 以下代码来源于【参考 1】
    final int x;
    // 错误的构造函数
    public FinalFieldExample() {
      x = 3;
      y = 4;
      // 此处就是讲 this 逸出，
      global.obj = this;
    }


### 总结

在Java语言里面， Happens-Before 的语义本质上是一种可见性， A Happens-Before B，意味着A事件对B事件来说是可见的，无论A事件和B事件是否发生在同一个线程里。



























