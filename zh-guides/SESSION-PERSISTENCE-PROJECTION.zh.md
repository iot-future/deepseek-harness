# 会话持久化与投影运行时：从可恢复事实到一致读模型

> 本文从一个实际问题出发：当 agent 正在流式输出、调用工具或修改会话状态时，进程重启、浏览器重连、插件热重载、历史查询和 UI 展示怎样仍然得到同一份可解释的结果？
>
> 它不是 API 清单，而是围绕“需求为什么存在、组件各自守住什么、拿掉会发生什么、它们如何协作”展开的技术导读。范围覆盖 `core/session`、`session-persistence`、`session-persistence-jsonl`、`session-projection`、`session-projection-cache` 及其准备/查询协作路径。

---

## 0. 先给结论：不要持久化“当前对象”，要持久化“已经发生的事实”

一个会话里会不断出现两类信息：

- **事实**：用户消息已被接受、模型产生了分片、工具调用开始或结束、计划被改为开启、待办被整体改写。
- **观点**：当前对话标题、待办列表、token 用量、模型选择、页面上的统计卡片。

本项目把两类信息严格分开：

```text
                 已提交的 SessionEvent 日志
             （唯一权威、仅追加、连续 seq）
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
  模型消息 history        持久化 JSONL         领域投影状态
  surface / messages       或其他 provider     title / todo / stats …
          │                   │                   │
          │                   │                   ▼
          │                   │            客户端全量 view
          │                   │                   │
          └───────────────────┴──────────┬────────┘
                                          ▼
                              可丢弃的投影检查点缓存
                         （缩短回放，不取代、不修改日志）
```

这里的核心不是“多存一份数据”，而是建立一个单向依赖：

> **事实可以生成观点；观点绝不能反过来篡改事实。**

因此：

1. 会话重启时，系统从日志重新得到模型历史和领域状态；
2. 投影缓存损坏或过期时，系统丢弃它并从日志重算，而不是信任它；
3. UI 不必复制每个领域的事件解释逻辑，只接收 host 算好的当前值；
4. 持久化、投影、遥测、fork、回放都读取同一条事件流，不再维护多份“当前会话”。

这是一套事件溯源（Event Sourcing）与读模型（CQRS Projection）的运行时实现；但真正重要的不是名称，而是它消除了“状态 A 与状态 B 要持续同步”的问题类别。

---

## 1. 从一次中断的 agent 任务开始

假设 agent 正在执行下面的轮次：

```text
turn/start
  → user/message
  → step/start
  → assistant/chunk × N
  → tool/call
  → 进程突然退出
```

此时系统至少要回答六个问题：

| 问题 | 如果答案不确定，会造成什么后果 |
|---|---|
| 用户和模型此前看到了什么？ | 恢复后上下文丢失，模型可能重复或误解工作。 |
| 已经接受的分片和工具调用能否审计？ | UI 回放不完整，调用关联与时间线失真。 |
| 这个未结束的轮次应被当作成功、失败还是不存在？ | 直接截断会静默丢掉真实工作；继续执行又可能重复副作用。 |
| 当前待办、标题、计划状态是什么？ | 每个页面或插件各自扫描日志，逻辑会逐渐分叉。 |
| 浏览器恢复连接后如何得到一个一致的基线？ | 某些卡片比另一些卡片新，随后推送可能倒退页面状态。 |
| 读性能怎样随历史增长而保持可接受？ | 每次冷启动、列表或详情页都要从头重放全部历史。 |

如果把答案放进一个可变的 `Session` 对象、几个 UI store 或零散的 JSON 字段里，进程退出后这些答案会一起消失或互相矛盾。项目的方案是按依赖关系逐层补齐能力：

```text
先有不可变事实
  → 再让事实可持久化
  → 再让持久事实能安全恢复
  → 再从事实计算领域读模型
  → 最后把读模型检查点化以加速读取
```

顺序不能反过来。没有权威日志，投影只能缓存不可靠的内存状态；没有恢复语义，文件只是“写出去的字节”，不是可继续执行的会话。

---

## 2. 组件全景：每一层各自解决什么问题

“缺少组件就不能正常运行”需要准确理解：`Session` 与持久化是可恢复会话的基础；投影与缓存是按需装配的能力 seam。没有投影，最小 agent loop 仍能运行；但任何依赖投影状态的 host/UI 功能必须明确失败，不能悄悄给一个猜测的默认值。

