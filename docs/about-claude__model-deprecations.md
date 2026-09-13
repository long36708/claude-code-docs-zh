---
title: 模型弃用
url: https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations
description: 查看哪些 Claude 模型处于活跃、已弃用或已退役状态，并查找模型和 API 参数的退役日期及推荐替代方案。
---

随着更安全、更强大的模型发布，Anthropic 会定期退役旧模型。依赖 Anthropic 模型的应用程序可能需要偶尔更新才能继续正常运行。受影响的客户始终会通过电子邮件和文档收到通知。

本页列出了所有 API 弃用项以及推荐的替代方案。

## 概述

Anthropic 使用以下术语来描述模型生命周期：

* **活跃（Active）：** 该模型受到完全支持，并推荐使用。
* **旧版（Legacy）：** 该模型将不再接收更新，未来可能会被弃用。
* **已弃用（Deprecated）：** 该模型仍可正常使用，但不再推荐。Anthropic 会提供推荐的替代方案并指定退役日期。
* **已退役（Retired）：** 该模型不再可用。对已退役模型的请求将会失败。

<Warning>
  已弃用的模型可能不如活跃模型可靠。请将工作负载迁移到活跃模型，以保持最高水平的支持和可靠性。
</Warning>

本页上的日期适用于 Anthropic 运营的平台：Claude API、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 和 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)。合作伙伴运营的平台（Amazon Bedrock 和 Google Cloud）自行设定退役时间表，因此模型的生命周期状态和日期可能有所不同。请参阅 [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock#supported-models)、[Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy#api-model-ids) 和 [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai#api-model-ids) 模型表。

## 迁移到替代方案

一旦模型被弃用，请在退役日期之前将所有使用迁移到合适的替代方案。在退役日期之后对模型的请求将会失败。

为了帮助衡量替代模型在您的任务上的性能，请考虑在退役日期之前尽早使用新模型对您的应用程序进行全面测试。

有关迁移到最新 Claude 模型的具体说明，请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。

## 通知

Anthropic 会通知对即将退役的模型有活跃部署的客户，对于公开发布的模型，会在模型退役前至少提前 60 天发出通知。

## 审核模型使用情况

为了帮助识别已弃用模型的使用情况，客户可以访问其 API 使用情况的审核记录。请按照以下步骤操作：

1. 前往 Claude Console 中的[使用情况](https://platform.claude.com/usage)页面。
2. 点击 **Export**。
3. 查看下载的 CSV 文件，了解按 API 密钥和模型细分的使用情况。

此审核将帮助您找到应用程序仍在使用已弃用模型的任何实例，使您能够在退役日期之前优先更新到较新的模型。

## 最佳实践

1. 定期查看文档以获取有关模型弃用的更新。
2. 在当前模型的退役日期之前尽早使用较新的模型测试您的应用程序。
3. 尽快更新您的代码以使用推荐的替代模型。
4. 如果您在迁移方面需要帮助或有任何疑问，请联系支持团队。

## 弃用的弊端与缓解措施

Anthropic 目前弃用和退役模型是为了确保新模型发布所需的容量。这会带来一些弊端：

* 重视特定模型的用户必须迁移到新版本
* 研究人员将无法访问用于持续性研究和比较研究的模型
* 模型退役会带来与安全和模型福祉相关的风险

Anthropic 希望在未来某个时候能够再次公开提供过去的模型。在此期间，Anthropic 已承诺长期保存模型权重并采取其他措施来帮助缓解这些影响。有关更多详细信息，请参阅[关于模型弃用和保存的承诺](https://www.anthropic.com/research/deprecation-commitments)。

## 模型状态

<Note>
  [Claude Mythos Preview](https://anthropic.com/glasswing)（`claude-mythos-preview`）已弃用。要迁移到 [Claude Mythos 5](https://anthropic.com/glasswing)（`claude-mythos-5`），请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/models/fable-5/migration-guide#migrating-from-claude-mythos-preview)。
</Note>

下表列出了当前模型和最近退役的模型及其状态：

| API 模型名称                   | 当前状态 | 弃用日期             | 暂定退役日期               |
| -------------------------- | ---- | ---------------- | -------------------- |
| claude-fable-5-1           | 活跃   | 不适用              | 不早于 2027 年 9 月 1 日   |
| claude-fable-5             | 活跃   | 不适用              | 不早于 2027 年 6 月 9 日   |
| claude-opus-5              | 活跃   | 不适用              | 不早于 2027 年 7 月 24 日  |
| claude-opus-4-8            | 活跃   | 不适用              | 不早于 2027 年 5 月 28 日  |
| claude-opus-4-7            | 活跃   | 不适用              | 不早于 2027 年 4 月 16 日  |
| claude-opus-4-6            | 活跃   | 不适用              | 不早于 2027 年 2 月 5 日   |
| claude-opus-4-5-20251101   | 活跃   | 不适用              | 不早于 2026 年 11 月 24 日 |
| claude-opus-4-1-20250805   | 已退役  | 2026 年 6 月 5 日   | 2026 年 8 月 5 日       |
| claude-opus-4-20250514     | 已退役  | 2026 年 4 月 14 日  | 2026 年 6 月 15 日      |
| claude-sonnet-5            | 活跃   | 不适用              | 不早于 2027 年 6 月 30 日  |
| claude-sonnet-4-6          | 活跃   | 不适用              | 不早于 2027 年 2 月 17 日  |
| claude-sonnet-4-5-20250929 | 活跃   | 不适用              | 不早于 2026 年 9 月 29 日  |
| claude-sonnet-4-20250514   | 已退役  | 2026 年 4 月 14 日  | 2026 年 6 月 15 日      |
| claude-3-7-sonnet-20250219 | 已退役  | 2025 年 10 月 28 日 | 2026 年 2 月 19 日      |
| claude-haiku-4-5-20251001  | 活跃   | 不适用              | 不早于 2026 年 10 月 15 日 |
| claude-3-5-haiku-20241022  | 已退役  | 2025 年 12 月 19 日 | 2026 年 2 月 19 日      |
| claude-3-haiku-20240307    | 已退役  | 2026 年 2 月 19 日  | 2026 年 4 月 20 日      |

## 弃用历史

以下各节列出了所有弃用项，最新的公告排在最前面。

### 2026-06-05：Claude Opus 4.1 模型

<Note>
  该模型已于 2026 年 8 月 5 日退役。
</Note>

2026 年 6 月 5 日，Anthropic 通知了使用 Claude Opus 4.1 的开发者，该模型即将在 Claude API 上退役。

| 退役日期           | 已弃用模型                      | 推荐替代方案            |
| -------------- | -------------------------- | ----------------- |
| 2026 年 8 月 5 日 | `claude-opus-4-1-20250805` | `claude-opus-4-8` |

### 2026-04-14：Claude Sonnet 4 和 Claude Opus 4 模型

<Note>
  这些模型已于 2026 年 6 月 15 日退役。
</Note>

2026 年 4 月 14 日，Anthropic 通知了使用 Claude Sonnet 4 和 Claude Opus 4 模型的开发者，这些模型即将在 Claude API 上退役。

| 退役日期            | 已弃用模型                      | 推荐替代方案              |
| --------------- | -------------------------- | ------------------- |
| 2026 年 6 月 15 日 | `claude-sonnet-4-20250514` | `claude-sonnet-4-6` |
| 2026 年 6 月 15 日 | `claude-opus-4-20250514`   | `claude-opus-4-8`   |

### 2026-02-19：Claude Haiku 3 模型

<Note>
  该模型已于 2026 年 4 月 20 日退役。
</Note>

2026 年 2 月 19 日，Anthropic 通知了使用 Claude Haiku 3 模型的开发者，该模型即将在 Claude API 上退役。

| 退役日期            | 已弃用模型                     | 推荐替代方案                      |
| --------------- | ------------------------- | --------------------------- |
| 2026 年 4 月 20 日 | `claude-3-haiku-20240307` | `claude-haiku-4-5-20251001` |

### 2025-12-19：Claude Haiku 3.5 模型

<Note>
  该模型已于 2026 年 2 月 19 日退役。
</Note>

2025 年 12 月 19 日，Anthropic 通知了使用 Claude Haiku 3.5 模型的开发者，该模型即将在 Claude API 上退役。

| 退役日期            | 已弃用模型                       | 推荐替代方案                      |
| --------------- | --------------------------- | --------------------------- |
| 2026 年 2 月 19 日 | `claude-3-5-haiku-20241022` | `claude-haiku-4-5-20251001` |

### 2025-10-28：Claude Sonnet 3.7 模型

<Note>
  该模型已于 2026 年 2 月 19 日退役。
</Note>

2025 年 10 月 28 日，Anthropic 通知了使用 Claude Sonnet 3.7 模型的开发者，该模型即将在 Claude API 上退役。

| 退役日期            | 已弃用模型                        | 推荐替代方案              |
| --------------- | ---------------------------- | ------------------- |
| 2026 年 2 月 19 日 | `claude-3-7-sonnet-20250219` | `claude-sonnet-4-6` |

### 2025-08-13：Claude Sonnet 3.5 模型

<Note>
  这些模型已于 2025 年 10 月 28 日退役。
</Note>

2025 年 8 月 13 日，Anthropic 通知了使用 Claude Sonnet 3.5 模型的开发者，这些模型即将退役。

| 退役日期             | 已弃用模型                        | 推荐替代方案              |
| ---------------- | ---------------------------- | ------------------- |
| 2025 年 10 月 28 日 | `claude-3-5-sonnet-20240620` | `claude-sonnet-4-6` |
| 2025 年 10 月 28 日 | `claude-3-5-sonnet-20241022` | `claude-sonnet-4-6` |

### 2025-06-30：Claude Opus 3 模型

<Note>
  该模型已于 2026 年 1 月 5 日退役。
</Note>

2025 年 6 月 30 日，Anthropic 通知了使用 Claude Opus 3 模型的开发者，该模型即将退役。

| 退役日期           | 已弃用模型                    | 推荐替代方案            |
| -------------- | ------------------------ | ----------------- |
| 2026 年 1 月 5 日 | `claude-3-opus-20240229` | `claude-opus-4-8` |

### 2025-01-21：Claude 2、Claude 2.1 和 Claude Sonnet 3 模型

<Note>
  这些模型已于 2025 年 7 月 21 日退役。
</Note>

2025 年 1 月 21 日，Anthropic 通知了使用 Claude 2、Claude 2.1 和 Claude Sonnet 3 模型的开发者，这些模型即将退役。

| 退役日期            | 已弃用模型                      | 推荐替代方案              |
| --------------- | -------------------------- | ------------------- |
| 2025 年 7 月 21 日 | `claude-2.0`               | `claude-opus-4-8`   |
| 2025 年 7 月 21 日 | `claude-2.1`               | `claude-opus-4-8`   |
| 2025 年 7 月 21 日 | `claude-3-sonnet-20240229` | `claude-sonnet-4-6` |

### 2024-09-04：Claude 1 和 Instant 模型

<Note>
  这些模型已于 2024 年 11 月 6 日退役。
</Note>

2024 年 9 月 4 日，Anthropic 通知了使用 Claude 1 和 Instant 模型的开发者，这些模型即将退役。

| 退役日期            | 已弃用模型                | 推荐替代方案                      |
| --------------- | -------------------- | --------------------------- |
| 2024 年 11 月 6 日 | `claude-1.0`         | `claude-haiku-4-5-20251001` |
| 2024 年 11 月 6 日 | `claude-1.1`         | `claude-haiku-4-5-20251001` |
| 2024 年 11 月 6 日 | `claude-1.2`         | `claude-haiku-4-5-20251001` |
| 2024 年 11 月 6 日 | `claude-1.3`         | `claude-haiku-4-5-20251001` |
| 2024 年 11 月 6 日 | `claude-instant-1.0` | `claude-haiku-4-5-20251001` |
| 2024 年 11 月 6 日 | `claude-instant-1.1` | `claude-haiku-4-5-20251001` |
| 2024 年 11 月 6 日 | `claude-instant-1.2` | `claude-haiku-4-5-20251001` |

## API 参数弃用

Anthropic 偶尔会弃用不再适用于当前模型的请求参数。API 如何处理已弃用的参数取决于模型，如下表所示。大多数 SDK 会在其请求类型中保留已弃用的参数，以便现有代码能够继续通过类型检查。Python SDK（v1.0 及更高版本）移除了 `temperature`、`top_p` 和 `top_k`，因此传递这些参数会引发 `TypeError`。

| 参数                            | 状态                         | 行为                                                                                                    | 推荐替代方案                                                                                                                              |
| ----------------------------- | -------------------------- | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `temperature`、`top_p`、`top_k` | 已弃用（Claude Opus 4.7 及更高版本） | 在 Claude 4.7 及更高版本的模型以及 [Claude Mythos Preview](https://anthropic.com/glasswing) 上设置为非默认值时，返回 400 错误。 | 省略这些参数，并使用[提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)来引导模型行为。 |

有关迁移步骤，请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。
