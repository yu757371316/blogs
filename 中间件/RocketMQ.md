##### 调优

```
jvm：调优、关闭偏向锁特性、滚动GC日志
锁：为了避免锁带来的延迟(上下文切换)、利用CAS原语将RocketMQ核心链路无锁化
内存：缓存(当后台内存回收的速度不及分配内存的速度时，会进入直接回收(DirectReclaim)，应用程序会自旋等待内存回收完毕，产生巨大的延迟vm.extra_free_kbytes)、匿名内存(内核也会回收匿名内存页、匿名内存页被换出后下一次访问会产生文件IO、导致延迟vm.swappiness)
Page Cache：将数据文件映射到内存中，写消息的时候首先写入Page Cache，并通过异步刷盘的模式将消息持久化(同时也支持同步刷盘)，消息可以直接从Page Cache中读取
降级：丢卒保车，二八原则实践
限流：计数器、滑动窗口、令牌桶、漏桶
熔断：借鉴Hystrix思路，中间件团队自研了一套消息引擎熔断机制
高可用：Master/Slave结构、采用主备同步复制的方式避免故障时消息的丢失、通过维护一个递增的全局唯一SequenceID来保证数据强一致。同时引入故障自动恢复机制以降低故障恢复时间，提升系统的可用性、Master和Slave任一节点故障时，其它节点能够在秒级时间内切换到单主状态继续提供服务
```


##### 角色

```
topic：主题，默认的队列数量是4个
生产者：三种方式发送消息：同步、异步和单向.(单向发送是指只负责发送消息而不等待服务器回应且没有回调函数触发，适用于某些耗时非常短但对可靠性要求并不高的场景，例如日志收集。)
消费者:拉取型消费者和推送型消费者(从实现上看还是从消息服务器中拉取消息，不同于Pull的是Push首先要注册消费监听器，当监听器处触发后才开始消费消息。)，定时向所有的broker发送心跳和订阅关系
Broker：Broker 有Master和Slave两种类型，Master既可以写又可以读，Slave 不可以写只可以读。从物理结构上看Broker的集群部署方式有四种：单Master、多Master、多Master多Slave(同步刷盘)、多Master多Slave(异步刷盘)，每个broker与所有的nameserver保持长连接及心跳，并会定时将Topic信息注册到NameServer。每隔30S向nameServer发送心跳
NameServer：几乎无状态的，可以横向扩展，节点之间相互之间无通信，通过部署多台机器来标记自己是一个伪集群。每个Broker在启动的时候会到NameServer注册，Producer在发送消息前会根据Topic到NameServer获取到Broker的路由信息，Consumer也会定时获取Topic的路由信息。定时(10s)运行一个任务,去检查一下各个Broker的最近一次心跳时间,如果某个Broker超过120s都没发送心跳了,那么就认为这个Broker已经挂掉了。
消息队列：一个 Topic下可以设置多个消息队列，发送消息时RocketMQ会轮询该Topic下的所有队列将消息发出去
消息消费模式：集群消费（Clustering）和广播消费（Broadcasting）。默认是集群消费，该模式下一个消费者集群共同消费一个主题的多个队列，一个队列只会被一个消费者消费，如果某个消费者挂掉，分组内其它消费者会接替挂掉的消费者继续消费。而广播消费会发给消费者组中的每一个消费者进行消费
消息顺序：顺序消费（Orderly）和并行消费（Concurrently）。顺序消费表示消息消费的顺序同生产者为每个消息队列发送的顺序一致，所以如果正在处理全局顺序是强制性的场景，需要确保使用的主题只有一个消息队列。并行消费不再保证消息顺序，消费的最大并行数量受每个消费者客户端指定的线程池限制。
Filtersrv：Consumer拉取使用过滤类方式订阅的消费消息时，从 Broker对应的Filtersrv列表随机选择一个拉取消息。如果选择不到Filtersrv,则无法拉取消息。因此,Filtersrv一定要做高可用
```
##### 一次完整的通信流程
```
Producer与NameServer集群中的其中一个节点(随机选择)建立长连接,定期从NameServer获取Topic路由信息,并向提供Topic服务的Broker Master建立长连接,且定时向Broker发送心跳。

Producer只能将消息发送到Brokermaster,但是Consumer则不一样,它同时和提供Topic服务的Master和Slave建立长连接,既可以从Broker Master订阅消息,也可以从Broker Slave订阅消息。

NameServer：启动初始化配置、创建NamesrvController实例，并开启两个定时任务(每隔10s扫描移除处于不激活的Broker、每隔10s打印一次KV配置)、启动服务器并监听Broker
producer：轮训某个Topic下面的所有队列实现发送方的负载均衡
Broker：进行处理Producer发送消息请求，Consumer消费消息的请求，并且进行消息的持久化，以及HA策略和服务端过滤
[Consumer](https://gitee.com/seeks/blogs/blob/master/images/Rocket_Consumer.png)：注册消息监听处理器、定时获取NameServer地址以及topic路由信息、定时向所有broker发送心跳和订阅关系、定时清理下线的broker、定时持久化消费进度(集群模式存在broker、广播模式存在本地)、动态调整消费线程池、启动拉服务、启动负载均衡(消费端会通过RebalanceService线程，10秒钟做一次基于Topic下的所有队列负载。); 消费者默认是从broker主节点拉消息，主节点返回消息的同时，也会根据各个brokerslave的负载，返回下次建议拉取消息的节点给消费者

```

