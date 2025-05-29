## 一 什么是ZAB协议? ZAB协议有哪些角色
### 1.1 什么是ZAB协议
✅ ZAB协议: 就是Zookeeper**原子广播协议**，是一个基于Paxos协议实现的一个分布式一致性协议  
✅ 它主要包括2个阶段: **崩溃恢复阶段** 和 **消息广播阶段**  


### 1.2 ZAB协议有哪些角色
#### 1.2.1 Follower
✅ Follower可以处理读请求  
✅ 不能处理写请求，写请求需要转发给Leader  
✅ 需要将Leader同步过来的事务消息写入本地  
✅ 参与选举  

#### 1.2.2 Leader
✅ 负责处理读写请求  
✅ 如果是写请求，将事务写入本地后，需要同步给其他Follower节点和Observer节点  

#### 1.2.3 Observer
✅ Observer也可以处理读请求  
✅ 不能处理写请求，写请求需要转发给Leader  
✅ 需要将Leader同步过来的事务消息写入本地  
✅ 和Follower区别在于，Observer不能参与选举  

## 二 ZXID是什么
✅ ZXID表示一个事务消息的ID  
✅ ZXID是一个64位的数据，前32位表示选举轮次或者任期electionEpoch, 后32位表示任期内事务消息计数器，每新增一个消息，就会+1  
✅ 选举完成后，就会更新ZXID中的epoch选举轮次  

## 三 ZAB协议如何进行选举
✅ 第一: 先比较ZXID，最大的被选举为Leader, 说明它同步的数据最多
✅ 第二: 如果ZXID相同，则比较SID, 每一个Server都配置的有1个myid，如果ZXID相同，则SID最大的被选举为Leader
✅ 第三: 可以给自己投票
✅ 第四: 超过半数以上的节点认可则认为选举成功


## 四 崩溃恢复阶段流程
### 4.1 什么场景会触发崩溃恢复
✅ 集群刚开始启动  
✅ Leader节点故障  
✅ Leader节点重启  
✅ 网络分区，无法联系Leader  

### 4.2 崩溃恢复流程
#### 4.2.1 重新选举Leader
✅ Follower都会定期向 Leader 发送心跳，如果长时间(tickTime*syncTime)未收到Leader心跳请求，则Follower则开始准备重新选举  
✅ Follower会将自己状态置为LOOKING状态，更新自己的ZXID(增加选举轮次), 然后向其他节点发送投票请求  
✅ 其他Follower节点收到请求后会和自己维护的ZXID进行比较，如果大于自己当前ZXID，则投票；小于或者等于自己的ZXID则不投票  
✅ 当某一个Follower节点收到半数以上节点的响应，则认为选举成功  

#### 4.2.2 Leader 发起日志同步
Leader 检查每个 Follower 的最后ZXID, 和自己的比较
按照情况，分别对每个Follower发
如果Follower的ZXID小于Leader, 则Follower需要补日志，发送DIFF请求
如果Follower的ZXID大于Leader, 则Follower需要删除日志，发送TRUNC请求，回滚超前事务
如果Follower的ZXID和Leader差的太多，或者新加入的FOLLOWER, 此时发送SNAP请求，将数据全量同步给FOLLOWER

#### 4.2.3 完成数据同步，进入消息广播阶段
✅ 多数派 Follower同步完成
✅ 新Leader 才切换到 ACTIVE 状态
✅ 并正式接收客户端写请求，此时进入消息广播阶段


## 五 消息广播阶段流程
🧠 本质上就是正常的读写流程:  
✅ 客户端发起写请求, 如果落在Follower需要转发到Leader  
✅ Leader创建事务提案  
✅ Leader广播事务提案给所有Follower  
✅ Follower写本地 WAL 日志，回复 ACK  
✅ Leader收到多数 ACK，广播 Commit  

## 六 Raft协议和ZAB协议比较
### 6.1 相似点
✅ 都是分布式一致性协议  
✅ 都是强一致性协议  
✅ 都需要过半的票数才能达成一致  
✅ 都需要保证顺序性

### 6.2 不同点
📌  **目的不同**
1. [ ] ZAB: 专门为Zookeeper设计
2. [ ] Raft: 是通用的分布式数据一致性协议

📌  **角色不一样**
1. [ ] ZAB协议: Leader、Follower、Observer
2. [ ] Raft协议: Leader、Follower、Candidate

📌  **Follower职责不一样**
1. [ ] ZAB: Follower还可以处理读请求
2. [ ] Raft: 只能由Leader处理读写请求

📌  **选举时候对过半节点要求不一样**
1. [ ] ZAB: ZAB协议是过半的Follower, 不包含Observer
2. [ ] Raft: 过半的Follower节点就行

📌  **选举机制不同**
1. [ ] ZAB: Follower先将自己变为LOOKING状态，然后向其他节点发起投票，投票原则是先比较zxid，如果zxid相同则比较myid
2. [ ] Raft: ollower会将自己先置为Candidate角色，然后向其他节点发起投票，如果投票数未能超过半数以上，那么选举超时之后，最先超时的节点，会增加选军轮次，重新发起投票

📌  **协议是否有快照**
1. [ ] ZAB: ZAB协议本身没有快照（Zookeeper在实现的时候提供了）
2. [ ] Raft: Raft协议有快照，对某时刻状态机中数据进行快照，减少磁盘容量的使用

