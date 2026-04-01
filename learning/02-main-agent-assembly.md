# 主 Agent 装配：Claude Code 如何把一次提问变成一次可执行的工程回合

## 1. 主 Agent 不是“模型 + prompt”，而是一次完整装配

Claude Code 里最值得研究的一点，是它把一次用户提问当成了一次“运行时装配”过程，而不是一次固定模板调用。

从 `screens/REPL.tsx:2746-2801` 可以看到，一次 query 前至少要组装这些对象：

- 当前消息序列 `messages`
- `ToolUseContext`
- 最新工具集合 `freshTools`
- 最新 MCP client 集合 `freshMcpClients`
- `defaultSystemPrompt`
- `userContext`
- `systemContext`
- `effectiveSystemPrompt`

这意味着主 Agent 不是静态实体，而是“每轮重建一次工作现场”的运行体。

## 2. 装配第一层：入口确定当前会话的宿主形态

`main.tsx:585-830` 的职责可以理解为“选择宿主”：

- 本地交互式 REPL
- print / SDK 模式
- `connect` 到外部 server
- `ssh` 到远端宿主
- `assistant` / remote 场景

到 `main.tsx:3134-3146` 或 `3176-3191` 这类分支时，才真正把会话交给 `launchRepl(...)`。也就是说，主 Agent 不是直接从 CLI 起飞，而是先被放进一个特定宿主环境。

这和 `learn-claude-code` 最大的不同是：教程版默认“宿主只有一个”，Claude Code 则默认“宿主很多，但核心 Agent 要复用”。

## 3. 装配第二层：REPL 将全局状态裁成“本轮状态”

### 3.1 `ToolUseContext` 是主 Agent 的实际运行接口

`Tool.ts:158-220` 定义的 `ToolUseContext` 里包含：

- `options.tools`
- `options.commands`
- `options.mcpClients`
- `options.agentDefinitions`
- `getAppState` / `setAppState`
- `readFileState`
- `abortController`
- `appendSystemMessage`
- `sendOSNotification`

这等于告诉我们：Claude Code 的主 Agent 真正依赖的不是 React props，也不是全局变量，而是一份统一运行接口。

### 3.2 工具和权限在装配阶段就已经绑定

`REPL.tsx:2701-2726` 会先把 skill frontmatter 带来的 allowed tools 写入 store；`REPL.tsx:2746-2755` 再重新从 `toolUseContext.options` 拿到最新工具集。这说明：

- 工具不是固定列表
- 权限不是工具调用时再判断
- “这一轮模型看见什么”本身就是装配结果

这是主 Agent 装配最重要的设计之一。

## 4. 装配第三层：system prompt 不是常量，而是运行结果

`REPL.tsx:2768-2787` 展示了一个非常典型的 Claude Code 风格：

1. 先用 `getSystemPrompt(...)` 生成默认系统提示
2. 再基于 coordinator / agentDefinition / 自定义提示做 `buildEffectiveSystemPrompt(...)`
3. 最后把结果挂进 `toolUseContext.renderedSystemPrompt`

这和很多 Agent Demo 里“一段 SYSTEM 常量走天下”完全不同。

Claude Code 的 system prompt 至少反映这些变量：

- 当前工具池
- 当前模型
- 当前 MCP 环境
- 当前附加目录
- 当前是否是 coordinator 模式
- 当前是否有 custom / append system prompt

也就是说，Claude Code 把 prompt 视作 Runtime 反射层，而不是静态文案。

## 5. 装配第四层：真正进入 Agent Loop

### 5.1 `query()` 是主回合引擎

`query.ts:219-239` 暴露了 `query(...)` 这个 generator；`query.ts:241-307` 里的 `queryLoop()` 维护跨 iteration 的状态。

核心思想和 `learn-claude-code/s01` 相同：

- 发请求
- 看 stop_reason / tool_use
- 执行工具
- 继续下一轮

但 Claude Code 在这之上增加了工程层：

- auto compact tracking
- memory prefetch
- max output token recovery
- tool-use summary
- stop hook / continuation logic

### 5.2 `REPL.tsx` 只是消费这个循环

`REPL.tsx:2793-2803` 的逻辑其实很清楚：

```ts
for await (const event of query(...)) {
  onQueryEvent(event)
}
```

所以 REPL 不是主循环本身，而是主循环的事件消费者与 UI 投影层。

这和教程版相比是一个巨大升级：

- 教程版：循环与渲染混在一起
- Claude Code：循环产出事件，UI 再处理事件

## 6. `QueryEngine` 把这个装配流程抽成可复用会话引擎

### 6.1 构造函数已经暴露了装配输入项

`QueryEngine.ts:130-173` 的 `QueryEngineConfig` 很像一个“会话蓝图”，里面需要的输入包括：

- cwd
- tools / commands / mcpClients / agents
- `canUseTool`
- `getAppState` / `setAppState`
- `initialMessages`
- `readFileCache`
- 模型、预算、JSON schema、thinking config

这和我们在 REPL 中看到的装配项基本对齐，说明 Claude Code 已经把主 Agent 的运行条件抽象清楚了。

### 6.2 `submitMessage()` 说明“一次会话，多次 turn”的边界

`QueryEngine.ts:175-206` 和 `209-250` 表明：

- 一个 `QueryEngine` 对应一整个 conversation
- 每次 `submitMessage()` 对应一个 turn
- 但权限拒绝、文件缓存、技能发现、memory path 去重等状态会跨 turn 持续

这比“一次请求一个临时 agent”更接近真实工作形态。

## 7. Claude Code 主 Agent 装配的 6 个关键词

### 7.1 宿主分离

入口与 REPL / SDK / remote 分离。

### 7.2 上下文重算

每轮都重新计算可用工具、MCP、system prompt 和 user context。

### 7.3 状态分层

消息状态、UI 状态、权限状态、文件缓存状态、持久化状态分离。

### 7.4 事件驱动

`query()` 产出 event，REPL 消费 event。

### 7.5 运行时反射

prompt、工具、模式都来自当前 runtime，而不是固定代码常量。

### 7.6 容器化会话

`QueryEngine` 把一次 conversation 封装成可复用引擎，便于 headless / SDK 扩展。

## 8. 和 `learn-claude-code` 对照后能看出的升级路线

### 教程版主张

- `s01`: 先把 loop 建起来
- `s02`: 用 dispatch map 管工具
- `s05`: 按需注入知识

### Claude Code 的工程化升级

- 把 loop 封进 `query.ts`
- 把会话封进 `QueryEngine`
- 把工具封进 `ToolUseContext`
- 把提示词封进 runtime 装配
- 把 UI 封进 REPL 事件消费层

所以它不是推翻了最小范式，而是沿着最小范式一路往上盖楼。

## 9. 对 Agent 开发者的启发

### 启发 1：主 Agent 需要“装配层”

如果你的 Agent 只有：

- `messages`
- `system prompt`
- `tools`

那它更像一个 demo。要变成产品，通常都需要一个显式装配层，把本轮环境重建出来。

### 启发 2：先抽 `ToolUseContext`，再抽别的

Claude Code 的经验是：统一运行接口一旦成立，工具、子代理、MCP、权限、compact 都更容易协同。

### 启发 3：把 Loop 事件化

如果 query loop 只能直接改 UI，你之后就很难扩到：

- SDK
- WebSocket remote session
- server mode
- background summarization

事件化是扩展面的起点。
