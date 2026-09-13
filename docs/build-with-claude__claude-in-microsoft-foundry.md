---
title: Microsoft Foundry 中的 Claude
url: https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry
description: 通过 Microsoft Foundry 使用 Azure 原生端点和身份验证访问 Claude 模型。
---

本指南向您展示如何使用 Anthropic 的某个客户端 SDK 或直接 HTTP 请求，在 Microsoft Foundry 中设置并调用 Claude API。当您在 Microsoft Foundry 中访问 Claude 时，您的 Claude 使用费用将通过 Azure Marketplace 计费。您可以使用包括 Claude Fable 5.1、Claude Opus 5、Claude Opus 4.8 和 Claude Sonnet 5 在内的 Claude 模型，以及诸如 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)之类的功能，同时通过您的 Azure 订阅管理成本。

Claude 在 Foundry 资源中提供 Global Standard 和 US Data Zone Standard 部署类型，通过 Azure Marketplace 以 Claude Consumption Units 计费。有关详细信息，请访问 [Microsoft Foundry 中的 Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-in-microsoft-foundry-pricing)。

## 托管选项

Microsoft Foundry 中的 Claude 模型提供两种托管选项。您在配置部署时选择托管选项。

|        | 托管在 Azure 上                           | 托管在 Anthropic 上                                                                                                                                                      |
| ------ | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 推理运行位置 | 在 Azure 基础设施上运行的 Anthropic 运营服务       | 在 Anthropic 基础设施上运行的 Anthropic 运营服务                                                                                                                                  |
| 模型可用性  | Opus、Sonnet 和 Haiku 系列中的最新模型          | Microsoft Foundry 上可用的所有 Claude 模型                                                                                                                                   |
| 部署类型   | Global Standard、US Data Zone Standard | Global Standard                                                                                                                                                      |
| 推荐用于   | 大多数工作负载                               | [访问尚未托管在 Azure 上的功能或模型](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry#additional-features-not-supported-when-hosted-on-azure) |

<Note>
  Anthropic 作为 Microsoft 的独立处理方。通过 Microsoft Foundry 使用 Claude 的客户受 Anthropic 的数据使用条款约束。对于托管在 Azure 上的部署，提示和补全内容保留在 Azure 内。只有使用元数据和被 Anthropic 安全系统标记的内容才会流出到 Anthropic。Anthropic 将继续提供其安全和数据承诺。
</Note>

## 前提条件

在开始之前，请确保您具备：

* 一个有效的 Azure 订阅
* 对 [Foundry 门户](https://ai.azure.com/)的访问权限
* 已安装 [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)（Entra ID cURL 示例需要，其他情况下可选）
* 允许您使用该资源的 Azure RBAC 角色，例如 **Foundry User**（原 Azure AI User）或 **Cognitive Services User**

## 安装 SDK

Anthropic 的[客户端 SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview) 通过特定于平台的包或客户端类支持 Foundry。本页上的示例还展示了使用 cURL 和 ant CLI 的请求。要设置 CLI，请参阅 [CLI 快速入门](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/quickstart)。

<Note>
  Foundry 受 C#、Java、PHP、Python 和 TypeScript SDK 支持。Foundry 目前在 Go 和 Ruby SDK 中不可用。
</Note>

<Tabs>
  <Tab title="Python">
    ```bash
    pip install -U "anthropic"

    # 对于 Entra ID 身份验证，还需安装 Azure Identity 库
    pip install azure-identity
    ```
  </Tab>

  <Tab title="TypeScript">
    ```bash
    npm install @anthropic-ai/foundry-sdk

    # 对于 Entra ID 身份验证，还需安装 Azure Identity 库
    npm install @azure/identity
    ```
  </Tab>

  <Tab title="C#">
    ```bash
    dotnet add package Anthropic.Foundry
    ```
  </Tab>

  <Tab title="Go">
    ```bash
    # Go SDK 尚未原生支持 Foundry（请参阅身份验证
    # 示例，了解如何使用标准 Go SDK 作为变通方案）
    go get github.com/anthropics/anthropic-sdk-go
    ```
  </Tab>

  <Tab title="Java">
    <Tabs>
      <Tab title="Gradle">
        ```kotlin
        implementation("com.anthropic:anthropic-java-foundry:2.58.0")

        // 对于 Entra ID 身份验证，还需添加 Azure Identity 库
        implementation("com.azure:azure-identity:1.18.3")
        ```
      </Tab>

      <Tab title="Maven">
        ```xml
        <dependency>
            <groupId>com.anthropic</groupId>
            <artifactId>anthropic-java-foundry</artifactId>
            <version>2.58.0</version>
        </dependency>
        <!-- For Entra ID authentication, also add the Azure Identity library -->
        <dependency>
            <groupId>com.azure</groupId>
            <artifactId>azure-identity</artifactId>
            <version>1.18.3</version>
        </dependency>
        ```
      </Tab>
    </Tabs>
  </Tab>

  <Tab title="PHP">
    ```bash
    composer require "anthropic-ai/sdk" "guzzlehttp/guzzle:^7"
    ```
  </Tab>

  <Tab title="Ruby">
    ```bash
    # Ruby SDK 尚未原生支持 Foundry（请参阅 Authentication
    # 示例，了解如何使用标准 Ruby SDK 作为变通方案）
    # Gemfile
    gem "anthropic"
    ```
  </Tab>
</Tabs>

## 预配

Foundry 使用两级层次结构：**资源**包含您的安全和计费配置，而**部署**是您通过 API 调用的模型实例。您将首先创建一个 Foundry 资源，然后在其中创建一个或多个 Claude 部署。

### 预配 Foundry 资源

创建一个 Foundry 资源，这是在 Azure 中使用和管理服务所必需的。您可以按照这些说明创建 [Foundry 资源](https://learn.microsoft.com/en-us/azure/ai-services/multi-service-resource?pivots=azportal#create-a-new-azure-ai-foundry-resource)。或者，您可以从创建 [Foundry 项目](https://learn.microsoft.com/en-us/azure/foundry/how-to/create-projects)开始，这涉及创建一个 Foundry 资源。

要预配您的资源：

1. 导航到 [Foundry 门户](https://ai.azure.com/)。
2. 创建一个新的 Foundry 资源或选择一个现有资源。
3. 使用 Azure 颁发的 API 密钥或 Entra ID（原 Azure Active Directory）配置访问管理，以进行基于角色的访问控制。
4. 可选地将资源配置为专用网络（Azure Virtual Network）的一部分，以限制对您资源的网络访问。
5. 记下您的资源名称。您将在 API 端点中将其用作 `{resource}`（例如，`https://{resource}.services.ai.azure.com/anthropic/v1/*`）。

### 创建 Foundry 部署

创建资源后，部署一个 Claude 模型以使其可用于 API 调用。以下步骤描述了新的 Foundry 门户（**New Foundry** 切换开关已打开）：

1. 登录 Foundry 门户。从门户主页，选择右上角导航中的 **Discover**，然后选择左侧窗格中的 **Models** 以打开模型目录。

2. 搜索并选择一个 Claude 模型（例如，claude-opus-5）。无论支持多少种托管选项，每个模型在目录中只出现一次。

3. 在模型卡片上，选择 **Deploy**，然后选择 **Custom settings** 以打开部署设置窗格。如果您改为选择 **Default settings**，则对于在两种托管选项中都可用的模型，部署将自动配置为托管在 Azure 上。

4. 在您的第一个 Claude 部署时，查看 Azure Marketplace 条款，选择一个行业，然后选择 **Agree and Proceed** 以接受条款并订阅 Azure Marketplace 产品。

5. 配置部署：

   * **Deployment name：** 默认为模型 ID，但您可以自定义它（例如，`my-claude-deployment`）。部署名称在创建后无法更改。
   * **Region scope：** 选择 Global，或者对于托管在 Azure 上的模型，选择 Data Zone。选择 Data Zone 会创建一个 US Data Zone Standard 部署，它将推理保留在美国境内，等同于在 Claude API 上设置 [`inference_geo: "us"`](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency#inference-geo)。
   * **Model version：** 展开 **Model version settings** 并从 **Model version** 下拉菜单中选择一个版本。每个[托管选项](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry#hosting-options)都列为一个单独的模型版本，并标有其托管选项（例如，版本 1 为托管在 Anthropic 上，版本 2 为托管在 Azure 上）。

6. 选择 **Deploy** 并等待预配完成。

7. 部署完成后，选择右上角导航中的 **Build**，然后选择左侧窗格中的 **Models**，并打开您的部署。**Details** 选项卡显示 **Target URI**（您的端点 URL）和 **Key**（您的 API 密钥）。

如果 **New Foundry** 切换开关已关闭，则您处于经典门户布局中。在那里，打开左侧窗格中的 **Model catalog** 以查找和部署模型，并打开 **Models + endpoints**（在 **My assets** 下）以查看您的部署及其端点详细信息。

<Note>
  您选择的部署名称将成为您在 API 请求的 `model` 参数中传递的值。您可以使用不同的名称创建同一模型的多个部署，以管理单独的配置或速率限制。
</Note>

## 身份验证

Microsoft Foundry 中的 Claude 支持两种身份验证方法：API 密钥和 Entra ID 令牌。两种方法都使用格式为 `https://{resource}.services.ai.azure.com/anthropic/v1/*` 的 Azure 托管端点。

### API 密钥身份验证

预配您的 Foundry Claude 资源后，您可以从 Foundry 门户获取 API 密钥：

1. 在 Foundry 门户中，选择右上角导航中的 **Build**，然后选择左侧窗格中的 **Models**。
2. 打开您的 Claude 部署并选择 **Details** 选项卡。
3. 复制 **Key** 值（并记下您端点的 **Target URI**）。
4. 在您的请求中使用 `api-key` 或 `x-api-key` 标头，或将其提供给 SDK。

Foundry SDK 需要一个 API 密钥以及资源名称或基础 URL。如果定义了以下环境变量，C#、Java、PHP、Python 和 TypeScript SDK 会自动从中读取：

* `ANTHROPIC_FOUNDRY_API_KEY` - 您的 API 密钥
* `ANTHROPIC_FOUNDRY_RESOURCE` - 您的资源名称（例如，`example-resource`）
* `ANTHROPIC_FOUNDRY_BASE_URL` - 资源名称的替代方案：完整的基础 URL（例如，`https://example-resource.services.ai.azure.com/anthropic/`）。C# SDK 不读取此变量：它始终从资源名称构造基础 URL。

<Note>
  `resource` 和 `base_url` 参数是互斥的。请提供资源名称（SDK 使用它将 URL 构造为 `https://{resource}.services.ai.azure.com/anthropic/`）或直接提供完整的基础 URL。
</Note>

**使用 API 密钥的示例：**

<CodeGroup>
  ```bash cURL
  curl https://{resource}.services.ai.azure.com/anthropic/v1/messages \
    -H "content-type: application/json" \
    -H "api-key: YOUR_AZURE_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [
        {"role": "user", "content": "Hello!"}
      ]
    }'
  ```

  ```bash CLI
  # ant 读取 ANTHROPIC_API_KEY 并将其作为 x-api-key 发送，Foundry 接受该方式
  export ANTHROPIC_API_KEY="YOUR_AZURE_API_KEY"

  ant messages create \
    --base-url https://example-resource.services.ai.azure.com/anthropic \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello!"}' \
    --transform content
  ```

  ```python Python
  import os
  from anthropic import AnthropicFoundry

  client = AnthropicFoundry(
      api_key=os.environ.get("ANTHROPIC_FOUNDRY_API_KEY"),
      resource="example-resource",  # your resource name
  )

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello!"}],
  )
  print(message.content)
  ```

  ```typescript TypeScript
  import AnthropicFoundry from "@anthropic-ai/foundry-sdk";

  const client = new AnthropicFoundry({
    apiKey: process.env.ANTHROPIC_FOUNDRY_API_KEY,
    resource: "example-resource" // your resource name
  });

  const message = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello!" }]
  });
  console.log(message.content);
  ```

  ```csharp C#
  using Anthropic.Foundry;
  using Anthropic.Models.Messages;

  var client = new AnthropicFoundryClient(
      new AnthropicFoundryApiKeyCredentials(
          Environment.GetEnvironmentVariable("ANTHROPIC_FOUNDRY_API_KEY")!,
          "example-resource"
      )
  );

  var response = await client.Messages.Create(new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello!" }],
  });

  Console.WriteLine(
      string.Join("", response.Content
          .Select(block => block.Value)
          .OfType<TextBlock>()
          .Select(textBlock => textBlock.Text)));
  ```

  ```go Go
  // Go SDK 尚未原生支持 Foundry。本示例使用
  // 标准 Go SDK 作为变通方案。WithoutEnvironmentDefaults 可防止
  // 客户端从环境中读取 ANTHROPIC_API_KEY 或 ANTHROPIC_AUTH_TOKEN，
  // 从而避免将 Claude API 凭据发送到您的 Foundry
  // 端点。Foundry 不支持的功能会在服务器端而非
  // 客户端失败。如需完整的 Foundry 支持，请使用 C#、Java、PHP、
  // Python 或 TypeScript SDK。
  package main

  import (
  	"context"
  	"fmt"
  	"os"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/option"
  )

  func main() {
  	client := anthropic.NewClient(
  		option.WithoutEnvironmentDefaults(),
  		option.WithBaseURL("https://example-resource.services.ai.azure.com/anthropic"),
  		option.WithAPIKey(os.Getenv("ANTHROPIC_FOUNDRY_API_KEY")),
  	)

  	message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		Model:     "claude-opus-5",
  		MaxTokens: 1024,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hello!")),
  		},
  	})
  	if err != nil {
  		panic(err)
  	}
  	fmt.Println(message.Content)
  }
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.foundry.backends.FoundryBackend;
  import com.anthropic.models.messages.MessageCreateParams;

  void main() {
      // 需要环境变量：ANTHROPIC_FOUNDRY_API_KEY、ANTHROPIC_FOUNDRY_RESOURCE
      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(FoundryBackend.fromEnv())
          .build();

      MessageCreateParams params = MessageCreateParams.builder()
          .model("claude-opus-5")
          .maxTokens(1024)
          .addUserMessage("Hello!")
          .build();

      client.messages().create(params).content().stream()
          .flatMap(block -> block.text().stream())
          .forEach(textBlock -> IO.println(textBlock.text()));
  }
  ```

  ```php PHP
  use Anthropic\Foundry;

  $client = Foundry\Client::withCredentials(
      apiKey: getenv('ANTHROPIC_FOUNDRY_API_KEY'),
      baseUrl: 'https://example-resource.services.ai.azure.com/anthropic',
  );

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          ['role' => 'user', 'content' => 'Hello!']
      ],
      model: 'claude-opus-5',
  );
  echo array_find($message->content, fn ($block) => $block->type === 'text')->text;
  ```

  ```ruby Ruby
  # Ruby SDK 尚未原生支持 Foundry。本示例使用
  # 标准 Ruby SDK 作为变通方案。请显式传递凭据：如果不传递，
  # 客户端会回退到 ANTHROPIC_API_KEY 或
  # ANTHROPIC_AUTH_TOKEN 环境变量，并可能将 Claude API
  # 凭据发送到您的 Foundry 端点。Foundry 不支持的功能
  # 会在服务器端而非客户端失败。如需完整的
  # Foundry 支持，请使用 C#、Java、PHP、Python 或 TypeScript SDK。
  require "anthropic"

  client = Anthropic::Client.new(
    base_url: "https://example-resource.services.ai.azure.com/anthropic",
    api_key: ENV.fetch("ANTHROPIC_FOUNDRY_API_KEY")
  )

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello!"}]
  )

  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

<Warning>
  请妥善保管您的 API 密钥。切勿将它们提交到版本控制或公开共享。任何能够访问您 API 密钥的人都可以通过您的 Foundry 资源向 Claude 发出请求。
</Warning>

### Microsoft Entra 身份验证

Entra ID 身份验证让您可以使用 Azure RBAC 管理访问权限，与您组织的身份管理集成，并避免手动处理 API 密钥。要使用 Entra ID 令牌：

1. 为您的 Foundry 资源启用 [Microsoft Entra ID 身份验证](https://learn.microsoft.com/en-us/azure/ai-foundry/model-inference/how-to/configure-entra-id)。
2. 从 Entra ID 获取访问令牌。
3. 在 `Authorization: Bearer {TOKEN}` 标头中使用该令牌。

**使用 Entra ID 的示例：**

<CodeGroup>
  ```bash cURL
  # 获取 Microsoft Entra ID 令牌
  ACCESS_TOKEN=$(az account get-access-token --resource https://ai.azure.com --query accessToken -o tsv)

  # 使用令牌发出请求。将 {resource} 替换为您的资源名称
  curl https://{resource}.services.ai.azure.com/anthropic/v1/messages \
    -H "content-type: application/json" \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [
        {"role": "user", "content": "Hello!"}
      ]
    }'
  ```

  ```bash CLI
  # ant CLI 可以通过 --auth-token 发送 bearer 令牌，但已设置的
  # ANTHROPIC_API_KEY 环境变量会优先于它（CLI
  # 仅打印一条控制台通知），因此您的请求可能会使用
  # 错误的凭据进行身份验证。对于 Entra ID 流程，请改用 cURL 示例或某个
  # SDK 示例。
  ```

  ```python Python
  from anthropic import AnthropicFoundry
  from azure.identity import DefaultAzureCredential, get_bearer_token_provider

  # 使用令牌提供程序模式获取 Microsoft Entra ID 令牌
  token_provider = get_bearer_token_provider(
      DefaultAzureCredential(), "https://ai.azure.com/.default"
  )

  # 使用 Entra ID 身份验证创建客户端
  client = AnthropicFoundry(
      resource="example-resource",  # your resource name
      azure_ad_token_provider=token_provider,  # Use token provider for Entra ID auth
  )

  # 发起请求
  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello!"}],
  )
  print(message.content)
  ```

  ```typescript TypeScript
  import AnthropicFoundry from "@anthropic-ai/foundry-sdk";
  import { DefaultAzureCredential, getBearerTokenProvider } from "@azure/identity";

  // 使用令牌提供程序模式获取 Entra ID 令牌
  const credential = new DefaultAzureCredential();
  const tokenProvider = getBearerTokenProvider(credential, "https://ai.azure.com/.default");

  // 使用 Entra ID 身份验证创建客户端
  const client = new AnthropicFoundry({
    resource: "example-resource", // your resource name
    azureADTokenProvider: tokenProvider // Use token provider for Entra ID auth
  });

  // 发起请求
  const message = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello!" }]
  });
  console.log(message.content);
  ```

  ```csharp C#
  using Anthropic.Foundry;
  using Anthropic.Models.Messages;
  using Azure.Identity;

  var client = new AnthropicFoundryClient(
      new AnthropicFoundryIdentityTokenCredentials(
          new DefaultAzureCredential(),
          "example-resource"
      )
  );

  var response = await client.Messages.Create(new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello!" }],
  });

  Console.WriteLine(
      string.Join("", response.Content
          .Select(block => block.Value)
          .OfType<TextBlock>()
          .Select(textBlock => textBlock.Text)));
  ```

  ```go Go
  // Go SDK 尚未原生支持 Foundry。此示例使用
  // 标准 Go SDK 作为变通方案，并使用静态 Entra ID 令牌：内置不支持自动
  // 令牌刷新，因此您的应用程序必须自行刷新令牌
  // （它们通常在 1 小时后过期）。WithoutEnvironmentDefaults
  // 可防止客户端同时从环境中读取 ANTHROPIC_API_KEY 或
  // ANTHROPIC_AUTH_TOKEN，并向您的 Foundry 端点发送 Claude API
  // 凭据。如需完整的 Foundry 支持，请使用
  // C#、Java、PHP、Python 或 TypeScript SDK。
  package main

  import (
  	"context"
  	"fmt"
  	"os"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/option"
  )

  func main() {
  	// 获取 Entra ID 访问令牌，例如使用 Azure CLI：
  	//   az account get-access-token --resource https://ai.azure.com \
  	//     --query accessToken -o tsv
  	client := anthropic.NewClient(
  		option.WithoutEnvironmentDefaults(),
  		option.WithBaseURL("https://example-resource.services.ai.azure.com/anthropic"),
  		option.WithAuthToken(os.Getenv("AZURE_ACCESS_TOKEN")),
  	)

  	message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		Model:     "claude-opus-5",
  		MaxTokens: 1024,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hello!")),
  		},
  	})
  	if err != nil {
  		panic(err)
  	}
  	fmt.Println(message.Content)
  }
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.foundry.backends.FoundryBackend;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.azure.identity.AuthenticationUtil;
  import com.azure.identity.DefaultAzureCredentialBuilder;
  import java.util.function.Supplier;

  void main() {
      Supplier<String> bearerTokenSupplier = AuthenticationUtil.getBearerTokenSupplier(
          new DefaultAzureCredentialBuilder().build(),
          "https://ai.azure.com/.default"
      );

      AnthropicClient client = AnthropicOkHttpClient.builder()
          .backend(FoundryBackend.builder()
              .bearerTokenSupplier(bearerTokenSupplier)
              .resource("example-resource")
              .build())
          .build();

      MessageCreateParams params = MessageCreateParams.builder()
          .model("claude-opus-5")
          .maxTokens(1024)
          .addUserMessage("Hello!")
          .build();

      client.messages().create(params).content().stream()
          .flatMap(block -> block.text().stream())
          .forEach(textBlock -> IO.println(textBlock.text()));
  }
  ```

  ```php PHP
  use Anthropic\Foundry;

  // 获取 Entra ID 访问令牌，例如使用 Azure CLI：
  //   az account get-access-token --resource https://ai.azure.com \
  //     --query accessToken -o tsv
  $token = getenv('AZURE_ACCESS_TOKEN');

  $client = Foundry\Client::withCredentials(
      authToken: $token,
      baseUrl: 'https://example-resource.services.ai.azure.com/anthropic',
  );

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          ['role' => 'user', 'content' => 'Hello!']
      ],
      model: 'claude-opus-5',
  );
  echo array_find($message->content, fn ($block) => $block->type === 'text')->text;
  ```

  ```ruby Ruby
  # Ruby SDK 尚未原生支持 Foundry。本示例使用
  # 标准 Ruby SDK 作为变通方案，并使用静态 Entra ID 令牌：未内置自动
  # 令牌刷新功能，因此您的应用程序必须自行刷新令牌
  # （它们通常在 1 小时后过期）。请显式传递凭据：
  # 如果不传递，客户端将回退到 ANTHROPIC_API_KEY 或
  # ANTHROPIC_AUTH_TOKEN 环境变量。如需完整的 Foundry 支持，请使用
  # C#、Java、PHP、Python 或 TypeScript SDK。
  require "anthropic"

  # 获取 Entra ID 访问令牌，例如使用 Azure CLI：
  #   az account get-access-token --resource https://ai.azure.com \
  #     --query accessToken -o tsv
  client = Anthropic::Client.new(
    base_url: "https://example-resource.services.ai.azure.com/anthropic",
    auth_token: ENV.fetch("AZURE_ACCESS_TOKEN")
  )

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello!"}]
  )

  puts message.content.find { it.type == :text }.text
  ```
</CodeGroup>

## 关联请求 ID

Foundry 在 HTTP 响应标头中包含请求标识符，用于调试和跟踪。联系支持时，请同时提供 `request-id` 和 `apim-request-id`（Azure API Management）值，以帮助团队在 Anthropic 和 Azure 系统中快速定位和调查您的请求。

## 功能支持

Microsoft Foundry 中的 Claude 支持大多数 Claude 功能。您可以在[功能概述](https://platform.claude.com/docs/zh-CN/build-with-claude/overview)中找到当前支持的所有功能。

### 上下文窗口

Claude Fable 5.1、Claude Fable 5、Claude Opus 5、Claude Opus 4.8、Claude Opus 4.7、Claude Opus 4.6、Claude Sonnet 5 和 Claude Sonnet 4.6 在 Microsoft Foundry 上具有 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)。其他 Claude 模型，包括 Claude Sonnet 4.5，具有 200k 令牌上下文窗口。

### Microsoft Foundry 中的 Claude 不支持的 Claude 功能

* Admin API
* Advisor 工具
* Claude Managed Agents
* Compliance API
* Models API
* Message Batches API
* 服务器端回退（[`fallbacks` 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)；请改用[客户端回退模式](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#client-side-fallback)）
* [计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)和[浏览器使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)工具集（`computer_toolset_20260801` 和 `browser_toolset_20260801` 目前在 Microsoft Foundry 上不可用；beta 版计算机使用工具版本仍然可用）

### 托管在 Azure 上时不支持的其他功能

以下功能可用于托管在 Anthropic 上的部署，但不支持托管在 Azure 上的部署：

* [代码执行](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)
* 晚于 `web_search_20250305` 和 `web_fetch_20250910` 的[网络搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)和[网络获取](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)工具版本。托管在 Azure 上的部署仅支持这些基础版本，因此动态过滤、响应包含和缓存绕过不可用。
* [Agent Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview)
* [程序化工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling)
* [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)

针对托管在 Azure 上的部署使用这些功能的请求按设计返回 `400 Bad Request` 错误。Claude Code 会检测托管在 Azure 上的部署并自动调整其功能集。

## API 响应

Microsoft Foundry 中 Claude 的 API 响应遵循标准的 [Claude API 响应格式](https://platform.claude.com/docs/zh-CN/api/messages/create)。这包括响应正文中的 `usage` 对象，它为您的请求提供详细的令牌消耗信息。`usage` 对象在所有平台（Claude API、Amazon Bedrock、AWS 上的 Claude Platform、Foundry 和 Google Cloud）上都是一致的。

有关特定于 Foundry 的响应标头的详细信息，请参阅[关联请求 ID](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry#correlation-request-ids)。

## API 模型 ID 和部署

生命周期术语（已弃用、已停用）在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中定义。Microsoft Foundry 遵循 Claude API 生命周期计划。

以下 Claude 模型可通过 Foundry 使用：

| 模型                | 默认部署名称            | 托管在 Azure 上 | 托管在 Anthropic 上 |
| ----------------- | ----------------- | ----------- | --------------- |
| Claude Fable 5.1  | claude-fable-5-1  |             | ✓               |
| Claude Fable 5    | claude-fable-5    |             | ✓               |
| Claude Opus 5     | claude-opus-5     | ✓           | ✓               |
| Claude Opus 4.8   | claude-opus-4-8   | ✓           | ✓               |
| Claude Opus 4.7   | claude-opus-4-7   |             | ✓               |
| Claude Opus 4.6   | claude-opus-4-6   |             | ✓               |
| Claude Opus 4.5   | claude-opus-4-5   |             | ✓               |
| Claude Sonnet 5   | claude-sonnet-5   | ✓           | ✓               |
| Claude Sonnet 4.6 | claude-sonnet-4-6 |             | ✓               |
| Claude Sonnet 4.5 | claude-sonnet-4-5 |             | ✓               |
| Claude Haiku 4.5  | claude-haiku-4-5  | ✓           | ✓               |

默认情况下，部署名称与上表中显示的模型 ID 匹配。但是，您可以在 Foundry 门户中使用不同的名称创建自定义部署，以管理不同的配置、版本或速率限制。在您的 API 请求中使用部署名称（不一定是模型 ID）。

<Info>
  [Claude Mythos Preview](https://anthropic.com/glasswing) 是一个研究预览版，可供 Microsoft Foundry 上受邀的客户使用。
</Info>

<Tip>
  正在升级到更新的 Claude 模型？在 Claude Code 中，运行 `/claude-api migrate` 即可在您的整个代码库中应用模型 ID 替换和破坏性参数变更。该技能会检测您的代码所面向的云平台，并针对该平台调整模型 ID 格式和功能变更。请参阅[迁移到更新的 Claude 模型](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model)。
</Tip>

## 计费

Microsoft Foundry 中的 Claude 通过 [Azure Marketplace](https://azuremarketplace.microsoft.com/) 计费。使用量以 Claude Consumption Units (CCU) 计量，按小时计量，并在您的 Azure 账单上按月后付费开具发票。CCU 不是预付费积分。没有 CCU 余额或承诺。

有关 CCU 价格、转换机制和每个模型的令牌费率，请参阅 [Microsoft Foundry 中的 Claude 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-in-microsoft-foundry-pricing)。

## 在托管选项之间迁移

要将现有部署从一个托管选项移动到另一个：

1. 创建模型另一个托管版本（托管在 Azure 上或托管在 Anthropic 上）的新部署。这可以在同一个 Foundry 资源中，也可以在新的资源中。
2. 更新您的应用程序以在 `model` 参数中传递新的部署名称。
3. 流量迁移后删除旧部署。

如果新部署在同一个 Foundry 资源中，则您的端点 URL 和身份验证保持不变。如果您创建了新资源，请更新应用程序的端点和凭据以指向它。

## 监控和日志记录

Azure 通过标准 Azure 模式为您的 Claude 使用提供监控和日志记录：

* **Azure Monitor：** 跟踪 API 使用情况、延迟和错误率
* **Azure Log Analytics：** 查询和分析请求/响应日志
* **Cost Management：** 监控和预测与 Claude 使用相关的成本

Anthropic 建议至少以 30 天滚动周期记录您的活动，以了解使用模式并调查任何潜在问题。

<Note>
  Azure 的日志记录服务在您的 Azure 订阅中配置。启用日志记录不会让 Microsoft 或 Anthropic 访问超出计费和服务运营所需范围的您的内容。
</Note>

## 故障排除

### 身份验证错误

**错误：** `401 Unauthorized` 或 `Invalid API key`

* **解决方案：** 验证您的 API 密钥是否正确。您可以在 Foundry 门户中您部署的 **Details** 选项卡（在 **Build** > **Models** 下）找到它。
* **解决方案：** 如果使用 Microsoft Entra ID，请确保您的访问令牌有效且未过期。令牌通常在 1 小时后过期。

**错误：** `403 Forbidden`

* **解决方案：** 您的 Azure 帐户可能缺少必要的权限。请确保您已分配适当的 Azure RBAC 角色（例如，**Foundry User**（原 Azure AI User）或 **Cognitive Services User**）。

### 速率限制

**错误：** `429 Too Many Requests`

* **解决方案：** 您已超出速率限制。在您的应用程序中实现指数退避和重试逻辑。
* **解决方案：** 考虑通过 Azure 门户或 Azure 支持请求提高速率限制。

#### 速率限制标头

Foundry 在响应中不包含 Anthropic 的标准速率限制标头（`anthropic-ratelimit-tokens-limit`、`anthropic-ratelimit-tokens-remaining`、`anthropic-ratelimit-tokens-reset`、`anthropic-ratelimit-input-tokens-limit`、`anthropic-ratelimit-input-tokens-remaining`、`anthropic-ratelimit-input-tokens-reset`、`anthropic-ratelimit-output-tokens-limit`、`anthropic-ratelimit-output-tokens-remaining` 和 `anthropic-ratelimit-output-tokens-reset`）。请改为通过 Azure 的监控工具管理速率限制。

### 模型和部署错误

**错误：** `Model not found` 或 `Deployment not found`

* **解决方案：** 验证您使用的是正确的部署名称。如果您尚未创建自定义部署，请使用默认模型 ID（例如，claude-opus-5）。
* **解决方案：** 确保该模型/部署在您的 Azure 区域中可用。

**错误：** `Invalid model parameter`

* **解决方案：** model 参数应包含您的部署名称，该名称可以在 Foundry 门户中自定义。验证部署是否存在并已正确配置。

## 后续步骤

<CardGroup cols={2}>
  <Card title="功能概述" icon="stack" href="https://platform.claude.com/docs/zh-CN/build-with-claude/overview">
    探索 Claude 的高级功能和能力。
  </Card>

  <Card title="定价" icon="chart" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-in-microsoft-foundry-pricing">
    了解 Anthropic 针对模型和功能的定价结构。
  </Card>

  <Card title="模型弃用" icon="arrow-clockwise" href="https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations">
    随着更安全、更强大的模型推出，Anthropic 会定期停用较旧的模型。查看所有 API 弃用以及推荐的替代方案。
  </Card>
</CardGroup>

## 其他资源

<CardGroup cols={2}>
  <Card title="Foundry 模型目录" icon="grid" href="https://ai.azure.com/catalog/publishers/anthropic">
    在 Foundry 目录中浏览 Anthropic 模型。
  </Card>

  <Card title="Azure AI Foundry 定价" icon="calculator" href="https://azure.microsoft.com/en-us/pricing/details/ai-foundry/#pricing">
    查看 Microsoft 针对 Azure AI Foundry 的定价详情。
  </Card>

  <Card title="模型定价" icon="table" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing#model-pricing">
    查看 Anthropic 的每个模型定价详情。
  </Card>

  <Card title="Azure 门户" icon="cloud" href="https://portal.azure.com/">
    管理您的 Azure 资源。
  </Card>
</CardGroup>
