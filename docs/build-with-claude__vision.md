---
title: 视觉
url: https://platform.claude.com/docs/zh-CN/build-with-claude/vision
description: Claude 的视觉能力使其能够理解和分析图像，为多模态交互开辟了令人兴奋的可能性。
---

本指南介绍如何向 Claude 发送图像、适用的限制和费用，以及在哪里可以找到[基于坐标的工作流](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates)的相关指导。

***

## 向 Claude 发送图像

您可以通过以下方式使用 Claude 的视觉能力：

* [claude.ai](https://claude.ai/)。像上传文件一样上传图像，或直接将图像拖放到聊天窗口中。
* Claude Console 中的 [Playground](https://platform.claude.com/playground)。直接将图像添加到任意 User 消息块中。
* API 请求。请参阅以下示例。

在 API 中，使用以下三种来源类型之一，将图像作为 `image` 内容块提供给 Claude：

1. 嵌入在请求正文中的 base64 编码图像
2. 指向在线托管图像的 URL 引用
3. 由 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 返回的 `file_id`（上传一次，多次引用）

<Note>
  在 Amazon Bedrock 和 Google Cloud 上，目前仅支持 base64 编码的来源。
</Note>

<Tip>
  正如[将长文档放在查询之前](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#long-context-prompting)可以改善文本提示的效果一样，当图像位于文本之前时，Claude 的表现最佳。放在文本之后或与文本穿插的图像仍然表现良好，但如果您的用例允许，请优先采用"先图像后文本"的结构。
</Tip>

### Base64 编码图像示例

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
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
            "type": "image",
            "source": {
              "type": "base64",
              "media_type": "image/jpeg",
              "data": "$BASE64_IMAGE_DATA"
            }
          },
          {
            "type": "text",
            "text": "Describe this image."
          }
        ]
      }
    ]
  }
  EOF
  ```

  ```bash CLI
  curl -sSo ./vision-example.jpg \
    https://platform.claude.com/docs/images/vision-example.jpg

  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: image
          source:
            type: base64
            media_type: image/jpeg
            data: "@./vision-example.jpg"
        - type: text
          text: Describe this image.
  YAML
  ```

  ```python Python
  image1_data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC"
  image1_media_type = "image/png"

  client = anthropic.Anthropic()
  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "image",
                      "source": {
                          "type": "base64",
                          "media_type": image1_media_type,
                          "data": image1_data,
                      },
                  },
                  {"type": "text", "text": "Describe this image."},
              ],
          }
      ],
  )
  print(message)
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic();

  const message = await anthropic.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: {
              type: "base64",
              media_type: "image/jpeg",
              data: imageData // Base64-encoded image data as string
            }
          },
          {
            type: "text",
            text: "Describe this image."
          }
        ]
      }
    ]
  });

  console.log(message);
  ```

  ```csharp C#
  using System.Collections.Generic;
  using Anthropic;
  using Anthropic.Models.Messages;

  AnthropicClient client = new();

  string imageData = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC";

  var message = await client.Messages.Create(new MessageCreateParams
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
                  new ContentBlockParam(new ImageBlockParam(
                      new ImageBlockParamSource(new Base64ImageSource()
                      {
                          Data = imageData,
                          MediaType = MediaType.ImagePng,
                      })
                  )),
                  new ContentBlockParam(new TextBlockParam("Describe this image.")),
              }),
          }
      ]
  });

  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  imageData := "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC"

  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.NewImageBlockBase64("image/png", imageData),
  			anthropic.NewTextBlock("Describe this image."),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(message)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();
  String imageData =
    "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC";

  List<ContentBlockParam> contentBlockParams = List.of(
    ContentBlockParam.ofImage(
      ImageBlockParam.builder()
        .source(
          Base64ImageSource.builder()
            .mediaType(Base64ImageSource.MediaType.IMAGE_PNG)
            .data(imageData)
            .build()
        )
        .build()
    ),
    ContentBlockParam.ofText(TextBlockParam.builder().text("Describe this image.").build())
  );
  Message message = client
    .messages()
    .create(
      MessageCreateParams.builder()
        .model(Model.CLAUDE_OPUS_5)
        .maxTokens(1024)
        .addUserMessageOfBlockParams(contentBlockParams)
        .build()
    );

  IO.println(message);
  ```

  ```php PHP
  $client = new Client();

  $imageData = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC";

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'image',
                      'source' => [
                          'type' => 'base64',
                          'media_type' => 'image/png',
                          'data' => $imageData,
                      ],
                  ],
                  ['type' => 'text', 'text' => 'Describe this image.'],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo json_encode($message, JSON_PRETTY_PRINT), PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  image_data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC"

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: {
              type: "base64",
              media_type: "image/png",
              data: image_data
            }
          },
          { type: "text", text: "Describe this image." }
        ]
      }
    ]
  )

  puts message
  ```
</CodeGroup>

### 基于 URL 的图像示例

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [
        {
          "role": "user",
          "content": [
            {
              "type": "image",
              "source": {
                "type": "url",
                "url": "https://platform.claude.com/docs/images/vision-example.jpg"
              }
            },
            {
              "type": "text",
              "text": "Describe this image."
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
        - type: image
          source:
            type: url
            url: https://platform.claude.com/docs/images/vision-example.jpg
        - type: text
          text: Describe this image.
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
                      "type": "image",
                      "source": {
                          "type": "url",
                          "url": "https://platform.claude.com/docs/images/vision-example.jpg",
                      },
                  },
                  {"type": "text", "text": "Describe this image."},
              ],
          }
      ],
  )
  print(message)
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic();

  const message = await anthropic.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: {
              type: "url",
              url: "https://platform.claude.com/docs/images/vision-example.jpg"
            }
          },
          {
            type: "text",
            text: "Describe this image."
          }
        ]
      }
    ]
  });

  console.log(message);
  ```

  ```csharp C#
  using System.Collections.Generic;
  using Anthropic;
  using Anthropic.Models.Messages;

  AnthropicClient client = new();

  var message = await client.Messages.Create(new MessageCreateParams
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
                  new ContentBlockParam(new ImageBlockParam(
                      new ImageBlockParamSource(new UrlImageSource()
                      {
                          Url = "https://platform.claude.com/docs/images/vision-example.jpg",
                      })
                  )),
                  new ContentBlockParam(new TextBlockParam("Describe this image.")),
              }),
          }
      ]
  });

  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.NewImageBlock(anthropic.URLImageSourceParam{
  				URL: "https://platform.claude.com/docs/images/vision-example.jpg",
  			}),
  			anthropic.NewTextBlock("Describe this image."),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(message)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  List<ContentBlockParam> contentBlockParams = List.of(
    ContentBlockParam.ofImage(
      ImageBlockParam.builder()
        .source(
          UrlImageSource.builder()
            .url("https://platform.claude.com/docs/images/vision-example.jpg")
            .build()
        )
        .build()
    ),
    ContentBlockParam.ofText(TextBlockParam.builder().text("Describe this image.").build())
  );
  Message message = client
    .messages()
    .create(
      MessageCreateParams.builder()
        .model(Model.CLAUDE_OPUS_5)
        .maxTokens(1024)
        .addUserMessageOfBlockParams(contentBlockParams)
        .build()
    );
  System.out.println(message);
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
                      'type' => 'image',
                      'source' => [
                          'type' => 'url',
                          'url' => 'https://platform.claude.com/docs/images/vision-example.jpg',
                      ],
                  ],
                  ['type' => 'text', 'text' => 'Describe this image.'],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo json_encode($message, JSON_PRETTY_PRINT), PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: {
              type: "url",
              url: "https://platform.claude.com/docs/images/vision-example.jpg"
            }
          },
          { type: "text", text: "Describe this image." }
        ]
      }
    ]
  )

  puts message
  ```
