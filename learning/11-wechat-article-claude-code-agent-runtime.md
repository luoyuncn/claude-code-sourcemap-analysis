# 拆开 Claude Code 2.1.88：Agent Runtime 如何运行

研究 `Claude Code`，最容易犯的错，是把它看成一个会调模型、会调工具的 CLI。沿着 `restored-src/src/` 往下拆，这个判断很快就会失效。

Claude Code 2.1.88 更像一套完整的 Agent Runtime。CLI 只是宿主壳，prompt 只是输入，真正决定系统行为的是回合装配、权限治理、状态迁移、多代理调度、扩展接入和远程桥接。

这篇文章只做一件事：顺着一次请求穿过系统的路径，把这些零件拆开，并和 `learn-claude-code` 对照着看。

## 一、先看结论

如果只从使用者视角看，Claude Code 像是一个会帮你写代码的 Agent。但从源码结构看，它做的事更接近一套运行时。

它不负责创造智能。智能来自模型。Claude Code 负责的是另外几件更工程化的事情：

1. 把用户输入和当前工作环境装配成当前这一轮的执行上下文。
2. 决定这一轮模型能看见什么工具、什么资源、什么技能。
3. 在模型返回 `tool_use` 之后，执行工具、记录状态、更新上下文，再把控制权送回模型。
4. 当上下文膨胀、会话切换、子代理生成、远程连接建立时，保证这套状态机还能继续跑。

这四件事凑在一起，才是 Runtime。

所以它的核心不是“某个 prompt 写得多巧”，而是这一整套运转机构能否稳定、可恢复、可扩展、可治理。

## 二、一轮请求怎么跑

先把一轮请求穿过去，很多部件就自然能对上。

### 1. 入口先判定宿主

`main.tsx` 的 `main()` 在很早的阶段就开始重写 argv 和切换模式。最典型的几段在：

- `main.tsx:609-640`：把 `cc://` 和 `cc+unix://` 改写成 direct connect 流程
- `main.tsx:679-699`：把 `claude assistant` 改写成主命令路径
- `main.tsx:702-794`：把 `claude ssh <host>` 改写成本地 UI + 远端执行的 SSH 会话
- `main.tsx:797-833`：在初始化之前判断当前是不是 non-interactive、是不是 SDK、是不是 remote 入口

这意味着 `main.tsx` 干的不是传统 CLI 的“命令分发”，而是更像一个 host router。它先决定：

- 这次会话是在本地 CLI 跑
- 还是连到一个 `cc://` server
- 还是通过 `ssh` 把执行迁到远端
- 还是进入 `assistant` / SDK / remote session 形态

这一步很关键。因为从这里开始，Claude Code 的设计目标就已经不是“本地命令行程序”，而是“同一套 Runtime 可以挂在多个宿主上”。

再看 `main.tsx:3960-4037` 和 `4055-4095`，这个判断会更明确：

- `claude server` 可以把 Runtime 变成 session host
- `claude open <cc-url>` 可以把本地进程变成 remote client
- `claude ssh` 可以把 UI 和执行宿主拆开

这是第一层骨架：**它先是 Runtime，后是 CLI。**

### 2. REPL 在装配回合

很多人会自然以为：用户输入一段文本，系统把它丢给模型，模型开始输出工具调用。这种理解只适合最小 demo。

Claude Code 的 `REPL.tsx` 做的事情复杂得多。

关键逻辑在 `REPL.tsx:2701-2803`。这一段最重要的不是 `query(...)` 调用本身，而是调用之前发生的事情：

1. 把当前 slash command / skill 的 `allowed-tools` 写入 `toolPermissionContext`
2. 重新读取当前 store 中最新的 tools 和 MCP clients，而不是用闭包里旧快照
3. 根据当前模型、当前工具、当前 working directories、当前 MCP 连接，重新生成默认 system prompt
4. 叠加 coordinator mode、scratchpad、terminal focus、custom system prompt、append system prompt 等额外上下文
5. 最后才调用 `query(...)`

其中 `REPL.tsx:2701-2726` 很能说明问题。skill frontmatter 里的 `allowed-tools` 不只是文档元数据，它会在这一轮开始前被写进全局的 `alwaysAllowRules.command`。也就是说，技能不是“告诉模型应该怎么做”，而是会同步改变当前这一轮的能力边界。

