---
title: 在 Google Cloud 中使用 WIF
url: https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/gcp
description: 使用 Google 签名的身份令牌（而非静态 API 密钥）将 Google Cloud 工作负载（Cloud Run、Cloud Functions、App Engine、GCE、GKE）联合到 Claude API。
---

任何能够访问实例元数据服务器的 Google Cloud 计算环境（Cloud Run、Cloud Functions、App Engine、Compute Engine (GCE) 以及启用了 Workload Identity 的 GKE）都可以为其附加的服务账号请求一个由 Google 签名的 "identity token"（身份令牌）。该令牌的颁发者为 `https://accounts.google.com`，Anthropic 可以通过标准的 OIDC 发现机制直接对其进行验证，无需额外的 Google Cloud 配置。

本指南介绍如何在 Anthropic 中注册 Google 颁发者、将 Google 服务账号绑定到 Anthropic 服务账号，以及让您的工作负载将其身份令牌交换为短期有效的 Claude API 访问令牌。

## 前提条件

* 熟悉 [WIF 概念](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#concepts)：服务账号、联合颁发者和联合规则。
* 一个 Google Cloud 项目，其中有运行在 Cloud Run、Cloud Functions、App Engine、Compute Engine 或 GKE 上的工作负载。
* 附加到该工作负载的用户管理的 Google 服务账号（而非 Compute Engine 默认服务账号）。
* 在 Claude Console 中为您的 Anthropic 组织创建服务账号、联合颁发者和联合规则的权限。

## 配置 Google Cloud

Google 会自动向任何附加了服务账号的工作负载颁发身份令牌。除了附加正确的服务账号之外，Google 端无需启用任何内容，但标准计算环境与 GKE 之间的步骤略有不同。

<Tabs>
  <Tab title="Cloud Run、Cloud Functions、App Engine、GCE">
    为您的服务或实例附加一个专用服务账号：

    ```bash CLI
    gcloud run deploy my-service \
      --service-account inference-worker@my-project.iam.gserviceaccount.com
    ```

    在工作负载内部，元数据服务器会按需返回一个已签名的身份令牌。请使用您打算在 Anthropic 端注册的 `audience` 来请求它，并包含 `format=full`，以便响应中携带 `email` 声明：

    ```text wrap
    GET http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://api.anthropic.com&format=full
    Metadata-Flavor: Google
    ```

    或者，使用 gcloud CLI：

    ```bash CLI
    gcloud auth print-identity-token \
      --audiences="https://api.anthropic.com" \
      --include-email
    ```

    SDK 的等效用法请参阅[获取并使用令牌](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/gcp#acquire-and-use-the-token)。

    解码后的令牌负载如下所示：

    ```json
    {
      "iss": "https://accounts.google.com",
      "aud": "https://api.anthropic.com",
      "sub": "104892...",
      "azp": "104892...",
      "email": "inference-worker@my-project.iam.gserviceaccount.com",
      "email_verified": true,
      "exp": 1775527120
    }
    ```

    `sub` 声明是 Google 服务账号的不透明数字唯一 ID。`email` 声明是人类可读的服务账号地址。请在您的联合规则中同时匹配 `sub` 和 `email`。
  </Tab>

  <Tab title="启用 Workload Identity 的 GKE">
    在您的集群上启用 [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)，并使用 `iam.gke.io/gcp-service-account` 注解将您的 Kubernetes 服务账号绑定到 Google 服务账号：

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: inference-worker
      namespace: prod
      annotations:
        iam.gke.io/gcp-service-account: inference-worker@my-project.iam.gserviceaccount.com
    ```

    完成此绑定后，GKE 元数据服务器返回的 Google 签名令牌与 Cloud Run 和 GCE 的情况完全相同：相同的 `https://accounts.google.com` 颁发者、相同的 `email` 声明、相同的获取 URL。请完全按照下一节的说明配置 Anthropic。

    来自 GKE 的 `format=full` 令牌还额外包含 `google.compute_engine.project_id`、`google.compute_engine.zone` 和 `google.compute_engine.instance_name` 声明，您可以在联合规则的 `condition` 匹配器中引用它们（例如 `claims.google.compute_engine.project_id == "my-project"` 这样的 CEL 表达式），以将访问范围限定到特定的集群或节点池。

    <Note>
      如果您不想将 Kubernetes 服务账号绑定到 Google 服务账号，GKE Pod 也可以改用集群自身的 OIDC 颁发者（`https://container.googleapis.com/v1/projects/PROJECT/locations/REGION/clusters/CLUSTER`）配合投射的 `serviceAccountToken` 卷。该路径使用的是每个集群各自的颁发者，而非 `accounts.google.com`。有关该模式，请参阅[在 Kubernetes 中使用 WIF](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/kubernetes)。
    </Note>
  </Tab>
</Tabs>

## 配置 Anthropic

在 Claude Console 中，打开 **Settings → Workload identity**，点击 **Connect workload**，然后选择 **Google Cloud** 磁贴。向导将引导您完成注册颁发者、创建服务账号以及创建联合规则的步骤。

向导会为您创建这些资源。无论您是在向导中输入这些值，还是将它们发送到 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api)，都请使用以下值：

**联合颁发者：** Google 公开发布其 OIDC 发现文档，因此请使用发现模式。这一个颁发者即可覆盖所有 Google Cloud 平台（Cloud Run、GCE、Cloud Functions、App Engine 以及启用 Workload Identity 的 GKE）。请使用规则而非颁发者来区分工作负载。

```json
{
  "name": "gcp",
  "issuer_url": "https://accounts.google.com",
  "jwks": { "type": "discovery" }
}
```

**联合规则：** 同时匹配 `sub` 和 `email` 声明。`email` 是可读的服务账号地址；`sub` 是服务账号的数字唯一 ID，Google 永远不会重复使用它，因此固定该值可以在服务账号被删除、之后又以相同 email 创建新账号的情况下保护该规则。使用 `gcloud iam service-accounts describe SA_EMAIL --format='value(uniqueId)'` 查找唯一 ID。

```json
{
  "name": "gcp-inference-worker",
  "issuer_id": "fdis_...",
  "match": {
    "audience": "https://api.anthropic.com",
    "claims": {
      "sub": "104892101234567890123",
      "email": "inference-worker@my-project.iam.gserviceaccount.com"
    }
  },
  "target": {
    "type": "service_account",
    "service_account_id": "svac_..."
  },
  "workspace_id": "wrkspc_...",
  "oauth_scope": "workspace:developer",
  "token_lifetime_seconds": 600
}
```

## 获取并使用令牌

在您的 Google Cloud 工作负载内部，从元数据服务器获取身份令牌，在 `POST /v1/oauth/token` 处进行交换，然后使用返回的 bearer 令牌调用 Claude API。当您提供一个从元数据服务器返回新身份令牌的令牌提供程序可调用对象时，每个 Anthropic SDK 都会为您处理交换和刷新循环，如以下示例所示。

<CodeGroup>
  ```bash cURL
  # 从元数据服务器获取 Google 签名的身份令牌
  JWT=$(curl -sS -H "Metadata-Flavor: Google" \
    "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://api.anthropic.com&format=full")

  # 将其交换为 Anthropic 访问令牌
  RESPONSE=$(curl -sS https://api.anthropic.com/v1/oauth/token \
    -H "content-type: application/json" \
    --data @- <<JSON
  {
    "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "assertion": "$JWT",
    "federation_rule_id": "$ANTHROPIC_FEDERATION_RULE_ID",
    "organization_id": "$ANTHROPIC_ORGANIZATION_ID",
    "service_account_id": "$ANTHROPIC_SERVICE_ACCOUNT_ID",
    "workspace_id": "$ANTHROPIC_WORKSPACE_ID"
  }
  JSON
  )
  ACCESS_TOKEN=$(echo "$RESPONSE" | jq -r .access_token)

  # 调用 Claude API
  curl -sS https://api.anthropic.com/v1/messages \
    -H "authorization: Bearer $ACCESS_TOKEN" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello from Cloud Run"}]
    }' | jq -r '.content[] | select(.type == "text") | .text'
  ```

  ```python Python
  import os
  import anthropic
  import google.auth.transport.requests
  import google.oauth2.id_token
  from anthropic import WorkloadIdentityCredentials

  AUDIENCE = "https://api.anthropic.com"


  def fetch_google_identity_token() -> str:
      request = google.auth.transport.requests.Request()
      return google.oauth2.id_token.fetch_id_token(request, AUDIENCE)


  client = anthropic.Anthropic(
      credentials=WorkloadIdentityCredentials(
          identity_token_provider=fetch_google_identity_token,
          federation_rule_id=os.environ["ANTHROPIC_FEDERATION_RULE_ID"],
          organization_id=os.environ["ANTHROPIC_ORGANIZATION_ID"],
          service_account_id=os.environ["ANTHROPIC_SERVICE_ACCOUNT_ID"],
          workspace_id=os.environ.get("ANTHROPIC_WORKSPACE_ID"),
      ),
  )

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello from Cloud Run"}],
  )
  print(next(block.text for block in message.content if block.type == "text"))
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";
  import { oidcFederationProvider } from "@anthropic-ai/sdk/lib/credentials/oidc-federation";

  const METADATA_URL =
    "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://api.anthropic.com&format=full";

  async function fetchGoogleIdentityToken(): Promise<string> {
    const response = await fetch(METADATA_URL, {
      headers: { "Metadata-Flavor": "Google" }
    });
    return response.text();
  }

  const client = new Anthropic({
    credentials: oidcFederationProvider({
      identityTokenProvider: fetchGoogleIdentityToken,
      federationRuleId: process.env.ANTHROPIC_FEDERATION_RULE_ID!,
      organizationId: process.env.ANTHROPIC_ORGANIZATION_ID!,
      serviceAccountId: process.env.ANTHROPIC_SERVICE_ACCOUNT_ID,
      workspaceId: process.env.ANTHROPIC_WORKSPACE_ID,
      baseURL: "https://api.anthropic.com",
      fetch
    })
  });

  const message = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello from Cloud Run" }]
  });
  for (const block of message.content) {
    if (block.type === "text") {
      console.log(block.text);
    }
  }
  ```

  ```go Go
  const audience = "https://api.anthropic.com"

  googleIDToken := func(ctx context.Context) (string, error) {
  	creds, err := idtoken.NewCredentials(&idtoken.Options{Audience: audience})
  	if err != nil {
  		return "", err
  	}
  	tok, err := creds.Token(ctx)
  	if err != nil {
  		return "", err
  	}
  	return tok.Value, nil
  }

  client := anthropic.NewClient(
  	option.WithFederationTokenProvider(googleIDToken, option.FederationOptions{
  		FederationRuleID: os.Getenv("ANTHROPIC_FEDERATION_RULE_ID"),
  		OrganizationID:   os.Getenv("ANTHROPIC_ORGANIZATION_ID"),
  		ServiceAccountID: os.Getenv("ANTHROPIC_SERVICE_ACCOUNT_ID"),
  		WorkspaceID:      os.Getenv("ANTHROPIC_WORKSPACE_ID"),
  	}),
  )

  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Hello from Cloud Run")),
  	},
  })
  if err != nil {
  	panic(err)
  }
  for _, block := range message.Content {
  	if textBlock, ok := block.AsAny().(anthropic.TextBlock); ok {
  		fmt.Println(textBlock.Text)
  		break
  	}
  }
  ```

  ```java Java
  HttpClient http = HttpClient.newHttpClient();
  HttpRequest metadataRequest = HttpRequest.newBuilder()
          .uri(URI.create("http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://api.anthropic.com&format=full"))
          .header("Metadata-Flavor", "Google")
          .build();

  IdentityTokenProvider fetchGoogleIdentityToken = () -> {
      try {
          return http.send(metadataRequest, HttpResponse.BodyHandlers.ofString()).body();
      } catch (Exception e) {
          throw new RuntimeException(e);
      }
  };

  AnthropicClient client = AnthropicOkHttpClient.builder()
          .federationTokenProvider(
                  fetchGoogleIdentityToken,
                  System.getenv("ANTHROPIC_FEDERATION_RULE_ID"),
                  System.getenv("ANTHROPIC_ORGANIZATION_ID"),
                  System.getenv("ANTHROPIC_SERVICE_ACCOUNT_ID"))
          .build();

  var message = client.messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessage("Hello from Cloud Run")
          .build());

  IO.println(message.content());
  ```

  ```csharp C#
  var credentials = new WorkloadIdentityCredentials(new WorkloadIdentityOptions
  {
      FederationRuleId = Environment.GetEnvironmentVariable("ANTHROPIC_FEDERATION_RULE_ID")!,
      OrganizationId = Environment.GetEnvironmentVariable("ANTHROPIC_ORGANIZATION_ID"),
      ServiceAccountId = Environment.GetEnvironmentVariable("ANTHROPIC_SERVICE_ACCOUNT_ID"),
      WorkspaceId = Environment.GetEnvironmentVariable("ANTHROPIC_WORKSPACE_ID"),
      IdentityTokenProvider = new MetadataTokenProvider(),
  });
  using var client = new AnthropicOidcClient(credentials);

  var message = await client.Messages.Create(new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello from Cloud Run" }],
  });
  foreach (var block in message.Content)
  {
      if (block.Value is TextBlock textBlock)
      {
          Console.WriteLine(textBlock.Text);
      }
  }

  class MetadataTokenProvider : IIdentityTokenProvider
  {
      private const string METADATA_URL =
          "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://api.anthropic.com&format=full";

      private static readonly HttpClient httpClient = new()
      {
          DefaultRequestHeaders = { { "Metadata-Flavor", "Google" } },
      };

      public async Task<string> GetIdentityTokenAsync(CancellationToken ct = default)
      {
          return await httpClient.GetStringAsync(METADATA_URL, ct);
      }
  }
  ```

  ```bash CLI
  # 将 Google 签名的身份令牌写入 CLI 可读取的文件
  ANTHROPIC_IDENTITY_TOKEN_FILE=$(mktemp)
  curl -sS -H "Metadata-Flavor: Google" \
    "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://api.anthropic.com&format=full" \
    > "$ANTHROPIC_IDENTITY_TOKEN_FILE"
  export ANTHROPIC_IDENTITY_TOKEN_FILE

  # ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  # ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID 从环境变量中读取。
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello from Cloud Run"}'
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Credentials\WorkloadIdentityCredentials;

  const METADATA_URL = 'http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://api.anthropic.com&format=full';

  $context = stream_context_create([
      'http' => ['header' => "Metadata-Flavor: Google\r\n"],
  ]);

  $credentials = new WorkloadIdentityCredentials(
      identityTokenProvider: fn() => file_get_contents(METADATA_URL, false, $context),
      federationRuleId: getenv('ANTHROPIC_FEDERATION_RULE_ID'),
      organizationId: getenv('ANTHROPIC_ORGANIZATION_ID'),
      serviceAccountId: getenv('ANTHROPIC_SERVICE_ACCOUNT_ID'),
      workspaceId: getenv('ANTHROPIC_WORKSPACE_ID') ?: null,
  );
  $client = new Client(credentials: $credentials);

  $message = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello from Cloud Run']],
  );
  $textBlock = array_find($message->content, static fn ($block): bool => $block->type === 'text');
  echo $textBlock->text, PHP_EOL;
  ```

  ```ruby Ruby
  require "anthropic"
  require "net/http"

  METADATA_URL = "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://api.anthropic.com&format=full"

  credentials = Anthropic::WorkloadIdentityCredentials.new(
    identity_token_provider: -> { Net::HTTP.get(URI(METADATA_URL), {"Metadata-Flavor" => "Google"}) },
    federation_rule_id: ENV.fetch("ANTHROPIC_FEDERATION_RULE_ID"),
    organization_id: ENV.fetch("ANTHROPIC_ORGANIZATION_ID"),
    service_account_id: ENV.fetch("ANTHROPIC_SERVICE_ACCOUNT_ID"),
    workspace_id: ENV["ANTHROPIC_WORKSPACE_ID"]
  )
  client = Anthropic::Client.new(credentials: credentials)

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello from Cloud Run"}]
  )
  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

Google 身份令牌大约在一小时后过期。SDK 会在过期前自动重新调用令牌提供程序并重新进行交换。对于运行时间超过访问令牌 `expires_in` 的 shell 脚本，请按定时器刷新并重复交换。

## 验证设置

在您的工作负载内部，解码身份令牌并确认声明与您的规则匹配：

```bash cURL
curl -sS -H "Metadata-Flavor: Google" \
  "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://api.anthropic.com&format=full" \
  | jq -rR 'split(".")[1] | gsub("-";"+") | gsub("_";"/") | @base64d | fromjson'
```

检查 `iss` 是否为 `https://accounts.google.com`、`aud` 是否为 `https://api.anthropic.com`，以及 `email` 是否与您联合规则中的值匹配。然后运行上一节中的交换。成功的交换会返回一个以 `sk-ant-oat01-` 开头的 `access_token` 以及一个以秒为单位的 `expires_in` 值。如果交换失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`），请查看[身份验证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)了解拒绝原因，并参阅[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)；Google Cloud 端最常见的原因是缺少 `email` 声明（请使用 `format=full` 请求令牌以便包含该声明）。

## 限定规则范围

<Warning>
  Google 的 `sub` 声明是服务账号的不透明数字唯一 ID， 没有稳定的前缀。带有尾随 `*` 的 `subject_prefix` 会匹配 所有 Google Cloud 项目中的任意服务账号，其中任何一个 都可能获得联合的 Anthropic 令牌。
</Warning>

将规则的 `match` 块锁定到符合您用例的最小范围：

* **精确匹配 `sub`：** 在 `claims.sub` 中设置完整的数字唯一 ID，切勿对 Google 令牌使用 `subject_prefix`。
* **固定 `email` 声明：** 在 `sub` 之外添加 `claims.email`，使稳定 ID 和可读地址都必须匹配。
* **固定受众：** 将 `audience` 设置为您从元数据服务器请求的确切值，以便拒绝为其他使用方签发的令牌。
* **在 GKE 上固定项目：** 对于 `format=full` 令牌，添加诸如 `claims.google.compute_engine.project_id == "my-project"` 的 `condition`，将规则限制为某一个项目的节点。

## 后续步骤

* 阅读 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation) 页面，了解完整的资源模型和 SDK 凭据优先级。
* 为每个环境（生产、预发布）添加单独的联合规则，以便您可以撤销其中一个而不影响其他环境。
