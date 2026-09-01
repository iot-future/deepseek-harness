# DeepSeek Harness 模块全景介绍

> 面向 Java 工程师：先讲「每个模块解决什么问题」，再用 Java 概念逐条对照。
> 配套阅读：[CORDIS.zh.md](CORDIS.zh.md)（底层插件框架）、[QUICKSTART.zh.md](QUICKSTART.zh.md)（快速上手）。

## 一、先看整体：这个项目分几层

DeepSeek Harness（`dsh`）是一个**插件化的智能体运行框架**，核心思想是「一切皆插件」。整仓库从外到内分三层：

```
┌─────────────────────────────────────────────────────────────┐
│  第一层：顶层目录（语言/文档/工具/入口）                        │
│  vendor/  packages/  python/  native/  examples/  docs/      │
│  apps/  scripts/  website/  .agents/                         │
├─────────────────────────────────────────────────────────────┤
│  第二层：packages/ 下的「组」（group，共 49 个）                │
│  core/  llm/  web/  shell/  fs/  session/  sdk/ ...          │
│  每个组内部再细分为多个「包」（package）                        │
├─────────────────────────────────────────────────────────────┤
│  第三层：包（package）                                        │
│  例如 core/ 组内有 session/ tools/ agent/ agent-loop/ 等子包  │
└─────────────────────────────────────────────────────────────┘
```

> Java 对照：顶层目录 ≈ 一个 Maven 多模块工程的外壳；`packages/<group>/<pkg>/` ≈ `com.xxx.<group>.<pkg>` 的包结构；每个组 ≈ 一个 Maven 子模块。

**第一层——顶层目录**：

| 目录 | 大白话作用 | Java 对照 |
|---|---|---|
| `packages/` | **核心本体**，49 个组、约 50 个包 | 源码主模块（`src/main`） |
| `vendor/` | 内嵌的 Cordis 插件框架源码 | 把第三方框架源码 `vendored` 进仓库 |
| `python/` | Python SDK + 内置运行时 | 另一个语言的客户端 SDK |
| `native/` | Node 原生扩展（landlock 沙箱） | JNI / native 库 |
| `examples/` | 可运行的 `cordis.yml` 配置叶子 | 可运行的 Demo 工程 |
| `apps/` | 应用装配（含大量配置与快照） | 组装层 / 部署配置 |
| `docs/` | 架构、目录、cookbook、postmortem | 文档 |
| `.agents/` | Agent 工作流 + 笔记 | 团队约定 |
| `scripts/` | 仓库门禁与生成器 | CI / 构建脚本 |
| `website/` | VitePress 文档站 | 对外网站 |

---

## 二、五大类总览

`packages/` 下的 49 个组，按职责可以归纳为**五大类**。这是理解整个项目的钥匙：

| # | 大类 | 一句话定位 | 类比（Java） |
|---|---|---|---|
| 一 | **核心主干 Core Spine** | 产品 API 骨架，一切插件的公共契约 | 框架核心接口层（`core` 模块） |
| 二 | **能力层 Capabilities** | 智能体「能干什么」，每个都是可替换插件 | 可插拔的功能组件（`@Service` + SPI） |
| 三 | **状态与数据层 State & Data** | 会话、身份、设置的持久化 | 数据访问层（DAO / Repository） |
| 四 | **协作与治理层 Collaboration & Governance** | 人机交互、审批、循环控制 | 安全/流程控制层（Filter / 拦截器） |
| 五 | **接口与支撑层 Interfaces & Tooling** | 对外通信协议 + 基础设施 | RPC 网关 / SDK / 工具链 |

> 记住一条铁律贯穿所有层：**「模型能看到什么 ⟺ 会话日志记录了什么」**（model-visible ⟺ logged）。任何要送给大模型的信息，都必须在会话日志里可重建。这是五类模块协作时的核心约束。

---

## 三、第一类：核心主干 Core Spine

**这是整个产品的「骨架 API」**，其他所有插件都建立在它之上。它只存在于 `core/` 一个组，但内部又细分为 7 个子包：

