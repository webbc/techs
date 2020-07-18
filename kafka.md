# Kafka

## 简介

分布式消息引擎系统、也是一个分布式流处理平台

## MQ作用

削峰、解耦

## 特性

- 分布式
- 消息引擎系统（消息中间件）
- 小型流处理平台

## 基本概念

- 消息：Record

Kafka就是处理消息的

- 主题：Topic

存储消息的逻辑容器，一般和业务挂钩

- 分区：Parition

消息序列，有序的；每个主题下可以有多个分区

- 消息位移：Offset

分区内消息的位置，递增的一个值

- 副本：Replica

Kafka 中同一条消息能够被拷贝到多个地方以提供数据冗余，这些地方
就是所谓的副本。副本还分为领导者副本和追随者副本，各自有不同的角色划分。副本是在分区层级下的，即每个分区可配置多个副本实现高可用

- 生产者：Producer

向Topic生产消息的程序

- 消费者：Consumer

从Topic消费消息的程序

- 消费者位移：Consumer Offset

消费者消费消息的位置，每个消费者有自己的offset

- 消费者组：Consumer Group

多个消费实例组成的消费者组，同时消费多个分区数据

- 重平衡：Rebalance

同组中如果有消费实例挂掉，会把这个实例订阅的Parition分配给其他消费实例

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka.png)

### Kafka种类

- Apache Kafka（社区版Kafka）
- Confluent Kafka（商业版Kafka，提供了更多高级功能，比如跨数据中心备份、注册中心和集群监控），librdkafka也是这个公司的人开发的
- CDH/HDP Kafka（云平台Kafka）

### Kafka版本

目前最新版本是2.5.0，从0.7、0.8、0.9、0.10、0.11、1.0、2.0发展到现在。

## 快速开始

### 安装

```
wget http://xxx.tgz
tar -xzf kafka_2.12-2.5.0.tgz
cd kafka_2.12-2.5.0
```

### 启动

```
#先启动zookeeper
bin/zookeeper-server-start.sh config/zookeeper.properties

#再启动Kafka
bin/kafka-server-start.sh config/server.properties
```

### 创建Topic

```
# 创建
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test

# 查看topic列表，是否创建成功
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

### 启动生产者

```
 bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test
```

### 启动消费者

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```

## 少三2使用

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/ss2_kafka.png)

### Topic

只有1个，ngame2_log，存储游戏数据日志，副本2个

### 生产者

Broom进程

### 消费者

- gmscribe

消费数据，将需要的数据写入MySQL，然后第二日根据昨日数据计算出运营数据，提供给GM后台查询

- scribe

消费数据，将需要的数据保存成文件的形式，提供给BA。主要生成KPI数据、生态行为日志

- snp_scribe

消费数据，将需要的数据保存到MySQL，0点根据当前状态从MySQL中读取数据，写入文件，提供给BA。主要生成快照数据

- timedb_scribe

消费数据，将需要的数据保存成文件的形式，提供给BA，BA入库后，提供接口给GM后台查询玩家行为日志

### Kafka版本

kafka_2.11-1.1.1

### Kafka客户端

- Producer 

用的golang写的sarama库

- Consumer

用的confluent-kafka-go库，这个是用cgo调libkafkard库，这个是kafka官网推荐的golang客户端


## 集群部署方案

- 机器

需要多台做高可用，不然就是个伪集群

- 操作系统

因为Kafka是由Scala和Java写的，编译出来的是.class文件，JVM是跨平台的，所以Windows和Linux都可以用。推荐用Linux，可以用到ZeroCopy。

- 磁盘

Kafka的消息数据是存在磁盘的，使用磁盘大都是顺序读写，一定程度上规避了随机读写操作慢的问题，所以用机械磁盘就够了。

磁盘容量方面需要考虑：每天的消息数、消息的平均大小、保存时间、备份数、是否启用压缩

假设每天1亿条1KB大小的消息，保存两份且留存两周的时间，那么总的空间大小就等于 1亿 * 1KB * 2 / 1000 / 1000 = 200GB。一般情况下Kafka 集群除了消息数据还有其他类型的数据，比如索引数据等，也需要为这些数据预留出10%的磁盘空间，因此总的存储容量就是220GB。如果要保存两周，那么整体容量即为220GB * 14，大约 3TB。 Kafka支持数据的压缩，假设压缩比是0.75，那么最后你需要规划的存储空间就是 0.75 * 3 = 2.25TB。

- 内存

Kafka重度依赖底层OS提供的page cache功能。当有写操作时，OS只是将数据写入到page cache，同时标记page属性为dirty。刷盘的操作都交给操作系统的flush线程来做。当读操作发生时，先从page cache中查找，找不到才会去磁盘中找。

- 带宽

假设是1Gbps的单机带宽，机器上只部署Kafka，如果超过70%可能就会出现丢包，也就是说单台Kafka服务器最多能使用到700M带宽资源，肯定不能全部用完，最好能预留2/3的资源，即单机服务器700 / 3 = 240Mb。如果1小时需要处理1TB的数据，每秒需要处理 1TB / 3600 * 8 = 2336Mb。2336 / 240 = 10台机器。

### 重要参数

#### Broker级别参数

