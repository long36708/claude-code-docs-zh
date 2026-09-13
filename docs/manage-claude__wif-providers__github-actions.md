---
title: 在 GitHub Actions 中使用 WIF
url: https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/github-actions
description: 使用短期身份令牌而非长期 API 密钥，将 GitHub Actions 工作流认证到 Claude API。
---

每次 GitHub Actions 工作流运行都可以从 GitHub 托管的颁发者 `https://token.actions.githubusercontent.com` 请求一个已签名的身份令牌。借助 "Workload Identity Federation"（工作负载身份联合），即 WIF，您的工作流可以将该令牌交换为一个短期的 Anthropic 访问令牌，这样您的 CI 作业就可以调用 Claude API，而无需在仓库中存储 `ANTHROPIC_API_KEY` 密钥。

令牌的 `sub` 声明编码了仓库和触发上下文。对于推送到某个分支的情况，其格式为 `repo:<owner>/<repo>:ref:refs/heads/<branch>`。拉取请求运行使用 `repo:<owner>/<repo>:pull_request`，而受环境门控的部署使用 `repo:<owner>/<repo>:environment:<name>`。您的联合规则会针对此声明（以及其他声明，例如 `repository_owner` 和 `ref`）进行匹配，以决定允许哪些工作流运行进行认证。

## 前提条件

