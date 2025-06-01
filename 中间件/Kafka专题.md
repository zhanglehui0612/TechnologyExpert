## 一 核心架构与原理
### 1.1 Kafka 的核心组件有哪些？各自的作用是什么？
#### 1.1.1 topic
✅ Kafka的消息按照主题进行分类，一般不同的业务会创建不同的主题  

#### 1.1.2 partition
✅ 每一个主题可以设置一个或者多个分区Partition，是 Kafka 并行处理的基本单位  
✅ 这样可以提升并发度和吞吐量，方便扩展，以及方便负载均衡  

#### 1.1.3 replica
✅ 每个 Partition 可以有多个 Replica, 其中一个Leader副本，多个Follower副本  
✅ Leader副本负责接收和写入数据以及处理消费者读  
✅ Follower副本只负责从Leader同步，然后用于Leader副本选举  

#### 1.1.4 in-sync replica(ISR)
✅ 我们知道每一个follower副本需要从leader同步数据  
✅ 但是因为各种原因或者问题，可能有的follower同步的快，有的同步的慢，因此需要将ISR列表中的副本剔除掉  
✅ ISR作用就是代表当前可以正常参加选举Leader的副本列表  

#### 1.1.5 producer
✅ 向 Kafka 集群发送消息，可以配置为同步或异步模式发送，支持分区策略  

#### 1.1.6 consumer group & consumer
✅ 消费者组表示有一组消费者订阅了某一个主题  
✅ 消费者组因为成员加入和离开，经常需要rebalance操作  
✅ 消费组内的成员订阅了某一个主题，会将该主题的分区分配给这些消费者；如果组内消费者数量>分区数，则可能有一些消费者不能无法消费；如果组内消费者数量<分区数，则有的消费者可能分到多个分区  

#### 1.1.7 broker
✅ Kafka服务器节点，用于接收、存储和消费消息  
✅ 多个 Broker 构成 Kafka 集群  

#### 1.1.8 zookeeeper (kraft)
✅ broker集群存在单点问题，当一个broker故障，那么需要将该节点的上的replica重新再其他节点之间选举一个新的leader, 故障检测和选举机制就是通过zookeeper实现  
✅ 另外zookeeeper还用于用于管理 Kafka 集群的元数据  

#### 1.1.9 KafkaController
✅ Kafka Controller 是 Kafka 集群中一个 特殊的 Broker 实例，负责整个集群的 控制逻辑与元数据协调工作  
✅ 比如负责 分区副本的 Leader 选举、集群状态变更通知、节点状态监控 和 元数据同步 等关键控制功能  



## 二 Kafka如何实现或者保证可用性的
### 2.1 可用性是什么
✅ 可用性是指当某一broker节点宕机，那么主题还可以继续读写，不存在单点故障问题  

### 2.2 分区 + 副本机制
✅ kafka 针对主题设置多个分区，提升并发度和吞吐量，降低节点故障对整个主题带来的问题，这样只会影响小部分  
✅ 每一个分区设置一个Leader副本和Follower副本，这些副本均匀分布在broker集群中，每一个broker都可能存在一个或多个副本  
✅ 如果发生单点故障，那么进行故障转移就好  

### 2.3 ISR机制
✅ 对于分区的副本，Follower需要从Leader同步数据，如果Follower跟不上，那么这个节点就会从ISR同步副本列表移除  
✅ 只有处于ISR同步副本列表中的节点，才可以参加Leader节点选举  

### 2.4 消息持久化
✅ 对于队列或者分区中的消息，如果只是存在内存中，那么故障或者重启，数据就丢失了，这样也无法谈可用性  
✅ 所以需要将消息写入到磁盘进行持久化  



## 三 Kafka如何实现数据一致性与可靠性
### 3.1 什么是数据一致性与可靠性
✅ 数据一致性: 确保同一 Partition 的所有副本数据一致  
✅ 数据可靠性: 写不丢 + 读不丢 + 顺序不乱  

