# 01. 代码地图

## 项目定位

从 `README.md` 可以看出，这个仓库是对 Anthropic Claude Code CLI 的逆向还原/复现项目。工程形态上它是：

- Bun + TypeScript 主仓库
- React + Ink 终端 UI
- 多运行模式 CLI
- 带工具调用循环的 agent 框架
- 带远程会话和 remote-control 能力的客户端

`package.json` 也能确认这一点：

- 构建入口：`src/entrypoints/cli.tsx`
- 运行时：Bun
- 前端层：`react`、`react-reconciler`
- CLI：`@commander-js/extra-typings`
- MCP：`@modelcontextprotocol/sdk`
- 模型与云厂商：Anthropic、Bedrock、Vertex、Foundry

## 顶层目录

### `src/`

绝大多数真正的运行时代码都在这里。

### `packages/`

本地 workspace 原生扩展，主要是 N-API 模块：

- `audio-capture-napi`
- `color-diff-napi`
- `image-processor-napi`
- `modifiers-napi`
- `url-handler-napi`

这些不是主控制流，但提供平台能力。

### `scripts/`

仓库维护脚本，不是产品主逻辑。

### `dist/`

构建产物。

### `docs/`

这次新增的导读文档目录。

## `src/` 里的主模块分层

### 1. 入口与启动

- `src/entrypoints/cli.tsx`
- `src/main.tsx`
- `src/setup.ts`
- `src/entrypoints/*`

职责：

- 解析命令行
- 走 fast path
- 初始化配置、日志、analytics、插件、技能、权限上下文
- 决定进入 REPL、headless、remote、bridge、ssh、server 等哪条路径

### 2. UI 与交互层

- `src/screens/REPL.tsx`
- `src/components/*`
- `src/hooks/*`
- `src/ink/*`
- `src/state/*`

职责：

- 终端 UI 渲染
- 输入、滚动、快捷键、提示框、通知
- AppState 状态管理
- 将用户输入转换成 query 调用

### 3. 会话与 agent 主链路

- `src/QueryEngine.ts`
- `src/query.ts`
- `src/query/*`
- `src/utils/processUserInput/*`
- `src/utils/handlePromptSubmit.ts`

职责：

- 处理一次用户输入
- 构造 system/user context
- 发起模型请求
- 消费流式响应
- 识别并执行工具调用
- 处理 compact、hook、continue、budget

### 4. tools 框架

- `src/Tool.ts`
- `src/tools.ts`
- `src/tools/*`
- `src/services/tools/*`

职责：

- 定义 Tool 接口
- 注册和筛选可用工具
- 执行 tool use
- 处理 tool progress、tool result、tool hooks

### 5. 权限系统

- `src/utils/permissions/*`
- `src/hooks/useCanUseTool.tsx`
- `src/hooks/toolPermission/*`

职责：

- 建立权限上下文
- 匹配 allow/deny/ask 规则
- auto mode / classifier / hook 联合决策
- 触发交互式权限弹窗或非交互回调

### 6. 命令系统

- `src/commands.ts`
- `src/commands/*`

职责：

- 注册 slash commands
- 区分 prompt command、local command、JSX command
- 在 REPL/headless 中为输入预处理和辅助功能提供入口

### 7. MCP 与外部扩展

- `src/services/mcp/*`
- `src/services/oauth/*`
- `src/plugins/*`
- `src/skills/*`

职责：

- 管理 MCP server 配置、连接、工具、资源、命令
- 管理 OAuth 登录流程
- 管理插件与技能

### 8. 远程与会话联动

- `src/remote/*`
- `src/bridge/*`
- `src/server/*`
- `src/ssh/*`
- `src/assistant/*`

职责：

- 远程会话 viewer/client
- remote-control / bridge
- direct connect server/client
- SSH 远程执行
- assistant 模式

### 9. 会话持久化与恢复

- `src/utils/sessionStorage.ts`
- `src/utils/sessionRestore.ts`
- `src/utils/conversationRecovery.ts`
- `src/history.ts`