而 `REPL.tsx:2768-2787` 则展示了另一层事实：system prompt 在 Claude Code 里不是一个常量，而是每一轮按当前运行时状态重新反射出来的结果。

这一步非常重要，因为它决定了 Claude Code 的执行单元不是“消息”，而是“一个回合”。每一回合在发起模型请求之前，都会先把当前世界重新装配一遍。

### 3. `query.ts` 是执行内核

如果说 `main.tsx` 是宿主层，`REPL.tsx` 是回合装配层，那么 `query.ts` 才是 Claude Code 真正的 Agent Kernel。

`query.ts:181-198` 先定义了 `QueryParams`，里面除了常见的 `messages`、`systemPrompt`、`userContext`、`toolUseContext`，还有几个特别有意思的字段：

- `fallbackModel`
- `maxTurns`
- `skipCacheWrite`
- `taskBudget`

这些字段已经说明，Claude Code 的 query 不是“问模型一次”，而是一条能受预算、轮数和恢复策略约束的执行链。

更关键的是 `query.ts:203-217` 的 `State`。它明确把跨迭代的可变状态列出来：

- `messages`
- `toolUseContext`
- `autoCompactTracking`
- `maxOutputTokensRecoveryCount`
- `hasAttemptedReactiveCompact`
- `maxOutputTokensOverride`
- `pendingToolUseSummary`
- `turnCount`
- `transition`

这说明 `query()` 的内部模型不是 request-response，而是一个循环状态机。

再看 `query.ts:241-307` 和后面的主循环，Claude Code 每轮至少要处理这些问题：

1. 当前是否需要压缩上下文
2. 当前是否需要做 memory prefetch / skill discovery prefetch
3. 当前工具结果是否需要做 budget trimming
4. 当前这一轮请求该走哪个模型、用什么 thinking config、带哪些工具
5. 收到流式输出后，何时写 assistant message，何时写 stream event
6. 收到 `tool_use` 之后，工具如何执行、结果如何回灌、状态如何继续

这套逻辑和 `learn-claude-code` 的 `s01` 是同一条血脉，只是工业化程度完全不同。`s01` 讲的是：

```text
while stop_reason == tool_use:
  调模型
  执行工具
  回写 tool_result
```

Claude Code 保留了这个骨架，但把它扩展成了一个受限的、带恢复机制的、可做预算管理和上下文迁移的循环内核。

### 4. `QueryEngine` 复用内核

理解 Claude Code 的一个关键点，是不要把 REPL 和核心执行逻辑混成一坨。它其实已经在主动抽离。

`QueryEngine.ts:130-183` 直接把 `QueryEngine` 定义成“拥有 query lifecycle 和 session state 的对象”。这句话基本就是 Runtime 的自我描述。

它内部维护的是一组长期状态：

- `mutableMessages`
- `abortController`
- `permissionDenials`
- `totalUsage`
- `readFileState`
- `discoveredSkillNames`
- `loadedNestedMemoryPaths`

而 `submitMessage()` 在 `QueryEngine.ts:675-686` 最终还是调用了 `query(...)`。也就是说：

- `query.ts` 提供循环内核
- `QueryEngine` 提供会话对象和持久状态
- `REPL.tsx` 提供交互宿主

这就是一个很典型的 Runtime 分层。

`QueryEngine.ts:687-875` 还有一层很值得注意。它在流式处理过程中，会把：

- assistant
- user
- compact boundary
- progress
- attachment

都明确写进 `mutableMessages` 和 transcript。它并不只关心“最后的回复是什么”，而是把整个回合里发生的事件都变成会话状态的一部分。

这件事后面会直接影响 compact、resume 和 remote session。

## 三、Runtime 的 7 层结构

如果把整个运行时压成一张架构图，我会这么画：

```text
Host Shell
  -> Turn Assembler
    -> Query Kernel
      -> Capability Governance
      -> Continuity System
      -> Concurrency System
      -> Extension System
      -> Remote Bridge
```

下面把这几层逐层拆开。

## 四、能力治理

大多数 Agent Demo 的工具系统，核心只有一层：注册一个 schema，再挂一个 handler。

