---
title: 在 AWS 上使用 WIF
url: https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/aws
description: 使用 Workload Identity Federation 和 STS 签发的身份令牌，将 Lambda、EC2、ECS 或 EKS 上的 AWS 工作负载认证到 Claude API。
---

AWS 工作负载可以通过交换 AWS 签名的 OIDC 身份令牌来向 Claude API 进行身份认证，而无需使用静态 API 密钥。推荐的路径是调用 AWS STS [`GetWebIdentityToken`](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetWebIdentityToken.html) API，该 API 在工作负载拥有 AWS 凭证的任何地方都可以使用：Lambda、EC2、ECS 和 EKS。EKS 工作负载也可以选择使用 [Kubernetes 投射令牌路径](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/aws#use-eks-projected-service-account-tokens)，该路径配置步骤更少，但只能在 pod 内部使用。

本指南展示了这两种路径。有关底层概念（服务账户、联合签发者和联合规则），请参阅 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)。

## 前提条件

* 熟悉 [WIF 概念](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#concepts)：服务账户（service accounts）、联合签发者（federation issuers）和联合规则（federation rules）。
* 一个附加了 IAM 角色的 AWS 工作负载（EKS pod、ECS 任务、Lambda 函数或 EC2 实例）。
* 工作负载中可用的 `aws` CLI 或 AWS SDK。
* 在 Claude Console 中为您的 Anthropic 组织创建服务账户、联合签发者和联合规则的权限。

## 使用 STS Web 身份令牌（推荐）

AWS STS `GetWebIdentityToken` API 返回一个由 AWS 签名的 OIDC 令牌，用于断言调用者的 IAM 身份。由于它使用工作负载的环境 AWS 凭证，因此同一集成可覆盖 Lambda、EC2、ECS 和 EKS。

### 配置 AWS

<Steps>
  <Step title="为账户启用出站 Web 身份联合">
    这是一个账户级别的标志，默认关闭。在 AWS 控制台中，打开 **IAM**，选择 **Account settings**，然后启用 **Outbound web identity federation**。要以编程方式启用它：

    ```bash
    python3 -c "import boto3; boto3.client('iam').enable_outbound_web_identity_federation()"
    ```

    如果未启用此功能，对 `GetWebIdentityToken` 的调用将失败并返回 `OutboundWebIdentityFederationDisabledException`。
  </Step>

  <Step title="授予工作负载的 IAM 角色调用该 API 的权限">
    将此策略附加到您的 Lambda 函数、EC2 实例或 ECS 任务所使用的 IAM 角色：

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": ["sts:GetWebIdentityToken"],
          "Resource": "*"
        }
      ]
    }
    ```
  </Step>

  <Step title="查找您账户的 STS 签发者 URL">
    启用出站联合后，**IAM > Account settings** 页面会显示一个 **Get Token Issuer URL** 字段，其值的形式为 `https://<uuid>.tokens.sts.global.api.aws`。此 URL 对您的 AWS 账户是唯一的；请复制它以供下一步使用。要以编程方式获取它：

    ```bash
    python3 -c "import boto3; print(boto3.client('iam').get_outbound_web_identity_federation_info())"
    ```
  </Step>
</Steps>

### 配置 Anthropic

在 Claude Console 中，打开 **Settings → Workload identity**，点击 **Connect workload**，然后选择 **AWS** 磁贴。向导将引导您完成注册签发者、创建服务账户和创建联合规则的过程。

向导会为您创建这些资源。无论您是在向导中输入这些值，还是将它们发送到 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api)，都请使用以下值：

**联合签发者：** 注册您在上一步中复制的每账户 STS 签发者 URL。它公开了一个公共 JWKS 端点，因此请使用发现模式（discovery mode）。

```json
{
  "name": "aws-sts",
  "issuer_url": "https://<uuid>.tokens.sts.global.api.aws",
  "jwks": { "type": "discovery" }
}
```

**联合规则：** 匹配您传递给 `GetWebIdentityToken` 的受众（audience）以及 `sub` 声明中调用角色的 IAM 角色 ARN。`sub` 值是调用该 API 的工作负载的 IAM 角色 ARN，形式为 `arn:aws:iam::<account>:role/<role-name>`。该令牌还携带一个 `https://sts.amazonaws.com/` 声明，其中包含 `aws_account`、`org_id`、`principal_id` 以及您传递的任何 `request_tags`；您可以使用规则的 `claims` 映射或 CEL `condition` 对这些内容进行匹配，以实现更精细的控制。

