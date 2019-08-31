## 1、基础架构：一条SQL查询语句是如何执行的？

SQL语句在MySQL各个功能模块中的执行过程：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190831154353799.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

大体来说，MySQL 可以分为 Server 层和存储引擎层两部分

- Server层包括：连接器、查询缓存、分析器、优化器、执行器等。
- 存储引擎负责数据的存储和提取。常用的存储引擎InnoDB、MyISAM

查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。建议大多数情况下不要使用查询缓存。MySQL8.0版本把查询缓存功能已去除。


## 2、日志系统：一条SQL查询语句是如何执行的？

与查询流程不一样的是，更新流程还涉及两个重要的日志模块：redo log（重做日志）和binlog（归档日志）。

### 日志模块：redo log

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190831154516315.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

在做更新操作时，如果每一次更新操作都立即写进磁盘，那么整个过程IO成本、查找成本都很高。redo log就是为了解决这一问题，MySQL通过WAL（Write-Ahead Logging）技术，先把更新操作写入redo log日志，并更新内存，这时更新操作就完成了，同时InnoDB引擎会在适当的时候（系统比较空闲时），将一批操作更新到磁盘里面。如果redo log写满时，就不能再执行新的更新，得停下来先擦掉redo log一些记录，写入磁盘，再执行新的更新。

有了redo log，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为crash-safe


### 日志模块：binlog

redo log和binlog不同点：

1. redo log 是 InnoDB 引擎特有的；binlog是MySQL的Server层实现的，所有引擎都可以使用
2. redo log是物理日志，记录的是“在某个数据页上做了什么修改”；binlog是逻辑日志，记录的是这个语句的原始逻辑，比如“给ID=2这一行的c字段加1”。binlog两种模式：statement格式记录SQL语句，row格式记录行的内容，两条（更新前和更新后）。
3. redo log 是循环写的，空间固定会用完；binlog是可以追加写入的。“追加写”是指binlog文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

update语句执行流程：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190831154617226.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


将redo log的写入拆成了两个步骤：prepare 和 commit，这就是“两阶段提交”。

“两阶段提交”的目的是为了让两份日志之间的逻辑一致。保证一致性。


数据恢复过程：

- 首先，找到最近的一次全量备份，从这个备份恢复到临时库；
- 然后，从备份的时间点开始，将备份的binlog依次取出来，重放到中午误删表之前的那个时刻。
- 最后把表数据从临时库取出来，按需要恢复到线上库去。


## 3、事务隔离：为什么你改了我还看不见？

事物就是要保证一组数据库操作要么全部成功，要么全部失败。

事物支持是在引擎层实现的，但并不是所有引擎都支持事物，MyISAM引擎就不支持事物。

多事务同时执行的时候，可能会出现的问题：脏读、不可重复读、幻读。

SQL标准的事物隔离级别：

- 读未提交。是指一个事物还没提交时，它做的变更就能够被其他事物看到。
- 读提交。是指一个事物提交之后，它做的变更才会被其他事物看到。
- 可重复读。是指一个事物在执行过程中看到的数据，总是跟这个事物启动时看到的数据是一致的。
- 串行化。串行化，顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突时，后访问的事物必须等待前一个事物执行完成，才能继续执行。


MySQL事物启动方式：

- 显式启动事务语句， begin 或 start transaction。提交语句commit，回滚语句rollback。
- set autocommit=0，这个命令会将这个线程的自动提交关闭。


## 04 | 深入浅出索引（上）

索引的出现是为了提高数据库查询效率，就像书的目录一样。

常见的索引模型：

- 哈希表
- 有序数组
- 搜索树

哈希表这种结构适用于只有等值查询的场景，比如 Memcached及其他一些nosql引擎。由于哈希表存储不是有序的，因此做区间查询的速度是很慢的。

而有序数组在等值查询和范围查询场景中的性能就都非常优秀。有序数组只适用于静态存储引擎，由于它是有序存储，因此在插入数据时，要挪动后面的所有数据，成本太高。

为了让一个查询尽量少地读磁盘，就必须让查询过程访问尽量少的数据块。因此使用N叉数，而不是二叉树。


### InnoDB 的索引模型

在 InnoDB 中，表都是根据主键顺序以索引的形式存放的，这种存储方式的表称为索引组织表。InnoDB使用的是B+树索引模型。

每一个索引在 InnoDB 里面对应一棵 B+ 树。B+树能够很好地配合磁盘的读写特性，减少单词查询时访问磁盘的次数。

主键索引的叶子节点存的是整行数据。在 InnoDB 里，主键索引也被称为聚簇索引。

非主键索引的叶子节点内容是主键的值。在 InnoDB 里，非主键索引也被称为二级索引。

普通索引的查询方式是先搜索普通索引树，找到主键值，再去搜索主键索引树。这个过程也被称为回表。也就是说，基于非主键索引的查询需要多扫描一棵索引树。因此，我们在应用中应尽量使用主键查询。

数据页的分裂与合并：当数据页已满时，有新的数据插入就会申请新的数据页，并把部分数据移动过去，这个过程称为数据页的分裂，不仅性能会受影响，而且数据页的利用率也会降低。当相邻的两个数据页由于数据删除，导致利用率很低时，就会将数据页合并。

从性能和存储空间方面考量，自增主键往往是更合理的选择。



## 05 | 深入浅出索引（下）

有普通索引查询主键值，再回到主键索引树搜索的过程称为回表。那如何优化索引避免回表呢：

**覆盖索引**。指的是普通索引树上，节点已经包含了要查询的信息，也就是普通索引“覆盖了”我们的查询需求，因此就不用再使用主键查询，避免了回表。由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的优化手段。

**最左前缀原则**。B+ 树这种索引结构，可以利用索引的“最左前缀”，来定位记录。不只是索引的全部定义，只要满足最左前缀，就可以利用索引来加速检索。最左前缀可以是联合索引的最左N个字段，也可以是字符串索引的最左M个字符。

联合索引如何安排字段顺序：

- 如果可以通过改变字段顺序，从而少维护一个索引，那么将是优先考虑的
- 第二个考虑原则是空间

### 索引下推

而 MySQL 5.6 引入的索引下推优化，在索引遍历过程中，对索引包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。


## 06 | 全局锁和表锁 ：给表加个字段怎么有这么多阻碍？

数据库锁设计的初衷是处理并发问题。根据加锁的范围，MySQL 里面的锁大致可以分成全局锁、表级锁和行锁三类。

### 全局锁

全局锁就是对整个数据库实例进行加锁。要想使数据库处于只读状态，加全局读锁，使用命令：Flush table with read lock（FTWRL）。使用全局读锁后，其他线程的数据更新语句、数据定义语句和更新类事物的提交语句都将被阻塞。

全局锁的典型使用场景是，做全库逻辑备份。也就是把整库每个表都select出来存成文本。

MyISAM不支持可重复读的事物隔离级别，因数据库备份时要使用FTWRL（只读全局锁）来进行备份，期间只能进行读操作。因此建议使用InnoDB引擎。InnoDB使用mysqldump备份时，建议使用参数--single-transaction

设置全库只读：Flush table with read lock（FTWRL）、   set global readonly=true。建议使用FTWRL，原因是：

- 有些系统中readonly值会用来做逻辑判断，比如判断是主库还是从库。因此，修改global变量的方式影响面更大，不建议使用。
- 异常处理机制有差异。执行FTWRL命令后，如果由于客户端发生异常断开，那么MySQL会自动释放这个全局锁，整个库回到正常更新状态。而设置readonly后，如果客户端发生异常，MySQL会一直保持readonly状态，数据库长时间处于不可写状态，风险较高。


### 表级锁

表锁一般是引擎不支持行锁是才被用到。

MySQL表级锁有两种：表锁、元数据锁（meta data lock， MDL）

表锁语法：lock tables ... read/write。  unlock tables 释放锁。

MDL: 不需要显示使用，在访问一个表时会被自动加上。MDL的作用是，保证读写的正确性。当对一个表做增删改查、结构变更操作的时候，加MDL锁。

- 读锁之间不互斥，多个线程可以同时对一张表增删改查
- 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，两个线程要同时给一个表加字段，其中一个要等另一个执行完才能执行。


### 如何安全的给小表加字段：

如果一个表有频繁的查询语句，而且客户端有重试机制，这时改变表结构可能会导致库的线程爆满，从而整个库挂掉。当有多个查询在执行时，语句结束后并不会马上释放MDL读锁，而是等到整个事物提交后释放，这时更改表结构会被阻塞，它被阻塞后，之后的所有操作都会被阻塞，整个表就不可读写了。

如何安全加字段：

- 解决长事物。如果要做DDL变更的表有长事物在执行，要考虑暂停DDL，或者kill掉这个长事物。如果这个表是热点表，有频繁的请求，那么kill未必管用，因为新的请求马上就来了。可以通过在alter table语句里面设定等待时间，如果等待时间内能拿到MDL写锁就执行，拿不到就放弃，不用阻塞后面的业务语句，之后再重试这个过程。

MariaDB 已经合并了AliSQL的这个功能。


## 07| 行锁功过：怎么减少行锁对性能的影响？

行锁是针对数据库表中行记录的锁。存储引擎InnoDB支持行锁，MyISAM不支持行锁。

**两阶段锁协议**：在InnoDB事物中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。

如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。原因是这个锁的时间最少，减少了事务之间的锁等待，提升了并发度。


### 死锁和死锁检测

当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。

应对死锁有两种策略：

- 一种策略是，直接进入等待，直到超时。超时时间通过参数innodb_lock_wait_timeout来设置。默认为50s
- 另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事物得以继续执行。开启这个逻辑，通过设置参数innodb_deadlock_detect 为on来实现。默认为on。


