






## 20 | 幻读是什么，幻读有什么问题？

### 幻读是什么？

**注意：下面的假设场景都是不成立的，是为了反正间隙锁的存在。**

![](https://img-blog.csdnimg.cn/20190301101612691.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

幻读指的是一个事物在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行。

幻读说明：

1. 在可重复读隔离级别下，普通的查询是快照读，是不会看到别的事物插入的数据的。因此，幻读在“当前读”下才会出现。
2. 上面sessionB的修改结果，被sessionA之后的select语句用“当前读”看到，不能成为幻读。幻读仅专指“新插入的行”。


### 幻读有什么问题？

#### 1、破坏语义

![](https://img-blog.csdnimg.cn/20190301101809958.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

sessionA在T1时刻锁住了d=5的记录，但是在T2时刻sessionB更新语句可以执行，同样T4时刻sessionC的更新语句也可以执行，因为SessionA没有锁住id=5 和 id=1的行。因此就破坏了SessionA在T1时刻要锁住 d=5的数据行。


#### 2、造成数据不一致

![](https://img-blog.csdnimg.cn/20190301101911763.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

执行顺序：

![](https://img-blog.csdnimg.cn/20190301101952712.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

结果：(0,5,100)、(1,5,100) 和 (5,5,100）

-----------------------------

![](https://img-blog.csdnimg.cn/20190301102151555.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

执行顺序：
![](https://img-blog.csdnimg.cn/20190301102220374.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

结果：(0,5,5)、(1,5,100),和(5,5,100)

上面两种假设都会造成严重的数据不一致问题。


### 如何解决幻读？

产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此为了解决幻读问题，InnoDB只好引入新的锁，也就是间隙锁（Gap Lock）。

间隙锁，锁的是两个值之间的空隙。例如下面插入6条记录产生的7个间隙：
![](https://img-blog.csdnimg.cn/20190301102306632.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

当你执行 select * from t where d=5 for update 的时候，就不止是给数据库中已有的6个记录加上了行锁，还同时加了7个间隙锁。这样就确保了无法再插入新的记录。

两种类型行锁的冲突关系：
![](https://img-blog.csdnimg.cn/20190301102337375.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

但是间隙锁不一样，跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作。间隙锁之间都不存在冲突关系。间隙锁的目标是保护这个间隙，不允许插入值。但它们之间是不冲突的。

间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间。例如  (-∞,0]、(0,5]、(5,10]、(10,15]、(15,20]、(20,25]、(25，+supernum]

间隙锁和 next-key lock 的引入帮我们解决了幻读的问题。但是间隙锁的引入，可能会导致同样的语句锁住更大的范围，这就影响了并发读。

![](https://img-blog.csdnimg.cn/2019030110240342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

间隙锁是在可重复读隔离级别下才会生效的。如果把隔离级别设置为读提交的话，就没有间隙锁了。但同时要解决可能出现的数据和日志不一致的问题，需要把binlog格式设置为row。


## 21 | 为什么我只改一行的语句，锁这么多？

加锁规则：包含了两个“原则“，两个“优化”和一个“bug”。

1. 原则1: 加锁的基本单位是 next-key lock。next-key lock是前开后闭区间。
2. 原则2: 查找过程中访问到的对象才会加锁
3. 优化1: 索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁
4. 优化2: 索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。
5. 一个bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。


### 案例一：等值查询间隙锁

![](https://img-blog.csdnimg.cn/20190305223017487.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


1，根据原则1，sessionA加锁范围是 （5，10]

2，同时根据优化2，这是等值查询，而id=10不满足查询条件，next-key lock退化成间隙锁，因此最终加锁范围是 (5,10)


### 案例二：非唯一索引等值锁

覆盖索引上的锁：

![](https://img-blog.csdnimg.cn/20190305223110638.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

在这个例子中，lock in share mode 只锁覆盖索引，但是如果是 for update ，系统会认为你接下来要更新数据，因此会顺便给主键索引上满足条件的行加上锁。

锁是加在索引上的；需要注意的是，如果你要用 lock in share mode 来给行加读锁避免数据被更新的话，就必须的绕过覆盖索引的优化，在查询字段中加入索引中不存在的字段，因此查询时会回表，使用主键索引查询，从而在主键索引上加锁。例如sessionA 查询改为： select d from t where c=5 lock in share mode;


### 案例三：主键索引范围锁

![](https://img-blog.csdnimg.cn/20190305223137125.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


主键索引加锁：行锁 id=10 和 next-key lock(10, 15]


### 案例四：非唯一索引范围锁

![](https://img-blog.csdnimg.cn/20190305223215883.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


sessionA加锁是：索引c上的 (5,10],(10,15] 两个 next-key lock。主键id索引上id=10行锁。


### 案例五：唯一索引范围锁 bug

![](https://img-blog.csdnimg.cn/20190305223245835.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


由于唯一索引的范围查询会扫描到第一个不满足条件的行为止，索引sessionA的加锁范围是 (10, 15], (15, 20]。


> 林晓斌认为：既然是唯一索引，当查询到id=15行时，就应该停止向前扫描，不必加(15, 20] next-key lock。



### 案例六：非唯一索引上存在“等值”的例子

![](https://img-blog.csdnimg.cn/20190305223316511.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

delete语句加锁的逻辑，其实是跟 select ... for update 是类似的，也就是文章开始时总结的几点。

![](https://img-blog.csdnimg.cn/20190305223436975.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20190305223502172.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

蓝色区域左右两边是虚线，表示开区间，即 (c=5,id=5)  和 (c=15,id=15) 这两行上都没有锁。


### 案例七： limit 语句加锁

![](https://img-blog.csdnimg.cn/20190305223532763.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

加锁范围：(c=5,id=5) 到 (c=10,id=30)这个前开后闭区间。因为有 limit 2的限制，因此遍历到(c=10,id=30)这一行时，已满足条件，就结束了。

![](https://img-blog.csdnimg.cn/2019030522355750.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


这个例子的指导意义就是，在删除数据的时候尽量加limit。这样不仅可以控制删除数据的条数，让操作更安全，还可以减小加锁的范围。


### 案例八：一个死锁的例子

![](https://img-blog.csdnimg.cn/20190305223630346.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

sessionA在索引c上加锁： next-key lock (5,10] 和 间隙锁 (10,15)。sessionB的加锁过程是，先加间隙锁 (5,10)，加锁成功，再加行锁 c=10，这时候被锁住阻塞。sessionA这时执行insert语句，会被sessionB的间隙锁锁住。由于出现了死锁，InnoDB让sessionB回滚。

在分析加锁规则时可以用 next-key lock来分析，但是具体执行的时候，是要分成间隙锁和行锁两段来执行的。


### 小结

上面所有案例都是在可重复读隔离级别下验证的。同时，可重复读隔离级别遵守两阶段锁协议，所有加锁的资源，都是在事务提交或者回滚的时候才释放的。

在读提交隔离级别下还有一个优化，即：语句执行过程中加上的行锁，在语句执行完成后，就要把“不满足条件的行”上的行锁直接释放了，不需要等到事务提交。也就是说，读提交隔离级别下，锁的范围更小，锁的时间更短。





















