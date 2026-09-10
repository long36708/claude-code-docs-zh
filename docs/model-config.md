> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 模型配置

> 配置 Claude Code 使用的模型、工作量级别、扩展上下文和自动压缩窗口

<h2 id="available-models">
  可用的模型
</h2>

对于 Claude Code 中的 `model` 设置，你可以配置以下任一项：

* 一个**模型别名**
* 一个**模型名称**
  * Anthropic API：一个完整的\*\*[模型名称](https://platform.claude.com/docs/en/about-claude/models/overview)\*\*
  * Amazon Bedrock：一个推理配置文件 ARN
  * Microsoft Foundry：一个部署名称
  * Google Cloud 的 Agent Platform：一个版本名称

有关哪种模型和工作量级别适合不同类型工作的指导，请参阅博客上的 [Choosing a Claude model and effort level in Claude Code](https://claude.com/blog/claude-model-and-effort-level-in-claude-code)。

<Note>
  `ANTHROPIC_BASE_URL` 改变请求发送的位置，而不是哪个模型回答它们。要通过 LLM 网关路由 Claude，请参阅 [LLM gateways](/docs/zh-CN/llm-gateway)。
</Note>

<h3 id="model-aliases">
  模型别名
</h3>

使用模型别名来选择模型设置，而无需记住确切的版本号：

| 模型别名             | 行为                                                                                                                                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`default`**    | 特殊值，清除任何模型覆盖并恢复到[你的账户的运行时默认值](#default-model-setting)。本身不是模型别名                                                                                                                                                                        |
| **`best`**       | 在 Fable 对你可用的地方使用 [`fable` 别名解析到的模型](#fable-alias-resolution)，否则使用与 `opus` 相同的模型                                                                                                                                                      |
| **`fable`**      | 为你最困难和运行时间最长的任务使用[你的提供商的 Fable 模型](#fable-alias-resolution)                                                                                                                                                                           |
| **`sonnet`**     | 为日常编码任务使用最新的 Sonnet 模型                                                                                                                                                                                                                |
| **`opus`**       | 为复杂推理任务使用最新的 Opus 模型                                                                                                                                                                                                                  |
| **`haiku`**      | 为简单任务使用快速高效的 Haiku 模型                                                                                                                                                                                                                 |
| **`sonnet[1m]`** | 为长会话使用具有 [100 万令牌上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows#context-window-sizes-by-model) 的 Sonnet。当 `sonnet` 已经解析到具有其原生 1M 窗口的 Sonnet 5 时无效；在 [LLM 网关](/docs/zh-CN/llm-gateway) 后面，为 Sonnet 5 选择 1M 窗口 |
| **`opus[1m]`**   | 为长会话使用具有 [100 万令牌上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows#context-window-sizes-by-model) 的 Opus                                                                                                   |
| **`opusplan`**   | 特殊模式，在 Plan Mode 期间使用 `opus`，然后在执行期间切换到 `sonnet`                                                                                                                                                                                      |

`opus` 和 `sonnet` 别名解析到的版本取决于提供商：

| 提供商                                                     | `opus`   | `sonnet`   |
| :------------------------------------------------------ | :------- | :--------- |
| Anthropic API                                           | Opus 5   | Sonnet 5   |
| [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) | Opus 5   | Sonnet 4.6 |
| Amazon Bedrock、Google Cloud 的 Agent Platform            | Opus 5   | Sonnet 4.5 |
| Microsoft Foundry                                       | Opus 4.6 | Sonnet 4.5 |

<span id="fable-alias-resolution" />

除非你设置 `ANTHROPIC_DEFAULT_FABLE_MODEL`，否则 `fable` 别名解析到 Fable 5.1，除了在 [Claude apps gateway](/docs/zh-CN/claude-apps-gateway) 会话中，其中 `fable` 和 `best` 解析到 Fable 5。在 v2.1.257 之前，`fable` 在每个提供商上都解析到 Fable 5。

未配置为提供 `claude-fable-5-1` 的网关会拒绝对该模型的请求。要通过提供它的网关使用 Fable 5.1，请使用 `/model claude-fable-5-1` 选择它。

当别名解析到较旧的模型时，可以通过显式选择完整模型名称或设置 `ANTHROPIC_DEFAULT_OPUS_MODEL` 或 `ANTHROPIC_DEFAULT_SONNET_MODEL` 来获得较新的模型。

在 v2.1.219 之前，`opus` 在 Anthropic API 上从 v2.1.154 开始解析到 Opus 4.8，在 Claude Platform on AWS、Amazon Bedrock 和 Google Cloud 的 Agent Platform 上从 v2.1.207 开始解析到 Opus 4.8。在 v2.1.207 之前，`opus` 在 Claude Platform on AWS 上解析到 Opus 4.7，在 Amazon Bedrock 和 Google Cloud 的 Agent Platform 上解析到 Opus 4.6。

别名指向你的提供商的推荐版本，并随时间更新。要固定到特定版本，请使用完整模型名称，例如 `claude-opus-5`，或设置相应的环境变量，如 `ANTHROPIC_DEFAULT_OPUS_MODEL`。

<Note>
  Opus 5 需要 Claude Code v2.1.219 或更高版本。Sonnet 5 需要 v2.1.197 或更高版本。Opus 4.8 需要 v2.1.154 或更高版本。运行 `claude update` 进行升级。
</Note>

<h3 id="work-with-fable">
  使用 Fable
</h3>

[Claude Fable 5.1](https://platform.claude.com/docs/en/about-claude/models/overview) 和 Claude Fable 5 是 Claude Code 中最强大的模型，适合于比单次会话更大的任务。它们能够维持长时间的自主会话，在行动前进行调查，并比较小的模型更频繁地验证其工作。Fable 5.1 是较新的版本。

这两个 Fable 模型都不是任何计划或提供商上的账户类型默认值。显式选择一个：

* **Fable 5.1**：运行 `/model fable`，或使用 `claude --model fable` 启动。在 [Claude apps gateway](/docs/zh-CN/claude-apps-gateway) 会话中，别名解析到 Fable 5，改为运行 `/model claude-fable-5-1`。
* **Fable 5**：按模型 ID 选择它。在 Anthropic API 上，运行 `/model claude-fable-5` 或使用 `claude --model claude-fable-5` 启动。在其他提供商上，使用你的提供商的 Fable 5 模型 ID 或使用 `ANTHROPIC_DEFAULT_FABLE_MODEL` [固定它](#pin-models-for-third-party-deployments)。

如果你直接连接到 Anthropic API，并且你的用户设置将 `claude-fable-5` 或 `claude-fable-5[1m]` 作为模型，例如因为你在 v2.1.257 之前在 `/model` 选择器中选择了 Fable，Claude Code 会在你第一次运行 v2.1.257 或更高版本时将该保存的值更改为 `fable` 或 `fable[1m]` 别名。启动模型行显示 `(auto-updated)` 一次。项目、本地或托管设置中的 `claude-fable-5` 值保持原样。

Fable 模型的安全分类器标记的请求，最常见于网络安全和生物学领域，会触发[自动模型回退](#automatic-model-fallback)。

要充分利用 Fable：

* **描述结果，而不是步骤**：给它你想要的结果，让它规划路径。要保持它朝着该结果工作，[设置一个目标](/docs/zh-CN/goal)。
* **给它模糊的问题**：根本原因调查、中断调试和架构决策是额外调查和验证发挥作用的地方。
* **跳过验证提醒**：它用更少的提示验证自己的工作，所以测试或检查的提醒通常是不必要的。
* **规划更大的任务**：给它你通常会分成几部分的工作。它能够维持长时间的会话而不失去思路。

<Note>
  Fable 5.1 需要 Claude Code v2.1.257 或更高版本。如果来自较旧版本的请求失败，请参阅 [Claude Code does not support this model](/docs/zh-CN/errors#claude-code-does-not-support-this-model)。Fable 5 需要 v2.1.170 或更高版本。运行 `claude update` 进行升级。有关零数据保留下的可用性，请参阅 [Model availability under ZDR](/docs/zh-CN/zero-data-retention#model-availability-under-zdr)。
</Note>

在 Anthropic API 上，`/model` 选择器仅在服务器报告它对你的组织可用后才列出 Fable 模型。当你输入 `/model fable` 或 Fable 模型 ID 时，Claude Code 直接与服务器检查可用性，所以即使选择器未列出该条目，输入的选择也可以成功。

<h4 id="fable-and-usage-credits">
  Fable 和使用额度
</h4>

根据你的计划和座位等级，Fable 使用可以计入[使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans)，而不是从你的计划的包含限额中扣除。当这样做时，`/model` 选择器在 Fable 行上显示"需要使用额度"。要管理使用额度，请参阅 [Add usage credits to your subscription](/docs/zh-CN/costs#add-usage-credits-to-your-subscription)。

在交互式会话中，Claude Code 在 Fable 请求计入使用额度之前显示同意提示。企业计划的成员（具有组织计费）不会看到该提示。你可以继续使用 Fable 并使用使用额度，或切换到你的默认模型。你也可以关闭提示：

* 在 `/model` 选择器中，你保持当前模型。
* 在会话中途，Claude Code 继续在你的默认模型上进行该轮。

在你选择继续使用 Fable 并使用使用额度后，Claude Code 不会再显示该提示。

在与 [Remote Control](/docs/zh-CN/remote-control) 连接的会话中、[后台会话](/docs/zh-CN/agent-view)中或 [agent team](/docs/zh-CN/agent-teams) 队友的会话中，可能没有人在终端，所以 Claude Code 会将中途同意提示保持到 [`dialogExpiry`](/docs/zh-CN/settings-reference#dialogexpiry) 截止时间，默认为五分钟。如果到截止时间没有人回答，Claude Code 会结束该轮而不发送请求，并在记录中添加通知，Remote Control 客户端也会显示该通知。你的模型选择保持不变，Claude Code 会在你的下一条消息上再次请求同意。

提示等待时你可以做什么取决于会话：

* 在 Remote Control 连接或队友的会话中，在终端按任意键取消截止时间，Claude Code 会等待你的答案。
* 在后台会话中，在截止时间前回答。
* 如果你在任何人在终端输入之前从远程客户端发送新消息，Claude Code 会以相同的方式结束该轮，你的新消息开始下一轮。在有人在终端输入后，Claude Code 继续等待答案并将你的新消息排队在其后面。

在带有 `-p` 标志的[非交互模式](/docs/zh-CN/headless)中以及通过 Agent SDK，Claude Code 永远不会显示同意提示。当 Fable 请求在那里会计入使用额度时，Claude Code 会在不询问的情况下计入它。

<h3 id="setting-your-model">
  设置你的模型
</h3>

你可以通过多种方式配置你的模型，按优先级顺序列出：

1. **在会话期间**：使用 `/model <alias|name>` 立即切换，或运行不带参数的 `/model` 打开选择器。请参阅 [when Claude Code asks you to confirm the switch](/docs/zh-CN/prompt-caching#switching-models)
2. **在启动时**：使用 `claude --model <alias|name>` 启动
3. **环境变量**：设置 `ANTHROPIC_MODEL=<alias|name>`
4. **设置**：使用 `model` 字段在你的设置文件中永久配置
5. **[新会话的默认值](#set-a-default-model-for-new-sessions)**：设置 `ANTHROPIC_DEFAULT_MODEL=<alias|name>`

`/model` 通过在你的用户设置中写入 `model` 字段来保存你的选择作为新会话的默认值。在选择器中：

* `Enter`：切换模型并保存为你的默认值
* `s`：仅为此会话切换模型

直接输入 `/model <name>` 的行为类似于 `Enter`。如果你在[非交互模式](/docs/zh-CN/headless)中使用 `-p` 标志设置带有 `/model` 的模型，你的选择仅适用于当前会话，不会保存为你的默认值；该模式中的 `/model` 需要 Claude Code v2.1.205 或更高版本。项目和托管设置仍然优先，并在下次启动时重新应用。你的管理员配置的[组织默认模型](#organization-default-model)也会在下次启动时重新应用。

在 v2.1.144 到 v2.1.152 中，`/model` 仅适用于当前会话，选择器中的 `d` 保存默认值。

`--model` 标志和 `ANTHROPIC_MODEL` 环境变量仅适用于你使用它们启动的会话。要同时在不同的终端中运行不同的模型，请使用自己的 `--model` 标志启动每个终端，而不是使用 `/model` 切换。

当 Claude Code 与 Anthropic API 通话时，直接或通过代理它的 [LLM 网关](/docs/zh-CN/llm-gateway)，`/model` 选择器中的价格会出现，行上的价格是该行选择的模型的价格。在 [Amazon Bedrock 等第三方提供商](/docs/zh-CN/third-party-integrations)上以及在 [Claude apps gateway](/docs/zh-CN/claude-apps-gateway) 上，你的提供商或网关决定你支付的费用，所以选择器行不显示价格。价格仅是显示标签；它不影响行选择的模型或你的提供商计费的内容。在 v2.1.206 之前，[Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 和网关会话显示 Anthropic 列表价格，一行可能显示与其选择的模型不同的模型的价格。

使用 `claude --resume`、`--continue` 或 `/resume` 选择器启动的恢复会话保持它们保存记录时使用的模型，无论当前 `model` 设置如何。如果恢复的模型已被停用或被 [`availableModels`](#restrict-model-selection) 排除，会话会回退到正常的优先级顺序。这可以防止另一个会话的 `/model` 选择在恢复时改变模型。在使用提供商特定部署 ID 而不是 Anthropic 模型 ID 的提供商上，例如 Amazon Bedrock、Google Cloud 的 Agent Platform 和 Microsoft Foundry，根本不会恢复记录模型，会话通过正常的优先级顺序解析其模型。

你为新启动使用 `--model` 或 `ANTHROPIC_MODEL` 选择的模型仍然优先于恢复的模型。从 v2.1.195 开始，[`ANTHROPIC_DEFAULT_OPUS_MODEL`](#environment-variables) 系列变量也是如此。[`ANTHROPIC_DEFAULT_MODEL`](#set-a-default-model-for-new-sessions) 也可以，在其部分中列出的条件下。

当启动时的活动模型来自项目或托管设置而不是你自己的选择时，启动标题显示哪个设置文件设置了它。运行 `/model` 进行覆盖；项目或托管设置在下次启动时重新应用。在嵌入 Claude Code 并设置 [`CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST`](/docs/zh-CN/env-vars) 的平台上，主机的模型配置优先于托管模型设置，而托管 `availableModels` 允许列表保持有效，除非主机提供自己的；[Exceptions to managed settings precedence](/docs/zh-CN/settings#exceptions-to-managed-settings-precedence) 说明主机覆盖的键和变量。

如果你或你的组织配置 [PreModelSwitch hooks](/docs/zh-CN/hooks#premodelswitch)，它们在请求的切换应用之前运行，可以阻止它或要求你确认。

当 Claude Code 无法判断你的组织的[托管插件](/docs/zh-CN/settings-reference#enabledplugins)提供哪些 PreModelSwitch hooks 时，例如因为托管插件加载失败，它会拒绝切换而不是未检查地应用它，并在每次新尝试时再次检查。请参阅 [Model switch was blocked by a PreModelSwitch hook](/docs/zh-CN/errors#model-switch-was-blocked-by-a-premodelswitch-hook) 了解消息和恢复。

当你通过 [Agent SDK](/docs/zh-CN/agent-sdk/overview) `setModel()` 方法切换模型，或从通过 [Remote Control](/docs/zh-CN/remote-control) 连接的设备切换，或运行 Claude Code CLI 为你切换的应用（如 [Desktop app](/docs/zh-CN/desktop)）时，Claude Code 会检查该字符串是否是它识别的字符串，然后再保存它。此检查需要 Claude Code v2.1.200 或更高版本。检查 Remote Control 选择需要你的机器上的 Claude Code v2.1.260 或更高版本。在 Anthropic API 上，Claude Code 识别：

* 一个模型别名
* `/model` 选择器中的一个条目
* 任何以 `claude-` 开头的名称
* 你自己配置的值作为[自定义模型选项](#add-a-custom-model-option)或在 [`modelOverrides`](#override-model-ids-per-version) 中

Claude Code 使用 `Model "<name>" is not a recognized model id.` 拒绝无法识别的字符串，会话保持其当前模型，而不是保存字符串并在下一个请求时失败。请参阅[错误参考](/docs/zh-CN/errors#model-is-not-a-recognized-model-id)了解恢复步骤。

检查仅在 Anthropic API 上运行。在 Amazon Bedrock、Google Cloud 的 Agent Platform、Microsoft Foundry、[Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 以及在 [LLM 网关](/docs/zh-CN/llm-gateway) 或自定义 `ANTHROPIC_BASE_URL` 后面，你的提供商或网关定义模型名称，所以 Claude Code 不检查地通过任何字符串。检查也不涵盖 `--model` 标志、`ANTHROPIC_MODEL` 环境变量或 `model` 设置；那里的拼写错误值会在第一个请求时产生 [There's an issue with the selected model](/docs/zh-CN/errors#theres-an-issue-with-the-selected-model)。Claude Code 仍然可以在请求时在每个提供商上写入[无法识别的模型诊断行](/docs/zh-CN/errors#unrecognized-model-id-on-a-request)。

当请求的模型有计划的停用日期或自动重新映射到较新版本时，Claude Code 显示一个警告，命名请求的模型。交互式会话将其显示为启动通知。从 v2.1.182 开始，当使用默认文本输出格式在[非交互模式](/docs/zh-CN/headless)中时，相同的警告被写入 stderr。检查也涵盖在[子代理 frontmatter](/docs/zh-CN/sub-agents) 中设置的 `model`。对于 `--output-format json` 和 `stream-json`，stderr 警告被抑制；改为从[结果消息](/docs/zh-CN/headless#get-structured-output)的 `modelUsage` 字段读取实际模型。

例如，在 Opus 上启动会话：

```bash theme={null}
claude --model opus
```

然后从会话内切换模型：

```text theme={null}
/model sonnet
```

示例设置文件：

```json theme={null}
{
    "permissions": {
        "allow": ["Bash(npm run lint)"]
    },
    "model": "opus"
}
```

<h4 id="set-a-default-model-for-new-sessions">
  为新会话设置默认模型
</h4>

设置 `ANTHROPIC_DEFAULT_MODEL=<alias|name>` 来选择你的会话默认启动的模型。需要 Claude Code v2.1.236 或更高版本。

Claude Code 仅在以下都未选择模型时才在变量的模型上启动新会话：

* `--model` 标志
* `ANTHROPIC_MODEL`
* 任何设置文件中的 `model` 值，包括你使用 `/model` 保存的选择
* [组织默认模型](#organization-default-model)

你使用 `/model` 保存的选择在后续启动时也优先于变量。设置 `ANTHROPIC_MODEL` 时，Claude Code 在下次启动时返回到该变量的模型，无论你使用 `/model` 保存了什么。

Claude Code 也将 Default 选项解析为变量的模型，除非应用了组织默认模型。当 Default 选项解析为变量的模型时，`/model` 选择器中的 Default 行显示标签"由 ANTHROPIC\_DEFAULT\_MODEL 设置"。

Claude Code 在这些情况下忽略变量，Default 选项解析如同你未设置它：

* 你将其设置为 `default`、`inherit`、`opusplan` 或 `haiku`
* [`enforceAvailableModels`](#enforce-the-allowlist-for-the-default-model) 已打开
* [`availableModels`](#restrict-model-selection) 或[组织模型限制](#organization-model-restrictions)排除该模型
* 该模型对你的账户不可用

当新会话将在变量的模型上启动时，你使用 `claude --resume`、`--continue` 或 `/resume` 选择器恢复的会话也会在其上启动。Claude Code 不会恢复该会话的记录中保存的模型。否则 Claude Code 在你[恢复会话](#setting-your-model)时不使用该变量。

<h4 id="a-new-session-starts-on-a-different-model-than-you-picked">
  新会话在与你选择的不同的模型上启动
</h4>

当你使用 `/model` 选择模型，而你的下一个会话在其他东西上启动时，这些是常见的原因：

* **你为一个会话选择了它。** 在选择器中按 `s`、使用 `--model` 启动以及在非交互模式中运行 `/model` 都仅适用于当前会话，并保持你保存的默认值不变。
* **优先级更高的东西设置了模型。** 项目或托管设置中的 `model` 值、你的 shell 中的 `ANTHROPIC_MODEL` 或你的管理员设置为覆盖用户选择的[组织默认值](#organization-default-model)在每次启动时都适用。你的 `/model` 选择仍然被保存；它被超越。当项目或托管设置设置模型时，启动标题命名该文件。
* **Claude Code 无法保存你的选择。** `/model` 写入 `~/.claude/settings.json`。如果你无法写入该文件，例如因为另一个工具生成它或将其链接到只读副本，你选择的模型持续该会话，下次启动读取旧值。在生成文件的工具中设置 `model`，或使文件可写。请参阅 [A change you made in Claude Code is lost in new sessions](/docs/zh-CN/settings#a-change-you-made-in-claude-code-is-lost-in-new-sessions)。
* **你恢复了一个会话。** 你使用 `claude --resume` 或 `--continue` 恢复的会话通常[保持它使用的模型](#setting-your-model)而不是你当前的默认值。

<h2 id="restrict-model-selection">
  限制模型选择
</h2>

企业管理员可以在[托管或策略设置](/docs/zh-CN/managed-settings)中使用 `availableModels` 来限制用户可以选择的模型。条目可以匹配模型系列（如 `sonnet`）、版本前缀（如 `claude-sonnet-4-5`）或完整模型 ID（如 `claude-sonnet-4-5-20250929`）。版本前缀也会匹配扩展它的后续模型 ID，因此 `claude-fable-5` 允许 Fable 5 和 Fable 5.1，而 `claude-fable-5-1` 仅允许 Fable 5.1。

在嵌入 Claude Code 并设置 [`CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST`](/docs/zh-CN/env-vars) 的平台上，主机的模型配置优先于托管模型设置，而托管的 `availableModels` 允许列表保持有效，除非主机提供自己的列表；[托管设置优先级的例外](/docs/zh-CN/settings#exceptions-to-managed-settings-precedence)说明了主机覆盖的密钥和变量。

当设置 `availableModels` 时，允许列表适用于用户可以指定模型的所有地方：

* **主会话模型**：`/model`、`--model` 标志、`ANTHROPIC_MODEL` 环境变量、`model` 设置、[`ANTHROPIC_DEFAULT_MODEL`](#set-a-default-model-for-new-sessions) 以及[恢复会话](#setting-your-model)时恢复的模型
* **别名解析**：环境变量 `ANTHROPIC_DEFAULT_OPUS_MODEL`、`ANTHROPIC_DEFAULT_SONNET_MODEL`、`ANTHROPIC_DEFAULT_HAIKU_MODEL` 和 `ANTHROPIC_DEFAULT_FABLE_MODEL` 不能将允许的别名重定向到列表外的模型
* **快速模式**：当 `/fast` 会隐式切换到列表外的 Opus 模型时，它会拒绝切换，并显示消息"不在您组织的允许模型中"
* **子代理和队友模型**：[子代理](/docs/zh-CN/sub-agents#choose-a-model)前置元数据中的 `model` 字段、Agent 工具的 `model` 参数、[代理团队](/docs/zh-CN/agent-teams#specify-teammates-and-models)队友模型、`CLAUDE_CODE_SUBAGENT_MODEL` 以及在 v2.1.197 及更早版本中，`/agents` 向导中的模型选择器&#x20;
* **技能和命令模型**：[技能和命令](/docs/zh-CN/skills)中的 `model` 前置元数据
* **顾问模型**：配置的 [`advisorModel`](/docs/zh-CN/advisor) 设置和 `--advisor` 标志
* **后台代理模型**：在[分派选择器](/docs/zh-CN/agent-view)中选择的模型

在 Anthropic API 和 [AWS 上的 Claude Platform](/docs/zh-CN/claude-platform-on-aws) 上，模型系列别名 `opus`、`sonnet`、`haiku` 或 `fable` 在允许列表允许该模型时解析为其通常的模型。当允许列表阻止该模型时，Claude Code 会替换允许列表允许的该系列的最新版本，并显示一条通知，命名请求的和替换的模型。例如，使用 `["sonnet", "claude-opus-4-6"]`，`/model opus` 和 `--model opus` 都会选择 Claude Opus 4.6，这是允许的最新 Opus。在 v2.1.205 之前，其最新发布版本在列表外的别名被拒绝或替换，就像任何其他被阻止的选择一样，即使列表允许较旧版本。

替换需要一个允许的版本来落地：当允许列表不允许别名系列的任何版本时，别名遵循下面的拒绝和替换行为，就像任何其他被阻止的值一样。

Claude Code 根据模型的设置位置处理任何其他被阻止的选择：

* **`/model`**：Claude Code 拒绝切换并显示错误
* **`--model` 标志、`ANTHROPIC_MODEL` 或 `model` 设置**：Claude Code 在启动时用警告替换该值，命名请求的和替换的模型，会话在默认模型上启动
* **[`ANTHROPIC_DEFAULT_MODEL`](#set-a-default-model-for-new-sessions)**：Claude Code 忽略该变量
* **子代理或队友覆盖**：Claude Code 在后备模型上运行子代理或队友，而不是使请求失败。有关子代理后备，请参阅[选择模型](/docs/zh-CN/sub-agents#choose-a-model)，有关队友后备，请参阅[指定队友和模型](/docs/zh-CN/agent-teams#specify-teammates-and-models)。

  在交互式会话中，当 Claude Code 通过此后备或上面的最新允许版本替换来替换子代理的模型时，它会警告您，命名请求的和替换的模型；它不报告队友的后备。

  在上面的最新允许版本替换操作的地方，被阻止的系列别名遵循它。在 v2.1.222 之前，别名在每个提供商上都像任何其他被阻止的值一样回退
* **技能或命令覆盖**：Claude Code 忽略覆盖，包括被阻止的系列别名，技能或命令在会话模型上运行。[在子代理中运行](/docs/zh-CN/skills#run-skills-in-a-subagent)的技能或命令遵循上面的子代理行为
* **`advisorModel` 设置**：顾问对会话被禁用
* **`--advisor` 标志**：Claude Code 在启动时以错误退出。在[后台会话](/docs/zh-CN/agent-view)中，它在没有顾问的情况下启动会话，而不是退出

Claude Code 从 `/model` 选择器中隐藏排除的模型。列表中没有内置选择器行的完整模型 ID（如列表固定的较旧版本）在 `/model` 选择器中显示为其自己的标记行，除非 Claude Code 用 [`modelPicker`](/docs/zh-CN/settings-reference#modelpicker) 阵容替换内置选项。在 v2.1.199 之前，这样的 ID 只能通过键入 `/model <id>` 来选择。

Claude Code 代表您进行的模型更改以相同的方式进行检查：

* **[后备模型链](#fallback-model-chains)**：允许列表外的条目被删除
* **Plan 模式升级**：在 Anthropic API 和 AWS 上的 Claude Platform 上，升级（如 [`opusplan`](#opusplan-model-setting)）到排除的模型使用升级系列允许的最新版本。在具有提供商特定模型 ID 的提供商上，以及当不允许任何版本时，升级被跳过，规划继续在会话的模型上进行
* **[自动模型后备](#automatic-model-fallback)**：目标被排除的后备不运行，因此标记的请求以拒绝结束
* **[Auto 模式分类器](/docs/zh-CN/permission-modes#eliminate-prompts-with-auto-mode)**：分类器的 Claude Sonnet 5 默认仅在允许列表允许 Sonnet 5 时适用。当它被排除时，分类器在会话的模型上运行，允许列表已经管理该模型，或在会话运行[Fable 模型](#work-with-fable)时在 Opus 模型上运行。在 Anthropic API 以外的提供商上，该 Opus 后备在提供商的默认 Opus 模型上运行，不咨询允许列表。需要 Claude Code v2.1.210 或更高版本
* **[快速模式](/docs/zh-CN/fast-mode)**：当会话之后运行的模型在允许列表外时，启用快速模式被拒绝

```json theme={null}
{
  "availableModels": ["sonnet", "haiku"]
}
```

<h3 id="surface-coverage">
  表面覆盖
</h3>

每个表面都强制执行它接收的允许列表。哪个交付机制到达每个表面不同：

| 交付机制                                                       | CLI 和 IDE | 桌面本地会话 | Web、移动和云会话                                                                                                                                                          | Agent SDK 和非交互式 | Cowork     |
| :--------------------------------------------------------- | :-------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :-------------- | :--------- |
| 来自管理控制台的[服务器管理设置](/docs/zh-CN/server-managed-settings)          | 强制执行      | 强制执行   | 强制执行                                                                                                                                                                | 强制执行            | 未交付        |
| [MDM 或托管设置文件](/docs/zh-CN/managed-settings#delivery-mechanisms) | 强制执行      | 强制执行   | 在 Anthropic 托管环境中未交付；在[自托管环境](/docs/zh-CN/self-hosted-environments)中，根据[Claude Code 如何组合托管源](/docs/zh-CN/managed-settings#how-claude-code-combines-managed-sources)从运行器镜像强制执行 | 强制执行            | 在部署的地方强制执行 |

* 云会话在[Web 上的 Claude Code](/docs/zh-CN/claude-code-on-the-web) 或桌面应用中默认在 Anthropic 管理的 VM 上运行：部署到您的设备的设置不会到达它们，因此通过服务器管理设置交付允许列表。您的组织路由到[自托管环境](/docs/zh-CN/self-hosted-environments)的会话在您自己的计算上运行，也读取运行器镜像中的托管设置文件。[Claude Code 如何组合托管源](/docs/zh-CN/managed-settings#how-claude-code-combines-managed-sources)说明了该文件何时适用。云会话中的中途模型切换在请求的模型被允许列表排除时被拒绝。会话创建时的服务器端拒绝适用于[组织模型限制](#organization-model-restrictions)，而不是 `availableModels` 设置密钥。
* Cowork 是 Claude 桌面应用中的代理工作选项卡，在 Claude Code 上运行其会话，但根据设计，不从 claude.ai 管理控制台接收服务器管理设置。托管设置文件在会话运行的地方存在时适用于 Cowork 会话；远程 Cowork 会话在 Anthropic 管理的 VM 上运行，其中不存在设备部署的文件。
* [第三方提供商](/docs/zh-CN/server-managed-settings#platform-availability)上的会话，如 Amazon Bedrock、Google Cloud 的 Agent Platform、Microsoft Foundry 和 [AWS 上的 Claude Platform](/docs/zh-CN/claude-platform-on-aws)，不接收服务器管理设置，因此在那里通过 MDM 或托管设置文件交付允许列表。
* 服务器管理交付还需要会话使用[符合条件的登录或密钥](/docs/zh-CN/server-managed-settings#platform-availability)进行身份验证。仅通过 [`apiKeyHelper`](/docs/zh-CN/settings-reference#apikeyhelper) 脚本生成密钥的舰队应通过 MDM 或托管设置文件交付允许列表。
* 桌面 Code 选项卡也托管[SSH 会话](/docs/zh-CN/desktop#ssh-sessions)，它们从运行的远程主机读取托管设置文件。请参阅[桌面托管设置](/docs/zh-CN/desktop#managed-settings)。
* claude.ai 和桌面应用中的模型选择器隐藏或灰显您的组织的允许列表排除的模型。选择器状态是用户的便利；强制执行发生在会话中。

<h3 id="default-model-behavior">
  默认模型行为
</h3>

单独来说，`availableModels` 将默认选项保留在系统的[运行时默认](#default-model-setting)上，用于帐户，直到您也设置 [`enforceAvailableModels`](#enforce-the-allowlist-for-the-default-model)。如果该默认值是您打算限制的模型，也设置 `enforceAvailableModels`。

空的 `availableModels` 数组永远不会启用默认模型强制执行：使用 `availableModels: []`，命名的模型选择被阻止，但帐户类型的默认模型无论 `enforceAvailableModels` 如何都保持可用。

<h3 id="enforce-the-allowlist-for-the-default-model">
  为默认模型强制执行允许列表
</h3>

在托管设置中将 `enforceAvailableModels: true` 与非空的 `availableModels` 一起设置，以将允许列表扩展到默认选项。这需要 Claude Code v2.1.175 或更高版本。

```json theme={null}
{
  "availableModels": ["sonnet", "haiku"],
  "enforceAvailableModels": true
}
```

默认选项解析为帐户类型默认值，或当管理员设置了一个时解析为[组织默认模型](#organization-default-model)。当该模型不在允许列表中时，默认选项改为解析为命名允许的、可用模型的第一个 `availableModels` 条目，`/model` 选择器的默认行显示该模型。这适用于到达默认值的所有地方：会话启动、在 `/model` 中选择默认值、[后备模型链](#fallback-model-chains)中的 `"default"` 关键字以及排除选择被删除时使用的后备。

`enforceAvailableModels` 仅在 `availableModels` 非空时重新映射默认选项。使用 `availableModels: []`，帐户类型的默认模型保持可用，因此设置不能将用户锁定在每个模型之外。当 `availableModels` 非空但没有条目解析为允许的和可用的模型时，强制执行被跳过，默认解析为帐户类型默认值，仅在 `--debug` 下可见警告。在列表中保留至少一个保证可用的条目以避免这种情况。

在您交付的最高排名托管源中部署两个密钥。默认情况下，Claude Code 仅读取该源，因此放在托管设置文件中的对在管理控制台交付任何设置时被忽略；在[Claude Code 如何组合托管源](/docs/zh-CN/managed-settings#how-claude-code-combines-managed-sources)中的选择加入合并下，Claude Code 仍然忽略来自排名低于设置 `availableModels` 的源的 `modelOverrides` 映射。

<h3 id="control-the-model-users-run-on">
  控制用户运行的模型
</h3>

`model` 设置是初始选择，不是强制执行。它设置会话启动时哪个模型处于活动状态，但用户仍然可以打开 `/model` 并选择默认值，无论 `model` 设置为什么，默认值都解析为系统的[运行时默认](#default-model-setting)，除非 [`enforceAvailableModels`](#enforce-the-allowlist-for-the-default-model) 重定向它。

要完全控制模型体验，请组合这些设置：

* **`availableModels`**：限制用户可以切换到的命名模型
* **`enforceAvailableModels`**：将 `availableModels` 允许列表扩展到默认选项，因此默认值不能解析到列表外的模型
* **`model`**：设置会话启动时的初始模型选择
* **`ANTHROPIC_DEFAULT_SONNET_MODEL`** / **`ANTHROPIC_DEFAULT_OPUS_MODEL`** / **`ANTHROPIC_DEFAULT_HAIKU_MODEL`** / **`ANTHROPIC_DEFAULT_FABLE_MODEL`**：控制 `sonnet`、`opus`、`haiku` 和 `fable` 别名解析为什么，以及[帐户类型默认](#default-model-setting)使用哪个版本

此示例在 Sonnet 4.5 上启动用户，将选择器限制为 Sonnet 和 Haiku，并确保默认值解析为允许列表上的模型，而不是层级默认值：

```json theme={null}
{
  "model": "claude-sonnet-4-5",
  "availableModels": ["claude-sonnet-4-5", "haiku"],
  "enforceAvailableModels": true,
  "env": {
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-5"
  }
}
```

没有 `enforceAvailableModels` 或 `env` 块，在选择器中选择默认值的用户会获得[运行时默认](#default-model-setting)，而不是在 `model` 中固定的版本。这两个设置覆盖不同的范围：`enforceAvailableModels` 使默认值遵守允许列表，而 `env` 块固定允许的别名（如 `sonnet`）解析为哪个版本。当限制模型系列足够时单独使用 `enforceAvailableModels`；当您还需要固定特定版本时添加 `env` 块。

<h3 id="merge-behavior">
  合并行为
</h3>

当 Claude Code 应用的托管设置定义 `availableModels` 时，该列表单独适用，除了[提供自己的主机平台](/docs/zh-CN/settings#exceptions-to-managed-settings-precedence)：用户、项目或本地设置中的条目不能扩展它，Claude Code 也永远不会跨托管源合并 `availableModels`；[Claude Code 如何组合托管源](/docs/zh-CN/managed-settings#how-claude-code-combines-managed-sources)说明了哪个源的列表适用。否则，来自用户、项目和本地设置的列表像其他数组设置一样[连接和去重](/docs/zh-CN/settings#settings-precedence)。在 Claude Code v2.1.175 之前，来自较低优先级范围的条目合并到托管列表中，而不是被它替换。

在有效列表中，命名系列中特定模型的条目，无论是版本前缀还是完整模型 ID，都禁用该系列的通配符条目：`["sonnet", "claude-sonnet-4-5"]` 仅允许 Sonnet 4.5 版本，而不是每个 Sonnet 模型。

<h3 id="mantle-model-ids">
  Mantle 模型 ID
</h3>

当启用[Amazon Bedrock Mantle 端点](/docs/zh-CN/amazon-bedrock#use-the-mantle-endpoint)时，`availableModels` 中以 `anthropic.` 开头的条目被添加到 `/model` 选择器作为自定义选项，并路由到 Mantle 端点。这是对[为第三方部署固定模型](#pin-models-for-third-party-deployments)中描述的别名匹配的例外。该设置仍然将选择器限制为列出的条目，Mantle ID 嵌入系列名称，因此它计为特定条目并禁用该系列的通配符：在任何 Mantle ID 旁边，列出您想保持可选择的版本前缀或完整 ID。请参阅[合并行为](#merge-behavior)。

<h3 id="organization-model-restrictions">
  组织模型限制
</h3>

Claude Enterprise 计划上的组织管理员通过在 claude.ai 管理控制台中禁用单个模型来限制成员可以运行的模型。此限制在 Claude Code 进行身份验证时与帐户的权利一起交付，与设置中的任何 `availableModels` 列表分开，服务器在创建会话时独立强制执行相同的限制。需要 Claude Code v2.1.187 或更高版本。

当成员登录或使用自己的 API 密钥时，限制适用。组织范围的凭证，如组织服务密钥，不与用户绑定，因此限制不适用于它们。

Claude Console 没有模型限制控制。没有 Claude Enterprise 计划的组织，包括其成员通过 Anthropic API 进行身份验证的组织，使用[托管设置](/docs/zh-CN/managed-settings)中的 [`availableModels`](#restrict-model-selection) 限制模型，添加 [`enforceAvailableModels`](#enforce-the-allowlist-for-the-default-model) 以覆盖默认选项。这些设置由 Claude Code 本身强制执行，而不是由服务器强制执行。

受限模型从 `/model` 选择器中隐藏。使用 `--model`、`ANTHROPIC_MODEL` 环境变量或 `model` 设置按名称选择它显示通知 `Model "<name>" is restricted by your organization's settings. Using <model> instead.` 并且会话在允许的模型上启动。为受限模型键入 `/model <name>` 被拒绝，显示 `Model '<name>' is restricted by your organization's settings. Run /model to choose a different model.` 并且会话保持其当前模型。

[模型系列别名](#restrict-model-selection)（如 `opus`）在组织允许时解析为其通常的模型。当组织限制该模型时，Claude Code 替换组织允许的该系列的最新版本，具有相同的替换通知。`/model <alias>` 仅在其系列的每个版本都被限制时被拒绝；使用 `--model`、`ANTHROPIC_MODEL` 或 `model` 设置的别名在这种情况下仍在启动时被替换。在 v2.1.205 之前，系列别名基于其最新发布版本单独被替换或拒绝，即使允许较旧版本。

限制适用于组织范围或按角色：

* 在组织级别禁用模型会为每个成员删除它。
* 角色级别访问向不同的自定义角色授予不同的模型，持有多个角色的成员可以使用其任何角色授予的模型。
* Haiku 模型始终可用，无法禁用，因此每个成员至少保留一个可用模型。
* 访问更改在大约一分钟内对新请求生效；`/model` 选择器在下次会话启动时反映它。

两个限制一起适用：仅当模型被 `availableModels` 允许且不被组织限制时，模型才可选择。组织限制仅到达 Anthropic API 和 [LLM 网关](/docs/zh-CN/llm-gateway)部署上的会话；在任何其他提供商上，改用 `availableModels`。

<h2 id="organization-default-model">
  组织默认模型
</h2>

Claude Enterprise 计划上的组织管理员可以从 claude.ai 管理控制台为 Claude Code 成员设置默认模型，可以为整个组织设置，也可以按自定义角色设置。设置后，"默认"选项将解析为该模型。需要 Claude Code v2.1.196 或更高版本。

`/model` 选择器中的"默认"行显示组织默认值的名称，并带有"组织默认"标签。无论管理员是为整个组织设置默认值还是为您的角色设置默认值，标签都显示"组织默认"。角色默认值适用于该自定义角色的成员，优先于组织范围的默认值；当您的多个角色设置不同的默认值时，应用最强大的模型。

组织默认值是一个起点，而不是限制。这些选择优先于它：

* `--model` 标志和 `ANTHROPIC_MODEL` 环境变量
* [托管设置](/docs/zh-CN/managed-settings)中的 `model` 值或通过 `--settings` 提供的值
* 您的用户、项目或本地设置中的 `model` 值，包括您使用 `/model` 保存的模型

管理员还可以配置组织默认值以覆盖用户选择。启用覆盖后，它优先于用户、项目和本地设置中的 `model` 值，因此您使用 `/model` 保存的模型在当前会话中应用，组织默认值在下次启动时返回。当您的选择不同时，`/model` 显示 `您的组织的默认值（<model>）在重启时应用`。即使启用了覆盖，`--model` 标志、`ANTHROPIC_MODEL`、托管设置和 `--settings` 仍然优先。覆盖功能仅对有限的组织可用；请咨询您的 Anthropic 账户团队了解可用性。

要限制成员可以选择的模型，请改用[组织模型限制](#organization-model-restrictions)或 [`availableModels`](#restrict-model-selection)。

Claude Code 在启动时读取组织默认值一次，因此管理员在会话中期更改的默认值在下次启动时生效。

当组织默认值不覆盖用户选择时，管理员更改后的第一次交互式启动会从您的用户设置中清除 `model` 键一次，以便应用新的默认值。它不会更改文件中的任何其他内容，您在该启动后使用 `/model` 保存的模型会被保留。

组织默认值在被采用之前会通过这些限制检查：

* [`availableModels`](#restrict-model-selection) 本身不适用于组织默认值，因此允许列表外的组织默认值仍然适用。当同时设置了 [`enforceAvailableModels`](#enforce-the-allowlist-for-the-default-model) 时，允许列表外的组织默认值会被重新映射到第一个允许列表条目，就像任何其他默认值一样
* [组织模型限制](#organization-model-restrictions)拒绝的组织默认值会被替换为其系列中最新的允许模型，或当该系列的每个版本都被限制时被替换为成本较低的系列
* 您的账户完全无法使用的组织默认值会被跳过，"默认"选项的解析方式与[没有组织默认值](#default-model-setting)时相同

从 v2.1.199 开始，当组织默认值是与您的账户类型通常默认值不同的模型系列时，`/model` 选择器会为该通常系列保留单独的行，以便您仍然可以为会话切换到它。在 v2.1.196 到 v2.1.198 中，该行在选择器中缺失。

组织默认值仅适用于使用 Anthropic API 进行身份验证的会话。要在其他任何地方设置默认值，包括 [LLM gateway](/docs/zh-CN/llm-gateway) 部署，请改用[托管设置](/docs/zh-CN/managed-settings)中的 `model` 键。

<h2 id="organization-effort-limits">
  组织工作量限制
</h2>

您的组织可以通过两种方式限制[工作量级别](#adjust-effort-level)。在 Claude Enterprise 计划中，组织管理员设置按角色的工作量限制，如下所述。在任何计划和任何提供商上，包括 Amazon Bedrock、Google Cloud 的 Agent Platform 和 Microsoft Foundry，[`maxEffortLevel`](/docs/zh-CN/settings-reference#maxeffortlevel) 托管设置在客户端上限制工作量。当两者都适用于某个模型时，较低的限制适用。

Claude Enterprise 计划上的组织管理员可以为每个自定义角色按模型设置最大[工作量级别](#adjust-effort-level)，以及角色级别的[组织模型限制](#organization-model-restrictions)。超过限制的级别不会在 `/effort` 选择器中提供，使用 `--effort` 或 `/effort` 命名更高级别会以限制级别运行。在交互式会话和纯文本 `--print` 运行中，警告会命名请求的和应用的级别；使用 `json` 或 `stream-json` 输出或在后台代理中，限制会静默应用。限制是按模型的，因此切换模型可以改变哪些级别可用。当您的多个角色授予相同的模型时，最不严格的限制适用。需要 Claude Code v2.1.195 或更高版本。

工作量限制与[组织模型限制](#organization-model-restrictions)一起交付，并到达相同的会话。

<h2 id="special-model-behavior">
  特殊模型行为
</h2>

<h3 id="default-model-setting">
  `default` 模型设置
</h3>

`default` 的行为取决于您的账户类型：

* **Max、Team Premium、Enterprise 和 Anthropic API**：默认为 Opus 5
* **Claude Platform on AWS、Amazon Bedrock 和 Google Cloud's Agent Platform**：默认为 Opus 5
* **Pro 和 Team Standard**：默认为 Sonnet 5
* **Microsoft Foundry**：默认为 Sonnet 4.5

在 v2.1.219 之前，`default` 在 Anthropic API 上解析为 Opus 4.8，在 Max、Team Premium 和 Enterprise 按量付费上从 v2.1.154 开始解析为 Opus 4.8，在 Claude Platform on AWS、Amazon Bedrock 和 Google Cloud's Agent Platform 上从 v2.1.207 开始解析为 Opus 4.8。在 v2.1.207 之前，`default` 在 Claude Platform on AWS 上解析为 Opus 4.7，在 Amazon Bedrock 和 Google Cloud's Agent Platform 上解析为 Sonnet 4.5。

当管理员设置了[组织默认模型](#organization-default-model)时，`default` 会解析为该模型，而不是上面的账户类型默认值。需要 Claude Code v2.1.196 或更高版本。`default` 也可以解析为您使用 [`ANTHROPIC_DEFAULT_MODEL`](#set-a-default-model-for-new-sessions) 设置的模型，具体条件见其部分说明。

当托管设置[对默认模型强制执行允许列表](#enforce-the-allowlist-for-the-default-model)且账户类型默认值不在 `availableModels` 中时，`default` 会解析为强制执行的默认值，而不是上面的账户类型默认值。当两者都适用时，组织默认值首先替换账户类型默认值，然后强制执行应用于它：允许列表中的组织默认值被保留，而列表外的值解析为强制执行的默认值。

Fable 模型在任何计划或提供商上都不是账户类型默认值。使用 `/model` 选择一个会将其保存为用户设置中的选定模型，以便后续会话从它开始。关于 Claude Code v2.1.257 对保存的 Fable 5 选择所做的一次性更改，请参阅[使用 Fable](#work-with-fable)。

<h3 id="opusplan-model-setting">
  `opusplan` 模型设置
</h3>

`opusplan` 模型别名提供了一种自动化混合方法：

* **在计划模式下**：使用 `opus` 进行复杂推理和架构决策
* **在执行模式下**：自动切换到 `sonnet` 进行代码生成和实现

这将 Opus 的推理能力与 Sonnet 的执行效率相结合。

计划模式 Opus 阶段使用与 `opus` 模型设置相同的上下文窗口。在 Opus [自动升级到 1M 上下文](#extended-context)的订阅层上，`opusplan` 在计划模式下也会获得升级。要在您不在自动升级层上时为两个阶段强制使用 1M 上下文，[设置模型](#setting-your-model)为 `opusplan[1m]`，例如使用 `/model opusplan[1m]`。使用 `/model` 设置它需要 Claude Code v2.1.265 或更高版本；在早期版本上，使用 `--model` 标志或 `model` 设置。

当 [`availableModels`](#restrict-model-selection) 排除最新的 Opus 但允许较旧版本时，例如 `["sonnet", "claude-opus-4-6"]`，`opusplan` 使用最新的允许的 Opus 进行规划，仅当每个 Opus 都被排除时才保持在 Sonnet 上。在计划模式下通常会升级到 Sonnet 的 Haiku 会话同样使用最新的允许的 Sonnet，仅当每个 Sonnet 都被排除时才保持在 Haiku 上。在 v2.1.205 之前，当升级系列的最新版本被排除时，计划模式会保持在会话的模型上，即使允许列表允许较旧的版本。

较旧的允许版本的替换适用于 Anthropic API 和 [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws)。在 Amazon Bedrock、Google Cloud's Agent Platform、Microsoft Foundry 和 Mantle 上，其部署使用提供商特定的模型 ID，当升级模型被排除时，计划模式会保持在会话的模型上。

关于 Claude 在任务中途决定何时咨询第二个模型而不是在计划边界处切换的混合方法，请参阅[顾问工具](/docs/zh-CN/advisor)。

<h3 id="fallback-model-chains">
  回退模型链
</h3>

当主模型过载、不可用或返回另一个不可重试的服务器错误时，Claude Code 可以切换到回退模型，而不是使请求失败。身份验证、计费、速率限制、请求大小和传输错误，以及[您组织的策略检查拒绝](/docs/zh-CN/errors#automatic-retries)，永远不会触发切换；这些遵循其正常的重试和错误处理。

配置一个或多个回退模型，Claude Code 会按顺序尝试它们，在切换时显示通知。切换仅持续当前轮次，因此您的下一条消息会首先再次尝试主模型。Claude Code 在删除重复项后将链限制为三个模型，并忽略额外条目。

使用 `--fallback-model` 标志为一个会话设置链，该标志接受逗号分隔的列表：

```bash theme={null}
claude --fallback-model sonnet,haiku
```

要在会话间保持链，在[设置](/docs/zh-CN/settings)中将 `fallbackModel` 设置为数组：

```json theme={null}
{
  "fallbackModel": ["claude-sonnet-5", "claude-haiku-4-5"]
}
```

`--fallback-model` 标志优先于 `fallbackModel` 设置。每个条目接受模型名称或别名，`"default"` 扩展为默认模型。

Claude Code 在启动时不确认链，`/status` 也不显示它。切换发生时显示的通知是回退已配置的第一个可见迹象。

当请求失败转移时，Claude Code 会按顺序尝试每个条目，直到一个接受它。无法到达的条目，例如在设置中固定的已停用模型，会以相同方式失败转移到下一个。Claude Code 在该遍历开始前删除两种条目：

* **超出允许列表**：当 Claude Code 读取链时，会删除 [`availableModels`](#restrict-model-selection) 不允许的任何条目。
* **压缩期间上下文窗口较小**：链也涵盖[压缩](/docs/zh-CN/context-window#what-survives-compaction)，但 Claude Code 不会回退到上下文窗口小于主模型的模型，因为在那里进行摘要会首先切断部分对话。如果每个回退都较小，压缩会显示原始错误，您可以重试。

Claude Code 也将链应用于[子代理](/docs/zh-CN/sub-agents)。当子代理的请求失败转移时，Claude Code 会按顺序尝试您配置的回退模型，子代理继续在接受请求的模型上运行。您的会话模型保持不变。在 v2.1.247 之前，链涵盖的失败会结束子代理。

<h3 id="automatic-model-fallback">
  自动模型回退
</h3>

本部分涵盖来自 Fable 模型和 Opus 5 的基于内容的回退。关于模型过载或不可用时的基于可用性的回退，请参阅[回退模型链](#fallback-model-chains)。

Fable 模型和 Opus 5 运行安全分类器，最常标记网络安全和生物学内容。当分类器标记请求且标记的类别有回退模型时，Claude Code 在该模型上重新运行请求并在记录中显示通知。对于这两个类别，回退模型取决于哪个模型拒绝：

* **Fable 5.1 和 Fable 5**：生物学标记的请求在 Opus 5 上重新运行，网络安全标记的请求在 Opus 4.8 上重新运行。
* **Opus 5**：网络安全标记的请求在 Opus 4.8 上重新运行。生物学标记的请求以拒绝结束，因为 Opus 5 运行自己的生物学分类器，没有回退模型。

在 Amazon Bedrock、Google Cloud's Agent Platform 和 Microsoft Foundry 上，Claude Code 通过您的部署解析这些目标，如果您设置了 `ANTHROPIC_DEFAULT_OPUS_MODEL`，具有回退的类别会在固定模型上重新运行；请参阅[在 Bedrock、Agent Platform 和 Foundry 上启用回退](#enable-fallback-on-bedrock-agent-platform-and-foundry)。

回退后，会话继续在回退模型上。要返回到您的原始模型，运行 [`/model`](#setting-your-model)。

基于类别的回退需要 Claude Code v2.1.219 或更高版本。在 v2.1.219 之前，每个标记的 Fable 5 请求都在您提供商的默认 Opus 模型上重新运行，Opus 5 不是回退源。

回退模型针对 [`availableModels`](#restrict-model-selection) 进行检查。当它被阻止时，不会发生回退。拒绝显示为正常错误，会话的模型保持不变。

<h4 id="check-what-triggered-fallback">
  检查触发回退的原因
</h4>

回退可以在会话的第一个请求上触发，在您发送任何不寻常的内容之前，因为第一个请求携带工作区上下文，例如您的 CLAUDE.md 内容和 git 状态。包含安全或生物学材料的存储库可以仅在该上下文上触发分类器。

要检查自定义是否是触发器，使用 `claude --safe-mode` 启动会话，这会禁用自定义，例如 CLAUDE.md、skills、MCP 服务器和 hooks。Git 状态和目录名称不是自定义，仍然包括在内。

<h4 id="ask-before-switching">
  切换前询问
</h4>

要决定每次请求被标记时发生什么，而不是自动切换，运行 `/config` 并关闭**当消息被标记时切换模型**，或在您的设置文件中将 [`switchModelsOnFlag`](/docs/zh-CN/settings-reference#switchmodelsonflag) 设置为 `false`。标记的请求然后暂停会话，有两个选项：切换到回退模型，或编辑提示并在当前模型上重试。

某些情况的行为不同：

* 当标记的类别没有回退模型时，例如 Opus 5 上的生物学标记，Claude Code 不显示提示，请求以拒绝结束。
* 如果两个模型都标记相同的请求，您可以编辑提示并重试，或启动新会话。
* 在移动[网络上的 Claude Code](/docs/zh-CN/claude-code-on-the-web) 会话上，不支持编辑和重试。切换模型，或从桌面浏览器或桌面应用继续会话。
* 在[非交互模式](/docs/zh-CN/cli-reference#cli-flags)和无法显示提示的 SDK 集成中，标记的请求以拒绝结束轮次。
* 当回退目标被 [`availableModels`](#restrict-model-selection) 阻止时，Claude Code 不显示提示。标记的请求以拒绝结束，与目标被阻止时的自动回退相同。

<h4 id="enable-fallback-on-bedrock-agent-platform-and-foundry">
  在 Bedrock、Agent Platform 和 Foundry 上启用回退
</h4>

在 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock)、[Google Cloud's Agent Platform](/docs/zh-CN/google-vertex-ai) 和 [Microsoft Foundry](/docs/zh-CN/microsoft-foundry) 上，模型 ID 是提供商特定的，因此自动回退仅在 Claude Code 可以识别两个涉及的模型时运行：

* Claude Code 必须将当前模型识别为回退源。当模型 ID 包含 `claude-fable-5`、匹配 `ANTHROPIC_DEFAULT_FABLE_MODEL` 的值或使用 [`modelOverrides`](#override-model-ids-per-version) 映射时，Fable 5.1 和 Fable 5 被识别。Opus 5 通过其提供商模型 ID 或 [`modelOverrides`](#override-model-ids-per-version) 映射被识别。
* 回退模型必须在您的部署中解析。如果您设置了 `ANTHROPIC_DEFAULT_OPUS_MODEL`，标记的请求会在该模型上为每个具有回退的类别重新运行；Opus 5 上的生物学标记仍以拒绝结束。如果您没有设置它，网络安全标记的请求会在提供商模型列表中的 Opus 4.8 条目上重新运行，来自 Fable 模型的生物学标记请求会在 Opus 5 条目上重新运行。

如果任一模型无法识别，Claude Code 不会自动切换。标记的请求以拒绝消息结束，您可以使用 [`/model`](#setting-your-model) 切换模型并重试。将 `ANTHROPIC_DEFAULT_FABLE_MODEL` 设置为您的 Fable 模型 ID 可启用 Fable 识别。将 `ANTHROPIC_DEFAULT_OPUS_MODEL` 设置为 Opus 模型 ID 为标记的类别提供回退目标，除非固定值命名 Opus 系列外的模型或拒绝的模型；然后 Claude Code 不会切换，拒绝成立。

<h4 id="security-research-and-biology-workloads">
  安全研究和生物学工作负载
</h4>

进攻性安全或生物学中的工作负载，包括渗透测试、Capture the Flag (CTF) 练习和生物学相邻代码库，经常触发回退，通常在第一个请求上。对于 Fable 5.1 或 Fable 5 上的实质性生物学工作，Claude Code 在第一个标记的请求处将会话移动到 Opus 5，后来的生物学标记请求在那里以拒绝结束，因为 Opus 5 没有生物学回退。在 Opus 5 上，您从第一个标记的请求获得这些拒绝。

这是这些域的预期路由，不是账户标记。如果您的组织需要 Fable 级别的能力来完成这项工作，请向您的 Anthropic 账户团队询问受信任的访问计划。

<h3 id="adjust-effort-level">
  调整努力级别
</h3>

[努力级别](https://platform.claude.com/docs/en/build-with-claude/effort)控制自适应推理，让模型根据任务复杂性决定是否以及在每一步上思考多少。较低的努力对于直接的任务更快且更便宜，而较高的努力为复杂问题提供更深入的推理。

可用的努力级别取决于模型。此处未列出的模型不支持努力：

| 模型                                  | 级别                                  |
| :---------------------------------- | :---------------------------------- |
| Fable 5.1 和 Fable 5                 | `low`、`medium`、`high`、`xhigh`、`max` |
| Opus 5、Sonnet 5、Opus 4.8 和 Opus 4.7 | `low`、`medium`、`high`、`xhigh`、`max` |
| Opus 4.6 和 Sonnet 4.6               | `low`、`medium`、`high`、`max`         |

如果您设置活动模型不支持的级别，Claude Code 会回退到该模型支持的最高级别或以下。例如，`xhigh` 在 Opus 4.6 上运行为 `high`。您的组织或您自己的设置也可以限制模型提供的级别；请参阅[组织努力限制](#organization-effort-limits)。

关闭 [`ultracode`](/docs/zh-CN/settings-reference#ultracode) 设置时，Claude Code 按此顺序解析会话的努力级别，采用首先适用的：

1. 明确选择：[`CLAUDE_CODE_EFFORT_LEVEL`](/docs/zh-CN/env-vars#variables) 环境变量、使用 `--effort` 启动或会话中的 `/effort`（[非交互式 `/effort` 的效果更窄](#non-interactive-effort)）
2. 模型的默认努力，在 Fable 5、Opus 4.8 或 Opus 4.7 上：从您第一次运行这些模型之一开始，Claude Code 在会话间保持该模型的默认努力，即使您的设置解析不同的级别。Opus 5 和 Fable 5.1 没有这样的保持。您设置的级别是否结束保持取决于您如何设置它，例如：
   * **结束保持**：交互式确认级别，在 `/effort` 滑块或 `/model` 选择器中使用 `Enter` 或在 `/effort` 后键入的级别，或从连接设备的[远程控制](/docs/zh-CN/remote-control#what-connected-devices-see)努力控制中选择级别
   * **为后续会话保留保持**：启动时的 `--effort`，或在 `/effort` 滑块或 `/model` 选择器中的 `s`
3. 您的设置：您为模型保存的级别或 [`effortLevel`](/docs/zh-CN/settings-reference#effortlevel) 键，在 [`modelSettings`](/docs/zh-CN/settings-reference#modelsettings) 中说明它们之间和跨设置文件的优先级
4. 模型的默认努力：在支持努力的每个模型上为 `high`，除了 Opus 4.7 默认为 `xhigh`，当您的组织为其[组织默认模型](#organization-default-model)设置默认努力级别时，当您运行该模型时该级别是默认值

当您在机器上的交互式会话中设置 `low`、`medium`、`high` 或 `xhigh` 时，您通过如何确认它来选择它持续多长时间：

* 在 `/effort` 滑块或 `/model` 选择器中使用 `Enter`，或在 `/effort` 后键入的级别：将级别保存为您的默认值并在后续会话中应用它
* 在 `/effort` 滑块或 `/model` 选择器中的 `s`：仅将级别应用于此会话。需要 Claude Code v2.1.257 或更高版本

Claude Code 在用户设置中的 [`modelSettings`](/docs/zh-CN/settings-reference#modelsettings) 键下按模型保存级别，因此每个模型保持其自己的保存级别。

`max` 是最深的推理级别。除非您通过 `CLAUDE_CODE_EFFORT_LEVEL` 环境变量设置它，Claude Code 仅将 `max` 应用于当前会话。

<Note>
  您从通过[远程控制](/docs/zh-CN/remote-control#what-connected-devices-see)连接的手机或浏览器上的努力控制中选择的级别仅适用于该会话。
</Note>

<span id="non-interactive-effort" />

当您在 [`-p` 运行](/docs/zh-CN/headless)中使用 `/effort` 设置级别时，Claude Code 仅将其应用于该会话，不将其保存为您的默认值。在 Fable 5、Opus 4.8 和 Opus 4.7 上，该级别也既不结束模型默认努力的保持，也不为会话覆盖它。当该保持有效时，非交互式 `/effort` 报告 `Not applied`，因此改为在启动时传递 `--effort`。

`/effort` 菜单也提供 `ultracode`。Ultracode 是 Claude Code 设置而不是模型努力级别：它向模型发送 `xhigh`，并另外让 Claude 为实质性任务编排[动态工作流](/docs/zh-CN/workflows)。关于它可以在哪里持久设置，请参阅 [`ultracode`](/docs/zh-CN/settings-reference#ultracode) 设置。

您可以通过以下任何方式打开 ultracode：

* **`/effort`**：运行 `/effort ultracode`，或从菜单中选择它
* **`--effort` 标志**：使用 `claude --effort ultracode` 启动，这会在 `xhigh` 努力和 ultracode 打开的情况下启动会话
* **`ultracode` 设置**：在设置文件中、使用 `--settings` 或在 Agent SDK 控制请求中设置 [`"ultracode": true`](/docs/zh-CN/settings-reference#ultracode)。[`applyFlagSettings()`](/docs/zh-CN/agent-sdk/typescript#applyflagsettings) 请求也接受 `effortLevel: "ultracode"`
* **`/model` 选择器**：在选择模型时使用箭头键将努力滑块移动到 `ultracode`。Claude Code 为当前会话打开它，即使您将该模型保存为默认值

将 `ultracode` 传递给 `--effort` 标志或 Agent SDK `effortLevel` 值需要 Claude Code v2.1.203 或更高版本。在 v2.1.203 之前，`--effort ultracode` 打印 `Unknown --effort value 'ultracode'`，会话以默认努力开始。

持久化的 `effortLevel` 设置和 `CLAUDE_CODE_EFFORT_LEVEL` 环境变量不接受 `ultracode`。当 `CLAUDE_CODE_EFFORT_LEVEL` 设置为 `xhigh` 以外的级别时，请求以该级别运行，ultracode 的工作流编排保持不活跃。选择 ultracode 然后显示警告，环境变量覆盖会话的努力。

当 ultracode 不可用时，例如当[工作流被关闭](/docs/zh-CN/workflows#turn-workflows-off)时，`--effort ultracode` 仅设置 `xhigh` 努力。

<h4 id="choose-an-effort-level">
  选择努力级别
</h4>

每个级别在令牌支出和能力之间进行权衡。默认值适合大多数编码任务；当您想要不同的平衡时进行调整。

| 级别          | 何时使用                                                                  |
| :---------- | :-------------------------------------------------------------------- |
| `low`       | 保留用于短的、范围有限的、延迟敏感的、不是智能敏感的任务                                          |
| `medium`    | 减少成本敏感工作的令牌使用，可以权衡一些智能                                                |
| `high`      | 平衡令牌使用和智能。除 Opus 4.7 外，每个模型上的默认值                                      |
| `xhigh`     | 更高令牌支出的更深推理。Opus 4.7 上的默认值                                            |
| `max`       | 可以改进要求任务的性能，但可能显示收益递减，容易过度思考。在广泛采用前测试                                 |
| `ultracode` | 一个 Claude Code 设置，为每个实质性任务规划[动态工作流](/docs/zh-CN/workflows)，每条消息 `xhigh` 推理 |

努力规模按模型校准，因此相同的级别名称在模型间不代表相同的基础值。

<h4 id="use-ultrathink-for-one-off-deep-reasoning">
  使用 ultrathink 进行一次性深度推理
</h4>

在您的提示中的任何地方包含 `ultrathink` 以请求该轮次的更深推理，而不改变您的会话努力设置。Claude Code 识别关键字并添加上下文内指令。发送到 API 的努力级别保持不变。Claude Code 将其他短语如"think"、"think hard"和"think more"作为普通提示文本传递，不将它们识别为关键字。

<h4 id="set-the-effort-level">
  设置努力级别
</h4>

您可以通过以下任何方式更改努力：

* **`/effort`**：运行 `/effort` 不带参数以打开交互式滑块，`/effort` 后跟级别名称以直接设置它，或 `/effort auto` 以清除活动模型的保存级别。您可以在 Claude 工作时运行它，一旦您确认[缓存警告](/docs/zh-CN/prompt-caching#changing-effort-level)（如果 Claude Code 显示一个），Claude Code 会将新级别应用于轮次中的下一个请求
* **在 `/model` 中**：选择模型时使用左/右箭头键调整努力滑块
* **`--effort` 标志**：启动 Claude Code 时传递级别名称以为单个会话设置它
* **环境变量**：将 `CLAUDE_CODE_EFFORT_LEVEL` 设置为级别名称或 `auto`
* **设置**：在 [`modelSettings`](/docs/zh-CN/settings-reference#modelsettings) 中设置每个模型的级别，或将 [`effortLevel`](/docs/zh-CN/settings-reference#effortlevel) 设置为 `low`、`medium`、`high` 或 `xhigh` 作为没有级别的模型的默认值。`max` 在任一键中都不被接受为级别，`ultracode` 有其自己的 [`ultracode`](/docs/zh-CN/settings-reference#ultracode) 键
* **从连接的设备**：在[远程控制](/docs/zh-CN/remote-control#what-connected-devices-see)会话中，从您的手机或浏览器上的努力控制中选择级别。该级别仅适用于当前会话，尽管它也结束[对模型默认努力的保持](#adjust-effort-level)。需要 Claude Code v2.1.234 或更高版本
* **Skill 和子代理 frontmatter**：在 [skill](/docs/zh-CN/skills#frontmatter-reference) 或[子代理](/docs/zh-CN/sub-agents#supported-frontmatter-fields) markdown 文件中设置 `effort` 以在该 skill 或子代理运行时覆盖努力级别

Frontmatter 努力在该 skill 或子代理活跃时应用，覆盖会话级别但不覆盖环境变量。

如果您在[托管设置](/docs/zh-CN/managed-settings)中设置 `effortLevel`，Claude Code 在[努力解析顺序](#adjust-effort-level)的设置步骤处应用它，用户仍然可以使用 `/effort` 或 `--effort` 更改级别。要将用户保持在或低于某个级别，设置 [`maxEffortLevel`](/docs/zh-CN/settings-reference#maxeffortlevel)。

努力滑块在选择支持的模型时出现在 `/model` 中。当前努力级别也显示在会话标题中模型名称旁边，例如"with low effort"，因此您可以确认哪个设置处于活跃状态，而无需打开 `/model`。页脚也在启动和更改时简要显示努力级别。

<h4 id="adaptive-reasoning-and-fixed-thinking-budgets">
  自适应推理和固定思考预算
</h4>

自适应推理使思考在每一步上可选，因此 Claude 可以更快地响应例行提示，并为受益于它的步骤保留更深入的思考。如果您想要 Claude 比当前级别产生的更频繁或更少地思考，您可以直接在您的提示或 `CLAUDE.md` 中说出来；模型在其努力设置内响应该指导。

Fable 模型、Sonnet 5 和 Opus 4.7 及更高版本始终使用自适应推理。固定思考预算模式和 `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` 不适用于它们。

在 Opus 4.6 和 Sonnet 4.6 上，您可以设置 `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1` 以恢复到由 `MAX_THINKING_TOKENS` 控制的先前固定思考预算。请参阅[环境变量](/docs/zh-CN/env-vars)。

<h3 id="extended-thinking">
  扩展思考
</h3>

扩展思考是 Claude 在响应前发出的推理。在支持[自适应推理](#adjust-effort-level)的模型上，努力级别是对发生多少思考的主要控制；下面的设置打开或关闭思考并控制它如何显示。在 Anthropic API 上关闭思考时，Claude Code 向它知道[不接受该组合](/docs/zh-CN/errors#effort-isnt-available-with-thinking-turned-off)的模型（如 Opus 5）发送努力 `high` 而不是更高级别。

| 控制       | 如何设置                                                                                                                                                                                                                                      |
| :------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 当前会话的切换  | 在 macOS 上按 `Option+T` 或在 Windows 和 Linux 上按 `Alt+T`                                                                                                                                                                                       |
| 设置全局默认值  | 运行 `/config` 并切换思考模式。保存为 `~/.claude/settings.json` 中的 `alwaysThinkingEnabled`                                                                                                                                                             |
| 通过环境变量禁用 | 设置 [`MAX_THINKING_TOKENS=0`](/docs/zh-CN/env-vars)，这在 Anthropic API 上关闭思考，除了 Fable 模型。在[第三方提供商](/docs/zh-CN/third-party-integrations)上，Claude Code 改为省略 `thinking` 参数，自适应推理模型可能仍然思考。其他值仅适用于[固定思考预算](#adaptive-reasoning-and-fixed-thinking-budgets) |

您不能在 Fable 模型上关闭思考。会话切换、`alwaysThinkingEnabled` 和 `MAX_THINKING_TOKENS=0` 在那里没有效果，Fable 模型根据努力级别按步骤决定思考多少。

Claude Code 默认折叠思考输出。按 `Ctrl+O` 切换详细模式并将推理视为灰色斜体文本。Anthropic API 上的交互式会话默认接收编辑的思考块，因此如果您想要完整摘要在展开时可用，在[设置](/docs/zh-CN/settings)中设置 `showThinkingSummaries: true`。您需要为所有生成的思考令牌付费，即使折叠或编辑。

<h3 id="extended-context">
  扩展上下文
</h3>

Fable 5.1、Fable 5、Sonnet 5、Opus 4.6 及更高版本和 Sonnet 4.6 支持[100 万令牌上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows#context-window-sizes-by-model)，用于具有大型代码库的长会话。

可用性因模型和计划而异。在 Anthropic API 上，Fable 5.1、Fable 5、Sonnet 5 和 Opus 4.7 及更高版本默认运行 1M 窗口。

在 Max、Team 和 Enterprise 计划上，包括 Team Standard 和 Team Premium 席位，Opus 自动升级到 1M 上下文，无需额外配置。Sonnet 4.6 与 1M 上下文不是自动升级的一部分，在包括 Max 的每个订阅计划上都需要[使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans)。

| 计划                    | Opus 与 1M 上下文                                                                               | Sonnet 4.6 与 1M 上下文                                                                         |
| --------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Max、Team 和 Enterprise | 包含在订阅中                                                                                      | 需要[使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans) |
| Pro                   | 需要[使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans) | 需要[使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans) |
| API 和按量付费             | 完全访问                                                                                        | 完全访问                                                                                        |

Claude Code 仅在直接连接到 Anthropic API 时检查这些计划要求。如果您将 `ANTHROPIC_BASE_URL` 指向[LLM 网关](/docs/zh-CN/llm-gateway#subscriptions-and-gateways)，您保存的 claude.ai 登录保持活跃凭证，Claude Code 不检查账户的使用额度。`/model` 中的 `[1m]` 选项保持可用，网关决定请求是否成功。在 v2.1.229 之前，当 Claude Code 无法确认账户上的使用额度时，它在该配置中拒绝 `/model sonnet[1m]`。

要关闭 1M 上下文，设置 `CLAUDE_CODE_DISABLE_1M_CONTEXT=1`。Claude Code 从模型选择器中删除 1M 模型变体。在具有本地 1M 窗口的模型上，例如 Sonnet 5 和 Fable 模型，它也将模型视为具有 200K 上下文窗口：

* 启用自动压缩时，会话在 200K 边界处通过[自动压缩](#set-the-auto-compact-window)进行压缩。将自动压缩窗口设置在 200K 以上不会解除保持，因为 Claude Code 将该窗口限制为模型的上下文窗口。
* 禁用自动压缩时，会话在 200K 边界处停止，出现[上下文限制错误](/docs/zh-CN/errors#prompt-is-too-long)，而不是压缩。

在 v2.1.223 之前，Claude Code 仅将 Sonnet 5、Opus 4.8 和 Opus 5 会话保持在 200K。请参阅[环境变量](/docs/zh-CN/env-vars)。

1M 上下文窗口使用标准模型定价，超过 200K 的令牌没有溢价。对于扩展上下文包含在您的订阅中的计划，使用仍由您的订阅覆盖。对于通过使用额度访问扩展上下文的计划，令牌计费到使用额度。

如果您的账户支持 1M 上下文，该选项会出现在最新版本的 Claude Code 的 `/model` 选择器中。如果您看不到它，请尝试重新启动您的会话。

您也可以使用 `[1m]` 后缀与模型别名或完整模型名称：

```text theme={null}
# 使用 opus[1m] 或 sonnet[1m] 别名
/model opus[1m]
/model sonnet[1m]

# 或将 [1m] 附加到完整模型名称
/model claude-opus-4-8[1m]
```

<h4 id="sonnet-5-context-window">
  Sonnet 5 上下文窗口
</h4>

在 Anthropic API 上，Sonnet 5 始终运行 1M 上下文窗口。没有 200K 变体，没有 `[1m]` 后缀可选择，任何计划上都不需要使用额度。会话在窗口填满前自动压缩，默认约 967K 令牌；设置 [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](/docs/zh-CN/env-vars) 以选择不同的阈值。

两个配置将窗口预算为 200K：

* **LLM 网关**：当 `ANTHROPIC_BASE_URL` 指向[网关](/docs/zh-CN/llm-gateway)时，Claude Code 无法验证 1M 支持。要使用完整窗口，在模型选择器中选择 Sonnet 5 (1M context)，它映射到 `sonnet[1m]`。
* **`CLAUDE_CODE_DISABLE_1M_CONTEXT=1`**：将具有本地 1M 窗口的每个模型上的会话保持在 200K 窗口；请参阅[扩展上下文](#extended-context)了解保持如何被强制执行。对于需要限制上下文的部署很有用。

<h2 id="context-window-and-auto-compaction">
  上下文窗口和自动压缩
</h2>

自动压缩窗口是指在 Claude Code 压缩对话之前，上下文窗口可以有多满。关于压缩保留和删除的内容，请参阅[压缩后保留的内容](/docs/zh-CN/context-window#what-survives-compaction)。

<h3 id="set-the-auto-compact-window">
  设置自动压缩窗口
</h3>

您可以在三个地方设置自动压缩窗口：

* **对于此会话及以后的会话**：运行 `/autocompact` 命令并指定一个值，例如 `/autocompact 500k`。Claude Code 将其保存到您的用户设置中作为 [`autoCompactWindow`](/docs/zh-CN/settings-reference#autocompactwindow)，并将其应用于当前会话；如果更高优先级的[设置范围](/docs/zh-CN/settings#settings-precedence)（例如托管设置）设置了该键，该命令会保存您的值，但会话会保持该范围的窗口，命令会说明这一点。运行 `/autocompact auto` 以返回为您的模型调整的窗口。
* **对于一次启动**：启动 Claude Code 时传递 [`--autocompact`](/docs/zh-CN/cli-reference#cli-flags)。该标志会为该次启动覆盖您保存的设置，而不会更改它，`claude --autocompact auto` 会以调整的窗口运行会话，即使您保存的设置有一个值。与 `/autocompact` 不同，该标志不会被更高优先级的设置范围（例如托管设置）抢占。
* **在脚本和云环境中**：设置 [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](/docs/zh-CN/env-vars)。设置后，它优先于命令、标志和设置，`/autocompact` 会报告该覆盖而不是更改窗口。

命令和标志接受 100K 到 1M 令牌的窗口大小，采用以下任何形式：

* 纯令牌计数，例如 `200000`
* `k` 或 `M` 后缀，例如 `500k` 或 `1M`
* 100 到 1000 之间的裸数字，表示千位，所以 `200` 设置 200,000

环境变量仅接受纯令牌计数。Claude Code 将窗口限制在模型的上下文窗口。

<h3 id="default-auto-compact-thresholds">
  默认自动压缩阈值
</h3>

如果您没有设置自动压缩窗口，Claude Code 会在对话达到模型的上下文限制时进行压缩，除了以下会话：

* [云会话](/docs/zh-CN/claude-code-on-the-web)在对话接近模型限制时进行压缩
* Sonnet 4.6 和 Opus 4.6（不带[扩展上下文](#extended-context)）在 200K 边界处进行压缩，Opus 4.8 和 Opus 5 在使用 200K 上下文窗口运行时也是如此，例如在 Amazon Bedrock、Google Cloud 的 Agent Platform 和 Microsoft Foundry 上
* 当您设置 [`CLAUDE_CODE_DISABLE_1M_CONTEXT=1`](/docs/zh-CN/env-vars) 时，具有原生 1M 窗口的模型（例如 Sonnet 5 和 Fable 模型）在 200K 边界处进行压缩
* 使用原生 1M 窗口运行的模型（例如 Sonnet 5、Fable 模型以及 Anthropic API 上的 Opus 4.7 及更高版本）在窗口填满之前进行压缩，默认情况下约为 967K 令牌。在 Amazon Bedrock、Google Cloud 的 Agent Platform 和 Microsoft Foundry 上，[为第三方部署固定模型](#pin-models-for-third-party-deployments)说明了哪些模型使用该窗口；对于将 Sonnet 5 预算为 200K 的配置，请参阅 [Sonnet 5 上下文窗口](#sonnet-5-context-window)
* 在 Claude Code 不识别的模型 ID（例如 [LLM 网关](/docs/zh-CN/llm-gateway)别名）上的会话在 Claude Code 为该 ID 假设的上下文窗口处进行压缩；请参阅[为网关或自定义模型 ID 更正窗口](#correct-the-window-for-a-gateway-or-custom-model-id)

<h3 id="correct-the-window-for-a-gateway-or-custom-model-id">
  为网关或自定义模型 ID 更正窗口
</h3>

在 [LLM 网关](/docs/zh-CN/llm-gateway)或其他自定义部署上，Claude Code 可能会为模型 ID 假设一个与模型实际窗口不同的上下文窗口，无论它是否将 ID 解析为 Claude 模型。设置 [`CLAUDE_CODE_MAX_CONTEXT_TOKENS`](/docs/zh-CN/env-vars) 为 Claude Code 应该假设的窗口。

变量的应用方式取决于 ID。当 Claude Code 不以 `claude-`（任何大小写）开头时，或当它携带 Claude Code 在读取 ID 时剥离的后缀（例如 Google Cloud 的 Agent Platform 上使用的 `@YYYYMMDD` 日期）时，Claude Code 将 ID 视为提供商或自定义拼写。在 v2.1.259 之前，Claude Code 没有计算剥离的后缀，所以带有日期后缀的无法识别的 `claude-` ID 被视为裸 `claude-` 名称。

无法识别的提供商或自定义拼写、相同拼写加上 `[1m]` 和所有其他 ID 是三种不同的情况：

* 如果 Claude Code 无法将提供商或自定义拼写解析为它识别的模型，且 ID 不包含 `[1m]`，则变量直接应用，主动压缩在声明的窗口处继续。
* 如果 Claude Code 无法将提供商或自定义拼写解析为它识别的模型，且 ID 包含 `[1m]`（任何大小写），Claude Code 为其假设 1M 窗口，变量本身不适用。要在保持主动压缩的同时更正窗口，还要设置 [`CLAUDE_CODE_DISABLE_1M_CONTEXT=1`](/docs/zh-CN/env-vars)。设置该变量后，Claude Code 会像相同拼写但不带 `[1m]` 一样调整 ID 的大小，所以当 `CLAUDE_CODE_MAX_CONTEXT_TOKENS` 适用于该无标记拼写时，它就会应用。

  使用声明的窗口大于 200K 时，Claude Code 会显示一个[启动警告](/docs/zh-CN/errors#the-200k-limit-isnt-enforced)，表示 200K 限制未被强制执行。此配置中的警告是预期的。
* 如果 ID 解析为 Claude Code 识别的模型，或 ID 是没有 Claude Code 要剥离的后缀的裸 `claude-` 名称（任何大小写），变量仅在您也设置 [`DISABLE_COMPACT`](/docs/zh-CN/env-vars) 时生效，这会禁用所有压缩。

  例如，包含 Claude Code 知道的 Claude 模型名称的 ID（例如 `anthropic/claude-opus-4-8`、`us.anthropic.claude-…-v1:0` 或带日期的 `claude-sonnet-4-5@20250929`）解析为该模型。这包括也包含 `[1m]` 的 ID：即使设置了 `CLAUDE_CODE_DISABLE_1M_CONTEXT`，Claude Code 也会将 `claude-opus-4-8[1m]` 解析为 Opus 4.8。

对于 Claude Code 不识别的模型 ID，设置 [`CLAUDE_CODE_DISABLE_UNKNOWN_MODEL_WINDOW_ENFORCEMENT=1`](/docs/zh-CN/env-vars) 以让 Claude Code 仅在 API 以 Claude Code 识别的[过长错误](/docs/zh-CN/errors#prompt-is-too-long)拒绝对话后才进行压缩。当网关[将错误重写](/docs/zh-CN/llm-gateway-connect#troubleshoot-gateway-errors)为 Claude Code 不识别的措辞时，Claude Code 不会运行该恢复。

<h2 id="checking-your-current-model">
  检查您当前的模型
</h2>

您可以在两个位置查看您当前使用的模型：

* 在[状态行](/docs/zh-CN/statusline)中（如果已配置）
* 在 `/status` 中，它也显示您的账户信息

<h2 id="add-a-custom-model-option">
  添加自定义模型选项
</h2>

使用 `ANTHROPIC_CUSTOM_MODEL_OPTION` 向 `/model` 选择器添加单个自定义条目，而无需替换内置别名。这对于测试 Claude Code 默认不列出的模型 ID 很有用。对于 LLM 网关部署，当设置 `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1` 时，Claude Code 可以从网关的 `/v1/models` 端点自动填充选择器，因此仅当发现被禁用或未返回您想要的模型时才需要此变量。请参阅 [网关模型发现](/docs/zh-CN/llm-gateway-protocol#model-discovery)。

要列出多个模型，按您自己的顺序和您选择的标签，请设置 [`modelPicker`](/docs/zh-CN/settings-reference#modelpicker)。其条目说明当该阵容替换内置阵容时选择器保留哪些行。

此示例设置所有三个变量以使网关路由的 Opus 部署可选择。Claude Code 在启动时读取环境变量，因此在启动 `claude` 之前运行导出，或重启现有会话以获取它们：

```bash theme={null}
export ANTHROPIC_CUSTOM_MODEL_OPTION="my-gateway/claude-opus-5"
export ANTHROPIC_CUSTOM_MODEL_OPTION_NAME="Opus via Gateway"
export ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION="Custom deployment routed through the internal LLM gateway"
```

`ANTHROPIC_CUSTOM_MODEL_OPTION_NAME` 和 `ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION` 是可选的：

* 如果您省略名称，当 Claude Code [识别 ID](#customize-pinned-model-display-and-capabilities) 时条目显示模型的名称，否则显示模型 ID。
* 如果您省略描述，Claude Code 使用 `Custom model (<model-id>)`。

Claude Code 在内置条目之后列出自定义条目，您追加的任何 [`modelPicker`](/docs/zh-CN/settings-reference#modelpicker) 行都在其后面。

Claude Code 跳过对 `ANTHROPIC_CUSTOM_MODEL_OPTION` 中设置的模型 ID 的验证，因此您可以使用您的 API 端点接受的任何字符串。

当设置 [`availableModels`](#restrict-model-selection) 时，也要在允许列表中包含自定义模型 ID。否则 Claude Code 会从选择器中过滤自定义条目，并拒绝对其进行 `--model` 选择，就像任何其他被排除的模型一样。

嵌入了系列名称的自定义 ID（例如 `my-gateway/claude-opus-5`）计为该系列的特定条目并禁用其通配符，因此还要列出您打算保持可选择的版本。请参阅 [合并行为](#merge-behavior)。

<h2 id="environment-variables">
  环境变量
</h2>

使用以下环境变量来控制别名映射到的模型名称。每个值必须是完整的模型名称，或您的 API 提供商的等效标识符。要选择会话启动时使用的模型，请设置 [`ANTHROPIC_DEFAULT_MODEL`](#set-a-default-model-for-new-sessions)，此表中省略了该变量。

| 环境变量                             | 描述                                                                                                                                                                                                                                                                                                                             |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ANTHROPIC_DEFAULT_FABLE_MODEL`  | 用于 `fable` 的模型，以及 Claude Code 识别为 Fable 模型的模型 ID，用于[第三方提供商上的自动模型回退](#automatic-model-fallback)                                                                                                                                                                                                                                 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL`   | 用于 `opus` 的模型，或在 Plan Mode 活跃时用于 `opusplan` 的模型。                                                                                                                                                                                                                                                                               |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | 用于 `sonnet` 的模型，或在 Plan Mode 不活跃时用于 `opusplan` 的模型。                                                                                                                                                                                                                                                                            |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL`  | 用于 `haiku` 的模型，或[后台功能](/docs/zh-CN/costs#background-token-usage)                                                                                                                                                                                                                                                                    |
| `CLAUDE_CODE_SUBAGENT_MODEL`     | [subagents](/docs/zh-CN/sub-agents#choose-a-model)、[agent team](/docs/zh-CN/agent-teams#specify-teammates-and-models) 队友和[工作流](/docs/zh-CN/workflows)代理的默认模型，这些代理没有以其他方式分配模型。接受别名（如 `haiku`）或完整模型名称。每次调用的模型或定义的 `model` 字段（包括 `inherit`）优先。要更改该设置，请设置 [`CLAUDE_CODE_SUBAGENT_MODEL_FORCE`](/docs/zh-CN/sub-agents#run-every-subagent-on-one-model) |

注意：`ANTHROPIC_SMALL_FAST_MODEL` 已弃用，改为使用 `ANTHROPIC_DEFAULT_HAIKU_MODEL`。

<h3 id="pin-models-for-third-party-deployments">
  为第三方部署固定模型
</h3>

当通过 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock)、[Google Cloud's Agent Platform](/docs/zh-CN/google-vertex-ai)、[Microsoft Foundry](/docs/zh-CN/microsoft-foundry) 或 [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 部署 Claude Code 时，在向用户推出前固定模型版本。

不固定模型，Claude Code 会使用模型别名（如 `fable`、`opus`、`sonnet` 和 `haiku`），这些别名会解析为每个提供商的内置默认模型 ID。该默认值可能滞后于最新的 Anthropic 版本，并且它指向的模型可能尚未在用户账户中启用。当默认值不可用时，Amazon Bedrock 和 Google Cloud's Agent Platform 用户会看到通知并回退到该默认模型的早期版本，或当默认值是 Opus 模型且没有 Opus 版本可用时回退到默认 Sonnet 模型。Microsoft Foundry 用户会看到错误，因为 Microsoft Foundry 没有等效的启动检查。

在 Amazon Bedrock 和 Google Cloud's Agent Platform 上，以特定 Sonnet 或 Opus 版本启动会话的用户（例如使用 `--model`、`ANTHROPIC_MODEL` 或 `model` 设置），会将该版本固定为会话的默认值，用于匹配的别名：启动检查会跳过它替换的内置默认值，并且不显示回退通知。在 v2.1.211 之前，即使会话模型被显式配置，检查也会运行并可能显示通知。

<Warning>
  在初始设置中将模型环境变量设置为特定版本 ID。固定让您控制用户何时迁移到新模型。
</Warning>

对您的提供商使用以下环境变量和特定版本的模型 ID：

| 提供商                           | 示例                                                                   |
| :---------------------------- | :------------------------------------------------------------------- |
| Amazon Bedrock                | `export ANTHROPIC_DEFAULT_OPUS_MODEL='us.anthropic.claude-opus-4-8'` |
| Google Cloud's Agent Platform | `export ANTHROPIC_DEFAULT_OPUS_MODEL='claude-opus-4-8'`              |
| Microsoft Foundry             | `export ANTHROPIC_DEFAULT_OPUS_MODEL='claude-opus-4-8'`              |

对 `ANTHROPIC_DEFAULT_FABLE_MODEL`、`ANTHROPIC_DEFAULT_SONNET_MODEL` 和 `ANTHROPIC_DEFAULT_HAIKU_MODEL` 应用相同的模式。有关所有提供商的当前和旧版模型 ID，请参阅[模型概览](https://platform.claude.com/docs/en/about-claude/models/overview)。要将用户升级到新模型版本，请更新这些环境变量并重新部署。

要为固定模型启用[扩展上下文](#extended-context)，请在 `ANTHROPIC_DEFAULT_OPUS_MODEL`、`ANTHROPIC_DEFAULT_SONNET_MODEL` 或 `ANTHROPIC_DEFAULT_FABLE_MODEL` 中的模型 ID 后附加 `[1m]`：

```bash theme={null}
export ANTHROPIC_DEFAULT_OPUS_MODEL='claude-opus-4-8[1m]'
```

使用 `[1m]` 后缀，1M 上下文窗口适用于固定别名的所有使用，包括 [`opusplan`](#opusplan-model-setting) 的 plan-mode Opus 阶段和 `model` frontmatter 命名别名的 [subagents](/docs/zh-CN/sub-agents#choose-a-model)。

* Claude Code 在将模型 ID 发送到您的提供商之前会删除该后缀。
* 仅当底层模型[支持 1M 上下文](https://platform.claude.com/docs/en/build-with-claude/context-windows#context-window-sizes-by-model)时才附加 `[1m]`。
* 该后缀按变量读取，而不是按模型读取。在 Amazon Bedrock、Google Cloud's Agent Platform 和 Microsoft Foundry 上，一个变量中没有 `[1m]` 的模型 ID 使用 200K 上下文，即使另一个变量使用相同的模型和后缀。Sonnet 5 在这些提供商上始终以 1M 窗口运行，从不需要该后缀。

<Note>
  通过 [MDM 或托管设置文件](/docs/zh-CN/managed-settings#delivery-mechanisms) 提供的 `availableModels` 允许列表在使用第三方提供商时仍然适用；[服务器托管设置不会在那里提供](/docs/zh-CN/server-managed-settings#platform-availability)。

  过滤与模型别名（如 `opus`）、版本前缀（如 `claude-opus-4-8`）或完整提供商形式的模型 ID 匹配。提供商特定的前缀（如 `us.anthropic.`）不会被删除，因此要允许特定模型，请列出其完整提供商形式 ID，或通过 [`modelOverrides`](#override-model-ids-per-version) 映射它。对于固定模型，该 ID 是您在其 `ANTHROPIC_DEFAULT_*_MODEL` 变量中设置的值。任何 `[1m]` 后缀在匹配前都会从允许列表条目和请求的模型中删除。
</Note>

<h3 id="customize-pinned-model-display-and-capabilities">
  自定义固定模型显示和功能
</h3>

当您在第三方提供商上固定模型时，其在 `/model` 选择器中的行如果 Claude Code 识别固定 ID，则默认显示模型的名称，否则显示原始 ID：

* **已识别**：Claude Code 知道的模型的确切 ID，例如其 Anthropic API ID 或您的提供商或网关的形式，带或不带 `[1m]` 后缀。固定 `us.anthropic.claude-sonnet-4-5-20250929-v1:0`，该行显示 `Sonnet 4.5`。
* **未识别**：任何其他 ID，例如应用推理配置文件 ARN 或 Claude Code 不知道的模型版本，除非 [`modelOverrides`](#override-model-ids-per-version) 条目将模型映射到该确切字符串。在 Microsoft Foundry 上，部署名称是用户定义的，因此 Claude Code 永远不会识别固定 ID，无论是否映射，该行默认显示部署名称。

当行显示模型的名称时，其默认描述包括固定 ID，以便您仍然可以看到固定了哪个 ID。

Claude Code 也可能无法识别固定模型支持的功能。您可以自己设置显示名称和描述，并为每个固定模型使用伴随环境变量声明功能。

这些变量在第三方提供商（如 Amazon Bedrock、Google Cloud's Agent Platform 和 Microsoft Foundry）上生效。`_NAME` 和 `_DESCRIPTION` 变量在 `ANTHROPIC_BASE_URL` 指向 [LLM gateway](/docs/zh-CN/llm-gateway) 时也生效。当直接连接到 `api.anthropic.com` 时无效。

| 环境变量                                                  | 描述                                                                             |
| ----------------------------------------------------- | ------------------------------------------------------------------------------ |
| `ANTHROPIC_DEFAULT_OPUS_MODEL_NAME`                   | 固定 Opus 模型在 `/model` 选择器中的显示名称。未设置时，如果 Claude Code 识别固定 ID，该行显示模型的名称，否则显示固定 ID |
| `ANTHROPIC_DEFAULT_OPUS_MODEL_DESCRIPTION`            | 固定 Opus 模型在 `/model` 选择器中的显示描述。未设置时，该行显示以 `Custom Opus model` 开头的默认描述          |
| `ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES` | 固定 Opus 模型支持的功能的逗号分隔列表                                                         |

相同的 `_NAME`、`_DESCRIPTION` 和 `_SUPPORTED_CAPABILITIES` 后缀可用于 `ANTHROPIC_DEFAULT_SONNET_MODEL`、`ANTHROPIC_DEFAULT_HAIKU_MODEL`、`ANTHROPIC_DEFAULT_FABLE_MODEL` 和 `ANTHROPIC_CUSTOM_MODEL_OPTION`。

Claude Code 通过将模型 ID 与已知模式匹配来启用[工作量级别](#adjust-effort-level)和[扩展思考](#extended-thinking)等功能。提供商特定的 ID（如 Amazon Bedrock ARN 或自定义部署名称）通常与这些模式不匹配，导致支持的功能被禁用。设置 `_SUPPORTED_CAPABILITIES` 以告诉 Claude Code 模型实际支持的功能：

| 功能值                    | 启用                                          |
| ---------------------- | ------------------------------------------- |
| `effort`               | [工作量级别](#adjust-effort-level)和 `/effort` 命令 |
| `xhigh_effort`         | `xhigh` 工作量级别                               |
| `max_effort`           | `max` 工作量级别                                 |
| `thinking`             | [扩展思考](#extended-thinking)                  |
| `adaptive_thinking`    | 根据任务复杂性动态分配思考的自适应推理                         |
| `interleaved_thinking` | 工具调用之间的思考                                   |

设置 `_SUPPORTED_CAPABILITIES` 时，列出的功能对匹配的固定模型启用，未列出的功能被禁用。未设置变量时，Claude Code 回退到基于模型 ID 的内置检测。

此示例将 Opus 固定到 Amazon Bedrock 自定义模型 ARN，设置友好名称，并声明其功能：

```bash theme={null}
export ANTHROPIC_DEFAULT_OPUS_MODEL='arn:aws:bedrock:us-east-1:123456789012:custom-model/abc'
export ANTHROPIC_DEFAULT_OPUS_MODEL_NAME='Opus via Bedrock'
export ANTHROPIC_DEFAULT_OPUS_MODEL_DESCRIPTION='Opus 4.7 routed through a Bedrock custom endpoint'
export ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES='effort,xhigh_effort,max_effort,thinking,adaptive_thinking,interleaved_thinking'
```

<h3 id="override-model-ids-per-version">
  按版本覆盖模型 ID
</h3>

在嵌入 Claude Code 并设置 [`CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST`](/docs/zh-CN/env-vars) 的平台上，主机的模型配置优先于托管模型设置，而托管 `availableModels` 允许列表保持有效，除非主机提供自己的；[托管设置优先级的例外](/docs/zh-CN/settings#exceptions-to-managed-settings-precedence)说明主机覆盖的键和变量。

上面的家族级环境变量为每个家族别名配置一个模型 ID。如果您需要将同一家族中的多个版本映射到不同的提供商 ID，请改用 `modelOverrides` 设置。

`modelOverrides` 将单个 Anthropic 模型 ID 映射到 Claude Code 发送到您的提供商 API 的提供商特定字符串。当用户在 `/model` 选择器中选择映射的模型时，Claude Code 会使用您配置的值而不是内置默认值。

这让企业管理员可以将每个模型版本路由到特定的 Amazon Bedrock 推理配置文件 ARN、Google Cloud's Agent Platform 版本名称或 Microsoft Foundry 部署名称，用于治理、成本分配或区域路由。

在您的[设置文件](/docs/zh-CN/settings#where-settings-live)中设置 `modelOverrides`：

```json theme={null}
{
  "modelOverrides": {
    "claude-opus-4-7": "arn:aws:bedrock:us-east-2:123456789012:application-inference-profile/opus-prod",
    "claude-opus-4-6": "arn:aws:bedrock:us-east-2:123456789012:application-inference-profile/opus-46-prod",
    "claude-sonnet-4-6": "arn:aws:bedrock:us-east-2:123456789012:application-inference-profile/sonnet-prod"
  }
}
```

键必须是[模型概览](https://platform.claude.com/docs/en/about-claude/models/overview)中列出的 Anthropic 模型 ID。对于带日期的模型 ID，请包含日期后缀，完全按照其显示的方式。未知的键会被忽略。

要停止针对网关别名等 ID 的 `[claude-code:unrecognized_model]` [诊断行](/docs/zh-CN/errors#unrecognized-model-id-on-a-request)，请添加一个以该 ID 作为其值的条目。

覆盖替换了支持 `/model` 选择器中每个条目的内置模型 ID。在 Amazon Bedrock 上，`modelOverrides` 条目优先于 Claude Code 在启动时自动发现的任何推理配置文件。Claude Code 将已经是提供商原生的值（如 Amazon Bedrock 推理配置文件 ARN 或 Microsoft Foundry 部署名称）按原样传递给提供商。

当您通过 `--model`、`ANTHROPIC_MODEL` 环境变量或 `ANTHROPIC_DEFAULT_*_MODEL` 环境变量直接传递 Anthropic 模型 ID 时，覆盖也适用。在 Amazon Bedrock、Google Cloud's Agent Platform 和 [Mantle](/docs/zh-CN/amazon-bedrock#use-the-mantle-endpoint) 上，没有 `modelOverrides` 条目的 Anthropic 模型 ID 解析为与该版本的 `/model` 选择器行相同的提供商特定 ID（当提供商支持该版本时）。Mantle 支持版本的子集。对于该子集之外的 Anthropic 模型 ID，Claude Code 将原始 ID 发送到 Mantle 而不进行映射，除非 `modelOverrides` 条目覆盖它。在 v2.1.200 之前，`--model` 和环境变量值直接到达提供商，不经过覆盖映射。

`modelOverrides` 与 `availableModels` 一起工作。允许列表针对 Anthropic 模型 ID 进行评估，而不是覆盖值，因此 `availableModels` 中的条目（如 `"opus"`）即使在 Opus 版本映射到 ARN 时也会继续匹配。当在托管设置中设置 `enforceAvailableModels` 时，强制执行的默认值通过 `modelOverrides` 从[托管设置](/docs/zh-CN/managed-settings#how-claude-code-combines-managed-sources)解析。管理员的映射（如固定到推理配置文件 ARN 的版本）在强制执行的默认值中得到遵守。来自用户或项目设置的覆盖不会影响它。

当 `availableModels` 在[托管设置](/docs/zh-CN/managed-settings)中设置时，仅来自托管设置的 `modelOverrides` 适用于通过 `--model` 或上述环境变量直接传递的 Anthropic 模型 ID。Claude Code 忽略用户或项目设置中针对这些 ID 的覆盖，并且永远不会通过任何设置源的 `modelOverrides` 解析托管列表排除的 ID。此托管源限制需要 Claude Code v2.1.200 或更高版本。有关如何处理被阻止的 ID，请参阅[限制模型选择](#restrict-model-selection)。

<h3 id="prompt-caching-configuration">
  Prompt caching 配置
</h3>

Claude Code 自动使用 [prompt caching](/docs/zh-CN/prompt-caching) 来优化性能并降低成本。您可以全局禁用 prompt caching 或针对特定模型层级禁用：

| 环境变量                            | 描述                                       |
| ------------------------------- | ---------------------------------------- |
| `DISABLE_PROMPT_CACHING`        | 设置为 `1` 以禁用所有模型的 prompt caching。优先于按模型设置 |
| `DISABLE_PROMPT_CACHING_HAIKU`  | 设置为 `1` 以仅禁用 Haiku 模型的 prompt caching    |
| `DISABLE_PROMPT_CACHING_SONNET` | 设置为 `1` 以仅禁用 Sonnet 模型的 prompt caching   |
| `DISABLE_PROMPT_CACHING_OPUS`   | 设置为 `1` 以仅禁用 Opus 模型的 prompt caching     |
| `DISABLE_PROMPT_CACHING_FABLE`  | 设置为 `1` 以仅禁用 Fable 模型的 prompt caching    |

要为主对话和 subagents 分别选择缓存 TTL，请参阅[自己选择 TTL](/docs/zh-CN/prompt-caching#choose-the-ttl-yourself)。有关什么会触发缓存未命中，请参阅 [Claude Code 如何使用 prompt caching](/docs/zh-CN/prompt-caching)。
