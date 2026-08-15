# Raft 查漏补缺与面试准备

> 目标：不是背诵术语，而是建立一套能推导答案的模型。学习完成后，应当能在白板上独立画出 Leader 选举、日志复制、提交、网络分区恢复和快照追赶流程，并解释每个字段为什么存在。

## 一、先修正当前最容易混淆的地方

本次交流暴露出的核心问题不是“不会 Raft”，而是把三种不同维度的状态混在了一起：

1. **领导权维度**：现在处于哪个任期、谁是 Leader。
2. **日志历史维度**：每个位置保存了哪一任 Leader 创建的什么命令。
3. **处理进度维度**：日志复制、提交、应用分别进行到了哪里。

后续遇到任何字段，先判断它属于哪个维度。

当前需要重点修正的误区：

- `currentTerm` 不会改写旧日志的 term。
- `prevLogIndex/prevLogTerm` 不是随便选择的，由每个 Follower 的 `nextIndex` 推导。
- “复制到多数节点”不总是等于“可以直接提交”，还要检查日志是否属于当前任期。
- `commitIndex` 不是持久化进度，而是安全提交水位。
- `lastApplied` 不是持久化进度，而是状态机执行进度。
- “已提交”只表示多数派已经形成不可逆决定，不表示所有节点都有该日志。
- Snapshot 不是“一致性证明”，而是被压缩日志产生的完整状态机结果。
- 日志回退由 Leader 驱动，Follower 只负责接受、拒绝、截断和追加。
- “只从 Leader 读取”并不自动等于线性一致读，还必须确认 Leader 身份并等待状态机追上。

---

## 二、Raft 的三条认知轴

### 2.1 领导权轴：Term

每个节点都维护：

```text
currentTerm
role = Follower | Candidate | Leader
votedFor
```

`currentTerm` 是该节点见过的最大任期号，是一个持久化的逻辑时钟。

稳定状态下，Leader 和能收到其心跳的 Follower 通常处于同一任期。网络分区时，不同节点的 `currentTerm` 可以暂时不同。

规则：

- 节点发起新一轮选举前先增加 `currentTerm`。
- 所有 Raft RPC 都携带发送者的 term。
- 收到更大 term 后，节点更新 `currentTerm` 并退回 Follower。
- 收到更小 term 的 RPC，直接拒绝。
- `votedFor` 记录本任期投给了谁，保证每个节点每任期最多投一票。

### 2.2 日志历史轴：Index 与 Entry Term

每条日志至少包含：

```text
LogEntry {
    index
    term
    command
}
```

含义：

- `index`：日志在序列中的位置。
- `term`：创建该日志时 Leader 的任期。
- `command`：需要状态机执行的操作。

例如：

```text
log[3] = {
    index: 3,
    term: 2,
    command: "set x = 10"
}
```

表示第 3 条日志由 Term 2 的 Leader 创建。

最重要的规则：

> 日志 entry 的 term 在创建后永远不变。

如果现在的 Leader 是 Term 4，它重新复制一条 Term 2 的旧日志，这条日志仍然是：

```text
log[N].term = 2
```

不会变成 4。

### 2.3 处理进度轴：复制、提交、应用

Leader 为每个 Follower 维护：

```text
nextIndex[Follower]
matchIndex[Follower]
```

所有节点维护：

```text
commitIndex
lastApplied
```

含义：

- `nextIndex[F]`：Leader 下一次准备从哪个 index 开始给 Follower F 发送日志。
- `matchIndex[F]`：Leader 已确认 Follower F 与自己匹配到的最高 index。
- `commitIndex`：该节点已知“已经安全提交、不会被覆盖”的最高 index。
- `lastApplied`：该节点状态机已经执行完成的最高 index。

基本不变量：

```text
lastApplied <= commitIndex <= lastLogIndex
```

成功复制后通常有：

```text
nextIndex[F] = matchIndex[F] + 1
```

完整处理链：

```text
日志持久化
    ↓
复制到其他节点
    ↓
达到安全提交条件，推进 commitIndex
    ↓
按顺序应用到状态机，推进 lastApplied
```

这四个阶段不能混为一谈。

---

## 三、用一个统一例子认识所有字段

假设当前 Leader 的状态如下：

```text
Leader.currentTerm = 4

index:       1    2    3    4
log.term:    1    2    2    4
command:     A    B    X   no-op
状态:       已提交 已提交 未提交 未提交

commitIndex = 2
lastApplied = 2
```

逐项解释：

- 当前领导权处于 Term 4。
- index 3 的日志是在 Term 2 创建的，虽然现在由 Term 4 的 Leader 保存和复制，它仍属于 Term 2。
- index 4 的 no-op 是当前 Term 4 创建的日志。
- index 1～2 已经提交并应用。
- index 3～4 仍可能被覆盖，不能应用到状态机。

假设某个 Follower 的：

```text
nextIndex[F] = 3
```

Leader 将计算：

```text
prevLogIndex = nextIndex[F] - 1 = 2
prevLogTerm  = Leader.log[2].term = 2
```

然后发送：

```text
AppendEntries {
    term: 4,
    prevLogIndex: 2,
    prevLogTerm: 2,
    entries: [
        (index=3, term=2, command=X),
        (index=4, term=4, command=no-op)
    ],
    leaderCommit: 2
}
```

这个例子后面会反复使用。

---

## 四、Leader 选举

### 4.1 选举过程

稳定状态下，Leader 周期性发送心跳。心跳不是“选举超时事件”，而是：

```text
entries = []
```

的 `AppendEntries`。

Follower 本地维护随机化的选举计时器。如果在一个选举超时窗口内既没有收到合法 Leader 的 AppendEntries，也没有向合法 Candidate 授票，它会：

