# 为什么说 Claude Code 不是一个 CLI，而是一套 Agent Runtime

如果你把 Claude Code 2.1.88 只看成“一个会调模型、会跑 Bash 的命令行工具”，那其实只看到了它最表面的那层壳。

这次沿着 `restored-src/src/` 往下拆，我越来越确定一件事：Claude Code 最值得学习的，不是某个 prompt 技巧，也不是某个工具定义，而是它已经把编码 Agent 做成了一套完整的 Runtime。

## 一、从 `learn-claude-code` 往上看，最容易看懂 Claude Code

`learn-claude-code` 这套仓库其实特别有价值，因为它把 Claude Code 背后的范式拆成了最小单元：

- `s01` 讲清楚 Agent Loop
- `s02` 讲清楚 Tool Use
- `s04` 讲清楚 Subagent 的本质是上下文隔离
- `s05` 讲清楚 Skill 应该按需加载
- `s06` 讲清楚 Compact 是长会话的生存条件
- `s07-s12` 讲清楚任务系统、后台执行、团队协作、自治与 worktree 隔离

如果说 `learn-claude-code` 是教科书版，那么 Claude Code 2.1.88 就是生产版。

区别不在“原理变了”，而在“同一套原理被推进到了工程系统级别”。

## 二、Claude Code 真正的主干，不在 CLI，而在 Runtime 内核

它的真正主干链路大致是：

- `main.tsx` 负责入口改写和模式分流
- `REPL.tsx` 负责交互状态装配
- `query.ts` / `QueryEngine.ts` 负责 Agent Loop
- `Tool.ts` / `tools.ts` 负责能力治理
- `AgentTool.tsx` / `tasks/*` 负责多代理与任务化执行
- `compact.ts` / `sessionStorage.ts` / `memdir.ts` 负责续航与恢复
- `loadSkillsDir.ts` / `services/mcp/client.ts` 负责扩展系统

这意味着 Claude Code 的核心问题早就不是“怎么调用一次模型”，而是：

- 如何持续运行
- 如何治理工具
- 如何恢复会话
- 如何扩展能力
- 如何托管多个代理

这正是 Runtime 的问题域。

## 三、它最强的地方，是把 Agent 的“工程现实”全部显式化了

很多 demo 型 Agent，默认世界非常单纯：

- 当前目录就是工作区
- 工具看得到就能调用
- 子代理只是再跑一次 loop
- 对话太长就随便总结一下
- 远程能力就是多一个 API

Claude Code 不是这样。

它把这些事情都显式建模了：

- working directory 是资源边界
- permission mode 是能力治理层
- subagent 有 fresh、fork、background、remote 等不同语义
- compact 后要重建 plan、skill、tool delta、async agent 状态
- session title、tag、worktree、PR link 要能 append-only 恢复
- MCP 不只是工具列表，而是带认证、资源、技能、重连机制的能力总线

你会发现，Claude Code 的工程重点从来不在“让模型多想一步”，而在“让模型所处的世界足够稳定、可控、可恢复”。

## 四、这套 Runtime 最值得普通开发者学的 4 件事

### 1. 不要把工具系统做成 dispatch map 就停下

dispatch map 只是开始。更重要的是：

- 工具何时可见
- 工具何时可调用
- 子代理的工具池是否独立
- skill 是否能绑定 allowed-tools

Claude Code 真正厉害的地方，是把 tool system 做成了 capability governance。

### 2. Compact 的目标不是省 token，而是保住工作态

教程版 compact 的重点是“无限会话”；Claude Code 的重点是“压缩完还能继续干活”。

这就是为什么它会在 compact 后重新注入：

- plan
- skill
- deferred tools
- agent listing
- MCP instructions

这不是摘要器，这是状态搬运器。

### 3. 多代理不是多开几个模型，而是任务运行时

Claude Code 的多代理体系里，有：

- `AgentTool`
- `LocalAgentTask`
- `RemoteAgentTask`
- `coordinatorMode`
- `task-notification` 协议

这说明真正的多代理能力，不是“并发开几个请求”，而是：

- 有任务 ID
- 有状态
- 有输出文件
- 有通知协议
- 可 resume
- 可 worktree 隔离

### 4. 扩展系统要统一成能力图

本地 skill、MCP tool、MCP resource、MCP skill、command，在 Claude Code 里都不是散装功能，而是统一进入 Runtime 能力平面。

这会让你的系统在后期非常受益，因为：

- prompt 能统一反射
- UI 能统一展示
- 权限能统一治理
- resume 能统一恢复

## 五、Claude Code 给开发 Agent 的人最大的启发

我觉得最大启发其实只有一句话：

**不要把自己理解成“在做一个 AI 功能”，而要把自己理解成“在做一个能让模型稳定工作的 Runtime”。**

一旦这个视角成立，很多设计决策会自然变清楚：

- 为什么要有 transcript
- 为什么要有 metadata
- 为什么要有 compact boundary
- 为什么要有 task framework
- 为什么要有 worktree isolation
- 为什么要把 MCP 当成能力总线

因为这些都不是装饰，它们是 Runtime 的基础设施。

## 六、`learn-claude-code` 和 Claude Code，应该怎么一起学

我的建议非常直接：

第一遍先看 `learn-claude-code`，把范式学会。

第二遍再看 Claude Code 2.1.88，对着源码找这些范式在真实产品里是如何被扩展成：

- 状态层
- 治理层
- 恢复层
- 任务层
- 宿主层

这样你不会被源码体量淹没，也不会只记住一些零散技巧。

## 七、最后的判断

Claude Code 值得研究的地方，不是因为它“功能很多”，而是因为它代表了一种非常成熟的 Agent 产品思路：

模型负责智能，Runtime 负责世界。

而真正强大的 Agent 产品，拼到最后，拼的恰恰是这个世界搭得够不够好。
