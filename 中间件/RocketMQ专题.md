## 一 核心架构与原理
### 1.1 BrokerServer
#### 1.1.1 Topic
✅ RocketMQ消息也是按照主题分类  

#### 1.1.2 Queue
✅ 为了提升并发，方便扩展，RocketMQ提供了队列，类似于kafka的分区  
✅ 默认一个主题是4个队列，这些队列分散在broker上  

#### 1.1.3 CommitLog
✅ 无论什么主题，什么队列中的消息，最终都会写入到CommitLog  
✅ CommitLog保存了消息来自哪一个topic, queueId是多少，时间戳，消息体  
✅ 一般情况下读取消息不会直接从CommitLog读取消息  

#### 1.1.4 ConsumeQueue
✅ CommitLog存储任何主题任何对列数据，查找数据不方便  
✅ 因此，每一条消息写完之后，会在主题/队列ID/ConsumeQueue文件中记录当前消息在CommitLog中的offset以及消息大小等，相当于是索引  
✅ 消费者会通过 ConsumeQueue 查出消息偏移量，从 CommitLog 中精准地拉取原始数据  
✅ 注意: 每一个主题下每一个队列都有一个ConsumeQueue文件  

#### 1.1.5 IndexFile
✅ 保存消息 key → CommitLog offset 的映射关系  
✅ 用于按 messageKey 或其他属性快速检索消息  


### 1.2 NameServer
✅ 负责Broker Server的注册，每一个Broker都会向所有的NameServer进行注册  
✅ 负责管理broker集群和消息路由  
✅ 管理Broker集群的元数据信息，比如Broker地址、Topic   
✅ 提供心跳检测，检查Broker Server是否存活  
✅ 为生产者和消费者提供路由,生产者和消费者会从多个NameServer中选择一个NameServer进行通信，如果某一个名称服务器NameServer挂了，就会从NameServer集群中选择一个其他的NameServer进行通信  


### 1.3 Producer
✅ 向 NameServer 发送请求，获取当前 Topic 的路由信息  
✅ 创建消息对象 Message  
✅ 选择消息队列，也可以自定义选择逻辑，通过实现MessageQueueSelector接口  
✅ 构造请求并通过 Netty 发送消息  


### 1.4 Consumer
✅ 消费者模式分为集群模式和广播模式：集群模式下，消费者组内的所有消费者共同消费订阅的主题，主题的消息队列根据一定算法分配给组内的消费者；广播模式下，组内所有消费者都会消费订阅的主题  
✅ 消费方式有顺序消费MessageListenerOrderly和并发消费MessageListenerConcurrently，并发数量默认20，可以通过consumeThreadMin和consumeThreadMax进行设置  
✅ 当消费者启动，或者有新的消费者加入，或者有消费者退出，或者消费者组内新增订阅或者取消订阅主题都可能触发消费者rebalance操作，重新分配队列  
✅ 消费者在consumeMessage消费的时候，消费成功返回CONSUME_SUCCESS，失败抛出异常或者返回RECONSUME_LATER，默认是 16 次，可以通过maxReconsumeTimes修改最大重试次数  



## 二 RocketMQ如何保证高性能、高可靠和高科用性的
### 2.1 如何保证可用性的
#### 2.1.1 主从模式
✅ 主从模式下，RocketMQ可以配置一个Master节点，多个Slave节点  
✅ Master 负责接收和存储消息，Slave 仅从 Master 同步数据  
✅ RocketMQ 支持两种主从同步策略: 异步复制,即主节点写入成功即可，不等待 Slave；同步双写Master 会同步把消息写入 Slave只有当 Master 和 Slave 都写入成功，才向 Producer 返回写成功  
✅ 不支持自动主从切换, 需要手动切换  

#### 2.1.2 DLedger模式
🧩 DLedger 模式 下，高可用性是通过 Raft 共识协议 实现的   
🧩 通常需要部署N个节点(N是大于1的奇数) ,每一个节点都需要存储数据，但是只有一个Leader 节点用于接收客户端数据，并且同步数据到从节点  

🎯 **Leader 节点**    
✅ 负责处理所有写请求（如生产者发来的消息）  
✅ 将消息写入本地日志后，向Follower发送Append 请求复制日志  
✅ 等待超过一半节点确认后，Leader提交并返回成功  

🎯 **Follower 节点**  
✅ 接收并保存 Leader 推送的消息   
✅ 参与 Leader 的选举  

🎯 **选举机制**  
✅ 每个节点都有定期的心跳机制  
✅ 如果 Follower 在指定时间未收到 Leader 心跳，则会发起 Leader 选举  
✅ 通过投票机制选出新的 Leader  