职责：

- transcript JSONL 持久化
- resume / continue / fork session
- 文件历史、attribution、todo、compact 边界恢复

## 关键抽象对象

### `AppState`

定义在 `src/state/AppStateStore.ts`，是 REPL 的“全局前端状态树”。里面不只是 UI 状态，还包括：

- `toolPermissionContext`
- `mcp`
- `plugins`
- `tasks`
- `replBridge*`
- `remoteConnectionStatus`
- `fileHistory`
- `attribution`

理解这个对象以后，你就更容易看懂 REPL 为什么这么大。

### `ToolPermissionContext`

定义在 `src/Tool.ts`。它描述“当前会话到底允许什么、不允许什么、是否要弹窗、是否是 auto mode、有没有额外工作目录”等。

### `ToolUseContext`

也定义在 `src/Tool.ts`。它是 tools 运行时拿到的执行上下文，里面有：

- 当前工具池
- 当前命令池
- `getAppState` / `setAppState`
- abortController
- 文件读取缓存
- notification/hook/compact 等能力

你可以把它理解成“agent runtime 的 request context”。

### `Message` / `SDKMessage`

- 内部 UI 和会话日志使用 `src/types/message.*`
- 远程、SDK、stream-json 使用 `src/entrypoints/agentSdkTypes.ts`

项目里很多适配器都在干一件事：在内部 `Message` 和外部 `SDKMessage` 之间来回转换。

## 这个仓库最重要的设计原则

### 1. 同一套核心逻辑服务多种运行模式

REPL、headless、remote viewer、bridge 都共用大量底层能力。

### 2. 编译期开关 + 运行期开关并存

两类开关同时存在：

- 编译期：`feature('...')`
- 运行期：GrowthBook、环境变量、settings、policy limits

这决定了很多代码“看起来存在，但当前构建/当前会话未必启用”。

### 3. 重逻辑在 `utils/` 和 `services/`

不要只盯着 `screens/` 或 `tools/`。这个仓库的大量真正业务逻辑在：

- `src/utils/*`
- `src/services/*`

### 4. 会话是第一等公民

这不是一个简单的 CLI 命令集合，而是一个“可恢复、可继续、可远程接管、可持久化”的会话系统。

## 哪些目录要谨慎阅读

### `src/src/*`、`*/src/*`

这些目录大多是自动生成的 type stub 或桥接文件，不是核心运行逻辑。

### 典型 stub 文件

当前明显是占位实现的模块有：

- `src/server/server.ts`
- `src/server/sessionManager.ts`
- `src/server/parseConnectUrl.ts`
- `src/server/connectHeadless.ts`
- `src/daemon/main.ts`
- `src/daemon/workerRegistry.ts`
- `src/ssh/createSSHSession.ts`
- `src/ssh/SSHSessionManager.ts`

阅读时不要把这些当成已完成的实现。

## 推荐按模块的阅读顺序

### 如果你想先懂“程序如何跑起来”

1. `src/entrypoints/cli.tsx`
2. `src/main.tsx`
3. `src/setup.ts`

### 如果你想先懂“用户输入为什么会触发工具调用”

1. `src/screens/REPL.tsx`
2. `src/utils/handlePromptSubmit.ts`
3. `src/utils/processUserInput/processUserInput.ts`
4. `src/QueryEngine.ts`
5. `src/query.ts`
6. `src/services/tools/toolExecution.ts`

### 如果你想先懂“远程/bridge 到底在干什么”

1. `src/bridge/initReplBridge.ts`
2. `src/bridge/replBridge.ts`
3. `src/bridge/remoteBridgeCore.ts`
4. `src/bridge/bridgeMain.ts`
5. `src/remote/RemoteSessionManager.ts`

### 如果你想先懂“为什么它有这么多权限/MCP/策略逻辑”

1. `src/tools.ts`
2. `src/utils/permissions/permissionSetup.ts`
3. `src/utils/permissions/permissions.ts`
4. `src/services/mcp/config.ts`
5. `src/services/mcp/client.ts`
