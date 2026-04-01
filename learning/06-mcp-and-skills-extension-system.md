# MCP 与 Skills 扩展系统

## 1. Claude Code 的扩展系统，不是“插件市场”，而是能力编排层

如果只看界面层，Claude Code 的扩展能力好像就是：

- 有一些本地 skills
- 能连一些 MCP server
- 会多出一些命令和工具

但从 `restored-src/src/` 的实现看，它真正做的是把“知识、工具、资源、认证、执行上下文”统一建模成一套扩展系统。

这一层的关键文件主要有两组：

- `skills/loadSkillsDir.ts`
- `services/mcp/client.ts`

前者负责把 markdown 技能变成运行时可调用能力；后者负责把 MCP server 变成带认证、连接缓存、资源发现、工具发现、命令发现的远程能力平面。

## 2. Skill 在 Claude Code 里不是提示词片段，而是正式命令对象

### 2.1 frontmatter 已经说明它是“行为单元”

`skills/loadSkillsDir.ts:185-264` 解析的字段远超过教程版 `s05` 的 `name + description`：

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

这说明 Claude Code 的 skill 不是“把一段说明文字塞进上下文”，而是一个带运行时语义的命令定义。

最值得注意的几个字段是：

1. `allowed-tools`
2. `context: fork`
3. `agent`
4. `shell`
5. `model / effort`

它们共同决定了一件事：skill 不只是知识，还会改变执行方式。

### 2.2 `createSkillCommand(...)` 把 markdown 提升成可执行能力

`skills/loadSkillsDir.ts:270-400` 展示了一个很关键的转化过程。skill 最终会被做成一个 `Command`，拥有：

- 展示名
- 描述
- 参数提示
- 路径约束
- prompt 生成函数
- source / loadedFrom 元信息

其中 `getPromptForCommand(...)` 会做几件重要的事：

1. 参数替换
2. 注入 `Base directory for this skill`
3. 替换 `${CLAUDE_SKILL_DIR}`
4. 替换 `${CLAUDE_SESSION_ID}`
5. 必要时执行 markdown 里的内联 shell

这最后一条非常重要。它说明 Claude Code 允许 skill 不是纯静态文档，而是“文档 + 小型运行时脚本入口”。

### 2.3 但它又明确区分了本地 skill 与远程 skill 的信任边界

`skills/loadSkillsDir.ts:371-396` 有一段特别值得反复看：

- 本地 skill 可以执行 inline shell
- `loadedFrom === 'mcp'` 的 skill 不允许执行 inline shell

代码注释写得很直白：MCP skill 是 remote / untrusted，因此绝不能执行 `!\`...\`` 或 ```! ... ``` 这一类内联命令。

这个设计特别成熟，因为它没有把“扩展”一视同仁，而是按来源区分信任等级：

- 本地 skill：可信，可绑定脚本
- MCP skill：远程，不可信，只能当知识与命令元数据看待

这比很多“统一插件接口”设计更现实。

## 3. Skill 目录不是单一来源，而是多源加载

`skills/loadSkillsDir.ts:638-659` 和前面的 `getSkillsPath(...)` 一起，已经能看出它的 skill 来源并不单一：

- managed settings
- user settings
- project settings
- `--add-dir`
- 插件目录
- 兼容旧的 `/commands/`

再加上 `LoadedFrom` 枚举里的：

- `skills`
- `commands_DEPRECATED`
- `plugin`
- `managed`
- `bundled`
- `mcp`

Claude Code 实际上在做的是“多来源能力合流”。

这意味着它的扩展系统关心的不只是“有没有这个 skill”，还关心：

- 这个能力从哪里来
- 是否允许用户直接调用
- 是否被策略锁住
- 是否只能在特定路径下生效

这已经是运行时治理问题，而不是简单配置加载问题。

## 4. MCP 在 Claude Code 里不是插件清单，而是远程能力总线

### 4.1 `services/mcp/client.ts` 是一个完整连接编排器

`services/mcp/client.ts` 的体量已经说明，Claude Code 对 MCP 的理解远不止“列出工具然后 call 一下”。

它处理的事情包括：

- stdio / SSE / HTTP / WebSocket / claude.ai proxy 多种 transport
- OAuth / bearer token / session ingress token
- 请求超时与长连接超时分离
- 连接缓存
- 断线重连
- 会话过期检测
- 资源、工具、命令、skills 的发现
- cleanup 与进程回收

仅从 transport 初始化部分就能看出来：

- `services/mcp/client.ts:621-865`

