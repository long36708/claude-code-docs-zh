---
title: Claude Sonnet 5 的新功能
url: https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5
description: Claude Sonnet 5 中新功能和行为变更的概述。
---

Claude Sonnet 5 是 Anthropic Sonnet 模型系列的下一代产品。它是 Claude Sonnet 4.6 的直接替换升级，包含三项行为变更：[adaptive thinking（自适应思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)默认开启；手动 extended thinking（扩展思考）现在会返回 400 错误（它在 Claude Sonnet 4.6 上已被弃用）；将采样参数（`temperature`、`top_p`、`top_k`）设置为非默认值会返回 400 错误。本页总结了发布时的所有新内容，包括新的 tokenizer（分词器）。

## 新模型

| 模型              | API 模型 ID         | 描述         |
| --------------- | ----------------- | ---------- |
| Claude Sonnet 5 | `claude-sonnet-5` | 速度与智能的最佳结合 |

Claude Sonnet 5 默认支持 [1M 令牌的 context window（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)（1M 令牌既是默认值也是最大值；没有更小的上下文变体）、128k 最大输出令牌、[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)，以及与 Claude Sonnet 4.6 相同的工具和平台功能集，但 [Priority Tier（优先级层）](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models)除外，该功能在 Claude Sonnet 5 上不可用。在 Claude API 和 Google Cloud 上，Claude Sonnet 5 还支持[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)以及[计算机使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)的稳定版本 `computer_toolset_20260801`，这两者 Claude Sonnet 4.6 均不支持；较早的 `computer_20251124` 版本在两个模型上仍然可用。要升级现有集成，请参阅[从 `computer_20251124` 迁移](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#migrate-from-computer-20251124)。

有关完整的定价和规格，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。

## 行为变更

### 自适应思考默认开启

在 Claude Sonnet 4.6 上，不带 `thinking` 字段的请求在不进行思考的情况下运行。在 Claude Sonnet 5 上，相同的请求会以[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)运行。要关闭思考，请传入 `thinking: {type: "disabled"}`。由于 `max_tokens` 是总输出（思考加响应文本）的硬性限制，对于在 Claude Sonnet 4.6 上不进行思考运行的工作负载，请重新审视该值。

### 不接受采样参数

将 `temperature`、`top_p` 或 `top_k` 设置为非默认值会返回 400 错误。迁移时请移除这些参数；默认值（或省略该参数）是可接受的。请使用 system prompt（系统提示）指令来引导模型行为。这对 Sonnet 级模型来说是新的约束；相同的约束此前已在 Claude Opus 4.7 上引入。

### 手动扩展思考已移除

手动扩展思考（`thinking: {type: "enabled", budget_tokens: N}`）在 Claude Sonnet 4.6 上已被弃用；在 Claude Sonnet 5 上它已被移除并返回 400 错误，与 Claude Opus 4.8 和 Claude Opus 4.7 上相同。请改用带有 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)的自适应思考。

<CodeGroup exclude="shell">
  ```python Python
  # Claude Sonnet 5 不支持（返回 400）
  thinking = {"type": "enabled", "budget_tokens": 32000}

  # 请改用此方式
  thinking = {"type": "adaptive"}
  ```

  ```typescript TypeScript
  // Claude Sonnet 5 不支持（返回 400）
  const legacyThinking = { type: "enabled", budget_tokens: 32000 };

  // 请改用此方式
  const thinking = { type: "adaptive" };
  ```

  ```csharp C#
  // Claude Sonnet 5 不支持（返回 400）
  var legacyThinking = new ThinkingConfigEnabled(budgetTokens: 32000);

  // 请改用此项
  var thinking = new ThinkingConfigAdaptive();
  ```

  ```go Go
  // Claude Sonnet 5 不支持（返回 400）
  legacyThinking := anthropic.ThinkingConfigParamUnion{
  	OfEnabled: &anthropic.ThinkingConfigEnabledParam{BudgetTokens: 32000},
  }

  // 请改用此方式
  thinking := anthropic.ThinkingConfigParamUnion{
  	OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
  }
  ```

  ```java Java
  // Claude Sonnet 5 不支持（返回 400）
  var legacyThinking = ThinkingConfigEnabled.builder().budgetTokens(32_000L).build();

  // 请改用此方式
  var thinking = ThinkingConfigAdaptive.builder().build();
  ```

  ```php PHP
  // Claude Sonnet 5 不支持（返回 400）
  $thinking = ['type' => 'enabled', 'budget_tokens' => 32000];

  // 请改用此方式
  $thinking = ['type' => 'adaptive'];
  ```

  ```ruby Ruby
  # Claude Sonnet 5 不支持（返回 400）
  legacy_thinking = {type: "enabled", budget_tokens: 32_000}

  # 请改用此方式
  thinking = {type: "adaptive"}
  ```
