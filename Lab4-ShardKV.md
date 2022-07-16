# Lab4A shardctrler

MIT 6.824 课程的lab4 是在lab2 Raft算法基础上实现一个multi-raft的KV集群服务，并且要实现raft组的动态增删，以及不同raft组拉取切片的过程。在我看来，lab4是4个实验中最难的一部分，lab2虽说繁琐一些，但至少有论文作为参考；而lab4没有参考，更多需要自己构思。我自己对这方面也没有过什么了解，因此实现大部分参考了一些开源的实现，我个人觉得非常好的就是[这篇博客](https://github.com/OneSizeFitsQuorum/MIT6.824-2021/)，我在理解实验过程中废了很多时间，所以我想多谈一下我对实验要求的基本理解，具体实现和高深的原理可以参考上面那篇博客。

lab4A 要实现的是配置的管理服务，和lab3类似，不过这里存储的是集群的配置。类似于HDFS的Master角色。它主要记录每个Raft组副本数；以及KV的shard被分配到了哪个raft组。下面是整体架构图：

![](.\images\Lab4-整体架构.png)

整个集群分为三个部分：

- ShardCtrler：存储集群的配置信息
- ShardKVServer：每个Group中有多个副本，存储KV数据，KV数据被切分成不同的切片，平均分配给不同的Group，一个Group负责同样的切片，并且Group内部副本数据一致；当集群配置改变时，Group之间互相传递切片数据。
- client：通过ShardCtrler获取最新配置，然后访问ShardKVServer对KV数据操作。

在ShardCtrler实现中最重要的是它存储的数据结构：

```go
type Config struct {
	Num    int              // config number
	Shards [NShards]int     // shard -> gid
	Groups map[int][]string // gid -> servers[]
}
```

举个例子说明：

现在数据库存储的Key为 a, b, c, d, e, f, g, h, i, j，对应于10个shard，NShards = 10；现有3个Group，3个groupId = 1，2，3

```go
Num    = 1
Shards = [1,1,1,2,2,2,3,3,3,3]
Groups = {
    1 : [1-1,1-2,1-3],
    2 : [2-1,2-2,2-3],
    3 : [3-1,3-2,3-3]
}
```

数据库Key的划分由一个散列函数决定

a, b, c被划分到了groupid = 1的集群中，集群有三个server副本：[1-1,1-2,1-3]

同理 d, e, f,被划分到了groupid = 2的集群中，集群有三个server副本：[2-1,2-2,2-3]

同理g, h, i, j被划分到了groupid = 2的集群中，集群有三个server副本：[3-1,3-2,3-3]



现在如果group3离开了，那shardctrler要对shard重新分配进行负载均衡，保证每个Group之间所承载的shard差距不会超过1，修改后的：

```go
Num    = 2
Shards = [1,1,1,2,2,2,1,1,2,2]
Groups = {
    1 : [1-1,1-2,1-3],
    2 : [2-1,2-2,2-3]
}
```

现在只有2个Group，a, b, c，g, h,被划分到了集群1中，其余被划分到了集群2中

具体代码实现以及集群迁移的思路可以参考文章开始提到大佬的博客，我这里就不再赘述。

# Lab4B shardKV

lab4B相较于lab3，都是实现了KV数据的存取，不同的是lab4B实现的是一个multi-raft的集群KV服务，当一个Group离开或者加入导致COnfig改变后，需要对KV数据进行Group之间的迁移，这是本实验的难点。为了实现功能，设计如下数据结构：

```go
type ShardKV struct {
	stateMachines map[int]*Shard // KV stateMachines
	...
}

type Shard struct {
	KV     map[string]string
	Status ShardStatus
}

//分区状态
type ShardStatus uint8

const (
	Serving ShardStatus = iota
	Pulling
	BePulling
	GCing
)
```

相比于lab3，这里对KV map进行了一层抽象，每个切片作为一组数据，并且维护切片的状态，每个ShardKV有多个切片。为了理解分区状态之间切换关系，现有如下示例：

1. <mark>**开始时集群中有G1、G2两个group，分别维护（s1、s2）和（s3、s4）切片。**</mark>

此时G1中切片状态：

| 切片ID |  状态   |
| :----: | :-----: |
|   s1   | Serving |
|   s2   | Serving |

G2切片状态：

| 切片ID |  状态   |
| :----: | :-----: |
|   s3   | Serving |
|   s4   | Serving |

2. <mark>**现集群配置改变：G1 负责（s1、s2、s4），G2负责s3**</mark>

G1 中切片的状态

| 切片ID |  状态   |
| :----: | :-----: |
|   s1   | Serving |
|   s2   | Serving |
|   s4   | Puling  |

G2切片状态：
| 切片ID |   状态    |
| :----: | :-------: |
|   s3   |  Serving  |
|   s4   | BePulling |

3. <mark>**之后G1开始拉取G2中切片s4，拉取完成后：**</mark>

G1 中切片的状态

| 切片ID |  状态   |
| :----: | :-----: |
|   s1   | Serving |
|   s2   | Serving |
|   s4   |  GCing  |

G2切片状态：

| 切片ID |   状态    |
| :----: | :-------: |
|   s3   |  Serving  |
|   s4   | BePulling |

4. <mark>**拉取完成后，G1会在自己副本之间同步数据，然后G1会通知G2删除切片s4，G1通知成功后会变成Serving，er在G2通知各个副本删除数据过程中s4的状态也会变成GCing。**</mark>



在看作者源码过程中，我觉的应该从主动触发和被动触发两个方面入手，主动触发就是实现的协程，被动触发就是供外部调用的RPC函数。在实现中，总共包括5个协程，理解这5个协程就可以很好理解作者思路：

1. **applier**

   这部分主要就是从和Raft进行通信的chan中获取可以应用的命令，根据不同命令执行操作即可，包括一条普通命令或者一条执行快照的命令：

   ```go
   switch command.Op {
   case Operation:
       //一条KV的操作
       kvOperation := command.Data.(CommandRequest)
       response = kv.applyOperation(&applyMsg, &kvOperation)
   case Configuration:
       //获取到了新的配置，更新本机congfig信息
       nextConfig := command.Data.(shardctrler.Config)
       response = kv.applyConfiguration(&nextConfig)
   case InsertShards:
       //拉取到了其它Group的shard，安装shard
       shardsInfo := command.Data.(ShardOperationResponse)
       response = kv.applyInsertShards(&shardsInfo)
   case DeleteShards:
       //某个shard已经不再归本group管理，删除shard
       shardInfo := command.Data.(ShardOperationRequest)
       response = kv.applyDeleteShards(&shardInfo)
   case EmptyEntry:
       //一条空日志
   	response = kv.applyEmptyEntry()
   }
   ```

2. **configureAction**

   不断尝试拉取最新配置，拉取成功后，调用Execute在Raft集群间同步

3. **migrationAction**

   获取Pulling的Shard，拉取数据，拉取成功后，调用Execute在Raft集群间同步

4. **gcAction**

   获取GCing的Shard，通知group删除，通知后，调用Execute在Raft集群间更改Shard状态

5. **checkEntryInCurrentTermAction**

   在一个Term中插入一条空语句，防止活锁

然后就是三个供RPC调用的函数：

```go
//ShardKV的客户端调用此函数，用于KV值的增删改查
func (kv *ShardKV) Command(request *CommandRequest, response *CommandResponse)
//Shard内部调用，用于拉取本机上的shard切片数据
func (kv *ShardKV) GetShardsData(request *ShardOperationRequest, response *ShardOperationResponse)
//Shard内部调用，用于通知本机shard GC
func (kv *ShardKV) DeleteShardsData(request *ShardOperationRequest, response *ShardOperationResponse)
```









