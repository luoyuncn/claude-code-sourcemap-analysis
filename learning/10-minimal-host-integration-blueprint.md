# 从 `learn-claude-code` 到 Claude Code 的最小宿主集成蓝图

## 1. 如果你要自己开发 Agent，应该复制什么，不该复制什么

研究 Claude Code 2.1.88 最容易犯的错，是试图一比一复刻它的全部工程细节。

这通常没有必要。

真正值得复制的，是它背后的运行时分层。换句话说，不是照着源码堆模块，而是提炼出一个能逐步长大的 Agent Host Blueprint。

这一篇就把前面几篇拆出来的结论，收敛成一套最小可落地架构。

## 2. 蓝图总览：一个可进化的 Agent Host 应该至少分 6 层

```text
Layer 0  Entry / Host Shell
         CLI / TUI / SDK / Server / Remote Adapter

Layer 1  Agent Runtime Kernel
         query loop / message model / tool registry / prompt reflection

Layer 2  Governance
         permission / sandbox / working dirs / capability filtering

Layer 3  Continuity
         compact / transcript / memory / title / resume / metadata

Layer 4  Concurrency
         subagents / tasks / notifications / worktree / teammate / remote task

Layer 5  Extension
         skills / MCP / resources / auth / commands / external capability graph
```

`learn-claude-code` 基本覆盖了 Layer 1 到 Layer 4 的教学版骨架；Claude Code 2.1.88 则把六层全部做成了产品化实现。

## 3. 第一步：先做最小 Kernel，不要一上来做“复杂 Agent 平台”

### 3.1 最小 Kernel 只需要四件事

如果你现在从零开发一个 coding agent，第一阶段只做这些就够了：

1. `while stop_reason == tool_use` 主循环
2. 独立的工具 dispatch map
3. 基础 transcript
4. 至少一个 fresh subagent

这正是 `learn-claude-code` 的 `s01-s04` 在教的东西。

### 3.2 这个阶段的目标不是“聪明”，而是“边界清楚”

你需要先回答清楚：

- 消息如何追加
- 工具结果如何回写
- 子代理是否污染父上下文
- transcript 如何落盘

如果这些边界没建立，后面加 memory、task、MCP 只会越来越乱。

## 4. 第二步：把工具系统升级成“能力治理层”

当最小 Kernel 跑通后，下一步不要急着加更多工具，而应该先补治理层。

### 4.1 需要补上的不是 handler，而是 visibility 与 permission

Claude Code 最值得借鉴的一点，是它把工具分成两个问题：

1. 工具会不会执行
2. 工具会不会先被模型看到

真实产品里，第二个问题更重要。

### 4.2 最小可复制做法

建议至少实现：

- 工具 schema 与元数据分离
- allow / deny 规则
- 按模式过滤工具池
- 独立的 worker tool pool 组装

做到这一步，你的 Agent 就从“能用工具”进入“能被治理地用工具”。

## 5. 第三步：尽早补齐 Continuity，而不是等 context 爆了再说

很多团队会把 compact、resume、memory 当后期优化项。Claude Code 的源码正好反过来说明：

它们是 Runtime 生存能力。

### 5.1 最小 Continuity 套件

建议按这个顺序补：

1. transcript append-only 落盘
2. compact boundary
3. summary + transcript path
4. title / tag / metadata cache
5. `/resume`
6. 文件化 memory

### 5.2 这里最该学 Claude Code 的是什么

不是它具体怎么压缩，而是它对“持久化分层”的理解：

- 长期记忆
- 会话元数据
- 当前工作态

三者必须拆开。

## 6. 第四步：多代理一定要先做 Task Framework，再做“Agent Team”

很多人做多代理时，一上来就是：

- leader
- planner
- coder
- reviewer

但 Claude Code 的实现提醒我们，真正先要补的是任务框架。

### 6.1 任务框架至少应包含

