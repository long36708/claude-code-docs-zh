---
title: Claude Opus 5 提示指南
url: https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5
description: Claude Opus 5 的行为差异与提示模式，涵盖响应冗长度、智能体叙述、任务范围界定、子智能体委派、自我纠正，以及禁用思考时的输出伪影。
---

本指南涵盖 Claude Opus 5 特有的提示模式。有关该模型的能力和 API 变更，请参阅 [Claude Opus 5 新特性](https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5)。有关适用于所有当前 Claude 模型的技术，请参阅[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)。

Claude Opus 5 专为复杂的智能体编码和企业工作而构建，在长周期智能体任务方面尤为擅长。它在现有的 Claude Opus 4.8 提示上开箱即用即可表现良好。以下模式涵盖了最常需要调优的行为。

<Note>
  有关从 Claude Opus 4.8 迁移时的 API 变更（思考默认开启，且禁用思考的上限为 `high` effort），请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-4-8-to-claude-opus-5)。
</Note>

## 能力提升

与 Claude Opus 4.8 相比，与提示最相关的改进包括：

* **智能体编码：** Claude Opus 5 在困难的编码任务上表现最强：多文件功能、较大规模的重构以及端到端的功能开发。它会完成整个任务，而不是留下存根或占位符；当预先给出完整的任务规范并让其自行运行时，它的表现最佳。它在单轮编辑等较简单的任务上也表现良好，只是与先前模型的差异较小。
* **代码审查与缺陷发现：** Claude Opus 5 以高精确率和高召回率审查代码：它每次审查都能以很高的比率发现真实缺陷，而且其额外发现大多是真实问题而非误报。在较低的 effort 设置下准确性依然保持，这支持在审查时先进行快速审查，之后再进行更彻底的审查。如果您的审查提示中写有"只报告高严重性问题"或"保守一些"，模型可能会严格按字面遵循该指令而减少报告；请改为要求它报告所有问题，并在单独的一轮中进行筛选。
* **较低 effort 下的效率：** `low` 和 `medium` [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 能以远低于更高设置的令牌数和延迟产出高质量结果。从默认值（`high`）开始，并根据您的评估进行调整：在质量能够保持的地方，大胆使用 `low` 和 `medium` 作为控制令牌成本和响应时间的主要手段；对于要求苛刻的编码和智能体工作，则提升到 `xhigh`。如果您沿用了先前模型的 effort 默认值，请在您自己的评估上重新进行一次 effort 扫描。完整建议请参阅 [Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#recommended-effort-levels-for-claude-opus-5)。
* **视觉：** Claude Opus 5 在图表、文档和示意图理解，以及 UI 和前端视觉复刻方面表现出色。请重新验证您为先前模型调优的任何提示侧视觉变通方案；它们可能已不再需要。当模型拥有可迭代分析、裁剪并以视觉方式验证其工作的工具时，视觉性能最强，而且"tool use"（工具使用）是比单纯思考更具成本效益的手段。
* **长上下文工作：** Claude Opus 5 拥有 [100 万令牌的 "context window"（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)，这既是默认值也是最大值，并且其指令遵循、工具调用和推理能力在整个窗口范围内保持一致。
* **办公与文档任务：** Claude Opus 5 能够生成并处理包含非平凡公式的复杂多工作表电子表格，并能制作结构良好的幻灯片。请在提示中告知它需要遵循的任何特定样式或模板。
* **多智能体协调：** Claude Opus 5 能很好地协调子智能体团队，具备有效的"编写者-验证者"模式，且很少出现智能体相互覆盖工作成果的情况。对于成本敏感的工作负载，请限制委派；参见[控制子智能体生成](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#controlling-subagent-spawning)。

## 响应长度与冗长度

Claude Opus 5 默认的面向用户响应比先前的 Opus 模型更长。[effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)控制的是模型[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost)的多少，而不是它说多少：降低 effort 可以减少思考量，但并不能可靠地缩短可见响应。要控制响应长度，请明确地在提示中提出要求。

一条简短的简洁性指令就很有效。例如，对于面向用户的多轮产品：

```text wrap
Keep responses focused, brief, and concise. Keep disclaimers and caveats short, and spend most of the response on the main answer. When asked to explain something, give a high-level summary unless an in-depth explanation is specifically requested.
```

在较长的"system prompt"（系统提示）中，请在提示末尾附近搭配一条简短的提醒：

```text wrap
<tone_preference>
Keep outputs reasonably concise.
</tone_preference>
```

## 面向用户的进度更新

Claude Opus 5 在智能体工作期间很乐于叙述：它倾向于宣布自己即将做什么，而且它在智能体会话中每条消息的输出通常比先前模型更长。就任务期间如何与用户沟通给出明确指导会对它有所帮助。要减少叙述，请描述您想要的节奏和形式：

```text wrap
Before your first tool call, say in one sentence what you're about to do. While working, give a brief update only when you find something important or change direction. When you finish, lead with the outcome: your first sentence should answer "what happened" or "what did you find," with supporting detail after it for readers who want it.
```

要增加叙述或改变其风格，同样的手段也可反向使用：明确描述更新应该是什么样子并提供示例。您所期望的沟通风格的正面示例，往往比关于不该做什么的指令更有效。

## 书面交付物长度

与对话冗长度不同，Claude Opus 5 写入磁盘的文件（报告、Markdown 文档、摘要）通常比先前模型更长。如果您的产品包含由 Claude 撰写的文档，请添加明确的长度校准：

```text wrap
Match the length of written documents to what the task needs: cover the substance, but do not pad with filler sections, redundant summaries, or boilerplate.
```

## 任务范围与过度验证

Claude Opus 5 无需被告知就会验证自己的工作。如果您的提示包含明确的验证指令（"对任何非平凡任务都包含最终验证步骤"、"使用子智能体进行验证"），请将其删除：此类指令会导致 Claude Opus 5 过度验证，删除它们可以减少浪费的令牌而不损失质量。这同样适用于添加了单独验证步骤的旧版框架脚手架。

Claude Opus 5 还可能扩大任务范围，添加未被要求的步骤，或对任务应该是什么套用自己的判断。对于范围狭窄的任务，请明确约束范围：

```text wrap
Deliver what was asked, at the scope intended. Make routine judgment calls yourself, and check in only when different readings of the request would lead to materially different work. If the request seems mistaken or a better approach exists, say so in a sentence and continue with the task as asked rather than quietly narrowing, widening, or transforming it. Finish the whole task, and stop short of actions that are clearly beyond what was asked.
```

## 控制子智能体生成

Claude Opus 5 比先前模型更乐于委派给子智能体。委派在真正独立且规模可观的工作线上会带来回报，但应用于小任务时会成倍增加成本和时间。如果您的框架支持子智能体，请就哪些场景值得委派给出明确指导，或对可启动的智能体数量设置确定性上限。例如：

```text wrap
Delegate to a subagent only for large tasks that are genuinely independent and parallelizable, such as a wide multi-file investigation. Do not delegate work you can finish yourself in a handful of tool calls, and do not use subagents to verify or double-check your own work. If one subagent can complete the task, use one rather than several, and keep spawn counts low.
```

如果您的框架是 Claude Code 或 Claude Agent SDK，确定性上限即 `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` 和 `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` 环境变量以及 SDK 的 `max_budget_usd` 选项。它们需要 Claude Code 2.1.217 或更高版本，因此在将固定版本的 SDK 指向 Claude Opus 5 之前请先更新。仅当您使用其 `claude_code` 系统提示预设时，Claude Code 才会在 Claude Opus 5 上自行添加委派指令；若使用自定义系统提示或省略系统提示，请自行添加委派指令，例如本节中的示例。请参阅 Agent SDK 文档中的[限制子智能体深度、并发数和支出](https://code.claude.com/docs/zh-CN/agent-sdk/subagents#cap-subagent-depth-concurrency-and-spend)。

## 自我纠正

Claude Opus 5 无需提示就能很好地发现并修复自己的错误。避免指示它进行本已执行的复查（"仔细检查你的答案"、"回复前重新验证"）；与验证指令一样，这些指令会与模型自身的行为叠加，增加成本却不改善结果。

该模型对其先前陈述的纠正叙述也比先前模型更多，这在面向用户的产品中可能并不理想。要将纠正叙述限制在重要的纠正上：

```text wrap
Only correct an earlier statement when the error would change the user's code, conclusions, or decisions. State corrections plainly and briefly, then continue the task. For slips that change nothing for the user, make the fix and move on without noting it.
```

## 在禁用思考的情况下运行

Claude Opus 5 默认开启[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)运行，且只有在 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 为 `high` 或更低时才能禁用思考；请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-4-8-to-claude-opus-5)。禁用思考后，模型的可见输出中偶尔会出现两种伪影。两者的主要缓解措施都是保持思考开启，并通过较低的 effort 级别而非禁用思考来控制令牌成本：对于大多数任务，在 `low` effort 下开启思考的表现优于在相近成本下禁用思考。

**以文本形式出现的工具调用。** 禁用思考后，模型偶尔会将工具调用写入其面向用户的文本中，而不是发出结构化的 `tool_use` 块。该轮会正常完成，但调用永远不会运行；在智能体循环中，泄漏的文本会留在对话历史中，因此后续轮次也会受到影响。这在搜索等工具密集型工作负载中最为常见。

**输出中的内部 XML 标签。** 禁用思考后，模型可能会在其可见响应中发出 `<thinking>` 标签或其他内部 XML 标签。如果您的系统提示包含指示模型不要思考或不要推理的规则，请将其删除；此类指令会增加标签泄漏。

对于必须保持禁用思考的集成，一条合并的指令即可缓解这两种伪影：它明确允许模型在工具调用前发言，在没有合适工具时提供强制调用之外的替代方案，并给出一条禁止内部标签的通用规则：

```text wrap
When you use a tool, you may say a brief sentence first. If no tool can express what the user asked for, say so instead of guessing. Do not include internal or system XML tags in your response.
```

点名指出思考标签的指令不如通用形式有效，因此请避免具体点名它们。
