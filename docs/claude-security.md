> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 扫描代码库中的漏洞

> 安装 Claude Security 插件，在 Claude Code 会话中扫描代码库中的漏洞，并将发现的问题转化为由您审查和应用的补丁。

Claude Security 插件会在 Claude Code 会话中对您的代码库运行多 Agent 漏洞扫描。[默认情况下](#choose-a-scan-effort)，当您扫描整个仓库时，Claude Agent 会梳理您的架构、搜寻漏洞，并在撰写报告之前独立审查每一个发现的问题。

使用该插件可以扫描整个仓库，或[仅扫描一组变更](#scan-only-your-changes)，例如某个分支的 diff、某个 Pull Request 的 diff 或单个提交。然后，将您选择的发现问题转化为补丁，由您自行审查和应用。请仅扫描您拥有或有权限测试的代码。

您可以在会话、shell 或 GitHub Actions 中运行扫描：

* **在会话中**：使用[您在 Claude Code 中可访问的任意模型](#models-and-providers)[扫描并修复您的代码库](#scan-and-fix-your-codebase)
* **从 shell 中**：使用 [`claude-security` 命令](#scan-from-your-shell)
* **在 GitHub Actions 中**：添加该插件的作业以[扫描 Pull Request](#scan-pull-requests-in-github-actions)

在会话或 shell 中运行的扫描会计入您的[用量](https://code.claude.com/docs/zh-CN/costs.md)，而在 GitHub Actions 中运行的扫描会计入拥有该作业模型凭据的账户。

> [!NOTE]
>
> 如果您需要一项监控仓库的托管服务，或希望在 [Claude Mythos](https://platform.claude.com/docs/en/about-claude/models/introducing-claude-fable-5-and-claude-mythos-5) 上运行扫描，请参阅 [Claude Security](https://claude.com/product/claude-security) 产品，该产品在 Enterprise 计划中提供。

该插件也有别于 Claude Code 中已有的审查工具：[安全指导插件](https://code.claude.com/docs/zh-CN/security-guidance.md)会在 Claude 编写代码时对其进行审查，而 [Code Review](https://code.claude.com/docs/zh-CN/code-review.md) 会审查 Pull Request。有关这些层级如何叠加配合，请参阅[该插件如何与其他安全工具配合使用](#how-the-plugin-fits-with-other-security-tools)。

## 前提条件

要运行此插件，您需要：

* 付费计划、Anthropic API 访问权限或[第三方提供商](#models-and-providers)，用于扫描编排其 Agent 时所使用的[动态工作流](https://code.claude.com/docs/zh-CN/workflows.md)。在 Pro 计划中，请从 `/config` 中的 Dynamic workflows 行启用它们。
* Python 3.9 或更高版本，并以 `python3` 的形式在您的 `PATH` 中可用。可使用 `python3 --version` 进行检查。该插件的工具仅使用 Python 标准库。
* Linux、macOS 或 Windows。
* Git，用于变更扫描以及将发现的问题转换为补丁；这些任务不支持其他版本控制系统。完整扫描可在任何目录中运行，无论是否使用版本控制。

## 模型和提供商

该插件本身不发起任何模型调用，因此在您的会话中运行扫描不需要单独的 API 密钥或提供商设置。

* **模型**：负责查找漏洞、验证发现结果以及编写和审查补丁的 Agent 均在[您会话所用的模型](https://code.claude.com/docs/zh-CN/sub-agents.md#choose-a-model)上运行。如需更改，请在开始扫描之前于会话中运行 [`/model`](https://code.claude.com/docs/zh-CN/model-config#setting-your-model)。少数辅助步骤改用 [`sonnet` 别名](https://code.claude.com/docs/zh-CN/model-config#model-aliases)。
* **提供商**：扫描可在付费计划、Anthropic API 访问权限或[第三方提供商](https://code.claude.com/docs/zh-CN/third-party-integrations.md)上运行，例如 [Amazon Bedrock](https://code.claude.com/docs/zh-CN/amazon-bedrock.md)、[Google Cloud 的 Agent Platform](https://code.claude.com/docs/zh-CN/google-vertex-ai.md) 或 [Microsoft Foundry](https://code.claude.com/docs/zh-CN/microsoft-foundry.md)。

在第三方提供商上，`sonnet` 别名解析到的版本可能与在 Anthropic API 上不同。如果您的账户无法使用该版本，请[固定您的模型版本](https://code.claude.com/docs/zh-CN/model-config.md#pin-models-for-third-party-deployments)，包括 `ANTHROPIC_DEFAULT_SONNET_MODEL`。

[自动模型回退](https://code.claude.com/docs/zh-CN/model-config.md#automatic-model-fallback)会重新运行触发模型安全防护措施的请求。在 Amazon Bedrock、Google Cloud 的 Agent Platform 和 Microsoft Foundry 上，根据[您的部署设置方式](https://code.claude.com/docs/zh-CN/model-config.md#enable-fallback-on-bedrock-agent-platform-and-foundry)，该请求可能改为以拒绝消息结束。

## 安装插件

在 VS Code 扩展或桌面应用中，按照[安装插件](https://code.claude.com/docs/zh-CN/plugins/install.md#install-a-plugin)中的说明进行安装。

在终端中，运行 `claude` 启动 Claude Code，然后在其输入框中输入以下内容，从 [Anthropic 官方市场](https://code.claude.com/docs/zh-CN/plugins/anthropic-marketplaces.md)进行安装：

```text theme={null}
/plugin install claude-security@claude-plugins-official
```

该命令会打开插件的详细信息，您可以在其中选择一个[安装作用域](https://code.claude.com/docs/zh-CN/plugins/install.md#install-a-plugin)以开始安装。选择用户作用域可使该插件在每个项目中可用。

如果安装失败，请根据 Claude Code 报告的消息进行处理：

* `Marketplace "claude-plugins-official" not found`：使用 `/plugin marketplace add anthropics/claude-plugins-official` 添加该市场，然后重试安装。
* `Plugin "claude-security" not found in marketplace "claude-plugins-official"`：您本地的市场目录副本可能已过期。使用 `/plugin marketplace update claude-plugins-official` 刷新它，然后重试安装。如果重试后仍显示相同的消息，请参阅[市场中未找到插件](https://code.claude.com/docs/zh-CN/plugins/troubleshooting.md#plugin-not-found-in-marketplace)。

查看安装摘要。如果其中显示 `Run /reload-plugins to apply.`，请参阅[无需重启即可应用插件更改](https://code.claude.com/docs/zh-CN/plugins/cli-reference.md#reload-plugins)，以在当前会话中激活该插件。

插件激活后，即可开始[扫描并修复您的代码库](#scan-and-fix-your-codebase)。

### 更新插件

如果某个命令或选项无法正常工作，请[检查](https://code.claude.com/docs/zh-CN/plugins/install.md#install-a-plugin)您的插件版本是否为 0.13.0 或更高版本。

如需自行更新插件，请参阅[立即更新插件](https://code.claude.com/docs/zh-CN/plugins/install.md#update-plugins-now)。除非关闭了自动更新，否则 Claude Code 也会[自动更新插件](https://code.claude.com/docs/zh-CN/plugins/install.md#keep-plugins-updated)。

### 卸载插件

如需移除插件，请从 `/plugin` 菜单中将其卸载，或在终端中运行 `claude plugin uninstall claude-security`。

## 扫描并修复您的代码库

在 Claude Code 提示符处，`/claude-security` 会打开一个任务菜单，例如扫描代码库、扫描一组更改以及建议补丁。标准流程是先运行完整扫描，然后将其发现转化为补丁：

**1. 从干净的工作树开始**

如果您之后需要补丁，请提交所有更改和未跟踪的文件，或者在 shell 中使用 `git stash -u` 将它们暂存，直到扫描结束。

**2. 打开 Claude Security 菜单**

在您的仓库中启动 Claude Code，运行 `/claude-security`，然后选择 **Scan codebase**。

**3. 选择扫描内容**

插件会先读取您的仓库，然后提供扫描整个仓库或某个重点区域的选项。选择整个仓库，或者回答"I don't know"，插件会根据仓库的大小选择一个合理的默认值。

**4. 确认运行**

扫描可能需要一段时间，可能会消耗大量 token，并且在完成之前需要保持 Claude Code 处于打开状态。在您确认之前，不会运行任何内容。

**5. 阅读报告**

当扫描在终端会话中启动时，插件的[实时视图](#watch-a-running-scan)（默认开启）会打开，并显示代码的实时地图。您也可以在终端中使用 [`/workflows`](https://code.claude.com/docs/zh-CN/workflows#watch-the-run) 跟踪每个阶段。结果会写入您启动 Claude Code 的文件夹中一个带时间戳的目录，详见[阅读扫描结果](#read-the-scan-results)。

**6. 将发现转化为补丁**

再次运行 `/claude-security` 并选择 **Suggest patches**，然后选择要处理的发现。经过审查的补丁会写入报告的 `patches/` 文件夹；[修复发现](#fix-findings)介绍了每个补丁如何构建和审查，以及之后需要检查的内容。

**7. 应用您接受的补丁**

阅读每个补丁的[说明](#patches-are-never-applied-automatically)，然后在仓库根目录的 shell 中使用 `git apply` 应用该补丁，并为其单独创建一个 Pull Request。补丁不会被自动应用。

您也可以直接请求某个任务，将其作为命令的参数，例如 `/claude-security scan my branch`。或者用自然语言提出请求，例如"使用 Claude Security 插件扫描整个仓库"。该插件在[自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md)下效果最佳，该模式允许扫描在每一步都无需权限提示即可继续进行。

### 查看正在运行的扫描

当您在终端会话中启动扫描时，插件的 [mod](https://code.claude.com/docs/zh-CN/plugins/mods/overview.md) 会在一个窗格或提示符上方的一条横栏中打开扫描的实时视图。您需要 Claude Code v2.1.292 或更高版本。

* **关闭**：按 `Ctrl+X X` 将窗格缩小为横栏，再按一次隐藏横栏
* **重新打开**：在扫描运行期间输入 `/claude-security-view`
* **关闭该功能**：输入 `/plugin configure claude-security` 并更改 **Live scan view**

### 继续已停止的扫描

如果会话中的扫描在写入报告之前停止，您可以继续该扫描，除非会话显示安全防护措施停止了扫描中的某个 Agent。在这种情况下，请改为启动新的扫描。

要继续扫描，请保持您的检出和扫描的 `CLAUDE-SECURITY-<timestamp>/` 目录不变，并在运行该扫描的会话中输入 `/claude-security continue the scan`。如果您已关闭该会话，请先[恢复它](https://code.claude.com/docs/zh-CN/sessions.md#resume-a-session)。

### 仅扫描您的更改

当您的分支包含其基础分支所没有的提交时，`/claude-security` 菜单会提供仅扫描该 diff 的选项，以便您在合并之前检查分支。在进行更改扫描之前，请提交您希望扫描的编辑，并使用 `git stash -u` 暂存其余内容，或者运行会读取工作树的[完整扫描](#scan-and-fix-your-codebase)。

要扫描您的某个打开的 Pull Request 或单个提交，请先在 shell 中将其检出，然后在 Claude Code 提示符处提出请求：

* **Pull Request**：运行 `git checkout feature-auth`（指定 Pull Request 的分支名称），然后运行 `/claude-security scan my branch`
* **单个提交**：运行 `git checkout abc1234`，然后运行 `/claude-security scan commit abc1234`

### 限定大型仓库的扫描范围

对于大型仓库，请一次扫描一个区域，而不是整个目录树。从插件提供的重点范围中选择一个，例如您的 API 层或身份验证代码，运行规模会根据您的选择自动调整。报告的覆盖范围部分会说明哪些内容已检查、哪些未检查。您可以随时对其他区域运行另一次扫描。

### 选择扫描力度

扫描力度决定扫描的深度。在 `low`、`medium` 和 `high` 下，您的发现都会经过相同的独立验证。

* **`low`**：快速初步检查。扫描代码库时会跳过架构图。
* **`medium`**：默认值。
* **`high`**：包含 `medium` 的所有内容，并在同一代码中更深入地查找漏洞。

在 Claude Code 提示符处的请求中指明扫描力度，例如针对代码库的某一部分或针对您的更改：

* **代码库的某一部分**：`/claude-security scan my public API code at high`
* **您的更改**：`/claude-security scan my branch at high`

如果对大型仓库的扫描发现的问题少于预期，请选择 `high` 扫描力度、再扫描一次，或两者都做。每种方式都可能报告单次 `medium` 扫描未发现的问题。

代码库扫描可能会消耗大量 token，成本随仓库大小、扫描力度和模型的增加而增长。要减少花费，请扫描[较小的区域](#scope-large-repositories)、选择 `low`，或在开始之前[更改模型](#models-and-providers)。

### 运行更彻底的更改扫描

将[扫描力度](#choose-a-scan-effort)替换为 `--change-scan-effort high`，例如 `/claude-security scan my branch --change-scan-effort high`。与 `high` 扫描力度相比，会有更多 Agent 读取该更改，因此扫描耗时更长，成本也会高出数倍。其报告还会列出早于该更改就已存在的漏洞。

### 向扫描提供系统相关信息

要向扫描提供有关您系统的信息，请指定所扫描文件夹中最多 10 个文件，或输入您自己的说明。

在您的请求中提供这些内容，或在扫描询问时提供：

```text theme={null}
/claude-security scan this repository, and read docs/arch.md first
```

请仅提供最新且您信任的说明和文件，因为错误的陈述可能会掩盖漏洞。同时请保持 README 和 `SECURITY.md` 为最新状态，因为扫描会参考其中关于受信任用户和预期行为的描述。

要从此扫描[获取补丁](#scan-and-fix-your-codebase)，请提交您指定的文件，而不是暂存它们。

### 阅读扫描结果

在会话中完成扫描后，可以在您启动 Claude Code 的文件夹中一个带时间戳的 `CLAUDE-SECURITY-<timestamp>/` 目录中找到其结果：

* **`CLAUDE-SECURITY-RESULTS.md`**：报告，包含每个发现的 ID（例如 `F1`），以及其影响、利用场景、严重程度、置信度和建议
* **`CLAUDE-SECURITY-RESULTS.jsonl`**：以机器可读形式呈现的相同发现，每行一个 JSON 对象
* **`CLAUDE-SECURITY-RESULTS.sarif`**：以 [SARIF 2.1.0](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html) 日志形式呈现的相同发现，适用于 GitHub 代码扫描以及任何读取该标准的其他工具。扫描会按照 [CWE](https://cwe.mitre.org/) 弱点类别对发现进行分类
* **`CLAUDE-SECURITY-REVISION-<commit>.json`**：修订戳，记录扫描了哪个提交、使用的[扫描力度](#choose-a-scan-effort)、未提交的更改是否包含在扫描的目录树中，以及该次运行的验证彻底程度。在版本控制之外进行的扫描会用 `UNVERSIONED` 代替提交

该目录是扫描对您的检出所做的唯一更改，并且它带有自己的 `.gitignore`，因此误执行的 `git add` 不会将报告纳入提交。某些打包和部署工具不会读取该文件，因此在打包或部署时请将报告目录移出。若要将报告保留在历史记录中作为审计追踪，请删除该 `.gitignore` 文件并提交该目录。

只有在独立的验证 Agent 分析之后，发现才会出现在报告中，这使报告保持简短且值得阅读。扫描是非确定性的：对同一代码的两次扫描可能会发现不同的问题。请定期运行扫描。

## 修复发现的问题

从 `/claude-security` 菜单中选择 **Suggest patches** 即可启动修复流程，也可以用自然语言提出请求，例如"fix finding F3"，然后从报告中选择要处理的发现项。每个补丁都在您仓库的临时副本中起草。Agent 仍有可能误改您自己的文件，因此请在任务结束时运行 `git status`。

在交付之前，一个独立于编写该补丁的 Agent 会对其进行审查。它会在可行时运行您项目的测试，并自行阅读 diff，检查补丁可能引入的任何新问题。

该审查必须确认补丁：

* 解决了其对应的那一个发现项
* 未引入新的漏洞
* 除此之外不改变任何行为

当审查无法确认以上三点全部成立时，插件会写一份简短说明解释原因，而不是生成补丁。

### 基于较早报告的补丁

在有新提交之后，您仍然可以使用某次扫描的报告，但前提是被扫描的提交位于您当前的历史记录中。如果不在，请接受插件提供的重新扫描。

### 补丁永远不会被自动应用

是否应用补丁由您决定。补丁会写入报告的 `patches/` 文件夹，每个已修补的发现项对应一个 `F<n>.patch`，旁边附有一份解释更改内容的说明。您可以让 Claude 应用某个补丁并创建 Pull Request，也可以在仓库根目录下从您的 shell 中应用它。

请先阅读补丁的说明文件 `F<n>.md`，因为其中包含确切的命令，例如：

```bash theme={null}
git -C "$(git rev-parse --show-toplevel)" apply CLAUDE-SECURITY-<timestamp>/patches/F1.patch
```

请将每个补丁放在单独的 Pull Request 中应用，以便对其单独进行审查和测试。如果 `git apply` 拒绝应用某个补丁，请阅读其说明。

## 分诊您已有的发现

Vulnerability Triage 会检查列表中的发现在您的代码中是否成立。该列表可以是来自其他扫描器的 SARIF 2.1.0 文件，也可以是先前扫描生成的 `CLAUDE-SECURITY-<timestamp>/` 目录。请仅对来自您信任的来源的列表进行分诊。

如果您之后需要补丁，请确保除列表本身之外，没有任何未提交或未跟踪的内容。在发现的文件路径所相对的目录中启动 Claude Code，该目录通常是您仓库的根目录。

然后输入带有列表路径的命令：

```text theme={null}
/claude-security triage results.sarif
```

报告为 `FINDINGS-TRIAGE.md`，位于一个新的 `CLAUDE-SECURITY-<timestamp>/` 目录中。分诊否决的发现列在 **Not validated** 下。请亲自检查该部分，因为如果您的列表只用寥寥数语描述某个真实发现，分诊更有可能将其否决。报告旁边的 `FINDINGS-TRIAGE.sarif` 是添加了判定结果的您的列表。

要将报告中的有效发现转换为补丁，请在分诊结束时接受相应提议。如果您先进行了提交、拉取或切换分支，请重新开始分诊。有关补丁作业完成后需要检查的内容，请参阅[修复发现](#fix-findings)。

## 创建威胁模型

`/claude-security:threat-model` 命令会起草一份威胁模型，这是一个可以提供给后续扫描使用的文件。

在仓库根目录启动 Claude Code，然后输入：

```text theme={null}
/claude-security:threat-model
```

Claude 会读取您的代码和任何扫描报告，然后询问您的系统情况以及它还可以读取哪些内容：文件夹、您的[连接器](https://code.claude.com/docs/zh-CN/mcp.md)，或您粘贴的文本。

> [!WARNING]
>
> 如果您允许 Claude 读取您的连接器，它的搜索词会离开您的计算机，并且这些搜索词可能包含该报告中的字词。Claude 可能会被它在其中读取到的内容或您粘贴的文本误导。

当 Claude 显示草稿时：

1. 在选择 **Yes** 将其保存为 `.claude-security/threat-model.md` 之前，请先检查草稿，因为错误的陈述可能会使后续扫描漏掉某个漏洞。
2. 如果存在扫描报告，请选择是否让 Claude 在报告所在文件夹中写入 `CLAUDE-SECURITY-PRECONDITIONS.md`。该文件会说明在 Claude 读取的来源中，每项发现的前提条件是 `met`、`not met` 还是 `not shown`。
3. Claude 完成后，请先输入 [`/clear`](https://code.claude.com/docs/zh-CN/commands#all-commands)，再运行该插件的其他命令。

要确保扫描会读取威胁模型，请[在您的请求中指明它](#tell-a-scan-about-your-system)。

## 分析攻击面

攻击面分析会检查项目中攻击者可以触及的部分，并提出缩小该攻击面的更改建议。

在仓库的根目录中启动 Claude Code，在[可用](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode)的情况下使用[自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#switch-permission-modes)，然后输入：

```text theme={null}
/claude-security:attack-surface-analysis
```

分析可能需要很长时间，并消耗大量 token。在分析结束之前，请保持 Claude Code 处于打开状态，并且不要提交、切换分支或编辑文件。

分析不能取代用于查找漏洞的[扫描](#scan-and-fix-your-codebase)。分析会查找以下内容：

* 默认配置可以触及但没有任何默认功能使用的代码
* 在登录之前或无需任何用户操作即可触及的敏感代码
* 会悄无声息地关闭某项安全功能的选项组合

报告为启动 Claude Code 所在文件夹中的 `CLAUDE-SECURITY-<timestamp>/CLAUDE-SECURITY-RESULTS.md`。如果分析编写了补丁，其旁边的 `attack-surface-patches/PATCHES.md` 会对这些补丁进行说明。在您亲自检查之前，请将发现的问题和补丁视为未经测试。

## 从 shell 扫描

`claude-security` 命令会扫描您运行它时所在的文件夹，并将[相同的报告](#read-the-scan-results)写入该文件夹。扫描在[插件的沙箱](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/claude-security/README.md#running-in-a-sandbox)中运行。它使用您的 Claude Code 登录信息或您的提供商的环境变量进行登录。

沙箱仅在 Linux 上运行：

* **Linux**：需要 x86-64 或 arm64 上的内核 5.19 或更高版本、Python 3.10 或更高版本，以及非 root 用户。在 Ubuntu 24.04 或更高版本上，请添加 `--sudo`，它会使用 `sudo` 设置沙箱。如果您的系统不满足这些要求，请[在会话中扫描](#scan-and-fix-your-codebase)。
* **Windows**：添加 `--no-sandbox`，否则命令会停止
* **macOS 和 WSL 2**：不支持该命令，因此请[在会话中扫描](#scan-and-fix-your-codebase)

> [!WARNING]
>
> 该命令不会加载您仓库的 `.claude/` 设置或 hook，因此其中的拒绝规则不适用。在沙箱中，您组织的[服务器托管设置](https://code.claude.com/docs/zh-CN/server-managed-settings.md#security-considerations)也不适用。如果您添加 `--no-sandbox`，扫描将以您用户的访问权限在[自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode)下运行。请参阅插件的[信任模型](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/claude-security/SECURITY.md#trust-model)。

该命令包含在插件中。要将其添加到您的 `PATH`，请在 Claude Code 中输入 `/claude-security:install-cli`，并按照其输出的内容操作。

然后在您的仓库中运行它，如果您的系统需要，请加上 `--sudo` 或 `--no-sandbox`：

* **扫描代码库**：`claude-security`
* **扫描此分支的更改**：`claude-security --changes`
* **以全屏视图扫描**：`claude-security --tui`，需要 Node.js 18 或更高版本
* **列出其他选项**：`claude-security --help`

该命令在[自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode)下运行，如果自动模式不可用则会停止。如果您的 Claude Code 模型不支持自动模式，请添加 `--model` 并指定一个支持自动模式的模型。对于任何其他原因，请[在会话中扫描](#scan-and-fix-your-codebase)。

## 在 GitHub Actions 中扫描 Pull Request

在 Pull Request 上，插件的 GitHub Actions 作业会扫描变更、发布审查和建议的修复，并且可以阻止合并。它还可以按计划扫描整个仓库。

> [!WARNING]
>
> 仅在您信任所有拥有推送权限的人的仓库中使用此作业。任何向 Pull Request 分支添加文本或代码的人都可以操纵扫描、让其运行这些代码，并使其使用该作业的凭据发送模型请求。默认情况下，对于来自 fork 的 Pull Request，该作业会失败。
>
> 在公共仓库中，审查、作业摘要或作业的报告制品可能会泄露尚未修复的漏洞。若要将发现结果保密，请改为[在您的会话中扫描](#scan-only-your-changes)。

该作业需要 GitHub.com 上由 GitHub 托管的 Linux 运行器，以及 [Anthropic API 访问权限](https://platform.claude.com/docs/en/get-api-key)或[第三方提供商](https://code.claude.com/docs/zh-CN/third-party-integrations.md)，因为它无法使用付费计划登录。请按照插件的 [GitHub Actions 指南](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/claude-security/CI.md)添加工作流，并创建仅供此作业使用的凭据。若要升级到 0.13.0 版本，请在每个工作流中将其提交放在 `@` 之后：`d0d514288960e7a78e1a76e4a3ec694416458726`。

使用该指南中的示例工作流时，来自您仓库分支的草稿 Pull Request 在您将其标记为可供审查之前，会在不经扫描的情况下通过。来自 Dependabot 的 Pull Request 也会在不经扫描的情况下通过，因此请自行审查其变更。

## 该插件如何与其他安全工具配合

Claude Security 插件是纵深防御体系中的深度扫描层，与 [security guidance 插件](https://code.claude.com/docs/zh-CN/security-guidance.md)、[Code Review](https://code.claude.com/docs/zh-CN/code-review.md)、托管的 [Claude Security](https://claude.com/product/claude-security) 产品以及您现有的扫描工具协同工作：

| 阶段 | 工具 | 覆盖范围 |
| :- | :- | :- |
| 会话中 | [Security guidance 插件](https://code.claude.com/docs/zh-CN/security-guidance.md) | Claude 编写的代码中的常见漏洞，在同一会话中修复 |
| 按需深度扫描 | Claude Security 插件 | 对仓库或 diff 进行多代理扫描，提供经过独立审查的发现和补丁 |
| Pull Request 时 | [Code Review](https://code.claude.com/docs/zh-CN/code-review.md)，Team 和 Enterprise 计划 | 结合完整代码库上下文的多代理正确性与安全性审查 |
| Pull Request 时深度扫描 | [GitHub Actions 中的 Claude Security 插件](#scan-pull-requests-in-github-actions) | 对 Pull Request 的变更进行多代理扫描 |
| 托管 | [Claude Security](https://claude.com/product/claude-security)，Enterprise 计划 | 监控已连接仓库的托管扫描 |
| CI 中 | 您现有的静态分析和依赖扫描工具 | 特定语言的规则、供应链检查和策略执行 |

该插件不会取代您现有的源代码安全工具。请将其与静态分析、依赖扫描和代码审查一起运行：它会像人类安全研究人员一样对您的代码进行推理，从而补充这些工具提供的确定性检查。

如果您的组织使用托管的 Claude Security 产品，`/claude-security:hosted` 会在您的会话中显示其发现。使用之前，请阅读[要求](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/claude-security/README.md#findings-from-hosted-claude-security)和[风险](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/claude-security/SECURITY.md#findings-from-hosted-claude-security)。

## 故障排除

**`/claude-security` 菜单打开时显示 Python 警告。** 该插件需要 `PATH` 中有 3.9 或更高版本的 `python3`。如果完全找不到 `python3`，菜单会警告在安装 `python3` 之前 Claude Security 将无法工作；如果 `PATH` 中的第一个 `python3` 版本过旧，警告会指出找到的版本。请安装 Python 3，或将更新版本的 `python3` 放在 `PATH` 的前面，然后启动新会话。

**在 Fable 模型上扫描时，您可能会看到 "safeguards flagged this message" 通知。** 该消息会指出您正在运行的模型。Fable 的网络安全分类器会对某些请求发出警示，而 Claude Code 会通过[自动模型回退](https://code.claude.com/docs/zh-CN/model-config.md#automatic-model-fallback)在 Opus 模型上重新运行触发警示的请求。这是预期行为。请求重新运行后，扫描应仍能成功完成。如果扫描停止，请参阅[继续已停止的扫描](#continue-a-stopped-scan)。

## 相关资源

要深入了解此页面涉及的部分：

* [Security guidance 插件](https://code.claude.com/docs/zh-CN/security-guidance.md)：在同一会话中，在 Claude 编写代码时捕获问题
* [Code Review](https://code.claude.com/docs/zh-CN/code-review.md)：设置 PR 时间多代理审查
* [Claude Security](https://claude.com/product/claude-security)：监控连接存储库的托管服务
* [Claude Code 安全](https://code.claude.com/docs/zh-CN/security.md)：Claude Code 如何处理信任、权限和保护措施
* [安装和管理插件](https://code.claude.com/docs/zh-CN/plugins/install.md)：从官方市场查找和安装其他插件