- broker.id：标识broker的唯一ID，多台broker不要重复
- log.dirs：指定Kafka消息持久化的目录，需要注意磁盘挂载，可以指定多个目录，多个目录用逗号分隔，然后Kafka会负载均衡的将分区写入到多个目录中，默认值是/tmp/kafka-logs
- zookeeper.connect：指定Zookeeper的集群地址，多个地址用csv格式，例如：10.24.16.34:2181,10.24.16.31:2181,10.24.16.28:2181，无默认值
- listeners：broker监听的csv列表，格式：[协议]://[主机名]:[端口],[协议]://[主机名]:[端口]，这个参数主要是给客户端连接用的，Kafka目前支持的协议有PLAINTEXT、SSL和SASL_SSL。
- unclean.leader.election.enable：是否开启unclean leader选举，ISR中的副本都有资格成为新的leader，但是如果ISR为空且leader宕机了，如果这个设置值为true，则代表从非ISR副本中选择一个副本成为leader，但是这样会导致数据丢失，因为非ISR的副本的含义就是和leader副本不同步的副本，这样就会导致发给原先leader副本的消息，没有同步到新的leader副本（非ISR副本中），导致数据丢失，这个参数的默认值是false。
- delete.topic.enable：是否允许Kafka删除Topic
- log.retention.{hours|minutes|ms}：控制消息数据保留的时间，如果同时设置hours|minutes|ms，优先ms，其次minutes，最后hours，默认的保存时间是7天，表示Kafka只会保存最近7天的消息数据。
- log.retention.bytes：空间维度保留数据策略，分区消息日志文件如果大于这个值，就会自动清理该分区过期的日志段文件，默认值是-1，表示不会根据消息日志文件总大小来删除日志。
- min.insync.replicas：该参数需要和producer端的acks参数配合使用，当producer的acks配置成-1，表示producer端寻求最高等级的持久化保证，只有在设置了acks为-1的情况下，min.insync.replicas才有用，表示指定broker端必须成功响应的clients消息发送的最少副本数。如果broker无法满足，那么clients发送的消息不会被认为成功提交。
- num.network.threads：处理转发网络请求的线程数，默认值是3。
- num.io.threads：真正处理网络请求的线程数，默认值是8，KafkaBroker会默认启动8个线程以轮询的方式不断监听转发过来的网络请求并实时处理。
- message.max.bytes：KafkaBroker能接收的最大消息大小，默认值是977K，如果消息比较大，需要注意这个参数配置
- auto.create.topics.enable：是否自动创建topic，这个最好设置成false。避免创建一些莫名其妙的topic，如果在启动生产者的时候不小心打错了Topic，那就自动创建了一个错误的Topic。

#### Topic级别参数

因为Broker端参数是对所有Topic而言的，可以给不同的Topic设置不同的参数。

- delete.rentention.ms：每个Topic可以设置自己的日志留存时间来覆盖全局默认值
- max.message.bytes：覆盖全局的max.message.bytes参数，每个Topic可以设置不同的最大消息大小
- rentention.bytes：覆盖全局的log.rentention.bytes，每个Topic设置不同的日志留存空间

#### JVM参数

Kafka推荐使用最新版本的jdk版本。
JVM堆大小设置成6G
GC如果是jdk8以上就用g1gc，java7的话cpu资源充足选CMS，否则使用吞吐量收集器  

#### 操作系统参数

- 文件描述符限制：ulimit -n 1000000，小了容易报“Too many open files”
- 关闭swap
- 设置更长时间的flush时间：Kafka依赖操作系统的PageCache，刷盘操作也是操作系统来做的，这个时间默认值是5s。
- 最好使用Ext4或XFS文件系统

## Producer

发送一个消息首先需要确定往哪个Topic的那个分区发送。Kafka有个分区器，如果消息指定了key，那个会用key的hash值来选择目标分区，如果没有指定key，则会使用轮询策略。当然也可以直接指定分区。

### 分区策略

- 轮询策略

假设一个topic有3个partition，第一条消息就写到partition0，第二条消息就写到partition1，第三条消息就写到partition2......。优点是分区消息量很均衡，

- 随机策略

消息被随机写到任一分区，本质上是想让消息均匀的达打到不同的分区，不如直接使用轮询策略

- 消息键保序策略

Kafka可以为每条消息定义消息key，消息key一致的消息可以写到相同的分区

- 直接指定分区策略


sarama客户端提供了这4种分区策略

### 重要参数

- acks

指定在发送响应前，leader broker必须要确保已成功写入该消息的副本数，当前取值有3个：0、1、-1。

acks = 0，表示producer发送消息后完全可以不用管leader broker端的处理结果，就可以开启下一轮的发送了，这种情况就相当于是发送之后就不管了，是不是失败也不知道。

acks = -1

表示发送消息，leader broker不仅会将消息写入本地日志，同时还会等待ISR中所有其他副本都成功写入他们的本地日志，才会回给producer信息。只要ISR中的所有副本都正常工作，那么发送的消息就不会丢失。

acks = 1

是0和-1的折中方案，默认值就是这个，表示leader broker将消息写入本地日志，就可以返回给producer了，不需要等待ISR中的其他副本写入该消息，只要leader broker没问题，那么发送的消息就不会丢失。

- compression

表示produer端是否启动压缩，默认值是none不启用，目前Kafka支持Gzip、Snappy和LZ4压缩方式

- retry

写入请求可能由于某种原因产生瞬断导致消息发送失败，这种情况是通常可以自动恢复的；如果把错误封装到回调函数中返还给producer的话，producer也只能重试。所以producer内部实现了自动重试，只需要设置retry参数，可以设置重试的最大次数，

