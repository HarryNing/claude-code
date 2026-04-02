# 03. Remote / Bridge / Server 架构

这一部分最容易混淆，因为仓库里同时存在多种“远程”能力。

先做概念切分。

## 先区分四个概念

### 1. Remote session

含义：

- agent 在远端跑
- 本地 CLI 主要负责展示消息、发送输入、代答权限请求

核心文件：

- `src/remote/RemoteSessionManager.ts`
- `src/remote/SessionsWebSocket.ts`
- `src/remote/sdkMessageAdapter.ts`
- `src/hooks/useRemoteSession.ts`

### 2. Remote Control / Bridge

含义：

- agent 运行所需的“环境”在本地机器上
- 远端 UI 或远端控制层通过 bridge 驱动本地执行

核心文件：

- `src/bridge/initReplBridge.ts`
- `src/bridge/replBridge.ts`
- `src/bridge/remoteBridgeCore.ts`
- `src/bridge/bridgeMain.ts`

### 3. Direct Connect

含义：

- 连接一个 Claude Code session server
- 类似本地/局域网 server-client 模式

当前状态：

- 客户端部分存在
- 服务端关键实现当前仓库里仍是 stub

核心文件：

- `src/server/createDirectConnectSession.ts`
- `src/server/directConnectManager.ts`
- `src/server/server.ts` 目前是 stub

### 4. SSH Remote

含义：

- 通过 SSH 在远端主机上运行 Claude Code
- 本地负责认证代理和 UI

当前状态：

- `main.tsx` 已保留接线与入口
- `src/ssh/*` 当前仍是 stub

## 一、Remote session 这条链路

### 启动入口

`src/main.tsx` 中和 remote 相关的路径主要有两类：

- `--remote`
- `claude assistant [sessionId]`

前者更偏“创建/连接远端会话”，后者更偏“附着到正在运行的 assistant 会话”。

### 核心对象：`RemoteSessionManager`

文件：

- `src/remote/RemoteSessionManager.ts`

它本质上是一个“远端会话客户端编排器”，负责三件事：

- 通过 `SessionsWebSocket` 订阅消息
- 通过 HTTP POST 发送用户消息
- 处理远端发来的 permission control request

也就是说，它不是执行引擎，而是本地的 transport + control coordinator。

### 核心传输层：`SessionsWebSocket`

文件：

- `src/remote/SessionsWebSocket.ts`

职责：

- 连到 `/v1/sessions/ws/{sessionId}/subscribe`
- 用 OAuth header 鉴权
- 保持 ping
- 在暂时断开时自动重连
- 把消息转回 `RemoteSessionManager`

### 消息适配：`sdkMessageAdapter`

文件：

- `src/remote/sdkMessageAdapter.ts`

它解决的问题是：

- 远端通过 WebSocket 发来的格式是 `SDKMessage`
- 本地 REPL 渲染用的是内部 `Message`

所以它负责把：

- assistant
- tool_progress
- status
- compact_boundary
- result

这些外部消息转换成内部可渲染消息。

### REPL 接入点：`useRemoteSession`

文件：

- `src/hooks/useRemoteSession.ts`

这个 hook 负责把远程会话真正接进 REPL 生命周期：

- 初始化 `RemoteSessionManager`
- 将远程消息写入本地 `messages`
- 维护远程连接状态
- 处理远端发来的 tool permission request

## 二、Bridge 是怎么工作的

Bridge 的目标是“把本地机器接到远端控制平面上”，所以它不是 viewer，而是 worker 侧能力。

这里有两条实现路径。

## 2.1 REPL 内的 bridge

### 入口：`initReplBridge`

文件：

- `src/bridge/initReplBridge.ts`

它是 REPL 专用包装层，负责：

- 检查是否启用 bridge
- 检查 OAuth
- 检查组织策略 `allow_remote_control`
- 获取 cwd、git、title、org、token 等 bootstrap 信息
- 决定走 env-based bridge 还是 env-less bridge

这是 REPL bridge 的“前置编排器”。

### 两种 bridge 实现

#### A. env-based bridge

核心文件：

- `src/bridge/replBridge.ts`

它依赖“environment + poll/dispatch”模型，核心流程是：

1. 注册 environment
2. 创建 bridge session
3. 轮询 work
4. 建立 ingress transport
5. 收发 SDK/control 消息
6. 定期 heartbeat / refresh token

这条路径比较重，但兼容更成熟。

#### B. env-less bridge

核心文件：

- `src/bridge/remoteBridgeCore.ts`

它的关键思想是：

- 不再先注册 environment，再通过 work-dispatch 获得 worker 身份
- 而是直接创建 code session，然后调用 `/bridge` 换取 worker jwt

源码注释里已经说得很清楚：env-less 关注的是“去掉 environments API 层”，不是简单等于 CCR v2。

### REPL 里的运行时挂载

文件：

- `src/hooks/useReplBridge.tsx`

这个 hook 的职责是：

- 监听 `AppState.replBridgeEnabled`
- 初始化或 teardown bridge handle
- 把本地消息增量写到 bridge
- 把远端发来的 inbound message 注入本地命令队列
- 维护 `replBridgeConnected`、`replBridgeSessionUrl`、`replBridgeError` 等状态

所以 `initReplBridge.ts` 更像初始化器，`useReplBridge.tsx` 才是 REPL 生命周期里的桥接控制器。

## 2.2 独立 `claude remote-control` 模式

### 入口：`bridgeMain.ts`

文件：

