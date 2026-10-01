> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 为您的组织管理 mods

> 使用托管设置控制 Claude Code mods：停止用户安装的 mods、仅允许您自己的 mods、查看 mod 可以执行的操作，以及使用您自己的 mod 强制执行策略。

[mod](/docs/zh-CN/plugins/mods/overview) 是在 Claude Code 内运行代码的插件，具有安装它的用户的权限。Mods 不是沙箱化的。通过[托管设置](/docs/zh-CN/managed-settings)，您可以决定 mods 是否在用户的机器上运行、运行哪些 mods 以及运行顺序。您还可以安装自己的 mod，用于监视或拒绝其他 mods 的操作。

本页面适用于为 Claude Code 部署托管设置的人员，无论是通过文件、MDM 还是从 claude.ai 管理控制台部署。在 Claude Code v2.1.287 及更高版本中，Mods 默认处于启用状态。从与您要执行的操作相匹配的部分开始：

* **排除用户自己的 mods，有或没有您自己的 mods**：[停止用户安装的 mods 加载](#stop-user-installed-mods-from-loading)
* **查看当您不做任何更改时用户会获得什么**：[了解默认情况下会发生什么](#know-what-happens-by-default)
* **保持 mods 开启并设置其他限制**：[选择允许的程度](#choose-how-much-to-allow)

<Note>
  这些情况在其他页面上有介绍：

  * **您之前没有部署过托管设置**：从[部署托管设置](/docs/zh-CN/managed-settings)开始
  * **您想控制用户可以安装哪些插件**：请参阅[为您的组织管理插件](/docs/zh-CN/plugins/org)
</Note>

<h2 id="stop-user-installed-mods-from-loading">
  停止用户安装的 mods 加载
</h2>

要防止用户带来的每个 mod 加载，请在[内置保护](#know-what-happens-by-default)上设置 `allowManagedModsOnly` 选项，这是一个策略 mod，Claude Code 在用户安装的每个 mod 之前加载。该选项位于 `pluginConfigs` 下的托管设置中，由 `cc-plugin-sec-default@builtin` 键入：

```json managed-settings.json theme={null}
{
  "pluginConfigs": {
    "cc-plugin-sec-default@builtin": {
      "options": {
        "allowManagedModsOnly": true
      }
    }
  }
}
```

设置了托管设置中的选项后：

* **用户带来的任何 mod 都不会加载**：这包括用户安装的插件中的 mod、使用 `--plugin-dir` 加载的 mod 以及[Claude 在会话期间编写的](/docs/zh-CN/plugins/mods/create#ask-claude-for-a-mod) mod
* **您组织的 mods 仍然加载**：[计为您组织的](#install-your-organizations-mods) mod 不会被检查。所有其他 mod 都计为用户的 mod，不会加载。这包括您从 GitHub 或其他远程市场启用的插件中的 mod，以及您的组织为其成员在 claude.ai 上启用的 mod。如果没有计为您的 mod，则不会加载任何已安装的 mod。
* **用户无法撤销它**：保护程序仅从托管设置读取选项，因此用户、项目或本地设置文件中的相同条目，或使用 `--settings` 传递的文件中的条目不会改变任何内容
* **文件或 MDM 策略涵盖每个提供商**：当您以文件形式或通过 MDM 提供选项时，它在 Amazon Bedrock、Google Cloud 的 Agent Platform 和 Microsoft Foundry 上的工作方式相同。对于从 claude.ai 管理控制台的交付，请参阅[平台可用性](/docs/zh-CN/server-managed-settings#platform-availability)
* **用户的其他自定义保持工作**：他们[设置文件中的 hooks](/docs/zh-CN/hooks)、状态行和 `/goal` 不受影响
* **内置 mods 继续运行**：内置于 Claude Code 的 mods，例如 `AGENTS.md` 支持，各有[自己的开关](/docs/zh-CN/plugins/mods/overview#mods-built-into-claude-code)

要确认用户机器上的选项，请使用 `--plugin-dir` 和包含 mod 的目录路径（例如 `claude --plugin-dir ./first-mod`）启动该机器上的 Claude Code。mod 的 hooks 不会运行，成绩单和调试日志会显示[保护程序的消息](/docs/zh-CN/plugins/mods/troubleshoot#messages-from-the-built-in-guard)，其中命名了 mod 和 `allowManagedModsOnly`。如果 mod 加载，请参阅[检查策略是否生效](/docs/zh-CN/managed-settings#check-that-a-policy-is-in-force)和[决定选项是否生效的规则](#set-options-on-the-built-in-guard)。

如果您在早期访问期间将 `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS` 设置为 `0`，请将其替换为此选项。Claude Code v2.1.287 及更高版本在任何值下都会忽略该变量，因此那里的 `0` 会使 mods 保持开启。

<h2 id="know-what-happens-by-default">
  了解默认情况下会发生什么
</h2>

如果您没有自己的 mod 设置，这就是您的用户会获得的：

* **Mods 处于开启状态。** 用户可以安装包含来自您的插件设置允许的任何市场的 mod 的插件，或使用 `--plugin-dir` 从目录加载一个。
* **内置保护程序首先运行。** Claude Code 在用户安装的每个 mod 之前加载一个名为 `sec-default@builtin` 的内置 mod。用户无法将其关闭。`/plugin` 和调试日志将其列为 `cc-plugin-sec-default`。保护程序在以下任一情况为真时加载：

  * 机器有托管设置
  * 用户使用 Team 或 Enterprise 计划登录到 Claude Code

  使用 API 密钥进行身份验证的用户，或通过 Amazon Bedrock、Google Cloud 的 Agent Platform 或 Microsoft Foundry，仅在具有托管设置的机器上获得保护程序。
* **保护程序保护您管理的内容。** 用户的 mod 无法更改您的托管 hooks 接收或决定的内容、系统提示、您的托管 `CLAUDE.md` 和其他托管说明、任何 mod 读取的设置内容，或您的托管 MCP 服务器的工具和描述。
* **允许所有其他内容。** 保护程序不添加其他限制。用户的 mod 仍然可以读写文件、启动进程、发出网络请求、重写工具调用和提示、拒绝工具调用、批准否则会提示的工具调用，以及在界面中绘制，所有这些都具有该用户的权限。
* **拒绝规则和您的托管 hooks 优先。** 保护程序加载的地方，用户的 mod 无法批准 `deny` 规则拒绝的调用，无论哪个设置文件持有该规则。来自托管设置中 `PreToolUse` hook 的块也是最终的。两者都适用于 Claude 的工具调用。两者都不适用于 mod 自己的 [`$.fs` 和 `$.process` 调用](/docs/zh-CN/plugins/mods/api#reach-files-processes-and-the-network)：即使 `Read(.env)` 被拒绝，mod 仍然可以使用 `$.fs.read` 读取该文件或启动执行此操作的程序。要限制这些调用，请防止 mod 加载或在[策略 mod](#enforce-a-policy-with-a-mod-of-your-own) 中挂接调用。
* **其他权限检查可以被覆盖。** 批准工具调用的用户 mod 可以批准 `ask` 规则会提示的调用，或 `PreToolUse` hook 在托管设置外阻止的调用。在自动模式下，mod 批准的调用运行时不进行分类器检查。

保护程序的源代码在 [Claude Code 存储库的 `mods/sec-default` 目录](https://github.com/anthropics/claude-code/tree/main/mods/sec-default)中是公开的。

<h3 id="know-which-controls-still-apply">
  了解哪些控制仍然适用
</h3>

Mods 不会替换您已有的控制：

* **设置 hooks 继续工作。** 设置文件和插件的 `hooks/hooks.json` 中的命令、HTTP、提示和代理 hooks 照常运行，与 mods 一起。关于它们的任何内容都没有被弃用。
* **拒绝规则在保护程序加载的地方优先。** 用户的 mod 无法批准 `deny` 规则拒绝的调用，除非您设置 [`allowModsToOverrideDenyRules`](#set-options-on-the-built-in-guard)。
* **托管 hooks 首先运行。** 托管设置中的 `PreToolUse` hook 在任何 mod 看到工具调用之前运行，其块是最终的。如果 mod 随后重写调用，您的托管 hooks 在重写的调用上再次运行，因此块仍然适用。来自其他设置文件和插件的 `PreToolUse` hooks 在最后一个 mod 之后运行，因此返回自己结果代替运行工具的 mod 会阻止这些运行。请参阅[mods 运行的顺序](/docs/zh-CN/plugins/mods/events#the-order-mods-run-in)。
* **网络策略涵盖 `$.http.fetch`。** 如果您的组织关闭了网络获取，或为会话关闭了非必要的网络流量，Claude Code 会拒绝 mod 使用 `$.http.fetch` 发出的网络请求。该策略不涵盖 mod 使用 `$.process.run` 启动的程序。该程序使用用户自己的访问权限到达网络。
* **插件控制涵盖 mods。** Mod 是一个插件，因此[限制用户可以安装的内容的设置](/docs/zh-CN/plugins/org#restrict-what-users-can-install)，例如 `strictKnownMarketplaces`，决定它是否可以被安装。
* **Mods 无法更改权限提示。** Mod 可以重新设置 Claude Code 界面的大部分样式，但不能更改权限提示，因此它无法更改提示显示的内容。Mod 仍然可以在提示出现之前批准或拒绝工具调用，如[了解默认情况下会发生什么](#know-what-happens-by-default)所述。
* **信任提示首先出现。** 在用户尚未信任的目录中的交互式会话中，在他们回答信任提示之前，没有 mod 加载。
* **`--safe-mode` 关闭已安装的 mods，包括您的。** 使用 `claude --safe-mode` 启动会话以检查 mod 是否导致了问题。

这些控制中的任何一个都不会沙箱化 mod。您允许的 mod 以用户身份运行，具有用户对文件、进程和网络的访问权限。

<h2 id="decide-whether-to-leave-mods-on">
  决定是否保持 mods 开启
</h2>

Mod 可以做的比插件的其他部分更多，因为它在 Claude Code 内运行。它看到每个提示和工具调用，可以更改它们，并可以在权限提示出现之前允许或拒绝工具调用。

用户可以加载什么作为 mod 取决于您已有的插件控制：

| 您今天的插件控制 | 用户可以加载什么作为 mod |
| :- | :- |
| 无 | 来自任何市场的 mod、来自任何带有 `--plugin-dir` 的目录的 mod，或 Claude 在会话期间编写的 mod |
| 市场允许列表 | 来自您允许的市场的 mod，或来自任何带有 `--plugin-dir` 的目录的 mod。Claude 在会话期间编写的 mod 仅在允许列表[包含 `skills-dir`](/docs/zh-CN/plugins/org#keep-skills-directory-plugins-loading) 时加载。 |
| 市场允许列表和 `disableSideloadFlags` | 来自您允许的市场的 mod |

[为您的组织管理插件](/docs/zh-CN/plugins/org)列出了插件加载的每种方式以及控制每种方式的设置。

要在用户安装市场中的 mods 之前检查它们，请参阅[查看 mod 可以执行的操作](#review-what-a-mod-can-do)。要在您完成此操作之前排除用户的 mods，请参阅[停止用户安装的 mods 加载](#stop-user-installed-mods-from-loading)。

<h3 id="review-what-a-mod-can-do">
  查看 mod 可以执行的操作
</h3>

您可以看到 mod 能够做什么而无需运行它。在您的 shell 中，在插件的目录上运行 `claude plugin validate`：

```bash theme={null}
claude plugin validate ./some-mod
```

输出中的两行描述了 mod 的代码：

```text theme={null}
  ❯ ./register.js hooks: session.start, tool.call, ui.render{component=Pane}
  ❯ ./register.js calls: $.fs.read, $.http.fetch, $.store.set, $.ui.open
```

`hooks:` 行列出了 mod 接收的事件。`calls:` 行列出了其代码调用的 mods API 方法。[mods API](/docs/zh-CN/plugins/mods/api)（在 mod 的代码中写作 `$`）是 mod 到达文件、进程和网络的方式。Claude Code 拒绝加载以此命令无法读取的方式使用 mods API 的 mod。

查看 `calls:` 行以获取这些：

| 调用 | 它的含义 |
| :- | :- |
| `$.fs.read`, `$.fs.write` | 读取或写入用户可以访问的任何地方的文件 |
| `$.process.run`, `$.process.spawn` | 以用户身份启动程序 |
| `$.http.fetch` | 发出网络请求 |
| `$.env.get`, `$.settings.read` | 读取环境变量和设置，可以保存 API 密钥。输出中的 `env reads:` 行命名每个变量。 |
| `$.env.set` | 为 Claude Code 以及它启动的每个命令和 MCP 服务器设置环境变量，可以改变这些程序运行的内容。`env writes:` 行命名每个变量。 |
| `$.mcp.call` | 在连接的 MCP 服务器上调用工具，在会话的权限规则下 |
| `$.model.complete` | 使用用户的计划或 API 密钥进行模型调用 |
| `$.prompt.submit` | 提交提示，可以将其作为用户自己的话语发送 |
| `$.session.send` | 发送另一个会话或子代理的 Claude 读取的消息 |

在 `hooks:` 行中，[`tool.call`](/docs/zh-CN/plugins/mods/reference#tools) 和 [`prompt.submit`](/docs/zh-CN/plugins/mods/reference#prompts-and-what-claude-reads) 意味着 mod 看到每个工具调用和每个提示，并可以更改它们。[`session.append`](/docs/zh-CN/plugins/mods/reference#session) 意味着 mod 可以在存储之前重写对话的每一行。[`ui.render{component=AskUserQuestion}`](/docs/zh-CN/plugins/mods/interface#change-what-claude-code-already-draws) 意味着 mod 可以重新绘制 Claude 用来询问用户问题的对话框。`tool.check` 意味着 mod 可以在权限提示出现之前批准或拒绝工具调用。[了解默认情况下会发生什么](#know-what-happens-by-default)列出了您的哪些规则和 hooks 优先于其答案。

<h2 id="choose-how-much-to-allow">
  选择允许的程度
</h2>

Mod 策略的范围从根本没有已安装的 mods 到用户选择的任何 mod，以及您自己的 mod 检查其他 mods，每一个都是几个托管设置。在第一列中找到您想要的策略，并设置第二列命名的内容。[部署托管设置](/docs/zh-CN/managed-settings)涵盖托管设置的位置。

| 您想要什么 | 设置 |
| :- | :- |
| 没有已安装的 mods，hooks 保持不变 | 设置 [`allowManagedModsOnly`](#set-options-on-the-built-in-guard) 并且不部署您自己的 mods |
| 没有已安装的 mods 和根本没有 hooks，包括您的托管 hooks | 将 `disableAllHooks` 设置为 `true` |
| 仅您组织的 mods | 设置保护程序的 [`allowManagedModsOnly` 选项](#stop-user-installed-mods-from-loading)，并[安装您的 mods](#install-your-organizations-mods) 以便它们计为您的 |
| 来自您批准的市场的任何 mod | 保持您的[市场限制](/docs/zh-CN/plugins/org#restrict-what-users-can-install)，并将 `disableSideloadFlags` 设置为 `true` |
| 任何 mod，您自己的 mod 检查其他 mods | [安装您的 mod](#install-your-organizations-mods)，并在 `prependPlugins` 中与 `sec-default@builtin` 一起列出它 |

每个设置的作用：

* **`allowManagedModsOnly`**：内置保护程序上的选项。用户自己的 mods 不加载，他们的设置 hooks、状态行和 `/goal` 继续工作。[停止用户安装的 mods 加载](#stop-user-installed-mods-from-loading)列出了它涵盖的内容。
* **`allowManagedHooksOnly`**：更广泛的设置。仅[您组织的 mods](#install-your-organizations-mods) 和内置于 Claude Code 的 mods 加载。用户自己安装的 mod 不加载。该设置还阻止用户自己的设置文件中的 hooks。在设置之前，请阅读[`allowManagedHooksOnly` 下运行什么](/docs/zh-CN/settings-reference#what-runs-under-allowmanagedhooksonly)。
* **`disableAllHooks`**：最广泛的设置。在托管设置中，它停止每个已安装插件中的 mods，包括您的，并关闭设置文件中的每个 hook，因此您的托管设置中的 `PreToolUse` hook 不再阻止任何内容。自定义状态行和 `/goal` 也停止工作。在设置之前，请阅读[`disableAllHooks`](/docs/zh-CN/settings-reference#disableallhooks)。
* **`disableSideloadFlags`**：在启动时拒绝 `--plugin-dir` 和 `--plugin-url`，因此没有人从目录加载 mod，并防止 Claude 在会话期间编写的 mods 加载。该设置还拒绝 `--agents` 和 `--mcp-config`。在设置之前，请阅读[`disableSideloadFlags`](/docs/zh-CN/settings-reference#disablesideloadflags)。

内置于 Claude Code 的 Mods，例如 `AGENTS.md` 支持，不受这些设置的影响。每个都有[自己的开关](/docs/zh-CN/plugins/mods/overview#mods-built-into-claude-code)。

mod 未加载的用户在其调试日志中找到原因。[拒绝消息](/docs/zh-CN/plugins/mods/troubleshoot#refusal-messages)列出了 `allowManagedHooksOnly` 和 `disableAllHooks` 的行，[来自内置保护程序的消息](/docs/zh-CN/plugins/mods/troubleshoot#messages-from-the-built-in-guard)有 `allowManagedModsOnly` 的行。

<h3 id="set-options-on-the-built-in-guard">
  在内置保护程序上设置选项
</h3>

内置保护程序采用两个选项。在托管设置中的 `pluginConfigs` 下设置它们，由 `cc-plugin-sec-default@builtin` 键入，如[停止用户安装的 mods 加载](#stop-user-installed-mods-from-loading)中的示例所示。

该表给出了您的用户在每个选项未设置和设置为 `true` 时获得的内容：

| 选项 | 未设置 | `true` |
| :- | :- | :- |
| `allowManagedModsOnly` | 用户自己的 mods 加载 | 仅[您组织的 mods](#install-your-organizations-mods) 和内置于 Claude Code 的 mods 加载。Claude Code 拒绝所有其他 mods，包括用户安装的或使用 `--plugin-dir` 命名的。 |
| `allowModsToOverrideDenyRules` | 拒绝规则优先于用户的 mods | 批准工具调用的用户 mod 可以批准 `deny` 规则拒绝的调用 |

这些规则决定选项是否生效：

* **id 在这里有一个拼写**：Claude Code 仅在 `cc-plugin-sec-default@builtin` 下读取选项。`prependPlugins` 也接受 `sec-default@builtin`，而 `pluginConfigs` 不接受。
* **仅托管设置计数**：用户、项目或本地设置文件中的相同条目，或使用 `--settings` 传递的文件中的相同条目既不设置选项也不放松选项
* **保护程序必须加载**：如果您设置 `prependPlugins`，[在列表中命名保护程序](#install-your-organizations-mods)。保护程序不加载的地方，两个选项都不适用。
* **保护程序失败关闭**：如果保护程序无法读取托管设置，它会拒绝每个用户的 mod 加载。如果它无法检查用户的 mod 批准的调用的拒绝规则，它会拒绝该调用。

[来自内置保护程序的消息](/docs/zh-CN/plugins/mods/troubleshoot#messages-from-the-built-in-guard)是您的用户在任一选项适用时看到的内容。

<h2 id="run-your-organization’s-own-mods">
  运行您组织自己的 mods
</h2>

您可以将自己的 mods 部署给每个用户，选择它们相对于用户 mods 的运行位置，并使用一个来强制执行策略。

<h3 id="install-your-organizations-mods">
  安装您组织的 mods 并设置顺序
</h3>

您组织的 mods 在用户 mods 不存在的地方加载，并且可以在用户 mods 之前运行，因此 Claude Code 必须能够判断 mod 是否来自您。只有当以下所有条件都为真时，它才会将 mod 视为您组织的：

* 托管的 `enabledPlugins` 将 mod 的插件设置为 `true`
* 托管设置通过绝对路径将插件的 [marketplace](/docs/zh-CN/plugins/create-marketplace) 命名为用户机器上的目录。`extraKnownMarketplaces` 条目可以做到这一点，并且也为用户注册 marketplace。
* marketplace 通过相对路径列出插件，因此 Claude Code [从该目录就地加载它](/docs/zh-CN/plugins/loading#in-place-and-copied-plugins)

为了满足这些条件，让您的设备管理将 marketplace 目录复制到每台机器上的相同路径。使该目录及其上方的每个目录仅可由管理员写入，就像托管设置文件一样。任何可以在那里写入的人都可以重写您的 mod。您从 claude.ai 管理员控制台交付的托管设置可以携带这些密钥，但它们无法将目录放在机器上。

该目录包含 marketplace 的清单和插件：

```text theme={null}
/opt/acme/claude-plugins/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    └── acme-guard/
        ├── .claude-plugin/
        │   └── plugin.json
        └── hooks/
            ├── hooks.json
            └── register.js
```

清单通过相对于该目录的路径列出插件：

```json /opt/acme/claude-plugins/.claude-plugin/marketplace.json theme={null}
{
  "name": "acme-tools",
  "owner": { "name": "Acme" },
  "plugins": [
    { "name": "acme-guard", "source": "./plugins/acme-guard", "description": "Acme policy mod" }
  ]
}
```

Claude Code 复制到其缓存中的插件计为用户的，即使托管的 `enabledPlugins` 启用了它。这涵盖了来自 GitHub、git、URL 或 npm 源的每个插件。其 mod 在用户 mods 中运行，`prependPlugins` 和 `appendPlugins` 跳过它，并且它不在 `allowManagedModsOnly` 或 `allowManagedHooksOnly` 下加载。用户的调试日志有一行以插件的 id 和 `is enabled by managed settings, but` 开头。

Claude Code 每次即将采取行动（例如运行工具）时都会引发一个事件，并依次将其传递给每个 mod。计为您的 mod [在用户 mods 之前运行](/docs/zh-CN/plugins/mods/events#the-order-mods-run-in)，即使您没有在任何地方列出它。要设置其位置，请在两个设置之一中列出其 id。id 是插件的名称、`@` 和 marketplace 的名称，例如 `acme-guard@acme-tools`。

* **`prependPlugins`**：您的 mod 在任何用户 mod 之前看到每个事件，在之后看到每个结果。它可以更改事件、拒绝事件或跳过用户 mods。
* **`appendPlugins`**：您的 mod 在每个用户 mod 之后运行，因此它只看到这些 mods 传递的事件，以及它们传递的形式

此示例在 `/opt/acme/claude-plugins` 声明 `acme-tools` marketplace，启用来自它的 `acme-guard`，并首先运行该 mod，内置保护在其后：

```json managed-settings.json theme={null}
{
  "extraKnownMarketplaces": {
    "acme-tools": {
      "source": { "source": "directory", "path": "/opt/acme/claude-plugins" }
    }
  },
  "enabledPlugins": { "acme-guard@acme-tools": true },
  "prependPlugins": ["acme-guard@acme-tools", "sec-default@builtin"]
}
```

每个密钥做一项工作：

* **`extraKnownMarketplaces`**：命名保存 `acme-tools` marketplace 的目录。`path` 是包含 `.claude-plugin/marketplace.json` 的目录的绝对路径。
* **`enabledPlugins`**：为接收这些托管设置的每个用户打开 `acme-guard`
* **`prependPlugins`**：将 `acme-guard` 放在第一位，内置保护放在第二位，都在用户安装的任何 mod 之前。Claude Code 遵循您列出的顺序。

要确认用户的机器收到了设置，请参阅 [检查策略是否生效](/docs/zh-CN/managed-settings#check-that-a-policy-is-in-force)。

要确认 mod 运行的位置，请在该机器上使用 `claude --debug` 启动会话，并在 [调试日志](/docs/zh-CN/plugins/mods/troubleshoot#read-the-debug-log) 中搜索 mod 的 id：

* **`hooks module acme-guard@acme-tools loaded`，带有 `tier prepend`**：mod 计为您组织的，并首先运行
* **同一行带有 `tier user`**：Claude Code 将其视为用户的 mod。第二行 `prependPlugins names acme-guard@acme-tools, which is not an enabled managed plugin with a hooks module; skipped` 表示列表跳过了它。

这些规则决定了两个列表中哪些 id 生效：

* **列表替换默认值**：当您在托管设置中设置 `prependPlugins` 时，在其中命名 `sec-default@builtin` 以保留内置保护。保护是内置的，不需要 `enabledPlugins` 条目。
* **您自己的 id 必须计为您的**：在托管设置中，Claude Code 跳过其插件不满足组织 mod 三个条件的 id
* **存储库无法设置它们**：Claude Code 从托管设置读取两个设置，从不从存储库的设置文件读取。用户可以在 `~/.claude/settings.json` 中设置它们以仅在没有托管设置的机器上对其自己的 mods 进行排序，并且仅当他们未使用 Team 或 Enterprise 计划登录时。在其他任何地方，Claude Code 忽略用户设置中的两个密钥。那里的列表既不添加也不删除内置保护。

<h3 id="enforce-a-policy-with-a-mod-of-your-own">
  使用您自己的 mod 强制执行策略
</h3>

要阻止每个用户的 mod，您不需要自己的 mod。设置 [`allowManagedModsOnly`](#stop-user-installed-mods-from-loading)。当您想允许某些用户的 mods 并拒绝其他的，或记录 mods 的作用时，编写策略 mod。

每次另一个 mod 即将加载时，您的 mod 会收到 `claude plugin validate` 打印的列表，在名为 [`plugin.register`](/docs/zh-CN/plugins/mods/reference#other-mods) 的事件中。`prependPlugins` 中的 mod 可以读取该列表并拒绝该 mod。它也可以 [按名称钩住任何 mods API 调用](/docs/zh-CN/plugins/mods/api#reach-files-processes-and-the-network) 以记录或拒绝每个其他 mod 的该调用。名称是没有 `$.` 的方法，因此 `fs.write` 上的钩子看到每个 `$.fs.write` 调用。

此策略 mod 拒绝任何用户的 mod，其自己的代码调用 `$.process.run` 或 `$.process.spawn`。它也保留审计日志，将每个工具调用和每个 mod 写入的文件写入调试日志。因为它首先运行，日志记录了在任何用户 mod 更改之前请求的内容。将其保存为 `acme-guard/hooks/register.js`：

```javascript acme-guard/hooks/register.js theme={null}
// 用户 mod 不得调用的方法，每个拼写为 namespace.method
const BLOCKED_CALLS = ['process.run', 'process.spawn']

export function register(on) {
  // 每次另一个 mod 即将加载时运行
  on('plugin.register', async ($, e, next) => {
    // 保留该 mod 代码中在阻止列表上的调用
    const blocked = e.uses.calls.filter((call) => BLOCKED_CALLS.includes(call))
    if (e.tier === 'user' && blocked.length > 0) {
      // 返回 refuse 阻止 mod 加载，文本是原因
      return { refuse: 'Acme policy: mods may not call ' + blocked.join(', ') }
    }
    // 让每个其他 mod 加载
    return next(e)
  })

  // 记录每个工具调用，然后让它继续不变
  on('tool.call', async ($, e, next) => {
    $.ui.log('audit tool.call ' + e.tool, { to: 'debug' })
    return next(e)
  })

  // 记录哪个 mod 写了文件，然后是路径，引用因为 mod 选择了它
  on('fs.write', async ($, e, next) => {
    $.ui.log('audit fs.write by ' + next.origin.plugin + ' ' + JSON.stringify(e.path), { to: 'debug' })
    return next(e)
  })
}
```

该文件注册三个钩子：

* **`plugin.register`**：决定另一个 mod 是否加载。它拒绝调用阻止方法的用户 mod，并传递每个其他 mod。
* **`tool.call`**：为每个工具调用向调试日志写入一行，例如 `audit tool.call Bash`，并且不改变任何内容
* **`fs.write`**：为每个 `$.fs.write` 调用另一个 mod 进行的写入一行，例如 `audit fs.write by reader "/tmp/notes.md"`，并且不改变任何内容。mod 的名称首先出现，路径被引用，因此 mod 选择的路径无法冒充该行的另一个字段。

`plugin.register` 钩子读取事件的两个字段：

* **`e.tier`**：mod 将运行的位置，`prepend`、`user`、`append` 或 `builtin` 之一。每个人安装的每个 mod 都是 `user`。
* **`e.uses.calls`**：mod 调用的 mods API 方法，每个拼写为 `namespace.method`，例如 `process.run`，不带 `claude plugin validate` 打印的 `$.`

当用户安装调用 `$.process.run` 的 mod 时，mod 不加载，其调试日志有一行以 `refused by acme-guard:` 和您的原因结尾。拒绝也到达 [热重新加载插件目录的会话](/docs/zh-CN/plugins/mods/troubleshoot#find-out-why-a-mod-does-nothing) 中的记录。要在不拒绝整个 mod 的情况下阻止调用，请从该调用名称上的钩子返回 `{ deny: 'your reason' }`。

要将审计行发送到调试日志以外的地方，请从相同的钩子调用 `$.http.fetch`。

会话可以在没有您的 mod 的情况下运行。如果运行已安装 mods 的工作线程 [崩溃三次](/docs/zh-CN/plugins/mods/troubleshoot#mods-that-run-in-the-hooks-worker-are-off-for-this-session)，Claude Code 卸载每个不是内置的 mod，包括您的，直到用户运行 `/reload-plugins` 或启动新会话。并且使用 `--safe-mode` 启动 Claude Code 的用户运行时没有已安装的 mods，包括您的。

[创建 mod](/docs/zh-CN/plugins/mods/create) 涵盖 mod 需要的文件。[测试判断其他 mods 的 mod](/docs/zh-CN/plugins/mods/test#test-a-mod-that-judges-other-mods) 有此策略 mod 的测试文件。

<h4 id="refuse-mods-when-your-check-fails">
  当您的检查失败时拒绝 mods
</h4>

如果您的 `plugin.register` 钩子抛出或超过其时间限制，Claude Code 跳过钩子，因此检查失败打开，它正在检查的 mod 加载。要失败关闭并拒绝用户 mods，将检查移到命名函数中并添加返回拒绝的 `.catch` 处理程序。此版本的文件仅显示 `plugin.register` 钩子，因此在 `register` 中保留第一个版本的两个审计钩子：

```javascript acme-guard/hooks/register.js theme={null}
const BLOCKED_CALLS = ['process.run', 'process.spawn']

// 与之前相同的检查，移到其自己的函数中
async function checkMod($, e, next) {
  const blocked = e.uses.calls.filter((call) => BLOCKED_CALLS.includes(call))
  if (e.tier === 'user' && blocked.length > 0) {
    return { refuse: 'Acme policy: mods may not call ' + blocked.join(', ') }
  }
  return next(e)
}

export function register(on) {
  // 处理程序仅在 checkMod 抛出或超过其时间限制时运行
  on('plugin.register', checkMod).catch(async ($, e, next) => {
    // 让您组织的 mods 和内置 mods 加载
    if (e.tier !== 'user') return next(e)
    // 拒绝无法检查的用户 mod
    return { refuse: 'Acme policy check failed, so this mod was not loaded' }
  })
}
```

处理程序就位后，正在检查时检查抛出或超时的 mod 不加载，拒绝行携带第二个原因，如 `refused by acme-guard: Acme policy check failed, so this mod was not loaded`。处理程序将 `user` 层外的每个 mod 传递给 `next(e)`，因此失败的检查不会停止您组织列出的 mods。[处理失败的钩子](/docs/zh-CN/plugins/mods/events#handle-a-hook-that-fails) 涵盖其他事件的 `.catch`。

<h2 id="next-steps">
  后续步骤
</h2>

* [插件安全](/docs/zh-CN/plugins/security)：任何插件可以在用户机器上执行的操作，以及如何在安装前查看一个
* [Mods 概述](/docs/zh-CN/plugins/mods/overview)：什么是 mod 以及它与 hooks、skills 和 MCP 服务器的比较
* [mods 运行的顺序](/docs/zh-CN/plugins/mods/events#the-order-mods-run-in)：`prependPlugins` 和 `appendPlugins` 如何与用户的 mods 配合
* [设置和环境变量](/docs/zh-CN/plugins/mods/reference#settings-and-environment-variables)：本页命名的每个设置在一个表中