### 3.2 生产者写入数据不丢失
✅ acks=all: 等待所有 ISR 成员确认写入，才返回成功  
✅ min.insync.replicas: 限制 ISR 至少有几个副本在线，否则拒绝写入   
✅ enable.idempotence=true: Producer 幂等写入，自动去重，避免因重试产生重复数据  
✅ retries + retry.backoff.ms: 写入失败重试，避免因网络异常或 Leader 切换时自动重试写入  

### 3.3 服务端数据不丢失
✅ 多副本机制: 每个 Partition 有多个副本，防止单点故障导致数据丢失  
✅ ISR 机制: 只与健康副本同步，写入数据至少被 ISR 成员持久化  
✅ Controller 自动选主: Broker 崩溃后立即从 ISR 中选出新 Leader，继续服务  
✅ 日志文件持久化: 数据写入磁盘，即使 Broker 挂掉也能恢复  

### 3.4 消费者消费数据不丢失
✅ Consumer Group 自动重平衡: 某个消费实例宕机，其他实例接管，继续消费  
✅ 手动提交 offset: 处理完再提交，防止“处理失败但 offset 已提交”导致数据丢  

### 3.5 数据顺序不乱
✅ 只能保证单个分区顺序不乱  
✅ 每一个Paritition, 服务端Leader副本是单线程处理，而且又是顺序写的方式写入磁盘  
✅ Kafka 为每条消息分配严格递增的 offset，因此可以确保 Partition 内的消息顺序绝对可靠，不会乱序  

## 五 谈一谈ISR机制?
### 5.1 什么是ISR
✅ ISR就是同步副本，ISR只代表健康的副本，我们知道分区主要分为多个副本，副本有区分为Leader副本和多个Follower副本  
✅ Leader副本肯定是在ISR列表中  
✅ Follower副本需要从Leader同步数据，当Follower副本同步的比较慢就会从ISR列表中移除  
✅ 当触发Leader选举的时候，会从ISR列表中选择一个Follower副本来提升为Leader副本  

### 5.2 什么情况下才认为副本需要被移除
✅ replica.lag.time.max.ms  
✅ 允许 Follower 副本 最大滞后时间， 毫秒为单位  
✅ 如果一个 Follower 在 replica.lag.time.max.ms 时间内没有从 Leader 拉取到新数据，就会被认为“落后”，Kafka 自动将它移出 ISR， 默认值：10 秒  
✅ 当前时间 - Follower 最后 fetch 数据的时间 > replica.lag.time.max.ms  



## 六 Kafka 吞吐量和延迟的影响因素有哪些？如何优化？
### 6.1 吞吐量因素
1. [ ] batch.size: 批处理大小
2. [ ] compression.type: 消息压缩类型
3. [ ] 分区数量: 分区数量越多，吞吐量越高
4. [ ] 副本数量: 副本数量越多，吞吐量越低
5. [ ] 磁盘 I/O: 磁盘 I/O越强，吞吐量越高
6. [ ] 网络带宽: 带宽越大，吞吐量越高
7. [ ] num.network.threads: 网络线程数,更多的网络线程可以并行处理更多的客户端请求，减少请求的等待时间，提高吞吐量
8. [ ] num.io.threads:指定了 Kafka broker 用于处理磁盘 I/O 操作,更多的 I/O 线程允许 Kafka broker 更快地处理磁盘上的数据，特别是在高吞吐量和高写入负载的情况下。增加 I/O 线程数能够更好地利用磁盘带宽和提高磁盘 I/O 性能

### 6.2 延迟因素
1. [ ] linger.ms: 如果 batch.size 没满，就会最多等 linger.ms 毫秒看看有没有新消息加入 batch，因此linger.ms配置的低一些，延迟就会低一些，因为等待的时间就少一点
2. [ ] acks: 需要多个副本确认，延迟就会越高
3. [ ] log.flush.interval.ms: 间隔多少时间刷盘一次，频率降低，可以降低延迟，但是增加数据丢失风险
4. [ ] log.flush.interval.messages: 默认是很大的值，所以取决于log.flush.interval.ms，如果要配置1，就是1条消息就刷一次



## 七 Kafka为什么读写性能这么快？
### 7.1 分区机制
✅ 通过对主题进行分区，支持高并发读写  

