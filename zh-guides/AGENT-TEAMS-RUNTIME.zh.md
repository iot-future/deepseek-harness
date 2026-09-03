# Agent Teams 团队运行时：从需求到可靠协作

> 本文面向希望理解本项目多智能体协作机制的读者。重点不是罗列 API，而是回答：**为什么需要每一个组件、拿掉它会发生什么，以及它们怎样共同把“多个 agent 一起做事”变成可恢复、可追踪、可收尾的运行时。**
>
> 适用范围：当前 `packages/experimental/agent-team` 及其兄弟包 `tool-agent-team`。这是一个私有、实验性的实现，接口和存储形式尚不承诺稳定。

## 0. 先给结论：它不是“多个聊天窗口”，而是一个持久协作领域

一个普通的子 agent 只能回答“父 agent 给我的这件事做完了吗”。但当任务需要多个人并行时，真正的问题会变成：

- 谁属于这支团队，谁有权创建或停止成员？
- 成员不在线时，消息如何不丢；重试时，消息如何不重复？
- 两人同时更新同一项任务时，如何避免后写的人悄悄覆盖前写的人？
- 进程重载、崩溃或插件卸载时，已经接受的工作、邮件和成员如何恢复或收尾？
- 多人共用一个 checkout 时，怎样降低改到同一文件的概率，又不虚假承诺“绝不会冲突”？

当前实现把这些问题收束为一个小型领域运行时：**以 Lead Session 为唯一协调账本，用连续子 agent 执行工作，用 roster、mailbox、任务 DAG 记录协作事实，用投影恢复当前状态，并用一次性等待和受控停机管理活跃运行时。**

```text
                 ┌─────────────────────────────────┐
                 │        Lead Session 日志          │
                 │  团队的唯一持久协调账本（真源）     │
                 └──────────────┬──────────────────┘
                                │ 追加 Team 领域事件
      ┌─────────────────────────┼──────────────────────────┐
      ▼                         ▼                          ▼
  roster / 成员表          mailbox / 邮箱             task board / 任务板
      │                         │                          │
      ▼                         ▼                          ▼
 continuable 子会话       target inbox / 历史        owner + revision + DAG
      │                         │                          │
      └─────────────── TeamService 协调 ──────────────┘
                                │
                        scoped Team tools / Web UI
```

这里最重要的心智模型是：

> **团队不是一个独立数据库对象，而是一个以 Root Session 为聚合根的、可回放的协作领域。**
>
> Root Session 活着并可持久化，团队就有稳定的协调中心；成员运行与否只是这个协调中心周围的暂时状态。

---

## 1. 从一个真实需求开始：并行修改功能，为什么“多开几个 agent”不够

设想用户要求：“实现一个新能力，同时让三位成员分别分析现有实现、修改后端、补测试，最后由 Lead 汇总。”

最朴素的做法是直接创建三个子 agent。它能并行，但很快遇到以下断点：

| 需要回答的问题 | 只有普通子 agent 时的缺口 | 产生的风险 |
|---|---|---|
| 成员是否有稳定名字与职责？ | 子会话 id 可用，但没有团队名册和角色 | 同一人无法稳定寻址；恢复后难以重建职责 |
| 成员能互相通信吗？ | 默认通信是 parent-child，不是 peer-to-peer | worker 只能把结果交回父级，无法直接协调 |
| 离线成员能接收消息吗？ | 消息可能只存在于临时 inbox | 重启或冷恢复后遗失工作指令 |
| 公共任务谁负责？ | 没有共享 owner、依赖、版本 | 两人重复做；陈旧更新覆盖新状态 |
| 如何判断能否收尾？ | 没有可观察的成员、邮件、任务状态 | Lead 可能在结果尚未到齐时提前答复 |
| 卸载或退出时如何停止？ | 各异步操作没有共同的停机边界 | 产生孤儿子会话、悬挂等待或未落盘回执 |

因此 Agent Teams 的目标不是提高“同时运行的数量”，而是补齐一组协作语义：**身份、通信、任务一致性、恢复和生命周期**。

### 1.1 术语澄清：没有显式 `team_create` 或 `team_delete`

这套实现刻意没有单独的 Team 实体创建与删除 API。

- 一个普通运行时 Root 自动是 Lead；`TeamId` 就是该 Root 的 `SessionId`。
- Lead 创建的是具名、可继续执行的直接子成员，而不是创建一个独立 Team 记录。
- 团队历史不删除；任务可以写入 `deleted` tombstone，但 roster 中的名字和成员记录不复用。

这样做省掉了“Team 表、Team 创建事务、Team 删除级联”的另一套生命周期，同时把团队历史固定在本来就必须持久化的会话日志中。代价是团队天然只服务于一个 Root Session，不能被多个独立进程当成共享对象并发操作。

---

## 2. 先看全局结构：各组件分别守住哪一道关

下面这张表是理解运行时的索引。后续各节会逐个展开。

