---
title: Google Cloud 上的 Claude
url: https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai
description: Anthropic 的 Claude 模型可通过 [Google Cloud's Agent Platform](https://cloud.google.com/vertex-ai) 使用。
---

用于在 Google Cloud's Agent Platform 上访问 Claude 的 API 与 [Messages API](https://platform.claude.com/docs/zh-CN/api/messages/create) 几乎完全相同，但在请求格式上有两个关键区别：

* 在 Agent Platform 上，`model` 不在请求体中传递，而是在 Google Cloud 端点 URL 中指定。
* 在 Agent Platform 上，`anthropic_version` 在请求体中传递（而不是作为请求头），并且必须设置为值 `vertex-2023-10-16`。

Anthropic 的官方 [client SDKs](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview)（客户端 SDK）也支持 Agent Platform。本指南将引导您使用 Anthropic 的客户端 SDK 之一向 Agent Platform 上的 Claude 发出请求。

请注意，本指南假设您已经拥有一个能够使用 Agent Platform 的 Google Cloud 项目。有关所需设置的更多信息和完整演练，请参阅 [Agent Platform 上的 Anthropic Claude 模型](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude)。

## 安装用于访问 Agent Platform 的 SDK

首先，为您选择的语言安装 Anthropic 的 [客户端 SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview)。

<Tabs>
  <Tab title="Python">
    ```bash
    pip install -U "anthropic[vertex]"
    ```
  </Tab>

  <Tab title="TypeScript">
    ```bash
    npm install @anthropic-ai/vertex-sdk
    ```
  </Tab>

  <Tab title="C#">
    ```bash
    dotnet add package Anthropic.Vertex
    ```
  </Tab>

  <Tab title="Go">
    ```bash
    go get github.com/anthropics/anthropic-sdk-go
    ```
  </Tab>

  <Tab title="Java">
    <CodeGroup exclude="shell, python, typescript, csharp, go, php, ruby">
      ```groovy Gradle
      implementation("com.anthropic:anthropic-java:2.58.0")
      implementation("com.anthropic:anthropic-java-vertex:2.58.0")
      ```

      ```xml Maven
      <dependency>
          <groupId>com.anthropic</groupId>
          <artifactId>anthropic-java</artifactId>
          <version>2.58.0</version>
      </dependency>
      <dependency>
          <groupId>com.anthropic</groupId>
          <artifactId>anthropic-java-vertex</artifactId>
          <version>2.58.0</version>
      </dependency>
      ```

      ```java Java
      import com.anthropic.client.AnthropicClient;
      import com.anthropic.client.okhttp.AnthropicOkHttpClient;
      import com.anthropic.models.messages.Message;
      import com.anthropic.models.messages.MessageCreateParams;
      import com.anthropic.models.messages.Model;
      import com.anthropic.vertex.backends.VertexBackend;

      void main() {
          AnthropicClient client = AnthropicOkHttpClient.builder()
              .backend(VertexBackend.fromEnv())
              .build();

          MessageCreateParams params = MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5)
              .maxTokens(1024L)
              .addUserMessage("What is the capital of France?")
              .build();

          Message response = client.messages().create(params);
          response.content().stream()
              .flatMap(block -> block.text().stream())
              .forEach(textBlock -> IO.println(textBlock.text()));
      }
      ```
    </CodeGroup>
  </Tab>

  <Tab title="PHP">
    ```bash
    composer require anthropic-ai/sdk google/auth
    ```
  </Tab>

  <Tab title="Ruby">
    ```bash
    # Gemfile
    gem "anthropic"
    gem "googleauth"
    ```
  </Tab>
</Tabs>

## 访问 Agent Platform

### 模型可用性

请注意，Anthropic 模型的可用性因区域而异。请在 [Model Garden](https://cloud.google.com/model-garden) 中搜索"Claude"，或前往 [Anthropic Claude 模型](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude) 获取最新信息。

#### API 模型 ID

生命周期术语（Deprecated（已弃用）、Retired（已停用））在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中定义。合作伙伴运营平台上的生命周期日期由合作伙伴设定，可能与 Claude API 的时间表不同。有关 Agent Platform 上任何模型的当前停用日期，请参阅 [Google Cloud 关于 Agent Platform 上 Claude 模型的文档](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude)。

| 模型                     | Agent Platform API 模型 ID    |
| ---------------------- | --------------------------- |
| Claude Fable 5.1       | claude-fable-5-1            |
| Claude Fable 5         | claude-fable-5              |
| Claude Opus 5          | claude-opus-5               |
| Claude Opus 4.8        | claude-opus-4-8             |
| Claude Opus 4.7        | claude-opus-4-7             |
| Claude Opus 4.6        | claude-opus-4-6             |
| Claude Sonnet 5        | `claude-sonnet-5`           |
| Claude Sonnet 4.6      | claude-sonnet-4-6           |
| Claude Sonnet 4.5      | claude-sonnet-4-5\@20250929 |
| Claude Sonnet 4 已弃用。   | claude-sonnet-4\@20250514   |
| Claude Sonnet 3.7 已停用。 | claude-3-7-sonnet\@20250219 |
| Claude Opus 4.5        | claude-opus-4-5\@20251101   |
| Claude Opus 4.1 已弃用。   | claude-opus-4-1\@20250805   |
| Claude Opus 4 已弃用。     | claude-opus-4\@20250514     |
| Claude Haiku 4.5       | claude-haiku-4-5\@20251001  |
| Claude Haiku 3.5 已弃用。  | claude-3-5-haiku\@20241022  |

<Tip>
  正在升级到更新的 Claude 模型？在 Claude Code 中，运行 `/claude-api migrate` 即可在您的整个代码库中应用模型 ID 替换和破坏性参数变更。该技能会检测您的代码所面向的云平台，并针对该平台调整模型 ID 格式和功能变更。请参阅[迁移到更新的 Claude 模型](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model)。
</Tip>

### 发出请求

在运行请求之前，您可能需要运行 `gcloud auth application-default login` 以向 Google Cloud 进行身份验证。

以下示例展示了如何在 Agent Platform 上通过 Claude 生成文本：

<CodeGroup>
  ```bash cURL
  MODEL_ID=claude-opus-5
  PROJECT_ID=MY_PROJECT_ID

  curl https://aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/publishers/anthropic/models/${MODEL_ID}:rawPredict \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    -H "Content-Type: application/json" \
    -d '{
      "anthropic_version": "vertex-2023-10-16",
      "messages": [{"role": "user", "content": "Hey Claude!"}],
      "max_tokens": 100
    }'
  ```

  ```bash CLI
  # ant CLI 不支持 Agent Platform。
  ```

  ```python Python
  from anthropic import AnthropicVertex

  project_id = "MY_PROJECT_ID"
  region = "global"

  client = AnthropicVertex(project_id=project_id, region=region)

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=100,
      messages=[
          {
              "role": "user",
              "content": "Hey Claude!",
          }
      ],
  )
  print(message)
  ```

  ```typescript TypeScript
  import { AnthropicVertex } from "@anthropic-ai/vertex-sdk";

  const projectId = "MY_PROJECT_ID";
  const region = "global";

  // 走标准的 `google-auth-library` 流程。
  const client = new AnthropicVertex({
    projectId,
    region
  });

  const result = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 100,
    messages: [
      {
        role: "user",
        content: "Hey Claude!"
      }
    ]
  });
  console.log(JSON.stringify(result, null, 2));
  ```

  ```csharp C#
  using Anthropic.Models.Messages;
  using Anthropic.Vertex;

  var projectId = "MY_PROJECT_ID";
  var region = "global";

  var client = new AnthropicVertexClient(new AnthropicVertexCredentials(region, projectId));

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 100,
      Messages = [new() { Role = Role.User, Content = "Hey Claude!" }]
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  import (
  	"context"
  	"fmt"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/vertex"
  )
  // ...
  	// 使用默认的 Google Cloud 凭据
  	client := anthropic.NewClient(
  		vertex.WithGoogleAuth(context.Background(), "global", "MY_PROJECT_ID"),
  	)

  	message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeOpus5,
  		MaxTokens: 100,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hey Claude!")),
  		},
  	})
  	if err != nil {
  		panic(err)
  	}
  	fmt.Printf("%+v\n", message)
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.models.messages.Message;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;
  import com.anthropic.vertex.backends.VertexBackend;

  void main() {
      // 使用默认的 Google Cloud 凭据
      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(VertexBackend.fromEnv())
          .build();

      Message message = client
          .messages()
          .create(
              MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5)
                  .maxTokens(100)
                  .addUserMessage("Hey Claude!")
                  .build()
          );

      IO.println(message);
  }
  ```

  ```php PHP
  <?php

  use Anthropic\Vertex;

  $client = Vertex\Client::fromEnvironment(
      location: 'global',
      projectId: 'MY_PROJECT_ID',
  );

  $message = $client->messages->create(
      maxTokens: 100,
      messages: [
          ['role' => 'user', 'content' => 'Hey Claude!']
      ],
      model: 'claude-opus-5',
  );
  $textBlock = array_find($message->content, static fn ($block): bool => $block->type === 'text');
  echo $textBlock->text;
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::VertexClient.new(
    region: "global",
    project_id: "MY_PROJECT_ID"
  )

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 100,
    messages: [{role: "user", content: "Hey Claude!"}]
  )

  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

有关更多详细信息，请参阅 [客户端 SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview) 和官方 [Agent Platform 文档](https://cloud.google.com/vertex-ai/docs)。

Claude 也可通过 [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 和 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 使用。

## 数据保留

此产品的数据处理由 Google Cloud 管理。有关详细信息，请参阅 [Agent Platform 与零数据保留](https://cloud.google.com/vertex-ai/generative-ai/docs/data-governance)。

## 活动日志记录

Agent Platform 提供 [请求-响应日志记录服务](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/request-response-logging)，允许您记录与您的使用相关的提示和补全内容。

Anthropic 建议您至少以 30 天滚动周期记录您的活动，以便了解您的活动情况并调查任何潜在的滥用行为。

<Note>
  开启此服务不会授予 Google 或 Anthropic 对您内容的任何访问权限。
</Note>

## 功能支持

有关包含 Google Cloud 可用性的完整功能列表，请参阅[功能概览](https://platform.claude.com/docs/zh-CN/build-with-claude/overview)。

### 支持的功能亮点

* [Messages API](https://platform.claude.com/docs/zh-CN/api/messages/create)
* [提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)
* [思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)
* [工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)，包括 [Bash 工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/bash-tool)、[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)、[计算机使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)、[记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool) 和 [文本编辑器工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/text-editor-tool)
* [网页搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)
* [引用](https://platform.claude.com/docs/zh-CN/build-with-claude/citations)
* [结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)

### 不支持的功能

* 输入源（图像和文档的 URL 源、Files API）
* 服务器端工具（代码执行、网页抓取、advisor）
* 智能体基础设施（Agent Skills、MCP 连接器、程序化工具调用）
* API 端点（Message Batches、Models、Admin、Compliance、Usage and Cost）
* Claude Managed Agents
* 服务器端回退（[`fallbacks` 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)；请改用[客户端回退模式](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#client-side-fallback)）

### 上下文窗口

Claude Fable 5.1、Claude Fable 5、Claude Opus 5、Claude Opus 4.8、Claude Opus 4.7、Claude Opus 4.6、Claude Sonnet 5 和 Claude Sonnet 4.6 在 Agent Platform 上拥有 [100 万令牌的 context window（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)。其他 Claude 模型，包括 Sonnet 4.5 和 Sonnet 4（已弃用），拥有 20 万令牌的上下文窗口。

Agent Platform 将请求负载限制为 30 MB。在发送大型文档或大量图像时，您可能会在达到令牌限制之前先达到此限制。

## 全球、多区域和区域端点

Agent Platform 提供三种端点类型：

* **全球端点：** 动态路由以实现最大可用性
* **多区域端点：** 在某一地理区域内（例如美国或欧盟）进行动态路由，在满足数据驻留要求的同时提供高可用性
* **区域端点：** 保证数据通过特定地理区域路由

区域端点和多区域端点的定价比全球端点高出 10%。

<Note>
  这仅适用于 Claude Sonnet 4.5 及未来的模型。较旧的模型（Claude Sonnet 4（已弃用）、Opus 4（已弃用）及更早的模型）保持其现有的定价结构。
</Note>

### 何时使用各选项

**全球端点（推荐）：**

* 提供最大的可用性和正常运行时间
* 将请求动态路由到具有可用容量的区域
* 无定价溢价
* 最适合数据驻留要求灵活的应用
* 仅支持按需付费流量（预置吞吐量需要使用区域端点）

**多区域端点：**

* 在某一地理区域内（目前为 `us` 和 `eu`）跨区域动态路由请求
* 适用于需要在较大地理范围内满足数据驻留要求、但希望获得比单一区域更高可用性的场景
* 定价比全球端点高出 10%
* 仅支持按需付费流量（预置吞吐量需要使用区域端点）

**区域端点：**

* 通过特定地理区域路由流量
* 单区域数据驻留、严格合规要求或预置吞吐量所必需
* 同时支持按需付费和预置吞吐量
* 10% 的定价溢价反映了专用区域容量的基础设施成本

### 实现

**使用全球端点（推荐）：**

初始化客户端时，将 `region` 参数设置为 `"global"`：

<CodeGroup>
  ```bash cURL
  MODEL_ID=claude-opus-5
  PROJECT_ID=MY_PROJECT_ID

  curl https://aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/publishers/anthropic/models/${MODEL_ID}:rawPredict \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    -H "Content-Type: application/json" \
    -d '{
      "anthropic_version": "vertex-2023-10-16",
      "messages": [{"role": "user", "content": "Hey Claude!"}],
      "max_tokens": 100
    }'
  ```

  ```bash CLI
  # ant CLI 不支持 Agent Platform。
  ```

  ```python Python
  from anthropic import AnthropicVertex

  project_id = "MY_PROJECT_ID"
  region = "global"

  client = AnthropicVertex(project_id=project_id, region=region)

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=100,
      messages=[
          {
              "role": "user",
              "content": "Hey Claude!",
          }
      ],
  )
  print(message)
  ```

  ```typescript TypeScript
  import { AnthropicVertex } from "@anthropic-ai/vertex-sdk";

  const projectId = "MY_PROJECT_ID";
  const region = "global";

  const client = new AnthropicVertex({
    projectId,
    region
  });

  const result = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 100,
    messages: [
      {
        role: "user",
        content: "Hey Claude!"
      }
    ]
  });
  console.log(JSON.stringify(result, null, 2));
  ```

  ```csharp C#
  using Anthropic.Models.Messages;
  using Anthropic.Vertex;

  var projectId = "MY_PROJECT_ID";
  var region = "global";

  var client = new AnthropicVertexClient(new AnthropicVertexCredentials(region, projectId));

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 100,
      Messages = [new() { Role = Role.User, Content = "Hey Claude!" }]
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  import (
  	"context"
  	"fmt"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/vertex"
  )
  // ...
  	// 使用默认的 Google Cloud 凭据
  	client := anthropic.NewClient(
  		vertex.WithGoogleAuth(context.Background(), "global", "MY_PROJECT_ID"),
  	)

  	message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeOpus5,
  		MaxTokens: 100,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hey Claude!")),
  		},
  	})
  	if err != nil {
  		panic(err)
  	}
  	fmt.Printf("%+v\n", message)
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;
  import com.anthropic.vertex.backends.VertexBackend;
  import com.google.auth.oauth2.GoogleCredentials;

  void main() throws Exception {
      // 使用默认的 Google Cloud 凭据
      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(
              VertexBackend.builder()
                  .googleCredentials(GoogleCredentials.getApplicationDefault())
                  .region("global")
                  .project("MY_PROJECT_ID")
                  .build()
          )
          .build();

      var message = client
          .messages()
          .create(
              MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5)
                  .maxTokens(100)
                  .addUserMessage("Hey Claude!")
                  .build()
          );

      IO.println(message);
  }
  ```

  ```php PHP
  <?php

  use Anthropic\Vertex;

  $client = Vertex\Client::fromEnvironment(
      location: 'global',
      projectId: 'MY_PROJECT_ID',
  );

  $message = $client->messages->create(
      maxTokens: 100,
      messages: [
          ['role' => 'user', 'content' => 'Hey Claude!']
      ],
      model: 'claude-opus-5',
  );

  $textBlock = array_find($message->content, static fn ($block): bool => $block->type === 'text');
  echo $textBlock->text;
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::VertexClient.new(
    region: "global",
    project_id: "MY_PROJECT_ID"
  )

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 100,
    messages: [{role: "user", content: "Hey Claude!"}]
  )

  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

