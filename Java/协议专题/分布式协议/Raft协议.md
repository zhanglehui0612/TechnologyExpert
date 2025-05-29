## 一 什么是Raft协议，有哪些角色
### 1.1 什么是ZAB协议
✅ Raft协议是一个通用的分布式一致性协议  


### 1.2 ZAB协议有哪些角色
#### 1.2.1 Follower
✅ Raft协议中节点启动默认是Follower角色  
✅ Follower角色负责同步Leader发送过来的数据写入本地   
✅ 参与选举  

#### 1.2.2 Leader
✅ Leader节点负责读写请求，客户端的所有请求都是通过Leader节点进行的  
✅ Leader节点会定期向Follower发送心跳，告知Follower自己还活着   

#### 1.2.3 Candidate
✅ 当Follower节点要开始选举的时候，则会将自己状态变为Candidate，表示正在竞选Leader  

## 二 Raft协议是如何进行选举的?
### 2.1 **选举超时时间**(Election Timout)和**心跳超时时间**(HeartBeat Timeout)以及任期(Term)
#### 2.1.1 心跳超时时间
✅ Leader定期主动发送心跳到Follower, 如果在超过一定时间没有收到来自Leader节点的心跳请求，就认为Leader故障，需要触发重新选举

#### 2.1.2 选举超时时间
✅ 同时可能有多个候选节点参与选举，导致选票被瓜分，没有一个候选节点可以获取超过半数以上的投票，导致一致无法选举出来主节点  
✅ 如果无法完成选举，通过设置一个随机的选举超时时间，那么先超时到期的节点会增加选举轮次term，重新发起投票  

### 2.2 选举流程
✅ Follower节点将自己转变成Candidate角色  
✅ Candidate候选节点增加选举轮次，将自己的term选举轮次+1，然后向其他节点发送投票请求  
✅ 其他节点判断选举轮次是否大于自己的选举轮次term，如果大于则更新选举轮次，然后给该节点投票; 如果小于或者等于是不会投票的  
✅ 多个候选同时发起选举，可能存在瓜分选票的问题，这时候可以根据选举超时时间进行控制，先到期的重新发起选举  
✅ 选举成功，Candidate转变为Leader，Leader开始向Follower按照固定间隔发送心跳包  


## 三 Raft协议工作机制
✅ 客户端将请求发送到集群，由Leader处理写请求；和ZAB协议不同，默认情况下，Follower是不接收任何读写请求的，全部是Leader接收  
✅ Leader将客户端数据封装成Append LogEntry, 写在本地日志序列, 写入本地成功，开始向Follower发送RPC请求，进行日志同步  
✅ Follower收到Leader的Append Entry的请求，将数据写入本地日志序列，然后对Leader进行ack响应  
✅ Leader收到半数以上的Follower的ack响应，认为该Log Entry已经被提交，此时会将写入的Append LogEntry应用到本地状态复制机  
✅ Leader向所有的Follower发Commit消息，告诉他们这个Append LogEntry已经被提交  
✅ Follower收到Leader的Commit消息则开始将日志应用到自己本地的状态机中  
✅ Leader收到超过半数以上Follower的确认，认为写入数据成功，开始返回客户端  

## 四 Raft协议优化
### 4.1 Raft协议网络分区是怎么处理多个Leader并存的情况的
✅ 当发生网络分区的时候，可能会触发新的Leader选举，此时集群可能存在2个Leader节点  
✅ 当网络分区恢复后，两个Leader就会互相给对方发送心跳  
✅ 当某一个Leader发现自己的选举轮次低于另外一个Leader，则会辞去Leader角色  
✅ 此时会将部分数据回滚，然后以Follower角色重新加入到集群中，进行数据同步  

### 4.2 写请求太多，Leader节点压力太大，怎么优化？
#### 4.2.1 背景: 单Raft Leader 容易成为瓶颈
🤔Raft 强一致协议要求所有写请求都要经过 Leader  
🤔如果一个系统所有写操作都由单个Raft集群处理，Leader会承受全部写入、复制、提交、磁盘落盘、快照的压力  
🤔当写入量很大时，Leader 会成为系统瓶颈，CPU、I/O、带宽都可能饱和  

#### 4.2.2 Multi-Raft 分组机制
✅ 将单个Raft拆分成多个Raft 分组  
✅ 每一个Raft分组是一个单独的Raft集群，需要有自己单独的Leader  
✅ 需要将数据进行路由，可以根据数据的hashcode 、范围分片又或者其他算法路由到不同的Raft分组  
✅ 可以搞一个Raft 代理，只负责进行请求或者数据路由，也可以直接在Raft客户端做这个事情  


## 五 Raft协议和ZAB协议比较
### 5.1 相似点
✅ 都是分布式一致性协议  
✅ 都是强一致性协议  
✅ 都需要过半的票数才能达成一致  
✅ 都需要保证顺序性  

### 5.2 不同点
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





