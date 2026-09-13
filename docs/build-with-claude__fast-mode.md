---
title: 快速模式（研究预览版）
url: https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode
description: 从受支持的 Claude Opus 模型获得最高 2.5 倍的每秒输出令牌数。
---

快速模式（fast mode）以高级定价从 Claude Opus 5 和 Claude Opus 4.8 提供最高 2.5 倍的每秒输出令牌数。在您的请求中设置 `speed: "fast"` 并附带 `fast-mode-2026-02-01` beta 标头即可选择启用。

<Note>
  快速模式目前处于研究预览阶段。请联系您的客户经理申请访问权限。如果您没有客户经理，请[加入快速模式的等候名单](https://claude.com/fast-mode)。
</Note>

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

## 支持的模型

以下模型支持快速模式：

* Claude Opus 5 (claude-opus-5)
* Claude Opus 4.8 (claude-opus-4-8)

<Note>
  Claude Opus 5 和 Claude Opus 4.8 的快速模式仅在 Claude API（包括 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)）上作为研究预览版提供。它在 Amazon Bedrock、Claude Platform on AWS、Google Cloud 或 Microsoft Foundry 上不可用。
</Note>

<Note>
  快速模式在 Claude Opus 4.7 上不可用。向 `claude-opus-4-7` 发送带有 `speed: "fast"` 的请求会返回错误；与 Claude Opus 4.6（见下一条说明）不同，请求不会回退到标准速度。该模型本身仍可在标准速度下使用。要继续使用快速模式，请迁移到 [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-47) 或 Claude Opus 4.8。
</Note>