Claude Code 的重点不在这里。它真正做的是把“工具存在”与“工具是否可见、是否可调用、是否需要审批”拆开。

### 1. `ToolPermissionContext` 是执行环境

`Tool.ts:123-138` 里的 `ToolPermissionContext` 有这些字段：

- `mode`
- `additionalWorkingDirectories`
- `alwaysAllowRules`
- `alwaysDenyRules`
- `alwaysAskRules`
- `isBypassPermissionsModeAvailable`
- `isAutoModeAvailable`
- `shouldAvoidPermissionPrompts`
- `awaitAutomatedChecksBeforeDialog`
- `prePlanMode`

这段类型定义非常重要。因为它告诉我们，Claude Code 的权限模型不是一个 `"acceptEdits" | "bypass"` 枚举，而是一块完整的运行时上下文。

它同时携带：

- 当前模式
- 目录边界
- 允许规则
- 拒绝规则
- 提问规则
- UI 是否允许弹权限框
- 自动模式当前是否可进入

所以在 Claude Code 里，“权限”这个词的真实含义是：**当前这轮、当前这个执行体，在当前这组目录和规则下，哪些能力是可见且可执行的。**

### 2. 权限规则是逐层叠加的

`utils/permissions/permissionSetup.ts:913-1015` 很适合拿来理解这块逻辑。

它先构建 `additionalWorkingDirectories`，还特地处理了 `process.env.PWD` 是 symlink、`getOriginalCwd()` 是 real path 的情况。然后它再叠加：

- CLI 传入的 allow / deny
- disk 上的 permission rules
- settings 中的 `additionalDirectories`
- `--add-dir`

最后才得到完整的 `toolPermissionContext`。

也就是说，权限上下文不是静态配置读取结果，而是一个多来源合流后的运行结果。

### 3. Auto Mode 是能力门控

`permissionSetup.ts:1078-1127` 清楚地展示了 auto mode 的真实语义：

- 先看 GrowthBook 动态配置
- 再看 settings 是否禁用
- 再看当前主模型是否支持 auto mode
- 再看 `disableFastMode` breaker 是否触发

最后才得到两个关键结论：

- `carouselAvailable`
- `canEnterAuto`

这意味着 auto mode 在 Claude Code 里是一个运行时 capability gate，不是一个 UI 选项。

### 4. 工具先过滤再暴露给模型

`tools.ts:253-269` 的 `filterToolsByDenyRules(...)` 做了一个产品级系统一定要做的动作：在工具真正进入模型视野之前，先按 deny rules 把它们剔除。

这一点非常关键。

很多系统只在调用时做拦截，结果模型还是能看见这个工具，于是会不断尝试调用，造成：

- 错误推理
- token 浪费
- 模型对环境边界的误判

Claude Code 在 `tools.ts:271-327` 和 `345-349` 继续把这套策略落到底：

- built-in tools 先按 mode 过滤
- MCP tools 再按 deny rules 过滤
- 最后 dedupe 成完整 tool pool

这是一个非常成熟的设计。因为它把“工具系统”从 dispatch map 升级成了 capability governance。

## 五、连续性系统

Claude Code 另一块明显高于一般 Agent Demo 的地方，在于它对连续性的处理。

这里的“连续性”不是指用户能看到聊天记录，而是指系统能否在经历压缩、恢复、切换之后，继续像之前那样工作。

### 1. Compact 是状态迁移

`services/compact/compact.ts:518-585` 几乎是整套 Runtime 最值得反复读的代码之一。

压缩成功之后，它做的事情不是简单塞一段 summary，而是重建一组后压缩附件：

- 读过的文件附件
- 异步 agent 附件
- plan attachment
- plan mode attachment
- skill attachment
- deferred tools delta
- agent listing delta
- MCP instructions delta

这背后其实是一个非常明确的判断：**对 Agent 来说，压缩后的首要任务不是保留对话内容，而是保留工作态。**

这也是为什么 Claude Code 的 compact 之后，系统还能继续知道：

- 当前有哪些已加载技能
- 当前有哪些工具 schema 已经在上下文里
- 当前有哪些异步子代理还在跑
- 当前是否处于 plan mode