如果死锁检测过多，就会消耗大量的CPU资源。比如1000个并发线程同时更新一行记录，那么死锁检测就是100万量级。

如何解决：

- 如果能够保证这个业务不会出现死锁，可以临时把死锁检测关掉。不推荐，关掉死锁检测意味着可能出现大量的超时，这是业务有损的。
- 另一种是控制并发度。客户端无法控制，要在服务端控制。中间件实现或修改MySQL源码，基本思路是对于相同行的更新，在引入引擎之前排队。
- 把一行变为多行，减少锁等待个数。这需要业务上的支持和设计


## 08 | 事务到底是隔离的还是不隔离的？

begin/start transaction 命令并不是事物的的起点，在执行的它们之后的第一个操作InnoDB表的语句，事物才真正启动。要想立即启动事物可以使用start transaction with consistent snapshot这个命令。

**视图：**

- 一个是view。它是一个用查询语句定义的虚拟表，在调用的时候执行查询语句并生成结果
- 一个是InnoDB在实现MVCC时用到的一致性读视图，即consistent read view。用于支持RC（Read Committed读提交）和RR（Repeatable Read可重复读）隔离级别的实现


在可重复读隔离级别下,事务在启动的时候就“拍了个快照”。注意,这个快照是基于整库的。
InnoDB 利用了“所有数据都有多个版本”的这个特性,实现了“秒级创建快照”的能力。

对于一个事务视图来说,除了自己的更新总是可见以外,有三种情况: 

1. 版本未提交,不可见
2. 版本已提交，但是在创建视图数组之后提交的，不可见
3. 在视图数组创建之前提交的，可见

更新数据都是先读后写的,而这个读,只能读当前的值,称为“当前读”(current read)。
除了 update 语句外,select 语句如果加锁,也是当前读。

可重复读的核心就是一致性读(consistent read);而事务更新数据的时候,只能用当前读。如果当前的记录的行锁被其他事务占用的话,就需要进入锁等待。

读提交的逻辑和可重复读的逻辑类似,它们最主要的区别是:

- 在可重复读隔离级别下,只需要在事务开始的时候创建一致性视图,之后事务里的其他查询都共用这个一致性视图；
- 在读提交隔离级别下,每一个语句执行前都会重新算出一个新的视图。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190831155922492.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)



## 09 | 普通索引和唯一索引，应该怎么选择？

**change buffer：**
    
当需要更新一个数据页时,如果数据页在内存中就直接更新,而如果这个数据页还没有在内存中的话,在不影响数据一致性的前提下,InooDB 会将这些更新操作缓存在 change buffer中，这样就不需要从磁盘中读入这个数据页了。在下次查询需要访问这个数据页的时候,将数据页读入内存中，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。

唯一索引要确定其唯一性，因此必须要将数据页读取到内存中，判断其是否唯一。所以唯一索引用不到change buffer，普通索引可以用到。

change buffer 使用的是buffer pool里的内存，change buffer实际上是可以持久化的数据，将 change buffer 中的操作应用到原数据页,得到最新结果的过程称为 merge。系统后台会定期merge，数据库正常关闭也会merge。

### change buffer 的使用场景

对于读多写少的业务来说，页面在写完之后马上读的概率很小，因此很多更新操作会缓存之change buffer 中，这样的话使用效果会很好。但是业务是写入之后会马上读取的话，会触发merge，这样随机访问IO的次数不会减少，同时又增加了change buffer的维护成本，这样反而起到了副作用。

### 索引选择和实践

普通索引和唯一索引应该怎么选择,其实,这两类索引在查询能力上是没差别的,主要考虑的是对更新性能的影响。所以,我建议你尽量选择普通索引。

### change buffer 和 redo log

redo log 主要节省的是随机写磁盘的 IO 消耗(转成顺序写),而 change buffer 主要节省的则是随机读磁盘的 IO 消耗。


## 10 | MySQL为什么有时候会选错索引？

使用哪个索引是由 MySQL 来确定的，确切的说是优化器的工作。


### 优化器的逻辑

查询扫描行数是优化器重要的依据条件之一。

一个索引上不同的值越多，这个索引的区分度就越好。索引上不同值的个数也称为基数。基数是通过采样统计来得出的值，采样统计时，InnoDB默认会选择N个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。

analyze table t 命令，可以用来重新统计索引信息。在实践中，如果发现explain的结果预估的rows值跟实际情况差距比较大，可以使用这个方法处理。


### 索引选择异常和处理

大多数时候优化器都能找到正确的索引，但偶尔会选错索引。这是有以下处理方法：

1. 采用force index强行选择一个索引。缺点：变更的及时性。线上系统修改SQL语句还要测试、发布等，不够敏捷。
2. 修改语句，引导MySQL使用我们期望的索引。例如：在逻辑结果一致时可以把：order by b limit 1 改为 order by b,a limit 1（注意文章的上下文）。缺点：根据数据特征诱导了一下优化器，不具备通用性。
3. 在有些场景下，我们可以新建一个更合适的索引，来提供给优化器做选择，或删掉误用的索引。缺点是找到更合适的索引比较难。如果误用的索引没必要存在，可以删除。


## 11 | 怎么给字符串字段加索引？

MySQL支持前缀索引，你可以定义字符串的一部分作为索引。默认地，创建时不指定长度索引就会包含整个字符串。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190831153158870.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190831153242687.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


前缀索引只取字符串前几位，比整个字符串索引占用空间更小。但可能会增加额外的记录扫描次数，因为依据前缀查询后，要去主键索引查找判断是否正确，这时有可能前缀一样后面的字符串不一致，就需要再去字符串索引查找，这就增加了记录扫描次数（回主键查找次数）。

使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。

如何确定前缀的长度：
1、统计出这个列上有多少个不同的值：

    mysql> select count(distinct email) as L from SUser;