```json
{
  "name": "prod-inference",
  "issuer_id": "fdis_...",
  "match": {
    "subject_prefix": "arn:aws:iam::123456789012:role/inference-worker",
    "audience": "https://api.anthropic.com"
  },
  "target": { "type": "service_account", "service_account_id": "svac_..." },
  "workspace_id": "wrkspc_...",
  "oauth_scope": "workspace:developer",
  "token_lifetime_seconds": 600
}
```

请在工作负载允许的范围内尽可能具体。匹配确切的角色 ARN，并且仅当多个 IAM 角色应映射到同一个 Anthropic 服务账户时，才放宽 `subject_prefix`（例如，放宽为 `arn:aws:iam::123456789012:role/*`）。

### 获取并使用令牌

以 `https://api.anthropic.com` 作为受众调用 `GetWebIdentityToken`，然后将结果传递给 SDK 的联合凭证。令牌提供者是一个可调用对象，因此 SDK 会在每次刷新时重新调用 STS。

<Note>
  `GetWebIdentityToken` 仅在区域性 STS 端点上可用。如果您收到 `'STS' object has no attribute 'get_web_identity_token'` 或类似错误，请将您的 STS 客户端固定到某个区域（例如，`boto3.client("sts", region_name="us-east-1")`），并确保您的 AWS SDK 版本足够新以包含该 API。
</Note>