这不是摘要行为，这是运行时状态迁移。

### 2. Compact boundary 是显式边界

同一段代码里，`compact.ts:596-623` 会创建：

- `compact boundary marker`
- `compact summary message`

还会把压缩前已经发现的 deferred tools 一起写进 boundary metadata。

这非常重要。因为 Claude Code 没有偷偷“改写历史”，而是把压缩视作一次正式事件，显式写进消息流。

从 Runtime 角度看，这相当于：

- 前面是一段完整执行历史
- 这里发生了一次状态折叠
- 后面从新的状态基线继续执行

这样设计，resume、调试、remote replay 都会清晰得多。

### 3. Session Storage 是状态尾索引

`utils/sessionStorage.ts:722-839` 解释了 Claude Code 为什么能稳定做 `/resume`。

它会不断把这些 metadata 重新追加到 transcript 尾部：

- `last-prompt`
- `custom-title`
- `tag`
- `agent-name`
- `agent-color`
- `agent-setting`
- `mode`
- `worktree-state`
- `pr-link`

注意这里的关键不是“存了什么”，而是“写在哪里”。它用的是 append-only transcript 尾部重 stamping 机制，让 resume 所需字段始终落在 tail scan 可见窗口里。

这是一个非常漂亮的工程做法。它避开了：

- 全文件重写
- 额外索引数据库
- 复杂同步协议

代价只是追加几条 JSONL 元数据。

### 4. Memory 是文件系统协议

`memdir/memdir.ts:34-102` 和 `199-257` 表明 Claude Code 的 Memory 系统是一套显式的 file-based protocol：

- `MEMORY.md` 是索引入口，不是正文
- 每条 memory 独立存文件
- 有固定类型
- 有清晰的“什么应该存、什么不应该存”规则

这种设计和很多 embedding memory 正好相反。Claude Code 更偏向一个可以被审计、可被版本化、可被约束的记忆系统。

这很符合它整个 Runtime 的风格：重要状态不隐藏，要显式化、文件化、可恢复。

## 六、并发系统

很多人在谈多代理时，讨论重点放在：

- leader / planner / coder / reviewer 如何分工
- prompt 应该怎么写

Claude Code 的源码提醒我们，这些都不是第一问题。第一问题是：**多代理怎么被 Runtime 托管。**

### 1. `AgentTool` 的 schema 就是 spawn 协议

`AgentTool.tsx:81-138` 这段 schema 很能说明问题。它允许模型和宿主显式指定：

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

这已经不是“调用一个 task 工具”，而是在声明一个执行体应该如何被创建。

这里面至少区分了：

- 角色
- 生命周期
- 隔离方式
- 命名与路由
- 权限模式
- 执行目录

所以 Claude Code 的 spawn 语义从一开始就是显式的。

### 2. worker 工具池会重新装配

`AgentTool.tsx:568-577` 是一个非常关键的产品级决策：

子代理会根据自己的 permission mode 重新 `assembleToolPool(...)`，而不是直接继承父代理的工具集合。

这样做的意义很大：

- 父代理的审批历史不会直接泄露给子代理
- 不同 agent type 可以绑定不同能力边界
- 工具集可以随着 spawn 语义一起变化

这也是为什么 Claude Code 的多代理不是“共享一个世界”，而是“每个执行体拿到自己的局部世界”。

### 3. worktree 是官方隔离模式

`AgentTool.tsx:582-593` 会在需要时创建 `agent worktree`。后面的 `643-685` 则决定：

- 没有改动时自动清理
- 有改动则保留
- 如果是 hook-based worktree，则默认保留

这是很典型的 Runtime 思路。worktree 在这里不再是工程师手工使用的 git 技巧，而是 Runtime 提供的一种隔离执行面。

### 4. `LocalAgentTask` 接入统一任务框架

`tasks/LocalAgentTask/LocalAgentTask.tsx:197-260` 会把后台 agent 的结果包装成 `<task-notification>` XML：

- `task-id`
- `output-file`
- `status`
- `summary`
- 可选 `result`
- 可选 `usage`
- 可选 `worktree`

这段设计非常重要。因为它意味着多代理通信不是自由文本，而是结构化事件协议。

