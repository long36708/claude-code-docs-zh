---
title: PHP SDK
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/php
description: 安装并配置 Anthropic PHP SDK，使用值对象和构建器模式
---

Anthropic PHP 库为任何 PHP 8.1.0+ 应用程序提供了便捷访问 Claude API 的方式。

<Info>
  PHP SDK 目前处于 beta 阶段。API 可能会在版本之间发生变化。
</Info>

<Info>
  有关包含代码示例的 API 功能文档，请参阅 [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)。本页面涵盖 PHP 特定的 SDK 功能和配置。
</Info>

## 安装

该 SDK 使用 [PSR-18](https://www.php-fig.org/psr/psr-18/) 进行 HTTP 通信，并会自动发现任何已安装的 PSR-18 客户端。推荐使用 [Guzzle](https://docs.guzzlephp.org/)，因为 SDK 会为其配置流式传输，无需额外设置：

```bash
composer require "anthropic-ai/sdk" "guzzlehttp/guzzle:^7"
```

## 要求

PHP 8.1.0 或更高版本。

## 用法

此库使用命名参数来指定可选参数。具有默认值的参数必须通过名称设置。

```php
$client = new Client();

$message = $client->messages->create(
  maxTokens: 1024,
  messages: [['role' => 'user', 'content' => 'Hello, Claude']],
  model: 'claude-opus-5',
);

$textBlock = array_find($message->content, static fn ($block): bool => $block->type === 'text');
echo $textBlock->text;
```

有关包括 Workload Identity Federation（工作负载身份联合）在内的身份验证选项，请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)。如果您的 API 密钥是可访问多个工作区的[个人密钥或服务账户密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)，请在 `anthropic-workspace-id` 请求头中设置工作区 ID；[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)展示了此 SDK 的按请求设置选项。

## 值对象

建议使用静态 `with` 构造函数 `Base64ImageSource::with(data: "U3RhaW5sZXNzIHJvY2tz", ...)` 和命名参数来初始化"value objects"（值对象）。

不过，也提供了构建器 `(new Base64ImageSource)->withData("U3RhaW5sZXNzIHJvY2tz")`。

## 流式传输

该 SDK 支持使用"Server-Sent Events"（服务器发送事件），即 SSE 进行"streaming"（流式传输）响应。

```php
$client = new Client();

$stream = $client->messages->createStream(
  maxTokens: 1024,
  messages: [['role' => 'user', 'content' => 'Hello, Claude']],
  model: 'claude-opus-5',
);

foreach ($stream as $event) {
  echo $event->type . PHP_EOL;
}
```

流式传输需要一个能够增量返回响应体的 HTTP 客户端。当 Guzzle 是被发现的 PSR-18 客户端时，SDK 会自动为其配置流式传输。如果使用带缓冲的客户端，`foreach` 循环会在响应完成时一次性产出所有事件，而不是增量产出；如果您观察到这种现象，请安装 Guzzle，或通过 `streamingTransporter` 请求选项提供一个支持流式传输的 PSR-18 客户端：

```php
$client = new Anthropic\Client(
  requestOptions: Anthropic\RequestOptions::with(streamingTransporter: $myStreamingClient),
);
```

## 错误处理

当库无法连接到 API，或者 API 返回非成功状态码（即 4xx 或 5xx 响应）时，会抛出 `Anthropic\Core\Exceptions\APIException` 的子类：

```php
<?php
// ...
use Anthropic\Core\Exceptions\APIConnectionException;
use Anthropic\Core\Exceptions\APIStatusException;
use Anthropic\Core\Exceptions\RateLimitException;
// ...
try {
  $message = $client->messages->create(
    maxTokens: 1024,
    messages: [['role' => 'user', 'content' => 'Hello, Claude']],
    model: 'claude-opus-5',
  );
} catch (APIConnectionException $e) {
  echo "The server could not be reached", PHP_EOL;
  echo $e->getPrevious()?->getMessage(), PHP_EOL;
} catch (RateLimitException $_) {
  echo "A 429 status code was received; we should back off a bit.", PHP_EOL;
} catch (APIStatusException $e) {
  echo "Another non-200-range status code was received", PHP_EOL;
  echo $e->getMessage();
}
```

错误代码如下：

| 原因          | 错误类型                           |
| ----------- | ------------------------------ |
| HTTP 400    | `BadRequestException`          |
| HTTP 401    | `AuthenticationException`      |
| HTTP 403    | `PermissionDeniedException`    |
| HTTP 404    | `NotFoundException`            |
| HTTP 409    | `ConflictException`            |
| HTTP 422    | `UnprocessableEntityException` |
| HTTP 429    | `RateLimitException`           |
| HTTP >= 500 | `InternalServerException`      |
| 其他 HTTP 错误  | `APIStatusException`           |
| 超时          | `APITimeoutException`          |
| 网络错误        | `APIConnectionException`       |

