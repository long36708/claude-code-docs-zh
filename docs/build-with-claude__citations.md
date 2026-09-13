---
title: 引用
url: https://platform.claude.com/docs/zh-CN/build-with-claude/citations
description: 让 Claude 的回答以您的源文档为依据。引用功能会返回支持每项论断的确切段落，以便您验证答案并向用户展示来源。
---

## Compatibility
- [ZDR](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): eligible (excludes [Covered Models](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#model-specific-data-retention-requirements))
- Platforms: Claude API, Claude Platform on AWS, Amazon Bedrock, Google Cloud, Microsoft Foundry

Claude 在回答有关文档的问题时可以提供详细的 "citations"（引用），帮助您追踪和验证每个回答背后的来源。

所有[活跃模型](https://platform.claude.com/docs/zh-CN/models/overview)均支持引用功能。

<Tip>
  请使用[引用功能反馈表单](https://forms.gle/9n9hSrKnKe3rpowH9)分享您对引用功能的反馈和建议。
</Tip>

以下示例展示了如何通过 Messages API 在纯文本文档上启用引用：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [
        {
          "role": "user",
          "content": [
            {
              "type": "document",
              "source": {
                "type": "text",
                "media_type": "text/plain",
                "data": "The grass is green. The sky is blue."
              },
              "title": "My Document",
              "context": "This is a trustworthy document.",
              "citations": {"enabled": true}
            },
            {
              "type": "text",
              "text": "What color is the grass and sky?"
            }
          ]
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: document
          source:
            type: text
            media_type: text/plain
            data: The grass is green. The sky is blue.
          title: My Document
          context: This is a trustworthy document.
          citations:
            enabled: true
        - type: text
          text: What color is the grass and sky?
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "document",
                      "source": {
                          "type": "text",
                          "media_type": "text/plain",
                          "data": "The grass is green. The sky is blue.",
                      },
                      "title": "My Document",
                      "context": "This is a trustworthy document.",
                      "citations": {"enabled": True},
                  },
                  {"type": "text", "text": "What color is the grass and sky?"},
              ],
          }
      ],
  )
  print(response)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "text",
              media_type: "text/plain",
              data: "The grass is green. The sky is blue."
            },
            title: "My Document",
            context: "This is a trustworthy document.",
            citations: { enabled: true }
          },
          {
            type: "text",
            text: "What color is the grass and sky?"
          }
        ]
      }
    ]
  });
  console.log(response);
  ```

  ```csharp C#
  var client = new AnthropicClient();

  var response = await client.Messages.Create(
      new()
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 1024,
          Messages =
          [
              new()
              {
                  Role = Role.User,
                  Content = new MessageParamContent(new List<ContentBlockParam>
                  {
                      new ContentBlockParam(new DocumentBlockParam(
                          new DocumentBlockParamSource(new PlainTextSource()
                          {
                              Data = "The grass is green. The sky is blue.",
                          })
                      )
                      {
                          Title = "My Document",
                          Context = "This is a trustworthy document.",
                          Citations = new CitationsConfigParam { Enabled = true },
                      }),
                      new ContentBlockParam(new TextBlockParam("What color is the grass and sky?")),
                  }),
              },
          ],
      }
  );

  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.ContentBlockParamUnion{
  				OfDocument: &anthropic.DocumentBlockParam{
  					Source: anthropic.DocumentBlockParamSourceUnion{
  						OfText: &anthropic.PlainTextSourceParam{
  							Data: "The grass is green. The sky is blue.",
  						},
  					},
  					Title:     anthropic.String("My Document"),
  					Context:   anthropic.String("This is a trustworthy document."),
  					Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
  				},
  			},
  			anthropic.NewTextBlock("What color is the grass and sky?"),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  PlainTextSource source = PlainTextSource.builder()
      .data("The grass is green. The sky is blue.")
      .build();

  DocumentBlockParam documentParam = DocumentBlockParam.builder()
      .source(source)
      .title("My Document")
      .context("This is a trustworthy document.")
      .citations(CitationsConfigParam.builder().enabled(true).build())
      .build();

  TextBlockParam textBlockParam = TextBlockParam.builder()
      .text("What color is the grass and sky?")
      .build();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(1024)
      .addUserMessageOfBlockParams(
          List.of(
              ContentBlockParam.ofDocument(documentParam),
              ContentBlockParam.ofText(textBlockParam)
          )
      )
      .build();

  Message message = client.messages().create(params);
  System.out.println(message);
  ```

  ```php PHP
  $client = new Client();

  $response = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'document',
                      'source' => [
                          'type' => 'text',
                          'media_type' => 'text/plain',
                          'data' => 'The grass is green. The sky is blue.',
                      ],
                      'title' => 'My Document',
                      'context' => 'This is a trustworthy document.',
                      'citations' => ['enabled' => true],
                  ],
                  [
                      'type' => 'text',
                      'text' => 'What color is the grass and sky?',
                  ],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo json_encode($response, JSON_PRETTY_PRINT);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "text",
              media_type: "text/plain",
              data: "The grass is green. The sky is blue."
            },
            title: "My Document",
            context: "This is a trustworthy document.",
            citations: { enabled: true }
          },
          {
            type: "text",
            text: "What color is the grass and sky?"
          }
        ]
      }
    ]
  )

  puts response
  ```
</CodeGroup>

<Tip>
  **与基于提示的方法的比较**

  与通过提示让 Claude 引用来源相比，引用功能具有以下优势：

  * **节省成本：** 如果您基于提示的方法要求 Claude 输出直接引文，您可能会节省成本，因为 `cited_text` 不计入您的输出令牌。
  * **更高的引用可靠性：** 由于 API 会将引用解析为以下各节所述的响应格式并直接提取 `cited_text`，因此可以保证引用包含指向所提供文档的有效指针。
  * **更高的引用质量：** 在 Anthropic 的评估中，与纯粹基于提示的方法相比，引用功能引用文档中最相关引文的可能性显著更高。
</Tip>

***

## 引用的工作原理

按以下步骤将引用功能与 Claude 集成：

<Steps>
  <Step title="提供文档并启用引用">
    * 以任一受支持的格式包含文档：[PDF](https://platform.claude.com/docs/zh-CN/build-with-claude/citations#pdf-documents)、[纯文本](https://platform.claude.com/docs/zh-CN/build-with-claude/citations#plain-text-documents)或[自定义内容](https://platform.claude.com/docs/zh-CN/build-with-claude/citations#custom-content-documents)文档。
    * 在每个文档上设置 `citations.enabled=true`。目前，在一个请求中必须对所有文档全部启用或全部不启用引用。
    * 目前仅支持文本引用。尚不支持图像引用。
  </Step>

  <Step title="文档被处理">
    * 文档内容会被 "chunked"（分块），以定义可能引用的最小粒度。例如，句子分块允许 Claude 引用单个句子，或将多个连续句子串联起来以引用一个段落或更长的篇幅。

      * **对于 PDF：** 按照 [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)中所述提取文本，并将内容分块为句子。目前不支持引用 PDF 中的图像。
      * **对于纯文本文档：** 内容被分块为可供引用的句子。
      * **对于自定义内容文档：** 您提供的内容块按原样使用，不做进一步分块。
  </Step>

  <Step title="Claude 提供带引用的回答">
    * 响应现在可能包含多个文本块，其中每个文本块可以包含 Claude 提出的一项论断以及支持该论断的引用列表。

    * 引用指向源文档中的特定位置。这些引用的格式取决于被引用文档的类型。

      * **对于 PDF：** 引用包含页码范围（从 1 开始索引）。
      * **对于纯文本文档：** 引用包含字符索引范围（从 0 开始索引）。
      * **对于自定义内容文档：** 引用包含与所提供的原始内容列表相对应的内容块索引范围（从 0 开始索引）。

    * 提供文档索引以指示引用来源，该索引根据您原始请求中所有文档的列表从 0 开始编号。
  </Step>
</Steps>

<Tip>
  **自动分块与自定义内容**

  默认情况下，纯文本和 PDF 文档会自动分块为句子。如果您需要对引用粒度进行更多控制（例如，针对项目符号列表或转录文本），请改用自定义内容文档。有关更多详细信息，请参阅[文档类型](https://platform.claude.com/docs/zh-CN/build-with-claude/citations#document-types)。

  例如，如果您希望 Claude 能够引用 RAG 块中的特定句子，则应将每个 RAG 块放入一个纯文本文档中。否则，如果您不希望进行任何进一步的分块，或者希望自定义任何额外的分块，则可以将 RAG 块放入自定义内容文档中。
</Tip>

### 可引用与不可引用的内容

* 文档 `source` 内容中的文本可以被引用。
* `title` 和 `context` 是可选字段，会传递给模型，但不用于被引用的内容。
* `title` 有长度限制，因此 `context` 字段可用于以文本或字符串化 JSON 的形式存储文档元数据。

### 引用索引

* 文档索引根据请求中所有文档内容块的列表（跨越所有消息）从 0 开始编号。
* 字符索引从 0 开始，结束索引为不包含（exclusive）。
* 页码从 1 开始，结束页码为不包含。
* 内容块索引根据自定义内容文档中提供的 `content` 列表从 0 开始编号，结束索引为不包含。

### 令牌成本

* 由于系统提示的附加内容和文档分块，启用引用会导致输入令牌略有增加。
* 然而，引用功能在输出令牌方面非常高效。在内部，模型以标准化格式输出引用，然后将其解析为被引用文本和文档位置索引。`cited_text` 字段是为方便起见而提供的，不计入输出令牌。
* 在后续对话轮次中传回时，`cited_text` 也不计入输入令牌。

### 功能兼容性

引用功能可与其他 API 功能配合使用，包括 "prompt caching"（提示缓存）、[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)和[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)。有关提示缓存的详细信息，请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)。

<Warning>
  **引用与结构化输出不兼容**

  引用不能与[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)一起使用。如果您在任何用户提供的文档（`document` 块或 `search_result` 块）上启用了引用，同时又包含了 `output_config.format` 参数（或已弃用的 `output_format` 参数），API 将返回 400 错误。

  这是因为引用需要将引用块与文本输出交错排列，这与结构化输出的严格 JSON schema 约束不兼容。
</Warning>

#### 将提示缓存与引用结合使用

引用和提示缓存可以有效地结合使用。

响应中生成的引用块无法直接缓存，但它们所引用的源文档可以被缓存。为了优化性能，请将 `cache_control` 应用于您的顶层文档内容块。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [
        {
          "role": "user",
          "content": [
            {
              "type": "document",
              "source": {
                "type": "text",
                "media_type": "text/plain",
                "data": "This is a very long document with thousands of words..."
              },
              "citations": {"enabled": true},
              "cache_control": {"type": "ephemeral"}
            },
            {
              "type": "text",
              "text": "What does this document say about API features?"
            }
          ]
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages create --model claude-opus-5 --max-tokens 1024 <<'YAML'
  messages:
    - role: user
      content:
        - type: document
          source:
            type: text
            media_type: text/plain
            data: This is a very long document with thousands of words...
          citations:
            enabled: true
          cache_control:
            type: ephemeral
        - type: text
          text: What does this document say about API features?
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  # 长文档内容（例如技术文档）
  long_document = (
      "This is a very long document with thousands of words..." + " ... " * 1000
  )  # Minimum cacheable length

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "document",
                      "source": {
                          "type": "text",
                          "media_type": "text/plain",
                          "data": long_document,
                      },
                      "citations": {"enabled": True},
                      "cache_control": {
                          "type": "ephemeral"
                      },  # Cache the document content
                  },
                  {
                      "type": "text",
                      "text": "What does this document say about API features?",
                  },
              ],
          }
      ],
  )
  print(response)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // 长文档内容（例如技术文档）
  const longDocument =
    "This is a very long document with thousands of words..." + " ... ".repeat(1000); // Minimum cacheable length

  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "text",
              media_type: "text/plain",
              data: longDocument
            },
            citations: { enabled: true },
            cache_control: { type: "ephemeral" } // Cache the document content
          },
          {
            type: "text",
            text: "What does this document say about API features?"
          }
        ]
      }
    ]
  });
  console.log(response);
  ```

  ```csharp C#
  var client = new AnthropicClient();

  // 长文档内容（例如技术文档）
  var longDocument =
      "This is a very long document with thousands of words..."
      + string.Concat(Enumerable.Repeat(" ... ", 1000)); // Minimum cacheable length

  var response = await client.Messages.Create(
      new()
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 1024,
          Messages =
          [
              new()
              {
                  Role = Role.User,
                  Content = new MessageParamContent(new List<ContentBlockParam>
                  {
                      new ContentBlockParam(new DocumentBlockParam(
                          new DocumentBlockParamSource(new PlainTextSource() { Data = longDocument })
                      )
                      {
                          Citations = new CitationsConfigParam { Enabled = true },
                          CacheControl = new CacheControlEphemeral(), // Cache the document content
                      }),
                      new ContentBlockParam(new TextBlockParam("What does this document say about API features?")),
                  }),
              },
          ],
      }
  );

  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  // 长文档内容（例如技术文档）
  longDocument := "This is a very long document with thousands of words..." +
  	strings.Repeat(" ... ", 1000) // Minimum cacheable length

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.ContentBlockParamUnion{
  				OfDocument: &anthropic.DocumentBlockParam{
  					Source: anthropic.DocumentBlockParamSourceUnion{
  						OfText: &anthropic.PlainTextSourceParam{Data: longDocument},
  					},
  					Citations:    anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
  					CacheControl: anthropic.NewCacheControlEphemeralParam(), // Cache the document content
  				},
  			},
  			anthropic.NewTextBlock("What does this document say about API features?"),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  // 长文档内容（例如技术文档）
  String longDocument =
      "This is a very long document with thousands of words..."
          + " ... ".repeat(1000); // Minimum cacheable length

  DocumentBlockParam documentParam = DocumentBlockParam.builder()
      .source(PlainTextSource.builder().data(longDocument).build())
      .citations(CitationsConfigParam.builder().enabled(true).build())
      .cacheControl(CacheControlEphemeral.builder().build()) // Cache the document content
      .build();

  TextBlockParam textBlockParam = TextBlockParam.builder()
      .text("What does this document say about API features?")
      .build();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(1024)
      .addUserMessageOfBlockParams(
          List.of(
              ContentBlockParam.ofDocument(documentParam),
              ContentBlockParam.ofText(textBlockParam)
          )
      )
      .build();

  Message message = client.messages().create(params);
  System.out.println(message);
  ```

  ```php PHP
  $client = new Client();

  // 长文档内容（例如技术文档）
  $longDocument =
      'This is a very long document with thousands of words...'
      . str_repeat(' ... ', 1000); // Minimum cacheable length

  $response = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'document',
                      'source' => [
                          'type' => 'text',
                          'media_type' => 'text/plain',
                          'data' => $longDocument,
                      ],
                      'citations' => ['enabled' => true],
                      'cache_control' => ['type' => 'ephemeral'], // Cache the document content
                  ],
                  [
                      'type' => 'text',
                      'text' => 'What does this document say about API features?',
                  ],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo json_encode($response, JSON_PRETTY_PRINT);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 长文档内容（例如技术文档）
  long_document =
    "This is a very long document with thousands of words..." +
    " ... " * 1000 # Minimum cacheable length

  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "text",
              media_type: "text/plain",
              data: long_document
            },
            citations: { enabled: true },
            cache_control: { type: "ephemeral" } # Cache the document content
          },
          {
            type: "text",
            text: "What does this document say about API features?"
          }
        ]
      }
    ]
  )

  puts response
  ```
