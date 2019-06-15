*****

## 第6章 消息中间件

**定义**：面向消息的系统（消息中间件）是在分布式系统中完成消息的发送和接收的基础软件。

### 6.2.1 如何解决消息发送一致性

消息发送一致性是指产生消息的业务动作与消息发送的一致。就是说，如果业务操作成功了，那么由这个业务操作产生的消息一定要发送出去。如果业务行为没有发生或者失败，那么就不应该把消息发送出去。

#### 6.2.1.3 JMS解决方案

在JMS的API中，有提供以XA开头的接口，它们其实就是支持XA协议的接口。JMS中定义的XA系列的接口就是为了实现分布式事务的支持。

但是引入分布式事务会带来如下问题：

- 会带来一些开销并增加复杂性
- 对业务操作有限制，要求业务操作的资源必须支持 XA 协议，才能够与发送消息一起来做分布式事务。这会成为限制，因为并不是所有需要与发送消息一起做分布式事务的业务操作都支持 XA 协议。

#### 6.2.1.4 最终一致性解决方案

![](https://img-blog.csdnimg.cn/20190615174909234.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019061517494515.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)



发送消息的正向流程和检查业务操作结果的反向流程合起来，就是解决业务操作与发送消息一致性的最终一致性方案。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175035367.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 6.2.2 如何解决消息中间件与使用者的强依赖问题

消息中间件如果变成了业务应用的必要依赖，那么一旦消息中间件系统（消息存储、业务应用于消息中间件的网络）出现问题，就会导致业务操作无法继续进行。

要解决这个问题，有如下三种思路：

- 提高消息中间件系统的可靠性，但是没有办法保证百分之百可靠。
- 对于消息中间件系统中影响业务操作进行的部分，使其可靠性于业务自身的可靠性相同。
- 可以提供弱依赖的支持，能够较好的保证一致性。

如果消息中间件出现问题，我们可以通过消息存储（数据库或磁盘），延迟投递的方案来保证消息一定发送成功。大致有以下几种方式：1、把消息中间件所需要的消息表和业务数据表发到同一个业务数据库中，这样可以把业务操作和写入消息作为一个本地事务来完成。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175209295.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)



这个方案对业务系统有如下三个影响：

- 需要业务自己的数据库承载消息数据
- 需要让消息中间件去访问业务数据
- 需要业务操作的对象是一个数据库，或者说支持事务的存储，并且这个存储必须能够支持消息中间件的需要。

2、基于第一种改进，消息中间件不再直接和数据库打交道，完全由业务数据库来控制消息的生成、获取、发送及重试的策略。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175301615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

3、上面的方案都依赖于支持事务的数据库操作，具有一定的限制性。我们可以考虑把消息存储到本地磁盘。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175320851.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


这种方式的风险是，如果消息中间件和本地磁盘的数据都出问题时，消息就会丢失。所以，从业务数据上进行消息补发才是最彻底的容灾手段，因为这样才能保证只要业务数据在，就一定可以有办法恢复消息。

本地磁盘作为消息存储的方式有两种用法：

1. 作为一致性发送消息的解决方案的容灾手段。平时不工作，出现问题时才切换到该方式上；
2. 直接使用该方式工作，这样可以控制业务操作本身调用发送消息的接口处理时间，此外也有机会在业务应用于消息中间件之间做一些批处理的工作。

业务操作和发送消息一致性的方案所带来的两个限制：

- 需要确定要发送的消息的内容。
- 需要实现对业务的线程。也就是说为了支持反向流程的工作，业务应用必须能够根据反向流程中发回来的消息内容进行业务操作检查，确认消息所指向的业务操作的状态。

### 6.2.3 消息模型对消息接收的影响

在JMS中，有 Queue（点对点） 和 Topic（发布/订阅）两种模型。

6.2.3.1 JMS Queue 模型

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175612604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

