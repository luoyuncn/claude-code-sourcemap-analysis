# 外部交互模式与多宿主运行面

## 1. Claude Code 不是单一 CLI，而是同一 Runtime 的多种宿主外壳

如果只从 `main.tsx` 的入口去看，Claude Code 2.1.88 最值得注意的一点是：它已经不把自己当成“一个本地终端程序”。

前面几篇已经看过 `main.tsx` 的分流逻辑，这里重点只看外部交互相关的几条：

- `main.tsx:3193-3227` 的 `claude ssh`
- `main.tsx:3960-4037` 的 `claude server`
- `main.tsx:4055-4095` 的 `claude open` / `cc://`

从这三组入口可以直接得出一个结论：

Claude Code 的核心已经是 Agent Runtime，本地 CLI 只是它的一种 host。

## 2. `ssh`、`server`、`open/connect` 三条入口对应三种宿主关系

### 2.1 `claude ssh`：远端执行，本地渲染

`main.tsx:3193-3227` 的注释非常清楚：

- 探测远端环境
- 必要时部署 binary
- 建立 ssh + unix socket 转发
- 工具在远端跑
- UI 在本地渲染

这说明 Claude Code 已经把“执行宿主”和“交互宿主”拆开了。

### 2.2 `claude server`：把 Agent Runtime 变成 session host

`main.tsx:3960-4037` 会启动一个 session server，并允许配置：

- `--port`
- `--host`
- `--auth-token`
- `--unix`
- `--workspace`
- `--idle-timeout`
- `--max-sessions`

这里最重要的，不是某个参数，而是它说明 Claude Code 核心已经可以被“托管”为一个会话服务器。

### 2.3 `claude open` / `cc://`：把交互端变成远程客户端

`main.tsx:4055-4095` 里，`open <cc-url>` 会：

- 解析 connect URL
- 建立 direct connect session
- 必要时重写本地 cwd
- 然后运行 headless connect

这说明 Claude Code 甚至已经把“连接现有 Agent 会话”做成一等产品面，而不是内部调试能力。

## 3. RemoteSessionManager 说明远程会话不是“远程打印日志”

### 3.1 它负责的其实是一套双向协议

`remote/RemoteSessionManager.ts:50-95` 和 `95-140` 说明这个对象负责：

- WebSocket 订阅远程 session
- HTTP / event 方式发送本地输入
- 管理 permission request / response
- 处理 reconnect / disconnect / viewer-only 行为

这里的关键词不是 transport，而是协议。

### 3.2 permission prompt 也被远程化了

`remote/RemoteSessionManager.ts:153-198` 表明，来自远端的 `control_request` 会被识别成：

- 权限请求
- 控制取消请求
- 控制响应

然后再回调给本地 UI。

这意味着 Claude Code 远程模式不是“远端自己弹权限框”，而是把控制消息桥接回本地宿主。

这对 Agent 产品非常重要，因为：

- 用户真正交互的地方可能不在执行宿主
- 但决策权和确认权仍然在用户手里

## 4. `sdkMessageAdapter` 是远端消息桥接到本地 REPL 的关键层

### 4.1 它把远端 SDK 消息翻译成本地消息类型

`remote/sdkMessageAdapter.ts:21-148` 负责把远程传回的：

- assistant
- partial assistant
- result
- init
- status
- tool progress
- compact boundary

转换成本地 REPL 能理解的：

- `AssistantMessage`
- `StreamEvent`
- `SystemMessage`

这个适配层很关键，因为它说明 Claude Code 的显示层并不依赖唯一消息格式，而是通过 adapter 做宿主解耦。

### 4.2 tool result 与普通 user text 也被区别处理

`remote/sdkMessageAdapter.ts:150-214` 里，对 remote user message 的处理很细：

- tool_result 内容块可转换成本地 `UserMessage`
- 普通 user text 是否渲染，要看历史回放还是 live 模式

这表明远端消息并不是“原样贴回来”，而是要重新适配到本地 transcript 语义。

一句话说，这一层做的是：

