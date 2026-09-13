---
title: 模型 ID 与版本管理
url: https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions
description: Claude 模型 ID 的结构与版本管理方式，包括随 Claude 4.6 代引入的无日期格式，以及它对稳定性的意义。
---

每个 Claude 模型 ID 都标识该模型的一个固定版本。当您在 API 请求中使用某个模型 ID 时，在该 ID 的整个生命周期内，其底层模型保持不变。此保证适用于模型 ID，而不适用于 Claude API 为某些早期模型所接受的便捷别名（请参阅 [4.6 代之前](https://platform.claude.com/docs/zh-CN/about-claude/models/model-ids-and-versions#before-the-4-6-generation)）。

## 模型 ID 格式

Claude 模型 ID 遵循带版本的命名方案。

### 4.6 代及之后

从 Claude 4.6 代开始，模型 ID 使用无日期格式：

```text wrap
claude-{name}-{major}[-{minor}]
```

主版本发布（例如 Claude Sonnet 5 和 Claude Opus 5）会省略次版本段。

例如：`claude-sonnet-4-6`、`claude-sonnet-5`、`claude-opus-4-6`、`claude-opus-4-7`、`claude-opus-4-8` 和 `claude-opus-5`

在 Amazon Bedrock 上，对应的格式为：

```text wrap
anthropic.claude-{name}-{major}[-{minor}]
```

例如：`anthropic.claude-sonnet-4-6`、`anthropic.claude-sonnet-5`、`anthropic.claude-opus-4-7`、`anthropic.claude-opus-4-8`、`anthropic.claude-opus-5`

Claude Opus 4.6 是最后一个包含 `-v1` 后缀的 Bedrock 模型 ID（`anthropic.claude-opus-4-6-v1`）。Anthropic 从 Claude Sonnet 4.6 开始去掉了该后缀。

在 Google Cloud 上，格式与 Claude API 一致。

### 4.6 代之前

4.6 代之前的模型在 ID 中包含一个 "snapshot date"（快照日期）：

```text wrap
claude-{name}-{major}-{minor}-{YYYYMMDD}
```

例如：`claude-sonnet-4-5-20250929`、`claude-haiku-4-5-20251001`

在 Amazon Bedrock 上，这些模型使用以下格式：

```text wrap
anthropic.claude-{name}-{major}-{minor}-{YYYYMMDD}-v1:0
```

例如：`anthropic.claude-sonnet-4-5-20250929-v1:0`

在 Google Cloud 上，日期以 `@` 分隔：

```text wrap
claude-{name}-{major}-{minor}@{YYYYMMDD}
```

例如：`claude-haiku-4-5@20251001`

在 Claude API 上，这些模型还有更短的 "alias"（别名）（例如 `claude-sonnet-4-5`），指向该次版本最新的带日期快照。

## 无日期 ID 是固定快照

一个常见的误解是，诸如 `claude-sonnet-4-6` 之类的无日期模型 ID 会像常青指针一样，路由到最新或性能最佳的版本。事实并非如此。

对于 4.6 代及之后的模型，无日期 ID 就是该版本的规范模型 ID。它映射到单一、固定的模型快照。Anthropic 不会更新现有模型 ID 的权重或配置。当有更新版本可用时，它会以新的模型 ID 发布。

这与 Claude API 上为早期模型提供的无日期别名不同。诸如 `claude-sonnet-4-5` 之类的别名是一个便捷指针，会解析为该次版本最新的带日期快照。而诸如 `claude-sonnet-4-6` 之类的 4.6 代 ID 并不是别名，它本身就是快照。

每个模型 ID，无论带日期还是无日期，都有各自独立的弃用和停用时间表。

## 模型权重与服务基础设施

对于给定的 ID，"model weights"（模型权重）是固定的，但围绕模型的 "serving infrastructure"（服务基础设施）可能会随时间变化。该基础设施包括请求路由器、安全分类器和采样逻辑等组件。

偶尔，即使模型 ID 和权重没有变化，基础设施更新也会在可观察的行为上产生细微差异。如果您在一个此前稳定的模型 ID 上注意到意外的行为差异，基础设施更新是最可能的原因。

## 当前模型 ID

有关当前模型 ID 的完整列表及其在 Amazon Bedrock 和 Google Cloud 上的对应 ID，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。