消息会根据到达的顺序形成一个队列，所有连接到 JMS Queue 上的应用共同消费了所有的消息。如果Queue里面的消息被一个应用处理了，那么其他应用将收不到这个消息。消息从发送端发送出来时不能确定最终会被哪个应用消费，但是可以明确的是只有一个应用会去消费这条消息。JMS Queue 模型也被称为 Peer To Peer （PTP）方式。


#### 6.2.3.2 JMS Topic 模型

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175707604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


从发送消息的部分和 JMS Topic 内部的逻辑来看，JMS Queue 和 JMS Topic 是一样的，二者最大的区别在于消息接收的部分，在 Topic 模型中，接收消息的应用可以独立收到所有到达 Topic 的消息。JMS Topic 模型也被称为 Pub/Sub 方式。

每个应用可以与JMS 建立多个连接，每个 Connection 都有一个唯一的ClientId，每个连接都是独立获取消息，遵从上述描述。


#### 6.2.3.4 我们需要什么样的消息模型

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175756639.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175812115.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### 6.2.4 消息订阅者订阅消息的方式

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175839681.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175902471.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### 6.2.5 保证消息可靠性的做法

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615175953820.png)

中间有三个阶段需要保证可靠：消息发送者报消息发送到消息中间件，消息中间件把消息存入消息存储，消息中间件把消息投递给消息接收者。


#### 6.2.5.1 消息发送端可靠性的保证

消息发送者需要注意是否真正的发送成功，需要对异常情况进行处理。


#### 6.2.5.2 消息存储可靠性的保证

**1、实现基于文件的消息存储**

要完全实现基于文件的消息存储需要解决的问题还是比较多的，既要保证读写的吞吐量，又要保证存储的可靠性，应对断电、程序崩溃等问题。因此转向采用现有的数据库引擎的实现。

**2、采用数据库作为消息存储**

采用数据库作为消息存储时，数据库表的设计是相对比较复杂的，通常会采用冗余数据的方式来提高性能。同时需要考虑到的是，消费者集群出现问题，恢复正常后如何正确、快速的处理堆积的消息。

如果单机出现硬件问题，那么就需要考虑容灾方案：

- 单机的Raid
- 多机的数据同步。利用存储系统自身的机制完成。数据库的同步复制。
- 应用双写。主要是应对存储系统的数据复制有延迟的情况，但同时也让应用变的更复杂。

**3、基于双机内存的消息存储**

由于磁盘IO的原因，上述两种方法系统性能都会受到限制。我们可以使用混合方式进行存储的管理。用双机的内存来保证数据的可靠，正常情况下消息持久存储是不工作的，而基于内存来存储消息则能够提供很高的吞吐量。一旦一个机器出现故障，则停止另一台机器的数据写操作，并把当前数据罗盘。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615180205926.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


### 6.2.5.3 消息系统的扩容处理

**1、消息中间件自身如何扩容**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615180311774.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

通过server标识消息是来自于哪个消息中间件应用的。

**2、消息存储的扩容处理**

通过服务端主动调度安排投递的方式绕开了根据消息 ID 取消息这个动作，所以可以实现数据库存储的便利扩容。


#### 6.2.5.4 消息投递的可靠性保证

消息中间件需要显示地收到接收者确认消息处理完毕的信号才能删除消息。消息接收者不能在收到消息、业务没有处理完成时就去确认消息。同样要注意异常的处理，千万不要吃掉异常然后确认消息处理成功，这样会丢消息。

消息投递处理的优化：

- 多线程的方式处理。投递线程只负责投递消息，等待消息结果返回的工作放到另外的线程池中完成。更新数据库时可以通过 batch 来处理消息的更新、删除操作，从而提升性能。
- 如果一个应用中有多个订阅者订阅同样的消息，那么可以通过共享连接，消息只发送一次，传到单机生成多个实例的方式优化。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615180428455.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### 6.2.6 订阅者视角的消息重复的产生和应对

#### 6.2.6.1 消息重复的产生原因

第一类原因是消息发送端应用的消息重复发送：

- 消息中间件在接到消息后存储成功，但这时消息中间件出现问题，导致应用端没有收到消息发送成功的返回，从而进行重试产生了重复。
- 消息中间件负载高，消息成功写入消息存储，返回时响应超时。
- 由于网络出现问题，导致消息中间件无法响应，发送端重试发送，导致重复。

