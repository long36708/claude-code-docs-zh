---
title: 身份验证
url: https://platform.claude.com/docs/zh-CN/manage-claude/authentication
description: 使用 API 密钥、Workload Identity Federation 或 App Attest 对 Claude API 进行身份验证。
---

Claude API 支持三种请求身份验证方式：

| 方法                                                                                                                               | 凭证                                          | 最适合                                                                       |
| -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------- |
| [API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#api-keys)                                           | `x-api-key` 标头中的静态 `sk-ant-api...` 密钥       | 本地开发、原型设计、脚本，以及由您控制密钥存储的服务器                                               |
| [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#workload-identity-federation) | 由您的身份提供商的身份令牌交换而来的短期 bearer 令牌              | 云平台（AWS、Google Cloud、Azure）上的生产工作负载、CI/CD 流水线和 Kubernetes，适用于您希望消除静态密钥的场景 |
| [App Attest](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#app-attest)                                     | 颁发给您已注册的 iOS 或 macOS 应用的真实、经过证明的安装实例的短期访问令牌 | 分发给最终用户的 iOS 和 macOS 应用，应用直接调用 Claude API，无需后端或代理                         |

API 密钥和 Workload Identity Federation 授予对 Claude API 端点的相同访问权限。选择 API 密钥可快速上手：个人密钥用于您自己的开发，服务账户密钥用于任何共享用途。当您的工作负载已经拥有可以联合的平台颁发身份时，请迁移到 Workload Identity Federation。对于您分发给最终用户的 iOS 和 macOS 应用，请使用 App Attest。

## API 密钥

"API key"（API 密钥）是您在 Claude Console 中生成的静态密钥，并在每个请求的 `x-api-key` 标头中发送。

### 密钥类型

创建密钥时，您需要选择其类型，这决定了密钥可以做什么、在哪里有效以及何时停止工作：

| 密钥类型          | 代表身份                                                                                                         | 有效范围                                                                  | 停止工作的条件                                                                            |
| ------------- | ------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **个人密钥**      | 您本人（用户），具有您的角色和权限                                                                                            | 单个工作区，或您的角色允许使用 API 的工作区，在创建密钥时选择                                     | 您失去对组织的访问权限，或者对于单工作区密钥，失去对该工作区的访问权限。当您被从组织中移除时，个人密钥会被归档。如果您被重新邀请，请创建新密钥；已归档的密钥不会恢复 |
| **服务账户密钥**    | 一个[服务账户](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#service-accounts) | 单个工作区，或服务账户有权访问的任何内容，在创建密钥时选择。服务账户有权访问 Default Workspace 以及它已被添加到的工作区 | 服务账户被归档，或者对于单工作区密钥，服务账户被从该工作区中移除                                                   |
| **工作区密钥**（旧版） | 无人：它属于创建它的工作区                                                                                                | 该工作区                                                                  | 它过期、被禁用或删除，或其工作区被归档，无论其创建者是否离开组织                                                   |

个人密钥和服务账户密钥是基于身份的：每个密钥都属于您的组织已在管理的某个用户或服务账户，并且每个请求都以该身份执行。当该身份被从组织中移除时，密钥即停止工作。这意味着密钥不会意外地比拥有它们的人员或工作负载存续更久。对于新的集成，请优先使用它们而非工作区密钥。

将个人密钥用于您自己的开发和脚本。共享的个人密钥以一个人的身份行事，并会在该人离开时失效。对于共享或自动化的工作负载（CI、生产服务），请让组织管理员创建一个服务账户，以便工作负载拥有自己的身份。

工作区 API 密钥仍然有效，但应被视为旧版；建议优先使用基于身份的密钥或 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)。如需迁移，请参阅[替换工作区 API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#replacing-workspace-api-keys)。

### 创建和使用密钥

* **创建密钥：** 在 Claude Console 中前往 [Settings → API keys](https://platform.claude.com/settings/keys)，然后点击 **Create key**。为密钥命名并选择[过期时间](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-expiration)。对于个人密钥，将 **Linked account** 设置为您自己；对于在多个用户之间共享的密钥，将其设置为某个服务账户。您还可以将密钥的范围限定到特定工作区，这样在以后的请求中就无需手动设置工作区 ID。
* **使用密钥：** 在直接 HTTP 请求中设置 `x-api-key` 标头，或设置 `ANTHROPIC_API_KEY` 环境变量，[客户端 SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview) 会自动读取它。

```http
POST /v1/messages
x-api-key: YOUR_API_KEY
anthropic-version: 2023-06-01
content-type: application/json
```

请将 API 密钥存储在密钥管理器中，定期轮换，并禁用或删除任何您怀疑已泄露的密钥。在 [API keys 页面](https://platform.claude.com/settings/keys)上，**Disable** 是可逆的（Admin API 会将密钥的 `status` 报告为 `"inactive"`，而 **Re-enable** 会将其恢复为 `"active"`），而 **Delete** 是永久性的：密钥会被归档，并仍会以 `status: "archived"` 出现在 [List API Keys](https://platform.claude.com/docs/zh-CN/api/admin/api_keys/list) 中。已过期的密钥只能被删除。您还可以在创建密钥时设置[过期时间](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-expiration)，以限制泄露的凭证可被使用的时长。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello, Claude"}]
    }'
  ```

  ```python Python
  client = Anthropic(api_key="my-anthropic-api-key")
  # 或者，在环境变量中设置 ANTHROPIC_API_KEY 后：
  client = Anthropic()
  ```

  ```typescript TypeScript
  const client = new Anthropic({ apiKey: "my-anthropic-api-key" });
  // 或者，在环境中设置了 ANTHROPIC_API_KEY 时：
  // const client = new Anthropic();
  ```

  ```go Go
  client := anthropic.NewClient(
  	option.WithAPIKey("sk-ant-api03-..."), // defaults to os.LookupEnv("ANTHROPIC_API_KEY")
  )
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;

  // 显式指定
  AnthropicClient client = AnthropicOkHttpClient.builder()
    .apiKey("my-anthropic-api-key")
    .build();

  // 从 ANTHROPIC_API_KEY（或 anthropic.apiKey 系统属性）读取
  AnthropicClient clientFromEnv = AnthropicOkHttpClient.fromEnv();
  ```

  ```csharp C#
  using Anthropic;

  AnthropicClient client = new() { ApiKey = "my-anthropic-api-key" };
  // 或者，在环境中设置 ANTHROPIC_API_KEY 后：
  // AnthropicClient client = new();
  ```

  ```php PHP
  // 从环境变量中读取 ANTHROPIC_API_KEY
  $client = new Client();
  // 或者显式传入密钥：
  $client = new Client(apiKey: 'my-anthropic-api-key');
  ```

  ```ruby Ruby
  anthropic = Anthropic::Client.new(api_key: "my-anthropic-api-key")
  # 或者，在环境变量中设置 ANTHROPIC_API_KEY 后：
  anthropic = Anthropic::Client.new
  ```

  ```bash CLI
  # zsh、bash 和 Windows 的对应写法请参阅 /docs/en/cli-sdks-libraries/cli/authentication#api-key
  export ANTHROPIC_API_KEY=sk-ant-api03-...
  ```
</CodeGroup>

### 选择工作区

为特定工作区创建的 API 密钥仅在该工作区中有效，使用这些密钥的 API 请求可以省略工作区 ID。

如果您的 API 密钥未限定到某个工作区，则必须在每个请求的 `anthropic-workspace-id` 标头中指定工作区 ID。请参阅以下示例，了解如何在请求或 SDK 中设置此标头。

[Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 仅在密钥未限定到特定工作区时才接受个人密钥或服务账户密钥。

您可以在 Claude Console 的 [Settings → Workspaces](https://platform.claude.com/settings/workspaces) 的 **ID** 列中找到工作区的 ID，或通过调用 [List Workspaces](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/list) 端点获取。两者都不会列出 Default Workspace 的 ID：请从在该工作区中运行的任何请求（例如，使用来自 Default Workspace 的工作区密钥发出的请求）的 `anthropic-workspace-id` [响应标头](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#identify-the-workspace-behind-an-api-response)中读取，或从 [List API Keys](https://platform.claude.com/docs/zh-CN/api/admin/api_keys/list) 中此类密钥的 `scope.workspace_id` 读取。

<CodeGroup>
  ```bash cURL
  # 对于多工作区密钥，每个请求都必须包含此项。
  # 对于单工作区密钥，请省略 anthropic-workspace-id 请求头。
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-workspace-id: wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello, Claude"}]
    }'
  ```

  ```bash CLI
  # 对于多工作区密钥，每条命令都必须指定。
  # 对于单工作区密钥，请省略 --workspace-id。
  ant messages create \
    --workspace-id wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello, Claude"}'
  ```

  ```python Python
  client = Anthropic()  # reads ANTHROPIC_API_KEY

  # 对于多工作区密钥，每个请求都必须包含此项。
  # 对于单工作区密钥，请省略 extra_headers。
  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello, Claude"}],
      extra_headers={"anthropic-workspace-id": "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"},
  )
  print(message.content)

  # 或者为此客户端发出的每个请求统一设置一次：
  workspace_client = Anthropic(
      default_headers={"anthropic-workspace-id": "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"},
  )
  ```

  ```typescript TypeScript
  const client = new Anthropic(); // reads ANTHROPIC_API_KEY

  // 对于多工作区密钥，每个请求都必须提供。
  // 对于单工作区密钥，请省略第二个参数。
  const message = await client.messages.create(
    {
      model: "claude-opus-5",
      max_tokens: 1024,
      messages: [{ role: "user", content: "Hello, Claude" }]
    },
    { headers: { "anthropic-workspace-id": "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ" } }
  );
  console.log(message.content);

  // 或者为此客户端发出的每个请求统一设置一次：
  const workspaceClient = new Anthropic({
    defaultHeaders: { "anthropic-workspace-id": "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ" }
  });
  ```

  ```csharp C#
  AnthropicClient client = new(); // reads ANTHROPIC_API_KEY

  MessageCreateParams parameters = new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello, Claude" }],
  };

  // 对于多工作区密钥，每个请求都必须提供。
  // 对于单工作区密钥，直接调用 client.Messages.Create(parameters)。
  var message = await client
      .WithOptions(options =>
          options with
          {
              ExtraHeaders = new Dictionary<string, string>
              {
                  ["anthropic-workspace-id"] = "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
              },
          }
      )
      .Messages.Create(parameters);
  Console.WriteLine(message);

  // 或者为此客户端发出的每个请求统一设置一次：
  AnthropicClient workspaceClient = new(new ClientOptions
  {
      ExtraHeaders = new Dictionary<string, string>
      {
          ["anthropic-workspace-id"] = "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
      },
  });
  ```

  ```go Go
  client := anthropic.NewClient() // reads ANTHROPIC_API_KEY

  // 对于多工作区密钥，每个请求都必须提供。
  // 对于单工作区密钥，请省略该选项。
  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Hello, Claude")),
  	},
  }, option.WithHeader("anthropic-workspace-id", "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"))
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(message.Content)

  // 或者为来自此客户端的每个请求统一设置一次：
  workspaceClient := anthropic.NewClient(
  	option.WithHeader("anthropic-workspace-id", "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"),
  )
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv(); // reads ANTHROPIC_API_KEY

  // 对于多工作区密钥，每个请求都必须包含此项。
  // 对于单工作区密钥，请省略 putAdditionalHeader。
  Message message = client.messages().create(MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(1024)
      .addUserMessage("Hello, Claude")
      .putAdditionalHeader("anthropic-workspace-id", "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ")
      .build());

  IO.println(message.content());

  // 或者为此客户端发出的每个请求统一设置一次：
  AnthropicClient workspaceClient = AnthropicOkHttpClient.builder()
      .fromEnv()
      .putHeader("anthropic-workspace-id", "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ")
      .build();
  ```

  ```php PHP
  $client = new Client(); // reads ANTHROPIC_API_KEY

  // 对于多工作区密钥，每个请求都必须提供。
  // 对于单工作区密钥，请省略 requestOptions。
  $message = $client->messages->create(
      model: Model::CLAUDE_OPUS_5,
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello, Claude']],
      requestOptions: [
          'extraHeaders' => ['anthropic-workspace-id' => 'wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ'],
      ],
  );

  echo json_encode($message->content), PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new # reads ANTHROPIC_API_KEY

  # 对于多工作区密钥，每个请求都必须提供。
  # 对于单工作区密钥，请省略 request_options。
  message = client.messages.create(
    model: Anthropic::Model::CLAUDE_OPUS_5,
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}],
    request_options: {extra_headers: {"anthropic-workspace-id" => "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"}}
  )

  puts message.content
  ```
</CodeGroup>

如果使用未限定到工作区的密钥发出的请求省略了该标头，API 会返回 400 `invalid_request_error`：

```json JSON
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "anthropic-workspace-id is required when authenticating with an identity-linked API key; send the id of the workspace this request acts in."
  },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

