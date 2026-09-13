---
title: 面向 Claude 的 API 使用入门
url: https://platform.claude.com/docs/zh-CN/claude_api_primer
description: 本指南旨在向 Claude 介绍使用 Claude API 的基础知识。它提供了关于模型 ID/基本 Messages API、工具使用、流式传输、思考的说明和示例，仅此而已。
---

# 面向 Claude 的 API 使用入门

> 本指南旨在向 Claude 介绍使用 Claude API 的基础知识。它提供了关于模型 ID/基本 Messages API、工具使用、流式传输、思考的说明和示例，仅此而已。

## 模型

```text wrap
Recommended default for most work, including complex agentic coding: Claude Opus 5: claude-opus-5
Step up for the hardest long-running agentic and research tasks, at 2x Claude Opus 5 pricing: Claude Fable 5.1: claude-fable-5-1
Previous Opus model: Claude Opus 4.8: claude-opus-4-8
Smart model: Claude Sonnet 5: claude-sonnet-5
For fast, cost-effective tasks: Claude Haiku 4.5: claude-haiku-4-5-20251001
```

## 调用 API

### 基本请求和响应

<CodeGroup exclude="shell:cURL, typescript, csharp, go, java, php, ruby">
  ```bash CLI
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{"role": "user", "content": "Hello, Claude"}'
  ```

  ```python Python
  import anthropic

  message = anthropic.Anthropic().messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello, Claude"}],
  )
  print(message)
  ```
</CodeGroup>

```json Output
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Hello!"
    }
  ],
  "model": "claude-opus-5",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 12,
    "output_tokens": 6
  }
}
```

### 多轮对话

Messages API 是无状态的，这意味着您始终需要将完整的对话历史发送给 API。您可以使用这种模式随时间逐步构建对话。较早的对话轮次不一定需要真正来自 Claude。您可以使用合成的 `assistant` 消息。