</CodeGroup>

## 新的分词器

Claude Sonnet 5 使用新的分词器。相同的输入文本产生的令牌数比在 Claude Sonnet 4.6 上大约多 30%。确切的增幅取决于内容。这不是 API 变更：请求、响应和 streaming（流式传输）事件保持相同的结构，无需更改代码。

此变更会影响您以令牌为单位进行度量或预算的所有内容：

* **令牌计数：** 对于相同文本，`usage` 字段和[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)结果比在 Claude Sonnet 4.6 上更高。不要重复使用针对早期模型测得的计数；请针对 Claude Sonnet 5 重新计数。
* **以文本衡量的上下文窗口容量：** 上下文窗口为 1M 令牌，但每个令牌平均覆盖的文本更少，因此相同的窗口容纳的文本比 Claude Sonnet 4.6 上更少。
* **`max_tokens` 预算：** 针对 Claude Sonnet 4.6 调整的输出限制可能会在 Claude Sonnet 5 上截断等效的输出。请重新审视那些设置得接近您预期输出长度的限制。
* **每次请求的成本：** 每令牌定价低于 Claude Sonnet 4.6（请参阅[定价](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5#pricing)），但由于相同文本会产生更多令牌，等效请求的成本不会按相同比例直接下降。

## 从 Claude Sonnet 4.6 继承的 API 约束

<Note>
  此约束与 Claude Sonnet 4.6 相比没有变化。除了三项[行为变更](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5#behavior-changes)（请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5#migration-guide)）之外，已在 Claude Sonnet 4.6 上运行的代码无需其他更改。
</Note>

### 不支持助手消息预填充

预填充助手消息会返回 `400` 错误，与 Claude Sonnet 4.6 相同。请改用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)、系统提示指令或 `output_config.format`。

## 能力提升

Claude Sonnet 5 是相对于 Claude Sonnet 4.6 的能力升级，且价格更低。对于需要比 Claude Sonnet 4.6 更强能力、但又不想迁移到 Opus 级模型的工作负载，它也是一个选择。

相对于 Claude Sonnet 4.6，最大的提升体现在编码和智能体任务上。有关基准测试结果，请参阅 [Anthropic 的透明度中心](https://www.anthropic.com/transparency)。

## 网络安全防护措施

Claude Sonnet 5 是首个具备实时网络安全防护措施的 Sonnet 级模型。涉及被禁止或高风险网络安全主题的请求可能会被拒绝。拒绝会以成功的 HTTP 200 响应返回，并带有 `stop_reason: "refusal"`，而不是错误。有关防护措施会拦截哪些内容，以及合法的安全工作如何申请加入网络验证计划（Cyber Verification Program），请参阅 [Claude Opus 和 Sonnet 上的实时网络防护措施](https://support.claude.com/en/articles/14604842-real-time-cyber-safeguards-on-claude-opus-and-sonnet)。

## 定价

Claude Sonnet 5 的定价为每百万输入令牌 2 美元、每百万输出令牌 10 美元，每令牌定价低于 Claude Sonnet 4.6 的 3 美元/15 美元。由于[新的分词器](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5#new-tokenizer)对相同文本产生的令牌数大约多 30%，与 Claude Sonnet 4.6 相比，等效请求的成本不会按每令牌价格的比例直接下降。确切的差异取决于内容和工作负载形态。

有关完整定价（包括批处理和 prompt caching（提示缓存）费率），请参阅[定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

## 可用性

发布时，Claude Sonnet 5 可在以下平台使用：

* **Claude API：** 对所有客户可用。
* **AWS：** 通过 [Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 和 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 提供。在 Amazon Bedrock 上，Claude Sonnet 5 也可通过 `InvokeModel` API 访问，由与 Claude in Amazon Bedrock 相同的基础设施提供服务。旧版 [Claude on Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)集成不包含 Claude Sonnet 5。
* **Google Cloud：** 通过 [Claude on Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 提供。
* **Microsoft Foundry：** 通过 [Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 提供。

对于签订了 ZDR 协议的组织，Claude Sonnet 5 支持[零数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。

## 迁移指南

Claude Sonnet 5 是 Claude Sonnet 4.6 的直接替换。请更新您的模型 ID：

<CodeGroup exclude="shell">
  ```python Python
  model = "claude-sonnet-4-6"  # Before
  model = "claude-sonnet-5"  # After
  ```

  ```typescript TypeScript
  const legacyModel = "claude-sonnet-4-6"; // Before
  const model = "claude-sonnet-5"; // After
  ```

  ```csharp C#
  var legacyModel = Model.ClaudeSonnet4_6; // Before
  var model = Model.ClaudeSonnet5; // After
  ```

  ```go Go
  // 之前
  legacyModel := anthropic.ModelClaudeSonnet4_6
  // 之后
  model := anthropic.ModelClaudeSonnet5
  ```

  ```java Java
  var legacyModel = Model.CLAUDE_SONNET_4_6; // Before
  var model = Model.CLAUDE_SONNET_5; // After
  ```

  ```php PHP
  $model = 'claude-sonnet-4-6'; // Before
  $model = 'claude-sonnet-5'; // After
  ```

  ```ruby Ruby
  legacy_model = "claude-sonnet-4-6" # Before
  model = "claude-sonnet-5" # After
  ```
</CodeGroup>

然后检查以下内容：

1. **令牌预算和计数：** [新的分词器](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5#new-tokenizer)对相同文本产生的令牌数大约多 30%。确切的增幅取决于内容和工作负载形态。请使用[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)重新计算提示的令牌数，并重新审视那些设置得接近您预期输出长度的 `max_tokens` 限制。
2. **扩展思考：** 如果您仍在设置 `budget_tokens`，请迁移到[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。手动扩展思考（`thinking: {type: "enabled"}`）不受支持，并会返回 400 错误。
3. **采样参数：** 将采样参数（`temperature`、`top_p`、`top_k`）设置为非默认值的请求会返回 400 错误；迁移时请将其移除。工具定义和响应结构没有变化，并且助手消息预填充在 Claude Sonnet 4.6 上就已不受支持。

有关详细信息，请参阅[从 Claude Sonnet 4.6 迁移到 Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide#migrating-from-claude-sonnet-4-6-to-claude-sonnet-5)。

## 后续步骤

<CardGroup>
  <Card title="模型概览" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/overview">
    所有当前 Claude 模型的完整规格和定价。
  </Card>

  <Card title="令牌计数" icon="database" href="https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting">
    在迁移之前，使用新的分词器度量您的提示。
  </Card>

  <Card title="自适应思考" icon="brain" href="https://platform.claude.com/docs/zh-CN/build-with-claude/thinking">
    Claude Sonnet 5 上推荐的思考开启模式。
  </Card>

  <Card title="上下文窗口" icon="sliders" href="https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows">
    1M 令牌上下文窗口的工作原理。
  </Card>

  <Card title="定价" icon="shield" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    完整定价，包括批处理和提示缓存费率。
  </Card>
</CardGroup>
