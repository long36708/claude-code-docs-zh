---
title: MCP 隧道
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview
description: 将 Claude 安全地连接到运行在您私有网络中的 MCP 服务器，无需开放入站端口或将服务暴露到公共互联网。
---

MCP 隧道（MCP tunnels）让您可以将 Claude 连接到运行在您私有网络内部的 "Model Context Protocol"，即 MCP 服务器。流量通过仅出站的连接传输，因此您无需开放入站防火墙端口、将服务暴露到公共互联网，或在您的源站上将 Anthropic 的 IP 范围加入允许列表。

<Note>
  MCP 隧道目前处于研究预览阶段。[申请访问权限](https://claude.com/form/claude-managed-agents)以试用。它们按"原样"提供，不附带任何正常运行时间、支持或连续性承诺，并且依赖于第三方网络提供商（Cloudflare），该提供商对底层传输不作任何可用性承诺。Anthropic 可能随时修改或停止提供 MCP 隧道。
</Note>

有关零数据保留和 HIPAA BAA 资格，请参阅 [API 和数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)。

## 工作原理

[隧道栈](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)（tunnel stack）由运行在您网络内部的两个组件组成：

* **[cloudflared](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)：** Cloudflare 的开源隧道连接器。它发起到[隧道边缘](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)（tunnel edge）的仅出站连接，并将来自 Anthropic 的加密流量传送到您的代理。
* **[代理（Proxy）](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)：** Anthropic 的路由组件。它终止[内层 TLS](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)（inner TLS），验证上游 IP 是否位于允许的范围内，并根据主机名将每个请求路由到正确的[上游 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)。

您暴露的每个 MCP 服务器都会在您的隧道域名下获得一个主机名（例如 `docs.<your-tunnel-domain>`）。您可以在 Claude Console 中将这些主机名附加到 Managed Agent 会话，或通过 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)将它们传递给 Messages API。

## 前提条件

在部署之前，请确保您具备：

* 一个部署目标：Kubernetes 集群，或安装了 Docker 和 Docker Compose 的虚拟机。