### 7.2 批量处理与压缩
✅ 批量写入：生产者会将多个消息打包成一个批次发送，减少了网络请求和消息发送的次数，提升了吞吐量  
✅ 压缩：Kafka 支持多种压缩算法（如 Snappy、GZIP 等），压缩消息后可以显著减小消息的大小，从而减少网络传输和存储的开销  
✅ 批量拉取：Kafka 消费者通过 fetch.max.bytes 和 fetch.min.bytes 配置来控制每次拉取的消息批量大小。消费者每次拉取时，会尽可能拉取更多的数据，以减少频繁的网络请求  

### 7.3 长连接与连接复用
✅ Kafka 客户端（生产者和消费者）与 Broker 之间通常会保持长连接，而不是每次操作都重新建立连接。这可以减少因频繁建立和关闭连接带来的网络开销，提升性能  
✅ Kafka 通过使用 Netty 等高效的 I/O 库来处理网络请求，这种库能够支持高并发的网络连接和异步 I/O 操作，提高网络 I/O 的处理能力  

### 7.4 顺序写入
✅ 随机写会导致磁盘寻址（seek）和磁头移动的开销，极大地降低磁盘性能   
✅ Kafka 将消息按顺序追加到日志文件中，减少了磁盘的寻址时间  

### 7.5 零拷贝技术
✅ Kafka 利用操作系统提供的 sendfile() 系统调用将文件内容从磁盘直接发送到网络中，无需将数据从磁盘读取到内存再通过网络发送  
✅ 极大减少了 CPU 和内存的占用，提高了吞吐量  

### 7.6 OS Page Cache(操作系统缓存)
✅ 减少磁盘 I/O，提高读取速度  

### 7.7 设计了索引机制，提升读取和查询性能



## 八 Kafk故障转移流程
### 8.1 Broker故障(非Controller Leader所在broker)
#### 8.1.1 Broker 故障检测
✅ 每个 Broker 启动时会在 ZooKeeper 的 /brokers/ids 节点下创建临时节点  
✅ 如果 Broker 宕机，ZooKeeper 会自动删除该节点  
✅ Controller Broker 通过监听 /brokers/ids 节点的变化，发现 Broker 失效  

#### 8.1.2 触发故障转移流程
✅ 当 Controller 发现 Broker 节点消失时，会触发 BrokerChangeListener  
✅ 执行 onBrokerFailed 流程  

#### 8.1.3 处理 Leader 副本故障
✅ 遍历该 Broker 所拥有的 Leader 分区副本  
✅ 将分区状态迁移为 OfflinePartition  
✅ 启动 Leader 重新选举流程  
✅ 新 Leader 选定后，将分区状态更新为 OnlinePartition  
✅ 通知该分区follower副本所在的在其他broker新的Leader副本信息  

#### 8.1.4 处理 Follower 副本故障
✅ 从对应 Partition 的 ISR列表 中移除当前Follower副本  
✅ 将这些副本状态迁移为 OfflineReplica  
✅ 剩余 ISR 会停止向其发送数据同步  

#### 8.1.5 旧 Leader 恢复后的处理
✅ 以 Follower 身份重新加入集群  
✅ 向新 Leader 请求同步数据  
✅ 数据追上后重新加入 ISR  

### 8.2 Leader Controller 故障转移
#### 8.2.1 其他broker重新选举新的controller
✅ Controller被选举为新的Leader, 即向zookpeer路径下写入数据成功，即表示被选举为新的Leader  
✅ 当该Controller所在Broker故障，则会被检测到，所以会被重新触发选举新的Controller  

#### 8.2.2 初始化和启动新的Controller
✅ 注册对 /brokers、/topic、/isr 等节点的监听器  
✅ 启动 PartitionStateMachine 和 ReplicaStateMachine  
✅ 触发自动 Rebalance  

#### 8.2.3 开始处理broker的分区和副本状态迁移



## 九 Kafka 网络通信机制
### 9.1 Kafka是基于主从Reactor线程模型 + I/O多路复用网络模型实现网路通信的

### 9.2 Kafka 服务端启动时创建 SocketServer
1. [ ] 初始化 ServerSocketChannel，绑定端口
2. [ ] 启动 1 个 Acceptor 线程 + N 个 Processor（I/O）线程

