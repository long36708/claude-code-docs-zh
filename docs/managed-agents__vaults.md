---
title: 使用 vault 进行身份验证
url: https://platform.claude.com/docs/zh-CN/managed-agents/vaults
description: 在创建会话时注册每个用户的凭证。
---

Vault（保管库）和 credential（凭证）是身份验证原语，让您可以一次性注册第三方服务的凭证，并在创建会话时通过 ID 引用它们。这意味着您无需运行自己的密钥存储、无需在每次调用时传输令牌，也不会丢失代理代表哪个最终用户执行操作的记录。

vault 引用是一个按会话设置的参数，因此您可以在 `agent` 资源粒度上管理您的产品，在 `session` 资源粒度上管理您的用户。

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

## 创建 vault

<Warning>
  Vault 和凭证的作用域为工作区，这意味着任何具有工作区访问权限的 API 密钥都可以在创建会话时引用它们。要撤销访问权限，请删除 vault 或凭证。
</Warning>

vault 是与某个最终用户关联的 `credentials` 集合。为其指定一个 `display_name`，并可选择使用 `metadata` 进行标记，以便您将其映射回自己的用户记录。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  vault_id=$(curl --fail-with-body -sS https://api.anthropic.com/v1/vaults \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    --data @- <<'EOF' | jq -r '.id'
  {
    "display_name": "Alice",
    "metadata": {"external_user_id": "usr_abc123"}
  }
  EOF
  )
  echo "$vault_id"  # "vlt_01ABC..."
  ```

  <MultiFileExample language="cli" label="CLI">
    ```bash CLI
    VAULT_ID=$(ant beta:vaults create --transform id --raw-output < alice.vault.yaml)
    echo "$VAULT_ID"  # "vlt_01ABC..."
    ```

    <File filename="alice.vault.yaml">
      ```yaml
      display_name: Alice
      metadata:
        external_user_id: usr_abc123
      ```
    </File>
  </MultiFileExample>

  ```python Python
  vault = client.beta.vaults.create(
      display_name="Alice",
      metadata={"external_user_id": "usr_abc123"},
  )
  print(vault.id)  # "vlt_01ABC..."
  ```

  ```typescript TypeScript
  const vault = await client.beta.vaults.create({
    display_name: "Alice",
    metadata: { external_user_id: "usr_abc123" },
  });
  console.log(vault.id); // "vlt_01ABC..."
  ```

  ```csharp C#
  var vault = await client.Beta.Vaults.Create(new()
  {
      DisplayName = "Alice",
      Metadata = new Dictionary<string, string> { ["external_user_id"] = "usr_abc123" },
  });
  Console.WriteLine(vault.ID); // "vlt_01ABC..."
  ```

  ```go Go
  vault, err := client.Beta.Vaults.New(ctx, anthropic.BetaVaultNewParams{
  	DisplayName: "Alice",
  	Metadata:    map[string]string{"external_user_id": "usr_abc123"},
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(vault.ID) // "vlt_01ABC..."
  ```

  ```java Java
  var vault = client.beta().vaults().create(VaultCreateParams.builder()
      .displayName("Alice")
      .metadata(VaultCreateParams.Metadata.builder()
          .putAdditionalProperty("external_user_id", JsonValue.from("usr_abc123"))
          .build())
      .build());
  IO.println(vault.id()); // "vlt_01ABC..."
  ```

  ```php PHP
  $vault = $client->beta->vaults->create(
      displayName: 'Alice',
      metadata: ['external_user_id' => 'usr_abc123'],
  );
  echo $vault->id . "\n"; // "vlt_01ABC..."
  ```

  ```ruby Ruby
  vault = client.beta.vaults.create(
    display_name: "Alice",
    metadata: {external_user_id: "usr_abc123"}
  )
  puts vault.id # "vlt_01ABC..."
  ```
</CodeGroup>

响应是完整的 vault 记录：

```json
{
  "type": "vault",
  "id": "vlt_01ABC...",
  "display_name": "Alice",
  "metadata": { "external_user_id": "usr_abc123" },
  "created_at": "2026-03-18T10:00:00Z",
  "updated_at": "2026-03-18T10:00:00Z",
  "archived_at": null
}
```

## 添加凭证

支持两类凭证：

* **MCP 凭证**（`mcp_oauth`、`static_bearer`）：每个凭证以 `mcp_server_url` 作为键。当代理在会话运行时连接到该 URL 的服务器时，令牌会自动注入。
* **环境变量**（`environment_variable`）：每个凭证以 `secret_name`（环境变量名称）作为键，并以不透明占位符的形式存储在沙箱中。当代理发起出站请求时，不透明占位符会在出口处被替换为真实密钥。代理永远不会看到密钥值。对于任何通过环境变量进行身份验证的服务（例如 CLI、SDK 或直接 API 调用），请使用此类型。

您提供的实际凭证值（`token`、`access_token`、`refresh_token`、`client_secret`、`secret_value`）被视为敏感的只写字段，永远不会在 API 响应中返回。

<Note>
  环境变量凭证（`environment_variable`）尚不支持与[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)一起使用。
</Note>

<Tabs>
  <Tab title="MCP OAuth">
    当 MCP 服务器使用 OAuth 2.0 时，请使用 `mcp_oauth`。如果您提供了 `refresh` 块，Anthropic 会在访问令牌过期时代您刷新。

    `refresh.token_endpoint_auth.type` 字段指示如何对刷新调用进行身份验证：

    * `none`：公共客户端
    * `client_secret_basic`：使用客户端密钥的 HTTP Basic 身份验证
    * `client_secret_post`：客户端密钥位于 POST 请求体中

    <CodeGroup defaultLanguage="CLI">
      ```bash cURL
      credential_id=$(curl --fail-with-body -sS "https://api.anthropic.com/v1/vaults/$vault_id/credentials" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" \
        -H "content-type: application/json" \
        --data @- <<'EOF' | jq -r '.id'
      {
        "display_name": "Alice's Slack",
        "auth": {
          "type": "mcp_oauth",
          "mcp_server_url": "https://mcp.slack.com/mcp",
          "access_token": "xoxp-...",
          "expires_at": "2099-12-31T23:59:59Z",
          "refresh": {
            "token_endpoint": "https://slack.com/api/oauth.v2.access",
            "client_id": "1234567890.0987654321",
            "scope": "channels:read chat:write",
            "refresh_token": "xoxe-1-...",
            "token_endpoint_auth": {"type": "client_secret_post", "client_secret": "abc123..."}
          }
        }
      }
      EOF
      )
      ```

      ```bash CLI
      CREDENTIAL_ID=$(ant beta:vaults:credentials create \
        --vault-id "$VAULT_ID" \
        --display-name "Alice's Slack" \
        --transform id --raw-output <<'YAML'
      auth:
        type: mcp_oauth
        mcp_server_url: https://mcp.slack.com/mcp
        access_token: xoxp-...
        expires_at: "2099-12-31T23:59:59Z"
        refresh:
          token_endpoint: https://slack.com/api/oauth.v2.access
          client_id: "1234567890.0987654321"
          scope: channels:read chat:write
          refresh_token: xoxe-1-...
          token_endpoint_auth:
            type: client_secret_post
            client_secret: abc123...
      YAML
      )
      ```

      ```python Python
      credential = client.beta.vaults.credentials.create(
          vault_id=vault.id,
          display_name="Alice's Slack",
          auth={
              "type": "mcp_oauth",
              "mcp_server_url": "https://mcp.slack.com/mcp",
              "access_token": "xoxp-...",
              "expires_at": "2099-12-31T23:59:59Z",
              "refresh": {
                  "token_endpoint": "https://slack.com/api/oauth.v2.access",
                  "client_id": "1234567890.0987654321",
                  "scope": "channels:read chat:write",
                  "refresh_token": "xoxe-1-...",
                  "token_endpoint_auth": {"type": "client_secret_post", "client_secret": "abc123..."},
              },
          },
      )
      ```

      ```typescript TypeScript
      const credential = await client.beta.vaults.credentials.create(vault.id, {
        display_name: "Alice's Slack",
        auth: {
          type: "mcp_oauth",
          mcp_server_url: "https://mcp.slack.com/mcp",
          access_token: "xoxp-...",
          expires_at: "2099-12-31T23:59:59Z",
          refresh: {
            token_endpoint: "https://slack.com/api/oauth.v2.access",
            client_id: "1234567890.0987654321",
            scope: "channels:read chat:write",
            refresh_token: "xoxe-1-...",
            token_endpoint_auth: {
              type: "client_secret_post",
              client_secret: "abc123...",
            },
          },
        },
      });
      ```

      ```csharp C#
      var credential = await client.Beta.Vaults.Credentials.Create(vault.ID, new()
      {
          DisplayName = "Alice's Slack",
          Auth = new BetaManagedAgentsMcpOAuthCreateParams
          {
              Type = BetaManagedAgentsMcpOAuthCreateParamsType.McpOAuth,
              McpServerUrl = "https://mcp.slack.com/mcp",
              AccessToken = "xoxp-...",
              ExpiresAt = DateTimeOffset.Parse("2099-12-31T23:59:59Z"),
              Refresh = new()
              {
                  TokenEndpoint = "https://slack.com/api/oauth.v2.access",
                  ClientID = "1234567890.0987654321",
                  Scope = "channels:read chat:write",
                  RefreshToken = "xoxe-1-...",
                  TokenEndpointAuth = new BetaManagedAgentsTokenEndpointAuthPostParam
                  {
                      Type = BetaManagedAgentsTokenEndpointAuthPostParamType.ClientSecretPost,
                      ClientSecret = "abc123...",
                  },
              },
          },
      });
      ```

      ```go Go
      credential, err := client.Beta.Vaults.Credentials.New(ctx, vault.ID, anthropic.BetaVaultCredentialNewParams{
      	DisplayName: anthropic.String("Alice's Slack"),
      	Auth: anthropic.BetaVaultCredentialNewParamsAuthUnion{
      		OfMCPOAuth: &anthropic.BetaManagedAgentsMCPOAuthCreateParams{
      			Type:         anthropic.BetaManagedAgentsMCPOAuthCreateParamsTypeMCPOAuth,
      			MCPServerURL: "https://mcp.slack.com/mcp",
      			AccessToken:  "xoxp-...",
      			ExpiresAt:    anthropic.Time(time.Date(2099, time.December, 31, 23, 59, 59, 0, time.UTC)),
      			Refresh: anthropic.BetaManagedAgentsMCPOAuthRefreshParams{
      				TokenEndpoint: "https://slack.com/api/oauth.v2.access",
      				ClientID:      "1234567890.0987654321",
      				Scope:         anthropic.String("channels:read chat:write"),
      				RefreshToken:  "xoxe-1-...",
      				TokenEndpointAuth: anthropic.BetaManagedAgentsMCPOAuthRefreshParamsTokenEndpointAuthUnion{
      					OfClientSecretPost: &anthropic.BetaManagedAgentsTokenEndpointAuthPostParam{
      						Type:         anthropic.BetaManagedAgentsTokenEndpointAuthPostParamTypeClientSecretPost,
      						ClientSecret: "abc123...",
      					},
      				},
      			},
      		},
      	},
      })
      if err != nil {
      	panic(err)
      }
      ```

      ```java Java
      var credential = client.beta().vaults().credentials().create(vault.id(),
          CredentialCreateParams.builder()
              .displayName("Alice's Slack")
              .auth(BetaManagedAgentsMcpOAuthCreateParams.builder()
                  .type(BetaManagedAgentsMcpOAuthCreateParams.Type.MCP_OAUTH)
                  .mcpServerUrl("https://mcp.slack.com/mcp")
                  .accessToken("xoxp-...")
                  .expiresAt(OffsetDateTime.parse("2099-12-31T23:59:59Z"))
                  .refresh(BetaManagedAgentsMcpOAuthRefreshParams.builder()
                      .tokenEndpoint("https://slack.com/api/oauth.v2.access")
                      .clientId("1234567890.0987654321")
                      .scope("channels:read chat:write")
                      .refreshToken("xoxe-1-...")
                      .clientSecretPostTokenEndpointAuth("abc123...")
                      .build())
                  .build())
              .build());
      ```

      ```php PHP
      $credential = $client->beta->vaults->credentials->create(
          vaultID: $vault->id,
          displayName: "Alice's Slack",
          auth: ManagedAgentsMCPOAuthCreateParams::with(
              type: 'mcp_oauth',
              mcpServerURL: 'https://mcp.slack.com/mcp',
              accessToken: 'xoxp-...',
              expiresAt: new DateTimeImmutable('2099-12-31T23:59:59Z'),
              refresh: ManagedAgentsMCPOAuthRefreshParams::with(
                  tokenEndpoint: 'https://slack.com/api/oauth.v2.access',
                  clientID: '1234567890.0987654321',
                  scope: 'channels:read chat:write',
                  refreshToken: 'xoxe-1-...',
                  tokenEndpointAuth: ManagedAgentsTokenEndpointAuthPostParam::with(
                      type: 'client_secret_post',
                      clientSecret: 'abc123...',
                  ),
              ),
          ),
      );
      ```

      ```ruby Ruby
      credential = client.beta.vaults.credentials.create(
        vault.id,
        display_name: "Alice's Slack",
        auth: {
          type: "mcp_oauth",
          mcp_server_url: "https://mcp.slack.com/mcp",
          access_token: "xoxp-...",
          expires_at: "2099-12-31T23:59:59Z",
          refresh: {
            token_endpoint: "https://slack.com/api/oauth.v2.access",
            client_id: "1234567890.0987654321",
            scope: "channels:read chat:write",
            refresh_token: "xoxe-1-...",
            token_endpoint_auth: {
              type: "client_secret_post",
              client_secret: "abc123..."
            }
          }
        }
      )
      ```
    </CodeGroup>
  </Tab>

  <Tab title="MCP 静态 bearer">
    当 MCP 服务器接受固定的 bearer 令牌（API 密钥、个人访问令牌或类似令牌）时，请使用 `static_bearer`。无需刷新流程。

    <CodeGroup defaultLanguage="CLI">
      ```bash cURL
      curl --fail-with-body -sS "https://api.anthropic.com/v1/vaults/$vault_id/credentials" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" \
        -H "content-type: application/json" \
        --data @- <<'EOF'
      {
        "display_name": "Linear API key",
        "auth": {
          "type": "static_bearer",
          "mcp_server_url": "https://mcp.linear.app/mcp",
          "token": "lin_api_your_linear_key"
        }
      }
      EOF
      ```

      ```bash CLI
      ant beta:vaults:credentials create --vault-id "$VAULT_ID" <<'YAML'
      display_name: Linear API key
      auth:
        type: static_bearer
        mcp_server_url: https://mcp.linear.app/mcp
        token: lin_api_your_linear_key
      YAML
      ```

      ```python Python
      bearer_credential = client.beta.vaults.credentials.create(
          vault_id=vault.id,
          display_name="Linear API key",
          auth={
              "type": "static_bearer",
              "mcp_server_url": "https://mcp.linear.app/mcp",
              "token": "lin_api_your_linear_key",
          },
      )
      ```

      ```typescript TypeScript
      const bearerCredential = await client.beta.vaults.credentials.create(vault.id, {
        display_name: "Linear API key",
        auth: {
          type: "static_bearer",
          mcp_server_url: "https://mcp.linear.app/mcp",
          token: "lin_api_your_linear_key",
        },
      });
      ```

      ```csharp C#
      var bearerCredential = await client.Beta.Vaults.Credentials.Create(vault.ID, new()
      {
          DisplayName = "Linear API key",
          Auth = new BetaManagedAgentsStaticBearerCreateParams
          {
              Type = BetaManagedAgentsStaticBearerCreateParamsType.StaticBearer,
              McpServerUrl = "https://mcp.linear.app/mcp",
              Token = "lin_api_your_linear_key",
          },
      });
      ```

      ```go Go
      bearerCredential, err := client.Beta.Vaults.Credentials.New(ctx, vault.ID, anthropic.BetaVaultCredentialNewParams{
      	DisplayName: anthropic.String("Linear API key"),
      	Auth: anthropic.BetaVaultCredentialNewParamsAuthUnion{
      		OfStaticBearer: &anthropic.BetaManagedAgentsStaticBearerCreateParams{
      			Type:         anthropic.BetaManagedAgentsStaticBearerCreateParamsTypeStaticBearer,
      			MCPServerURL: "https://mcp.linear.app/mcp",
      			Token:        "lin_api_your_linear_key",
      		},
      	},
      })
      if err != nil {
      	panic(err)
      }
      _ = bearerCredential
      ```

      ```java Java
      var bearerCredential = client.beta().vaults().credentials().create(vault.id(),
          CredentialCreateParams.builder()
              .displayName("Linear API key")
              .auth(BetaManagedAgentsStaticBearerCreateParams.builder()
                  .type(BetaManagedAgentsStaticBearerCreateParams.Type.STATIC_BEARER)
                  .mcpServerUrl("https://mcp.linear.app/mcp")
                  .token("lin_api_your_linear_key")
                  .build())
              .build());
      ```

      ```php PHP
      $bearerCredential = $client->beta->vaults->credentials->create(
          vaultID: $vault->id,
          displayName: 'Linear API key',
          auth: ManagedAgentsStaticBearerCreateParams::with(
              type: 'static_bearer',
              mcpServerURL: 'https://mcp.linear.app/mcp',
              token: 'lin_api_your_linear_key',
          ),
      );
      ```

      ```ruby Ruby
      bearer_credential = client.beta.vaults.credentials.create(
        vault.id,
        display_name: "Linear API key",
        auth: {
          type: "static_bearer",
          mcp_server_url: "https://mcp.linear.app/mcp",
          token: "lin_api_your_linear_key"
        }
      )
      ```
    </CodeGroup>
  </Tab>

  <Tab title="环境变量">
    使用 `environment_variable` 通过环境变量向外部服务进行身份验证，例如 CLI、SDK 或直接 API 调用。环境变量凭证适用于在出站请求中原样发送密钥值的客户端，因此在配置之前，请先查看本标签页中的客户端适用条件。

    `networking.allowed_hosts` 数组控制密钥可以被替换到哪些出站主机。使用 `"type": "limited"` 并指定具体列表；如果调用方会访问您无法预先枚举的域名，则使用 `"type": "unrestricted"`。

    出于安全考虑，强烈建议限制域名，这可以防止您的密钥被共享给未经授权的主机。

    <Note>
      vault 凭证上的 `networking.allowed_hosts` 控制哪些请求使用该密钥，而不是哪些请求被允许。要让代理真正访问某个域名，该域名还必须在[环境级别](https://platform.claude.com/docs/zh-CN/managed-agents/environments)被允许。两个级别都必须包含该域名（通过 `unrestricted` 网络设置，或在 `allowed_hosts` 中显式列出该域名），经过密钥替换的请求才能成功。
    </Note>

    可选的 `injection_location` 字段限定密钥被替换的位置；完整语义见示例之后的说明。

    <CodeGroup defaultLanguage="CLI">
      ```bash cURL
      curl --fail-with-body -sS "https://api.anthropic.com/v1/vaults/$vault_id/credentials" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" \
        -H "content-type: application/json" \
        --data @- <<'EOF' | jq '.auth.injection_location'
      {
        "auth": {
          "type": "environment_variable",
          "secret_name": "NOTION_API_KEY",
          "secret_value": "ntn_your-secret-here",
          "networking": {
            "type": "limited",
            "allowed_hosts": ["api.notion.com"]
          },
          "injection_location": {"header": true}
        },
        "display_name": "Notion API key for sandbox"
      }
      EOF
      ```

      ```bash CLI
      ant beta:vaults:credentials create \
        --vault-id "$VAULT_ID" \
        --transform 'auth.injection_location' --format json <<'YAML'
      display_name: Notion API key for sandbox
      auth:
        type: environment_variable
        secret_name: NOTION_API_KEY
        secret_value: ntn_your-secret-here
        injection_location:
          header: true
        networking:
          type: limited
          allowed_hosts: [api.notion.com]
      YAML
      ```

      ```python Python
      env_credential = client.beta.vaults.credentials.create(
          vault_id=vault.id,
          display_name="Notion API key for sandbox",
          auth={
              "type": "environment_variable",
              "secret_name": "NOTION_API_KEY",
              "secret_value": "ntn_your-secret-here",
              "networking": {
                  "type": "limited",
                  "allowed_hosts": ["api.notion.com"],
              },
              "injection_location": {"header": True},
          },
      )
      if env_credential.auth.type == "environment_variable":
          location = env_credential.auth.injection_location
          print(f"header: {location.header}, body: {location.body}")  # header: True, body: False
      ```

      ```typescript TypeScript
      const envVarCredential = await client.beta.vaults.credentials.create(vault.id, {
        display_name: "Notion API key for sandbox",
        auth: {
          type: "environment_variable",
          secret_name: "NOTION_API_KEY",
          secret_value: "ntn_your-secret-here",
          networking: {
            type: "limited",
            allowed_hosts: ["api.notion.com"],
          },
          injection_location: { header: true },
        },
      });
      if (envVarCredential.auth.type === "environment_variable") {
        console.log(envVarCredential.auth.injection_location); // { header: true, body: false }
      }
      ```

      ```csharp C#
      var envVarCredential = await client.Beta.Vaults.Credentials.Create(vault.ID, new()
      {
          DisplayName = "Notion API key for sandbox",
          Auth = new BetaManagedAgentsEnvironmentVariableCreateParams
          {
              Type = BetaManagedAgentsEnvironmentVariableCreateParamsType.EnvironmentVariable,
              SecretName = "NOTION_API_KEY",
              SecretValue = "ntn_your-secret-here",
              Networking = new BetaManagedAgentsLimitedCredentialNetworkingParams
              {
                  Type = BetaManagedAgentsLimitedCredentialNetworkingParamsType.Limited,
                  AllowedHosts = ["api.notion.com"],
              },
              InjectionLocation = new() { Header = true },
          },
      });
      if (envVarCredential.Auth.TryPickBetaManagedAgentsEnvironmentVariableAuthResponse(out var envVarAuth))
      {
          var injectionLocation = envVarAuth.InjectionLocation;
          Console.WriteLine($"Header: {injectionLocation.Header}, Body: {injectionLocation.Body}"); // "Header: True, Body: False"
      }
      ```

      ```go Go
      envVarCredential, err := client.Beta.Vaults.Credentials.New(ctx, vault.ID, anthropic.BetaVaultCredentialNewParams{
      	DisplayName: anthropic.String("Notion API key for sandbox"),
      	Auth: anthropic.BetaVaultCredentialNewParamsAuthUnion{
      		OfEnvironmentVariable: &anthropic.BetaManagedAgentsEnvironmentVariableCreateParams{
      			Type:        anthropic.BetaManagedAgentsEnvironmentVariableCreateParamsTypeEnvironmentVariable,
      			SecretName:  "NOTION_API_KEY",
      			SecretValue: "ntn_your-secret-here",
      			Networking: anthropic.BetaManagedAgentsCredentialNetworkingParamsUnion{
      				OfLimited: &anthropic.BetaManagedAgentsLimitedCredentialNetworkingParams{
      					Type:         anthropic.BetaManagedAgentsLimitedCredentialNetworkingParamsTypeLimited,
      					AllowedHosts: []string{"api.notion.com"},
      				},
      			},
      			InjectionLocation: anthropic.BetaManagedAgentsInjectionLocationParams{
      				Header: anthropic.Bool(true),
      			},
      		},
      	},
      })
      if err != nil {
      	panic(err)
      }
      if envVarAuth, ok := envVarCredential.Auth.AsAny().(anthropic.BetaManagedAgentsEnvironmentVariableAuthResponse); ok {
      	injectionLocation := envVarAuth.InjectionLocation
      	fmt.Printf("Header:%t Body:%t\n", injectionLocation.Header, injectionLocation.Body) // "Header:true Body:false"
      }
      ```

      ```java Java
      var envVarCredential = client.beta().vaults().credentials().create(vault.id(),
          CredentialCreateParams.builder()
              .displayName("Notion API key for sandbox")
              .auth(BetaManagedAgentsEnvironmentVariableCreateParams.builder()
                  .type(BetaManagedAgentsEnvironmentVariableCreateParams.Type.ENVIRONMENT_VARIABLE)
                  .secretName("NOTION_API_KEY")
                  .secretValue("ntn_your-secret-here")
                  .limitedNetworking(List.of("api.notion.com"))
                  .injectionLocation(BetaManagedAgentsInjectionLocationParams.builder()
                      .header(true)
                      .build())
                  .build())
              .build());
      envVarCredential.auth().environmentVariable().ifPresent(envVarAuth -> {
          var injectionLocation = envVarAuth.injectionLocation();
          IO.println("header=" + injectionLocation.header() + " body=" + injectionLocation.body()); // header=true body=false
      });
      ```

      ```php PHP
      $envVarCredential = $client->beta->vaults->credentials->create(
          vaultID: $vault->id,
          displayName: 'Notion API key for sandbox',
          auth: ManagedAgentsEnvironmentVariableCreateParams::with(
              type: ManagedAgentsEnvironmentVariableCreateParams\Type::ENVIRONMENT_VARIABLE,
              secretName: 'NOTION_API_KEY',
              secretValue: 'ntn_your-secret-here',
              networking: ManagedAgentsLimitedCredentialNetworkingParams::with(
                  type: ManagedAgentsLimitedCredentialNetworkingParams\Type::LIMITED,
                  allowedHosts: ['api.notion.com'],
              ),
              injectionLocation: ManagedAgentsInjectionLocationParams::with(header: true),
          ),
      );
      if ($envVarCredential->auth instanceof ManagedAgentsEnvironmentVariableAuthResponse) {
          $injectionLocation = $envVarCredential->auth->injectionLocation;
          echo 'header: ' . json_encode($injectionLocation->header) . "\n"; // header: true
          echo 'body: ' . json_encode($injectionLocation->body) . "\n"; // body: false
      }
      ```

      ```ruby Ruby
      env_credential = client.beta.vaults.credentials.create(
        vault.id,
        display_name: "Notion API key for sandbox",
        auth: {
          type: "environment_variable",
          secret_name: "NOTION_API_KEY",
          secret_value: "ntn_your-secret-here",
          networking: {
            type: "limited",
            allowed_hosts: ["api.notion.com"]
          },
          injection_location: {header: true}
        }
      )
      if env_credential.auth.type == :environment_variable
        env_credential.auth.injection_location => {header:, body:}
        puts "header: #{header}, body: #{body}" # header: true, body: false
      end
      ```
    </CodeGroup>

    请求负载通常由代理正在处理的内容组装而成，因此请求体是更广的暴露面。大多数服务从请求头中读取 API 密钥，因此仅启用 `header` 是更窄的配置。它将该凭证的替换范围限定为请求头的值。

    凭证的 `injection_location` 控制密钥被替换到出站请求的哪些部分。它是一个可选对象，与 `networking` 同级，包含两个布尔字段：`header`（请求头）和 `body`（请求体）。`injection_location` 独立于 `networking.allowed_hosts`：`allowed_hosts` 限定密钥被替换到哪些主机，而 `injection_location` 限定密钥被替换到请求的哪些部分。

    `injection_location` 在创建和更新时的行为不同：

    | 操作   | `injection_location` 行为                                                           |
    | ---- | --------------------------------------------------------------------------------- |
    | 创建凭证 | 如果您提供了该对象，其中省略的任何字段默认为 `false`：`{"header": true}` 会创建一个仅限请求头的凭证。完全省略该对象则两个位置都会启用。 |
    | 更新凭证 | 字段逐个合并：`{"body": false}` 会禁用请求体替换，并保持 `header` 不变。                                |

    凭证必须至少启用一个位置，因此会导致两个位置都被禁用的创建或更新操作将返回 400 错误。为 `injection_location` 对象或其中任一字段显式传递 `null` 也会返回 400 错误（"omit the field instead"，即请改为省略该字段）。响应始终返回两个字段及其解析后的值。

    位于已禁用位置的占位符既不会被替换也不会被移除。请求会带着该位置上的字面不透明占位符字符串发送给第三方。如果到达第三方的请求包含字面占位符字符串，则要么该凭证的该位置已被禁用，要么目标主机未被该凭证的 `networking.allowed_hosts` 覆盖。

    <Note>
      在 Console 中创建的凭证仅启用请求头注入。如果您的客户端在请求体中发送密钥（例如表单编码的令牌请求），占位符会按字面原样传递，服务会以其自身的身份验证错误拒绝该请求。请在创建凭证时在 Console 表单中启用请求体注入，或使用 `{"injection_location": {"body": true}}` 更新凭证。
    </Note>

    替换发生在出口处，而不是在沙箱内部。任何在本地处理凭证的程序看到的都是不透明占位符，而不是真实值：在启动时验证凭证格式的客户端可能会拒绝它，而根据密钥计算请求签名的客户端（例如 AWS SigV4）会生成无效签名。环境变量凭证适用于在出站请求中、在凭证的 `injection_location` 所启用的位置原样发送密钥值的客户端。

    替换仅针对出站方向。如果客户端使用存储的密钥获取会话令牌（例如 OAuth 客户端凭证授权），返回的令牌会以未脱敏的形式到达沙箱。对于基于交换的流程，请自行执行交换，并将生成的令牌存储在 vault 中。

    <Tip>
      将 API 密钥的权限范围限定为代理所需的权限。代理可以执行该密钥允许的任何操作，因此权限超出必要范围的密钥会在代理行为异常时扩大影响范围。
    </Tip>
  </Tab>
</Tabs>

凭证按提供的原样存储，直到会话运行时才会进行验证。无效凭证会在会话期间表现为身份验证错误或下游错误，该错误会被发出，但不会阻止会话继续进行。

约束：

* **每个 vault 内键唯一。** `mcp_server_url`（MCP 凭证）和 `secret_name`（环境变量凭证）在 vault 的活动凭证中必须唯一。创建重复项会返回 409。
* **键不可变。** 要更改 `mcp_server_url` 或 `secret_name`，请归档该凭证并创建一个新凭证。
* **每个 vault 最多 20 个凭证。**

## 在创建会话时引用 vault

创建会话时传递 `vault_ids`：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  session_id=$(curl --fail-with-body -sS https://api.anthropic.com/v1/sessions \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    --data @- <<EOF | jq -r '.id'
  {
    "agent": "$agent_id",
    "environment_id": "$environment_id",
    "vault_ids": ["$vault_id"],
    "title": "Alice's Slack digest"
  }
  EOF
  )
  ```

  ```bash CLI
  SESSION_ID=$(ant beta:sessions create \
    --agent "$AGENT_ID" \
    --environment-id "$ENVIRONMENT_ID" \
    --vault-id "$VAULT_ID" \
    --title "Alice's Slack digest" \
    --transform id --raw-output)
  ```

  ```python Python
  session = client.beta.sessions.create(
      agent=agent.id,
      environment_id=environment.id,
      vault_ids=[vault.id],
      title="Alice's Slack digest",
  )
  ```

  ```typescript TypeScript
  const session = await client.beta.sessions.create({
    agent: agent.id,
    environment_id: environment.id,
    vault_ids: [vault.id],
    title: "Alice's Slack digest",
  });
  ```

  ```csharp C#
  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = agent.ID,
      EnvironmentID = environment.ID,
      VaultIds = [vault.ID],
      Title = "Alice's Slack digest",
  });
  ```

  ```go Go
  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent: anthropic.BetaSessionNewParamsAgentUnion{
  		OfString: anthropic.String(agent.ID),
  	},
  	EnvironmentID: environment.ID,
  	VaultIDs:      []string{vault.ID},
  	Title:         anthropic.String("Alice's Slack digest"),
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var session = client.beta().sessions().create(SessionCreateParams.builder()
      .agent(agent.id())
      .environmentId(environment.id())
      .vaultIds(List.of(vault.id()))
      .title("Alice's Slack digest")
      .build());
  ```

  ```php PHP
  $session = $client->beta->sessions->create(
      agent: $agent->id,
      environmentID: $environment->id,
      vaultIDs: [$vault->id],
      title: "Alice's Slack digest",
  );
  ```

  ```ruby Ruby
  session = client.beta.sessions.create(
    agent: agent.id,
    environment_id: environment.id,
    vault_ids: [vault.id],
    title: "Alice's Slack digest"
  )
  ```
</CodeGroup>

运行时行为：

* 当没有 MCP 凭证按 `mcp_server_url` 匹配时，会尝试以未经身份验证的方式连接，如果服务器要求身份验证则会报错。
* 当多个 vault 包含匹配的凭证时，第一个匹配的 vault 优先。
* 在[多代理会话](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)中，vault 凭证适用于每个线程。自身定义中声明了匹配 MCP 服务器的代理会使用这些凭证进行身份验证。请参阅[将代理连接到 MCP 服务器](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#connect-agents-to-mcp-servers)。

## 轮换凭证

密钥值、`display_name` 以及（环境变量凭证上的）`injection_location` 可以更新。`injection_location` 的更新按字段合并，如[添加凭证](https://platform.claude.com/docs/zh-CN/managed-agents/vaults#add-a-credential)的"环境变量"标签页中所述。对于正在运行的会话，`injection_location` 更新的传播方式与密钥轮换相同：会话的凭证会在无需重启的情况下重新解析（如[凭证生命周期](https://platform.claude.com/docs/zh-CN/managed-agents/vaults#credential-lifecycle)中所述），更新后的位置将应用于会话后续的出站请求。结构性字段（`mcp_server_url`、`secret_name`、`token_endpoint`、`client_id`）在创建后即被锁定。要更改它们，请归档该凭证并创建一个新凭证。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl --fail-with-body -sS \
    "https://api.anthropic.com/v1/vaults/$vault_id/credentials/$credential_id" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    --data @- <<'EOF' > /dev/null
  {
    "auth": {
      "type": "mcp_oauth",
      "access_token": "xoxp-new-...",
      "expires_at": "2099-12-31T23:59:59Z",
      "refresh": {"refresh_token": "xoxe-1-new-..."}
    }
  }
  EOF
  ```

  ```bash CLI
  ant beta:vaults:credentials update \
    --vault-id "$VAULT_ID" \
    --credential-id "$CREDENTIAL_ID" <<'YAML'
  auth:
    type: mcp_oauth
    access_token: xoxp-new-...
    expires_at: "2099-12-31T23:59:59Z"
    refresh:
      refresh_token: xoxe-1-new-...
  YAML
  ```

  ```python Python
  client.beta.vaults.credentials.update(
      credential.id,
      vault_id=vault.id,
      auth={
          "type": "mcp_oauth",
          "access_token": "xoxp-new-...",
          "expires_at": "2099-12-31T23:59:59Z",
          "refresh": {"refresh_token": "xoxe-1-new-..."},
      },
  )
  ```

  ```typescript TypeScript
  await client.beta.vaults.credentials.update(credential.id, {
    vault_id: vault.id,
    auth: {
      type: "mcp_oauth",
      access_token: "xoxp-new-...",
      expires_at: "2099-12-31T23:59:59Z",
      refresh: {
        refresh_token: "xoxe-1-new-...",
      },
    },
  });
  ```

  ```csharp C#
  await client.Beta.Vaults.Credentials.Update(credential.ID, new()
  {
      VaultID = vault.ID,
      Auth = new BetaManagedAgentsMcpOAuthUpdateParams
      {
          Type = BetaManagedAgentsMcpOAuthUpdateParamsType.McpOAuth,
          AccessToken = "xoxp-new-...",
          ExpiresAt = DateTimeOffset.Parse("2099-12-31T23:59:59Z"),
          Refresh = new() { RefreshToken = "xoxe-1-new-..." },
      },
  });
  ```

  ```go Go
  _, err = client.Beta.Vaults.Credentials.Update(ctx, credential.ID, anthropic.BetaVaultCredentialUpdateParams{
  	VaultID: vault.ID,
  	Auth: anthropic.BetaVaultCredentialUpdateParamsAuthUnion{
  		OfMCPOAuth: &anthropic.BetaManagedAgentsMCPOAuthUpdateParams{
  			Type:        anthropic.BetaManagedAgentsMCPOAuthUpdateParamsTypeMCPOAuth,
  			AccessToken: anthropic.String("xoxp-new-..."),
  			ExpiresAt:   anthropic.Time(time.Date(2099, time.December, 31, 23, 59, 59, 0, time.UTC)),
  			Refresh: anthropic.BetaManagedAgentsMCPOAuthRefreshUpdateParams{
  				RefreshToken: anthropic.String("xoxe-1-new-..."),
  			},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().vaults().credentials().update(credential.id(),
      CredentialUpdateParams.builder()
          .vaultId(vault.id())
          .auth(BetaManagedAgentsMcpOAuthUpdateParams.builder()
              .type(BetaManagedAgentsMcpOAuthUpdateParams.Type.MCP_OAUTH)
              .accessToken("xoxp-new-...")
              .expiresAt(OffsetDateTime.parse("2099-12-31T23:59:59Z"))
              .refresh(BetaManagedAgentsMcpOAuthRefreshUpdateParams.builder()
                  .refreshToken("xoxe-1-new-...")
                  .build())
              .build())
          .build());
  ```

  ```php PHP
  $client->beta->vaults->credentials->update(
      $credential->id,
      vaultID: $vault->id,
      auth: ManagedAgentsMCPOAuthUpdateParams::with(
          type: 'mcp_oauth',
          accessToken: 'xoxp-new-...',
          expiresAt: new DateTimeImmutable('2099-12-31T23:59:59Z'),
          refresh: ManagedAgentsMCPOAuthRefreshUpdateParams::with(refreshToken: 'xoxe-1-new-...'),
      ),
  );
  ```

  ```ruby Ruby
  client.beta.vaults.credentials.update(
    credential.id,
    vault_id: vault.id,
    auth: {
      type: "mcp_oauth",
      access_token: "xoxp-new-...",
      expires_at: "2099-12-31T23:59:59Z",
      refresh: {refresh_token: "xoxe-1-new-..."}
    }
  )
  ```
</CodeGroup>

## 凭证生命周期

凭证会定期重新解析，无论是在会话期间还是在 vault 生命周期中。这确保了凭证的轮换、归档或删除能够在无需重启的情况下传播到正在运行的会话。

如需在凭证被归档、删除或刷新失败时收到通知，您可以订阅与这些生命周期变更相关的 vault 和凭证 [webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks)。

| 事件                                | 触发条件                                                     |
| --------------------------------- | -------------------------------------------------------- |
| `vault.archived`                  | Vault 已归档。同时会为每个底层凭证发出一个 `vault_credential.archived` 事件。 |
| `vault.deleted`                   | Vault 已删除。同时会为每个底层凭证发出一个 `vault_credential.deleted` 事件。  |
| `vault_credential.archived`       | 凭证已归档，可能是直接归档，也可能是 vault 归档的结果。                          |
| `vault_credential.deleted`        | 凭证已删除，可能是直接删除，也可能是 vault 删除的结果。                          |
| `vault_credential.refresh_failed` | `mcp_oauth` 凭证无法刷新（刷新令牌无效，或 OAuth 服务器返回不可恢复的错误）。         |

<Note>
  这是一份非完整的 webhook 列表；完整列表请参阅[订阅 webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks)。
</Note>

对于 `mcp_oauth` 凭证，重新解析还会在访问令牌过期时刷新它。如果刷新失败，会发出 `vault_credential.refresh_failed` 事件。

### 诊断 OAuth 刷新失败

要诊断刷新失败的原因，请调用 `POST /v1/vaults/{vault_id}/credentials/{credential_id}/mcp_oauth_validate`（或在 SDK 中调用 `client.beta.vaults.credentials.mcp_oauth_validate(...)`）。这让您可以决定如何处理该失败；正确的操作取决于错误类型。

顶层 `status` 告诉您下一步该做什么：

* `valid`：令牌有效；无需操作。
* `invalid`：授权已失效，或 OAuth 服务器以 4xx 拒绝了刷新。提示最终用户重新授权。
* `unknown`：暂时性错误（5xx、429 或网络故障）。等待并重试。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl --fail-with-body -sS -X POST \
    "https://api.anthropic.com/v1/vaults/$vault_id/credentials/$credential_id/mcp_oauth_validate?beta=true" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"
  ```

  ```bash CLI
  ant beta:vaults:credentials mcp-oauth-validate \
    --vault-id "$VAULT_ID" \
    --credential-id "$CREDENTIAL_ID" \
    --transform status --raw-output  # "valid", "invalid", or "unknown"
  ```

  ```python Python
  validation = client.beta.vaults.credentials.mcp_oauth_validate(
      credential.id,
      vault_id=vault.id,
  )
  print(validation.status)  # "valid", "invalid", or "unknown"
  ```

  ```typescript TypeScript
  const validation = await client.beta.vaults.credentials.mcpOAuthValidate(
    credential.id,
    { vault_id: vault.id },
  );
  console.log(validation.status); // "valid", "invalid", or "unknown"
  ```

  ```csharp C#
  var validation = await client.Beta.Vaults.Credentials.McpOAuthValidate(credential.ID, new()
  {
      VaultID = vault.ID,
  });
  Console.WriteLine(validation.Status.Raw()); // "valid", "invalid", or "unknown"
  ```

  ```go Go
  validation, err := client.Beta.Vaults.Credentials.MCPOAuthValidate(ctx, credential.ID, anthropic.BetaVaultCredentialMCPOAuthValidateParams{
  	VaultID: vault.ID,
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(validation.Status) // "valid", "invalid", or "unknown"
  ```

  ```java Java
  var validation = client.beta().vaults().credentials().mcpOAuthValidate(credential.id(),
      CredentialMcpOAuthValidateParams.builder()
          .vaultId(vault.id())
          .build());
  IO.println(validation.status()); // valid, invalid, or unknown
  ```

  ```php PHP
  $validation = $client->beta->vaults->credentials->mcpOAuthValidate(
      $credential->id,
      vaultID: $vault->id,
  );
  echo $validation->status . "\n"; // "valid", "invalid", or "unknown"
  ```

  ```ruby Ruby
  validation = client.beta.vaults.credentials.mcp_oauth_validate(
    credential.id,
    vault_id: vault.id
  )
  puts validation.status # :valid, :invalid, or :unknown
  ```
</CodeGroup>

响应是一个 `vault_credential_validation` 对象。`mcp_probe` 包含失败的 MCP 握手步骤；`refresh` 包含所尝试刷新的结果。

```json
{
  "type": "vault_credential_validation",
  "credential_id": "vcrd_01ABC...",
  "vault_id": "vlt_01XYZ...",
  "validated_at": "2026-04-29T17:12:00Z",
  "has_refresh_token": false,
  "status": "invalid",
  "mcp_probe": {
    "method": "initialize",
    "http_response": {
      "status_code": 401,
      "content_type": "application/json",
      "body": "{\"error\":\"invalid_token\"}",
      "body_truncated": false
    }
  },
  "refresh": {
    "status": "no_refresh_token",
    "http_response": null
  }
}
```

## 其他操作

* **列出 vault 或凭证：** 分页返回，最新的在前。默认排除已归档的记录（传递 `include_archived=true` 可将其包含在内）。
* **归档 vault：** `POST /v1/vaults/{id}/archive`。级联到所有凭证。密钥会被清除；记录会保留以供审计。之后引用此 vault 的会话将失败；正在运行的会话会继续。
* **归档凭证：** `POST /v1/vaults/{id}/credentials/{cred_id}/archive`。清除密钥负载；凭证键（`mcp_server_url` 或 `secret_name`）仍然可见，并被释放以供替换凭证使用。
* **删除 vault 或凭证：** 硬删除。记录不会保留。如果您需要审计记录，请使用归档。