同一文件里还有另一个很关键的点：后台代理的状态会被统一记录成 task state，包含：

- `progress`
- `messages`
- `pendingMessages`
- `isBackgrounded`
- `retain`
- `diskLoaded`

换句话说，本地子代理在 Claude Code 里不是 Promise，而是 task。

### 5. Coordinator 把协作变成系统角色

`coordinator/coordinatorMode.ts:116-214` 直接把 coordinator 的系统提示写成一份调度手册：

- 什么时候应该启 worker
- 什么任务应该并行
- worker 完成后结果会怎样送回来
- 哪些事情不该委托

这说明 Claude Code 的多代理不是在 prompt 上临时拼出来的团队，而是 Runtime 官方支持的一种角色模式。

## 七、扩展系统

Claude Code 的另一块关键能力，是它把本地技能、远程 MCP 工具、MCP 资源、MCP 命令、MCP skill 都收进了同一套扩展平面。

### 1. Skill 是正式命令对象

`skills/loadSkillsDir.ts:237-264` 解析的 frontmatter 远超过教程版 `s05`：

- `allowed-tools`
- `arguments`
- `when_to_use`
- `version`
- `model`
- `disable-model-invocation`
- `user-invocable`
- `hooks`
- `context`
- `agent`
- `effort`
- `shell`

而 `loadSkillsDir.ts:270-399` 又把它真正转成一个 `Command`。

这里最关键的判断有两个。

第一，skill 不是纯知识块，它会改变执行语义。比如：

- `allowed-tools` 会改变这一轮权限
- `context: fork` 会影响执行上下文
- `model` / `effort` 会影响本轮模型策略

第二，Claude Code 明确区分本地 skill 和远程 skill 的信任边界。`loadSkillsDir.ts:371-395` 直接禁止 MCP skill 执行内联 shell，因为它们是 remote and untrusted。

这是一种非常成熟的 provenance-aware extension design。扩展不是平等的，来源决定信任等级。

### 2. MCP 是远程能力总线

`services/mcp/client.ts` 的规模已经说明，这一层远远不只是 `listTools` + `callTool`。

关键逻辑在 `mcp/client.ts:2169-2198` 和 `2226-2371`。

如果一个 MCP server 支持 resources，Claude Code 会并行拉取：

- tools
- commands
- mcp skills
- resources

然后再统一装配回来。

同时，它会处理：

- local vs remote server 的并发差异
- 401 needs-auth 缓存
- 无 token 发现路径
- `createMcpAuthTool(...)`
- 资源工具 `ListMcpResourcesTool` / `ReadMcpResourceTool`

`mcp/client.ts:2301-2318` 还专门处理了“最近返回过 401 的 server 暂时不重连”的情况。这不是细节优化，而是一个产品级判断：MCP 连接状态本身就是 Runtime 状态的一部分。

所以在 Claude Code 里，MCP 的含义是：

- 远程工具平面
- 远程资源平面
- 远程命令平面
- 远程 skill 平面
- 远程认证状态平面

这已经不是“插件系统”，而是能力图。

## 八、远程桥接

如果说 `main.tsx` 证明它是多宿主 Runtime，那么 `remote/RemoteSessionManager.ts` 和 `remote/sdkMessageAdapter.ts` 则说明，它并没有把远程会话简化成“把远端输出打印回来”。

### 1. `RemoteSessionManager` 管理控制协议

`RemoteSessionManager.ts:146-214` 这一段很能说明问题。

远端发回来的消息会被分成：

- `control_request`
- `control_cancel_request`
- `control_response`
- `SDKMessage`

其中 `control_request` 里最重要的一种是 `can_use_tool`。也就是说，远端执行体需要权限确认时，并不是自己随便处理，而是把 permission request 桥接回本地宿主。

这是一个很本质的设计决定：**执行宿主可以远程化，但控制权不能漂走。**

### 2. `sdkMessageAdapter` 负责语义映射

`sdkMessageAdapter.ts:21-148` 把远端 SDK 消息转换成本地 REPL 理解的：

- `AssistantMessage`
- `StreamEvent`
- `SystemMessage`
- `CompactBoundaryMessage`

这看起来像适配层，实际上是在做一件更重要的事：把远端运行时事件重新纳入本地 transcript 语义。

