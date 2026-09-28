---
title: 使用 ant apply 以代码形式管理资源
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/apply
description: 将智能体、环境、技能、记忆存储和部署声明为代码仓库中的文件，并使用 ant apply 使 API 中的资源与这些文件保持同步。
---

`ant apply` 根据文件创建和更新 Claude API 资源，包括智能体、环境、技能、记忆存储和部署。这些文件存放在您的代码仓库中，其变更与代码经过相同的审查流程。您在文件中描述每个资源，运行 `ant apply`，然后批准它显示的计划。接着提交它写入的 `claude-lock.json`，这样下次运行时会更新相同的资源，而不会创建新资源。

要安装 CLI 并进行身份验证，请参阅 [CLI 快速入门](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/quickstart)。`ant apply` 需要 CLI 1.30.0 或更高版本。

## 应用您的第一个智能体

将智能体编写为 `agents/` 下的 Markdown 文件，然后应用它：

<CodeGroup>
  <CodeGroupItem>
    ```bash CLI
    ant apply agents/summarizer.md
    ```

    <File filename="agents/summarizer.md">
      ```markdown
      ---
      name: Summarizer
      model: claude-opus-5-5
      tools:
        - type: agent_toolset_20260401
      ---

      You are a helpful assistant that writes concise summaries.
      ```
    </File>
  </CodeGroupItem>
</CodeGroup>