如果标头值不是有效的工作区 ID，则会返回 400 `invalid_request_error`，消息为 `anthropic-workspace-id header must be a valid workspace ID.`。如果工作区不存在，或密钥所属的用户或服务账户无权访问该工作区，API 会返回 404 `not_found_error`，消息为 ``Workspace `<id>` not found.``，与任何未知工作区的响应相同。

Workload Identity Federation 则在令牌交换时选择工作区；详情请参阅 [WIF 参考](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference)。

### 密钥过期

当您在 Claude Console 的 [API keys 页面](https://platform.claude.com/settings/keys)创建 API 密钥时，需要选择过期时间：预设值（3 小时、1 天、7 天或 30 天）、自定义时长，或者对于您存储在密钥管理器中并自行轮换的密钥选择 **Never**。如果您的组织设有最长过期时间策略，Console 会将预设值和自定义时长限制在策略允许的最大值内，并且 **Never** 不可用。现有密钥保持其当前行为；过期时间在创建时设置，之后无法更改。当您在 Claude Console 中[创建 Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)时，同样的过期选项也适用。

Anthropic 会在过期临近时向密钥创建者发送电子邮件：对于创建时有效期至少为 14 天的密钥，在过期前 7 天发送；对于有效期至少为 7 天的密钥，在过期前 1 天发送。有效期更短的密钥过期时不会发送警告邮件。

密钥过期后，使用它发出的请求会返回 `401 authentication_error`。请创建新密钥以恢复访问；已过期的密钥无法重新激活。

Console 的 API keys 表格会显示每个密钥的过期时间，Admin API 会在 [List API Keys](https://platform.claude.com/docs/zh-CN/api/admin/api_keys/list) 和 [Retrieve API Key](https://platform.claude.com/docs/zh-CN/api/admin/api_keys/retrieve) 端点上报告每个密钥的 `expires_at` 时间戳，以便您在密钥过期前进行审计和轮换。对于没有过期时间的密钥，该字段为 `null`。

过期时间限制了泄露凭证的有效期，但它不能替代良好的密钥管理习惯。无论过期时间如何，请将密钥存储在密钥管理器中，并禁用或删除任何您怀疑已泄露的密钥。

### 替换工作区 API 密钥

如果您有工作区密钥，您可能希望将其替换为 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference) 或个人密钥或服务账户密钥。这可提供更好的安全性和可观测性。

有关配置 Workload Identity Federation 的详细信息，请参阅 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference)，它比长期密钥更受推荐。

