---
title: OpenAI SDK 兼容性
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/openai-sdk
description: Anthropic 提供了一个兼容层，使您能够使用 OpenAI SDK 来测试 Claude API。只需少量代码更改，您就可以快速评估 Anthropic 模型的能力。
---

<Note>
  此兼容层主要用于测试和比较模型能力，对于大多数用例而言，不应被视为长期或生产就绪的解决方案。虽然我们计划保持其功能完整且不引入破坏性变更，但优先保障的是 [Claude API](https://platform.claude.com/docs/zh-CN/api/overview) 的可靠性和有效性。

  有关已知兼容性限制的更多信息，请参阅[重要的 OpenAI 兼容性限制](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/openai-sdk#important-openai-compatibility-limitations)。

  如果您在使用 OpenAI SDK 兼容性功能时遇到任何问题，请通过此[兼容性反馈表单](https://forms.gle/oQV4McQNiuuNbz9n8)分享您的反馈。
</Note>

<Tip>
  为获得最佳体验并使用 Claude API 的完整功能集（[PDF 处理](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)、[引用](https://platform.claude.com/docs/zh-CN/build-with-claude/citations)、[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)和[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)），请使用原生 [Claude API](https://platform.claude.com/docs/zh-CN/api/overview)。
</Tip>

## OpenAI SDK 入门

要使用 OpenAI SDK 兼容性功能，您需要：

1. 使用官方 OpenAI SDK

2. 进行以下更改

   * 更新您的 base URL，使其指向 Claude API
   * 将您的 API 密钥替换为 [Claude API 密钥](https://platform.claude.com/settings/keys)
   * 如果您的密钥是可访问多个工作区的[个人密钥或服务账户密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)，还需在每个请求中发送 `anthropic-workspace-id` 请求头（例如，Python SDK 中的 `default_headers` 或 TypeScript 中的 `defaultHeaders`）；请参阅[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)
   * 更新您的模型名称以使用 [Claude 模型](https://platform.claude.com/docs/zh-CN/models/overview)

3. 查看以下各节以了解支持哪些功能

### 快速入门示例

<CodeGroup exclude="shell">
  ```python Python
  import os

  from openai import OpenAI

  client = OpenAI(
      api_key=os.environ.get("ANTHROPIC_API_KEY"),  # Your Claude API key
      base_url="https://api.anthropic.com/v1/",  # the Claude API endpoint
  )

  response = client.chat.completions.create(
      model="claude-opus-5",  # Claude model name
      messages=[
          {"role": "system", "content": "You are a helpful assistant."},
          {"role": "user", "content": "Who are you?"},
      ],
  )

  print(response.choices[0].message.content)
  ```

  ```typescript TypeScript
  import OpenAI from "openai";

  const openai = new OpenAI({
    apiKey: process.env.ANTHROPIC_API_KEY, // Your Claude API key
    baseURL: "https://api.anthropic.com/v1/" // Claude API endpoint
  });

  const response = await openai.chat.completions.create({
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: "Who are you?" }
    ],
    model: "claude-opus-5" // Claude model name
  });

  console.log(response.choices[0].message.content);
  ```

  ```csharp C#
  using System.ClientModel;
  using OpenAI;
  using OpenAI.Chat;

  ChatClient chatClient = new(
      model: "claude-opus-5", // Claude model name
      credential: new ApiKeyCredential(
          Environment.GetEnvironmentVariable("ANTHROPIC_API_KEY")), // Your Claude API key
      options: new OpenAIClientOptions()
      {
          Endpoint = new Uri("https://api.anthropic.com/v1/") // the Claude API endpoint
      });

  ChatCompletion completion = chatClient.CompleteChat(
      new SystemChatMessage("You are a helpful assistant."),
      new UserChatMessage("Who are you?"));

  Console.WriteLine(completion.Content[0].Text);
  ```

  ```go Go
  package main

  import (
  	"context"
  	"fmt"
  	"os"

  	"github.com/openai/openai-go/v3"
  	"github.com/openai/openai-go/v3/option"
  )

  func main() {
  	client := openai.NewClient(
  		option.WithAPIKey(os.Getenv("ANTHROPIC_API_KEY")),   // Your Claude API key
  		option.WithBaseURL("https://api.anthropic.com/v1/"), // the Claude API endpoint
  	)

  	response, err := client.Chat.Completions.New(context.Background(), openai.ChatCompletionNewParams{
  		Model: "claude-opus-5", // Claude model name
  		Messages: []openai.ChatCompletionMessageParamUnion{
  			openai.SystemMessage("You are a helpful assistant."),
  			openai.UserMessage("Who are you?"),
  		},
  	})
  	if err != nil {
  		panic(err)
  	}

  	fmt.Println(response.Choices[0].Message.Content)
  }
  ```

  ```java Java
  import com.openai.client.OpenAIClient;
  import com.openai.client.okhttp.OpenAIOkHttpClient;
  import com.openai.models.chat.completions.ChatCompletion;
  import com.openai.models.chat.completions.ChatCompletionCreateParams;

  public class QuickStart {
      public static void main(String[] args) {
          OpenAIClient client = OpenAIOkHttpClient.builder()
                  .apiKey(System.getenv("ANTHROPIC_API_KEY")) // Your Claude API key
                  .baseUrl("https://api.anthropic.com/v1/") // the Claude API endpoint
                  .build();

          ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
                  .model("claude-opus-5") // Claude model name
                  .addSystemMessage("You are a helpful assistant.")
                  .addUserMessage("Who are you?")
                  .build();

          ChatCompletion completion = client.chat().completions().create(params);
          System.out.println(completion.choices().get(0).message().content().orElse(""));
      }
  }
  ```

  ```php PHP
  <?php
  // OpenAI 没有官方的 PHP SDK，因此此处不提供示例。
  // 如需在 PHP 中使用 Claude，请改用原生 Claude API：
  // https://platform.claude.com/docs/en/cli-sdks-libraries/overview
  ```

  ```ruby Ruby
  require "openai"

  openai = OpenAI::Client.new(
    api_key: ENV["ANTHROPIC_API_KEY"], # Your Claude API key
    base_url: "https://api.anthropic.com/v1/" # the Claude API endpoint
  )

  response = openai.chat.completions.create(
    model: "claude-opus-5", # Claude model name
    messages: [
      {role: "system", content: "You are a helpful assistant."},
      {role: "user", content: "Who are you?"}
    ]
  )

  puts response.choices.first.message.content
  ```
</CodeGroup>

## 重要的 OpenAI 兼容性限制

### API 行为

以下是与使用 OpenAI 相比最显著的差异：

* 函数调用的 `strict` 参数会被忽略，这意味着工具使用的 JSON 不保证遵循所提供的 schema。如需保证 schema 一致性，请使用原生 [Claude API 的结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)。
* 不支持音频输入；它将被忽略并从输入中移除
* 不支持提示缓存，但 [Anthropic SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview) 中支持该功能
* 系统/开发者消息会被提升并拼接到对话的开头，因为 Anthropic 仅支持单条初始系统消息。

大多数不受支持的字段会被静默忽略，而不会产生错误。这些内容均在以下各节中有详细说明。

### 输出质量注意事项

如果您对提示进行了大量调整，那么它很可能是专门针对 OpenAI 调优的。请考虑参照[提示最佳实践指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)为 Claude 重新调整提示。

### 系统/开发者消息提升

OpenAI SDK 的大多数输入都能直接映射到 Anthropic 的 API 参数，但一个明显的区别在于对 system prompt（系统提示）/开发者提示的处理。通过 OpenAI，这两种提示可以放置在聊天对话的任意位置。由于 Anthropic 仅支持一条初始系统消息，API 会获取所有系统/开发者消息，并用单个换行符（`\n`）将它们拼接在一起。然后，这个完整的字符串会作为单条系统消息放在消息列表的开头。

### 思考支持

您可以通过添加 `thinking` 参数来启用[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。在当前模型上，思考是自适应的，由 Claude 决定何时思考以及思考的深度；在 Claude 5 模型上，思考默认开启；手动配置的 extended thinking（扩展思考）是一种旧版模式。尽管思考能够提升 Claude 在复杂任务上的推理能力，但 OpenAI SDK 不会返回 Claude 的详细思考过程。如需完整的思考功能（包括访问 Claude 的逐步推理输出），请使用原生 Claude API。

<CodeGroup exclude="shell">
  ```python Python
  response = client.chat.completions.create(
      model="claude-sonnet-4-6",
      messages=[{"role": "user", "content": "Who are you?"}],
      extra_body={"thinking": {"type": "enabled", "budget_tokens": 2000}},
  )
  ```

  ```typescript TypeScript
  const response = await openai.chat.completions.create({
    messages: [{ role: "user", content: "Who are you?" }],
    model: "claude-sonnet-4-6",
    // @ts-expect-error
    thinking: { type: "enabled", budget_tokens: 2000 }
  });
  ```

  ```csharp C#
  // .NET SDK 没有类似 Python 的 extra_body 参数，因此本示例
  // 使用 SDK 文档中记载的协议方法发送 thinking 参数
  // （即原始 JSON 请求体）。
  BinaryData input = BinaryData.FromString("""
      {
        "model": "claude-sonnet-4-6",
        "messages": [{ "role": "user", "content": "Who are you?" }],
        "thinking": { "type": "enabled", "budget_tokens": 2000 }
      }
      """);

  using BinaryContent content = BinaryContent.Create(input);
  ClientResult result = chatClient.CompleteChat(content);
  ```

  ```go Go
  response, err := client.Chat.Completions.New(
  	context.Background(),
  	openai.ChatCompletionNewParams{
  		Model: "claude-sonnet-4-6",
  		Messages: []openai.ChatCompletionMessageParamUnion{
  			openai.UserMessage("Who are you?"),
  		},
  	},
  	option.WithJSONSet("thinking", map[string]any{"type": "enabled", "budget_tokens": 2000}),
  )
  ```

  ```java Java
  ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
          .model("claude-sonnet-4-6")
          .addUserMessage("Who are you?")
          .putAdditionalBodyProperty("thinking",
                  JsonValue.from(Map.of("type", "enabled", "budget_tokens", 2000)))
          .build();

  ChatCompletion completion = client.chat().completions().create(params);
  ```

  ```php PHP
  <?php
  // OpenAI 没有官方的 PHP SDK，因此此处不提供示例。
  // 如需在 PHP 中使用 Claude，请改用原生 Claude API：
  // https://platform.claude.com/docs/en/cli-sdks-libraries/overview
  ```

  ```ruby Ruby
  response = openai.chat.completions.create(
    model: "claude-sonnet-4-6",
    messages: [{role: "user", content: "Who are you?"}],
    request_options: {extra_body: {thinking: {type: "enabled", budget_tokens: 2000}}}
  )
  ```
</CodeGroup>

## 速率限制

速率限制遵循 Anthropic 针对 `/v1/messages` 端点的[标准限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)。

## 详细的 OpenAI 兼容 API 支持

### 请求字段

#### 简单字段

| 字段                      | 支持状态                                                                                                                  |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `model`                 | 使用 Claude 模型名称                                                                                                        |
| `max_tokens`            | 完全支持                                                                                                                  |
| `max_completion_tokens` | 完全支持                                                                                                                  |
| `stream`                | 完全支持                                                                                                                  |
| `stream_options`        | 完全支持                                                                                                                  |
| `top_p`                 | 完全支持                                                                                                                  |
| `parallel_tool_calls`   | 完全支持                                                                                                                  |
| `stop`                  | 所有非空白停止序列均有效                                                                                                          |
| `temperature`           | 介于 0 和 1 之间（含边界）。大于 1 的值将被限制为 1。                                                                                      |
| `n`                     | 必须恰好为 1                                                                                                               |
| `logprobs`              | 忽略                                                                                                                    |
| `metadata`              | 忽略                                                                                                                    |
| `response_format`       | 忽略。如需 JSON 输出，请配合原生 Claude API 使用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs) |
| `prediction`            | 忽略                                                                                                                    |
| `presence_penalty`      | 忽略                                                                                                                    |
| `frequency_penalty`     | 忽略                                                                                                                    |
| `seed`                  | 忽略                                                                                                                    |
| `service_tier`          | 忽略                                                                                                                    |
| `audio`                 | 忽略                                                                                                                    |
| `logit_bias`            | 忽略                                                                                                                    |
| `store`                 | 忽略                                                                                                                    |
| `user`                  | 忽略                                                                                                                    |
| `modalities`            | 忽略                                                                                                                    |
| `top_logprobs`          | 忽略                                                                                                                    |
| `reasoning_effort`      | 忽略                                                                                                                    |

#### `tools` / `functions` 字段

<Accordion title="显示字段">
  <Tabs>
    <Tab title="Tools">
      `tools[n].function` 字段

      | 字段            | 支持状态                                                                                                                       |
      | ------------- | -------------------------------------------------------------------------------------------------------------------------- |
      | `name`        | 完全支持                                                                                                                       |
      | `description` | 完全支持                                                                                                                       |
      | `parameters`  | 完全支持                                                                                                                       |
      | `strict`      | 忽略。如需严格的 schema 验证，请配合原生 Claude API 使用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs) |
    </Tab>

    <Tab title="Functions">
      `functions[n]` 字段

      <Info>
        OpenAI 已弃用 `functions` 字段，并建议改用 `tools`。
      </Info>

      | 字段            | 支持状态                                                                                                                       |
      | ------------- | -------------------------------------------------------------------------------------------------------------------------- |
      | `name`        | 完全支持                                                                                                                       |
      | `description` | 完全支持                                                                                                                       |
      | `parameters`  | 完全支持                                                                                                                       |
      | `strict`      | 忽略。如需严格的 schema 验证，请配合原生 Claude API 使用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs) |
    </Tab>
  </Tabs>
</Accordion>

#### `messages` 数组字段

<Accordion title="显示字段">
  <Tabs>
    <Tab title="Developer 角色">
      `messages[n].role == "developer"` 的字段

      <Info>
        开发者消息会作为初始系统消息的一部分被提升到对话开头
      </Info>

      | 字段        | 支持状态       |
      | --------- | ---------- |
      | `content` | 完全支持，但会被提升 |
      | `name`    | 忽略         |
    </Tab>

    <Tab title="System 角色">
      `messages[n].role == "system"` 的字段

      <Info>
        系统消息会作为初始系统消息的一部分被提升到对话开头
      </Info>

      | 字段        | 支持状态       |
      | --------- | ---------- |
      | `content` | 完全支持，但会被提升 |
      | `name`    | 忽略         |
    </Tab>

    <Tab title="User 角色">
      `messages[n].role == "user"` 的字段

      | 字段        | 变体                               | 子字段      | 支持状态 |
      | --------- | -------------------------------- | -------- | ---- |
      | `content` | `string`                         |          | 完全支持 |
      |           | `array`, `type == "text"`        |          | 完全支持 |
      |           | `array`, `type == "image_url"`   | `url`    | 完全支持 |
      |           |                                  | `detail` | 忽略   |
      |           | `array`, `type == "input_audio"` |          | 忽略   |
      |           | `array`, `type == "file"`        |          | 忽略   |
      | `name`    |                                  |          | 忽略   |
    </Tab>

    <Tab title="Assistant 角色">
      `messages[n].role == "assistant"` 的字段

      | 字段              | 变体                           | 支持状态 |
      | --------------- | ---------------------------- | ---- |
      | `content`       | `string`                     | 完全支持 |
      |                 | `array`, `type == "text"`    | 完全支持 |
      |                 | `array`, `type == "refusal"` | 忽略   |
      | `tool_calls`    |                              | 完全支持 |
      | `function_call` |                              | 完全支持 |
      | `audio`         |                              | 忽略   |
      | `refusal`       |                              | 忽略   |
    </Tab>

    <Tab title="Tool 角色">
      `messages[n].role == "tool"` 的字段

      | 字段             | 变体                        | 支持状态 |
      | -------------- | ------------------------- | ---- |
      | `content`      | `string`                  | 完全支持 |
      |                | `array`, `type == "text"` | 完全支持 |
      | `tool_call_id` |                           | 完全支持 |
      | `tool_choice`  |                           | 完全支持 |
      | `name`         |                           | 忽略   |
    </Tab>

    <Tab title="Function 角色">
      `messages[n].role == "function"` 的字段

      | 字段            | 变体                        | 支持状态 |
      | ------------- | ------------------------- | ---- |
      | `content`     | `string`                  | 完全支持 |
      |               | `array`, `type == "text"` | 完全支持 |
      | `tool_choice` |                           | 完全支持 |
      | `name`        |                           | 忽略   |
    </Tab>
  </Tabs>
</Accordion>

### 响应字段

| 字段                                | 支持状态    |
| --------------------------------- | ------- |
| `id`                              | 完全支持    |
| `choices[]`                       | 长度始终为 1 |
| `choices[].finish_reason`         | 完全支持    |
| `choices[].index`                 | 完全支持    |
| `choices[].message.role`          | 完全支持    |
| `choices[].message.content`       | 完全支持    |
| `choices[].message.tool_calls`    | 完全支持    |
| `object`                          | 完全支持    |
| `created`                         | 完全支持    |
| `model`                           | 完全支持    |
| `finish_reason`                   | 完全支持    |
| `content`                         | 完全支持    |
| `usage.completion_tokens`         | 完全支持    |
| `usage.prompt_tokens`             | 完全支持    |
| `usage.total_tokens`              | 完全支持    |
| `usage.completion_tokens_details` | 始终为空    |
| `usage.prompt_tokens_details`     | 始终为空    |
| `choices[].message.refusal`       | 始终为空    |
| `choices[].message.audio`         | 始终为空    |
| `logprobs`                        | 始终为空    |
| `service_tier`                    | 始终为空    |
| `system_fingerprint`              | 始终为空    |

### 错误消息兼容性

兼容层与 OpenAI API 保持一致的错误格式。但是，详细的错误消息不会完全相同。请仅将错误消息用于日志记录和调试。

### 请求头兼容性

虽然 OpenAI SDK 会自动管理请求头，但以下是 Claude API 支持的完整请求头列表，供需要直接处理它们的开发者参考。

| 请求头                              | 支持状态             |
| -------------------------------- | ---------------- |
| `x-ratelimit-limit-requests`     | 完全支持             |
| `x-ratelimit-limit-tokens`       | 完全支持             |
| `x-ratelimit-remaining-requests` | 完全支持             |
| `x-ratelimit-remaining-tokens`   | 完全支持             |
| `x-ratelimit-reset-requests`     | 完全支持             |
| `x-ratelimit-reset-tokens`       | 完全支持             |
| `retry-after`                    | 完全支持             |
| `request-id`                     | 完全支持             |
| `openai-version`                 | 始终为 `2020-10-01` |
| `authorization`                  | 完全支持             |
| `openai-processing-ms`           | 始终为空             |
