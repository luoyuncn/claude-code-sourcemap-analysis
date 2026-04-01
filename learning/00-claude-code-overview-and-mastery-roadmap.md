# Claude Code 2.1.88 总览与学习路线图

## 1. 这次拆解到底在看什么

这个仓库不是官方源码仓库，而是基于 `@anthropic-ai/claude-code@2.1.88` 的 source map 还原结果。`README.md` 已经明确说明了它的研究性质，同时给出了我们最重要的入口目录：`restored-src/src/`。

如果只看目录层级，Claude Code 很像一个“大而全的 CLI 应用”；但如果把它放回 Agent 视角，它更像一个分层很深的“Agent Runtime”：

1. `main.tsx` 负责入口改写、模式分流、CLI 编排。
2. `replLauncher.tsx` 和 `screens/REPL.tsx` 负责交互层、状态层、消息层。
3. `query.ts` 和 `QueryEngine.ts` 负责主循环、tool-use 驱动、恢复与压缩。
4. `Tool.ts`、`tools.ts`、`tools/*` 负责能力边界。
5. `tasks/*`、`tools/AgentTool/*`、`coordinator/*` 负责多代理与任务化执行。
6. `skills/*`、`services/mcp/*`、`memdir/*`、`services/compact/*` 负责扩展、记忆和上下文续航。

一句话概括：Claude Code 2.1.88 已经不是“给模型包一层 Shell 工具”了，而是把编码 Agent 运行时需要的几乎所有工程机制都系统化了。

## 2. 它和 `learn-claude-code` 是什么关系

`D:\dev\github\learn-claude-code` 的价值，不是复刻 Claude Code 的实现细节，而是给我们提供一个“最小范式字典”：

- `s01` 告诉我们 Agent 的最小单位是 `while + stop_reason == tool_use`
- `s02` 告诉我们工具只是 dispatch map，不该污染主循环
- `s04` 告诉我们子代理的本质是上下文隔离
- `s05` 告诉我们技能应该按需注入，而不是全部塞进 system prompt
- `s06` 告诉我们上下文压缩不是优化项，而是长会话生存条件
- `s07-s12` 告诉我们任务系统、后台执行、团队协议、自治与 worktree 隔离，会把 Agent 从“单线程助手”变成“工程系统”

Claude Code 2.1.88 基本把这 12 节最小范式都做成了生产化版本，只是实现方式更复杂、约束更多、与 UI/权限/远程执行耦合更深。

## 3. 读源码时最值得盯住的 8 条主线

### 3.1 入口主线

`main.tsx` 并不只是在解析命令行。它还做了：

- URL/深链改写
- `assistant`、`ssh`、`connect` 等模式剥离
- interactive / print / sdk / remote 判定
- REPL 与 server 两条运行面切换

这说明 Claude Code 从设计上就不是单一 CLI，而是多入口、多模式共享同一套 Agent 核心。

### 3.2 Agent 主循环主线

真正的 Agent Loop 不在 `main.tsx`，而在：

- `screens/REPL.tsx` 的交互式调用路径
- `query.ts` 的核心循环
- `QueryEngine.ts` 的 SDK / headless 封装

它们一起构成了“UI 上下文”和“模型驱动循环”之间的桥。

### 3.3 工具治理主线

`Tool.ts` + `tools.ts` 的关键价值，不是“把工具注册起来”，而是：

- 用统一类型约束 tool schema / permission / progress / result
- 把 deny rule 前移到“模型可见性”阶段
- 把 built-in tool、MCP tool、agent-specific tool 放进同一个池子里治理

这是典型的“能力不是平铺，而是被策略包裹”的设计。

### 3.4 多代理主线

`tools/AgentTool/*`、`tasks/*`、`coordinator/coordinatorMode.ts` 表明 Claude Code 的多代理不是一个外挂功能，而是运行时核心能力：

- 可以同步
- 可以后台
- 可以 fork 自身上下文
- 可以 worktree 隔离
- 可以 remote session 化
- 可以 in-process teammate 化

### 3.5 上下文续航主线

`services/compact/compact.ts` 不是单纯摘要器，而是：

- 压缩器
- 状态搬运器
- prompt cache 命中优化器
- 计划/技能/异步任务“附件保留器”

换句话说，Claude Code 不是“压缩聊天记录”，而是在压缩后重建一个还能继续工作的 Agent 状态。

### 3.6 记忆主线

`memdir/*` 说明它已经把长期记忆做成文件系统协议，而不是隐式 embedding cache：

- 有 `MEMORY.md` 入口索引
- 有类型约束
- 有 team/private 作用域
- 有大小与行数上限

这更像知识操作系统，而不是一个“帮你记点事”的 feature。

### 3.7 扩展主线

`skills/loadSkillsDir.ts` 与 `services/mcp/client.ts` 组合在一起后，Claude Code 的扩展系统有三层：

