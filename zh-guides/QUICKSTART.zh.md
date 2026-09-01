# DeepSeek Harness（`dsh`）中文项目介绍

> 面向 Java 工程师的快速上手文档：用大白话讲清「解决什么问题」，再用 Java 概念对照 TS 术语。

## 一、这是什么？一句话讲清

**DeepSeek Harness 是一个「智能体（Agent）运行框架」**，由 DeepSeek AI 开源。它让大语言模型（LLM）不再只是「聊天」，而是能**调用工具、读写文件、执行命令、跑后台任务**，像一个真正能干活的人那样完成复杂任务的「引擎底座」。

> Java 对照：它不是某个具体业务系统，而是类似 **Spring 框架** 这种「基础设施」。你在上面写各种「能力组件」，框架负责把它们组装起来、调度起来。

**最核心的一句话：一切皆插件（Everything is a plugin）。** 整个产品没有任何「特权内核」——模型适配器、工具注册表、会话日志、甚至「智能体循环」本身，全都是插件，都能从配置里替换。

## 二、核心架构：它怎么做到「一切皆插件」？

### 底层框架 Cordis（内嵌的第三方框架）

dsh 跑在一个叫 **Cordis** 的插件框架上。Cordis 的设计思想浓缩成 5 个概念：

| # | 概念 | 大白话 | Java 对照 |
|---|---|---|---|
| 1 | **插件（Plugin/Service）** | 一个能干活的功能单元 | `@Component` / 一个 Bean |
| 2 | **上下文（Context）** | 一个「服务仓库」，所有服务挂在这里 | Spring 的 `ApplicationContext` |
| 3 | **依赖注入（inject）** | 插件声明「我需要哪些服务」，框架按依赖自动排好加载顺序 | `@Autowired` / 构造器注入 |
| 4 | **类型化事件（Typed Events）** | 插件之间靠「发事件」通信，有 4 种派发模式 | Spring Event / 观察者模式 |
| 5 | **可逆副作用（Reversible effects）** | 注册的东西能「干净地撤销」，热重载/卸载不会残留 | `try-with-resources` / 容器销毁回调 |

用生活比喻串起来：

> 想象一个**饭店**（Context）。每个功能——点菜（模型调用）、做菜（工具执行）、记账（会话日志）——都是一位**员工**（插件）。员工上岗时在「登记表」上**注册自己的岗位**（`ctx.effect()` 注册服务），并写明「我上岗需要厨师先就位」（`inject` 声明依赖）。大堂经理不用认识每个员工本人，只要喊「会做菜的来一个」（通过 `ctx.<key>` 按岗位找服务），框架就会把对应员工叫来。员工之间靠**传话**（事件）协调。打烊时，所有登记过的东西自动**撤销**（可逆副作用），不会留下烂摊子。

**关键推论**：没有需要「打补丁」的核心代码。你要扩展 dsh，不是去改它的内核，而是**写一个新插件，挂到别人旁边**。

## 三、一条消息的完整旅程（Turn Flow）

这是理解运行机制最直观的部分。术语先对齐：

- **step（步骤）** = 一次模型请求 + 它触发的工具调用
- **turn（轮次）** = 零个或多个 step，从「接到输入」到「该干的活干完」

```text
turn/start                                    # 一轮开始
  领取下一条输入 + 一条排队中的消息
  组装提示词片段 + 工具 schema
  → agent/pre-step                            # 拦截点：决定「模型能看到什么」
    step/start
    把进入的消息记为 user/message
    从日志推导出模型历史
    agent/request → llm/stream → assistant/chunk* → assistant/message   # 调模型、流式返回
    tool/call* → tools/pre-execute → tools/execute → tools/post-execute → tool/result*  # 模型要调工具，走工具流水线
    step/end
    (如果工具还欠一次请求，或来了新输入) → 领取 → 下一步
  → agent/turn-stopping                       # 拦截点：结束前最后一道关
turn/end                                      # 一轮结束
```

> Java 对照：这条流水线就像 Servlet 的 **Filter 链**。其中 `agent/pre-step`、`agent/request`、`llm/stream`、`tools/*` 这些是 **waterfall（瀑布）事件**——每个监听器必须调用 `next()` 才能把请求传给下一个，否则就是「截胡」（短路）。这跟 Spring Security 的过滤器链 `chain.doFilter()` 一模一样。

## 四、两个贯穿全局的铁律

1. **模型可见 ⟺ 已记录**：凡是要发给模型的内容，必须能从会话日志（Session Log）重建出来。所以新增「让模型看到的新信息」= 必须新增一个会话事件。会话日志是模型上下文的唯一来源（fork 恢复、回放、遥测、持久化都从它派生）。

2. **能力 seam 三件套**：每个「可替换能力」由三个角色构成——
   - **Service Definition**（接口声明，Java 的 `interface`）
   - **Service Provider**（实现，Java 的 `implements` 类）
   - **Consumer**（使用方，通常是暴露给模型的工具）

   这正是「换一个 Provider 就能改变整个产品」的原因：文件系统、Bash、终端、LSP 共享同一个「执行世界」，把它们指向远程沙箱，就一起搬过去了。

## 五、项目目录结构（挑重点）

