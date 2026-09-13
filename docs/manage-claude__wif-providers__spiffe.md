---
title: 将 WIF 与 SPIFFE 配合使用
url: https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/spiffe
description: 使用来自 SPIRE 或任何其他符合 SPIFFE 规范的颁发者的 JWT-SVID，向 Claude API 验证 SPIFFE 工作负载的身份。
---

[SPIFFE](https://spiffe.io/) 是 CNCF 为工作负载颁发身份的标准。[SPIRE](https://spiffe.io/docs/latest/spire-about/) 是其开源参考实现，此外还有多款商业产品也会颁发符合 SPIFFE 规范的身份。Anthropic 可与任何能够发出 OIDC 兼容 JWT-SVID 的 SPIFFE 实现进行联合。有关当前实现的列表，请参阅 SPIFFE 项目网站上的 [Commercial software that implements SPIFFE](https://spiffe.io/docs/latest/spiffe-about/overview/#commercial-software-that-implements-spiffe)。

联合可以通过位于公共 HTTPS URL 的 OIDC 发现文档进行（`discovery` 模式，受 [URL 约束](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#url-fields)限制），也可以通过直接注册 JWKS 进行（`inline` 模式）。

JWT-SVID 规范将 `sub` 定义为工作负载的 SPIFFE ID，而 SPIFFE Workload API 要求调用方在获取时提供 `aud`，因此这些声明在各个实现之间是一致的。Anthropic 还额外要求 `iss` 和 `iat`，这两者都不是 JWT-SVID 规范强制要求的，因此请配置您的实现以填充这两个声明（在 SPIRE 中，`iss` 是 `jwt_issuer` 服务器设置，`iat` 会自动设置）。具备这些条件后，本指南的[配置 Anthropic](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/spiffe#configure-anthropic)、[获取并使用令牌](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/spiffe#acquire-and-use-the-token)以及[限定规则范围](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/spiffe#scope-your-rule)部分适用于任何 SPIFFE 实现。

SPIFFE 为每个工作负载分配一个形式为 `spiffe://<trust-domain>/<path>` 的稳定身份 URI，SPIRE 则通过 Workload API 按需将该身份颁发为 JWT-SVID。JWT-SVID 是一个普通的已签名 JWT，其 `sub` 声明是工作负载的 SPIFFE ID，其 `aud` 声明由工作负载在获取时提供。

从 SPIRE 信任域到标准 OIDC 的桥梁是 [SPIRE OIDC Discovery Provider](https://github.com/spiffe/spire/blob/main/support/oidc-discovery-provider/README.md)，这是一个独立的辅助程序，为信任域的 JWT 签名密钥发布 `/.well-known/openid-configuration` 和 JWKS 端点。在发现提供程序运行的情况下，JWT-SVID 的验证方式与任何其他 OIDC 令牌相同：将发现 URL 注册为联合颁发者，编写一条与工作负载的 SPIFFE ID 匹配的联合规则，然后让工作负载将其 JWT-SVID 提交给 Anthropic 的令牌交换端点。

本页的示例使用 SPIRE，适用于任何运行 SPIRE Agent 的地方：Kubernetes pod、虚拟机和裸金属主机。

<Note>
  如果您的 Kubernetes 集群未运行 SPIRE，而您希望改用集群原生的投射服务账户令牌进行身份验证，请参阅[将 WIF 与 Kubernetes 配合使用](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/kubernetes)。
</Note>

## 前提条件

* 熟悉 [WIF 概念](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#concepts)：服务账户、联合颁发者和联合规则。
* 一个已颁发工作负载身份的 SPIFFE 部署（本页示例使用 SPIRE Server 和 Agent），以及需要调用 Claude API 的工作负载的注册条目。
* 信任域的 OIDC 发现端点（在 SPIRE 中为 [OIDC Discovery Provider](https://github.com/spiffe/spire/blob/main/support/oidc-discovery-provider/README.md)），以可公开访问的 HTTPS 端点运行，或者已导出 JWKS 以用于 `inline` 注册。
* 您的 SPIFFE 颁发者已配置为将 JWT-SVID 上的 `iss` 声明设置为您将注册为联合颁发者 `issuer_url` 的值。对于 `discovery` 模式，这是发现端点的公共 URL（在 SPIRE 中为 `jwt_issuer` 服务器设置）。
* 您的工作负载可以获取 JWT-SVID。WIF 仅接受 JWT-SVID，不接受 X.509-SVID。
* 有权在 Claude Console 中为您的 Anthropic 组织创建服务账户、联合颁发者和联合规则。

获取 JWT-SVID 时请求的受众（audience）值始终为 `https://api.anthropic.com`。请在 spiffe-helper 的 `jwt_audience`、Workload API 的 `FetchJWTSVID` 调用以及联合规则的 `audience` 匹配器中使用此值。

## 配置 SPIRE

本节中的说明特定于 SPIRE。如果您使用其他 SPIFFE 颁发者，请根据其自身文档配置其 OIDC 发现端点和 JWT-SVID 获取方式，然后继续阅读[配置 Anthropic](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/spiffe#configure-anthropic)。

如果您已经在运行带有 OIDC Discovery Provider 的 SPIRE，那么与 Anthropic 联合在 SPIRE 端需要三件事：一个与发现 URL 匹配的 `jwt_issuer`、一个将调用 Claude API 的工作负载的注册条目，以及一种让该工作负载获取带有 Anthropic 受众的 JWT-SVID 的方式。以下各小节将逐一介绍。配置片段仅显示与 Anthropic 联合相关的设置，而非完整的 SPIRE 部署配置。

<Tip>
  首次设置 SPIRE？请按照 [SPIRE 快速入门](https://spiffe.io/docs/latest/try/)部署 SPIRE Server 和 Agent，然后将 [OIDC Discovery Provider](https://github.com/spiffe/spire/blob/main/support/oidc-discovery-provider/README.md) 作为独立服务与 SPIRE Server 一起添加。发现模式联合依赖于该提供程序已部署且可公开访问。该提供程序不属于默认 SPIRE 安装的一部分。
</Tip>

### 验证 JWT 颁发者

Anthropic 通过将 JWT-SVID 的 `iss` 声明与已注册的联合颁发者进行匹配，并从该颁发者的发现文档中获取 JWKS 来验证 JWT-SVID。两个 SPIRE 设置必须使用同一个 URL：SPIRE Server 的 `jwt_issuer`（它会成为每个签发的 JWT-SVID 中的 `iss` 声明）和 OIDC Discovery Provider 的 `domains` 列表（它决定发现文档和 JWKS 从哪个主机提供）。这个共享的 URL 就是您向 Anthropic 注册的内容。

信任域和颁发者 URL 是相互独立的。信任域（`spiffe://prod.example.com`）限定 `sub` 声明的范围。颁发者 URL（`https://oidc-discovery.prod.example.com`）是 Anthropic 获取签名密钥的位置。它们不需要共享主机名。

确认 SPIRE Server 的配置中已设置 `jwt_issuer`，并指向发现提供程序的公共 URL。以下示例还显示了默认的 JWT-SVID 生命周期。SPIRE 的内置默认值为 5 分钟，这足够短，因此需要持续轮换（请参阅[运行 spiffe-helper](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/spiffe#run-spiffe-helper)）。Anthropic 的令牌交换端点会拒绝任何生命周期超过联合颁发者所配置最大值的身份令牌，该最大值默认为 1 小时（请参阅[验证规则](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#validation-rules)）。此检查适用于所有 SPIFFE 实现，而不仅限于 SPIRE，因此请将 `default_jwt_svid_ttl`（或任何按条目的覆盖值）保持在该最大值或以下。

```text server.conf
server {
    trust_domain         = "prod.example.com"
    jwt_issuer           = "https://oidc-discovery.prod.example.com"
    default_jwt_svid_ttl = "5m"
    # ...
}
```

在 OIDC Discovery Provider 的配置中，相同的主机名必须出现在 `domains` 下，并且该提供程序必须能够访问 SPIRE Server 的 API 套接字。该提供程序通过 HTTPS 提供发现文档和 JWKS。可使用其内置的 ACME 支持终止 TLS，或在其前面放置一个负责终止 TLS 的负载均衡器。

```text oidc-discovery-provider.conf
domains = ["oidc-discovery.prod.example.com"]

server_api {
    address = "unix:///run/spire/sockets/private/api.sock"
}

acme {
    email        = "platform@example.com"
    tos_accepted = true
}
```

<Note>
  该示例使用 `server_api`，它将发现提供程序连接到 SPIRE Server 的特权 API 套接字。该提供程序还接受一个 `workload_api` 块（包含 `socket_path` 和 `trust_domain`），改为通过 SPIRE Agent 的 Workload API 获取信任包。当发现提供程序不应访问 Server API，或运行在无法访问 Server 的节点上时，请使用此方式。
</Note>

### 注册工作负载

每个调用 Claude API 的工作负载都需要一个 SPIRE 注册条目，将其运行时选择器映射到一个 SPIFFE ID。如果工作负载已经注册，请记下其 SPIFFE ID，您将在联合规则的 `subject_prefix` 中使用它。如果尚未注册，请进行注册。对于 Kubernetes pod，选择器通常是命名空间和 Kubernetes 服务账户：

```bash CLI
# 将 NODE_UID 替换为节点的 UID：
#   kubectl get node <node-name> -o jsonpath='{.metadata.uid}'
spire-server entry create \
    -spiffeID spiffe://prod.example.com/ns/inference/sa/worker \
    -parentID spiffe://prod.example.com/spire/agent/k8s_psat/prod-cluster/NODE_UID \
    -selector k8s:ns:inference \
    -selector k8s:sa:worker
```

<Note>
  所示的 `parentID` 是单个节点自动生成的 agent ID。对于集群范围的注册，请将条目的父级设为[节点别名](https://spiffe.io/docs/latest/deploying/registering/#mapping-workloads-to-multiple-nodes)，以便它匹配每个节点上的工作负载，正如 [SPIRE Kubernetes 快速入门](https://spiffe.io/docs/latest/try/getting-started-k8s/)所做的那样。
</Note>

Kubernetes 之外的工作负载使用主机级选择器，例如 `unix:uid:1000`（`unix:path` 也可用，但需要在 agent 的 unix 工作负载证明器配置中设置 `discover_workload_path = true`）。运行 [spire-controller-manager](https://github.com/spiffe/spire-controller-manager) 的集群可以使用 `ClusterSPIFFEID` 自定义资源声明条目，而无需直接调用 `spire-server entry create`。

### 运行 spiffe-helper

[spiffe-helper](https://github.com/spiffe/spiffe-helper) 是一个 sidecar 实用程序，它连接到 SPIRE Agent 套接字，为给定受众获取 JWT-SVID，将其写入文件，并在过期前重新获取。该辅助程序默认以守护进程模式运行。以下示例显式设置了 `daemon_mode = true`。

```text helper.conf
agent_address = "/run/spire/sockets/agent.sock"
# The JWT-SVID file is written under cert_dir
cert_dir      = "/var/run/secrets/anthropic.com"
daemon_mode   = true

jwt_svids = [{
    jwt_audience       = "https://api.anthropic.com"
    jwt_svid_file_name = "token"
}]
```

在 Kubernetes 中，将 spiffe-helper 作为 sidecar 容器运行，与您的应用容器共享一个基于内存的 `emptyDir` 卷（`medium: Memory`），这样持有者 SVID 永远不会落到节点的磁盘上。将 SPIRE Agent 套接字从主机挂载到 sidecar 中，在两个容器中将共享卷挂载到 `/var/run/secrets/anthropic.com`，并在应用容器上设置 `ANTHROPIC_IDENTITY_TOKEN_FILE=/var/run/secrets/anthropic.com/token`。在虚拟机和裸金属上，将 spiffe-helper 作为系统服务与工作负载一起运行，并让两者指向一个共享目录。

## 配置 Anthropic

在 Claude Console 中，打开 **Settings → Workload identity**，点击 **Connect workload**，然后选择 **Custom OIDC**。向导将引导您完成注册颁发者、创建服务账户和创建联合规则的过程。

向导会为您创建这些资源。无论您是在向导中输入这些值，还是将它们发送到 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api)，都请使用以下值：

**联合颁发者：** 以 `discovery` 模式注册 OIDC Discovery Provider 的公共 URL。Anthropic 从此 URL 获取 `/.well-known/openid-configuration`，并跟随返回的 `jwks_uri` 获取信任域的签名密钥。

```json
{
  "name": "spire-prod",
  "issuer_url": "https://oidc-discovery.prod.example.com",
  "jwks": { "type": "discovery" }
}
```

如果发现提供程序无法从公共互联网访问，请自行获取 JWKS（`curl https://oidc-discovery.prod.example.com/keys`），并使用返回的 `keys` 数组的内容，以 `"jwks": {"type": "inline", "keys": [...]}` 注册颁发者。在 `inline` 模式下，`issuer_url` 仅用于与 JWT-SVID 的 `iss` 声明进行比较。Anthropic 永远不会尝试访问它。

<Warning>
  SPIRE 会频繁轮换 JWT 签名密钥，默认与 CA 的节奏相同（`ca_ttl`，24 小时）。如果您使用内联 JWKS 而非发现 URL 注册颁发者，则必须在 SPIRE 每次轮换时更新 JWKS：在工作负载开始提交新密钥之前添加新密钥，并在使用旧密钥签名的令牌过期后**移除已被取代的密钥**。留在内联 JWKS 中的过时密钥将无限期地保持受信任状态。
</Warning>

要在不暴露公共发现端点的情况下自动更新 JWKS，请配置一个 SPIRE Server [BundlePublisher](https://spiffe.io/docs/latest/deploying/spire_server/#built-in-plugins) 插件（`aws_s3`、`gcp_cloudstorage` 或 `k8s_configmap`），并设置 `format = "jwks"`，以便在每次轮换时将 JWT 签名密钥推送到外部存储，然后通过 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api#federation-issuers) 更新颁发者的内联密钥。

**联合规则：** 匹配 JWT-SVID 的 `sub`（SPIFFE ID）以及您配置 spiffe-helper 请求的 `aud`。SPIFFE ID 是 URI 字符串，`subject_prefix` 将其作为不透明文本进行匹配，因此精确值或带尾部 `*` 的前缀匹配都适用。对于更复杂的模式，请使用 CEL `condition`。

```json
{
  "name": "spire-inference-worker",
  "issuer_id": "fdis_...",
  "match": {
    "subject_prefix": "spiffe://prod.example.com/ns/inference/sa/worker",
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

`token_lifetime_seconds` 是交换返回的 Anthropic 访问令牌的生命周期，而不是 JWT-SVID 的生命周期。SDK 会自动刷新访问令牌。

请在工作负载允许的范围内尽可能具体。仅当在该路径下注册的每个工作负载都应映射到同一个 Anthropic 服务账户时，才将 `subject_prefix` 放宽为 `spiffe://prod.example.com/ns/inference/*`。将规则的 `fdrl_...` ID 添加到工作负载的 `ANTHROPIC_FEDERATION_RULE_ID` 环境变量中。

## 获取并使用令牌

Anthropic SDK 既可以从 spiffe-helper 维护的文件中读取 JWT-SVID，也可以通过令牌提供程序可调用对象直接调用 SPIFFE Workload API。文件路径方式是最简单的集成方式，适用于所有 SDK 语言。可调用对象方式省去了 sidecar，但需要您的应用语言中有 SPIFFE Workload API 客户端。

<Tabs>
  <Tab title="基于文件，使用 spiffe-helper">
    在 spiffe-helper 将新的 JWT-SVID 写入 `/var/run/secrets/anthropic.com/token` 的情况下，将 `ANTHROPIC_IDENTITY_TOKEN_FILE` 设置为该路径，同时设置 `ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_SERVICE_ACCOUNT_ID` 和 `ANTHROPIC_WORKSPACE_ID`。SDK 在每次令牌交换时读取该文件，因此它始终会获取最近轮换的 SVID，并在 Anthropic 访问令牌过期前自动刷新。有关每个值的来源，请参阅[环境变量](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#environment-variables)。

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

      ```bash CLI
      # 读取 spiffe-helper 写入到
      # ANTHROPIC_IDENTITY_TOKEN_FILE 的 JWT-SVID，以及 ANTHROPIC_FEDERATION_RULE_ID、
      # ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID。
      ant messages create \
        --model claude-opus-5 \
        --max-tokens 1024 \
        --message '{role: user, content: "Hello, Claude"}'
      ```

      ```python Python
      import anthropic

      # 读取 spiffe-helper 写入到
      # ANTHROPIC_IDENTITY_TOKEN_FILE 的 JWT-SVID，以及 ANTHROPIC_FEDERATION_RULE_ID、
      # ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID。
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

      // 读取 spiffe-helper 写入到
      // ANTHROPIC_IDENTITY_TOKEN_FILE 的 JWT-SVID，以及 ANTHROPIC_FEDERATION_RULE_ID、
      // ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID。
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

      ```csharp C#
      // 读取 spiffe-helper 写入到
      // ANTHROPIC_IDENTITY_TOKEN_FILE 的 JWT-SVID，以及 ANTHROPIC_FEDERATION_RULE_ID、
      // ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID。
      using var client = new AnthropicClient();

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

      ```go Go
      // 读取 spiffe-helper 写入到
      // ANTHROPIC_IDENTITY_TOKEN_FILE 的 JWT-SVID，以及 ANTHROPIC_FEDERATION_RULE_ID、
      // ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID。
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
      // 读取 spiffe-helper 写入到
      // ANTHROPIC_IDENTITY_TOKEN_FILE 的 JWT-SVID，以及 ANTHROPIC_FEDERATION_RULE_ID、
      // ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID。
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var message = client.messages().create(MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5)
              .maxTokens(1024)
              .addUserMessage("Hello, Claude")
              .build());

      IO.println(message.content());
      ```

      ```php PHP
      use Anthropic\Client;

      // 读取 spiffe-helper 写入到
      // ANTHROPIC_IDENTITY_TOKEN_FILE 的 JWT-SVID，以及 ANTHROPIC_FEDERATION_RULE_ID、
      // ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID。
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

      # 读取 spiffe-helper 写入到
      # ANTHROPIC_IDENTITY_TOKEN_FILE 的 JWT-SVID，以及 ANTHROPIC_FEDERATION_RULE_ID、
      # ANTHROPIC_ORGANIZATION_ID、ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID。
      client = Anthropic::Client.new

      message = client.messages.create(
        model: "claude-opus-5",
        max_tokens: 1024,
        messages: [{role: "user", content: "Hello, Claude"}]
      )
      puts message.content.find { it.type == :text }.text
      ```
    </CodeGroup>
  </Tab>

  <Tab title="通过 SPIFFE Workload API 的可调用对象">
    直接链接 SPIFFE Workload API 客户端的工作负载可以跳过 spiffe-helper，并向 SDK 传递一个从 agent 套接字获取新 JWT-SVID 的可调用对象。SDK 在每次令牌交换之前调用该可调用对象，因此工作负载始终提交未过期的 SVID。Python（[py-spiffe](https://github.com/HewlettPackard/py-spiffe)）和 Go（[go-spiffe](https://github.com/spiffe/go-spiffe)）拥有成熟的 Workload API 客户端。

    <CodeGroup exclude="shell, typescript, java, csharp, php, ruby">
      ```python Python
      import os
      import anthropic
      from anthropic import WorkloadIdentityCredentials
      from spiffe import JwtSource

      AUDIENCE = "https://api.anthropic.com"

      # 连接到位于 SPIFFE_ENDPOINT_SOCKET 的 SPIRE Agent 套接字。
      jwt_source = JwtSource()


      def fetch_jwt_svid() -> str:
          svid = jwt_source.fetch_svid(audience={AUDIENCE})  # audience is a set of strings
          return svid.token


      client = anthropic.Anthropic(
          credentials=WorkloadIdentityCredentials(
              identity_token_provider=fetch_jwt_svid,
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

      ```go Go
      import (
      	"context"
      	"fmt"
      	"os"

      	"github.com/anthropics/anthropic-sdk-go"
      	"github.com/anthropics/anthropic-sdk-go/option"
      	"github.com/spiffe/go-spiffe/v2/svid/jwtsvid"
      	"github.com/spiffe/go-spiffe/v2/workloadapi"
      )
      // ...
      	const audience = "https://api.anthropic.com"

      	ctx := context.Background()
      	source, err := workloadapi.NewJWTSource(ctx)
      	if err != nil {
      		panic(err)
      	}
      	defer source.Close()

      	fetchJWTSVID := func(ctx context.Context) (string, error) {
      		svid, err := source.FetchJWTSVID(ctx, jwtsvid.Params{Audience: audience})
      		if err != nil {
      			return "", err
      		}
      		return svid.Marshal(), nil
      	}

      	client := anthropic.NewClient(
      		option.WithFederationTokenProvider(fetchJWTSVID, option.FederationOptions{
      			FederationRuleID: os.Getenv("ANTHROPIC_FEDERATION_RULE_ID"),
      			OrganizationID:   os.Getenv("ANTHROPIC_ORGANIZATION_ID"),
      			ServiceAccountID: os.Getenv("ANTHROPIC_SERVICE_ACCOUNT_ID"),
      			WorkspaceID:      os.Getenv("ANTHROPIC_WORKSPACE_ID"),
      		}),
      	)

      	message, err := client.Messages.New(ctx, anthropic.MessageNewParams{
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
    </CodeGroup>

    <Note>
      对于其他语言，请使用您运行时的 SPIFFE Workload API 客户端获取 JWT-SVID（或通过 shell 调用 `spire-agent api fetch jwt`），将其写入文件，并像基于文件的选项卡中那样将 `ANTHROPIC_IDENTITY_TOKEN_FILE` 设置为该路径。
    </Note>
  </Tab>
</Tabs>

## 验证设置

在接入 SDK 之前，直接从 SPIRE Agent 获取一个 JWT-SVID，并确认其声明与您的联合规则所期望的一致。如果您使用其他 SPIFFE 实现，请使用其 CLI 或 Workload API 客户端获取 JWT-SVID，并以相同方式解码负载。

<Note>
  Workload API 会对调用进程进行证明。对于 Kubernetes 注册条目，请在满足该条目选择器且已挂载 agent 套接字的 pod 内运行此命令（例如使用 `kubectl exec`）。在虚拟机和裸金属上，请以与该条目的 `unix:` 选择器匹配的用户或进程身份运行。从未经证明的主机 shell 运行会返回 `no identity issued`，这是验证步骤中最常见的失败原因。
</Note>

```bash CLI
spire-agent api fetch jwt \
    -audience https://api.anthropic.com \
    -socketPath /run/spire/sockets/agent.sock \
    -output json \
  | jq -r '.[0].svids[0].svid' \
  | jq -rR 'split(".")[1] | gsub("-";"+") | gsub("_";"/") | @base64d | fromjson'
```

`-output json` 标志将 SVID 响应和信任包响应作为一个包含两个元素的 JSON 数组返回，因此 `jq -r '.[0].svids[0].svid'` 可提取出裸令牌。在没有 `-output` 的旧版 SPIRE 上，该命令会改为打印一个带标签的块。在这种情况下，请将默认输出通过管道传给 `awk '/^[[:space:]]*eyJ/{print $1; exit}'` 以提取令牌行。检查 `iss` 是否为您注册的 OIDC Discovery Provider URL，`sub` 是否为工作负载的 SPIFFE ID，以及 `aud` 是否包含 `https://api.anthropic.com`。然后运行[获取并使用令牌](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/spiffe#acquire-and-use-the-token)中的 cURL 示例。成功的交换会返回一个以 `sk-ant-oat01-` 开头的 `access_token`。如果交换失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`），请查看[身份验证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)了解拒绝原因，并参阅[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)。SPIRE 端最常见的原因是 SPIRE Server 的 `jwt_issuer` 与注册为联合颁发者的 URL 不匹配。

## 限定规则范围

SPIFFE ID 路径约定由运维人员定义，因此联合规则的 `subject_prefix` 匹配器应反映您的注册条目所使用的路径方案。常见方案包括 `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`（spire-controller-manager 中 `ClusterSPIFFEID` 资源发出的默认值）以及用于虚拟机和裸金属工作负载的 `spiffe://<trust-domain>/host/<hostname>/<service>`。

<Warning>
  `subject_prefix` 为 `spiffe://prod.example.com/*` 会匹配信任域中的每个工作负载。如果没有 `audience` 匹配器，该规则还会接受为任何受众签发的 JWT-SVID，包括工作负载为不相关的依赖方请求的那些。
</Warning>

将规则的 `match` 块锁定到适合您用例的最窄范围：

* **固定到单个工作负载：** 将 `subject_prefix` 设置为完整的 SPIFFE ID，不带尾部 `*`。
* **始终设置受众：** 在规则上要求 `audience`，并为 spiffe-helper（或 Workload API 调用）配置相同的值，以便拒绝为其他依赖方签发的 SVID。
* **按路径段限定范围：** 使用 `spiffe://prod.example.com/ns/inference/*` 授权在某个命名空间下注册的每个工作负载，并为每个命名空间创建单独的规则和 Anthropic 服务账户，而不是放宽一条规则。
* **每个信任域一个颁发者：** 每个 SPIRE 信任域都有自己的签名密钥和 OIDC Discovery Provider。将每个信任域注册为单独的联合颁发者，并将规则绑定到拥有其所匹配 SPIFFE ID 的颁发者。

## 后续步骤

<CardGroup cols={2}>
  <Card title="将 WIF 与 Okta 配合使用" icon="lock" href="https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/okta">
    使用工作负载身份联合将 Okta 服务应用身份联合到 Claude API。
  </Card>

  <Card title="工作负载身份联合" icon="cloud" href="https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation">
    使用来自您自己的身份提供商的短期身份令牌（而非长期静态 API 密钥）向 Claude API 验证工作负载的身份。
  </Card>

  <Card title="WIF 参考" icon="book" href="https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference">
    工作负载身份联合的环境变量、验证规则、配置文件配置和错误参考。
  </Card>

  <Card title="将 WIF 与 Kubernetes 配合使用" icon="cube" href="https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/kubernetes">
    使用投射服务账户令牌从自管理的 Kubernetes 集群向 Claude API 进行身份验证。
  </Card>
</CardGroup>
