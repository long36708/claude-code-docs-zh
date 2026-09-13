---
title: Claude Sonnet 4.5
url: https://platform.claude.com/docs/zh-CN/models/sonnet-4-5/overview
description: Claude Sonnet 4.5 参考：生命周期状态、各平台上的模型 ID、上下文窗口、输出限制、定价和迁移资源。Claude Sonnet 4.5 是旧版模型；Claude Sonnet 5 是当前的 Sonnet 模型。
---

**Legacy.** Released September 29, 2025.

Although Claude Sonnet 4.5 is still available, you should consider migrating to Claude Sonnet 5 for improved performance. [See Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview) · [Migrate to Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide#migrating-from-sonnet-45)

Model ID: `claude-sonnet-4-5-20250929`

Context window: 200K tokens · Max output: 64K tokens · Input pricing: $3 / MTok · Output pricing: $15 / MTok

[Announcement](https://www.anthropic.com/news/claude-sonnet-4-5)

## 与当前产品阵容的比较

| Model                                                                                | Context | Max output | Price / MTok | Thinking             | Default effort | Knowledge cutoff |
| :----------------------------------------------------------------------------------- | :------ | :--------- | :----------- | :------------------- | :------------- | :--------------- |
| [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) | 1M      | 128K       | $10 / $50    | Adaptive (always on) | `high`         | Jun 2026         |
| [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/overview)       | 1M      | 128K       | $5 / $25     | Adaptive             | `high`         | May 2026         |
| [Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview)   | 1M      | 128K       | $2 / $10     | Adaptive             | `high`         | Jan 2026         |
| **Claude Sonnet 4.5** (this model)                                                   | 200K    | 64K        | $3 / $15     | Extended             | —              | Jan 2025         |
| [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/haiku-4-5/overview) | 200K    | 64K        | $1 / $5      | Extended             | —              | Feb 2025         |

* **Context:** 1M tokens is roughly 555k words or 2.5M Unicode characters on the current tokenizer (introduced with Claude Opus 4.7); models before it fit about 750k words in 1M tokens. 200k tokens is roughly 150k words.
* **Max output:** Synchronous Messages API limit. On the Message Batches API, Claude Opus 5, Claude Sonnet 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, and Claude Sonnet 4.6 support up to 300k output tokens with the output-300k-2026-03-24 beta header.
* **Price / MTok:** Input / output, base price per million tokens. Batch API requests are 50% off; prompt caching reads cost 10% of the base input price (2.5% on Claude Fable 5.1 and Claude Mythos 5.1). See Pricing for the full list.
* **Thinking:** Adaptive thinking lets the model decide how much to think, steered by effort. Extended thinking is the manual budget\_tokens mode on earlier models.
* **Default effort:** The effort parameter’s default on the Claude API. Models without a value don’t support the parameter.
* **Knowledge cutoff:** Reliable knowledge cutoff: the date through which the model’s knowledge is most extensive and reliable.

## 规格

### Model IDs

| Platform                                                                                                                 | Model ID                                    |
| :----------------------------------------------------------------------------------------------------------------------- | :------------------------------------------ |
| Claude API                                                                                                               | `claude-sonnet-4-5-20250929`                |
| Claude API alias                                                                                                         | `claude-sonnet-4-5`                         |
| [Amazon Bedrock (InvokeModel)](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy) | `anthropic.claude-sonnet-4-5-20250929-v1:0` |
| [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)                             | `claude-sonnet-4-5@20250929`                |
| [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)                | `claude-sonnet-4-5`                         |
| [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)                | `claude-sonnet-4-5`                         |

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

| Feature                                                                                    | Value                  |
| :----------------------------------------------------------------------------------------- | :--------------------- |
| [Context window](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows) | 200K tokens            |
| Max output                                                                                 | 64K tokens             |
| [Thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)              | Extended               |
| [Default effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)          | Not supported          |
| Input → output                                                                             | Text and images → text |
| Reliable knowledge cutoff                                                                  | Jan 2025               |
| Training data cutoff                                                                       | Jul 2025               |

### Availability

| Feature                                                                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| :------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Status](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations) | Active (legacy)                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Released                                                                         | September 29, 2025                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Retirement                                                                       | Not sooner than September 29, 2026                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Platforms                                                                        | Claude API, [Amazon Bedrock (InvokeModel)](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy), [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai), [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry), [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) |

## 资源

<CardGroup cols={3}>
  <Card title="迁移到 Claude Sonnet 5" icon="arrows-left-right" href="https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide#migrating-from-sonnet-45">
    从 Claude Sonnet 4.5 及更早的 Sonnet 模型迁移到 Claude Sonnet 5 时会发生哪些变化。
  </Card>

  <Card title="Claude Sonnet 5" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview">
    当前的 Sonnet 模型：概述、规格和资源。
  </Card>
</CardGroup>

## 参考

<CardGroup cols={3}>
  <Card title="系统提示" icon="text" href="https://platform.claude.com/docs/zh-CN/release-notes/system-prompts#claude-sonnet-4-5">
    Claude Sonnet 4.5 在 claude.ai 和 Claude 应用上使用的系统提示。
  </Card>

  <Card title="系统卡" icon="file" href="https://www.anthropic.com/claude-sonnet-4-5-system-card">
    Claude Sonnet 4.5 的安全评估和部署决策。
  </Card>

  <Card title="定价" icon="coins" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整价格列表，包括批量折扣和提示缓存费率。
  </Card>

  <Card title="模型 ID 和版本控制" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions">
    模型 ID、别名和固定快照的工作原理。
  </Card>

  <Card title="模型弃用" icon="clock" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    每个 Claude 模型的生命周期状态和退役承诺。
  </Card>

  <Card title="Amazon Bedrock（Opus 4.6 及更早版本）" icon="cloud" href="https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy">
    Claude Sonnet 4.5 使用 InvokeModel Bedrock 集成和 Bedrock 风格的模型 ID。
  </Card>
</CardGroup>
