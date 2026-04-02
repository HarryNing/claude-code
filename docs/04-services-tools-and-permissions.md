# 04. Services / Tools / Permissions 基础设施

这一部分是整个项目的“发动机舱”。很多复杂度不在 UI，而在这些基础设施里。

## 一、Tool 系统的组织方式

## 1.1 Tool 抽象

核心定义在：

- `src/Tool.ts`

这里定义了几个最重要的运行时抽象：

- `Tool`
- `Tools`
- `ToolPermissionContext`
- `ToolUseContext`

最重要的认识是：

- Tool 不只是一个函数
- 它是“带 schema、描述、权限需求、渲染能力、执行能力的运行时对象”

## 1.2 Tool 注册中心

核心文件：

- `src/tools.ts`

这个文件做三件事：

1. 定义 built-in tools 的全集 `getAllBaseTools()`
2. 根据 feature gate / env / settings / permission context 过滤工具
3. 组合 built-in tools 和 MCP tools

最值得看的函数：

- `getAllBaseTools`
- `getTools`
- `assembleToolPool`
- `filterToolsByDenyRules`

关键原则：

- “工具是否存在”不等于“这次会话能不能用”
- 真正暴露给模型的工具集是运行时拼出来的

## 1.3 Tool 执行链

核心文件：

- `src/services/tools/toolExecution.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/services/tools/StreamingToolExecutor.ts`

职责分工：

- `toolExecution.ts`：执行单个 tool use
- `toolOrchestration.ts`：批量调度 tool use
- `StreamingToolExecutor.ts`：在流式 assistant 响应还没结束时启动工具

### 单个 tool use 的核心步骤

`toolExecution.ts` 大致做下面这些事：

1. 找到具体 tool 定义
2. 校验 input schema
3. 跑 pre-tool hooks
4. 做权限检查
5. 调 `tool.call(...)`
6. 处理 progress / result / error
7. 跑 post-tool hooks
8. 把结果包装成 `tool_result` user message

这就是整个 agent 工具调用闭环的中心。

## 二、权限系统怎么工作

## 2.1 启动阶段：构建权限上下文

核心文件：

- `src/utils/permissions/permissionSetup.ts`

它负责：

- 读取 settings / CLI / session 级规则
- 解析 allow/deny/ask 规则
- 构建 `ToolPermissionContext`
- 处理 auto mode / bypass / plan mode 的初始化

你可以把它理解成“编译当前会话的权限配置”。

## 2.2 运行阶段：判断某次 tool use

核心文件：

- `src/utils/permissions/permissions.ts`

它负责真正回答：

- 这次调用是 allow、deny 还是 ask
- deny/ask 的原因是什么
- 是否命中规则、hook、classifier、mode 限制

最重要的函数：

- `hasPermissionsToUseTool`

### 决策来源不是单一的

一次权限结果可能来自：

- 显式 allow/deny/ask 规则
- 当前 permission mode
- classifier
- hook
- sandbox override
- working directory 校验

所以权限系统不是一个简单 if/else，而是多来源合并。

## 2.3 交互层：`useCanUseTool`

核心文件：

- `src/hooks/useCanUseTool.tsx`

它的作用是把底层权限结果接入 UI：

- `allow` 直接继续
- `deny` 记录并返回
- `ask` 时创建交互式确认流程

这个 hook 会把：

- 权限检查
- UI 队列
- classifier 结果
- coordinator / swarm 特殊处理

全部接到一起。

所以它是 REPL 中“权限系统真正落地”的桥接层。

## 三、MCP 是怎么接入主流程的

## 3.1 配置层

核心文件：

- `src/services/mcp/config.ts`

负责：

- 解析 `.mcp.json`、settings、CLI `--mcp-config`
- 合并不同来源的 server config
- 做企业策略过滤
- 处理 enable/disable 状态

## 3.2 客户端连接层

核心文件：

- `src/services/mcp/client.ts`

这是 MCP 体系最重的文件之一。它负责：

- 连接 stdio / SSE / streamable HTTP / SDK transport
- 获取 tools / commands / resources
- 调用 MCP tool
- 处理 MCP auth
- 处理 elicitation
- 管理连接缓存与重连

理解 MCP 时最重要的一点是：

- MCP tools 不是在 `src/tools/` 里静态写死的
- 它们是运行时从外部 server 动态拉进来的

## 3.3 React 接入层

核心文件：

- `src/services/mcp/useManageMCPConnections.ts`
- `src/services/mcp/MCPConnectionManager.tsx`

它们负责把 MCP 连接状态写进 `AppState.mcp`：

