# DeepSeek Harness 工作流程：从输入一句话到项目反馈

> 面向 Java 工程师：先讲「一句话进来后发生了什么」，再用 Java 概念逐条对照。
> 配套阅读：[CORDIS.zh.md](CORDIS.zh.md)（底层插件框架）、[MODULES.zh.md](MODULES.zh.md)（模块全景）、[QUICKSTART.zh.md](QUICKSTART.zh.md)（快速上手）。

## 一、先建立整体感：一家「一切皆插件」的餐厅

DeepSeek Harness（`dsh`）是一个**插件化的智能体运行框架**。可以用一家餐厅来理解它：

| 餐厅角色 | dsh 对应 | 说明 |
|---|---|---|
| 顾客 | 用户 | 输入一句话（点一道菜） |
| 点单记录本 | **会话日志（session log）** | 追加式记录，是唯一真相来源 |
| 服务员 | **`agent-loop` 驱动循环** | 接单、派活、上菜，是真正「跑循环」的地方 |
| 大模型后厨 | `llm` 适配器 | 生成内容 |
| 厨房工具（切菜/查库存） | `tools` 注册表里的工具 | 模型决定调用时才执行 |
| 店规/领班 | `guard` / `interaction` | 审批、限流、防失控 |

**最关键的一条**：这个项目没有「特权核心」。连模型适配器、工具注册表、会话日志、驱动循环本身都是插件，都可以从配置里替换掉。所以理解它 = 理解「这条链路里每个环节是哪个插件、通过什么事件串起来」。

---

## 二、三个必须先懂的核心概念

| 概念 | 大白话 | 精确含义 | Java 对照 |
|---|---|---|---|
| **step（一步）** | 服务员去后厨下单一次 | **一次模型请求 + 它调用的所有工具** | 一次服务调用 |
| **turn（一轮）** | 从点单到这道菜上齐 | **0 个或多个 step**。输入进来就开一轮，模型不再欠任何东西就结束 | 一个完整事务/请求生命周期 |
| **session log（会话日志）** | 那本点单记录本 | **追加式**的类型化事件日志，是**唯一真相来源**。模型看到的「历史对话」是从日志**推导**出来的，不是单独存的 | Event Sourcing 事件流 |

> 铁律：**模型可见 ⟺ 已记录**（model-visible ⟺ logged）。任何要发给大模型的内容，都必须能从会话日志里重建出来。这是一条贯穿整个项目的硬约束——想让模型看到新信息，就必须先新增一种会话事件类型。

`step` 和 `turn` 的层级关系：

```
turn（一轮）
 └── step（一步）: 一次模型请求 + 它调的工具
 └── step（一步）: 又一次模型请求 + 它调的工具
 └── ...
```

一轮里可能有多个 step（模型反复调工具），也可能零个 step（输入被拒绝、输入为空）。

---

## 三、完整流程总览

一张图看懂从输入到反馈：

```
用户输入一句话
   │
   ▼
[1] 入口：通过 ctx.agents 拿到/创建这个会话的 Agent（一个会话 = 一个 Agent）
   │  输入经 Agent.send() / followup() / steer() / inject() 进入 inbox（收件箱）
   │  inbox 有两条队列：next-turn（下一轮） 和 next-step（下一步）
   ▼
[2] 唤醒驱动循环（agent-loop 的 wakeDriver）
   │  记录 turn/start 到日志（开一轮）
   ▼
[3] preStep（预步）：从 inbox 认领输入
   │  → 组装系统提示词（system-prompt 模块）
   │  → 触发 agent/pre-step 事件（插件可拦截：改写输入 或 拒绝这步）
   ▼
[4] 记录 step/start，把 user/message 写入日志
   ▼
[5] 组装请求（buildRequest）：
   │  deriveMessages() 从日志推导出模型历史 + 系统提示词 + 工具 schema
   │  → 触发 agent/request 事件（插件可改模型/参数）
   ▼
[6] 调大模型：ctx.llm.stream() 流式请求
   │  → assistant/chunk*（逐个 token 流回，记录到日志）
   │  → 组装成 assistant/message（记录到日志）
   ▼
[7] 有没有工具调用？
   ├─ 没有 → 这步完成，直接到 [8]
   └─ 有   → 走工具执行管线（见第四节）
   ▼
[8] 记录 step/end
   │  判断：工具还欠一次请求？或又有新输入？→ 回到 [3] 再来一步
   │  否则 → 记录 turn/end（一轮结束）
   ▼
[9] 反馈：UI/编辑器监听 session/event 事件流，把日志渲染成对话界面
```

