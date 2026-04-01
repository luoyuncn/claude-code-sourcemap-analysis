# Runtime、状态模型与 Query Loop

## 1. 为什么 Claude Code 的主循环不是一个文件

在 `learn-claude-code` 里，Agent Loop 可以收敛成一个 `while True`。但在 Claude Code 2.1.88 里，主循环被拆成了四层：

1. `main.tsx` 负责入口、模式和会话准备。
2. `replLauncher.tsx` 负责把 App 和 REPL 装进 UI 容器。
3. `screens/REPL.tsx` 负责交互层状态拼装。
4. `query.ts` / `QueryEngine.ts` 负责模型驱动循环。

这种拆分不是“写得复杂”，而是因为生产级 Agent 已经同时面对三种现实：

- UI 模式和 headless 模式要共用核心逻辑
- 会话状态不能只活在 React 组件里
- 工具、MCP、技能、权限、压缩都要在每轮调用前动态装配

## 2. 入口到循环的装配链

### 2.1 `main.tsx` 不是业务逻辑，而是运行时总开关

`main.tsx:585-830` 展示了它在 `main()` 里完成的大量分流工作：

- URL / deep link 处理
- `assistant` 模式改写
- `ssh` 模式改写
- interactive 与 non-interactive 判定
- clientType 判定

在 `main.tsx:3134-3146`，继续会话的交互场景最终会走到 `launchRepl(...)`。说明主入口的职责是：把“这次会话应该以什么形态运行”决定清楚，而不是直接写 Agent Loop。

### 2.2 `launchRepl()` 很薄，但边界很清晰

`replLauncher.tsx:12-21` 基本只做一件事：把 `App` 和 `REPL` 组装后交给 `renderAndRun`。这层看似简单，但把“外壳 UI”与“REPL 内核”分开了。

它的意义在于：Agent Runtime 可以继续扩展，但 UI 宿主的接口保持稳定。

## 3. REPL 层负责把“本轮上下文”装配出来

### 3.1 真正送进 `query()` 的不是裸消息，而是一个运行时上下文包

`REPL.tsx:2746-2801` 是整个交互式主循环最关键的代码片段之一：

- 先通过 `getToolUseContext(...)` 取到本轮最新工具上下文
- 从 store 里拿到最新的 tools 与 mcpClients
- 计算默认 system prompt、user context、system context
- 再调用 `query(...)`

也就是说，Claude Code 不是“初始化一次 system prompt 然后一直用”。它每轮都会依据当前状态重新装配：

- 权限模式
- MCP 连接
- 附加目录
- coordinator 用户上下文
- proactive / kairos 状态

这就是它比教程版更像 Runtime 的地方。

### 3.2 Skill frontmatter 对本轮权限是即时生效的

`REPL.tsx:2701-2726` 会把 skill frontmatter 带来的 `additionalAllowedTools` 写入 `toolPermissionContext.alwaysAllowRules.command`。这意味着 skill 不是纯提示注入，它还能改变本轮工具权限面。

这里的设计非常关键：

- 技能不只是知识模板
- 技能还是“本轮工具策略”的一部分
- 这个策略是按 turn 生效，而不是永久污染全局 store

## 4. `query.ts`：生产级 Agent Loop 的真正内核

### 4.1 它仍然是 `while (true)`，只是状态更厚

`query.ts:181-217` 定义了 `QueryParams` 和内部 `State`。这说明 Claude Code 没有抛弃经典 Loop，只是把状态对象做厚了：

- `messages`
- `toolUseContext`
- `autoCompactTracking`
- `maxOutputTokensRecoveryCount`
- `pendingToolUseSummary`
- `transition`

`query.ts:241-307` 的 `queryLoop()` 里，仍然是一个无限循环；这和 `learn-claude-code/s01` 在思想上完全一致。

### 4.2 它在一轮里处理的事情，远比“tool_use -> tool_result”多

`query.ts:720-860` 显示，一次 assistant 响应到来后，它要做：