<CodeGroup>
  ```bash cURL
  JWT=$(aws sts get-web-identity-token \
    --region us-east-1 \
    --audience "https://api.anthropic.com" \
    --signing-algorithm RS256 \
    --duration-seconds 900 \
    --query WebIdentityToken --output text)

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
      "messages": [{"role": "user", "content": "Hello from AWS"}]
    }' | jq -r '.content[] | select(.type == "text") | .text'
  ```

  ```python Python
  import os

  import anthropic
  import boto3
  from anthropic import WorkloadIdentityCredentials


  def get_sts_web_identity_token() -> str:
      sts = boto3.client("sts", region_name="us-east-1")
      resp = sts.get_web_identity_token(
          Audience=["https://api.anthropic.com"],
          SigningAlgorithm="RS256",
          DurationSeconds=900,
      )
      return resp["WebIdentityToken"]


  client = anthropic.Anthropic(
      credentials=WorkloadIdentityCredentials(
          identity_token_provider=get_sts_web_identity_token,
          federation_rule_id=os.environ["ANTHROPIC_FEDERATION_RULE_ID"],
          organization_id=os.environ["ANTHROPIC_ORGANIZATION_ID"],
          service_account_id=os.environ["ANTHROPIC_SERVICE_ACCOUNT_ID"],
          workspace_id=os.environ.get("ANTHROPIC_WORKSPACE_ID"),
      ),
  )

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello from AWS"}],
  )
  print(next(block.text for block in message.content if block.type == "text"))
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";
  import { oidcFederationProvider } from "@anthropic-ai/sdk/lib/credentials/oidc-federation";
  import { STSClient, GetWebIdentityTokenCommand } from "@aws-sdk/client-sts";

  const sts = new STSClient({ region: "us-east-1" });

  async function getStsWebIdentityToken(): Promise<string> {
    const out = await sts.send(
      new GetWebIdentityTokenCommand({
        Audience: ["https://api.anthropic.com"],
        SigningAlgorithm: "RS256",
        DurationSeconds: 900
      })
    );
    return out.WebIdentityToken!;
  }

  const client = new Anthropic({
    credentials: oidcFederationProvider({
      identityTokenProvider: getStsWebIdentityToken,
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
    messages: [{ role: "user", content: "Hello from AWS" }]
  });
  for (const block of message.content) {
    if (block.type === "text") {
      console.log(block.text);
    }
  }
  ```

  ```go Go
  ctx := context.TODO()
  cfg, err := config.LoadDefaultConfig(ctx, config.WithRegion("us-east-1"))
  if err != nil {
  	panic(err)
  }
  stsClient := sts.NewFromConfig(cfg)

  getStsToken := option.IdentityTokenFunc(func(ctx context.Context) (string, error) {
  	out, err := stsClient.GetWebIdentityToken(ctx, &sts.GetWebIdentityTokenInput{
  		Audience:         []string{"https://api.anthropic.com"},
  		SigningAlgorithm: "RS256",
  		DurationSeconds:  aws.Int32(900),
  	})
  	if err != nil {
  		return "", err
  	}
  	return *out.WebIdentityToken, nil
  })

  client := anthropic.NewClient(
  	option.WithFederationTokenProvider(getStsToken, option.FederationOptions{
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
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Hello from AWS")),
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
  StsClient sts = StsClient.builder().region(Region.US_EAST_1).build();

  IdentityTokenProvider getStsToken = () -> sts.getWebIdentityToken(
                  GetWebIdentityTokenRequest.builder()
                          .audience("https://api.anthropic.com")
                          .signingAlgorithm("RS256")
                          .durationSeconds(900)
                          .build())
          .webIdentityToken();

  AnthropicClient client = AnthropicOkHttpClient.builder()
          .federationTokenProvider(
                  getStsToken,
                  System.getenv("ANTHROPIC_FEDERATION_RULE_ID"),
                  System.getenv("ANTHROPIC_ORGANIZATION_ID"),
                  System.getenv("ANTHROPIC_SERVICE_ACCOUNT_ID"))
          .build();

  var message = client.messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessage("Hello from AWS")
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
      IdentityTokenProvider = new StsTokenProvider(),
  });
  using var client = new AnthropicOidcClient(credentials);

  var message = await client.Messages.Create(new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello from AWS" }],
  });
  foreach (var block in message.Content)
  {
      if (block.Value is TextBlock textBlock)
      {
          Console.WriteLine(textBlock.Text);
      }
  }

  class StsTokenProvider : IIdentityTokenProvider
  {
      private readonly AmazonSecurityTokenServiceClient _sts = new(Amazon.RegionEndpoint.USEast1);

      public async Task<string> GetIdentityTokenAsync(CancellationToken ct = default)
      {
          var resp = await _sts.GetWebIdentityTokenAsync(new GetWebIdentityTokenRequest
          {
              Audience = ["https://api.anthropic.com"],
              SigningAlgorithm = "RS256",
              DurationSeconds = 900,
          }, ct);
          return resp.WebIdentityToken;
      }
  }
  ```

  ```bash CLI
  TOKEN_FILE=$(mktemp)
  aws sts get-web-identity-token \
    --region us-east-1 \
    --audience "https://api.anthropic.com" \
    --signing-algorithm RS256 \
    --duration-seconds 900 \
    --query WebIdentityToken --output text > "$TOKEN_FILE"

  export ANTHROPIC_IDENTITY_TOKEN_FILE="$TOKEN_FILE"
  # ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  # ANTHROPIC_SERVICE_ACCOUNT_ID 和 ANTHROPIC_WORKSPACE_ID 从环境变量中读取
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello from AWS"}'
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Credentials\WorkloadIdentityCredentials;
  use Aws\Sts\StsClient;

  $sts = new StsClient(['region' => 'us-east-1', 'version' => 'latest']);
  $client = new Client(credentials: new WorkloadIdentityCredentials(
      identityTokenProvider: fn() => $sts->getWebIdentityToken([
          'Audience' => ['https://api.anthropic.com'],
          'SigningAlgorithm' => 'RS256',
          'DurationSeconds' => 900,
      ])['WebIdentityToken'],
      federationRuleId: getenv('ANTHROPIC_FEDERATION_RULE_ID'),
      organizationId: getenv('ANTHROPIC_ORGANIZATION_ID'),
      serviceAccountId: getenv('ANTHROPIC_SERVICE_ACCOUNT_ID'),
      workspaceId: getenv('ANTHROPIC_WORKSPACE_ID') ?: null,
  ));

  $message = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello from AWS']],
  );
  $textBlock = array_find($message->content, static fn ($block): bool => $block->type === 'text');
  echo $textBlock->text, PHP_EOL;
  ```

  ```ruby Ruby
  require "anthropic"
  require "aws-sdk-sts"

  sts = Aws::STS::Client.new(region: "us-east-1")
  client = Anthropic::Client.new(
    credentials: Anthropic::WorkloadIdentityCredentials.new(
      identity_token_provider: -> {
        sts.get_web_identity_token(
          audience: ["https://api.anthropic.com"],
          signing_algorithm: "RS256",
          duration_seconds: 900,
        ).web_identity_token
      },
      federation_rule_id: ENV.fetch("ANTHROPIC_FEDERATION_RULE_ID"),
      organization_id: ENV.fetch("ANTHROPIC_ORGANIZATION_ID"),
      service_account_id: ENV.fetch("ANTHROPIC_SERVICE_ACCOUNT_ID"),
      workspace_id: ENV["ANTHROPIC_WORKSPACE_ID"],
    ),
  )

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello from AWS"}]
  )
  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

### 验证设置

在工作负载内部，直接交换 STS 签发的令牌并检查响应：

```bash cURL
JWT=$(aws sts get-web-identity-token \
  --region us-east-1 \
  --audience "https://api.anthropic.com" \
  --signing-algorithm RS256 \
  --duration-seconds 900 \
  --query WebIdentityToken --output text)