##### [PushConsumer组件之间的交互](https://gitee.com/seeks/blogs/blob/master/images/Rocket_PushConsumer%E7%BB%84%E4%BB%B6%E5%85%B3%E7%B3%BB%E5%9B%BE.png)
```
RebalanceService：均衡消息队列服务，负责分配当前 Consumer 可消费的消息队列(MessageQueue);当有新的Consumer的加入或移除、每20S、PushConsumer启动时，都会重新分配消息队列。
PullMessageService：拉取消息服务，不断不断不断从 Broker 拉取消息，并提交消费任务到 ConsumeMessageService
ConsumeMessageService：消费消息服务，不断不断不断消费消息,并处理消费结果。
RemoteBrokerOffsetStore：Consumer消费进度管理,负责从Broker 获取消费进度，同步消费进度到Broker。定时持久化(每5S)，拉取消息、分配消息队列等等操作，都会进行消费进度持久化
ProcessQueue ：消息处理队列。
MQClientInstance：封装对Namesrv,Broker的API调用，提供给Producer、Consumer 使用。

```

##### RocketMQ 刷盘实现

```
Broker在消息的存取时直接操作的是内存(内存映射文件),这可以提供系统的吞吐量,但是无法避免机器掉电时数据丢失,所以需要持久化到磁盘中。
刷盘的最终实现都是使用NIO中的 MappedByteBuffer.force() 将映射区的数据写入到磁盘,如果是同步刷盘的话,在Broker把消息写到CommitLog映射区后,就会等待写入完成。
异步而言，只是唤醒对应的线程，不保证执行的时机
```
##### 回溯消费

```
回溯消费是指Consumer已经消费成功的消息，由于业务上的需求需要重新消费，要支持此功能，Broker在向Consumer投递成功消息后，消息仍然需要保留。并且重新消费一般是按照时间维度。
RocketMQ支持按照时间回溯消费，时间维度精确到毫秒，可以向前回溯，也可以向后回溯。
```


##### RocketMQ TAG 过滤原理

```
Tags：过滤实现比较简单，主要是在客户端实现。
Sql Filter：上传一段自己的java代码到broker端过滤
```

##### [RocketMQ事务消息概要](https://gitee.com/seeks/blogs/blob/master/images/RocketMq%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF%E6%A6%82%E8%A6%81.png)

```
事务消息发送及提交：
1、发送消（half消息(半消息-会将Topic替换为HalfMessage的Topic)
2、broker响应消息写入结果
3、根据broker响应结果执行本地事务(如果写入失败，此时half消息对业务不可见，本地逻辑也不执行)
4、根据本地事务状态执行Commit或者Rollback(Commit操作生成消息索引,还原topic，则消息对消费者可见)
补偿流程：
1、对没有Commit/Rollback的事务消息(pending状态的消息)，从broker服务端发起一次“回查”
2、Producer收到回查消息，检查回查消息对应的本地事务的状态，根据本地事务状态，重新Commit或者Rollback
补偿阶段用于解决消息Commit或者Rollback发生超时或者失败的情况。
```
##### 定时/延时消息
```
开源版本的RocketMQ中延时消息并不支持任意时间的延时，需要设置几个固定的延时等级，目前默认设置为：1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h，从1s到2h分别对应着等级1到18，而阿里云中的版本(要付钱)是可以支持40天内的任何时刻（毫秒级别）
1、Producer在自己发送的消息上设置好需要延时的级别。
2、Broker发现此消息是延时消息，将Topic进行替换成延时Topic，每个延时级别都会作为一个单独的queue，将自己的Topic作为额外信息存储。
3、构建ConsumerQueue
4、定时任务定时扫描每个延时级别的ConsumerQueue。
5、拿到ConsumerQueue中的CommitLog的Offset，获取消息，判断是否已经达到执行时间
6、如果达到，那么将消息的Topic恢复，进行重新投递。如果没有达到则延迟没有达到的这段时间执行任务。
持久化二级TimeWheel时间轮
```
##### 消息的可用性

