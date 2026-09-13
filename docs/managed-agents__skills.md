---
title: 技能
url: https://platform.claude.com/docs/zh-CN/managed-agents/skills
description: 在 Claude Managed Agents 中为智能体附加预构建或自定义技能，为其提供可复用的、基于文件系统的专业知识，以支持特定领域的工作流。
---

"Skills"（技能）是可复用的、基于文件系统的资源，可为您的智能体提供特定领域的专业知识：工作流、上下文和最佳实践，将通用智能体转变为专家。您添加的每个技能都会对会话的 "context window"（上下文窗口）产生少量开销，添加有助于模型使用该技能的指令和元数据。请在 [Agent Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview) 概述中了解更多信息。

技能通过两种方式到达您的智能体：通过智能体的 `skills` 数组附加它们，或者[从挂载到会话上的 GitHub 仓库加载它们](https://platform.claude.com/docs/zh-CN/managed-agents/skills#load-skills-from-a-github-repository)。附加的技能分为两种类型。所有技能的工作方式相同：当它们与任务相关时，您的智能体会自动调用它们。

* **预构建的 Anthropic 技能：** 常见的文档任务，例如 PowerPoint、Excel、Word 和 PDF 处理（`pptx`、`xlsx`、`docx`、`pdf`）。
* **自定义技能：** 您编写并上传到工作区的技能。

要了解如何编写自定义技能，请参阅 [Agent Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview) 和[技能编写最佳实践](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices)。要将自定义技能上传到您的工作区，请参阅[创建自定义技能](https://platform.claude.com/docs/zh-CN/managed-agents/skills#create-a-custom-skill)。

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

## 创建自定义技能

自定义技能是一个包含 `SKILL.md` 文件以及任何支持文件的目录，以 zip 压缩包或单独文件的形式上传到您的工作区。创建技能会返回 `skill_*` ID，您在将其附加到智能体时会引用该 ID。Anthropic 预构建技能已在每个工作区中可用，无需执行此步骤。如果只使用预构建技能，请跳至[将技能附加到智能体](https://platform.claude.com/docs/zh-CN/managed-agents/skills#attach-skills-to-an-agent)。

这些示例省略了可选的 `display_name` 字段，因此技能的显示名称派生自 `SKILL.md` 中的 `name` 字段。显式的 `display_name` 最多可包含 255 个字符，并且在您的工作区内不需要唯一。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -X POST "https://api.anthropic.com/v1/skills" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -F "files[]=@example_skill.zip"
  ```

  ```bash CLI
  ant skills create --file example_skill.zip
  ```

  ```python Python
  import anthropic
  from anthropic.lib import files_from_dir

  client = anthropic.Anthropic()

  skill = client.skills.create(
      files=files_from_dir("example_skill"),
  )

  print(f"Created skill: {skill.id}")
  print(f"Latest version: {skill.latest_version_id}")
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";
  import { toFile } from "@anthropic-ai/sdk";
  import fs from "node:fs";

  const client = new Anthropic();

  const skill = await client.skills.create({
    files: [await toFile(fs.createReadStream("example_skill.zip"), "example_skill.zip")]
  });

  console.log(`Created skill: ${skill.id}`);
  console.log(`Latest version: ${skill.latest_version_id}`);
  ```

  ```csharp C#
  using System.IO;
  using Anthropic;
  using Anthropic.Models.Skills;

  AnthropicClient client = new();

  var parameters = new SkillCreateParams
  {
      Files = [
          new FileStream("example_skill.zip", FileMode.Open, FileAccess.Read)
      ],
  };

  var skill = await client.Skills.Create(parameters);

  Console.WriteLine($"Created skill: {skill.ID}");
  Console.WriteLine($"Latest version: {skill.LatestVersionID}");
  ```

  ```go Go
  package main

  import (
  	"context"
  	"fmt"
  	"io"
  	"log"
  	"os"

  	"github.com/anthropics/anthropic-sdk-go"
  )

  func main() {
  	client := anthropic.NewClient()

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

  	fmt.Printf("Created skill: %s\n", skill.ID)
  	fmt.Printf("Latest version: %s\n", skill.LatestVersionID)
  }
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.core.MultipartField;
  import com.anthropic.models.skills.Skill;
  import com.anthropic.models.skills.SkillCreateParams;
  import java.io.IOException;
  import java.io.InputStream;
  import java.nio.file.Files;
  import java.nio.file.Path;

  void main() throws IOException {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      SkillCreateParams params = SkillCreateParams.builder()
          .addFile(MultipartField.<InputStream>builder()
              .value(Files.newInputStream(Path.of("example_skill.zip")))
              .filename("example_skill.zip")
              .contentType("application/zip")
              .build())
          .build();

      Skill skill = client.skills().create(params);

      IO.println("Created skill: " + skill.id());
      IO.println("Latest version: " + skill.latestVersionId());
  }
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Core\FileParam;

  $client = new Client();

  $skill = $client->skills->create(
      files: [
          FileParam::fromResource(fopen('example_skill.zip', 'r')),
      ],
  );

  echo "Created skill: {$skill->id}\n";
  echo "Latest version: {$skill->latestVersionID}\n";
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::Client.new

  skill = client.skills.create(
    files: [
      File.open("example_skill.zip", "rb")
    ]
  )

  puts "Created skill: #{skill.id}"
  puts "Latest version: #{skill.latest_version_id}"
  ```
</CodeGroup>

要列出、检索、删除自定义技能以及管理其版本，请参阅[管理自定义技能](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide#managing-custom-skills)。有关完整的请求和响应模式，请参阅[创建技能 API 参考](https://platform.claude.com/docs/zh-CN/api/skills/create)。技能包直接上传到 Skills API，而不是通过 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传。

## 将技能附加到智能体

在创建智能体时附加技能。每个[会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)最多支持 500 个技能，按会话中所有智能体去重后的集合计数（请参阅[多智能体编排](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)）。

<Note>
  挂载更多技能会增加会话沙箱启动所需的时间。请仅附加每个智能体完成其任务所需的技能。
</Note>

`skills` 数组中的每个条目使用以下字段：

| 字段         | 描述                                                                                                                                                                      |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`     | 预构建技能使用 `anthropic`，工作区编写的技能使用 `custom`。                                                                                                                                |
| `skill_id` | 技能标识符。对于 Anthropic 技能，使用短名称（例如 `xlsx`）。对于自定义技能，使用创建时返回的 `skill_*` ID（请参阅[创建自定义技能](https://platform.claude.com/docs/zh-CN/managed-agents/skills#create-a-custom-skill)）。 |
| `version`  | 固定到特定版本或使用 `latest`。可选。省略时默认为 `latest`。适用于 Anthropic 技能和自定义技能。                                                                                                          |

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  agent=$(curl -sS https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    --json @- <<'EOF'
  {
    "name": "Financial Analyst",
    "model": "claude-opus-5",
    "system": "You are a financial analysis agent.",
    "skills": [
      {"type": "anthropic", "skill_id": "xlsx"},
      {"type": "custom", "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv", "version": "latest"}
    ]
  }
  EOF
  )
  ```

  <MultiFileExample language="cli" label="CLI">
    ```bash CLI
    ant beta:agents create < agent.yaml
    ```

    <File filename="agent.yaml">
      ```yaml
      name: Financial Analyst
      model: claude-opus-5
      system: You are a financial analysis agent.
      skills:
        - type: anthropic
          skill_id: xlsx
        - type: custom
          skill_id: skill_01AbCdEfGhIjKlMnOpQrStUv
          version: latest
      ```
    </File>
  </MultiFileExample>

  ```python Python
  agent = client.beta.agents.create(
      name="Financial Analyst",
      model="claude-opus-5",
      system="You are a financial analysis agent.",
      skills=[
          {
              "type": "anthropic",
              "skill_id": "xlsx",
          },
          {
              "type": "custom",
              "skill_id": "skill_01AbCdEfGhIjKlMnOpQrStUv",
              "version": "latest",
          },
      ],
  )
  ```

  ```typescript TypeScript
  const agent = await client.beta.agents.create({
    name: "Financial Analyst",
    model: "claude-opus-5",
    system: "You are a financial analysis agent.",
    skills: [
      {
        type: "anthropic",
        skill_id: "xlsx"
      },
      {
        type: "custom",
        skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv",
        version: "latest"
      }
    ]
  });
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Agents;

  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Financial Analyst",
      Model = BetaManagedAgentsModel.ClaudeOpus5,
      System = "You are a financial analysis agent.",
      Skills =
      [
          new BetaManagedAgentsAnthropicSkillParams { Type = BetaManagedAgentsAnthropicSkillParamsType.Anthropic, SkillID = "xlsx" },
          new BetaManagedAgentsCustomSkillParams { Type = BetaManagedAgentsCustomSkillParamsType.Custom, SkillID = "skill_01AbCdEfGhIjKlMnOpQrStUv", Version = "latest" },
      ],
  });
  ```

  ```go Go
  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "Financial Analyst",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: anthropic.BetaManagedAgentsModelClaudeOpus5,
  	},
  	System: anthropic.String("You are a financial analysis agent."),
  	Skills: []anthropic.BetaManagedAgentsSkillParamsUnion{
  		{OfAnthropic: &anthropic.BetaManagedAgentsAnthropicSkillParams{
  			SkillID: "xlsx",
  			Type:    anthropic.BetaManagedAgentsAnthropicSkillParamsTypeAnthropic,
  		}},
  		{OfCustom: &anthropic.BetaManagedAgentsCustomSkillParams{
  			SkillID: "skill_01AbCdEfGhIjKlMnOpQrStUv",
  			Type:    anthropic.BetaManagedAgentsCustomSkillParamsTypeCustom,
  			Version: anthropic.String("latest"),
  		}},
  	},
  })
  if err != nil {
  	panic(err)
  }
  _ = agent
  ```

  ```java Java
  import com.anthropic.models.beta.agents.*;

  var agent = client.beta().agents().create(
      AgentCreateParams.builder()
          .name("Financial Analyst")
          .model(BetaManagedAgentsModel.CLAUDE_OPUS_5)
          .system("You are a financial analysis agent.")
          .addSkill(
              BetaManagedAgentsAnthropicSkillParams.builder()
                  .type(BetaManagedAgentsAnthropicSkillParams.Type.ANTHROPIC)
                  .skillId("xlsx")
                  .build()
          )
          .addSkill(
              BetaManagedAgentsCustomSkillParams.builder()
                  .type(BetaManagedAgentsCustomSkillParams.Type.CUSTOM)
                  .skillId("skill_01AbCdEfGhIjKlMnOpQrStUv")
                  .version("latest")
                  .build()
          )
          .build()
  );
  ```

  ```php PHP
  $agent = $client->beta->agents->create(
      name: 'Financial Analyst',
      model: 'claude-opus-5',
      system: 'You are a financial analysis agent.',
      skills: [
          ['type' => 'anthropic', 'skillID' => 'xlsx'],
          ['type' => 'custom', 'skillID' => 'skill_01AbCdEfGhIjKlMnOpQrStUv', 'version' => 'latest'],
      ],
  );
  ```

  ```ruby Ruby
  agent = client.beta.agents.create(
    name: "Financial Analyst",
    model: "claude-opus-5",
    system_: "You are a financial analysis agent.",
    skills: [
      {type: "anthropic", skill_id: "xlsx"},
      {type: "custom", skill_id: "skill_01AbCdEfGhIjKlMnOpQrStUv", version: "latest"}
    ]
  )
  ```
</CodeGroup>

## 从 GitHub 仓库加载技能

技能也可以存放在您的代码库中。当会话通过 [`github_repository` 资源](https://platform.claude.com/docs/zh-CN/managed-agents/github)挂载仓库时，会在会话启动时扫描仓库根目录下的 `.claude/skills` 目录，在其中找到的每个技能都会对智能体可用。无需上传，也无需在智能体的 `skills` 数组中添加条目。智能体可以看到每个已发现技能的名称、描述及其在沙箱中的路径，并在任务匹配时读取该技能的 `SKILL.md`，包括技能附带的任何脚本和资源。发现机制依赖于[智能体工具集](https://platform.claude.com/docs/zh-CN/managed-agents/tools)中智能体的 `read` 工具，该工具默认启用；禁用了 `read` 的智能体不会加载仓库技能。

<Warning>
  仓库技能是智能体指令，因此挂载的仓库是您智能体信任边界的一部分。任何能够向仓库提交代码的人（已合并的外部拉取请求、被入侵的依赖项、贡献者）都可以添加或更改技能，平台会在会话启动时加载它而没有审查步骤，并且 `bash` 和 `web_fetch` 等会话工具会赋予这些指令真实的影响力。请仅挂载您信任的仓库，并在挂载接受外部贡献的仓库之前审查 `.claude/skills`。
</Warning>

<Note>
  仓库技能发现在云沙箱中运行。[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)不支持 GitHub 仓库资源。
</Note>

发现机制仅在 `.claude/skills/<skill-name>/SKILL.md` 这一确切位置查找技能，即仓库根目录下一级目录深度：

* `your-repo/`

  * `.claude/`

    * `skills/`

      * `code-review/`
        * `SKILL.md`

      * `release-process/`

        * `SKILL.md`
        * `scripts/`
          * `run_checks.sh`

  * `src/`

不符合此布局的位置不会在会话启动时被发现：

* `.claude/skills/SKILL.md`：一个没有技能目录包裹的 `SKILL.md`
* `.claude/skills/tools/code-review/SKILL.md`：嵌套超过一级目录深度
* `skills/code-review/SKILL.md`：位于 `.claude` 之外的 `skills` 目录

位于仓库其他位置的 `.claude/skills` 目录（例如在某个包的子目录内）不会在会话启动时被公布；当智能体读取该子树下的文件时，这些技能仍然可能出现。

仓库技能使用与您上传的自定义技能相同的 `SKILL.md` 格式。有关格式和编写指南，请参阅 [Agent Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview) 和[技能编写最佳实践](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices)。

要从仓库加载技能，请创建一个挂载该仓库的会话。这与[访问 GitHub](https://platform.claude.com/docs/zh-CN/managed-agents/github#token-permissions) 中展示的请求相同；`mount_path` 是可选的，默认为 `/workspace/<repo-name>`：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  session_id=$(curl -fsS https://api.anthropic.com/v1/sessions \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    --data @- <<JSON | jq -r '.id'
  {
    "agent": "$agent_id",
    "environment_id": "$environment_id",
    "resources": [
      {
        "type": "github_repository",
        "url": "https://github.com/org/repo",
        "mount_path": "/workspace/repo",
        "authorization_token": "ghp_your_github_token"
      }
    ]
  }
  JSON
  )
  ```

  ```bash CLI
  SESSION_ID=$(ant beta:sessions create \
    --agent "$AGENT_ID" \
    --environment-id "$ENVIRONMENT_ID" \
    --transform id --raw-output <<'EOF'
  resources:
    - type: github_repository
      url: https://github.com/org/repo
      mount_path: /workspace/repo
      authorization_token: ghp_your_github_token
  EOF
  )
  ```

  ```python Python
  session = client.beta.sessions.create(
      agent=agent.id,
      environment_id=environment.id,
      resources=[
          {
              "type": "github_repository",
              "url": "https://github.com/org/repo",
              "mount_path": "/workspace/repo",
              "authorization_token": "ghp_your_github_token",
          },
      ],
  )
  ```

  ```typescript TypeScript
  const session = await client.beta.sessions.create({
    agent: agent.id,
    environment_id: environment.id,
    resources: [
      {
        type: "github_repository",
        url: "https://github.com/org/repo",
        mount_path: "/workspace/repo",
        authorization_token: "ghp_your_github_token",
      },
    ],
  });
  ```

  ```csharp C#
  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = agent.ID,
      EnvironmentID = environment.ID,
      Resources =
      [
          new BetaManagedAgentsGitHubRepositoryResourceParams
          {
              Type = "github_repository",
              Url = "https://github.com/org/repo",
              MountPath = "/workspace/repo",
              AuthorizationToken = "ghp_your_github_token",
          },
      ],
  });
  ```

  ```go Go
  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent:         anthropic.BetaSessionNewParamsAgentUnion{OfString: anthropic.String(agent.ID)},
  	EnvironmentID: environment.ID,
  	Resources: []anthropic.BetaSessionNewParamsResourceUnion{
  		{
  			OfGitHubRepository: &anthropic.BetaManagedAgentsGitHubRepositoryResourceParams{
  				Type:               anthropic.BetaManagedAgentsGitHubRepositoryResourceParamsTypeGitHubRepository,
  				URL:                "https://github.com/org/repo",
  				MountPath:          anthropic.String("/workspace/repo"),
  				AuthorizationToken: "ghp_your_github_token",
  			},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var session = client.beta().sessions().create(SessionCreateParams.builder()
      .agent(agent.id())
      .environmentId(environment.id())
      .addResource(BetaManagedAgentsGitHubRepositoryResourceParams.builder()
          .type(BetaManagedAgentsGitHubRepositoryResourceParams.Type.GITHUB_REPOSITORY)
          .url("https://github.com/org/repo")
          .mountPath("/workspace/repo")
          .authorizationToken("ghp_your_github_token")
          .build())
      .build());
  ```

  ```php PHP
  $session = $client->beta->sessions->create(
      agent: $agent->id,
      environmentID: $environment->id,
      resources: [
          [
              'type' => 'github_repository',
              'url' => 'https://github.com/org/repo',
              'mountPath' => '/workspace/repo',
              'authorizationToken' => 'ghp_your_github_token',
          ],
      ],
  );
  ```

  ```ruby Ruby
  session = client.beta.sessions.create(
    agent: agent.id,
    environment_id: environment.id,
    resources: [
      {
        type: "github_repository",
        url: "https://github.com/org/repo",
        mount_path: "/workspace/repo",
        authorization_token: "ghp_your_github_token"
      }
    ]
  )
  ```