2、依次取不同长度的前缀来统计不同值，看哪个值不小于 L * 95%（5%接受区分度损失比例）：

    mysql> select
      count(distinct left(email,4)）as L4,
      count(distinct left(email,5)）as L5,
      count(distinct left(email,6)）as L6,
      count(distinct left(email,7)）as L7,
    from SUser;


### 前缀索引对覆盖索引的影响

使用前缀索引就用不上覆盖索引对查询性能的优化了。例如在不使用前缀索引情况下，覆盖索引含有要查询的信息，就不用回表查询ID索引了。使用了前缀索引，还需查询ID索引确认是否是要查找的记录。


### 其他方式

如果遇到前缀的区分度不够好的情况时，该怎么办：

1. 第一种方式是使用倒序存储。有时字符串倒叙会有很好的区分度。

    mysql> select field_list from t where id_card = reverse('input_id_card_string');

2. 第二种方式是使用hash字段。你可以在表上再创建一个整数字段，来保存身份证的校验码，同时在这个字段上创建索引。

    mysql> alter table t add id_card_crc int unsigned, add index(id_card_crc);


    mysql> select field_list from t where id_card_crc=crc32('input_id_card_string') and id_card='input_id_card_string'


### 异同点

相同点：都不支持范围查询。倒叙存储是按照倒叙字符串排序的，hash字段只能支持等值查询。

区别：

1. 从占用额外空间来看，倒叙存储方式在主键索引上，不会消耗额外的存储空间，而hash需要增加一个字段。当然倒叙存储方式使用4个字节的前缀长度应该是不够的，再长一点，消耗和hash也差不多。
2. 在CPU消耗方面，倒叙需要调用reverse函数，hash需要调用crc32()函数。reverse函数的额外消耗CPU资源会更小些。
3. 从查询效率上看，使用hash字段方式的查询性能相对更稳定一些。hash每次查询平均扫描次数接近1。倒叙使用前缀索引，会增加回表查询次数。


### 总结：

1. 直接创建完整索引,这样可能比较占用空间;
2. 创建前缀索引,节省空间,但会增加查询扫描次数,并且不能使用覆盖索引;
3. 倒序存储,再创建前缀索引,用于绕过字符串本身前缀的区分度不够的问题；
4. 创建 hash 字段索引,查询性能稳定,有额外的存储和计算消耗,跟第三种方式一样,都不支持范围扫描。


## 12 | 为什么我的MySQL会“抖”一下？

### 你的 SQL 语句为什么变“慢”了

当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据数据页的内容一致，称为“干净页”。不论是脏页还是干净页，都在内存中。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190831153904760.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


因此不难想象，平时执行很快的更新操作，其实就是在写内存和日志，而MySQL偶尔“抖”一下的那个瞬间，可能就是在刷脏页。

**什么情况下回触发数据库的flush过程：**

1. 第一种情况是，InnoDB的redo log写满了。这时系统会停止所有更新操作，把checkpoint往前推进，为redo log留出空间继续写。checkpoint推进的这段范围上的脏页都flush到磁盘上后，redo log才可继续写。
2. 第二种情况是，系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是脏页，就要先将脏页写到磁盘。
3. 第三种情况是，MySQL认为系统“空闲”的时候。
4. 第四种情况是，MySQL正常关闭的情况。这时，MySQL会把内存的脏页都flush到磁盘上。


**上述四种场景对性能的影响：**

1. 第三种是空闲时候写，第四种是数据库关闭时候写，通常不会关注性能问题。
2. 第一种是redo log写满了，要flush脏页。这时系统不再接受更新了，所有的更新都会堵住。
3. 第二种是内存不够了，现将脏页写到磁盘，数据页淘汰机制（最久不使用）。InnoDB 用缓冲池(buffer pool)管理内存,缓冲池中的内存页有三种状态:
	1. 还没使用的
	2. 使用了是干净页
	3. 使用了是脏页
    如果淘汰的是脏页，要先flush到磁盘后，才能复用。

### InnoDB 刷脏页的控制策略

首先告诉InnoDB所在主机的IO能力，这样InnoDB才能知道需要全力刷脏页的时候，可以刷多快。通过innodb_io_capacity这个参数设置，该值建议设置成磁盘的IOPS（可以通过fio工具测试）。

InnoDB怎么控制引擎按照“全力”的百分比来刷脏页？

刷盘速度要参考两个因素：一个是脏页比例，一个是redo log写盘速度。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190831154138620.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


无论是你的查询语句在需要内存的时候可能要求淘汰一个脏页,还是由于刷脏页的逻辑会占用 IO 资源并可能影响到了你的更新语句,都可能是造成你从业务端感知到 MySQL“抖”了一下的原因。要避免这种情况，就要合理地设置innodb_io_capacity的值，并且平时要多关注脏页的比例，不要让它经常接近75%。

在刷脏页时，如果跟它相邻的数据页也还是脏页的话，也会被放到一起刷（该行为在机械硬盘时代很有意义，但在SSD时IOPS往往就不是瓶颈了）。innodb_flush_neighbors参数来控制这个行为。值为1时，会有上述的“连坐”机制；值为0时，表示不找邻居，只刷自己。MySQL8.0，该参数默认值为0。


## 13 | 为什么表数据删掉一半，表文件大小不变？

主要内容：数据库表的空间回收。为什么简单地删除表数据到不到表空间回收的效果，如何正确回收空间？

InnoDB表包含两部分：表结构定义和数据。MySQL8.0之前，表结构是存在 .frm 为后缀的文件里，8.0版本则允许把表结构定义放在系统数据表中了。

### 参数 innodb_file_per_table

参数 innodb_file_per_table 设置为OFF时，表的数据放在共享表空间，也就是跟数据字典放在一起；当设置为ON表示，每个InnoDB表数据存储在一个以.idb为后缀的文件中。从MySQL5.6.6开始，默认为ON。

建议设置该值为ON。因为单独存储为一个文件更容易管理，可以通过drop table命令直接删除这个文件。如果放在共享表空间中，即使表删掉了，空间也是不会回收的。

### 数据删除流程

记录和数据页删除后，会被标记为删除可复用。记录的复用只限于符合范围条件的数据，当有新的记录插入时，如果位置范围是在删除的记录上，则复用。对于数据页则是可复用与任何位置。

如果相邻的两个数据页利用率都很小，系统会把两个数据页上的数据合并到一个页上，另外一个数据页就被标记为可复用。

如果用delete命令把整个表的数据删除，那么所有的数据页都被标记为可复用。但磁盘上，文件不会变小。

delete命令只是把记录的位置，或者数据页标记为“可复用”，但磁盘文件的大小是不会变的。这些可复用，而没有被利用的空间，看起来就像是“空洞”。

不仅删除数据会造成空洞，插入数据，更新索引上的值也会造成空洞。当一个在一个已满的数据页上插入数据，就会申请一个新的页面保存数据，页分裂完成后，老的数据页尾就留下了空洞。更新索引值，可以理解为删除一个旧的值，再插入一个新值。

### 如何去除空洞，收缩表空间：重建表

可以使用 alter table A engine=InnoDB命令来重建表。该命令的执行流程是：新建一个与表A结构相同的表B，然后按照主键ID递增的顺序，把数据一行一行从表A读出插入表B。表B就没有像表A中的空洞了，索引更加紧凑。完成后用表B替换表A。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826213749832.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

在MySQL5.5之前，流程和上述描述差不多，区别是临时表B不需要你自己创建，MySQL会自动完成上述操作。在上述往临时表插入数据过程中，如果有新的数据要写入表A的话，就会造成数据丢失。因此在整个DDL过程中，表A不能有更新，也就是DDL不是Online的。

MySQL5.6版本开始引入Online DDL，对操作流程做了优化：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826213819998.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826213838523.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826213905897.png)


## 14 | count(*)这么慢，我该怎么办？

**count(*)的实现方式**

不同的MySQL引擎，count(*)有不同的实现方式：

- MyISAM引擎把一个表的总行数存在了磁盘上，因此执行count(*)时会直接返回这个数，效率很高；
- InnoDB引擎需要把数据一行一行地从引擎里面读出来，然后累计计数。

InnoDB没有把总行数存起来的原因是：由于多版本并发控制（MVCC）的原因，InnoDB表“应该返回多少行”也是不确定的。

在保证逻辑正确的前提下，尽量减少扫描的数据量，是数据库系统设计的通用法则之一。

小结：

- MyISAM表虽然count(*)很快，但是不支持事物
- show table status 命令虽然返回很快，但是不准确
- InnoDB表直接count(*)会遍历全表，虽然结果准确，但会导致性能问题

如果要经常统计操作记录总数的话，应该自己把操作记录表的行数存起来。


### 用缓存系统保存计数

使用redis缓存来保持计数会存在不精确的问题：

- redis可能会出现异常重启的情况，这时如果内存中的数据没用及时持久化的话，会丢失更新计数。
- 即使redis正常工作，也会出现逻辑上不精确
	- 先插入数据库数据，再更新redis计数值。这时在它们中间有一个线程读取了数据库，是有最新的插入数据，但读redis时没有计数没有增加这条记录，就会出现数据不精确
	- 如果先更新redis计数值，再插入数据库数据。同样在它们中间一个线程先读取redis，再读取数据库，这时不能读取到要新插入到数据。


### 在数据库保存计数

如果把计数存在一个表中，就可以解决存在缓存中出现的问题：

- InnoDB支持崩溃恢复不丢失数据
- 通过InnoDB事物特性，可以确保在执行累加和插入操作未提交事物时，其他事物是看不到中间状态结果的。


### 不同的count用法

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826214434947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


count(*), count(主键 id), count(字段), count(1) 性能分析：

- count(主键 id): InnoDB会遍历整张表，取出每一行id，然后返回给server层。server层再判断id不可能为null，按行累加。
- count(1): 遍历整张表，但不取值，server层对返回的每一行放”1”进去，判断不可能为空，按行累加。
- count(字段)：
	- 如果该字段定义为非null的话，一行行从记录里取出这个字段，判断不可能为null，按行累加。
	- 如果这个“字段”定义允许为 null,那么执行的时候,判断到有可能是 null,还要把值取出来再判断一下，不是null时才累加。
- count(*) 是例外,并不会把全部字段取出来,而是专门做了优化,不取值。count(*) 肯定不是 null,按行累加。

结论是:按照效率排序的话,count(字段)<count(主键 id)<count(1)≈count(*),所以建议尽量使用count(*)。


## 15 | 答疑文章（一）：日志和索引相关问题



## 16 | “order by”是怎么工作的？

### 全字段排序

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826214618758.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


排序动作的执行，可能在内存中完成，也可能需要使用外部排序，利用磁盘临时文件辅助排序。这取决于排序所需要的内存和MySQL为排序开辟的内存（sort_buffer）的大小。sort_buffer的大小由sort_buffer_size参数决定。

内部排序使用快速排序算法，外部排序一般使用归并排序算法。


### rowid排序

上述排序如果单行数据很大，那么排序性能就会很差。

修改参数，当单行长度超过该值时，MySQL就会换一种算法。先取排序的字段和ID，然后排序，最后再回表取出结果集。

    SET max_length_for_sort_data = 16;


![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826214709750.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 全字段排序 VS rowid 排序

如果MySQL认为内存足够大，会 优先选择全字段排序，若内存不够用则选择rowid排序。

MySQL的一个设计思想：如果内存够，就要多利用内存，尽量减少磁盘访问。


建立联合索引，避免执行排序：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826214753826.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


使用覆盖索引，避免回表查询：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019082621481143.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)



## 17 如何正确地显示随机消息？

### 内存临时表

Extra 字段显示 Using temporary，表示的是需要使用临时表； Using filesort，表示的是需要执行排序操作。

对于InnoDB表来说，执行全字段排序会减少磁盘访问，因此会被优先选择。

对于内存表，回表过程只是简单地根据数据行的位置，直接访问内存得到数据，根本不会导致多访问磁盘。优化器这时就会优先考虑的是，用于排序的行越小越好（是因为单行数据很大，那么排序性能就会很差），所以，MySQL这时就会选择rowid排序。

如果你创建的表没有主键，或者把一个表的主键删掉了，那么InnoDB会自己生成一个长度为6字节的rowid来作为主键。


![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826210342342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826210424466.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826210453137.png)


order by rand() 使用了内存临时表，内存临时表排序的时候使用的是rowid排序方法。

### 磁盘临时表

如果临时表大小超过了tmp_table_size（默认值是16M），那么内存临时表就会转成磁盘临时表。

当使用磁盘临时表的时候，对应的就是一个没有显式索引的InnoDB表的排序过程。

临时文件算法-->归并排序算法

优先队列排序算法:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826210557289.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 随机排序方法

    mysql> select count(*) into @C from t;
    set @Y1 = floor(@C * rand());
    set @Y2 = floor(@C * rand());
    set @Y3 = floor(@C * rand());
    select * from t limit @Y1，1； // 在应用代码里面取 Y1、Y2、Y3 值，拼出 SQL 后执行
    select * from t limit @Y2，1；
    select * from t limit @Y3，1；

在实际应用的过程中，比较规范的用法就是：尽量将业务逻辑写在业务代码中，让数据库只做“读写数据”的事情。


## 18 | 为什么这些SQL语句逻辑相同，性能却差异巨大？

### 条件字段函数操作

如果对字段做了函数计算，就用不上索引了。原因：对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。 优化器会选择遍历索引（主键索引或字段索引，取决于索引大小）。

要想使用到索引的快速定位能力，就要把字段的函数操作改为范围查询。比如下面两个：

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826210720624.png)---》

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826210751825.png)
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826210825197.png) ---》   where id=9999


