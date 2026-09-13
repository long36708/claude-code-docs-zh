---
title: Python SDK
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/python
description: 安装并配置 Anthropic Python SDK，支持同步和异步客户端
---

Anthropic Python SDK 为 Python 应用程序提供了便捷访问 Claude API 的方式。它支持同步和异步操作、streaming（流式传输），以及与 Amazon Bedrock、Claude Platform on AWS、Google Cloud 和 Microsoft Foundry 的集成。

<Info>
  有关带代码示例的 API 功能文档，请参阅 [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)。本页面涵盖 Python 特有的 SDK 功能和配置。
</Info>

## 安装

```bash
pip install anthropic
```

如需特定平台的集成或更好的异步性能，请使用附加依赖进行安装：

```bash
# 用于 Amazon Bedrock 支持
pip install "anthropic[bedrock]"

# 用于 Google Cloud 支持
pip install "anthropic[vertex]"

# 用于 Claude Platform on AWS 支持
pip install "anthropic[aws]"

# Microsoft Foundry 支持已包含在基础包中

# 用于通过 aiohttp 提升异步性能
pip install "anthropic[aiohttp]"
```

## 要求

需要 Python 3.10 或更高版本。如果您正在从 SDK 的 0.x 版本升级，请参阅 [v1 迁移指南](https://github.com/anthropics/anthropic-sdk-python/blob/main/MIGRATION.md)了解破坏性变更列表。

## 用法

```python
import os
from anthropic import Anthropic

client = Anthropic(
    # 这是默认值，可以省略
    api_key=os.environ.get("ANTHROPIC_API_KEY"),
)

message = client.messages.create(
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": "Hello, Claude",
        }
    ],
    model="claude-opus-5",
)

for block in message.content:
    if block.type == "text":
        print(block.text)
```

<Tip>
  可以考虑使用 [python-dotenv](https://pypi.org/project/python-dotenv/) 将 `ANTHROPIC_API_KEY="my-anthropic-api-key"` 添加到您的 `.env` 文件中，这样您的 API 密钥就不会存储在源代码管理中。
</Tip>

有关包括 Workload Identity Federation（工作负载身份联合）在内的身份验证选项，请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)。如果您的 API 密钥是可访问多个工作区的[个人密钥或服务账户密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)，请在 `anthropic-workspace-id` 请求头中设置工作区 ID；[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)展示了此 SDK 的按请求设置选项。

## 异步用法

```python
import os
import asyncio
from anthropic import AsyncAnthropic

client = AsyncAnthropic(
    api_key=os.environ.get("ANTHROPIC_API_KEY"),
)


async def main() -> None:
    message = await client.messages.create(
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": "Hello, Claude",
            }
        ],
        model="claude-opus-5",
    )
    print(message.content)


asyncio.run(main())
```

### 使用 aiohttp 获得更好的并发性能

为了提升异步性能，您可以使用 `aiohttp` HTTP 后端来替代默认的 `httpx2`：

```python
import os
import asyncio
from anthropic import AsyncAnthropic, DefaultAioHttpClient


async def main() -> None:
    async with AsyncAnthropic(
        api_key=os.environ.get("ANTHROPIC_API_KEY"),
        http_client=DefaultAioHttpClient(),
    ) as client:
        message = await client.messages.create(
            max_tokens=1024,
            messages=[
                {
                    "role": "user",
                    "content": "Hello, Claude",
                }
            ],
            model="claude-opus-5",
        )
        print(message.content)


asyncio.run(main())
```

## 流式传输响应

SDK 支持使用 "Server-Sent Events"（服务器发送事件），即 SSE 进行流式传输响应。

```python
client = Anthropic()

stream = client.messages.create(
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": "Hello, Claude",
        }
    ],
    model="claude-opus-5",
    stream=True,
)
for event in stream:
    print(event.type)
```

异步客户端使用完全相同的接口：

```python
client = AsyncAnthropic()

stream = await client.messages.create(
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": "Hello, Claude",
        }
    ],
    model="claude-opus-5",
    stream=True,
)
async for event in stream:
    print(event.type)
```

### 流式传输辅助工具

SDK 还提供了流式传输辅助工具，它们使用上下文管理器，并提供对累积文本和最终消息的访问：

```python
async def main() -> None:
    async with client.messages.stream(
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": "Say hello there!",
            }
        ],
        model="claude-opus-5",
    ) as stream:
        async for text in stream.text_stream:
            print(text, end="", flush=True)
        print()

        message = await stream.get_final_message()
        print(message.to_json())


asyncio.run(main())
```

使用 `client.messages.stream(...)` 进行流式传输会暴露各种辅助工具，包括累积功能和 SDK 特有的事件。

或者，您可以使用 `client.messages.create(..., stream=True)`，它只返回流中事件的可迭代对象，并且占用更少的内存（它不会为您构建最终的消息对象）。

## 令牌计数

您可以通过 `usage` 响应属性查看给定请求的确切用量：

```python
message = client.messages.create(...)
print(message.usage)
# Usage(input_tokens=25, output_tokens=13)
```

您也可以在发出请求之前计算令牌数：

```python
count = client.messages.count_tokens(
    model="claude-opus-5", messages=[{"role": "user", "content": "Hello, world"}]
)
print(count.input_tokens)  # 10
```

## 工具使用

此 SDK 支持 tool use（工具使用），也称为函数调用。更多详情请参阅[使用 Claude 进行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。

### 工具辅助工具

SDK 提供了将工具定义为纯 Python 函数并运行的辅助工具。`@beta_tool` 装饰器会根据函数签名和文档字符串生成工具 schema：

```python
import json
from anthropic import Anthropic, beta_tool

client = Anthropic()


@beta_tool
def get_weather(location: str) -> str:
    """Get the weather for a given location.

    Args:
        location: The city and state, for example, San Francisco, CA
    Returns:
        A JSON-encoded string with the location, temperature, and weather condition.
    """
    return json.dumps(
        {
            "location": location,
            "temperature": "68°F",
            "condition": "Sunny",
        }
    )


# 使用 tool_runner 自动处理工具调用
runner = client.beta.messages.tool_runner(
    max_tokens=1024,
    model="claude-opus-5",
    tools=[get_weather],
    messages=[
        {"role": "user", "content": "What is the weather in SF?"},
    ],
)
for message in runner:
    print(message)
```

每次迭代都会发出一个 API 请求。如果响应中包含对给定工具之一的调用，该工具会被自动调用，其结果会在下一次迭代中直接返回给模型。

## 消息批处理

此 SDK 在 `client.messages.batches` 下提供对[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)的支持。

### 创建批处理

Message Batches 接受一个请求数组，其中每个对象都有一个 `custom_id` 标识符，以及与标准 Messages API 相同的请求 `params`：

```python
client.messages.batches.create(
    requests=[
        {
            "custom_id": "my-first-request",
            "params": {
                "model": "claude-opus-5",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": "Hello, world"}],
            },
        },
        {
            "custom_id": "my-second-request",
            "params": {
                "model": "claude-opus-5",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": "Hi again, friend"}],
            },
        },
    ]
)
```

### 获取批处理结果

一旦 Message Batch 处理完成（由 `.processing_status == 'ended'` 表示），您就可以使用 `.batches.results()` 访问结果：

```python
client = anthropic.Anthropic()
batch_id = "batch_abc123"
result_stream = client.messages.batches.results(batch_id)
for entry in result_stream:
    if entry.result.type == "succeeded":
        print(entry.result.message.content)
```

## 文件上传

与文件上传对应的请求参数可以以多种不同形式传递：

* `PathLike` 对象（例如 `pathlib.Path`）
* `(filename, content, content_type)` 元组
* `BinaryIO` 类文件对象

```python
from pathlib import Path
from anthropic import Anthropic

client = Anthropic()

# 使用文件路径上传
client.files.upload(
    file=Path("/path/to/file"),
)

# 使用字节上传
client.files.upload(
    file=("file.txt", b"my bytes", "text/plain"),
)
```

异步客户端使用完全相同的接口。如果您传递 `PathLike` 实例，文件内容会自动以异步方式读取。

## 错误处理

当库无法连接到 API，或者 API 返回非成功状态码（即 4xx 或 5xx 响应）时，会抛出 `APIError` 的子类：

```python
import anthropic
# ...
try:
    message = client.messages.create(
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": "Hello, Claude",
            }
        ],
        model="claude-opus-5",
    )
except anthropic.APIConnectionError as e:
    print("The server could not be reached")
    print(e.__cause__)  # an underlying Exception, likely raised within httpx2
except anthropic.RateLimitError as e:
    print("A 429 status code was received; we should back off a bit.")
except anthropic.APIStatusError as e:
    print("Another non-200-range status code was received")
    print(e.status_code)
    print(e.response)
```

错误代码如下：

| 状态码   | 错误类型                       |
| ----- | -------------------------- |
| 400   | `BadRequestError`          |
| 401   | `AuthenticationError`      |
| 403   | `PermissionDeniedError`    |
| 404   | `NotFoundError`            |
| 409   | `ConflictError`            |
| 422   | `UnprocessableEntityError` |
| 429   | `RateLimitError`           |
| >=500 | `InternalServerError`      |
| N/A   | `APIConnectionError`       |

## 请求 ID

> 有关调试请求的更多信息，请参阅[请求 ID](https://platform.claude.com/docs/zh-CN/api/errors#request-id)。

SDK 中的所有对象响应都提供一个 `_request_id` 属性，该属性来自 `request-id` 响应头，以便您可以快速记录失败的请求并将其报告给 Anthropic。

```python
message = client.messages.create(
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
    model="claude-opus-5",
)
print(message._request_id)  # e.g., req_018EeWyXxfu5pfWkrYcMdjWG
```

<Note>
  与其他使用 `_` 前缀的属性不同，`_request_id` 属性是公开的。除非另有文档说明，所有其他带 `_` 前缀的属性、方法和模块都是私有的。
</Note>

## 重试

某些错误默认会自动重试 2 次，并采用短暂的指数退避。连接错误（例如由于网络连接问题）、408 Request Timeout、409 Conflict、429 Rate Limit 以及 >=500 的内部错误默认都会重试。

您可以使用 `max_retries` 选项来配置或禁用此行为：

```python
# 为所有请求配置默认值：
client = Anthropic(
    max_retries=0,  # default is 2
)

# 或者，按请求单独配置：
client.with_options(max_retries=5).messages.create(
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
    model="claude-opus-5",
)
```

## 超时

默认情况下，请求在 10 分钟后超时。您可以通过 `timeout` 选项进行配置，该选项接受一个浮点数或 `httpx2.Timeout` 对象：

```python
import httpx2
from anthropic import Anthropic

# 为所有请求配置默认值：
client = Anthropic(
    timeout=20.0,  # 20 seconds (default is 10 minutes)
)

# 更细粒度的控制：
client = Anthropic(
    timeout=httpx2.Timeout(60.0, read=5.0, write=10.0, connect=2.0),
)

# 按请求覆盖：
client.with_options(timeout=5.0).messages.create(
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
    model="claude-opus-5",
)
```

超时时，SDK 会抛出 `APITimeoutError`。

请注意，超时的请求[默认会重试两次](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/python#retries)。

## 长时间请求

<Warning>
  对于运行时间较长的请求，请考虑使用流式传输 [Messages API](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/python#streaming-responses)。
</Warning>

避免在不使用流式传输的情况下设置较大的 `max_tokens` 值。某些网络可能会在一段时间后断开空闲连接，这可能导致请求失败或[超时](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/python#timeouts)，而无法收到来自 Anthropic 的响应。

如果预计非流式传输请求耗时超过约 10 分钟，SDK 将抛出 `ValueError`。传递 `stream=True` 或在客户端或请求级别覆盖 `timeout` 选项可禁用此错误。

对于非流式传输请求，如果预期请求延迟超过[超时](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/python#timeouts)时间，客户端将终止连接并在未收到响应的情况下重试。

SDK 设置了 [TCP socket keep-alive](https://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html) 选项，以减少某些网络上空闲连接超时的影响。可以通过向客户端传递自定义 `http_client` 选项来覆盖此设置。

## 自动分页

Claude API 中的列表方法是分页的。您可以使用 `for` 语法遍历所有页面中的项目：

```python
client = Anthropic()

all_batches = []
# 根据需要自动获取更多页面。
for batch in client.messages.batches.list(limit=20):
    all_batches.append(batch)
print(all_batches)
```

异步迭代：

```python
async def main() -> None:
    all_batches = []
    async for batch in client.messages.batches.list(limit=20):
        all_batches.append(batch)
    print(all_batches)


asyncio.run(main())
```

或者，您可以使用 `.has_next_page()`、`.next_page_info()` 或 `.get_next_page()` 方法对页面进行更精细的控制：

```python
first_page = await client.messages.batches.list(limit=20)

if first_page.has_next_page():
    print(f"will fetch next page using these details: {first_page.next_page_info()}")
    next_page = await first_page.get_next_page()
    print(f"number of items we just fetched: {len(next_page.data)}")

# 非异步用法请移除 `await`。
```

或者直接处理返回的数据：

```python
first_page = await client.messages.batches.list(limit=20)

print(f"next page cursor: {first_page.last_id}")
for batch in first_page.data:
    print(batch.id)

# 非异步用法请移除 `await`。
```

## 默认请求头

SDK 会自动发送值为 `2023-06-01` 的 `anthropic-version` 请求头。

如有需要，您可以通过在客户端对象上或按请求设置默认请求头来覆盖它。

<Warning>
  覆盖默认请求头可能导致 SDK 中出现类型不正确以及其他意外或未定义的行为。
</Warning>

```python
# 为客户端上的所有请求设置默认请求头
client = Anthropic(
    default_headers={"anthropic-version": "My-Custom-Value"},
)

# 或按请求单独覆盖
client.messages.with_raw_response.create(
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
    model="claude-opus-5",
    extra_headers={"anthropic-version": "My-Custom-Value"},
)
```

## 类型系统

### 请求参数

嵌套的请求参数是 [TypedDicts](https://docs.python.org/3/library/typing.html#typing.TypedDict)。响应是 [Pydantic 模型](https://docs.pydantic.dev)，它们还提供了诸如序列化回 JSON 等辅助方法（[`v1`](https://docs.pydantic.dev/1.10/usage/models/)、[`v2`](https://docs.pydantic.dev/latest/concepts/serialization/)）。

类型化的请求和响应可在您的编辑器中提供自动补全和文档。如果您希望在 VS Code 中看到类型错误以便更早发现 bug，请将 `python.analysis.typeCheckingMode` 设置为 `basic`。

### 响应模型

要将 Pydantic 模型转换为字典，请使用辅助方法：

```python
message = client.messages.create(...)

# 转换为 JSON 字符串
json_str = message.to_json()

# 转换为字典
data = message.to_dict()
```

### 处理 null 与缺失字段

在响应中，您可以区分显式为 `null` 的字段与未返回（缺失）的字段：

```python
response = client.messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
if response.my_field is None:
    if "my_field" not in response.model_fields_set:
        print("field was not in the response")
    else:
        print("field was null")
```

## 高级用法

### 访问原始响应数据（例如请求头）

`httpx2` 返回的"原始" `Response` 可以通过客户端上的 `.with_raw_response` 属性访问。这对于访问响应头或其他元数据非常有用：

```python
client = Anthropic()

response = client.messages.with_raw_response.create(
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
    model="claude-opus-5",
)

print(response.headers.get("request-id"))
message = (
    response.parse()
)  # get the object that `messages.create()` would have returned
print(message.content)
```

这些方法返回一个 `APIResponse` 对象。在异步客户端上，它们返回 `AsyncAPIResponse`，并且 `.parse()`、`.read()`、`.text()` 和 `.json()` 必须使用 await。

### 流式传输响应体

`.with_raw_response` 方式会在您发出请求时立即读取完整的响应体。若要改为流式传输响应体，请使用 `.with_streaming_response`，它需要上下文管理器，并且只有在您调用 `.read()`、`.text()`、`.json()`、`.iter_bytes()`、`.iter_text()`、`.iter_lines()` 或 `.parse()` 时才会读取响应体。在异步客户端中，这些是异步方法。

```python
with client.messages.with_streaming_response.create(
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
    model="claude-opus-5",
) as response:
    print(response.headers.get("request-id"))

    for line in response.iter_lines():
        print(line)
```

需要使用上下文管理器，以确保响应能够被可靠地关闭。

### 日志记录

SDK 使用标准库 `logging` 模块。

您可以通过将环境变量 `ANTHROPIC_LOG` 设置为 `debug` 或 `info` 来启用日志记录：

```bash
export ANTHROPIC_LOG=debug
```

### 发出自定义/未文档化的请求

此库经过类型化，以便于访问已文档化的 API。如果您需要访问未文档化的端点、参数或响应属性，仍然可以使用此库。

#### 未文档化的端点

要向未文档化的端点发出请求，您可以使用 `client.get`、`client.post` 和其他 HTTP 动词。发出这些请求时，客户端上的选项（如重试）会得到遵守。

```python
import httpx2

response = client.post(
    "/foo",
    cast_to=httpx2.Response,
    body={"my_param": True},
)

print(response.json())
```

#### 未文档化的请求参数

如果您想显式发送额外参数，可以使用 `extra_query`、`extra_body` 和 `extra_headers` 请求选项。

<Warning>
  `extra_` 参数会覆盖同名的已文档化参数。出于安全原因，请确保这些方法仅用于可信的输入数据。
</Warning>

#### 未文档化的响应属性

要访问未文档化的响应属性，您可以像 `response.unknown_prop` 这样访问额外字段。您还可以通过 `response.model_extra` 以字典形式获取 Pydantic 模型上的所有额外字段。

### 配置 HTTP 客户端

SDK 使用 [httpx2](https://httpx2.pydantic.dev) 发送请求，它是 `httpx` 的一个 API 兼容分支。要自定义 HTTP 客户端（包括代理和传输），请将您自己的 [httpx2 客户端](https://httpx2.pydantic.dev/api/#client)作为 `http_client` 传入：

```python
import httpx2
from anthropic import Anthropic, DefaultHttpxClient

client = Anthropic(
    # 或使用 `ANTHROPIC_BASE_URL` 环境变量
    base_url="http://my.test.server.example.com:8083",
    http_client=DefaultHttpxClient(
        proxy="http://my.test.proxy.example.com",
        transport=httpx2.HTTPTransport(local_address="0.0.0.0"),
    ),
)
```

您还可以使用 `with_options()` 按请求自定义客户端：

```python
client.with_options(http_client=DefaultHttpxClient(...))
```

<Note>
  请使用 `DefaultHttpxClient` 和 `DefaultAsyncHttpxClient`，而不是原始的 `httpx2.Client` 和 `httpx2.AsyncClient`，以确保保留 SDK 的默认配置（如超时和连接限制）。`http_client` 参数必须是 `httpx2` 客户端。传入来自独立 `httpx` 包的客户端会引发 `TypeError`。
</Note>

对 `httpx` 本身进行补丁的追踪和模拟工具（例如 OpenTelemetry 的 `HTTPXClientInstrumentor`、Sentry 的 `httpx` 集成、`respx` 或 `pytest-httpx`）默认情况下看不到 SDK 的请求。要使用它们，请在启动时、在任何代码导入 `httpx` 之前调用一次 `httpx2.alias_httpx()`。这会使整个进程中的 `import httpx` 解析为 `httpx2`。

### 管理 HTTP 资源

默认情况下，每当客户端被[垃圾回收](https://docs.python.org/3/reference/datamodel.html#object.__del__)时，库会关闭底层 HTTP 连接。如有需要，您可以使用 `.close()` 方法手动关闭客户端，或使用在退出时关闭的上下文管理器。

```python
with Anthropic() as client:
    message = client.messages.create(...)

# HTTP 客户端会自动关闭
```

## Beta 功能

Beta 功能在正式发布之前提供，以便获取早期反馈并测试新功能。您可以在[使用 Claude 构建概览](https://platform.claude.com/docs/zh-CN/build-with-claude/overview)中查看 Claude 所有能力和工具的可用性。

您可以通过客户端的 `beta` 属性访问大多数 beta API 功能。要启用特定的 beta 功能，您需要在创建消息时将相应的 [beta 请求头](https://platform.claude.com/docs/zh-CN/api/beta-headers)添加到 `betas` 字段中。

例如，要启用[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)：

```python
client = Anthropic()

response = client.beta.messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
    betas=["context-management-2025-06-27"],
)
```

## 平台集成

<Note>
  有关带代码示例的详细平台设置指南，请参阅：

  * [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)
  * [Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)
  * [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)
  * [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)
  * [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)
</Note>

所有五个客户端类都包含在基础 `anthropic` 包中：

| 提供商                           | 客户端                                            | 额外依赖                               |
| ----------------------------- | ---------------------------------------------- | ---------------------------------- |
| Agent Platform                | `from anthropic import AnthropicVertex`        | `pip install "anthropic[vertex]"`  |
| Bedrock                       | `from anthropic import AnthropicBedrockMantle` | `pip install "anthropic[bedrock]"` |
| Bedrock（`bedrock-runtime` 路径） | `from anthropic import AnthropicBedrock`       | `pip install "anthropic[bedrock]"` |
| Claude Platform on AWS        | `from anthropic import AnthropicAWS`           | `pip install "anthropic[aws]"`     |
| Foundry                       | `from anthropic import AnthropicFoundry`       | 无                                  |

`AnthropicAWS` 客户端处于 beta 阶段。请将 `workspace_id` 传递给构造函数，或设置 `ANTHROPIC_AWS_WORKSPACE_ID` 环境变量。

新项目请使用 `AnthropicBedrockMantle`；`AnthropicBedrock` 保留给使用 Bedrock `InvokeModel` API 的现有应用程序。

## 语义化版本

此包总体上遵循 [SemVer](https://semver.org/spec/v2.0.0.html) 约定，但某些向后不兼容的变更可能会作为次要版本发布：

1. 仅影响静态类型、不破坏运行时行为的变更。
2. 对库内部的变更，这些内部在技术上是公开的，但并非为外部使用而设计或文档化。
3. 预计在实践中不会影响绝大多数用户的变更。

### 确定已安装的版本

如果您已升级到最新版本但没有看到预期的新功能，那么您的 Python 环境很可能仍在使用旧版本。您可以通过以下方式确定运行时使用的版本：

```python
print(anthropic.__version__)
```

## 其他资源

* [GitHub 仓库](https://github.com/anthropics/anthropic-sdk-python)
* [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)
* [流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)
* [使用 Claude 进行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)