还有个参数retry.backoff参数表示2个重试之间的间隔时间，默认值是100毫秒，Kafka也考虑了不能不停发重试和重试间隔，这点对我们写业务也有帮助。

重试可能会带来的问题：如果是broker已经写入成功了，只是没有响应给producer，那么这条消息就会被重复发送，那么这个需要Consumer那边做去重处理了。

- maxMessageBytes

设置发送一条消息最大的大小，Sarama客户端默认值是1000000B=977KB，如果不够需要设大一点。

- timeout

当producer发送给broker之后，broker需要在规定时间内返回给producer，这段时间就是控制这个的，如果到这个时间了还没有返回的话，producer就认为超时了，Sarama客户端默认值是10秒。

### 消息压缩

压缩：Producer端和Broker端
解压缩：Consumer端

Producer端和Broker不要使用不同的压缩算法，不然需要先解压缩Producer端发来的消息，在使用新的压缩方式继续压缩，压缩还是很占CPU资源的。消息中会带有压缩方式，Consumer只要获取到消息就可以解压缩了。
最佳实践：Producer端压缩、Broker端保持、Consumer端解压缩

目前支持的压缩方式：Gzip、Snappy和LZ4

### 无丢失消息

Kafka只对已提交的消息做有限度的持久化保证，这个之前已经说过了。

生产者丢失消息
不要直接使用send方法，而是要使用异步的方法，在回调中做异常处理

消费者端丢失消息
 - 大部分原因是offset设置的不合理，最好的方式就是先消费消息，再设置offset
 - 如果是多线程异步处理消费消息，Consumer程序不要开启自动提交位移，而是要应用程序手动提交位移

保证消息不丢失最佳实践

- 1、需要处理send之后的返回的错误信息。

- 2、设置acks = all。acks是Producer的一个参数，代表了你对“已提交”消息的定义。如果设置成all，则表明ISR中的所有副本Broker都要接收到消息，该消息才算是“已提交”。这是最高等级的“已提交”定义。

- 3、设置retry为一个较大的值。这里的retry同样是Producer的参数，对应前面提到的Producer自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了retries > 0的 Producer能够自动重试消息发送，避免消息丢失

- 4、设置unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么
它一旦成为新的Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false，
即不允许这种情况的发生。

- 5、设置 replication.factor >= 3。这也是Broker 端的参数。最好将消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余

- 6、设置 min.insync.replicas > 1，默认值是1，这依然是 Broker 端参数，控制的是消息至少要被写入到多少个副本才算是“已提交”，这个参数只有在producer设置acks=-1时才有效。这个值不能大于ISR中副本的集合，不然producer就提交不成功消息了，那么整个消息系统就出问题了。

- 7、确保 replication.factor>min.insync.replicas。如果两者相等，那么只要有一个副本挂
机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要
在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas+1

- 8、 确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设
置成 false，并采用手动提交位移的方式。

### 消息投递保证

- 最多一次（At most once） 消息可能会丢，但绝不会重复传输
- 至少一次（At least one） 消息绝不会丢，但可能会重复传输
- 精确一次（Exactly once）每条消息肯定会被传输一次且仅传输一次，很多时候这是用户所想要的。　　

实现最多一次就是发送之后就不管了，也不重试，但是有可能Broker并没有写入成功，所以可能会丢失消息，这种是不能接受的。

有这样的情况，Producer发出去了，但是Broker写入成功后由于某种原因没有返回响应信息。如果Broker有重试，那么消息就会重复，这就是至少一次。

#### 精确一次

Kafka已经在0.11版本支持精确一次语义。主要是幂等、事务上。

- 幂等producer

只需要设置enable.idempotence=true即可，producer就是幂等producer了，Kafka自动帮你做消息去重，但是只能实现单个分区的幂等性。如果要实现多分区的原子性，需要用事务型，同时也是单会话的，如果producer宕机重启就不保证是幂等的了。

幂等Producer在初始化的时候会生成一个ID，简称PID。对于每个PID，该Producer发送到每个Partition的数据都有对应的Sequence Number序号，从0开始单调递增。Broker端在缓存中保存了这 Sequence Numbler，对于接收的每条消息，如果其序号比Broker缓存中序号大于1则接受它，否则将其丢弃，这样就解决了消息重复提交了。

- 事务producer

首先enable.idempotence=true，其次设置个transctional.id事务ID。多次发送消息可以封装成一个原子操作，要么都成功，要么都失败。

## Consumer

### ConsumerGroup

Consumer Group 是 Kafka 提供的可扩展且具有容错性的消费者机制
一个ConsumerGroup可以有多个消费者实例，每个组都有个groupId，组内所有consumer一起订阅主题内的所有分区，每个分区只能被1个consumer实例消费。

- ConsumerGroup下可以有1个或多个Consumer实例，实例可以是进程或线程
- groupId是字符串，唯一标识某个ConsumerGroup
- ConsumerGroup下的所有实例订阅的主题的单个分区，只能分配给组内的某个Consumer实例消费。这个分区也可以被其他的Group消费。

通过ConsumerGroup机制可以实现消息引擎系统2大模型：点对点模型、发布/订阅模型。如果只有1个consumerGroup就是点对点模型，如果有多个ConsumerGroup就是发布/订阅模型。

Consumer个数最好设置成订阅的Topic分区总数。

假设3个Topic，分别是a,b,c。每个Topic的对应的分区数是1,2,3。那么最好使用6个实例来消费。
如果启动3个实例，就会有1个实例消费2个分区，如果启动8个实例，就会有2个实例没在干活。

