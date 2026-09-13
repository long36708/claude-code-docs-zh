---
title: Claude Opus 4.8 提示指南
url: https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-4-8
description: Claude Opus 4.8 的行为差异与提示模式，涵盖冗长度、努力程度校准、工具使用、子智能体以及前端默认设置。
---

本指南涵盖 Claude Opus 4.8 特有的提示模式。有关从 Claude Opus 4.8 迁移到最新 Opus 模型所涉及的 API 变更，请参阅[从 Claude Opus 4.8 迁移到 Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-4-8-to-claude-opus-5)。有关适用于所有当前 Claude 模型的技术，请参阅[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)。

Claude Opus 4.8 在长周期智能体工作、知识工作、视觉和记忆任务方面具有特别的优势。它在现有的 Claude Opus 4.7 提示上开箱即用表现良好。以下模式涵盖了最常需要调优的行为。

<Note>
  有关自 Claude Opus 4.7 以来的 API 参数变更（采样参数、effort 默认值、1M 上下文窗口默认值、对话中途的系统消息以及拒绝停止详情），请参阅[从 Claude Opus 4.7 迁移到 Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-47)，该文档涵盖了迁移到最新 Opus 模型过程中的相同变更；Claude Opus 4.8 具有相同的这些行为。
</Note>

## 响应长度与冗长度

Claude Opus 4.8 会根据其判断的任务复杂程度来校准响应长度，而不是默认采用固定的"verbosity"（冗长度）。这通常意味着在简单查询上给出更短的回答，而在开放式分析上给出长得多的回答。

如果您的产品依赖于特定风格或冗长度的输出，您可能需要调优您的提示。例如，要降低冗长度，您可以添加：

```text wrap
Provide concise, focused responses. Skip non-essential context, and keep examples minimal.
```

如果您看到某些特定类型冗长的具体示例（例如过度解释），您可以在提示中添加额外的指令来防止它们。展示 Claude 如何以适当简洁程度进行沟通的正面示例，往往比负面示例或告诉模型不要做什么的指令更有效。

## 校准努力程度与思考深度

[effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)允许您在 Claude 的智能与令牌消耗之间进行调节，以能力换取更快的速度和更低的成本。对于编码和智能体用例，请从 `xhigh` 努力级别开始；对于大多数对智能敏感的用例，请至少使用 `high` 努力级别。尝试其他努力级别以进一步调节令牌用量和智能：

* **`max`：** 最大努力在某些用例中可以带来性能提升，但可能因令牌用量增加而出现收益递减。此设置有时也容易过度思考。请针对对智能要求高的任务测试 max 努力级别。
* **`xhigh`：** 超高努力是大多数编码和智能体用例的最佳设置。
* **`high`：** 此设置在令牌用量和智能之间取得平衡。对于大多数对智能敏感的用例，请至少使用 `high` 努力级别。
* **`medium`：** 适合需要减少令牌用量并愿意以智能作为交换的成本敏感型用例。
* **`low`：** 保留用于简短、范围明确的任务以及对智能不敏感的延迟敏感型工作负载。

Claude Opus 4.8 严格遵守努力级别，尤其是在低端。在 `low` 和 `medium` 下，模型会将其工作范围限定在所要求的内容上，而不会超额完成。这对延迟和成本有利，但对于以 `low` 努力级别运行的中等复杂任务，存在一定的思考不足风险。

如果您在复杂问题上观察到浅层推理，请将努力级别提高到 `high` 或 `xhigh`，而不是通过提示来绕过它。如果您出于延迟考虑需要将努力级别保持在 `low`，请添加有针对性的指导：

```text wrap
This task involves multistep reasoning. Think carefully through the problem before responding.
```

对于此模型而言，努力级别可能比以往任何 Opus 都更重要，因此在升级时请积极地对其进行试验。

在 Claude Opus 4.8 上，除非您显式设置 `thinking: {type: "adaptive"}`，否则思考功能处于关闭状态。[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)的触发行为是可引导的。如果您发现模型思考的频率高于您的期望（这在系统提示庞大或复杂时可能发生），请添加指导来引导它。一如既往，请衡量任何提示变更对性能的影响。示例：

```text wrap
Thinking adds latency and should only be used when it will meaningfully improve answer quality — typically for problems that require multistep reasoning. When in doubt, respond directly.
```

反之，如果您在 `medium` 下运行困难的工作负载并看到思考不足，首要手段是提高努力级别。如果您需要更精细的控制，请直接通过提示来要求。

<Note>
  如果您以 `max` 或 `xhigh` 努力级别运行 Claude Opus 4.8，请设置较大的最大输出令牌预算，以便模型有空间在其子智能体和工具调用之间进行思考和行动。从 64k 令牌开始，并在此基础上调优。
</Note>

## 工具使用触发

