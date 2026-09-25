> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 发布和分发插件

> 通过您自己的市场或 Anthropic 的社区市场发布 Claude Code 插件，包括发布前检查清单以及用户如何获取更新。

发布 Claude Code 插件意味着在市场中列出它，市场是一个 JSON 目录，列出插件及其获取位置，这样其他人可以按名称安装它并接收您的更新。您可以运行自己的市场或将您的插件提交到 Anthropic 的社区市场。要在不发布的情况下共享插件，请将插件的目录或其 `.zip` 发送给人们以供他们自己加载。

本页面适用于已准备好共享的工作插件的作者。

<Note>
  这些情况在其他页面上有介绍：

  * **您的插件还未完成**：从[创建插件](/docs/zh-CN/plugins/create)开始
  * **您维护的 CLI 或 SDK 在官方市场中有插件**：请参阅[从您的 CLI 推荐您的插件](/docs/zh-CN/plugins/cli-hints)
</Note>

从[选择如何分发](#choose-how-to-distribute)开始，比较分发选项。如果您已经知道您的路线，请转到[为发布准备您的插件](#prepare-your-plugin-for-release)，然后按照您的路线部分了解要告诉用户什么以及他们如何接收您的更新。

<h2 id="choose-how-to-distribute">
  选择如何分发
</h2>

根据谁需要安装插件来选择分发选项：

| 路线                                                      | 谁可以安装                                         | 您需要什么                                                        | 用户是否自动获取您的更新？ |
| :------------------------------------------------------ | :-------------------------------------------- | :----------------------------------------------------------- | :------------ |
| [无市场](#share-a-plugin-without-a-marketplace)            | 您发送插件文件夹或其 `.zip` 的人                          | 插件的文件夹                                                       | 无。他们加载您发送的副本  |
| [您自己的市场](#publish-through-your-own-marketplace)         | 任何可以访问存储库的人，可以是您的团队可以克隆的私有存储库                 | 一个 git 存储库或其他具有列出您的插件的 `.claude-plugin/marketplace.json` 的主机 | 关闭            |
| [Anthropic 的社区市场](#submit-to-the-community-marketplace) | 任何添加 `anthropics/claude-plugins-community` 的人 | 通过插件目录提交表单的提交                                                | 关闭            |

自动更新是用户端的每个市场设置，在后台获取新版本。

<h2 id="prepare-your-plugin-for-release">
  为发布准备您的插件
</h2>

名称、版本、验证和从市场安装决定了发布是否对安装它的人有效。在第一次发布前检查它们，以及在之后的每次发布前再次检查。

<Steps>
  <Step title="选择永久名称">
    用户通过 `name@marketplace` 安装、启用和配置您的插件，因此重命名的插件对每个现有安装都是不同的插件。选择一个 kebab-case 名称，例如 `deploy-helper`，因为 `claude plugin validate` 会对其他形式发出警告，并将其视为永久的。在 `plugin.json` 中设置 `displayName` 以获取用户看到的标签。
  </Step>

  <Step title="决定如何版本化">
    如果您在 `plugin.json` 中设置 `version` 并稍后推送提交而不更改它，`claude plugin update` 会打印 `<name> is already at the latest version (1.0.0).`，用户保留旧副本。要么在每次发布时增加 `version`，要么在 git 托管的市场中省略它，以便 Claude Code 改用提交 SHA。请参阅[版本和更新](/docs/zh-CN/plugins/loading#versions-and-updates)。
  </Step>

  <Step title="验证">
    在您的 shell 中，运行 `claude plugin validate --strict ./your-plugin`。干净的运行会打印 `✔ Validation passed`。

    * **在 CI 中**：保持 `--strict`，它也会因为警告（例如未知的清单字段或缺少 `version`）而以退出代码 1 失败运行。如果您在上一步中选择省略 `version`，则删除 `--strict`。
    * **路径**：验证报告不以 `./` 开头的组件路径。在 hook 命令和 MCP 服务器配置中，将文件引用为 `${CLAUDE_PLUGIN_ROOT}/...`。请参阅[路径规则](/docs/zh-CN/plugins/manifest-reference#path-rules)。
  </Step>

  <Step title="从本地市场安装它">
    在您的 shell 中，使用 `claude plugin marketplace add ./path-to-marketplace` 添加列出插件的本地市场，从中安装插件，并启动会话以确认它加载。

    * 对于最小的有效市场，请参阅[创建市场](/docs/zh-CN/plugins/create-marketplace)。
    * 要了解安装是加载您的源目录还是缓存副本，请参阅[就地和复制的插件](/docs/zh-CN/plugins/loading#in-place-and-copied-plugins)。
  </Step>

  <Step title="填写用户看到的元数据">
    在 `plugin.json` 中设置 `description`、`author`、`homepage` 和 `repository`，并在插件根目录添加 `README.md`。`homepage` 必须解析为 URL。[清单参考](/docs/zh-CN/plugins/manifest-reference#fields)列出了每个字段。
  </Step>

  <Step title="运行您的 eval 套件">
    如果您有 eval 套件，在您的 shell 中运行 `claude plugin eval`。它运行插件的测试用例并对结果进行评分，这在您更改插件时捕获回归。请参阅[使用 evals 测试插件](/docs/zh-CN/plugin-evals)。
  </Step>
</Steps>

<h2 id="share-a-plugin-without-a-marketplace">
  不使用市场共享插件
</h2>

如果插件在 git 存储库中，人们可以克隆它并加载检出，或从他们的 shell 启动 Claude Code，使用 `--plugin-url` 指向您附加到发布的 `.zip`。要获取您的下一个版本，他们拉取或再次下载。如果它不在存储库中，请将目录或其 `.zip` 发送给他们。他们可以通过以下两种方式之一加载它：

* **对于一个会话**：他们从他们的 shell 启动 Claude Code，使用 `claude --plugin-dir ./deploy-helper`，其中路径是克隆、解压的文件夹或 `.zip` 本身。请参阅[为一个会话加载插件的标志](/docs/zh-CN/plugins/cli-reference#flags-that-load-a-plugin-for-one-session)。
* **对于每个会话**：他们将插件目录（带有其 `.claude-plugin/plugin.json`）移到 `~/.claude/skills/` 下，以便 Claude Code [在每个会话中加载它](/docs/zh-CN/plugins/loading#find-where-a-plugin-came-from)。

将 `.claude-plugin/marketplace.json` 添加到同一存储库是让人们按名称安装和使用命令更新的方式；请参阅[通过您自己的市场发布](#publish-through-your-own-marketplace)。

<h3 id="ship-a-plugin-with-your-own-tool">
  使用您自己的工具发布插件
</h3>

如果您维护 CLI 或 SDK，在市场中发布插件，并让您的安装程序或安装后消息运行或打印用户需要的两个命令：`claude plugin marketplace add <source>`，然后 `claude plugin install <name>@<marketplace>`。对于当某人使用您的工具时的会话内发现，请参阅[从您的 CLI 推荐您的插件](/docs/zh-CN/plugins/cli-hints)。

<h2 id="publish-through-your-own-marketplace">
  通过您自己的市场发布
</h2>

您自己的市场是一个 `.claude-plugin/marketplace.json` 文件，列出您的插件，添加到 git 存储库。一旦文件在存储库中，插件就会发布，无需提交表单。您可以将文件保留在插件自己的存储库中或单独的存储库中。

<h3 id="add-the-marketplace-file-to-your-repository">
  将市场文件添加到您的存储库
</h3>

要从插件自己的存储库发布，请在 `.claude-plugin/` 中的 `plugin.json` 旁边保存市场文件，其中一个条目的 `source` 是 `"./"` 即存储库根目录。给条目与 `plugin.json` 相同的 `name`，根据[保持条目名称和清单名称相同](/docs/zh-CN/plugins/create-marketplace#keep-the-entry-name-and-the-manifest-name-the-same)：

```json .claude-plugin/marketplace.json theme={null}
{
  "name": "your-marketplace",
  "owner": { "name": "Your Name" },
  "plugins": [
    { "name": "deploy-helper", "source": "./" }
  ]
}
```

在您的 shell 中，在推送前在存储库中运行 `claude plugin validate .` 以检查文件。

[创建市场](/docs/zh-CN/plugins/create-marketplace)涵盖了一个存储库中有多个插件的布局。

<h3 id="control-who-can-install">
  控制谁可以安装
</h3>

任何可以克隆存储库的人都可以从中安装，因此如果存储库是私有的，市场也是私有的。对于 git 存储库以外的主机，请参阅[托管市场](/docs/zh-CN/plugins/host-marketplace)。要到达整个公司的每个人，包括不使用 git 的人，请参阅[向整个公司推出](/docs/zh-CN/plugins/host-marketplace#roll-out-to-a-whole-company)。

<h3 id="tell-users-how-to-install">
  告诉用户如何安装
</h3>

告诉您的用户添加市场，然后从他们的 shell 安装插件，用您的替换源和名称：

* 添加市场一次：`claude plugin marketplace add your-org/your-marketplace`，其中参数是 GitHub `owner/repo` 简写、URL 或路径
* 安装插件：`claude plugin install deploy-helper@your-marketplace`
* 或从会话内同时执行两者：`/plugin install deploy-helper --marketplace your-org/your-marketplace`。需要 Claude Code v2.1.275 或更高版本。请参阅[在一个命令中添加市场和安装](/docs/zh-CN/plugins/install#add-a-marketplace-and-install-in-one-command)

<h3 id="ship-updates-to-users">
  向用户发布更新
</h3>

用户在请求时或为您的市场启用自动更新时接收发布：

* **按请求**：用户的 shell 中的 `claude plugin update deploy-helper@your-marketplace` 刷新市场，当您的插件版本更改时安装新副本
* **自动更新**：默认为您的市场关闭。请参阅[启用自动更新](/docs/zh-CN/plugins/host-marketplace#turn-on-auto-update)。启用后，它在会话启动后的延迟后执行与 `claude plugin update` 相同的操作

[安装插件](/docs/zh-CN/plugins/install)涵盖用户端命令，[自动更新何时运行](/docs/zh-CN/plugins/loading#when-auto-update-runs)涵盖时间。

<h2 id="submit-to-the-community-marketplace">
  提交到社区市场
</h2>

Anthropic 的社区市场 `claude-community` 是列出通过插件目录提交表单提交的插件的公共市场。

用户在 Claude Code 会话中使用 `/plugin marketplace add anthropics/claude-plugins-community` 添加社区市场，并从中安装为 `@claude-community`。

关于社区市场与官方市场的区别，请参阅 [Anthropic 的市场](/docs/zh-CN/plugins/anthropic-marketplaces)。

要将您的插件提交到社区市场，请使用以下应用内表单之一：

* **claude.ai**：[claude.ai/admin-settings/directory/submissions/plugins/new](https://claude.ai/admin-settings/directory/submissions/plugins/new)
* **Console**：[platform.claude.com/plugins/submit](https://platform.claude.com/plugins/submit)

claude.ai 表单需要 Team 或 Enterprise 组织以及目录权限，Owners 默认持有该权限。不属于 Team 或 Enterprise 组织的个人作者可以改用 Console 表单。

在您的 shell 中，在提交前本地运行 `claude plugin validate ./your-plugin`，用您的插件目录的路径替换 `./your-plugin`。当验证通过时，Claude Code 打印 `✔ Validation passed`，或如果有警告则打印 `✔ Validation passed with warnings`。警告不会使验证失败；添加 `--strict` 以将它们视为错误。

列出的插件出现在 [`anthropics/claude-plugins-community`](https://github.com/anthropics/claude-plugins-community) 目录中，在几乎所有情况下都固定到特定的提交 SHA。

提交和您的插件出现在 `marketplace.json` 中之间可能会有延迟。要检查您的插件是否可安装，请在[社区目录](https://github.com/anthropics/claude-plugins-community/blob/main/.claude-plugin/marketplace.json)中搜索其名称。

官方市场 `claude-plugins-official` 不通过这些表单接受提交。如果您与 Anthropic 合作伙伴联系合作，请询问他们关于官方市场列表的信息。

<h2 id="ship-updates-renames-and-removals">
  发布更新、重命名和删除
</h2>

<h3 id="release-a-new-version">
  发布新版本
</h3>

如果您通过您自己的市场发布，并且您的 `plugin.json` 设置了 `version`，请增加它并推送。运行 `claude plugin update` 或启用自动更新的用户然后接收新版本，如[向用户发布更新](#ship-updates-to-users)下所述。

<h3 id="tag-a-release">
  标记发布
</h3>

当其他插件在您的上声明版本范围时，在 git 中标记发布，因为这些范围针对标记进行解析。否则您不需要标记。

要标记，请从插件目录在您的 shell 中运行 `claude plugin tag`。它创建一个 `{name}--v{version}` 标记。添加 `--push` 以将标记发送到 `origin`。[`plugin tag` 参考](/docs/zh-CN/plugins/cli-reference#plugin-tag)列出了其标志。

<h3 id="rename-or-remove-a-plugin">
  重命名或删除插件
</h3>

永远不要更改已发布插件的 `name`。重命名后，已安装它的用户会丢失插件，因为他们的安装记录在旧名称下。您的市场文件中的 `renames` 条目会改为迁移它们。当您想要不同的标签时，更改 `displayName`。

如果重命名是不可避免的，请使用市场文件的 `renames` 映射，以便现有安装迁移而不是因为 [`Plugin "<name>" not found in marketplace`](/docs/zh-CN/plugins/troubleshooting#plugin-not-found-in-marketplace) 而失败。要从市场中删除插件，或获取完整的 `renames` 详细信息，请参阅托管页面上的[重命名或删除插件](/docs/zh-CN/plugins/host-marketplace#rename-or-remove-a-plugin)。[市场参考](/docs/zh-CN/plugins/marketplace-reference#top-level-fields)有该字段。

<h2 id="declare-dependencies">
  声明依赖项
</h2>

如果您的插件需要来自同一市场的另一个插件被启用，请在 `plugin.json` 的 `dependencies` 数组中列出它。每个条目是一个裸名称或具有 semver `version` 范围的对象。当用户安装您的插件时，Claude Code 也会安装并启用依赖项。

[插件依赖项](/docs/zh-CN/plugins/dependencies)涵盖范围语法、跨市场依赖项以及用户如何修剪他们不再需要的依赖项。

<h2 id="next-steps">
  后续步骤
</h2>

* [托管和维护市场](/docs/zh-CN/plugins/host-marketplace)：发布新版本并让用户保持最新
* [插件依赖项](/docs/zh-CN/plugins/dependencies)：声明和版本化您的插件所依赖的插件
* [从您的 CLI 推荐您的插件](/docs/zh-CN/plugins/cli-hints)：提示您的 CLI 的 Claude Code 用户安装插件
* [测量插件成本和使用情况](/docs/zh-CN/plugins/measure)：查看您的插件在上下文中的成本以及人们是否使用它
