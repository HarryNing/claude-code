# 02. 运行主链路与会话流转

## 总体心智模型

这个项目的运行链路可以压缩成一句话：

`CLI 入口 -> 初始化 -> 选择运行模式 -> 收集上下文 -> 调模型 -> 执行工具 -> 持久化/恢复会话`

但源码里它被拆成多层，以便同时支持：

- 交互式 REPL
- `--print` / SDK headless
- remote session viewer
- remote-control / bridge
- worktree / swarm / background task

## 第一层：CLI 入口

### `src/entrypoints/cli.tsx`

这是轻量入口层，职责不是承载全部业务，而是做 fast path 分流：

- `--version`
- `--daemon-worker`
- `remote-control`
- `daemon`
- background sessions
- `environment-runner`
- `self-hosted-runner`
- `--worktree --tmux`

如果不走这些 fast path，它最后会动态导入 `src/main.tsx`。

这个设计的核心目的有两个：

- 减少简单命令的启动开销
- 避免不必要地加载整个 React/Ink/commands/module graph

## 第二层：完整 CLI 初始化

### `src/main.tsx`

这是真正的大入口，也是整个仓库最重要的单文件之一。

它主要做五件事：

1. 预热一些昂贵读取
2. 配置 commander 命令行
3. 在 `preAction` 里做统一初始化
4. 解析默认 action 的复杂参数
5. 决定最终进入哪个运行模式

### `preAction` 阶段做什么

`main.tsx` 的 `program.hook('preAction', ...)` 是关键初始化点，里面会做：

- `init()`
- 挂 analytics sinks
- 跑 migrations
- 拉 remote managed settings
- 拉 policy limits
- 启动 settings sync

也就是说，绝大多数“所有命令都共享的全局初始化”都在这里完成。

## 第三层：setup

### `src/setup.ts`

`setup()` 是进入实际会话前的环境准备层。它不负责 query 本身，但负责把会话运行环境准备好。

核心职责：

- 绑定/修正 cwd
- 可选创建 worktree
- 启动 UDS messaging
- 捕获 hooks 配置快照
- 初始化 session memory
- 初始化日志和一些环境变量

理解它的关键点：

- `main.tsx` 负责“我应该跑什么模式”
- `setup.ts` 负责“在这个模式里，工作目录和会话环境应该长什么样”

## 第四层：运行模式分流

### 模式一：交互式 REPL

入口：

- `src/main.tsx`
- `src/replLauncher.tsx`
- `src/screens/REPL.tsx`

这一条路径会：

- 创建 `AppState`
- 挂载 `App + REPL`
- 初始化 hooks、notifications、MCP 连接、bridge、remote session、快捷键、prompt suggestion 等

### 模式二：headless / `--print`

入口：

- `src/main.tsx`
- `src/cli/print.ts`

这一条路径不走 Ink UI，而是走 `runHeadless()` / `runHeadlessStreaming()`。

它会：

- 创建一个 headless 版 `AppState`
- 在必要时连接 MCP
- 组装 tools/commands
- 通过 `ask()`/`QueryEngine` 驱动完整 agent 循环
- 把结果输出成 text/json/stream-json

### 模式三：remote session viewer

入口：

- `src/main.tsx` 中的 `--remote` / `claude assistant`
- `src/remote/RemoteSessionManager.ts`
- `src/hooks/useRemoteSession.ts`

本地 CLI 主要扮演 viewer/client：

- WebSocket 收流式消息
- HTTP POST 发用户消息
- 本地只渲染和代理权限决策
- 真正的 agent 执行在远端

### 模式四：remote-control / bridge

入口：

- REPL 内启用：`src/bridge/initReplBridge.ts`
- 独立 worker/standalone：`src/bridge/bridgeMain.ts`

这条路径的目的不是“连接到远端执行”，而是“把本地环境变成远端可操控的执行环境”。

详细见 `03-remote-bridge-architecture.md`。

## REPL 中，一次输入如何流动

下面这条链路最值得理解：

1. `src/screens/REPL.tsx`
2. `src/utils/handlePromptSubmit.ts`
3. `src/utils/processUserInput/processUserInput.ts`
4. `src/query.ts`
5. `src/services/tools/*`

### 1. REPL 收到输入

`REPL.tsx` 负责：

- 输入框状态
- 发送时机
- 正在处理时的 UI 反馈
- 与命令队列、桥接消息、远程消息的整合

### 2. `handlePromptSubmit`

这里开始把 UI 行为变成会话行为：