```
刷盘：同步和异步的策略，当选择同步刷盘之后，如果刷盘超时会给返回FLUSH_DISK_TIMEOUT
主从同步：同步和异步两种模式来进行复制，当然选择同步可以提升可用性，但是消息的发送RT时间会下降10%左右。
消息发送：
    同步发送：消息发送出去后，producer会等到broker回应后才能继续发送下一个消息
    异步发送：发送方发出数据后，不等接收方发回响应，接着发送下个数据包的通讯方式，但是需要消费方实现callback
    OneWay：发送特点为发送方只负责发送消息，不等待服务器回应且没有回调函数触发，即只发送请求不等待应答.效率最高
存储：采用混合型的存储结构，即为Broker单个实例下所有的队列共用一个日志数据文件(即为CommitLog)来存储。(RocketMQ采用混合型存储结构的缺点在于，会存在较多的随机读操作，因此读的效率偏低。同时消费消息需要依赖ConsumeQueue，构建该逻辑消费队列需要一定开销。)
```

##### 高性能日志存储

```
commitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件
config：保存一些配置信息，包括一些Group，Topic以及Consumer消费offset等信息。
consumeQueue:消息消费队列，引入的目的主要是提高消息消费的性能，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。consumequeue文件可以看成是基于topic的commitlog索引文件，故consumequeue文件夹的组织方式如下：topic/queue/file三层组织结构
```
##### 网络模型

```
Netty网络框架   1+N1+N2+M的线程模型
1个acceptor线程，N1个IO线程，N2个线程用来做Shake-hand,SSL验证,编解码;M个线程用来做业务处理。
这样的好处将编解码，和SSL验证等一些可能耗时的操作放在了一个单独的线程池，不会占据我们业务线程和IO线程。
```
##### 消费模式

```
集群消费：同一个GroupId都属于一个集群，一般来说一条消息只会被任意一个消费者处理。
广播消费：广播消费的消息会被集群中所有消费者进行消息，但是要注意一下因为广播消费的offset在服务端保存成本太高，所以客户端每一次重启都会从最新消息消费，而不是上次保存的offset。
```
##### 消费模型

```
两种模型都是客户端主动去拉消息
MQPullConsumer：拉取消息需传入拉取消息的offset和每次拉取多少消息量，具体拉取哪里的消息，拉取多少是由客户端控制。
MQPushConsumer：同样也是客户端主动拉取消息，但是消息进度是由服务端保存，Consumer会定时上报自己消费到哪里，所以Consumer下次消费的时候是可以找到上次消费的点，一般来说使用PushConsumer我们不需要关心offset和拉取多少数据，直接使用即可。
```
##### [重平衡](https://mp.weixin.qq.com/s/8fB-Z5oFPbllp13EcqC9dw)

```
1、重平衡定时任务每隔20s定时拉取broker,topic的最新信息
3、随机(因为消费者客户端启动时会启动一个线程，向所有broker 发送心跳包)选取当前Topic的一个Broker，获取当前Broker，当前ConsumerGroup的所有机器ID。
3、然后进行策略分配。
由于重平衡是定时做的，所以这里有可能会出现某个Queue同时被两个Consumer消费，所以会出现消息重复投递。
时机：
1、消费者启动之后
2、消费者数量发生变更(broker主动通知)
3、每20秒会触发检查一次rebalance
```
##### 高可用机制
```
Namesrv：启动多个Namesrv,各nameSrv没有任何关系,不进行通信与数据同步。通过Broker循环注册多个Namesrv。Producer、Consumer只需要从Namesrv列表选择一个可连接的进行通信即可
Broker：启动多个Broker分组形成集群实现高可用。Broker分组 = Master(读写)节点x1+Slave(读)节点xN。分组之间没有任何关系,不进行通信与数据同步。每个分组Master节点不断发送新的CommitLog给Slave节点。Slave节点不断上报本地的CommitLog已经同步到的位置给Master节点。消费进度目前不支持Master/Slave 同步

```

##### SendResult

```
有一个sendStatus状态，表示消息的发送状态。一共有四种状态
1. FLUSH_DISK_TIMEOUT ： 表示没有在规定时间内完成刷盘(需要Broker的刷盘策设置成SYNC_FLUSH 才会报这个错误)。
2. FLUSH_SLAVE_TIMEOUT:表示在主备方式下，并且Broker被设置成SYNC_MASTER 方式，没有在设定时间内完成主从同步。
3. SLAVE_NOT_AVAILABLE：这个状态产生的场景和FLUSH_SLAVE_TIMEOUT 类似， 表示在主备方式下，并且Broker被设置成SYNC_MASTER ，但是没有找到被配置成Slave 的Broker 。
4. SEND OK ：表示发送成功，发送成功的具体含义，比如消息是否已经被存储到磁盘？消息是否被同步到了Slave上？消息在Slave 上是否被写入磁盘？需要结合所配置的刷盘策略、主从策略来定。这个状态还可以简单理解为，没有发生上面列出的三个问题状态就是SEND OK
```
##### 集群支持

