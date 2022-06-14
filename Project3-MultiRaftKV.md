# Project3 MultiRaftKV

这一节中，最难的就是 Project 3B，引无数英雄竞折腰！！当然撑过 3B，你就解放了。

## Membership Change

在 Project3A 中我们需要实现 Leader Transfer 和新增或移除一个 Region 里面的节点，这里并不是很难但是你要处理好细节，不然就是 BUG 满天飞。

### Leader Transfer

Leader transfer 会作为一条 Admin 指令被 propose，你直接调用`d.RaftGroup.TransferLeader()`方法即可。Leader transfer 不需要被整个 Raft group 确认，是单边发起的。步骤如下：

1. 收到 `AdminCmdType_TransferLeader` 请求，调用 `d.RaftGroup.TransferLeader()`。
2. 通过 `Step()` 函数输入 Raft。
3. 先判断自己是不是 Leader，因为只有 Leader 才有权利转移，否则直接忽略。
4. 判断自己的 `leadTransferee` 是否为空，如果不为空，则说明已经有 Leader Transfer 在执行，忽略本次。这里我是参考了 Etcd 的设计，如果正在执行的 `leadTransferee` 和你要转移目标的不同，则终止之前的转移，将 `leadTransferee` 设置为本次转移目标。说起来比较拗口，可以参考 Etcd。
5. 如果目标节点拥有和自己一样新的日志，则发送 `pb.MessageType_MsgTimeoutNow` 到目标节点。否则启动 append 流程同步日志。当同步完成后再发送 `pb.MessageType_MsgTimeoutNow`。
6. 当 Leader 的 `leadTransferee` 不为空时，不接受任何 propose，因为正在转移。
7. 如果在一个 `electionTimeout` 时间内都没有转移成功，则放弃本次转移，重置 `leadTransferee`。因为目标节点可能已经挂了。
8. 目标节点收到 `pb.MessageType_MsgTimeoutNow` 时，应该立刻重置自己的定时器并自增 term 开始选举。

整个 Leader Transfer 不复杂，就是当目标节点拥有和原来 Leader 一样新的日志时，其迅速发起选举，因为其拥有最新的日志，所以选举肯定能够成功，故其可以成为新的 Leader。

### Add/Remove Node

具体的流程文档里写的差不多，和其他 propose 不一样的是它的 propose 是通过 `d.RaftGroup.ProposeConfChange()` 方式提交的，下面我主要说明下在 apply 时需要做些什么。

1. 读取原有的 region，调用 `d.Region()` 即可
2. 修改 `region.Peers`，是删除就删除，是增加就增加一个 peer。如果删除的目标节点正好是自己本身，那么直接调用 `d.destroyPeer()` 方法销毁自己，并直接 return。后面的操作你都不用管了。
3. 设置 `region.RegionEpoch.ConfVer++`，这个可以看 PingCAP 里面的 blog，那里有说明。
4. 持久化修改后的 Region，写到 kvDB 里面。使用 `meta.WriteRegionState()` 方法。注意使用的是 `rspb.PeerState_Normal`，因为其要正常服务请求的。
5. 调用 `d.insertPeerCache()` 或 `d.removePeerCache()` 方法，这决定了你的消息是否能够正常发送，`peer.go` 里面的 `peerCache`注释上说明了为什么这么做。
6. 调用 `d.RaftGroup.ApplyConfChange()` 方法，因为刚刚修改的是 RawNode 上层的 peers 信息，Raft 内部的 peers 还没有修改。
7. 调用 Raft 的 `addNode()` 或 `removeNode()` 方法，方法里面只要修改下 `Prs` 就行了。同时如果自己是 Leader，尝试 commit，因为移除节点可能能够推进 commit。
8. 调用 `notifyHeartbeatScheduler()` 方法，这是啥？

先给大家看一眼我的 `notifyHeartbeatScheduler()` 方法：

```Go
func (d *peerMsgHandler) notifyHeartbeatScheduler(region *metapb.Region, peer *peer) {
  clonedRegion := new(metapb.Region)
  err := util.CloneMsg(region, clonedRegion)
  if err != nil {
    return
  }
  d.ctx.schedulerTaskSender <- &runner.SchedulerRegionHeartbeatTask{
    Region:          clonedRegion,
    Peer:            peer.Meta,
    PendingPeers:    peer.CollectPendingPeers(),
    ApproximateSize: peer.ApproximateSize,
  }
}
```

可以很明显看到这个方法和 `HeartbeatScheduler()` 很像，为什么这么做？是为了快速刷新 scheduler 那里的 region 缓存，能有效的解决你在测试用例里面遇到的 no region 问题。这个方法等会在 split 那里也会派上用场。

### 疑难杂症

