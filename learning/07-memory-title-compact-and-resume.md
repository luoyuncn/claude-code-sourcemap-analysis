# Memory、标题、Compact 与 Resume

## 1. Claude Code 真正解决的问题，不是“上下文太长”，而是“长会话还能不能继续工作”

很多 Agent Demo 谈压缩时，默认目标只是：

- 把 token 降下来
- 让下一轮还能继续问答

Claude Code 2.1.88 处理的是更难的问题：

- 压缩之后还能继续执行任务吗
- 还能记得 plan / skill / agent 状态吗
- 还能恢复标题、tag、worktree、PR 绑定吗
- `/resume` 时还能回到正确语义吗

这套能力主要落在三组文件里：

- `memdir/memdir.ts`
- `services/compact/compact.ts`
- `utils/sessionStorage.ts`

它们共同组成了 Claude Code 的“会话续航层”。

## 2. Memory 是文件系统协议，不是隐式 embedding

### 2.1 `MEMORY.md` 只是入口索引，不是记忆本体

`memdir/memdir.ts:34-102` 明确了 `ENTRYPOINT_NAME = 'MEMORY.md'`，并给入口文件设置了严格限制：

- 最多 200 行
- 最多 25KB

如果超限，还会自动截断并追加 warning。

这背后的思想非常成熟：入口文件只应该扮演“索引”角色，而不是堆积大量正文。

### 2.2 记忆内容被做成强约束的类型系统

`memdir/memdir.ts:199-257` 构造 memory prompt 时，不是泛泛说“你可以记住一些事”，而是明确规定：

- memory 有固定类型
- 什么该写入 memory
- 什么不该写入 memory
- `MEMORY.md` 只放 pointer
- 每条记忆单独存文件

注释里还特别强调：可以从当前项目状态推导出的内容，不该存成 memory。

这意味着 Claude Code 的记忆不是“把一切都存起来”，而是把长期、跨会话、用户相关的信息做结构化沉淀。

### 2.3 它连“目录已存在”都在 prompt 里消除无谓动作

`memdir/memdir.ts:116-147` 和 `129-146` 的设计很细：

- Runtime 先确保 memory 目录存在
- prompt 再明确告诉模型不要 `mkdir`、不要检查是否存在，直接写

这是典型的 Claude Code 风格：

- 不让模型猜
- 不让模型为工具性琐事浪费回合
- 宿主先把执行条件准备好

## 3. Compact 不是摘要功能，而是“状态搬运器”

### 3.1 压缩前先做上下文净化

`services/compact/compact.ts:145-223` 会在压缩前做两件事：

1. strip images / documents
2. 去掉那些压缩后会重新注入的 attachment

这说明 Claude Code 很清楚：compact 输入本身也要控成本，而且不该把“之后会重建”的上下文再喂一遍。

### 3.2 压缩请求本身还要考虑 prompt cache

`services/compact/compact.ts:431-491` 和 `1136-1230` 展示了一个很高级的思路：

- 如果允许 prompt cache sharing
- 就用 `runForkedAgent(...)` 走 fork 路径
- 复用主线程已有的缓存前缀

也就是说，Claude Code 不只是关心“压缩能不能成功”，还关心“压缩这件事本身会不会把成本打爆”。

这已经不是普通摘要器，而是成本感知型运行时机制。

### 3.3 Compact 后会重建可运行状态，而不是只塞一段 summary

`services/compact/compact.ts:517-585` 是整套逻辑最值得学习的地方。

压缩成功后，它会重建：

- read file attachments
- async agent attachments
- plan attachment
- plan mode attachment
- skill attachment
- deferred tools delta
- agent listing delta
- MCP instructions delta

也就是说，压缩后留下来的不只是“对话摘要”，而是一个重新可执行的 Agent 状态入口。

这一点和教程版 `s06` 的差别非常大。

### 3.4 Compact 边界会被显式写进消息流

`services/compact/compact.ts:596-642` 会创建：

- compact boundary marker
- compact summary user message

并且记录压缩前已发现的工具集合。

这意味着 compact 不是偷偷改写 messages，而是会在 transcript 里留下清晰的状态边界。

### 3.5 连远程会话 keep-alive 都被纳入 compact 过程

`services/compact/compact.ts:1159-1176` 会在 compaction 期间发送 keep-alive，避免 remote WebSocket 因为空闲断开。

这是一个很能体现 Claude Code 工程水平的细节：

压缩不是局部算法问题，而会影响整个宿主会话活性。

## 4. Session Storage 的重点不是“存日志”，而是“低成本恢复元数据”

