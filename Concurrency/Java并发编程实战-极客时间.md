## 学习攻略 | 如何才能学好并发编程？


并发编程领域抽象成三个核心问题：分工，同步和互斥。

**分工**：把一个功能分给不同的线程共同实现

**同步**：当某个条件不满足时，线程需要等待，当某个条件满足时，线程需要被唤醒执行。管程是解决并发问题的万能钥匙。

**互斥**：保证线程安全，同一时刻，只允许一个线程访问共享变量。可见性，有序性，原子性。核心技术是锁。

全景图：

![](https://img-blog.csdnimg.cn/20190304224159219.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)



## 01 | 可见性、原子性和有序性问题：并发编程Bug的源头




































