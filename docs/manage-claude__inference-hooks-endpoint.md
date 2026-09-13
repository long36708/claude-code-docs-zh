---
title: 开发 Inference hooks 集成
url: https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint
description: 构建 AI 安全服务器，用于接收已签名的 Inference hooks 请求、验证这些请求，并返回允许或拒绝的裁决。
---

<Note>
  Inference hooks 目前处于 beta 阶段，面向 Claude Enterprise 组织提供。在 beta 期间，字段名称、请求结构和标头可能会发生变化。
</Note>

Inference hooks 集成是一个 AI 安全服务器：一个由 Anthropic 调用的 HTTPS 服务。对于每个受管控的请求，您的服务器会收到一个携带对话记录的已签名 `POST`，并以允许或拒绝的 "verdict"（裁决）作为响应。本页记录了构建该服务器所需的协议：请求和裁决的模式、签名验证以及运行契约。

要启用 Inference hooks 并将其指向您的端点，请参阅[配置 Inference hooks](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)。要了解 Inference hooks 是什么以及何时使用它们，请参阅 [Inference hooks 概述](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks)。

## 完成第一次裁决往返

最小的可用集成是一个读取每个请求并允许它的服务器。运行以下服务器之一，将其暴露在一个公共 `https://` URL 上（例如，放在您控制的主机上的 TLS 终止反向代理之后，而不是反向隧道服务；请参阅[接收请求](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#receive-a-request)），然后让您的管理员[将其设置为端点并测试连接](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)：**Test connection**（测试连接）结果会报告您的服务器返回的允许裁决。

<CodeGroup exclude="shell">
  ```python Python
  # 运行方式：python server.py
  from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer


  class VerdictHandler(BaseHTTPRequestHandler):
      protocol_version = "HTTP/1.1"  # keep the connection open between verdicts

      def do_POST(self):
          # 读取并丢弃请求体；转录内容可能达数兆字节。
          self.rfile.read(int(self.headers.get("Content-Length", 0)))
          verdict = b'{"action": "allow"}'
          self.send_response(200)
          self.send_header("Content-Type", "application/json")
          self.send_header("Content-Length", str(len(verdict)))
          self.end_headers()
          self.wfile.write(verdict)


  ThreadingHTTPServer(("", 8000), VerdictHandler).serve_forever()
  ```

  ```typescript TypeScript
  // 运行方式：node server.ts
  import { createServer } from "node:http";

  createServer((request, response) => {
    // 在响应之前先读取完请求体；转录内容可能有数兆字节。
    request.resume();
    request.on("end", () => {
      response.writeHead(200, { "Content-Type": "application/json" });
      response.end('{"action": "allow"}');
    });
  }).listen(8000);
  ```

  ```csharp C#
  #:sdk Microsoft.NET.Sdk.Web
  #:property PublishAot=false
  // Run with: dotnet run server.cs

  var app = WebApplication.Create();

  app.MapPost("/{**path}", async (HttpRequest request) =>
  {
      // Drain the body; transcripts can be megabytes.
      await request.Body.CopyToAsync(Stream.Null);
      return Results.Text("""{"action": "allow"}""", "application/json");
  });

  app.Run("http://0.0.0.0:8000");
  ```

  ```go Go
  // 运行方式：go run server.go
  package main

  import (
  	"io"
  	"log"
  	"net/http"
  )

  func main() {
  	http.HandleFunc("POST /", func(writer http.ResponseWriter, request *http.Request) {
  		// 读取并丢弃响应体以便复用连接；转录内容可能达数兆字节。
  		io.Copy(io.Discard, request.Body)
  		writer.Header().Set("Content-Type", "application/json")
  		writer.Write([]byte(`{"action": "allow"}`))
  	})
  	log.Fatal(http.ListenAndServe(":8000", nil))
  }
  ```

  ```java Java
  // 运行方式：java VerdictServer.java
  import com.sun.net.httpserver.HttpServer;

  void main() throws IOException {
      HttpServer server = HttpServer.create(new InetSocketAddress(8000), 0);
      server.createContext("/", exchange -> {
          // 直接排空请求体而不进行缓冲；转录内容可能达数兆字节。
          exchange.getRequestBody().transferTo(OutputStream.nullOutputStream());
          byte[] verdict = "{\"action\": \"allow\"}".getBytes(StandardCharsets.UTF_8);
          exchange.getResponseHeaders().set("Content-Type", "application/json");
          exchange.sendResponseHeaders(200, verdict.length);
          try (OutputStream responseBody = exchange.getResponseBody()) {
              responseBody.write(verdict);
          }
      });
      server.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
      server.start();
  }
  ```

  ```php PHP
  <?php
  // 运行方式：php -S 0.0.0.0:8000 server.php

  // 读取并丢弃请求体；转录内容可能达数兆字节。
  file_get_contents('php://input');

  http_response_code(200);
  header('Content-Type: application/json');
  echo '{"action": "allow"}';
  ```

  ```ruby Ruby
  # webrick 在 Ruby 3.4 中是普通 gem：运行 gem install webrick，或添加 gem "webrick"。
  # 运行方式：ruby server.rb
  require "webrick"

  server = WEBrick::HTTPServer.new(Port: 8000)
  server.mount_proc("/") do |request, response|
    request.body # Drain the body; transcripts can be megabytes.
    response.status = 200
    response["Content-Type"] = "application/json"
    response.body = '{"action": "allow"}'
  end
  server.start
  ```
</CodeGroup>

<Note>
  这些服务器接受所有请求，包括未签名的请求。在强制执行之前，请添加[签名验证](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#verify-the-signature)。
</Note>

## 接收请求

Anthropic 会向您的管理员配置的 URL 发送一个 HTTPS `POST`。整个配置的 URL 就是端点：没有固定的路径后缀，因此可以选择任何适合您服务器的路径。

请将您的 AI 安全服务器托管在 Anthropic 可以访问的位置：一个位于 443 端口的 `https://` URL，位于可公开路由的主机上（私有、环回和运营商级 NAT 地址范围会在连接时被拒绝），其证书可通过公共 CA 信任库验证，并且响应时不进行重定向。配置的 URL 必须是最终目的地。不支持反向隧道主机（ngrok 及类似的隧道服务）：Anthropic 的网络策略会阻止它们。请将您的服务器托管在您控制的域名上。[配置 Inference hooks](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration) 介绍了您的管理员如何设置和测试该 URL。

每个请求都携带以下固定标头，以及您的管理员配置的任何[自定义请求标头](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)，并且一旦您的组织拥有签名密钥，还会携带[验证签名](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#verify-the-signature)中描述的 `webhook-*` 签名标头：

| 标头                | 值                  |
| ----------------- | ------------------ |
| `Content-Type`    | `application/json` |
| `User-Agent`      | `anthropic-dlp/1`  |
| `Accept-Encoding` | `identity`         |

目前只有一种 hook 事件："prompt frame"（提示帧），每个受管控的推理请求在推理开始之前发送一次。Anthropic 会挂起该请求，直到您的 AI 安全服务器响应或裁决超时时间到期。

## 提示帧

请求正文是一个包含以下字段的 JSON 对象：

| 字段           | 类型            | 描述                                                                                                                                                                   |
| ------------ | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`       | string        | hook 事件。目前始终为 `"prompt"`；未来会引入其他事件类型，因此请妥善处理无法识别的值（请参阅[向前兼容性](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#forward-compatibility)）。 |
| `request_id` | string        | 用于关联的不透明的每次推理调用标识符。等于 `webhook-id` 标头。                                                                                                                               |
| `tenant_id`  | string 或 null | 请求所属组织的不透明标识符。                                                                                                                                                       |
| `actor`      | object        | 请求所归属的主体，按 `type` 区分（`"user"` 是目前发送的唯一值）：`id`（带标签的标识符，对于同一账户在各请求之间保持稳定）和 `email_address`（如可用）。`id` 和 `email_address` 都可以为 null。                                      |
| `source`     | object        | 发起请求的应用程序：`application`（请参阅 [Source 值](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#source-values)）。                                |
| `messages`   | array         | 截至推理时刻的对话记录。请参阅[内容块](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#content-blocks)。                                                  |
| `session_id` | string 或 null | 不透明的对话标识符（如存在）。不要解析它。对于 Claude Code，它是一个尽力而为的、由客户端声明的会话标识符。                                                                                                          |
| `model`      | string 或 null | 此请求的公开模型标识符（如可用）。                                                                                                                                                    |
| `metadata`   | object        | 保留的扩展映射，键和值均为字符串，目前发送为空。不要对其有任何要求，并容忍它的缺失、它的存在以及出现的任何键。                                                                                                              |

<Note>
  请求目前还携带其中某些字段的已弃用旧别名。请读取本页记录的字段名称并忽略任何其他字段；这些别名仅为早期集成而存在。
</Note>

请求正文示例：

```json
{
  "type": "prompt",
  "request_id": "req_abc123",
  "tenant_id": "11111111-1111-1111-1111-111111111111",
  "actor": {
    "type": "user",
    "id": "user_01AbCdEfGhIjKlMnOpQrStUv",
    "email_address": "alice@example.com"
  },
  "source": {
    "application": "claude-ai"
  },
  "session_id": "22222222-2222-2222-2222-222222222222",
  "model": "claude-sonnet-4-5",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "Summarize the attached report."
        },
        {
          "type": "attachment",
          "file_name": "q2-report.pdf",
          "media_type": "application/pdf",
          "size_bytes": 48213,
          "text": "Q2 revenue grew 14% quarter over quarter..."
        }
      ]
    }
  ],
  "metadata": {}
}
```

### 内容块

`messages` 中的每个条目都有一个值为 `user` 或 `assistant` 的 `role`（工具结果出现在 `user` 角色下，与公开的 Messages API 内容模型一致），以及一个按 `type` 区分的块组成的 `content` 数组：

| 块 `type`      | 字段                                                                                                                                                               |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `text`        | `text`：文本内容。                                                                                                                                                     |
| `tool_use`    | `id`：匹配的工具结果所引用的标识符。`tool_name`：工具的名称。`input`：模型传递给工具的参数。                                                                                                        |
| `tool_result` | `content`：工具的文本输出，各部分以换行符连接；图像等二进制部分会被占位标记替换，且绝不会发送原始字节。`is_error`：工具调用是否失败。`tool_name`：工具的名称，以便策略可以根据工具身份进行判断，而无需交叉引用先前的块。`tool_use_id`：匹配的 `tool_use` 块的 `id`。 |
| `attachment`  | `file_name`：原始文件名或路径。`media_type`：附件的媒体类型。`size_bytes`：原始文件的大小。`text`：附件的文本内容（如可用），例如提取的文档文本、音频转录或链接元数据。绝不会发送原始附件字节。                                             |

`type` 无法识别的块是向前兼容的新增内容。它唯一保证的字段是 `type`；您的策略可以检查存在的任何其他字段，但不得因为无法识别的类型而拒绝请求。

### 对话记录包含的内容

对话记录是最终用户所看到的对话，截至推理时刻：对话文本、工具调用及其结果、提取的附件文本以及先前的轮次。它绝不包含系统提示、工具定义、Anthropic 内部上下文、Claude 的隐藏推理或原始文件字节。

如果某一轮次的每个块都被排除，则该轮次会被完全省略，因此不要假设 user 和 assistant 严格交替。

对话记录不经截断发送，因此带有大型附件的长对话会产生较大的请求正文，上限为 10 MB。请提高您服务器的正文大小限制以接受该上限。一些常见的默认值要小得多，包括 nginx 的 `client_max_body_size` 为 1 MB，Express 的 `express.json()` 为 100 kB，而被拒绝的正文会被计为 webhook 失败，因此在 **Allow the request**（允许请求）失败处理设置下，超大的提示将未经检查地到达模型。

### Source 值

`source.application` 是一个开放字符串，而不是封闭枚举。已知值为 `claude-ai` 和 `claude-code`；[连接测试](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)使用 `config-test`。可能会出现新值，您的服务器不得因为无法识别的值而拒绝请求。

请将 `source.application` 视为建议性的路由元数据，而不是信任边界：不要仅凭它做出安全关键的策略决定。

## 返回裁决

对于两种结果，均以 HTTP 200 和 JSON 裁决正文响应；由 `action` 字段区分。要允许请求：

```json
{
  "action": "allow"
}
```

要拒绝请求：

```json
{
  "action": "deny",
  "deny_reason": "This prompt appears to contain customer payment card data, which your organization's policy does not allow.",
  "reference_id": "scan_01HXPT4R9V"
}
```

| 字段             | 约束                                            | 语义                                                                                                                                                                                     |
| -------------- | --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `action`       | `"allow"` 或 `"deny"`；必填                       | `allow` 让推理继续进行；`deny` 拒绝它。                                                                                                                                                            |
| `deny_reason`  | string 或 null；最多 500 个字符，更长的值会被截断             | 当 `action` 为 `deny` 时向最终用户显示；在 `allow` 时被忽略。                                                                                                                                           |
| `reference_id` | string 或 null；最多 50 个字符，取自 `[A-Za-z0-9._:/-]` | 您自己为此次评估设定的标识符。它会记录在该拒绝的 `inference_hooks_request_denied` [合规活动](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)中，且绝不会向最终用户显示。请保持其不透明：不包含请求内容，也不包含个人数据。 |

拒绝绝不会因格式问题而被丢弃：超长的 `deny_reason` 会被截断，格式错误的 `reference_id` 会被静默丢弃，而 `action` 仍会被执行。

反之则不成立。除带有可解析裁决的 HTTP 200 之外的任何响应都是 webhook 失败，此时将应用您组织的[失败处理](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)设置而非裁决。特别是：

* 不要用错误状态码来表示拒绝。非 200 响应是失败，而不是拒绝。
* 除 `allow` 或 `deny` 之外的任何 `action` 值都被视为 webhook 失败。

Anthropic 最多读取 64 KiB 的响应正文，且正文必须未经压缩。不会跟随重定向，并且会忽略 cookie。裁决正文中的未知字段会被忽略，因此您可以在本页记录的字段之外返回更丰富的对象。

## 验证签名

请求按照 [Standard Webhooks](https://www.standardwebhooks.com/) 规范使用三个标头进行签名。Anthropic 以小写形式发送标头名称，而代理可以自由更改其大小写，因此请以不区分大小写的方式查找它们。

| 标头                  | 内容                                                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `webhook-id`        | 此次投递的唯一标识符。等于正文的 `request_id`。将其用作幂等键，并作为签名载荷的第一个组成部分。                                                                                |
| `webhook-timestamp` | 请求签名时的 Unix 时间（以秒为单位，十进制字符串）。拒绝与您服务器时钟相差超过五分钟（任一方向）的时间戳。                                                                              |
| `webhook-signature` | 一个或多个以空格分隔的 `v1,<base64>` 值，每个值都是对 `{webhook-id}.{webhook-timestamp}.{raw body bytes}` 计算的 HMAC-SHA256。如果任一值与您计算的值匹配，则接受请求，并使用常量时间比较。 |

大多数验证错误由以下两个细节引起：

* **验证原始字节。** 对收到的正文原样计算 HMAC，在任何 JSON 解析或重新编码之前进行。
* **使用标准 base64 解码器解码密钥。** 签名密钥是 `whsec_` 前缀之后的值，使用标准 base64 字母表（`+` 和 `/`）编码，标头中的签名也是如此。只要密钥包含 `+` 或 `/`（大多数情况下都是如此），URL 安全解码器就会得出错误的密钥字节。

一旦您的组织拥有签名密钥，Anthropic 发送的每个请求都会被签名，并且[启用 Inference hooks 需要签名密钥](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)，因此请拒绝任何未签名到达的请求。有一个例外：在您的组织首次保存之前发送的连接测试会以未签名形式到达，因为签名密钥尚不存在。在您的管理员确认密钥存在之前接受未签名的请求，之后则拒绝它们。

[轮换密钥](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration#rotate-your-signing-secret)是立即切换的，但使用先前密钥签名的请求在之后大约一分钟内仍可能到达，外加任何已在传输中的请求。请让您的 AI 安全服务器在切换期间同时接受两个密钥的签名，以免这些滞后的请求被拒绝。

以下示例是服务器实现，因此没有 shell 选项卡：AI 安全服务器是一个长期运行的 HTTPS 服务，而不是一次性请求。每个示例仅使用该语言的标准库；[Standard Webhooks](https://www.standardwebhooks.com/) 项目也为大多数语言发布了验证库。

<CodeGroup exclude="shell">
  ```python Python
  import base64
  import hashlib
  import hmac
  import time

  TOLERANCE_SECONDS = 300


  def verify(secret: str, headers: dict[str, str], body: bytes) -> bool:
      """Return True if the body was signed by Anthropic for this organization.

      Anthropic sends header names in lowercase, but proxies are free to
      re-case them, so normalize the lookup to lowercase.
      """
      lowercased = {name.lower(): value for name, value in headers.items()}
      try:
          message_id = lowercased["webhook-id"]
          timestamp = lowercased["webhook-timestamp"]
          signatures = lowercased["webhook-signature"]
      except KeyError:
          return False  # unsigned request: not from Anthropic

      try:
          signed_at = int(timestamp)
      except ValueError:
          return False
      if abs(time.time() - signed_at) > TOLERANCE_SECONDS:
          return False  # replayed, or the clocks disagree

      try:
          key = base64.b64decode(secret.removeprefix("whsec_"), validate=True)
      except ValueError:
          return False  # misconfigured secret: reject rather than crash

      payload = f"{message_id}.{timestamp}.".encode() + body
      expected = b"v1," + base64.b64encode(
          hmac.new(key, payload, hashlib.sha256).digest()
      )

      # 比较字节：compare_digest 对 str 遇到非 ASCII 输入时会抛出异常。
      return any(
          hmac.compare_digest(expected, candidate.encode())
          for candidate in signatures.split()
      )
  ```

  ```typescript TypeScript
  import { createHmac, timingSafeEqual } from "node:crypto";
  import type { IncomingHttpHeaders } from "node:http";

  const TOLERANCE_SECONDS = 300;

  /**
   * Returns true if the body was signed by Anthropic for this organization.
   *
   * Node lowercases incoming header names, matching how Anthropic sends
   * them, so look them up in lowercase.
   */
  export function verify(secret: string, headers: IncomingHttpHeaders, body: Buffer): boolean {
    const messageId = headers["webhook-id"];
    const timestamp = headers["webhook-timestamp"];
    const signatures = headers["webhook-signature"];
    if (
      typeof messageId !== "string" ||
      typeof timestamp !== "string" ||
      typeof signatures !== "string"
    ) {
      return false; // unsigned request: not from Anthropic
    }

    const signedAt = Number(timestamp);
    if (
      !Number.isFinite(signedAt) ||
      Math.abs(Date.now() / 1000 - signedAt) > TOLERANCE_SECONDS
    ) {
      return false; // replayed, or the clocks disagree
    }

    const key = Buffer.from(secret.replace(/^whsec_/, ""), "base64");
    const payload = Buffer.concat([Buffer.from(`${messageId}.${timestamp}.`), body]);
    const expected = Buffer.from(
      "v1," + createHmac("sha256", key).update(payload).digest("base64")
    );

    return signatures.split(" ").some((candidate) => {
      const candidateBytes = Buffer.from(candidate);
      return (
        candidateBytes.length === expected.length && timingSafeEqual(candidateBytes, expected)
      );
    });
  }
  ```

  ```csharp C#
  using System.Security.Cryptography;
  using System.Text;

  static class InferenceHooks
  {
      private const int ToleranceSeconds = 300;

      /// <summary>
      /// 如果请求体由 Anthropic 为该组织签名，则返回 true。
      /// Anthropic 发送的标头名称为小写，但代理可以自由地
      /// 更改其大小写，因此应以不区分大小写的方式进行匹配。
      /// </summary>
      public static bool Verify(string secret, IReadOnlyDictionary<string, string> headers, byte[] body)
      {
          // 如果代理传递了大小写重复的名称，TryAdd 会保留第一个值；
          // 而复制构造函数则会对此抛出异常。
          var lookup = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
          foreach (var (name, value) in headers)
          {
              lookup.TryAdd(name, value);
          }

          if (!lookup.TryGetValue("webhook-id", out var messageId) ||
              !lookup.TryGetValue("webhook-timestamp", out var timestamp) ||
              !lookup.TryGetValue("webhook-signature", out var signatures))
          {
              return false; // unsigned request: not from Anthropic
          }

          if (!long.TryParse(timestamp, out var signedAt) ||
              Math.Abs(DateTimeOffset.UtcNow.ToUnixTimeSeconds() - signedAt) > ToleranceSeconds)
          {
              return false; // replayed, or the clocks disagree
          }

          // 标准 base64 字母表：URL 安全的解码器会推导出错误的密钥字节。
          var encodedKey = secret.StartsWith("whsec_") ? secret["whsec_".Length..] : secret;
          byte[] key;
          try
          {
              key = Convert.FromBase64String(encodedKey);
          }
          catch (FormatException)
          {
              return false; // misconfigured secret: reject rather than crash
          }

          byte[] payload = [.. Encoding.UTF8.GetBytes($"{messageId}.{timestamp}."), .. body];
          var expected = Encoding.UTF8.GetBytes(
              "v1," + Convert.ToBase64String(HMACSHA256.HashData(key, payload)));

          // FixedTimeEquals 为常量时间比较，长度不匹配时返回 false。
          return signatures.Split(' ', StringSplitOptions.RemoveEmptyEntries).Any(candidate =>
              CryptographicOperations.FixedTimeEquals(Encoding.UTF8.GetBytes(candidate), expected));
      }
  }
  ```

  ```go Go
  package hooks

  import (
  	"crypto/hmac"
  	"crypto/sha256"
  	"encoding/base64"
  	"net/http"
  	"strconv"
  	"strings"
  	"time"
  )

  const toleranceSeconds = 300

  // verify 报告 body 是否由 Anthropic 为该组织签名。
  // net/http 在查找时会规范化标头名称，因此大小写改变后的名称仍能匹配。
  func verify(secret string, header http.Header, body []byte) bool {
  	messageID := header.Get("webhook-id")
  	timestamp := header.Get("webhook-timestamp")
  	signatures := header.Get("webhook-signature")
  	if messageID == "" || timestamp == "" || signatures == "" {
  		return false // unsigned request: not from Anthropic
  	}

  	signedAt, err := strconv.ParseInt(timestamp, 10, 64)
  	if err != nil {
  		return false
  	}
  	age := time.Now().Unix() - signedAt
  	if age > toleranceSeconds || age < -toleranceSeconds {
  		return false // replayed, or the clocks disagree
  	}

  	// 标准 base64 字母表：URL 安全的解码器会得出错误的密钥字节。
  	key, err := base64.StdEncoding.DecodeString(strings.TrimPrefix(secret, "whsec_"))
  	if err != nil {
  		return false
  	}

  	mac := hmac.New(sha256.New, key)
  	mac.Write([]byte(messageID + "." + timestamp + "."))
  	mac.Write(body)
  	expected := "v1," + base64.StdEncoding.EncodeToString(mac.Sum(nil))

  	for _, candidate := range strings.Fields(signatures) {
  		if hmac.Equal([]byte(candidate), []byte(expected)) { // constant-time
  			return true
  		}
  	}
  	return false
  }
  ```

  ```java Java
  import java.nio.charset.StandardCharsets;
  import java.security.GeneralSecurityException;
  import java.security.MessageDigest;
  import java.time.Instant;
  import java.util.Base64;
  import java.util.HashMap;
  import java.util.Locale;
  import java.util.Map;
  import javax.crypto.Mac;
  import javax.crypto.spec.SecretKeySpec;

  public final class InferenceHookVerifier {
      private static final long TOLERANCE_SECONDS = 300;

      /**
       * Returns true if the body was signed by Anthropic for this organization.
       *
       * <p>Anthropic sends header names in lowercase, but proxies are free to
       * re-case them, so normalize the lookup to lowercase.
       */
      public static boolean verify(String secret, Map<String, String> headers, byte[] body) {
          Map<String, String> lowercased = new HashMap<>();
          headers.forEach((name, value) -> lowercased.put(name.toLowerCase(Locale.ROOT), value));

          String messageId = lowercased.get("webhook-id");
          String timestamp = lowercased.get("webhook-timestamp");
          String signatures = lowercased.get("webhook-signature");
          if (messageId == null || timestamp == null || signatures == null) {
              return false; // unsigned request: not from Anthropic
          }

          long signedAt;
          try {
              signedAt = Long.parseLong(timestamp);
          } catch (NumberFormatException _) {
              return false;
          }
          if (Math.abs(Instant.now().getEpochSecond() - signedAt) > TOLERANCE_SECONDS) {
              return false; // replayed, or the clocks disagree
          }

          // 标准 base64 字母表：URL 安全的解码器会推导出错误的密钥字节。
          byte[] key;
          try {
              key = Base64.getDecoder().decode(
                      secret.startsWith("whsec_") ? secret.substring("whsec_".length()) : secret);
          } catch (IllegalArgumentException _) {
              return false; // misconfigured secret: reject rather than crash
          }

          byte[] expected;
          try {
              Mac mac = Mac.getInstance("HmacSHA256");
              mac.init(new SecretKeySpec(key, "HmacSHA256"));
              mac.update((messageId + "." + timestamp + ".").getBytes(StandardCharsets.UTF_8));
              expected = ("v1," + Base64.getEncoder().encodeToString(mac.doFinal(body)))
                      .getBytes(StandardCharsets.UTF_8);
          } catch (GeneralSecurityException impossible) {
              // 每个 JVM 都自带 HmacSHA256，因此这在运行时永远不会触发。
              throw new IllegalStateException(impossible);
          }

          for (String candidate : signatures.split(" ")) {
              if (MessageDigest.isEqual(candidate.getBytes(StandardCharsets.UTF_8), expected)) {
                  return true; // MessageDigest.isEqual is constant-time
              }
          }
          return false;
      }
  }
  ```

  ```php PHP
  const TOLERANCE_SECONDS = 300;

  /**
   * Returns true if the body was signed by Anthropic for this organization.
   *
   * Anthropic sends header names in lowercase, but proxies are free to
   * re-case them, so normalize the lookup to lowercase.
   */
  function verify(string $secret, array $headers, string $body): bool
  {
      $lowercased = array_change_key_case($headers, CASE_LOWER);
      $messageId = $lowercased['webhook-id'] ?? null;
      $timestamp = $lowercased['webhook-timestamp'] ?? null;
      $signatures = $lowercased['webhook-signature'] ?? null;
      if ($messageId === null || $timestamp === null || $signatures === null) {
          return false; // unsigned request: not from Anthropic
      }

      $signedAt = filter_var($timestamp, FILTER_VALIDATE_INT);
      if ($signedAt === false || abs(time() - $signedAt) > TOLERANCE_SECONDS) {
          return false; // replayed, or the clocks disagree
      }

      // 标准 base64 字母表：URL 安全的解码器会推导出错误的密钥字节。
      $encodedKey = str_starts_with($secret, 'whsec_') ? substr($secret, strlen('whsec_')) : $secret;
      $key = base64_decode($encodedKey, strict: true);
      if ($key === false) {
          return false;
      }

      $payload = "{$messageId}.{$timestamp}." . $body;
      $expected = 'v1,' . base64_encode(hash_hmac('sha256', $payload, $key, binary: true));

      foreach (explode(' ', $signatures) as $candidate) {
          if (hash_equals($expected, $candidate)) { // constant-time
              return true;
          }
      }
      return false;
  }
  ```

  ```ruby Ruby
  # base64 在 Ruby 3.4 中是捆绑 gem：由 Bundler 管理的应用需添加 gem "base64"。
  require "base64"
  require "openssl"

  TOLERANCE_SECONDS = 300

  # 如果请求体由 Anthropic 为该组织签名，则返回 true。
  #
  # Anthropic 发送的标头名称为小写，但代理可以自由地
  # 更改其大小写，因此将查找统一规范化为小写。
  def verify(secret, headers, body)
    lowercased = headers.transform_keys(&:downcase)
    message_id = lowercased["webhook-id"]
    timestamp = lowercased["webhook-timestamp"]
    signatures = lowercased["webhook-signature"]
    if message_id.nil? || timestamp.nil? || signatures.nil?
      return false # unsigned request: not from Anthropic
    end

    signed_at = Integer(timestamp, exception: false)
    if signed_at.nil? || (Time.now.to_i - signed_at).abs > TOLERANCE_SECONDS
      return false # replayed, or the clocks disagree
    end

    # 标准 base64 字母表：URL 安全的解码器会推导出错误的密钥字节。
    begin
      key = Base64.strict_decode64(secret.delete_prefix("whsec_"))
    rescue ArgumentError
      return false # misconfigured secret: reject rather than crash
    end

    # 单独传入请求体，这样其编码就无需与前缀的编码一致。
    hmac = OpenSSL::HMAC.new(key, "SHA256")
    hmac.update("#{message_id}.#{timestamp}.")
    hmac.update(body)
    expected = "v1," + Base64.strict_encode64(hmac.digest)

    signatures.split(" ").any? do |candidate|
      # fixed_length_secure_compare 在长度不匹配时会抛出异常，因此先筛查长度。
      candidate.bytesize == expected.bytesize &&
        OpenSSL.fixed_length_secure_compare(candidate, expected)
    end
  end
  ```
</CodeGroup>

## 运行语义

### 超时和重试

您的管理员可将裁决超时设置在 1 到 10,000 毫秒之间（默认为 5,000 毫秒）。该预算涵盖整个交换过程：连接、TLS 握手、请求和响应。

Anthropic 仅在连接尝试失败时重试，且恰好重试一次，延迟 100 毫秒。重试共享相同的超时预算，并携带相同的 `webhook-id` 和相同的签名。一旦您的 AI 安全服务器已响应，该交换绝不会被重试。

### Webhook 失败

超时、非 200 状态码（包括重定向）、无法解析或超大的响应正文以及无法访问的端点都属于 webhook 失败。webhook 失败绝不会变成拒绝；相反，由您组织的[失败处理](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration)设置决定受影响的请求是被阻止还是未经检查地继续进行。

### 熔断器

可归因于您的 AI 安全服务器的持续 webhook 失败会触发 "circuit breaker"（熔断器），从而停止强制执行：Anthropic 停止联系您的服务器，并对每个请求应用失败处理。

从触发后 10 分钟开始，Anthropic 会测试您的服务器是否已恢复：最多大约每分钟一次，将一个由您组织自身流量承载的请求投递到您的服务器进行检查，其签名和结构与任何其他请求相同。请正常响应它。有效的裁决（允许或拒绝）会重置熔断器并恢复强制执行。webhook 失败会使熔断器保持触发状态，并继续测试。无论哪种情况，测试请求本身都会为其用户继续进行：其裁决不会被强制执行，失败的测试也不会阻止它，即使在 **Block the request**（阻止请求）设置下也是如此。管理员也可以随时重置熔断器，并且管理员的配置更改会停止自动测试；请参阅[熔断器](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration#circuit-breaker)。

每次触发都会在[活动源](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)中记录为一个 `inference_hooks_circuit_breaker_tripped` 活动，每次触发一个活动。在熔断器处于触发状态期间，不会记录每个请求的 Inference hooks 活动，因此触发活动是活动源中关于触发窗口的唯一记录。

### 延迟

强制执行会将您的 AI 安全服务器的往返时间添加到您组织中每个受管控请求的 "latency"（延迟）中。请保持裁决快速，并在向大型组织推出之前对您的服务器进行负载测试。

### 源 IP 地址

发往您的 AI 安全服务器的请求源自 `160.79.106.0/24`，这是 Anthropic 公布的[出站 IP 范围](https://platform.claude.com/docs/zh-CN/api/ip-addresses)的一部分。请将该地址块加入允许列表，而不是同一页面上的入站范围，后者并不涵盖它。允许列表可以缩小您服务器的暴露面，但它不能替代签名验证：该地址块承载的 Anthropic 出口流量不仅限于 Inference hooks。

## 向前兼容性

该协议在不破坏正确编写的服务器的前提下演进。您的服务器必须忽略：

* 提示帧上未知的顶级字段。
* `metadata` 中未知的键。
* 新的 `source.application` 值。
* 新的 `actor.type` 值。`actor` 是按 `type` 区分的联合类型，`"user"` 是目前发送的唯一种类；未来的种类仅保证 `type` 存在。
* `type` 无法识别的内容块。

绝不要因为无法识别的块类型或字段而拒绝请求；读取您已知的字段并跳过其余部分。

未来会引入其他 hook 事件类型。新的事件类型是您的服务器无法通过跳过字段来处理的新增内容：该请求仍然需要裁决。当顶级 `type` 是您无法识别的值时，请返回允许裁决而不是错误状态码；错误响应属于 [webhook 失败](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#webhook-failures)，而持续的失败会触发[熔断器](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#circuit-breaker)。

## 设计您的集成

生产环境的 AI 安全服务器除了线路协议之外还需要做出一些设计选择。

**基于 `webhook-id` 去重。** `webhook-id` 标头对每次投递都是唯一的，并且等于正文的 `request_id`，而连接失败重试会复用它，因此它可以用作幂等键。如果您记录裁决，请以它作为记录的键。

**记录裁决并关联拒绝。** 存储您返回的每个裁决及其 `reference_id`。每次拒绝都会记录为一个 `inference_hooks_request_denied` 合规活动，其中携带您的服务器返回的 `reference_id`，因此您可以将[活动源](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)中的拒绝与您自己系统中的匹配记录关联起来。

**使用始终允许的服务器进行归档。** 要实时捕获对话记录而不对其进行管控，请无条件返回 `{"action": "allow"}`，并在响应后持久化该帧。这是轮询 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 的一种基于推送的替代方案，而在持久化之前先作出响应可以使您的往返时间不进入用户的关键路径。

**为最终用户编写 `deny_reason`。** 您返回的文本就是用户在其请求被阻止时看到的内容，截断为 500 个字符。请告诉他们需要更改什么，例如要删除哪类内容，而不是输出只有您的团队才能解读的扫描器代码。

## 后续步骤

<CardGroup cols={2}>
  <Card title="配置 Inference hooks" href="https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration">
    启用 Inference hooks，连接并测试您的端点，并控制强制执行、失败处理和推出。
  </Card>

  <Card title="Inference hooks 概述" href="https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks">
    Inference hooks 是什么、裁决往返如何工作以及何时使用它们。
  </Card>
</CardGroup>
