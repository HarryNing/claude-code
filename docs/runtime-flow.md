# Runtime Flow

## 1. 真正的入口和启动流程

### 入口 1：`src/entrypoints/cli.tsx`

这是整个 CLI 的真实入口。

它负责两件事：

1. 先做非常轻量的早期分流
2. 只有在需要完整 CLI 时，才动态导入 `src/main.tsx`

这个文件里要重点看：

- `main()`
- 对 `--version`、`--daemon-worker`、`remote-control`、`daemon`、`--worktree --tmux` 等 fast path 的处理
- 最后这句：
  - `const { main: cliMain } = await import('../main.jsx')`

也就是说，真正的大部分初始化都延迟到 `src/main.tsx`。

### 入口 2：`src/main.tsx`

这是主调度器，不是简单的 Commander 壳子。

它同时负责：

- 早期环境准备
- 识别是否是交互式会话
- 设置全局 bootstrap 状态
- 构造 Commander 命令和选项
- 执行 `init()`
- 按模式分流到 REPL、headless、resume、remote、teleport 等路径

关键函数：

- `main()`
- `run()`
- `getInputPrompt()`

### 初始化：`src/entrypoints/init.ts`

`init()` 是“一次性系统初始化”。

它做的事情包括：

- 启用配置系统
- 应用安全环境变量
- 注册优雅退出
- 初始化遥测/事件日志
- 初始化远程托管设置和 policy limits 的加载 promise
- 配置 mTLS / proxy / HTTP agent
- 准备 scratchpad 和 LSP cleanup

注意这个原则：

- `init()` 不是启动 UI
- `init()` 是把系统运行环境先搭好

## 2. CLI 模式如何分流

`src/main.tsx` 里真正重要的是模式分流，不是选项定义本身。

### 模式矩阵

| 模式 | 判定位置 | 实际执行入口 |
| --- | --- | --- |
| 普通交互 REPL | 默认路径 | `launchRepl()` -> `src/screens/REPL.tsx` |
| headless / `-p` / SDK | `isNonInteractiveSession` | `src/cli/print.ts::runHeadless` |
| resume / continue | `options.resume` / `options.continue` | `loadConversationForResume()` + `processResumedConversation()` + `launchRepl()` |
| teleport / remote | `options.teleport` / `options.remote` | teleport API + remote session config + `launchRepl()` |
| ssh | 早期 argv 重写 | 最终仍走本地 REPL，但工具会切到远端会话 |
| bridge / remote-control | `src/entrypoints/cli.tsx` fast path | `src/bridge/bridgeMain.ts` |
| daemon | `src/entrypoints/cli.tsx` fast path | `src/daemon/main.ts` |

### 交互式与非交互式的关键差别

#### 交互式

- 走 `launchRepl()`
- 核心组件是 `src/screens/REPL.tsx`
- 直接调用 `query()`
- UI 状态由 `AppState` 驱动

#### 非交互式

- 走 `src/cli/print.ts`
- 创建 `headlessStore`
- 用 `StructuredIO` 收发输入输出
- 通过 `QueryEngine` 驱动会话
- 仍然复用底层 `query()`

一个很重要的事实：

- REPL 当前不是通过 `QueryEngine` 驱动的
- `QueryEngine` 主要服务于 headless / SDK / print 路径

## 3. REPL 是怎么起来的

### 外层挂载

流程非常直：

1. `src/main.tsx` 调 `launchRepl()`
2. `src/replLauncher.tsx` 动态导入 `App` 和 `REPL`
3. `src/components/App.tsx` 包上顶层 Provider
4. `src/state/AppState.tsx` 创建 store
5. `src/screens/REPL.tsx` 成为真正的会话主界面

### 顶层 Provider

`src/components/App.tsx` 主要包 3 层：

- `AppStateProvider`
- `StatsProvider`
- `FpsMetricsProvider`

其中最关键的是 `AppStateProvider`。

### AppState 的职责

`src/state/AppState.tsx` + `src/state/AppStateStore.ts` 负责 UI 可观察状态。

REPL 常读的状态包括：