</CodeGroup>

在此示例中：

* 文档内容通过文档块上的 `cache_control` 进行缓存。
* 文档上启用了引用。
* Claude 可以在受益于已缓存文档内容的同时生成带引用的响应。
* 使用同一文档的后续请求将受益于已缓存的内容。

## 文档类型

### 选择文档类型

引用功能支持三种文档类型。文档可以直接在消息中提供（base64、文本或 URL），也可以通过 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传并通过 `file_id` 引用：

| 类型    | 最适用于                 | 分块方式  | 引用格式         |
| ----- | -------------------- | ----- | ------------ |
| 纯文本   | 简单文本文档、散文            | 句子    | 字符索引（从 0 开始） |
| PDF   | 包含文本内容的 PDF 文件       | 句子    | 页码（从 1 开始）   |
| 自定义内容 | 列表、转录文本、特殊格式、更细粒度的引用 | 无额外分块 | 块索引（从 0 开始）  |

<Note>
  对于 `document` 块不支持的文件类型（例如 .docx 和 .xlsx），请将文件转换为纯文本，并将内容直接包含在消息内容中。已经是纯文本的文件（例如 .csv 和 .md 文件）也可以使用显式的 `text/plain` 内容类型上传。请参阅[处理其他文件格式](https://platform.claude.com/docs/zh-CN/build-with-claude/files#working-with-other-file-formats)。
</Note>

### 纯文本文档

纯文本文档会自动分块为句子。您可以内联提供它们，也可以通过其 `file_id` 引用它们：

<Tabs>
  <Tab title="内联文本">
    本页顶部的入门示例展示了每种 SDK 中完整的纯文本请求。文档块使用 `text` 源：

    ```json
    {
      "type": "document",
      "source": {
        "type": "text",
        "media_type": "text/plain",
        "data": "Plain text content..."
      },
      "title": "Document Title",
      "context": "Context about the document that will not be cited from",
      "citations": { "enabled": true }
    }
    ```
  </Tab>

  <Tab title="Files API">
    这些示例将通过 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传的文件作为 `document` 源进行引用。

    <CodeGroup>
      ```bash cURL
      curl -X POST https://api.anthropic.com/v1/messages \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "content-type: application/json" \
        -d @- <<EOF
      {
        "model": "claude-opus-5",
        "max_tokens": 1024,
        "messages": [
          {
            "role": "user",
            "content": [
              {
                "type": "document",
                "source": {"type": "file", "file_id": "$FILE_ID"},
                "title": "Document Title",
                "context": "Context about the document that will not be cited from",
                "citations": {"enabled": true}
              },
              {
                "type": "text",
                "text": "Summarize this document."
              }
            ]
          }
        ]
      }
      EOF
      ```

      ```bash CLI
      ant messages create <<YAML
      model: claude-opus-5
      max_tokens: 1024
      messages:
        - role: user
          content:
            - type: document
              source:
                type: file
                file_id: $FILE_ID
              title: Document Title
              context: Context about the document that will not be cited from
              citations:
                enabled: true
            - type: text
              text: Summarize this document.
      YAML
      ```

      ```python Python
      cited_response = client.messages.create(
          model="claude-opus-5",
          max_tokens=1024,
          messages=[
              {
                  "role": "user",
                  "content": [
                      {
                          "type": "document",
                          "source": {"type": "file", "file_id": file_id},
                          "title": "Document Title",
                          "context": "Context about the document that will not be cited from",
                          "citations": {"enabled": True},
                      },
                      {"type": "text", "text": "Summarize this document."},
                  ],
              }
          ],
      )
      print(cited_response)
      ```

      ```typescript TypeScript
      const citedResponse = await client.messages.create({
        model: "claude-opus-5",
        max_tokens: 1024,
        messages: [
          {
            role: "user",
            content: [
              {
                type: "document",
                source: { type: "file", file_id: uploaded.id },
                title: "Document Title",
                context: "Context about the document that will not be cited from",
                citations: { enabled: true },
              },
              {
                type: "text",
                text: "Summarize this document.",
              },
            ],
          },
        ],
      });
      console.log(citedResponse);
      ```

      ```csharp C#
      var citedResponse = await client.Messages.Create(
          new MessageCreateParams
          {
              Model = Model.ClaudeOpus5,
              MaxTokens = 1024,
              Messages =
              [
                  new MessageParam
                  {
                      Role = Role.User,
                      Content = new List<ContentBlockParam>
                      {
                          new DocumentBlockParam
                          {
                              Source = new FileDocumentSource { FileID = fileId },
                              Title = "Document Title",
                              Context = "Context about the document that will not be cited from",
                              Citations = new CitationsConfigParam { Enabled = true },
                          },
                          new TextBlockParam { Text = "Summarize this document." },
                      }
                  }
              ]
          });

      Console.WriteLine(citedResponse);
      ```

      ```go Go
      citedMsg, err := client.Messages.New(context.Background(),
      	anthropic.MessageNewParams{
      		Model:     anthropic.ModelClaudeOpus5,
      		MaxTokens: 1024,
      		Messages: []anthropic.MessageParam{
      			anthropic.NewUserMessage(
      				anthropic.ContentBlockParamUnion{
      					OfDocument: &anthropic.DocumentBlockParam{
      						Source: anthropic.DocumentBlockParamSourceUnion{
      							OfFile: &anthropic.FileDocumentSourceParam{FileID: fileID},
      						},
      						Title:     anthropic.String("Document Title"),
      						Context:   anthropic.String("Context about the document that will not be cited from"),
      						Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
      					},
      				},
      				anthropic.NewTextBlock("Summarize this document."),
      			),
      		},
      	})
      if err != nil {
      	log.Fatal(err)
      }
      fmt.Println(citedMsg)
      ```

      ```java Java
      MessageCreateParams citedParams = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessageOfBlockParams(List.of(
              ContentBlockParam.ofDocument(DocumentBlockParam.builder()
                  .fileSource(fileId)
                  .title("Document Title")
                  .context("Context about the document that will not be cited from")
                  .citations(CitationsConfigParam.builder().enabled(true).build())
                  .build()),
              ContentBlockParam.ofText(TextBlockParam.builder()
                  .text("Summarize this document.")
                  .build())
          ))
          .build();

      Message citedMessage = client.messages().create(citedParams);
      System.out.println(citedMessage);
      ```

      ```php PHP
      $citedResponse = $client->messages->create(
          maxTokens: 1024,
          messages: [
              [
                  'role' => 'user',
                  'content' => [
                      [
                          'type' => 'document',
                          'source' => ['type' => 'file', 'fileID' => $fileId],
                          'title' => 'Document Title',
                          'context' => 'Context about the document that will not be cited from',
                          'citations' => ['enabled' => true],
                      ],
                      ['type' => 'text', 'text' => 'Summarize this document.'],
                  ],
              ],
          ],
          model: 'claude-opus-5',
      );

      echo $citedResponse;
      ```

      ```ruby Ruby
      cited_response = client.messages.create(
        model: "claude-opus-5",
        max_tokens: 1024,
        messages: [
          {
            role: "user",
            content: [
              {
                type: "document",
                source: { type: "file", file_id: file_id },
                title: "Document Title",
                context: "Context about the document that will not be cited from",
                citations: { enabled: true }
              },
              {
                type: "text",
                text: "Summarize this document."
              }
            ]
          }
        ]
      )

      puts cited_response
      ```
    </CodeGroup>
  </Tab>
</Tabs>

<Accordion title="纯文本引用示例">
  ```json
  {
    "type": "char_location",
    "cited_text": "The exact text being cited", // not counted toward output tokens
    "document_index": 0,
    "document_title": "Document Title",
    "start_char_index": 0, // 0-indexed
    "end_char_index": 50 // exclusive
  }
  ```
</Accordion>

### PDF 文档

PDF 文档可以以 base64 编码数据、URL 或 `file_id` 的形式提供。PDF 文本会被提取并分块为句子。由于尚不支持图像引用，因此属于文档扫描件且不包含可提取文本的 PDF 无法被引用。

<Tabs>
  <Tab title="Base64">
    <CodeGroup>
      ```bash cURL
      PDF_BASE64=$(base64 /path/to/document.pdf | tr -d '\n')

      curl https://api.anthropic.com/v1/messages \
        -H "content-type: application/json" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -d '{
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
                    "data": "'"$PDF_BASE64"'"
                  },
                  "title": "Document Title",
                  "context": "Context about the document that will not be cited from",
                  "citations": {"enabled": true}
                },
                {
                  "type": "text",
                  "text": "Summarize this document."
                }
              ]
            }
          ]
        }'
      ```

      ```bash CLI
      ant messages create <<'YAML'
      model: claude-opus-5
      max_tokens: 1024
      messages:
        - role: user
          content:
            - type: document
              source:
                type: base64
                media_type: application/pdf
                data: "@/path/to/document.pdf"
              title: Document Title
              context: Context about the document that will not be cited from
              citations:
                enabled: true
            - type: text
              text: Summarize this document.
      YAML
      ```

      ```python Python
      client = anthropic.Anthropic()

      pdf_base64 = base64.standard_b64encode(
          pathlib.Path("/path/to/document.pdf").read_bytes()
      ).decode()

      response = client.messages.create(
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
                              "data": pdf_base64,
                          },
                          "title": "Document Title",
                          "context": "Context about the document that will not be cited from",
                          "citations": {"enabled": True},
                      },
                      {"type": "text", "text": "Summarize this document."},
                  ],
              }
          ],
      )
      print(response)
      ```

      ```typescript TypeScript
      const client = new Anthropic();

      const pdfBase64 = Buffer.from(await readFile("/path/to/document.pdf")).toString("base64");

      const response = await client.messages.create({
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
                title: "Document Title",
                context: "Context about the document that will not be cited from",
                citations: { enabled: true }
              },
              {
                type: "text",
                text: "Summarize this document."
              }
            ]
          }
        ]
      });
      console.log(response);
      ```

      ```csharp C#
      var client = new AnthropicClient();

      var pdfBase64 = Convert.ToBase64String(await File.ReadAllBytesAsync("/path/to/document.pdf"));

      var response = await client.Messages.Create(
          new()
          {
              Model = Model.ClaudeOpus5,
              MaxTokens = 1024,
              Messages =
              [
                  new()
                  {
                      Role = Role.User,
                      Content = new MessageParamContent(new List<ContentBlockParam>
                      {
                          new ContentBlockParam(new DocumentBlockParam(
                              new DocumentBlockParamSource(new Base64PdfSource() { Data = pdfBase64 })
                          )
                          {
                              Title = "Document Title",
                              Context = "Context about the document that will not be cited from",
                              Citations = new CitationsConfigParam { Enabled = true },
                          }),
                          new ContentBlockParam(new TextBlockParam("Summarize this document.")),
                      }),
                  },
              ],
          }
      );

      Console.WriteLine(response);
      ```

      ```go Go
      client := anthropic.NewClient()

      pdfBytes, err := os.ReadFile("/path/to/document.pdf")
      if err != nil {
      	log.Fatal(err)
      }
      pdfBase64 := base64.StdEncoding.EncodeToString(pdfBytes)

      response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
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
      					Title:     anthropic.String("Document Title"),
      					Context:   anthropic.String("Context about the document that will not be cited from"),
      					Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
      				},
      			},
      			anthropic.NewTextBlock("Summarize this document."),
      		),
      	},
      })
      if err != nil {
      	log.Fatal(err)
      }
      fmt.Println(response)
      ```

      ```java Java
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      byte[] pdfBytes = Files.readAllBytes(Path.of("/path/to/document.pdf"));
      String pdfBase64 = Base64.getEncoder().encodeToString(pdfBytes);

      DocumentBlockParam documentParam = DocumentBlockParam.builder()
          .source(Base64PdfSource.builder().data(pdfBase64).build())
          .title("Document Title")
          .context("Context about the document that will not be cited from")
          .citations(CitationsConfigParam.builder().enabled(true).build())
          .build();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessageOfBlockParams(
              List.of(
                  ContentBlockParam.ofDocument(documentParam),
                  ContentBlockParam.ofText(TextBlockParam.builder().text("Summarize this document.").build())
              )
          )
          .build();

      Message message = client.messages().create(params);
      System.out.println(message);
      ```

      ```php PHP
      $client = new Client();

      $pdfBase64 = base64_encode(file_get_contents('/path/to/document.pdf'));

      $response = $client->messages->create(
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
                              'data' => $pdfBase64,
                          ],
                          'title' => 'Document Title',
                          'context' => 'Context about the document that will not be cited from',
                          'citations' => ['enabled' => true],
                      ],
                      [
                          'type' => 'text',
                          'text' => 'Summarize this document.',
                      ],
                  ],
              ],
          ],
          model: 'claude-opus-5',
      );

      echo json_encode($response, JSON_PRETTY_PRINT);
      ```

      ```ruby Ruby
      client = Anthropic::Client.new

      pdf_base64 = Base64.strict_encode64(File.binread("/path/to/document.pdf"))

      response = client.messages.create(
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
                  data: pdf_base64
                },
                title: "Document Title",
                context: "Context about the document that will not be cited from",
                citations: { enabled: true }
              },
              {
                type: "text",
                text: "Summarize this document."
              }
            ]
          }
        ]
      )

      puts response
      ```
    </CodeGroup>
  </Tab>

  <Tab title="URL">
    <CodeGroup>
      ```bash cURL
      curl https://api.anthropic.com/v1/messages \
        -H "content-type: application/json" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -d '{
          "model": "claude-opus-5",
          "max_tokens": 1024,
          "messages": [
            {
              "role": "user",
              "content": [
                {
                  "type": "document",
                  "source": {
                    "type": "url",
                    "url": "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf"
                  },
                  "title": "Document Title",
                  "context": "Context about the document that will not be cited from",
                  "citations": {"enabled": true}
                },
                {
                  "type": "text",
                  "text": "Summarize this document."
                }
              ]
            }
          ]
        }'
      ```

      ```bash CLI
      ant messages create <<'YAML'
      model: claude-opus-5
      max_tokens: 1024
      messages:
        - role: user
          content:
            - type: document
              source:
                type: url
                url: https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf
              title: Document Title
              context: Context about the document that will not be cited from
              citations:
                enabled: true
            - type: text
              text: Summarize this document.
      YAML
      ```

      ```python Python
      client = anthropic.Anthropic()

      response = client.messages.create(
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
                          "title": "Document Title",
                          "context": "Context about the document that will not be cited from",
                          "citations": {"enabled": True},
                      },
                      {"type": "text", "text": "Summarize this document."},
                  ],
              }
          ],
      )
      print(response)
      ```

      ```typescript TypeScript
      const client = new Anthropic();

      const response = await client.messages.create({
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
                },
                title: "Document Title",
                context: "Context about the document that will not be cited from",
                citations: { enabled: true }
              },
              {
                type: "text",
                text: "Summarize this document."
              }
            ]
          }
        ]
      });
      console.log(response);
      ```

      ```csharp C#
      var client = new AnthropicClient();

      var response = await client.Messages.Create(
          new()
          {
              Model = Model.ClaudeOpus5,
              MaxTokens = 1024,
              Messages =
              [
                  new()
                  {
                      Role = Role.User,
                      Content = new MessageParamContent(new List<ContentBlockParam>
                      {
                          new ContentBlockParam(new DocumentBlockParam(
                              new DocumentBlockParamSource(new UrlPdfSource()
                              {
                                  Url = "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf",
                              })
                          )
                          {
                              Title = "Document Title",
                              Context = "Context about the document that will not be cited from",
                              Citations = new CitationsConfigParam { Enabled = true },
                          }),
                          new ContentBlockParam(new TextBlockParam("Summarize this document.")),
                      }),
                  },
              ],
          }
      );

      Console.WriteLine(response);
      ```

      ```go Go
      client := anthropic.NewClient()

      response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
      	Model:     anthropic.ModelClaudeOpus5,
      	MaxTokens: 1024,
      	Messages: []anthropic.MessageParam{
      		anthropic.NewUserMessage(
      			anthropic.ContentBlockParamUnion{
      				OfDocument: &anthropic.DocumentBlockParam{
      					Source: anthropic.DocumentBlockParamSourceUnion{
      						OfURL: &anthropic.URLPDFSourceParam{
      							URL: "https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf",
      						},
      					},
      					Title:     anthropic.String("Document Title"),
      					Context:   anthropic.String("Context about the document that will not be cited from"),
      					Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
      				},
      			},
      			anthropic.NewTextBlock("Summarize this document."),
      		),
      	},
      })
      if err != nil {
      	log.Fatal(err)
      }
      fmt.Println(response)
      ```

      ```java Java
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      DocumentBlockParam documentParam = DocumentBlockParam.builder()
          .source(UrlPdfSource.builder()
              .url("https://assets.anthropic.com/m/1cd9d098ac3e6467/original/Claude-3-Model-Card-October-Addendum.pdf")
              .build())
          .title("Document Title")
          .context("Context about the document that will not be cited from")
          .citations(CitationsConfigParam.builder().enabled(true).build())
          .build();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessageOfBlockParams(
              List.of(
                  ContentBlockParam.ofDocument(documentParam),
                  ContentBlockParam.ofText(TextBlockParam.builder().text("Summarize this document.").build())
              )
          )
          .build();

      Message message = client.messages().create(params);
      System.out.println(message);
      ```

      ```php PHP
      $client = new Client();

      $response = $client->messages->create(
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
                          'title' => 'Document Title',
                          'context' => 'Context about the document that will not be cited from',
                          'citations' => ['enabled' => true],
                      ],
                      [
                          'type' => 'text',
                          'text' => 'Summarize this document.',
                      ],
                  ],
              ],
          ],
          model: 'claude-opus-5',
      );

      echo json_encode($response, JSON_PRETTY_PRINT);
      ```

      ```ruby Ruby
      client = Anthropic::Client.new

      response = client.messages.create(
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
                },
                title: "Document Title",
                context: "Context about the document that will not be cited from",
                citations: { enabled: true }
              },
              {
                type: "text",
                text: "Summarize this document."
              }
            ]
          }
        ]
      )

      puts response
      ```
    </CodeGroup>
  </Tab>

  <Tab title="Files API">
    这些示例将通过 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传的文件作为 `document` 源进行引用。

    <CodeGroup>
      ```bash cURL
      curl -X POST https://api.anthropic.com/v1/messages \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "content-type: application/json" \
        -d @- <<EOF
      {
        "model": "claude-opus-5",
        "max_tokens": 1024,
        "messages": [
          {
            "role": "user",
            "content": [
              {
                "type": "document",
                "source": {"type": "file", "file_id": "$FILE_ID"},
                "title": "Document Title",
                "context": "Context about the document that will not be cited from",
                "citations": {"enabled": true}
              },
              {
                "type": "text",
                "text": "Summarize this document."
              }
            ]
          }
        ]
      }
      EOF
      ```

      ```bash CLI
      ant messages create <<YAML
      model: claude-opus-5
      max_tokens: 1024
      messages:
        - role: user
          content:
            - type: document
              source:
                type: file
                file_id: $FILE_ID
              title: Document Title
              context: Context about the document that will not be cited from
              citations:
                enabled: true
            - type: text
              text: Summarize this document.
      YAML
      ```

      ```python Python
      cited_response = client.messages.create(
          model="claude-opus-5",
          max_tokens=1024,
          messages=[
              {
                  "role": "user",
                  "content": [
                      {
                          "type": "document",
                          "source": {"type": "file", "file_id": file_id},
                          "title": "Document Title",
                          "context": "Context about the document that will not be cited from",
                          "citations": {"enabled": True},
                      },
                      {"type": "text", "text": "Summarize this document."},
                  ],
              }
          ],
      )
      print(cited_response)
      ```

      ```typescript TypeScript
      const citedResponse = await client.messages.create({
        model: "claude-opus-5",
        max_tokens: 1024,
        messages: [
          {
            role: "user",
            content: [
              {
                type: "document",
                source: { type: "file", file_id: uploaded.id },
                title: "Document Title",
                context: "Context about the document that will not be cited from",
                citations: { enabled: true },
              },
              {
                type: "text",
                text: "Summarize this document.",
              },
            ],
          },
        ],
      });
      console.log(citedResponse);
      ```

      ```csharp C#
      var citedResponse = await client.Messages.Create(
          new MessageCreateParams
          {
              Model = Model.ClaudeOpus5,
              MaxTokens = 1024,
              Messages =
              [
                  new MessageParam
                  {
                      Role = Role.User,
                      Content = new List<ContentBlockParam>
                      {
                          new DocumentBlockParam
                          {
                              Source = new FileDocumentSource { FileID = fileId },
                              Title = "Document Title",
                              Context = "Context about the document that will not be cited from",
                              Citations = new CitationsConfigParam { Enabled = true },
                          },
                          new TextBlockParam { Text = "Summarize this document." },
                      }
                  }
              ]
          });

      Console.WriteLine(citedResponse);
      ```

      ```go Go
      citedMsg, err := client.Messages.New(context.Background(),
      	anthropic.MessageNewParams{
      		Model:     anthropic.ModelClaudeOpus5,
      		MaxTokens: 1024,
      		Messages: []anthropic.MessageParam{
      			anthropic.NewUserMessage(
      				anthropic.ContentBlockParamUnion{
      					OfDocument: &anthropic.DocumentBlockParam{
      						Source: anthropic.DocumentBlockParamSourceUnion{
      							OfFile: &anthropic.FileDocumentSourceParam{FileID: fileID},
      						},
      						Title:     anthropic.String("Document Title"),
      						Context:   anthropic.String("Context about the document that will not be cited from"),
      						Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
      					},
      				},
      				anthropic.NewTextBlock("Summarize this document."),
      			),
      		},
      	})
      if err != nil {
      	log.Fatal(err)
      }
      fmt.Println(citedMsg)
      ```

      ```java Java
      MessageCreateParams citedParams = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessageOfBlockParams(List.of(
              ContentBlockParam.ofDocument(DocumentBlockParam.builder()
                  .fileSource(fileId)
                  .title("Document Title")
                  .context("Context about the document that will not be cited from")
                  .citations(CitationsConfigParam.builder().enabled(true).build())
                  .build()),
              ContentBlockParam.ofText(TextBlockParam.builder()
                  .text("Summarize this document.")
                  .build())
          ))
          .build();

      Message citedMessage = client.messages().create(citedParams);
      System.out.println(citedMessage);
      ```

      ```php PHP
      $citedResponse = $client->messages->create(
          maxTokens: 1024,
          messages: [
              [
                  'role' => 'user',
                  'content' => [
                      [
                          'type' => 'document',
                          'source' => ['type' => 'file', 'fileID' => $fileId],
                          'title' => 'Document Title',
                          'context' => 'Context about the document that will not be cited from',
                          'citations' => ['enabled' => true],
                      ],
                      ['type' => 'text', 'text' => 'Summarize this document.'],
                  ],
              ],
          ],
          model: 'claude-opus-5',
      );

      echo $citedResponse;
      ```

      ```ruby Ruby
      cited_response = client.messages.create(
        model: "claude-opus-5",
        max_tokens: 1024,
        messages: [
          {
            role: "user",
            content: [
              {
                type: "document",
                source: { type: "file", file_id: file_id },
                title: "Document Title",
                context: "Context about the document that will not be cited from",
                citations: { enabled: true }
              },
              {
                type: "text",
                text: "Summarize this document."
              }
            ]
          }
        ]
      )

      puts cited_response
      ```
    </CodeGroup>
  </Tab>
</Tabs>

<Accordion title="PDF 引用示例">
  ```json
  {
    "type": "page_location",
    "cited_text": "The exact text being cited", // not counted toward output tokens
    "document_index": 0,
    "document_title": "Document Title",
    "start_page_number": 1, // 1-indexed
    "end_page_number": 2 // exclusive
  }
  ```
</Accordion>

### 自定义内容文档

自定义内容文档让您可以控制引用粒度。不会进行额外的分块，各块将按照所提供的内容块提供给模型。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [
        {
          "role": "user",
          "content": [
            {
              "type": "document",
              "source": {
                "type": "content",
                "content": [
                  {"type": "text", "text": "First chunk"},
                  {"type": "text", "text": "Second chunk"}
                ]
              },
              "title": "Document Title",
              "context": "Context about the document that will not be cited from",
              "citations": {"enabled": true}
            },
            {
              "type": "text",
              "text": "Summarize this document."
            }
          ]
        }
      ]
    }'
  ```

  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: document
          source:
            type: content
            content:
              - type: text
                text: First chunk
              - type: text
                text: Second chunk
          title: Document Title
          context: Context about the document that will not be cited from
          citations:
            enabled: true
        - type: text
          text: Summarize this document.
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "document",
                      "source": {
                          "type": "content",
                          "content": [
                              {"type": "text", "text": "First chunk"},
                              {"type": "text", "text": "Second chunk"},
                          ],
                      },
                      "title": "Document Title",
                      "context": "Context about the document that will not be cited from",
                      "citations": {"enabled": True},
                  },
                  {"type": "text", "text": "Summarize this document."},
              ],
          }
      ],
  )
  print(response)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "content",
              content: [
                { type: "text", text: "First chunk" },
                { type: "text", text: "Second chunk" }
              ]
            },
            title: "Document Title",
            context: "Context about the document that will not be cited from",
            citations: { enabled: true }
          },
          {
            type: "text",
            text: "Summarize this document."
          }
        ]
      }
    ]
  });
  console.log(response);
  ```

  ```csharp C#
  var client = new AnthropicClient();

  var response = await client.Messages.Create(
      new()
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 1024,
          Messages =
          [
              new()
              {
                  Role = Role.User,
                  Content = new MessageParamContent(new List<ContentBlockParam>
                  {
                      new ContentBlockParam(new DocumentBlockParam(
                          new DocumentBlockParamSource(new ContentBlockSource()
                          {
                              Content = new ContentBlockSourceContent(new List<MessageContentBlockSourceContent>
                              {
                                  new TextBlockParam("First chunk"),
                                  new TextBlockParam("Second chunk"),
                              }),
                          })
                      )
                      {
                          Title = "Document Title",
                          Context = "Context about the document that will not be cited from",
                          Citations = new CitationsConfigParam { Enabled = true },
                      }),
                      new ContentBlockParam(new TextBlockParam("Summarize this document.")),
                  }),
              },
          ],
      }
  );

  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.ContentBlockParamUnion{
  				OfDocument: &anthropic.DocumentBlockParam{
  					Source: anthropic.DocumentBlockParamSourceUnion{
  						OfContent: &anthropic.ContentBlockSourceParam{
  							Content: anthropic.ContentBlockSourceContentUnionParam{
  								OfContentBlockSourceContent: []anthropic.ContentBlockSourceContentItemUnionParam{
  									{OfText: &anthropic.TextBlockParam{Text: "First chunk"}},
  									{OfText: &anthropic.TextBlockParam{Text: "Second chunk"}},
  								},
  							},
  						},
  					},
  					Title:     anthropic.String("Document Title"),
  					Context:   anthropic.String("Context about the document that will not be cited from"),
  					Citations: anthropic.CitationsConfigParam{Enabled: anthropic.Bool(true)},
  				},
  			},
  			anthropic.NewTextBlock("Summarize this document."),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  DocumentBlockParam documentParam = DocumentBlockParam.builder()
      .source(ContentBlockSource.builder()
          .contentOfBlockSource(
              List.of(
                  ContentBlockSourceContent.ofText(TextBlockParam.builder().text("First chunk").build()),
                  ContentBlockSourceContent.ofText(TextBlockParam.builder().text("Second chunk").build())
              )
          )
          .build())
      .title("Document Title")
      .context("Context about the document that will not be cited from")
      .citations(CitationsConfigParam.builder().enabled(true).build())
      .build();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(1024)
      .addUserMessageOfBlockParams(
          List.of(
              ContentBlockParam.ofDocument(documentParam),
              ContentBlockParam.ofText(TextBlockParam.builder().text("Summarize this document.").build())
          )
      )
      .build();

  Message message = client.messages().create(params);
  System.out.println(message);
  ```

  ```php PHP
  $client = new Client();

  $response = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'document',
                      'source' => [
                          'type' => 'content',
                          'content' => [
                              ['type' => 'text', 'text' => 'First chunk'],
                              ['type' => 'text', 'text' => 'Second chunk'],
                          ],
                      ],
                      'title' => 'Document Title',
                      'context' => 'Context about the document that will not be cited from',
                      'citations' => ['enabled' => true],
                  ],
                  [
                      'type' => 'text',
                      'text' => 'Summarize this document.',
                  ],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo json_encode($response, JSON_PRETTY_PRINT);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "content",
              content: [
                { type: "text", text: "First chunk" },
                { type: "text", text: "Second chunk" }
              ]
            },
            title: "Document Title",
            context: "Context about the document that will not be cited from",
            citations: { enabled: true }
          },
          {
            type: "text",
            text: "Summarize this document."
          }
        ]
      }
    ]
  )

  puts response
  ```