| 层次 / 组件 | 产生它的需求 | 设计目的与具体作用 | 缺少它的直接后果 | 核心心智模型 |
|---|---|---|---|---|
| `Session` + `SessionStore` | 必须先定义什么是会话的权威历史 | 接受、验证、冻结并顺序编号 `SessionEvent`；发布会话生命周期和事件流 | 事实没有唯一写入口；内存状态、通知和持久化输入会分叉 | 日志不是通知副本，日志就是会话事实 |
| `SessionHeader` | 有些信息决定存储身份和谱系，却不是对话历史 | 把格式版本、cwd、父会话、种子边界等放在日志外 | 把不可回放元数据误当消息事件，fork/replay 会带入不应继承的状态 | 对话事实与存储身份是两类数据 |
| `SessionPersistence` seam | 会话要跨进程存在，且后端必须可替换 | 定义 append、load、inspect、prepare、suffix read 等持久化语义 | loop 被具体文件格式绑定；其他后端要复制恢复逻辑 | 服务声明“必须保证什么”，后端决定“怎样落盘” |
| `PersistenceCoordinator` + write-behind | 高频事件、HMR、flush、退役与崩溃修复不能散在每个后端 | 集中串行化、批处理、接管、修复、退役和失败处理 | provider 很容易在相同的竞态上各自出错 | 生命周期编排是共享能力，不是文件格式细节 |
| JSONL provider | 需要一个实际的可审计、append-only 存储实现 | 每会话一份 header + 事件流，fsync、原子首写、撕裂尾修复 | 抽象接口没有可运行的默认实现 | 物理编码可优化，逻辑事件必须保真 |
| `SessionPreparation` | 历史查询、恢复和发布不能对同一冷日志重复构造不同对象 | 保留精确的未发布 Session，供检查、设置、发布或回滚 | 查询与恢复重复解析；设置与发布之间会换成另一份 Session | 先准备准确对象，再原子地公开它 |
| `SessionProjectionRegistry` | 每个领域都需要“当前状态”，但不能各自订阅/扫描日志 | 用一个驱动器运行领域纯 fold，维护水位和一致快照 | todo、goal、title 等会在 host/UI 重复实现状态机 | 框架拥有驱动，领域只拥有纯计算 |
| `ProjectionDefinition` | 领域状态要可重放、可校验、可缓存、可对外裁剪 | 声明 `init`、同步 `apply`、schema、`stateVersion` 和可选 `view` | 领域变成不透明 getter，无法做增量推进、缓存或冷恢复 | 输入是事件，输出是确定状态 |
| `SessionProjectionCache` | 冷会话列表和恢复不应每次完整回放所有事件 | 持久化 `(key, ver, seq, val)` 检查点，回放后缀 | 系统仍正确，但冷读成本线性随日志增长 | 缓存只能落后，不能领先或冒充事实 |
| `SessionQuery` observation | 读历史与恢复要复用同一个冷态切面 | 借用 live 或 prepared 的精确不可变观察，并可带投影基线 | 读者各自读取不同 revision，重复 I/O 和解析 | 一次观察是一份可租借的、一致的证据 |

后文按这张表从“事实如何产生”逐步走到“如何高效地读取观点”。

---

## 3. 第一层：`Session` 如何把一次操作变成可恢复事实

### 3.1 需求背景：仅有可变消息数组为什么不够

最直观的会话实现通常是一个数组：收到用户输入就 `messages.push(...)`，UI 和模型都读这个数组，再发一个事件通知其他模块。

这种做法存在两份状态：

```text
messages 数组       ← 业务当前状态
事件通知流           ← 旁观者看到的变化
```

只要任何一个监听器漏处理、异步失败、重载后未补齐或更新顺序不同，两者就可能不一致。持久化、遥测、模型 history、浏览器回放又会各自发展出第三、第四份解释。

`Session` 反过来设计：它只拥有一条私有的仅追加 `log`。消息数组、surface、请求 header、模型 history 都是这条日志的投影。`Session.append()` 是会话事实的唯一入口。

### 3.2 写入时的四个约束

每次 `append()` 在真正进入日志前完成以下工作：

1. **无损 JSON 快照**：拒绝函数、循环引用、`Map`、`Date`、非有限数字等不能稳定持久化的数据；记录进去的正是被校验的那份值。
2. **连续序号**：新事件序号恒为 `seq = log.length`，因此 `events[i].seq === i`。日志中不存在“跳过但以后再补”的位置。
3. **深度冻结**：接受后的事件及嵌套消息不可变，调用方不能事后改写已经被持久化或投影的历史。
4. **surface 转换校验**：能生成模型消息的事件必须声明 `append` 或 `replace` 如何影响模型可见的有序 surface；非法替换在事实入口被拒绝。

这四项约束共同回答了“为什么日志可回放”：不是因为稍后有一段聪明的修复代码，而是因为每个被接纳的事实从一开始就拥有稳定的值、稳定的位置和可验证的转换规则。

### 3.3 `SessionHeader`：不要把所有东西都塞进事件流

日志中的事件应描述“可回放的会话事实”。但 `version`、`cwd`、`createdAt`、`parentSession`、`seedLength`、`delegationDepth` 等字段并不都是对话的一部分：它们用于定位存储、识别一次会话生命周期、约束恢复和解释谱系。

因此这些字段放进不可变的 `SessionHeader`，而不是伪造一个 `session/meta` 事件。这样 fork/replay 复制的仍是对话历史；存储身份和生命周期边界则由新的会话 header 显式承担。

这是一个容易迁移的判断标准：

> 若一个字段参与“下一次如何解释历史”，它可能属于事件；若它只说明“这份历史是谁、何时、在哪里被保存”，它应是日志外的身份元数据。

### 3.4 三类事件边界：发布、观察与持久性不能混为一谈

`SessionStore` 为同一会话提供三种不同语义：

| 事件 | 含义 | 失败语义 |
|---|---|---|
| `session/created` | 一个 Session 准备公开；同步异常可否决发布 | 适合安装必须在会话开始即成立的能力 |
| `session/event` | 事件已提交到内存日志后的观察流 | 观察者异常被记录并隔离，不能撤销事实 |
| `session/flush` | 调用方要求所有监听者跨过持久性检查点 | 等待全部 listener，失败会回到调用方 |

