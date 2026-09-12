> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 创建和分发 plugin marketplace

> 构建和托管 plugin marketplace，以在团队和社区中分发 Claude Code 扩展。

**plugin marketplace** 是一个目录，让你能够将 plugins 分发给他人。Marketplace 提供集中式发现、版本跟踪、自动更新以及对多种源类型（包括 git 存储库和本地路径）的支持。本指南展示了如何创建自己的 marketplace，与你的团队或社区共享 plugins。

想要从现有 marketplace 安装 plugins？请参阅[发现和安装预构建的 plugins](/docs/zh-CN/discover-plugins)。

<h2 id="overview">
  概述
</h2>

创建和分发 marketplace 涉及：

1. **创建 plugins**：使用 skills、agents、hooks、MCP servers 或 LSP servers 构建一个或多个 plugins。本指南假设你已经有要分发的 plugins；有关如何创建 plugins 的详细信息，请参阅[创建 plugins](/docs/zh-CN/plugins)。
2. **创建 marketplace 文件**：定义一个 `marketplace.json`，列出你的 plugins 及其位置。请参阅[创建 marketplace 文件](#create-the-marketplace-file)。
3. **托管 marketplace**：推送到 GitHub、GitLab 或其他 git 主机。请参阅[托管和分发 marketplaces](#host-and-distribute-marketplaces)。
4. **与用户共享**：用户使用 `/plugin marketplace add` 添加你的 marketplace 并安装单个 plugins。请参阅[发现和安装 plugins](/docs/zh-CN/discover-plugins)。

一旦你的 marketplace 上线，你可以通过推送更改到你的存储库来更新它。用户使用 `/plugin marketplace update` 刷新他们的本地副本。

<h2 id="walkthrough-create-a-local-marketplace">
  演练：创建本地 marketplace
</h2>

此示例创建一个包含一个 plugin 的 marketplace：一个用于代码审查的 `quality-review` skill。你将创建目录结构、添加 skill、创建 plugin manifest 和 marketplace 目录，然后安装并测试它。

<Steps>
  <Step title="创建目录结构">
    ```bash theme={null}
    mkdir -p my-marketplace/.claude-plugin
    mkdir -p my-marketplace/plugins/quality-review-plugin/.claude-plugin
    mkdir -p my-marketplace/plugins/quality-review-plugin/skills/quality-review
    ```
  </Step>

  <Step title="创建 skill">
    创建一个 `SKILL.md` 文件，定义 `quality-review` skill 的功能。

    ```markdown my-marketplace/plugins/quality-review-plugin/skills/quality-review/SKILL.md theme={null}
    ---
    description: Review code for bugs, security, and performance
    ---

    Review the code I've selected or the recent changes for:
    - Potential bugs or edge cases
    - Security concerns
    - Performance issues
    - Readability improvements

    Be concise and actionable.
    ```
  </Step>

  <Step title="创建 plugin manifest">
    创建一个 `plugin.json` 文件，描述该 plugin。manifest 位于 `.claude-plugin/` 目录中。

    ```json my-marketplace/plugins/quality-review-plugin/.claude-plugin/plugin.json theme={null}
    {
      "name": "quality-review-plugin",
      "description": "Adds a quality-review skill for quick code reviews",
      "version": "1.0.0",
      "author": {
        "name": "Your Name"
      }
    }
    ```

    <Note>
      设置 `version` 意味着用户仅在你更改此字段时才会收到更新，因此在每次发布时都要提升版本号。具有 command source 的 plugin 不会被此字段固定。如果你省略 `version`，版本来自 [版本管理](/docs/zh-CN/plugins-reference#version-management) 中的下一个来源。
    </Note>
  </Step>

  <Step title="创建 marketplace 文件">
    创建列出你的 plugin 的 marketplace 目录。

    ```json my-marketplace/.claude-plugin/marketplace.json theme={null}
    {
      "name": "my-plugins",
      "owner": {
        "name": "Your Name"
      },
      "plugins": [
        {
          "name": "quality-review-plugin",
          "source": "./plugins/quality-review-plugin",
          "description": "Adds a quality-review skill for quick code reviews"
        }
      ]
    }
    ```
  </Step>

  <Step title="添加和安装">
    从包含 `my-marketplace` 的目录启动 Claude Code 并运行以下命令。install 命令打开一个 plugin 详情视图，你可以在其中选择安装范围来确认安装。检查安装摘要：如果它报告 `Run /reload-plugins to activate.`，请参阅 [不重启应用而应用 plugin 更改](/docs/zh-CN/discover-plugins#apply-plugin-changes-without-restarting)。

    ```shell theme={null}
    /plugin marketplace add ./my-marketplace
    /plugin install quality-review-plugin@my-plugins
    ```
  </Step>

  <Step title="尝试一下">
    在编辑器中选择一些代码并运行你的新 skill。Plugin skills 使用 plugin 名称进行命名空间划分。

    ```shell theme={null}
    /quality-review-plugin:quality-review
    ```
  </Step>
</Steps>

要了解更多关于 plugins 可以做什么的信息，包括 hooks、agents、MCP servers 和 LSP servers，请参阅 [Plugins](/docs/zh-CN/plugins)。

<Note>
  **plugins 如何安装**：当用户安装 plugin 时，Claude Code 将 plugin 目录复制到缓存位置，除了 link mode 中的 command source，它被就地使用。复制的 plugins 无法使用 `../shared-utils` 之类的路径引用其目录外的文件，因为这些文件不会被复制。

  如果你需要在 plugins 之间共享文件，请使用符号链接。有关详细信息，请参阅 [Plugin 缓存和文件解析](/docs/zh-CN/plugins-reference#plugin-caching-and-file-resolution)。
</Note>

<h2 id="create-the-marketplace-file">
  创建 marketplace 文件
</h2>

在你的存储库根目录中创建 `.claude-plugin/marketplace.json`。此文件定义你的 marketplace 的名称、所有者信息以及包含其源的 plugins 列表。

每个 plugin 条目至少需要一个 `name` 和 `source`（告诉 Claude Code 从哪里获取它）。有关所有可用字段，请参阅下面的[完整架构](#marketplace-schema)。

```json theme={null}
{
  "name": "company-tools",
  "owner": {
    "name": "DevTools Team",
    "email": "devtools@example.com"
  },
  "plugins": [
    {
      "name": "code-formatter",
      "source": "./plugins/formatter",
      "description": "Automatic code formatting on save",
      "version": "2.1.0",
      "author": {
        "name": "DevTools Team"
      }
    },
    {
      "name": "deployment-tools",
      "source": {
        "source": "github",
        "repo": "company/deploy-plugin"
      },
      "description": "Deployment automation tools"
    }
  ]
}
```

<h2 id="marketplace-schema">
  Marketplace 架构
</h2>

<h3 id="required-fields">
  必需字段
</h3>

| 字段        | 类型     | 描述                                                                                                                                                                                                                                                                                                  | 示例             |
| :-------- | :----- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------- |
| `name`    | string | Marketplace 标识符，采用 kebab-case 格式，不包含空格、控制字符或双向格式化字符。这是面向公众的：用户在安装 plugins 时会看到它（例如，`/plugin install my-tool@your-marketplace`）。每个用户只能为每个名称注册一个 marketplace：添加第二个同名 marketplace 时，Claude Code 会替换第一个。要在一个 marketplace 名称下发布多个 plugins，请在[单个 `marketplace.json`](#create-the-marketplace-file) 中列出它们。 | `"acme-tools"` |
| `owner`   | object | Marketplace 维护者信息（[见下面的字段](#owner-fields)）                                                                                                                                                                                                                                                          |                |
| `plugins` | array  | 可用 plugins 列表                                                                                                                                                                                                                                                                                       | 见下文            |

<Note>
  **保留名称**：以下 marketplace 名称为 Anthropic 官方使用保留，第三方 marketplaces 无法使用：`claude-code-marketplace`、`claude-code-plugins`、`claude-plugins-official`、`claude-plugins-community`、`claude-community`、`anthropic-marketplace`、`anthropic-plugins`、`agent-skills`、`anthropic-agent-skills`、`knowledge-work-plugins`、`life-sciences`、`claude-for-legal`、`claude-for-financial-services`、`financial-services-plugins`、`first-party-plugins`、`claude-tag-plugins`、`healthcare`。冒充官方 marketplaces 的名称（如 `official-claude-plugins` 或 `anthropic-plugins-v2`）也被阻止。保留这些名称可防止第三方 marketplace 将自己呈现为 Anthropic 发布的来源。

  Claude Code 每次加载 marketplace 时都会重新检查保留名称，而不仅仅是在添加时。在该名称成为保留名称之前以其中一个名称注册的 marketplace 停止加载，并报告它是[从不受信任的来源注册的](/docs/zh-CN/errors#marketplace-is-registered-from-an-untrusted-source)。移除该 marketplace 并从官方 Anthropic 来源重新添加它。受新保留名称影响的第三方 marketplace 在你以不同名称重新添加它后立即再次加载。在 v2.1.205 之前，`first-party-plugins` 和 `healthcare` 不是保留的，已在保留名称下注册的 marketplace 继续加载。在 v2.1.265 之前，`claude-tag-plugins` 不是保留的。
</Note>

<h3 id="owner-fields">
  所有者字段
</h3>

| 字段      | 类型     | 必需 | 描述                    |
| :------ | :----- | :- | :-------------------- |
| `name`  | string | 是  | 维护者或团队的名称             |
| `email` | string | 否  | 维护者的联系电子邮件            |
| `url`   | string | 否  | 网站、GitHub 个人资料或组织 URL |

<h3 id="optional-fields">
  可选字段
</h3>

| 字段                                    | 类型     | 描述                                                                                                                                                                                      |
| :------------------------------------ | :----- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `$schema`                             | string | 用于编辑器自动完成和验证的 JSON Schema URL。Claude Code 在加载时忽略此字段。                                                                                                                                    |
| `description`                         | string | 简短的 marketplace 描述                                                                                                                                                                      |
| `version`                             | string | Marketplace 清单版本                                                                                                                                                                        |
| `metadata.pluginRoot`                 | string | Claude Code 解析裸 plugin 源名称的目录。见[相对路径](#relative-paths)。需要 Claude Code v2.1.239 或更高版本。                                                                                                   |
| `allowCrossMarketplaceDependenciesOn` | array  | 此 marketplace 中的 plugins 可能依赖的其他 marketplaces。来自此处未列出的 marketplace 的依赖项在安装时被阻止。见[依赖来自另一个 marketplace 的 plugin](/docs/zh-CN/plugin-dependencies#depend-on-a-plugin-from-another-marketplace)。 |
| `renames`                             | object | 从前一个 plugin `name` 到其当前名称的映射，或如果 plugin 被移除则映射到 `null`。当你重命名或移除 `plugins` 中的条目时，让现有用户自动迁移。见[重命名或移除 plugin](#rename-or-remove-a-plugin)。需要 Claude Code v2.1.193 或更高版本。                   |

`description` 和 `version` 也可以在 `metadata` 下接受，以实现向后兼容性。

<h2 id="plugin-entries">
  Plugin 条目
</h2>

`plugins` 数组中的每个 plugin 条目描述一个 plugin 及其位置。你可以包含 [plugin manifest 架构](/docs/zh-CN/plugins-reference#plugin-manifest-schema)中的任何字段，如 `description`、`version`、`author`、`commands` 和 `hooks`，加上这些 marketplace 特定的字段：`source`、`category`、`tags`、`strict`、`relevance`、`headers` 和 `headersHelper`。

<h3 id="required-fields-2">
  必需字段
</h3>

| 字段       | 类型             | 描述                                                                                                      |
| :------- | :------------- | :------------------------------------------------------------------------------------------------------ |
| `name`   | string         | Plugin 标识符（kebab-case，无空格、控制字符或双向格式化字符）。这是面向公众的：用户在安装时会看到它（例如，`/plugin install my-plugin@marketplace`）。 |
| `source` | string\|object | 从哪里获取 plugin（见下面的 [Plugin 源](#plugin-sources)）                                                          |

<h3 id="optional-plugin-fields">
  可选 plugin 字段
</h3>

**标准元数据字段：**

| 字段               | 类型      | 描述                                                                                                                                                                                                    |
| :--------------- | :------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `displayName`    | string  | 在 UI 界面中显示的人类可读名称。当条目和 plugin 的 `plugin.json` 都未设置时，用户会看到 plugin 的 `name`。可以包含空格和任何大小写。不用于命名空间或查找。                                                                                                    |
| `description`    | string  | 简短的 plugin 描述                                                                                                                                                                                         |
| `version`        | string  | Plugin 版本。如果设置（在此处或在 `plugin.json` 中），plugin 将固定到此字符串，用户仅在其更改时才会收到更新。具有 [`command` 源](#command-sources)的 plugin 不会被任一字段固定。如果在两个地方都未设置，版本来自 [版本管理](/docs/zh-CN/plugins-reference#version-management)中的下一个源。 |
| `author`         | object  | Plugin 作者信息（`name` 必需；`email` 和 `url` 可选）                                                                                                                                                             |
| `homepage`       | string  | Plugin 主页或文档 URL                                                                                                                                                                                      |
| `repository`     | string  | 源代码存储库 URL                                                                                                                                                                                            |
| `license`        | string  | SPDX 许可证标识符（例如，MIT、Apache-2.0）                                                                                                                                                                        |
| `keywords`       | array   | 用于 plugin 发现和分类的标签                                                                                                                                                                                    |
| `metadata`       | object  | 自由格式对象，用于你自己的字段，如权利或目录数据。Claude Code 不读取它。在 v2.1.222 之前，`claude plugin validate` 将该键报告为无法识别的字段。                                                                                                       |
| `category`       | string  | Plugin 类别以供组织                                                                                                                                                                                         |
| `tags`           | array   | 用于可搜索性的标签                                                                                                                                                                                             |
| `strict`         | boolean | 控制 `plugin.json` 是否是组件定义的权威（默认：true）。见下面的 [Strict 模式](#strict-mode)。                                                                                                                                  |
| `relevance`      | object  | 告诉 Claude Code 何时向用户建议此 plugin 的信号。仅对管理员在托管设置中允许列表的 marketplace 生效。见 [为你的组织推荐 plugin](/docs/zh-CN/plugin-relevance)。                                                                                       |
| `defaultEnabled` | boolean | Plugin 安装后是否启用（默认：true）。设置为 `false` 以安装禁用的 plugin，直到用户选择启用。优先于 plugin 的 `plugin.json` 中的同一字段。见 [默认启用](/docs/zh-CN/plugins-reference#default-enablement)。                                                   |

条目和 plugin 自己的 `plugin.json` 都可以设置显示字段 `displayName`、`description`、`author`、`homepage`、`repository`、`license` 和 `keywords`。在 plugin 列表和详情中，安装前后：

* 对于你在条目上设置的字段，用户会看到条目的值，即使 `plugin.json` 设置了不同的值。
* 对于条目未设置的字段，用户会看到 `plugin.json` 的值。

安装前，Claude Code 只能为具有 [相对路径源](#relative-paths)的条目读取 `plugin.json`，其 plugin 文件位于 marketplace 内部。对于具有任何其他源类型的条目，用户在安装 plugin 之前只会看到条目自己的字段。

**组件配置字段：**

| 字段           | 类型             | 描述                                    |
| :----------- | :------------- | :------------------------------------ |
| `skills`     | string\|array  | 包含 `<name>/SKILL.md` 的 skill 目录的自定义路径 |
| `commands`   | string\|array  | 平面 `.md` skill 文件或目录的自定义路径            |
| `agents`     | string\|array  | agent 文件的自定义路径                        |
| `hooks`      | string\|object | 自定义 hooks 配置或 hooks 文件的路径             |
| `mcpServers` | string\|object | MCP server 配置或 MCP 配置的路径              |
| `lspServers` | string\|object | LSP server 配置或 LSP 配置的路径              |

**存档身份验证字段：**

当条目在需要凭证的服务器上具有 [`archive` 源](#zip-archives)时设置这些字段。

| 字段              | 类型     | 描述                                                                                                                                                              |
| :-------------- | :----- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `headers`       | object | Claude Code 在下载此条目的存档时发送的 HTTP 标头。覆盖 marketplace 中相同名称的标头。需要 Claude Code v2.1.238 或更高版本。                                                                        |
| `headersHelper` | string | 命令，将此条目的存档下载的 HTTP 标头打印为一个 JSON 对象，用于过期的凭证。见 [验证存档下载](#authenticate-archive-downloads)。条目还必须设置 [`"strict": false`](#strict-mode)。需要 Claude Code v2.1.238 或更高版本。 |

<h2 id="plugin-sources">
  Plugin 源
</h2>

Plugin 源告诉 Claude Code 在你的 marketplace 中列出的每个单独 plugin 从哪里获取。这些在 `marketplace.json` 中每个 plugin 条目的 `source` 字段中设置。

Claude Code 将每个已安装的 plugin 复制到本地版本化 plugin 缓存中，位置为 `~/.claude/plugins/cache`，除了[链接模式](#copy-mode-and-link-mode)中的 [`command` 源](#command-sources)，Claude Code 会就地使用。Claude Code 还会[将 plugin 的符合条件的 Node.js 包依赖项安装](/docs/zh-CN/plugins-reference#node-js-package-dependencies)到缓存副本中。

| 源            | 类型                           | 字段                               | 注释                                                                                                                                                                       |
| ------------ | ---------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 相对路径         | `string`（例如 `"./my-plugin"`） | 无                                | marketplace repo 中的本地目录。必须以 `./` 开头，除非你在 [`metadata.pluginRoot`](#relative-paths) 下写一个[裸名](#relative-paths)。Claude Code 相对于 marketplace 根目录解析路径，而不是 `.claude-plugin/` 目录 |
| `github`     | object                       | `repo`、`ref?`、`sha?`             |                                                                                                                                                                          |
| `url`        | object                       | `url`、`ref?`、`sha?`              | Git URL 源                                                                                                                                                                |
| `git-subdir` | object                       | `url`、`path`、`ref?`、`sha?`       | git repo 中的子目录。稀疏克隆以最小化大型 monorepos 的带宽                                                                                                                                  |
| `npm`        | object                       | `package`、`version?`、`registry?` | 通过 `npm install` 安装                                                                                                                                                      |
| `archive`    | object                       | `url`、`sha256?`                  | 通过 HTTPS 下载的 Zip 存档。在用户机器上无需 git 或 npm 即可工作。需要 Claude Code v2.1.224 或更高版本                                                                                                |
| `command`    | object                       | `command`、`timeout?`、`mode?`     | 通过运行本地命令生成的 plugin 目录，每个会话重新运行一次以获取更改。需要 Claude Code v2.1.229 或更高版本                                                                                                      |

<Note>
  **Marketplace 源与 plugin 源**：这些是控制不同事物的不同概念。

  * **Marketplace 源**：从哪里获取 `marketplace.json` 目录本身。在用户运行 `/plugin marketplace add` 或在 `extraKnownMarketplaces` 设置中设置。基于 Git 的 marketplace 源支持 `ref`（分支/标签）但不支持 `sha`。
  * **Plugin 源**：从哪里获取 marketplace 中列出的单个 plugin。在 `marketplace.json` 内每个 plugin 条目的 `source` 字段中设置。基于 Git 的 plugin 源支持 `ref`（分支/标签）和 `sha`（精确提交）。

  例如，托管在 `acme-corp/plugin-catalog` 的 marketplace（marketplace 源）可以列出从 `acme-corp/code-formatter` 获取的 plugin（plugin 源）。marketplace 源和 plugin 源指向不同的存储库，并独立固定。
</Note>

下面的基于 git 的源类型是 `github`、`url` 和 `git-subdir`。当在其中任何一个上同时设置 `ref` 和 `sha` 时，`sha` 是有效的固定。Claude Code 直接获取并检出固定的提交。

在大多数 git 主机上，包括 GitHub、GitLab 和 Bitbucket，这意味着即使上游的 `ref` 命名的分支或标签已被删除，只要提交仍然可从存储库到达，安装也会成功。某些服务器（如 AWS CodeCommit）不支持通过 SHA 获取提交。在这些服务器上，`ref` 必须仍然存在，固定的提交必须可从其到达。

如果你通过**组织设置 > Plugins** 分发 plugins，只允许某些源类型。见[通过组织设置分发](#distribute-through-organization-settings)。

<h3 id="relative-paths">
  相对路径
</h3>

对于同一存储库中的 plugins，使用以 `./` 开头的路径：

```json theme={null}
{
  "name": "my-plugin",
  "source": "./plugins/my-plugin"
}
```

路径相对于 marketplace 根目录解析，即包含 `.claude-plugin/` 的目录。在上面的示例中，`./plugins/my-plugin` 指向 `<repo>/plugins/my-plugin`，即使 `marketplace.json` 位于 `<repo>/.claude-plugin/marketplace.json`。不要使用 `../` 来引用 marketplace 根目录外的路径。在 macOS 和 Linux 上，Claude Code 拒绝在前导 `./` 之后任何地方包含反斜杠的条目路径，所以在每个平台上将分隔符写为 `/`。

裸名是没有 `/` 的单个目录名，例如 `"formatter"`。要写裸名而不是 `./` 路径，请设置 [`metadata.pluginRoot`](#optional-fields) 为它们解析的目录。使用 `"pluginRoot": "./plugins"`，Claude Code 将 `"source": "formatter"` 解析为 `./plugins/formatter`。需要 Claude Code v2.1.239 或更高版本。

`metadata.pluginRoot` 本身必须是 marketplace 内的相对路径。Claude Code 对已经以 `./` 开头的源忽略它。包含 `/` 的源，例如 `team-a/formatter`，不是裸名，即使设置了 `metadata.pluginRoot`，仍然需要 `./` 前缀。

<Note>
  Claude Code 相对于 marketplace 的本地副本解析相对路径，所以当用户从 git 源或本地目录添加你的 marketplace 时它们有效。如果用户通过直接 URL 添加你的 marketplace 到 `marketplace.json` 文件，相对路径将无法解析，因为 Claude Code 仅下载该文件。对于基于 URL 的分发，请改用任何其他[plugin 源](#plugin-sources)。见[故障排除](#plugins-with-relative-paths-fail-in-url-based-marketplaces)了解详情。
</Note>

<h3 id="github-repositories">
  GitHub 存储库
</h3>

```json theme={null}
{
  "name": "github-plugin",
  "source": {
    "source": "github",
    "repo": "owner/plugin-repo"
  }
}
```

你可以固定到特定的分支、标签或提交：

```json theme={null}
{
  "name": "github-plugin",
  "source": {
    "source": "github",
    "repo": "owner/plugin-repo",
    "ref": "v2.0.0",
    "sha": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0"
  }
}
```

| 字段     | 类型     | 描述                               |
| :----- | :----- | :------------------------------- |
| `repo` | string | 必需。`owner/repo` 格式的 GitHub 存储库   |
| `ref`  | string | 可选。Git 分支或标签（默认为存储库默认分支）         |
| `sha`  | string | 可选。完整的 40 字符 git 提交 SHA 以固定到精确版本 |

<h3 id="git-repositories">
  Git 存储库
</h3>

```json theme={null}
{
  "name": "git-plugin",
  "source": {
    "source": "url",
    "url": "https://gitlab.com/team/plugin.git"
  }
}
```

你可以固定到特定的分支、标签或提交：

```json theme={null}
{
  "name": "git-plugin",
  "source": {
    "source": "url",
    "url": "https://gitlab.com/team/plugin.git",
    "ref": "main",
    "sha": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0"
  }
}
```

| 字段    | 类型     | 描述                                                                                                   |
| :---- | :----- | :--------------------------------------------------------------------------------------------------- |
| `url` | string | 必需。完整的 git 存储库 URL（`https://` 或 `git@`）。`.git` 后缀是可选的，所以 Azure DevOps 和 AWS CodeCommit URL 不带后缀也可以工作 |
| `ref` | string | 可选。Git 分支或标签（默认为存储库默认分支）                                                                             |
| `sha` | string | 可选。完整的 40 字符 git 提交 SHA 以固定到精确版本                                                                     |

<h3 id="git-subdirectories">
  Git 子目录
</h3>

使用 `git-subdir` 指向位于 git 存储库子目录中的 plugin。Claude Code 使用稀疏的部分克隆来仅获取子目录，最小化大型 monorepos 的带宽。

```json theme={null}
{
  "name": "my-plugin",
  "source": {
    "source": "git-subdir",
    "url": "https://github.com/acme-corp/monorepo.git",
    "path": "tools/claude-plugin"
  }
}
```

你可以固定到特定的分支、标签或提交：

```json theme={null}
{
  "name": "my-plugin",
  "source": {
    "source": "git-subdir",
    "url": "https://github.com/acme-corp/monorepo.git",
    "path": "tools/claude-plugin",
    "ref": "v2.0.0",
    "sha": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0"
  }
}
```

`url` 字段也接受 GitHub 简写（`owner/repo`）或 SSH URL（`git@github.com:owner/repo.git`）。

| 字段     | 类型     | 描述                                                    |
| :----- | :----- | :---------------------------------------------------- |
| `url`  | string | 必需。Git 存储库 URL、GitHub `owner/repo` 简写或 SSH URL        |
| `path` | string | 必需。repo 中包含 plugin 的子目录路径（例如，`"tools/claude-plugin"`） |
| `ref`  | string | 可选。Git 分支或标签（默认为存储库默认分支）                              |
| `sha`  | string | 可选。完整的 40 字符 git 提交 SHA 以固定到精确版本                      |

<h3 id="npm-packages">
  npm 包
</h3>

作为 npm 包分发的 Plugins 使用 `npm install` 安装。这适用于公共 npm registry 上的任何包或你的团队托管的私有 registry。

```json theme={null}
{
  "name": "my-npm-plugin",
  "source": {
    "source": "npm",
    "package": "@acme/claude-plugin"
  }
}
```

要固定到特定版本，请添加 `version` 字段：

```json theme={null}
{
  "name": "my-npm-plugin",
  "source": {
    "source": "npm",
    "package": "@acme/claude-plugin",
    "version": "2.1.0"
  }
}
```

要从私有或内部 registry 安装，请添加 `registry` 字段：

```json theme={null}
{
  "name": "my-npm-plugin",
  "source": {
    "source": "npm",
    "package": "@acme/claude-plugin",
    "version": "^2.0.0",
    "registry": "https://npm.example.com"
  }
}
```

| 字段         | 类型     | 描述                                                        |
| :--------- | :----- | :-------------------------------------------------------- |
| `package`  | string | 必需。包名称或作用域包（例如，`@org/plugin`）                             |
| `version`  | string | 可选。版本或版本范围（例如，`2.1.0`、`^2.0.0`、`~1.5.0`）                  |
| `registry` | string | 可选。自定义 npm registry URL。默认为系统 npm registry（通常为 npmjs.org） |

<h3 id="zip-archives">
  Zip 存档
</h3>

使用 `archive` 将 plugin 分发为 Claude Code 通过 HTTPS 下载的 zip 文件，这样安装在用户机器上无需 git 或 npm 即可工作。在任何静态文件服务器或工件存储库上托管文件，例如 S3 bucket、Artifactory 通用存储库或 nginx。需要 Claude Code v2.1.224 或更高版本。在 v2.1.120 到 v2.1.223 版本上，安装 plugin 失败并显示 `This plugin uses a source type your Claude Code version does not support. Update Claude Code and try again.`；在更早的版本上，包含 `archive` 条目的 marketplace 完全无法加载。

此条目从工件服务器上的 zip 文件安装 plugin：

```json theme={null}
{
  "name": "my-plugin",
  "source": {
    "source": "archive",
    "url": "https://artifacts.example.com/claude-plugins/my-plugin-2.1.0.zip"
  }
}
```

构建 zip 时，你可以直接 zip plugin 的内容或 zip plugin 文件夹本身。Claude Code 在存档的顶部查找 `.claude-plugin/`，然后在单个顶级文件夹内查找，所以两种布局都可以安装：

```text theme={null}
my-plugin.zip          my-plugin.zip
├── .claude-plugin/    └── my-plugin/
│   └── plugin.json        ├── .claude-plugin/
└── commands/              │   └── plugin.json
                           └── commands/
```

Claude Code 不会查找超过一个文件夹的深度，所以嵌套更深的 plugin 无法安装。Claude Code 拒绝大于 256 MiB 的存档。

要固定精确文件，请添加 `sha256` 字段，其中包含存档的摘要：

```json theme={null}
{
  "name": "my-plugin",
  "source": {
    "source": "archive",
    "url": "https://artifacts.example.com/claude-plugins/my-plugin-2.1.0.zip",
    "sha256": "6bfa50e3d2e00c052b46abe51fff89346ac803e45771f76dcf6df1ab74cca5e1"
  }
}
```

如果下载的文件与固定不匹配，Claude Code 拒绝安装并报告 [`Plugin archive integrity check failed`](/docs/zh-CN/errors#plugin-archive-integrity-check-failed)。

存档源接受这些字段：

| 字段       | 类型     | 描述                                                                                                      |
| :------- | :----- | :------------------------------------------------------------------------------------------------------ |
| `url`    | string | 必需。zip 存档的 HTTPS URL。Claude Code 拒绝 `http://` URL，以及环回、链接本地和云元数据主机。每个重定向跳转必须满足相同的规则，否则 Claude Code 拒绝下载 |
| `sha256` | string | 可选。存档的 SHA-256 摘要，为 64 个十六进制字符，大写或小写。Claude Code 验证每次下载并在不匹配时拒绝安装                                       |

`sha256` 摘要也用作 plugin 的版本，当 `plugin.json` 和 marketplace 条目都未声明版本时。见[版本管理](/docs/zh-CN/plugins-reference#version-management)。如果你声明 `version`，该版本字符串是更新信号，所以在更改 zip 及其摘要后，也要提升版本，否则用户保留缓存副本。

<h4 id="authenticate-archive-downloads">
  验证存档下载
</h4>

要验证存档下载，例如从私有 registry 下载，请设置 Claude Code 随之发送的 HTTP 标头。在你注册 marketplace 的 `url` 源上设置 `headers`，例如 [`extraKnownMarketplaces`](/docs/zh-CN/settings-reference#extraknownmarketplaces) 条目。在 Claude Code v2.1.238 或更高版本上，你可以在 plugin 的条目上设置它，在 `source` 旁边。

如果你要放在 `headers` 中的值是短期的，例如你的 registry 按需生成的令牌，请在同一位置设置 `headersHelper` 命令。Claude Code 运行命令并将其打印的 JSON 对象作为该位置的标头发送。需要 Claude Code v2.1.238 或更高版本。

你选择的位置决定了哪些下载获得标头以及 Claude Code 何时运行命令：

| 位置                  | 获得标头的下载                                   | Claude Code 何时运行设置在那里的 `headersHelper`                                                |
| :------------------ | :---------------------------------------- | :------------------------------------------------------------------------------------ |
| Marketplace `url` 源 | 在 marketplace URL 的源上的存档下载，意味着相同的方案、主机和端口 | 在每次获取 marketplace 的 `marketplace.json` 之前和在该源上的每次存档下载之前。Claude Code 将一次运行的输出重用最多 60 秒 |
| Plugin 条目           | 仅该条目的下载                                   | 仅当用户自己安装或更新该单个 plugin 时，并[接受命令](#how-users-accept-a-headershelper-command)            |

当两个位置都设置相同名称的标头时，Claude Code 发送条目的值。在一个位置内，命令打印的标头覆盖相同名称的列出的标头。

<h5 id="add-a-headershelper-to-a-plugin-entry">
  向 plugin 条目添加 headersHelper
</h5>

此条目在 `source` 旁边设置 `headersHelper`。它还设置 `"strict": false`，这是 Claude Code 对设置 `headersHelper` 的 `marketplace.json` 条目所需的。使用 [`"strict": false`](#strict-mode)，marketplace 条目是 plugin 的完整定义，所以用户可以在接受命令之前查看 plugin 包含的内容：

```json theme={null}
{
  "name": "my-plugin",
  "description": "Formatting commands for internal services",
  "strict": false,
  "commands": "./commands",
  "source": {
    "source": "archive",
    "url": "https://registry.example.com/plugins/my-plugin-2.1.0.zip"
  },
  "headersHelper": "/opt/bin/mint-registry-token.sh"
}
```

要检查条目，运行 `claude plugin install my-plugin@your-marketplace`。Claude Code 显示你命令和存档 URL，并在你接受后下载 zip。

在 v2.1.238 之前，Claude Code 下载条目的存档时不带其 `headers` 或 `headersHelper`，所以依赖它们的安装失败并显示 `HTTP 401 while downloading plugin archive from`，后跟 URL，registry 的状态代码代替 401。

<h4 id="write-the-headershelper-command">
  编写 headersHelper 命令
</h4>

无论你在 marketplace 的 `url` 源还是在 plugin 条目上设置 `headersHelper`，编写命令以满足这些要求：

* **命令文本**：最多 500 个可打印 ASCII 字符，没有四个或更多空格的运行。
* **输出**：在 stdout 上打印一个标头名称和字符串值的 JSON 对象，然后在 10 秒内以 0 退出。
* **Shell 和工作目录**：Claude Code 通过 `sh` 运行命令，或在 Windows 上通过 `cmd.exe`，从配置目录 `~/.claude` 或 [`CLAUDE_CONFIG_DIR`](/docs/zh-CN/env-vars#variables)。给出绝对路径或 `PATH` 上的命令，因为相对路径相对于该目录解析，而不是用户的项目。
* **Claude Code 移除的变量**：从 `marketplace.json` 条目或项目的 `.claude/settings.json` 或 `.claude/settings.local.json` 中设置的命令的环境中，Claude Code 移除每个名称包含 `TOKEN`、`SECRET`、`KEY` 或 `AUTH` 等词的变量，包括 `ANTHROPIC_API_KEY`。Claude Code 不对用户设置、`--settings` 文件或托管设置中设置的命令应用此移除。
* **Claude Code 设置的变量**：`CLAUDE_CODE_MARKETPLACE_URL` 和 `CLAUDE_CODE_MARKETPLACE_NAME` 用于 `url` 源的命令，以及 `CLAUDE_CODE_PLUGIN_NAME` 和 `CLAUDE_CODE_PLUGIN_ARCHIVE_URL` 用于条目的命令。`CLAUDE_CODE_MARKETPLACE_NAME` 在用户通过 URL 添加 marketplace 后的第一次获取时未设置，因为该获取是提供名称的。

生成持有者令牌的命令打印如下对象：

```json theme={null}
{"Authorization": "Bearer eyJhbGciOiJSUzI1NiJ9"}
```

<h4 id="when-claude-code-skips-a-headershelper-command-or-drops-its-output">
  Claude Code 何时跳过 headersHelper 命令或丢弃其输出
</h4>

Claude Code 不运行 `headersHelper` 命令，或在这些情况下丢弃来自 `headers` 或命令输出的标头：

* **命令失败**：如果命令以非零退出、运行超过 10 秒或打印除 JSON 字符串值对象之外的任何内容，Claude Code 不进行它运行命令的获取或下载。
* **Marketplace URL 不以 `https://` 开头**：Claude Code 不运行该 `url` 源的命令，仅发送其 `headers` 字段中列出的标头。
* **重定向离开源**：当下载被重定向离开存档 URL 的源时，Claude Code 丢弃 marketplace `url` 源和 plugin 条目的 `headers` 值和命令输出。
* **条目设置路由或身份标头**：Claude Code 从条目的 `headers` 和命令输出中丢弃请求路由和客户端身份名称，例如 `Host`、`Cookie` 和 `X-Forwarded-*`，并保留身份验证名称，例如 `Authorization`。Claude Code 以这种方式过滤每个 `marketplace.json` 条目，以及[内联设置条目](/docs/zh-CN/settings-reference#extraknownmarketplaces)取决于哪个文件声明它。
* **命令在 `--add-dir` 目录的设置中设置**：Claude Code 忽略它，在 `url` 源和[内联 plugin 条目](/docs/zh-CN/settings-reference#extraknownmarketplaces)上都一样，仅发送该文件的 `headers`。
* **托管设置阻止命令**：将 [`disableCommandPluginSources`](/docs/zh-CN/settings-reference#disablecommandpluginsources) 设置为 `true` 阻止 `headersHelper` 命令，[`allowManagedHooksOnly`](/docs/zh-CN/settings-reference#allowmanagedhooksonly) 也阻止它们，除非 `disableCommandPluginSources` 明确为 `false`。在任一阻止下，Claude Code 仍然为托管设置本身声明的 marketplace 运行命令。

<h4 id="how-users-accept-a-headershelper-command">
  用户如何接受 headersHelper 命令
</h4>

用户每次从 plugin 的自己的视图在 `/plugin` 或使用 `claude plugin install` 或 `claude plugin update` 自己安装或更新该单个 plugin 时接受 plugin 条目的命令。Claude Code 显示命令和存档 URL，并仅在用户接受后运行命令。在非交互式 shell 中，传递 [`--yes`](/docs/zh-CN/plugins-reference#plugin-install) 以接受它。

Claude Code 仅运行它显示的命令，用于它显示的存档 URL。如果条目的命令或存档 URL 在此期间更改，Claude Code 拒绝安装或更新。仅查询字符串中的更改不计算。

<h5 id="installs-and-updates-that-refuse-the-command-instead-of-asking">
  拒绝命令而不是询问的安装和更新
</h5>

在任何其他操作上，而不是单个 plugin 安装或更新，Claude Code 既不运行条目的命令也不下载其存档，所以 plugin 保持其已安装版本或保持未安装。用户看到的取决于操作：

* **一次安装多个 plugins、从 plugin 建议或作为另一个 plugin 的依赖项**：Claude Code 拒绝具有命令的 plugin 并将用户指向该 plugin 在 `/plugin` 中的自己的视图。批量安装中的其他 plugins 仍然安装。依赖被拒绝 plugin 的 plugin 无法安装，直到用户自己安装被拒绝的 plugin。
* **后台自动更新，或会话启动用于从未下载其存档的 plugin**：Claude Code 在 `/plugin` 错误选项卡中列出 plugin，以便用户知道手动安装或更新它。找到条目的自动更新仍然宣传已安装版本列表无。

<h5 id="when-a-marketplace-url-source’s-command-runs">
  Marketplace `url` 源的命令何时运行
</h5>

Marketplace `url` 源的 `headersHelper` 在设置文件中声明，例如 [`extraKnownMarketplaces`](/docs/zh-CN/settings-reference#extraknownmarketplaces) 条目，而不是在 marketplace 发布的目录中，所以 Claude Code 不会在每次安装或更新时要求用户接受它。声明它的设置文件决定了 Claude Code 何时运行它：

| 设置文件                                                        | Claude Code 何时运行命令                                                                                              |
| :---------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------- |
| 用户设置、`--settings` 文件或机器上的托管设置文件                             | 无需询问，包括在后台 marketplace 刷新期间                                                                                     |
| 项目的 `.claude/settings.json` 或 `.claude/settings.local.json` | 仅在用户接受该文件夹本身的[工作区信任对话框](/docs/zh-CN/permissions#what-runs-before-you-trust-a-folder)后。`-p` 或 SDK 会话不计为接受它，父文件夹的信任也不计 |
| 服务器托管设置                                                     | 仅在用户在[安全批准对话框](/docs/zh-CN/server-managed-settings#security-approval-dialogs)中批准交付的设置后                               |

在 `-p` 或 SDK 会话中，Claude Code 无法显示安全批准对话框。它应用其他交付的设置，但 marketplace 获取和任何需要命令的存档下载失败，直到用户在交互式会话中批准。

对于这些文件之一中的[内联 plugin 条目](/docs/zh-CN/settings-reference#extraknownmarketplaces)，Claude Code 要求与该文件中 marketplace 级别命令相同的文件夹信任或设置批准，用户也在每次安装或更新时接受条目的命令。

<h3 id="command-sources">
  Command 源
</h3>

当本地安装的工具生成 plugin 目录时使用 `command`，例如为当前选定的工具链呈现其 plugin 的 IDE。Claude Code 在用户安装 plugin 时运行命令，并在后台每个会话重新运行一次，所以你的用户无需重新安装即可获取工具的更改输出。需要 Claude Code v2.1.229 或更高版本。在 v2.1.120 到 v2.1.228 上，安装 plugin 失败并显示 `This plugin uses a source type your Claude Code version does not support. Update Claude Code and try again.`，在更早的版本上整个 marketplace 无法加载。

此条目从工具打印的任何目录安装 plugin：

```json theme={null}
{
  "name": "my-plugin",
  "source": {
    "source": "command",
    "command": "my-tool claude-plugin-path"
  }
}
```

Claude Code 通过平台 shell 运行命令，macOS 和 Linux 上的 `sh` 或 Windows 上的 `cmd.exe`，从用户的主目录。命令必须在 stdout 上打印恰好一行并以代码 0 退出。该行是包含完整 plugin 的目录的绝对路径，在命令退出时，路径可能在运行之间更改。

Claude Code 停止运行超过 `timeout` 秒的命令，安装或更新失败。Claude Code 也在这些情况下拒绝打印的路径，安装或更新以相同方式失败：

* 目录在其顶级没有 plugin 内容，例如 `.claude-plugin/` 目录或 `skills/`、`commands/`、`agents/` 或 `hooks/` 目录
* 目录是 Claude Code 启动的目录，或其父目录之一
* 在 Windows 上，路径是 UNC 路径

Command 源接受这些字段：

| 字段        | 类型     | 描述                                                                                                          |
| :-------- | :----- | :---------------------------------------------------------------------------------------------------------- |
| `command` | string | 必需。Shell 命令，在 stdout 上打印 plugin 目录的绝对路径作为单行并以 0 退出。必须是可打印 ASCII，最多 500 字符，没有四个或更多空格的运行，所以用户可以查看他们被要求接受的整个命令 |
| `timeout` | number | 可选。等待命令的整数秒数，然后放弃（默认：60，最大：600）                                                                             |
| `mode`    | string | 可选。`"copy"`（默认）将打印的目录复制到 plugin 缓存中。`"link"` 就地使用打印的目录。见[复制模式和链接模式](#copy-mode-and-link-mode)               |

<h4 id="copy-mode-and-link-mode">
  复制模式和链接模式
</h4>

使用默认的 `"mode": "copy"`，Claude Code 将打印的目录复制到版本化 plugin 缓存中，并从目录内容的哈希派生[plugin 版本](/docs/zh-CN/plugins-reference#version-management)。你的工具可以在命令退出后删除或重写目录，产生相同内容的重新运行计为最新。Claude Code 拒绝安装大于 256 MiB 或包含超过 20,000 个条目的目录。

为大型 plugin 目录设置 `"mode": "link"`，不应复制，例如呈现的 SDK 导出。Claude Code 用打印目录的每个顶级条目的链接填充 plugin 的缓存条目，并就地使用文件，所以没有复制、文件内容未哈希，大小限制不适用。如果顶级条目是指向打印目录外的符号链接，安装失败。Claude Code 也跳过链接模式 plugin 的[Node.js 包依赖项安装](/docs/zh-CN/plugins-reference#node-js-package-dependencies)，所以打印已包含 plugin 需要的任何 `node_modules` 的目录。

保持打印的目录就位，只要 plugin 保持安装，因为 Claude Code 在每次启动时通过这些链接加载 plugin。Claude Code 从打印目录的真实路径及其顶级条目派生[plugin 版本](/docs/zh-CN/plugins-reference#version-management)，而不是文件内部，所以打印不同的路径以表示新内容。在打印目录中或其下方启动的会话中，Claude Code 根本不加载 plugin。

Claude Code 不支持 Windows 上的链接模式，拒绝在那里安装链接模式 plugin。改为声明 `"mode": "copy"`。

<h4 id="how-users-accept-the-command">
  用户如何接受命令
</h4>

Claude Code 在用户的机器上运行你的命令，所以它将每次运行绑定到用户的明确接受：

* 当用户从 `/plugin` 中的 plugin 详情屏幕安装 plugin，或在交互式终端中使用 `claude plugin install` 或 `claude plugin update` 安装或更新它时，Claude Code 首先向他们显示确切的命令字符串，并为该安装记录接受的命令。可以在接受相同命令的记录接受上进行的 `claude plugin update` 显示无。在非交互式 shell 中，例如配置脚本，传递 `--yes` 到 `claude plugin install` 或 `claude plugin update` 以接受它打印的命令。
* 每条其他路径仅运行用户已接受的命令。这包括从 `/plugin` 启动的更新和[何时 Claude Code 重新运行命令](#when-claude-code-re-runs-the-command)中描述的后台运行。当未接受任何内容时，Claude Code 拒绝运行命令并告诉用户如何查看它。Claude Code 从不将 command 源 plugin 安装为另一个 plugin 的依赖项，所以用户自己先安装它。
* 如果你更改条目的 `command` 或切换其 `mode`，用户保留他们已有的版本，Claude Code 停止重新运行命令。在交互式会话中，`/plugin` 错误选项卡显示新命令，直到用户通过运行 `claude plugin update <plugin>@<marketplace>` 查看并接受它。

管理员可以使用托管设置 [`disableCommandPluginSources`](/docs/zh-CN/settings-reference#disablecommandpluginsources) 在整个组织中阻止 command 源。如果组织设置 [`allowManagedHooksOnly`](/docs/zh-CN/settings-reference#allowmanagedhooksonly)，Claude Code 默认阻止 command 源。

<h4 id="when-claude-code-re-runs-the-command">
  Claude Code 何时重新运行命令
</h4>

打印的目录反映工具在命令运行时的状态，所以 Claude Code 在这些时间重新运行命令：

* 每次用户安装或更新 plugin 时
* 每个会话一次用于每个启用的 command 源 plugin，在后台，会话启动后不久。此运行不通过 marketplace 自动更新，所以它不依赖 marketplace 的[自动更新设置](/docs/zh-CN/discover-plugins#configure-auto-updates)
* 在启动或 `/reload-plugins` 时，当启用的 plugin 的已安装版本从 plugin 缓存中丢失时

当用户设置 [`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`](/docs/zh-CN/env-vars) 时，Claude Code 跳过两个后台运行。显式安装和更新仍然使用该变量集运行命令。

当命令的哈希输出已更改时，Claude Code 将结果安装为新版本并在运行的交互式会话中重新加载它，切换[`/reload-plugins` 切换的相同组件](/docs/zh-CN/plugins-reference#environment-variables)。用户看到 plugin 已重新加载的通知。如果就地重新加载会使会话的提示缓存失效，Claude Code 改为提示用户运行 `/reload-plugins`，它[警告缓存成本并在使用 `--force` 重新运行时应用](/docs/zh-CN/prompt-caching#enabling-or-disabling-a-plugin)。

<h3 id="advanced-plugin-entries">
  高级 plugin 条目
</h3>

此示例显示了使用许多可选字段的 plugin 条目，包括命令、agents、hooks 和 MCP servers 的自定义路径：

```json theme={null}
{
  "name": "enterprise-tools",
  "source": {
    "source": "github",
    "repo": "company/enterprise-plugin"
  },
  "description": "Enterprise workflow automation tools",
  "version": "2.1.0",
  "author": {
    "name": "Enterprise Team",
    "email": "enterprise@example.com"
  },
  "homepage": "https://docs.example.com/plugins/enterprise-tools",
  "repository": "https://github.com/company/enterprise-plugin",
  "license": "MIT",
  "keywords": ["enterprise", "workflow", "automation"],
  "category": "productivity",
  "commands": [
    "./commands/core/",
    "./commands/enterprise/",
    "./commands/experimental/preview.md"
  ],
  "agents": ["./agents/security-reviewer.md", "./agents/compliance-checker.md"],
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh"
          }
        ]
      }
    ]
  },
  "mcpServers": {
    "enterprise-db": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"]
    }
  },
  "strict": false
}
```

需要注意的关键事项：

* **`commands` 和 `agents`**：你可以指定多个目录或单个文件。路径相对于 plugin 根目录，必须保持在其内部。
  * Claude Code 拒绝解析到 plugin 目录外的路径，例如 `./../shared.md`，带有 [`path escapes plugin directory`](/docs/zh-CN/errors#path-escapes-plugin-directory) 错误，仍然加载 plugin 而不带该组件
* **`${CLAUDE_PLUGIN_ROOT}`**：在 hook 命令和 MCP server 配置中使用此变量来引用 plugin 安装目录中的文件。
  * 查看[替换表](/docs/zh-CN/plugins-reference#environment-variables)了解每个服务器类型在哪些配置字段中替换它
  * 对于应该在 plugin 更新后保留的依赖项或状态，请改用 [`${CLAUDE_PLUGIN_DATA}`](/docs/zh-CN/plugins-reference#persistent-data-directory)
* **`strict: false`**：由于这设置为 false，plugin 不需要自己的 `plugin.json`。marketplace 条目定义了一切。见下面的 [Strict 模式](#strict-mode)。

默认情况下，plugin 的 skills 从其 `source` 下的 `skills/` 目录加载。`skills` 字段中列出的路径添加到该扫描中：

```json theme={null}
"skills": ["./skills/", "./extra-skills/"]
```

当多个 plugin 条目在 marketplace 根目录（`source: "./"`）共享一个 `skills/` 文件夹时，改为列出特定子目录，以便每个条目仅加载自己的 skills：

```json theme={null}
"source": "./",
"skills": ["./skills/code-review", "./skills/docs"]
```

使用 marketplace 根源，列出的路径是该条目的完整集合，共享 `skills/` 文件夹中的其他目录不会加载。列出 `./skills/` 本身或 plugin 根目录会保持完整扫描。如果列出的路径都不存在，则改为运行默认扫描。

<h3 id="strict-mode">
  Strict 模式
</h3>

`strict` 字段控制 `plugin.json` 是否是组件定义（skills、agents、hooks、MCP servers、输出样式）的权威。

| 值          | 行为                                                                      |
| :--------- | :---------------------------------------------------------------------- |
| `true`（默认） | `plugin.json` 是权威。marketplace 条目可以用额外的组件补充它，两个源都被合并。                    |
| `false`    | marketplace 条目是完整的定义。如果 plugin 也有声明组件的 `plugin.json`，那就是冲突，plugin 无法加载。 |

**何时使用每种模式：**

* **`strict: true`**：plugin 有自己的 `plugin.json` 并管理自己的组件。marketplace 条目可以在顶部添加额外的 skills 或 hooks。这是默认值，适用于大多数 plugins。
* **`strict: false`**：marketplace 操作员想要完全控制。plugin repo 提供原始文件，marketplace 条目定义这些文件中的哪些被公开为 skills、agents、hooks 等。当 marketplace 以不同于 plugin 作者意图的方式重组或策划 plugin 的组件时很有用。

<h2 id="host-and-distribute-marketplaces">
  托管和分发 marketplaces
</h2>

<h3 id="host-on-github-recommended">
  在 GitHub 上托管（推荐）
</h3>

GitHub 是托管和分发 marketplace 的推荐方式：

1. **创建存储库**：为你的 marketplace 设置一个新存储库
2. **添加 marketplace 文件**：使用你的 plugin 定义创建 `.claude-plugin/marketplace.json`
3. **与团队共享**：用户使用 `/plugin marketplace add owner/repo` 添加你的 marketplace

**优点**：内置版本控制、问题跟踪和团队协作功能。

<h3 id="host-on-other-git-services">
  在其他 git 服务上托管
</h3>

任何 git 托管服务都可以工作，例如 GitLab、Bitbucket 和自托管服务器。用户使用完整的存储库 URL 添加：

```shell theme={null}
/plugin marketplace add https://gitlab.com/company/plugins.git
```

<h3 id="private-repositories">
  私有存储库
</h3>

Claude Code 支持从私有存储库安装 plugins。如果你通过[**组织设置 > Plugins**](https://claude.ai/admin-settings/plugins)分发你的 marketplace，你的 git 凭证不涉及：组织同步通过 Claude GitHub App 或你的组织的 GitHub Enterprise App 读取 marketplace 存储库，plugin 源如果无法进行身份验证必须是公开的。有关完整规则，请参阅[通过组织设置分发](#distribute-through-organization-settings)。

<h4 id="commands-you-run">
  你运行的命令
</h4>

当你运行 `/plugin marketplace add`、`/plugin install`、`/plugin update` 或 `/plugin marketplace update` 时，Claude Code 使用你现有的 git 凭证助手，所以通过 `gh auth login`、macOS Keychain 或 `git-credential-store` 的 HTTPS 访问工作方式与你的终端中相同。SSH 访问工作，只要主机已经在你的 `known_hosts` 文件中，并且密钥已加载到 `ssh-agent` 中，因为 Claude Code 会抑制主机指纹和密钥密码的交互式 SSH 提示。GitHub `owner/repo` 简写源默认通过 SSH 克隆；设置 [`CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1`](/docs/zh-CN/env-vars#variables) 以改为通过 HTTPS 克隆它们。

<h4 id="background-auto-updates">
  后台自动更新
</h4>

默认情况下，后台刷新会为其 `git pull` 禁用 git 凭证助手，所以即使配置了助手，pull 也无法对 HTTPS 上的私有存储库进行身份验证。SSH 远程不受影响：加载到 `ssh-agent` 中的密钥以与你运行的命令相同的方式对后台 pulls 进行身份验证。当后台 pull 失败时，Claude Code 会回退到从头重新克隆 marketplace。重新克隆确实使用你存储的 git 凭证，但它可能在大型存储库上[超时](#git-operations-time-out)，所以私有 marketplace 自动更新可能会间歇性失败。

两个设置使私有 marketplaces 的行为可预测：

* 设置 `CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1` 以在后台 pull 失败时保留现有克隆，而不是删除并重新克隆。你的 plugins 继续从最后同步的状态工作，使用 `/plugin marketplace update` 的手动更新仍然使用你的凭证进行 pull。
* 配置 git 凭证助手，例如使用 `gh auth setup-git` 用于 GitHub，以便重新克隆回退可以在不提示的情况下进行身份验证。

在你的环境中设置提供商令牌（如 `GITHUB_TOKEN`）本身不会启用后台身份验证。令牌仅通过配置的凭证助手（例如 `gh` CLI 的助手，它读取 `GH_TOKEN` 和 `GITHUB_TOKEN`）生效。

要使后台 pull 本身通过 HTTPS 进行身份验证，请配置全局 git URL 重写。重写在远程 URL 中嵌入令牌，所以即使后台 pull 禁用凭证助手，它也会生效，成功的 pull 会跳过重新克隆回退。以下示例重写 marketplace 存储库的 URL 以包含访问令牌：

```bash theme={null}
git config --global url."https://x-access-token:YOUR_TOKEN@github.com/acme-corp/plugins".insteadOf "https://github.com/acme-corp/plugins"
```

将重写范围限制在 marketplace 存储库或组织路径。仅以主机为基础的重写适用于机器上对该主机的每个 fetch 和 push，并覆盖你的正常凭证，包括对你自己的存储库的 pushes。

每个提供商在重写的 URL 中期望不同的用户名，相同的路径范围适用于每个提供商。对于自托管服务器，请将主机名替换为你的服务器的主机名：

| 提供商       | 重写的 URL 形式                                                        |
| :-------- | :---------------------------------------------------------------- |
| GitHub    | `https://x-access-token:YOUR_TOKEN@github.com/acme-corp/plugins`  |
| GitLab    | `https://oauth2:YOUR_TOKEN@gitlab.com/acme-corp/plugins`          |
| Bitbucket | `https://x-token-auth:YOUR_TOKEN@bitbucket.org/acme-corp/plugins` |

重写以纯文本形式在你的 gitconfig 中存储令牌，所以使用对 marketplace 存储库具有只读访问权限的令牌。

<Note>
  在 CI/CD 环境中，在从私有存储库安装 plugins 之前配置 git 凭证助手。在 GitHub Actions 上，导出对 marketplace 存储库具有读取访问权限的令牌作为 `GH_TOKEN`，然后运行 `gh auth setup-git`。默认工作流令牌只能访问工作流自己的存储库，所以另一个存储库中的私有 marketplace 需要个人访问令牌或应用令牌。在管道中配置的全局 URL 重写也直接对后台 pull 进行身份验证。
</Note>

<h3 id="distribute-through-organization-settings">
  通过组织设置分发
</h3>

如果你在 Team 或 Enterprise 计划上通过[**组织设置 > Plugins**](https://claude.ai/admin-settings/plugins)分发 plugins，这些源规则适用：

* marketplace 存储库必须是私有或内部的。组织同步通过 Claude GitHub App 或你的组织的 GitHub Enterprise App 读取它。
* 每个 plugin 源必须是 `github`、`url` 或 `git-subdir` 类型，或[相对路径](#relative-paths)，以 `./` 开头。如果你在 `metadata.pluginRoot` 下按裸名称列出 plugin，组织同步会将其拒绝为不支持的源，所以写出路径，例如 `./plugins/deploy-tools`。
* plugin 源可以在两种情况下是私有的：
  * 与 marketplace 存储库的所有者共享的 github.com 源
  * 在你的组织的 GitHub Enterprise 主机上安装了 GHE App 的源
* 组织同步在没有凭证的情况下获取所有其他源，所以不同所有者下的 github.com 存储库和其他主机上的存储库（例如 GitLab 或 Bitbucket）必须是公开的。

有关管理员工作流，请参阅[为你的组织管理 plugins](https://support.claude.com/en/articles/13837433)。

要包含私有 plugins，请将 plugin 文件夹放在 marketplace 存储库内，并使用[相对路径](#relative-paths)引用它们。组织同步在分发期间打包每个 plugin，所以用户永远不需要访问单独的源存储库。

例如，这个 `marketplace.json` plugin 条目引用你在 marketplace 存储库中的 `plugins/deploy-tools` 处提交的 plugin：

```json theme={null}
{
  "name": "deploy-tools",
  "source": "./plugins/deploy-tools"
}
```

<h4 id="keep-executables-out-of-the-top-level-bin-directory">
  将可执行文件保留在顶级 bin 目录之外
</h4>

不要在你通过组织设置分发的任何 plugin 中包含顶级 `bin/` 目录。claude.ai 拒绝具有该目录的 plugin，无论 plugin 是通过 marketplace 同步还是直接上传到达：

* **Marketplace 同步**：组织同步拒绝该 plugin 并同步 marketplace 的其余部分。错误消息以 `Plugin contains a top-level bin/ directory` 开头。
* **直接上传**：如果你改为在[**组织设置 > Plugins**](https://claude.ai/admin-settings/plugins)中上传 plugin，claude.ai 会以相同的消息拒绝上传。

将可执行文件保留在另一个目录中，例如 `scripts/`，并从你的[skills、hooks 或 MCP server 配置](/docs/zh-CN/plugins-reference#environment-variables)中将它们引用为 `${CLAUDE_PLUGIN_ROOT}/scripts/<name>`。

<h3 id="require-marketplaces-for-your-team">
  为你的团队要求 marketplaces
</h3>

你可以配置你的存储库，以便当团队成员[信任项目文件夹](/docs/zh-CN/permissions#what-runs-before-you-trust-a-folder)时，Claude Code 会为他们添加你的 marketplace，无需单独的提示。将你的 marketplace 添加到 `.claude/settings.json`：

```json theme={null}
{
  "extraKnownMarketplaces": {
    "company-tools": {
      "source": {
        "source": "github",
        "repo": "your-org/claude-plugins"
      }
    }
  }
}
```

你也可以指定默认应启用哪些 plugins：

```json theme={null}
{
  "enabledPlugins": {
    "code-formatter@company-tools": true,
    "deployment-tools@company-tools": true
  }
}
```

有关完整的配置选项，请参阅 [Plugin 设置](/docs/zh-CN/settings-reference#plugin-settings)。

<Note>
  如果你使用带有相对路径的本地 `directory` 或 `file` 源，路径将相对于你的存储库的主检出解析。当你从 git worktree 运行 Claude Code 时，路径仍然指向主检出，所以所有 worktrees 共享相同的 marketplace 位置。Marketplace 状态存储一次每个用户在 `~/.claude/plugins/known_marketplaces.json` 中，而不是每个项目。
</Note>

<h3 id="pre-populate-plugins-for-containers">
  为容器预填充 plugins
</h3>

对于容器镜像和 CI 环境，你可以在构建时预填充 plugins 目录，以便 Claude Code 启动时已经有 marketplaces 和 plugins 可用，无需在运行时克隆任何内容。设置 `CLAUDE_CODE_PLUGIN_SEED_DIR` 环境变量以指向此目录。

要分层多个种子目录，请在 Unix 上用 `:` 分隔路径，或在 Windows 上用 `;` 分隔。Claude Code 按顺序搜索每个目录，第一个包含给定 marketplace 或 plugin 缓存的种子获胜。

种子目录镜像 `~/.claude/plugins` 的结构：

```
$CLAUDE_CODE_PLUGIN_SEED_DIR/
  known_marketplaces.json
  marketplaces/<name>/...
  cache/<marketplace>/<plugin>/<version>/...
```

要构建种子目录，请在镜像构建期间运行 Claude Code 一次，安装你需要的 plugins，然后将生成的 `~/.claude/plugins` 目录复制到你的镜像中，并将 `CLAUDE_CODE_PLUGIN_SEED_DIR` 指向它。

要跳过复制步骤，请在构建期间将 `CLAUDE_CODE_PLUGIN_CACHE_DIR` 设置为你的目标种子路径，以便 plugins 直接安装到那里：

```bash theme={null}
CLAUDE_CODE_PLUGIN_CACHE_DIR=/opt/claude-seed claude plugin marketplace add your-org/plugins
CLAUDE_CODE_PLUGIN_CACHE_DIR=/opt/claude-seed claude plugin install my-tool@your-plugins
```

然后在你的容器的运行时环境中设置 `CLAUDE_CODE_PLUGIN_SEED_DIR=/opt/claude-seed`，以便 Claude Code 在启动时从种子读取。

在启动时，Claude Code 将种子的 `known_marketplaces.json` 中找到的 marketplaces 注册到主配置中，并使用在 `cache/` 下找到的 plugin 缓存，而无需重新克隆。这在交互模式和使用 `-p` 标志的非交互模式中都有效。

行为详情：

* **只读**：种子目录永远不会被写入。由于 git pull 会在只读文件系统上失败，种子 marketplaces 的自动更新被禁用。
* **种子条目优先**：在每次启动时，种子中声明的 marketplaces 会覆盖用户配置中的任何匹配条目。要选择退出种子 plugin，请使用 `/plugin disable` 而不是删除 marketplace。
* **路径解析**：Claude Code 通过在运行时探测 `$CLAUDE_CODE_PLUGIN_SEED_DIR/marketplaces/<name>/` 来定位 marketplace 内容，而不是信任存储在种子 JSON 内的路径。这意味着即使在与构建时不同的路径上挂载，种子也能正确工作。
* **变更被阻止**：针对种子管理的 marketplace 运行 `/plugin marketplace remove` 或 `/plugin marketplace update` 会失败，并提示你要求管理员更新种子镜像。
* **与设置组合**：如果 `extraKnownMarketplaces` 或 `enabledPlugins` 声明的 marketplace 已经存在于种子中，Claude Code 使用种子副本而不是克隆。

<h3 id="managed-marketplace-restrictions">
  托管 marketplace 限制
</h3>

对于需要严格控制 plugin 源的组织，管理员可以使用托管设置中的 [`strictKnownMarketplaces`](/docs/zh-CN/settings-reference#strictknownmarketplaces) 设置限制用户允许添加哪些 plugin marketplaces。要同时拒绝为单次运行 sideload plugins、agents 和 MCP servers 的 CLI 标志，请将其与 [`disableSideloadFlags`](/docs/zh-CN/settings-reference#disablesideloadflags) 配对。要允许列表哪些 marketplaces 的 plugins 可以作为上下文安装建议出现，请设置 [`pluginSuggestionMarketplaces`](/docs/zh-CN/settings-reference#pluginsuggestionmarketplaces)。

`strictKnownMarketplaces` 匹配 plugin 来自的 marketplace，而不是其中的条目，所以用户仍然可以从允许的 marketplace 安装具有[`command` 源](#command-sources)的 plugin。要同时阻止 command 源，请设置 [`disableCommandPluginSources`](/docs/zh-CN/settings-reference#disablecommandpluginsources)。

当在托管设置中配置 `strictKnownMarketplaces` 时，限制行为取决于值：

| 值        | 行为                                                 |
| -------- | -------------------------------------------------- |
| 未定义（默认）  | 无限制。用户可以添加任何 marketplace                           |
| 空数组 `[]` | 完全锁定。阻止每个 marketplace 源，包括官方 Anthropic marketplace |
| 源列表      | 允许列表强制执行。用户只能添加与条目匹配的 marketplaces                 |

<h4 id="common-configurations">
  常见配置
</h4>

禁用所有 marketplace 添加，包括官方 Anthropic marketplace：

```json theme={null}
{
  "strictKnownMarketplaces": []
}
```

仅允许官方 Anthropic marketplace。单个存储库条目的匹配是精确的，所以此条目不涵盖同一存储库的 `ref` 或 `path` 变体：

```json theme={null}
{
  "strictKnownMarketplaces": [
    {
      "source": "github",
      "repo": "anthropics/claude-plugins-official"
    }
  ]
}
```

使用此条目，Claude Code 保持已注册的官方 marketplace 可用，在新机器上，在你首次以交互方式启动 Claude Code 时自动注册 marketplace。

自动注册不涵盖每台机器。它最常遗漏：

* 在机器首次交互启动之前运行的非交互环境。
* Claude Code 已在阻止 marketplace 的策略下以交互方式运行的机器，例如空数组锁定。Claude Code 记录被阻止的尝试，在策略更改后不重试。

在这些机器上，将 marketplace 添加到同一 `managed-settings.json` 中的 [`extraKnownMarketplaces`](/docs/zh-CN/settings-reference#extraknownmarketplaces)，以便 Claude Code 自动注册它，或运行 `claude plugin marketplace add anthropics/claude-plugins-official`。

仅允许特定 marketplaces：

```json theme={null}
{
  "strictKnownMarketplaces": [
    {
      "source": "github",
      "repo": "acme-corp/approved-plugins"
    },
    {
      "source": "github",
      "repo": "acme-corp/security-tools",
      "ref": "v2.0"
    },
    {
      "source": "url",
      "url": "https://plugins.example.com/marketplace.json"
    }
  ]
}
```

使用[所有者通配符](/docs/zh-CN/settings-reference#owner-wildcards)条目允许 GitHub 组织下的每个 marketplace 存储库。所有者通配符需要 Claude Code v2.1.223 或更高版本。

```json theme={null}
{
  "strictKnownMarketplaces": [
    {
      "source": "github",
      "repo": "acme-corp/*"
    }
  ]
}
```

使用主机上的正则表达式模式匹配允许来自内部 git 服务器的所有 marketplaces。这是 [GitHub Enterprise Server](/docs/zh-CN/github-enterprise-server#plugin-marketplaces-on-ghes) 或自托管 GitLab 实例的推荐方法：

```json theme={null}
{
  "strictKnownMarketplaces": [
    {
      "source": "hostPattern",
      "hostPattern": "^github\\.example\\.com$"
    }
  ]
}
```

使用路径上的正则表达式模式匹配允许来自特定目录的基于文件系统的 marketplaces：

```json theme={null}
{
  "strictKnownMarketplaces": [
    {
      "source": "pathPattern",
      "pathPattern": "^/opt/approved/"
    }
  ]
}
```

使用 `".*"` 作为 `pathPattern` 来允许任何文件系统路径，同时仍然使用 `hostPattern` 控制网络源。

<Note>
  `strictKnownMarketplaces` 限制用户可以添加的内容，但不会自行注册 marketplaces。要为用户自动注册允许的 marketplace，请在同一 `managed-settings.json` 中将其添加到 [`extraKnownMarketplaces`](/docs/zh-CN/settings-reference#extraknownmarketplaces)。

  官方 Anthropic marketplace 是唯一 Claude Code 自行注册的，仅当允许列表允许时。自动注册也遗漏一些机器，例如非交互环境和早期策略阻止它的机器。要覆盖这些机器，也将官方 marketplace 添加到 `extraKnownMarketplaces`。有关两个设置并排，请参阅 [`strictKnownMarketplaces` 参考](/docs/zh-CN/settings-reference#strictknownmarketplaces)。
</Note>

<h4 id="how-restrictions-work">
  限制如何工作
</h4>

限制在任何网络或文件系统操作之前进行检查。检查在 marketplace 添加以及 plugin 安装、更新、刷新和自动更新时运行。如果 marketplace 在配置策略之前被添加，其源不再与允许列表匹配，Claude Code 会拒绝从中安装或更新 plugins。相同的强制执行也适用于 `blockedMarketplaces`。

要阻止 GitHub 所有者下的每个 marketplace 存储库，请在 `blockedMarketplaces` 条目中使用所有者通配符形式：`{ "source": "github", "repo": "untrusted-org/*" }`。需要 Claude Code v2.1.223 或更高版本。有关匹配规则（在阻止列表和允许列表之间不同），请参阅[所有者通配符](/docs/zh-CN/settings-reference#owner-wildcards)。

当用户添加 Claude Code [克隆而不是获取](/docs/zh-CN/discover-plugins#add-from-other-git-hosts)的 `https://` 存储库 URL（例如裸 `github.com` 或 `gitlab.com` 存储库 URL）时，Claude Code 也会根据 `blockedMarketplaces` 中的 `url` 条目检查它。如果条目命名相同的 URL，Claude Code 会阻止添加。在该比较中，Claude Code 忽略 `.git` 后缀和用户在 `#` 后附加的任何 ref。需要 Claude Code v2.1.232 或更高版本。在 v2.1.232 之前，Claude Code 仅针对它作为托管 `marketplace.json` 文件获取的 URL 匹配 `url` 条目。

允许列表对大多数源类型使用精确匹配，除了所有者通配符 `github` 条目。要允许 marketplace，所有指定的字段必须匹配：

* 对于 GitHub 源：`repo` 是必需的，要么命名一个存储库，要么使用所有者通配符形式 `owner/*` 来覆盖该所有者下的每个存储库。有关通配符条目如何匹配（包括大小写规则），请参阅[所有者通配符](/docs/zh-CN/settings-reference#owner-wildcards)。对于单个存储库条目，`ref` 必须完全匹配或在 marketplace 源和允许列表条目中都不存在，相同的规则适用于 `path`
* 对于 URL 源：完整 URL 必须完全匹配
* 对于 `hostPattern` 源：marketplace 主机与正则表达式模式匹配
* 对于 `pathPattern` 源：marketplace 的文件系统路径与正则表达式模式匹配

允许列表的精确匹配将仅因尾部斜杠、`.git` 后缀或 `ssh://` 和 `https://` 方案不同的 URL 视为不同的值。如果你的组织的 marketplace 可以通过多个 URL 形式克隆，优先使用 `hostPattern` 条目而不是字面 URL，以便 `https://`、`ssh://` 和 `user@host:path` 形式都匹配。

因为 `strictKnownMarketplaces` 在[托管设置](/docs/zh-CN/managed-settings)中设置，个别用户和项目配置无法覆盖这些限制。

有关完整的配置详细信息，包括所有支持的源类型和与 `extraKnownMarketplaces` 的比较，请参阅 [strictKnownMarketplaces 参考](/docs/zh-CN/settings-reference#strictknownmarketplaces)。

<h3 id="version-resolution-and-release-channels">
  版本解析和发布渠道
</h3>

Plugin 版本确定缓存路径和更新检测：如果解析的版本与用户已有的版本匹配，`/plugin update` 和自动更新会跳过该 plugin。对于 git 源，如果你省略 `version`，Claude Code 使用源的解析提交 SHA，所以用户在该提交更改时获得更新；这是内部或积极开发的 plugins 的最简单设置。有关完整的解析顺序（包括 `archive` 源），请参阅[版本管理](/docs/zh-CN/plugins-reference#version-management)。

<Warning>
  设置 `version` 为除了 [`command`](#command-sources) 之外的每个源类型固定 plugin，其版本始终包括命令生成内容的哈希。如果你在 `plugin.json` 中声明 `"version": "1.0.0"` 并推送新提交而不改变该字符串，这些源的现有用户保留缓存副本，因为 Claude Code 看到相同的版本。在每个发布时提升该字段，或省略它以回退到解析的版本。

  避免在 `plugin.json` 和 marketplace 条目中都设置 `version`。Claude Code 总是无声地使用 `plugin.json` 值，所以陈旧的 manifest 版本可能会掩盖你在 `marketplace.json` 中设置的版本。
</Warning>

<h4 id="set-up-release-channels">
  设置发布渠道
</h4>

要为你的 plugins 支持"稳定"和"最新"发布渠道，你可以设置两个指向同一 repo 的不同 refs 或 SHAs 的 marketplaces。然后你可以通过托管设置以两种方式之一将每个用户组分配给其自己的 marketplace：

* 部署单独的[端点管理设置](/docs/zh-CN/managed-settings#delivery-mechanisms)（例如托管设置文件或 MDM 配置文件）到每个组的设备。[Claude Code 如何组合托管源](/docs/zh-CN/managed-settings#precedence-within-the-managed-tier)说明每个组的文件或配置文件是否适用于也有组织范围源的设备。
* 为每个组定义一个 [Claude apps gateway 策略](/docs/zh-CN/claude-apps-gateway-config#managed)。网关应用第一个匹配规则适合用户的策略，所以对策略进行排序，以便每个用户到达其组的策略。组策略的 `extraKnownMarketplaces` 替换全局策略的映射而不是与其合并，所以在组的策略中列出组需要的每个 marketplace，而不仅仅是其渠道 marketplace。

来自管理控制台的服务器管理设置[适用于你的组织中的每个用户](/docs/zh-CN/server-managed-settings#current-limitations)，所以它们无法进行每个组的分配。

<Warning>
  每个渠道必须解析为不同的版本。如果你使用显式版本，`plugin.json` 必须在每个固定的 ref 处声明不同的 `version`。如果你省略 `version`，不同的提交 SHA 已经区分了渠道。如果两个 refs 解析为相同的版本字符串，Claude Code 会将它们视为相同并跳过更新。
</Warning>

<h5 id="example">
  示例
</h5>

```json theme={null}
{
  "name": "stable-tools",
  "plugins": [
    {
      "name": "code-formatter",
      "source": {
        "source": "github",
        "repo": "acme-corp/code-formatter",
        "ref": "stable"
      }
    }
  ]
}
```

```json theme={null}
{
  "name": "latest-tools",
  "plugins": [
    {
      "name": "code-formatter",
      "source": {
        "source": "github",
        "repo": "acme-corp/code-formatter",
        "ref": "latest"
      }
    }
  ]
}
```

<h5 id="assign-channels-to-user-groups">
  将渠道分配给用户组
</h5>

通过上述[设置发布渠道](#set-up-release-channels)中描述的每个组端点管理设置或网关策略将每个 marketplace 分配给其用户组。例如，稳定组接收：

```json theme={null}
{
  "extraKnownMarketplaces": {
    "stable-tools": {
      "source": {
        "source": "github",
        "repo": "acme-corp/stable-tools"
      }
    }
  }
}
```

早期访问组改为接收 `latest-tools`：

```json theme={null}
{
  "extraKnownMarketplaces": {
    "latest-tools": {
      "source": {
        "source": "github",
        "repo": "acme-corp/latest-tools"
      }
    }
  }
}
```

<h4 id="pin-dependency-versions">
  固定依赖版本
</h4>

Plugin 可以将其依赖约束到 semver 范围，以便对依赖的更新不会破坏依赖的 plugin。有关 `{plugin-name}--v{version}` git 标签约定、范围语法以及如何组合对同一依赖的多个约束，请参阅[约束 plugin 依赖版本](/docs/zh-CN/plugin-dependencies)。

<h3 id="rename-or-remove-a-plugin">
  重命名或删除 plugin
</h3>

Plugin 的 `name` 是其稳定标识符。用户在 `enabledPlugins`、`pluginConfigs` 和 `/plugin install` 命令中引用它，所以改变它会破坏每个现有的安装。要改变 UI 中显示的标签而不破坏安装，请设置 [`displayName`](#optional-plugin-fields) 并保持 `name` 不变。

如果你必须改变 plugin 的 `name`，或者你从 `plugins` 数组中删除 plugin，请添加顶级 `renames` 条目，以便现有用户迁移而不是看到 `plugin-not-found` 错误。自动迁移需要 Claude Code v2.1.193 或更高版本。将每个前名称映射到其当前名称，或映射到 `null` 如果 plugin 不再存在。以下示例将 `formatter` 重命名为 `code-formatter` 并记录 `legacy-linter` 已被删除：

```json theme={null}
{
  "name": "acme-tools",
  "owner": { "name": "Acme" },
  "plugins": [
    { "name": "code-formatter", "source": "./plugins/code-formatter" }
  ],
  "renames": {
    "formatter": "code-formatter",
    "legacy-linter": null
  }
}
```

当用户启动 Claude Code 时旧名称仍在其设置中，Claude Code 遵循 `renames` 映射：

* 如果条目指向新名称，Claude Code 在其新名称下加载 plugin 并显示一行通知，例如 `在"acme-tools" marketplace 中重命名为"code-formatter"`。然后它在用户、项目和本地设置范围中为 `enabledPlugins` 和 `pluginConfigs` 都将旧键重写为新键，所以通知只出现一次。
* 对于 `null` 条目，Claude Code 删除旧键，通知报告 plugin 已从 marketplace 中删除。
* 如果重命名的 plugin 使用远程源，例如 `github` 或 `npm`，Claude Code 在重命名后报告 `plugin-cache-miss`，用户必须运行 `/plugin install` 一次以在新名称下获取它。

将 `renames` 视为仅追加历史：即使在你期望每个用户都已迁移后，也要保持旧条目就位。Claude Code 遵循链，所以如果你稍后将 `code-formatter` 重命名为 `formatter-pro`，请添加第二个条目而不是编辑第一个。仍然启用原始 `formatter` 的用户然后通过两个条目解析到 `formatter-pro`。

在编辑映射后运行 `claude plugin validate .`；它拒绝任何链形成循环或不终止于 `null` 或 `plugins` 中列出的名称的条目。

<Note>
  托管和策略设置对 Claude Code 是只读的，所以在那里启用的 plugins 无法自动重写。重命名的 plugin 仍然在每个会话中加载，但重命名通知会重复出现，直到管理员更新托管设置文件中的 `enabledPlugins` 以使用新名称。相同的情况适用于通过其他只读源（例如 `--add-dir`）启用的 plugins。
</Note>

早期版本的 Claude Code 忽略 `renames` 字段并为旧名称报告 `plugin-not-found`。

<h2 id="validation-and-testing">
  验证和测试
</h2>

在共享前测试你的 marketplace。验证检查文件结构；要测试 plugin 是否改变了 Claude 在实际提示上的行为，请在发布新版本前使用 [`claude plugin eval`](/docs/zh-CN/plugin-evals) 运行其 eval 套件。

从你的 marketplace 目录验证 JSON 语法：

```bash theme={null}
claude plugin validate .
```

或从 Claude Code 内：

```shell theme={null}
/plugin validate .
```

添加 marketplace 进行测试：

```shell theme={null}
/plugin marketplace add ./path/to/marketplace
```

安装测试 plugin 以验证一切正常：

```shell theme={null}
/plugin install test-plugin@marketplace-name
```

有关完整的 plugin 测试工作流，请参阅[本地测试你的 plugins](/docs/zh-CN/plugins#test-your-plugins-locally)。有关技术故障排除，请参阅[Plugins 参考](/docs/zh-CN/plugins-reference)。

<h2 id="manage-marketplaces-from-the-cli">
  从 CLI 管理 marketplaces
</h2>

Claude Code 提供非交互式 `claude plugin marketplace` 子命令用于脚本编写和自动化。这些等同于交互式会话中可用的 `/plugin marketplace` 命令。

<h3 id="plugin-marketplace-add">
  Plugin marketplace add
</h3>

从 GitHub 存储库、git URL、远程 URL 或本地路径添加 marketplace。

```bash theme={null}
claude plugin marketplace add <source> [options]
```

**参数：**

* `<source>`：GitHub `owner/repo` 简写、git URL、指向 `marketplace.json` 文件的远程 URL 或本地目录路径。要固定到分支或标签，请将 `@ref` 附加到 GitHub 简写或 `#ref` 附加到 git URL

URL 必须包含其方案。从 Claude Code v2.1.196 开始，没有方案的主机（如 `gitlab.example.com/team/plugins`）被拒绝为无效的 `owner/repo` 简写，错误会告诉你添加 `https://` 或为本地路径使用 `./`。早期版本会将其误读为 GitHub 存储库路径，并在克隆时失败，出现 GitHub 未找到错误。

**选项：**

| 选项                    | 描述                                                                                                                 | 默认值    |
| :-------------------- | :----------------------------------------------------------------------------------------------------------------- | :----- |
| `--scope <scope>`     | 声明 marketplace 的位置：`user`、`project` 或 `local`。见 [Plugin 安装范围](/docs/zh-CN/plugins-reference#plugin-installation-scopes) | `user` |
| `--sparse <paths...>` | 通过 git sparse-checkout 限制检出到特定目录。对 monorepos 有用                                                                    |        |

从 GitHub 使用 `owner/repo` 简写添加 marketplace：

```bash theme={null}
claude plugin marketplace add acme-corp/claude-plugins
```

使用 `@ref` 固定到特定分支或标签：

```bash theme={null}
claude plugin marketplace add acme-corp/claude-plugins@v2.0
```

从非 GitHub 主机上的 git URL 添加：

```bash theme={null}
claude plugin marketplace add https://gitlab.example.com/team/plugins.git
```

从直接提供 `marketplace.json` 文件的远程 URL 添加：

```bash theme={null}
claude plugin marketplace add https://example.com/marketplace.json
```

从本地目录添加以进行测试：

```bash theme={null}
claude plugin marketplace add ./my-marketplace
```

在项目范围声明 marketplace，以便通过 `.claude/settings.json` 与你的团队共享：

```bash theme={null}
claude plugin marketplace add acme-corp/claude-plugins --scope project
```

对于 monorepo，限制检出到包含 plugin 内容的目录：

```bash theme={null}
claude plugin marketplace add acme-corp/monorepo --sparse .claude-plugin plugins
```

<h3 id="plugin-marketplace-list">
  Plugin marketplace list
</h3>

列出所有配置的 marketplaces。

```bash theme={null}
claude plugin marketplace list [options]
```

**选项：**

| 选项       | 描述       |
| :------- | :------- |
| `--json` | 输出为 JSON |

使用 `--json`，每个条目包括 `name`、`source`、一个包含 marketplace 存储的本地缓存路径的 `installLocation` 字段，以及源特定字段：GitHub 源的 `repo`、git 和 URL 源的 `url`，以及本地源的 `path`。当 marketplace 使用固定分支或标签添加时，GitHub 和 git 源也包括 `ref` 字段。

<h3 id="plugin-marketplace-remove">
  Plugin marketplace remove
</h3>

删除配置的 marketplace。别名 `rm` 也被接受。

```bash theme={null}
claude plugin marketplace remove <name> [options]
```

**参数：**

* `<name>`：marketplace 名称要删除，如 `claude plugin marketplace list` 所示。这是来自 `marketplace.json` 的 `name`，而不是你传递给 `add` 的源

**选项：**

| 选项                | 描述                                                                                                                                                                                                | 默认值    |
| :---------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :----- |
| `--scope <scope>` | 限制删除到单个设置范围：`user`、`project` 或 `local`。见 [Plugin 安装范围](/docs/zh-CN/plugins-reference#plugin-installation-scopes)。省略时，声明从每个可编辑的范围中删除。给定时，仅删除该范围的声明；当 marketplace 仍在另一个范围中声明时，共享状态、缓存和已安装的 plugin 数据将被保留 | （所有范围） |

<Warning>
  从其最后剩余的范围中删除 marketplace 也会卸载你从它安装的任何 plugins。要刷新 marketplace 而不丢失已安装的 plugins，请改用 `claude plugin marketplace update`。
</Warning>

<h3 id="plugin-marketplace-update">
  Plugin marketplace update
</h3>

从其源刷新 marketplaces 以检索新 plugins 和版本更改。使用分支或标签 `ref` 添加的 marketplace 会更新到该 ref 的最新提交，而不是存储库的默认分支。

```bash theme={null}
claude plugin marketplace update [name]
```

**参数：**

* `[name]`：marketplace 名称要更新，如 `claude plugin marketplace list` 所示。如果省略，更新所有 marketplaces

`remove` 和 `update` 在针对种子管理的 marketplace 运行时都会失败，这是只读的。更新所有 marketplaces 时，种子管理的条目被跳过，其他 marketplaces 仍然更新。要更改种子提供的 plugins，请要求你的管理员更新种子镜像。见 [为容器预填充 plugins](#pre-populate-plugins-for-containers)。

<h2 id="troubleshooting">
  故障排除
</h2>

<h3 id="marketplace-not-loading">
  Marketplace 未加载
</h3>

**症状**：无法添加 marketplace 或从中看到 plugins

**解决方案**：

* 验证 marketplace URL 是否可访问
* 检查 `.claude-plugin/marketplace.json` 是否存在于指定路径
* 使用 `claude plugin validate .` 或 `/plugin validate .` 确保 JSON 语法有效。要检查 skill、agent 和 command frontmatter，请参阅[验证没有 manifest 的 plugin 或目录](#validate-a-plugin-or-a-directory-without-a-manifest)
* 对于私有存储库，确认你有访问权限

<h3 id="marketplace-validation-errors">
  Marketplace 验证错误
</h3>

从你的 marketplace 目录运行 `claude plugin validate .` 或 `/plugin validate .` 来检查问题。当指向 marketplace 目录时，验证器检查 `marketplace.json` 是否存在 schema 错误、重复的 plugin 名称和源路径遍历。对于 `source` 是本地路径的每个条目，它还验证该 plugin 自己的 `plugin.json`，并在条目的 `version` 与 `plugin.json` 中的版本不匹配时发出警告。在 plugin 的 `plugin.json` 中发现的问题以条目索引为前缀，形式为 `plugins[2] plugin.json →`。

从 Claude Code v2.1.196 开始，每个条目的检查还会：

* 包括 `source` 为 `.` 的 plugins
* 在 `marketplace.json` 位于 `.claude-plugin` 目录外时运行，针对文件自己的目录解析源
* 即使文件的另一部分有 schema 错误，也报告每个条目的问题

早期版本跳过 marketplace 根目录中的 plugins，仅从 `.claude-plugin/marketplace.json` 开始下降。

从 marketplace 目录，Claude Code 不会打开 plugins 的 skill、agent、command 或 hook 文件。要查找这些文件中的错误，请参阅[验证没有 manifest 的 plugin 或目录](#validate-a-plugin-or-a-directory-without-a-manifest)。下表列出了从 marketplace 目录中最常见的错误，以及每个错误的原因和修复方法：

| 错误                                                                                                       | 原因                                                                                         | 解决方案                                                                                                       |
| :------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------- |
| `No manifest found in directory. Expected .claude-plugin/marketplace.json or .claude-plugin/plugin.json` | 你命名的目录没有 `.claude-plugin/marketplace.json` 或 `plugin.json`，也没有 skill、agent 或 command 文件可检查 | 从 marketplace 根目录运行，或使用必需字段创建 `.claude-plugin/marketplace.json`                                            |
| `Invalid JSON syntax: Unexpected token...`                                                               | marketplace.json 中的 JSON 语法错误                                                              | 检查缺少的逗号、多余的逗号或未引用的字符串                                                                                      |
| `Duplicate plugin name "x" found in marketplace`                                                         | 两个 plugins 共享相同的名称                                                                         | 给每个 plugin 一个唯一的 `name` 值                                                                                  |
| `plugins[0].source: Path contains ".."`                                                                  | 源路径包含 `..`                                                                                 | 使用相对于 marketplace 根目录的路径，不包含 `..`。见[相对路径](#relative-paths)                                                 |
| `Marketplace name cannot contain control or bidirectional-formatting characters`                         | marketplace `name` 包含 Unicode 双向格式化字符或控制字符，如转义或换行符                                         | 从名称中删除该字符。在 v2.1.247 之前，这些字符产生 `Marketplace name impersonates an official Anthropic/Claude marketplace` 错误 |
| `Plugin name cannot contain control or bidirectional-formatting characters`                              | plugin `name` 包含 Unicode 双向格式化字符或控制字符，如转义或换行符                                              | 从名称中删除该字符。在 v2.1.247 之前，Claude Code 没有运行此检查                                                                |

**警告**（非阻止）：

* `Marketplace has no plugins defined`：将至少一个 plugin 添加到 `plugins` 数组
* `No marketplace description provided`：添加顶级 `description` 以帮助用户理解你的 marketplace
* `Plugin name "x" is not kebab-case`：重命名为仅包含小写字母、数字和连字符（例如，`my-plugin`）。Claude Code 接受其他形式，但 claude.ai marketplace 同步会拒绝它们。
* `Marketplace name "x" is reserved in Claude Desktop`：marketplace 名称为 `org`、`org-provisioned` 或 `unknown`，任何大小写。Claude Code 接受这些名称，但 Claude Desktop 的托管 marketplace 同步会拒绝整个 marketplace。重命名 marketplace。在 v2.1.221 之前，`claude plugin validate` 没有运行此检查。
* `Marketplace name "x" is not accepted by Claude Desktop` 或 `Plugin name "x" is not accepted by Claude Desktop`：Claude Desktop 接受最多 128 个字符的名称，由字母、数字、`.`、`_` 和 `-` 组成，以字母或数字开头。Claude Code 接受其他形式，但 Claude Desktop 的托管 marketplace 同步会拒绝名称检查失败的 marketplace，并静默删除名称检查失败的 plugin 条目。重命名 marketplace 或 plugin。在 v2.1.221 之前，`claude plugin validate` 没有运行这些检查。

<h4 id="validate-a-plugin-or-a-directory-without-a-manifest">
  验证没有 manifest 的 plugin 或目录
</h4>

要查找 skill、agent 和 command 文件，其 frontmatter 无法解析，请运行 `claude plugin validate` 并命名包含它们的目录。Claude Code 不会查看你命名的目录之外。除了一次针对具有 `plugin.json` 的 plugin 的运行外，每次运行都需要 Claude Code v2.1.233 或更高版本。

<h5 id="pick-the-directory-to-name">
  选择要命名的目录
</h5>

Claude Code 根据你命名的目录检查不同的文件。在第一列中找到你想检查的内容，并运行该行的命令：

| 要检查                                                     | 运行                                                                                | Claude Code 检查                                                                        |
| :------------------------------------------------------ | :-------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------ |
| 具有 `plugin.json` 的 plugin                               | `claude plugin validate ./plugins/my-plugin`                                      | `plugin.json`、`hooks/hooks.json` 和 plugin 根目录下的 `skills`、`agents` 和 `commands` 目录     |
| 一个 skill、agent 或 command 目录，例如没有 `plugin.json` 的 plugin | `claude plugin validate .claude/skills`、`~/.claude/agents` 或 `./my-plugin/agents` | 该目录中的每个 skill、agent 或 command 文件                                                      |
| 其 skill 是其根 `SKILL.md` 的文件夹                             | `claude plugin validate ./skills`，命名包含该文件夹的 `skills` 目录                           | 每个文件夹的根 `SKILL.md`。包含目录必须命名为 `skills`；位于另一个名称下的文件夹，如 `plugins/`，没有检查其根 `SKILL.md` 的运行 |
| 一个项目的三个目录一次                                             | `claude plugin validate .claude`，或当项目没有 `.claude-plugin/` manifest 时的项目根目录        | `.claude/skills`、`.claude/agents` 和 `.claude/commands`                                |
| 你的用户级目录                                                 | `claude plugin validate ~/.claude`                                                | `~/.claude/skills`、`~/.claude/agents` 和 `~/.claude/commands`                          |

<h5 id="check-a-plugin-whose-skill-is-its-root-skill-md">
  检查其 skill 是其根 `SKILL.md` 的 plugin
</h5>

当你针对 plugin 目录运行 `claude plugin validate` 时，Claude Code 不会检查 plugin 根目录下的 `SKILL.md`。当 plugin 位于名为 `skills` 的目录中时，运行该命令两次：

* 命名该 `skills` 目录以检查 plugin 的根 `SKILL.md`。
* 命名 plugin 目录以检查其余部分。

当 plugin 位于另一个名称下（如 `plugins/`）时，`skills` 目录运行不可用，没有运行检查其根 `SKILL.md`。

<h5 id="check-files-behind-symlinks">
  检查符号链接后面的文件
</h5>

当你运行 `claude plugin validate` 时，Claude Code 不会跟随你命名的目录内的符号链接。它所做的取决于链接的位置：

* **plugin 或 `.claude` 根目录下的链接 `skills`、`agents` 或 `commands` 目录**：Claude Code 警告其中没有任何内容被读取。
* **`skills`、`agents` 或 `commands` 目录内的链接条目**：Claude Code 跳过它并警告，每个目录，它跳过了多少条目，会话会加载。
* **你命名的 `skills`、`agents` 或 `commands` 目录本身是符号链接，或其父 `.claude` 目录是**：Claude Code 报告错误并检查其中的任何内容。改为命名真实目录。

在两个 skills 情况下，运行通过警告。要检查链接的文件，再次运行并命名直接包含它们的目录：

* **其 `skills` 目录[链接到同级 plugin 的 skills](/docs/zh-CN/plugins-reference#share-files-within-a-marketplace-with-symlinks) 的 plugin**：命名同级 plugin 的目录。
* **`~/.claude/skills` 或 `.claude/skills` 中的[符号链接 skill 条目](/docs/zh-CN/skills#where-skills-live)**：Claude Code 在会话中跟随该条目。要检查它，命名一个名为 `skills` 的目录，该目录包含真实文件夹。

<h5 id="read-the-validation-results">
  读取验证结果
</h5>

干净的运行以 `Validation passed` 结束。

`No manifest found in directory` 意味着 Claude Code 在那里找不到 `plugin.json` 或 `marketplace.json`，也找不到它在其下探测的目录中的 skill、agent 或 command 文件。改为命名包含你的文件的 `skills`、`agents` 或 `commands` 目录。

Claude Code 从这些运行中报告的两个错误，以及每个错误的修复：

* `YAML frontmatter failed to parse: ...`：修复 skill、agent 或 command 文件的 frontmatter 块中的 YAML。在你这样做之前，会话从文件中读取不到 frontmatter 字段
* `Invalid JSON syntax: ...` 在 `hooks/hooks.json` 上：修复 JSON 语法。在你这样做之前，会话加载 plugin 时不带该文件中的 hooks。Claude Code 仅在 plugin 运行中报告此错误

在 plugin 运行中，Claude Code 还会警告 plugin 根目录下的 `CLAUDE.md`。对于你通过 [component path fields](/docs/zh-CN/plugins-reference#component-path-fields) 在 `plugin.json` 中设置的路径，Claude Code 检查每个路径是否存在，但不读取那里的文件。

<h3 id="plugin-installation-failures">
  Plugin 安装失败
</h3>

**症状**：Marketplace 出现但 plugin 安装失败

**解决方案**：

* 验证 plugin 源 URL 是否可访问
* 检查 plugin 目录是否包含必需的文件
* 对于 GitHub 源，确保存储库是公开的或你有访问权限
* 通过手动克隆/下载来测试 plugin 源
* 如果源同时固定了 `ref` 和 `sha`，删除的上游分支或标签不会阻止大多数 git 主机（包括 GitHub、GitLab 和 Bitbucket）上的安装。在不支持通过 SHA 获取提交的服务器上（如 AWS CodeCommit），`ref` 必须仍然存在，固定的提交必须可从其到达。如果安装仍然失败，请确认固定的提交仍然存在于存储库中

<h3 id="private-repository-authentication-fails">
  私有存储库身份验证失败
</h3>

**症状**：从私有存储库安装 plugins 时出现身份验证错误

**解决方案**：

对于手动安装和更新：

* 验证你已使用你的 git 提供商进行身份验证（例如，对于 GitHub 运行 `gh auth status`）
* 检查你的凭证助手是否配置正确：`git config --global credential.helper`
* 运行 `git ls-remote <marketplace-url>` 来测试 git 是否可以自行进行身份验证。如果 git 要求输入用户名或密码，请先存储凭证：对于 GitHub over HTTPS，运行 `gh auth setup-git`，对于 SSH 远程，将你的密钥加载到 `ssh-agent`

对于后台自动更新：

* 默认情况下，后台刷新会为拉取禁用 git 凭证助手，因此拉取无法通过 HTTPS 进行身份验证。在 `ssh-agent` 中加载了密钥的 SSH 远程仍然可以进行身份验证。失败的拉取会触发从头重新克隆，这使用你存储的凭证，但在大型存储库上可能超时
* 设置 `CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1` 以在后台拉取失败时保留现有克隆
* 配置 git 凭证助手，例如 `gh auth setup-git`，以便重新克隆回退可以进行身份验证
* 如果重新克隆在大型存储库上超时，请使用 [`CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS`](#git-operations-time-out) 增加限制
* 配置一个 [git URL 重写](#private-repositories) 作用于 marketplace 存储库，以便后台拉取直接进行身份验证
* 或使用 `/plugin marketplace update <name>` 手动更新私有 marketplaces，这使用你的凭证

<h3 id="marketplace-updates-fail-in-offline-environments">
  Marketplace 更新在离线环境中失败
</h3>

**症状**：Marketplace `git pull` 在后台失败，Claude Code 反复尝试无法成功的重新克隆。

**原因**：默认情况下，当 `git pull` 失败时，Claude Code 会尝试从头重新克隆。在离线或隔离的环境中，重新克隆以相同的方式失败，之后对先前缓存的恢复是尽力而为的。刷新在启动后在后台运行，因此不会延迟启动，但每个会话都会重复失败的尝试，每个 git 操作都可以等待 [120 秒超时](#git-operations-time-out)。

**解决方案**：设置 `CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1` 以在拉取失败时跳过重新克隆尝试并继续使用现有缓存：

```bash theme={null}
export CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1
```

对于存储库永远无法访问的完全离线部署，请改用 [`CLAUDE_CODE_PLUGIN_SEED_DIR`](#pre-populate-plugins-for-containers) 在构建时预填充 plugins 目录。

<h3 id="git-operations-time-out">
  Git 操作超时
</h3>

**症状**：Plugin 安装或 marketplace 更新失败，出现超时错误，如"Git clone timed out after 120s"或"Git pull timed out after 120s"。

**原因**：Claude Code 对所有 git 操作使用 120 秒超时，包括克隆 plugin 存储库和拉取 marketplace 更新。大型存储库或缓慢的网络连接可能超过此限制。

**解决方案**：使用 `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` 环境变量增加超时。该值以毫秒为单位：

```bash theme={null}
export CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS=300000  # 5 minutes
```

<h3 id="plugins-with-relative-paths-fail-in-url-based-marketplaces">
  相对路径 Plugins 在基于 URL 的 Marketplaces 中失败
</h3>

**症状**：通过 URL（如 `https://example.com/marketplace.json`）添加了 marketplace，但具有相对路径源（如 `"./plugins/my-plugin"`）的 plugins 无法安装，出现 `its marketplace entry path does not stay inside the marketplace directory` 错误。已安装的 plugins 无法加载，出现 `Plugin source path refused` 错误。两条消息都有一个[错误参考条目](/docs/zh-CN/errors#marketplace-entry-path-does-not-stay-inside-the-marketplace-directory)。

**原因**：添加基于 URL 的 marketplace 仅下载 `marketplace.json` 文件本身，Claude Code 不会从该服务器按相对路径获取 plugin 文件。marketplace 条目中的相对路径引用远程服务器上未下载的文件。

**解决方案**：

* **使用外部源**：将 plugin 条目更改为除相对路径外的任何 [plugin 源](#plugin-sources)：
  ```json theme={null}
  { "name": "my-plugin", "source": { "source": "github", "repo": "owner/repo" } }
  ```
* **使用基于 Git 的 Marketplace**：在 Git 存储库中托管你的 marketplace 并使用 git URL 添加它。基于 Git 的 marketplaces 克隆整个存储库，使相对路径有效。

<h3 id="files-not-found-after-installation">
  安装后文件未找到
</h3>

**症状**：Plugin 安装但对文件的引用失败，特别是 plugin 目录外的文件

**原因**：Plugins 被复制到缓存目录而不是就地使用，除了[链接模式中的 `command` 源](#copy-mode-and-link-mode)。引用 plugin 目录外文件的路径（如 `../shared-utils`）不会工作，因为这些文件不会被复制。

**解决方案**：见 [Plugin 缓存和文件解析](/docs/zh-CN/plugins-reference#plugin-caching-and-file-resolution) 了解解决方法，包括符号链接和目录重组。

有关其他调试工具和常见问题，请参阅[调试和开发工具](/docs/zh-CN/plugins-reference#debugging-and-development-tools)。

<h2 id="see-also">
  另见
</h2>

* [发现和安装预构建的 plugins](/docs/zh-CN/discover-plugins) - 从现有 marketplaces 安装 plugins
* [Plugins](/docs/zh-CN/plugins) - 创建你自己的 plugins
* [Plugins 参考](/docs/zh-CN/plugins-reference) - 完整的技术规范和架构
* [Plugin 设置](/docs/zh-CN/settings-reference#plugin-settings) - Plugin 配置选项
* [strictKnownMarketplaces 参考](/docs/zh-CN/settings-reference#strictknownmarketplaces) - 托管 marketplace 限制