| 组件 | 它解决的需求 | 设计目的与具体作用 | 缺少它会怎样 | 核心心智模型 |
|---|---|---|---|---|
| Profile patch | 团队能力必须显式启用，且不能与旧工具同名冲突 | 挂载 Team 服务与工具，关闭冲突的全局控制工具 | 运行时没有能力，或模型看到同名但语义不同的工具 | 组合是产品的一部分，不是隐式副作用 |
| `TeamService` | 需要统一入口而非让工具直接改状态 | 装配 roster、mailbox、任务板、投影、恢复、停机 | 领域规则散落在工具和 UI 中，无法保证一致 | 服务门面协调领域对象，不替代其职责 |
| 隐式身份与 `TeamRoster` | 需要稳定成员、角色和权限 | 由 Root 推导 Lead；持久化成员的 provisioning/active/failed 边 | 无法可靠寻址、授权与恢复成员 | 身份先于调度；运行状态不等于身份 |
| `TeamJournal` + Projection | 状态需要经得起 reload 和崩溃 | 串行追加/flush Lead 事件，并从事件投影当前状态 | 内存状态消失；并发读改写互相踩踏 | 事件是事实，投影是当前读模型 |
| Invariant companion | 持久事件不能写入非法状态 | 追加前在已提交前缀上试回放候选事件 | 损坏日志延迟到恢复时才暴露 | 在写入口拒绝非法历史，而不是事后修复 |
| `TeamMailbox` | peer 消息需要可恢复且不重复 | `queued` 后投递，目标持久记录后 `delivered` 确认 | 离线消息丢失；重试导致重复执行 | 发送成功先表示“已入账”，不是“已执行” |
| `TeamTaskBoard` | 多人需要共享计划和并发更新保护 | 任务 DAG、owner、revision CAS、写入范围提示 | 重复工作、依赖颠倒、陈旧更新覆盖 | 任务板是协作协议，不是文件锁 |
| `TeamActivity` | 等待协作结果不能轮询 | 监听下一次领域或成员状态变化 | 无效模型轮询、token 和调度浪费 | 等待的是“下一条边”，醒来后重新读真源 |
| `TeamRuntimeLifecycle` | 卸载/退出时需要统一停机语义 | 关闭准入、取消可中断操作、有界等待、drain 子树 | 新旧操作交错、孤儿 worker、停机悬挂 | 先切断入口，再结算已接纳工作 |
| Scoped tools / Remote UI | 模型和浏览器需要操作能力，但不能绕过授权 | 仅向精确 Team 成员作用域注册工具；Remote 返回读视图 | 非成员可误用工具；前端复制领域规则 | 工具和 UI 是领域能力的消费者，不是事实源 |

---

## 3. 第一层基础：为什么选择“Lead Session 日志”做团队账本

### 3.1 需求背景：协作状态必须在成员不在线时仍可解释

一个成员运行时的内存状态无法覆盖以下情况：

```text
成员 A 已接受任务
  → 服务重载
  → 成员 B 要继续依赖它的结果
  → Lead 需要知道 A 是真的创建成功、失败，还是创建到一半
```

如果 roster、任务和消息仅在内存中，进程消失后没有办法区分“从未发生”“发生但还没落盘”“已落盘但尚未投递”。因此 Team 不能把协调状态放在某个长期驻留的 manager 对象里。

### 3.2 设计：四类领域事件写入精确的 Lead Session

`TeamJournal` 将每支 Team 的变更串行化，并在操作对外成功前执行 `flush`。当前领域事件只有四类：

| 事件 | 记录的事实 | 它不是 |
|---|---|---|
| `team/member` | 一个成员的完整生命周期快照 | 当前 `running` / `idle` 状态 |
| `team/task` | 一个任务的完整版本化快照 | 仅给 UI 的临时卡片状态 |
| `team/message/queued` | 消息已成为待投递的持久意图 | 消息一定已被目标模型消费 |
| `team/message/delivered` | 目标 Session 已经持久记录该消息 | 目标工作一定已完成 |

```text
命令（spawn / send / update task）
  │
  ▼
按 Lead SessionId 进入 TeamJournal 的串行队列
  │
  ▼
读取当前 projection → 校验业务规则 → append Team event → flush
  │
  ▼
更新 projection，并通知等待者
```

这里有两个相互配合的机制：

1. **每个 Lead 一条异步事务尾链。** `TeamJournal.transact()` 按 Root id 串行化完整的“读取 → 判断 → 追加”过程；它不是数据库事务，却足以消除同一 Node 进程内的竞态读改写。
2. **事件先 flush，再发布成功。** 例如 `send_message` 返回 `queued` 之前，消息已经在 Lead 日志中；重启后可以从日志找回它。

### 3.3 为什么不是直接保存一个 Team JSON 对象

直接更新 Team JSON 看似简单，却会把三个问题绑死：

- 状态变更历史和当前值混在一起，难以审计“谁先 claim、谁后 release”；
- 不能自然表达“已排队但未确认投递”这类中间事实；
- 进程崩溃在写入之间时，无法靠回放决定下一步恢复动作。

