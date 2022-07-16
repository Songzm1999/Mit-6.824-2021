# Lab3

Lab3需要在Lab2的Raft基础上，实现一个键值数据库，客户端提供的操作包括Get、Put、Append，并且要实现在客户端的去重操作。我觉得这张图可以很好讲解各个模块的执行流程：

![](.\images\Lab3-架构.png)

## 3A

在Part-A部分需要完成KVRaft的大部分工作,只是不需要考虑Snapshot,这部分在Part-B中完成

### Client

在Client这部分的数据比较简单,唯一需要注意的是为了保证在Server端每个命令的去重操作,需要在Client发送命令时,给每个命令赋值一个ID:

```go
type Clerk struct {
	servers   []*labrpc.ClientEnd
	clientId  int64			//客户端ID
	leaderId  int			//保存Leader信息,减少访问次数
	requestId int64			//每个操作赋值一个ID
	mu        sync.Mutex

	logFile *os.File
	logger  *log.Logger
}
```

在Client中包括Get、Put、Append操作,Client维护一个自增的requestId保证每个操作的唯一性,在操作开始时将字段自增一:`atomic.AddInt64(&ck.requestId, 1)`

Get、Put、Append操作需要注意当RPC失败、超时、Leader错误时要重新发送RPC，直到发送成功：

![](.\images\Lab3-Client.jpg)

以Get操作为例，PutAppend同理：

```go
func (ck *Clerk) Get(key string) string {
	atomic.AddInt64(&ck.requestId, 1)
	args := GetArgs{
		Key:   key,
		CliId: ck.clientId,
		ReqId: atomic.LoadInt64(&ck.requestId),
	}

	for {
		reply := GetReply{}
		ck.log("Clerk Get Call leader = %v, args = %v", ck.leaderId, args)
		ok := ck.servers[ck.leaderId].Call("KVServer.Get", &args, &reply)
		if !ok {
			ck.leaderId = (ck.leaderId + 1) % len(ck.servers)
			continue
		}
		ck.log("Clerk Get Call leader = %v, args = %v ,reply = %v", ck.leaderId, args, reply)
		//RPC return success
		switch reply.Err {
		case OK:
			return reply.Value
		case ErrNoKey:
			return ""
		case ErrWrongLeader:
			ck.leaderId = (ck.leaderId + 1) % len(ck.servers)
			continue
		case ErrTimeOut:
			continue
		default:
			ck.log("RPC Get return wrong param reply = %v", reply)
			panic("RPC Get return wrong param reply = %v")
		}
	}
}
```

### Server

#### 如何去重

在指导手册中提到了要对Client的每个请求去重，每次操作在Server中只能执行一次，客户端默认是线性执行每次操作，只有在本次操作完成之后才会进行下一步，因此可以在Server中维护一个数据结构保存上一次的结果：

```go
type KVServer struct {
	lastOptions    map[int64]int64           // key = clientId, value = RequestId 检查请求是否重复
	notifyResults  map[int64]*NotifyMsg      // key = clientId, value = NotifyMsg 保存上次请求结果
	notifyChans    map[int64]chan *NotifyMsg // key = MsgId, 用于applier和watiCmd之间同步
}
```

- lastOptions的key是ClientID，值是请求的RequestID，用于检查操作是否重复
- notifyResults的key是ClientID，值是上次请求的结果，如果请求重复，直接从该map中取值即可
- notifyChans的key是每次RPC的ID，值是操作结果，用于多线程返回结果

在检查去重时，不能当一个RPC到达时就直接对比lastOptions，依然需要将该操作提交到Raft，当获得到majority的认可后再检查去重，一次请求可以被 commit 多次，但一定只能 apply 一次。**因为存在以下这种情况**：

1. 集群中有五个节点s1、s2、s3、s4、s5，leader = s1，存在数据 X = 1，Client1向s1发出请求Get(X)，Sercer记录本次操作，然后返回，但是该数据丢包了
2. 返回RPC后，集群中间断开了，分为了s1、s2和s3、s4、s5，s3、s4、s5重新选举s3成为leader，然后client2向s3发送Put(X , 2)，成功返回，此时X = 2为最新数据
3. C1发现请求超时，再一次向s1发送请求，如果在请求开始时就去重，返回X= 1，不能保证强一致性，最好方法就是将请求提交到Raft中，只有获得majority的认可后再返回

第二个需要注意的点就是将操作提交到raft后，监听本次RPC的chan，Server中applier提交了某次请求后，会将结果放入对应的chan中。