1. 增加 `currentTerm`。
2. 从 Follower 变成 Candidate。
3. 给自己投票，并持久化 `currentTerm` 和 `votedFor`。
4. 重置自己的随机选举计时器。
5. 并发向其他节点发送 `RequestVote`。
6. 获得整个配置中多数节点的票后成为 Leader。

多数票计算：

```text
quorum = floor(N / 2) + 1
```

例如：

```text
3 节点需要 2 票
5 节点需要 3 票
```

自己的票也计入多数票。

### 4.2 RequestVote 携带什么

核心字段：

```text
RequestVote {
    term
    candidateId
    lastLogIndex
    lastLogTerm
}
```

投票者只有在以下条件都满足时才会投票：

1. Candidate 的 term 不小于自己的 `currentTerm`。
2. 自己在这个 term 尚未投票，或者已经投给同一个 Candidate。
3. Candidate 的日志至少和自己一样新。

### 4.3 日志新旧如何比较

按以下顺序比较：

```text
先比较 lastLogTerm
term 相同，再比较 lastLogIndex
```

等价条件：

```text
candidate.lastLogTerm > voter.lastLogTerm
||
(
    candidate.lastLogTerm == voter.lastLogTerm
    &&
    candidate.lastLogIndex >= voter.lastLogIndex
)
```

例子：

```text
Candidate A: (lastLogTerm=5, lastLogIndex=10)
Candidate B: (lastLogTerm=4, lastLogIndex=100)
```

A 被认为更新，因为 term 优先于 index。日志条数更多不等于日志更新。

### 4.4 两个 Candidate 同时发起选举

可能发生瓜分选票，没有任何人获得多数。

此时每个 Candidate 等待自己的随机选举超时。更早超时的节点进入更高任期并重新发起选举，通常会先拿到多数票。

随机化超时的目的不是保证永不冲突，而是降低反复同时发起选举的概率。

### 4.5 任期变化时如何退位

任何节点收到比自己更大的 term：

```text
remoteTerm > currentTerm
```

都必须：

```text
currentTerm = remoteTerm
role = Follower
votedFor = null
```

这使网络分区中的旧 Leader 在重新接触集群后自动退位。

还有一个容易漏掉的规则：Candidate 收到 `term == currentTerm` 的合法 AppendEntries，说明本任期已经选出了 Leader，也应当退回 Follower。只有比较更大 term 才退位是不完整的。

### 4.6 PreVote 是什么

PreVote 是常见工程扩展，不是理解基础 Raft 的前置条件。

节点正式增加 term 之前，先询问其他节点：“如果我进入下一任期，你们是否可能投我？”

作用是避免被隔离的节点反复增加 term。否则它恢复网络后可能用一个很大的 term 迫使健康 Leader 退位，引发无意义的重新选举。

---

## 五、AppendEntries 与日志复制

### 5.1 AppendEntries 的字段

```text
AppendEntries {
    term
    leaderId
    prevLogIndex
    prevLogTerm
    entries[]
    leaderCommit
}
```

字段作用：

- `term`：Leader 当前任期。
- `leaderId`：供客户端重定向等用途。
- `prevLogIndex`：本批新日志前一条日志的位置。
- `prevLogTerm`：Leader 本地 `log[prevLogIndex]` 的 term。
- `entries[]`：需要追加的日志；为空时就是心跳。
- `leaderCommit`：Leader 当前已知的提交水位。

### 5.2 prevLogIndex/prevLogTerm 如何确定

Leader 针对每个 Follower 独立计算：

```text
prevLogIndex = nextIndex[Follower] - 1
prevLogTerm  = Leader.log[prevLogIndex].term
```

快照边界需要分三种情况：

```text
prevLogIndex == snapshot.lastIncludedIndex
    → prevLogTerm = snapshot.lastIncludedTerm

prevLogIndex > snapshot.lastIncludedIndex
    → 从 Leader 仍保留的日志中读取 term

prevLogIndex < snapshot.lastIncludedIndex，且旧日志已被删除
    → 无法构造该位置的 prevLogTerm，应改发 InstallSnapshot
```

这两个字段表达：

> 你我能不能在这个位置接上同一段历史？

它们不是：

- 当前 `commitIndex`；
- Leader 的 `currentTerm`；
- 全集群统一的固定值。

不同 Follower 的进度不同，因此每个 Follower 收到的 `prevLogIndex/prevLogTerm` 可以不同。

### 5.3 Leader 刚当选时如何初始化

Leader 不知道每个 Follower 的真实进度，先乐观假设它们和自己一样新：

```text
nextIndex[F] = Leader.lastLogIndex + 1
matchIndex[F] = 0   // 或实现中的“未知”初始状态
```

假设 Leader 最后日志是：

```text
(index=5, term=4)
```

第一次会尝试：

```text
nextIndex[F] = 6
prevLogIndex = 5
prevLogTerm  = 4
```

Follower 不匹配就拒绝，Leader 回退 `nextIndex` 后重试。

### 5.4 Follower 收到 AppendEntries 后做什么

简化流程：

1. 如果 RPC term 小于 `currentTerm`，拒绝。
2. 如果 RPC term 更大，更新 term 并退回 Follower；Candidate 收到同 term 的合法 AppendEntries 也退回 Follower。
3. 检查本地是否存在 `prevLogIndex`。
4. 检查该位置的 term 是否等于 `prevLogTerm`。
5. 不匹配则拒绝。
6. 匹配后，从 `entries[]` 的第一条冲突日志开始处理。
7. 如果同一 index 的 term 不同，删除该位置及其后全部日志。
8. 追加 Leader 发来的剩余日志。
9. 仅当 `leaderCommit > commitIndex` 时推进提交水位：

   ```text
   commitIndex = min(
       leaderCommit,
       本次 RPC 已验证与 Leader 匹配的最高 index
   )
   ```

   这样既保证 `commitIndex` 单调递增，也不会提交尚未验证匹配的本地后缀。

