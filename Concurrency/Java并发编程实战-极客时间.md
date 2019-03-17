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






























