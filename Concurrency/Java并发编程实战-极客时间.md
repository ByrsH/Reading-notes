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


## 15 | Lock和Condition（下）：Dubbo如何用管程实现异步转同步？

Lock 有别于 synchronized 隐式锁的三个特征：能够响应中断、支持超时和非阻塞地获取锁。

Java 语言内置的管程里只有一个条件变量，而 Lock&Condition 实现了管程模型里面的多个条件变量。

Lock&Condition 实现的管程里的线程等待通知只能使用 await()、signal()、signalAll()，等同与 wait()、notify()、notifyAll()，但是不能调用它们。


### 同步与异步

调用方是否需要等待结果，如果需要就是同步；如果不需要等待结果就是异步。

程序支持异步的两种方式：

1. 调用方创建一个子线程，在子线程中执行方法调用，这种调用我们称为异步调用；
2. 方法实现的时候，创建一个新的线程执行主要逻辑，主线程直接 return，这种方法一般称为异步方法。


### Dubbo 源码分析

通过等待-通知机制来实现。


## 16 | Semaphore：如何快速实现一个限流器？

### 信号量模型

简单概括为：一个计数器，一个等待队列，三个方法。

三个方法语义：

- init() ：设置计数器的初始值
- down() ： 计数器的值减 1；如果此时计数器的值小于 0 ，则当前线程将被阻塞，否则当前线程可以继续执行。
- up() ： 计数器值加 1；如果此时计数器的值小于或等于 0 ，则唤醒等待队列中的一个线程，并将其从等待队列中移除。


### 如何使用信号量

使用 down() 和 up() 方法的特性来保证互斥性。


### 快速实现一个限流器

semaphore 可以允许多个线程访问临界区。

利用 semaphore 的 init() 初始化 N 个池化资源，再用 down() 和 up() 方法的特性来实现当访问资源线程大于 N 时线程阻塞的功能。


## 17 | ReadWriteLock：如何快速实现一个完备的缓存？

读写锁遵循以下三条基本原则：

1. 允许多个线程同时读共享变量；
2. 只允许一个线程写共享变量；
3. 如果一个写线程正在执行写操作，此时禁止读线程读共享变量。


### 快速实现一个缓存
    
    class Cache<K,V> {
      final Map<K, V> m =
    new HashMap<>();
      final ReadWriteLock rwl =
    new ReentrantReadWriteLock();
      final Lock r = rwl.readLock();
      final Lock w = rwl.writeLock();
      V get(K key) {
    V v = null;
    // 读缓存
    r.lock(); ①
    try {
      v = m.get(key); ②
    } finally{
      r.unlock(); ③
    }
    // 缓存中存在，返回
    if(v != null) {   ④
      return v;
    }  
    // 缓存中不存在，查询数据库
    w.lock(); ⑤
    try {
      // 再次验证
      // 其他线程可能已经查询过数据库
      v = m.get(key); ⑥
      if(v == null){  ⑦
    // 查询数据库
    v= 省略代码无数
    m.put(key, v);
      }
    } finally{
      w.unlock();
    }
    return v;
      }
    }



### 读写锁的升级与降级

先获取读锁，然后再获取写锁，叫做锁的升级。但是 ReadWriteLock 并不支持这种升级，当读锁还没释放的时候，获取写锁会导致写锁永久等待。

先获取写锁，然后再获取读锁，叫做锁的降级，是允许的。


### 总结

读写锁类似于 ReentrantLock， 也支持公平模式和非公平模式。只有写锁支持条件变量，读锁不支持。


## 18 | StampedLock：有没有比读写锁更快的锁？

StampedLock 锁比读写锁性能更好。


### StampedLock 支持的三种锁模式

ReadWriteLock 支持两种模式：一种是读锁，一种是写锁。StampedLock 支持三种模式：写锁、悲观读锁和乐观读锁。

ReadWriteLock 支持多个线程同时读，但是当多个线程同时读的时候，所有的写操作会被阻塞；而 StampedLock 提供的乐观读，是允许一个线程获取写锁的，不是所有的写操作都被阻塞。

乐观读这个操作是无锁的，所以相比较 ReadWriteLock 的读锁，乐观读的性能更好一些。

如果执行乐观读操作的期间，存在写操作，应把乐观读升级为悲观读锁。


### 进一步理解乐观读

数据库的乐观锁和 StampedLock 的乐观读很相似。数据库乐观锁通过增加一个 version 的字段版本，每次更新的时候都会 version + 1，version会作为更新的where条件，如果更新成功表示期间没有修改，如果更新失败则表示数据已经被修改。

    update product_doc
    set version=version+1，...
    where id=777 and version=9


### StampedLock 使用注意事项

StampedLock 适用于读多写少的场景，简单的场景可以替代 ReadWriteLock。使用 StampedLock 时还应注意，它不支持重入；StampedLock 的悲观读锁、写锁都不支持条件变量；如果线程阻塞在 StampedLock 的 readLock() 或者 writeLock() 时，此时调用该阻塞线程的 interrupt() 方法，会导致 CPU 飙升。因此，在需要支持中断功能时，一定使用可中断的悲观读锁 readLockInterruptibly() 和写锁 writeLockInterruptibly() 。


### 总结

StampedLock 读模板：


    final StampedLock sl =
      new StampedLock();
    
    // 乐观读
    long stamp =
      sl.tryOptimisticRead();
    // 读入方法局部变量
    ......
    // 校验 stamp
    if (!sl.validate(stamp)){
      // 升级为悲观读锁
      stamp = sl.readLock();
      try {
    // 读入方法局部变量
    .....
      } finally {
    // 释放悲观读锁
    sl.unlockRead(stamp);
      }
    }
    // 使用方法局部变量执行业务操作
    ......


StampedLock 写模板：


    long stamp = sl.writeLock();
    try {
      // 写共享变量
      ......
    } finally {
      sl.unlockWrite(stamp);
    }



## 19 | CountDownLatch和CyclicBarrier：如何让多线程步调一致？

### 利用并行优化系统

对于串行化的系统，优化性能首先想到的是能否利用多线程并行处理。


    while(存在未对账订单){
      // 查询未对账订单
      Thread T1 = new Thread(()->{
    	pos = getPOrders();
      });
      T1.start();
      // 查询派送单
      Thread T2 = new Thread(()->{
    	dos = getDOrders();
      });
      T2.start();
      // 等待 T1、T2 结束
      T1.join();
      T2.join();
      // 执行对账操作
      diff = check(pos, dos);
      // 差异写入差异库
      save(diff);
    }


### 用 CountDownLatch 实现线程等待

上述实现是通过 join() 方法等待两个线程操作完退出的方式，达到保证到调用 check 方法时都已查询完成。当程序优化使用线程池时，join方法就失效了。这时就要通过 CountDownLatch 来实现了。

    
    // 创建 2 个线程的线程池
    Executor executor =
      Executors.newFixedThreadPool(2);
    while(存在未对账订单){
      // 计数器初始化为 2
      CountDownLatch latch =
    	new CountDownLatch(2);
      // 查询未对账订单
      executor.execute(()-> {
    pos = getPOrders();
    latch.countDown();
      });
      // 查询派送单
      executor.execute(()-> {
    	dos = getDOrders();
    	latch.countDown();
      });
      
      // 等待两个查询操作结束
      latch.await();
      
      // 执行对账操作
      diff = check(pos, dos);
      // 差异写入差异库
      save(diff);
    }
    

### 进一步优化性能

上述模型可以看作生产者-消费者模型。两个查询是生产者，查询数据分别存入两个队列，在都完成一次查询后，通知另一个线程消费，执行对账操作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191027153255678.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 用 CyclicBarrier 实现线程同步

CyclicBarrier 的计数器有自动重置的功能，当减到 0 的时候，会自动重置你设置的初始值。



    // 订单队列
    Vector<P> pos;
    // 派送单队列
    Vector<D> dos;
    // 执行回调的线程池
    Executor executor =
      Executors.newFixedThreadPool(1);
    final CyclicBarrier barrier =
      new CyclicBarrier(2, ()->{
    	executor.execute(()->check());
      });
      
    void check(){
      P p = pos.remove(0);
      D d = dos.remove(0);
      // 执行对账操作
      diff = check(p, d);
      // 差异写入差异库
      save(diff);
    }
      
    void checkAll(){
      // 循环查询订单库
      Thread T1 = new Thread(()->{
    	while(存在未对账订单){
      // 查询订单库
      pos.add(getPOrders());
      // 等待
      barrier.await();
    }
      });
      T1.start();  
      // 循环查询运单库
      Thread T2 = new Thread(()->{
    	while(存在未对账订单){
      // 查询运单库
      dos.add(getDOrders());
      // 等待
      barrier.await();
    }
      });
      T2.start();
    }
    

