# Module Map

## 1. 顶层目录地图

按文件量看，最重要的顶层目录大致如下：

| 目录 | 文件数 | 作用 |
| --- | ---: | --- |
| `src/utils` | 693 | 公共基础设施，包含权限、git、settings、telemetry、session、memory、swarm 等 |
| `src/components` | 593 | Ink/React 组件层 |
| `src/tools` | 285 | 内建工具实现 |
| `src/services` | 235 | API、MCP、analytics、oauth、policy、compact 等服务 |
| `src/commands` | 226 | 斜杠命令 |
| `src/hooks` | 147 | React hooks 和交互逻辑 |
| `src/cli` | 127 | headless/SDK/print 相关实现 |
| `src/ink` | 104 | 自定义终端 UI 基础设施 |
| `src/screens` | 51 | 终端屏幕，核心是 REPL |
| `src/bridge` | 34 | remote-control / bridge 相关 |

## 2. 主干目录应该怎么理解

### `src/entrypoints`

职责：

- 进程级入口
- 初始化入口
- SDK 类型入口

重点文件：

- `src/entrypoints/cli.tsx`
- `src/entrypoints/init.ts`

### `src/main.tsx`

不要把它只当作“一个很大的 CLI 文件”。

它实际承担的是：

- 顶层模式调度
- Commander 配置
- session / resume / remote / teleport / print 的切换中心

### `src/screens`

真正重要的是：

- `src/screens/REPL.tsx`

这是主交互会话控制器，不只是 UI 展示层。

### `src/state`

职责：

- 管 REPL / 会话内可观察状态
- 提供 React store 和 selector

重点文件：

- `src/state/AppStateStore.ts`
- `src/state/AppState.tsx`
- `src/state/onChangeAppState.ts`

### `src/bootstrap`

职责：

- 管进程级/会话级单例状态
- 记录计数器、session id、cwd、telemetry 句柄等

重点文件：

- `src/bootstrap/state.ts`

如果你想找“为什么某个值不是从 React props 传下来的”，大概率会在这里找到。

### `src/query`

职责：

- 主 agent loop
- token budget / compact / transitions / deps 抽象

重点文件：

- `src/query.ts`
- `src/query/config.ts`
- `src/query/tokenBudget.ts`
- `src/query/stopHooks.ts`

### `src/services/api`

职责：

- 和 Claude API / provider 对接
- provider 差异封装
- retry / fallback / usage 统计

重点文件：

- `src/services/api/claude.ts`
- `src/services/api/errors.ts`
- `src/services/api/withRetry.ts`

### `src/tools`

职责：

- 所有 builtin tool 的定义和执行逻辑

你应该先看：

- `src/tools.ts`
- `src/Tool.ts`

然后再按优先级看：

1. `BashTool`
2. `FileReadTool`
3. `FileEditTool`
4. `FileWriteTool`
5. `AgentTool`
6. `MCPTool`
7. `TodoWriteTool`

### `src/commands`

职责：

- `/xxx` 命令定义
- 一部分命令只生成 prompt
- 一部分命令是本地执行
- 一部分命令会弹本地 JSX UI

中心文件：

- `src/commands.ts`

### `src/context`

职责：

- 为模型生成会话上下文

核心内容：

- git status
- CLAUDE.md / memory
- current date

重点文件：

- `src/context.ts`
- `src/utils/claudemd.ts`

## 3. 服务层怎么分

### MCP

这是扩展体系里最重要的一层之一。

职责：

- 解析配置
- 连接 stdio / sse / http / ws / sdk 型 MCP server
- 把 MCP server 转成工具、命令、资源

重点文件：

- `src/services/mcp/config.ts`
- `src/services/mcp/client.ts`
- `src/services/mcp/MCPConnectionManager.tsx`

你可以这样理解：

- `config.ts` 负责“有哪些 server”
- `client.ts` 负责“怎么连、连上后变成什么”
- `MCPConnectionManager.tsx` 负责在 REPL 中把连接状态接到 UI/store

### OAuth

重点文件：

- `src/services/oauth/index.ts`
- `src/services/oauth/client.ts`
- `src/services/oauth/auth-code-listener.ts`

职责：

- OAuth PKCE 登录流程
- 自动浏览器跳转 + 本地回调端口
- profile / token 信息拉取

### Policy Limits

重点文件：

- `src/services/policyLimits/index.ts`

职责：

- 拉取组织级策略限制
- 缓存到本地
- 运行时决定某些功能是否允许

### Analytics / GrowthBook

重点文件：

- `src/services/analytics/index.ts`
- `src/services/analytics/growthbook.ts`

职责：

- 事件 sink
- feature gate
- 远程配置和实验开关

### Skills / Plugins

重点文件：

- `src/skills/loadSkillsDir.ts`
- `src/plugins/builtinPlugins.ts`
- `src/utils/plugins/loadPluginCommands.ts`

设计原则：

- skills 本质上会被转成 command
- plugin 可以提供 skill / hooks / MCP server
- 这些扩展不会重写主循环，只是注入更多命令/工具/配置

