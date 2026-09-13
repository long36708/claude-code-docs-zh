---
title: 迁移到 Claude Opus 5
url: https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide
description: 从早期 Claude 模型迁移到 Claude Opus 5：模型 ID、破坏性变更、建议变更以及迁移检查清单。
---

<Note>
  本指南涵盖 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 代码的迁移。如果您使用 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)，则除了更新模型名称之外无需进行任何更改。
</Note>

<Tip>
  **使用 Claude API skill 自动完成迁移。** 在 Claude Code 中，运行 `/claude-api migrate` 以调用内置的 [Claude API skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model)。它适用于以任何当前 Claude 模型作为目标：

  ```text wrap
  /claude-api migrate this project to claude-opus-5
  ```

  该 skill 会在您的整个代码库中应用模型 ID 替换，并根据需要处理破坏性参数变更、prefill（预填充）替换以及针对目标模型的 effort（努力程度）校准，然后生成一份需要手动验证的事项清单。在编辑任何文件之前，它会要求您确认迁移范围（整个工作目录、某个子目录或特定的文件列表）。该 skill 还会检测 Amazon Bedrock 和 Claude Platform on AWS 客户端，并针对这些平台调整模型 ID 格式和功能变更。
</Tip>

Claude Opus 5 相比 Claude Opus 4.8 是一次阶跃式的改进，在深度推理、智能体与长周期任务以及测试时计算扩展方面表现强劲。有关行为差异和特定于模型的提示模式，请参阅 [Claude Opus 5 提示指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5)。