### 隐式类型转换

在mysql查询语句中，如果作为条件的值类型与字段类型不一样时，可能会存在类型转换，比如vachar类型和整数。在mysql中，字符串和数字比较的话，是将字符串转换成数字。如果varchar类型的字段转换成数字的话，将会用到函数，从而就用不上索引，将全表扫描。


### 隐式字符编码转换

当两个表的字符集不同时（例如utf8、utf8mb4），做表关联查询的时候用不上关联字段的索引。原因：当两个字符集不同时，MySQL就会先做类型转换，再进行比较。转换规则是“按数据长度增加的方向”进行转换，类似于程序设计语言中的自动类型转换。如果是被驱动表里的字段做转换，就会用到函数操作，因此该字段的索引将无法使用，优化器会放弃走树搜索功能。由此可以得出导致被驱动表做全表扫描的直接原因是：**连接过程中要求在被驱动表的索引字段上加函数操作。**

优化方案：

1. 改变表的编码，两个表统一编码，就不会存在字符编码转换了。这需要DDL操作，如果数据量大或业务上暂时不允许则要改变SQL语句了
2. 改变SQL语句，主动把驱动表的关联字段改变编码和被驱动表一样。


例如：

    alter table trade_detail modify tradeid varchar(32) CHARACTER SET utf8mb4 default null;

    select d.* from tradelog l, trade_detail d where d.tradeid=CONVERT(l.tradeid USING utf8) and l.id=2;


### 总结：

对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。


## 19 | 为什么我只查一行的语句，也执行这么慢？

### 第一类：查询长时间不返回

碰到这种情况大概率表被锁住了。可以使用 show processlist 命令，查看当前语句处于什么状态，然后根据不同的状态，用不同的方法处理。

### 等MDL锁

如果语句对应的状态是 Waiting for table metadata lock, 那么表示的是，现在有一个线程正在表t上请求或者持有MDL写锁，把select语句堵住了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826211152420.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

这类问题的处理方式，就是kill掉持有MDL写锁的线程。


### 等 flush

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826211451436.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

当状态是 waiting for table flush 时，表示一个线程正要对表t做flush操作，flush操作通常很快。而这个flush table命令被别的语句堵住了，flush命令又堵住了select语句。


### 等行锁

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019082621154136.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)



通过下面语句查询阻塞线程，然后通过  kill 线程号，杀死线程。

    mysql> select * from t sys.innodb_lock_waits where locked_table=`'test'.'t'`\G



### 第二类 慢查询

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190826213209528.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

session B 更新完100万次，会生成100万个回滚日志（undo log）。 session A 第一个select是一致性读，会依次执行 undo log，执行了100万次以后，才将1结果返回。 session A第二个select查询是当前读，因此会直接得出1000001 结果。



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


## 33 | 我查这么多数据，会不会把数据库内存打爆？

### 全表扫描对 server 层的影响

服务端取数据和发数据的流程是：

1. 获取一行，写到 net_buffer 中。这块内存的大小是由参数 net_buffer_length 定义的，默认是 16k；
2. 重复获取行，直到 net_buffer 写满，调用网络接口发出去。
3. 如果发生成功，就清空 net_buffer，然后继续取下一行，并写入 net_buffer。
4. 如果发生函数返回 EAGAIN 或 WSAEWOULDBLOCK ，就表示本地网络栈（socket send buffer）写满了，进入等待。直到网络栈重新可写，再继续发送。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190721151721490.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


从流程中可以看到：

1. 一个查询在发送过程中，占用的 MySQL 内部的内存最大就是 net_buffer_length 这么大，并不会达到 200G；
2. socket send buffer 也不可能达到 200G （默认定义 /proc/sys/net/core/wmem_default），如果 socket send buffer 被写满，就会暂停读数据的流程。

MySQL 是“边读边发的”。如果客户端接收得慢，会导致 MySQL 服务端由于结果发不出去，这个事务的执行时间变长。

如果客户端不去读 socket receive buffer 中的内容，服务端线程的状态就会是 “Sending to client”，表示服务器端的网络栈写满了。

对于正常的线上业务来说，如果一个查询的返回结果不会很多的话，建议使用 mysql_store_result 这个接口，也就是直接把查询结果保存在本地内存。如果执行了大查询，导致客户端占用很多内存，就需要使用 mysql_use_result 接口了，也就是不缓存，读一个处理一个。

一个查询语句的状态变化过程：

- MySQL 查询语句进入执行阶段后，首先把状态设置成 “Sending data”；
- 然后，发送执行结果的列相关的信息（meta data）给客户端；
- 再继续执行语句的流程；
- 执行完成后，把状态设置成空字符串。

仅当一个线程处于“等待客户端接收结果”的状态，才会显示“Sending to client”；而如果显示成“Sending data”，它的意思只是“正在执行”。

查询的结果是分段发给客户端的，因此扫描全表，查询返回大量的数据，并不会把内存打爆。


### 全表扫描对 InnoDB 的影响

内存的数据页是在 Buffer Pool（BP）中管理的，在 WAL里 Buffer Pool 不仅起到了加速更新的作用，还起到了加速查询的作用。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019072115213699.png)

InnoDB 内存管理用的是最近最少使用（LRU）算法，算法的核心就是淘汰最久未使用的数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190721152202664.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

为了防止业务查询出大量不经常使用的数据，把 Buffer Pool 的空间之前的数据全部淘汰，存储这些未经常使用的数据，导致 Buffer Pool 命中率急剧下降，磁盘压力增加，SQL语句响应变慢，InnoDB对 LRU 算法做了改进。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190721152239216.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

在 InnoDB 实现上，按照 5:3 的比例把整个 LRU 链表分成了 young 区域和 old 区域。图中 LRU_old 指向的就是 old 区域的第一个位置，是整个链表的 5/8 处。靠近链表头部的 5/8 是 young 区，靠近链表尾部的 3/8 是 old 区域。

改进后 LRU 算法执行流程：

1. 图7中状态1，要访问数据页 P3，由于 P3 在 young 区，因此和优化前的算法一样，将其移动到链表头部，变成状态 2；
2. 之后访问一个新的不存在与当前链表的数据页，依然淘汰链表尾部的数据页 Pm，但是新插入的数据页 Px ，是放在 LRU_old 处。
3. 处于 old 区域的数据页，每次被访问的时候都要做下面这个判断：
	- 若这个数据页在 LRU 链表中存在的时间超过了 1 秒，就把它移动到链表头部；
	- 如果这个数据页在 LRU 链表中存在的时间短于 1 秒，位置保持不变。1 秒这个时间，是由参数 innodb_old_blocks_time 控制，默认是 1000，单位毫秒。

通过这个策略，在扫描大表的过程中，虽然也用到了 Buffer Pool，但是对 young 区域完全没有影响，从而保证了 Buffer Pool 响应正常业务的查询命中率。


### 小结

由于 MySQL 采用的是边算边发的逻辑，因此对于数据量很大的查询结果来说，不会在 server 端保存完整的结果集。所有，如果客户端读结果不及时，会堵住 MySQL 的查询过程，但是不会把内存打爆。

对于 InnoDB 引擎内部，由于有淘汰策略，大查询也不会导致内存暴涨。并且，由于 InnoDB 对 LRU 算法做了改进，冷数据的全部扫描，对 Buffer Pool 的影响也能做到可控。


## 34 | 到底可不可以使用join？

### Index Nested-Loop join

    select * from t1 straight_join t2 on (t1.a=t2.a);

这个语句中改用 straight_join 让 MySQL 使用固定的连接方式执行查询，这样优化器只会按照我们指定的方式去 join。在这个语句中，t1 是驱动表，t2 是被驱动表。

这条语句的执行过程是，先遍历表 t1 ，然后根据从表 t1 中取出的每一行数据中的 a 值，去表 t2 中查找满足条件的记录。类似嵌套查询，用上了被驱动表中索引，称之为“Index Nested-Loop Join”，简称 NLJ。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190721152434866.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

#### 能不能使用 join ？

如果不使用 join ，那么客户端会先用一个SQL语句查出 t1 表，再for循环连接数据库N次查询。效率显然不高，也就是 N+1 问题。

#### 怎么选择驱动表？

join 语句执行过程中，驱动表是走全表扫描，被驱动表是走树搜索（前提是被驱动表上有索引）。

假设被驱动表的行数是 M，每次在被驱动表查一行数据，要先搜索索引 a，再搜索主键索引。时间复杂度为 2*log2M。驱动表扫描 N 行，那么整个过程中复杂度近似  N + N*2*log2M。

因此使用 join 语句的话，需要让小表做驱动。


### Simple Nested-Loop Join

如果被驱动表（t2）在匹配时没有用到索引，那么每次到 t2 去匹配的时候，就要做一次全表扫描，扫描行数就是 N * M 了。虽然结果是正确的，但是算法看上去太“笨重了”。这个算法叫做“Simple Nested-Loop Join”。


### Block Nested-Loop Join

MySQL也并没有使用 Simple Nested-Loop Join 算法，而是使用了 “Block Nested-Loop Join” 算法，简称 BNL。

被驱动表上没有可用的索引，算法的流程如下：

1. 把表 t1 的数据读入线程内存 join_buffer 中，如果没有where条件，则是整个 t1 表放入内存；
2. 扫描 t2 表，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190721152550198.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