Claude Opus 4.8 倾向于优先进行推理而非工具调用。这在大多数情况下会产生更好的结果。然而，提高努力设置是增加工具使用程度的有用手段，尤其是在知识工作中。`high` 或 `xhigh` 努力设置在智能体搜索和编码中表现出明显更多的工具使用。对于您希望更多工具使用的场景，您也可以调整提示，明确指示模型何时以及如何正确使用其工具。例如，如果您发现模型没有使用您的网络搜索工具，请清楚地描述它为什么应该使用以及应该如何使用。

## 面向用户的进度更新

Claude Opus 4.8 在长智能体轨迹中会向用户提供更规律、更高质量的更新。如果您曾添加脚手架来强制输出中间状态消息（"每 3 次工具调用后，总结进度"），请尝试将其移除。如果您发现 Claude Opus 4.8 面向用户的更新的长度或内容未能很好地适配您的用例，请在提示中明确描述这些更新应该是什么样子并提供示例。

## 更字面化的指令遵循

Claude Opus 4.8 会按字面和明确的方式解读提示，尤其是在较低的努力级别下。它不会默默地将一条指令从一个项目泛化到另一个项目，也不会推断您未提出的请求。这种字面化的好处是精确和更少的反复折腾，并且对于具有精心调优提示的 API 用例、结构化提取以及您希望行为可预测的流水线，它通常表现更好。如果您需要 Claude 广泛地应用某条指令，请明确说明范围（例如，"将此格式应用于每个部分，而不仅仅是第一个部分"）。

## 语气与写作风格

与任何新模型一样，长篇写作的文风可能会发生变化。Claude Opus 4.8 倾向于直接、有主见的风格，极少使用以认同为先的措辞，并且很少使用表情符号。如果您的产品依赖于特定的语气，请对照新的基线重新评估风格提示。

例如，如果您的产品语气更温暖或更具对话性，请添加：

```text wrap
Use a warm, collaborative tone. Acknowledge the user's framing before answering.
```

## 控制子智能体生成

Claude Opus 4.8 默认倾向于生成更少的"subagent"（子智能体）。然而，此行为可通过提示进行引导；请就何时需要子智能体向 Claude Opus 4.8 提供明确指导。以下是一个编码用例的简单示例：

```text wrap
Do not spawn a subagent for work you can complete directly in a single response (e.g. refactoring a function you can already see).

Spawn multiple subagents in the same turn when fanning out across items or reading multiple files.
```

## 设计与前端默认设置

Claude Opus 4.8 具有强烈的设计直觉，并有一致的默认自有风格：温暖的奶油色/米白色背景（约 `#F4F1EA`）、衬线展示字体（Georgia、Fraunces、Playfair）、斜体单词强调，以及赤陶色/琥珀色强调色。这对于编辑类、酒店服务类和作品集类需求效果很好，但对于仪表盘、开发工具、金融科技、医疗保健或企业应用会显得不合适。该默认风格会出现在幻灯片和 Web UI 中。

此默认风格具有持续性。通用指令（"不要使用奶油色"、"让它干净简约"）往往会使模型转向另一种固定的配色方案，而不是产生多样性。有两种方法可靠有效：

**1. 指定具体的替代方案。** 模型会精确遵循明确的规格：

```text wrap
Design a desktop landing page for a supplement brand called AEFRM.

The visual direction should come from a cold monochrome atmosphere using pale silver-gray tones that gradually deepen into blue-gray and near-black, similar to a misted metallic surface.

The page should feel sharp and controlled, with a strong sense of structure and restraint.

Use this tonal system across the full page instead of introducing bright accent colors.

Use the uploaded image on the hero design in black and white.

The layout should be built with clear horizontal sections and a centered max-width container. Use 4px corner radius consistently across cards, buttons, inputs, and media frames. Margins should feel generous, with enough empty space around each section so the page breathes.

Typography should use a square, angular sans-serif with wider letter spacing than usual, especially in headings and navigation, so the text feels more engineered and less compressed. Headline text can be large and uppercase, while supporting copy remains short and sparse. The sub texts should be written with Alumni Sans SC in 4-6px like tiny little texts on corners bottom centre like that.

For the structure, start with a hero section containing a strong product statement, one short supporting paragraph, and a clean product placeholder or packshot frame. Below that, add a benefit grid with three or four blocks, then a formulation or ingredients section, and finally a cta.

Buttons should be flat and precise, with subtle hover changes using transition: all 160ms ease out where brightness and border contrast shift slightly rather than using dramatic motion.

Color palette should stay within this range:
#E9ECEC, #C9D2D4, #8C9A9E, #44545B, #11171B.
```

**2. 让模型在构建之前提出选项。** 这会打破默认风格并让用户掌握控制权。如果您之前依赖 `temperature` 来获得设计多样性，请使用此方法；它会在多次运行中产生有意义的不同方向。示例提示：

