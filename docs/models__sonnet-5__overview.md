---
title: Claude Sonnet 5
url: https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview
description: Claude Sonnet 5 概览：它的用途、各平台上的模型 ID、上下文窗口、输出限制、定价、可用性，以及使用它进行构建的指南和资源。
---

**Latest.** Released June 30, 2026.

The best combination of speed and intelligence

Model ID: `claude-sonnet-5`

Context window: 1M tokens · Max output: 128K tokens · Input pricing: $2 / MTok · Output pricing: $10 / MTok

[Announcement](https://www.anthropic.com/news/claude-sonnet-5) · [What’s new](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5) · [Migration guide](https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide)

## 概述

Claude Sonnet 5 是 Anthropic Sonnet 模型系列的下一代产品。它是 Claude Sonnet 4.6 的直接替换升级，包含三项行为变更：[adaptive thinking（自适应思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)默认开启；手动 extended thinking（扩展思考）现在会返回 400 错误（它在 Claude Sonnet 4.6 上已被弃用）；将采样参数（`temperature`、`top_p`、`top_k`）设置为非默认值会返回 400 错误。本页总结了发布时的所有新内容，包括新的 tokenizer（分词器）。

[Claude Sonnet 5 的新功能](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5)

## 对比情况

| Model                                                                                | Context | Max output | Price / MTok | Latency  | Thinking             | Default effort | Knowledge cutoff |
| :----------------------------------------------------------------------------------- | :------ | :--------- | :----------- | :------- | :------------------- | :------------- | :--------------- |
| [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) | 1M      | 128K       | $10 / $50    | Slower   | Adaptive (always on) | `high`         | Jun 2026         |
| [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/overview)       | 1M      | 128K       | $5 / $25     | Moderate | Adaptive             | `high`         | May 2026         |
| **Claude Sonnet 5** (this model)                                                     | 1M      | 128K       | $2 / $10     | Fast     | Adaptive             | `high`         | Jan 2026         |
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

| Platform                                                                                                  | Model ID                    |
| :-------------------------------------------------------------------------------------------------------- | :-------------------------- |
| Claude API                                                                                                | `claude-sonnet-5`           |
| [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)       | `anthropic.claude-sonnet-5` |
| [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)              | `claude-sonnet-5`           |
| [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) | `claude-sonnet-5`           |
| [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) | `claude-sonnet-5`           |

### Pricing

| Feature                                                                                   | Value                                                                  |
| :---------------------------------------------------------------------------------------- | :--------------------------------------------------------------------- |
| Input                                                                                     | $2 / MTok                                                              |
| Output                                                                                    | $10 / MTok                                                             |
| [5m cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $2.50 / MTok                                                           |
| [1h cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $4 / MTok                                                              |
| [Cache read](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)     | $0.20 / MTok                                                           |
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
| Comparative latency                                                                                                            | Fast                   |
| Input → output                                                                                                                 | Text and images → text |
| Reliable knowledge cutoff                                                                                                      | Jan 2026               |
| Training data cutoff                                                                                                           | Jan 2026               |

### Availability

| Feature                                                                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                               |
| :------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Status](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations) | Active (latest)                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Released                                                                         | June 30, 2026                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Retirement                                                                       | Not sooner than June 30, 2027                                                                                                                                                                                                                                                                                                                                                                                                       |
| Platforms                                                                        | Claude API, [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock), [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai), [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry), [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) |

## 须知事项

* 在 [Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#extended-output-beta) 上，Claude Sonnet 5 通过 `output-300k-2026-03-24` beta 请求头支持最多 300k 输出令牌。
* 将 `temperature`、`top_p` 或 `top_k` 设置为非默认值会返回 400 错误。请参阅 [Claude Sonnet 5 的新特性](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5#sampling-parameters-not-accepted)。
* 使用 [Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 以编程方式查询限制和功能。

## 资源

<CardGroup cols={3}>
  <Card title="Claude Sonnet 5 提示指南" icon="lightbulb" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-sonnet-5">
    针对特定模型的提示指导。
  </Card>

  <Card title="自适应思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    在 Claude Sonnet 5 上默认开启。使用 `effort` 控制思考深度。
  </Card>

  <Card title="Effort" icon="sliders" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    在 Claude API 和 Claude Code 上，effort 默认为 `high`。请根据工作负载选择级别。
  </Card>

  <Card title="上下文窗口" icon="stack" href="https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows">
    默认为 1M 令牌。了解上下文窗口（context window）的计算和管理方式。
  </Card>
</CardGroup>

## 参考

<CardGroup cols={3}>
  <Card title="系统卡" icon="file" href="https://www.anthropic.com/claude-sonnet-5-system-card">
    Claude Sonnet 5 的安全评估和部署决策。
  </Card>

  <Card title="定价" icon="coins" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整价格列表，包括批处理折扣和提示缓存（prompt caching）费率。
  </Card>

  <Card title="模型 ID 与版本管理" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions">
    模型 ID、别名和固定快照的工作方式。
  </Card>

  <Card title="模型弃用" icon="clock" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    每个 Claude 模型的生命周期状态和退役承诺。
  </Card>
</CardGroup>