### 总结

CountDownLatch 主要用来解决一个线程等待多个线程的场景，CyclicBarrier 是一组线程之间互相等待。CountDownLatch 的计数器是不能循环利用的，但是 CyclicBarrier 的计数器是可以循环利用的，而且具有自动重置的功能，一旦计数器减到 0 会自动重置到初始设置的值。另外， CyclicBarrier 还可以设置回调函数。


## 20 | 并发容器：都有哪些“坑”需要我们填？

### 同步容器及其注意事项

Java 中的容器主要分为四大类：List、Map、Set 和 Queue。这里面并不是所有容器都是线程安全的，我们可以通过把非线程安全的容器封装在对象内部，然后控制访问路径就能达到线程安全的目的。

Java SDK 提供的包装类：

    List list = Collections.
      synchronizedList(new ArrayList());
    Set set = Collections.
      synchronizedSet(new HashSet());
    Map map = Collections.
      synchronizedMap(new HashMap());

基于 synchronized 关键字实现的容器称之为同步容器。

用迭代器变量包装后的同步容器，要注意对容器加锁保证互斥。


### 并发容器及其注意事项

Java 在 1.5 之前的线程安全的容器，主要指的是同步容器，所有方法使用 synchronized 来保证互斥，性能差。1.5 之后提供了性能更改的容器，称之为并发容器。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191027153454737.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 一、List

CopyOnWriteArrayList 就是写的时候将共享变量新复制一份出来，这样就可以读操作完全无锁。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191027153533793.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

CopyOnWriteArrayList 仅适用于写操作非常少的场景，而且能容忍读写的短暂不一致，新写的元素不能立刻被遍历到。CopyOnWriteArrayList 迭代器是只读的，不支持增删改。


### 二、Map

ConcurrentHashMap 的 key是无序的，而 ConcurrentSkipListMap 的 key 是有序的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191027153611260.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

ConcurrentSkipListMap 里面的 SkipList 数据结构称之为“跳表”，跳表插入、删除、查询操作平均的时间复杂度是 O(log n)。理论上和并发线程数无关，对性能要求更高的场景下，可以尝试使用 ConcurrentSkipListMap。


### 三、Set

内容同上。


### 四、Queue

可以从两个维度分类 Queue 并发容器。一个维度是阻塞与非阻塞，阻塞指的是当队列已满时，入队操作阻塞；当队列为空时，出队操作阻塞。另一个维度是单端与双端，单端指的是只能队尾入队，队首出队；双端指的是队首队尾皆可入队出队。阻塞队列都用 Blocking 关键字标识，单端队列使用 Queue 标识，双端队列使用 Deque 标识。

#### 1、单端阻塞队列：

ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue、LinkedTransferQueue、PriorityBlockingQueue 和 DelayQueue

SynchronousQueue 的生产者线程的入队操作必须等待消费者线程的出队操作。LinkedTransferQueue 性能要比 LinkedBlockingQueue 好；PriorityBlockingQueue 支持按优先级出队；DelayQueue 支持延时出队。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191027153736328.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


#### 2、双端阻塞队列：LinkedBlockingDeque

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191027153813807.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


#### 3、单端非阻塞队列：ConcurrentLinkedQueue

#### 4、双端非阻塞队列：ConcurrentLinkedDuque


在使用队列时也要注意容器界限问题，如果容量没有界限可能会到时 OOM。



## 21 | 原子类：无锁工具类的典范

### 无锁方案的实现原理

CPU 为了解决并发问题，提供了 CAS 指令（Compare And Swap）。作为一条 CPU 指令，CAS 指令本身是能够保证原子性的。

使用 CAS 解决并发问题，一般都会伴有自旋，也就是循环尝试。使用 CAS 方案时注意 ABA 问题，Java中提供了AtomicStampedReference和AtomicMarkableReference来解决ABA问题，内部有维护一个版本号，每次修改都同时修改版本号。