- 远程协议消息
- 本地显示消息

之间的语义对齐。

## 5. 外部模式的成熟度，体现在“边缘交互面也被 Runtime 化”

除了 server / connect / ssh，源码里还有两块很容易被忽视的外部交互面：

- `voice/voiceModeEnabled.ts`
- `buddy/companion.ts`

它们虽然不直接参与 Agent Loop 主干，但非常能说明 Claude Code 的产品成熟度。

### 5.1 voice mode 不是按钮开关，而是“特性门控 + 认证门控”

`voice/voiceModeEnabled.ts:16-54` 把 voice mode 的启用拆成两层：

1. GrowthBook kill-switch
2. Anthropic OAuth auth 检查

代码注释甚至明确说：

- voice endpoint 只适用于 claude.ai auth
- API key / Bedrock / Vertex / Foundry 都不行

这意味着 voice 不是“UI 功能”，而是受宿主认证能力约束的外部交互面。

### 5.2 buddy / companion 说明产品层状态也可挂接到 Runtime

`buddy/companion.ts:15-133` 通过用户 ID 决定性生成 companion，并把“骨架”和“灵魂”分开：

- bones 由用户 ID 派生
- 存储态只保留可持久部分
- 配置编辑不能伪造 rarity

虽然这是偏体验功能，但它传达出一个重要信号：

Claude Code 的产品层体验不是硬编码在 UI 里，而是也通过可恢复、可推导的运行时状态来管理。

## 6. 为什么这些外部模式值得 Agent 开发者重点学习

### 6.1 因为它们证明 Runtime 与宿主可以解耦

Claude Code 已经把核心能力拆成：

- 运行时核心
- 本地 REPL 宿主
- 远程连接宿主
- server 宿主
- ssh 宿主

这意味着同一套 Agent Loop，可以被挂到不同产品外壳上。

### 6.2 因为它们证明“权限、消息、状态”都应该跨宿主统一

远端会话如果只是“回传文本”，产品一定会失控。

Claude Code 之所以稳，是因为它跨宿主统一了：

- message protocol
- permission protocol
- session metadata
- transcript semantics

### 6.3 因为它们证明 UI 只是 Runtime 的一个投影

不管是本地 TUI、headless connect、voice，还是 buddy，这些都不是核心能力本身，而是 Runtime 状态在不同交互面的投影。

## 7. 和 `learn-claude-code` 的关系

`learn-claude-code` 主要讲的是最小 Harness 内核：

- agent loop
- tools
- subagents
- compact
- task system

它没有展开多宿主问题，而 Claude Code 2.1.88 正好补上了这一课：

- 同一内核如何挂到 server
- 如何挂到 remote session
- 如何通过 ssh 把执行迁到远端
- 如何把远程 SDK 消息重新映射回本地 REPL

可以说：

- `learn-claude-code` 是单宿主 Agent 教科书
- Claude Code 是多宿主 Agent Runtime 标本

## 8. 对 Agent 开发者的启发

### 启发 1：先把核心 Runtime 与宿主壳分开

如果一开始就把：

- 渲染
- transport
- prompt loop
- session state

写死在一起，后面很难长出 server、remote 或 SDK。

### 启发 2：远程模式一定要设计控制协议，不要只传文本

至少要显式桥接：

- permission request
- permission cancel
- streaming delta
- result
- compact boundary

### 启发 3：消息适配层是必须的

本地 REPL 消息格式和远程 SDK 消息格式通常不一样。Claude Code 的 `sdkMessageAdapter` 是一个非常实用的中间层模式。

### 启发 4：外部交互面要受认证与特性门控约束

voice mode 这种实现提醒我们：很多看似 UI 层的功能，本质上依赖后端身份和产品策略，应该纳入 Runtime gating，而不是前端随便显隐。

## 9. 一句话总结

Claude Code 2.1.88 的外部交互模式告诉我们，它真正做成的不是“一个终端里的 coding assistant”，而是一套能够在本地、远程、服务端、SSH 与其他交互面之间切换的多宿主 Agent Runtime。