这里对 SSE、HTTP、WebSocket 的 header、authProvider、timeout 和 proxy 都做了独立处理。它的目标不是“能连上”，而是“不同传输都能被产品化治理”。

### 4.2 MCP 的认证不是附属功能，而是连接状态的一部分

`services/mcp/client.ts:2301-2318` 会对近期返回 401 的 server 做 `needs-auth` 缓存，避免无意义重连探测。

如果 server 需要认证，Claude Code 不会硬报错停住，而是把它降级成：

- `client.type === 'needs-auth'`
- 提供 `createMcpAuthTool(...)`

这很像把“认证未完成”也建模成运行时状态，而不是异常分支。

对产品来说，这一点特别关键，因为现实世界的远程能力并不总是“在线且已授权”的。

### 4.3 MCP 连接成功后，不只拉工具，还拉命令、资源和技能

`services/mcp/client.ts:2169-2198` 与 `2342-2369` 是整个 MCP 子系统最关键的地方。

如果 server 支持 resources，Claude Code 会并行获取：

- tools
- commands
- mcp skills
- resources

然后再把它们装回统一输出：

- `tools`
- `commands`
- `resources`

其中还有两个关键细节：

1. 如果 server 支持资源，会补充 `ListMcpResourcesTool` 和 `ReadMcpResourceTool`
2. 如果开启 `MCP_SKILLS`，还会从资源里提取 mcp skills

这说明 MCP 在 Claude Code 里并不是“远程工具列表”，而是一个远程能力平台：

- tool 平面
- resource 平面
- command 平面
- skill 平面

### 4.4 断线与会话过期被明确当成一等问题

`services/mcp/client.ts:1216-1400` 以及 `1675-1703` 很能说明 Claude Code 的工程取向。

它不假设 remote server 永远稳定，而是显式处理：

- terminal connection errors
- SSE reconnect exhausted
- HTTP session expired
- close 后清掉 fetch/cache/connect 缓存

这背后的思想很重要：

MCP 集成不是“连接一次，永久有效”，而是一个需要被恢复、重建、清理的长期运行状态机。

## 5. 和 `learn-claude-code` 的关系

`learn-claude-code/docs/zh/s05-skill-loading.md` 给出的是最小范式：

- skill 目录扫描
- 系统提示只放 skill 描述
- 真正需要时，再通过 `load_skill` 注入完整内容

这个最小范式非常对，但 Claude Code 在生产里把它继续推进了五步：

1. 把 skill 升级成正式 `Command`
2. 把 frontmatter 升级成运行时元数据
3. 把 `allowed-tools` 与权限联动
4. 允许本地 skill 绑定 shell 执行
5. 把远程 MCP 扩展纳入同一能力图

也就是说：

- `learn-claude-code` 讲清了“按需知识注入”
- Claude Code 解决了“按需知识如何进入真实产品运行时”

## 6. 为什么说 MCP 不是“插件系统”，而是宿主能力桥

如果只把 MCP 当插件系统，会天然忽略三件事：

1. 认证
2. 资源
3. 连接生命周期

Claude Code 的实现恰恰表明，MCP 真正的价值是：

- 让宿主能力外置
- 让工具与资源可远程发现
- 让认证与连接状态进入 Runtime

这意味着在 Agent 产品里，MCP 更像：

- 外部能力总线
- 远程资源目录
- 标准化权限与认证入口

而不是一个“第三方插件列表”。

## 7. 对 Agent 开发者的启发

### 启发 1：扩展系统最好统一建模为“能力图”

不要只管理 tool，还要考虑：

- prompt / skill
- resource
- command
- auth state
- source provenance

### 启发 2：必须按来源区分信任等级

本地 skill 可以执行脚本，不代表远程 skill 也该有同样权限。Claude Code 对 MCP skill 的不信任默认值，非常值得照抄。

### 启发 3：frontmatter 不该只服务展示

最有价值的 frontmatter 字段，通常不是 `name`，而是：

- `allowed-tools`
- `context`
- `agent`
- `model`
- `effort`

这些字段决定 skill 如何真正改变运行时。

### 启发 4：MCP 集成要先设计连接生命周期，再设计工具调用

如果没有：

- auth cache
- reconnect
- cleanup
- stale-cache invalidation

那么你的 MCP 集成很快就会在真实场景里变成一堆偶发错误。

## 8. 一句话总结

Claude Code 2.1.88 的扩展系统，本质上不是“给模型多塞几个外挂”，而是把本地技能、远程工具、资源发现、认证状态和命令语义统一收编到同一个 Agent Runtime 里。

这也是它比教程版更像“操作系统”的原因。
