# Claude Code Best Code Reading Guide

## 1. 先建立正确认知

这个仓库不是“从零设计出来的整洁项目”，而是一个对 Anthropic Claude Code 的逆向还原/重建仓库。

这直接带来 4 个阅读原则：

1. 要优先区分“真实主链路”与“占位/stub/兼容代码”。
2. 不要先被文件数量吓住，真正影响运行的核心链路没有那么多。
3. 交互式 REPL 和 `--print`/SDK 共享底层 `query()` 能力，但外层包装并不相同。
4. 这个仓库大量功能以“工具 + 权限 + 消息流”的方式组织，而不是传统 MVC。

当前仓库规模大致是：

- `src/` 约 `2799` 个文件
- `packages/` 约 `21` 个文件
- 顶层最大目录依次是 `utils`、`components`、`tools`、`services`、`commands`

## 2. 你最应该先理解的总原则

### 原则 A：所有能力最终都会回到消息流

无论是普通输入、斜杠命令、恢复会话、工具调用、MCP、子代理，最后都会被整理进 `Message[]`，再交给主循环处理。

核心文件：

- `src/screens/REPL.tsx`
- `src/utils/handlePromptSubmit.ts`
- `src/utils/processUserInput/processUserInput.ts`
- `src/query.ts`

### 原则 B：状态分成两层

这个仓库不是只有一个全局 store。

1. `src/bootstrap/state.ts`
   - 进程级/会话级单例状态
   - 比如 session id、cwd、token 统计、telemetry、全局开关
2. `src/state/AppStateStore.ts` + `src/state/AppState.tsx`
   - UI 和当前会话的可观察状态
   - 比如工具权限上下文、MCP 客户端、任务、提示输入、REPL 展示状态

### 原则 C：工具系统是主引擎的一部分，不是附属功能

这个项目不是“模型回答为主，工具只是补充”，而是“模型回答和工具循环共同组成一次 turn”。

关键文件：

- `src/Tool.ts`
- `src/tools.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/services/tools/StreamingToolExecutor.ts`
- `src/services/tools/toolExecution.ts`
- `src/utils/permissions/*`

### 原则 D：优先读真实入口，不要先钻目录

最快的方式不是先从 `src/` 目录平铺扫，而是顺着入口往下读。

建议顺序：

1. `src/entrypoints/cli.tsx`
2. `src/main.tsx`
3. `src/entrypoints/init.ts`
4. `src/replLauncher.tsx`
5. `src/components/App.tsx`
6. `src/state/AppState.tsx`
7. `src/screens/REPL.tsx`
8. `src/utils/handlePromptSubmit.ts`
9. `src/utils/processUserInput/processUserInput.ts`
10. `src/query.ts`
11. `src/services/api/claude.ts`
12. `src/services/tools/*`
13. `src/tools.ts`
14. `src/commands.ts`
15. `src/context.ts`

## 3. 先看哪条链路

### 如果你只想最快理解“用户输入之后发生了什么”

先读：

1. `src/screens/REPL.tsx`
2. `src/utils/handlePromptSubmit.ts`
3. `src/utils/processUserInput/processUserInput.ts`
4. `src/query.ts`
5. `src/services/tools/toolExecution.ts`

### 如果你想理解 CLI 是怎么分流模式的

先读：

1. `src/entrypoints/cli.tsx`
2. `src/main.tsx`
3. `src/cli/print.ts`

### 如果你想理解扩展能力

先读：

1. `src/services/mcp/config.ts`
2. `src/services/mcp/client.ts`
3. `src/skills/loadSkillsDir.ts`
4. `src/utils/plugins/loadPluginCommands.ts`
5. `src/plugins/builtinPlugins.ts`

### 如果你想理解安全边界

先读：

1. `src/hooks/useCanUseTool.tsx`
2. `src/utils/permissions/permissionSetup.ts`
3. `src/utils/permissions/permissions.ts`
4. `src/tools/BashTool/*`

## 4. 你最容易读偏的地方

### 不要把这些目录当成主阅读对象

- `src/src`
- `src/bootstrap/src`
- `src/cli/src`
- `src/components/src`
- `src/services/src`
- `src/types/src`
- `src/utils/src`
- `src/commands/src`

这些目录里很多文件是自动生成的 type stub，不是主运行实现。

它们的来源可以直接看：

- `scripts/create-type-stubs.mjs`

### 不要高估这些区域的完成度

下面这些区域当前要么明显不完整，要么很多文件是 stub，要么依赖 feature gate：

- `src/server/*`
- `src/assistant/index.ts`
- `src/environment-runner/main.ts`
- `src/self-hosted-runner/main.ts`
- `src/coordinator/*` 的一部分

## 5. 文档导航

继续读下面两份：

- `./runtime-flow.md`
  - 从入口到一次对话 turn 的完整执行链路
- `./module-map.md`
  - 目录职责、重点文件、哪些地方值得深入、哪些地方先跳过
