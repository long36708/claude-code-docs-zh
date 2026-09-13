---
title: Claude Opus 4.7
url: https://platform.claude.com/docs/zh-CN/models/opus-4-7/overview
description: Claude Opus 4.7 参考：生命周期状态、各平台上的模型 ID、上下文窗口、输出限制、定价和迁移资源。Claude Opus 4.7 是旧版模型；Claude Opus 5 是当前的 Opus 模型。
---

**Legacy.** Released April 16, 2026.

Although Claude Opus 4.7 is still available, you should consider migrating to Claude Opus 5 for improved performance. [See Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/overview) · [Migrate to Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-47)

Model ID: `claude-opus-4-7`

Context window: 1M tokens · Max output: 128K tokens · Input pricing: $5 / MTok · Output pricing: $25 / MTok

[Announcement](https://www.anthropic.com/news/claude-opus-4-7)

## 与当前产品阵容的比较

| Model                                                                                | Context | Max output | Price / MTok | Thinking             | Default effort | Knowledge cutoff |
| :----------------------------------------------------------------------------------- | :------ | :--------- | :----------- | :------------------- | :------------- | :--------------- |
| [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) | 1M      | 128K       | $10 / $50    | Adaptive (always on) | `high`         | Jun 2026         |
| [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/overview)       | 1M      | 128K       | $5 / $25     | Adaptive             | `high`         | May 2026         |
| **Claude Opus 4.7** (this model)                                                     | 1M      | 128K       | $5 / $25     | Adaptive             | `high`         | Jan 2026         |
| [Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview)   | 1M      | 128K       | $2 / $10     | Adaptive             | `high`         | Jan 2026         |
| [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/haiku-4-5/overview) | 200K    | 64K        | $1 / $5      | Extended             | —              | Feb 2025         |

* **Context:** 1M tokens is roughly 555k words or 2.5M Unicode characters on the current tokenizer (introduced with Claude Opus 4.7); models before it fit about 750k words in 1M tokens. 200k tokens is roughly 150k words.
* **Max output:** Synchronous Messages API limit. On the Message Batches API, Claude Opus 5, Claude Sonnet 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, and Claude Sonnet 4.6 support up to 300k output tokens with the output-300k-2026-03-24 beta header.
* **Price / MTok:** Input / output, base price per million tokens. Batch API requests are 50% off; prompt caching reads cost 10% of the base input price (2.5% on Claude Fable 5.1 and Claude Mythos 5.1). See Pricing for the full list.
* **Thinking:** Adaptive thinking lets the model decide how much to think, steered by effort. Extended thinking is the manual budget\_tokens mode on earlier models.
* **Default effort:** The effort parameter’s default on the Claude API. Models without a value don’t support the parameter.
* **Knowledge cutoff:** Reliable knowledge cutoff: the date through which the model’s knowledge is most extensive and reliable.

## 规格

### Model IDs

| Platform                                                                                                  | Model ID                    |
| :-------------------------------------------------------------------------------------------------------- | :-------------------------- |
| Claude API                                                                                                | `claude-opus-4-7`           |
| [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)       | `anthropic.claude-opus-4-7` |
| [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)              | `claude-opus-4-7`           |
| [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) | `claude-opus-4-7`           |
| [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) | `claude-opus-4-7`           |

### Pricing

| Feature                                                                                   | Value                                                                  |
| :---------------------------------------------------------------------------------------- | :--------------------------------------------------------------------- |
| Input                                                                                     | $5 / MTok                                                              |
| Output                                                                                    | $25 / MTok                                                             |
| [5m cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $6.25 / MTok                                                           |
| [1h cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $10 / MTok                                                             |
| [Cache read](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)     | $0.50 / MTok                                                           |
| [Batch API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)    | 50% discount on input and output                                       |
| Full price list                                                                           | [Pricing](https://platform.claude.com/docs/zh-CN/about-claude/pricing) |

### Capabilities

| Feature                                                                                                                        | Value                  |
| :----------------------------------------------------------------------------------------------------------------------------- | :--------------------- |
| [Context window](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)                                     | 1M tokens              |
| Max output                                                                                                                     | 128K tokens            |
| [Max output (Batch API, beta)](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#extended-output-beta) | 300K tokens            |
| [Thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)                                                  | Adaptive               |
| [Default effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)                                              | `high`                 |
| Input → output                                                                                                                 | Text and images → text |
| Reliable knowledge cutoff                                                                                                      | Jan 2026               |
| Training data cutoff                                                                                                           | Jan 2026               |

### Availability

| Feature                                                                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                               |
| :------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Status](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations) | Active (legacy)                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Released                                                                         | April 16, 2026                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Retirement                                                                       | Not sooner than April 16, 2027                                                                                                                                                                                                                                                                                                                                                                                                      |
| Platforms                                                                        | Claude API, [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock), [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai), [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry), [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) |

## 资源

<CardGroup cols={3}>
  <Card title="迁移到 Claude Opus 5" icon="arrows-left-right" href="https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-47">
    从 Claude Opus 4.7 迁移到 Claude Opus 5 时会发生哪些变化。
  </Card>

  <Card title="Claude Opus 5" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/opus-5/overview">
    当前的 Opus 模型：概述、规格和资源。
  </Card>
</CardGroup>

## 参考

<CardGroup cols={3}>
  <Card title="系统提示" icon="text" href="https://platform.claude.com/docs/zh-CN/release-notes/system-prompts#claude-opus-4-7">
    Claude Opus 4.7 在 claude.ai 和 Claude 应用上使用的系统提示。
  </Card>

  <Card title="系统卡" icon="file" href="https://www.anthropic.com/claude-opus-4-7-system-card">
    Claude Opus 4.7 的安全评估和部署决策。
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
</CardGroup>
