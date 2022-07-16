# Lab2

## 前言

在MIT 6.824 的Lab2实验中，需要实现的是一个Raft算法，包括领导者选举、发送日志、持久化数据、快照截取四个部分。在实验之前建议一定要把Raft论文Figure 2 的每个细节都搞明白。

这里推荐一个讲解视频：https://www.bilibili.com/video/BV1pr4y1b7H5?spm_id_from=333.337.search-card.all.click

另外还有一个模拟Raft算法的网站：https://raft.github.io/raftscope/index.html，网站可以模拟丢包、超时、宕机等，如果对论文有什么问题多模拟几遍也可以加深理解。

## 概述

首先要介绍一下Raft的数据结构：

```go
type Raft struct {
    mu        sync.Mutex          // Lock to protect shared access to this peer's state
    peers     []*labrpc.ClientEnd // RPC end points of all peers
    persister *Persister          // Object to hold this peer's persisted state
    me        int                 // this peer's index into peers[]
    dead      int32               // set by Kill()

    //persistent state on all servers
    CurrentTerm int
    VotedFor    int
    Log         []LogEntry

    //volatile state on all servers
    CommitIndex   int    // 需要提交的日志Index
    LastApplied   int    // 已经提交的日志Index
    LastSnapIndex int    // 上次快照执行到的日志Index
    LastSnapTerm  int    // 上次快照执行到的日志Term
    SnapShotData  []byte // 上次快照的数据，用于向Folloer发送

    //volatile state on leaders
    NextIndex  []int
    MatchIndex []int

    RaftState             RAFTSTATE
    ElectionTimeoutTicker *time.Ticker  // 超时重新选举计时器
    ApplyCh               chan ApplyMsg // 应用日志的通道
    HeartBeatTimer        []*time.Timer // Leader发送HeartBeat的定时器

    logFile *os.File
    logger  *log.Logger
}
```

其中我觉得有几组是需要重点理解的：

