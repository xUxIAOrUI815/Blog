# 七个核心组件

User 
Interfaces
Permission System
Tools
State & Persistence
Execution Environment
所有入口最终都收敛到同一个agent loop，Claude Code 的CLI、SDK、headless模式并不是各有一套执行引擎，而是共享核心循环，只是渲染和交互方式不同。

# Where does reasoning live? 推理放在哪里？

模型负责决定下一步要做什么；harness负责验证、授权、执行和记录。

模型不直接访问文件系统、shell、联网，而是输出结构化的 `tool_use` 请求，然后由外部系统解析、检查权限、调用工具、收集结果。

LLM 不是操作系统权限主体，只是提出操作意图，真正执行动作的是 deterministic harness。

和LangGraph式的显式状态图设计不同，LangGraph通常把流程控制显式写成节点和边，而Claude Code 是简单的 ReAct循环 + 工具/权限/上下文/恢复系统。

Claude Code的大部分代码是 operational infrastructure。生产级Agent的工程量主要不在“智能逻辑”，而是在“让智能可靠运行的系统逻辑”。

# Agent Loop 

Claude Code 的核心循环是，reactive loop，即ReAct风格。倾向于边做边看环境反馈。

循环本身不复杂，而是每一步背后的系统。

```
上下文怎么组装？
工具怎么暴露？
权限怎么判断？
多个工具能不能并发执行？
命令失败怎么办？
上下文超了怎么办？
结果怎么保存？
子 Agent 的内容要不要塞回主上下文？
```

# 权限系统 

Claude Code的安全设计是 deny-first 

Claude Code 的权限模式包括：

- plan
- default
- acceptEdits
- auto
- dontAsk
- bypassPermission
- bubble

权限系统是多层防御。
用户对权限弹窗的通过率非常高，容易出现 approval fatigue。用户习惯性批准之后，弹窗不再是可靠安全机制。Claude Code 用 deny rules、sandbox、classifier和hooks等机制把安全边界前移。

举个例子，对于能够执行工具的Agent，不应该只设计用户确认点击，而是：

```
工具白名单/黑名单
危险动作分级
只读操作和写操作区分
命令sandbox
权限不可跨 session 默认继承
工具调用前后 hook
拒绝后的可恢复反馈
```

# 上下文管理

Claude Code 把 context window 当成核心资源约束。不能够简单地做截取，而是要 compaction pipeline。

Claude Code 的方案是一个五层的 compaction pipeline：
- Budget reduction 
- Snip
- Microcompact
- Context collapse
- Auto-compact

没有一种压缩策略可以处理所有上下文问题，Claude Code 使用渐进式压缩。这种压缩方法比传统的按时间先后删除（即删除最早消息）和简单summary（超过多少长度就 summarization一次）不同。

Claude Code 使用更细的策略。
先做低成本、低损失的压缩，如果不够，再做更强更有损的压缩。

Claude Code 的其他减少上下文压力的机制还有：
1. Claude Code 懒加载
2. 工具 schema 延迟加载
3. 子 Agent 只返回 summary
4. 单个 tool_result 限制大小

# 扩展机制

Claude Code 同时有 MCP、plugins、skills、hooks 四类扩展机制。这四类机制的区别主要在插入 Agent Loop 的位置不同、上下文成本不同。

机制 | 作用 | 插入位置 | 上下文成本
--- | --- | --- | ---
MCP | 接入外部工具和服务 | model 可调用的工具池 | 高
Plugin | 打包分发多种扩展能力 | 多个位置 | 中
Skill | 注入特定领域能力/指令 | context assembly | 低
Hook | 拦截生命周期事件 | tool 前后/session 事件 | 默认近似零

基于不同扩展需求消耗的“成本”（如 context budget）不同，对不同的任务或者典型的调用场景来使用不同的扩展机制。

比如：
如果所有扩展都做成工具，模型就要看到大量工具schema，上下文会膨胀。
如果所有扩展都做成提示词，又无法控制工具执行生命周期。
如果所有扩展都做成提示词，又无法表达模型可主动调用的新能力。

# 子 Agent 

Claude Code 的多Agent 架构不是 AutoGen 框架的多角色对话，而是 parent-agent对worker-agent的任务委派。

主 Agent 可以通过 AgentTool 启动子 Agent。
子 Agent 的特点是：
```
有独立上下文窗口
可以有独立工具集
可以有独立权限模式
可以有独立transcript
可以运行在 worktree 隔离环境中
最后只把 summary 返回给父 Agent
```

避免上下文爆炸。

# 会话持久化

session persistence 依赖 append-only JSONL transcript。
不把整个 session 状态放进复杂数据库，而是把用户消息、模型消息、工具调用、工具结果、compact boundary 等事件不断追加到 transcript 文件中。
安全选择：resume/fork 不恢复 session-scope permissions。也就是说，即使恢复一个旧会话，之前临时批准过的权限不会自动继承。信任决策应该有作用域和生命周期。

# 同样的问题，不同的部署上下文导致不同架构

Agent 架构没有最优解，部署上下文决定架构答案。

Claude Code 是面向开发者本地代码仓库的 CLI/IDE coding harness。
OpenClaw 是多通道个人助理gateway，连接 WhatsApp/Telegram/Slack/Discord 等入口。

coding agent 关注“动作安全”和“上下文压缩”；personal assistant gateway 关注“入口身份”“多通道路由”“长期记忆”。

不同类型的 Agent，安全边界、记忆机制、工具注册方式和运行时架构都不同。

