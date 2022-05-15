# 概念和架构
## 消息
Pulsar 采用 [发布-订阅](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)的设计模式(简称 pub-sub)

### Producers
producer 发送消息可以同步或异步发送，同步发送需要等待 broker 的ack ，异步发送会将消息放于阻塞队列中，并立即返回，客户端将在后台将消息发送给 broker。

### 消费者

消息确认分为单独确认模式和累计确认模式。单独确认模式需要想 broker 确认每一条消息。累计确认模式只需确认消费的最后一条消息，所有之前的消息都不会被再次发送给那个消费者。累计确认模式不能用在 shared 订阅模式。

如果一个消息没有被成功消费，而你想让 Broker 自动重新交付这个消息，那么你可以为未被认可的消息启用自动重新交付机制。 在启用自动重新交付的情况下，客户端跟踪整个acktimeout时间范围内的未确认的消息，并在指定确认超时时向代理发送重新交付未确认的消息请求。


### 消息保留和过期
有留存策略的消息，即使被确认后也不会被删除。消息超过 TTL 后，即使没有被确认，也会被删除。



## 架构
![image.png](https://cdn.nlark.com/yuque/0/2022/png/12431514/1646038075076-452ba28f-2e37-4f5f-8108-45a014705688.png#clientId=ue068f40e-ec60-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=421&id=ud2480503&margin=%5Bobject%20Object%5D&name=image.png&originHeight=514&originWidth=789&originalType=binary&ratio=1&rotation=0&showTitle=false&size=329996&status=done&style=none&taskId=uf1ea5d93-d081-4ce9-b32d-b2fe6a316e4&title=&width=646)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/12431514/1646042481325-461260cc-d627-45ad-bfe9-af2ba787b1c9.png#clientId=ue068f40e-ec60-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=347&id=u329b2c29&margin=%5Bobject%20Object%5D&name=image.png&originHeight=470&originWidth=796&originalType=binary&ratio=1&rotation=0&showTitle=false&size=246948&status=done&style=none&taskId=u3409cc86-7981-4dde-9cdd-41fbd01607f&title=&width=587)
### Ledgers
Ledger是一个只追加的数据结构，并且只有一个写入器，这个写入器负责多个BookKeeper存储节点（就是Bookies）的写入。 Ledger的条目会被复制到多个bookies。

### 服务发现
每个 topic 只能被一个 broker 处理。客户端发出的读取，更新或删除主题的初始请求将发送给可能不是处理该主题的 broker 。 如果这个 broker 不能处理该主题的请求，broker 将会把该请求重定向到可以处理主题请求的 broker。


## Pulsar Clients

#### Reader 接口

Reader 读取 topic 消息时，可以指定 message ID 开始消费消息。

Reader 接口对流处理系统中，需要用到 effectively-once(仅仅一次) 语义的场景是很有帮助的。 Pulsar能够将主题的消息进行重放，并从重放后的位置开始读取消息，是满足流处理的场景的重要基础。 Reader 接口为 Pulsar 客户端在 Topic 内提供了一种能“手动管理起始位置”的底层抽象。

## 跨机房复制





## 多租户

租户可以跨集群分布，每个租户都可以有单独的[认证和授权](https://pulsar.apache.org/docs/zh-CN/security-overview)机制。 租户也是存储配额、[消息 TTL](https://pulsar.apache.org/docs/zh-CN/cookbooks-retention-expiry#time-to-live-ttl) 和隔离策略的管理单元。


## 认证和授权

Pulsar 支持可插拔的[认证](https://pulsar.apache.org/docs/zh-CN/security-overview)机制， Pulsar Proxy 或者 Pulsar Broker 都支持该机制。 Pulsar 也支持可插拔的[鉴权](https://pulsar.apache.org/docs/zh-CN/security-authorization)机制。 认证和鉴权机制一起保证了客户端对于主题，命名空间和租户的访问权限。


## topic 压缩
当[通过命令行](https://pulsar.apache.org/docs/zh-CN/cookbooks-compaction)触发topic压缩，Pulsar将会从头到尾迭代整个topic。 对于它碰到的每个key，压缩程序将会只保留这个key最近的事件。


## Proxy support with SNI routing





## 配置 advertised 监听器








# Pulsar schema

### 理解 Schema




























# Pulsar functions























# Pulsar IO



















# Pulsar SQL















# 层级存储

























# 事务






















# Kubernetes(Helm)




















# 部署






















# 系统管理























# 安全
## Pulsar security overview



















# 性能

















# 客户端库



























# Admin API
## Pulsar admin interface

HTTP调用发起方需要能够处理307 Temporary Redirect返回。



















# 适配器





