- `toolPermissionContext`
- `mcp`
- `plugins`
- `agentDefinitions`
- `fileHistory`
- `tasks`
- `notifications`
- `elicitation`
- `expandedView`

而 `src/state/onChangeAppState.ts` 负责把某些状态变化同步到外部系统，比如：

- 权限模式变化
- model 变化
- 设置变化

## 4. 一次用户输入如何进入会话引擎

这是最核心的阅读链路。

### 第 1 步：用户在 `PromptInput` 中提交

`src/screens/REPL.tsx` 把 `onSubmit` 传给 `PromptInput`。

真正重要的不是输入框组件，而是提交后调用的逻辑：

- `onSubmit`
- `handlePromptSubmit()`

### 第 2 步：`handlePromptSubmit()`

文件：

- `src/utils/handlePromptSubmit.ts`

它做 4 件事：

1. 判断当前是立即执行、排队、还是本地 JSX 命令
2. 处理 pasted text/image 引用
3. 处理本地立即命令，如某些 `/config` 类 UI 命令
4. 把普通 prompt 统一转成 `QueuedCommand` 后继续执行

如果当前已经在跑 turn，它不会强行并发开第二个 turn，而是：

- 根据情况中断当前 turn
- 或者把新的输入排队

### 第 3 步：`executeUserInput()`

还是在 `src/utils/handlePromptSubmit.ts`。

这里开始进入“真正的输入执行层”。

关键职责：

- 创建新的 `AbortController`
- 调用 `processUserInput()`
- 收集这一轮产生的新消息 `newMessages`
- 判断 `shouldQuery`
- 最后调用 `onQuery()`

### 第 4 步：`processUserInput()`

文件：

- `src/utils/processUserInput/processUserInput.ts`

这是“把原始输入翻译成系统可处理消息”的核心层。

它会判断输入属于哪一类：

- 普通 prompt
- slash command
- 带附件/图片的 prompt
- hook 拦截后的 prompt

结果是一个 `ProcessUserInputBaseResult`，其中最关键的是：

- `messages`
- `shouldQuery`
- `allowedTools`
- `model`
- `effort`

也就是说，到这里系统才决定：

- 这次输入是只执行本地命令就结束
- 还是要继续进入模型调用循环

### 第 5 步：`onQuery()`

实现位置：

- `src/screens/REPL.tsx`

这里开始把“输入阶段”切换为“模型/工具 turn 阶段”。

它会：

1. 用 `QueryGuard` 防止并发 turn
2. 把新消息塞进当前消息列表
3. 准备 `ToolUseContext`
4. 拉取上下文：
   - `getSystemPrompt()`
   - `getUserContext()`
   - `getSystemContext()`
5. 组合出最终 system prompt
6. 调用 `query()`

### 第 6 步：`query()`

文件：

- `src/query.ts`

这是最底层、最核心的 agent loop。

它内部维护的是“一个 turn 内的循环”，而不是整个会话生命周期。

主要职责：

- 规范化消息
- 注入 user/system context
- 调用 API 流式返回
- 解析 assistant message / thinking / tool_use
- 执行工具
- 处理 stop hooks
- 处理 compact / token budget / fallback / retry
- 把产生的新消息继续喂回下一次循环

你可以把它理解成：

- REPL/SDK 的共同底层 turn machine

## 5. 工具调用在 query 里怎么发生

### 工具定义

核心接口在：

- `src/Tool.ts`

工具注册表在：

- `src/tools.ts`

这里决定有哪些 builtin tools 会进入模型可见的工具池。

### 工具权限

REPL 里常用的权限入口是：

- `src/hooks/useCanUseTool.tsx`

真正的规则判定在：

- `src/utils/permissions/permissions.ts`
- `src/utils/permissions/permissionSetup.ts`

设计原则是：

- 先根据模式、规则、hook、classifier 得到 allow / deny / ask
- 再决定是否弹 UI 或自动拒绝/允许

### 工具执行

底层执行主线：

1. `query.ts`
2. `src/services/tools/toolOrchestration.ts::runTools`
3. `src/services/tools/StreamingToolExecutor.ts`
4. `src/services/tools/toolExecution.ts::runToolUse`

