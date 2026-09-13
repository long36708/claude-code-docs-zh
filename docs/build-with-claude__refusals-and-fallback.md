---
title: 拒绝与回退
url: https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback
description: Claude Fable 和 Claude Opus 模型如何返回分类器拒绝，以及如何在回退模型上重试被拒绝的请求。
---

Claude Fable 5.1、Claude Fable 5 和 Claude Opus 5 包含可以拒绝请求的安全分类器。发生这种情况时，您会收到一个正常的响应（而不是错误），其中带有 `stop_reason: "refusal"`。其 `stop_details.category` 指明了策略领域（请参阅[拒绝的样子](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response)）。通常，您仍然可以通过将同一请求发送到另一个 Claude 模型来获得答案。本页向您展示如何识别拒绝以及如何设置该重试。

当您基于这些模型中的任何一个进行构建，并希望被拒绝的请求自动转交给另一个模型时，请阅读本页。如果您在响应中看到了 `"refusal"` 并想知道接下来该怎么做，本页同样适用。

相关页面：

* [停止原因与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons)：`stop_reason` 值的完整列表。
* [回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)：当您自行构建重试时，如何避免支付两次提示缓存成本。
* [SDK 中间件](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/middleware)：封装了所有这些功能的 SDK 辅助工具。
* [回退与计费 cookbook](https://platform.claude.com/cookbook/fable-5-fallback-billing-guide)：一个完整的端到端示例。

最简单的设置（在 Claude API 上处于 beta 阶段）：将 `fallbacks` 设置为 `"default"`，API 会在 Anthropic 针对该拒绝类别推荐的回退模型上重试被拒绝的请求。对于没有推荐回退的类别，拒绝保持不变。

<CodeGroup>
  ```bash cURL
  curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: server-side-fallback-2026-07-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-fable-5",
      "max_tokens": 1024,
      "fallbacks": "default",
      "messages": [{"role": "user", "content": "Hello, Claude"}]
    }' | jq -r '.model'
  ```

  ```bash CLI
  ant beta:messages create \
    --model claude-fable-5 \
    --max-tokens 1024 \
    --message '{"role":"user","content":"Hello, Claude"}' \
    --fallbacks default \
    --beta server-side-fallback-2026-07-01 \
    --transform model --raw-output
  ```

  ```python Python
  client = Anthropic()

  response = client.beta.messages.create(
      model="claude-fable-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello, Claude"}],
      fallbacks="default",
      betas=["server-side-fallback-2026-07-01"],
  )
  print(response.model)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-fable-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    fallbacks: "default",
    betas: ["server-side-fallback-2026-07-01"]
  });
  console.log(response.model);
  ```

  ```csharp C#
  AnthropicClient client = new();

  BetaMessage response = await client.Beta.Messages.Create(
      new()
      {
          Model = Messages::Model.ClaudeFable5,
          MaxTokens = 1024,
          Messages = [new() { Content = "Hello, Claude", Role = Role.User }],
          Fallbacks = new Default(),
          Betas = [AnthropicBeta.ServerSideFallback2026_07_01],
      }
  );

  Console.WriteLine(response.Model.Raw());
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.Background(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeFable5,
  	MaxTokens: 1024,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello, Claude")),
  	},
  	Fallbacks: anthropic.BetaFallbacksParamOfDefault(),
  	Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaServerSideFallback2026_07_01},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Println(response.Model)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  BetaMessage response = client.beta().messages().create(MessageCreateParams.builder()
      .model(Model.CLAUDE_FABLE_5)
      .maxTokens(1024L)
      .addUserMessage("Hello, Claude")
      .fallbacksDefault()
      .addBeta(AnthropicBeta.SERVER_SIDE_FALLBACK_2026_07_01)
      .build());

  IO.println(response.model().asString());
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      model: 'claude-fable-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello, Claude']],
      fallbacks: 'default',
      betas: ['server-side-fallback-2026-07-01'],
  );

  echo $response->model, PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-fable-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}],
    fallbacks: :default,
    betas: ["server-side-fallback-2026-07-01"]
  )

  puts response.model
  ```
</CodeGroup>

以下各节介绍拒绝响应包含的内容、何时使用服务端或客户端回退，以及各自的计费方式。

## 拒绝的样子

拒绝是一个成功的 HTTP 200 响应，带有 `stop_reason: "refusal"`：

```json
{
  "id": "msg_01XFUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "model": "claude-fable-5",
  "content": [],
  "stop_reason": "refusal",
  "stop_details": {
    "type": "refusal",
    "category": "cyber",
    "explanation": "This request was declined because it could enable cyber harm."
  },
  "usage": {
    "input_tokens": 412,
    "output_tokens": 0
  }
}
```

`stop_details` 对象解释了拒绝的原因：

* **`category`：** 指明触发分类器的策略领域。
* **`explanation`：** 人类可读的描述。该文本并不稳定，因此请直接显示它而不要解析它。
* **`recommended_model`：** 仅在设置了 `fallbacks`（[服务端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)，beta）的请求中出现。当 API 跳过了回退尝试时（例如，回退模型受到速率限制），它会指明一个可直接重试的模型，否则为 `null`。它是一个提示，而非保证。
* 当拒绝未映射到某个命名类别时，`category` 和 `explanation` 均为 `null`。该 `null` 是一个正常的、永久的值，而不是占位符。
* 对于除 `refusal` 之外的所有停止原因，`stop_details` 本身为 `null`。

| `category`               | 含义                                                                                                                |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| `"cyber"`                | 该请求可能助长网络危害，例如恶意软件或漏洞利用开发。良性的网络安全工作也可能触发此类别。                                                                      |
| `"bio"`                  | 该请求可能助长生物危害，例如危险的实验室方法。有益的生命科学工作也可能触发此类别。                                                                         |
| `"frontier_llm"`         | 该请求可能协助开发竞争性 AI 模型，这在 [Anthropic 的商业条款](https://www.anthropic.com/legal/commercial-terms)下受到限制。良性的机器学习工作也可能触发此类别。 |
| `"reasoning_extraction"` | 该请求要求模型在响应文本中复现其内部推理。若要以结构化形式获取推理，请改用[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。  |
| `"general_harms"`        | 该请求属于四个命名类别之外的使用策略领域。良性工作也可能触发此类别。                                                                                |

拒绝可能在任何输出之前到达，也可能在部分输出之后于流式传输中途到达。无论哪种情况，都应将任何部分输出视为不完整并丢弃。

<Note>
  **拒绝的计费方式：** 对于在任何输出之前到达的拒绝，您不会被计费。`content` 为空，令牌计数会出现在 `usage` 中但不收费。该请求仍会计入您的速率限制。流式传输中途的拒绝会按正常费率对输入令牌和已流式传输的输出计费。
</Note>

## 选择回退方式

有三种方式可以在另一个模型上重试被拒绝的请求。合适的方式取决于您的运行环境以及您需要多大的控制权。

| 您的情况                  | 使用                                                                                                                                                                                     | 原因                  |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| Claude API，最简单的设置     | [服务端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)                                                                           | 一个请求，一个响应。API 处理重试。 |
| 任何平台，使用 Anthropic SDK | [SDK 中间件](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#client-side-fallback)                                                                         | 在客户端配置一次。重试自动进行。    |
| 原始 HTTP 或自定义重试逻辑      | [手动重试](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#manual-retry)并配合[回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit) | 完全控制。回退抵扣可降低成本。     |

服务端回退和 SDK 中间件会为您应用回退抵扣。只有当您自行构建重试时，才需要参阅[回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)页面。

## 服务端回退

服务端回退在单次 API 调用内重试被拒绝的请求。在默认模式下，当主模型拒绝且该拒绝类别有推荐的回退时，API 会在 Anthropic 针对该类别推荐的模型上运行同一请求。您也可以改为[自行指定最多三个回退模型](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#naming-your-own-fallback-models)。无论哪种方式，您都会收到一个指明作答模型的响应，因此您的用户在一次往返中即可获得答案。

<Note>
  服务端回退在 Claude API 上处于 beta 阶段。[Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing) 不支持 `fallbacks` 参数（包含该参数的批处理项会作为出错结果返回），并且在 Amazon Bedrock、Google Cloud 或 Microsoft Foundry 上不可用。在这些平台上，请改用[配合 SDK 中间件的客户端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#client-side-fallback)。
</Note>

### 发起请求

将 `fallbacks` 参数设置为字符串 `"default"`，并发送 `server-side-fallback-2026-07-01` beta 标头。然后，API 会应用所请求模型的服务端定义的默认路由，该路由根据分类器报告的拒绝类别选择推荐的回退模型，因此被拒绝的请求可以得到处理，而无需您在推荐发生变化时维护模型列表。

默认路由绝不会因您未选择的模型而引发预先的[超大图像拒绝](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#oversized-image-error)：如果某个路由模型会对标记为 `"oversized_image": "error"` 的图像进行缩放，则该模型会从路由中被剔除，因此被标记的图像绝不会以缩放后的形式被处理。

<CodeGroup>
  ```bash cURL
  curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: server-side-fallback-2026-07-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-fable-5",
      "max_tokens": 1024,
      "fallbacks": "default",
      "messages": [{"role": "user", "content": "Hello, Claude"}]
    }' |
    jq -c '{
      stop_reason,
      model,
      # usage.iterations 中出现 fallback_message 条目表示回退模型已运行；
      # 请结合 stop_reason 确认响应是否由回退模型提供。
      served_by_fallback: (
        any(.usage.iterations[]?; .type == "fallback_message")
        and .stop_reason != "refusal"
      )
    }'
  ```

  ```bash CLI
  ant beta:messages create \
    --model claude-fable-5 \
    --max-tokens 1024 \
    --message '{"role":"user","content":"Hello, Claude"}' \
    --fallbacks default \
    --beta server-side-fallback-2026-07-01 \
    --format json |
    jq -c '{
      stop_reason,
      model,
      # usage.iterations 中出现 fallback_message 条目表示回退模型已运行；
      # 请结合 stop_reason 确认响应是否由回退模型提供。
      served_by_fallback: (
        any(.usage.iterations[]?; .type == "fallback_message")
        and .stop_reason != "refusal"
      )
    }'
  ```

  ```python Python
  client = Anthropic()

  response = client.beta.messages.create(
      model="claude-fable-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello, Claude"}],
      fallbacks="default",
      betas=["server-side-fallback-2026-07-01"],
  )

  # usage.iterations 中出现 fallback_message 条目表示运行了回退模型；
  # 请结合 stop_reason 确认该响应是否由回退模型提供。
  fallback_ran = any(
      iteration.type == "fallback_message"
      for iteration in response.usage.iterations or []
  )
  served_by_fallback = fallback_ran and response.stop_reason != "refusal"

  print(
      json.dumps(
          {
              "stop_reason": response.stop_reason,
              "model": response.model,
              "served_by_fallback": served_by_fallback,
          }
      )
  )
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-fable-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    fallbacks: "default",
    betas: ["server-side-fallback-2026-07-01"]
  });

  // usage.iterations 中出现 fallback_message 条目表示运行了回退模型；
  // 请结合 stop_reason 确认该响应是否由回退模型提供。
  const { stop_reason, model, usage } = response;
  const servedByFallback =
    (usage.iterations ?? []).some((entry) => entry.type === "fallback_message") &&
    stop_reason !== "refusal";

  console.log(
    JSON.stringify({
      stop_reason,
      model,
      served_by_fallback: servedByFallback
    })
  );
  ```

  ```csharp C#
  AnthropicClient client = new();

  var response = await client.Beta.Messages.Create(
      new()
      {
          Model = Messages::Model.ClaudeFable5,
          MaxTokens = 1024,
          Messages =
          [
              new() { Content = "Hello, Claude", Role = Role.User },
          ],
          Fallbacks = new Default(),
          Betas = [AnthropicBeta.ServerSideFallback2026_07_01],
      }
  );

  // A fallback_message entry in usage.iterations means a fallback model ran;
  // pair it with stop_reason to confirm the fallback served the response.
  bool fallbackRan = (response.Usage.Iterations ?? []).Any(iteration =>
      iteration.TryPickBetaFallbackMessageIterationUsage(out _)
  );
  bool servedByFallback =
      fallbackRan && response.StopReason?.Value() != BetaStopReason.Refusal;

  Console.WriteLine(
      JsonSerializer.Serialize(
          new
          {
              stop_reason = response.StopReason?.Raw(),
              model = response.Model.Raw(),
              served_by_fallback = servedByFallback,
          }
      )
  );
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.Background(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeFable5,
  	MaxTokens: 1024,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello, Claude")),
  	},
  	Fallbacks: anthropic.BetaFallbacksParamOfDefault(),
  	Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaServerSideFallback2026_07_01},
  })
  if err != nil {
  	panic(err)
  }

  // usage.iterations 中出现 fallback_message 条目表示运行了回退模型；
  // 请结合 stop_reason 确认该响应是否由回退模型提供。
  fallbackRan := slices.ContainsFunc(
  	response.Usage.Iterations,
  	func(iteration anthropic.BetaIterationsUsageItemUnion) bool {
  		_, isFallback := iteration.AsAny().(anthropic.BetaFallbackMessageIterationUsage)
  		return isFallback
  	},
  )
  servedByFallback := fallbackRan && response.StopReason != anthropic.BetaStopReasonRefusal

  summary, err := json.Marshal(struct {
  	StopReason       anthropic.BetaStopReason `json:"stop_reason"`
  	Model            anthropic.Model          `json:"model"`
  	ServedByFallback bool                     `json:"served_by_fallback"`
  }{response.StopReason, response.Model, servedByFallback})
  if err != nil {
  	panic(err)
  }
  fmt.Println(string(summary))
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  BetaMessage response = client.beta().messages().create(
      MessageCreateParams.builder()
          .model(Model.CLAUDE_FABLE_5)
          .maxTokens(1024L)
          .addUserMessage("Hello, Claude")
          .fallbacksDefault()
          .addBeta(AnthropicBeta.SERVER_SIDE_FALLBACK_2026_07_01)
          .build()
  );

  // 出现 fallback_message 用量条目表示响应由回退模型生成；
  // 出现 refusal 停止原因则表示没有任何模型提供该响应。
  List<BetaUsage.Iteration> iterations =
      response.usage().iterations().orElse(List.of());
  boolean servedByFallback =
      iterations.stream().anyMatch(BetaUsage.Iteration::isFallbackMessage)
          && response.stopReason().filter(BetaStopReason.REFUSAL::equals).isEmpty();

  IO.println("""
      {"stop_reason":"%s","model":"%s","served_by_fallback":%b}\
      """.formatted(
          response.stopReason().map(BetaStopReason::asString).orElse("null"),
          response.model().asString(),
          servedByFallback));
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello, Claude']],
      model: 'claude-fable-5',
      fallbacks: 'default',
      betas: ['server-side-fallback-2026-07-01'],
  );

  // usage.iterations 中出现 fallback_message 条目表示运行了回退模型；
  // 请结合 stop_reason 确认该响应是否由回退模型提供。
  $iterations = $response->usage->iterations ?? [];
  $servedByFallback = array_any($iterations, fn($entry) => $entry->type === 'fallback_message')
      && $response->stopReason !== 'refusal';

  echo json_encode([
      'stop_reason' => $response->stopReason,
      'model' => $response->model,
      'served_by_fallback' => $servedByFallback,
  ]), PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-fable-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}],
    fallbacks: :default,
    betas: ["server-side-fallback-2026-07-01"]
  )

  # A fallback_message entry in usage.iterations means a fallback model ran;
  # pair it with stop_reason to confirm the fallback served the response.
  iterations = response.usage.iterations || []
  served_by_fallback = iterations.any? { it.type == :fallback_message } &&
    response.stop_reason != :refusal

  stop_reason = response.stop_reason
  model = response.model
  puts JSON.generate({stop_reason:, model:, served_by_fallback:})
  ```
</CodeGroup>

Anthropic 根据模型的能力，为每个模型单独并针对每个策略类别设置安全保障：根据类别的不同，被标记的请求可能会回退到能力较弱的模型，也可能被拒绝。`"default"` 模式为您编码了这些按模型、按类别的推荐，因此被拒绝的请求会在 Anthropic 针对该类别推荐的模型上重试。无论哪种方式，回退都是可见的：响应会指明处理它的模型，并且 `fallback` 内容块会标记交接点。

路由在服务端应用，不会在 [Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 上按模型发布。要查看哪个模型处理了被拒绝的请求，请检查响应的顶层 `model` 字段，并在 `usage.iterations` 中查找 `fallback_message` 条目，正如本页示例所做的那样。

只有安全分类器的拒绝才会触发回退。所请求模型上的速率限制、过载或服务器错误会原样返回给您。

<Note>
  beta 标头必须精确携带日期 `2026-07-01`（同时支持 `"default"` 和显式列表形式）或 `2026-06-01`（仅接受显式列表形式）。在任何其他 `server-side-fallback-*` 值下，`fallbacks` 参数会被拒绝并返回 400 错误。如果您是基于此功能的早期预览版构建的，请将 beta 标头以及请求和响应结构一并更新为本页所示的形式。
</Note>

### 指定您自己的回退模型

除了默认路由，您还可以将 `fallbacks` 设置为最多包含三个模型的列表。当所请求的模型拒绝时，API 会在同一请求上运行链中的下一个模型。当您希望精确控制由哪些模型处理被拒绝的请求时（例如固定使用您的应用程序已验证过的模型），请使用此形式。

指定的回退模型会计入[超大图像检查](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#oversized-image-error)：如果请求的图像块设置了 `"oversized_image": "error"`，则会预先针对所请求的模型和每个指定的回退模型进行检查；如果其中任何一个会缩放该图像，则请求被拒绝，并且拒绝中报告的缩放目标适用于所有这些模型。

高亮显示的行是与默认路由请求的唯一区别。

<CodeGroup>
  ```bash cURL
  curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: server-side-fallback-2026-07-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-fable-5",
      "max_tokens": 1024,
      "fallbacks": [{"model": "claude-opus-4-8"}],
      "messages": [{"role": "user", "content": "Hello, Claude"}]
    }' | jq -r '.model'
  ```

  ```bash CLI
  ant beta:messages create \
    --model claude-fable-5 \
    --max-tokens 1024 \
    --message '{"role":"user","content":"Hello, Claude"}' \
    --fallbacks '[{"model":"claude-opus-4-8"}]' \
    --beta server-side-fallback-2026-07-01 \
    --transform model --raw-output
  ```

  ```python Python
  client = Anthropic()

  response = client.beta.messages.create(
      model="claude-fable-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello, Claude"}],
      fallbacks=[{"model": "claude-opus-4-8"}],
      betas=["server-side-fallback-2026-07-01"],
  )
  print(response.model)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-fable-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    fallbacks: [{ model: "claude-opus-4-8" }],
    betas: ["server-side-fallback-2026-07-01"]
  });
  console.log(response.model);
  ```

  ```csharp C#
  AnthropicClient client = new();

  BetaMessage response = await client.Beta.Messages.Create(
      new()
      {
          Model = Messages::Model.ClaudeFable5,
          MaxTokens = 1024,
          Messages = [new() { Content = "Hello, Claude", Role = Role.User }],
          Fallbacks = new([new(Messages::Model.ClaudeOpus4_8)]),
          Betas = [AnthropicBeta.ServerSideFallback2026_07_01],
      }
  );

  Console.WriteLine(response.Model.Raw());
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.Background(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeFable5,
  	MaxTokens: 1024,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello, Claude")),
  	},
  	Fallbacks: anthropic.BetaFallbacksParamUnion{
  		OfBetaFallbackArray: []anthropic.BetaFallbackParam{{Model: anthropic.ModelClaudeOpus4_8}},
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaServerSideFallback2026_07_01},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Println(response.Model)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  BetaMessage response = client.beta().messages().create(MessageCreateParams.builder()
      .model(Model.CLAUDE_FABLE_5)
      .maxTokens(1024L)
      .addUserMessage("Hello, Claude")
      .fallbacksOfFallbackParams(List.of(BetaFallbackParam.builder()
          .model(Model.CLAUDE_OPUS_4_8)
          .build()))
      .addBeta(AnthropicBeta.SERVER_SIDE_FALLBACK_2026_07_01)
      .build());

  IO.println(response.model().asString());
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      model: 'claude-fable-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello, Claude']],
      fallbacks: [['model' => 'claude-opus-4-8']],
      betas: ['server-side-fallback-2026-07-01'],
  );

  echo $response->model, PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-fable-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}],
    fallbacks: [{model: "claude-opus-4-8"}],
    betas: ["server-side-fallback-2026-07-01"]
  )

  puts response.model
  ```
</CodeGroup>

`fallbacks` 列表适用以下几条规则：

* 条目按顺序尝试。每个条目必须与其他条目以及所请求的模型互不相同。
* 每个条目必须是所请求模型的允许目标之一。设置 beta 标头后，该列表会作为 `allowed_fallback_models` 发布在 [Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 中该模型的条目上。
* 每个条目指定一个 `model`，并且可以仅针对该次尝试覆盖 `max_tokens`、`thinking`、`output_config` 和 `speed`。
* 该请求必须作为对每个指定模型的直接请求都有效。如果某个回退模型不支持请求所使用的功能，API 会预先拒绝该请求。
* 与默认模式一样，只有安全分类器的拒绝才会触发回退。所请求模型上的速率限制、过载或服务器错误会原样返回给您。
* 如果回退模型受到速率限制或过载，则不会进行回退尝试，而是返回之前的拒绝。此时拒绝的 `stop_details.recommended_model` 会指明一个可直接重试的模型。请根据您预期的拒绝量来规划回退模型的速率限制，否则在负载下回退会退化为拒绝。

两种模式下响应的结构相同：处理该轮次的模型出现在顶层 `model` 字段中，`fallback` 内容块标记交接点，`usage.iterations` 记录每次尝试。

### 响应包含的内容

响应看起来与任何其他消息一样，但有两处新增：

* 顶层 `model` 字段报告生成所返回消息的模型，无论是所请求的模型还是回退模型。

* `fallback` 内容块标记 `content` 中一个模型的输出让位于下一个模型的每个位置：`{"type": "fallback", "from": {"model": ...}, "to": {"model": ...}}`。

  * 当拒绝的那一跳是所请求的模型时，`from.model` 会回显您发送的模型字符串。
  * `to.model` 始终是继续处理的模型的已解析 ID。

对于在任何输出之前发生的拒绝，`fallback` 块是第一个内容块。例如，当默认路由针对该拒绝的类别选择 Claude Opus 4.8 时：

```json
{
  "id": "msg_01XFUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "model": "claude-opus-4-8",
  "content": [
    {
      "type": "fallback",
      "from": { "model": "claude-fable-5" },
      "to": { "model": "claude-opus-4-8" }
    },
    { "type": "text", "text": "Hi! How can I help you today?" }
  ],
  "stop_reason": "end_turn",
  "stop_details": null,
  "usage": {
    "input_tokens": 412,
    "output_tokens": 264,
    "cache_read_input_tokens": 0,
    "cache_creation_input_tokens": 0,
    "iterations": [
      {
        "type": "message",
        "model": "claude-fable-5",
        "input_tokens": 535,
        "output_tokens": 0,
        "cache_read_input_tokens": 0,
        "cache_creation_input_tokens": 0
      },
      {
        "type": "fallback_message",
        "model": "claude-opus-4-8",
        "input_tokens": 412,
        "output_tokens": 264,
        "cache_read_input_tokens": 0,
        "cache_creation_input_tokens": 0
      }
    ]
  }
}
```

`usage.iterations` 数组记录每次尝试。拒绝的模型以普通的 `message` 条目出现，处理该轮次的模型以 `fallback_message` 条目出现。如果链中的每个模型都拒绝，则响应为最后一个模型的拒绝，其中每个较早的跳转各有一个 `message` 条目，最后一个有一个 `fallback_message` 条目。

[粘性路由](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#sticky-routing)可以将后续轮次直接发送到回退模型。这样的轮次不携带 `fallback` 内容块，因为该轮次没有模型拒绝。可通过 `usage.iterations` 中的 `fallback_message` 条目、所请求模型没有 `message` 条目以及响应的 `model` 字段来识别它。

### 继续对话

在下一轮中，按您收到的原样发回助手内容。在输出中途回退之后，`content` 可能包含拒绝模型在交接前生成的块类型。下表说明了在回显该轮次时应保留哪些、丢弃哪些。

| 块类型                                                                    | 在下一轮中                                                           |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- |
| `fallback`                                                             | 精确保留在其出现的位置。API 使用其位置来验证其周围的思考块，因此如果该块被省略或移动，回显了边界两侧思考块的请求会被拒绝。 |
| `text`                                                                 | 保留。                                                             |
| 最后一个 `fallback` 块之后的任何块                                                | 保留。                                                             |
| 最后一个 `fallback` 块之前的 `thinking`、`redacted_thinking` 或 `connector_text` | 丢弃。                                                             |
| 最后一个 `fallback` 块之前的客户端 `tool_use`                                     | 丢弃。                                                             |
| 最后一个 `fallback` 块之前的 `server_tool_use`                                 | 与其结果配对时保留。没有匹配结果时丢弃。                                            |

<Note>
  `connector_text` 块携带某些使用工具的响应在工具调用之间包含的叙述文本。
</Note>

### 流式传输

在流式传输请求中，重试发生在同一个流上，您已收到的任何内容都不会失效。您看到的内容取决于拒绝发生的时间。

**当拒绝发生在任何输出之前时：**

* `message_start` 指明回退模型，`fallback` 块是第一个内容块。
* 由于 `message_start` 会等待回退尝试开始，因此首字节时间包含被拒绝的尝试。

**当拒绝发生在输出中途时：**

* 打开的内容块关闭，`fallback` 块（一对普通的 `content_block_start` 和 `content_block_stop`，没有增量）标记边界。
* 回退模型从部分输出处继续。只有部分输出的 `text` 块会作为上下文传递给回退模型。其他块类型保留在 `content` 中。
* `message_start` 已经指明了所请求的模型，因此请从 `fallback` 块的 `to.model` 以及最终 `message_delta` 的 `usage.iterations` 中的 `fallback_message` 条目读取处理模型。

### 非流式传输响应

在非流式传输请求中，输出中途的拒绝行为有所不同：响应会省略被拒绝模型的部分输出，回退模型从头开始作答。结果看起来像是在任何输出之前发生的拒绝，`fallback` 块位于首位。被拒绝的尝试及其输出令牌仍会出现在 `usage.iterations` 中。

<Note>
  **工具使用期间的拒绝：** 已完成的工具工作不会阻止回退。当拒绝在服务器工具（例如网页搜索或代码执行）已在请求内执行完毕之后触发时，回退尝试会继续进行：已完成的工具结果会被带过去，回退模型可以继续调用服务器工具。唯一不会重试的情况是：流式传输中的拒绝在任何类型的工具使用块（客户端工具、服务器工具或 MCP 工具调用）仍在流上打开时触发：该拒绝会直接返回，并且如果设置了 `fallback-credit-2026-07-01` 标头，它仍会携带一个可通过继续部分响应来兑换的抵扣令牌。非流式传输请求不受影响；API 会清除部分工作并在响应前重试。
</Note>

### 计费与速率限制

在生成任何输出之前就拒绝的尝试不计费：其令牌会在其 `usage.iterations` 条目上报告，但不收费。每次生成了输出的尝试（包括在响应中途拒绝的尝试）都按运行它的模型的费率单独计费。`usage.iterations` 数组是您被计费内容的逐次尝试记录。顶层 `usage` 计数仅描述生成所返回消息的那次尝试。来自不同模型的令牌绝不会被汇总到一个字段中。

每次运行的尝试（包括拒绝的尝试）都计入其自身模型的速率限制。

### 粘性路由

对话回退后，API 会记录哪个模型处理了它。该对话后续包含 `fallbacks` 的请求会直接发送到该回退模型，而不运行所请求的模型。这避免了在每一轮都为一次可预见会再次被拒绝的尝试付费。

路由决策的几个特性：

* 它保留约 1 小时，并限定在您的组织范围内。
* 它以对话前缀的内容哈希加上处理它的模型的形式存储。消息内容本身不会被存储。
* 它是尽力而为的，因此您的代码必须能够处理所请求的模型在任何时候被再次尝试的情况。

粘性路由同时适用于流式传输和非流式传输请求。在流式传输请求中，路由决策在流打开之前做出，因此 `message_start` 事件的 `model` 字段已经携带回退模型的 ID。

## 配合 SDK 中间件的客户端回退

每个 Anthropic SDK 都包含一个拒绝回退中间件。您在客户端上用您的回退模型列表配置一次。之后通过 `client.beta.messages` 的调用会在任何平台上自动重试被拒绝的请求。该中间件还会在其处理的每个请求上发送 `fallback-credit-2026-07-01` beta 标头，因此重试会被重新定价，而无需逐请求设置。

### 设置

将中间件传递给客户端构造函数，并在一个对话的各个请求之间共享一个 `BetaFallbackState` 实例。

<CodeGroup>
  ```bash cURL
  # refusal-fallback 中间件是 SDK 功能。请参阅
  # 服务器端回退部分了解等效的单请求方法，
  # 或参阅回退额度页面了解原始 HTTP 重试模式。
  ```

  ```bash CLI
  # refusal-fallback 中间件是 SDK 功能。请参阅
  # 服务器端回退部分了解等效的单请求方法，
  # 或参阅回退额度页面了解原始 HTTP 重试模式。
  ```

  ```python Python
  from anthropic import Anthropic, BetaFallbackState, BetaRefusalFallbackMiddleware

  # 遇到拒绝时，中间件会在列出的回退模型上重试，并
  # 自动在其处理的每个请求上发送 fallback-credit beta 标头。
  client = Anthropic(
      middleware=[BetaRefusalFallbackMiddleware([{"model": "claude-opus-4-8"}])],
  )

  state = BetaFallbackState()  # pins follow-ups to the model that accepted

  # 流式传输：遇到拒绝时，中间件会在回退模型上重试，并
  # 将其事件拼接到已打开的流上。
  with (
      state,
      client.beta.messages.stream(
          max_tokens=1024,
          model="claude-fable-5",
          messages=[{"role": "user", "content": "Hello, Claude"}],
      ) as stream,
  ):
      for text in stream.text_stream:
          print(text, end="", flush=True)
      final_message = stream.get_final_message()
  print(f"\nserved by: {final_message.model}")

  # 非流式传输：复用该状态可使对话保持固定。
  with state:
      message = client.beta.messages.create(
          max_tokens=1024,
          model="claude-fable-5",
          messages=[{"role": "user", "content": "Hello, Claude"}],
      )
  print(f"served by: {message.model}")
  ```

  ```typescript TypeScript
  import { BetaFallbackState, betaRefusalFallbackMiddleware } from "@anthropic-ai/sdk";

  // 遇到拒绝时，中间件会在列出的回退模型上重试，并且
  // 会在其处理的每个请求上自动发送 fallback-credit beta 标头。
  const client = new Anthropic({
    middleware: [betaRefusalFallbackMiddleware([{ model: "claude-opus-4-8" }])]
  });

  // 在整个对话中共享同一个状态，以便后续请求保持
  // 固定在已接受请求的模型上。
  const fallbackState = new BetaFallbackState();

  // 流式传输：遇到拒绝时，中间件会在回退模型上重试，并且
  // 将其事件拼接到已打开的流上。
  const stream = client.beta.messages
    .stream(
      {
        max_tokens: 1024,
        model: "claude-fable-5",
        messages: [{ role: "user", content: "Hello, Claude" }]
      },
      { fallbackState }
    )
    .on("text", (text) => process.stdout.write(text));

  const finalMessage = await stream.finalMessage();
  console.log("\nserved by:", finalMessage.model);

  // 非流式传输：复用该状态可使对话保持固定。
  const message = await client.beta.messages.create(
    {
      max_tokens: 1024,
      model: "claude-fable-5",
      messages: [{ role: "user", content: "Hello, Claude" }]
    },
    { fallbackState }
  );
  console.log("served by:", message.model);
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Helpers;
  using Anthropic.Models.Beta.Messages;
  using Messages = Anthropic.Models.Messages;

  // 遇到拒绝时，处理程序会在列出的回退模型上重试，并且
  // 会在其处理的每个请求上自动发送 fallback-credit beta 标头。
  AnthropicClient client = new()
  {
      Handlers =
      [
          new BetaRefusalFallbackHandler { Fallbacks = [new(Messages::Model.ClaudeOpus4_8)] },
      ],
  };

  // 将共享此状态的后续请求固定到已接受请求的模型上。
  BetaFallbackState fallbackState = BetaFallbackState.Create();

  MessageCreateParams parameters = new()
  {
      Model = Messages::Model.ClaudeFable5,
      MaxTokens = 1024,
      Messages = [new() { Content = "Hello, Claude", Role = Role.User }],
  };

  // 流式传输：如果流以拒绝结束，处理程序会将回退
  // 模型的事件拼接到仍处于打开状态的流上。
  BetaMessageContentAggregator aggregator = new();
  using (fallbackState.Use())
  {
      var responseUpdates = client.Beta.Messages.CreateStreaming(parameters);
      await foreach (BetaRawMessageStreamEvent rawEvent in responseUpdates.CollectAsync(aggregator))
      {
          if (
              rawEvent.TryPickContentBlockDelta(out var deltaEvent)
              && deltaEvent.Delta.TryPickText(out var textDelta)
          )
          {
              Console.Write(textDelta.Text);
          }
      }
  }
  BetaMessage streamedMessage = aggregator.Message();
  Console.WriteLine($"\nserved by: {streamedMessage.Model.Raw()}");

  // 非流式传输：复用该状态可使对话保持固定在已接受请求的模型上。
  using (fallbackState.Use())
  {
      BetaMessage message = await client.Beta.Messages.Create(parameters);
      Console.WriteLine($"served by: {message.Model.Raw()}");
  }
  ```

  ```go Go
  import (
  // ...
  	"github.com/anthropics/anthropic-sdk-go/lib/betafallback"
  // ...
  )

  func main() {
  	ctx := context.Background()

  	// 该中间件会在请求被拒绝时依次在每个回退模型上重试，
  	// 并自动将请求加入 fallback-credit beta。
  	client := anthropic.NewClient(
  		option.WithMiddleware(betafallback.BetaRefusalFallbackMiddleware(
  			[]anthropic.BetaFallbackParam{{Model: anthropic.ModelClaudeOpus4_8}},
  		)),
  	)

  	// 每个对话一个状态：共享该状态的请求会固定到
  	// 已接受请求的模型上，因此后续请求不会再询问曾拒绝的模型。
  	state := &betafallback.BetaFallbackState{}
  	conversation := betafallback.WithBetaFallbackState(state)

  	params := anthropic.BetaMessageNewParams{
  		MaxTokens: 1024,
  		Model:     anthropic.ModelClaudeFable5,
  		Messages: []anthropic.BetaMessageParam{
  			anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello, Claude")),
  		},
  	}

  	// 流式传输：遇到拒绝时，中间件会就地重试，将回退模型的
  	// 事件拼接到已打开的流上，形成一条连续的消息。
  	stream := client.Beta.Messages.NewStreaming(ctx, params, conversation)
  	defer stream.Close()
  	var streamed anthropic.BetaMessage
  	for stream.Next() {
  		event := stream.Current()
  		if err := streamed.Accumulate(event); err != nil {
  			panic(err)
  		}
  		switch eventVariant := event.AsAny().(type) {
  		case anthropic.BetaRawContentBlockDeltaEvent:
  			if textDelta, ok := eventVariant.Delta.AsAny().(anthropic.BetaTextDelta); ok {
  				fmt.Print(textDelta.Text)
  			}
  		}
  	}
  	if err := stream.Err(); err != nil {
  		panic(err)
  	}
  	fmt.Println("\nserved by:", streamed.Model)

  	// 非流式传输：共享状态会将此后续请求固定到
  	// 处理了流式传输轮次的模型上。
  	message, err := client.Beta.Messages.New(ctx, params, conversation)
  	if err != nil {
  		panic(err)
  	}
  	fmt.Println("served by:", message.Model)
  }
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.core.RequestOptions;
  import com.anthropic.core.http.StreamResponse;
  import com.anthropic.helpers.BetaFallbackState;
  import com.anthropic.helpers.BetaMessageAccumulator;
  import com.anthropic.helpers.BetaRefusalFallbackInterceptor;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.BetaRawMessageStreamEvent;
  import com.anthropic.models.beta.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;

  void main() {
      // 该拦截器会在回退模型上重试被拒绝的请求。它会自动
      // 为其处理的每个请求添加 fallback-credit beta 标头。
      AnthropicClient client = AnthropicOkHttpClient.builder()
          .fromEnv()
          .addInterceptor(BetaRefusalFallbackInterceptor.builder()
              .addFallback(Model.CLAUDE_OPUS_4_8)
              .build())
          .build();

      // 在请求之间共享同一状态，使后续请求固定在已接受请求的模型上。
      BetaFallbackState state = BetaFallbackState.create();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_FABLE_5)
          .maxTokens(1024)
          .addUserMessage("Hello, Claude")
          .build();

      // 流式传输：发生拒绝时，回退模型的事件会被拼接到已打开的流上。
      BetaMessageAccumulator accumulator = BetaMessageAccumulator.create();
      try (StreamResponse<BetaRawMessageStreamEvent> streamResponse = client.beta()
              .messages()
              .createStreaming(params, RequestOptions.builder().fallbackState(state).build())) {
          streamResponse.stream()
              .peek(accumulator::accumulate)
              .forEach(event -> event.contentBlockDelta()
                  .flatMap(deltaEvent -> deltaEvent.delta().text())
                  .ifPresent(textDelta -> IO.print(textDelta.text())));
      }
      IO.println("\nserved by: " + accumulator.message().model().asString());

      // 非流式传输：复用同一状态可使对话保持固定。
      BetaMessage message = client.beta()
          .messages()
          .create(params, RequestOptions.builder().fallbackState(state).build());
      IO.println("served by: " + message.model().asString());
  }
  ```

  ```php PHP
  use Anthropic\Beta\Messages\BetaRawContentBlockDeltaEvent;
  use Anthropic\Beta\Messages\BetaTextDelta;
  use Anthropic\Client;
  use Anthropic\Lib\Middleware\BetaFallbackState;
  use Anthropic\Lib\Middleware\RefusalFallbackMiddleware;
  use Anthropic\Lib\Streaming\MessageAccumulator;

  // 只需配置一次回退链。遇到拒绝时，中间件会沿着回退链重试
  // 该请求，并为您发送 fallback-credit beta 请求头。
  $client = new Client(
      requestOptions: [
          'middleware' => [new RefusalFallbackMiddleware([['model' => 'claude-opus-4-8']])],
      ],
  );

  // 在整个对话中共享同一个状态，以便后续请求始终固定
  // 到已接受请求的模型上。
  $state = new BetaFallbackState();

  // 流式传输：遇到拒绝时，中间件会将回退模型的事件拼接
  // 到仍处于打开状态的流上。累加器的模型即为实际提供服务的模型。
  $stream = $client->beta->messages->createStream(
      model: 'claude-fable-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello, Claude']],
      requestOptions: ['fallbackState' => $state],
  );
  $accumulator = MessageAccumulator::forBetaMessages();
  foreach ($stream as $event) {
      $accumulator->accumulate($event);
      if ($event instanceof BetaRawContentBlockDeltaEvent
          && $event->delta instanceof BetaTextDelta) {
          echo $event->delta->text;
      }
  }
  echo "\nserved by: {$accumulator->message()->model}\n";

  // 非流式传输：使用相同的中间件。复用该状态可使对话始终
  // 固定到已接受请求的模型上。
  $message = $client->beta->messages->create(
      model: 'claude-fable-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello, Claude']],
      requestOptions: ['fallbackState' => $state],
  );
  echo "served by: {$message->model}\n";
  ```

  ```ruby Ruby
  # 遇到拒绝时，中间件会沿回退链重试该请求。
  # 它会在处理的每个请求上发送 fallback-credit beta 标头。
  client = Anthropic::Client.new(
    middleware: [Anthropic::BetaRefusalFallbackMiddleware.new([{model: "claude-opus-4-8"}])]
  )

  # 在整个对话中共享同一状态，使后续请求保持
  # 固定在已接受请求的模型上。
  state = Anthropic::BetaFallbackState.new

  # 流式传输：遇到拒绝时，中间件会将回退模型的
  # 事件拼接到仍处于打开状态的流上。
  stream = client.beta.messages.stream(
    model: "claude-fable-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}],
    request_options: {fallback_state: state}
  )
  stream.text.each { print it }
  puts "\nserved by: #{stream.accumulated_message.model}"

  # 非流式传输：复用该状态可使对话保持固定在已接受请求的模型上。
  message = client.beta.messages.create(
    model: "claude-fable-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}],
    request_options: {fallback_state: state}
  )
  puts "served by: #{message.model}"
  ```
</CodeGroup>

### 行为方式

* 重试按顺序遍历您的回退列表。自身也拒绝的回退模型会将请求传递给下一个条目。
* 当列表中的每个模型都已拒绝时，中间件返回最终的拒绝（最后一个模型的拒绝响应），而不是抛出错误。
* 来自 Claude Fable 5.1 或 Claude Fable 5 的思考块会原样传递。每次重试都会重新发送您的原始请求体，中间件在后续请求中从对话历史中移除的唯一块是它自己添加的 `fallback` 边界块。回退模型无法读取 Claude Fable 5.1 的块，这些块[仅为该模型或更新的模型保留](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-for-model)，因此 API 会丢弃它们。
* 通过中间件处理的响应在每个模型边界处包含一个 `fallback` 内容块，与服务端回退响应相同。中间件会在后续请求中为您管理这些块。
* 接受请求的模型会记录在 `BetaFallbackState` 中，因此共享该状态的后续请求会固定使用它，而不是重新询问已拒绝的模型。

<Note>
  中间件和服务端 `fallbacks` 参数完成相同的工作。请配置其中之一，切勿在同一请求上同时配置两者。要从安装了中间件的应用程序发送服务端 `fallbacks` 请求，请使用一个不带中间件的单独客户端实例。
</Note>

## 自行编写重试

通过原始 HTTP 或使用自定义重试逻辑时，请实现中间件所封装的模式：

<Steps>
  <Step title="检测拒绝">
    检查响应中是否有 `stop_reason: "refusal"`。
  </Step>

  <Step title="在回退模型上重新发送">
    发送同一请求，并将 `model` 设置为回退模型，例如 Claude Opus 4.8。另一个模型通常可以处理 Claude Fable 5.1 或 Claude Fable 5 拒绝的请求。您如何处理对话历史取决于您是否兑换[回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)：

    * **不兑换抵扣：** 您可以保留较早的 `thinking` 和 `redacted_thinking` 块，也可以剥离它们以节省输入令牌。无论哪种方式，回退模型都无法使用它们：它会忽略 Claude Fable 5 的块，而 Claude Fable 5.1 的块[仅为该模型或更新的模型保留](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-for-model)，因此 API 会丢弃它们。
    * **兑换抵扣：** 原样发送请求体，因为兑换要求精确匹配。服务器会在兑换时处理较早模型的思考块，因此不要剥离它们（请参阅[必须与被拒绝请求匹配的字段](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit#reference)）。
  </Step>

  <Step title="保持使用回退模型">
    对于多轮对话，后续轮次请继续使用回退模型，而不要切换回去。
  </Step>
</Steps>

手动重试会从头写入回退模型的提示缓存，这比读取现有缓存的成本更高。[回退抵扣](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)会退还该成本；请在您自行构建的每次重试中兑换它。

## Message Batches 中的拒绝

[Message Batch](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing) 中被拒绝的请求会以 `result.type: "succeeded"` 和 `stop_reason: "refusal"` 返回。批处理结果携带与同步响应相同的 `stop_details` 对象，因此您可以通过 `stop_reason` 或 `stop_details.type` 检测拒绝。一个区别是：批处理拒绝不会生成回退抵扣，因此批处理结果上的 `stop_details` 绝不会包含 `fallback_credit_token`。

服务端回退不适用于批处理（包含 `fallbacks` 的批处理请求会产生逐项的出错结果）。要重试被拒绝的批处理项：

1. 从结果中收集被拒绝的项。
2. 从任何多轮历史中剥离 Claude Fable 5.1 或 Claude Fable 5 的思考块。
3. 将它们作为新批处理或直接请求在回退模型上重新提交。

## 常见陷阱

* **在不同的模型上重试。** 将被拒绝的请求重新发送到同一模型通常会再次被拒绝。请将重试指向回退模型。
* **按请求而非按轮次或按会话规划重试预算。** 单个轮次可能产生多次拒绝，例如一个智能体加上其子智能体。
* **在每条请求路径上配置回退。** 重试处理程序、错误恢复分支和后台工作进程都需要它。一个在没有回退的情况下重新发出请求的处理程序，恰恰会在最可能需要保护的请求上失去保护。
* **为子智能体调用提供它们自己的回退。** `fallbacks` 参数不会传播到从工具执行内部发起的模型调用中。
* **让回退成为请求的属性，而不是环境状态的属性。** 共享标志、缓存的配置值或全局开关可能会失去同步，并悄无声息地使请求失去保护。当您无法确认回退处于活动状态时，请配置它，而不要假设它已开启。
* **将拒绝作为独立的信号进行监测。** 拒绝是 HTTP 200，因此基于错误率或 5xx 响应构建的监控永远看不到它。为每次拒绝发出一个事件，为每个由回退处理的响应发出一个事件（`usage.iterations` 中的 `fallback_message` 条目标记后者），然后针对两个计数之间的差距发出告警。
* **根据 `stop_reason` 或 `stop_details.type` 进行分支，而不是根据 `content` 或内部的 `stop_details` 字段。** `stop_details` 对象在拒绝时始终存在，但其 `category` 和 `explanation` 字段可能为 `null`。请直接检查 `stop_reason` 是否等于 `"refusal"`。

## 后续步骤

<CardGroup>
  <Card title="回退抵扣" icon="scales" href="https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit">
    当您自行构建重试时，避免支付两次提示缓存成本。
  </Card>

  <Card title="停止原因与回退" icon="code" href="https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons">
    每个 `stop_reason` 值及其处理方式。
  </Card>

  <Card title="SDK 中间件" icon="settings" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/middleware">
    SDK 中间件的工作原理，包括拒绝回退辅助工具。
  </Card>

  <Card title="迁移指南" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide">
    将现有应用程序迁移到 Claude Fable 5.1。
  </Card>
</CardGroup>