<CodeGroup exclude="shell:cURL, typescript, csharp, go, java, php, ruby">
  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content: Hello, Claude
    - role: assistant
      content: Hello!
    - role: user
      content: Can you describe LLMs to me?
  YAML
  ```

  ```python Python
  import anthropic

  message = anthropic.Anthropic().messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {"role": "user", "content": "Hello, Claude"},
          {"role": "assistant", "content": "Hello!"},
          {"role": "user", "content": "Can you describe LLMs to me?"},
      ],
  )
  print(message)
  ```
</CodeGroup>

### 预填充 Claude 的响应

您可以在输入消息列表的最后一个位置预填充 Claude 响应的一部分。使用此技术可以塑造 Claude 的响应。以下示例使用 `"max_tokens": 1` 从 Claude 获取单个多项选择答案。

<Note>
  Claude 4.6 及更高版本的模型以及 Claude Mythos Preview 不支持 assistant 消息预填充；对这些模型的请求必须以 user 消息结尾。下面的示例使用支持预填充的模型。
</Note>

<CodeGroup exclude="shell:cURL, typescript, csharp, go, java, php, ruby">
  ```bash CLI
  ant messages create <<'YAML'
  model: claude-sonnet-4-5
  max_tokens: 1
  messages:
    - role: user
      content: "What is latin for Ant? (A) Apoidea, (B) Rhopalocera, (C) Formicidae"
    - role: assistant
      content: "The answer is ("
  YAML
  ```

  ```python Python
  import anthropic

  message = anthropic.Anthropic().messages.create(
      model="claude-sonnet-4-5",
      max_tokens=1,
      messages=[
          {
              "role": "user",
              "content": "What is latin for Ant? (A) Apoidea, (B) Rhopalocera, (C) Formicidae",
          },
          {"role": "assistant", "content": "The answer is ("},
      ],
  )
  print(message.content[0].text)
  ```
</CodeGroup>

### 视觉

Claude 可以读取请求中的文本和图像。图像支持 `base64` 和 `url` 两种来源类型，以及 `image/jpeg`、`image/png`、`image/gif` 和 `image/webp` 媒体类型。

<CodeGroup exclude="shell:cURL, typescript, csharp, go, java, php, ruby">
  ```bash CLI
  IMAGE_URL="https://platform.claude.com/docs/images/vision-example.jpg"

  # 选项 1：Base64 编码的图像（@ 前缀会自动将二进制文件编码为 base64）
  curl -sSo vision-example.jpg "$IMAGE_URL"

  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: image
          source:
            type: base64
            media_type: image/jpeg
            data: "@./vision-example.jpg"
        - type: text
          text: What is in the above image?
  YAML

  # 选项 2：通过 URL 引用的图像
  ant messages create <<YAML
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: image
          source:
            type: url
            url: $IMAGE_URL
        - type: text
          text: What is in the above image?
  YAML
  ```

  ```python Python
  import anthropic
  import base64
  import httpx2

  # 选项 1：Base64 编码的图像
  image_url = "https://platform.claude.com/docs/images/vision-example.jpg"
  image_media_type = "image/jpeg"
  image_data = base64.standard_b64encode(httpx2.get(image_url).content).decode("utf-8")

  message = anthropic.Anthropic().messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "image",
                      "source": {
                          "type": "base64",
                          "media_type": image_media_type,
                          "data": image_data,
                      },
                  },
                  {"type": "text", "text": "What is in the above image?"},
              ],
          }
      ],
  )
  print(next(block.text for block in message.content if block.type == "text"))

  # 选项 2：通过 URL 引用的图像
  message_from_url = anthropic.Anthropic().messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "image",
                      "source": {
                          "type": "url",
                          "url": "https://platform.claude.com/docs/images/vision-example.jpg",
                      },
                  },
                  {"type": "text", "text": "What is in the above image?"},
              ],
          }
      ],
  )
  print(next(block.text for block in message_from_url.content if block.type == "text"))
  ```
</CodeGroup>

## 思考

思考有时可以帮助 Claude 处理非常困难的任务。当前的机制是 [adaptive thinking（自适应思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)（`thinking: {"type": "adaptive"}`）：由 Claude 决定何时思考以及思考多少，您通过 [`effort`](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 参数而非令牌预算来引导思考深度。Claude 4.6 及更高版本的模型以及 Claude Mythos Preview 支持自适应思考。在 Claude 5 模型和 Claude Mythos Preview 上，当省略 `thinking` 参数时，思考默认开启。

在所有模型上，只要启用了思考，temperature 就必须设置为 1（或保持未设置）。在 Claude 4.7 及更高版本的模型以及 Claude Mythos Preview 上，`temperature` 已弃用，仅接受其默认值，即使思考处于关闭状态也是如此。

以下模型支持思考：

* Claude Opus 5（claude-opus-5，仅支持自适应思考，默认开启）
* Claude Sonnet 5（`claude-sonnet-5`，仅支持自适应思考，默认开启）
* Claude Opus 4.8（claude-opus-4-8，仅支持自适应思考）
* Claude Opus 4.7（`claude-opus-4-7`，仅支持自适应思考）
* Claude Opus 4.6（`claude-opus-4-6`，自适应思考或旧版手动思考）
* Claude Sonnet 4.6（`claude-sonnet-4-6`，自适应思考或旧版手动思考）
* Claude Opus 4.5（`claude-opus-4-5-20251101`，仅支持旧版手动思考）
* Claude Sonnet 4.5（`claude-sonnet-4-5-20250929`，仅支持旧版手动思考）
* Claude Haiku 4.5（`claude-haiku-4-5-20251001`，仅支持旧版手动思考）

<Note>
  在 Claude 4.7 及更高版本的模型上，不支持手动 extended thinking（扩展思考）（带有 `budget_tokens` 值的 `type: enabled`），并会返回 400 错误。请改用 [自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)（`type: adaptive`）。
</Note>

### 思考的工作原理

当思考开启时，Claude 会创建 `thinking` 内容块，在其中输出其内部推理。API 响应包含 `thinking` 内容块，后跟 `text` 内容块。

<CodeGroup exclude="shell:cURL, typescript, csharp, go, java, php, ruby">
  ```bash CLI
  ant messages create --transform content --format yaml <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  thinking:
    type: adaptive
    display: summarized
  messages:
    - role: user
      content: Are there an infinite number of prime numbers such that n mod 4 == 3?
  YAML
  ```

  ```python Python
  import anthropic

  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      thinking={"type": "adaptive", "display": "summarized"},
      messages=[
          {
              "role": "user",
              "content": "Are there an infinite number of prime numbers such that n mod 4 == 3?",
          }
      ],
  )

  # 响应包含摘要化的思考块和文本块
  for block in response.content:
      if block.type == "thinking":
          print(f"\nThinking summary: {block.thinking}")
      elif block.type == "text":
          print(f"\nResponse: {block.text}")
  ```
</CodeGroup>

手动扩展思考（`thinking: {"type": "enabled", "budget_tokens": N}`）是旧版机制。它仅适用于支持思考的 Claude 4 至 4.6 模型；Claude 4.7 及更高版本的模型会以 400 错误拒绝 `type: enabled`，并改用 [自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。使用手动扩展思考时，`budget_tokens` 设置 Claude 允许用于其内部推理过程的最大令牌数；该限制适用于完整的思考令牌，而非摘要输出。除非您使用 [交错思考](https://platform.claude.com/docs/zh-CN/claude_api_primer#interleaved-thinking)，否则 `budget_tokens` 必须小于 `max_tokens`，以便 Claude 在思考完成后有空间撰写其响应。

## 思考与工具使用

思考可以与 tool use（工具使用）一起使用，使 Claude 能够对工具选择和结果处理进行推理。

重要限制：

1. **工具选择限制：** 仅支持 `tool_choice: {"type": "auto"}`（默认）或 `tool_choice: {"type": "none"}`。
2. **保留思考块：** 在工具使用期间，您必须将最后一条 assistant 消息的 `thinking` 块传回 API。

### 保留思考块

<CodeGroup exclude="shell:cURL, typescript, csharp, go, java, php, ruby">
  ```bash CLI
  # 第一次请求：捕获助手的 content 数组（thinking + tool_use
  # 块，签名保持完整）并保存为紧凑的 JSON。
  ASSISTANT_CONTENT=$(ant messages create \
    --transform content --format jsonl <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  thinking:
    type: adaptive
    display: summarized
  tools:
    - name: get_weather
      description: Get the current weather for a location.
      input_schema:
        type: object
        properties:
          location:
            type: string
            description: The city name.
        required: [location]
  messages:
    - role: user
      content: "What's the weather in Paris?"
  YAML
  )

  TOOL_USE_ID=$(printf '%s' "$ASSISTANT_CONTENT" \
    | jq -r '.[] | select(.type == "tool_use") | .id')

  # 第二次请求：将捕获的块原样作为助手
  # 消息传回。thinking 块必须与 tool_use 块一同传递。
  ant messages create <<YAML
  model: claude-opus-5
  max_tokens: 16000
  thinking:
    type: adaptive
    display: summarized
  tools:
    - name: get_weather
      description: Get the current weather for a location.
      input_schema:
        type: object
        properties:
          location:
            type: string
            description: The city name.
        required: [location]
  messages:
    - role: user
      content: "What's the weather in Paris?"
    - role: assistant
      content: $ASSISTANT_CONTENT
    - role: user
      content:
        - type: tool_result
          tool_use_id: $TOOL_USE_ID
          content: "Current temperature: 72°F"
  YAML
  ```

  ```python Python
  import anthropic

  client = anthropic.Anthropic()

  weather_tool = {
      "name": "get_weather",
      "description": "Get the current weather for a location.",
      "input_schema": {
          "type": "object",
          "properties": {"location": {"type": "string", "description": "The city name."}},
          "required": ["location"],
      },
  }

  weather_data = {"temperature": 72}

  # 第一次请求 - Claude 返回思考内容和工具请求
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      thinking={"type": "adaptive", "display": "summarized"},
      tools=[weather_tool],
      messages=[{"role": "user", "content": "What's the weather in Paris?"}],
  )

  # 提取思考块和工具使用块
  thinking_block = next(
      (block for block in response.content if block.type == "thinking"), None
  )
  tool_use_block = next(
      (block for block in response.content if block.type == "tool_use"), None
  )

  # 第二次请求 - 包含思考块和工具结果
  continuation = client.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      thinking={"type": "adaptive", "display": "summarized"},
      tools=[weather_tool],
      messages=[
          {"role": "user", "content": "What's the weather in Paris?"},
          # 请注意，thinking_block 与 tool_use_block 一同被传入
          {"role": "assistant", "content": [thinking_block, tool_use_block]},
          {
              "role": "user",
              "content": [
                  {
                      "type": "tool_result",
                      "tool_use_id": tool_use_block.id,
                      "content": f"Current temperature: {weather_data['temperature']}°F",
                  }
              ],
          },
      ],
  )

  for block in continuation.content:
      if block.type == "text":
          print(block.text)
  ```
</CodeGroup>

### 交错思考

Interleaved thinking（交错思考）使 Claude 能够在工具调用之间进行思考，在决定下一步之前对工具结果进行推理。

<Info>
  在具有 [自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)（`thinking: {type: "adaptive"}`）的模型上，交错思考会自动启用。无需 beta 标头。Sonnet 4.6 既支持配合手动扩展思考使用 `interleaved-thinking-2025-05-14` beta 标头，也支持自适应思考。
</Info>

在使用手动扩展思考的较旧模型（Claude 4、4.5 和 Sonnet 4.6 模型）上，通过在 API 请求中添加 beta 标头 `interleaved-thinking-2025-05-14` 来启用交错思考：

<CodeGroup exclude="shell:cURL, typescript, csharp, go, java, php, ruby">
  ```bash CLI
  ant beta:messages create --beta interleaved-thinking-2025-05-14 <<'YAML'
  model: claude-sonnet-4-6
  max_tokens: 16000
  thinking:
    type: enabled
    budget_tokens: 10000
  tools:
    - name: calculator
      description: Perform arithmetic calculations.
      input_schema:
        type: object
        properties:
          expression:
            type: string
            description: The math expression to evaluate.
        required:
          - expression
    - name: database_query
      description: Query the product database.
      input_schema:
        type: object
        properties:
          query:
            type: string
            description: The database query.
        required:
          - query
  messages:
    - role: user
      content: "What's the total revenue if we sold 150 units of product A at $50 each?"
  YAML
  ```

  ```python Python
  import anthropic

  client = anthropic.Anthropic()

  calculator_tool = {
      "name": "calculator",
      "description": "Perform arithmetic calculations.",
      "input_schema": {
          "type": "object",
          "properties": {
              "expression": {
                  "type": "string",
                  "description": "The math expression to evaluate.",
              }
          },
          "required": ["expression"],
      },
  }

  database_tool = {
      "name": "database_query",
      "description": "Query the product database.",
      "input_schema": {
          "type": "object",
          "properties": {
              "query": {"type": "string", "description": "The database query."}
          },
          "required": ["query"],
      },
  }

  response = client.beta.messages.create(
      model="claude-sonnet-4-6",
      max_tokens=16000,
      thinking={"type": "enabled", "budget_tokens": 10000},
      tools=[calculator_tool, database_tool],
      messages=[
          {
              "role": "user",
              "content": "What's the total revenue if we sold 150 units of product A at $50 each?",
          }
      ],
      betas=["interleaved-thinking-2025-05-14"],
  )

  for block in response.content:
      if block.type == "thinking":
          print(f"Thinking: {block.thinking}")
      elif block.type == "tool_use":
          print(f"Tool call: {block.name}({block.input})")
      elif block.type == "text":
          print(f"Response: {block.text}")
  ```
</CodeGroup>

使用交错思考时，且仅在使用交错思考时（而非常规的手动扩展思考），`budget_tokens` 可以超过 `max_tokens` 参数，因为在这种情况下 `budget_tokens` 表示一个 assistant 轮次内所有思考块的总预算。

## 工具使用

### 指定客户端工具

客户端工具在 API 请求的 `tools` 顶级参数中指定。每个工具定义包括：

| 参数             | 描述                                                       |
| -------------- | -------------------------------------------------------- |
| `name`         | 工具的名称。必须匹配正则表达式 `^[a-zA-Z0-9_-]{1,64}$`。                 |
| `description`  | 关于工具的功能、何时应使用以及其行为方式的详细纯文本描述。                            |
| `input_schema` | 一个 [JSON Schema](https://json-schema.org/) 对象，定义工具的预期参数。 |

```json
{
  "name": "get_weather",
  "description": "Get the current weather in a given location",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "The city and state, e.g. San Francisco, CA"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "The unit of temperature, either 'celsius' or 'fahrenheit'"
      }
    },
    "required": ["location"]
  }
}
```

### 工具定义的最佳实践

**提供极其详细的描述。** 这是迄今为止影响工具性能的最重要因素。您的描述应解释有关工具的每个细节，包括：

* 工具的功能
* 何时应使用（以及何时不应使用）
* 每个参数的含义及其如何影响工具的行为
* 任何重要的注意事项或限制

**对于复杂工具，考虑使用 `input_examples`。** 对于具有嵌套对象、可选参数或格式敏感输入的工具，您可以使用 `input_examples` 字段（beta）提供具体示例。这有助于 Claude 理解预期的输入模式。详情请参阅 [提供工具使用示例](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/define-tools#providing-tool-use-examples)。

良好工具描述的示例：

```json
{
  "name": "get_stock_price",
  "description": "Retrieves the current stock price for a given ticker symbol. The ticker symbol must be a valid symbol for a publicly traded company on a major US stock exchange like NYSE or NASDAQ. The tool will return the latest trade price in USD. It should be used when the user asks about the current or most recent price of a specific stock. It will not provide any other information about the stock or company.",
  "input_schema": {
    "type": "object",
    "properties": {
      "ticker": {
        "type": "string",
        "description": "The stock ticker symbol, e.g. AAPL for Apple Inc."
      }
    },
    "required": ["ticker"]
  }
}
```

## 控制 Claude 的输出

### 强制工具使用

您可以通过在 `tool_choice` 字段中指定工具来强制 Claude 使用特定工具：

```python
tool_choice = {"type": "tool", "name": "get_weather"}
```

使用 `tool_choice` 参数时，有四个可能的选项：

* `auto` 允许 Claude 决定是否调用任何提供的工具（默认）。
* `any` 告诉 Claude 它必须使用所提供工具中的一个。
* `tool` 强制 Claude 始终使用特定工具。
* `none` 阻止 Claude 使用任何工具。

在 Claude Fable 5.1 和 Claude Mythos 5.1 上，`any` 和 `tool` 会返回 400 错误。请将 `tool_choice` 保持为 `auto`，并在工具定义上设置 `"strict": true`，以保证 Claude 发出的任何调用都符合该工具的 `input_schema`。请参阅 [严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)。

### JSON 输出

工具不一定需要是客户端函数。只要您希望模型返回遵循所提供 schema 的 JSON 输出，就可以随时使用工具。

### 思维链

使用工具时，Claude 通常会展示其"chain of thought"（思维链），即它用来分解问题并确定使用哪些工具的逐步推理。

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "<thinking>To answer this question, I will: 1. Use the get_weather tool to get the current weather in San Francisco. 2. Use the get_time tool to get the current time in the America/Los_Angeles timezone, which covers San Francisco, CA.</thinking>"
    },
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw90lq917835lq9",
      "name": "get_weather",
      "input": { "location": "San Francisco, CA" }
    }
  ]
}
```

