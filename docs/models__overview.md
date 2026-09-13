---
title: 模型概览
url: https://platform.claude.com/docs/zh-CN/models/overview
description: Claude 是由 Anthropic 开发的一系列最先进的大型语言模型。本指南介绍可用的模型并比较它们的性能。
---

# 模型概览

Claude 是由 Anthropic 开发的一系列最先进的大型语言模型。比较当前的模型阵容，查找每个平台的模型 ID，并打开每个模型的页面以查看其完整规格和资源。

<HomeQuickChip icon="Signpost" href="https://platform.claude.com/docs/zh-CN/about-claude/models/choosing-a-model">
  选择模型
</HomeQuickChip>

<HomeQuickChip icon="DollarSign" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
  定价
</HomeQuickChip>

<HomeQuickChip icon="ArrowUpCircle" href="https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide">
  迁移指南
</HomeQuickChip>

## 比较模型

如果您不确定使用哪个模型，对于大多数工作负载，请从 [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/overview) 开始。对于要求严苛的推理和长周期智能体工作，或者当您在更高努力程度下对 Claude Opus 5 的评估仍然不达标时，请使用 [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview)。所有当前模型都支持文本和图像输入、文本输出、多语言能力、视觉以及 "tool use"（工具使用）。每个模型的页面都列出了其可用的平台。