- `clients`
- `tools`
- `commands`
- `resources`

REPL 和部分 headless 逻辑再从这个状态池里动态获取可用能力。

## 3.4 MCP 对主流程的影响

MCP 不只是额外工具，还会影响：

- slash commands
- resources
- channels / notification
- auth flow
- permission relay

所以它更像一个“扩展子平台”，不是单一插件点。

## 四、OAuth / Analytics / Policy 这几个横切服务

## 4.1 OAuth

核心文件：

- `src/services/oauth/index.ts`
- `src/services/oauth/client.ts`

`OAuthService` 的职责很清晰：

- 启动 auth code + PKCE 流程
- 启本地 listener 接收回调
- 支持浏览器自动回调和手动粘贴 code
- 交换 token 并补 profile 信息

这层被 bridge、remote、MCP auth、普通 API 调用共享。

## 4.2 Analytics

核心文件：

- `src/services/analytics/index.ts`
- `src/services/analytics/sink.ts`
- `src/services/analytics/growthbook.ts`

设计上很值得注意：

- `analytics/index.ts` 故意几乎无依赖
- 先把事件排队
- sink 挂上后再统一发出去

这样可以避免启动期的 import cycle 和早期事件丢失。

GrowthBook 在这个仓库里不只是实验系统，还承担运行时能力开关的角色。

## 4.3 Policy Limits

核心文件：

- `src/services/policyLimits/index.ts`

这个模块是“组织级能力限制”的统一入口，比如：

- 是否允许 remote sessions
- 是否允许 remote control

特点：

- fail-open
- 有本地缓存
- 后台轮询更新

它直接影响 `main.tsx` 里很多模式是否可进入。

## 五、Compaction、PromptSuggestion、Session Memory 这些“次核心”

## 5.1 Compaction

核心目录：

- `src/services/compact/*`

作用：

- 控制上下文长度
- 自动 compact
- microcompact
- snip compact
- reactive compact

`query.ts` 会直接调用这些能力，所以这不是边缘功能，而是长会话稳定运行的关键。

## 5.2 Prompt Suggestion

核心目录：

- `src/services/PromptSuggestion/*`

作用：

- 基于当前上下文预测用户可能的下一步输入
- 在 REPL 里作为交互增强

这部分不是主执行链路，但深度接入了 `AppState` 和 REPL 生命周期。

## 5.3 Session Memory / Magic Docs / Team Memory

代表目录：

- `src/services/SessionMemory/*`
- `src/services/MagicDocs/*`
- `src/services/teamMemorySync/*`

这些模块体现的是“围绕会话做长期辅助”的能力，而不是当轮 agent loop 的主干。

## 六、Swarm / Team / 子代理基础设施

主要在：

- `src/utils/swarm/*`
- `src/tasks/*`
- `src/tools/AgentTool/*`

这层的作用是让“一个 session 可以派生多个执行单元”。

重点不是 UI，而是这些基础设施：

- teammate backend 选择
- permission sync
- in-process runner
- leader/worker 协调
- task lifecycle

如果你未来要深入多 agent，这一层必须单独读。

## 七、这个仓库的真正复杂度来自哪里

不是来自 React 组件，而是来自下面这些叠加：

- 多运行模式共用核心逻辑
- 工具系统动态组合
- 权限系统多来源决策
- MCP 动态接入
- 会话可恢复
- remote / bridge / viewer 三套远程语义并存
- feature gate / GrowthBook / env / settings / policy 多层开关并存

## 八、最值得优先阅读的文件

如果你要理解这层基础设施，建议按顺序看：

1. `src/Tool.ts`
2. `src/tools.ts`
3. `src/services/tools/toolExecution.ts`
4. `src/services/tools/toolOrchestration.ts`
5. `src/services/tools/StreamingToolExecutor.ts`
6. `src/utils/permissions/permissionSetup.ts`
7. `src/utils/permissions/permissions.ts`
8. `src/hooks/useCanUseTool.tsx`
9. `src/services/mcp/config.ts`
10. `src/services/mcp/client.ts`
11. `src/services/mcp/useManageMCPConnections.ts`
12. `src/services/oauth/index.ts`
13. `src/services/analytics/index.ts`
14. `src/services/policyLimits/index.ts`
15. `src/services/compact/*`

## 最后一个总判断

这个项目最有价值的部分，不是“某个具体工具有没有实现”，而是它已经把 Claude Code 这类产品的核心工程骨架还原得很清楚：

- 统一会话模型
- 工具运行时
- 权限网关
- 扩展协议接入
- 远程控制与恢复机制

读懂这一层，你就基本读懂了整个项目的设计重心。