1. 本地 skill / command
2. MCP tool / resource / prompt
3. agent frontmatter / plugin agent / managed policy

这意味着“扩展 Agent”并不等于“给它加插件”，而是给它扩展知识、工具、资源、代理定义与执行上下文。

### 3.8 会话恢复主线

`utils/sessionStorage.ts` 非常关键。它不是简单记日志，而是在维护：

- transcript
- summary
- custom title / ai title
- tag
- worktree session
- remote task / sidechain / agent metadata

它支撑了 `/resume`、任务恢复、标题恢复、压缩后续跑等一整套体验。

## 4. 推荐学习顺序

### 第一阶段：先掌握“最小内核”

1. `learn-claude-code/docs/zh/s01-the-agent-loop.md`
2. `learn-claude-code/docs/zh/s02-tool-use.md`
3. `learn-claude-code/docs/zh/s04-subagent.md`
4. `learn-claude-code/docs/zh/s05-skill-loading.md`
5. `learn-claude-code/docs/zh/s06-context-compact.md`

先把“Agent Loop 是什么”建立起来，再看 Claude Code 会轻松很多。

### 第二阶段：回到 Claude Code 看运行时骨架

1. `restored-src/src/main.tsx`
2. `restored-src/src/replLauncher.tsx`
3. `restored-src/src/screens/REPL.tsx`
4. `restored-src/src/query.ts`
5. `restored-src/src/QueryEngine.ts`

这一步的目标不是看完，而是画出消息流和状态流。

### 第三阶段：看能力层与策略层

1. `restored-src/src/Tool.ts`
2. `restored-src/src/tools.ts`
3. `restored-src/src/utils/permissions/*`
4. `restored-src/src/entrypoints/sandboxTypes.ts`

这一步理解的是“模型能做什么，为什么能做，以及什么时候不能做”。

### 第四阶段：看高级 Agent 能力

1. `restored-src/src/tools/AgentTool/*`
2. `restored-src/src/tasks/*`
3. `restored-src/src/coordinator/coordinatorMode.ts`
4. `restored-src/src/remote/*`

这一步能看清 Claude Code 的多代理范式到底和教程版相差多少。

### 第五阶段：看可持续运行能力

1. `restored-src/src/services/compact/*`
2. `restored-src/src/memdir/*`
3. `restored-src/src/utils/sessionStorage.ts`

这一步理解为什么 Claude Code 能从“会话助手”进化到“持久 Agent Runtime”。

### 第六阶段：看扩展生态

1. `restored-src/src/skills/*`
2. `restored-src/src/services/mcp/*`
3. `restored-src/src/plugins/*`

这一步对应的是“如何把你的 Agent 做成平台”。

## 5. 对 Agent 开发者最重要的总启发

### 启发 1：真正难的不是 Loop，而是 Runtime

`learn-claude-code` 已经证明，最小 Agent Loop 非常短。但 Claude Code 2.1.88 告诉我们，真实产品难点在：

- 模式切换
- 权限治理
- 上下文续航
- 会话恢复
- 远程执行
- 多代理调度
- 扩展协议

所以开发 Agent 时，不要高估“prompt engineering”，也不要低估 runtime engineering。

### 启发 2：能力必须被策略包裹

Claude Code 没有把工具当成“模型可任意调用的函数列表”，而是把它们放在：

- permission mode
- deny/allow rule
- sandbox
- worktree
- additional directories
- mode-aware filtering

这些边界之中。

对工程 Agent 来说，安全和能力从来不是两套系统，而是一套系统的两面。

### 启发 3：长程记忆的核心不是“存”，而是“恢复可工作状态”

摘要、记忆、标题、技能附件、计划附件、异步任务附件，这些设计共同说明了一件事：

Claude Code 在乎的不是“把历史保留下来”，而是“让模型在丢掉大部分历史后仍然知道自己下一步该做什么”。

这比普通聊天产品对“上下文管理”的要求高得多。

### 启发 4：最强的产品形态不是单 Agent，而是可组合 Agent Runtime

Claude Code 的形态已经明显超出“终端里的一个助手”：

- 它可 server 化
- 可 connect 化
- 可 remote 化
- 可 ssh 化
- 可 fork / background / team 化

这说明未来的 Agent 产品，更像“操作系统”而不是“单点功能”。

## 6. 本目录后续文档怎么读

- `01` 看运行时状态与循环
- `02-main` 看主 Agent 的装配过程
- `03-05` 看能力边界与资源模型
- `06-08` 看扩展、记忆、多代理
- `09` 看多入口与外围模式
- `10` 看如何把这些抽成自己的最小实现
- `11` 看适合对外传播的公众号文章版总结

如果只想抓核心，优先看 `01`、`03`、`06`、`07`、`08`、`10`。