可以看到整个过程中扫描的行数是 N + M。由于 join_buffer是以无序数组的方式组织，因此对表 t2 中的每一行，都要做 N 次判断。总共需要在内存中做 N * M 次判断。虽然 SNL 算法也是做了 N * M 次判断，但是BNL是在内存中做的判断，因此性能更好。

对于哪个表做驱动表更好，这里要考虑的因素是 join_buffer 的大小。如果 join_buffer 足够大，那么无论谁做驱动表，join_buffer 都可以一次性存入驱动表的全部数据，算法的扫描行数值都是 N + M ，内存判断次数值都是 N * M。

如果 join_buffer 存不下驱动表的全部内容，那么就会采用分段放的策略。其执行过程变为：

1. 扫描表 t1 ，顺序读取数据行放入 join_buffer 中，放到某一行时 join_buffer 满了，继续第 2 步；
2. 扫描表 t2，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。
3. 清空 join_buffer；
4. 继续扫描表 t1，顺序读取剩余数据放入 join_buffer 中，继续执行第 2 步。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190721152650595.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


用 λ * N 表示驱动表的分段数，λ 的取值范围是 (0, 1)。那么算法的执行过程：

1. 扫描行数是： N +  λ * N * M ;
2. 比较次数还是 N * M ; 


因此，在 join_buffer 大小不足以存入驱动表的全部数据情况下，驱动表为小表时，性能更好。

第一个问题：能不能使用 join 语句？

1. 在被驱动表能用上索引的情况下，也就是 NLJ 算法，是可以使用 join 语句的。
2. 如果被驱动表不能使用索引，也就是 BNL 算法，扫描行数和比较次数会很多，特别是在大表的情况下，会占用大量的系统资源。所以join 尽量不要用。


第二个问题：如果使用 join 语句，应该使用大表做驱动表，还是小表做驱动表？

1. 如果是 NLJ 算法，应该选择小表；
2. 如果是 BNL 算法：
	1. 在 join_buffer_size 足够大时，都一样；
	2. 在 join_buffer_size 不够大的时候（通常情况下），选择小表。

所以这个问题的结论是选择小表做驱动表。

更准确的说，在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。


## 35 | join语句怎么优化？Multi-Range Read 优化

### MRR 优化的主要目的是尽量使用顺序读盘。

回表是一行行搜索主键索引的，因为大多数的数据都是按照主键递增顺序插入得到的，所以可以认为，如果按照主键的顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能。

MRR 优化后的执行流程：

1. 根据索引 a，定位到满足条件的记录，将 id 值放入 read_rnd_buffer 中；
2. 将 read_rnd_buffer 中的 id 进行递增排序；
3. 排序后的 id 数组，一次到主键 id 索引中查找记录，并作为结果返回。

read_rnd_buffer 的大小是由 read_rnd_buffer_size 参数控制。如果步骤 1 中 read_rnd_buffer 放满了，那么就先执行 2 、3 ，清空 read_rnd_buffer。之后继续找索引 a 的下一个记录，循环此步骤。

如果想要稳定地使用 MRR 优化的话，需要设置 set optimizer_switch="mrr_cost_based=off"。现在的优化器策略更倾向于不使用 MRR。

需要注意的是，由于我们在 read_rnd_buffer 中按照 id 进行了排序，所以最后得到的结果集也是按照 id 递增的顺序。

MRR 能够提升性能的核心在于，查询语句在 a 索引上做范围查询，这样就可以排序主键 id，然后顺序读。


### Batched Key Access

MySQL 5.6 版本引入 Batched Key Access （BKA）算法。其实就是对 NLJ 算法的优化。

NLJ 算法是从驱动表 t1 中，一行行取出 a 的值，再到被驱动表 t2 去做 join。对于 t2 表来说每次都是匹配一个值，这时 MRR 优化是用不上的。

NLJ 算法是没有用到 join_buffer的。要想一次性多传些值给表 t2，使用到 MRR 优化，可以通过先把从表 t1 取出来的数据放入 join_buffer，然后再传给表 t2，这就是 BKA 算法。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190721192011130.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


如果要使用 BKA 优化算法的话，需要设置：

    set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';


### BNL 算法的性能问题

在使用 BNL 算法时，如果被驱动表是一个大的冷数据表，除了会导致 IO 压力大以外，还会影响 Buffer Pool 的命中率。

多次扫描一个冷表，这个语句的执行时间超过 1 秒时，就会把冷表的数据页移到 LRU 链表头部。这种情况是对应与冷表数据量小于 Buffer Pool 的 3/8 时。如果冷表数据很大，old区域的数据页很有可能在 1 秒之内就被淘汰了。这样就不会进入young 区域，导致 young 区域的数据页没有被合理地淘汰。

大表 join 操作虽然对 IO 有影响，但是在语句执行结束后，对 IO 的影响也就结束了。而对于 Buffer Pool 的影响就是持续性的，需要依靠后续的查询请求慢慢恢复内存命中率。

为了减少这种影响，可以考虑增多 join_buffer_size 的值，减少对被驱动表的扫描次数。

BNL 算法对系统的影响主要包括三个方面：

1. 可能会多次扫描被驱动表，占用磁盘 IO 资源；
2. 判断 join 条件需要执行 M*N 次对比，如果是大表就会占用非常多的 CPU 资源；
3. 可能会导致 Buffer  Pool 的热数据被淘汰，影响内存命中率。

优化常见的做法是在被驱动表上给 join 字段加索引，把 BNL 算法转成 BKA 算法。


### BNL 转 BKA

一些情况下，可以直接在被驱动表上建索引，这时可以直接转成 BKA 算法。但是有些时候不适合建索引，比如这个 join SQL语句是低频执行，而且经过 where 条件过滤后数据量很少，如果这时加索引的话就很浪费。如果不加索引，就要比对很多次，占用 CPU 资源。可以同过使用临时表来解决：

1. 把表 t2 中满足条件的数据放在临时表 tmp_t 中；
2. 为了让 join 使用 BKA 算法，给临时表 tmp_t 的字段 b 加上索引；
3. 让表 t1 和 tmp_t 做 join 操作。

    create temporary table temp_t(id int primary key, a int, b int, index(b))engine=innodb;
    insert into temp_t select * from t2 where b>=1 and b<=2000;
    select * from t1 join temp_t on (t1.b=temp_t.b);


### 扩展 -hash join

如果 join_buffer 里面维护的不是一个无序的数组，而是一个哈希表的话，那么就不是 10 亿次判断，而是 100 万次hash 查找，这样速度就快很多。但目前MySQL还不支持。

可以把优化放在业务端实现。比如把表 t1 的数据存入一个hash 结构里，再取出表 t2 数据，然后去 hash 结构中比较。


### 小结

优化方法中：

1. BKA 优化是 MySQL 已经内置支持的，建议默认使用；
2. BNL 算法效率低，建议尽量转成 BKA 算法。优化的方向是给驱动表的关联字段加上索引；
3. 基于临时表的改进方案，对于能够提前过滤出小数据的 join 语句来说，效果还是很好的；
4. MySQL 目前的版本还不支持 hash join， 但你可以配合应用端自己模拟出来，理论上效果好于临时表方案。


## 36 | 为什么临时表可以重名？

内存表：指的是使用 Memory 引擎的表，建表语法是 create table ... engine=memory。这种表的数据都保存在内存里，系统重启的时候会被清空，但是表结构还在。

临时表：可以使用各种引擎类型。如果是使用 InnoDB 引擎或者 MyISAM 引擎的临时表，写数据的时候是写到磁盘上的。


### 临时表的特性

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190728175205930.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


临时表在使用上的几个特点：

1. 建表语法是 create temporary table ...。
2. 一个临时表只能被创建它的 session 访问，对其他线程不可见。
3. 临时表可以与普通表同名。
4. session A 内有同名的临时表和普通表的时候，show create 语句，以及增删改查语句访问的是临时表。
5. show tables 命令不显示临时表。
6. 创建临时表的 session 结束后，会自动删除临时表。


在 join 优化时使用临时表的原因：

1. 不同 session 的临时表是可以重名的，如果有多个 session 同时执行 join 优化，不需要担心表名重复导致建表失败的问题。
2. 不需要担心数据删除问题。如果使用普通表，在流程执行过程中客户端放生异常断开，或者数据库发生异常重启，还需要专门来清理中间过程中生成的数据表。而临时表会自动回收。




### 临时表的应用

临时表经常会被用在复杂查询的优化过程中。分库分表系统的跨库查询就是一个典型的使用场景。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190728175333438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

当查询条件里面没有用到分区字段 f，只能到所有的分区中取查找满足条件的所有行，然后统一做 order by 操作。有两种常用的思路。

第一种思路是，在Proxy 层的进程代码中实现排序。优点是处理速度快，拿到分库的数据以后，直接在内存中参与计算。不过这个方案缺点也很明显：

1. 需要的开发工作量比较大。如果涉及到 group by、join 等操作，对中间层的开发能力要求比较高。
2. 对 Proxy 端的压力比较大，尤其是很容易出现内存不够用和CPU 瓶颈的问题。


第二种思路是，把各个分库拿到的数据，汇总到一个 MySQL 实例的一个表中，然后在这个汇总实例上做逻辑操作。流程类似于：

- 在汇总库上创建一个临时表 temp_ht，表里包含三个字段 v、k、t_modified；
- 在各个分库上执行


    select v,k,t_modified from ht_x where k >= M order by t_modified desc limit 100;


- 把分库执行的结果插入到 temp_ht 表中；
- 执行下面语句得到结果

    select v from temp_ht order by t_modified desc limit 100;