### Java 如何实现原子化的 count += 1

    do {
      // 获取当前值
      oldV = xxxx；
      // 根据当前值计算新值
      newV = ...oldV...
    }while(!compareAndSet(oldV,newV);


### 原子类概览

Java SDK 并发包里的原子类分为五个类别：原子化的基本数据类型、原子化的对象引用类型、原子化数组、原子化对象属性更新器和原子化的累加器。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191027153929715.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


#### 1、原子化的基本数据类型

AtomicBoolean、AtomicInteger 和 AtomicLong


    getAndIncrement() //原子化i++
    getAndDecrement() //原子化的i--
    incrementAndGet() //原子化的++i
    decrementAndGet() //原子化的--i
    //当前值+=delta，返回+=前的值
    getAndAdd(delta)
    //当前值+=delta，返回+=后的值
    addAndGet(delta)
    //CAS操作，返回是否成功
    compareAndSet(expect, update)
    //以下四个方法
    //新值可以通过传入func函数来计算
    getAndUpdate(func)
    updateAndGet(func)
    getAndAccumulate(x,func)
    accumulateAndGet(x,func)


#### 2、原子化的对象引用类型

AtomicReference、AtomicStampedReference 和 AtomicMarkableReference，利用它们可以实现对象引用的原子化更新。注意AtomicReference 有ABA 问题，其他两个可以解决 ABA 问题。其原理是增加版本号。


#### 3、原子化数组

AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray，利用这些原子类，可以原子化地更新数组里面的每一个元素。


#### 4、原子化对象属性更新器

AtomicIntegerFieldUpdater、AtomicLongFieldUpdater 和 AtomicReferenceFieldUpdater，利用它们可以原子化地更新对象的属性。都是通过反射机制实现的。

对象属性必须是 volatile 类型的，只有这样才能保证可见性。


#### 5、原子化的累加器

DoubleAccumulator、DoubleAdder、LongAccumulator 和 LongAdder，仅仅用来执行累加操作，速度更快，但不支持 compareAndSet() 方法。


### 总结

无锁方案相比较互斥锁方案，首先性能好，其次是基本不会出现死锁问题（但可能出现饥饿和活锁问题，因为自旋会反复重试）。



## 22 | Executor与线程池：如何创建正确的线程池？

线程是一个重量级的对象，应该避免频繁创建和销毁。


### 线程池是生产者-消费者模式

线程池的设计是采用生产者 - 消费者模式，线程池的使用方是生产者，线程池本身是消费者。


### 如何使用 Java 中的线程池


    ThreadPoolExecutor(
      int corePoolSize,
      int maximumPoolSize,
      long keepAliveTime,
      TimeUnit unit,
      BlockingQueue<Runnable> workQueue,
      ThreadFactory threadFactory,
      RejectedExecutionHandler handler)

各参数的含义：

- corePoolSize：表示线程池保有的最小线程数。
- maximumPoolSize：表示线程池创建的最大线程数。
- keepAliveTime & unit：如果一个线程空闲了 keepAliveTime 和 unit 长时间，而且线程池的线程数大于 corePoolSize，那么这个空闲的线程就要被回收了。
- workQueue：工作队列
- threadFactory：通过这个参数可以自定义如何创建线程，例如指定一个有意义的名字。
- handler：可以自定义任务的拒绝策略。当所有线程都在忙，并且工作队列也已满，那么再提交任务，线程池就会拒绝接收。ThreadPoolExecutor 提供了以下 4 种策略：
	- CallerRunsPolicy：提交任务的线程自己去执行该任务
	- AbortPolicy：默认的拒绝策略，会 throws RejectedExecutionException
	- DiscardPolicy：直接丢弃任务，没有任何异常抛出
	- DiscardOldestPolicy：丢弃最老的任务，把最早进入工作队列的任务丢弃，然后把新任务加入到工作队列


### 使用线程池要注意些什么

线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

说明：Executors 返回的线程池对象的弊端如下：

1）FixedThreadPool 和 SingleThreadPool:
允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM。

2）CachedThreadPool 和 ScheduledThreadPool:
允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM。


如果使用默认的拒绝策略，要注意 catch 拒绝任务时抛出的异常。另外调用 execute() 方法提交任务时，如果运行中出现异常，会导致执行任务的线程终止。因此需要捕获所有的异常并按需处理。

    try {
      //业务逻辑
    } catch (RuntimeException x) {
      //按需处理
    } catch (Throwable x) {
      //按需处理
    }


## 23 | Future：如何用多线程实现最优的“烧水泡茶”程序？

### 如何获取任务执行结果


    // 提交Runnable任务
    Future<?> submit(Runnable task);
    
    // 提交Callable任务
    <T> Future<T> submit(Callable<T> task);
    
    // 提交Runnable任务及结果引用  
    <T> Future<T> submit(Runnable task, T result);

Future 接口的5个方法，需要注意的是两个 get() 方法是阻塞式的。


    // 取消任务
    boolean cancel(
      boolean mayInterruptIfRunning);
    // 判断任务是否已取消  
    boolean isCancelled();
    // 判断任务是否已结束
    boolean isDone();
    // 获得任务执行结果
    get();
    // 获得任务执行结果，支持超时
    get(long timeout, TimeUnit unit);


3个submit 方法之间的区别：

1. 提交 Runnable 任务 submit(Runnable task)：Runnable 接口的 run 方法没有返回值，所以返回的 Future 仅可以用来断言任务已经结束。
2. 提交 Callable 任务 submit(Callable<T> task)：返回的 Future 可以通过 get() 方法获取返回值。
3. 提交 Runnable 任务及结果引用 submit(Runnable task, T result) ： result 用于主子线程共享数据，Runnable 的实现类包含 result 参数构造器，这样就可以在线程中操作 result 了。Future 通过get() 方法获取到的和传入的是同一个对象。


### FutureTask 使用：

    // 创建FutureTask
    FutureTask<Integer> futureTask
      = new FutureTask<>(()-> 1+2);
    // 创建线程池
    ExecutorService es =
      Executors.newCachedThreadPool();
    // 提交FutureTask
    es.submit(futureTask);
    // 获取计算结果
    Integer result = futureTask.get();
    

### 实现最优的“烧水泡茶”程序

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191031103830729.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


    // 创建任务T2的FutureTask
    FutureTask<String> ft2
      = new FutureTask<>(new T2Task());
    // 创建任务T1的FutureTask
    FutureTask<String> ft1
      = new FutureTask<>(new T1Task(ft2));
    // 线程T1执行任务ft1
    Thread T1 = new Thread(ft1);
    T1.start();
    // 线程T2执行任务ft2
    Thread T2 = new Thread(ft2);
    T2.start();
    // 等待线程T1执行结果
    System.out.println(ft1.get());
    
    // T1Task需要执行的任务：
    // 洗水壶、烧开水、泡茶
    class T1Task implements Callable<String>{
      FutureTask<String> ft2;
      // T1任务需要T2任务的FutureTask
      T1Task(FutureTask<String> ft2){
    	  this.ft2 = ft2;
      }
      @Override
      String call() throws Exception {
    	  System.out.println("T1:洗水壶...");
    	  TimeUnit.SECONDS.sleep(1);
    
    	  System.out.println("T1:烧开水...");
    	  TimeUnit.SECONDS.sleep(15);
    	  // 获取T2线程的茶叶  
    	  String tf = ft2.get();
    	  System.out.println("T1:拿到茶叶:"+tf);
    
    	  System.out.println("T1:泡茶...");
    	  return "上茶:" + tf;
      }
    }
    // T2Task需要执行的任务:
    // 洗茶壶、洗茶杯、拿茶叶
    class T2Task implements Callable<String> {
      @Override
      String call() throws Exception {
    	  System.out.println("T2:洗茶壶...");
    	  TimeUnit.SECONDS.sleep(1);
    
    	  System.out.println("T2:洗茶杯...");
    	  TimeUnit.SECONDS.sleep(2);
    
    	  System.out.println("T2:拿茶叶...");
    	  TimeUnit.SECONDS.sleep(1);
    	  return "龙井";
      }
    }
    // 一次执行结果：
    T1:洗水壶...
    T2:洗茶壶...
    T1:烧开水...
    T2:洗茶杯...
    T2:拿茶叶...
    T1:拿到茶叶:龙井
    T1:泡茶...
    上茶:龙井



## 24 | CompletableFuture：异步编程没那么难

异步化是利用多线程优化性能这个核心方案得以实施的基础。


### CompletableFuture 的核心优势

- 无需手工维护线程
- 语义更清晰
- 代码更简练并且专注于业务逻辑


### 创建 CompletableFuture 对象

    //使用默认线程池
    static CompletableFuture<Void>
      runAsync(Runnable runnable)
    static <U> CompletableFuture<U>
      supplyAsync(Supplier<U> supplier)
    //可以指定线程池  
    static CompletableFuture<Void>
      runAsync(Runnable runnable, Executor executor)
    static <U> CompletableFuture<U>
      supplyAsync(Supplier<U> supplier, Executor executor)  

Runnable 接口 run() 方法没有返回值，Supplier 接口的 get() 方法是有返回值的。后面两个参数可以指定线程池参数。

CompletableFuture 默认情况下使用公共的 ForkJoinPool 线程池，这个线程池默认的线程数是 CPU 的核数，也可以通过 JVM option:-Djava.util.concurrent.ForkJoinPool.common.parallelism 来设置线程数。强烈建议要根据不同的业务类型创建不同的线程池，以避免互相干扰。

创建完 CompletableFuture 对象后，会自动地异步执行 runnable.run() 方法或者 supplier.get() 方法。


### 如何理解 CompletionStage 接口

任务的时序关系：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191102235610791.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

AND 聚合关系指的是所有依赖的任务都完成后才开始执行当前任务。OR 聚合关系指的是依赖的任务只要有一个完成就可以执行当前任务。


#### 1、描述串行关系

CompletionStage 接口里面描述串行关系，主要是 thenApply、thenAccept、thenRun 和 thenCompose 四个系列的接口。

    CompletionStage<R> thenApply(fn);
    CompletionStage<R> thenApplyAsync(fn);
    CompletionStage<Void> thenAccept(consumer);
    CompletionStage<Void> thenAcceptAsync(consumer);
    CompletionStage<Void> thenRun(action);
    CompletionStage<Void> thenRunAsync(action);
    CompletionStage<R> thenCompose(fn);
    CompletionStage<R> thenComposeAsync(fn);

用例：

    CompletableFuture<String> f0 =
      CompletableFuture.supplyAsync(
        () -> "Hello World")  //①
      .thenApply(s -> s + " QQ")  //②
      .thenApply(String::toUpperCase);//③
    
    System.out.println(f0.join());
    //输出结果
    HELLO WORLD QQ


### 2、描述 AND 汇聚关系

    CompletionStage<R> thenCombine(other, fn);
    CompletionStage<R> thenCombineAsync(other, fn);
    CompletionStage<Void> thenAcceptBoth(other, consumer);
    CompletionStage<Void> thenAcceptBothAsync(other, consumer);
    CompletionStage<Void> runAfterBoth(other, action);
    CompletionStage<Void> runAfterBothAsync(other, action);


### 3、描述 OR 汇聚关系

    CompletionStage applyToEither(other, fn);
    CompletionStage applyToEitherAsync(other, fn);
    CompletionStage acceptEither(other, consumer);
    CompletionStage acceptEitherAsync(other, consumer);
    CompletionStage runAfterEither(other, action);
    CompletionStage runAfterEitherAsync(other, action);


### 4、异常处理

    CompletionStage exceptionally(fn);
    CompletionStage<R> whenComplete(consumer);
    CompletionStage<R> whenCompleteAsync(consumer);
    CompletionStage<R> handle(fn);
    CompletionStage<R> handleAsync(fn);

exceptionally() 方法处理异常，非常类似于 try{} catch{}。whenComplete 和 handle 系列方法就非常类似于 try{} finally{} ，它们之间的区别在于 whenComplete() 不支持返回结果，handle 支持返回结果。


    CompletableFuture<Integer>
      f0 = CompletableFuture
          .supplyAsync(()->7/0))
    	  .thenApply(r->r*10)
    	  .exceptionally(e->0);
    System.out.println(f0.join());



## 25 | CompletionService：如何批量执行异步任务？

### 利用 Completion 实现询价系统

CompletionService 的实现原理是内部维护了一个阻塞队列，把任务执行结果的 Future 对象加入到阻塞队列中。


### 如何创建 CompletionService


    ExecutorCompletionService(Executor executor)
    
    ExecutorCompletionService(Executor executor, BlockingQueue<Future<V>> completionQueue)

如果不指定 completionQueue， 那么默认会使用无界的 LinkedBlockingQueue。


    // 创建线程池
    ExecutorService executor =
      Executors.newFixedThreadPool(3);
    // 创建CompletionService
    CompletionService<Integer> cs = new
      ExecutorCompletionService<>(executor);
    // 异步向电商S1询价
    cs.submit(()->getPriceByS1());
    // 异步向电商S2询价
    cs.submit(()->getPriceByS2());
    // 异步向电商S3询价
    cs.submit(()->getPriceByS3());
    // 将询价结果异步保存到数据库
    for (int i=0; i<3; i++) {
      Integer r = cs.take().get();
      executor.execute(()->save(r));
    }