### 5.5 同一 index 的 term 相同怎么办

如果 Follower 已有：

```text
(index=N, term=T)
```

Leader 发来的同一位置也是：

```text
(index=N, term=T)
```

则认为是同一条日志：

- 不删除；
- 不重复追加；
- 继续检查下一条。

这是日志匹配性质：

> 两份日志中如果某条日志的 index 和 term 都相同，那么它之前的日志也相同。

在正确的、非拜占庭 Raft 实现中，不会出现“index 和 term 相同，但 command 不同”。如果出现，说明实现、存储介质或数据完整性已经违反 Raft 的基本假设。

### 5.6 同一 index 的 term 不同怎么办

例如：

```text
Leader:   (1,t1) (2,t1) (3,t2) (4,t3)
Follower: (1,t1) (2,t1) (3,t2) (4,t4) (5,t4)
```

双方在 index 3 匹配，在 index 4 冲突。

Follower 执行：

```text
删除本地 index 4 及其全部后缀
追加 Leader 的 (4,t3)
```

未提交日志可以被覆盖，已提交日志不会发生这种冲突。

### 5.7 谁负责回退，谁负责删除

- Leader 维护并回退 `nextIndex`，负责寻找共同前缀。
- Follower 负责拒绝不匹配请求。
- 找到共同前缀后，Follower 负责删除冲突后缀并追加 Leader 日志。

所以不能说“Follower 主动回退寻找一致点”。

### 5.8 冲突快速回退

最简单实现每次：

```text
nextIndex--
```

但大量冲突日志会产生很多 RPC。

常见优化是 Follower 返回：

```text
conflictTerm
conflictIndex
```

Leader 一次跳过整个冲突任期，而不是逐条回退。

### 5.9 心跳为什么也带 prevLogIndex/prevLogTerm

心跳本质是：

```text
entries = []
```

的 AppendEntries，因此仍然携带日志锚点和 `leaderCommit`。

它同时完成：

- 证明 Leader 存活；
- 重置 Follower 的选举计时器；
- 传播 Leader 的提交水位；
- 检测日志是否落后或冲突；
- 维持 ReadIndex、CheckQuorum 或租约所需的多数派联系。

注意：空心跳本身不一定立即删除 Follower 多出来的未提交后缀。当 Leader 以后在相同 index 发送冲突日志时，Follower 才会按规则截断。

---

## 六、复制、提交与应用

### 6.1 matchIndex 表达什么

Leader 收到 Follower 对 AppendEntries 的成功确认后，更新：

```text
matchIndex[F] = 本次确认匹配的最高 index
nextIndex[F]  = matchIndex[F] + 1
```

`matchIndex[F] >= N` 的含义不是“Follower 有任意 N 条日志”，而是：

> Leader 已确认 Follower 的日志至少到 index N 都与自己匹配。

### 6.2 commitIndex 表达什么

```text
commitIndex = N
```

表示：

> index 1～N 已经成为不可逆的集群决定，不会再被未来 Leader 覆盖。

它不是：

- 已经写到几个节点；
- 已经刷盘到哪里；
- 状态机已经执行到哪里；
- 与 Leader 日志完全同步到哪里。

### 6.3 Leader 如何推进 commitIndex

Leader 寻找最大的 N，使：

```text
N > commitIndex

多数节点的 matchIndex >= N

Leader.log[N].term == Leader.currentTerm
```

然后：

```text
commitIndex = N
```

Leader 自己也计入多数节点。

例如 5 节点：

```text
Leader = 4
F1     = 4
F2     = 4
F3     = 3
F4     = 2
```

已经有 3/5 节点匹配到 index 4。

如果：

```text
Leader.currentTerm = 4
Leader.log[4].term = 4
```

就可以推进：

```text
commitIndex = 4
```

### 6.4 log[N].term == currentTerm 检查谁

检查的是 Leader 自己的日志：

```text
Leader.log[N].term == Leader.currentTerm
```

它表达：

> Leader 准备直接提交的第 N 条日志，是不是在自己当前任期创建的？

不是让每个 Follower 分别检查：

```text
Follower.log[N].term == Follower.currentTerm
```

Follower 成功确认 `matchIndex >= N` 后，Leader 已经知道该 Follower 保存的是同一条 `(index, term)` 日志。

### 6.5 为什么旧任期日志不能仅凭多数副本直接提交

这是 Raft 最容易答错的地方。

假设 5 个节点 A、B、C、D、E。

Term 2：

```text
A 是 Leader
A 创建 X(index=2, term=2)
X 只复制到 A、B
```

Term 3：

```text
A 宕机
E 当选 Leader
E 创建冲突日志 Y(index=2, term=3)
Y 暂时只存在于 E
```

Term 4：

```text
E 宕机，A 恢复并再次当选
A 把旧日志 X 复制给 C
```

此时：

```text
A: X(term=2)
B: X(term=2)
C: X(term=2)
D: 空
E: Y(term=3)
```

X 已经存在于 3/5 节点，但 X 仍然是 Term 2 的日志。

如果 A 此时宕机，E 发起 Term 5 选举：

- E 最后日志 term=3；
- C 最后日志 term=2；
- E 的日志在投票规则中被认为更新；
- C 可以投票给不含 X 的 E；
- E 获得 C、D、E 三票后成为 Leader；
- E 的 Y 最终会覆盖 C 上的 X。

所以：

> 一条旧任期日志即使暂时存在于多数节点，也不能仅凭当前副本数宣布它首次提交。

