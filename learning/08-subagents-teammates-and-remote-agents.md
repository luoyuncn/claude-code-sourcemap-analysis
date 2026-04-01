# Subagents、队友与远程代理

## 1. Claude Code 的多代理，不是一个 `task()` 工具，而是一套任务运行时

如果我们只从 `learn-claude-code` 的 `s04` 出发，子代理的核心定义很简单：

- 新上下文
- 独立循环
- 最后返回摘要

这个定义没有错，但 Claude Code 2.1.88 已经把它扩成了一整套任务系统。核心文件包括：

- `tools/AgentTool/AgentTool.tsx`
- `tools/AgentTool/runAgent.ts`
- `tasks/LocalAgentTask/LocalAgentTask.tsx`
- `tasks/InProcessTeammateTask/types.ts`
- `tasks/RemoteAgentTask/RemoteAgentTask.tsx`
- `coordinator/coordinatorMode.ts`

从这些文件能看出，Claude Code 的多代理至少有五种形态：

1. 同步子代理
2. 后台本地子代理
3. fork 子代理
4. in-process teammate
5. remote agent

这已经不是“会不会分任务”，而是“Runtime 如何管理多个执行体”。

## 2. AgentTool 的输入 schema 已经暴露了多代理范式

### 2.1 spawn 时可控的，不只是 prompt

`tools/AgentTool/AgentTool.tsx:81-138` 的 schema 里，用户和模型可以控制的内容包括：

- `description`
- `prompt`
- `subagent_type`
- `model`
- `run_in_background`
- `name`
- `team_name`
- `mode`
- `isolation`
- `cwd`

这很关键，因为它说明 Claude Code 的 spawn 不是“只有一个 task prompt”，而是明确区分了：

- 角色
- 命名
- 权限模式
- 运行位置
- 隔离方式
- 生命周期

这套 schema，本质上已经是一份多代理运行协议。

### 2.2 worker 的工具池会重新装配，而不是直接继承父代理

`tools/AgentTool/AgentTool.tsx:567-577` 会用 worker 自己的 permission context 调用 `assembleToolPool(...)`。

这说明 Claude Code 不认为“父代理能用什么，子代理就一定该用什么”。

这个选择特别重要，因为它让多代理系统具备了真正的边界：

- 每个 agent 可以有自己的 permission mode
- 工具池是 agent-specific 的
- 父代理审批结果不会简单裸传给子代理

## 3. worktree、fork 与 cwd override 一起定义了“隔离语义”

### 3.1 worktree 不是 Git 技巧，而是 Runtime 隔离模式

`tools/AgentTool/AgentTool.tsx:582-593` 在 `isolation === 'worktree'` 时会创建 agent worktree。

后续 `643-685` 又定义了 cleanup 逻辑：

- 无改动则自动删除
- 有改动则保留
- hook-based worktree 直接保留

这说明 worktree 在 Claude Code 里是官方隔离通道，而不是用户手工最佳实践。

### 3.2 fork 子代理不是 fresh subagent，而是“上下文继承型子代理”

`tools/AgentTool/AgentTool.tsx:595-636` 和 `runAgent.ts:368-379` 一起看，语义会非常清楚：

- 普通子代理：新起上下文
- fork 子代理：继承父消息流、读文件缓存和更完整的执行上下文

而且 fork + worktree 时，还会额外注入 `buildWorktreeNotice(...)`，提醒子代理重新理解路径映射和文件新鲜度。

这说明 Claude Code 已经把 spawn 语义细分成：

- `fresh context`
- `forked context`
- `forked context + isolated filesystem`

这三者不能混为一谈。

## 4. `runAgent(...)` 让子代理真正变成独立执行体

### 4.1 它不是简单调用模型，而是在重建一个局部 Runtime

`tools/AgentTool/runAgent.ts:248-329` 的参数已经能说明问题：

- `forkContextMessages`
- `availableTools`
- `allowedTools`
- `override.systemPrompt`
- `override.abortController`
- `worktreePath`
- `description`
- `transcriptSubdir`