**使用多区域端点：**

将 `region` 参数设置为多区域标识符：`"us"` 表示美国，`"eu"` 表示欧盟。SDK 会将请求路由到相应的多区域端点（`https://aiplatform.us.rep.googleapis.com` 或 `https://aiplatform.eu.rep.googleapis.com`），该端点会在该地理范围内的各区域之间动态平衡流量。

<CodeGroup>
  ```bash cURL
  MODEL_ID=claude-opus-5
  LOCATION=us # Multi-region identifier: "us" or "eu"
  PROJECT_ID=MY_PROJECT_ID

  curl https://aiplatform.${LOCATION}.rep.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/publishers/anthropic/models/${MODEL_ID}:rawPredict \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    -H "Content-Type: application/json" \
    -d '{
      "anthropic_version": "vertex-2023-10-16",
      "messages": [{"role": "user", "content": "Hey Claude!"}],
      "max_tokens": 100
    }'
  ```

  ```bash CLI
  # ant CLI 不支持 Agent Platform。
  ```

  ```python Python
  from anthropic import AnthropicVertex

  project_id = "MY_PROJECT_ID"
  region = "us"  # Multi-region identifier: "us" or "eu"

  client = AnthropicVertex(project_id=project_id, region=region)

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=100,
      messages=[
          {
              "role": "user",
              "content": "Hey Claude!",
          }
      ],
  )
  print(message)
  ```

  ```typescript TypeScript
  import { AnthropicVertex } from "@anthropic-ai/vertex-sdk";

  const projectId = "MY_PROJECT_ID";
  const region = "us"; // Multi-region identifier: "us" or "eu"

  const client = new AnthropicVertex({
    projectId,
    region
  });

  const result = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 100,
    messages: [
      {
        role: "user",
        content: "Hey Claude!"
      }
    ]
  });
  console.log(JSON.stringify(result, null, 2));
  ```

  ```csharp C#
  using Anthropic.Models.Messages;
  using Anthropic.Vertex;

  var projectId = "MY_PROJECT_ID";
  var region = "us"; // Multi-region identifier: "us" or "eu"

  var client = new AnthropicVertexClient(new AnthropicVertexCredentials(region, projectId));

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 100,
      Messages = [new() { Role = Role.User, Content = "Hey Claude!" }]
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  import (
  	"context"
  	"fmt"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/vertex"
  )
  // ...
  	// 多区域标识符："us" 或 "eu"
  	client := anthropic.NewClient(
  		vertex.WithGoogleAuth(context.Background(), "us", "MY_PROJECT_ID"),
  	)

  	message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeOpus5,
  		MaxTokens: 100,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hey Claude!")),
  		},
  	})
  	if err != nil {
  		panic(err)
  	}
  	fmt.Printf("%+v\n", message)
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;
  import com.anthropic.vertex.backends.VertexBackend;
  import com.google.auth.oauth2.GoogleCredentials;

  void main() throws Exception {
      // 多区域标识符："us" 或 "eu"
      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(
              VertexBackend.builder()
                  .googleCredentials(GoogleCredentials.getApplicationDefault())
                  .region("us")
                  .project("MY_PROJECT_ID")
                  .build()
          )
          .build();

      var message = client
          .messages()
          .create(
              MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5)
                  .maxTokens(100)
                  .addUserMessage("Hey Claude!")
                  .build()
          );

      IO.println(message);
  }
  ```

  ```php PHP
  <?php

  use Anthropic\Vertex;

  $client = Vertex\Client::fromEnvironment(
      location: 'us', // Multi-region identifier: "us" or "eu"
      projectId: 'MY_PROJECT_ID',
  );

  $message = $client->messages->create(
      maxTokens: 100,
      messages: [
          ['role' => 'user', 'content' => 'Hey Claude!']
      ],
      model: 'claude-opus-5',
  );
  $textBlock = array_find($message->content, static fn ($block): bool => $block->type === 'text');
  echo $textBlock->text;
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::VertexClient.new(
    region: "us", # Multi-region identifier: "us" or "eu"
    project_id: "MY_PROJECT_ID"
  )

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 100,
    messages: [{role: "user", content: "Hey Claude!"}]
  )

  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

**使用区域端点：**

指定一个特定区域，例如 `"us-east5"` 或 `"europe-west1"`：

<CodeGroup>
  ```bash cURL
  # 特定区域端点支持 Claude Sonnet 4.6 及更早版本；较新的模型使用全球或多区域端点
  MODEL_ID=claude-sonnet-4-6
  LOCATION=us-east5 # Specify a specific region
  PROJECT_ID=MY_PROJECT_ID

  curl https://${LOCATION}-aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/publishers/anthropic/models/${MODEL_ID}:rawPredict \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    -H "Content-Type: application/json" \
    -d '{
      "anthropic_version": "vertex-2023-10-16",
      "messages": [{"role": "user", "content": "Hey Claude!"}],
      "max_tokens": 100
    }'
  ```

  ```bash CLI
  # ant CLI 不支持 Agent Platform。
  ```

  ```python Python
  from anthropic import AnthropicVertex

  project_id = "MY_PROJECT_ID"
  region = "us-east5"  # Specify a specific region

  client = AnthropicVertex(project_id=project_id, region=region)

  message = client.messages.create(
      # 特定区域端点支持 Claude Sonnet 4.6 及更早版本；较新的模型使用全球或多区域端点
      model="claude-sonnet-4-6",
      max_tokens=100,
      messages=[
          {
              "role": "user",
              "content": "Hey Claude!",
          }
      ],
  )
  print(message)
  ```

  ```typescript TypeScript
  import { AnthropicVertex } from "@anthropic-ai/vertex-sdk";

  const projectId = "MY_PROJECT_ID";
  const region = "us-east5"; // Specify a specific region

  const client = new AnthropicVertex({
    projectId,
    region
  });

  const result = await client.messages.create({
    // 特定区域端点支持 Claude Sonnet 4.6 及更早版本；较新的模型使用全球或多区域端点
    model: "claude-sonnet-4-6",
    max_tokens: 100,
    messages: [
      {
        role: "user",
        content: "Hey Claude!"
      }
    ]
  });
  console.log(JSON.stringify(result, null, 2));
  ```

  ```csharp C#
  using Anthropic.Models.Messages;
  using Anthropic.Vertex;

  var projectId = "MY_PROJECT_ID";
  var region = "us-east5"; // Specify a specific region

  var client = new AnthropicVertexClient(new AnthropicVertexCredentials(region, projectId));

  var parameters = new MessageCreateParams
  {
      // 特定区域端点支持 Claude Sonnet 4.6 及更早版本；较新的模型使用全球或多区域端点
      Model = Model.ClaudeSonnet4_6,
      MaxTokens = 100,
      Messages = [new() { Role = Role.User, Content = "Hey Claude!" }]
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  import (
  	"context"
  	"fmt"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/vertex"
  )
  // ...
  	// 指定特定区域
  	client := anthropic.NewClient(
  		vertex.WithGoogleAuth(context.Background(), "us-east5", "MY_PROJECT_ID"),
  	)

  	message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		// 特定区域端点支持 Claude Sonnet 4.6 及更早版本；较新的模型使用全球或多区域端点
  		Model:     anthropic.ModelClaudeSonnet4_6,
  		MaxTokens: 100,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hey Claude!")),
  		},
  	})
  	if err != nil {
  		panic(err)
  	}
  	fmt.Printf("%+v\n", message)
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;
  import com.anthropic.vertex.backends.VertexBackend;
  import com.google.auth.oauth2.GoogleCredentials;

  void main() throws Exception {
      // 使用默认的 Google Cloud 凭据并指定特定区域
      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(
              VertexBackend.builder()
                  .googleCredentials(GoogleCredentials.getApplicationDefault())
                  .region("us-east5") // Specify a specific region
                  .project("MY_PROJECT_ID")
                  .build()
          )
          .build();

      var message = client
          .messages()
          .create(
              MessageCreateParams.builder()
                  // 特定区域端点支持 Claude Sonnet 4.6 及更早版本；较新的模型使用全球或多区域端点
                  .model(Model.CLAUDE_SONNET_4_6)
                  .maxTokens(100)
                  .addUserMessage("Hey Claude!")
                  .build()
          );

      IO.println(message);
  }
  ```

  ```php PHP
  <?php

  use Anthropic\Vertex;

  $client = Vertex\Client::fromEnvironment(
      location: 'us-east5',
      projectId: 'MY_PROJECT_ID',
  );

  $message = $client->messages->create(
      maxTokens: 100,
      messages: [
          ['role' => 'user', 'content' => 'Hey Claude!']
      ],
      // 特定区域端点支持 Claude Sonnet 4.6 及更早版本；较新的模型使用全球或多区域端点
      model: 'claude-sonnet-4-6',
  );
  echo $message->content[0]->text;
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::VertexClient.new(
    region: "us-east5", # Specify a specific region
    project_id: "MY_PROJECT_ID"
  )

  message = client.messages.create(
    # 特定区域端点支持 Claude Sonnet 4.6 及更早版本；较新的模型使用全球或多区域端点
    model: "claude-sonnet-4-6",
    max_tokens: 100,
    messages: [{role: "user", content: "Hey Claude!"}]
  )

  puts message.content.first.text
  ```
</CodeGroup>

<Note>
  Claude Mythos Preview 是一个研究预览版，面向 Agent Platform 上受邀的客户提供。有关更多信息，请参阅 [Project Glasswing](https://anthropic.com/glasswing)。
</Note>

## 其他资源

* **Agent Platform 定价：** [cloud.google.com 上的生成式 AI 定价](https://cloud.google.com/vertex-ai/generative-ai/pricing)
* **Claude 模型文档：** [Agent Platform 上的 Claude](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude)
* **Google 博客文章：** [Claude 模型的全球端点](https://cloud.google.com/blog/products/ai-machine-learning/global-endpoint-for-claude-models-generally-available-on-vertex-ai)
* **Anthropic 定价详情：** [云平台定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#cloud-platform-pricing)