### CompletionService 接口说明


    Future<V> submit(Callable<V> task);
    Future<V> submit(Runnable task, V result);
    Future<V> take()
      throws InterruptedException;
    Future<V> poll();
    Future<V> poll(long timeout, TimeUnit unit)
      throws InterruptedException;

如果阻塞队列是空的，那么调用 take() 方法的线程会被阻塞，而 poll() 方法会返回 null 值。


### 利用 CompletionService 实现 Dubbo 中的 ForKing Cluster


    // 创建线程池
    ExecutorService executor =
      Executors.newFixedThreadPool(3);
    // 创建CompletionService
    CompletionService<Integer> cs =
      new ExecutorCompletionService<>(executor);
    // 用于保存Future对象
    List<Future<Integer>> futures =
      new ArrayList<>(3);
    //提交异步任务，并保存future到futures
    futures.add(
      cs.submit(()->geocoderByS1()));
    futures.add(
      cs.submit(()->geocoderByS2()));
    futures.add(
      cs.submit(()->geocoderByS3()));
    // 获取最快返回的任务执行结果
    Integer r = 0;
    try {
      // 只要有一个成功返回，则break
      for (int i = 0; i < 3; ++i) {
    	r = cs.take().get();
    	//简单地通过判空来检查是否成功返回
    	if (r != null) {
     	 break;
    	}
      }
    } finally {
      //取消所有任务
      for(Future<Integer> f : futures)
    f.cancel(true);
    }
    // 返回结果
    return r;


### 总结

当需要批量提交异步任务的时候建议你使用 CompletionService。CompletionService 将线程池 Executor 和阻塞队列 BlockingQueue 的功能融合在一起，能够让批量异步任务的管理更简单。


## 26 | Fork/Join：单机版的MapReduce

对于简单的并行任务，可以通过“线程池 + Future”的方案来解决；如果任务之间有聚合关系，无论是 AND 聚合还是 OR 聚合，都可以通过 CompletableFuture 来解决；而批量的并行任务，则可以通过 CompletionService 来解决。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191106094825978.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

分治，即分而治之，是一种解决复杂问题的思维方法和模式；具体指的是把一个复杂的问题分解成多个相似的子问题，然后再把子问题分解成更小的子问题，直到子问题简单到可以直接求解。


### 分治任务模型

分治模型可分为两个阶段：一个阶段是任务分解，也就是将任务迭代地分解为子任务，直至子任务可以直接计算出结果；另一个阶段是结果合并，即逐层合并子任务的执行结果，直至获得最终结果。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191106094850725.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

任务和子任务具有相似性体现在其算法是相同的，但数据规模是不同的。这种相似性问题，往往都采用递归算法。


### Fork/Join 的使用

Fork/Join 是一个并行计算的框架，主要就是用来支持分治任务模型的，这个计算框架里的 Fork 对应的是分治任务模型里的任务分解，Join 对应的是结果合并。Fork/Join 计算框架主要包含两部分，一部分是分治任务的线程池 ForkJoinPool，另一部分是分治任务 ForkJoinTask。

ForkJoinTask 是一个抽象类，其中 fork() 方法会异步执行一个子任务，而 join() 方法则会阻塞当前线程来等待子任务的执行结果。

使用 Fork/Join 来实现斐波那契数列：


    static void main(String[] args){
      //创建分治任务线程池  
      ForkJoinPool fjp =
    	new ForkJoinPool(4);
      //创建分治任务
      Fibonacci fib =
    	new Fibonacci(30);   
      //启动分治任务  
      Integer result =
    	fjp.invoke(fib);
      //输出结果  
      System.out.println(result);
    }
    //递归任务
    static class Fibonacci extends
    RecursiveTask<Integer>{
      final int n;
      Fibonacci(int n){this.n = n;}
      protected Integer compute(){
    	if (n <= 1)
      		return n;
    	Fibonacci f1 = new Fibonacci(n - 1);
    	//创建子任务  
    	f1.fork();
    	Fibonacci f2 = new Fibonacci(n - 2);
    	//等待子任务结果，并合并结果  
    	return f2.compute() + f1.join();
      }
    }


### ForkJoinPool 工作原理

ForkJoinPool 本质上也是一个生产者 - 消费者的实现，但是其更加智能。ThreadPoolExecutor 内部只有一个任务队列，而 ForkJoinPool 内部有多个任务队列，当通过 invoke() 或者 submit() 方法提交任务是，ForkJoinPool 会根据一定的路由规则把任务提交到一个任务队列中，如果任务在执行过程中会创建子任务，那么子任务会提交到工作线程对应的任务队列。

如果工作线程任务队列空了，那么它会执行其他任务队列中的任务。任务队列采用的是双端队列，正常线程获取任务和“窃取任务”分别是从不同的端获取的，避免不必要的数据竞争。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191106095101625.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 模拟 MapReduce 统计单次数量


    static void main(String[] args){
      String[] fc = {"hello world",
      "hello me",
      "hello fork",
      "hello join",
      "fork join in world"};
      //创建ForkJoin线程池
      ForkJoinPool fjp =
      new ForkJoinPool(3);
      //创建任务
      MR mr = new MR(
      fc, 0, fc.length);  
      //启动任务
      Map<String, Long> result =
      fjp.invoke(mr);
      //输出结果
      result.forEach((k, v)->
    System.out.println(k+":"+v));
    }
    //MR模拟类
    static class MR extends
      RecursiveTask<Map<String, Long>> {
      private String[] fc;
      private int start, end;
      //构造函数
      MR(String[] fc, int fr, int to){
    this.fc = fc;
    this.start = fr;
    this.end = to;
      }
      @Override protected
      Map<String, Long> compute(){
    if (end - start == 1) {
      return calc(fc[start]);
    } else {
      int mid = (start+end)/2;
      MR mr1 = new MR(
      fc, start, mid);
      mr1.fork();
      MR mr2 = new MR(
      fc, mid, end);
      //计算子任务，并返回合并的结果
      return merge(mr2.compute(),
      mr1.join());
    }
      }
      //合并结果
      private Map<String, Long> merge(
      Map<String, Long> r1,
      Map<String, Long> r2) {
    Map<String, Long> result =
    new HashMap<>();
    result.putAll(r1);
    //合并结果
    r2.forEach((k, v) -> {
      Long c = result.get(k);
      if (c != null)
    result.put(k, c+v);
      else
    result.put(k, v);
    });
    return result;
      }
      //统计单词数量
      private Map<String, Long>
      calc(String line) {
    Map<String, Long> result =
    new HashMap<>();
    //分割单词
    String [] words =
    line.split("\\s+");
    //统计单词数量
    for (String w : words) {
      Long v = result.get(w);
      if (v != null)
    result.put(w, v+1);
      else
    result.put(w, 1L);
    }
    return result;
      }
    }


### 总结

Fork/Join 并行计算框架主要解决的是分治任务，核心组件是 ForkJoinPool。ForkJoinPool 支持任务窃取机制，能够让所有线程的工作量基本平衡，不会出现有的线程很忙，有的线程很闲的状况，所以性能很好。Java 1.8 提供的 Stream API 里面并行流是以 ForkJoinPool 为基础的。默认情况下所有的并行流计算都共享一个 ForkJoinPool，如果是 I/O 密集型的并行流计算，那会因为一个很慢的 I/O 计算而拖慢整个系统的性能。所有建议用不同的 ForkJoinPool 执行不同类型的计算任务。



## 27 | 并发工具类模块热点问题答疑

Semaphore 允许多个线程访问一个临界区，如果需要访问共享变量就会存在并发问题，所以必须加锁，也就是说 Semaphore 需要锁中锁。

申请锁和释放锁需要成对出现，最佳实践 try{} finally{}。

CyclicBarrier 是同步调用回调函数之后才唤醒等待的线程。所以，当遇到回调函数的时候，一定要确认下执行回调函数的线程是哪一个。执行回调函数的线程是将 CyclicBarrier 内部计数器减到 0 的那个线程。


## 28 | Immutability模式：如何利用不变性解决并发问题？

不变性模式：简单来讲，就是对象一旦被创建之后，状态就不再发生变化。当变量只有读操作，没有写操作，就保持了不变性，也就没有了并发问题。


### 快速实现具备不可变性的类

将一个类所有的属性都用 final 关键字修饰，并且只允许存在只读方法，那么这个类就具备不可变性了。更严格的做法是类本身也是 final 的。

JDK 中很多类具备不可变性，例如 String 、Long、Integer、Double 等，它们都具备：类和属性都是 final 的，所有方法均是只读的。


### 利用享元模式避免创建重复对象