* 一个隧道。在 Claude Console 中创建一个（请参阅[创建隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#create-a-tunnel)）或通过 API 创建；Helm chart 的设置钩子也可以在安装期间为您创建一个。

* 一种让您的栈向 Tunnels API 进行身份验证的方式。选择其一：

  * **[编程式访问](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#credential-provisioning)（推荐）。** 在创建隧道时设置 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)（工作负载身份联合）。您的栈从您的身份提供商签发短期 API 令牌，获取隧道令牌，并自动生成和注册 CA 证书。需要管理联合规则的权限、一个已注册的 OIDC 颁发者，以及一条具有 `workspace:manage_tunnels` 作用域的联合规则。
  * **[手动](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#credential-provisioning)。** 自行提供静态凭据：来自 Console 的隧道令牌，以及由您在其中注册的 CA 签名的服务器证书。请参阅[获取连接详情](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#get-the-connection-details)和[添加 CA 证书](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#add-a-ca-certificate)。

* 一个或多个运行在您私有网络中的 MCP 服务器。示例请参阅[远程 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/remote-mcp-servers)。

* [网络要求](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview#network-requirements)中列出的出站连接能力。

### 网络要求

| 组件          | 目标                                          | 端口 / 协议        | 使用阶段    |
| ----------- | ------------------------------------------- | -------------- | ------- |
| 设置组件        | `api.anthropic.com`                         | 443 TCP        | 配置和令牌轮换 |
| cloudflared | 隧道边缘（`198.41.192.0/19`、`2606:4700:a0::/44`） | 7844 TCP 和 UDP | 运行时     |
| 代理          | 您的上游 MCP 服务器                                | 按配置            | 运行时     |

## 安全模型

### 安全层

三个独立的层保护每个请求：

| 层                                  | 防护对象                       |
| ---------------------------------- | -------------------------- |
| Anthropic 与传输提供商之间的外层 mTLS，带 IP 验证 | 未经授权的客户端访问隧道               |
| 从 Anthropic 后端到您的代理的内层 TLS         | 传输提供商或任何网络中间方对有效负载的检查      |
| 每个 MCP 服务器上的 OAuth                 | 已通过身份验证的隧道流量对 MCP 工具的未授权使用 |

隧道传输运行在 Cloudflare 的网络上。由于代理使用只有您持有的证书终止内层 TLS，Cloudflare 无法读取请求或响应的有效负载。在注册 CA 证书之前，Anthropic 不会连接到隧道，因此有效负载在穿越 Cloudflare 网络时始终是加密的。Cloudflare 确实会接收连接元数据；请参阅[传输提供商可以观察到的内容](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview#what-the-transport-provider-can-observe)。

### 责任共担模型

| Anthropic 负责              | 您的组织负责                                              |
| ------------------------- | --------------------------------------------------- |
| 隧道访问控制                    | 通过您的隧道传输的所有内容和流量，以及遵守适用的第三方可接受使用政策（包括 Cloudflare 的） |
| 在连接到您的代理之前验证您的 CA 证书      | 遵守这些页面上的部署指南                                        |
| 确保 Claude 仅向您的组织拥有的隧道发送请求 | 保护隧道令牌和 TLS 私钥                                      |
|                           | 管理服务器证书并在其过期前续期                                     |
|                           | 在每个 MCP 服务器上配置 OAuth                                |
|                           | 限制代理和 MCP 服务器的网络访问                                  |
|                           | 如果您怀疑发生泄露，通知 Anthropic                              |

<Warning>
  如果攻击者获得了您的隧道令牌**以及**您的某个 TLS 私钥，他们就可以冒充您的代理并读取 MCP 请求有效负载。请将两者都视为高价值机密。有关加固指南，请参阅 [MCP 隧道安全](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/security)。
</Warning>

### 传输提供商可以观察到的内容

Cloudflare 提供出站传输。它无法读取 MCP 请求或响应的有效负载，但确实会接收以下连接元数据：

* 运行 cloudflared 的主机的出口 IP 地址
* cloudflared 主机指纹
* 连接时序和字节量
* 分配给您隧道的 `*.tunnel.anthropic.com` 子域名

Anthropic 与 Cloudflare 的协议限制了 Cloudflare 对这些遥测数据的使用。Cloudflare 在本研究预览中充当子处理方。

## 部署隧道

如果您是 MCP 隧道的新用户，请先从快速入门开始，在本地获得一个可用的隧道，然后再配置生产部署。

<CardGroup cols={2}>
  <Card title="快速入门" icon="rocket" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/quickstart">
    获得可用隧道的最短路径：使用 Docker Compose 和示例 MCP 服务器。
  </Card>

  <Card title="使用 Helm 部署" icon="stack" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-helm">
    使用 Anthropic Helm chart 在 Kubernetes 集群上安装。
  </Card>

  <Card title="使用 Docker Compose 部署" icon="cube" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose">
    使用 Docker Compose 在虚拟机上安装。
  </Card>
</CardGroup>

如何在它们之间选择：

* **部署目标**

  * 部署到 Kubernetes 时使用 **Helm**。
  * 单主机或本地测试使用 **Docker Compose**。

* **设置阶段的身份验证**

  * 当您拥有 OIDC 身份提供商（例如 Kubernetes 集群、云 IAM 或 SPIFFE）时，使用**编程式访问**（通过 Workload Identity Federation）。
  * 当您没有，或正在测试时，使用**手动凭据**。

## 使用通过隧道连接的 MCP 服务器

一旦您的隧道处于活动状态（它拥有一个活动的 CA 证书并且您的隧道栈已连接），上游 MCP 服务器即可从 Claude Managed Agents 和 Messages API 访问。

<Note>
  通过 Console 创建的 MCP 隧道不能作为 claude.ai 中的连接器使用。
</Note>

在这两种情况下，隧道都会将加密流量传送到您的 MCP 服务器，但不会向其进行身份验证。如果上游 MCP 服务器需要自己的身份验证（OAuth、bearer 令牌），请按照您对任何其他 MCP 服务器的方式提供；它与隧道无关。

### Managed Agents（Console）

1. 在 **Managed Agents > Sessions** 中，创建一个会话并选择 **Create new agent**，以便您可以编辑 MCP 服务器列表。
2. 点击 **+ MCP Server** 并打开下拉菜单。会话所在工作区中至少拥有一个活动证书的隧道会显示在列表顶部，位于公共连接器目录之上。
3. 选择隧道，并提供您的代理路由到特定 MCP 服务器的 **Subdomain**（子域名），以及上游 MCP 服务器期望的 **Path**（路径）。**Resolves to** 行会显示确切的 URL。

### Messages API

在 `mcp_servers` 数组中传递上游 MCP 服务器的 URL，方式与任何其他远程 MCP 服务器相同。请求正文和 `anthropic-beta` 标头遵循标准的 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)格式；只有 `url` 是隧道特有的。以下示例使用 MCP 连接器的 `mcp-client` beta 标头，它与 [Tunnels API](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/reference) 使用的 `mcp-tunnels` beta 是分开的。请在创建隧道的工作区中发出请求：使用该工作区的 API 密钥，或者如果您的密钥可以访问多个工作区，则将 [`anthropic-workspace-id` 标头](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)设置为该工作区。

URL 的主机为 `<subdomain>.<your-tunnel-domain>`。路径取决于您的上游 MCP 服务器，而非隧道：FastMCP 的 `streamable-http` 传输在 `/mcp` 提供服务，其他服务器可能使用 `/` 或自定义路径（请查阅服务器的文档）。代理会原样转发路径。

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
      "messages": [{"role": "user", "content": "Use the hello tool to greet tunnel."}],
      "mcp_servers": [
        {
          "type": "url",
          "url": "https://echo.YOUR_TUNNEL_DOMAIN_HERE/mcp",
          "name": "echo"
        }
      ],
      "tools": [{"type": "mcp_toolset", "mcp_server_name": "echo"}]
    }'
  ```

  ```bash CLI
  ant beta:messages create --beta mcp-client-2025-11-20 <<'YAML'
  model: claude-opus-5
  max_tokens: 1000
  messages:
    - role: user
      content: Use the hello tool to greet tunnel.
  mcp_servers:
    - type: url
      url: https://echo.YOUR_TUNNEL_DOMAIN_HERE/mcp
      name: echo
  tools:
    - type: mcp_toolset
      mcp_server_name: echo
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=1000,
      messages=[{"role": "user", "content": "Use the hello tool to greet tunnel."}],
      mcp_servers=[
          {
              "type": "url",
              "url": "https://echo.YOUR_TUNNEL_DOMAIN_HERE/mcp",
              "name": "echo",
          }
      ],
      tools=[{"type": "mcp_toolset", "mcp_server_name": "echo"}],
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
        content: "Use the hello tool to greet tunnel."
      }
    ],
    mcp_servers: [
      {
        type: "url",
        url: "https://echo.YOUR_TUNNEL_DOMAIN_HERE/mcp",
        name: "echo"
      }
    ],
    tools: [
      {
        type: "mcp_toolset",
        mcp_server_name: "echo"
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
      Model = Messages::Model.ClaudeOpus5,
      MaxTokens = 1000,
      Messages = new List<BetaMessageParam>
      {
          new() { Role = Role.User, Content = "Use the hello tool to greet tunnel." }
      },
      McpServers = new List<BetaRequestMcpServerUrlDefinition>
      {
          new()
          {
              Url = "https://echo.YOUR_TUNNEL_DOMAIN_HERE/mcp",
              Name = "echo"
          }
      },
      Tools = new List<BetaToolUnion>
      {
          new BetaMcpToolset("echo")
      },
      Betas = ["mcp-client-2025-11-20"]
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
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Use the hello tool to greet tunnel.")),
  	},
  	MCPServers: []anthropic.BetaRequestMCPServerURLDefinitionParam{
  		{
  			URL:  "https://echo.YOUR_TUNNEL_DOMAIN_HERE/mcp",
  			Name: "echo",
  		},
  	},
  	Tools: []anthropic.BetaToolUnionParam{
  		{OfMCPToolset: &anthropic.BetaMCPToolsetParam{
  			MCPServerName: "echo",
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
          .addUserMessage("Use the hello tool to greet tunnel.")
          .addMcpServer(BetaRequestMcpServerUrlDefinition.builder()
              .url("https://echo.YOUR_TUNNEL_DOMAIN_HERE/mcp")
              .name("echo")
              .build())
          .addTool(BetaMcpToolset.builder()
              .mcpServerName("echo")
              .build())
          .addBeta("mcp-client-2025-11-20")
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
          ['role' => 'user', 'content' => 'Use the hello tool to greet tunnel.']
      ],
      model: 'claude-opus-5',
      mcpServers: [
          [
              'type' => 'url',
              'url' => 'https://echo.YOUR_TUNNEL_DOMAIN_HERE/mcp',
              'name' => 'echo',
          ],
      ],
      tools: [
          [
              'type' => 'mcp_toolset',
              'mcpServerName' => 'echo',
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
      { role: "user", content: "Use the hello tool to greet tunnel." }
    ],
    mcp_servers: [
      {
        type: "url",
        url: "https://echo.YOUR_TUNNEL_DOMAIN_HERE/mcp",
        name: "echo"
      }
    ],
    tools: [
      {
        type: "mcp_toolset",
        mcp_server_name: "echo"
      }
    ],
    betas: ["mcp-client-2025-11-20"]
  )

  puts response
  ```
</CodeGroup>

有关向上游 MCP 服务器进行身份验证（`authorization_token`）以及其他 `mcp_servers` 选项，请参阅 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="安全" icon="lock" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/security">
    加固指南、凭据轮换和泄露响应。
  </Card>

  <Card title="故障排除" icon="wrench" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/troubleshooting">
    诊断连接、TLS 和路由问题。
  </Card>

  <Card title="参考" icon="book" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/reference">
    代理配置字段、Tunnels API、证书要求和设置组件。
  </Card>

  <Card title="MCP 连接器" icon="link" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector">
    从 Messages API 使用通过隧道连接的服务器。
  </Card>
</CardGroup>