- tool input backfill
- recoverable error withhold
- streaming tool executor 分发
- tool-use block 收集
- follow-up 触发

`query.ts:1380-1470` 继续展示：

- 执行 tool batch
- 根据结果更新 tool context
- 生成 tool use summary
- 为下一次 API 调用准备继续条件

教程版是“读一个 block，执行工具，回填结果”；Claude Code 是“围绕 tool-use 构造一个稳定、可恢复、可摘要、可统计的执行闭环”。

## 5. `QueryEngine.ts`：把 REPL 循环抽成 headless / SDK 能复用的引擎

### 5.1 `QueryEngine` 的定位非常明确

`QueryEngine.ts:175-183` 直接说明：

> 一个 `QueryEngine` 对应一个 conversation，`submitMessage()` 对应一个 turn，状态会跨 turn 持续存在。

这意味着它不是简单包装 `query()`，而是给 SDK/非交互模式提供一个“可复用会话引擎”。

### 5.2 它持有的状态说明了 Claude Code 认为什么才叫“会话”

`QueryEngine.ts:184-206` 里保存了：

- `mutableMessages`
- `permissionDenials`
- `totalUsage`
- `readFileState`
- `discoveredSkillNames`
- `loadedNestedMemoryPaths`

这说明“会话”不是一串文本消息，而是：

- 消息历史
- 文件读取缓存
- 技能发现状态
- 记忆注入状态
- 权限失败历史
- token 使用历史

### 5.3 QueryEngine 负责把 streaming 事件、记录落盘和 resume 连起来

`QueryEngine.ts:675-850` 的 for-await 循环，会把 `query()` 产生的各种消息分门别类处理：

- assistant / user / compact boundary 进 transcript
- progress / attachment 也会内联记录
- `message_delta` 更新 stop_reason 与 usage
- structured output / max turns reached 等 attachment 被单独处理

这层设计让 headless 场景依然拥有：

- 可恢复 transcript
- 可统计 usage
- 可转发 partial event
- 可在中断后延续状态

## 6. Claude Code 的状态模型为什么值得学

### 6.1 “消息状态”与“运行状态”被分开了

在教程版里，几乎所有状态都塞进 `messages[]`。Claude Code 则显式分出了：

- `messages`: 语义历史
- `ToolUseContext`: 本轮运行参数与共享能力
- `AppState`: UI / 权限 / MCP / task / agent 等全局运行态
- `readFileState`: 文件缓存态
- `sessionStorage`: 可恢复持久态

这套拆分有两个收益：

1. 不必把所有运行信息都喂给模型
2. 不会因为上下文压缩而丢掉系统自身的工作状态

### 6.2 `ToolUseContext` 是 Claude Code Runtime 的中心接口

`Tool.ts:158-220` 显示 `ToolUseContext` 里同时包含：

- tools / commands / mcp / agentDefinitions
- `getAppState` / `setAppState`
- `abortController`
- `readFileState`
- `appendSystemMessage`
- `sendOSNotification`
- nested memory 去重状态

也就是说，在 Claude Code 里，工具不是直接依赖全局变量，而是依赖一个统一的运行时上下文接口。这是非常值得复用的工程模式。

## 7. 对 Agent 开发者的启发

### 启发 1：最小 Loop 永远正确，但不够

`learn-claude-code` 证明了最小 Loop 是永恒骨架；Claude Code 证明了真实产品要把这个骨架包进一个状态丰富的 Runtime。

### 启发 2：会话引擎要独立于 UI

如果你的 Agent 核心写死在前端组件里，未来很难支持：

- SDK
- batch
- remote session
- server 模式
- reconnect / resume

Claude Code 通过 `query.ts` + `QueryEngine.ts` 做到了同一内核多宿主。

### 启发 3：把“每轮组装上下文”当成一等能力

不要迷信一次性 system prompt。更成熟的做法是：

- 静态提示只放底座
- 轮次相关状态动态拼装
- 权限、技能、MCP、工作目录都在 turn 级别生效

这会让你的 Agent 从“静态提示应用”变成“活的运行时”。