### 6.6 当前任期日志为什么能成为“安全封条”

Term 4 的 A 应当再追加：

```text
Z(index=3, term=4, command=no-op)
```

并把它复制到 A、B、C。

此时：

```text
A/B/C:
    X(index=2, term=2)
    Z(index=3, term=4)
```

任何未来 Candidate 都需要多数票。它至少要从 A、B、C 中获得一票。

一个不含 Z、最后日志 term 只有 3 的 Candidate，会被拥有 Term 4 日志的节点拒绝。结合多数派相交和选举限制，未来合法 Leader 必须包含 Z，也必然包含 Z 前面的 X。

因此：

```text
多数节点 matchIndex >= 3
log[3].term == currentTerm == 4
```

Leader 可以：

```text
commitIndex = 3
```

index 3 被直接提交，index 2 随日志前缀一起被间接提交。

注意：

```text
log[2].term 仍然是 2
```

提交不会修改 entry 的出生任期。

### 6.7 no-op 的作用

新 Leader 常在自己的 term 追加一条 no-op：

```text
LogEntry {
    term: currentTerm,
    command: no-op
}
```

它没有业务效果，但有共识效果：

- 建立当前任期日志；
- 确认当前 Leader 能联系多数派；
- 提交继承自旧 Leader 的日志前缀；
- 为安全线性读建立当前任期的提交基础。

也可以由当前任期的真实客户端命令承担相同作用，但没有客户端流量时 no-op 能主动完成这件事。

### 6.8 lastApplied 表达什么

```text
lastApplied = N
```

表示状态机已经执行完 index 1～N。

例如：

```text
lastLogIndex = 8
commitIndex  = 5
lastApplied  = 3
```

状态分类：

```text
1～3：已提交、已应用
4～5：已提交、尚未应用
6～8：本节点已有但未提交、未应用，可能被覆盖
```

`lastApplied` 不是持久化位置。日志通常在复制成功响应前已经持久化，但状态机可能仍未执行。

---

## 七、网络分区与故障恢复

### 7.1 五节点分成 2 + 3

假设原 Leader 位于 2 节点少数派，另 3 个节点组成多数派。

少数派旧 Leader：

- 可能暂时仍认为自己是 Leader；
- 可以把新日志写到自己和另一个节点；
- 2/5 不是多数，无法推进 `commitIndex`；
- 不能向客户端返回写成功；
- 无法确认自己仍掌握领导权，不能保证线性一致读。

多数派：

- 可以进入更高 term；
- 可以选出新 Leader；
- 可以提交新的写；
- 满足 ReadIndex 等条件后可以提供线性一致读。

### 7.2 旧 Leader 什么时候知道自己失去领导权

基础 Raft 中，旧 Leader 在完全隔离期间不一定立即知道自己已被替代。

网络恢复后，它收到更高 term 的 RequestVote、AppendEntries 或响应：

```text
remoteTerm > currentTerm
```

便更新 term 并退回 Follower。

常见 `CheckQuorum` 扩展会要求 Leader 持续联系多数节点。如果一个选举超时时间内联系不到多数派，Leader 主动退位，减少旧 Leader 长时间自认为有效的问题。

### 7.3 少数派未提交日志如何处理

旧 Leader 在少数派期间追加的日志：

- 没有提交；
- 没有应用到状态机；
- 客户端没有收到成功确认。

网络恢复后：

1. 新 Leader 用更高 term 使旧 Leader 退回 Follower。
2. 新 Leader 通过 `nextIndex` 回退寻找共同前缀。
3. Follower 在第一条冲突日志处删除后缀。
4. 追加新 Leader 的权威日志。

这些日志可以安全丢弃，因为系统从未对外承诺它们成功。

客户端如果只收到超时，无法判断请求是否提交，应使用请求 ID、幂等操作或去重表安全重试。

---

## 八、线性一致读

### 8.1 线性一致读的要求

如果写操作已经向客户端返回成功，那么之后开始的读必须看到这次写。

与写并发发生的读可以看到旧值或新值，但必须存在一个尊重真实时间顺序的原子执行点。

### 8.2 为什么“只读 Leader”还不够

网络分区中的旧 Leader 可能尚未得知自己已经失去领导权。

如果它只读本地状态机，可能返回新 Leader 提交之前的旧数据，违反线性一致性。

所以 Leader 在提供线性一致读前至少要确认：

1. 自己仍是当前 Leader。
2. 状态机已应用到本次读要求的提交位置。

### 8.3 三种常见读方案

#### 方案一：把读写入日志

把读请求作为日志条目复制并提交，然后执行读取。

优点：

- 推理最简单；
- 安全性最直接。

缺点：

- 每次读都写日志；
- 需要多数派复制；
- 延迟和写放大最高。

#### 方案二：ReadIndex

典型流程：

1. 新 Leader 先确保本任期至少有一条日志已经提交，通常是 no-op。否则它可能还不知道旧任期日志真正提交到了哪里。
2. Leader 记录当前 `commitIndex`，作为本次 `readIndex`。
3. 携带本次读专属的 context 与多数节点交换心跳，并等待当前任期的多数派响应，不能复用旧心跳 ACK。
4. 等待：

   ```text
   lastApplied >= readIndex
   ```

5. 从本地状态机读取。

ReadIndex 不为每次读追加日志，但通常需要一次多数派确认。

#### 方案三：Lease Read

Leader 根据最近一次多数派心跳和租约时间，认为租约有效期内不会出现另一个合法 Leader，于是直接读本地状态机。

优点是最快，缺点是正确性依赖：

- 有界时钟漂移；
- 合理的租约与选举超时配置；
- CheckQuorum 等实现约束。

### 8.4 Follower 能否提供线性一致读

