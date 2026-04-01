# 工具治理与权限系统：Claude Code 为什么不是“给模型一堆函数”

## 1. Claude Code 的工具系统有两个关键词

很多 Agent 系统把工具理解成：

- 一个 JSON schema 列表
- 一个名字到 handler 的映射

Claude Code 2.1.88 当然也有这两层，但它更重要的两点是：

1. 工具是类型化能力
2. 工具是受治理能力

这意味着它在设计上关心的不只是“模型能不能调这个工具”，还关心：

- 工具默认是否安全
- 模型这一轮是否应该看见这个工具
- 权限规则是否需要前置到工具发现阶段
- 子代理和主线程是否应该拿到同一组工具

## 2. `Tool.ts`：统一工具抽象

### 2.1 `ToolUseContext` 把工具调用从“裸函数调用”升级成“上下文调用”

`Tool.ts:123-138` 定义了 `ToolPermissionContext`，把权限模式、allow/deny/ask 规则、额外工作目录、auto mode 能力都纳入同一对象。

`Tool.ts:158-220` 定义了 `ToolUseContext`，把工具执行所需的一切放进去：

- tools
- commands
- mcpClients / mcpResources
- agentDefinitions
- `getAppState` / `setAppState`
- `abortController`
- `appendSystemMessage`
- `sendOSNotification`
- 记忆附件相关状态

这等于明确了一件事：**工具调用是 Runtime 行为，不是普通函数调用。**

### 2.2 `buildTool()` 体现了 fail-closed 思想

`Tool.ts:703-791` 的 `buildTool()` 很值得学。它给工具定义补齐了一批默认值：

- `isConcurrencySafe` 默认 `false`
- `isReadOnly` 默认 `false`
- `isDestructive` 默认 `false`
- `checkPermissions` 默认 allow，但统一走权限系统
- `userFacingName` 默认取 `name`

注意这里的重点不是默认 allow，而是“所有工具都必须经过统一约束入口”。这让 Claude Code 后续能统一处理：

- UI 展示
- 权限提示
- 自动分类器
- 只读 / 破坏性标记

## 3. `tools.ts`：模型可见工具集是在运行时算出来的

### 3.1 `getAllBaseTools()` 是能力全集，不是实际工具集

`tools.ts:193-250` 展示了一个非常大的 built-in tool 列表：

- Agent
- Bash / Read / Edit / Write
- WebFetch / WebSearch
- Todo / Task 系列
- Skill
- MCP 资源工具
- SendMessage / TaskStop
- Worktree
- REPL
- PowerShell

但这里返回的是“理论上可用”的全集，不是模型当前真的能用的集合。

### 3.2 `getTools()` 把模式过滤和 deny rule 过滤都做了

`tools.ts:271-327` 会根据环境和权限做筛选：

- `CLAUDE_CODE_SIMPLE` 只保留简化工具
- REPL 模式下隐藏 primitive tools
- blanket deny rule 命中的工具直接过滤掉
- 再根据 `isEnabled()` 做最后一轮裁剪

这就很关键了：**权限不只发生在调用时，也发生在展示时。**

对模型来说，“看不见某个工具”和“看见但被拒绝”是两种完全不同的行为信号。Claude Code 显然更偏向前者。

### 3.3 `assembleToolPool()` 是工具治理的中心

`tools.ts:329-367` 的 `assembleToolPool()` 负责：

1. 拿 built-in tools
2. 过滤 deny rule 命中的 MCP tools
3. 按名字排序
4. 去重

这里还有一个非常工程化的细节：为 prompt cache 稳定性保持 built-in tools 的有序前缀。说明 Claude Code 在工具治理时，已经把 cache 命中率也算进去了。

## 4. 权限系统不是“一个弹窗”，而是一张上下文网

### 4.1 `permissionSetup.ts` 先构造权限上下文，再跑系统

`permissionSetup.ts:913-1010` 展示了 `additionalWorkingDirectories`、`isBypassPermissionsModeAvailable`、disk rules、CLI allow/disallow tools 的组合方式。

重点有三个：

1. 当前工作目录不是唯一工作目录
2. allow/disallow CLI 参数会改写工具可见面
3. 权限上下文是多源合并的结果

也就是说，Claude Code 的权限不是某个单独模块里的一个枚举，而是一个由：

- CLI
- settings
- disk rules
- mode
- symlink / working directory

共同决定的上下文对象。

### 4.2 auto mode 也是被治理的，不是简单开关

`permissionSetup.ts:1116-1157` 里，auto mode 是否可进入要看：

- GrowthBook gate
- settings
- 当前模型是否支持
- fast mode breaker 是否触发

这表明 Claude Code 已经把权限模式做成运行时能力矩阵，而不是布尔开关。

## 5. 子代理的工具池和父代理不是同一套

`AgentTool.tsx:568-577` 说明 worker 的工具池会用它自己的 permission context 重新 `assembleToolPool(...)`，而不是直接继承父工具集。

这是一个特别重要的设计点：

- 父代理受到的限制，不一定应该传给子代理
- 子代理应当有自己的 permission mode
- 但 fork path 又可能为了 prompt cache 命中刻意继承 exact tools

Claude Code 在这里没有“一刀切”，而是把工具治理和 spawn 策略绑在了一起。

## 6. Skill 和工具权限还能联动

`REPL.tsx:2701-2726` 会把 skill frontmatter 的 `allowed-tools` 写进 `alwaysAllowRules.command`。`loadSkillsDir.ts:242-264` 也把 `allowed-tools` 作为正式 frontmatter 字段解析出来。

这意味着 skill 在 Claude Code 里不是纯文本知识块，它还会改变这一轮工具权限边界。

这是一个很值得借鉴的设计：

- skill = 知识模板
- skill 也 = 工具白名单模板

让“如何做事”和“允许做什么事”在一个能力单元里绑定起来。

## 7. 和 `learn-claude-code` 的关系

`learn-claude-code/s02` 告诉我们：加一个工具，本质上只是给 dispatch map 加一个 handler。

Claude Code 没有否定这个结论，只是在其外面加了四层现实世界机制：

1. 类型与默认行为
2. 模式过滤
3. 权限治理
4. 工具池装配与缓存稳定性

所以可以把它理解为：

- 教程版回答“工具如何工作”
- Claude Code 回答“工具如何在产品里稳定、安全地工作”

## 8. 对 Agent 开发者的启发

### 启发 1：工具定义和工具治理必须分层

先有“这个工具做什么”，再有“这个工具何时可见、何时可调、何时必须审批”。

### 启发 2：权限前置比权限后置更重要

让模型一开始就少看见不该用的工具，往往比事后拒绝更稳、更省 token，也更少误导。

### 启发 3：子代理必须有独立工具边界

多代理系统里，如果所有代理都共用同一套工具与同一权限视图，很容易让系统失控。

### 启发 4：能力系统要为 cache、UI 和 telemetry 预留位置

Claude Code 的工具抽象之所以稳，是因为它从一开始就没把工具当成“纯业务函数”。它把产品面、性能面和治理面都揉进去了。