### 9.3 Acceptor 使用 Selector 监听客户端连接请求
1. [ ] 接收到连接后，创建非阻塞 SocketChannel
2. [ ] 将连接以轮询方式分配给某个 Processor 的 newConnections 队列

### 9.4 Processor 线程向Selector注册连接，监听读写事件
1. [ ] 解析请求为 Request 对象
2. [ ] 将请求放入共享的 RequestChannel

### 9.5 KafkaRequestHandler 从 RequestChannel 拉取请求,分发给 KafkaApis 处理

### 9.6 处理完毕后，响应结果写入 Processor 的 responseQueue



## 十 Broker 数据写入和同步流程
### 10.1 Leader写入数据
✅ 消息所在分区的Leader副本收到消息，通过顺序写的方式写入到日志文件，并且更新偏移量offset索引和时间戳索引  
✅ 此时判断确认机制acks = 1, 表示只需要Leader写入成功即可向客户端确认  
✅ 判断 acks=-1，不能立即响应，必须等 ISR 全部副本同步这条消息，创建DelayedProduce延迟任务到时间轮  

### 10.2 Follower副本同步数据
✅ 每一个Follower启动后，都会启动一个FollowerFetchThread线程，专门负责从Leader读取数据，然后进行本地更新  
✅ 该则会自己定期向Leader读取数据，携带当前brokerId、什么分区、replica.fetch.min.bytes最小字节数以及replica.fetch.wait.max.ms最长等待时间  
✅ 读取数据后，写入本地，如果处理同步超过了时间，则会被从ISR列表移除  

### 10.3 Leader处理同步请求
✅ 每一个请求发送给Leader副本所在borker之后，Leader副本会判断是否需要更新高水位线以及当前ISR同步副本列表是不是不包含这个Follower副本  
✅ 如果ISR列表中所有副本都同步完成，Leader 监控所有 ISR 的 LogEndOffset ≥ 当前消息 offset，则会更新高水位线  
✅ 如果该副本跟上了进度，则会加入到ISR列表中  
✅ 如果当前查询的数据不满足最小字节数或者没有数据时候，则Leader副本会创建DelayedFetch延迟任务到时间轮  



## 十一 如何保证消息不丢失
### 11.1 生产者端
#### 11.1.1 消息丢失场景
✅ acks = 0，意味着生产者不需要等待服务端broker返回响应，就认为消息发送成功，那么可能存在消息丢失  
✅ 发送消息失败，未捕获异常进行重试  
✅ RecordAccumulator的buffer.memory内存满导致消息被丢弃,如果没捕获或忽略，消息直接丢失  

#### 11.1.2 解决方案
✅ acks = 1或者 =-1， 至少要保证消息在分区Leader副本上保存成功  
✅ 发送失败，需要进行异常捕获和重试，重试N次不行则一直重试或者转移到其他队列或者保存到本地消息表，进行后续补偿  
✅ 如果客户端崩溃，那么消息在发送之前进行记录，然后消息发送后需要变更状态，如果不存在或者状态未变更说明之前就没发送  

### 11.2 服务器端
#### 11.2.1 消息丢失场景
✅ acks = 0或者1， Leader保存本地消息后，Follower还未同步就崩溃了， 新的Follower成为主节点后这个数据就丢失了，后续的KRaft很好的解决这个问题  
✅ os page cache还未刷到磁盘，然后broker崩溃宕机，即便是配置ack=-1, 那么Leader上的消息也丢失了  
✅ 如果支持不干净的副本选举，Leader 宕机后，Kafka 选择一个不在 ISR 中的副本当新的 Leader, 数据可能丢失  
✅ 磁盘损坏、数据文件损坏  

#### 11.2.2 解决方案
✅ 如果不考虑性能，acks可以设置成-1  
✅ log.flush.interval.messages和log.flush.interval.ms两个参数控制刷盘，log.flush.interval.messages=1可以保证每一条消息刷一次盘  
✅ 禁止不干净的副本参与选举  
✅ 针对磁盘损坏，可以使用RAID或者定期检查磁盘健康，异地容灾备份等  

