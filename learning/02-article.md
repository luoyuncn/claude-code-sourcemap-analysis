# 拆 Claude Code 2.1.88：从最小 Agent Loop 到工程化 Agent Runtime

## 一、先说结论

如果只用一句话总结 Claude Code 2.1.88，我会这么说：

**它不是“一个会写代码的聊天机器人”，而是一套围绕模型构建的工程化 Agent Runtime。**

`learn-claude-code` 已经把最小范式讲得很清楚：Agent 的本质是一个循环，模型决定何时调用工具、何时结束；Harness 决定模型能看到什么、能做什么、怎样恢复、怎样协作。Claude Code 2.1.88 的意义，不在于它发明了新的 Agent 原理，而在于它把这套原理做成了产品级运行时。

它的主线并不复杂：

1. 入口分流
2. 组装本轮上下文
3. 让模型在 `query()` 里跑 tool-use loop
4. 把工具、权限、记忆、MCP、任务、多代理、远程会话等工程问题都包在 Runtime 里

换句话说，Claude Code 的真正护城河不只是模型能力，而是它把“模型如何在真实工程环境里持续工作”这件事做深了。

## 二、最小范式没有变，变的是工程层数

在 `learn-claude-code/s01` 里，最小 Agent Loop 只有不到 30 行。到了 Claude Code 2.1.88，这个循环依然存在，但它被拆进了多层结构：

- `main.tsx` 决定会话从哪里进来
- `REPL.tsx` 决定当前轮次拿什么上下文
- `query.ts` 决定这轮怎么跑
- `QueryEngine.ts` 决定 headless / SDK 怎么复用同一个内核

这说明一个非常重要的事实：

**真正让 Agent 产品难做的，不是 Loop 本身，而是 Loop 外围那一整圈运行时。**

你要处理的从来不只是“模型会不会调用工具”，而是：

- 这轮工具列表是不是最新的
- 权限是否允许模型看见这个工具
- MCP 服务器是不是刚连上
- 上下文是否已压缩
- 技能是否该按需注入
- 子代理是否该继承上下文还是重新开局
- 用户是否在远程会话里
- 本轮状态是否需要写进 transcript，供之后 resume

Claude Code 2.1.88 的成熟，恰恰体现在这些外围问题都已经不是补丁，而是结构的一部分。

## 三、它真正强的地方：把“能力”做成“受治理的能力”

很多 Agent Demo 的工具系统只是一个 JSON schema 列表。Claude Code 不是。

从 `Tool.ts`、`tools.ts`、`permissionSetup.ts` 可以看出，它把工具系统做成了三层结构：

1. 工具定义层：schema、默认行为、progress、result
2. 工具池装配层：built-in、MCP、agent-specific tools 合并
3. 权限治理层：deny/allow rule、mode、sandbox、workspace boundary

它甚至会在模型发起调用之前，就先把被 deny rule 命中的工具从“可见工具集”里拿掉。也就是说，权限不是事后弹窗，而是前置塑形。

这给 Agent 开发者一个很重要的提醒：

**不要把工具当成模型的自由函数集。工具应该是带边界、带策略、带上下文的系统接口。**

在 Claude Code 这里，工具不是孤立存在的。工具总是和这些东西一起出现：

- `ToolUseContext`
- `ToolPermissionContext`
- sandbox config
- additional working directories
- current mode
- agent scope

也正因为如此，Claude Code 才能同时支持普通主线程、后台代理、fork 子代理、teammate、remote session 和 MCP server，而不至于权限体系崩掉。

## 四、上下文管理已经从“摘要”进化成“状态续航”

Claude Code 的另一个巨大价值，在于它几乎把“上下文管理”做成了一门独立工程学。

很多人把 compact 理解成“聊天太长了，做个摘要”。这在 Claude Code 里只说对了一半。

`services/compact/compact.ts` 展示的其实是一套更高级的机制：

- 在压缩前先生成 summary
- 清理 read file cache
- 重新附加 file attachments
- 重新附加 async agent 状态
- 重新附加 plan attachment
- 重新附加 invoked skills
- 重新附加 MCP 指令增量
- 重写 compact boundary
- 重新把 session metadata 放回 transcript 尾部

这背后的思想非常强：

**压缩的目标不是“变短”，而是“压缩后依然能继续干活”。**

这和普通聊天摘要完全不是一个层次。

同样的思想在 `sessionStorage.ts` 里也很明显。它不只保存 transcript，还区分：