### 4.1 `reAppendSessionMetadata()` 解释了为什么 resume 能稳定工作

`utils/sessionStorage.ts:694-839` 很值得逐段看。

它会把这些元数据不断重新追加到 transcript 尾部：

- `last-prompt`
- `custom-title`
- `tag`
- `agent-name`
- `agent-color`
- `agent-setting`
- `mode`
- `worktree-state`
- `pr-link`

原因不是简单冗余，而是为了保证这些信息始终落在 tail scan 可见窗口里。

这是一种特别聪明的做法：

- 不做全文件重写
- 不做昂贵索引
- 只靠 append-only + tail read，就能支持 resume picker 与会话恢复

### 4.2 `custom-title` 和 `ai-title` 被故意分成两种 entry

`utils/sessionStorage.ts:2625-2687` 非常关键。

Claude Code 明确区分：

- `custom-title`
- `ai-title`

这样做的好处是：

1. 用户标题始终优先于 AI 标题
2. 恢复时不会把旧的 AI 标题重新 stamp 到尾部
3. 可以避免 resume 后“AI 旧标题覆盖用户新标题”的 bug

这背后不是 UI 小修小补，而是对 append-only 日志语义的精细设计。

### 4.3 resume 不是简单读文件，而是“恢复内存态”

`utils/sessionStorage.ts:2754-2816` 的 `restoreSessionMetadata(...)` 会把恢复到的元数据写回内存缓存，包括：

- title
- tag
- agent name / color
- mode
- worktree session
- PR metadata

这意味着 resume 不只是让历史可见，而是恢复 Runtime 当前会依赖的元数据状态。

### 4.4 轻量读取逻辑优先 tail，再回退 head

`utils/sessionStorage.ts:4767-4812` 展示了简洁但很强的读取策略：

- title 优先 tail
- 用户 title 优先于 AI title
- 再必要时从 head 中补读

这使得会话列表和恢复逻辑大多不需要全文件扫描。

对一个高频 append 的 JSONL transcript 系统来说，这是非常实用的工程解法。

## 5. Claude Code 如何区分“长期记忆”和“当前会话状态”

这套实现里有三种不同层次的持久化：

### 5.1 Memory

保留跨会话长期信息，例如：

- 用户偏好
- 项目长期背景
- 协作习惯

### 5.2 Session metadata

保留会话级状态，例如：

- 标题
- tag
- 当前模式
- worktree
- PR 关联

### 5.3 Compact summary + attachments

保留当前对话继续运行所需的短期工作态，例如：

- 当前计划
- 已调用技能
- 工具增量
- 子代理状态

这是一个特别重要的分层。很多 Agent 产品把三者混在一起，结果就是：

- 该长期保存的没结构
- 该短期恢复的丢失
- 该会话级展示的恢复不稳定

Claude Code 把它们拆得很清楚。

## 6. 和 `learn-claude-code` 的关系

`learn-claude-code/docs/zh/s06-context-compact.md` 讲清楚了最小版本：

- micro compact
- auto compact
- transcript 落盘

Claude Code 继承了这个范式，但把它扩展成“可运行恢复”版本：

1. 压缩输入先净化
2. 压缩过程考虑 prompt cache
3. 压缩后重建 plan / skill / tools / async agents
4. transcript 尾部反复追加关键元数据
5. resume 时把元数据重新写回内存态

所以可以这么理解：

- `learn-claude-code` 解决“长会话如何不爆 context”
- Claude Code 解决“长会话压缩后如何继续像原来那样工作”

## 7. 对 Agent 开发者的启发

### 启发 1：压缩目标应该是“恢复工作能力”，不是“生成摘要”

如果 compact 之后 plan、tools、agent state 都丢了，那只是降低了 token，不叫恢复能力。

### 启发 2：长期记忆、会话元数据、短期工作态一定要分层

它们的：

- 生命周期
- 可见范围
- 恢复方式
- 数据格式

都不一样。

### 启发 3：append-only transcript 很适合 Agent，但要配 tail-friendly metadata 设计

Claude Code 的 title / tag / worktree 设计表明，只要 tail 语义设计好，JSONL 也能支撑非常可靠的 resume。

### 启发 4：Memory 最好是用户可审计、文件化、强类型的

相比隐式向量缓存，这种方式更透明，也更适合真正进入工程生产。

## 8. 一句话总结

Claude Code 2.1.88 真正厉害的地方，不是“会做 compact”，而是它把 compact、memory、title、resume、worktree、plan 全部收束成了一条连续的会话续航链。

所以它压缩的不是聊天记录，而是在压缩后重建一个还能继续干活的 Agent。