事件日志不是为了“保存很多历史”本身，而是让每个关键边沿都有可判定的持久事实。当前实现甚至让 Team 事件不进入 conversation surface：它们服务协作恢复，不增加模型历史 token。真正要让模型看到的 peer 消息，会以目标 Session 的 `user/message` 进入该成员自己的模型历史。

### 3.4 Projection：把历史压缩为可读的当前状态

`teamProjectionDefinition` 从 Lead Session 的事件流折叠出：

```text
TeamState
├── members        # 持久 roster
├── tasks          # 每个任务最新完整快照
├── messages       # 所有已 queued 的邮件
├── delivered      # 已确认邮件 id
└── nextTaskNumber # 下一个 task-<n>
```

投影的价值在于把“存储格式”和“业务读取”分开：

- 写路径追加事件；
- 读路径只看 `TeamState`；
- `queued - delivered` 自然就是等待恢复的 mailbox；
- `TeamMemberView`、`TeamTaskView` 再在持久快照上叠加 live status、owner 名称、ready 和路径重叠警告。

**为什么不能缺少投影？** 如果每个工具各自扫描、解释日志，成员、消息和任务的规则会分叉；如果把可变状态藏在服务字段中，reload 后又失去恢复依据。投影让“一个日志，一套解释规则”成为强约束。

### 3.5 Invariant companion：在写入时防止制造不可回放历史

Projection 能识别损坏事件，但仅在恢复时发现错误还不够。`invariant.ts` 在每一个候选 Team 事件发布前，复制已提交 projection 并尝试应用候选事件；若出现非法状态则拒绝写入。

它保护的例子包括：

- 新成员不是从 `provisioning` 开始；
- 成员改了不可变的 name/provider/context；
- task revision 不连续；
- task dependency 成环；
- message 未 queued 就标记 delivered；
- 同一个 message 被 delivered 两次。

其心智模型是：

> **日志不是“出了问题再修”的容器；日志一旦成为恢复依据，就必须在写入边界保证它始终可回放。**

---

## 4. 第二层基础：成员身份、权限与创建恢复

### 4.1 需求背景：运行着的 agent 不等于团队成员

系统中可能同时存在普通 Root、普通 fork、one-shot 子 agent、continuable 子 agent。若仅凭“它是一个子 agent”授予 Team 权限，会误把无关 worker 纳入协作领域。

`TeamRoster.tryMembership()` 使用**精确 live `Agent` 身份 + 父子关系 + 持久 roster**判断身份：

```text
普通 Root Agent              → 隐式 Lead，TeamId = Root SessionId
roster 中的 continuable child → teammate
roster 外的普通 subagent      → 不是 Team 成员
父级已释放的 continuable child → 不会被错误重解释为新 Lead
```

这不是单纯的权限检查，而是领域边界：只有属于同一 Root Team 的 agent 才能读写同一份 roster、任务和 mailbox。

### 4.2 为什么 Team 采用隐式 Root，而不是显式 Team 记录

隐式 Root 的设计目的有三点：

1. **消除第二个生命周期根。** Session 本来就有持久化、恢复、父子关联和清理能力；Team 直接附着于它，无须再维护独立 Team 创建与销毁事务。
2. **让授权可精确定位。** `TeamId` 与 Lead Session 一一对应，任何领域操作都可追溯到一个 live Lead。
3. **避免 fork 污染。** 普通 fork 是新的 Root；它继承的祖先 Team 事件因 `teamId` 不匹配而不会进入新 Root 的 projection。

代价也很明确：该运行时不是跨 Session 共享组织系统，不能让不同 Root 或不同进程共同操作同一 Team。

### 4.3 Roster 状态机：先记录可恢复意图，再启动副作用

创建 teammate 看似只是调用 `startContinuable()`，但这会在“写 roster”和“子 Session 接收第一条消息”之间产生崩溃窗口。当前流程刻意拆成两阶段：

```text
Lead 调用 spawn_teammate
  │
  ├─ 1. 检查 Lead 权限、名字、成员上限
  ├─ 2. append + flush member(provisioning)
  ├─ 3. 为预留 childId 启动 fresh 或 fork continuable child
  ├─ 4. 确认 child 的初始 prompt 已被目标 Session 持久接受
  └─ 5. append + flush member(active)

任何创建失败
  └─ append + flush member(failed)，名字仍保留
```

持久 phase 只有三个：

| phase | 表示的事实 | 与 live status 的区别 |
|---|---|---|
| `provisioning` | 已经声明要创建，尚未证明 child 可恢复 | 不等于 child 正在运行 |
| `active` | child 的 lineage、provider、初始消息均已被持久证明 | child 此刻仍可能 `inactive` |
| `failed` | 创建或恢复已得到明确失败结论 | 不是“稍后可重试同名创建” |

这一设计回答了一个容易被忽略的问题：**为什么失败的成员也占用名字和名额？**

因为名字既是模型和 UI 的寻址键，也是历史解释的一部分。若失败后复用 `reviewer`，历史邮件、任务 owner 和新成员会混为一谈。保留失败记录牺牲一点容量，换来稳定寻址和可审计性。

### 4.4 恢复：不是盲目重试，而是“持久证据对账”

