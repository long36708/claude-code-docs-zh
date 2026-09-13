---
title: 在 Kubernetes 中使用 WIF
url: https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/kubernetes
description: 使用投射的服务账户令牌，从自管理的 Kubernetes 集群向 Claude API 进行身份验证。
---

自管理的 Kubernetes 集群（kubeadm、k3s、OpenShift 以及本地部署发行版）通过 [projected service account tokens](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection)（投射的服务账户令牌）为每个 pod 签发 OIDC "JSON Web Token"（JSON Web 令牌），即 JWT。集群的 API 服务器充当 OIDC issuer（颁发者），每个令牌的 `sub` 声明遵循 `system:serviceaccount:<namespace>:<service-account>` 的形式。您可以通过读取集群的发现文档来找到集群的颁发者 URL：

```bash cURL
kubectl get --raw /.well-known/openid-configuration | jq -r .issuer
```

<Note>
  本页介绍的机制（投射的服务账户令牌，集群 API 服务器作为 OIDC 颁发者）是 Kubernetes 本身原生的，因此它是所有 Kubernetes 发行版的基础。如果您运行在托管的 Kubernetes 服务上，云提供商指南会介绍在哪里找到由提供商管理的颁发者 URL：[AWS (EKS)](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/aws#use-eks-projected-service-account-tokens)、[Google Cloud (GKE)](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/gcp) 或 [Azure (AKS)](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/azure)。如果您的集群运行 SPIRE，则颁发者是 SPIRE OIDC Discovery Provider 而非集群 API 服务器；请参阅 [SPIFFE](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/spiffe)。对于任何其他发行版或未在此列出的托管提供商，请遵循本指南并使用您的集群报告的颁发者 URL。
</Note>

## 前提条件

* 熟悉 [WIF 概念](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#concepts)：服务账户、联合颁发者和联合规则。

* 一个在 API 服务器上配置了 [`--service-account-issuer`](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) 标志的 Kubernetes 集群。大多数发行版默认会设置此项；kubeadm 集群通常使用 `https://kubernetes.default.svc.cluster.local`。如果您无法直接访问 API 服务器配置，您的平台团队可以确认该值。

* 满足以下条件之一，以便 Anthropic 能够验证令牌签名：

  * 颁发者的 JWKS 端点可通过 HTTPS 在 443 端口从公共互联网访问，或者
  * 您可以从集群内部获取 JWKS 并以 `inline` 模式注册它（在[配置 Anthropic](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/kubernetes#configure-anthropic) 中介绍）。

* 拥有在 Claude Console 中为您的 Anthropic 组织创建服务账户、联合颁发者和联合规则的权限。

## 配置 Kubernetes

将服务账户令牌投射到您的 pod 中，并使用您的联合规则所期望的受众和生命周期。`serviceAccountToken` 投射会将一个新的 JWT 写入挂载路径，并在 `expirationSeconds` 到期之前对其进行轮换。

```yaml Pod
apiVersion: v1
kind: Pod
metadata:
  name: inference-worker
  namespace: inference
spec:
  serviceAccountName: inference-worker
  volumes:
    - name: anthropic-token
      projected:
        sources:
          - serviceAccountToken:
              audience: https://api.anthropic.com
              expirationSeconds: 3600
              path: token
  containers:
    - name: app
      image: your-registry/inference-worker:latest
      env:
        - name: ANTHROPIC_IDENTITY_TOKEN_FILE
          value: /var/run/secrets/anthropic.com/token
        - name: ANTHROPIC_FEDERATION_RULE_ID
          value: fdrl_...
        - name: ANTHROPIC_ORGANIZATION_ID
          value: 00000000-0000-0000-0000-000000000000
        - name: ANTHROPIC_SERVICE_ACCOUNT_ID
          value: svac_...
        - name: ANTHROPIC_WORKSPACE_ID  # required when the rule covers multiple workspaces
          value: wrkspc_...
      volumeMounts:
        - name: anthropic-token
          mountPath: /var/run/secrets/anthropic.com
          readOnly: true
```

为此 pod 签发的令牌携带 `sub: "system:serviceaccount:inference:inference-worker"` 和 `aud: ["https://api.anthropic.com"]`。

## 配置 Anthropic

在 Claude Console 中，打开 **Settings → Workload identity**，点击 **Connect workload**，然后选择 **Kubernetes** 磁贴。向导会引导您完成注册颁发者、创建服务账户以及创建联合规则的步骤。

向导会为您创建这些资源。无论您是在向导中输入这些值，还是将它们发送到 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api)，都请使用以下值：

**联合颁发者：** 许多自管理集群使用诸如 `https://kubernetes.default.svc.cluster.local` 之类无法从公共互联网访问的颁发者 URL。如果您的集群属于这种情况，请选择 **inline** JWKS 来源并粘贴集群的密钥。从集群内部获取它们：

```bash cURL
kubectl get --raw /openid/v1/jwks
```

然后使用返回的 `keys` 数组的内容（而不是外层的 `{"keys": [...]}` 包装）来配置颁发者：

```json
{
  "name": "onprem-k8s",
  "issuer_url": "https://kubernetes.default.svc.cluster.local",
  "jwks": {
    "type": "inline",
    "keys": [{ "kty": "RSA", "kid": "...", "n": "...", "e": "AQAB" }]
  }
}
```

在 `inline` 模式下，`issuer_url` 仅用于与 JWT 的 `iss` 声明进行比较；Anthropic 从不尝试访问它。如果您的颁发者可公开访问，请改用 `"jwks": {"type": "discovery"}`。

<Warning>
  使用 `inline` 密钥时，当集群轮换其服务账户签名密钥时，您需要负责更新颁发者。轮换很少发生（通常仅在集群升级期间），但在您推送新的 JWKS 之前，令牌交换会因签名错误而失败。
</Warning>

**联合规则：** 匹配服务账户的 `sub` 声明以及您在投射令牌上设置的受众。

```json
{
  "name": "onprem-inference",
  "issuer_id": "fdis_...",
  "match": {
    "subject_prefix": "system:serviceaccount:inference:inference-worker",
    "audience": "https://api.anthropic.com"
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

请在工作负载允许的范围内尽可能具体。仅当命名空间中的每个服务账户都应映射到同一个 Anthropic 服务账户时，才将 `subject_prefix` 放宽为 `system:serviceaccount:inference:*`（末尾的 `*` 使其成为前缀匹配）。将规则的 `fdrl_...` ID 添加到您 pod 的 `ANTHROPIC_FEDERATION_RULE_ID` 环境变量中。

## 获取并使用令牌

[配置 Kubernetes](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/kubernetes#configure-kubernetes) 中的 pod 规约将 `ANTHROPIC_IDENTITY_TOKEN_FILE` 设置为投射的挂载路径，同时还设置了 `ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_SERVICE_ACCOUNT_ID` 和 `ANTHROPIC_WORKSPACE_ID`。有了这些设置，SDK 会在每次交换时从磁盘读取令牌，并自动刷新 Anthropic 访问令牌。

<CodeGroup>
  ```bash cURL
  JWT=$(cat "$ANTHROPIC_IDENTITY_TOKEN_FILE")

  ACCESS_TOKEN=$(curl -sS https://api.anthropic.com/v1/oauth/token \
    -H "content-type: application/json" \
    --data @- <<JSON | jq -r .access_token
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

  # 从 pod 的环境中读取 ANTHROPIC_IDENTITY_TOKEN_FILE、ANTHROPIC_FEDERATION_RULE_ID、
  # ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID
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

  // 从 pod 的环境中读取 ANTHROPIC_IDENTITY_TOKEN_FILE、ANTHROPIC_FEDERATION_RULE_ID、
  // ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID
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
  // 从 pod 的环境中读取 ANTHROPIC_IDENTITY_TOKEN_FILE、ANTHROPIC_FEDERATION_RULE_ID、
  // ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID
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
  # 读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  # ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello, Claude"}'
  ```

  ```php PHP
  use Anthropic\Client;

  // 读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  // ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
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

  # 读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  # ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}]
  )
  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

## 验证设置

成功的交换会返回一个以 `sk-ant-oat01-` 开头的 `access_token` 以及一个以秒为单位的 `expires_in` 值。如果交换失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`），请查看[身份验证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)以了解拒绝原因，并参阅[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)；Kubernetes 端最常见的原因是 JWKS 密钥不匹配（对于 `inline` 模式，请使用 `kubectl get --raw /openid/v1/jwks` 重新获取并更新颁发者）。

## 限定规则范围

<Warning>
  `subject_prefix` 为 `system:serviceaccount:*` 会匹配集群中的每个服务账户，因此任何 pod 都可以获取联合的 Anthropic 令牌。如果没有 `audience` 匹配器，该规则还会匹配集群的默认受众令牌，而每个 pod 都已经投射了这些令牌。
</Warning>

将规则的 `match` 块锁定到适合您用例的最小范围：

* **固定命名空间和服务账户名称：** 使用完整的 `system:serviceaccount:<namespace>:<name>` 值，末尾不带 `*`。
* **始终设置受众：** 在规则上要求 `audience`，并在 pod 的 `serviceAccountToken` 投射上设置相同的值，以便拒绝默认受众令牌。
* **每个命名空间使用单独的规则：** 为每个命名空间创建不同的规则和 Anthropic 服务账户，而不是扩大一条规则的范围。
* **将 inline-JWKS 颁发者限定到单个集群：** 当多个集群共享一个颁发者 URL 时，将每个集群的 JWKS 注册为各自的联合颁发者，并仅将规则绑定到该颁发者。

## 后续步骤

* [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)：概念、令牌交换流程以及 SDK 配置选项。
* [WIF 参考](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference)：环境变量、JWKS 来源模式以及规则匹配模式。
