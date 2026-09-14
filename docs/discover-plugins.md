> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 通过市场发现和安装预构建插件

> 从市场发现和安装插件，以使用新 skills、agents 和功能扩展 Claude Code。

插件通过 skills、agents、hooks 和 MCP servers 扩展 Claude Code。插件市场是帮助您发现和安装这些扩展的目录，无需自己构建。

想要创建和分发自己的市场？请参阅[创建和分发插件市场](/docs/zh-CN/plugin-marketplaces)。

<h2 id="how-marketplaces-work">
  市场如何工作
</h2>

市场是他人创建和共享的插件目录。使用市场是一个两步过程：

<Steps>
  <Step title="添加市场">
    这会向 Claude Code 注册目录，以便您可以浏览可用内容。尚未安装任何插件。
  </Step>

  <Step title="安装单个插件">
    浏览目录并安装您想要的插件。
  </Step>
</Steps>

<h2 id="official-anthropic-marketplace">
  官方 Anthropic 市场
</h2>

Claude Code 在您首次以交互方式启动它时会自动添加官方 Anthropic 市场（`claude-plugins-official`）。如果 Claude Code 无法添加它，例如因为您的网络阻止了下载或[市场政策](/docs/zh-CN/plugin-marketplaces#managed-marketplace-restrictions)阻止了之前的尝试，请使用 `/plugin marketplace add anthropics/claude-plugins-official` 自己添加。

要浏览可用内容，请运行 `/plugin` 并转到**发现**选项卡，或在 [claude.com/plugins](https://claude.com/plugins) 查看目录。

要从官方市场安装插件，请使用 `/plugin install <name>@claude-plugins-official`。例如，要安装 GitHub 集成：

```shell theme={null}
/plugin install github@claude-plugins-official
```

`/plugin` 在终端 CLI 中打开一个交互式面板。如果 Claude 回复说 `/plugin` 在此环境中不可用，请使用 Claude 桌面应用中的[插件浏览器](/docs/zh-CN/desktop#install-plugins)，或在 `.claude/settings.json` 中的 [`enabledPlugins`](/docs/zh-CN/settings-reference#enabledplugins) 下声明插件以用于云会话。

如果安装失败，请匹配 Claude Code 报告的消息：

* `Marketplace "claude-plugins-official" not found`：使用 `/plugin marketplace add anthropics/claude-plugins-official` 添加市场，然后重试安装。
* 插件[在市场中未找到](#install-plugins)：检查插件名称。

<Note>
  官方市场由 Anthropic 维护，包含由 Anthropic 自行决定的内容。应用内提交表单将插件添加到[社区市场](#community-marketplace)，而不是官方市场。要独立分发插件，请[创建您自己的市场](/docs/zh-CN/plugin-marketplaces)并与用户共享。
</Note>

官方市场包括多个插件类别：

<h3 id="code-intelligence">
  代码智能
</h3>

代码智能插件启用 Claude Code 的内置 LSP 工具，使 Claude 能够跳转到定义、查找引用并在编辑后立即查看类型错误。这些插件配置[语言服务器协议](https://microsoft.github.io/language-server-protocol/)连接，这是为 VS Code 代码智能提供支持的相同技术。在[云会话](/docs/zh-CN/claude-code-on-the-web)中，Claude Code 不启动插件语言服务器，因此 Claude 在那里不会获得 LSP 工具。

在使用这些插件之前，请从下表安装语言服务器二进制文件；插件不会为您安装它。如果您已经安装了语言服务器，当您打开项目时，Claude 可能会提示您安装相应的插件。

| 语言         | 插件                  | 所需二进制文件                      |
| :--------- | :------------------ | :--------------------------- |
| C/C++      | `clangd-lsp`        | `clangd`                     |
| C#         | `csharp-lsp`        | `csharp-ls`                  |
| Go         | `gopls-lsp`         | `gopls`                      |
| Java       | `jdtls-lsp`         | `jdtls`                      |
| Kotlin     | `kotlin-lsp`        | `kotlin-language-server`     |
| Lua        | `lua-lsp`           | `lua-language-server`        |
| PHP        | `php-lsp`           | `intelephense`               |
| Python     | `pyright-lsp`       | `pyright-langserver`         |
| Rust       | `rust-analyzer-lsp` | `rust-analyzer`              |
| Swift      | `swift-lsp`         | `sourcekit-lsp`              |
| TypeScript | `typescript-lsp`    | `typescript-language-server` |

您也可以[为其他语言创建自己的 LSP 插件](/docs/zh-CN/plugins-reference#lsp-servers)。

<Note>
  如果在安装插件后在 `/plugin` 错误选项卡中看到 `Executable not found in $PATH`，请从上表安装所需的二进制文件。
</Note>

<h4 id="what-claude-gains-from-code-intelligence-plugins">
  Claude 从代码智能插件获得的功能
</h4>

安装代码智能插件并且其语言服务器二进制文件可用后，Claude 获得两项功能：

* **自动诊断**：在 Claude 进行的每次文件编辑后，语言服务器报告错误和警告，因此 Claude 看到类型错误、缺失导入和语法问题，无需运行编译器或 linter。如果 Claude 引入错误，它会注意到并在同一轮中修复它。
* **代码导航**：Claude 可以使用语言服务器跳转到定义、查找引用、获取悬停时的类型信息、列出符号、查找实现和追踪调用层次结构。这些操作为 Claude 提供比基于 grep 的搜索更精确的导航，尽管可用性可能因语言和环境而异。

您不需要配置诊断，只需安装插件即可。要自己读取诊断，当 Claude Code 显示指示器（如**在 2 个文件中发现 3 个新诊断问题**）时，请按 **Ctrl+O**。

如果遇到问题，请参阅[代码智能故障排除](#code-intelligence-issues)。

<h3 id="external-integrations">
  外部集成
</h3>

这些插件捆绑预配置的 [MCP servers](/docs/zh-CN/mcp)，以便您可以连接 Claude 到外部服务，无需手动设置：

* **源代码控制**：`github`、`gitlab`
* **项目管理**：`atlassian`（Jira/Confluence）、`asana`、`linear`、`notion`
* **设计**：`figma`
* **基础设施**：`vercel`、`firebase`、`supabase`
* **通信**：`slack`
* **监控**：`sentry`

<h3 id="automatic-security-review">
  自动安全审查
</h3>

`security-guidance` 插件审查 Claude 所做的每项更改是否存在常见漏洞，并指示 Claude 在同一会话中修复发现的问题。有关其检查内容以及如何添加特定于项目的规则，请参阅[在 Claude 编写代码时捕获安全问题](/docs/zh-CN/security-guidance)。

<h3 id="development-workflows">
  开发工作流
</h3>

为常见开发任务添加 skills 和 agents 的插件：

* **commit-commands**：Git 提交工作流，包括提交、推送和 PR 创建
* **pr-review-toolkit**：用于审查拉取请求的专门 agents
* **agent-sdk-dev**：使用 Claude Agent SDK 构建的工具
* **plugin-dev**：用于创建您自己的插件的工具包

<h3 id="output-styles">
  输出样式
</h3>

自定义 Claude 的响应方式：

* **explanatory-output-style**：关于实现选择的教育见解
* **learning-output-style**：用于技能构建的交互式学习模式

<h2 id="community-marketplace">
  社区市场
</h2>

[`anthropics/claude-plugins-community`](https://github.com/anthropics/claude-plugins-community) 上的社区市场托管已通过 Anthropic 自动验证和安全筛选的第三方插件。每个插件都固定到目录中的特定提交 SHA。与官方市场不同，您需要手动添加它：

```shell theme={null}
/plugin marketplace add anthropics/claude-plugins-community
```

然后使用 `claude-community` 市场名称从中安装插件：

```shell theme={null}
/plugin install <plugin-name>@claude-community
```

要将您自己的插件提交到社区市场，请参阅创建插件指南中的[将您的插件提交到社区市场](/docs/zh-CN/plugins#submit-your-plugin-to-the-community-marketplace)。

<h2 id="try-it-add-the-demo-marketplace">
  尝试：添加演示市场
</h2>

Anthropic 还维护一个[演示插件市场](https://github.com/anthropics/claude-code/tree/main/plugins)（`claude-code-plugins`），其中包含展示插件系统可能性的示例插件。与官方市场不同，您需要手动添加此市场。

<Steps>
  <Step title="添加市场">
    在 Claude Code 中，为 `anthropics/claude-code` 市场运行 `plugin marketplace add` 命令：

    ```shell theme={null}
    /plugin marketplace add anthropics/claude-code
    ```

    这会下载市场目录并使其插件对您可用。
  </Step>

  <Step title="浏览可用插件">
    运行 `/plugin` 打开插件管理器。这会打开一个选项卡式界面，您可以使用 **Tab** 循环切换，或使用 **Shift+Tab** 向后切换：

    * **发现**：从所有市场浏览可用插件
    * **已安装**：查看和管理已安装的插件
    * **市场**：添加、删除或更新已添加的市场
    * **错误**：查看任何插件加载错误
    * **统计**：查看[每个 skill 在上下文中的成本以及它的使用频率](/docs/zh-CN/skills#find-unused-skills)，在 `/skill-doctor` 可用的会话中

    转到**发现**选项卡以查看您刚添加的市场中的插件。当您的管理员通过 [`pluginSuggestionMarketplaces`](/docs/zh-CN/settings-reference#pluginsuggestionmarketplaces) 托管设置将市场列入允许列表时，标记为与您当前工作目录相关的插件会在顶部固定，并带有**建议用于此目录**标签。
  </Step>

  <Step title="安装插件">
    选择一个插件以查看其详细信息。详细信息窗格显示插件包含的内容及其成本：

    * **上下文成本**估计，因此您可以查看插件将在每个回合中向您的[上下文窗口](/docs/zh-CN/features-overview#understand-context-costs)添加多少个令牌
    * 插件的**最后更新**日期
    * 一个**将安装**部分，列出插件的命令、agents、skills、hooks 和 MCP 及 LSP 服务器，因此您可以在安装前查看它添加的确切内容

    并非每个插件都提供这些字段背后的数据。对于来自本地或自定义市场的插件，您可能看不到**上下文成本**和**最后更新**行，**将安装**部分可能显示**组件将在安装时被发现**。

    选择安装范围：

    * **用户范围**：在所有项目中为自己安装
    * **项目范围**：为此存储库上的所有协作者安装
    * **本地范围**：仅在此存储库中为自己安装

    例如，选择 **commit-commands**（添加 git 工作流 skills 的插件）并将其安装到您的用户范围。

    您也可以从命令行直接安装：

    ```shell theme={null}
    /plugin install commit-commands@claude-code-plugins
    ```

    请参阅[设置文件](/docs/zh-CN/settings#where-settings-live)以了解有关范围的更多信息。
  </Step>

  <Step title="使用您的新插件">
    如果安装摘要报告 `Run /reload-plugins to activate.`，Claude Code 随后会为您运行该重新加载。如果重新加载警告您的下一条消息会重新读取对话，请运行 `/reload-plugins --force` 来激活插件。

    插件 skills 由插件名称命名空间，因此 **commit-commands** 提供诸如 `/commit-commands:commit` 之类的 skills。

    通过对文件进行更改并运行来尝试：

    ```shell theme={null}
    /commit-commands:commit
    ```

    这会暂存您的更改、生成提交消息并创建提交。

    每个插件的工作方式不同。检查**发现**选项卡中的插件详细信息以查看它提供的命令和 skills，或访问其主页以获取使用指导。
  </Step>
</Steps>

<h2 id="add-marketplaces">
  添加市场
</h2>

使用 `/plugin marketplace add` 命令从不同来源添加市场。

<Tip>
  **快捷方式**：您可以使用 `/plugin market` 代替 `/plugin marketplace`，以及使用 `rm` 代替 `remove`。
</Tip>

* **GitHub 存储库**：`owner/repo` 格式，例如 `anthropics/claude-code`
* **Git URL**：任何 git 存储库 URL，包括 GitLab、Bitbucket 和自托管服务器
* **本地路径**：目录或 `marketplace.json` 文件的直接路径
* **远程 URL**：托管 `marketplace.json` 文件的直接 URL

<h3 id="add-from-github">
  从 GitHub 添加
</h3>

使用 `owner/repo` 格式添加包含 `.claude-plugin/marketplace.json` 文件的 GitHub 存储库，其中 `owner` 是 GitHub 用户名或组织，`repo` 是存储库名称。

例如，`anthropics/claude-code` 指的是由 `anthropics` 拥有的 `claude-code` 存储库：

```shell theme={null}
/plugin marketplace add anthropics/claude-code
```

<h3 id="add-from-other-git-hosts">
  从其他 Git 主机添加
</h3>

通过提供完整 URL 添加 git 市场存储库。对于 `https://` URL，是否包含 `.git` 后缀取决于主机：

* **`github.com` 和 `gitlab.com`**：Claude Code 识别带有或不带 `.git` 后缀的存储库 URL 并克隆它。添加不带后缀的 `gitlab.com` URL 需要 Claude Code v2.1.232 或更高版本。在 v2.1.232 之前，Claude Code 将其视为托管 `marketplace.json` 文件的直接链接。
* **Azure DevOps**：省略后缀。Claude Code 克隆任何路径包含 `/_git/` 的 URL。如果您在 `/_git/` 路径后附加 `.git`，克隆将失败。
* **所有其他主机，包括自管理的 GitLab 服务器**：包含 `.git` 后缀，以便 Claude Code 克隆存储库，而不是将 URL 视为托管 `marketplace.json` 文件的直接链接。对于克隆 URL 不带后缀的主机（如 AWS CodeCommit），请改为在 [`extraKnownMarketplaces`](/docs/zh-CN/settings-reference#extraknownmarketplaces) 中添加市场作为 git 条目。Claude Code 克隆 git 条目，无论其 URL 是否以 `.git` 结尾。

Claude Code 也克隆具有嵌套子组的 `gitlab.com` URL，例如 `https://gitlab.com/group/subgroup/project`。

包含 `https://` 前缀。Claude Code v2.1.196 及更高版本会拒绝没有前缀的主机，例如 `gitlab.com/company/plugins.git`，将其视为无效的 GitHub `owner/repo` 简写，错误消息会告诉您添加前缀。早期版本会将其误读为 GitHub 存储库路径，并在克隆时失败。

使用 HTTPS：

```shell theme={null}
/plugin marketplace add https://gitlab.com/company/plugins.git
```

使用 SSH：

```shell theme={null}
/plugin marketplace add git@gitlab.com:company/plugins.git
```

Claude Code 克隆 SSH 地址，无论其是否以 `.git` 结尾。

要添加特定分支或标签，请在 `#` 后附加 ref：

```shell theme={null}
/plugin marketplace add https://gitlab.com/company/plugins.git#v1.0.0
```

<h3 id="add-from-local-paths">
  从本地路径添加
</h3>

添加包含 `.claude-plugin/marketplace.json` 文件的本地目录：

```shell theme={null}
/plugin marketplace add ./my-marketplace
```

您也可以添加 `marketplace.json` 文件的直接路径：

```shell theme={null}
/plugin marketplace add ./path/to/marketplace.json
```

<h3 id="add-from-remote-urls">
  从远程 URL 添加
</h3>

通过 URL 添加远程 `marketplace.json` 文件：

```shell theme={null}
/plugin marketplace add https://example.com/marketplace.json
```

<Note>
  与基于 Git 的市场相比，基于 URL 的市场有一些限制。如果从基于 URL 的市场安装插件失败，请参阅[故障排除](/docs/zh-CN/plugin-marketplaces#plugins-with-relative-paths-fail-in-url-based-marketplaces)。
</Note>

<h2 id="install-plugins">
  安装插件
</h2>

添加市场后，您可以按名称安装插件：

```shell theme={null}
/plugin install plugin-name@marketplace-name
```

该命令打开该插件的详情，您可以在其中选择[安装范围](/docs/zh-CN/settings#where-settings-live)。当您运行 `/plugin`，转到**发现**选项卡，然后在插件上按 **Enter** 时，您会看到相同的选择：

* **用户范围**：在所有项目中为自己安装
* **项目范围**：为此存储库上的所有协作者安装，这会将插件添加到 `.claude/settings.json`
* **本地范围**：仅在此存储库中为自己安装，不与协作者共享

要在没有交互式步骤的情况下安装，请使用 [`claude plugin install`](/docs/zh-CN/plugins-reference#plugin-install) shell 命令，该命令默认安装到用户范围，除非您传递 `--scope`。对于具有[`command` 源](/docs/zh-CN/plugin-marketplaces#how-users-accept-the-command)的插件，传递 `--yes` 以接受它显示的命令。

您也可能看到具有**托管**范围的插件。这些由管理员通过[托管设置](/docs/zh-CN/managed-settings)安装，无法修改。

Claude Code 在其本地市场目录副本中查找插件。您命名插件的方式控制 Claude Code 是否首先刷新该副本：

* **带有市场名称**：当您安装 `plugin-name@marketplace-name` 时，在会话中或使用 `claude plugin install`，Claude Code 在查找前刷新该市场。即使您关闭了市场的[自动更新](#configure-auto-updates)或设置了 `DISABLE_AUTOUPDATER`，Claude Code 也会运行刷新。在 v2.1.232 之前，Claude Code 在查找前不刷新市场。Claude Code 在以下情况下跳过此刷新：
  * 市场未[从 GitHub、其他 Git 主机或远程 URL 添加](#add-marketplaces)。
  * [种子目录](/docs/zh-CN/plugin-marketplaces#pre-populate-plugins-for-containers)提供市场。
  * Claude Code 在过去 30 秒内刷新了市场。
  * 您设置了 [`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`](/docs/zh-CN/env-vars)。
  * [托管设置](/docs/zh-CN/plugin-marketplaces#managed-marketplace-restrictions)阻止市场，在这种情况下 Claude Code 也拒绝安装。
* **仅插件名称**：当您在会话中运行 `/plugin install plugin-name` 时，Claude Code 仅刷新它也在[后台更新](#configure-auto-updates)的市场，并且仅在查找失败后。当您运行 `claude plugin install plugin-name` 时，Claude Code 读取缓存的目录而不刷新。要安装在上次刷新后发布的插件，请在会话中运行 `/plugin marketplace update <marketplace-name>` 或在 shell 中运行 [`claude plugin marketplace update <marketplace-name>`](/docs/zh-CN/plugin-marketplaces#plugin-marketplace-update)，然后重试安装。

如果命名安装前的刷新失败，例如因为您离线，Claude Code 仍会在缓存目录中查找插件。`claude plugin install` 在其成功消息中报告 `marketplace not refreshed`，`/plugin install` 在插件详情上方或其未找到消息中显示失败。

当您从 `/plugin` 界面安装时，安装摘要告诉您插件在当前会话中是否处于活跃状态：

* `Plugin is now active.`：Claude Code 在安装过程中激活了插件。
* `Run /reload-plugins to activate.`：插件尚未处于活跃状态，因为激活它会[使提示缓存失效](/docs/zh-CN/prompt-caching#enabling-or-disabling-a-plugin)或因为激活尝试失败。Claude Code 随后会为您运行 `/reload-plugins`。如果该重新加载警告提示缓存，请运行 `/reload-plugins --force` 以[在不重启的情况下应用插件更改](#apply-plugin-changes-without-restarting)。
* 如果插件加载失败，摘要会报告失败，`/plugin` **错误**选项卡显示详情。

在 v2.1.221 之前，在您运行 `/reload-plugins` 或重启之前，当前会话中没有安装生效。

`claude plugin install` shell 命令不在会话中运行，因此 Claude Code 在您下次启动 Claude Code 时加载它安装的插件，或当您在已打开的会话中运行 `/reload-plugins` 时。

<Warning>
  在安装插件之前，请确保您信任该插件。Anthropic 不控制插件中包含的 MCP servers、文件或其他软件，也无法验证它们是否按预期工作。检查每个插件的主页以获取更多信息。
</Warning>

<h2 id="manage-installed-plugins">
  管理已安装的插件
</h2>

运行 `/plugin` 并转到**已安装**选项卡以查看、启用、禁用或卸载您的插件。该列表按范围分组并排序，以便您首先看到问题：具有加载错误或未解决依赖项的插件出现在顶部，然后是您的收藏夹，禁用的插件折叠在底部的折叠标题后面。

从列表中您可以：

* 按 `f` 以收藏或取消收藏选定的插件
* 输入以按插件名称或描述筛选
* 按 Enter 打开插件的详细视图并启用、禁用或卸载它

卸载项目的 `.claude/settings.json` 启用的插件时，Claude Code 会询问您指的是哪个范围：仅为您禁用它，这会将覆盖写入您的 `.claude/settings.local.json` 并为项目保留已安装的插件，或为所有人卸载它，这会将其从共享的 `.claude/settings.json` 中删除。

详细视图显示插件贡献的组件：commands、skills、agents、hooks、MCP servers 和 LSP servers。相同的清单也可以从命令行通过 `claude plugin details` 获得。

Claude Code 还在**已安装**选项卡中的**最近未使用**标题下列出您自己安装但至少两周内未使用过的市场插件，跨越至少 10 个会话。详细视图为每个插件显示一条**最后使用**行。使用这些来查找您不再使用但仍在增加启动和上下文成本的插件，然后禁用或卸载它们。

两种类型的插件永远不会被列为未使用：

* 您的组织管理的插件或您使用 `--plugin-dir` 加载的插件
* 贡献主题、输出样式、监视器或工作流的插件，因为这些提供的价值无需跟踪调用

当您的组织使用 [`strictKnownMarketplaces`](/docs/zh-CN/settings-reference#strictknownmarketplaces) 限制市场时，**最近未使用**标题和**最后使用**行都被隐藏。

插件的[语言服务器](/docs/zh-CN/plugins#add-lsp-servers-to-your-plugin)在提供诊断或回答代码导航请求时被计为已使用，因此其服务器在您的会话中处于活跃状态的 LSP 插件不会被列为未使用。在 v2.1.203 之前，无法计算语言服务器活动作为使用，因此贡献 LSP 服务器的插件完全免除，与主题和输出样式插件仍然相同的方式。

在计算语言服务器活动的版本的第一个会话中，还会重置每个尚未记录任何使用的 LSP 插件的使用记录，因此 Claude Code 不会根据在其服务器活动被跟踪之前记录的数据将您之前安装的插件判断为未使用。

当您安装声明依赖项的插件时，安装输出会列出哪些依赖项与其一起自动安装。

您也可以使用直接命令管理插件：

* 当您运行 `/plugin disable`、`/plugin enable` 或 `/plugin uninstall` 时，Claude Code 会打开插件面板以应用更改并保持其打开。按 **Esc** 以在输入另一个命令之前关闭面板。[应用插件更改而不重启](#apply-plugin-changes-without-restarting)描述了更改在您的会话中何时生效。
* 对于脚本编写，请改用 `claude plugin` shell 命令，这些命令不会打开面板。

列出已安装的插件而不打开菜单：

```shell theme={null}
/plugin list
```

传递 `--enabled` 或 `--disabled` 以仅显示处于该状态的插件。

禁用插件而不卸载：

```shell theme={null}
/plugin disable plugin-name@marketplace-name
```

重新启用已禁用的插件：

```shell theme={null}
/plugin enable plugin-name@marketplace-name
```

在这些标识符中，`plugin-name` 是 [marketplace entry](/docs/zh-CN/plugin-marketplaces#plugin-entries) 中插件的 `name`，它可能与插件自己的 `plugin.json` 中的 `name` 不同。

从 Claude Code v2.1.195 开始，`/plugin` 界面中的**启用**和**禁用**适用于两个名称不同的插件，`/plugin enable` 和 `/plugin disable` 接受任一名称。当您在早期版本中禁用此类插件时，Claude Code 报告 `already disabled` 并将其保持启用状态。

完全删除插件：

```shell theme={null}
/plugin uninstall plugin-name@marketplace-name
```

`--scope` 选项允许您使用 CLI 命令针对特定范围：

```shell theme={null}
claude plugin install formatter@your-org --scope project
claude plugin uninstall formatter@your-org --scope project
```

<h3 id="apply-plugin-changes-without-restarting">
  应用插件更改而不重启
</h3>

当您关闭 `/plugin` 菜单时，Claude Code 会为您运行 `/reload-plugins` 以应用您在其中所做的更改，例如安装、启用、禁用和卸载插件。如果重新加载会[使 prompt cache 失效](/docs/zh-CN/prompt-caching#enabling-or-disabling-a-plugin)，它会发出警告并改为保留更改待处理；运行 `/reload-plugins --force` 以无论如何应用它们。如果 Claude 在您关闭菜单时仍在响应，重新加载会在响应完成后运行。

对于在菜单外发生的插件更改，请自己运行 `/reload-plugins`。这些更改包括：

* 您在另一个终端中运行的 `claude plugin` 命令
* 编辑您使用 [`--plugin-dir`](/docs/zh-CN/plugins#test-your-plugins-locally) 加载的插件，同时您开发它
* 插件[自动更新](#configure-auto-updates)，其通知要求您重新加载
* [`--plugin-dir` 文件夹](/docs/zh-CN/plugins#test-your-plugins-locally)中的更改，Claude Code 保留了该更改，因为应用它会使 prompt cache 失效

在 v2.1.268 之前，您在菜单中启用、禁用或卸载的插件，以及在安装期间未激活的安装，保持待处理状态，直到您运行 `/reload-plugins`。

`/reload-plugins` 也在没有交互式终端的会话中运行，例如桌面应用、Agent SDK 和[非交互式模式](/docs/zh-CN/headless)与 `-p`。需要 Claude Code v2.1.260 或更高版本。这些会话中适用两个限制：

* 该命令仅在您直接将其输入到会话中时运行，例如在 `-p` 提示或桌面应用的提示框中。当您通过远程连接（例如[远程控制](/docs/zh-CN/remote-control)或中继聊天消息）发送它时，该命令会拒绝而不重新加载任何内容。
* 重新加载不会连接或断开插件 MCP servers。这些更改在您的下一个会话中生效。

Claude Code 重新加载所有活跃插件并显示插件、skills、agents、hooks、插件 MCP servers 和插件 LSP servers 的计数，在没有交互式终端的会话中省略插件 MCP server 计数。在 skills 计数中，Claude Code 包括插件提供的每个 skill：其 `commands/` 条目和其 `SKILL.md` skills。在 v2.1.246 之前，Claude Code 仅计算 `commands/` 条目，因此它可以重新加载插件的 `SKILL.md` skills 并仍然在摘要中报告 `0 skills`。

重新加载在下一个请求时会产生令牌成本：新加载的组件在附加到对话的内容中宣布自己，而现有历史记录仍然从 prompt cache 读取。提供 MCP servers 的插件在其工具未被 [tool search](/docs/zh-CN/mcp#scale-with-mcp-tool-search) 延迟时成本更高：该更改使缓存失效，下一个请求重新读取整个对话。有关详细信息，请参阅[启用或禁用插件](/docs/zh-CN/prompt-caching#enabling-or-disabling-a-plugin)。

<h2 id="manage-marketplaces">
  管理市场
</h2>

您可以通过交互式 `/plugin` 界面或 CLI 命令管理市场。

<h3 id="use-the-interactive-interface">
  使用交互式界面
</h3>

运行 `/plugin` 并转到**市场**选项卡以：

* 查看所有已添加的市场及其来源和状态
* 添加新市场
* 更新市场列表以获取最新插件
* 删除您不再需要的市场

<h3 id="use-cli-commands">
  使用 CLI 命令
</h3>

您也可以使用直接命令管理市场。

列出所有配置的市场：

```shell theme={null}
/plugin marketplace list
```

刷新市场的插件列表：

```shell theme={null}
/plugin marketplace update marketplace-name
```

删除市场：

```shell theme={null}
/plugin marketplace remove marketplace-name
```

<Warning>
  删除市场将卸载您从中安装的任何插件。
</Warning>

<h3 id="configure-auto-updates">
  配置自动更新
</h3>

Claude Code 可以在启动后在后台自动更新市场及其已安装的插件。为市场启用自动更新后，Claude Code 会刷新市场数据并将已安装的插件更新到磁盘上的最新版本。

Claude Code 在您的会话启动后检查市场和插件更新，延迟时间最多为十分钟，因此运行中的会话继续使用它在启动时加载的版本。如果任何插件已更新，您将看到提示您运行 `/reload-plugins` 的通知，或新版本在您下次启动时加载。

自动更新还会排除其市场条目声明 `headersHelper` 的插件：Claude Code [既不运行命令也不下载该路径上的存档](/docs/zh-CN/plugin-marketplaces#installs-and-updates-that-refuse-the-command-instead-of-asking)；该部分说明 Claude Code 何时在 `/plugin` 错误选项卡中列出插件，以便您可以从其自己的视图中更新它。

Claude Code 更新具有[`command` 源](/docs/zh-CN/plugin-marketplaces#command-sources)的插件，其更新频率与市场自动更新设置和 `DISABLE_AUTOUPDATER` 不同。相反，它[每个会话重新运行一次命令](/docs/zh-CN/plugin-marketplaces#when-claude-code-re-runs-the-command)，当其[哈希](/docs/zh-CN/plugins-reference#version-management)已更改时，将输出安装为新的插件版本。

通过 UI 为单个市场切换自动更新：

1. 运行 `/plugin` 打开插件管理器
2. 选择**市场**
3. 从列表中选择市场
4. 选择**启用自动更新**或**禁用自动更新**

`claude-plugins-official` 和大多数其他官方 Anthropic 市场默认启用自动更新。第三方和本地开发市场默认禁用自动更新。

管理员还可以在托管设置中的每个 [`extraKnownMarketplaces`](/docs/zh-CN/settings-reference#extraknownmarketplaces) 条目上设置 `"autoUpdate": true` 以为组织市场启用自动更新，而无需每个用户都切换它。

要禁用 Claude Code 和从市场获取的插件的自动更新，请设置 `DISABLE_AUTOUPDATER` 环境变量。具有[`command` 源](/docs/zh-CN/plugin-marketplaces#command-sources)的插件遵循其自己的每个会话一次的重新解析。有关详细信息，请参阅[自动更新](/docs/zh-CN/setup#auto-updates)。

要在禁用 Claude Code 自动更新的同时保持插件自动更新启用，请设置 `FORCE_AUTOUPDATE_PLUGINS=1` 以及 `DISABLE_AUTOUPDATER`：

```bash theme={null}
export DISABLE_AUTOUPDATER=1
export FORCE_AUTOUPDATE_PLUGINS=1
```

<h2 id="configure-team-marketplaces">
  配置团队市场
</h2>

团队管理员可以通过将市场配置添加到 `.claude/settings.json` 来为项目设置自动市场安装。当团队成员[信任存储库文件夹](/docs/zh-CN/permissions#what-runs-before-you-trust-a-folder)后，Claude Code 会自动添加这些市场，无需进一步提示。

从 Claude Code v2.1.195 开始，添加市场不会在任何加载插件的路径上安装来自外部源的插件。仅由项目的 `.claude/settings.json` 启用且来自外部源（如 GitHub 存储库或 npm 包）的插件在团队成员安装之前不会加载。在此之前，Claude Code 会将该插件报告为未安装，并显示要运行的 `claude plugin install` 命令。

将 `extraKnownMarketplaces` 添加到您项目的 `.claude/settings.json`：

```json theme={null}
{
  "extraKnownMarketplaces": {
    "my-team-tools": {
      "source": {
        "source": "github",
        "repo": "your-org/claude-plugins"
      }
    }
  }
}
```

有关完整配置选项（包括 `extraKnownMarketplaces` 和 `enabledPlugins`），请参阅[插件设置](/docs/zh-CN/settings-reference#plugin-settings)。

<h2 id="security">
  安全性
</h2>

插件和市场是高度受信任的组件，可以使用您的用户权限在您的机器上执行任意代码。仅从您信任的来源安装插件和添加市场。组织可以使用[托管市场限制](/docs/zh-CN/plugin-marketplaces#managed-marketplace-restrictions)限制用户允许添加的市场。

<h2 id="troubleshooting">
  故障排除
</h2>

<h3 id="/plugin-command-not-recognized">
  /plugin 命令无法识别
</h3>

如果您看到"未知命令"或 `/plugin` 命令未出现：

1. **检查您的版本**：运行 `claude --version` 以查看安装的内容。
2. **更新 Claude Code**：
   * **Homebrew**：`brew upgrade claude-code`，或如果您安装了该 cask，则为 `brew upgrade claude-code@latest`
   * **npm**：`npm install -g @anthropic-ai/claude-code@latest`
   * **本地安装程序**：从[设置](/docs/zh-CN/setup)重新运行安装命令
3. **重启 Claude Code**：更新后，重启您的终端并再次运行 `claude`。

<h3 id="common-issues">
  常见问题
</h3>

如果插件 skills 未出现，使用 `rm -rf ~/.claude/plugins/cache` 清除缓存，重启 Claude Code，然后重新安装插件。

有关详细的故障排除和解决方案，请参阅市场指南中的[故障排除](/docs/zh-CN/plugin-marketplaces#troubleshooting)。有关调试工具，请参阅[调试和开发工具](/docs/zh-CN/plugins-reference#debugging-and-development-tools)。

<h3 id="code-intelligence-issues">
  代码智能问题
</h3>

* **语言服务器未启动**：验证二进制文件已安装且在您的 `$PATH` 中可用。检查 `/plugin` 错误选项卡以获取详细信息。
* **高内存使用**：`rust-analyzer` 和 `pyright` 等语言服务器在大型项目上可能消耗大量内存。如果您遇到内存问题，请使用 `/plugin disable <plugin-name>` 禁用插件，并改为依赖 Claude 的内置搜索工具。
* **monorepos 中的误报诊断**：如果工作区配置不正确，语言服务器可能会报告内部包的未解析导入错误。这些不会影响 Claude 编辑代码的能力。

<h2 id="next-steps">
  后续步骤
</h2>

* **构建您自己的插件**：请参阅[插件](/docs/zh-CN/plugins)以创建 skills、agents 和 hooks
* **创建市场**：请参阅[创建插件市场](/docs/zh-CN/plugin-marketplaces)以将插件分发给您的团队或社区
* **技术参考**：请参阅[插件参考](/docs/zh-CN/plugins-reference)以获取完整规范