```text wrap
Before building, propose 4 distinct visual directions tailored to this brief (each as: bg hex / accent hex / typeface — one-line rationale). Ask the user to pick one, then implement only that direction.
```

此外，与之前的模型相比，Claude Opus 4.8 需要更少的前端设计提示即可避免用户称之为"AI slop"（AI 粗制品）美学的通用模式。对于早期模型，Anthropic 曾推荐使用 [frontend-design skill](https://github.com/anthropics/claude-code/blob/main/plugins/frontend-design/skills/frontend-design/SKILL.md) 中较长的提示片段。然而，Claude Opus 4.8 只需更精简的提示指导即可生成独特、有创意的前端。此提示片段与前述关于多样性的提示建议配合使用效果良好：

```text wrap
<frontend_aesthetics>
NEVER use generic AI-generated aesthetics like overused font families (Inter, Roboto, Arial, system fonts), cliched color schemes (particularly purple gradients on white or dark backgrounds), predictable layouts and component patterns, and cookie-cutter design that lacks context-specific character. Use unique fonts, cohesive colors and themes, and animations for effects and micro-interactions.
</frontend_aesthetics>
```

## 交互式编码产品

Claude Opus 4.8 的令牌用量和行为在只有单个用户轮次的自主、异步编码智能体与具有多个用户轮次的交互式、同步编码智能体之间可能有所不同。具体而言，它在交互式环境中倾向于使用更多令牌，主要是因为它在用户轮次之后会进行更多推理。这可以在长时间的交互式编码会话中改善长周期连贯性、指令遵循和编码能力，但也伴随着更多的令牌用量。为了在编码产品中同时最大化性能和令牌效率，请使用 `xhigh` 或 `high` 努力级别，添加自动模式等自主功能，并减少用户所需的人工交互次数。

当然，在限制所需用户交互次数时，重要的是在第一个人类轮次中预先指定任务、意图和相关约束。预先提供规格完善、清晰且准确的任务描述，有助于最大化自主性和智能，同时最小化用户轮次之后的额外令牌用量。由于 Claude Opus 4.8 比之前的模型更加自主，这种使用模式有助于最大化性能。相比之下，在多个用户轮次中逐步传达的模糊或规格不足的提示往往会相对降低令牌效率，有时还会降低性能。

## 代码审查框架

Claude Opus 4.8 在发现 bug 方面明显优于之前的模型，并且在内部评估中具有更高的召回率和精确率。然而，如果您的代码审查框架是针对早期模型调优的，您最初可能会看到较低的召回率。这很可能是框架效应，而非能力退化。当审查提示中包含诸如"仅报告高严重性问题"、"保持保守"或"不要吹毛求疵"之类的内容时，Claude Opus 4.8 可能会比早期模型更忠实地遵循该指令：它可能会同样彻底地调查代码、识别出 bug，然后不报告它判断为低于您所述标准的发现。这可能表现为模型进行了相同深度的调查，但将更少的调查转化为报告的发现，尤其是在较低严重性的 bug 上。精确率通常会上升，但测得的召回率可能会下降，即使模型底层的 bug 发现能力已经提高。

一些推荐的提示用语：

```text wrap
Report every issue you find, including ones you are uncertain about or consider low-severity. Do not filter for importance or confidence at this stage - a separate verification step will do that. Your goal here is coverage: it is better to surface a finding that later gets filtered out than to silently drop a real bug. For each finding, include your confidence level and an estimated severity so a downstream filter can rank them.
```

此提示可以在没有实际第二步的情况下使用，但将置信度过滤移出发现步骤通常会有帮助。如果您的框架有单独的验证、去重或排序阶段，请明确告诉模型它在发现阶段的工作是覆盖而非过滤。

如果您确实希望模型在单次处理中进行自我过滤，请具体说明标准在哪里，而不是使用"重要"之类的定性术语：例如，"报告任何可能导致不正确行为、测试失败或误导性结果的 bug；仅省略纯粹的风格或命名偏好之类的细枝末节。"

针对您的评估或测试用例的子集迭代提示，以验证召回率或 F1 分数的提升。

## 计算机使用

Claude Opus 4.8 支持 `computer_toolset_20260801` 工具集（在 Claude API 和 Google Cloud 上）以及更早的 `computer_20251124` 工具版本。对于网页内的任务，Claude Opus 4.8 还支持[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)（`browser_toolset_20260801`）。[计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)能力可跨分辨率工作，最高分辨率为 2576px / 3.75MP。内部计算机使用测试表明，以 1080p 发送图像可在性能和成本之间取得良好平衡。

对于特别成本敏感的工作负载，720p 或 1366×768 是性能强劲的低成本选项。请自行进行测试以找到适合您用例的理想设置；尝试不同的努力设置也有助于调节模型的行为。