这一区分尤其重要。`append()` 的热路径不能等待磁盘 I/O，否则流式响应会被持久化速度拖住；但在模型请求前、工具副作用前、轮次完成或退出时，调用方又需要一个明确的“已经落盘”边界。因此，事件提交与持久确认必须是两个阶段。

`SessionStore.flush(session)` 是唯一受作用域保护的 flush 入口。它不是普通的通知，而是让所有持久性监听器结束后再返回的屏障。

### 3.5 缺少本层会发生什么

没有仅追加日志和连续 `seq`，后续组件都没有可靠输入：

- 持久化不知道哪些是已提交事实，哪些是调用方仍可修改的对象；
- 投影无法记录“已处理到哪一个事件”；
- fork 无法定义稳定前缀；
- 崩溃恢复无法区分“最后完整事件”与“半写事件”；
- 客户端无法用一个单调游标拒绝过期推送。

所以 `Session` 并不是普通数据容器，而是整个运行时的一本顺序账本。

---

## 4. 第二层：持久化不是“把数组写文件”，而是可恢复生命周期

### 4.1 为什么需要 capability seam

会话日志需要落盘，但 agent loop 不应知道 JSONL、SQLite 或对象存储的细节。否则更换后端、实现备份、开发内存后端或测试时，都要改 loop 的核心流程。

`SessionPersistence` 因此定义为 `ctx.sessionPersistence` 上的 Service Definition。它声明以下语义，而不规定介质：

```text
create / ensureMaterialized  注册或显式物化会话
append                        只追加连续事件，成功即已持久
inspect                       只读逻辑视图，不提交修复
load                          返回可恢复的平衡日志，并提交必要修复
prepare                       取得独占的、未发布的精确 Session
readFrom                      从水位后的物理连续前缀读取
list / listSnapshots          列举轻量 header 与 revision
```

这就是一个完整 capability seam 的分工：

```text
Service Definition:  dsh-session-persistence（承诺语义）
Service Provider:    dsh-session-persistence-jsonl（文件与 fsync 机制）
Consumer:            agent loop、session query、控制器、恢复/列表功能
```

持久化层可替换，消费方不需要条件判断“当前是不是 JSONL”。

### 4.2 `PersistenceCoordinator`：把难题从 provider 中抽出来

文件后端真正难的并不是序列化一行 JSON，而是下面这些运行时情况：

- 同一会话的并发 flush、load、append 不可交错写入；
- 高频 `assistant/chunk` 不应一条事件一次 fsync；
- 会话销毁时，尚未写入的事件必须排空；
- HMR 后新后端要接管仍活着的 Session；
- 冷恢复要区分物理撕裂与逻辑未结束轮次；
- 新建 session 和已存在日志发生同 id 冲突时必须拒绝。

`PersistenceCoordinator` 将这些存储无关的生命周期编排集中起来。provider 只实现窄小的存储原语：读取前缀、读取 revision、原子首批追加、后续追加、提交修复、列举和可选关闭。

其关键数据结构体现了两种不同的所有权：

```text
按 SessionId：状态、持久 cursor、每 id 串行链、准备结果
按精确 Session 对象：初始化和 write-behind 控制器
```

不能只按 id 管理。id 描述持久存储槽位，而 live `Session` 对象描述本次内存生命周期；用错其中之一会让旧会话的清理误伤同 id 的新会话，或把不同生命周期的事件混入同一缓冲区。

### 4.3 有界 write-behind：性能优化不能改变事实语义

流式模型可能产生大量 `assistant/chunk`。若每个 chunk 都直接写文件并 fsync，会带来严重写放大；若为性能直接丢弃 chunk，又会破坏回放、时间线、引用关系和连续序号。

项目选择保持逻辑事件完全不变，只优化物理批次：

```text
session/event
  → 每会话 SessionWriteBehind 入队
  → 首个待写事件启动固定窗口
  → 窗口内新增事件合并为一个连续批次
  → 通过每 id 串行链 appendBatch
  → 后端确认耐久后才推进持久 cursor
```

这里是**固定窗口**，不是“每来一个事件就重置计时器”的防抖：持续流式输出不能无限推迟第一次落盘。显式 `flush` 会取消等待，排空当前和屏障期间新接纳的事件。

后台写失败时，失败批次保留在队列中而不是静默丢弃；自动重试暂停，避免故障期间形成计时器风暴；下一次明确 `flush`、退役或 provider 卸载会将失败显式暴露给相应调用方。

这体现了重要边界：

> 批处理仅缩小 I/O 次数，不改变“哪些事件发生过”、事件顺序或显式耐久性检查点。

### 4.4 JSONL provider：物理表示可以压缩，逻辑表示不能偷换

第一方 `JsonlSessionPersistence` 为每个 Session 保存一份逻辑 JSONL：第一条为 header，后续为连续事件。默认物理编码使用带校验和的 Zstandard 帧；连续 chunk 可以打包为一条存储记录以减少包装开销。

但读取时必须恢复精确的逻辑 `SessionEvent` 序列：原始 `seq`、时间、chunk 边界和 surface 元数据都不可丢失。原因不是保守地保留更多字节，而是这些事件已经是其他机制的共同事实：日志连续性、UI 回放、遥测、事件引用与崩溃恢复都依赖它们。

