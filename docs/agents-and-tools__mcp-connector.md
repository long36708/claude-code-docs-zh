---
title: MCP 连接器
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector
description: 无需 MCP 客户端，直接从 Messages API 连接到远程 MCP 服务器，并对单个工具进行允许列表、拒绝列表或配置。
---

## Compatibility
- Status: Beta
- [Beta header](https://platform.claude.com/docs/en/api/beta-headers): `mcp-client-2025-11-20`
- [ZDR](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): not eligible
- Platforms: Claude API (beta), Claude Platform on AWS (beta), Microsoft Foundry (beta); not available on Amazon Bedrock, Google Cloud

Claude 的 "Model Context Protocol"，即 MCP 连接器功能使您能够直接从 Messages API 连接到远程 MCP 服务器，而无需单独的 MCP 客户端。

<Note>
  此功能的先前版本（`mcp-client-2025-04-04`）已弃用。请参阅[已弃用版本：mcp-client-2025-04-04](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#deprecated-version-mcp-client-2025-04-04)。
</Note>

## 主要功能

* **直接 API 集成：** 无需实现 MCP 客户端即可连接到 MCP 服务器
* **工具调用支持：** 通过 Messages API 访问 MCP 工具
* **灵活的工具配置：** 启用所有工具、将特定工具加入允许列表，或将不需要的工具加入拒绝列表
* **按工具配置：** 使用自定义设置配置单个工具
* **OAuth 身份验证：** 支持用于已认证服务器的 OAuth Bearer 令牌
* **多服务器：** 在单个请求中连接到多个 MCP 服务器

## Claude 何时使用 MCP 工具

一旦连接了 MCP 服务器，当用户的请求与某个工具所描述的能力相对应时，Claude 就会调用该工具——无论是显式的（"在 Jira 中搜索未解决的 bug"）还是隐式的（在附加了 Jira 服务器的情况下询问"是什么阻碍了发布？"）。

对于有关已连接服务的一般知识性问题，Claude **不会**调用 MCP 工具。在附加了 Notion 服务器的情况下询问"Notion 数据库是如何工作的？"会直接得到回答；而询问"我的 Projects 数据库里有什么？"则会触发该工具。

您可以通过 system prompt（系统提示）来引导 Claude 调用 MCP 工具的积极程度。有关一般指导和示例措辞，请参阅 [Claude 何时使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview#when-claude-uses-tools)。

## 限制

* 在 [MCP 规范](https://modelcontextprotocol.io/introduction#explore-mcp)的功能集中，目前仅支持[工具调用](https://modelcontextprotocol.io/docs/concepts/tools)。
* 服务器必须通过 HTTP 公开暴露（支持 Streamable HTTP 和 SSE 两种传输方式）。本地 STDIO 服务器无法直接连接。

## 在 Messages API 中使用 MCP 连接器

MCP 连接器使用两个组件：

1. **MCP 服务器定义**（`mcp_servers` 数组）：定义服务器连接详细信息（URL、身份验证）
2. **MCP 工具集**（`tools` 数组）：配置要启用哪些工具以及如何配置它们

### 基本示例

此示例以默认配置启用 MCP 服务器中的所有工具：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: mcp-client-2025-11-20" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1000,
      "messages": [{"role": "user", "content": "What tools do you have available?"}],
      "mcp_servers": [
        {
          "type": "url",
          "url": "https://example-server.modelcontextprotocol.io/sse",
          "name": "example-mcp",
          "authorization_token": "YOUR_TOKEN"
        }
      ],
      "tools": [
        {
          "type": "mcp_toolset",
          "mcp_server_name": "example-mcp"
        }
      ]
    }'
  ```

  ```bash CLI
  ant beta:messages create --beta mcp-client-2025-11-20 <<'YAML'
  model: claude-opus-5
  max_tokens: 1000
  messages:
    - role: user
      content: What tools do you have available?
  mcp_servers:
    - type: url
      url: https://example-server.modelcontextprotocol.io/sse
      name: example-mcp
      authorization_token: YOUR_TOKEN
  tools:
    - type: mcp_toolset
      mcp_server_name: example-mcp
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=1000,
      messages=[{"role": "user", "content": "What tools do you have available?"}],
      mcp_servers=[
          {
              "type": "url",
              "url": "https://example-server.modelcontextprotocol.io/sse",
              "name": "example-mcp",
              "authorization_token": "YOUR_TOKEN",
          }
      ],
      tools=[{"type": "mcp_toolset", "mcp_server_name": "example-mcp"}],
      betas=["mcp-client-2025-11-20"],
  )

  print(response)
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic();

  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 1000,
    messages: [
      {
        role: "user",
        content: "What tools do you have available?"
      }
    ],
    mcp_servers: [
      {
        type: "url",
        url: "https://example-server.modelcontextprotocol.io/sse",
        name: "example-mcp",
        authorization_token: "YOUR_TOKEN"
      }
    ],
    tools: [
      {
        type: "mcp_toolset",
        mcp_server_name: "example-mcp"
      }
    ],
    betas: ["mcp-client-2025-11-20"]
  });

  console.log(response);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1000,
      Messages = new List<BetaMessageParam>
      {
          new() { Role = Role.User, Content = "What tools do you have available?" }
      },
      McpServers = new List<BetaRequestMcpServerUrlDefinition>
      {
          new()
          {
              Url = "https://example-server.modelcontextprotocol.io/sse",
              Name = "example-mcp",
              AuthorizationToken = "YOUR_TOKEN"
          }
      },
      Tools = new List<BetaToolUnion>
      {
          new BetaMcpToolset("example-mcp")
      },
      Betas = [AnthropicBeta.McpClient2025_11_20]
  };

  var message = await client.Beta.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Beta.Messages.New(context.TODO(), anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1000,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("What tools do you have available?")),
  	},
  	MCPServers: []anthropic.BetaRequestMCPServerURLDefinitionParam{
  		{
  			URL:                "https://example-server.modelcontextprotocol.io/sse",
  			Name:               "example-mcp",
  			AuthorizationToken: anthropic.String("YOUR_TOKEN"),
  		},
  	},
  	Tools: []anthropic.BetaToolUnionParam{
  		{OfMCPToolset: &anthropic.BetaMCPToolsetParam{
  			MCPServerName: "example-mcp",
  		}},
  	},
  	Betas: []anthropic.AnthropicBeta{
  		anthropic.AnthropicBetaMCPClient2025_11_20,
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.beta.messages.BetaMcpToolset;
  // ...
  import com.anthropic.models.beta.messages.BetaRequestMcpServerUrlDefinition;
  // ...

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1000L)
          .addUserMessage("What tools do you have available?")
          .addMcpServer(BetaRequestMcpServerUrlDefinition.builder()
              .url("https://example-server.modelcontextprotocol.io/sse")
              .name("example-mcp")
              .authorizationToken("YOUR_TOKEN")
              .build())
          .addTool(BetaMcpToolset.builder()
              .mcpServerName("example-mcp")
              .build())
          .addBeta(AnthropicBeta.MCP_CLIENT_2025_11_20)
          .build();

      BetaMessage response = client.beta().messages().create(params);
      IO.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $message = $client->beta->messages->create(
      maxTokens: 1000,
      messages: [
          ['role' => 'user', 'content' => 'What tools do you have available?']
      ],
      model: 'claude-opus-5',
      mcpServers: [
          [
              'type' => 'url',
              'url' => 'https://example-server.modelcontextprotocol.io/sse',
              'name' => 'example-mcp',
              'authorization_token' => 'YOUR_TOKEN',
          ],
      ],
      tools: [
          [
              'type' => 'mcp_toolset',
              'mcp_server_name' => 'example-mcp',
          ],
      ],
      betas: ['mcp-client-2025-11-20'],
  );

  echo $message;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 1000,
    messages: [
      { role: "user", content: "What tools do you have available?" }
    ],
    mcp_servers: [
      {
        type: "url",
        url: "https://example-server.modelcontextprotocol.io/sse",
        name: "example-mcp",
        authorization_token: "YOUR_TOKEN"
      }
    ],
    tools: [
      {
        type: "mcp_toolset",
        mcp_server_name: "example-mcp"
      }
    ],
    betas: ["mcp-client-2025-11-20"]
  )

  puts response
  ```
</CodeGroup>

## MCP 服务器配置

`mcp_servers` 数组中的每个 MCP 服务器定义连接详细信息：

```json
{
  "type": "url",
  "url": "https://example-server.modelcontextprotocol.io/sse",
  "name": "example-mcp",
  "authorization_token": "YOUR_TOKEN"
}
```

### 字段说明

| 属性                    | 类型     | 必填 | 说明                                                                                                                                                                                                                                     |
| --------------------- | ------ | -- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`                | string | 是  | 目前仅支持 "url"。                                                                                                                                                                                                                           |
| `url`                 | string | 是  | MCP 服务器的 URL。必须以 https\:// 开头。                                                                                                                                                                                                         |
| `name`                | string | 是  | 此 MCP 服务器的唯一标识符。必须被 `tools` 数组中恰好一个 MCPToolset 引用。                                                                                                                                                                                     |
| `authorization_token` | string | 否  | MCP 服务器要求时使用的 OAuth 授权令牌。有关如何获取令牌，请参阅[身份验证](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#authentication)；有关协议详细信息，请参阅 [MCP 规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)。 |

## MCP 工具集配置

MCPToolset 位于 `tools` 数组中，用于配置启用 MCP 服务器中的哪些工具以及应如何配置它们。

### 基本结构

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "example-mcp",
  "default_config": {
    "enabled": true,
    "defer_loading": false
  },
  "configs": {
    "specific_tool_name": {
      "enabled": true,
      "defer_loading": true
    }
  }
}
```

### 字段说明

| 属性                | 类型     | 必填 | 说明                                                                                                           |
| ----------------- | ------ | -- | ------------------------------------------------------------------------------------------------------------ |
| `type`            | string | 是  | 必须为 "mcp\_toolset"。                                                                                          |
| `mcp_server_name` | string | 是  | 必须与 `mcp_servers` 数组中定义的服务器名称匹配。                                                                             |
| `default_config`  | object | 否  | 应用于此工具集中所有工具的默认配置。`configs` 中的单个工具配置会覆盖这些默认值。                                                                |
| `configs`         | object | 否  | 按工具的配置覆盖。键为工具名称，值为配置对象。                                                                                      |
| `cache_control`   | object | 否  | 此工具集的 [prompt caching（提示缓存）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)缓存断点配置。 |

### 工具配置选项

每个工具（无论是在 `default_config` 中还是在 `configs` 中配置）都支持以下字段：

| 属性              | 类型      | 默认值     | 说明                                                                                                                        |
| --------------- | ------- | ------- | ------------------------------------------------------------------------------------------------------------------------- |
| `enabled`       | boolean | `true`  | 此工具是否启用。                                                                                                                  |
| `defer_loading` | boolean | `false` | 如果为 true，则工具描述最初不会发送给模型。与[工具搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)配合使用。 |

有关 Anthropic 提供的工具的完整目录以及 `defer_loading` 等可选属性，请参阅[工具参考](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-reference)。要在大型工具集中进行搜索，请参阅[工具搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)。

### 配置合并

配置值按以下优先级合并（从高到低）：

1. `configs` 中的工具特定设置
2. 工具集级别的 `default_config`
3. 系统默认值

示例：

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-calendar-mcp",
  "default_config": {
    "defer_loading": true
  },
  "configs": {
    "search_events": {
      "enabled": false
    }
  }
}
```