可以设计，但不能随便读本地状态。

Follower 通常需要：

1. 从 Leader 获取安全的 ReadIndex。
2. 等待自己的 `lastApplied >= readIndex`。
3. 再读取本地状态机。

否则只能明确提供 stale read。

---

## 九、哪些状态必须持久化

### 9.1 Raft 论文中的持久状态

所有节点必须持久化：

```text
currentTerm
votedFor
log[]
```

并且必须在回复相关 RPC 之前持久化。

原因：

- `currentTerm`：任期不能在重启后倒退。
- `votedFor`：节点不能重启后忘记投票，在同一 term 投给两个 Candidate。
- `log[]`：Follower 的成功确认可能被 Leader 计入多数派，重启后不能丢失已确认日志。

快照及其元数据也必须可靠保存：

```text
lastIncludedIndex
lastIncludedTerm
snapshot state
configurationAsOf(lastIncludedIndex)
```

这里保存的是截至快照边界已经进入日志历史的最新配置，不能错误地打包快照边界之后的配置。

### 9.2 论文中的易失状态

所有节点：

```text
commitIndex
lastApplied
```

Leader 额外维护：

```text
nextIndex[]
matchIndex[]
```

恢复方式：

- `commitIndex` 可以通过 Leader 的 `leaderCommit` 重新获知。
- `lastApplied` 可以通过快照和已提交日志重放恢复。
- 新 Leader 重新初始化 `nextIndex/matchIndex`。

### 9.3 工程实现中的补充

论文把 `lastApplied` 列为易失状态，是建立在状态机能够从快照和日志重建的模型上。

如果状态机本身持久化且命令有外部副作用，工程实现通常还要原子保存：

- 状态机快照；
- 对应的 applied index；
- 请求去重信息。

否则重启后重复应用非幂等命令可能产生重复副作用。

---

## 十、Snapshot 与日志压缩

### 10.1 为什么需要 Snapshot

Raft 日志不能无限增长。每个节点可以把已经应用到状态机的前缀压缩成快照：

```text
Snapshot {
    stateMachineState
    lastIncludedIndex
    lastIncludedTerm
    configurationAsOf(lastIncludedIndex)
}
```

然后删除快照覆盖的日志前缀。

快照点必须来自已提交并已应用的状态：

```text
lastIncludedIndex <= lastApplied <= commitIndex
```

### 10.2 Leader 和 Follower 都会创建快照

创建本地 Snapshot 不是 Leader 专属行为。

每个节点都可以根据自己的：

- 日志大小；
- lastApplied；
- 存储空间；
- 快照策略；

独立创建快照并回收日志。

不同节点的快照位置可以不同：

```text
Leader     snapshot index = 1000
Follower A snapshot index = 900
Follower B snapshot index = 700
```

正确性不受影响。

### 10.3 为什么创建快照不等待所有 Follower

“已提交”只要求多数派，不要求所有节点。

如果 Leader 必须等待最慢 Follower 才能压缩日志，那么一个长期宕机的节点会让所有节点的日志无限增长。

因此日志压缩是本地行为，不以最慢 Follower 的进度作为安全前提。

实现可以为了性能额外保留一段旧日志，减少发送快照的概率，但这只是优化。

### 10.4 Snapshot 不是一致性证明

假设：

```text
Leader 已应用到 index=1000
Leader 已把 1～1000 压缩成快照
Follower 只应用到 index=100
```

Leader 已经删除日志 101～1000。

如果直接从 index=1001 给 Follower 发送日志，Follower 的状态机会变成：

```text
执行了 1～100
跳过 101～1000
继续执行 1001 以后
```

状态必然错误。

Snapshot 传输的是：

> 执行完 1～1000 后的完整状态机物化结果。

它不是一句“这些日志已经取得一致”的证明。

### 10.5 什么时候发送 InstallSnapshot

当 Leader 为某个 Follower 回退 `nextIndex` 时，发现 Follower 需要的下一条日志已经不在 Leader 当前可用的日志范围内，就必须发送快照。

如果实现已删除快照边界前的全部旧日志，常见判断表现为：

```text
nextIndex[F] <= snapshot.lastIncludedIndex
```

如果实现为了性能在快照之外又保留了一段旧日志，则应先继续使用这些日志；不能只看 `nextIndex <= lastIncludedIndex` 就机械发送快照。

Leader 无法再用 AppendEntries 补齐，只能发送：

```text
InstallSnapshot {
    term
    leaderId
    lastIncludedIndex
    lastIncludedTerm
    offset
    data
    done
}
```

大快照通常分块传输，Follower 完整接收后原子安装，不能让半个快照暴露给状态机。

### 10.6 lastIncludedIndex/lastIncludedTerm 的作用

例如：

```text
lastIncludedIndex = 1000
lastIncludedTerm  = 8
```

表示：

- 快照包含了日志 1～1000 的状态机结果；
- 原日志 index 1000 的 term 是 8。

这对 `(index, term)` 是日志压缩后的“虚拟锚点”。

它用于：

- 判断 Follower 的后缀是否建立在同一历史上；
- 后续从 index 1001 恢复 AppendEntries；
- 参与日志新旧判断；
- 保持日志匹配性质跨越快照边界。

它不是 Follower 主动回退的起点。

### 10.7 Follower 安装快照时如何处理旧日志

情况一：Follower 本地存在：

```text
(index=1000, term=8)
```

与快照边界相同。

根据日志匹配性质，双方到 index 1000 的历史一致。Follower 可以：

- 安装快照；
- 删除 index 1000 及之前被快照覆盖的日志；
- 保留 index 1000 之后的日志后缀。

保留后缀不等于提交后缀。后续 AppendEntries 如果发现冲突，仍会截断。

情况二：Follower 本地：