curl -sS https://api.anthropic.com/v1/oauth/token \
  -H "content-type: application/json" \
  -d "{
    \"grant_type\": \"urn:ietf:params:oauth:grant-type:jwt-bearer\",
    \"assertion\": \"$JWT\",
    \"federation_rule_id\": \"fdrl_...\",
    \"organization_id\": \"00000000-0000-0000-0000-000000000000\",
    \"service_account_id\": \"svac_...\",
    \"workspace_id\": \"wrkspc_...\"
  }" | jq
```

成功的交换会返回一个以 `sk-ant-oat01-` 开头的 `access_token` 和一个以秒为单位的 `expires_in` 值。如果交换失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`），请查看[认证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)以了解拒绝原因，并参阅[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)；AWS 端最常见的原因是 `iss` 不匹配（每账户 STS 签发者 URL 必须与注册的 `issuer_url` 完全匹配）。

## 使用 EKS 投射服务账户令牌

如果您的工作负载运行在 EKS pod 中，您可以跳过 STS 调用，直接从磁盘读取 Kubernetes 投射的服务账户令牌。Kubernetes 原生地将一个兼容 OIDC 的令牌投射到 pod 中，SDK 可以从文件路径读取它，因此不需要令牌提供者可调用对象。与 STS 路径相比，此路径少了两个 AWS 配置步骤，但只能在 pod 内部使用；其底层机制与[通用 Kubernetes 集成](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/kubernetes)相同。

此路径还需要一个[启用了 IAM OIDC 提供者](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)的 EKS 集群，以及对该集群的 `kubectl` 访问权限。

### 配置您的 EKS 集群

<Steps>
  <Step title="查找您集群的 OIDC 签发者 URL">
    每个 EKS 集群都有一个唯一的 OIDC 签发者。使用 AWS CLI 获取它：

    ```bash CLI
    aws eks describe-cluster \
      --name <cluster-name> \
      --query "cluster.identity.oidc.issuer" \
      --output text
    ```

    输出类似于 `https://oidc.eks.us-west-2.amazonaws.com/id/6FA42E7BFDE8549CB...`。您将在下一节中将此 URL 注册为联合签发者。
  </Step>

  <Step title="创建服务账户并投射一个 Anthropic 受众令牌">
    EKS pod 身份 webhook 会检测 `eks.amazonaws.com/role-arn` 注解，并自动投射一个 `aud: sts.amazonaws.com` 的令牌，将其路径公开为 `AWS_WEB_IDENTITY_TOKEN_FILE`。该令牌用于 AWS 角色代入。对于 Anthropic 交换，请投射第二个 `audience: https://api.anthropic.com` 的令牌，并将其挂载到专用路径。

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: inference-worker
      namespace: inference
      annotations:
        eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/inference-worker
    ```

    ```yaml
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
  </Step>

  <Step title="注意令牌的声明结构">
    投射的令牌是由您集群的 OIDC 签发者签名的 JSON Web Token（JWT）。其 `sub` 声明遵循 Kubernetes 约定 `system:serviceaccount:<namespace>:<service-account-name>`：

    ```json
    {
      "iss": "https://oidc.eks.us-west-2.amazonaws.com/id/6FA42E7BFDE8549CB...",
      "sub": "system:serviceaccount:inference:inference-worker",
      "aud": ["https://api.anthropic.com"],
      "kubernetes.io": {
        "namespace": "inference",
        "serviceaccount": { "name": "inference-worker", "uid": "..." }
      },
      "exp": 1775527120,
      "iat": 1775523520
    }
    ```

    `serviceAccountToken` 投射将 `aud` 设置为 `https://api.anthropic.com`。位于 `AWS_WEB_IDENTITY_TOKEN_FILE` 的另一个由 IRSA 注入的令牌携带 `aud: sts.amazonaws.com`，用于 AWS API 调用，而非此交换。
  </Step>
