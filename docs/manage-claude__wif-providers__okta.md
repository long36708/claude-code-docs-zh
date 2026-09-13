---
title: 将 WIF 与 Okta 配合使用
url: https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/okta
description: 通过 Workload Identity Federation 将 Okta 服务应用身份联合到 Claude API。
---

Okta 可以通过 OAuth 2.0 `client_credentials` 授权方式向**服务应用**（service application）颁发 OIDC 访问令牌，从而充当工作负载身份提供方。您的工作负载向 Okta 进行身份验证（通常使用 `private_key_jwt`，因此无需存储共享密钥），获得一个已签名的 "JSON Web Token"（JSON Web 令牌），即 JWT，然后将该 JWT 与 Anthropic 交换以获取一个短期访问令牌。

Okta 授权服务器的颁发者 URL 格式为 `https://<your-domain>.okta.com/oauth2/<auth-server-id>`。如果您使用内置的默认服务器，则路径为 `/oauth2/default`。

<Note>
  您必须使用 Okta **自定义授权服务器**（包括 `default` 服务器）。由 Okta 组织授权服务器直接颁发的令牌（即路径中不含授权服务器 ID 的 `/oauth2/v1/token` 端点）无法被外部方验证，因为 Okta 不会为其发布签名密钥。
</Note>

配置 Okta 以及向 Okta 进行身份验证的方式有很多，这些内容超出了本文档的范围。请确保您的配置和身份验证机制遵循贵公司的指导方针和安全实践。

## 前提条件

* 熟悉 [WIF 概念](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#concepts)：服务账户、联合颁发者和联合规则。
* 一个已启用 API Access Management 的 Okta 组织（自定义授权服务器需要此功能）。
* 有权在 Claude Console 中为您的 Anthropic 组织创建服务账户、联合颁发者和联合规则。
* 一个能够从 Okta 的 `/v1/token` 端点请求令牌并能访问 `api.anthropic.com` 的工作负载。

## 配置 Okta

总体而言，您需要：

1. 创建一个 Okta 服务应用。
2. 为您的默认授权服务器（或新建一个自定义授权服务器）配置受众（audience）、作用域（scope）、访问策略，以及您希望用于匹配的任何自定义声明（claim）。

具体的导航路径取决于您的 Okta 组织配置和管理控制台版本。以下编号步骤介绍了一种常见路径：

1. **创建服务应用集成。** 在 Okta Admin Console 中，创建一个类型为 **API Services**（OIDC，机器对机器）的新应用集成。记下生成的 **Client ID**。
2. **配置客户端身份验证。** 若要实现无密钥设置，请选择 **Public key / Private key**（`private_key_jwt`）并注册您工作负载的公钥 JWK。或者，如果您的环境能够安全地存储客户端密钥，也可以使用客户端密钥。对于下面的示例，您可能需要在应用上禁用 DPoP 要求；请确保您的生产环境设置符合贵组织的安全要求。
3. **设置受众。** 在您的自定义授权服务器上，将受众设置为 `https://api.anthropic.com`，以便颁发的访问令牌携带该 `aud` 声明。Anthropic 会根据此固定值验证 `aud`。
4. **授予作用域。** 在您的自定义授权服务器上，确保至少存在一个允许该服务应用请求的作用域（例如 `anthropic.access`）。Okta 会拒绝不包含已授予作用域的 `client_credentials` 请求。
5. **创建访问策略。** 在您的自定义授权服务器上，创建一个访问策略，其中至少包含一条规则，允许您的服务应用请求您在第 4 步中授予的作用域。
6. **（可选）添加自定义声明。** 如果您希望基于客户端 ID 以外的内容进行匹配，请在授权服务器的 **Claims** 选项卡中向访问令牌添加声明。

对于使用 `client_credentials` 的服务应用，Okta 会将所颁发访问令牌的 `sub` 声明设置为该应用的 **Client ID**，并将 `iss` 设置为授权服务器的颁发者 URL。

## 配置 Anthropic

在 Claude Console 中，打开 **Settings → Workload identity**，点击 **Connect workload**，然后选择 **Custom OIDC**。向导将引导您完成注册颁发者、创建服务账户以及创建联合规则的过程。

向导会为您创建这些资源。无论您是在向导中输入这些值，还是将它们发送到 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api)，都请使用以下值：