### Offset

每个消费实例消费指定的分区，都会有一个位移来表示当前消费的进度，其实可以理解成map，key是分区，value是offset。
老版本的KafkaComsumer的Offset是保存在ZooKeeper中的。保存在ZooKeeper中是为了让Kafka无状态。不过由于写Offset太过频繁，经常写ZooKeeper会拖慢整个Kafka集群的性能。新版本0.9的KafkaConsumer保存在了内部主题 __consumer_offsets。

### 内部主题

Consumer将位移数据作为一条普通的Kafka消息，提交到__consumer_offsets这个Topic中，主要作用是保存Kafka消费者的位移信息，所以不要删除这个主题的数据，也不要尝试往这个主题写数据，全部交给Kafka处理吧。

消息Key就是<groupId、主题名、分区号>，Value就是offset。分区数默认是50，副本增长因子默认是3。

### Rebalance

Rebalance本质上是一种协议规则，规定了一个ConsumerGroup下的所有Consumer如何达到一致，来分配订阅Topic下的分区。比如某个ConsumerGroup下有20个Consumer 实例，它订阅了一个具有 100个分区的Topic。Kafka平均会为每个Consumer分配5个分区。这个分配的过程就叫Rebalance。

条件：

- 1、ConsumerGroup中有新的Consumer加入，或者有Consumer奔溃
- 2、由于可以使用正则来消费指定的Topic，如果有新的符合规则的Topic可以被ConsumerGroup消费，会触发Rebalance
- 3、订阅的Topic下的分区数变化，增加Topic下的分区。

Rebalance过程中所有消费实例是停止消费的状态。

监听器：发生Rebalance的时候Consumer可以使用监听器来感知partition的变化，必须要指定ConsumerGroup才行。主要就是监听2个事件，Rebalance前会先发送OnRevoke事件，Rebalance结束会发送OnAssign事件，consumer也可以用这个来实现将offset保存到数据库中。

### Coordinator

为ConsumerGroup服务的组件，负责为ConsumerGroup执行Rebalance以及提供位移管理和组成员管理等。Consumer在提交位移时，其实是向Coordinator所在的Broker提交位移。当Consumer应用启动时，也是向Coordinator所在的Broker发送各种请求，然后由Coordinator负责执行消费者组的注册、成员管理记录等元数据管理操作。

### 避免不必要的Rebalance

Rebalance会影响Consumer的性能，在Rebalance期间，Consumer啥都干不了，consumer数量越多，Rebalance时间越长，所有Consumer都需要重新分配消费的Parition。所以业务上最好避免不需要的Rebalance。

Rebalance的时机：

-  组成员数发生变化：经常发生，比如说停进程、启进程
-  订阅主题数发生变化：一旦上线，万年不变
-  订阅主题的分区数发生变化：一旦上线，万年不变

组成员数发生变化主要2种，要么增加，要么是减少。增加的情况不用管，代表业务增加实例了，我们肯定是希望发生Rebalance的。

组成员数减少发生的Rebalance：

ConsumerGroup在Rebalance完成之后，每个实例都需要向Coordinator定期发送心跳，让Kafka知道消费者还存活着，如果某个Consumer不能及时发送心跳，那么Coordinator会认为这个Consumer已经挂掉，需要从ConsumerGroup中移除，从而导致发生Rebalance。

- session.timeout.ms

这个参数是Coordinator容忍consumer发送心跳的最长时间，该参数的默认值是10秒，如果Coordinator在10秒之内没有收到ConsumerGroup下某个Consumer实例的心跳，它就会认为这个Consumer实例已经挂了。可以这么说，session.timout.ms决定了Consumer存活性的时间间隔。

- heartbeat.interval.ms

这个参数是控制发送心跳请求频率。这个值设置得越小，Consumer实例发送心跳请求的频率就越高。频繁地发送心跳请求会额外消耗带宽资源，但好处是能够更加快速地知晓当前是否开启Rebalance，因为，目前Coordinator通知各个Consumer实例开启Rebalance的方法，就是将 REBALANCE_NEEDED标志封装进心跳请求的响应体中。

- max.poll.interval.ms 

控制Consumer实际消费能力对Rebalance的影响，限定了Consumer应用程序两次调用poll方法的最大时间间隔。它的默认值是5分钟，表示Consumer程序如果在5分钟之内无法消费完poll方法返回的消息，那么Consumer会主动发起LeaveGroup的请求，Coordinator也会开启新一轮Rebalance。所以如果消费之后做很耗时的操作，就需要考虑把这个值改大。

推荐session.timeout.ms为6秒，heartbeat.interval.ms为2秒。session.timeout.ms >= 3 * heartbeat.interval.ms，至少要给Consumer3次机会来发送消息。

- Consumer的GC表现

是否因GC问题导致产生Rebalance

### Offset提交

Offset：是下一次将要消费的位移。Consumer需要向Kafka上报消费进度，专业术语就是提交位移。如果某个Consumer消费了多个分区的数据，那么就需要为每个分区提交Offset。提交Offset主要是为了表示消费进度，当Consumer发生故障后，就能够从Kafka中读取之前提交的位移值，然后从相应的位置开始继续消费。

提交Offset还是得慎重的，如果消费了10条消息，提交的位移是20，位移11-19的消息就消费不到了，如果提交的位移是5，那么位移5-9的消息就会被重复消费。所以提交Offset要慎重，Kafka只认你传的Offset。