结果为：

* `search_events`：`enabled: false`（来自 configs），`defer_loading: true`（来自 default\_config）
* 所有其他工具：`enabled: true`（系统默认值），`defer_loading: true`（来自 default\_config）

## 常见配置模式

### 以默认配置启用所有工具

最简单的模式：启用服务器中的所有工具：

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-calendar-mcp"
}
```

### 允许列表：仅启用特定工具

将 `enabled: false` 设为默认值，然后显式启用特定工具：

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-calendar-mcp",
  "default_config": {
    "enabled": false
  },
  "configs": {
    "search_events": {
      "enabled": true
    },
    "create_event": {
      "enabled": true
    }
  }
}
```

### 拒绝列表：禁用特定工具

默认启用所有工具，然后显式禁用不需要的工具。在构建只读助手时，或者当您希望在状态更改之前有人工确认步骤时，建议将写入类或破坏性工具加入拒绝列表：

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-calendar-mcp",
  "configs": {
    "delete_all_events": {
      "enabled": false
    },
    "share_calendar_publicly": {
      "enabled": false
    }
  }
}
```

### 混合：带有按工具配置的允许列表

将允许列表与每个工具的自定义配置相结合：

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-calendar-mcp",
  "default_config": {
    "enabled": false,
    "defer_loading": true
  },
  "configs": {
    "search_events": {
      "enabled": true,
      "defer_loading": false
    },
    "list_events": {
      "enabled": true
    }
  }
}
```