| 子包 | 职责 | ctx 键 | Java 对照 |
|---|---|---|---|
| `scope/` | 作用域上下文注册原语 | 无（纯库） | 一个底层工具类 |
| `session/` | 事件溯源会话日志 + 内存存储 | `ctx.sessions` | 领域仓储（Repository） |
| `system-prompt/` | 提示词与工具 schema 组装注册表 | `ctx.systemPrompt` | 模板组装器 |
| `tools/` | 作用域工具注册表 + 执行管道 | `ctx.tools` | 工具注册中心（Handler Mapping） |
| `agent/` | Agent 接口、注册表、事件词汇 | `ctx.agents` | 领域接口 + 事件定义 |
| `agent-default-model/` | 部署默认模型选择 | `ctx.agentDefaultModel` | 默认配置 Bean |
| `agent-loop/` | 默认具体 agent 驱动器（主循环） | `ctx.agentLoop` | 主循环引擎（核心实现） |

关键设计点：

- **`agent` 定义契约，`agent-loop` 是它的默认实现**。扩展插件只依赖 `agent` 这个 seam，所以主循环（driver）可以整体替换。Java 对照：`agent` ≈ `interface Agent`，`agent-loop` ≈ `class DefaultAgent implements Agent`，消费者只依赖接口。
- 这 7 个子包共同构成「会话 → 提示词 → 工具 → agent → 主循环」这条**默认控制脊柱（control spine）**。Java 对照：相当于一个框架的 `spring-core`——定义了 `ApplicationContext`、`BeanFactory` 这类最基础的契约，业务都基于它们写。

> 生活比喻：核心主干是**厨房的操作台和灶台**——不生产具体菜，但规定了「菜必须在这个台子上切、在这个灶上炒」，所有厨具（能力插件）都得适配这个台面。

---

## 四、第二类：能力层 Capabilities

**智能体「能干什么」都归这一类。** 它是最大的一类，每个能力都是一个**可替换的插件**。

### 4.1 先理解「能力 seam」三角色

框架里每个能力都由**三个角色**组成，这是理解能力层的钥匙：

| 角色 | 职责 | Java 对照 |
|---|---|---|
| **Service Definition** | 能力的抽象接口（对外契约） | `interface` |
| **Service Provider** | 具体实现（可替换） | `@Service` 实现类 |
| **Consumer** | 使用该能力的插件/工具 | 调用方 |

> 铁律：**扩展插件只依赖 Service Definition，绝不依赖具体 Provider**。所以换一个 Provider（比如把本地文件系统换成沙箱文件系统），所有 Consumer 都不用改。Java 对照：这就是 **SPI（Service Provider Interface）** 机制——面向接口编程，实现可插拔。

### 4.2 能力清单

| 组 | 能力（大白话） | 关键角色 |
|---|---|---|
| `llm/` | 调大模型（抽象服务 + DeepSeek 等 provider 适配器） | 定义/提供/消费 |
| `web/` | 搜索、抓取网页 | seam + provider + 工具 |
| `shell/` | 执行 bash 命令 | 执行器 seam + 本地实现 + 工具 |
| `subprocess/` | 起子进程（含进程树） | 定义 + 本地 provider |
| `terminal/` | 持久 PTY 终端会话 | 定义 + 本地实现 + 工具 |
| `code-runtime/` | 代码执行（worker 线程） | 定义 + provider + Code Mode 消费 |
| `sandbox/` | 进程隔离（bwrap/Landlock/Seatbelt 后端） | 限制 seam + 多后端 |
| `e2b/` | E2B 云端沙箱（POC） | provider（实验性） |
| `fs/` | 文件系统读写（含策略） | seam + 本地实现 + 文件工具 |
| `lsp/` | 语言服务器（代码语义分析） | seam + stdio provider + `lsp` 工具 |
| `skill/` | 技能（provider 注册表 + 目录/加载器） | 注册表 + 本地 provider + 工具 |
| `subagent/` | 子代理调度（委托任务） | 注册表 + 委托工具 |
| `workflow/` | 工作流引擎 | seam + worker 线程 + 工具 |
| `jobs/` | 后台任务（`job_*` 控制工具） | 运行时 + 工具 |
| `compaction/` | 上下文压缩 | 定义 + 基础 provider + 命令消费 |
| `spill/` | 工具结果溢出存储 | 存储 seam + 本地实现 + 策略 |
| `todo/` | 待办（`todo_write` 工具） | 单一工具 |
| `attachment/` | 附件标识/校验/内容寻址存储 | 持久化附件 |
| `mcp/` | 桥接外部 MCP 工具到 `ctx.tools` | MCP 客户端桥 |