- **CommitIndex和LastApplied**：commit是需要提交的，applied是已经提交的，换句话说，applied <= commit <= 最后一条日志 
- **NextIndex和MatchIndex**：nextIndex是下一次发送给Follower的index，match是已经匹配的，在绝大多数情况下，NextIndex = MatchIndex + 1，但是在官方给的[手册](https://thesquareplanet.com/blog/students-guide-to-raft/)中，不建议这样做，而是 matchIndex = prevLogIndex + len(entries[])，在我的实现中，NextIndex是用来向Follower发送日志的，MatchIndex是用来更新Leader的CommitIndex，MatchIndex只有在成功向Follower发送日志后才会更新
- **ElectionTimeoutTicker和HeartBeatTimer：**ElectionTimeoutTicker是单独的一个ticker，用来通知选举超时事件，后者是一个数组，通过监听每个timer，当事件触发时并且Raft状态是Leader，则会向Follower发送相应的心跳。

## Lab2A

Lab2A 实现的基本功能就是要能选举出一个Leader，**这部分可以分为三个逻辑：**

- **超时选举：**通过启动时启动一个定时器，超时时间有两种触发条件：

  - 收到Leader的心跳或者日志
  - 收到其他成员的投票请求

  重置超时时间采用随机数，论文中是150ms-300ms。

  超时后向其他成员发送投票请求，如果获得半数的投票，则成为Leader

- **发起投票请求：**依次以单独线程方式发送投票请求

  如果返回结果的Term大于自己的Term，则要返回到Follower状态，并且更新任期号为最新的

  如果成功，统计投票结果，获得半数的投票，则成为Leader

- **收到投票请求：**处理逻辑由下图表示，有两点需要注意的：

  - 需要判断Candidate的Term和自身的Term，如果Candidate的Term小，是不能投票的
  - 只有当Raft的状态是Follower、当前任期没有投过票、Candidate的日志要更新(论文5.4.1节有关于日志更新的解释)才能投票

  ![](https://github.com/Songzm1999/Mit-6.824-2021/blob/master/images/lab2-RequestVote.jpg)

  

## Lab2B

lab2b要实现的就是接受日志、发送给Follower、应用日志，日志的结构体如下：

```go
type LogEntry struct {
	Term     int				// 任期号
	Index    int				// 从启动开始的下标
	Commands interface{}		// 指令
}
```

在lab2b中我的实现可以分为五个部分：Leader接受客户端日志(Start())、Leader向Follower发送日志(sendAppendEntries)、Follower接受Leader的日志(AppendEntries)、应用日志(applyLogs)、Leader检查commitIndex(checkCommitIndex)

### Leader接受客户端日志

这部分在代码中的函数就是Start函数，逻辑处理比较简单，就是从上层接收到一个日志后，检查是否是Leader，如果是的话，追加到日志列表末尾，然后向Follower发送即可。

### Leader向Follower发送日志

向Follower发送日志的逻辑如下：

![](https://github.com/Songzm1999/Mit-6.824-2021/blob/master/images/lab2-sendAppendEntries.jpg)

需要注意的点如下：

- 该函数的触发有两个地方
  - Leader收到一个日志，需要向Follower同步
  - heartbeat超时，发送一个心跳

  需要首先检查是否是Leader，如果不是退出即可

- 调用一个RPC，可能会失败（这一点会在Lab2C中模拟一个不稳定的网络），因此失败时需要重传，但是如果该Raft断网的话，可能会造成过多的goroutine，因此需要有一个超时机制限制。包括lab2A中的RequestVote RPC以及lab2D中的 InstallSnapshot RPC都需要实现重传和超时的机制，可以用下面这个模板：

  ```go
  RPCTimer := time.NewTimer(RPCTimeout)
  defer RPCTimer.Stop()
  
  for rf.killed() == false {
      rf.mu.Lock()
      // 封装args
      rf.mu.Unlock()
  
      resCh := make(chan bool, 1)
  
      go func(args *AppendEntriesArgs, reply *AppendEntriesReply) {
          ok := rf.peers[server].Call("Raft.AppendEntries", args, reply)
          if !ok {
              time.Sleep(time.Millisecond * 10)
          }
          resCh <- ok
      }(&args, &reply)
  
      select {
          case <-RPCTimer.C:
          // 超时退出循环
          break
          case ok := <-resCh:
          if !ok {
              //发送失败，重传
              continue
          }
      }
  }
  ```

- RPC成功后，reply的返回值如果是失败的话，有两种情况

  - 当前Term小于Follower的Term
  - args中的preIogIndex或者prelogTerm不匹配，需要更新nextIndex重传，论文中是前移一位，但是在lab2C中需要对这部分优化

### Follower接受Leader的日志

这部分逻辑也比较简单：

1. 判断Leader 的Term和自身的Term大小关系，如果小的话，证明Leader过期了，返回false
2. 判断args中PreLogIndex和PreLogTerm是否和自己Logs匹配，匹配成功的话追加到PreLogIndex后面，否则返回false
3. 日志可能为空，这时候也要判断PreLogIndex，提醒Leader自己没有全部日志，需要Leader重发

### 应用日志

这部分在每个Raft都应该作为一个goroutine和程序共存亡，它需要循环的检查commitIndex和LastApplied的大小关系，如果有需要提交的日志，则放入到ApplyCh中，供上层service执行命令。

**唯一需要注意的是，在将log放入ApplyCh的时候不能有持有锁，否则会在Lab2D中的测试中出现问题。**

### Leader检查commitIndex

这部分同应用日志一样，应该作为一个单独goroutine不断循环检查，但是需要加一个是否是Leader的判断条件。它检查MatchIndex[]来确认那些日志已经获得集群一半的回复，从而更新commitIndex来提交日志。



## Lab2C

2C需要实现的是状态的持久化，在论文中，Raft的状态是需要持久化在硬盘上的，而在试验中，使用一个Persister来实现硬盘的功能，我们要做的就是把数据放入到Persister中，2C本身并不难，实现两个函数即可：

```go
func (rf *Raft) persist() {
	data := rf.getPersistData()
	rf.persister.SaveRaftState(data)
}

func (rf *Raft) getPersistData() []byte {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(rf.CurrentTerm)
	e.Encode(rf.VotedFor)
	e.Encode(rf.CommitIndex)
	e.Encode(rf.LastApplied)
	//e.Encode(rf.SnapSize)
	e.Encode(rf.LastSnapIndex)
	e.Encode(rf.LastSnapTerm)
	e.Encode(rf.SnapShotData)
	e.Encode(rf.Log)
	return w.Bytes()
}

//
// restore previously persisted state.
//
func (rf *Raft) readPersist(data []byte) {
	if data == nil || len(data) < 1 { // bootstrap without any state?
		return
	}
	r := bytes.NewBuffer(data)
	d := labgob.NewDecoder(r)
	var currentTerm, voteFor, commitIndex, lastApplied, lastSnapIndex, lastSnapTerm int
	var logs []LogEntry
	var snapshotData []byte
	if d.Decode(&currentTerm) != nil ||
		d.Decode(&voteFor) != nil ||
		d.Decode(&commitIndex) != nil ||
		d.Decode(&lastApplied) != nil ||
		d.Decode(&lastSnapIndex) != nil ||
		d.Decode(&lastSnapTerm) != nil ||
		d.Decode(&snapshotData) != nil ||
		d.Decode(&logs) != nil {
		rf.logger.Printf("Read Persisit got an error....\n")
	} else {
		rf.CurrentTerm = currentTerm
		rf.VotedFor = voteFor
		rf.CommitIndex = commitIndex
		rf.LastApplied = lastApplied
		rf.LastSnapIndex = lastSnapIndex
		rf.LastSnapTerm = lastSnapTerm
		rf.SnapShotData = snapshotData
		rf.Log = logs
	}
}
```

这里根据自己设计的Raft数据结构，将必须持久化的数据执行即可，然后在程序执行过程中，这些数据发生变化时，执行persist()函数即可。

2C的本身逻辑不难，但是在测试中会对之前的一些逻辑进行测试，因此2C是我debug时间最长的，主要卡在了`TestFigure8Unreliable2C`这个测试上，该测试模拟了一个不稳定的网络环境，每个Raft可能掉线，Raft发出的包随机丢包，还发送了1000条指令，最终验证每个raft状态是否一致，主要改进的地方有两点：

1. **改进nextIndex的更新方式**

   在论文中关于PreLogIndex不匹配时提出的解决方案是逐步前移，而如果一个Leader领先Follower几百几千条，而且又是在一个不稳定的环境中，这样方式几乎是不可能的，我最初完全是按照论文来的，在这里卡了3、4天，没办法放弃了，搜索了一些网上的解决方案，是修改`AppendEntriesReply`的返回值：

   ```go
   type AppendEntriesReply struct {
   	Term          int
   	Success       bool
   	ConflictIndex int
   }
   ```

   网上有一些解决方案是添加ConflictIndex和ConflictTerm来通知Leader，我这里处理是如果出现不匹配的情况，直接将ConflictIndex赋值为commitIndex，Leader也从Follower的提交日志下一步开始重传，这可能会多传送一些日志，但大大简化了处理。

2. **RPC失败后重传**

   增加了第一步的改进后，测试还是不能通过，通过检查日志可以得出，更新了几十个Term后也无法选出一个Leader，Leader也无法发送心跳，这就导致整个集群不断有Raft超时重选，造成了动荡，改进方法就是如果RPC没有收到回应，就会重传，直到超时或者收到回复，这一步在2B中已经解释过



## Lab2D

在2D中，为了避免日志过长，因此需要对Raft和上层服务数据进行快照，在2021年的实验中，主要实现为四个函数：`CondInstallSnapshot`、`Snapshot`、`InstallSnapshot`、`sendInstallSnapshot`

后两个是Follower和Leader之间的RPC，用于向落后的Follower发送快照，前两个均有上层服务触发，两个有些许不同，

`Snapshot`当service发现日志过长时会通知Raft截取快照

`CondInstallSnapshot`是又InstallSnapshot将快照数据发送到ApplyCh后，上层检查过后再发送到raft安装快照

至于为什么这么做，可以参考[链接](https://www.cnblogs.com/sun-lingyu/p/14591757.html)，我谈一下我的理解：

InstallSnapshot和AppendEntries是独立的，现在先**假设我们可以直接通过InstallSnapshot安装快照。**

InstallSnapshot和应用普通日志都会涉及对Raft的修改，因此都需要持有锁。通过阅读测试源码，我们可以发现当上层service收到一条普通日之后会检查日志长度，如果超过阈值时会调用Snapshot来执行快照，而Snapshot中需要对Raft状态进行修改，就必须获得锁，这就导致锁的冲突。

解决办法就是向ApplyCh发送日志时不持有锁，servcie合适引用不关心，而此时InstallSnapshot也不应该涉及对状态修改，而是将快照数据也发送给ApplyCh，这样一来，当service收到普通日志就可以直接应用，而收到快照数据则执行阻塞执行CondInstallSnapshot，在函数中获取锁，保证执行日志和安装快照的一致性。

- Snapshot：上层触发，可以直接安装快照
- InstallSnapshot：有Leader调用，获取快照数据后发送到ApplyCh，具体何时执行有service决定
- CondInstallSnapshot：当ApplyCh收到快照后，触发这个函数，判断快照的截至Index是否大于commitIndex，从而判断是否需要执行安装快照

```go
type InstallSnapshotArgs struct {
	Term              int
	LeaderId          int
	LastIncludedIndex int
	LastIncludedTerm  int
	Data              []byte
}

type InstallSnapshotreply struct {
	Term int
}

func (rf *Raft) CondInstallSnapshot(lastIncludedTerm int, lastIncludedIndex int, snapshot []byte) bool {
	// Your code here (2D).
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if lastIncludedIndex <= rf.CommitIndex {
        //已经commit了Index的日志，不再安装快照
		return false
	}
	//1.截取日志
    //2.安装快照
    //3.持久化数据

	return true
}

func (rf *Raft) Snapshot(index int, snapshot []byte) {
	// Your code here (2D).
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if index > rf.LastSnapIndex {
		//1.截取日志
        //2.安装快照
        //3.持久化数据
	}
}

func (rf *Raft) InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotreply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if args.Term >= rf.CurrentTerm {
		//更新Term、State等数据

		if args.LastIncludedIndex > rf.LastSnapIndex {
			applyMsg := ApplyMsg{
				SnapshotValid: true,
				Snapshot:      args.Data,
				SnapshotTerm:  args.LastIncludedTerm,
				SnapshotIndex: args.LastIncludedIndex,
			}
            //将快照数据发送到ApplyCh
			go func() {
				rf.ApplyCh <- applyMsg
			}()
		}
	}
	reply.Term = rf.CurrentTerm
}

func (rf *Raft) sendInstallSnapshot(server int) {
	RPCTimer := time.NewTimer(RPCTimeout)
	defer RPCTimer.Stop()
	rf.mu.Lock()
	args := InstallSnapshotArgs{
		Term:              rf.CurrentTerm,
		LeaderId:          rf.me,
		LastIncludedIndex: rf.LastSnapIndex,
		LastIncludedTerm:  rf.LastSnapTerm,
		Data:              rf.SnapShotData,
	}
	rf.mu.Unlock()


	for rf.killed() == false {

		rf.mu.Lock()
		if rf.RaftState != LEADER {
			rf.mu.Unlock()
			return
		}
		rf.mu.Unlock()

		resCh := make(chan bool, 1)
		reply := InstallSnapshotreply{}

		go func(args *InstallSnapshotArgs, reply *InstallSnapshotreply) {
			ok := rf.peers[server].Call("Raft.InstallSnapshot", args, reply)
			
			if !ok {
				time.Sleep(time.Millisecond * 10)
			}
			resCh <- ok
		}(&args, &reply)

		select {
		case <-RPCTimer.C:
			rf.logger.Printf("appendtopeer, rpctimeout: peer:%d, args:%+v", server, args)
			break
		case ok := <-resCh:
			if !ok {
				rf.logger.Printf("appendtopeer not ok")
				continue
			}
		}

		//RPC success
		rf.mu.Lock()
		if replyTerm > rf.CurrentTerm {
			//更新State
            rf.mu.Unlock()
			return
		}

		//更新MatchIndex和NextIndex
		if args.LastIncludedIndex > rf.MatchIndex[server] {
			rf.MatchIndex[server] = args.LastIncludedIndex
		}
		if args.LastIncludedIndex+1 > rf.NextIndex[server] {
			rf.NextIndex[server] = args.LastIncludedIndex + 1
		}
		rf.mu.Unlock()
		return
	}
}

```



**日志的获取**：2D的另一个难点就是当截取快照后，Index不再和logs中的下标一致，因此需要对日志的获取进行重构：

```go
func (rf *Raft) getRealIthLog(index int) LogEntry {
	if index-rf.LastSnapIndex < 0 || index-rf.LastSnapIndex >= len(rf.Log) {
		rf.logger.Printf("Out of Range: index = %d, LastSnapIndex = %d", index, rf.LastSnapIndex)
        panic("Wrong index of Log")
	}
	return rf.Log[index-rf.LastSnapIndex]
}
```

## 其它

**关于加锁**：程序中很多地方都会涉及到多线程加锁的问题，在我最初实现中，每个Raft内部函数开始加锁、结束释放。但这造成一个问题，就是如果函数存在互相调用，就要先解锁，调用完函数在加锁，结果就是大量的加锁解锁操作，不仅维护困难而且造成额外开销。之后我对程序重构，涉及到内部调用的函数都不加锁，由调用者加锁，而调用者函数主要包括单独的goroutine和RPC，就是可以主动触发的那些函数来加锁。

**不能完全按照论文操作**：开始时我是完全按照论文的实现，但是发现很多问题，如nextIndex的更新等，因此还是要根据需要进行优化。
