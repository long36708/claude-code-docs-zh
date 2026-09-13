---
title: Claude Fable 5.1
url: https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview
description: Claude Fable 5.1 概览：它的用途、各平台上的模型 ID、上下文窗口、输出限制、定价、可用性，以及使用它进行构建的相关资源。
---

**Latest.** Released September 1, 2026.

For demanding reasoning and long-horizon agentic work

Model ID: `claude-fable-5-1`

Context window: 1M tokens · Max output: 128K tokens · Input pricing: $10 / MTok · Output pricing: $50 / MTok

[Announcement](https://www.anthropic.com/claude-fable-and-mythos-5-1) · [What’s new](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1) · [Migration guide](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide)

## 概述

Claude Fable 5.1 在 Claude Fable 5 的基础上进行了扩展，输入和输出价格保持不变，缓存读取成本降至四分之一，并带来了更强的长时间运行的智能体编码、多步骤研究，以及文档、电子表格和幻灯片处理能力。对于大多数工作负载，请从 Claude Opus 5 开始（请参阅[选择模型](https://platform.claude.com/docs/zh-CN/about-claude/models/choosing-a-model)）。当您需要高要求的推理和长周期智能体工作，或者在 Claude Opus 5 上以更高 effort 运行的评估仍然达不到要求时，请使用 Claude Fable 5.1。Claude Mythos 5.1 仅向 [Project Glasswing](https://anthropic.com/glasswing) 参与者提供相同的能力。

如果您已经在调用 Claude Fable 5，有三项变更是破坏性的：[强制工具使用会返回错误](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#forced-tool-use-is-not-supported)、[早期模型无法读取其思考块](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#thinking-blocks-are-tied-to-the-model-that-produced-them)，以及[编辑早期轮次会使思考块失效](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#editing-earlier-turns-invalidates-thinking-blocks)。有五项是新增的：[按消息设置 effort](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#change-effort-mid-conversation-beta)（测试版）、[轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#turn-scoped-system-messages-beta)（测试版）、[工具调用之间可读的进度更新](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#progress-updates-between-tool-calls-beta)（`display: "updates"`，测试版）、[更低的缓存读取价格](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#pricing)，以及[内容溯源](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1#content-provenance)。

[Claude Fable 5.1 的新特性](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1)

## Claude Fable 5.1 与 Claude Mythos 5.1

[Claude Mythos 5.1](https://platform.claude.com/docs/zh-CN/models/mythos-5-1/overview) 提供相同的功能，但仅限受邀使用，作为 [Project Glasswing](https://anthropic.com/glasswing) 的一部分。它与 Claude Fable 5.1 共享相同的规格和定价。如需获取访问权限，请联系您的 Anthropic、AWS 或 Google Cloud 客户团队。

## 对比情况

| Model                                                                                | Context | Max output | Price / MTok | Latency  | Thinking             | Default effort | Knowledge cutoff |
| :----------------------------------------------------------------------------------- | :------ | :--------- | :----------- | :------- | :------------------- | :------------- | :--------------- |
| **Claude Fable 5.1** (this model)                                                    | 1M      | 128K       | $10 / $50    | Slower   | Adaptive (always on) | `high`         | Jun 2026         |
| [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/overview)       | 1M      | 128K       | $5 / $25     | Moderate | Adaptive             | `high`         | May 2026         |
| [Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview)   | 1M      | 128K       | $2 / $10     | Fast     | Adaptive             | `high`         | Jan 2026         |
| [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/haiku-4-5/overview) | 200K    | 64K        | $1 / $5      | Fastest  | Extended             | —              | Feb 2025         |

* **Context:** 1M tokens is roughly 555k words or 2.5M Unicode characters on the current tokenizer (introduced with Claude Opus 4.7); models before it fit about 750k words in 1M tokens. 200k tokens is roughly 150k words.
* **Max output:** Synchronous Messages API limit. On the Message Batches API, Claude Opus 5, Claude Sonnet 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, and Claude Sonnet 4.6 support up to 300k output tokens with the output-300k-2026-03-24 beta header.
* **Price / MTok:** Input / output, base price per million tokens. Batch API requests are 50% off; prompt caching reads cost 10% of the base input price (2.5% on Claude Fable 5.1 and Claude Mythos 5.1). See Pricing for the full list.
* **Latency:** Comparative latency, relative to the current lineup, as published in the models overview. Actual latency depends on prompt length, output length, and thinking effort.
* **Thinking:** Adaptive thinking lets the model decide how much to think, steered by effort. Extended thinking is the manual budget\_tokens mode on earlier models.
* **Default effort:** The effort parameter’s default on the Claude API. Models without a value don’t support the parameter.
* **Knowledge cutoff:** Reliable knowledge cutoff: the date through which the model’s knowledge is most extensive and reliable.

## 规格

### Model IDs

| Platform                                                                                                  | Model ID                     |
| :-------------------------------------------------------------------------------------------------------- | :--------------------------- |
| Claude API                                                                                                | `claude-fable-5-1`           |
| [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)       | `anthropic.claude-fable-5-1` |
| [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)              | `claude-fable-5-1`           |
| [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) | `claude-fable-5-1`           |
| [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) | `claude-fable-5-1`           |

### Pricing

| Feature                                                                                   | Value                                                                  |
| :---------------------------------------------------------------------------------------- | :--------------------------------------------------------------------- |
| Input                                                                                     | $10 / MTok                                                             |
| Output                                                                                    | $50 / MTok                                                             |
| [5m cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $12.50 / MTok                                                          |
| [1h cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $20 / MTok                                                             |
| [Cache read](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)     | $0.25 / MTok                                                           |
| [Batch API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)    | 50% discount on input and output                                       |
| Full price list                                                                           | [Pricing](https://platform.claude.com/docs/zh-CN/about-claude/pricing) |

### Capabilities

| Feature                                                                                    | Value                  |
| :----------------------------------------------------------------------------------------- | :--------------------- |
| [Context window](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows) | 1M tokens              |
| Max output                                                                                 | 128K tokens            |
| [Thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)              | Adaptive (always on)   |
| [Default effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)          | `high`                 |
| Comparative latency                                                                        | Slower                 |
| Input → output                                                                             | Text and images → text |
| Reliable knowledge cutoff                                                                  | Jun 2026               |
| Training data cutoff                                                                       | Jun 2026               |

### Availability

| Feature                                                                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                               |
| :------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Status](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations) | Active (latest)                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Released                                                                         | September 1, 2026                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Retirement                                                                       | Not sooner than September 1, 2027                                                                                                                                                                                                                                                                                                                                                                                                   |
| Platforms                                                                        | Claude API, [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock), [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai), [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry), [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) |

## 资源

<CardGroup cols={3}>
  <Card title="Claude Fable 5.1 提示指南" icon="lightbulb" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1">
    针对长周期和智能体工作的模型专属提示指导。
  </Card>

  <Card title="迁移到 Claude Fable 5.1" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide">
    从 Claude Fable 5、Claude Opus 5 或 Claude Opus 4.8 迁移时会发生哪些变化。
  </Card>

  <Card title="保留的思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking">
    该模型的思考块在何种情况下保持可用：跨模型切换以及跨对话变更。
  </Card>

  <Card title="按消息设置努力程度" icon="sliders" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta">
    在对话进行中途更改努力程度级别，而不会使提示缓存失效。
  </Card>

  <Card title="拒绝与回退" icon="shield" href="https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback">
    处理分类器拒绝，并在另一个 Claude 模型上重试。
  </Card>

  <Card title="自适应思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    Claude Fable 5.1 上唯一的思考模式。使用 `effort` 控制思考深度。
  </Card>
</CardGroup>

## 参考

<CardGroup cols={3}>
  <Card title="系统提示" icon="text" href="https://platform.claude.com/docs/zh-CN/release-notes/system-prompts/claude-fable-5-1">
    Claude Fable 5.1 在 claude.ai 和 Claude 应用中使用的系统提示。
  </Card>

  <Card title="系统卡" icon="file" href="https://www.anthropic.com/claude-fable-5-1-mythos-5-1-system-card">
    Claude Fable 5.1 和 Claude Mythos 5.1 的安全评估与部署决策。
  </Card>

  <Card title="定价" icon="coins" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整价格列表，包括批量折扣和提示缓存费率。
  </Card>

  <Card title="模型 ID 与版本管理" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions">
    模型 ID、别名和固定快照的工作方式。
  </Card>

  <Card title="模型弃用" icon="clock" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    每个 Claude 模型的生命周期状态和退役承诺。
  </Card>
</CardGroup>