</CodeGroup>

### Files API 图像示例

对于您将重复使用的图像，或者当您希望避免编码开销时，请使用 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)。上传图像一次，然后在后续消息中引用返回的 `file_id`，而无需重新发送 base64 数据。

<Tip>
  在多轮对话和智能体工作流中，每个请求都会重新发送完整的对话历史。 如果图像采用 base64 编码，则每一轮的请求负载中都会包含完整的图像字节， 随着对话的增长，这可能会显著增加请求大小和 "latency"（延迟）。 将图像上传到 Files API 并通过 `file_id` 引用它们， 无论对话历史中累积了多少图像，都能使请求负载保持较小。
</Tip>

<CodeGroup>
  ```bash cURL
  # 首先，将您的图像上传到 Files API
  FILE_ID=$(curl -sS -X POST https://api.anthropic.com/v1/files \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -F "file=@vision-example.jpg" | jq -r '.id')

  # 然后在您的消息中使用返回的 file_id
  curl https://api.anthropic.com/v1/messages \
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
            "type": "image",
            "source": {
              "type": "file",
              "file_id": "$FILE_ID"
            }
          },
          {
            "type": "text",
            "text": "Describe this image."
          }
        ]
      }
    ]
  }
  EOF
  ```

  ```bash CLI
  curl -sSo vision-example.jpg \
    https://platform.claude.com/docs/images/vision-example.jpg

  # 首先，将您的图像上传到 Files API
  FILE_ID=$(ant files upload \
    --file ./vision-example.jpg \
    --transform id --raw-output)

  # 然后在您的消息中使用返回的 file_id
  ant messages create \
    --transform content --format yaml <<YAML
  model: claude-opus-5
  max_tokens: 1024
  messages:
    - role: user
      content:
        - type: image
          source:
            type: file
            file_id: $FILE_ID
        - type: text
          text: Describe this image.
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  # 上传图像文件
  with open("vision-example.jpg", "rb") as f:
      file_upload = client.files.upload(file=("vision-example.jpg", f, "image/jpeg"))

  # 在消息中使用已上传的文件
  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {
                      "type": "image",
                      "source": {"type": "file", "file_id": file_upload.id},
                  },
                  {"type": "text", "text": "Describe this image."},
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

  // 上传图像文件
  const fileUpload = await anthropic.files.upload({
    file: await toFile(fs.createReadStream("vision-example.jpg"), undefined, {
      type: "image/jpeg"
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
            type: "image",
            source: {
              type: "file",
              file_id: fileUpload.id
            }
          },
          {
            type: "text",
            text: "Describe this image."
          }
        ]
      }
    ]
  });

  console.log(response);
  ```

  ```csharp C#
  using System.Collections.Generic;
  using Anthropic;
  using Anthropic.Core;
  using Anthropic.Models.Files;
  using Anthropic.Models.Messages;

  AnthropicClient client = new();

  // 上传图像文件
  var fileUpload = await client.Files.Upload(new FileUploadParams
  {
      File = new BinaryContent
      {
          Stream = File.OpenRead("vision-example.jpg"),
          FileName = "vision-example.jpg",
          ContentType = new("image/jpeg"),
      },
  });

  // 在消息中使用已上传的文件
  var response = await client.Messages.Create(new MessageCreateParams
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
                  new ContentBlockParam(new ImageBlockParam(
                      new ImageBlockParamSource(new FileImageSource(fileUpload.ID))
                  )),
                  new ContentBlockParam(new TextBlockParam("Describe this image.")),
              }),
          }
      ]
  });

  Console.WriteLine(response);
  ```

  ```go Go
  client := anthropic.NewClient()

  // 上传图像文件
  file, err := os.Open("vision-example.jpg")
  if err != nil {
  	log.Fatal(err)
  }
  defer file.Close()

  fileUpload, err := client.Files.Upload(context.Background(),
  	anthropic.FileUploadParams{
  		File: anthropic.File(file, "vision-example.jpg", "image/jpeg"),
  	})
  if err != nil {
  	log.Fatal(err)
  }

  // 在消息中使用已上传的文件
  message, err := client.Messages.New(context.Background(),
  	anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeOpus5,
  		MaxTokens: 1024,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(
  				anthropic.NewImageBlock(anthropic.FileImageSourceParam{
  					FileID: fileUpload.ID,
  				}),
  				anthropic.NewTextBlock("Describe this image."),
  			),
  		},
  	})
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(message.Content)
  ```

  ```java Java
  import com.anthropic.core.MultipartField;
  import com.anthropic.models.files.FileMetadata;
  import com.anthropic.models.files.FileUploadParams;
  // ...
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      // 上传图像文件
      FileMetadata file = client.files().upload(
        FileUploadParams.builder()
          .file(
            MultipartField.<InputStream>builder()
              .value(Files.newInputStream(Path.of("vision-example.jpg")))
              .filename("vision-example.jpg")
              .contentType("image/jpeg")
              .build()
          )
          .build()
      );

      // 在消息中使用已上传的文件
      ImageBlockParam imageParam = ImageBlockParam.builder().fileSource(file.id()).build();

      MessageCreateParams params = MessageCreateParams.builder()
        .model(Model.CLAUDE_OPUS_5)
        .maxTokens(1024)
        .addUserMessageOfBlockParams(
          List.of(
            ContentBlockParam.ofImage(imageParam),
            ContentBlockParam.ofText(
              TextBlockParam.builder().text("Describe this image.").build()
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

  // 上传图像文件
  $fileUpload = $client->files->upload(
      file: FileParam::fromResource(fopen('vision-example.jpg', 'rb'), contentType: 'image/jpeg'),
  );

  // 在消息中使用已上传的文件
  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  [
                      'type' => 'image',
                      'source' => ['type' => 'file', 'fileID' => $fileUpload->id],
                  ],
                  ['type' => 'text', 'text' => 'Describe this image.'],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo json_encode($message, JSON_PRETTY_PRINT), PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 上传图像文件
  file_upload = client.files.upload(
    file: Anthropic::FilePart.new(
      File.open("vision-example.jpg", "rb"),
      content_type: "image/jpeg"
    )
  )

  # 在消息中使用已上传的文件
  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: { type: "file", file_id: file_upload.id }
          },
          { type: "text", text: "Describe this image." }
        ]
      }
    ]
  )

  puts message.content
  ```
