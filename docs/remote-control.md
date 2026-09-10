> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 使用 Remote Control 从任何设备继续本地会话

> 使用 Remote Control 从您的手机、平板电脑或任何浏览器继续本地 Claude Code 会话。适用于 claude.ai/code 和 Claude 移动应用。

<Note>
  Remote Control 在所有计划中都可用。在 Team 和 Enterprise 上，在所有者在 [Claude Code 管理员设置](https://claude.ai/admin-settings/claude-code)中启用 Remote Control 切换之前，它默认处于关闭状态。
</Note>

Remote Control 将 [claude.ai/code](https://claude.ai/code) 或 Claude 应用（[iOS](https://apps.apple.com/us/app/claude-by-anthropic/id6473753684) 和 [Android](https://play.google.com/store/apps/details?id=com.anthropic.claude)）连接到在您的机器上运行的 Claude Code 会话。在您的办公桌上启动一个任务，然后从沙发上的手机或另一台计算机上的浏览器继续。

当您在机器上启动 Remote Control 会话时，Claude 始终在本地运行，因此您的代码执行和文件系统访问保留在您的机器上。使用 Remote Control，您可以：

* **远程使用您的完整本地环境**：您的文件系统、[MCP servers](/docs/zh-CN/mcp)、工具和项目配置都保持可用，输入 `@` 会自动完成本地项目中的文件路径。
* **同时从两个界面工作**：对话和 [subagents](/docs/zh-CN/sub-agents) 和 [dynamic workflows](/docs/zh-CN/workflows) 的进度在所有连接的设备上保持同步，因此您可以从终端、浏览器和手机交替发送消息。
* **从您的手机或浏览器发送图像和文件**：在 Claude 应用或 claude.ai/code 中附加照片或文件，可以带有或不带有标题。Claude 直接将附加的照片视为您消息的一部分。Claude Code 将其他文件下载到您的机器，并将其作为 `@` 文件引用传递给 Claude。
* **在中断后恢复**：如果您的笔记本电脑进入睡眠状态或网络断开，当您的机器重新上线时，Claude Code 会自动重新连接。在连接重建时，Claude Code 会对来自 subagents 和工作流的消息、权限提示和状态更新进行排队，并在连接恢复后传递它们。

与[网络上的 Claude Code](/docs/zh-CN/claude-code-on-the-web)（在云基础设施上运行）不同，Remote Control 会话直接在您的机器上运行并与您的本地文件系统交互。网络和移动界面只是该本地会话的一个窗口。

本页涵盖设置、如何启动和连接到会话，以及 Remote Control 与网络上的 Claude Code 的比较。

<h2 id="requirements">
  要求
</h2>

在使用 Remote Control 之前，请确认您的环境满足以下条件：

* **订阅**：在 Pro、Max、Team 和 Enterprise 计划中可用。不支持 API 密钥。在 Team 和 Enterprise 上，Owner 必须首先在 [Claude Code 管理员设置](https://claude.ai/admin-settings/claude-code)中启用 Remote Control 切换。
* **身份验证**：运行 `claude` 并使用 `/login` 通过 claude.ai 登录（如果您还没有登录）。如果没有符合条件的登录，`claude remote-control` 会以错误退出，而 `claude --remote-control` 仍会启动交互式会话，并在启动后不久显示 Remote Control 失败通知。
* **API 端点**：在以下任何配置中都不可用：
  * 您使用 Amazon Bedrock、Google Cloud 的 Agent Platform 或 Microsoft Foundry。
  * 您将 [`ANTHROPIC_BASE_URL`](/docs/zh-CN/env-vars) 指向 `api.anthropic.com` 以外的主机，例如 [LLM gateway](/docs/zh-CN/llm-gateway) 或代理。取消设置该变量以使用 Remote Control。在 v2.1.196 之前，Claude Code 允许使用自定义 `ANTHROPIC_BASE_URL` 进行 Remote Control。
  * 您通过企业 [Claude apps gateway](/docs/zh-CN/claude-apps-gateway) 登录。
* **功能标志评估**：[`DISABLE_TELEMETRY`、`DO_NOT_TRACK`、`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 和 `DISABLE_GROWTHBOOK`](/docs/zh-CN/env-vars) 各自禁用 Remote Control 可用性所依赖的功能标志评估。在您的 shell 环境或 [`settings.json` 文件](/docs/zh-CN/settings-reference#all-settings)的 `env` 块中取消设置该变量，以使用 Remote Control。
* **工作区信任**：在您的项目目录中至少运行一次 `claude` 以接受工作区信任对话框。启动信任对话框永远不会为您的主目录保存信任，因此请从项目目录启动 Remote Control。

<h2 id="start-a-remote-control-session">
  启动 Remote Control 会话
</h2>

您可以从 CLI 或 VS Code 扩展启动 Remote Control 会话。CLI 提供三种调用模式；VS Code 使用 `/remote-control` 命令。

<Tabs>
  <Tab title="服务器模式">
    导航到您的项目目录并运行：

    ```bash theme={null}
    claude remote-control
    ```

    在您接受 Remote Control 的一次性确认之前，`claude remote-control` 会解释它的作用并在启动服务器之前询问 `Enable Remote Control? (y/n)`。回答 `y` 以接受并启动服务器。如果您拒绝，Claude Code 会退出而不启动服务器，并在您下次运行该命令时再次询问。

    该进程在您的终端中以服务器模式保持运行，等待远程连接。它显示一个会话 URL，您可以使用该 URL 从[另一个设备连接](#connect-from-another-device)，您可以按空格键显示 QR 码以从手机快速访问。当远程会话处于活动状态时，终端显示连接状态和工具活动。

    可用标志：

    | 标志                                              | 描述                                                                                                                                                                                                                                         |
    | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
    | `--name "My Project"`                           | 设置自定义会话标题，在 claude.ai/code 的会话列表中可见。                                                                                                                                                                                                       |
    | `--remote-control-session-name-prefix <prefix>` | 未设置显式名称时自动生成的会话名称的前缀。默认为您的机器的主机名，生成类似 `myhost-graceful-unicorn` 的名称。设置 `CLAUDE_REMOTE_CONTROL_SESSION_NAME_PREFIX` 以获得相同效果。                                                                                                                |
    | `-c`, `--continue`                              | 恢复此目录中最后一个服务器启动的会话，而不是创建新会话。请参阅[停止服务器后恢复会话](#resume-sessions-after-stopping-the-server)。不能与 `--session-id`、`--spawn`、`--capacity` 或 `--create-session-in-dir` 结合使用。需要 Claude Code v2.1.200 或更高版本；早期版本会将该标志拒绝为未知参数。                         |
    | `--session-id <id>`                             | 通过其 ID 恢复一个会话。请参阅[停止服务器后恢复会话](#resume-sessions-after-stopping-the-server)。不能与 `--continue`、`--spawn`、`--capacity` 或 `--create-session-in-dir` 结合使用。需要 Claude Code v2.1.200 或更高版本；早期版本会将该标志拒绝为未知参数。                                         |
    | `--spawn <mode>`                                | 服务器如何创建会话。<br />• `same-dir`（默认）：所有会话共享当前工作目录，因此如果编辑相同的文件可能会冲突。<br />• `worktree`：每个按需会话都获得自己的 [git worktree](/docs/zh-CN/worktrees)。需要 git 存储库。<br />• `session`：单会话模式。恰好提供一个会话并拒绝其他连接。仅在启动时设置。<br />在运行时按 `w` 在 `same-dir` 和 `worktree` 之间切换。 |
    | `--capacity <N>`                                | 最大并发会话数。默认为 32。不能与 `--spawn=session` 一起使用。                                                                                                                                                                                                 |
    | `--[no-]create-session-in-dir`                  | 在服务器启动时在当前目录中预创建一个会话，以便您有地方立即输入。在 `worktree` 模式下，此会话保留在当前目录中，而按需会话获得隔离的 worktrees。默认启用。如果您传递 `--no-create-session-in-dir` 以不创建任何会话启动，Claude Code 会在您停止服务器时存档服务器的会话，因此没有任何内容可以[恢复](#resume-sessions-after-stopping-the-server)。             |
    | `--permission-mode <mode>`                      | 为服务器的会话设置启动[权限模式](/docs/zh-CN/permission-modes)，例如 `acceptEdits`。接受 `manual` 作为 `default` 的别名；无法识别的模式会在启动时停止服务器并列出有效的模式。                                                                                                                        |
    | `--debug-file <path>`                           | 将调试日志写入给定的文件。                                                                                                                                                                                                                              |
    | `--verbose`                                     | 显示详细的连接和会话日志。                                                                                                                                                                                                                              |
    | `--sandbox` / `--no-sandbox`                    | 启用或禁用[沙箱](/docs/zh-CN/sandboxing)以进行文件系统和网络隔离。默认关闭。                                                                                                                                                                                             |

    在 `remote-control` 之后给出这些标志。

    如果您在 `remote-control` 之前传递全局 `claude` 标志，或包装脚本添加了一个，Claude Code 不会将该标志转移到服务器创建的会话。Claude Code 仅在丢弃该标志已知不会改变这些会话可以做什么时才允许该标志通过，例如 `--verbose` 或 `--model`。对于任何其他标志，例如 `--settings`，Claude Code [拒绝启动](/docs/zh-CN/errors#not-carried-over-to-the-sessions-remote-control-starts)并命名要删除的标志。在 v2.1.248 之前，`remote-control` 之前的任何选项都会导致 Claude Code 拒绝其后的标志，并出现 `unknown option` 错误。

    Claude Code 在打印帮助之前检查 Remote Control 资格，因此当您未使用符合条件的帐户登录时，`claude remote-control --help` 会返回错误而不是此标志列表。
  </Tab>

  <Tab title="交互式会话">
    要启动启用了 Remote Control 的普通交互式 Claude Code 会话，请使用 `--remote-control` 标志（或 `--rc`）：

    ```bash theme={null}
    claude --remote-control
    ```

    可选地为会话传递一个名称：

    ```bash theme={null}
    claude --remote-control "My Project"
    ```

    这为您提供了一个完整的交互式会话在您的终端中，您也可以从 claude.ai 或 Claude 应用控制。与 `claude remote-control`（服务器模式）不同，您可以在会话也可远程使用时在本地输入消息。
  </Tab>

  <Tab title="从现有会话">
    如果您已经在 Claude Code 会话中并想远程继续它，请使用 `/remote-control`（或 `/rc`）命令：

    ```text theme={null}
    /remote-control
    ```

    传递一个名称作为参数以设置自定义会话标题：

    ```text theme={null}
    /remote-control My Project
    ```

    这启动一个 Remote Control 会话，该会话继承您当前的对话历史记录。

    在您接受 Remote Control 的一次性确认之前，会出现一个对话框，然后 `/remote-control` 连接。选择**启用 Remote Control** 以接受并连接。如果您选择**算了** 或按 Esc，Claude Code 不会连接，并在您下次运行 `/remote-control` 时再次询问。

    此命令不支持 `--verbose`、`--sandbox` 和 `--no-sandbox` 标志。
  </Tab>

  <Tab title="VS Code">
    在 [Claude Code VS Code 扩展](/docs/zh-CN/vs-code)中，在提示框中输入 `/remote-control` 或 `/rc`。

    ```text theme={null}
    /remote-control
    ```

    当 Remote Control 打开时，Claude Code 在提示框页脚中显示 **Remote Control** 指示器。会话连接后，单击指示器直接转到会话，或在 [claude.ai/code](https://claude.ai/code) 的会话列表中找到它。Claude Code 也会在对话中发布会话 URL。要断开连接，请再次运行 `/remote-control`。

    与 CLI 不同，VS Code 命令不接受名称参数或显示 QR 码。会话标题从您的对话历史记录或第一条提示派生。
  </Tab>
</Tabs>

<h3 id="check-connection-status">
  检查连接状态
</h3>

在交互式终端会话中，当连接处于活动状态时，`/rc active` 指示器位于输入框下方的页脚中，如果终端太窄无法容纳它，则隐藏。指示器文本是指向 claude.ai 上会话的链接。使用向下箭头键选择它并按 Enter，或再次运行 `/remote-control`，打开状态面板，其中包含会话 URL 和 QR 码，您可以使用它从[另一个设备连接](#connect-from-another-device)。状态面板还提供断开连接选项。选择它以关闭 Remote Control；您的本地会话继续在终端中运行。

如果连接失败，Claude Code 会显示一条通知，说明失败原因，并将指示器切换到保留在页脚中的失败状态。要再次读取原因，请使用向下箭头键选择指示器并按 Enter。要重新连接，请运行 `/remote-control`，除非[原因说会话在其他地方被接管或结束，或服务器找不到它](#session-ended-elsewhere)。

在重新连接之前读取原因。<span id="session-ended-elsewhere" />当会话从另一个设备、应用或 Claude Code 会话被接管或结束，或服务器找不到它时，原因会说明是哪种情况，Claude Code 会省略其通常的建议来运行 `/remote-control`：

* **另一个设备或 Claude Code 会话接管了会话**：仅当您想从该设备收回它时才运行 `/remote-control`。
* **您从另一个设备或应用结束或存档了会话**：仅当您想要它回来时才运行 `/remote-control`；Claude Code 会重新打开存档的会话。
* **服务器找不到会话**：它可能已从另一个设备或应用中删除。

<h3 id="session-url-reminders">
  会话 URL 提醒
</h3>

当 Remote Control 连接时，Claude Code 会在切换到您的手机或浏览器最有帮助时提醒您会话 URL，因此您不必在 `/remote-control` 中查找链接。提醒会在以下任一时刻出现在提示框上方：

* **长回合**：当回合运行时间超过服务器调整的阈值时，Claude Code 会显示**仍在工作**通知，带有**从您的手机检查**链接，因此您可以从手机或浏览器跟踪回合，而不是在终端等待。Claude Code 在回合结束时删除它。
* **重复的权限提示**：在您在会话中回答了多个[权限提示](/docs/zh-CN/permissions)后，**从您的手机批准工具调用**通知会显示会话 URL。Claude Code 在您的下一个回合开始时删除它。

提醒可以出现在任何连接的会话中，包括 Remote Control [自动连接](#enable-remote-control-for-all-sessions)的会话。它们不会在每次这些条件发生时出现，每个提醒在所有会话中总共只出现几次。您无法配置或关闭它们；每个都会自动清除。

<h3 id="connect-from-another-device">
  从另一个设备连接
</h3>

一旦 Remote Control 会话处于活动状态，您有几种方式从另一个设备连接：

* **打开会话 URL** 在任何浏览器中直接转到 [claude.ai/code](https://claude.ai/code) 上的会话。
* **扫描 QR 码** 显示在会话 URL 旁边，直接在 Claude 应用中打开它。使用 `claude remote-control` 时，按空格键切换 QR 码显示。
* **打开 [claude.ai/code](https://claude.ai/code) 或 Claude 应用** 并在会话列表中按名称查找会话。在 Claude 移动应用中，点击导航中的**代码**以访问会话列表。Remote Control 会话在在线时显示带有绿色状态点的计算机图标。

当您连接时，设备显示会话已在后台运行的任何子代理和工作流。从设备停止其中一个，Claude Code 会停止您机器上的该任务。

远程会话标题按以下顺序选择：

1. 您传递给 `--name`、`--remote-control` 或 `/remote-control` 的名称
2. 您使用 `/rename` 设置的标题
3. 现有对话历史记录中的最后一条有意义的消息
4. 自动生成的名称，如 `myhost-graceful-unicorn`，其中 `myhost` 是您的机器的主机名或您使用 `--remote-control-session-name-prefix` 设置的前缀

如果您没有设置显式名称，一旦您发送提示，Claude Code 会更新标题以反映您的提示。Claude Code 将自动生成的标题与您的对话语言相匹配，或与配置的 [`language`](/docs/zh-CN/settings-reference#language) 设置相匹配；语言匹配需要 Claude Code v2.1.176 或更高版本。

当您从 claude.ai 或 Claude 应用重命名会话时，Claude Code 也会更新在 `claude --resume` 中显示的本地标题。Claude Code 将相同的重命名应用于提示栏上显示的会话名称，以及当会话[在后台运行](/docs/zh-CN/agent-view)时 `claude agents` 列表中显示的会话名称。在 v2.1.221 之前，从 claude.ai 或 Claude 应用中的会话列表重命名仅更新标题，CLI 保留其以前的会话名称；`/rename`（在 CLI 本身中运行）在任何版本上设置名称。

如果您还没有 Claude 应用，请在 Claude Code 中使用 `/mobile` 命令显示 [iOS](https://apps.apple.com/us/app/claude-by-anthropic/id6473753684) 或 [Android](https://play.google.com/store/apps/details?id=com.anthropic.claude) 的下载 QR 码。

<h3 id="what-connected-devices-see">
  连接的设备看到什么
</h3>

连接的设备会在您的终端中实时显示对话。这些情况超出了普通消息：

* **压缩和 `/clear`**：当 Claude Code [压缩对话](/docs/zh-CN/context-window#what-survives-compaction)时，连接的设备显示进度，然后显示对话被压缩的位置。当您运行 `/clear` 时，对话也会在连接的设备上重置。
* **使用 `/resume` 切换对话**：连接的设备不会接收切换到的对话的标题或早期历史记录，但双向的新消息会进出您的终端中打开的任何对话。要再次从设备处理原始对话，请在您的终端中运行 `/resume` 并切换回它。
* **使用 `/teleport` 拉取会话**：当您使用 `/teleport` 将[Claude Code on the web 会话](/docs/zh-CN/claude-code-on-the-web#from-web-to-terminal)拉入您的终端时，连接的设备不会接收拉取的对话的早期历史记录。双向的新消息会进出拉取的对话，这现在是您的终端中打开的对话。
* **来自您其他会话的消息**：使用[跨会话消息传递](/docs/zh-CN/cross-session-messaging)，相同的连接在您不同机器上的自己的会话之间以及来自您的[Claude Code on the web](/docs/zh-CN/claude-code-on-the-web) 会话的消息，通过 Anthropic 服务器，就像其余 Remote Control 流量一样。[在其他机器上的消息会话](/docs/zh-CN/cross-session-messaging#message-sessions-on-other-machines)涵盖传递规则，[控制入站消息](/docs/zh-CN/cross-session-messaging#control-inbound-messages)涵盖入站控制。需要 Claude Code v2.1.224 或更高版本。
* **您在回合中途发送的提示**：当您在当前回合结束之前从连接的设备发送提示时，Claude Code 会将其排队并在该回合完成后将其保留在设备的记录中。
* **您的更改的差异**：当会话的目录在 git 存储库中时，连接的设备的差异窗格显示您未提交更改的差异。设备通过连接请求差异，Claude Code 在您的机器上计算它。当您的工作树是干净的时，Claude Code 改为提供您的分支自从它从默认分支分叉以来的更改。在 v2.1.247 之前，Claude Code 仅向由 `claude remote-control` 提供的会话中的连接设备报告差异。
* **模型**：当您从连接的设备选择[模型](/docs/zh-CN/model-config)时，Claude Code 在该模型上运行会话。终端的 `/model` 选择器、`/status` 和 `/config` 显示该模型。需要 Claude Code v2.1.238 或更高版本。
  * 您从设备的模型控制中选择的模型仅适用于当前会话。当您从设备向交互式会话发送 `/model <name>` 时，Claude Code 也会为新会话设置您的默认值。
  * 如果您发送 Claude Code 无法识别的名称，例如预期模型 ID 的显示名称，Claude Code [拒绝选择](/docs/zh-CN/errors#model-is-not-a-recognized-model-id)，会话保留其当前模型。在 v2.1.260 之前，Claude Code 从设备的模型控制中保存无法识别的选择，您的下一条消息失败。
* **努力级别**：当您从连接的设备使用 `/effort` 或设备的努力控制设置[努力级别](/docs/zh-CN/model-config#adjust-effort-level)时，Claude Code 将其应用于您机器上的会话，claude.ai/code 显示会话正在使用的级别。如果您使用 `CLAUDE_CODE_EFFORT_LEVEL` 固定了一个级别，会话保留该级别，Claude Code 拒绝从努力控制中选择不同的级别。从努力控制中选择级别需要您机器上的 Claude Code v2.1.234 或更高版本。
* **连接失败后重新连接**：运行 `/remote-control` 以重新连接。如果压缩重写了对话或您在此期间使用 `/resume` 切换了对话，Claude Code 会存档它正在使用的服务器会话，而不是将其保留在会话列表中。您仍然可以通过[过滤存档的会话](/docs/zh-CN/claude-code-on-the-web#archive-sessions)找到它。在设备仍然连接时切换对话不会存档会话。

<h3 id="enable-remote-control-for-all-sessions">
  为所有会话启用 Remote Control
</h3>

Remote Control 仅在您显式运行 `claude remote-control`、`claude --remote-control` 或 `/remote-control` 时激活，除非自动连接已打开。要为每个交互式会话打开自动连接，请在 Claude Code 中运行 `/config` 并设置**为所有会话启用 Remote Control**。切换有三个值：

* **`true`**：当交互式会话启动时自动连接。
* **`false`**：关闭自动连接，尽管来自[托管设置](/docs/zh-CN/managed-settings)的 `true` 会优先，因为 Claude Code 将选择保存到您的用户设置。项目或本地设置（`.claude/settings.json`、`.claude/settings.local.json`）中的 `false` 甚至会关闭自动连接，即使托管 `true` 也是如此。
* **`default`**：清除您的选择并遵循您的组织的管理员默认值（如果已设置），否则遵循 Claude Code 的当前默认值。

相同的切换出现在 CLI 之外：

* **桌面应用**：**设置 > Claude Code > 默认启用远程控制**。
* **VS Code 扩展**：[命令菜单](/docs/zh-CN/vs-code#use-the-prompt-box)的设置部分中的**为所有会话启用 Remote Control**。需要 Claude Code v2.1.203 或更高版本。

要从设置文件改为打开自动连接，请在您的用户 `~/.claude/settings.json` 或[托管设置](/docs/zh-CN/managed-settings)中将 [`remoteControlAtStartup`](/docs/zh-CN/settings-reference#remotecontrolatstartup) 设置为 `true`。在项目或本地设置（`.claude/settings.json`、`.claude/settings.local.json`）中，Claude Code 遵守 `false` 并为该存储库关闭自动连接，但忽略 `true`，因此签入的文件无法为打开存储库的每个人打开 Remote Control。

自动连接使用您自己的 claude.ai 帐户登录，因此它启动的会话仅出现在您自己的帐户的 Claude 应用中，并且不向任何其他人授予访问权限。

启用此设置后，每个交互式 Claude Code 进程注册一个远程会话。如果您运行多个实例，每个实例都获得自己的远程会话。要从单个进程运行多个并发会话，请改用[服务器模式](#start-a-remote-control-session)。

<h3 id="resume-sessions-after-stopping-the-server">
  停止服务器后恢复会话
</h3>

当您使用 Ctrl+C 停止 `claude remote-control` 时，它提供的会话停止从您的手机或浏览器响应。只要您没有在同一目录中运行另一个 `claude remote-control` 并且没有使用 `--no-create-session-in-dir` 启动此会话，Claude Code 就不会存档它们。要将它们恢复，请在同一目录中运行以下命令之一：

* **`claude remote-control`**：恢复服务器提供的每个会话。
* **`claude remote-control --continue`**：仅恢复服务器启动的会话，并在该会话结束时退出。如果此目录没有记录，Claude Code 会使用此存储库的其他 git worktrees 中最新的。
* **`claude remote-control --session-id <id>`**：仅恢复您传递的 ID 的会话，并在该会话结束时退出。ID 是会话 URL 在 claude.ai/code 中 `/code/` 和任何 `?` 之间的部分。

这些命令在服务器停止后约四小时内有效。之后，运行 `claude remote-control` 以启动新会话。如果您在此期间存档了会话，`--continue` 和 `--session-id` 会在 Claude Code v2.1.228 或更高版本上取消存档它。

要恢复您使用 `claude --remote-control` 或 `/remote-control` 启动的会话，请使用 `claude --continue` 或 `claude --resume` 恢复对话。Claude Code 是否重新连接以及连接到哪个会话取决于对话的[重新连接记录](#resume-outcomes)。

如果您在第二个终端中恢复对话，而第一个终端仍然打开 Remote Control，Claude Code 会在第二个终端中打印通知，并改为在那里关闭 Remote Control，而不是从第一个终端接管会话。当 Remote Control 在那里保持关闭时，该终端中的 Claude 看不到[您在其他机器上的会话](/docs/zh-CN/cross-session-messaging#see-which-sessions-claude-can-reach)，它们也无法到达它。在第二个终端中运行 `/remote-control` 以将 Remote Control 移到它。

当您在曾经打开 Remote Control 的 Claude Desktop 或 IDE 扩展中恢复对话时，Claude Code 会将其重新附加到现有的 claude.ai 会话，而不是向会话列表添加新会话。

<h2 id="connection-and-security">
  连接和安全
</h2>

您的本地 Claude Code 会话仅发出出站 HTTPS 请求，从不在您的机器上打开入站端口。当您启动 Remote Control 时，它向 Anthropic API 注册并轮询工作。当您从另一个设备连接时，服务器通过流连接在网络或移动客户端和您的本地会话之间路由消息。

所有流量都通过 Anthropic API 通过 TLS 传输，与任何 Claude Code 会话的传输安全相同。连接使用多个短期凭证，每个凭证的范围限定为单一目的并独立过期。

Remote Control 连接时，会话记录（包括您的消息、Claude 的响应和工具活动）存储在 Anthropic 服务器上。存储的记录保持您的设备之间的对话同步，并让会话在网络中断后重新连接。执行和文件系统访问保留在您的机器上，存储的记录根据[数据使用](/docs/zh-CN/data-usage)政策保留。

要完全关闭 Remote Control，请使用 [`disableRemoteControl`](/docs/zh-CN/settings-reference#disableremotecontrol) 设置。具有零数据保留等合规要求的组织无法启用 Remote Control。

<h2 id="trusted-devices">
  受信任的设备
</h2>

<Note>
  受信任的设备目前处于测试阶段。功能和特性可能会随着体验的完善而演变。

  受信任的设备在 Team 和 Enterprise 计划中可用。在所有者启用它之前，它默认处于关闭状态。
</Note>

受信任的设备是一个组织范围的设置，要求成员在从 claude.ai、Claude 移动应用或 Claude Desktop 查看或控制 Remote Control 会话之前验证其设备。它将 Remote Control 访问权限与已知设备和最近的身份验证绑定，而不仅仅是已登录的账户。

当设置打开时，与 Remote Control 会话交互需要以下两项：

* **已注册的设备**：成员用于 Remote Control 的每个浏览器、手机或桌面应用都会注册自己的凭证。注册仅在完整登录后不久提供，因此设备作为真实身份验证的一部分加入受信任列表，而不是在后台静默加入。
* **最近的登录**：成员的登录不能超过 18 小时。成员不需要每天重新登录，而是使用 Face ID、Touch ID、Windows Hello 或通行密钥确认存在。此生物识别步骤立即刷新会话。

生物识别检查通过操作系统或浏览器在设备上运行，与通行密钥登录的机制相同。Anthropic 从不接收或存储指纹、面部数据或任何其他生物识别信息。仅存储设备的公钥和基本元数据，如显示名称、平台和注册时间。

该设置仅适用于 Remote Control。常规 Claude 聊天、终端中的 Claude Code 和 API 使用不受影响。

<h3 id="enable-trusted-devices-for-your-organization">
  为您的组织启用受信任的设备
</h3>

所有者从 Claude Code 管理员控制台启用该设置。

<Steps>
  <Step title="打开 Claude Code 管理员设置">
    转到 [claude.ai/admin-settings/claude-code](https://claude.ai/admin-settings/claude-code)。**需要受信任的设备**切换出现在 Remote Control 设置下方。
  </Step>

  <Step title="打开需要受信任的设备">
    该设置适用于组织的每个成员以及在您启用它后启动的 Remote Control 会话。在切换打开之前已经运行的会话不会被追溯保护，并继续运行而不需要设备要求，直到它们结束。不提供按团队或按项目的范围。
  </Step>

  <Step title="告诉成员期望什么">
    在启用该设置后，成员第一次从浏览器、手机或桌面应用查看或控制新的 Remote Control 会话时，系统会提示他们注册该设备。提前告知他们可以避免混淆。
  </Step>
</Steps>

<h3 id="what-members-see">
  成员看到什么
</h3>

注册是每个设备的一次性步骤。之后，唯一可见的变化是偶尔的生物识别提示。

* **首次在每个设备上使用**：成员被要求注册。如果他们的登录不是最近的，他们首先通过您的正常流程登录，包括配置的 SSO，然后确认注册。
* **日常使用**：拥有已注册设备和最近登录的成员看不到任何提示。当登录超过 18 小时时，下一次 Remote Control 交互会显示单个 Face ID、Touch ID、Windows Hello 或通行密钥提示。
* **未注册的设备**：Remote Control 会话无法查看或控制，直到设备被注册。该设备上的常规 Claude 聊天不受影响。
* **没有平台身份验证器**：在没有 Face ID、Touch ID 或 Windows Hello 的机器上的成员可以使用硬件安全密钥，或重新登录而不是升级。
* **在终端中**：运行 Claude Code 的机器在开发人员登录到 CLI 时自动接收自己的凭证。终端中没有单独的注册步骤。

<h3 id="manage-enrolled-devices">
  管理已注册的设备
</h3>

成员可以从账户设置中查看和撤销自己的设备。

打开 [claude.ai/settings/account](https://claude.ai/settings/account#trusted-devices) 并找到**受信任的设备**部分，查看每个已注册设备及其名称、平台和注册日期。删除设备会立即撤销其凭证，设备可以在新登录后重新注册。凭证如果不续期也会自动过期，因此未使用的设备会自动从受信任列表中删除。

对于丢失或被盗的设备，成员从此页面删除它。如果成员无法登录，管理员可以在管理员控制台中使用**到处登出**为该成员撤销每个会话和已注册设备，之后成员重新注册他们仍然持有的设备。

<h2 id="remote-control-vs-claude-code-on-the-web">
  Remote Control 与网络上的 Claude Code 的比较
</h2>

Remote Control 和[网络上的 Claude Code](/docs/zh-CN/claude-code-on-the-web)都使用 claude.ai/code 界面。关键区别在于会话运行的位置：Remote Control 在您的机器上执行，因此您的本地 MCP servers、工具和项目配置保持可用。网络上的 Claude Code 在云中执行。

当您处于本地工作中间并想从另一个设备继续时，使用 Remote Control。当您想在没有任何本地设置的情况下启动任务、处理您没有克隆的存储库或并行运行多个任务时，使用网络上的 Claude Code。

<h2 id="mobile-push-notifications">
  移动推送通知
</h2>

当 Remote Control 处于活动状态时，Claude 可以向您的手机发送推送通知。

Claude 决定何时推送。它通常在长时间运行的任务完成或需要您的决定来继续时发送一个。您也可以在提示中请求推送，例如 `notify me when the tests finish`。除了下面的两个开/关切换外，没有按事件配置。

要设置移动推送通知：

<Steps>
  <Step title="安装 Claude 移动应用">
    下载 Claude 应用（[iOS](https://apps.apple.com/us/app/claude-by-anthropic/id6473753684) 或 [Android](https://play.google.com/store/apps/details?id=com.anthropic.claude)）。
  </Step>

  <Step title="使用您的 Claude Code 账户登录">
    使用您在终端中用于 Claude Code 的相同账户和组织。
  </Step>

  <Step title="允许通知">
    接受来自操作系统的通知权限提示。
  </Step>

  <Step title="在 Claude Code 中启用推送">
    在您的终端中，运行 `/config` 并启用**当 Claude 决定时推送**以获取主动通知，启用**当需要操作时推送**以获取权限提示和问题，或两者都启用。
  </Step>
</Steps>

如果通知没有到达：

* 如果 `/config` 显示**未注册移动设备**，请在您的手机上打开 Claude 应用，以便它可以刷新其推送令牌。下次 Remote Control 连接时，警告会清除。
* 在 iOS 上，焦点模式和通知摘要可能会抑制或延迟推送。检查设置 → 通知 → Claude。
* 在 Android 上，激进的电池优化可能会延迟传递。在系统设置中将 Claude 应用从电池优化中豁免。

Claude Code 在您在连接的终端中输入或专注时会跳过移动推送通知。从 v2.1.181 开始，您可以将 [`CLAUDE_CLIENT_PRESENCE_FILE`](/docs/zh-CN/env-vars) 设置为标记文件路径，以将其扩展到您在机器上的任何时间，即使在另一个窗口中：当文件存在时，通知会被跳过。配置屏幕锁定侦听器或类似工具，以在屏幕解锁时创建文件，在屏幕锁定时删除文件。

<h2 id="limitations">
  限制
</h2>

* **每个交互式进程一个远程会话**：在服务器模式之外，每个 Claude Code 实例一次支持一个远程会话。使用[服务器模式](#start-a-remote-control-session)从单个进程运行多个并发会话。
* **本地进程必须保持运行**：Remote Control 作为本地进程运行。如果您关闭终端、退出 VS Code 或以其他方式停止 `claude` 进程，会话将离线，直到您[恢复它](#resume-sessions-after-stopping-the-server)。除非 Claude 正在执行任务中，否则 claude.ai 和 Claude 应用会在进程退出后的几秒内显示会话离线。要在从 SSH 断开连接后保持远程机器上的会话运行，请在 `tmux` 或 `screen` 内启动它。
* **服务器模式中的崩溃会话**：如果由 `claude remote-control` 提供的会话崩溃，请从连接的设备向其发送消息。Claude Code 将再次提供它。您不必重新启动服务器。需要 Claude Code v2.1.238 或更高版本。
* **已连接会话上的 HTTP 403 拒绝**：一旦交互式会话连接，当您的机器和 Anthropic 服务器之间的某些内容以 HTTP 403 响应时（在 VPN 或网络更改后可能发生），Claude Code 会重试最多三分钟。如果拒绝持续更长时间，Claude Code 会断开连接，原因会说明是什么拒绝了：网络边缘，或您自己网络上的代理、VPN 或防火墙。
* **扩展网络中断**：如果您的机器处于唤醒状态但无法到达网络，接下来的操作取决于模式：
  * **服务器模式**：Claude Code 在大约 10 分钟后放弃，`claude remote-control` 进程退出。再次运行 `claude remote-control` 以启动新会话。
  * **交互式会话**：继续在本地工作。Claude Code 会在中断期间持续重试，并在网络恢复时自动重新连接。
* **存在心跳失败**：如果交互式会话断开连接并显示 `could not reach the Remote Control server for about 30 minutes`，运行 `/remote-control` 以重新连接。Claude Code 仅在会话的存在心跳失败而其余连接保持正常时显示此消息；它在大约 30 分钟后重新注册会话，然后断开连接。
* **转发的对话过期**：Claude Code 会保持权限提示和 `AskUserQuestion` 问题打开，直到您回答。当 Claude Code 将另一种对话转发到远程会话时，例如安全拒绝后显示的模型选择提示，默认情况下它会等待五分钟，然后关闭对话并继续使用对话的无操作默认值。设置 [`dialogExpiry`](/docs/zh-CN/settings-reference#dialogexpiry) 以调整或禁用截止时间。需要 Claude Code v2.1.224 或更高版本。
* **Fable 使用额度同意提示未被转发**：Claude Code 仅在会话运行的位置显示中途[Fable 使用额度同意提示](/docs/zh-CN/model-config#fable-and-usage-credits)，而不是在您的设备上。当会话在终端中运行且那里没有人在 Claude Code 关闭提示之前回答时，该轮结束而不发送请求；请参阅[确认提示未被回答](/docs/zh-CN/errors#the-prompt-to-confirm-went-unanswered)。
* **某些命令仅限本地**：仅在终端界面中运行的命令，例如 `/plugin` 或 `/resume`，仅从本地 CLI 工作，无论您是否传递参数。以下命令可从移动和网络工作：
  * 文本输出命令：`/compact`、`/clear`、`/context`、`/usage`、`/exit`、`/usage-credits`、`/recap` 和 `/reload-plugins`。`/usage-credits` 打印计费 URL 而不是打开浏览器。`/reload-plugins` 仅在会话在交互式终端中运行时工作；没有交互式终端的会话会拒绝它。
  * `/model`、`/effort`、`/fast`、`/color` 和 `/rename`：将值作为参数传递，例如 `/model sonnet` 或 `/effort high`。从移动和网络，`/model` 和 `/effort` 在终端选择器或滑块的位置接受参数。
  * `/mcp`：从移动应用，返回服务器状态的文本摘要而不是打开选择器。在网络上，`/mcp` 单独打开 [claude.ai 连接器](/docs/zh-CN/mcp#use-mcp-servers-from-claude-ai) 的目录而不是返回摘要。`reconnect`、`enable` 和 `disable` [子命令](/docs/zh-CN/commands#all-commands)可从两者工作。与本地 CLI 不同，不带服务器名称的 `/mcp reconnect` 会重新连接每个已失败或需要身份验证的服务器。
  * `/config`，从 v2.1.181 开始：从移动应用，传递 `key=value` 以设置一个设置，或不带参数运行它以列出您可以设置的键。在网络上，`/config` 打开您设置的 Claude Code 部分，并忽略命令后的文本。
  * 在 Team 和 Enterprise 上，从移动或网络运行的 `/usage-credits` 不会向您的管理员发送[使用额度请求](/docs/zh-CN/costs#add-usage-credits-to-your-subscription)。发送需要仅在交互式 CLI 中出现的确认，因此命令告诉您改为在那里运行它。在 v2.1.211 之前，文本形式在没有确认的情况下发送请求。
  * `/autocompact`，从 v2.1.221 开始：将窗口大小作为参数传递，例如 `/autocompact 500k`。不带参数时，它将当前窗口大小打印为文本，而不是打开命令在终端会话中显示的对话框。
  * `/advisor`，从 v2.1.260 开始：将模型作为参数传递，例如 `/advisor opus`，或传递 `off` 以关闭顾问。两种形式都仅适用于当前会话，并保持您保存的默认值不变。不带参数时，它将当前顾问打印为文本，而不是打开选择器。

<h2 id="troubleshooting">
  故障排除
</h2>

<h3 id="remote-control-requires-a-claude-ai-subscription">
  "Remote Control 需要 claude.ai 订阅"
</h3>

您未使用 claude.ai 账户进行身份验证，或另一个凭证优先于您的登录。该消息采用以下形式之一：

* 已登出，来自 `/remote-control` 或 `--remote-control`：`Remote Control requires a claude.ai subscription.`
* 已登出，来自 `claude remote-control`：`You must be logged in to use Remote Control. Remote Control is only available with claude.ai subscriptions.`
* 已登入，但正在使用 API 密钥或令牌：`Remote Control requires claude.ai subscription auth.` 后跟正在使用的凭证，例如 `ANTHROPIC_API_KEY is set, so this session is using API-key auth`。`apiKeyHelper` 设置和 `ANTHROPIC_AUTH_TOKEN` 的命名方式相同。

运行 `claude auth login` 并选择 claude.ai 选项。如果消息名称为 `ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN`，请在设置它的任何地方删除它：您的 shell 环境或[设置文件](/docs/zh-CN/settings-reference#env)的 `env` 块。如果它名称为 `apiKeyHelper`，请删除该设置。

在 v2.1.206 之前，在未登出的情况下运行 `/remote-control` 会报告 `Unknown command: /remote-control` 而不是此消息。

<h3 id="remote-control-requires-a-full-scope-login-token">
  "Remote Control 需要完整范围的登录令牌"
</h3>

您使用来自 `claude setup-token` 或 `CLAUDE_CODE_OAUTH_TOKEN` 环境变量的长期令牌进行身份验证。这些令牌仅限于进行模型请求，因此无法建立 Remote Control 会话。运行 `claude auth login` 以改用完整范围的会话令牌进行身份验证。

<h3 id="unable-to-determine-your-organization-for-remote-control-eligibility">
  "无法确定您的组织以进行 Remote Control 资格检查"
</h3>

您的缓存账户信息已过期或不完整。运行 `claude auth login` 以刷新它。

<h3 id="remote-control-isn’t-enabled-for-this-account">
  "Remote Control 尚未为此账户启用"
</h3>

Claude Code 检查了您登录的账户的 Remote Control 可用性，检查结果为关闭。通常的原因是缓存的权利在计划更改后已过期。运行 `claude auth logout` 然后 `claude auth login` 以刷新它们，如果您使用的是旧版本，请更新 Claude Code。

运行 `claude doctor` 以查看哪个单独的资格检查失败。环境变量冲突、无法到达的检查和您的组织的 Remote Control 设置各自产生自己的消息，因此此错误意味着账户级别的检查本身。

在 v2.1.239 之前，此消息读作"Remote Control is not yet enabled for your account"。在 v2.1.154 之前，禁用功能标志评估的变量（例如 `DISABLE_TELEMETRY` 或 `DO_NOT_TRACK`）也会产生此消息；下面的"Remote Control 需要功能标志评估"条目涵盖该配置。

<h3 id="couldn’t-verify-remote-control-eligibility">
  "无法验证 Remote Control 资格"
</h3>

Claude Code 无法到达功能标志服务以检查是否为您的账户启用了 Remote Control，通常是因为您离线或代理阻止了请求。一旦您有网络访问权限，请重试，或运行 `claude doctor` 以获取详细信息。相关消息"无法验证您的组织的 Remote Control 策略"具有相同的原因和相同的修复。这两条消息都在 v2.1.178 中添加。

<h3 id="remote-control-requires-feature-flag-evaluation">
  "Remote Control 需要功能标志评估"
</h3>

设置了以下变量之一：[`DISABLE_TELEMETRY`、`DO_NOT_TRACK`、`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 或 `DISABLE_GROWTHBOOK`](/docs/zh-CN/env-vars)。每个变量都禁用 Remote Control 可用性所依赖的功能标志评估，完整消息会命名 Claude Code 找到的变量。在设置它的任何地方取消设置该变量，在您的 shell 环境中或在 [`settings.json` 文件](/docs/zh-CN/settings-reference#all-settings)的 `env` 块中。在 2.1.154 之前的版本上，相同的配置会产生"Remote Control is not yet enabled for your account"。

<h3 id="remote-control-is-only-available-when-using-claude-via-api-anthropic-com">
  "Remote Control 仅在通过 api.anthropic.com 使用 Claude 时可用"
</h3>

该会话不是直接与 Anthropic API 通信，因此没有 claude.ai 后端可配对。这发生在 Amazon Bedrock、Google Cloud 的 Agent Platform 和 Microsoft Foundry 上。当 [`ANTHROPIC_BASE_URL`](/docs/zh-CN/env-vars) 指向 `api.anthropic.com` 以外的主机时，例如 [LLM 网关](/docs/zh-CN/llm-gateway) 或代理，即使您使用 claude.ai 登录，也会发生这种情况。在 v2.1.196 之前，Claude Code 对于自定义 `ANTHROPIC_BASE_URL` 不显示此消息。有关完整原因列表，请参阅[错误参考](/docs/zh-CN/errors#remote-control-requires-the-anthropic-api)。

该消息会命名将会话路由离开 Anthropic API 的内容，例如 `CLAUDE_CODE_USE_BEDROCK` 或自定义 `ANTHROPIC_BASE_URL`。如果您有符合条件的 claude.ai 登录，请取消设置命名的变量，如果您在那里设置了它，请从[设置](/docs/zh-CN/settings)中的 `env` 密钥中删除它，然后重启会话。在 v2.1.219 之前，该消息仅是本节标题中的句子，因此在较旧的版本上，请自己检查您的环境以查找提供程序变量，例如 `CLAUDE_CODE_USE_BEDROCK` 和 `CLAUDE_CODE_USE_VERTEX`，以及 `ANTHROPIC_BASE_URL`。

<h3 id="remote-control-is-disabled-by-your-organization’s-policy">
  "Remote Control 被您的组织的策略禁用"
</h3>

策略阻止 Remote Control，或 Claude Code 无法在此机器上加载您的组织的策略，同时保持 Remote Control 关闭。按顺序检查这些原因：

* **错误提及 `disableRemoteControl`**：您的 IT 管理员已通过[托管设置](/docs/zh-CN/managed-settings)在此设备上禁用了 Remote Control，独立于组织范围的切换和您的登录方式。
* **您的 claude.ai 计划是 Pro 或 Max**：Claude Code 仍然以来自较早登录的 Team 或 Enterprise 组织身份登录，因此它检查该组织的 Remote Control 策略。运行 `/status` 以查看您的登录使用的计划和组织。运行 `claude auth logout` 然后 `claude auth login` 以在您当前的计划下重新登录。
* **组织策略未在此机器上加载**：运行 `claude doctor` 并阅读 `Organization policy` 行。如果该行显示策略未加载，这就是保持 Remote Control 关闭的原因。在 v2.1.261 之前，`claude doctor` 不打印此行。
* **消息未说联系您的组织管理员**：您的组织有与 Remote Control 不兼容的 HIPAA 配置，`/status` 在其 `Compliance` 行中列出 `HIPAA`。在这种状态下，管理面板的 Remote Control 切换呈灰色，因此所有者无法在那里更改它。联系 Anthropic 支持以讨论选项。在 v2.1.267 之前，这种情况显示"Remote Control isn't available for your organization due to its compliance policy"。
* **否则，所有者尚未为您的组织启用它**：Remote Control 在 Team 和 Enterprise 计划上默认处于关闭状态。所有者可以在 [claude.ai/admin-settings/claude-code](https://claude.ai/admin-settings/claude-code) 通过打开 **Remote Control** 切换来启用它。此切换是服务器端组织设置。

<h3 id="remote-credentials-fetch-failed">
  "Remote credentials fetch failed"
</h3>

Claude Code 无法从 Anthropic API 获取短期凭证以建立连接。使用 `--verbose` 重新运行以查看完整错误：

```bash theme={null}
claude remote-control --verbose
```

常见原因：

* 未登录：运行 `claude` 并使用 `/login` 使用您的 claude.ai 账户进行身份验证。Remote Control 不支持 API 密钥身份验证。
* 网络或代理问题：防火墙或代理可能阻止出站 HTTPS 请求。Remote Control 需要访问端口 443 上的 Anthropic API。
* 会话创建失败：如果您还看到 `Session creation failed — see debug log`，失败发生在设置的早期。检查您的订阅是否处于活动状态。

过期的登录令牌不会导致此错误。当 Anthropic API 拒绝保存的令牌时，例如因为另一个 Claude Code 进程已经刷新了它，Claude Code 会刷新令牌并自动重试。在 v2.1.224 之前，过期的令牌会导致 Remote Control 启动失败并显示此消息，因此设置为[自动连接](#enable-remote-control-for-all-sessions)的会话可能在启动时间歇性失败。

<h3 id="couldn’t-reconnect-to-your-remote-control-session">
  "无法重新连接到您的 Remote Control 会话"
</h3>

当您使用 `claude --resume` 或 `claude --continue` 恢复对话时，Claude Code 会重新连接到该对话中记录的 Remote Control 会话。此消息意味着重新连接因可能是临时的原因（例如网络中断或服务器错误）而失败，因此 Claude Code 无法确认远程会话是否仍然存在。

运行 `/remote-control` 以重试连接，或使用 `claude --remote-control` 启动新会话以创建新的 Remote Control 会话。您的本地会话继续运行而不使用 Remote Control。

<span id="resume-outcomes" />恢复时，您也可以获得以下结果之一而不是此消息：

* **服务器报告记录的会话已消失，或重新连接记录命名不同的账户**：Claude Code 按照对话的重新连接记录所说的内容进行操作：
  * **记录命名您登录的账户**：Claude Code 使用自动生成的名称启动替换会话，并将对话的早期消息排除在外。例如，在您从 claude.ai 或 Claude 应用中删除会话后，您会看到这种情况。
  * **记录命名不同的账户**：Claude Code 启动新会话而不显示消息，无论记录的会话是否仍然存在，都不包括对话的早期消息。
  * **记录未说明哪个账户拥有会话，或 Claude Code 无法读取您保存的登录**：Claude Code 显示 [`Previous session is unavailable — run /remote-control to start a new one`](#previous-session-is-unavailable) 而不是此消息，不启动任何内容，并从对话中删除记录。
* **您在恢复前关闭了 Remote Control**：除非托管 Claude Code 的应用已告诉它该应用拥有 claude.ai 会话，否则当您从 CLI 的[状态面板](#check-connection-status)、VS Code 扩展或基于[Agent SDK](/docs/zh-CN/agent-sdk/overview) 构建的主机关闭 Remote Control 时，Claude Code 删除了重新连接记录，因此它不会重新连接。当拥有的应用关闭它时，Claude Code 保留记录并重新连接。
* **此机器上的另一个 Claude Code 仍然拥有会话**：您会看到以 `Remote Control not started here` 开头的通知，Claude Code [在恢复的会话中保持 Remote Control 关闭](#resume-sessions-after-stopping-the-server)。在那里运行 `/remote-control` 以移动它。

<span id="reconnect-history" />在 v2.1.232 之前，当服务器报告记录的会话已消失时，Claude Code 的响应不同。从 v2.1.227 到 v2.1.231，Claude Code 拒绝启动替换，即使记录与您的账户匹配。在 v2.1.226 之前，Claude Code 启动替换，无论记录是否与您的账户匹配，在 v2.1.224 到 v2.1.226 中，在该机器上登录的账户下创建它，从不是另一个账户的，不上传对话的早期消息到它。在 v2.1.200 之前，Claude Code 在任何重新连接失败后创建新会话。

<h3 id="previous-session-is-unavailable">
  "Previous session is unavailable — run /remote-control to start a new one"
</h3>

Claude Code 无法恢复之前的 Remote Control 会话，而是停止而不是自动启动新会话。在您使用 `claude --resume` 或 `claude --continue` 恢复对话后，或在 Claude Code [在断开连接后自动重新连接](/docs/zh-CN/errors#remote-control-couldnt-refresh-your-login)后，您可能会看到此消息。

运行 `/remote-control` 以在当前登录下启动新的 Remote Control 会话；您的本地会话继续运行而不使用 Remote Control。相关消息 `Remote Control could not verify the signed-in account — run /remote-control to reconnect` 具有相同的修复；当登录的账户在验证和重新连接之间更改或无法读取时，Claude Code 显示它。如果您在不首先重启 Claude Code 的情况下在 `Previous session is unavailable` 后运行 `/remote-control`，Claude Code 会将对话的早期消息排除在新会话之外。

在恢复时，Claude Code [仅在对话的重新连接记录命名拥有会话的账户时才启动新会话](#resume-outcomes)，因为服务器以相同的方式报告您删除的会话和由另一个账户拥有的会话。v2.1.227 之前的 Claude Code 没有记录该账户，当 Claude Code 无法读取您保存的登录时，它无法检查记录。v2.1.232 之前的 Claude Code 显示 `Remote Control could not resume the previous session under the current login — run /remote-control to start fresh` 而不是，在[不同的情况集](#reconnect-history)中。

<h3 id="remote-control-got-an-unexpected-server-response">
  "Remote Control got an unexpected server response"
</h3>

Remote Control 服务器接受了请求，但以此版本的 Claude Code 无法读取的形式回复，同时创建远程会话或获取其凭证。在同一版本上重试会以相同的方式失败。运行 `claude update`，然后运行 `/remote-control` 以重新连接。此消息在 v2.1.225 中添加。

<h3 id="your-organization-requires-trusted-devices-for-remote-control-but-this-device-is-not-enrolled">
  "您的组织需要受信任的设备用于 Remote Control，但此设备未注册"
</h3>

您的组织已[启用受信任的设备](#trusted-devices)，此机器尚未注册。在 Claude Code 中运行 `/login`。注册作为登录的一部分进行，没有单独的注册命令。

<h3 id="session-expired-for-trusted-device-check">
  "session expired for trusted-device check"
</h3>

您的登录已超过 18 小时。在 Claude Code 中运行 `/login`，或在 claude.ai 或移动应用提示您时使用 Face ID、Touch ID、Windows Hello 或通行密钥确认。请参阅[受信任的设备](#trusted-devices)。

<h2 id="choose-the-right-approach">
  选择正确的方法
</h2>

Claude Code offers several ways to work when you're not at your terminal. They differ in what triggers the work, where Claude runs, and how much you need to set up.

|                                                          | Trigger                                                                                        | Claude runs on                                                                               | Setup                                                                                                                                | Best for                                                      |
| :------------------------------------------------------- | :--------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------ |
| [Dispatch](/docs/en/desktop#sessions-from-dispatch)           | Message a task from the Claude mobile app                                                      | Your machine (Desktop)                                                                       | [Pair the mobile app with Desktop](https://support.claude.com/en/articles/13947068)                                                  | Delegating work while you're away, minimal setup              |
| [Remote Control](/docs/en/remote-control)                     | Drive a running session from [claude.ai/code](https://claude.ai/code) or the Claude mobile app | Your machine (CLI or VS Code)                                                                | Run `claude remote-control`                                                                                                          | Steering in-progress work from another device                 |
| [Channels](/docs/en/channels)                                 | Push events from a chat app like Telegram or Discord, or your own server                       | Your machine (CLI)                                                                           | [Install a channel plugin](/docs/en/channels#quickstart) or [build your own](/docs/en/channels-reference)                                      | Reacting to external events like CI failures or chat messages |
| [Slack](/docs/en/slack)                                       | Mention `@Claude` in a team channel                                                            | Anthropic cloud                                                                              | [Install the Slack app](/docs/en/slack#setting-up-claude-code-in-slack) with [Claude Code on the web](/docs/en/claude-code-on-the-web) enabled | PRs and reviews from team chat                                |
| [Self-hosted environments](/docs/en/self-hosted-environments) | Start a [cloud session](/docs/en/claude-code-on-the-web) and pick your organization's environment   | Your organization's infrastructure                                                           | [Deploy runners](/docs/en/self-hosted-environments-quickstart), on Team and Enterprise plans                                              | Cloud sessions that must run inside your network              |
| [Scheduled tasks](/docs/en/scheduled-tasks)                   | Set a schedule                                                                                 | [CLI](/docs/en/scheduled-tasks), [Desktop](/docs/en/desktop-scheduled-tasks), or [cloud](/docs/en/routines) | Pick a frequency                                                                                                                     | Recurring automation like daily reviews                       |

<h2 id="related-resources">
  相关资源
</h2>

* [网络上的 Claude Code](/docs/zh-CN/claude-code-on-the-web)：在云中运行会话而不是在您的机器上，通过[云环境](/docs/zh-CN/cloud-environments)配置
* [跨会话消息传递](/docs/zh-CN/cross-session-messaging)：让 Claude 在其他机器上或在[网络上的 Claude Code](/docs/zh-CN/claude-code-on-the-web) 上向您的会话发送消息
* [Channels](/docs/zh-CN/channels)：将 Telegram、Discord 或 iMessage 转发到会话中，以便 Claude 在您离开时对消息做出反应
* [Dispatch](/docs/zh-CN/desktop#sessions-from-dispatch)：从您的手机发送任务消息，它可以生成 Desktop 会话来处理它
* [身份验证](/docs/zh-CN/authentication)：设置 `/login` 并管理 claude.ai 的凭证
* [CLI 参考](/docs/zh-CN/cli-reference)：包括 `claude remote-control` 的标志和命令的完整列表
* [安全](/docs/zh-CN/security)：Remote Control 会话如何适应 Claude Code 安全模型
* [数据使用](/docs/zh-CN/data-usage)：在本地和远程会话期间通过 Anthropic API 流动的数据
