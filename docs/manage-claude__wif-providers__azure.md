---
title: 将 WIF 与 Microsoft Entra ID 配合使用
url: https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure
description: 将 Azure 托管标识和 Entra Workload Identity 与 Claude API 联合，使您的 Azure 工作负载无需静态 API 密钥即可调用 Claude。
---

Azure 工作负载通过出示由 Microsoft Entra ID 颁发的 "JSON Web Token"（JSON Web 令牌），即 JWT，然后将其交换为短期有效的 Anthropic 访问令牌，来向 Claude API 进行身份验证。在每个 Azure 平台上，设置流程都遵循相同的结构：

1. **注册令牌受众：** 在您的 Microsoft Entra 租户中创建一个应用注册，用于代表 Claude API 受众。租户中的每个工作负载都为其请求 Entra 令牌。
2. **为您的平台设置标识：** 在 VM、VM 规模集、App Service、Functions 和 Container Apps 上使用 "managed identity"（托管标识），或在 AKS 上使用 Entra Workload Identity。
3. **配置 Anthropic：** 注册您租户的 Entra 颁发者，创建服务账户，并编写与令牌声明匹配的联合规则。
4. **在运行时交换：** 您的工作负载在 `POST /v1/oauth/token` 处将其 Entra 颁发的令牌交换为 `sk-ant-oat01-...` Anthropic 访问令牌，并使用它调用 Claude。