</CodeGroup>

对于私有仓库，资源的 `authorization_token` 必须具有访问该仓库的权限。这与任何仓库挂载所使用的个人访问令牌流程相同；请参阅[访问 GitHub](https://platform.claude.com/docs/zh-CN/managed-agents/github#token-permissions)。

已发现的技能遵循仓库的检出状态：当资源设置了 `checkout` 分支或提交时使用该分支或提交，否则使用仓库的默认分支。扫描仅在会话启动时运行一次。会话进行中推送的提交不会被获取；要加载更新后的技能，请启动新会话。

仓库技能与通过智能体的 `skills` 数组附加的技能协同工作。如果某个仓库技能与某个附加技能同名，或与来自另一个已挂载仓库的技能同名，则两者都可用；每个技能都会以其各自的路径公布。

## 后续步骤

<CardGroup cols={2}>
  <Card title="云环境设置" icon="settings" href="https://platform.claude.com/docs/zh-CN/managed-agents/environments">
    为您的会话自定义云沙箱。
  </Card>

  <Card title="通过 API 使用 Agent Skills" icon="code" href="https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide">
    了解如何使用 Agent Skills 通过 API 扩展 Claude 的能力。
  </Card>

  <Card title="Files API" icon="file" href="https://platform.claude.com/docs/zh-CN/build-with-claude/files">
    上传文件一次，即可在多个 API 请求中引用。
  </Card>

  <Card title="在 API 中开始使用 Agent Skills" icon="graduation-cap" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/quickstart">
    了解如何在 10 分钟内使用 Agent Skills 通过 Claude API 创建文档。
  </Card>
</CardGroup>