</CodeGroup>

<Accordion title="引用示例">
  ```json
  {
    "type": "content_block_location",
    "cited_text": "The exact text being cited", // not counted toward output tokens
    "document_index": 0,
    "document_title": "Document Title",
    "start_block_index": 0, // 0-indexed
    "end_block_index": 1 // exclusive
  }
  ```
</Accordion>

***

## 响应结构

启用引用后，响应将包含多个带有引用的文本块：

```json
{
  "content": [
    { "type": "text", "text": "According to the document, " },
    {
      "type": "text",
      "text": "the grass is green",
      "citations": [
        {
          "type": "char_location",
          "cited_text": "The grass is green.",
          "document_index": 0,
          "document_title": "Example Document",
          "start_char_index": 0,
          "end_char_index": 20
        }
      ]
    },
    { "type": "text", "text": " and " },
    {
      "type": "text",
      "text": "the sky is blue",
      "citations": [
        {
          "type": "char_location",
          "cited_text": "The sky is blue.",
          "document_index": 0,
          "document_title": "Example Document",
          "start_char_index": 20,
          "end_char_index": 36
        }
      ]
    },
    {
      "type": "text",
      "text": ". Information from page 5 states that "
    },
    {
      "type": "text",
      "text": "water is essential",
      "citations": [
        {
          "type": "page_location",
          "cited_text": "Water is essential for life.",
          "document_index": 1,
          "document_title": "PDF Document",
          "start_page_number": 5,
          "end_page_number": 6
        }
      ]
    },
    {
      "type": "text",
      "text": ". The custom document mentions "
    },
    {
      "type": "text",
      "text": "important findings",
      "citations": [
        {
          "type": "content_block_location",
          "cited_text": "These are important findings.",
          "document_index": 2,
          "document_title": "Custom Content Document",
          "start_block_index": 0,
          "end_block_index": 1
        }
      ]
    }
  ]
}
```

