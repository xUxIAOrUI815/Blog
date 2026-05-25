> https://blakecrosley.com/blog/ai-agent-approval-prompts-not-authorization

# 前言

核心观点：Agent 里的“Approve / Allow”弹窗不是授权机制，它只是一次人工点击事件；真正的授权必须是一个有范围、有证据、有时效、有审计、有撤销能力的权限对象。


讨论的是 AI Agent 在调用工具、执行命令时，不能把“用户点了一次同意“当成完整的安全授权。
Agent 需要更工程化的授权系统。

# approval 和 authorization

Approval 是一次决策事件，而 Authorization 是被授予的一段受约束的行动权力。
必须回答：这个权限是谁授予的？授予给哪个agent？允许调用哪个工具？能操作哪个资源？风险等级是什么？有效期多久？依据是什么？失败后如何回滚？之后如何审计统计？

Agent工程中，这种决策是非常重要的，因为传统应用中用户点击按钮意味着用户自己直接操作系统，但是Agent场景下，用户点击按钮意味着用户委托一个具有规划、解释、重试、调用工具能力的系统去执行动作。

由于代理执行者的存在，用户的授权边界必须界定清晰。

# approval prompt

普通的approval prompt 失败是因为**把一个高上下文的决策压缩成一个低上下文的点击**。

因为Agent在发起一个工具调用之前，可能已经做了很多事情，读文件、规划步骤、选择工具、填充参数等等，但用户看到的 approval prompt 往往只显示最后一步。

这会造成：
- Scope loss: 用户看到工具名但看不到它会影响哪个文件、账号、租户、环境或仓库数据。
- Evidence loss: 用户看到动作，但看不到支持这个动作的证据。
- Fatigue: 用户被频繁打断后机械点击同意。
- Persuasion: Agent 用自信、流畅、合理的语言包装一个风险动作，让用户更容易批准。

Agent的说服力本身就是风险，即使模型不是恶意的，Agent也可能过度包装方案、淡化不确定性，或者把高风险动作夹在一串低风险动作中。

不只有 tool call 失败才是问题，用户被误导批准一个不该批准的tool call，本身也是系统安全失败。

# 一个合格的授权对象

![](/blogs/ai-agent1/4cd6acb38de61639.png)

如果没有足够清晰的字段，approval就只是 vibes with a button，即“凭感觉点按钮”。

# commit point 

commit point 指的是 Agent 从“可逆探索”进入“有副作用”操作的那一刻。

**approval 必须发生在 commit point 之前。
**

运行阶段包括：
- 可逆探索阶段：让Agent在策略内自由执行并记录日志；
- 草稿阶段：让Agent准备artifact并展示preview；
- 风险分类阶段：先计算风险；
- commit point：根据策略暂停，并请求人类授权；
- 执行后记录结果、证据和rollback状态。

工程上更好的做法是对动作语义做分类。比如按照风险分类并基于语义来明确动作造成的影响和对该动作需要进行的操作（如中风险的write_file，需要diff preview，高风险的send_email，在发送前必须审批 recipient + content）。

# sticky approval 

always approve/always reject 的可以减少频繁打断，但也会放大整个run的权限风险。

既需要sticky approval，也需要有scope和expiry。
