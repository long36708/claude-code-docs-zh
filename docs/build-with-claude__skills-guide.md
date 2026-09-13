---
title: 通过 API 使用 Agent Skills
url: https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide
description: 了解如何通过 API 使用 Agent Skills 扩展 Claude 的能力。
---

Agent Skills 通过有组织的指令、脚本和资源文件夹来扩展 Claude 的能力。本指南向您展示如何在 Claude API 中使用预构建 Skills 和自定义 Skills。

<Note>
  有关完整的 API 参考（包括请求/响应模式和所有参数），请参阅：

  * [Skill 管理 API 参考](https://platform.claude.com/docs/zh-CN/api/skills/list) - Skills 的 CRUD 操作
  * [Skill 版本 API 参考](https://platform.claude.com/docs/zh-CN/api/skills/versions/list) - 版本管理
</Note>

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

## 快速链接

<CardGroup cols={2}>
  <Card title="在 API 中开始使用 Agent Skills" icon="rocket" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/quickstart">
    了解如何在 10 分钟内使用 Agent Skills 通过 Claude API 创建文档。
  </Card>

  <Card title="Skill 编写最佳实践" icon="hammer" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices">
    了解如何编写 Claude 能够发现并成功使用的高效 Skills。
  </Card>
</CardGroup>

## 概述

<Note>
  如需深入了解 Agent Skills 的架构和实际应用，请阅读工程博客文章：[Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)。
</Note>

Skills 通过 [code execution tool（代码执行工具）](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool) 与 Messages API 集成。无论是使用由 Anthropic 管理的预构建 Skills，还是您上传的自定义 Skills，集成方式都完全相同：两者都需要代码执行，并使用相同的 `container` 结构。

### 使用 Skills

无论来源如何，Skills 在 Messages API 中的集成方式都相同。您在 `container` 参数中通过 `skill_id`、`type` 和可选的 `version` 指定 Skills，它们会在代码执行环境中运行。

您可以使用来自两种来源的 Skills：

| 方面           | Anthropic Skills               | 自定义 Skills                                                                      |
| ------------ | ------------------------------ | ------------------------------------------------------------------------------- |
| **Type 值**   | `anthropic`                    | `custom`                                                                        |
| **Skill ID** | 短名称：`pptx`、`xlsx`、`docx`、`pdf` | 自动生成：`skill_01AbCdEfGhIjKlMnOpQrStUv`                                           |
| **版本格式**     | 基于日期：`20251013` 或 `latest`     | 版本 ID：`skver_01AbCdEfGhIjKlMnOpQrStUv` 或 `latest`                               |
| **管理方式**     | 由 Anthropic 预构建并维护             | 通过 [Skills API](https://platform.claude.com/docs/zh-CN/api/skills/create) 上传和管理 |
| **可用性**      | 对所有用户可用                        | 仅限您的工作区私有                                                                       |

两种来源的 Skills 都会由 [List Skills 端点](https://platform.claude.com/docs/zh-CN/api/skills/list) 返回（使用 `source` 参数进行筛选）。集成方式和执行环境完全相同。唯一的区别在于 Skills 的来源以及管理方式。

### 前提条件

要使用 Skills，您需要：

1. 来自 [Claude Console](https://platform.claude.com/settings/keys) 的 **Claude API 密钥**
2. 在请求中启用 **[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)**

Skills 需要代码执行工具，因此请使用其[模型兼容性列表](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#compatibility)中的模型。

***

## 在 Messages 中使用 Skills

### Container 参数

Skills 通过 Messages API 中的 `container` 参数指定。每个请求最多可以包含 20 个 Skills。

Anthropic Skills 和自定义 Skills 的结构完全相同。指定必需的 `type` 和 `skill_id`，并可选择包含 `version` 以固定到特定版本：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "container": {
        "skills": [
          {
            "type": "anthropic",
            "skill_id": "pptx",
            "version": "latest"
          }
        ]
      },
      "messages": [{
        "role": "user",
        "content": "Create a presentation about renewable energy"
      }],
      "tools": [{
        "type": "code_execution_20250825",
        "name": "code_execution"
      }]
    }'
  ```

  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: anthropic
        skill_id: pptx
        version: latest
  messages:
    - role: user
      content: Create a presentation about renewable energy
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [{"type": "anthropic", "skill_id": "pptx", "version": "latest"}]
      },
      messages=[
          {"role": "user", "content": "Create a presentation about renewable energy"}
      ],
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        {
          type: "anthropic",
          skill_id: "pptx",
          version: "latest"
        }
      ]
    },
    messages: [
      {
        role: "user",
        content: "Create a presentation about renewable energy"
      }
    ],
    tools: [
      {
        type: "code_execution_20250825",
        name: "code_execution"
      }
    ]
  });
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "pptx",
                  Version = "latest",
              },
          ],
      },
      Messages = [new() { Role = Role.User, Content = "Create a presentation about renewable energy" }],
      Tools = [new CodeExecutionTool20250825()],
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "pptx",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Create a presentation about renewable energy")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;
  import com.anthropic.models.messages.CodeExecutionTool20250825;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .container(ContainerParams.builder()
              .addSkill(SkillParams.builder()
                  .type(SkillParams.Type.ANTHROPIC)
                  .skillId("pptx")
                  .version("latest")
                  .build())
              .build())
          .addUserMessage("Create a presentation about renewable energy")
          .addTool(CodeExecutionTool20250825.builder().build())
          .build();

      Message response = client.messages().create(params);
      System.out.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $message = $client->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Create a presentation about renewable energy']
      ],
      model: 'claude-opus-5',
      container: [
          'skills' => [
              [
                  'type' => 'anthropic',
                  'skillID' => 'pptx',
                  'version' => 'latest'
              ]
          ]
      ],
      tools: [
          ['type' => 'code_execution_20250825', 'name' => 'code_execution']
      ]
  );

  echo $message;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        {
          type: "anthropic",
          skill_id: "pptx",
          version: "latest"
        }
      ]
    },
    messages: [
      { role: "user", content: "Create a presentation about renewable energy" }
    ],
    tools: [
      { type: "code_execution_20250825", name: "code_execution" }
    ]
  )
  puts message
  ```
</CodeGroup>

### 下载生成的文件

当 Skills 创建文档（Excel、PowerPoint、PDF、Word）时，它们会在响应中返回 `file_id` 属性。您必须使用 Files API 下载这些文件。

**工作原理：**

1. Skills 在代码执行期间创建文件。
2. 响应在代码执行工具结果块中为每个创建的文件包含一个 `file_id`（请参阅[响应格式](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#response-format)）。
3. 使用 Files API 下载实际的文件内容。
4. 保存到本地或按需处理。

要为 Skills 提供待处理的输入文件，请[使用 Files API 上传它们](https://platform.claude.com/docs/zh-CN/build-with-claude/files#uploading-a-file)，并在请求中通过 [container upload 块](https://platform.claude.com/docs/zh-CN/build-with-claude/files#container-upload-blocks)引用它们。

**示例：创建并下载 Excel 文件**

<CodeGroup>
  ```bash cURL
  # 第 1 步：使用 Skill 创建文件
  RESPONSE=$(curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "container": {
        "skills": [
          {"type": "anthropic", "skill_id": "xlsx", "version": "latest"}
        ]
      },
      "messages": [{
        "role": "user",
        "content": "Create an Excel file with a simple budget spreadsheet"
      }],
      "tools": [{
        "type": "code_execution_20250825",
        "name": "code_execution"
      }]
    }')

  # 第 2 步：从响应中提取 file_id（使用 jq）
  FILE_ID=$(echo "$RESPONSE" | jq -r '.content[] | select(.type=="bash_code_execution_tool_result") | .content | select(.type=="bash_code_execution_result") | .content[] | select(.file_id) | .file_id')

  # 第 3 步：从元数据中获取文件名
  FILENAME=$(curl "https://api.anthropic.com/v1/files/$FILE_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" | jq -r '.filename')

  # 第 4 步：使用 Files API 下载文件
  curl "https://api.anthropic.com/v1/files/$FILE_ID/content" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    --output "$FILENAME"

  echo "Downloaded: $FILENAME"
  ```

  ```bash CLI
  # 第 1 步：使用 xlsx Skill 创建文件
  # 第 2 步：使用 --transform（GJSON 路径）从响应中提取 file_id
  FILE_ID=$(ant messages create \
    --transform 'content.#.content.content.#.file_id|@flatten|0' \
    --raw-output <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: anthropic
        skill_id: xlsx
        version: latest
  messages:
    - role: user
      content: Create an Excel file with a simple budget spreadsheet
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  )

  # 第 3 步：从文件元数据中获取文件名
  FILENAME=$(ant files retrieve-metadata \
    --file-id "$FILE_ID" \
    --transform filename \
    --raw-output)

  # 第 4 步：使用 Files API 下载文件
  ant files download --file-id "$FILE_ID" --output "$FILENAME" > /dev/null

  printf 'Downloaded: %s\n' "$FILENAME"
  ```

  ```python Python
  client = anthropic.Anthropic()

  # 第 1 步：使用 Skill 创建文件
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [{"type": "anthropic", "skill_id": "xlsx", "version": "latest"}]
      },
      messages=[
          {
              "role": "user",
              "content": "Create an Excel file with a simple budget spreadsheet",
          }
      ],
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )


  # 第 2 步：从响应中提取文件 ID
  def extract_file_ids(response):
      file_ids = []
      for item in response.content:
          if item.type == "bash_code_execution_tool_result":
              content_item = item.content
              if content_item.type == "bash_code_execution_result":
                  # 每个内容项都是一个携带 file_id 的 bash_code_execution_output 块
                  for file in content_item.content:
                      file_ids.append(file.file_id)
      return file_ids


  # 第 3 步：使用 Files API 下载文件
  for file_id in extract_file_ids(response):
      file_metadata = client.files.retrieve_metadata(file_id=file_id)
      file_content = client.files.download(file_id=file_id)

      # 第 4 步：保存到磁盘
      file_content.write_to_file(file_metadata.filename)
      print(f"Downloaded: {file_metadata.filename}")
  ```

  ```typescript TypeScript
  import { writeFile } from "node:fs/promises";

  const client = new Anthropic();

  // 第 1 步：使用 Skill 创建文件
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [{ type: "anthropic", skill_id: "xlsx", version: "latest" }]
    },
    messages: [
      {
        role: "user",
        content: "Create an Excel file with a simple budget spreadsheet"
      }
    ],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  });

  // 第 2 步：从响应中提取文件 ID
  const fileIds: string[] = [];
  for (const block of response.content) {
    if (
      block.type === "bash_code_execution_tool_result" &&
      block.content.type === "bash_code_execution_result"
    ) {
      for (const outputBlock of block.content.content) {
        fileIds.push(outputBlock.file_id);
      }
    }
  }

  // 第 3 步：下载每个文件并保存到磁盘
  for (const fileId of fileIds) {
    const fileMetadata = await client.files.retrieveMetadata(fileId);
    const fileResponse = await client.files.download(fileId);

    await writeFile(fileMetadata.filename, Buffer.from(await fileResponse.arrayBuffer()));
    console.log(`Downloaded: ${fileMetadata.filename}`);
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  // 第 1 步：使用 Skill 创建文件
  var parameters = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "xlsx",
                  Version = "latest",
              },
          ],
      },
      Messages = [new() { Role = Role.User, Content = "Create an Excel file with a simple budget spreadsheet" }],
      Tools = [new CodeExecutionTool20250825()],
  };

  var response = await client.Messages.Create(parameters);

  // 第 2 步：从响应中提取文件 ID
  List<string> fileIds = [];
  foreach (var block in response.Content)
  {
      if (block.TryPickBashCodeExecutionToolResult(out var toolResult)
          && toolResult.Content.TryPickBashCodeExecutionResultBlock(out var result))
      {
          foreach (var output in result.Content)
          {
              fileIds.Add(output.FileID);
          }
      }
  }

  // 第 3 步：下载每个文件并保存到磁盘
  foreach (var fileId in fileIds)
  {
      var fileMetadata = await client.Files.RetrieveMetadata(fileId);
      using var download = await client.Files.Download(fileId);
      using var downloadStream = await download.ReadAsStream();
      using var outputFile = File.Create(fileMetadata.Filename);
      await downloadStream.CopyToAsync(outputFile);
      Console.WriteLine($"Downloaded: {fileMetadata.Filename}");
  }
  ```

  ```go Go
  func main() {
  	client := anthropic.NewClient()

  	// 第 1 步：使用 Skill 创建文件
  	response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  		Model:     "claude-opus-5",
  		MaxTokens: 4096,
  		Container: anthropic.MessageCreateParamsContainerUnion{
  			OfContainers: &anthropic.ContainerParams{
  				Skills: []anthropic.SkillParams{
  					{
  						Type:    anthropic.SkillParamsTypeAnthropic,
  						SkillID: "xlsx",
  						Version: anthropic.String("latest"),
  					},
  				},
  			},
  		},
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Create an Excel file with a simple budget spreadsheet")),
  		},
  		Tools: []anthropic.ToolUnionParam{
  			{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  		},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	// 第 2 步：从响应中提取文件 ID
  	fileIDs := extractFileIDs(response)

  	// 第 3 步：使用 Files API 下载文件
  	for _, fileID := range fileIDs {
  		fileMetadata, err := client.Files.GetMetadata(context.TODO(), fileID)
  		if err != nil {
  			log.Fatal(err)
  		}

  		fileContent, err := client.Files.Download(context.TODO(), fileID)
  		if err != nil {
  			log.Fatal(err)
  		}

  		// 第 4 步：保存到磁盘
  		out, err := os.Create(fileMetadata.Filename)
  		if err != nil {
  			log.Fatal(err)
  		}
  		if _, err := io.Copy(out, fileContent.Body); err != nil {
  			log.Fatal(err)
  		}
  		out.Close()
  		fileContent.Body.Close()
  		fmt.Printf("Downloaded: %s\n", fileMetadata.Filename)
  	}
  }

  func extractFileIDs(response *anthropic.Message) []string {
  	var fileIDs []string
  	for _, item := range response.Content {
  		switch v := item.AsAny().(type) {
  		case anthropic.BashCodeExecutionToolResultBlock:
  			if v.Content.Type == "bash_code_execution_result" {
  				for _, output := range v.Content.Content {
  					fileIDs = append(fileIDs, output.FileID)
  				}
  			}
  		}
  	}
  	return fileIDs
  }
  ```

  ```java Java
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;
  import com.anthropic.models.messages.CodeExecutionTool20250825;
  import com.anthropic.models.messages.ContentBlock;
  import com.anthropic.models.files.FileMetadata;
  import com.anthropic.core.http.HttpResponse;
  // ...
  void main() throws Exception {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      // 第 1 步：使用 Skill 创建文件
      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .container(ContainerParams.builder()
              .addSkill(SkillParams.builder()
                  .type(SkillParams.Type.ANTHROPIC)
                  .skillId("xlsx")
                  .version("latest")
                  .build())
              .build())
          .addUserMessage("Create an Excel file with a simple budget spreadsheet")
          .addTool(CodeExecutionTool20250825.builder().build())
          .build();

      Message response = client.messages().create(params);

      // 第 2 步：从响应中提取文件 ID
      List<String> fileIds = new ArrayList<>();
      for (ContentBlock block : response.content()) {
          if (block.isBashCodeExecutionToolResult()) {
              var content = block.asBashCodeExecutionToolResult().content();
              if (content.isBashCodeExecutionResultBlock()) {
                  for (var outputBlock : content.asBashCodeExecutionResultBlock().content()) {
                      fileIds.add(outputBlock.fileId());
                  }
              }
          }
      }

      // 第 3 步：使用 Files API 下载文件
      for (String fileId : fileIds) {
          FileMetadata fileMetadata = client.files().retrieveMetadata(fileId);
          HttpResponse fileContent = client.files().download(fileId);

          // 第 4 步：保存到磁盘
          try (InputStream is = fileContent.body();
               FileOutputStream fos = new FileOutputStream(fileMetadata.filename())) {
              is.transferTo(fos);
          }
          System.out.println("Downloaded: " + fileMetadata.filename());
      }
  }
  ```

  ```php PHP
  $client = new Client();

  // 第 1 步：使用 Skill 创建文件
  $response = $client->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Create an Excel file with a simple budget spreadsheet']
      ],
      model: 'claude-opus-5',
      container: [
          'skills' => [
              ['type' => 'anthropic', 'skillID' => 'xlsx', 'version' => 'latest']
          ]
      ],
      tools: [
          ['type' => 'code_execution_20250825', 'name' => 'code_execution']
      ]
  );

  // 第 2 步：从响应中提取文件 ID
  function extractFileIds($response) {
      $fileIds = [];
      foreach ($response->content as $item) {
          if ($item->type === 'bash_code_execution_tool_result') {
              $contentItem = $item->content;
              if ($contentItem->type === 'bash_code_execution_result') {
                  foreach ($contentItem->content as $file) {
                      $fileIds[] = $file->fileID;
                  }
              }
          }
      }
      return $fileIds;
  }

  // 第 3 步：使用 Files API 下载文件
  foreach (extractFileIds($response) as $fileId) {
      $fileMetadata = $client->files->retrieveMetadata($fileId);
      $fileContent = $client->files->download($fileId);

      // 第 4 步：保存到磁盘
      file_put_contents($fileMetadata->filename, $fileContent);
      echo "Downloaded: {$fileMetadata->filename}\n";
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 第 1 步：使用 Skill 创建文件
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [{ type: "anthropic", skill_id: "xlsx", version: "latest" }]
    },
    messages: [
      {
        role: "user",
        content: "Create an Excel file with a simple budget spreadsheet"
      }
    ],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  )

  # 第 2 步：从响应中提取文件 ID
  def extract_file_ids(response)
    file_ids = []
    response.content.each do |item|
      if item.type == :bash_code_execution_tool_result
        content_item = item.content
        if content_item.type == :bash_code_execution_result
          content_item.content.each do |file|
            file_ids << file.file_id
          end
        end
      end
    end
    file_ids
  end

  # 第 3 步：使用 Files API 下载文件
  extract_file_ids(response).each do |file_id|
    file_metadata = client.files.retrieve_metadata(file_id)

    file_content = client.files.download(file_id)

    # 第 4 步：保存到磁盘
    File.binwrite(file_metadata.filename, file_content.read)
    puts "Downloaded: #{file_metadata.filename}"
  end
  ```
</CodeGroup>

**其他 Files API 操作：**

<CodeGroup>
  ```bash cURL
  # 获取文件元数据
  curl "https://api.anthropic.com/v1/files/$FILE_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"

  # 列出所有文件
  curl "https://api.anthropic.com/v1/files" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"

  # 删除文件
  curl -X DELETE "https://api.anthropic.com/v1/files/$FILE_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  # 获取文件元数据
  ant files retrieve-metadata \
    --file-id "$FILE_ID" \
    --transform '{filename,size_bytes}' \
    --format yaml

  # 列出所有文件
  ant files list --transform '{filename,created_at}' --format yaml

  # 删除文件
  ant files delete --file-id "$FILE_ID" >/dev/null
  ```

  ```python Python
  client = anthropic.Anthropic()
  file_id = "file_011CNha8iCJcU1wXNR6q4V8w"
  # 获取文件元数据
  file_info = client.files.retrieve_metadata(file_id=file_id)
  print(f"Filename: {file_info.filename}, Size: {file_info.size_bytes} bytes")

  # 列出所有文件
  for file in client.files.list():
      print(f"{file.filename} - {file.created_at}")

  # 删除文件
  client.files.delete(file_id=file_id)
  ```

  ```typescript TypeScript
  const client = new Anthropic();
  const fileId = "file_011CNha8iCJcU1wXNR6q4V8w";

  // 获取文件元数据
  const fileInfo = await client.files.retrieveMetadata(fileId);
  console.log(`Filename: ${fileInfo.filename}, Size: ${fileInfo.size_bytes} bytes`);

  // 列出所有文件
  for await (const file of client.files.list()) {
    console.log(`${file.filename} - ${file.created_at}`);
  }

  // 删除文件
  await client.files.delete(fileId);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var fileId = "file_011CNha8iCJcU1wXNR6q4V8w";

  // 获取文件元数据
  var fileInfo = await client.Files.RetrieveMetadata(fileId);
  Console.WriteLine($"Filename: {fileInfo.Filename}, Size: {fileInfo.SizeBytes} bytes");

  // 列出文件
  await foreach (var file in (await client.Files.List()).Paginate())
  {
      Console.WriteLine($"{file.Filename} - {file.CreatedAt}");
  }

  // 删除文件
  await client.Files.Delete(fileId);
  ```

  ```go Go
  client := anthropic.NewClient()
  fileID := "file_011CNha8iCJcU1wXNR6q4V8w"

  // 获取文件元数据
  fileInfo, err := client.Files.GetMetadata(context.TODO(), fileID)
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Printf("Filename: %s, Size: %d bytes\n", fileInfo.Filename, fileInfo.SizeBytes)

  // 列出所有文件
  files := client.Files.ListAutoPaging(context.TODO(), anthropic.FileListParams{})
  for files.Next() {
  	file := files.Current()
  	fmt.Printf("%s - %s\n", file.Filename, file.CreatedAt)
  }
  if files.Err() != nil {
  	log.Fatal(files.Err())
  }

  // 删除文件
  _, err = client.Files.Delete(context.TODO(), fileID)
  if err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.files.FileMetadata;
  import com.anthropic.models.files.FileListPage;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();
      String fileId = "file_011CNha8iCJcU1wXNR6q4V8w";

      // 获取文件元数据
      FileMetadata fileInfo = client.files().retrieveMetadata(fileId);
      System.out.println("Filename: " + fileInfo.filename() + ", Size: " + fileInfo.sizeBytes() + " bytes");

      // 列出文件（第一页）
      FileListPage files = client.files().list();
      for (var file : files.data()) {
          System.out.println(file.filename() + " - " + file.createdAt());
      }

      // 删除文件
      client.files().delete(fileId);
  }
  ```

  ```php PHP
  $client = new Client();
  $fileId = 'file_011CNha8iCJcU1wXNR6q4V8w';

  // 获取文件元数据
  $fileInfo = $client->files->retrieveMetadata($fileId);
  echo "Filename: {$fileInfo->filename}, Size: {$fileInfo->sizeBytes} bytes\n";

  // 列出文件（第一页）
  foreach ($client->files->list()->getItems() as $file) {
      echo "{$file->filename} - {$file->createdAt->format(DATE_ATOM)}\n";
  }

  // 删除文件
  $client->files->delete($fileId);
  ```

  ```ruby Ruby
  client = Anthropic::Client.new
  file_id = "file_011CNha8iCJcU1wXNR6q4V8w"

  # 获取文件元数据
  file_info = client.files.retrieve_metadata(file_id)
  puts "Filename: #{file_info.filename}, Size: #{file_info.size_bytes} bytes"

  # 列出所有文件
  client.files.list.auto_paging_each do |file|
    puts "#{file.filename} - #{file.created_at}"
  end

  # 删除文件
  client.files.delete(file_id)
  ```
</CodeGroup>

<Note>
  有关完整详情，请参阅 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)。
</Note>

### 多轮对话

响应的 `container` 对象携带容器的 `id` 和 `expires_at` 时间戳（有关生命周期详情，请参阅[容器复用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#container-reuse)）。通过指定容器 ID，可在多条消息之间复用同一容器：

<CodeGroup>
  ```bash cURL
  # 多轮容器复用不太适合一次性的 shell
  # 命令；使用某个 SDK 选项会更合适。从第一个响应中获取
  # container.id，然后在下一个请求中将其作为
  # "container": {"id": "...", "skills": [...]} 与对话历史一起传入。
  ```

  ```bash CLI
  # 第一个请求创建容器
  CONTAINER_ID=$(ant messages create \
    --transform container.id \
    --raw-output <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - {type: anthropic, skill_id: xlsx, version: latest}
  messages:
    - role: user
      content: Create a sample sales dataset and analyze it
  tools:
    - {type: code_execution_20250825, name: code_execution}
  YAML
  )

  # 使用同一容器继续对话
  ant messages create <<YAML
  model: claude-opus-5
  max_tokens: 4096
  container:
    id: $CONTAINER_ID  # Reuse container
    skills:
      - {type: anthropic, skill_id: xlsx, version: latest}
  messages:
    - role: user
      content: Create a sample sales dataset and analyze it
    - role: assistant
      content: []  # the assistant's text from the first response
    - role: user
      content: What was the total revenue?
  tools:
    - {type: code_execution_20250825, name: code_execution}
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  # 第一个请求创建容器
  response1 = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [{"type": "anthropic", "skill_id": "xlsx", "version": "latest"}]
      },
      messages=[
          {"role": "user", "content": "Create a sample sales dataset and analyze it"}
      ],
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )

  # 使用同一容器继续对话
  messages = [
      {"role": "user", "content": "Create a sample sales dataset and analyze it"},
      {
          # 沿用助手的文本；container.id 携带执行状态
          "role": "assistant",
          "content": "\n".join(
              block.text for block in response1.content if block.type == "text"
          ),
      },
      {"role": "user", "content": "What was the total revenue?"},
  ]

  response2 = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "id": response1.container.id,  # Reuse container
          "skills": [{"type": "anthropic", "skill_id": "xlsx", "version": "latest"}],
      },
      messages=messages,
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // 第一个请求创建容器
  const response1 = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [{ type: "anthropic", skill_id: "xlsx", version: "latest" }]
    },
    messages: [{ role: "user", content: "Create a sample sales dataset and analyze it" }],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  });

  // 使用同一容器继续对话
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: "Create a sample sales dataset and analyze it" },
    {
      role: "assistant",
      // 将助手的文本向前传递；container.id 携带执行状态
      content: response1.content
        .filter((block) => block.type === "text")
        .map((block) => block.text)
        .join("\n")
    },
    { role: "user", content: "What was the total revenue?" }
  ];

  const response2 = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      id: response1.container!.id, // Reuse container
      skills: [{ type: "anthropic", skill_id: "xlsx", version: "latest" }]
    },
    messages,
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  });
  ```

  ```csharp C#
  AnthropicClient client = new();

  // 使用 Skill 的第一个请求
  var parameters1 = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "xlsx",
                  Version = "latest",
              },
          ],
      },
      Messages = [new() { Role = Role.User, Content = "Create a sample sales dataset and analyze it" }],
      Tools = [new CodeExecutionTool20250825()],
  };

  var response1 = await client.Messages.Create(parameters1);

  // 在同一容器中继续对话
  // 将助手的文本向前传递；container.id 携带执行状态
  var assistantText = string.Join(
      "\n",
      response1.Content.Select(block => block.TryPickText(out var text) ? text.Text : null).Where(text => text is not null)
  );

  var parameters2 = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          ID = response1.Container!.ID,
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "xlsx",
                  Version = "latest",
              },
          ],
      },
      Messages =
      [
          new() { Role = Role.User, Content = "Create a sample sales dataset and analyze it" },
          new() { Role = Role.Assistant, Content = assistantText },
          new() { Role = Role.User, Content = "What was the total revenue?" },
      ],
      Tools = [new CodeExecutionTool20250825()],
  };

  var response2 = await client.Messages.Create(parameters2);
  Console.WriteLine(response2);
  ```

  ```go Go
  client := anthropic.NewClient()

  response1, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "xlsx",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Create a sample sales dataset and analyze it")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  // 将助手的文本向前传递；container.id 携带执行状态
  var textParts []string
  for _, block := range response1.Content {
  	if block.Type == "text" {
  		textParts = append(textParts, block.Text)
  	}
  }
  assistantText := strings.Join(textParts, "\n")

  response2, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			ID: anthropic.String(response1.Container.ID), // Reuse container
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "xlsx",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Create a sample sales dataset and analyze it")),
  		{
  			Role:    anthropic.MessageParamRoleAssistant,
  			Content: []anthropic.ContentBlockParamUnion{anthropic.NewTextBlock(assistantText)},
  		},
  		anthropic.NewUserMessage(anthropic.NewTextBlock("What was the total revenue?")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println(response2)
  ```

  ```java Java
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;
  import com.anthropic.models.messages.CodeExecutionTool20250825;
  import com.anthropic.models.messages.ContentBlock;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params1 = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .container(ContainerParams.builder()
              .addSkill(SkillParams.builder()
                  .type(SkillParams.Type.ANTHROPIC)
                  .skillId("xlsx")
                  .version("latest")
                  .build())
              .build())
          .addUserMessage("Create a sample sales dataset and analyze it")
          .addTool(CodeExecutionTool20250825.builder().build())
          .build();

      Message response1 = client.messages().create(params1);

      MessageCreateParams params2 = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .container(ContainerParams.builder()
              .id(response1.container().get().id())
              .addSkill(SkillParams.builder()
                  .type(SkillParams.Type.ANTHROPIC)
                  .skillId("xlsx")
                  .version("latest")
                  .build())
              .build())
          .addUserMessage("Create a sample sales dataset and analyze it")
          // 将助手的文本向前传递；container.id 携带执行状态
          .addAssistantMessage(response1.content().stream()
              .filter(ContentBlock::isText)
              .map(block -> block.asText().text())
              .collect(Collectors.joining("\n")))
          .addUserMessage("What was the total revenue?")
          .addTool(CodeExecutionTool20250825.builder().build())
          .build();

      Message response2 = client.messages().create(params2);
      System.out.println(response2);
  }
  ```

  ```php PHP
  $client = new Client();

  $response1 = $client->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Create a sample sales dataset and analyze it']
      ],
      model: 'claude-opus-5',
      container: [
          'skills' => [
              ['type' => 'anthropic', 'skillID' => 'xlsx', 'version' => 'latest']
          ]
      ],
      tools: [
          ['type' => 'code_execution_20250825', 'name' => 'code_execution']
      ]
  );

  $messages = [
      ['role' => 'user', 'content' => 'Create a sample sales dataset and analyze it'],
      // 将助手的文本向前传递；container.id 携带执行状态
      ['role' => 'assistant', 'content' => implode("\n", array_map(
          fn ($block) => $block->text,
          array_filter($response1->content, fn ($block) => $block->type === 'text'),
      ))],
      ['role' => 'user', 'content' => 'What was the total revenue?']
  ];

  $response2 = $client->messages->create(
      maxTokens: 4096,
      messages: $messages,
      model: 'claude-opus-5',
      container: [
          'id' => $response1->container->id,
          'skills' => [
              ['type' => 'anthropic', 'skillID' => 'xlsx', 'version' => 'latest']
          ]
      ],
      tools: [
          ['type' => 'code_execution_20250825', 'name' => 'code_execution']
      ]
  );

  echo $response2;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  response1 = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [{ type: "anthropic", skill_id: "xlsx", version: "latest" }]
    },
    messages: [
      { role: "user", content: "Create a sample sales dataset and analyze it" }
    ],
    tools: [
      { type: "code_execution_20250825", name: "code_execution" }
    ]
  )

  messages = [
    { role: "user", content: "Create a sample sales dataset and analyze it" },
    {
      # 将助手的文本向前传递；container.id 携带执行状态
      role: "assistant",
      content: response1.content.filter_map { |block| block.text if block.type == :text }.join("\n")
    },
    { role: "user", content: "What was the total revenue?" }
  ]

  response2 = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      id: response1.container.id,
      skills: [
        { type: "anthropic", skill_id: "xlsx", version: "latest" }
      ]
    },
    messages: messages,
    tools: [
      { type: "code_execution_20250825", name: "code_execution" }
    ]
  )

  puts response2
  ```
</CodeGroup>

### 长时间运行的操作

Skills 可能会执行需要多轮的操作。请处理 `pause_turn` 停止原因：

<CodeGroup>
  ```bash cURL
  # 初始请求
  RESPONSE=$(curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "container": {
        "skills": [
          {
            "type": "custom",
            "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
            "version": "latest"
          }
        ]
      },
      "messages": [{
        "role": "user",
        "content": "Generate and process a large sample dataset"
      }],
      "tools": [{
        "type": "code_execution_20250825",
        "name": "code_execution"
      }]
    }')

  # 如果 stop_reason 为 "pause_turn"，则在同一容器中继续，
  # 将上一个响应的 content 数组作为 assistant 轮次追加到 messages 中。
  # 重复此继续请求，直到 stop_reason 不再是 "pause_turn"。
  STOP_REASON=$(echo "$RESPONSE" | jq -r '.stop_reason')
  CONTAINER_ID=$(echo "$RESPONSE" | jq -r '.container.id')

  RESPONSE=$(curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d "{
      \"model\": \"claude-opus-5\",
      \"max_tokens\": 4096,
      \"container\": {
        \"id\": \"$CONTAINER_ID\",
        \"skills\": [{
          \"type\": \"custom\",
          \"skill_id\": \"skill_01AbCdEfGhIjKlMnOpQrStUv\",
          \"version\": \"latest\"
        }]
      },
      \"messages\": [],
      \"tools\": [{
        \"type\": \"code_execution_20250825\",
        \"name\": \"code_execution\"
      }]
    }")
  ```

  ```bash CLI
  RESP=$(mktemp)

  # 初始请求：将完整的 JSON 响应保存到临时文件
  ant messages create > "$RESP" <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: custom
        skill_id: skill_01AbCdEfGhIjKlMnOpQrStUv
        version: latest
  messages:
    - role: user
      content: Generate and process a large sample dataset
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML

  # 如果 stop_reason 为 "pause_turn"，则在同一容器中继续，
  # 将上一个响应的 content 数组作为 assistant 轮次
  # 追加到 messages 中。重复直到 stop_reason 不再是 "pause_turn"。
  CONTAINER_ID=$(jq -r '.container.id' "$RESP")

  ant messages create > "$RESP" <<YAML
  model: claude-opus-5
  max_tokens: 4096
  container:
    id: $CONTAINER_ID
    skills:
      - type: custom
        skill_id: skill_01AbCdEfGhIjKlMnOpQrStUv
        version: latest
  messages: [] # replace with conversation history + prior assistant content
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  messages = [{"role": "user", "content": "Generate and process a large sample dataset"}]
  max_retries = 10

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [
              {
                  "type": "custom",
                  "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
                  "version": "latest",
              }
          ]
      },
      messages=messages,
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )

  # 处理长时间操作的 pause_turn
  for _ in range(max_retries):
      if response.stop_reason != "pause_turn":
          break

      messages.append({"role": "assistant", "content": response.content})
      response = client.messages.create(
          model="claude-opus-5",
          max_tokens=4096,
          container={
              "id": response.container.id,
              "skills": [
                  {
                      "type": "custom",
                      "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
                      "version": "latest",
                  }
              ],
          },
          messages=messages,
          tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
      )
  ```

  ```typescript TypeScript
  const client = new Anthropic();
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: "Generate and process a large sample dataset" }
  ];
  const maxRetries = 10;

  let response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [{ type: "custom", skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv", version: "latest" }]
    },
    messages,
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  });

  // 处理长时间操作的 pause_turn
  for (let i = 0; i < maxRetries; i++) {
    if (response.stop_reason !== "pause_turn") {
      break;
    }

    messages.push({
      role: "assistant",
      content: response.content as Anthropic.ContentBlockParam[]
    });
    response = await client.messages.create({
      model: "claude-opus-5",
      max_tokens: 4096,
      container: {
        id: response.container!.id,
        skills: [
          { type: "custom", skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv", version: "latest" }
        ]
      },
      messages,
      tools: [{ type: "code_execution_20250825", name: "code_execution" }]
    });
  }
  ```

  ```csharp C#
  using System.Text.Json;
  // ...
  AnthropicClient client = new();

  List<MessageParam> messages =
  [
      new() { Role = Role.User, Content = "Generate and process a large sample dataset" },
  ];

  var maxRetries = 10;
  string? containerId = null;
  Message? response = null;

  for (var i = 0; i < maxRetries; i++)
  {
      var parameters = new MessageCreateParams
      {
          Model = "claude-opus-5",
          MaxTokens = 4096,
          Container = containerId is null
              ? new ContainerParams
              {
                  Skills =
                  [
                      new SkillParams
                      {
                          Type = SkillParamsType.Custom,
                          SkillID = "skill_01AbCdEfGhIjKlMnOpQrStUv",
                          Version = "latest",
                      },
                  ],
              }
              : new ContainerParams
              {
                  ID = containerId,
                  Skills =
                  [
                      new SkillParams
                      {
                          Type = SkillParamsType.Custom,
                          SkillID = "skill_01AbCdEfGhIjKlMnOpQrStUv",
                          Version = "latest",
                      },
                  ],
              },
          Messages = messages,
          Tools = [new CodeExecutionTool20250825()],
      };

      response = await client.Messages.Create(parameters);
      containerId = response.Container!.ID;

      if (response.StopReason != StopReason.PauseTurn)
      {
          break;
      }

      // 追加已暂停轮次的内容并继续
      var assistantContent = JsonSerializer.SerializeToElement(
          response.Content.Select(block => block.Json).ToArray()
      );
      messages.Add(new() { Role = Role.Assistant, Content = new MessageParamContent(assistantContent) });
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  messages := []anthropic.MessageParam{
  	anthropic.NewUserMessage(anthropic.NewTextBlock("Generate and process a large sample dataset")),
  }
  maxRetries := 10

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeCustom,
  					SkillID: "skill_01AbCdEfGhIjKlMnOpQrStUv",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: messages,
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  for i := 0; i < maxRetries; i++ {
  	if response.StopReason != anthropic.StopReasonPauseTurn {
  		break
  	}

  	messages = append(messages, response.ToParam())

  	response, err = client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  		Model:     "claude-opus-5",
  		MaxTokens: 4096,
  		Container: anthropic.MessageCreateParamsContainerUnion{
  			OfContainers: &anthropic.ContainerParams{
  				ID: anthropic.String(response.Container.ID), // Reuse container
  				Skills: []anthropic.SkillParams{
  					{
  						Type:    anthropic.SkillParamsTypeCustom,
  						SkillID: "skill_01AbCdEfGhIjKlMnOpQrStUv",
  						Version: anthropic.String("latest"),
  					},
  				},
  			},
  		},
  		Messages: messages,
  		Tools: []anthropic.ToolUnionParam{
  			{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  		},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}
  }

  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;
  import com.anthropic.models.messages.CodeExecutionTool20250825;
  import com.anthropic.models.messages.StopReason;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      List<MessageParam> messages = new ArrayList<>();
      messages.add(
          MessageParam.builder()
              .role(MessageParam.Role.USER)
              .content("Generate and process a large sample dataset")
              .build()
      );
      int maxRetries = 10;

      Message response = client.messages().create(
          MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5)
              .maxTokens(4096L)
              .container(ContainerParams.builder()
                  .addSkill(SkillParams.builder()
                      .type(SkillParams.Type.CUSTOM)
                      .skillId("skill_01AbCdEfGhIjKlMnOpQrStUv")
                      .version("latest")
                      .build())
                  .build())
              .messages(messages)
              .addTool(CodeExecutionTool20250825.builder().build())
              .build());

      for (int i = 0; i < maxRetries; i++) {
          if (!response.stopReason().isPresent()
                  || !response.stopReason().get().equals(StopReason.PAUSE_TURN)) {
              break;
          }

          messages.add(response.toParam());

          response = client.messages().create(
              MessageCreateParams.builder()
                  .model(Model.CLAUDE_OPUS_5)
                  .maxTokens(4096L)
                  .container(ContainerParams.builder()
                      .id(response.container().get().id())
                      .addSkill(SkillParams.builder()
                          .type(SkillParams.Type.CUSTOM)
                          .skillId("skill_01AbCdEfGhIjKlMnOpQrStUv")
                          .version("latest")
                          .build())
                      .build())
                  .messages(messages)
                  .addTool(CodeExecutionTool20250825.builder().build())
                  .build());
      }
  }
  ```

  ```php PHP
  $client = new Client();

  $messages = [
      ['role' => 'user', 'content' => 'Generate and process a large sample dataset']
  ];
  $maxRetries = 10;

  $response = $client->messages->create(
      maxTokens: 4096,
      messages: $messages,
      model: 'claude-opus-5',
      container: [
          'skills' => [
              [
                  'type' => 'custom',
                  'skillID' => 'skill_01AbCdEfGhIjKlMnOpQrStUv',
                  'version' => 'latest'
              ]
          ]
      ],
      tools: [['type' => 'code_execution_20250825', 'name' => 'code_execution']]
  );

  for ($i = 0; $i < $maxRetries; $i++) {
      if ($response->stopReason !== 'pause_turn') {
          break;
      }

      $messages[] = ['role' => 'assistant', 'content' => $response->content];

      $response = $client->messages->create(
          maxTokens: 4096,
          messages: $messages,
          model: 'claude-opus-5',
          container: [
              'id' => $response->container->id,
              'skills' => [
                  [
                      'type' => 'custom',
                      'skillID' => 'skill_01AbCdEfGhIjKlMnOpQrStUv',
                      'version' => 'latest'
                  ]
              ]
          ],
          tools: [['type' => 'code_execution_20250825', 'name' => 'code_execution']]
      );
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  messages = [
    { role: "user", content: "Generate and process a large sample dataset" }
  ]
  max_retries = 10

  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        {
          type: "custom",
          skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
          version: "latest"
        }
      ]
    },
    messages: messages,
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  )

  max_retries.times do
    break if response.stop_reason != :pause_turn

    messages << { role: "assistant", content: response.content }

    response = client.messages.create(
      model: "claude-opus-5",
      max_tokens: 4096,
      container: {
        id: response.container.id,
        skills: [
          {
            type: "custom",
            skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
            version: "latest"
          }
        ]
      },
      messages: messages,
      tools: [{ type: "code_execution_20250825", name: "code_execution" }]
    )
  end
  ```
</CodeGroup>

<Note>
  响应可能包含 `pause_turn` 停止原因，这表示 API 暂停了一个长时间运行的 Skill 操作。您可以在后续请求中原样提供该响应，让 Claude 继续其轮次；如果您想中断对话并提供额外指导，也可以修改内容。
</Note>

### 使用多个 Skills

在单个请求中组合多个 Skills 以处理复杂的工作流：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "container": {
        "skills": [
          {
            "type": "anthropic",
            "skill_id": "xlsx",
            "version": "latest"
          },
          {
            "type": "anthropic",
            "skill_id": "pptx",
            "version": "latest"
          },
          {
            "type": "custom",
            "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
            "version": "latest"
          }
        ]
      },
      "messages": [{
        "role": "user",
        "content": "Analyze sales data and create a presentation"
      }],
      "tools": [{
        "type": "code_execution_20250825",
        "name": "code_execution"
      }]
    }'
  ```

  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: anthropic
        skill_id: xlsx
        version: latest
      - type: anthropic
        skill_id: pptx
        version: latest
      - type: custom
        skill_id: skill_01AbCdEfGhIjKlMnOpQrStUv
        version: latest
  messages:
    - role: user
      content: Analyze sales data and create a presentation
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [
              {"type": "anthropic", "skill_id": "xlsx", "version": "latest"},
              {"type": "anthropic", "skill_id": "pptx", "version": "latest"},
              {
                  "type": "custom",
                  "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
                  "version": "latest",
              },
          ]
      },
      messages=[
          {"role": "user", "content": "Analyze sales data and create a presentation"}
      ],
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        {
          type: "anthropic",
          skill_id: "xlsx",
          version: "latest"
        },
        {
          type: "anthropic",
          skill_id: "pptx",
          version: "latest"
        },
        {
          type: "custom",
          skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
          version: "latest"
        }
      ]
    },
    messages: [
      {
        role: "user",
        content: "Analyze sales data and create a presentation"
      }
    ],
    tools: [
      {
        type: "code_execution_20250825",
        name: "code_execution"
      }
    ]
  });
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "xlsx",
                  Version = "latest",
              },
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "pptx",
                  Version = "latest",
              },
              new SkillParams
              {
                  Type = SkillParamsType.Custom,
                  SkillID = "skill_01AbCdEfGhIjKlMnOpQrStUv",
                  Version = "latest",
              },
          ],
      },
      Messages = [new() { Role = Role.User, Content = "Analyze sales data and create a presentation" }],
      Tools = [new CodeExecutionTool20250825()],
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "xlsx",
  					Version: anthropic.String("latest"),
  				},
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "pptx",
  					Version: anthropic.String("latest"),
  				},
  				{
  					Type:    anthropic.SkillParamsTypeCustom,
  					SkillID: "skill_01AbCdEfGhIjKlMnOpQrStUv",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Analyze sales data and create a presentation")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;
  import com.anthropic.models.messages.CodeExecutionTool20250825;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .container(ContainerParams.builder()
              .skills(List.of(
                  SkillParams.builder()
                      .type(SkillParams.Type.ANTHROPIC)
                      .skillId("xlsx")
                      .version("latest")
                      .build(),
                  SkillParams.builder()
                      .type(SkillParams.Type.ANTHROPIC)
                      .skillId("pptx")
                      .version("latest")
                      .build(),
                  SkillParams.builder()
                      .type(SkillParams.Type.CUSTOM)
                      .skillId("skill_01AbCdEfGhIjKlMnOpQrStUv")
                      .version("latest")
                      .build()
              ))
              .build())
          .addUserMessage("Analyze sales data and create a presentation")
          .addTool(CodeExecutionTool20250825.builder().build())
          .build();

      Message response = client.messages().create(params);
      System.out.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  $message = $client->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Analyze sales data and create a presentation']
      ],
      model: 'claude-opus-5',
      container: [
          'skills' => [
              [
                  'type' => 'anthropic',
                  'skillID' => 'xlsx',
                  'version' => 'latest'
              ],
              [
                  'type' => 'anthropic',
                  'skillID' => 'pptx',
                  'version' => 'latest'
              ],
              [
                  'type' => 'custom',
                  'skillID' => 'skill_01AbCdEfGhIjKlMnOpQrStUv',
                  'version' => 'latest'
              ]
          ]
      ],
      tools: [
          ['type' => 'code_execution_20250825', 'name' => 'code_execution']
      ]
  );

  echo $message;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        {
          type: "anthropic",
          skill_id: "xlsx",
          version: "latest"
        },
        {
          type: "anthropic",
          skill_id: "pptx",
          version: "latest"
        },
        {
          type: "custom",
          skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
          version: "latest"
        }
      ]
    },
    messages: [
      { role: "user", content: "Analyze sales data and create a presentation" }
    ],
    tools: [
      { type: "code_execution_20250825", name: "code_execution" }
    ]
  )
  puts message
  ```
</CodeGroup>

***

## 管理自定义 Skills

<Warning id="workspace-scoped-access">
  **自定义 Skills 对您的整个工作区可访问，而不是限定于某个最终用户、对话或会话。** 任何有权访问某个工作区的 API 密钥都可以读取、调用和删除上传到该工作区的每一个自定义 Skill。每个服务账户，以及每个组织角色允许 API 访问的用户，除了您将其添加到的任何工作区之外，还可以使用默认工作区（Default Workspace），因此请将必须保持隔离的 Skills 放在各自独立的[工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#api-keys-and-resource-scoping)中，并仅使用限定于该工作区的密钥访问它们。

  如果您正在基于 Skills API 构建多租户平台，请为每个租户创建单独的[工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces)。工作区是自定义 Skills 的隔离边界，因此每个租户一个工作区可以让每个租户的 Skills 与其他所有租户实现硬隔离。默认情况下，每个组织最多可以拥有 100 个工作区（请参阅[工作区的工作原理](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#how-workspaces-work)）；如果您需要更多工作区用于租户隔离，请联系您的客户团队。
</Warning>

### 创建 Skill

Skill 包是一个目录，其顶层包含一个带有 `name` 和 `description` YAML frontmatter 的 `SKILL.md` 文件，以及任何辅助脚本或资源。请参阅[在 API 中开始使用 Agent Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/quickstart) 了解如何编写一个 Skill，并参阅示例之后的**要求**列表了解完整约束。

上传您的自定义 Skill，使其在您的工作区中可用。您可以上传 zip 压缩包或单独的文件对象。Python SDK 还提供了一个接受目录路径的 `files_from_dir` 辅助函数。

文件通过您附加的文件名来标识（cURL 示例中的 `;filename=` 后缀以及 SDK 示例中的 filename 参数）。对于本演练中的 skill，请使用 `zip -r financial_skill.zip financial_skill/` 创建一个 zip，并用它替换 zip 上传选项中的 `example_skill.zip` 占位符。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -X POST "https://api.anthropic.com/v1/skills" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -F "files[]=@financial_skill/SKILL.md;filename=financial_skill/SKILL.md" \
    -F "files[]=@financial_skill/analyze.py;filename=financial_skill/analyze.py"
  ```

  <MultiFileExample language="cli" label="CLI">
    ```bash CLI
    zip -r financial_skill.zip financial_skill/
    ant skills create --file financial_skill.zip
    ```

    <File filename="financial_skill/SKILL.md">
      ```markdown
      ---
      name: financial-skill
      description: Docs example skill.
      ---
      ```
    </File>

    <File filename="financial_skill/analyze.py">
      ```python
      print("financial analysis helper")
      ```
    </File>
  </MultiFileExample>

  ```python Python
  from anthropic.lib import files_from_dir

  client = anthropic.Anthropic()

  # 选项 1：使用 zip 文件
  skill = client.skills.create(
      files=[open("example_skill.zip", "rb")],
  )

  # 选项 2：使用文件元组 (filename, file_content, mime_type)
  skill = client.skills.create(
      files=[
          (
              "financial_skill/SKILL.md",
              open("financial_skill/SKILL.md", "rb"),
              "text/markdown",
          ),
          (
              "financial_skill/analyze.py",
              open("financial_skill/analyze.py", "rb"),
              "text/x-python",
          ),
      ],
  )

  # 选项 3：使用 files_from_dir 辅助函数（仅限 Python）
  skill = client.skills.create(
      files=files_from_dir("financial_skill"),
  )

  print(f"Created skill: {skill.id}")
  print(f"Latest version: {skill.latest_version_id}")
  ```

  ```typescript TypeScript
  import { toFile } from "@anthropic-ai/sdk";
  import fs from "node:fs";
  // ...

  const client = new Anthropic();

  // 选项 1：使用 zip 文件
  const skillFromZip = await client.skills.create({
    files: [await toFile(fs.createReadStream("example_skill.zip"), "example_skill.zip")]
  });

  // 选项 2：使用单独的文件对象
  const skill = await client.skills.create({
    files: [
      await toFile(fs.createReadStream("financial_skill/SKILL.md"), "financial_skill/SKILL.md", {
        type: "text/markdown"
      }),
      await toFile(
        fs.createReadStream("financial_skill/analyze.py"),
        "financial_skill/analyze.py",
        { type: "text/x-python" }
      )
    ]
  });

  console.log(`Created skill: ${skill.id}`);
  console.log(`Latest version: ${skill.latest_version_id}`);
  ```

  ```csharp C#
  using Anthropic.Core;
  // ...

  AnthropicClient client = new();

  // 选项 1：使用 zip 文件
  var parameters = new SkillCreateParams
  {
      Files = [File.OpenRead("example_skill.zip")],
  };

  var skill = await client.Skills.Create(parameters);

  // 选项 2：使用单独的文件（带路径限定的文件名可保留 Skill 的目录结构）
  var parameters2 = new SkillCreateParams
  {
      Files =
      [
          new BinaryContent
          {
              Stream = File.OpenRead("financial_skill/SKILL.md"),
              FileName = "financial_skill/SKILL.md",
          },
          new BinaryContent
          {
              Stream = File.OpenRead("financial_skill/analyze.py"),
              FileName = "financial_skill/analyze.py",
          },
      ],
  };

  var skill2 = await client.Skills.Create(parameters2);

  Console.WriteLine($"Created skill: {skill.ID}");
  Console.WriteLine($"Latest version: {skill.LatestVersionID}");
  Console.WriteLine($"Created skill 2: {skill2.ID}");
  ```

  ```go Go
  client := anthropic.NewClient()

  // 选项 1：使用 zip 文件
  zipFile, err := os.Open("example_skill.zip")
  if err != nil {
  	log.Fatal(err)
  }
  defer zipFile.Close()

  skill, err := client.Skills.New(context.TODO(), anthropic.SkillNewParams{
  	Files: []io.Reader{zipFile},
  })
  if err != nil {
  	log.Fatal(err)
  }

  // 选项 2：使用单独的文件
  skillMd, err := os.Open("financial_skill/SKILL.md")
  if err != nil {
  	log.Fatal(err)
  }
  defer skillMd.Close()

  analyzePy, err := os.Open("financial_skill/analyze.py")
  if err != nil {
  	log.Fatal(err)
  }
  defer analyzePy.Close()

  skill2, err := client.Skills.New(context.TODO(), anthropic.SkillNewParams{
  	Files: []io.Reader{
  		anthropic.File(skillMd, "financial_skill/SKILL.md", "text/markdown"),
  		anthropic.File(analyzePy, "financial_skill/analyze.py", "text/x-python"),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("Created skill: %s\n", skill.ID)
  fmt.Printf("Latest version: %s\n", skill.LatestVersionID)
  fmt.Printf("Created skill 2: %s\n", skill2.ID)
  ```

  ```java Java
  import com.anthropic.core.MultipartField;
  import com.anthropic.models.skills.SkillCreateParams;
  import com.anthropic.models.skills.Skill;
  // ...
  void main() throws Exception {
  // ...
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      // 选项 1：使用 zip 文件
      SkillCreateParams params = SkillCreateParams.builder()
          .addFile(MultipartField.<InputStream>builder()
              .value(Files.newInputStream(Path.of("example_skill.zip")))
              .filename("example_skill.zip")
              .contentType("application/zip")
              .build())
          .build();

      Skill skill = client.skills().create(params);

      // 选项 2：使用单独的文件（带路径的文件名可保留 Skill 的目录结构）
      SkillCreateParams params2 = SkillCreateParams.builder()
          .addFile(MultipartField.<InputStream>builder()
              .value(Files.newInputStream(Path.of("financial_skill/SKILL.md")))
              .filename("financial_skill/SKILL.md")
              .contentType("text/markdown")
              .build())
          .addFile(MultipartField.<InputStream>builder()
              .value(Files.newInputStream(Path.of("financial_skill/analyze.py")))
              .filename("financial_skill/analyze.py")
              .contentType("text/x-python")
              .build())
          .build();

      Skill skill2 = client.skills().create(params2);

      System.out.println("Created skill: " + skill.id());
      System.out.println("Latest version: " + skill.latestVersionId());
      System.out.println("Created skill 2: " + skill2.id());
  }
  ```

  ```php PHP
  use Anthropic\Core\FileParam;
  // ...

  $client = new Client();

  // 选项 1：使用 zip 文件
  $skill = $client->skills->create(
      files: [
          FileParam::fromResource(fopen('example_skill.zip', 'r')),
      ],
  );

  // 选项 2：使用单独的文件
  $skill = $client->skills->create(
      files: [
          FileParam::fromResource(
              fopen('financial_skill/SKILL.md', 'r'),
              filename: 'financial_skill/SKILL.md',
              contentType: 'text/markdown',
          ),
          FileParam::fromResource(
              fopen('financial_skill/analyze.py', 'r'),
              filename: 'financial_skill/analyze.py',
              contentType: 'text/x-python',
          ),
      ],
  );

  echo "Created skill: {$skill->id}\n";
  echo "Latest version: {$skill->latestVersionID}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 选项 1：使用 zip 文件
  skill = client.skills.create(
    files: [
      File.open("example_skill.zip", "rb")
    ]
  )

  # 选项 2：使用单独的文件
  skill = client.skills.create(
    files: [
      Anthropic::FilePart.new(
        Pathname("financial_skill/SKILL.md"),
        filename: "financial_skill/SKILL.md",
        content_type: "text/markdown"
      ),
      Anthropic::FilePart.new(
        Pathname("financial_skill/analyze.py"),
        filename: "financial_skill/analyze.py",
        content_type: "text/x-python"
      )
    ]
  )

  puts "Created skill: #{skill.id}"
  puts "Latest version: #{skill.latest_version_id}"
  ```
</CodeGroup>

**要求：**

* 必须在上传根目录（或单个外层文件夹的顶层）包含一个 `SKILL.md` 文件

* `display_name` 是可选的：省略时，它从 `SKILL.md` 的 `name` 派生；显式指定的值最多可为 255 个字符，且在工作区内无需唯一

* 上传总大小必须小于 30 MB（未压缩）

* YAML frontmatter 要求：

  * `name`：最多 64 个字符，仅限小写字母/数字/连字符，不得包含 XML 标签，不得包含保留字（"anthropic"、"claude"）
  * `description`：最多 1024 个字符，非空，不得包含 XML 标签

有关完整的请求/响应模式，请参阅 [Create Skill API 参考](https://platform.claude.com/docs/zh-CN/api/skills/create)。

### 列出 Skills

检索您的工作区可用的所有 Skills，包括 Anthropic 预构建 Skills 和您的自定义 Skills。使用 `source` 参数按 skill 类型筛选：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  # 列出所有 Skills
  curl "https://api.anthropic.com/v1/skills" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"

  # 仅列出自定义 Skills
  curl "https://api.anthropic.com/v1/skills?source=custom" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  # 列出所有 Skills
  ant skills list

  # 仅列出自定义 Skills
  ant skills list --source custom
  ```

  ```python Python
  client = anthropic.Anthropic()

  # 列出所有 Skills
  for skill in client.skills.list():
      print(f"{skill.id}: {skill.display_name} (source: {skill.source.type})")

  # 仅列出自定义 Skills
  custom_skills = client.skills.list(source="custom")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // 列出所有 Skills
  for await (const skill of client.skills.list()) {
    console.log(`${skill.id}: ${skill.display_name} (source: ${skill.source.type})`);
  }

  // 仅列出自定义 Skills
  const customSkills = await client.skills.list({
    source: "custom"
  });
  ```

  ```csharp C#
  AnthropicClient client = new();

  // List all Skills
  await foreach (var skill in (await client.Skills.List()).Paginate())
  {
      Console.WriteLine($"{skill.ID}: {skill.DisplayName} (source: {skill.Source.Type})");
  }

  // List only custom Skills
  var customSkills = await client.Skills.List(new SkillListParams { Source = "custom" });
  ```

  ```go Go
  client := anthropic.NewClient()

  // 列出所有 Skills
  skills := client.Skills.ListAutoPaging(context.TODO(), anthropic.SkillListParams{})

  for skills.Next() {
  	skill := skills.Current()
  	fmt.Printf("%s: %s (source: %s)\n", skill.ID, skill.DisplayName, skill.Source.Type)
  }
  if skills.Err() != nil {
  	log.Fatal(skills.Err())
  }

  // 仅列出自定义 Skills
  customSkills := client.Skills.ListAutoPaging(context.TODO(), anthropic.SkillListParams{
  	Source: anthropic.String("custom"),
  })

  for customSkills.Next() {
  	skill := customSkills.Current()
  	fmt.Printf("%s: %s (source: %s)\n", skill.ID, skill.DisplayName, skill.Source.Type)
  }
  if customSkills.Err() != nil {
  	log.Fatal(customSkills.Err())
  }
  ```

  ```java Java
  import com.anthropic.models.skills.SkillListParams;
  import com.anthropic.models.skills.SkillListPage;
  import com.anthropic.models.skills.Skill;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      // 列出 Skills（第一页）
      SkillListPage skills = client.skills().list();

      for (Skill skill : skills.data()) {
          System.out.println(skill.id() + ": " + skill.displayName() + " (source: " + skill.source().type() + ")");
      }

      // 仅列出自定义 Skills
      SkillListParams customParams = SkillListParams.builder()
          .source("custom")
          .build();

      SkillListPage customSkills = client.skills().list(customParams);
  }
  ```

  ```php PHP
  $client = new Client();

  // 列出 Skills（第一页）
  foreach ($client->skills->list()->getItems() as $skill) {
      echo "{$skill->id}: {$skill->displayName} (source: {$skill->source->type})\n";
  }

  // 仅列出自定义 Skills
  $customSkills = $client->skills->list(
      source: 'custom',
  );
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 列出所有 Skills
  client.skills.list.auto_paging_each do |skill|
    puts "#{skill.id}: #{skill.display_name} (source: #{skill.source.type})"
  end

  # 仅列出自定义 Skills
  custom_skills = client.skills.list(
    source: "custom"
  )
  ```
</CodeGroup>

有关分页和筛选选项，请参阅 [List Skills API 参考](https://platform.claude.com/docs/zh-CN/api/skills/list)。

### 检索 Skill

获取特定 Skill 的详细信息：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl "https://api.anthropic.com/v1/skills/skill_01AbCdEfGhIjKlMnOpQrStUv" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant skills retrieve --skill-id skill_01AbCdEfGhIjKlMnOpQrStUv
  ```

  ```python Python
  client = anthropic.Anthropic()

  skill = client.skills.retrieve(skill_id="skill_01AbCdEfGhIjKlMnOpQrStUv")

  print(f"Skill: {skill.display_name}")
  print(f"Latest version: {skill.latest_version_id}")
  print(f"Created: {skill.created_at}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const skill = await client.skills.retrieve("skill_01AbCdEfGhIjKlMnOpQrStUv");

  console.log(`Skill: ${skill.display_name}`);
  console.log(`Latest version: ${skill.latest_version_id}`);
  console.log(`Created: ${skill.created_at}`);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var skill = await client.Skills.Retrieve("skill_01AbCdEfGhIjKlMnOpQrStUv");

  Console.WriteLine($"Skill: {skill.DisplayName}");
  Console.WriteLine($"Latest version: {skill.LatestVersionID}");
  Console.WriteLine($"Created: {skill.CreatedAt}");
  ```

  ```go Go
  client := anthropic.NewClient()

  skill, err := client.Skills.Get(
  	context.TODO(),
  	"skill_01AbCdEfGhIjKlMnOpQrStUv",
  )
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("Skill: %s\n", skill.DisplayName)
  fmt.Printf("Latest version: %s\n", skill.LatestVersionID)
  fmt.Printf("Created: %s\n", skill.CreatedAt)
  ```

  ```java Java
  import com.anthropic.models.skills.Skill;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      Skill skill = client.skills().retrieve("skill_01AbCdEfGhIjKlMnOpQrStUv");

      System.out.println("Skill: " + skill.displayName());
      System.out.println("Latest version: " + skill.latestVersionId());
      System.out.println("Created: " + skill.createdAt());
  }
  ```

  ```php PHP
  $client = new Client();

  $skill = $client->skills->retrieve('skill_01AbCdEfGhIjKlMnOpQrStUv');

  echo "Skill: {$skill->displayName}\n";
  echo "Latest version: {$skill->latestVersionID}\n";
  echo "Created: {$skill->createdAt->format(DATE_ATOM)}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  skill = client.skills.retrieve("skill_01AbCdEfGhIjKlMnOpQrStUv")

  puts "Skill: #{skill.display_name}"
  puts "Latest version: #{skill.latest_version_id}"
  puts "Created: #{skill.created_at}"
  ```
</CodeGroup>

### 删除 Skill

删除 Skill 也会移除其所有版本。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -X DELETE "https://api.anthropic.com/v1/skills/skill_01AbCdEfGhIjKlMnOpQrStUv" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant skills delete --skill-id skill_01AbCdEfGhIjKlMnOpQrStUv >/dev/null
  ```

  ```python Python
  client = anthropic.Anthropic()

  client.skills.delete(skill_id="skill_01AbCdEfGhIjKlMnOpQrStUv")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  await client.skills.delete("skill_01AbCdEfGhIjKlMnOpQrStUv");
  ```

  ```csharp C#
  AnthropicClient client = new();

  await client.Skills.Delete("skill_01AbCdEfGhIjKlMnOpQrStUv");
  ```

  ```go Go
  client := anthropic.NewClient()

  _, err := client.Skills.Delete(
  	context.TODO(),
  	"skill_01AbCdEfGhIjKlMnOpQrStUv",
  )
  if err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      client.skills().delete("skill_01AbCdEfGhIjKlMnOpQrStUv");
  }
  ```

  ```php PHP
  $client = new Client();

  $client->skills->delete('skill_01AbCdEfGhIjKlMnOpQrStUv');
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  client.skills.delete("skill_01AbCdEfGhIjKlMnOpQrStUv")
  ```
</CodeGroup>

### 版本管理

Skills 支持版本管理，以便安全地管理更新：

**Anthropic Skills：**

* 版本使用日期格式：`20251013`
* 进行更新时发布新版本
* 指定确切版本以保证稳定性

**自定义 Skills：**

* 自动生成的版本 ID：`skver_01AbCdEfGhIjKlMnOpQrStUv`
* 使用 `"latest"` 始终获取最新版本
* 更新 Skill 文件时创建新版本

新版本是完整的快照，而不是增量：每次都要上传 Skill 的完整文件集。您省略的文件不会被沿用，并且新版本 `SKILL.md` 中的 `name` 必须与 Skill 的现有名称匹配。以下示例重新上传了[创建 Skill](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide#creating-a-skill) 中完整的 `financial_skill/` 包。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  # 创建新版本
  NEW_VERSION=$(curl -X POST "https://api.anthropic.com/v1/skills/skill_01AbCdEfGhIjKlMnOpQrStUv/versions" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -F "files[]=@financial_skill/SKILL.md;filename=financial_skill/SKILL.md" \
    -F "files[]=@financial_skill/analyze.py;filename=financial_skill/analyze.py")

  VERSION_ID=$(echo "$NEW_VERSION" | jq -r '.id')

  # 使用特定版本
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d "{
      \"model\": \"claude-opus-5\",
      \"max_tokens\": 4096,
      \"container\": {
        \"skills\": [{
          \"type\": \"custom\",
          \"skill_id\": \"skill_01AbCdEfGhIjKlMnOpQrStUv\",
          \"version\": \"$VERSION_ID\"
        }]
      },
      \"messages\": [{\"role\": \"user\", \"content\": \"Use updated Skill\"}],
      \"tools\": [{\"type\": \"code_execution_20250825\", \"name\": \"code_execution\"}]
    }"

  # 使用最新版本
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "container": {
        "skills": [{
          "type": "custom",
          "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
          "version": "latest"
        }]
      },
      "messages": [{"role": "user", "content": "Use latest Skill version"}],
      "tools": [{"type": "code_execution_20250825", "name": "code_execution"}]
    }'
  ```

  ```bash CLI
  # 创建新版本
  VERSION_ID=$(ant skills:versions create \
    --skill-id skill_01AbCdEfGhIjKlMnOpQrStUv \
    --file financial_skill.zip \
    --transform id \
    --raw-output)

  # 使用特定版本
  ant messages create <<YAML
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: custom
        skill_id: skill_01AbCdEfGhIjKlMnOpQrStUv
        version: "$VERSION_ID"
  messages:
    - role: user
      content: Use updated Skill
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML

  # 使用最新版本
  ant messages create <<YAML
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: custom
        skill_id: skill_01AbCdEfGhIjKlMnOpQrStUv
        version: latest
  messages:
    - role: user
      content: Use latest Skill version
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  ```

  ```python Python
  from anthropic.lib import files_from_dir

  client = anthropic.Anthropic()

  # 创建新版本

  new_version = client.skills.versions.create(
      skill_id="skill_01AbCdEfGhIjKlMnOpQrStUv",
      files=files_from_dir("financial_skill"),
  )

  # 使用特定版本
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [
              {
                  "type": "custom",
                  "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
                  "version": new_version.id,
              }
          ]
      },
      messages=[{"role": "user", "content": "Use updated Skill"}],
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )

  # 使用最新版本
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [
              {
                  "type": "custom",
                  "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
                  "version": "latest",
              }
          ]
      },
      messages=[{"role": "user", "content": "Use latest Skill version"}],
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )
  ```

  ```typescript TypeScript
  import fs from "node:fs";

  const client = new Anthropic();

  // 从完整 financial_skill/ 包的 zip 文件创建新版本
  const newVersion = await client.skills.versions.create("skill_01AbCdEfGhIjKlMnOpQrStUv", {
    files: [fs.createReadStream("financial_skill.zip")]
  });

  // 使用特定版本
  const specificVersionResponse = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        {
          type: "custom",
          skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
          version: newVersion.id
        }
      ]
    },
    messages: [{ role: "user", content: "Use updated Skill" }],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  });

  // 使用最新版本
  const latestVersionResponse = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        {
          type: "custom",
          skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
          version: "latest"
        }
      ]
    },
    messages: [{ role: "user", content: "Use latest Skill version" }],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  });
  ```

  ```csharp C#
  using Anthropic.Core;
  using Anthropic.Models.Skills.Versions;
  // ...
  AnthropicClient client = new();

  // 创建新版本
  var versionParams = new VersionCreateParams
  {
      Files =
      [
          new BinaryContent
          {
              Stream = File.OpenRead("financial_skill/SKILL.md"),
              FileName = "financial_skill/SKILL.md",
          },
          new BinaryContent
          {
              Stream = File.OpenRead("financial_skill/analyze.py"),
              FileName = "financial_skill/analyze.py",
          },
      ],
  };

  var newVersion = await client.Skills.Versions.Create("skill_01AbCdEfGhIjKlMnOpQrStUv", versionParams);

  // 使用特定版本
  var specificVersionParams = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Custom,
                  SkillID = "skill_01AbCdEfGhIjKlMnOpQrStUv",
                  Version = newVersion.ID,
              },
          ],
      },
      Messages = [new() { Role = Role.User, Content = "Use updated Skill" }],
      Tools = [new CodeExecutionTool20250825()],
  };

  var response = await client.Messages.Create(specificVersionParams);
  Console.WriteLine(response);

  // 使用最新版本
  var latestVersionParams = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Custom,
                  SkillID = "skill_01AbCdEfGhIjKlMnOpQrStUv",
                  Version = "latest",
              },
          ],
      },
      Messages = [new() { Role = Role.User, Content = "Use latest Skill version" }],
      Tools = [new CodeExecutionTool20250825()],
  };

  var latestResponse = await client.Messages.Create(latestVersionParams);
  Console.WriteLine(latestResponse);
  ```

  ```go Go
  client := anthropic.NewClient()

  // 创建新版本
  skillMd, err := os.Open("financial_skill/SKILL.md")
  if err != nil {
  	log.Fatal(err)
  }
  defer skillMd.Close()
  analyzePy, err := os.Open("financial_skill/analyze.py")
  if err != nil {
  	log.Fatal(err)
  }
  defer analyzePy.Close()

  newVersion, err := client.Skills.Versions.New(
  	context.TODO(),
  	"skill_01AbCdEfGhIjKlMnOpQrStUv",
  	anthropic.SkillVersionNewParams{
  		Files: []io.Reader{
  			anthropic.File(skillMd, "financial_skill/SKILL.md", "text/markdown"),
  			anthropic.File(analyzePy, "financial_skill/analyze.py", "text/x-python"),
  		},
  	},
  )
  if err != nil {
  	log.Fatal(err)
  }

  // 使用特定版本
  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeCustom,
  					SkillID: "skill_01AbCdEfGhIjKlMnOpQrStUv",
  					Version: anthropic.String(newVersion.ID),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Use updated Skill")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)

  // 使用最新版本
  latestResponse, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeCustom,
  					SkillID: "skill_01AbCdEfGhIjKlMnOpQrStUv",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Use latest Skill version")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(latestResponse)
  ```

  ```java Java
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.Message;
  import com.anthropic.models.messages.Model;
  import com.anthropic.core.MultipartField;
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;
  import com.anthropic.models.messages.CodeExecutionTool20250825;
  import com.anthropic.models.skills.versions.VersionCreateParams;
  import com.anthropic.models.skills.versions.SkillVersion;
  import java.io.InputStream;
  import java.nio.file.Files;
  import java.nio.file.Path;

  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  // 从完整 financial_skill/ 包的 zip 文件创建新版本
  VersionCreateParams versionParams = VersionCreateParams.builder()
      .addFile(MultipartField.<InputStream>builder()
          .value(Files.newInputStream(Path.of("financial_skill.zip")))
          .filename("financial_skill.zip")
          .contentType("application/zip")
          .build())
      .build();

  SkillVersion newVersion = client.skills().versions()
      .create("skill_01AbCdEfGhIjKlMnOpQrStUv", versionParams);

  // 使用特定版本
  MessageCreateParams specificVersionParams = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(4096L)
      .container(ContainerParams.builder()
          .addSkill(SkillParams.builder()
              .type(SkillParams.Type.CUSTOM)
              .skillId("skill_01AbCdEfGhIjKlMnOpQrStUv")
              .version(newVersion.id())
              .build())
          .build())
      .addUserMessage("Use updated Skill")
      .addTool(CodeExecutionTool20250825.builder().build())
      .build();

  Message response = client.messages().create(specificVersionParams);
  System.out.println(response);

  // 使用最新版本
  MessageCreateParams latestVersionParams = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(4096L)
      .container(ContainerParams.builder()
          .addSkill(SkillParams.builder()
              .type(SkillParams.Type.CUSTOM)
              .skillId("skill_01AbCdEfGhIjKlMnOpQrStUv")
              .version("latest")
              .build())
          .build())
      .addUserMessage("Use latest Skill version")
      .addTool(CodeExecutionTool20250825.builder().build())
      .build();

  Message latestResponse = client.messages().create(latestVersionParams);
  System.out.println(latestResponse);
  ```

  ```php PHP
  use Anthropic\Core\FileParam;
  // ...

  $client = new Client();

  // 创建新版本
  $newVersion = $client->skills->versions->create(
      skillID: 'skill_01AbCdEfGhIjKlMnOpQrStUv',
      files: [
          FileParam::fromResource(
              fopen('financial_skill/SKILL.md', 'r'),
              filename: 'financial_skill/SKILL.md',
              contentType: 'text/markdown',
          ),
          FileParam::fromResource(
              fopen('financial_skill/analyze.py', 'r'),
              filename: 'financial_skill/analyze.py',
              contentType: 'text/x-python',
          ),
      ],
  );

  // 使用特定版本
  $response = $client->messages->create(
      maxTokens: 4096,
      messages: [['role' => 'user', 'content' => 'Use updated Skill']],
      model: 'claude-opus-5',
      container: [
          'skills' => [[
              'type' => 'custom',
              'skillID' => 'skill_01AbCdEfGhIjKlMnOpQrStUv',
              'version' => $newVersion->id
          ]]
      ],
      tools: [['type' => 'code_execution_20250825', 'name' => 'code_execution']]
  );
  echo $response;

  // 使用最新版本
  $latestResponse = $client->messages->create(
      maxTokens: 4096,
      messages: [['role' => 'user', 'content' => 'Use latest Skill version']],
      model: 'claude-opus-5',
      container: [
          'skills' => [[
              'type' => 'custom',
              'skillID' => 'skill_01AbCdEfGhIjKlMnOpQrStUv',
              'version' => 'latest'
          ]]
      ],
      tools: [['type' => 'code_execution_20250825', 'name' => 'code_execution']]
  );
  echo $latestResponse;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 创建新版本
  new_version = client.skills.versions.create(
    "skill_01AbCdEfGhIjKlMnOpQrStUv",
    files: [
      Anthropic::FilePart.new(
        Pathname("financial_skill/SKILL.md"),
        filename: "financial_skill/SKILL.md",
        content_type: "text/markdown"
      ),
      Anthropic::FilePart.new(
        Pathname("financial_skill/analyze.py"),
        filename: "financial_skill/analyze.py",
        content_type: "text/x-python"
      )
    ]
  )

  # 使用特定版本
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [{
        type: "custom",
        skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
        version: new_version.id
      }]
    },
    messages: [{ role: "user", content: "Use updated Skill" }],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  )
  puts response

  # 使用最新版本
  latest_response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [{
        type: "custom",
        skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
        version: "latest"
      }]
    },
    messages: [{ role: "user", content: "Use latest Skill version" }],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  )
  puts latest_response
  ```
</CodeGroup>

有关完整详情，请参阅 [Create Skill Version API 参考](https://platform.claude.com/docs/zh-CN/api/skills/versions/create)。

***

## Skills 的加载方式

当您在容器中指定 Skills 时：

1. **元数据发现：** Claude 在系统提示中看到每个 Skill 的元数据（名称、描述）。
2. **文件加载：** Skill 文件被复制到容器中的 `/skills/{skill-name}/`。该目录是 Skill 的名称（Anthropic Skill 为 `pptx`，自定义 Skill 为 `SKILL.md` 的 `name`），而不是其 `skill_01...` ID。
3. **自动使用：** 当与您的请求相关时，Claude 会自动加载并使用 Skills。
4. **组合：** 多个 Skills 可组合在一起以处理复杂的工作流。

Claude 仅在需要时才加载完整的 Skill 指令。

***

## 用例

Skills 既适用于组织工作，也适用于个人工作。组织使用它们为文档应用品牌格式、围绕公司模板组织笔记和报告，以及运行公司特定的分析流程。个人使用它们来实现自定义文档模板、专用数据管道，以及代码生成或部署规范。

### 示例：财务建模

组合 Excel 和自定义 DCF 分析 Skills：

<CodeGroup>
  ```bash cURL
  # 创建自定义 DCF 分析 Skill
  DCF_SKILL=$(curl -X POST "https://api.anthropic.com/v1/skills" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -F "files[]=@dcf_skill/SKILL.md;filename=dcf_skill/SKILL.md")

  DCF_SKILL_ID=$(echo "$DCF_SKILL" | jq -r '.id')

  # 与 Excel 配合使用以创建财务模型
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d "{
      \"model\": \"claude-opus-5\",
      \"max_tokens\": 4096,
      \"container\": {
        \"skills\": [
          {
            \"type\": \"anthropic\",
            \"skill_id\": \"xlsx\",
            \"version\": \"latest\"
          },
          {
            \"type\": \"custom\",
            \"skill_id\": \"$DCF_SKILL_ID\",
            \"version\": \"latest\"
          }
        ]
      },
      \"messages\": [{
        \"role\": \"user\",
        \"content\": \"Build a DCF valuation model for a SaaS company\"
      }],
      \"tools\": [{
        \"type\": \"code_execution_20250825\",
        \"name\": \"code_execution\"
      }]
    }"
  ```

  ```bash CLI
  # 创建自定义 DCF 分析 Skill
  DCF_SKILL_ID=$(ant skills create \
    --file dcf_skill.zip \
    --transform id \
    --raw-output)

  # 与 Excel 配合使用以创建财务模型
  ant messages create <<YAML
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: anthropic
        skill_id: xlsx
        version: latest
      - type: custom
        skill_id: $DCF_SKILL_ID
        version: latest
  messages:
    - role: user
      content: Build a DCF valuation model for a SaaS company
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  ```

  ```python Python
  from anthropic.lib import files_from_dir

  client = anthropic.Anthropic()

  # 创建自定义 DCF 分析 Skill

  dcf_skill = client.skills.create(
      files=files_from_dir("/path/to/dcf_skill"),
  )

  # 与 Excel 配合使用以创建财务模型
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [
              {"type": "anthropic", "skill_id": "xlsx", "version": "latest"},
              {"type": "custom", "skill_id": dcf_skill.id, "version": "latest"},
          ]
      },
      messages=[
          {
              "role": "user",
              "content": "Build a DCF valuation model for a SaaS company",
          }
      ],
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )
  print(response)
  ```

  ```typescript TypeScript
  import Anthropic, { toFile } from "@anthropic-ai/sdk";
  import fs from "node:fs";

  const client = new Anthropic();

  // 创建自定义 DCF 分析 Skill
  const dcfSkill = await client.skills.create({
    files: [await toFile(fs.createReadStream("dcf_skill.zip"), "dcf_skill.zip")]
  });

  // 与 Excel 配合使用以创建财务模型
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        { type: "anthropic", skill_id: "xlsx", version: "latest" },
        { type: "custom", skill_id: dcfSkill.id, version: "latest" }
      ]
    },
    messages: [
      {
        role: "user",
        content: "Build a DCF valuation model for a SaaS company"
      }
    ],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  });
  console.log(response);
  ```

  ```csharp C#
  using Anthropic.Core;
  // ...
  AnthropicClient client = new();

  // 创建自定义 DCF 分析 Skill
  var dcfSkill = await client.Skills.Create(new SkillCreateParams
  {
      Files =
      [
          new BinaryContent
          {
              Stream = File.OpenRead("dcf_skill/SKILL.md"),
              FileName = "dcf_skill/SKILL.md",
          },
      ],
  });

  // 与 Excel 配合使用以创建财务模型
  var parameters = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "xlsx",
                  Version = "latest",
              },
              new SkillParams
              {
                  Type = SkillParamsType.Custom,
                  SkillID = dcfSkill.ID,
                  Version = "latest",
              },
          ],
      },
      Messages = [new() { Role = Role.User, Content = "Build a DCF valuation model for a SaaS company" }],
      Tools = [new CodeExecutionTool20250825()],
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  // 自定义 DCF 分析 Skill（ID 从 Skills API 创建响应中获取）
  dcfSkillID := "skill_01AbCdEfGhIjKlMnOpQrStUv"

  // 与 Excel 配合使用以创建财务模型
  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "xlsx",
  					Version: anthropic.String("latest"),
  				},
  				{
  					Type:    anthropic.SkillParamsTypeCustom,
  					SkillID: dcfSkillID,
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Build a DCF valuation model for a SaaS company")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;
  import com.anthropic.models.messages.CodeExecutionTool20250825;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      // 自定义 DCF 分析 Skill（ID 从 Skills API 创建响应中获取）
      String dcfSkillId = "skill_01AbCdEfGhIjKlMnOpQrStUv";

      // 与 Excel Skill 配合使用以创建财务模型
      MessageCreateParams params = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .container(ContainerParams.builder()
              .skills(List.of(
                  SkillParams.builder()
                      .type(SkillParams.Type.ANTHROPIC)
                      .skillId("xlsx")
                      .version("latest")
                      .build(),
                  SkillParams.builder()
                      .type(SkillParams.Type.CUSTOM)
                      .skillId(dcfSkillId)
                      .version("latest")
                      .build()
              ))
              .build())
          .addUserMessage("Build a DCF valuation model for a SaaS company")
          .addTool(CodeExecutionTool20250825.builder().build())
          .build();

      Message response = client.messages().create(params);
      System.out.println(response);
  }
  ```

  ```php PHP
  $client = new Client();

  // 自定义 DCF 分析 Skill（ID 从 Skills API 创建响应中获取）
  $dcfSkillId = 'skill_01AbCdEfGhIjKlMnOpQrStUv';

  // 与 Excel 配合使用以创建财务模型
  $message = $client->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Build a DCF valuation model for a SaaS company']
      ],
      model: 'claude-opus-5',
      container: [
          'skills' => [
              ['type' => 'anthropic', 'skillID' => 'xlsx', 'version' => 'latest'],
              ['type' => 'custom', 'skillID' => $dcfSkillId, 'version' => 'latest']
          ]
      ],
      tools: [
          ['type' => 'code_execution_20250825', 'name' => 'code_execution']
      ]
  );
  echo $message;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 创建自定义 DCF 分析 Skill
  dcf_skill = client.skills.create(
    files: [
      Anthropic::FilePart.new(
        Pathname("dcf_skill/SKILL.md"),
        filename: "dcf_skill/SKILL.md",
        content_type: "text/markdown"
      )
    ]
  )

  # 与 Excel 配合使用以创建财务模型
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        { type: "anthropic", skill_id: "xlsx", version: "latest" },
        { type: "custom", skill_id: dcf_skill.id, version: "latest" }
      ]
    },
    messages: [
      { role: "user", content: "Build a DCF valuation model for a SaaS company" }
    ],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  )
  puts response
  ```
</CodeGroup>

***

## 限制与约束

### 请求限制

* **每个请求的最大 Skills 数量：** 20

* **最大 Skill 上传大小：** 30 MB（所有文件合计，未压缩）

* **YAML frontmatter 要求：**

  * `name`：最多 64 个字符，仅限小写字母/数字/连字符，不得包含 XML 标签，不得包含保留字（"anthropic"、"claude"）
  * `description`：最多 1024 个字符，非空，不得包含 XML 标签

### 环境约束

Skills 在代码执行容器中运行，具有以下限制：

* **无网络访问：** 无法进行外部 API 调用
* **无运行时包安装：** 仅可使用预安装的包
* **隔离环境：** 除非您指定现有的容器 ID，否则会创建一个全新的容器

有关可用的包，请参阅[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)。

***

## 最佳实践

### 何时使用多个 Skills

当任务涉及多种文档类型或领域时，组合使用 Skills：

**良好的用例：**

* 数据分析（Excel）+ 演示文稿创建（PowerPoint）
* 报告生成（Word）+ 导出为 PDF
* 自定义领域逻辑 + 文档生成

**应避免：**

* 包含未使用的 Skills（影响性能）

### 版本管理策略

本节中的 SDK 标签页展示了要包含在 Messages 请求中的 `container` 值。cURL 和 CLI 标签页展示了完整的请求。

**对于生产环境：** 固定特定版本，这样 Skill 更新永远不会改变您已部署的行为。如果您省略 `version` 或将其设置为 `"latest"`，请求将使用该 Skill 的最新版本，因此[工作区](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide#workspace-scoped-access)中任何人上传的版本都会立即改变您的生产代理所运行的内容。版本 ID 来自[版本管理](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide#versioning)中的创建版本响应，或来自 [List Skill Versions API](https://platform.claude.com/docs/zh-CN/api/skills/versions/list)。该 ID 始终是字符串，因此即使它看起来像数字，也请在 JSON 或 YAML 中为其加上引号。

<CodeGroup>
  ```bash cURL
  # 固定到特定版本以确保稳定性
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "container": {
        "skills": [{
          "type": "custom",
          "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
          "version": "skver_01AbCdEfGhIjKlMnOpQrStUv"
        }]
      },
      "messages": [{"role": "user", "content": "Analyze the sales data"}],
      "tools": [{"type": "code_execution_20250825", "name": "code_execution"}]
    }'
  ```

  ```bash CLI
  # 固定到特定版本以确保稳定性
  ant messages create <<YAML
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: custom
        skill_id: skill_01AbCdEfGhIjKlMnOpQrStUv
        version: "skver_01AbCdEfGhIjKlMnOpQrStUv"
  messages:
    - role: user
      content: Analyze the sales data
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  ```

  ```python Python
  # 固定到特定版本以确保稳定性
  container = {
      "skills": [
          {
              "type": "custom",
              "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
              "version": "skver_01AbCdEfGhIjKlMnOpQrStUv",
          }
      ]
  }
  ```

  ```typescript TypeScript
  // 固定到特定版本以确保稳定性
  const container: Anthropic.ContainerParams = {
    skills: [
      {
        type: "custom",
        skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
        version: "skver_01AbCdEfGhIjKlMnOpQrStUv"
      }
    ]
  };
  ```

  ```csharp C#
  using Anthropic.Models.Messages;

  // 固定到特定版本以确保稳定性
  var container = new ContainerParams
  {
      Skills =
      [
          new SkillParams
          {
              Type = SkillParamsType.Custom,
              SkillID = "skill_01AbCdEfGhIjKlMnOpQrStUv",
              Version = "skver_01AbCdEfGhIjKlMnOpQrStUv",
          },
      ],
  };
  ```

  ```go Go
  // 固定到特定版本以确保稳定性
  container := anthropic.MessageCreateParamsContainerUnion{
  	OfContainers: &anthropic.ContainerParams{
  		Skills: []anthropic.SkillParams{
  			{
  				Type:    anthropic.SkillParamsTypeCustom,
  				SkillID: "skill_01AbCdEfGhIjKlMnOpQrStUv",
  				Version: anthropic.String("skver_01AbCdEfGhIjKlMnOpQrStUv"),
  			},
  		},
  	},
  }
  ```

  ```java Java
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;

  void main() {
      // 固定到特定版本以确保稳定性
      ContainerParams container = ContainerParams.builder()
          .addSkill(SkillParams.builder()
              .type(SkillParams.Type.CUSTOM)
              .skillId("skill_01AbCdEfGhIjKlMnOpQrStUv")
              .version("skver_01AbCdEfGhIjKlMnOpQrStUv")
              .build())
          .build();
  }
  ```

  ```php PHP
  // 固定到特定版本以确保稳定性
  $container = [
      'skills' => [[
          'type' => 'custom',
          'skillID' => 'skill_01AbCdEfGhIjKlMnOpQrStUv',
          'version' => 'skver_01AbCdEfGhIjKlMnOpQrStUv'
      ]]
  ];
  ```

  ```ruby Ruby
  # 固定到特定版本以确保稳定性
  container = {
    skills: [{
      type: "custom",
      skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
      version: "skver_01AbCdEfGhIjKlMnOpQrStUv"
    }]
  }
  ```
</CodeGroup>

**对于开发环境：** 使用 `latest`，以便在迭代过程中自动获取最新版本。

<CodeGroup>
  ```bash cURL
  # 在活跃开发阶段使用 latest
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "container": {
        "skills": [{
          "type": "custom",
          "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
          "version": "latest"
        }]
      },
      "messages": [{"role": "user", "content": "Analyze the sales data"}],
      "tools": [{"type": "code_execution_20250825", "name": "code_execution"}]
    }'
  ```

  ```bash CLI
  # 在活跃开发阶段使用 latest
  ant messages create <<YAML
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: custom
        skill_id: skill_01AbCdEfGhIjKlMnOpQrStUv
        version: latest
  messages:
    - role: user
      content: Analyze the sales data
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  ```

  ```python Python
  # 在活跃开发阶段使用 latest
  container = {
      "skills": [
          {
              "type": "custom",
              "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
              "version": "latest",
          }
      ]
  }
  ```

  ```typescript TypeScript
  // 在活跃开发阶段使用 latest
  const container: Anthropic.ContainerParams = {
    skills: [
      {
        type: "custom",
        skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
        version: "latest"
      }
    ]
  };
  ```

  ```csharp C#
  using Anthropic.Models.Messages;

  // 在活跃开发阶段使用 latest
  var container = new ContainerParams
  {
      Skills =
      [
          new SkillParams
          {
              Type = SkillParamsType.Custom,
              SkillID = "skill_01AbCdEfGhIjKlMnOpQrStUv",
              Version = "latest",
          },
      ],
  };
  ```

  ```go Go
  // 在活跃开发阶段使用 latest
  container := anthropic.MessageCreateParamsContainerUnion{
  	OfContainers: &anthropic.ContainerParams{
  		Skills: []anthropic.SkillParams{
  			{
  				Type:    anthropic.SkillParamsTypeCustom,
  				SkillID: "skill_01AbCdEfGhIjKlMnOpQrStUv",
  				Version: anthropic.String("latest"),
  			},
  		},
  	},
  }
  ```

  ```java Java
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;

  void main() {
      // 在活跃开发阶段使用 latest
      ContainerParams container = ContainerParams.builder()
          .addSkill(SkillParams.builder()
              .type(SkillParams.Type.CUSTOM)
              .skillId("skill_01AbCdEfGhIjKlMnOpQrStUv")
              .version("latest")
              .build())
          .build();
  }
  ```

  ```php PHP
  // 在活跃开发中使用 latest
  $container = [
      'skills' => [[
          'type' => 'custom',
          'skillID' => 'skill_01AbCdEfGhIjKlMnOpQrStUv',
          'version' => 'latest'
      ]]
  ];
  ```

  ```ruby Ruby
  # 在活跃开发阶段使用 latest
  container = {
    skills: [{
      type: "custom",
      skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
      version: "latest"
    }]
  }
  ```
</CodeGroup>

### 提示缓存注意事项

如果您使用 [prompt caching（提示缓存）](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)，更改容器中的 Skills 列表会破坏缓存。Skills 以固定顺序渲染到系统提示中，因此相同的列表会产生相同的可缓存前缀：

<CodeGroup>
  ```bash cURL
  # Skills 以固定且利于缓存的顺序渲染到系统提示中
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "container": {
        "skills": [
          {"type": "anthropic", "skill_id": "xlsx", "version": "latest"}
        ]
      },
      "messages": [{"role": "user", "content": "Analyze sales data"}],
      "tools": [{"type": "code_execution_20250825", "name": "code_execution"}]
    }'

  # 更改 Skills 列表（[xlsx] 与 [xlsx, pptx]）会改变前缀：导致缓存未命中，而相同的列表则会命中缓存
  curl https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 4096,
      "container": {
        "skills": [
          {"type": "anthropic", "skill_id": "xlsx", "version": "latest"},
          {"type": "anthropic", "skill_id": "pptx", "version": "latest"}
        ]
      },
      "messages": [{"role": "user", "content": "Create a presentation"}],
      "tools": [{"type": "code_execution_20250825", "name": "code_execution"}]
    }'
  ```

  ```bash CLI
  # Skills 以固定且利于缓存的顺序渲染到系统提示中
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: anthropic
        skill_id: xlsx
        version: latest
  messages:
    - role: user
      content: Analyze sales data
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML

  # 更改 Skills 列表（[xlsx] 与 [xlsx, pptx]）会改变前缀：导致缓存未命中，而相同的列表则会命中缓存
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: anthropic
        skill_id: xlsx
        version: latest
      - type: anthropic
        skill_id: pptx
        version: latest
  messages:
    - role: user
      content: Create a presentation
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  # Skills 会以固定的、对缓存友好的顺序渲染到系统提示中
  response1 = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [{"type": "anthropic", "skill_id": "xlsx", "version": "latest"}]
      },
      messages=[{"role": "user", "content": "Analyze sales data"}],
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )

  # 更改 Skills 列表（[xlsx] 与 [xlsx, pptx]）会改变前缀：导致缓存未命中，而相同的列表则会命中缓存
  response2 = client.messages.create(
      model="claude-opus-5",
      max_tokens=4096,
      container={
          "skills": [
              {"type": "anthropic", "skill_id": "xlsx", "version": "latest"},
              {
                  "type": "anthropic",
                  "skill_id": "pptx",
                  "version": "latest",
              },  # prefix change: cache miss
          ]
      },
      messages=[{"role": "user", "content": "Create a presentation"}],
      tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
  )
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  // Skills 以固定且利于缓存的顺序渲染到系统提示中
  const response1 = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [{ type: "anthropic", skill_id: "xlsx", version: "latest" }]
    },
    messages: [{ role: "user", content: "Analyze sales data" }],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  });

  // 更改 Skills 列表（[xlsx] 与 [xlsx, pptx]）会改变前缀：导致缓存未命中，而相同的列表则会缓存命中
  const response2 = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        { type: "anthropic", skill_id: "xlsx", version: "latest" },
        { type: "anthropic", skill_id: "pptx", version: "latest" } // prefix change: cache miss
      ]
    },
    messages: [{ role: "user", content: "Create a presentation" }],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  });
  ```

  ```csharp C#
  AnthropicClient client = new();

  // Skills 以固定且利于缓存的顺序渲染到系统提示中
  var parameters1 = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "xlsx",
                  Version = "latest",
              },
          ],
      },
      Messages = [new() { Role = Role.User, Content = "Analyze sales data" }],
      Tools = [new CodeExecutionTool20250825()],
  };

  var response1 = await client.Messages.Create(parameters1);
  Console.WriteLine(response1);

  // 不同的 Skill 集合（[xlsx] 与 [xlsx, pptx]）= 不同的前缀：缓存未命中（相同集合则缓存命中）
  var parameters2 = new MessageCreateParams
  {
      Model = "claude-opus-5",
      MaxTokens = 4096,
      Container = new ContainerParams
      {
          Skills =
          [
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "xlsx",
                  Version = "latest",
              },
              new SkillParams
              {
                  Type = SkillParamsType.Anthropic,
                  SkillID = "pptx",
                  Version = "latest",
              },
          ],
      },
      Messages = [new() { Role = Role.User, Content = "Create a presentation" }],
      Tools = [new CodeExecutionTool20250825()],
  };

  var response2 = await client.Messages.Create(parameters2);
  Console.WriteLine(response2);
  ```

  ```go Go
  client := anthropic.NewClient()

  // Skills 会以固定且利于缓存的顺序渲染到系统提示中
  response1, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "xlsx",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Analyze sales data")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response1)

  // 更改 Skills 列表（[xlsx] 与 [xlsx, pptx]）会改变前缀：导致缓存未命中，而相同的列表则会命中缓存
  response2, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "xlsx",
  					Version: anthropic.String("latest"),
  				},
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "pptx",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Create a presentation")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(response2)
  ```

  ```java Java
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;
  import com.anthropic.models.messages.CodeExecutionTool20250825;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      // Skills 会以固定且有利于缓存的顺序渲染到系统提示中
      MessageCreateParams params1 = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .container(ContainerParams.builder()
              .skills(List.of(
                  SkillParams.builder()
                      .type(SkillParams.Type.ANTHROPIC)
                      .skillId("xlsx")
                      .version("latest")
                      .build()
              ))
              .build())
          .addUserMessage("Analyze sales data")
          .addTool(CodeExecutionTool20250825.builder().build())
          .build();

      Message response1 = client.messages().create(params1);
      System.out.println(response1);

      // 更改 Skills 列表（[xlsx] 与 [xlsx, pptx]）会改变前缀：导致缓存未命中，而相同的列表则会命中缓存
      MessageCreateParams params2 = MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(4096L)
          .container(ContainerParams.builder()
              .skills(List.of(
                  SkillParams.builder()
                      .type(SkillParams.Type.ANTHROPIC)
                      .skillId("xlsx")
                      .version("latest")
                      .build(),
                  SkillParams.builder()
                      .type(SkillParams.Type.ANTHROPIC)
                      .skillId("pptx")
                      .version("latest")
                      .build()
              ))
              .build())
          .addUserMessage("Create a presentation")
          .addTool(CodeExecutionTool20250825.builder().build())
          .build();

      Message response2 = client.messages().create(params2);
      System.out.println(response2);
  }
  ```

  ```php PHP
  $client = new Client();

  // Skills 以固定且利于缓存的顺序渲染到系统提示中
  $response1 = $client->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Analyze sales data']
      ],
      model: 'claude-opus-5',
      container: [
          'skills' => [
              ['type' => 'anthropic', 'skillID' => 'xlsx', 'version' => 'latest']
          ]
      ],
      tools: [
          ['type' => 'code_execution_20250825', 'name' => 'code_execution']
      ]
  );
  echo $response1;

  // 更改 Skills 列表（[xlsx] 与 [xlsx, pptx]）会改变前缀：导致缓存未命中，而相同的列表则会命中缓存
  $response2 = $client->messages->create(
      maxTokens: 4096,
      messages: [
          ['role' => 'user', 'content' => 'Create a presentation']
      ],
      model: 'claude-opus-5',
      container: [
          'skills' => [
              ['type' => 'anthropic', 'skillID' => 'xlsx', 'version' => 'latest'],
              ['type' => 'anthropic', 'skillID' => 'pptx', 'version' => 'latest']
          ]
      ],
      tools: [
          ['type' => 'code_execution_20250825', 'name' => 'code_execution']
      ]
  );
  echo $response2;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # Skills 以固定且有利于缓存的顺序渲染到系统提示中
  response1 = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [{ type: "anthropic", skill_id: "xlsx", version: "latest" }]
    },
    messages: [{ role: "user", content: "Analyze sales data" }],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  )
  puts response1

  # 更改 Skills 列表（[xlsx] 与 [xlsx, pptx]）会改变前缀：导致缓存未命中，而相同的列表则会缓存命中
  response2 = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 4096,
    container: {
      skills: [
        { type: "anthropic", skill_id: "xlsx", version: "latest" },
        { type: "anthropic", skill_id: "pptx", version: "latest" } # prefix change: cache miss
      ]
    },
    messages: [{ role: "user", content: "Create a presentation" }],
    tools: [{ type: "code_execution_20250825", name: "code_execution" }]
  )
  puts response2
  ```
</CodeGroup>

为获得最佳缓存性能，请在各请求之间保持 Skills 列表（包括其顺序）一致。固定自定义 Skill 版本也有帮助：使用 `"latest"` 时，如果发布的新版本更改了 Skill 的描述，则可能使缓存的前缀失效。

### 错误处理

妥善处理与 Skill 相关的错误：

<CodeGroup>
  ```bash cURL
  # 此错误处理流程不太适合用一次性 shell
  # 命令来表达；使用某个 SDK 选项会更合适。失败的请求
  # 会返回 HTTP 400 及错误 JSON，其 .error.message 会指明
  # Skill 的问题所在。
  ```

  ```bash CLI
  if ! RESULT=$(ant messages create \
    --transform-error error.message \
    --format-error yaml 2>&1 <<'YAML'
  model: claude-opus-5
  max_tokens: 4096
  container:
    skills:
      - type: custom
        skill_id: skill_01AbCdEfGhIjKlMnOpQrStUv
        version: latest
  messages:
    - role: user
      content: Process data
  tools:
    - type: code_execution_20250825
      name: code_execution
  YAML
  ); then
    case "$RESULT" in
      *skill*)
        printf 'Skill error: %s\n' "$RESULT"
        # 处理特定于 skill 的错误
        ;;
      *)
        printf '%s\n' "$RESULT" >&2
        exit 1
        ;;
    esac
  fi
  ```

  ```python Python
  client = anthropic.Anthropic()

  try:
      response = client.messages.create(
          model="claude-opus-5",
          max_tokens=4096,
          container={
              "skills": [
                  {
                      "type": "custom",
                      "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
                      "version": "latest",
                  }
              ]
          },
          messages=[{"role": "user", "content": "Process data"}],
          tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
      )
  except anthropic.BadRequestError as e:
      if "skill" in str(e):
          print(f"Skill error: {e}")
          # 处理特定于 skill 的错误
      else:
          raise
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  try {
    const response = await client.messages.create({
      model: "claude-opus-5",
      max_tokens: 4096,
      container: {
        skills: [
          { type: "custom", skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv", version: "latest" }
        ]
      },
      messages: [{ role: "user", content: "Process data" }],
      tools: [{ type: "code_execution_20250825", name: "code_execution" }]
    });
    console.log(response);
  } catch (error) {
    if (error instanceof Anthropic.BadRequestError && error.message.includes("skill")) {
      console.error(`Skill error: ${error.message}`);
      // 处理技能特定的错误
    } else {
      throw error;
    }
  }
  ```

  ```csharp C#
  using Anthropic.Exceptions;
  // ...
  AnthropicClient client = new();

  try
  {
      var parameters = new MessageCreateParams
      {
          Model = "claude-opus-5",
          MaxTokens = 4096,
          Container = new ContainerParams
          {
              Skills =
              [
                  new SkillParams
                  {
                      Type = SkillParamsType.Custom,
                      SkillID = "skill_01AbCdEfGhIjKlMnOpQrStUv",
                      Version = "latest",
                  },
              ],
          },
          Messages = [new() { Role = Role.User, Content = "Process data" }],
          Tools = [new CodeExecutionTool20250825()],
      };

      var response = await client.Messages.Create(parameters);
      Console.WriteLine(response);
  }
  catch (AnthropicBadRequestException e) when (e.Message.Contains("skill"))
  {
      Console.WriteLine($"Skill error: {e.Message}");
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     "claude-opus-5",
  	MaxTokens: 4096,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeCustom,
  					SkillID: "skill_01AbCdEfGhIjKlMnOpQrStUv",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Process data")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20250825: &anthropic.CodeExecutionTool20250825Param{}},
  	},
  })

  if err != nil {
  	var apierr *anthropic.Error
  	if errors.As(err, &apierr) && apierr.Type() == anthropic.ErrorTypeInvalidRequestError &&
  		strings.Contains(apierr.Error(), "skill") {
  		fmt.Printf("Skill error: %v\n", apierr)
  	} else {
  		log.Fatal(err)
  	}
  	return
  }
  fmt.Println(response)
  ```

  ```java Java
  import com.anthropic.errors.BadRequestException;
  import com.anthropic.models.messages.ContainerParams;
  import com.anthropic.models.messages.SkillParams;
  import com.anthropic.models.messages.CodeExecutionTool20250825;
  // ...
  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      try {
          MessageCreateParams params = MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5)
              .maxTokens(4096L)
              .container(ContainerParams.builder()
                  .addSkill(SkillParams.builder()
                      .type(SkillParams.Type.CUSTOM)
                      .skillId("skill_01AbCdEfGhIjKlMnOpQrStUv")
                      .version("latest")
                      .build())
                  .build())
              .addUserMessage("Process data")
              .addTool(CodeExecutionTool20250825.builder().build())
              .build();

          Message response = client.messages().create(params);
          System.out.println(response);
      } catch (BadRequestException e) {
          if (e.getMessage().contains("skill")) {
              System.err.println("Skill error: " + e.getMessage());
          } else {
              throw e;
          }
      }
  }
  ```

  ```php PHP
  use Anthropic\Core\Exceptions\BadRequestException;

  $client = new Client();

  try {
      $message = $client->messages->create(
          maxTokens: 4096,
          messages: [
              ['role' => 'user', 'content' => 'Process data']
          ],
          model: 'claude-opus-5',
          container: [
              'skills' => [
                  [
                      'type' => 'custom',
                      'skillID' => 'skill_01AbCdEfGhIjKlMnOpQrStUv',
                      'version' => 'latest'
                  ]
              ]
          ],
          tools: [
              ['type' => 'code_execution_20250825', 'name' => 'code_execution']
          ]
      );
      echo $message;
  } catch (BadRequestException $e) {
      if (str_contains($e->getMessage(), 'skill')) {
          echo "Skill error: " . $e->getMessage();
      } else {
          throw $e;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  begin
    response = client.messages.create(
      model: "claude-opus-5",
      max_tokens: 4096,
      container: {
        skills: [
          {
            type: "custom",
            skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
            version: "latest"
          }
        ]
      },
      messages: [{ role: "user", content: "Process data" }],
      tools: [{ type: "code_execution_20250825", name: "code_execution" }]
    )
  rescue Anthropic::Errors::BadRequestError => e
    if e.message.include?("skill")
      puts "Skill error: #{e.message}"
    else
      raise
    end
  end
  ```
</CodeGroup>

***

## 从 `skills-2025-10-02` 迁移

Skills API 已结束 beta 阶段，不再需要 beta 标头。从 `skills-2025-10-02` 迁移是可选的：仍然发送该标头的请求会继续正常工作，并继续返回 beta 响应结构，因此现有集成在您更改之前会一直正常工作。移除该标头会将这些请求切换为本页所记录的结构：

|                | 使用 `skills-2025-10-02`                              | 不使用该标头                                                                                                                   |
| -------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Skill 标签       | `display_title`（最多 64 个字符，每个工作区内唯一）                 | `display_name`（最多 255 个字符，不唯一）；省略时从 `SKILL.md` 的 `name` 派生                                                               |
| 最新版本指针         | `latest_version`，一个纪元微秒字符串，例如 `"1759178010641129"`  | `latest_version_id`，一个版本 ID，例如 `"skver_01AbCdEfGhIjKlMnOpQrStUv"`；`GET /v1/skills/{skill_id}/versions/latest` 可在一次调用中解析它 |
| URL 中的版本标识符    | 纪元微秒字符串                                             | 版本 ID（`skver_...`）。在 beta 下以 `skill_version_` 前缀捕获的 ID 可作为输入被接受。                                                         |
| 版本对象           | 包含 `directory`（始终等于 Skill 的 `name`）                 | 无 `directory` 字段                                                                                                         |
| `source`       | 字符串，`"custom"` 或 `"anthropic"`                      | 对象，例如 `{"type": "custom"}`；示例目录的值为 `"anthropic_example"`                                                                 |
| 列表响应           | `{ data, has_more, next_page }`                     | `{ data, next_page }`；`limit` 从 1 到 1,000（默认 20）                                                                         |
| 版本列表顺序         | 最旧的在前                                               | 最新的在前，默认 `limit` 为 20。一种结构的分页游标在另一种结构上无效。                                                                                |
| 删除 Skill       | 当存在任何版本时返回 400 错误                                   | 删除 Skill 及其所有版本                                                                                                          |
| 删除 Skill 的唯一版本 | 允许，留下一个没有版本的 Skill                                  | 返回 400 错误；请先上传替代版本，或删除该 Skill                                                                                            |
| 上传布局           | 文件必须位于名称与 Skill `name` 匹配的顶层目录内                     | `SKILL.md` 可以位于上传的根目录；无论哪种方式，存储路径都相同                                                                                     |
| 响应类型           | `CreateSkillResponse`、`GetSkillResponse`，以及每个操作一个类型 | `Skill`、`SkillVersion`、`DeletedSkill`、`DeletedSkillVersion`                                                              |

迁移步骤：

1. **移除 beta 标头。** 从您的请求中删除 `anthropic-beta: skills-2025-10-02`。在 SDK 中，调用 `client.skills` 而不是 `client.beta.skills`；继续使用 `client.beta.skills` 仅在[不再发送该标头的 SDK 版本](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide#sdk-beta-namespace)上有效。更早的版本即使没有 `betas` 参数，也会从 `client.beta.skills` 发送该标头。
2. **重命名代码中的字段**：将 `display_title` 改为 `display_name`，将 `latest_version` 改为 `latest_version_id`，并读取 `source.type` 而不是将 `source` 与字符串进行比较。
3. **使用版本 ID。** 在您存储纪元微秒版本的任何地方，改为存储版本的 `id`，或使用 `latest`。Messages 请求中的 Skill 引用接受版本 ID、`latest`，或（对于 Anthropic Skills）目录版本。
4. **检查删除调用。** `DELETE /v1/skills/{skill_id}` 现在会连同 Skill 一起移除每个版本。如果您曾依赖 beta 的拒绝行为作为保护措施，请添加您自己的检查。

<Warning>
  迁移后，`client.skills.delete(skill_id)` 和 `client.beta.skills.delete(skill_id)` 会在一次调用中删除 Skill 及其所有版本。
</Warning>

在 beta 下所有版本都已被删除的 Skill 没有可返回的当前版本：`GET /v1/skills/{skill_id}` 返回 400 错误，并且在您向其上传版本之前，该 Skill 会从列表响应中被省略。您仍然可以删除它。

### SDK beta 命名空间

从 Python SDK 1.2.0、TypeScript SDK 0.122.0、Go SDK 1.68.0、Java SDK 2.59.0、Ruby SDK 1.67.0 和 C# SDK 12.44.0 开始，`client.beta.skills` 不再发送 `skills-2025-10-02`，并返回与 `client.skills` 相同的结构，类型名称带有 `Beta` 前缀（`BetaSkill`、`BetaSkillVersion`、`BetaDeletedSkill`、`BetaDeletedSkillVersion`）。它接受 `betas` 参数，用于仍处于 beta 阶段的 Skills 功能。在 beta Messages 类型中，容器 Skill 引用类型从 `BetaSkill` 重命名为 `BetaContainerSkill`（字段相同：`type`、`skill_id`、`version`）；`BetaSkill` 现在用于命名 Skill 资源，与非 beta 类型中的 `Skill` 和 `ContainerSkill` 相对应。更早的 SDK 版本的类型对应 beta 结构；如果您依赖这些类型，请在迁移之前继续使用更早的版本。

## 数据保留

Agent Skills 不在 ZDR 安排的覆盖范围内。Skill 定义和执行数据根据 Anthropic 的标准数据保留政策进行保留。

有关所有功能的 ZDR 资格，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。

## 审计日志

如果您的组织启用了 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api)，其 [Activity Feed](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed) 会记录使用 Claude API 密钥或从 Claude Console 进行的 Skills 和 Skill 版本的创建与删除操作。在 Compliance API 关闭期间发生的操作不会被记录，且之后无法恢复，因此在依赖此审计跟踪之前，请先[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。

## 后续步骤

<CardGroup cols={3}>
  <Card title="API 参考" icon="book" href="https://platform.claude.com/docs/zh-CN/api/skills/create">
    包含所有端点的完整 API 参考
  </Card>

  <Card title="Skill 编写最佳实践" icon="edit" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices">
    了解如何编写 Claude 能够发现并成功使用的高效 Skills。
  </Card>

  <Card title="代码执行工具" icon="terminal" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool">
    在沙盒容器中运行 Python 和 bash 代码，以分析数据、生成文件并迭代解决方案。
  </Card>
</CardGroup>