- `src/bridge/bridgeMain.ts`

这是 standalone bridge / bridge worker 的主程序。它和 REPL bridge 的差别在于：

- 它自己负责长期轮询和工作调度
- 它可以管理多个 session
- 它会在本地 spawn Claude child process 来承接具体 work

### 核心职责

`bridgeMain.ts` 主要负责：

- 初始化 bridge config
- 注册环境
- 决定 spawn mode
- 轮询服务端 work
- 为每个 work/session spawn 本地子进程
- 管理 active sessions、timeouts、heartbeats、reconnect
- 维护 crash recovery pointer

### 关键对象

#### `SessionSpawner`

文件：

- `src/bridge/sessionRunner.ts`

职责：

- 用 `child_process.spawn` 启动本地 Claude 子进程
- 监听 stdout/stderr
- 提取 activity
- 转发 permission request

#### `BridgeApiClient`

文件：

- `src/bridge/bridgeApi.ts`

职责：

- 和 bridge 后端 API 通信
- register / poll / ack / reconnect / stop / heartbeat

#### `workSecret`

文件：

- `src/bridge/workSecret.ts`

职责：

- 解析服务端下发的 work secret
- 从 secret 里拿到 session ingress token 和 api base url
- 构造 SDK URL
- 注册 worker

它是把“服务端分配的 work”变成“本地 child process 可连 ingress 的执行凭证”的关键桥梁。

## 三、resume / reconnect / session pointer / ingress 是什么关系

这四个概念很关键，也最容易混。

## 3.1 session pointer

文件：

- `src/bridge/bridgePointer.ts`

它是本地 crash recovery 文件，不是服务端会话本身。

里面记录：

- `sessionId`
- `environmentId`
- `source`

作用：

- bridge 崩溃后，下次启动可以知道“上次跑到哪一个 session/environment”
- `--continue` 可以跨 worktree 查找最新 pointer

要点：

- pointer 是本地文件
- 通过 mtime 判断 TTL
- 只是恢复索引，不是消息数据本身

## 3.2 resume

resume 更偏“用户视角的继续会话”。

在 bridge 语境里，resume 可能做两件事：

- 用 pointer 找回原 session/environment
- 重新把本地进程接到原来的远程控制会话

在普通 CLI 语境里，resume 更多依赖 transcript JSONL 和 session metadata。

## 3.3 reconnect

reconnect 更偏“传输/worker 层恢复”，不是用户层 resume。

典型位置：

- `api.reconnectSession(...)`
- bridge token 失效后重排队
- transport 断开后重建

它的语义是：

- 让后端重新把 work 分发回来
- 或让会话重新获得有效 worker/ingress 连接

不是“重新载入历史消息”。

## 3.4 session ingress

你可以把 ingress 理解成“进入某个 session runtime 的实时入口”。

相关位置：

- `buildSdkUrl(...)`
- `session_ingress_token`
- `updateSessionIngressAuthToken(...)`

作用：

- 建立 WebSocket/SSE/worker 通道
- 让本地子进程或 REPL bridge 能真实进入那个 session 的执行面

一句话总结：

- pointer 解决“本地知道之前连的是谁”
- resume 解决“用户要继续哪个会话”
- reconnect 解决“连接/worker 断了怎么续”
- ingress 解决“真正通过什么通道进入该 session”

## 四、Direct Connect 的现状

这条链路当前是“客户端有骨架，服务端没补完”。

### 已存在

- `src/server/createDirectConnectSession.ts`
- `src/server/directConnectManager.ts`

说明：

- 已经有“向 server 创建 session”和“通过 ws 管理 session”的客户端形状
- `DirectConnectSessionManager` 也能处理 control request / interrupt

### 目前是 stub

- `src/server/server.ts`
- `src/server/sessionManager.ts`
- `src/server/parseConnectUrl.ts`
- `src/server/connectHeadless.ts`

所以你阅读 direct connect 时，应该把它看成“接口设计和客户端路径已经逆出来，但 server 还没补齐”。

## 五、daemon 的现状

从入口上看：

- `src/entrypoints/cli.tsx` 已经给 `--daemon-worker` 和 `daemon` 留了 fast path

但关键文件：

- `src/daemon/main.ts`
- `src/daemon/workerRegistry.ts`

当前仍是 stub。

所以这里更像“预留了架构位置”，还不是完整实现。

## 六、最值得优先阅读的文件

如果你只想快速建立 remote/bridge 的清晰模型，建议按这个顺序：

1. `src/bridge/initReplBridge.ts`
2. `src/bridge/replBridge.ts`
3. `src/bridge/remoteBridgeCore.ts`
4. `src/bridge/bridgeMain.ts`
5. `src/bridge/bridgePointer.ts`
6. `src/bridge/workSecret.ts`
7. `src/bridge/sessionRunner.ts`
8. `src/remote/RemoteSessionManager.ts`
9. `src/remote/SessionsWebSocket.ts`
10. `src/remote/sdkMessageAdapter.ts`
11. `src/hooks/useReplBridge.tsx`
12. `src/hooks/useRemoteSession.ts`
13. `src/server/createDirectConnectSession.ts`
14. `src/server/directConnectManager.ts`

## 最后一个总判断

这个仓库的“远程能力”并不是单一系统，而是三层叠加：

- 远端执行，本地查看：remote session
- 本地执行，远端控制：bridge
- 点对点 server-client：direct connect

理解这三层以后，你就不会再把 `remote`、`assistant`、`bridge`、`server` 混成一件事。