---

## 四、分阶段详解

### 阶段 1：入口 —— 输入怎么进来

一个运行中的 `dsh` 是一个「插件树」，启动时从 profile / bundle / `cordis.yml` 逐层组装而成。用户输入通过不同的入口进入：

| 入口方法 | 作用 | 大白话 |
|---|---|---|
| `agent.followup(message)` | 普通后续一轮，唤醒驱动 | 「接着聊下一句」 |
| `agent.steer(message)` | 往最近的 step 注入 steering，唤醒驱动 | 「中途插话」 |
| `agent.inject(message)` | 注入模型可见上下文，**不唤醒**驱动 | 「偷偷塞背景资料，等下一句再一起送」 |
| `agent.send(message, target, wakeup)` | 底层统一入口，直接指定目标队列与是否唤醒 | 上面三个的底层实现 |

输入进入 `inbox`（收件箱），它是 agent 拥有的**两个有序待处理消息队列**：

| 队列 | 触发时机 |
|---|---|
| `next-turn` | 作为新的一轮（turn）的输入 |
| `next-step` | 作为当前轮里下一步（step）的输入 |

> Java 对照：`inbox` ≈ 一个带「下一轮 / 下一步」双队列的**消息缓冲队列**（类似 `BlockingQueue` + 优先级分组）。`followup/steer/inject` 是三个预设别名的便捷方法。

### 阶段 2：唤醒驱动 —— `wakeDriver`

`agent-loop` 是真正的主循环驱动，核心类是 `ReactLoopAgent`（源码 `packages/core/agent-loop/src/agent.ts`）。它内部有一个**状态机**（`Phase`）：

| 状态 | 含义 | Java 对照 |
|---|---|---|
| `idle` | 无活动 | 空闲状态 |
| `maintenance` | 跑非 turn 的维护任务 | 后台任务 |
| `running` | 正在驱动 turn/step 循环 | 运行状态 |

收到 `wakeup` 输入时，若处于 `idle`，则进入 `running` 并启动 `kick()` 主循环；若正处于 `running` 或 `maintenance`，则「挂起」唤醒请求（`wakeRequested`），等收敛后再补跑。

### 阶段 3：preStep —— 认领输入 + 拦截点

`preStep()` 做三件事：

1. `inbox.claim(target, turn)` 从收件箱**认领**这一步的输入（turn 边界会额外带上一条 `next-turn` 消息）。
2. `ctx.systemPrompt.assemble(...)` **组装系统提示词**（见阶段 5 的提示词来源）。
3. 触发 **`agent/pre-step` 事件**（waterfall），这是第一个关键拦截点。

`agent/pre-step` 的返回值是一个 `PreStepDecision`：

```ts
type PreStepDecision =
  | { kind: 'reject' }                                    // 拒绝：不进入任何 step
  | { kind: 'enter'; messages: UserMessage[] }            // 进入：改写/替换后进入
```

> 插件可以在这里**改写**认领到的消息、或者**拒绝**这一步。被拒绝或第一次进入就被改写成空，这个 turn 仍然会「消耗一个 turn 边界」记录到日志（但不会发起模型请求），保证日志完整记录了这次尝试。

### 阶段 4：落盘 step/start 与 user/message

- 记录 `step/start`（开一步）。
- 把 `enter` 决定的每条消息追加为 `user/message` 会话事件。

> 注意：`user/message` 不只是人类的直接输入，还包括 `agent.inject()` 注入的合成上下文（文件变更通知、子目录 AGENTS.md、技能内容、定时通知等），靠 `source` 字段区分来源。

### 阶段 5：组装请求 —— `buildRequest`