```
单Master：除了配置简单没什么优点
多Master：配置简单，性能最高，单台机器重启或宕机期间，该机器下未被消费的消息在机器恢复前不可订阅，影响消息实时性
多Master多Slave：每个Master配一个Slave，有多对Master-Slave，集群采用异步复制方式，主备有短暂消息延迟，毫秒级
多Master多Slave：每个Master配一个Slave，有多对Master-Slave，集群采用同步双写方式，主备都写成功，向应用返回成功；性能比异步集群略低，当前版本主宕备不能自动切换为主
注意：RocketMQ里面，1台机器只能要么是Master，要么是Slave。这个在初始的机器配置里面，就定死了。不会像kafka那样存在master动态选举的功能。其中Master的broker id = 0，Slave的broker id > 0
```
##### 顺序消息
```
MessageQueueSelector
1、保证顺序的消息要发送到同一个messagequeue中(自定义发送策略可实现消息只发送到同一个队列)
2、一个messagequeue只能被一个消费者消费，这点是由消息队列的分配机制来保证的
3、一个消费者内部对一个mq的消费要保证是有序的
顺序消费会带来一些问题：
1. 遇到消费失败的消息，无法跳过，当前队列消费暂停
2. 降低了消息处理的性能
```
##### 消费者数量控制对于队列

```
读和写队列不一致，会存在消息无法消费到的问题
1、如果读队列大于消费者数，启动一个消费者，那么这个消费者会消费所有队列，
2、如果读队列等于消费者数，那么意味着消息会均衡分摊到这些消费者上
3、如果消费者数大于readQueueNumbs，那么会有一些消费者消费不到消息，浪费资源
```
##### 消息的的可靠性原则
```
1、所有消费者在设置监听的时候会提供一个回调，业务实现消费回调的时候，只有返回CONSUME_SUCCESS才会认为消费成功了，异常或RECONSUME_LATER，RocketMQ就会认为这批消息消费失败了
2、衰减重试：为了保证消息肯定至少被消费一次，消费失败的消息在延迟的某个时间点(默认是10秒,业务可设置)后，再次投递到这个ConsumerGroup。而如果一直这样重复消费都持续失败到一定次数(默认16次),就会投递到DLQ死信队列。应用可以监控死信队列来做人工干预
3、消费者可以从消息中获取当前消息重试了多少次，超过次数的消息，我们可以存到数据库，以便后期人工干预
```

##### 消息消费失败时重试的原理
```
rocketmq针对每个topic都定义了延迟队列，当消息消费失败时，会发回给broker存入延迟队列中，每个消费者在启动时默认订阅延迟队列，这样消费失败的消息在一段时候后又能够重新消费。延迟时间是和延迟级别一一对应的，延迟时间是随失败次数逐渐增加的，最后一次间隔2小时
```
##### 生产者负载均衡

```
生产者发送时，会自动轮询当前所有可发送的broker，一条消息发送成功，下次换另外一个broker发送，以达到消息平均落到所有的broker上。
• 尽量不要选择刚刚选择过的broker
• 不要选择延迟容错内的broker
这里需要注意一点：假如某个Broker宕机，意味生产者最长需要30秒才能感知到。在这期间会向宕机的Broker发送消息。当一条消息发送到某个Broker失败后，会往该broker自动再重发，假如还是发送失败，则抛出发送失败异常。业务捕获异常，重新发送即可。
```
##### 消费者的负载均衡
```
consumer在启动的时候会实例化rebalanceImpl，这个类负责消费端的负载均衡
MqClientInstance里会调用doRebalance()来进行负载均衡；
consumer负载均衡是指将topicMessageQueue中的消息队列分配到消费者组的具体消费者里去；
• consumer的负载均衡由rebalanceImpl调用allocateMesasgeQueueStratage.allocate()完成；
• 每次有新的consumer加入group就会重新做一下负载；
• 每10秒自动做一次负载；
策略：
• 分页模式(随机分配模式)
• 手动配置模式
• 指定机房模式
• 就近机房模式
• 统一哈希模式
• 环型模式
```
##### 主从切换

```
RocketMQ4.5版本之前，用SlaveBroker同步数据，尽量保证数据不丢失，但是一旦Master故障了，Slave是没法自动切换成Master的。需要手动操作
RocketMQ 4.5之后支持了一种叫做Dledger机制，基于Raft协议实现的一个机制。可以自动实现主从切换，中间耗时大概数十秒
```