```text
log[1000].term != 8
```

或者根本没有 index 1000。

无法证明原后缀建立在同一历史上，应：

- 丢弃原有日志；
- 原子安装快照；
- 从 index 1001 重新接收 Leader 日志。

### 10.8 安装后如何更新进度

对于一个比本地状态更新的快照：

```text
commitIndex = max(commitIndex, lastIncludedIndex)
lastApplied = max(lastApplied, lastIncludedIndex)
```

状态机直接恢复为快照状态，不是逐条重放被压缩日志。

Leader 收到安装成功响应后，可将该 Follower 的复制进度视为至少到达：

```text
matchIndex = lastIncludedIndex
nextIndex  = lastIncludedIndex + 1
```

然后恢复 AppendEntries。

### 10.9 本地 Snapshot 与 InstallSnapshot 的区别

- **本地 Snapshot**：Leader 和 Follower 都能独立创建，用于压缩自己的日志。
- **InstallSnapshot RPC**：只能由当前 Leader 驱动，用于让落后过多的 Follower 追上。

Follower 不会把自己的快照主动推给 Leader。Raft 的日志同步始终由当前 Leader 驱动。

---

## 十一、成员变更

成员配置本身也属于共识状态，不能在没有安全过渡协议的情况下直接从旧配置切换到新配置。

如果旧、新配置各自形成不相交的多数派，可能同时选出两个 Leader。

经典做法是 Joint Consensus：

1. Leader 把联合配置 `C_old,new` 追加到日志。
2. 节点一旦把该配置写入自己的日志，就立即使用“本地日志中的最新配置”参与后续选举和提交判断，不等待它先被提交。
3. `C_old,new` 及联合阶段的其他日志，都必须同时获得旧配置多数派和新配置多数派。
4. 联合配置提交后，Leader 再追加纯新配置 `C_new`。
5. 节点收到 `C_new` 后同样立即按新配置工作，最终由新配置多数派提交它。

部分工程实现使用一次只增加或删除一个投票节点的安全变更流程，并要求：

- 变更串行执行；
- 任意时刻至多存在一条未提交配置；
- 前一次配置提交后才能继续下一次；
- Leader 先建立当前任期的已提交日志，再发起配置变化；
- 节点对本地日志中的最新配置立即生效，而不是等配置提交；
- 新节点通常先作为 Learner 追赶，再获得投票权，避免降低可用性。

Learner 追赶主要解决可用性问题，不能替代配置变更协议本身的安全条件。不要把“新增节点加入通信列表”等同于“立即获得投票权”，也不要自行拼装未经证明的单节点变更流程。

---

## 十二、Raft 与 Ceph、DAOS 的边界

### 12.1 Ceph 不是整个集群一个数据 Leader

Ceph 中：

- MON 维护集群地图；
- 每个 PG 有一个 Primary OSD；
- 普通 RADOS 写由 PG Primary 排序并复制到 acting set；
- 默认读也由 PG Primary 处理。

不同 PG 的 Primary 分布在不同 OSD：

```text
PG1: [OSD1(primary), OSD2, OSD3]
PG2: [OSD2(primary), OSD3, OSD1]
PG3: [OSD3(primary), OSD1, OSD2]
```

因此集群总体读写负载可按 PG 分散，但单个热点 PG 默认仍受其 Primary 限制。

Ceph 处理读热点的常见方式：

- `pg-upmap-primary` / read balancer 调整 Primary 分布；
- RBD `read_from_replica=balance/localize` 从副本读取；
- 增加 PG 数分散多个对象；
- 上层拆分单个热点对象。

这些机制不能把单个 PG 的写操作变成多个 Primary 并行提交。

Ceph 在单个 RADOS 对象、当前 PG interval 内的一致性主要来自：

- Primary 排序；
- 按当前可写 acting set 和 `min_size` 规则完成副本持久化；
- PG peering 选出权威历史；
- read lease 防止被隔离的旧 Primary 返回陈旧数据。

这不代表跨多个 RADOS 对象存在全局线性一致事务。它也不是直接复用一个全局 Raft 日志，Ceph 的写确认规则不能简单类比成 Raft 多数派提交。

### 12.2 DAOS 也不是所有数据都经过 Raft Leader

DAOS 中“Leader”至少有多种含义：

- RDB Service Leader：维护 pool/container 等服务元数据；
- DTX Leader：协调某笔分布式事务；
- 冗余组中的 Leader Replica：协调某组对象更新。

普通对象 fetch 可以选择有效副本，不一定总去 DTX Leader。若非 Leader 副本遇到本地尚不能确定最终状态或可见性的 DTX，客户端会根据服务端结果刷新状态、重试，必要时转向 DTX Leader；不能把 `prepared`、`committable` 和 `committed` 当成同一种状态。

DAOS 数据一致性主要依靠：

```text
epoch/HLC
+ MVCC
+ DTX 状态
+ 冲突检测
+ 必要时向 Leader 重试
```

需要谨慎区分：

- 基础 epoch IO；
- 显式分布式事务；
- DFS 的一致性模式；
- 上层块设备是否限制为单写者。

DAOS 官方明确描述的是分布式事务的可串行化，不能不加前提地把所有普通 IO 都称为线性一致。

在你的项目里，Raft 主要对应 DAOS 管理服务/RDB 元数据一致性，不代表所有 bdev 数据 IO 都先经过同一个 Raft Leader。

---

## 十三、常见错误表达与正确说法

### 错误一

> Leader 发送选举超时事件。

正确：

> Leader 发送 AppendEntries 心跳，Follower 自己的随机选举计时器超时后发起选举。

### 错误二

> Candidate 的日志条目少就一定更旧。

正确：