提交Offset的方法：

- 自动提交

设置enable.auto.commit为true就代表使用自动提交方式，Consumer在后台默默帮你做这个事情，还有个参数auto.commit.interval.ms，它表示自动提交的周期。默认值是5秒，表示Kafka每5秒自动提交一次Offset。

自动提交的策略：

Kafka在调用poll方法的时候，会提交上次poll返回的消息，顺序上是没有问题的，可以保证不会出现消费丢失的情况，但是可能会出现重复消费情况，假设我们在位移提交之后3秒（位移提交默认是5秒）出现Rebalance情况。所有Consumer从上一次提交位移的地方进行消费，但是这个Offset是3秒之前的offset，所以在Rebalance发生前3秒的消费都会被重复消费。虽然可以用auto.commit.interval.ms这个参数来提高提交的频率，但是不能根治这个问题。

- 手动提交

先设置enable.auto.commit为false就代表不使用自动提交方式，然后在需要提交的地方调用Commit方法。手动提交的时机应该是在消费完消息后，然后在进行提交操作。

手动提交又分为异步和同步提交。同步提交就是需要等待Broker返回才能知道提交的结果，这个期间Consumer处于阻塞状态，会影响TPS。异步提交就是只提交给Broker，然后Consumer还可以继续处理其他事情，然后在回调中处理。

实际使用中需要按照业务来自己定义手动提交的策略。比如可以按照每消费100条消息就提交offset，也可以按照固定时间来提交offset。

其实手动提交也不能保证不出现重复消费，还需要业务自己做幂等性处理。

### 重置消费者位移

#### 重设位移的策略

- 位移维度

Earliest策略表示将位移调整到主题当前最早位移处。这个最早位移不一定就是0，因为在
生产环境中，很久远的消息会被Kafka自动删除，所以当前最早位移很可能是一个大于0的值。

Latest策略表示把位移重设成最新末端位移。如果总共向某个主题发送了15条消息，那么最新末端位移就是15。如果你想跳过所有历史消息，打算从最新的消息处开始消费的话，可以使用 Latest 策略

Current 策略表示将位移调整成消费者当前提交的最新位移。有时候你可能会碰到这样的场
景：你修改了消费者程序代码，并重启了消费者，结果发现代码有问题，你需要回滚之前的
代码变更，同时也要把位移重设到消费者重启时的位置，那么，Current 策略就可以帮你实
现这个功能。

Specified-Offset策略则是比较通用的策略，表示消费者把位移值调整到你指定的位移处。这个策略的典型使用场景是，消费者程序在处理某条错误消息时，你可以手动地“跳过”此消息的处理。

Shift-By-N策略指定的就是位移的相对数值，即你给出要跳过的一段消息的距离即可。这里的“跳”是双向的，你既可以向前“跳”，也可以向后“跳”。比如，你想把位移重设成当前位移的前100条位移处，此时你需要指定N为-10

- 时间维度

DateTime允许你指定一个时间，然后将位移重置到该时间之后的最早位移处。常见的使用场景是，你想重新消费昨天的数据，那么你可以使用该策略重设位移到昨天0点

Duration策略则是指给定相对的时间间隔，然后将位移调整到距离当前给定时间间隔的位处，具体格式是PnDTnHnMnS。

#### 重设位移的方法

- 使用ConsumerApi中的Seek方法
- 通过kafka-consumer-groups脚本，0.11版本后引入

## Kafka核心技术

### 集群管理

Apache ZooKeeper 是一个提供高可靠性的分布式协调服务框架，它使用的数据模型类似于文件系统的树形结构，根目录也是以“/”开始。该结构上的每个节点被称为znode，用来保存一些元数据协调信息。ZooKeeper常被用来实现集群成员管理、分布式锁、领导者选举等功能，从功能上来说和etcd很像。

如果以znode持久性来划分，znode可分为持久性znode和临时znode。持久性znode不会因为ZooKeeper集群重启而消失，而临时 znode 则与创建该znode的ZooKeeper 会话绑定，一旦会话结束，该节点会被自动删除。

ZooKeeper赋予客户端监控znode变更的能力，即所谓的Watch通知功能。一旦znode节点被创建、删除，子节点数量发生变化，或是znode所存的数据本身变更，ZooKeeper 会通过节点变更监听器 (ChangeHandler) 的方式显式通知客户端。

### 副本机制

在创建主题的时候需要创建主题的分区数，还有一个参数就是副本数量，可以为每个分区设置副本数量。

副本主要是追加写的提交日志，同一个分区的副本保存相同的消息序列，且同一个分区的副本均匀分散在不同的Broker下，避免一台Broker宕机带来的数据不可用。创建topic的时候，副本数量不能大于broker数量，不然会报错，想想Kafka也不可能给你这么搞。

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/Kafka_fb2.png)

Kafka副本包括领导者副本和追随者副本，每个分区创建时都需要选举一个副本作为领导者副本，其他的追随者副本。Kafka的追随者副本是不对客户端提供服务的，它只会定时从领导者副本中异步拉取消息同步，对客户端提供服务的只有领导者副本。当领导者副本所在的Broker宕机了，Kafka依赖于Zookeeper提供的检测功能会实时检测到，开始新一轮的选举，从追随者副本中选举出一个领导者副本，如果之前的领导者又启来了，那么还是只会加入到追随者副本中。