</CodeGroup>

有关更多示例代码和参数详情，请参阅 [Messages API 示例](https://platform.claude.com/docs/zh-CN/api/messages/create)。

### 多张图像

您可以在单个请求中包含多张图像，Claude 会对它们进行联合分析。这对于比较图像、询问差异或处理一系列图像（例如文档的各个页面）非常有用。发送多张图像时，请用简短的文本标签（`Image 1:`、`Image 2:` 等）引入每张图像，以便您可以在提示和后续轮次中按名称引用它们。

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [
        {
          "role": "user",
          "content": [
            {
              "type": "text",
              "text": "Image 1:"
            },
            {
              "type": "image",
              "source": {
                "type": "base64",
                "media_type": "image/png",
                "data": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC"
              }
            },
            {
              "type": "text",
              "text": "Image 2:"
            },
            {
              "type": "image",
              "source": {
                "type": "base64",
                "media_type": "image/png",
                "data": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGNgYPgPAAEDAQAIicLsAAAAAElFTkSuQmCC"
              }
            },
            {
              "type": "text",
              "text": "How are these images different?"
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
        - type: text
          text: "Image 1:"
        - type: image
          source:
            type: base64
            media_type: image/png
            data: iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC
        - type: text
          text: "Image 2:"
        - type: image
          source:
            type: base64
            media_type: image/png
            data: iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGNgYPgPAAEDAQAIicLsAAAAAElFTkSuQmCC
        - type: text
          text: How are these images different?
  YAML
  ```

  ```python Python
  image1_data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC"
  image2_data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGNgYPgPAAEDAQAIicLsAAAAAElFTkSuQmCC"

  client = anthropic.Anthropic()
  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[
          {
              "role": "user",
              "content": [
                  {"type": "text", "text": "Image 1:"},
                  {
                      "type": "image",
                      "source": {
                          "type": "base64",
                          "media_type": "image/png",
                          "data": image1_data,
                      },
                  },
                  {"type": "text", "text": "Image 2:"},
                  {
                      "type": "image",
                      "source": {
                          "type": "base64",
                          "media_type": "image/png",
                          "data": image2_data,
                      },
                  },
                  {"type": "text", "text": "How are these images different?"},
              ],
          }
      ],
  )
  print(message)
  ```

  ```typescript TypeScript
  const anthropic = new Anthropic();

  const image1Data =
    "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC";
  const image2Data =
    "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGNgYPgPAAEDAQAIicLsAAAAAElFTkSuQmCC";

  const message = await anthropic.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: "Image 1:"
          },
          {
            type: "image",
            source: {
              type: "base64",
              media_type: "image/png",
              data: image1Data
            }
          },
          {
            type: "text",
            text: "Image 2:"
          },
          {
            type: "image",
            source: {
              type: "base64",
              media_type: "image/png",
              data: image2Data
            }
          },
          {
            type: "text",
            text: "How are these images different?"
          }
        ]
      }
    ]
  });

  console.log(message);
  ```

  ```csharp C#
  AnthropicClient client = new();

  string image1Data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC";
  string image2Data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGNgYPgPAAEDAQAIicLsAAAAAElFTkSuQmCC";

  var message = await client.Messages.Create(new MessageCreateParams
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
                  new ContentBlockParam(new TextBlockParam("Image 1:")),
                  new ContentBlockParam(new ImageBlockParam(
                      new ImageBlockParamSource(new Base64ImageSource()
                      {
                          Data = image1Data,
                          MediaType = MediaType.ImagePng,
                      })
                  )),
                  new ContentBlockParam(new TextBlockParam("Image 2:")),
                  new ContentBlockParam(new ImageBlockParam(
                      new ImageBlockParamSource(new Base64ImageSource()
                      {
                          Data = image2Data,
                          MediaType = MediaType.ImagePng,
                      })
                  )),
                  new ContentBlockParam(new TextBlockParam("How are these images different?")),
              }),
          }
      ]
  });

  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  image1Data := "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC"
  image2Data := "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGNgYPgPAAEDAQAIicLsAAAAAElFTkSuQmCC"

  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(
  			anthropic.NewTextBlock("Image 1:"),
  			anthropic.NewImageBlockBase64("image/png", image1Data),
  			anthropic.NewTextBlock("Image 2:"),
  			anthropic.NewImageBlockBase64("image/png", image2Data),
  			anthropic.NewTextBlock("How are these images different?"),
  		),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(message)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  String image1Data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC";
  String image2Data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGNgYPgPAAEDAQAIicLsAAAAAElFTkSuQmCC";

  List<ContentBlockParam> contentBlockParams = List.of(
      ContentBlockParam.ofText(TextBlockParam.builder().text("Image 1:").build()),
      ContentBlockParam.ofImage(
          ImageBlockParam.builder()
              .source(
                  Base64ImageSource.builder()
                      .mediaType(Base64ImageSource.MediaType.IMAGE_PNG)
                      .data(image1Data)
                      .build()
              )
              .build()
      ),
      ContentBlockParam.ofText(TextBlockParam.builder().text("Image 2:").build()),
      ContentBlockParam.ofImage(
          ImageBlockParam.builder()
              .source(
                  Base64ImageSource.builder()
                      .mediaType(Base64ImageSource.MediaType.IMAGE_PNG)
                      .data(image2Data)
                      .build()
              )
              .build()
      ),
      ContentBlockParam.ofText(
          TextBlockParam.builder().text("How are these images different?").build()
      )
  );

  Message message = client
      .messages()
      .create(
          MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5)
              .maxTokens(1024)
              .addUserMessageOfBlockParams(contentBlockParams)
              .build()
      );

  IO.println(message);
  ```

  ```php PHP
  $client = new Client();

  $image1Data = 'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC';
  $image2Data = 'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGNgYPgPAAEDAQAIicLsAAAAAElFTkSuQmCC';

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          [
              'role' => 'user',
              'content' => [
                  ['type' => 'text', 'text' => 'Image 1:'],
                  [
                      'type' => 'image',
                      'source' => [
                          'type' => 'base64',
                          'media_type' => 'image/png',
                          'data' => $image1Data,
                      ],
                  ],
                  ['type' => 'text', 'text' => 'Image 2:'],
                  [
                      'type' => 'image',
                      'source' => [
                          'type' => 'base64',
                          'media_type' => 'image/png',
                          'data' => $image2Data,
                      ],
                  ],
                  ['type' => 'text', 'text' => 'How are these images different?'],
              ],
          ],
      ],
      model: 'claude-opus-5',
  );

  echo $message;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  image1_data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGP4z8AAAAMBAQDJ/pLvAAAAAElFTkSuQmCC"
  image2_data = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVR4nGNgYPgPAAEDAQAIicLsAAAAAElFTkSuQmCC"

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          { type: "text", text: "Image 1:" },
          {
            type: "image",
            source: {
              type: "base64",
              media_type: "image/png",
              data: image1_data
            }
          },
          { type: "text", text: "Image 2:" },
          {
            type: "image",
            source: {
              type: "base64",
              media_type: "image/png",
              data: image2_data
            }
          },
          { type: "text", text: "How are these images different?" }
        ]
      }
    ]
  )

  puts message
  ```
</CodeGroup>

在多轮对话中，以同样的方式在后续的 `user` 轮次中添加新图像。Claude 可以访问先前轮次中的每张图像，因此诸如"这些与前两张相似吗？"之类的后续问题无需在新轮次的内容中再次包含先前的图像即可正常工作。

***

## 图像限制和费用

### 请求限制

每条消息或每个请求的最大图像数量为：

* 在 [claude.ai](https://claude.ai/) 上每条消息 20 张。
* 在 API 上，对于具有 200k 令牌 "context window"（上下文窗口）的模型，每个请求 100 张。
* 在 API 上，对于所有其他模型，每个请求 600 张。

每张图像的最大尺寸为 8000x8000 像素。

如果单个 API 请求包含超过 20 张图像，则该请求中的每张图像都将适用更严格的单图尺寸限制。请求中的所有 `image` 块都计入此阈值，包括您重新发送的先前对话轮次中的图像，以及嵌套在 `tool_result` 内容中的图像（例如，返回给计算机使用工具的屏幕截图）。在 Amazon Bedrock 和 Google Cloud 上，PDF 等文档块也计入此阈值。超过更严格限制的图像将被拒绝，并返回 `invalid_request_error`，其消息中会提及 "many-image requests" 并以像素为单位说明当前限制。要在所有平台上保持在限制之内，请将每张图像调整为任一边均不超过 2000 像素，或将请求中的图像和文档块保持在 20 个或更少。

每张图像的最大大小为：

* 直接使用 Claude API 时为 10 MB（base64 编码）。
* 在 Amazon Bedrock 和 Google Cloud 上为 5 MB（base64 编码）。
* 在 [claude.ai](https://claude.ai/) 上为 10 MB。

<Note>
  尽管 API 支持每个请求最多 600 张图像，但可能会先达到[请求大小限制](https://platform.claude.com/docs/zh-CN/api/overview#request-size-limits)（标准端点为 32 MB；在某些合作伙伴运营的平台上更低，例如 Amazon Bedrock 和 Google Cloud）。对于大量图像，请考虑使用 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#files-api-image-example) 上传并通过 `file_id` 引用，以保持请求负载较小。

  即使使用 Files API，包含许多大图像的请求也可能在达到 600 张图像数量之前失败。请在上传之前减小图像尺寸或文件大小（例如，通过降采样）（请参阅[分辨率和令牌费用](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#evaluate-image-size)）。
</Note>

### 支持的格式

Claude 支持 JPEG、PNG、GIF 和 WebP 图像（`image/jpeg`、`image/png`、`image/gif`、`image/webp`）。不支持动画，仅使用第一帧。

### 分辨率和令牌费用

Claude 以图块（patch）而非像素的方式查看图像。每个图块是图像中一个 28×28 像素的块，称为 "visual token"（视觉令牌）。因此，一张图像消耗 `⌈width / 28⌉ × ⌈height / 28⌉` 个视觉令牌。

每个模型都有一个最大原生图像分辨率，以长边限制和视觉令牌限制来表示。超过任一限制的图像在处理前会被缩小；有关确切规则，请参阅 [Claude 如何调整图像大小和填充图像](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#how-claude-resizes-and-pads-images)。例外情况是您返回给[计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#handle-coordinate-scaling-for-higher-resolutions)和[浏览器使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool#targets-and-coordinates)工具集的屏幕截图和缩放图像：对于超过模型限制的 `tool_result` 图像，API 会以验证错误拒绝它，而不是将其缩小，因此请在返回这些图像之前在您的应用程序中调整其大小。若要让任何其他超大图像以错误形式被拒绝而不是被缩小，请设置图像块的 [`transformations` 字段](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#oversized-image-error)。

| 分辨率层级 | 模型                  | 最大长边    | 最大视觉令牌数 |
| ----- | ------------------- | ------- | ------- |
| 高分辨率  | Claude 4.7 及更高版本的模型 | 2576 px | 4784    |
| 标准    | 所有其他模型              | 1568 px | 1568    |

高分辨率支持在所列模型上自动启用，无需 beta 标头或客户端选择加入。

下表显示了每个层级上若干图像尺寸的缩小后分辨率和视觉令牌费用：

| 图像尺寸                    | 标准层级：缩小至    | 标准层级：令牌数 | 高分辨率层级：缩小至   | 高分辨率层级：令牌数 |
| ----------------------- | ----------- | -------- | ------------ | ---------- |
| 200x200 px（0.04 百万像素）   | 不调整大小       | 64       | 不调整大小        | 64         |
| 1000x1000 px（1 百万像素）    | 不调整大小       | 1296     | 不调整大小        | 1296       |
| 1092x1092 px（1.19 百万像素） | 不调整大小       | 1521     | 不调整大小        | 1521       |
| 1920x1080 px（2.07 百万像素） | 1456x819 px | 1560     | 不调整大小        | 2691       |
| 2000x1500 px（3 百万像素）    | 1269x952 px | 1564     | 不调整大小        | 3888       |
| 3840x2160 px（8.29 百万像素） | 1456x819 px | 1560     | 2576x1449 px | 4784       |

当图像被缩小时，Claude 会在保持其宽高比的同时，将其缩放到符合该层级限制的最大尺寸。这为令牌费用设定了上限。有关精确规则和参考实现，请参阅 [Claude 如何调整图像大小和填充图像](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#how-claude-resizes-and-pads-images)。

要估算费用，请将令牌数乘以您所使用[模型的每令牌价格](https://claude.com/pricing)。例如，按 Claude Haiku 4.5 每百万输入令牌 $1 美元（标准层级）计算，1000×1000 的图像每千张约花费 $1.30 美元。按 Claude Opus 5 每百万 $5 美元（高分辨率层级）计算，同一图像每千张约花费 $6.48 美元，而 4K 图像每千张约花费 $23.92 美元。

与标准层级模型上的同一图像相比，高分辨率图像最多可能使用大约三倍的视觉令牌。如果您不需要高分辨率为计算机使用、屏幕截图理解和密集文档所提供的额外保真度，请在发送前对图像进行降采样以控制令牌费用。为了最大限度地减少延迟并简化[基于坐标的工作流](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates)，请优先在上传图像之前调整其大小。

### 图像质量指导

向 Claude 提供图像时，请牢记以下几点以获得最佳效果：

* **图像清晰度：** 确保图像清晰，不要过于模糊或像素化。
* **文本：** 如果图像包含重要文本，请确保其清晰可读且不要太小。避免仅为了放大文本而裁剪掉关键的视觉上下文。
* **调整大小：** 请考虑到如果您的图像过大，可能会被调整大小（请参阅[分辨率和令牌费用](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#evaluate-image-size)）；例如，这可能会使文本变得不太清晰。请考虑预先调整图像大小、裁剪图像，或两者兼用。若要让超大图像以错误形式被拒绝而不是被调整大小（这对[坐标工作流](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates)很重要），请使用 [`"oversized_image": "error"`](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates#oversized-image-error) 标记图像块。
* **图像压缩：** 在发送图像之前使用有损格式（例如 JPEG 或 WebP（有损模式））对其进行压缩，可以通过减小请求大小来降低延迟。但是，这可能会引入对模型性能有害的伪影，尤其是在应用多次压缩时。例如，重度 JPEG 压缩可能会使文本难以阅读。请通过检查实际发送到 API 的图像来确认您的压缩设置适合该任务。

***

## 坐标和边界框

有关边界框、点和像素坐标，请参阅[坐标和边界框](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates)。Claude 返回的是相对于其在调整大小后所看到的图像的绝对像素坐标；该指南介绍了 Claude 如何调整图像大小和填充图像，以及如何预先调整大小或重新缩放，以使坐标与您的原始图像对齐。

***

## 局限性

尽管 Claude 的图像理解能力处于前沿水平，但仍有一些局限性需要注意：

* **人物识别：** Claude [不能用于](https://www.anthropic.com/legal/aup)指认图像中人物的姓名，并且会拒绝这样做。
* **准确性：** 在解读低质量、旋转过的或小于 200 像素的极小图像时，Claude 可能会产生幻觉或出错。
* **空间推理：** Claude 的坐标和定位输出是近似的。请遵循[坐标和边界框](https://platform.claude.com/docs/zh-CN/build-with-claude/vision-coordinates)中的指导，并在依赖输出之前对其进行验证。
* **计数：** Claude 可以给出图像中物体的大致数量，但可能并不总是精确无误，尤其是在存在大量小物体的情况下。
* **AI 生成的图像：** Claude 无法判断图像是否由 AI 生成，如果被问及，可能会给出错误答案。请勿依赖它来检测伪造或合成图像。
* **不当内容：** Claude 不会处理违反[可接受使用政策](https://www.anthropic.com/legal/aup)的不当或露骨图像。
* **医疗保健应用：** 尽管 Claude 可以分析一般的医学图像，但它并非为解读 CT 或 MRI 等复杂诊断扫描而设计。Claude 的输出不应被视为专业医疗建议或诊断的替代品。

请始终仔细审查和验证 Claude 的图像解读，尤其是在高风险用例中。在没有人工监督的情况下，请勿将 Claude 用于需要完美精度或敏感图像分析的任务。

***

## 常见问题

<AccordionGroup>
  <Accordion title="Claude 支持哪些图像文件类型？">
    JPEG、PNG、GIF 和 WebP。请参阅[支持的格式](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#supported-formats)。
  </Accordion>

  <Accordion title="Claude 可以读取图像 URL 吗？">
    可以。在 `image` 内容块中使用 `url` 来源类型代替 `base64`。请参阅[基于 URL 的图像示例](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#url-based-image-example)。
  </Accordion>

  <Accordion title="我可以上传的图像文件大小有限制吗？">
    有。请参阅[请求限制](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#request-limits)，了解 Claude API、Amazon Bedrock、Google Cloud 和 claude.ai 上的单图限制和整体请求大小限制。
  </Accordion>

  <Accordion title="一个请求中可以包含多少张图像？">
    每个 API 请求最多 600 张（对于具有 200k 令牌上下文窗口的模型为 100 张），在 claude.ai 上每轮最多 20 张。有关详细信息以及超过 20 张图像时适用的更低单图尺寸限制，请参阅[请求限制](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#request-limits)。
  </Accordion>

  <Accordion title="Claude 会读取图像元数据吗？">
    不会，Claude 不会解析或接收传递给它的图像中的任何元数据。
  </Accordion>

  <Accordion title="我可以删除已上传的图像吗？">
    不可以。图像上传是临时性的，不会在 API 请求持续时间之外存储。 上传的图像在处理完成后会自动删除。
  </Accordion>

  <Accordion title="在哪里可以找到有关图像上传数据隐私的详细信息？">
    有关如何处理上传的图像和其他数据的信息，请参阅 Anthropic 隐私政策页面。 Anthropic 不会使用上传的图像来训练模型。
  </Accordion>

  <Accordion title="如果 Claude 的图像解读似乎有误怎么办？">
    如果 Claude 的图像解读似乎不正确：

    1. 确保图像清晰、高质量且方向正确。
    2. 尝试使用提示工程技术来改善结果。
    3. 如果问题仍然存在，请在 claude.ai 中标记输出（点赞/点踩）或联系[支持团队](https://support.claude.com/)。

    您的反馈有助于改进 Claude！
  </Accordion>

  <Accordion title="Claude 可以生成或编辑图像吗？">
    不可以，Claude 仅是一个图像理解模型。它可以解读和分析图像，但不能生成、制作、编辑、操纵或创建图像。
  </Accordion>
</AccordionGroup>

***

## 后续步骤

<CardGroup cols={2}>
  <Card title="多模态 cookbook" icon="image" href="https://platform.claude.com/cookbook/multimodal-getting-started-with-vision">
    获取有关解读图表和从表单中提取内容等任务的技巧和最佳实践方法。
  </Card>

  <Card title="API 参考" icon="code" href="https://platform.claude.com/docs/zh-CN/api/messages/create">
    请参阅 Messages API 文档，包括涉及图像的 API 调用示例。
  </Card>
</CardGroup>