- `custom-title`
- `ai-title`
- `summary`
- `tag`
- task summary
- subagent metadata
- worktree session metadata

为什么要这么麻烦？因为一个真正可恢复的 Agent，会话恢复时需要恢复的不是“聊天文本”，而是“工作现场”。

## 五、多代理不是外挂，而是 Claude Code 的第二主线

从 `tools/AgentTool/*`、`tasks/*`、`coordinator/coordinatorMode.ts` 往下看，你会发现 Claude Code 的多代理设计已经非常成熟：

- `AgentTool` 负责 spawn 入口
- `runAgent.ts` 负责装配子代理的 system prompt、tool pool、agent context
- `LocalAgentTask` 负责本地后台代理
- `InProcessTeammateTask` 负责同进程队友
- `RemoteAgentTask` 负责远程会话型代理
- `coordinatorMode.ts` 负责给主代理注入“协调者角色”

尤其值得注意的是两种 spawn 语义：

1. `subagent_type` 明确指定时，通常是 fresh context
2. `subagent_type` 省略时，在 fork gate 开启下会走 inherited context

这实际上已经把“任务委派”和“上下文复制”做成了两种不同机制，而不是一个模糊的子任务调用。

配合 `LocalAgentTask` / `RemoteAgentTask` 的 task state，Claude Code 让代理执行具备了：

- 可视化进度
- 可停止
- 可 background
- 可 resume
- 可输出文件追踪
- 可在 compact 后继续被主线程感知

这比 `learn-claude-code` 的 s04/s08/s09/s10/s11/s12 更复杂，但精神是同一条线：**把协作从 prompt 技巧，升级成 runtime 协议。**

## 六、Skills + MCP：它已经不是一个 Agent，而是一个平台

如果说多代理让 Claude Code 更像“操作系统”，那 Skills 和 MCP 则让它更像“平台”。

`skills/loadSkillsDir.ts` 说明本地 skill 已经具备这些能力：

- frontmatter 解析
- allowed-tools 限制
- argument substitution
- shell injection
- `context: fork`
- `agent`、`effort`、`model` frontmatter

而 `services/mcp/client.ts` 则展示了另一套更平台化的能力：

- stdio / SSE / streamable HTTP / WebSocket 多传输
- OAuth / auth cache / retry
- MCP tools / prompts / resources
- MCP skill builder
- 输出截断与二进制持久化

更妙的是，这两套系统并不是平行存在，而是互相打通的：

- MCP 可以带来 tool
- MCP 也可以带来 skill
- 但 MCP skill 又被当成远程不可信内容，所以禁止执行内联 shell

这说明 Claude Code 的扩展系统已经不再是“插件机制”，而是“可治理的扩展计算面”。

## 七、对做 Agent 的人，最该学什么

### 1. 学结构，不要学表面 feature

Claude Code 有 voice、buddy、ssh、server、connect、assistant、remote review、ultraplan 等大量 feature，但真正值得学的是它背后的结构：

- 多入口共享一个 query runtime
- 工具系统与权限系统一体化
- 压缩系统与恢复系统一体化
- 多代理系统与任务系统一体化
- 扩展系统与信任边界一体化

如果只学 feature list，很快会陷入“抄功能”。如果学结构，就能迁移到你自己的领域。

### 2. 把 Runtime 当成产品本体

很多人以为 Agent 产品的产品力来自提示词。Claude Code 告诉我们，真正的产品力更多来自 Runtime：

- 什么时候弹权限
- 工具是否自动裁剪
- 压缩后还能不能继续工作
- resume 能否找回标题和工作状态
- 子代理能否继续上下文
- worktree 是否自动清理

这些才是真实用户每天都在感知的体验。

### 3. 先做最小范式，再逐层工程化

学习顺序上最靠谱的方法，依然是：

1. 先用 `learn-claude-code` 建立最小心智模型
2. 再用 Claude Code 2.1.88 理解真实工程问题
3. 最后抽出适合你自己产品的那一层

不要上来就试图复刻完整 Claude Code。正确的做法是：

- 先做 `s01-s06`
- 再补 `s07-s08`
- 再判断你是否真的需要 `s09-s12`

## 八、最后一句话

Claude Code 2.1.88 最值得学习的，不是它有多“聪明”，而是它如何把一个聪明模型放进一个能长期、稳定、受控、可恢复、可扩展地工作的工程环境里。

这正是 Agent 时代真正稀缺的能力：

**不是再发明一个 Loop，而是把 Loop 托举成 Runtime。**