Kafka追随者副本对外不提供服务的原因：

- 1、消息产生在领导者副本，其他追随者副本同步需要一定时间，所以如果有消费者需要消费，那么有可能消费不到最新的数据
- 2、更容易实现单调读，假设有2个追随者副本F1和F2，有可能F1拉到最新的，F2还没有拉到，那么一个消费者先从F1读，之后在从F2读，第一次消费的消息在第二次消费的时候不见了。

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka_fb.png)

### ISR

Kafka引入了ISR副本集合，ISR副本都是与Leader同步的副本，不在ISR中的副本就认为与Leader不同步的。Leader副本天然在ISR集合中，ISR不只是追随者副本集合，它必然包括 Leader 副本。甚至在某些情况下，ISR 只有 Leader 这一个副本。

Broker端参数有个replica.lag.time.max.ms参数，这个参数规定了追随者副本与领导者副本落后的最长时间间隔，默认值是10s。只要在10s内，Kafka就认为追随者副本与领导者副本是同步的，即使追随者副本落后于领导者副本。如果同步过程的速度持续慢于Leader副本消息的写入速度，那么在replica.lag.time.max.ms时间后，此Follower副本就会被认为是与Leader副本不同步的，不能再放入ISR中。Kafka会自动收缩ISR集合，将该副本“踢出”ISR。

### Unclean 领导者选举

Kafka 把所有不在 ISR 中的存活副本都称为非同步副本，非同步副本落后Leader太多，如果选择这些副本中的其中一个作为新Leader，就可能出现数据的丢失，因为这些副本中保存的消息远远落后于老Leader中的消息。Kafka选举这种副本的过程称为Unclean领导者选举，Broker端参数 unclean.leader.election.enable控制是否允许Unclean领导者选举。开启Unclean领导者选举可能会造成数据丢失，它使得分区Leader副本一直存在，不至于停止对外提供服务，因此提升了高可用性，线上建议不开启这个参数。

### 日志文件

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka_log_dir.png)

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka_log_segment.png)

在创建topic的时候，在Kafka日志文件目录下，会创建对应<topic-分区号>的目录，每个目录都有对应的日志段文件和索引文件，.log文件就是日志段文件，.index和.timeindex就是索引文件。

- .log文件

.log文件就是保存着Kafka消息数据，每个.log文件都包含一段位移范围的Kafka消息，00000000000060270917.log表示该文件保存的第一条消息的位移是60270917。每个.log文件有最大上限，是由log.segment.bytes参数控制的，默认是1G，如果.log文件被填满了，会重新创建一组新的日志段文件和索引文件，这个过程叫日志切分。当前正在写的.log文件称为当前日志段。

- .index文件

位移索引文件，帮助broker更快定位某条消息的具体文件位置，每个文件都由若干条索引项组成，Kafka不会为每条消息都保存索引项，而是待写入若干条消息才增加一个索引项，log.index.interval.bytes这个参数可以设置这个时间间隔，默认值是4KB，即Kafka分区至少写入4KB的数据才会在索引文件中增加一个索引项。

他们的索引项是升序的，.index中是按照位移升序来顺序保存的，且每个索引项占用的大小是固定的。所以Kafka可以利用二分查找法来方便的定位目标的索引项，这种时间复杂度是O(logN)。如果没有索引文件，Kafka只能从头开始遍历消息文件，时间复杂度是O(N)，使用索引文件可以减低broker端的CPU开销。

位移索引文件的索引项保存的是<相对位移,物理位置>二元组，每个索引项占用8个字节，其中相对位移和物理位置都占用4个字节，所以Kafka强制要求索引文件大小是8的整数倍。相对位移是相对于索引文件起始位移的差值，这样就只需要保存4B，而不是保存位移8B，当然在查找的时候肯定是要转换成具体位移的。

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka_index.png)

如果要找3000位移的数据，通过二分查找法找到小于3000最大的索引项（2650,1150100），然后在数据文件从1150100字节处开始遍历，直到找到3000位移的数据。

- .timeindex文件

时间戳索引文件，0.10版本加入的，按照时间戳升序来顺序保存的，主要是为了让broker找某段时间内的消息数据，时间戳索引文件的索引项保存的是<时间戳,相对位移>二元组，每个索引项占用12位，时间戳字段占8个字节，相对位移占4个字节，所以Kafka强制要求时间索引文件大小是12的整数倍。

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka_time_index.png)

假设我们要寻找1499824275743附近的消息，找到（1499824275740,7447）这个索引项，然后根据7447去查找位移索引文件，找到（7053,3061002），然后从日志文件第3061002字节处开始的消息就是满足该时间戳条件的消息。

### Kafka请求

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka_req.png)

Kafka的Broker端有个SocketServer组件，类似于Reactor模式中的Dispatcher，它也有对应的 Acceptor线程和一个工作线程池，工作线程池有个别名就是网络线程池。Kafka中有个参数num.network.threads，默认是3，表示每台Broker会启动3个网络线程池来专门处理客户端请求。

 - Acceptor线程采用轮询的方式将请求公平地发到所有网络线程中。
 - 网络线程拿到请求后，它不是自己处理，而是将请求放入到一个共享请求队列中。
