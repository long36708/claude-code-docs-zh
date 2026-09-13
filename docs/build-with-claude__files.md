---
title: Files API
url: https://platform.claude.com/docs/zh-CN/build-with-claude/files
description: 上传文件一次，在 Messages 请求中通过 file_id 引用它们，并下载由 skills 或代码执行工具创建的输出。
---

## Compatibility
- [ZDR](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): not eligible
- Platforms: Claude API, Claude Platform on AWS (beta), Microsoft Foundry (beta) [1]; not available on Amazon Bedrock, Google Cloud
1. 在 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上，Files API 需要 [Hosted on Anthropic 部署](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry#additional-features-not-supported-when-hosted-on-azure)。

Files API 让您可以上传和管理文件以便与 Claude API 一起使用，而无需在每次请求时重新上传内容。这在使用[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)提供输入（例如数据集和文档）然后下载输出（例如图表）时特别有用。除本指南外，您还可以[直接浏览 API 参考](https://platform.claude.com/docs/zh-CN/api/files/upload)。

## 文件类型支持

在 Messages 请求中引用 `file_id` 在所有支持给定文件类型的模型上均受支持。[图像](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)在所有当前的 Claude 模型上均受支持。对于 [PDF](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support) 和[代码执行工具支持的其他文件类型](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#compatibility)，请参阅链接页面了解模型支持情况。

## Files API 的工作原理

Files API 提供了一种"创建一次、多次使用"的文件处理方式：

* **上传文件**到 Anthropic 的安全存储并获得唯一的 `file_id`
* **下载文件**，这些文件由 skills 或代码执行工具创建
* **引用文件**，在 [Messages](https://platform.claude.com/docs/zh-CN/api/messages/create) 请求中使用 `file_id` 而不是重新上传内容
* **管理您的文件**，通过列出、检索和删除操作

<Warning id="workspace-scoped-access">
  **上传的文件可供您的整个工作区访问，而不限定于某个终端用户、对话或会话。** 任何有权访问某个工作区的 API 密钥都可以访问上传到该工作区的任何文件。每个服务账户，以及每个组织角色允许 API 访问的用户，除了您将其添加到的任何工作区之外，还可以使用默认工作区（Default Workspace），因此请将必须保持隔离的文件放在它们自己的[工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#api-keys-and-resource-scoping)中，并仅使用限定于该工作区的密钥访问它们。切勿接受来自终端用户或其他不受信任来源的 `file_id` 值：用户提供的文件 ID 会让您应用程序的一个用户读取另一个用户上传的内容。请将文件 ID 视为服务器端引用，并在您的应用程序中维护用户与其文件之间的映射关系。

  如果您正在基于 Files API 构建多租户应用程序，请为每个租户创建一个单独的[工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces)。工作区是文件的隔离边界，因此每个租户一个工作区可以使每个租户的数据与其他所有租户实现硬隔离。每个组织最多可以拥有 100 个工作区；如果您需要更多，请联系您的客户团队。
</Warning>

## 如何使用 Files API

### 上传文件

上传文件以便在将来的 API 调用中引用：

<CodeGroup>
  ```bash cURL
  FILE_ID=$(curl -X POST https://api.anthropic.com/v1/files \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -F "file=@/path/to/document.pdf" | jq -r '.id')
  echo "$FILE_ID"
  ```

  ```bash CLI
  FILE_ID=$(ant files upload \
    --file /path/to/document.pdf \
    --transform id \
    --raw-output)
  echo "$FILE_ID"
  ```

  ```python Python
  uploaded = client.files.upload(
      file=("document.pdf", open("/path/to/document.pdf", "rb"), "application/pdf"),
  )
  file_id = uploaded.id
  print(file_id)
  ```

  ```typescript TypeScript
  const uploaded = await client.files.upload({
    file: await toFile(
      fs.createReadStream("/path/to/document.pdf"),
      undefined,
      { type: "application/pdf" },
    ),
  });
  console.log(uploaded.id);
  ```

  ```csharp C#
  var uploaded = await client.Files.Upload(
      new FileUploadParams
      {
          File = new BinaryContent
          {
              Stream = File.OpenRead("/path/to/document.pdf"),
              FileName = "document.pdf",
              ContentType = new("application/pdf")
          }
      });

  var fileId = uploaded.ID;
  Console.WriteLine(fileId);
  ```

  ```go Go
  f, err := os.Open("/path/to/document.pdf")
  if err != nil {
  	log.Fatal(err)
  }
  defer f.Close()

  response, err := client.Files.Upload(context.Background(),
  	anthropic.FileUploadParams{
  		File: anthropic.File(f, "document.pdf", "application/pdf"),
  	})
  if err != nil {
  	log.Fatal(err)
  }

  fileID := response.ID
  fmt.Println(fileID)
  ```

  ```java Java
  FileMetadata file = client.files().upload(
      FileUploadParams.builder()
          .file(MultipartField.<InputStream>builder()
              .value(Files.newInputStream(Path.of("/path/to/document.pdf")))
              .filename("document.pdf")
              .contentType("application/pdf")
              .build())
          .build()
  );

  String fileId = file.id();
  System.out.println(fileId);
  ```

  ```php PHP
  $file = $client->files->upload(
      file: FileParam::fromResource(fopen('/path/to/document.pdf', 'rb'), contentType: 'application/pdf'),
  );

  $fileId = $file->id;
  echo $fileId;
  ```

  ```ruby Ruby
  file = client.files.upload(
    file: Anthropic::FilePart.new(
      Pathname("/path/to/document.pdf"),
      content_type: "application/pdf"
    )
  )

  file_id = file.id
  puts file_id
  ```
</CodeGroup>

上传文件的响应包括：

```json Response
{
  "id": "file_011CNha8iCJcU1wXNR6q4V8w",
  "type": "file",
  "filename": "document.pdf",
  "mime_type": "application/pdf",
  "size_bytes": 1024000,
  "created_at": "2025-01-01T00:00:00Z",
  "downloadable": false,
  "expires_at": null
}
```

对于您上传的文件，`downloadable` 为 `false`。只有由 [skills](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide) 或[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)创建的文件才能下载。请参阅[下载文件](https://platform.claude.com/docs/zh-CN/build-with-claude/files#downloading-a-file)。

### 在消息中使用文件

上传后，通过将上传响应中的 `id` 作为 `file_id` 传递来引用该文件：

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
            "type": "text",
            "text": "Please summarize this document for me."
          },
          {
            "type": "document",
            "source": {
              "type": "file",
              "file_id": "$FILE_ID"
            }
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
        - type: text
          text: Please summarize this document for me.
        - type: document
          source:
            type: file
            file_id: $FILE_ID
  YAML
  ```

  ```python Python
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {"type": "text", "text": "Please summarize this document for me."},
                  {
                      "type": "document",
                      "source": {
                          "type": "file",
                          "file_id": file_id,
                      },
                  },
              ],
          }
      ],
  )
  print(response)
  ```

  ```typescript TypeScript
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: "Please summarize this document for me.",
          },
          {
            type: "document",
            source: {
              type: "file",
              file_id: uploaded.id,
            },
          },
        ],
      },
    ],
  });

  console.log(response);
  ```

  ```csharp C#
  var response = await client.Messages.Create(
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
                      new TextBlockParam { Text = "Please summarize this document for me." },
                      new DocumentBlockParam
                      {
                          Source = new FileDocumentSource { FileID = fileId }
                      }
                  }
              }
          ]
      });

  Console.WriteLine(response);
  ```

  ```go Go
  msg, err := client.Messages.New(context.Background(),
  	anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeOpus5,
  		MaxTokens: 1024,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(
  				anthropic.NewTextBlock("Please summarize this document for me."),
  				anthropic.NewDocumentBlock(anthropic.FileDocumentSourceParam{
  					FileID: fileID,
  				}),
  			),
  		},
  	})
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(msg)
  ```

  ```java Java
  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(1024)
      .addUserMessageOfBlockParams(List.of(
          ContentBlockParam.ofText(TextBlockParam.builder()
              .text("Please summarize this document for me.")
              .build()),
          ContentBlockParam.ofDocument(DocumentBlockParam.builder()
              .fileSource(fileId)
              .build())
      ))
      .build();

  Message message = client.messages().create(params);
  System.out.println(message);
  ```

  ```php PHP
  $response = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  ['type' => 'text', 'text' => 'Please summarize this document for me.'],
                  [
                      'type' => 'document',
                      'source' => [
                          'type' => 'file',
                          'fileID' => $fileId,
                      ],
                  ],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo $response;
  ```

  ```ruby Ruby
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          { type: "text", text: "Please summarize this document for me." },
          {
            type: "document",
            source: {
              type: "file",
              file_id: file_id
            }
          }
        ]
      }
    ]
  )

  puts response
  ```
</CodeGroup>

### 文件类型和内容块

Files API 支持不同的文件类型，它们对应不同的内容块类型：

| 文件类型                                                                                                                             | MIME 类型                                              | 内容块类型              | 用例         |
| -------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ------------------ | ---------- |
| PDF                                                                                                                              | `application/pdf`                                    | `document`         | 文本分析、文档处理  |
| 纯文本                                                                                                                              | `text/plain`                                         | `document`         | 文本分析、处理    |
| 图像                                                                                                                               | `image/jpeg`, `image/png`, `image/gif`, `image/webp` | `image`            | 图像分析、视觉任务  |
| [数据集及其他](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#upload-and-analyze-your-own-files) | 不定                                                   | `container_upload` | 分析数据、创建可视化 |

#### 文档块

对于 PDF 和文本文件，使用 `document` 内容块：

```json
{
  "type": "document",
  "source": {
    "type": "file",
    "file_id": "file_011CNha8iCJcU1wXNR6q4V8w"
  },
  "title": "Document Title", // Optional
  "context": "Context about the document", // Optional
  "citations": { "enabled": true } // Optional, enables citations
}
```

#### 图像块

对于图像，使用 `image` 内容块：

```json
{
  "type": "image",
  "source": {
    "type": "file",
    "file_id": "file_011CPMxVD3fHLUhvTqtsQA5w"
  }
}
```

#### 容器上传块

要将文件发送到[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#upload-and-analyze-your-own-files)，使用 `container_upload` 内容块：

```json
{
  "type": "container_upload",
  "file_id": "file_011CNha8iCJcU1wXNR6q4V8w"
}
```

### 处理其他文件格式

对于 `document` 块不支持的文件类型（例如 .docx 和 .xlsx），请将文件转换为纯文本并将内容直接包含在您的消息中。已经是纯文本的文件（例如 .csv 和 .md 文件）既可以通过这种方式读取，也可以通过 Files API 以显式的 `text/plain` 内容类型上传。若要分析数据集而不是将其作为文本读取，请使用 `container_upload` 块将其上传给[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#upload-and-analyze-your-own-files)。

以下示例读取一个文本文件并将其内容作为纯文本发送：

<CodeGroup>
  ```bash cURL
  # 读取文本文件
  # 注意：对于包含特殊字符的文件，请考虑使用 base64 编码
  TEXT_CONTENT=$(cat document.txt)

  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d @- <<EOF
  {
    "model": "claude-opus-5",
    "max_tokens": 1024,
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "Here's the document content:\n\n${TEXT_CONTENT}\n\nPlease summarize this document."
          }
        ]
      }
    ]
  }
  EOF
  ```

  ```bash CLI
  # "@./path" 引用会将文件内容直接内联到该字段中。
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --transform 'content.#(type=="text").text' \
    --raw-output <<'YAML'
  messages:
    - role: user
      content:
        - type: text
          text: "Here's the document content:"
        - type: text
          text: "@./document.txt"
        - type: text
          text: "Please summarize this document."
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  # 读取文本文件
  with open("document.txt") as f:
      text_content = f.read()

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "text",
                      "text": f"Here's the document content:\n\n{text_content}\n\nPlease summarize this document.",
                  }
              ],
          }
      ],
  )

  for block in response.content:
      if block.type == "text":
          print(block.text)
  ```

  ```typescript TypeScript
  import fs from "node:fs/promises";
  // ...
  const client = new Anthropic();

  // 读取文本文件
  const textContent = await fs.readFile("document.txt", "utf-8");

  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: `Here's the document content:\n\n${textContent}\n\nPlease summarize this document.`
          }
        ]
      }
    ]
  });

  const textBlock = response.content.find(
    (block): block is Anthropic.TextBlock => block.type === "text"
  );
  console.log(textBlock?.text);
  ```

  ```csharp C#
  AnthropicClient client = new();

  // 读取文本文件
  string textContent = await File.ReadAllTextAsync("document.txt");

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages = [new()
      {
          Role = Role.User,
          Content = $"Here's the document content:\n\n{textContent}\n\nPlease summarize this document."
      }]
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

  ```go Go
  client := anthropic.NewClient()

  // 读取文本文件
  textContent, err := os.ReadFile("document.txt")
  if err != nil {
  	log.Fatal(err)
  }

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock(
  			fmt.Sprintf("Here's the document content:\n\n%s\n\nPlease summarize this document.", string(textContent)),
  		)),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  for _, block := range response.Content {
  	if textBlock, ok := block.AsAny().(anthropic.TextBlock); ok {
  		fmt.Println(textBlock.Text)
  	}
  }
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  // 读取文本文件
  String textContent = Files.readString(Path.of("document.txt"));

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(1024L)
      .addUserMessage("Here's the document content:\n\n" + textContent + "\n\nPlease summarize this document.")
      .build();

  Message response = client.messages().create(params);
  response.content().stream()
      .flatMap(block -> block.text().stream())
      .forEach(textBlock -> System.out.println(textBlock.text()));
  ```

  ```php PHP
  $client = new Client();

  // 读取文本文件
  $textContent = file_get_contents("document.txt");

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'text',
                      'text' => "Here's the document content:\n\n{$textContent}\n\nPlease summarize this document."
                  ]
              ]
          ]
      ],
      model: 'claude-opus-5',
  );

  foreach ($message->content as $block) {
      if ($block->type === 'text') {
          echo $block->text, PHP_EOL;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 读取文本文件
  text_content = File.read("document.txt")

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: "Here's the document content:\n\n#{text_content}\n\nPlease summarize this document."
          }
        ]
      }
    ]
  )

  message.content.each do |block|
    puts block.text if block.type == :text
  end
  ```