**为什么 add node 时并没有新建 peer 的操作？**
那是因为 peer 不是你来创建的。在 `store_worker.go` 的 `onRaftMessage()` 方法中可以看到，当目标的 peer 不存在时，它会调用 `d.maybeCreatePeer()` 尝试创建 peer。新的 peer 的 startKey，endKey 均为空，因为他们等着 snapshot 的到来，以此来更新自己。这也是为什么在 `ApplySnapshot()` 方法中你需要调用 `ps.isInitialized()`，Project2 的笔记有提到。

**删除节点时会遇到 Request timeout 问题。**

首先这种情况发生在测试用例是设置了网络是 unreliable 的，且存在 remove node 到最后两个节点，然后被 remove 的那个正好是 Leader。

考虑如下情况：

只剩两个节点，然后被移除的那个节点正好是 Leader。因为网络是 unreliable，Leader 广播给另一个 Node 的心跳正好被丢了，也就是另一个节点的 commit 并不会被推进，也就是对方节点并不会执行 remove node 操作。而这一切 Leader 并不知道，它自己调用 `d.destroyPeer()` 已经销毁了。此时另一个节点并没有移除 Leader，它会发起选举，但是永远赢不了，因为需要收到被移除 Leader 的投票。

具体也可以看这个帖子：[https://asktug.com/t/topic/274196](https://asktug.com/t/topic/274196)

解决办法很简单，在 propose 阶段，如果已经处于两节点，被移除的正好是 Leader，那么直接拒绝该 propose，并且发起 Transfer Leader 到另一个节点上即可。Client 到时候会重试 remove node 指令。

当然我还在 apply 阶段 DestroyPeer 上面做了一个保险（有必要，虽然用到的概率很低），我让 Leader 在自己被 remove 前重复多次发送心跳到目标节点，尝试推动目标节点的 commit。重复多次是为了抵消测试用例的 unreliable。代码如下：

```Go
func (d *peerMsgHandler) startToDestroyPeer() {
  if len(d.Region().Peers) == 2 && d.IsLeader() {
    var targetPeer uint64 = 0
    for _, peer := range d.Region().Peers {
      if peer.Id != d.PeerId() {
        targetPeer = peer.Id
        break
      }
    }
    if targetPeer == 0 {
      panic("This should not happen")
    }

    m := []pb.Message{{
      To:      targetPeer,
      MsgType: pb.MessageType_MsgHeartbeat,
      Commit:  d.peerStorage.raftState.HardState.Commit,
    }}
    for i := 0; i < 10; i++ {
      d.Send(d.ctx.trans, m)
      time.Sleep(100 * time.Millisecond)
    }
  }
  d.destroyPeer()
}
```

## Split

Split 的分裂逻辑，假设在一个 store 上原本只有一个 regionA（存有 1~100 的数据），当 regionA 容量超出 split 阀值时触发 split 操作。首先我们需要找到那个 split key（其实是 50）。方法是调用 badger 的 API，按照**字典顺序**遍历这个 store 上的 kv，找到一分为二的 key。

> 虽然 badger 遍历的时候是按照字典顺序遍历的，但并不意味着 kv 在 badger 里面是按顺序存储的。

之后直接分裂成 regionA（存有 0~49） 和 regionB（存有 50~100）。注意 regionA 和 regionB 还是公用着同一个 store，也就是公用同一个 badger。也就是并不存在数据迁移的过程，你不需要把 regionA 里面的数据搬到 regionB 里去。

**那这么分裂有什么意义，反正都是在一个 store 上？**

1. 数据的粒度不一样，更细的粒度，可以实现更加精细的管理，比如当 regionA 访问压力过大时，可以单独增加 regionA 的 peer 的数量，分摊压力。

2. 比如原本 0~100 的范围里面你只能使用一个 Raft Group 处理请求，然后你把它一分为二为两个 region，可以用两个 Raft Group，能提升访问性能。

### 触发 Split 流程

1. `peer_msg_handler.go` 中的 `onTick()` 定时检查，调用 `onSplitRegionCheckTick()` 方法，它会生成一个 `SplitCheckTask` 任务发送到 `split_checker.go` 中。
2. 检查时如果发现满足 split 要求，则生成一个 `MsgTypeSplitRegion` 请求。
3. 在 `peer_msg_handler.go` 中的 `HandleMsg()` 方法中调用 `onPrepareSplitRegion()`，发送 `SchedulerAskSplitTask` 请求到 `scheduler_task.go` 中，申请其分配新的 region id 和 peer id。申请成功后其会发起一个 `AdminCmdType_Split` 的 AdminRequest 到 region 中。
4. 之后就和接收普通 AdminRequest 一样，propose 等待 apply。注意 propose 的时候检查 splitKey 是否在目标 region 中和 regionEpoch 是否为最新，因为目标 region 可能已经产生了分裂。

### Apply split 流程

庆幸不用实现 merge 操作，不然又是一场噩梦。前面 propose 的流程和 Add/Remove Node 差不多，这里主要说一下 apply 流程。

1. 基于原来的 region clone 一个新 region，这里把原来的 region 叫 leftRegion，新 region 叫 rightRegion。Clone region 可以通过 `util.CloneMsg()` 方法。
2. leftRegion 和 rightRegion 的 `RegionEpoch.Version++`。
3. 修改 rightRegion 的 Id，StartKey，EndKey 和 Peers。
4. 修改 leftRegion 的 EndKey。
5. 持久化 leftRegion 和 rightRegion 的信息。
6. 通过 `createPeer()` 方法创建新的 peer 并注册进 router，同时发送 `message.MsgTypeStart` 启动 peer。
7. 更新 storeMeta 里面的 regionRanges，同时使用 `storeMeta.setRegion()` 进行设置。注意加锁。
8. 调用 `d.notifyHeartbeatScheduler()`，原因上面有说。这里我 notify 的时候替新 region 也就是 rightRegion 也 notify 了。因为存在新 region 的 Leader 还没选出来，测试用例已经超时的问题，通常报的错是 no region 问题。

```Go
// notify new region created
d.notifyHeartbeatScheduler(leftRegion, d.peer)
d.notifyHeartbeatScheduler(rightRegion, newPeer)
```

另一种解决方法可以见，不过我感觉有点像是直接改了测试用例：[https://asktug.com/t/topic/274159](https://asktug.com/t/topic/274159)。

在创建 peer 的时候你可能会遇到 split request 中的 `NewPeerIds` 和你当前 region peers 数量不一致的问题，我这里是在 apply 之前做了一个判断，如果两者长度不相同，直接拒绝本次 split request。产生的原因我还没有想清楚，希望有人遇到了解答下。大概的猜想是 PD 收到了被 partition 的 Leader 的信息，所以 PD 发起的 split 中的 peers 数量是 outdated 的，但是奇怪的是它传入 split request 的 RegionEpoch 是正确的，如果真的是被 partition 了，其 RegionEpoch 应该也是 outdated 的。反正百思不得其解。

## 注意事项

因为 Membership Change 和 split 修改了 regionEpoch 和 regionRanges，所以你在 apply 每一个 entry 时，你都要判断请求的 regionEpoch 是否正确且目标的 key 是不是在该 region 里面，如果不在，应该返回 `ErrRespStaleCommand()` 错误。

在 `nclient >= 8 && crash = true && split = true` 这种条件下，测试在 Delete 阶段卡死问题，这是因为在 apply `CmdType_Put` 和 `CmdType_Delete` 请求的时候没有更新 `SizeDiffHint`。因此需要在 `Put` 的时候，`SizeDiffHint` 加上 `key` 和 `value` 的大小；在 `Delete` 的时候，减去 `key` 的大小。

## Scheduler

这一节实现上层的调度器，对应的就是 TiKV 里面的 PD。这部分主要实现了一个收集心跳的函数和一个 region 的调度器。

### processRegionHeartbeat()

在 `processRegionHeartbeat()` 收到汇报来的心跳，先检查一下 RegionEpoch 是否是最新的，如果是新的则调用 `c.putRegion()` 和 `c.updateStoreStatusLocked()`  进行更新。

判断 RegionEpoch 是否最新的方法，官方文档里已经有说明。Version 和 ConfVer 均最大的，即为最新。

### Schedule()

这一部分主要负责 region 的调度，从 region size 最大的 store 中取出一个 region 放到 region size 最小的 store 中。按如下流程处理即可。

1. 选出 suitableStores，并按照 regionSize 进行排序。SuitableStore 是那些满足 `DownTime()` 时间小于 `MaxStoreDownTime` 的 store。
2. 开始遍历 suitableStores，从 regionSize 最大的开始遍历，依次调用 `GetPendingRegionsWithLock()`，`GetFollowersWithLock()` 和 `GetLeadersWithLock()`。直到找到一个目标 region。如果实在找不到目标 region，直接放弃本次操作。
3. 判断目标 region 的 store 数量，如果小于 `cluster.GetMaxReplicas()`，直接放弃本次操作。
4. 再次从 suitableStores 开始遍历，这次从 regionSize 最小的开始遍历，选出一个目标 store，目标 store 不能在原来的 region 里面。如果目标 store 找不到，直接放弃。
5. 判断两个 store 的 regionSize 是否小于 `2*ApproximateSize` 。是的话直接放弃。
6. 调用 `cluster.AllocPeer()` 创建 peer，创建 `CreateMovePeerOperator` 操作，返回结果。

## 总结

整个 Project3 中最难的还是 project3B，里面会遇到形形色色的问题，重点是那先问题还不会稳定复现。只有多打日志，合理分析，多问问，才能克服问题。