### 并行工具使用

默认情况下，Claude 可能会使用多个工具来回答用户查询。您可以通过设置 `disable_parallel_tool_use=true` 来禁用此行为。

## 处理工具使用和工具结果内容块

### 处理来自客户端工具的结果

响应的 `stop_reason` 为 `tool_use`，并包含一个或多个 `tool_use` 内容块，其中包括：

* `id`：此特定工具使用块的唯一标识符。
* `name`：正在使用的工具的名称。
* `input`：一个包含传递给工具的输入的对象。

当您收到工具使用响应时，您应该：

1. 从 `tool_use` 块中提取 `name`、`id` 和 `input`。
2. 在您的代码库中运行与该工具名称对应的实际工具。
3. 通过发送包含 `tool_result` 的新消息来继续对话：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "15 degrees"
    }
  ]
}
```

### 处理 `max_tokens` 停止原因

如果 Claude 的响应在工具使用期间因达到 `max_tokens` 限制而被截断，请使用更高的 `max_tokens` 值重试请求。

### 处理 `pause_turn` 停止原因

使用网络搜索等服务器工具时，API 可能会返回 `pause_turn` 停止原因。通过在后续请求中按原样传回已暂停的响应来继续对话。

## 错误排查

### 工具执行错误

如果工具本身在执行期间抛出错误，请返回带有 `"is_error": true` 的错误消息：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "ConnectionError: the weather service API is not available (HTTP 500)",
      "is_error": true
    }
  ]
}
```