</CodeGroup>

<Note>
  对于包含图像的 .docx 文件，请先将其转换为 PDF 格式，然后使用 [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)来利用内置的图像解析功能。这样可以使用来自 PDF 文档的引用。
</Note>

### 管理文件

#### 列出文件

检索您已上传文件的列表。该端点是分页的：每个请求最多返回 `limit` 个文件（默认 20 个，最多 1,000 个），响应中的 `next_page` 游标在作为 `page` 参数传回时可获取下一页。文件按最新优先排序。请参阅[列出文件 API 参考](https://platform.claude.com/docs/zh-CN/api/files/list)。SDK 返回第一页并提供自动分页辅助工具。CLI 示例使用 `--max-items` 限制总数：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/files \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant files list --max-items 10
  ```

  ```python Python
  client = anthropic.Anthropic()
  files = client.files.list()
  print(files)
  ```

  ```typescript TypeScript
  const client = new Anthropic();
  const files = await client.files.list();
  console.log(files);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var files = await client.Files.List();
  Console.WriteLine(files);
  ```

  ```go Go
  client := anthropic.NewClient()

  files, err := client.Files.List(context.TODO(), anthropic.FileListParams{})
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(files)
  ```

  ```java Java
  import com.anthropic.models.files.FileListPage;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      FileListPage files = client.files().list();
      System.out.println(files);
  }
  ```

  ```php PHP
  $client = new Client();

  $files = $client->files->list();
  echo $files;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  files = client.files.list
  puts files
  ```
</CodeGroup>

若要在一个请求中检查一组已知文件而不是分页，请将最多 100 个文件 ID 作为 `ids[]` 查询参数传递。`ids[]` 请求始终返回单个页面（`next_page` 为 `null`），任何无法解析为您工作区中文件的 ID 都会从 `data` 中被静默省略；请将返回的 ID 与请求的 ID 进行比较以检测缺失项。`ids[]` 不能与 `page` 或 `limit` 组合使用。

#### 获取文件元数据

检索特定文件的信息：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/files/$FILE_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant files retrieve-metadata \
    --file-id "$FILE_ID"
  ```

  ```python Python
  file = client.files.retrieve_metadata(file_id)
  print(file)
  ```

  ```typescript TypeScript
  const file = await client.files.retrieveMetadata(uploaded.id);
  console.log(file);
  ```

  ```csharp C#
  var file = await client.Files.RetrieveMetadata(fileId);
  Console.WriteLine(file);
  ```

  ```go Go
  metadata, err := client.Files.GetMetadata(context.TODO(), fileID)
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(metadata)
  ```

  ```java Java
  FileMetadata metadata = client.files().retrieveMetadata(fileId);

  System.out.println(metadata);
  ```

  ```php PHP
  $file = $client->files->retrieveMetadata($fileId);
  echo $file;
  ```

  ```ruby Ruby
  file = client.files.retrieve_metadata(file_id)
  puts file
  ```
