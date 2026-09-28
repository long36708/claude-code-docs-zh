---
title: Claude Opus 5.5
url: https://platform.claude.com/docs/zh-CN/models/opus-5-5/overview
description: Claude Opus 5.5 概览：适用场景、各平台上的模型 ID、上下文窗口、输出限制、定价、可用性，以及使用它进行构建的指南和资源。
---

**Latest.** Released September 22, 2026.

For long-running agentic coding and knowledge work

Model ID: `claude-opus-5-5`

Context window: 1M tokens · Max output: 128K tokens · Input pricing: $4 / MTok · Output pricing: $20 / MTok

[Announcement](https://www.anthropic.com/claude-opus-5-5) · [What’s new](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5) · [Migration guide](https://platform.claude.com/docs/zh-CN/models/opus-5-5/migration-guide)

## 概述

Claude Opus 5.5 专为长时间运行的智能体编码和知识工作而构建，定价为每百万输入/输出令牌 $4 / $20 美元。有四项"breaking changes"（破坏性变更）会影响已在 Claude Opus 5 上运行的代码：[无法禁用思考](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#thinking-cant-be-disabled)、[强制工具使用会返回错误](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#forced-tool-use-is-not-supported)、[思考块与模型和对话绑定](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#thinking-blocks-are-tied-to-the-model-that-produced-them)，以及在 Claude API 和 Google Cloud 上[不再接受早期的 `computer_20251124` 计算机使用工具](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#computer-20251124-is-not-supported)。前三项同样适用于 Claude Fable 5.1。另有一项变更会改变响应结构，但不会导致任何请求失败：[工具调用之间的文本以 `thinking` 块返回](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5#text-between-tool-calls)，在默认的 `display` 设置下，这些块中的文本为空。如果应用程序将这些文本作为进度更新流式传输给用户，那么在它设置一个会返回文本的 `display` 值之前，它在工具调用之间将保持静默。

[Claude Opus 5.5 的新功能](https://platform.claude.com/docs/zh-CN/models/opus-5-5/whats-new-opus-5-5)

## 对比情况

| Model                                                                                | Context | Max output | Price / MTok | Latency  | Thinking             | Default effort | Knowledge cutoff |
| :----------------------------------------------------------------------------------- | :------ | :--------- | :----------- | :------- | :------------------- | :------------- | :--------------- |
| [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) | 1M      | 128K       | $10 / $50    | Slower   | Adaptive (always on) | `high`         | Jun 2026         |
| **Claude Opus 5.5** (this model)                                                     | 1M      | 128K       | $4 / $20     | Moderate | Adaptive (always on) | `medium`       | Jun 2026         |
| [Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview)   | 1M      | 128K       | $2 / $10     | Fast     | Adaptive             | `high`         | Jan 2026         |
| [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/haiku-4-5/overview) | 200K    | 64K        | $1 / $5      | Fastest  | Extended             | —              | Feb 2025         |

* **Context:** 1M tokens is roughly 555k words or 2.5M Unicode characters on the current tokenizer (introduced with Claude Opus 4.7); models before it fit about 750k words in 1M tokens. 200k tokens is roughly 150k words.
* **Max output:** Synchronous Messages API limit. On the Message Batches API, Claude Opus 5.5, Claude Opus 5, Claude Sonnet 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, and Claude Sonnet 4.6 support up to 300k output tokens with the output-300k-2026-03-24 beta header.
* **Price / MTok:** Input / output, base price per million tokens. Batch API requests are 50% off; prompt caching reads cost 10% of the base input price (2.5% on Claude Fable 5.1 and Claude Mythos 5.1, 5% on Claude Opus 5.5). See Pricing for the full list.
* **Latency:** Comparative latency, relative to the current lineup, as published in the models overview. Actual latency depends on prompt length, output length, and thinking effort.
* **Thinking:** Adaptive thinking lets the model decide how much to think, steered by effort. Extended thinking is the manual budget\_tokens mode on earlier models.
* **Default effort:** The effort parameter’s default on the Claude API. Models without a value don’t support the parameter.
* **Knowledge cutoff:** Reliable knowledge cutoff: the date through which the model’s knowledge is most extensive and reliable.

## 规格

### Model IDs

| Platform                                                                                                  | Model ID                    |
| :-------------------------------------------------------------------------------------------------------- | :-------------------------- |
| Claude API                                                                                                | `claude-opus-5-5`           |
| [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)       | `anthropic.claude-opus-5-5` |
| [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)              | `claude-opus-5-5`           |
| [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) | `claude-opus-5-5`           |
| [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) | `claude-opus-5-5`           |

### Pricing

| Feature                                                                                   | Value                            |
| :---------------------------------------------------------------------------------------- | :------------------------------- |
| Input                                                                                     | $4 / MTok                        |
| Output                                                                                    | $20 / MTok                       |
| [5m cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $5 / MTok                        |
| [1h cache write](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching) | $8 / MTok                        |
| [Cache read](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)     | $0.20 / MTok                     |
| [Batch API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)    | 50% discount on input and output |

[Full price list](https://platform.claude.com/docs/zh-CN/about-claude/pricing)

### Capabilities

| Feature                                                                                                                        | Value                  |
| :----------------------------------------------------------------------------------------------------------------------------- | :--------------------- |
| [Context window](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)                                     | 1M tokens              |
| Max output                                                                                                                     | 128K tokens            |
| [Max output (Batch API, beta)](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#extended-output-beta) | 300K tokens            |
| [Thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)                                                  | Adaptive (always on)   |
| [Default effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)                                              | `medium`               |
| Comparative latency                                                                                                            | Moderate               |
| Input → output                                                                                                                 | Text and images → text |
| Reliable knowledge cutoff                                                                                                      | Jun 2026               |
| Training data cutoff                                                                                                           | Jun 2026               |

### Availability

| Feature                                                                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                               |
| :------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Status](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations) | Active (latest)                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Released                                                                         | September 22, 2026                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Retirement                                                                       | Not sooner than September 22, 2027                                                                                                                                                                                                                                                                                                                                                                                                  |
| Platforms                                                                        | Claude API, [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock), [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai), [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry), [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) |

## 须知事项

* "Adaptive thinking"（自适应思考）始终开启，且无法关闭。您可以使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)控制思考深度。
* 在 [Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#extended-output-beta) 上，使用 `output-300k-2026-03-24` beta 标头时，Claude Opus 5.5 最多支持 300k 个输出令牌。
* 最小可缓存提示长度为 512 个令牌。请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#cache-limitations)。
* 使用 [Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 以编程方式查询限制和功能。

## 资源

<CardGroup cols={3}>
  <Card title="为 Claude Opus 5.5 编写提示" icon="lightbulb" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5-5">
    Claude Opus 5.5 特有的行为差异和提示模式。
  </Card>

  <Card title="Effort" icon="sliders" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    用于控制思考深度、延迟和成本的设置。请为每种工作负载选择合适的级别。
  </Card>

  <Card title="自适应思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    自适应思考的工作原理以及思考块的保留方式。
  </Card>

  <Card title="快速模式" icon="lightning" href="https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode">
    Claude API 上更低延迟的 Claude Opus 5.5（研究预览版），单独定价。
  </Card>
</CardGroup>

## 参考

<CardGroup cols={3}>
  <Card title="系统提示" icon="text" href="https://platform.claude.com/docs/zh-CN/release-notes/system-prompts/claude-opus-5-5">
    Claude Opus 5.5 在 claude.ai 和 Claude 应用中使用的系统提示。
  </Card>

  <Card title="系统卡" icon="file" href="https://www.anthropic.com/claude-opus-5-5-system-card">
    Claude Opus 5.5 的安全评估和部署决策。
  </Card>

  <Card title="定价" icon="coins" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整价目表，包括批处理折扣和提示缓存费率。
  </Card>

  <Card title="模型 ID 与版本管理" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions">
    模型 ID、别名和固定快照的工作方式。
  </Card>

  <Card title="模型弃用" icon="clock" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    每个 Claude 模型的生命周期状态和退役承诺。
  </Card>
</CardGroup>
