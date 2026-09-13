---
title: C# SDK
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/csharp
description: 安装并配置 Anthropic C# SDK，用于集成 IChatClient 的 .NET 应用程序
---

Anthropic C# SDK 为使用 C# 编写的应用程序提供了便捷访问 Claude API 的方式。

<Info>
  C# SDK 目前处于 beta 阶段。API 可能会在版本之间发生变化。
</Info>

<Info>
  有关带代码示例的 API 功能文档，请参阅 [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)。本页面涵盖 C# 特定的 SDK 功能和配置。
</Info>

<Warning>
  从版本 10+ 开始，`Anthropic` 包现已成为 Anthropic 官方的 C# SDK。3.X 及以下的包版本此前用于 tryAGI 社区构建的 SDK，该 SDK 已迁移至 [`tryAGI.Anthropic`](https://www.nuget.org/packages/tryagi.Anthropic/)。如果您需要在项目中继续使用之前的客户端，请将您的包引用更新为 `tryAGI.Anthropic`。
</Warning>

## 安装

从 [NuGet](https://www.nuget.org/packages/Anthropic) 安装该包：

```bash
dotnet add package Anthropic
```

## 要求

此库需要 .NET Standard 2.0 或更高版本。

## 用法

```csharp
using System;
using Anthropic;
using Anthropic.Models.Messages;

AnthropicClient client = new();

MessageCreateParams parameters = new()
{
    MaxTokens = 1024,
    Messages =
    [
        new()
        {
            Role = Role.User,
            Content = "Hello, Claude",
        },
    ],
    Model = Model.ClaudeOpus5,
};

var message = await client.Messages.Create(parameters);

foreach (var block in message.Content)
{
    if (block.TryPickText(out var textBlock))
    {
        Console.WriteLine(textBlock.Text);
    }
}
```

有关包括 Workload Identity Federation（工作负载身份联合）在内的身份验证选项，请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)。如果您的 API 密钥是可访问多个工作区的[个人密钥或服务账户密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)，请在 `anthropic-workspace-id` 请求头中设置工作区 ID；[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)展示了此 SDK 的按请求设置选项。

## 客户端配置

使用环境变量配置客户端：

```csharp
using Anthropic;

// 通过 ANTHROPIC_API_KEY、ANTHROPIC_AUTH_TOKEN 和 ANTHROPIC_BASE_URL 环境变量进行配置
AnthropicClient client = new();
```

或手动配置：

```csharp
using Anthropic;

AnthropicClient client = new() { ApiKey = "my-anthropic-api-key" };
```

或结合使用这两种方法。

可用选项请参见下表：

| 属性          | 环境变量                   | 必需    | 默认值                           |
| ----------- | ---------------------- | ----- | ----------------------------- |
| `ApiKey`    | `ANTHROPIC_API_KEY`    | false | -                             |
| `AuthToken` | `ANTHROPIC_AUTH_TOKEN` | false | -                             |
| `BaseUrl`   | `ANTHROPIC_BASE_URL`   | true  | `"https://api.anthropic.com"` |

### 修改配置

要临时使用修改后的客户端配置，同时复用相同的连接和线程池，请在任意客户端或服务上调用 `WithOptions`：

```csharp
using System;

var message = await client
    .WithOptions(options =>
        options with
        {
            BaseUrl = "https://example.com",
            Timeout = TimeSpan.FromSeconds(42),
        }
    )
    .Messages.Create(parameters);

Console.WriteLine(message);
```

使用 [`with` 表达式](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/with-expression)可以轻松构造修改后的选项。

`WithOptions` 方法不会影响原始客户端或服务。

## 流式传输

