> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 选择权限模式

> 控制 Claude 在采取行动前是否需要征求您的同意。在 CLI 中使用 Shift+Tab 切换权限模式，在 VS Code 中使用模式指示器，或在 Desktop 中使用模式选择器。

权限模式设置 Claude 在会话中可以在不先询问您的情况下执行哪些操作。在 Manual 模式下，Claude Code 会在大多数编辑文件、运行 shell 命令或访问网络的操作前停止并询问您。在[自动模式](#eliminate-prompts-with-auto-mode)中，第二个模型（分类器）会审查操作而不是您；[分类器如何评估操作](#how-the-classifier-evaluates-actions)列出了它审查的操作以及哪些跳过它。

在 Pro、Max 和 Team 计划上，内置的起始权限模式是自动模式。[会话在哪个模式下启动](#which-mode-a-session-starts-in)涵盖了改变起始权限模式的表面和设置。您也可以随时更改正在运行的会话的权限模式。

<h2 id="available-modes">
  可用模式
</h2>

每种模式在便利性和监督之间做出不同的权衡。下表显示了在每种模式下 Claude 无需权限提示即可执行的操作。Manual 模式显示在其配置值 `default` 下。

| 模式                                                                  | 无需询问即可运行                                                   | 最适合           |
| :------------------------------------------------------------------ | :--------------------------------------------------------- | :------------ |
| `default`                                                           | 仅读取                                                        | 自己审查每个操作，敏感工作 |
| [`acceptEdits`](#auto-approve-file-edits-with-acceptedits-mode)     | 读取、文件编辑和常见文件系统命令（`mkdir`、`touch`、`mv`、`cp` 等）              | 迭代审查的代码       |
| [`plan`](#analyze-before-you-edit-with-plan-mode)                   | 读取，加上当[自动模式](#eliminate-prompts-with-auto-mode)可用时分类器批准的命令 | 在更改前探索代码库     |
| [`auto`](#eliminate-prompts-with-auto-mode)                         | 所有操作，带有后台安全检查                                              | 长任务、减少提示疲劳    |
| [`dontAsk`](#allow-only-pre-approved-tools-with-dontask-mode)       | 仅预先批准的工具                                                   | 锁定的 CI 和脚本    |
| [`bypassPermissions`](#skip-all-checks-with-bypasspermissions-mode) | 所有操作                                                       | 仅限隔离容器和虚拟机    |

在 CLI 中、`claude --help` 中、VS Code 和 JetBrains 扩展中以及桌面应用中，审查每个操作的模式被命名为 **Manual**。其配置值为 `default`，这是 hooks 和 SDK 集成使用的值。CLI 在任何地方都接受 `manual` 作为别名，例如 `claude --permission-mode manual` 或 `"defaultMode": "manual"`。Manual 标签和 `manual` 别名需要 Claude Code v2.1.200 或更高版本。桌面应用的标签不依赖于您的 CLI 版本。

对[受保护路径](#protected-paths)的写入永远不会自动批准，唯一的例外是 `bypassPermissions` 模式，以及可使用绕过权限的 plan 模式会话，也就是以[将 `bypassPermissions` 放入模式循环](#switch-permission-modes)的方式启动的会话。

模式设置基线。在顶部分层[权限规则](/docs/zh-CN/permissions#manage-permissions)以预先批准或阻止特定工具。拒绝规则在每种模式下都会阻止，包括 `bypassPermissions`。拒绝和询问规则不适用于 [`EndConversation`](/docs/zh-CN/tools-reference#endconversation-tool-behavior)，只要 Claude 仍然有至少一个其他工具可以调用。允许规则在 `bypassPermissions` 中无效。

<h3 id="actions-no-mode-auto-approves">
  任何模式都不会自动批准的操作
</h3>

Claude Code 在任何模式下都不会自动批准以下操作，包括 `bypassPermissions`。每个项目链接到说明在每种模式下会发生什么的部分：

* 与显式[询问规则](/docs/zh-CN/permissions#manage-permissions)匹配的工具
* 您的组织[设置为 `ask`](/docs/zh-CN/mcp#organization-controls-on-connector-tools) 的连接器工具，在该设置到达 Claude Code 的会话中
* 需要用户交互的工具：内置的 `AskUserQuestion` 工具和标记为 [`requiresUserInteraction`](/docs/zh-CN/mcp#require-approval-for-a-specific-tool) 的 MCP 工具
* `rm` 和 `rmdir` 移除针对[关键路径](#critical-paths)的操作，没有允许规则或 `PreToolUse` hook `"allow"` 批准
* [跨会话消息传递保障](#skip-all-checks-with-bypasspermissions-mode)
* 当 [`permissions.blockReadsOutsideWorkingDirectories`](/docs/zh-CN/settings-reference#permissions-blockreadsoutsideworkingdirectories) 打开时，在工作目录外读取：识别的文件读取 Bash 命令和任何[非沙箱化重试](/docs/zh-CN/sandboxing#the-unsandboxed-retry-escape-hatch)，即使在自动模式和 `bypassPermissions` 模式下也需要批准才能在沙箱外运行。需要 Claude Code v2.1.257 或更高版本

<h2 id="common-setups">
  常见设置
</h2>

权限模式决定 Claude 是否在操作前询问，[Bash 沙箱](/docs/zh-CN/sandboxing)和外部[隔离边界](/docs/zh-CN/sandbox-environments)决定操作运行后可以到达什么。下表将目标与获得该目标的标志或设置以及所需的隔离配对，作为起点。[可用模式](#available-modes)列出了在每种模式下无需提示即可运行的内容。

| 您想要              | 从以下开始                                                                                                                    | 需要的隔离                                                                                                                                           | 注意                                                                                                                                     |
| :--------------- | :----------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------- |
| 自己审查每个操作         | Manual 模式：`claude --permission-mode default`                                                                             | 无                                                                                                                                               | 敏感工作、不熟悉的代码                                                                                                                            |
| 在本地迭代，更少提示，无分类器  | Manual 模式加上 Bash 沙箱在[自动允许模式](/docs/zh-CN/sandboxing#sandbox-modes)：`claude --permission-mode default`，然后运行 `/sandbox` 并选择自动允许 | 内置 Bash 沙箱，在 macOS、Linux 和 WSL2 上                                                                                                               | 拒绝规则仍然适用，询问规则命名命令（如 `Bash(git push *)`）仍然会提示。要从设置文件启用沙箱，请改为将 [`sandbox.enabled`](/docs/zh-CN/settings-reference#sandbox-enabled) 设置为 `true` |
| 在更改任何内容前探索       | `claude --permission-mode plan`                                                                                          | 无                                                                                                                                               | Claude Code 阻止编辑，直到您[批准计划](#review-and-approve-a-plan)                                                                                 |
| 在自动模式下无需干预工作     | `claude --permission-mode auto`，Pro、Max 和 Team 上的[内置起始权限模式](#which-mode-a-session-starts-in)                             | 无；沙箱或容器增加深度防御                                                                                                                                   | 需要[支持的模型](#eliminate-prompts-with-auto-mode)，您的组织可以[关闭自动模式](#eliminate-prompts-with-auto-mode)                                         |
| 在 CI 中使用精确允许列表运行 | `claude -p "run the test suite" --permission-mode dontAsk --allowedTools "Bash(npm test)" "Read"`                        | 无，超出您的 CI 运行器提供的                                                                                                                                | [网络上的 Claude Code](/docs/zh-CN/claude-code-on-the-web) 忽略设置文件中的 `dontAsk`                                                                   |
| 在容器内完全无人值守运行     | `claude -p "<prompt>" --dangerously-skip-permissions`                                                                    | 必需：容器、虚拟机或[沙箱运行时](/docs/zh-CN/sandbox-environments#sandbox-runtime)；在 Linux 和 macOS 上，以[非 root 用户](#skip-all-checks-with-bypasspermissions-mode)身份运行 | 网络上的 Claude Code 忽略设置文件中的此模式。在此 `-p` 运行中，[仍会提示的少数调用](#skip-all-checks-with-bypasspermissions-mode)被拒绝                                  |

Bash 沙箱和自动模式独立工作并结合，除了在 plan 模式下，其中[自动允许不会扩大批准](/docs/zh-CN/sandboxing#sandbox-modes)。有关完整交互，请参阅[沙箱化如何与权限和权限模式相关](/docs/zh-CN/sandboxing#how-sandboxing-relates-to-permissions-and-permission-modes)和[隔离如何与权限模式相关](/docs/zh-CN/sandbox-environments#how-isolation-relates-to-permission-modes)。

<h2 id="which-mode-a-session-starts-in">
  会话在哪个模式下启动
</h2>

当您在终端中启动新会话时，Claude Code 从以下第一个适用的获取权限模式：

1. `--permission-mode` 标志或 `--dangerously-skip-permissions`

2. [设置文件](/docs/zh-CN/settings#where-settings-live)中的 `permissions.defaultMode`

   如果您在 `.claude/settings.json` 或 `.claude/settings.local.json` 中设置 `"auto"`，该值不会生效，Claude Code 然后使用内置默认值而不是来自 `~/.claude/settings.json` 的 `defaultMode`。如果您在这两个文件中设置 `"bypassPermissions"`，它也不会生效，会话以 Manual 模式启动。其他值从任何设置文件应用。

3. 内置默认值

VS Code 扩展启动的对话遵循[切换权限模式](#switch-permission-modes)中的扩展自己的列表。有关 Claude Code 在恢复会话中启动的权限模式，请参阅[恢复时的权限模式](/docs/zh-CN/sessions#permission-mode-on-resume)。

内置 `auto` 默认值在 macOS、Linux 和 WSL 上需要 Claude Code v2.1.228 或更高版本，在本机 Windows 上需要 v2.1.233 或更高版本。在较早的版本上，内置默认值是 Manual。

内置默认值取决于您如何运行 Claude Code、您的计划以及 Claude Code 是否可以获取其功能标志。匹配您会话的第一行适用。该表涵盖您在终端或通过 VS Code 扩展启动的会话；对于桌面应用和 claude.ai，请参阅[切换权限模式](#switch-permission-modes)中的 Desktop 和 Web 选项卡。

| 您如何运行 Claude Code                                                                                                                                                                 | 内置起始权限模式  |
| :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------- |
| 任何设置文件将 `disableAutoMode` 设置为 `"disable"`                                                                                                                                         | `default` |
| [功能标志获取](/docs/zh-CN/env-vars#features-that-need-feature-flag-fetching)关闭                                                                                                              | `default` |
| 您的[安装或升级后的第一个会话](/docs/zh-CN/env-vars#first-session-after-an-install-or-upgrade)到添加此默认值的版本，除非在全新安装后，Claude Code 及时获取标志                                                                 | `default` |
| `claude -p` 或 [Agent SDK](/docs/zh-CN/agent-sdk/permissions)                                                                                                                           | `default` |
| Amazon Bedrock、Google Cloud 的 Agent Platform、Microsoft Foundry、[AWS 上的 Claude Platform](/docs/zh-CN/claude-platform-on-aws) 或已登录的 [Claude apps gateway](/docs/zh-CN/claude-apps-gateway) 会话 | `default` |
| Pro、Max 或 Team 计划，在终端或通过 [VS Code 扩展](/docs/zh-CN/vs-code)                                                                                                                             | `auto`    |
| Enterprise 计划或 Claude Console API 密钥                                                                                                                                              | `default` |

当功能标志获取关闭或在[安装或升级后的第一个会话](/docs/zh-CN/env-vars#first-session-after-an-install-or-upgrade)中标志尚未到达时，VS Code 扩展在选择起始权限模式时忽略每个设置文件。

当标志、设置文件或内置默认值选择 `auto` 但自动模式对会话不可用时，Claude Code 以 Manual 启动会话。自动模式在会话不满足[可用性要求](#eliminate-prompts-with-auto-mode)时不可用，例如设置文件关闭它或不支持它的模型，或当 Anthropic 已在服务器端临时关闭它时。

内置默认值第一次在自动模式下启动您的一个会话时，Claude Code 显示链接到此页面的通知：

* 在终端中，一次，在会话顶部
* 在 VS Code 扩展中，作为新对话屏幕上的卡片，保留直到您关闭它

在 Pro、Max 和 Team 计划上，如果您的 `~/.claude/settings.json` 将 `defaultMode` 设置为 `auto` 以外的值，且没有其他设置文件设置它，您的会话继续以该模式启动。Claude Code 在终端或 VS Code 扩展中询问一次是否将设置更改为自动模式。如果您拒绝，您的设置保持原样。

<h3 id="start-in-a-different-mode">
  以不同的权限模式启动
</h3>

您可以为一个会话设置起始权限模式，或作为机器、项目或组织中每个会话的默认值。当多个设置文件设置 `permissions.defaultMode` 时，[设置优先级](/docs/zh-CN/settings#settings-precedence)决定，因此项目或托管值优先于 `~/.claude/settings.json`。要更改已运行会话的权限模式，请参阅[切换权限模式](#switch-permission-modes)。

| 要为以下设置起始权限模式     | 执行此操作                                                                                                                                                                                                                |
| :--------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 您即将启动的一个会话       | 将权限模式作为标志传递，例如 `claude --permission-mode default`                                                                                                                                                                    |
| 您在此机器上启动的每个终端会话  | 在 `~/.claude/settings.json` 中设置 `permissions.defaultMode`。有关 VS Code 扩展读取的内容，请参阅[切换权限模式](#switch-permission-modes)                                                                                                   |
| 您在一个项目中启动的每个终端会话 | 在项目的 `.claude/settings.json` 中设置 `permissions.defaultMode`。您在终端中启动的会话遵守除 `auto` 和 `bypassPermissions` 外的每个值；VS Code 扩展启动的会话不读取项目设置以获取起始权限模式                                                                          |
| 您的组织中的每个终端会话     | 在[托管设置](/docs/zh-CN/managed-settings)中设置 `permissions.defaultMode`。终端会话以该模式启动，人们仍然可以切换到自动模式；有关 VS Code 扩展读取的内容，请参阅[切换权限模式](#switch-permission-modes)。要移除自动模式以便没有人可以选择它，请改为将 `permissions.disableAutoMode` 设置为 `"disable"` |

此示例使您机器上的每个终端会话以 Manual 模式启动，其配置值为 `default`。将其保存在 `~/.claude/settings.json` 中：

```json theme={null}
{
  "permissions": {
    "defaultMode": "default"
  }
}
```

您启动的下一个会话在状态栏中显示 `⏸ manual mode on`。

<h2 id="switch-permission-modes">
  切换权限模式
</h2>

每个界面都有自己的控件用于在会话期间切换权限模式，以及自己的方式来选择新会话启动的权限模式。选择您的界面以查看其控件。

<Tabs>
  <Tab title="CLI">
    **在会话期间**：按 `Shift+Tab` 循环权限模式。从 `auto`，第一次按下切换到 `default`，循环然后运行 `default` → `acceptEdits` → `plan` → 回到 `default`。可选模式（如下所述）在 `plan` 之后插入。状态栏显示活动模式为灰色 `⏸ manual mode on`（对于 `default`），或为 `⏵⏵ accept edits on`、`⏸ plan mode on`、`⏵⏵ auto mode on`、`⏵⏵ don't ask on` 或 `⏵⏵ bypass permissions on`。

    并非每个模式都在默认循环中：

    * `auto`：当[自动模式可用](#eliminate-prompts-with-auto-mode)时出现；循环到它会在没有确认提示的情况下切换权限模式
    * `bypassPermissions`：在您使用 `--permission-mode bypassPermissions`、`--dangerously-skip-permissions`、`--allow-dangerously-skip-permissions` 或[用户、`--settings` 或托管设置](/docs/zh-CN/settings-reference#permissions-defaultmode)中的 `permissions.defaultMode: "bypassPermissions"` 启动后出现。`--allow-` 变体将权限模式添加到循环中而不激活它
    * `dontAsk`：永远不会在循环中出现；使用 `--permission-mode dontAsk` 设置它

    启用的可选模式在 `plan` 之后插入，`bypassPermissions` 优先，`auto` 最后。如果您同时启用了两者，您将在循环到 `auto` 的途中循环通过 `bypassPermissions`。

    **从 Bash 权限提示**：在 Manual 和 `acceptEdits` 权限模式下，当[自动模式](#eliminate-prompts-with-auto-mode)可用时，Claude Code 将**是的，并切换到自动模式**添加到 Bash 命令的权限提示。选择它以批准命令并将会话切换到自动模式。[PowerShell 工具](/docs/zh-CN/tools-reference#powershell-tool)提示不提供该选项。需要 Claude Code v2.1.247 或更高版本。

    Claude Code 不会将该选项添加到由您的[`ask` 规则](/docs/zh-CN/permissions#manage-permissions)之一或[hook](/docs/zh-CN/hooks#pretooluse-decision-control)强制的提示，因为自动模式仍然向您显示这些提示，所以切换不会移除它们。

    **在启动时**：将权限模式作为标志传递。

    ```bash theme={null}
    claude --permission-mode plan
    ```

    **作为默认值**：在您想要的范围设置 `permissions.defaultMode`，如[以不同的权限模式启动](#start-in-a-different-mode)中所述。

    相同的 `--permission-mode` 标志适用于 `-p` 用于[非交互式运行](/docs/zh-CN/headless)。
  </Tab>

  <Tab title="VS Code">
    **在会话期间**：点击提示框底部的模式指示器。它为此页面上的模式使用这些标签：

    | UI 标签              | 模式                  |
    | :----------------- | :------------------ |
    | Manual             | `default`           |
    | Edit automatically | `acceptEdits`       |
    | Plan               | `plan`              |
    | Auto               | `auto`              |
    | Bypass permissions | `bypassPermissions` |

    **作为默认值**：要固定对话启动的权限模式，请在您的 VS Code 用户设置中将 `claudeCode.initialPermissionMode` 设置为 `default`、`manual`、`acceptEdits`、`plan` 或 `bypassPermissions`。该设置不接受 `auto`；要以 Auto 启动，请将其保留未设置并从模式指示器中选择**Auto**一次，如下面的第 2 项所述。扩展在以下第一个适用的中启动每个新对话：

    1. `claudeCode.initialPermissionMode`
    2. 您从模式指示器最后选择的模式，如果它是 Manual、Edit automatically 或 Auto。选择 Plan 或 Bypass permissions 仅适用于该对话
    3. 来自[托管设置](/docs/zh-CN/managed-settings)或 `~/.claude/settings.json` 的 `permissions.defaultMode`，在 Pro、Max 和 Team 计划上具有[功能标志获取](#which-mode-a-session-starts-in)可用
    4. 您的计划、提供商和组织设置的[内置默认值](#which-mode-a-session-starts-in)

    扩展永远不会从项目的 `.claude/settings.json` 或 `.claude/settings.local.json` 读取起始权限模式，在不满足第 3 项条件的对话中根本不读取任何设置文件。当 `claudeCode.claudeProcessWrapper` 被设置时，第 3 和 4 项也不适用：这些对话以 Manual 启动，除非第 1 或第 2 项设置权限模式。

    当[自动模式可用](#eliminate-prompts-with-auto-mode)时，Auto 出现在模式指示器中。

    Bypass permissions 需要扩展设置中的**Allow dangerously skip permissions**切换。没有它，权限模式不会出现在指示器中，来自第 1 或第 3 项的 `bypassPermissions` 值以 Manual 启动对话。来自任何项的 Auto 同样在自动模式不可用时以 Manual 启动对话。

    有关扩展特定的详细信息，请参阅 [VS Code 指南](/docs/zh-CN/vs-code)。
  </Tab>

  <Tab title="JetBrains">
    JetBrains 插件在 IDE 终端中运行 Claude Code，因此切换权限模式的工作方式与 CLI 中相同：按 `Shift+Tab` 循环，或在启动时传递 `--permission-mode`。
  </Tab>

  <Tab title="Desktop">
    **在会话期间**：在 Code 选项卡中，使用发送按钮旁边的模式选择器。并非每个模式都出现在选择器中：

    * **Auto**：当[自动模式可用](#eliminate-prompts-with-auto-mode)时出现
    * **Bypass permissions**：在 Pro 和 Max 计划上需要 Desktop 设置中的**Allow bypass permissions mode**切换；在 Team 和 Enterprise 计划上，组织策略控制它

    Cowork 选项卡不使用这些模式。Cowork 有自己的权限模式，单独启用，Cowork 选项卡在为您的账户启用超出其默认值的模式之前不显示模式选择器。请参阅 [Cowork 文档](https://claude.com/docs/cowork/overview)。

    有关 desktop 特定的详细信息，请参阅 Desktop 指南中的[选择权限模式](/docs/zh-CN/desktop#choose-a-permission-mode)。

    **作为默认值**：在[设置](/docs/zh-CN/settings#where-settings-live)中设置 `defaultMode`。桌面应用读取与 CLI 相同的设置文件，并将权限模式应用于新的本地会话。

    您在模式选择器中选择的模式会按文件夹记住，并对该文件夹优先于 `defaultMode`。Plan 是例外：选择它仅适用于当前会话。

    有关 `defaultMode` 在设置文件中的位置，请参阅[以不同的权限模式启动](#start-in-a-different-mode)下的示例。
  </Tab>

  <Tab title="Web and mobile">
    在 [claude.ai/code](https://claude.ai/code) 或移动应用中使用提示框旁边的模式下拉菜单。权限提示出现在 claude.ai 中以供批准。显示哪些模式取决于会话在何处运行：

    * **Cloud sessions** 在 [Claude Code on the web](/docs/zh-CN/claude-code-on-the-web) 上：Accept edits、Plan 和 Auto。Accept edits 对应于 `default` 模式：云会话预先批准文件编辑，无论模式如何，因此下拉菜单显示 Accept edits 而不是 Manual。云会话仍然遵守设置中的 `defaultMode: "acceptEdits"`。Auto 模式仅在您的组织允许且所选模型支持时出现。Bypass permissions 不可用。
    * **[Remote Control](/docs/zh-CN/remote-control) sessions** 在您的本地机器上：Manual、Accept edits 和 Plan。您无法从应用中选择 Auto 或 Bypass permissions。
      * 除了 Bypass permissions，下拉菜单显示本地会话所在的权限模式，包括从终端设置的模式。它在应用或终端中权限模式更改时更新。会话永远不会向 claude.ai 报告 Bypass permissions，因此从终端切换到它不会改变下拉菜单显示的内容。
      * 由[桌面应用](/docs/zh-CN/desktop)或 [VS Code 扩展](/docs/zh-CN/vs-code)托管的会话在权限模式更改时向 claude.ai 报告，与在终端中托管的会话相同。
      * 在 v2.1.202 之前，使用 `/remote-control` 或 `claude --remote-control` 连接的会话根本不报告其权限模式，因此 claude.ai 和移动应用可能显示会话不在的权限模式。不匹配仅影响标签。Claude Code 从会话的实际权限模式生成权限提示，它们仍然出现在应用中以供批准。

    对于 Remote Control，运行会话的本地机器必须使用您的 claude.ai 账户登录；不支持 API 密钥。您也可以在启动该本地会话时设置起始权限模式：

    ```bash theme={null}
    claude remote-control --permission-mode acceptEdits
    ```
  </Tab>
</Tabs>

<h2 id="auto-approve-file-edits-with-acceptedits-mode">
  使用 acceptEdits 模式自动批准文件编辑
</h2>

`acceptEdits` 模式让 Claude 在您的工作目录中创建和编辑文件而无需提示。当此模式处于活动状态时，状态栏显示 `⏵⏵ accept edits on`。

除了文件编辑外，`acceptEdits` 模式还自动批准常见的文件系统 Bash 命令：`mkdir`、`touch`、`rm`、`rmdir`、`mv`、`cp` 和 `sed`。当这些命令带有安全环境变量（如 `LANG=C` 或 `NO_COLOR=1`）或进程包装器（如 `timeout`、`nice` 或 `nohup`）作为前缀时，也会自动批准。与文件编辑一样，自动批准仅适用于工作目录或 `additionalDirectories` 内的路径。超出该范围的路径、对[受保护路径](#protected-paths)的写入、`rm` 和 `rmdir` 移除针对[关键路径](#critical-paths)的操作以及所有其他 Bash 命令（除了[内置只读集合](/docs/zh-CN/permissions#read-only-commands)）仍然会提示。

当启用 [PowerShell 工具](/docs/zh-CN/tools-reference#powershell-tool)时，`acceptEdits` 模式还会自动批准 `Set-Content`、`Add-Content`、`Clear-Content` 和 `Remove-Item` 在范围内路径上的操作，以及它们的常见别名。相同的范围和受保护路径规则适用，`Remove-Item` 获得[自己的检查](#remove-item-in-powershell)。包含引号字符的位置参数，如 `Set-Content .\notes.txt "It's done"` 中的撇号，即使在范围内路径上也仍然会提示，因为 Claude Code 无法静态验证其引用和未引用读数不同的参数。通过命名参数（如 `-Value`）传递内容以避免提示。

当您想在编辑器中或通过 `git diff` 事后查看更改，而不是逐个批准每个编辑时，使用 `acceptEdits`。

从 Manual 模式按一次 `Shift+Tab` 进入它，或直接启动它：

```bash theme={null}
claude --permission-mode acceptEdits
```

<h2 id="analyze-before-you-edit-with-plan-mode">
  使用 plan mode 在编辑前进行分析
</h2>

Plan mode 告诉 Claude 研究并提议更改而不进行编辑。Claude 读取文件、运行 shell 命令进行探索并编写计划，但不编辑您的源代码。除了在[绕过权限可用](#skip-all-checks-with-bypasspermissions-mode)的会话中，编辑保持阻止状态，直到您批准计划。

当[自动模式](/docs/zh-CN/auto-mode-config)可用且 `useAutoModeDuringPlan` 设置打开（默认情况下是这样）时，分类器在规划期间审查 shell 命令而不是提示您。批准的命令运行，拒绝的命令被阻止。否则，[内置只读集合](/docs/zh-CN/permissions#read-only-commands)外的命令会提示批准，包括当沙箱的[自动允许模式](/docs/zh-CN/sandboxing#sandbox-modes)启用时。在绕过权限可用的会话中，分类器和提示都不适用于规划命令；[使用 bypassPermissions 模式跳过所有检查](#skip-all-checks-with-bypasspermissions-mode)涵盖仍会在那里提示的少数事项。在 v2.1.212 到 v2.1.217 中，没有绕过权限的会话为只读集合外的每个命令提示，无论自动模式是否可用。

通过按 `Shift+Tab` 或在单个提示前加上 `/plan` 进入 plan mode。您也可以从 CLI 启动 plan mode：

```bash theme={null}
claude --permission-mode plan
```

再次按 `Shift+Tab` 以退出 plan mode 而不批准计划。

<h3 id="review-and-approve-a-plan">
  审查并批准计划
</h3>

当计划准备好时，Claude 会呈现它并询问如何继续。从该提示中，您可以选择：

* **是的，并使用自动模式**：批准并以[自动模式](#eliminate-prompts-with-auto-mode)启动。当自动模式不可用时，此选项读取**是的，自动接受编辑**。如果您使用启用的绕过权限启动会话，该选项读取**是的，并为此会话切换到绕过权限（无进一步提示）**。
* **是的，手动批准编辑**：批准并逐个审查每个编辑。
* **否，继续规划**：保持在 plan mode 并告诉 Claude 要更改什么。

批准计划退出 plan mode 并将会话切换到每个批准选项描述的权限模式，因此 Claude 开始编辑。要再次规划，使用 `Shift+Tab` 循环回到 plan mode，或在下一个提示前加上 `/plan`。

按 `Ctrl+G` 在默认文本编辑器中打开建议的计划并在 Claude 继续之前直接编辑它。当启用 [`showClearContextOnPlanAccept`](/docs/zh-CN/settings-reference#showclearcontextonplanaccept) 时，列表获得第一个选项，该选项批准计划并清除规划上下文。

接受计划也会根据计划为会话提供[生成的标题](/docs/zh-CN/sessions#name-your-sessions)，除非您已经命名了会话。

<h3 id="set-plan-mode-as-the-default">
  将 plan mode 设置为默认值
</h3>

要使 plan mode 成为项目的终端会话的默认值，请在 `.claude/settings.json` 中将 `defaultMode` 设置为 `plan`，如[以不同的权限模式启动](#start-in-a-different-mode)下的示例所示。[VS Code 扩展](/docs/zh-CN/vs-code)启动的对话不读取项目设置以获取起始权限模式。在那里，改为在您的 VS Code 用户设置中将 `claudeCode.initialPermissionMode` 设置为 `plan`。

<h2 id="eliminate-prompts-with-auto-mode">
  使用自动模式消除权限提示
</h2>

自动模式让 Claude 无需常规权限提示即可执行。一个独立的分类器模型在操作运行前审查这些操作，阻止任何超出您请求范围、针对无法识别的基础设施或看起来由 Claude 读取的恶意内容驱动的操作。显式的[询问规则](/docs/zh-CN/permissions#manage-permissions)仍然会强制显示提示。

在 Pro、Max 和 Team 计划上，自动模式是[会话启动时的内置权限模式](#which-mode-a-session-starts-in)。

分类器还会审查 Claude 使用 [`SendMessage`](/docs/zh-CN/tools-reference) 发送给另一个代理的每条消息，无论是纯文本还是结构化的[代理团队](/docs/zh-CN/agent-teams)消息，在 Claude Code 交付之前，无论是在自动模式还是在[计划模式中分类器审查命令](#analyze-before-you-edit-with-plan-mode)时；发送审查需要 Claude Code v2.1.222 或更高版本。

分类器还会审查并批准或阻止针对[关键路径](#critical-paths)的 `rm` 和 `rmdir` 删除，例如 `rm -rf /` 和 `rm -rf ~`，包括当删除位于命令或进程替换内部时。

自动模式还会促使 Claude 继续工作而不停下来提出澄清问题，尽管当您的提示或技能明确依赖它时 Claude 仍然会提问。为了在仍然提示您的模式中获得更强的自主行为，请改为设置[主动输出风格](/docs/zh-CN/output-styles)。

<Warning>
  自动模式减少了权限提示，但不保证安全。将其用于您信任总体方向的任务，而不是作为敏感操作审查的替代品。
</Warning>

自动模式仅在您的账户满足以下所有要求时可用：

* **计划**：所有计划。
* **组织**：在 Team 和 Enterprise 上，自动模式默认可用。管理员可以通过在[托管设置](/docs/zh-CN/managed-settings)中将 `permissions.disableAutoMode` 设置为 `"disable"` 来为组织关闭它。
* **模型**：在 Anthropic API 和 [AWS 上的 Claude Platform](/docs/zh-CN/claude-platform-on-aws) 上，Claude Opus 4.6 或更高版本、Sonnet 4.6 或更高版本，或[Fable 模型](/docs/zh-CN/model-config#work-with-fable)。在 Amazon Bedrock、Google Cloud 的 Agent Platform、Microsoft Foundry 和已登录的[Claude 应用网关](/docs/zh-CN/claude-apps-gateway)会话上，仅支持 Claude Sonnet 5、Opus 4.7 或更高版本以及 Fable 模型。较旧的模型，包括 Sonnet 4.5、Opus 4.5、Haiku 和 claude-3 模型，在任何提供商上都不受支持。
* **提供商**：在 Anthropic API、AWS 上的 Claude Platform、Amazon Bedrock、Google Cloud 的 Agent Platform、Microsoft Foundry 和已登录的 Claude 应用网关会话上默认可用。

如果 Claude Code 报告自动模式不可用，首先检查这些要求以及任何设置文件是否设置了 [`disableAutoMode`](/docs/zh-CN/settings-reference#disableautomode)。Anthropic 也可能已在服务器端关闭了自动模式，或者服务器可能为您的账户拒绝了自动模式。接收到任一答案的会话会保持自动模式关闭，直到会话结束，因此请稍后启动新会话。

一条单独的消息，其中命名了一个模型并说自动模式"无法确定"操作的安全性，意味着分类器请求失败。该失败通常是暂时的，但在 Amazon Bedrock 上，它可能会重复，直到您的账户可以调用命名的模型。有关原因和处理方法，请参阅[错误参考](/docs/zh-CN/errors#auto-mode-cannot-determine-the-safety-of-an-action)。

如果您在[设置](/docs/zh-CN/settings-reference#all-settings)中设置了 `defaultMode: "auto"`，而终端会话在没有错误的情况下以手动模式启动，则该设置可能在 `.claude/settings.json` 或 `.claude/settings.local.json` 中。`auto` 不会从这些文件生效。将其移至 `~/.claude/settings.json`。对于 VS Code 扩展启动的对话，请改为检查扩展自己的列表[切换权限模式](#switch-permission-modes)。

<h3 id="enable-auto-mode-on-bedrock-agent-platform-or-foundry">
  Bedrock、Agent Platform 或 Foundry 上的自动模式
</h3>

在 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock)、[Google Cloud 的 Agent Platform](/docs/zh-CN/google-vertex-ai)、[Microsoft Foundry](/docs/zh-CN/microsoft-foundry) 和已登录的[Claude 应用网关](/docs/zh-CN/claude-apps-gateway)会话上，自动模式默认出现在 `Shift+Tab` 循环中。出现在循环中不会改变会话启动时的权限模式：在这些提供商上，终端会话以您的 [`defaultMode`](/docs/zh-CN/settings-reference#permissions-defaultmode) 启动，除非您更改它，否则为手动模式，而[VS Code 扩展](/docs/zh-CN/vs-code)中的对话以手动模式启动，除非 `claudeCode.initialPermissionMode` 或您在扩展中选择的模式设置了一个。这些提供商上仅支持 Claude Sonnet 5、Opus 4.7 或更高版本以及 Fable 模型。

要使自动模式成为默认启动权限模式，请在用户或托管设置中设置 `"permissions": {"defaultMode": "auto"}`。在 VS Code 扩展启动的会话中，改为从模式指示器中选择**自动**。[切换权限模式](#switch-permission-modes)涵盖了什么会优先于该选择。

[`/doctor`](/docs/zh-CN/commands#all-commands) 检查在这些提供商上提议此用户设置默认值，就像在 Anthropic API 上一样。

要防止开发人员使用自动模式，请在[托管设置](/docs/zh-CN/managed-settings)中将 `disableAutoMode` 设置为 `"disable"`。这会从 `Shift+Tab` 循环中删除 `auto`，并且使用 `--permission-mode auto` 启动的会话以手动模式启动。已在自动模式中运行的会话在设置从[管理员部署的源](/docs/zh-CN/managed-settings#which-managed-source-claude-code-uses)到达该会话时会离开它，并显示 `auto mode disabled by settings`。在 v2.1.251 之前，运行中的会话会保持自动模式直到结束。

在 v2.1.158 到 v2.1.206 中，这些提供商上的自动模式处于关闭状态，直到您设置 `CLAUDE_CODE_ENABLE_AUTO_MODE=1`，并且 Claude Code 在这些提供商上忽略 `defaultMode: "auto"`，除非也设置了该变量。该变量仍然被接受以保持兼容性，从 v2.1.207 开始无效。

<h3 id="what-the-classifier-blocks-by-default">
  分类器默认阻止的内容
</h3>

分类器信任您的工作目录和在会话启动时为其配置的远程。在会话期间使用 `git remote add` 或 `git remote set-url` 添加或重新指向的远程不受信任，其他所有内容都被视为外部，直到您[配置受信任的基础设施](/docs/zh-CN/auto-mode-config)。在 v2.1.200 之前，会话中期添加的远程也受信任。

**默认阻止**：

* 下载和执行代码，如 `curl | bash`
* 向外部端点发送敏感数据
* 生产部署和迁移
* 云存储上的大规模删除
* 授予 IAM 或仓库权限
* 修改共享基础设施
* 不可逆地销毁会话前存在的文件
* 强制推送
* 提交或推送会在运行时向仓库外发送秘密或敏感数据的更改，或扩大部署公开的内容。这涵盖了将秘密交给不已接收它的目标的 CI 工作流或部署配置、读取秘密存储并发送数据的脚本或设置步骤，以及扩大部署发布内容的配置更改，例如注册表、可见性、工件或源映射设置。检查适用于任何分支，即使仓库是公开的也适用，并在更改落地时触发，无论该落地是否触发管道；清除它需要命名执行效果，而不仅仅是提交或推送。在 v2.1.211 之前，此检查的范围仅限于默认分支：推送到那里时，如果携带敏感内容、相对于您要求的隐藏或误描述的更改、从仓库外移植的内容或绕过您要求的审查的内容，则被阻止
* `git reset --hard`、`git checkout -- .`、`git restore .`、`git clean -fd`、`git stash drop` 或 `git stash clear`，分类器假设这会丢弃未提交的更改
* 当 HEAD 处的提交不是在此会话中创建时的 `git commit --amend`
* 从 v2.1.198 开始，当 HEAD 处的提交已被推送时的 `git commit --amend`。仅消息改写不被阻止：`--amend -m` 在没有新暂存内容的情况下，对于 Claude 在此会话期间创建的提交
* `terraform destroy`、`pulumi destroy`、`cdk destroy` 或 `terragrunt destroy`，以及应用销毁资源的计划

Claude Code v2.1.195 及更高版本默认阻止更多类别。其中几个取决于[环境](/docs/zh-CN/auto-mode-config#define-trusted-infrastructure)条目，例如敏感的远程目标和受保护的 IaC 范围，您可以将其缩小到具体名称。

* 写入秘密管理器，或更改 DNS 记录或 TLS 证书
* 合并没有人类批准的拉取请求、批准 Claude 自己的拉取请求或禁用 CI 检查
* 发布本身是自动化命令的评论，例如 `atlantis apply` 或机器人的 `/deploy` 或 `/merge`
* 切换、调整或删除生产功能标志
* 将基础设施更改应用于受保护的 IaC 范围，或排空和删除集群节点
* 对共享计算集群的写入超出您命名的资源，例如标签选择器或 `--all` 捕获其他用户的作业
* 创建在每个节点上运行或拦截集群流量的 Kubernetes 资源，例如 DaemonSets 和准入 webhooks
* 交互式 shell 或端口转发到敏感的远程目标
* 打开隧道或反向 shell，使本地服务可从公网访问
* 将实时凭证或令牌打印到记录或文件中
* 访问在您的[环境](/docs/zh-CN/auto-mode-config#define-trusted-infrastructure)中列为敏感数据位置的位置，或从其中复制数据。从 v2.1.198 开始，这也会阻止从一个位置向条目排除的受众发送数据
* 绕过您的内部包注册表路由包安装到公开注册表。从 v2.1.198 开始，这也适用于您在对话中告诉 Claude 存在内部注册表或镜像的情况，而不仅仅是在您的环境中列出的情况
* 使用禁用安全防护的标志运行命令，如 `--insecure`
* 启动在没有人类批准或沙箱的情况下运行的自主代理循环，例如使用 `--dangerously-skip-permissions` 或 `--no-sandbox` 启动的循环。从 v2.1.198 开始，这也涵盖运行禁用隔离和按操作批准的第三方代理或评估工具，例如使用 `--yes-always` 启动的运行器
* [Chrome 中的 Claude](/docs/zh-CN/chrome)浏览器操作可能会向外源发送页面内容、cookie 或凭证

Claude Code v2.1.198 及更高版本也默认阻止这些：

* 通过通配符、glob 或年龄过滤器而不是特定命名路径删除 `/tmp`、`$TMPDIR` 或其他共享暂存或缓存目录中的文件
* 在您自己的消息未授权这些详细信息给该收件人时，在发送、上传、发布或写入给其他人或共享系统的内容中包含敏感详细信息。当仓库在信任边界外或公开时，PR 和问题正文、提交消息和评论计为这种类型的出站内容，包括您组织自己的公开仓库；内部文件路径、代码名称、实时 API 响应数据（如电子邮件或账户标识符）和基础设施标识符计为敏感详细信息。PR、问题和提交消息范围需要 Claude Code v2.1.200 或更高版本。PR 或问题正文中的实时个人数据（如电子邮件地址、账户或组织标识符或使用指标）需要您命名这些详细信息和收件人，无论仓库的可见性或信任边界如何。该检查需要 Claude Code v2.1.203 或更高版本
* 向 Claude Code 自己的 tmux 窗格发送按键以驱动其自己的界面，分类器将其视为 Claude 更改自己的权限或监督

Claude Code v2.1.200 及更高版本也默认阻止这些：

* 注释掉、删除或强制通过保护安全行为的测试或断言，例如身份验证、访问控制、输入验证或沙箱
* 删除或拆除 Claude 在会话中未创建的有状态资源，当没有更具体的删除规则适用且您未命名该资源时
* 在不适合任务的第三方主机处重新指向 API 基础 URL、代理端点、webhook 接收器或注册表镜像，包括在 `.env.example` 等示例文件中
* 使用 `git remote set-url` 或 `git remote add` 更改推送去向，除非您命名了新远程
* 推送秘密或个人或受信任的数据到已知为公开的仓库，或推送不属于该仓库自己工作的机密材料。dotfiles 仓库自己的主题是个人或受信任数据的唯一例外，来自私有仓库到任何公开表面的内容以相同方式被阻止；两项改进都需要 Claude Code v2.1.203 或更高版本。在 v2.1.203 之前，个人数据与机密材料分组，仅当它不属于该仓库自己的工作时才被阻止。当仓库的可见性未确定时，分类器不会仅凭这一点就阻止；它改为根据其他规则判断内容
* 针对不同的仓库或组织打开拉取请求、使用 `gh repo fork` 进行分叉或推送到第三方仓库，除非您命名了该外部目标

Claude Code v2.1.203 及更高版本也默认阻止这些：

* 来自敏感本地存储或其名称、路径或类型将其标记为敏感的文件的内容进入提交、推送、PR 或问题文本、gist 或粘贴或包发布，除非您命名了源和目标。会话记录和对话日志、凭证和配置点文件夹（如 SSH 密钥、云凭证、浏览器配置文件和 shell 历史记录）以及用户数据导出都计为此，仓库是私有的不会清除它

Claude Code v2.1.205 及更高版本也默认阻止这些：

* 写入 Claude Code 会话记录，即 `~/.claude/projects/` 或您配置的配置目录下的 `.jsonl` 历史文件，无论是直接还是通过 shell 命令。该规则也涵盖 Claude Code 为其自己的检查附加到每个记录条目的元数据行。读取记录不被阻止
* 递归强制删除，例如 `rm -rf "$VAR"` 或 `Remove-Item -Recurse -Force $dir`，其目标是分类器看不到在对话中任何地方分配的 shell 变量，或以一个为根的 glob。该值仅来自较早的命令输出，分类器从不接收，因此分类器无法根据其他删除规则验证删除目标。当您命名被删除的确切路径或 Claude 使用解析的文字路径写入的命令重新运行删除时，该块会清除。目标分类器可以解析的删除不受影响。目标是裸 `*` 或以 `/*` 或 `\*` 结尾的 `Remove-Item` 目标永远不会到达分类器：Claude Code [直接拒绝它们](#remove-item-in-powershell)

Claude Code v2.1.257 及更高版本也默认阻止这些：

* 从云实例元数据端点请求凭证，例如 `169.254.169.254`，或使用机器自己的服务账户或节点身份显式验证云、集群或注册表调用
* 通过直接请求以外的路由到达公开主机，例如隧道、反向 shell 或重写为指向外部的解析器或代理配置
* 读取属于主机而不是您的任务的凭证，例如节点证书或节点的容器注册表身份验证
* 连接到或扫描 Claude 未启动的同级容器、pod 或 VM，或容器下的节点

如果 Claude Code 在允许其中之一的地方运行，请在 `autoMode.environment` 中的[主机包含条目](/docs/zh-CN/auto-mode-config#define-trusted-infrastructure)中描述该设置。

Claude Code v2.1.261 及更高版本也默认阻止这些：

* 在消息、PR 或问题文本、文档或链接将被打开或获取的任何其他地方发布或写入公开粘贴、图表或数据共享服务的链接，当 URL 本身携带被共享的内容时，除非您命名了该服务

**默认允许**：

* 工作目录中的本地文件操作
* 安装在您的锁定文件或清单中声明的依赖项
* 读取 `.env` 并向其匹配的 API 发送凭证
* 只读 HTTP 请求
* 推送到您正在处理的仓库的任何分支，包括默认分支。其名称将其标记为部署或发布目标的非默认分支，例如 `production` 或 `gh-pages`，不被涵盖：分类器根据其自己的条款判断推送到那里。推送的内容仍然根据其他规则进行检查，[`permissions.deny` 规则](/docs/zh-CN/permissions#manage-permissions)仍然可以在每种模式中[按书写](/docs/zh-CN/permissions#bash-rule-limits)阻止推送命令，远程自己的分支保护仍然适用。在 v2.1.211 之前，仅允许推送到您启动的分支、Claude 创建的分支和到默认分支的常规推送，在 v2.1.203 之前任何直接推送到默认分支都被阻止

Claude Code v2.1.195 及更高版本也默认允许这些：

* 删除 Claude 在同一会话中较早创建的确切作业
* 作为您的任务的一部分读取、审查或编写与安全相关的代码、配置和威胁模型
* 在同一多代理会话中一起工作的代理之间的消息
* 向您在 [`environment`](/docs/zh-CN/auto-mode-config#define-trusted-infrastructure) 中列出的受信任域、存储桶和服务发送数据。这仅涵盖数据流，不涵盖同一基础设施上的破坏性或凭证操作
* [Chrome 中的 Claude](/docs/zh-CN/chrome)导航到受信任的内部域、localhost 或您命名的 URL

沙箱网络访问请求通过分类器路由，而不是默认允许。从 v2.1.198 开始，分类器重用其对网络主机和端口的判决，而不是在每次连接时重新运行：

* 允许被重用直到新内容进入对话，此时该主机再次被检查
* Claude Code v2.1.234 及更高版本重用由对话超出分类器上下文窗口引起的拒绝，直到新内容进入对话或直到[压缩](/docs/zh-CN/costs#reduce-token-usage)缩小分类器读取的内容。Claude Code 然后再次检查主机
* 分类器通过评估请求达到的拒绝在交互式 CLI 中持续一个回合。在[非交互模式](/docs/zh-CN/headless)和 Agent SDK 会话中，Claude Code 为其余运行重用该拒绝，因为这些会话没有回合边界
* 更改您的权限模式或规则会删除所有缓存的判决

运行 `claude auto-mode defaults` 以将完整规则列表打印为 JSON。如果常规操作被阻止，管理员可以通过 `autoMode.environment` 设置添加受信任的仓库、存储桶和服务：请参阅[配置自动模式](/docs/zh-CN/auto-mode-config)。

推送到您正在处理的仓库的任何分支并创建与您的请求匹配的拉取请求无需提示即可运行，除非推送或拉取请求属于[阻止列表](#what-the-classifier-blocks-by-default)，例如秘密或敏感数据离开仓库，或针对不同仓库或组织的拉取请求。要在保持自动模式的同时在这些命令之前需要人类检查点，请添加 `permissions.ask` 规则，这些规则与命令[按书写](/docs/zh-CN/permissions#bash-rule-limits)匹配：请参阅[常见边界](/docs/zh-CN/auto-mode-config#common-boundaries)。

<h3 id="first-read-outside-the-working-directories">
  工作目录外的第一次读取
</h3>

当 [`permissions.blockReadsOutsideWorkingDirectories`](/docs/zh-CN/settings-reference#permissions-blockreadsoutsideworkingdirectories) 关闭时，文件读取在自动模式中无需提示即可运行，包括在[工作目录](/docs/zh-CN/permissions#working-directories)外的读取。Claude 第一次在它们外的路径上使用 Read、Grep 或 Glob 工具时，Claude Code 会询问您是否继续允许这些读取。

该提示不会出现在非交互式 `-p` 运行或后台会话中；那里的读取照常运行。

无论您的答案如何，Claude 都会继续工作：

* **继续允许**：读取运行，工作目录外的后续读取照常运行，Claude Code 记录您的答案，以便提示不再出现
* **从现在开始阻止**：读取被拒绝，Claude Code 在您的用户设置中将 [`permissions.blockReadsOutsideWorkingDirectories`](/docs/zh-CN/settings-reference#permissions-blockreadsoutsideworkingdirectories) 设置为 `true`，这使文件工具在每个后续会话和每种权限模式中拒绝此类读取。要稍后让 Claude 读取此类路径，请使用 `/add-dir` 添加其目录或删除该设置。
* **下次再问**：读取被拒绝，工作目录外的下一次读取再次提示

<h3 id="boundaries-you-state-in-conversation">
  您在对话中陈述的边界
</h3>

分类器将您在对话中陈述的边界视为阻止信号。如果您告诉 Claude"不要推送"或"在我审查后再部署"，分类器会阻止匹配的操作，即使默认规则会允许它们。边界保持有效，直到您在后续消息中解除它。Claude 自己的判断条件已满足不会解除它。

边界不存储为规则。分类器在每次检查时从记录中重新读取它们，因此如果[上下文压缩](/docs/zh-CN/costs#reduce-token-usage)删除了陈述它的消息，边界可能会丢失。为了获得硬保证，请改为添加[拒绝规则](/docs/zh-CN/permissions#permission-rule-syntax)。

<h3 id="when-auto-mode-falls-back">
  自动模式何时回退
</h3>

当自动模式无法批准您的会话操作时，会发生什么取决于情况：

* **被阻止的操作**：Claude Code 显示通知并在 `/permissions` 下的**最近拒绝**选项卡中列出操作，您可以按 `r` 使用手动批准重试它。当分类器对操作[没有判决](/docs/zh-CN/errors#auto-mode-cannot-determine-the-safety-of-an-action)时，因为分类器自己的请求的单独安全检查拒绝了它或其响应未解析，Claude Code 拒绝该操作而不显示通知或**最近拒绝**条目。
* **重复阻止**：如果分类器连续阻止操作 3 次或总共 20 次，自动模式暂停，Claude Code 恢复提示。批准提示的操作会恢复自动模式。这些阈值不可配置。任何允许的操作重置连续计数器，而总计数器在会话中持续，仅在其自己的限制触发回退时重置。当[分类器自己的请求的单独安全检查拒绝](/docs/zh-CN/errors#auto-mode-cannot-determine-the-safety-of-an-action)时，Claude Code 不计算拒绝到任一阈值；链接的条目涵盖 Claude Code 如何处理这些拒绝。
* **无法提示的会话**：[非交互式](/docs/zh-CN/headless) `-p` 运行没有 [`--permission-prompt-tool`](/docs/zh-CN/cli-reference#cli-flags) 没有回退提示。当重复阻止达到阈值时，操作不运行，Claude 继续工作。当[分类器自己的请求的单独安全检查拒绝](/docs/zh-CN/errors#auto-mode-cannot-determine-the-safety-of-an-action)时也适用相同情况。Claude Code 在任一情况下都不停止运行。
* **检查期间的模式切换**：如果您在分类器检查待处理时切换权限模式，Claude Code 会丢弃新模式不会请求的判决，而不是应用它：您改为被提示批准，或在 [`dontAsk` 模式](#allow-only-pre-approved-tools-with-dontask-mode)中操作被自动拒绝。

重复阻止通常意味着分类器缺少关于您的基础设施的上下文。使用 `/feedback` 报告误报，或让管理员[配置受信任的基础设施](/docs/zh-CN/auto-mode-config)。

<span id="how-the-classifier-evaluates-actions" />

<AccordionGroup>
  <Accordion title="分类器如何评估操作">
    每个操作都经过固定的决策顺序。第一个匹配的步骤获胜：

    1. 与您的[允许、询问或拒绝规则](/docs/zh-CN/permissions#manage-permissions)匹配的操作立即解决。写入[受保护路径](#protected-paths)的操作即使允许规则匹配也会路由到分类器，Claude Code v2.1.218 及更高版本中针对[关键路径](#critical-paths)的 `rm` 和 `rmdir` 删除也是如此。标记为 [`requiresUserInteraction`](/docs/zh-CN/mcp#require-approval-for-a-specific-tool) 的 MCP 工具即使允许规则匹配也会直接提示您，您的组织在会话中设置为 `ask` 的[连接器工具](/docs/zh-CN/mcp#organization-controls-on-connector-tools)也是如此，其中该设置到达 Claude Code。与命令内容匹配的询问规则，例如 `Bash(git push *)`，回退到权限提示
    2. 只读操作和工作目录中的文件编辑被自动批准，除了写入[受保护路径](#protected-paths)和[工作目录外的第一次读取](#first-read-outside-the-working-directories)，这会提示您
    3. 其他所有内容都转到分类器。在步骤 1 中直接提示您的连接器工具和`requiresUserInteraction` MCP 工具永远不会到达分类器，因此组织要求的批准或同意步骤都不会被自动批准
    4. 如果分类器阻止，Claude 接收原因并尝试替代方案。在大多数会话中，原因是固定文本 `Blocked by classifier` 而不是书面解释，在 Claude Code v2.1.208 及更高版本中；请参阅[审查拒绝](/docs/zh-CN/auto-mode-config#review-denials)

    进入自动模式时，授予任意代码执行的广泛允许规则被删除：

    * 笼统的 `Bash(*)` 或 `PowerShell(*)`
    * 通配符解释器，如 `Bash(python*)`
    * 包管理器运行命令
    * `Agent` 允许规则
    * [`Monitor`](/docs/zh-CN/tools-reference#monitor-tool) 允许规则，因为 Claude Code 通过 shell 运行 Monitor 命令

    狭窄的规则，如 `Bash(npm test)` 保持有效。Claude Code 在您离开自动模式时恢复删除的规则。在 v2.1.236 之前，Claude Code 在自动模式中保持 `Monitor` 允许规则有效，因此与整个工具匹配的规则在没有分类器审查的情况下批准 Monitor 命令。

    Claude Code 还在会丢弃未提交工作的命令之前运行 `git status`，例如 `git reset --hard` 或 `rm -rf`，并向分类器显示是否存在暂存、修改或未跟踪的工作。Claude Code 在该检查中报告未跟踪的文件，即使仓库的 git 配置设置了 `status.showUntrackedFiles=no`。

    分类器看到用户消息、除了只读查找（如文件读取和搜索）之外的工具调用，以及您的 CLAUDE.md 内容。工具结果被剥离，因此文件或网页中的恶意内容无法直接操纵它。您可以使用 [PostToolUse hook 的 `classifierContext` 字段](/docs/zh-CN/hooks#annotate-a-result-for-the-auto-mode-classifier)注释调用的结果，分类器将其读取为应用程序提供的上下文。

    单独的服务器端探针扫描传入的工具结果并在 Claude 读取之前标记可疑内容。有关这些层如何协同工作的更多信息，请参阅[自动模式公告](https://claude.com/blog/auto-mode)和[工程深度潜水](https://www.anthropic.com/engineering/claude-code-auto-mode)。
  </Accordion>

  <Accordion title="自动模式如何处理子代理">
    分类器在三个点检查[子代理](/docs/zh-CN/sub-agents)工作：

    1. 在子代理启动之前，委托的任务描述被评估，因此危险看起来的任务在生成时被阻止。
    2. 当子代理运行时，其每个操作都通过分类器，使用与父会话相同的规则，子代理 frontmatter 中的任何 `permissionMode` 都被忽略。
    3. 当子代理完成时，分类器审查其完整的操作历史；如果该返回检查标记了一个问题，安全警告被添加到子代理的结果前面。当单独的 API 安全检查拒绝审查请求本身时，Claude Code 仍然返回子代理的结果，前面加上警告，说工作未审查，应被视为不受信任。

    步骤 1 需要 Claude Code v2.1.178 或更高版本。较早的版本在步骤 2 和 3 应用分类器，但在子代理启动之前没有评估任务描述。
  </Accordion>

  <Accordion title="成本和延迟">
    分类器默认在 Claude Sonnet 5 上运行，而不是在您的 `/model` 选择上。Anthropic 配置的服务器端分类器模型优先于该默认值。当您的会话模型是 Claude Sonnet 4.6 时，或当 [`availableModels`](/docs/zh-CN/model-config#restrict-model-selection) 排除 Sonnet 5 时，分类器改为在会话的模型上运行，或在会话在[Fable 模型](/docs/zh-CN/model-config#work-with-fable)上运行时在 Opus 模型上运行；在 Anthropic API 以外的提供商上，该 Opus 回退是提供商的默认 Opus 模型。

    会话的第一个自动模式请求验证 Sonnet 5 默认值：如果请求成功，Sonnet 5 保持会话的分类器模型，如果它因模型不可用而失败，会话改为使用回退。在该验证解决后，分类器的模型在会话中不会改变。

    在 Enterprise 计划和使用 Claude API、[AWS 上的 Claude Platform](/docs/zh-CN/claude-platform-on-aws)、Amazon Bedrock、Google Cloud 的 Agent Platform 或 Microsoft Foundry 的账户上，分类器调用计入您的令牌使用。每次检查发送记录的一部分加上待处理操作，在执行前添加往返。在受保护路径外的读取和工作目录编辑跳过分类器，因此开销主要来自 shell 命令和网络操作。

    分类器重用沙箱网络判决用于主机和端口，因此重复连接到同一主机不会各自添加检查。[分类器默认阻止的内容](#what-the-classifier-blocks-by-default)描述允许和拒绝持续多长时间。
  </Accordion>
</AccordionGroup>

<h2 id="allow-only-pre-approved-tools-with-dontask-mode">
  使用 dontAsk 模式仅允许预先批准的工具
</h2>

如果您设置 `dontAsk` 模式，Claude Code 会自动拒绝所有原本会提示的工具调用。Claude 仅运行与您的 `permissions.allow` 规则、[只读 Bash 命令](/docs/zh-CN/permissions#read-only-commands)匹配的操作，以及由 [PreToolUse hook](/docs/zh-CN/permissions#extend-permissions-with-hooks) 批准的调用。在 CI 管道或受限环境中使用此模式，您可以预先定义 Claude 可以执行的操作；会话永远不会等待输入。当此模式处于活动状态时，状态栏显示 `⏵⏵ don't ask on`。

Claude Code 拒绝与您的显式 [`ask` 规则](/docs/zh-CN/permissions#manage-permissions)匹配的调用，而不是提示。它还拒绝内置的 `AskUserQuestion` 工具，即使您的 allow 规则与其匹配，以及您的组织[设置为 `ask`](/docs/zh-CN/mcp#organization-controls-on-connector-tools) 的连接器工具在该设置到达 Claude Code 的会话中。它以相同的方式拒绝标记为 [`_meta["anthropic/requiresUserInteraction"]`](/docs/zh-CN/mcp#require-approval-for-a-specific-tool) 的 MCP 工具，因为其批准卡需要此模式永远不会收集的答案；这需要 Claude Code v2.1.199 或更高版本。

`rm` 和 `rmdir` 移除针对[关键路径](#critical-paths)的操作，如 `rm -rf /` 和 `rm -rf ~`，即使 allow 规则与其匹配或 `PreToolUse` hook 允许它们，也被拒绝。

[网络上的 Claude Code](/docs/zh-CN/claude-code-on-the-web)上的云会话忽略 `defaultMode: "dontAsk"`；有关详细信息，请参阅 [bypassPermissions](#skip-all-checks-with-bypasspermissions-mode)。

在启动时使用标志设置它：

```bash theme={null}
claude --permission-mode dontAsk
```

<h2 id="skip-all-checks-with-bypasspermissions-mode">
  使用 bypassPermissions 模式跳过所有检查
</h2>

`bypassPermissions` 模式禁用权限提示和安全检查，以便工具调用立即执行，包括对[受保护路径](#protected-paths)的写入。

[任何模式都不会自动批准的操作](#actions-no-mode-auto-approves)在此模式下仍会提示。

两个[跨会话消息传递](/docs/zh-CN/cross-session-messaging)保护措施在此模式下仍然适用，以及在具有可用绕过权限的计划模式会话中：

* [`isolatePeerMachines`](/docs/zh-CN/settings-reference#isolatepeermachines)批准提示用于发送到超出此机器的会话的消息仍然出现。
* 当没有[`crossSessionInbound`](/docs/zh-CN/cross-session-messaging#control-inbound-messages)值适用时，Claude Code 会从您的另一个会话中的入站消息保留以供您批准，仅当发送会话将自己标识为也绕过权限提示时才无需询问即可传递。如果您在保留消息时离开权限模式，Claude Code 会重新应用入站规则，并传递任何现在接受的保留消息。

在具有可用绕过权限的会话中，Claude Code 也不强制执行[计划模式的](#analyze-before-you-edit-with-plan-mode)块。Claude 仍然被指示在不编辑的情况下进行计划，但它在计划期间尝试的文件编辑或 shell 命令无需提示即可运行。显式[询问规则](/docs/zh-CN/permissions#manage-permissions)和针对[关键路径](#critical-paths)的 `rm` 和 `rmdir` 删除仍会提示。

<Warning>
  仅在隔离环境（如容器、虚拟机或没有互联网访问的开发容器）中使用此模式，其中 Claude Code 无法损害您的主机系统。
</Warning>

您无法从未启用此模式的会话进入 `bypassPermissions`。在启动时使用[`permissions.defaultMode: "bypassPermissions"`](/docs/zh-CN/settings-reference#permissions-defaultmode)或使用启用标志启用它：

```bash theme={null}
claude --permission-mode bypassPermissions
```

`--dangerously-skip-permissions` 标志是等效的。

Claude Code 在您使用[`--restricted`](/docs/zh-CN/cli-reference#cli-flags)启动的会话中拒绝 `bypassPermissions`。`--restricted` 需要 Claude Code v2.1.248 或更高版本。

第一次使用此模式启动交互式会话时，Claude Code 会显示一个警告对话框，要求您接受对在没有权限检查的情况下执行的操作的责任。Claude Code 将您的接受保存到用户设置，因此该对话框仅出现一次。如果您拒绝，Claude Code 会退出。在[非交互模式](/docs/zh-CN/headless)中不显示对话框，使用 `--bg` 启动的[后台会话](/docs/zh-CN/agent-view)会被拒绝，直到您在交互式会话中接受对话框。

在 Linux 和 macOS 上，当以 root 身份或在 `sudo` 下运行时，Claude Code 拒绝以此模式启动：

```text theme={null}
--dangerously-skip-permissions cannot be used with root/sudo privileges for security reasons
```

在识别的沙箱内自动跳过检查。要在容器中自主运行，请使用[开发容器](/docs/zh-CN/devcontainer)配置，该配置以非 root 用户身份运行 Claude Code。

[网络上的 Claude Code](/docs/zh-CN/claude-code-on-the-web)不遵守来自您的设置文件的 `defaultMode: "bypassPermissions"` 或 `"dontAsk"`，因此存储库的签入设置无法在绕过权限模式下启动云会话。该设置被静默忽略，会话改为以模式下拉菜单中显示的权限模式启动。有关云会话提供的模式，请参阅[切换权限模式](#switch-permission-modes)。

<Warning>
  `bypassPermissions` 不提供针对提示注入或意外操作的保护。对于权限提示少得多的后台安全检查，请改用[自动模式](#eliminate-prompts-with-auto-mode)。管理员可以通过在[托管设置](/docs/zh-CN/managed-settings)中将 `permissions.disableBypassPermissionsMode` 设置为 `"disable"` 来阻止此模式。
</Warning>

<h2 id="protected-paths">
  受保护的路径
</h2>

对一小组路径的写入永远不会自动批准，唯一的例外是 `bypassPermissions` 模式，以及可使用[绕过权限](#skip-all-checks-with-bypasspermissions-mode)的 plan 模式会话。这可以防止意外损坏存储库状态和 Claude 自己的配置。

| 模式                      | 受保护路径写入                                                                                                                            |
| :---------------------- | :--------------------------------------------------------------------------------------------------------------------------------- |
| `default`、`acceptEdits` | 提示                                                                                                                                 |
| `plan`                  | 在[绕过权限](#skip-all-checks-with-bypasspermissions-mode)可用的会话中允许。否则，当[自动模式](#eliminate-prompts-with-auto-mode)在规划期间可用时路由到分类器，当它不可用时提示 |
| `auto`                  | 路由到分类器                                                                                                                             |
| `dontAsk`               | 拒绝                                                                                                                                 |
| `bypassPermissions`     | 允许                                                                                                                                 |

在使用 [`--restricted`](/docs/zh-CN/cli-reference#cli-flags) 启动的会话中，需要 Claude Code v2.1.248 或更高版本，分类器无法批准受保护路径的写入。

设置文件中的 [`permissions.allow`](/docs/zh-CN/permissions#manage-permissions) 规则不会预先批准受保护路径的写入。安全检查在 Claude Code 评估设置中的允许规则之前运行，因此 `~/.claude/settings.json` 或 `.claude/settings.json` 中的条目（如 `Edit(.claude/**)`）不会改变上表中的每个模式结果。在提示的模式中，`.claude/` 写入的提示提供**是的，允许 Claude 在此会话中编辑其自己的设置**，这会在该会话中批准后续的 `.claude/` 写入而无需再次提示。

受保护的目录：

* `.git`
* `.config/git`
* `.vscode`
* `.idea`
* `.husky`
* `.cargo`
* `.devcontainer`
* `.yarn`
* `.mvn`
* `.claude`，除了 `.claude/worktrees`，Claude 在其中存储自己的 git worktrees

受保护的文件：

* `.gitconfig`、`.gitmodules`
* `.bashrc`、`.bash_profile`、`.bash_login`、`.bash_aliases`、`.bash_logout`、`.zshrc`、`.zprofile`、`.zshenv`、`.zlogin`、`.zlogout`、`.profile`、`.envrc`
* `.npmrc`、`.yarnrc`、`.yarnrc.yml`、`.pnp.cjs`、`.pnp.loader.mjs`、`.pnpmfile.cjs`、`bunfig.toml`、`.bunfig.toml`
* `.bazelrc`、`.bazelversion`、`.bazeliskrc`
* `.pre-commit-config.yaml`、`lefthook.yml`、`lefthook.yaml`、`.lefthook.yml`、`.lefthook.yaml`
* `gradle-wrapper.properties`、`maven-wrapper.properties`
* `.devcontainer.json`
* `.ripgreprc`、`pyrightconfig.json`
* `.mcp.json`、`.claude.json`

<h2 id="critical-paths">
  关键路径
</h2>

Claude Code 永远不会让 [`permissions.allow`](/docs/zh-CN/permissions#manage-permissions) 规则或返回 `"allow"` 的 [`PreToolUse` hook](/docs/zh-CN/permissions#extend-permissions-with-hooks) 批准针对关键路径的 `rm` 或 `rmdir` 命令，即使在跳过其他提示的模式下。此断路器防止模型错误。匹配的拒绝规则仍然完全阻止命令。

会发生什么取决于您的权限模式：

| 模式                      | Claude Code 对关键路径移除的处理                                                              |
| :---------------------- | :---------------------------------------------------------------------------------- |
| `default`、`acceptEdits` | 要求您批准它                                                                              |
| `plan`                  | 要求您批准它。当[自动模式在规划期间可用](#analyze-before-you-edit-with-plan-mode)且没有绕过权限可用时，改为将其发送到分类器 |
| `auto`                  | 将其发送到[分类器](#eliminate-prompts-with-auto-mode)                                       |
| `dontAsk`               | 拒绝它                                                                                 |
| `bypassPermissions`     | 要求您批准它                                                                              |

如果显式[询问规则](/docs/zh-CN/permissions#manage-permissions)与命令匹配，Claude Code 即使在 `auto` 模式下也会询问您。在询问的模式中，[`PermissionRequest` hook](/docs/zh-CN/hooks#permissionrequest)可以像回答任何其他一样回答提示。

Claude Code 将 `rm` 或 `rmdir` 目标视为关键路径，当它是以下任何一个时：

* 文件系统根目录
* 顶级目录，意味着根目录的任何直接子目录，如 `/usr`、`/etc` 或 `/data`
* 您的主目录
* Windows 驱动器根目录及其顶级目录，如 `C:\` 和 `C:\Windows`
* 您的工作目录及其父目录
* 您的额外工作目录及其父目录，但仅当移除是其下的 glob 时，如 `rm -rf <dir>/*`。`rm -rf <dir>` 在目录本身上不会触发此检查

Claude Code 也将直接在 shell 变量下的 glob 或尾部斜杠视为关键路径移除，如 `rm -rf "$DIR"/*`，因为当变量为空时命令变成从文件系统根目录的移除。

使用 `$(...)` 或反引号隐藏命令替换中的移除，或使用 `<(...)` 的进程替换，不会跳过检查。Claude Code 找到关键路径移除，无论它位于替换内部（如 `echo "$(rm -rf ~)"`），还是位于同一命令中的其他地方。

<h3 id="remove-item-in-powershell">
  PowerShell 中的 Remove-Item
</h3>

当您启用 [PowerShell 工具](/docs/zh-CN/tools-reference#powershell-tool)时，Claude Code 给 `Remove-Item` 自己的检查，与 `rm` 关键路径列表分开。结果取决于目标，第一个匹配的情况适用：

* **系统路径**：文件系统根目录及其顶级目录、驱动器根目录及其顶级目录和您的主目录。Claude Code 在每种模式下拒绝命令，不询问您。
* **通配符**：裸 `*` 或任何以 `/*` 或 `\*` 结尾的目标，包括 shell 变量下的 glob，如 `$dir/*`。Claude Code 在每种模式下拒绝命令，不询问您，在[分类器](#eliminate-prompts-with-auto-mode)看到它之前。
* **您的工作目录或其父目录之一，带有 `-Recurse`**：Claude Code 将命令视为任何其他需要在您的权限模式下批准的命令，因此它在询问的模式下询问您，在 `auto` 模式下将其发送到分类器，在 `dontAsk` 模式下拒绝它。`bypassPermissions` 模式跳过此检查。

<h2 id="see-also">
  另请参阅
</h2>

* [权限](/docs/zh-CN/permissions)：允许、询问和拒绝规则；托管策略
* [配置自动模式](/docs/zh-CN/auto-mode-config)：告诉分类器您的组织信任哪些基础设施
* [Hooks](/docs/zh-CN/hooks)：通过 `PreToolUse` 和 `PermissionRequest` hooks 的自定义权限逻辑
* [安全](/docs/zh-CN/security)：保障措施和最佳实践
* [沙箱](/docs/zh-CN/sandboxing)：Bash 命令的文件系统和网络隔离
* [非交互模式](/docs/zh-CN/headless)：使用 `-p` 标志运行 Claude Code