当 Lead 重启时，`TeamRoster.reconcileProvisioning()` 检查每个悬而未决的 `provisioning` 记录。它不会重新发送原始 prompt，而是检查 child 的独立持久 Session 是否同时满足：

- `parentSession` 指向当前 Lead；
- 子会话 descriptor 表明它是同一 provider 的 `continuable` child；
- 初始用户消息已经出现在 pending inbox 或历史中。

全部满足才补写 `active`；否则补写 `failed`。这是一种**证据驱动的补偿**：恢复不猜测，而是根据另一个持久系统留下的事实来结算中间态。

没有这一组件时，系统无法区分“provider 还没来得及创建”“child 已创建但 Lead 未写 active”“child 根本不属于该 Team”，最终要么产生孤儿 worker，要么错误地把失败成员标记为可用。

---

## 5. 可靠通信：持久 mailbox 如何做到“可重试而不重复”

### 5.1 需求背景：消息成功不是一个瞬间

对 peer 消息来说，至少存在三个不同的成功点：

```text
A. 发送方接受了调用
B. 协调端已经持久记录“应当投递”
C. 目标 Session 已经持久记录该消息
```

若把 A 当成成功，崩溃后会丢消息；若把 B 当成“已送达”，用户会误以为目标一定看到了；若反复重试而没有 C 的去重，目标可能重复执行同一工作。

### 5.2 设计：先入账，再投递，目标持久化后确认

`TeamMailbox` 使用双事件协议：

```text
sender
  │ send_message / followup_task
  ▼
Lead 日志 append + flush: team/message/queued
  │                         │
  │                         └── 这一步成功后，消息已具备恢复依据
  ▼
按 target 串行尝试投递
  │
  ├─ quiet  → 对 live target `inject()`；不唤醒 idle agent
  └─ wakeup → 对 target `followup()`；必要时 cold resume 并启动下一 turn
  │
  ▼
target Session 持久记录带 TeamMessageSource 的 user/message
  │
  ▼
Lead 日志 append + flush: team/message/delivered
```

返回值因此有明确含义：

| 状态 | 含义 | 发送方应做什么 |
|---|---|---|
| `accepted` | 目标已持久接收，Lead 已确认 delivered | 继续后续流程 |
| `queued` | 消息已经安全写入协调账本，但当前未完成投递 | **不要重发**；恢复或目标启动时会重试 |

这正是 mailbox 最重要的语义：**`queued` 不是失败，而是持久化的待办。**

### 5.3 两种投递模式不是优化，而是两种业务意图

| 工具 | `delivery` | 需求语义 | 行为 |
|---|---|---|---|
| `send_message` | `quiet` | “留一条信息，等对方自然继续时再看” | live target 注入消息但不唤醒；inactive target 保持 queued |
| `followup_task` | `wakeup` | “这是要继续执行的任务” | 进入目标的下一次 FIFO turn，必要时恢复目标 |

若只有一种“发送”动作，要么所有通知都无谓唤醒 agent，增加 token 与调度成本；要么真正的工作移交无法可靠启动。区分 quiet 与 wakeup，是把业务意图放入协议而不是让调用方猜测。

### 5.4 去重和顺序：不是 exactly-once，而是可靠的单进程模型

每条 peer 消息都有稳定 `TeamMessageId`，并以 `TeamMessageSource` 写入目标 Session。投递前，邮箱会折叠检查目标的 live inbox 和持久历史；若已存在同一个 message id，只补写 Lead 侧 acknowledgement，不再注入一次。

此外，mailbox 为每个 target 保留独立 dispatch tail，保证同一目标按 Lead 日志的 queued 顺序进入投递队列；不同目标仍可并行。

它提供的保证应该准确理解为：

- 同一进程内：**持久 queued、按 target 有序重试、目标侧 id 去重**；
- 跨进程并发操作同一 Team：**不保证 exactly-once，也不支持共识**。

若把这层误解为分布式消息队列的 exactly-once，部署到多个 harness 进程就会出现重复或竞争。这里的边界是刻意的：先把单进程恢复做扎实，而不是伪装成分布式系统。

---

## 6. 共享任务板：用版本化协议协调，而不假装提供文件锁

### 6.1 需求背景：消息能传递，仍无法解决“谁该做什么”

成员之间聊天并不能天然防止以下情况：

- 两人都以为自己负责同一个任务；
- B 依赖 A 的结果却先开始；
- A 刚更新任务，B 用旧界面数据提交修改；
- 任务 owner 被中断后，系统擅自把任务转给别人，造成真实工作和状态脱节。

`TeamTaskBoard` 将任务从“文本约定”提升为带状态机的共享协议。

### 6.2 任务快照：状态、owner、依赖和 revision 必须一起保存

```text
TeamTaskSnapshot
├── id: task-<n>                 # Team 内单调递增
├── revision                     # 每次变更 +1
├── subject / description
├── status                       # pending / in_progress / completed / deleted
├── ownerId?                     # in_progress 的责任人
├── blockedBy[]                  # 依赖任务
└── writeScopes[]                # 计划修改的相对路径前缀
```

