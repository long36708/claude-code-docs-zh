---
title: Claude Fable 5 和 Claude Mythos 5 简介
url: https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5
description: Claude Fable 5 和 Claude Mythos 5 的功能、API 变更及可用性。
---

<Note>
  Claude Fable 5.1 和 Claude Mythos 5.1 基于这些模型构建。请参阅 [Claude Fable 5.1 的新特性](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1)。
</Note>

<Tip>
  对 Claude Fable 5 和 Claude Mythos 5 的访问已恢复。请参阅[我们的声明](https://www.anthropic.com/news/redeploying-fable-5)了解更多信息。
</Tip>

Claude Fable 5 专为高要求的推理和长周期智能体工作而构建。Claude Mythos 5 具有相同的功能，仅通过 [Project Glasswing](https://anthropic.com/glasswing) 以有限发布的形式提供。

对集成而言最重要的变化是：Claude Fable 5 包含可以拒绝请求的安全分类器。Claude Mythos 5 不包含这些分类器。如果您的集成调用 Claude Fable 5，请为三项变更做好规划：针对拒绝的新响应处理、在另一个 Claude 模型上重试的回退选项，以及新的计费规则。[Claude Fable 5 上的拒绝、回退和计费](https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5#refusals-fallback-and-billing-on-claude-fable-5)对这三项进行了总结。

## 模型

| 模型              | API 模型 ID         | 描述                                                                                 |
| --------------- | ----------------- | ---------------------------------------------------------------------------------- |
| Claude Fable 5  | `claude-fable-5`  | 专为高要求的推理和长周期智能体工作而构建                                                               |
| Claude Mythos 5 | `claude-mythos-5` | 具有 Claude Fable 5 的功能，但不含安全分类器。通过 Project Glasswing 提供。Claude Mythos Preview 的继任者。 |

Claude Fable 5 和 Claude Mythos 5 具有相同的规格和定价：

* **上下文窗口和输出：** 默认提供 [100 万令牌的 "context window"（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)，每个请求最多可输出 128k 令牌。
* **定价：** 每百万输入令牌 10 美元，每百万输出令牌 50 美元。

有关所有当前模型的规格，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。

## Claude Fable 5 上的拒绝、回退和计费

Claude Fable 5 包含可以拒绝某些请求的安全分类器。Claude Mythos 5 不包含这些分类器，因此本节仅适用于 Claude Fable 5。以下各节总结了拒绝对您的集成意味着什么。每一节都链接到完整指南。

### 拒绝

当 Claude Fable 5 拒绝请求时，Messages API 会以成功的 HTTP 200 响应返回 `stop_reason: "refusal"`，而不是错误。响应还会报告是哪个分类器拒绝了该请求。有关响应结构和处理指南，请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。

### 回退

Claude Fable 5 拒绝的请求通常可以由另一个 Claude 模型处理。有三种重试方式：

* **服务器端：** 传递 `fallbacks` 参数让 API 为您重试，可使用其 `"default"` 模式采用 Anthropic 推荐的模型，或自行指定模型（在 Claude API 上处于测试阶段）。请参阅[服务器端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)。
* **客户端：** 使用 [SDK 中间件](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/middleware)在任何平台上从客户端重试。请参阅[客户端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#client-side-fallback)。
* **手动：** 在任何平台上使用任何语言自行构建重试逻辑。请参阅[回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)。

### 计费

对于在生成任何输出之前就被拒绝的请求，您不会被计费。当您在另一个模型上重试时，[回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)会退还切换模型产生的 "prompt caching"（提示缓存）成本，从而避免您为该成本支付两次。

## 可用性

* **Claude Fable 5** 可在 Claude API、[Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)、[Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 和 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上使用。
* **Claude Mythos 5** 仅向 [Project Glasswing](https://anthropic.com/glasswing) 中获得批准的客户提供。如需访问权限，请联系您的 Anthropic、AWS 或 Google Cloud 客户团队。无法访问 Claude Mythos 5 的客户可以使用 Claude Fable 5，它不需要访问审批，并提供相同的功能。

Claude Fable 5 和 Claude Mythos 5 采用 30 天数据保留期，除非获得 Anthropic 明确授权，否则不适用零数据保留。两者均被指定为[受保护模型（Covered Models）](https://support.claude.com/en/articles/15425695)。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。

## 提示

Claude Fable 5 对与其他 Claude 模型相同的提示技巧做出响应，但在如何构建长上下文提示和推理指令方面存在一些差异。请参阅[为 Claude Fable 5 编写提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5)。

## Claude Fable 5 和 Claude Mythos 5 上的 Messages API

### 自适应思考始终开启

Claude Fable 5 和 Claude Mythos 5 始终启用思考。不支持传递 `thinking: {"type": "disabled"}`。要降低或以其他方式控制思考深度，请使用 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 参数。

### 从不返回原始思考内容

Claude Fable 5 和 Claude Mythos 5 从不返回原始思维链。`thinking.display` 设置控制思考块中包含的内容：

* `"summarized"` 返回包含可读推理摘要的思考块。
* `"omitted"`（默认值）返回 `thinking` 字段为空的思考块。

在同一模型上的多轮对话中，请原样传回思考块。有关跨模型处理，请参阅 [Claude Fable 5 和 Claude Mythos 5 上的思考输出](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-output-on-claude-fable-5-and-claude-mythos-5)。

## 支持的功能

Claude Fable 5 和 Claude Mythos 5 支持：

* [Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)
* [任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)（测试版：设置 `task-budgets-2026-03-13` 标头）
* [记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)
* [代码执行](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)
* [程序化工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling)
* 通过[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)清除工具结果（测试版：设置 `context-management-2025-06-27` 标头）
* [压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)
* [视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)

## 从早期模型迁移

分步说明位于迁移指南中：

* 从 Claude Mythos Preview 迁移：请参阅[从 Claude Mythos Preview 迁移到 Claude Mythos 5](https://platform.claude.com/docs/zh-CN/models/fable-5/migration-guide#migrating-from-claude-mythos-preview)。
* 从 Claude Opus 4.8 迁移：请参阅[从 Claude Opus 4.8 迁移到 Claude Fable 5](https://platform.claude.com/docs/zh-CN/models/fable-5/migration-guide#migrating-from-claude-opus-48)。

## 后续步骤

<CardGroup>
  <Card title="模型概览" icon="settings" href="https://platform.claude.com/docs/zh-CN/models/overview">
    所有当前 Claude 模型的规格和比较。
  </Card>

  <Card title="自适应思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    Claude Fable 5 和 Claude Mythos 5 上唯一的思考模式。
  </Card>

  <Card title="拒绝与回退" icon="shield" href="https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback">
    Claude Fable 5 如何拒绝请求，以及如何在另一个模型上重试。
  </Card>

  <Card title="回退抵扣" icon="coins" href="https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit">
    避免在重试时为提示缓存成本支付两次。
  </Card>

  <Card title="回退与计费 cookbook" icon="book-open" href="https://platform.claude.com/cookbook/fable-5-fallback-billing-guide">
    一个关于拒绝处理、回退和计费的完整端到端示例。
  </Card>

  <Card title="Effort" icon="sliders" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    控制 Claude Fable 5 和 Claude Mythos 5 上的思考深度和成本。
  </Card>

  <Card title="为 Claude Fable 5 编写提示" icon="terminal" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5">
    Fable 专属的提示技巧。
  </Card>
</CardGroup>