Broker端还有个IO线程池，负责从该队列中取出请求，执行真正的处理。如果是PRODUCE 生产请求，则将消息写入到底层的磁盘日志中；如果是FETCH请求，则从磁盘或页缓存中读取消息。
- IO 线程池处中的线程才是执行请求逻辑的线程。Broker端参数num.io.threads控制了这
个线程池中的线程数。该参数默认值是 8，表示每台Broker启动后自动创建8个IO
线程处理请求。你可以根据实际硬件条件设置此线程池的个数。
- 当IO线程处理完请求后，会将生成的响应发送到网络线程池的响应队列中，然后由对应的网络线程负责将Response返还给客户端
- 请求队列是所有网络线程共享的，而响应队列则是每个网络线程专属的。这么设计的原因就在于，Dispatcher只是用于请求分发而不负责响应回传，因此只能让每个网络线程自己发送 Response给客户端，所以这些Response也就没必要放在一个公共的地方
-  Purgatory组件是Kafka中的“炼狱”组件。它是用来缓存延时请求（DelayedRequest）的。所谓延时请求，就是那些一时未满足条件不能立刻处理的请求。比如设置了acks=all的PRODUCE请求，一旦设置了acks=all，那么该请求就必须等待ISR中所有副本都接收了消息后才能返回，此时处理该请求的IO线程就必须等待其他Broker的写入结果。当请求不能立刻处理时，它就会暂存
在Purgatory中。稍后一旦满足了完成条件，IO线程会继续处理该请求，并将Response放入对应网络线程的响应队列中。

### 控制器

控制器组件，是Apache Kafka的核心组件。它的主要作用是在 Apache ZooKeeper 的帮助下管理和协调整个Kafka集群，集群中任意一台Broker都能充当控制
器的角色，但是，在运行过程中，只能有一个Broker成为控制器，行使其管理和协调的职
责。


#### 控制器选举

Broker在启动时，会尝试去ZooKeeper中创建/controller 节点。Kafka 当前选举控制器的规则是：第一个成功创建 /controller 节点的 Broker 会被指定为控制器。

#### 控制器作用

- 1、主题管理（创建、删除、增加分区）

当我们执行kafka-topics脚本时，大部分的后台工作都是控制器来完成的

- 2、分区重分配

kafka-reassign-partitions脚本提供的对已有主题分区进行细粒度的分配功能。这部分功能也是控制器实现的。

- 3、Preferred领导者选举

Preferred领导者选举主要是Kafka为了避免部分Broker负载过重而提供的一种换Leader 的方案。在专栏后面说到工具的时候。

- 4、集群成员管理

自动检测新增 Broker、Broker 主动关闭及被动宕机。这种自动检测是依赖于前面提到的 Watch 功能和 ZooKeeper临时节点组合实现的。

控制器组件会利用Watch机制检查ZooKeeper的/brokers/ids节点下的子节点数量变更。目前，当有新Broker启动后，它会在/brokers下创建专属的znode节点。一旦创建完毕，ZooKeeper会通过 Watch机制将消息通知推送给控制器，控制器就能自动地感知到这个变化，进而开启后续的新增 Broker作业。

侦测Broker存活性则是依赖于的另一个机制：临时节点。每个Broker启动后，
会在/brokers/ids下创建一个临时 znode。当 Broker 宕机或主动关闭后，该 Broker 与
ZooKeeper 的会话结束，这个znode会被自动删除。同理，ZooKeeper的Watch 机制将这一变更推送给控制器，这样控制器就能知道有Broker关闭或宕机了，从而进行“善
后”。

- 5、数据服务

向其他 Broker 提供数据服务。控制器上保存了最全的集群元数据信息，其他所有Broker会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。


保存主要的信息：

- 所有主题信息。包括具体的分区信息，比如领导者副本是谁，ISR 集合中有哪些副本等。
- 所有 Broker 信息。包括当前都有哪些运行中的 Broker，哪些正在关闭中的 Broker 等。


值得注意的是，这些数据其实在 ZooKeeper 中也保存了一份。每当控制器初始化时，它都
会从ZooKeeper上读取对应的元数据并填充到自己的缓存中。有了这些数据，控制器就能对外提供数据服务了。这里的对外主要是指对其他Broker而言，控制器通过向这些Broker发送请求的方式将这些数据同步到其他Broker上。

#### 控制器故障转移

在 Kafka 集群运行过程中，只能有一台Broker充当控制器的角色，那么这就存在单点的风险，答案就是，为控制器提供故障转移功能（Failover）。故障转移指的是，当运行中的控制器突然宕机或意外终止时，Kafka能够快速地感知到，并立即启用备用控制器来代替之前失败的控制器，该过程是自动完成的，无需你手动干预。

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka_controller.png)

## Kafka命令

### 主题类

#### 创建

```
bin/kafka-topics.sh --create --bootstrap-server broker_host:port --replication-factor 1 --partitions 1 --topic test
```

- create：创建主题
- partitions：分区数量
- replication-factor：副本数量
- bootstrap-server：broker地址，老版本是--zookeeper参数

#### 查询主题列表

```
bin/kafka-topics.sh --list --bootstrap-server broker_host:port
```

#### 查询单主题详细信息

```
bin/kafka-topics.sh --bootstrap-server broker_host:port --describe --topic <topic_name>
```
#### 修改主题

- 修改分区数量

```
bin/kafka-topics.sh --bootstrap-server broker_host:port --alter --topic my_topic_name --partitions 40
```

- 添加主题级别参数

```
bin/kafka-configs.sh --bootstrap-server broker_host:port --entity-type topics --entity-name my_topic_name --alter --add-config x=y
```