| Feature                                                                                                      | Claude Fable 5.1                                                                     | Claude Opus 5                                                                  | Claude Sonnet 5                                                                    | Claude Haiku 4.5                                                                     |
| :----------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------- | :----------------------------------------------------------------------------- | :--------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------- |
| Description                                                                                                  | For demanding reasoning and long-horizon agentic work                                | For complex agentic coding and enterprise work                                 | The best combination of speed and intelligence                                     | The fastest model with near-frontier intelligence                                    |
| Model page                                                                                                   | [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) | [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/overview) | [Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview) | [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/haiku-4-5/overview) |
| Comparative latency                                                                                          | Slower                                                                               | Moderate                                                                       | Fast                                                                               | Fastest                                                                              |
| [Pricing](https://platform.claude.com/docs/zh-CN/about-claude/pricing)                                       | $10 / input MTok, $50 / output MTok                                                  | $5 / input MTok, $25 / output MTok                                             | $2 / input MTok, $10 / output MTok                                                 | $1 / input MTok, $5 / output MTok                                                    |
| Claude API ID                                                                                                | `claude-fable-5-1`                                                                   | `claude-opus-5`                                                                | `claude-sonnet-5`                                                                  | `claude-haiku-4-5-20251001`                                                          |
| [Thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)                                | Adaptive (always on)                                                                 | Adaptive                                                                       | Adaptive                                                                           | Extended                                                                             |
| [Default effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)                            | `high`                                                                               | `high`                                                                         | `high`                                                                             | Not supported                                                                        |
| [Context window](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)                   | 1M tokens                                                                            | 1M tokens                                                                      | 1M tokens                                                                          | 200K tokens                                                                          |
| Max output                                                                                                   | 128K tokens                                                                          | 128K tokens                                                                    | 128K tokens                                                                        | 64K tokens                                                                           |
| Reliable knowledge cutoff                                                                                    | Jun 2026                                                                             | May 2026                                                                       | Jan 2026                                                                           | Feb 2025                                                                             |
| Training data cutoff                                                                                         | Jun 2026                                                                             | May 2026                                                                       | Jan 2026                                                                           | Jul 2025                                                                             |
| [Retirement](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)                         | Not sooner than September 1, 2027                                                    | Not sooner than July 24, 2027                                                  | Not sooner than June 30, 2027                                                      | Not sooner than October 15, 2026                                                     |
| Claude API alias                                                                                             | `claude-fable-5-1`                                                                   | `claude-opus-5`                                                                | `claude-sonnet-5`                                                                  | `claude-haiku-4-5`                                                                   |
| [Amazon Bedrock ID](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)       | `anthropic.claude-fable-5-1`                                                         | `anthropic.claude-opus-5`                                                      | `anthropic.claude-sonnet-5`                                                        | `anthropic.claude-haiku-4-5`                                                         |
| [Google Cloud ID](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)              | `claude-fable-5-1`                                                                   | `claude-opus-5`                                                                | `claude-sonnet-5`                                                                  | `claude-haiku-4-5@20251001`                                                          |
| [Microsoft Foundry ID](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) | `claude-fable-5-1`                                                                   | `claude-opus-5`                                                                | `claude-sonnet-5`                                                                  | `claude-haiku-4-5`                                                                   |
| [Claude Platform on AWS ID](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) | `claude-fable-5-1`                                                                   | —                                                                              | `claude-sonnet-5`                                                                  | `claude-haiku-4-5`                                                                   |

* **Comparative latency:** Relative to the current lineup. Actual latency depends on prompt length, output length, and thinking effort.
* **Pricing:** Base price per million tokens. Batch API requests are 50% off; prompt cache reads cost 10% of the base input price (2.5% on Claude Fable 5.1 and Claude Mythos 5.1). See Pricing for cache writes, long-context, and per-platform pricing.
* **Claude API ID:** Every Claude model ID is a pinned snapshot, including the dateless IDs used from the 4.6 generation on.
* **Thinking:** Adaptive thinking lets the model decide how much to think, steered by effort. Extended thinking is the manual thinking.type “enabled” + budget\_tokens mode on earlier models; it is deprecated on Claude Opus 4.6 and Claude Sonnet 4.6 and not accepted on later models.
* **Default effort:** The effort parameter’s default on the Claude API. Set effort explicitly to use a different level.
* **Context window:** 1M tokens is roughly 555k words or 2.5M Unicode characters on the current tokenizer (introduced with Claude Opus 4.7); models before it fit about 750k words in 1M tokens. 200k tokens is roughly 150k words.
* **Max output:** Synchronous Messages API limit. On the Message Batches API, Claude Opus 5, Claude Sonnet 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, and Claude Sonnet 4.6 support up to 300k output tokens with the output-300k-2026-03-24 beta header.
* **Reliable knowledge cutoff:** The date through which the model’s knowledge is most extensive and reliable. Training data cutoff (under Show all details) is the broader range of data used. See Anthropic’s Transparency Hub for details.
* **Retirement:** Anthropic’s commitment for Anthropic-operated platforms (Claude API, Claude Platform on AWS, Microsoft Foundry). Amazon Bedrock and Google Cloud set their own dates.
* **Claude API alias:** For models before the 4.6 generation, the alias is a convenience pointer that resolves to the dated ID. Dateless IDs are their own pinned snapshot; the alias row repeats them.
* **Amazon Bedrock ID:** The ID on Bedrock’s Messages-API endpoint (Claude Opus 4.7 and later, plus Claude Haiku 4.5); a model offered only through Bedrock’s InvokeModel integration shows that ID instead. Bedrock offers global endpoints (dynamic routing) and regional endpoints (guaranteed data routing) for Claude Sonnet 4.5 and later, and sets its own lifecycle dates.
* **Google Cloud ID:** Google Cloud offers global, multi-region, and regional endpoints, and sets its own lifecycle dates.
* **Microsoft Foundry ID:** Foundry deployments default to the Claude API model ID (the alias, where one exists); the deployment name is what you send. Foundry follows the Claude API lifecycle schedule.
* **Claude Platform on AWS ID:** Claude Platform on AWS uses the Claude API model IDs (the dateless form where the Claude API has an alias), not Bedrock-style IDs, and follows Anthropic’s first-party model lifecycle.

See [Model IDs and versioning](https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions) and [Pricing](https://platform.claude.com/docs/zh-CN/about-claude/pricing).

Legacy models (still available): [Claude Fable 5](https://platform.claude.com/docs/zh-CN/models/fable-5/overview), [Claude Opus 4.8](https://platform.claude.com/docs/zh-CN/models/opus-4-8/overview), [Claude Opus 4.7](https://platform.claude.com/docs/zh-CN/models/opus-4-7/overview), [Claude Opus 4.6](https://platform.claude.com/docs/zh-CN/models/opus-4-6/overview), [Claude Opus 4.5](https://platform.claude.com/docs/zh-CN/models/opus-4-5/overview), [Claude Sonnet 4.6](https://platform.claude.com/docs/zh-CN/models/sonnet-4-6/overview), [Claude Sonnet 4.5](https://platform.claude.com/docs/zh-CN/models/sonnet-4-5/overview).

选定模型后，请[了解如何进行您的第一次 API 调用](https://platform.claude.com/docs/zh-CN/get-started)。要了解模型 ID、别名和快照的工作原理，请参阅[模型 ID 与版本管理](https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions)；要了解每个模型背后的可靠知识截止日期和训练数据截止日期，请参阅 [Anthropic 透明度中心](https://www.anthropic.com/transparency)。

## 使用 Models API

您可以通过 [Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 以编程方式查询模型能力和令牌限制。响应中包含每个可用模型的 `max_input_tokens`、`max_tokens` 以及一个 `capabilities` 对象。

## 提示与输出性能

当前的 Claude 模型在以下方面表现出色：

* **性能：** 在推理、编码、多语言任务、长上下文处理、诚实性和图像处理方面均达到顶级水平。有关通用及特定模型的提示指导，请参阅[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)。
* **引人入胜的响应：** Claude 模型非常适合需要丰富、类人交互的应用程序。如果您更喜欢简洁的响应，请调整您的提示以引导模型生成所需长度的输出。详情请参阅[提示工程指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering)。
* **输出质量：** 从上一代模型迁移时，您可能会注意到整体性能有较大提升。如果您使用的是 Claude Opus 4.8 或更早版本，请参阅[迁移到 Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide)。

## 开始使用 Claude

如果您已准备好开始探索 Claude 能为您做什么，那就开始吧！无论您是希望将 Claude 集成到应用程序中的开发者，还是想亲身体验 AI 强大功能的用户，以下资源都能为您提供帮助。

<CardGroup cols={3}>
  <Card title="Claude 简介" icon="check" href="https://platform.claude.com/docs/zh-CN/intro">
    探索 Claude 的能力和开发流程。
  </Card>

  <Card title="快速入门" icon="lightning" href="https://platform.claude.com/docs/zh-CN/get-started">
    了解如何在几分钟内进行您的第一次 API 调用。
  </Card>

  <Card title="选择模型" icon="compass" href="https://platform.claude.com/docs/zh-CN/about-claude/models/choosing-a-model">
    建立标准并为您的用例选择合适的模型。
  </Card>

  <Card title="定价" icon="coins" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整定价，包括批量折扣和 "prompt caching"（提示缓存）费率。
  </Card>

  <Card title="模型弃用" icon="clock" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    每个模型的生命周期状态和退役承诺。
  </Card>

  <Card title="Claude Console" icon="code" href="https://platform.claude.com/">
    直接在浏览器中编写和测试提示。
  </Card>
</CardGroup>

想与 Claude 聊天吗？请访问 [claude.ai](https://claude.ai)。如果您有疑问，请联系[支持团队](https://support.claude.com/)或 [Discord 社区](https://www.anthropic.com/discord)。