- `taskId`
- `status`
- `progress`
- `outputFile`
- `notification protocol`
- `abortController`
- `resume metadata`

### 6.2 没有这些，多代理只会变成“并发调用模型”

真正的 Agent Runtime 多代理，关注的是：

- 生命周期
- 恢复
- 输出观测
- 中途继续对话
- 任务切前台/后台

这就是 Claude Code 的 `LocalAgentTask`、`RemoteAgentTask`、`AgentTool` 真正在做的事情。

## 7. 第五步：扩展系统要从一开始就按“多来源能力图”设计

### 7.1 不要把 skills 和 MCP 设计成两个平行世界

Claude Code 的一个高明之处，是本地 skill 与远程 MCP 虽然信任等级不同，但最终都进入同一能力平面：

- tool
- command
- skill
- resource

这会让后续的：

- prompt reflection
- permission
- UI 展示
- telemetry

都更统一。

### 7.2 最小扩展系统路线

推荐分三步做：

1. 本地 markdown skill
2. 多目录来源 + frontmatter 元数据
3. MCP 接入工具 / 资源 / 认证

如果一开始就做“插件市场”，通常会把简单问题做复杂。

## 8. 第六步：宿主壳要后置，但内核必须为它预留接口

Claude Code 能长出：

- REPL
- server
- direct connect
- ssh remote
- remote session viewer

不是因为它后期魔改成功，而是因为核心运行时本来就没有和某一个宿主绑死。

### 8.1 你的内核至少要预留这些接口

- message adapter
- task notification channel
- permission bridge
- transcript reader / writer
- session metadata store

只要这几层接口独立，后面换宿主会轻松很多。

## 9. 一个现实可执行的演进路线

### 阶段 A：最小单代理版

对应 `learn-claude-code s01-s04`

- query loop
- tools
- transcript
- fresh subagent

### 阶段 B：可治理版

对应 Claude Code 的工具与权限层

- tool visibility
- permission mode
- working directory model
- sandbox

### 阶段 C：可续航版

对应 compact / memory / resume

- transcript summary
- compact boundary
- metadata tail-stamping
- file memory

### 阶段 D：可并发版

对应 AgentTool + Task Framework

- background task
- progress tracking
- structured notifications
- worktree isolation

### 阶段 E：可扩展版

对应 Skills + MCP

- markdown skills
- frontmatter behavior
- remote resources
- auth / reconnect

### 阶段 F：多宿主版

对应 server / remote / ssh

- message adapter
- remote permission bridge
- session host

## 10. 最值得避免的 5 个误区

### 误区 1：把 Agent Runtime 做成一堆 prompt 模板

Claude Code 的价值不在 prompt 写得多花，而在它有完整状态层、治理层和恢复层。

### 误区 2：只做工具，不做能力治理

这会让系统早期看起来很能干，后期很难上线。

### 误区 3：只做 summary，不做 resume metadata

最终用户会发现会话看起来能恢复，但工作态其实早就断了。

### 误区 4：多代理只做并发，不做任务抽象

这样只能得到几个同时跑的 LLM 调用，得不到真正可管理的 worker。

### 误区 5：MCP 只当插件，不当远程能力平台

最后一定会在 auth、resource 和 reconnect 上补一堆临时逻辑。

## 11. 对开发 Agent 的人的一个核心建议

如果你只能从 Claude Code 里学一件事，那就学这件事：

**不要把自己当成“在写一个会调用模型的产品功能”，而要把自己当成“在写一个让模型能长期工作、可被治理、可被恢复、可被扩展的 Runtime”。**

这是 Claude Code 和大部分“AI 功能”代码最本质的差别。

## 12. 一句话总结

`learn-claude-code` 教我们怎样从零搭起 Agent Harness；Claude Code 2.1.88 教我们怎样把这套 Harness 演化成真正的 Agent Host。

前者适合入门，后者适合定架构。