在这两条路径上，您向 Anthropic 出示的令牌都携带您租户特定的 Entra 颁发者，以及位于 `sub` 和 `oid` 声明中的托管标识对象 ID；不同之处仅在于工作负载获取该令牌的方式。请根据您的工作负载运行位置选择相应章节：对于 VM、VM 规模集、App Service、Functions 或 Container Apps，请参阅[使用托管标识](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#use-a-managed-identity)；对于 AKS，请参阅[在 AKS 上使用 Entra Workload Identity](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#use-entra-workload-identity-on-aks)。

## 前提条件

* 熟悉 [WIF 概念](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#concepts)：服务账户、联合颁发者和联合规则。
* 一个有权分配托管标识（或在 AKS 上配置 Entra Workload Identity）的 Azure 订阅。
* 有权在您的 Microsoft Entra 租户中创建一个应用注册和服务主体（共享的 Claude API 受众）。Entra 仅为租户中存在的受众颁发令牌，因此在任何令牌请求成功之前，必须先完成[注册令牌受众](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#register-the-token-audience)步骤。
* 您的 Microsoft Entra 租户 ID。可在 Azure 门户的 **Microsoft Entra ID → Overview → Tenant ID** 下找到。
* 有权在 Claude Console 中为您的 Anthropic 组织创建服务账户、联合颁发者和联合规则。

## 注册令牌受众

仅当所请求的受众以带有服务主体的应用注册形式存在于您的租户中时，Microsoft Entra ID 才会颁发令牌。创建一个应用注册来代表 Claude API 受众；租户中的每个工作负载都可以为其请求令牌。如果没有此注册，令牌请求将失败并返回"resource not found in tenant"错误（托管标识端点返回 `AADSTS50001`，Entra 令牌端点返回 `AADSTS500011`）。

```bash
# 创建代表 Claude API 受众（audience）的应用注册。
APP_ID=$(az ad app create --display-name claude-api-federation --query appId -o tsv)

# 请求 v2.0 访问令牌并设置 api://<APP_ID> 标识符 URI。
az ad app update --id "$APP_ID" \
  --identifier-uris "api://$APP_ID" \
  --set api.requestedAccessTokenVersion=2

# 创建服务主体，以便受众在您的租户中得到解析。
az ad sp create --id "$APP_ID"
```

<Note>
  请使用 `api://<APP_ID>` 标识符 URI 格式。Entra 将 `https://` 标识符 URI 限制为您自己租户的已验证域，因此诸如 `https://api.anthropic.com` 之类的 URI 在大多数租户中无法注册；而 `api://<APP_ID>` 在任何地方都被接受。设置 `requestedAccessTokenVersion: 2` 后，此受众的令牌为 v2.0，这也是本指南所假定的版本。如果您复用一个发出 v1.0 令牌的现有注册，请参阅[如果您的令牌是 v1.0](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#if-your-tokens-are-v1-0)。
</Note>

## 使用托管标识

当您的工作负载运行在 VM、VM 规模集、App Service、Functions 或 Container Apps 上时，请使用此路径。工作负载从平台的本地令牌端点为其分配的托管标识请求 Entra 颁发的 JWT，然后将该 JWT 与 Anthropic 进行交换。

### 配置托管标识

<Steps>
  <Step title="附加托管标识">
    在您的 Azure 资源上启用系统分配或用户分配的托管标识。在 Azure 门户中，打开该资源，转到 **Identity**，然后开启 **System assigned**（或附加一个用户分配的标识）。

    创建标识后，记下其 **Object (principal) ID**。此 GUID 在颁发的令牌中同时作为 `sub` 和 `oid` 声明出现，您的 Anthropic 联合规则将基于它进行匹配。您可以在资源的 **Identity** 页面上找到它；对于用户分配的标识，它是托管标识资源 **Overview** 页面上的 **Object (principal) ID**。（托管标识在 Microsoft Entra ID 中只有服务主体，没有应用注册。）
  </Step>

  <Step title="查找平台的令牌端点">
    附加标识后，平台会公开一个本地令牌端点：

    * **VM 和 VM 规模集：** IMDS，地址为 `http://169.254.169.254/metadata/identity/oauth2/token`，需携带请求头 `Metadata: true` 和 `api-version=2018-02-01`。
    * **App Service、Functions 和 Container Apps：** `IDENTITY_ENDPOINT` 环境变量中的 URL，需将请求头 `X-IDENTITY-HEADER` 设置为 `IDENTITY_HEADER` 的值，并使用 `api-version=2019-08-01`。在这些平台上无法访问 IMDS。

    如果资源拥有多个用户分配的托管标识，请在令牌请求中添加 `client_id=<IDENTITY_CLIENT_ID>` 以选择其中一个。Azure 建议始终指定它。如果不指定，结果取决于该资源是否同时启用了系统分配的标识：如果启用了，请求会静默回退到该标识，随后无法通过您联合规则的 `oid` 匹配；如果未启用，一旦附加了第二个用户分配的标识，请求就会直接失败。
  </Step>

  <Step title="解码示例令牌">
    从端点请求一个令牌并解码其负载，以确认您的联合规则需要匹配的声明。（有关解码命令，请参阅[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)。）托管标识的 v2.0 令牌携带以下声明：

    ```json
    {
      "iss": "https://login.microsoftonline.com/<TENANT_ID>/v2.0",
      "sub": "9f8e7d6c-1a2b-3c4d-5e6f-...",
      "aud": "<APP_ID>",
      "oid": "9f8e7d6c-1a2b-3c4d-5e6f-...",
      "tid": "<TENANT_ID>",
      "azp": "<IDENTITY_CLIENT_ID>",
      "ver": "2.0",
      "exp": 1775527120
    }
    ```

    | 声明    | 值                                                                                                                                                | 在以下情况下匹配此声明                                                                                                                                    |
    | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
    | `oid` | 托管标识的对象 ID，与 `sub` 相同                                                                                                                            | 您希望授权一个特定的托管标识。这是默认做法；[配置 Anthropic](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#configure-anthropic) 中的规则即匹配此声明。 |
    | `azp` | 调用方标识的客户端 ID                                                                                                                                     | 您希望授权共享同一应用注册的所有工作负载。对于托管标识，`azp` 对该标识是唯一的，因此它等同于 `oid`。                                                                                       |
    | `aud` | 受众应用注册的客户端 ID（来自[注册令牌受众](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#register-the-token-audience)的 `<APP_ID>` GUID） | 始终匹配。规则的 `audience` 字段必须与令牌的 `aud` 值完全相等。                                                                                                      |
    | `tid` | 您的租户 ID                                                                                                                                          | 您希望实现纵深防御。颁发者 URL 已经固定了租户。                                                                                                                     |

    如果解码后令牌的 `ver` 声明为 `1.0`，则声明名称和值会有所不同。请在继续之前参阅[如果您的令牌是 v1.0](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#if-your-tokens-are-v1-0)。
  </Step>
</Steps>

### 配置 Anthropic

在 Claude Console 中，打开 **Settings → Workload identity**，点击 **Connect workload**，然后选择 **Microsoft Entra** 磁贴。向导将引导您完成注册颁发者、创建服务账户和创建联合规则的过程。

向导会为您创建这些资源。无论您是在向导中输入这些值，还是将它们发送到 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api)，都请使用以下值：

**联合颁发者：** 在向导的 **Token issuer** 选择器中选择 **v2.0 (login.microsoftonline.com)**。（选择器默认为 v1；该默认值是为复用仍发出 v1.0 令牌的旧注册的租户而设。）Entra 在每租户颁发者 URL 处发布 OIDC 发现文档，因此请使用发现模式。您联合的每个 Microsoft Entra 租户都需要自己的颁发者记录。

```json
{
  "name": "azure-prod-tenant",
  "issuer_url": "https://login.microsoftonline.com/<TENANT_ID>/v2.0",
  "jwks": { "type": "discovery" },
  "max_jwt_lifetime_seconds": 86400
}
```

<Warning>
  托管标识工作负载需要 `max_jwt_lifetime_seconds: 86400`。Azure 颁发的托管标识令牌的 `iat` 与 `exp` 之间最长可达 24 小时，因为它会在该时间窗口内缓存每个资源的令牌，且不提供强制提前刷新的方法；而颁发者的 1 小时默认值会拒绝这些令牌，因此交换会失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`）。Connect workload 向导的 Microsoft Entra 磁贴创建颁发者时将 `max_jwt_lifetime_seconds` 设置为 `7500`，且在创建期间不提供更改该值的字段，因此请先完成向导，然后打开 **Settings → Workload identity → Issuers**，编辑该颁发者，并将该值提高到 `86400`。您也可以通过 Admin API 更新颁发者。
</Warning>

接受的生命周期越长，意味着泄露的 Entra 令牌可被交换的时间越长。如果令牌泄露，应对手段是禁用联合规则；而严格的 `oid` 匹配从一开始就限制了哪些标识可以交换令牌，如[限定规则范围](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#scope-your-rule)中所述。

**联合规则：** 基于托管标识的对象 ID 和您的租户 ID 进行匹配。对于本指南配置的 v2.0 令牌，`audience` 值是受众应用注册的客户端 ID（来自[注册令牌受众](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#register-the-token-audience)的 `<APP_ID>` GUID）。请使用解码后令牌中的确切 `aud` 值。

```json
{
  "name": "azure-inference-worker",
  "issuer_id": "fdis_...",
  "match": {
    "audience": "<APP_ID>",
    "claims": {
      "oid": "9f8e7d6c-1a2b-3c4d-5e6f-...",
      "tid": "<TENANT_ID>"
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

`token_lifetime_seconds` 是交换返回的 Anthropic 访问令牌的生命周期，而非 Entra 令牌的生命周期；SDK 会为您刷新它。

### 获取并使用令牌

在运行时，您的工作负载获取其 Entra 令牌，在 `POST /v1/oauth/token` 处进行交换，并使用返回的持有者令牌调用 Claude。当您提供一个令牌提供程序可调用对象时，每个 Anthropic SDK 都会处理交换和刷新循环，如以下示例所示。cURL 选项卡展示了原始流程。

这些示例从平台的令牌端点获取托管标识令牌：在 VM 和 VM 规模集上为 IMDS，在 App Service、Functions 和 Container Apps 上为 `IDENTITY_ENDPOINT` 服务。请将 `api://<APP_ID>` 资源值中的 `<APP_ID>` 替换为来自[注册令牌受众](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#register-the-token-audience)的受众应用注册的客户端 ID。

<Tip>
  如果您的工作负载已经使用 Azure Identity 客户端库，请将其令牌获取方式（使用作用域 `api://<APP_ID>/.default` 的 `DefaultAzureCredential`）作为标识令牌提供程序传入，而不是直接调用令牌端点。该库会在每个 Azure 平台上选择正确的端点，包括使用 Entra Workload Identity 的 AKS。
</Tip>

<CodeGroup>
  ```bash cURL
  # 1. 获取 Entra 颁发的令牌（托管标识）。
  #    在 VM 或 VM 规模集上，使用 IMDS。若有多个用户分配的
  #    标识，请追加 &client_id=<IDENTITY_CLIENT_ID>。
  ENTRA_TOKEN=$(curl -sS -H "Metadata: true" \
    "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=api://<APP_ID>" \
    | jq -r .access_token)

  #    在 App Service、Functions 或 Container Apps 上，请改用本地令牌
  #    服务（在这些环境中无法访问 IMDS）：
  # ENTRA_TOKEN=$(curl -sS -H "X-IDENTITY-HEADER: $IDENTITY_HEADER" \
  #   "$IDENTITY_ENDPOINT?api-version=2019-08-01&resource=api://<APP_ID>" \
  #   | jq -r .access_token)

  #    对于使用 Entra Workload Identity 的 AKS，请改用
  #    “在 AKS 上使用 Entra Workload Identity”一节中的两跳交换流程。

  # 2. 将其交换为 Anthropic 访问令牌。
  RESPONSE=$(curl -sS https://api.anthropic.com/v1/oauth/token \
    -H "content-type: application/json" \
    -d @- <<JSON
  {
    "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "assertion": "$ENTRA_TOKEN",
    "federation_rule_id": "$ANTHROPIC_FEDERATION_RULE_ID",
    "organization_id": "$ANTHROPIC_ORGANIZATION_ID",
    "service_account_id": "$ANTHROPIC_SERVICE_ACCOUNT_ID",
    "workspace_id": "$ANTHROPIC_WORKSPACE_ID"
  }
  JSON
  )

  ACCESS_TOKEN=$(echo "$RESPONSE" | jq -r .access_token)

  # 3. 使用 bearer 令牌调用 Claude API。
  curl https://api.anthropic.com/v1/messages \
    -H "authorization: Bearer $ACCESS_TOKEN" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello from Azure"}]
    }' | jq -r '.content[] | select(.type == "text") | .text'
  ```

  ```python Python
  import os

  import anthropic
  import requests
  from anthropic import WorkloadIdentityCredentials

  # 受众应用注册的标识符 URI（请参阅"注册令牌受众"）。
  AUDIENCE = "api://<APP_ID>"


  def fetch_entra_token() -> str:
      """Fetch a managed identity token from the platform's token endpoint."""
      # 若有多个用户分配的标识，请在请求参数中添加 client_id=<IDENTITY_CLIENT_ID>
      # 以选择其中一个。
      if endpoint := os.environ.get("IDENTITY_ENDPOINT"):
          # App Service、Functions、Container Apps
          response = requests.get(
              endpoint,
              headers={"X-IDENTITY-HEADER": os.environ["IDENTITY_HEADER"]},
              params={"api-version": "2019-08-01", "resource": AUDIENCE},
              timeout=5,
          )
      else:
          # VM 或 VM 规模集：Azure 实例元数据服务 (IMDS)
          response = requests.get(
              "http://169.254.169.254/metadata/identity/oauth2/token",
              headers={"Metadata": "true"},
              params={"api-version": "2018-02-01", "resource": AUDIENCE},
              timeout=5,
          )
      response.raise_for_status()
      return response.json()["access_token"]


  client = anthropic.Anthropic(
      credentials=WorkloadIdentityCredentials(
          identity_token_provider=fetch_entra_token,
          federation_rule_id=os.environ["ANTHROPIC_FEDERATION_RULE_ID"],
          organization_id=os.environ["ANTHROPIC_ORGANIZATION_ID"],
          service_account_id=os.environ["ANTHROPIC_SERVICE_ACCOUNT_ID"],
          workspace_id=os.environ.get("ANTHROPIC_WORKSPACE_ID"),
      ),
  )

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello from Azure"}],
  )
  print(next(block.text for block in message.content if block.type == "text"))
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";
  import { oidcFederationProvider } from "@anthropic-ai/sdk/lib/credentials/oidc-federation";

  // 受众应用注册的标识符 URI（请参阅“注册令牌受众”）。
  const AUDIENCE = "api://<APP_ID>";

  async function fetchEntraToken(): Promise<string> {
    // App Service、Functions 和 Container Apps 会注入 IDENTITY_ENDPOINT；
    // VM 和 VM 规模集使用 IMDS。
    // 若有多个用户分配的标识，请追加 &client_id=<IDENTITY_CLIENT_ID>。
    const identityEndpoint = process.env.IDENTITY_ENDPOINT;
    const url = identityEndpoint
      ? `${identityEndpoint}?api-version=2019-08-01&resource=${AUDIENCE}`
      : `http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=${AUDIENCE}`;
    const headers: Record<string, string> = identityEndpoint
      ? { "X-IDENTITY-HEADER": process.env.IDENTITY_HEADER! }
      : { Metadata: "true" };
    const response = await fetch(url, { headers });
    const body = (await response.json()) as { access_token: string };
    return body.access_token;
  }

  const client = new Anthropic({
    credentials: oidcFederationProvider({
      identityTokenProvider: fetchEntraToken,
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
    messages: [{ role: "user", content: "Hello from Azure" }]
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
  	"os"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/option"
  )

  // 受众应用注册的标识符 URI（请参阅“注册令牌受众”）。
  const audience = "api://<APP_ID>"

  // fetchEntraToken 从平台的令牌端点获取托管标识令牌：
  // 在 VM 和 VM 规模集上为 IMDS，在 App Service、Functions
  // 和 Container Apps 上为 IDENTITY_ENDPOINT 服务。
  func fetchEntraToken(ctx context.Context) (string, error) {
  	// 若有多个用户分配的标识，请追加 &client_id=<IDENTITY_CLIENT_ID>。
  	tokenURL := "http://169.254.169.254/metadata/identity/oauth2/token" +
  		"?api-version=2018-02-01&resource=" + audience
  	header, value := "Metadata", "true"
  	if endpoint := os.Getenv("IDENTITY_ENDPOINT"); endpoint != "" {
  		tokenURL = endpoint + "?api-version=2019-08-01&resource=" + audience
  		header, value = "X-IDENTITY-HEADER", os.Getenv("IDENTITY_HEADER")
  	}
  	req, err := http.NewRequestWithContext(ctx, http.MethodGet, tokenURL, nil)
  	if err != nil {
  		return "", err
  	}
  	req.Header.Set(header, value)
  	resp, err := http.DefaultClient.Do(req)
  	if err != nil {
  		return "", fmt.Errorf("call token endpoint: %w", err)
  	}
  	defer resp.Body.Close()
  	var body struct {
  		AccessToken string `json:"access_token"`
  	}
  	if err := json.NewDecoder(resp.Body).Decode(&body); err != nil {
  		return "", fmt.Errorf("decode token response: %w", err)
  	}
  	return body.AccessToken, nil
  }

  func main() {
  	client := anthropic.NewClient(
  		option.WithFederationTokenProvider(fetchEntraToken, option.FederationOptions{
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
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hello from Azure")),
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
  HttpClient http = HttpClient.newHttpClient();
  // 受众应用注册的标识符 URI（请参阅“注册令牌受众”）。
  String audience = "api://<APP_ID>";
  // App Service、Functions 和 Container Apps 会注入 IDENTITY_ENDPOINT；
  // VM 和 VM Scale Sets 使用 IMDS。
  // 若有多个用户分配的标识，请追加 &client_id=<IDENTITY_CLIENT_ID>。
  String identityEndpoint = System.getenv("IDENTITY_ENDPOINT");
  HttpRequest tokenRequest = identityEndpoint != null
          ? HttpRequest.newBuilder(URI.create(identityEndpoint + "?api-version=2019-08-01&resource=" + audience))
                  .header("X-IDENTITY-HEADER", System.getenv("IDENTITY_HEADER"))
                  .build()
          : HttpRequest.newBuilder(URI.create("http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=" + audience))
                  .header("Metadata", "true")
                  .build();

  IdentityTokenProvider fetchEntraToken = () -> {
      try {
          var response = http.send(tokenRequest, HttpResponse.BodyHandlers.ofString());
          return new ObjectMapper().readTree(response.body()).get("access_token").asText();
      } catch (Exception e) {
          throw new RuntimeException(e);
      }
  };

  AnthropicClient client = AnthropicOkHttpClient.builder()
          .federationTokenProvider(
                  fetchEntraToken,
                  System.getenv("ANTHROPIC_FEDERATION_RULE_ID"),
                  System.getenv("ANTHROPIC_ORGANIZATION_ID"),
                  System.getenv("ANTHROPIC_SERVICE_ACCOUNT_ID"),
                  System.getenv("ANTHROPIC_WORKSPACE_ID"))
          .build();

  var message = client.messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessage("Hello from Azure")
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
      IdentityTokenProvider = new EntraTokenProvider(),
  });
  using var client = new AnthropicOidcClient(credentials);

  var message = await client.Messages.Create(new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello from Azure" }],
  });
  foreach (var block in message.Content)
  {
      if (block.Value is TextBlock textBlock)
      {
          Console.WriteLine(textBlock.Text);
      }
  }

  class EntraTokenProvider : IIdentityTokenProvider
  {
      // 受众应用注册的标识符 URI（请参阅“注册令牌受众”）。
      private const string Audience = "api://<APP_ID>";

      private static readonly HttpClient httpClient = new();

      public async Task<string> GetIdentityTokenAsync(CancellationToken ct = default)
      {
          // App Service、Functions 和 Container Apps 会注入 IDENTITY_ENDPOINT；
          // VM 和 VM Scale Sets 使用 IMDS。
          // 若有多个用户分配的标识，请追加 &client_id=<IDENTITY_CLIENT_ID>。
          var identityEndpoint = Environment.GetEnvironmentVariable("IDENTITY_ENDPOINT");
          using var request = identityEndpoint is not null
              ? new HttpRequestMessage(HttpMethod.Get,
                  $"{identityEndpoint}?api-version=2019-08-01&resource={Audience}")
              {
                  Headers = { { "X-IDENTITY-HEADER", Environment.GetEnvironmentVariable("IDENTITY_HEADER") } },
              }
              : new HttpRequestMessage(HttpMethod.Get,
                  $"http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource={Audience}")
              {
                  Headers = { { "Metadata", "true" } },
              };
          using var response = await httpClient.SendAsync(request, ct);
          response.EnsureSuccessStatusCode();
          using var json = await JsonDocument.ParseAsync(
              await response.Content.ReadAsStreamAsync(ct), default, ct);
          return json.RootElement.GetProperty("access_token").GetString()!;
      }
  }
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Credentials\WorkloadIdentityCredentials;

  // 受众应用注册的标识符 URI（请参阅“注册令牌受众”）。
  const AUDIENCE = 'api://<APP_ID>';

  function fetchEntraToken(): string
  {
      // App Service、Functions 和 Container Apps 会注入 IDENTITY_ENDPOINT；
      // VM 和 VM Scale Sets 使用 IMDS。
      // 若有多个用户分配的标识，请追加 &client_id=<IDENTITY_CLIENT_ID>。
      $identityEndpoint = getenv('IDENTITY_ENDPOINT');
      if ($identityEndpoint !== false) {
          $url = $identityEndpoint . '?api-version=2019-08-01&resource=' . AUDIENCE;
          $header = 'X-IDENTITY-HEADER: ' . getenv('IDENTITY_HEADER');
      } else {
          $url = 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=' . AUDIENCE;
          $header = 'Metadata: true';
      }
      $context = stream_context_create([
          'http' => ['header' => $header . "\r\n"],
      ]);
      $body = json_decode(file_get_contents($url, false, $context), true);
      return $body['access_token'];
  }

  $credentials = new WorkloadIdentityCredentials(
      identityTokenProvider: fetchEntraToken(...),
      federationRuleId: getenv('ANTHROPIC_FEDERATION_RULE_ID'),
      organizationId: getenv('ANTHROPIC_ORGANIZATION_ID'),
      serviceAccountId: getenv('ANTHROPIC_SERVICE_ACCOUNT_ID'),
      workspaceId: getenv('ANTHROPIC_WORKSPACE_ID') ?: null,
  );
  $client = new Client(credentials: $credentials);

  $message = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello from Azure']],
  );
  $textBlock = array_find($message->content, static fn ($block): bool => $block->type === 'text');
  echo $textBlock->text, PHP_EOL;
  ```

  ```ruby Ruby
  require "anthropic"
  require "json"
  require "net/http"

  # 受众应用注册的标识符 URI（请参阅“注册令牌受众”）。
  AUDIENCE = "api://<APP_ID>"

  def fetch_entra_token
    # App Service、Functions 和 Container Apps 会注入 IDENTITY_ENDPOINT；
    # VM 和 VM 规模集使用 IMDS。
    # 若有多个用户分配的标识，请追加 &client_id=<IDENTITY_CLIENT_ID>。
    if (endpoint = ENV["IDENTITY_ENDPOINT"])
      url = "#{endpoint}?api-version=2019-08-01&resource=#{AUDIENCE}"
      headers = {"X-IDENTITY-HEADER" => ENV.fetch("IDENTITY_HEADER")}
    else
      url = "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=#{AUDIENCE}"
      headers = {"Metadata" => "true"}
    end
    response = Net::HTTP.get(URI(url), headers)
    JSON.parse(response).fetch("access_token")
  end

  credentials = Anthropic::WorkloadIdentityCredentials.new(
    identity_token_provider: -> { fetch_entra_token },
    federation_rule_id: ENV.fetch("ANTHROPIC_FEDERATION_RULE_ID"),
    organization_id: ENV.fetch("ANTHROPIC_ORGANIZATION_ID"),
    service_account_id: ENV.fetch("ANTHROPIC_SERVICE_ACCOUNT_ID"),
    workspace_id: ENV["ANTHROPIC_WORKSPACE_ID"]
  )
  client = Anthropic::Client.new(credentials: credentials)

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello from Azure"}]
  )
  puts message.content.find { it.type == :text }.text
  ```

  ```bash CLI
  # 将 Entra 颁发的访问令牌写入 CLI 可读取的文件。
  # 此处展示的是 VM 或 VM 规模集（IMDS）的情况。在 App Service、Functions 或
  # Container Apps 上，请改为从 "$IDENTITY_ENDPOINT?api-version=2019-08-01&resource=api://<APP_ID>" 获取，
  # 并附带 -H "X-IDENTITY-HEADER: $IDENTITY_HEADER"。
  # 若有多个用户分配的标识，请追加 &client_id=<IDENTITY_CLIENT_ID>。
  ANTHROPIC_IDENTITY_TOKEN_FILE=$(mktemp)
  trap 'rm -f "$ANTHROPIC_IDENTITY_TOKEN_FILE"' EXIT
  curl -sS -H "Metadata: true" \
    "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=api://<APP_ID>" \
    | jq -r .access_token > "$ANTHROPIC_IDENTITY_TOKEN_FILE"
  export ANTHROPIC_IDENTITY_TOKEN_FILE

  # ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  # ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID 从环境变量中读取。
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello from Azure"}'
  ```
</CodeGroup>

### 验证设置

从您的 Azure 资源运行[获取并使用令牌](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#acquire-and-use-the-token)中所示的 cURL 交换，并确认 `POST /v1/oauth/token` 返回 `200`，其中 `access_token` 以 `sk-ant-oat01-` 开头，且 `expires_in` 值以秒为单位。如果交换失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`），请查看[身份验证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)了解拒绝原因，然后解码 Entra 令牌（有关命令，请参阅[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)），并检查最常见的 Azure 端原因：

* **颁发者不匹配：** 注册的 `issuer_url` 必须与令牌的 `iss` 声明完全匹配。v2.0 令牌携带 `https://login.microsoftonline.com/<TENANT_ID>/v2.0`；如果解码后的 `ver` 声明为 `1.0`，请参阅[如果您的令牌是 v1.0](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#if-your-tokens-are-v1-0)。
* **令牌生命周期：** 托管标识令牌的 `iat` 与 `exp` 之间最长可达 24 小时。如果颁发者仍使用向导的 `7500`（或 1 小时默认值），请按照[配置 Anthropic](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#configure-anthropic) 中所述将 `max_jwt_lifetime_seconds` 提高到 `86400`。
* **受众不匹配：** 规则的 `audience` 必须与令牌的 `aud` 完全相等：对于本指南配置的 v2.0 令牌，即受众应用注册的客户端 ID。
* **声明名称不匹配：** 基于令牌未携带的声明进行匹配的规则永远不会通过。v1.0 令牌在 `appid` 而非 `azp` 中携带客户端 ID；请参阅[如果您的令牌是 v1.0](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#if-your-tokens-are-v1-0)。

## 在 AKS 上使用 Entra Workload Identity

当您的工作负载运行在 AKS pod 中时，请使用此路径。Entra Workload Identity 将 Kubernetes 服务账户与用户分配的托管标识联合：Kubernetes 将一个服务账户令牌（由 AKS 集群的 OIDC 颁发者签名）投射到 pod 中 `AZURE_FEDERATED_TOKEN_FILE` 所指的路径。该投射令牌并非 Entra 颁发的令牌，因此为了保持在本页所述的 Entra 中介路径上，工作负载需执行两跳交换：首先在 `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token`（联合 `client_credentials` 授权）处将投射令牌兑换为 Entra 颁发的访问令牌，然后将该 Entra 令牌作为标识令牌传递给 Anthropic SDK。

<Tip>
  AKS pod 也可以跳过 Entra 交换，直接向 Anthropic 出示 Kubernetes 投射的服务账户令牌。该路径向 Anthropic 注册的是您 AKS 集群的 OIDC 颁发者，而非您的 Entra 租户。有关该流程，请参阅[将 WIF 与 Kubernetes 配合使用](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/kubernetes)。
</Tip>

### 配置 Entra Workload Identity

<Steps>
  <Step title="在集群上启用 OIDC 颁发者和工作负载标识">
    启用工作负载标识会为您安装 `azure-workload-identity` 变更 webhook；仅在非 AKS 集群上才需手动部署它。记录集群的 OIDC 颁发者 URL，以供后续步骤中创建联合凭据时使用。

    ```bash
    az aks update \
      --resource-group <RESOURCE_GROUP> \
      --name <CLUSTER_NAME> \
      --enable-oidc-issuer \
      --enable-workload-identity

    AKS_OIDC_ISSUER=$(az aks show \
      --resource-group <RESOURCE_GROUP> \
      --name <CLUSTER_NAME> \
      --query oidcIssuerProfile.issuerUrl -o tsv)
    ```
  </Step>

  <Step title="创建用户分配的托管标识">
    从该标识中记录两个值：**Client ID** 用于服务账户注解（并作为 `AZURE_CLIENT_ID` 注入到 pod 中），**Object (principal) ID** 则作为您的 Anthropic 联合规则所匹配的 `oid` 声明出现。

    ```bash
    az identity create \
      --resource-group <RESOURCE_GROUP> \
      --name claude-inference-identity \
      --location <LOCATION>

    # 填入服务账号注解中；以 AZURE_CLIENT_ID 的形式注入到 pod。
    IDENTITY_CLIENT_ID=$(az identity show \
      --resource-group <RESOURCE_GROUP> \
      --name claude-inference-identity \
      --query clientId -o tsv)

    # 作为 oid 声明出现，供您的联合规则进行匹配。
    IDENTITY_OBJECT_ID=$(az identity show \
      --resource-group <RESOURCE_GROUP> \
      --name claude-inference-identity \
      --query principalId -o tsv)
    ```
  </Step>

  <Step title="创建带注解的 Kubernetes 服务账户">
    `azure-workload-identity` webhook 读取 `azure.workload.identity/client-id` 注解以将 `AZURE_CLIENT_ID` 注入到 pod 中，[获取并使用令牌](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#acquire-and-use-the-token-2)中的示例会从环境中读取该值。

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: claude-inference
      namespace: inference
      annotations:
        azure.workload.identity/client-id: <IDENTITY_CLIENT_ID>
    ```
  </Step>

  <Step title="在托管标识上创建联合凭据">
    联合凭据针对该特定服务账户信任您集群的 OIDC 颁发者。`--audience api://AzureADTokenExchange` 值是 Entra 为传入的 Kubernetes 服务账户令牌设定的固定受众；它与您之前注册的 Claude API 受众无关。

    ```bash
    az identity federated-credential create \
      --resource-group <RESOURCE_GROUP> \
      --identity-name claude-inference-identity \
      --name claude-inference-aks \
      --issuer "$AKS_OIDC_ISSUER" \
      --subject system:serviceaccount:inference:claude-inference \
      --audience api://AzureADTokenExchange
    ```
  </Step>

  <Step title="为 pod 添加标签并设置其服务账户">
    pod 必须携带 `azure.workload.identity/use: "true"` 标签，并以带注解的服务账户身份运行。随后 webhook 会将 `AZURE_FEDERATED_TOKEN_FILE`、`AZURE_CLIENT_ID` 和 `AZURE_TENANT_ID` 注入到 pod 中。`AZURE_FEDERATED_TOKEN_FILE` 处的文件包含由 AKS 集群的 OIDC 颁发者签名的 Kubernetes 投射服务账户令牌。

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: inference-worker
      namespace: inference
      labels:
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: claude-inference
      containers:
        - name: app
          image: your-registry/inference-worker:latest
    ```
  </Step>

  <Step title="解码示例令牌">
    您的 Anthropic 联合规则所看到的令牌并非投射文件；而是 `client_credentials` 交换返回的 Entra 颁发的令牌。在带标签的 pod 内部，运行[获取并使用令牌](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#acquire-and-use-the-token-2)中 cURL 示例的第 1 步并解码结果。它携带与托管标识路径相同的声明结构：

    ```json
    {
      "iss": "https://login.microsoftonline.com/<TENANT_ID>/v2.0",
      "sub": "9f8e7d6c-1a2b-3c4d-5e6f-...",
      "aud": "<APP_ID>",
      "oid": "9f8e7d6c-1a2b-3c4d-5e6f-...",
      "tid": "<TENANT_ID>",
      "azp": "<IDENTITY_CLIENT_ID>",
      "ver": "2.0",
      "exp": 1775527120
    }
    ```

    `sub` 和 `oid` 是托管标识的对象 ID，`aud` 是受众应用注册的客户端 ID，`azp` 是托管标识的客户端 ID（即 `AZURE_CLIENT_ID` 的值）。生命周期与托管标识路径不同：`client_credentials` 令牌的 `iat` 与 `exp` 之间默认为随机的 60 到 90 分钟窗口，而非 24 小时。
  </Step>
</Steps>

### 配置 Anthropic

在 Claude Console 中，打开 **Settings → Workload identity**，点击 **Connect workload**，然后选择 **Microsoft Entra** 磁贴。向导将引导您完成注册颁发者、创建服务账户和创建联合规则的过程。

向导会为您创建这些资源。无论您是在向导中输入这些值，还是将它们发送到 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api)，都请使用以下值：

**联合颁发者：** 在向导的 **Token issuer** 选择器中选择 **v2.0 (login.microsoftonline.com)**。（选择器默认为 v1；该默认值是为复用仍发出 v1.0 令牌的旧注册的租户而设。）Entra 在每租户颁发者 URL 处发布 OIDC 发现文档，因此请使用发现模式。您联合的每个 Microsoft Entra 租户都需要自己的颁发者记录。

```json
{
  "name": "azure-prod-tenant",
  "issuer_url": "https://login.microsoftonline.com/<TENANT_ID>/v2.0",
  "jwks": { "type": "discovery" },
  "max_jwt_lifetime_seconds": 7500
}
```

<Warning>
  Connect workload 向导的 Microsoft Entra 磁贴创建颁发者时将 `max_jwt_lifetime_seconds` 设置为 `7500`（略多于 2 小时），这涵盖了 `client_credentials` 令牌默认的 60 到 90 分钟生命周期。租户令牌生命周期策略或 "Continuous Access Evaluation"（持续访问评估），即 CAE，可能会延长该生命周期。如果您解码后令牌的 `exp` 减去 `iat` 超过 7500 秒，请在 **Settings → Workload identity → Issuers** 中编辑颁发者并相应提高 `max_jwt_lifetime_seconds`，否则交换会失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`）。如果您的租户还运行来自[使用托管标识](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#use-a-managed-identity)的托管标识工作负载，请使用该章节的 `86400` 值，它同时涵盖两条路径。
</Warning>

接受的生命周期越长，意味着泄露的 Entra 令牌可被交换的时间越长。如果令牌泄露，应对手段是禁用联合规则；而严格的 `oid` 匹配从一开始就限制了哪些标识可以交换令牌，如[限定规则范围](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#scope-your-rule)中所述。

**联合规则：** 基于托管标识的对象 ID 和您的租户 ID 进行匹配。对于本指南配置的 v2.0 令牌，`audience` 值是受众应用注册的客户端 ID（来自[注册令牌受众](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#register-the-token-audience)的 `<APP_ID>` GUID）。请使用解码后令牌中的确切 `aud` 值。

```json
{
  "name": "azure-inference-worker",
  "issuer_id": "fdis_...",
  "match": {
    "audience": "<APP_ID>",
    "claims": {
      "oid": "9f8e7d6c-1a2b-3c4d-5e6f-...",
      "tid": "<TENANT_ID>"
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

`token_lifetime_seconds` 是交换返回的 Anthropic 访问令牌的生命周期，而非 Entra 令牌的生命周期；SDK 会为您刷新它。

### 获取并使用令牌

在运行时，pod 执行两跳交换：它将 Kubernetes 投射的令牌（`AZURE_FEDERATED_TOKEN_FILE` 处的文件）作为联合 `client_credentials` 断言发送到 Entra 的令牌端点，然后在 `POST /v1/oauth/token` 处交换所得的 Entra 访问令牌。当您将 Entra 获取操作作为令牌提供程序可调用对象提供时，每个 Anthropic SDK 都会处理第二次交换和刷新循环，如以下示例所示。cURL 选项卡展示了原始流程。

示例中出现了两个不同的客户端 ID。`<APP_ID>` 是来自[注册令牌受众](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#register-the-token-audience)的受众应用注册的客户端 ID；作用域 `api://<APP_ID>/.default` 向 Entra 请求一个面向该受众的令牌。`$AZURE_CLIENT_ID` 是托管标识的客户端 ID，由 webhook 注入，用于标识调用方。请勿将二者互相替换。

<Tip>
  如果您的工作负载已经使用 Azure Identity 客户端库，请将其令牌获取方式（使用作用域 `api://<APP_ID>/.default` 的 `DefaultAzureCredential`）作为标识令牌提供程序传入，而不是自行执行两跳交换。该库读取相同的 `AZURE_FEDERATED_TOKEN_FILE`、`AZURE_CLIENT_ID` 和 `AZURE_TENANT_ID` 环境变量，并处理 Entra 交换。
</Tip>

<CodeGroup>
  ```bash cURL
  # 1. 将 Kubernetes 投射的令牌（位于 $AZURE_FEDERATED_TOKEN_FILE）
  #    交换为 Entra 签发的 JWT。
  ENTRA_JWT=$(curl -sS "https://login.microsoftonline.com/$AZURE_TENANT_ID/oauth2/v2.0/token" \
    -d grant_type=client_credentials \
    -d "client_id=$AZURE_CLIENT_ID" \
    --data-urlencode "scope=api://<APP_ID>/.default" \
    -d client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer \
    --data-urlencode "client_assertion@$AZURE_FEDERATED_TOKEN_FILE" \
    | jq -r .access_token)

  # 2. 将 Entra JWT 交换为 Anthropic 访问令牌。
  ACCESS_TOKEN=$(curl -sS https://api.anthropic.com/v1/oauth/token \
    -H "content-type: application/json" \
    -d @- <<JSON | jq -r .access_token
  {
    "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "assertion": "$ENTRA_JWT",
    "federation_rule_id": "$ANTHROPIC_FEDERATION_RULE_ID",
    "organization_id": "$ANTHROPIC_ORGANIZATION_ID",
    "service_account_id": "$ANTHROPIC_SERVICE_ACCOUNT_ID",
    "workspace_id": "$ANTHROPIC_WORKSPACE_ID"
  }
  JSON
  )

  # 3. 调用 Claude API。
  curl -sS https://api.anthropic.com/v1/messages \
    -H "authorization: Bearer $ACCESS_TOKEN" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello from Azure"}]
    }' | jq -r '.content[] | select(.type == "text") | .text'
  ```

  ```python Python
  import os
  from pathlib import Path

  import anthropic
  import requests
  from anthropic import WorkloadIdentityCredentials


  def fetch_entra_token_via_federation() -> str:
      federated_token = Path(os.environ["AZURE_FEDERATED_TOKEN_FILE"]).read_text()
      response = requests.post(
          f"https://login.microsoftonline.com/{os.environ['AZURE_TENANT_ID']}/oauth2/v2.0/token",
          data={
              "client_id": os.environ["AZURE_CLIENT_ID"],
              "grant_type": "client_credentials",
              "scope": "api://<APP_ID>/.default",
              "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
              "client_assertion": federated_token,
          },
          timeout=5,
      )
      response.raise_for_status()
      return response.json()["access_token"]


  client = anthropic.Anthropic(
      credentials=WorkloadIdentityCredentials(
          identity_token_provider=fetch_entra_token_via_federation,
          federation_rule_id=os.environ["ANTHROPIC_FEDERATION_RULE_ID"],
          organization_id=os.environ["ANTHROPIC_ORGANIZATION_ID"],
          service_account_id=os.environ["ANTHROPIC_SERVICE_ACCOUNT_ID"],
          workspace_id=os.environ.get("ANTHROPIC_WORKSPACE_ID"),
      ),
  )

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello from Azure"}],
  )
  print(next(block.text for block in message.content if block.type == "text"))
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";
  import { oidcFederationProvider } from "@anthropic-ai/sdk/lib/credentials/oidc-federation";
  import { readFile } from "node:fs/promises";

  async function fetchEntraTokenViaFederation(): Promise<string> {
    const federatedToken = await readFile(process.env.AZURE_FEDERATED_TOKEN_FILE!, "utf8");
    const response = await fetch(
      `https://login.microsoftonline.com/${process.env.AZURE_TENANT_ID}/oauth2/v2.0/token`,
      {
        method: "POST",
        headers: { "content-type": "application/x-www-form-urlencoded" },
        body: new URLSearchParams({
          client_id: process.env.AZURE_CLIENT_ID!,
          grant_type: "client_credentials",
          scope: "api://<APP_ID>/.default",
          client_assertion_type: "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
          client_assertion: federatedToken
        })
      }
    );
    const body = (await response.json()) as { access_token: string };
    return body.access_token;
  }

  const client = new Anthropic({
    credentials: oidcFederationProvider({
      identityTokenProvider: fetchEntraTokenViaFederation,
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
    messages: [{ role: "user", content: "Hello from Azure" }]
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

  func fetchEntraTokenViaFederation(ctx context.Context) (string, error) {
  	federatedToken, err := os.ReadFile(os.Getenv("AZURE_FEDERATED_TOKEN_FILE"))
  	if err != nil {
  		return "", err
  	}
  	form := url.Values{
  		"client_id":             {os.Getenv("AZURE_CLIENT_ID")},
  		"grant_type":            {"client_credentials"},
  		"scope":                 {"api://<APP_ID>/.default"},
  		"client_assertion_type": {"urn:ietf:params:oauth:client-assertion-type:jwt-bearer"},
  		"client_assertion":      {strings.TrimSpace(string(federatedToken))},
  	}
  	tokenURL := "https://login.microsoftonline.com/" + os.Getenv("AZURE_TENANT_ID") + "/oauth2/v2.0/token"
  	req, err := http.NewRequestWithContext(ctx, http.MethodPost, tokenURL, strings.NewReader(form.Encode()))
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
  		option.WithFederationTokenProvider(option.IdentityTokenFunc(fetchEntraTokenViaFederation), option.FederationOptions{
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
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hello from Azure")),
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
  IdentityTokenProvider fetchEntraTokenViaFederation = () -> {
      try {
          var form = Map.of(
                          "client_id", System.getenv("AZURE_CLIENT_ID"),
                          "grant_type", "client_credentials",
                          "scope", "api://<APP_ID>/.default",
                          "client_assertion_type", "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
                          "client_assertion", Files.readString(Path.of(System.getenv("AZURE_FEDERATED_TOKEN_FILE"))))
                  .entrySet().stream()
                  .map(entry -> entry.getKey() + "=" + URLEncoder.encode(entry.getValue(), UTF_8))
                  .collect(Collectors.joining("&"));
          var request = HttpRequest.newBuilder(URI.create(
                          "https://login.microsoftonline.com/" + System.getenv("AZURE_TENANT_ID") + "/oauth2/v2.0/token"))
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
                  fetchEntraTokenViaFederation,
                  System.getenv("ANTHROPIC_FEDERATION_RULE_ID"),
                  System.getenv("ANTHROPIC_ORGANIZATION_ID"),
                  System.getenv("ANTHROPIC_SERVICE_ACCOUNT_ID"),
                  System.getenv("ANTHROPIC_WORKSPACE_ID"))
          .build();

  var message = client.messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessage("Hello from Azure")
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
      IdentityTokenProvider = new EntraFederationTokenProvider(),
  });
  using var client = new AnthropicOidcClient(credentials);

  var message = await client.Messages.Create(new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello from Azure" }],
  });
  foreach (var block in message.Content)
  {
      if (block.Value is TextBlock textBlock)
      {
          Console.WriteLine(textBlock.Text);
      }
  }

  class EntraFederationTokenProvider : IIdentityTokenProvider
  {
      private static readonly HttpClient Http = new();

      public async Task<string> GetIdentityTokenAsync(CancellationToken ct = default)
      {
          var federatedToken = await File.ReadAllTextAsync(
              Environment.GetEnvironmentVariable("AZURE_FEDERATED_TOKEN_FILE")!, ct);
          var tenantId = Environment.GetEnvironmentVariable("AZURE_TENANT_ID");
          var form = new FormUrlEncodedContent(new Dictionary<string, string>
          {
              ["client_id"] = Environment.GetEnvironmentVariable("AZURE_CLIENT_ID")!,
              ["grant_type"] = "client_credentials",
              ["scope"] = "api://<APP_ID>/.default",
              ["client_assertion_type"] = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
              ["client_assertion"] = federatedToken,
          });
          var response = await Http.PostAsync(
              $"https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token", form, ct);
          response.EnsureSuccessStatusCode();
          using var json = await JsonDocument.ParseAsync(
              await response.Content.ReadAsStreamAsync(ct), default, ct);
          return json.RootElement.GetProperty("access_token").GetString()!;
      }
  }
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Credentials\WorkloadIdentityCredentials;

  function fetchEntraTokenViaFederation(): string
  {
      $ch = curl_init('https://login.microsoftonline.com/' . getenv('AZURE_TENANT_ID') . '/oauth2/v2.0/token');
      curl_setopt_array($ch, [
          CURLOPT_RETURNTRANSFER => true,
          CURLOPT_POSTFIELDS => http_build_query([
              'client_id' => getenv('AZURE_CLIENT_ID'),
              'grant_type' => 'client_credentials',
              'scope' => 'api://<APP_ID>/.default',
              'client_assertion_type' => 'urn:ietf:params:oauth:client-assertion-type:jwt-bearer',
              'client_assertion' => file_get_contents(getenv('AZURE_FEDERATED_TOKEN_FILE')),
          ]),
      ]);
      $body = json_decode(curl_exec($ch), true);
      curl_close($ch);
      return $body['access_token'];
  }

  $client = new Client(
      credentials: new WorkloadIdentityCredentials(
          identityTokenProvider: fetchEntraTokenViaFederation(...),
          federationRuleId: getenv('ANTHROPIC_FEDERATION_RULE_ID'),
          organizationId: getenv('ANTHROPIC_ORGANIZATION_ID'),
          serviceAccountId: getenv('ANTHROPIC_SERVICE_ACCOUNT_ID'),
          workspaceId: getenv('ANTHROPIC_WORKSPACE_ID') ?: null,
      ),
  );

  $message = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello from Azure']],
  );
  $textBlock = array_find($message->content, static fn ($block): bool => $block->type === 'text');
  echo $textBlock->text, PHP_EOL;
  ```

  ```ruby Ruby
  require "anthropic"
  require "json"
  require "net/http"

  def fetch_entra_token_via_federation
    tenant_id = ENV.fetch("AZURE_TENANT_ID")
    federated_token = File.read(ENV.fetch("AZURE_FEDERATED_TOKEN_FILE"))
    response = Net::HTTP.post_form(
      URI("https://login.microsoftonline.com/#{tenant_id}/oauth2/v2.0/token"),
      "client_id" => ENV.fetch("AZURE_CLIENT_ID"),
      "grant_type" => "client_credentials",
      "scope" => "api://<APP_ID>/.default",
      "client_assertion_type" => "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
      "client_assertion" => federated_token
    )
    JSON.parse(response.body).fetch("access_token")
  end

  client = Anthropic::Client.new(
    credentials: Anthropic::WorkloadIdentityCredentials.new(
      identity_token_provider: -> { fetch_entra_token_via_federation },
      federation_rule_id: ENV.fetch("ANTHROPIC_FEDERATION_RULE_ID"),
      organization_id: ENV.fetch("ANTHROPIC_ORGANIZATION_ID"),
      service_account_id: ENV.fetch("ANTHROPIC_SERVICE_ACCOUNT_ID"),
      workspace_id: ENV["ANTHROPIC_WORKSPACE_ID"]
    )
  )

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello from Azure"}]
  )
  puts message.content.find { it.type == :text }.text
  ```

  ```bash CLI
  # 1. 将 Kubernetes 投射的令牌交换为 Entra 颁发的访问
  # 令牌，并将其写入 CLI 可读取的临时文件。
  ANTHROPIC_IDENTITY_TOKEN_FILE=$(mktemp)
  trap 'rm -f "$ANTHROPIC_IDENTITY_TOKEN_FILE"' EXIT
  curl -sS "https://login.microsoftonline.com/$AZURE_TENANT_ID/oauth2/v2.0/token" \
    -d client_id="$AZURE_CLIENT_ID" \
    -d grant_type=client_credentials \
    --data-urlencode "scope=api://<APP_ID>/.default" \
    -d client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer \
    --data-urlencode client_assertion@"$AZURE_FEDERATED_TOKEN_FILE" \
    | jq -r .access_token > "$ANTHROPIC_IDENTITY_TOKEN_FILE"
  export ANTHROPIC_IDENTITY_TOKEN_FILE

  # 2. 调用 Claude API。ANTHROPIC_FEDERATION_RULE_ID、
  # ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID
  # 从环境变量中读取。
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello from Azure"}'
  ```
</CodeGroup>

### 验证设置

在带标签的 pod 内部，运行[获取并使用令牌](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#acquire-and-use-the-token-2)中所示的 cURL 交换，并确认 `POST /v1/oauth/token` 返回 `200`，其中 `access_token` 以 `sk-ant-oat01-` 开头，且 `expires_in` 值以秒为单位。如果交换失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`），请查看[身份验证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)了解拒绝原因，然后解码第 1 步中 Entra 颁发的令牌（有关命令，请参阅[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)），并检查最常见的 Azure 端原因：

* **颁发者不匹配：** 注册的 `issuer_url` 必须与令牌的 `iss` 声明完全匹配。v2.0 令牌携带 `https://login.microsoftonline.com/<TENANT_ID>/v2.0`；如果解码后的 `ver` 声明为 `1.0`，请参阅[如果您的令牌是 v1.0](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#if-your-tokens-are-v1-0)。
* **令牌生命周期：** 如果租户令牌生命周期策略或 CAE 将 `client_credentials` 令牌延长至超过 7500 秒，请按照[配置 Anthropic](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#configure-anthropic-2) 中所述提高颁发者的 `max_jwt_lifetime_seconds`。
* **受众不匹配：** 规则的 `audience` 必须与令牌的 `aud` 完全相等：对于本指南配置的 v2.0 令牌，即受众应用注册的客户端 ID。
* **声明名称不匹配：** 基于令牌未携带的声明进行匹配的规则永远不会通过。v1.0 令牌在 `appid` 而非 `azp` 中携带客户端 ID；请参阅[如果您的令牌是 v1.0](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure#if-your-tokens-are-v1-0)。

## 如果您的令牌是 v1.0

本指南将受众应用注册配置为 `api.requestedAccessTokenVersion: 2`，因此它展示的每个令牌都是 v2.0。如果您复用一个未设置 `requestedAccessTokenVersion` 的现有注册，Entra 会改为颁发 v1.0 令牌。解码一个示例令牌并检查其 `ver` 声明；如果它是 `1.0`，则有四处变化：

* **颁发者：** `iss` 声明为 `https://sts.windows.net/<TENANT_ID>/`，而非 `https://login.microsoftonline.com/<TENANT_ID>/v2.0`。请完全按照令牌 `iss` 声明所携带的内容注册颁发者 URL。这两个 URL 共享相同的 JWKS，因此发现模式对二者均适用。
* **向导选择器：** 在 Connect workload 向导的 **Token issuer** 选择器中选择 **v1 (sts.windows.net)**，而非 **v2.0 (login.microsoftonline.com)**。
* **受众：** `aud` 声明是您作为 `resource` 传入的标识符 URI（例如 `api://<APP_ID>`），而非注册的客户端 ID。请将联合规则的 `audience` 设置为解码后令牌中的确切 `aud` 值。
* **客户端 ID 声明：** 调用方标识的客户端 ID 出现在 `appid` 中，而非 `azp`。这两个声明永远不会出现在同一个令牌中，因此基于 `azp` 匹配的规则对 v1.0 令牌永远不会通过。

`oid`、`sub` 和 `tid` 声明在两个版本中携带相同的值，因此本指南的其余部分无需更改即可适用。

## 限定规则范围

除了 `claims` 映射之外（或代替它），联合规则还可以使用 `subject_prefix` 匹配令牌的主体；有关这些字段如何组合，请参阅[规则匹配语义](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#rule-matching-semantics)。这些标识的 Entra `sub` 值是固定长度的规范 GUID，因此包含完整 36 字符对象 ID 的 `subject_prefix` 仅匹配该主体；这是 Entra 主体格式的特性，而非 `subject_prefix` 的普遍特性。

<Warning>
  您租户中的每个标识都可以为已注册的受众请求令牌， 因此仅凭 `audience` 和 `tid` 无法识别特定的工作负载。 省略 `oid`（或 `azp`/`appid`）匹配的规则，或使用通配符或 部分 GUID `subject_prefix` 的规则，会授权租户中的每个托管标识和服务 主体。
</Warning>

将规则的 `match` 块锁定到适合您用例的最窄范围：

* **将 `oid` 作为精确值匹配：** 将 `claims.oid` 设置为托管标识的完整对象 ID。设置为该完整对象 ID 的 `subject_prefix` 与之等效（Console 向导会同时设置两者）；切勿使用通配符或部分 GUID 的 `subject_prefix`，它会匹配比您预期更多的标识。
* **固定 `tid` 作为纵深防御：** 颁发者 URL 已经固定了您的租户，但添加 `claims.tid` 可防止日后编辑颁发者记录时出现配置漂移。
* **固定受众：** 将 `audience` 设置为解码后令牌中的确切 `aud` 值，以便拒绝为其他应用程序铸造的令牌。
* **为每个托管标识使用单独的规则：** 为每个标识创建一条规则，而不是用一条规则授权多个标识，这样您就可以撤销单个工作负载的访问权限而不影响其他工作负载。

## 后续步骤

* 在[工作负载身份联合](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)中查看完整的配置模型。
* 请参阅适用于 AWS、Google Cloud、GitHub Actions 和 Kubernetes 的[提供商指南](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#identity-providers)。
* 有关环境变量、配置文件和凭据优先级，请参阅 [WIF 参考](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference)。