## 重试

某些错误默认会自动重试两次，并采用短暂的指数退避。

连接错误（例如，由于网络连接问题）、408 Request Timeout、409 Conflict、429 Rate Limit、>=500 内部错误以及超时默认都会重试。

您可以使用 `maxRetries` 选项来配置或禁用此行为：

```php
use Anthropic\RequestOptions;
// ...
// 为所有请求配置默认值：
$client = new Client(requestOptions: RequestOptions::with(maxRetries: 0));

// 或者，按请求单独配置：
$result = $client->messages->create(
  maxTokens: 1024,
  messages: [['role' => 'user', 'content' => 'Hello, Claude']],
  model: 'claude-opus-5',
  requestOptions: RequestOptions::with(maxRetries: 5),
);
```

## 分页

Claude API 中的列表方法是分页的。

此库为每个列表响应提供自动分页迭代器，因此您无需手动请求后续页面：

```php
$client = new Client();

$page = $client->beta->messages->batches->list(limit: 20);

// 获取当前页的条目
foreach ($page->getItems() as $item) {
  echo $item->id, PHP_EOL;
}
// 发起额外的网络请求，以获取当前页及其之后所有页的条目
foreach ($page->pagingEachItem() as $item) {
  echo $item->id, PHP_EOL;
}
```

## 高级用法

### 未记录的属性

您可以向任何端点发送未记录的参数，并读取未记录的响应属性，如下所示：

<Note>
  同名的 `extra*` 参数会覆盖已记录的参数。
</Note>

```php
<?php
// ...
use Anthropic\RequestOptions;
// ...
$message = $client->messages->create(
  maxTokens: 1024,
  messages: [['role' => 'user', 'content' => 'Hello, Claude']],
  model: 'claude-opus-5',
  requestOptions: RequestOptions::with(
    extraQueryParams: ['my_query_parameter' => 'value'],
    extraBodyParams: ['my_body_parameter' => 'value'],
    extraHeaders: ['my-header' => 'value'],
  ),
);
```

### 未记录的请求参数

如果您想显式发送额外参数，可以在发出请求时使用 `RequestOptions::with()` 下的 `extraQueryParams`、`extraBodyParams` 和 `extraHeaders` 选项，如前面的示例所示。

### 未记录的端点

要向未记录的端点发出请求，同时保留身份验证、重试和其他客户端功能的优势，您可以使用 `client->request` 发出请求，如下所示：

```php
$client = new Client();

$response = $client->request(
  method: "post",
  path: '/undocumented/endpoint',
  query: ['dog' => 'woof'],
  headers: ['useful-header' => 'interesting-value'],
  body: ['hello' => 'world']
);
```

## 平台集成

<Note>
  有关包含代码示例的详细平台设置指南，请参阅：

  * [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)
  * [Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)
  * [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)
  * [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)
  * [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)
</Note>

PHP SDK 支持以下平台：

* **Agent Platform：** `Anthropic\Vertex\Client`。使用 `::fromEnvironment()`。
* **Bedrock：** `Anthropic\Bedrock\MantleClient`。使用 `new MantleClient(awsRegion: ...)`。
* **Bedrock（旧版）：** `Anthropic\Bedrock\Client`。使用 `::fromEnvironment()` 或 `::withCredentials()`。
* **Claude Platform on AWS：** `Anthropic\Aws\Client`（需要 `aws/aws-sdk-php` 作为软依赖）。使用 `new Anthropic\Aws\Client(workspaceId: ...)` 或设置 `ANTHROPIC_AWS_WORKSPACE_ID`。以 beta 形式提供。
* **Foundry：** `Anthropic\Foundry\Client`。使用 `::withCredentials()`。

新项目请使用 `MantleClient`；`Anthropic\Bedrock\Client` 保留给使用 Bedrock `InvokeModel` API 的现有应用程序。

## 语义化版本

此包遵循 [SemVer](https://semver.org/spec/v2.0.0.html) 约定。由于该库处于初始开发阶段且主版本号为 `0`，API 可能随时发生变化。

此包将对（非运行时）PHPDoc 类型定义的改进视为非破坏性变更。

## 其他资源

* [GitHub 仓库](https://github.com/anthropics/anthropic-sdk-php)
* [Packagist](https://packagist.org/packages/anthropic-ai/sdk)
* [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)
* [流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)