Claude Opus 5 是 Claude Opus 4.8 的直接替换升级，定价相同，即每百万输入令牌 5 美元、每百万输出令牌 25 美元；请参阅 [Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。对于已在 Claude Opus 4.8 上运行的代码，有两项破坏性变更，详见[破坏性变更](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#breaking-changes)。Claude Opus 5 支持与 Claude Opus 4.8 相同的功能集，包括 [1M 令牌 "context window"（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)（默认启用，无需 beta 头）、[128k 最大输出令牌](https://platform.claude.com/docs/zh-CN/models/overview)、["adaptive thinking"（自适应思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)、["prompt caching"（提示缓存）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)、[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)、[Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)、[PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)、[视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)，以及服务器端和客户端[工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)，但有两个例外：[web fetch](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool) 在 Claude Opus 5 上不可用，且 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models) 在 Claude Opus 5 上不受支持。有关模型可用性，请参阅各工具页面。

## 从 Claude Opus 4.8 迁移到 Claude Opus 5

<Note>
  本节仅涵盖相对于 Claude Opus 4.8 的差异。如果您的代码基于 Claude Opus 4.7 或更早版本，请改用以下章节：[从 Claude Opus 4.7 迁移到 Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-47) 或 [从 Claude Opus 4.6 及更早的 Opus 模型迁移到 Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-46)。这些章节包含本节的差异，外加来自早期模型的破坏性变更（采样参数被拒绝、手动扩展思考被拒绝、预填充被移除、新的分词器）。
</Note>

### 更新您的模型名称

```python
# Opus 迁移
model = "claude-opus-4-8"  # Before
model = "claude-opus-5"  # After
```

`claude-opus-5` 是一个不带日期后缀的固定模型 ID，与 `claude-opus-4-8` 和 `claude-sonnet-5` 采用相同的命名方案。

### 破坏性变更

1. **默认开启思考：** 在 Claude Opus 4.8 上，不带 `thinking` 字段的请求在不思考的情况下运行；在 Claude Opus 5 上，相同的请求会以[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)运行。`max_tokens` 仍然是总输出（思考加响应文本）的硬性上限，因此对于在 Claude Opus 4.8 上不思考运行的工作负载，请重新审视该值。即使思考文本未返回给您，思考令牌也按输出令牌计费，因此尽管每令牌定价不变，在 Claude Opus 4.8 上不思考运行的工作负载在 Claude Opus 5 上每个请求可能产生更多输出令牌；请参阅[成本控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#cost-control)。要保留旧行为，请传递 `thinking: {type: "disabled"}`，但须遵守下一项中的 effort 上限；请注意，在禁用思考的情况下，模型偶尔可能会以纯文本形式输出工具调用，或在其可见输出中包含内部 XML 标签，因此在可能的情况下，请优先使用启用思考的较低 effort 级别；在无法做到时，请参阅[在禁用思考的情况下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)了解缓解措施。

   响应结构也随之改变。开启思考后，响应可能在第一个 `text` 块之前以一个或多个 `thinking` 块开头，并且由于 `thinking.display` 在 Claude Opus 5 上默认为 `"omitted"`，这些块到达时 `thinking` 字段为空，同时附带其 `signature`。按位置读取回复的代码（例如 `content[0].text`，或将第一个 `content_block_start` 事件视为文本的流处理程序）在这些响应上会出错。请改为按 `type` 字段选择内容块：从 `type` 为 `"text"` 的块中读取 `text`，并在处理流事件时根据块类型进行分支。要接收可读的思考摘要而非空的 `thinking` 字段，请设置 `display: "summarized"`；请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。

   如果您运行工具使用循环，请在返回工具结果时将每个助手响应中的 `thinking` 块完整且未经修改地传回 API，包括 `thinking` 字段为空的块。请按原样回传助手消息，而不是按类型过滤其内容块或重新构建它：API 会以 400 错误拒绝经过编辑、重新排序或部分丢弃的思考块。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。

2. **禁用思考的上限为 `high` effort：** 您仍然可以通过 `thinking: {type: "disabled"}` 关闭思考，但仅限于 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别为 `high` 或更低时。将 `thinking: {type: "disabled"}` 与 effort `xhigh` 或 `max` 组合的请求会返回 400 错误。Claude Opus 4.8 接受这种组合，因此请在迁移前审查禁用思考的请求。

   该检查在每个请求上强制执行：每个请求的 effort 和思考配置都会独立验证，因此在禁用思考的情况下将 effort 提升到 `xhigh` 或 `max` 的请求会被拒绝，即使对话中较早的请求已被接受。

   之前（在 Claude Opus 4.8 上被接受，在 Claude Opus 5 上被拒绝）：

   ```python
   client.messages.create(
       model="claude-opus-4-8",
       max_tokens=16000,
       thinking={"type": "disabled"},
       output_config={"effort": "xhigh"},
       messages=[{"role": "user", "content": "..."}],
   )
   ```

   之后（Claude Opus 5），要么移除 `thinking` 字段以重新启用思考：

   ```python
   client.messages.create(
       model="claude-opus-5",
       max_tokens=16000,
       output_config={"effort": "xhigh"},  # thinking is on by default
       messages=[{"role": "user", "content": "..."}],
   )
   ```

   要么保持禁用思考并降低 effort：

   ```python
   client.messages.create(
       model="claude-opus-5",
       max_tokens=16000,
       thinking={"type": "disabled"},
       output_config={"effort": "high"},  # or "medium", "low"
       messages=[{"role": "user", "content": "..."}],
   )
   ```

### 建议变更

这些变更不是必需的，但会改善您的体验：

1. **为能力关键型工作测试 `max` effort：** Claude Opus 5 支持全套 [effort 级别](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)（`low`、`medium`、`high`、`xhigh`、`max`）。在最大能力比令牌开销更重要的场景中，请测试 `max` effort。它可以在最苛刻的任务上带来收益，但可能因令牌使用量增加而出现收益递减，并且在较简单的任务上可能容易过度思考。如果您以 `xhigh` 或 `max` effort 运行，请设置较大的 `max_tokens`，以便模型有足够空间进行思考和行动；从 64k 令牌开始，然后据此调整。

2. **考虑自动回退：** Claude Opus 5 附带网络安全安全分类器，其网络安全类别的拒绝可以回退到 Claude Opus 4.8。要在另一个模型上自动重新运行被拒绝的请求，请考虑使用带 `"default"` 模式的 `fallbacks` 参数（`fallbacks: "default"`），它会根据拒绝类别选择推荐的回退模型，而不是依赖手动维护的模型列表。服务器端回退处于 beta 阶段；`"default"` 模式需要 `server-side-fallback-2026-07-01` beta 头。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。

3. **缓存更短的提示：** Claude Opus 5 上可缓存的最小提示长度为 512 令牌，低于 Claude Opus 4.8 上的 1,024 令牌。在 Claude Opus 4.8 上因过短而无法缓存的提示现在可以创建缓存条目，无需更改代码。有关各模型的最小值，请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#cache-limitations)。

4. **在对话中途更改工具（beta）：** 您可以在对话的轮次之间添加或移除工具，而不会使较早轮次的[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)命中失效。请发送 beta 头 `mid-conversation-tool-changes-2026-07-01`。这对于随任务推进逐步公开工具或淘汰工具的智能体工作负载很有用；如果不使用它，更改后的工具列表会使缓存的前缀失效。

5. **重新调整长度和详细程度提示：** Claude Opus 5 上默认的可见响应和书面交付物比 Claude Opus 4.8 上更长，而降低 effort 会减少思考量，但不能可靠地缩短可见响应。请改为明确提示要求简洁或目标长度。请参阅[响应长度和详细程度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#response-length-and-verbosity)和[书面交付物长度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#written-deliverable-length)。

6. **移除沿用的验证指令并限制范围：** Claude Opus 5 无需指示即会验证自己的工作，因此请移除从为早期模型调优的提示中沿用下来的显式验证或自检指令；保留它们会导致过度验证。对于范围狭窄的任务，请明确限制任务范围。在多智能体框架中，请就哪些场景需要委派给出明确指导，或限制子智能体的数量，因为 Claude Opus 5 比早期模型更容易进行委派。请参阅[任务范围与过度验证](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#task-scope-and-over-verification)和[控制子智能体生成](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#controlling-subagent-spawning)。

### 迁移检查清单

* 将模型名称从 `claude-opus-4-8` 更新为 `claude-opus-5`。
* 审查不带 `thinking` 字段运行的工作负载：它们在 Claude Opus 5 上会开启思考运行。重新审视 `max_tokens`（它仍然是总输出（思考加响应文本）的硬性上限），或在 effort 为 `high` 或更低时传递 `thinking: {type: "disabled"}` 以保留旧行为。如果您禁用思考，请查看[在禁用思考的情况下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)，了解可能出现的输出瑕疵及其提示缓解措施。
* 更新按位置读取内容的响应解析代码，例如 `content[0].text` 或假定第一个内容块为文本的流处理程序：开启思考后，`thinking` 块会在 `text` 块之前到达。请改为按 `type` 选择内容块。
* 如果您运行工具使用循环，请在返回工具结果时将 `thinking` 块完整且未经修改地传回；修改过的块会返回 400 错误。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。
* 验证任何解析 `thinking` 字段的代码仅将其视为显示文本。`thinking.display` 在 Claude Opus 5 上默认为 `"omitted"`，与 Claude Opus 4.8 相同，因此思考块到达时 `thinking` 字段为空；设置 `display: "summarized"` 以接收可读摘要。请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。
* 审查禁用思考的请求：`thinking: {type: "disabled"}` 与 effort `xhigh` 或 `max` 组合会返回 400 错误，并在每个请求上强制执行。请重新启用思考或将 effort 降低到 `high` 或更低。
* 重新评估您的 `effort` 设置：在您自己的评估上重新进行一次 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 扫描，而不是沿用为早期模型调优的设置。`low` 和 `medium` effort 值得作为成本和延迟控制手段进行测试，并在最大能力比令牌开销更重要的场景中测试 `max` effort。如果您以 `xhigh` 或 `max` effort 运行，请将 `max_tokens` 提高到至少 64k 作为起点。
* 审查接近缓存最小值的提示：512 令牌或以上的提示现在可以创建缓存条目，低于 Claude Opus 4.8 上的 1,024 令牌。
* 处理 `stop_reason: "refusal"`，并考虑使用 `fallbacks: "default"`（beta）在推荐的回退模型上自动重新运行被拒绝的请求。
* 如果您的组织有 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models) 承诺，请单独规划容量：Priority Tier 在 Claude Opus 5 上不受支持，而 Claude Opus 4.8 保留该支持。
* 对于智能体工作负载，请考虑[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)（beta）和对话中途工具更改（beta）。
* 重新调整长度和详细程度提示：Claude Opus 5 上默认的可见响应和书面交付物更长，而降低 effort 会减少思考量，但不能可靠地缩短可见响应。请明确提示要求简洁或目标长度。请参阅[响应长度和详细程度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#response-length-and-verbosity)和[书面交付物长度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#written-deliverable-length)。
* 移除从为早期模型调优的提示中沿用下来的验证和自检指令（它们在 Claude Opus 5 上会导致过度验证），对范围狭窄的任务明确限制任务范围，并在多智能体框架中引导或限制子智能体委派。请参阅[任务范围与过度验证](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#task-scope-and-over-verification)和[控制子智能体生成](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#controlling-subagent-spawning)。
* 在您自己的工作负载上重新建立成本和延迟基线。每令牌定价与 Claude Opus 4.8 相比没有变化，但思考令牌按输出令牌计费，因此原本不思考运行的工作负载每个请求可能产生更多输出令牌。

## 从 Claude Opus 4.7 迁移到 Claude Opus 5

Claude Opus 5 在现有的 Claude Opus 4.7 提示和评估上应具有强劲的开箱即用性能，定价相同，即每百万输入令牌 5 美元、每百万输出令牌 25 美元。它支持与 Claude Opus 4.7 相同的功能集，包括 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)、[128k 最大输出令牌](https://platform.claude.com/docs/zh-CN/models/overview)、[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)、[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)、[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)、[Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)、[PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)、[视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)，以及服务器端和客户端[工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)，但有两个例外：[web fetch](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool) 在 Claude Opus 5 上不可用，且 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models) 在 Claude Opus 5 上不受支持。它还新增了[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)，并公开记录了[拒绝停止详情](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response)。在 Claude API 和 Google Cloud 上，Claude Opus 5 还支持作为稳定版 `computer_toolset_20260801` 工具集的[计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)，以及用于网页内任务的[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)，这两者 Claude Opus 4.7 均不支持；基于早期 `computer_20251124` 版本的现有集成在两个模型上均可继续正常工作，无需更改。要升级现有集成，请参阅[从 `computer_20251124` 迁移](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#migrate-from-computer-20251124)。

<Note>
  如果您的代码基于 Claude Opus 4.6 或更早版本，请改用[从 Claude Opus 4.6 及更早的 Opus 模型迁移到 Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-46)。该章节包含仅从 Claude Opus 4.7 升级所未涵盖的破坏性变更（采样参数被拒绝、手动扩展思考被拒绝、新的分词器）。
</Note>

### 更新您的模型名称

```python
# Opus 迁移
model = "claude-opus-4-7"  # Before
model = "claude-opus-5"  # After
```

### 破坏性变更

1. **默认开启思考：** 在 Claude Opus 4.7 上，不带 `thinking` 字段的请求在不思考的情况下运行；在 Claude Opus 5 上，相同的请求会以[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)运行。`max_tokens` 仍然是总输出（思考加响应文本）的硬性上限，因此对于在 Claude Opus 4.7 上不思考运行的工作负载，请重新审视该值。即使思考文本未返回给您，思考令牌也按输出令牌计费，因此尽管每令牌定价不变，在 Claude Opus 4.7 上不思考运行的工作负载在 Claude Opus 5 上每个请求可能产生更多输出令牌；请参阅[成本控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#cost-control)。要保留旧行为，请传递 `thinking: {type: "disabled"}`，但须遵守下一项中的 effort 上限；请注意，在禁用思考的情况下，模型偶尔可能会以纯文本形式输出工具调用，或在其可见输出中包含内部 XML 标签，因此在可能的情况下，请优先使用启用思考的较低 effort 级别；在无法做到时，请参阅[在禁用思考的情况下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)了解缓解措施。

   响应结构也随之改变。开启思考后，响应可能在第一个 `text` 块之前以一个或多个 `thinking` 块开头，并且由于 `thinking.display` 在 Claude Opus 5 上默认为 `"omitted"`，这些块到达时 `thinking` 字段为空，同时附带其 `signature`。按位置读取回复的代码（例如 `content[0].text`，或将第一个 `content_block_start` 事件视为文本的流处理程序）在这些响应上会出错。请改为按 `type` 字段选择内容块：从 `type` 为 `"text"` 的块中读取 `text`，并在处理流事件时根据块类型进行分支。要接收可读的思考摘要而非空的 `thinking` 字段，请设置 `display: "summarized"`；请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。

   如果您运行工具使用循环，请在返回工具结果时将每个助手响应中的 `thinking` 块完整且未经修改地传回 API，包括 `thinking` 字段为空的块。请按原样回传助手消息，而不是按类型过滤其内容块或重新构建它：API 会以 400 错误拒绝经过编辑、重新排序或部分丢弃的思考块。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。

2. **禁用思考的上限为 `high` effort：** 您可以通过 `thinking: {type: "disabled"}` 关闭思考，但仅限于 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别为 `high` 或更低时。将 `thinking: {type: "disabled"}` 与 effort `xhigh` 或 `max` 组合的请求会返回 400 错误。Claude Opus 4.7 接受这种组合，因此请在迁移前审查禁用思考的请求。

   该检查在每个请求上强制执行：每个请求的 effort 和思考配置都会独立验证，因此在禁用思考的情况下将 effort 提升到 `xhigh` 或 `max` 的请求会被拒绝，即使对话中较早的请求已被接受。

   之前（在 Claude Opus 4.7 上被接受，在 Claude Opus 5 上被拒绝）：

   ```python
   client.messages.create(
       model="claude-opus-4-7",
       max_tokens=16000,
       thinking={"type": "disabled"},
       output_config={"effort": "xhigh"},
       messages=[{"role": "user", "content": "..."}],
   )
   ```

   之后（Claude Opus 5），要么移除 `thinking` 字段以开启思考运行：

   ```python
   client.messages.create(
       model="claude-opus-5",
       max_tokens=16000,
       output_config={"effort": "xhigh"},  # thinking is on by default
       messages=[{"role": "user", "content": "..."}],
   )
   ```

   要么保持禁用思考并降低 effort：

   ```python
   client.messages.create(
       model="claude-opus-5",
       max_tokens=16000,
       thinking={"type": "disabled"},
       output_config={"effort": "high"},  # or "medium", "low"
       messages=[{"role": "user", "content": "..."}],
   )
   ```

### 变更内容

以下各项不是破坏性变更；它们描述了在您更换模型 ID 后值得检查的行为差异。

1. **采样参数（未变更）：** 将 `temperature`、`top_p` 或 `top_k` 设置为非默认值在 Claude Opus 5 上会返回 400 错误，与 Claude Opus 4.7 相同。大多数 SDK 为了与早期模型兼容仍然定义了这些字段，因此设置它们的代码可以通过类型检查，即使 API 会拒绝该请求。Python SDK（v1.0 及更高版本）未定义这些字段，传递它们会引发 `TypeError`。如果您在迁移到 Opus 4.7 时已移除这些参数，则无需进一步更改。

2. **Effort 默认值为 `high`：** Claude Opus 5 上的 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)默认值在 Claude API 和 Claude Code 上为 `high`。如果您已显式设置 effort，您的设置保持不变。

3. **Effort 级别重新校准：** 与 Claude Opus 4.7 相比，Claude Opus 5 上每个 effort 级别背后的令牌分配有所变化，并且 Claude Opus 5 支持全套 effort 级别（`low`、`medium`、`high`、`xhigh`、`max`）。请在您自己的评估上重新进行一次 effort 扫描，而不是沿用为 Claude Opus 4.7 调优的设置。`low` 和 `medium` effort 值得作为成本和延迟控制手段进行测试，并在最大能力比令牌开销更重要的场景中测试 `max` effort。如果您以 `xhigh` 或 `max` effort 运行，请设置较大的 `max_tokens`，以便模型有足够空间进行思考和行动；从 64k 令牌开始，然后据此调整。请参阅 [Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)。

4. **1M 上下文窗口为默认值：** Claude Opus 5 默认提供完整的 1M 令牌[上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)，无需 beta 头，也没有长上下文溢价。如果您的客户端为了与旧模型兼容而传递了上下文窗口 beta 头，您可以在 Claude Opus 5 上将其移除。

5. **对话中途系统消息：** Claude Opus 5 接受在 `messages` 数组中紧跟用户轮次之后的 `role: "system"` 消息（须遵守[放置规则](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)）。对于从一开始就适用的指令，请使用顶层 `system` 字段。Claude Opus 4.7 会以 400 错误拒绝 `messages` 中的 `role: "system"`。如果您维护着通过重建完整消息历史来更新指令的代码路径，您可以简化它们，并保留较早轮次的[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)命中。

6. **拒绝停止详情：** 拒绝响应上的 `stop_details` 对象（自 Claude Opus 4.7 起可用）现已公开记录。当模型拒绝请求时，除了现有的 `refusal` 停止原因外，它还会标识拒绝的类别。无需 beta 头，也无法选择退出。请参阅[处理停止原因](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons)。

7. **更低的提示缓存最小值：** Claude Opus 5 上可缓存的最小提示长度为 512 令牌，低于 Claude Opus 4.7。在 Claude Opus 4.7 上因过短而无法缓存的提示现在可以创建缓存条目，无需更改代码。有关各模型的最小值，请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#cache-limitations)。

8. **快速模式：** Claude Opus 5 支持[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)（研究预览版）；快速模式在 Claude Opus 4.7 上不可用，在该模型上带有 `speed: "fast"` 的请求会返回错误。`speed: "fast"` 参数和 `fast-mode-2026-02-01` beta 头在 Claude Opus 5 上可照常使用，无需更改。

### 建议变更

这些变更不是必需的，但会改善您的体验：

1. **考虑自动回退：** Claude Opus 5 附带网络安全安全分类器，其网络安全类别的拒绝可以回退到 Claude Opus 4.8。要在另一个模型上自动重新运行被拒绝的请求，请考虑使用带 `"default"` 模式的 `fallbacks` 参数（`fallbacks: "default"`），它会根据拒绝类别选择推荐的回退模型，而不是依赖手动维护的模型列表。服务器端回退处于 beta 阶段；`"default"` 模式需要 `server-side-fallback-2026-07-01` beta 头。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。

2. **在对话中途更改工具（beta）：** 您可以在对话的轮次之间添加或移除工具，而不会使较早轮次的[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)命中失效。请发送 beta 头 `mid-conversation-tool-changes-2026-07-01`。这对于随任务推进逐步公开工具或淘汰工具的智能体工作负载很有用；如果不使用它，更改后的工具列表会使缓存的前缀失效。

3. **重新调整长度和详细程度提示：** Claude Opus 5 上默认的可见响应和书面交付物比早期 Opus 模型上更长，而降低 effort 会减少思考量，但不能可靠地缩短可见响应。请改为明确提示要求简洁或目标长度。请参阅[响应长度和详细程度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#response-length-and-verbosity)和[书面交付物长度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#written-deliverable-length)。

4. **移除沿用的验证指令并限制范围：** Claude Opus 5 无需指示即会验证自己的工作，因此请移除从为早期模型调优的提示中沿用下来的显式验证或自检指令；保留它们会导致过度验证。对于范围狭窄的任务，请明确限制任务范围。在多智能体框架中，请就哪些场景需要委派给出明确指导，或限制子智能体的数量，因为 Claude Opus 5 比早期模型更容易进行委派。请参阅[任务范围与过度验证](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#task-scope-and-over-verification)和[控制子智能体生成](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#controlling-subagent-spawning)。

### 迁移检查清单

* 将模型名称从 `claude-opus-4-7` 更新为 `claude-opus-5`（或更新别名）。
* 审查不带 `thinking` 字段运行的工作负载：它们在 Claude Opus 5 上会开启思考运行。重新审视 `max_tokens`（它仍然是总输出（思考加响应文本）的硬性上限），或在 effort 为 `high` 或更低时传递 `thinking: {type: "disabled"}` 以保留旧行为。如果您禁用思考，请查看[在禁用思考的情况下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)，了解可能出现的输出瑕疵及其提示缓解措施。
* 更新按位置读取内容的响应解析代码，例如 `content[0].text` 或假定第一个内容块为文本的流处理程序：开启思考后，`thinking` 块会在 `text` 块之前到达。请改为按 `type` 选择内容块。
* 如果您运行工具使用循环，请在返回工具结果时将 `thinking` 块完整且未经修改地传回；修改过的块会返回 400 错误。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。
* 验证任何解析 `thinking` 字段的代码仅将其视为显示文本。`thinking.display` 在 Claude Opus 5 上默认为 `"omitted"`，与 Claude Opus 4.7 相同，因此思考块到达时 `thinking` 字段为空；设置 `display: "summarized"` 以接收可读摘要。请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。
* 审查禁用思考的请求：`thinking: {type: "disabled"}` 与 effort `xhigh` 或 `max` 组合会返回 400 错误，并在每个请求上强制执行。请重新启用思考或将 effort 降低到 `high` 或更低。
* 如果您在 Opus 4.7 迁移期间已移除采样参数，则无需采取任何操作。如果您通过 400 重试路径重新添加了它们，请移除该重试路径。
* 重新评估您的 `effort` 设置：在您自己的评估上重新进行一次 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 扫描，而不是沿用为 Claude Opus 4.7 调优的设置。将 `low` 和 `medium` effort 作为成本和延迟控制手段进行测试，并在最大能力比令牌开销更重要的场景中测试 `max` effort。如果您以 `xhigh` 或 `max` effort 运行，请将 `max_tokens` 提高到至少 64k 作为起点。
* 移除任何上下文窗口 beta 头。1M 上下文窗口在 Claude API、Amazon Bedrock、Google Cloud 和 Microsoft Foundry 上均为默认值。
* 如果您通过重建对话历史来更新指令，请考虑改用对话中途系统消息以保留提示缓存命中。
* 验证您的停止原因处理逻辑会在拒绝时读取 `stop_details`（自 Claude Opus 4.7 起可用；现已公开记录），并考虑使用 `fallbacks: "default"`（beta）在推荐的回退模型上自动重新运行被拒绝的请求。
* 审查接近缓存最小值的提示：512 令牌或以上的提示现在可以创建缓存条目。
* 如果您使用 [web fetch](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)，请规划替代方案：它在 Claude Opus 5 上不可用。
* 如果您的组织有 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models) 承诺，请注意 Priority Tier 在 Claude Opus 5 上不受支持。
* 如果您在 Claude Opus 4.7 上使用了快速模式，除模型 ID 外无需更改请求：`speed: "fast"` 和 `fast-mode-2026-02-01` beta 头在 Claude Opus 5 上可照常使用，无需更改。
* 对于智能体工作负载，请考虑[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)（beta）和对话中途工具更改（beta）。
* 重新调整长度和详细程度提示，并移除从为早期模型调优的提示中沿用下来的验证和自检指令。
* 在您选择的 effort 级别上重新建立成本和延迟基线。每令牌定价与 Claude Opus 4.7 相比没有变化，但思考令牌按输出令牌计费，因此原本不思考运行的工作负载每个请求可能产生更多输出令牌。

## 从 Claude Opus 4.6 及更早的 Opus 模型迁移到 Claude Opus 5

Claude Opus 5 在现有的 Claude Opus 4.6 提示和评估上应具有强劲的开箱即用性能，且定价相同，但在迁移时有一些值得了解的行为和 API 变更。这些变更大多在 Claude Opus 4.7 中生效；另外两项，即默认开启思考和禁用思考的 effort 上限，在 Claude Opus 5 上生效。本节涵盖了所有这些变更，因此对于直接来自 Claude Opus 4.6 的代码而言是完整的。Claude Opus 5 支持与 Claude Opus 4.6 相同的功能集，包括：

* [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)，按标准 API 定价，无长上下文溢价
* [128k 最大输出令牌](https://platform.claude.com/docs/zh-CN/models/overview)
* [自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)
* [提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)
* [批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)
* [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)
* [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)
* [视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)
* 服务器端和客户端[工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)（[bash](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/bash-tool)、[代码执行](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)、[计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)、[文本编辑器](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/text-editor-tool)、[网页搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)、[MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)、[记忆](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)）

两个例外：[web fetch](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool) 在 Claude Opus 5 上不可用，且 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models) 在 Claude Opus 5 上不受支持。在 Claude API 和 Google Cloud 上，Claude Opus 5 还支持作为稳定版 `computer_toolset_20260801` 工具集的[计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)，以及用于网页内任务的[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)，这两者 Claude Opus 4.6 或更早的 Opus 模型均不支持；基于早期 `computer_20251124` 版本的现有集成在 Claude Opus 5 上可继续正常工作，无需更改。要升级现有集成，请参阅[从 `computer_20251124` 迁移](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#migrate-from-computer-20251124)。

### 更新您的模型名称

```python
# Opus 迁移
model = "claude-opus-4-6"  # Before
model = "claude-opus-5"  # After
```

### 破坏性变更

1. **扩展思考已移除：** `thinking: {type: "enabled", budget_tokens: N}` 在 Claude Opus 4.7 或更高版本的模型上不再受支持，并会返回 400 错误。请切换到 [adaptive thinking（自适应思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)（`thinking: {type: "adaptive"}`），并使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)来控制思考深度。在 Claude Opus 5 上，自适应思考**默认开启**：`thinking: {type: "adaptive"}` 是有效的，并且等同于完全省略 `thinking` 字段（参见下一项）。

   之前（Claude Opus 4.6）：

   <CodeGroup>
     ```bash cURL
     curl https://api.anthropic.com/v1/messages \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -H "content-type: application/json" \
       -d '{
         "model": "claude-opus-4-6",
         "max_tokens": 16000,
         "thinking": {
           "type": "enabled",
           "budget_tokens": 10000
         },
         "messages": [
           {
             "role": "user",
             "content": "..."
           }
         ]
       }'
     ```

     ```bash CLI
     ant messages create <<'YAML'
     model: claude-opus-4-6
     max_tokens: 16000
     thinking:
       type: enabled
       budget_tokens: 10000
     messages:
       - role: user
         content: "..."
     YAML
     ```

     ```python Python
     client.messages.create(
         model="claude-opus-4-6",
         max_tokens=16000,
         thinking={"type": "enabled", "budget_tokens": 10000},
         messages=[{"role": "user", "content": "..."}],
     )
     ```

     ```typescript TypeScript
     await client.messages.create({
       model: "claude-opus-4-6",
       max_tokens: 16000,
       thinking: { type: "enabled", budget_tokens: 10000 },
       messages: [{ role: "user", content: "..." }]
     });
     ```

     ```csharp C#
     using Anthropic;
     using Anthropic.Models.Messages;

     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = "claude-opus-4-6",
         MaxTokens = 16000,
         Thinking = new ThinkingConfigEnabled(budgetTokens: 10000),
         Messages = [new() { Role = Role.User, Content = "..." }]
     };

     var response = await client.Messages.Create(parameters);
     Console.WriteLine(response);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     "claude-opus-4-6",
     	MaxTokens: 16000,
     	Thinking:  anthropic.ThinkingConfigParamOfEnabled(10000),
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("...")),
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response)
     ```

     ```java Java
     AnthropicClient client = AnthropicOkHttpClient.fromEnv();

     MessageCreateParams params = MessageCreateParams.builder()
         .model("claude-opus-4-6")
         .maxTokens(16000L)
         .enabledThinking(10000L)
         .addUserMessage("...")
         .build();

     Message response = client.messages().create(params);
     IO.println(response);
     ```

     ```php PHP
     $client = new Client();

     $message = $client->messages->create(
         maxTokens: 16000,
         messages: [['role' => 'user', 'content' => '...']],
         model: 'claude-opus-4-6',
         thinking: ['type' => 'enabled', 'budget_tokens' => 10000],
     );
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     message = client.messages.create(
       model: "claude-opus-4-6",
       max_tokens: 16000,
       thinking: {
         type: "enabled",
         budget_tokens: 10000
       },
       messages: [
         { role: "user", content: "..." }
       ]
     )
     ```
   </CodeGroup>

   之后（Claude Opus 5）：

   <CodeGroup>
     ```bash cURL
     curl https://api.anthropic.com/v1/messages \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -H "content-type: application/json" \
       -d '{
         "model": "claude-opus-5",
         "max_tokens": 16000,
         "thinking": {
           "type": "adaptive"
         },
         "output_config": {
           "effort": "high"
         },
         "messages": [
           {
             "role": "user",
             "content": "..."
           }
         ]
       }'
     ```

     ```bash CLI
     ant messages create <<'YAML'
     model: claude-opus-5
     max_tokens: 16000
     thinking:
       type: adaptive
     output_config:
       effort: high
     messages:
       - role: user
         content: "..."
     YAML
     ```

     ```python Python
     client.messages.create(
         model="claude-opus-5",
         max_tokens=16000,
         thinking={"type": "adaptive"},
         output_config={"effort": "high"},  # or "max", "xhigh", "medium", "low"
         messages=[{"role": "user", "content": "..."}],
     )
     ```

     ```typescript TypeScript
     await client.messages.create({
       model: "claude-opus-5",
       max_tokens: 16000,
       thinking: { type: "adaptive" },
       output_config: { effort: "high" }, // or "max", "xhigh", "medium", "low"
       messages: [{ role: "user", content: "..." }]
     });
     ```

     ```csharp C#
     using Anthropic;
     using Anthropic.Models.Messages;

     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = "claude-opus-5",
         MaxTokens = 16000,
         Thinking = new ThinkingConfigAdaptive(),
         OutputConfig = new OutputConfig { Effort = Effort.High }, // or Max, Xhigh, Medium, Low
         Messages = [new() { Role = Role.User, Content = "..." }]
     };

     var response = await client.Messages.Create(parameters);
     Console.WriteLine(response);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     "claude-opus-5",
     	MaxTokens: 16000,
     	Thinking: anthropic.ThinkingConfigParamUnion{
     		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
     	},
     	OutputConfig: anthropic.OutputConfigParam{
     		Effort: anthropic.OutputConfigEffortHigh, // or Max, Xhigh, Medium, Low
     	},
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("...")),
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response)
     ```

     ```java Java
     AnthropicClient client = AnthropicOkHttpClient.fromEnv();

     MessageCreateParams params = MessageCreateParams.builder()
         .model("claude-opus-5")
         .maxTokens(16000L)
         .thinking(ThinkingConfigAdaptive.builder().build())
         .outputConfig(OutputConfig.builder()
             .effort(OutputConfig.Effort.HIGH) // or MAX, XHIGH, MEDIUM, LOW
             .build())
         .addUserMessage("...")
         .build();

     Message response = client.messages().create(params);
     IO.println(response);
     ```

     ```php PHP
     $client = new Client();

     $message = $client->messages->create(
         maxTokens: 16000,
         messages: [['role' => 'user', 'content' => '...']],
         model: 'claude-opus-5',
         thinking: ['type' => 'adaptive'],
         outputConfig: ['effort' => 'high'], // or 'max', 'xhigh', 'medium', 'low'
     );
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     message = client.messages.create(
       model: "claude-opus-5",
       max_tokens: 16000,
       thinking: {
         type: "adaptive"
       },
       output_config: {
         effort: "high" # or "max", "xhigh", "medium", "low"
       },
       messages: [
         { role: "user", content: "..." }
       ]
     )
     ```
   </CodeGroup>

   自适应思考可以通过提示和 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)进行引导；请参阅[选择 effort 级别](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#choosing-an-effort-level)。

2. **思考默认开启：** 在 Claude Opus 4.6 和 Claude Opus 4.7 上，不带 `thinking` 字段的请求会在不思考的情况下运行；在 Claude Opus 5 上，相同的请求会以[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)运行。`max_tokens` 仍然是总输出（思考加上响应文本）的硬性限制，因此对于之前不带思考运行的工作负载，请重新审视该值。即使思考文本没有返回给您，思考令牌也会按输出令牌计费，因此尽管每令牌的定价没有变化，之前不带思考运行的工作负载在 Claude Opus 5 上每个请求可能会产生更多的输出令牌；请参阅[成本控制](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost#cost-control)。要保留旧行为，请传递 `thinking: {type: "disabled"}`，但需遵守下一项中的 effort 上限；请注意，在禁用思考的情况下，模型偶尔可能会以纯文本形式发出工具调用，或在其可见输出中包含内部 XML 标签，因此在可能的情况下，请优先选择启用思考的较低 effort 级别；在无法做到的情况下，请参阅[在禁用思考的情况下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)了解缓解措施。

   响应的结构也随之改变。在思考开启的情况下，响应可能在第一个 `text` 块之前以一个或多个 `thinking` 块开头，并且由于思考内容在 Claude Opus 5 上默认被省略（本列表中的第 5 项），这些块到达时其 `thinking` 字段为空，同时附带其 `signature`。按位置读取回复的代码（例如 `content[0].text`，或将第一个 `content_block_start` 事件视为文本的流处理程序）在这些响应上会出错。请改为按 `type` 字段选择内容块：从 `type` 为 `"text"` 的块中读取 `text`，并在处理流事件时根据块类型进行分支。

   如果您运行工具使用循环，请在返回工具结果时将每个助手响应中的 `thinking` 块完整且未经修改地传回 API，包括 `thinking` 字段为空的块。请按接收到的原样回传助手消息，而不是按类型过滤其内容块或重新构建它：API 会以 400 错误拒绝经过编辑、重新排序或部分丢弃的思考块。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。

3. **禁用思考的上限为 `high` effort：** 您可以使用 `thinking: {type: "disabled"}` 关闭思考，但仅限于 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别为 `high` 或更低时。在 Claude Opus 5 上，将 `thinking: {type: "disabled"}` 与 effort `xhigh` 或 `max` 组合的请求会返回 400 错误，并在每个请求上强制执行。在迁移之前，请审查禁用思考的请求：重新启用思考，或将 effort 降低到 `high` 或更低。

4. **采样参数已移除：** 在 Claude Opus 4.7 或更高版本的模型（包括 Claude Opus 5）上，将 `temperature`、`top_p` 或 `top_k` 设置为任何非默认值都会返回 400 错误。Python SDK（v1.0 及更高版本）未定义这些参数，传递它们会引发 `TypeError`。最安全的迁移路径是从请求负载中完全省略这些参数。在 Claude Opus 5 上，提示是引导模型行为的推荐方式。如果您之前使用 `temperature = 0` 来实现确定性，请注意它在之前的模型上也从未保证输出完全相同。

5. **思考内容默认省略：** 在 Claude Opus 4.7 及更高版本的模型上，思考块仍会出现在响应流中，但除非您明确选择启用，否则其 `thinking` 字段为空。这是相对于 Claude Opus 4.6 的一项静默变更，在 Claude Opus 4.6 上默认会返回摘要化的思考文本。要恢复摘要化的思考内容，请将 `thinking.display` 设置为 `"summarized"`：

   <CodeGroup exclude="shell">
     ```python Python
     thinking = {
         "type": "adaptive",
         "display": "summarized",
     }
     ```

     ```typescript TypeScript
     const thinking = {
       type: "adaptive",
       display: "summarized"
     };
     ```

     ```csharp C#
     var thinking = new ThinkingConfigAdaptive { Display = Display.Summarized };
     ```

     ```go Go
     thinking := anthropic.ThinkingConfigParamUnion{
     	OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{
     		Display: anthropic.ThinkingConfigAdaptiveDisplaySummarized,
     	},
     }
     ```

     ```java Java
     ThinkingConfigAdaptive thinking = ThinkingConfigAdaptive.builder()
         .display(ThinkingConfigAdaptive.Display.SUMMARIZED)
         .build();
     ```

     ```php PHP
     $thinking = ['type' => 'adaptive', 'display' => 'summarized'];
     ```

     ```ruby Ruby
     thinking = {
       type: "adaptive",
       display: "summarized"
     }
     ```
   </CodeGroup>

   在 Claude Opus 4.7 及更高版本的模型上，默认值为 `"omitted"`。如果您的产品向用户流式传输推理内容，新的默认值会表现为输出开始前的长时间停顿；请设置 `display: "summarized"` 以恢复思考期间的可见进度。详情请参阅[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)。

6. **更新的令牌计数：** Claude Opus 4.7 引入了新的分词器（tokenizer），后续的 Opus 模型（包括 Claude Opus 5）也使用该分词器。它有助于提升在各类任务上的性能，并且与 Claude Opus 4.7 之前的模型相比，在处理文本时可能会使用大约 1 倍到 1.35 倍的令牌（最多约多 35%，因内容而异）。

   [`/v1/messages/count_tokens`](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting) 对 Claude Opus 5 返回的令牌数与对 Claude Opus 4.6 返回的不同。令牌效率可能因工作负载形态而异。

   提示干预、`task_budget` 和 `effort` 可以帮助控制成本并确保适当的令牌使用量。这些控制手段可能会以模型智能为代价。请更新您的 `max_tokens` 参数以提供额外的余量，包括压缩触发器。Claude Opus 5 以标准 API 定价提供 1M 上下文窗口，无长上下文附加费用。

7. **预填充移除（沿袭自 Opus 4.6）：** 在 Claude Opus 4.7 及更高版本的模型（包括 Claude Opus 5）上，预填充助手消息会返回 400 错误。请改用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)、系统提示指令或 `output_config.format`。

### 选择 effort 级别

[effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)允许您在 Claude 的智能与令牌消耗之间进行调节，以能力换取更快的速度和更低的成本。Claude Opus 5 支持全套 effort 级别，默认为 `high`。请在您自己的评估上重新进行一次 effort 扫描，而不是沿用为早期模型调优的设置：

* **`max`：** 可以在最苛刻的任务上带来收益，但可能因令牌使用量增加而出现收益递减，并且在较简单的任务上容易过度思考。请在最大能力比令牌消耗更重要的场景中测试它。
* **`xhigh`：** 为需要比默认更深入的长时间运行的智能体和编码工作提供扩展能力。
* **`high`：** 默认值。在大多数任务上平衡令牌使用量和智能。
* **`medium`：** 相对于默认值的节省成本的降级选项，值得作为成本和延迟控制手段进行测试。
* **`low`：** 最高效。保留用于简短、范围明确的任务和对延迟敏感的工作负载。

如果您以 `xhigh` 或 `max` effort 运行，请设置较大的 `max_tokens`，以便模型有空间进行思考和行动；从 64k 令牌开始，然后在此基础上调优。对于该模型而言，effort 比以往任何 Opus 都更重要。升级时请积极地对其进行实验。

### 行为变更

Claude Opus 4.7 引入了若干与 Claude Opus 4.6 不同的行为差异，这些差异不是 API 破坏性变更，但可能需要更新提示或移除脚手架。它们延续到 Claude Opus 5，并附带本列表中注明的调整。

1. **响应长度因用例而异：** Claude Opus 4.7 会根据其判断的任务复杂程度来校准响应长度，而不是默认采用固定的详细程度。这通常意味着在简单查询上回答更短，而在开放式分析上回答长得多。

   如果您的产品依赖于特定风格或详细程度的输出，您可能需要调优您的提示。例如，要降低详细程度，请添加："Provide concise, focused responses. Skip non-essential context, and keep examples minimal." 如果您看到特定类型的过度解释，请在提示中添加有针对性的指令来防止它们。

   展示 Claude 如何以适当简洁程度进行沟通的正面示例，往往比负面示例或告诉模型不要做什么的指令更有效。在 Claude Opus 5 上，默认的可见响应和书面交付物比早期 Opus 模型更长，并且降低 effort 会减少思考量，但不能可靠地缩短可见响应；请明确提示要求简洁或目标长度。请参阅[响应长度和详细程度](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#response-length-and-verbosity)。

2. **更字面化的指令遵循：** Claude Opus 4.7 比 Claude Opus 4.6 更字面化、更明确地解释提示，尤其是在较低的 effort 级别下。它不会默默地将一项指令从一个项目泛化到另一个项目，也不会推断您没有提出的请求。这种字面化的好处是精确性和更少的反复折腾。对于具有精心调优的提示、结构化提取以及您希望行为可预测的流水线的 API 用例，它通常表现更好。对于迁移到 Claude Opus 5，进行一次提示和测试框架审查可能特别有帮助。

3. **更直接的语气：** 与任何新模型一样，长篇写作的散文风格可能会发生变化。与 Claude Opus 4.6 更温暖的风格相比，Claude Opus 4.7 更直接、更有主见，较少使用以认可为先的措辞，表情符号也更少。如果您的产品依赖于特定的语气，请根据新的基线重新评估风格提示。

4. **智能体轨迹中内置的进度更新：** Claude Opus 4.7 在长时间的智能体轨迹中会向用户提供更规律、更高质量的更新。如果您添加了脚手架来强制输出中间状态消息（"After every 3 tool calls, summarize progress"），请尝试移除它。如果您发现 Claude Opus 4.7 面向用户的更新的长度或内容与您的用例校准不佳，请在提示中明确描述这些更新应该是什么样子并提供示例。

5. **子智能体生成方式已改变：** Claude Opus 4.7 默认倾向于比 Claude Opus 4.6 生成更少的子智能体，而 Claude Opus 5 比早期模型更容易委派给子智能体。该行为可以通过提示向任一方向引导；请就何时需要子智能体给出明确指导，或限制子智能体的数量。请参阅[控制子智能体生成](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#controlling-subagent-spawning)。

6. **更严格的 effort 校准：** 与 Claude Opus 4.6 相比有显著变化，Claude Opus 4.7 严格遵守 [effort 级别](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)，尤其是在低端。在 `low` 和 `medium` 下，模型会将其工作范围限定在所要求的内容上，而不是做超出要求的事情。

   这对延迟和成本有利，但对于以 `low` effort 运行的中等复杂任务，存在一定的思考不足风险。如果您在复杂问题上观察到浅层推理，请将 effort 提高到 `high` 或 `xhigh`，而不是通过提示来绕过它。

   如果您出于延迟考虑需要将 effort 保持在 `low`，请添加有针对性的指导："This task involves multistep reasoning. Think carefully through the problem before responding." 请参阅 [Claude Opus 4.7 的推荐 effort 级别](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#recommended-effort-levels-for-claude-opus-4-7)。

7. **默认更少的工具调用：** Claude Opus 4.7 倾向于比 Claude Opus 4.6 更少地使用工具，而更多地使用推理。这在大多数情况下会产生更好的结果。

   要增加工具使用，请提高 effort 设置。`high` 或 `xhigh` effort 设置在智能体搜索和编码中表现出明显更多的工具使用。您也可以调整提示，明确指示模型何时以及如何正确使用其工具。

8. **实时网络安全防护措施：** 这是 Claude Opus 4.7 中新增的功能，涉及被禁止或高风险主题的请求可能会导致拒绝。对于渗透测试、漏洞研究或红队演练等合法的安全工作，请申请加入 [Cyber Verification Program](https://support.claude.com/en/articles/14604842-real-time-cyber-safeguards-on-claude-opus-and-sonnet) 以请求降低限制。申请途径取决于您访问 Claude 的方式。

9. **高分辨率图像支持：** Claude Opus 4.7 是第一个支持高分辨率图像的 Claude 模型。最大图像分辨率为长边 2,576 像素，高于之前模型的 1,568 像素。这为视觉密集型工作负载带来了收益，对于计算机使用、屏幕截图理解和文档分析尤其有价值。

   高分辨率支持是自动的，不需要 beta 标头或客户端选择启用。需要规划两件事：

   * 全分辨率图像使用的图像令牌可能比之前的模型多出约 3 倍（每张图像最多 4,784 个令牌，而之前的上限约为每张图像 1,600 个令牌）。请为图像密集型工作负载重新规划 `max_tokens` 和成本预期，或者如果您不需要额外的保真度，请在发送前进行降采样。
   * 在 Claude Opus 4.7 上，模型返回的指向坐标和边界框坐标与实际图像像素为 1:1 对应，因此不需要进行缩放因子转换。

   详情请参阅 [Claude Opus 4.7 上的高分辨率图像支持](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#high-resolution-image-support-on-claude-opus-4-7)。

### 推荐的变更

这些不是必需的，但会改善您的体验：

1. **重新评估 `max_tokens`：** 由于相同的文本在 Claude Opus 4.7 及更高版本的模型上会产生更高的令牌计数，请更新您的 `max_tokens` 参数以提供额外的余量，包括压缩触发器。提示干预、[`task_budget`](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets) 和 [`effort`](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 可以帮助控制成本并确保适当的令牌使用量。

2. **审查令牌计数预期：** 任何在客户端估算令牌或假设固定的令牌与字符比率的代码路径都应针对 Claude Opus 5 重新测试。请使用[令牌计数端点](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)进行验证。

3. **采用[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)（beta）：** Claude Opus 4.7 引入了 task budgets（任务预算）。这些预算让您可以告知 Claude 它在一个完整的智能体循环中有多少令牌可用，包括思考、工具调用、工具结果和最终输出。模型会看到一个持续的倒计时，并利用它来确定工作优先级，并在预算消耗时优雅地完成任务。要使用它，请设置 beta 标头 `task-budgets-2026-03-13`，并将以下内容添加到您的输出配置中：

   <CodeGroup exclude="shell">
     ```python Python
     output_config = {
         "effort": "high",
         "task_budget": {"type": "tokens", "total": 128000},
     }
     ```

     ```typescript TypeScript
     const output_config = {
       effort: "high",
       task_budget: { type: "tokens", total: 128000 }
     };
     ```

     ```csharp C#
     var outputConfig = new BetaOutputConfig
     {
         Effort = Effort.High,
         TaskBudget = new BetaTokenTaskBudget
         {
             Total = 128000,
         },
     };
     ```

     ```go Go
     outputConfig := anthropic.BetaOutputConfigParam{
     	Effort: anthropic.BetaOutputConfigEffortHigh,
     	TaskBudget: anthropic.BetaTokenTaskBudgetParam{
     		Total: 128000,
     	},
     }
     ```

     ```java Java
     BetaOutputConfig outputConfig = BetaOutputConfig.builder()
         .effort(BetaOutputConfig.Effort.HIGH)
         .taskBudget(BetaTokenTaskBudget.builder()
             .total(128000L)
             .build())
         .build();
     ```

     ```php PHP
     $outputConfig = [
         'effort' => 'high',
         'taskBudget' => [
             'type' => 'tokens',
             'total' => 128000,
         ],
     ];
     ```

     ```ruby Ruby
     output_config = {
       effort: :high,
       task_budget: {
         type: :tokens,
         total: 128_000
       }
     }
     ```
   </CodeGroup>

   您可能需要针对您的用例尝试不同的任务预算。如果给模型的任务预算过于严格，它可能会不那么彻底地完成任务，并将其预算作为约束条件提及。

   对于质量比速度更重要的开放式智能体任务，请不要设置任务预算。请将任务预算保留用于您需要模型将其工作范围限定在令牌配额内的工作负载。任务预算的最小值为 20k 令牌。

   任务预算不是硬性上限；它是模型知晓的一项建议。它与 `max_tokens` 不同：

   * **`task_budget`：** 跨整个智能体循环的建议性上限。模型可以看到它并利用它来调整自己的节奏。
   * **`max_tokens`：** 每个请求生成令牌的硬性上限。它不会传递给模型，因此模型不知道它。

   当您希望模型自我调节时使用 `task_budget`，并使用 `max_tokens` 作为限制使用量的硬性上限。

4. **在 `max` 或 `xhigh` effort 下设置较大的 `max_tokens`：** 如果您以 `max` 或 `xhigh` effort 运行 Claude Opus 4.7 或更高版本的模型，请设置较大的最大输出令牌预算，以便模型有空间在其子智能体和工具调用中进行思考和行动。从 64k 令牌开始，然后在此基础上调优。

5. **如果不需要高分辨率，请对图像进行降采样：** Claude Opus 4.7 及更高版本的模型支持最高 2576px / 3.75MP 的图像。高分辨率图像使用更多令牌。如果不需要额外的图像保真度，请在发送给 Claude 之前对图像进行降采样，以避免令牌使用量增加。请参阅[图像和视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)。

6. **考虑自动回退：** Claude Opus 5 附带网络安全安全分类器，其网络类别的拒绝可以回退到 Claude Opus 4.8。要在另一个模型上自动重新运行被拒绝的请求，请考虑使用带有 `"default"` 模式的 `fallbacks` 参数（`fallbacks: "default"`），它会根据拒绝类别选择推荐的回退模型，而不是使用手动维护的模型列表。服务器端回退处于 beta 阶段；`"default"` 模式需要 `server-side-fallback-2026-07-01` beta 标头。请参阅[拒绝和回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。

7. **缓存更短的提示：** Claude Opus 5 上可缓存的最小提示长度为 512 个令牌，低于早期的 Opus 模型。之前因太短而无法缓存的提示现在可以创建缓存条目，无需更改代码。有关各模型的最小值，请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#cache-limitations)。

8. **在对话中途更改工具（beta）：** 您可以在对话的轮次之间添加或移除工具，而不会使早期轮次的[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)命中失效。请发送 beta 标头 `mid-conversation-tool-changes-2026-07-01`。这对于随着任务推进逐步公开工具或淘汰工具的智能体工作负载很有用；如果没有它，更改后的工具列表会使缓存的前缀失效。

9. **移除沿用的验证指令并限定范围：** Claude Opus 5 无需被告知就会验证自己的工作，因此请移除从为早期模型调优的提示中沿用下来的明确验证或自检指令；保留它们会导致过度验证。对于范围狭窄的任务，请明确限定任务范围。请参阅[任务范围和过度验证](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#task-scope-and-over-verification)。

### 迁移清单

* 将模型名称从 `claude-opus-4-6` 更新为 `claude-opus-5`（或更新别名）。
* 从请求负载中移除 `temperature`、`top_p` 和 `top_k`。
* 将 `thinking: {type: "enabled", budget_tokens: N}` 替换为 `thinking: {type: "adaptive"}` 加上 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)，或完全移除 `thinking` 字段；自适应思考在 Claude Opus 5 上默认开启。
* 审查之前不带 `thinking` 字段运行的工作负载：它们在 Claude Opus 5 上会带思考运行。重新审视 `max_tokens`（它仍然是总输出（思考加上响应文本）的硬性限制），或在 effort 为 `high` 或更低时传递 `thinking: {type: "disabled"}` 以保留旧行为。
* 更新按位置读取内容的响应解析代码，例如 `content[0].text` 或假设第一个内容块是文本的流处理程序：在思考开启的情况下，`thinking` 块会在 `text` 块之前到达。请改为按 `type` 选择内容块。
* 如果您运行工具使用循环，请在返回工具结果时将 `thinking` 块完整且未经修改地传回；修改过的块会返回 400 错误。请参阅[保留思考块](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserving-thinking-blocks)。
* 审查禁用思考的请求：`thinking: {type: "disabled"}` 与 effort `xhigh` 或 `max` 组合会返回 400 错误，并在每个请求上强制执行。重新启用思考，或将 effort 降低到 `high` 或更低。
* 移除所有助手消息预填充。
* 如果您的 UI 显示思考内容，请明确选择启用思考摘要。
* 在更新的分词方式下重新对端到端成本和延迟进行基准测试；思考令牌按输出令牌计费，因此之前不带思考运行的工作负载每个请求也可能产生更多的输出令牌。
* 重新调优 `max_tokens` 以适应更新的分词方式。
* 重新测试所有客户端令牌计数估算。
* 如果您的应用程序发送图像，请为[高分辨率图像支持](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#high-resolution-image-support-on-claude-opus-4-7)重新规划预算（每张全分辨率图像的图像令牌最多约多出 3 倍）。如果您不需要额外的保真度，请在发送前进行降采样。
* 如果您使用模型返回的指向坐标或边界框坐标，请移除所有缩放因子转换；在 Claude Opus 4.7 及更高版本的模型上，坐标与实际图像像素为 1:1 对应。
* 针对行为变更审查提示（响应长度、字面化、语气、进度更新、子智能体、effort 校准、工具触发、网络安全防护措施、高分辨率图像处理）。
* 在移除现有长度控制提示的情况下重新确定响应长度基线，然后进行明确调优。
* 如果使用 `xhigh` 或 `max` effort，请将 `max_tokens` 提高到至少 64k 作为起点。
* 考虑为智能体工作流采用任务预算（beta）和对话中途工具更改（beta）。
* 处理 `stop_reason: "refusal"`，并考虑使用 `fallbacks: "default"`（beta）在推荐的回退模型上自动重新运行被拒绝的请求。
* 审查接近缓存最小值的提示：512 个令牌或更多的提示现在可以在 Claude Opus 5 上创建缓存条目。
* 如果您使用 [web fetch](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)，请规划替代方案：它在 Claude Opus 5 上不可用。
* 如果您的组织有 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models) 承诺，请注意 Priority Tier 在 Claude Opus 5 上不受支持。
* 移除从为早期模型调优的提示中沿用下来的验证和自检指令；它们会在 Claude Opus 5 上导致过度验证。
* 如果您的产品从事合法的安全工作，请申请加入 [Cyber Verification Program](https://support.claude.com/en/articles/14604842-real-time-cyber-safeguards-on-claude-opus-and-sonnet)，以获得对网络安全内容更低限制的访问权限。

### 从 Claude Opus 4.5 或更早版本迁移

如果您要从 Claude Opus 4.5、Opus 4.1 或更早的模型直接迁移到 Claude Opus 5，请应用**本节前面的所有变更**，以及以下在 Opus 4.5 和 Opus 4.7 之间生效的累积变更。如果您要从 Opus 4.6 迁移，本节前面的变更就是您所需的全部内容。

#### 更新您的模型名称

```python
# Opus 迁移
model = "claude-opus-4-5"  # Before
model = "claude-opus-5"  # After
```

#### 破坏性变更

1. **预填充移除**已在[从 Claude Opus 4.6 迁移的破坏性变更](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#opus-46-breaking-changes)中涵盖。

2. **工具参数引号处理：** Claude Opus 4.6 及更高版本的模型在工具调用参数中可能会产生略有不同的 JSON 字符串转义（例如，对 Unicode 转义或正斜杠转义的不同处理）。如果您将工具调用的 `input` 作为原始字符串解析而不是使用 JSON 解析器，请验证您的解析逻辑。标准 JSON 解析器（例如 `json.loads()` 或 `JSON.parse()`）会自动处理这些差异。

#### 推荐的变更

这些变更会改善您在 Claude Opus 4.7 及更高版本模型上的体验。标记为 **（在 Opus 4.7 上必需）** 的项目在 Opus 4.6 发布时是可选建议，但现在是强制性的；其余项目仍为推荐。

1. **迁移到自适应思考（在 Opus 4.7 上必需）：** `thinking: {type: "enabled", budget_tokens: N}` 在 Claude Opus 4.7 及更高版本的模型上会返回 400 错误。请切换到 `thinking: {type: "adaptive"}` 并使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)来控制思考深度；在 Claude Opus 5 上，`thinking: {type: "adaptive"}` 等同于省略 `thinking` 字段，后者默认以自适应思考运行。请参阅[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。

   <CodeGroup>
     ```bash cURL
     curl -sS https://api.anthropic.com/v1/messages \
       -H "content-type: application/json" \
       -H "x-api-key: $ANTHROPIC_API_KEY" \
       -H "anthropic-version: 2023-06-01" \
       -d '{
         "model": "claude-opus-5",
         "max_tokens": 16000,
         "thinking": {"type": "adaptive"},
         "output_config": {"effort": "high"},
         "messages": [{"role": "user", "content": "Your prompt here"}]
       }'
     ```

     ```python Before
     response = client.beta.messages.create(
         model="claude-opus-4-5",
         max_tokens=16000,
         thinking={"type": "enabled", "budget_tokens": 32000},
         betas=["interleaved-thinking-2025-05-14"],
         messages=[{"role": "user", "content": "Your prompt here"}],
     )
     ```

     ```python After
     response = client.messages.create(
         model="claude-opus-5",
         max_tokens=16000,
         thinking={"type": "adaptive"},
         output_config={"effort": "high"},
         messages=[{"role": "user", "content": "Your prompt here"}],
     )
     ```

     ```bash CLI
     ant messages create <<'YAML'
     model: claude-opus-5
     max_tokens: 16000
     thinking:
       type: adaptive
     output_config:
       effort: high
     messages:
       - role: user
         content: Your prompt here
     YAML
     ```

     ```typescript TypeScript
     const client = new Anthropic();

     const response = await client.messages.create({
       model: "claude-opus-5",
       max_tokens: 16000,
       thinking: { type: "adaptive" },
       output_config: { effort: "high" },
       messages: [{ role: "user", content: "Your prompt here" }]
     });
     ```

     ```csharp C#
     using Anthropic;
     using Anthropic.Models.Messages;

     AnthropicClient client = new();

     var parameters = new MessageCreateParams
     {
         Model = Model.ClaudeOpus5,
         MaxTokens = 16000,
         Thinking = new ThinkingConfigAdaptive(),
         OutputConfig = new OutputConfig { Effort = Effort.High },
         Messages = [new() { Role = Role.User, Content = "Your prompt here" }]
     };

     var response = await client.Messages.Create(parameters);
     Console.WriteLine(response);
     ```

     ```go Go
     client := anthropic.NewClient()

     response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
     	Model:     anthropic.ModelClaudeOpus5,
     	MaxTokens: 16000,
     	Thinking: anthropic.ThinkingConfigParamUnion{
     		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
     	},
     	OutputConfig: anthropic.OutputConfigParam{
     		Effort: anthropic.OutputConfigEffortHigh,
     	},
     	Messages: []anthropic.MessageParam{
     		anthropic.NewUserMessage(anthropic.NewTextBlock("Your prompt here")),
     	},
     })
     if err != nil {
     	log.Fatal(err)
     }
     fmt.Println(response)
     ```

     ```java Java
     import com.anthropic.models.messages.OutputConfig;
     import com.anthropic.models.messages.ThinkingConfigAdaptive;
     // ...
     public class AdaptiveThinkingExample {
         public static void main(String[] args) {
             AnthropicClient client = AnthropicOkHttpClient.fromEnv();

             MessageCreateParams params = MessageCreateParams.builder()
                 .model(Model.CLAUDE_OPUS_5)
                 .maxTokens(16000L)
                 .thinking(ThinkingConfigAdaptive.builder().build())
                 .outputConfig(OutputConfig.builder()
                     .effort(OutputConfig.Effort.HIGH)
                     .build())
                 .addUserMessage("Your prompt here")
                 .build();

             Message response = client.messages().create(params);
             System.out.println(response);
         }
     }
     ```

     ```php PHP
     $client = new Client();

     $response = $client->messages->create(
         maxTokens: 16000,
         messages: [['role' => 'user', 'content' => 'Your prompt here']],
         model: 'claude-opus-5',
         thinking: ['type' => 'adaptive'],
         outputConfig: ['effort' => 'high'],
     );
     ```

     ```ruby Ruby
     client = Anthropic::Client.new

     response = client.messages.create(
       model: "claude-opus-5",
       max_tokens: 16000,
       thinking: { type: "adaptive" },
       output_config: { effort: "high" },
       messages: [{ role: "user", content: "Your prompt here" }]
     )
     ```
   </CodeGroup>

   请注意，此迁移还会从 `client.beta.messages.create` 迁移到 `client.messages.create`。自适应思考和 effort 不需要 beta SDK 命名空间或任何 beta 标头。

2. **移除 effort beta 标头：** effort 参数不需要 beta 标头。请从您的请求中移除 `betas=["effort-2025-11-24"]`。

3. **移除细粒度工具流式传输 beta 标头：** 细粒度工具流式传输不需要 beta 标头。请从您的请求中移除 `betas=["fine-grained-tool-streaming-2025-05-14"]`。

4. **移除交错思考 beta 标头：** 自适应思考会在 Claude Opus 4.7、Opus 4.6 和 Sonnet 4.6 上自动启用交错思考。请从您的请求中移除 `betas=["interleaved-thinking-2025-05-14"]`。该标头在使用手动扩展思考的 Sonnet 4.6 上仍然有效，但手动模式已弃用。

5. **迁移到 output\_config.format：** 如果使用结构化输出，请将 `output_format={...}` 更新为 `output_config={"format": {...}}`。API 仍然接受已弃用的 `output_format` 参数，但它将在未来的模型版本中被移除。Python SDK（v1.0 及更高版本）在 `client.beta.messages.create()` 或 `count_tokens()` 上不接受 `output_format={...}`。`parse()` 和 `stream()` 辅助函数的 `output_format=Model` 参数保持不变。

### 从 Claude 4.1 或更早版本迁移

如果您要从 Opus 4.1 或更早的模型直接迁移到 Claude Opus 5，请应用本节前面的所有变更，以及本小节中的附加变更。

```python
# 来自 Opus 4.1
model = "claude-opus-4-1-20250805"  # Before
model = "claude-opus-5"  # After

# 来自 Sonnet 3.7
model = "claude-3-7-sonnet-20250219"  # Before
model = "claude-opus-5"  # After
```

#### 附加的破坏性变更

1. **移除采样参数**

   <Warning>
     从 Claude 3.x 模型迁移时，这是一项破坏性变更。
   </Warning>

   从 Claude Opus 4.7 开始，将 `temperature`、`top_p` 或 `top_k` 设置为任何非默认值都会返回 400 错误。Python SDK（v1.0 及更高版本）未定义这些参数，传递它们会引发 `TypeError`。最安全的迁移路径是从请求中完全省略这些参数，并使用提示来引导模型的行为。如果您之前使用 `temperature = 0` 来实现确定性，请注意它从未保证输出完全相同。

   <CodeGroup exclude="shell">
     ```python Python
     # 之前 - 这在 Claude 4+ 模型中会报错
     response = client.messages.create(
         model="claude-3-7-sonnet-20250219",
         temperature=0.7,
         top_p=0.9,  # Non-default sampling params return 400 on Opus 4.7
         # ...
     )

     # 之后
     response = client.messages.create(
         model="claude-opus-5",
         # ...
     )
     ```

     ```typescript TypeScript
     // 之前 - 这在 Claude 4+ 模型中会报错
     await client.messages.create({
       model: "claude-3-7-sonnet-20250219",
       temperature: 0.7,
       top_p: 0.9 // Non-default sampling params return 400 on Opus 4.7
       // ...
     });

     // 之后
     await client.messages.create({
       model: "claude-opus-5"
       // ...
     });
     ```

     ```csharp C#
     // 之前 - 这在 Claude 4+ 模型中会报错
     await client.Messages.Create(new MessageCreateParams
     {
         Model = "claude-3-7-sonnet-20250219",
         Temperature = 0.7,
         TopP = 0.9, // Non-default sampling params return 400 on Opus 4.7
         // ...
     });

     // 之后
     await client.Messages.Create(new MessageCreateParams
     {
         Model = "claude-opus-5",
         // ...
     });
     ```

     ```go Go
     // 之前 - 这在 Claude 4+ 模型中会报错
     client.Messages.New(ctx, anthropic.MessageNewParams{
     	Model:       "claude-3-7-sonnet-20250219",
     	Temperature: anthropic.Float(0.7),
     	TopP:        anthropic.Float(0.9), // Non-default sampling params return 400 on Opus 4.7
     	// ...
     })

     // 之后
     client.Messages.New(ctx, anthropic.MessageNewParams{
     	Model: "claude-opus-5",
     	// ...
     })
     ```

     ```java Java
     // 之前 - 这在 Claude 4+ 模型中会报错
     client.messages().create(MessageCreateParams.builder()
         .model("claude-3-7-sonnet-20250219")
         .temperature(0.7)
         .topP(0.9) // Non-default sampling params return 400 on Opus 4.7
         // ...
         .build());

     // 之后
     client.messages().create(MessageCreateParams.builder()
         .model("claude-opus-5")
         // ...
         .build());
     ```

     ```php PHP
     // 之前 - 这在 Claude 4+ 模型中会报错
     $client->messages->create(
         model: 'claude-3-7-sonnet-20250219',
         temperature: 0.7,
         topP: 0.9, // Non-default sampling params return 400 on Opus 4.7
         // ...
     );

     // 之后
     $client->messages->create(
         model: 'claude-opus-5',
         // ...
     );
     ```

     ```ruby Ruby
     # 之前 - 这在 Claude 4+ 模型中会报错
     client.messages.create(
       model: "claude-3-7-sonnet-20250219",
       temperature: 0.7,
       top_p: 0.9, # Non-default sampling params return 400 on Opus 4.7
       # ...
     )

     # 之后
     client.messages.create(
       model: "claude-opus-5",
       # ...
     )
     ```
   </CodeGroup>

2. **更新工具版本**

   <Warning>
     从 Claude 3.x 模型迁移时，这是一项破坏性变更。
   </Warning>

   请更新到最新的工具版本。移除所有使用 `undo_edit` 命令的代码。

   <CodeGroup exclude="shell">
     ```python Python
     # 之前
     tools = [{"type": "text_editor_20250124", "name": "str_replace_editor"}]

     # 之后
     tools = [{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}]
     ```

     ```typescript TypeScript
     // 之前
     const legacyTools = [{ type: "text_editor_20250124", name: "str_replace_editor" }];

     // 之后
     const tools = [{ type: "text_editor_20250728", name: "str_replace_based_edit_tool" }];
     ```

     ```csharp C#
     var parameters = new MessageCreateParams
     {
         // 之前：{"type": "text_editor_20250124", "name": "str_replace_editor"}
         // 之后：
         Tools = [new ToolTextEditor20250728()],
         // ...
     };
     ```

     ```go Go
     params := anthropic.MessageNewParams{
     	// 之前：{"type": "text_editor_20250124", "name": "str_replace_editor"}
     	// 之后：
     	Tools: []anthropic.ToolUnionParam{
     		{OfTextEditor20250728: &anthropic.ToolTextEditor20250728Param{}},
     	},
     	// ...
     }
     ```

     ```java Java
     MessageCreateParams params = MessageCreateParams.builder()
         // 之前：{"type": "text_editor_20250124", "name": "str_replace_editor"}
         // 之后：
         .addTool(ToolTextEditor20250728.builder().build())
         // ...
         .build();
     ```

     ```php PHP
     $message = $client->messages->create(
         // 之前：['type' => 'text_editor_20250124', 'name' => 'str_replace_editor']
         // 之后：
         tools: [new ToolTextEditor20250728()],
         // ...
     );
     ```

     ```ruby Ruby
     # 之前
     legacy_tools = [{type: "text_editor_20250124", name: "str_replace_editor"}]

     # 之后
     tools = [{type: "text_editor_20250728", name: "str_replace_based_edit_tool"}]
     ```
   </CodeGroup>

   * **文本编辑器：** 使用 `text_editor_20250728` 和 `str_replace_based_edit_tool`。详情请参阅[文本编辑器工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/text-editor-tool)文档。
   * **代码执行：** 升级到 `code_execution_20260521`。有关迁移说明，请参阅[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#upgrade-to-latest-tool-version)文档。

3. **处理 `refusal` 停止原因**

   更新您的应用程序以[处理 `refusal` 停止原因](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/handle-streaming-refusals)：

   <CodeGroup exclude="shell">
     ```python Python
     response = client.messages.create(...)

     if response.stop_reason == "refusal":
         # 妥善处理拒绝情况
         pass
     ```

     ```typescript TypeScript
     const response = await client.messages.create(/* ... */);

     if (response.stop_reason === "refusal") {
       // 妥善处理拒绝情况
     }
     ```

     ```csharp C#
     var response = await client.Messages.Create(...);

     if (response.StopReason?.Value() == StopReason.Refusal)
     {
         // 妥善处理拒绝情况
     }
     ```

     ```go Go
     response, _ := client.Messages.New(ctx, params) // your existing request

     if response.StopReason == anthropic.StopReasonRefusal {
     	// 妥善处理拒绝情况
     }
     ```

     ```java Java
     Message response = client.messages().create(...);

     StopReason reason = response.stopReason().orElse(StopReason.END_TURN);
     if (reason.equals(StopReason.REFUSAL)) {
         // 妥善处理拒绝情况
     }
     ```

     ```php PHP
     $response = $client->messages->create(...);

     if ($response->stopReason === 'refusal') {
         // 适当处理拒绝情况
     }
     ```

     ```ruby Ruby
     response = client.messages.create(...)

     if response.stop_reason == :refusal
       # 适当处理拒绝情况
     end
     ```
   </CodeGroup>

4. **处理 `model_context_window_exceeded` 停止原因**

   当生成因达到上下文窗口限制（而非所请求的 `max_tokens` 限制）而停止时，Claude 4.5+ 模型会返回 `model_context_window_exceeded` 停止原因。请更新您的应用程序以处理这一新的停止原因：

   <CodeGroup exclude="shell">
     ```python Python
     response = client.messages.create(...)

     if response.stop_reason == "model_context_window_exceeded":
         # 妥善处理上下文窗口限制
         pass
     ```

     ```typescript TypeScript
     const response = await client.messages.create(/* ... */);

     if (response.stop_reason === "model_context_window_exceeded") {
       // 妥善处理上下文窗口限制
     }
     ```

     ```csharp C#
     var response = await client.Messages.Create(...);

     if (response.StopReason?.Raw() == "model_context_window_exceeded")
     {
         // 妥善处理上下文窗口限制
     }
     ```

     ```go Go
     response, _ := client.Messages.New(ctx, params) // your existing request

     if response.StopReason == "model_context_window_exceeded" {
     	// 妥善处理上下文窗口限制
     }
     ```

     ```java Java
     Message response = client.messages().create(...);

     StopReason reason = response.stopReason().orElse(StopReason.END_TURN);
     if (reason.equals(StopReason.of("model_context_window_exceeded"))) {
         // 妥善处理上下文窗口限制
     }
     ```

     ```php PHP
     $response = $client->messages->create(...);

     if ($response->stopReason === 'model_context_window_exceeded') {
         // 妥善处理上下文窗口限制
     }
     ```

     ```ruby Ruby
     response = client.messages.create(...)

     if response.stop_reason == :model_context_window_exceeded
       # 妥善处理上下文窗口限制
     end
     ```
   </CodeGroup>

5. **验证工具参数处理（尾随换行符）**

   Claude 4.5+ 模型会保留工具调用字符串参数中之前被去除的尾随换行符。如果您的工具依赖于对工具调用参数的精确字符串匹配，请验证您的逻辑能正确处理尾随换行符。

6. **针对行为变更更新您的提示**

   Claude 4+ 模型具有更简洁、更直接的沟通风格，并且需要明确的指示。请查阅[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)以获取优化指导。

#### 附加的推荐变更

* **移除旧版 beta 标头：** 移除 `token-efficient-tools-2025-02-19` 和 `output-128k-2025-02-19`。所有 Claude 4+ 模型都内置了令牌高效的工具使用，这些标头没有任何效果。

### 迁移清单（从 Claude Opus 4.5 或更早版本迁移）

* 将模型 ID 更新为 `claude-opus-5`
* 应用所有[从 Claude Opus 4.6 迁移的破坏性变更](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#opus-46-breaking-changes)（移除扩展思考、默认开启思考、禁用思考时的 effort 上限、移除采样参数、默认省略思考显示、更新分词方式）
* **破坏性变更：** 移除助手消息预填充（会返回 400 错误）；请改用结构化输出或 `output_config.format`
* **在 Opus 4.7 上为破坏性变更：** 将 `thinking: {type: "enabled", budget_tokens: N}` 替换为 `thinking: {type: "adaptive"}` 并配合 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)（在 Opus 4.7 上会返回 400）
* 确认工具调用的 JSON 解析使用标准 JSON 解析器
* 移除 `effort-2025-11-24` beta 头（effort 参数不需要它）
* 移除 `fine-grained-tool-streaming-2025-05-14` beta 头
* 移除 `interleaved-thinking-2025-05-14` beta 头（自适应思考会自动启用交错思考）
* 将 `output_format` 迁移到 `output_config.format`（如适用）
* 如果从 Claude 4.1 或更早版本迁移：移除 `temperature`、`top_p` 和 `top_k`（非默认值在 Opus 4.7 上会返回 400）
* 如果从 Claude 4.1 或更早版本迁移：更新工具版本（`text_editor_20250728`、`code_execution_20260521`）
* 如果从 Claude 4.1 或更早版本迁移：处理 `refusal` 停止原因
* 如果从 Claude 4.1 或更早版本迁移：处理 `model_context_window_exceeded` 停止原因
* 如果从 Claude 4.1 或更早版本迁移：确认工具字符串参数对尾部换行符的处理
* 如果从 Claude 4.1 或更早版本迁移：移除旧版 beta 头（`token-efficient-tools-2025-02-19`、`output-128k-2025-02-19`）
* 按照[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)审查并更新提示
* 在部署到生产环境之前先在开发环境中测试

## 从 Claude Sonnet 5 迁移到 Claude Opus 5

Claude Opus 5 和 Claude Sonnet 5 共享相同的 API 接口：两者都默认开启 [adaptive thinking（自适应思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)，两者在 Claude API 和 Claude Code 上都将 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)默认设为 `high`，两者都默认提供 [100 万令牌的 context window（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)以及 [128k 最大输出令牌](https://platform.claude.com/docs/zh-CN/models/overview)，并且两者都不支持 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models)。手动扩展思考和非默认采样参数在两个模型上都会返回 400 错误，助手预填充也是如此。

### 更新您的模型名称

```python
model = "claude-sonnet-5"  # Before
model = "claude-opus-5"  # After
```

### 变更内容

1. **定价：** Claude Opus 5 的定价为每百万输入令牌 5 美元，每百万输出令牌 25 美元。Claude Sonnet 5 的定价为每百万输入/输出令牌 2/10 美元。完整定价请参阅 [Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

2. **禁用思考的 effort 上限为 `high`：** 在 Claude Sonnet 5 上，`thinking: {type: "disabled"}` 在任何 effort 级别下都被接受。在 Claude Opus 5 上，仅当 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别为 `high` 或更低时才被接受；将 `thinking: {type: "disabled"}` 与 effort `xhigh` 或 `max` 组合的请求会返回 400 错误，并在每个请求上强制执行。请在迁移之前审查禁用思考的请求。

3. **对话中途的系统消息：** Claude Opus 5 接受在 `messages` 数组中紧跟用户轮次之后的 `role: "system"` 消息（须遵守[放置规则](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)）。此功能在 Claude Sonnet 5 上不可用。如果您维护着通过重建完整消息历史来更新指令的代码路径，您可以简化它们，并保留较早轮次上的 [prompt cache（提示缓存）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)命中。

4. **Web fetch 不可用：** [web fetch](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool) 工具在 Claude Sonnet 5 上可用，但在 Claude Opus 5 上不可用。

### 迁移清单

* 将模型名称从 `claude-sonnet-5` 更新为 `claude-opus-5`。
* 审查禁用思考的请求：`thinking: {type: "disabled"}` 与 effort `xhigh` 或 `max` 组合在 Claude Opus 5 上会返回 400 错误。请重新启用思考，或将 effort 降低到 `high` 或更低。
* 如果您使用 [web fetch](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)，请规划替代方案：它在 Claude Opus 5 上不可用。
* 针对 Claude Opus 5 重新运行[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)，而不是复用针对 Claude Sonnet 5 测得的计数，并在您自己的工作负载上重新建立成本和延迟基线；每令牌定价有所不同。
