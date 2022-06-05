## `etcd raft`模块
相关模块详解：
> `Node`: 对`etcd raft`模块的具体实现的封装；
> `上层模块`：`etcd raft`的调用者，上层模块通过`Node`提供的`API`与底层的`etcd raft`模块交互。
> `Cluster`: 表示一个集群，记录了集群的基础信息；
> `Member`: 组成`CLuster`的元素之一，封装了一个节点的基本信息；
> `Peer`: 集群中某个节点对另一个集群另一个节点的称呼。

### `Entry`结构
```protobuf 3
message Entry {
	optional uint64     Term  = 2 [(gogoproto.nullable) = false]; // must be 64-bit aligned for atomic operations
	optional uint64     Index = 3 [(gogoproto.nullable) = false]; // must be 64-bit aligned for atomic operations
	optional EntryType  Type  = 1 [(gogoproto.nullable) = false];
	optional bytes      Data  = 4;
}
```
```go
type Entry struct {
	Term  uint64    `protobuf:"varint,2,opt,name=Term" json:"Term"` // Entry 任期Term
	Index uint64    `protobuf:"varint,3,opt,name=Index" json:"Index"` // Entry 索引
	Type  EntryType `protobuf:"varint,1,opt,name=Type,enum=raftpb.EntryType" json:"Type"` // Entry Type EntryNormal-正常数据，EntryConfigChange--集群变化信息
	Data  []byte    `protobuf:"bytes,4,opt,name=Data" json:"Data,omitempty"` // 操作数据
}
```
### `Message`结构
```protobuf 3
message Message {
	optional MessageType type        = 1  [(gogoproto.nullable) = false];
	optional uint64      to          = 2  [(gogoproto.nullable) = false];
	optional uint64      from        = 3  [(gogoproto.nullable) = false];
	optional uint64      term        = 4  [(gogoproto.nullable) = false];
	// logTerm is generally used for appending Raft logs to followers. For example,
	// (type=MsgApp,index=100,logTerm=5) means leader appends entries starting at
	// index=101, and the term of entry at index 100 is 5.
	// (type=MsgAppResp,reject=true,index=100,logTerm=5) means follower rejects some
	// entries from its leader as it already has an entry with term 5 at index 100.
	optional uint64      logTerm     = 5  [(gogoproto.nullable) = false];
	optional uint64      index       = 6  [(gogoproto.nullable) = false];
	repeated Entry       entries     = 7  [(gogoproto.nullable) = false];
	optional uint64      commit      = 8  [(gogoproto.nullable) = false];
	optional Snapshot    snapshot    = 9  [(gogoproto.nullable) = false];
	optional bool        reject      = 10 [(gogoproto.nullable) = false];
	optional uint64      rejectHint  = 11 [(gogoproto.nullable) = false];
	optional bytes       context     = 12;
}
```
### `Raft`结构
```go
type raft struct {
	id uint64

	Term uint64
	Vote uint64

	readStates []ReadState

	// the log
	raftLog *raftLog

	maxMsgSize         uint64
	maxUncommittedSize uint64
	// TODO(tbg): rename to trk.
	prs tracker.ProgressTracker

	state StateType

	// isLearner is true if the local raft node is a learner.
	isLearner bool

	msgs []pb.Message

	// the leader id
	lead uint64
	// leadTransferee is id of the leader transfer target when its value is not zero.
	// Follow the procedure defined in raft thesis 3.10.
	leadTransferee uint64
	// Only one conf change may be pending (in the log, but not yet
	// applied) at a time. This is enforced via pendingConfIndex, which
	// is set to a value >= the log index of the latest pending
	// configuration change (if any). Config changes are only allowed to
	// be proposed if the leader's applied index is greater than this
	// value.
	pendingConfIndex uint64
	// an estimate of the size of the uncommitted tail of the Raft log. Used to
	// prevent unbounded log growth. Only maintained by the leader. Reset on
	// term changes.
	uncommittedSize uint64

	readOnly *readOnly

	// number of ticks since it reached last electionTimeout when it is leader
	// or candidate.
	// number of ticks since it reached last electionTimeout or received a
	// valid message from current leader when it is a follower.
	electionElapsed int

	// number of ticks since it reached last heartbeatTimeout.
	// only leader keeps heartbeatElapsed.
	heartbeatElapsed int

	checkQuorum bool
	preVote     bool

	heartbeatTimeout int
	electionTimeout  int
	// randomizedElectionTimeout is a random number between
	// [electiontimeout, 2 * electiontimeout - 1]. It gets reset
	// when raft changes its state to follower or candidate.
	randomizedElectionTimeout int
	disableProposalForwarding bool

	tick func()
	step stepFunc

	logger Logger

	// pendingReadIndexMessages is used to store messages of type MsgReadIndex
	// that can't be answered as new leader didn't committed any log in
	// current term. Those will be handled as fast as first log is committed in
	// current term.
	pendingReadIndexMessages []pb.Message
}
```
### `config`结构
```go
type Config struct {
	// ID is the identity of the local raft. ID cannot be 0.
	ID uint64

	// ElectionTick is the number of Node.Tick invocations that must pass between
	// elections. That is, if a follower does not receive any message from the
	// leader of current term before ElectionTick has elapsed, it will become
	// candidate and start an election. ElectionTick must be greater than
	// HeartbeatTick. We suggest ElectionTick = 10 * HeartbeatTick to avoid
	// unnecessary leader switching.
	ElectionTick int
	// HeartbeatTick is the number of Node.Tick invocations that must pass between
	// heartbeats. That is, a leader sends heartbeat messages to maintain its
	// leadership every HeartbeatTick ticks.
	HeartbeatTick int

	// Storage is the storage for raft. raft generates entries and states to be
	// stored in storage. raft reads the persisted entries and states out of
	// Storage when it needs. raft reads out the previous state and configuration
	// out of storage when restarting.
	Storage Storage
	// Applied is the last applied index. It should only be set when restarting
	// raft. raft will not return entries to the application smaller or equal to
	// Applied. If Applied is unset when restarting, raft might return previous
	// applied entries. This is a very application dependent configuration.
	Applied uint64

	// MaxSizePerMsg limits the max byte size of each append message. Smaller
	// value lowers the raft recovery cost(initial probing and message lost
	// during normal operation). On the other side, it might affect the
	// throughput during normal replication. Note: math.MaxUint64 for unlimited,
	// 0 for at most one entry per message.
	MaxSizePerMsg uint64
	// MaxCommittedSizePerReady limits the size of the committed entries which
	// can be applied.
	MaxCommittedSizePerReady uint64
	// MaxUncommittedEntriesSize limits the aggregate byte size of the
	// uncommitted entries that may be appended to a leader's log. Once this
	// limit is exceeded, proposals will begin to return ErrProposalDropped
	// errors. Note: 0 for no limit.
	MaxUncommittedEntriesSize uint64
	// MaxInflightMsgs limits the max number of in-flight append messages during
	// optimistic replication phase. The application transportation layer usually
	// has its own sending buffer over TCP/UDP. Setting MaxInflightMsgs to avoid
	// overflowing that sending buffer. TODO (xiangli): feedback to application to
	// limit the proposal rate?
	MaxInflightMsgs int

	// CheckQuorum specifies if the leader should check quorum activity. Leader
	// steps down when quorum is not active for an electionTimeout.
	CheckQuorum bool

	// PreVote enables the Pre-Vote algorithm described in raft thesis section
	// 9.6. This prevents disruption when a node that has been partitioned away
	// rejoins the cluster.
	PreVote bool

	// ReadOnlyOption specifies how the read only request is processed.
	//
	// ReadOnlySafe guarantees the linearizability of the read only request by
	// communicating with the quorum. It is the default and suggested option.
	//
	// ReadOnlyLeaseBased ensures linearizability of the read only request by
	// relying on the leader lease. It can be affected by clock drift.
	// If the clock drift is unbounded, leader might keep the lease longer than it
	// should (clock can move backward/pause without any bound). ReadIndex is not safe
	// in that case.
	// CheckQuorum MUST be enabled if ReadOnlyOption is ReadOnlyLeaseBased.
	ReadOnlyOption ReadOnlyOption

	// Logger is the logger used for raft log. For multinode which can host
	// multiple raft group, each raft group can have its own logger
	Logger Logger

	// DisableProposalForwarding set to true means that followers will drop
	// proposals, rather than forwarding them to the leader. One use case for
	// this feature would be in a situation where the Raft leader is used to
	// compute the data of a proposal, for example, adding a timestamp from a
	// hybrid logical clock to data in a monotonically increasing way. Forwarding
	// should be disabled to prevent a follower with an inaccurate hybrid
	// logical clock from assigning the timestamp and then forwarding the data
	// to the leader.
	DisableProposalForwarding bool
}
```