SDK 定义了返回响应"chunk"（块）流的方法，每个块在到达后即可单独处理，而无需等待完整响应。"Streaming"（流式传输）方法通常对应于 [SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) 或 [JSONL](https://jsonlines.org) 响应。

流式传输方法的名称始终带有 `Streaming` 后缀，即使它没有非流式传输的变体。

这些流式传输方法返回 [`IAsyncEnumerable`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1)：

```csharp
using System;
using Anthropic.Models.Messages;

MessageCreateParams parameters = new()
{
    MaxTokens = 1024,
    Messages =
    [
        new()
        {
            Role = Role.User,
            Content = "Hello, Claude",
        },
    ],
    Model = Model.ClaudeOpus5,
};

await foreach (var message in client.Messages.CreateStreaming(parameters))
{
    Console.WriteLine(message);
}
```

## 错误处理

SDK 会抛出自定义的非受检异常类型：

* `AnthropicApiException`：API 错误的基类。每个 HTTP 状态码对应抛出的异常子类请参见下表：

| 状态码 | 异常                                       |
| --- | ---------------------------------------- |
| 400 | `AnthropicBadRequestException`           |
| 401 | `AnthropicUnauthorizedException`         |
| 403 | `AnthropicForbiddenException`            |
| 404 | `AnthropicNotFoundException`             |
| 422 | `AnthropicUnprocessableEntityException`  |
| 429 | `AnthropicRateLimitException`            |
| 5xx | `Anthropic5xxException`                  |
| 其他  | `AnthropicUnexpectedStatusCodeException` |

此外，所有 4xx 错误都继承自 `Anthropic4xxException`。

* `AnthropicSseException`：在初始 HTTP 响应成功后，SSE 流式传输过程中遇到错误时抛出。

* `AnthropicIOException`：I/O 网络错误。

* `AnthropicInvalidDataException`：无法解释已成功解析的数据。例如，访问一个本应为必需的属性，但 API 意外地在响应中省略了它。

* `AnthropicException`：所有异常的基类。

## 重试

SDK 默认自动重试 2 次，请求之间采用短暂的指数退避。

仅以下错误类型会被重试：

* 连接错误（例如，由于网络连接问题）
* 408 Request Timeout
* 409 Conflict
* 429 Rate Limit（速率限制）
* 5xx Internal

API 也可能明确指示 SDK 重试或不重试某个请求。

要设置自定义重试次数，请使用 `MaxRetries` 属性配置客户端：

```csharp
using Anthropic;

AnthropicClient client = new() { MaxRetries = 3 };
```

或使用 `WithOptions` 配置单次方法调用：

```csharp
using System;

var message = await client
    .WithOptions(options =>
        options with { MaxRetries = 3 }
    )
    .Messages.Create(parameters);

Console.WriteLine(message);
```

## 超时

请求默认在 10 分钟后超时。

要设置自定义超时，请使用 `Timeout` 选项配置客户端：

```csharp
using System;
using Anthropic;

AnthropicClient client = new() { Timeout = TimeSpan.FromSeconds(42) };
```

或使用 `WithOptions` 配置单次方法调用：

```csharp
using System;

var message = await client
    .WithOptions(options =>
        options with { Timeout = TimeSpan.FromSeconds(42) }
    )
    .Messages.Create(parameters);

Console.WriteLine(message);
```

## 分页

SDK 定义了返回分页结果列表的方法。它提供了便捷的方式来访问结果，既可以一次访问一页，也可以跨所有页面逐项访问。

### 自动分页

要遍历所有页面的全部结果，请使用 `Paginate` 方法，它会根据需要自动获取更多页面。该方法返回 [`IAsyncEnumerable`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1)：

```csharp
using System;

var page = await client.Messages.Batches.List(parameters);
await foreach (var item in page.Paginate())
{
    Console.WriteLine(item);
}
```

### 手动分页

要访问单个页面的项目并手动请求下一页，请使用 `Items` 属性以及 `HasNext` 和 `Next` 方法：

```csharp
var page = await client.Messages.Batches.List();
while (true)
{
    foreach (var item in page.Items)
    {
        Console.WriteLine(item);
    }
    if (!page.HasNext())
    {
        break;
    }
    page = await page.Next();
}
```

## 响应验证

在极少数情况下，API 可能返回与预期类型不匹配的响应。默认情况下，SDK 在这种情况下不会抛出异常。只有当您直接访问该属性时，它才会抛出 `AnthropicInvalidDataException`。

如果您希望预先检查响应是否完全类型正确，可以调用 `Validate`：

```csharp
var message = await client.Messages.Create(parameters);
message.Validate();
```

或使用 `ResponseValidation` 选项配置客户端：

```csharp
using Anthropic;

AnthropicClient client = new() { ResponseValidation = true };
```

或使用 `WithOptions` 配置单次方法调用：

```csharp
using System;

var message = await client
    .WithOptions(options =>
        options with { ResponseValidation = true }
    )
    .Messages.Create(parameters);

Console.WriteLine(message);
```

## IChatClient 集成

SDK 提供了 `Microsoft.Extensions.AI.Abstractions` 库中 `IChatClient` 接口的实现。这使得 `AnthropicClient`（以及 `Anthropic.Services.IBetaService`）可以与集成了这些核心抽象的其他库一起使用。例如，MCP C# SDK（`ModelContextProtocol`）库中的工具可以直接与通过 `IChatClient` 暴露的 `AnthropicClient` 一起使用。

```csharp
using Anthropic;
using Microsoft.Extensions.AI;
using ModelContextProtocol.Client;

// 通过 ANTHROPIC_API_KEY、ANTHROPIC_AUTH_TOKEN 和 ANTHROPIC_BASE_URL 环境变量进行配置
AnthropicClient client = new();

IChatClient chatClient = client.AsIChatClient("claude-opus-5")
    .AsBuilder()
    .UseFunctionInvocation()
    .Build();

// 使用 MCP C# SDK 中的 McpClient
McpClient learningServer = await McpClient.CreateAsync(
    new HttpClientTransport(new() { Endpoint = new("https://learn.microsoft.com/api/mcp") }));

ChatOptions options = new() { Tools = [.. await learningServer.ListToolsAsync()] };

Console.WriteLine(await chatClient.GetResponseAsync("Tell me about IChatClient", options));
```

## 请求与响应

要向 Claude API 发送请求，请构建一个 `Params` 类的实例并将其传递给相应的客户端方法。收到响应后，它会被反序列化为一个 C# 类的实例。

例如，`client.Messages.Create` 应使用 `MessageCreateParams` 的实例调用，并将返回 `Task<Message>` 的实例。

## 高级用法

### 二进制响应

SDK 定义了返回二进制响应的方法，用于不一定需要解析的 API 响应，例如非 JSON 数据。

这些方法返回 `HttpResponse`：

```csharp
using System;
using Anthropic.Models.Files;

FileDownloadParams parameters = new() { FileID = "file_id" };

var response = await client.Files.Download(parameters);

Console.WriteLine(response);
```

要将响应内容保存到文件或任意 [`Stream`](https://learn.microsoft.com/en-us/dotnet/api/system.io.stream)，请使用 [`CopyToAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.io.stream.copytoasync) 方法：

```csharp
using System.IO;

using var response = await client.Files.Download(parameters);
using var contentStream = await response.ReadAsStream();
using var fileStream = File.Open(path, FileMode.OpenOrCreate);
await contentStream.CopyToAsync(fileStream); // Or any other Stream
```

### 原始响应

SDK 定义了将响应反序列化为 C# 类实例的方法。要访问响应头、状态码或原始响应体，请在客户端或服务上的任意 HTTP 方法调用前加上 `WithRawResponse`：

```csharp
var response = await client.WithRawResponse.Messages.Create(parameters);
var statusCode = response.StatusCode;
var headers = response.Headers;
```

原始的 `HttpResponseMessage` 也可以通过 `RawMessage` 属性访问。

对于非流式传输响应，如有需要，您可以将响应反序列化为 C# 类的实例：

```csharp
using System;
using Anthropic.Models.Messages;

var response = await client.WithRawResponse.Messages.Create(parameters);
Message deserialized = await response.Deserialize();
Console.WriteLine(deserialized);
```

对于流式传输响应，如有需要，您可以将响应反序列化为 `IAsyncEnumerable`：

```csharp
using System;

var response = await client.WithRawResponse.Messages.CreateStreaming(parameters);
await foreach (var item in response.Enumerate())
{
    Console.WriteLine(item);
}
```

### 日志记录

<Warning>
  所有日志消息仅用于调试。日志消息的格式和内容可能会在版本之间发生变化。
</Warning>

通过设置环境变量启用调试日志：

```bash
export ANTHROPIC_LOG=debug
```

### 未文档化的 API 功能

SDK 的类型设计便于使用已文档化的 API。不过，它也支持使用 API 中未文档化或尚未支持的部分。

## 平台集成

<Note>
  有关带代码示例的详细平台设置指南，请参阅：

  * [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)
  * [Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)
  * [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)
  * [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)
  * [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)
</Note>

C# SDK 通过独立的 NuGet 包支持以下平台：

* **Agent Platform：** `Anthropic.Vertex`。客户端设置请参阅 [Google Cloud 上的 Claude](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)。
* **Bedrock：** `Anthropic.Bedrock`。对于 Messages-API Bedrock 端点，请使用 `AnthropicBedrockMantleClient`，或使用 `AnthropicBedrockClient`（`bedrock-runtime` 路径）。`AnthropicBedrockMantleClient` 接受一个可选的 `MantleAwsClientOptions` 配置对象；`AnthropicBedrockClient` 接受 `AnthropicBedrockCredentialsHelper.FromEnv()` 或显式凭证。
* **Claude Platform on AWS：** `Anthropic.Aws`。使用 `AnthropicAwsClient`；在客户端上设置 `WorkspaceId` 或设置 `ANTHROPIC_AWS_WORKSPACE_ID` 环境变量（请参阅[工作区](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#workspaces)）。目前以 beta 形式提供。
* **Foundry：** `Anthropic.Foundry`。将 `AnthropicFoundryClient` 与 `DefaultAnthropicFoundryCredentials.FromEnv()` 或显式凭证一起使用。

新项目请使用 `AnthropicBedrockMantleClient`；`AnthropicBedrockClient` 保留用于使用 Bedrock `InvokeModel` API 的现有应用程序。

## 语义化版本控制

<Warning>
  尽管此包的版本号为 10+，但它目前处于 beta 阶段。在 beta 期间，次要版本或补丁版本中可能会出现破坏性变更。一旦该库达到稳定版本，将更严格地遵循 SemVer 约定。请通过[提交 issue](https://github.com/anthropics/anthropic-sdk-csharp/issues/new) 分享反馈。
</Warning>

此包总体上遵循 [SemVer](https://semver.org/spec/v2.0.0.html) 约定，但某些向后不兼容的变更可能会作为次要版本发布：

1. 对库内部的更改，这些内部在技术上是公开的，但并非为外部使用而设计或文档化。
2. 预计在实践中不会影响绝大多数用户的更改。

我们非常重视向后兼容性，以确保您可以获得顺畅的升级体验。

## 其他资源

* [GitHub 仓库](https://github.com/anthropics/anthropic-sdk-csharp)
* [NuGet 包](https://www.nuget.org/packages/Anthropic)
* [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)
* [流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)
