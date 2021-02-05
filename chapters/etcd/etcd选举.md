[toc]

### leader选举

Raft协议中，一个节点有三个状态：Leader、Follower和Candidate，但同一时刻只能处于其中一种状态。Raft选举实际是指选举Leader，选举是由候选者（Candidate）主动发起，而不是由其它第三者。

并且约束只有Leader才能接受写和读请求，只有Candidate才能发起选举。*如果一个Follower和它的Leader失联（失联时长超过一个Term），则它自动转为Candidate，并发起选举*。

发起选举的目的是Candidate请求（Request）其它**所有**节点投票给自己，如果Candidate获得多数节点（a **majority** of nodes）的投票（Votes），则自动成为Leader，这个过程即叫**Leader选举**。

在Raft协议中，正常情况下Leader会周期性（不能大于Term）的向所有节点发送**AppendEntries** RPC，以维持它的Leader地位。

相应的，如果一个Follower在一个Term内没有接收到Leader发来的AppendEntries RPC，则它在延迟随机时间（150ms~300ms）后，即向所有其它节点发起选举。

采取随机时间的目的是避免多个Followers同时发起选举，而同时发起选举容易导致所有Candidates都未能获得多数Followers的投票（脑裂，比如各获得了一半的投票，谁也不占多数，导致选举无效需重选），因而延迟随机时间可以提高一次选举的成功性。



​	选举是 raft 共识协议的重要组成部分，重要的功能都将是由选举出的 leader 完成。不像 Paxos，选举对 Paxos 只是性能优化的一种方式。选举是 raft 集群启动后的第一件事，没有 leader，集群将不允许任何的数据更新操作。选举完成以后，集群会通过心跳的方式维持 leader 的地位，一旦 leader 失效，会有新的 follower 起来竞选 leader。

​		选举的发起，一般是从 Follower 检测到心跳超时开始的，v3 支持客户端指定某个节点强行开始选举。选举的过程其实很简单，就是一个 candidate 广播选举请求，如果收到多数节点同意就把自己状态变成 leader。

​		当集群已经产生了 leader，则 leader 会在固定间隔内给所有节点发送心跳。其他节点收到心跳以后重置心跳等待时间，只要心跳等待不超时，follower 的状态就不会改变。

### 日志复制

在 raft 集群中，所有日志都必须首先提交至 leader 节点。leader 再向 follower 同步日志，follower 在收到日志之后向 leader 反馈结果，leader 在确认日志内容正确之后将此条目提交并存储于本地磁盘。

WAL (Write-ahead logging)，是用于向系统提供原子性和持久性的一系列技术。在使用 WAL 的系提供中，所有的修改在提交之前都要先写入 log 文件中。etcd 的 WAL 由日志存储与快照存储两部分组成，其中 Entry 负责存储具体日志的内容，而 Snapshot 负责在日志内容发生变化的时候保存 raft 的状态。WAL 会在本地磁盘的一个指定目录下分别日志条目与快照内容。



日志复制由leader复制到follower. 超过半数写入成功才算成功。



[《深入浅出 etcd》part 2 – 解析 etcd 的心跳和选举机制](https://www.infoq.cn/article/Y2lv9myMjC9hNISiAehT)

