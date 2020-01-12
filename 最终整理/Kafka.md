# Kafka 相关知识点

> [《Kafka面试题参考》](https://blog.csdn.net/linke1183982890/article/details/83303003)

## 一. Kafka 的元素，组成，架构

Kafka将消息以topic为单位进行归纳
将向Kafka topic发布消息的程序成为producers.
将预订topics并消费消息的程序成为consumer.
Kafka以集群的方式运行，可以由一个或多个服务组成，每个服务叫做一个broker.
producers通过网络将消息发送到Kafka集群，集群向消费者提供消息

## 二. 数据传输的事物定义有哪三种？

数据传输的事务定义通常有以下三种级别：
（1）最多一次: 消息不会被重复发送，最多被传输一次，但也有可能一次不传输
（2）最少一次: 消息不会被漏发送，最少被传输一次，但也有可能被重复传输.
（3）精确的一次（Exactly once）: 不会漏传输也不会重复传输,每个消息都传输被一次而且仅仅被传输一次，这是大家所期望的

## 三. Kafka判断一个节点是否还活着有那两个条件？

（1）节点必须可以维护和ZooKeeper的连接，Zookeeper通过心跳机制检查每个节点的连接
（2）如果节点是个follower,他必须能及时的同步leader的写操作，延时不能太久

## 四. producer是否直接将数据发送到broker的leader(主节点)？

producer直接将数据发送到broker的leader(主节点)，不需要在多个节点进行分发，为了帮助producer做到这点，所有的Kafka节点都可以及时的告知:哪些节点是活动的，目标topic目标分区的leader在哪。这样producer就可以直接将消息发送到目的地了

## 五. Kafka consumer是否可以消费指定分区消息？

Kafa consumer消费消息时，向broker发出"fetch"请求去消费特定分区的消息，consumer指定消息在日志中的偏移量（offset），就可以消费从这个位置开始的消息，customer拥有了offset的控制权，可以向后回滚去重新消费之前的消息，这是很有意义的

## 六. Kafka消息是采用Pull模式，还是Push模式？

- 推模式、拉模式：ActiveMQ 推模式可以将消息推送给服务，不需要服务端进行持续的轮询。但是推模式最大的弊端是，如果生产者的生产速度过快，有大量的消息堆积，消费者可能撑不住宕机。
- 反过来，拉模式由于自身通过轮询的方式拉取消息，消费速度相对可控制，所以可以实现一种削峰限流的作用。 

Kafka最初考虑的问题是，customer应该从brokes拉取消息还是brokers将消息推送到consumer，也就是pull还push。在这方面，Kafka遵循了一种大部分消息系统共同的传统的设计：producer将消息推送到broker，consumer从broker拉取消息

一些消息系统比如Scribe和Apache Flume采用了push模式，将消息推送到下游的consumer。这样做有好处也有坏处：由broker决定消息推送的速率，对于不同消费速率的consumer就不太好处理了。消息系统都致力于让consumer以最大的速率最快速的消费消息，但不幸的是，push模式下，当broker推送的速率远大于consumer消费的速率时，consumer恐怕就要崩溃了。最终Kafka还是选取了传统的pull模式

Pull模式的另外一个好处是consumer可以自主决定是否批量的从broker拉取数据。Push模式必须在不知道下游consumer消费能力和消费策略的情况下决定是立即推送每条消息还是缓存之后批量推送。如果为了避免consumer崩溃而采用较低的推送速率，将可能导致一次只推送较少的消息而造成浪费。Pull模式下，consumer就可以根据自己的消费能力去决定这些策略

Pull有个缺点是，如果broker没有可供消费的消息，将导致consumer不断在循环中轮询，直到新消息到t达。为了避免这点，Kafka有个参数可以让consumer阻塞知道新消息到达(当然也可以阻塞知道消息的数量达到某个特定的量这样就可以批量发

## 七. Kafka存储在硬盘上的消息格式是什么？

消息由一个固定长度的头部和可变长度的字节数组组成。头部包含了一个版本号和CRC32校验码。

- 消息长度: 4 bytes (value: 1+4+n)
- 版本号: 1 byte
- CRC校验码: 4 bytes
- 具体的消息: n bytes

## 八. Kafka 高效文件存储设计

> 参考地址：[《kafka的消息消费机制、consumer的负载均衡、文件存储机制》](https://blog.csdn.net/qq_37334135/article/details/78598289)

1. Kafka 把 topic 中一个 parition 大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
2. 通过索引信息可以快速定位 message 和确定 response 的最大大小。
3. 通过 index 元数据全部映射到内存，可以避免 segment 文件的 IO 磁盘操作。
4. 通过索引文件稀疏存储，可以大幅降低 index 文件元数据占用空间大小。

### 1. kafka 文件存储基本结构

在 Kafka 文件存储中，同一个 Topic 下有多个不同 partition，每个 partition 为一个目录。partition命名规则为**Topic名称 + 有序序号**。如果 partition 数量为 num，则第一个 partition 序号从 0 开始，序号最大值为 num - 1。  
例如，自己创建一个名为 orderMq 的 Topic：

```bash
[root@mini1 bin]# ./kafka-topics.sh --create --zookeeper mini1:2181 --replication-factor 2 --partitions 3 --topic orderMq
Created topic "orderMq".
```

orderMq 这个 topic 对应的 partitions 在三台机器上名称分别为：

```bash
drwxr-xr-x. 2 root root 4096 11月 21 22:25 orderMq-0
drwxr-xr-x. 2 root root 4096 11月 21 22:25 orderMq-2
```
```bash
drwxr-xr-x. 2 root root 4096 11月 14 18:45 orderMq-1
drwxr-xr-x. 2 root root 4096 11月 14 18:45 orderMq-2
```
```bash
drwxr-xr-x. 2 root root 4096 11月 21 22:25 orderMq-0
drwxr-xr-x. 2 root root 4096 11月 21 22:25 orderMq-1
```

> 注：重复的是副本，partition 名分别为 orderMq-0, orderMq-1, orderMq-2；

每个 partition（即每个目录）相当于一个巨型文件被平均分配到多个**大小相等**的 **segment（段）**数据文件中，但每个 segment **消息数量不一定相等**。这种特性方便旧 segment 文件快速被删除，默认保留7天的数据。例如在 orderMq-0 目录下：

```bash
[root@mini3 orderMq-0]# ll
-rw-r--r--. 1 root root 10485760 11月 21 22:31 00000000000000000000.index
-rw-r--r--. 1 root root      219 11月 22 05:22 00000000000000000000.log
```

由上可知，index 和 log 为后缀名的文件的合称，就是 segment 文件。每个 partition 只需要支持顺序读写就行了，segment 文件生命周期（什么时候创建，什么时候删除）由服务端配置参数决定。

### 2. segment 文件

Segment 文件由两大部分组成，分别为**索引文件 (index file)** 和**数据文件 (data file)**，这两个文件一一对应，成对出现。如下图所示：  

<img src="https://img-blog.csdn.net/20171121231650289?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzczMzQxMzU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" alt="Segment 文件列表" style="zoom:80%;" />

Segment 文件命名规则：partition 全局的**第一个 segment 从 0 开始，后续每个 segment 文件名为上一个 segment 文件最后一条消息的 offset 值**。数值最大为 64 位 long 大小，19 位数字字符长度，没有数字用 0 填充。
索引文件存储大量元数据，数据文件存储大量消息。**索引文件中元数据指向对应数据文件中message的物理偏移地址。**如下图所示：

<img src="https://img-blog.csdn.net/20171121231842315?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzczMzQxMzU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" alt="索引与数据的对应关系" style="zoom:80%;" />

上述图中 index 文件存储大量元数据，log 文件存储大量消息，index 文件中元数据指向对应 log 文件中消息的物理偏移地址。其中以 index 文件中元数据<code>3, 497</code>为例，依次在数据文件中表示第 3 个消息（当前 Segment 的第 3 个，全局 partition 表示第 368769 + 3 = 368772 个消息)，以及该消息在对应 log 文件中的物理偏移地址为 497。

### 3. kafka 查找消息

读取 offset=368776 的消息，需要通过下面两个步骤查找。

1. 第一步：查找segment file
- 以起始偏移量命名并排序这些文件，只要根据offset 二分查找文件列表，就可以快速定位到具体文件。
	- 00000000000000000000.index 表示最开始的文件，起始偏移量 (offset) 为 0
	- 00000000000000368769.index 的消息量起始偏移量为 368770 = 368769 + 1
	- 00000000000000737337.index 的起始偏移量为 737338=737337 + 1
- 其他后续文件依次类推。最后 offset=368776 时定位到 00000000000000368769.index 和对应log文件。
2. 第二步：通过 Segment 查找消息
	- 当 offset=368776 时，依次定位到 00000000000000368769.index 的元数据物理位置和 00000000000000368769.log 的物理偏移地址，然后再通过 00000000000000368769.log 顺序查找直到 offset=368776 为止。

## 九. Kafka 与传统消息系统之间有三个关键区别

(1).Kafka 持久化日志，这些日志可以被重复读取和无限期保留
(2).Kafka 是一个分布式系统：它以集群的方式运行，可以灵活伸缩，在内部通过复制数据提升容错能力和高可用性
(3).Kafka 支持实时的流式处理

## 十. Kafka创建Topic时如何将分区放置到不同的Broker中

- 副本因子不能大于 Broker 的个数；
- 第一个分区（编号为0）的第一个副本放置位置是随机从 brokerList 选择的；
- 其他分区的第一个副本放置位置相对于第0个分区依次往后移。也就是如果我们有5个 Broker，5个分区，假设第一个分区放在第四个 Broker 上，那么第二个分区将会放在第五个 Broker 上；第三个分区将会放在第一个 Broker 上；第四个分区将会放在第二个 Broker 上，依次类推；
- 剩余的副本相对于第一个副本放置位置其实是由 nextReplicaShift 决定的，而这个数也是随机产生的

## 十一. Kafka新建的分区会在哪个目录下创建

在启动 Kafka 集群之前，我们需要配置好 log.dirs 参数，其值是 Kafka 数据的存放目录，这个参数可以配置多个目录，目录之间使用逗号分隔，通常这些目录是分布在不同的磁盘上用于提高读写性能。  
当然我们也可以配置 log.dir 参数，含义一样。只需要设置其中一个即可。  
如果 log.dirs 参数只配置了一个目录，那么分配到各个 Broker 上的分区肯定只能在这个目录下创建文件夹用于存放数据。  
但是如果 log.dirs 参数配置了多个目录，那么 Kafka 会在哪个文件夹中创建分区目录呢？答案是：Kafka 会在含有分区目录最少的文件夹中创建新的分区目录，分区目录名为 Topic名+分区ID。注意，是分区文件夹总数最少的目录，而不是磁盘使用量最少的目录！也就是说，如果你给 log.dirs 参数新增了一个新的磁盘，新的分区目录肯定是先在这个新的磁盘上创建直到这个新的磁盘目录拥有的分区目录不是最少为止。

## 十二. kafka的ack机制
request.required.acks有三个值 0 1 -1  
0:生产者不会等待broker的ack，这个延迟最低但是存储的保证最弱当server挂掉的时候就会丢数据  
1：服务端会等待ack值 leader副本确认接收到消息后发送ack但是如果leader挂掉后他不确保是否复制完成新leader也会导致数据丢失  
-1：同样在1的基础上 服务端会等所有的follower的副本受到数据后才会受到leader发出的ack，这样数据不会丢失

## 十三. Kafka的消费者如何消费数据
消费者每次消费数据的时候，消费者都会记录消费的物理偏移量（offset）的位置  
等到下次消费时，他会接着上次位置继续消费

## 十四. 消费者负载均衡策略
一个消费者组中的一个分片对应一个消费者成员，他能保证每个消费者成员都能访问，如果组中成员太多会有空闲的成员

## 十五. 数据有序
一个消费者组里它的内部是有序的
消费者组与消费者组之间是无序的

## 十六. kafka生产数据时数据的分组策略
生产者决定数据产生到集群的哪个partition中
每一条消息都是以（key，value）格式
Key是由生产者发送数据传入
所以生产者（key）决定了数据产生到集群的哪个partition

## 十七. Kafka 消息积压

> [《07、完了！生产事故！几百万消息在消息队列里积压了几个小时！》](https://blog.csdn.net/A_BlackMoon/article/details/85197943)

**问题：**如何解决消息队列的延时以及过期失效问题？消息队列满了以后该怎么处理？有几百万消息持续积压几小时，说说怎么解决？

分析该问题，主要需要分析该场景。首先看出现该问题的原因：

- 是否是消费者问题？
- 是否是消息队列集群所在节点的问题，比如磁盘满了？

### 17.1 问题 1：服务端故障

**假设服务端出了故障，大量消息再 MQ 里积压**：  

解决方法一：修复消费者的问题，恢复消费速度，然后等待消费完毕。（当然这种方法不能在面试的时候讲）  
**解决方法二**：临时紧急扩容；

1. 先找到并修复 consumer 的问题，恢复消费速度，**然后停掉现有所有 consumer**。
2. 新建一个 topic，具有原先 10 倍的 partition，临时建立好原先 10 到 20 倍的队列数量；
3. 写一个临时分发数据的 consumer 程序，**这个程序部署上去，消费积压的数据**。消费之后不做耗时的处理，<font color=red>**直接均匀轮询，写入临时建好 10 倍数量的队列**</font>。
4. 临时征用 10 倍的机器，部署 consumer，每一批 consumer 消费一个临时队列的数据。
	- 这种做法相当于**<font color=red>临时将队列资源、消费者资源扩大十倍</font>**，以十倍的速度来消费数据；
5. 快速消费完积压的数据之后，恢复原先的架构，重新用修复后的原 consumer 机器来消费消息。

## 十八. 消息中间件的高可用性

实现高可用性的方式一般都是**进行 replication**。对于 kafka，如果没有提供高可用性机制，一旦一个或多个 Broker 宕机，则宕机期间其上所有 Partition 都无法继续提供服务。若该 Broker永远不能再恢复，那么所有的数据也就将丢失，这是不可容忍的。所以 kafka 高可用性的设计也是进行 Replication。  
**Replica 的分布**：为了尽量做好负载均衡和容错能力，需要将同一个 Partition 的 Replica 尽量分散到不同的机器。  
**Replica 的同步**：当有很多 Replica 的时候，一般来说，对于这种情况有两个处理方法：

- **同步复制**：当 producer 向所有的 Replica 写入成功消息后才返回。一致性得到保障，但是延迟太高，吞吐率降低。
- **异步复制**：所有的 Replica 选取一个一个 leader，producer 向 leader 写入成功即返回（即生产者参数 acks = 1）。leader 负责将消息同步给其他的所有 Replica。但是消息同步一致性得不到保证，但是保证了快速的响应。

而 kafka 选取了一个折中的方式：**ISR (in-sync replicas)**。producer 每次发送消息，将消息发送给 leader，leader 将消息同步给他“信任”的“小弟们”就算成功，巧妙的均衡了确保数据不丢失以及吞吐率。具体步骤：

1. 在所有的 Replica 中，**leader 会维护一个与其基本保持同步的 Replica 列表**，该列表称为**ISR (in-sync Replica)**；每个 Partition 都会有一个 ISR，而且是由 leader 动态维护。
2. 如果一个 replica 落后 leader 太多，leader 会将其剔除。如果另外的 replica 跟上脚步，leader 会将其加入。
3. **同步**：leader 向 ISR 中的所有 replica 同步消息，当收到所有 ISR 中 replica 的 ack 之后，leader 才 commit。
4. **异步**：收到同步消息的 ISR 中的 replica，异步将消息同步给 ISR 集合外的 replica。


## 十九. 避免消息中间件故障后引发系统整体故障

如果所有的 replica 都不工作了，有两种解决方式：

1. 等待 ISR 中任一个 Replica 恢复，并选取他为 leader；
	- 等待时间较长，降低了可用性；若 ISR 中的所有 Replica 都无法恢复或者数据丢失，则该 partition 将永不可用
2. 选择第一个回复的 Replica 为新的 leader，无论他是否在 ISR 中；
	- 所选 leader 可能并未包含已被之前 leader commit 的消息，因此会造成数据丢失；
	- 可用性较高；

## 二十. Kafka 消息投递

> 参考地址：[《Kafka消息投递语义-消息不丢失，不重复，不丢不重》](https://www.cnblogs.com/3gods/p/7530828.html)

- **问题 1**：使用 Kafka 的时候，你们怎么保证投递出去的消息一定不会丢失？
- **问题 2**：你们怎么保证投递出去的消息只有一条且仅仅一条，不会出现重复的数据？

### 20.1 Kafka 消息投递语义

kafka 有三种消息投递语义：

- **At most Once**：最多一次；消息不会重复，但可能丢失；
- **At least Once**：最少一次；消息不会丢失，但可能重复；
- <font color=red>**Exactly Once**</font>：最佳情况，只且消费一次；消息不会重复，也不会丢失；

整体的消息投递语义由**生产者、消费者**两端同时保证。

#### 20.1.1 Producer 生产者端

Producer 端保证消息投递重复性，是通过 **Producer 的 <font color=red>acks</font> 参数**与 **Broker 端的 <font color=red>min.insync.replicas</font> 参数**决定的。  

Producer 端的 **acks 参数**值信息如下：

- **acks = 0**：不等待任何响应的发送消息；
- **acks = 1**：**leader 分片写消息成功**，就返回响应给生产者；
- <font color=red>**acks = -1(all)**</font>：要求 ISR 集合至少两个 Replica，而且必须全部 Replica 都写入成功，才返回响应给 Producer；
	- 无论 ISR 少于两个 Replica，或者不是全部 Replica 写入成功，都会抛出异常；

前面 Producer 的 acks = 1 可以保证写入 Leader 副本，对大部分情况没有问题。但如果刚刚一条消息写入 Leader，还没有把这条消息同步给其他 Replica，Leader 就挂了，那么这条消息也就丢失了。所以如果保证消息的完全投递，还是需要令 acks = all；

#### 20.1.2 Broker 节点端

首先上面说到，为了配合 Producer acks 参数为 all，需要令 Broker 端参数 min.insync.replicas = 2；  
min.insync.replicas 参数是用来配合 Producer acks 参数的。因为如果 acks 设置为 all，但某个 Topic 只有 leader 一个 Replica（或者某个 Kafka 集群中由于同步很慢，导致所有 follower 全部被剔除 ISR 集合），这样 acks = -1 就演变成了 acks = 1。  
所以需要 Broker 端设置 min.insync.replicas 参数：当参数值为 2 时，如果副本数小于 2 个，会抛出异常。

> 注：然而在笔者的使用环境中，订阅是 Kafka 主要的使用场景之一，方式是对于想要订阅的某个 Topic，每个用户创建并独享一个不会重复的消费组。所以这样的情况下，环境下的 min.insync.replicas 只能等于 1；

除此之外，broker 端还有一个需要注意的参数 **unclean.leader.election.enable**。该参数为 true 的时候，表示在 leader 下线的时候，可以从非 ISR 集合中选举出新的 Leader。这样的话可能会造成数据的丢失。所以如果需要在 Broker 端的 unclean.leader.election.enable 设置为 false。

#### 20.1.3 Consumer 消费者端

Consumer 端比较麻烦，原因是需要考虑到某个 Consumer 宕机后，同 Consumer Group 会发生负载均衡，同 Group 其他的 Consumer 会重新接管并继续消费。

假设两种场景：

**第一个场景，Consumer 先提交 offset，再处理消息**。代码如下：

```java
List<String> messages = consumer.poll();
consumer.commitOffset();
processMsg(messages);
```

这种情形下，提交 offset 成功，但处理消息失败，同时当前 Consumer 宕机，这时候发生负载均衡，其他 Consumer 从**已经提交的 offset** 之后继续消费。这样的情况保证了 **<font color=red>at most once</font>** 的消费语义，当然也可能会丢消息。

**第二个场景，Consumer 先处理消息，再提交 offset**。代码如下：

```java
List<String> messages = consumer.poll();
processMsg(messages);
consumer.commitOffset();
```

这种情形下，消息处理成功，提交 offset 失败，同时当前 Consumer 宕机，这时候发生负载均衡，其他 Consumer 依旧从同样的 offset 拉取消息消费。这样的情况保证了 **<font color=red>at least once</font>** 的消费语义，可能会重复消费消息。

上述机制的保证都不是直接一个配置可以解决的，而是 Consumer 端代码的处理先后顺序问题完成的。

## 其他提问

- 说说你们公司线上生产环境用的是什么消息中间件？Kafka。
- 那你们线上系统是有哪些技术挑战，为什么必须要在系统里引入消息中间件？
	- 对某些数据进行增删改后，需要通知其他业务，所以使用到了订阅模式；
	- 在并发量较大的情况下，需要使用消息队列，对流入后台的请求进行缓冲；
- 你们的消息中间件技术选型为什么是 RabbitMQ？为什么不用 RocketMQ 或者是 Kafka？技术选型的依据是什么？
	- 
- 如果消费了重复的消息怎么保证数据的准确性？
- 你们线上业务用消息中间件的时候，是否需要保证消息的顺序性？如果不需要保证消息顺序是为什么？假如我有一个场景要保证消息的顺序，你们应该如何保证？
	- 业务修改，数据上报的场景，先删后增；但如果并发量大，可能会出现问题，变成删删增增，结果不正确。此外，存入数据库中的数据应该由收到数据的顺序决定，所以用到了 Kafka 消息队列的顺序性；
	- 构建 Topic 时建了 3 个 Partition，在投递消息的时候，按照单位 ID 进行 Hash 计算，相同 ID 的单位保证是进入同一个 Partition，这样就进入了相同的队列，也就保证了消息队列的顺序性。
- 你们用的是Kafka？那你说说Kafka的底层架构原理，磁盘上数据如何存储的，整体分布式架构是如何实现的，如何保证数据的高容错性，零拷贝等技术是如何运用的，高吞吐量下如何优化生产者和消费者的性能？那你看过Kafka的源码没有，说说你对Kafka源码的理解？
- 你们用的是RocketMQ？RocketMQ很大的一个特点是对分布式事务的支持，你说说他在分布式事务支持这块机制的底层原理？RocketMQ的源码看过么，聊聊你对RocketMQ源码的理解？
- 如果让你来动手实现一个分布式消息中间件，整体架构你会如何设计实现？