要将工作区密钥替换为个人密钥或服务账户密钥：

1. **确定密钥类型。** 您自己的工具应使用个人密钥。共享或无人值守的工作负载应使用服务账户密钥。
2. 如有必要，**创建服务账户**。您可能需要请组织管理员在 [Settings → Service accounts](https://platform.claude.com/settings/service-accounts) 中创建一个，并将其添加到相关工作区。
3. **创建新密钥。** 除非需要多个工作区，否则请专门为该集成的工作区创建密钥。
4. **部署新密钥。** 在集成读取旧密钥的所有位置替换旧密钥，通常是 `ANTHROPIC_API_KEY` 环境变量或密钥管理器条目。对于多工作区密钥，还需按照[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)中所示发送 `anthropic-workspace-id` 标头。
5. **删除旧密钥。** 确认请求成功后，在 [API keys 页面](https://platform.claude.com/settings/keys)上删除工作区密钥。

## Workload Identity Federation

"Workload Identity Federation"（工作负载身份联合），即 WIF，允许工作负载使用由您已信任的"identity provider"（身份提供商），即 IdP 颁发的短期身份令牌进行身份验证，例如 AWS IAM、Google Cloud 或任何符合标准的 OIDC 颁发者（例如 GitHub Actions、Kubernetes 服务账户、SPIFFE、Microsoft Entra ID 或 Okta）。工作负载在 `POST /v1/oauth/token` 处将其 IdP 颁发的 JWT 交换为短期 Claude API 访问令牌，SDK 会在该令牌过期前自动刷新。无需生成、分发或轮换任何 `sk-ant-api...` 字符串。

联合身份验证将长期 Claude API 密钥从您的环境中移除，从而缩小了泄露凭证的影响范围，并让您能够使用已用于云资源的相同 IdP 控制来管理访问。它本身并不能保证端到端的安全性：信任链的强度取决于您的身份提供商的配置，而上游一跳处的长期密钥（例如，可以生成 IdP 令牌的静态云凭证）仍可能破坏它。请将联合身份验证与您的提供商的控制措施结合使用，例如 IP 允许列表、MFA 和审计日志。

要配置联合身份验证，您需要在 Claude Console 中创建三个资源（一个服务账户、一个联合颁发者和一条联合规则），然后将您的 SDK 指向该规则。完整的设置流程请参阅 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)。

## App Attest

App Attest 对直接从设备调用 Claude API 的 iOS 和 macOS 应用进行身份验证。每个安装实例都使用 Apple 的 App Attest 服务证明自己是您在 Claude Console 中注册的应用的真实、未经修改的构建版本。然后，Anthropic 会向设备颁发一个短期访问令牌，其用量计入您的工作区。令牌的范围限定于您的工作区，一小时后过期，并且仅授权 [Messages API](https://platform.claude.com/docs/zh-CN/api/messages/create) 调用。

要注册您的应用并获取客户端 ID，请参阅[适用于 iOS 和 macOS 应用的 App Attest](https://platform.claude.com/docs/zh-CN/manage-claude/app-attest)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="设置 Workload Identity Federation" icon="lock" href="https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation">
    配置颁发者、规则和服务账户，然后交换令牌
  </Card>

  <Card title="身份提供商指南" icon="cloud" href="https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#identity-providers">
    适用于 AWS、Google Cloud、Azure、GitHub Actions、Kubernetes、SPIFFE 和 Okta 的分步指南
  </Card>

  <Card title="WIF 参考" icon="book" href="https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference">
    环境变量、验证规则、配置文件配置和错误参考
  </Card>

  <Card title="适用于 iOS 和 macOS 应用的 App Attest" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/manage-claude/app-attest">
    让您应用的真实安装实例无需附带 API 密钥即可调用 Claude API
  </Card>

  <Card title="客户端 SDK" icon="code" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview">
    Python、TypeScript、C#、Go、Java、PHP、Ruby 和 CLI
  </Card>
</CardGroup>