全量快照而非局部 patch 的好处是：回放单个 task 时，不必解释一长串“标题改了、owner 换了、依赖删了”的增量事件；每个 revision 就是一份可验证、可展示的完整任务事实。

### 6.3 CAS：把“我看到的旧任务”变成明确的并发前提

所有更新都必须携带 `expectedRevision`：

```text
读取 task-7，revision = 3
  │
  ├─ 成员 A: expectedRevision=3，claim 成功 → revision=4
  └─ 成员 B: expectedRevision=3，claim 失败 → TEAM_TASK_STALE_REVISION
```

这比“最后写入者覆盖”更适合 agent 协作：失败方知道自己的上下文陈旧，必须重新读取再决定。浏览器 Remote 层还会将 revision 冲突单独映射为 `team-task-conflict`，而不是混入一般业务拒绝。

**没有 CAS 会怎样？** 两个 agent 都可能在各自上下文里确认“任务仍未领取”，随后都写出 `in_progress`，最后一个写入者静默抹掉第一个 owner。问题不是并发本身，而是旧认知被当成新事实提交。

### 6.4 DAG 与授权：任务板管理协调次序，不管理工作结果

任务可以通过 `blockedBy` 建立依赖，系统要求 blocker 存在、未删除且整张图无环。只有所有 blocker 已完成的 pending task 才可 claim。

权限规则同样很窄：

| 动作 | 谁能执行 | 目的 |
|---|---|---|
| create / read | 任一 Team 成员 | 让协作信息对等可见 |
| claim | 任一成员，且任务 ready、无其他 owner | 自主领取可并行工作 |
| edit / release / complete / reopen / delete | 当前 owner 或 Lead | 防止旁观者篡改他人工作 |
| reassign | 仅 Lead | 保持委派权集中 |

这里有意没有“成员失活自动释放 owner”。`inactive`、interrupt、进程退出都不说明成员没有实际完成工作；自动释放会制造双 owner。系统选择保守：让 Lead 显式 reassign 或 release。

### 6.5 `writeScopes` 为什么只是警告，不是锁

所有成员共享同一个工作目录。任务可记录 `src/api`、`tests/foo.spec.ts` 等预期写入范围，视图会在两个 in-progress task 路径前缀重叠时给出 warning。

但它绝不阻止写入，因为：

- 路径前缀无法证明两个改动在语义上冲突或不冲突；
- Bash、formatter、codegen、外部工具可以绕过文件 API 的版本保护；
- 任务 owner 可能崩溃，锁若不释放会永久阻塞；
- “看起来安全”的伪锁比清晰 warning 更危险。

因此正确心智模型是：

> **任务 owner 是责任归属；`writeScopes` 是协调提示；真正的并发控制仍由文件版本检查、重新读取/rebase、Lead 最终 review 和测试共同完成。**

---

## 7. 等待与可观察性：避免把协作变成轮询风暴

### 7.1 需求背景：Lead 怎样在不浪费模型轮次的前提下等结果

没有等待机制时，Lead 只能不断调用 `list_agents` 和 `team_task_list`。这会消耗 token、制造无意义的工具调用，而且仍可能错过状态变化的时机。

`TeamActivity` 提供一次性 waiter：等待调用注册之后发生的下一条 Team 领域边或成员 live-status 变化。

```text
wait_agent 开始
  │
  ├─ 邮件 queued / delivered
  ├─ roster 变更
  ├─ task 变更
  ├─ 成员 running / idle / inactive 状态变化
  ├─ timeout
  └─ runtime dispose
       ↓
返回后重新 list / get 真源状态
```

它不传输“变化详情”，只告诉调用方是否被变化唤醒或超时。这能避免把一个可能已过期的事件快照当成最终状态。

### 7.2 为什么 `wait_agent` 不会唤醒成员

等待是观察行为，不是调度行为。若 `wait_agent` 顺便唤醒 inactive 成员，调用方就无法表达“我只想等，不想启动更多工作”。所以工具层会先检查是否存在其他 `running` 或 `provisioning` 成员；没有则立即返回 `noProgress`，要求先用 `followup_task` 明确唤醒所需成员。

这体现了一个通用原则：**观察与命令分离。** 一个“wait” API 不应暗中改变系统状态。

---

## 8. 生命周期：为什么停机必须是一个正式协议

### 8.1 需求背景：异步操作和插件卸载可以交错发生

团队运行时同时可能存在：成员创建、消息投递、目标确认、等待者、活跃子会话。若卸载时直接移除服务，会出现：

- 已接受的 spawn 在服务消失后继续创建 child；
- queued 邮件投递到一半，无人确认 delivered；
- wait 永远不返回；
- Team child 与非 Team child 被一并错误停止。

### 8.2 设计：一个共同 cancellation fact + 有界结算

`TeamRuntimeLifecycle` 用一个 `AbortController` 代表整个 Team 服务的准入状态。销毁路径严格按顺序执行：