在此示例中：

* `search_events` 已启用，且 `defer_loading: false`
* `list_events` 已启用，且 `defer_loading: true`（继承自 default\_config）
* 所有其他工具均被禁用

## 验证规则

API 强制执行以下验证规则：

* **服务器必须存在：** MCPToolset 中的 `mcp_server_name` 必须与 `mcp_servers` 数组中定义的服务器匹配
* **服务器必须被使用：** `mcp_servers` 中定义的每个 MCP 服务器必须被恰好一个 MCPToolset 引用
* **每个服务器对应唯一工具集：** 每个 MCP 服务器只能被一个 MCPToolset 引用
* **未知工具名称：** 如果 `configs` 中的工具名称在 MCP 服务器上不存在，后端会记录警告但不会返回错误（MCP 服务器的工具可用性可能是动态的）

## 响应内容类型

当 Claude 使用 MCP 工具时，响应中会包含两种新的内容块类型：

### MCP 工具使用块

```json
{
  "type": "mcp_tool_use",
  "id": "mcptoolu_014Q35RayjACSWkSj4X2yov1",
  "name": "echo",
  "server_name": "example-mcp",
  "input": { "param1": "value1", "param2": "value2" }
}
```

### MCP 工具结果块

```json
{
  "type": "mcp_tool_result",
  "tool_use_id": "mcptoolu_014Q35RayjACSWkSj4X2yov1",
  "is_error": false,
  "content": [
    {
      "type": "text",
      "text": "Hello"
    }
  ]
}
```

## 多个 MCP 服务器

您可以通过在 `mcp_servers` 中包含多个服务器定义，并在 `tools` 数组中为每个服务器包含对应的 MCPToolset，来连接到多个 MCP 服务器：

```json
{
  "model": "claude-opus-5",
  "max_tokens": 1000,
  "messages": [
    {
      "role": "user",
      "content": "Use tools from both mcp-server-1 and mcp-server-2 to complete this task"
    }
  ],
  "mcp_servers": [
    {
      "type": "url",
      "url": "https://mcp.example1.com/sse",
      "name": "mcp-server-1",
      "authorization_token": "TOKEN1"
    },
    {
      "type": "url",
      "url": "https://mcp.example2.com/sse",
      "name": "mcp-server-2",
      "authorization_token": "TOKEN2"
    }
  ],
  "tools": [
    {
      "type": "mcp_toolset",
      "mcp_server_name": "mcp-server-1"
    },
    {
      "type": "mcp_toolset",
      "mcp_server_name": "mcp-server-2",
      "default_config": {
        "defer_loading": true
      }
    }
  ]
}
```