这一步一旦缺失，remote session 就只能是日志流；一旦做对，remote session 才能继续拥有：

- 本地滚动 UI
- 本地 compact 语义
- 本地 resume 语义
- 本地权限交互

这也是为什么 Claude Code 的远程能力不像“远程终端”，而更像“远程会话镜像”。

## 九、对照 `learn-claude-code`

`learn-claude-code` 很适合做骨架对照，因为它把 Claude Code 这台机器拆成了最小原型。

大致可以这么对应：

- `s01` 对应 `query.ts` 的 loop kernel
- `s02` 对应 `tools.ts` 的工具装配入口
- `s04` 对应 `AgentTool` / `runAgent` 的 fresh subagent 语义
- `s05` 对应 `loadSkillsDir.ts` 的按需技能注入
- `s06` 对应 `compact.ts` 的上下文折叠
- `s07-s12` 对应 task framework、background agents、team coordination、worktree isolation

但教程版和产品版之间有一条关键分界线。

教程版关注的是“范式最小成立”：

- loop 能不能闭合
- tool_use 能不能执行
- subagent 能不能隔离
- compact 能不能触发

Claude Code 关注的是“范式如何在真实产品里长期存活”：

- 权限边界怎么治理
- transcript 怎么恢复
- metadata 怎么落盘
- 子代理怎么 resume
- 扩展系统怎么分信任等级
- remote session 怎么桥接本地控制面

这就是从 Harness Demo 到 Runtime 的差别。

## 十、给 Agent 开发者的启发

如果要把这次拆解最后落到实践上，我觉得有六条结论最重要。

### 1. 先把自己当成运行时开发者

真正需要构建的是 Runtime。至少要把下面几层拆开：

- Host shell
- Turn assembler
- Query kernel
- Capability governance
- Continuity
- Concurrency
- Extension

如果这些层一开始就糊在一起，后面一定会被 compact、resume、task、remote 模式反噬。

### 2. 工具系统重在边界和可见性

Claude Code 最值得抄的，不是工具数量，而是它在工具进入模型视野之前就做过滤。

模型看见什么，往往比模型最终能不能调更重要。

### 3. Compact 重在状态迁移

一旦把 compact 理解成状态迁移，很多设计自然就会对：

- 为什么要有 compact boundary
- 为什么要重建 attachments
- 为什么要保留 tool discovery 状态

反过来，如果把它当摘要器来写，系统很快就会在长会话里变钝。

### 4. 多代理先有 Task Framework

如果没有：

- task id
- output file
- lifecycle state
- structured notification
- resume metadata

那么所谓多代理，本质上只是几个并发请求。

### 5. 扩展系统要分信任等级

本地 markdown skill、插件 skill、MCP skill、MCP resource、MCP tool，不应该是同一类东西。Claude Code 之所以稳，就是因为它承认扩展来源不同，信任等级就不同。

### 6. 尽早建立消息适配层和控制桥

这不是“等以后再重构”的问题。宿主壳一旦和内核黏死，远程化成本会非常高。

Claude Code 提前把：

- protocol message
- local transcript message
- permission request
- session metadata

都拆开了，所以它才能在 CLI、server、remote、ssh 之间切换。

## 十一、最后判断

Claude Code 2.1.88 最值得研究的，不是它“会不会写代码”，也不是它“工具多不多”。真正值得研究的是，它已经把编码 Agent 所需要的工程现实全部收进了一套 Runtime 里。

它知道：

- 一次请求不是一句 prompt，而是一个回合
- 一个回合不是一次 API 调用，而是一段状态机迭代
- 一个会话不是消息数组，而是可压缩、可恢复、可迁移的运行状态
- 一个子代理不是一句委托话术，而是一个可治理的执行体
- 一个扩展不是一段额外提示词，而是一块带来源、带权限、带信任边界的能力

这套理解，比任何单个技巧都重要。

因为对今天做 Agent 的工程师来说，真正的分水岭已经不是“会不会调模型”，而是有没有能力把模型放进一个设计良好的世界里，让它长时间、稳定地工作。

Claude Code 给出的答案很明确：模型负责智能，Runtime 负责世界。

而工程的难度，恰恰就在这个世界怎么搭。