```text
1. close()：关闭新 spawn / send 的准入，并取消可中断操作
2. activity.close()：唤醒全部 waiters
3. settle()：等待已经接纳的成员创建与 mailbox dispatch
4. drain：仅释放 roster 记录的、仍 live 的 continuable direct child
5. 汇总非正常错误；超出 disposalTimeoutMs 明确报错
6. 解除 projection 注册
```

这里最细微也最重要的区分是：**只停止 Team 所拥有的孩子。** 通过 roster 和精确 parent 关系筛选，Lead 的普通 non-Team continuable child 不会被误杀。

### 8.3 为什么要“先关闭准入，再等待”

如果先等待再关闭，新操作可以不断进入，停机永远无法收敛；如果只关闭不等待，已接纳操作会被无声丢弃。这个顺序是所有并发服务都可复用的模式：

> **Admission cutoff → cancel / drain admitted work → bounded settle → release resources。**

它既给已经确认受理的操作一个确定结局，也让关闭有上界。

---

## 9. 工具与 UI：它们负责呈现和命令，不能成为第二个事实源

### 9.1 Scoped tools：只在正确的 agent 上出现正确的能力

`tool-agent-team` 不直接拥有 roster、mailbox 或 task 状态。它做三件事：

1. 为每个精确的 Team 成员 `agent.ctx` 注册系统提示词和工具；
2. 将模型输入转换为 `TeamService` 请求；
3. 用完整 schema 约束并紧凑渲染模型可见结果。

工具面如下：

| 类别 | 工具 | 领域调用 |
|---|---|---|
| 成员 | `spawn_teammate`、`list_agents`、`interrupt_agent` | roster |
| 通信 | `send_message`、`followup_task` | mailbox |
| 协作等待 | `wait_agent` | activity |
| 任务 | `team_task_create`、`team_task_list`、`team_task_get`、`team_task_update` | task board |

工具按 agent scope 注册，因此普通子 agent 不会意外获得 peer 权限；成员销毁时注册也会反向解除。工具层还向模型明确共享 checkout、write scope 仅提示、`queued` 不可重发和 Lead 必须等待成员这些协作规则。

### 9.2 Profile patch：让“可选能力”真的可选

团队能力通过实验 profile patch 显式加入，而不是默认进入所有会话。patch 会：

- 插入 `agent-team` 和 `tool-agent-team`；
- 配置成员、任务、消息、停机上限；
- 关闭与 Team 同名的旧全局 continuable-child 控制工具；
- 保留 one-shot `subagent` / `subagent_fork` 的独立语义。

如果不这样组合，模型可能同时看到名称重叠、权限模型不同的工具，或简单任务也被强行带入团队协作的延迟和 token 成本。

### 9.3 Web UI：读投影、发领域命令、显示冲突

浏览器侧通过生成的 Remote API 请求：

- `agentTeams.view` 读取成员和非删除任务的当前视图；
- `agentTeams.createTask` / `updateTask` 写任务；
- 打开 teammate 时沿 session 的父地址找到 Lead Session。

Remote 层把可预期的 Team 拒绝编码成业务结果，而把真正的 transport 异常留给 RPC 失败处理。这让 UI 能区分“revision 过期，请刷新”与“网络调用失败”。

无论工具还是 UI，都不应自己缓存并判断权威状态；它们必须在需要时从 `TeamService` 的投影重读。

---

## 10. 把所有组件串起来：一次协作任务的完整旅程

以下用“Lead 让 `reviewer` 分析改动，并等待结果后完成任务”为例：

```text
[0] 启动装配
profile patch
  → TeamService 注册 projection、恢复监听、生命周期 effect
  → scoped 工具安装到现有/后续 Team 成员

[1] 创建成员
Lead: spawn_teammate(reviewer)
  → roster: member(provisioning) 已 flush
  → subagent provider 启动 continuable child
  → child 初始 prompt 已持久接收
  → roster: member(active) 已 flush

[2] 建立工作协议
Lead / teammate: team_task_create(... write_scopes, blocked_by)
  → task-1 revision=1 写入 Lead 日志
reviewer: team_task_update(task-1, expected_revision=1, action=claim)
  → task-1 revision=2, in_progress, owner=reviewer

[3] 分发与执行
Lead: followup_task(reviewer, "请审查 …")
  → mailbox: message/queued 已 flush
  → child 下一 turn 接收并持久化 user/message(source=team-message)
  → mailbox: message/delivered 已 flush
  → reviewer 执行其 agent-loop

[4] 协作进度
reviewer: send_message(lead, "发现 …")
  → 同一套 queued → target receipt → delivered 协议
reviewer: complete task-1（携带当前 revision）
  → task 新 revision 已 flush

[5] Lead 等待与汇总
Lead: wait_agent()
  → 被消息、task 或 status 边唤醒
  → list_agents / team_task_get 重新读取投影
  → 汇总结果并回复用户

[6] 运行时结束
service dispose
  → 关闭准入 → 结算已接纳操作 → drain roster 子树 → 释放 waiters
```

每一步中，谁负责什么非常清晰：