原因总结起来就是，在消息成功写入消息存储后，由于各种原因使得发送端没有收到“成功”的返回结果，并且又有重试机制，因而导致重复。

解决方法是，重试发送消息使用同样的消息ID，而不要在消息中间件端产生消息ID，可以避免这类情况发送。

第二类原因是消息到达了消息存储，由消息中间件向外投递时产生重复：

- 消息被投递到消息接收者应用进行处理，处理完后应用出现问题，消息中间件不知道消息处理结果，会再次投递。
- 由于网络问题，消息中间件不知道消息处理结果，会再次投递。
- 由于消息应用处理时间较长，消息中间件因为消息超时会再次投递。
- 消息中间件出现问题，没能收到消息结果并处理，会再次投递。
- 消息存储故障，没能更新投递状态，会再次投递。

原因总结起来就是，消息接收者成功处理完消息后，消息中间件不能及时更新投递状态造成的。

解决方法有两种：一种是采用分布式事务来解决，不过这种方式比较复杂，成本也高。另一种是消息接收者的消息处理是幂等操作。

幂等的含义是采用同样的输入多次调用处理函数，会得到同样的结果。

#### 6.2.6.2 JMS的消息确认方式与消息重复的关系

消息接收端对收到的消息确认方式：

- AUTO_ACKNOWLEDGE。自动确认，JMS客户端在收到消息后会自动确认。但是确认时可能消息还没处理或没有处理完成，显然这种方式对于消息投递来说是不可靠的。
- CLIENT_ACKNOWLEDGE。客户端自己确认。客户端需要自己调用 Message 接口的 acknowledge() 方法以进行消息接收成功确认。
- DUPS_OK_ACKNOWLEDGE。在消息接收方的消息处理函数执行结束后进行确认，一方面保证消息一定是处理结束后才进行确认，另一方面也不需要客户端主动调用 Message 接口确认。

从上面的确认方式，消息接收者对消息的接收会出现两种情况：

1. at least once（至少一次）。消息被投递给消息接收者至少一次，也可能多于一次。处理完消息后没有确认的情况下，会造成该现象。
2. at most once（至多一次）。消息被投递给消息接收者至多一次。接收到消息时立刻确认的情况下，会造成该现象。

6.2.7 消息投递的其他属性

**1、消息优先级**

**2、订阅者消息处理顺序和分级订阅**

**3、自定义属性**

**4、局部顺序。** 是指在众多的消息中，和某件事情相关的多条消息之间有顺序，而多个事情之间的消息则没有顺序。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615180916453.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### 6.2.8 保证顺序的消息队列的设计

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615180955775.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019061518101714.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


#### 6.2.8.1 单机多队列的问题和优化


如果单机的队列数量特别多，性能就会有明显的下降，原因是队列数量很多时，消息写入就接近于随机写了。一个改进措施是把发送到这台机器的消息数据进行顺序写入，然后再根据队列做一个索引，每个队列的索引是独立的，其中保存的只是相对于存储数据的物理队列的索引位置。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615181049449.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


#### 6.2.8.2 解决本地消息存储的可靠性

采用消息同步复制的方式解决可靠性的问题。

- 把单个的消息中间件机器变为主（Master）备（Slave）两个节点，Slave节点订阅Master节点上的所有消息，以进行消息的备份。需要注意的是这是异步操作，Slave订阅收到的消息总会比Master略少一些，存在丢消息的可能。
- 第二种是采用同步复制的方式，而非订阅的方式。Master收到消息后会主动写往Slave，并收到Slave的响应后才向消息发送者返回“成功”消息。

#### 6.2.8.3 如何支持队列的扩容

基本策略是让消息发送者知道应该把消息写入迁移的新队列中，也需要让消息订阅者知道，当前队列消费完数据后需要迁移到新队列去消费消息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615181148331.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### 6.2.9 Push和Pull方式的对比

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615181205834.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


****