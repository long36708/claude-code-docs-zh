---
title: Claude Sonnet 4.6
url: https://platform.claude.com/docs/zh-CN/models/sonnet-4-6/overview
description: Claude Sonnet 4.6 参考：生命周期状态、各平台上的模型 ID、上下文窗口、输出限制、定价以及迁移资源。Claude Sonnet 4.6 是旧版模型；Claude Sonnet 5 是当前的 Sonnet 模型。
---

**Legacy.** Released February 17, 2026.

Although Claude Sonnet 4.6 is still available, you should consider migrating to Claude Sonnet 5 for improved performance. [See Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview) · [Migrate to Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide#migrating-from-claude-sonnet-4-6-to-claude-sonnet-5)

Model ID: `claude-sonnet-4-6`

Context window: 1M tokens · Max output: 128K tokens · Input pricing: $3 / MTok · Output pricing: $15 / MTok

[Announcement](https://www.anthropic.com/news/claude-sonnet-4-6)

## 与当前模型阵容的比较

| Model                                                                                | Context | Max output | Price / MTok | Thinking                       | Default effort | Knowledge cutoff |
| :----------------------------------------------------------------------------------- | :------ | :--------- | :----------- | :----------------------------- | :------------- | :--------------- |
| [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) | 1M      | 128K       | $10 / $50    | Adaptive (always on)           | `high`         | Jun 2026         |
| [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/overview)       | 1M      | 128K       | $5 / $25     | Adaptive                       | `high`         | May 2026         |
| [Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview)   | 1M      | 128K       | $2 / $10     | Adaptive                       | `high`         | Jan 2026         |
| **Claude Sonnet 4.6** (this model)                                                   | 1M      | 128K       | $3 / $15     | Adaptive (extended deprecated) | `high`         | Aug 2025         |
| [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/haiku-4-5/overview) | 200K    | 64K        | $1 / $5      | Extended                       | —              | Feb 2025         |

* **Context:** 1M tokens is roughly 555k words or 2.5M Unicode characters on the current tokenizer (introduced with Claude Opus 4.7); models before it fit about 750k words in 1M tokens. 200k tokens is roughly 150k words.
* **Max output:** Synchronous Messages API limit. On the Message Batches API, Claude Opus 5, Claude Sonnet 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, and Claude Sonnet 4.6 support up to 300k output tokens with the output-300k-2026-03-24 beta header.
* **Price / MTok:** Input / output, base price per million tokens. Batch API requests are 50% off; prompt caching reads cost 10% of the base input price (2.5% on Claude Fable 5.1 and Claude Mythos 5.1). See Pricing for the full list.
* **Thinking:** Adaptive thinking lets the model decide how much to think, steered by effort. Extended thinking is the manual budget\_tokens mode on earlier models.
* **Default effort:** The effort parameter’s default on the Claude API. Models without a value don’t support the parameter.
* **Knowledge cutoff:** Reliable knowledge cutoff: the date through which the model’s knowledge is most extensive and reliable.

## 规格

### Model IDs

| Platform                                                                                                                 | Model ID                      |
| :----------------------------------------------------------------------------------------------------------------------- | :---------------------------- |
| Claude API                                                                                                               | `claude-sonnet-4-6`           |
| [Amazon Bedrock (InvokeModel)](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy) | `anthropic.claude-sonnet-4-6` |
| [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)                             | `claude-sonnet-4-6`           |
| [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)                | `claude-sonnet-4-6`           |
| [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)                | `claude-sonnet-4-6`           |

### Pricing

| Feature                                                                                   | Value                                                                  |
| :---------------------------------------------------------------------------------------- | :--------------------------------------------------------------------- |
| Input                                                                                     | $3 / MTok                                                              |
| Output                                                                                    | $15 / MTok                                                             |
| [5m cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $3.75 / MTok                                                           |
| [1h cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $6 / MTok                                                              |
| [Cache read](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)     | $0.30 / MTok                                                           |
| [Batch API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)    | 50% discount on input and output                                       |
| Full price list                                                                           | [Pricing](https://platform.claude.com/docs/zh-CN/about-claude/pricing) |

### Capabilities

| Feature                                                                                                                        | Value                          |
| :----------------------------------------------------------------------------------------------------------------------------- | :----------------------------- |
| [Context window](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)                                     | 1M tokens                      |
| Max output                                                                                                                     | 128K tokens                    |
| [Max output (Batch API, beta)](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#extended-output-beta) | 300K tokens                    |
| [Thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)                                                  | Adaptive (extended deprecated) |
| [Default effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)                                              | `high`                         |
| Input → output                                                                                                                 | Text and images → text         |
| Reliable knowledge cutoff                                                                                                      | Aug 2025                       |
| Training data cutoff                                                                                                           | Jan 2026                       |

### Availability

| Feature                                                                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| :------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Status](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations) | Active (legacy)                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Released                                                                         | February 17, 2026                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Retirement                                                                       | Not sooner than February 17, 2027                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Platforms                                                                        | Claude API, [Amazon Bedrock (InvokeModel)](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy), [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai), [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry), [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) |

## 资源

<CardGroup cols={3}>
  <Card title="迁移到 Claude Sonnet 5" icon="arrows-left-right" href="https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide#migrating-from-claude-sonnet-4-6-to-claude-sonnet-5">
    从 Claude Sonnet 4.6 迁移到 Claude Sonnet 5 时会发生哪些变化。
  </Card>

  <Card title="Claude Sonnet 5" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview">
    当前的 Sonnet 模型：概述、规格和资源。
  </Card>
</CardGroup>

## 参考

<CardGroup cols={3}>
  <Card title="系统提示" icon="text" href="https://platform.claude.com/docs/zh-CN/release-notes/system-prompts#claude-sonnet-4-6">
    Claude Sonnet 4.6 在 claude.ai 和 Claude 应用中使用的 "system prompt"（系统提示）。
  </Card>

  <Card title="系统卡" icon="file" href="https://www.anthropic.com/claude-sonnet-4-6-system-card">
    Claude Sonnet 4.6 的安全评估和部署决策。
  </Card>

  <Card title="定价" icon="coins" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整价格列表，包括批量折扣和 "prompt caching"（提示缓存）费率。
  </Card>

  <Card title="模型 ID 与版本管理" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions">
    模型 ID、别名和固定快照的工作方式。
  </Card>

  <Card title="模型弃用" icon="clock" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    每个 Claude 模型的生命周期状态和退役承诺。
  </Card>

  <Card title="Amazon Bedrock（Opus 4.6 及更早版本）" icon="cloud" href="https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy">
    Claude Sonnet 4.6 使用 InvokeModel Bedrock 集成和 Bedrock 风格的模型 ID。
  </Card>
</CardGroup>