当有许多工具可用时，Claude 会根据工具名称和描述进行选择。清晰、具体的工具描述可以提高选择的准确性。对于大型工具集（跨多个服务器的数十个工具），请考虑结合[工具搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)启用 [`defer_loading`](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#tool-configuration-options)，以便每次查询只呈现相关的工具。

## 身份验证

对于需要 OAuth 身份验证的 MCP 服务器，您需要获取访问令牌。MCP 连接器测试版支持在 MCP 服务器定义中传递 `authorization_token` 参数。 API 使用者应在发起 API 调用之前自行处理 OAuth 流程并获取访问令牌，并根据需要刷新令牌。

### 获取用于测试的访问令牌

MCP inspector 可以引导您完成获取测试用访问令牌的过程。

1. 使用以下命令运行 inspector。您的机器上需要安装 Node.js。

   ```bash
   npx @modelcontextprotocol/inspector
   ```

2. 在左侧边栏中，对于 **Transport type**，选择 **SSE** 或 **Streamable HTTP**。

3. 输入 MCP 服务器的 URL。

4. 在右侧区域，点击 **Need to configure authentication?** 后面的 **Open Auth Settings**。

5. 点击 **Quick OAuth Flow** 并在 OAuth 界面上进行授权。

6. 按照 inspector 中 **OAuth Flow Progress** 部分的步骤操作，并点击 **Continue**，直到到达 **Authentication complete**。

7. 复制 `access_token` 的值。

8. 将其粘贴到 MCP 服务器配置中的 `authorization_token` 字段。

### 使用访问令牌

通过上述任一 OAuth 流程获取访问令牌后，您就可以在 MCP 服务器配置中使用它：

```json
{
  "mcp_servers": [
    {
      "type": "url",
      "url": "https://example-server.modelcontextprotocol.io/sse",
      "name": "authenticated-server",
      "authorization_token": "YOUR_ACCESS_TOKEN_HERE"
    }
  ]
}
```

有关 OAuth 流程的详细说明，请参阅 MCP 规范中的[授权部分](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)。

## 客户端 MCP 辅助函数

如果您自行管理 MCP 客户端连接（例如，使用本地 stdio 服务器、MCP 提示或 MCP 资源），SDK 提供了在 MCP 类型与 Claude API 类型之间进行转换的辅助函数。当您将适用于您所用语言的 MCP SDK（例如 [TypeScript MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk)）与 Anthropic SDK 一起使用时，这可以省去手动编写转换代码。

<Note>
  当您拥有可通过 URL 访问的远程服务器且只需要工具支持时，请使用 [`mcp_servers` API 参数](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#using-the-mcp-connector-in-the-messages-api)。当您需要本地服务器、提示、资源，或需要通过基础 SDK 对连接进行更多控制时，请使用客户端辅助函数。
</Note>

### 安装

同时安装 Anthropic SDK 和 MCP SDK：

<Tabs>
  <Tab title="Python">
    MCP 辅助函数包含在 `mcp` extra 中，需要 Python 3.10 或更高版本：

    ```bash
    pip install "anthropic[mcp]"
    ```
  </Tab>

  <Tab title="TypeScript">
    ```bash
    npm install @anthropic-ai/sdk @modelcontextprotocol/sdk
    ```
  </Tab>

  <Tab title="C#">
    辅助函数位于单独的 `Anthropic.Mcp` 包中；MCP 客户端本身来自官方的 [ModelContextProtocol 包](https://www.nuget.org/packages/ModelContextProtocol)：

    ```bash
    dotnet add package Anthropic.Mcp
    dotnet add package ModelContextProtocol
    ```
  </Tab>

  <Tab title="Go">
    辅助函数位于 Go SDK 的 `mcp` 子包中，该子包基于 [MCP Go SDK](https://github.com/modelcontextprotocol/go-sdk) 构建：

    ```bash
    go get github.com/anthropics/anthropic-sdk-go/mcp
    ```
  </Tab>

  <Tab title="Java">
    辅助函数位于单独的 `anthropic-java-mcp` 构件中，需要 Java 17 或更高版本（核心 SDK 支持 Java 8）：

    <Tabs>
      <Tab title="Gradle">
        ```kotlin
        implementation("com.anthropic:anthropic-java-mcp:2.58.0")
        ```
      </Tab>

      <Tab title="Maven">
        ```xml
        <dependency>
            <groupId>com.anthropic</groupId>
            <artifactId>anthropic-java-mcp</artifactId>
            <version>2.58.0</version>
        </dependency>
        ```
      </Tab>
    </Tabs>
  </Tab>

  <Tab title="PHP">
    辅助函数使用官方的 [MCP PHP SDK](https://packagist.org/packages/mcp/sdk)：

    ```bash
    composer require "anthropic-ai/sdk" "guzzlehttp/guzzle:^7" "mcp/sdk"
    ```
  </Tab>

  <Tab title="Ruby">
    辅助函数使用官方的 [`mcp` gem](https://rubygems.org/gems/mcp)：

    ```bash
    bundle add anthropic mcp
    ```
  </Tab>
</Tabs>

### 可用的辅助函数

导入适用于您所用语言的辅助函数：

<CodeGroup exclude="shell">
  ```python Python
  from anthropic.lib.tools.mcp import (
      async_mcp_tool,
      mcp_message,
      mcp_resource_to_content,
      mcp_resource_to_file,
  )
  ```

  ```typescript TypeScript
  import {
    mcpTools,
    mcpMessages,
    mcpResourceToContent,
    mcpResourceToFile
  } from "@anthropic-ai/sdk/helpers/beta/mcp";
  ```

  ```csharp C#
  using Anthropic.Helpers.Beta;
  using Anthropic.Helpers.Beta.Mcp;
  ```

  ```go Go
  import (
  	"github.com/anthropics/anthropic-sdk-go/mcp"
  )

  ```

  ```java Java
  import com.anthropic.helpers.McpBetaTool;
  import com.anthropic.mcp.BetaMcp;
  ```

  ```php PHP
  use Anthropic\Lib\Tools\BetaMcp;
  ```

  ```ruby Ruby
  require "anthropic"

  # 这些辅助函数通过 Anthropic::Mcp 模块公开
  ```
</CodeGroup>

辅助函数的名称和确切签名遵循各语言的惯例；下表显示的是 TypeScript 形式：

| 辅助函数                             | 说明                                                                     |
| -------------------------------- | ---------------------------------------------------------------------- |
| `mcpTools(tools, mcpClient)`     | 将 MCP 工具转换为 Claude API 工具，以便与 `client.beta.messages.toolRunner()` 一起使用 |
| `mcpMessages(messages)`          | 将 MCP 提示消息转换为 Claude API 消息格式                                          |
| `mcpResourceToContent(resource)` | 将 MCP 资源转换为 Claude API 内容块                                             |
| `mcpResourceToFile(resource)`    | 将 MCP 资源转换为用于上传的文件对象                                                   |

### 使用 MCP 工具

转换 MCP 工具以便与 SDK 的[工具运行器](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner)一起使用，该运行器会自动处理工具执行：

<CodeGroup exclude="shell">
  ```python Python
  from anthropic.lib.tools.mcp import async_mcp_tool
  from mcp import ClientSession
  from mcp.client.stdio import StdioServerParameters, stdio_client

  client = AsyncAnthropic()


  async def main() -> None:
      # 连接到 MCP 服务器
      server_params = StdioServerParameters(command="mcp-server")
      async with stdio_client(server_params) as (read, write):
          async with ClientSession(read, write) as mcp_client:
              await mcp_client.initialize()

              # 列出工具并将其转换为 Claude API 格式
              tools_result = await mcp_client.list_tools()
              runner = client.beta.messages.tool_runner(
                  model="claude-opus-5",
                  max_tokens=1024,
                  messages=[
                      {"role": "user", "content": "What tools do you have available?"},
                  ],
                  tools=[async_mcp_tool(tool, mcp_client) for tool in tools_result.tools],
              )

              final_message = await runner.until_done()
              print(final_message)


  asyncio.run(main())
  ```

  ```typescript TypeScript
  import {
    mcpTools,
    type MCPCallToolResultLike,
    type MCPClientLike
  } from "@anthropic-ai/sdk/helpers/beta/mcp";
  import { Client } from "@modelcontextprotocol/sdk/client/index.js";
  import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

  const anthropic = new Anthropic();

  // 连接到 MCP 服务器
  const transport = new StdioClientTransport({ command: "mcp-server", args: [] });
  const mcpClient = new Client({ name: "my-client", version: "1.0.0" });
  await mcpClient.connect(transport);

  // 列出工具并将其转换为 Claude API 格式
  const { tools } = await mcpClient.listTools();

  // MCP SDK 的 callTool 返回类型仍包含一种旧版结果形态，
  // 而 mcpTools 不接受该形态；需收窄类型。待 MCPClientLike 放宽后移除此处理。
  const mcpClientForTools: MCPClientLike = {
    callTool: (params) => mcpClient.callTool(params) as Promise<MCPCallToolResultLike>
  };

  const finalMessage = await anthropic.beta.messages.toolRunner({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "What tools do you have available?" }],
    tools: mcpTools(tools, mcpClientForTools)
  });

  console.log(finalMessage);
  ```

  ```csharp C#
  using Anthropic.Helpers.Beta;
  using Anthropic.Helpers.Beta.Mcp;
  using Anthropic.Models.Beta.Messages;
  using ModelContextProtocol.Client;
  using Messages = Anthropic.Models.Messages;

  var anthropic = new AnthropicClient();

  // 连接到 MCP 服务器
  await using var mcpClient = await McpClient.CreateAsync(
      new StdioClientTransport(new StdioClientTransportOptions { Command = "mcp-server" })
  );

  // 列出工具并将其转换为 Claude API 格式
  var tools = await BetaMcp.ListToolsAsync(mcpClient);
  var runner = anthropic.Beta.Messages.ToolRunner(
      new MessageCreateParams
      {
          Model = Messages::Model.ClaudeOpus5,
          MaxTokens = 1024,
          Messages =
          [
              new BetaMessageParam
              {
                  Role = Role.User,
                  Content = "What tools do you have available?",
              },
          ],
      },
      tools
  );

  var finalMessage = await runner.RunUntilDoneAsync();
  Console.WriteLine(finalMessage);
  ```

  ```go Go
  import (
  // ...

  // ...
  	"github.com/anthropics/anthropic-sdk-go/mcp"
  	mcpsdk "github.com/modelcontextprotocol/go-sdk/mcp"
  )

  func main() {
  	client := anthropic.NewClient()
  	ctx := context.Background()

  	// 连接到 MCP 服务器
  	mcpClient := mcpsdk.NewClient(&mcpsdk.Implementation{Name: "my-client", Version: "1.0.0"}, nil)
  	session, err := mcpClient.Connect(ctx, &mcpsdk.CommandTransport{Command: exec.Command("mcp-server")}, nil)
  	if err != nil {
  		log.Fatal(err)
  	}
  	defer session.Close()

  	// 列出工具并将其转换为 Claude API 格式
  	toolsResult, err := session.ListTools(ctx, nil)
  	if err != nil {
  		log.Fatal(err)
  	}
  	betaTools, err := mcp.NewBetaTools(toolsResult.Tools, session)
  	if err != nil {
  		log.Fatal(err)
  	}

  	runner := client.Beta.Messages.NewToolRunner(betaTools, anthropic.BetaToolRunnerParams{
  		BetaMessageNewParams: anthropic.BetaMessageNewParams{
  			Model:     anthropic.ModelClaudeOpus5,
  			MaxTokens: 1024,
  			Messages: []anthropic.BetaMessageParam{
  				anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("What tools do you have available?")),
  			},
  		},
  	})

  	finalMessage, err := runner.RunToCompletion(ctx)
  	if err != nil {
  		log.Fatal(err)
  	}
  	fmt.Println(finalMessage.RawJSON())
  }

  ```

  ```java Java
  import com.anthropic.helpers.BetaToolRunner;
  import com.anthropic.helpers.McpBetaTool;
  import com.anthropic.mcp.BetaMcp;
  import com.anthropic.models.beta.messages.BetaMessage;
  import com.anthropic.models.beta.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;
  import io.modelcontextprotocol.client.McpClient;
  import io.modelcontextprotocol.client.McpSyncClient;
  import io.modelcontextprotocol.client.transport.ServerParameters;
  import io.modelcontextprotocol.client.transport.StdioClientTransport;
  import io.modelcontextprotocol.json.McpJsonDefaults;
  import io.modelcontextprotocol.spec.McpSchema;
  // ...

  void main() throws Exception {
      AnthropicClient anthropic = AnthropicOkHttpClient.fromEnv();

      // 连接到 MCP 服务器
      StdioClientTransport transport = new StdioClientTransport(
              ServerParameters.builder("mcp-server").build(), McpJsonDefaults.getMapper());

      try (McpSyncClient mcpClient = McpClient.sync(transport)
              .clientInfo(new McpSchema.Implementation("my-client", "1.0.0"))
              .build()) {

          mcpClient.initialize();

          // 列出工具并将其转换为 Claude API 格式
          List<McpBetaTool> betaTools = BetaMcp.mcpTools(mcpClient.listTools().tools(), mcpClient);

          MessageCreateParams params = MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5)
                  .maxTokens(1024L)
                  .addUserMessage("What tools do you have available?")
                  .addTools(betaTools)
                  .build();

          // 运行器在每个助手轮次产生一条消息；最后一条是最终响应
          BetaToolRunner runner = anthropic.beta().messages().toolRunner(params);
          BetaMessage finalMessage = null;
          for (BetaMessage message : runner) {
              finalMessage = message;
          }
          IO.println(finalMessage);
      }
  }
  ```

  ```php PHP
  use Anthropic\Lib\Tools\BetaMcp;
  use Mcp\Client;
  use Mcp\Client\Transport\HttpTransport;

  $anthropic = new Anthropic();

  // 连接到 MCP 服务器。PHP MCP 客户端通过 HTTP 连接；请将其
  // 指向您服务器的端点。
  $mcp = Client::builder()->build();
  $mcp->connect(new HttpTransport('http://localhost:8000/mcp'));

  // 列出工具并将其转换为 Claude API 所需的格式
  $runner = $anthropic->beta->messages->toolRunner(
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'What tools do you have available?']],
      model: 'claude-opus-5',
      tools: BetaMcp::tools($mcp->listTools()->tools, $mcp),
  );

  echo $runner->runUntilDone(), "\n";
  ```

  ```ruby Ruby
  require "mcp"

  anthropic = Anthropic::Client.new

  # 连接到 MCP 服务器
  transport = MCP::Client::Stdio.new(command: "mcp-server")
  mcp_client = MCP::Client.new(transport: transport)
  mcp_client.connect

  # 列出工具并将其转换为 Claude API 格式
  runner = anthropic.beta.messages.tool_runner(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "What tools do you have available?" }],
    tools: Anthropic::Mcp.tools(mcp_client.tools, mcp_client)
  )

  final_message = runner.run_until_finished.last
  puts final_message
  ```
</CodeGroup>

### 使用 MCP 提示

将 MCP 提示消息转换为 Claude API 消息格式：

<CodeGroup exclude="shell">
  ```python Python
  from anthropic.lib.tools.mcp import mcp_message

  prompt = await mcp_client.get_prompt(name="my-prompt")
  response = await client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[mcp_message(message) for message in prompt.messages],
  )

  print(response)
  ```

  ```typescript TypeScript
  import { mcpMessages } from "@anthropic-ai/sdk/helpers/beta/mcp";

  const { messages } = await mcpClient.getPrompt({ name: "my-prompt" });
  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: mcpMessages(messages)
  });

  console.log(response);
  ```

  ```csharp C#
  var prompt = await mcpClient.GetPromptAsync("my-prompt");
  var response = await anthropic.Beta.Messages.Create(
      new MessageCreateParams
      {
          Model = Messages::Model.ClaudeOpus5,
          MaxTokens = 1024,
          Messages = BetaMcp.Messages(prompt.Messages),
      }
  );

  Console.WriteLine(response);
  ```

  ```go Go
  prompt, err := session.GetPrompt(ctx, &mcpsdk.GetPromptParams{Name: "my-prompt"})
  if err != nil {
  	log.Fatal(err)
  }

  messages := make([]anthropic.BetaMessageParam, 0, len(prompt.Messages))
  for _, promptMessage := range prompt.Messages {
  	message, err := mcp.ToMessage(promptMessage)
  	if err != nil {
  		log.Fatal(err)
  	}
  	messages = append(messages, message)
  }

  response, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages:  messages,
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response.RawJSON())
  ```

  ```java Java
  McpSchema.GetPromptResult prompt = mcpClient.getPrompt(
          new McpSchema.GetPromptRequest("my-prompt", Map.of()));

  BetaMessage response = anthropic.beta().messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024L)
          .messages(BetaMcp.mcpMessages(prompt.messages()))
          .build());

  IO.println(response);
  ```

  ```php PHP
  $prompt = $mcp->getPrompt('my-prompt');

  $response = $anthropic->beta->messages->create(
      maxTokens: 1024,
      messages: array_map(BetaMcp::message(...), $prompt->messages),
      model: 'claude-opus-5',
  );

  echo $response, "\n";
  ```

  ```ruby Ruby
  prompt = mcp_client.get_prompt(name: "my-prompt")

  response = anthropic.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: prompt["messages"].map { |message| Anthropic::Mcp.message(message) }
  )

  puts response
  ```
</CodeGroup>

### 使用 MCP 资源

将 MCP 资源转换为可包含在消息中的内容块，或转换为用于上传的文件对象：

<CodeGroup exclude="shell">
  ```python Python
  from anthropic.lib.tools.mcp import (
      mcp_resource_to_content,
      mcp_resource_to_file,
  )

  # 作为消息中的内容块
  resource = await mcp_client.read_resource(uri="file:///path/to/doc.txt")
  response = await client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  mcp_resource_to_content(resource),
                  {"type": "text", "text": "Summarize this document"},
              ],
          }
      ],
  )
  print(response)

  # 作为文件上传
  file_resource = await mcp_client.read_resource(
      uri="file:///path/to/data.json",
  )
  uploaded = await client.files.upload(
      file=mcp_resource_to_file(file_resource),
  )
  print(uploaded.id)
  ```

  ```typescript TypeScript
  import { mcpResourceToContent, mcpResourceToFile } from "@anthropic-ai/sdk/helpers/beta/mcp";

  // 作为消息中的内容块
  const resource = await mcpClient.readResource({ uri: "file:///path/to/doc.txt" });
  const response = await anthropic.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          mcpResourceToContent(resource),
          { type: "text", text: "Summarize this document" }
        ]
      }
    ]
  });
  console.log(response);

  // 作为文件上传
  const fileResource = await mcpClient.readResource({ uri: "file:///path/to/data.json" });
  const uploaded = await anthropic.files.upload({ file: mcpResourceToFile(fileResource) });
  console.log(uploaded.id);
  ```

  ```csharp C#
  // 作为消息中的内容块
  var resource = await mcpClient.ReadResourceAsync("file:///path/to/doc.txt");
  var response = await anthropic.Beta.Messages.Create(
      new MessageCreateParams
      {
          Model = Messages::Model.ClaudeOpus5,
          MaxTokens = 1024,
          Messages =
          [
              new BetaMessageParam
              {
                  Role = Role.User,
                  Content = new BetaMessageParamContent(
                      [
                          BetaMcp.ResourceToContent(resource),
                          new BetaTextBlockParam { Text = "Summarize this document" },
                      ]
                  ),
              },
          ],
      }
  );

  Console.WriteLine(response);

  // 作为文件上传
  var fileResource = await mcpClient.ReadResourceAsync("file:///path/to/data.json");
  var (filename, data, mediaType) = BetaMcp.ResourceToFile(fileResource);

  // 显式构建文件部分，以便资源的文件名和 MIME 类型
  // 能够传递到上传中。
  var file = new BinaryContent { Stream = new MemoryStream(data), FileName = filename };
  if (mediaType is not null)
  {
      file.ContentType = new(mediaType);
  }

  var uploaded = await anthropic.Files.Upload(new FileUploadParams { File = file });
  Console.WriteLine(uploaded.ID);
  ```

  ```go Go
  // 作为消息中的内容块
  resource, err := session.ReadResource(ctx, &mcpsdk.ReadResourceParams{URI: "file:///path/to/doc.txt"})
  if err != nil {
  	log.Fatal(err)
  }
  block, err := mcp.ResourceToBlock(resource)
  if err != nil {
  	log.Fatal(err)
  }

  response, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(
  			// ResourceToBlock 返回 tool-result 内容联合类型；消息
  			// 内容是单独的联合类型，因此需重新包装共享的变体
  			// （mcp.ToMessage 在内部执行相同操作）。
  			anthropic.BetaContentBlockParamUnion{
  				OfText:     block.OfText,
  				OfImage:    block.OfImage,
  				OfDocument: block.OfDocument,
  			},
  			anthropic.NewBetaTextBlock("Summarize this document"),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response.RawJSON())

  // 作为文件上传
  fileResult, err := session.ReadResource(ctx, &mcpsdk.ReadResourceParams{URI: "file:///path/to/data.json"})
  if err != nil {
  	log.Fatal(err)
  }
  fileReader, err := mcp.ResourceToFile(fileResult)
  if err != nil {
  	log.Fatal(err)
  }
  uploaded, err := client.Files.Upload(ctx, anthropic.FileUploadParams{File: fileReader})
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(uploaded.ID)
  ```

  ```java Java
  // 作为消息中的内容块
  McpSchema.ReadResourceResult resource = mcpClient.readResource(
          new McpSchema.ReadResourceRequest("file:///path/to/doc.txt"));

  List<BetaContentBlockParam> content =
          new ArrayList<>(BetaMcp.mcpResourceContents(resource));
  content.add(BetaContentBlockParam.ofText(
          BetaTextBlockParam.builder().text("Summarize this document").build()));

  BetaMessage response = anthropic.beta().messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024L)
          .addUserMessageOfBetaContentBlockParams(content)
          .build());

  IO.println(response);

  // 作为文件上传
  McpSchema.ReadResourceResult fileResource = mcpClient.readResource(
          new McpSchema.ReadResourceRequest("file:///path/to/data.json"));

  McpResourceFile resourceFile = BetaMcp.mcpResourceFiles(fileResource).getFirst();

  // 显式构建文件部分，以便资源的文件名和 MIME 类型
  // 能够传递到上传中。
  MultipartField.Builder<InputStream> fileField = MultipartField.<InputStream>builder()
          .value(new ByteArrayInputStream(resourceFile.content()))
          .filename(resourceFile.filename());
  if (resourceFile.mimeType() != null) {
      fileField.contentType(resourceFile.mimeType());
  }

  var uploaded = anthropic.files().upload(FileUploadParams.builder()
          .file(fileField.build())
          .build());

  IO.println(uploaded.id());
  ```

  ```php PHP
  // 作为消息中的内容块
  $resource = $mcp->readResource('file:///path/to/doc.txt');

  $response = $anthropic->beta->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  BetaMcp::resourceToContent($resource),
                  ['type' => 'text', 'text' => 'Summarize this document'],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo $response, "\n";

  // 作为文件上传
  $fileResource = $mcp->readResource('file:///path/to/data.json');
  $file = $anthropic->files->upload(file: BetaMcp::resourceToFile($fileResource));
  echo $file->id, "\n";
  ```

  ```ruby Ruby
  # 作为消息中的内容块
  resource = mcp_client.read_resource(uri: "file:///path/to/doc.txt")

  response = anthropic.beta.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          *Anthropic::Mcp.resource_to_contents(resource),
          { type: "text", text: "Summarize this document" }
        ]
      }
    ]
  )

  puts response

  # 作为文件上传
  file_resource = mcp_client.read_resource(uri: "file:///path/to/data.json")
  file = Anthropic::Mcp.resource_to_files(file_resource).first
  uploaded_file = anthropic.files.upload(file: file)
  puts uploaded_file.id
  ```
</CodeGroup>

### 错误处理

如果某个 MCP 值不受 Claude API 支持，转换函数会抛出 `UnsupportedMCPValueError`（在 Go 中，辅助函数返回 `UnsupportedValueError`；在 Java 和 C# 中，它们抛出 `AnthropicInvalidDataException`）。这可能发生在不受支持的内容类型、MIME 类型或资源链接上（请在转换之前使用您的 MCP 客户端解析资源链接）。

## 批量请求

您可以在 [Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing) 请求中包含 `mcp_servers`。通过 Batches API 进行的 MCP 工具调用与常规 Messages API 请求中的调用定价相同。

## 数据保留

MCP 连接器不在 ZDR 安排的覆盖范围内。与 MCP 服务器交换的数据（包括工具定义和执行结果）将根据 Anthropic 的标准数据保留政策进行保留。

有关所有功能的 ZDR 资格，请参阅 [API 和数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。

## 迁移指南

如果您正在使用已弃用的 `mcp-client-2025-04-04` 测试版标头，请按照本指南迁移到新版本。

### 主要变更

1. **新的测试版标头：** 从 `mcp-client-2025-04-04` 更改为 `mcp-client-2025-11-20`
2. **工具配置已移动：** 工具配置现在以 MCPToolset 对象的形式位于 `tools` 数组中，而不是在 MCP 服务器定义中
3. **更灵活的配置：** 新模式支持允许列表、拒绝列表和按工具配置

### 迁移步骤

**之前（已弃用）：**

```json
{
  "model": "claude-opus-5",
  "max_tokens": 1000,
  "messages": [
    // ...
  ],
  "mcp_servers": [
    {
      "type": "url",
      "url": "https://mcp.example.com/sse",
      "name": "example-mcp",
      "authorization_token": "YOUR_TOKEN",
      "tool_configuration": {
        "enabled": true,
        "allowed_tools": ["tool1", "tool2"]
      }
    }
  ]
}
```

**之后（当前）：**

```json
{
  "model": "claude-opus-5",
  "max_tokens": 1000,
  "messages": [
    // ...
  ],
  "mcp_servers": [
    {
      "type": "url",
      "url": "https://mcp.example.com/sse",
      "name": "example-mcp",
      "authorization_token": "YOUR_TOKEN"
    }
  ],
  "tools": [
    {
      "type": "mcp_toolset",
      "mcp_server_name": "example-mcp",
      "default_config": {
        "enabled": false
      },
      "configs": {
        "tool1": {
          "enabled": true
        },
        "tool2": {
          "enabled": true
        }
      }
    }
  ]
}
```

### 常见迁移模式

| 旧模式                                       | 新模式                                                                 |
| ----------------------------------------- | ------------------------------------------------------------------- |
| 无 `tool_configuration`（所有工具均启用）           | 不带 `default_config` 或 `configs` 的 MCPToolset                        |
| `tool_configuration.enabled: false`       | 带有 `default_config.enabled: false` 的 MCPToolset                     |
| `tool_configuration.allowed_tools: [...]` | 带有 `default_config.enabled: false` 并在 `configs` 中启用特定工具的 MCPToolset |

## 已弃用版本：mcp-client-2025-04-04

<Note type="warning">
  此版本已弃用。请使用上述[迁移指南](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#migration-guide)迁移到 `mcp-client-2025-11-20`。
</Note>

MCP 连接器的先前版本将工具配置直接包含在 MCP 服务器定义中：

```json
{
  "mcp_servers": [
    {
      "type": "url",
      "url": "https://example-server.modelcontextprotocol.io/sse",
      "name": "example-mcp",
      "authorization_token": "YOUR_TOKEN",
      "tool_configuration": {
        "enabled": true,
        "allowed_tools": ["example_tool_1", "example_tool_2"]
      }
    }
  ]
}
```

### 已弃用字段说明

| 属性                                 | 类型      | 说明                                                  |
| ---------------------------------- | ------- | --------------------------------------------------- |
| `tool_configuration`               | object  | **已弃用：** 请改用 `tools` 数组中的 MCPToolset                |
| `tool_configuration.enabled`       | boolean | **已弃用：** 请使用 MCPToolset 中的 `default_config.enabled` |
| `tool_configuration.allowed_tools` | array   | **已弃用：** 请使用 MCPToolset 中带有 `configs` 的允许列表模式       |
