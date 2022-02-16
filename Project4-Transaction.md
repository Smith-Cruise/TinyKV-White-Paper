# Project4 Transactions

这一节实现的事务本应该需要和 TinySQL 配合使用，但是因为我们只有实现 TinyKV 部分，所以有些地方看起来有些割裂。

## Percolator

首先说明的一点是，Percolator 基于单行事务实现了多行事务，Google BigTable 能够提供单行事务。在这里 TinyKV 也会通过锁来保证单行数据的原子性（但是单行事务好像没有，可能要结合 TinySQL）。Percolator 这里不具体说了，自己看论文。

Percolator 采用了 Snapshot Isolation 来实现 ACID，其并不能避免 **write skew** 问题。

Percolator 提供了 5 种 Column Family 分别为 lock，write，data，notify，ack_O。在 TinyKV 中我们只需要使用 Lock，Write 和 Data。其中 Data 使用 Default 替代，下面还是统一以 Data 的形式说明。

Data：实际的数据，存在多版本，版本号就是写入事务的 startTs。
Lock：锁标记，版本号就是写入事务的 startTs，同时每一个 lock 上含有 primary key 的值。
Write：Write 上存在 startTs 和 commitTs，startTs 是指向对应 Data 中的版本，commitTs 是这个 Write 的创建时间。

### Primary Key 作用

Percolator 本质上是一个 2PC，但是传统 2PC 在 Coordinator 挂了的时候，就会留下一堆烂摊子，后续接任者需要根据烂摊子的状态来判断是继续 commit 还是 rollback。而 primary key 在这里的作用就是标记这个烂摊子是继续 commit 还是直接 rollback。

此外，一个事务要保证自身写的操作不会被别的事务干扰（防止写写冲突），也就是事务 $T_1$ 在修改 $A$ 的时候，别的事务就不能修改 $A$。Percolator 在所有的 key 上加了 lock，而 lock 里面含有 primary key 信息，也就是将这些锁的状态和 primary key 绑定。Primary key 的 lock 在，它们就在；Primary key 的 lock 不在，那么它们的 lock 也不能在。

Primary key 是从写入的 keys 里面随机选的，在 commit 时需要**优先处理** primary key。 当 primary key 执行成功后，其他的 key 可以异步并发执行，因为 primary key 写入成功代表已经决议完成了，后面的状态都可以根据 primary key 的状态进行判断。

### 获取 StartTs，CommitTs

开启一个事务后，会从**中心授时器（TSO）**获取一个 startTs，中心授时器必须保证时间是**递增唯一**的。同样执行 commit 时，也会从 TSO 获取一个时间戳作为 commitTs。

### 数据写入流程

所有写入的数据都会先被保存在 client 的缓存中，只有当执行 commit 的时候，才会执行 2PC 的提交。

#### 1. Prewrite

每一个 key 需要执行如下操作：

> 注意，如下 3 步操作需要保证原子性，也就是需要开启单行事务，BigTable 是支持的，不过在 TinyKV 中，我们使用 `server.Latches.AcquireLatches()` 实现。

1. 检查要写入的 key 是否存大于 startTs 的 Write，如果存在，直接 abort，说明在你的事务开启后，已经存在写入并已提交，也就是存在写-写冲突。
2. 检查要写入 key 的数据是否存在 Lock（任意时间戳），即检测是否存在写写冲突，如果存在直接 abort。
2. 如果通过上面两项检查，写入 Lock 和 Data，时间戳为你的 startTs。Lock 里面包含着 primary key 信息。

**第 2 步为什么是任意时间戳？**

Lock 的 startTs 小于当前事务的 startTs：如果你读了，就会产生脏读，因为前一个事务都没有 commit 你就读了。

Lock 的 startTs 大于当前事务的 startTs：如果你读了并修改了然后提交，拥有这个 lock 的事务会产生不可重复读。

Lock 的 startTs 等于当前事务的 startTs：不可能发生，因为当你重启事务之后，是分配一个新的 startTs，你不可能使用一个过去的 startTs 去执行重试操作。

#### 2. Commit

优先 commit primary key，和 prewrite 一样，需要开启单行事务。

