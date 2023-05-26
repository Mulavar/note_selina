hashicorp中对每一类状态的结构定义都非常清晰，这里讲解其中最为核心的一些数据结构。

### 节点

一些关于节点的结构定义在state.go中。

#### 节点状态（RaftState）

每个raft节点在不同时间点可能有不同的角色，或称状态，结构如下：

```go
type RaftState uint32

const (
   // 跟随者，每个Raft节点启动时的初始状态都是这个
   Follower RaftState = iota

   // 候选者，当跟随者的超时器被触发后会自动成为候选者请求获取投票
   Candidate

   // 领导者，负责同步日志
   Leader

   // 关闭状态
   Shutdown
)
```



节点除去状态外还会有各自的信息，比如当前term等，hashicorp定义了结构raftState用于表示raft算法中易变的信息。

#### 节点信息（raftState）

```go
type raftState struct {
   // currentTerm commitIndex, lastApplied,  must be kept at the top of
   // the struct so they're 64 bit aligned which is a requirement for
   // atomic ops on 32 bit platforms.

   // The current term, cache of StableStore
   currentTerm uint64

   // Highest committed log entry
   commitIndex uint64

   // Last applied log to the FSM
   lastApplied uint64

   // protects 4 next fields
   lastLock sync.Mutex

   // Cache the latest snapshot index/term
   lastSnapshotIndex uint64
   lastSnapshotTerm  uint64

   // Cache the latest log from LogStore
   lastLogIndex uint64
   lastLogTerm  uint64

   // Tracks running goroutines
   routinesGroup sync.WaitGroup

   // The current state
   state RaftState
}
```

这些变量表示的意义都非常明确，直接查看变量名也能理解其意思，需要注意的是，currentTerm、commitIndex、lastApplied都必须在该结构体的开头定义，这是为了保证这几个64位变量在32位系统上操作能保障原子性。go在字节对齐中规定**变量或开辟的结构体、数组和切片值中的第一个64位字可以被认为是8字节对齐**，即使在32位系统上操作也能保证这些对象的原子性，具体资料可查看[Go之聊聊struct的内存对齐](https://juejin.im/post/6844904067244769294)。后续的四个64位变量lastSnapshotIndex、lastSnapshotTerm、lastLogIndex、lastLogTerm采用了加锁(lastLock)的方式保障在32位平台上处理的原子性，这是因为应用场景不同。lastSnapshotIndex和lastSnapshotTerm都是表示最新快照对应的日志索引信息，需要同时获取，所以使用了加锁保障读取最新快照的操作原子性：

```go
func (r *raftState) getLastSnapshot() (index, term uint64) {
   r.lastLock.Lock()
   index = r.lastSnapshotIndex
   term = r.lastSnapshotTerm
   r.lastLock.Unlock()
   return
}
```

lastLogIndex、lastLogTerm也相似，具有同时读取和设置的应用场景。

还剩下一个成员routinesGroup，routinesGroup是为了在关闭主协程时能先等待其他后台goroutine关闭，主要包括：

- r.run：执行raft算法流程
- r.runFSM
- r.runSnapshots



### RPC

在raft算法中，主节点和从节点通过RPC通信以复制日志或同步快照，这些RPC消息的结构定义在commands.go中。

#### RPC头（RPCHeader）

```go
type RPCHeader struct {
   // ProtocolVersion is the version of the protocol the sender is
   // speaking.
   ProtocolVersion ProtocolVersion
}
```

hashicorp定义了一个RPC头，存放通信版本，并声明了一个接口WithRPCHeader，每个具体的RPC结构都需要实现该接口，目前hashicorp中所有的RPC结构对该接口的实现都是返回RPCHeader。

```go
type WithRPCHeader interface {
   GetRPCHeader() RPCHeader
}
```



#### 日志复制（AppendEntriesRequest）

```go
type AppendEntriesRequest struct {
	RPCHeader

	// 当前任期
	Term   uint64
    // 领导者的服务器地址
	Leader []byte

	// 当前要复制的日志项之前一条日志的索引和任期号
	PrevLogEntry uint64
	PrevLogTerm  uint64

	// 新增的日志项
	Entries []*Log

	// 领导者的提交日志的索引
	LeaderCommitIndex uint64
}
```



#### 请求投票（RequestVoteRequest）

```go
type RequestVoteRequest struct {
   RPCHeader

   // 当前任期
   Term      uint64
   // 跟随者的服务器地址
   Candidate []byte

   // 该跟随者最新日志的索引和任期号
   LastLogIndex uint64
   LastLogTerm  uint64

   // 标识当前选举是否是由领导者主动转让领导位置而触发
   LeadershipTransfer bool
}
```

需要注意的是LeadershipTransfer成员，因为一个跟随者成为候选者有两种情况：

1. 领导者崩溃，此时超时机制触发，集群中无领导者，因此跟随者们允许将票投给新的候选人；
2. 领导者主动触发让位领导，此时LeadershipTransfer为true，即使集群中领导者仍然处于正常状态，跟随者们也允许将票投给新的候选者。



#### 安装快照（InstallSnapshotRequest）

```go
type InstallSnapshotRequest struct {
   RPCHeader
   // 快照版本号
   SnapshotVersion SnapshotVersion

   // 快照任期
   Term   uint64
   // 领导者服务器地址
   Leader []byte

   // 快照中包含的最新日志索引和任期号
   LastLogIndex uint64
   LastLogTerm  uint64

   // 集群成员（为了兼容仍保留）
   Peers []byte

   // 集群成员
   Configuration []byte
   // 配置日志项的索引
   ConfigurationIndex uint64

   // 快照大小（以字节为单位）
   Size int64
}
```

Peers和Configuration都是表示集群成员，在最初的快照版本，即当SnapshotVersion为0时使用Peers，而在后续的快照版本都使用了Configuration。



#### 超时（TimeoutNowRequest）

```go
type TimeoutNowRequest struct {
   RPCHeader
}
```

无实体内容，只是为了提醒其他节点发起新一轮选举。



另外，对应这些RPC请求都会有相应的RPC响应，结构上也都实现了WithRPCHeader接口，和对应的RPC请求结构大同小异，可自行查看commands.go。