这里有两个很关键的设计：

1. 工具不是一把梭串行执行
   - 并发安全工具可以并行
   - 非并发安全工具会独占执行
2. 工具结果仍然回到消息流
   - 最终会形成 `tool_result` 相关消息
   - 然后继续下一次模型循环

## 6. headless / SDK 路径怎么跑

### 总入口

文件：

- `src/cli/print.ts`

`src/main.tsx` 在非交互模式下会：

1. 创建 `headlessStore`
2. 初始化 MCP
3. 导入 `runHeadless()`

### `runHeadless()` 的核心职责

- 建立 `StructuredIO`
- 生成 `canUseTool`
- 初始化会话
- 调用 `runHeadlessStreaming()`
- 把输出转成 text/json/stream-json

### `QueryEngine`

文件：

- `src/QueryEngine.ts`

它不是最底层 turn loop，而是更上一层的“会话 orchestrator”。

负责：

- 持有 `mutableMessages`
- 管理总 usage / cost / permission denials
- 组装 system init message
- 接收一个个 prompt 并调用 `query()`
- 在 headless 模式下维护 transcript / structured output / result 终态

所以关系是：

- `QueryEngine` 管会话
- `query()` 管单次 agentic turn

## 7. resume / restore / session persistence

这一块主要在 `src/main.tsx` 分流。

关键文件：

- `src/utils/conversationRecovery.js`
- `src/utils/sessionRestore.ts`
- `src/utils/sessionStorage.js`

大致流程：

1. `main.tsx` 判断 `--resume` / `--continue`
2. `loadConversationForResume()` 取出 transcript
3. `processResumedConversation()` 恢复 messages、file history、agent 状态
4. 再把恢复后的结果传给 `launchRepl()`

所以恢复不是“单纯重放 JSON”，而是会恢复：

- 消息
- 文件历史快照
- agent 颜色/名称
- 某些工具输出替换状态

## 8. remote / teleport / CCR 路径

这里不要和 bridge/server 混为一谈。

### remote / teleport

这些路径在 `src/main.tsx` 里是相对真实可读的。

重点文件：

- `src/utils/teleport/api.ts`
- `src/remote/RemoteSessionManager.ts`
- `src/remote/SessionsWebSocket.ts`

大致职责：

- `teleport/api.ts`
  - 调 Claude Code Web / CCR 相关 API
  - 创建/获取远程会话
- `RemoteSessionManager`
  - 本地管理一个远端会话
  - HTTP 发消息，WS 收事件
  - 处理 permission request / response
- `SessionsWebSocket`
  - 管 WS 重连、ping、消息适配

### bridge / server

`src/bridge/bridgeMain.ts` 代码量很大，也相对真实，但：

- 依赖 feature gate
- 不是当前仓库最值得先读的主路径

而 `src/server/*` 这一块目前很多文件直接是 stub，不适合当第一批阅读对象。

## 9. 这套架构背后的核心设计思想

### 设计思想 1：消息优先

所有输入、工具结果、hook 结果、恢复结果都会变成统一消息流。

### 设计思想 2：UI 与引擎解耦，但没有完全分层

- UI 层在 `REPL.tsx`
- 引擎在 `query.ts`
- 但 `REPL.tsx` 仍然直接参与上下文拼装、turn 发起和状态管理

所以它不是一个极度解耦的 clean architecture，而是“工程化很强的实用型架构”。

### 设计思想 3：扩展点都挂在主链路两侧

扩展不是重写主循环，而是接入：

- commands
- tools
- MCP
- skills
- plugins
- hooks

### 设计思想 4：安全系统是主流程的一部分

权限系统不是外围校验，而是工具执行前的必经链路。

## 10. 建议你的二次阅读顺序

如果已经读完本文件，下一步推荐：

1. 先读 `./module-map.md`
2. 然后回源码按这个顺序深读：
   - `src/main.tsx`
   - `src/screens/REPL.tsx`
   - `src/utils/handlePromptSubmit.ts`
   - `src/query.ts`
   - `src/services/api/claude.ts`
   - `src/services/mcp/client.ts`
   - `src/utils/permissions/permissions.ts`
