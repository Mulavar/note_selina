raft算法的核心逻辑实现在raft.go中，api.go主要提供了封装给第三方开发者使用的函数。

在api.go里定义了最核心的入口函数`NewRaft(...)`，用于创建一个raft实体节点，raft节点封装了执行raft算法所需要的一些属性变量，并绑定一个了有限状态机(FSM)以执行请求。

```go
func NewRaft(conf *Config, fsm FSM, logs LogStore, stable StableStore, snaps SnapshotStore, trans Transport) (*Raft, error){
    // ...
}
```

该入口函数一共有6个参数，除了Config是已定义好的结构体，其余五个形参类型分别是五个接口，用户可以根据其实现定制化的类型，具体的参数意义在之后创建Raft节点中会讲解。



```go
func NewRaft(conf *Config, fsm FSM, logs LogStore, stable StableStore, snaps SnapshotStore, trans Transport) (*Raft, error){
    // 1. 检查配置是否有效
    if err := ValidateConfig(conf); err != nil {
		return nil, err
	}
    
    // 2. 创建Raft结构
    r := &Raft{
		// 用户提供的形参
		conf:                  *conf,
		fsm:                   fsm,
        logs:                  logs,
        stable:                stable,
        snapshots:             snaps,
		trans:                 trans,
        
        // 初始化一些配置信息，这些配置根据用户提供的conf或其他接口实现得到
        protocolVersion:       protocolVersion,
        localID:               localID,
		localAddr:             localAddr,
		logger:                logger,
        configurations:        configurations{},
        observers:             make(map[uint64]*Observer),
        
        // 初始化channel信息
        applyCh:               make(chan *logFuture),
		fsmMutateCh:           make(chan interface{}, 128),
		fsmSnapshotCh:         make(chan *reqSnapshotFuture),
		leaderCh:              make(chan bool, 1),
		configurationChangeCh: make(chan *configurationChangeFuture),
		rpcCh:                 trans.Consumer(),
		userSnapshotCh:        make(chan *userSnapshotFuture),
		userRestoreCh:         make(chan *userRestoreFuture),
		shutdownCh:            make(chan struct{}),
		verifyCh:              make(chan *verifyFuture, 64),
		configurationsCh:      make(chan *configurationsFuture, 8),
		bootstrapCh:           make(chan *bootstrapFuture),
		leadershipTransferCh:  make(chan *leadershipTransferFuture, 1),
	}
    
    // 3. 设置初始状态是FOLLOWER
    r.setState(Follower)
    
    // 4. 检索恢复当前term和上一条日志的term、index
    r.setCurrentTerm(currentTerm)
	r.setLastLog(lastLog.Index, lastLog.Term)
    
    // 5. 检索从上一次快照开始到最近日志范围内的所有配置变更日志，并处理
	snapshotIndex, _ := r.getLastSnapshot()
	for index := snapshotIndex + 1; index <= lastLog.Index; index++ {
		var entry Log
		if err := r.logs.GetLog(index, &entry); err != nil {
			r.logger.Error("failed to get log", "index", index, "error", err)
			panic(err)
		}
		r.processConfigurationLogEntry(&entry)
	}
    
    // 6. 装配一个专门用于处理心跳信息的函数
    trans.SetHeartbeatHandler(r.processHeartbeat)
    
    // 7. 启动主流程
    r.goFunc(r.run)
	r.goFunc(r.runFSM)
	r.goFunc(r.runSnapshots)
	return r, nil
}
```



#### 检查配置

配置检查包括但不限于以下内容，如：

- 检查通信协议版本是否符合要求；
- 服务器ID不能为空；
- 快照触发周期不能过于频繁；
- 选举超时控制必须比心跳超时控制长；
- 心跳超时控制不能过短；
- 选举超时控制不能过短；
- ……



#### 创建Raft节点

在创建raft节点过程中需要初始化两部分信息，用户提供的定制化配置，包括函数的6个形参变量以及服务器ID、服务器集群配置等信息，6个形参变量的意义分别如下：

- *Config(struct)：封装了Raft服务器所必须提供的配置信息，如服务器ID、心跳超时时间等；
- FSM(interface)：有限状态机接口，封装了三个方法：应用日志(Apply)、创建快照(Snapshot)、从快照恢复(Restore)。用户可以通过实现这三个方法创建自定义的FSM。
- LogStore(interface)：日志存储接口，提供了持久化的日志存储和检索能力。
- StableStore(interface)：关键信息存储接口。与LogStore不同的是，该接口提供的是存储一些关键的配置信息的功能，用于保障系统的安全性。
- SnapshotStore(interface)：快照存储接口。提供了持久化的快照存储和检索能力。
- Transport(interface)：传输层。定义了Raft节点之间的通讯方式。

另外，还需要初始化许多channel信息用于同步通信，如applyCh用于传递将要应用到FSM上的日志，fsmSnapshotCh用于处理快照请求。



#### 状态初始化

根据raft算法逻辑，每个raft节点在启动时都是FOLLOWER，当超时触发后会成为CANDIDATE并请求投票，获取大部分投票的节点将成为LEADER。因此在这里raft节点需要初始化自己的状态为FOLLOWER，并恢复当前节点的最新日志信息(term、index)。另外，raft节点需要记录最新的集群配置信息（根据raft算法描述，服务器总是使用最新的集群配置，不管该配置信息是否被提交）。



#### 启动主流程

在这一阶段，raft节点会分别使用三个协程分别启动三个主流程：

- r.run：执行raft算法流程
- r.runFSM：异步将日志应用到FSM上
- r.runSnapshots：负责处理制作快照，主要有定时器处理和响应用户请求两种方式