frontmatter（前置元数据）包含智能体的配置（即[定义您的智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)中的字段），正文则是其系统提示。`ant apply` 根据文件路径（此处为 `agents/` 目录）[推断](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/apply#kind-inference)该文件是一个智能体。

在交互式终端中，`ant apply` 会打印计划并等待您批准：

```text Output wrap
First apply  ./claude-lock.json does not exist yet and will be created

Resources will be created with
  credentials   API key (--api-key / ANTHROPIC_API_KEY)
  host          api.anthropic.com
  organization  1b0c2a4d-6c1f-4f0e-9a57-2e8d1c3b4a5f
  workspace     wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ

Preview  ./claude-lock.json (new)

± Name                    Plan
+ ./agents/summarizer.md  create

Resources  + 1 to create

Apply these changes? (y)es / (n)o / (d)etails y

Apply  ./claude-lock.json

± Name                    Status
+ ./agents/summarizer.md  created    agent_011CYm1BLqPXpQRk5khsSXrs

Resources  + 1 created

State written to ./claude-lock.json
```

输入 `d` 可先查看详细信息，包括每个新资源的字段，或每项更新的逐字段差异。`--dry-run` 会打印该详细计划后退出，不做任何更改。

要更改智能体，请编辑文件并再次运行 `ant apply`。此时计划显示的是更新，而不是创建。

## 提交 claude-lock.json

首次运行 `ant apply` 时，它会在您运行命令的目录中写入 `claude-lock.json`，即 lockfile（锁文件），因此请从仓库根目录运行。该文件记录了每个文件所创建资源的 ID，以及这些资源所在的组织和工作区：

```json claude-lock.json
{
  "version": 1,
  "origin": {
    "base_url": "https://api.anthropic.com",
    "organization_id": "1b0c2a4d-6c1f-4f0e-9a57-2e8d1c3b4a5f",
    "workspace_id": "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  },
  "resources": {
    "./agents/summarizer.md": {
      "kind": "agent",
      "id": "agent_011CYm1BLqPXpQRk5khsSXrs",
      "version": "1",
      "hash": "d23251c8d99b3613a64f3f8d87f5fad4",
      "remote_hash": "1b771bee5bdbf600a5ad972fdac32d94"
    }
  }
}
```

请将它与您的文件一起提交。下一次运行（无论是在您的机器上还是在 CI 中）正是通过它找到这些资源，而不会再次创建；您也可以从中读取智能体的 ID 来[启动会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)。其中的两个哈希值分别是上次发送内容和 API 返回内容的指纹。后续运行正是借此发现文件已被编辑，或资源在这些文件之外被更改。

## 扩展为项目

您也可以用文件以声明方式定义其他资源。每个文件包含您会发送到该类型创建端点的请求体：

* [环境](https://platform.claude.com/docs/zh-CN/managed-agents/environments)是 `environments/` 中的 YAML 文件。
* [记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory)是 `memory_stores/` 中的 YAML 文件。
* [部署](https://platform.claude.com/docs/zh-CN/managed-agents/scheduled-deployments)是 `deployments/` 中的 Markdown 文件：frontmatter 是请求体，正文文字则成为启动每个会话的消息。
* [技能](https://platform.claude.com/docs/zh-CN/managed-agents/skills)是根目录下包含 `SKILL.md` 的目录，按惯例放在 `skills/` 下，并作为一个整体包上传。

除技能外，任何资源都可以用 YAML、JSON 或 Markdown 编写。在 Markdown 中，frontmatter 是请求体，正文文字则填充该类型的文本字段，即智能体的 `system`、环境或记忆存储的 `description`，或部署的第一条消息。

资源之间通过路径相互引用。凡是 API 需要另一个资源 ID 的地方，都改为填写指向该资源文件的相对路径。在此项目中，reviewer 智能体在 `skills` 下列出 `../skills/pr-summary`，lead 智能体在其成员列表中列出 `./reviewer.md`，部署则通过路径指定其智能体、环境和记忆存储。`ant apply` 会按依赖顺序创建这些资源，并填入真实 ID。该项目包含六个文件：

<FileExplorer>
  <File filename="agents/reviewer.md">
    ```markdown
    ---
    name: Code reviewer
    model: claude-opus-5-5
    tools:
      - type: agent_toolset_20260401
    skills:
      - ../skills/pr-summary
    ---

    You review pull requests for correctness, security, and readability.
    ```
  </File>

  <File filename="agents/lead.md">
    ```markdown
    ---
    name: Engineering lead
    model: claude-opus-5-5
    multiagent:
      type: coordinator
      agents:
        - ./reviewer.md
    ---

    You coordinate engineering work. Delegate code review to the reviewer.
    ```
  </File>

  <File filename="skills/pr-summary/SKILL.md">
    ```markdown
    ---
    name: pr-summary
    description: Summarize a pull request's changes and risks in the team's review format.
    ---

    # PR summary

    List what changed, why, and anything a reviewer should look at closely, in three short sections.
    ```
  </File>

  <File filename="environments/cloud.yaml">
    ```yaml
    # yaml-language-server: $schema=https://platform.claude.com/schemas/ant/beta/environment.json
    name: review-env
    description: Cloud container with unrestricted networking for review sessions.
    config:
      type: cloud
      networking:
        type: unrestricted
    ```
  </File>

  <File filename="memory_stores/review-notes.yaml">
    ```yaml
    # yaml-language-server: $schema=https://platform.claude.com/schemas/ant/beta/memory_store.json
    name: Review notes
    description: Recurring issues and house-style decisions the reviewer has recorded between runs.
    ```
  </File>

  <File filename="deployments/nightly.md">
    ```markdown
    ---
    name: Nightly review
    agent: ../agents/reviewer.md # the API's agent field: sent as {type: agent, id, version}
    environment_id: ../environments/cloud.yaml # sent as the environment's ID
    resources:
      - path: ../memory_stores/review-notes.yaml
        access: read_write
    schedule:
      type: cron
      expression: "0 3 * * *"
      timezone: America/Los_Angeles
    ---

    Review any open pull requests. Start with the oldest.
    ```
  </File>
</FileExplorer>

应用整个目录：

```bash CLI
ant apply .
```

之后，`claude-lock.json` 会为项目中的每个文件各记录一个条目。

这些文件通过相对路径相互指向。`ant apply` 会将智能体和技能引用固定到它刚刚应用的版本，因此编辑 `reviewer.md` 或该技能后，所有引用它们的资源都会在同一次运行中更新。路径也可以用在对象内部，例如部署的 `resources` 条目，此时 `access` 等其他键会保留。

要指向不由这些文件管理的资源，请改为填写其 ID（`agent_...`、`skill_...`）。其他任何内容（例如 `{type: anthropic, skill_id: xlsx}`）都会按原样发送到 API。技能引用也可以是 `https://github.com/<owner>/<repo>/tree/<branch>/<dir>` 形式的 GitHub URL，例如 Anthropic 开源[技能仓库](https://github.com/anthropics/skills)中的某个目录。`ant apply` 会下载并上传该目录，并将其固定到解析出的提交，直到您使用 `--upgrade` 运行为止（对于私有仓库，请设置 `GITHUB_TOKEN`）。

### ant apply 如何推断文件的类型

`ant apply` 遍历目录时，会依次检查以下规则，并按第一个匹配的规则确定每个文件的类型：

1. 文件中的顶层 `type` 字段。
2. 文件的直接父目录：`agents/`、`environments/`、`memory_stores/` 或 `deployments/`。
3. 以类型名开头的文件名，例如 `environment_staging.md`。

不匹配上述任何规则的文件（例如 README 和 CI 配置）会被跳过，除非您在命令行中显式指定了它们。对于显式指定但不匹配任何规则的文件，Markdown 文件会被视为智能体，YAML 或 JSON 文件则会报错。

## 编辑并重新应用

不带参数运行 `ant apply` 时，它会协调 lockfile 跟踪的所有文件。在终端中，它还会列出 lockfile 所在目录下未被跟踪的资源文件，并询问是否添加。如果从文件中删除某个字段，且 API 允许清除该字段，资源上的该字段就会被清除。您从未设置过的字段，或 API 无法清除的字段，会保留当前值。

如果某个资源在这些文件之外（例如在 Claude Console 中）被编辑、归档或删除，计划末尾会显示 `This plan cannot be applied:` 及原因，随后命令以 `refusing to apply` 退出。传入 `--force` 可覆盖该编辑或创建替代资源。

删除文件后，其资源仍会保留，并显示警告；使用 `--prune` 可移除该资源（将其归档，技能则会被删除）。因此，重命名文件相当于声明一个新资源，旧资源会一直保留，直到您执行清理。

`ant apply` 无法接管您在 Console 中或通过 `ant beta:agents create` 创建的资源。它只管理 lockfile 中记录的资源，因此应用一个描述现有智能体的文件会创建第二个智能体。如果您通过 **Export as code** 从 Console 下载了智能体，下载内容会附带其自己的 `claude-lock.json`，因此应用它会更新您在 Console 中构建的资源。

## 在 CI 中运行 ant apply

在没有终端的环境中，`ant apply` 会打印计划，然后以 `cannot ask for confirmation without a terminal; re-run with --yes to apply, or --dry-run to see the plan only` 停止。请按如下方式设置 CI：

* 合并后，在默认分支上运行 `ant apply --yes .`，并指定项目目录。不带参数的 `ant apply --yes` 只会协调 lockfile 已跟踪的文件，会跳过新添加的文件。
* 在拉取请求中，运行 `ant apply --dry-run .` 为审查者打印计划。该命令仅供参考，即使计划被阻止也会以 0 退出。
* 在作业结束时提交更新后的 `claude-lock.json`，即使应用步骤中途失败也要提交，因为部分应用仍会记录已创建的资源。
* 每次只运行一个应用，因为 lockfile 没有任何锁定机制。
* 使用 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)（工作负载身份联合）而非存储的 API 密钥进行身份验证，所用身份须能访问 `claude-lock.json` 中记录的组织和工作区。如果凭据解析到其他组织或工作区，`ant apply` 会拒绝使用。

有关完整的 GitHub Actions 工作流，请参阅 [CLI README 中的 CI 示例](https://github.com/anthropics/anthropic-cli#in-ci)。

## 标志

| 标志                   | 作用                                                                                                   |
| -------------------- | ---------------------------------------------------------------------------------------------------- |
| `--dry-run`          | 打印计划后退出，不应用更改，也不写入 lockfile。即使计划被阻止也会以 0 退出。                                                         |
| `--yes`              | 无需确认即应用。没有终端时必须使用。                                                                                   |
| `--force`            | 即使资源在这些文件之外被更改、归档或删除，也照常应用。                                                                          |
| `--prune`            | 移除 lockfile 中存在但已不再由任何文件声明的资源。                                                                       |
| `--upgrade`          | 重新解析通过 GitHub URL 引用的技能；否则这些技能会保持固定在 lockfile 中记录的提交上。                                               |
| `--lock-file <path>` | 使用指定的 lockfile，而不是从当前目录向上查找。请为每个组织或工作区各保留一个 lockfile：如果 lockfile 中的组织或工作区与您的凭据不匹配，`ant apply` 会拒绝使用。 |
| `--verbose`, `-v`    | 在计划中显示未更改的资源和完整的字段值。                                                                                 |

## 后续步骤

<CardGroup cols={3}>
  <Card title="启动会话" icon="terminal" href="https://platform.claude.com/docs/zh-CN/managed-agents/sessions">
    通过 CLI 或 SDK 运行您应用的智能体
  </Card>

  <Card title="定时部署" icon="clock" href="https://platform.claude.com/docs/zh-CN/managed-agents/scheduled-deployments">
    部署字段、运行历史和暂停
  </Card>

  <Card title="CLI 脚本编写与自动化" icon="code" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/scripting">
    脚本编写模式以及在 Claude Code 中的使用
  </Card>
</CardGroup>
