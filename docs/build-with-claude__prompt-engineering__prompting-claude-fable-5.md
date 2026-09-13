---
title: 为 Claude Fable 5 编写提示
url: https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5
description: Claude Fable 5 和 Claude Mythos 5 的行为差异和提示模式，涵盖努力程度、指令遵循、长时间运行、记忆和脚手架变更。
---

本指南涵盖了 Claude Fable 5 和 Claude Mythos 5 特有的提示和脚手架模式。有关该模型的能力、API 变更、定价和可用性，请参阅[介绍 Claude Fable 5 和 Claude Mythos 5](https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5)。有关适用于所有当前 Claude 模型的技术，请参阅[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)。

Claude Fable 5 能够处理以前对于先前模型来说过于复杂、耗时或模糊的问题，尤其擅长需要人类花费数小时、数天或数周才能完成的端到端工作。取得最佳成果的团队将 Claude Fable 5 应用于他们最棘手的未解决问题；仅在较简单的工作负载上测试它往往会低估其能力范围。它在更直接的任务上也表现可靠。

Claude Fable 5 与 Claude Opus 4.8 有几个行为差异，可能需要更新提示或脚手架。这一级别的能力提升也是重新评估哪些指令、工具和护栏仍然需要的好时机。下面的模式涵盖了最常需要调整的行为。

<Note>
  有关 Claude Fable 5 和 Claude Mythos 5 特有的 API 参数变更（仅自适应思考、仅摘要思考输出、无扩展思考预算、`refusal` 停止原因和回退处理），请参阅[介绍 Claude Fable 5 和 Claude Mythos 5](https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5)。

  Claude Fable 5 运行安全分类器，针对攻击性网络安全技术（例如构建漏洞利用、恶意软件或攻击工具）、生物学和生命科学内容（例如实验室方法或分子机制），以及提取模型的摘要思考。良性的网络安全工作和有益的生命科学任务也可能触发这些保护措施。要自动重新路由被拒绝的请求，请配置[服务器端或客户端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)到 Claude Opus 4.8。
</Note>

## 能力提升

与 Claude Opus 4.8 相比，Claude Fable 5 在以下方面有所改进：

* **长时程自主性。** Claude Fable 5 能够在较长时间内持续产出有效成果，完成多天、目标导向的运行，并在长时间、复杂的任务中保持强大的指令保留能力。
* **在复杂、规范明确的问题上的首次正确性。** 早期测试者报告称，对于以前需要数天迭代的系统，现在可以单次实现。
* **视觉。** Claude Fable 5 以显著更高的准确性解读密集的技术图像、Web 应用程序和详细的屏幕截图，通常同时使用更少的输出令牌，并经过训练使用 bash 和裁剪工具来处理翻转、模糊或有噪声的图像。
* **企业工作流程。** Claude Fable 5 遵循指令、保持在范围内，并在财务分析、电子表格、幻灯片和文档上产出专业级输出。
* **代码审查和调试。** 查找错误的召回率（在安全分类器覆盖的网络安全领域之外）明显高于 Claude Opus 4.8，包括跨代码库和仓库历史的搜索。
* **处理模糊性。** 当给定复杂、多线程的请求并要求确定下一步时，Claude Fable 5 表现良好。
* **委派和协作。** Claude Fable 5 在调度和维持并行子代理方面明显更可靠，并可靠地管理与长时间运行的子代理和对等代理的持续通信。

