---
title: Amazon Bedrock 中的 Claude（Opus 4.7 及更高版本）
url: https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock
description: 通过 Amazon Bedrock 访问 Claude 模型，使用 AWS 原生的身份验证、计费和安全边界。
---

本指南将引导您完成在 Amazon Bedrock 中设置 Claude 并进行 API 调用的过程。Amazon Bedrock 中的 Claude 运行在 AWS 托管的基础设施上，具有零运营人员访问权限（Anthropic 人员无法访问推理基础设施），让您能够完全在 AWS 安全边界内构建敏感应用程序，同时使用与 Anthropic 第一方 API 相同的 Messages API 形式。

<Note>
  本页面介绍 Amazon Bedrock 中的 Claude，它在 AWS 托管的基础设施上通过位于 `/anthropic/v1/messages` 的 Messages API 提供 Claude 服务。之前的 Amazon Bedrock 集成（使用 ARN 版本化模型标识符的 `InvokeModel` 和 `Converse` API）仍然可用，相关文档请参阅 [Amazon Bedrock 上的 Claude（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)。如需在 AWS 上使用由 Anthropic 运营、通过 AWS Marketplace 计费且通常可当日获得新功能的替代方案，请参阅 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)。
</Note>

## 访问权限

Amazon Bedrock 为每个 Claude 模型单独设置访问条件。Claude Fable 5.1、Claude Fable 5、Claude Opus 4.8、Claude Sonnet 5、Claude Opus 4.7 和 Claude Haiku 4.5 向所有 Amazon Bedrock 客户开放。如需了解任何其他模型的当前访问条件，请在 AWS 控制台中查看 [Amazon Bedrock 模型访问](https://console.aws.amazon.com/bedrock/home#/modelaccess)。Claude Mythos Preview 需要通过 [Project Glasswing](https://anthropic.com/glasswing) 获得邀请。有关区域可用性，请参阅[区域](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock#regions)。

## 前提条件

在开始之前，请确保您具备：

* 一个 AWS 账户，并已为您打算使用的 Claude 模型启用 [Amazon Bedrock 模型访问](https://console.aws.amazon.com/bedrock/home#/modelaccess)。
* 已安装并配置 [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)（可选，用于凭证管理）。

Claude Mythos Preview 还需要一个已被 Bedrock Marketplace 团队加入允许列表的专用 AWS 账户。您的 Anthropic 客户经理可以提交您的账户 ID 以加入允许列表（通常在 24 小时内处理完成），完成后 AWS 会发送一封欢迎邮件。

## 身份验证

Amazon Bedrock 中的 Claude 支持三种身份验证路径。请选择最符合您安全要求的一种。

### Bedrock 服务角色（推荐）

使用带有 AWS 托管密钥的 Bedrock 服务角色，以获得最安全、长期有效的访问：

<Steps>
  <Step title="管理员：配置服务角色">
    AWS 管理员配置一个 Bedrock 服务角色，并授予开发人员对该服务角色 ARN 的 `iam:PassRole` 权限。
  </Step>

  <Step title="开发人员：传递角色">
    调用 API 时，Bedrock 会代表您代入该服务角色。有关如何将角色与您的请求关联，请参阅 [Amazon Bedrock 文档](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-mantle.html)。
  </Step>
</Steps>

### IAM 代入角色

用于身份联合访问，会话最长 12 小时：

<Steps>
  <Step title="管理员：配置 IAM 角色">
    创建一个作用范围限定于您的 Claude 模型的 IAM 角色。信任策略指定您的身份提供商（SAML、OIDC 或 AWS Identity Center）。权限策略仅对允许的模型 ARN 授予 `bedrock-mantle:CreateInference`。
  </Step>

  <Step title="开发人员：进行身份验证并代入角色">
    通过您的企业身份提供商进行身份验证，然后代入该 IAM 角色。AWS STS 会颁发临时凭证，SDK 或 CLI 使用这些凭证对请求进行签名。
  </Step>
</Steps>

### Bearer 令牌

用于无需 IAM 角色的短期访问（最长 12 小时，最不推荐）：

<Steps>
  <Step title="管理员：限制令牌类型">
    通过附加一个策略来阻止长期密钥：除非 `bedrock:BearerTokenType` 条件匹配短期令牌，否则拒绝 `bedrock:CallWithBearerToken`。
  </Step>

  <Step title="开发人员：生成令牌">
    使用 `aws-bedrock-token-generator` CLI 生成一个 bearer 令牌。在每个请求的 `x-api-key` 标头中传递该令牌。
  </Step>
</Steps>

## 安装 SDK

Anthropic 的[客户端 SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview) 通过 Bedrock 专用的包或模块支持 Amazon Bedrock 中的 Claude。

<Tabs>
  <Tab title="Python">
    ```bash
    pip install -U "anthropic[bedrock]"
    ```
  </Tab>

  <Tab title="TypeScript">
    ```bash
    npm install @anthropic-ai/bedrock-sdk
    ```
  </Tab>

  <Tab title="C#">
    ```bash
    dotnet add package Anthropic.Bedrock
    ```
  </Tab>

  <Tab title="Go">
    ```bash
    go get github.com/anthropics/anthropic-sdk-go/bedrock
    ```
  </Tab>

  <Tab title="Java">
    <Tabs>
      <Tab title="Gradle">
        ```kotlin
        implementation("com.anthropic:anthropic-java-bedrock:2.58.0")
        ```
      </Tab>

      <Tab title="Maven">
        ```xml
        <dependency>
            <groupId>com.anthropic</groupId>
            <artifactId>anthropic-java-bedrock</artifactId>
            <version>2.58.0</version>
        </dependency>
        ```
      </Tab>
    </Tabs>
  </Tab>

  <Tab title="PHP">
    ```bash
    composer require anthropic-ai/sdk aws/aws-sdk-php
    ```
  </Tab>

  <Tab title="Ruby">
    ```bash
    # Gemfile
    gem "anthropic"
    gem "aws-sdk-core"
    ```
  </Tab>
</Tabs>

## 发出您的第一个请求

端点遵循 `https://bedrock-mantle.{region}.api.aws/anthropic/v1/messages` 模式。与基于 `InvokeModel` 的集成不同，此端点使用标准的 SSE "streaming"（流式传输），并且请求体形式与 Anthropic 第一方 API 相同。

SDK 按照标准的 AWS 优先级顺序解析凭证和区域：首先是构造函数参数，然后是环境变量（`AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`、`AWS_SESSION_TOKEN`、`AWS_REGION`），最后是 AWS 配置文件和凭证链（SSO、代入角色、ECS 任务角色、IMDS）。

<Tabs>
  <Tab title="cURL">
    ```bash
    curl https://bedrock-mantle.us-east-1.api.aws/anthropic/v1/messages \
      --aws-sigv4 "aws:amz:us-east-1:bedrock-mantle" \
      --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
      -H "x-amz-security-token: $AWS_SESSION_TOKEN" \
      -H "content-type: application/json" \
      -H "anthropic-version: 2023-06-01" \
      -d '{
        "model": "anthropic.claude-opus-5",
        "max_tokens": 1024,
        "messages": [
          {"role": "user", "content": "Hello, Claude"}
        ]
      }'
    ```
  </Tab>

  <Tab title="CLI">
    `ant` CLI 不支持 Amazon Bedrock。请使用 cURL 或 SDK。
  </Tab>

  <Tab title="Python">
    ```python
    from anthropic import AnthropicBedrockMantle

    client = AnthropicBedrockMantle(aws_region="us-east-1")

    message = client.messages.create(
        model="anthropic.claude-opus-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Hello, Claude"}],
    )

    print(next(block.text for block in message.content if block.type == "text"))
    ```
  </Tab>

  <Tab title="TypeScript">
    ```typescript
    import { AnthropicBedrockMantle } from "@anthropic-ai/bedrock-sdk";

    const client = new AnthropicBedrockMantle({
      awsRegion: "us-east-1"
    });

    const message = await client.messages.create({
      model: "anthropic.claude-opus-5",
      max_tokens: 1024,
      messages: [{ role: "user", content: "Hello, Claude" }]
    });

    const textBlock = message.content.find((block) => block.type === "text");
    if (textBlock) {
      console.log(textBlock.text);
    }
    ```
  </Tab>

  <Tab title="C#">
    ```csharp
    using Anthropic.Bedrock;
    using Anthropic.Models.Messages;

    var client = new AnthropicBedrockMantleClient(new() { AwsRegion = "us-east-1" });

    var message = await client.Messages.Create(new()
    {
        Model = "anthropic.claude-opus-5",
        MaxTokens = 1024,
        Messages = [new() { Role = Role.User, Content = "Hello, Claude" }],
    });

    foreach (var item in message.Content)
    {
        if (item.Value is TextBlock block)
        {
            Console.WriteLine(block.Text);
            break;
        }
    }
    ```
  </Tab>

  <Tab title="Go">
    ```go
    client, err := bedrock.NewMantleClient(context.Background(), bedrock.MantleClientConfig{
    	AWSRegion: "us-east-1",
    })
    if err != nil {
    	panic(err)
    }

    message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
    	Model:     "anthropic.claude-opus-5",
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
  </Tab>

  <Tab title="Java">
    ```java
    import com.anthropic.bedrock.backends.BedrockMantleBackend;
    import com.anthropic.client.AnthropicClient;
    import com.anthropic.client.okhttp.AnthropicOkHttpClient;
    import com.anthropic.models.messages.ContentBlock;
    import com.anthropic.models.messages.Message;
    import com.anthropic.models.messages.MessageCreateParams;

    void main() {
        AnthropicClient client = AnthropicOkHttpClient.builder()
            .backend(BedrockMantleBackend.fromEnv())
            .build();

        Message message = client.messages().create(
            MessageCreateParams.builder()
                .model("anthropic.claude-opus-5")
                .maxTokens(1024)
                .addUserMessage("Hello, Claude")
                .build()
        );

        message.content().stream()
                .filter(ContentBlock::isText)
                .findFirst()
                .ifPresent(block -> IO.println(block.asText().text()));
    }
    ```
  </Tab>

  <Tab title="PHP">
    ```php
    use Anthropic\Bedrock\MantleClient;

    $client = new MantleClient(awsRegion: 'us-east-1');

    $message = $client->messages->create(
        model: 'anthropic.claude-opus-5',
        maxTokens: 1024,
        messages: [
            ['role' => 'user', 'content' => 'Hello, Claude'],
        ],
    );

    echo array_find($message->content, fn ($block) => $block->type === 'text')->text;
    ```
  </Tab>

  <Tab title="Ruby">
    ```ruby
    require "anthropic"

    client = Anthropic::BedrockMantleClient.new(aws_region: "us-east-1")

    message = client.messages.create(
      model: "anthropic.claude-opus-5",
      max_tokens: 1024,
      messages: [{role: "user", content: "Hello, Claude"}]
    )

    puts message.content.find { it.type == :text }.text
    ```
  </Tab>
</Tabs>

<Tip>
  您也可以使用标准的 `Anthropic` 客户端：将 `base_url` 设置为 `https://bedrock-mantle.{region}.api.aws/anthropic`，并将您的 bearer 令牌作为 `api_key` 传递。此路径仅支持 bearer 令牌身份验证。SigV4 签名需要使用专用客户端。
</Tip>

## 支持的模型

Amazon Bedrock 中的 Claude 的模型 ID 带有 `anthropic.` 提供商前缀。模型的功能和行为记录在[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)页面上。

| 模型                    | 模型 ID                           | 访问权限                                                                                                |
| --------------------- | ------------------------------- | --------------------------------------------------------------------------------------------------- |
| Claude Fable 5.1      | anthropic.claude-fable-5-1      | 开放                                                                                                  |
| Claude Fable 5        | anthropic.claude-fable-5        | 开放                                                                                                  |
| Claude Opus 5         | anthropic.claude-opus-5         | 请参阅[访问权限](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock#access) |
| Claude Opus 4.8       | anthropic.claude-opus-4-8       | 开放                                                                                                  |
| Claude Opus 4.7       | anthropic.claude-opus-4-7       | 开放                                                                                                  |
| Claude Sonnet 5       | `anthropic.claude-sonnet-5`     | 开放                                                                                                  |
| Claude Haiku 4.5      | anthropic.claude-haiku-4-5      | 开放                                                                                                  |
| Claude Mythos Preview | anthropic.claude-mythos-preview | 仅限受邀（[Project Glasswing](https://anthropic.com/glasswing)）                                          |

在 Amazon Bedrock 上使用 Claude Fable 5.1 时，请使用 Claude Code 2.1.255 或更高版本；运行 `claude update` 进行升级。

<Tip>
  正在升级到更新的 Claude 模型？在 Claude Code 中，运行 `/claude-api migrate` 即可在您的整个代码库中应用模型 ID 替换和破坏性参数变更。该技能会检测您的代码所面向的云平台，并针对该平台调整模型 ID 格式和功能变更。请参阅[迁移到更新的 Claude 模型](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model)。
</Tip>

## 功能支持

有关包含 Amazon Bedrock 可用性的完整功能列表，请参阅[功能概览](https://platform.claude.com/docs/zh-CN/build-with-claude/overview)。

### 支持的功能亮点

* [Messages API](https://platform.claude.com/docs/zh-CN/api/messages/create)（`/anthropic/v1/messages`）
* [提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)
* [思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)
* [工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)，包括 [Bash 工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/bash-tool)、[计算机使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)、[记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)和[文本编辑器工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/text-editor-tool)
* [引用](https://platform.claude.com/docs/zh-CN/build-with-claude/citations)

### 不支持的功能

* [结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)
* 输入源（图像和文档的 URL 源、Files API）
* 服务器端工具（代码执行、网页搜索、网页抓取、advisor）
* 智能体基础设施（Agent Skills、MCP 连接器、程序化工具调用）
* API 端点（Message Batches、Models、Admin、Compliance、Usage and Cost）
* Claude Managed Agents
* 服务器端回退（[`fallbacks` 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)；请改用[客户端回退模式](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#client-side-fallback)）
* [计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)和[浏览器使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)工具集（`computer_toolset_20260801` 和 `browser_toolset_20260801` 目前在 Amazon Bedrock 上不可用；beta 版计算机使用工具版本仍然可用）

## 区域

Amazon Bedrock 中的 Claude 在以下 AWS 区域可用。Amazon Bedrock 提供两种端点类型：

* **全球（Global）：** 在所有可用区域之间动态路由，以实现最大可用性。无价格溢价。
* **区域（Regional）：** 端点解析到您指定的单个 AWS 区域，以满足数据驻留要求。区域端点相比全球端点有 10% 的价格溢价。如需在某一地理范围内跨多个区域路由，请使用[推理配置文件](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)（US、EU、JP 或 AU）。表中标记为 **In-region only**（仅限区域内）的区域支持无需推理配置文件的直接单区域路由。

全球端点适用于 Claude Fable 5.1、Claude Fable 5、Claude Opus 5、Claude Opus 4.8、Claude Opus 4.7、Claude Sonnet 5 和 Claude Haiku 4.5。对于 Claude Fable 5.1，区域端点目前仅在 `us-east-1` 可用。Claude Mythos Preview 仅支持区域端点，并在 `us-east-1` 可用。

| AWS 区域           | 位置            | 端点类型                       |
| ---------------- | ------------- | -------------------------- |
| `af-south-1`     | 非洲（开普敦）       | Global                     |
| `ap-northeast-1` | 亚太地区（东京）      | Global, JP, In-region only |
| `ap-northeast-2` | 亚太地区（首尔）      | Global                     |
| `ap-northeast-3` | 亚太地区（大阪）      | Global, JP                 |
| `ap-south-1`     | 亚太地区（孟买）      | Global                     |
| `ap-south-2`     | 亚太地区（海得拉巴）    | Global                     |
| `ap-southeast-1` | 亚太地区（新加坡）     | Global                     |
| `ap-southeast-2` | 亚太地区（悉尼）      | Global, AU                 |
| `ap-southeast-3` | 亚太地区（雅加达）     | Global                     |
| `ap-southeast-4` | 亚太地区（墨尔本）     | Global, AU, In-region only |
| `ca-central-1`   | 加拿大（中部）       | Global, US                 |
| `ca-west-1`      | 加拿大西部（卡尔加里）   | Global                     |
| `eu-central-1`   | 欧洲（法兰克福）      | Global, EU                 |
| `eu-central-2`   | 欧洲（苏黎世）       | Global, EU                 |
| `eu-north-1`     | 欧洲（斯德哥尔摩）     | Global, EU, In-region only |
| `eu-south-1`     | 欧洲（米兰）        | Global, EU                 |
| `eu-south-2`     | 欧洲（西班牙）       | Global, EU                 |
| `eu-west-1`      | 欧洲（爱尔兰）       | Global, EU, In-region only |
| `eu-west-2`      | 欧洲（伦敦）        | Global, EU                 |
| `eu-west-3`      | 欧洲（巴黎）        | Global, EU                 |
| `il-central-1`   | 以色列（特拉维夫）     | Global                     |
| `me-central-1`   | 中东（阿联酋）       | Global                     |
| `sa-east-1`      | 南美洲（圣保罗）      | Global                     |
| `us-east-1`      | 美国东部（弗吉尼亚北部）  | Global, US, In-region only |
| `us-east-2`      | 美国东部（俄亥俄）     | Global, US, In-region only |
| `us-west-1`      | 美国西部（加利福尼亚北部） | Global, US                 |
| `us-west-2`      | 美国西部（俄勒冈）     | Global, US, In-region only |

## 配额

默认配额为每分钟 200 万输入令牌（TPM）。您可以申请最高 500 万输入 TPM 和 50 万输出 TPM，无需 Anthropic 额外批准。AWS 在 Bedrock 端强制执行每分钟请求数（RPM）限制；如需调整 RPM，请联系 AWS 支持。

## 数据保留

此产品的数据处理由 Amazon Bedrock 管理。有关详细信息，请参阅 [Amazon Bedrock 中的数据保护](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html)。

## 监控和日志记录

Amazon Bedrock 中的 Claude 会向 CloudWatch 和 CloudTrail 发送日志。Anthropic 建议至少以 30 天滚动方式保留活动日志，以便了解使用模式并调查潜在问题。

## 支持

如需支持，请联系 **[bedrock-ant-eap@amazon.com](mailto:bedrock-ant-eap@amazon.com)**。请附上您的 AWS 账户 ID 以及任何失败的 API 响应中的 `request-id`。

<Note>
  **Claude Mythos Preview** 是一个研究预览模型，面向 Amazon Bedrock 上的受邀客户提供。有关更多信息，请参阅 [Project Glasswing](https://anthropic.com/glasswing)。
</Note>