</CodeGroup>

#### 删除文件

从您的工作区中移除文件：

<CodeGroup>
  ```bash cURL
  curl -X DELETE "https://api.anthropic.com/v1/files/$FILE_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant files delete \
    --file-id "$FILE_ID"
  ```

  ```python Python
  client.files.delete(file_id)
  ```

  ```typescript TypeScript
  await client.files.delete(uploaded.id);
  ```

  ```csharp C#
  await client.Files.Delete(fileId);
  ```

  ```go Go
  _, err = client.Files.Delete(context.TODO(), fileID)
  if err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  client.files().delete(fileId);
  ```

  ```php PHP
  $client->files->delete($fileId);
  ```

  ```ruby Ruby
  client.files.delete(file_id)
  ```
</CodeGroup>

### 下载文件

下载由 [skills](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide) 或[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)创建的文件。您上传的文件无法下载。生成文件的 `file_id` 出现在创建它的 Messages 响应的 [`bash_code_execution_tool_result` 内容块](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#retrieve-generated-files)中：

<CodeGroup>
  ```bash cURL
  curl -X GET "https://api.anthropic.com/v1/files/$FILE_ID/content" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    --output downloaded_file.txt
  ```

  ```bash CLI
  ant files download \
    --file-id "$FILE_ID" \
    --output downloaded_file.txt
  ```

  ```python Python
  file_content = client.files.download(file_id)

  file_content.write_to_file("downloaded_file.txt")
  ```

  ```typescript TypeScript
  const content = await client.files.download(uploaded.id);

  const bytes = Buffer.from(await content.arrayBuffer());
  await fsp.writeFile("downloaded_file.txt", bytes);
  ```

  ```csharp C#
  using var fileContent = await client.Files.Download(fileId);
  await using var source = await fileContent.ReadAsStream();
  await using var destination = File.Create("downloaded_file.txt");
  await source.CopyToAsync(destination);
  ```

  ```go Go
  func downloadFile(client anthropic.Client, fileID string) error {
  	resp, err := client.Files.Download(context.TODO(), fileID)
  	if err != nil {
  		return err
  	}
  	defer resp.Body.Close()

  	out, err := os.Create("downloaded_file.txt")
  	if err != nil {
  		return err
  	}
  	defer out.Close()

  	_, err = io.Copy(out, resp.Body)
  	return err
  }

  ```

  ```java Java
  try (HttpResponse response = client.files().download(fileId)) {
      try (InputStream body = response.body()) {
          Files.copy(body, Path.of("downloaded_file.txt"),
              StandardCopyOption.REPLACE_EXISTING);
      }
  }
  ```

  ```php PHP
  $fileContent = $client->files->download($fileId);

  file_put_contents('downloaded_file.txt', $fileContent);
  ```

  ```ruby Ruby
  file_content = client.files.download(file_id)

  File.binwrite("downloaded_file.txt", file_content.read)
  ```
</CodeGroup>

<Note>
  只有当文件的元数据显示 `"downloadable": true` 时，该文件才可下载，由 skills 或代码执行工具创建的文件即属于这种情况。下载您上传的文件会返回 400 错误。
</Note>

在 Claude API 上，Claude 使用代码执行工具生成的受支持的图像和视频文件（包括由 skills 创建的文件）在您下载时会携带已签名的 C2PA 内容凭证（Content Credentials）。请参阅[生成文件上的内容凭证](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#content-credentials-on-generated-files)了解凭证包含的内容以及如何验证它。

## 文件存储和限制

### 存储限制

* **最大文件大小：** 每个文件 500 MB
* **总存储量：** 每个组织 1 TB

### 文件生命周期

* 文件限定于其上传所在的工作区。同一工作区中的任何请求都可以引用它们；切勿接受来自不受信任来源的文件 ID（请参阅[工作区访问警告](https://platform.claude.com/docs/zh-CN/build-with-claude/files#workspace-scoped-access)）
* 文件上传后无法修改或重命名。要更改文件的内容，请上传新文件并删除旧文件
* 文件会一直保留，直到您使用 `DELETE /v1/files/{file_id}` 端点删除它们，或它们到达其 `expires_at`
* 已删除的文件无法恢复
* 文件在删除后不久即无法通过 API 访问，但它们可能仍存在于活跃的 Messages API 调用和相关的工具使用中
* 用户删除的文件将根据 Anthropic 的[数据保留政策](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)进行删除。有关所有功能的 ZDR 资格，请参阅 [API 和数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)

### 文件过期

要让文件自动过期，请在上传时包含一个 `expires_in_seconds` 表单字段。该值是一个介于 3,600（1 小时）和 7,776,000（90 天）之间的整数秒数。生成的 `expires_at` 时间戳（RFC 3339）出现在每个文件响应中，对于未设置过期时间上传的文件则为 `null`。过期时间在上传时设置一次，之后无法更改。

当文件到达其 `expires_at` 时：

* 下载其内容（`GET /v1/files/{file_id}/content`）会返回 404 错误
* 引用该文件的 Messages 请求会在推理之前失败
* 其元数据（`GET /v1/files/{file_id}`）在最多 30 天内仍可读取，其中 `expires_at` 为过去的时间
* 在该时间窗口内，它会继续出现在列表响应中；请将 `expires_at` 与当前时间进行比较以过滤已过期的文件

使用 `DELETE /v1/files/{file_id}` 删除已过期的文件会立即移除其元数据，而无需等待 30 天窗口期结束。

<Note>
  过期是一项生命周期功能，而非保证删除的控制手段。在 `expires_at` 之后，文件内容将无法再通过 API 检索，并从您的存储配额中释放；底层内容此后可能会为安全审查而保留有限的一段时间，然后才被永久删除，且文件元数据在过期后最多 30 天内仍然可见。要在文件计划过期之前移除它，请使用 `DELETE /v1/files/{file_id}`。
</Note>

### 审计日志

如果您的组织启用了 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api)，其[活动源（Activity Feed）](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)会记录使用 Claude API 密钥或从 Claude Console 进行的 Files API 操作：每次上传（`POST /v1/files`）、内容下载（`GET /v1/files/{file_id}/content`）和删除（`DELETE /v1/files/{file_id}`）分别显示为 `platform_file_uploaded`、`platform_file_content_downloaded` 或 `platform_file_deleted` 活动。列出文件和检索文件元数据不会被记录。在 Compliance API 关闭期间发生的操作不会被记录，且之后无法恢复，因此在依赖此审计记录之前，请先[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#monitoring-and-logging) 上，请改用 AWS CloudTrail 数据事件来审计文件操作。

## 从 `files-api-2025-04-14` 迁移

Files API 已结束 beta 阶段，不再需要 beta 标头。从 `files-api-2025-04-14` 迁移是可选的：仍然发送该标头的请求会继续正常工作并继续返回 beta 响应结构，因此现有集成在您更改之前会一直正常工作。移除该标头会将这些请求切换为本页所记录的结构：

|                        | 使用 `files-api-2025-04-14`               | 不使用该标头                                                       |
| ---------------------- | --------------------------------------- | ------------------------------------------------------------ |
| 列表响应                   | `{ data, has_more, first_id, last_id }` | `{ data, next_page }`；将 `next_page` 作为 `page` 查询参数传回         |
| 列表游标                   | `before_id`、`after_id`                  | `page`，或最多 100 个 `ids[]`（`before_id` 和 `after_id` 返回 400 错误） |
| 文件对象上的 `expires_at`    | 不返回                                     | 始终存在；当文件没有过期时间时为 `null`                                      |
| 上传文件部分的 `Content-Type` | 必需                                      | 可选；省略时会自动检测类型                                                |

迁移步骤：

1. **移除 beta 标头。** 从您的请求中删除 `anthropic-beta: files-api-2025-04-14`。在 SDK 中，调用 `client.files` 而不是 `client.beta.files`；继续使用 `client.beta.files` 仅在[不再发送该标头的 SDK 版本](https://platform.claude.com/docs/zh-CN/build-with-claude/files#sdk-beta-namespace)上有效。更早的版本即使没有 `betas` 参数也会从 `client.beta.files` 发送该标头。
2. **更新分页。** 将 `after_id`/`before_id` 循环替换为 `page`/`next_page` 游标，或使用[管理文件](https://platform.claude.com/docs/zh-CN/build-with-claude/files#managing-files)中展示的 SDK 自动分页辅助工具。
3. **读取 `expires_at`。** 该字段仅在不使用该标头时出现；`null` 表示文件没有过期时间（请参阅[文件过期](https://platform.claude.com/docs/zh-CN/build-with-claude/files#file-expiration)）。

### SDK beta 命名空间

从 Python SDK 1.2.0、TypeScript SDK 0.122.0、Go SDK 1.68.0、Java SDK 2.59.0、Ruby SDK 1.67.0 和 C# SDK 12.44.0 开始，`client.beta.files` 不再发送 `files-api-2025-04-14`，并返回与 `client.files` 相同的结构，类型名称带有 `Beta` 前缀。它接受一个 `betas` 参数，用于仍处于 beta 阶段的 Files 功能，例如在 [Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/files) beta 标头下的 `scope_id` 过滤。更早的 SDK 版本的类型定义对应 beta 结构；如果您依赖这些类型，请在迁移之前继续使用更早的版本。

携带 `anthropic-beta: managed-agents-2026-04-01` 但不携带 `files-api-2025-04-14` 的请求会收到本页所述的结构，并在 `GET /v1/files` 上有一项兼容性便利：`before_id` 和 `after_id` 仍被接受（不能与 `page` 或 `ids[]` 组合使用），且列表响应除 `next_page` 外还包含 `has_more`、`first_id` 和 `last_id`。更高版本的 Managed Agents beta 会收到普通结构。

## 错误处理

使用 Files API 时的常见错误包括：

* **文件未找到（404）：** 指定的 `file_id` 不存在或您无权访问它
* **无效的文件类型（400）：** 文件类型与内容块类型不匹配（例如，在文档块中使用图像文件）
* **不可下载（400）：** 您上传的文件具有 `"downloadable": false`，无法下载。只有由 skills 或代码执行工具创建的文件才能下载
* **超出上下文窗口大小（400）：** 文件大于上下文窗口大小（例如，在 `/v1/messages` 请求中使用 500 MB 的纯文本文件）
* **无效的文件名（400）：** 文件名不符合长度要求（1-255 个字符）或包含禁用字符（`<`、`>`、`:`、`"`、`|`、`?`、`*`、`\`、`/` 或 Unicode 字符 0-31）
* **文件过大（413）：** 文件超过 500 MB 限制
* **超出存储限制（400）：** 您的组织已达到 1 TB 存储限制

```json Output
{
  "type": "error",
  "error": {
    "type": "not_found_error",
    "message": "File `file_011CNha8iCJcU1wXNR6q4V8w` not found."
  },
  "request_id": "req_011CQFYcrRp7mCHLDsAYT8Qt"
}
```

## 使用和计费

Files API 操作是免费的：

* 上传文件
* 下载文件
* 列出文件
* 获取文件元数据
* 删除文件

在 Messages 请求中使用的文件内容按输入令牌计费。

### 速率限制

与文件相关的 API 调用限制为每分钟约 500 个请求。如需申请更高的限制，请[联系销售](mailto:sales@anthropic.com)。

## 后续步骤

<CardGroup cols={3}>
  <Card title="PDF 支持" icon="file" href="https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support">
    使用 Claude 处理 PDF。从您的文档中提取文本、分析图表并理解视觉内容。
  </Card>

  <Card title="代码执行工具" icon="terminal" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool">
    在沙盒容器中运行 Python 和 bash 代码，以分析数据、生成文件并迭代解决方案。
  </Card>

  <Card title="视觉" icon="image" href="https://platform.claude.com/docs/zh-CN/build-with-claude/vision">
    处理和分析视觉输入，并从图像生成文本和代码。
  </Card>
</CardGroup>