1. 和获取 startTs 的方法一样，从中心授时器获取一个 commitTs。
2. 检查 key 的 lock 的时间戳是否为事务的 startTs，不是直接 abort。因为存在一种可能，在你 commit 的时候，你前面的 prewrite 操作因为过于缓慢，超时，导致你的 lock 被其他事务 rollback 了，然后你这里读取到的 lock 实际上不属于你的，是别的事务的。
3. 新增一条 Write，写入的 Write 包含了（startTs 和 commitTs），startTs 的作用是帮助你查找对应的 Data，因为 Data 的时间戳是 startTs。
4. 删除其对应的 lock。论文里面是删除所有 commit timestamp 之前的 lock，我觉得都可以吧，因为 commit timestamp 之前的 lock 也只有它自己之前加上的 lock，不然第二步就已经错了。

之后开始提交其他所有的 key，步骤和 primary key 一样。（可以异步并发操作，加快速度）。

**如果执行完 3 Write 后，在第 4 步清除 lock 挂了怎么办？**

不可能发生，我们开启了单行事务，如果第 4 步没有执行成功，那么 1,2,3 会被回滚，满足 atomicity。

### 数据读取

1. 读取某一个 key 的数据，检查其是否存在小于或等于 startTs 的 lock，如果存在说明在本次读取时还存在未 commit 的事务，先等一会，如果等超时了 lock 还在，则尝试 rollback。如果直接强行读会产生脏读，读取了未 commit 的数据。
2. 查询 commitTs 小于 startTs 的最新的 Write，如果不存在则返回数据不存在。
3. 根据 Write 的 startTs 从 Data 中获取数据。

### Rollback 回滚操作

Rollback 取决于 primary key 的状态，primary key 就是那个 commit 和 rollback 的分水岭。存在如下三种可能：

1. Primary key 的 Lock 还在，代表之前的事务没有 commit，就选择回滚。
2. Primary key 上面的 Lock 已经不存在，且有了 Write，那么代表 primary key 已经被 commit 了，这里我们选择继续推进 commit。
3. Primary key 既没有 Lock 也没有 Write，那么说明之前连 Prewrite 阶段都还没开始，客户端重试即可。

### Percolator 额外补充

**会不会产生死锁？**

不可能，因为你可以注意到，任何事务遇到冲突都是回滚自身，而不会等待。（破坏死锁成立条件之一：循环等待）。但是会有可能产生活锁，下面推荐资料里有说。

**推荐资料：**

推几个链接，可以看一看，有介绍 Percolator 的 GC 如何实现等等。