* 熟悉 [WIF 概念](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#concepts)：服务账户、联合颁发者和联合规则。
* 一个您可以编辑工作流文件并授予 `id-token: write` 权限的 GitHub 仓库。
* 在 Claude Console 中为您的 Anthropic 组织创建服务账户、联合颁发者和联合规则的权限。
* 您的 Anthropic 组织 ID。您可以在 Claude Console 的 **Settings → Organization** 下找到它。

## 配置您的工作流

GitHub 仅向明确请求身份令牌的作业颁发身份令牌。在工作流或作业级别添加 `id-token: write` 权限：

```yaml
permissions:
  id-token: write
  contents: read
```

在作业内部，运行器会暴露两个环境变量：`ACTIONS_ID_TOKEN_REQUEST_URL` 和 `ACTIONS_ID_TOKEN_REQUEST_TOKEN`。以请求令牌作为 bearer 凭据、以您选择的 audience 作为查询参数调用请求 URL，然后将返回的 "JSON Web Token"（JSON Web 令牌），即 JWT 写入文件：

```yaml
- name: Fetch GitHub OIDC token
  run: |
    curl -sS -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=https://api.anthropic.com" \
      | jq -r .value > /tmp/gha-jwt
```

如果您更喜欢 JavaScript，`actions/github-script` 通过 `core.getIDToken(audience)` 提供了相同的功能：

```yaml
- name: Fetch GitHub OIDC token
  uses: actions/github-script@v8
  with:
    script: |
      const fs = require('fs');
      const token = await core.getIDToken('https://api.anthropic.com');
      fs.writeFileSync('/tmp/gha-jwt', token);
```

解码后的令牌携带描述该工作流运行的声明。您的联合规则会针对这些声明进行匹配：

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "sub": "repo:your-org/your-repo:ref:refs/heads/main",
  "aud": "https://api.anthropic.com",
  "repository": "your-org/your-repo",
  "repository_owner": "your-org",
  "ref": "refs/heads/main",
  "sha": "abc123...",
  "workflow": "CI",
  "actor": "octocat",
  "event_name": "push"
}
```

有关 `sub` 格式的完整列表，请参阅 [GitHub 的 OIDC subject 声明参考](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#example-subject-claims)。

## 配置 Anthropic

在 Claude Console 中，打开 **Settings → Workload identity**，点击 **Connect workload**，然后选择 **GitHub Actions** 磁贴。向导会引导您完成注册颁发者、创建服务账户以及创建联合规则的步骤。

向导会为您创建这些资源。无论您是在向导中输入这些值，还是将它们发送到 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api)，都请使用以下值：

**联合颁发者：** GitHub 公开发布其 OIDC 发现文档和 JWKS，因此请使用发现模式。当 GitHub 轮换密钥时，Anthropic 会自动刷新这些密钥。

```json
{
  "name": "github-actions",
  "issuer_url": "https://token.actions.githubusercontent.com",
  "jwks": { "type": "discovery" }
}
```

**联合规则：** 仅匹配您打算信任的工作流运行。有关如何安全地限定这些声明的范围，请参阅[限制哪些工作流可以进行认证](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/github-actions#restrict-which-workflows-can-authenticate)。

```json
{
  "name": "gha-main",
  "issuer_id": "fdis_...",
  "match": {
    "subject_prefix": "repo:your-org/your-repo:ref:refs/heads/main",
    "audience": "https://api.anthropic.com",
    "claims": {
      "repository_owner": "your-org"
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

请在工作负载允许的范围内尽可能具体。仅当规则必须匹配来自同一仓库的多种事件类型时，才将 `subject_prefix` 放宽为 `repo:your-org/your-repo:*`（并搭配 `claims.ref` 约束），因为 `sub` 的末尾段在 `ref:...`、`environment:...` 和 `pull_request` 事件之间各不相同。

## 获取并使用令牌

在作业上设置联合环境变量，然后正常调用 SDK。`Anthropic()` 会读取 `ANTHROPIC_IDENTITY_TOKEN_FILE`，在第一次请求时交换 JWT，并在访问令牌过期前自动刷新。

<CodeGroup>
  ```yaml Workflow
  name: Call Claude
  on: push

  permissions:
    id-token: write
    contents: read

  jobs:
    call-claude:
      runs-on: ubuntu-latest
      env:
        ANTHROPIC_FEDERATION_RULE_ID: fdrl_...
        ANTHROPIC_ORGANIZATION_ID: 00000000-0000-0000-0000-000000000000
        ANTHROPIC_SERVICE_ACCOUNT_ID: svac_...
        ANTHROPIC_WORKSPACE_ID: wrkspc_...  # required when the rule covers multiple workspaces
        ANTHROPIC_IDENTITY_TOKEN_FILE: /tmp/gha-jwt
      steps:
        - uses: actions/checkout@v5
        - name: Fetch GitHub OIDC token
          run: |
            curl -sS -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
              "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=https://api.anthropic.com" \
              | jq -r .value > "$ANTHROPIC_IDENTITY_TOKEN_FILE"
        - name: Run your script
          run: |
            pip install anthropic
            python your_script.py
  ```

  ```bash cURL
  JWT=$(cat /tmp/gha-jwt)

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

  curl https://api.anthropic.com/v1/messages \
    -H "authorization: Bearer $ACCESS_TOKEN" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello, Claude"}]
    }' | jq -r '.content[] | select(.type == "text") | .text'
  ```

  ```python Python
  import anthropic

  # 从作业环境中读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  # ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
  # 这些环境变量。
  client = anthropic.Anthropic()

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello, Claude"}],
  )
  print(next(block.text for block in message.content if block.type == "text"))
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";

  // 从作业环境中读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  // ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
  // 这些环境变量。
  const client = new Anthropic();

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
  // 从作业环境中读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  // ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
  // 这些环境变量。
  client := anthropic.NewClient()

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
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var message = client.messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessage("Hello, Claude")
          .build());

  IO.println(message.content());
  ```

  ```csharp C#
  var result = AnthropicCredentials.Resolve()
      ?? throw new InvalidOperationException("No federation credentials found in environment");
  using var client = new AnthropicOidcClient(result);

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
  ```

  ```bash CLI
  # 从作业环境中读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  # ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
  # 这些环境变量。
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello, Claude"}'
  ```

  ```php PHP
  use Anthropic\Client;

  // 从作业环境中读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  // ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
  // 这些环境变量。
  $client = new Client();

  $message = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello, Claude']],
  );
  $textBlock = array_find($message->content, static fn ($block): bool => $block->type === 'text');
  echo $textBlock->text, PHP_EOL;
  ```

  ```ruby Ruby
  require "anthropic"

  # 从作业环境中读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  # ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
  # （以上均来自作业环境变量）。
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}]
  )
  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

每个由 GitHub 颁发的身份令牌在颁发后大约五分钟过期。令牌请求端点（`ACTIONS_ID_TOKEN_REQUEST_URL`）在整个作业期间保持有效，因此您可以在任何时候获取新的令牌。SDK 在首次使用时交换令牌，并缓存得到的 Anthropic 访问令牌。对于运行时间超过 Anthropic 令牌生命周期的作业，SDK 会在每次刷新时重新读取 `ANTHROPIC_IDENTITY_TOKEN_FILE`，因此请定期重新运行获取步骤（或将其包装在后台循环中）以保持文件为最新。或者，向 SDK 传递一个直接调用 `ACTIONS_ID_TOKEN_REQUEST_URL` 的令牌提供者回调，而不是使用文件路径。

## 验证设置

成功的交换会返回一个以 `sk-ant-oat01-` 开头的 `access_token` 以及一个以秒为单位的 `expires_in` 值。被拒绝的交换会返回一个不透明的 `401` `authentication_error`，其固定消息为 `Authentication failed`，无论是哪项检查失败；在大多数情况下，拒绝原因会记录在[认证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)中该次尝试的条目上，而[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)会按顺序逐项说明这些检查。GitHub Actions 端最常见的原因是 `sub` 声明格式不匹配（其末尾段在 `ref:...`、`environment:...` 和 `pull_request` 事件之间各不相同）；历史条目会显示原因 `match_subject_prefix`。

## 限制哪些工作流可以进行认证

<Warning>
  仅使用 `repo:your-org/*` 作为 `subject_prefix` 会匹配您组织中的每个仓库，并且在没有 `ref` 约束的情况下，它还会匹配从 fork 触发的 `pull_request` 运行。任何能够针对匹配仓库发起拉取请求的人都可能获得联合的 Anthropic 令牌。
</Warning>

将规则的 `match` 块锁定到符合您用例的最小范围：

* **固定到单个仓库：** 使用 `subject_prefix: "repo:your-org/your-repo:*"`，这样组织中的其他仓库就不会匹配。
* **固定到受保护分支：** 在 `claims` 下添加 `"ref": "refs/heads/main"`（或您的发布分支），这样拉取请求运行和功能分支就不会匹配。
* **显式固定所有者：** 在 `claims` 下添加 `"repository_owner": "your-org"`，作为针对 `sub` 解析边界情况的纵深防御检查。
* **固定到部署环境：** 对于部署作业，匹配 `subject_prefix: "repo:your-org/your-repo:environment:production"`，并在 GitHub 中为该环境设置必需的审阅者进行门控。

## 后续步骤

* [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)：完整的设置演练、环境变量和凭据优先级。
* [认证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)：联合与 API 密钥的比较。
