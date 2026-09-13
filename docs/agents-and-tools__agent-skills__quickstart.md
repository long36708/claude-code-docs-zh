---
title: 在 API 中开始使用 Agent Skills
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/quickstart
description: 了解如何在 10 分钟内使用 Agent Skills 通过 Claude API 创建文档。
---

本教程向您展示如何使用 Agent Skills 创建 PowerPoint 演示文稿。您将学习如何启用 Skills、发出请求以及访问生成的文件。

## 前提条件

* 一个 [Claude API 密钥](https://platform.claude.com/settings/keys)或已登录的 [ant CLI](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/authentication)
* 适用于您所用语言的[客户端 SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview)，或 `curl` 和 `jq`
* 对发出 API 请求有基本的了解

## Agent Skills 概述

预构建的 Agent Skills 通过专业知识扩展 Claude 的能力，用于创建文档、分析数据和处理文件等任务。Anthropic 在 API 中提供以下预构建的 Agent Skills：

* **PowerPoint (pptx)：** 创建和编辑演示文稿
* **Excel (xlsx)：** 创建和分析电子表格
* **Word (docx)：** 创建和编辑文档
* **PDF (pdf)：** 生成 PDF 文档

<Note>
  要创建自定义 Skills，请参阅 [Agent Skills Cookbook](https://platform.claude.com/cookbook/skills-notebooks-01-skills-introduction)，其中提供了构建具有特定领域专业知识的自有 Skills 的示例。
</Note>

## 步骤 1：列出可用的 Skills

首先，检查有哪些 Skills 可用。使用 Skills API 列出所有由 Anthropic 管理的 Skills。每个语言选项卡都是一个连续脚本的节选，所有导入和客户端设置都位于顶部：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  # 列出 Anthropic 管理的 Skills
  curl --fail-with-body -sS "https://api.anthropic.com/v1/skills?source=anthropic" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  # 列出由 Anthropic 管理的 Skills
  ant skills list --source anthropic
  ```

  ```python Python
  # 列出 Anthropic 管理的 Skills
  skills = client.skills.list(source="anthropic")

  for skill in skills.data:
      print(f"{skill.id}: {skill.display_name}")
  ```

  ```typescript TypeScript
  // 列出由 Anthropic 管理的 Skills
  const skills = await client.skills.list({ source: "anthropic" });

  for (const skill of skills.data) {
    console.log(`${skill.id}: ${skill.display_name}`);
  }
  ```

  ```csharp C#
  // 列出 Anthropic 托管的 Skills
  var skills = await client.Skills.List(new SkillListParams { Source = "anthropic" });

  foreach (var skill in skills.Items)
  {
      Console.WriteLine($"{skill.ID}: {skill.DisplayName}");
  }
  ```

  ```go Go
  // 列出 Anthropic 托管的 Skills
  skills, err := client.Skills.List(ctx, anthropic.SkillListParams{
  	Source: anthropic.String("anthropic"),
  })
  if err != nil {
  	panic(err)
  }

  for _, skill := range skills.Data {
  	fmt.Printf("%s: %s\n", skill.ID, skill.DisplayName)
  }
  ```

  ```java Java
  // 列出 Anthropic 托管的 Skills
  SkillListPage skills = client.skills().list(
      SkillListParams.builder().source("anthropic").build()
  );

  for (Skill skill : skills.data()) {
      IO.println(skill.id() + ": " + skill.displayName());
  }
  ```

  ```php PHP
  // 列出 Anthropic 托管的 Skills
  $skills = $client->skills->list(source: 'anthropic');

  foreach ($skills->getItems() as $skill) {
      echo "{$skill->id}: {$skill->displayName}\n";
  }
  ```

  ```ruby Ruby
  # 列出 Anthropic 托管的 Skills
  skills = client.skills.list(source: "anthropic")

  skills.data.each do |skill|
    puts "#{skill.id}: #{skill.display_name}"
  end
  ```
</CodeGroup>

您会看到以下 Skills：`pptx`、`xlsx`、`docx` 和 `pdf`。

此 API 返回每个 Skill 的元数据：其名称和描述。Claude 在启动时加载这些元数据，以确定哪些 Skills 可用。这是 **"progressive disclosure"（渐进式披露）** 的第一层，在这一层中，Claude 发现 Skills，但尚未加载其完整指令。

## 步骤 2：创建演示文稿

使用 PowerPoint Skill 创建一个关于可再生能源的演示文稿。在 Messages API 中使用 `container` 参数指定 Skills：

<CodeGroup>
  ```bash cURL
  # 使用 PowerPoint Skill 创建消息
  response=$(
    curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
      -H "content-type: application/json" \
      -H "x-api-key: $ANTHROPIC_API_KEY" \
      -H "anthropic-version: 2023-06-01" \
      -d @- <<'EOF'
  {
    "model": "claude-opus-5",
    "max_tokens": 16000,
    "container": {
      "skills": [{"type": "anthropic", "skill_id": "pptx", "version": "latest"}]
    },
    "messages": [
      {"role": "user", "content": "Create a presentation about renewable energy with 5 slides"}
    ],
    "tools": [{"type": "code_execution_20260521", "name": "code_execution"}]
  }
  EOF
  )
  ```

  ```bash CLI
  # 使用 PowerPoint Skill 创建消息
  response=$(ant messages create --format json <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  container:
    skills:
      - type: anthropic
        skill_id: pptx
        version: latest
  messages:
    - role: user
      content: Create a presentation about renewable energy with 5 slides
  tools:
    - type: code_execution_20260521
      name: code_execution
  YAML
  )
  ```

  ```python Python
  # 使用 PowerPoint Skill 创建消息
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      container={
          "skills": [{"type": "anthropic", "skill_id": "pptx", "version": "latest"}]
      },
      messages=[
          {
              "role": "user",
              "content": "Create a presentation about renewable energy with 5 slides",
          }
      ],
      tools=[{"type": "code_execution_20260521", "name": "code_execution"}],
  )

  print(f"stop_reason={response.stop_reason}, blocks={len(response.content)}")
  ```

  ```typescript TypeScript
  // 使用 PowerPoint Skill 创建消息
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 16000,
    container: {
      skills: [{ type: "anthropic", skill_id: "pptx", version: "latest" }],
    },
    messages: [
      {
        role: "user",
        content: "Create a presentation about renewable energy with 5 slides",
      },
    ],
    tools: [{ type: "code_execution_20260521", name: "code_execution" }],
  });

  console.log(
    `stop_reason=${response.stop_reason}, blocks=${response.content.length}`,
  );
  ```

  ```csharp C#
  // 使用 PowerPoint Skill 创建消息
  var response = await client.Messages.Create(new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 16000,
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
      Messages =
      [
          new MessageParam
          {
              Role = Role.User,
              Content = "Create a presentation about renewable energy with 5 slides",
          },
      ],
      Tools = [new CodeExecutionTool20260521()],
  });

  Console.WriteLine($"stop_reason={response.StopReason?.Raw()}, blocks={response.Content.Count}");
  ```

  ```go Go
  // 使用 PowerPoint Skill 创建消息
  response, err := client.Messages.New(ctx, anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 16000,
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
  		anthropic.NewUserMessage(
  			anthropic.NewTextBlock("Create a presentation about renewable energy with 5 slides"),
  		),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{OfCodeExecutionTool20260521: &anthropic.CodeExecutionTool20260521Param{}},
  	},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Printf("stop_reason=%s, blocks=%d\n", response.StopReason, len(response.Content))
  ```

  ```java Java
  // 使用 PowerPoint Skill 创建消息
  Message response = client.messages().create(
      MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(16000)
          .container(
              ContainerParams.builder()
                  .addSkill(
                      SkillParams.builder()
                          .type(SkillParams.Type.ANTHROPIC)
                          .skillId("pptx")
                          .version("latest")
                          .build()
                  )
                  .build()
          )
          .addUserMessage("Create a presentation about renewable energy with 5 slides")
          .addTool(CodeExecutionTool20260521.builder().build())
          .build()
  );

  IO.println(
      "stop_reason=" + response.stopReason().orElse(null)
          + ", blocks=" + response.content().size()
  );
  ```

  ```php PHP
  // 使用 PowerPoint Skill 创建消息
  $response = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 16000,
      container: [
          'skills' => [['type' => 'anthropic', 'skillID' => 'pptx', 'version' => 'latest']],
      ],
      messages: [
          [
              'role' => 'user',
              'content' => 'Create a presentation about renewable energy with 5 slides',
          ],
      ],
      tools: [['type' => 'code_execution_20260521', 'name' => 'code_execution']],
  );

  printf("stop_reason=%s, blocks=%d\n", $response->stopReason, count($response->content));
  ```

  ```ruby Ruby
  # 使用 PowerPoint Skill 创建消息
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 16_000,
    container: {
      skills: [{type: "anthropic", skill_id: "pptx", version: "latest"}]
    },
    messages: [
      {
        role: "user",
        content: "Create a presentation about renewable energy with 5 slides"
      }
    ],
    tools: [{type: "code_execution_20260521", name: "code_execution"}]
  )

  puts "stop_reason=#{response.stop_reason}, blocks=#{response.content.length}"
  ```
</CodeGroup>

该请求包含以下部分：

* **`model`：** 一个[支持代码执行工具的模型](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#compatibility)
* **`container.skills`：** 指定 Claude 可以使用哪些 Skills
* **`type: "anthropic"`：** 表示这是一个由 Anthropic 管理的 Skill
* **`skill_id: "pptx"`：** PowerPoint Skill 的标识符
* **`version: "latest"`：** Skill 版本设置为最新发布的版本
* **`tools`：** 启用代码执行（Skills 所必需）

<Note>
  这些示例使用 `code_execution_20260521` 工具版本，步骤 3 的代码会解析当前工具版本返回的结果类型。Skills 也适用于较旧的[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)版本，例如 `code_execution_20250825`：任何当前的代码执行工具版本都满足 Skills 的要求。如果您使用不同的版本，请使用代码执行工具页面上列出的工具 `type`。
</Note>

当您发出此请求时，Claude 会自动将您的任务与相关的 Skill 进行匹配。由于您请求的是演示文稿，Claude 判断 PowerPoint Skill 是相关的，并加载其完整指令：这是渐进式披露的第二层。然后 Claude 运行该 Skill 的代码来创建您的演示文稿。

## 步骤 3：下载创建的文件

演示文稿是在代码执行容器中创建的，并保存为文件。步骤 2 的 `response` 包含一个带有文件 ID 的文件引用。提取文件 ID 并使用 Files API 下载该文件。示例将其保存到您的系统临时目录：

<CodeGroup>
  ```bash cURL
  # 提取文件 ID。代码执行工具通过其 Bash 子工具运行 Skill 的代码，
  # 生成的文件以 bash_code_execution_output 项的形式
  # 出现在 bash_code_execution_tool_result 块中。
  file_id=$(jq -r '
    last(
      .content[]
      | select(.type == "bash_code_execution_tool_result")
      | .content
      | select(.type == "bash_code_execution_result")
      | .content[].file_id
    ) // empty
  ' <<<"$response")

  if [[ -n "$file_id" ]]; then
    # 下载文件并保存
    output_path="${TMPDIR:-/tmp}/renewable_energy.pptx"
    curl --fail-with-body -sS "https://api.anthropic.com/v1/files/$file_id/content" \
      -H "x-api-key: $ANTHROPIC_API_KEY" \
      -H "anthropic-version: 2023-06-01" \
      -o "$output_path"
    echo "Presentation saved to $output_path"
  fi
  ```

  ```bash CLI
  # 提取文件 ID。代码执行工具通过其 Bash 子工具运行 Skill 的代码，
  # 生成的文件以 bash_code_execution_output 项的形式
  # 出现在 bash_code_execution_tool_result 块中。
  file_id=$(jq -r '
    last(
      .content[]
      | select(.type == "bash_code_execution_tool_result")
      | .content
      | select(.type == "bash_code_execution_result")
      | .content[].file_id
    ) // empty
  ' <<<"$response")

  if [[ -n "$file_id" ]]; then
    # 下载文件并保存
    output_path="${TMPDIR:-/tmp}/renewable_energy.pptx"
    ant files download --file-id "$file_id" --output "$output_path"
    echo "Presentation saved to $output_path"
  fi
  ```

  ```python Python
  # 提取文件 ID。代码执行工具通过其 Bash 子工具运行 Skill 的代码，
  # 生成的文件以 bash_code_execution_output 项的形式
  # 出现在 bash_code_execution_tool_result 块中。
  file_id = None
  for block in response.content:
      if block.type == "bash_code_execution_tool_result":
          if block.content.type == "bash_code_execution_result":
              for output in block.content.content:
                  file_id = output.file_id

  if file_id:
      # 下载文件并保存
      output_path = Path(tempfile.gettempdir()) / "renewable_energy.pptx"
      file_content = client.files.download(file_id=file_id)
      file_content.write_to_file(output_path)
      print(f"Presentation saved to {output_path}")
  ```

  ```typescript TypeScript
  // 提取文件 ID。代码执行工具通过其 Bash 子工具运行 Skill 的代码，
  // 生成的文件以 bash_code_execution_output 项的形式
  // 出现在 bash_code_execution_tool_result 块中。
  let fileId: string | undefined;
  for (const block of response.content) {
    if (
      block.type === "bash_code_execution_tool_result" &&
      block.content.type === "bash_code_execution_result"
    ) {
      for (const output of block.content.content) {
        fileId = output.file_id;
      }
    }
  }

  if (fileId) {
    // 下载文件并保存
    const outputPath = path.join(os.tmpdir(), "renewable_energy.pptx");
    const fileContent = await client.files.download(fileId);
    await fs.writeFile(outputPath, Buffer.from(await fileContent.arrayBuffer()));
    console.log(`Presentation saved to ${outputPath}`);
  }
  ```

  ```csharp C#
  // 提取文件 ID。代码执行工具通过其 Bash 子工具运行 Skill 的代码，
  // 生成的文件以 bash_code_execution_output 项的形式
  // 出现在 bash_code_execution_tool_result 块中。
  string? fileId = null;
  foreach (var block in response.Content)
  {
      if (block.TryPickBashCodeExecutionToolResult(out var bashResult)
          && bashResult.Content.TryPickBashCodeExecutionResultBlock(out var bashResultBlock))
      {
          foreach (var output in bashResultBlock.Content)
          {
              fileId = output.FileID;
          }
      }
  }

  if (fileId is not null)
  {
      // 下载文件并保存
      var outputPath = Path.Combine(Path.GetTempPath(), "renewable_energy.pptx");
      using var download = await client.Files.Download(fileId);
      await using var source = await download.ReadAsStream();
      await using var destination = File.Create(outputPath);
      await source.CopyToAsync(destination);
      Console.WriteLine($"Presentation saved to {outputPath}");
  }
  ```

  ```go Go
  // 提取文件 ID。代码执行工具通过其 Bash 子工具运行 Skill 的代码，
  // 生成的文件以 bash_code_execution_output 项的形式
  // 出现在 bash_code_execution_tool_result 块中。
  var fileID string
  for _, block := range response.Content {
  	switch result := block.AsAny().(type) {
  	case anthropic.BashCodeExecutionToolResultBlock:
  		if result.Content.Type == "bash_code_execution_result" {
  			for _, output := range result.Content.Content {
  				fileID = output.FileID
  			}
  		}
  	}
  }

  if fileID != "" {
  	// 下载文件并保存
  	outputPath := filepath.Join(os.TempDir(), "renewable_energy.pptx")
  	fileContent, err := client.Files.Download(ctx, fileID)
  	if err != nil {
  		panic(err)
  	}
  	defer fileContent.Body.Close()
  	outFile, err := os.Create(outputPath)
  	if err != nil {
  		panic(err)
  	}
  	defer outFile.Close()
  	if _, err := io.Copy(outFile, fileContent.Body); err != nil {
  		panic(err)
  	}
  	fmt.Printf("Presentation saved to %s\n", outputPath)
  }
  ```

  ```java Java
  // 提取文件 ID。代码执行工具通过其 Bash 子工具运行 Skill 的代码，
  // 生成的文件以 bash_code_execution_output 项的形式
  // 出现在 bash_code_execution_tool_result 块中。
  String fileId = null;
  for (ContentBlock block : response.content()) {
      if (block.isBashCodeExecutionToolResult()) {
          var content = block.asBashCodeExecutionToolResult().content();
          if (content.isBashCodeExecutionResultBlock()) {
              for (var output : content.asBashCodeExecutionResultBlock().content()) {
                  fileId = output.fileId();
              }
          }
      }
  }

  if (fileId != null) {
      // 下载文件并保存
      Path outputPath = Files.createTempFile("renewable_energy", ".pptx");
      try (HttpResponse fileContent = client.files().download(fileId)) {
          Files.copy(fileContent.body(), outputPath, StandardCopyOption.REPLACE_EXISTING);
      }
      IO.println("Presentation saved to " + outputPath);
  }
  ```

  ```php PHP
  // 提取文件 ID。代码执行工具通过其 Bash 子工具运行 Skill 的代码，
  // 生成的文件以 bash_code_execution_output 项的形式
  // 出现在 bash_code_execution_tool_result 块中。
  $fileId = null;
  foreach ($response->content as $block) {
      if ($block->type !== 'bash_code_execution_tool_result') {
          continue;
      }
      $resultBlock = $block->content;
      if ($resultBlock->type !== 'bash_code_execution_result') {
          continue;
      }
      foreach ($resultBlock->content as $output) {
          $fileId = $output->fileID;
      }
  }

  if ($fileId !== null) {
      // 下载文件并保存
      $outputPath = sys_get_temp_dir() . '/renewable_energy.pptx';
      $fileContent = $client->files->download($fileId);
      file_put_contents($outputPath, $fileContent);
      echo "Presentation saved to {$outputPath}\n";
  }
  ```

  ```ruby Ruby
  # 提取文件 ID。代码执行工具通过其 Bash 子工具运行 Skill 的代码，
  # 生成的文件以 bash_code_execution_output 项的形式
  # 出现在 bash_code_execution_tool_result 块中。
  file_id = nil
  response.content.each do |block|
    next unless block.type == :bash_code_execution_tool_result

    if block.content[:type].to_s == "bash_code_execution_result"
      Array(block.content[:content]).each { |output| file_id = output[:file_id] }
    end
  end

  if file_id
    # 下载文件并保存
    output_path = File.join(Dir.tmpdir, "renewable_energy.pptx")
    file_content = client.files.download(file_id)
    File.binwrite(output_path, file_content.read)
    puts "Presentation saved to #{output_path}"
  end
  ```
</CodeGroup>

<Note>
  有关处理生成文件的完整详细信息，请参阅代码执行工具文档中的[检索生成的文件](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#retrieve-generated-files)。
</Note>

## 尝试更多示例

尝试以下变体：

### 创建电子表格

<CodeGroup>
  ```bash cURL
  curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 16000,
      "container": {
        "skills": [{"type": "anthropic", "skill_id": "xlsx", "version": "latest"}]
      },
      "messages": [
        {"role": "user", "content": "Create a quarterly sales tracking spreadsheet with sample data"}
      ],
      "tools": [{"type": "code_execution_20260521", "name": "code_execution"}]
    }'
  ```

  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  container:
    skills:
      - type: anthropic
        skill_id: xlsx
        version: latest
  messages:
    - role: user
      content: Create a quarterly sales tracking spreadsheet with sample data
  tools:
    - type: code_execution_20260521
      name: code_execution
  YAML
  ```

  ```python Python
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      container={
          "skills": [{"type": "anthropic", "skill_id": "xlsx", "version": "latest"}]
      },
      messages=[
          {
              "role": "user",
              "content": "Create a quarterly sales tracking spreadsheet with sample data",
          }
      ],
      tools=[{"type": "code_execution_20260521", "name": "code_execution"}],
  )
  ```

  ```typescript TypeScript
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 16000,
    container: {
      skills: [{ type: "anthropic", skill_id: "xlsx", version: "latest" }]
    },
    messages: [
      {
        role: "user",
        content: "Create a quarterly sales tracking spreadsheet with sample data"
      }
    ],
    tools: [{ type: "code_execution_20260521", name: "code_execution" }]
  });
  ```

  ```csharp C#
  var response = await client.Messages.Create(
      new MessageCreateParams
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 16000,
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
          Messages =
          [
              new MessageParam
              {
                  Role = Role.User,
                  Content = "Create a quarterly sales tracking spreadsheet with sample data",
              },
          ],
          Tools = [new CodeExecutionTool20260521()],
      }
  );
  ```

  ```go Go
  response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 16000,
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
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Create a quarterly sales tracking spreadsheet with sample data")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{
  			OfCodeExecutionTool20260521: &anthropic.CodeExecutionTool20260521Param{},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  Message response = client.messages().create(
      MessageCreateParams.builder()
          .model(CLAUDE_OPUS_5)
          .maxTokens(16000)
          .container(
              ContainerParams.builder()
                  .addSkill(
                      SkillParams.builder()
                          .type(ANTHROPIC)
                          .skillId("xlsx")
                          .version("latest")
                          .build()
                  )
                  .build()
          )
          .addUserMessage("Create a quarterly sales tracking spreadsheet with sample data")
          .addTool(CodeExecutionTool20260521.builder().build())
          .build()
  );

  ```

  ```php PHP
  $response = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 16000,
      container: [
          'skills' => [
              ['type' => 'anthropic', 'skillID' => 'xlsx', 'version' => 'latest'],
          ],
      ],
      messages: [
          [
              'role' => 'user',
              'content' => 'Create a quarterly sales tracking spreadsheet with sample data',
          ],
      ],
      tools: [['type' => 'code_execution_20260521', 'name' => 'code_execution']],
  );
  ```

  ```ruby Ruby
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 16_000,
    container: {
      skills: [{type: "anthropic", skill_id: "xlsx", version: "latest"}]
    },
    messages: [
      {
        role: "user",
        content: "Create a quarterly sales tracking spreadsheet with sample data"
      }
    ],
    tools: [{type: "code_execution_20260521", name: "code_execution"}]
  )
  ```
</CodeGroup>

### 创建 Word 文档

<CodeGroup>
  ```bash cURL
  curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 16000,
      "container": {
        "skills": [{"type": "anthropic", "skill_id": "docx", "version": "latest"}]
      },
      "messages": [
        {"role": "user", "content": "Write a 2-page report on the benefits of renewable energy"}
      ],
      "tools": [{"type": "code_execution_20260521", "name": "code_execution"}]
    }'
  ```

  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  container:
    skills:
      - type: anthropic
        skill_id: docx
        version: latest
  messages:
    - role: user
      content: Write a 2-page report on the benefits of renewable energy
  tools:
    - type: code_execution_20260521
      name: code_execution
  YAML
  ```

  ```python Python
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      container={
          "skills": [{"type": "anthropic", "skill_id": "docx", "version": "latest"}]
      },
      messages=[
          {
              "role": "user",
              "content": "Write a 2-page report on the benefits of renewable energy",
          }
      ],
      tools=[{"type": "code_execution_20260521", "name": "code_execution"}],
  )
  ```

  ```typescript TypeScript
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 16000,
    container: {
      skills: [{ type: "anthropic", skill_id: "docx", version: "latest" }]
    },
    messages: [
      {
        role: "user",
        content: "Write a 2-page report on the benefits of renewable energy"
      }
    ],
    tools: [{ type: "code_execution_20260521", name: "code_execution" }]
  });
  ```

  ```csharp C#
  var response = await client.Messages.Create(
      new MessageCreateParams
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 16000,
          Container = new ContainerParams
          {
              Skills =
              [
                  new SkillParams
                  {
                      Type = SkillParamsType.Anthropic,
                      SkillID = "docx",
                      Version = "latest",
                  },
              ],
          },
          Messages =
          [
              new MessageParam
              {
                  Role = Role.User,
                  Content = "Write a 2-page report on the benefits of renewable energy",
              },
          ],
          Tools = [new CodeExecutionTool20260521()],
      }
  );
  ```

  ```go Go
  response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 16000,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "docx",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Write a 2-page report on the benefits of renewable energy")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{
  			OfCodeExecutionTool20260521: &anthropic.CodeExecutionTool20260521Param{},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  Message response = client.messages().create(
      MessageCreateParams.builder()
          .model(CLAUDE_OPUS_5)
          .maxTokens(16000)
          .container(
              ContainerParams.builder()
                  .addSkill(
                      SkillParams.builder()
                          .type(ANTHROPIC)
                          .skillId("docx")
                          .version("latest")
                          .build()
                  )
                  .build()
          )
          .addUserMessage("Write a 2-page report on the benefits of renewable energy")
          .addTool(CodeExecutionTool20260521.builder().build())
          .build()
  );

  ```

  ```php PHP
  $response = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 16000,
      container: [
          'skills' => [
              ['type' => 'anthropic', 'skillID' => 'docx', 'version' => 'latest'],
          ],
      ],
      messages: [
          [
              'role' => 'user',
              'content' => 'Write a 2-page report on the benefits of renewable energy',
          ],
      ],
      tools: [['type' => 'code_execution_20260521', 'name' => 'code_execution']],
  );
  ```

  ```ruby Ruby
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 16_000,
    container: {
      skills: [{type: "anthropic", skill_id: "docx", version: "latest"}]
    },
    messages: [
      {
        role: "user",
        content: "Write a 2-page report on the benefits of renewable energy"
      }
    ],
    tools: [{type: "code_execution_20260521", name: "code_execution"}]
  )
  ```
</CodeGroup>

### 生成 PDF

<CodeGroup>
  ```bash cURL
  curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 16000,
      "container": {
        "skills": [{"type": "anthropic", "skill_id": "pdf", "version": "latest"}]
      },
      "messages": [
        {"role": "user", "content": "Generate a PDF invoice template"}
      ],
      "tools": [{"type": "code_execution_20260521", "name": "code_execution"}]
    }'
  ```

  ```bash CLI
  ant messages create <<'YAML'
  model: claude-opus-5
  max_tokens: 16000
  container:
    skills:
      - type: anthropic
        skill_id: pdf
        version: latest
  messages:
    - role: user
      content: Generate a PDF invoice template
  tools:
    - type: code_execution_20260521
      name: code_execution
  YAML
  ```

  ```python Python
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=16000,
      container={
          "skills": [{"type": "anthropic", "skill_id": "pdf", "version": "latest"}]
      },
      messages=[
          {
              "role": "user",
              "content": "Generate a PDF invoice template",
          }
      ],
      tools=[{"type": "code_execution_20260521", "name": "code_execution"}],
  )
  ```

  ```typescript TypeScript
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 16000,
    container: {
      skills: [{ type: "anthropic", skill_id: "pdf", version: "latest" }]
    },
    messages: [
      {
        role: "user",
        content: "Generate a PDF invoice template"
      }
    ],
    tools: [{ type: "code_execution_20260521", name: "code_execution" }]
  });
  ```

  ```csharp C#
  var response = await client.Messages.Create(
      new MessageCreateParams
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 16000,
          Container = new ContainerParams
          {
              Skills =
              [
                  new SkillParams
                  {
                      Type = SkillParamsType.Anthropic,
                      SkillID = "pdf",
                      Version = "latest",
                  },
              ],
          },
          Messages =
          [
              new MessageParam
              {
                  Role = Role.User,
                  Content = "Generate a PDF invoice template",
              },
          ],
          Tools = [new CodeExecutionTool20260521()],
      }
  );
  ```

  ```go Go
  response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 16000,
  	Container: anthropic.MessageCreateParamsContainerUnion{
  		OfContainers: &anthropic.ContainerParams{
  			Skills: []anthropic.SkillParams{
  				{
  					Type:    anthropic.SkillParamsTypeAnthropic,
  					SkillID: "pdf",
  					Version: anthropic.String("latest"),
  				},
  			},
  		},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("Generate a PDF invoice template")),
  	},
  	Tools: []anthropic.ToolUnionParam{
  		{
  			OfCodeExecutionTool20260521: &anthropic.CodeExecutionTool20260521Param{},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  Message response = client.messages().create(
      MessageCreateParams.builder()
          .model(CLAUDE_OPUS_5)
          .maxTokens(16000)
          .container(
              ContainerParams.builder()
                  .addSkill(
                      SkillParams.builder()
                          .type(ANTHROPIC)
                          .skillId("pdf")
                          .version("latest")
                          .build()
                  )
                  .build()
          )
          .addUserMessage("Generate a PDF invoice template")
          .addTool(CodeExecutionTool20260521.builder().build())
          .build()
  );

  ```

  ```php PHP
  $response = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 16000,
      container: [
          'skills' => [
              ['type' => 'anthropic', 'skillID' => 'pdf', 'version' => 'latest'],
          ],
      ],
      messages: [
          [
              'role' => 'user',
              'content' => 'Generate a PDF invoice template',
          ],
      ],
      tools: [['type' => 'code_execution_20260521', 'name' => 'code_execution']],
  );
  ```

  ```ruby Ruby
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 16_000,
    container: {
      skills: [{type: "anthropic", skill_id: "pdf", version: "latest"}]
    },
    messages: [
      {
        role: "user",
        content: "Generate a PDF invoice template"
      }
    ],
    tools: [{type: "code_execution_20260521", name: "code_execution"}]
  )
  ```
</CodeGroup>

## 后续步骤

<CardGroup cols={2}>
  <Card title="Skill 编写最佳实践" icon="edit" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices">
    了解如何编写 Claude 能够发现并成功使用的高效 Skills。
  </Card>

  <Card title="通过 API 使用 Agent Skills" icon="book" href="https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide">
    了解如何使用 Agent Skills 通过 API 扩展 Claude 的能力。
  </Card>

  <Card title="创建自定义 Skills" icon="code" href="https://platform.claude.com/docs/zh-CN/api/skills/create">
    上传您自己的 Skills 以用于专门任务。
  </Card>

  <Card title="在 Claude Code 中使用 Skills" icon="terminal" href="https://code.claude.com/docs/en/skills">
    了解 Claude Code 中的 Skills。
  </Card>

  <Card title="Agent Skills Cookbook" icon="book" href="https://platform.claude.com/cookbook/skills-notebooks-01-skills-introduction">
    探索示例 Skills 和实现模式。
  </Card>
</CardGroup>
