# Sandbox 与工作区资源模型

## 1. Claude Code 把“工作目录”当成显式资源，而不是隐式前提

很多 Agent Demo 都默认：

- 当前目录就是工作区
- 只要 shell 能跑，文件就能读写

Claude Code 2.1.88 完全不是这个思路。它把以下对象都显式资源化了：

- 原始 cwd
- process PWD
- additional working directories
- worktree path
- remote workspace
- sandbox 允许读写路径

这使得它可以在复杂场景里仍然保持边界可控。

## 2. sandbox schema 已经说明它是独立子系统

`entrypoints/sandboxTypes.ts:1-140` 很直接地把 sandbox 类型定义成 SDK 与 settings 共享的单一事实来源。

其中包括：

- network 配置
- filesystem 配置
- `enabled`
- `failIfUnavailable`
- `autoAllowBashIfSandboxed`
- `allowUnsandboxedCommands`
- `ignoreViolations`
- `excludedCommands`

这说明 sandbox 在 Claude Code 里不是某个 shell 工具的小开关，而是顶层运行时配置。

## 3. filesystem sandbox 的设计重点：允许与拒绝都是一等公民

`SandboxFilesystemConfigSchema` 里同时有：

- `allowWrite`
- `denyWrite`
- `denyRead`
- `allowRead`
- `allowManagedReadPathsOnly`

它表达的是一种资源边界思路：

1. 先确定大边界
2. 再允许局部例外
3. 再允许 policy settings 覆盖用户级配置

这比“只做 allow list”更适合真实工程环境，因为项目目录往往包含：

- 可编辑区
- 只读依赖区
- 敏感配置区
- 工作副本区

## 4. 工作目录集合是动态扩展的

### 4.1 `additionalWorkingDirectories` 是资源边界的核心拼图

`ToolPermissionContext` 在 `Tool.ts:123-138` 中就内建了 `additionalWorkingDirectories`。

`permissionSetup.ts:913-1010` 展示了它的来源：

- `process.env.PWD` 的 symlink 情况
- settings 里的附加目录
- `--add-dir`
- workspace 校验结果

这说明 Claude Code 不是把“工作区”建模成单一路径，而是一个经过验证的目录集合。

### 4.2 这样设计的直接收益

- 可兼容 symlink 场景
- 可支持多 repo / 多目录工程
- 可支持 worktree / generated output / scratchpad
- 权限系统能基于目录集合，而不是基于当前 cwd 做模糊判断

## 5. worktree 是资源隔离，不只是 git feature

### 5.1 AgentTool 将 worktree 作为 isolation mode

`AgentTool.tsx:98-101` 把 `isolation` 作为输入 schema 的正式字段，支持 `worktree`。

`AgentTool.tsx:582-593` 会在需要时创建 agent worktree；`640-685` 里还定义了完成后的 cleanup 逻辑。

这说明在 Claude Code 里，worktree 不是用户手工技巧，而是 Runtime 官方隔离机制。

### 5.2 worktree 和 cwd override 会反向影响 agent prompt

`AgentTool.tsx:595-641` 里，fork + worktree 会注入 `buildWorktreeNotice(...)`，而普通子代理如果有 cwd override，则会跳过预构建 system prompt，让 `runAgent` 在新的 cwd 中重新构建。

这里很有意思：

- 资源边界变化
- 会影响系统提示
- 进而影响模型对路径世界的理解

这说明资源模型和 prompt 模型是耦合的。

## 6. remote / server 模式进一步把 workspace 提升为宿主资源

`main.tsx:3962-4007` 的 `claude server` 支持 `--workspace <dir>`。这意味着 server 模式下，workspace 已经是 session host 的默认资源边界。

`main.tsx:3193-3205` 的 ssh 场景也说明，远程宿主会决定实际运行位置。

所以 Claude Code 的资源模型并不只针对本地 CLI，而是针对“多宿主 Agent Runtime”。

## 7. 为什么这套资源建模对 Agent 很重要

### 7.1 因为 Agent 的真实安全边界通常不在 API 层，而在文件系统层

对 coding agent 来说，最重要的资源一般不是“函数能不能调”，而是：

- 哪些目录能读
- 哪些目录能写
- 哪些改动必须隔离
- 哪些会话可以跨 repo 恢复

Claude Code 把这些边界都做成显式对象，而不是散落在 handler 里。

### 7.2 因为资源边界也影响模型心智

当 cwd、worktree、remote workspace 变化时，模型对“当前项目”的理解也必须变化。Claude Code 在这点上做得很细，说明它把模型当作真正会受环境影响的执行者，而不是静态文本生成器。

## 8. 和 `learn-claude-code/s12` 的关系

`learn-claude-code/s12-worktree-task-isolation.md` 给出的最小范式是：

- 任务板管目标
- worktree 管目录
- 二者绑定

Claude Code 2.1.88 则进一步把这个思想推进到：

- worktree 是 AgentTool 的正式 isolation mode
- worktree 元数据可用于 resume
- worktree cleanup 与任务通知耦合
- sandbox / permission / cwd override 与 worktree 相互影响

它已经从“任务隔离技巧”进化成“运行时资源模型”。

## 9. 对 Agent 开发者的启发

### 启发 1：把 workspace 当成显式资源图

不要让所有 handler 都自己猜 cwd。

### 启发 2：隔离模式应当进入工具协议和任务协议

而不是只留给高级用户手动操作 git。

### 启发 3：prompt 和资源边界要联动

路径世界一旦变了，系统提示通常也该重建。

### 启发 4：先做最小目录边界，再做完整 sandbox

如果一开始做不出完整 sandbox，至少也要先做：

- allow/deny 目录集
- explicit cwd / worktree 机制
- per-agent resource boundary

这会比单纯做“命令审批弹窗”更接近真正的工程边界。