### 2.2 如何保证可靠性的
#### 2.2.1 生产者端支持失败重试

#### 2.2.2 服务端
✅ 主从模式下的同步双写可以保证数据不会丢失
✅ DLedger模式也可以保证数据不丢失

#### 2.2.3 消费者端RocketMQ 保证顺序消息主要依靠 “消息队列有序 + 单线程消费”


### 2.3 如何保证高性能的
#### 2.3.1 生产者端
✅ 批量发送，减少 IO 次数，提高带宽利用率  
✅ 消息自动压缩，减少传输体积  
✅ 连接复用 & 长连：使用 Netty 长连接 与 Broker 通信  

#### 2.3.2 服务端
✅ 高效磁盘存储结构（CommitLog）：所有消息顺序写入 CommitLog 文件（物理顺序追加），充分利用 OS 页缓存，避免随机写  
✅ 文件预分配 + 内存映射（Memory-Mapped File）+ 刷盘线程异步落盘  
✅ 零拷贝技术：使用 FileChannel.transferTo() 等方式将文件内容直接从磁盘映射到网络 buffer，避免多次内存拷贝  
✅ 使用索引，提升查询性能  
✅ 使用的高效通信模型：RocketMQ 使用Netty框架进行通信，Netty框架是基于NIO框架，异步非阻塞、零拷贝技术、内存池管理、基于事件驱动模型等技术实现，通信效率较高  

#### 2.3.3 消费端
✅ 批量拉取 + 批量消费  
✅ 并发消费模型  



## 三  RocketMQ 网络通信模型
1. RocketMQ使用Netty构建高性能网络通信模型，采用的是主从 Reactor 模式
2. Broker 启动时会初始化 NettyRemotingServer，其中包括 bossGroup 用于接收客户端连接请求，workerGroup 用于处理网络读写事件
3. 请求数据到达后会经过 Netty 的 ChannelPipeline，在这里会进行协议解码，RocketMQ 使用自定义协议结构 RemotingCommand 来封装请求命令与响应数据
4. 请求一旦解码完成后，会由 NettyServerHandler 根据请求类型分发给对应的 NettyRequestProcessor

## 四 RocketMQ如何实现事务消息的

## 五 如何防止消息丢失
### 5.1 生产端
✅ 第一: 同步发送有自动重试机制，默认重试2次，producer.setRetryTimesWhenSendFailed(2)修改重试次数  
✅ 第二: 异步发送必须手动重试，必须在回调方法 onException() 里手动实现重试逻辑  
✅ 第三: 重试N次失败，可以转移到其他队列，或者通过本地消息表持久化进行补偿  

### 5.2 服务器端
✅ 禁用主从模式下异步刷盘，可能造成数据丢失，最好是主从同步或者DLedger模式  
✅ 最好不要使用读写分离，即写数据会往堆外内存中写，然后异步刷盘，也可能丢失数据  

### 5.3 消费者端
✅ 第一: 消费者只要确保先消费，后提交消费位置即可，否则可能会消息丢失  
✅ 第二: 发生异常后消费者默认会进行重试，重试一定次数还是失败，会将消息扔到死信队列  



## 六 如何防止消息重复消费
### 6.1 生产者端
✅ 生产者端无法像kafka一样实现消息幂等，或者说没有必要做，因为成本比较大  
✅ 可以设置一些业务上唯一的key, 然后消费者端去去重  

### 6.2 服务端
✅ Broker 不做消息去重  

### 6.3 消费者端
✅ 注意使用的消费者组模式，如果是广播模式，所有消费者都会消费  
✅ 消息进行ack的时候，防止消息丢失，建议消费完后手动ack,  但是需要做好幂等，有可能会重复消费，因此可以通过Redis，或者数据库唯一索引等方式进行幂等判断  



## 七 如何监控RocketMQ 的性能瓶颈？有哪些关键指标？
### 7.1 生产者端
1. [ ] 每秒成功的发送数量
2. [ ] 发送失败次数
3. [ ] 发送平均延迟
4. [ ] 重试次数

### 7.2 服务端
1. [ ] 接收 Producer 消息的速率
2. [ ] Consumer 拉取消息的速率
3. [ ] 消息写入 Broker 的耗时（写磁盘）
4. [ ] Broker 磁盘使用情况
5. [ ] 网络处理线程阻塞时间

### 7.3 消费者端
1. [ ] 每秒消费的消息数
2. [ ] 拉取到消息和消费完成之间的耗时
3. [ ] 消息消费失败后的重试次数
4. [ ] 消费者当前滞后消息条数
5. [ ] 拉取失败次数


## 八 线上问题
### 8.1 Page Cache flush现成CPU 使用率高（线程繁忙、100%）