### 11.3 消费者端
#### 11.3.1 消息丢失场景
✅ 自动提交offset, 那么可能自动提交了，但是消息还没完成，导致消息丢失  
✅ 手动提交如果没消费完，就提交了offset, 消息丢失  
✅ 如果消息处理异常没有捕获或者处理，导致消息丢失  

#### 11.3.2 解决方案
✅ 禁止手动提交offset  
✅ 在业务完成后再进行ack  
✅ 对消息异常进行重试，重试n次不成功，可以写入本地消息表或者转移到其他存储，后续进行补偿  



## 十二 如何保证消息不重复消费
### 12.1 生产者端
1. [ ] 开启幂等发送
2. [ ] 确保消息中携带一些唯一的标记

### 12.2 服务器端

### 12.3 消费者端
1. [ ] 消费端如果是只有一个分区，那么可以根据序列号来处理
2. [ ] 通过Redis或者数据库唯一键来实现幂等



## 十三 如何监控 Kafka 的性能瓶颈？有哪些关键指标？
### 13.1 生产者端指标
1. [ ] 每秒发送消息数量
2. [ ] 每秒发送失败数量
3. [ ] 每秒重试次数
4. [ ] 请求平均延迟

### 13.2 服务端指标
1. [ ] 每秒进入 Kafka 的消息数
2. [ ] 每秒读写的字节流量
3. [ ] Broker 空闲处理线程比率，低表示过载
4. [ ] ISR 列表变化频率，频繁变化说明网络或负载问题
5. [ ] 刷盘速率和耗时，写入延迟的重要指标

### 13.3 消费者端指标
1. [ ] 每秒拉取消息数
2. [ ] 拉取延迟，异常高说明消费慢或服务端堵
3. [ ] 消费者延迟（未消费的消息数），是最直观的滞后指标
4. [ ] 获取写缓存锁的耗时
5. [ ] 刷盘耗时，影响持久化效率



## 十四 Kafka线上问题
### 14.1 kafka-client 升级到2.0升级到2.4 出现CPU100%问题
#### 14.1.1 问题表现
👉 架构部门对基础框架进行升级，包括了kafka-clients，由2.1 升级到 2.4  
👉 订单服务启动时候，频繁出现rebalance日志以及CPU 100%，最后选择回滚  

#### 14.1.2 排查步骤
✅ 第一: 通过jps指令查看进程号  
✅ 第二: 通过jstack指令， 抓线程堆栈: jstack <pid> > jstack.txt  
✅ 第三: 分析堆栈  
* 观察目标线程，发现KafkaConsumer 线程一般卡在 poll()、joinGroup()、rebalance等
* at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator.joinGroupIfNeeded(ConsumerCoordinator.java:...)at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:...)  

✅ 第四: 找出占用 CPU 的线程 ID  
* ps -mp <pid> -o THREAD,tid,time
* printf '%x\n' <TID> 转16进制

✅ 第五: 然后在 jstack 中查找该 TID 对应的线程栈  
* 如果该线程处在 poll() -> joinGroup() -> rebalance 中，就验证了是 Kafka client 内部循环导致 CPU 飙升。

✅ 第六: 发现线程状态是Runnable, 不是 BLOCKED、WAITING，也不是 IO，但CPU100%， 因此怀疑是死循环
✅ 第七: 多次通过jstack间隔打印线程堆栈，发现该线程都是这个状态，因此确定是死循环
✅ 第八: 通过google搜索rebalance 死循环问题，发现确实有2.4版本有死循环BUG，然后我们升级到2.5，发现没问题，最后通知架构部
✅ 第九: 最后排查原因是因为2.4改变了以前的rebalance模式，2.3以前的是: 所有消费者 立刻丢弃所有 partition,然后等新的分配结果,不存在「我还暂时保留旧 partition」的说法, 这样会游戏消息处理中断的问题，所以他们想优化。优化后就是Cooperative Rebalance（协作模式） — Kafka 2.4 默认新策略，client 启动时，如果检测到有 rebalance：不是立刻丢弃所有分区而是 只“部分”释放 coordinator 要它释放的 partition，然后继续消费 未被要求释放的 partition：假设 group 里有 3 个 partition：p0, p1, p2，现在有两个消费者 C1 和 C2。原始分配：C1 拿到 p0, p1C2 拿到 p2现在 C3 加入 group，Coordinator 想这样分配：C1 -> p0C2 -> p1C3 -> p2那么就需要：C1 放弃 p1C2 放弃 p2在 Cooperative 模式下，Coordinator 并不会说：“大家立刻放弃所有 partition”，而是告诉：C1：请你只 revoke 掉 p1C2：请你 revoke 掉 p2C1 手动先发出 rebalance 回调，告诉 Kafka：我放弃了 p1， Kafka Coordinator 等待这些「放弃通知」都收到，再进行下一轮 assign
* 如果某个 client（比如 C2）poll 太慢或crash，它迟迟不调用 onPartitionsRevoked，就不会声明自己放弃 partitionKafka coordinator 检测到有人没放弃，就认为「group 还不稳定」于是再次触发 rebalance，循环开始