首批 header 与事件通过临时文件、fsync、发布和目录同步完成物化；追加失败会尝试回滚到原文件大小，避免下次用未推进的 cursor 重试时写出重复序号。

### 4.5 崩溃恢复：物理损坏与逻辑中断必须分开处理

崩溃不是一种状态，而是至少两种不同问题：

| 情况 | 例子 | 恢复策略 |
|---|---|---|
| 物理撕裂尾部 | 最后一个 frame 或 JSONL 记录只写了一半 | 丢弃无法确认提交的尾部字节；保留此前完整记录 |
| 逻辑中断轮次 | `turn/start`、已完成消息和工具调用已落盘，但缺少 `step/end` / `turn/end` | 保留所有已提交事件，补合成的工具结果、step/turn closer，并以 `interrupted` 结束 |

直接从最后一个完整 `turn/end` 截断看似简单，但会删除崩溃前已经确认落盘的大量真实工作。项目的原则是：**从未完整写入的东西可以丢；已提交事实不能被静默删除。**

`inspect()` 允许历史查询看到内存中的平衡逻辑视图，但不改存储；`load()` 与 `prepare()` 则在冷恢复前提交需要的撕裂尾修复与合成 closer。这让“查看历史”与“准备继续运行”拥有不同副作用语义。

### 4.6 拒绝而不是猜测

恢复路径还负责验证：

- header 版本是否被当前 harness 支持；
- 存储 header 的 id、cwd 与请求的会话身份是否一致；
- 事件 `seq` 是否连续；
- 已提交前缀是否损坏；
- 未知事件是否显式标记 `ignorable`。

未知且必需的事件不能静默跳过：跳过它可能使之后的事件被错误解释。因而系统宁可报出“需要升级或日志损坏”，并保留原始日志位置，也不构造一份表面上可恢复、实际上缺失事实的会话。

### 4.7 `SessionPreparation`：恢复的关键不是“再创建一份对象”

冷会话有两个消费者：历史查询想读取它，恢复操作想激活它。如果两者都各自读文件、解压、解析、验证、冻结和构造 `Session`，不仅浪费 I/O，还可能在设置和发布间换成另一份对象图。

`SessionPreparation` 持有一份**精确的、尚未公开的 Session**：

```text
inspect / borrowSession
  → 读到不可变冷态；可保留为有限 LRU 准备结果
  → 准备恢复时检查 revision 是否仍一致
prepare
  → 独占这份准确 Session
  → 提交修复、建立持久 cursor
agent loop setup
  → 在该 Session 周围装配 agent 上下文与可选配置
enter + announce
  → 公开同一个 Session 和同一个 agent
```

“精确对象”是承重条件。若缓存的只是 `{ meta, events }`，恢复时仍要重新构造和验证 Session，设置阶段看到的对象与最后发布的对象不再有强关联。

这也是 prepare-then-publish 模式：先完成可能失败的 I/O、验证和装配，再公开一个已经满足前置条件的对象；失败时释放准备结果，不向系统泄露半初始化实体。

### 4.8 HMR 接管为何是另一条路径

HMR 时 Session 仍是 live 的，内存中的开放轮次才是权威。新 provider 可以接管存储中的已持久前缀，截断纯物理撕裂尾，并持久化 live Session 领先于磁盘的后缀；但它绝不能把 live 的开放轮次错误修成 `interrupted`。

这再次说明“冷恢复”与“接管活对象”不能共用一个含糊的 `load()`：两者面对的权威来源不同。

---

## 5. 第三层：投影把“无穷历史”变成“当前可读状态”

### 5.1 需求背景：为什么不能让每个领域自己扫描日志

日志善于保存事实，却不适合作为每次读取的界面模型。以待办、目标、标题、计划和用量为例，消费者真正想问的是：

```text
“现在的 todo 是什么？”
“当前 plan 是启用、关闭还是等待落地？”
“这个冷会话的标题是什么？”
```

如果每个领域分别在 host、API、浏览器中实现一次 `switch(event.type)`：

- 相同的状态转移会有多份实现；
- 不同页面会采用不同基线、重试和乱序处理；
- 新领域不得不修改客户端 Session 核心类；
- 冷会话没有 live agent 时，隐藏在 agent/UI 中的 getter 无处运行。

`SessionProjectionRegistry` 的目标是把这些重复的“订阅、游标、缓存、发布”收进框架，让领域只保留可测试的状态计算。

### 5.2 投影单元：把领域代码缩小为纯 fold

每个 `ProjectionDefinition` 包含：

```text
init(header)                 空日志对应的初始状态
apply(state, event)          同步纯状态转移
stateSchema                  检查点恢复前的状态校验
stateVersion                 状态结构或 fold 语义变化时的失效版本
wire.view(state)             可选：host 状态 → 客户端完整当前值
wire.viewSchema              可选：客户端边界校验
```

它刻意不是 `get(session)` 这样的不透明 getter。一个不透明 getter 让框架不知道：状态是否可检查点化、能否从中间继续、是否同步、该如何重放冷会话。显式的 `init + apply` 使框架可以持有水位、增量推进和持久检查点。

单元必须满足三条纪律：

1. `init`、`apply`、`view` 都必须同步；异步计算会让不同 key 落在不同事件位置，破坏一致切面。
2. 状态必须是 plain JSON；否则无法安全检查点化。
3. 不关心某事件时返回**同一个状态引用**，而不是内容相同的新对象；这既表达“状态未变”，也避免无意义的下游 view 与推送工作。