[深入理解分布式事务Percolator(一)](https://blog.csdn.net/maxlovezyy/article/details/88572692)

[深入理解分布式事务Percolator(二)](https://blog.csdn.net/maxlovezyy/article/details/99702091)

[深入理解分布式事务Percolator(三)](https://blog.csdn.net/maxlovezyy/article/details/99707690)

[TiDB 事务模型](https://book.tidb.io/session1/chapter6/tidb-transaction-mode.html)

## transaction.go

这里我们需要实现底层的 `MvccTxn` ，存粹只是一些基本的操作，比如写 Write，Lock 等等，这里着重讲几个方法。

### GetValue()

查询当前事务下，传入 key 对应的 Value。

1. 通过 `iter.Seek(EncodeKey(key, txn.StartTS))` 查找遍历 Write。
2. 判断找到 Write 的 key 是不是就是自己需要的 key，如果不是，说明不存在，直接返回。
3. 判断 Write 的 Kind 是不是 `WriteKindPut`，如果不是，说明不存在，直接返回。
4. 从 Default 中通过 `EncodeKey(key, write.StartTS)` 获取值。

### CurrentWrite()

查询当前事务下，传入 key 的最新 Write。

1. 通过 `iter.Seek(EncodeKey(key, math.MaxUint64))` 查询该 key 的最新 Write。
2. 如果 `write.StartTS > txn.StartTS`，继续遍历，直到找到 `write.StartTS == txn.StartTS` 的 Write。
3. 返回这个 Write 和 commitTs。

### MostRecentWrite()

查询传入 key 的最新 Write，这里不需要考虑事务的 startTs。

1. 通过 `iter.Seek(EncodeKey(key, math.MaxUint64))` 查找。
2. 判断目标 Write 的 key 是不是我们需要的，不是返回空，是直接返回该 Write。

## server.go

在 Percolator 中，Google Table 提供了单行锁，其能保证单行的操作是原子的，但是在 TinyKV 中，并不提供这样的保证。因为我们其实是多行的数据，不是单行。所以我们需要 `server.go` 中提供的 `Latches` 来保证对同一个 key 修改的原子性。（为什么我们是多行？看 Project1。）

当然 `Latches` 你不用也没事，因为测试用例根本测不出来，希望 TinyKV 以后在这里能够改进。

### KvGet()

获取单个 key 的 Value，步骤如下：

1. 通过 Latches 上锁对应的 key。
2. 获取 Lock，如果 Lock 的 startTs 小于当前的 startTs，说明存在你之前存在尚未 commit 的请求，中断操作，返回 `LockInfo`。
3. 否则直接获取 Value，如果 Value 不存在，则设置 `NotFound = true`。

### KvPrewrite()

进行 2PC 阶段中的第一阶段。

1. 对所有的 key 上锁。
2. 通过 `MostRecentWrite` 检查所有 key 的最新 Write，如果存在，且其 commitTs 大于当前事务的 startTs，说明存在 write conflict，终止操作。
3. 通过 `GetLock()` 检查所有 key 是否有 Lock，如果存在 Lock，说明当前 key 被其他事务使用中，终止操作。
4. 到这一步说明可以正常执行 Prewrite 操作了，写入 Default 数据和 Lock。

### KvCommit()

进行 2PC 阶段中的第二阶段。

1. 通过 Latches 上锁对应的 key。
2. 尝试获取每一个 key 的 Lock，并检查 `Lock.StartTs` 和当前事务的 startTs 是否一致，不一致直接取消。因为存在这种情况，客户端 Prewrite 阶段耗时过长，Lock 的 TTL 已经超时，被其他事务回滚，所以当客户端要 commit 的时候，需要先检查一遍 Lock。
3. 如果成功则写入 Write 并移除 Lock。

### KvScan()

这里需要自己实现一个 Scanner 先，其实和 Get 的流程差不多，无非就是单个 key 变成了批量 key，这里不说了。

### KvCheckTxnStatus()

用于 Client failure 后，想继续执行时先检查 Primary Key 的状态，以此决定是回滚还是继续推进 commit。

1. 通过 `CurrentWrite()` 获取 primary key 的 Write，如果不是 `WriteKindRollback`，则说明已经被 commit，不用管了，返回其 commitTs。
2. 检查 primary key 是否有 Lock，如果没有 lock，说明 primary key 已经被回滚了，创建一个 `WriteKindRollback` 并直接返回。
3. 检查 Lock 的 TTL，判断 Lock 是否超时，如果超时，移除该 Lock 和 Value，并创建一个 `WriteKindRollback` 标记回滚。否则直接返回，等待 Lock 超时为止。

### KvBatchRollback()

用于批量回滚 key 的操作。

1. 通过 `CurrentWrite` 获取 Write，如果已经是 `WriteKindRollback`，说明这个 key 已经被回滚完毕，跳过这个 key。
2. 否则先获取 Lock，如果获取 Lock 的 startTs 不是当前事务的 startTs，则终止操作，说明该 key 被其他事务拥有。
3. 否则移除 Lock，删除 Value 并增加 `WriteKindRollback`。

### KvResolveLock()

这个方法主要用于解决锁冲突，当客户端已经通过 `KvCheckTxnStatus()` 检查了 primary key 的状态，这里打算要么全部回滚，要么全部提交，具体取决于 `ResolveLockRequest` 的 CommitVersion。

```Go
if req.CommitVersion == 0 {
  // server.KvBatchRollback()
} else {
  // server.KvCommit()
}
```

## 总结

这一部分并不是很难，但是要搞懂事务背后的流程，不然很容易就是面向测试用例编程。