`buildRequest()` 组装一个「冻结」的模型请求，核心是 `deriveMessages()`：

```
会话日志（唯一真相）
   │  deriveMessages() 投影
   ▼
模型历史消息（messages）
```

同时拼上：

| 组成部分 | 来源 |
|---|---|
| 系统提示词 | `renderPrompt(assembly)`，由各插件贡献的 prompt section 按 `order` 排序拼接 |
| 工具 schema | `assembly.tools`，来自 `ctx.tools` 注册表 |
| provider / model | `agent/request` 事件（waterfall）可替换 |

`agent/request` 是第二个关键拦截点：插件可以替换 provider、model、推理强度等调用配置。之后通过 `ctx.llm.prepareCall()` 解析到具体的模型适配器，并把 `request/header`、`request/context` 记录到日志。

> Java 对照：`deriveMessages()` ≈ Event Sourcing 里的**重放（replay）**——不存最终状态，而是从事件流实时推导出当前状态。`agent/pre-step`、`agent/request` 都是 waterfall 事件 ≈ **Servlet Filter 链**，每个插件可环绕拦截或短路。

### 阶段 6：流式调用模型 —— `llm.stream`

`ctx.llm.stream(request)` 返回一个流，循环 `for await` 消费：

- 每个 `chunk`（token 级别）追加为 `assistant/chunk` 会话事件（保真回放用）。
- 全部消费完后用 `BlockAssembler` 组装成完整的 `assistant/message`（含 token 用量 `usage`）。

> `assistant/chunk` 是「原始流」，`assistant/message` 是「组装结果」——两者都落日志：前者保回放/UI 保真，后者供 `deriveMessages()` 推导历史。

### 阶段 7：工具执行管线（详见下一节）

如果 `assistant/message` 里含 `tool-call` 块，就进入工具执行管线；否则直接判定 step 完成。

### 阶段 8：收尾 —— step/end 与 turn/end

- 记录 `step/end`。
- 判断是否继续：
  - 工具执行后模型还「欠一次请求」（有工具结果待消费）→ 回到阶段 3 再来一步；
  - `inbox` 里 `next-step` 有新 steering → 再来一步；
  - 否则 → 记录 `turn/end`（一轮结束，携带 `TurnEndReason`）。
- `turn/end` 的结束原因有多种：`completed`（正常完成）、`max-tokens`（撞上限）、`aborted`（被取消）、`error`（出错）、`blocked`（被拒绝）。

> 在 turn 即将关闭前，会触发 `agent/turn-stopping` 事件（serial）：某个监听器如果不希望关闭，可以调用 `agent.steer(...)` 再塞一条 steering，主循环会重新读 inbox 继续跑。

### 阶段 9：反馈 —— UI 监听事件流

UI / 编辑器（`client/` 前端、`host/` 后端、`session-query/` 投影等）监听 `session/event` 事件流，把日志渲染成对话界面。因为**模型可见的内容都已在日志里**，UI 可以从日志完整重建整个对话。

---

## 五、工具执行管线（模型要调工具时）

`assistant/message` 里取出 `tool-call` 块后，走 `executeToolCalls()`（源码 `packages/core/agent-loop/src/tool-calls.ts`）：

```
assistant/message 里的 tool-call 块
   │
   ▼
解析参数 + 记录 tool/call 到日志
   │
   ▼
tools/pre-execute 事件（waterfall，插件可拦截）
   │
   ▼
tools/execute 事件（真正执行工具）
   │
   ▼
tools/post-execute 事件
   │
   ▼
记录 tool/result 到日志（结果作为下一步的输入，回灌给模型）
```

**并行调度**：工具调用有并行调度器，关键行为：

| 机制 | 说明 |
|---|---|
| 独享（exclusive）调用 | 形成屏障，等前面的调用排空 |
| 并行（parallel）调用 | 用有界并发池（上限 `maxParallelToolCalls`） |
| 重新分类 | 每次调度前都会**重新判定**并发模式（注册表变化可产生新的屏障） |
| 结果顺序 | 结果和上下文**按模型顺序**提交（不是谁先跑完谁先提交） |
| 取消 | 中止时给未启动的调用记录合成错误结果，保证回放仍有效 |