```go
//每次请求都会封装成一个Op提交到Raft
type Op struct {
	Optype string		//Get Put Append
	Key    string
	Value  string
	ReqId  int64
	CliId  int64
	MsgId  int64
}

type NotifyMsg struct {
	Err   Err
	Value string
}

//提供RPC
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	op := Op{
		Optype: "Get",
		Key:    args.Key,
		ReqId:  args.ReqId,
		CliId:  args.CliId,
		MsgId:  nrand(),		//每次RPC都会生成一个随机的ID，用于后续建立chan通知
         //去重操作是根据ClientId 和 RequestId共同检查维护的，而MsgId标识了不同的RPC请求，用于applier和waitCmd之间同步
	}
	notifyMsg := kv.waitCmd(op)
	reply.Err, reply.Value = notifyMsg.Err, notifyMsg.Value
	kv.log("Get func : args = %v, reply = %v\n", args, reply)
}

//提交到Raft后监听chan
func (kv *KVServer) waitCmd(op Op) *NotifyMsg {
	if _, _, isLeader := kv.rf.Start(op); !isLeader {
		//not leader
		return &NotifyMsg{Err: ErrWrongLeader}
	}

	notifyChan := make(chan *NotifyMsg, 1)
	kv.mu.Lock()
	kv.notifyChans[op.MsgId] = notifyChan
	kv.mu.Unlock()

	timer := time.NewTimer(ProccessTimeOut)
	defer func(msgId int64) {
		timer.Stop()
		kv.mu.Lock()
		delete(kv.notifyChans, msgId)
		kv.mu.Unlock()
	}(op.MsgId)

	select {
	case <-timer.C:
        //超时
		return &NotifyMsg{Err: ErrTimeOut}
	case notifyMsg := <-notifyChan:
		//执行成功
		return notifyMsg
	}
}

//applier作为单独go协程来监听Raft中commit的命令，执行成功后返回到该命令对应的chan中，来通知waitCmd结束阻塞并返回客户端
func (kv *KVServer) applier() {
	for kv.killed() == false {
		if applyMsg, ok := <-kv.applyCh; ok {
			kv.mu.Lock()
			if applyMsg.CommandValid && kv.lastApplied < applyMsg.CommandIndex {
                 //本命令是一个Command
				kv.lastApplied = applyMsg.CommandIndex
				op := applyMsg.Command.(Op)
                 //检查是否重复，此时以及获得了Raft的认可
				isRepeated := kv.isRepeatedReq(op.ReqId, op.CliId)

				if !isRepeated {
					notifyMsg := NotifyMsg{}
					switch op.Optype {
					case "Put":
						notifyMsg.Err = kv.kvStateMachine.Put(op.Key, op.Value)
					case "Append":
						notifyMsg.Err = kv.kvStateMachine.Append(op.Key, op.Value)
					case "Get":
						notifyMsg.Value, notifyMsg.Err = kv.kvStateMachine.Get(op.Key)
					default:
						kv.log("Wrong param in opTpye, op = %v", op)
						panic("Wrong param")
					}
					kv.log("Server applied op = %v\n", op)
					kv.notifyResults[op.CliId] = &notifyMsg
					kv.lastOptions[op.CliId] = op.ReqId
				}
                 //返回到对应的chan，只有在Leader中才会成功获取到chan，而Follower只需要简单将命令执行即可，不用通知对应的waitCmd
				if ch, ok := kv.notifyChans[op.MsgId]; ok {
					ch <- kv.notifyResults[op.CliId]
				}
			} else if applyMsg.SnapshotValid {
				//本次命令是一个Snapshot
                 ...
			}
			kv.mu.Unlock()
		}
	}
}

func (kv *KVServer) isRepeatedReq(reqId, clientId int64) bool {
	if preReqId, ok := kv.lastOptions[clientId]; ok {
		return preReqId == reqId
	}
	return false
}
```

