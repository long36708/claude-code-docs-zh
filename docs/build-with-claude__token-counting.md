---
title: 令牌计数
url: https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting
description: 在将消息发送给 Claude 之前统计其中的令牌数量。使用令牌计数来管理速率限制和成本、做出模型路由决策，并使提示符合目标长度。
---

## Compatibility
- [ZDR](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): eligible (excludes [Covered Models](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#model-specific-data-retention-requirements))
- Platforms: Claude API, Claude Platform on AWS, Amazon Bedrock, Google Cloud, Microsoft Foundry

"Token counting"（令牌计数）让您可以在将消息发送给 Claude 之前确定消息中的令牌数量。这有助于您就提示和用量做出明智的决策。借助令牌计数，您可以：

* 主动管理 "rate limits"（速率限制）和成本
* 做出明智的模型路由决策
* 将提示优化到特定长度

***

## 如何统计消息令牌

[令牌计数](https://platform.claude.com/docs/zh-CN/api/messages-count-tokens)端点接受与创建消息相同的结构化输入列表，包括对 "system prompt"（系统提示）、[工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)、[图像](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)和 [PDF](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support) 的支持。响应中包含输入令牌的总数。

<Note>
  令牌计数是一个**估计值**。在某些情况下，创建消息时实际使用的输入令牌数量可能会有少量差异。

  令牌计数可能包含 Anthropic 为系统优化而自动添加的令牌。**您无需为系统添加的令牌付费**。计费仅反映您的内容。
</Note>

### 支持的模型

所有[活跃模型](https://platform.claude.com/docs/zh-CN/about-claude/models/overview)都支持令牌计数，包括 Claude Opus 5 和 Claude Sonnet 5。

<Note>
  Claude 4.7 及更高版本的模型以及 Claude Mythos Preview 使用更新的 "tokenizer"（分词器）。相同的输入文本产生的令牌数量比早期模型大约多 30%。确切的增幅取决于内容和工作负载形态。请针对您计划使用的模型重新统计提示的令牌数，而不要复用针对早期模型测得的计数。
</Note>

### 统计基本消息中的令牌

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages/count_tokens \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "content-type: application/json" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "system": "You are a scientist",
      "messages": [{
        "role": "user",
        "content": "Hello, Claude"
      }]
    }'
  ```

  ```bash CLI
  ant messages count-tokens \
    --model claude-opus-5 \
    --system "You are a scientist" \
    --message '{role: user, content: "Hello, Claude"}'
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.count_tokens(
      model="claude-opus-5",
      system="You are a scientist",
      messages=[{"role": "user", "content": "Hello, Claude"}],
  )

  print(response.json())
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.countTokens({
    model: "claude-opus-5",
    system: "You are a scientist",
    messages: [
      {
        role: "user",
        content: "Hello, Claude"
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  using System;
  using System.Threading.Tasks;
  using Anthropic;
  using Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCountTokensParams
  {
      Model = Model.ClaudeOpus5,
      System = "You are a scientist",
      Messages = [new() { Role = Role.User, Content = "Hello, Claude" }]
  };

  var response = await client.Messages.CountTokens(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.CountTokens(context.TODO(), anthropic.MessageCountTokensParams{
  	Model: anthropic.ModelClaudeOpus5,
  	System: anthropic.MessageCountTokensParamsSystemUnion{
  		OfString: anthropic.String("You are a scientist"),
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Hello, Claude")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.messages.MessageCountTokensParams;
  import com.anthropic.models.messages.MessageTokensCount;
  // ...

  public class CountTokensExample {

    public static void main(String[] args) {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCountTokensParams params = MessageCountTokensParams.builder()
        .model(Model.CLAUDE_OPUS_5)
        .system("You are a scientist")
        .addUserMessage("Hello, Claude")
        .build();

      MessageTokensCount count = client.messages().countTokens(params);
      System.out.println(count);
    }
  }
  ```

  ```php PHP
  $client = new Client();

  $response = $client->messages->countTokens(
      messages: [
          ['role' => 'user', 'content' => 'Hello, Claude']
      ],
      model: 'claude-opus-5',
      system: 'You are a scientist',
  );

  echo json_encode($response);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.messages.count_tokens(
    model: "claude-opus-5",
    system: "You are a scientist",
    messages: [
      { role: "user", content: "Hello, Claude" }
    ]
  )

  puts response
  ```
</CodeGroup>

```json Output
{ "input_tokens": 14 }
```

### 统计包含工具的消息中的令牌

<Note>
  [服务器工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/server-tools)的令牌计数仅适用于第一次采样调用。
</Note>

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages/count_tokens \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "content-type: application/json" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "tools": [
        {
          "name": "get_weather",
          "description": "Get the current weather in a given location",
          "input_schema": {
            "type": "object",
            "properties": {
              "location": {
                "type": "string",
                "description": "The city and state, e.g. San Francisco, CA"
              }
            },
            "required": ["location"]
          }
        }
      ],
      "messages": [
        {
          "role": "user",
          "content": "What'\''s the weather like in San Francisco?"
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages count-tokens <<'YAML'
  model: claude-opus-5
  tools:
    - name: get_weather
      description: Get the current weather in a given location
      input_schema:
        type: object
        properties:
          location:
            type: string
            description: The city and state, e.g. San Francisco, CA
        required:
          - location
  messages:
    - role: user
      content: What's the weather like in San Francisco?
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.count_tokens(
      model="claude-opus-5",
      tools=[
          {
              "name": "get_weather",
              "description": "Get the current weather in a given location",
              "input_schema": {
                  "type": "object",
                  "properties": {
                      "location": {
                          "type": "string",
                          "description": "The city and state, e.g. San Francisco, CA",
                      }
                  },
                  "required": ["location"],
              },
          }
      ],
      messages=[{"role": "user", "content": "What's the weather like in San Francisco?"}],
  )

  print(response.json())
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.countTokens({
    model: "claude-opus-5",
    tools: [
      {
        name: "get_weather",
        description: "Get the current weather in a given location",
        input_schema: {
          type: "object",
          properties: {
            location: {
              type: "string",
              description: "The city and state, e.g. San Francisco, CA"
            }
          },
          required: ["location"]
        }
      }
    ],
    messages: [{ role: "user", content: "What's the weather like in San Francisco?" }]
  });

  console.log(response);
  ```

  ```csharp C#
  using System;
  using System.Collections.Generic;
  using System.Text.Json;
  using System.Threading.Tasks;
  using Anthropic;
  using Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCountTokensParams
  {
      Model = Model.ClaudeOpus5,
      Tools =
      [
          new MessageCountTokensTool(new Tool()
          {
              Name = "get_weather",
              Description = "Get the current weather in a given location",
              InputSchema = new InputSchema()
              {
                  Properties = new Dictionary<string, JsonElement>
                  {
                      ["location"] = JsonSerializer.SerializeToElement(new { type = "string", description = "The city and state, e.g. San Francisco, CA" }),
                  },
                  Required = ["location"],
              },
          }),
      ],
      Messages = [new() { Role = Role.User, Content = "What's the weather like in San Francisco?" }]
  };

  var count = await client.Messages.CountTokens(parameters);
  Console.WriteLine(count);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.CountTokens(context.TODO(), anthropic.MessageCountTokensParams{
  	Model: anthropic.ModelClaudeOpus5,
  	Tools: []anthropic.MessageCountTokensToolUnionParam{
  		{OfTool: &anthropic.ToolParam{
  			Name:        "get_weather",
  			Description: anthropic.String("Get the current weather in a given location"),
  			InputSchema: anthropic.ToolInputSchemaParam{
  				Properties: map[string]any{
  					"location": map[string]any{
  						"type":        "string",
  						"description": "The city and state, e.g. San Francisco, CA",
  					},
  				},
  				Required: []string{"location"},
  			},
  		}},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("What's the weather like in San Francisco?")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  jsonData, _ := json.MarshalIndent(response, "", "  ")
  fmt.Println(string(jsonData))
  ```

  ```java Java
  import com.anthropic.models.messages.MessageCountTokensParams;
  import com.anthropic.models.messages.MessageTokensCount;
  // ...
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      InputSchema schema = InputSchema.builder()
        .properties(
          JsonValue.from(
            Map.of(
              "location",
              Map.of(
                "type",
                "string",
                "description",
                "The city and state, e.g. San Francisco, CA"
              )
            )
          )
        )
        .putAdditionalProperty("required", JsonValue.from(List.of("location")))
        .build();

      MessageCountTokensParams params = MessageCountTokensParams.builder()
        .model(Model.CLAUDE_OPUS_5)
        .addTool(
          Tool.builder()
            .name("get_weather")
            .description("Get the current weather in a given location")
            .inputSchema(schema)
            .build()
        )
        .addUserMessage("What's the weather like in San Francisco?")
        .build();

      MessageTokensCount count = client.messages().countTokens(params);
      System.out.println(count);
  ```

  ```php PHP
  $client = new Client();

  $response = $client->messages->countTokens(
      messages: [
          ['role' => 'user', 'content' => "What's the weather like in San Francisco?"]
      ],
      model: 'claude-opus-5',
      tools: [
          [
              'name' => 'get_weather',
              'description' => 'Get the current weather in a given location',
              'input_schema' => [
                  'type' => 'object',
                  'properties' => [
                      'location' => [
                          'type' => 'string',
                          'description' => 'The city and state, e.g. San Francisco, CA'
                      ]
                  ],
                  'required' => ['location']
              ]
          ]
      ],
  );

  echo json_encode($response, JSON_PRETTY_PRINT);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.messages.count_tokens(
    model: "claude-opus-5",
    tools: [
      {
        name: "get_weather",
        description: "Get the current weather in a given location",
        input_schema: {
          type: "object",
          properties: {
            location: {
              type: "string",
              description: "The city and state, e.g. San Francisco, CA"
            }
          },
          required: ["location"]
        }
      }
    ],
    messages: [
      { role: "user", content: "What's the weather like in San Francisco?" }
    ]
  )

  puts response
  ```
</CodeGroup>

```json Output
{ "input_tokens": 403 }
```

### 统计包含图像的消息中的令牌

<CodeGroup>
  ```bash cURL
  #!/bin/sh

  IMAGE_URL="https://platform.claude.com/docs/images/vision-example.jpg"
  IMAGE_MEDIA_TYPE="image/jpeg"
  IMAGE_BASE64=$(curl -s "$IMAGE_URL" | base64 | tr -d '\n')

  curl https://api.anthropic.com/v1/messages/count_tokens \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d @- <<EOF
  {
    "model": "claude-opus-5",
    "messages": [
      {"role": "user", "content": [
        {"type": "image", "source": {
          "type": "base64",
          "media_type": "$IMAGE_MEDIA_TYPE",
          "data": "$IMAGE_BASE64"
        }},
        {"type": "text", "text": "Describe this image"}
      ]}
    ]
  }
  EOF
  ```

  ```bash CLI
  IMAGE_URL="https://platform.claude.com/docs/images/vision-example.jpg"
  curl -s "$IMAGE_URL" -o ./vision-example.jpg

  ant messages count-tokens <<'YAML'
  model: claude-opus-5
  messages:
    - role: user
      content:
        - type: image
          source:
            type: base64
            media_type: image/jpeg
            data: "@./vision-example.jpg"
        - type: text
          text: Describe this image
  YAML
  ```

  ```python Python
  import base64
  import httpx2

  image_url = "https://platform.claude.com/docs/images/vision-example.jpg"
  image_media_type = "image/jpeg"
  image_data = base64.standard_b64encode(httpx2.get(image_url).content).decode("utf-8")

  client = anthropic.Anthropic()

  response = client.messages.count_tokens(
      model="claude-opus-5",
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
                  {"type": "text", "text": "Describe this image"},
              ],
          }
      ],
  )
  print(response.json())
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic();

  const imageUrl = "https://platform.claude.com/docs/images/vision-example.jpg";
  const imageMediaType = "image/jpeg";
  const imageArrayBuffer = await (await fetch(imageUrl)).arrayBuffer();
  const imageData = Buffer.from(imageArrayBuffer).toString("base64");

  const response = await anthropic.messages.countTokens({
    model: "claude-opus-5",
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: {
              type: "base64",
              media_type: imageMediaType,
              data: imageData
            }
          },
          {
            type: "text",
            text: "Describe this image"
          }
        ]
      }
    ]
  });
  console.log(response);
  ```

  ```csharp C#
  using System;
  using System.Collections.Generic;
  using System.Net.Http;
  using System.Threading.Tasks;
  using Anthropic;
  using Anthropic.Models.Messages;

  AnthropicClient client = new();

  string imageUrl = "https://platform.claude.com/docs/images/vision-example.jpg";

  using HttpClient httpClient = new();
  byte[] imageBytes = await httpClient.GetByteArrayAsync(imageUrl);
  string imageData = Convert.ToBase64String(imageBytes);

  var parameters = new MessageCountTokensParams
  {
      Model = Model.ClaudeOpus5,
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = new MessageParamContent(new List<ContentBlockParam>
              {
                  new ContentBlockParam(new ImageBlockParam(
                      new ImageBlockParamSource(new Base64ImageSource()
                      {
                          Data = imageData,
                          MediaType = MediaType.ImageJpeg,
                      })
                  )),
                  new ContentBlockParam(new TextBlockParam("Describe this image")),
              }),
          }
      ]
  };

  var count = await client.Messages.CountTokens(parameters);
  Console.WriteLine(count);
  ```

  ```go Go
  imageURL := "https://platform.claude.com/docs/images/vision-example.jpg"

  req, err := http.NewRequest("GET", imageURL, nil)
  if err != nil {
  	log.Fatal(err)
  }
  req.Header.Set("User-Agent", "AnthropicDocsBot/1.0")

  resp, err := http.DefaultClient.Do(req)
  if err != nil {
  	log.Fatal(err)
  }
  defer resp.Body.Close()

  imageBytes, err := io.ReadAll(resp.Body)
  if err != nil {
  	log.Fatal(err)
  }
  imageData := base64.StdEncoding.EncodeToString(imageBytes)

  client := anthropic.NewClient()

  response, err := client.Messages.CountTokens(context.TODO(), anthropic.MessageCountTokensParams{
  	Model: anthropic.ModelClaudeOpus5,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.NewImageBlockBase64("image/jpeg", imageData),
  			anthropic.NewTextBlock("Describe this image"),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.messages.Base64ImageSource;
  // ...
  import com.anthropic.models.messages.MessageCountTokensParams;
  import com.anthropic.models.messages.MessageTokensCount;
  // ...
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      String imageUrl =
        "https://platform.claude.com/docs/images/vision-example.jpg";
      String imageMediaType = "image/jpeg";

      HttpClient httpClient = HttpClient.newHttpClient();
      HttpRequest request = HttpRequest.newBuilder().uri(URI.create(imageUrl)).build();
      byte[] imageBytes = httpClient
        .send(request, HttpResponse.BodyHandlers.ofByteArray())
        .body();
      String imageBase64 = Base64.getEncoder().encodeToString(imageBytes);

      ContentBlockParam imageBlock = ContentBlockParam.ofImage(
        ImageBlockParam.builder()
          .source(
            Base64ImageSource.builder()
              .mediaType(Base64ImageSource.MediaType.IMAGE_JPEG)
              .data(imageBase64)
              .build()
          )
          .build()
      );

      ContentBlockParam textBlock = ContentBlockParam.ofText(
        TextBlockParam.builder().text("Describe this image").build()
      );

      MessageCountTokensParams params = MessageCountTokensParams.builder()
        .model(Model.CLAUDE_OPUS_5)
        .addUserMessageOfBlockParams(List.of(imageBlock, textBlock))
        .build();

      MessageTokensCount count = client.messages().countTokens(params);
      System.out.println(count);
  ```

  ```php PHP
  $imageUrl = "https://platform.claude.com/docs/images/vision-example.jpg";
  $imageMediaType = "image/jpeg";
  $imageData = base64_encode(file_get_contents($imageUrl));

  $client = new Client();

  $response = $client->messages->countTokens(
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'image',
                      'source' => [
                          'type' => 'base64',
                          'media_type' => $imageMediaType,
                          'data' => $imageData
                      ]
                  ],
                  ['type' => 'text', 'text' => 'Describe this image']
              ]
          ]
      ],
      model: 'claude-opus-5',
  );
  print_r($response);
  ```

  ```ruby Ruby
  require "base64"
  require "net/http"

  image_url = "https://platform.claude.com/docs/images/vision-example.jpg"
  image_media_type = "image/jpeg"

  uri = URI(image_url)
  image_data = Base64.strict_encode64(Net::HTTP.get(uri))

  client = Anthropic::Client.new

  response = client.messages.count_tokens(
    model: "claude-opus-5",
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: {
              type: "base64",
              media_type: image_media_type,
              data: image_data
            }
          },
          { type: "text", text: "Describe this image" }
        ]
      }
    ]
  )
  puts response
  ```
</CodeGroup>

```json Output
{ "input_tokens": 1028 }
```

设置了 [`"oversized_image": "error"`](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#oversized-image-error) 的嵌入图像块会在计数时被拒绝，与 Messages API 拒绝它的方式完全相同。

### 统计包含思考的消息中的令牌

<Note>
  有关更多详细信息，请参阅[思考与上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-and-the-context-window)。

  * 来自**先前**助手轮次的思考块会被忽略，**不会**计入您的输入令牌
  * **当前**助手轮次的思考**会**计入您的输入令牌
</Note>

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages/count_tokens \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "content-type: application/json" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-sonnet-4-6",
      "thinking": {
        "type": "enabled",
        "budget_tokens": 16000
      },
      "messages": [
        {
          "role": "user",
          "content": "Are there an infinite number of prime numbers such that n mod 4 == 3?"
        },
        {
          "role": "assistant",
          "content": [
            {
              "type": "thinking",
              "thinking": "This is a nice number theory question. Lets think about it step by step...",
              "signature": "EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV..."
            },
            {
              "type": "text",
              "text": "Yes, there are infinitely many prime numbers p such that p mod 4 = 3..."
            }
          ]
        },
        {
          "role": "user",
          "content": "Can you write a formal proof?"
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages count-tokens <<'YAML'
  model: claude-sonnet-4-6
  thinking:
    type: enabled
    budget_tokens: 16000
  messages:
    - role: user
      content: Are there an infinite number of prime numbers such that n mod 4 == 3?
    - role: assistant
      content:
        - type: thinking
          thinking: >-
            This is a nice number theory question. Lets think about it step by step...
          signature: >-
            EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV...
        - type: text
          text: Yes, there are infinitely many prime numbers p such that p mod 4 = 3...
    - role: user
      content: Can you write a formal proof?
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.count_tokens(
      model="claude-sonnet-4-6",
      thinking={"type": "enabled", "budget_tokens": 16000},
      messages=[
          {
              "role": "user",
              "content": "Are there an infinite number of prime numbers such that n mod 4 == 3?",
          },
          {
              "role": "assistant",
              "content": [
                  {
                      "type": "thinking",
                      "thinking": "This is a nice number theory question. Let's think about it step by step...",
                      "signature": "EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV...",
                  },
                  {
                      "type": "text",
                      "text": "Yes, there are infinitely many prime numbers p such that p mod 4 = 3...",
                  },
              ],
          },
          {"role": "user", "content": "Can you write a formal proof?"},
      ],
  )

  print(response.json())
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.countTokens({
    model: "claude-sonnet-4-6",
    thinking: {
      type: "enabled",
      budget_tokens: 16000
    },
    messages: [
      {
        role: "user",
        content: "Are there an infinite number of prime numbers such that n mod 4 == 3?"
      },
      {
        role: "assistant",
        content: [
          {
            type: "thinking",
            thinking:
              "This is a nice number theory question. Let's think about it step by step...",
            signature:
              "EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV..."
          },
          {
            type: "text",
            text: "Yes, there are infinitely many prime numbers p such that p mod 4 = 3..."
          }
        ]
      },
      {
        role: "user",
        content: "Can you write a formal proof?"
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  using System;
  using System.Threading.Tasks;
  using System.Collections.Generic;
  using Anthropic;
  using Anthropic.Models.Messages;

  AnthropicClient client = new();

  var parameters = new MessageCountTokensParams
  {
      Model = Model.ClaudeSonnet4_6,
      Thinking = new ThinkingConfigEnabled(budgetTokens: 16000),
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = "Are there an infinite number of prime numbers such that n mod 4 == 3?"
          },
          new()
          {
              Role = Role.Assistant,
              Content = new MessageParamContent(new List<ContentBlockParam>
              {
                  new ContentBlockParam(new ThinkingBlockParam()
                  {
                      Thinking = "This is a nice number theory question. Let's think about it step by step...",
                      Signature = "EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV...",
                  }),
                  new ContentBlockParam(new TextBlockParam("Yes, there are infinitely many prime numbers p such that p mod 4 = 3...")),
              }),
          },
          new()
          {
              Role = Role.User,
              Content = "Can you write a formal proof?"
          }
      ]
  };

  var response = await client.Messages.CountTokens(parameters);
  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  thinkingBlock := anthropic.NewThinkingBlock(
  	"EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV...",
  	"This is a nice number theory question. Let's think about it step by step...",
  )

  textBlock := anthropic.NewTextBlock(
  	"Yes, there are infinitely many prime numbers p such that p mod 4 = 3...",
  )

  response, err := client.Messages.CountTokens(context.TODO(), anthropic.MessageCountTokensParams{
  	Model:    anthropic.ModelClaudeSonnet4_6,
  	Thinking: anthropic.ThinkingConfigParamOfEnabled(16000),
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Are there an infinite number of prime numbers such that n mod 4 == 3?")),
  		anthropic.NewAssistantMessage(thinkingBlock, textBlock),
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Can you write a formal proof?")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("%+v\n", response)
  ```

  ```java Java
  import com.anthropic.models.messages.MessageCountTokensParams;
  import com.anthropic.models.messages.MessageTokensCount;
  // ...
  import com.anthropic.models.messages.ThinkingBlockParam;
  // ...
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      List<ContentBlockParam> assistantBlocks = List.of(
        ContentBlockParam.ofThinking(
          ThinkingBlockParam.builder()
            .thinking(
              "This is a nice number theory question. Let's think about it step by step..."
            )
            .signature(
              "EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV..."
            )
            .build()
        ),
        ContentBlockParam.ofText(
          TextBlockParam.builder()
            .text("Yes, there are infinitely many prime numbers p such that p mod 4 = 3...")
            .build()
        )
      );

      MessageCountTokensParams params = MessageCountTokensParams.builder()
        .model(Model.CLAUDE_SONNET_4_6)
        .enabledThinking(16000)
        .addUserMessage("Are there an infinite number of prime numbers such that n mod 4 == 3?")
        .addAssistantMessageOfBlockParams(assistantBlocks)
        .addUserMessage("Can you write a formal proof?")
        .build();

      MessageTokensCount count = client.messages().countTokens(params);
      System.out.println(count);
  ```

  ```php PHP
  $client = new Client();

  $response = $client->messages->countTokens(
      messages: [
          [
              'role' => 'user',
              'content' => 'Are there an infinite number of prime numbers such that n mod 4 == 3?'
          ],
          [
              'role' => 'assistant',
              'content' => [
                  [
                      'type' => 'thinking',
                      'thinking' => 'This is a nice number theory question. Let\'s think about it step by step...',
                      'signature' => 'EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV...'
                  ],
                  [
                      'type' => 'text',
                      'text' => 'Yes, there are infinitely many prime numbers p such that p mod 4 = 3...'
                  ]
              ]
          ],
          [
              'role' => 'user',
              'content' => 'Can you write a formal proof?'
          ]
      ],
      model: 'claude-sonnet-4-6',
      thinking: [
          'type' => 'enabled',
          'budget_tokens' => 16000
      ],
  );

  echo json_encode($response);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.messages.count_tokens(
    model: "claude-sonnet-4-6",
    thinking: {
      type: "enabled",
      budget_tokens: 16000
    },
    messages: [
      {
        role: "user",
        content: "Are there an infinite number of prime numbers such that n mod 4 == 3?"
      },
      {
        role: "assistant",
        content: [
          {
            type: "thinking",
            thinking: "This is a nice number theory question. Let's think about it step by step...",
            signature: "EuYBCkQYAiJAgCs1le6/Pol5Z4/JMomVOouGrWdhYNsH3ukzUECbB6iWrSQtsQuRHJID6lWV..."
          },
          {
            type: "text",
            text: "Yes, there are infinitely many prime numbers p such that p mod 4 = 3..."
          }
        ]
      },
      {
        role: "user",
        content: "Can you write a formal proof?"
      }
    ]
  )

  puts response
  ```
</CodeGroup>

```json Output
{ "input_tokens": 88 }
```

### 统计包含 PDF 的消息中的令牌

<Note>
  令牌计数支持 PDF，其 [PDF 支持限制](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support#pdf-support-limitations)与 Messages API 相同。
</Note>

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages/count_tokens \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "content-type: application/json" \
    -H "anthropic-version: 2023-06-01" \
    -d @- <<EOF
  {
    "model": "claude-opus-5",
    "messages": [{
      "role": "user",
      "content": [
        {
          "type": "document",
          "source": {
            "type": "base64",
            "media_type": "application/pdf",
            "data": "$PDF_BASE64"
          }
        },
        {
          "type": "text",
          "text": "Please summarize this document."
        }
      ]
    }]
  }
  EOF
  ```

  ```bash CLI
  ant messages count-tokens <<'YAML'
  model: claude-opus-5
  messages:
    - role: user
      content:
        - type: document
          source:
            type: base64
            media_type: application/pdf
            data: "@./document.pdf"
        - type: text
          text: Please summarize this document.
  YAML
  ```

  ```python Python
  import base64
  import anthropic

  client = anthropic.Anthropic()

  with open("/path/to/document.pdf", "rb") as pdf_file:
      pdf_base64 = base64.standard_b64encode(pdf_file.read()).decode("utf-8")

  response = client.messages.count_tokens(
      model="claude-opus-5",
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "document",
                      "source": {
                          "type": "base64",
                          "media_type": "application/pdf",
                          "data": pdf_base64,
                      },
                  },
                  {"type": "text", "text": "Please summarize this document."},
              ],
          }
      ],
  )

  print(response.json())
  ```

  ```typescript TypeScript
  import { readFile } from "node:fs/promises";

  const client = new Anthropic();

  const pdfBase64 = await readFile("/path/to/document.pdf", { encoding: "base64" });

  const response = await client.messages.countTokens({
    model: "claude-opus-5",
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "base64",
              media_type: "application/pdf",
              data: pdfBase64
            }
          },
          {
            type: "text",
            text: "Please summarize this document."
          }
        ]
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  using System;
  using System.IO;
  using System.Threading.Tasks;
  using System.Collections.Generic;
  using Anthropic;
  using Anthropic.Models.Messages;

  AnthropicClient client = new();

  byte[] pdfBytes = await File.ReadAllBytesAsync("/path/to/document.pdf");
  string pdfBase64 = Convert.ToBase64String(pdfBytes);

  var parameters = new MessageCountTokensParams
  {
      Model = Model.ClaudeOpus5,
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = new MessageParamContent(new List<ContentBlockParam>
              {
                  new ContentBlockParam(new DocumentBlockParam(
                      new DocumentBlockParamSource(new Base64PdfSource()
                      {
                          Data = pdfBase64,
                      })
                  )),
                  new ContentBlockParam(new TextBlockParam("Please summarize this document.")),
              }),
          }
      ]
  };

  var count = await client.Messages.CountTokens(parameters);
  Console.WriteLine(count);
  ```

  ```go Go
  client := anthropic.NewClient()

  pdfBytes, err := os.ReadFile("/path/to/document.pdf")
  if err != nil {
  	log.Fatal(err)
  }
  pdfBase64 := base64.StdEncoding.EncodeToString(pdfBytes)

  response, err := client.Messages.CountTokens(context.TODO(), anthropic.MessageCountTokensParams{
  	Model: anthropic.ModelClaudeOpus5,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.NewDocumentBlock(anthropic.Base64PDFSourceParam{
  				Data: pdfBase64,
  			}),
  			anthropic.NewTextBlock("Please summarize this document."),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.messages.Base64PdfSource;
  // ...
  import com.anthropic.models.messages.DocumentBlockParam;
  import com.anthropic.models.messages.MessageCountTokensParams;
  import com.anthropic.models.messages.MessageTokensCount;
  // ...
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      byte[] fileBytes = Files.readAllBytes(Path.of("/path/to/document.pdf"));
      String pdfBase64 = Base64.getEncoder().encodeToString(fileBytes);

      ContentBlockParam documentBlock = ContentBlockParam.ofDocument(
        DocumentBlockParam.builder()
          .source(Base64PdfSource.builder().data(pdfBase64).build())
          .build()
      );

      ContentBlockParam textBlock = ContentBlockParam.ofText(
        TextBlockParam.builder().text("Please summarize this document.").build()
      );

      MessageCountTokensParams params = MessageCountTokensParams.builder()
        .model(Model.CLAUDE_OPUS_5)
        .addUserMessageOfBlockParams(List.of(documentBlock, textBlock))
        .build();

      MessageTokensCount count = client.messages().countTokens(params);
      System.out.println(count);
  ```

  ```php PHP
  $client = new Client();

  $pdfBase64 = base64_encode(file_get_contents("/path/to/document.pdf"));

  $response = $client->messages->countTokens(
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'document',
                      'source' => [
                          'type' => 'base64',
                          'media_type' => 'application/pdf',
                          'data' => $pdfBase64
                      ]
                  ],
                  [
                      'type' => 'text',
                      'text' => 'Please summarize this document.'
                  ]
              ]
          ]
      ],
      model: 'claude-opus-5',
  );

  echo json_encode($response);
  ```

  ```ruby Ruby
  require "base64"

  client = Anthropic::Client.new

  pdf_base64 = Base64.strict_encode64(File.binread("/path/to/document.pdf"))

  response = client.messages.count_tokens(
    model: "claude-opus-5",
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "base64",
              media_type: "application/pdf",
              data: pdf_base64
            }
          },
          {
            type: "text",
            text: "Please summarize this document."
          }
        ]
      }
    ]
  )

  puts response
  ```
</CodeGroup>

```json Output
{ "input_tokens": 2188 }
```

***

## Claude Fable 5 和 Claude Mythos 5 上的令牌计数

Claude Fable 5 和 Claude Mythos 5 使用随 Claude Opus 4.7 引入的分词器，对于相同的文本，它产生的令牌数量比 Claude Opus 4.7 之前的模型大约多 30%。确切的增幅取决于内容和工作负载形态。令牌计数端点会根据您传入的 `model` 所对应的分词器返回计数，因此若要衡量您的工作负载的差异，请对同一请求计数两次：一次使用您当前的模型，一次使用 `model: "claude-fable-5"`（或 `"claude-mythos-5"`），然后比较两个 `input_tokens` 值。

<Note>
  **计费与迁移：** Claude Fable 5 和 Claude Mythos 5 上的用量和计费反映的是该分词器的计数。如果您从 Claude Opus 4.7 之前的模型迁移，相同的内容会消耗大约多 30% 的令牌。确切的增幅取决于内容和工作负载形态。将工作负载迁移到 Claude Fable 5 和 Claude Mythos 5 时，请勿复用在 Claude Opus 4.7 之前的模型上测得的令牌计数来估算成本或 "context window"（上下文窗口）的适配情况。请使用 `model: "claude-fable-5"`（或 `"claude-mythos-5"`）统计您的提示。
</Note>

***

## 定价和速率限制

令牌计数**可免费使用**，但会受到基于您的[使用层级](https://platform.claude.com/docs/zh-CN/api/rate-limits#rate-limits)的每分钟请求数速率限制。如果您需要更高的限制，请在[速率限制](https://platform.claude.com/settings/limits)页面上使用**申请提高速率限制**。

| 使用层级  | 每分钟请求数（RPM） |
| ----- | ----------- |
| Start | 2,000       |
| Build | 4,000       |
| Scale | 8,000       |

<Note>
  令牌计数和消息创建具有各自独立的速率限制。其中一项的使用不会计入另一项的限制。
</Note>

***

## 常见问题

<AccordionGroup>
  <Accordion title="令牌计数是否使用提示缓存？">
    不会，令牌计数提供的是不使用缓存逻辑的估计值。虽然您可以在令牌计数请求中提供 `cache_control` 块，但 "prompt caching"（提示缓存）仅在实际创建消息时发生。
  </Accordion>
</AccordionGroup>

***

## 后续步骤

<CardGroup cols={2}>
  <Card title="统计消息令牌" icon="code" href="https://platform.claude.com/docs/zh-CN/api/messages-count-tokens">
    阅读令牌计数端点的完整 API 参考。
  </Card>

  <Card title="上下文窗口" icon="arrows-maximize" href="https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows">
    使用令牌计数使提示保持在模型的上下文窗口之内。
  </Card>

  <Card title="速率限制" icon="gauge" href="https://platform.claude.com/docs/zh-CN/api/rate-limits">
    在发送请求之前检查令牌计数，以保持在您的使用层级之内。
  </Card>

  <Card title="提示缓存" icon="database" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching">
    通过缓存提示前缀来降低重复提示的成本和延迟。
  </Card>
</CardGroup>
