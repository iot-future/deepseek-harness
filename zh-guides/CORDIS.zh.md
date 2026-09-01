# Cordis 框架中文介绍

> 面向 Java 工程师：先讲「Cordis 解决什么问题」，再用 Java 概念逐条对照。

## 一、Cordis 是什么

**Cordis 是 DeepSeek Harness（`dsh`）底层的插件框架**，一个很小的运行时。它被以 vendor 方式（源码内嵌）引入仓库，并被重命名为 `@deepseek-ai/cordis`。

在 dsh 里，**每一块能力——工具、大模型适配器、文件访问、甚至 agent loop 本身——都是一个插件**，挂载进一个共享的「上下文」。Cordis 就是支撑这套「一切皆插件」架构的骨架。

> Java 对照：Cordis 之于 dsh，就像 **Spring 内核（IoC 容器 + 事件机制）** 之于一个 Spring Boot 应用。它不干业务，只负责「把这些组件组装起来、让它们互相找到、互相通信、并在不用时干净地拆除」。

Cordis 的官方设计论文是 [_A Programming Paradigm for Spatiotemporal Composability_](https://github.com/cordiverse/paper)（一种「时空可组合」的编程范式）。

---

## 二、五个核心概念

理解这五点，Cordis 就通了：

| # | 概念 | 大白话 | Java 对照 |
|---|---|---|---|
| 1 | **插件（Plugin/Service）** | 一个能干活的功能单元 | `@Component` / 一个 Bean |
| 2 | **上下文（Context）** | 一个「服务仓库」，所有服务挂在这里 | `ApplicationContext` |
| 3 | **依赖注入（`inject`）** | 插件声明「我需要哪些服务」，框架按依赖决定启动顺序 | `@Autowired` |
| 4 | **类型化事件（Typed Events）** | 插件之间靠「发事件」通信 | Spring Event / 观察者模式 |
| 5 | **可逆副作用（Reversible effects）** | 注册的东西能「干净地撤销」，热重载/卸载不残留 | `try-with-resources` / `@PreDestroy` |

### 生活比喻：一家饭店

> **饭店**（Context）里有各种**员工**（插件）——点菜的、做菜的、记账的。员工上岗时在**登记表**上注册自己的岗位（`ctx.effect()`），并写明「我上岗前，厨师必须先就位」（`inject`）。大堂经理不直接认识每个员工，只喊「会做菜的来一个」（`ctx.<key>` 按名字找服务）。员工之间靠**传话**（事件）协调。打烊时，所有登记过的岗位自动**撤销**（可逆副作用），不留烂摊子。

### 关键推论

**没有需要「打补丁」的核心代码。** 扩展 dsh 的方式是「写一个新插件，挂到别人旁边」，而不是改它的内核。这在 Cordis 里叫 **时空可组合性**：插件可以在任意层级被叠加（compose）、替换（replace）、卸载（unload），且卸载时会把自己贡献的一切都撤销掉。

---

## 三、三种插件形态

Cordis 接受三种写法，最常用的是第一种：

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

// 1. 函数插件：最常用。loader 挂载时调用 apply(ctx)。
export function apply(ctx: Context) {
  console.log('hello from my first plugin')
}

// 2. 对象插件：带 apply 方法的对象。
export const objectPlugin = {
  name: 'object-plugin',
  apply(ctx: Context) {},
}

// 3. 类插件：Service 子类，用于「对外暴露一个服务」（见下文）。
export class MyService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'myTutorialService')
  }
}
```

> Java 对照：函数插件 ≈ 一个 `@Configuration` 里的 `@Bean` 方法；类插件（`Service` 子类）≈ 一个 `@Service` 类。

---

## 四、上下文 Context：服务仓库

`ctx` 是 Cordis 的核心对象，一切服务、事件、生命周期 API 都从它进入。它的定位就是**服务容器**：

- 一个服务占据一个稳定的名字，如 `ctx.tools`、`ctx.llm`、`ctx.sessions`。
- 其他插件**通过 key 查找服务**（`ctx.get('tools')`），而不是 `import` 具体实现——这样配置里就能换 provider 而不必改消费者。

### 核心 API

| API | 作用 | Java 对照 |
|---|---|---|
| `ctx.provide(name, value)` | 注册一个服务实现（当前 fiber 拥有） | `context.registerBean()` |
| `ctx.get(name)` | 读一个服务（无 `inject` 要求） | `context.getBean()` |
| `ctx.plugin(Plugin)` | 挂载一个插件 | 动态注册 Bean |
| `ctx.effect(disposer)` | 注册一个「可撤销的副作用」 | `try-with-resources` |
| `ctx.on(name, listener)` | 注册一个事件监听器 | `@EventListener` |
| `ctx.extend(meta?)` | 创建带额外元数据的子上下文 | 子容器 |
| `ctx.isolate(name, label?)` | 为某个服务创建独立作用域的子上下文 | 作用域隔离 |
| `ctx.intercept(name, config)` | 给某服务的 config 叠加拦截配置 | 配置覆盖 |
| `ctx.root` | 应用根上下文 | 根容器 |

`ctx` 本身是一个 **proxy**：普通属性读取走服务解析器，而 `extend()` / `isolate()` / `intercept()` 创建子上下文时**不会改动父上下文**。

---

## 五、事件：插件间通信

插件之间不用互相 import，而是靠**类型化事件**通信。事件名通过 TypeScript **声明合并**（`declare module`）注册到 `Context` 接口上。

### 五种分发模式

Cordis 的事件有五种派发方式，这是它区别于 Spring 事件的关键——**事件的「怎么发」是事件公开契约的一部分**：

| 模式 | 方法 | 是否 await | 分发顺序 | 有无返回值 | 用途 |
|---|---|---|---|---|---|
| `emit` | `ctx.emit()` | 否 | 按注册顺序观察 | 无 | 广播通知（fire-and-forget） |
| `parallel` | `ctx.parallel()` | 是 | 全部并行 | 无 | 并行扇出 |
| `serial` | `ctx.serial()` | 是 | 按顺序，直到某个返回非空 | 首个 bail 值 | 顺序执行、可中途 bail |
| `bail` | `ctx.bail()` | 否 | 按顺序，直到某个返回非空 | 首个 bail 值 | 同步版 serial |
| `waterfall` | `ctx.waterfall()` | 否 | 按顺序，中间件链 | 有（最外层返回值） | 环绕拦截、可短路 |

> Java 对照：`emit` ≈ Spring 的 `@EventListener`（广播）；`waterfall` ≈ **Servlet Filter 链 / Spring Security 过滤器链**（可短路）；`serial`/`bail` ≈ 责任链模式（Chain of Responsibility）。

### Waterfall 语义（重点）

`ctx.waterfall` 是**环绕中间件**。监听器接收 `(...args, next)`：

- 调用 `next()` → 执行下游监听器，下游返回值可通过 `next()` 拿到并包装后再返回；
- **不调用 `next()` 直接返回 → 短路**（veto），下游全部跳过。

```ts
// 例：一个策略监听器，拥有决策权时短路
ctx.on('some/waterfall-event', (request, next) => {
  if (我拥有决策权) {
    return 我的决定   // 不调 next()，直接短路
  }
  return next()        // 仅观察/标注，必须委托
})
```

协作式监听器通常修改一个共享的请求/决策对象然后委托；也可以完全替换结果。`prepend: true` 用于「必须在普通注册之前运行」的监听器。

> 在 dsh 里，`agent/pre-step`、`agent/request`、`llm/stream`、`tools/*` 都是 waterfall 事件——这正是「一个插件拦截请求、另一个放行」的机制来源。

---

## 六、依赖注入：`inject`

`inject` 声明一个插件**强依赖**的服务。Cordis 会把这个插件保持在 PENDING 状态，直到所有依赖的服务都就绪，才调用它的 `apply`。因此：

- **加载顺序由服务依赖决定，而不是 `cordis.yml` 里的排列顺序**。
- 依赖在运行期也被追踪：如果某个依赖服务被卸载/热替换，所有依赖它的插件也会被卸载，等服务回来再重新加载——这保证了消费者永远不会持有一个「已失效服务」的引用。

```ts
export const name = 'consumer'
export const inject = ['greeter']      // 声明依赖

export function apply(ctx: Context) {
  console.log(ctx.greeter.greet('world'))   // 此时 ctx.greeter 保证就绪
}
```

**可选依赖**：不要用 `inject`，而是在使用处用 `ctx.get('greeter')` 探测（未加载时返回 `undefined`）。

> Java 对照：`inject: ['greeter']` ≈ 构造器注入 `@Autowired`；`ctx.get('greeter')` ≈ `context.getBean()`（可选依赖）。

---

## 七、生命周期与可逆副作用

每个注册都应该有对应的 **disposer**（释放函数）：

- 从 `ctx.effect(() => disposer)` 返回一个；
- 或用 Cordis 提供的辅助方法（`ctx.on`、`ctx.provide` 等）自动处理。

插件卸载时，Cordis 会按**逆序**撤销它注册的一切。如果 teardown 顺序有要求，就把相关工作放在同一个 `effect` 里。

> Java 对照：`ctx.effect()` ≈ `try-with-resources` / `@PreDestroy` 销毁回调。区别在于 Cordis 把「注册 → 撤销」做成了**一等公民**，任何副作用（服务、监听器、定时器、提示词片段）都能成对地挂上、干净地卸下。

---

## 八、Loader：用 YAML 组装应用

Cordis 应用通常由一个**根 Context + Loader 插件**驱动，从 `cordis.yml` 读取配置来组装插件树：

```yaml
# cordis.yml —— 一份「插件清单」
- name: './greeter.ts'      # name 是模块说明符：相对路径 或 npm 包名
- name: './consumer.ts'
```

- 文件是一个**插件条目列表**，loader 挂载每一个条目。
- 条目**并发启动**，所以列表位置不保证加载顺序（顺序来自 `inject` 依赖）。
- 一个插件**加载失败是 loud failure**（进程报错退出），而不是被静默跳过。
- `dsh` 的 `--dump-config` 可以把实际启动的插件树打印出来，任何一行都能被用户 patch 替换。

---

## 九、与 dsh 的关系

| 层面 | 说明 |
|---|---|
| Cordis | 框架层（插件/上下文/事件/生命周期），vendor 内嵌 |
| dsh 各包 | 基于 Cordis 写出的具体插件（`ctx.tools`、`ctx.llm`、`ctx.agents`…） |
| `cordis.yml` / profile / bundle | 用 YAML 把插件组装成可运行应用的配置层 |

dsh 里的每个能力 seam（Service Definition / Provider / Consumer 三角色）都直接落在这套 Cordis 机制上：**Provider 用 `ctx.provide` 注册服务，Consumer 用 `inject` 依赖它，两者用事件通信**。这正是「换一个 provider 就改变整个产品」的底层原因。

---

## 十、快速上手（keyless，无需 API Key）

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install

mkdir -p tmp/cordis-tutorial && cd tmp/cordis-tutorial
node --import tsx ../../vendor/cordis/bin.js    # 每个示例都用这条命令运行
```

然后写 `hello.ts` 和 `cordis.yml` 即可运行第一个插件（详见 `docs/cordis-tutorial/`）。

---

## 十一、术语速查（Java ↔ Cordis 对照）

| Cordis 术语 | 含义 | Java 概念 |
|---|---|---|
| Plugin / Service | 功能单元 | Bean / `@Component` |
| Context（`ctx`） | 服务仓库 | `ApplicationContext` |
| `inject` | 声明硬依赖 | `@Autowired`（构造器注入） |
| `ctx.get(name)` | 可选依赖探测 | `context.getBean()` |
| `ctx.provide(name, value)` | 注册服务 | `registerBean()` |
| `ctx.effect()` | 可撤销副作用 | `try-with-resources` / `@PreDestroy` |
| `ctx.on(name, fn)` | 注册监听器 | `@EventListener` |
| `emit` / `parallel` | 广播 / 并行 | Spring Event 广播 |
| `waterfall` | 中间件链（可短路） | Servlet Filter 链 |
| `serial` / `bail` | 责任链（可 bail） | Chain of Responsibility |
| 声明合并（declare module） | 给接口加成员 | ——（TS 特有） |
| fiber | 插件实例的生命周期单元 | Bean 实例 + 其销毁回调 |
| disposer | 撤销注册的函数 | 销毁回调 / `AutoCloseable` |

---

## 十二、推荐阅读路径

1. `docs/cordis-primer.md` — 核心概念速览（中文 `cordis-primer.zh.md`）
2. `docs/cordis-tutorial/index.md` — 7 个可运行示例，手把手（中文 `*.zh.md`）
3. `docs/cordis-api/context.md` / `events.md` — 生成的 API 参考
4. `docs/architecture.md` — 如何映射到 dsh 整体架构
5. `vendor/README.md` — vendor 源码清单与同步流程

---

**一句话总结**：Cordis = 一个「一切皆插件」的微型运行时，核心是**上下文（服务仓库）+ 类型化事件（五种派发）+ 可逆副作用 + 依赖注入**。插件通过注册服务、监听事件来协作，通过声明依赖来决定启动顺序，通过可逆副作用来支持热重载与干净卸载——dsh 的整个架构都建立在这套机制之上。