### 5.3 为什么框架主动驱动，而不是领域各自订阅

注册表只订阅一次 `session/event`。每个已提交事件由注册表按注册单元驱动；每个 Session/单元保存一个 `{ state, observedSeq }` cell。

```text
已提交 event(seq = 42)
  → registry 找到每个注册单元的 cell
  → apply(previousState, event)
  → state 引用未变：只推进 observedSeq，无后续工作
  → state 引用变化：必要时计算 view
  → view 引用也变化：schema 校验后发布 (key, value, seq=42)
```

领域不拥有订阅，客户端也不重放领域事件。这样保证一个领域状态机在 host 只有一份实现；客户端接收的是成品值而不是另一套要长期同步的 reducer。

### 5.4 host state 与 client view 为什么要分开

一个投影内部可能需要丰富的中间状态，例如记录最近消息序号、累计值、是否有待确认意图；客户端却只需要一个简洁的标题、列表或状态标签。

因此项目区分：

```text
SessionProjectionStateMap  → host 内部折叠状态
SessionProjectionMap       → 客户端可见的完整当前值
```

`stateOf(session, key)` 供同进程 host 读取状态；`snapshot(session)` 只返回声明了 `wire` 的客户端值。不是所有内部字段都应穿过协议，也不是为了 UI 而牺牲 host fold 的表达能力。

### 5.5 一致快照与增量推送

`snapshot()` 同步读取所有客户端可见单元，返回：

```text
{ asOfSeq, values }
```

其中 `asOfSeq` 表示每个值都至少反映到的同一个日志位置。它不是装饰字段，而是客户端处理基线与实时推送的排序依据：新的 `session/projection` 帧带有致因事件的 `seq`，客户端采用高 seq 胜出的规则，旧基线不会覆盖更新帧，丢一帧只会暂时陈旧，下一基线或更高 seq 的值会自愈。

这里有两道成本闸门：

1. state 通过 `Object.is` 不变时，不计算 view；
2. state 变化但 view 仍复用同一引用时，不发布给客户端。

因此领域可以维护只供 host 使用的内部状态，而不让每次内部变化都变成网络帧。

### 5.6 注册本身也是 effect

`register()` 和 `onChanged()` 都挂在调用方 Cordis fiber 的 `ctx.effect()` 生命周期上。插件卸载时，对应 key 和缓存 cell 会自动撤销；同 key 被多个 preset 注册时引用计数保证最后一个注册者退出前不会错误移除该能力。

这不是资源清理细节，而是插件架构的一部分：

> 注册一项能力等同于创建一个可逆副作用。能力存在多久，取决于拥有它的插件生命周期，而不是某个全局数组忘记删除。

对于依赖某个投影的 host 消费者，当前约束是明确失败：缺少注册表或所需 key 不能静默替换成默认空值。因为“能力未装配”与“投影合法地为空”是不同的业务状态。

---

## 6. 第四层：持久投影缓存如何在加速时不制造第二真源

### 6.1 需求背景：日志正确，但冷读不能总是从零开始

纯日志回放的正确性很好，但长会话的冷读成本会随历史增长。会话列表如果要显示标题、子 agent 身份或摘要，每次都完整读取、解压和 fold 全部日志，代价会很高。

常见误区是把“当前投影”另存为权威状态，然后直接更新它。这只是在日志旁边重新制造一份会漂移的状态。

项目的缓存是另一种思路：记录“某个纯 fold 已经可靠处理到哪里”，而不是宣称“这是新的真相”。每个单元的检查点行概念上为：

```text
(sessionId, key, ver, seq, val)

ver  = 产生该状态的 ProjectionDefinition.stateVersion
seq  = val 已折叠到的最后一个事件位置
val  = 可校验的 host 状态 JSON
```

### 6.2 缓存的核心承诺：至多陈旧，绝不超前

缓存写入时执行的顺序是：

```text
1. 从 live registry 取得某个检查点切面
2. 先通过 SessionStore.flush() 确保该切面覆盖的日志已耐久
3. 再把检查点行写入 session_projcache 存储域
```

于是发生崩溃时只有两种结果：

- 日志已写，缓存尚未来得及写：下次多回放一段尾部；
- 缓存和日志都写：直接从检查点加尾部开始。

不允许出现“缓存宣称处理了 seq 100，但持久日志只到 seq 80”。那会产生并不存在的视图值，属于比慢更严重的错误。

因此缓存的写入可以 fail-soft：写失败只记录警告，稍后重试或从日志重算即可；日志持久化却不能把失败伪装成成功。两者的可靠性等级不同，因为两者承担的职责不同。

### 6.3 三类防止旧缓存混入新会话的证据

一个 session id 只是槽位名，不能单独证明缓存属于当前日志生命周期。缓存还使用：

| 证据 | 防止的问题 |
|---|---|
| `createdAt` + `cwd` 身份 | id 被重建或换了持久化来源时复用旧行 |
| `stateVersion` | fold 语义或序列化字段变更后按旧规则继续推进 |
| `stateSchema` / `viewSchema` | 文件损坏、外部写入或过期字段被当作可用状态 |

任一不匹配，缓存行都被看作不存在或不可用；系统回退到准确日志，而不是迁移一份无法证明正确性的派生数据。