🎯 **总结:**   
如果有任何一个 consumer 卡住（未调用 poll / revoke），coordinator 就会一直检测到 “有成员未准备好”，从而再次触发 rebalance


#### 14.1.3 官方怎么修复
✅ 2.6.0: 引入重试上限、避免死循环  
✅ 2.4+: coordinator 引入超时剔除机制:rebalance.timeout.ms 限制 rebalance 时间  
✅ 默认策略改回 RangeAssignor  

### 14.2 消费者配置项问题
#### 14.2.1 问题表现
✅ 消费者服务对应的Kafka Topic Lag超过阈值的报警，于是查看消费者服务的错误日志，发现有如下所示的大量TCP read timeout的错误，且消费者服务几乎不消费了，进而产生了更多Lag超过阈值的报警  
✅ kafka: error while consuming test/1: read tcp <Kafka broker IP>:54953-><Kafka broker IP>:9092: i/o timeout  

#### 14.2.2 排查原因
✅ 首先: 无脑重启消费者服务后发现Lag没有下降，TCP read timeout的错误也没有消失  
✅ 然后: 怀疑是是跨机房消费Kafka，最先想到的是网络丢包引起的问题  
* PING 了Kafka的Broker，发现没有丢包，而且ICMP数据包的时延是100多毫秒，也在正常范围内（以光速通过1万公里都需要33.33毫秒）
* ping -c 100 <目标IP或域名>: 100 packets transmitted, 100 received, 0% packet loss, time 99152ms rtt min/avg/max/mdev = 0.258/0.348/0.451/0.024 ms, 若有非 0 的 packet loss，说明确实存在丢包
* mtr -rwzbc 100 <目标IP或域名> 比PING更详细

✅ 然后: 既然没有丢包，说明网络层OK，那会不会是传输层的问题呢？于是先后运行nc和tcpdump命令
* nc -v -z -w 1 <Kafka broker IP> 9092: Connection to <Kafka broker IP> 9092 port [tcp/*] succeeded! nc命令的结果表示TCP连接建立成功
* sudo tcpdump 'port 9092 and host <Kafka broker IP>' -vvv: 从tcpdump的输出中也能找到三次握手的TCP数据包序列——Flags字段分别为[S]、[S.]和[.]（分别表示SYN、SYN+ACK和ACK），以及收发数据的序列——Flags字段分别为[P.]和[.]（分别表示PUSH和ACK）看起来网络层和传输层都正常，而且负责Kafka的同学反馈集群正常

✅ 最后: 会不会因为单次Fetch请求获取了过多字节数的消息，又因网速慢使得消费者服务读不过来，导致TCP read timeout呢？为了提升跨机房消费Kafka的吞吐量，我们曾增大了Consumer.Fetch.Default这个参数，远比默认值32768要大
* 也就是说，应该是在网速较慢，且Consumer.Fetch.Default这个参数过大时，消费者就会因TCP read timeout而停止消费！如果这个结论成立，那么就应该存在一个临界值，Consumer.Fetch.Default小于这个临界值就（大概率）能正常消费，大于了就（大概率）停止消费了。64kbps这里是小b，所以理论上在30秒内最大传输字节数应该是64 / 8 * 1000 * 30。