```text
packages/<group>/<pkg>/   核心代码（按领域分组，约 50 个包）
  core/      产品 API 主干：会话、提示词、工具、agent、agent 循环
  llm/       大模型能力：抽象服务 + DeepSeek 适配器
  shell/     Bash 能力
  fs/        文件系统能力
  lsp/       语言服务器能力
  session/   会话持久化
  subagent/  子智能体能力
  web/       网络能力（搜索/抓取）
  ...        还有 terminal、workflow、todo、plan 等约 40 个
docs/       架构文档、子系统说明、cookbook（实操手册）
examples/   可运行的示例 cordis.yml 配置
native/     C++ 原生沙箱（Landlock）
python/     Python SDK
vendor/     内嵌的 Cordis 源码
scripts/    仓库门禁脚本
```

> Java 对照：整个仓库是一个 **Maven 多模块（reactor）工程**，`packages/` 下的每个包就是一个 module，用 pnpm workspace（相当于 Maven 的 `<modules>`）管理。每个 npm 包统一命名为 `@deepseek-ai/dsh-<name>`。

## 六、上手开发

### 1. 环境准备（等价于装 JDK + Maven）

| 需求 | 说明 |
|---|---|
| Node.js | **22.19+ 或 24+**（相当于「JDK 版本」） |
| pnpm | 包管理器，等价于 Maven/Gradle（仓库固定 `pnpm@11.7.0`，先 `corepack enable`） |
| Git | 2.26+ |
| DeepSeek API Key | 可选，跑真实模型演示/测试才需要 |

### 2. 首次搭建

```sh
pnpm install           # 装依赖（等价 mvn install 拉依赖）
pnpm run typecheck     # 类型检查，成功退出即搭建完成
```

### 3. 跑起来

```sh
# 最简单：直接 npm 启动 Web UI（默认 http://127.0.0.1:3080）
npx @deepseek-ai/dsh web

# 从源码跑
pnpm run build
pnpm dsh web

# 一次性「无界面」跑一个任务（需要 API Key）
pnpm dsh --profile headless "summarize this workspace"
```

### 4. 常用命令速查

| 命令 | 作用 | Java 对照 |
|---|---|---|
| `pnpm run test` | 单元测试 | `mvn test` |
| `pnpm run typecheck` | TS 类型检查 | `mvn compile`（编译期校验） |
| `pnpm run lint` | 代码规范检查 | Checkstyle |
| `pnpm run build` | 构建产物 | `mvn package` |
| `pnpm run doc-sync` | 文档一致性门禁 | —— |
| `pnpm run test:e2e` | 真实 API 测试（无 Key 自动跳过） | 集成测试 |

### 5. 怎么「加一个新功能」（最常用的扩展动作）

dsh 里有 15+ 种标准扩展点。你只需对照下表选一种，不需要改内核：

| 想做什么 | 怎么做 |
|---|---|
| 接一个新模型供应商 | 在 `ctx.llm` 注册它的适配器 |
| 给模型一个新能力（新工具） | 在 `ctx.tools` 注册，schema 自动加入提示词 |
| 加 shell 执行 | 注册 `ctx.shell` 后端 |
| 加后台任务 | 注册 `ctx.jobs` |
| 拦截某个请求/工具/轮次 | 监听对应 `agent/*` 或 `tools/*` 事件 |
| 加持久会话状态 | 扩展 `SessionEventMap` |

**完整新包**的落盘步骤（有官方 checklist 校验）：

```text
packages/<group>/<pkg>/
  package.json     # 从 packages/core/tools 复制，改名字/依赖
  tsconfig.json    # extends 基座配置
  src/index.ts     # 服务默认导出或插件（name/inject/apply/Config）
  README.md        # 服务 API、事件、扩展点、设计说明
```

然后注册到 `tsconfig.host.json`（或 `tsconfig.client.json`）的 `references`，最后跑 `pnpm run constraints && pnpm run typecheck && pnpm run lint` 验证。

## 七、关键术语速查（Java ↔ TS 对照表）

| TS/项目术语 | 含义 | 你熟悉的 Java 概念 |
|---|---|---|
| Plugin / Service | 功能单元 | Bean / `@Component` |
| Context（`ctx`） | 服务仓库 | `ApplicationContext` |
| `inject` | 声明依赖 | `@Autowired` |
| Service Definition | 接口声明 | `interface` |
| Service Provider | 实现 | `implements` 类 |
| Consumer | 调用方 | 使用该接口的业务代码 |
| `ctx.effect()` | 注册可撤销副作用 | `try-with-resources` / `@PreDestroy` |
| waterfall 事件 | 中间件链（可短路） | Servlet Filter 链 |
| `ctx.<key>` | 按岗位找服务 | `ctx.getBean(Class)` |
| SessionEvent | 持久事件日志 | 事件溯源（Event Sourcing）里的 event |
| workspace（pnpm） | 多包管理 | Maven reactor 多模块 |
| `tsconfig.json` | 编译配置 | `pom.xml` / module 配置 |

## 八、推荐阅读路径（由浅入深）

1. `docs/cordis-primer.md` — 先吃透 Cordis 的 5 个概念（中文版 `cordis-primer.zh.md`）
2. `docs/architecture.md` — 整体架构（中文版 `architecture.zh.md`）
3. `docs/development.md` — 开发指南与命令（中文版 `development.zh.md`）
4. `docs/cookbook/adding-a-package.md` — 手把手加一个包（中文版 `adding-a-package.zh.md`）
5. `docs/cookbook/adding-a-tool.md` / `adding-an-llm-adapter.md` — 加工具 / 加模型适配器

---

**一句话总结**：dsh = 一个基于 Cordis 的「插件化智能体引擎」。你通过注册**服务**、监听**事件**、实现**能力 seam** 来扩展它，永远不需要改内核；上手的关键是先吃透「Cordis 五概念」和「turn flow 一条消息的旅程」这两块。