### 6.4 为什么需要“最低水位前一个事件”的锚点

从检查点续算不能只读“检查点之后”的事件。假设检查点说自己到 `seq=100`，但崩溃修复后存储日志实际缩短到 `seq=90`；若直接从 101 读取空尾部，系统无法发现缓存已经越过真实日志末尾。

`restoreFloor()` 因此从所有可用单元的最低水位前一个事件开始读。这个锚点让尾部读取不仅携带新事件，也证明存储日志至少延伸到哪里：

```text
checkpoint 最低 seq = 100
  → readFrom(99)
  → 读到 seq 99 或更后：能验证日志长度
  → 返回空：检查点可能越界，拒绝并从 seq 0 全量重读
```

这是一种很值得复用的“证据锚点”模式：当缓存声称覆盖某个位置时，恢复路径要读取足以验证该位置仍存在的最小证据，不能仅凭空结果相信缓存。

### 6.5 per-record 存储：隔离写放大与故障范围

投影缓存采用 `session_projcache` 域的每会话一条记录布局，而不是一个包含全部会话的大 JSON 文件。这样一次 checkpoint 只替换当前会话的小文档：

- 写放大不随历史会话数增长；
- 一个坏记录只影响一个会话；
- 存储域自己的写链保证内存读取与耐久写入不互相穿透；
- 缓存仍不依赖 JSONL 的路径或具体持久化 provider。

缓存与日志之间只有逻辑依赖，没有物理目录耦合；这保住了两个 seam 的可替换性。

---

## 7. 把所有组件串起来：三条关键时序

### 7.1 正常热路径：记录、投影、异步落盘

```text
agent / command / tool
  │ Session.append(event)
  ▼
验证 → JSON 快照 → 深冻结 → 分配连续 seq → 写入内存日志
  │
  ├─ session/event（已提交事实的观察流）
  │    ├─ ProjectionRegistry 同步推进所有已注册单元
  │    ├─ 需要时发布 client view + seq
  │    ├─ PersistenceCoordinator 进入 write-behind 队列
  │    └─ ProjectionCache 记录脏状态、按策略尝试 checkpoint
  │
  └─ append 立即返回，不等待 I/O

语义检查点 / turn/end / teardown
  │
  ▼
SessionStore.flush(session)
  → 等待持久化缓冲完全排空
  → 调用方得到可观察的失败或成功
```

一个重要细节：投影在 `session/event` 后立即跟进内存已提交事实，因此 live UI 的值可以很新；持久化则允许短暂滞后。二者不冲突，因为后者有显式 `flush` 边界，前者从不把缓存当真源。

### 7.2 冷查询：读取而不激活 agent

```text
SessionQuery.observeSession(id)
  │
  ├─ 若 Session 仍 live：借用它当前不可变日志切面
  │
  └─ 若 Session 已冷：borrowSession / inspect
        → 读取、验证、必要时在内存中配平逻辑日志
        → 保留精确的 prepared Session 供后续复用
        → 可用 ProjectionCache 播种 cell，再回放准确尾部

返回：{ header, events, cursor, projections?, source }
```

这条路径不会为了“展示历史”而创建长期运行的 agent；它把历史读取限制为一个可释放的观察 lease，既避免重复冷解析，也不混淆查询和执行生命周期。

### 7.3 冷恢复：先修复和准备，再发布精确对象

```text
ctx.agents.resume(sessionId)
  → sessionPersistence.prepare(sessionId)
      → 检查 revision
      → 对冷日志提交必要的撕裂尾 / interrupted-turn 修复
      → 预留精确 prepared Session
  → agent loop 在其周围完成 setup
  → SessionStore.enter(preparedSession)
  → SessionStore.announce(preparedSession)
  → 后续新事件继续进入同一日志与同一投影单元
```

恢复后的新事件不是“另开一段记忆”，而是从已验证的持久 cursor 继续追加。投影注册表水位、检查点和 UI 推送同样从同一条 `seq` 轴继续前进。

---

## 8. 这套设计实际保证什么，又刻意不保证什么

### 8.1 已建立的强保证

| 方面 | 保证 |
|---|---|
| 事实顺序 | 单个会话日志连续、仅追加；已提交事件不可被调用方事后修改 |
| 持久写入 | `append()` / `flush()` 的成功路径跨过后端耐久边界；同 id 操作串行 |
| 冷恢复 | 完整已提交中断轮次保留并配平；不可确认的物理尾碎片才丢弃 |
| 解释正确性 | 格式、未知必需事件、id/cwd、连续性和损坏均明确拒绝 |
| live 投影 | 所有已注册单元在同一内存事件流上推进；`snapshot()` 提供统一 `asOfSeq` |
| 冷投影 | 检查点版本/身份/schema 不匹配时回退日志；缓存不会超过持久日志 |
| 插件生命周期 | 投影注册、监听和缓存资源随拥有者 fiber 可逆清理 |

### 8.2 有意保留的边界

