> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 在 VS Code 中使用 Claude Code

> 安装和配置 VS Code 的 Claude Code 扩展。获得 AI 编码协助，包括内联差异、@-提及、计划审查和快捷键。

<img src="https://mintcdn.com/claude-code/-YhHHmtSxwr7W8gy/images/vs-code-extension-interface.jpg?fit=max&auto=format&n=-YhHHmtSxwr7W8gy&q=85&s=300652d5678c63905e6b0ea9e50835f8" alt="VS Code 编辑器，右侧打开 Claude Code 扩展面板，显示与 Claude 的对话" width="2500" height="1155" data-path="images/vs-code-extension-interface.jpg" />

VS Code 扩展为 Claude Code 提供了原生图形界面，直接集成到您的 IDE 中。这是在 VS Code 中使用 Claude Code 的推荐方式。

使用该扩展，您可以在接受 Claude 的计划之前审查和编辑它们、在进行编辑时自动接受、@-提及具有特定行范围的文件、访问对话历史记录，以及在单独的选项卡或窗口中打开多个对话。

<h2 id="prerequisites">
  前置条件
</h2>

安装前，请确保您拥有：

* VS Code 1.94.0 或更高版本
* Anthropic 账户：任何付费 Claude 订阅（Pro、Max、Team 或 Enterprise）或 Claude Console 账户都可以使用，无需 API 密钥。首次打开扩展时，您将[使用此账户登录](/docs/zh-CN/authentication#log-in-to-claude-code)。如果您通过第三方提供商（如 Amazon Bedrock 或 Google Cloud 的 Agent Platform）访问 Claude，请参阅[使用第三方提供商](#use-third-party-providers)了解设置说明。

<Tip>
  该扩展包含其自己的 CLI（命令行界面）副本用于聊天面板。要在 VS Code 的集成终端中运行 `claude`，您还需要[独立 CLI 安装](/docs/zh-CN/setup)。有关详细信息，请参阅 [VS Code 扩展与 Claude Code CLI](#vs-code-extension-vs-claude-code-cli)。
</Tip>

<h2 id="install-the-extension">
  安装扩展
</h2>

点击您的 IDE 的链接以直接安装：

* [为 VS Code 安装](vscode:extension/anthropic.claude-code)
* [为 Cursor 安装](cursor:extension/anthropic.claude-code)

或在 VS Code 中，按 `Cmd+Shift+X`（Mac）或 `Ctrl+Shift+X`（Windows/Linux）打开扩展视图，搜索"Claude Code"，然后点击**安装**。

该扩展也可以安装在其他 VS Code 分支中，如 Devin Desktop 或 Kiro。在编辑器的扩展视图中搜索"Claude Code"，或从 [Open VSX 注册表](https://open-vsx.org/extension/Anthropic/claude-code) 安装。如果您的编辑器无法安装该扩展，请[安装 CLI](/docs/zh-CN/quickstart) 并在其集成终端中运行 `claude`。CLI 可在任何终端中使用。

<Note>如果安装后扩展没有出现，请重启 VS Code 或从命令面板运行"Developer: Reload Window"。</Note>

<h2 id="get-started">
  开始使用
</h2>

安装后，您可以通过 VS Code 界面开始使用 Claude Code：

<Steps>
  <Step title="打开 Claude Code 面板">
    在整个 VS Code 中，Spark 图标表示 Claude Code：<img src="https://mintcdn.com/claude-code/c5r9_6tjPMzFdDDT/images/vs-code-spark-icon.svg?fit=max&auto=format&n=c5r9_6tjPMzFdDDT&q=85&s=3ca45e00deadec8c8f4b4f807da94505" alt="Spark icon" style={{display: "inline", height: "0.85em", verticalAlign: "middle"}} width="16" height="16" data-path="images/vs-code-spark-icon.svg" />

    打开 Claude 的最快方法是点击编辑器右上角**编辑器工具栏**中的 Spark 图标。只有当您打开了文件时，该图标才会出现。

    <img src="https://mintcdn.com/claude-code/mfM-EyoZGnQv8JTc/images/vs-code-editor-icon.png?fit=max&auto=format&n=mfM-EyoZGnQv8JTc&q=85&s=eb4540325d94664c51776dbbfec4cf02" alt="VS Code 编辑器显示编辑器工具栏中的 Spark 图标" width="2796" height="734" data-path="images/vs-code-editor-icon.png" />

    打开 Claude Code 的其他方式：

    * **活动栏**：点击左侧边栏中的 Spark 图标以打开会话列表。点击任何会话以将其作为完整编辑器选项卡打开，或开始新的会话。此图标在活动栏中始终可见。
    * **命令面板**：`Cmd+Shift+P`（Mac）或 `Ctrl+Shift+P`（Windows/Linux），输入"Claude Code"，然后选择一个选项，如"在新选项卡中打开"
    * **状态栏**：如果您已将 [`preferredLocation`](#extension-settings) 设置为 `sidebar`，或使用**Claude Code: Open in Side Bar** 打开了 Claude，请点击窗口右下角的 **✱ Claude Code**。即使没有打开文件，这也有效。

    您可以拖动 Claude 面板以在 VS Code 中的任何位置重新定位它。有关详细信息，请参阅[自定义您的工作流](#customize-your-workflow)。
  </Step>

  <Step title="登录">
    第一次打开面板时，会出现登录屏幕。点击**登录**并在浏览器中完成授权。

    如果您稍后看到**未登录 · 请运行 /login**，扩展程序会自动重新打开登录屏幕。如果它没有出现，请从命令面板使用**开发者：重新加载窗口**重新加载窗口。

    如果您在 shell 中设置了 `ANTHROPIC_API_KEY` 但仍然看到登录提示，VS Code 可能没有继承您的 shell 环境。从终端使用 `code .` 启动 VS Code，以便它继承您的环境变量，或改为使用您的 Claude 账户登录。

    登录后，会出现**学习 Claude Code** 检查清单。通过点击**显示给我**来完成每一项，或使用 X 关闭它。要稍后重新打开它，请在 VS Code 设置中的扩展程序 → Claude Code 下取消选中**隐藏入门**。
  </Step>

  <Step title="发送提示">
    要求 Claude 帮助您处理代码或文件，无论是解释某些内容的工作原理、调试问题还是进行更改。

    <Tip>Claude 会自动看到您选择的文本。按 `Option+K`（Mac）/ `Alt+K`（Windows/Linux）也可以在您的提示中插入 @-mention 引用（如 `@file.ts#5-10`）。</Tip>

    以下是询问文件中特定行的示例：

    <img src="https://mintcdn.com/claude-code/FVYz38sRY-VuoGHA/images/vs-code-send-prompt.png?fit=max&auto=format&n=FVYz38sRY-VuoGHA&q=85&s=ede3ed8d8d5f940e01c5de636d009cfd" alt="VS Code 编辑器，在 Python 文件中选择了第 2-3 行，Claude Code 面板显示关于这些行的问题，带有 @-mention 引用" width="3288" height="1876" data-path="images/vs-code-send-prompt.png" />
  </Step>

  <Step title="审查更改">
    您看到的内容取决于提示框底部显示的[权限模式](/docs/zh-CN/permission-modes#which-mode-a-session-starts-in)：

    * 在自动或自动编辑模式下，Claude 在不询问的情况下编辑工作区中的大多数文件。
    * 在手动模式下，当 Claude 想要编辑文件时，它会显示原始内容和建议更改的并排比较，然后要求权限。您可以接受、拒绝或告诉 Claude 改为做什么。如果您在接受之前直接在差异视图中编辑建议的内容，Claude 会被告知您修改了它，因此它不会假设文件与其原始建议相匹配。

          <img src="https://mintcdn.com/claude-code/FVYz38sRY-VuoGHA/images/vs-code-edits.png?fit=max&auto=format&n=FVYz38sRY-VuoGHA&q=85&s=e005f9b41c541c5c7c59c082f7c4841c" alt="VS Code 显示 Claude 建议更改的差异，以及询问是否进行编辑的权限提示" width="3292" height="1876" data-path="images/vs-code-edits.png" />
  </Step>
</Steps>

有关您可以使用 Claude Code 做什么的更多想法，请参阅[常见工作流](/docs/zh-CN/common-workflows)。

<Tip>
  从命令面板运行"Claude Code: Open Walkthrough"以获得基础知识的引导式教程。
</Tip>

<h2 id="use-the-prompt-box">
  使用提示框
</h2>

提示框支持多项功能：

* **权限模式**：点击提示框底部的模式指示器来切换权限模式。在 Pro、Max 和 Team 计划上，Auto 是内置的起始权限模式。请参阅[扩展程序如何选择起始权限模式](/docs/zh-CN/permission-modes#switch-permission-modes)了解会改变这一点的因素，以及指示器提供的每种权限模式。
  * **Auto**：分类器审查大多数操作，而不是询问您。请参阅 [auto 模式](/docs/zh-CN/permission-modes#eliminate-prompts-with-auto-mode)了解它审查和阻止的内容。
  * **Manual**：Claude 在文件编辑和大多数 shell 命令之前请求权限。
  * **Plan**：Claude 描述它将做什么，并在进行更改之前等待批准。VS Code 自动将计划作为完整的 Markdown 文档打开，您可以在其中添加内联注释以在 Claude 开始之前提供反馈。
  * **Edit automatically**：Claude 进行编辑而不询问。
* **Model**：从命令菜单中选择 **Switch model…** 以在会话中途更改模型。您也可以点击提示框底部的模型名称来打开相同的选择器。当当前模型支持[工作量级别](/docs/zh-CN/model-config#adjust-effort-level)时，选择器还会显示 **Effort** 行。模型名称按钮和 **Effort** 行需要 Claude Code v2.1.257 或更高版本。
* **Command menu**：点击 `/` 或输入 `/` 来打开命令菜单。选项包括附加文件、切换模型和切换扩展思考。Customize 部分提供对 MCP 服务器、slash commands、输出样式、hooks、memory、权限和插件的访问。带有终端图标的项目在集成终端中打开。
  * 要浏览 `/usage` 或 [`/remote-control`](/docs/zh-CN/remote-control) 等命令，请在 Customize 部分中选择 **Slash commands**。对话框会列出它们并带有过滤框。选择一个来运行它。在提示框中输入 `/` 仍会内联建议命令。需要 Claude Code v2.1.257 或更高版本。
  * 在 Customize 部分中选择 **Output styles** 来选择[输出样式](/docs/zh-CN/output-styles)，包括您的自定义样式。需要 Claude Code v2.1.257 或更高版本。

    要创建自定义样式，请从 **Output styles** 菜单中选择 **Build a custom style**。Claude Code 会在项目或用户级别为您编写[样式文件](/docs/zh-CN/output-styles#create-a-custom-output-style)。需要 Claude Code v2.1.261 或更高版本。
  * Settings 部分包括 **Enable Remote Control for all sessions**，它设置 [`remoteControlAtStartup`](/docs/zh-CN/settings-reference#remotecontrolatstartup) 来控制[新的交互式会话是否自动连接到 Remote Control](/docs/zh-CN/remote-control#enable-remote-control-for-all-sessions)。需要 Claude Code v2.1.203 或更高版本。

    当您在 VS Code 窗口中打开或关闭切换开关时，更改适用于该 VS Code 窗口中已打开的会话，而不仅仅是您之后启动的会话。如果您关闭它，打开的会话将断开连接。使用 Claude Code v2.1.261 或更高版本，更改也会到达您其他 VS Code 窗口中打开的会话。
  * Settings 部分还包括 **Focus view**，它隐藏工具调用、工具结果和思考在可展开的行后面，只留下您的提示和 Claude 的响应。Claude 的最新待办事项列表保持可见，Claude 提出的待处理问题的文本也保持可见；这需要 Claude Code v2.1.225 或更高版本。在那里切换它，使用 `Ctrl+Option+F`（Mac）/ `Ctrl+Alt+F`（Windows/Linux），或从命令面板使用 **Claude Code: Toggle Focus view**。更改适用于每个打开的会话并在会话之间持续。需要 Claude Code v2.1.221 或更高版本。
  * 要报告错误，请点击菜单底部的 **Report a problem**，或输入 `/bug` 或 `/feedback` 以及可选的描述来预填充报告。当您提交报告并且您在第一方连接上登录到 Anthropic 时，Claude Code 会将其发送给 Anthropic。在第三方提供商上，或没有 Anthropic 凭证的情况下，对话框仍会打开，但提交会显示错误并不发送任何内容：与 CLI 的 `/bug` 不同，扩展程序不会写入本地存档。需要 Claude Code v2.1.229 或更高版本。
* **Side questions**：输入 `/btw` 后跟一个问题来提问您的会话[而不添加到对话](/docs/zh-CN/interactive-mode#side-questions-with-%2Fbtw)。答案在聊天旁边的面板中打开，您可以在其中提出后续问题。线程在窗口重新加载后仍然存在。Claude Code 保留最新的 20 个交换，并根据 [`cleanupPeriodDays`](/docs/zh-CN/settings-reference#cleanupperioddays) 计划过期存储的线程，只要 Claude Code 可以[安全地确定保留期](/docs/zh-CN/claude-directory#cleaned-up-automatically)。要清除线程，请点击面板中的垃圾箱图标。需要 Claude Code v2.1.227 或更高版本。
* **Context indicator**：提示框显示您使用了多少 Claude 的上下文窗口。Claude 在需要时自动压缩，或者您可以手动运行 `/compact`。
* **Extended thinking**：让 Claude 花更多时间推理复杂问题。通过命令菜单（`/`）打开它。Claude 的推理在对话中显示为折叠块：点击一个块来阅读它，或按 `Ctrl+O` 来展开或折叠会话中的每个思考块。有关详细信息，请参阅[Extended thinking](/docs/zh-CN/model-config#extended-thinking)。
* **Multi-line input**：按 `Shift+Enter` 添加新行而不发送。这也适用于问题对话框的"Other"自由文本输入。

<h3 id="reference-files-and-folders">
  参考文件和文件夹
</h3>

使用 @-mentions 为 Claude 提供有关特定文件或文件夹的上下文。当您输入 `@` 后跟文件或文件夹名称时，Claude 会读取该内容，可以回答有关它的问题或对其进行更改。Claude Code 支持模糊匹配，因此您可以输入部分名称来找到您需要的内容：

```text wrap theme={null}
Explain the logic in @auth (fuzzy matches auth.js, AuthService.ts, etc.)
What's in @src/components/ (include a trailing slash for folders)
```

对于大型 PDF，您可以要求 Claude 读取特定页面而不是整个文件：单个页面、范围如第 1-10 页，或开放式范围如第 3 页及以后。

当您在编辑器中选择文本时，Claude 可以自动看到您突出显示的代码。提示框页脚显示选择了多少行。按 `Option+K`（Mac）/ `Alt+K`（Windows/Linux）来插入带有文件路径和行号的 @-mention（例如 `@app.ts#5-10`）。点击选择指示器来切换 Claude 是否可以看到您突出显示的文本 - 眼睛斜线图标表示选择对 Claude 隐藏。

您也可以在将文件拖入提示框时按住 `Shift` 来将它们添加为附件。点击任何附件上的 X 来从上下文中删除它。

<h3 id="resume-past-conversations">
  恢复过去的对话
</h3>

点击 Claude Code 面板顶部的 **Session history** 按钮来访问您的对话历史。您可以按关键字搜索或按时间浏览。点击任何对话来恢复它，包含完整的消息历史。有关恢复会话的更多信息，请参阅[管理会话](/docs/zh-CN/sessions)。

* **Session titles**：新会话根据您的第一条消息接收 AI 生成的标题。
* **Rename and archive**：将鼠标悬停在会话上以显示这些操作。重命名以给它一个描述性标题，或存档以将其移动到列表底部的 **Archived sessions** 组。

默认情况下，14 天内没有活动的会话会自动移动到 **Archived sessions**，除非它是打开的、未读的或在[组](#organize-sessions-into-groups)中。自动存档需要 Claude Code v2.1.265 或更高版本。要更改期间或关闭它，请打开[存档非活动会话设置](vscode://settings/claudeCode.archiveInactiveSessions)并选择天数或 **Never**。

要恢复存档的会话，请展开 **Archived sessions** 并点击 **Unarchive session**。在 v2.1.257 之前，该操作是 **Delete session**，它隐藏了一个会话而无法恢复。您之前删除的会话在升级后会出现在 **Archived sessions** 下。

当您恢复的对话以计划模式结束时，Claude Code 会恢复计划模式。需要 Claude Code v2.1.246 或更高版本。Claude Code 在两种情况下不会恢复它：

* 扩展程序从 `claudeCode.initialPermissionMode` 或从较早对话中继承的选择[选择起始权限模式](/docs/zh-CN/permission-modes#switch-permission-modes)
* 您配置了 `claudeCode.claudeProcessWrapper`

<h3 id="resume-cloud-sessions-from-claude-ai">
  从 Claude.ai 恢复云会话
</h3>

如果您使用[网络上的 Claude Code](/docs/zh-CN/claude-code-on-the-web)，您可以直接在 VS Code 中恢复这些云会话。这需要使用 **Claude.ai Subscription** 登录，而不是 Anthropic Console。

<Steps>
  <Step title="打开会话历史">
    点击 Claude Code 面板顶部的 **Session history** 按钮。
  </Step>

  <Step title="选择 Web 选项卡">
    对话框显示两个选项卡：Local 和 Web。点击 **Web** 来查看来自 claude.ai 的会话。
  </Step>

  <Step title="选择要恢复的会话">
    浏览或搜索您的云会话。点击任何会话来下载它并在本地继续对话。
  </Step>
</Steps>

<Note>
  只有使用 GitHub 存储库启动的网络会话才会出现在 Web 选项卡中。恢复会在本地加载对话历史；更改不会同步回 claude.ai。
</Note>

<h3 id="check-account-and-usage">
  检查账户和使用情况
</h3>

运行 `/usage` 来打开 Account & usage 对话框。对话框需要 claude.ai 登录，因此在[第三方提供商](#use-third-party-providers)上不提供。它显示您登录的账户、您的计划以及您计划限制的使用条形图，例如当前会话和周。每个条形图显示距离其限制重置还有多长时间。

对话框还分解了对您的计划限制有贡献的内容。它标记占最近使用量 10% 或更多的行为，例如缓存未命中、长上下文和子代理密集或高度并行的会话，每个都有减少它的提示。Attribution 表显示了每个 skill、subagent、plugin 和 MCP 服务器贡献了多少使用量。需要 Claude Code v2.1.174 或更高版本。

使用 Day 和 Week 切换来在过去 24 小时和过去 7 天之间切换。这些数字是近似的，并从此机器上的本地会话计算，因此不包括来自其他设备或 claude.ai 的使用情况。有关跟踪和减少使用情况的更多信息，请参阅[跟踪您的成本](/docs/zh-CN/costs#track-your-costs)。

<h2 id="customize-your-workflow">
  自定义您的工作流
</h2>

您可以重新定位 Claude 面板、运行多个对话、将会话列表组织成组，或切换到终端模式。

<h3 id="choose-where-claude-lives">
  选择 Claude 的位置
</h3>

您可以拖动 Claude 面板在 VS Code 中重新定位它。抓住面板的选项卡或标题栏并将其拖动到：

* **次级侧边栏**：窗口的右侧。在您编码时保持 Claude 可见。
* **主侧边栏**：左侧边栏，带有 Explorer、Search 等图标。
* **编辑器区域**：将 Claude 作为选项卡打开，与您的文件并排显示。适用于辅助任务。

<Tip>
  将侧边栏用于您的主要 Claude 会话，并为辅助任务打开其他选项卡。Claude 会记住您首选的位置。Activity Bar 会话列表图标与 Claude 面板分开：会话列表始终在 Activity Bar 中可见，而 Claude 面板图标仅在面板停靠到左侧边栏时才出现在那里。
</Tip>

运行 **Developer: Reload Window** 或重启 VS Code 后，聊天是否会返回其对话取决于它在哪里打开：

* **编辑器选项卡**：对话会随其选项卡返回。
* **侧边栏**：如果您在过去 10 分钟内发送了消息或 Claude 在其中做出了响应，对话会返回。如果它没有返回，请从 [Session history](#resume-past-conversations) 恢复对话。

<h3 id="run-multiple-conversations">
  运行多个对话
</h3>

使用命令面板中的 **Open in New Tab** 或 **Open in New Window** 来启动其他对话。每个对话维护其自己的历史记录和上下文，允许您并行处理不同的任务。

使用选项卡时，spark 图标上的小彩色点表示状态：蓝色表示权限请求待处理，橙色表示 Claude 在选项卡隐藏时完成。

<h3 id="organize-sessions-into-groups">
  将会话组织成组
</h3>

在 Activity Bar 的会话列表中，您可以将相关会话收集到命名的、可折叠的组中。需要 Claude Code v2.1.229 或更高版本。

* **对会话进行分组或取消分组**：右键单击会话以从其创建组、将其移动到现有组或将其从其组中删除。每个会话一次只属于一个组，因此将其移动到另一个组会将其从第一个组中删除。
* **一次移动多个会话**：`Cmd`-单击（Mac）/ `Ctrl`-单击（Windows/Linux）每个会话，或 `Shift`-单击以选择范围，然后右键单击选择。
* **从其选项卡对会话进行分组**：从命令面板运行 **Claude Code: Add Session Tab to Group**，或右键单击会话的编辑器选项卡，然后选择或创建组。需要 Claude Code v2.1.257 或更高版本。
* **重命名或删除组**：右键单击组标题。删除组仅删除组，其会话返回到未分组列表。

该扩展按工作区文件夹保存组，因此它们在窗口重新加载后仍然存在，并在您打开相同文件夹的每个窗口中出现。当您搜索列表时，该扩展在所有组中的一个平面列表中显示匹配项。

<h3 id="switch-to-terminal-mode">
  切换到终端模式
</h3>

默认情况下，该扩展打开图形聊天面板。如果您更喜欢 CLI 风格的界面，请打开 [Use Terminal setting](vscode://settings/claudeCode.useTerminal) 并勾选该框。

您也可以打开 VS Code 设置（Mac 上为 `Cmd+,` 或 Windows/Linux 上为 `Ctrl+,`），转到 Extensions → Claude Code，并勾选 **Use Terminal**。

<h2 id="manage-plugins">
  管理插件
</h2>

VS Code 扩展包含一个图形界面，用于安装和管理 [plugins](/docs/zh-CN/plugins)。在提示框中输入 `/plugins` 以打开**管理插件**界面。

<h3 id="install-plugins">
  安装插件
</h3>

插件对话框显示两个选项卡：**Plugins** 和 **Marketplaces**。

在 Plugins 选项卡中：

* **已安装的插件**显示在顶部，带有切换开关以启用或禁用它们
* **可用插件**来自您配置的市场，显示在下方
* 搜索以按名称或描述过滤插件
* 点击任何可用插件上的**安装**

安装插件时，选择安装范围：

* **为您安装**：在您的所有项目中可用（用户范围）
* **为此项目安装**：与项目协作者共享（项目范围）
* **本地安装**：仅供您使用，仅在此存储库中（本地范围）

<h3 id="manage-marketplaces">
  管理市场
</h3>

切换到 **Marketplaces** 选项卡以添加或删除插件源：

* 输入 GitHub 仓库、URL 或本地路径以添加新市场
* 点击刷新图标以更新市场的插件列表
* 点击垃圾桶图标以删除市场

进行更改后，横幅会提示您重启 Claude Code 以应用更改。

<Note>
  VS Code 中的插件管理在底层使用相同的 CLI 命令。您在扩展中配置的插件和市场也可在 CLI 中使用，反之亦然。
</Note>

有关插件系统的更多信息，请参阅 [Plugins](/docs/zh-CN/plugins) 和 [Plugin marketplaces](/docs/zh-CN/plugin-marketplaces)。

<h2 id="automate-browser-tasks-with-chrome">
  使用 Chrome 自动化浏览器任务
</h2>

将 Claude 连接到您的 Chrome 浏览器，以测试 Web 应用、使用控制台日志进行调试，以及在不离开 VS Code 的情况下自动化浏览器工作流。这需要 [Claude in Chrome 扩展](https://chromewebstore.google.com/detail/claude/fcoeoabgfenejglbffodgkkbkcdhcgfn) 版本 1.0.36 或更高版本。

在提示框中输入 `@browser`，然后输入您希望 Claude 执行的操作：

```text wrap theme={null}
@browser go to localhost:3000 and check the console for errors
```

您也可以打开附件菜单来选择特定的浏览器工具，例如打开新标签页或读取页面内容。

Claude 为浏览器任务打开新标签页并共享您浏览器的登录状态，因此它可以访问您已登录的任何网站。

有关设置说明、完整的功能列表和故障排除，请参阅 [在 Chrome 中使用 Claude Code](/docs/zh-CN/chrome)。

<h2 id="vs-code-commands-and-shortcuts">
  VS Code 命令和快捷键
</h2>

打开命令面板（Mac 上按 `Cmd+Shift+P` 或 Windows/Linux 上按 `Ctrl+Shift+P`），然后输入"Claude Code"以查看 Claude Code 扩展的所有可用 VS Code 命令。

某些快捷键取决于哪个面板处于"焦点"状态（接收键盘输入）。当光标在代码文件中时，编辑器处于焦点状态。当光标在 Claude 的提示框中时，Claude 处于焦点状态。使用 `Cmd+Esc` / `Ctrl+Esc` 在它们之间切换。

<Note>
  这些是用于控制扩展的 VS Code 命令。并非所有内置 Claude Code 命令都在扩展中可用。有关详细信息，请参阅 [VS Code 扩展与 Claude Code CLI](#vs-code-extension-vs-claude-code-cli)。
</Note>

| 命令                         | 快捷键                                                      | 描述                                                                                                                  |
| -------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Focus Input                | `Cmd+Esc` (Mac) / `Ctrl+Esc` (Windows/Linux)             | 在编辑器和 Claude 之间切换焦点                                                                                                 |
| Open in Side Bar           | -                                                        | 在侧边栏中打开 Claude                                                                                                      |
| Open in Terminal           | -                                                        | 在终端模式下打开 Claude                                                                                                     |
| Open in New Tab            | `Cmd+Shift+Esc` (Mac) / `Ctrl+Shift+Esc` (Windows/Linux) | 以编辑器选项卡形式打开新对话                                                                                                      |
| Open in New Window         | -                                                        | 在单独的窗口中打开新对话                                                                                                        |
| New Conversation           | `Cmd+N` (Mac) / `Ctrl+N` (Windows/Linux)                 | 开始新对话。需要 Claude 处于焦点状态且 `enableNewConversationShortcut` 设置为 `true`                                                  |
| Reopen Closed Session      | `Cmd+Shift+T` (Mac) / `Ctrl+Shift+T` (Windows/Linux)     | 重新打开最近关闭的 Claude 会话选项卡。当最后关闭的选项卡不是 Claude 会话时，会回退到 VS Code 的正常重新打开关闭编辑器功能。使用 `enableReopenClosedSessionShortcut` 禁用 |
| Insert @-Mention Reference | `Option+K` (Mac) / `Alt+K` (Windows/Linux)               | 插入对当前文件和选择的引用（需要编辑器处于焦点状态）                                                                                          |
| Toggle Focus view          | `Ctrl+Option+F` (Mac) / `Ctrl+Alt+F` (Windows/Linux)     | 隐藏或显示对话中的工具活动。在 Claude 面板或侧边栏可见时有效。需要 Claude Code v2.1.221 或更高版本                                                    |
| Rename Session Tab         | -                                                        | 重命名活动 Claude 选项卡中的会话。该命令也出现在选项卡的右键菜单中。需要 Claude Code v2.1.257 或更高版本                                                 |
| Add Session Tab to Group   | -                                                        | 将活动 Claude 选项卡中的会话添加到您选择或创建的[会话组](#organize-sessions-into-groups)。该命令也出现在选项卡的右键菜单中。需要 Claude Code v2.1.257 或更高版本    |
| Mark Session as Unread     | -                                                        | 在会话列表中将活动 Claude 选项卡中的会话标记为未读。该命令也出现在选项卡的右键菜单中。需要 Claude Code v2.1.257 或更高版本                                        |
| Show Logs                  | -                                                        | 查看扩展调试日志                                                                                                            |
| Logout                     | -                                                        | 登出您的 Anthropic 账户                                                                                                   |

<h3 id="launch-a-vs-code-tab-from-other-tools">
  从其他工具启动 VS Code 选项卡
</h3>

该扩展在 `vscode://anthropic.claude-code/open` 处注册了一个 URI 处理程序。使用它从您自己的工具（shell 别名、浏览器书签或任何可以打开 URL 的脚本）打开新的 Claude Code 选项卡。如果 VS Code 尚未运行，打开 URL 会先启动它。如果 VS Code 已在运行，URL 会在当前焦点的窗口中打开。

使用您的操作系统的 URL 打开程序调用处理程序。

<Tabs>
  <Tab title="macOS">
    ```bash theme={null}
    open "vscode://anthropic.claude-code/open"
    ```
  </Tab>

  <Tab title="Linux">
    ```bash theme={null}
    xdg-open "vscode://anthropic.claude-code/open"
    ```

    `xdg-open` 命令来自 `xdg-utils` 包。如果 shell 报告找不到它，请参阅 [xdg-open is not found on Linux](/docs/zh-CN/deep-links#xdg-open-is-not-found-on-linux)。
  </Tab>

  <Tab title="Windows">
    在 PowerShell 中：

    ```powershell theme={null}
    Start-Process "vscode://anthropic.claude-code/open"
    ```

    在 `cmd.exe` 中，`start` 将其第一个带引号的参数视为窗口标题，因此在 URL 之前传递一个空标题：

    ```cmd theme={null}
    start "" "vscode://anthropic.claude-code/open"
    ```
  </Tab>
</Tabs>

处理程序接受两个可选查询参数：

| 参数        | 描述                                                                                                                                                       |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `prompt`  | 在提示框中预填充的文本。必须进行 URL 编码。提示框会被预填充但不会自动提交。                                                                                                                 |
| `session` | 要恢复的会话 ID，而不是开始新对话。该会话必须属于 VS Code 中当前打开的工作区。如果找不到该会话，则改为开始新对话。如果该会话已在选项卡中打开，则该选项卡会获得焦点。要以编程方式捕获会话 ID，请参阅[继续对话](/docs/zh-CN/headless#continue-conversations)。 |

例如，要打开一个预填充了"review my changes"的选项卡：

```text theme={null}
vscode://anthropic.claude-code/open?prompt=review%20my%20changes
```

要启动终端会话而不是 VS Code 选项卡，请使用 CLI 的 `claude-cli://` 处理程序。请参阅[从链接启动会话](/docs/zh-CN/deep-links)。

<h2 id="configure-settings">
  配置设置
</h2>

该扩展有两种类型的设置：

* **VS Code 中的扩展设置**：控制扩展在 VS Code 中的行为。使用 `Cmd+,`（Mac）或 `Ctrl+,`（Windows/Linux）打开，然后转到扩展 → Claude Code。您也可以输入 `/` 并选择 **General Config** 来打开设置。
* **`~/.claude/settings.json` 中的 Claude Code 设置**：在扩展和 CLI 之间共享。用于允许的命令、环境变量、hooks 和 MCP 服务器。在 Pro、Max 和 Team 计划上，它也是权限模式对话开始时的一个输入。[切换权限模式](/docs/zh-CN/permission-modes#switch-permission-modes)列出了顺序。有关详细信息，请参阅[设置](/docs/zh-CN/settings)。

<Tip>
  将 `"$schema": "https://json.schemastore.org/claude-code-settings.json"` 添加到您的 `settings.json` 中，以在 VS Code 中直接获得所有可用设置的自动完成和内联验证。
</Tip>

<h3 id="extension-settings">
  扩展设置
</h3>

VS Code 从您的用户设置中读取 `initialPermissionMode`，并忽略工作区值。在 v2.1.225 之前，VS Code 将该设置默认为 `default` 并应用工作区值。

| 设置                                  | 默认值     | 描述                                                                                                                                                                                                                                                                                                                                                                           |
| ----------------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `useTerminal`                       | `false` | 在终端模式而不是图形面板中启动 Claude                                                                                                                                                                                                                                                                                                                                                       |
| `initialPermissionMode`             | -       | 控制新对话的批准提示：`default`、`plan`、`acceptEdits` 或 `bypassPermissions`。`manual` 是 `default` 的别名，选择模式指示器中标记为 **Manual** 的模式。当您将其留空时，扩展会选择起始权限模式，如[切换权限模式](/docs/zh-CN/permission-modes#switch-permission-modes)中所述。                                                                                                                                                                       |
| `preferredLocation`                 | `panel` | Claude 打开的位置：`sidebar`（右侧）或 `panel`（新标签页）                                                                                                                                                                                                                                                                                                                                    |
| `autosave`                          | `true`  | Claude 读取或写入文件前自动保存文件                                                                                                                                                                                                                                                                                                                                                        |
| `useCtrlEnterToSend`                | `false` | 使用 Ctrl/Cmd+Enter 而不是 Enter 来发送提示                                                                                                                                                                                                                                                                                                                                            |
| `enableNewConversationShortcut`     | `false` | 启用 Cmd/Ctrl+N 来开始新对话                                                                                                                                                                                                                                                                                                                                                         |
| `enableReopenClosedSessionShortcut` | `true`  | 使用 Cmd/Ctrl+Shift+T 重新打开最近关闭的 Claude 会话标签页。当最后关闭的标签页不是 Claude 会话时，快捷键会运行 VS Code 的正常重新打开关闭编辑器命令。                                                                                                                                                                                                                                                                             |
| `archiveInactiveSessions`           | `14`    | 在无活动的这么多天后[自动存档会话](#resume-past-conversations)：`1`、`2`、`7` 或 `14`。设置为 `0` 以关闭。需要 Claude Code v2.1.265 或更高版本                                                                                                                                                                                                                                                                  |
| `hideOnboarding`                    | `false` | 隐藏入门清单（毕业帽图标）                                                                                                                                                                                                                                                                                                                                                                |
| `focusView`                         | `false` | 将工具调用、工具结果和思考隐藏在可展开的行后面，只留下您的提示和 Claude 的响应。Claude 的最新待办事项列表保持可见；这需要 Claude Code v2.1.225 或更高版本。您也可以从命令菜单切换焦点视图。需要 Claude Code v2.1.221 或更高版本                                                                                                                                                                                                                                |
| `respectGitIgnore`                  | `true`  | 从文件搜索中排除 .gitignore 模式                                                                                                                                                                                                                                                                                                                                                       |
| `usePythonEnvironment`              | `true`  | 运行 Claude 时激活工作区的 Python 环境。需要 Python 扩展。                                                                                                                                                                                                                                                                                                                                    |
| `environmentVariables`              | `[]`    | 为 Claude 进程设置环境变量。对于共享配置，请改用 Claude Code 设置。                                                                                                                                                                                                                                                                                                                                 |
| `disableLoginPrompt`                | `false` | 跳过身份验证提示（用于第三方提供商设置）                                                                                                                                                                                                                                                                                                                                                         |
| `allowDangerouslySkipPermissions`   | `false` | 在模式选择器中添加绕过权限。仅在没有互联网访问的沙箱中使用。                                                                                                                                                                                                                                                                                                                                               |
| `claudeProcessWrapper`              | -       | 用于启动 Claude 进程的可执行文件。当存在时，捆绑的二进制路径作为参数传递。如果扩展构建不包含您的平台的二进制文件，请将其设置为单独安装的 `claude` 二进制文件。在包装的设置中，对话以手动模式开始，除非您设置了 `initialPermissionMode` 或在之前的对话中选择了手动、自动编辑或自动，因为扩展会跳过那里的设置和内置默认步骤；请参阅[切换权限模式](/docs/zh-CN/permission-modes#switch-permission-modes)。激活时出现"不支持的平台"错误意味着您的平台没有捆绑的二进制文件；请参阅[哪些平台有预构建的二进制文件](/docs/zh-CN/troubleshoot-install#native-binary-not-found-after-npm-install)。 |

<h2 id="use-a-screen-reader">
  使用屏幕阅读器
</h2>

该扩展的聊天面板可与屏幕阅读器配合使用。您无需打开任何设置：该扩展会为每个用户宣布对话活动，无需进行任何视觉更改。这与 CLI 的可选 [屏幕阅读器模式](/docs/zh-CN/accessibility) 不同，后者会调整终端界面。

聊天面板中的屏幕阅读器支持需要 Claude Code v2.1.236 或更高版本。

在对话期间，该扩展会宣布：

* **Claude 的回复**：该扩展在每条回复完成时宣布一次，在文本流入时保持沉默。您的屏幕阅读器将代码块读作行数摘要，按标签读取链接，逐个单元格读取表格；完整回复在记录中保持可读。
* **权限请求和问题**：当权限提示出现时，该扩展会宣布请求，并命名 Claude 想要使用的工具。当 Claude 向您提问以及当 Claude 完成计划并等待您审查时，它以相同方式宣布。
* **状态更改**：当 Claude 开始工作、Claude 准备好接收您的输入以及 Claude Code 开始压缩对话时，该扩展会宣布。
* **错误和模型提示**：该扩展宣布对话中的错误，并在 [使用额度同意提示](/docs/zh-CN/model-config#fable-and-usage-credits) 或 [标记请求提示](/docs/zh-CN/model-config#ask-before-switching) 出现时宣布。

记录中的每个回合都以视觉隐藏的标题开头，标题标记为启动该回合的提示，因此您可以使用屏幕阅读器的标题导航在回合之间跳转。您也可以使用 `Tab` 将焦点移动到记录本身，因为该扩展将其公开为标记的区域，并按您自己的速度读取。当 Claude 工作时，您的屏幕阅读器会读取一个文本标签来代替进度旋转器的动画。

当您重新打开会话或切换到另一个会话时，该扩展不会宣布任何内容：恢复的历史记录、待处理的权限提示和进行中的状态保持沉默，直到发生新的事情。

<h2 id="vs-code-extension-vs-claude-code-cli">
  VS Code extension vs. Claude Code CLI
</h2>

Claude Code 既可作为 VS Code extension（图形面板）使用，也可作为 CLI（终端中的命令行界面）使用。某些功能仅在 CLI 中可用。如果您需要仅限 CLI 的功能，请在 VS Code 的集成终端中运行 `claude`。这需要[独立 CLI 安装](/docs/zh-CN/setup)：extension 不会将 `claude` 添加到您的 PATH。请参阅[在 VS Code 中运行 CLI](#run-cli-in-vs-code)。

| 功能                  | CLI                   | VS Code Extension                                                  |
| ------------------- | --------------------- | ------------------------------------------------------------------ |
| Commands and skills | [全部](/docs/zh-CN/commands) | 子集（输入 `/` 查看可用项）                                                   |
| MCP server config   | 是                     | 是（在聊天面板中使用 `/mcp` [添加和管理服务器](#connect-to-external-tools-with-mcp)） |
| Checkpoints         | 是                     | 是                                                                  |
| `!` bash shortcut   | 是                     | 否                                                                  |
| Tab completion      | 是                     | 否                                                                  |

<h3 id="rewind-with-checkpoints">
  Rewind with checkpoints
</h3>

VS Code extension 支持 checkpoints，它可以跟踪 Claude 的文件编辑并让您回退到之前的状态。将鼠标悬停在任何消息上以显示回退按钮，然后从三个选项中选择：

* **Fork conversation from here**：从此消息开始新的对话分支，同时保持所有代码更改完整
* **Rewind code to here**：将文件更改恢复到对话中的此点，同时保持完整的对话历史记录
* **Fork conversation and rewind code**：开始新的对话分支并将文件更改恢复到此点

有关 checkpoints 如何工作及其限制的完整详情，请参阅 [Checkpointing](/docs/zh-CN/checkpointing)。

<h3 id="run-cli-in-vs-code">
  Run CLI in VS Code
</h3>

要在 VS Code 中使用 CLI，请打开集成终端（Windows/Linux 上为 `` Ctrl+` ``，Mac 上为 `` Cmd+` ``）并运行 `claude`。CLI 会自动与您的 IDE 集成，以支持 diff 查看和诊断共享等功能。

安装 extension 不会将 `claude` 放在您的 shell PATH 上。extension 为其聊天面板捆绑了 CLI 的私有副本，但在终端中输入 `claude` 需要[独立 CLI 安装](/docs/zh-CN/setup)。运行一次安装，此页面上的命令（包括 `claude mcp add` 和 `claude --resume`）将在任何终端中工作。如果安装后仍未找到 `claude`，请[验证您的 PATH](/docs/zh-CN/troubleshoot-install#verify-your-path)。

如果使用外部终端，请在 Claude Code 中运行 `/ide` 以将其连接到 VS Code。

<h3 id="switch-between-extension-and-cli">
  Switch between extension and CLI
</h3>

extension 和 CLI 共享相同的对话历史记录。要在 CLI 中继续 extension 对话，请在终端中运行 `claude --resume`。这将打开一个交互式选择器，您可以在其中搜索并选择您的对话。

<h3 id="include-terminal-output-in-prompts">
  Include terminal output in prompts
</h3>

使用 `@terminal:name` 在您的提示中引用终端输出，其中 `name` 是终端的标题。这让 Claude 可以看到命令输出、错误消息或日志，而无需复制粘贴。

<h3 id="monitor-background-processes">
  Monitor background processes
</h3>

与 CLI 相比，extension 中后台任务的可见性受限。为了获得更好的可见性，让 Claude 输出命令，以便您可以在 VS Code 的集成终端中运行它。

<h3 id="connect-to-external-tools-with-mcp">
  Connect to external tools with MCP
</h3>

MCP（Model Context Protocol）服务器为 Claude 提供对外部工具、数据库和 API 的访问。

要在不离开 VS Code 的情况下管理 MCP 服务器，请在聊天面板中输入 `/mcp`。从打开的对话框中，您可以添加服务器、删除保存在本地、用户或项目[范围](/docs/zh-CN/mcp#mcp-installation-scopes)的服务器、启用或禁用服务器、重新连接到服务器以及管理 OAuth 身份验证。在对话框中添加和删除服务器需要 Claude Code v2.1.261 或更高版本。

您也可以在 VS Code 的集成终端中运行 `claude mcp add`（`` Ctrl+` `` 或 `` Cmd+` ``）。对话框和终端命令保存到相同的 MCP 配置，来自任一方的更改在您之后启动的对话中生效。下面的示例添加了 GitHub 的远程 MCP 服务器，该服务器使用作为标头传递的[个人访问令牌](https://github.com/settings/personal-access-tokens)进行身份验证：

```bash theme={null}
claude mcp add --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer YOUR_GITHUB_PAT"
```

将 `YOUR_GITHUB_PAT` 替换为您的个人访问令牌。`claude mcp add` 命令保存配置而不验证凭据，因此此处接受占位符值，但服务器稍后无法连接。要验证连接，请启动新对话，输入 `/mcp`，并检查服务器是否显示**已连接**。具有错误凭据的服务器显示**失败**。

配置后，要求 Claude 使用这些工具（例如，"Review PR #456"）。

要查找要连接的服务器，请参阅[查找和构建 MCP 服务器](/docs/zh-CN/mcp#find-and-build-mcp-servers)。

<h2 id="work-with-git">
  使用 git
</h2>

Claude Code 与 git 集成，帮助直接在 VS Code 中进行版本控制工作流。要求 Claude 提交更改、创建拉取请求或跨分支工作。要在具有自己的文件和分支的隔离 worktree 中启动 Claude，请参阅 [使用 worktrees 运行并行会话](/docs/zh-CN/worktrees)。

<h3 id="create-commits-and-pull-requests">
  创建提交和拉取请求
</h3>

Claude 可以暂存更改、编写提交消息，并根据您的工作创建拉取请求：

```text wrap theme={null}
commit my changes with a descriptive message
create a pr for this feature
summarize the changes I've made to the auth module
```

创建拉取请求时，Claude 会根据实际代码更改生成描述，并可以添加有关测试或实现决策的上下文。

<h2 id="use-third-party-providers">
  使用第三方提供商
</h2>

默认情况下，Claude Code 直接连接到 Anthropic 的 API。如果您的组织使用 Amazon Bedrock、Google Cloud 的 Agent Platform 或 Microsoft Foundry 来访问 Claude，请配置扩展以改用您的提供商：

<Steps>
  <Step title="禁用登录提示">
    打开[禁用登录提示设置](vscode://settings/claudeCode.disableLoginPrompt)并勾选该框。

    您也可以打开 VS Code 设置（Mac 上按 `Cmd+,` 或 Windows/Linux 上按 `Ctrl+,`），搜索"Claude Code login"，然后勾选**禁用登录提示**。
  </Step>

  <Step title="配置您的提供商">
    按照您的提供商的设置指南进行操作：

    * [Amazon Bedrock 上的 Claude Code](/docs/zh-CN/amazon-bedrock)
    * [Google Cloud 的 Agent Platform 上的 Claude Code](/docs/zh-CN/google-vertex-ai)
    * [Microsoft Foundry 上的 Claude Code](/docs/zh-CN/microsoft-foundry)

    这些指南涵盖在 `~/.claude/settings.json` 中配置您的提供商，这确保您的设置在 VS Code 扩展和 CLI 之间共享。
  </Step>
</Steps>

在第三方提供商上，扩展不提供需要 claude.ai 账户的功能，例如使用情况跟踪、[语音听写](/docs/zh-CN/voice-dictation)和用于[从 Claude.ai 恢复云会话](#resume-cloud-sessions-from-claude-ai)的 Web 标签页。来自早期 `/login` 的 claude.ai 登录会保留下来但未被使用：扩展不会在任何请求中发送它。

<h2 id="security-and-privacy">
  安全和隐私
</h2>

您的代码保持私密。Claude Code 处理您的代码以提供协助，但不会将其用于训练模型。有关数据处理的详细信息以及如何选择退出日志记录，请参阅[数据和隐私](/docs/zh-CN/data-usage)。

启用自动编辑权限后，Claude Code 可以修改 VS Code 配置文件（如 `settings.json` 或 `tasks.json`），VS Code 可能会自动执行这些文件。为了在处理不受信任的代码时降低风险：

* 为不受信任的工作区启用 [VS Code 受限模式](https://code.visualstudio.com/docs/editor/workspace-trust#_restricted-mode)
* 使用手动模式而不是自动编辑或自动编辑
* 在接受更改之前仔细审查更改

<h3 id="the-built-in-ide-mcp-server">
  内置 IDE MCP 服务器
</h3>

当扩展处于活动状态时，它运行一个本地 MCP 服务器，CLI 会自动连接到该服务器。这是 CLI 在 VS Code 的原生 diff 查看器中打开 diff、读取您当前的 `@`-mentions 选择，以及——当您在 Jupyter notebook 中工作时——要求 VS Code 执行单元格的方式。

服务器名为 `ide`，从 `/mcp` 中隐藏，因为没有什么需要配置的。但是，如果您的组织使用 `PreToolUse` hook 来允许列表 MCP 工具，您需要知道它的存在。

**选择和打开文件上下文。** 连接时，CLI 会在您发送的每个提示中包含您当前的编辑器选择和活动文件的路径作为上下文。当发生这种情况时，记录会显示一行 `⧉ Selected N lines from <file>`。要排除敏感文件（如 `.env`），请为其路径添加 [`Read` 拒绝规则](/docs/zh-CN/permissions#read-and-edit)。匹配的拒绝规则可防止该文件的选定文本和打开文件通知到达 Claude。

**传输和身份验证。** 服务器绑定到 `127.0.0.1` 上的随机端口，范围在 10000–65535，端口不可配置。传输是未加密的 `ws://`；因为套接字仅限于本地回环，任何可以捕获流量的进程也可以从锁文件中读取令牌，所以 TLS 不会增加保护。每次扩展激活都会生成一个新的随机身份验证令牌，将其写入 `~/.claude/ide/<port>.lock` 处的锁文件，CLI 必须将其作为 `X-Claude-Code-Ide-Authorization` 标头呈现才能连接。锁文件在 `0700` 目录中具有 `0600` 权限，因此只有运行 VS Code 的用户才能读取它。如果设置了 `CLAUDE_CONFIG_DIR`，锁文件将写入 `$CLAUDE_CONFIG_DIR/ide/` 目录。

**暴露给模型的工具。** 服务器托管十几个工具，但只有两个对模型可见。其余的是 CLI 用于自己的 UI 的内部 RPC——打开 diff、读取选择、保存文件——在工具列表到达 Claude 之前被过滤掉。

| 工具名称（如 hooks 所见）           | 功能                                                | 只读 |
| -------------------------- | ------------------------------------------------- | -- |
| `mcp__ide__getDiagnostics` | 返回语言服务器诊断——VS Code 的问题面板中的错误和警告。可选地限定到一个文件。       | 是  |
| `mcp__ide__executeCode`    | 在活动 Jupyter notebook 的内核中运行 Python 代码。请参阅下面的确认流程。 | 否  |

**Jupyter 执行始终先询问。** `mcp__ide__executeCode` 无法静默运行任何内容。在每次调用时，代码被插入为活动 notebook 末尾的新单元格，VS Code 将其滚动到视图中，原生快速选择器要求您**执行**或**取消**。取消——或用 `Esc` 关闭选择器——会向 Claude 返回错误，不会运行任何内容。当没有活动 notebook、未安装 Jupyter 扩展 (`ms-toolsai.jupyter`) 或内核不是 Python 时，该工具也会直接拒绝。

<Note>
  快速选择器确认与 `PreToolUse` hooks 分开。`mcp__ide__executeCode` 的允许列表条目让 Claude *提议*运行单元格；VS Code 内的快速选择器是让它*实际*运行的原因。
</Note>

<a id="troubleshooting" />

<h2 id="fix-common-issues">
  修复常见问题
</h2>

<h3 id="extension-won’t-install">
  扩展程序无法安装
</h3>

* 确保您拥有兼容的 VS Code 版本（1.94.0 或更高版本）
* 检查 VS Code 是否有权限安装扩展程序
* 尝试直接从 [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code) 安装

<h3 id="spark-icon-not-visible">
  Spark 图标不可见
</h3>

当您打开文件时，Spark 图标会出现在**编辑器工具栏**（编辑器右上角）。如果您看不到它：

1. **打开文件**：该图标需要打开文件。仅打开文件夹是不够的。
2. **检查 VS Code 版本**：需要 1.94.0 或更高版本（帮助 → 关于）
3. **重启 VS Code**：从命令面板运行"Developer: Reload Window"
4. **禁用冲突的扩展程序**：临时禁用其他 AI 扩展程序（Cline、Continue 等）
5. **检查工作区信任**：该扩展程序在受限模式下不起作用

或者，如果您已将 [`preferredLocation`](#extension-settings) 设置为 `sidebar`，或使用**Claude Code: Open in Side Bar** 打开了 Claude，请点击**状态栏**（右下角）中的"✱ Claude Code"。即使没有打开文件，这也能工作。您也可以使用**命令面板**（`Cmd+Shift+P` / `Ctrl+Shift+P`）并输入"Claude Code"。

<h3 id="cmd-esc-does-nothing-on-macos">
  Cmd+Esc 在 macOS 上无效
</h3>

在 macOS Tahoe 及更高版本上，系统游戏覆盖快捷键默认绑定到 `Cmd+Esc`，并在按键到达 VS Code 之前拦截它。要释放该快捷键：

1. 打开系统设置
2. 转到键盘，然后键盘快捷键，然后游戏控制器
3. 清除游戏覆盖复选框

或者，将扩展程序重新绑定到不同的键：打开 VS Code [键盘快捷键编辑器](https://code.visualstudio.com/docs/configure/keybindings)（`Cmd+K Cmd+S`），搜索 `Claude Code: Focus input`，并分配新的绑定。

<h3 id="claude-code-never-responds">
  Claude Code 从不响应
</h3>

如果 Claude Code 没有响应您的提示：

1. **检查您的互联网连接**：确保您有稳定的互联网连接
2. **开始新对话**：尝试开始新对话以查看问题是否仍然存在
3. **尝试 CLI**：从终端运行 `claude` 以查看是否获得更详细的错误消息

如果问题仍然存在，请[在 GitHub 上提交问题](https://github.com/anthropics/claude-code/issues)，并提供有关错误的详细信息。

<h2 id="uninstall-the-extension">
  卸载扩展
</h2>

要卸载 Claude Code 扩展：

1. 打开扩展视图（Mac 上按 `Cmd+Shift+X` 或 Windows/Linux 上按 `Ctrl+Shift+X`）
2. 搜索"Claude Code"
3. 点击**卸载**

如果你在 VS Code 集成终端中运行 `claude`，Claude Code 会自动重新安装扩展。要保持卸载状态，请在 `/config` 中关闭**自动安装 IDE 扩展**，或将 [`autoInstallIdeExtension`](/docs/zh-CN/settings-reference#autoinstallideextension) 设置为 `false`。你也可以将 [`CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL`](/docs/zh-CN/env-vars) 环境变量设置为 `1`。

要同时删除扩展数据并重置所有设置，请删除你的平台对应的扩展存储目录。

在 macOS 上：

```bash theme={null}
rm -rf ~/Library/"Application Support"/Code/User/globalStorage/anthropic.claude-code
```

在 Linux 上：

```bash theme={null}
rm -rf ~/.config/Code/User/globalStorage/anthropic.claude-code
```

在 Windows 上，在 PowerShell 中：

```powershell theme={null}
Remove-Item -Recurse -Force "$env:APPDATA\Code\User\globalStorage\anthropic.claude-code"
```

如需更多帮助，请参阅[故障排除指南](/docs/zh-CN/troubleshooting)。

<h2 id="next-steps">
  后续步骤
</h2>

现在您已在 VS Code 中设置了 Claude Code：

* [探索常见工作流](/docs/zh-CN/common-workflows)以充分利用 Claude Code
* [设置 MCP 服务器](/docs/zh-CN/mcp)以使用外部工具扩展 Claude 的功能。在聊天面板中使用 `/mcp` 添加和管理它们。
* [配置 Claude Code 设置](/docs/zh-CN/settings)以自定义允许的命令、hooks 等。这些设置在扩展和 CLI 之间共享。
