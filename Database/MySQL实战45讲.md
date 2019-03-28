






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
