> Java 对照：这一整层 ≈ 一个「**能力插件市场**」。`llm` ≈ 数据库驱动接口 + MySQL/Oracle 驱动实现（换 provider 即换数据库）；`fs` ≈ 文件存储抽象 + 本地/云存储实现。每个能力都遵循「接口 + 实现 + 消费者」三段式。

---

## 五、第三类：状态与数据层 State & Data

**负责「记住东西」**：会话历史、身份、设置、凭证、目标、检索。

| 组 | 职责（大白话） | Java 对照 |
|---|---|---|
| `session/` | 持久会话数据平面（JSONL/SQLite 后端、投影、标题、上报） | 会话表 + ORM 层 |
| `session-query/` | 会话检索（语料库、血缘、事件关系、语义过滤、全文搜索） | 查询层 / 搜索引擎 |
| `storage/` | 非会话存储中枢 + 后端 + 领域形式 | 通用存储抽象 |
| `workspace/` | 工作区实体 | 领域实体（Entity） |
| `goal/` | 同会话目标的持久化与生命周期 | 目标状态机 |
| `schedule/` | 会话内定时后续操作 | 定时任务 |
| `feedback/` | 人类反馈 | 反馈记录 |
| `context/` | 模型可见的请求上下文（workspace 指令 + 时间上下文） | 请求上下文对象 |
| `identity/` | 共享匿名身份 | 当前用户上下文 |
| `settings/` | 用户设置 seam + 文件 provider | 配置中心 |
| `credentials/` | 凭证引用（env 优先于 `.env`） | 密钥管理 |

> Java 对照：这一层 ≈ **数据访问层（DAO）+ 配置中心**。`session` 是「会话日志的唯一权威来源」，对应 Java 里的**事件溯源（Event Sourcing）**——不是直接存最终状态，而是存一系列事件，状态由事件重放得出。

> 安全注意：`credentials` 坚持「密钥只在环境变量里」，绝不硬编码进代码或提交到仓库。Java 对照：就像数据库密码只放 `application-*.yml` 的加密配置或环境变量，不进 `src/main`。

---

## 六、第四类：协作与治理层 Collaboration & Governance

**负责人与智能体之间、以及智能体自身的「规矩」**：审批、权限、计划、循环卫生。

| 组 | 职责（大白话） | Java 对照 |
|---|---|---|
| `interaction/` | 批准/交互 seam、权限预设、命令、`ask-user` 工具 | 权限校验 + 审批流 |
| `plan/` | 计划模式（直接进入命令 + 评审退出） | 流程编排状态机 |
| `preset/` | 按会话从 `cordis.yml` 组装 agent | 装配器 / 工厂 |
| `guard/` | 循环卫生守卫（重复调用提醒 + `tools/execute` 截止执行器） | 熔断/限流拦截器 |
| `hooks/` | Claude Code / Codex 钩子桥接 + 线协议库 | 事件钩子 / Webhook |
| `extensions/` | agent 运行时自修改（实时检查插件、模型写的插件挂载/卸载） | 运行时动态类加载 |
| `bundle/` | 可安装的 `dsh --profile` 补丁层 | 可插拔补丁包 |

> Java 对照：这一层 ≈ **Spring Security（鉴权）+ 拦截器 + 生命周期管理**。`guard` 尤其像**熔断器/限流器**——防止智能体陷入死循环（重复调同一个工具）或让某个工具超时失控。

> 生活比喻：这是**餐厅的店规和领班**。`interaction` 是「大额消费要经理签字」，`guard` 是「同一桌不能重复上菜」的防呆机制，`plan` 是「先出方案给客人看、确认后再做」。

---

## 七、第五类：接口与支撑层 Interfaces & Tooling

**负责「怎么被外部调用」和「开发怎么跑起来」**：通信协议、SDK、Web UI、测试与工具。

