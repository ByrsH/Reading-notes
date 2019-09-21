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


## 03|互斥锁（上）：解决原子性问题

原子性问题的源头是线程切换，那么禁用线程切换就可以解决原子性问题。操作系统做线程切换是依赖CPU中断的，所以禁止CPU发生中断就能禁止线程切换。

在单核CPU时代，禁止CPU中断，可以保证线程不间断地执行，从而保证操作的原子性。但是在多核CPU场景下，当有两个线程同时在执行，禁止CPU中断能保证两个线程不间断执行，但是不能保证同一时刻只有一个线程执行，如果两个线程同时对一个变量进行操作的话，那么就有可能出现bug。

**同一时刻只有一个线程执行**，我们称之为**互斥**。如果能够保证对共享变量的修改是互斥的，那么，无论是单核CPU还是多核CPU，就都能保证原子性了。


### 简易锁模型

![](https://img-blog.csdnimg.cn/20190310152713394.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

把一段需要互斥执行的代码称为临界区。

### 改进后的锁模型

![](https://img-blog.csdnimg.cn/20190310152743462.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

锁和被保护资源间是有对应关系的。注意防止出现类似锁自家门来保护他家资源的问题。


### Java 语言提供的锁技术：synchronized

锁是一种通用的技术方案，Java语言提供的synchronized关键字，就是锁的一种实现。synchronized 关键字既可以用来修饰方法，也可以用来修饰代码块。


    class X {
      // 修饰非静态方法
      synchronized void foo() {
    	// 临界区
      }
      // 修饰静态方法
      synchronized static void bar() {
    	// 临界区
      }
      // 修饰代码块
      Object obj = new Object()；
      void baz() {
    	synchronized(obj) {
      		// 临界区
    	}
      }
    }  

Java 的一条隐式规则：

- 当synchronized 修饰静态方法的时候，锁定的是当前类的class 对象， 在上面的例子中就是 class X；
- 当synchronized 修饰非静态方法的时候，锁定的是当前实例对象this。


### 用 synchronized 解决 count+=1 问题

    
    class SafeCalc {
      long value = 0L;
      synchronized long get() {
    	return value;
      }
      synchronized void addOne() {
    	value += 1;
      }
    }
    
![](https://img-blog.csdnimg.cn/20190310153042244.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

addOne() 方法用synchronized 修饰，既保证了 value += 1 操作时原子性的，又保证了共享变量value 对其它线程的可见性。原因是根据管程中锁的规则：对一个锁的解锁 Happens-Before 于后续对这个锁的加锁。

get() 方法用 synchronized 修饰，保证了value 值的可见性。


### 锁和受保护资源的关系

受保护资源和锁之间的关联关系是 N:1 的关系。一个锁可以锁多个资源，但是多个锁不能锁一个资源。

    class SafeCalc {
      static long value = 0L;
      synchronized long get() {
    	return value;
      }
      synchronized static void addOne() {
    	value += 1;
      }
    }

![](https://img-blog.csdnimg.cn/2019031015311343.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

上面的例子中，get() 方法加的是 this 锁， addOne() 方法加的是 SafeCalc.class 锁，它们都保护 value 资源。但是由于是不同的锁，因此两个临界区就没有互斥关系，就导致了addOne() 方法对value的操作对临界区 get() 没有可见性保证，从而导致并发问题。


### 总结

synchronized 是Java在语言层面提供的互斥原语。锁一定有一个要锁定的对象。


## 04 | 互斥锁（下）：如何用一把锁保护多个资源？

### 保护没有关联关系的多个资源

    
    class Account {
      // 锁：保护账户余额
      private final Object balLock
    	= new Object();
      // 账户余额  
      private Integer balance;
      // 锁：保护账户密码
      private final Object pwLock
    	= new Object();
      // 账户密码
      private String password;
    
      // 取款
      void withdraw(Integer amt) {
    	synchronized(balLock) {
      		if (this.balance > amt){
    			this.balance -= amt;
      		}
    	}
      }
      // 查看余额
      Integer getBalance() {
    	synchronized(balLock) {
      		return balance;
    	}
      }
    
      // 更改密码
      void updatePassword(String pw){
    	synchronized(pwLock) {
      		this.password = pw;
   	 	}
      }
      // 查看密码
      String getPassword() {
    	synchronized(pwLock) {
      		return password;
    	}
      }
    }

上面展示的是用不同的锁对受保护资源进行精细化管理，能够提升性能。这种锁也叫细粒度锁。

如果在每个方法前面加一个synchronized 关键字，就是用 this 这一把锁来管理账户里的所有资源。但是这样会导致性能问题，取款、查看余额、修改密码、查看密码这四个操作都是串行的。

### 保护有关联关系的多个资源

    class Account {
      private int balance;
      // 转账
      synchronized void transfer(
      		Account target, int amt){
    	if (this.balance > amt) {
      		this.balance -= amt;
      		target.balance += amt;
    	}
      }
    }

上面的例子并不能保护临界区内的target.balance 资源，只能保护 this.balance 资源，因为锁是this。另一个线程中target对象的锁是另一个了，它们之间没有互斥。

![](https://img-blog.csdnimg.cn/20190311234134365.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

如果两个线程分别在两个CPU上执行，账户A，B，C都有200元余额，线程1执行账户A转账户B100元，线程2执行账户B转账户C100元，两个线程并不是互斥的，因此就可能出现账户B的余额可能是300，也可能是100。当两个线程同时进入临界区transfer，线程1执行完后B的余额是300（CPU缓存中），线程2执行完后B的余额是100（CPU缓存中），当线程1先写入内存，线程2后写入覆盖，账户B的余额就是100。当线程2线写入内存，线程1后写入覆盖，账户B的余额就是300。

![](https://img-blog.csdnimg.cn/20190311234241313.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 使用一个锁来保护多个资源的正确方式

一个锁来保护多个资源的关键是：锁能覆盖所有受保护资源。

上面的例子修改，所有的对象都的transfer方法执行都要对同一个Account.class 加锁，因此它们之间是互斥的。


    class Account {
      private int balance;
      // 转账
      void transfer(Account target, int amt){
    	synchronized(Account.class) {
      		if (this.balance > amt) {
    			this.balance -= amt;
    			target.balance += amt;
      		}
    	}
      }
    }


![](https://img-blog.csdnimg.cn/2019031123451817.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### 总结

当要保护多个资源时，关键要分析多个资源之间的关系。如果资源之间没有关系，那么就每个资源都有一把锁。如果资源之间有关联关系，就要选择一个粒度更大的锁，这个锁应该能够覆盖所有相关的资源。

原子性的本质是多个资源间有一致性的要求，操作的中间状态对外不可见。解决原子性问题，是要保证中间状态对外不可见。



## 05 | 一不小心就死锁了，怎么办？

上篇文章中以 Account.class 作为互斥锁，来解决银行业务里的转账问题，虽然这个方案不存在并发问题，但是所有的转账操作都是串行的，存在很大的性能问题。

### 向现实世界要答案

在现实世界中，账户转账是支持并发的，并且是可以并行的。那对应于上篇文章的例子如何实现呢？可以通过两把锁来实现，在transfer() 方法内部，先使用 this锁锁住转出账户，再使用target锁锁住转入账户，只有两者都成功时，才执行转账操作。

![](https://img-blog.csdnimg.cn/20190317174932707.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


注意：下面的代码可能会产生死锁

    class Account {
      private int balance;
      // 转账
      void transfer(Account target, int amt){
    	// 锁定转出账户
    	synchronized(this) {  
      		// 锁定转入账户
      		synchronized(target) {   
    			if (this.balance > amt) {
      				this.balance -= amt;
      				target.balance += amt;
    			}
      		}
    	}
      }
    }


### 没有免费的午餐

使用细粒度的锁可以提高并行度，是性能优化的一个重要手段。但它的代价就是可能会导致死锁。

![](https://img-blog.csdnimg.cn/20190317175226788.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


死锁：一组互相竞争资源的线程因互相等待，导致“永久”阻塞的现象。

![](https://img-blog.csdnimg.cn/20190317175404838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 如何预防死锁

当下面这四个条件都放生时才会出现死锁：


1. 互斥，共享资源 X 和 Y 只能被一个线程占用；
2. 占有且等待，线程 T1 已经取得共享资源 X，在等待共享资源 Y 的时候，不释放共享资源 X；
3. 不可抢占，其他线程不能强行抢占线程 T1 占有的资源；
4. 循环等待，线程 T1 等待线程 T2占有的资源，线程T2 等待线程 T1 占有的资源，就是循环等待。


只要破坏其中一个条件，就可以避免死锁的发生。

#### 1.破坏占用且等待条件

要破坏这个条件，可以一次性申请所有资源。


#### 2.破坏不可抢占条件

synchronized如果申请不到资源就会进入阻塞状态，同时线程已经占有的资源也不会释放。但是在 java.util.concurrent包下面提供的Lock是可以轻松解决这个问题。


#### 3.破坏循环等待条件

破坏这个条件，需要对资源进行排序，然后按序申请资源。


## 06 | 用“等待-通知”机制优化循环等待

相比于循环等待，更好的方案是：如果线程要求的条件不满足，则阻塞自己，进入等待状态；当线程要求的条件满足后，通知等待的线程重新执行。其中，使用线程阻塞的方式就能够避免循环等待消耗CPU的问题。


### 完美的就医流程

完整的等待-通知机制：线程首先获取互斥锁，当线程要求的条件不满足时，释放互斥锁，进入等待状态；当要求的条件满足时，通知等待的线程，重新获取互斥锁。


### 用 synchronized 实现等待-通知机制

等待队列和互斥锁是一对一的关系，每个互斥锁都有自己的等待队列。

![](https://img-blog.csdnimg.cn/20190402212646641.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

可以通过Java对象的 notify() 和 notifyAll() 方法通知等待的线程。当条件满足时调用 notify()，会通知等待队列（互斥锁的等待队列）中的线程，告诉它条件曾经满足过。

![](https://img-blog.csdnimg.cn/20190402212723889.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

之所以是曾经满足过，是因为 notify() 只能保证在通知时间点，条件是满足的。而被通知线程的执行时间点和通知时间点基本上不会重合，所以当线程执行时，很可能条件已经不满足了。还有一点需要注意，被通知的线程要想重新执行，仍然需要获取到互斥锁，因为在调用 wait() 时互斥锁已经释放。

wait()、notify()、notifyAll() 三个方法能够被调用的前提是已经获取了相应的互斥锁，所以会发现它们都是在 synchronized{} 内部被调用。如果在外部调用 target.wait() 的话，jvm会抛出一个运行时异常: java.lang.IllegalMonitorStateException。


### 一个更好的资源分配器

    class Allocator {
  	  private List<Object> als;
  	  // 一次性申请所有资源
      synchronized void apply(
        Object from, Object to){
        // 经典写法
        while(als.contains(from) ||
             als.contains(to)){
      	  try{
            wait();
      	  }catch(Exception e){
          }   
        }
      als.add(from);
      als.add(to);  
      }
      // 归还资源
      synchronized void free(
        Object from, Object to){
        als.remove(from);
        als.remove(to);
        notifyAll();
      }
    }


### 尽量使用 notifyAll()

notify() 是会随机地通知等待队列中的一个线程，而 notifyAll() 会通知等待队列中的所有线程。通常情况下尽量使用 notifyAll()， 因为使用notify() 有可能导致某些线程一直通知不到处于等待状态。


### 总结

等待-通知机制是一种非常普遍的线程间协作的方式。Java语言内置的 synchronized 配合 wait()、notify()、notifyAll() 这三个方法可以快速实现这种机制。这种实现背后的理论模型其实是管程。


## 07 | 安全性、活跃性以及性能问题

并发编程中主要有三个方面的问题：安全性问题、活跃性问题和性能问题。

### 安全性问题

线程安全本质上是指正确性，就是程序按照我们期望的执行，不出现意外的结果。

存在安全性问题的情况：存在共享数据并且该数据会发生变化，通俗地讲就是有多个线程会同时读写同一数据。也就是存在数据竞争的情况。

竞态条件：指的是程序的执行结果依赖线程执行的顺序。

也可以这样理解竞态条件：当某个线程发现状态变量满足执行条件后，开始执行操作；但是当这个线程执行操作的时候，其他线程同时修改了状态变量，导致状态变量不满足执行条件了，就会出问题。


    if (状态变量 满足 执行条件) {
      执行操作
    }

在并发环境里，线程的执行顺序是不确定的，如果程序存在竞态条件问题，那就意味着程序的执行结果是不确定的，也就存在bug。

面对数据竞争和竞态条件问题，可以通过锁来保证线程的安全性。


### 活跃性问题

活跃性问题，指的是某个操作无法执行下去。常见的就是死锁，除了死锁外还有“活锁”和“饥饿”。

死锁的表象是线程阻塞，活锁是没有发生阻塞，但是任然会存在执行不下去的情况。解决活锁的方案就是，谦让时，尝试等待一个随机事件就可以了。

所谓“饥饿”指的是线程因无法访问所需资源而无法执行下去的情况。比如在 CPU 繁忙的时候，优先级低的线程得到执行的机会很小，就可能发生线程“饥饿”；持有锁的线程，执行时间过长，也可能导致“饥饿”问题。

解决饥饿问题的方案：

1. 保证资源充足；
2. 公平地分配资源。使用公平锁，先来的线程排在等待队列的前面，优先获得资源；
3. 避免持有锁的线程长时间执行。


### 性能问题

阿姆达尔（Amdahl）定律，代表了处理器并行运算之后效率提升的能力。n 表示 CPU 的核数， p 表示并行百分比，S 就是提高性能倍数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190904215135795.png)


如何减少锁带来的性能影响：

1. 使用无锁的算法和数据结构。例如线程本地存储（Thread Local Storage， TLS）、写入时复制（Copy-on-write）、乐观锁等；Java 并发包里面的原子类是一种无锁的数据结构；Disruptor 则是一个无锁的内存队列。
2. 减少锁持有时间。例如使用细粒度的锁：分段锁、读写锁等


性能方面的度量指标：

1. 吞吐量：指的是单位时间内能处理的请求数量。吞吐量越高，说明性能越好。
2. 延迟：指的是从发出请求到收到响应的时间。延迟越小，说明性能越好。
3. 并发量：指的是能同时处理的请求数量，一般来说随着并发量的增加、延迟也会增加。


### 总结

并发编程是个复杂的技术领域，微观上涉及到原子性问题、可见性问题和有序性问题，宏观则表现为安全性、活跃性已经性能问题。



## 08 | 管程：并发编程的万能钥匙什么是管程

Java 采用的是管程技术，synchronized 关键字及 wait()、notify()、notifyAll() 这三个方法都是管程的组成部分。而管程和信号量是等价的，所谓等价指的是用管程能够实现信号量，也能用信号量实现管程。

管程指的是管理共享变量的操作过程，让他们支持并发。


### MESA 模型

管程模型有：Hasen 模型、Hoare 模型和 MESA 模型。Java 管程实现用的是 MESA 模型。

并发领域的两大核心问题：互斥，即同一时刻只允许一个线程访问共享资源；另一个是同步，即线程之间如何通信、协作。

管程解决互斥问题的思路，就是将共享变量及其对共享变量的操作统一封装起来。同一时刻的操作只允许一个线程进入管程。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190905213923420.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190905213944181.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


管程里还引入了条件变量的感念，而且每一个条件变量都对应一个等待队列。

当一个线程 T1 从入口等待队列中出列，从入口处进入管程中。只有满足某一条件时才能执行操作，这个条件就是条件变量。条件不满足时，就会进入这个条件变量的等待队列等待（通过调用 wait() 方法实现），此时管程是允许其他线程进入的。其他线程 T2 进入了管程，执行了某些操作，导致 T1 线程的条件满足，T2 要通知 T1 条件已经满足（通过调用 notify()/notifyAll() 方法实现）。T1 得到通知后，会从条件变量等待队列中出来，然后进入入口等待队列，等待从新进入管程。


### wait() 的正确姿势

MESA 管程特有的一个编程范式，就是需要在一个 while 循环里面调用 wait()。

    while(条件不满足) {
      wait();
    }

管程要求同一时刻只允许一个线程执行，当线程 T2 的操作使线程 T1 等待的条件满足时，究竟是哪个线程执行呢。不同的模型有不同的执行策略：

1. Hasen 模型里面，要求 notify() 放在代码的最后，这样 T2 通知完 T1 后，T2 就结束了，然后 T1 再执行，这样就能保证同一时刻只有一个线程执行。
2. Hoare 模型里面，T2 通知完 T1 后，T2 阻塞，T1 马上执行；等 T1 执行完，再唤醒 T2 ，也能保证同一时刻只有一个线程执行。但是 T2 多了一次阻塞唤醒操作。
3. MESA 模型里面，T2 通知完 T1 后，T2 还是会继续执行，T1 并不立即执行，仅仅是从条件变量的等待队列进入入口等待队列里面。这样做不好的地方是，当 T1 再次执行时，可能曾经满足的条件，现在已经不满足了。所以需要以循环的方式检验条件变量。


### notify() 何时可以使用

一般情况下尽量使用 notifyAll() 方法，只有满足下面三个条件时，才使用 notify():

1. 所有等待线程有相同的等待条件；
2. 所有等待线程被唤醒后，执行相同的操作；
3. 只需要唤醒一个线程。


### 总结

Java 内置的管程方案通过 synchronized 实现，synchronized 修饰的代码块，在编译期会自动生成相关加锁和解锁的代码，但是仅支持一个条件变量。Java 并发包实现的管程支持多个条件变量，不过并发包里的锁，需要开发人员自己进行加锁和解锁操作。


## 09 | Java线程（上）：Java线程的生命周期在 

Java 领域实现并发线程的主要手段是多线程，Java里面的线程本质上就是操作系统的线程。


### 通用的线程生命周期

线程生命周期内的 5 种状态：初始状态、可运行状态、运行状态、休眠状态和终止状态。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190912185804339.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


1、初始状态，指的是线程已经被创建，但还不允许分配 CPU 执行。这是在编程语言层面被创建，而操作系统层面，真正的线程还没被创建。

2、可运行状态，指的是线程可以分配 CPU 执行。这时真正的操作系统线程已经被成功创建，所以可以分配 CPU 执行。

3、运行状态，指的是有空闲的 CPU 时，操作系统会将其分配给一个可运行状态的线程，这时线程就是运行状态。

4、休眠状态，指的是运行状态的线程调用一个阻塞的 API 或等待某个事件，那么状态就会转为休眠，同时释放 CPU 使用权。当线程被唤醒时进入可运行状态。

5、终止状态，指的是线程执行完或者出现异常，意味着线程生命周期结束。


### Java 中线程的生命周期

Java 语言中线程共有六种状态：

1. NEW（初始化状态）
2. RUNNABLE（可运行/运行状态）
3. BLOCKED（阻塞状态）
4. WAITING（无时限等待）
5. TIMED_WAITING（有时限等待）
6. TERMINATED（终止状态）


其中 BLOCKED、WAITING、TIMED_WAITING 对应操作系统的休眠状态。当 Java 线程处于这三种状态之一，其就永远没有 CPU 的使用权。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190912185904325.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


#### 1、 RUNNABLE 与 BLOCKED 的状态转换

当线程等待 synchronized 的隐式锁时，会触发 RUNNABLE 转换成 BLOCKED 状态，当等待线程获得 synchronized 隐式锁时，就又会从 BLOCKED 状态转换为 RUNNABLE 状态。

线程调用阻塞式 API 时，操作系统层面线程会转换到休眠状态。而在 jvm 层面并不关心操作系统调度相关的状态，其状态还是 RUNNABLE。


#### 2、RUNNABLE 与 WAITING 的状态转换

第一种，获得 synchronized 隐式锁的线程，调用无参数的 Object.wait() 方法。

第二种，调用无参的 Thread.join() 方法。join() 是一种线程同步方法， B 线程中调用 A.join() ，那么 B 线程将会进入 WAITING 状态，当 A 线程执行完后，B 线程又转为 RUNNABLE 状态。

第三种，调用 LockSupport.park() 方法。Java 并发包中的锁，都是基于它实现的。调用 LockSupport.park() 方法，当前线程会进入 WAITING 状态；调用 LockSupport.unpark(Thread thread) 方法，会唤醒目标线程。


#### 3、RUNNABLE 与 TIMED_WAITING 的状态转换

1. 调用 Thread.sleep(long millis) 方法；
2. 获得 synchronized 隐式锁的线程，调用 Object.wait(long timeout) 方法；
3. 调用 Thread.join(long millis) 方法；
4. 调用 LockSupport.parkNanos(Object blocker, long deadline) 方法；
5. 调用 LockSupprot.parkUntil(long deadline) 方法；


#### 4、从 NEW 到 RUNNABLE 状态

Java 新创建的 Thread 对象是处于 NEW 状态，线程对象调用 start() 方法就会转到 RUNNABLE 状态。

创建和启动多线程（thread.start()）

- 扩展Thread类
- 实现Runable接口  
- 实现Callable接口     


#### 5、从 RUNNABLE 到 TERMINATED 状态

线程执行完 run() 方法后，会自动转换到 TERMINATED 状态。如果执行过程中异常抛出，也会导致线程终止。强制线程终止调用 interrupt() 方法。

**stop() 和 interrupt() 方法的区别**

stop() 方法会立即杀死线程，如果线程持有 ReentrantLock 锁，是不会调用 ReentrantLock 的 unlock() 去释放锁的，因此其他线程也就没有机会获得 ReentrantLock 锁，这是很危险的。类似的还有 suspend() 方法和 resume() 方法。

interrupt() 方法仅仅是通知线程中断，被通知的线程有机会执行一些后续的操作，同时也可以无视这个通知。收到通知的方式有两种：一种是异常，另一种是主动检测。

当线程 A 处于 WAITING、TIMED_WAITING 状态时，其他线程调用 A.interrupt() 方法，会是线程 A 转到 RUNNABLE 状态，同时线程 A 的代码会触发 InterruptedException 异常。

当线程 A 处于 RUNNABLE 状态时，并且阻塞在 Java.nio.channels.InterruptibleChannel 上时，如果其他线程调用 A.interrupt() ，A 会触发 ClosedByInterruptException 异常；如果阻塞在 Selector 上，会立即返回。

如果线程处于 RUNNABLE 状态，并且没有阻塞在某个 I/O 操作上，中断就要依赖线程主动检测中断状态了，通过调用 isInterrupted() 方法。



## 10 | Java线程（中）：创建多少线程才是合适的？

使用多线程的目的主要是降低延迟，提高吞吐量。

### 多线程的应用场景

在并发编程领域，提升性能的本质上就是提升硬件的利用率，具体的就是，提升 I/O 利用率和 CPU 的利用率。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190912190148421.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190912190214833.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

如果 CPU 和 I/O 设备的利用率都很低，那么可以尝试通过增加线程来提高吞吐量。

多核执行多线程提高 CPU 利用率。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190912190238932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 创建多少线程合适

程序大致分为 CPU 密集型计算和 I/O 密集型计算。

对于 CPU 密集型的计算场景，理论上“线程的数量=CPU 核数”就是最合适的。在工程上，一般会设置为“CPU 核数 + 1”。这样的话，当线程因为偶尔的内存页失效或者其他原因导致阻塞时，额外的线程可以用上，保证CPU的利用率。如果在增加线程也只是增加线程切换的成本。

对应 I/O 密集型操作，最佳的线程数与程序中 CPU 计算和 I/O 操作的耗时比相关，

单核 CPU 公式：  最佳线程数 = 1 + (I/O 耗时 / CPU 耗时)。

多核 CPU 公式：  最佳线程数 = CPU 核数 * [1 + (I/O 耗时 / CPU 耗时)]。

apm 工具测试 CPU 耗时和 I/O 耗时。


## 11 | Java线程（下）：为什么局部变量是线程安全的？

### 方法是如何被执行的

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915175821130.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


每个方法在调用栈里都有自己的独立空间，称为栈帧。每个栈帧里都有对应方法需要的参数和返回地址。当调用方法时，会创建新的栈帧，并压入调用栈；当方法返回时，对应的栈帧就会被自动弹出。也就是说栈帧和方法是同生共死的。方法的调用是利用栈结构解决的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019091517585410.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 局部变量存哪里？

局部变量放到了调用栈里。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915180044682.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 调用栈与线程

每个线程都有自己独立的调用栈。


### 线程封闭

线程封闭指的是：仅在单线程内访问数据


## 12 | 如何用面向对象思想写好并发程序？

### 一、封装共享变量

封装的通俗解释就是将属性和实现细节封装在对象内部，外界对象只能通过目标对象提供的公共方法来间接访问这些内部属性。

利用面向对象思想写并发程序的思路：将共享变量作为对象属性封装在内部，对所有公共方法制定并发访问策略。

对于那些不会发生变化的共享变量，建议使用 final 关键字来修饰。


### 二、识别共享变量间的约束条件

约束条件决定了并发访问策略。如果约束条件识别不足，很可能导致设计的并发访问策略有问题。


### 三、制定并发访问策略

并发访问策略：

1. 避免共享：避免共享的技术主要是利用线程本地存储以及每个任务分配独立的线程。
2. 不变模式：Java 领域应用的很少。例如 Actor 模式、CSP 模式以及函数式编程的基础都是不变模式。
3. 管程以及其他同步工具：管程是 Java 领域万能的解决方案，对于特定的场景，Java 并发包提供的读写锁、并发容器等同步工具更好。


一些宏观原则：

1. 优先使用成熟的工具类。
2. 迫不得已时才使用低级的同步原语：synchronized、Lock、Semaphore 等
3. 避免过早优化：安全第一，并发程序首先要保证安全，出现性能瓶颈后再优化。


## 13 | 理论基础模块热点问题答疑

锁应是私有的、不可变的、不可重用的。

方法调用是先计算参数，然后才压入栈中。日志输出时使用 {} 占位符是好的习惯，如果使用 + 拼接输出，那么不管日志级别是什么，都会计算 + 参数。

在触发 InterruptedException 异常的同时，jvm 会同时把线程的中断标志位清除，所以这个时候 th.isInterrupted() 返回的是 false。因此需要依据标志位做判断依据的话，要捕获异常，重新设置中断标志位。


## 14 | Lock和Condition（上）：隐藏在并发包中的管程

Java SDK 并发包通过 Lock 和 Condition 两个接口来实现管程，其中 Lock 用于解决互斥问题，Condition 用于解决同步问题。

### 再造管程的理由

破坏不可抢占条件三种方案：

1. 能够响应中断。synchronized 的问题是，在持有锁 A 后，如果尝试获取锁 B 失败，那么线程就进入阻塞状态，一旦死锁，就没有任何机会唤醒阻塞的线程。如果阻塞线程能够响应中断，那么就有机会是否持有的资源。
2. 支持超时。在一段时间内没有获取到锁，不是进入阻塞状态，而是返回一个错误。
3. 非阻塞地获取锁。在尝试获取锁失败，并不是进入阻塞状态，而是直接返回。


Lock 的三个接口就支持上面三种方案：


    // 支持中断的 API
    void lockInterruptibly()
      throws InterruptedException;
    // 支持超时的 API
    boolean tryLock(long time, TimeUnit unit)
      throws InterruptedException;
    // 支持非阻塞获取锁的 API
    boolean tryLock();


### 如何保证可见性

Java 里多线程的可见性是通过 Happens-Before 规则保证的。synchronized 的解锁 Happens-Before 与后续对这个锁的加锁。Java SDK 里面锁利用了 volatile 相关的 Happens-Before 规则。

Happens-Before 规则：

1. 顺序性规则：对于线程 T1，value+=1 Happens-Before 释放锁的操作 unlock()；
2. volatile 变量规则：由于 state = 1 会先读取 state，所以线程 T1 的 unlock() 操作 Happens-Before 线程 T2 的 lock() 操作；
3. 传递性规则：线程 T1 的 value+=1 Happens-Before 线程 T2 的 lock() 操作。

### 什么是可重入锁

可重入锁指的是线程可以重复获取同一把锁。

可重入函数指的是多个线程可以同时调用该函数，每个线程都能得到正确的结果；同时支持线程切换，无论被切换多少次，结果都是正确的。也就是函数是线程安全的。


### 公平锁与非公平锁

ReentrantLock 类有两个构造函数，fair 参数代表的是锁的公平策略，true 表示构造一个公平锁。

公平锁在唤醒一个等待线程时，谁的等待时间长就唤醒谁。非公平锁则不提供这个保证。


### 用锁的最佳实践

1、永远只在更新对象的成员变量时加锁

2、永远只在访问可变的成员变量时加锁

3、永远不在调用其他对象的方法时加锁
