**联合颁发者：** 使用您的 Okta 自定义授权服务器 URL 和发现模式。Anthropic 会读取 Okta 的 `.well-known/openid-configuration` 发现文档，并从其公布的 `jwks_uri` 获取 JWKS。

```json
{
  "name": "okta-prod",
  "issuer_url": "https://acme.okta.com/oauth2/aus1a2b3c4d5e6f7g8h9",
  "jwks": { "type": "discovery" }
}
```

**联合规则：** 基于 Okta 的 `sub` 声明进行匹配，该声明即服务应用的 Client ID。如果您在 Okta 中定义了自定义声明，则可以改为使用 `claims` 映射或 CEL `condition` 基于这些声明进行匹配。

```json
{
  "name": "okta-pipeline",
  "issuer_id": "fdis_...",
  "match": {
    "subject_prefix": "0oa1b2c3d4e5f6g7h8i9",
    "audience": "https://api.anthropic.com"
  },
  "target": { "type": "service_account", "service_account_id": "svac_..." },
  "workspace_id": "wrkspc_...",
  "oauth_scope": "workspace:developer",
  "token_lifetime_seconds": 600
}
```

## 获取令牌并调用 Claude API

与平台原生提供方（AWS、Google Cloud、Kubernetes）不同，这些提供方会在工作负载的运行时内部提供令牌（通过投射文件或本地元数据端点），而 Okta 不会。您的工作负载必须调用 Okta 的令牌端点来获取 JWT，然后将该 JWT 作为身份令牌传递给 Anthropic SDK。

