# 模型、工具、配置与运行时反射

## 1. Claude Code 不是静态 prompt 程序，而是反射式 Runtime

所谓“运行时反射”，在这里可以理解为：

**当前这一轮应该给模型什么 system prompt、什么工具、什么权限、什么上下文，不是写死的，而是由当前运行状态动态计算出来的。**

Claude Code 2.1.88 在这一点上非常典型。

## 2. 模型选择本身就是运行时决策

### 2.1 主线程模型和子代理模型是分开的

`QueryEngine.ts:130-149` 和 `runAgent.ts:340-345` 显示，主线程和子代理都可能有自己的模型选择逻辑：

- 主线程有 `userSpecifiedModel` / `fallbackModel`
- 子代理有 `agentDefinition.model`、父模型、显式 `model` 参数与 permission mode 共同参与决策

这说明 Claude Code 并不是“全局只有一个 model”。模型已经变成 Runtime 维度之一。

### 2.2 regular subagent 和 fork subagent 对 thinking 的处理不同

`runAgent.ts:666-695` 非常重要：

- fork child 在 `useExactTools` 路径下继承父级 thinking config
- regular subagent 默认把 thinking 关掉

原因很清晰：

- fork path 追求 cache 前缀一致
- 普通子代理更重视成本控制与输出可控性

这是一种非常实用的模型策略分层。

## 3. 工具集合是反射结果，不是常量

### 3.1 同一个会话，不同轮可能看到不同工具集

`REPL.tsx:2746-2755` 明确说明要从 `toolUseContext.options` 重新取 `freshTools` 和 `freshMcpClients`，而不是使用闭包中旧值。原因是 MCP 状态可能在两轮之间变化。

这意味着：

- 工具集是活的
- MCP 接入会改变 system prompt 与工具池
- 每轮 query 都必须重新反射当前能力面

### 3.2 skill-scoped allowed tools 也会改变可用工具面

`REPL.tsx:2701-2726` 的逻辑说明，技能前置说明甚至会直接改写 `alwaysAllowRules.command`。这进一步证明工具集是运行时反射结果。

## 4. system prompt 是工具、模式与配置的投影

### 4.1 `getSystemPrompt(...)` 接受的不只是模型名

`REPL.tsx:2772-2787` 调用 `getSystemPrompt(...)` 时传入的有：

- 本轮工具集
- mainLoopModel
- additionalWorkingDirectories
- mcpClients

也就是说，system prompt 并不是“描述模型是谁”，而是在描述：

- 当前有哪些能力
- 当前工作目录边界是什么
- 当前有哪些外部扩展系统

### 4.2 coordinator mode 会改变 user/system context

`coordinator/coordinatorMode.ts:80-145` 表明 coordinator 模式会额外注入：

- worker 可用工具说明
- scratchpad 目录说明
- worker 通信与 task-notification 协议说明

这非常像一个小型操作系统中的“进程调度器 prompt”。模型不是只听用户话，它还在听 Runtime 当前给它的角色定义。

## 5. 配置不是离线文件，而是控制运行时行为的图

### 5.1 `main.tsx` 的大量 CLI flag 反映了一个复杂配置平面

`main.tsx:976-988` 一大段 option 定义已经说明，Claude Code 的运行面是高度参数化的：

- output format
- json schema
- include partial messages
- permission mode
- allowed / disallowed tools
- custom system prompt
- mcp config
- continue / resume / fork-session

这不是“CLI 选项多”，而是说明 Claude Code 允许多个外部控制面直接影响 Agent Runtime。

### 5.2 server / connect / ssh 说明它已经具备 Host Runtime 抽象

`main.tsx:3960-4055` 展示了：

- `claude server`
- `claude connect`
- `claude ssh`

这些入口共享核心能力，但宿主与 transport 不同。也就是说，Claude Code 已经在做“把同一 Agent Runtime 部署到不同 Host 环境”的事情。

## 6. `learn-claude-code` 给了我们最小版本，Claude Code 给了生产版本

教程版的 s05 / s06 其实已经隐约指出：

- system prompt 可以分层
- skills 可以按需加载
- context 可以动态压缩

Claude Code 则把这三件事做成了统一反射流程：

1. 先看当前模式、当前工具、当前扩展
2. 再装配 prompt / context / permissions
3. 最后把这些投影成一次 query

这就是为什么 Claude Code 更像 Runtime，而不是 prompt app。

## 7. 反射式 Runtime 对 Agent 产品意味着什么

### 7.1 你的产品不要把“配置”看得太轻

很多团队把配置当成 YAML/JSON 附属品。Claude Code 的设计提醒我们：

- 配置决定工具面
- 配置决定权限面
- 配置决定模式切换
- 配置决定模型策略
- 配置决定宿主行为

换句话说，配置是 Runtime 行为图。

### 7.2 system prompt 应该更像“当前世界说明书”

而不是固定人格模板。

### 7.3 模型切换不应该只服务于成本，也应该服务于语义隔离

Claude Code 对 regular subagent 和 fork child 的模型/思考配置差异，就是一个很好的例子。

## 8. 对 Agent 开发者的启发

### 启发 1：把 prompt 生成从“字符串常量”升级成“反射器”

### 启发 2：把工具集与扩展集看成实时能力图

### 启发 3：把配置面设计成“会影响 Runtime 的一等对象”

### 启发 4：当你想做 server、SDK、remote 时，先把核心逻辑和宿主逻辑拆开

Claude Code 2.1.88 说明，一个成熟 Agent 系统最终会走向“核心运行时 + 多宿主外壳”的结构。
