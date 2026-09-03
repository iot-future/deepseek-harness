# CodeBuddy 项目协作说明

本文件补充根目录 `AGENTS.md`；发生冲突时，以 `AGENTS.md` 和目标目录中的更具体说明为准。

## 开始工作前

- 修改 `packages/` 前，先阅读 `docs/architecture.md`；遵循 capability seam、事件日志、投影与可逆 effect 的约束。
- 用 `pnpm` 管理 workspace，保持 ESM、严格 TypeScript 和最小必要改动。
- 不直接修改生成内容；先定位生成器和对应的权威源文件。

## 文档

- `docs/` 是正式双语文档树；新增或修改时遵循 `docs/AGENTS.md` 和双语 / i18n 门禁。
- `zh-guides/` 存放独立中文技术导读，不作为 `docs/` 的双语镜像。
- Agent Teams 的运行时导读位于 `zh-guides/AGENT-TEAMS-RUNTIME.zh.md`；修改实现时同步核对该文档与 `docs/subsystems/agent-team.zh.md` 的事实描述。

## 验证

- 按改动表面选择最小相关检查；不要默认执行完整测试套件。
- 提交前保持文件末尾恰好一个换行，并检查改动未混入凭证、构建产物或临时文件。
