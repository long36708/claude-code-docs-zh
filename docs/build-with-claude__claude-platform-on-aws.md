---
title: Claude Platform on AWS
url: https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws
description: 通过 AWS 访问 Claude 的完整平台功能，基础设施由 Anthropic 管理。
---

Claude Platform on AWS 为您提供完整的 Anthropic 平台体验，包括 Messages API、Agent Skills、代码执行和测试版功能，均可通过您的 AWS 账户访问。与由 AWS 运营推理栈的 [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 不同，Claude Platform on AWS 由 Anthropic 运营。AWS 提供身份验证层（SigV4 或 API 密钥）、基于 IAM 的访问控制，以及通过 AWS Marketplace 的计费集成。

<Note>
  Anthropic SDK 支持 Claude Platform on AWS。
</Note>

## 平台集成的工作原理

Claude 模型运行在 Anthropic 管理的基础设施上。这是一种通过 AWS 进行计费和访问的商业集成。Anthropic 是推理输入和输出的数据处理方。AWS 在 Marketplace 模式下处理计费和身份元数据。通过 Claude Platform on AWS 使用 Claude 的客户须遵守 Anthropic 的[数据使用条款](https://www.anthropic.com/legal)。

Claude Platform on AWS 具有以下运营特征：数据可能不驻留在 AWS 中，推理可能路由到 Anthropic 的主云，子服务可能在不另行通知的情况下发生变化。您可以按请求设置 [`inference_geo`](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#data-residency) 参数，将推理固定到特定地理区域。

Claude Platform on AWS 遵循与第一方 Claude API 相同的数据保留政策。"Zero Data Retention"（零数据保留），即 ZDR，可按需提供。请联系您的 Anthropic 客户代表为您的组织启用该功能。

## Claude Platform on AWS 与 Amazon Bedrock 的对比

这两种产品都允许您通过 AWS 使用 Claude，但它们在架构、API 接口和功能可用性方面有所不同。

| 方面               | Claude Platform on AWS                                                                                                                     | [Amazon Bedrock 中的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) | [Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy) |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **谁运营该栈**        | Anthropic                                                                                                                                  | AWS                                                                                                           | AWS                                                                                                                        |
| **API 接口**       | Claude API（`/v1/{endpoint}`）                                                                                                               | 位于 `/anthropic/v1/messages` 的 Messages API                                                                    | Bedrock Converse / InvokeModel                                                                                             |
| **功能可用性**        | 通常与 Claude API 同日提供（参见[功能限制](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#features-not-supported)）      | 依照 Amazon Bedrock 发布计划                                                                                        | 依照 Amazon Bedrock 发布计划                                                                                                     |
| **Agent Skills** | 可用（测试版）                                                                                                                                    | 不可用（需要代码执行）                                                                                                   | 不可用                                                                                                                        |
| **测试版功能**        | 通过 `anthropic-beta` 标头透传（参见[功能限制](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#features-not-supported)） | 不支持 `anthropic-beta` 标头                                                                                       | 不支持 `anthropic-beta` 标头                                                                                                    |
| **身份验证**         | AWS IAM / SigV4 或 API 密钥                                                                                                                   | AWS IAM / SigV4                                                                                               | AWS IAM / SigV4 或 bearer 令牌                                                                                                |
| **计费**           | AWS Marketplace                                                                                                                            | AWS（原生服务）                                                                                                     | AWS（原生服务）                                                                                                                  |
| **基础 URL**       | `aws-external-anthropic.{region}.api.aws`                                                                                                  | `bedrock-mantle.{region}.api.aws`                                                                             | `bedrock-runtime.{region}.amazonaws.com`                                                                                   |
| **SDK 客户端**      | 平台专用客户端类（例如 Python 中的 `AnthropicAWS`），测试版                                                                                                  | `AnthropicBedrockMantle`                                                                                      | `AnthropicBedrock` / Bedrock SDK                                                                                           |
| **控制台**          | Claude Console（`platform.claude.com`，通过 AWS Console 访问）                                                                                    | Bedrock Console                                                                                               | Bedrock Console                                                                                                            |
| **速率限制和配额**      | 由 Anthropic 管理                                                                                                                             | 由 AWS 管理                                                                                                      | 由 AWS 管理                                                                                                                   |
| **推理数据处理方**      | Anthropic                                                                                                                                  | AWS                                                                                                           | AWS                                                                                                                        |

如果您需要由 AWS 运营的 Claude，请参阅 [Amazon Bedrock 中的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)。Claude Platform on AWS 使用的容量池独立于第一方 Claude API 和 Amazon Bedrock。您可以在多个平台上运行工作负载，并在它们之间进行故障转移。

支持使用 [AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html) 将您的 VPC 连接到 Claude Platform on AWS 端点。

**何时选择 Bedrock：** 处于受监管行业、需要 FedRAMP High、IL4、IL5 或 HIPAA 就绪合规性的组织，或需要 AWS 作为唯一数据处理方的组织，应使用 [Amazon Bedrock 中的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)。Bedrock 完全运行在 AWS 控制的基础设施上，由 AWS 作为运营方。

**您正在使用哪种产品？** Claude 通过几种不同的产品提供：

* **Claude Platform on AWS**（本页）：通过 AWS Marketplace 计费的 Claude API 平台。在 [Claude Console](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#using-the-claude-console) 和 AWS Console 中管理。
* **[Amazon Bedrock 中的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)：** 一项 AWS 原生服务。在 Amazon Bedrock 控制台中管理，并作为 AWS 服务用量计费。
* **通过 AWS Marketplace 采购的 Claude Enterprise：** 一种 [claude.ai](https://claude.ai) 套餐（Claude 聊天产品），而非 API 平台。在 claude.ai 上管理，其账户和迁移行为与本页所述不同。请参阅 [Claude 帮助中心](https://support.claude.com)。
* **直接 Anthropic 账户：** 由 Anthropic 计费的第一方 Claude API 和 claude.ai 套餐。在 Claude Console 和 claude.ai 上管理。

## 设置您的账户

设置 Claude Platform on AWS 分为四个阶段：在 AWS Console 服务页面上注册、完成 Anthropic 组织设置、记录您的工作区 ID，以及登录 Claude Console。

<Note>
  通过 AWS Console 注册会预配一个与您的 AWS 账户绑定的新 Anthropic 组织。该组织独立于您公司在 Anthropic 已有的任何组织，包括通过 AWS Marketplace 采购的 Claude Enterprise 组织。第一方 Anthropic 组织中的 API 密钥、工作区和 Claude Console 设置不会沿用。

  如果您已有 Amazon Bedrock 私有报价，请在注册前联系您的 Anthropic 或 AWS 客户代表，以便您的折扣从第一个请求起生效。折扣无法追溯应用于私有报价被接受之前产生的用量。请参阅[私有报价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#private-offers)。
</Note>

<Steps>
  <Step title="在 AWS Console 中注册">
    1. 打开 [AWS Console](https://console.aws.amazon.com/) 并导航到 **Claude Platform on AWS** 服务页面。
    2. 选择 **Sign up**。
    3. 在注册页面上，查看条款（Anthropic 的最终用户许可协议、AWS 隐私声明和 AWS 客户协议），并勾选同意复选框。
    4. 选择 **Continue**。

    页面会显示 **Sign-up in progress** 横幅。请停留在该页面。注册需要几分钟时间，期间 AWS 会为您处理 AWS Marketplace 订阅，然后自动重定向。

    如果您的组织拥有来自 Anthropic 的私有报价，Console 会查找该报价并提示您在 AWS Marketplace 中接受。详情请参阅[私有报价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#private-offers)。

    <Note>
      如果您使用 Claude Platform on AWS，您的内容（例如提示和补全）将由 Anthropic 在 AWS 之外处理。有关内容和元数据如何处理和存储的详情，请参阅 Anthropic 的[数据使用政策](https://www.anthropic.com/legal)。
    </Note>
  </Step>

  <Step title="设置您的 Anthropic 组织">
    注册完成后，您将被重定向到 `platform.claude.com/partner-signup`。

    1. 输入您组织所有者的电子邮件地址，然后选择 **Get started**。
    2. 检查该邮箱收件箱中的设置链接并点击进入。如果您的浏览器显示 **Signed in as a different account** 页面，请选择 **Log out and continue**。
    3. 填写组织详情表单（组织名称、实体类型、国家/地区、预期用途），然后选择 **Complete setup**。

    完成设置会创建您的 Anthropic 组织，并接受 Anthropic 的商业服务条款和使用政策。AWS Console 服务页面现在会显示左侧导航栏，包含 **Home**、**API keys**、**Quickstart** 和 **Workspaces**。
  </Step>

  <Step title="创建您的工作区并记录其 ID">
    完成设置后，AWS Console 会提示您创建一个工作区。有关区域绑定、IAM 资源范围限定以及创建其他工作区的详情，请参阅[工作区](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#workspaces)。

    在 AWS Console **Claude Platform on AWS** 服务页面的 **Workspaces** 下或在 [Claude Console](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#using-the-claude-console) 中查找工作区 ID。工作区 ID 的格式为 `wrkspc_` 后跟一个字母数字标识符。
  </Step>

  <Step title="登录 Claude Console">
    对 Claude Console 的访问通过 AWS IAM 进行联合身份验证：

    1. 代入一个具有 `aws-external-anthropic:AssumeConsole` 权限的 IAM 角色。请参阅 [Claude Platform on AWS 的 IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#console-access)。
    2. 在 **Claude Platform on AWS** 服务页面中，选择 **Open Claude Console**。AWS Console 会签发一个 JWT 并将您重定向到 `platform.claude.com`。
    3. 首次登录时，系统会提示您输入电子邮件地址。请输入您的工作邮箱。平台会即时预配您的 Claude Console 用户。

    当您通过 AWS Console 登录时，Claude Console 的范围限定为您的 Claude Platform on AWS 组织。Claude Console 侧边栏左下角会显示 **Account managed by AWS** 指示标识。
  </Step>
</Steps>

### 从现有 Anthropic 组织迁移

注册 Claude Platform on AWS 始终会预配一个与您的 AWS 账户绑定的新 Anthropic 组织。不存在原地转换：现有组织（例如第一方 Claude API 组织）无法变为 Claude Platform on AWS 组织。

请将从现有组织的迁移规划为切换到新组织：

* **先创建新组织。** 通过 AWS Console 注册（参见[设置您的账户](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#set-up-your-account)）。如果您的迁移涉及私有报价，请在报价被接受之前完成注册：折扣从接受时起生效，不可追溯。请参阅[私有报价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#private-offers)。
* **重新创建访问权限和配置。** API 密钥、工作区和 Claude Console 设置不会从现有组织沿用。请在新组织中创建工作区，并将您的应用程序切换到 [Claude Platform on AWS 身份验证](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#authentication)。
* **更新您的集成。** Claude Platform on AWS 提供 Claude API（`/v1/{endpoint}`），因此请求和响应的结构与第一方 Claude API 相同。变化的是基础 URL、身份验证方法以及必需的 `anthropic-workspace-id` 标头；请参阅[发起请求](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#making-requests)。部分平台功能有所不同；请参阅[不支持的功能](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#features-not-supported)。
* **按您自己的节奏切换。** 新组织独立于您的现有组织，两者可以并行处理流量。无需硬切换：逐步转移工作负载，直到所有流量都在新组织上。

新组织运行起来后，差异主要集中在计费和身份验证方面，这些均通过 AWS 处理：

* **计费**转移到 AWS Marketplace：用量以 Claude Consumption Units 而非预付额度计费（参见[计费](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#billing)），您可以在 Billing 页面设置支出限额（参见[支出限额](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#spend-limits)）。过渡期间，计费保持独立：现有组织继续按当前方式计费。
* **身份验证和访问**转移到 AWS：请求使用 AWS 凭证或在 AWS Console（而非 Claude Console）中生成的 API 密钥进行身份验证（参见[身份验证](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#authentication)）。组织成员资格通过 AWS IAM 而非 Claude Console 管理（参见[可用页面](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#available-pages)），Anthropic 的客户端 SDK 提供平台专用客户端类（参见[安装 SDK](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#install-an-sdk)）。
* **日常 API 使用**与第一方 Claude API 的方式相同，[功能限制](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#features-not-supported)中注明的除外。在转移生产流量之前，请检查您的速率限制：新组织被置于 Start 层级，限额提升需通过您的 Anthropic 客户代表办理（参见[速率限制和配额](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#rate-limits-and-quotas)）。

对于行为不同的 Claude Enterprise（claude.ai）组织，请参阅[产品对比](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#claude-platform-on-aws-vs-amazon-bedrock)。

### 账户设置故障排除

* **"Sign-up failed: Failed to enable OutboundWebIdentityFederation"：** 如果您在首次提交时看到此横幅，请再次选择 **Continue**。IAM 启用可能需要片刻才能生效。
* **注册期间没有进度指示器：** 注册需要几分钟时间。在 AWS 预配您的账户期间，页面会显示一个静态的 **Sign-up in progress** 横幅，没有进度条。
* **点击设置链接后显示 "Signed in as a different account"：** 选择 **Log out and continue**。页面会使用您输入的电子邮件地址重新对您进行身份验证。
* **登录期间出现 "Not found" 消息：** 此消息可能在重定向期间短暂出现。您可以忽略它。
* **首次 API 调用后 Usage 页面没有数据：** 用量数据可能需要几分钟才会出现在 Claude Console 中。
* **首次 API 调用时出现 "Outbound web identity federation is disabled"：** 每个账户需启用一次联合身份验证。请参阅[启用出站 Web 身份联合](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#enable-outbound-web-identity-federation)。

## 发起 API 调用之前

请确保您具备：

1. 一个已订阅 Claude Platform on AWS 的有效 AWS 账户（参见[设置您的账户](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#set-up-your-account)）
2. 已安装并配置 [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
3. 在您的 AWS 账户上**已启用出站 Web 身份联合**，这是一次性设置步骤（参见[启用出站 Web 身份联合](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#enable-outbound-web-identity-federation)）
4. 您的工作区 ID（参见[获取您的工作区 ID](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#obtain-your-workspace-id)）
5. 调用 API 的 IAM 权限：对您工作区的 `aws-external-anthropic:CreateInference` 操作，如果您使用 API 密钥进行身份验证，还需要 `aws-external-anthropic:CallWithBearerToken`（参见 [IAM 策略](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#iam-policies)）

### 启用出站 Web 身份联合

Claude Platform on AWS 网关在服务器端调用 `sts:GetWebIdentityToken` 来生成一个 JWT 并转发给 Anthropic。此 STS 功能在每个 AWS 账户上**默认禁用**。每个账户需启用一次：

```bash CLI
aws iam enable-outbound-web-identity-federation
```

如果响应为 `[ERROR] (FeatureEnabled) ... already enabled`，则该设置已在您的账户上开启，您可以继续下一步。验证并获取您账户的颁发者 URL：

```bash CLI
aws iam get-outbound-web-identity-federation-info
```

<Warning>
  如果没有完成此步骤，每个请求都会返回 `"Outbound web identity federation is disabled for your account"`。这是最常见的设置错误。
</Warning>

### 获取您的工作区 ID

完成账户设置后，您可以从 AWS Console 创建工作区（参见[设置您的账户](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#set-up-your-account)）。工作区绑定到单个 AWS 区域。您可以在 [Claude Console](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#using-the-claude-console) 的 **Workspaces** 下或 AWS Console 服务页面的 **Workspaces** 部分找到工作区 ID。

设置 `ANTHROPIC_AWS_WORKSPACE_ID` 和 `AWS_REGION` 环境变量，以便 SDK 客户端自动读取它们：

```bash CLI
export ANTHROPIC_AWS_WORKSPACE_ID='wrkspc_01AbCdEf23GhIj'
export AWS_REGION='us-west-2'  # Your workspace's AWS region
```

区域是必需的。如果未设置区域，SDK 客户端会引发错误。请将 `aws_region`/`awsRegion` 传递给构造函数，或设置 `AWS_REGION`（或 `AWS_DEFAULT_REGION`）。支持所有 AWS 商业区域。

## 身份验证

Claude Platform on AWS 支持两种身份验证方法：使用 Signature Version 4（SigV4）请求签名的 AWS IAM（主要方式）和 API 密钥身份验证。两者使用相同的基础 URL 和请求格式。

### SigV4 身份验证

SigV4 是企业原生路径，可与您现有的 AWS IAM 策略、角色和审计集成。使用 [AWS 默认凭证提供程序链](https://docs.aws.amazon.com/sdkref/latest/guide/standardized-credentials.html)支持的任何方法配置 AWS 凭证：

* 环境变量（`AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`、`AWS_SESSION_TOKEN`）
* 共享凭证文件（`~/.aws/credentials`）
* 共享配置文件（`~/.aws/config`），包括 SSO 和 `credential_process`
* Web 身份（`AWS_WEB_IDENTITY_TOKEN_FILE` 和 `AWS_ROLE_ARN`），用于 IRSA 和 GitHub Actions
* ECS 容器凭证
* EC2 实例元数据服务（IMDS）

验证您的凭证是否有效：

```bash CLI
aws sts get-caller-identity
```

### API 密钥身份验证

对于更简单的集成路径（本地开发和脚本），您可以使用 API 密钥而非 SigV4 进行身份验证。设置 `ANTHROPIC_AWS_API_KEY` 环境变量或将 `apiKey` 传递给 SDK 构造函数。

在 **AWS Console** 的 **Claude Platform on AWS → API keys** 下生成 API 密钥。选择 **Generate a key**，然后复制密钥值。将 `aws-external-anthropic:CallWithBearerToken` IAM 操作授予应被允许使用 API 密钥身份验证的主体。

<Note>
  Claude Platform on AWS 的 API 密钥在 AWS Console 中管理，而非 Claude Console。在标准 [Claude Console](https://platform.claude.com/) 中创建的密钥（用于第一方 API 访问）不适用于 Claude Platform on AWS 端点。
</Note>

#### 短期 API 密钥

对于需要将凭证交给单独进程的工作负载（例如 LLM 网关、无服务器函数，或支持 bearer 令牌身份验证但不支持 SigV4 的工具），请从您的 AWS 凭证生成短期 API 密钥，而不是在 AWS Console 中预配长期密钥。

AWS 发布了适用于 [JavaScript](https://github.com/aws/token-generator-for-aws-external-anthropic-js)、[Python](https://github.com/aws/token-generator-for-aws-external-anthropic-python) 和 [Java](https://github.com/aws/token-generator-for-aws-external-anthropic-java) 的令牌生成器库。每个库通过标准提供程序链读取您的 AWS 凭证，并返回一个可与 `x-api-key` 标头配合使用的限时令牌。令牌有效期默认为 12 小时，上限为您请求的时长、您的 AWS 凭证到期时间和 12 小时三者中的最小值。有关安装和完整配置选项，请参阅链接的仓库 README。

将生成的令牌传递给 SDK，方式与传递 AWS Console 生成的 API 密钥相同：

<CodeGroup exclude="shell, csharp, go, php, ruby">
  ```python Python
  from token_generator_for_aws_external_anthropic import TokenGenerator
  from anthropic import AnthropicAWS

  token = TokenGenerator(region="us-west-2").get_token()

  client = AnthropicAWS(api_key=token, aws_region="us-west-2")
  ```

  ```typescript TypeScript
  import { getTokenProvider } from "@aws/token-generator-for-aws-external-anthropic";
  import AnthropicAws from "@anthropic-ai/aws-sdk";

  const tokenProvider = getTokenProvider({ region: "us-west-2" });
  const token = await tokenProvider();

  const client = new AnthropicAws({ apiKey: token, awsRegion: "us-west-2" });
  ```

  ```java Java
  import software.amazon.awsexternalanthropic.TokenGenerator;
  import software.amazon.awssdk.regions.Region;
  import com.anthropic.aws.backends.AwsBackend;
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;

  void main() {
      String token = TokenGenerator.builder().region(Region.US_WEST_2).build().getToken();

      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(AwsBackend.builder()
              .apiKey(token)
              .region(Region.US_WEST_2)
              .workspaceId(System.getenv("ANTHROPIC_AWS_WORKSPACE_ID"))
              .build())
          .build();
  }
  ```
</CodeGroup>

如果您可以在本地生成令牌，说明您的进程已经拥有 SigV4 凭证，此时 SigV4 身份验证通常是更简单的选择。当发起 API 调用的进程与持有 AWS 凭证的进程分离时，请使用短期密钥。

SDK 不会自动刷新短期密钥。令牌过期后，请生成新令牌并构造新客户端。使用该令牌的主体仍需要 `aws-external-anthropic:CallWithBearerToken` IAM 操作。

### 凭证优先级

平台专用客户端按以下顺序解析身份验证。参数名称因语言惯例而异：TypeScript 和 PHP 使用如下所示的 camelCase，Python 和 Ruby 使用 snake\_case，Go 使用首字母缩略词大写的 PascalCase，C# 和 Java 使用各自语言的属性或构建器惯用法。

1. `apiKey` 构造函数参数 → `x-api-key` 标头
2. `awsAccessKey` + `awsSecretAccessKey` 构造函数参数 → AWS SigV4
3. `awsProfile` 构造函数参数 → 使用命名配置文件的 AWS SigV4
4. `ANTHROPIC_AWS_API_KEY` 环境变量 → `x-api-key` 标头
5. 默认 AWS 凭证提供程序链 → AWS SigV4

### 区域解析

如果未将 `aws_region`/`awsRegion` 传递给构造函数，客户端会从环境中读取 `AWS_REGION`，并回退到 `AWS_DEFAULT_REGION` 以兼容标准 AWS SDK。区域是必需的且没有默认值：如果既未设置构造函数参数也未设置环境变量，`AnthropicAWS`/`AnthropicAws` 客户端会引发错误。

## 安装 SDK

Anthropic 的[客户端 SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview) 支持 Claude Platform on AWS。每个 SDK 都提供一个平台专用客户端类，用于处理 SigV4 签名、基于区域的基础 URL 构建以及 `anthropic-workspace-id` 标头。

<Tabs>
  <Tab title="Python">
    ```bash
    pip install -U "anthropic[aws]"
    ```

    <Tip>
      在使用 Homebrew Python 的 macOS 或其他外部管理的 Python 环境中，`pip install` 可能会因 PEP 668 `externally-managed-environment` 错误而失败。请先创建并激活虚拟环境：`python3 -m venv .venv && source .venv/bin/activate`。
    </Tip>
  </Tab>

  <Tab title="TypeScript">
    ```bash
    npm install @anthropic-ai/aws-sdk
    ```
  </Tab>

  <Tab title="C#">
    ```bash
    dotnet add package Anthropic.Aws
    ```
  </Tab>

  <Tab title="Go">
    ```bash
    go get github.com/anthropics/anthropic-sdk-go
    ```
  </Tab>

  <Tab title="Java">
    ```kotlin Gradle
    implementation("com.anthropic:anthropic-java-aws:2.58.0")
    ```

    ```xml Maven
    <dependency>
      <groupId>com.anthropic</groupId>
      <artifactId>anthropic-java-aws</artifactId>
      <version>2.58.0</version>
    </dependency>
    ```
  </Tab>

  <Tab title="PHP">
    ```bash
    composer require anthropic-ai/sdk aws/aws-sdk-php
    ```
  </Tab>

  <Tab title="Ruby">
    ```bash
    gem install anthropic aws-sdk-core
    ```
  </Tab>
</Tabs>

<Note>
  Claude Platform on AWS 的 SDK 客户端处于测试阶段。
</Note>

## 可用模型

以下模型在 Claude Platform on AWS 上可用：

| 模型                | 模型 ID             |
| ----------------- | ----------------- |
| Claude Fable 5.1  | claude-fable-5-1  |
| Claude Fable 5    | claude-fable-5    |
| Claude Opus 5     | claude-opus-5     |
| Claude Opus 4.8   | claude-opus-4-8   |
| Claude Opus 4.7   | claude-opus-4-7   |
| Claude Opus 4.6   | claude-opus-4-6   |
| Claude Sonnet 5   | claude-sonnet-5   |
| Claude Sonnet 4.6 | claude-sonnet-4-6 |
| Claude Opus 4.5   | claude-opus-4-5   |
| Claude Sonnet 4.5 | claude-sonnet-4-5 |
| Claude Haiku 4.5  | claude-haiku-4-5  |

模型 ID 与第一方 Claude API 完全相同。没有 Bedrock 风格的 ARN 或 `anthropic.` 前缀。

新模型通常与第一方 Claude API 同日在 Claude Platform on AWS 上发布。

<Tip>
  正在升级到更新的 Claude 模型？在 Claude Code 中，运行 `/claude-api migrate` 即可在您的整个代码库中应用模型 ID 替换和破坏性参数变更。该技能会检测您的代码所面向的云平台，并针对该平台调整模型 ID 格式和功能变更。请参阅[迁移到更新的 Claude 模型](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model)。
</Tip>

## 发起请求

Claude Platform on AWS 使用与第一方 Claude API 相同的 API 端点。区别在于基础 URL、身份验证方法，以及一个必需的 `anthropic-workspace-id` 标头，用于标识请求所针对的[工作区](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#workspaces)。

在运行这些示例之前，请完成[发起 API 调用之前](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#before-making-api-calls)中的步骤。

<CodeGroup>
  ```bash cURL
  # 请将 URL 和 --aws-sigv4 中的 us-west-2 替换为您的 AWS 区域
  # 如果您使用长期 IAM 用户凭证，请省略 x-amz-security-token 标头
  curl "https://aws-external-anthropic.us-west-2.api.aws/v1/messages" \
    --aws-sigv4 "aws:amz:us-west-2:aws-external-anthropic" \
    --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
    -H "x-amz-security-token: $AWS_SESSION_TOKEN" \
    -H "content-type: application/json" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-workspace-id: $ANTHROPIC_AWS_WORKSPACE_ID" \
    -d '{
      "model": "claude-sonnet-5",
      "max_tokens": 1024,
      "messages": [
        {"role": "user", "content": "Hello!"}
      ]
    }'
  ```

  ```bash CLI
  # 将 us-west-2 替换为您的 AWS 区域
  # ant 会读取 ANTHROPIC_API_KEY 并将其作为 x-api-key 发送。请在
  # AWS Console 中生成密钥（参见 API 密钥身份验证）。
  export ANTHROPIC_API_KEY="YOUR_AWS_API_KEY"

  ant messages create \
    --base-url https://aws-external-anthropic.us-west-2.api.aws \
    --workspace-id "$ANTHROPIC_AWS_WORKSPACE_ID" \
    --model claude-sonnet-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello!"}' \
    --transform content
  ```

  ```python Python
  from anthropic import AnthropicAWS

  client = AnthropicAWS()

  message = client.messages.create(
      model="claude-sonnet-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello!"}],
  )
  print(message)
  ```

  ```typescript TypeScript
  import AnthropicAws from "@anthropic-ai/aws-sdk";

  const client = new AnthropicAws();

  const message = await client.messages.create({
    model: "claude-sonnet-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello!" }]
  });
  console.log(message);
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Aws;

  var client = new AnthropicAwsClient();

  var message = await client.Messages.Create(new()
  {
      Model = Model.ClaudeSonnet5,
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello!" }]
  });

  Console.WriteLine(message);
  ```

  ```go Go
  client, err := anthropicaws.NewClient(context.Background(), anthropicaws.ClientConfig{})
  if err != nil {
  	panic(err)
  }

  message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeSonnet5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Hello!")),
  	},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Println(message)
  ```

  ```java Java
  import com.anthropic.aws.backends.AwsBackend;
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.models.messages.Message;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(AwsBackend.fromEnv())
          .build();

      Message message = client.messages().create(
          MessageCreateParams.builder()
              .model(Model.CLAUDE_SONNET_5)
              .maxTokens(1024)
              .addUserMessage("Hello!")
              .build()
      );

      IO.println(message);
  }
  ```

  ```php PHP
  use Anthropic\Aws\Client;

  $client = new Client();

  $message = $client->messages->create(
      model: 'claude-sonnet-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello!']],
  );

  echo $message;
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::AWSClient.new

  message = client.messages.create(
    model: "claude-sonnet-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello!" }]
  )

  puts message
  ```
</CodeGroup>

客户端从环境中读取 `AWS_REGION`（或 `AWS_DEFAULT_REGION`）和 `ANTHROPIC_AWS_WORKSPACE_ID`。您可以通过向构造函数传递 `aws_region` / `awsRegion` 或 `workspace_id` / `workspaceId` 来覆盖其中任一项。区域和工作区 ID 都是必需的。如果任一项无法解析，构造函数会引发错误。

<Note>
  `x-amz-security-token` 标头（cURL）仅在使用临时凭证（例如 IAM 角色、SSO 或 STS）时需要。使用长期 IAM 用户凭证时请省略它。SDK 客户端会根据凭证来源自动处理此项。
</Note>

`--aws-sigv4` 值遵循 `aws:amz:<region>:<service>` 格式。SigV4 服务名称为 `aws-external-anthropic`，区域必须与您端点 URL 中的区域匹配。任一项不匹配都会产生通用的签名拒绝错误，而非具体的诊断信息。

### 上下文窗口

Claude Platform on AWS 上的 "context window"（上下文窗口）大小与第一方 Claude API 完全相同。有关各模型的限制，请参阅[上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)。

## 功能支持

Claude Platform on AWS 直接使用 Claude API 端点，这意味着您可以获得与第一方 Claude API 完全一致的功能（[功能限制](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#features-not-supported)中注明的除外）：

* **功能访问：** 由于 Anthropic 同时运营这两个平台，大多数新功能和测试版标头无需单独的集成步骤即可在 Claude Platform on AWS 上使用。例外情况请参阅[功能限制](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#features-not-supported)。
* **测试版功能：** 传递标准的 `anthropic-beta` 标头即可访问测试版功能，与使用 Claude API 时一样。
* **Agent Skills：** 使用与 Claude API 相同的 `container.skills` 参数来使用预构建和自定义的 [Agent Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview)。所有预构建 Skills（PowerPoint、Excel、Word、PDF）均可开箱即用。
* **代码执行：** 使用[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)在 Anthropic 的托管沙箱中运行代码。
* **工具使用：** 计算机使用和所有其他 "tool use"（工具使用）[功能](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)均可用。
* **扩展思考：** 使用与 Claude API 相同的参数启用 "extended thinking"（扩展思考）。
* **流式传输：** 完整的 SSE "streaming"（流式传输）支持，用于实时响应。
* **批处理：** 为高吞吐量工作负载提交批量请求。
* **提示缓存：** 缓存工具、系统提示和消息历史以降低延迟和成本。所有 "prompt caching"（提示缓存）功能（5 分钟 TTL、1 小时 TTL 和自动缓存）均可用。
* **Files API：** 上传文件并在多个请求中引用。
* **客户管理的加密密钥（CMEK）：** [CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek) 仅支持 [AWS KMS](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms) 密钥。无法注册 Google Cloud KMS 和 Azure Key Vault 密钥。密钥必须是与其所附加工作区位于同一 AWS 账户和区域的单区域 KMS 密钥，且其密钥策略必须向 `aws-external-anthropic.amazonaws.com` 服务主体授予访问权限；请参阅[在 Claude Platform on AWS 上设置 CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#claude-platform-on-aws)。在 [Claude Console](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#using-the-claude-console) 中注册和附加密钥；外部密钥端点也可用，通过 [IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#encryption-keys)授权。没有单独的验证步骤：当您将密钥附加到工作区时，密钥会被隐式验证（附加调用会执行一轮加密/解密），因此密钥策略问题会在附加时而非注册时显现。
* **Compliance API：** [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 可用。访问通过 AWS IAM [`ListComplianceActivities` 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#compliance)授权。

有关与 Amazon Bedrock 的功能可用性差异，请参阅[对比表](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#claude-platform-on-aws-vs-amazon-bedrock)。

### Claude Managed Agents

[Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 在 Claude Platform on AWS 上可用，包括[智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)、[环境](https://platform.claude.com/docs/zh-CN/managed-agents/environments)、[会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)、[凭证保管库](https://platform.claude.com/docs/zh-CN/managed-agents/vaults)、[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory)、[webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks)、[多智能体编排](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)和[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)。

Claude Platform on AWS 上的会话行为与第一方 Claude Managed Agents 在两个方面有所不同：

* **自主会话重新身份验证：** 会话可以在没有任何[用户事件](https://platform.claude.com/docs/zh-CN/managed-agents/reference#event-types)的情况下自主运行最多 6 小时。6 小时后，会话需要重新身份验证才能继续。要重新身份验证，请向会话发送任意用户角色事件（参见[事件和流式传输](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)）。第一方 Claude Managed Agents 没有自主会话运行时限制。
* **[自托管环境上的记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)：** 在自托管环境上运行的会话无法附加记忆存储；包含记忆存储的会话会在创建时被拒绝。云环境上的会话照常附加记忆存储。在第一方 Claude Managed Agents 上，云环境和自托管环境上的会话都可以附加记忆存储。

### 不支持的功能

以下功能目前在 Claude Platform on AWS 上不可用：

* **HIPAA 就绪：** Anthropic 的 HIPAA 就绪计划不可用。请参阅 [API 和数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
* **计算机使用和浏览器使用工具集：** `computer_toolset_20260801` 和 `browser_toolset_20260801` 目前在 Claude Platform on AWS 上不可用。测试版[计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#earlier-tool-versions)工具版本仍然可用。

- **Admin API：** 工作区端点（`/v1/organizations/workspaces` 上的创建、获取、列出、更新和归档）和外部密钥端点（`/v1/organizations/external_keys` 上的注册、获取、列出、更新和删除，用于 [CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)；密钥在附加到工作区时验证，而非通过验证端点）可用。其他 Admin API 端点（组织成员、工作区成员、邀请、API 密钥、用量报告、成本报告和速率限制报告）目前不可用。请改为在 [Claude Console](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#using-the-claude-console) 中查看用量和成本数据。组织成员资格由 AWS IAM 管理。
- **工作区成员管理：** 无法向单个工作区添加或移除用户。访问由工作区 ARN 上的 AWS IAM 策略控制。
- **Claude Code 工作区和 Analytics API：** 具有自动速率限制的 Claude Code 工作区不可用。Claude Code 用量显示在通用用量视图中，而非专用界面。
- **OAuth 身份验证：** 不支持。请使用 SigV4 或 API 密钥身份验证。
- **快速模式：** 在 Claude Platform on AWS 上不可用。
- **OpenAI 兼容 API 端点：** 在 Claude Platform on AWS 上不可用。
- **MCP 隧道：** 仅支持通过公共互联网公开的 MCP 服务器。

## 数据驻留

Claude Platform on AWS 支持以下推理地理区域：

* **US：** 推理保留在美国数据中心内。适用 1.1 倍的定价乘数。
* **Global：** 推理可以路由到全球任何由 Anthropic 运营的数据中心。适用标准定价。

<Note>
  您的工作区所绑定的 AWS 区域决定了您调用哪个网关端点，以及 AWS 侧资源（IAM、CloudTrail、计费）的作用范围。它并不固定模型推理的运行位置。要将推理固定到特定地理区域，请在每个请求上设置 `inference_geo`，或配置工作区默认值。
</Note>

使用 `inference_geo` 参数为每个请求设置推理地理区域：

<Note>
  `inference_geo` 参数在 Claude 4.6 及更高版本的模型上受支持。在 Claude Opus 4.5、Claude Sonnet 4.5 或 Claude Haiku 4.5 上带有 `inference_geo` 的请求会返回 400 错误。有关模型可用性的详细信息，请参阅[数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)。
</Note>

<CodeGroup>
  ```bash cURL
  # 请将 URL 和 --aws-sigv4 中的 us-west-2 替换为您的 AWS 区域
  # 如果您使用长期 IAM 用户凭证，请省略 x-amz-security-token 标头
  curl "https://aws-external-anthropic.us-west-2.api.aws/v1/messages" \
    --aws-sigv4 "aws:amz:us-west-2:aws-external-anthropic" \
    --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
    -H "x-amz-security-token: $AWS_SESSION_TOKEN" \
    -H "content-type: application/json" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-workspace-id: $ANTHROPIC_AWS_WORKSPACE_ID" \
    -d '{
      "model": "claude-sonnet-5",
      "max_tokens": 1024,
      "inference_geo": "us",
      "messages": [
        {"role": "user", "content": "Hello!"}
      ]
    }'
  ```

  ```bash CLI
  # 将 us-west-2 替换为您的 AWS 区域
  # ant 会读取 ANTHROPIC_API_KEY 并将其作为 x-api-key 发送。请在
  # AWS Console 中生成密钥（参见 API 密钥身份验证）。
  export ANTHROPIC_API_KEY="YOUR_AWS_API_KEY"

  ant messages create \
    --base-url https://aws-external-anthropic.us-west-2.api.aws \
    --workspace-id "$ANTHROPIC_AWS_WORKSPACE_ID" \
    --model claude-sonnet-5 \
    --max-tokens 1024 \
    --inference-geo us \
    --message '{role: user, content: "Hello!"}' \
    --transform content
  ```

  ```python Python
  from anthropic import AnthropicAWS

  client = AnthropicAWS()
  message = client.messages.create(
      model="claude-sonnet-5",
      max_tokens=1024,
      inference_geo="us",
      messages=[{"role": "user", "content": "Hello!"}],
  )
  print(message)
  ```

  ```typescript TypeScript
  import AnthropicAws from "@anthropic-ai/aws-sdk";
  const client = new AnthropicAws();
  const message = await client.messages.create({
    model: "claude-sonnet-5",
    max_tokens: 1024,
    inference_geo: "us",
    messages: [{ role: "user", content: "Hello!" }]
  });
  console.log(message);
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Aws;

  var client = new AnthropicAwsClient();

  var message = await client.Messages.Create(new()
  {
      Model = Model.ClaudeSonnet5,
      MaxTokens = 1024,
      InferenceGeo = "us",
      Messages = [new() { Role = Role.User, Content = "Hello!" }]
  });

  Console.WriteLine(message);
  ```

  ```go Go
  client, err := anthropicaws.NewClient(context.Background(), anthropicaws.ClientConfig{})
  if err != nil {
  	panic(err)
  }

  message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  	Model:        anthropic.ModelClaudeSonnet5,
  	MaxTokens:    1024,
  	InferenceGeo: anthropic.String("us"),
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Hello!")),
  	},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Println(message)
  ```

  ```java Java
  import com.anthropic.aws.backends.AwsBackend;
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.models.messages.Message;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(AwsBackend.fromEnv())
          .build();

      Message message = client.messages().create(
          MessageCreateParams.builder()
              .model(Model.CLAUDE_SONNET_5)
              .maxTokens(1024)
              .inferenceGeo("us")
              .addUserMessage("Hello!")
              .build()
      );

      IO.println(message);
  }
  ```

  ```php PHP
  use Anthropic\Aws\Client;

  $client = new Client();

  $message = $client->messages->create(
      model: 'claude-sonnet-5',
      maxTokens: 1024,
      inferenceGeo: 'us',
      messages: [['role' => 'user', 'content' => 'Hello!']],
  );

  echo $message;
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::AWSClient.new

  message = client.messages.create(
    model: "claude-sonnet-5",
    max_tokens: 1024,
    inference_geo: "us",
    messages: [{ role: "user", content: "Hello!" }]
  )

  puts message
  ```
</CodeGroup>

如果您省略 `inference_geo`，请求将使用工作区的 `default_inference_geo`（如果已配置），否则使用 `global`。

工作区级别的推理地理区域控制（`allowed_inference_geos` 和 `default_inference_geo`）在 Claude Platform on AWS 上同样可用。请参阅[工作区级别限制](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency#workspace-level-restrictions)。

## 工作区

Claude Platform on AWS 上的推理和资源请求以某个 "workspace"（工作区）为目标。您在这些 API 调用的 `anthropic-workspace-id` 标头中传递工作区的 ID。工作区 ID 使用带标签的格式：`wrkspc_` 后跟一个字母数字标识符（例如 `wrkspc_01AbCdEf23GhIj`）。如果您还没有工作区 ID，请参阅[获取您的工作区 ID](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#obtain-your-workspace-id)。

### 工作区作用范围

工作区绑定到单个 AWS 区域。在 `us-west-2` 中创建的工作区只能通过 `us-west-2` 端点访问。用量、配额、成本、文件、批处理和 Skills 均按工作区汇总，从而在 Claude Console 中为您提供按区域的细分。

工作区还是 Claude Platform on AWS 的主要 IAM 资源。您可以使用工作区 ARN，通过 AWS IAM 策略授予或拒绝对特定工作区的访问。ARN 的资源段与您在 `anthropic-workspace-id` 标头中传递的带 `wrkspc_` 前缀的 ID 相同：

```text wrap
arn:aws:aws-external-anthropic:{region}:{account-id}:workspace/{workspace-id}
```

例如：

```text wrap
arn:aws:aws-external-anthropic:us-west-2:123456789012:workspace/wrkspc_01AbCdEf23GhIj
```

有关策略示例，请参阅 [IAM 策略](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#iam-policies)。

### 管理工作区

您可以从 AWS Console 的 **Workspaces** 页面或使用 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 工作区端点来创建额外的工作区、重命名工作区或归档工作区。新工作区会绑定到您调用以创建它的端点所在的 AWS 区域（请参阅[工作区作用范围](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#workspace-scoping)）。拥有 Admin 角色时，您还可以从 Claude Console 的 **Workspaces** 页面创建、重命名和归档工作区。

## 使用 Claude Console

Claude Platform on AWS 使用位于 [platform.claude.com](https://platform.claude.com) 的标准 Claude Console。当您从 AWS Console 登录时，Claude Console 侧边栏左下角会显示 **Account managed by AWS** 指示标识，并且 Console 的作用范围会限定为您的 Claude Platform on AWS 组织。它提供用量分析、成本细分、速率限制可见性、工作区管理，以及用于管理文件、Agent Skills、批处理作业和 Claude Managed Agents 资源（代理、会话、环境、凭证保管库、记忆存储和 webhook）的页面。

### 登录

对 Claude Console 的访问通过 AWS IAM 进行联合身份验证。有关完整的首次登录流程，请参阅[设置您的账户](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#set-up-your-account)。简而言之：

1. 代入一个具有 `aws-external-anthropic:AssumeConsole` 权限的 IAM 角色。请参阅 [Claude Platform on AWS 的 IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#console-access)。
2. 在 [AWS Console](https://console.aws.amazon.com/) 中导航到 Claude Platform on AWS 页面。
3. 选择 **Open Claude Console**。AWS Console 会签发一个 JWT 并将您重定向到 `platform.claude.com`。
4. 首次登录时，系统会提示您输入电子邮件地址。请输入您的工作邮箱。平台会即时（just-in-time）为您配置 Claude Console 用户。

有两个 Claude Console 角色可用：**Admin** 和 **Developer**。Admin 角色授予对 Claude Platform on AWS 可用的所有 Claude Console 页面和设置的访问权限。Developer 角色授予对用量、成本、速率限制和工作区信息的读取权限。请联系您的 Anthropic 客户代表，为某个主体分配 Admin 或 Developer 角色。

### 可用页面

**通过 AWS 网关**列指示该页面是否通过 AWS 网关读取和写入数据（因此受 [IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions)管控）。标记为**否**的页面直接从 Anthropic 读取组织级元数据，并绕过 IAM 操作检查。

| 页面                    | 可用    | 通过 AWS 网关 | 备注                                                                                                                                                                                                  |
| --------------------- | ----- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Usage**             | 是     | 否         | 按模型、工作区和维度查看令牌用量。请求发出后，数据可能需要几分钟才会显示。                                                                                                                                                               |
| **Cost**              | 是     | 否         | 按模型和工作区查看成本细分。AWS Cost Explorer 显示汇总的 [Claude Consumption Unit (CCU)](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#billing) 行项目。                                 |
| **Rate limits**       | 是     | 否         | 查看速率限制（只读）。层级提升需通过您的 Anthropic 客户代表办理；请参阅[速率限制和配额](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#rate-limits-and-quotas)。                                         |
| **Workspaces**        | 是     | 是（支出限额除外） | 查看按区域划分的工作区。拥有 Admin 角色时，您还可以创建、重命名和归档工作区，并设置每个工作区的[支出限额](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#spend-limits)。                                            |
| **Encryption keys**   | 是     | 是         | 在 **Settings → Encryption keys** 下，为 [CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#claude-platform-on-aws) 注册 AWS KMS 密钥（Admin 角色）。从工作区的 **Security** 设置中将已注册的密钥附加到该工作区。 |
| **Files**             | 是     | 是         | 查看和管理已上传的文件。                                                                                                                                                                                        |
| **Skills**            | 是     | 是         | 查看和管理 Agent Skills。                                                                                                                                                                                 |
| **Batches**           | 是     | 是         | 查看和管理批处理作业。                                                                                                                                                                                         |
| **Agents**            | 是     | 是         | 查看和管理代理定义。                                                                                                                                                                                          |
| **Sessions**          | 是     | 是         | 查看代理会话和事件历史。                                                                                                                                                                                        |
| **Environments**      | 是     | 是         | 查看和管理会话的云沙箱配置。                                                                                                                                                                                      |
| **Credential vaults** | 是     | 是         | 查看和管理用于会话身份验证的凭证保管库。                                                                                                                                                                                |
| **Memory stores**     | 是     | 是         | 查看和管理持久化的代理记忆。                                                                                                                                                                                      |
| **Webhooks**          | 是     | 是         | 在 **Settings → Webhooks** 下查看和管理 webhook 端点。                                                                                                                                                        |
| **API keys**          | 否     | 不适用       | 在 AWS Console 中管理 API 密钥（**Claude Platform on AWS → API keys**）。请参阅 [API 密钥身份验证](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#api-key-authentication)。           |
| **Members**           | 否     | 不适用       | 不适用。由 AWS IAM 管理访问。                                                                                                                                                                                 |
| **Billing**           | 是（有限） | 否         | 设置组织月度支出限额；请参阅[支出限额](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#spend-limits)。由 AWS Marketplace 管理开票。在 Cost 页面查看成本细分。                                          |
| **Claude Code**       | 否     | 不适用       | 在 Usage 页面查看 Claude Code 用量。                                                                                                                                                                        |

### 切换组织

Claude Console 不支持 Claude Platform on AWS 的组织切换。要访问其他组织，请退出登录，然后使用该组织所属 AWS 账户的 IAM 角色通过 AWS Console 重新进行身份验证。

## 速率限制和配额

Claude Platform on AWS 上的组织被置于 Start 层级。Anthropic 直接管理速率限制，而不是通过 AWS 配额系统。

Claude Platform on AWS 上的组织不会在用量层级之间自动移动。基于用量的层级晋升适用于第一方 Claude API 组织，而不适用于通过 AWS Marketplace 计费的组织。Claude Console 中的自助式 **Request rate limit increase** 流程同样不可用：Rate limits 页面会引导您联系您的 Anthropic 客户代表。

要申请更高的限制，请联系您的 Anthropic 客户代表或 [Anthropic 支持](https://support.claude.com)。请在您的申请中包含以下内容：

* 您需要提升限制的模型
* 每个模型的峰值每分钟输入令牌数和每分钟输出令牌数（而非每日总量）
* 您的输入中属于缓存或重复上下文的大致比例（对于大多数模型，缓存读取不计入输入令牌限制；请参阅[缓存感知 ITPM](https://platform.claude.com/docs/zh-CN/api/rate-limits#cache-aware-itpm)）

用量层级是固定的阶梯：每个层级将速率限制与[月度支出上限](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#spend-limits)配对，升至更高层级会同时提高两者。有关层级详情和每个模型的限制，请参阅[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)。

## 计费

Claude Platform on AWS 通过 [AWS Marketplace](https://aws.amazon.com/marketplace) 计费。用量以 Claude Consumption Units（CCU）计价，按小时计量，并按月后付方式在您的 AWS 账单上开票。CCU 不是预付额度。不存在 CCU 余额或承诺用量。

有关 CCU 价格、换算机制、折扣应用以及每个模型的令牌费率，请参阅 [Claude Platform on AWS 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-platform-on-aws-pricing)。

### 支出限额

Start、Build 和 Scale 用量层级各自带有月度支出上限；有关当前数值，请参阅[各层级支出上限](https://platform.claude.com/docs/zh-CN/api/rate-limits#spend-limits)。当您的组织在日历月内的用量达到其层级上限时，API 请求将以[支出上限错误](https://platform.claude.com/docs/zh-CN/api/rate-limits#reaching-your-spend-cap)失败，直到下个月第一天的 00:00 UTC，在此之前重试不会成功。支出上限和速率限制属于同一层级。要提高上限，或在达到上限后恢复访问，请通过您的 Anthropic 客户代表或 [Anthropic 支持](https://support.claude.com)申请层级提升（请参阅[速率限制和配额](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#rate-limits-and-quotas)）。

在 Billing 页面的 **Email recipients** 下添加至少一个收件人后，您还可以设置低于上限的自定义月度支出限额：

* **组织支出限额：** 前往 [Claude Console](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#using-the-claude-console) 中的 [Settings > Billing](https://platform.claude.com/settings/billing) 设置月度支出限额。
* **工作区支出限额：** 在 [Settings > Workspaces](https://platform.claude.com/settings/workspaces) 下选择一个工作区，并打开其 **Spend limits** 页面。

当用量达到您设置的限额时，请求将以 HTTP 400 失败（请参阅[支出限额错误](https://platform.claude.com/docs/zh-CN/api/rate-limits#setting-your-own-spend-limit)），直到下个月第一天的 00:00 UTC，或直到您提高或移除该限额。

支出按标价计算，可能需要大约 2 小时才能反映最近的用量，因此在请求开始失败之前，用量可能会超出上限或限额。超出部分会被计费。当层级上限或组织支出限额阻止您的请求时，系统会向 Billing 页面 **Email recipients** 下列出的收件人发送电子邮件通知。基于角色的收件人（例如所有管理员）在 Claude Platform on AWS 上不可用。层级上限通知还会发送到 AWS Marketplace 注册时使用的电子邮件地址。

## 监控和日志记录

AWS CloudTrail 可以捕获对 Claude Platform on AWS 的所有请求。工作区、外部密钥、合规、保管库和 webhook 操作默认记录为管理事件（Management events）。推理、批处理、文件、skill、模型、用户配置文件以及 Claude Managed Agents 操作（保管库和 webhook 除外）被归类为数据事件（Data events），需要显式配置数据事件日志记录，这会产生额外的 CloudTrail 费用。有关完整的事件类型分类，请参阅 [IAM 操作参考](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#route-to-action-mapping)；有关配置详情，请参阅 [AWS CloudTrail 文档](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/)。

### 请求 ID

每个响应在响应标头中包含两个请求 ID：

* **AWS 请求 ID（`x-amzn-requestid`）：** 主 ID，在 CloudTrail 中建立索引。通过 AWS 工具调查请求或联系 AWS 支持时使用此 ID。
* **Anthropic 请求 ID（`request-id`）：** 辅助 ID。联系 Anthropic 支持时使用此 ID。

<CodeGroup>
  ```bash cURL
  # 将 URL 和 --aws-sigv4 中的 us-west-2 替换为您的 AWS 区域
  # -i 会在输出中包含响应头
  # 如果您使用长期 IAM 用户凭证，请省略 x-amz-security-token 头
  curl -i "https://aws-external-anthropic.us-west-2.api.aws/v1/messages" \
    --aws-sigv4 "aws:amz:us-west-2:aws-external-anthropic" \
    --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
    -H "x-amz-security-token: $AWS_SESSION_TOKEN" \
    -H "content-type: application/json" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-workspace-id: $ANTHROPIC_AWS_WORKSPACE_ID" \
    -d '{
      "model": "claude-sonnet-5",
      "max_tokens": 1024,
      "messages": [
        {"role": "user", "content": "Hello!"}
      ]
    }'
  ```

  ```bash CLI
  # ant CLI 的输出格式仅打印响应正文，不打印响应标头。
  # 如需读取 x-amzn-requestid，请使用 cURL 示例（-i）或 SDK 示例。
  ```

  ```python Python
  from anthropic import AnthropicAWS

  client = AnthropicAWS()

  response = client.messages.with_raw_response.create(
      model="claude-sonnet-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello!"}],
  )

  print(response.headers.get("x-amzn-requestid"))  # AWS request ID
  print(response.headers.get("request-id"))  # Anthropic request ID

  message = response.parse()
  print(message.content)
  ```

  ```typescript TypeScript
  import AnthropicAws from "@anthropic-ai/aws-sdk";

  const client = new AnthropicAws();

  const { data: message, response } = await client.messages
    .create({
      model: "claude-sonnet-5",
      max_tokens: 1024,
      messages: [{ role: "user", content: "Hello!" }]
    })
    .withResponse();

  console.log(response.headers.get("x-amzn-requestid")); // AWS request ID
  console.log(response.headers.get("request-id")); // Anthropic request ID
  console.log(message.content);
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Aws;

  var client = new AnthropicAwsClient();

  var response = await client.WithRawResponse.Messages.Create(new()
  {
      Model = Model.ClaudeSonnet5,
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello!" }]
  });

  Console.WriteLine(response.Headers.GetValues("x-amzn-requestid").First()); // AWS request ID
  Console.WriteLine(response.Headers.GetValues("request-id").First()); // Anthropic request ID
  Console.WriteLine(response.Value.Content);
  ```

  ```go Go
  client, err := anthropicaws.NewClient(context.Background(), anthropicaws.ClientConfig{})
  if err != nil {
  	panic(err)
  }

  var response *http.Response
  message, err := client.Messages.New(
  	context.Background(),
  	anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeSonnet5,
  		MaxTokens: 1024,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hello!")),
  		},
  	},
  	option.WithResponseInto(&response),
  )
  if err != nil {
  	panic(err)
  }

  fmt.Println(response.Header.Get("x-amzn-requestid")) // AWS request ID
  fmt.Println(response.Header.Get("request-id"))       // Anthropic request ID
  fmt.Println(message.Content)
  ```

  ```java Java
  import com.anthropic.aws.backends.AwsBackend;
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.core.http.HttpResponseFor;
  import com.anthropic.models.messages.Message;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(AwsBackend.fromEnv())
          .build();

      HttpResponseFor<Message> response = client.messages().withRawResponse().create(
          MessageCreateParams.builder()
              .model(Model.CLAUDE_SONNET_5)
              .maxTokens(1024)
              .addUserMessage("Hello!")
              .build()
      );

      IO.println(response.headers().values("x-amzn-requestid").get(0)); // AWS request ID
      IO.println(response.requestId().orElse(null)); // Anthropic request ID
      IO.println(response.parse().content());
  }
  ```

  ```php PHP
  use Anthropic\Aws\Client;

  $client = new Client();

  $response = $client->messages->raw->create(
      model: 'claude-sonnet-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello!']],
  );

  echo $response->getHeaderLine('x-amzn-requestid') . "\n"; // AWS request ID
  echo $response->getHeaderLine('request-id') . "\n"; // Anthropic request ID
  echo $response->parse()->content;
  ```

  ```ruby Ruby
  # Ruby SDK 目前不支持访问原始响应标头。
  # 如需检查 x-amzn-requestid 标头，请使用其他 SDK 示例之一。
  ```
</CodeGroup>

Anthropic 建议至少以 30 天滚动周期记录您的活动，以便了解用量模式并调查问题。

<Note>
  AWS CloudTrail 在您的 AWS 账户内配置。启用日志记录不会向 AWS 或 Anthropic 提供超出计费和服务运营所需范围的对您内容的访问权限。
</Note>

## 从 Amazon Bedrock 迁移

如果您目前在 Bedrock 上使用 Claude，迁移到 Claude Platform on AWS 需要对您的整个集成进行更改。SigV4 签名仍受支持，但签名上下文、基础 URL、API 格式、模型 ID、SDK 客户端和包、流式传输格式、请求标头以及区域可用性都会发生变化。Claude Platform on AWS 还会配置一个新的 Anthropic 组织。下表总结了这些差异。

### 变化内容

迁移差异取决于您来自哪种 Bedrock 集成。下表同时展示了[当前的 Bedrock 集成](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)（位于 `bedrock-mantle.{region}.api.aws` 的 Messages API）和[旧版 InvokeModel 集成](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)。

| 方面               | 来自 [Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) | 来自 [Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy) | 到 Claude Platform on AWS                                                                                                                                                   |
| ---------------- | ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **基础 URL**       | `bedrock-mantle.{region}.api.aws`                                                                                | `bedrock-runtime.{region}.amazonaws.com`                                                                                      | `aws-external-anthropic.{region}.api.aws`                                                                                                                                  |
| **API 格式**       | 位于 `/anthropic/v1/messages` 的 Messages API                                                                       | Bedrock Converse / InvokeModel                                                                                                | Claude API（`/v1/{endpoint}`）                                                                                                                                               |
| **模型 ID**        | anthropic.claude-haiku-4-5                                                                                       | anthropic.claude-haiku-4-5-20251001-v1:0（带有 `us.` 或 `global.` 推理配置文件前缀）                                                       | claude-haiku-4-5                                                                                                                                                           |
| **SDK 客户端**      | `AnthropicBedrockMantle`                                                                                         | `AnthropicBedrock` / Bedrock SDK                                                                                              | 平台专用客户端（请参阅[安装 SDK](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#install-an-sdk)），处于 beta 阶段                                            |
| **SDK 包**        | `anthropic[bedrock]`、`@anthropic-ai/bedrock-sdk` 等                                                               | `anthropic[bedrock]`、`@anthropic-ai/bedrock-sdk` 或 AWS SDK                                                                    | `anthropic[aws]`、`@anthropic-ai/aws-sdk` 等（请参阅[安装 SDK](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#install-an-sdk)）                    |
| **SigV4 服务名称**   | `bedrock-mantle`                                                                                                 | `bedrock`                                                                                                                     | `aws-external-anthropic`                                                                                                                                                   |
| **流式传输格式**       | SSE                                                                                                              | AWS EventStream                                                                                                               | SSE（与 Claude API 相同）                                                                                                                                                       |
| **工作区标头**        | 不适用                                                                                                              | 不适用                                                                                                                           | 需要 `anthropic-workspace-id`                                                                                                                                                |
| **区域可用性**        | 请参阅 [Amazon Bedrock 区域](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-regions.html)               | 请参阅 [Amazon Bedrock 区域](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-regions.html)                            | 所有 AWS 商业区域                                                                                                                                                                |
| **Anthropic 组织** | 无需                                                                                                               | 无需                                                                                                                            | 注册时创建新组织。现有组织无法转换（请参阅[从现有 Anthropic 组织迁移](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#moving-from-an-existing-anthropic-organization)） |

如果您使用的是当前的 Bedrock 集成，请求正文格式已经是 Messages API。变化之处在于基础 URL、SigV4 服务名称、模型 ID，以及添加 `anthropic-workspace-id` 标头。如果您使用的是旧版 InvokeModel 或 Converse API，您还需要将请求和响应结构重写为 Messages API 格式。有关请求结构映射，请参阅 [Amazon Bedrock 上的 Claude（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)。

### 您将获得

* 通常可在当天访问新模型和功能（请参阅[功能限制](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#features-not-supported)）
* 用于文档生成的 Agent Skills（PowerPoint、Excel、Word、PDF）
* 在 Anthropic 托管沙箱中执行代码
* 通过 `anthropic-beta` 标头使用 beta 功能（请参阅[功能限制](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#features-not-supported)）
* 用于配额可见性和用量分析的 Claude Console
* 直接的 Anthropic 支持
* 作为 SigV4 替代方案的 API 密钥身份验证（请参阅 [API 密钥身份验证](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#api-key-authentication)）

### 保持不变的内容

* AWS IAM 身份验证（SigV4）
* AWS 作为开票方。计费渠道从原生 AWS 服务变为 AWS Marketplace（请参阅[商业注意事项](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#commercial-considerations)）。
* AWS 承诺用量抵扣

### 迁移陷阱

<Warning>
  **请先启用出站 Web 身份联合。** 如果您的 AWS 账户此前未使用过 Claude Platform on AWS，您必须在发出请求之前为每个账户[启用出站 Web 身份联合](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#enable-outbound-web-identity-federation)一次。如果没有此步骤，所有请求都会因联合错误而失败（有关确切的错误和补救措施，请参阅[启用出站 Web 身份联合](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#enable-outbound-web-identity-federation)）。Bedrock 不需要此步骤。
</Warning>

<Warning>
  **零数据保留（ZDR）在 Claude Platform on AWS 上需主动选择加入。** 在 Bedrock 上，AWS 是数据处理方，Anthropic 不保留推理输入或输出。Anthropic 的 ZDR 计划在那里不适用。在 Claude Platform on AWS 上，Anthropic 作为独立的数据处理方处理推理数据，ZDR 遵循第一方 Claude API 模式：可通过您的 Anthropic 客户代表申请获得。在迁移依赖数据保留保证的生产工作负载之前，请确认已加入 ZDR。
</Warning>

### 商业注意事项

* **Anthropic 服务条款：** 使用 Claude Platform on AWS 需要接受 Anthropic 的商业服务条款和使用政策。如果您的组织尚未接受这些条款（例如，如果您仅通过 Bedrock 使用过 Claude），系统会在账户设置期间提示您。请参阅[设置您的账户](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#set-up-your-account)。
* **折扣和私有报价：** 协商折扣和 AWS Marketplace 私有报价不会在 Bedrock 和 Claude Platform on AWS 之间自动转移。请与您的 Anthropic 客户代表合作，为 Claude Platform on AWS 设置商业条款。

## IAM 策略

Claude Platform on AWS 与 AWS IAM 集成以进行访问控制。您可以使用标准 IAM 策略语法，授予或拒绝对特定工作区上特定 API 操作的访问。

SigV4 服务名称和 IAM 操作命名空间为 `aws-external-anthropic`。操作遵循 `aws-external-anthropic:<Action>` 模式（例如 `aws-external-anthropic:CreateInference`）。

### 示例：拒绝批量推理

以下策略允许实时推理，同时阻止批处理：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "aws-external-anthropic:CreateInference",
        "aws-external-anthropic:CountTokens",
        "aws-external-anthropic:GetModel",
        "aws-external-anthropic:ListModels",
        "aws-external-anthropic:GetWorkspace"
      ],
      "Resource": "arn:aws:aws-external-anthropic:*:*:workspace/*"
    },
    {
      "Effect": "Allow",
      "Action": "aws-external-anthropic:ListWorkspaces",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "aws-external-anthropic:CreateBatchInference",
        "aws-external-anthropic:GetBatchInference",
        "aws-external-anthropic:ListBatchInferences"
      ],
      "Resource": "*"
    }
  ]
}
```

`GetBatchInference` 操作同时授权批处理元数据路由和批处理结果路由。拒绝它会阻止这两种读取。有关适用于 ZDR 敏感工作负载的仅 Deny 策略，请参阅 [ZDR 敏感工作区的功能锁定](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#feature-lockdown-for-a-zdr-sensitive-workspace)。

<Note>
  `ListWorkspaces` 是账户作用范围的，因此它出现在一个单独的、带有 `"Resource": "*"` 的 Allow 语句中。在账户作用范围的操作上指定工作区 ARN 没有任何效果（请参阅[配置自动化](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#provisioning-automation)）。

  此策略假定使用 AWS SigV4 身份验证。如果主体使用 API 密钥进行身份验证，还需将 `aws-external-anthropic:CallWithBearerToken` 添加到 `"Resource": "*"` 的 Allow 语句中。`CallWithBearerToken` 是一个无路由的身份验证层操作，不绑定到工作区 ARN。有关双语句模式，请参阅[按客户的工作区隔离](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#per-customer-workspace-isolation)。
</Note>

### 托管策略

AWS 为常见访问模式提供五个托管策略（`AnthropicFullAccess`、`AnthropicReadOnlyAccess`、`AnthropicInferenceAccess`、`AnthropicLimitedAccess` 和 `AnthropicSelfHostedEnvironmentAccess`）。有关每个策略授予的操作、IAM 操作的完整列表、路由到操作的映射以及其他策略示例，请参阅 [Claude Platform on AWS 的 IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#managed-policies)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="功能概览" icon="stack" href="https://platform.claude.com/docs/zh-CN/build-with-claude/overview">
    探索 Claude 的高级功能和能力。
  </Card>

  <Card title="定价" icon="chart" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-platform-on-aws-pricing">
    了解 Claude Platform on AWS 定价和 Claude Consumption Unit 费率。
  </Card>

  <Card title="模型弃用" icon="arrow-clockwise" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    随着更安全、更强大的模型发布，Anthropic 会定期停用旧模型。查看所有 API 弃用信息以及推荐的替代方案。
  </Card>
</CardGroup>

## 其他资源

<CardGroup cols={2}>
  <Card title="Claude Console" icon="browser" href="https://platform.claude.com">
    在 Claude Console 中查看用量、成本和工作区。通过 AWS Console 登录。
  </Card>

  <Card title="Claude in Amazon Bedrock" icon="cloud" href="https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock">
    如果您需要 AWS 作为唯一的数据处理方，请使用由 AWS 运营的 Claude。
  </Card>

  <Card title="AWS Marketplace" icon="coins" href="https://aws.amazon.com/marketplace">
    管理您的 AWS Marketplace 订阅和计费。
  </Card>
</CardGroup>
