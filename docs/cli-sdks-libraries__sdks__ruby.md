---
title: Ruby SDK
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/ruby
description: 安装并配置 Anthropic Ruby SDK，支持 Sorbet 类型、流式传输辅助工具和连接池
---

Anthropic Ruby 库为任何 Ruby 3.2.0+ 应用程序提供了便捷访问 Claude API 的方式。它附带了 Yard、RBS 和 RBI 格式的完整类型和文档字符串。HTTP 传输使用标准库的 `net/http`，并通过 `connection_pool` gem 实现连接池。

<Info>
  有关带代码示例的 API 功能文档，请参阅 [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)。本页面涵盖 Ruby 特有的 SDK 功能和配置。
</Info>

## 安装

使用 Bundler 将该 gem 添加到您应用程序的 `Gemfile` 中：

```bash
bundle add anthropic
```

## 要求

Ruby 3.2.0 或更高版本。

## 用法

```ruby
anthropic = Anthropic::Client.new(
  api_key: ENV["ANTHROPIC_API_KEY"] # This is the default and can be omitted
)

message = anthropic.messages.create(
  max_tokens: 1024,
  messages: [{role: "user", content: "Hello, Claude"}],
  model: :"claude-opus-5"
)

message.content.each do |block|
  puts block.text if block.type == :text
end
```

有关包括 Workload Identity Federation（工作负载身份联合）在内的身份验证选项，请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)。如果您的 API 密钥是可访问多个工作区的[个人密钥或服务账户密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)，请在 `anthropic-workspace-id` 请求头中设置工作区 ID；[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)展示了此 SDK 的按请求设置选项。

## 流式传输

SDK 支持使用"Server-Sent Events"（服务器发送事件），即 SSE 进行 streaming（流式传输）响应。

```ruby
anthropic = Anthropic::Client.new
stream = anthropic.messages.stream(
  max_tokens: 1024,
  messages: [{role: "user", content: "Hello, Claude"}],
  model: :"claude-opus-5"
)

stream.each do |message|
  puts(message.type)
end
```

### 流式传输辅助工具

该库为流式传输消息提供了多种便利功能，例如：

```ruby
anthropic = Anthropic::Client.new
stream = anthropic.messages.stream(
  max_tokens: 1024,
  messages: [{role: :user, content: "Say hello there!"}],
  model: :"claude-opus-5"
)

stream.text.each do |text|
  print(text)
end
```

使用 `anthropic.messages.stream(...)` 进行流式传输会暴露各种辅助工具，包括累积功能和 SDK 特有的事件。

## 输入模式与工具调用

SDK 提供了辅助机制，用于为工具定义结构化数据类，并让 Claude 自动执行它们。有关 tool use（工具使用）模式（包括工具运行器）的详细文档，请参阅[工具运行器 (SDK)](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner)。

```ruby
anthropic = Anthropic::Client.new
class CalculatorInput < Anthropic::BaseModel
  required :lhs, Float
  required :rhs, Float
  required :operator, Anthropic::InputSchema::EnumOf[:+, :-, :*, :/]
end

class Calculator < Anthropic::BaseTool
  input_schema CalculatorInput

  def call(expr)
    expr.lhs.public_send(expr.operator, expr.rhs)
  end
end

# 自动处理工具执行循环
anthropic.beta.messages.tool_runner(
  model: "claude-opus-5",
  max_tokens: 1024,
  messages: [{role: "user", content: "What's 15 * 7?"}],
  tools: [Calculator.new]
).each_message { |message| puts message.content }
```

## 结构化输出

有关包含 Ruby 示例的完整结构化输出文档，请参阅[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)。

## 错误处理

当库无法连接到 API，或者 API 返回非成功状态码（即 4xx 或 5xx 响应）时，会抛出 `Anthropic::Errors::APIError` 的子类：

```ruby
anthropic = Anthropic::Client.new
begin
  message = anthropic.messages.create(
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}],
    model: :"claude-opus-5"
  )
rescue Anthropic::Errors::APIConnectionError => e
  puts("The server could not be reached")
  puts(e.cause)  # an underlying Exception, likely raised within `net/http`
rescue Anthropic::Errors::RateLimitError => e
  puts("A 429 status code was received; we should back off a bit.")
rescue Anthropic::Errors::APIStatusError => e
  puts("Another non-200-range status code was received")
  puts(e.status)
end
```

错误代码如下：

| 原因          | 错误类型                       |
| ----------- | -------------------------- |
| HTTP 400    | `BadRequestError`          |
| HTTP 401    | `AuthenticationError`      |
| HTTP 403    | `PermissionDeniedError`    |
| HTTP 404    | `NotFoundError`            |
| HTTP 409    | `ConflictError`            |
| HTTP 422    | `UnprocessableEntityError` |
| HTTP 429    | `RateLimitError`           |
| HTTP >= 500 | `InternalServerError`      |
| 其他 HTTP 错误  | `APIStatusError`           |
| 超时          | `APITimeoutError`          |
| 网络错误        | `APIConnectionError`       |