| 阶段 | 事实源 | 副作用执行者 | 完成判据 |
|---|---|---|---|
| 创建成员 | Lead 日志 roster | subagent provider | child 初始 prompt 已持久化 |
| 发消息 | Lead 日志 mailbox | target inbox / followup | target Session 已持久化消息 id |
| 改任务 | Lead 日志 task snapshot | 无额外副作用 | CAS revision 成功 flush |
| 运行工作 | target 子 Session | agent-loop | 由成员自己的 turn 生命周期决定 |
| 等待 | TeamActivity | 无 | 观察到未来变化或超时 |
| 停机 | Lifecycle | subagent drain | 已接纳操作有界结算 |

这张表揭示了一个关键边界：**Team 管理协作事实与调度，不替成员判断业务工作是否正确。** 成员的实际模型推理、文件修改和测试仍属于其自己的 Session / agent-loop；Lead 负责最终集成与验证。

---

## 11. 关键约束与有意不做的事

理解“不做什么”与理解实现同样重要。

| 边界 | 当前选择 | 原因 |
|---|---|---|
| 团队拓扑 | 只有 Lead 的扁平直接 teammate | 避免嵌套授权、递归恢复与收尾复杂度 |
| 进程模型 | 单进程 | 没有分布式锁、跨进程事务或共识 |
| 工作区 | 共享 cwd / checkout | 保持既有 subagent same-world 契约，不隐式引入 branch/merge 策略 |
| 文件并发 | write scope 警告，不是锁 | 不能为 Bash、formatter、外部写入给出虚假互斥保证 |
| 成员名称 | 不可变、不可复用 | 让历史消息、任务 owner 和 UI 寻址稳定 |
| 任务 owner | 不自动释放 | idle / interrupt / 崩溃不等于实际工作不存在 |
| 团队删除 | 不支持删除历史 | 事件回放和审计需要稳定事实；只对 task 使用 tombstone |
| 默认启用 | 不启用 | 避免简单任务承担团队调度的 token 和延迟成本 |

这些不是“功能未完成”的简单列表，而是当前一致性模型的边界。若将来要支持跨进程成员、worktree 隔离或自动 owner 释放，必须同时重做身份、恢复、授权、冲突处理和生命周期协议，不能只加一个工具。

---

## 12. 可迁移的架构经验：如何把这套思想用到其他协作系统

### 12.1 模式一：选择天然存在的聚合根，而不是重复造根

这里使用 Root Session 作为 Team 的身份、日志和授权锚点。迁移到其他系统时，先问：是否已经有一个天然具备生命周期、持久化和权限上下文的对象？

- 面向一次客户请求的协作：请求 / workflow execution 可做聚合根；
- 面向一项长期业务的协作：case / order / project 可做聚合根；
- 面向一个会话内的 agent 协作：session 是合适的根。

前提是根能稳定存在并承载追加事实；若不同进程需要共同修改，则要换成具备并发控制的外部存储，而不是复制本实现的内存 serializer。

### 12.2 模式二：把“发出意图”和“对方接受”分成两个持久事实

mailbox 的 `queued → delivered` 是可靠协作最可复用的模式：

```text
持久记录意图（outbox）
  → 尝试副作用
  → 接收端持久去重键
  → 持久确认完成
```

它可迁移到邮件、webhook、订单履约、异步审批、文件处理等系统。核心不是某个消息队列，而是不要把“调用已返回”误当成“接收端已持久接受”。

### 12.3 模式三：单写者串行化 + CAS，各自处理不同层级的并发

两种并发防护不能相互替代：

- Journal serializer 保护同一进程里完整读校验写操作不交错；
- Task revision CAS 保护调用方持有的旧视图不覆盖新状态。

迁移时可以把前者换成数据库事务、actor mailbox 或分区顺序队列，把后者换成 row version / ETag / compare-and-swap。无论技术形式如何，仍要同时解决“服务端并发”和“客户端陈旧认知”。

### 12.4 模式四：持久 phase 与实时 status 分离

roster 的 `active` 是已被持久证明的生命周期结论；`running`、`idle`、`inactive` 是此刻运行时观察。混用两者会导致重启后把“暂时不在内存”误判成失败。

任何具备恢复需求的系统都应明确区分：

- **durable state**：重启后仍成立的业务事实；
- **live observation**：当前进程观察到的瞬时状态。

### 12.5 模式五：把权限绑定到精确运行时主体，而非可伪造字符串

Team API 接收 exact live `Agent` 并通过 roster 推导其 role；工具只在该 agent 的 scope 注册。相比“传入 memberName，然后判断名字”，这减少了伪造、过期和跨团队误用的空间。

迁移到服务端时，对应做法是把授权主体绑定到可信 session / credential / actor context，而不是让请求体里的 `ownerId` 成为权限来源。

### 12.6 模式六：区分提示性协调与强制性约束

`writeScopes` 只报告风险，不承担锁的职责。把不能完全执行的策略声明成“建议 + warning”，比宣称它是锁更诚实、更安全。

迁移时可用同样原则判断：

- 能被所有入口统一强制执行，才适合作为权限或锁；
- 只能覆盖一部分入口，就应做诊断、告警或人工 review 信号。