### 无效的工具名称

如果 Claude 尝试使用工具的方式无效（例如，缺少必需参数），请在工具定义中使用更详细的 `description` 值重试请求。

## 流式传输消息

创建 Message 时，您可以设置 `"stream": true`，以使用 server-sent events（服务器发送事件），即 SSE，增量地进行 streaming（流式传输）响应。

### 使用 SDK 进行流式传输

<CodeGroup exclude="shell:cURL, typescript, csharp, go, java, php, ruby">
  ```bash CLI
  ant messages create --stream --format jsonl \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello"}' \
    | jq -rj 'select(.delta.type? == "text_delta") | .delta.text'
  ```

  ```python Python
  import anthropic

  client = anthropic.Anthropic()

  with client.messages.stream(
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello"}],
      model="claude-opus-5",
  ) as stream:
      for text in stream.text_stream:
          print(text, end="", flush=True)
  ```
</CodeGroup>

### 事件类型

每个服务器发送事件都包含一个命名的事件类型和关联的 JSON 数据。每个流使用以下事件流程：

1. `message_start`：包含一个 `content` 为空的 `Message` 对象。
2. 一系列内容块，每个内容块都有 `content_block_start`、一个或多个 `content_block_delta` 事件以及 `content_block_stop`。
3. 一个或多个 `message_delta` 事件，表示对最终 `Message` 对象的顶级更改。
4. 最后一个 `message_stop` 事件。