除了这些具体改进之外，Claude Fable 5 在几乎所有任务上通常都比先前模型更有能力。Claude Fable 5 不适用于攻击性网络安全或生物学和生命科学工作；这些领域的请求可能会返回 [`stop_reason: "refusal"`](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。

## 默认更长的回合

在较高的[努力程度](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)设置下，针对困难任务的单个请求可能会运行许多分钟，尤其是当任务需要收集上下文、构建和自我验证时，而自主运行可能会延续数小时。这是团队在适应 Claude Fable 5 时遇到的最大转变之一。在迁移之前调整客户端超时、流式传输和面向用户的进度指示器，并考虑重构框架以异步检查运行情况，例如通过计划任务，而不是阻塞。为防止 Claude Fable 5 在任务模糊时过度规划：

```text wrap
When you have enough information to act, act. Do not re-derive facts already established in the conversation, re-litigate a decision the user has already made, or narrate options you will not pursue in user-facing messages. If you are weighing a choice, give a recommendation, not an exhaustive survey. This does not apply to thinking blocks.
```

## 考虑所有努力程度级别

[努力程度](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)是 Claude Fable 5 上智能、延迟和成本之间权衡的主要控制手段。对大多数任务使用 `high` 作为默认值，对最需要能力的工作负载使用 `xhigh`，对常规工作使用 `medium` 或 `low`。Claude Fable 5 上较低的努力程度设置仍然表现良好，并且通常超过先前模型上的 `xhigh` 性能。如果任务完成但耗时超过必要，或者如果您想要更快、更具交互性的工作方式，请降低努力程度。

在较高努力程度下处理常规工作时，Claude Fable 5 可能会收集上下文并进行超出任务需要的深思熟虑。同时，较高的努力程度通常会产生出色的验证行为、复杂的推理和最严谨的输出。为防止在较高努力程度下进行未经请求的整理或重构：

```text wrap
Don't add features, refactor, or introduce abstractions beyond what the task requires. A bug fix doesn't need surrounding cleanup and a one-shot operation usually doesn't need a helper. Don't design for hypothetical future requirements: do the simplest thing that works well. Avoid premature abstraction and half-finished implementations. Don't add error handling, fallbacks, or validation for scenarios that cannot happen. Trust internal code and framework guarantees. Only validate at system boundaries (user input, external APIs). Don't use feature flags or backwards-compatibility shims when you can just change the code.
```

## 强大的指令遵循

指令遵循能力得到了足够的改进，您可以用简短的指令来引导大多数行为，而不必逐一列举每个行为。例如，在未引导的情况下，Claude Fable 5 可能会进行超出任务需要的详细阐述，尤其是在较高的努力程度设置下：调查它不会采用的选项、详细解释根本原因、产生高度结构化的 PR 描述，或编写叙述下一行代码作用的注释。一条简短的简洁性指令与列出每个模式同样有效：

```text wrap
Lead with the outcome. Your first sentence after finishing should answer "what happened" or "what did you find": the thing the user would ask for if they said "just give me the TLDR." Supporting detail and reasoning come after. Being readable and being concise are different things, and readability matters more.

The way to keep output short is to be selective about what you include (drop details that don't change what the reader would do next), not to compress the writing into fragments, abbreviations, arrow chains like A → B → fails, or jargon.
```

这同样适用于长时间运行工作流程中的检查点行为。要让 Claude Fable 5 仅在真正需要您的地方停止，无需列举每种情况：

```text wrap
Pause for the user only when the work genuinely requires them: a destructive or irreversible action, a real scope change, or input that only they can provide. If you hit one of these, ask and end the turn, rather than ending on a promise.
```

## 在长时间运行期间为进度声明提供依据

在长时间自主运行中，指示 Claude Fable 5 根据实际工具结果审核进度。在 Anthropic 的测试中，即使在旨在引发此类情况的任务上，这也几乎消除了虚构的状态报告：

```text wrap
Before reporting progress, audit each claim against a tool result from this session. Only report work you can point to evidence for; if something is not yet verified, say so explicitly. Report outcomes faithfully: if tests fail, say so with the output; if a step was skipped, say that; when something is done and verified, state it plainly without hedging.
```

## 说明边界

Claude Fable 5 偶尔会采取未经请求的操作（在没有要求的情况下起草电子邮件、创建防御性的 git 分支备份）。明确定义 Claude Fable 5 应该和不应该做什么的约束：

```text wrap
When the user is describing a problem, asking a question, or thinking out loud rather than requesting a change, the deliverable is your assessment. Report your findings and stop. Don't apply a fix until they ask for one. Before running a command that changes system state (restarts, deletes, config edits), check that the evidence actually supports that specific action. A signal that pattern-matches to a known failure may have a different cause.
```

## 并行子代理

Claude Fable 5 比先前模型更容易调度并行子代理。频繁使用子代理，提供关于何时适合委派的明确指导，并优先选择协调器和子代理之间的异步通信，而不是阻塞直到每个子代理返回。在子任务之间保持上下文的长期子代理通过缓存读取节省时间和成本，并避免在最慢的子代理上形成瓶颈。

```text wrap
Delegate independent subtasks to subagents and keep working while they run. Intervene if a subagent goes off track or is missing relevant context.
```

## 构建记忆系统

当 Claude Fable 5 能够记录先前运行的经验教训并引用它们时，它的表现尤其出色。提供一个记录笔记的地方，简单到一个 Markdown 文件即可：

```text wrap
Store one lesson per file with a one-line summary at the top. Record corrections and confirmed approaches alike, including why they mattered. Don't save what the repo or chat history already records; update an existing note rather than creating a duplicate; delete notes that turn out to be wrong.
```

要从现有历史记录引导记忆系统，让 Claude Fable 5 审查过去的会话：

```text wrap
Reflect on the previous sessions we've had together. Use subagents to identify core themes and lessons, and store them in [X]. Make sure you know to reference [X] for future use.
```

## 提前停止的罕见情况

在长时间会话的深处，Claude Fable 5 偶尔会以仅文本的意图声明（"我现在将运行 X"）结束一个回合，而不发出相应的工具调用，或者在它已经有足够信息继续时暂停以请求许可。一个"继续"或"继续端到端地完成它"就足够了。要定义何时适合暂停，请将此与[强大的指令遵循](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5#strong-instruction-following)中的检查点指令配对。对于自主管道，添加系统提醒：

```text wrap
You are operating autonomously. The user is not watching in real time and cannot answer questions mid-task, so asking "Want me to…?" or "Shall I…?" will block the work. For reversible actions that follow from the original request, proceed without asking. Offering follow-ups after the task is done is fine; asking permission after already discussing with the user before doing the work is not. Before ending your turn, check your last paragraph. If it is a plan, an analysis, a question, a list of next steps, or a promise about work you have not done ("I'll…", "let me know when…"), do that work now with tool calls. End your turn only when the task is complete or you are blocked on input only the user can provide.
```

## 上下文预算担忧的罕见情况

在非常长的会话中，Claude Fable 5 偶尔会建议开始新会话、提议总结并移交，或削减自己的工作。这最常在框架向模型显示剩余令牌倒计时时触发。尽可能避免显示明确的上下文预算计数。如果框架必须显示它们，一句安慰会有所帮助：

```text wrap
You have ample context remaining. Do not stop, summarize, or suggest a new session on account of context limits. Continue the work.
```

## 给出原因，而不仅仅是请求

当 Claude Fable 5 理解请求背后的意图时，它往往表现更好：上下文让它能够将任务与相关信息联系起来，而不是自行推断意图。提供关于您为什么提出请求的上下文，尤其是对于依赖多个工作流的长时间运行代理：

```text wrap
I'm working on [the larger task] for [who it's for]. They need [what the output enables]. With that in mind: [request].
```

## 与用户沟通时的可读性

在扩展或代理式对话中（许多工具调用、大型工作上下文），Claude Fable 5 可能会产生难以理解的文本：密集的箭头链速记、深层实现细节、对用户从未看到的思考的引用，或过于技术性的措辞。一个沟通风格附录可以缓解这种情况：

```text wrap
Terse shorthand is fine between tool calls (that's you thinking out loud, and brevity there is good). Your final summary is different: it's for a reader who didn't see any of that.

If you've been working for a while without the user watching (overnight, across many tool calls, since they last spoke), your final message is their first look at any of it. Write it as a re-grounding, not a continuation of your working thread: the outcome first, then the one or two things you need from them, each explained as if new. The vocabulary you built up while working is yours, not theirs; leave it behind unless you re-introduce it.

When you write the summary at the end, drop the working shorthand. Write complete sentences. Spell out terms. Don't use arrow chains, hyphen-stacked compounds, or labels you made up earlier. When you mention files, commits, flags, or other identifiers, give each one its own plain-language clause. Open with the outcome: one sentence on what happened or what you found. Then the supporting detail. If you have to choose between short and clear, choose clear.
```

## 创建发送给用户的工具

在运行长时间、异步代理时，给代理一种方式来呈现用户必须完全按原样看到的消息，而不结束其回合：一个交付物（生成的代码片段或起草的消息）、带有具体数字的进度更新，或对用户在循环中途提出的问题的直接回复。该工具的输入是要显示的消息；当 Claude 调用它时，直接在您的 UI 中渲染输入，并返回一个简单的确认作为工具结果。工具输入永远不会被摘要，因此内容完整到达。

```json
{
  "name": "send_to_user",
  "description": "Display a message directly to the user. Use this for progress updates, partial results, or content the user must see exactly as written before the task finishes.",
  "input_schema": {
    "type": "object",
    "properties": {
      "message": {
        "type": "string",
        "description": "The content to display to the user."
      }
    },
    "required": ["message"]
  }
}
```

每当您的用户体验依赖于在任务中途逐字交付内容或直接用户交互时，添加此工具。对于仅叙述常规进度的代理，模型自己的摘要通常就足够了。仅定义工具本身是不够的；如果系统提示中没有指令，Claude Fable 5 很少会调用它。将工具与引导语言配对，例如：

```text wrap
Between tool calls, when you have content the user must read verbatim (a partial deliverable, a direct answer to their question), call the send_to_user tool with that content. Use send_to_user only for user-facing content, not for narration or reasoning.
```

不要通过 `send_to_user` 路由叙述或内部推理；为非面向用户的内容过度调用它会违背其目的。

## 推荐的脚手架变更

* **从您难度范围的顶端开始。** 选择一个比您分配给先前模型更难的任务，让 Claude Fable 5 界定范围、提出澄清问题并执行。
* **在长时间运行的提示中明确自我验证。** 独立的、全新上下文的验证器子代理往往优于自我批评。对于长时间运行的任务，指示：`Establish a method for checking your own work at an interval of [X] as you build. Run this every [X interval], verifying your work with subagents against the specification.`
* **重构现有的提示和技能。** 为先前模型开发的技能对于 Claude Fable 5 来说往往过于规定性，并可能降低输出质量。如果默认性能更好，请审查并考虑删除较旧的指令。Claude Fable 5 还能很好地根据它从手头任务中学到的内容即时更新技能。
* **不要指示 Claude 在响应中重现其推理。** 告诉模型在响应文本中回显、转录或解释其内部推理的提示、技能或框架指令可能会触发 Claude Fable 5 上的 [`reasoning_extraction` 拒绝类别](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response)，导致向 Claude Opus 4.8 的回退增加。迁移时审核现有技能和系统提示中的反思或展示思考指令。如果您的应用程序需要推理可见性，请改为从[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)读取结构化的 `thinking` 块，并使用[发送给用户的工具](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5#create-a-send-to-user-tool)在长时间运行期间呈现进度。
* **创建发送给用户的工具。** 对于长时间、异步代理，客户端工具可以逐字向用户交付消息而不结束回合。请参阅[创建发送给用户的工具](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5#create-a-send-to-user-tool)。