### 12.7 模式七：停机是一等协议，不是析构函数里的顺手清理

生命周期的“关闭准入 → 取消 → 有界等待 → 精确释放”可复用在任何具备后台任务、连接、订阅和 worker 的模块中。关键是先定义：

1. 哪些操作在关闭前已被接纳？
2. 哪些可以取消？
3. 哪些必须结算或记录失败？
4. 超时后的可见后果是什么？
5. 哪些资源属于本组件，哪些不能误释放？

### 12.8 模式八：模型可见信息与内部协调记录分层

Team 任务和 roster 事件不会自动进入模型历史；真正要给某个成员看的内容才作为带 source 的 `user/message` 写到其 Session。这个边界能控制 token 成本，也让模型上下文保留可重建性。

在其他 agent 系统中同样应区分：

- 审计 / 协调元数据：供运行时恢复、UI 和运维使用；
- 模型输入：必须明确、可追溯、最小化地进入上下文。

---

## 13. 若要在另一个系统复用，推荐按这个顺序落地

不要从“先写一个 `spawnWorker` API”开始。更稳妥的实施顺序是：

1. **确定协作聚合根和边界。** 一支 Team 对应什么？哪些 agent 能属于它？是否单进程？
2. **定义持久事实。** 最小事件集合通常包括成员生命周期、任务快照、发件意图、接收确认。
3. **实现可验证投影。** 从空状态回放所有事件，确保非法流被拒绝或显式失败。
4. **为写路径设计原子性。** 选择每聚合根的 serializer、数据库事务或 actor；所有读校验写都必须在同一边界内。
5. **实现成员 provisioning / recovery。** 先记录意图，后触发外部副作用，再用持久证据结算中间态。
6. **实现 mailbox 的接收端去重。** 只靠发送端重试不够，必须让目标保存稳定消息 id。
7. **实现任务 CAS 和依赖校验。** 将业务并发冲突变成显式、可恢复的响应。
8. **把模型工具和 UI 放在最外层。** 它们只转译请求、展示投影，不持有事实源。
9. **最后设计停机和容量上限。** 成员数、pending 消息、任务数、消息尺寸、关闭超时都应是经过校验的部署配置。
10. **用崩溃窗口测试验证。** 特别覆盖“意图已落盘、副作用未完成”“目标已接收、确认未写”“恢复与创建并发”“dispose 与投递并发”。

最值得避免的反模式是：先做内存数组和几个 REST/工具接口，等需要恢复时再补日志。届时身份、消息语义和任务更新往往已散落在各层，几乎只能重写。

---

## 14. 源码阅读地图

| 想回答的问题 | 首选文件 |
|---|---|
| Team 服务怎样装配、恢复和停机？ | `packages/experimental/agent-team/src/index.ts` |
| 谁是 Lead / teammate，创建如何恢复？ | `packages/experimental/agent-team/src/roster.ts` |
| 邮件如何排队、投递、确认、去重？ | `packages/experimental/agent-team/src/mailbox.ts` |
| 任务 CAS、授权、DAG 怎样工作？ | `packages/experimental/agent-team/src/task-board.ts` |
| 同一 Team 的写操作如何串行？ | `packages/experimental/agent-team/src/journal.ts` |
| 事件如何解析、回放并校验状态机？ | `packages/experimental/agent-team/src/projection.ts` |
| 如何在 append 前阻止非法 Team 事件？ | `packages/experimental/agent-team/src/invariant.ts` |
| 等待与停机如何实现？ | `packages/experimental/agent-team/src/activity.ts`、`lifecycle.ts` |
| 模型能看到哪些 Team 工具和策略？ | `packages/experimental/tool-agent-team/src/index.ts` |
| 如何在 profile 中显式启用并解决工具冲突？ | `packages/experimental/agent-team-profile/cordis.patch.yml` |
| 浏览器如何读取和修改 Team 视图？ | `packages/experimental/client-ui-agent-team/src/client/mount.ts` |
| 当前持久类型与公开服务接口是什么？ | `docs/subsystems/agent-team.zh.md` |
| 为什么做出这些边界选择？ | `.agents/notes/implemented/feature/2026-08-05-agent-teams.zh.md` |

---

## 15. 最终总结

Agent Teams 的价值不在于“多开几个 agent”，而在于它为同一会话内的多 agent 协作建立了完整的运行时秩序：

```text
Root Session 提供身份和持久账本
  + roster 把 agent 变成可授权、可恢复的成员
  + mailbox 把 peer 消息变成可重试、可去重的交付协议
  + task board 把分工变成带版本和依赖的协作状态机
  + projection / invariant 让历史始终可解释、可恢复
  + activity / lifecycle 让等待与退出可控
  + scoped tools / UI 将能力安全地交给模型和用户
  = 一个可恢复的单进程 Agent Team 运行时
```

把它带到其他系统时，最应该保留的不是 API 名称，而是这条设计顺序：**先选真源，再定义可持久事实；先为崩溃窗口设计确认，再触发副作用；先明确一致性边界，再对外承诺能力。**
