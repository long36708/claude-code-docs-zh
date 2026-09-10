> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 在网络上使用 Claude Code

> 使用 `--cloud` 和 `--teleport` 在网络和终端之间移动会话，管理和共享会话，以及从云端自动修复拉取请求。

<Note>
  Claude Code on the web 处于研究预览阶段，适用于 Pro、Max 和 Team 用户，以及拥有高级席位或 Chat + Claude Code 席位的 Enterprise 用户。
</Note>

Claude Code on the web 在 [claude.ai/code](https://claude.ai/code) 的 Anthropic 管理的云基础设施上运行任务，或在路由到你的组织的[自托管环境](/docs/zh-CN/self-hosted-environments)时在那里运行。会话即使在关闭浏览器后也会持续，你可以从 Claude 移动应用监控它们。

<Tip>
  初次使用 Claude Code on the web？从[入门](/docs/zh-CN/web-quickstart)开始，连接你的 GitHub 账户并提交你的第一个任务。
</Tip>

本页涵盖网络产品本身：

* [云环境](#cloud-environments)：会话运行的位置，以及在哪里配置
* [GitHub 身份验证选项](#github-authentication-options)：两种连接 GitHub 的方式
* [在网络和终端之间移动任务](#move-tasks-between-web-and-terminal)，使用 `--cloud` 和 `--teleport`
* [处理会话](#work-with-sessions)：权限模式、审查、共享、归档、删除
* [自动修复拉取请求](#auto-fix-pull-requests)：自动响应 CI 失败和审查评论
* [安全和隔离](#security-and-isolation)：会话如何隔离
* [限制](#limitations)：速率限制和平台限制

<h2 id="cloud-environments">
  云环境
</h2>

每个云会话都在一个[云环境](/docs/zh-CN/cloud-environments)中运行，这是一个保存的配置，控制网络访问、环境变量和设置脚本。如果你还没有环境，入门会设置一个**默认**环境，具有[**受信任**网络访问](/docs/zh-CN/cloud-environments#access-levels)，要么为你创建它，要么要求你创建它。请参阅[默认环境](/docs/zh-CN/cloud-environments#the-default-environment)，了解在你的计划上会发生哪种情况，以及当你有多个环境时会话如何选择环境。

相同的环境适用于你启动云会话的任何地方：网络、终端、[Claude Tag](https://claude.com/docs/claude-tag/overview)、[routines](/docs/zh-CN/routines) 以及移动和 Desktop 应用。Claude Tag 频道会话仅使用组织级环境，要么是[共享环境](/docs/zh-CN/cloud-environments#organization-shared-environments)，要么是[自托管环境](/docs/zh-CN/self-hosted-environments)。

请参阅[配置云环境](/docs/zh-CN/cloud-environments)以更改环境允许的内容、设置变量或添加设置脚本，以及[已安装的工具](/docs/zh-CN/cloud-environments#installed-tools)以了解会话在没有任何配置的情况下包含的内容。

<h2 id="github-authentication-options">
  GitHub 身份验证选项
</h2>

云会话需要访问你的 GitHub 存储库来克隆代码和推送分支。你可以通过两种方式授予访问权限：

| 方法               | 工作原理                                                  | 最适合                                        |
| :--------------- | :---------------------------------------------------- | :----------------------------------------- |
| **GitHub App**   | 在[网络入门](/docs/zh-CN/web-quickstart)期间授权 Claude GitHub App。 | 浏览器入门；想要[自动修复](#auto-fix-pull-requests)的团队 |
| **`/web-setup`** | 在终端中运行 `/web-setup` 以将本地 `gh` CLI 令牌同步到你的 Claude 账户。  | 已经使用 `gh` 的个人开发者                           |

<Note>
  使用任一方法，云会话都可以访问连接的 GitHub 账户可以看到的任何存储库，而不仅仅是安装了 Claude GitHub App 的存储库。App 安装启用 PR webhooks 用于[自动修复](#auto-fix-pull-requests)；它不是会话级别的访问控制。要限制你的团队可以从云会话访问哪些存储库，请在 GitHub 本身上限制访问，例如通过限制连接的 GitHub 账户的团队或存储库成员资格。
</Note>

任一方法都可以。有关 `/schedule` 如何在创建 routine 之前检查访问权限，请参阅[存储库和分支权限](/docs/zh-CN/routines#repositories-and-branch-permissions)。有关 `/web-setup` 演练，请参阅[从终端连接](/docs/zh-CN/web-quickstart#connect-from-your-terminal)。

快速网络设置是一个组织设置，允许成员使用 `/web-setup` 连接 GitHub，在浏览器入门期间跳过 Claude GitHub App 安装提示，并让浏览器入门为他们创建[**默认**环境](/docs/zh-CN/cloud-environments#the-default-environment)，而不是显示环境表单。在 Team 和 Enterprise 计划上，默认情况下它是关闭的，这会隐藏 `/web-setup`。[所有者](/docs/zh-CN/server-managed-settings#access-control)可以在 [**Admin settings > Claude Code**](https://claude.ai/admin-settings/claude-code) 处使用**快速网络设置**切换来打开它。

<Note>
  启用了[零数据保留](/docs/zh-CN/zero-data-retention)的组织无法使用 `/web-setup` 或其他云会话功能。
</Note>

<h2 id="move-tasks-between-web-and-terminal">
  在网络和终端之间移动任务
</h2>

这些工作流需要[Claude Code CLI](/docs/zh-CN/quickstart)登录到相同的 claude.ai 账户。你可以从终端启动新的云会话，或将云会话拉入终端以在本地继续。云会话即使在关闭笔记本电脑后也会持续，你可以从任何地方（包括 Claude 移动应用）监控它们。

<Note>
  从 CLI，会话切换是单向的：你可以使用 `--teleport` 将云会话拉入终端，但不能将现有的终端会话推送到网络。带有任务描述的 `--cloud` 标志为你的当前存储库创建一个新的云会话；带有 `-p` 和会话 ID 或 claude.ai/code URL 时，它改为[将消息排队到该现有会话](/docs/zh-CN/claude-code-on-the-web#send-follow-ups-from-the-cli)。[Desktop 应用](/docs/zh-CN/desktop#continue-in-another-surface)提供了一个"在...中继续"菜单，可以将本地会话发送到网络。
</Note>

<h3 id="from-terminal-to-web">
  从终端到网络
</h3>

使用 `--cloud` 标志从命令行启动云会话：

```bash theme={null}
claude --cloud "Fix the authentication bug in src/auth/login.ts"
```

这在 claude.ai 上创建一个新的云会话。云 VM 在你的当前分支处克隆你当前目录的 GitHub 远程，而不是你的本地检出，所以如果你有本地提交，请先推送。`--cloud` 一次只能处理单个存储库。任务在云中运行，而你继续在本地工作。较旧的 `--remote` 拼写仍然作为 `--cloud` 的已弃用别名工作。

当云容器启动时，CLI 显示设置步骤的实时清单，例如克隆存储库和运行你的[设置脚本](/docs/zh-CN/cloud-environments#setup-scripts)。它排队你在配置期间输入的消息，并在会话准备好后发送它们。

<Note>
  `--cloud` 创建云会话。`--remote-control` 无关：它公开本地 CLI 会话以从网络进行监控。请参阅[Remote Control](/docs/zh-CN/remote-control)。
</Note>

在 Claude Code CLI 中使用 `/tasks` 检查进度，或在 claude.ai 或 Claude 移动应用上打开会话以直接交互。从那里你可以引导 Claude、提供反馈或回答问题，就像任何其他对话一样。

如果 Claude 提出问题且会话处于空闲状态，你仍然可以在返回时回答，直到[环境过期](#environment-expired)，会话从你的答案继续。

<h4 id="tips-for-cloud-tasks">
  云任务的提示
</h4>

**在本地规划，远程执行**：对于复杂的任务，在 plan mode 中启动 Claude 以协作制定方法，然后将工作发送到云：

```bash theme={null}
claude --permission-mode plan
```

在 plan mode 中，Claude 读取文件、运行命令来探索并提出计划，而不编辑源代码。一旦你满意，将计划保存到存储库、提交和推送，以便云 VM 可以克隆它。然后为自主执行启动云会话：

```bash theme={null}
claude --cloud "Execute the migration plan in docs/migration-plan.md"
```

**并行运行任务**：每个 `--cloud` 命令创建自己的云会话，独立运行。你可以启动多个任务，它们都将在单独的会话中同时运行：

```bash theme={null}
claude --cloud "Fix the flaky test in auth.spec.ts"
claude --cloud "Update the API documentation"
claude --cloud "Refactor the logger to use structured output"
```

使用 Claude Code CLI 中的 `/tasks` 监控所有会话。当会话完成时，你可以从网络界面创建 PR 或[teleport](#from-web-to-terminal) 会话到终端以继续工作。

<h4 id="send-local-repositories-without-github">
  发送没有 GitHub 的本地存储库
</h4>

当你从未连接到 GitHub 的存储库运行 `claude --cloud` 时，Claude Code 会捆绑你的本地存储库并直接上传到云会话。捆绑包包括你的完整存储库历史，跨所有分支，加上对跟踪文件的任何未提交更改。

在 macOS、Linux 和 WSL 上，Claude Code 会将名称类似于凭证或密钥的文件的未提交更改排除在上传之外，并列出它排除的文件的名称。这涵盖 `.env` 文件、Terraform `*.tfvars` 文件和密钥文件，如 `id_rsa` 和 `*.pem`。会话以每个的已提交版本开始，或如果没有提交则没有文件。在链接的 worktree、submodule 或类似布局中，Claude Code 将这些更改与其余部分一起上传，并列出它上传的文件的名称。

当 GitHub 访问不可用时，此回退会自动激活。要即使在 GitHub 已连接时也强制它，请设置 `CCR_FORCE_BUNDLE=1`：

```bash theme={null}
CCR_FORCE_BUNDLE=1 claude --cloud "Run the test suite and fix any failures"
```

捆绑的存储库必须满足这些限制：

* 目录必须是具有至少一个提交的 git 存储库
* 捆绑的存储库必须在 100 MB 以下。较大的存储库回退到仅捆绑当前分支，然后回退到工作树的单个压缩快照，仅在快照仍然太大时失败
* 未跟踪的文件不包括；在你想要云会话看到的文件上运行 `git add`
* 从捆绑创建的会话无法推送回远程，除非你也配置了[GitHub 身份验证](#github-authentication-options)

<h3 id="send-follow-ups-from-the-cli">
  从 CLI 发送后续消息
</h3>

一旦云会话运行，无论它在哪里执行，从任何你使用 `claude auth login` 登录的机器上的 `claude` CLI 向它发送后续消息。CLI 使用你的 Anthropic 账户凭证进行身份验证，不发送本地会话状态，所以命令不需要从启动会话的机器运行，在每个 shell 中都是相同的，包括 PowerShell。

该命令发布一条消息并退出：

```bash theme={null}
claude -p "your message" --cloud <session-id>
```

CLI 将消息排队到会话中并退出，不等待回复。使用它来引导长时间运行的会话、在当前步骤仍在完成时排队下一步，或从[CI 脚本](/docs/zh-CN/self-hosted-environments-testing#run-the-test-loop)发送后续消息。你也可以在 stdin 上管道消息，而不是作为参数传递：`echo "your message" | claude -p --cloud <session-id>`。

对于 `<session-id>`，传递裸 ID，如 `session_...` 或 `cse_...`，或会话的 `claude.ai/code/<id>` URL，带或不带方案或查询字符串。在 claude.ai/code 的会话列表中找到 ID。

<Note>
  `--cloud` 需要 Anthropic 账户。当 Claude Code 配置为 Amazon Bedrock、Google Cloud 的 Agent Platform 或其他第三方提供商时，它不可用。仅通过 `ANTHROPIC_BASE_URL` 配置的[LLM gateway](/docs/zh-CN/llm-gateway)不算作第三方提供商进行此检查，但你仍然需要使用 `claude auth login` 登录。你的组织的 `allow_remote_sessions` 策略也必须启用。所有者可以在 claude.ai/admin-settings/claude-code 处的 Claude Code 管理设置中打开它。
</Note>

<h4 id="output-and-errors">
  输出和错误
</h4>

成功时，命令打印会话 ID 和查看会话的链接：

```
Sent to cloud session.
Session ID: session_01DiUkqY2kzbUbDmW1w96rfi
View: https://claude.ai/code/session_01DiUkqY2kzbUbDmW1w96rfi?from=cli&m=0
```

传递 `--output-format json` 以获得机器可读的结果：成功时为 `{ok, session_id, url}`，或发送失败时为 `{ok: false, session_id, error}`，例如当会话丢失或已归档时。配置错误，如不支持的提供商或禁用的组织策略，打印到 stderr 而不是 JSON。`--output-format stream-json` 不支持 `--cloud <session-id>`。

CLI 使用 `Error: ` 前缀错误。失败的交付被包装为 `failed to send message to cloud session <id>: <reason>`。

| 消息                                                                                                                          | 含义                                                                                                                                                                     |
| --------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Cloud sessions aren't available with <provider>. They run on Anthropic's infrastructure and require an Anthropic account.` | Claude Code 配置为第三方提供商。消息使用你的配置使用的标签命名提供商，如 `Amazon Bedrock` 或 `Google Vertex AI`。删除该提供商的配置，例如通过取消设置 `CLAUDE_CODE_USE_BEDROCK`，并使用 Anthropic 账户登录（`claude auth login`）。 |
| `Cloud sessions are disabled by your organization's policy. Contact your organization admin to enable them.`                | `allow_remote_sessions` 组织策略已关闭。                                                                                                                                       |
| `Couldn't verify your organization's policy for cloud sessions. Check your network connection and try again.`               | Claude Code 无法获取你的组织的策略，所以它拒绝发送而不是假设云会话被允许。检查你的网络连接并重试。                                                                                                                |
| `Attaching to an existing cloud session is not enabled for your account.`                                                   | 你运行了 `--cloud <session-id>` 而没有 `-p`。使用 `claude -p "your message" --cloud <session-id>` 发送消息。                                                                          |
| `Session not found: <id>`                                                                                                   | ID 或 URL 与你可以访问的会话不匹配。根据会话的 claude.ai/code URL 检查它。                                                                                                                    |
| `cloud session <id> is archived and cannot accept new messages`                                                             | 会话已被归档。改为启动新会话。                                                                                                                                                        |

<h3 id="from-web-to-terminal">
  从网络到终端
</h3>

使用以下任何方式将云会话拉入终端：

* **使用 `--teleport`**：从命令行，运行 `claude --teleport` 以获得交互式会话选择器，或 `claude --teleport <session-id>` 以直接恢复特定会话。如果你有未提交的更改，系统会提示你先隐藏它们。
* **使用 `/teleport`**：在现有 CLI 会话内，运行 `/teleport` 或 `/tp` 以打开相同的会话选择器，无需重启 Claude Code。
* **从 `/tasks`**：运行 `/tasks` 以查看你的后台会话，然后按 `t` teleport 到其中一个。
* **从网络界面**：从会话菜单中选择**在终端中打开**以复制可以粘贴到终端中的命令。
* **从云会话内部**：输入 `/teleport`，Claude Code 会回复确切的 `claude --teleport <session-id>` 命令用于该会话，准备从存储库的检出运行。需要会话环境中的 Claude Code v2.1.223 或更高版本。

当你 teleport 一个会话时，Claude 验证你在正确的存储库中，从云会话获取并检出分支，并将完整的对话历史加载到终端中。终端获得会话的自己的副本：那里的新工作保持本地，不会出现在 claude.ai 上的云会话或 Claude 移动应用中。要在 teleport 后继续从你的手机引导，在本地会话中启动[`/remote-control`](/docs/zh-CN/remote-control)。

`--teleport` 不同于 `--resume`。`--resume` 从此机器的本地历史重新打开对话，不列出云会话；`--teleport` 拉取云会话及其分支。

<h4 id="teleport-requirements">
  Teleport 要求
</h4>

Teleport 在恢复会话之前检查这些要求。如果任何要求未满足，你会看到错误或被提示解决问题。

| 要求         | 详情                                                                                                                                                                                                |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 干净的 git 状态 | 你的工作目录必须没有未提交的更改。Teleport 会在需要时提示你隐藏更改。                                                                                                                                                           |
| 正确的存储库     | 你必须从同一存储库的检出运行 `--teleport`，而不是从分叉运行。如果你从不同存储库的检出运行它，Claude Code 会显示一个错误，命名会话的存储库和你的检出的。如果 Claude Code 无法将你的远程解析为主机名，例如 SSH 主机别名如 `git@work:owner/repo.git`，它会要求你确认，并在远程的所有者和存储库名称与会话的存储库匹配时接受检出。 |
| 分支可用       | 云会话中的分支必须已被推送到远程。Teleport 会自动获取并检出它。                                                                                                                                                              |
| 相同账户       | 你必须认证到云会话中使用的相同 claude.ai 账户。                                                                                                                                                                     |

<h4 id="teleport-is-unavailable">
  `--teleport` 不可用
</h4>

Teleport 需要 claude.ai 订阅身份验证。如果你通过 API 密钥进行身份验证，运行 `/login` 以改为使用你的 claude.ai 账户登录。如果错误命名你的提供商，云会话不通过第三方提供商可用；请参阅[错误表](#output-and-errors)。如果你已通过 claude.ai 登录且 `--teleport` 仍不可用，你的组织可能已禁用云会话。

<h2 id="work-with-sessions">
  处理会话
</h2>

会话出现在 claude.ai/code 的侧边栏中。从那里你可以审查更改、与队友共享、归档完成的工作或永久删除会话。

<h3 id="manage-context">
  管理上下文
</h3>

云会话支持产生文本输出的[内置命令](/docs/zh-CN/commands)。仅在终端界面中运行的命令，如 `/plugin` 或 `/resume`，不可用。在云会话中打开选择器或面板的命令表现不同：

* **`/model`、`/effort`、`/fast`、`/color` 和 `/rename`**：将值作为参数传递，例如 `/model sonnet`，而不是打开终端选择器或滑块。参数形式需要会话环境中的 Claude Code v2.1.205 或更高版本，并遵循每个命令的[可用性说明](/docs/zh-CN/commands#all-commands)：当模型的[启动默认工作量保持](/docs/zh-CN/model-config#adjust-effort-level)生效时，`/effort` 报告 `Not applied`，而 `/fast` 仅在以快速模式启动的会话中工作。
* **`/config`**：在网络上，打开你的设置的 Claude Code 部分，而不是设置值，命令后的文本（包括 `key=value`）被忽略。要更改云会话的设置，请使用[环境变量](/docs/zh-CN/cloud-environments#set-environment-variables)或将[设置文件](/docs/zh-CN/settings)提交到存储库。

对于上下文管理特别是：

| 命令         | 在云会话中工作 | 注释                                                     |
| :--------- | :------ | :----------------------------------------------------- |
| `/compact` | 是       | 总结对话以释放上下文。接受可选的焦点指令，如 `/compact keep the test output` |
| `/context` | 是       | 显示当前在上下文窗口中的内容                                         |
| `/clear`   | 否       | 从侧边栏启动新会话                                              |

自动压缩在上下文窗口接近容量时自动运行。Claude Code on the web 在云会话中自己设置 [`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`](/docs/zh-CN/env-vars)，所以压缩在[自动压缩窗口](/docs/zh-CN/model-config#set-the-auto-compact-window)的中途触发，而不是当窗口填满时。该值覆盖你在[环境变量](/docs/zh-CN/cloud-environments#set-environment-variables)中添加的值，所以在那里添加变量不会改变压缩何时触发。

要改为更改自动压缩窗口，请在你的环境变量中设置 [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](/docs/zh-CN/env-vars)，或在变量未设置的会话中运行带有令牌计数的 [`/autocompact`](/docs/zh-CN/commands#all-commands)。

[Subagents](/docs/zh-CN/sub-agents) 的工作方式与本地相同。Claude 可以使用 Agent 工具生成它们，以将研究或并行工作卸载到单独的上下文窗口中，保持主对话更轻。在你的存储库的 `.claude/agents/` 中定义的 Subagents 会自动被拾取。

[Agent teams](/docs/zh-CN/agent-teams) 默认关闭，但可以通过将 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 添加到你的[环境变量](/docs/zh-CN/cloud-environments#set-environment-variables)来启用。

<h3 id="permission-modes-in-cloud-sessions">
  云会话中的权限模式
</h3>

你从[模式下拉菜单](/docs/zh-CN/permission-modes#switch-permission-modes)为云会话选择[权限模式](/docs/zh-CN/permission-modes)，既在你创建任务时，也在会话运行时。当你重新打开其 Anthropic 托管[环境已过期](#environment-expired)的会话，或向自托管运行程序在空闲时[释放的会话](/docs/zh-CN/self-hosted-environments-reference#runner-cli-flags)发送消息时，Claude Code 在它所在的权限模式中恢复会话。

<h3 id="review-changes">
  审查更改
</h3>

每个会话显示一个 diff 指示器，显示添加和删除的行数，如 `+42 -18`。选择它以打开 diff 视图，在特定行上留下内联评论，并使用你的下一条消息将它们发送给 Claude。

Claude Code 从原始 git blob 内容计算这些 diffs，包括 Claude 编辑时显示的每个文件 diffs，所以存储库中配置的 diff 驱动程序和 `textconv` 过滤器不适用。对于不是会话自己的检出之一的存储库中的文件，如在会话期间克隆到工作区内的文件，每个文件 diff 显示 Claude 的编辑本身，而不是 git 比较。

有关完整演练（包括 PR 创建），请参阅[审查和迭代](/docs/zh-CN/web-quickstart#review-and-iterate)。要让 Claude 自动监控 PR 以查找 CI 失败和审查评论，请参阅[自动修复拉取请求](#auto-fix-pull-requests)。

<h3 id="share-sessions">
  共享会话
</h3>

要共享会话，请根据下面的账户类型切换其可见性。之后，按原样共享会话链接。打开链接时，收件人会看到最新状态，但他们的视图不会实时更新。

<h4 id="share-from-an-enterprise-or-team-account">
  从 Enterprise 或 Team 账户共享
</h4>

对于 Enterprise 和 Team 账户，两个可见性选项是**私有**和**团队**。团队可见性使会话对你的 claude.ai 组织的其他成员可见。[Claude in Slack](/docs/zh-CN/slack) 会话会自动以团队可见性共享。

默认情况下启用存储库访问验证，基于连接到收件人账户的 GitHub 账户。你的账户显示名称对所有有访问权限的收件人可见。

<h4 id="share-from-a-max-or-pro-account">
  从 Max 或 Pro 账户共享
</h4>

对于 Max 和 Pro 账户，两个可见性选项是**私有**和**公开**。公开可见性使会话对任何登录到 claude.ai 的用户可见。

在共享之前检查你的会话是否包含敏感内容。会话可能包含来自私有 GitHub 存储库的代码和凭证。默认情况下不启用存储库访问验证。

要要求收件人拥有存储库访问权限，或从共享会话中隐藏你的名称，请转到 [**Settings > Claude Code > Sharing settings**](https://claude.ai/settings/claude-code)。

<h3 id="archive-sessions">
  归档会话
</h3>

你可以归档会话以保持你的会话列表有序。归档的会话从默认会话列表中隐藏，但可以通过筛选已归档会话来查看。

要归档会话，请在侧边栏中悬停在会话上并选择归档图标。

<h3 id="delete-sessions">
  删除会话
</h3>

删除会话会永久删除会话及其数据。此操作无法撤销。你可以通过两种方式删除会话：

* **从侧边栏**：筛选已归档会话，然后悬停在你想删除的会话上并选择删除图标
* **从会话菜单**：打开会话，选择会话标题旁的下拉菜单，然后选择**删除**

删除会话前会要求你确认。

<h2 id="auto-fix-pull-requests">
  自动修复拉取请求
</h2>

Claude 可以监视拉取请求并自动响应 CI 失败和审查评论。Claude 订阅 PR 上的 GitHub 活动，当检查失败或审查者留下评论时，Claude 会调查并推送修复（如果有明确的修复）。

<Note>
  自动修复需要在你的存储库上安装 Claude GitHub App。如果你还没有，请从 [GitHub App 页面](https://github.com/apps/claude) 安装它。
</Note>

根据 PR 来自何处以及你使用的设备，有几种方法可以打开自动修复：

* **在 Claude Code on the web 中创建的 PR**：打开 CI 状态栏并选择**自动修复**
* **从终端**：在 PR 的分支上运行 [`/autofix-pr`](/docs/zh-CN/commands)。Claude Code 使用 `gh` 检测打开的 PR，生成网络会话，并一步启用自动修复
* **从移动应用**：告诉 Claude 自动修复 PR，例如"watch this PR and fix any CI failures or review comments"
* **任何现有 PR**：将 PR URL 粘贴到会话中并告诉 Claude 自动修复它

自动修复是按 PR 的切换开关。要停止监视，请在网络会话中打开 CI 状态栏并清除**自动修复**切换，或告诉 Claude 停止监视 PR。

<h3 id="how-claude-responds-to-pr-activity">
  Claude 如何响应 PR 活动
</h3>

当自动修复处于活动状态时，Claude 接收 PR 的 GitHub 事件，包括新的审查评论和 CI 检查失败。对于每个事件，Claude 调查并决定如何进行：

* **明确的修复**：如果 Claude 对修复有信心且不与早期指令冲突，Claude 会进行更改、推送它，并在会话中解释所做的工作
* **模糊的请求**：如果审查者的评论可以以多种方式解释或涉及架构上重要的内容，Claude 会在采取行动前询问你
* **重复或无操作事件**：如果事件是重复的或不需要更改，Claude 会在会话中记录它并继续

GitHub 不会在基础分支推进并创建合并冲突时发出 webhook，因此自动修复无法自行对冲突做出反应。要解决冲突，请打开会话并要求 Claude 进行变基。

Claude 可能会作为解决审查评论线程的一部分在 GitHub 上回复它们。这些回复使用你的 GitHub 账户发布，所以它们出现在你的用户名下，但每个回复都标记为来自 Claude Code，以便审查者知道它是由代理编写的，而不是由你直接编写的。

<Warning>
  如果你的存储库使用注释触发的自动化，例如 Atlantis、Terraform Cloud 或在 `issue_comment` 事件上运行的自定义 GitHub Actions，请注意 Claude 可以代表你回复，这可能会触发这些工作流。在启用自动修复之前审查你的存储库的自动化，并考虑为可能部署基础设施或运行特权操作的 PR 注释的存储库禁用自动修复。
</Warning>

<h2 id="security-and-isolation">
  安全和隔离
</h2>

每个云会话通过多个层与你的机器和其他会话分离：

* **隔离的虚拟机**：每个会话在隔离的、Anthropic 管理的 VM 中运行。你的组织路由到[自托管环境](/docs/zh-CN/self-hosted-environments)的会话改为在你自己的基础设施上运行，其中隔离是你的部署的责任
* **网络访问控制**：在 Anthropic 托管的环境中，网络访问默认受限，可以禁用。在自托管环境中，你在自己的网络边界处限制会话出口。当在禁用网络访问的情况下运行时，Claude Code 仍然可以与 Anthropic API 通信，这可能允许数据从 VM 中退出。
* **凭证保护**：在 Anthropic 托管的环境中，git 凭证和签名密钥保持在沙箱外，代理使用作用域凭证代表会话进行身份验证。在自托管环境中，你的部署提供 git 凭证；请参阅[配置 git](/docs/zh-CN/self-hosted-environments-deploy#configure-git)
* **API 凭证**：在 Pro 和 Max 计划的 Anthropic 托管环境中，你[添加到云环境](/docs/zh-CN/cloud-environments#add-api-credentials)的密钥保持在沙箱外，以相同的方式，在它们离开会话后附加到匹配的请求。自托管环境没有 API 凭证，Team 和 Enterprise 计划还没有
* **安全分析**：代码在会话的隔离环境内分析和修改，然后创建 PR

<h2 id="troubleshooting">
  故障排除
</h2>

对于出现在对话中的运行时 API 错误，如 `API Error: 500`、`529 Overloaded`、`429` 或 `Prompt is too long`，请参阅[错误参考](/docs/zh-CN/errors)。这些错误及其修复与 CLI 和 Desktop 应用共享。下面的部分涵盖特定于云会话的问题。

<h3 id="session-creation-failed">
  会话创建失败
</h3>

如果新会话无法启动，显示 `Session creation failed` 或在配置时停滞，Claude Code 无法为会话分配 VM。

* 检查 [status.claude.com](https://status.claude.com) 以了解云会话事件
* 一分钟后重试，因为容量是按需配置的
* 确认你的存储库可访问。连接的 GitHub 账户必须通过 Claude GitHub App 授权或通过 `/web-setup` 同步的 `gh` 令牌在 GitHub 上拥有对存储库的访问权限。不需要在存储库上安装该应用。请参阅 [GitHub 身份验证选项](#github-authentication-options)。

<h3 id="unable-to-get-organization-uuid">
  无法获取组织 UUID
</h3>

`claude --cloud` 和 `claude --teleport` 需要使用 claude.ai 账户登录。如果你使用 API 密钥进行身份验证，或你的存储账户详情已过期，这些命令会失败，显示 `Unable to get organization UUID` 或消息 API 密钥身份验证不足。使用 API 密钥身份验证或过期账户详情，运行不带会话 ID 的 `claude --teleport` 会在会话选择器中显示 `Error loading Claude Code sessions`，而不是任一消息，相同的修复适用。

运行 `/login` 以使用你的 claude.ai 账户登录，然后重试命令。如果错误命名你的提供商，请参阅[错误表](#output-and-errors)：云会话不通过第三方提供商可用。

<h3 id="remote-control-session-expired-or-access-denied">
  Remote Control 会话已过期或访问被拒绝
</h3>

`--teleport` 通过与云会话使用的相同 Remote Control 会话基础设施连接，所以身份验证和会话过期错误会显示 Remote Control 措辞。你可能会看到 `Remote Control session expired` 或 `Access denied`。连接令牌是短期的，并限定于你的账户。

* 在本地运行 `/login` 以刷新你的凭证，然后重新连接
* 确认你已登录到拥有会话的相同账户
* 如果你看到 `Remote Control may not be available for this organization`，所有者尚未为你的组织启用云会话

<h3 id="environment-expired">
  环境已过期
</h3>

云会话在不活动一段时间后停止，会话的 VM 被回收。会话在等待你批准[MCP 连接器](/docs/zh-CN/cloud-environments#network-access)工具调用或登录到 MCP 服务器时计为不活动，它可以在该等待期间过期。在网络上，会话在会话列表中标记为已过期。

从 [claude.ai/code](https://claude.ai/code) 重新打开会话以配置新 VM，并恢复你的对话历史。在 VM 被回收时仍在运行的后台工作，如 subagents 和 shell 命令，不会被恢复。

<h2 id="limitations">
  限制
</h2>

在依赖云会话进行工作流之前，请考虑这些约束：

* **速率限制**：Claude Code on the web 与你账户内所有其他 Claude 和 Claude Code 使用共享速率限制。并行运行多个任务会按比例消耗更多速率限制。云 VM 没有单独的计算费用。
* **存储库身份验证**：你只能在认证到相同账户时将会话从网络移动到本地
* **平台限制**：存储库克隆和拉取请求创建需要 GitHub。自托管[GitHub Enterprise Server](/docs/zh-CN/github-enterprise-server) 实例支持 Team 和 Enterprise 计划。GitLab、Bitbucket 和其他非 GitHub 存储库可以作为[本地捆绑](#send-local-repositories-without-github)发送到云会话，但会话无法将结果推送回远程
* **组织 IP 允许列表**：云会话从 Anthropic 管理的基础设施而不是你的网络调用 Anthropic API，而[自托管环境](/docs/zh-CN/self-hosted-environments)中的会话从你自己的网络调用它。如果你的组织启用了 [IP 允许列表](https://support.claude.com/en/articles/13200993-restrict-access-to-claude-with-ip-allowlisting)，每个 Anthropic 托管的云会话都会失败，显示身份验证错误。这同样适用于[代码审查](/docs/zh-CN/code-review)和[routines](/docs/zh-CN/routines)在 Anthropic 托管的环境中运行；路由到自托管环境的 routine 从你自己的网络调用 API。联系 [Anthropic 支持](https://support.claude.com/)以从你的组织的 IP 允许列表中豁免 Anthropic 托管的服务。

<h2 id="related-resources">
  相关资源
</h2>

* [云环境](/docs/zh-CN/cloud-environments)：为云会话配置网络访问、环境变量和设置脚本
* [Ultrareview](/docs/zh-CN/ultrareview)：在云沙箱中运行深度多代理代码审查
* [Routines](/docs/zh-CN/routines)：按计划、通过 API 调用或响应 GitHub 事件自动化工作
* [Hooks 配置](/docs/zh-CN/hooks)：在会话生命周期事件处运行脚本
* [所有设置](/docs/zh-CN/settings-reference)：所有配置选项
* [安全](/docs/zh-CN/security)：隔离保证和数据处理
* [数据使用](/docs/zh-CN/data-usage)：Anthropic 从云会话保留的内容
* [Claude Tag](https://claude.com/docs/claude-tag/overview)：在 Slack 中由组织管理的 @Claude，运行在相同的云基础设施上