| 边界 | 当前选择 | 为什么不能误解 |
|---|---|---|
| 分布式写入 | 每 id 的进程内串行化，不提供跨进程共识 | 外部持续写入会使 revision 重试延后，不等于分布式锁 |
| 后端兼容 | 预发布格式为 v0，只有有限明确的旧形状规范化 | 不承诺泛化迁移；读不懂必须拒绝 |
| 持久化延迟 | write-behind 有固定批处理窗口 | 它限制主动等待，不是端到端 fsync SLA |
| 投影缓存 | 可选、fail-soft、可能陈旧 | 它优化读取，不能承担业务写入真相 |
| 投影运行成本 | 每已提交事件会经过每个已注册单元 | 当前依赖纯轻量 fold 和引用闸门；热点出现后可再做事件类型预过滤 |
| 客户端语义 | 客户端接收全量值和 seq，不解释领域事件 | 这不是“客户端功能不足”，而是避免双份 reducer 的刻意边界 |

理解这些边界的价值在于避免过度承诺。一个单进程可恢复运行时不应伪装成提供 exactly-once、跨进程事务和分布式共识的系统。

---

## 9. 可复用的架构经验与设计模式

### 9.1 一个真源，多个可丢弃视图

**模式：Event log as authority, projections as disposable views。**

适用于订单状态、任务编排、审批流、聊天、IoT 指令、工作流执行记录等场景。先定义最小不可变事实集合，再从事实推导当前状态、UI、检索索引和统计数据。

迁移前要问：

- 这个状态能否由已记录事实确定？能，就不要额外做权威可变字段。
- 这个字段只为加速读取吗？是，就必须携带版本/水位，并允许丢弃重建。
- 这个信息是否影响外部副作用的去重或恢复？是，就应被建模为持久事实，而非临时内存标记。

### 9.2 事实和观点分层，避免“同步两个真相”

**模式：immutable fact + pure projection。**

日志不改写；所有需要“改历史展示”的行为都改投影规则。例如模型 surface 可替换一段可见节点，但被替换的原始事件仍留在日志中。投影可以变化、缓存可以失效、UI 可以重连，事实不因这些读侧决策而丢失。

这比“给两份状态加同步器”更可靠，因为同步器只能补救分歧，单向派生让分歧本身不易表达。

### 9.3 写后缓冲 + 明确耐久屏障

**模式：fast commit in memory, asynchronous durability, explicit barrier。**

适用于高频 telemetry、流式输出、编辑器操作等。热路径只做验证和内存提交；后台按有界窗口写入；真正依赖顺序和耐久性的操作必须调用可等待屏障。

关键不是“异步写”本身，而是明确告诉调用方：

- 哪些操作已经被接受但还未耐久；
- 哪个 API 可以把它们全部排空；
- 失败会在哪个边界暴露；
- 退出时如何处理未完成队列。

### 9.4 物理损坏与业务中断分开恢复

**模式：discard uncommitted bytes; preserve committed intent; synthesize a terminal business state。**

一个文件尾部写一半，与一个业务操作已经执行一半，不能使用同一“截断”策略。前者没有足够证据，丢弃合理；后者已有持久事实，应该保留并追加可解释的结束状态。

迁移到付款、发货、邮件或工作流时，可把“合成 closer”换成失败状态、补偿请求或待人工处理标记；核心仍是让恢复后状态机闭合，而不是假装中间事实从未发生。

### 9.5 先准备，后发布

**模式：prepare → configure → publish; rollback by disposal。**

任何要在公开前完成验证、授权、资源装配的实体，都适合使用：

```text
构造未公开对象
  → 完成可能失败的准备
  → 公开精确对象
  → 失败则仅释放准备资源
```

这避免“对象已经被全局看见，但其依赖还没装好”的半初始化状态。数据库迁移、连接切换、worker 激活、插件挂载和远程会话恢复都可采用这个顺序。

### 9.6 per-key/per-aggregate 串行化与版本证据

**模式：single-writer queue per aggregate + revision validation。**

`PersistenceCoordinator` 用每 SessionId promise chain 防止同一日志的写入互相插入；准备结果还用 revision 确认读到的冷状态仍对应当前存储。前者解决“本进程的动作交错”，后者解决“读取期间底层来源变了”。

迁移时可替换为 actor mailbox、数据库事务、分区队列或乐观锁，但要保留两层问题意识：服务端写入互斥与读者持有证据的新鲜度并不是一回事。

### 9.7 缓存必须是“带水位的证据”，不是无来源快照

**模式：versioned checkpoint + tail replay + anchor validation。**

只存一个 JSON 快照无法判断它是否过期、是否来自同一生命周期、是否跨过了真实日志末尾。可靠检查点至少应有：

- 身份绑定；
- 状态版本；
- 已覆盖位置；
- schema 校验；
- 用最小锚点验证来源没有向后缩短；
- 回退全量重建的路径。

这适用于搜索索引、水位消费、物化视图、统计预聚合、流处理状态和离线同步游标。

### 9.8 注册即 effect，卸载即反注册

**模式：resource ownership follows plugin/fiber lifecycle。**

不要让插件向全局 registry `push()` 后希望某个未来分支记得删除。让注册操作返回 disposer，并由拥有者的生命周期自动释放。这样 HMR、测试隔离、嵌套实例和多 preset 共存都有清晰的资源所有权。

---

## 10. 若要迁移到另一个系统，推荐的落地顺序

不要从“先建一张当前状态表”或“先加缓存”开始。更稳妥的实施顺序如下：

