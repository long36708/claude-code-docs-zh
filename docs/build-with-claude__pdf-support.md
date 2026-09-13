---
title: PDF 支持
url: https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support
description: 使用 Claude 处理 PDF：从您的文档中提取文本、分析图表并理解视觉内容。
---

## Compatibility
- [ZDR](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): eligible (excludes [Covered Models](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#model-specific-data-retention-requirements))
- Platforms: Claude API, Claude Platform on AWS, Amazon Bedrock, Google Cloud, Microsoft Foundry

您可以向 Claude 询问您所提供的 PDF 中的任何文本、图片、图表和表格。一些示例用例：

* 分析财务报告并理解图表/表格
* 从法律文档中提取关键信息
* 协助文档翻译
* 将文档信息转换为结构化格式

## 开始之前

### 检查 PDF 要求

Claude 可处理任何标准 PDF。请确保您的请求大小满足以下要求：

| 要求        | 限制                                                                                      |
| --------- | --------------------------------------------------------------------------------------- |
| 最大请求大小    | 32 MB（[因平台而异](https://platform.claude.com/docs/zh-CN/api/overview#request-size-limits)） |
| 每个请求的最大页数 | 600（当请求的上下文窗口小于 1M 令牌时为 100）                                                            |
| 格式        | 标准 PDF（无密码/加密）                                                                          |

这两项限制均针对整个请求负载，包括与 PDF 一起发送的任何其他内容。对于大型 PDF，请考虑使用 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传并通过 `file_id` 引用，以保持请求负载较小。

<Tip>
  内容密集的 PDF（大量小字体页面、复杂表格或大量图形）可能会在达到页数限制之前就填满 "context window"（上下文窗口）。包含大型 PDF 的请求也可能在达到页数限制之前失败，即使使用 Files API 也是如此。请尝试将文档拆分为多个部分；对于大文件，由于每一页都会作为图像处理，对嵌入的图像进行降采样也会有所帮助。
</Tip>

由于 PDF 支持依赖于 Claude 的视觉能力，因此它与其他视觉任务一样受到相同的[限制和注意事项](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#limitations)的约束。

### 支持的平台和模型

所有[活跃模型](https://platform.claude.com/docs/zh-CN/models/overview)均支持 PDF 处理。有关通过 Amazon Bedrock 的 Converse API 使用 PDF 支持的信息，请参阅 [Amazon Bedrock PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support#amazon-bedrock-pdf-support)。

### Amazon Bedrock PDF 支持

当通过 Converse API（属于 [Amazon Bedrock 上的 Claude（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)的一部分）使用 PDF 支持时，有两种不同的文档处理模式：

<Note>
  **重要提示：** 要在 Converse API 中使用 Claude 完整的 PDF 视觉理解能力，您必须启用 "citations"（引用）。如果未启用引用，API 将回退为仅进行基本文本提取。了解更多关于[使用引用](https://platform.claude.com/docs/zh-CN/build-with-claude/citations)的信息。
</Note>

#### 文档处理模式

1. **Converse Document Chat**（原始模式 - 仅文本提取）

   * 提供 PDF 的基本文本提取
   * 无法分析 PDF 中的图像、图表或视觉布局
   * 一个 3 页的 PDF 大约使用 1,000 个令牌
   * 未启用引用时自动使用

2. **Claude PDF Chat**（新模式 - 完整视觉理解）

   * 提供 PDF 的完整视觉分析
   * 能够理解和分析图表、图形、图像和视觉布局
   * 将每一页同时作为文本和图像处理，以实现全面理解
   * 一个 3 页的 PDF 大约使用 7,000 个令牌
   * **需要在 Converse API 中启用引用**

#### 主要限制

* **Converse API：** PDF 视觉分析需要启用引用。目前没有在不启用引用的情况下使用视觉分析的选项（与 InvokeModel API 不同）。
* **InvokeModel API：** 提供对 PDF 处理的完全控制，无需强制启用引用。

#### 常见问题

如果在使用 Converse API 时 Claude 看不到您 PDF 中的图像或图表，您很可能需要启用引用标志。如果没有启用，Converse 将回退为仅进行基本文本提取。

<Note>
  这是 Converse API 的一个已知限制。对于需要在不启用引用的情况下进行 PDF 视觉分析的应用程序，请考虑改用 InvokeModel API。
</Note>

<Note>
  纯文本文件（如 .txt、.csv 或 .md）可以直接在文档块中使用：将它们以 MIME 类型 `text/plain` 上传到 Files API，并通过 `file_id` 引用。二进制格式（如 .xlsx 或 .docx）在文档块中不受支持，必须先转换为文本或 PDF。请参阅[处理其他文件格式](https://platform.claude.com/docs/zh-CN/build-with-claude/files#working-with-other-file-formats)。
</Note>

## 使用 Claude 处理 PDF

### 发送您的第一个 PDF 请求

从一个使用 Messages API 的简单示例开始。您可以通过三种方式向 Claude 提供 PDF：

1. 作为指向在线托管 PDF 的 URL 引用
2. 作为 `document` 内容块中的 base64 编码 PDF
3. 通过来自 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 的 `file_id`

<Note>
  在 Amazon Bedrock 和 Google Cloud 上，目前仅支持 base64 编码的来源。在 Microsoft Foundry 上，托管在 Azure 上的部署不支持 Files API。
</Note>

#### 选项 1：基于 URL 的 PDF 文档

最简单的方法是直接从 URL 引用 PDF：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{
          "role": "user",
          "content": [{
              "type": "document",
              "source": {
                  "type": "url",
                  "url": "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
              }
          },
          {
              "type": "text",
              "text": "What are the key findings in this document?"
          }]
      }]
  }'
  ```

  ```bash CLI
  ant messages create --transform content --format yaml <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: document
          source:
            type: url
            url: https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf
        - type: text
          text: What are the key findings in this document?
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()
  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "document",
                      "source": {
                          "type": "url",
                          "url": "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf",
                      },
                  },
                  {"type": "text", "text": "What are the key findings in this document?"},
              ],
          }
      ],
  )

  print(message.content)
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic();

  const response = await anthropic.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "url",
              url: "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
            }
          },
          {
            type: "text",
            text: "What are the key findings in this document?"
          }
        ]
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  var client = new AnthropicClient();

  // 使用 URL 创建文档块
  var documentParam = new DocumentBlockParam
  {
      Source = new UrlPdfSource
      {
          Url = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf",
      },
  };

  // 创建包含文档和文本内容块的消息
  var message = await client.Messages.Create(new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = new List<ContentBlockParam>
              {
                  documentParam,
                  new TextBlockParam("What are the key findings in this document?"),
              },
          },
      ],
  });

  Console.WriteLine(string.Join("\n", message.Content));
  ```

  ```go Go
  client := anthropic.NewClient()

  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.NewDocumentBlock(anthropic.URLPDFSourceParam{
  				URL: "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf",
  			}),
  			anthropic.NewTextBlock("What are the key findings in this document?"),
  		),
  	},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Printf("%+v\n", message.Content)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  // 使用 URL 创建文档块
  DocumentBlockParam documentParam = DocumentBlockParam.builder()
    .source(
      UrlPdfSource.builder()
        .url(
          "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
        )
        .build()
    )
    .build();

  // 创建包含文档和文本内容块的消息
  MessageCreateParams params = MessageCreateParams.builder()
    .model(Model.CLAUDE_OPUS_5)
    .maxTokens(1024)
    .addUserMessageOfBlockParams(
      List.of(
        ContentBlockParam.ofDocument(documentParam),
        ContentBlockParam.ofText(
          TextBlockParam.builder()
            .text("What are the key findings in this document?")
            .build()
        )
      )
    )
    .build();

  Message message = client.messages().create(params);
  System.out.println(message.content());
  ```

  ```php PHP
  $client = new Client();

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'document',
                      'source' => [
                          'type' => 'url',
                          'url' => 'https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf',
                      ],
                  ],
                  [
                      'type' => 'text',
                      'text' => 'What are the key findings in this document?',
                  ],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo $message;
  ```

  ```ruby Ruby
  anthropic = Anthropic::Client.new

  message = anthropic.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "url",
              url: "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
            }
          },
          {type: "text", text: "What are the key findings in this document?"}
        ]
      }
    ]
  )

  puts(message.content)
  ```
</CodeGroup>

响应会在 `content` 中以文本块的形式返回 Claude 的分析，并在 `usage` 中返回令牌消耗情况：

```json Output
{
  "id": "msg_01Hfp8YuFjQ55VgWbpdHDehB",
  "type": "message",
  "role": "assistant",
  "model": "claude-opus-5",
  "content": [
    {
      "type": "text",
      "text": "This document is an addendum to the Claude 3 model card, reporting updated evaluation results. The key findings include..."
    }
  ],
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 45000,
    "output_tokens": 300
  }
}
```

#### 选项 2：Base64 编码的 PDF 文档

如果您需要从本地系统发送 PDF，或者在没有可用 URL 的情况下：

<CodeGroup>
  ```bash cURL
  # 方法 1：获取并编码远程 PDF
  curl -sL "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf" | base64 | tr -d '\n' > pdf_base64.txt

  # 方法 2：编码本地 PDF 文件
  # base64 document.pdf | tr -d '\n' > pdf_base64.txt

  # 使用 pdf_base64.txt 的内容创建 JSON 请求文件
  jq -n --rawfile PDF_BASE64 pdf_base64.txt '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{
          "role": "user",
          "content": [{
              "type": "document",
              "source": {
                  "type": "base64",
                  "media_type": "application/pdf",
                  "data": $PDF_BASE64
              }
          },
          {
              "type": "text",
              "text": "What are the key findings in this document?"
          }]
      }]
  }' > request.json

  # 使用该 JSON 文件发送 API 请求
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d @request.json
  ```

  ```bash CLI
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --transform content \
    --format yaml <<'YAML'
  messages:
    - role: user
      content:
        - type: document
          source:
            type: base64
            media_type: application/pdf
            data: "@./document.pdf"
        - type: text
          text: What are the key findings in this document?
  YAML
  ```

  ```python Python
  import base64
  import httpx2

  # 首先，加载并编码 PDF
  pdf_url = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
  pdf_data = base64.standard_b64encode(
      httpx2.get(pdf_url, follow_redirects=True).content
  ).decode("utf-8")

  # 替代方案：从本地文件加载
  # with open("document.pdf", "rb") as f:
  #     pdf_data = base64.standard_b64encode(f.read()).decode("utf-8")

  # 使用 base64 编码发送给 Claude
  client = anthropic.Anthropic()
  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "document",
                      "source": {
                          "type": "base64",
                          "media_type": "application/pdf",
                          "data": pdf_data,
                      },
                  },
                  {"type": "text", "text": "What are the key findings in this document?"},
              ],
          }
      ],
  )

  print(message.content)
  ```

  ```typescript TypeScript
  // 方法 1：获取远程 PDF 并进行编码
  const pdfURL =
    "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf";
  const pdfResponse = await fetch(pdfURL);
  const arrayBuffer = await pdfResponse.arrayBuffer();
  const pdfBase64 = Buffer.from(arrayBuffer).toString("base64");

  // 方法 2：从本地文件加载
  // import { readFile } from "node:fs/promises";
  // const pdfBase64 = (await readFile('document.pdf')).toString('base64');

  // 发送包含 base64 编码 PDF 的 API 请求
  const anthropic = new Anthropic();
  const response = await anthropic.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "base64",
              media_type: "application/pdf",
              data: pdfBase64
            }
          },
          {
            type: "text",
            text: "What are the key findings in this document?"
          }
        ]
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  var client = new AnthropicClient();

  // 方法 1：下载远程 PDF 并进行编码
  var pdfUrl = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf";
  using var httpClient = new HttpClient();
  var pdfBase64 = Convert.ToBase64String(await httpClient.GetByteArrayAsync(pdfUrl));

  // 方法 2：从本地文件加载
  // var pdfBase64 = Convert.ToBase64String(await File.ReadAllBytesAsync("document.pdf"));

  // 使用 base64 数据创建文档块
  var documentParam = new DocumentBlockParam
  {
      Source = new Base64PdfSource { Data = pdfBase64 },
  };

  // 创建包含文档和文本内容块的消息
  var message = await client.Messages.Create(new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = new List<ContentBlockParam>
              {
                  documentParam,
                  new TextBlockParam("What are the key findings in this document?"),
              },
          },
      ],
  });

  Console.WriteLine(string.Join("\n", message.Content));
  ```

  ```go Go
  // 首先，加载并编码 PDF
  pdfURL := "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
  resp, err := http.Get(pdfURL)
  if err != nil {
  	panic(err)
  }
  defer resp.Body.Close()
  pdfBytes, err := io.ReadAll(resp.Body)
  if err != nil {
  	panic(err)
  }
  pdfBase64 := base64.StdEncoding.EncodeToString(pdfBytes)

  // 备选方案：从本地文件加载（在 imports 中添加 "os"）
  // pdfBytes, err := os.ReadFile("document.pdf")
  // pdfBase64 := base64.StdEncoding.EncodeToString(pdfBytes)

  // 使用 base64 编码发送给 Claude
  client := anthropic.NewClient()
  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.NewDocumentBlock(anthropic.Base64PDFSourceParam{
  				Data: pdfBase64,
  			}),
  			anthropic.NewTextBlock("What are the key findings in this document?"),
  		),
  	},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Printf("%+v\n", message.Content)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  // 方法 1：下载远程 PDF 并进行编码
  String pdfUrl =
    "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf";
  HttpClient httpClient = HttpClient.newBuilder().followRedirects(HttpClient.Redirect.NORMAL).build();
  HttpRequest request = HttpRequest.newBuilder().uri(URI.create(pdfUrl)).GET().build();

  HttpResponse<byte[]> response = httpClient.send(
    request,
    HttpResponse.BodyHandlers.ofByteArray()
  );
  String pdfBase64 = Base64.getEncoder().encodeToString(response.body());

  // 方法 2：从本地文件加载
  // byte[] fileBytes = Files.readAllBytes(Path.of("document.pdf"));
  // String pdfBase64 = Base64.getEncoder().encodeToString(fileBytes);

  // 使用 base64 数据创建文档块
  DocumentBlockParam documentParam = DocumentBlockParam.builder()
    .source(Base64PdfSource.builder().data(pdfBase64).build())
    .build();

  // 创建包含文档和文本内容块的消息
  MessageCreateParams params = MessageCreateParams.builder()
    .model(Model.CLAUDE_OPUS_5)
    .maxTokens(1024)
    .addUserMessageOfBlockParams(
      List.of(
        ContentBlockParam.ofDocument(documentParam),
        ContentBlockParam.ofText(
          TextBlockParam.builder()
            .text("What are the key findings in this document?")
            .build()
        )
      )
    )
    .build();

  Message message = client.messages().create(params);
  System.out.println(message.content());
  ```

  ```php PHP
  $client = new Client();

  // 首先，加载并编码 PDF
  $pdf_url = 'https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf';
  $pdf_data = base64_encode(file_get_contents($pdf_url));

  // 替代方案：从本地文件加载
  // $pdf_data = base64_encode(file_get_contents('document.pdf'));

  // 使用 base64 编码发送给 Claude
  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'document',
                      'source' => [
                          'type' => 'base64',
                          'media_type' => 'application/pdf',
                          'data' => $pdf_data,
                      ],
                  ],
                  [
                      'type' => 'text',
                      'text' => 'What are the key findings in this document?',
                  ],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo $message;
  ```

  ```ruby Ruby
  require "open-uri"

  # 首先，加载并编码 PDF
  pdf_url = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
  pdf_bytes = URI.open(pdf_url, "rb") { |f| f.read }
  pdf_data = [pdf_bytes].pack("m0") # Base64-encode without newlines

  # 备选方案：从本地文件加载
  # pdf_data = [File.binread("document.pdf")].pack("m0")

  # 使用 base64 编码发送给 Claude
  anthropic = Anthropic::Client.new
  message = anthropic.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "base64",
              media_type: "application/pdf",
              data: pdf_data
            }
          },
          {type: "text", text: "What are the key findings in this document?"}
        ]
      }
    ]
  )

  puts(message.content)
  ```
</CodeGroup>

#### 选项 3：Files API

对于您将重复使用的 PDF，或者当您希望避免编码开销时，请使用 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)：

<CodeGroup>
  ```bash cURL
  # 首先，将您的 PDF 上传到 Files API
  FILE_ID=$(curl -sS -X POST https://api.anthropic.com/v1/files \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -F "file=@document.pdf" | jq -r '.id')

  # 然后在您的消息中使用返回的 file_id
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d @- <<EOF
  {
    "model": "claude-opus-5",
    "max_tokens": 1024,
    "messages": [{
      "role": "user",
      "content": [{
        "type": "document",
        "source": {
          "type": "file",
          "file_id": "$FILE_ID"
        }
      },
      {
        "type": "text",
        "text": "What are the key findings in this document?"
      }]
    }]
  }
  EOF
  ```

  ```bash CLI
  # 首先，将您的 PDF 上传到 Files API
  FILE_ID=$(ant files upload \
    --file ./document.pdf \
    --transform id \
    --raw-output)

  # 然后在您的消息中使用返回的 file_id
  ant messages create \
    --transform content \
    --format yaml <<YAML
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: document
          source:
            type: file
            file_id: $FILE_ID
        - type: text
          text: What are the key findings in this document?
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  # 上传 PDF 文件
  with open("/path/to/document.pdf", "rb") as f:
      file_upload = client.files.upload(file=("document.pdf", f, "application/pdf"))

  # 在消息中使用已上传的文件
  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "document",
                      "source": {"type": "file", "file_id": file_upload.id},
                  },
                  {"type": "text", "text": "What are the key findings in this document?"},
              ],
          }
      ],
  )

  print(message.content)
  ```

  ```typescript TypeScript
  import Anthropic, { toFile } from "@anthropic-ai/sdk";
  import fs from "node:fs";

  const anthropic = new Anthropic();

  // 上传 PDF 文件
  const fileUpload = await anthropic.files.upload({
    file: await toFile(fs.createReadStream("/path/to/document.pdf"), undefined, {
      type: "application/pdf"
    })
  });

  // 在消息中使用已上传的文件
  const response = await anthropic.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "file",
              file_id: fileUpload.id
            }
          },
          {
            type: "text",
            text: "What are the key findings in this document?"
          }
        ]
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  var client = new AnthropicClient();

  // 上传 PDF 文件
  var fileUpload = await client.Files.Upload(new FileUploadParams
  {
      File = new BinaryContent
      {
          Stream = File.OpenRead("/path/to/document.pdf"),
          FileName = "document.pdf",
          ContentType = new("application/pdf"),
      },
  });

  // 在消息中使用已上传的文件
  var message = await client.Messages.Create(new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = new List<ContentBlockParam>
              {
                  new DocumentBlockParam
                  {
                      Source = new FileDocumentSource { FileID = fileUpload.ID },
                  },
                  new TextBlockParam("What are the key findings in this document?"),
              },
          },
      ],
  });

  Console.WriteLine(string.Join("\n", message.Content));
  ```

  ```go Go
  client := anthropic.NewClient()

  // 上传 PDF 文件
  pdfFile, err := os.Open("/path/to/document.pdf")
  if err != nil {
  	panic(err)
  }
  defer pdfFile.Close()

  fileUpload, err := client.Files.Upload(context.TODO(), anthropic.FileUploadParams{
  	File: anthropic.File(pdfFile, "document.pdf", "application/pdf"),
  })
  if err != nil {
  	panic(err)
  }

  // 在消息中使用已上传的文件
  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.NewDocumentBlock(anthropic.FileDocumentSourceParam{
  				FileID: fileUpload.ID,
  			}),
  			anthropic.NewTextBlock("What are the key findings in this document?"),
  		),
  	},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Printf("%+v\n", message.Content)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  // 上传 PDF 文件
  FileMetadata file = client
    .files()
    .upload(FileUploadParams.builder().file(Path.of("/path/to/document.pdf")).build());

  // 在消息中使用已上传的文件
  MessageCreateParams params = MessageCreateParams.builder()
    .model(Model.CLAUDE_OPUS_5)
    .maxTokens(1024)
    .addUserMessageOfBlockParams(
      List.of(
        ContentBlockParam.ofDocument(
          DocumentBlockParam.builder().fileSource(file.id()).build()
        ),
        ContentBlockParam.ofText(
          TextBlockParam.builder()
            .text("What are the key findings in this document?")
            .build()
        )
      )
    )
    .build();

  Message message = client.messages().create(params);
  System.out.println(message.content());
  ```

  ```php PHP
  use Anthropic\Core\FileParam;

  $client = new Client();

  // 上传 PDF 文件
  $file_upload = $client->files->upload(
      file: FileParam::fromResource(fopen('/path/to/document.pdf', 'r'), contentType: 'application/pdf'),
  );

  // 在消息中使用已上传的文件
  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'document',
                      'source' => [
                          'type' => 'file',
                          'fileID' => $file_upload->id,
                      ],
                  ],
                  [
                      'type' => 'text',
                      'text' => 'What are the key findings in this document?',
                  ],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo $message;
  ```

  ```ruby Ruby
  anthropic = Anthropic::Client.new

  # 上传 PDF 文件
  file_upload = File.open("/path/to/document.pdf", "rb") do |f|
    anthropic.files.upload(
      file: Anthropic::FilePart.new(f, filename: "document.pdf", content_type: "application/pdf")
    )
  end

  # 在消息中使用已上传的文件
  message = anthropic.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {type: "file", file_id: file_upload.id}
          },
          {type: "text", text: "What are the key findings in this document?"}
        ]
      }
    ]
  )

  puts(message.content)
  ```
</CodeGroup>

### PDF 支持的工作原理

当您向 Claude 发送 PDF 时，会发生以下步骤：

<Steps>
  <Step title="系统提取文档的内容。">
    * 系统将文档的每一页转换为图像。
    * 每一页的文本会被提取出来，并与每一页的图像一起提供。
  </Step>

  <Step title="Claude 同时分析文本和图像，以更好地理解文档。">
    * 文档以文本和图像组合的形式提供以供分析。
    * 这使用户能够就 PDF 的视觉元素（如图表、示意图和其他非文本内容）寻求见解。
  </Step>

  <Step title="Claude 作出响应，并在相关时引用 PDF 的内容。">
    Claude 在响应时可以同时引用文本和视觉内容。您可以通过将 PDF 支持与以下功能集成来进一步提升性能：

    * [使用提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support#use-prompt-caching)：提升重复分析的性能。
    * [处理文档批次](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support#process-document-batches)：用于大批量文档处理。
    * [工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)：从文档中提取特定信息以用作工具输入。
  </Step>
</Steps>

### 估算您的成本

PDF 文件的令牌数取决于从文档中提取的文本总量和页数：

* 文本令牌成本：根据内容密度，每页通常使用 1,500–3,000 个令牌。适用标准 API 定价，无额外 PDF 费用。
* 图像令牌成本：由于每一页都会被转换为图像，因此适用相同的[基于图像的成本计算](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#evaluate-image-size)。

您可以使用[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)来估算您特定 PDF 的成本。

## 优化 PDF 处理

### 提升性能

遵循以下最佳实践以获得最佳结果：

* 在请求中将 PDF 放在文本之前
* 使用标准字体
* 确保文本清晰易读
* 将页面旋转至正确的竖直方向
* 在提示中使用逻辑页码（来自 PDF 查看器）
* 必要时将大型 PDF 拆分为多个部分
* 为重复分析启用提示缓存

### 扩展您的实现

对于大批量处理，请考虑以下方法：

#### 使用提示缓存

使用 "prompt caching"（提示缓存）缓存 PDF，以提升重复查询的性能，详见[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)：

<CodeGroup>
  ```bash cURL
  curl -sL "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf" | base64 | tr -d '\n' > pdf_base64.txt
  # 使用 pdf_base64.txt 的内容创建 JSON 请求文件
  jq -n --rawfile PDF_BASE64 pdf_base64.txt '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{
          "role": "user",
          "content": [{
              "type": "document",
              "source": {
                  "type": "base64",
                  "media_type": "application/pdf",
                  "data": $PDF_BASE64
              },
              "cache_control": {
                  "type": "ephemeral"
              }
          },
          {
              "type": "text",
              "text": "Which model has the highest human preference win rates across each use-case?"
          }]
      }]
  }' > request.json

  # 然后使用该 JSON 文件发起 API 调用
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d @request.json
  ```

  ```bash CLI
  ant messages create --transform content --format yaml <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: document
          source:
            type: base64
            media_type: application/pdf
            data: "@./document.pdf"
          cache_control:
            type: ephemeral
        - type: text
          text: Which model has the highest human preference win rates across each use-case?
  YAML
  ```

  ```python Python
  import base64
  import httpx2

  # 首先，加载并编码 PDF
  pdf_url = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
  pdf_data = base64.standard_b64encode(
      httpx2.get(pdf_url, follow_redirects=True).content
  ).decode("utf-8")

  # 使用已缓存的文档创建消息
  client = anthropic.Anthropic()
  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "document",
                      "source": {
                          "type": "base64",
                          "media_type": "application/pdf",
                          "data": pdf_data,
                      },
                      "cache_control": {"type": "ephemeral"},
                  },
                  {
                      "type": "text",
                      "text": "Which model has the highest human preference win rates across each use-case?",
                  },
              ],
          }
      ],
  )

  print(message.content)
  ```

  ```typescript TypeScript
  // 首先，加载并编码 PDF
  const pdfURL =
    "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf";
  const pdfResponse = await fetch(pdfURL);
  const arrayBuffer = await pdfResponse.arrayBuffer();
  const pdfBase64 = Buffer.from(arrayBuffer).toString("base64");

  // 使用已缓存的文档创建消息
  const anthropic = new Anthropic();
  const response = await anthropic.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "base64",
              media_type: "application/pdf",
              data: pdfBase64
            },
            cache_control: { type: "ephemeral" }
          },
          {
            type: "text",
            text: "Which model has the highest human preference win rates across each use-case?"
          }
        ]
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  var client = new AnthropicClient();

  // 下载并编码 PDF
  var pdfUrl = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf";
  using var httpClient = new HttpClient();
  var pdfBase64 = Convert.ToBase64String(await httpClient.GetByteArrayAsync(pdfUrl));

  var message = await client.Messages.Create(new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages =
      [
          new()
          {
              Role = Role.User,
              Content = new List<ContentBlockParam>
              {
                  new DocumentBlockParam
                  {
                      Source = new Base64PdfSource { Data = pdfBase64 },
                      CacheControl = new CacheControlEphemeral(),
                  },
                  new TextBlockParam("Which model has the highest human preference win rates across each use-case?"),
              },
          },
      ],
  });

  Console.WriteLine(message);
  ```

  ```go Go
  // 首先，加载并编码 PDF
  pdfURL := "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
  resp, err := http.Get(pdfURL)
  if err != nil {
  	panic(err)
  }
  defer resp.Body.Close()
  pdfBytes, err := io.ReadAll(resp.Body)
  if err != nil {
  	panic(err)
  }
  pdfBase64 := base64.StdEncoding.EncodeToString(pdfBytes)

  // 创建带有 cache_control 的文档块
  client := anthropic.NewClient()
  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.ContentBlockParamUnion{
  				OfDocument: &anthropic.DocumentBlockParam{
  					Source: anthropic.DocumentBlockParamSourceUnion{
  						OfBase64: &anthropic.Base64PDFSourceParam{
  							Data: pdfBase64,
  						},
  					},
  					CacheControl: anthropic.NewCacheControlEphemeralParam(),
  				},
  			},
  			anthropic.NewTextBlock("Which model has the highest human preference win rates across each use-case?"),
  		),
  	},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Printf("%+v\n", message.Content)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  // 下载并编码 PDF
  String pdfUrl =
    "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf";
  HttpClient httpClient = HttpClient.newBuilder().followRedirects(HttpClient.Redirect.NORMAL).build();
  HttpRequest request = HttpRequest.newBuilder().uri(URI.create(pdfUrl)).GET().build();

  HttpResponse<byte[]> response = httpClient.send(
    request,
    HttpResponse.BodyHandlers.ofByteArray()
  );
  String pdfBase64 = Base64.getEncoder().encodeToString(response.body());

  MessageCreateParams params = MessageCreateParams.builder()
    .model(Model.CLAUDE_OPUS_5)
    .maxTokens(1024)
    .addUserMessageOfBlockParams(
      List.of(
        ContentBlockParam.ofDocument(
          DocumentBlockParam.builder()
            .source(Base64PdfSource.builder().data(pdfBase64).build())
            .cacheControl(CacheControlEphemeral.builder().build())
            .build()
        ),
        ContentBlockParam.ofText(
          TextBlockParam.builder()
            .text(
              "Which model has the highest human preference win rates across each use-case?"
            )
            .build()
        )
      )
    )
    .build();

  Message message = client.messages().create(params);
  System.out.println(message);
  ```

  ```php PHP
  $client = new Client();

  // 加载并编码 PDF
  $pdf_url = 'https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf';
  $pdf_data = base64_encode(file_get_contents($pdf_url));

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'document',
                      'source' => [
                          'type' => 'base64',
                          'media_type' => 'application/pdf',
                          'data' => $pdf_data,
                      ],
                      'cache_control' => ['type' => 'ephemeral'],
                  ],
                  [
                      'type' => 'text',
                      'text' => 'Which model has the highest human preference win rates across each use-case?',
                  ],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo $message;
  ```

  ```ruby Ruby
  require "open-uri"

  # 加载并编码 PDF
  pdf_url = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
  pdf_bytes = URI.open(pdf_url, "rb") { |f| f.read }
  pdf_data = [pdf_bytes].pack("m0") # Base64-encode without newlines

  anthropic = Anthropic::Client.new

  message = anthropic.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "base64",
              media_type: "application/pdf",
              data: pdf_data
            },
            cache_control: {type: "ephemeral"}
          },
          {
            type: "text",
            text: "Which model has the highest human preference win rates across each use-case?"
          }
        ]
      }
    ]
  )

  puts(message.content)
  ```
</CodeGroup>

#### 处理文档批次

使用 [Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing) 在一个请求中处理多个 PDF：

<CodeGroup>
  ```bash cURL
  curl -sL "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf" | base64 | tr -d '\n' > pdf_base64.txt
  # 使用 pdf_base64.txt 的内容创建 JSON 请求文件
  jq -n --rawfile PDF_BASE64 pdf_base64.txt '{
      "requests": [
      {
          "custom_id": "my-first-request",
          "params": {
              "model": "claude-opus-5",
              "max_tokens": 1024,
              "messages": [{
                  "role": "user",
                  "content": [{
                      "type": "document",
                      "source": {
                          "type": "base64",
                          "media_type": "application/pdf",
                          "data": $PDF_BASE64
                      }
                  },
                  {
                      "type": "text",
                      "text": "Which model has the highest human preference win rates across each use-case?"
                  }]
              }]
          }
      },
      {
          "custom_id": "my-second-request",
          "params": {
              "model": "claude-opus-5",
              "max_tokens": 1024,
              "messages": [{
                  "role": "user",
                  "content": [{
                      "type": "document",
                      "source": {
                          "type": "base64",
                          "media_type": "application/pdf",
                          "data": $PDF_BASE64
                      }
                  },
                  {
                      "type": "text",
                      "text": "Extract 5 key insights from this document."
                  }]
              }]
          }
      }]
  }' > request.json

  # 然后使用该 JSON 文件发起 API 调用
  curl https://api.anthropic.com/v1/messages/batches \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d @request.json
  ```

  ```bash CLI
  ant messages:batches create <<'YAML'
  requests:
    - custom_id: my-first-request
      params:
        model: claude-opus-5
        max_tokens: 1024
        messages:
          - role: user
            content:
              - type: document
                source:
                  type: base64
                  media_type: application/pdf
                  data: "@./document.pdf"
              - type: text
                text: >-
                  Which model has the highest human preference win rates
                  across each use-case?
    - custom_id: my-second-request
      params:
        model: claude-opus-5
        max_tokens: 1024
        messages:
          - role: user
            content:
              - type: document
                source:
                  type: base64
                  media_type: application/pdf
                  data: "@./document.pdf"
              - type: text
                text: Extract 5 key insights from this document.
  YAML
  ```

  ```python Python
  import base64
  import httpx2

  # 首先，加载并编码 PDF
  pdf_url = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
  pdf_data = base64.standard_b64encode(
      httpx2.get(pdf_url, follow_redirects=True).content
  ).decode("utf-8")

  # 创建一批使用该文档的请求
  client = anthropic.Anthropic()
  message_batch = client.messages.batches.create(
      requests=[
          {
              "custom_id": "my-first-request",
              "params": {
                  "model": "claude-opus-5",
                  "max_tokens": 1024,
                  "messages": [
                      {
                          "role": "user",
                          "content": [
                              {
                                  "type": "document",
                                  "source": {
                                      "type": "base64",
                                      "media_type": "application/pdf",
                                      "data": pdf_data,
                                  },
                              },
                              {
                                  "type": "text",
                                  "text": "Which model has the highest human preference win rates across each use-case?",
                              },
                          ],
                      }
                  ],
              },
          },
          {
              "custom_id": "my-second-request",
              "params": {
                  "model": "claude-opus-5",
                  "max_tokens": 1024,
                  "messages": [
                      {
                          "role": "user",
                          "content": [
                              {
                                  "type": "document",
                                  "source": {
                                      "type": "base64",
                                      "media_type": "application/pdf",
                                      "data": pdf_data,
                                  },
                              },
                              {
                                  "type": "text",
                                  "text": "Extract 5 key insights from this document.",
                              },
                          ],
                      }
                  ],
              },
          },
      ]
  )

  print(message_batch)
  ```

  ```typescript TypeScript
  // 首先，加载并编码 PDF
  const pdfURL =
    "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf";
  const pdfResponse = await fetch(pdfURL);
  const arrayBuffer = await pdfResponse.arrayBuffer();
  const pdfBase64 = Buffer.from(arrayBuffer).toString("base64");

  // 创建一批使用该文档的请求
  const anthropic = new Anthropic();
  const response = await anthropic.messages.batches.create({
    requests: [
      {
        custom_id: "my-first-request",
        params: {
          model: "claude-opus-5",
          max_tokens: 1024,
          messages: [
            {
              role: "user",
              content: [
                {
                  type: "document",
                  source: {
                    type: "base64",
                    media_type: "application/pdf",
                    data: pdfBase64
                  }
                },
                {
                  type: "text",
                  text: "Which model has the highest human preference win rates across each use-case?"
                }
              ]
            }
          ]
        }
      },
      {
        custom_id: "my-second-request",
        params: {
          model: "claude-opus-5",
          max_tokens: 1024,
          messages: [
            {
              role: "user",
              content: [
                {
                  type: "document",
                  source: {
                    type: "base64",
                    media_type: "application/pdf",
                    data: pdfBase64
                  }
                },
                {
                  type: "text",
                  text: "Extract 5 key insights from this document."
                }
              ]
            }
          ]
        }
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  var client = new AnthropicClient();

  // 下载并编码 PDF
  var pdfUrl = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf";
  using var httpClient = new HttpClient();
  var pdfBase64 = Convert.ToBase64String(await httpClient.GetByteArrayAsync(pdfUrl));

  var batch = await client.Messages.Batches.Create(new BatchCreateParams
  {
      Requests =
      [
          new()
          {
              CustomID = "my-first-request",
              Params = new()
              {
                  Model = Model.ClaudeOpus5,
                  MaxTokens = 1024,
                  Messages =
                  [
                      new()
                      {
                          Role = Role.User,
                          Content = new List<ContentBlockParam>
                          {
                              new DocumentBlockParam
                              {
                                  Source = new Base64PdfSource { Data = pdfBase64 },
                              },
                              new TextBlockParam("Which model has the highest human preference win rates across each use-case?"),
                          },
                      },
                  ],
              },
          },
          new()
          {
              CustomID = "my-second-request",
              Params = new()
              {
                  Model = Model.ClaudeOpus5,
                  MaxTokens = 1024,
                  Messages =
                  [
                      new()
                      {
                          Role = Role.User,
                          Content = new List<ContentBlockParam>
                          {
                              new DocumentBlockParam
                              {
                                  Source = new Base64PdfSource { Data = pdfBase64 },
                              },
                              new TextBlockParam("Extract 5 key insights from this document."),
                          },
                      },
                  ],
              },
          },
      ],
  });

  Console.WriteLine(batch);
  ```

  ```go Go
  // 首先，加载并编码 PDF
  pdfURL := "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
  resp, err := http.Get(pdfURL)
  if err != nil {
  	panic(err)
  }
  defer resp.Body.Close()
  pdfBytes, err := io.ReadAll(resp.Body)
  if err != nil {
  	panic(err)
  }
  pdfBase64 := base64.StdEncoding.EncodeToString(pdfBytes)

  // 创建一批使用该文档的请求
  client := anthropic.NewClient()
  batch, err := client.Messages.Batches.New(context.TODO(), anthropic.MessageBatchNewParams{
  	Requests: []anthropic.MessageBatchNewParamsRequest{
  		{
  			CustomID: "my-first-request",
  			Params: anthropic.MessageBatchNewParamsRequestParams{
  				Model:     anthropic.ModelClaudeOpus5,
  				MaxTokens: 1024,
  				Messages: []anthropic.MessageParam{
  					anthropic.NewUserMessage(
  						anthropic.NewDocumentBlock(anthropic.Base64PDFSourceParam{
  							Data: pdfBase64,
  						}),
  						anthropic.NewTextBlock("Which model has the highest human preference win rates across each use-case?"),
  					),
  				},
  			},
  		},
  		{
  			CustomID: "my-second-request",
  			Params: anthropic.MessageBatchNewParamsRequestParams{
  				Model:     anthropic.ModelClaudeOpus5,
  				MaxTokens: 1024,
  				Messages: []anthropic.MessageParam{
  					anthropic.NewUserMessage(
  						anthropic.NewDocumentBlock(anthropic.Base64PDFSourceParam{
  							Data: pdfBase64,
  						}),
  						anthropic.NewTextBlock("Extract 5 key insights from this document."),
  					),
  				},
  			},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Printf("%+v\n", batch)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  // 下载并编码 PDF
  String pdfUrl =
    "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf";
  HttpClient httpClient = HttpClient.newBuilder().followRedirects(HttpClient.Redirect.NORMAL).build();
  HttpRequest request = HttpRequest.newBuilder().uri(URI.create(pdfUrl)).GET().build();

  HttpResponse<byte[]> response = httpClient.send(
    request,
    HttpResponse.BodyHandlers.ofByteArray()
  );
  String pdfBase64 = Base64.getEncoder().encodeToString(response.body());

  BatchCreateParams params = BatchCreateParams.builder()
    .addRequest(
      BatchCreateParams.Request.builder()
        .customId("my-first-request")
        .params(
          BatchCreateParams.Request.Params.builder()
            .model(Model.CLAUDE_OPUS_5)
            .maxTokens(1024)
            .addUserMessageOfBlockParams(
              List.of(
                ContentBlockParam.ofDocument(
                  DocumentBlockParam.builder()
                    .source(Base64PdfSource.builder().data(pdfBase64).build())
                    .build()
                ),
                ContentBlockParam.ofText(
                  TextBlockParam.builder()
                    .text(
                      "Which model has the highest human preference win rates across each use-case?"
                    )
                    .build()
                )
              )
            )
            .build()
        )
        .build()
    )
    .addRequest(
      BatchCreateParams.Request.builder()
        .customId("my-second-request")
        .params(
          BatchCreateParams.Request.Params.builder()
            .model(Model.CLAUDE_OPUS_5)
            .maxTokens(1024)
            .addUserMessageOfBlockParams(
              List.of(
                ContentBlockParam.ofDocument(
                  DocumentBlockParam.builder()
                    .source(Base64PdfSource.builder().data(pdfBase64).build())
                    .build()
                ),
                ContentBlockParam.ofText(
                  TextBlockParam.builder()
                    .text("Extract 5 key insights from this document.")
                    .build()
                )
              )
            )
            .build()
        )
        .build()
    )
    .build();

  MessageBatch batch = client.messages().batches().create(params);
  System.out.println(batch);
  ```

  ```php PHP
  $client = new Client();

  // 加载并编码 PDF
  $pdf_url = 'https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf';
  $pdf_data = base64_encode(file_get_contents($pdf_url));

  $batch = $client->messages->batches->create(
      requests: [
          [
              'custom_id' => 'my-first-request',
              'params' => [
                  'model' => 'claude-opus-5',
                  'max_tokens' => 1024,
                  'messages' => [
                      [
                          'role' => 'user',
                          'content' => [
                              [
                                  'type' => 'document',
                                  'source' => [
                                      'type' => 'base64',
                                      'media_type' => 'application/pdf',
                                      'data' => $pdf_data,
                                  ],
                              ],
                              [
                                  'type' => 'text',
                                  'text' => 'Which model has the highest human preference win rates across each use-case?',
                              ],
                          ],
                      ],
                  ],
              ],
          ],
          [
              'custom_id' => 'my-second-request',
              'params' => [
                  'model' => 'claude-opus-5',
                  'max_tokens' => 1024,
                  'messages' => [
                      [
                          'role' => 'user',
                          'content' => [
                              [
                                  'type' => 'document',
                                  'source' => [
                                      'type' => 'base64',
                                      'media_type' => 'application/pdf',
                                      'data' => $pdf_data,
                                  ],
                              ],
                              [
                                  'type' => 'text',
                                  'text' => 'Extract 5 key insights from this document.',
                              ],
                          ],
                      ],
                  ],
              ],
          ],
      ],
  );

  echo $batch;
  ```

  ```ruby Ruby
  require "open-uri"

  # 加载并编码 PDF
  pdf_url = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
  pdf_bytes = URI.open(pdf_url, "rb") { |f| f.read }
  pdf_data = [pdf_bytes].pack("m0") # Base64-encode without newlines

  anthropic = Anthropic::Client.new

  message_batch = anthropic.messages.batches.create(
    requests: [
      {
        custom_id: "my-first-request",
        params: {
          model: "claude-opus-5",
          max_tokens: 1024,
          messages: [
            {
              role: "user",
              content: [
                {
                  type: "document",
                  source: {
                    type: "base64",
                    media_type: "application/pdf",
                    data: pdf_data
                  }
                },
                {
                  type: "text",
                  text: "Which model has the highest human preference win rates across each use-case?"
                }
              ]
            }
          ]
        }
      },
      {
        custom_id: "my-second-request",
        params: {
          model: "claude-opus-5",
          max_tokens: 1024,
          messages: [
            {
              role: "user",
              content: [
                {
                  type: "document",
                  source: {
                    type: "base64",
                    media_type: "application/pdf",
                    data: pdf_data
                  }
                },
                {
                  type: "text",
                  text: "Extract 5 key insights from this document."
                }
              ]
            }
          ]
        }
      }
    ]
  )

  puts(message_batch)
  ```
</CodeGroup>

批次以异步方式处理。要检查进度并在处理结束后获取结果，请参阅[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="视觉" icon="image" href="https://platform.claude.com/docs/zh-CN/build-with-claude/vision">
    Claude 的视觉能力使其能够理解和分析图像，为多模态交互开辟了令人兴奋的可能性。
  </Card>

  <Card title="尝试 PDF 示例" icon="file" href="https://platform.claude.com/cookbook/multimodal-getting-started-with-vision">
    在 Claude Cookbook 示例中探索 PDF 处理的实际示例。
  </Card>

  <Card title="查看 API 参考" icon="code" href="https://platform.claude.com/docs/zh-CN/api/messages/create">
    查看 PDF 支持的完整 API 文档。
  </Card>
</CardGroup>
