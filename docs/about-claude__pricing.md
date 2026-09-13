---
title: 定价
url: https://platform.claude.com/docs/zh-CN/about-claude/pricing
description: 了解 Anthropic 针对模型和功能的定价结构
---

本页面提供 Anthropic 模型和功能的详细定价信息。所有价格均以美元计价。

如需获取最新的定价信息，请访问 [claude.com/pricing](https://claude.com/pricing)。

## 模型定价

下表显示了所有 Claude 模型的定价：

| 模型                                                                                                                        | 基础输入令牌       | 5 分钟缓存写入      | 1 小时缓存写入     | 缓存命中与刷新       | 输出令牌       |
| ------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------- | ------------ | ------------- | ---------- |
| Claude Fable 5.1                                                                                                          | $10 / MTok   | $12.50 / MTok | $20 / MTok   | $0.25 / MTok1 | $50 / MTok |
| Claude Mythos 5.1（[限量提供](https://anthropic.com/glasswing)）                                                                | $10 / MTok   | $12.50 / MTok | $20 / MTok   | $0.25 / MTok1 | $50 / MTok |
| Claude Fable 5                                                                                                            | $10 / MTok   | $12.50 / MTok | $20 / MTok   | $1 / MTok     | $50 / MTok |
| Claude Mythos 5（[限量提供](https://anthropic.com/glasswing)）                                                                  | $10 / MTok   | $12.50 / MTok | $20 / MTok   | $1 / MTok     | $50 / MTok |
| Claude Opus 5                                                                                                             | $5 / MTok    | $6.25 / MTok  | $10 / MTok   | $0.50 / MTok  | $25 / MTok |
| Claude Opus 4.8                                                                                                           | $5 / MTok    | $6.25 / MTok  | $10 / MTok   | $0.50 / MTok  | $25 / MTok |
| Claude Opus 4.7                                                                                                           | $5 / MTok    | $6.25 / MTok  | $10 / MTok   | $0.50 / MTok  | $25 / MTok |
| Claude Opus 4.6                                                                                                           | $5 / MTok    | $6.25 / MTok  | $10 / MTok   | $0.50 / MTok  | $25 / MTok |
| Claude Opus 4.5                                                                                                           | $5 / MTok    | $6.25 / MTok  | $10 / MTok   | $0.50 / MTok  | $25 / MTok |
| Claude Opus 4.1（[已停用，Bedrock 和 Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)）  | $15 / MTok   | $18.75 / MTok | $30 / MTok   | $1.50 / MTok  | $75 / MTok |
| Claude Opus 4（[已停用，Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)）              | $15 / MTok   | $18.75 / MTok | $30 / MTok   | $1.50 / MTok  | $75 / MTok |
| Claude Sonnet 5                                                                                                           | $2 / MTok    | $2.50 / MTok  | $4 / MTok    | $0.20 / MTok  | $10 / MTok |
| Claude Sonnet 4.6                                                                                                         | $3 / MTok    | $3.75 / MTok  | $6 / MTok    | $0.30 / MTok  | $15 / MTok |
| Claude Sonnet 4.5                                                                                                         | $3 / MTok    | $3.75 / MTok  | $6 / MTok    | $0.30 / MTok  | $15 / MTok |
| Claude Sonnet 4（[已停用，Bedrock 和 Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)）  | $3 / MTok    | $3.75 / MTok  | $6 / MTok    | $0.30 / MTok  | $15 / MTok |
| Claude Haiku 4.5                                                                                                          | $1 / MTok    | $1.25 / MTok  | $2 / MTok    | $0.10 / MTok  | $5 / MTok  |
| Claude Haiku 3.5（[已停用，Bedrock 和 Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)） | $0.80 / MTok | $1 / MTok     | $1.60 / MTok | $0.08 / MTok  | $4 / MTok  |

*1 Claude Fable 5.1 和 Claude Mythos 5.1 的缓存命中与刷新按基础输入价格的 0.025 倍计费。所有其他模型使用标准的 0.1 倍乘数。*

<Note id="claude-sonnet-5-introductory-pricing">
  Claude Sonnet 5 每百万输入/输出令牌 $2/$10 的定价在发布时作为截至 2026 年 8 月 31 日的推介定价公布，现已成为标准价格。原定于 2026 年 9 月 1 日上调至每百万输入/输出令牌 $3/$15 的计划将不会实施。
</Note>

<Note>
  MTok = 百万令牌。"Base Input Tokens"（基础输入令牌）列显示标准输入定价，"5m Cache Writes"（5 分钟缓存写入）、"1h Cache Writes"（1 小时缓存写入）和 "Cache Hits & Refreshes"（缓存命中与刷新）列专用于 [prompt caching（提示缓存）](https://platform.claude.com/docs/zh-CN/about-claude/pricing#prompt-caching)，"Output Tokens"（输出令牌）列显示输出定价。有关缓存列和定价乘数的说明，请参阅[提示缓存定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#prompt-caching)。
</Note>

<Note>
  Claude 4.7 及更高版本的模型以及 Claude Mythos Preview 使用更新的 tokenizer（分词器），这有助于它们在各类任务上获得更好的性能。对于相同的文本，该分词器产生的令牌数量大约多 30%。具体增幅取决于内容和工作负载形态。Claude Sonnet 4.6 及更早的模型使用之前的分词器。
</Note>

有关 Claude Platform on AWS 的定价，请参阅 [Claude Platform on AWS 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-platform-on-aws-pricing)。

## 云平台定价

本节涵盖由合作伙伴运营的云平台，在这些平台上由云提供商向您开具账单。有关通过市场计费、由 Anthropic 运营的云平台，请参阅 [Claude Platform on AWS 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-platform-on-aws-pricing)和 [Claude in Microsoft Foundry 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-in-microsoft-foundry-pricing)。

Claude 模型可在 [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 和 [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 上使用。如需官方定价，请访问：

* [Amazon Bedrock 定价](https://aws.amazon.com/bedrock/pricing/)
* [Google Cloud 定价](https://cloud.google.com/vertex-ai/generative-ai/pricing#claude-models)

<Note>
  **Claude 4.5 及后续模型的区域和多区域端点定价**

  从 Claude Sonnet 4.5、Haiku 4.5 和 Opus 4.5 开始：

  * **Bedrock** 提供两种端点类型：全球端点（动态路由以实现最大可用性）和区域端点（保证数据通过特定地理区域路由）。
  * **Google Cloud** 提供三种端点类型：全球端点、多区域端点（在某一地理区域内动态路由）和区域端点。

  区域和多区域端点相比全球端点包含 10% 的溢价。Claude API（第一方）默认为全球路由；有关第一方数据驻留选项和定价，请参阅[数据驻留定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#data-residency-pricing)。

  **适用范围：** 此定价结构适用于 Claude Sonnet 4.5、Haiku 4.5、Opus 4.5 以及所有未来的模型。更早的模型（Claude Opus 4.1 及之前的版本）保留其现有定价。

  有关实现细节和代码示例：

  * 适用于 Opus 4.7、Haiku 4.5 及更高版本模型的 [Amazon Bedrock 全球端点与区域端点](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock#regions)，或适用于 Bedrock 上所有其他模型的[旧版集成](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy#global-vs-regional-endpoints)
  * [Google Cloud 全球、多区域和区域端点](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai#global-multi-region-and-regional-endpoints)
</Note>

## Claude Platform on AWS 定价

[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 通过 AWS Marketplace 使用 Claude Consumption Units（Claude 消费单位），即 CCU 进行计费。Anthropic 按标准的每模型、每功能费率以美元对您的令牌用量进行计价，应用任何协商折扣，按每 CCU $0.01 将结果转换为 CCU，并每小时向 AWS Marketplace 报告 CCU 数量。您的 AWS 账单显示单一的 CCU 明细项。

| 概念         | 详情                                                                                                                                            |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **计费单位**   | Claude Consumption Unit（CCU）                                                                                                                  |
| **CCU 价格** | 每 CCU $0.01（固定；折扣在令牌到 CCU 的转换环节应用，而非应用于 CCU 价格）                                                                                               |
| **转换**     | 令牌用量按标准的每模型、每功能费率以美元计价（与 [Claude API 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#model-pricing)相同），然后按每 CCU $0.01 转换为 CCU |
| **计费周期**   | 每小时向 AWS Marketplace 计量；按月开具发票                                                                                                                |
| **付款模式**   | 仅限后付（后付费）；无预付额度                                                                                                                               |
| **折扣**     | 以计量更少的 CCU 的形式应用                                                                                                                              |
| **税费**     | 税前计量；由 AWS Marketplace 处理税费                                                                                                                   |
| **成本可见性**  | 在 Claude Console 中实时查看明细（通过 AWS Console 访问）；AWS Cost Explorer 显示汇总的 CCU                                                                       |

<Note>
  **Claude Consumption Units。** 如果客户通过某些市场平台（例如 Claude Platform on AWS）访问服务，用量将以 Claude Consumption Units（"CCU"）而非按 MTok 开具发票。CCU 是仅用于市场平台开票的计量单位。一百（100）CCU 代表按 [claude.com/pricing#api](https://claude.com/pricing#api) 上的适用价格计算、并在应用任何折扣后应付的 $1.00 美元服务费用。
</Note>

### 推理地理位置

对于 Claude 4.6 及更高版本的模型，使用 `inference_geo: "us"` 将应用 1.1 倍的定价乘数。`inference_geo: "global"`（默认）使用标准定价。详情请参阅[数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)。

### 私有报价

当您在 AWS Console 的 **Claude Platform on AWS** 服务页面注册时，AWS Console 会查找与您账户关联的任何私有报价，并提示您在 AWS Marketplace 中接受该报价。有关私有报价条款，请联系您的 Anthropic 客户代表。

<Note>
  如果您已有 Amazon Bedrock 私有报价，请在开始使用 Claude Platform on AWS 之前联系您的 Anthropic 或 AWS 客户代表，以确保您的折扣得到正确应用。折扣无法追溯应用于接受私有报价之前产生的用量。
</Note>

## Claude in Microsoft Foundry 定价

[Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 通过 Azure Marketplace 使用 Claude Consumption Units（CCU）进行计费。Anthropic 按标准的每模型、每功能费率以美元对您的令牌用量进行计价，应用任何协商折扣，按每 CCU $0.01 将结果转换为 CCU，并每小时向 Azure Marketplace 报告 CCU 数量。您的 Azure 账单显示单一的 CCU 明细项。

| 概念         | 详情                                                                                                                                            |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **计费单位**   | Claude Consumption Unit（CCU）                                                                                                                  |
| **CCU 价格** | 每 CCU $0.01（固定；折扣在令牌到 CCU 的转换环节应用，而非应用于 CCU 价格）                                                                                               |
| **转换**     | 令牌用量按标准的每模型、每功能费率以美元计价（与 [Claude API 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#model-pricing)相同），然后按每 CCU $0.01 转换为 CCU |
| **计费周期**   | 每小时向 Azure Marketplace 计量；按月开具发票                                                                                                              |
| **付款模式**   | 仅限后付（后付费）；无预付额度                                                                                                                               |
| **折扣**     | 以计量更少的 CCU 的形式应用                                                                                                                              |
| **税费**     | 税前计量；由 Azure Marketplace 处理税费                                                                                                                 |
| **成本可见性**  | Azure Cost Management 显示汇总的 CCU                                                                                                               |

<Note>
  **Claude Consumption Units。** 如果客户通过某些市场平台（例如 Claude Platform on AWS、Claude in Microsoft Foundry）访问服务，用量将以 Claude Consumption Units（"CCU"）而非按 MTok 开具发票。CCU 是仅用于市场平台开票的计量单位。一百（100）CCU 代表按 [claude.com/pricing#api](https://claude.com/pricing#api) 上的适用价格计算、并在应用任何折扣后应付的 $1.00 美元服务费用。
</Note>

### 推理地理位置

托管在 Azure 上的部署可以使用 US Data Zone Standard 部署类型，该类型将推理保持在美国境内。这等同于 Claude API 上的 `inference_geo: "us"`，并应用相同的 1.1 倍定价乘数。详情请参阅[数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)。

## 特定功能定价

### 提示缓存

提示缓存通过在多次 API 调用之间重用提示中先前已处理的部分来降低成本和延迟。API 无需在每次请求时重新处理相同的大型系统提示、文档或对话历史，而是以标准输入价格的一小部分从缓存中读取。

启用提示缓存有两种方式：

* **自动缓存：** 在请求的顶层添加单个 `cache_control` 字段。系统会随着对话的增长自动管理缓存断点。这是大多数用例的推荐起点。
* **显式缓存断点：** 将 `cache_control` 直接放置在各个内容块上，以便精细控制具体缓存哪些内容。

提示缓存使用以下相对于基础输入令牌费率的定价乘数：

| 缓存操作     | 乘数                                                               | 持续时间         |
| -------- | ---------------------------------------------------------------- | ------------ |
| 5 分钟缓存写入 | 基础输入价格的 1.25 倍                                                   | 缓存有效期 5 分钟   |
| 1 小时缓存写入 | 基础输入价格的 2 倍                                                      | 缓存有效期 1 小时   |
| 缓存读取（命中） | 基础输入价格的 0.1 倍（在 Claude Fable 5.1 和 Claude Mythos 5.1 上为 0.025 倍） | 与之前的写入持续时间相同 |

缓存写入令牌在内容首次存储时收费。缓存读取令牌在后续请求检索缓存内容时收费。一次缓存命中的费用为标准输入价格的 10%，这意味着对于 5 分钟持续时间（1.25 倍写入），缓存在一次缓存读取后即可回本；对于 1 小时持续时间（2 倍写入），则在两次缓存读取后回本。在 Claude Fable 5.1 和 Claude Mythos 5.1 上，一次缓存命中的费用为标准输入价格的 2.5%（每百万令牌 $0.25 美元）。

这些乘数可与其他定价修正项叠加，包括 Batch API 折扣和数据驻留。

有关实现细节、支持的模型和代码示例，请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)。

### 数据驻留定价

对于 Claude 4.6 及更高版本的模型，通过 `inference_geo` 参数指定仅限美国的推理会对所有令牌定价类别产生 1.1 倍的乘数，包括输入令牌、输出令牌、缓存写入和缓存读取。全球路由（默认）使用标准定价。

这适用于 Claude API（第一方）和 Claude Platform on AWS。在 Claude in Microsoft Foundry 上，相同的 1.1 倍乘数适用于使用 US Data Zone Standard 部署类型的部署（请参阅[推理地理位置](https://platform.claude.com/docs/zh-CN/about-claude/pricing#foundry-inference-geography)）。由合作伙伴运营的平台（Bedrock 和 Google Cloud）有独立的区域定价。详情请参阅 [Bedrock](https://aws.amazon.com/bedrock/pricing/) 和 [Google Cloud](https://cloud.google.com/vertex-ai/generative-ai/pricing#claude-models)。更早的模型不支持 `inference_geo` 参数，始终使用标准定价；在这些模型上包含该参数的请求将返回 400 错误。

如需更多信息，请参阅[数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)。

### 快速模式定价

[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)目前处于研究预览阶段，以溢价为 Claude Opus 5 和 Claude Opus 4.8 提供显著更快的输出。快速模式定价适用于整个上下文窗口，包括超过 200k 输入令牌的请求。快速模式仅在 Claude API（第一方）上可用；在 Claude Platform on AWS 或由合作伙伴运营的云平台上不可用。

| 模型                              | 输入         | 输出         |
| ------------------------------- | ---------- | ---------- |
| Claude Opus 5 / Claude Opus 4.8 | $10 / MTok | $50 / MTok |

快速模式在 Claude Opus 4.7（带有 `speed: "fast"` 的请求会返回错误）或 Claude Opus 4.6（请求以标准速度运行并按标准费率计费）上不可用。请参阅[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#supported-models)。

快速模式定价可与其他定价修正项叠加：

* [提示缓存乘数](https://platform.claude.com/docs/zh-CN/about-claude/pricing#prompt-caching)在快速模式定价之上应用
* [数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)乘数在快速模式定价之上应用

快速模式不可与 [Batch API](https://platform.claude.com/docs/zh-CN/about-claude/pricing#batch-processing) 一起使用。

如需更多信息，请参阅[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)。

### 批处理

Batch API 允许异步处理大量请求，输入和输出令牌均享受 50% 的折扣。

| 模型                                                                                                                        | 批量输入         | 批量输出          |
| ------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------- |
| Claude Fable 5.1                                                                                                          | $5 / MTok    | $25 / MTok    |
| Claude Mythos 5.1（[限量提供](https://anthropic.com/glasswing)）                                                                | $5 / MTok    | $25 / MTok    |
| Claude Fable 5                                                                                                            | $5 / MTok    | $25 / MTok    |
| Claude Mythos 5（[限量提供](https://anthropic.com/glasswing)）                                                                  | $5 / MTok    | $25 / MTok    |
| Claude Opus 5                                                                                                             | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.8                                                                                                           | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.7                                                                                                           | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.6                                                                                                           | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.5                                                                                                           | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.1（[已停用，Bedrock 和 Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)）  | $7.50 / MTok | $37.50 / MTok |
| Claude Opus 4（[已停用，Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)）              | $7.50 / MTok | $37.50 / MTok |
| Claude Sonnet 5                                                                                                           | $1 / MTok    | $5 / MTok     |
| Claude Sonnet 4.6                                                                                                         | $1.50 / MTok | $7.50 / MTok  |
| Claude Sonnet 4.5                                                                                                         | $1.50 / MTok | $7.50 / MTok  |
| Claude Sonnet 4（[已停用，Bedrock 和 Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)）  | $1.50 / MTok | $7.50 / MTok  |
| Claude Haiku 4.5                                                                                                          | $0.50 / MTok | $2.50 / MTok  |
| Claude Haiku 3.5（[已停用，Bedrock 和 Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)） | $0.40 / MTok | $2 / MTok     |

有关批处理的更多信息，请参阅[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)。

### 长上下文定价

Claude 4.6 及更高版本的模型以及 [Claude Mythos Preview](https://anthropic.com/glasswing) 以标准定价包含完整的 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)。（一个 900k 令牌的请求与一个 9k 令牌的请求按相同的每令牌费率计费。）提示缓存和批处理折扣在整个上下文窗口内按标准费率适用。

### 工具使用定价

工具使用请求的定价基于：

1. 发送给模型的输入令牌总数（包括 `tools` 参数中的令牌）
2. 生成的输出令牌数量
3. 对于服务器端工具，还有额外的基于用量的定价（例如，网页搜索按每次执行的搜索收费）

客户端工具的定价与任何其他 Claude API 请求相同，但服务器端工具可能会根据其具体用量产生额外费用。

工具使用产生的额外令牌来自：

* API 请求中的 `tools` 参数（工具名称、描述和模式）
* API 请求和响应中的 `tool_use` 内容块
* API 请求中的 `tool_result` 内容块

当您使用 `tools` 时，API 还会自动为模型包含一个启用工具使用的特殊系统提示。下表列出了每个模型所需的工具使用令牌数量（不包括前面列出的额外令牌）。请注意，该表假设至少提供了 1 个工具。如果未提供 `tools`，则工具选择为 `none` 时使用 0 个额外的系统提示令牌。

| 模型                                                                                                                        | 工具选择                           | 工具使用系统提示令牌数       |
| ------------------------------------------------------------------------------------------------------------------------- | ------------------------------ | ----------------- |
| Claude Opus 5                                                                                                             | `auto`, `none`***`any`, `tool` | 286 个令牌***406 个令牌 |
| Claude Opus 4.8                                                                                                           | `auto`, `none`***`any`, `tool` | 290 个令牌***410 个令牌 |
| Claude Opus 4.7                                                                                                           | `auto`, `none`***`any`, `tool` | 675 个令牌***804 个令牌 |
| Claude Opus 4.6                                                                                                           | `auto`, `none`***`any`, `tool` | 497 个令牌***589 个令牌 |
| Claude Opus 4.5                                                                                                           | `auto`, `none`***`any`, `tool` | 496 个令牌***588 个令牌 |
| Claude Opus 4.1（[已退役，Bedrock 和 Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)）  | `auto`, `none`***`any`, `tool` | 313 个令牌***315 个令牌 |
| Claude Opus 4（[已退役，Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)）              | `auto`, `none`***`any`, `tool` | 313 个令牌***315 个令牌 |
| Claude Sonnet 5                                                                                                           | `auto`, `none`***`any`, `tool` | 354 个令牌***474 个令牌 |
| Claude Sonnet 4.6                                                                                                         | `auto`, `none`***`any`, `tool` | 497 个令牌***589 个令牌 |
| Claude Sonnet 4.5                                                                                                         | `auto`, `none`***`any`, `tool` | 496 个令牌***588 个令牌 |
| Claude Sonnet 4（[已退役，Bedrock 和 Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)）  | `auto`, `none`***`any`, `tool` | 313 个令牌***315 个令牌 |
| Claude Haiku 4.5                                                                                                          | `auto`, `none`***`any`, `tool` | 496 个令牌***588 个令牌 |
| Claude Haiku 3.5（[已退役，Bedrock 和 Google Cloud 除外](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)） | `auto`, `none`***`any`, `tool` | 264 个令牌***355 个令牌 |

这些令牌数会加到您正常的输入和输出令牌中，以计算请求的总费用。

有关当前各模型的价格，请参阅[模型定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#model-pricing)部分。

有关工具使用实现和最佳实践的更多信息，请参阅[工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。

### 特定工具定价

#### Bash 工具

bash 工具定义会向您的请求中添加以下 input tokens（输入令牌）。这是在每个模型的[工具使用系统提示](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview#pricing)之外额外添加的，只要存在任何工具，该系统提示就会生效。

| 模型                                              | 额外输入令牌  |
| ----------------------------------------------- | ------- |
| Claude Opus 5、Claude Opus 4.8 和 Claude Opus 4.7 | 325 个令牌 |
| Claude Opus 4.6、Claude Sonnet 4.6 及更早版本         | 244 个令牌 |

以下内容会消耗额外的令牌：

* 命令输出（stdout/stderr）
* 错误消息
* 大型文件内容

完整定价详情请参阅[工具使用定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#tool-use-pricing)。

#### 代码执行工具

**与 web search（网页搜索）或 web fetch（网页抓取）一起使用时，代码执行是免费的。** 当您的 API 请求中包含 `web_search_20260209`（或更高版本）或 `web_fetch_20260209`（或更高版本）时，除标准的输入和输出令牌费用外，代码执行工具调用不会产生额外费用。

在不与这些工具一起使用时，代码执行按执行时间计费，并与令牌用量分开跟踪：

* 执行时间最少按 5 分钟计算
* 每个组织每月可获得 **1,550 小时的免费**用量
* 超出 1,550 小时的额外用量按**每个容器每小时 0.05 美元**计费
* 如果请求中包含文件，即使未调用该工具，也会按执行时间计费，因为文件会被预加载到容器中

代码执行用量会在响应中进行跟踪：

```json
{
  "usage": {
    "input_tokens": 105,
    "output_tokens": 239,
    "server_tool_use": {
      "code_execution_requests": 1
    }
  }
}
```

#### 文本编辑器工具

文本编辑器工具采用与 Claude 配合使用的其他工具相同的定价结构。它遵循基于您所使用的 Claude 模型的标准输入和输出令牌（token）定价。

除基础令牌外，文本编辑器工具还需要以下额外的输入令牌：

| 工具                                 | 额外输入令牌  |
| ---------------------------------- | ------- |
| `text_editor_20250429`（Claude 4.x） | 700 个令牌 |

完整定价详情请参阅[工具使用定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#tool-use-pricing)。

#### 网页搜索工具

除令牌用量外，网页搜索的使用会额外计费：

```json
{
  "usage": {
    "input_tokens": 105,
    "output_tokens": 6039,
    "cache_read_input_tokens": 7123,
    "cache_creation_input_tokens": 7345,
    "server_tool_use": {
      "web_search_requests": 1
    }
  }
}
```

网页搜索在 Claude API 上的价格为**每 1,000 次搜索 10 美元**，另加搜索生成内容的标准令牌费用。在整个对话过程中检索到的网页搜索结果均计为输入令牌，无论是在单个轮次中执行的搜索迭代，还是在后续的对话轮次中。

每次网页搜索计为一次使用，与返回的结果数量无关。如果网页搜索过程中发生错误，该次网页搜索将不会计费。

#### 网页抓取工具

Web fetch（网页抓取）的使用除标准令牌费用外**不收取任何额外费用**：

```json
{
  "usage": {
    "input_tokens": 25039,
    "output_tokens": 931,
    "cache_read_input_tokens": 0,
    "cache_creation_input_tokens": 0,
    "server_tool_use": {
      "web_fetch_requests": 1
    }
  }
}
```

Web fetch 工具在 Claude API 上**无需额外费用**即可使用。您只需为成为对话上下文一部分的抓取内容支付标准令牌费用。

为防止无意中抓取会消耗过多令牌的大型内容，请使用 `max_content_tokens` 参数，根据您的使用场景和预算考虑设置适当的限制。

典型内容的令牌使用量示例：

* 普通网页（10 kB）：约 2,500 个令牌
* 大型文档页面（100 kB）：约 25,000 个令牌
* 研究论文 PDF（500 kB）：约 125,000 个令牌

#### 计算机使用工具

计算机使用遵循标准的[工具使用定价](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview#pricing)。使用计算机使用工具时：

**工具集定义开销：** 声明 `computer_toolset_20260801` 及其默认成员会为请求增加约 4,500 个输入令牌（在 Claude Fable 5、Claude Mythos 5、Claude Opus 5 和 Claude Opus 4.8 上约为 4,520 个，在 Claude Sonnet 5 上约为 4,590 个），其中涵盖了成员工具定义和工具使用系统提示。通过 `configs` 禁用 `zoom` 可减少其中约 410 个令牌。请求的确切数量会在响应的 `usage` 中报告，您也可以使用[令牌计数端点](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)提前进行估算。

**早期工具版本：** 以下数据适用于 `computer_20251124` 和 `computer_20250124` 工具版本，不适用于 `computer_toolset_20260801`：

* 系统提示开销：向系统提示中添加 466–499 个令牌
* 工具定义：每个工具定义约 735 个输入令牌（使用 `computer_20250124` 测量）

**额外令牌消耗：**

* 工具结果中返回的屏幕截图和缩放图像，按图像输入计费（请参阅[视觉定价](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#evaluate-image-size)）
* 返回给 Claude 的工具执行结果

<Note>
  如果您在计算机使用的同时还使用 bash 或文本编辑器工具，这些工具有各自的令牌成本，详见其各自的文档页面。
</Note>

#### 浏览器使用工具

浏览器使用遵循标准的[工具使用定价](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview#pricing)。使用浏览器使用工具时：

**工具集定义开销：** 声明 `browser_toolset_20260801` 及其默认成员会为请求增加约 6,600 个输入令牌（在 Claude Fable 5、Claude Mythos 5、Claude Opus 5 和 Claude Opus 4.8 上约为 6,610 个，在 Claude Sonnet 5 上约为 6,670 个），其中涵盖成员工具定义和工具使用系统提示。启用全部四个可选成员会增加约 880 个令牌，而使用 `configs` 禁用成员则会减少令牌数量。请求的确切令牌数会在响应的 `usage` 中报告，您也可以使用[令牌计数端点](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)提前进行估算。

**额外的令牌消耗：**

* 工具结果中返回的屏幕截图和缩放图像，按图像输入计费（请参阅[视觉定价](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#evaluate-image-size)）
* 返回给 Claude 的文本工具结果，例如无障碍树、页面文本以及控制台或网络条目

<Note>
  如果您在浏览器使用之外还同时使用计算机使用工具、bash 工具、文本编辑器工具或您自己的工具，这些工具各自有其令牌成本，详见其各自的文档页面。
</Note>

## Claude Managed Agents 定价

[Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 按两个维度计费：令牌和会话运行时间。

### 令牌

Claude Managed Agents 会话消耗的所有令牌均按[模型定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#model-pricing)中所示的费率计费。[提示缓存](https://platform.claude.com/docs/zh-CN/about-claude/pricing#prompt-caching)乘数同样适用。在会话内触发的网页搜索按标准的每 1,000 次搜索 $10 收费。在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-platform-on-aws-pricing) 上，会话令牌和运行时间费用按标准费率转换为 Claude Consumption Units。当代理的 `model.speed` 设置为 `"fast"` 时，适用[快速模式](https://platform.claude.com/docs/zh-CN/about-claude/pricing#fast-mode-pricing)溢价定价。

[数据驻留乘数](https://platform.claude.com/docs/zh-CN/about-claude/pricing#data-residency-pricing)同样适用：当代理的 `model.inference_geo` 固定为 `"us"` 时，运行该代理的会话所消耗的令牌按标准费率的 1.1 倍计费，这与 Messages API 上仅限美国推理所适用的乘数相同。

以下 Messages API 修正项**不**适用于 Claude Managed Agents 会话：

| 修正项                                                                                          | 不适用的原因               |
| -------------------------------------------------------------------------------------------- | -------------------- |
| [Batch API 折扣](https://platform.claude.com/docs/zh-CN/about-claude/pricing#batch-processing) | 会话是有状态且交互式的。没有批处理模式。 |
| [云平台定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#cloud-platform-pricing)  | 在由合作伙伴运营的云平台上不可用。    |

### 会话运行时间

| SKU    | 费率          | 计量方式             |
| ------ | ----------- | ---------------- |
| 会话运行时间 | 每会话小时 $0.08 | `running` 状态持续时间 |

运行时间精确到毫秒计量，且仅在会话状态为 `running` 时累计。处于 `idle`（等待您的下一条消息或工具确认）、`rescheduling` 或 `terminated` 状态的时间不计入运行时间。

<Note>
  使用 Claude Managed Agents 时，会话运行时间取代了[代码执行](https://platform.claude.com/docs/zh-CN/about-claude/pricing#code-execution-tool)的容器小时计费模式。您不会在会话运行时间之外另行支付容器小时费用。
</Note>

### 计算示例

一个使用 Claude Opus 5、消耗 50,000 个输入令牌和 15,000 个输出令牌的一小时编码会话：

| 明细项    | 计算                       | 费用         |
| ------ | ------------------------ | ---------- |
| 输入令牌   | 50,000 × $5 / 1,000,000  | $0.25      |
| 输出令牌   | 15,000 × $25 / 1,000,000 | $0.375     |
| 会话运行时间 | 1.0 小时 × $0.08           | $0.08      |
| **总计** |                          | **$0.705** |

如果提示缓存处于活动状态，且其中 40,000 个输入令牌为缓存读取：

| 明细项      | 计算                            | 费用         |
| -------- | ----------------------------- | ---------- |
| 未缓存的输入令牌 | 10,000 × $5 / 1,000,000       | $0.05      |
| 缓存读取令牌   | 40,000 × $5 × 0.1 / 1,000,000 | $0.02      |
| 输出令牌     | 15,000 × $25 / 1,000,000      | $0.375     |
| 会话运行时间   | 1.0 小时 × $0.08                | $0.08      |
| **总计**   |                               | **$0.525** |

<Note>
  处理 10,000 张支持工单的计算示例：

  * 每次对话平均约 3,700 个令牌
  * 使用 Claude Haiku 4.5，输入 $1/MTok，输出 $5/MTok
  * 总费用：每 10,000 张工单约 $37.00
</Note>

有关此计算的详细演示，请参阅[客户支持代理指南](https://platform.claude.com/docs/zh-CN/about-claude/use-case-guides/customer-support-chat)。

## 其他定价注意事项

### 成本优化策略

使用 Claude 构建代理时：

1. **使用合适的模型：** 简单任务选择 Haiku，大多数生产工作负载选择 Sonnet，最复杂的推理选择 Opus
2. **实施提示缓存：** 降低重复上下文的成本
3. **批量操作：** 对时间不敏感的任务使用 Batch API
4. **监控使用模式：** 跟踪令牌消耗以发现优化机会

<Tip>
  对于高用量的代理应用，请联系[企业销售团队](https://claude.com/contact-sales)以获取定制定价方案。
</Tip>

### 速率限制

速率限制因使用层级而异，并影响您可以发出的请求数量：

* **Start 层级：** 用于入门的初级限制
* **Build 层级：** 为成长中的应用提供更高的限制
* **Scale 层级：** 为生产工作负载提供最高的标准限制

有关速率限制的详细信息，请参阅[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)。

如需超出 Scale 层级的限制或定制定价方案，请[联系销售团队](https://claude.com/contact-sales)。

### 批量折扣

高用量用户可能可以获得批量折扣。这些折扣将根据具体情况逐一协商。

* 标准使用层级使用[模型定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#model-pricing)中所示的定价
* 企业客户可以[联系销售](mailto:sales@anthropic.com)获取定制定价
* 可能提供学术和研究折扣

### 企业定价

对于有特定需求的企业客户：

* 定制速率限制
* 批量折扣
* 专属支持
* 定制条款

请通过 [sales@anthropic.com](mailto:sales@anthropic.com) 或 [Claude Console](https://platform.claude.com/settings/limits) 联系销售团队，讨论企业定价选项。

## 计费与付款

* 计费基于每月实际用量
* 所有付款均以美元结算
* 提供信用卡和发票选项
* 可在 [Claude Console](https://platform.claude.com/) 中跟踪用量

## 常见问题

### 令牌用量如何计算？

令牌是模型处理的文本片段。粗略估计，1 个令牌大约相当于英文中的 4 个字符或 0.75 个单词。确切数量因语言和内容类型而异。

### 是否有免费层级或试用？

新用户会获得少量免费额度用于测试 API。有关企业评估延长试用的信息，请[联系销售](mailto:sales@anthropic.com)。

### 折扣如何叠加？

Batch API 和提示缓存折扣可以组合使用。例如，同时使用这两项功能与标准 API 调用相比可显著节省成本。有关乘数如何相互作用，请参阅[提示缓存定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#prompt-caching)。

### 接受哪些付款方式？

标准账户接受主流信用卡。企业客户可以安排发票及其他付款方式。

如有其他定价相关问题，请联系 [support@anthropic.com](mailto:support@anthropic.com)。