1. **确定聚合根。** 一个订单、一个工作流实例、一次会话或一个设备任务，谁拥有其顺序事实？
2. **定义最小事件词汇。** 每个事件只记录已发生、以后仍需解释的事实；给事件确定稳定身份和顺序规则。
3. **在写入口建立不变量。** 序号、载荷可序列化性、状态机前置条件在追加时拒绝，而不是等读侧发现。
4. **分离元数据和可回放状态。** 存储身份、来源位置和谱系不要伪装为业务事件。
5. **抽象持久化 seam。** 在接口层规定 append、load、修复、列举和失败语义；把文件/数据库/对象存储细节留给 provider。
6. **为崩溃窗口设计恢复协议。** 明确哪些字节可丢、哪些已提交事实必须保留、怎样让中断状态机闭合。
7. **引入 prepare-then-publish。** 让历史读取、验证、授权和恢复使用同一个未发布对象，而不是各自重新构造。
8. **把读侧写成纯投影单元。** 领域提供 `init + apply + schema + version`，框架统一处理订阅、水位、快照和推送。
9. **最后再加入检查点缓存。** 先证明从完整日志重放正确，再用版本、水位和身份使缓存可丢弃、可验证。
10. **测试崩溃与生命周期边界。** 覆盖写到一半、完整轮次未结束、缓存领先/落后、未知事件、HMR 接管、dispose 与 flush 并发、准备后来源变更等场景。

最需要避免的反模式是：一开始让 UI、服务和存储各维护一份“当前状态”，等恢复需求出现后再试图把它们拼成日志。那时事件语义、幂等键、恢复边界和版本规则已经散落在各层，重构成本通常远高于先确立一个真源。

---

## 11. 源码阅读地图

| 想深入的问题 | 首选位置 |
|---|---|
| 事件如何被快照、冻结、编号并发布？ | [`packages/core/session/src/index.ts`](../packages/core/session/src/index.ts) |
| 模型 history 如何从日志 surface 派生？ | [`packages/core/session/src/surface.ts`](../packages/core/session/src/surface.ts) |
| 持久化服务承诺哪些语义？ | [`packages/session/session-persistence/src/index.ts`](../packages/session/session-persistence/src/index.ts) |
| 批处理、串行化、准备、修复、HMR 接管和退役怎样协作？ | [`packages/session/session-persistence/src/coordinator.ts`](../packages/session/session-persistence/src/coordinator.ts) |
| write-behind 的固定窗口和 flush 屏障怎样实现？ | [`packages/session/session-persistence/src/write-behind.ts`](../packages/session/session-persistence/src/write-behind.ts) |
| JSONL 如何物化、追加、fsync 和修复？ | [`packages/session/session-persistence-jsonl/src/index.ts`](../packages/session/session-persistence-jsonl/src/index.ts) |
| 投影单元、cell、水位、snapshot 和 restore 怎样实现？ | [`packages/session/session-projection/src/index.ts`](../packages/session/session-projection/src/index.ts) |
| 检查点如何与日志保持“缓存不领先”关系？ | [`packages/session/session-projection-cache/src/index.ts`](../packages/session/session-projection-cache/src/index.ts) |
| 历史查询怎样借用 live/prepared 一致切面？ | [`packages/session-query/session-query/src/observation.ts`](../packages/session-query/session-query/src/observation.ts) |
| agent loop 怎样拿 preparation 并恢复发布？ | [`packages/core/agent-loop/src/index.ts`](../packages/core/agent-loop/src/index.ts) |

配套的权威说明和设计证据：

- [会话持久化包说明](../packages/session/session-persistence/README.zh.md)
- [会话投影包说明](../packages/session/session-projection/README.zh.md)
- [投影缓存包说明](../packages/session/session-projection-cache/README.zh.md)
- [会话投影子系统文档](../docs/subsystems/session-projection.zh.md)
- [会话持久化作为抽象服务的决策](../.agents/notes/implemented/architecture/2026-06-14-session-persistence.zh.md)
- [共享持久化协调器的决策](../.agents/notes/implemented/architecture/2026-06-18-shared-persistence-write-coordinator.zh.md)
- [可复用 Session 准备阶段的决策](../.agents/notes/implemented/architecture/2026-08-05-session-preparation.zh.md)
- [会话投影与命令日志的原始 RFC](../.agents/notes/proposed/architecture/2026-07-27-session-projection-and-command-log.zh.md)

---

## 12. 最终总结

这次会话持久化/投影重构的真正产物不是“多了 JSONL 文件、投影注册表和缓存”，而是一条清晰的数据责任链：

```text
SessionEvent
  = 已发生、可验证、不可变的事实

SessionPersistence
  = 事实跨进程仍可读取、续写和修复的承诺

SessionPreparation
  = 一份冷事实从检查走到恢复发布时的精确所有权

ProjectionDefinition / Registry
  = 从事实得到当前业务状态的唯一计算地点

ProjectionCache
  = 可验证、可丢弃、只用于缩短回放的加速层
```

把它迁移到其他系统时，最应保留的是四条原则：

1. **先选唯一事实源，再构建所有读模型。**
2. **先定义崩溃后的可解释状态，再优化热路径和读取速度。**
3. **把缓存设计成带位置与版本的证据，而不是第二份真相。**
4. **把计算、存储、生命周期和载体分别放在可替换的 seam 中。**

这样得到的不是一套只能服务当前 agent 的技巧，而是一种普适的运行时设计方法：让系统在正常时快，在中断后仍诚实，在演进时仍能解释过去。