<CodeGroup>
  ```bash cURL
  # 1. 向 Okta 请求访问令牌（使用 private_key_jwt 的 client_credentials）。
  OKTA_JWT=$(curl -sS "https://acme.okta.com/oauth2/aus1a2b3c4d5e6f7g8h9/v1/token" \
    -d grant_type=client_credentials \
    -d scope=anthropic.access \
    -d client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer \
    --data-urlencode client_assertion="$SIGNED_CLIENT_ASSERTION" \
    | jq -r .access_token)

  # 2. 将 Okta JWT 交换为 Anthropic 访问令牌。
  ACCESS_TOKEN=$(curl -sS https://api.anthropic.com/v1/oauth/token \
    -H "content-type: application/json" \
    -d @- <<JSON | jq -r .access_token
  {
    "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "assertion": "$OKTA_JWT",
    "federation_rule_id": "$ANTHROPIC_FEDERATION_RULE_ID",
    "organization_id": "$ANTHROPIC_ORGANIZATION_ID",
    "service_account_id": "$ANTHROPIC_SERVICE_ACCOUNT_ID",
    "workspace_id": "$ANTHROPIC_WORKSPACE_ID"
  }
  JSON
  )

  # 3. 调用 Claude API。
  curl https://api.anthropic.com/v1/messages \
    -H "authorization: Bearer $ACCESS_TOKEN" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{"model": "claude-opus-5", "max_tokens": 1024, "messages": [{"role": "user", "content": "Hello, Claude"}]}' \
    | jq -r '.content[] | select(.type == "text") | .text'
  ```

  ```python Python
  import os
  import httpx2
  import anthropic
  from anthropic import WorkloadIdentityCredentials


  def fetch_okta_token() -> str:
      response = httpx2.post(
          f"{os.environ['OKTA_ISSUER']}/v1/token",
          data={
              "grant_type": "client_credentials",
              "scope": "anthropic.access",
              "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
              # 构建 RFC 7523 client_assertion JWT，并使用您的 Okta 应用私钥进行签名
              "client_assertion": build_signed_client_assertion(),
          },
      )
      response.raise_for_status()
      return response.json()["access_token"]


  client = anthropic.Anthropic(
      credentials=WorkloadIdentityCredentials(
          identity_token_provider=fetch_okta_token,
          federation_rule_id=os.environ["ANTHROPIC_FEDERATION_RULE_ID"],
          organization_id=os.environ["ANTHROPIC_ORGANIZATION_ID"],
          service_account_id=os.environ["ANTHROPIC_SERVICE_ACCOUNT_ID"],
          workspace_id=os.environ.get("ANTHROPIC_WORKSPACE_ID"),
      ),
  )

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello, Claude"}],
  )
  print(next(block.text for block in message.content if block.type == "text"))
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";
  import { oidcFederationProvider } from "@anthropic-ai/sdk/lib/credentials/oidc-federation";

  async function fetchOktaToken(): Promise<string> {
    const response = await fetch(`${process.env.OKTA_ISSUER}/v1/token`, {
      method: "POST",
      headers: { "content-type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        grant_type: "client_credentials",
        scope: "anthropic.access",
        client_assertion_type: "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
        // 构建 RFC 7523 client_assertion JWT，并使用您的 Okta 应用私钥进行签名
        client_assertion: buildSignedClientAssertion()
      })
    });
    const body = (await response.json()) as { access_token: string };
    return body.access_token;
  }

  const client = new Anthropic({
    credentials: oidcFederationProvider({
      identityTokenProvider: fetchOktaToken,
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
    messages: [{ role: "user", content: "Hello, Claude" }]
  });
  for (const block of message.content) {
    if (block.type === "text") {
      console.log(block.text);
    }
  }
  ```

  ```go Go
  package main

  import (
  	"context"
  	"encoding/json"
  	"fmt"
  	"net/http"
  	"net/url"
  	"os"
  	"strings"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/option"
  )

  func fetchOktaToken(ctx context.Context) (string, error) {
  	form := url.Values{
  		"grant_type":            {"client_credentials"},
  		"scope":                 {"anthropic.access"},
  		"client_assertion_type": {"urn:ietf:params:oauth:client-assertion-type:jwt-bearer"},
  		// 构建 RFC 7523 client_assertion JWT，并使用您的 Okta 应用私钥进行签名
  		"client_assertion": {buildSignedClientAssertion()},
  	}
  	req, err := http.NewRequestWithContext(ctx, http.MethodPost,
  		os.Getenv("OKTA_ISSUER")+"/v1/token", strings.NewReader(form.Encode()))
  	if err != nil {
  		return "", err
  	}
  	req.Header.Set("content-type", "application/x-www-form-urlencoded")
  	resp, err := http.DefaultClient.Do(req)
  	if err != nil {
  		return "", err
  	}
  	defer resp.Body.Close()
  	var body struct {
  		AccessToken string `json:"access_token"`
  	}
  	if err := json.NewDecoder(resp.Body).Decode(&body); err != nil {
  		return "", err
  	}
  	return body.AccessToken, nil
  }

  func main() {
  	client := anthropic.NewClient(
  		option.WithFederationTokenProvider(option.IdentityTokenFunc(fetchOktaToken), option.FederationOptions{
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
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hello, Claude")),
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
  }
  ```

  ```java Java
  IdentityTokenProvider fetchOktaToken = () -> {
      try {
          var form = Map.of(
                          "grant_type", "client_credentials",
                          "scope", "anthropic.access",
                          "client_assertion_type", "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
                          // 构建 RFC 7523 client_assertion JWT，并使用您的 Okta 应用私钥进行签名
                          "client_assertion", buildSignedClientAssertion())
                  .entrySet().stream()
                  .map(entry -> entry.getKey() + "=" + URLEncoder.encode(entry.getValue(), UTF_8))
                  .collect(Collectors.joining("&"));
          var request = HttpRequest.newBuilder(URI.create(System.getenv("OKTA_ISSUER") + "/v1/token"))
                  .header("content-type", "application/x-www-form-urlencoded")
                  .POST(HttpRequest.BodyPublishers.ofString(form))
                  .build();
          var response = HttpClient.newHttpClient().send(request, HttpResponse.BodyHandlers.ofString());
          return new ObjectMapper().readTree(response.body()).get("access_token").asText();
      } catch (Exception e) {
          throw new RuntimeException(e);
      }
  };

  AnthropicClient client = AnthropicOkHttpClient.builder()
          .federationTokenProvider(
                  fetchOktaToken,
                  System.getenv("ANTHROPIC_FEDERATION_RULE_ID"),
                  System.getenv("ANTHROPIC_ORGANIZATION_ID"),
                  System.getenv("ANTHROPIC_SERVICE_ACCOUNT_ID"))
          .build();

  var message = client.messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessage("Hello, Claude")
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
      IdentityTokenProvider = new OktaTokenProvider(),
  });
  using var client = new AnthropicOidcClient(credentials);

  var message = await client.Messages.Create(new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello, Claude" }],
  });
  foreach (var block in message.Content)
  {
      if (block.Value is TextBlock textBlock)
      {
          Console.WriteLine(textBlock.Text);
      }
  }

  class OktaTokenProvider : IIdentityTokenProvider
  {
      private static readonly HttpClient Http = new();

      public async Task<string> GetIdentityTokenAsync(CancellationToken ct = default)
      {
          var form = new FormUrlEncodedContent(new Dictionary<string, string>
          {
              ["grant_type"] = "client_credentials",
              ["scope"] = "anthropic.access",
              ["client_assertion_type"] = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
              // 构建 RFC 7523 client_assertion JWT，并使用您的 Okta 应用私钥进行签名
              ["client_assertion"] = BuildSignedClientAssertion(),
          });
          var response = await Http.PostAsync(
              $"{Environment.GetEnvironmentVariable("OKTA_ISSUER")}/v1/token", form, ct);
          response.EnsureSuccessStatusCode();
          using var json = await JsonDocument.ParseAsync(
              await response.Content.ReadAsStreamAsync(ct), default, ct);
          return json.RootElement.GetProperty("access_token").GetString()!;
      }
  }
  ```

  ```bash CLI
  # 1. 从 Okta 请求访问令牌并将其写入临时文件。
  ANTHROPIC_IDENTITY_TOKEN_FILE=$(mktemp)
  curl -sS "$OKTA_ISSUER/v1/token" \
    -d grant_type=client_credentials \
    -d scope=anthropic.access \
    -d client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer \
    --data-urlencode client_assertion="$SIGNED_CLIENT_ASSERTION" \
    | jq -r .access_token > "$ANTHROPIC_IDENTITY_TOKEN_FILE"
  export ANTHROPIC_IDENTITY_TOKEN_FILE

  # 2. 调用 Claude API。CLI 会读取 ANTHROPIC_FEDERATION_RULE_ID、
  # ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和
  # ANTHROPIC_IDENTITY_TOKEN_FILE，并执行令牌交换。
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello, Claude"}'
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Credentials\WorkloadIdentityCredentials;

  function fetchOktaToken(): string
  {
      $ch = curl_init(getenv('OKTA_ISSUER') . '/v1/token');
      curl_setopt_array($ch, [
          CURLOPT_RETURNTRANSFER => true,
          CURLOPT_POSTFIELDS => http_build_query([
              'grant_type' => 'client_credentials',
              'scope' => 'anthropic.access',
              'client_assertion_type' => 'urn:ietf:params:oauth:client-assertion-type:jwt-bearer',
              // 构建 RFC 7523 client_assertion JWT，并使用您的 Okta 应用私钥进行签名
              'client_assertion' => buildSignedClientAssertion(),
          ]),
      ]);
      $body = json_decode(curl_exec($ch), true);
      curl_close($ch);
      return $body['access_token'];
  }

  $client = new Client(
      credentials: new WorkloadIdentityCredentials(
          identityTokenProvider: fetchOktaToken(...),
          federationRuleId: getenv('ANTHROPIC_FEDERATION_RULE_ID'),
          organizationId: getenv('ANTHROPIC_ORGANIZATION_ID'),
          serviceAccountId: getenv('ANTHROPIC_SERVICE_ACCOUNT_ID'),
          workspaceId: getenv('ANTHROPIC_WORKSPACE_ID') ?: null,
      ),
  );

  $message = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello, Claude']],
  );
  echo array_find($message->content, static fn ($block): bool => $block->type === 'text')->text, PHP_EOL;
  ```

  ```ruby Ruby
  require "anthropic"
  require "json"
  require "net/http"

  def fetch_okta_token
    uri = URI("#{ENV.fetch('OKTA_ISSUER')}/v1/token")
    response = Net::HTTP.post_form(
      uri,
      "grant_type" => "client_credentials",
      "scope" => "anthropic.access",
      "client_assertion_type" => "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
      # 构建 RFC 7523 client_assertion JWT，并使用您的 Okta 应用私钥进行签名
      "client_assertion" => build_signed_client_assertion
    )
    JSON.parse(response.body).fetch("access_token")
  end

  client = Anthropic::Client.new(
    credentials: Anthropic::WorkloadIdentityCredentials.new(
      identity_token_provider: -> { fetch_okta_token },
      federation_rule_id: ENV.fetch("ANTHROPIC_FEDERATION_RULE_ID"),
      organization_id: ENV.fetch("ANTHROPIC_ORGANIZATION_ID"),
      service_account_id: ENV.fetch("ANTHROPIC_SERVICE_ACCOUNT_ID"),
      workspace_id: ENV["ANTHROPIC_WORKSPACE_ID"]
    )
  )

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}]
  )
  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

每个 SDK 选项卡都展示了可调用模式：每当 Anthropic 访问令牌即将过期时，Anthropic SDK 会再次调用您的身份令牌提供程序，因此您的 Okta 获取程序应在每次调用时返回一个新的令牌，而不是无限期地缓存某个令牌。`ant` CLI 会在每次交换时重新读取 `ANTHROPIC_IDENTITY_TOKEN_FILE`，因此对于长时间运行的 shell，请通过定时器刷新该文件。

## 验证设置

成功的交换会返回一个以 `sk-ant-oat01-` 开头的 `access_token` 以及一个以秒为单位的 `expires_in` 值。如果交换失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`），请查看[身份验证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)了解拒绝原因，并参阅[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)；Okta 端最常见的原因是 `issuer_url` 不匹配（它必须包含 `/oauth2/<auth-server-id>` 路径；Okta 组织授权服务器不可用）。

## 限定规则范围

<Warning>
  同一 Okta 授权服务器下的多个服务应用共享同一个颁发者。 省略 `subject_prefix` 的规则会匹配该服务器上的每一个服务应用， 因此任何能够注册服务应用的团队都可能获得联合的 Anthropic 令牌。
</Warning>

将规则的 `match` 块锁定到符合您用例的最小范围：

* **固定精确的 Client ID：** 将 `subject_prefix` 设置为服务应用的完整 Client ID，末尾不带 `*`。
* **固定受众：** 匹配您在授权服务器上配置的 `audience` 值，以便拒绝为其他受众签发的令牌。
* **基于自定义声明进行匹配：** 若要实现更细粒度的范围限定，请在授权服务器的 **Claims** 选项卡中添加声明，并使用规则的 `claims` 映射或 CEL `condition` 对其进行匹配。
* **每个服务应用使用一条规则：** 为每个服务应用创建单独的联合规则，而不是在多个应用之间共享一条规则。

## 后续步骤

* 查阅 [WIF 参考](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference)，了解完整的凭证解析顺序和配置文件配置。
* 参阅 [WIF 参考](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#rule-matching-semantics)，了解如何使用 CEL 表达式匹配自定义 Okta 声明。
