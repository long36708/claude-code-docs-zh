---
title: 功能概览
url: https://platform.claude.com/docs/zh-CN/build-with-claude/overview
description: 探索 Claude 的高级功能与能力。
---

Claude 的 API 功能面分为五个领域：

* **模型能力：** 控制 Claude 的推理方式和响应格式。
* **工具：** 让 Claude 在网络上或您的环境中执行操作。
* **工具基础设施：** 处理大规模的工具发现与编排。
* **上下文管理：** 保持长时间运行的会话高效。
* **文件与资产：** 管理您提供给 Claude 的文档和数据。

如果您是新用户，请从[模型能力](https://platform.claude.com/docs/zh-CN/build-with-claude/overview#model-capabilities)和[工具](https://platform.claude.com/docs/zh-CN/build-with-claude/overview#tools)开始。当您准备好优化成本、延迟或规模时，再回到其他部分。

有关管理和治理，请参阅 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)、[Usage and Cost API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api) 和 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api)。

## 功能可用性

以下各表中的"可用性"列列出了提供某项功能的平台。未带标签列出的平台表示该功能在该平台上为稳定版、完全受支持并推荐用于生产环境，无需 beta 标头，并享有标准的 API [版本控制](https://platform.claude.com/docs/zh-CN/api/versioning)保证。平台名称后的标签表示该功能在该平台上属于以下分类之一。并非所有功能都会经历每个阶段，某项功能可能在任何阶段进入或跳过某些阶段。

| 分类                  | 描述                                                                                                                                                                                                                                                                                                                                  |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Beta**            | 预览功能，用于收集反馈并针对尚不成熟的用例进行迭代。可用性可能受限，包括需要注册或加入等候名单，并且可能不会公开宣布。 功能可能会根据反馈发生重大变化或被终止。不保证可持续用于生产环境。可能会在通知后发生破坏性变更，并且可能存在某些特定平台的限制。Claude API 和 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上的 Beta 功能带有 [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers)。 |
| **Deprecated（已弃用）** | 功能仍可使用，但不再推荐。会提供迁移路径和移除时间表。                                                                                                                                                                                                                                                                                                         |
| **Retired（已停用）**    | 功能不再可用。                                                                                                                                                                                                                                                                                                                             |

**平台标签：** Claude API（Anthropic 第一方）· [Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)（由 AWS 运营）· [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)（由 Anthropic 在 AWS 上运营）· [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)（由 Google 运营）· [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)（由 Anthropic 在 Azure 上运营）

## 模型能力

引导 Claude 及其直接输出的方式，包括响应格式、推理深度和输入模态。

<Tip>
  您可以通过编程方式查询某个模型支持哪些能力。[Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 会为每个可用模型返回 `max_input_tokens`、`max_tokens` 以及一个 `capabilities` 对象。
</Tip>

ZDR 列表示某项功能是否可在"Zero Data Retention"（零数据保留），即 ZDR 安排下使用。对于大多数功能，这仅取决于该功能机制所保留的内容；对于与特定模型绑定的功能，模型级别的 ZDR 可用性同样适用。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。

| 功能                                                                                      | 描述                                                                                                                                              | 零数据保留（ZDR）                                                                                                     | 可用性                                                                                               |
| --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| [上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)       | 最多 1M 令牌，用于处理大型文档、庞大的代码库和长对话。                                                                                                                   | 符合 ZDR 条件                                                                                                      | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)              | 让 Claude 动态决定何时思考以及思考多少。这是 Claude 4.7 及更高版本模型上唯一的思考模式。使用 effort 参数控制思考深度。                                                                       | 符合 ZDR 条件                                                                                                      | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)        | 异步处理大量请求以节省成本。每个批次可发送包含大量查询的请求。Batch API 调用的费用比标准 API 调用低 50%。                                                                                  | 不符合 ZDR 条件                                                                                                     | <PlatformAvailability claudeApi claudePlatformAws />                                              |
| [引用](https://platform.claude.com/docs/zh-CN/build-with-claude/citations)                | 让 Claude 的响应以源文档为依据。借助 Citations，Claude 可以提供其生成响应时所使用的确切句子和段落的详细引用，从而产生更可验证、更可信的输出。                                                             | 符合 ZDR 条件                                                                                                      | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)             | 使用地理控制来控制模型推理的运行位置。通过 `inference_geo` 参数为每个请求指定 `"global"` 或 `"us"` 路由。                                                                         | 符合 ZDR 条件                                                                                                      | <PlatformAvailability claudeApi claudePlatformAws />                                              |
| [Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)               | 使用 effort 参数控制 Claude 响应时使用的令牌数量，在响应的详尽程度与令牌效率之间进行权衡。                                                                                           | 符合 ZDR 条件                                                                                                      | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)        | 当您在另一个模型上重试被拒绝的请求时，避免重复支付提示缓存费用。拒绝响应会携带一个抵扣令牌，在重试时回传该令牌，重试将按照对话从一开始就在新模型上进行的方式计费。Message Batches 的结果不包含回退抵扣令牌。                                  | 不符合 ZDR 条件\*                                                                                                   | <PlatformAvailability claudeApiBeta claudePlatformAwsBeta bedrockBeta vertexAiBeta azureAiBeta /> |
| [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)          | 处理和分析 PDF 文档中的文本和视觉内容。                                                                                                                          | 符合 ZDR 条件                                                                                                      | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [搜索结果](https://platform.claude.com/docs/zh-CN/build-with-claude/search-results)         | 通过提供带有正确来源归属的搜索结果，为 RAG 应用启用自然引用。为自定义知识库和工具实现网络搜索级别的引用质量。                                                                                       | 符合 ZDR 条件                                                                                                      | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [服务端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback) | 在单次 API 调用内重试被拒绝的请求。使用 `"default"` 模式应用 Anthropic 推荐的回退模型，或自行指定最多三个模型；当所请求的模型拒绝时，API 会对同一请求运行链中的下一个模型。`fallbacks` 参数在 Message Batches API 中不可用。 | 不符合 ZDR 条件\*                                                                                                   | <PlatformAvailability claudeApiBeta />                                                            |
| [结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)    | 通过两种方式保证模式一致性：用于结构化数据响应的 JSON 输出，以及用于经验证的工具输入的严格工具使用。                                                                                           | [符合 ZDR 条件（有限定）](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs#data-retention)\* | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)                 | 针对复杂任务的增强推理能力，在给出最终答案之前，让 Claude 的逐步思考过程透明可见。                                                                                                   | 符合 ZDR 条件                                                                                                      | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |

## 工具

Claude 通过 `tool_use` 调用的内置工具。服务端工具由平台运行；客户端工具由您实现并执行。

### 服务端工具

| 功能                                                                                           | 描述                                                  | ZDR         | 可用性                                                                   |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------- | ----------- | --------------------------------------------------------------------- |
| [顾问工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool)        | 将速度更快的执行模型与智能程度更高的顾问模型配对，后者在生成过程中为长周期智能体工作负载提供战略指导。 | 符合 ZDR 条件   | <PlatformAvailability claudeApiBeta claudePlatformAwsBeta />          |
| [代码执行](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool) | 在沙盒环境中运行代码，用于高级数据分析、计算和文件处理。与网页搜索或网页抓取一起使用时免费。      | 不符合 ZDR 条件  | <PlatformAvailability claudeApi claudePlatformAws azureAi />†         |
| [网页抓取](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)      | 从指定网页和 PDF 文档中检索完整内容以进行深入分析。                        | 符合 ZDR 条件\* | <PlatformAvailability claudeApi claudePlatformAws azureAi />          |
| [网页搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)     | 利用来自整个网络的最新真实世界数据增强 Claude 的全面知识。                   | 符合 ZDR 条件\* | <PlatformAvailability claudeApi claudePlatformAws vertexAi azureAi /> |

### 客户端工具

| 功能                                                                                          | 描述                                                | ZDR       | 可用性                                                                                       |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------- |
| [Bash](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/bash-tool)          | 执行 bash 命令和脚本，与系统 shell 交互并执行命令行操作。               | 符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />             |
| [浏览器使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)  | 在您自己的浏览器环境中导航、阅读网页并与之交互。                          | 符合 ZDR 条件 | <PlatformAvailability claudeApi vertexAi />                                               |
| [计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool) | 通过截取屏幕截图并发出鼠标和键盘命令来控制计算机界面。                       | 符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAwsBeta bedrockBeta vertexAi azureAiBeta /> |
| [记忆](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)          | 使 Claude 能够跨对话存储和检索信息。随时间构建知识库、维护项目上下文并从过去的交互中学习。 | 符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />             |
| [文本编辑器](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/text-editor-tool)  | 使用内置的文本编辑器界面创建和编辑文本文件，以完成文件操作任务。                  | 符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />             |

## 工具基础设施

支持发现、编排和扩展工具使用的基础设施。

| 功能                                                                                                        | 描述                                                                                                           | ZDR        | 可用性                                                                           |
| --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ---------- | ----------------------------------------------------------------------------- |
| [Agent Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview)             | 使用 Skills 扩展 Claude 的能力。使用预构建的 Skills（PowerPoint、Excel、Word、PDF）或通过指令和脚本创建自定义 Skills。Skills 使用渐进式披露来高效管理上下文。 | 不符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAwsBeta azureAiBeta />†         |
| [细粒度工具流式传输](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/fine-grained-tool-streaming) | 无需缓冲/JSON 验证即可流式传输工具使用参数，降低接收大型参数的延迟。                                                                        | 符合 ZDR 条件  | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi /> |
| [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)                          | 直接从 Messages API 连接到远程 [MCP](https://platform.claude.com/docs/zh-CN/mcp) 服务器，无需单独的 MCP 客户端。                  | 不符合 ZDR 条件 | <PlatformAvailability claudeApiBeta claudePlatformAwsBeta azureAiBeta />      |
| [编程式工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling)     | 使 Claude 能够在代码执行容器内以编程方式调用您的工具，降低多工具工作流的延迟和令牌消耗。                                                             | 不符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAws azureAi />†                 |
| [工具搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)                 | 使用基于正则表达式和 BM25 的搜索按需动态发现和加载工具，从而扩展到数千个工具，优化上下文使用并提高工具选择的准确性。                                                | 符合 ZDR 条件  | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi /> |

## 上下文管理

用于控制和优化 Claude 上下文窗口的基础设施。

| 功能                                                                                                          | 描述                                                       | ZDR       | 可用性                                                                                               |
| ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------- |
| [压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)                                   | 针对长时间运行对话的服务端上下文摘要。当上下文接近窗口限制时，API 会自动对对话的较早部分进行摘要。      | 符合 ZDR 条件 | <PlatformAvailability claudeApiBeta claudePlatformAwsBeta bedrockBeta vertexAiBeta azureAiBeta /> |
| [上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)                           | 通过可配置的策略自动管理对话上下文。支持在接近令牌限制时清除工具结果，以及在扩展思考对话中管理思考块。      | 符合 ZDR 条件 | <PlatformAvailability claudeApiBeta claudePlatformAwsBeta bedrockBeta vertexAiBeta azureAiBeta /> |
| [自动提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#automatic-caching)         | 将提示缓存简化为单个 API 参数。系统会自动缓存您请求中最后一个可缓存的块，并随着对话的增长将缓存点向前移动。 | 符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [提示缓存（5 分钟）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)                       | 为 Claude 提供更多背景知识和示例输出，以降低成本和延迟。                         | 符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [提示缓存（1 小时）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#1-hour-cache-duration) | 延长至 1 小时的缓存时长，适用于访问频率较低但重要的上下文，作为标准 5 分钟缓存的补充。           | 符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |
| [令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)                             | 令牌计数使您能够在将消息发送给 Claude 之前确定其中的令牌数量，帮助您就提示和用量做出明智的决策。     | 符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAws bedrock vertexAi azureAi />                     |

## 文件与资产

管理与 Claude 配合使用的文件和资产。

| 功能                                                                          | 描述                                                 | ZDR        | 可用性                                                                   |
| --------------------------------------------------------------------------- | -------------------------------------------------- | ---------- | --------------------------------------------------------------------- |
| [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) | 上传和管理文件以供 Claude 使用，无需在每次请求时重新上传内容。支持 PDF、图像和文本文件。 | 不符合 ZDR 条件 | <PlatformAvailability claudeApi claudePlatformAwsBeta azureAiBeta />† |

\* **结构化输出：** 您的提示和 Claude 的输出不会被存储。仅缓存 JSON 模式，自上次使用起最多保留 24 小时。**网页搜索和网页抓取：** 符合 ZDR 条件，但启用[动态过滤](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool#dynamic-filtering)时除外。**回退抵扣和服务端回退：** 这些功能不保留任何消息内容，但它们处理来自 Claude Fable 模型的拒绝，而这些模型[在 ZDR 下不可用](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。请参阅 [ZDR 详情](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)。

† 在 Microsoft Foundry 上，功能可用性因[托管选项](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry#hosting-options)而异。这些功能在 Hosted on Anthropic 部署上可用，而在 Hosted on Azure 部署上不可用。