在Server中保存数据时，参考了[清华学长](https://github.com/LebronAl/MIT6.824-2021/blob/master/docs/lab3.md)的实现，将KV数据保存在一个状态机中：

```go
package kvraft

type KVStateMachine interface {
	Get(key string) (string, Err)
	Put(key string, value string) Err
	Append(key string, value string) Err
}

type MemoryKV struct {
	KV map[string]string
}

func NewMemoryKV() *MemoryKV {
	return &MemoryKV{KV: make(map[string]string)}
}

func (memoryKV *MemoryKV) Get(key string) (string, Err) {
	if value, ok := memoryKV.KV[key]; ok {
		return value, OK
	}
	return "", ErrNoKey
}

func (memoryKV *MemoryKV) Put(key string, value string) Err {
	memoryKV.KV[key] = value
	return OK
}

func (memoryKV *MemoryKV) Append(key string, value string) Err {
	memoryKV.KV[key] += value
	return OK
}
```



## 3B

partB 要实现snapshot功能，这部分如果阅读过Raft中Lab2D的测试代码感觉会简单不少。3B主要是对Server的修改

因为手册中提到要如果系统崩溃，也要能检查去重，因此涉及去重的数据都需要持久化：

```go
func (kv *KVServer) getSnapShotData() []byte {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(kv.kvStateMachine)
	e.Encode(kv.lastOptions)
	e.Encode(kv.notifyResults)
	return w.Bytes()
}

func (kv *KVServer) readSnapShot(snapshot []byte) {
	r := bytes.NewBuffer(snapshot)
	d := labgob.NewDecoder(r)
	var stateMachine MemoryKV
	var lastOptions map[int64]int64
	var notifyResults map[int64]*NotifyMsg

	if d.Decode(&stateMachine) != nil ||
		d.Decode(&lastOptions) != nil ||
		d.Decode(&notifyResults) != nil {
		kv.log("KVServer readerSnapshot got an error....\n")
	} else {
		kv.kvStateMachine = &stateMachine
		kv.lastOptions = lastOptions
		kv.notifyResults = notifyResults
	}
}
```

至于在什么时候执行Snapshot，需要执行Command后判断日志长度是否超过阈值，超过的话执行一次Snapshot

执行过程中检查了CondInstallSnapshot，是为了普通Command和Snapshot之间的同步，可以参考之前文章。

```go
func (kv *KVServer) applier() {
	for kv.killed() == false {
		if applyMsg, ok := <-kv.applyCh; ok {
			kv.mu.Lock()
			if applyMsg.CommandValid && kv.lastApplied < applyMsg.CommandIndex {
                  //普通命令
				...
				//check if need snapshot
				if kv.maxraftstate != -1 && kv.persister.RaftStateSize() >= kv.maxraftstate {
					snapshot := kv.getSnapShotData()
					kv.rf.Snapshot(applyMsg.CommandIndex, snapshot)
					kv.log("KVServer take an snapshot index = %v\n", applyMsg.CommandIndex)
				}
			} else if applyMsg.SnapshotValid {
                  //一次Snapshot的命令
				if kv.rf.CondInstallSnapshot(applyMsg.SnapshotTerm, applyMsg.SnapshotIndex, applyMsg.Snapshot) {
					kv.readSnapShot(applyMsg.Snapshot)
					kv.lastApplied = applyMsg.SnapshotIndex
					kv.log("KVServer CondInstallSnapshot an snapshot index = %v\n", applyMsg.CommandIndex)
				}
			}
			kv.mu.Unlock()
		}
	}
}
```

## 其它

Lab3本身实现没有特别难的点，但是会涉及到对Raft的实现检查，尤其是2C和2D，在我调试过程中始终不能通过TestSpeed3A的测试，3A的第二个测试TestSpeed3A进行了1000次KV操作，要求平均耗时在33ms以内，但在我程序中从来都没成功过，大概耗时在40/50ms。查看日志后，发现大部分耗时都在Raft persist时持有锁的时候，到后期，一次persist就要几十ms，log增长势必会造成persist时间增长。

在我调试了很长时间后发现我在persist时对applyIndex和CommitIndex也执行了持久化，这就造成了提交一次日志，或者CommitIndex更新都需要执行persist，不可避免造成不必要的时间浪费，在删除对二者的持久化后，执行时间降了不少，依然不能通过

然后我将persist变成单独go执行，不再阻塞执行，顺利通过了测试TestSpeed3A。

但这引入了另一个问题：在3A后面的测试模拟了Raft和KVServer宕机的情况，如果go persist()就可能存在这种情况：

1. Raft Leader s1将Entries发送到各个Follower s2、s3，Follower追加到Log中，然后在新线程中执行persist，返回Leader执行成功；
2. Leader得到majority的认可后，commit该日志，KVServer得到commit后，返回给客户端，表示执行成功；
3. 然而此时，虽然s1 成功执行了persist，但s2、s3还没完成就全部宕机了；全部启动后，s2、s3还没有保存到上一条日志，此时s2成为了leader，客户端访问时得到了不一致的结果。

总之，在我的实现中，无法100次测试全部通过，为了可靠性，就不得不阻塞执行persist；为了速度，就得牺牲可靠性，鱼和熊掌不可兼得。



