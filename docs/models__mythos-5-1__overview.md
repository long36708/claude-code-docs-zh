---
title: Claude Mythos 5.1
url: https://platform.claude.com/docs/zh-CN/models/mythos-5-1/overview
description: Claude Mythos 5.1 概览：与 Claude Fable 5.1 相同的模型，仅通过 Project Glasswing 以邀请方式提供。包括模型 ID、规格、定价以及如何申请访问权限。
---

**Invite only.** Released September 1, 2026.

Claude Fable 5.1 for Project Glasswing participants

Model ID: `claude-mythos-5-1`

Context window: 1M tokens · Max output: 128K tokens · Input pricing: $10 / MTok · Output pricing: $50 / MTok

[Announcement](https://www.anthropic.com/claude-fable-and-mythos-5-1) · [What’s new](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1) · [Migration guide](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#migrating-from-claude-mythos-5-to-claude-mythos-5-1)

Claude Mythos 5.1 is offered separately, by invitation only, as part of Project Glasswing. It shares Claude Fable 5.1’s specifications and pricing. For access, contact your Anthropic, AWS, or Google Cloud account team. [See Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) · [Project Glasswing](https://anthropic.com/glasswing)

## 对比情况

| Model                                                                                | Context | Max output | Price / MTok | Latency  | Thinking             | Default effort | Knowledge cutoff |
| :----------------------------------------------------------------------------------- | :------ | :--------- | :----------- | :------- | :------------------- | :------------- | :--------------- |
| [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) | 1M      | 128K       | $10 / $50    | Slower   | Adaptive (always on) | `high`         | Jun 2026         |
| **Claude Mythos 5.1** (this model)                                                   | 1M      | 128K       | $10 / $50    | Slower   | Adaptive (always on) | `high`         | Jun 2026         |
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

| Platform                                                                                                  | Model ID                      |
| :-------------------------------------------------------------------------------------------------------- | :---------------------------- |
| Claude API                                                                                                | `claude-mythos-5-1`           |
| [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)       | `anthropic.claude-mythos-5-1` |
| [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)              | `claude-mythos-5-1`           |
| [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) | `claude-mythos-5-1`           |

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

| Feature                                                                          | Value                                                                                                                                                                                                                                                                                                                    |
| :------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Status](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations) | Active (invite only)                                                                                                                                                                                                                                                                                                     |
| Released                                                                         | September 1, 2026                                                                                                                                                                                                                                                                                                        |
| Retirement                                                                       | Not sooner than September 1, 2027                                                                                                                                                                                                                                                                                        |
| Platforms                                                                        | Claude API, [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock), [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai), [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) |

## 参考资料

<CardGroup cols={3}>
  <Card title="从 Claude Mythos 5 迁移" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide#migrating-from-claude-mythos-5-to-claude-mythos-5-1">
    从 Claude Mythos 5 迁移时会发生哪些变化。
  </Card>

  <Card title="系统卡" icon="file" href="https://www.anthropic.com/claude-fable-5-1-mythos-5-1-system-card">
    Claude Fable 5.1 和 Claude Mythos 5.1 的安全评估与部署决策。
  </Card>

  <Card title="定价" icon="coins" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整价格列表，包括批量折扣和提示缓存费率。
  </Card>

  <Card title="模型 ID 与版本管理" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions">
    模型 ID、别名和固定快照的工作原理。
  </Card>

  <Card title="模型弃用" icon="clock" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    每个 Claude 模型的生命周期状态和退役承诺。
  </Card>
</CardGroup>