<Note>
  快速模式在 Claude Opus 4.6 上不可用。向 `claude-opus-4-6` 发送带有 `speed: "fast"` 的请求不会返回错误：它们以标准速度运行，并按[标准费率](https://platform.claude.com/docs/zh-CN/about-claude/pricing)而非快速模式的高级费率计费，且响应会报告 [`usage.speed: "standard"`](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#checking-which-speed-was-used)。要继续使用快速模式，请迁移到 [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-46) 或 Claude Opus 4.8。
</Note>

## 快速模式的工作原理

快速模式使用更快的推理配置运行同一模型。智能或能力没有任何变化。

* 与标准速度相比，每秒输出令牌数最高提升 2.5 倍
* 速度优势集中在"output tokens per second"（每秒输出令牌数），即 OTPS，而非"time to first token"（首令牌时间），即 TTFT
* 相同的模型权重和行为（不是不同的模型）
* 与 [streaming（流式传输）](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)兼容，在流式传输中 OTPS 的提升最为明显

## 基本用法

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: fast-mode-2026-02-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "speed": "fast",
      "messages": [{
        "role": "user",
        "content": "Refactor this module to use dependency injection"
      }]
    }'
  ```

  ```bash CLI
  ant beta:messages create \
    --beta fast-mode-2026-02-01 \
    --transform 'content.#(type=="text").text' \
    --raw-output <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  speed: fast
  messages:
    - role: user
      content: Refactor this module to use dependency injection
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      speed="fast",
      betas=["fast-mode-2026-02-01"],
      messages=[
          {"role": "user", "content": "Refactor this module to use dependency injection"}
      ],
  )

  for block in response.content:
      if block.type == "text":
          print(block.text)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    speed: "fast",
    betas: ["fast-mode-2026-02-01"],
    messages: [
      {
        role: "user",
        content: "Refactor this module to use dependency injection"
      }
    ]
  });

  const textBlock = response.content.find(
    (block): block is Anthropic.Beta.Messages.BetaTextBlock => block.type === "text"
  );
  console.log(textBlock?.text);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var response = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Speed = Speed.Fast,
      Betas = ["fast-mode-2026-02-01"],
      Messages = [
          new() { Role = Role.User, Content = "Refactor this module to use dependency injection" }
      ],
  });

  foreach (var block in response.Content)
  {
      if (block.TryPickText(out var textBlock))
      {
          Console.WriteLine(textBlock.Text);
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 4096,
  	Speed:     anthropic.BetaMessageNewParamsSpeedFast,
  	Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaFastMode2026_02_01},
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Refactor this module to use dependency injection")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  for _, block := range response.Content {
  	if textBlock, ok := block.AsAny().(anthropic.BetaTextBlock); ok {
  		fmt.Println(textBlock.Text)
  	}
  }
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  BetaMessage response = client.beta().messages().create(
          MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5)
                  .maxTokens(4096L)
                  .speed(MessageCreateParams.Speed.FAST)
                  .addBeta(AnthropicBeta.FAST_MODE_2026_02_01)
                  .addUserMessage("Refactor this module to use dependency injection")
                  .build());

  response.content().stream()
          .flatMap(block -> block.text().stream())
          .forEach(textBlock -> IO.println(textBlock.text()));
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      model: 'claude-opus-5',
      maxTokens: 4096,
      speed: 'fast',
      betas: ['fast-mode-2026-02-01'],
      messages: [
          ['role' => 'user', 'content' => 'Refactor this module to use dependency injection'],
      ],
  );

  foreach ($response->content as $block) {
      if ($block->type === 'text') {
          echo $block->text, PHP_EOL;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    speed: "fast",
    betas: ["fast-mode-2026-02-01"],
    messages: [{role: "user", content: "Refactor this module to use dependency injection"}]
  )

  response.content.each do |block|
    puts block.text if block.type == :text
  end
  ```
</CodeGroup>

## 定价

快速模式的定价是在整个 context window（上下文窗口）范围内对标准费率乘以一个倍数，包括超过 200k 输入令牌的请求。下表显示了受支持模型的快速模式定价：

| 模型                              | 输入             | 输出             |
| ------------------------------- | -------------- | -------------- |
| Claude Opus 5 / Claude Opus 4.8 | $10 USD / MTok | $50 USD / MTok |

快速模式定价可与其他定价修正项叠加：

* [提示缓存倍数](https://platform.claude.com/docs/zh-CN/about-claude/pricing#prompt-caching)在快速模式定价之上叠加适用
* [数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)倍数在快速模式定价之上叠加适用

有关完整的定价详情，请参阅[定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#fast-mode-pricing)页面。

## 速率限制

快速模式拥有专用的 rate limit（速率限制），与标准 Opus 速率限制相互独立。当超出您的快速模式速率限制时，API 会返回 `429` 错误，并附带 `retry-after` 标头，指示容量何时可用。

响应中包含指示您的快速模式速率限制状态的标头：

| 标头                                       | 描述               |
| ---------------------------------------- | ---------------- |
| `anthropic-fast-input-tokens-limit`      | 每分钟快速模式输入令牌的最大数量 |
| `anthropic-fast-input-tokens-remaining`  | 剩余的快速模式输入令牌数     |
| `anthropic-fast-input-tokens-reset`      | 快速模式输入令牌限制重置的时间  |
| `anthropic-fast-output-tokens-limit`     | 每分钟快速模式输出令牌的最大数量 |
| `anthropic-fast-output-tokens-remaining` | 剩余的快速模式输出令牌数     |
| `anthropic-fast-output-tokens-reset`     | 快速模式输出令牌限制重置的时间  |

有关各层级的速率限制，请参阅[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)页面。

## 检查使用了哪种速度

响应的 `usage` 对象包含一个 `speed` 字段，用于指示使用了哪种速度，值为 `"fast"` 或 `"standard"`。在[不支持快速模式的模型](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#supported-models)上请求 `speed: "fast"` 会返回错误，超出快速模式的速率限制或容量（`429` 或 `529`）也会返回错误。当带有 `speed: "fast"` 的请求成功时，`usage.speed` 为 `"fast"`。如果您使用 Claude Opus 4.6 并请求快速模式，其行为是独特的。它不会像其他不支持快速模式的模型那样返回错误，而是静默切换到标准速度。尽管 Opus 4.6 不会报错，但 `speed` 字段会准确显示 `"standard"`。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: fast-mode-2026-02-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "speed": "fast",
      "messages": [{"role": "user", "content": "Hello"}]
    }'
  ```

  ```bash CLI
  ant beta:messages create \
    --beta fast-mode-2026-02-01 \
    --transform usage.speed \
    --raw-output <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  speed: fast
  messages:
    - role: user
      content: Hello
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      speed="fast",
      betas=["fast-mode-2026-02-01"],
      messages=[{"role": "user", "content": "Hello"}],
  )

  print(response.usage.speed)  # "fast" or "standard"
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    speed: "fast",
    betas: ["fast-mode-2026-02-01"],
    messages: [{ role: "user", content: "Hello" }]
  });

  console.log(response.usage.speed); // "fast" or "standard"
  ```

  ```csharp C#
  AnthropicClient client = new();

  var response = await client.Beta.Messages.Create(new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 1024,
      Speed = Speed.Fast,
      Betas = ["fast-mode-2026-02-01"],
      Messages = [new() { Role = Role.User, Content = "Hello" }],
  });

  Console.WriteLine(response.Usage.Speed);  // "fast" or "standard"
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Speed:     anthropic.BetaMessageNewParamsSpeedFast,
  	Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaFastMode2026_02_01},
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response.Usage.Speed) // "fast" or "standard"
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024L)
          .speed(MessageCreateParams.Speed.FAST)
          .addBeta(AnthropicBeta.FAST_MODE_2026_02_01)
          .addUserMessage("Hello")
          .build();

  BetaMessage response = client.beta().messages().create(params);
  IO.println(response.usage().speed().orElseThrow());  // "fast" or "standard"
  ```

  ```php PHP
  $client = new Client();

  $response = $client->beta->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      speed: 'fast',
      betas: ['fast-mode-2026-02-01'],
      messages: [['role' => 'user', 'content' => 'Hello']],
  );

  echo $response->usage->speed;  // "fast" or "standard"
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    speed: "fast",
    betas: ["fast-mode-2026-02-01"],
    messages: [{ role: "user", content: "Hello" }]
  )

  puts(response.usage.speed)  # "fast" or "standard"
  ```
</CodeGroup>

```json Output
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
// ...
  "usage": {
    "input_tokens": 8,
    "output_tokens": 12,
    "speed": "fast"
  }
}
```

要跟踪整个组织的快速模式使用情况和成本，请参阅[使用量和成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api)。

## 重试与回退

### 自动重试

当超出快速模式速率限制时，API 会返回 `429` 错误并附带 `retry-after` 标头。Anthropic SDK 默认会自动重试这些请求最多 2 次（可通过 `max_retries` 配置），每次重试前会等待服务器指定的延迟时间。由于快速模式使用持续的令牌补充机制，`retry-after` 延迟通常很短，一旦容量可用，请求即可成功。

### 回退到标准速度

<Note>
  本节介绍在快速模式受到速率限制时可选择启用的客户端回退。这与 [Claude Opus 4.6](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#supported-models) 上的行为不同，在 Claude Opus 4.6 上快速模式不可用，请求会自动以标准速度运行。
</Note>

如果您更希望回退到标准速度而不是等待快速模式容量，请捕获速率限制错误并在不带 `speed: "fast"` 的情况下重试。在初始快速请求上将 `max_retries` 设置为 `0`，以跳过自动重试并在遇到速率限制错误时立即失败。

<Note>
  从快速速度回退到标准速度将导致 [prompt cache（提示缓存）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)未命中。不同速度的请求不共享缓存前缀。
</Note>

由于将 `max_retries` 设置为 `0` 也会禁用针对其他瞬时错误（过载、内部服务器错误）的重试，以下示例会针对这些情况以默认重试设置重新发出原始请求。

<CodeGroup exclude="shell:cURL">
  ```bash CLI
  # `ant` 会自动重试 429/5xx，且没有按请求设置的 max_retries
  # 覆盖选项，因此在快速模式遇到 429 时，回退会在内置
  # 重试耗尽后运行。--transform-error 会暴露 error.type 以供分支判断。
  create_message_with_fast_fallback() {
    local speed="$1" max_attempts="${2:-3}" body out
    body=${3:-$(cat)}
    out=$(
      ant beta:messages create --beta fast-mode-2026-02-01 \
        ${speed:+--speed "$speed"} \
        --transform-error error.type --format-error yaml <<<"$body" 2>/dev/null
    ) && { printf '%s\n' "$out"; return; }
    case "$out" in
      rate_limit_error)
        if [[ -n "$speed" ]]; then
          create_message_with_fast_fallback "" "$max_attempts" "$body"
          return
        fi ;;
      overloaded_error | api_error | "")
        if (( max_attempts > 1 )); then
          create_message_with_fast_fallback "$speed" $((max_attempts - 1)) "$body"
          return
        fi ;;
    esac
    printf '%s\n' "${out:-connection_error}" >&2
    return 1
  }

  MESSAGE=$(
    create_message_with_fast_fallback fast <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content: Hello
  YAML
  )
  ```

  ```python Python
  client = anthropic.Anthropic()


  def create_message_with_fast_fallback(max_retries=0, max_attempts=3, **params):
      try:
          return client.with_options(max_retries=max_retries).beta.messages.create(
              **params
          )
      except anthropic.RateLimitError:
          if params.get("speed") == "fast":
              del params["speed"]
              return create_message_with_fast_fallback(max_retries=max_retries, **params)
          raise
      except (
          anthropic.APIStatusError,
          anthropic.APIConnectionError,
      ) as error:
          if isinstance(error, anthropic.APIStatusError) and error.status_code < 500:
              raise
          if max_attempts > 1:
              return create_message_with_fast_fallback(
                  max_retries=max_retries, max_attempts=max_attempts - 1, **params
              )
          raise


  message = create_message_with_fast_fallback(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello"}],
      betas=["fast-mode-2026-02-01"],
      speed="fast",
      max_retries=0,
  )
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  async function createMessageWithFastFallback(
    params: Anthropic.Beta.MessageCreateParamsNonStreaming,
    requestOptions?: Anthropic.RequestOptions,
    maxAttempts: number = 3
  ): Promise<Anthropic.Beta.Messages.BetaMessage> {
    try {
      return await client.beta.messages.create(params, requestOptions);
    } catch (e) {
      if (e instanceof Anthropic.RateLimitError && params.speed === "fast") {
        const { speed, ...rest } = params;
        return createMessageWithFastFallback(rest);
      }
      if (
        e instanceof Anthropic.InternalServerError ||
        e instanceof Anthropic.APIConnectionError
      ) {
        if (maxAttempts > 1) {
          return createMessageWithFastFallback(params, undefined, maxAttempts - 1);
        }
      }
      throw e;
    }
  }

  const message = await createMessageWithFastFallback(
    {
      model: "claude-opus-5",
      max_tokens: 1024,
      messages: [{ role: "user", content: "Hello" }],
      betas: ["fast-mode-2026-02-01"],
      speed: "fast"
    },
    { maxRetries: 0 }
  );
  ```

  ```csharp C#
  AnthropicClient client = new();

  async Task<BetaMessage> CreateMessageWithFastFallback(
      MessageCreateParams parameters,
      int? maxRetries = null,
      int maxAttempts = 3)
  {
      try
      {
          var requestClient = maxRetries is int retries
              ? client.WithOptions(options => options with { MaxRetries = retries })
              : client;
          return await requestClient.Beta.Messages.Create(parameters);
      }
      catch (AnthropicRateLimitException)
      {
          if (parameters.Speed is not null)
          {
              return await CreateMessageWithFastFallback(
                  parameters with { Speed = null });
          }
          throw;
      }
      catch (Anthropic5xxException)
      {
          if (maxAttempts > 1)
          {
              return await CreateMessageWithFastFallback(
                  parameters, maxAttempts: maxAttempts - 1);
          }
          throw;
      }
  }

  var message = await CreateMessageWithFastFallback(
      new MessageCreateParams
      {
          Model = "claude-opus-5",
          MaxTokens = 1024,
          Messages = [new() { Role = Role.User, Content = "Hello" }],
          Betas = ["fast-mode-2026-02-01"],
          Speed = Speed.Fast,
      },
      maxRetries: 0);
  ```

  ```go Go
  func createMessageWithFastFallback(
  	ctx context.Context,
  	client *anthropic.Client,
  	params anthropic.BetaMessageNewParams,
  	maxAttempts int,
  	opts ...option.RequestOption,
  ) (*anthropic.BetaMessage, error) {
  	message, err := client.Beta.Messages.New(ctx, params, opts...)
  	if err != nil {
  		var apierr *anthropic.Error
  		if errors.As(err, &apierr) && apierr.StatusCode == 429 && params.Speed != "" {
  			params.Speed = ""
  			return createMessageWithFastFallback(ctx, client, params, maxAttempts)
  		}
  		if (errors.As(err, &apierr) && apierr.StatusCode >= 500) || !errors.As(err, &apierr) {
  			if maxAttempts > 1 {
  				return createMessageWithFastFallback(ctx, client, params, maxAttempts-1)
  			}
  		}
  		return nil, err
  	}
  	return message, nil
  }

  func main() {
  	client := anthropic.NewClient()
  	message, err := createMessageWithFastFallback(
  		context.TODO(),
  		&client,
  		anthropic.BetaMessageNewParams{
  			Model:     anthropic.ModelClaudeOpus5,
  			MaxTokens: 1024,
  			Messages: []anthropic.BetaMessageParam{
  				anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello")),
  			},
  			Speed: anthropic.BetaMessageNewParamsSpeedFast,
  			Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaFastMode2026_02_01},
  		},
  		3,
  		option.WithMaxRetries(0),
  	)
  	if err != nil {
  		panic(err)
  	}
  	fmt.Println(message)
  }
  ```

  ```java Java
  import com.anthropic.errors.InternalServerException;
  import com.anthropic.errors.RateLimitException;
  // ...
  // 禁用 SDK 自动重试，以便由下方的回退逻辑处理
  AnthropicClient client =
          AnthropicOkHttpClient.builder().fromEnv().maxRetries(0).build();

  BetaMessage createMessageWithFastFallback(
          MessageCreateParams params, int maxAttempts) {
      try {
          return client.beta().messages().create(params);
      } catch (RateLimitException e) {
          if (params.speed().isPresent()) {
              MessageCreateParams retryParams = params.toBuilder()
                      .speed(Optional.empty())
                      .build();
              return createMessageWithFastFallback(retryParams, maxAttempts);
          }
          throw e;
      } catch (InternalServerException e) {
          if (maxAttempts > 1) {
              return createMessageWithFastFallback(params, maxAttempts - 1);
          }
          throw e;
      }
  }

  void main() {
      BetaMessage message = createMessageWithFastFallback(
              MessageCreateParams.builder()
                      .model(Model.CLAUDE_OPUS_5)
                      .maxTokens(1024L)
                      .addUserMessage("Hello")
                      .addBeta(AnthropicBeta.FAST_MODE_2026_02_01)
                      .speed(MessageCreateParams.Speed.FAST)
                      .build(),
              3);
      message.content().stream()
              .flatMap(block -> block.text().stream())
              .forEach(textBlock -> IO.println(textBlock.text()));
  }
  ```

  ```php PHP
  use Anthropic\Core\Exceptions\APIConnectionException;
  use Anthropic\Core\Exceptions\InternalServerException;
  use Anthropic\Core\Exceptions\RateLimitException;
  use Anthropic\RequestOptions;
  // ...
  $client = new Client();

  function createMessageWithFastFallback(
      Client $client,
      array $params,
      ?RequestOptions $requestOptions = null,
      int $maxAttempts = 3,
  ) {
      try {
          return $client->beta->messages->create(
              ...$params,
              requestOptions: $requestOptions,
          );
      } catch (RateLimitException $e) {
          if (isset($params['speed'])) {
              unset($params['speed']);
              return createMessageWithFastFallback($client, $params);
          }
          throw $e;
      } catch (InternalServerException | APIConnectionException $e) {
          if ($maxAttempts > 1) {
              return createMessageWithFastFallback(
                  $client, $params, maxAttempts: $maxAttempts - 1
              );
          }
          throw $e;
      }
  }

  $message = createMessageWithFastFallback(
      $client,
      [
          'model' => 'claude-opus-5',
          'maxTokens' => 1024,
          'messages' => [['role' => 'user', 'content' => 'Hello']],
          'betas' => ['fast-mode-2026-02-01'],
          'speed' => 'fast',
      ],
      RequestOptions::with(maxRetries: 0),
  );
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  def create_message_with_fast_fallback(client, request_options: {}, max_attempts: 3, **params)
    client.beta.messages.create(**params, request_options: request_options)
  rescue Anthropic::Errors::RateLimitError
    raise unless params[:speed] == "fast"
    params.delete(:speed)
    create_message_with_fast_fallback(client, **params)
  rescue Anthropic::Errors::InternalServerError, Anthropic::Errors::APIConnectionError
    raise unless max_attempts > 1
    create_message_with_fast_fallback(client, max_attempts: max_attempts - 1, **params)
  end

  message = create_message_with_fast_fallback(
    client,
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello" }],
    betas: ["fast-mode-2026-02-01"],
    speed: "fast",
    request_options: { max_retries: 0 }
  )
  ```
</CodeGroup>

## 注意事项

* **提示缓存：** 在快速速度和标准速度之间切换会使提示缓存失效。不同速度的请求不共享缓存前缀。
* **支持的模型：** Claude Opus 5 和 Claude Opus 4.8 支持快速模式。请参阅[支持的模型](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#supported-models)。
* **TTFT：** 快速模式的优势集中在每秒输出令牌数（OTPS），而非首令牌时间（TTFT）。
* **Batch API：** 快速模式不适用于 [Batch API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)。
* **Priority Tier：** 快速模式不适用于 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers) 承诺。
* **Claude Platform on AWS：** 快速模式目前在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上不可用。

## 后续步骤

<CardGroup cols={2}>
  <Card title="结构化输出" icon="code-brackets" href="https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs">
    从智能体工作流中获取经过验证的 JSON 结果。
  </Card>

  <Card title="定价" icon="calculator" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing#fast-mode-pricing">
    了解 Anthropic 针对模型和功能的定价结构。
  </Card>

  <Card title="Effort" icon="gauge" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    使用 effort 参数控制 Claude 在响应时使用多少令牌，在响应的详尽程度与令牌效率之间进行权衡。
  </Card>

  <Card title="流式传输消息" icon="arrow-right" href="https://platform.claude.com/docs/zh-CN/build-with-claude/streaming">
    通过服务器发送事件以增量方式流式传输 Messages API 响应，包括文本、工具使用和扩展思考增量。
  </Card>
</CardGroup>
