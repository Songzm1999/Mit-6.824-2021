​	相比于Paxos，Raft最大特性就是易于理解，Raft主要做了两方面的事情：

- **问题分解：**把共识算法分为三个子问题，分别是领导者选举(leader election)、日志复制(log replication)、安全性(safety)
- **状态简化：**对算法做出一些限制，减少状态数量和可能产生的变动

# 1.复制状态机

复制状态机的核心理念是：相同的初始状态+相同的输入=相同的结束状态。

简单一点描述就是：多个节点上，从相同的初始状态开始，执行相同的一串命令，产生相同的最终状态。

Raft中，leader将client请求（command）封装到log entry中，将这些log entries复制到所有的follower节点中，大家按相同顺序应用log entries中的command

![](https://github.com/Songzm1999/Mit-6.824-2021/blob/master/images/Raft-F1.png)

# 2.状态简化

任何时刻，每一个服务器节点都处于**leader、follower、candidate**这三个状态之一

不用像Paxos那样考虑状态之间的共存和互相影响

![](https://github.com/Songzm1999/Mit-6.824-2021/blob/master/images/Raft-F4.png)



Raft把时间分割成任意长度的**任期（term）**，任期采用连续的整数标记

在每次任期以一次选举开始，如果一次选举没有选出leader（两个节点相同票数、votes少于总数一半），这一任期会以没有leader结束，并开始新的任期

![](https://github.com/Songzm1999/Mit-6.824-2021/blob/master/images/Raft-F5.png)



- Raft算法中服务器节点之间使用RPC进行通信，并且Raft中只有两种主要的RPC：
- **RequestVote RPC（请求投票）**：由candidate在选举期间发起。
- **AppendEntries RPC（追加条目）**：由leader发起，用来复制日志和提供一种心跳机制。
- 服务器之间的通信会交换当前任期号；如果一个服务器上的当前任期号比其他的小，该服务器会将自己的任期号更新为较大的那个值
- 如果一个candidate或者leader发现自己的任期号过期了，会立即回到follower状态
- 如果一个节点收到一个包含过期的任期号请求，他会直接拒绝这个请求



# 3.领导者选举

首先是领导者选举（leader election）。Raft内部有一种心跳机制，如果存在leader，那么它就会周期性地向所有follower发送心跳，来维持自己的地位。如果follower一段时间没有收到心跳，那么他就会认为系统中没有可用的leader了，然后开始进行选举。

开始一个选举过程后，follower先增加自己的当前任期号，并转换到candidate状态。然后投票给自己，并且并行地向集群中的其他服务器节点发送投票请求。<mark>最终会有三种结果：</mark>

- ①它获得超过半数选票赢得了选举 -> 成为主并开始发送心跳

- ②其他节点赢得了选举 -> 收到新leader的心跳后，如果新leader的任期号不小于自己当前的任期号，那么就从candidate回到follower状态。

- ③一段时间之后没有任何获胜者 -> 每个candidate都在一个自己的随机选举超时时间后增加任期号开始新一轮投票。
- 论文中给出的随机选举超过时间为150-300ms



```go
//请求投票的RPC Request
type RequestVoteRequest struct{
	term		int		//自己当前任期号
	candidateId	int		//自己的ID
	lastLogIndex int	//自己最后一个日志号
	lastLogTerm	int		//自己最后一个日志的任期
}

//请求投票RPC Response
type RequestVoteResponse struct{
    term 		int		//自己当前任期号
    voteGranted	bool	//自己会不会投票给这个candidate
}
```



**对于没有成为candidate的follower节点，对于同一个任期，会按照先来先得的原则投票**



# 4.日志复制

一条日志中需要三个信息：

- 状态机指令
- leader的任期号
- 日志号（日志索引）

![](https://github.com/Songzm1999/Mit-6.824-2021/blob/master/images/Raft-F6.png)



Leader接收到客户端的指令后，会把指令作为一个新的条目追加到日志中去，然后并行发送给follower，让它们复制该条目。**当该条目被超过半数的follower复制后**，leader就可以**在本地执行**该指令并把结果返回客户端。

**提交**：我们把本地指令执行，也就是leader应用日志于状态机这一步称为提交。（即使追加成功、并且被半数follower复制，如果在这时宕机，也可能没有提交）



在此过程中，leader或follower随时都有崩溃或缓慢的可能性，**Raft必须要在有宕机的情况下继续支持日志复制，并且保证每个副本日志顺序的一致**（以保证复制状态机的实现）：

**<mark>①如果有follower因为某些原因没有给leader响应</mark>**，那么leader会不断地重发追加条目请求，哪怕leader已经回复了客户端。

**<mark>②如果有follower崩溃后恢复，</mark>**这时Raft追加条目的一致性检查生效，保证follower能按顺序恢复崩溃后的缺失的日志。

<mark>Raft的一致性检查：</mark>leader在每一个发往follower的追加条目RPC中，会放入前一个日志条目的搜索引位置和任期号，如果follower在它的日志中找不到前一个日志，那么它就会拒绝此日志，leader收到follower的拒绝后，**会发送前一个日志条目，从而逐渐向前定位到follower第一个缺失的日志。**

**<mark>③如果leader崩溃，</mark>**那么崩溃的leader可能已经复制了日志到部分follower但还没有提交，而被选出的新leader又可能不具备这些日志（因为没有提交，所以不违背下面的安全性规则），这样就有部分follower中的日志和新leader的日志不相同。Raft在这种情况下，**leader通过强制follower复制它的日志来解决不一致的问题，这意味着follower中跟leader冲突的日志条目会被新leader的日志条目覆盖（因为没有提交，所以不违背外部一致性）。**

![](https://github.com/Songzm1999/Mit-6.824-2021/blob/master/images/Raft-F7.png)

例如，在上图中，当第一条成为leader后(可以由自身、a、b、e、f投票产生)，a-f都可能发生，c-f都会被强制要求复制成leader的日志



- 通过这种机制，leader在当权后不需要任何特殊的操作使日志恢复到一致状态。
- leader只需要进行正常操作，日志就能在回复AppendEntries一致性检查失败时候自动趋于一致
- leader从来不会覆盖或者删除自己的日志条目

```go
//追加日志RPC Request
type AppendEntriesRequest struct{
    term			int		//自己当前的任期号
    leaderId		int		//leader的ID
    preLogIndex		int		//前一个日志的日志号
    preLogTerm		int		//前一个日志的任期号
    entries			[]byte	//当前日志体
    leaderCommit	int		//leader已提交日志号
}

//追加日志的RPC Response
type AppendEntriesResponse struct{
    term		int		//自己当前任期号
    success		bool	//如果follower包括前一个日志，返回true
}
```

如果leaderCommit > commitIndex，那么把commitIndex设为min(leaderCommit ,index of last new entry)

leader复制日志——> follower复制日志——> leader确认follower收到日志，应用日志到状态机——> follower根据leaderCommit标号来提交自己的日志



# 5.安全性

为了完全保证每一个状态机都会按照相同的顺序执行相同的命令，Raft补充了几个规则：

## 5.1 leader宕机处理：选举限制

如果一个follower落后了leader若干条日志（没有遗漏一整个任期），那么在下次选举中，按照领导者选举里的规则，它依旧有可能当选leader。它在当选新leader后就无法补上之前缺失的那部分日志

**所以这里需要对领导者选举增加一个限制，保证被选出来的leader一定包含了之前各任期的所有被提交的日志条目**

RequestVote RPC执行了这样的限制：RPC中包含了candidate的日志信息，**<mark>如果投票者自己的日志比candidate的还新，它会拒绝掉该投票请求。</mark>**

```go
//请求投票的RPC Request
type RequestVoteRequest struct{
	term		int		//自己当前任期号
	candidateId	int		//自己的ID
	lastLogIndex int	//自己最后一个日志号
	lastLogTerm	int		//自己最后一个日志的任期
}
```

Raft通过比较两份日志中最后一条日志条目的索引值和任期号来定义谁的日志比较新。

- 如果两份日志最后条目的任期号不同，那么任期号大的日志更“新”。
- 如果两份日志最后条目的任期号相同，那么日志较长的那个更“新”。



## 5.2 leader宕机处理：新leader是否提交之前任期内的日志条目

如果每个leader在提交某个日志条目之前崩溃了，以后的leader会试图完成该日志条目的复制，而非提交

- <mark>**Raft永远不会通过计算副本数目的方式来提交之前任期内的日志条目**</mark>
- **<mark>只有自己任期内的日志才能够通过计算副本数目的方式提交</mark>**
- 一旦当前任期的某个日志以这种方式被提交，由于日志匹配特性，之前所有日志条目也都会被简介提交



## 5.3 Follower和Candidate宕机处理

Follower和Candidate崩溃后的处理方式要简单一些，并且两者的处理方式相同：

- 如果Follower或Candidate崩溃了，那么后续发给他们的RequestVote和AppendEntries RPC都会失效
- RPC通过无限期的重试来处理这种失败，如果崩溃的机器重启了，那么这些RPC就会成功
- 如果一个服务器在完成了一个RPC，但是还没有相应的时候崩溃了，那么他重启之后会再次收到相同的请求。



## 5.4 时间与可用性限制

raft算法整体不依赖客观时间，也就是说，哪怕因为网络或其他因素，造成后发的RPC先到，也不会影响Raft的正确性

只要整个系统满足下面的时间要求，Raft就可以选举并维持一个稳定的leader：

<mark>广播时间(broadcastTime) << 选举超时时间(electionTImeout) << 平均故障时间(MTBF)</mark>

广播时间和平均故障时间是由系统决定的，但是选举超时时间是自己选择的，Raft的RPC需要接受并将信息落盘，所有广播时间大约是0.5ms到20ms，取决于存储的技术。因此，选举超时时间可能需要在10ms到500ms之间，大多数服务器的平均故障时间都在几个月甚至更长。



# 6.集群成员变更

# 7.总结与性能测试

## 7.1 深入理解复制状态机

把复制状态机需要同步的数据量按大小进行分类，它们分别适合不同类型的共识算法

- 数据量非常小，如集群成员信息、配置文件、分布式锁、小容量分布式任务队列
  - 无leader的共识算法（如Basic Paxos），实现有Chubby、Zookeeper等
- 数据量比较大大可以拆分成为不相干的各部分，如大规模存储系统
  - 有leader的共识算法（如Multi Paxos，Raft）实现有GFS、HDFS
- 不仅数据量大，数据之间还存在关联
  - 一个共识算法集群容纳不了所有的数据，这种情况下，就要把数据分片（partition）到多个状态机中，状态机之间通过两阶段提交来保证一致性
  - 这类场景主要就是一些如Spanner、OceanBase、TiDB等支持分布式事务的分布式数据库

## 7.2 Raft基本概念总结

共识算法的三个主要特性：

- 共识算法可以保证任何在非拜占庭情况下的正确性
  - 通常来说，共识算法可以解决网络延迟、网络分区、丢包、重复发送、乱序问题，无法解决拜占庭问题（如存储不可靠、消息错误）
- 共识算法可以保证大多数机器正常情况下集群的高可用性，而少部分机器缓慢不影响整个集群的性能
- 不依赖外部时间来保证日志的一致性
  - 这一点既是共识算法的优势，因为共识算法不受硬件影响，不会因外部因素造成错误，但也造成了一些限制，让共识算法受网络影响很大，在异地容灾场景下，共识算法的支持性比较差



**no-op 补丁**

- 一个节点当选leader后，立刻发送一个自己任期的空日志体的AppendEntries RPC。这样，就可以把之前任期内满足条件的日志都提交了
- 一旦no-op完成复制，就可以把之前任期内符合提交的日志保护起来了，从而可以使他们安全提交。因为没有日志体，这个过程应该是很快的
- 目前大部分应用于生产系统的raft算法，都是启用no-op的

## 7.3 集群成员变更

## 7.4 日志压缩机制

随着Raft的不断运行，log不断累积，需要一个机制安全的清理状态机上的log

- Raft采用一种快照技术，每个节点在达到一定条件之后，可以把日志中的命令都写入自己的快照，然后可以把已并入快照的日志删除
- 快照中一个key只会保留最新的一份value，占用空间比日志小得多
- 如果一个follower落后leader很多，如果老的日志被清理了，leader怎么同步给follower呢？
  - raft的策略是直接向follower发送自己的快照

![](https://github.com/Songzm1999/Mit-6.824-2021/blob/master/images/Raft-F12.png)



## 7.5 只读操作处理

- 直观上讲，raft的读只要直接读取leader上的结果就行了。
- 直接从leader的状态机取值，实际上并不是线性一致性读（一般也称作强一致性读）。
- 我们对**线性一致性读**的定义：读到的结果要是读请求发起时已经完成提交的结果（快照）。
- 在leader和其他节点发生了网络分区情况下，其他节点可能已经重新选出了一个leader,而如果老leader在没有访问其他节点的情况下直接拿自身的值返回客户端，这个读取的结果就有可能不是最新的。



·要追求强一致性读的话，就需要让这个读的过程或结果，也在大多数节点上达到共识。

·稳妥的方法：把读也当做一个log,由leader发到所有的所有节点上寻求共识，这个读的log提交后，得到的结果是一定符合线性一致性的。

**·<mark>优化后的方法，要符合以下规则：</mark>**

1. **线性一致性读一定要发往leader.**

2. **如果一个leader在它的任期内还没有提交一个日志，那么它要在提交了一个日志后才能反馈**client读请求。（可以通过no-op补丁来优化这一点）

  因为只有在自己任期内提交了一个日志，leader才能确认之前任期的哪些日志已被提交，才不会出现已提交的数据读取不到的情况。

3. **优化过后的线性一致性读，也至少需要一轮RPC(leader确认的心跳）**。并不比写操作快多少（写操作最少也就一轮RPC).

·所以，还可以更进一步，因为读的这轮RPC仅仅是为了确认集群中没有新leader产生。那么如果leader上一次心跳发送的时间还不到选举超时时间的下界，集群就不能选出一个新leader,那么这段时间就可以不经过这轮心跳确认，直接返回读的结果。（但不建议使用这种方法）

·如果不要求强一致性读，怎么样利用follower承载更大的读压力呢？

- ·①follower接收到读请求后，向leader请求readlndex.
- ·②follower等待自身状态机应用日志到readlndex.
- ·③follower查询状态机结果，并返回客户端。



















# 8.拓展：ParallelRaft