利用享元模式可以减少创建对象的数量，从而减少内存占用。Java 语言里的基本数据类型的包装类型 Long、Integer、Short、Byte 等都用到了享元模式。

享元模式创建对象的逻辑：创建之前，首先查看对象池中是否存在；如果已经存在，就利用对象池里的对象；如果不存在，就会创建一个新的对象，并且把新对象放入对象池中。

由于基础类型的包装类使用了享元模式，内部缓存了部分数据，因此不适合做锁。很有可能共用的缓存对象当做了共用的锁去使用。


### 使用 Immutability 模式的注意事项

1、对象的所有属性都是 final 的，并不能保证不可变性（成员变量为普通对象的属性是可变的）；

2、不可变对象也需要正确发布。

在使用 Immutability 模式的时候一定要确认保持不变性的边界在哪里，是否要求属性对象也具备不可变性。

不可变对象虽然是线程安全的，但是并不意味着引用这些不可变对象的对象就是线程安全的。如果仅仅要保证变量的可见性，那么可以将变量声明为 volatile 变量。如果要保证原子性，那么可以通过原子类来实现。


    public class SafeWM {
      class WMRange{
    	final int upper;
    	final int lower;
    	WMRange(int upper,int lower){
    		//省略构造函数实现
    	}
      }
      final AtomicReference<WMRange>
    	rf = new AtomicReference<>(
      		new WMRange(0,0)
    	);
      // 设置库存上限
      void setUpper(int v){
    	while(true){
      		WMRange or = rf.get();
      		// 检查参数合法性
     		if(v < or.lower){
    			throw new IllegalArgumentException();
      		}
      		WMRange nr = new
      		WMRange(v, or.lower);
      		if(rf.compareAndSet(or, nr)){
    			return;
      		}
    	}
      }
    }
    

## 29 | Copy-on-Write模式：不是延时策略的COW

Copy-on-Write 顾名思义写时复制。经常缩写为 COW 或者 CoW。


### Copy-on-Write 模式的应用领域

CopyOnWriteArrayList 和 CopyOnWriteArraySet 这两个容器实现的读操作是无锁的。

Copy-on-Write 最大的应用领域是在函数式编程领域。函数式编程的基础是不可变性。


### 一个真实案例

Rpc 负载均衡服务提供方路由信息，读多写少的场景。

Copy-on-Write 的缺点就是消耗内存，每次修改都需要复制一个新的对象出来。


## 30 | 线程本地存储模式：没有共享，就没有伤害

Java 提供了线程本地存储（ThreadLocal）能够做到避免共享。


### ThreadLocal 的使用方法

    static class SafeDateFormat {
      //定义ThreadLocal变量
      static final ThreadLocal<DateFormat>
      tl=ThreadLocal.withInitial(
    	()-> new SimpleDateFormat(
      	"yyyy-MM-dd HH:mm:ss"));
      
      static DateFormat get(){
    	return tl.get();
      }
    }
    //不同线程执行下面代码
    //返回的df是不同的
    DateFormat df =
      SafeDateFormat.get()；


### ThreadLocal 的工作原理


