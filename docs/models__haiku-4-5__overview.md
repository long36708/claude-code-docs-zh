---
title: Claude Haiku 4.5
url: https://platform.claude.com/docs/zh-CN/models/haiku-4-5/overview
description: Claude Haiku 4.5 概览：它的用途、各平台上的模型 ID、上下文窗口、输出限制、定价、可用性，以及使用它进行构建的指南和资源。
---

**Latest.** Released October 15, 2025.

The fastest model with near-frontier intelligence

Model ID: `claude-haiku-4-5-20251001`

Context window: 200K tokens · Max output: 64K tokens · Input pricing: $1 / MTok · Output pricing: $5 / MTok

[Announcement](https://www.anthropic.com/news/claude-haiku-4-5) · [Migration guide](https://platform.claude.com/docs/zh-CN/models/haiku-4-5/migration-guide)

## 对比情况

| Model                                                                                | Context | Max output | Price / MTok | Latency  | Thinking             | Default effort | Knowledge cutoff |
| :----------------------------------------------------------------------------------- | :------ | :--------- | :----------- | :------- | :------------------- | :------------- | :--------------- |
| [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) | 1M      | 128K       | $10 / $50    | Slower   | Adaptive (always on) | `high`         | Jun 2026         |
| [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/overview)       | 1M      | 128K       | $5 / $25     | Moderate | Adaptive             | `high`         | May 2026         |
| [Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview)   | 1M      | 128K       | $2 / $10     | Fast     | Adaptive             | `high`         | Jan 2026         |
| **Claude Haiku 4.5** (this model)                                                    | 200K    | 64K        | $1 / $5      | Fastest  | Extended             | —              | Feb 2025         |

* **Context:** 1M tokens is roughly 555k words or 2.5M Unicode characters on the current tokenizer (introduced with Claude Opus 4.7); models before it fit about 750k words in 1M tokens. 200k tokens is roughly 150k words.
* **Max output:** Synchronous Messages API limit. On the Message Batches API, Claude Opus 5, Claude Sonnet 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, and Claude Sonnet 4.6 support up to 300k output tokens with the output-300k-2026-03-24 beta header.
* **Price / MTok:** Input / output, base price per million tokens. Batch API requests are 50% off; prompt caching reads cost 10% of the base input price (2.5% on Claude Fable 5.1 and Claude Mythos 5.1). See Pricing for the full list.
* **Latency:** Comparative latency, relative to the current lineup, as published in the models overview. Actual latency depends on prompt length, output length, and thinking effort.
* **Thinking:** Adaptive thinking lets the model decide how much to think, steered by effort. Extended thinking is the manual budget\_tokens mode on earlier models.
* **Default effort:** The effort parameter’s default on the Claude API. Models without a value don’t support the parameter.
* **Knowledge cutoff:** Reliable knowledge cutoff: the date through which the model’s knowledge is most extensive and reliable.

## 规格

### Model IDs

| Platform                                                                                                                 | Model ID                                   |
| :----------------------------------------------------------------------------------------------------------------------- | :----------------------------------------- |
| Claude API                                                                                                               | `claude-haiku-4-5-20251001`                |
| Claude API alias                                                                                                         | `claude-haiku-4-5`                         |
| [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)                      | `anthropic.claude-haiku-4-5`               |
| [Amazon Bedrock (InvokeModel)](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy) | `anthropic.claude-haiku-4-5-20251001-v1:0` |
| [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)                             | `claude-haiku-4-5@20251001`                |
| [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)                | `claude-haiku-4-5`                         |
| [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)                | `claude-haiku-4-5`                         |

### Pricing

| Feature                                                                                   | Value                                                                  |
| :---------------------------------------------------------------------------------------- | :--------------------------------------------------------------------- |
| Input                                                                                     | $1 / MTok                                                              |
| Output                                                                                    | $5 / MTok                                                              |
| [5m cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $1.25 / MTok                                                           |
| [1h cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $2 / MTok                                                              |
| [Cache read](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)     | $0.10 / MTok                                                           |
| [Batch API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)    | 50% discount on input and output                                       |
| Full price list                                                                           | [Pricing](https://platform.claude.com/docs/zh-CN/about-claude/pricing) |

### Capabilities

| Feature                                                                                    | Value                  |
| :----------------------------------------------------------------------------------------- | :--------------------- |
| [Context window](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows) | 200K tokens            |
| Max output                                                                                 | 64K tokens             |
| [Thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)              | Extended               |
| [Default effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)          | Not supported          |
| Comparative latency                                                                        | Fastest                |
| Input → output                                                                             | Text and images → text |
| Reliable knowledge cutoff                                                                  | Feb 2025               |
| Training data cutoff                                                                       | Jul 2025               |

### Availability

| Feature                                                                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| :------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [Status](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations) | Active (latest)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Released                                                                         | October 15, 2025                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Retirement                                                                       | Not sooner than October 15, 2026                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Platforms                                                                        | Claude API, [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock), [Amazon Bedrock (InvokeModel)](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy), [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai), [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry), [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) |

## 须知事项

* `claude-haiku-4-5` 是一个便捷别名，会解析为固定快照 `claude-haiku-4-5-20251001`。请参阅[模型 ID 与版本管理](https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions)。
* Claude Haiku 4.5 使用手动 "extended thinking"（扩展思考）（`thinking.type: "enabled"`），而非自适应思考。
* 可通过 [Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 以编程方式查询限制和功能。

## 资源

<CardGroup cols={3}>
  <Card title="扩展思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking">
    Claude Haiku 4.5 支持通过 `budget_tokens` 进行手动扩展思考。
  </Card>

  <Card title="选择模型" icon="scales" href="https://platform.claude.com/docs/zh-CN/about-claude/models/choosing-a-model">
    何时以效率优先从 Haiku 开始，何时选用更大的模型。
  </Card>

  <Card title="降低延迟" icon="gauge" href="https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/reduce-latency">
    与产品线中最快的模型搭配效果良好的技术。
  </Card>
</CardGroup>

## 参考

<CardGroup cols={3}>
  <Card title="系统提示" icon="text" href="https://platform.claude.com/docs/zh-CN/release-notes/system-prompts#claude-haiku-4-5">
    Claude Haiku 4.5 在 claude.ai 和 Claude 应用中使用的系统提示。
  </Card>

  <Card title="系统卡" icon="file" href="https://www.anthropic.com/claude-haiku-4-5-system-card">
    Claude Haiku 4.5 的安全评估和部署决策。
  </Card>

  <Card title="定价" icon="coins" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整价格表，包括批量折扣和提示缓存费率。
  </Card>

  <Card title="模型 ID 与版本管理" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions">
    模型 ID、别名和固定快照的工作原理。
  </Card>

  <Card title="模型弃用" icon="clock" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    每个 Claude 模型的生命周期状态和退役承诺。
  </Card>

  <Card title="Amazon Bedrock（Opus 4.6 及更早版本）" icon="cloud" href="https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy">
    Claude Haiku 4.5 也可通过 InvokeModel Bedrock 集成和 Bedrock 风格的模型 ID 使用。
  </Card>
</CardGroup>