#### 删除主题

```
bin/kafka-topics.sh --bootstrap-server broker_host:port --delete --topic <topic_name>
```

## 常用脚本

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka_bin.png)

connect-standalone和connect-distributed是Kafka Connect组件的功能。

kafka-acls：用于设置Kafka权限的，比如设置哪些用户可以访问Kafka的哪些主题之类的权限。

kafka-broker-api-versions：主要目的是验证不同 Kafka 版本之间服务器和客户端的适配性。

kafka-configs：主要用于配置修改，可以指定修改topic、client、user、broker对象

kafka-console-consumer和kafka-consoleproducer：通过控制台来向Kafka生产和消费消息。

kafka-producer-perf-test 和 kafka-consumer-perf-test：生产者和消费者的性能测试工具。

kafka-consumer-groups：查看consumerGroup操作

kafka-delete-records：删除Kafka的分区消息，由于Kafka本身有自己的自动消息删除策略，这个脚本其实用的并不多

kafka-dump-log：查看Kakfa消息文件的内容，包括消息的元数据

kafka-mirror-maker：实现Kafka集群间的消息同步的。

kafka-preferred-replica-election：脚本是执行Preferred Leader选举的。它可以为指定
的主题执行“换 Leader”的操作

kafka-reassign-partitions：执行分区副本迁移以及副本文件路径迁移

kafka-topics：主题管理

kafka-run-class：可以用这个脚本执行任何带main方法的Kafka类

kafka-server-start 和 kafka-server-stop ：启动和停止 Kafka Broker 进程

kafka-streams-application-reset：脚本用来给Kafka Streams应用程序重设位移，以便重
新消费数据。

kafka-verifiable-producer和kafka-verifiable-consumer：脚本是用来测试生产者和消费
者功能的。它们是很旧的脚本，几乎用不到。前面的Console Producer和Console Consumer 完全可以替代它们。

zookeeper*：zookeeper开头的脚本是用来管理和运维ZooKeeper的。

## 常见问题

### Kafka为什么快

- 磁盘顺序写

Kafka的消息是不断追加到文件中的，这个特性使它可以充分利用磁盘的顺序读写能力。
顺序读写降低了硬盘磁头的寻道时间，只需要很少的扇区旋转时间，所以速度远快于随机读写。
Kafka官方给出的测试数据(Raid-5, 7200rpm)
顺序I/O: 600MB/s
随机I/O: 100KB/s

- ZeroCopy

![image](https://raw.githubusercontent.com/webbc/webbc.github.io/master/kafka/kafka_zero_copy.png)

通常的方式：

a. 操作系统将数据从磁盘读取到内核空间中的页面缓存中

b. 应用程序将数据从内核空间读取到用户空间缓冲区中

c. 应用程序将数据写回到内核空间的套接字缓冲区中

d. 操作系统将数据从套接字缓冲区复制到通过网络发送的网卡缓冲区

linux操作系统 “零拷贝” 机制使用了sendfile方法， 允许操作系统将数据从Page Cache 直接发送到网络，只需要最后一步的copy操作将数据复制到网卡缓冲区，这样避免重新复制数据 。

 - PageCache

为了优化读写性能，Kafka利用了操作系统本身的PageCache，就是利用操作系统自身的内存而不是JVM空间内存。这样做的好处：

  a. 避免Object消耗：如果是使用Java堆，Java对象的内存消耗比较大，通常是所存储数据的两倍甚至更多。

  b. 避免GC问题：随着JVM中数据不断增多，垃圾回收将会变得复杂与缓慢，使用系统缓存就不会存在GC问题

相比于使用JVM或in-memory cache等数据结构，利用操作系统的Page Cache更加简单可靠。首先，操作系统层面的缓存利用率会更高，因为存储的都是紧凑的字节结构而不是独立的对象。其次，操作系统本身也对于Page Cache做了大量优化，提供了 write-behind、read-ahead以及flush等多种机制。再者，即使服务进程重启，系统缓存依然不会消失。

通过操作系统的Page Cache，Kafka的读写操作基本上是基于内存的，读写速度得到了极大的提升。

- 分区分段+索引

Kafka的message是按topic分类存储的，topic中的数据又是按照一个一个的partition即分区存储到不同broker节点。每个partition对应了操作系统上的一个文件夹，partition实际上又是按照segment分段存储的。这也非常符合分布式系统分区分片的设计思想。

通过这种分区分段的设计，Kafka的message消息实际上是分布式存储在一个一个小的segment中的，每次文件操作也是直接操作的segment。为了进一步的查询优化，Kafka又默认为分段后的数据文件建立了索引文件，就是文件系统上的.index文件。这种分区分段+索引的设计，不仅提升了数据读取的效率，同时也提高了数据操作的并行度。

- 端到端批量压缩

在某些情况下，瓶颈实际上不是CPU或磁盘，而是网络带宽。特别是需要通过广域网在数据中心之间发送消息的数据管道。当然，用户可以一次压缩每个消息，在不用Kafka的任何支持，但这会导致压缩率非常低，因为大量冗余是由于相同类型消息之间的重复（例如， Web日志中的JSON或用户代理或通用字符串值）。高效压缩需要将多个消息压缩在一起，而不是分别压缩每个消息。Kafka以有效的批处理格式支持此操作。一批消息可以压缩在一起，然后以这种形式发送到服务器。这批消息将以压缩形式写入，并保持压缩在日志中，并且仅由使用者解压缩。