> 先比较 `lastLogTerm`，只有 term 相同才比较 `lastLogIndex`。

### 错误三

> `prevLogIndex` 就是 Leader 当前最新 index。

正确：

> 刚当选时第一次尝试通常如此；稳定计算公式是 `nextIndex[F]-1`，每个 Follower 可以不同。

### 错误四

> Follower 主动回退到双方一致位置。

正确：

> Leader 回退 `nextIndex` 并重试，Follower 只返回拒绝或成功。

### 错误五

> 日志复制到多数节点后一定可以立即提交。

正确：

> Leader 通过副本数直接推进 commitIndex 时，还要求 `Leader.log[N].term == currentTerm`；旧任期日志通过当前任期日志间接提交。

### 错误六

> `lastApplied` 表示已经持久化到哪里。

正确：

> `lastApplied` 表示状态机执行到哪里；日志持久化发生在更前面的阶段。

### 错误七

> `commitIndex=5` 表示和 Leader 一致到 5。

正确：

> 它表示该节点知道日志 1～5 已安全提交。复制匹配进度由 `matchIndex` 表达。

### 错误八

> 已提交表示全部节点都有。

正确：

> 已提交通常建立在多数派上，落后或宕机节点可以没有这些日志。

### 错误九

> Snapshot 只是告诉 Follower 大家已经一致，可以从下一条开始。

正确：

> Snapshot 是被删除日志产生的完整状态机结果。缺失该状态的 Follower 必须安装后才能接后续日志。

### 错误十

> 只要从 Leader 读就是线性一致。

正确：

> 还要确认它仍是有效 Leader，并等待状态机应用到安全 ReadIndex。

---

## 十四、一面高频题与口述答案

### 14.1 完整讲 Leader 选举

口述答案：

> Follower 在一个选举超时窗口内既没有收到合法 Leader 的 AppendEntries，也没有向合法 Candidate 授票，就增加 currentTerm、转为 Candidate、给自己投票并持久化 currentTerm 和 votedFor，然后并发发送 RequestVote。投票者要求候选者 term 不落后、本任期尚未投给其他人，并且候选者日志至少一样新，日志新旧先比 lastLogTerm，再比 lastLogIndex。候选者获得整个配置的多数票后成为 Leader。瓜分选票时，各 Candidate 在新的随机超时后进入更高任期重选；节点看到更高 term，或者 Candidate 收到同 term 合法 Leader 的 AppendEntries，都会退回 Follower。

### 14.2 prevLogIndex/prevLogTerm 是什么

口述答案：

> Leader 为每个 Follower 维护 nextIndex，发送 AppendEntries 时使用 `prevLogIndex=nextIndex-1`，`prevLogTerm=Leader.log[prevLogIndex].term`。Follower 用这对字段检查双方能否在该位置接上同一日志前缀；不匹配就拒绝，Leader 回退 nextIndex 重试。它们是日志复制锚点，不是 commitIndex，也不一定等于 Leader 当前 term。

### 14.3 冲突日志怎么修复

口述答案：

> Leader 通过 nextIndex 回退寻找双方最后一个匹配位置。Follower 在同 index、不同 term 的第一条冲突日志处删除该条及其后全部后缀，再追加 Leader 的日志；同 index、同 term 的日志保留并跳过。Leader 控制回退和重发，Follower 执行截断与追加。

### 14.4 多数复制是否一定能提交

口述答案：

> 不一定。Leader 直接推进 commitIndex 时，除了多数节点 matchIndex 不小于 N，还要求 `Leader.log[N].term == currentTerm`。旧任期日志即使暂时存在于多数节点，也可能被一个拥有更高 lastLogTerm、但不含该日志的未来 Leader 覆盖。新 Leader 通常追加当前任期 no-op；当 no-op 复制到多数节点并提交后，其之前的旧任期日志随前缀一起间接提交。

### 14.5 commitIndex 和 lastApplied 的区别

口述答案：

> commitIndex 是已经安全决定、永远不会被覆盖的最高日志位置；lastApplied 是状态机已经执行的最高位置。提交可以快于状态机应用，所以 lastApplied 可以小于 commitIndex。此时共识安全没有被破坏，但读取状态机可能得到落后结果，线性读要等 lastApplied 追到 readIndex。

### 14.6 哪些状态必须持久化

口述答案：

> Raft 论文要求 currentTerm、votedFor 和 log 持久化，并在回复相关 RPC 前落盘。currentTerm 保证任期不倒退，votedFor 防止重启后同任期投两票，log 防止已被多数派计数的日志重启后丢失。commitIndex、lastApplied、nextIndex 和 matchIndex 在论文中可重建，但实际系统通常还持久化快照和 applied 位置以高效恢复并避免重复副作用。

### 14.7 网络分区如何避免脑裂写

口述答案：

> 5 节点分成 2+3 时，少数派旧 Leader 可能仍自认为 Leader，也能本地追加日志，但拿不到 3 票，无法提交和返回成功。多数派可以选出更高任期 Leader 并继续提交。网络恢复后旧 Leader 看到更高 term 退回 Follower，少数派未提交后缀由新 Leader 的 AppendEntries 截断覆盖。安全性来自多数派，而不是要求任何时刻只有一个节点自称 Leader。

### 14.8 如何提供线性一致读

口述答案：

> 只读本地 Leader 不够，因为被隔离的旧 Leader 可能不知道自己已失效。ReadIndex 要先确保 Leader 已提交一条当前任期日志，再记录 commitIndex，并携带本次读的唯一 context 与多数派交换心跳，确认当前任期领导权；最后等待 lastApplied 不小于 readIndex 后读取状态机。Lease Read 可以省掉每次多数派确认，但依赖时钟漂移和租约假设。