> 三个 `tools/*` 事件都是 waterfall，是权限、审批、超时、记录等策略插件的挂载点。例如 `interaction` 的审批、`guard` 的超时截止执行器，都通过这些事件实现。

---

## 六、进入哪些模块（核心主干）

这是理解流程最关键的一张表。`ctx.xxx` 是 Cordis 依赖注入容器里的键（相当于 Spring 里按名字取 Bean）：

| 模块 | 目录 | 职责 | `ctx` 键 | Java 对照 |
|---|---|---|---|---|
| `scope` | `core/scope` | 作用域注册原语（每个 agent 独立的世界） | 无（纯库） | 作用域 / `@Scope` |
| `session` | `core/session` | 追加式事件日志 + 内存存储，唯一真相 | `ctx.sessions` | Event Sourcing + Repository |
| `system-prompt` | `core/system-prompt` | 提示词分节 + 工具 schema 组装 | `ctx.systemPrompt` | 模板组装器 |
| `tools` | `core/tools` | 工具注册表 + 守卫执行管线 | `ctx.tools` | 命令注册中心（Handler Mapping） |
| `agent` | `core/agent` | `Agent` 接口、实时注册表、`agent/*` 事件 | `ctx.agents` | 接口 + 服务注册中心 |
| `agent-default-model` | `core/agent-default-model` | 部署默认模型选择 | `ctx.agentDefaultModel` | 默认配置 Bean |
| `agent-loop` | `core/agent-loop` | 默认驱动（**真正跑循环的地方**） | `ctx.agentLoop` | 主循环引擎 / 状态机 |
| `llm` | `llm/llm` | 消息/流词汇 + 模型适配器 seam | `ctx.llm` | 抽象接口 + SPI 适配器 |

**关键设计**：

- **`agent` 定义契约，`agent-loop` 是它的默认实现**。扩展插件只依赖 `agent` 这个接口，所以主循环（driver）本身也能被替换。Java 对照：`agent` ≈ `interface Agent`，`agent-loop` ≈ `class ReactLoopAgent implements Agent`，消费者只依赖接口。
- 这 7 个核心子包构成「会话 → 提示词 → 工具 → agent → 主循环」这条**默认控制脊柱（control spine）**。

---

## 七、外围能力 seam（模型能调用的工具都来自这里）

核心主干只负责「怎么循环」，模型具体「能做什么」全由外围 seam 提供。每个 seam 都是三角色：**接口（Service Definition）/ 实现（Provider）/ 使用方（Consumer）**：

| seam | 目录 | 大白话 | 关键工具/角色 |
|---|---|---|---|
| `fs` | `fs` | 文件读写（含策略） | 文件工具 |
| `shell` | `shell` | 执行 bash 命令 | 执行器 + 本地实现 |
| `subprocess` | `subprocess` | 子进程 | 定义 + 本地 provider |
| `terminal` | `terminal` | 持久 PTY 终端会话 | 终端工具 |
| `web` | `web` | 搜索 / 抓网页 | 搜索/抓取工具 |
| `lsp` | `lsp` | 语言服务器（代码语义分析） | `lsp` 工具 |
| `skill` | `skill` | 技能注册表 | 技能加载工具 |
| `subagent` | `subagent` | 派生子智能体 | 委托工具 |
| `sandbox` | `sandbox` | 沙箱隔离（bwrap/Landlock/Seatbelt） | 限制 seam |
| `workflow` | `workflow` | 工作流引擎 | worker 线程 + 工具 |

> 关键设计：**换一个 Provider，整个产品能力就变了**。比如把 fs/shell 的 Provider 指向远程沙箱，Bash、终端、LSP 会一起跟着走，不用改任何使用方代码。Java 对照：这就是 **SPI（Service Provider Interface）**——面向接口编程，实现可插拔。

---

## 八、事件三域

流程里的事件分三个域，选对域是大多数改动的第一步：