这意味着 `runAgent` 做的不是“替父代理问一次模型”，而是创建一个具备自己上下文、工具集、终止控制、转录路径和恢复元数据的局部执行环境。

### 4.2 它还会按 agent 类型裁剪上下文

`tools/AgentTool/runAgent.ts:385-410` 对 `Explore` / `Plan` 这种只读 agent 做了特化：

- 可以省掉 `CLAUDE.md`
- 可以省掉大块 `gitStatus`

这说明 Claude Code 非常重视“不同 agent 不该吃同一份系统上下文”。

换句话说，它不是单纯复用主代理 prompt，而是在做 agent-specific context slimming。

### 4.3 权限提示是否能冒泡，也被纳入 agent 运行语义

`tools/AgentTool/runAgent.ts:412-460` 会根据：

- `agentPermissionMode`
- `isAsync`
- `canShowPermissionPrompts`

来决定是否避免 permission prompt、是否等待自动检查等。

这让子代理和后台代理在权限交互上有了明确行为差异，而不是共用主线程 UI 逻辑。

## 5. LocalAgentTask 说明后台代理已经被纳入统一任务框架

### 5.1 任务状态里保存了 Agent 的可观测运行面

`tasks/LocalAgentTask/LocalAgentTask.tsx:116-147` 的 `LocalAgentTaskState` 里保存了很多关键信息：

- `agentId`
- `prompt`
- `selectedAgent`
- `progress`
- `messages`
- `pendingMessages`
- `retain`
- `diskLoaded`
- `isBackgrounded`

这意味着后台代理不是“一次性 Promise”，而是 UI、resume、notification、output file 都能感知的持久任务对象。

### 5.2 任务通知已经被协议化成 XML

`tasks/LocalAgentTask/LocalAgentTask.tsx:197-260` 会把 agent 完成、失败、停止都包装成 `<task-notification>`。

通知体里还会带上：

- `task-id`
- `output-file`
- `status`
- `summary`
- 可选 `result`
- 可选 `usage`
- 可选 `worktree`

这是个非常重要的模式：多代理通信不是自由文本，而是结构化事件。

### 5.3 前台转后台也有明确控制面

`tasks/LocalAgentTask/LocalAgentTask.tsx:526-652` 通过：

- `registerAgentForeground(...)`
- `backgroundSignal`
- `backgroundAgentTask(...)`

把一个原本前台运行的 agent 转成后台任务。

这里的重点不是“能不能异步”，而是：

- 任务能否被 UI 接管
- 能否中途转入后台
- 能否继续保持通知与输出文件

这非常接近进程调度思路。

## 6. Coordinator 模式把“多代理协作”上升为系统角色

`coordinator/coordinatorMode.ts:80-145` 直接给 coordinator 注入一整套系统提示：

- 你是协调者，不是执行者
- 可以用 `AGENT_TOOL_NAME` 启 worker
- 可以用 `SEND_MESSAGE_TOOL_NAME` 继续既有 worker
- 可以用 `TASK_STOP_TOOL_NAME` 停止 worker
- worker 结果会以 `<task-notification>` 形式作为 user-role message 返回

而 `145-220` 之后的内容又进一步规定：

- 哪些任务应该并行
- 哪些任务不要委托
- 如何看待 verification

这说明在 Claude Code 里，多代理不是一个外挂 feature，而是一种明确的系统角色模式。

## 7. In-process teammate 说明“队友”不等于“一次性子任务”

`tasks/InProcessTeammateTask/types.ts:13-76` 定义了 `TeammateIdentity` 与 `InProcessTeammateTaskState`。

队友有自己独立的：

- `agentId`
- `agentName`
- `teamName`
- `planModeRequired`
- `parentSessionId`
- `permissionMode`
- `awaitingPlanApproval`
- `isIdle`
- `shutdownRequested`

而且 `messages` 还特意说明：

- 这里只保存 zoomed transcript 视图
- mailbox 消息不存这里

这背后反映的不是“子代理多了个名字”，而是 Claude Code 已经在区分：

