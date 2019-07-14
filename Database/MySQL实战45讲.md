






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



## 22 | MySQL有哪些“饮鸩止渴”提高性能的方法？

### 短连接风暴

正常的短连接模式就是连接到数据库后，执行很少的SQL语句就断开，下次需要的时候再重连。如果使用的是短连接，在业务高峰期时，就可能出现连接数突然暴涨的情况。

max_connections 参数，用来控制一个MySQL实例同时存在的连接数上限，超过这个值系统会拒绝接下来的连接请求，并报错提示“Too many connections”。

碰到超过连接池上限的情况，调高 max_connections 参数并不太可取。因为max_connections 值太大，就会有更多的连接进来，系统的负载可能会进一步加大，大量的资源耗费在权限验证等逻辑上，已经连接的线程拿不到CPU资源去执行业务的SQL请求。下面有两种有损的方案推荐给你。


#### 第一种方法：先处理掉那些占着连接但是不工作的线程。

通过 kill connection + id 主动剔除线程空闲的连接。这个行为跟事先设置 wait_timeout 的效果是一样的。wait_timeout参数表示线程空闲 wait_timeout 多秒之后，就会被MySQL直接断开连接。

如果连接数过多，你可以优先断开事务外空闲太久的连接；如果这样还不够，再考虑断开事务内空闲太久的连接。因为如果断开事务内的连接，就会造成MySQL事务回滚，对业务是有影响的。

我们可以通过 show processlist 查看那些线程处于 sleep（空闲） 状态，然后通过查 information_schema 库的 innodb_trx表，找出trx_mysql_thread_id的值，表示值为该id的线程是处于事务内的。

    mysql> show processlist;
    
    mysql> select * from information_schema.innodb_trx\G;


从数据库端主动断开连接可能是有损的，尤其是有点应用端收到错误后，不重新连接，而是直接使用这个已经不能用的句柄重试查询。这会导致应用端看上去，“MySQL一直没恢复”。


#### 第二种方法：减少连接过程的消耗

让数据库跳过权限验证阶段。跳过权限验证的方法是：重启数据库，并使用 skip-grant-tables 参数启动。这样，整个MySQL会跳过所有的权限验证阶段，包括连接过程和语句执行过程。

但这样做的风险极高，不建议使用该方案。特别是数据库可以外网访问的话。


### 慢查询性能问题

在mysql中，会引发性能问题的慢查询，大体有以下三种可能：

1. 索引没有设计好；
2. SQL语句没写好；
3. mysql 选错了索引。

#### 第一种可能：索引没有设计好

这种场景一般就是通过紧急创建索引来解决。最高效的做法就是直接执行 alter table 语句。

比较理想的是能够在备库先执行。假设现在的服务是一主一备，主库A，备库B，这个方案的执行流程是：

1. 在备库B上执行 set sql_log_bin=off，也就是不写binlog，然后执行 alter table 语句加上索引；
2. 执行主备切换；
3. 这时主库是B，备库是A。然后在A上执行1的操作。

在紧急处理时，上述方案的效率是最高的。但是在平时考虑使用 gh-ost 方案可能更稳妥。


#### 第二种可能：语句没写好

由于sql语句书写不当，导致语句没有使用上索引。这时我们可以通过改写SQL语句来处理。MySql5.7 提供了 query_rewrite 功能，可以把输入的语句改写成另一种模式。

    mysql> insert into query_rewrite.rewrite_rules(pattern, replacement, pattern_database) values ("select * from t where id + 1 = ?", "select * from t where id = ? - 1", "db1");
    
    call query_rewrite.flush_rewrite_rules();



#### 第三种可能：MySql选错了索引

这时的应急方案就是给这个语句加上 force index。同样的也可以使用查询重写功能，给原来的语句加上 force index。

在现实系统中，出现最多的是前两种情况，即：索引没设计好和语句没写好。而这两种情况可以在系统上线前避免的。

1. 测试环境，打开慢查询日志，并且设置 long_query_time = 0;
2. 在测试表里插入模拟线上的数据，做一遍回归测试；
3. 观察慢查询日志里面的每类语句的输出，特别留意 Rows_examined 字段是否与预期一致。

也可以使用开源工具 pt-query-digest 来分析。

### QPS突增问题

有时候由于业务突然出现高峰，或者应用程序bug，导致某个语句的QPS突然暴涨，也可能导致MySql压力过大，影响服务。

如果是因功能引起的，最理想的情况是让业务把新功能下掉。


## 23 | MySQL是怎么保证数据不丢的？

只要 redo log 和 binlog 保证持久化到磁盘，就能确保MySQL异常重启后，数据可以恢复。

binlog的写入机制

binlog的写入逻辑：事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。

