> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Desktop application

> 充分利用 Claude Code Desktop：使用 Git 隔离的并行会话、拖放窗格布局、集成终端和文件编辑器、侧边聊天、计算机使用、从手机 Dispatch 会话、可视化 diff 审查、应用预览、PR 监控、连接器和企业配置。

Claude Desktop 应用有三个选项卡：**Chat** 用于对话，**Cowork** 用于 [Dispatch 和更长的代理工作](https://claude.com/product/cowork)，**Code** 用于软件开发。本页是 Code 选项卡的参考。

<CardGroup cols={3}>
  <Card title="下载 macOS 版本" icon="apple" href="https://claude.ai/api/desktop/darwin/universal/dmg/latest/redirect?utm_source=claude_code&utm_medium=docs">
    适用于 Intel 和 Apple Silicon 的通用版本
  </Card>

  <Card title="下载 Windows 版本" icon="windows" href="https://claude.ai/api/desktop/win32/x64/setup/latest/redirect?utm_source=claude_code&utm_medium=docs">
    适用于 x64 处理器
  </Card>

  <Card title="获取 Claude for Linux（测试版）" icon="linux" href="/docs/zh-CN/desktop-linux">
    Ubuntu 和 Debian 的 apt 或 .deb
  </Card>
</CardGroup>

对于 Windows ARM64，请下载 [ARM64 安装程序](https://claude.ai/api/desktop/win32/arm64/setup/latest/redirect?utm_source=claude_code\&utm_medium=docs)。在 Linux 上，使用 apt 安装；请参阅 [Claude Desktop on Linux](/docs/zh-CN/desktop-linux)。

安装后，启动 Claude，登录，然后点击 **Code** 选项卡。有关首次会话的演练，请参阅[快速开始指南](/docs/zh-CN/desktop-quickstart)。

在 Code 选项卡中，每个对话都是一个**会话**：它有自己的聊天历史和项目文件夹，独立于任何其他会话。侧边栏列出你的会话，让你可以并行运行多个会话。在一个会话中，你可以：

* [使用 diff 视图审查和评论更改](#review-changes-with-diff-view)，然后[通过 CI 监控生成的 PR](#monitor-pull-request-status)
* [在浏览器窗格中预览你的运行应用](#preview-your-app)，同时 Claude 验证自己的更改，并[在其旁边打开外部网站](#browse-external-sites)
* 在 iOS Simulator 窗格中[观看 Claude 运行和测试你的 iOS 应用](/docs/zh-CN/desktop-ios-simulator)
* [整理窗格](#arrange-your-workspace)，将聊天、diff、浏览器、终端和文件编辑器并排放置
* 提出[侧边问题](#ask-a-side-question-without-derailing-the-session)，使用会话的上下文而不偏离主线
* 让 Claude [检查、消息或存档你的其他会话](#work-across-sessions)
* [连接外部工具](#connect-external-tools)，如 GitHub、Slack 和 Linear
* 让 Claude [打开应用和控制你的屏幕](#let-claude-use-your-computer)
* 在你的机器上、[云中](#run-long-running-tasks-in-the-cloud)或通过 [SSH](#ssh-sessions) 运行

有关[计划的定期工作](/docs/zh-CN/desktop-scheduled-tasks)、[快捷键](#keyboard-shortcuts)或[从手机发送任务](#sessions-from-dispatch)，请参阅链接的页面和部分。如果你已经使用基于终端的 CLI，请参阅 [CLI 比较](#coming-from-the-cli)了解哪些内容可以继续使用。

<h2 id="start-a-session">
  启动会话
</h2>

在发送第一条消息之前，在提示区域配置四件事：

* **环境**：选择 Claude 运行的位置。选择 **Local** 用于你的机器，**Cloud** 用于[云会话](#cloud-sessions)，该会话在你关闭应用后继续运行，[**SSH 连接**](#ssh-sessions)用于你管理的远程机器，或在 Windows 上选择 [**WSL 发行版**](/docs/zh-CN/desktop-wsl)。请参阅[环境配置](#environment-configuration)。
* **项目文件夹**：选择 Claude 工作的文件夹或存储库。对于云会话，你可以添加[多个存储库](#run-long-running-tasks-in-the-cloud)。
* **模型**：从发送按钮旁的下拉菜单中选择一个[模型](/docs/zh-CN/model-config#available-models)。你可以在会话期间更改此设置。
* **权限模式**：从[模式选择器](#choose-a-permission-mode)中选择 Claude 拥有多少自主权。你可以在会话期间更改此设置。

输入你的任务并按 **Enter** 启动。每个会话独立跟踪其自己的上下文和更改。

<h2 id="work-with-code">
  使用代码
</h2>

为 Claude 提供正确的上下文，控制它自主执行的工作量，并审查它所做的更改。

<h3 id="use-the-prompt-box">
  使用提示框
</h3>

输入您想让 Claude 做的事情，然后按 **Enter** 发送。Claude 会读取您的项目文件，进行更改，并根据您的[权限模式](#choose-a-permission-mode)运行命令。您可以随时重定向 Claude：点击停止按钮立即中断，或输入更正并按 **Enter** 发送，无需停止正在运行的操作。Claude 会在当前操作完成后立即读取更正，并在下一步之前进行调整。

提示框旁边的 **+** 按钮让您可以访问文件附件、[skill](#use-skills)、[连接器](#connect-external-tools) 和[插件](#install-plugins)。

<h3 id="accept-a-suggested-prompt">
  接受建议的提示词
</h3>

Claude 回复后，Code 选项卡可能会在空的输入框中以灰色文本显示建议的下一条提示词。Claude Code 通过一个简短的后台请求，根据您的对话[生成每条建议](/docs/zh-CN/interactive-mode#prompt-suggestions)，该请求会计入您套餐的用量限制或您的 API 费用。

* **使用建议**：按 **Tab** 或 **右箭头** 将其放入输入框，根据需要进行编辑，然后按 **Enter** 发送。在接受建议之前按 **Enter** 不会发送它。
* **编写自己的提示词**：直接开始输入。建议仅在输入框为空且没有附加文件时显示。

前往 **Settings > Claude Code**，在 **Sessions** 下关闭 **Prompt suggestions**，即可从每个会话下次启动或恢复时起停止显示建议。

<h3 id="add-files-and-context-to-prompts">
  向提示添加文件和上下文
</h3>

提示框支持两种方式来引入外部上下文：

* **@mention 文件**：输入 `@` 后跟文件名，将文件添加到对话上下文。Claude 随后可以读取和引用该文件。@mention 在云端或 WSL 会话中不可用。
* **附加文件**：使用附件按钮将图像、PDF 和其他文件附加到您的提示，或直接将文件拖放到提示中。这对于共享错误的屏幕截图、设计模型或参考文档很有用。

<h3 id="choose-a-permission-mode">
  选择权限模式
</h3>

权限模式控制 Claude 在会话期间的自主程度：它是否在编辑文件、运行命令或两者之前询问。您可以随时使用发送按钮旁边的模式选择器切换权限模式。要自己批准每项更改，请切换到 Manual。

要为新的本地会话设置默认模式，请将 `permissions.defaultMode` 添加到您的[设置文件](/docs/zh-CN/settings#where-settings-live)。桌面应用读取与 CLI 相同的设置文件。您在选择器中选择的模式会被记住（按文件夹），并对该文件夹优先于 `defaultMode`，除了 Plan，它仅适用于当前会话。

| 模式 | 设置键 | 行为 |
| - | - | - |
| **Manual** | `default` | Claude 在编辑文件或运行命令之前询问。您会看到 diff，可以接受或拒绝每项更改。 |
| **Accept edits** | `acceptEdits` | Claude 自动接受文件编辑和常见的文件系统命令，如 `mkdir`、`touch` 和 `mv`，但在运行其他终端命令之前仍会询问。当您信任文件更改并希望更快迭代时，请使用此选项。 |
| **Plan** | `plan` | Claude 读取文件并运行命令进行探索，然后提出计划而不编辑您的源代码。适合复杂任务，您想先审查方法。 |
| **Auto** | `auto` | Claude 运行时不需要常规提示；在执行 shell 命令和网络请求等操作之前，后台分类器会检查它们是否与您的请求一致。当[自动模式可用](#auto-mode-availability)时出现；没有单独的设置切换。 |
| **Bypass permissions** | `bypassPermissions` | Claude 运行时不需要权限提示，除了[任何模式都不会自动批准的操作](/docs/zh-CN/permission-modes#actions-no-mode-auto-approves)、当 Claude [在外部网站上操作](#browse-external-sites)时的安全分类器，或桌面操作（Claude 总是首先询问），例如[归档会话](#work-across-sessions)。等同于 CLI 中的 `--dangerously-skip-permissions`。在 Pro 和 Max 计划上，在您的设置 → Claude Code 中启用它，在"Allow bypass permissions mode"下；在 Team 和 Enterprise 计划上没有设置切换，由组织策略控制它。仅在沙箱容器或虚拟机中使用。 |

Code 选项卡的早期版本将这些模式标记为 Ask permissions、Auto accept edits 和 Plan mode。

`dontAsk` 权限模式仅在 [CLI](/docs/zh-CN/permission-modes#allow-only-pre-approved-tools-with-dontask-mode) 中可用。

<Tip title="最佳实践">
  在 Plan 中开始复杂任务，以便 Claude 在进行更改之前制定方法。一旦您批准计划，切换到 Accept edits 或 Manual 来执行它。有关此工作流的更多信息，请参阅[先探索，然后计划，然后编码](/docs/zh-CN/best-practices#explore-first-then-plan-then-code)。
</Tip>

云端会话支持 Accept edits、Plan 和 Auto。Accept edits 对应于 `default` 模式：云端会话预先批准文件编辑，因此选择器显示 Accept edits 而不是 Manual。Bypass permissions 在云端会话中不可用，包括[自托管环境](/docs/zh-CN/self-hosted-environments)中的会话。

Enterprise 管理员可以限制哪些权限模式可用。有关详细信息，请参阅[企业配置](#enterprise-configuration)。

<h4 id="auto-mode-availability">
  自动模式可用性
</h4>

自动模式对 Anthropic API 上的所有用户可用，需要 Claude Opus 4.6 或更高版本、Sonnet 4.6 或更高版本、Haiku 5.5，或 [Fable 模型](/docs/zh-CN/model-config#work-with-fable)。组织管理员可以使用[托管设置](#managed-settings)中的 `disableAutoMode` 键关闭自动模式。

在将 Desktop 路由到 Google Cloud 的 Agent Platform 的 Enterprise 部署中，自动模式也默认可用；有关支持的模型，请参阅 [Bedrock、Agent Platform 或 Foundry 上的自动模式](/docs/zh-CN/permission-modes#enable-auto-mode-on-bedrock-agent-platform-or-foundry)。

<h3 id="preview-your-app">
  预览您的应用
</h3>

Claude 可以启动开发服务器并在浏览器窗格中打开它以验证其更改。这适用于前端 Web 应用以及后端服务器：Claude 可以测试 API 端点、查看服务器日志，并迭代它发现的问题。在大多数情况下，Claude 在编辑项目文件后自动启动服务器。您也可以随时要求 Claude 进行预览。默认情况下，Claude [自动验证](#auto-verify-changes)每次编辑后的更改。

浏览器窗格也可以打开项目中的静态 HTML 文件、PDF、图像和视频。在聊天中点击 HTML、PDF、图像或视频路径以在那里打开它。

从浏览器窗格，您可以：

* 直接在浏览器窗格中与运行的应用交互
* 观看 Claude 自动验证其自己的更改：它拍摄屏幕截图、检查 DOM、点击元素、填充表单，并修复它发现的问题
* 从浏览器窗格标题栏中的 **Dev servers** 菜单启动或停止服务器，或一次停止所有服务器
* 通过浏览器窗格 **⋮** 菜单中的 **Keep cookies**，选择浏览器在您退出应用后是否保留 cookie，这样您就不必在开发期间重新登录

Claude 根据您的项目创建初始服务器配置。如果您的应用使用自定义开发命令，编辑 `.claude/launch.json` 以匹配您的设置。有关完整参考，请参阅[配置预览服务器](#configure-preview-servers)。

要清除浏览器保存的数据，请在浏览器窗格的 **⋮** 菜单中选择 **Clear browsing data**。要完全关闭浏览器，请在 **Settings > Claude Code** 中关闭 **Browser tools**。

<h3 id="browse-external-sites">
  浏览外部网站
</h3>

浏览器窗格是一个选项卡式浏览器，因此您可以在运行的应用旁边打开文档、问题跟踪器或任何其他网站。要打开浏览器，在 macOS 上按 **Cmd+Shift+B** 或在 Windows 上按 **Ctrl+Shift+B**，或点击会话标题栏中的 **Browser**。您可以在窗格中登录网站，包括弹出式登录流程，例如 Google OAuth。

您第一次点击聊天中的外部链接时，会出现一个对话框，询问链接是在浏览器窗格中打开还是在您的默认浏览器中打开。要在之后更改您的选择，请使用浏览器窗格 **⋮** 菜单中的 **Open links in built-in browser**。在 macOS 上 **Cmd** 点击或在 Windows 上 **Ctrl** 点击会直接在您的默认浏览器中打开链接。

Claude 可以使用与[验证您的应用](#preview-your-app)相同的工具读取和交互外部页面，并进行两项额外的安全检查：

* 安全分类器在每个权限模式中审查 Claude 在外部页面上的写入操作，例如点击和输入。这些与[自动模式](#choose-a-permission-mode)使用的分类器相同，当它们标记操作时，无论模式如何，您都会收到权限提示。
* 在 Auto 和 Bypass permissions 以外的权限模式中，在 Claude 导航到新网站之前也会应用域名允许列表检查。

<h4 id="approve-claude’s-actions-on-a-site">
  批准 Claude 在网站上的操作
</h4>

Claude 第一次在外部网站上操作时，会出现权限卡，Claude 等待您的选择：**Allow once**、**Always allow** 或 **Deny**。**Allow once** 批准操作而不保存任何内容。**Always allow** 在您的设备上保存该网站的批准，您可以在设置中撤销它。每个网站都需要自己的批准，包括子域。您的本地开发服务器和项目文件不需要批准，因此[自动验证](#auto-verify-changes)继续工作而不需要提示。

即使在批准的网站上，Claude 也不会在没有您的输入的情况下购买物品、创建账户或绕过 CAPTCHA。在浏览器窗格中浏览使用与 [Claude in Chrome extension](/docs/zh-CN/chrome) 相同的安全模型。有关 Claude 如何处理敏感网站和风险操作的信息，请参阅[安全使用 Chrome 中的 Claude](https://support.claude.com/en/articles/12902428-using-claude-in-chrome-safely)。

<h4 id="choose-between-the-browser-and-the-chrome-extension">
  在浏览器和 Chrome 扩展之间选择
</h4>

浏览器窗格使用干净的浏览器配置文件，与您的个人浏览器分开，没有您保存的登录或历史记录。将其用于构建和测试您的应用以及不需要您的身份的网站。当您想让 Claude 在您的登录会话中以您的身份操作时，请改用 [Claude in Chrome extension](/docs/zh-CN/chrome)，它共享您的浏览器的登录状态。

<h4 id="restrict-external-browsing-for-your-organization">
  为您的组织限制外部浏览
</h4>

浏览器遵循与 Claude in Chrome 扩展相同的[网站允许列表和阻止列表控制](https://support.claude.com/en/articles/13065128-claude-in-chrome-admin-controls)。如果您的组织已经为扩展配置了这些列表，浏览器会自动遵循它们。管理员也可以使用 [`browserExternalPageTools` 托管设置](#managed-settings)关闭 Claude 在外部页面上的工具。禁用工具后，用户仍然可以访问外部网站；Claude 的工具无法读取或操作它们。

要完全关闭外部浏览，请将 [`disableBrowserExternalNavigation` 托管设置](#managed-settings)设置为 `true`。这会阻止浏览器中的所有外部导航，包括您的组织允许列表上的网站；localhost 开发服务器和文件预览继续工作。使用 `browserExternalPageTools` 让用户继续浏览外部网站而不使用 Claude 的工具，使用 `disableBrowserExternalNavigation` 为用户和 Claude 阻止外部网站。

<h3 id="review-changes-with-diff-view">
  使用 diff 视图审查更改
</h3>

Claude 对您的代码进行更改后，diff 视图让您在创建 Pull Request 之前逐个文件审查修改。

当 Claude 更改文件时，会出现一个 diff 统计指示器，显示添加和删除的行数，例如 `+12 -1`。点击此指示器打开 diff 查看器，它在左侧显示文件列表，在右侧显示每个文件的更改。

要对特定行进行评论，点击 diff 中的任何行以打开评论框。输入您的反馈并按 **Enter** 添加评论。在多行添加评论后，一次提交所有评论：

* **macOS**：按 **Cmd+Enter**
* **Windows**：按 **Ctrl+Enter**

Claude 读取您的评论并进行请求的更改，这些更改显示为您可以审查的新 diff。

<h3 id="review-your-code">
  审查您的代码
</h3>

要让 Claude 在您提交之前审查您的更改，请在[输入框](#use-the-prompt-box)中输入 `/code-review`。审查完成后，结果会出现在对话中。

在本地、[SSH](#ssh-sessions) 和 [WSL](/docs/zh-CN/desktop-wsl) 会话中，结果会显示为一张按文件分组的 **Code review** 卡片。使用该卡片处理这些结果：

* 点击 **Walk through in diff** 打开 diff 视图，逐条查看结果。当前 diff 中的结果会显示在其对应的行上，您可以在那里点击 **Fix this one** 或忽略它。
* 点击 **Apply fixes** 让 Claude 修复仍未处理的结果。

在任何会话中，您也可以在输入框中要求 Claude 修复审查发现的问题。有关 `/code-review` 检查的内容及其接受的参数，请参阅[在本地审查 diff](/docs/zh-CN/code-review#review-a-diff-locally)。

<h3 id="monitor-pull-request-status">
  监控 Pull Request 状态
</h3>

打开 Pull Request 后，CI 状态栏会出现在会话中。Claude Code 使用 GitHub CLI 轮询检查结果并显示失败。

* **Auto-fix CI & address comments**：启用后，Claude 会通过读取失败输出并迭代来自动尝试修复失败的 CI 检查。在本地会话中，当评论作者是仓库所有者、组织成员、协作者或 GitHub App 时，Claude 还会处理除您之外的其他人留下的新审查评论。
* **Auto-merge when ready**：启用后，Claude 在所有检查通过后合并 PR。合并方法是 squash。请先在您的 [GitHub 仓库设置](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-auto-merge-for-pull-requests-in-your-repository)中启用自动合并；没有它，Claude 无法合并 PR。

要启用这些选项，请点击状态栏中的 **CI**。要在 PR 合并或关闭后自动归档会话，请在 **Settings > Claude Code** 中打开[自动归档](#work-in-parallel-with-sessions)。

<Note>
  PR 监控需要在您的机器上安装并认证 [GitHub CLI (`gh`)](https://cli.github.com/)。如果未安装 `gh`，Desktop 会在您第一次尝试创建 PR 时提示您安装它。
</Note>

<h2 id="arrange-your-workspace">
  整理工作区
</h2>

Code 选项卡围绕可以以任何布局排列的窗格构建：聊天、diff、浏览器、终端、文件、计划、任务和子代理，以及 macOS 上的 [iOS Simulator](/docs/zh-CN/desktop-ios-simulator)。通过其标题拖动窗格来重新定位它，或拖动窗格边缘来调整大小。在 macOS 上按 **Cmd+\\** 或在 Windows 上按 **Ctrl+\\** 来关闭焦点窗格。点击会话标题栏中的 **Terminal**、**Changes** 或 **Browser** 来打开终端、diff 或浏览器窗格。它们旁边的 **⋮** 菜单可打开更多窗格（例如 **Files**），并在窗口过窄无法显示这些按钮时容纳它们。

要在多个屏幕上工作，可以将 diff 或终端等窗格弹出到其自己的窗口中，完成后再停靠回来。Claude 继续在主窗口中工作。

<Note>
  本部分中的窗格布局、终端、文件编辑器和视图模式需要 Claude Desktop v1.2581.0 或更高版本。在 macOS 上打开 **Claude → Check for Updates** 或在 Windows 上打开 **Help → Check for Updates** 来更新。
</Note>

<h3 id="run-commands-in-the-terminal">
  在终端中运行命令
</h3>

集成终端让您无需切换到另一个应用即可在会话旁运行命令。点击会话标题栏中的 **Terminal**，或在 macOS 或 Windows 上按 **Ctrl+\`**。终端在您的会话工作目录中打开，并与 Claude 共享相同的环境，因此 `npm test` 或 `git status` 等命令看到 Claude 正在编辑的相同文件。要打开第二个终端选项卡，点击终端窗格标题中的 **+** 或右键点击聊天中的文件夹来选择 **Open in terminal**。终端在本地和 [SSH](#ssh-sessions) 会话中可用。

<h3 id="open-and-edit-files">
  打开和编辑文件
</h3>

点击聊天或 diff 查看器中的文件路径在文件窗格中打开它。HTML、PDF、图像和视频路径改为在[浏览器窗格](#preview-your-app)中打开。进行现场编辑并点击 **Save** 来写回。如果文件自您打开它以来在磁盘上更改，窗格会警告您并让您覆盖或丢弃。点击 **Discard** 来恢复您的编辑，或点击窗格标题中的路径来复制绝对路径。

文件窗格在本地和 SSH 会话中可用。对于云端会话，要求 Claude 进行更改。

<h3 id="open-files-in-other-apps">
  在其他应用中打开文件
</h3>

右键点击聊天、diff 查看器或文件窗格中的任何文件路径来打开上下文菜单：

* **Attach as context**：将文件添加到您的下一个提示词
* **Open in**：在已安装的编辑器（如 VS Code、Cursor 或 Zed）中打开文件
* **Show in Finder**（macOS）、**Show in Explorer**（Windows）：打开包含文件夹
* **Copy path**：将绝对路径复制到您的剪贴板

<h3 id="switch-view-modes">
  切换视图模式
</h3>

视图模式控制聊天会话记录中显示的详细程度。要切换视图模式，请从会话标题旁的插入符号打开会话菜单并选择 **Transcript view**，或在 macOS 或 Windows 上按 **Ctrl+O** 循环切换。只有在 Claude 于您正在查看的会话中产生思考后，Thinking 模式才会出现在菜单中。

| 模式 | 显示内容 |
| - | - |
| **Normal** | 工具调用折叠成摘要，带有完整文本回复 |
| **Thinking** | 工具调用折叠成摘要，加上 Claude 的思考 |
| **Verbose** | Claude 采取的每个工具调用、文件读取和中间步骤，加上 Claude 的思考 |

使用 Thinking 来跟踪 Claude 的推理，工具调用仍然折叠。在调试 Claude 为什么采取特定操作时使用 Verbose。Claude Desktop 1.46388.1 之前的版本也列出了 Summary 模式，仍然设置为 Summary 的会话在您更新后会以 Normal 打开。

<h3 id="keyboard-shortcuts">
  快捷键
</h3>

在 macOS 上按 **Cmd+/** 或在 Windows 上按 **Ctrl+/** 来查看 Code 选项卡中可用的所有快捷键。在 Windows 上，对下面的快捷键使用 **Ctrl** 代替 **Cmd**。会话循环、终端切换和视图模式切换在每个平台上使用 **Ctrl**。

| 快捷键 | 操作 |
| - | - |
| `Cmd` `/` | 显示快捷键 |
| `Cmd` `N` | 新会话 |
| `Cmd` `W` | 关闭会话 |
| `Ctrl` `Tab` / `Ctrl` `Shift` `Tab` | 下一个或上一个会话 |
| `Cmd` `Shift` `]` / `Cmd` `Shift` `[` | 下一个或上一个会话 |
| `Esc` | 停止 Claude 的回复 |
| `Tab` / `Right arrow` | 在空输入框中[接受建议的提示词](#accept-a-suggested-prompt) |
| `Cmd` `Shift` `D` | 切换 diff 窗格 |
| `Cmd` `Shift` `B` | 切换浏览器窗格 |
| `Cmd` `Shift` `S` | 在浏览器中选择元素 |
| `Ctrl` `` ` `` | 切换终端窗格 |
| `Cmd` `\` | 关闭焦点窗格 |
| `Cmd` `;` | 打开侧边聊天 |
| `Ctrl` `O` | 循环视图模式 |
| `Cmd` `Shift` `M` | 打开权限模式菜单 |
| `Cmd` `Shift` `I` | 打开模型菜单 |
| `Cmd` `Shift` `E` | 打开工作量菜单 |
| `1`–`9` | 在打开的菜单中选择项目 |

这些快捷键适用于 Code 选项卡。在 Desktop 中，`Shift+Tab` 不会像在终端的[交互模式](/docs/zh-CN/interactive-mode#keyboard-shortcuts)中那样循环切换权限模式。

<h3 id="check-usage">
  检查使用情况
</h3>

点击模型选择器旁的使用环形图来查看您当前的上下文窗口使用情况和您的计划在该期间的使用情况。上下文使用是按会话的；计划使用在所有 Claude Code 使用入口上共享。

<h2 id="let-claude-use-your-computer">
  让 Claude 使用你的计算机
</h2>

计算机使用让 Claude 打开你的应用、控制你的屏幕，并像你一样直接在你的机器上工作。要求 Claude 与没有 CLI 的桌面工具交互，或自动化只能通过 GUI 工作的东西。对于运行和测试 iOS 应用，Desktop 会打开专用的 [iOS Simulator 窗格](/docs/zh-CN/desktop-ios-simulator)，而不是控制你的屏幕；该窗格无需启用计算机使用即可工作。

<Note>
  计算机使用是 macOS 和 Windows 上的研究预览版，需要 Pro 或 Max 计划。它在 Team 或 Enterprise 计划上不可用。Claude Desktop 应用必须运行。
</Note>

计算机使用默认关闭。[在设置中启用它](#enable-computer-use)，然后 Claude 才能控制你的屏幕。在 macOS 上，你还需要授予辅助功能和屏幕录制权限。

在 macOS 上，计算机使用也可以在后台运行：Claude 在你批准的应用中工作，同时你继续工作。

<Warning>
  与[沙箱化 Bash 工具](/docs/zh-CN/sandboxing)不同，计算机使用在你的实际桌面上运行，可以访问你批准的任何内容。Claude 检查每个操作并标记来自屏幕内容的潜在提示注入，但信任边界不同。有关最佳实践，请参阅[计算机使用安全指南](https://support.claude.com/en/articles/14128542)。
</Warning>

<h3 id="when-computer-use-applies">
  何时应用计算机使用
</h3>

Claude 有多种方式与应用或服务交互，计算机使用是最广泛和最慢的。它首先尝试最精确的工具：

* 如果你有一个服务的[连接器](#connect-external-tools)，Claude 使用连接器。
* 如果任务是 shell 命令，Claude 使用 Bash。
* 如果任务是浏览器工作且你已设置[Chrome 中的 Claude](/docs/zh-CN/chrome)，Claude 使用那个。
* 如果任务是运行或测试 iOS 应用，Claude 使用 [iOS Simulator 窗格](/docs/zh-CN/desktop-ios-simulator)，它不使用屏幕控制。
* 如果以上都不适用，Claude 使用计算机使用。

[按应用访问层](#app-permissions)强化了这一点：浏览器限制为仅查看，终端和 IDE 限制为仅点击，即使计算机使用处于活跃状态，也会引导 Claude 使用专用工具。屏幕控制保留给其他工具无法到达的东西，如原生应用、硬件控制面板或没有 API 的专有工具。

<h3 id="enable-computer-use">
  启用计算机使用
</h3>

计算机使用默认关闭。如果你要求 Claude 做需要它的事情而它关闭时，Claude 会告诉你如果在设置中启用计算机使用，它可以完成任务。

<Steps>
  <Step title="更新桌面应用">
    确保你有最新版本的 Claude Desktop。在 macOS 和 Windows 上，在 [claude.com/download](https://claude.com/download) 下载或更新；在 Linux 上，通过你的包管理器更新（[说明](/docs/zh-CN/desktop-linux)）。然后重启应用。
  </Step>

  <Step title="打开切换">
    在桌面应用中，转到**设置 > 此计算机 > 系统**。在**计算机使用**下，打开**启用计算机使用**。在 Windows 上，切换立即生效，设置完成。在 macOS 上，继续下一步。

    如果你看不到切换，确认你在 macOS 或 Windows 上使用 Pro 或 Max 计划，然后更新并重启应用。
  </Step>

  <Step title="授予 macOS 权限">
    在 macOS 上，在切换生效之前授予两个系统权限：

    * **Accessibility**：让 Claude 点击、输入和滚动
    * **Screen Recording**：让 Claude 看到你屏幕上的内容

    设置页面显示每个权限的当前状态。如果任一被拒绝，点击徽章打开相关的系统设置窗格。
  </Step>
</Steps>

<h3 id="app-permissions">
  应用权限
</h3>

Claude 第一次需要使用应用时，会话中会出现提示。点击**允许此会话**或**拒绝**。批准持续当前会话，或在 [Dispatch 生成的会话](#sessions-from-dispatch)中持续 30 分钟。

提示还显示 Claude 为该应用获得的控制级别。这些层由应用类别固定，无法更改：

| 层 | Claude 可以做什么 | 适用于 |
| :- | :- | :- |
| 仅查看 | 在屏幕截图中看到应用 | 浏览器、交易平台 |
| 仅点击 | 点击和滚动，但不能输入或使用快捷键 | 终端、IDE |
| 完全控制 | 点击、输入、拖动和使用快捷键 | 其他所有内容 |

像终端、Finder 或文件浏览器以及系统设置或设置这样具有广泛影响的应用在提示中显示额外警告，以便你知道批准它们授予什么。

**设置 > 此计算机 > 系统**中的**计算机使用**部分包含以下选项：

* **拒绝的应用**：在此处添加应用以拒绝它们而不提示。Claude 可能仍然通过允许应用中的操作间接影响被拒绝的应用，但它无法直接与被拒绝的应用交互。
* **Claude 完成时取消隐藏应用**：当计算机使用不在后台运行时，Claude 隐藏你的其他窗口，以便它仅与批准的应用交互。当 Claude 完成时，隐藏的窗口被恢复，除非你关闭此设置。

<h2 id="manage-sessions">
  管理会话
</h2>

每个会话是一个独立的对话，拥有自己的上下文和更改。您可以并行运行多个会话、开启侧边聊天、让 Claude 查看您的其他会话并向其发送消息、将工作发送到云端，或让 Dispatch 从您的手机为您启动会话。

<h3 id="work-in-parallel-with-sessions">
  使用会话并行工作
</h3>

点击侧边栏中的 **+ New session**，或在 macOS 上按 **Cmd+N**、在 Windows 上按 **Ctrl+N**，即可并行处理多个任务。按 **Ctrl+Tab** 和 **Ctrl+Shift+Tab** 可在侧边栏中循环切换会话。对于 Git 仓库，选择分支名称旁边的 **worktree** 选项，即可使用 [Git worktrees](/docs/zh-CN/worktrees) 为会话提供项目的独立隔离副本，这样一个会话中的更改在您提交之前不会影响其他会话。

要同时查看两个会话，请在 macOS 上按住 **Cmd** 或在 Windows 上按住 **Ctrl** 并点击侧边栏中的会话。该会话会在第二个窗格中打开，与您已打开的会话并排显示。分屏处于活跃状态时，点击侧边栏中的另一个会话会替换当前具有焦点的窗格。在 macOS 上按 **Cmd+\\** 或在 Windows 上按 **Ctrl+\\** 可关闭具有焦点的窗格并返回单个会话。

Worktree 默认存储在 `<project-root>/.claude/worktrees/` 中。您可以在设置 → Claude Code 中的"Worktree location"下将其更改为自定义目录。您还可以设置一个分支前缀，该前缀会添加到每个 worktree 分支名称的前面，这有助于让 Claude 创建的分支保持条理。完成后要删除 worktree，请将鼠标悬停在侧边栏中的会话上并点击存档图标。要让会话在其 Pull Request 合并或关闭时自动存档，请在设置 → Claude Code 中打开 **Auto-archive after PR merge or close**。自动存档仅适用于已完成运行的本地会话。

要在新 worktree 中包含被 gitignore 的文件（如 `.env`），请在项目根目录中创建一个 [`.worktreeinclude` 文件](/docs/zh-CN/worktrees#copy-gitignored-files-into-worktrees)。

<Note>
  会话隔离需要 [Git](https://git-scm.com/downloads)。大多数 Mac 默认已包含 Git。在终端中运行 `git --version` 进行检查；如果输出了版本号，则说明 Git 已安装。如果遇到 Git 错误，请在 [Cowork 选项卡](https://claude.com/product/cowork)中请 Claude 帮助排查您的设置。
</Note>

使用侧边栏顶部的控件可按状态、项目或环境筛选会话，并按项目对会话分组。要重命名会话，请点击活跃会话顶部工具栏中的会话标题。

要检查上下文使用情况，请参阅[检查使用情况](#check-usage)。当上下文填满时，Claude 会自动总结对话并继续工作。您也可以输入 `/compact` 提前触发总结并释放上下文空间。有关压缩工作原理的详细信息，请参阅[上下文窗口](/docs/zh-CN/how-claude-code-works#the-context-window)。

当 Code 会话完成任务且您当前未在查看该会话时，桌面应用会发送操作系统通知。对于属于某个[项目](/docs/zh-CN/claude-projects#see-what-needs-you-in-overview)的会话，您将收到该项目的通知。

<h3 id="ask-a-side-question-without-derailing-the-session">
  在不偏离会话的情况下提出侧边问题
</h3>

侧边聊天让您可以向 Claude 提出一个使用会话上下文的问题，但不会向主对话添加任何内容。当您想要理解一段代码、检查某个假设或探索某个想法，又不想让会话偏离方向时，可以使用它。

在 macOS 上按 **Cmd+;** 或在 Windows 上按 **Ctrl+;** 打开侧边聊天，或在输入框中输入 `/btw`。侧边聊天可以读取主线程中截至该时刻的所有内容。完成后，关闭侧边聊天，即可从中断处继续主会话。

侧边聊天可在本地、SSH 和 WSL 会话中使用。桌面应用不会将侧边聊天保存到磁盘，因此关闭应用后无法再返回之前的侧边聊天。

<h3 id="watch-background-tasks">
  查看后台任务
</h3>

任务窗格显示当前会话内正在运行的后台工作：子代理、后台 shell 命令和[动态工作流](/docs/zh-CN/workflows)。会话中有后台工作后，可通过标题栏 **⋮** 菜单中的 **Background tasks** 打开该窗格。

点击任意条目可在子代理窗格中查看其输出或将其停止。要查看其他会话正在做什么，请使用[侧边栏](#work-in-parallel-with-sessions)，或请 Claude [为您查看](#work-across-sessions)。

<h3 id="work-across-sessions">
  跨会话工作
</h3>

Claude 可以列出您的其他 Code 选项卡会话，读取每个会话一直在做的工作，并在它们之间发送消息。用自然语言提问即可："哪个会话涉及了身份验证重构？"、"API 会话得出了什么结论？"或"告诉支付会话 schema 已更改"。您也可以请 Claude 重命名或存档会话。Claude 存档会话的方式与侧边栏的存档图标相同，因此可以让它清理 PR 已合并的会话。

通过此使用入口，Claude 只能看到桌面应用自身运行的会话：Code 选项卡中的本地、[SSH](#ssh-sessions) 和 [WSL](/docs/zh-CN/desktop-wsl) 会话。Claude 看不到云端会话，也看不到您从终端 CLI 或 VS Code 扩展启动的会话，即使它们位于同一项目的 worktree 中也是如此。因此，如果打开了九个终端 worktree 和两个桌面会话，在其中一个桌面会话中回答的 Claude 只会报告另一个桌面会话。Claude 从不会列出您正在提问的那个会话。默认情况下，它可以看到最近活跃的 20 个会话，并跳过已存档的会话，除非您明确要求。[跨会话消息传递](/docs/zh-CN/cross-session-messaging)则另外让 Claude 可以向[您的其他 Claude Code 会话](/docs/zh-CN/cross-session-messaging#see-which-sessions-claude-can-reach)发送消息，包括终端会话。

当 Claude 通过此使用入口向另一个会话发送消息时，Claude Code 会在对方会话中将其显示为一张卡片，卡片标有发送会话的标题和返回链接，因此您始终可以知道消息来自何处。如果接收会话正在执行任务，Claude Code 会暂存该消息，Claude 会在当前工作完成后读取它。接收方的 Claude 可以回复，Claude Code 会通过此使用入口将回复传回。Claude 无法向已存档的会话发送消息，并会在消息未送达时告知您。

Claude Code 在跨会话时应用四项安全行为：

* 在存档任何会话之前，Claude 会先询问您。在每种权限模式下您都会看到批准卡片，包括 Auto 和 Bypass permissions。
* 通过此使用入口，Claude 无法从无人查看的会话（例如定时任务运行）发送跨会话消息，也无法向此类会话发送消息。
* Claude Code 会根据接收会话的[入站控制](/docs/zh-CN/cross-session-messaging#control-inbound-messages)检查来自此使用入口的每条消息，即使接收会话本身未启用[跨会话消息传递](/docs/zh-CN/cross-session-messaging#availability)。如果您在接收会话中将 [`crossSessionInbound`](/docs/zh-CN/settings-reference#crosssessioninbound) 设置为 `refuse`，Claude Code 会丢弃来自此使用入口的消息。Claude Code 会向 Claude 桌面应用报告该拒绝。在 v2.1.234 之前，对于未启用跨会话消息传递的接收会话，Claude Code 会丢弃来自此使用入口的所有消息。
* Claude Code 会引用每条传入消息并注明其来自哪个发送会话，且 Claude 在据此执行操作时仍遵循接收会话自身的权限设置。

Claude 还可以建议新会话。当它注意到值得修复但超出当前任务范围的问题时，会在聊天中以任务标签的形式提供该工作。点击该标签即可在拥有独立 worktree 的新会话中开始该工作；Claude 会继续您的当前会话而不受干扰。

<h3 id="run-long-running-tasks-in-the-cloud">
  在云端运行长时间运行的任务
</h3>

对于大型重构、测试套件、迁移或其他长时间运行的任务，请在启动会话时选择 **Cloud** 而不是 **Local**。云端会话默认在 Anthropic 管理的基础设施上运行，即使您关闭应用或关闭计算机也会继续运行。您可以随时回来查看进度，或引导 Claude 转向不同的方向。您也可以从 [claude.ai/code](https://claude.ai/code) 或 [Claude 移动应用](/docs/zh-CN/mobile)监控云端会话。

云端会话还支持多个仓库。选择云端环境后，点击所选仓库旁边的 **+** 按钮即可向会话添加更多仓库。每个仓库都有自己的分支选择器。这对于跨越多个代码库的任务很有用，例如更新共享库及其使用方。

有关云端会话工作方式的更多信息，请参阅[在云端使用 Claude Code](/docs/zh-CN/claude-code-on-the-web)。当一项工作需要许多云端会话时，请在侧边栏中选择 **Projects** 创建一个[项目](/docs/zh-CN/claude-projects)，Claude 会在一个对话中为您启动并跟踪这些会话。

<h3 id="continue-in-another-surface">
  在另一个使用入口继续
</h3>

要在其他地方继续会话，请通过会话标题旁边的下拉箭头或侧边栏中该会话所在的行打开会话菜单，然后选择 **Open in**：

* 选择 **Cloud** 可将会话作为[云端会话](/docs/zh-CN/claude-code-on-the-web)继续，您的对话会以摘要形式带过去。在您确认之前，对话框会说明您的文件是否也会一并移动，以及云端会话就绪后此会话是否会被存档。通过 [SSH](#ssh-sessions) 或在 [WSL](/docs/zh-CN/desktop-wsl) 中运行的会话无法以这种方式移动。
* 选择已安装的编辑器或文件管理器，可在其中打开该会话在磁盘上的文件夹。

<h3 id="control-which-sessions-appear-on-your-other-devices">
  控制哪些会话显示在您的其他设备上
</h3>

本地会话在 [Remote Control](/docs/zh-CN/remote-control) 将其连接后，会显示在您的其他设备上。已连接的会话会出现在 [claude.ai/code](https://claude.ai/code) 的会话列表中，以及登录了您 claude.ai 账户的设备上的 Claude 应用中。

本地会话会在您为其打开 Remote Control 时连接，或在启动时自动连接：

* **您为该会话打开它**：使用该会话的 **Remote Control** 开关，或在其输入框中输入 `/remote-control`。
* **在启动时连接**：当 **Settings > Claude Code** 中的 **Connect new sessions to Remote Control** 处于打开状态时，新会话会自动连接。如果您从未更改过该设置，Desktop 会遵循您的用户设置或托管设置中的 [`remoteControlAtStartup`](/docs/zh-CN/settings-reference#remotecontrolatstartup)，其次是您组织的默认值。

要查看会话是否已连接，请查看工具栏中会话标题前的笔记本电脑图标。会话已连接或正在连接时，该图标会高亮显示。点击它可打开该会话的 **Remote Control** 开关。

要让会话不出现在您的其他设备上，请在所需的级别关闭 Remote Control：

* **单个会话**：关闭其 **Remote Control** 开关。在启动时已自动连接的会话中，输入 `/remote-control` 会保持 Remote Control 打开，并显示 `Remote Control is already on. This session connected automatically when it started.`。点击该行上的 **Turn off** 即可断开连接。
* **此计算机上的新 Desktop 会话**：在 **Settings > Claude Code** 中关闭 **Connect new sessions to Remote Control**。如果它已显示为关闭，请先将其打开再关闭，以便 Desktop 保存您的选择。保存后，它优先于 `remoteControlAtStartup` 和默认值。
* **此计算机上的任何会话，包括 CLI**：在 `~/.claude/settings.json` 中将 [`disableRemoteControl`](/docs/zh-CN/settings-reference#disableremotecontrol) 设置为 `true`，以阻止会话连接。保存该文件时已连接的会话会保持连接，直到您为其关闭 Remote Control。

要隐藏已显示在您其他设备上的会话，请在 Desktop 中将其存档。Desktop 也会存档该会话的 Remote Control 副本，使其从这些设备上的默认会话列表中移除。要在那里查看或删除它，请参阅[存档会话](/docs/zh-CN/claude-code-on-the-web#archive-sessions)。

<h3 id="sessions-from-dispatch">
  来自 Dispatch 的会话
</h3>

[Dispatch](https://support.claude.com/en/articles/13947068) 是一个与 Claude 的持久对话，位于 [Cowork](https://claude.com/product/cowork) 选项卡中。您向 Dispatch 发送任务消息，由它决定如何处理。

任务可以通过两种方式成为 Code 会话：您直接提出要求，例如"打开一个 Claude Code 会话并修复登录错误"；或者 Dispatch 判断该任务属于开发工作，并自行创建一个会话。通常会路由到 Code 的任务包括修复错误、更新依赖、运行测试或创建 Pull Request。研究、文档编辑和电子表格工作则保留在 Cowork 中。

无论哪种方式，Code 会话都会出现在 Code 选项卡的侧边栏中，并带有 **Dispatch** 徽章。当它完成或需要您批准时，您会在手机上收到推送通知。

如果您启用了[计算机使用](#let-claude-use-your-computer)，由 Dispatch 创建的 Code 会话也可以使用它。这些会话中的应用批准会在 30 分钟后过期并重新提示，而不是像常规 Code 会话那样在整个会话期间有效。

有关设置、配对和 Dispatch 设置，请参阅 [Dispatch 帮助文章](https://support.claude.com/en/articles/13947068)。Dispatch 需要 Pro 或 Max 计划，Team 或 Enterprise 计划不可用。

Dispatch 是您离开终端时与 Claude 协作的几种方式之一。有关与其他选项的比较，请参阅[平台和集成](/docs/zh-CN/platforms#work-when-you-are-away-from-your-terminal)。

<h2 id="extend-claude-code">
  扩展 Claude Code
</h2>

连接外部服务、添加可重用工作流、自定义 Claude 的行为并配置预览服务器。要在一个地方管理连接器、skills 和插件，请点击侧边栏中的**自定义**。[Cowork](https://claude.com/product/cowork) 标签页在桌面应用中从此自定义配置获取其 skills、插件和连接器，该配置通过你的 claude.ai 账户同步，而不是从 CLI 的 `~/.claude` 目录。

Claude Code 还会在你使用同一账户登录的终端会话中加载为你的 claude.ai 账户启用的 skills 和插件。请参阅[从 claude.ai 同步的 Skills](/docs/zh-CN/skills#how-synced-skills-behave) 和[从 claude.ai 同步的插件](/docs/zh-CN/plugins/loading#synced-plugins)。

<h3 id="connect-external-tools">
  连接外部工具
</h3>

对于本地和 [SSH](#ssh-sessions) 会话，点击提示框旁的 **+** 按钮并选择 **Connectors** 来添加集成，如 Google Calendar、Slack、GitHub、Linear、Notion 等。你可以在会话之前或期间添加连接器。**+** 按钮在云会话或 WSL 会话中不可用，但 [routines](/docs/zh-CN/routines) 在 routine 创建时配置连接器。

要管理或断开连接器，请在桌面应用中转到设置 → Connectors，或从提示框中的 Connectors 菜单中选择 **Manage connectors**。

连接后，Claude 可以读取你的日历、发送消息、创建问题并直接与你的工具交互。你可以询问 Claude 在你的会话中配置了哪些连接器。

连接器是[MCP servers](/docs/zh-CN/mcp)，具有图形设置流程。使用它们快速与支持的服务集成。对于连接器中未列出的集成，通过[设置文件](/docs/zh-CN/mcp#installing-mcp-servers)手动添加 MCP servers。你也可以[创建自定义连接器](https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp)。

<h3 id="use-skills">
  使用 skills
</h3>

[Skills](/docs/zh-CN/skills)扩展 Claude 可以做的事情。Claude 在相关时自动加载它们，或者你可以直接调用一个：在提示框中输入 `/` 或点击 **+** 按钮并选择 **Slash commands** 来浏览可用的内容。这包括[内置命令](/docs/zh-CN/commands)、你的[自定义 skills](/docs/zh-CN/skills#create-your-first-skill)、来自你的代码库的项目 skills 以及来自任何[已安装插件](/docs/zh-CN/plugins/install)的 skills。选择一个，它会在输入字段中突出显示。在它之后输入你的任务并照常发送。

你可以在 Claude 工作时发送命令，就像任何其他消息一样，会话在轮次完成后返回空闲状态。在 v2.1.206 之前，在轮次中间发送的命令可能会导致会话显示为运行状态，你之后发送的消息未被传递。

本地会话从 `~/.claude/skills/` 加载你的个人 skills。[SSH](#ssh-sessions) 会话从远程主机的主目录读取 `~/.claude/skills/`，而不是从你的机器。

本地和云会话也加载为你的 claude.ai 账户启用的 skills，除非你的组织设置了 [`disableSideloadFlags`](/docs/zh-CN/settings-reference#disablesideloadflags)。云会话改为加载它们，而不是 `~/.claude/skills/`，如[Cowork 和云会话中的 Skills](/docs/zh-CN/skills#skills-in-cowork-and-cloud-sessions)所述。

<h3 id="install-plugins">
  安装插件
</h3>

[Plugins](/docs/zh-CN/plugins/overview)是可重用的包，为 Claude Code 添加 skills、agents、hooks、MCP servers 和 LSP 配置。你可以从桌面应用安装插件，而无需使用终端。

对于本地和 [SSH](#ssh-sessions) 会话，点击提示框旁的 **+** 按钮并选择 **Plugins** 来查看你已安装的插件及其 skills。要添加插件，从子菜单中选择 **Add plugin** 来打开插件浏览器，它显示来自你配置的[市场](/docs/zh-CN/plugins/overview)的可用插件，包括官方 Anthropic 市场。选择 **Manage plugins** 来启用、禁用或卸载插件。

你可以将插件限定到你的用户账户、特定项目或仅本地。如果你的组织集中管理插件，这些插件在桌面会话中的可用方式与在 CLI 中相同，除了桌面应用在 [`disableSideloadFlags`](/docs/zh-CN/settings-reference#disablesideloadflags) 下扣留的那些。

插件浏览器在云会话中不可用，从桌面应用安装的插件不可用于云会话。云会话也不会安装存储库的 `.claude/settings.json` 声明的插件，如[从你的设置中继承的内容](/docs/zh-CN/cloud-environments#what-carries-over-from-your-setup)所述。插件在 WSL 会话中不可用。有关完整的插件参考，包括创建你自己的插件，请参阅 [plugins](/docs/zh-CN/plugins/overview)。

<h3 id="configure-preview-servers">
  配置预览服务器
</h3>

Claude 自动检测你的开发服务器设置并将配置存储在启动会话时选择的文件夹根目录的 `.claude/launch.json` 中。Preview 使用此文件夹作为其工作目录，因此如果你选择了父文件夹，具有自己开发服务器的子文件夹将不会自动检测。要使用子文件夹的服务器，要么直接在该文件夹中启动会话，要么手动添加配置。

要自定义服务器的启动方式，例如使用 `yarn dev` 而不是 `npm run dev`，或更改端口，请编辑 `.claude/launch.json`。该文件支持带注释的 JSON。

```json theme={null}
{
  "version": "0.0.1",
  "configurations": [
    {
      "name": "my-app",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 3000
    }
  ]
}
```

你可以定义多个配置来从同一项目运行不同的服务器，例如前端和 API。请参阅下面的[示例](#examples)。

<h4 id="auto-verify-changes">
  自动验证更改
</h4>

启用 `autoVerify` 时，Claude 在编辑文件后自动验证代码更改。它拍摄屏幕截图、检查错误并在完成响应之前确认更改有效。

自动验证默认开启。可通过在 `.claude/launch.json` 中添加 `"autoVerify": false` 来按项目禁用它，或在 Browser 窗格的 **⋮** 菜单中关闭 **Auto-verify changes**。

```json theme={null}
{
  "version": "0.0.1",
  "autoVerify": false,
  "configurations": [
    {
      "name": "my-app",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 3000
    }
  ]
}
```

禁用时，预览工具仍然可用，你可以随时要求 Claude 验证。自动验证使其在每次编辑后自动进行。

<h4 id="configuration-fields">
  配置字段
</h4>

`configurations` 数组中的每个条目接受以下字段：

| 字段 | 类型 | 描述 |
| - | - | - |
| `name` | string | 此服务器的唯一标识符 |
| `runtimeExecutable` | string | 要运行的命令，例如 `npm`、`yarn` 或 `node` |
| `runtimeArgs` | string\[] | 传递给 `runtimeExecutable` 的参数，例如 `["run", "dev"]` |
| `port` | number | 你的服务器监听的端口。默认为 3000 |
| `cwd` | string | 相对于你的项目根目录的工作目录。默认为项目根目录。使用 `${workspaceFolder}` 显式引用项目根目录 |
| `env` | object | 其他环境变量作为键值对，例如 `{ "NODE_ENV": "development" }`。不要在这里放置秘密，因为此文件被提交到你的存储库。要将秘密传递给你的开发服务器，在[本地环境编辑器](#local-sessions)中设置它们。 |
| `autoPort` | boolean | 如何处理端口冲突。请参阅[端口冲突](#port-conflicts) |
| `program` | string | 用 `node` 运行的脚本。请参阅[何时使用 `program` vs `runtimeExecutable`](#when-to-use-program-vs-runtimeexecutable) |
| `args` | string\[] | 传递给 `program` 的参数。仅在设置 `program` 时使用 |
| `url` | string | preview 打开的地址，而不是 `http://localhost:<port>`。请参阅[在特定 URL 打开 preview](#open-the-preview-at-a-specific-url) |

<a id="when-to-use-program-vs-runtimeexecutable" />

<h5 id="when-to-use-program-vs-runtimeexecutable">
  何时使用 `program` vs `runtimeExecutable`
</h5>

使用 `runtimeExecutable` 和 `runtimeArgs` 通过包管理器启动开发服务器。例如，`"runtimeExecutable": "npm"` 和 `"runtimeArgs": ["run", "dev"]` 运行 `npm run dev`。

当你有一个想用 `node` 直接运行的独立脚本时，使用 `program`。例如，`"program": "server.js"` 运行 `node server.js`。使用 `args` 传递其他标志。

<a id="open-the-preview-at-a-specific-url" />

<h5 id="open-the-preview-at-a-specific-url">
  在特定 URL 打开 preview
</h5>

默认情况下，preview 打开 `http://localhost:<port>`。当你的服务器需要不同的地址时，设置 `url`。常见情况是需要本地 HTTPS 的服务器、使用 `*.localhost` 子域的应用以及通过重定向登录你的应用。

```json theme={null}
{
  "version": "0.0.1",
  "configurations": [
    {
      "name": "my-app",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 8443,
      "url": "https://localhost:8443"
    }
  ]
}
```

Localhost 地址直接打开，完全像默认端口地址一样。这包括 `localhost`、任何 `*.localhost` 子域、`127.0.0.1` 和 `::1`。出于安全考虑，localhost `url` 必须仅是你的服务器的源 — 没有路径或查询，端口必须与条目的端口匹配。要显示特定页面，在 preview 打开后要求 Claude 导航到那里。带有路径、查询或不匹配端口的 localhost `url` 被报告为配置错误，该错误命名 url 并显示修复。

对于任何其他地址，Desktop 在 preview 首次打开它时要求你的许可，就像你在 preview 中浏览到新网站时一样。外部地址可能包括路径。选择**始终允许**以在将来跳过该网站的提示。限制 preview 中外部网站的组织策略仍然适用。

要预览你已经自己运行的服务器，设置 `url` 而不设置命令。Claude 将 preview 附加到你的运行服务器，而不是启动一个：

```json theme={null}
{
  "version": "0.0.1",
  "configurations": [
    {
      "name": "my-app",
      "url": "https://app.localhost:3000"
    }
  ]
}
```

`url` 必须是 `http` 或 `https`，且不能包含用户名或密码。

<h4 id="port-conflicts">
  端口冲突
</h4>

`autoPort` 字段控制当你的首选端口已在使用时会发生什么：

* **`true`**：Claude 自动查找并使用空闲端口。适合大多数开发服务器。
* **`false`**：Claude 失败并出现错误。当你的服务器必须使用特定端口时使用此选项，例如 OAuth 回调或 CORS 允许列表。
* **未设置（默认）**：Claude 询问服务器是否需要该确切端口，然后保存你的答案。

当 Claude 选择不同的端口时，它通过 `PORT` 环境变量将分配的端口传递给你的服务器。

<h4 id="examples">
  示例
</h4>

这些配置显示了不同项目类型的常见设置：

<Tabs>
  <Tab title="Next.js">
    此配置使用 Yarn 在端口 3000 上运行 Next.js 应用：

    ```json theme={null}
    {
      "version": "0.0.1",
      "configurations": [
        {
          "name": "web",
          "runtimeExecutable": "yarn",
          "runtimeArgs": ["dev"],
          "port": 3000
        }
      ]
    }
    ```
  </Tab>

  <Tab title="多个服务器">
    对于具有前端和 API 服务器的 monorepo，定义多个配置。前端使用 `autoPort: true`，因此如果 3000 被占用，它会选择空闲端口，而 API 服务器需要端口 8080：

    ```json theme={null}
    {
      "version": "0.0.1",
      "configurations": [
        {
          "name": "frontend",
          "runtimeExecutable": "npm",
          "runtimeArgs": ["run", "dev"],
          "cwd": "apps/web",
          "port": 3000,
          "autoPort": true
        },
        {
          "name": "api",
          "runtimeExecutable": "npm",
          "runtimeArgs": ["run", "start"],
          "cwd": "server",
          "port": 8080,
          "env": { "NODE_ENV": "development" },
          "autoPort": false
        }
      ]
    }
    ```
  </Tab>

  <Tab title="Node.js 脚本">
    要直接运行 Node.js 脚本而不是使用包管理器命令，使用 `program` 字段：

    ```json theme={null}
    {
      "version": "0.0.1",
      "configurations": [
        {
          "name": "server",
          "program": "server.js",
          "args": ["--verbose"],
          "port": 4000
        }
      ]
    }
    ```
  </Tab>
</Tabs>

<h2 id="environment-configuration">
  环境配置
</h2>

你在[启动会话](#start-a-session)时选择的环境决定了 Claude 执行的位置以及你如何连接：

* **Local**：在你的机器上运行，直接访问你的文件
* **Cloud**：在 Anthropic 管理的基础设施上运行。即使你关闭应用，会话也会继续。
* **SSH**：在你通过 SSH 连接的远程机器上运行，例如你自己的服务器、云虚拟机或开发容器
* **WSL**（Windows）：在你的机器上的 [WSL 2 发行版](/docs/zh-CN/desktop-wsl)内运行，使用其 Linux 工具链和本地路径

<h3 id="local-sessions">
  本地会话
</h3>

桌面应用并不总是继承你的完整 shell 环境。在 macOS 上，当你从 Dock 或 Finder 启动应用时，它读取你的 shell 配置文件，例如 `~/.zshrc` 或 `~/.bashrc`，来提取 `PATH` 和一组固定的 Claude Code 变量，但你在那里导出的其他变量不会被拾取。在 Windows 上，应用继承用户和系统环境变量，但不读取 PowerShell 配置文件。

要在任何平台上为本地会话和开发服务器设置环境变量，在提示框中打开环境下拉菜单，将鼠标悬停在 **Local** 上，然后点击齿轮图标来打开本地环境编辑器。你在此处保存的变量在你的机器上加密存储，并适用于你启动的每个本地会话和预览服务器。你也可以将变量添加到你的 `~/.claude/settings.json` 文件中的 `env` 键，尽管这些仅到达 Claude 会话而不是开发服务器。有关支持的变量的完整列表，请参阅[环境变量](/docs/zh-CN/env-vars)。

[扩展思考](/docs/zh-CN/model-config#extended-thinking)默认启用，这改进了复杂推理任务的性能，但会使用额外的 token。在 Anthropic API 上，在本地环境编辑器中将 `MAX_THINKING_TOKENS` 设置为 `0` 来关闭思考；这对 Opus 5.5、Sonnet 5.5、Haiku 5.5 或 Fable 模型没有影响，它们始终使用扩展思考。在 Anthropic API 上关闭思考后，Claude Code 会向它已知[不接受该组合](/docs/zh-CN/errors#effort-isnt-available-with-thinking-turned-off)的模型（例如 Opus 5）发送努力级别 `high`，而不是更高级别。

在具有[自适应推理](/docs/zh-CN/model-config#adjust-effort-level)的模型上，对于为正数的 `MAX_THINKING_TOKENS` 值，Claude Code 会忽略该数值本身，因为思考深度改由自适应推理控制。在 Opus 4.6 和 Sonnet 4.6 上，将 `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` 设置为 `1` 可使用固定思考预算；Fable 模型、Sonnet 5 及更高版本、Haiku 5.5 以及 Opus 4.7 及更高版本始终使用自适应推理，没有固定预算模式。

<h4 id="local-sessions-on-managed-devices">
  托管设备上的本地会话
</h4>

你的管理员可以使用 [`disableDesktopLocalSessions` 托管设置](#managed-settings)关闭本地会话。当他们这样做时，**Local** 保留在环境下拉菜单中但被灰显，无法选择，并显示一个工具提示说你的组织关闭了它，在 Windows 上 [WSL](/docs/zh-CN/desktop-wsl) 条目（其在托管设备上的可用性[单独管理](/docs/zh-CN/admin-setup#wsl-sessions-in-claude-code-desktop)）也以相同方式灰显。新会话默认为第一个 SSH 连接（如果已配置），如果你尝试继续现有会话，Desktop 会显示一条消息说此设备上不可用本地会话。选择 [SSH](#ssh-sessions) 或 [cloud](#cloud-sessions) 环境，或联系你的 IT 团队。

<h3 id="cloud-sessions">
  云会话
</h3>

云会话即使在你关闭应用后也会在后台继续。使用计入你的[订阅计划限制](/docs/zh-CN/costs)，没有单独的计算费用。

你可以创建具有不同网络访问级别和环境变量的自定义云环境。要管理它们，在提示框中打开环境下拉菜单并选择 **Cloud**：

* **添加环境**：选择 **Add cloud environment**
* **编辑或存档你自己的环境之一**：将鼠标悬停在它上面并点击齿轮图标

有关配置网络访问和环境变量的详细信息，请参阅[配置云环境](/docs/zh-CN/cloud-environments)。

<h3 id="ssh-sessions">
  SSH 会话
</h3>

SSH 会话让你在远程机器上运行 Claude Code，同时使用桌面应用作为你的界面。这对于使用存在于云虚拟机、开发容器或具有特定硬件或依赖项的服务器上的代码库很有用。

要添加 SSH 连接，请在启动会话之前在输入框中打开环境下拉菜单，然后选择 **SSH > Add SSH connection…** 并填写连接详细信息：

* **Name**：此连接的友好标签
* **SSH host**：`user@hostname` 或在 `~/.ssh/config` 中定义的主机
* **SSH port**：如果留空，则默认为 22，或使用您 SSH 配置中的端口
* **SSH key (optional)**：您的私钥路径，例如 `~/.ssh/id_ed25519`。留空则使用您的 SSH 配置或 SSH agent。

添加后，该连接会出现在环境下拉菜单的 **SSH** 下。选择它即可在该机器上启动会话。Claude 在远程机器上运行，可以访问其文件和工具。

远程机器必须运行 Linux 或 macOS。Desktop 在你第一次连接时会自动在远程机器上安装 Claude Code。连接后，SSH 会话支持权限模式、connectors、plugins 和 MCP servers。

<h4 id="open-an-ssh-session-from-a-link">
  通过链接打开 SSH 会话
</h4>

`claude://code/new` 链接会打开 Desktop 的新会话页面，并且可以指定一个 SSH 连接。将此类链接放入运维手册、仪表板或 wiki 页面中，即可打开已为正确的机器和文件夹设置好的 Desktop。对于会去除此类链接的平台，请参阅[链接显示为纯文本而不可点击](/docs/zh-CN/deep-links#the-link-renders-as-plain-text-instead-of-being-clickable)。

SSH 链接需要 Claude Desktop v2.110.0 或更高版本。

以下链接指定了 `build.example.com` 上的用户 `dev`、端口 2222 和文件夹 `/srv/payments`，并填入一条提示词：

```text theme={null}
claude://code/new?ssh_host=dev%40build.example.com&ssh_port=2222&ssh_folder=/srv/payments&q=Investigate%20the%20failed%20deploy
```

SSH 链接接受以下参数，其中只有 `ssh_host` 是必需的：

| 参数 | 值 |
| :- | :- |
| `ssh_host` | `host` 或 `user@host`，写法与 **SSH host** 字段相同。该值不能以 `-` 开头，主机部分只能包含字母、数字、`.`、`_`、`:` 和 `-` |
| `ssh_port` | 1 到 65535 之间的端口号 |
| `ssh_folder` | 远程机器上的文件夹。以 `/` 或 `~/` 开头，或使用 `~` |
| `q` | 用于输入框的 URL 编码文本 |

`~/.ssh/config` 中的别名仅对拥有该条目的人可用作 `ssh_host`。要匹配用户已有的连接，请在链接中使用与该连接相同的用户、主机和端口。

当您打开链接时，Desktop 会在选择连接之前要求您确认：

* **您已有的连接**：如果主机、用户和端口与您的某个连接匹配，Desktop 会询问是否使用它，并向您显示该连接的名称和主机，如果链接指定了文件夹，还会显示文件夹。
* **新连接**：否则，Desktop 会打开用于添加 SSH 连接的对话框。当您添加连接时，Desktop 会在保存任何内容之前询问是否连接，并向您显示链接中的主机，如果链接指定了端口和文件夹，还会显示它们。

在您确认之前，Desktop 不会保存链接中的主机、端口或文件夹，也不会使用它们选择或打开连接。如果已经选择了某个 SSH 连接，新会话页面仍可以像没有链接时一样自行连接到该连接，即使链接指定了相同的主机也是如此。链接不能携带密钥文件、密码或命令。

任何人都可以编写链接，因此请检查它填入的内容：

* **确认之前**：检查主机和文件夹。
* **发送之前**：检查提示词和所选环境。

Desktop 会在链接打开时填入提示词，替换您尚未发送的任何文本，并且绝不会替您发送。它将提示词视为纯文本，因此开头的 `/` 或 `!` 以及 `@` 文件提及不会作为命令或提及生效。如果您取消，提示词会保留在输入框中，您之前选择的环境也不会改变。

链接不会绕过 [`sshHostAllowlist`](#restrict-which-ssh-hosts-users-can-connect-to)。Desktop 会在连接时检查允许列表。

如果链接打开了 Desktop 却没有出现关于连接的对话框，请检查是否存在以下原因之一：

* **您已退出登录**：请登录，然后再次打开链接。
* **另一个对话框处于打开状态**：关闭它，然后再次打开链接。
* **链接无效**：Desktop 会显示一条消息说明需要修正的内容，并且不会填入提示词。
* **Desktop 版本早于 v2.110.0**：早期版本会忽略 SSH 参数，仅带着提示词打开新会话页面。
* **SSH 会话已关闭**：如果您的管理员将允许列表设置为空数组，Desktop 会拒绝 SSH 链接。

<h4 id="pre-configure-ssh-connections-for-your-team">
  为你的团队预配置 SSH 连接
</h4>

管理员可以通过在[托管设置](/docs/zh-CN/managed-settings)中设置 `sshConfigs` 来向团队成员分发 SSH 连接。以这种方式定义的连接会自动出现在每个用户的环境下拉菜单中，并显示为托管的，因此用户可以选择它们，但不能在应用中编辑或删除它们。

以下示例预配置了一个单个连接：

```json theme={null}
{
  "sshConfigs": [
    {
      "id": "shared-dev-vm",
      "name": "Shared Dev VM",
      "sshHost": "user@dev.example.com",
      "sshPort": 22,
      "sshIdentityFile": "~/.ssh/id_ed25519"
    }
  ]
}
```

每个条目需要 `id`、`name` 和 `sshHost`。`sshPort` 和 `sshIdentityFile` 字段是可选的。用户也可以将 `sshConfigs` 添加到他们自己的 `~/.claude/settings.json`。

<h4 id="restrict-which-ssh-hosts-users-can-connect-to">
  限制用户可以连接的 SSH 主机
</h4>

管理员可以通过在[托管设置](/docs/zh-CN/managed-settings)中设置 `sshHostAllowlist` 来将 Desktop 的 SSH 会话限制为一组已批准的主机。设置后，用户只能连接到其解析的主机名与其中一个模式匹配的主机。将其设置为空数组可禁用 SSH 会话。[`sshHostAllowlist` 参考条目](/docs/zh-CN/settings-reference#sshhostallowlist)说明了空数组如何与其他托管来源中的列表组合。

以下示例允许连接到 `devboxes.example.com` 下的任何主机以及单个命名的堡垒主机：

```json theme={null}
{
  "sshHostAllowlist": ["*.devboxes.example.com", "bastion.example.com"]
}
```

<Warning>
  如果您的组织提供[服务器托管设置](/docs/zh-CN/server-managed-settings)，请在那里设置 `sshHostAllowlist`。默认情况下，Desktop 仅从[提供策略键的最高优先级托管来源](/docs/zh-CN/managed-settings#how-claude-code-combines-managed-sources)读取该键。如果该来源未设置该键，Desktop 会忽略较低优先级的 MDM 策略或托管设置文件中的列表，并将该键视为[未设置](/docs/zh-CN/settings-reference#sshhostallowlist)。Desktop 不会显示任何警告。

  同时，请在每个用户的机器上保留相同的列表，放在该机器上最高优先级的 MDM 策略或托管设置文件中。Desktop 在启动时获取服务器托管设置，并且不保留缓存副本，因此在获取成功之前，适用的是该机器上的列表。
</Warning>

模式不区分大小写。`*` 匹配任何主机，`*.example.com` 匹配 `example.com` 和任何子域。其他任何内容都是精确匹配。检查针对通过 `ssh -G` 进行 `~/.ssh/config` 解析后的主机名运行，因此允许 `Host` 别名和 `ProxyCommand`/`ProxyJump` 条目，只要解析的 `HostName` 匹配。

`sshHostAllowlist` 仅从托管设置中读取；用户或项目设置中的值被忽略。只有 Claude Desktop 应用遵守此设置；Claude Code CLI 和 IDE 扩展不读取它，它也不限制通过 Bash 工具运行的 `ssh` 命令。它管理 Desktop 应用连接到的主机，而不是网络出口，因此如果你需要硬边界，请将其与你的组织的网络或零信任控制配对。

<h2 id="enterprise-configuration">
  企业配置
</h2>

Team 或 Enterprise 计划上的组织可以通过管理员控制台控制、托管设置文件和设备管理策略来管理桌面应用行为。

<h3 id="admin-console-controls">
  管理员控制台控制
</h3>

这些设置通过[管理员设置控制台](https://claude.ai/admin-settings/claude-code)配置：

* **Desktop**：控制您的组织中的用户是否可以在桌面应用中访问 Claude Code
* **Cloud sessions**：为您的组织启用或禁用[云端会话](/docs/zh-CN/claude-code-on-the-web)
* **Remote Control**：为您的组织启用或禁用 [Remote Control](/docs/zh-CN/remote-control)

在启用了 HIPAA 的 Enterprise 组织中，**Desktop** 开关默认关闭，[Owner](/docs/zh-CN/server-managed-settings#access-control) 可以将其打开。应用 [HIPAA 配置](/docs/zh-CN/hipaa-setup)会将其关闭（即使之前已打开），因此 Owner 之后必须重新将其打开。**Cloud sessions** 和 **Remote Control** 也默认关闭，并且一旦组织应用了 HIPAA 配置，Owner 就无法将它们打开。

<Note>
  Cowork 下的 OpenTelemetry 表单位于管理员控制台的[数据和隐私设置](https://claude.ai/admin-settings/data-privacy-controls)中的**监控**下，仅适用于 Cowork 会话。在此机器上的 Cowork 会话中，桌面应用将该收集器作为 `OTEL_*` 环境变量传递给 Claude Code，因此该表单生效，尽管该会话中的 Claude Code [从不获取管理员控制台设置](#managed-settings)。

  要从 Code 选项卡会话导出遥测，请在 Claude Code 托管设置的 `env` 块中设置 `CLAUDE_CODE_ENABLE_TELEMETRY` 和 `OTEL_*` 变量，如[监控的管理员配置](/docs/zh-CN/monitoring-usage#administrator-configuration)中所示。本地、云端和 SSH 会话各自[从不同来源读取托管设置](#managed-settings)。有关云端会话可以到达的主机，请参阅[网络访问](/docs/zh-CN/cloud-environments#network-access)。有关 Code 选项卡会话报告的 `service.name`，请参阅[服务信息](/docs/zh-CN/monitoring-usage#service-information)。
</Note>

<h3 id="managed-settings">
  托管设置
</h3>

托管设置覆盖项目和用户设置，并应用于 Desktop 中的 Claude Code 会话。您可以在您的组织的[托管设置](/docs/zh-CN/managed-settings)文件中设置这些键，或通过管理员控制台远程推送它们。

| 键 | 描述 |
| - | - |
| `permissions.disableBypassPermissionsMode` | 设置为 `"disable"` 以防止用户启用绕过权限模式。 |
| `disableAutoMode` | 设置为 `"disable"` 以从模式选择器中删除 [Auto](/docs/zh-CN/permission-modes#eliminate-prompts-with-auto-mode) 模式。也在 `permissions` 下接受。 |
| `autoMode` | 自定义自动模式分类器在您的组织中信任和阻止的内容。请参阅[配置自动模式](/docs/zh-CN/auto-mode-config)。 |
| `browserExternalPageTools` | 设置为 `"disabled"` 以防止 Claude 使用工具在[浏览器窗格](#browse-external-sites)中读取或作用于外部页面。用户仍然可以自己导航到外部网站，本地开发服务器预览不受影响。 |
| `disableMobileSimulatorTools` | 设置为 `true` 以阻止 Claude 在 [iOS Simulator 窗格](/docs/zh-CN/desktop-ios-simulator#turn-off-simulator-access)中控制和捕获设备的工具。该窗格仍可用于用户自己的点击；仅删除 Claude 的访问权限。该值必须是 JSON 布尔值 `true`；字符串 `"true"` 被忽略。 |
| `disableBrowserExternalNavigation` | 设置为 `true` 以完全关闭[浏览器窗格](#browse-external-sites)中的外部浏览。用户和 Claude 都无法导航到外部网站，localhost 开发服务器预览不受影响。该值必须是 JSON 布尔值 `true`；字符串 `"true"` 被忽略。 |
| `sshConfigs` | 预配置[SSH 连接](#pre-configure-ssh-connections-for-your-team)，在环境下拉菜单中显示。用户无法编辑或删除托管连接。 |
| `sshHostAllowlist` | 限制 [SSH 会话](#restrict-which-ssh-hosts-users-can-connect-to)连接到已解析主机名与这些模式之一匹配的主机。仅从托管设置中读取。 |
| `disableDesktopLocalSessions` | 设置为 `true` 以关闭[在设备上运行的 Code 会话](#local-sessions-on-managed-devices)，仅保留到其他主机的 SSH 会话和云端会话可用。该值必须是 JSON 布尔值 `true`。仅从托管设置中读取。需要 Claude Desktop v1.37937.0 或更高版本。 |
| `disableSshSavedPasswords` | 设置为 `true` 以阻止 Desktop 提供记住 SSH 密码的选项，并阻止其使用或显示之前保存的密码。启用此设置不会删除这些密码。仅从托管设置中读取。需要 Claude Desktop v1.49585.0 或更高版本。 |
| `managedMcpServers` | 将 MCP 服务器配置推送到所有用户。仅在第三方 (3P) Desktop 部署中可用。在每个条目中，设置 `"http"`、`"sse"` 或 `"stdio"` 的传输、连接详细信息，以及可选的 `toolPolicy` 映射，该映射限制该服务器中用户可以调用的工具。通过托管设置文件、MDM 或 Claude apps gateway 策略的 [`desktop` 块](/docs/zh-CN/claude-apps-gateway-config#claude-desktop-overlay)提供它，因为 3P 部署不接收管理员控制台设置。要通过网关提供它，您需要网关服务器上的 Claude Code v2.1.232 或更高版本。这是桌面应用自己的键；Claude Code 读取自己的[同名托管设置](/docs/zh-CN/managed-mcp#provide-servers-through-managed-settings)，具有不同的条目形状。 |

哪些托管设置到达 Desktop 会话取决于该会话运行的位置。模型限制（如 [`availableModels`](/docs/zh-CN/model-config#restrict-model-selection)）在 Desktop 的 Claude Code 会话中的执行方式与在终端 CLI 中相同；请参阅[使用入口覆盖范围](/docs/zh-CN/model-config#surface-coverage)。

* **此机器上的本地会话**：部署到磁盘的托管设置文件适用。通过管理员控制台远程推送的托管设置也在会话使用[符合条件的登录](/docs/zh-CN/server-managed-settings#platform-availability)向 Anthropic 的 API 进行身份验证时到达这些会话，遵循与终端 CLI 相同的[设置优先级](/docs/zh-CN/settings#settings-precedence)。
* **[云端会话](#cloud-sessions)**：接收[服务器管理的设置](/docs/zh-CN/server-managed-settings)；设备部署的文件无法到达它们，因为它们在 Anthropic 管理的虚拟机上运行。路由到[自托管环境](/docs/zh-CN/self-hosted-environments)的会话也读取运行程序镜像中的托管设置文件。[Claude Code 如何组合托管源](/docs/zh-CN/managed-settings#how-claude-code-combines-managed-sources)说明该文件何时适用。
* **[SSH 会话](#ssh-sessions)**：会话从远程主机读取托管设置文件。Desktop 本身在本地机器上读取 `sshConfigs`、`sshHostAllowlist`、`disableSshSavedPasswords` 和 `disableDesktopLocalSessions`。如果您提供多个托管源，它[默认只从其中一个](/docs/zh-CN/managed-settings#how-claude-code-combines-managed-sources)读取。
* **[Cowork](https://claude.com/docs/cowork/overview) 会话**：在此机器上的 Cowork 会话中，Claude Code 永远不会获取管理员控制台设置，即使用户使用 Team 或 Enterprise 帐户登录，并读取部署到机器的策略，除非您的 Claude Desktop 配置设置了 `requireCoworkFullVmSandbox`。远程 Cowork 会话两者都不接收。请参阅[策略应用的位置和时间](/docs/zh-CN/managed-settings#where-and-when-a-policy-applies)了解哪些设备文件到达 Cowork，以及[MCP 权限规则](/docs/zh-CN/permissions#mcp)了解 `Bash` 和 `WebFetch` 规则如何应用于 Cowork 的工具。

在本地和 SSH 会话中，桌面应用直接将每个用户连接的 claude.ai 连接器传递给 Claude Code。无论您使用哪个设置源或文件位置，都没有 MCP 设置或 `managed-mcp.json` 到达这些连接器。要在这些会话中阻止连接器的工具，请使用您的组织的[连接器工具控制](/docs/zh-CN/mcp#organization-controls-on-connector-tools)。[连接器如何到达 Claude Code](/docs/zh-CN/mcp#how-connectors-reach-claude-code)显示在每种会话中哪些设置管理连接器。

`permissions.disableBypassPermissionsMode` 和 `disableAutoMode` 也在用户和项目设置中工作，但将它们放在托管设置中可防止用户覆盖它们。

有关仅托管源可以设置的权限、插件和交付键，请参阅[仅托管设置可以设置的键](/docs/zh-CN/managed-settings#managed-only-settings)。

<h3 id="device-management-policies">
  设备管理策略
</h3>

IT 团队可以通过 macOS 上的 MDM、Windows 上的组策略或 Linux 上的策略文件管理桌面应用。可用的策略包括启用或禁用 Claude Code 功能、在 macOS 和 Windows 上控制自动更新以及设置自定义部署 URL。

* **macOS**：通过使用 Jamf 或 Kandji 等工具的 `com.anthropic.claudefordesktop` 偏好域配置
* **Windows**：通过 `SOFTWARE\Policies\Claude` 处的注册表配置
* **Linux**：通过位于 `/etc/claude-desktop/managed-settings.json` 的 root 所有的文件配置，该文件以 JSON 对象形式保存策略键。如果除 root 之外的任何人可以写入该文件或其所在文件夹，Desktop 将拒绝使用该文件。它与 Claude Code 的[托管设置文件](/docs/zh-CN/managed-settings)是不同的文件。

<h3 id="network-access-requirements">
  网络访问要求
</h3>

Desktop 从 Anthropic CDN 主机加载其应用程序代码和用户内容。

```text theme={null}
anthropic.com
*.anthropic.com
claude.ai
*.claude.ai
claude.com
*.claude.com
claude.app
*.claude.app
*.claudeusercontent.com
*.claudemcpcontent.com
```

流量在端口 443 上使用 HTTPS，除非您为 [OTLP](/docs/zh-CN/monitoring-usage)、LLM 网关或 MCP 服务器配置自定义端口。

有关代理服务器、自定义证书颁发机构、mTLS 和独立 CLI 需要的域，请参阅[网络配置](/docs/zh-CN/network-config)。

要减少防火墙通配符的数量，请改为允许这些 Anthropic 主机。某些子域是动态生成的，必须保持为通配符。

```text theme={null}
anthropic.com
api.anthropic.com
a-api.anthropic.com
a-cdn.anthropic.com
s-cdn.anthropic.com
assets-proxy.anthropic.com
claude.ai
a.claude.ai
assets.claude.ai
downloads.claude.ai
*.livepreview.claude.ai
claude.com
platform.claude.com
*.livepreview.claude.app
*.claudeusercontent.com
*.claudemcpcontent.com
```

如果您的组织为 Claude 启用了[IP 允许列表](https://support.claude.com/en/articles/13200993-restrict-access-to-claude-with-ip-allowlisting)，请通过与 `claude.ai` 和 `api.anthropic.com` 相同的代理出口路由 `bridge.claudeusercontent.com`。如果您无法以这种方式路由它，请将您的代理用于该主机的出口地址添加到您的组织的 IP 允许列表，但仅当该地址专用于您的组织时：共享代理出口范围也允许代理供应商的其他客户。

Anthropic 根据连接到达的地址检查与该主机的连接是否符合您的组织的 IP 允许列表。如果您的代理通过不在该允许列表上的地址为其发送流量，Chrome 中的 Claude 和通过网桥连接的其他功能将停止工作，而应用的其余部分继续工作。

从 [Google Fonts](/docs/zh-CN/artifacts#improve-the-visual-design) 加载字体的 [Artifact](/docs/zh-CN/artifacts) 也会请求 `fonts.googleapis.com` 和 `fonts.gstatic.com`。两个主机都是可选的。如果您阻止它们，Artifact 将以备用字体呈现。使用快速拒绝而不是静默丢弃来阻止，以便字体请求立即失败，而不是延迟页面的首次呈现。

Artifact 还可以从 `cdnjs.cloudflare.com`、`cdn.jsdelivr.net`、`cdn.tailwindcss.com`、`code.jquery.com` 和 `unpkg.com` 加载 JavaScript 库（如 React 或图表包），而不能从其他任何外部主机加载。如果您阻止这些主机，Artifact 中依赖库的部分将无法工作，与被阻止的字体不同，被阻止的库没有备用方案。这里也使用快速拒绝，以便被阻止的库请求立即失败，而不是挂起直到超时。

<h3 id="authentication-and-sso">
  身份验证和 SSO
</h3>

Team 和 Enterprise 组织可以要求所有用户使用 SSO。有关计划级别的详细信息，请参阅[身份验证](/docs/zh-CN/authentication)，有关 SAML 配置，请参阅[设置 SSO](https://support.claude.com/en/articles/13132885-setting-up-single-sign-on-sso)；OIDC 设置在 [Claude Enterprise Administrator Guide](https://claude.com/resources/tutorials/claude-enterprise-administrator-guide) 中介绍。

<h3 id="data-handling">
  数据处理
</h3>

Claude Code 在本地会话中本地处理您的代码，或在云端会话中在 Anthropic 管理的基础设施上处理，除非您的组织将它们路由到[自托管环境](/docs/zh-CN/self-hosted-environments)。云端会话（包括在自托管环境中）将对话和代码上下文发送到 Anthropic 的 API 进行处理；本地和 SSH 会话将它们发送到您的部署配置的任何[模型提供商](#feature-comparison)，默认为 Anthropic 的 API。有关数据保留、隐私和合规性的详细信息，请参阅[数据处理](/docs/zh-CN/data-usage)。

<h3 id="deployment">
  部署
</h3>

Desktop 可以通过企业部署工具分发：

* **macOS**：通过 MDM（如 Jamf 或 Kandji）使用 `.dmg` 安装程序分发
* **Windows**：通过 MSIX 包部署。有关企业部署选项（包括静默安装），请参阅[为 Windows 部署 Claude Desktop](https://support.claude.com/en/articles/12622703-deploy-claude-desktop-for-windows)

有关需要在防火墙中加入允许列表的域，请参阅上面的[网络访问要求](#network-access-requirements)。有关代理设置、自定义证书颁发机构和 LLM 网关，请参阅[网络配置](/docs/zh-CN/network-config)。

有关完整的企业配置参考，请参阅[企业配置指南](https://support.claude.com/en/articles/12622667-enterprise-configuration)。

<h2 id="coming-from-the-cli">
  来自 CLI？
</h2>

如果您已经使用 Claude Code CLI，Desktop 运行相同的底层引擎，具有图形界面。您可以在同一机器上同时运行两者，甚至在同一项目上。每个维护单独的会话列表，您可以将 CLI 会话带入 Desktop。它们通过 CLAUDE.md 文件共享配置和项目记忆。

要将 CLI 会话移动到 Desktop，在终端中运行 `/desktop`。Claude 保存您的会话并在桌面应用中打开它，然后退出 CLI。当您使用 Claude 订阅登录时，此命令在 macOS 和 x64 Windows 上可用。它不适用于 API 密钥身份验证或 Amazon Bedrock、Google Cloud 的 Agent Platform 或 Microsoft Foundry。

从您的 shell，[`claude --desktop`](/docs/zh-CN/cli-reference#cli-flags) 直接打开 Desktop，无需启动终端会话。它需要 Claude Code v2.1.285 或更高版本，并具有与 `/desktop` 相同的平台和登录要求。没有其他参数时，它在当前目录中打开 Desktop。要在 Desktop 中打开现有的 CLI 会话，添加 `--continue` 以获取此目录中最近的对话，或使用 `--resume` 和 `/status` 显示的会话 ID：

```bash theme={null}
claude --desktop --resume <session-id>
```

Claude Code 打印 `Opening session <session-id> in Claude Desktop`，会话在应用中打开，命令退出。会话名称不能代替 ID。Claude Code 不会移动在另一个终端中打开或仍在后台运行的会话。如果未安装 Claude Desktop，命令会打印下载链接并退出。

您也可以从 Desktop 内部使用 `/resume` 选择 CLI 会话。此命令在本地会话中可用，不在 SSH、WSL 或云端会话中可用。

要在 Desktop 中继续终端会话：

1. 在终端中关闭会话。
2. 在 Desktop 输入框中，输入 `/resume`。Desktop 列出您从 CLI 在此计算机上启动的会话。按标题、文件夹或分支搜索，并预览每个会话的停止位置。
3. 选择会话。它在应用中继续，具有完整的对话和上下文。

Desktop 继续相同的会话而不是副本，所以之后在终端中 `claude --resume` 仍然可以找到它。

<Tip>
  何时使用 Desktop vs CLI：当您想要在一个窗口中管理并行会话、并排排列窗格或可视化审查更改时，使用 Desktop。当您需要脚本、自动化或更喜欢终端工作流时，使用 CLI。
</Tip>

<h3 id="cli-flag-equivalents">
  CLI 标志等效项
</h3>

此表显示了常见 CLI 标志的桌面应用等效项。未列出的标志没有桌面等效项，因为它们是为脚本或自动化设计的。

| CLI | Desktop 等效项 |
| - | - |
| `--model sonnet` | 发送按钮旁的模型下拉菜单 |
| `--resume`, `--continue` | 点击侧边栏中的会话，或在输入框中输入 `/resume` 来选择您从 CLI 启动的会话 |
| `--permission-mode` | 发送按钮旁的模式选择器 |
| `--dangerously-skip-permissions` | 绕过权限模式。在 Pro 和 Max 套餐上，在设置 → Claude Code → "允许绕过权限模式"中启用它；在 Team 和 Enterprise 套餐上，由组织策略控制 |
| `--add-dir` | 在云端会话中使用 **+** 按钮添加多个存储库 |
| `--allowedTools`, `--disallowedTools` | 无每个会话的等效项。[设置文件](/docs/zh-CN/settings)中的权限规则仍然适用。 |
| `--verbose` | [Verbose 视图模式](#switch-view-modes) |
| `--print`, `--output-format` | 不可用。Desktop 仅是交互式的。 |
| `ANTHROPIC_MODEL` 环境变量 | 发送按钮旁的模型下拉菜单 |
| `MAX_THINKING_TOKENS` 环境变量 | 在本地环境编辑器中设置。请参阅[环境配置](#environment-configuration)。 |

<h3 id="shared-configuration">
  共享配置
</h3>

Desktop 和 CLI 读取相同的配置文件，因此您的设置会沿用：

* 项目中的 **[CLAUDE.md](/docs/zh-CN/memory)** 和 `CLAUDE.local.md` 文件由两者共同使用
* 在 `~/.claude.json` 或 `.mcp.json` 中配置的 **[MCP 服务器](/docs/zh-CN/mcp)** 在两者中均可使用
* 在设置中定义的 **[hook](/docs/zh-CN/hooks)** 和 **[skill](/docs/zh-CN/skills)** 适用于两者
* `~/.claude.json` 和 `~/.claude/settings.json` 中的 **[设置](/docs/zh-CN/settings)** 是共享的。权限规则、允许的工具和 `settings.json` 中的其他设置适用于 Desktop 会话。
* **模型**：相同的[模型](/docs/zh-CN/model-config#available-models)在两者中都可用。在 Desktop 中，从发送按钮旁的下拉菜单中选择模型。您可以在会话期间从相同的下拉菜单更改模型。

<h4 id="mcp-servers-from-the-claude-desktop-chat-app">
  来自 Claude Desktop 聊天应用的 MCP 服务器
</h4>

Desktop 应用会将 `claude_desktop_config.json` 中的 MCP 服务器加载到本地 Code 选项卡会话中，与来自 `~/.claude.json` 和 `.mcp.json` 的服务器一起使用。在 `claude_desktop_config.json` 中定义的服务器在 Desktop 聊天使用入口和本地 Code 选项卡会话中都可用。

如果您在 `claude_desktop_config.json` 和 `~/.claude.json` 或 `.mcp.json` 中定义相同的服务器名称，本地会话中的 Code 选项卡只连接一次并使用 `claude_desktop_config.json` 中的定义。

该应用还将来自 `~/.claude.json` 的 stdio 服务器重新传递到本地会话中的嵌入式 CLI。当 `~/.claude.json` 的顶层（用户作用域）和 `.mcp.json` 定义相同的 stdio 服务器名称时，Code 选项卡使用 `~/.claude.json` 中的定义，这与 CLI 的[作用域层次结构](/docs/zh-CN/mcp#scope-hierarchy-and-precedence)不同。

<Note>
  独立 CLI 不读取 `claude_desktop_config.json`。在 macOS 和 WSL 上，运行 `claude mcp add-from-claude-desktop` 将这些服务器复制到 `~/.claude.json`。请参阅[从 Claude Desktop 导入 MCP 服务器](/docs/zh-CN/mcp#import-mcp-servers-from-claude-desktop)了解导入流程和作用域选项。
</Note>

<h3 id="feature-comparison">
  功能比较
</h3>

此表比较了 CLI 和 Desktop 之间的核心功能。有关 CLI 标志的完整列表，请参阅 [CLI 参考](/docs/zh-CN/cli-reference)。

| 功能 | CLI | Desktop |
| - | - | - |
| 权限模式 | 所有模式，包括 `dontAsk` | Manual、Accept edits、Plan 和 Auto。启用后，绕过权限会出现在模式选择器中：在 Pro 和 Max 套餐上通过设置开关启用，在 Team 和 Enterprise 套餐上通过组织策略启用 |
| [第三方提供商](/docs/zh-CN/third-party-integrations) | Amazon Bedrock、Google Cloud 的 Agent Platform、Microsoft Foundry | 默认使用 Anthropic 的 API。对于网关路由，请参阅[将桌面应用连接到网关](/docs/zh-CN/llm-gateway-connect#desktop-app)。要在 Amazon Bedrock、Google Cloud 的 Agent Platform、Microsoft Foundry 或自托管 LLM 网关上运行 Code 选项卡，请参阅 [Claude Desktop on 3P](https://claude.com/docs/third-party/claude-desktop/overview)。 |
| [MCP 服务器](/docs/zh-CN/mcp) | 在设置文件中配置 | 本地和 SSH 会话的连接器 UI，或设置文件 |
| [插件](/docs/zh-CN/plugins/overview) | `/plugin` 命令 | 插件管理器 UI |
| @mention 文件 | 基于文本 | 带自动完成；仅本地和 SSH 会话 |
| 文件附件 | 不可用 | 图像、PDF |
| 会话隔离 | [`--worktree`](/docs/zh-CN/cli-reference) 标志 | 启动会话时的 **worktree** 选项 |
| 多个会话 | 单独的终端 | 侧边栏选项卡 |
| 定期任务 | Cron 作业、CI 管道 | [定时任务](/docs/zh-CN/desktop-scheduled-tasks) |
| 计算机使用 | [在 macOS 上通过 `/mcp` 启用](/docs/zh-CN/computer-use) | 在 macOS 和 Windows 上的[应用和屏幕控制](#let-claude-use-your-computer) |
| iOS 模拟器 | 通过[计算机使用](/docs/zh-CN/computer-use#test-a-simulator-flow)驱动模拟器 | [iOS Simulator 窗格](/docs/zh-CN/desktop-ios-simulator)自动打开 |
| Dispatch 集成 | 不可用 | 侧边栏中的 [Dispatch 会话](#sessions-from-dispatch) |
| 脚本和自动化 | [`--print`](/docs/zh-CN/cli-reference)、[Agent SDK](/docs/zh-CN/headless) | 不可用 |

<h3 id="what’s-not-available-in-desktop">
  Desktop 中不可用的内容
</h3>

以下功能在 Desktop 中不可用，除非另有说明：

* **第三方提供商**：Desktop 默认连接到 Anthropic 的 API。要通过网关路由 Desktop，或在 Amazon Bedrock、Google Cloud 的 Agent Platform、Microsoft Foundry 或自托管 LLM 网关上运行 Code 选项卡，请按照[第三方提供商行](#feature-comparison)中的链接操作。
* **Linux (beta)**：Linux 桌面应用中尚不提供计算机使用。请参阅 [Claude Desktop on Linux](/docs/zh-CN/desktop-linux)。
* **内联代码建议**：Desktop 不提供自动完成风格的代码补全。它通过对话式提示词和显式代码更改工作，并可以在 Claude 回复后[建议您的下一个提示词](#accept-a-suggested-prompt)。
* **Agent team**：协调的团队（由 Claude 作为团队负责人从共享任务列表中为队友分配任务）在 [CLI](/docs/zh-CN/agent-teams) 中可用，不在 Desktop 中。对于一个会话内的多 Agent 工作，请使用[动态工作流](/docs/zh-CN/workflows)，它可在 Desktop 中运行；Claude 也可以直接[向您的其他会话发送消息并管理它们](#work-across-sessions)。
* **Terminal-dialog 命令**：在终端中打开交互式面板的内置命令，其行为在 Code 选项卡中有所不同。直接编辑[设置文件](/docs/zh-CN/settings)来管理权限规则和配置，或从独立 CLI 运行命令。
  * 没有参数形式的命令，例如 `/permissions`，会回复 `isn't available in this environment`。
  * `/config` 打开设置 → Claude Code。命令后的文本会被忽略，所以 `/config theme=dark` 不会设置主题。

<h2 id="troubleshooting">
  故障排除
</h2>

下面的部分涵盖特定于桌面应用的问题。对于出现在聊天中的运行时 API 错误，如 `API Error: 500`、`529 Overloaded`、`429` 或 `Prompt is too long`，请参阅[错误参考](/docs/zh-CN/errors)。这些错误及其修复在 CLI、Desktop 和 Web 中是相同的。

<h3 id="check-your-version">
  检查你的版本
</h3>

要查看你运行的桌面应用版本：

* **macOS**：点击菜单栏中的 **Claude**，然后点击 **About Claude**
* **Windows**：点击 **Help**，然后点击 **About Claude**

点击版本号将其复制到你的剪贴板。

<h4 id="claude-code-version-in-the-code-tab">
  Code 选项卡中的 Claude Code 版本
</h4>

要查看会话运行的 Claude Code 版本，请在 **Code** 选项卡的本地会话中输入 `/status`，然后查看 **Claude Code** 行，其中会显示版本号，例如 `2.1.286`。

要为本地会话获取更新的版本，请在 macOS 上打开 **Claude → Check for Updates**，或在 Windows 上打开 **Help → Check for Updates**，然后启动新会话。

在本地会话中，**Code** 选项卡运行其自己的 Claude Code 副本，该副本有自己的版本号。桌面应用会下载并更新该副本，因此它可能与终端中的 `claude` 命令版本不同，并且更新其中一个不会更新另一个。

<h3 id="403-or-authentication-errors-in-the-code-tab">
  Code 选项卡中的 403 或身份验证错误
</h3>

如果在使用 Code 选项卡时看到 `Error 403: Forbidden` 或其他身份验证失败：

1. 从应用菜单中注销并重新登录。这是最常见的修复。
2. 验证你有活跃的付费订阅：Pro、Max、Team 或 Enterprise。
3. 如果 CLI 工作但 Desktop 不工作，完全退出桌面应用，而不仅仅是关闭窗口，然后重新打开并登录。
4. 检查你的互联网连接和代理设置。

<h3 id="blank-or-stuck-screen-on-launch">
  启动时屏幕空白或卡住
</h3>

如果应用打开但显示空白或无响应的屏幕：

1. 重启应用。
2. 检查待处理的更新。在 macOS 和 Windows 上，应用在启动时自动更新；在 Linux 上，通过 apt 更新，如 [Claude Desktop on Linux](/docs/zh-CN/desktop-linux) 中所述。
3. 在托管网络上，确认你的防火墙允许[网络访问要求](#network-access-requirements)中的 CDN 主机。
4. 在 Windows 上，在 **Windows 日志 → 应用程序** 下的事件查看器中检查崩溃日志。

<h3 id="failed-to-load-session">
  "Failed to load session"
</h3>

如果你看到 `Failed to load session`，选定的文件夹可能不再存在，Git 存储库可能需要未安装的 Git LFS，或文件权限可能阻止访问。尝试选择不同的文件夹或重启应用。

<h3 id="session-not-finding-installed-tools">
  会话找不到已安装的工具
</h3>

如果 Claude 找不到 `npm`、`node` 或其他 CLI 命令等工具，验证工具在你的常规终端中工作，检查你的 shell 配置文件是否正确设置 PATH，并重启桌面应用以重新加载环境变量。

<h3 id="git-and-git-lfs-errors">
  Git 和 Git LFS 错误
</h3>

在其自己的 worktree 中运行的会话需要 Git。如果你看到"Git is required"，安装 [Git](https://git-scm.com/downloads)，或在 Windows 上安装 [Git for Windows](https://git-scm.com/downloads/win)，然后重试。在 Windows 上，1.49585.0 之前的 Claude Desktop 版本在启动任何本地会话之前要求 Git；如果你看到该提示且不使用 worktrees，请更新应用。

如果你看到"Git LFS is required by this repository but is not installed"，从 [git-lfs.com](https://git-lfs.com/) 安装 Git LFS，运行 `git lfs install`，并重启应用。

<h3 id="mcp-servers-not-working-on-windows">
  MCP servers 在 Windows 上不工作
</h3>

如果 MCP server 切换不响应或服务器在 Windows 上连接失败，检查服务器在你的设置中是否正确配置，重启应用，验证服务器进程在任务管理器中运行，并查看服务器日志以获取连接错误。

<h3 id="app-won’t-quit">
  应用无法退出
</h3>

* **macOS**：按 Cmd+Q。如果应用不响应，使用 Cmd+Option+Esc 强制退出，选择 Claude，然后点击强制退出。
* **Windows**：使用 Ctrl+Shift+Esc 的任务管理器来结束 Claude 进程。

<h3 id="windows-specific-issues">
  Windows 特定问题
</h3>

* **安装后 PATH 未更新**：打开新的终端窗口。PATH 更新仅适用于新的终端会话。
* **并发安装错误**：如果你看到关于另一个安装正在进行的错误，但实际上没有，尝试以管理员身份运行安装程序。

<h3 id="branch-doesn’t-exist-yet-when-opening-in-cli">
  在 CLI 中打开时"Branch doesn't exist yet"
</h3>

云端会话可以创建在您的本地机器上不存在的分支。点击会话中的分支名称并选择 **Copy branch name**，然后在本地获取它：

```bash theme={null}
git fetch origin <branch-name>
git checkout <branch-name>
```

<h3 id="still-stuck">
  仍然卡住？
</h3>

* 在桌面应用中打开 Help → Get Support，或直接访问 [Claude 支持中心](https://support.claude.com/)
* 对于在独立 `claude` CLI 中也能重现的问题，在 [GitHub Issues](https://github.com/anthropics/claude-code/issues) 上搜索或提交错误

提交问题时，包括你的桌面应用版本、你的操作系统、确切的错误消息和相关日志。在 macOS 上，检查 Console.app。在 Windows 上，检查事件查看器 → Windows 日志 → 应用程序。在将日志摘录发布到公开问题之前，请查看它们；它们可能包含来自你的环境的文件路径和其他详细信息。