### 14.9 Follower 落后到日志已压缩怎么办

口述答案：

> 如果 Follower 需要的日志已不在 Leader 保留的日志范围内，Leader 只能通过 InstallSnapshot 传输执行完该日志前缀后的完整状态机。Follower 原子安装快照，把 commitIndex 和 lastApplied 至少推进到 lastIncludedIndex，然后 Leader 从下一条恢复 AppendEntries。

### 14.10 Snapshot 后缀如何处理

口述答案：

> 如果 Follower 本地在 lastIncludedIndex 上存在相同 term，根据日志匹配性质，可以保留该位置之后的后缀，后续再由 AppendEntries 验证；如果该位置 term 不同或不存在，原后缀没有共同历史基础，应丢弃并从快照下一条重新同步。

---

## 十五、自测题

先口述答案，再看下一节。

1. `currentTerm=7` 时，为什么 `log[10].term` 仍然可以是 4？
2. Candidate A 的最后日志是 `(term=5,index=10)`，B 是 `(term=4,index=100)`，谁更新？
3. Leader 的日志 term 序列是 `[1,1,2,3,3]`，`nextIndex[F]=4`，`prevLogIndex/prevLogTerm` 是多少？
4. Follower 在同一 index 存在相同 term 时为什么不重写？
5. Follower 在同一 index term 不同时删除哪部分日志？
6. 5 节点中 Leader、F1、F2 的 `matchIndex >= 8`，是否构成多数？
7. 如果 `currentTerm=6`、`log[8].term=5`，能否仅凭上一题的多数副本直接把 commitIndex 推到 8？
8. 如何让上一题的旧任期日志安全提交？
9. `lastLogIndex=10, commitIndex=7, lastApplied=5`，三段日志分别是什么状态？
10. 为什么旧 Leader 位于少数派时不能提供线性一致读？
11. `currentTerm`、`votedFor` 为什么必须在回复 RequestVote 前持久化？
12. Leader 快照到 1000，Follower 只应用到 100，为什么不能直接发 index 1001？
13. Follower 的 `log[1000].term` 与快照的 `lastIncludedTerm` 相同时，后缀能否保留？
14. 本地创建 Snapshot 与 InstallSnapshot RPC 有什么区别？
15. Ceph PG Primary 和 Raft Leader 为什么不能直接画等号？
16. ReadIndex 为什么要求 Leader 先提交一条当前任期日志，并为本次读等待带唯一 context 的多数派响应？
17. Joint Consensus 中，配置条目是在提交后生效，还是写入本地日志后立即生效？

## 十六、自测题答案

1. `currentTerm` 描述当前领导权；entry term 描述日志出生任期，创建后不变。
2. A 更新，先比较 lastLogTerm。
3. `prevLogIndex=3`，`prevLogTerm=2`。
4. 相同 index 和 term 表示同一日志及其前缀一致，跳过即可。
5. 删除第一条冲突日志及其后全部后缀，再追加 Leader 日志。
6. 构成。Leader 自己计票，3/5 是多数。
7. 不能。index 8 是旧任期日志，不能仅凭当前副本数首次直接提交。
8. 追加并多数复制一条 Term 6 日志，提交它时 index 8 随前缀间接提交。
9. 1～5 已提交已应用；6～7 已提交未应用；8～10 未提交未应用、可能被覆盖。
10. 它无法联系多数派确认自己仍然是 Leader，可能已经有更高任期 Leader 提交了新数据。
11. 防止任期倒退和重启后同任期重复投票。
12. Follower 缺少 101～1000 对状态机造成的修改，而 Leader 已删除这些日志；必须用快照补齐物化状态。
13. 可以保留，但保留不等于提交，后续冲突仍由 AppendEntries 截断。
14. 所有节点都能本地创建快照压缩日志；InstallSnapshot 由 Leader 用来让落后 Follower 追赶。
15. Ceph 是每 PG 一个 Primary，数据操作不经过一个全局 Raft 日志；两者的复制、故障恢复和一致性机制不同。
16. 当前任期日志让 Leader确定自己掌握的安全提交水位；唯一 context 的当前任期多数派响应证明本次读发生时它仍是 Leader，旧 ACK 不能证明这一点。
17. 节点把配置条目写入本地日志后就立即使用日志中的最新配置；Joint 阶段要求新旧配置各自多数，不能等条目提交后才切换 quorum 规则。

---

## 十七、建议学习顺序

### 第一轮：只掌握字段

能脱稿解释：

```text
currentTerm
entry.term
nextIndex
prevLogIndex/prevLogTerm
matchIndex
commitIndex
lastApplied
lastIncludedIndex/lastIncludedTerm
```

### 第二轮：画两条主流程

白板画：

1. Follower 超时到 Leader 当选。
2. 客户端写入到所有状态机应用。

要求每一步标出哪个字段发生变化。

### 第三轮：攻克三个反例

1. 两 Candidate 瓜分选票。
2. 旧任期日志已经在多数节点却仍不能直接提交。
3. 旧 Leader 位于少数派却仍自认为 Leader。

### 第四轮：恢复路径

练习：

- 冲突日志截断；
- 节点宕机重启；
- Follower 落后触发 InstallSnapshot；
- Snapshot 边界相同和不同两种后缀处理。

### 第五轮：项目关联

分别回答：

- DAOS 哪些元数据由 RDB/Raft 维护？
- 普通 DAOS 对象数据是否经过同一个 Raft Leader？
- Ceph PG Primary 与 Raft Leader 的相似点和本质差异？
- 你的控制面为什么需要幂等和重试，即使底层 RDB 已经有 Raft？

完成标准不是“看懂”，而是：

> 关掉文档后，能独立画图、解释字段来源，并回答“为什么不能采用更简单做法”。