### 流式传输支持

对于 "streaming"（流式传输）响应，引用以 `content_block_delta` 事件中的 `citations_delta` 增量类型到达。每个增量包含一条引用，用于添加到当前 `text` 内容块的 `citations` 列表中。

<AccordionGroup>
  <Accordion title="流式传输事件示例">
    ```sse
    event: message_start
    data: {"type": "message_start", ...}

    event: content_block_start
    data: {"type": "content_block_start", "index": 0, ...}

    event: content_block_delta
    data: {"type": "content_block_delta", "index": 0,
           "delta": {"type": "text_delta", "text": "According to..."}}

    event: content_block_delta
    data: {"type": "content_block_delta", "index": 0,
           "delta": {"type": "citations_delta",
                     "citation": {
                         "type": "char_location",
                         "cited_text": "...",
                         "document_index": 0,
                         ...
                     }}}

    event: content_block_stop
    data: {"type": "content_block_stop", "index": 0}

    event: message_stop
    data: {"type": "message_stop"}
    ```
  </Accordion>
</AccordionGroup>

## 后续步骤

<CardGroup cols={2}>
  <Card title="流式传输消息" icon="wifi-high" href="https://platform.claude.com/docs/zh-CN/build-with-claude/streaming">
    在处理文本增量的同时处理 `citations_delta` 增量类型，以便在流式传输过程中渲染带引用的响应。
  </Card>

  <Card title="搜索结果" icon="book-bookmark" href="https://platform.claude.com/docs/zh-CN/build-with-claude/search-results">
    将 RAG 管道中的搜索结果作为具有内置引用支持的一等内容块传入。
  </Card>

  <Card title="PDF 支持" icon="file" href="https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support">
    了解 Claude 如何从 PDF 中提取文本，以及基于页码的引用如何映射回您的源文件。
  </Card>

  <Card title="Files API" icon="hard-drives" href="https://platform.claude.com/docs/zh-CN/build-with-claude/files">
    一次上传文档，即可在多个引用请求中通过 `file_id` 引用它们。
  </Card>
</CardGroup>
