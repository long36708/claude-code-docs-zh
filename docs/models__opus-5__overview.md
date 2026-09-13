---
title: Claude Opus 5
url: https://platform.claude.com/docs/zh-CN/models/opus-5/overview
description: Claude Opus 5 概览：它的用途、各平台上的模型 ID、上下文窗口、输出限制、定价、可用性，以及使用它进行构建的指南和资源。
---

**Latest.** Released July 24, 2026.

For complex agentic coding and enterprise work

Model ID: `claude-opus-5`

Context window: 1M tokens · Max output: 128K tokens · Input pricing: $5 / MTok · Output pricing: $25 / MTok

[Announcement](https://www.anthropic.com/news/claude-opus-5) · [What’s new](https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5) · [Migration guide](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide)

## 概述

Claude Opus 5 相比 Claude Opus 4.8 是一次跨越式的改进，在深度推理、智能体与长周期任务以及测试时计算扩展方面提升最大。本页总结了 Claude Opus 5 的所有新特性，包括对话中途工具变更，以及针对在 Claude Opus 4.8 上运行的代码的两项破坏性变更：思考默认开启，以及仅在 effort 为 `high` 或更低时才能禁用思考。

[Claude Opus 5 的新特性](https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5)

## 对比情况

| Model                                                                                | Context | Max output | Price / MTok | Latency  | Thinking             | Default effort | Knowledge cutoff |
| :----------------------------------------------------------------------------------- | :------ | :--------- | :----------- | :------- | :------------------- | :------------- | :--------------- |
| [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) | 1M      | 128K       | $10 / $50    | Slower   | Adaptive (always on) | `high`         | Jun 2026         |
| **Claude Opus 5** (this model)                                                       | 1M      | 128K       | $5 / $25     | Moderate | Adaptive             | `high`         | May 2026         |
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

| Platform                                                                                                  | Model ID                  |
| :-------------------------------------------------------------------------------------------------------- | :------------------------ |
| Claude API                                                                                                | `claude-opus-5`           |
| [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)       | `anthropic.claude-opus-5` |
| [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)              | `claude-opus-5`           |
| [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) | `claude-opus-5`           |

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
| Comparative latency                                                                                                            | Moderate               |
| Input → output                                                                                                                 | Text and images → text |
| Reliable knowledge cutoff                                                                                                      | May 2026               |
| Training data cutoff                                                                                                           | May 2026               |

### Availability

| Feature                                                                          | Value                                                                                                                                                                                                                                                                                                                    |
| :------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Status](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations) | Active (latest)                                                                                                                                                                                                                                                                                                          |
| Released                                                                         | July 24, 2026                                                                                                                                                                                                                                                                                                            |
| Retirement                                                                       | Not sooner than July 24, 2027                                                                                                                                                                                                                                                                                            |
| Platforms                                                                        | Claude API, [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock), [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai), [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) |

## 须知事项

* 在 [Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#extended-output-beta) 上，Claude Opus 5 通过 `output-300k-2026-03-24` beta 标头支持最多 300k 输出令牌。
* 可缓存提示的最小长度为 512 个令牌。请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#cache-limitations)。
* 使用 [Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 以编程方式查询限制和功能。

## 资源

<CardGroup cols={3}>
  <Card title="Claude Opus 5 提示指南" icon="lightbulb" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5">
    针对特定模型的提示指导。
  </Card>

  <Card title="Effort" icon="sliders" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    在 Claude Opus 5 上，effort（努力程度）默认为 `high`，且比在早期模型上更为重要。请根据每种工作负载选择相应级别。
  </Card>

  <Card title="自适应思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    默认开启。禁用思考需要 effort 为 `high` 或更低。
  </Card>

  <Card title="快速模式" icon="lightning" href="https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode">
    Claude API 上延迟更低的 Claude Opus 5（研究预览版），单独定价。
  </Card>
</CardGroup>

## 参考

<CardGroup cols={3}>
  <Card title="系统提示" icon="text" href="https://platform.claude.com/docs/zh-CN/release-notes/system-prompts#claude-opus-5">
    Claude Opus 5 在 claude.ai 和 Claude 应用中使用的系统提示。
  </Card>

  <Card title="系统卡" icon="file" href="https://www.anthropic.com/claude-opus-5-system-card">
    Claude Opus 5 的安全评估和部署决策。
  </Card>

  <Card title="定价" icon="coins" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整价格列表，包括批处理折扣和提示缓存费率。
  </Card>

  <Card title="模型 ID 与版本管理" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions">
    模型 ID、别名和固定快照的工作方式。
  </Card>

  <Card title="模型弃用" icon="clock" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    每个 Claude 模型的生命周期状态和退役承诺。
  </Card>
</CardGroup>