![在这里插入图片描述](https://img-blog.csdnimg.cn/20190728175523813.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### 为什么临时表可以重名？

在创建临时表时，MySQL会给表创建一个 frm 文件保存表结构定义，还要有地方保存数据。frm 文件放在临时目录下，文件名后缀是 .frm，前缀是“#sql{进程 id}_{线程 id}_序列号”。

临时表数据存放：

- 在 5.6 及以前版本，MySQL 会在临时文件目录下创建一个相同前缀、以 .ibd 为后缀的文件，用来存放数据文件；
- 从 5.7 版本开始，MySQL 引入了一个临时文件表空间，专门用来存放临时文件数据。不需要再创建 .ibd 文件了。

在内存中每一个表都对应一个 table_def_key，用来区分不同的表。

- 普通表的 table_def_key 的值是由“库名 + 表名”组成。
- 临时表是在 “库名 + 表名”的基础上，又加了“server_id + thread_id”。

因此对于两个不同的 session 创建相同表名的临时表，它们的 table_def_key 不同，磁盘文件名也不同，因此可以并存。

每个线程都有维护自己的临时表链表。每次 session 操作表的时候，先遍历链表检查是否有这个名字的临时表，如果有就优先操作临时表，如果没有再操作普通表；在 session 结束时候，对链表里的每个临时表，执行 “DROP TEMPORARY TABLE + 表名”操作。binlog 中也记录了这个 DROP ... 这条命令。


### 临时表和主备复制

在主备复制时，普通表的操作需要用到临时表，因此备库也要执行操作临时表相关的语句，所以对临时表的操作会写入 binlog。如果当前的 binlog_format=row，那么跟临时表相关的语句，就不会记录到binlog中，因为row格式会记录改变前后的真实数据值。只在 binlog_format=statment/mixed 的时候，binlog 才会记录临时表的操作。

主库 M 上的两个 session 创建了同名的临时表 t1，这两个 create temporary table t1 语句都会被传到备库 s 上。MySQL在记录 binlog 的时候，会把主库执行这个语句的线程 id 写到 binlog 中，table_def_key 的命名规则是：库名 + t1 + "M 的 serverId" + "session A/B 的 thread id"


### 小结

临时表一般用于处理比较复杂的计算逻辑。由于临时表是每个线程自己可见的，所以不需要考虑多个线程执行同一个逻辑时的重名问题。相处退出时，临时表也能自动删除。

在 binlog_format='row' 的时候，临时表的操作不记录到binlog中。



## 37 | 什么时候会使用内部临时表？

union 执行流程


    (select 1000 as f) union (select id from t1 order by id desc limit 2);

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190728175717441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

可以看到，这里的内存临时表起到了暂存数据的作用，而且计算过程中还用上了临时表主键 id 的唯一性约束，实现了 union 的语义。如果上面的 union 改成 union all 的话，就没有了“去重”的语义，得到的结果直接作为结果集的一部分返回给客户端。因此也用不上临时表了。


### group by 执行流程

    select id%10 as m, count(*) as c from t1 group by m;

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190728175935714.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

1. 创建内存临时表，表里有两个字段 m 和 c，主键是 m；
2. 扫描表 t1 的索引 a，依次取出叶子节点上的 id 值，计算 id%10 的结果，记为 x；
	- 如果临时表中没有主键为 x 的行，就插入一个记录 (x,1);
	- 如果表中有主键为 x 的行，就将 x 这一行的 c 值加1；
3. 遍历完成后，在根据字段 m 做排序，得到结果集返回给客户端。（注：5.7 版本有这一步，8.0版本就没有最后的排序）



内存表的大小是有限制的，参数 tmp_table_size 就是控制这个内存大小的，默认是 16M。如果执行过程中发现内存临时表大小到达了上限，就会被内存临时表转成磁盘临时表，磁盘临时表默认使用 InnoDB 引擎。


### group by 优化方法 -- 索引

group by 的语义逻辑，是统计不同的值出现的个数。由于每一行的 id%100 的结果是无序的，所以我们就需要一个临时表，来记录并统计结果。

但是如果扫描过程中可以保证出现的数据是有序的，那么 group by 将不再使用临时表，也不需要排序（5.7）。只需要从左到右，顺序扫描，依次累加。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190728180008340.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


- 当碰到第一个 1 的时候，已经知道累计了 x 个 0，结果集里的第一行就是 （0,x）;
- 当碰到第一个 2 的时候，已经知道累计了 y 个 1，结果集里的第二行就是 （1,y）;


在 MySQL 5.7 版本支持了 generated column 机制，用来实现列数据的关联更新。你可以用下面的方法创建一个列 z，然后在 z 列上创建一个索引。

    alter table t1 add column z int generated always as(id % 100), add index(z);

    select z, count(*) as c from t1 group by z;


### group by 优化方法 -- 直接排序

在 group by 语句中加入 SQL_BIG_RESULT 这个提示，就可以告诉优化器：这个语句涉及的数据量很大，请直接用磁盘临时表。避免是先使用内存临时表，在转换为磁盘临时表。

MySQL的优化器发现，磁盘临时表是 B+ 数存储，存储效率不如数组高，就会直接用数组存储。

    select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;

执行流程：

1. 初始化 sort_buffer，确定放入一个整型字段，记为 m；
2. 扫描表 t1 的索引 a，依次取出里面的 id 值，将 id%100 的值存入 sort_buffer 中；
3. 扫描完成后，对 sort_buffer 的字段 m 做排序 （如果 sort_buffer 内存不够用，就会利用磁盘临时文件辅助排序）；
4. 排序完成后，就得到了一个有序数组。
5. 根据有序数组，得到数组里面的不同值，以及每个值的出现次数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190728180152487.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


MySQL什么时候会使用内部临时表：

1. 如果语句执行过程可以一边读数据，一边直接得到结果，是不需要额外内存的，否则就需要额外的内存，来保存中间结果。
2. join_buffer 是无序数组，sort_buffer 是有序数组，临时表是二维表结构；
3. 如果执行执行逻辑需要用到二维表特性，就会优化考虑使用临时表。比如我们的例子中，union 需要用到唯一索引约束，group by 还需要用到另外一个字段来保存累计数。

### 小结

group by 使用原则：

1. 如果对 group by语句的结果没有排序要求，要在语句后面加 order by null（8.0 版本没有加 order by 将不再排序）；
2. 尽量让 group by 过程用上表的索引，确认方法是 explain 结果里没有 Using temporary 和 Using filesort；
3. 如果 group by 需要统计的数据量不大，尽量只是用内存临时表；也可以通过适当调大 tmp_table_size 参数，来避免用到磁盘临时表；
4. 如果数据量是在太大，使用 SQL_BIG_RESULT 这个提示，来告诉优化器直接使用排序算法得到 group by 的结果。


## 38 | 都说InnoDB好，那还要不要使用Memory引擎？

### 内存表的数据组织结构

InnoDB 表的数据就放在主键索引树上，主键索引是 B+ 树。主键索引上的值是有序存储的，在执行 select * 的时候，就会按照叶子节点从左到右扫描。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019080122043253.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

与 InnoDB 引擎不同，Memory 引擎的数据和索引是分开的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190801220453311.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

内存表的数据部分以数组的方式单独存放，而主键 id 索引里，存的是每个数据的位置。主键 id 是 hash 索引，可以看到索引上的key并不是有序的。

InnoDB 和 Memory 引擎的数据组织方式是不同的：

- InnoDB 引擎把数据放在主键索引上，其他索引上保存的是主键 id。这种方式，我们称之为索引组织表（Index Organizied Table）。
- Memory 引擎采用的是把数据单独存放，索引上保存数据位置的数据组织形式，我们称之为堆组织表（Heap Organizied Table）。

两个引擎的一些典型不同：

1. InnoDB 表的数据总是有序存放的，而内存表的数据是按照写入顺序存放的；
2. 当数据文件有空洞的时候，InnoDB 表在插入数据的时候，为了保证数据有序性，只能在固定的位置写入新值，而内存表找到空位就可以插入新值；
3. 数据位置发生变化的时候，InnoDB 表只需要修改主键索引，，而内存表需要修改所有索引；
4. InnoDB 表用主键索引查询时需要走一次索引查找，用普通索引查找时需要走两次索引查找。而内存表没有这个区别，所有索引的“地位”都是相同的；
5. InnoDB 支持变长数据类型，不同记录的长度可能不同；内存表不支持 Blob 和 Text 字段，并且即使定义了 varchar(N)，实际也当作 char(N)，也就是固定长度字符串来存储，因此内存表的每行数据长度相同。


### hash 索引和 B-Tree 索引

内存表也支持 B-Tree 索引。

    alter table t1 add index a_btree_index using btree (id);

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190801220652574.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190801220726546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


不建议在生产环境使用内存表的原因：

1. 锁粒度问题；
2. 数据持久化问题。


### 内存表的锁

内存表不支持行锁，只支持表锁。因此，一张表只要有更新，就会堵住其他所有在这个表上的读写操作。


数据持久化问题

数据放在内存中，是内存表的优势，同时也是一个劣势。数据库在重启的时候，所有的内存表都会被清空。

把普通内存表都用 InnoDB 表来代替：

1. 如果你的表更新量大，那么并发读一个很重要的参考标准，InnoDB支持行锁，并发度比内存表好；
2. 能放到内存表的数据量都不大。如果考虑的是读性能，一个读 QPS 很高并且数据量不大的表，即使使用 InnoDB，数据也会缓存在 InnoDB Buffer Pool 里的，因此性能也不会差。


内存临时表可以忽视掉内存表的两个不足：

1. 临时表不会被其他线程访问，没有并发性的问题；
2. 临时表重启后也是需要删除的，清空数据这个问题不存在；
3. 备库的临时表也不会影响主库的用户线程。

在查询语句优化时，使用内存临时表比 InnoDB 临时表效果更好的原因：

1. 相比于 InnoDB 表，使用内存表不需要写磁盘，速度更快。
2. 索引 b 使用 hash 索引，查找的速度比 B-Tree 索引快；
3. 临时表数据小，占用的内存有限。


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


## 40 | insert语句的锁为什么这么多？

### insert ... select 语句

当使用 insert ... select 语句时，会在 select 表主键索引上加 next-key lock 锁，锁住需要访问的资源，从而保证在执行时 binlog 和数据的一致性。


### insert 循环写入

对于 insert into t select ... from t 语句的执行，在5.7 版本上所有间隙会加上 next-key lock，但在 8.0 版本，next-key lock 只会加在需要访问的资源间隙。 

### insert 唯一键冲突

在可重复读（repeatable read）隔离级别下，发生唯一键冲突的时候，并不只是简单地报错返回，还在冲突的索引上加了锁。主键索引和唯一索引加的都是 next-key lock（读锁）。


![在这里插入图片描述](https://img-blog.csdnimg.cn/20190801221057845.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190801221119268.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### insert into ... on duplicate key update

insert into ... on duplicate key update 这个语义的逻辑是，插入一行数据，如果碰到唯一键约束，就执行后面的更新语句。会在索引上加一个排他的 next-key lock （写锁）。

如果有多个列违反了唯一性约束，就会按照索引的顺序，修改跟第一个索引冲突的行。


### 小结

在可重复读隔离级别下，使用 insert ... select 语句在两个表直接拷贝数据，会给表里扫描到的记录和间隙加读锁。

如果 insert 和 select 的对象是同一个表，则有可能会造成循环写入。需要引入用户临时表做优化。

insert 语句如果出现唯一键冲突，会在冲突的唯一值上加共享的 next-key lock（S 锁）。因此，在遇到唯一键约束报错后，要尽快提交或回滚事务，避免加锁时间过长。



## 41 | 怎么最快地复制一张表？

如果可以控制对源表的扫描行数和加锁范围很小的话，我们简单地使用 insert ... select 语句即可实现。为了避免对源表加读锁，更稳妥的方案是先将数据写到外部文件，然后再写回目标表。

### mysqldump 方法

使用 mysqldump 命令将数据导出组成一组 INSERT 语句：

    mysqldump -h$host -P$port -u$user --add-locks=0 --no-create-info --single-transaction  --set-gtid-purged=OFF db1 t --where="a>900" --result-file=/client_tmp/t.sql

主要参数含义：
1. -single-transaction 的作用是，在导出数据时不需要对表 db1.t 加表锁，而是使用 START TRANSACTION WITH CONSISTENT SNAPSHOT 的方法；
2. -add-locks 设置为 0，表示在输出的文件结果里，不增加“LOCK TABLES t WRITE;”;
3. -no-create-info 表示不需要导出表结构；
4. -set-gtid-purged=off 表示不输出跟 GTID 相关的信息；
5. -result-file 指定了输出文件的路径，其中 client 表示生成的文件是在客户端机器上的。
6. -skip-extended-insert 表示生成的 insert 语句一条直插入一行数据。

然后可以用下面命令，将这些 insert 语句放到 db2 库里去执行：

    mysql -h127.0.0.1 -P13000  -uroot db2 -e "source /client_tmp/t.sql"

执行流程：

1. 打开文件，默认以分号为结尾读取一条条的 SQL 语句；
2. 将 SQL 语句发送到服务端执行。


### 导出 CSV 文件

直接将结果导出成 .csv 文件：

    select * from db1.t where a>900 into outfile '/server_tmp/t.csv';

需要注意的是：

1. 这条语句会将结果保存在服务端。
2. into outfile 指定了文件的生成位置（/server_tmp/），这个位置受参数 secure_file_priv 的限制。其值和作用分别是：
	1. 如果设置为 empty，表示不限制文件生成的位置，这是不安全的设置；
	2. 如果设置为一个表示路径的字符串，就要求生成的文件只能放在这个指定的目录，或者它的子目录；
	3. 如果设置为 NULL ，就表示禁止在这个 MySQL 实例上执行 select ... into outfile 操作。
4. 这条命令不会覆盖同名的文件，如果有同名文件，将会报错。
5. 生成的文本文件中，原则上一个数据行对应文本文件的一行。换行符、制表符等符号，前面会加上"\"转义符。

用下面的 load data 命令将数据导入到目标表 db2.t 中：

    load data infile '/server_tmp/t.csv' into table db2.t;

执行流程：

1. 打开文件 /server_tmp/t.csv，以制表符（\t）作为字段间的分隔符，以换行符（\n）作为记录之间的分隔符，进行数据读取；
2. 启动事务；
3. 判断每一行的字段数与表 db2.t 是否相同：
	1. 不相同，直接报错，事务回滚；
	2. 相同，则构造成一行，调用 InnoDB 引擎接口，写入到表中。
4. 重复步骤3，知道整个文件读完，提交事务。

在备库重放：

1. 主库执行完成后，将 /server_tmp/t.csv 文件的内容直接写到 binlog 文件中。
2. 往 binlog 文件中写入语句 load data local infile '/tmp/SQL_LOAD_MB-1-0' INTO TABLE `db2`.`t`。
3. 包这个 binlog 日志传到备库。
4. 备库的 apply 线程在执行这个事务日志时：
	1. 先将 binlog 中的 t.csv 文件内容读出来，写入本地临时目录 /tmp/SQL_LOAD_MB-1-0 中；
	2. 再执行 load data 语句，往备库的 db2.t 表中插入跟主库相同的数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825163434784.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


load data 命令两种用法：

1. 不加 “local”，是读取服务端的文件，文件必须在 secure_file_priv 指定的目录或子目录下；
2. 加上 “local”，读取的是客户端的文件。MySQL 客户端会先把本地文件传给服务端，然后执行上述的 load data 流程。

select ... into outfile 方法不会生成表结构文件，mysqldump 提供了一个参数 -tab, 可以同时导出表结构定义文件（t.sql）和 csv（t.txt） 数据文件：

    mysqldump -h$host -P$port -u$user ---single-transaction  --set-gtid-purged=OFF db1 t --where="a>900" --tab=$secure_file_priv

### 物理拷贝方法

MySQL 5.6 版本引入了可传输表空间的方法，可以通过导出 + 导入表空间的方式，实现物理拷贝表的功能。

执行流程：

1. 执行 create table r like t，创建一个相同表结构的空表；
2. 执行 alter table r discard tablespace，这时候 r.ibd 文件会被删除；
3. 执行 flush table t for export，这时候 db1 目录下会生成一个 t.cfg 文件；
4. 在 db1 目录下执行 cp t.cfg rcfg; cp t.ibd r.ibd；
5. 执行 unlock tables，这时候 t.cfg 文件会被删除；
6. 执行 alter table r import tablespace，将这个 r.ibd 文件作为表 r 的新的表空间。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825163629514.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 小结

三种拷贝表方法的优缺点：

1. 物理拷贝方式速度最快，尤其对于大表拷贝来说是最快的方法。但有其局限性：
	1. 必须是全表拷贝，不能只拷贝部分数据；
	2. 需要的服务器上拷贝数据，用户无法登录数据库主机的场景下无法使用；
	3. 由于是通过拷贝物理文件实现的，源表和目标表都是使用 InnoDB 引擎时才能使用；
2. 用 mysqldump 生成含有 INSERT 语句文件的方法，可以在 where 参数增加过滤条件，来实现只导出部分数据。不足之一是，不能使用 join 这种比较复杂的 where 条件写法。
3. 用 select ... into outfile 的方法是灵活的，支持所有的 SQL 写法。缺点是，每次只能导出一张表，而且表结构也需要另外的语句单独备份。



## 42 | grant之后要跟着flush privileges吗？

grant 语句是用来给用户赋权的。

用户权限范围：

### 全局权限

给用户 ua 赋予最高权限：

    grant all privileges on *.* to 'ua'@'%' with grant option;

这个 grant 命令做了两个动作：

1. 磁盘上，将 mysql.user 表里，用户 'ua'@'%' 这一行的所有表示权限的字段的值都修改为 'Y'；
2. 内存里，从数组 acl_users 中找到这个用户对应的对象，将 access 值（权限位）修改为二进制的“全1”。

基于上面的分析可以知道：

1. grant 命令对于全局权限，同时更新了磁盘和内存。命令完成后即时生效，接下来新创建的连接会使用新的权限。
2. 对于一个已经存在的连接，它的全局权限不受 grant 命令的影响。

收回上面 grant 语句赋予的权限，执行动作与上面相反：

    revoke all privileges on *.* from 'ua'@'%';


### db 权限

MySQL 也支持库级别的权限定义。使用下面命令让用户拥有库 db1 的所有权限：

    grant all privileges on db1.* to 'ua'@'%' with grant option;

基于库的权限记录保存在 mysql.db 表中，在内存里则保存在数组 acl_dbs 中。这条命令做了如下动作：

1. 磁盘上，将 mysql.db 表里，用户 'ua'@'%' 这一行的所有表示权限的字段的值都修改为 'Y'；
2. 内存里，增加一个对象到数组 acl_dbs 中，这个对象的权限位为“全1”。

grant 修改 db 权限的时候，是同时对磁盘和内存生效的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825164100140.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 表权限和列权限

MySQL也支持更细粒度的表权限和列权限。表权限定义存放在表 mysql.tables_priv 中，列权限定义存放在表 mysql.columns_priv 中。这两类权限，组合起来存放在内存的 hash 结构 column_pri_hash 中。

赋权命令：

    create table db1.t1(id int, a int);
    
    grant all privileges on db1.t1 to 'ua'@'%' with grant option;GRANT SELECT(id), INSERT (id,a) ON mydb.mytbl TO 'ua'@'%' with grant option;

这两个权限每次 grant 的时候都会修改数据表，也会同步修改内存的 hash 结构。因此这两类权限的操作，会影响到已经存在的连接，也就是立即生效。

flush privileges 命令会清空 acl_users 数组，然后从 mysql.user 表中读取数据重新加载，重新构造一个 acl_users 数组。对于 db 权限、表权限和列权限，MySQL也做同样的处理。

正常情况下，grant/revoke 语句执行完后，内存和数据表会保持同步更新，因此没有必要跟着执行 flush privileges 命令。


### flush privilege 使用场景

在不规范操作的情况下，会造成内存和数据表中的数据不一致，比如直接操作系统表。这时就需要 flush privilege 语句来重建内存数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825164247960.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 小结

grant 语句会同时修改数据表和内存，判断权限的时候是使用内存数据。因此，规范地使用 grant 和 revoke 语句，是不需要随后加 flush privilege 语句的。

flush privilege 语句本身会用数据表的数据重建一份内存权限数据，所以在权限数据可能存在不一致的情况下再使用。而这种不一致往往是由于直接用 DML 语句操作系统权限表导致的，所以尽量不要使用这类语句。



## 43 | 要不要使用分区表？

### 分区表是什么？

把表数据分别存入不同的分区，每个分区对应一个 .idb 文件。对于引擎层来说，这是 n 个表，对于 Server 层来说，这是 1 个表。


### 分区表的引擎层行为

验证对应引擎层来说是 n 个表。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825164338498.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825164357485.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


如果单表过大，不使用分区表的话，就要使用手动分表的方式。分区表和手工分表，一个是由 server 层来决定使用哪个分区，一个是由应用层代码决定使用哪个分表。从引擎层看，这两种方式没有什么差别。性能方面也没有实质的差别。主要区别在 server 层。


### 分区策略

每当第一次访问一个分区表的时候，MySQL 需要把所有的分区都访问一遍。一个典型的报错情况是，分区表的个数已经超过了，open_files_limit 参数的值，就会报错。

MyISAM 分区表使用的分区策略称为通用分区策略（generic partitioning），每次访问分区都由 server 层控制，有比较严重的性能问题。

MySQL 5.7.9 开始，InnoDB 引擎引入 本地分区策略（native partitioning）。这个策略是在 InnoDB 内部自己管理打开分区的行为。

从 MySQL 8.0 版本开始，就不允许创建 MyISAM 分区表了，只允许创建已经实现本地分区策略的引擎。


### 分区表的 server 层行为

1、MySQL 在第一次打开分区表的时候，需要访问所有的分区；
2、在 server 层，认为这是同一张表，因此所有分区共用同一个 MDL 锁；
3、在引擎层，认为是不同的表，因此 MDL 锁之后的执行过程，会根据分区表规则，只访问必要的分区。


### 分区表的应用场景

分区表的一个显而易见的优势是对业务透明，相对于用户分表来说，业务代码更简洁。分区表可以很方便的清理历史数据。



## 44 | 答疑文章（三）：说一说这些好问题

### join 的写法

即使在 SQL 语句中写成了 left join，执行过程还是有可能不是从左到右连接的，在被优化器优化后可能变为 join。也就是说，使用 left join 时， 左边的表不一定是驱动表。

如果需要 left join 的语义，就不能把驱动表的字段放在 where 条件里面做等值判断或不等值判断，必须都写在 on 里面。

join 语句条件是否放到 on 里面，最后优化后都会放到 where 条件里。


### Simple Nested Loop Join 的性能问题

Simple Nested Loop Join 算法相比于 BNL 算法的性能差距在哪：

1. BNL 算法是读取驱动表的数据到 join buffer中，然后遍历被驱动表依次比较 join buffer 中的数据。其比较过程是在内存中完成。
2. SNL 算法是顺序遍历读取驱动表中的每一行数据，到被驱动表中做全表扫描。即使被驱动表数据都在 buffer pool 中，每次查找“下一个记录的操作”，都是类似指针操作。而 join buffer 是数组，遍历的成本更低。
3. SNL 算法对被驱动表做全表扫描时，如果数据没有在 buffer pool 中，就需要等待数据从磁盘读入；从磁盘读入数据到 buffer pool 中，会影响正常业务的命中率，因为被驱动表的数据会多次访问。


### distinct 和 group by 的性能

distinct 的语义是：按照某个字段 a 做分组，相同的 a 的值只返回一行。

group by 在没有聚合函数的情况下，其执行逻辑是一样的，因此性能相同。

执行流程：

1. 创建一个临时表，临时表有一个字段 a ，并且在这个字段 a 上创建一个唯一索引；
2. 遍历表 t，一次取数据插入临时表中；
	1. 如果发现唯一键冲突，就跳过；
	2. 否则插入成功；
3. 遍历完成后，将临时表作为结果集返回给客户端。


### 备库自增主键问题

自增 id 的生成顺序和 binlog 的写入顺序是不同的。

binlog 在记录 insert 语句之前，会先记录 SET INSERT_ID 语句，这个语句表示，这个线程里下一次需要用到自增值的时候，不论当前表的自增值是多少，固定用这个值。因此即使 insert 语句在备库上执行的顺序不同，也不会造成主备数据不一致的问题。


### 45 | 自增id用完怎么办？

### 表定义自增值 id

表定义的自增值达到上限后的逻辑是：再申请下一个 id 时，得到的值保持不变。


### InnoDB 系统自增 row_id

如果创建的 InnoDB 表没有指定主键，那么 InnoDB 会给你创建一个不可见的，长度为 6 个字节（类型是 8 字节的 bigint unsigned，实现上只留 6 个字节的长度）的 row_id。

row_id 的取值特征：

1. row_id 写入表中的范围，是从 0 到 2^48 - 1；
2. 当 dict_sys.row_id=2^48 时，如果再有插入数据的行为要来申请 row_id，拿到以后再取最后 6 个字节的话就是 0 。


写入表的 row_id 是从 0 到 2^48 - 1。达到上限后，下一个值就是 0 ，然后继续循环。

在 InnoDB 逻辑里，申请到 row_id=N 后，就将这行数据写入表中。如果表中已经存在 row_id=N 的行，新写入的行就会覆盖原有的行。因此在建 InnoDB 表时应该主动创建主键。覆盖数据，就意味着数据丢失，影响的是数据可靠性；报主键冲突，是插入失败，影响的是可用性。一般情况下可靠性优先于可用性。


### Xid

MySQL 中 Xid 是用来对应事务的。

MySQL 内部维护了一个全局变量 global_query_id，每次执行语句的时候将它赋值给 Query_id，然后加 1。事务的第一条执行语句执行时，会把 Query_id 赋值给事务的 Xid。

global_query_id 是一个纯内存变量，重启之后就清零。MySQL 重启之后会重新生成新的 binlog 文件。 global_query_id 到达上限 2^64 - 1 后，会继续从 0 开始计数。


### InnoDB trx_id

Xid 是由 server 层维护的。InnoDB 内部使用 Xid ，是为了能够在 InnoDB 事务和 server 之间做关联。

InnoDB 内部维护了一个 max_trx_id 全局变量，每次申请一个新的 trx_id 时，就获得 max_trx_id 的当前值，然后将 max_trx_id 加1。

InnoDB 数据可见性的核心思想是：每一行数据都记录了更新它的 trx_id，当一个事务读到一行数据的时候，判断这个数据是否可见的方法，就是通过事务的一致性视图与这行数据的 trx_id 做对比。

对于只读事务，InnoDB 并不会分配 trx_id，其值是把当前事务的 trx 变量的指针地址转成整数，再加上 2^48。这样做是为了保证：同一个只读事务查出来的 trx_id 是一样的；并发的只读事务查出来的 trx_id 是不同的。

有时候实验的时候会发现，trx_id 的值不止加1。这是因为：

1. update 和 delete 语句除了事务本身，还涉及到标记删除旧数据，也就是把数据放到 purge 队列里等待后续物理删除，这个操作也会把 max_trx_id + 1，因此在一个事务中至少加 2；
2. InnoDB 的后台操作，比如表的索引信息统计这类操作，也是会启动内部事务的。


只读事务不分配 trx_id 的好处：

- 减小事务视图里面活跃事务数组的大小。只读事务不影响数据的可见性判断。
- 减少 trx_id 的申请次数，从而大大减少并发事务申请 trx_id  的锁冲突。


### thread_id

系统保存了一个全局变量 thread_id_counter，每新建一个连接，就将 thread_id_counter 赋值给这个新连接的线程变量。大小是 4 个字节，达到 2^32 -1 后，会重置为 0。


### 小结

每种自增 id 达到上限后的表现不同：

1. 表的自增 id 达到上限后，再申请时它的值就不会改变，进而导致继续插入数据时报主键冲突的错误。
2. row_id 达到上限后，则会归 0 在重新递增，如果出现相同的 row_id ，后写的数据会覆盖之前的数据。
3. Xid 只需要不在同一个 binlog 文件中出现重复值即可。虽然理论上会出现重复值，但是概率极小，可以忽略不计。
4. InnoDB 的 max_trx_id 递增值每次 MySQL 重启都会保存起来，当达到上限后，从 0 开始重新递增，会造成脏读的情况。
5. thread_id 是标记线程的 id。
















































