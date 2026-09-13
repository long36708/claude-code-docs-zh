---
title: TypeScript SDK
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/typescript
description: 为 Node.js、Deno、Bun 和浏览器环境安装并配置 Anthropic TypeScript SDK
---

此库提供了从 TypeScript 或 JavaScript 便捷访问 Claude API 的方式。

<Info>
  有关带代码示例的 API 功能文档，请参阅 [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)。本页面涵盖 TypeScript 特有的 SDK 功能和配置。
</Info>

## 安装

```bash
npm install @anthropic-ai/sdk
```

## 要求

支持 TypeScript >= 4.9。

支持以下运行时：

* Node.js 20 LTS 或更高（[非 EOL](https://endoflife.date/nodejs)）版本。
* Deno v1.28.0 或更高版本。
* Bun 1.0 或更高版本。
* Cloudflare Workers。
* Vercel Edge Runtime。
* Jest 28 或更高版本，使用 `"node"` 环境（目前不支持 `"jsdom"`）。
* Nitro v2.6 或更高版本。
* Web 浏览器：默认禁用，以避免暴露您的秘密 API 凭据（请参阅 [API 密钥最佳实践](https://support.claude.com/en/articles/9767949-api-key-best-practices-keeping-your-keys-safe-and-secure)）。通过显式将 `dangerouslyAllowBrowser` 设置为 `true` 来启用浏览器支持。

请注意，目前不支持 React Native。

如果您对其他运行时环境感兴趣，请在 [GitHub 仓库](https://github.com/anthropics/anthropic-sdk-typescript)中提交 issue 或为已有 issue 点赞。

## 用法

```typescript
const client = new Anthropic({
  apiKey: process.env["ANTHROPIC_API_KEY"] // This is the default and can be omitted
});

const message = await client.messages.create({
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello, Claude" }],
  model: "claude-opus-5"
});

for (const block of message.content) {
  if (block.type === "text") {
    console.log(block.text);
  }
}
```

有关包括 Workload Identity Federation（工作负载身份联合）在内的身份验证选项，请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)。如果您的 API 密钥是可访问多个工作区的[个人密钥或服务账户密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)，请在 `anthropic-workspace-id` 请求头中设置工作区 ID；[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)展示了此 SDK 的按请求设置选项。

## 请求和响应类型

此库包含所有请求参数和响应字段的 TypeScript 定义。您可以像这样导入并使用它们：

```typescript
const client = new Anthropic({
  apiKey: process.env["ANTHROPIC_API_KEY"] // This is the default and can be omitted
});

const params: Anthropic.MessageCreateParams = {
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello, Claude" }],
  model: "claude-opus-5"
};
const message: Anthropic.Message = await client.messages.create(params);
```

每个方法、请求参数和响应字段的文档都以文档字符串（docstring）形式提供，并在大多数现代编辑器中悬停时显示。

## 计算令牌数

您可以通过 `usage` 响应属性查看给定请求的确切用量，例如：

```typescript
const message = await client.messages.create(/* ... */);
console.log(message.usage);
// { input_tokens: 25, output_tokens: 13 }
```

## 流式传输响应

SDK 支持使用"Server Sent Events"（服务器发送事件），即 SSE 进行 streaming（流式传输）响应。

```typescript
const client = new Anthropic();

const stream = await client.messages.create({
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello, Claude" }],
  model: "claude-opus-5",
  stream: true
});
for await (const messageStreamEvent of stream) {
  console.log(messageStreamEvent.type);
}
```

如果您需要取消流，可以从循环中 `break`，或调用 `stream.controller.abort()`。

## 流式传输辅助工具

此库为流式传输消息提供了多种便捷功能，例如：

```typescript
const anthropic = new Anthropic();

const stream = anthropic.messages
  .stream({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: "Say hello there!"
      }
    ]
  })
  .on("text", (text) => {
    console.log(text);
  });

const message = await stream.finalMessage();
console.log(message);
```

使用 `client.messages.stream(...)` 进行流式传输会提供各种便捷的辅助工具，包括事件处理器和累积功能。

或者，您可以使用 `client.messages.create({ ..., stream: true })`，它只返回流中事件的异步可迭代对象，因此占用更少的内存（它不会为您构建最终的消息对象）。

## 工具辅助工具

此 SDK 提供了辅助工具，便于在 Messages API 中创建和运行工具。您可以使用 Zod schema 或 JSON Schema 来描述工具的输入。然后，您可以使用 `client.beta.messages.toolRunner()` 方法运行这些工具。此方法负责将所选模型生成的输入传递给正确的工具，并将结果传回模型。

有关 tool use（工具使用）的更多详情，请参阅[使用 Claude 进行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。

```typescript
import { betaZodTool } from "@anthropic-ai/sdk/helpers/beta/zod";
import { z } from "zod";

const anthropic = new Anthropic();

const weatherTool = betaZodTool({
  name: "get_weather",
  inputSchema: z.object({
    location: z.string()
  }),
  description: "Get the current weather in a given location",
  run: (input) => {
    return `The weather in ${input.location} is foggy and 60°F`;
  }
});

const finalMessage = await anthropic.beta.messages.toolRunner({
  model: "claude-opus-5",
  max_tokens: 1000,
  messages: [{ role: "user", content: "What is the weather in San Francisco?" }],
  tools: [weatherTool]
});

console.log(finalMessage.content);
```

### 工具错误

要将工具的错误报告回模型，请从 `run` 函数中抛出 `ToolError`。与普通的 `Error` 不同，`ToolError` 接受内容块，允许您在错误响应中包含图像或其他结构化内容：

```typescript
import { ToolError } from "@anthropic-ai/sdk/lib/tools/BetaRunnableTool";

const screenshotTool = betaZodTool({
  name: "take_screenshot",
  inputSchema: z.object({ url: z.string() }),
  run: async (input) => {
    if (!isValidUrl(input.url)) {
      throw new ToolError(`Invalid URL: ${input.url}`);
    }
    const result = await takeScreenshot(input.url);
    if (result.error) {
      // 附上错误截图，以便模型能看到出了什么问题
      throw new ToolError([
        { type: "text", text: `Failed to load page: ${result.error}` },
        {
          type: "image",
          source: { type: "base64", data: result.screenshot, media_type: "image/png" }
        }
      ]);
    }
    return {
      type: "image",
      source: { type: "base64", data: result.screenshot, media_type: "image/png" }
    };
  }
});
```

如果抛出的是普通 `Error`，其消息将被转换为文本内容块。

## 工具使用

此 SDK 支持工具使用，也称为函数调用。有关更多详情，请参阅[使用 Claude 进行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。

## MCP 辅助工具

此 SDK 提供了与 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 服务器集成的辅助工具。这些辅助工具将 MCP 类型转换为 Claude API 类型，减少使用 MCP 工具、提示和资源时的样板代码。

<Tip>
  Claude API 还支持 [`mcp_servers` 参数](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)，让 Claude 可以直接连接到远程 MCP 服务器。当您拥有可通过 URL 访问的远程服务器且只需要工具支持时，请使用 `mcp_servers`。当您需要本地 MCP 服务器、提示、资源，或需要对 MCP 连接进行更多控制时，请使用 MCP 辅助工具。
</Tip>

```typescript
import {
  mcpTools,
  mcpMessages,
  mcpResourceToContent,
  mcpResourceToFile
} from "@anthropic-ai/sdk/helpers/beta/mcp";
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const anthropic = new Anthropic();

// 连接到 MCP 服务器
const transport = new StdioClientTransport({ command: "mcp-server", args: [] });
const mcpClient = new Client({ name: "my-client", version: "1.0.0" });
await mcpClient.connect(transport);

// 使用 MCP 提示
const { messages } = await mcpClient.getPrompt({ name: "my-prompt" });
const response = await anthropic.beta.messages.create({
  model: "claude-opus-5",
  max_tokens: 1024,
  messages: mcpMessages(messages)
});
console.log(response.content);

// 通过 toolRunner 使用 MCP 工具
const { tools } = await mcpClient.listTools();
const finalMessage = await anthropic.beta.messages.toolRunner({
  model: "claude-opus-5",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Use the available tools" }],
  tools: mcpTools(tools, mcpClient)
});
console.log(finalMessage.content);

// 将 MCP 资源用作内容
const resource = await mcpClient.readResource({ uri: "file:///path/to/doc.txt" });
await anthropic.beta.messages.create({
  model: "claude-opus-5",
  max_tokens: 1024,
  messages: [
    {
      role: "user",
      content: [
        mcpResourceToContent(resource),
        { type: "text", text: "Summarize this document" }
      ]
    }
  ]
});

// 将 MCP 资源作为文件上传
const fileResource = await mcpClient.readResource({ uri: "file:///path/to/data.json" });
await anthropic.files.upload({ file: mcpResourceToFile(fileResource) });
```

### MCP 错误处理

如果某个 MCP 值不受 Claude API 支持（例如，不支持的内容类型、不支持的 MIME 类型、非 http/https 的资源链接），转换函数会抛出 `UnsupportedMCPValueError`。

## 消息批处理

此 SDK 在 `client.messages.batches` 命名空间下提供对[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)的支持。

### 创建批处理

Message Batches 接受一个请求数组，其中每个对象都有一个 `custom_id` 标识符，以及与标准 Messages API 完全相同的请求 `params`：

```typescript
const batch = await client.messages.batches.create({
  requests: [
    {
      custom_id: "my-first-request",
      params: {
        model: "claude-opus-5",
        max_tokens: 1024,
        messages: [{ role: "user", content: "Hello, world" }]
      }
    },
    {
      custom_id: "my-second-request",
      params: {
        model: "claude-opus-5",
        max_tokens: 1024,
        messages: [{ role: "user", content: "Hi again, friend" }]
      }
    }
  ]
});
```

### 获取批处理结果

一旦 Message Batch 处理完成（由 `.processing_status === 'ended'` 表示），您就可以使用 `.batches.results()` 访问结果

```typescript
const results = await client.messages.batches.results(batch.id);
for await (const entry of results) {
  if (entry.result.type === "succeeded") {
    console.log(entry.result.message.content);
  }
}
```

## 文件上传

与文件上传对应的请求参数可以以多种不同形式传递：

* `File`（或具有相同结构的对象）
* `fetch` 的 `Response`（或具有相同结构的对象）
* `fs.ReadStream`
* `toFile` 辅助工具的返回值

请显式设置 content-type，因为 files API 不会为您推断它：

```typescript
import fs from "node:fs";
import Anthropic, { toFile } from "@anthropic-ai/sdk";

const client = new Anthropic();

// 如果您可以访问 Node `fs`，请使用 `fs.createReadStream()`：
await client.files.upload({
  file: await toFile(fs.createReadStream("/path/to/file"), undefined, {
    type: "application/json"
  })
});

// 或者，如果您有 Web `File` API，可以传入一个 `File` 实例：
await client.files.upload({
  file: new File(["my bytes"], "file.txt", { type: "text/plain" })
});
// 您也可以传入 `fetch` 的 `Response`：
await client.files.upload({
  file: await fetch("https://somesite/file")
});

// 或者 `Buffer` / `Uint8Array`
await client.files.upload({
  file: await toFile(Buffer.from("my bytes"), "file", { type: "text/plain" })
});
await client.files.upload({
  file: await toFile(new Uint8Array([0, 1, 2]), "file", { type: "text/plain" })
});
```

## 处理错误

当库无法连接到 API， 或者 API 返回非成功状态码（即 4xx 或 5xx 响应）时， 会抛出 `APIError` 的子类：

```typescript
const message = await client.messages
  .create({
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    model: "claude-opus-5"
  })
  .catch(async (err) => {
    if (err instanceof Anthropic.APIError) {
      console.log(err.status); // 400
      console.log(err.name); // BadRequestError
      console.log(err.headers); // {server: 'nginx', ...}
    } else {
      throw err;
    }
  });
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

SDK 中的所有对象响应都提供一个 `_request_id` 属性，该属性取自 `request-id` 响应头，以便您可以快速记录失败的请求并将其报告给 Anthropic。

```typescript
const message = await client.messages.create({
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello, Claude" }],
  model: "claude-opus-5"
});
console.log(message._request_id); // req_018EeWyXxfu5pfWkrYcMdjWG
```

## 重试

某些错误默认会自动重试 2 次，并采用短暂的指数退避。 连接错误（例如，由于网络连接问题）、408 Request Timeout、409 Conflict、 429 Rate Limit 以及 >=500 的内部错误默认都会重试。

您可以使用 `maxRetries` 选项来配置或禁用此行为：

```typescript
// 为所有请求配置默认值：
const client = new Anthropic({
  maxRetries: 0 // default is 2
});

// 或者，按请求单独配置：
await client.messages.create(
  {
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    model: "claude-opus-5"
  },
  { maxRetries: 5 }
);
```

## 超时

默认情况下，请求在 10 分钟后超时。但是，如果您指定了较大的 `max_tokens` 值并且 *未*使用流式传输，则默认超时将使用以下公式动态计算：

```typescript
const minimum = 10 * 60;
const calculated = (60 * 60 * maxTokens) / 128_000;
return calculated < minimum ? minimum * 1000 : calculated * 1000;
```

这将产生最长 60 分钟的超时，按 `max_tokens` 参数缩放，除非在请求或客户端级别被覆盖。

您可以使用 `timeout` 选项进行配置：

```typescript
// 为所有请求配置默认值：
const client = new Anthropic({
  timeout: 20 * 1000 // 20 seconds (default is 10 minutes)
});

// 按请求覆盖：
await client.messages.create(
  {
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    model: "claude-opus-5"
  },
  { timeout: 5 * 1000 }
);
```

超时时，会抛出 `APIConnectionTimeoutError`。

请注意，超时的请求[默认会重试两次](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/typescript#retries)。

## 长请求

<Warning>
  对于运行时间较长的请求，请考虑使用流式传输的 [Messages API](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/typescript#streaming-responses)。
</Warning>

避免在不使用流式传输的情况下设置较大的 `max_tokens` 值。 某些网络可能会在一段时间后断开空闲连接，这 可能导致请求失败或[超时](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/typescript#timeouts)，而未收到来自 Anthropic 的响应。

如果预计非流式传输请求的时长超过大约 10 分钟，此 SDK 也会抛出错误。 传递 `stream: true` 或在客户端或请求级别[覆盖](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/typescript#timeouts) `timeout` 选项可禁用此错误。

对于非流式传输请求，如果预期请求延迟超过[超时](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/typescript#timeouts)时间， 将导致客户端在未收到响应的情况下终止连接并重试。

当 `fetch` 实现支持时，SDK 会设置 [TCP socket keep-alive](https://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html) 选项， 以减少某些网络上空闲连接超时的影响。 这可以通过配置自定义代理来[覆盖](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/typescript#configuring-proxies)。

## 自动分页

Claude API 中的列表方法是分页的。 您可以使用 `for await ... of` 语法遍历所有页面中的项目：

```typescript
async function fetchAllMessageBatches() {
  const allMessageBatches = [];
  // 根据需要自动获取更多页面。
  for await (const messageBatch of client.messages.batches.list({ limit: 20 })) {
    allMessageBatches.push(messageBatch);
  }
  return allMessageBatches;
}
```

或者，您可以一次请求单个页面：

```typescript
let page = await client.messages.batches.list({ limit: 20 });
for (const messageBatch of page.data) {
  console.log(messageBatch);
}

// 提供了用于手动分页的便捷方法：
while (page.hasNextPage()) {
  page = await page.getNextPage();
  // ...
}
```

## 默认请求头

SDK 会自动发送设置为 `2023-06-01` 的 `anthropic-version` 请求头。

如果需要，您可以通过在每个请求上设置默认请求头来覆盖它。

请注意，这样做可能会导致 SDK 中出现不正确的类型以及其他意外或未定义的行为。

```typescript
const client = new Anthropic();

const message = await client.messages.create(
  {
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    model: "claude-opus-5"
  },
  { headers: { "anthropic-version": "My-Custom-Value" } }
);
```

## 高级用法

### 访问原始 Response 数据（例如，请求头）

`fetch()` 返回的"原始" `Response` 可以通过所有方法返回的 `APIPromise` 类型上的 `.asResponse()` 方法访问。 此方法在收到成功响应的请求头后立即返回，并且不会消费响应体，因此您可以自由编写自定义解析或流式传输逻辑。

您还可以使用 `.withResponse()` 方法获取原始 `Response` 以及解析后的数据。 与 `.asResponse()` 不同，此方法会消费响应体，并在解析完成后返回。

```typescript
const client = new Anthropic();

const response = await client.messages
  .create({
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    model: "claude-opus-5"
  })
  .asResponse();
console.log(response.headers.get("X-My-Header"));
console.log(response.statusText); // access the underlying Response object

const { data: message, response: raw } = await client.messages
  .create({
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    model: "claude-opus-5"
  })
  .withResponse();
console.log(raw.headers.get("X-My-Header"));
console.log(message.content);
```

### 日志记录

<Warning>
  所有日志消息仅用于调试。日志消息的格式和内容 可能会在不同版本之间发生变化。
</Warning>

#### 日志级别

您可以通过两种方式配置日志级别：

1. 通过 `ANTHROPIC_LOG` 环境变量
2. 使用 `logLevel` 客户端选项（如果设置，则覆盖环境变量）

```typescript
const client = new Anthropic({
  logLevel: "debug" // Show all log messages
});
```

可用的日志级别，从最详细到最简略：

* `'debug'` - 显示调试消息、信息、警告和错误
* `'info'` - 显示信息消息、警告和错误
* `'warn'` - 显示警告和错误（默认）
* `'error'` - 仅显示错误
* `'off'` - 禁用所有日志记录

在 `'debug'` 级别，所有 HTTP 请求和响应都会被记录，包括请求头和请求体。 某些与身份验证相关的请求头会被隐去，但请求和响应体中的敏感数据 可能仍然可见。

#### 自定义日志记录器

默认情况下，此库将日志输出到 `globalThis.console`。您也可以提供自定义日志记录器。 支持大多数日志库，包括 [pino](https://www.npmjs.com/package/pino)、[winston](https://www.npmjs.com/package/winston)、[bunyan](https://www.npmjs.com/package/bunyan)、[consola](https://www.npmjs.com/package/consola)、[signale](https://www.npmjs.com/package/signale) 和 [@std/log](https://jsr.io/@std/log)。如果您的日志记录器无法工作，请提交 issue。

提供自定义日志记录器时，`logLevel` 选项仍然控制发出哪些消息；低于 所配置级别的消息不会发送到您的日志记录器。

```typescript
import pino from "pino";

const logger = pino();

const client = new Anthropic({
  logger: logger.child({ name: "Anthropic" }),
  logLevel: "debug" // Send all messages to pino, allowing it to filter
});
```

### 发起自定义/未文档化的请求

此库的类型定义便于访问已文档化的 API。如果您需要访问未文档化的 端点、参数或响应属性，仍然可以使用此库。

#### 未文档化的端点

要向未文档化的端点发起请求，您可以使用 `client.get`、`client.post` 和其他 HTTP 动词。 发起这些请求时，客户端上的选项（例如重试）会被遵循。

```typescript
await client.post("/some/path", {
  body: { some_prop: "foo" },
  query: { some_query_arg: "bar" }
});
```

#### 未文档化的请求参数

要使用未文档化的参数发起请求，您可以在未文档化的 参数上使用 `// @ts-expect-error`。此库不会在运行时验证请求是否与类型匹配，因此您 发送的任何额外值都将按原样发送。

```typescript
client.messages.create({
  // ...
  // @ts-expect-error baz is not yet public
  baz: "undocumented option"
});
```

对于使用 `GET` 动词的请求，任何额外参数都将放在查询字符串中；所有其他请求将在 请求体中发送额外参数。

如果您想显式发送额外参数，可以使用 `query`、`body` 和 `headers` 请求 选项来实现。

#### 未文档化的响应属性

要访问未文档化的响应属性，您可以在响应对象上使用 `// @ts-expect-error` 来访问 响应对象，或将响应对象强制转换为所需类型。与请求参数一样，SDK 不会 验证或剥离来自 API 响应中的额外属性。

### 自定义 fetch 客户端

默认情况下，此库期望已定义全局 `fetch` 函数。

如果您想使用不同的 `fetch` 函数，可以对全局进行 polyfill：

```typescript
import fetch from "my-fetch";

globalThis.fetch = fetch;
```

或者将其传递给客户端：

```typescript
import fetch from "my-fetch";

const client = new Anthropic({ fetch });
```

### Fetch 选项

如果您想设置自定义 `fetch` 选项而不覆盖 `fetch` 函数，可以在创建客户端或发起请求时提供 `fetchOptions` 对象。（请求特定的选项会覆盖客户端选项。）

```typescript
const client = new Anthropic({
  fetchOptions: {
    // `RequestInit` 选项
  }
});
```

### 配置代理

要修改代理行为，您可以提供自定义 `fetchOptions`，为请求添加运行时特定的代理 选项：

<Tabs>
  <Tab title="Node.js">
    ```typescript
    import * as undici from "undici";

    const proxyAgent = new undici.ProxyAgent("http://localhost:8888");
    const client = new Anthropic({
      fetchOptions: {
        dispatcher: proxyAgent
      }
    });
    ```
  </Tab>

  <Tab title="Bun">
    ```typescript
    const client = new Anthropic({
      fetchOptions: {
        proxy: "http://localhost:8888"
      }
    });
    ```
  </Tab>

  <Tab title="Deno">
    ```typescript
    import Anthropic from "npm:@anthropic-ai/sdk";

    const httpClient = Deno.createHttpClient({ proxy: { url: "http://localhost:8888" } });
    const client = new Anthropic({
      fetchOptions: {
        client: httpClient
      }
    });
    ```
  </Tab>
</Tabs>

## Beta 功能

Beta 功能在正式发布之前提供，以便获取早期反馈并测试新功能。您可以在[使用 Claude 构建概览](https://platform.claude.com/docs/zh-CN/build-with-claude/overview)中查看 Claude 所有功能和工具的可用性。

您可以通过客户端的 beta 属性访问大多数 beta API 功能。要启用特定的 beta 功能，您需要在创建消息时将相应的 [beta 请求头](https://platform.claude.com/docs/zh-CN/api/beta-headers)添加到 `betas` 字段中。

例如，要启用[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)：

```typescript
const client = new Anthropic();
const response = await client.beta.messages.create({
  model: "claude-opus-5",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello, Claude" }],
  betas: ["context-management-2025-06-27"]
});
```

## 运行时支持

<Accordion title="浏览器用法">
  启用 `dangerouslyAllowBrowser` 选项可能很危险，因为它会在客户端代码中暴露您的秘密 API 凭据。Web 浏览器本质上不如服务器环境安全，任何能够访问浏览器的用户都有可能检查、提取和滥用这些凭据。这可能导致他人使用您的凭据进行未经授权的访问，并可能危及敏感数据或功能。

  **什么情况下这可能不危险？**

  在某些场景下，启用浏览器支持可能不会带来重大风险：

  * **内部工具：** 如果应用程序仅在受控的内部环境中使用，且用户是可信的，则凭据暴露的风险可以得到缓解。
  * **开发或调试目的：** 临时启用此功能可能是可以接受的，前提是凭据是短期有效的、不同时用于生产环境，或者经常轮换。
</Accordion>

## 平台集成

<Note>
  有关带代码示例的详细平台设置指南，请参阅：

  * [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)
  * [Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)
  * [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)
  * [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)
  * [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)
</Note>

TypeScript SDK 支持以下平台：

* **Agent Platform：** `npm install @anthropic-ai/vertex-sdk`：提供 `AnthropicVertex` 客户端
* **Bedrock：** `npm install @anthropic-ai/bedrock-sdk`：提供 `AnthropicBedrockMantle` 客户端，以及用于 `bedrock-runtime` 路径的 `AnthropicBedrock`
* **Claude Platform on AWS：** `npm install @anthropic-ai/aws-sdk`：提供 `AnthropicAws` 客户端。将 `workspaceId` 传递给构造函数，或设置 `ANTHROPIC_AWS_WORKSPACE_ID` 环境变量。以 beta 形式提供。
* **Foundry：** `npm install @anthropic-ai/foundry-sdk`：提供 `AnthropicFoundry` 客户端

新项目请使用 `AnthropicBedrockMantle`；`AnthropicBedrock` 保留给使用 Bedrock `InvokeModel` API 的现有应用程序。

## 语义化版本控制

此包总体上遵循 [SemVer](https://semver.org/spec/v2.0.0.html) 约定，但某些向后不兼容的更改可能会作为次要版本发布：

1. 仅影响静态类型而不破坏运行时行为的更改。
2. 对库内部的更改，这些内部在技术上是公开的，但并非为外部使用而设计或文档化。
3. 预计在实践中不会影响绝大多数用户的更改。

我们认真对待向后兼容性，以确保您可以获得顺畅的升级体验。

## 常见问题

有关常见问题、issue 和社区支持，请参阅 [GitHub 仓库](https://github.com/anthropics/anthropic-sdk-typescript)。

## 其他资源

* [GitHub 仓库](https://github.com/anthropics/anthropic-sdk-typescript)
* [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)
* [流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)
* [使用 Claude 进行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)