![在这里插入图片描述](https://img-blog.csdnimg.cn/20191113102312146.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


    class Thread {
      //内部持有ThreadLocalMap
      ThreadLocal.ThreadLocalMap
    	threadLocals;
    }
    class ThreadLocal<T>{
      public T get() {
    	//首先获取线程持有的
    	//ThreadLocalMap
    	ThreadLocalMap map =
      		Thread.currentThread().threadLocals;
    	//在ThreadLocalMap中
    	//查找变量
    	Entry e = map.getEntry(this);
    	return e.value;  
      }
      static class ThreadLocalMap{
    	//内部是数组而不是Map
    	Entry[] table;
    	//根据ThreadLocal查找Entry
    	Entry getEntry(ThreadLocal key){
      	//省略查找逻辑
    	}
    	//Entry定义
      static class Entry extends
    	WeakReference<ThreadLocal>{
      		Object value;
    	}
      }
    }
    

Java 的实现中，ThreadLocalMap 是属于 Thread，ThreadLocal 仅仅是一个工具类，内部并不持有任何与线程相关的数据，所有和线程相关的数据都存储在 Thread 里面。不容易产生内存泄漏。Thread 被回收时，ThreadLocalMap 就能被回收。


### ThreadLocal 与内存泄漏

线程池中的 ThreadLocal 有可能导致内存泄漏，因为线程池中的线程存活时间太长，往往都是和程序同生共死，这就意味着 ThreadLocalMap 一直都不会被回收。Entry 对 ThreadLocal 是弱引用，Entry对 value 是强引用。

可以通过 try{} finally{} 方案，手动释放资源。


    ExecutorService es;
    ThreadLocal tl;
    es.execute(()->{
      //ThreadLocal增加变量
      tl.set(obj);
      try {
   	 	// 省略业务逻辑代码
      }finally {
    	//手动清理ThreadLocal
    	tl.remove();
      }
    });
    

### InheritableThreadLocal 与继承性

通过 ThreadLocal 创建的线程变量，其子线程是无法继承的，也就是无法访问父线程创建的变量。

InheritableThreadLocal 是 ThreadLocal 的子类，是可以继承父线程创建的线程变量。同样不建议在线程池中使用 InheritableThreadLocal，不仅会导致内存泄漏，还有可能因为线程的动态创建导致继承关系错乱。


### 总结

线程本地存储模式本质上是一种避免共享的方案。



## 31 | Guarded Suspension模式：等待唤醒机制的规范实现

### Guarded Suspension 模式

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191124175740816.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


    class GuardedObject<T>{
      //受保护的对象
      T obj;
      final Lock lock =
    	new ReentrantLock();
      final Condition done =
    	lock.newCondition();
      final int timeout=1;
      //获取受保护对象  
      T get(Predicate<T> p) {
    	lock.lock();
    	try {
      		//MESA管程推荐写法
      		while(!p.test(obj)){
    			done.await(timeout,
      			TimeUnit.SECONDS);
      		}
    	}catch(InterruptedException e){
      		throw new RuntimeException(e);
    	}finally{
      		lock.unlock();
    	}
    	//返回非空的受保护对象
    	return obj;
      }
      //事件通知方法
      void onChanged(T obj) {
    	lock.lock();
    	try {
      		this.obj = obj;
      		done.signalAll();
    	} finally {
      		lock.unlock();
    	}
      }
    }


### 扩展 guarded Suspension 模式

GuardedObject 里面维护一个唯一性 ID 和 GuardedObject 对象的 Map 对象，这样就可以查找到对应的 GuardedObject 了，实现唤醒。


    class GuardedObject<T>{
      //受保护的对象
      T obj;
      final Lock lock =
    new ReentrantLock();
      final Condition done =
    lock.newCondition();
      final int timeout=2;
      //保存所有GuardedObject
      final static Map<Object, GuardedObject>
      gos=new ConcurrentHashMap<>();
      //静态方法创建GuardedObject
      static <K> GuardedObject
      create(K key){
    GuardedObject go=new GuardedObject();
    gos.put(key, go);
    return go;
      }
      static <K, T> void
      fireEvent(K key, T obj){
    GuardedObject go=gos.remove(key);
    if (go != null){
      go.onChanged(obj);
    }
      }
      //获取受保护对象  
      T get(Predicate<T> p) {
    lock.lock();
    try {
      //MESA管程推荐写法
      while(!p.test(obj)){
    done.await(timeout,
      TimeUnit.SECONDS);
      }
    }catch(InterruptedException e){
      throw new RuntimeException(e);
    }finally{
      lock.unlock();
    }
    //返回非空的受保护对象
    return obj;
      }
      //事件通知方法
      void onChanged(T obj) {
    lock.lock();
    try {
      this.obj = obj;
      done.signalAll();
    } finally {
      lock.unlock();
    }
      }
    }


## 32 | Balking模式：再谈线程安全的单例模式

当状态满足某个条件时，执行某个业务逻辑，其本质就是一个 if 而已，在多线程场景里，就是一种“多线程版本的 if”。总结成设计模式，叫做 Balking 模式。


### Balking 模式的经典实现

    boolean changed=false;
    //自动存盘操作
    void autoSave(){
      synchronized(this){
    	if (!changed) {
      		return;
    	}
    	changed = false;
      }
      //执行存盘操作
      //省略且实现
      this.execSave();
    }
    //编辑操作
    void edit(){
      //省略编辑逻辑
      ......
      change();
    }
    //改变状态
    void change(){
      synchronized(this){
    	changed = true;
      }
    }


### 用 volatile 实现 Balking 模式

使用 volatile 的前提是对原子性没有要求。


    //路由表信息
    public class RouterTable {
      //Key:接口名
      //Value:路由集合
      ConcurrentHashMap<String, CopyOnWriteArraySet<Router>>
    rt = new ConcurrentHashMap<>();
      //路由表是否发生变化
      volatile boolean changed;
      //将路由表写入本地文件的线程池
      ScheduledExecutorService ses=
    Executors.newSingleThreadScheduledExecutor();
      //启动定时任务
      //将变更后的路由表写入本地文件
      public void startLocalSaver(){
    ses.scheduleWithFixedDelay(()->{
      autoSave();
    }, 1, 1, MINUTES);
      }
      //保存路由表到本地文件
      void autoSave() {
    if (!changed) {
      return;
    }
    changed = false;
    //将路由表写入本地文件
    //省略其方法实现
    this.save2Local();
      }
      //删除路由
      public void remove(Router router) {
    Set<Router> set=rt.get(router.iface);
    if (set != null) {
      set.remove(router);
      //路由表已发生变化
      changed = true;
    }
      }
      //增加路由
      public void add(Router router) {
    Set<Router> set = rt.computeIfAbsent(
      route.iface, r ->
    new CopyOnWriteArraySet<>());
    set.add(router);
    //路由表已发生变化
    changed = true;
      }
    }


单次初始化场景：


    class InitTest{
      boolean inited = false;
      synchronized void init(){
    	if(inited){
      		return;
    	}
    	//省略doInit的实现
    	doInit();
    	inited=true;
      }
    }


单例模式的双重检查方案


    class Singleton{
      private static volatile
    	Singleton singleton;
      //构造方法私有化  
      private Singleton() {}
      //获取实例（单例）
      public static Singleton
      getInstance() {
      	//第一次检查
    	if(singleton==null){
      		synchronize{Singleton.class){
    			//获取锁后二次检查
    			if(singleton==null){
      				singleton=new Singleton();
    			}
      		}
      	}
      	return singleton;
      }
    }


双重检查中的第一次检查，完全是出于对性能的考量：避免执行加锁操作，因为加锁操作很耗时。加锁之后的二次检查，则是出于对安全性负责。


## 33 | Thread-Per-Message模式：最简单实用的分工方法

解决并发编程问题，首要问题也是解决宏观的分工问题。


### 如何理解 Thread-Per-Message 模式

委托他人办理的方式，叫做 Thread-Per-Message 模式，简而言之就是为每个任务分配一个独立的线程。


### 用 Thread 实现 Thread-Per-Message 模式

Java 线程是和操作系统线程一一对应的，这种做法本质上是将 Java 线程的调度权完全委托给操作系统。这样的好处是稳定、可靠。缺点是创建成本高。线程池是一种解决方案。

另外一种方案是：轻量级线程。轻量级线程创建的成本很低，在创建速度和内存占用比操作系统线程提升了至少一个数量级。例如go 语言、Lua 语言中的协程。


### 用 Fiber 实现 Thread-Per-Message 模式

OpenJDK 有个 Loom项目，其中的轻量级线程叫做 Fiber 。


    final ServerSocketChannel ssc =
      ServerSocketChannel.open().bind(
    new InetSocketAddress(8080));
    //处理请求
    try{
      while (true) {
    // 接收请求
    final SocketChannel sc =
      serverSocketChannel.accept();
    Fiber.schedule(()->{
      try {
    // 读Socket
    ByteBuffer rb = ByteBuffer
      .allocateDirect(1024);
    sc.read(rb);
    //模拟处理请求
    LockSupport.parkNanos(2000*1000000);
    // 写Socket
    ByteBuffer wb =
      (ByteBuffer)rb.flip()
    sc.write(wb);
    // 关闭Socket
    sc.close();
      } catch(Exception e){
    throw new UncheckedIOException(e);
      }
    });
      }//while
    }finally{
      ssc.close();
    }


可以使用 apache betch 压测工具测试。


## 34 | Worker Thread模式：如何避免重复创建线程？

在 Java 中频繁地创建、销毁线程非常影响性能，同时无限制地创建线程还可能导致 OOM。使用 Worker Thread 模式可以避免频繁的创建、销毁线程。


### Worker Thread 模式及其实现

Worker Thread 模式可以类比现实生活中的车间里的工人场景，有活大家一起干，没活歇着。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191124180738800.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 正确地创建线程池

应该创建有限数量的线程和有界的任务队列。同时也应该指明拒绝策略，用于当请求量大于有界队列的容量时，合理的拒绝请求。为了便于调试和诊断问题，建议给线程赋予一个业务相关的名字。


    ExecutorService es = new ThreadPoolExecutor(
      50, 500,
      60L, TimeUnit.SECONDS,
      //注意要创建有界队列
      new LinkedBlockingQueue<Runnable>(2000),
      //建议根据业务需求实现ThreadFactory
      r->{
    	return new Thread(r, "echo-"+ r.hashCode());
      },
      //建议根据业务需求实现RejectedExecutionHandler
      new ThreadPoolExecutor.CallerRunsPolicy());
    

### 避免线程死锁

如果提交到相同线程池的任务不是相互独立的，而是有依赖关系的，那么就有可能导致线程死锁。

出现问题时，可以通过查看线程栈信息来排查。解决方案可以为不同类型的任务创建不同的线程池。


## 35 | 两阶段终止模式：如何优雅地终止线程？

### 如何理解两阶段终止模式

两阶段终止模式，就是把终止过程分成两个阶段，其中第一个阶段主要是线程 T1 向线程 T2 发送终止指令，第二个阶段则是线程 T2 响应终止指令。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191129101834565.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


Java 线程进入终止状态的前提是线程进入 RUNNABLE 状态，当线程处于休眠状态时，可以通过 interrupt() 方法，将休眠状态的线程转换到 RUNNABLE 状态。

RUNNABLE 状态转换到终止状态，优雅的方式是让 Java 线程自己执行完 run() 方法。一般采用的方法是设置一个标志位，然后线程在合适的时机检查这个标志位，如果发现符合终止条件，则自动退出 run() 方法。

终止指令包括两方面内容： interrupt()  方法和线程终止的标志位。


### 用两阶段终止模式终止监控操作

注意 JVM 的异常处理会清除线程的中断状态，所以应该设置自己的线程终止标志位。


    class Proxy {
      //线程终止标志位
      volatile boolean terminated = false;
      boolean started = false;
      //采集线程
      Thread rptThread;
      //启动采集功能
      synchronized void start(){
    //不允许同时启动多个采集线程
    if (started) {
      return;
    }
    started = true;
    terminated = false;
    rptThread = new Thread(()->{
      while (!terminated){
    //省略采集、回传实现
    report();
    //每隔两秒钟采集、回传一次数据
    try {
      Thread.sleep(2000);
    } catch (InterruptedException e){
      //重新设置线程中断状态
      Thread.currentThread().interrupt();
    }
      }
      //执行到此处说明线程马上终止
      started = false;
    });
    rptThread.start();
      }
      //终止采集功能
      synchronized void stop(){
    //设置中断标志位
    terminated = true;
    //中断线程rptThread
    rptThread.interrupt();
      }
    }


### 如何优雅地终止线程池

线程执行 shutdown() 方法后会拒绝接收新的任务，但是会等待线程池中正在执行的任务和已经进入阻塞队列的任务都执行完之后才最终关闭线程池。

线程执行 shutdownNow() 方法后，会拒绝接收新任务，同时还会中断线程池中正在执行的任务，阻塞队列中的任务会作为返回值返回。


## 36 | 生产者-消费者模式：用流水线思想提高效率

### 生产者-消费者模式的优点

生产者-消费者模式的核心是一个任务队列，生产者线程生产任务，并将任务添加到任务队列中，而消费者线程从任务队列中获取任务并执行。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191129102007671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

生产者-消费者模式的一个重要的优点就是解耦，生产者和消费者之间没有任何依赖关系，它们之间的通信只能通过任务队列。

生产者-消费者模式还有一个重要的优点是支持异步，生产者线程只需将任务添加到任务队列而无需等待任务被执行完。并且能够平衡生产者和消费者的速度差异，通过任务队列来平衡它们之间的速度差异。


### 支持批量执行以提升性能

生产者-消费者模式比较适合应用的一个场景是批量执行任务。当批量执行多个任务的性能高于单个多次执行任务时，就可以考虑使用生产者-消费者模式了。消费者可以从队列中获取多个任务，然后再一次批量执行，来提升性能。


### 支持分阶段提交以提升性能

可以制定不同的规则，来控制执行任务的时间或者数量。例如日志刷盘，不同等级、时间等规则进行刷盘操作。


## 37 | 设计模式模块热点问题答疑

### 避免共享的设计模式

Immutability 模式、Copy-on-Write 模式和线程本地存储模式本质上都是为了避免共享。使用 Immutability 模式需要注意对象属性的不可变性，使用 Copy-on-Write 模式需要注意性能问题，使用线程本地存储需要注意异步执行问题，ThreadLocal 是线程封闭的，不同线程之间不可共享。


### 多线程版本 IF 的设计模式

Guarded Suspension 模式和 Balking 模式都可以简单的理解为“多线程版本的 if”，区别在于前者会等待 if 条件变为真，后者则不需要等待。


### 三种最简单的分工模式

Thread-Per-Message 模式、Worker Thread 模式和生产者 - 消费者模式是三种最简单使用的多线程分工方法。


## 38 | 案例分析（一）：高性能限流器Guava RateLimiter

### 经典限流算法：令牌桶算法

Guava 采用的是令牌桶算法，其核心是要想通过限流器，必须拿到令牌。算法如下：

1. 令牌以固定的速率添加到令牌桶中，假设限流的速率是 r/秒，则令牌每 1/r 秒会添加一个；
2. 假设令牌桶的容量是 b ，如果令牌桶已满，则新的令牌会被丢弃；
3. 请求能够通过令牌桶的前提是令牌桶中有令牌。

这里的容量 b 是 burst 的缩写，意义是限流器允许的最大突发流量。同时能够获得 b 个令牌。

在高并发场景下，当系统压力已经临近极限的时候，定时器的精度误差会非常大，同时定时器本身会创建调度线程，也会对系统的性能产生影响。


### Guava 如何实现令牌桶算法

Guava 实现令牌桶算法的关键是记录并动态计算下一令牌发放的时间。

    class SimpleLimiter {
      //当前令牌桶中的令牌数量
      long storedPermits = 0;
      //令牌桶的容量
      long maxPermits = 3;
      //下一令牌产生时间
      long next = System.nanoTime();
      //发放令牌间隔：纳秒
      long interval = 1000_000_000;
      
      //请求时间在下一令牌产生时间之后,则
      // 1.重新计算令牌桶中的令牌数
      // 2.将下一个令牌发放时间重置为当前时间
      void resync(long now) {
    if (now > next) {
      //新产生的令牌数
      long newPermits=(now-next)/interval;
      //新令牌增加到令牌桶
      storedPermits=min(maxPermits,
    storedPermits + newPermits);
      //将下一个令牌发放时间重置为当前时间
      next = now;
    }
      }
      //预占令牌，返回能够获取令牌的时间
      synchronized long reserve(long now){
    resync(now);
    //能够获取令牌的时间
    long at = next;
    //令牌桶中能提供的令牌
    long fb=min(1, storedPermits);
    //令牌净需求：首先减掉令牌桶中的令牌
    long nr = 1 - fb;
    //重新计算下一令牌产生时间
    next = next + nr*interval;
    //重新计算令牌桶中的令牌
    this.storedPermits -= fb;
    return at;
      }
      //申请令牌
      void acquire() {
    //申请令牌时的时间
    long now = System.nanoTime();
    //预占令牌
    long at=reserve(now);
    long waitTime=max(at-now, 0);
    //按照条件等待
    if(waitTime > 0) {
      try {
    TimeUnit.NANOSECONDS
      .sleep(waitTime);
      }catch(InterruptedException e){
    e.printStackTrace();
      }
    }
      }
    }


### 总结

经典的限流算法有：令牌桶算法（Token Bucket），另一个是漏桶算法（Leaky Bucket）。令牌桶算法是从令牌桶中取令牌，只有取到才能通过限流器。漏桶算法是按照一定的速率将水漏掉，只有还能注水的时候，才能通过限流器。


## 39 | 案例分析（二）：高性能网络应用框架Netty

### 网络编程的性能瓶颈

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191202123221539.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191202123244305.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


采用 reactor 模式来实现非阻塞式 API ，从而实现一个线程处理多个连接。


### Reactor 模式

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019120212330420.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

主程序启动 Reactor 模式，会循环调用 handle_events() 方法，该方法的核心逻辑是：首先通过同步时间多路选择器提供的 select() 方法监听网络事件，当有网络事件就绪后就遍历事件处理器处理该网络事件。


### Netty 中的线程模型

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191202123335253.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


一个网络连接在生命周期内只对应一个 EventLoop，一个 EventLoop 对应一个Java线程，但是在 NIO 模型中一个 EventLoop 对应多个连接。

处理 TCP 连接请求和读写请求是通过两个不同的 socket 完成的。在 Netty 中，bossGroup 就用来处理连接请求，而 workerGroup 是用来处理读写请求的。bossGroup 处理完连接请求后，会将这个连接交个workerGroup 处理。


### 用 Netty 实现 Echo 程序服务器

默认情况下，Netty 会创建“2*CPU 核数”个 EventLoop。永远不要将一个长时间运行的任务放入到执行队列中，因为它将阻塞需要在同一个线程上执行的任何其他任务。如果必须要要进行阻塞调用或者执行长时间运行的任务，建议使用一个专门的EventExecutor，就是在添加事物处理器handler时（add...方法）指定EventExecutor。当事件到达该handler 时会被这个EventExecutorGroup 中的某个 EventExecutor 处理，从而把它从该 channel 绑定的 EventLoop 队列中移除，这样 EventLoop 能够处理其他的事件任务。

    //事件处理器
    final EchoServerHandler serverHandler
      = new EchoServerHandler();
    //boss线程组  
    EventLoopGroup bossGroup
      = new NioEventLoopGroup(1);
    //worker线程组  
    EventLoopGroup workerGroup
      = new NioEventLoopGroup();
    try {
      ServerBootstrap b = new ServerBootstrap();
      b.group(bossGroup, workerGroup)
       .channel(NioServerSocketChannel.class)
       .childHandler(new ChannelInitializer<SocketChannel>() {
     @Override
     public void initChannel(SocketChannel ch){
       ch.pipeline().addLast(serverHandler);
     }
    });
      //bind服务端端口  
      ChannelFuture f = b.bind(9090).sync();
      f.channel().closeFuture().sync();
    } finally {
      //终止工作线程组
      workerGroup.shutdownGracefully();
      //终止boss线程组
      bossGroup.shutdownGracefully();
    }
    
    //socket连接处理器
    class EchoServerHandler extends
    ChannelInboundHandlerAdapter {
      //处理读事件  
      @Override
      public void channelRead(
    ChannelHandlerContext ctx, Object msg){
      ctx.write(msg);
      }
      //处理读完成事件
      @Override
      public void channelReadComplete(
    ChannelHandlerContext ctx){
      ctx.flush();
      }
      //处理异常事件
      @Override
      public void exceptionCaught(
    ChannelHandlerContext ctx,  Throwable cause) {
      cause.printStackTrace();
      ctx.close();
      }
    }



## 40 | 案例分析（三）：高性能队列Disruptor

Disruptor 是一款高性能的有界内存队列。性能高的原因如下：

1. 内存分配更加合理，使用 RingBuffer 数据结构，数组元素在初始化时全部创建，提高缓存命中率；对象循环利用，避免频繁GC;
2. 能够避免伪共享，提升缓存利用率；
3. 采用无锁算法，避免频繁加锁、解锁的性能消耗；
4. 支持批量消费，消费者可以无锁方式消费多个消息。


### RingBuffer 如何提升性能

程序的局部性原理指的是在一段时间内程序的执行会限定在一个局部范围内。时间局部性指的是程序中的某条指令一旦被执行，不久后这条指令很可能再次被执行；如果某条数据被访问，不久后这条数据很可能再次被访问。空间局部性是指某块内存一旦被访问，不久之后这块内存附近的内存也很可能被访问。

CPU 的缓存就利用了程序的局部性原理，如果程序能够利用局部性原理，那么将会提升性能。

RingBuffer 在初始化时会创建所有元素，所以这些元素的地址大概率是连续的，这就利用上了 CPU 加载缓存的优化。另外生产者在发布 Event 的时候，并不是创建一个新的 Event，而是通过 event.set() 方法更改 Event，这样 RingBuffer 创建的 Event 是可以循环利用的，还能避免频繁创建、删除 Event 导致的频繁 GC 问题。


### 如何避免伪共享

伪共享指的是由于共享缓存行导致缓存无效的场景。比如缓存行中的某个变量被修改缓存无效，导致缓存行中的其他变量也变为无效，从而不能利用缓存。

为了避免伪共享，可以通过缓存行填充的技术，使每个变量独占一个缓存行、不共享缓存行。


    //前：填充56字节
    class LhsPadding{
    	long p1, p2, p3, p4, p5, p6, p7;
    }
    class Value extends LhsPadding{
    	volatile long value;
    }
    //后：填充56字节
    class RhsPadding extends Value{
    	long p9, p10, p11, p12, p13, p14, p15;
    }
    class Sequence extends RhsPadding{
      //省略实现
    }


### Disruptor 中的无锁算法

Disruptor 使用 CAS（Compare And Swap/Set）操作。这是一个 CPU 级别的指令，原理类似于乐观锁。CAS 操作比锁消耗的资源少很多，因为它不牵扯操作系统，直接在 CPU 上操作。

当只有一个生产者时，不使用锁，也不使用 CAS。


### 总结

Disruptor 的优化思路大体分为两个方面：一个是利用无锁算法避免锁的争用，另一个是将硬件（CPU）的性能发挥极致。

Java 8 中提供了避免伪共享的注解：@sun.misc.Contended，需要设置 jvm 的参数 -XX:-RestrictContended。避免伪共享的代价是牺牲内存，请谨慎使用。


## 41 | 案例分析（四）：高性能数据库连接池HiKariCP

### 什么是数据库连接池

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191208185111411.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### FastList 解决了哪些性能问题

FastList 是 HiKariCP 新的数据结构，相较于 ArrayList 优化了 remove(Object element) 方法的查找顺序变为逆序查找。另个一是保证了get(index) 参数不会越界，没有对 index 进行越界检查。


### ConcurrentBag 解决了哪些性能问题

核心设计使用 ThreadLocal 避免部分并发问题。


## 42 | Actor模型：面向对象原生的并发模型

### Hello Actor 模型

Actor 模型本质上是一种计算模型，基本的计算单元称为 Actor。在 Actor 模型中，所有的计算都是在 Actor 中执行的。Actor 模型也被认为是一种并发计算模型。目前 Java 领域能完备地支持 Actor 模型而且比较成熟的类库就是 Akka.

    //该Actor当收到消息message后，
    //会打印Hello message
    static class HelloActor
    	extends UntypedActor {
      @Override
      public void onReceive(Object message) {
    	System.out.println("Hello " + message);
      }
    }
    
    public static void main(String[] args) {
      //创建Actor系统
      ActorSystem system = ActorSystem.create("HelloSystem");
      //创建HelloActor
      ActorRef helloActor =
    	system.actorOf(Props.create(HelloActor.class));
      //发送消息给HelloActor
      helloActor.tell("Actor", ActorRef.noSender());
    }


### 消息和对象方法的区别

Actor 内部的工作模式可以类比为只有一个消费者线程的生产者-消费者模式。Actor 中的消息机制完全是异步的。而调用对象方法是同步的。

调用对象的方法，需要持有对象的引用，所有的对象必须在同一个进程中。而在 Actor 中，发送消息和接收消息的 Actor 可以不在一个进程中，也可以不在同一台机器上。Actor 不但适合并发计算，还使用于分布式计算。


### Actor 的规范化定义

包含三部分能力：

1. 处理能力，处理接收到的消息。
2. 存储能力，Actor 可以存储自己的内部状态，且内部状态在不同 Actor 之间是绝对隔离的。
3. 通信能力，Actor 可以和其他 Actor 之间通信。


Actor 接收到一个消息后，可以做下面三件事：

1. 创建更多的 Actor；
2. 发消息给其他 Actor；
3. 确定如何处理下一条消息；

### 用 Actor 实现累加器


    //累加器
    static class CounterActor extends UntypedActor {
      private int counter = 0;
      @Override
      public void onReceive(Object message){
    //如果接收到的消息是数字类型，执行累加操作，
    //否则打印counter的值
    if (message instanceof Number) {
      counter += ((Number) message).intValue();
    } else {
      System.out.println(counter);
    }
      }
    }
    public static void main(String[] args) throws InterruptedException {
      //创建Actor系统
      ActorSystem system = ActorSystem.create("HelloSystem");
      //4个线程生产消息
      ExecutorService es = Executors.newFixedThreadPool(4);
      //创建CounterActor
      ActorRef counterActor =
    system.actorOf(Props.create(CounterActor.class));
      //生产4*100000个消息
      for (int i=0; i<4; i++) {
    es.execute(()->{
      for (int j=0; j<100000; j++) {
    counterActor.tell(1, ActorRef.noSender());
      }
    });
      }
      //关闭线程池
      es.shutdown();
      //等待CounterActor处理完所有消息
      Thread.sleep(1000);
      //打印结果
      counterActor.tell("", ActorRef.noSender());
      //关闭Actor系统
      system.shutdown();
    }


### 总结

Actor 模型理论上不保证消息百分百送达，也不保证消息送达的顺序和发送的顺序是一致的，甚至无法保证消息会被百分百处理。


## 43 | 软件事务内存：借鉴数据库的并发经验
借鉴数据库的事务管理，总结出新的并发解决方案：软件事务内存（Software Transactional Memory，简称 STM）。传统数据库事务支持4个特性：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability）。STM 不涉及持久化。


### 用 STM 实现转账

借助第三方类库 Multiverse 来实现转账程序：


    class Account{
      //余额
      private TxnLong balance;
      //构造函数
      public Account(long balance){
    	this.balance = StmUtils.newTxnLong(balance);
      }
      //转账
      public void transfer(Account to, int amt){
    	//原子化操作
    	atomic(()->{
      		if (this.balance.get() > amt) {
    		this.balance.decrement(amt);
    		to.balance.increment(amt);
      	}
    	});
      }
    }


MVCC（Multi-Version Concurrency Control）多版本并发控制。在事务开启时，会给数据库打一个快照，以后所有的读写都是基于这个快照。提交事务时，如果所有读写过的数据在该事务执行期间没有发生变化，那么就可以提交；如果发生变化，说明该事务和其他事务读写的数据冲突了，这时是不可以提交的。可以通过给每条数据增加一个版本号，来记录数据是否放生了变化。


## 44 | 协程：更轻量级的线程

协程可以理解为一种轻量级的线程。线程是在内核态中调度的，而协程是在用户态调度的。相比于线程，协程切换的成本更低。


### Golang 中的协程

    
    import (
      "fmt"
      "time"
    )
    func hello(msg string) {
      fmt.Println("Hello " + msg)
    }
    func main() {
      //在新的协程中执行hello方法
      go hello("World")
      fmt.Println("Run in main")
      //等待100毫秒让协程执行结束
      time.Sleep(100 * time.Millisecond)
    }


### 利用协程实现同步

基于协程实现同步非阻塞方案。

    -- 创建socket
    local sock = ngx.socket.tcp()
    -- 设置socket超时时间
    sock:settimeouts(connect_timeout, send_timeout, read_timeout)
    -- 连接到目标地址
    local ok, err = sock:connect(host, port)
    if not ok then
    -  -- 省略异常处理
    end
    -- 发送请求
    local bytes, err = sock:send(request_data)
    if not bytes then
      -- 省略异常处理
    end
    -- 读取响应
    local line, err = sock:receive()
    if err then
      -- 省略异常处理
    end
    -- 关闭socket
    sock:close()   
    -- 处理读取到的数据line
    handle(line)


### 结构化并发编程

三种基本的控制结构是：顺序结构、选择结构和循环结构。

上述的结构都是由一个入口和一个出口，中间不管怎样组合都最终汇聚到一个出口。而结构化并发编程，不管是创建线程还是协程，都会出现不同的分支，而且最终是不汇聚的，因此也不是线性的。所以被某些人认为是违背了结构化程序设计。


## 45 | CSP模型：Golang的主力队员

Golang 解决协程之间的协作问题：一种是支持协程之间以共享内存的方式通信，Golang 提供了管程和原子类来对协程进行同步控制；另一种方案是支持协程之间以消息传递（Message-Passing）的方式通信，本质上是要避免共享，Golang 的这个方案是基于 CSP（Communication Sequential Processes）模型实现的。


### 什么是 CSP 模型

CSP模型与 Actor 模型很相似。Golang 中推荐以通信的方式共享内存。Golang 中协程之间通信推荐的是使用 channel。


### CSP 模型与生产者-消费者模式

channel 可以类比为生产者-消费者模式中的阻塞队列。channel 的容量为 0 时，被称为无缓冲的 channel，容量大于 0 的则被称为有缓冲的 channel。


### CSP 模型与 Actor 模型的区别

第一个区别是：Actor 模型中没有 channel。

第二个区别是：Actor 模型中发送消息是非阻塞的，而 CSP 模型中是阻塞的。

第三个区别是：Actor 模型理论上不保证百分百送达，CSP 模型中能保证消息百分百送达，但同时有死锁的风险。































