| 组 | 职责（大白话） | Java 对照 |
|---|---|---|
| `api/` | Remote BFF 装配 + Typert RPC 网关 | BFF / 网关层 |
| `typert/` | 类型图生成、产物加载、运行时注册表 | 代码生成器 |
| `sdk/` | 进程外运行时 SDK（JSON-RPC、TS 客户端、服务器插件） | 客户端 SDK |
| `acp/` | 自动化专用的 ACP 服务器 | 自动化协议端点 |
| `boot/` | 共享 app bin 启动胶水 | `main` 启动类 / `@SpringBootApplication` |
| `host/` | Web GUI 宿主半侧（API 网关 + HTTP 路由） | 后端 Web 层（Controller） |
| `client/` | Web GUI 浏览器半侧（shell、协议、对象服务、slot、`ui-*`） | 前端 SPA |
| `examples/` | 演示组合包（agent-spine + CLI/ACP/JSON-RPC bin） | 示例工程 |
| `test-support/` | 测试基础设施（testkit、不变式、回放、Loader 冒烟） | 测试工具库 |
| `runtime-diagnostics/` | 运行时诊断（不变式校验） | 断言/校验框架 |
| `util/` | 低层零依赖工具（`Branded<B>`、路径、超时、留存） | 通用工具类（`commons`） |

> Java 对照：`sdk` + `api` ≈ 你对外提供的 **RPC 网关 + 客户端 SDK**（类似 Dubbo 的 `api` 模块 + 生成的 stub）；`boot` ≈ `@SpringBootApplication` 启动类；`client` + `host` ≈ 前后端分离里的**前端 SPA + 后端 Controller**；`util` ≈ `commons-lang`。

---

## 八、五大类如何协作：一条请求的旅程

用一个「用户问一句话」的简化流程，串起五大类：

```
用户提问
   │
   ▼
[第五类 boot/api/sdk]  接住外部请求（HTTP / JSON-RPC / ACP）
   │
   ▼
[第一类 core]          主循环 agent-loop 启动，从 session 读历史
   │
   ▼
[第三类 session/context] 组装「模型可见上下文」→ 只有被日志记录的内容才送模型
   │
   ▼
[第二类 llm]           调大模型生成下一步
   │
   ▼
[第二类 tools]         模型决定调工具（fs 读文件 / web 搜索 / shell 执行…）
   │
   ▼
[第四类 guard/interaction] 需要审批？循环是否失控？超时？→ 拦截/放行
   │
   ▼
[第三类 session]       把结果写回会话日志（可重建）
   │
   └── 回到 core 主循环，直到任务完成
```

> 核心结论：**第一类定骨架，第二类提供能力，第三类记住状态，第四类把守规矩，第五类对接外界。** 模型与工具的每一次交互，都被第三类记录、被第一类驱动、被第四类约束。

---

## 九、术语速查（Java ↔ dsh 对照）

| dsh 概念 | 含义 | Java 概念 |
|---|---|---|
| group / package | `packages/<group>/<pkg>/` | Maven 模块 / 包 |
| Service Definition | 能力抽象接口 | `interface` |
| Service Provider | 具体实现 | `@Service` 实现类 |
| Consumer | 能力使用方 | 调用方（依赖接口） |
| capability seam | 能力的三角色分界线 | SPI 机制 |
| ctx key（如 `ctx.tools`） | 服务在容器里的名字 | Bean 名（`getBean("tools")`） |
| session log | 事件溯源会话日志 | Event Sourcing 事件流 |
| model-visible ⟺ logged | 模型可见必已记录 | 审计日志（Audit） |
| agent-loop | 默认主循环驱动 | 核心调度引擎 |
| preset / bundle | 按 YAML 组装插件 | 装配配置 / 补丁包 |

---

## 十、推荐阅读路径

1. [CORDIS.zh.md](CORDIS.zh.md) — 底层插件框架（先读这个，模块都建立在它之上）
2. `docs/architecture.md` — 模块如何映射到整体架构
3. `docs/module-graph.md` — 生成的依赖图（`pnpm run gen-module-graph`）
4. `packages/README.md` — 49 个组的权威一览表
5. `packages/core/README.md` — 核心主干的子包与 ctx key 映射

---

**一句话总结**：DeepSeek Harness 是「插件总线 + 五大类模块」——**核心主干定骨架、能力层提供可插拔能力、状态层记住一切、治理层把守规矩、接口层对接外界**，而这一切都建立在 Cordis「一切皆插件」的机制之上。