## 重试

某些错误默认会自动重试 2 次，并采用短暂的指数退避。

连接错误（例如由于网络连接问题）、408 Request Timeout、409 Conflict、429 Rate Limit、>=500 内部错误以及超时默认都会重试。

您可以使用 `max_retries` 选项来配置或禁用此行为：

```ruby
# 为所有请求配置默认值：
anthropic = Anthropic::Client.new(
  max_retries: 0 # default is 2
)

# 或者，按请求单独配置：
anthropic.messages.create(
  max_tokens: 1024,
  messages: [{role: "user", content: "Hello, Claude"}],
  model: :"claude-opus-5",
  request_options: {max_retries: 5}
)
```

## 超时

默认情况下，请求在 10 分钟后超时。您可以使用 `timeout` 选项进行配置：

```ruby
# 为所有请求配置默认值：
anthropic = Anthropic::Client.new(
  timeout: 20 # 20 seconds (default is 10 minutes)
)

# 或者，按请求单独配置：
anthropic.messages.create(
  max_tokens: 1024,
  messages: [{role: "user", content: "Hello, Claude"}],
  model: :"claude-opus-5",
  request_options: {timeout: 5}
)
```

超时时会抛出 `Anthropic::Errors::APITimeoutError`。

请注意，超时的请求默认会被重试。

## 分页

Claude API 中的列表方法是分页的。

该库为每个列表响应提供自动分页迭代器，因此您无需手动请求后续页面：

```ruby
anthropic = Anthropic::Client.new
page = anthropic.messages.batches.list(limit: 20)

# 从页面中获取单个条目。
batch = page.data[0]
puts(batch.id)

# 根据需要自动获取更多页面。
page.auto_paging_each do |batch|
  puts(batch.id)
end
```

或者，您可以使用 `#next_page?` 和 `#next_page` 方法对页面进行更精细的控制。

```ruby
anthropic = Anthropic::Client.new
page = anthropic.messages.batches.list(limit: 20)
loop do
  page.data&.each { |batch| puts(batch.id) }
  break unless page.next_page?
  page = page.next_page
end
```

## 文件上传

