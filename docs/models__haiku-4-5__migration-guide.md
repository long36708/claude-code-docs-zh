---
title: 迁移到 Claude Haiku 4.5
url: https://platform.claude.com/docs/zh-CN/models/haiku-4-5/migration-guide
description: 从早期 Haiku 模型迁移到 Claude Haiku 4.5：模型 ID、破坏性变更以及迁移检查清单。
---

<Note>
  本指南涵盖 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 代码的迁移。如果您使用 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)，则除了更新模型名称之外无需进行任何更改。
</Note>

<Tip>
  **使用 Claude API skill 自动完成迁移。** 在 Claude Code 中，运行 `/claude-api migrate` 以调用内置的 [Claude API skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model)。它适用于以任何当前 Claude 模型作为目标：

  ```text wrap
  /claude-api migrate this project to claude-haiku-4-5
  ```

  该 skill 会在您的整个代码库中应用模型 ID 替换，并根据需要处理破坏性参数变更、prefill（预填充）替换以及针对目标模型的 effort（努力程度）校准，然后生成一份需要手动验证的事项清单。在编辑任何文件之前，它会要求您确认迁移范围（整个工作目录、某个子目录或特定的文件列表）。该 skill 还会检测 Amazon Bedrock 和 Claude Platform on AWS 客户端，并针对这些平台调整模型 ID 格式和功能变更。
</Tip>

Claude Haiku 4.5 是速度最快、最智能的 Haiku 模型，具有接近前沿的性能，为交互式应用和大批量处理提供高端模型质量。

有关功能的完整概述，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。

<Note>
  有关 Claude Haiku 4.5 的定价，请参阅 [Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。
</Note>

<Tip>
  若要在编码和推理任务上获得显著的性能提升，请考虑通过 `thinking: {type: "enabled", budget_tokens: N}` 启用 "extended thinking"（扩展思考）。
</Tip>

<Note>
  扩展思考会影响 "prompt caching"（提示缓存）的[效率](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#caching-with-thinking-blocks)。

  扩展思考在 Claude 4.6 模型中已弃用，并在 Claude Opus 4.7 中被移除。如果使用较新的模型，请改用[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。
</Note>

## 从 Claude Haiku 3.5 及更早的 Haiku 模型迁移到 Claude Haiku 4.5

**更新您的模型名称：**

```python
# 来自 Haiku 3.5
model = "claude-3-5-haiku-20241022"  # Before
model = "claude-haiku-4-5-20251001"  # After
```

**查看新的速率限制：** Haiku 4.5 的 "rate limit"（速率限制）与 Haiku 3.5 是分开的。详情请参阅[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)文档。

**探索新功能：** 有关上下文感知、更大的输出容量（64k 令牌）、更高的智能水平以及更快的速度的详细信息，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。

### 破坏性变更

这些破坏性变更适用于从 Claude 3.x Haiku 模型迁移的情况。

1. **更新采样参数**

   <Warning>
     从 Claude 3.x 模型迁移时，这是一项破坏性变更。
   </Warning>

   仅使用 `temperature` 或 `top_p` 其中之一，不要同时使用两者。在 Claude Haiku 4.5 上同时设置两者会返回 400 错误。

2. **更新工具版本**

   <Warning>
     从 Claude 3.x 模型迁移时，这是一项破坏性变更。
   </Warning>

   更新到最新的工具版本（`text_editor_20250728`、`code_execution_20250825`）。移除所有使用 `undo_edit` 命令的代码。

3. **处理 `refusal` 停止原因**

   更新您的应用程序以[处理 `refusal` 停止原因](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/handle-streaming-refusals)。

4. **针对行为变化更新您的提示**

   Claude 4 模型具有更简洁、更直接的沟通风格。请查看[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)以获取优化指导。

### Haiku 4.5 迁移检查清单

* 将模型 ID 更新为 `claude-haiku-4-5-20251001`
* **破坏性变更：** 将工具版本更新到最新（`text_editor_20250728`、`code_execution_20250825`）；不支持旧版本
* **破坏性变更：** 移除所有使用 `undo_edit` 命令的代码（如适用）
* **破坏性变更：** 更新采样参数，仅使用 `temperature` 或 `top_p` 其中之一，不要同时使用两者（同时设置两者会返回 400 错误）
* 在您的应用程序中处理新的 `refusal` 停止原因
* 查看并针对新的速率限制进行调整（与 Haiku 3.5 分开）
* 按照[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)查看并更新提示
* 考虑为复杂推理任务启用扩展思考
* 在生产部署之前先在开发环境中进行测试