**警告：** `message_delta` 事件的 `usage` 字段中显示的令牌计数是*累计的*。

### 内容块增量类型

#### 文本增量

```json
{
  "type": "content_block_delta",
  "index": 0,
  "delta": { "type": "text_delta", "text": "Hello frien" }
}
```

#### 输入 JSON 增量

对于 `tool_use` 内容块，增量是*部分 JSON 字符串*：

```json
{"type": "content_block_delta","index": 1,"delta": {"type": "input_json_delta","partial_json": "{\"location\": \"San Fra"}}}
```

#### 思考增量

在流式传输中使用思考时：

```json
{
  "type": "content_block_delta",
  "index": 0,
  "delta": {
    "type": "thinking_delta",
    "thinking": "Let me solve this step by step..."
  }
}
```

### 基本流式传输请求示例

```sse
event: message_start
data: {"type": "message_start", "message": {"id": "msg_1nZdL29xx5MUA1yADyHTEsnR8uuvGzszyY", "type": "message", "role": "assistant", "content": [], "model": "claude-opus-5", "stop_reason": null, "stop_sequence": null, "usage": {"input_tokens": 25, "output_tokens": 1}}}

event: content_block_start
data: {"type": "content_block_start", "index": 0, "content_block": {"type": "text", "text": ""}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "Hello"}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "!"}}

event: content_block_stop
data: {"type": "content_block_stop", "index": 0}

event: message_delta
data: {"type": "message_delta", "delta": {"stop_reason": "end_turn", "stop_sequence":null}, "usage": {"output_tokens": 15}}

event: message_stop
data: {"type": "message_stop"}
```