</Steps>

### 配置 Anthropic

在 Claude Console 中，打开 **Settings → Workload identity**，点击 **Connect workload**，然后选择 **AWS** 磁贴。向导将引导您完成注册签发者、创建服务账户和创建联合规则的过程。

向导会为您创建这些资源。无论您是在向导中输入这些值，还是将它们发送到 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api)，都请使用以下值：

**联合签发者：** EKS 签发者公开了一个公共 JWKS 端点，因此请使用发现模式。签发者 URL 必须与令牌的 `iss` 声明完全匹配。每个集群注册一个签发者。

```json
{
  "name": "prod-eks-uswest2",
  "issuer_url": "https://oidc.eks.us-west-2.amazonaws.com/id/6FA42E7BFDE8549CB...",
  "jwks": { "type": "discovery" }
}
```

**联合规则：** 匹配 Kubernetes `sub` 声明和 Anthropic 受众 `https://api.anthropic.com`。（请投射一个具有该受众的专用服务账户令牌；不要重用 IRSA 默认的 `sts.amazonaws.com` 令牌。）

```json
{
  "name": "prod-inference",
  "issuer_id": "fdis_...",
  "match": {
    "subject_prefix": "system:serviceaccount:inference:inference-worker",
    "audience": "https://api.anthropic.com"
  },
  "target": { "type": "service_account", "service_account_id": "svac_..." },
  "workspace_id": "wrkspc_...",
  "oauth_scope": "workspace:developer",
  "token_lifetime_seconds": 600
}
```

请在工作负载允许的范围内尽可能具体。仅当命名空间中的每个服务账户都应映射到同一个 Anthropic 服务账户时，才将 `subject_prefix` 放宽为 `system:serviceaccount:inference:*`（末尾的 `*` 使其成为前缀匹配）。

### 获取并使用令牌

在 pod 内部，投射的令牌位于 `/var/run/secrets/anthropic.com/token`（在 Pod 规范中公开为 `ANTHROPIC_IDENTITY_TOKEN_FILE`）。将该文件传递给 SDK 的联合凭证，SDK 会处理交换和刷新。

<CodeGroup>
  ```bash cURL
  JWT=$(cat "$ANTHROPIC_IDENTITY_TOKEN_FILE")

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
      "messages": [{"role": "user", "content": "Hello from EKS"}]
    }' | jq -r '.content[] | select(.type == "text") | .text'
  ```

  ```python Python
  import os

  import anthropic
  from anthropic import IdentityTokenFile, WorkloadIdentityCredentials

  client = anthropic.Anthropic(
      credentials=WorkloadIdentityCredentials(
          identity_token_provider=IdentityTokenFile(
              os.environ["ANTHROPIC_IDENTITY_TOKEN_FILE"]
          ),
          federation_rule_id=os.environ["ANTHROPIC_FEDERATION_RULE_ID"],
          organization_id=os.environ["ANTHROPIC_ORGANIZATION_ID"],
          service_account_id=os.environ["ANTHROPIC_SERVICE_ACCOUNT_ID"],
          workspace_id=os.environ.get("ANTHROPIC_WORKSPACE_ID"),
      ),
  )

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello from EKS"}],
  )
  print(next(block.text for block in message.content if block.type == "text"))
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";
  import { oidcFederationProvider } from "@anthropic-ai/sdk/lib/credentials/oidc-federation";
  import { identityTokenFromFile } from "@anthropic-ai/sdk/lib/credentials/identity-token";

  const client = new Anthropic({
    credentials: oidcFederationProvider({
      identityTokenProvider: identityTokenFromFile(process.env.ANTHROPIC_IDENTITY_TOKEN_FILE!),
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
    messages: [{ role: "user", content: "Hello from EKS" }]
  });
  for (const block of message.content) {
    if (block.type === "text") {
      console.log(block.text);
    }
  }
  ```

  ```go Go
  tokenPath := os.Getenv("ANTHROPIC_IDENTITY_TOKEN_FILE")

  readToken := option.IdentityTokenFunc(func(ctx context.Context) (string, error) {
  	raw, err := os.ReadFile(tokenPath)
  	if err != nil {
  		return "", fmt.Errorf("read identity token: %w", err)
  	}
  	return string(raw), nil
  })

  client := anthropic.NewClient(
  	option.WithFederationTokenProvider(readToken, option.FederationOptions{
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
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Hello from EKS")),
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
          .addUserMessage("Hello from EKS")
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
      Messages = [new() { Role = Role.User, Content = "Hello from EKS" }],
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
    --message '{role: user, content: "Hello from EKS"}'
  ```

  ```php PHP
  use Anthropic\Client;

  // 读取 ANTHROPIC_FEDERATION_RULE_ID、ANTHROPIC_ORGANIZATION_ID、
  // ANTHROPIC_SERVICE_ACCOUNT_ID、ANTHROPIC_WORKSPACE_ID 和 ANTHROPIC_IDENTITY_TOKEN_FILE
  $client = new Client();

  $message = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello from EKS']],
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
    messages: [{role: "user", content: "Hello from EKS"}]
  )
  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

<Tip>
  Pod 规范已经设置了 `ANTHROPIC_IDENTITY_TOKEN_FILE`、`ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_SERVICE_ACCOUNT_ID` 和 `ANTHROPIC_WORKSPACE_ID`，因此您可以不带任何参数构造客户端，SDK 会自动读取联合环境变量。
</Tip>

### 验证设置

在 pod 内部，直接交换投射的令牌并检查响应：

```bash cURL
JWT=$(cat "$ANTHROPIC_IDENTITY_TOKEN_FILE")