- 会话执行上下文
- 团队通信上下文
- UI 观察上下文

### 7.1 队友视图还做了内存上限

`tasks/InProcessTeammateTask/types.ts:89-120` 给 UI mirror message array 设了 cap，避免 swarm 场景下内存爆炸。

这个细节很重要，因为它说明多代理一旦进真实产品，问题很快会从“怎么 spawn”变成“怎么活下去”。

## 8. RemoteAgentTask 把远程执行纳入了同一任务抽象

### 8.1 remote task 不是特例，而是统一框架中的一种 task type

`tasks/RemoteAgentTask/RemoteAgentTask.tsx:22-59` 定义了 `RemoteAgentTaskState`，包含：

- `remoteTaskType`
- `sessionId`
- `command`
- `title`
- `todoList`
- `log`
- `reviewProgress`
- `ultraplanPhase`

这说明远程代理并没有另起一套完全独立的体系，而是被纳入统一任务框架。

### 8.2 启动远程代理前先校验环境前提

`tasks/RemoteAgentTask/RemoteAgentTask.tsx:121-160` 里 `checkRemoteAgentEligibility(...)` 会检查：

- 是否登录
- 是否存在 cloud environment
- 是否在 git 仓库
- 是否有 git remote
- GitHub app 是否安装
- 是否被组织策略禁止

这说明 remote agent 不是“把本地 agent 换个 URL”，而是一个更强依赖宿主环境的执行平面。

### 8.3 远程代理也会写 sidecar，支持 resume 恢复

`tasks/RemoteAgentTask/RemoteAgentTask.tsx:386-460` 注册远程任务时，会把 task identity 写入 session sidecar。

`468-531` 恢复时又会：

- 扫 sidecar
- fetch live CCR status
- 如果还活着就重建 task state
- 继续开始 polling

这与本地 agent 的 transcript / metadata 方案是一脉相承的：所有执行体都必须能恢复。

### 8.4 远程代理完成依赖轮询与事件增量

`tasks/RemoteAgentTask/RemoteAgentTask.tsx:538-600` 的 poller 会：

- 拉取 remote session events
- 增量写入 output file
- 检查 session status
- 触发 completion checker
- 发出结构化通知

这说明 remote agent 在 Claude Code 里并不是 fire-and-forget webhook，而是持续被本地 Runtime 观察和编排的远程执行体。

## 9. 和 `learn-claude-code` 的关系

Claude Code 这套多代理运行时，可以看成把 `learn-claude-code` 的后半段全部产品化了：

- `s04` 对应 fresh subagent / fork subagent
- `s08` 对应 background local agent
- `s09` 对应 teammate identity 与消息协作
- `s10` 对应结构化请求-响应和任务通知协议
- `s11` 对应 idle / autonomy / team semantics
- `s12` 对应 worktree 隔离与任务绑定

教程版是在讲概念骨架，Claude Code 则把它们拆进：

- tool layer
- task layer
- transcript layer
- permission layer
- UI layer
- remote layer

## 10. 对 Agent 开发者的启发

### 启发 1：spawn 语义一定要显式化

至少要明确区分：

- fresh
- fork
- background
- teammate
- remote

否则系统很快会在上下文继承、权限继承和输出恢复上混乱。

### 启发 2：子代理返回值不该只是一段文本

真实产品里还需要：

- progress
- transcript
- output file
- usage
- lifecycle notification

Claude Code 的 task model 值得直接参考。

### 启发 3：多代理协作一定要协议化

`<task-notification>` 这种结构化事件，比“让 worker 随便说几句”稳得多，也更适合 resume、UI、SDK 和自动化消费。

### 启发 4：remote agent 最好复用本地 task 抽象

一旦本地与远程是两套完全不同的数据模型，后续 resume、面板展示、通知和停止操作都会变得很痛苦。

## 11. 一句话总结

Claude Code 2.1.88 的多代理范式，已经不是“主代理偶尔开个小号”，而是一个具备角色、边界、任务状态、恢复机制和远程执行面的完整 Agent 任务运行时。