| 域 | 前缀 | 特性 | 何时用 |
|---|---|---|---|
| **Session 事件** | `turn/*` `step/*` `user/message` `assistant/*` `tool/*` | **持久**事实，追加到日志并广播 | 事实必须在重载后存活 |
| **Agent 事件** | `agent/*` | 携带活 `Agent`：inbox、step、status、request、validation、continuation | 观察/拦截在飞的工作 |
| **Capability 事件** | `fs/*` `tools/*` `telemetry/*` | 给 seam 挂策略和适配器 | 不 import 主循环就能扩展 |

事件分发模式（详见 [CORDIS.zh.md](CORDIS.zh.md)）：

| 事件 | 模式 | 含义 |
|---|---|---|
| `agent/pre-step` | waterfall | 可拒绝/改写 step（要调 `next()` 委托） |
| `agent/request` | waterfall | 可替换调用配置 |
| `agent/request-error` | waterfall | 可返回 `{kind:'retry'}` 接管恢复 |
| `llm/stream` | waterfall | 流式拦截 |
| `tools/pre-execute` `tools/execute` `tools/post-execute` | waterfall | 工具管线拦截 |
| `agent/turn-stopping` | serial | turn 关闭前最后拦截（无 `next()`） |
| `agent/status` `agent/created` `agent/disposed` 等 | emit | 广播通知 |

---

## 九、关键源码位置

| 内容 | 文件 |
|---|---|
| 主循环驱动 `ReactLoopAgent` | `packages/core/agent-loop/src/agent.ts`（`turn()` / `step()` / `preStep()` / `buildRequest()`） |
| 工具并行调度器 | `packages/core/agent-loop/src/tool-calls.ts`（`executeToolCalls()`） |
| `Agent` 接口 / 事件词汇 | `packages/core/agent/src/types.ts`、`runtime-types.ts` |
| 会话事件词汇 `SessionEventMap` | `packages/core/session/src/types.ts` |
| 提示词组装 | `packages/core/system-prompt/src/index.ts` |
| 工具注册表与管线 | `packages/core/tools/src/index.ts` |
| 架构总览（权威流转图） | `docs/architecture.md` 的「Turn flow」一节 |
| 核心子包契约 | `docs/subsystems/core.md` |

---

## 十、术语速查（Java ↔ dsh 对照）

| dsh 概念 | 含义 | Java 概念 |
|---|---|---|
| step | 一次模型请求 + 其工具调用 | 一次服务调用 |
| turn | 0+ 个 step 的边界 | 一个请求生命周期 |
| session log | 追加式唯一真相日志 | Event Sourcing 事件流 |
| `deriveMessages()` | 从日志推导模型历史 | 事件重放（replay） |
| inbox | 双队列待处理消息 | `BlockingQueue` + 分组 |
| `ctx.xxx` | 容器里的服务 | Bean 名（`getBean`） |
| `agent/pre-step` 等 waterfall | 可短路中间件链 | Servlet Filter 链 |
| PreStepDecision | 拒绝/进入 step | 拦截器返回值 |
| capability seam | 接口/实现/使用方三角色 | SPI 机制 |
| Branded ID | 编译期区分不同 ID | 强类型 wrapper |

---

## 十一、推荐阅读路径

1. [CORDIS.zh.md](CORDIS.zh.md) — 底层插件框架（先读这个，流程里的 `ctx`、事件都建立在它之上）
2. [MODULES.zh.md](MODULES.zh.md) — 五大类模块全景
3. `docs/architecture.md` — 权威事件流转图（Turn flow 一节）
4. `docs/subsystems/core.md` — `agent` / `agent-loop` 契约
5. `docs/subsystems/tools.md` — 工具管线与 `ToolDefinition`
6. `docs/subsystems/session.md` — 会话事件词汇

---

**一句话总结**：`agent-loop` 是发动机（跑 turn/step 循环），`session` 是行车记录仪（唯一真相，历史靠 `deriveMessages` 推导），`system-prompt` + `tools` + `llm` 是发动机的燃料与零件，`agent` 是方向盘接口，外围 seam 是可随时替换的功能模块——**全部通过 `ctx` 容器注入，全部是插件，靠类型化事件串成一条「输入 → 循环 → 反馈」的流水线**。