与文件上传对应的请求参数可以以原始内容、[`Pathname`](https://rubyapi.org/3.2/o/pathname) 实例、[`StringIO`](https://rubyapi.org/3.2/o/stringio) 等形式传递。

```ruby
anthropic = Anthropic::Client.new
require "pathname"

# 使用 `Pathname` 发送文件名和/或避免将大文件分页加载到内存中：
file_metadata = anthropic.files.upload(file: Pathname("/path/to/file"))

# 或者，直接传入文件内容或 `StringIO`：
file_metadata = anthropic.files.upload(file: File.read("/path/to/file"))

# 或者，若要控制文件名和/或内容类型：
file = Anthropic::FilePart.new(File.read("/path/to/file"), filename: "/path/to/file", content_type: "...")
file_metadata = anthropic.files.upload(file: file)

puts(file_metadata.id)
```

请注意，您也可以传递原始 `IO` 描述符，但这会禁用重试，因为库无法确定该描述符是文件还是管道（管道无法回绕）。

## Sorbet

该库提供完整的 [RBI](https://sorbet.org/docs/rbi) 定义，并且不依赖 sorbet-runtime。

您可以像这样提供类型安全的请求参数：

```ruby
anthropic = Anthropic::Client.new
anthropic.messages.create(
  max_tokens: 1024,
  messages: [Anthropic::MessageParam.new(role: "user", content: "Hello, Claude")],
  model: :"claude-opus-5"
)
```

或者，等效地：

```ruby
anthropic = Anthropic::Client.new
# 哈希可以使用，但不是类型安全的：
anthropic.messages.create(
  max_tokens: 1024,
  messages: [{role: "user", content: "Hello, Claude"}],
  model: :"claude-opus-5"
)

# 您也可以展开（splat）一个完整的 Params 类：
params = Anthropic::MessageCreateParams.new(
  max_tokens: 1024,
  messages: [Anthropic::MessageParam.new(role: "user", content: "Hello, Claude")],
  model: :"claude-opus-5"
)
anthropic.messages.create(**params)
```

### 枚举

由于该库不依赖 `sorbet-runtime`，因此无法提供 [`T::Enum`](https://sorbet.org/docs/tenum) 实例。作为替代，SDK 提供"tagged symbols"（带标签的符号），它们在运行时始终是原始类型：

```ruby
# :auto
puts(Anthropic::MessageCreateParams::ServiceTier::AUTO)

# 显示的类型：`T.all(Anthropic::MessageCreateParams::ServiceTier, Symbol)`
T.reveal_type(Anthropic::MessageCreateParams::ServiceTier::AUTO)
```

枚举参数具有"宽松"类型，因此您既可以传入枚举常量，也可以传入其字面值：

```ruby
# 使用枚举常量可保留带标签的类型信息：
anthropic.messages.create(
  service_tier: Anthropic::MessageCreateParams::ServiceTier::AUTO,
  # ...
)

# 也允许使用字面量值：
anthropic.messages.create(
  service_tier: :auto,
  # ...
)
```

## BaseModel

所有参数和响应对象都继承自 `Anthropic::Internal::Type::BaseModel`，它提供了多种便利功能，包括：

1. 所有字段（包括未知字段）都可以通过 `obj[:prop]` 语法访问，并且可以使用 `obj => {prop: prop}` 或模式匹配语法进行解构。

2. 相等性基于结构等价；如果两次 API 调用返回相同的值，使用 == 比较响应将返回 true。

3. 实例和类本身都可以进行美化打印。

4. 诸如 `#to_h`、`#deep_to_h`、`#to_json` 和 `#to_yaml` 等辅助方法。

## 并发与连接池

`Anthropic::Client` 实例是线程安全的，但仅在没有进行中的 HTTP 请求时才是 fork 安全的。

每个 `Anthropic::Client` 实例都有自己的 HTTP 连接池，默认大小为 99。因此，在大多数场景下，建议每个应用程序只创建一次客户端。

当连接池中所有可用连接都被占用时，请求会等待新连接可用，排队时间计入请求超时。

除非另有说明，SDK 中的其他类没有保护其底层数据结构的锁。

## 发起自定义或未记录的请求

### 未记录的属性

您可以向任何端点发送未记录的参数，并读取未记录的响应属性，如下所示：

<Warning>
  同名的 `extra_` 参数会覆盖已记录的参数。出于安全原因，请确保这些方法仅用于可信的输入数据。
</Warning>

```ruby
anthropic = Anthropic::Client.new
value = "example"
message =
  anthropic.messages.create(
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}],
    model: :"claude-opus-5",
    request_options: {
      extra_query: {my_query_parameter: value},
      extra_body: {my_body_parameter: value},
      extra_headers: {"my-header": value}
    }
  )

puts(message[:my_undocumented_property])
```

### 未记录的请求参数

如果您想显式发送额外参数，可以在发起请求时通过 `request_options:` 参数下的 `extra_query`、`extra_body` 和 `extra_headers` 来实现，如上面的示例所示。

### 未记录的端点

要向未记录的端点发起请求，同时保留身份验证、重试等优势，您可以使用 `anthropic.request` 发起请求，如下所示：

```ruby
response = anthropic.request(
  method: :post,
  path: '/undocumented/endpoint',
  query: {"dog": "woof"},
  headers: {"useful-header": "interesting-value"},
  body: {"hello": "world"}
)
```

## 平台集成

<Note>
  有关带代码示例的详细平台设置指南，请参阅：

  * [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)
  * [Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)
  * [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)
  * [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)
</Note>

Ruby SDK 支持以下平台：

* **Agent Platform：** `Anthropic::VertexClient`。需要 `googleauth` gem。
* **Bedrock：** `Anthropic::BedrockMantleClient`，或用于 `bedrock-runtime` 路径的 `Anthropic::BedrockClient`。`Anthropic::BedrockMantleClient` 需要 `aws-sdk-core` gem；`Anthropic::BedrockClient` 需要 `aws-sdk-bedrockruntime` gem。
* **Claude Platform on AWS：** 属于主 `anthropic` gem 的一部分（需要 `aws-sdk-core` gem）。提供 `Anthropic::AWSClient`。向构造函数传递 `workspace_id:`，或设置 `ANTHROPIC_AWS_WORKSPACE_ID` 环境变量（请参阅[工作区](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#workspaces)）。目前处于 beta 阶段。
* **Foundry：** Ruby SDK 目前不支持。有关支持的 SDK，请参阅 [Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)。

新项目请使用 `Anthropic::BedrockMantleClient`；`Anthropic::BedrockClient` 保留给使用 Bedrock `InvokeModel` API 的现有应用程序。

## 语义化版本

该软件包遵循 [SemVer](https://semver.org/spec/v2.0.0.html) 约定。

该软件包将对（非运行时）`*.rbi` 和 `*.rbs` 类型定义的改进视为非破坏性变更。

## 其他资源

* [GitHub 仓库](https://github.com/anthropics/anthropic-sdk-ruby)
* [YARD 文档](https://gemdocs.org/gems/anthropic)
* [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)
* [流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)