![](https://img-blog.csdnimg.cn/201903230102392.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

每个线程都有自己的 binlog cache，但是共用一份 binlog 文件。

- 图中write，指的是把日志写入到文件系统的 page cache，由于是写入内存，并没有把数据持久化到磁盘，因此速度比较快
- 图中的fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS。


write 和 fsync 的时机，是由参数 sync_binlog 控制的：

1. 当 sync_binlog=0时，表示每次提交事务都只write，不fsync；
2. 当 sync_binlog=1时，表示每次提交事务都会执行 fsync；
3. 当 sync_binlog=N(N>1)时，表示每次提交事务都write，但累积N个事物后才fsync。

因此在出现IO瓶颈的场景中，将 sync_binlog 设置成一个比较大的值，可以提升性能。考虑到丢失日志量的可控性，比较常见的是将其设置为 100~1000 中的某个值。

将 sync_binlog 设置为N，对应的风险是：如果主机发生异常重启，会丢失最近N个事务的binlog日志。


### redo log 的写入机制

事务在执行过程中，生成的redo log 是要先写到 redo log buffer，并不是每次写入redo log buffer都要直接持久化到磁盘。在事务还没提交的时候，redo log buffer 中的部分日志有可能被持久化到磁盘。

redo log可能存在的三种状态：
![](https://img-blog.csdnimg.cn/20190323010517216.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

1、存在redo log buffer中，物理上是在MySQL进程内存中，也就是图中的红色部分；
2、写到磁盘（write），但是没有持久化（fsync），物理上是在文件系统的page cache 里面，也就是图中黄色部分；
3、持久化到磁盘，对应的是 hard disk，也就是图中绿色部分。

日志写到redo log buffer是很快的，write到 page cache 也很快，但是持久化到磁盘就慢很多了。

InnoDB提供了 innodb_flush_log_at_trx_commit 参数来控制 redo log的写入策略，它有三种可能的取值：

1. 设置为0时，表示每次事物提交时都只是把redo log留在 redo log buffer 中；
2. 设置为1时，表示每次事务提交时都将 redo log 直接持久化到磁盘；
3. 设置为2时，表示每次事物提交时都只是把redo log 写到 page cache。

InnoDB有一个后台线程，每隔1秒，就会把 redo log buffer中的日志，调用write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。

**注意**，事务在执行过程中的 redo log 也是会直接写入 redo log buffer中的，这些 redo log 也会被后台线程一起持久化到磁盘。也就是说，一个还没提交的事物的 redo log，也是可能被持久化到磁盘的。

除了后台线程每秒一次的轮询操作外，还有两种场景会把一个没提交事物的 redo log 写入到磁盘中。

1. redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘。由于这个事物并没有提交，所以只调用 write， 而没有调用 fsync。redo log留在了 page cache。
2. 并行事务提交时，顺带将这个事务的 redo log buffer 持久化到磁盘。事务A执行到一半，事务B提交，如果 innodb_flush_log_at_trx_commit 参数设置为1，那么事务B要不 redo log buffer 里的日志持久化到磁盘。这时就会带上 事务A 在 redo log buffer中的日志一起持久化到磁盘。

两阶段提交：redo log 先 prepare，再写 binlog， 最后再把 redo log commit。

双1配置指的是将 sync_binlog 和 innodb_flush_log_at_trx_commit 都设置成1。也就是说，一个事务完整提交前，需要等待两次刷盘，一次是 redo log（prepare阶段），一次是binlog。

日志逻辑序列号（log sequence number）LSN是单调递增的，用来对应 redo log 的一个个写入点。每写入length长度的redo log， LSN的值就会增加 length。

组提交：
![](https://img-blog.csdnimg.cn/2019032301085238.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)
![](https://img-blog.csdnimg.cn/20190323011000512.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

上图是三个并发事务（trx1、trx2、trx3）在 prepare 阶段，都写完了 redo log buffer，持久化到磁盘的过程，对应的LSN分别是 50、120、160。

1. trx1是第一个到达的，会被选为这组的leader；
2. 等 trx1 要开始写盘时，这个组里已经有了三个事务，这时候 LSN 也变成了160；
3. trx1去写盘时，带的就是LSN=160，因此等 trx1 返回时，LSN小于160的redo log ，都已经持久化到磁盘了；
4. 这时 trx2 和 trx3 可以直接返回了。

因此，一次组提交里面，组员越多，节约磁盘 IOPS 的效果越好。

MySQL为了让组提交的效果更好，把redo log做fsync的时间拖到了步骤1之后。
![](https://img-blog.csdnimg.cn/20190323011208147.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

这样一来 binlog 也可以组提交了，不过由于第3步执行的很快，导致能集合到一起持久化的binlog比较少。如果想提升 binlog 组提交的效果，可以通过设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 来实现。

1、binlog_group_commit_sync_delay参数表示延迟多少个微秒后才调用 fsync；
2、binlog_group_commit_sync_no_delay_count 参数，表示累计多少次后才调用 fsync。

两个条件是或的关系，也就是只要满足其中一个就会调用 fsync。

WAL机制主要得益于两个方面：

1. redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度快；
2. 组提交机制，可以大幅降低磁盘的 IOPS 消耗。


如果你的MySQL出现性能瓶颈，而且瓶颈在IO上，那么可以通过下面三种方法提升性能：

1. 设置binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count参数，减少binlog的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险。
2. 将 sync_binlog 设置为大于1的值（比较常见的是 100 ~ 1000）。这样做的风险是，主机掉电时会丢binlog日志。
3. 将 innodb_flush_log_at_trx_commit 设置为2，这样做的风险是，主机掉电的时候会丢失数据。


## 24 | MySQL是怎么保证主备一致的？

MySQL主备的基本原理
![](https://img-blog.csdnimg.cn/20190328202726170.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


在状态1中，客户端读写都直接访问节点A，而节点B是A的备库，只是将A的更新都同步过来，到本地执行。建议将节点B（备库）设置成只读（readonly）模式。原因如下：

1. 有时候一些查询会被放到备库上查询，设置只读可以防止误操作；
2. 防止切换逻辑有bug，比如切换过程中出现双写，造成主备不一致；
3. 可以用readonly状态，来判断节点的角色。

一条更新语句从节点A到节点B的内部流程：
![](https://img-blog.csdnimg.cn/20190328202758808.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

备库B和主库A之间维持了一个长连接。主库A内部有一个线程，专门用于服务备库B的这个长连接。一个事务日志同步过程是这样的：

1. 在备库B上通过 change master 命令，设置主库A的IP、端口、用户名、密码，以及要从哪个位置开始请求binlog，这个位置包含文件名和日志偏移量。
2. 在备库B上执行 start slave 命令，这时候备库会启动两个线程，就是图中的 io_thread 和 sql_thread。其中 io_thread 负责与主库建立连接。
3. 主库A校验完用户名、密码后，开始按照备库B传过来的位置，从本地读取binlog，发给B。
4. 备库B拿到binlog后，写到本地文件，称为中转日志（relay log）。
5. sql_thread 读取中转日志，解析日志里面的命令，并执行。


### binlog 的三种格式对比

binlog有三种格式：statement、row、mixed

当执行以下SQL语句后，不同的binlog日志格式会有不同的记录内容：

    mysql> delete from t /*comment*/  where a>=4 and t_modified<='2018-11-10' limit 1;

对于statement格式的binlog，可以通过以下语句查看记录：

    mysql> show binlog events in 'master.000001';

![](https://img-blog.csdnimg.cn/20190328203028112.png)

可以看到statement格式的binlog会记录完整执行的SQL语句。但是这会可能在某些情况下出现主备数据不一致的情况，上述执行的SQL语句中两个判断条件，并且还有一个limit 1 的限制，如果主库delete语句使用的是索引a，备库delete语句使用的是索引t_modified，就可能两个库删除的不是同一条记录，从而造成数据不一致的情况。

通过 show warnings;  可以看到是有告警信息的。

对于row格式的binlog存储内容：

![](https://img-blog.csdnimg.cn/20190328203140716.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

使用 mysqlbinlog 命令来解析：

    mysqlbinlog  -vv data/master.000001 --start-position=8900;

![](https://img-blog.csdnimg.cn/20190328203211615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

可以看到，当 binlog_format 使用的是 row 格式的时候，binlog里面记录了真实删除行的主键id，这样binlog传到备库去执行的时候，肯定会删除 id=4的行，就不会出现主备删除不同行的问题了。


### 为什么会有 mixed 格式的binlog？

mixed格式指的是statement 和 row 格式同时存在。考虑到statement格式可能会导致主备不一致，所以要使用 row 格式。但是row 格式又太占空间了，而且还消耗 IO 资源，影响执行速度。因此就有了 mixed 格式，当mysql认为这条 SQL 语句可能引起主备不一致时，就用 row 格式，否则就用 statement 格式。

因此，线上的MySQL设置的binlog格式至少应该是 mixed，现在越来越多的场景要求把格式设置成row，这么做的一个直接的好处就是恢复数据。

用 binlog 来恢复数据的标准做法是，用 mysqlbinlog 工具解析出来，然后把解析结果整个发给 MySQL 执行。类似下面的命令：


    mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;


### 循环复制问题

双M结构：

![](https://img-blog.csdnimg.cn/20190328203358656.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

双 M 结构，即：节点A和B之间总是互为主备关系。这样在切换的时候就不用再修改主备关系。

但是有一个问题需要解决，就是节点A上更新来一条语句，然后把生成的 binlog 发给节点 B，节点B执行完这条更新语句后也会生成binlog。如果节点A同时是B的备库，相当于又把节点 B 新生成的 binlog 拿过来执行来一次，然后节点 A 和 B间，会不断地循环执行这个跟新语句，也就是循环复制了。

MySQL在binlog 中记录了这个命令第一次执行时所在实例的 server id。因此，用下面的逻辑来解决循环复制的问题：

1. 规定两个库的 server id 必须不同，如果相同，则它们之间不能设定为主备关系；
2. 一个备库接到 binlog 并在重放的过程中，生成与原 binlog 的 server id 相同的新的 binlog;
3. 每个库在收到从自己的主库发过来的日志后，先判断server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。


## 25 | MySQL是怎么保证高可用的？

正常情况下，只要主库执行更新生成的所有binlog，都可以传到备库并被正确执行，备库就能达到跟主库一致的状态，这就是最终一致性。

![](https://img-blog.csdnimg.cn/20190407153427810.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### 主备延迟

所谓主备延迟，就是同一个事务，在备库执行完成的时间和主库执行完成的时间之间的差值。

在备库上执行  show slave status 命令，它的返回结果里面会显示 second_behind_master，用于表示当前主备延迟了多少秒。

如果主备库时间不一致，并不会导致主备延迟的值不准。因为备库在连接主库后会通过 SELECT UNIX_TIMESTAMP() 函数来获得当前主库的系统时间。在计算second_behind_master 时会扣除掉这个值。

网络正常时，主库传输binlog到备库的网络耗时是很少的，主备延迟最直接的表现是，备库消费中转日志（relay log）的速度，比主库生成 binlog 的速度要慢。


### 主备延迟的来源

第一种是在有些部署条件下，备库所在机器的性能要比主库所在的机器性能差。

第二种是备库的压力大。在主备选用相同规格的机器后，备库可能会用于提供一些读能力。当备库上的查询消耗了大量的 CPU 资源，影响了同步速度，就会造成主备延迟。通常的解决方案如下：

1. 一主多从。除了备库外，可以多接几个从库，让这些从库来分担读的压力。备库和从库概念上差不多，把在 HA过程中被选成新主库的，称为备库。这种方案通常会被采用，从库可以用来做备份。
2. 通过binlog输出到外部系统，比如 Hadoop 这类系统，让外部系统提供统计类查询的能力。

第三种是大事物。大事物的执行时间过长，主库上必须等事物执行完成才会写入 binlog，再传给备库。比如一次性地用 delete 语句删除太多数据。还有大表的DDL。

第四种是备库的并行复制能力。


### 可靠性优先策略

![](https://img-blog.csdnimg.cn/20190407153651540.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

图中的SBM是参数 second_behind_master 参数的简写。

可靠性优先策略中是有不可用状态的，判断SBM=0的过程中，主库和备库都是只读状态，当SBM=0时才把B设置为可写状态。这端时间内数据库是不可以写数据的。


### 可用性优先策略

如果强行切换，不判断 SBM 是否等于0，直接把连接切到备库B，并且让备库可以读写，那么系统几乎就没有不可用时间了。但是这样切换的代价就是可能出现数据不一致的情况。

![](https://img-blog.csdnimg.cn/20190407153732550.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20190407153757901.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

从上面的分析中，可以得出一些结论：

1. 使用 row 格式的binlog时，数据不一致的问题更容易被发现。
2. 主备切换的可用性优先策略会导致数据不一致。因此大多数情况下，建议使用可靠性优先策略。

可靠性优先策略下，异常切换的效果。主备延迟30分钟，主库A掉电，HA系统要切换B作为主库。

![](https://img-blog.csdnimg.cn/20190407153928370.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

在满足数据可靠性的前提下，MySQL高可用系统的可用性，是依赖于主备延迟的。延迟的时间越小，在主库故障的时候，服务恢复需要的时间就越短，可用性就越高。


## 26 | 备库为什么会延迟好几个小时？

![](https://img-blog.csdnimg.cn/20190414154751308.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

第一个黑色箭头表示客户端写入主库，第二个黑色箭头表示备库上 sql_thread 执行中转日志（relay log）。如果客户端写入主库的并发度远大于备库执行中转日志的并发度，那么将会造成严重地主备延迟。

![](https://img-blog.csdnimg.cn/20190414154824692.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

coordinator 就是原来的 sql_thread，不过现在它不再是直接更新数据了，只负责读取中转日志和分发事务。真正的日志更新变成了worker线程。worker线程的个数是由参数 slave_parallel_workers 决定的。根据经验，值设置为 8~16 之间最好（32核物理机）。

coordinator在分发的时候，需要满足以下两个基本要求：

1. 不能造成更新覆盖。这就要求更新同一行的两个事务，必须被分发到同一个worker中。
2. 同一个事务不能被拆开，必须放到同一个worker中。

### MySQL 5.5 版本的并行复制策略

MySQL5.5版本是不支持并行复制的，备库只能单线程复制。 林实现了两种并行策略：按表分发策略和按行分发策略。


#### 按表分发策略

按表分发事务的基本思路是，如果两个事务更新不同的表，它们就可以并行。因为数据是存储在两个不同的表里，所以按表分发，可以保证两个 worker 不会更新同一行。如果有跨表的事务，还是要放在一起考虑的。

![](https://img-blog.csdnimg.cn/20190414154941955.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


假设事务T要修改表 t1 和 t3，事务T的分配流程：

1. 由于事务 T 中涉及修改表 t1，而worker1队列中有事务在修改表 t1，事务T和队列中的某个事务要修改同一个表的数据，这种情况我们说事务T和worker_1是冲突的；
2. 按照这个逻辑，顺序判断事务 T 和每个 worker 队列的冲突关系，会发现事务 T 跟 worker_2 也冲突；
3. 当事务 T 跟多于一个 worker 冲突时，coordinator线程就会进入等待；
4. 每个 worker 继续执行，同时修改 hash_table。假设 worker_2 中涉及到修改表 t3 的事务执行完成，就会从 hash_table_2 中把 db1.t3 这一项去掉；
5. 这样 coordinator 会发现跟事务 T 冲突的 worker 只有 worker_1 ，因此就把它分配给 worker_1；
6. coordinator 继续读下一个中转日志，继续分配事务。

事务在分发时，跟 worker 的冲突关系：

1. 如果跟所有 worker 都不冲突，coordinator 线程就会把这个事务分配给最空闲的worker；
2. 如果只跟一个 worker 冲突，coordinator 线程就会把事务分配个这个存在冲突关系的 worker；
3. 如果跟多于一个的worker冲突，coordinator 线程就进入等待状态，直到和这个事务存在冲突关系的worker只剩下1个；


#### 按行分发策略

解决热点表的并行复制问题，就需要一个按行并行复制的方案。按行复制的核心思路是：如果两个事务没有更新相同的行，它们在备库上可以同时执行。显然，这个模式要求 binlog 格式必须是 row。

这时候，我们判断一个事务 T 和 worker 是否冲突，用的就不是“修改同一个表”，而是修改同一行。

基于行的策略，事务 hash 表中还需要考虑唯一键。即 key 应该是 “库名 + 表名 + 索引 a + a 的值”。可见相比于按表并行分发策略，按行并行策略在决定线程分发的时候，需要消耗更多的计算资源。

两个方案都有一些约束条件：

1. 要能够从 binlog 里面解析出表名、主键值和唯一索引的值。也就是说，主库的 binlog 格式必须是 row；
2. 表必须有主键；
3. 不能有外键。

虽然按行分发策略的并行度更高。不过，如果是要操作很多行的大事务的话，按行分发的策略有两个问题：

1. 耗费内存。hash表存储更多的数据
2. 耗费 CPU。解析 binlog，计算 hash 值。

可以通过设置阈值，单个事务如果超过设置的阈值，就暂时退化为单线程模式。


### MySQL 5.6 版本的并行复制策略

官方5.6版本支持了并行复制，只是支持的粒度是按库并行。

### MariaDB 的并行复制策略

MariaDB 策略，目标是“模拟主库的并行模式”。但它并没有实现“真正的模拟主库并发度”这个目标。

MariaDB 的并行复制策略利用了 redo log 组提交优化的特性：

1. 能够在同一组里提交的事务，一定不会修改同一行；
2. 主库上可以并行执行的事务，备库上也一定可以并行执行。

具体实现： 

1. 在一组里面一起提交的事务，有一个相同的commit_id，下一组就是 commit_id + 1;
2. commit_id 直接写到 binlog 里面；
3. 传到备库应用的时候，相同 commit_id 的事务分发到多个 worker 执行；
4. 这一组全部执行完成后，coordinator 再去取下一批。

![](https://img-blog.csdnimg.cn/20190414155333438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20190414155434462.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

可以看到，在主库执行的时候，多组事务是并行执行的，但在从库上同时只能一组事务执行，这样系统的吞吐量就不够。这个方案很容易被大事务拖后腿，如果 trx2 是一个超大事务，那么在 trx1 和 trx3执行完后，就只有一个worker线程在工作了，是对资源的浪费。


### MySQL 5.7 的并行复制策略

参数 slave-parallel-type 来控制并行复制策略：

1. 配置为 DATABASE, 表示使用 MySQL 5.6 版本的库并行策略；
2. 配置为 LOGICAL_CLOCK，表示使用优化过的类似 MariaDB 的策略。


MySQL 5.7 并行复制策略的思想是：

1. 同时处于 prepare 状态的事务，在备库执行时是可以并行的；
2. 处于 prepare 状态的事务，与处于 commit 状态的事务之间，在备库执行时也是可以并行的。

用于控制binlog 从 write 到 fsync 时间的参数，既可以减少 binlog 的写盘次数，也可以制造更多的“同时处于 prepare 阶段的事务”，从而提升备库复制并发度的目的。


### MySQL 5.7.22 的并行复制策略

新增了一个新的并行复制策略，基于 WRITESET 的并行复制。由参数 binlog-transaction-dependency-tracking，用来控制是否启用这个新策略。可取值：

1. COMMIT_ORDER，表示就是前面介绍的，根据同时进入 prepare 和 commit 来判断是否可以并行的策略；
2. WRITESET，表示的是对于事务涉及更新的每一行，计算出这一行的hash值，组成 writeset 集合。如果两个事务没有操作相同的行，也就是 writeset 没有交集，就可以并行；
3. WRITESET_SESSION，在 WRITESET 的基础上多了一个约束，在主库上同一个线程先后执行的事务，在备库也要保证相同的执行顺序。


## 27 | 主库出问题了，从库怎么办？

一主多从结构：

![](https://img-blog.csdnimg.cn/20190419225833201.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


图中虚线箭头表示的是主备关系，也就是 A 和 A' 互为主备关系。从库 B、C、D 指向的是主库A。一主多从的设置，一般用于读写分离，主库负责所有的写和一部分读，其他的读请求由从库分担。

在一主多从架构下，主备切换问题：

![](https://img-blog.csdnimg.cn/20190419225902947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

相比于一主一备的切换流程，一主多从在切换完成后，A'会成为新的主库，从库 B、C、D 也要改连接到 A'。从而主备切换的复杂性也相应增加了。


### 基于位点的主备切换

我们把节点B设置成A'的从库的时候，需要执行一条 change master 命令：

    
    CHANGE MASTER TO
    MASTER_HOST=$host_name
    MASTER_PORT=$port
    MASTER_USER=$user_name
    MASTER_PASSWORD=$password
    MASTER_LOG_FILE=$master_log_name
    MASTER_LOG_POS=$master_log_pos  

前四个参数分别是主库的 IP、端口、用户名和密码。最后两个参数是要从主库的 master_log_name 文件的 master_log_pos 这个位置的日志继续同步。这个位置就是我们所说的同步位点。

同步位点的获取是不精确的，考虑到切换过程中不能丢失数据，所以找位点的时候，总是要找一个“稍微往前”的，再通过判断跳过哪些在从库B上已经执行的事务。

一种取同步位点的方法：

1. 等待新主库 A' 把中转日志（relay log）全部同步完成；
2. 在 A' 上执行 show master status 命令，得到当前 A' 上最新的 File 和 Position；
3. 取原主库 A 故障的时刻 T；
4. 用 mysqlbinlog 工具解析 A' 的 file, 得到 T时刻的位点。

    mysqlbinlog File --stop-datetime=T --start-datetime=T

![](https://img-blog.csdnimg.cn/2019041922595591.png)

end_log_pos 后面的值“123”可以把它作为 $master_log_pos。当然这个值是不精确的，因为如果主库在刚传完insert语句的binlog时掉电，那么从库 B和备库 A' 都已执行过该insert语句，在从库执行 change master 命令时，就会把insert 的binlog同步到从库B执行，这时就会出错（主键重复），然后停止同步。

通常情况下，在切换任务的时候，要先主动跳过这些错误，有两种常用的方法：

1、主动跳过一个事务。每次碰到错误时，就执行一次跳过命令。


    set global sql_slave_skip_counter=1;
    start slave;

2、通过设置 slave_skip_errors 参数，直接设置跳过指定的错误。这种设置是在主备切换时，当切换完成，稳定执行一定时间后，还需把这个参数设置为空。

例如设置 slave_skip_errors 为 “1032,1062”。


### GTID

通过上述方式来实现主备切换，操作都很复杂，而且容易出错。MySQL5.6版本引入了GTID，彻底解决了这个困难。

Global Transaction identifier，也就是全局事务 ID，是一个事务在提交时生成的。格式： GTID=server_uuid:gno，server_uuid 是一个实例第一次启动时生成的，是一个全局唯一的值；gno 是一个整数，初始值为1，每次提交事务的时候分配给这个事务，并加1。

官方文档给的定义是： GTID=source_id:transaction_id

在启动MySQL实例时，通过加上参数 gtid_mode=on 和 enforce_gtid_consistency=on 就可以启动GTID模式了。在GTID模式下，每一个事务都与一个GTID一一对应。GTID的两种生成方式，由session变量gtid_next的值决定：

1. 如果 gtid_next=automatic，代表使用默认值。这时，MySQL就会把 server_uuid:gno 分配给这个事务
	1. 记录binlog的时候，先记录一行 SET @@SESSION.GTID_NEXT='server_uuid:gno';
	2. 把这个GTID加入到本实例的 GTID 集合。
2. 如果 gtid_next 是一个指定的 GTID 值，那么就有两种可能：
	1. 如果指定值已经在 GTID 集合中，接下来执行的这个事务直接被系统忽略，不执行该事务；
	2. 如果指定值没有在 GTID 集合中，就将该值分配给这个事务。系统不需要给这个事务生成新的 GTID， 因此gno 也不用加1。

这样，每个 MySQL 实例都维护了一个GTID 集合，用来对应“这个实例执行过的所有事务”。


### 基于 GTID 的主备切换

在 GTID 模式下，备库 B 要设置为新主库 A' 的从库的语法如下：

    
    CHANGE MASTER TO
    MASTER_HOST=$host_name
    MASTER_PORT=$port
    MASTER_USER=$user_name
    MASTER_PASSWORD=$password
    master_auto_position=1

master_auto_position=1 表示这个主备关系使用的是 GTID 协议。

在主备切换时，从库会把自己的 GTID 集合发送个 新主库 A' ，A' 会用自己的 GTID 集合与从库比较，算出差集。如果 A' 不包含差集中的所需要的 binlog 事务，那么直接返回错误；如果包含，则找出差集只中的第一个事务发给从库，之后就从这个事务开始，往后读文件，顺序取 binlog 发送给从库。


## 28 | 读写分离有哪些坑？

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019063019181575.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

读写分离的主要目标就是分摊主库的压力。图1 中的结构是客户端（client）主动做负载均衡，这种模式下一般会把数据库的连接信息放在客户端的连接层。由客户端来选择数据库进行查询。

还有一种架构是，在MySQL和客户端之间有一个中间代理层 proxy，客户端只连接 Proxy，由Proxy根据请求类型和上下文决定请求的分发路由。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190630191851624.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


两种架构的特点：

1. 客户端直连，因为少了一层Proxy转发，索引查询性能稍微好一点，并且整体架构简单，排查问题更方便。但是这种方案需要了解后端部署细节，所以在出现主备切换、库迁移等操作的时候，客户端会感知到，并且要调整数据库连接信息。为了更方便的管理，一般会使用管理后端的组件，例如zookeeper。
2. 待Proxy 的架构，对客户端比较友好。客户端不需要关注后端细节，连接维护、后端信息维护等工作，都是由Proxy完成。使用这种架构会增加系统的复杂度，因此对团队的要求会更高。

无论使用哪种架构，都会碰到由于主从可能存在延迟，客户端执行完一个更新事务后马上发起查询，如果查询选择的是从库的话，就有可能读到刚刚的事务更新之前的状态。

这种“在从库上会读到系统的一个过期状态”的现象，我们暂且称之为“过期读”。

处理方案：

- 强制走主库方案；
- sleep 方案；
- 判断主备无延迟方案；
- 配合 semi-sync 方案；
- 等主库位点方案；
- 等 GTID 方案。

### 强制走主库方案

强制走主库方案就是，将查询请求做分类：

1. 对于必须要拿到最新结果的请求，强制将其发到主库上。
2. 对于可以读到旧数据的请求，才将其发到从库上。

### Sleep 方案

主库更新后，读从库之前先 sleep 一下。

这种方式看起来不靠谱，存在的问题：

1. 如果这个查询请求本来 0.5 秒就可以在从库上拿到正确结果，也会等 n 秒。
2. 如果延迟超过 n 秒，还是会出现过期读。


### 判断主备无延迟方案

在确保主备无延迟的情况下，才进行从库请求。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190630192218116.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

**第一种**确保主备无延迟的方法是，每次从库执行查询请求前，先判断 seconds_behind_master 是否已经等于0（表示主备无延迟）。直到参数等于 0  时才能执行查询请求。

**第二种**方法，对比位点确保主备无延迟：

- Master_Log_File 和 Read_Master_Log_Pos，表示的是读到的主库的最新位点；
- Relay_Masert_Log_File 和 Exec_Master_Log_Pos，表示的是备库执行的最新位点。


如果两组值对应相等，就表示接收到的日志已经同步完成。

第三种方法，对比 GTID 集合确保主备无延迟：

- Auto_Position=1，表示这对主备关系使用了 GTID 协议。
- Retrieved_Gtid_Set，是备库收到的所有日志的 GTID 集合；
- Executed_Gtid_set，是备库所有已执行完成的 GTID 集合。


当两个集合相同时，表示备库接收到的日志都已同步完成。

可见对比位点和对比 GTID 方法，都要比判断 second_behind_master 是否为 0 更准确。

但是还没有达到精确的程度，因为发往备库的 binlog 是始终少于等于主库 binlog 内容的，即使从库 binlog 都执行完成，与主库的状态还是可能不同步的，就会出现过期读。


### 配合 semi-sync

半同步复制：semi-sync replication

1. 事务提交的时候，主库把 binlog 发给从库；
2. 从库收到 binlog 后，发回给主库一个 ack，表示收到了；
3. 主库收到这个 ack 后，才能给客户端返回 “事务完成”的确认。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190630192438923.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


semi-sycn 配合判断主备无延迟的方案，存在两个问题：

1. 一主多从的时候，在某些从库执行查询请求会存在过期读的现象；
2. 在持续延迟的情况下，可能出现过度等待的问题。

### 等主库位点方案

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190630192527163.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

执行逻辑：


1. trx1 事务更新完成后，马上执行 show master status 得到当前主库执行到的 File 和Position；
2. 选定一个从库执行查询语句；
3. 在从库上执行 select master_pos_wait(File, Position, 1);
4. 如果返回值是 >= 0 的正整数，则在这个从库执行查询语句；
5. 否则，到主库执行查询语句。

    select master_pos_wait(file, pos[, timeout]);

表示从命令开始执行，到应用完 file 和 pos 表示的 binlog 位置，执行了多少事务。

对于不予许过期读的要求，只有两个选择，一种是超时放弃，一种是转到主库查询。需要注意的是考虑到主库的压力，做好限流策略。


### GTID 方案

    select wait_for_executed_gtid_set(gtid_set, 1);

命令的逻辑是：

1. 等待，直到这个库执行的事务中包含传入的 gtid_set， 返回0；
2. 超时返回 1。

等 GTID 的执行逻辑：

1. trx1 事务更新完成后，从返回包直接获取这个事务的 GTID， 记为 gtid1；
2. 选定一个从库执行查询语句；
3. 在从库上执行  select wait_for_executed_gtid_set(gtid_set, 1);
4. 如果返回值是 0，则在这个从库执行查询语句；
5. 否则，到主库执行查询语句。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190630192737299.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


通过设置参数 session_track_gtids = OWN_GTID ，让 MySQL 在执行事务后，返回包中带上 GTID。然后通过 API 接口 mysql_session_track_get_first 从返回包中解析 GTID 的值。


### 小结


## 29 | 如何判断一个数据库是不是出问题了？

### select 1 判断

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190706174650291.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

select1成功返回，只能说明这个库的进程还在，并不能说明主库没有问题。

innodb_thread_concurrency 参数设置并发线程上限，一旦并发线程数达到这个值， InnoDB 在接收到新请求的时候，就会进入等待状态，直到有线程退出。默认值为0，表示不限制并发线程数量。如果不限制，CPU核数有限，线程上下文切换成本高。一般建议设置为 64-128 之间。

并发连接和并发查询是不同的概念。show processlist 的结果，指的是并发连接，达到几千个影响不大，多占些内存。“当前正在执行”的语句，才是并发查询，其占用CPU资源。

线程在进入锁等待以后，并发线程的计数会减一，也就是等行锁（也包括间隙锁）的线程并不在 innodb_thread_concurrency 中。原因是，进入锁等待的线程并不吃CPU了，再有就是如果计数不减一，线程等待数到达上限值，整个系统就会堵住了。

如果线程真正地在执行查询，比如上述的 select sleep(100) from t，还是要算进并发线程的计数的。


### 查表判断

可以通过在MySQL里面创建一张表，然后定期执行查询语句。这样就可以检测出由于并发线程过多导致数据库不可用的情况。

但是，如果遇到磁盘空间满的时候，这种方法就变得不可用。更新事务时要写 binlog，而一旦 binlog 所在磁盘的空间占用率达到 100%， 那么所有的更新语句和事务提交的 commit 语句就都会被堵住。但是可以进行正常的读数据。


### 更新判断

常见的做法是放一个 timestamp 字段，用来表示最后一次执行检测的时间。

节点可用性检测应该包含主库和备库。我们一般会把数据库 A 和 B 的主备关系设计为双 M 结构，所以在备库 B上执行的命令也要发回给主库 A。如果主库 A 和 备库 B 都用相同的更新命令，就可能出现行冲突，也就是可能会导致主备同步停止。所以更新的表就要有多行数据，id 对应每个数据库的 server_id 做主键。

    mysql> CREATE TABLE `health_check` (
      `id` int(11) NOT NULL,
      `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB;
    
    /* 检测命令 */
    insert into mysql.health_check(id, t_modified) values (@@server_id, now()) on duplicate key update t_modified=now();

MySQL 规定了主库和备库的 server_id 必须不同，这样就可以保证检测命令不会发生冲突。

如果检测命令的id一样，两个语句同时执行，那么在主库和备库上就都是“insert行为”，写到binlog里面就都是Write rows event，这个冲突就会导致主备同步停止。

但是依然存在一些问题，当 IO 利用率 100%时，表示系统的 IO 是在工作的，每个请求都有机会获得 IO 资源，执行任务。检测执行语句需要的资源很少，所以可能在拿到 IO 资源的时候就可以提交成功，并且在未到达超时时间时就返回给检测系统，那么就会得出“系统正常”的结论。但其实业务系统上正常的 SQL语句已经执行的很慢了。

上述所说的所有方法，都是基于外部检测的。存在两个问题：一是检测方式并不能真正完全的反应系统当前的状况，二是定时轮询检测有时间间隔，如不能及时发现，可能导致切换慢的问题。


### 内部统计

可以通过MySQL 统计的 IO 请求时间，来判断数据库是否出问题。

可以通过下面语句，查看各类 IO 操作统计值：


    select * from performance_schema.file_summary_by_event_name where event_name = 'wait/io/file/innodb/innodb_log_file';


![在这里插入图片描述](https://img-blog.csdnimg.cn/20190706174901257.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


从上到下分别统计的是：所有 IO 类型、读操作、写操作、其他类型数据。

performance_schema 统计是有性能损耗的，因此只需打开需要的监控即可。例如 redo log：


    mysql> update setup_instruments set ENABLED='YES', Timed='YES' where name like '%wait/io/file/innodb/innodb_log_file%';

检测语句，200 ms 为异常：

    mysql> select event_name,MAX_TIMER_WAIT  FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;

清空语句：

    mysql> truncate table performance_schema.file_summary_by_event_name;


### 小结


## 30 | 答疑文章（二）：

用动态的观点看加锁加锁规则：两个“原则”，两个“优化”，一个bug。

- 原则1：加锁的基本单位是 next-key lock。它是前开后闭区间。
- 原则2：查找过程中访问到的对象会加锁。
- 优化1：索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。
- 优化2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。
- 一个bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。


### 不等号条件里的等值查询

在执行过程中，通过树搜索的方式定位记录的时候，用的是“等值查询”的方法。


### 等值查询的过程


### 怎么看死锁

出现死锁后，执行 show engine innodb status 命令输出相关信息。LATESTDETECTED DEADLOCK 记录的是最后一次死锁信息。

1、由于锁是一个一个加的，要避免死锁，对同一组资源，要按照尽量相同的顺序访问；

2、在死锁发生时刻，资源占用越多，回滚成本就越大。因此 InnoDB 会选择回滚成本更小的语句。


### 怎么看锁等待

update 语句先插入再删除。


## 31 | 误删数据后除了跑路，还能怎么办？

误删数据分类：
1、使用 delete 语句误删数据行；
2、使用 drop table 或者 truncate table 语句误删数据表；
3、使用 drop database 语句误删数据库；
4、使用 rm 命令误删整个 MySQL 实例。


### 误删行

在前提是 binlog_format=row 和 binglog_row_image=NULL 的前提下，如果使用 delete 语句误删了数据行，可以用 Flashback 工具通过闪回把数据恢复回来。

具体恢复数据时，对单个事务做如下处理：

1. 对于 insert 语句，对应的 binlog event 类型是 Write_rows event，把它改成 Delete_rows event 即可；
2. 对于 delete 语句，将 Delete_rows event 改为 Write_rows event；
3. 对于Update_rows，binlog 里面记录了修改前和修改后的值，对调这两行的位置即可。

对于误删数据涉及到多个事务的话，需要将事务的顺序调过来再执行。

恢复数据比较安全的做法是，恢复出一个备份，或者找一个从库作为临时库，在这个临时库上执行这些操作，然后再将确认过的临时数据库的数据，恢复回主库。

预防误删数据方法：

1. 把 sql_safe_updates 参数设置为 on。这样的话，如果忘记在 delete 或者 update 语句中写 where 条件，或者 where 条件里面没有包含索引字段的话，这条语句执行就会报错。
2. 代码上线前，必须经过 SQL 审计。


### 误删库 / 表

误删了库或表，就要求线上有定期的全量备份，并且实时备份 binlog。

恢复数据的流程：

1. 取最近一次全量备份，假设这个库是一天一备，上次备份时当天 0 点；
2. 用备份恢复一个临时库；
3. 从日志备份里面，取出凌晨 0 点之后的日志；
4. 把这些日志，除了误删除数据的语句外，全部应用到临时库。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190714193602266.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


1、为了加速数据恢复，在使用 mysqlbillog 命令时，加 database 参数来指定误删表所在的库。

2、在应用日志的时候，需要跳过 12 点误操作的那个语句的 binlog：

- 如果没有使用 GTID 模式，只能在应用到 12 点的 binlog 文件的时候，先用 -stop-position 参数执行到误操作之前的日志，再用 -start-position 从误操作之后的日志继续执行。
- 如果使用了 GTID 模式，只需要把误操作的 GTID 加到临时库实例的 GTID 集合就行了。


使用 mysqlbinlog 方法恢复数据不够快的原因：

1. mysqlbinlog 并不能指定只解析一个表的日志；
2. 用 mysqlbinlog 解析出日志应用，应用日志的过程只能是单线程。


一种加速的方法是，将临时实例设置成线上备库的从库：

1. 在 start slave 之前，先通过执行 change replication filter replicate_do_table  = (tbl_name) 命令，就可以让临时库只同步误操作的表；
2. 这样也可以用上并行复制技术，加速数据恢复过程。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190714193815206.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


把之前删掉的 binlog 放回备库的操作步骤：

1. 从备份系统下载 master.000005 和 master.000006 这两个文件，放到备库的日志目录下；
2. 打开日志目录下的 master.index 文件，在文件开头加入两行，内容分别是 “./master.000005" 和 “./master.000006”;
3. 重启备库，目的是让备库重新识别这两个日志文件；
4. 现在备库上就有了临时库需要的所有 binlog 了，建立主备干洗，就可以正常同步了。


### 延迟复制备库

如果距离上一个全量备份时间较长，那么恢复时间也会很长。对于非常核心的业务，是不允许太长的恢复时间。可以考虑搭建延迟复制的备库。

延迟复制的备库是一种特殊的备库，通过 CHANGE MASTER TO MASTER_DELAY = N 命令，可以指定这个备库持续保持跟主库 N 秒的延迟。这样就可以在备库上先 stop slave，再通过之前的方法跳过误操作命令，就可以恢复出需要的数据。


### 预防误删库 / 表的方法

1、账号分离，不同的人有不同的权限，避免写错命令。

2、制定操作规范。避免写错要删除的表名。比如：在删除表之前先对表做改名操作。观察一段时间如果对业务没有影响，再通过管理系统删除有固定后缀的表。


### rm 删除数据

尽量把备份跨机房，或者最好是跨城市保存。


### 小结




## 32 | 为什么还有kill不掉的语句？

MySQL中有两个 kill 命令：一个是 kill query + 线程 id，表示终止这个线程中正在执行的语句；一个是 kill connection（可以缺省） + 线程 id，表示断开这个线程的连接，如果这个线程有语句正在执行，也是要先停止正在执行的语句。


### 收到 kill 以后，线程做什么？

kill 并不是马上停止的意思，而是告诉执行线程说，这条语句已经不需要继续执行了，可以开始“执行停止的逻辑了”。

当用户执行 kill query thread_id_B 时，MySQL 里处理 kill 命令的线程做了两件事：

1. 把 sessionB 的运行状态改成 THD::KILL_QUERY（将变量 killed 赋值为 THD::KILL_QUERY）；
2. 给 sessionB 的执行线程发一个信息。让其退出等待，来处理 THD::KILL_QUERY 状态。


1、一个语句执行过程中有多处 “埋点”，在这些 “埋点”的地方判断线程状态，如果发现线程状态是 THD::KILL_QUERY， 才开始进入语句终止逻辑；

2、如果处于等待状态，必须是一个可以被唤醒的等待，否则根本不会执行到 “埋点”处；

3、语句从开始进入终止逻辑，到终止逻辑完全完成，是有一个过程的。

先执行 set global innodb_thread_concurrency=2，将 InnoDB 的并发线程上限数设置为2，然后执行

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019071419402572.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


可以看到 session D 执行的 kill query C 命令没有什么效果，session E 执行了 kill connection 命令，才断开了 session C 的连接。但是这时，如果在 session E 中执行 show processlist，就会看到：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190714194103668.png)


id=12 这个线程的 Command 列显示的是 Killed。客户端虽然断开了连接，但实际上服务端上这条语句还在执行过程中。

使用 kill query 命令之所以无效，是因为 12 号线程的等待逻辑是：每 10 毫秒判断一下是否可以进入 InnoDB 执行，如果不行，就调用 nanosleep 函数进入 sleep 状态。在这个过程中没有去判断线程的状态，因此根本不会进入终止逻辑阶段。因为等行锁进入等待状态是可以被唤醒的。

当执行 kill connection 命令时，过程是这样：

1. 把 12 号线程状态设置为 KILL_CONNECTION;
2. 关掉 12 号线程的网络连接。

Command 列显示为 killed 是因为执行 show processlist 时有这样一个逻辑：如果一个线程的状态是 KILL_CONNECTION，就把 Command 列显示成 Killed。只有等到满足进入 InnoDB 的条件后，session C 的查询语句继续执行，然后才有可能判断线程的状态已经变成 KILL_QUERY 或者 KILL_CONNECTION ，再进入终止逻辑阶段。

造成 kill 无效的情况：

- 线程没有执行到判断线程状态的逻辑。上述是一种情况，还有就是由于 IO 压力过大，读写 IO 的函数一直无法返回，导致不能及时判断线程的状态。
- 终止逻辑耗时较长。就会出现 Command=killed 的情况，需要等到终止逻辑完成，语句才算真正完成。常见的场景是，超大事务执行期间被 kill、大查询回滚、DDL 命令执行到最后阶段被kill，都有可能由于需要等到 IO 资源或其他耗时操作到时终止逻辑耗时长。

MySQL 是停等协议，线程执行的语句还没有返回的时候，再往这个连接里面继续发命令也是没用的。


### 另外两个关于客户端的误解

**第一个误解**：如果库里面的表特别多，连接就会很慢。

我们感知到的连接过程慢，其实并不是连接慢，也不是服务端慢，而是客户端慢。

当使用默认参数连接的时候，MySQL 客户端会提供一个本地库名和表名的补全功能。在客户端连接成功后，需要多做一些操作：

1. 执行 show databases;
2. 切到 db1 库，执行 show tables；
3. 把这两个命令的结果用于构建一个本地的哈希表。最花时间的是这一步。

连接命令中加上 -A ，就可以关掉这个自动补全的功能。

**第二个误解**：-quick 参数会让服务端变得更快。但实际上并不能让服务端加速，反而可能会降低服务端的性能，会让客户端变得更快。

MySQL客户端发送请求后，接收服务端返回结果的方式有两种：

1. 一种是本地缓存，也就是在本地开一片内存，先把结果存起来。
2. 另一种是不缓存，读一个处理一个。

MySQL客户处默认采用第一种方式，如果加上 -quick 参数，就会使用第二种不缓存的方式。

使用 -quick 参数，可以达到以下三点效果：

1. 跳过表名自动补全功能
2. mysql_store_result 需要申请本地内存来缓存查询结果，如果查询结果太大，会耗费较多的本地内存，可能会影响客户端本地机器的性能；
3. 不会把执行命令记录到本地的命令历史文件。



























----------


## 39 | 自增主键为什么不是连续的？

### 自增值保存在哪？

表的结构定义存放在后缀名为 .frm 的文件中，但是并不会保存自增值。

不同的引擎对于自增值的保存策略不同。

- MyISAM 引擎的自增值保存在数据文件中。
- InnoDB引擎的自增值，其实是保存在了内存里，并且到了8.0版本后，才有了“自增持久化”的能力，也就能实现数据库发生重启，表的自增值可以恢复为MySQL重启前的值。
	- 在MySQL 5.7 及之前的版本，自增值保存在内存里，并没有持久化，当MySQL重启后会读取自增值的最大值 max(id)，然后将 max(id) + 1 作为这个表当前的自增值。但是当再插入一条记录，如果不设置id的值，那么id的值为 新生成的自增值而不是当前的自增值。如果执行delete语句删除最大的id，然后MySQL重启是会改变自增值的。
	- 在 MySQL 8.0 版本，将自增值的变更记录在redo log 中，重启的时候依靠 redo log 恢复重启之前的值。delete语句并不会改变自增值。当前自增值为之前max(id) + 1，同样使用自增id 插入记录，id值也为 新生成的自增值，而不是当前的自增值。

### 自增值的修改

如果id被定义为 AUTO_INCREMENT，在插入一行数据时，自增值的行为如下：

1. 如果插入数据时 id 字段指定为 0、null 或未指定值，那么就把当前表的AUTO_INCREMENT值填到自增字段；
2. 如果插入数据时 id 字段指定了具体的值，就直接使用语句里指定的值。

插入完数据后，依据插入的值后当前自增值的大小关系，自增值的变更也会有所不同，假设 x 为插入的值，y 为当前自增值：

- 当 x<y， 那么这个表的自增值不变；
- 当 x>=y，那么自增值变为新生成的自增值；

新的**自增值生成算法**是：从 auto_increment_offset 开始，以 auto_increment_increment 为步长，持续叠加，直到找到第一个大于 x 的值，作为新的自增值。

auto_increment_offset表示自增的初始值，auto_increment_increment 表示步长，默认值都是1。在一些场景下，需要改变默认值，比如双M的主备结构里要求双写的时候，我们就可能会设置 auto_increment_increment=2，避免两个库生成的主键发生冲突。

### 自增值的修改时机

![](https://img-blog.csdnimg.cn/20190401003850298.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

由于自增值的改变是在执行插入语句之前，而且当发现唯一键冲突的时候，自增值并没有改回去，就导致了自增主键id不连续的情况。
可见，唯一键冲突是导致自增主键 id 不连续的第一种原因。同样地，事物回滚也会产生类似的现象，这是第二种原因。
自增值不回退的原因是为了提升性能。如果允许自增值回退，那么将会出现主键冲突的情况，比如先申请到id的事务执行失败，后申请到的事务执行成功，那么就会出现自增值小于当前最大id，再后面的插入语句就会出现“主键冲突”。要解决这个问题每次申请id之前，要先判断表里 id 是否存在。或者是自增 id  的锁范围扩大，必须等到一个事务执行完成并提交，下一个事务才可以申请自增id。这两种方法都会导致性能下降。因此，InnoDB放弃了这个设计，语句执行失败也不回退自增 id。这样保证了自增 id 是递增的，但不保证是连续的。

### 自增锁的优化

自增 id 锁并不是一个事务锁，而是每次申请完就马上释放，以便允许别的事务再申请。

在Mysql5.0版本的时候，自增锁的范围是语句级别的。也就是说，如果一个语句申请了一个表自增锁，这个锁会等语句执行结束以后才释放。显然这样设计会影响并发度。

mysql5.1.22版本新增参数 innodb_autoinc_lock_mode ，默认值是1。

1. 这个参数值为0时，表示采用5.0版本策略。
2. 当值为1时：
	1. 普通insert语句，自增锁在申请之后就马上释放；
	2. 类似 insert… select 这样的批量插入语句，自增锁还是要等语句结束后才被释放；
3. 参数的值为2时，所有的申请自增主键的动作都是申请后就释放锁。

为了数据一致性才会在 insert… select 语句执行时值设置成1，等语句结束后才释放锁。

因为在一个事务执行 insert… select 语句过程中，另一个事务向同一个表执行插入语句，如果这时 binlog日志的格式是statement，那么将会先记录一个事务的插入，再记录另一个事务的插入。当binlog在备库执行时，insert… select 语句将会连续执行完成后，再执行另一个插入语句，这样跟主库的数据就会不一致。

要解决数据不一致问题又两种思路：

1. 批量插入语句固定生成连续的 id 值。所以，自增锁直到语句执行结束才释放，就是为了这个目的。
2. 在binlog中记录所有插入数据，到备库中执行时，不再依赖自增主键。通过设置 innodb_autoinc_lock_mode=2 ，同时binlog_format 设置成row.

在生产环境上，尤其是有 insert… select 这种批量插入数据的场景时，从并发的角度考虑，建议设置 innodb_autoinc_lock_mode=2 ，同时binlog_format=row。

这里说的批量插入数据，包含的语句是 insert…select、replace … select 和 load data 语句。普通的inset语句包含多个value值的情况下，申请完id后会立即释放锁，因为这类语句是可以计算出需要多少个 id 的。前面几种语句是不知道需要多少 id 的。

mysql 有一个批量申请自增 id 的策略：

1. 语句执行过程中第一次申请自增 id，会分配 1 个；
2. 第二次申请会分配2个；
3. 第三次申请会分配4个；
4. 以此类推，同一个语句去申请自增id，每次申请到的自增id个数都是上一次的两倍。

如果申请到的主键id没用完，比如第三次分配了4个，但第三次只插入了2条记录，那么其他语句申请的id 将会是第4个id 后的新自增id。这就是主键id出现自增 id 不连续的第三种原因。























