### Swarm / Teammate

重点文件：

- `src/utils/swarm/inProcessRunner.ts`
- `src/utils/swarm/backends/registry.ts`
- `src/tasks/InProcessTeammateTask/*`
- `src/tools/AgentTool/*`

职责：

- 子代理 / teammate 执行
- 进程内 runner
- tmux / iTerm2 / in-process backend 选择
- leader 与 worker 之间的权限同步

## 4. 哪些目录需要晚点读

### `src/bridge`

这一块很重要，但不是第一轮最值得读的。

原因：

- 功能依赖更多上下文
- 代码量大
- 偏 remote-control 环境

如果你以后要读它，建议先从这些文件切入：

- `src/bridge/bridgeMain.ts`
- `src/bridge/sessionRunner.ts`
- `src/bridge/replBridge.ts`

### `src/remote`

这是“连接远端会话”的更轻量一层，比 `bridge` 更容易读。

先看：

- `src/remote/RemoteSessionManager.ts`
- `src/remote/SessionsWebSocket.ts`

### `src/server`

当前不建议优先深入。

原因很直接：

- 多个关键文件是 stub

例如：

- `src/server/server.ts`
- `src/server/sessionManager.ts`
- `src/server/connectHeadless.ts`
- `src/server/parseConnectUrl.ts`

## 5. 这个仓库里的“伪目录”问题

下面这些目录里的很多文件不是你要读的核心实现：

- `src/src`
- `src/bootstrap/src`
- `src/cli/src`
- `src/commands/src`
- `src/components/src`
- `src/services/src`
- `src/types/src`
- `src/utils/src`

它们大多来自自动生成 stub 的过程，直接证据是：

- `scripts/create-type-stubs.mjs`

这个脚本会根据 TypeScript 错误生成占位模块，目的是让工程先能继续运行/构建，而不是补齐真实实现。

所以在阅读时要形成一个习惯：

- 优先看没有 `/src/` 镜像嵌套的真实文件
- 看到 `Auto-generated stub` 注释时立刻降权

## 6. packages 目录该怎么看

`packages/` 不是一个完整独立子系统群，更像运行时替身和少量移植实现。

### 值得知道的点

- `packages/color-diff-napi/src/index.ts`
  - 不是 stub
  - 是对原生模块的 TypeScript 替代实现
- `packages/audio-capture-napi/src/index.ts`
  - 用系统命令替代原生录音能力
- `packages/@ant/computer-use-mcp/src/index.ts`
  - 明确是 stub
  - 注释里已经说明 feature 关闭时仅用于保持 import 不报错

结论：

- `packages/` 不是第一优先级
- 只在你追某个具体能力时再回头看

## 7. 一张“定位问题时去哪找”的索引

### 想找启动流程

- `src/entrypoints/cli.tsx`
- `src/main.tsx`
- `src/entrypoints/init.ts`

### 想找 UI 和输入

- `src/screens/REPL.tsx`
- `src/components/App.tsx`
- `src/components/PromptInput/*`
- `src/state/AppState.tsx`

### 想找 prompt 如何变成消息

- `src/utils/handlePromptSubmit.ts`
- `src/utils/processUserInput/processUserInput.ts`
- `src/utils/processUserInput/processTextPrompt.ts`

### 想找模型调用循环

- `src/query.ts`
- `src/services/api/claude.ts`

### 想找工具执行

- `src/Tool.ts`
- `src/tools.ts`
- `src/services/tools/toolExecution.ts`
- `src/services/tools/toolOrchestration.ts`

### 想找权限判断

- `src/hooks/useCanUseTool.tsx`
- `src/utils/permissions/permissionSetup.ts`
- `src/utils/permissions/permissions.ts`

### 想找命令系统

- `src/commands.ts`
- `src/commands/*`

### 想找会话恢复/持久化

- `src/utils/sessionStorage.js`
- `src/utils/sessionRestore.ts`
- `src/utils/conversationRecovery.js`

### 想找 MCP

- `src/services/mcp/config.ts`
- `src/services/mcp/client.ts`
- `src/tools/MCPTool/*`

### 想找远程/CCR

- `src/utils/teleport/api.ts`
- `src/remote/RemoteSessionManager.ts`
- `src/remote/SessionsWebSocket.ts`

## 8. 最后给你的阅读策略

如果你想尽快吃透这个仓库，最省时间的方法是：

1. 先只跟主链路，不按目录扫
2. 主链路吃透后，再补权限、MCP、skills/plugins
3. 最后再看 bridge、swarm、remote 等高级能力

对这个仓库来说，最有效的阅读心智模型是：

- `main.tsx` 负责“模式选择”
- `REPL.tsx` 负责“交互控制”
- `handlePromptSubmit/processUserInput` 负责“输入翻译”
- `query.ts` 负责“主循环”
- `toolExecution` 负责“动作落地”
- `services/*` 负责“把外围能力接进主循环”