curl -sS https://api.anthropic.com/v1/oauth/token \
  -H "content-type: application/json" \
  -d "{
    \"grant_type\": \"urn:ietf:params:oauth:grant-type:jwt-bearer\",
    \"assertion\": \"$JWT\",
    \"federation_rule_id\": \"$ANTHROPIC_FEDERATION_RULE_ID\",
    \"organization_id\": \"$ANTHROPIC_ORGANIZATION_ID\",
    \"service_account_id\": \"$ANTHROPIC_SERVICE_ACCOUNT_ID\",
    \"workspace_id\": \"$ANTHROPIC_WORKSPACE_ID\"
  }" | jq
```

成功的交换会返回一个以 `sk-ant-oat01-` 开头的 `access_token` 和一个以秒为单位的 `expires_in` 值。如果交换失败并返回不透明的 `401` `authentication_error` 响应（消息为 `Authentication failed`），请查看[认证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)以了解拒绝原因，并参阅[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)；EKS 端最常见的原因是投射令牌的 `aud` 与规则不匹配（请投射一个 `audience: https://api.anthropic.com` 的令牌，而不是 IRSA 默认的 `sts.amazonaws.com`）。

## 限定规则范围

<Warning>
  `subject_prefix` 为 `arn:aws:iam::123456789012:role/*` 会匹配该账户中的每个 IAM 角色。任何能够代入任一匹配角色的主体都可以获取联合的 Anthropic 令牌。
</Warning>

将规则的 `match` 块锁定到适合您用例的最窄范围：

* **固定完整的角色 ARN：** 使用 `subject_prefix: "arn:aws:iam::<account>:role/<role-name>"`，末尾不带 `*`，这样账户中的其他角色就不会匹配。
* **固定账户 ID：** 使用 `claims` 映射或 CEL `condition` 匹配令牌 `https://sts.amazonaws.com/` 声明中的 `aws_account` 字段，作为针对前缀配置错误的纵深防御检查。
* **在 EKS 上固定命名空间和服务账户：** 使用确切的 `system:serviceaccount:<namespace>:<name>` 值，在 `system:serviceaccount:` 前缀之后不带 `*`。
* **为每个环境使用单独的规则：** 为生产、预发布和开发工作负载创建不同的规则，而不是放宽一个前缀来覆盖所有环境。

## 后续步骤

* 查看 [WIF 参考](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference)，了解完整的凭证优先级、配置文件配置和规则匹配参考。
* 对于不在 EKS 上的自管理 Kubernetes 集群，请参阅[在 Kubernetes 上使用 WIF](https://platform.claude.com/docs/zh-CN/manage-claude/wif-providers/kubernetes)。