- 读取当前上下文
- 构造 `ToolUseContext`
- 调 `processUserInput`
- 决定是否真正进入 query

### 3. `processUserInput`

这是“用户输入预处理层”，不是“调模型层”。

它会先处理：

- slash command
- pasted content
- 图片输入
- hook
- 本地 JSX 命令
- 是否需要立即返回而不是调模型

所以并不是所有用户输入都会直接进入 `query()`。

### 4. 进入 `query()`

`src/query.ts` 是单轮 query 的状态机核心。

它做的事情可以概括成：

- 规范化消息
- 拼装系统 prompt / user context / system context
- 发起模型流式请求
- 消费 assistant response
- 识别 tool_use
- 执行工具
- 把 tool_result 重新喂回消息链
- 视情况继续下一轮

### 5. 工具执行

真正的工具执行核心在：

- `src/services/tools/toolExecution.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/services/tools/StreamingToolExecutor.ts`

职责分工：

- `toolExecution.ts`：单个 tool use 怎么执行
- `toolOrchestration.ts`：多个 tool use 串行/并行怎么调度
- `StreamingToolExecutor.ts`：边流式接收边启动工具时怎么控制并发和顺序

## Headless 中，一次输入如何流动

和 REPL 共享核心，但外层驱动不同：

1. `src/cli/print.ts::runHeadless`
2. `src/cli/print.ts::runHeadlessStreaming`
3. `src/QueryEngine.ts::ask`
4. `src/QueryEngine.ts::QueryEngine.submitMessage`
5. `src/query.ts`

### `QueryEngine` 的角色

`QueryEngine` 的核心价值是把“单轮 query + 会话状态”封装成一个可复用对象。

它维护的不是纯函数式参数，而是会话级状态：

- `mutableMessages`
- `readFileState`
- `permissionDenials`
- `totalUsage`
- `abortController`

所以你可以把它理解成：

- `query.ts` 更像“单轮执行器”
- `QueryEngine.ts` 更像“多轮会话引擎包装层”

## 模型调用在什么地方

模型请求的关键实现主要在：

- `src/services/api/claude.ts`

最值得知道的函数：

- `queryModelWithStreaming`
- `queryWithModel`
- `queryHaiku`

在主链路里的位置是：

- `query.ts` 决定什么时候请求模型
- `services/api/claude.ts` 决定怎么真正请求模型与处理 provider 差异

## 工具调用在什么地方发生权限判断

关键链路：

1. `useCanUseTool.tsx` 或 headless 的 `hasPermissionsToUseTool`
2. `src/utils/permissions/permissions.ts`
3. `src/utils/permissions/permissionSetup.ts`

注意两层差异：

- `permissionSetup.ts` 更像启动阶段：构建权限上下文
- `permissions.ts` 更像运行阶段：判断某次具体 tool use 是 allow/deny/ask

## 会话如何持久化

核心文件：

- `src/utils/sessionStorage.ts`

这个文件的重要性非常高，因为它不仅是“保存聊天记录”，而是整个 resume/continue/fork 体系的基础。

它负责：

- transcript JSONL 写入
- session metadata
- title / mode / agent / attribution
- subagent transcript
- compact 边界
- worktree/session 关系

## 会话如何恢复

核心文件：

- `src/utils/conversationRecovery.ts`
- `src/utils/sessionRestore.ts`
- `src/main.tsx` 的 continue/resume 分支

恢复不只是“把消息读出来”，还包括：

- 恢复 file history
- 恢复 attribution
- 恢复 todo 状态
- 恢复 agent/model
- 恢复 worktree/session metadata

这也是为什么 resume 相关代码看起来分散，但本质是在做“完整会话恢复”。

## 最值得建立的三个心智模型

### 心智模型一：`main.tsx` 是模式选择器

它不是单纯 CLI 参数解析，而是整个运行模式的编排器。

### 心智模型二：`REPL.tsx` 和 `print.ts` 只是两个外壳

真正的 agent 核心在：

- `QueryEngine.ts`
- `query.ts`
- `services/tools/*`
- `utils/permissions/*`

### 心智模型三：这个项目是“会话引擎”，不是“命令集合”

命令、工具、MCP、remote、bridge、resume 最终都围绕同一个核心对象组织：

- 会话
- 消息链
- 工具上下文
- 权限上下文

## 推荐继续阅读的源码

读完这一篇以后，下一步建议直接看：

1. `src/QueryEngine.ts`
2. `src/query.ts`
3. `src/services/tools/toolExecution.ts`
4. `src/utils/sessionStorage.ts`
