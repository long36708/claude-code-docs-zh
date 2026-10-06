> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 环境变量

> 控制 Claude Code 行为的环境变量参考。

环境变量可以控制 Claude Code 的行为，例如模型选择、身份验证、请求路由和功能切换。许多相同的行为也可以通过[设置文件](/docs/zh-CN/settings)字段、[CLI 标志](/docs/zh-CN/cli-reference)或会话内命令（如 `/model`）进行配置。

本页面涵盖以下内容：

* [在 shell 或设置文件中设置环境变量](#set-environment-variables)
* [当行为可以通过多种方式设置时，检查哪个值适用](#precedence)
* [查找 Claude Code 读取的变量](#variables)
* [查看当变量关闭功能标志获取时，哪些功能停止工作](#features-that-need-feature-flag-fetching)

<h2 id="set-environment-variables">
  设置环境变量
</h2>

在 shell 中设置的变量仅在该终端会话期间有效，而在设置文件中的变量每次运行 `claude` 时都会应用。

<h3 id="in-your-shell">
  在 shell 中
</h3>

在启动 `claude` 之前设置变量：

<Tabs>
  <Tab title="macOS, Linux, WSL">
    ```bash theme={null}
    export API_TIMEOUT_MS="1200000"
    claude
    ```

    要为每个会话设置它，请将 `export` 行添加到 `~/.bashrc`、`~/.zshrc` 或您的 shell 配置文件中。
  </Tab>

  <Tab title="Windows PowerShell">
    ```powershell theme={null}
    $env:API_TIMEOUT_MS = "1200000"
    claude
    ```

    要为每个会话设置它，请运行 `[Environment]::SetEnvironmentVariable("API_TIMEOUT_MS", "1200000", "User")` 并打开一个新终端。
  </Tab>

  <Tab title="Windows CMD">
    ```batch theme={null}
    set API_TIMEOUT_MS=1200000
    claude
    ```

    要为每个会话设置它，请运行 `setx API_TIMEOUT_MS "1200000"` 并打开一个新终端。
  </Tab>
</Tabs>

赋值行在成功时不会打印任何内容。要确认变量已设置，请在同一 shell 中打印它：

<Tabs>
  <Tab title="macOS, Linux, WSL">
    ```bash theme={null}
    echo $API_TIMEOUT_MS
    ```
  </Tab>

  <Tab title="Windows PowerShell">
    ```powershell theme={null}
    echo $env:API_TIMEOUT_MS
    ```
  </Tab>

  <Tab title="Windows CMD">
    ```batch theme={null}
    echo %API_TIMEOUT_MS%
    ```
  </Tab>
</Tabs>

<h3 id="in-settings-files">
  在设置文件中
</h3>

在 `settings.json` 文件中的 `env` 键下添加变量，如果文件不存在则创建它。Claude Code 直接从文件中读取它们，因此无论如何启动 `claude`，它们都会生效。运行中的会话在您保存文件时会将新值和更改的值应用到其环境中，但在启动时读取其变量一次的功能（例如 [OpenTelemetry 监控](/docs/zh-CN/monitoring-usage)）会保持其启动值，直到您重新启动。从文件中删除变量不会在运行中的会话中取消设置它；删除在您下次启动 `claude` 时生效。

```json ~/.claude/settings.json theme={null}
{
  "env": {
    "API_TIMEOUT_MS": "1200000",
    "BASH_DEFAULT_TIMEOUT_MS": "300000"
  }
}
```

您选择的文件控制变量应用于谁：

| 文件 | 应用于 |
| :- | :- |
| `~/.claude/settings.json` | 您，在每个项目中 |
| `.claude/settings.json` | 在项目中工作的每个人，检入源代码控制 |
| `.claude/settings.local.json` | 您，仅在此项目中，当 Claude Code 将设置保存到它时被 gitignore；如果您手动创建它，请将其添加到您的 gitignore |
| 托管设置 | 您组织中的每个人，由管理员部署 |

请参阅 [设置文件](/docs/zh-CN/settings#where-settings-live) 了解每个文件的位置，以及 [设置优先级](/docs/zh-CN/settings#settings-precedence) 了解当多个文件设置相同变量时它们如何组合。

<h2 id="precedence">
  优先级
</h2>

某些行为同时具有环境变量和专用设置键，Claude Code 读取哪一个的顺序因键而异。对于 `ANTHROPIC_MODEL` 和 `CLAUDE_CODE_AUTO_CONNECT_IDE`，Claude Code 首先读取变量，仅当变量未设置时才使用 `model` 或 `autoConnectIde` 设置。对于您正在设置的对，请检查下面变量的行和 [设置参考](/docs/zh-CN/settings-reference) 上的键条目。

当同一变量在您的 shell 和设置文件 `env` 块中都设置时，在大多数会话中设置文件值适用。Claude Code 将每个 `env` 条目写入进程环境，替换从 shell 继承的值。[`env` 值如何与您的 shell 交互](/docs/zh-CN/settings-reference#how-env-values-interact-with-your-shell) 涵盖保留继承值的会话，以及 [`env` 设置](/docs/zh-CN/settings-reference#when-claude-code-applies-env-values) 说明何时应用它们。少数变量是特殊情况；[`env` 设置](/docs/zh-CN/settings-reference#env) 列出了例外。

在设置文件中，您可以设置变量，但不能删除变量。要覆盖无法取消设置的变量，例如由您无法控制的 shell 配置文件导出的过时 `CLAUDE_CODE_USE_VERTEX`，请在 `env` 块中将其设置为空字符串：`"CLAUDE_CODE_USE_VERTEX": ""`。Claude Code 将空值视为未设置以进行提供程序选择。子进程仍然继承空值。

在设置文件之间，`env` 值遵循 [设置优先级](/docs/zh-CN/settings#settings-precedence)，因此托管设置条目覆盖用户或项目设置中的相同变量。项目和本地设置无法设置某些变量，例如 `CLAUDE_CONFIG_DIR` 和 OpenTelemetry 导出程序变量。[Claude Code 在 `env` 中忽略的变量](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env) 列出了它们，以及仍然适用的 OpenTelemetry 关闭值。

环境变量与 CLI 标志和会话内命令的交互方式因功能而异：`--model` 和 `/model` 覆盖 `ANTHROPIC_MODEL`，而 `CLAUDE_CODE_EFFORT_LEVEL` 覆盖 `--effort` 和 `/effort`。当变量与另一个配置源交互时，[变量](#variables) 列表中的其行说明优先级或链接到记录它的页面。

Claude Code 在启动时读取 shell 环境变量，因此对它们的更改在您下次启动 `claude` 时生效。在设置文件中 `env` 键下设置的变量在文件更改时重新应用到运行中的会话，但 [在设置文件中](#in-settings-files) 描述的仅启动时例外除外。

<h2 id="variables">
  变量
</h2>

数值类变量（例如超时时间、token 预算和重试次数）除纯数字外，还接受科学记数法和数字分隔符写法，除非某个变量所在行注明它仅接受纯数字。例如，Claude Code 将 `2e3` 读取为 2000，将 `64_000` 读取为 64000。在 v2.1.211 之前，这些写法可能会在没有任何提示的情况下设置一个小得多的值，例如 `1e6` 会将超时时间设置为 1。

<Note>
  对于用于启用或关闭某项行为的变量，设置 `1`、`true`、`yes` 或 `on` 可启用该行为，设置 `0`、`false`、`no` 或 `off` 可关闭该行为，不区分大小写。

  有些变量只检查您是否设置了它们，因此任何非空值（包括 `0`）都会启用该行为；要关闭该行为，需取消设置该变量或将其设为空值。以下变量按此方式工作：

  * `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
  * `DISABLE_TELEMETRY`
  * `DISABLE_ERROR_REPORTING`
  * `CLAUDE_CODE_TMUX_TRUECOLOR`
  * `FALLBACK_FOR_ALL_PRIMARY_MODELS`
  * `IS_DEMO`

  还有一个变量有其自己的规则：`FORCE_HYPERLINK` 读取的是数字，因此只有 `0` 会将其关闭。每个变量所在行也会说明其自身的规则。
</Note>

| 变量 | 用途 |
| :- | :- |
| `ANTHROPIC_API_KEY` | 作为 `X-Api-Key` 标头发送的 API 密钥。设置后，即使您已登录，也会使用此密钥，而不是您的 Claude Pro、Max、Team 或 Enterprise 订阅。在非交互模式（`-p`）下，只要存在该密钥就始终会使用它。在交互模式下，系统会提示您批准该密钥一次，之后它才会覆盖您的订阅。要改用您的订阅，请运行 `unset ANTHROPIC_API_KEY` |
| `ANTHROPIC_AUTH_TOKEN` | `Authorization` 标头的自定义值（您在此处设置的值将以 `Bearer ` 为前缀） |
| `ANTHROPIC_AWS_API_KEY` | 用于 [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 的工作区 API 密钥，在 AWS Console 中生成。作为 `x-api-key` 发送，并优先于 AWS SigV4 |
| `ANTHROPIC_AWS_BASE_URL` | 覆盖 [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 端点 URL。用于自定义区域或通过 [LLM 网关](/docs/zh-CN/llm-gateway)路由时。默认为 `https://aws-external-anthropic.{region}.api.aws`。Claude Code 按照[与 Amazon Bedrock 相同的优先级](/docs/zh-CN/amazon-bedrock#3-configure-claude-code)解析区域 |
| `ANTHROPIC_AWS_WORKSPACE_ID` | [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 必需。每个请求都会将其作为 `anthropic-workspace-id` 标头发送 |
| `ANTHROPIC_BASE_URL` | 覆盖 API 端点，以便通过代理或网关路由请求。当设置为非第一方主机时，[MCP 工具搜索](/docs/zh-CN/mcp#scale-with-mcp-tool-search)默认处于禁用状态。如果您的代理会转发 `tool_reference` 块，请设置 `ENABLE_TOOL_SEARCH=true`。从 v2.1.196 开始，当此变量指向 `api.anthropic.com` 以外的主机时，[Remote Control](/docs/zh-CN/remote-control#requirements) 将被禁用，与其在 Amazon Bedrock、Google Cloud's Agent Platform 和 Microsoft Foundry 上的行为一致 |
| `ANTHROPIC_BEDROCK_BASE_URL` | 覆盖 Amazon Bedrock 端点 URL。用于自定义 Amazon Bedrock 端点或通过 [LLM 网关](/docs/zh-CN/llm-gateway)路由时。请参阅 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock) |
| `ANTHROPIC_BEDROCK_MANTLE_BASE_URL` | 覆盖 Amazon Bedrock Mantle 端点 URL。请参阅 [Mantle 端点](/docs/zh-CN/amazon-bedrock#use-the-mantle-endpoint) |
| `ANTHROPIC_BEDROCK_REGION_PREFIX` | Claude Code 优先尝试的跨区域推理配置文件前缀（`us`、`eu`、`apac`、`jp`、`au` 或 `global`），而不是根据 AWS 区域推导出的前缀。在 AWS GovCloud 区域中会被忽略。需要 Claude Code v2.1.224 或更高版本。请参阅 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock#cross-region-inference-profile-prefixes) |
| `ANTHROPIC_BEDROCK_SERVICE_TIER` | Amazon Bedrock [服务层级](https://docs.aws.amazon.com/bedrock/latest/userguide/service-tiers-inference.html)（`default`、`flex` 或 `priority`）。作为 `X-Amzn-Bedrock-Service-Tier` 标头发送。请参阅 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock#service-tiers) |
| `ANTHROPIC_BETAS` | 要包含在 API 请求中的其他 `anthropic-beta` 标头值的逗号分隔列表。Claude Code 已经会发送其所需的 beta 标头；使用此变量可在 Claude Code 添加原生支持之前选择加入某个 [Anthropic API beta](https://platform.claude.com/docs/en/api/beta-headers)。与需要 API 密钥身份验证的 [`--betas` 标志](/docs/zh-CN/cli-reference#cli-flags)不同，此变量适用于所有身份验证方式，包括 Claude.ai 订阅 |
| `ANTHROPIC_CUSTOM_HEADERS` | 要添加到请求中的自定义标头（`Name: Value` 格式，多个标头以换行分隔）。如果名称或值包含 HTTP 标头无法承载的字符（例如弯引号或零宽空格），请求将失败，并显示一个按位置标识该名称-值对的错误。需要 Claude Code v2.1.227 或更高版本。[无效的请求标头值](/docs/zh-CN/errors#invalid-request-header-value)列出了确切的字符集以及该检查的运行位置。如果某个值设置了凭据、组织或租户、路由或 API 行为相关的标头（例如 `Authorization` 或 `Host`），当它由服务器托管设置下发时，会被视为[需要批准的设置](/docs/zh-CN/server-managed-settings#environment-variables-and-the-approval-dialog)。如果此类值来自项目设置或本地设置，则遵循[`env` 值何时生效的规则](/docs/zh-CN/settings-reference#when-claude-code-applies-env-values) |
| `ANTHROPIC_CUSTOM_MODEL_OPTION` | 要作为自定义条目添加到 `/model` 选择器中的模型 ID。使用此变量可让非标准或网关特定的模型可供选择，而无需替换内置别名。请参阅[模型配置](/docs/zh-CN/model-config#add-a-custom-model-option) |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION` | `/model` 选择器中自定义模型条目的显示描述。未设置时默认为 `Custom model (<model-id>)` |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_NAME` | `/model` 选择器中自定义模型条目的显示名称。未设置时，如果 Claude Code [能识别该 ID](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities)，该条目显示模型名称，否则显示模型 ID |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_SUPPORTED_CAPABILITIES` | 自定义模型所支持的[功能](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities)的逗号分隔列表，例如 `effort,thinking`。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_FABLE_MODEL` | `fable` 别名解析到的模型 ID，也是 Claude Code 在第三方提供商上进行[自动模型回退](/docs/zh-CN/model-config#automatic-model-fallback)时识别为 Fable 模型的 ID。请参阅[模型配置](/docs/zh-CN/model-config#environment-variables) |
| `ANTHROPIC_DEFAULT_FABLE_MODEL_DESCRIPTION` | `/model` 选择器中固定的 Fable 模型的显示描述。未设置时，该行显示以 `Custom Fable model` 开头的默认描述。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_FABLE_MODEL_NAME` | `/model` 选择器中固定的 Fable 模型的显示名称。未设置时，如果 Claude Code 能识别固定的 ID，该行显示模型名称，否则显示固定的 ID。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_FABLE_MODEL_SUPPORTED_CAPABILITIES` | 固定的 Fable 模型所支持的[功能](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities)的逗号分隔列表，例如 `effort,thinking`。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `haiku` 别名解析到的模型 ID，也用于[后台功能](/docs/zh-CN/costs#background-token-usage)。请参阅[模型配置](/docs/zh-CN/model-config#environment-variables) |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL_DESCRIPTION` | `/model` 选择器中固定的 Haiku 模型的显示描述。未设置时，该行显示以 `Custom Haiku model` 开头的默认描述。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME` | `/model` 选择器中固定的 Haiku 模型的显示名称。未设置时，如果 Claude Code 能识别固定的 ID，该行显示模型名称，否则显示固定的 ID。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL_SUPPORTED_CAPABILITIES` | 固定的 Haiku 模型所支持的[功能](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities)的逗号分隔列表，例如 `effort,thinking`。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_MODEL` | 新会话默认启动时使用的模型。需要 Claude Code v2.1.236 或更高版本。请参阅[为新会话设置默认模型](/docs/zh-CN/model-config#set-a-default-model-for-new-sessions) |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `opus` 别名解析到的模型 ID，也是 `opusplan` 在计划模式处于活动状态时使用的模型。请参阅[模型配置](/docs/zh-CN/model-config#environment-variables) |
| `ANTHROPIC_DEFAULT_OPUS_MODEL_DESCRIPTION` | `/model` 选择器中固定的 Opus 模型的显示描述。未设置时，该行显示以 `Custom Opus model` 开头的默认描述。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_OPUS_MODEL_NAME` | `/model` 选择器中固定的 Opus 模型的显示名称。未设置时，如果 Claude Code 能识别固定的 ID，该行显示模型名称，否则显示固定的 ID。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES` | 固定的 Opus 模型所支持的[功能](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities)的逗号分隔列表，例如 `effort,thinking`。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `sonnet` 别名解析到的模型 ID，也是 `opusplan` 在计划模式未处于活动状态时使用的模型。请参阅[模型配置](/docs/zh-CN/model-config#environment-variables) |
| `ANTHROPIC_DEFAULT_SONNET_MODEL_DESCRIPTION` | `/model` 选择器中固定的 Sonnet 模型的显示描述。未设置时，该行显示以 `Custom Sonnet model` 开头的默认描述。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_SONNET_MODEL_NAME` | `/model` 选择器中固定的 Sonnet 模型的显示名称。未设置时，如果 Claude Code 能识别固定的 ID，该行显示模型名称，否则显示固定的 ID。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_DEFAULT_SONNET_MODEL_SUPPORTED_CAPABILITIES` | 固定的 Sonnet 模型所支持的[功能](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities)的逗号分隔列表，例如 `effort,thinking`。请参阅[模型配置](/docs/zh-CN/model-config#customize-pinned-model-display-and-capabilities) |
| `ANTHROPIC_FEDERATION_RULE_ID` | [Workload Identity Federation](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation) 的联合规则 ID。当您将其与 `ANTHROPIC_ORGANIZATION_ID` 一起设置时，Claude Code 会选择联合凭据，其优先级高于您的 `/login` 凭据。请参阅[身份验证优先级](/docs/zh-CN/authentication#authentication-precedence) |
| `ANTHROPIC_FOUNDRY_API_KEY` | 用于 Microsoft Foundry 身份验证的 API 密钥（请参阅 [Microsoft Foundry](/docs/zh-CN/microsoft-foundry)） |
| `ANTHROPIC_FOUNDRY_AUTH_TOKEN` | 用于 Microsoft Foundry 身份验证的 Bearer 令牌，例如 Microsoft Entra 访问令牌。Claude Code 将其作为 `Authorization: Bearer` 标头发送。优先于 `ANTHROPIC_FOUNDRY_API_KEY` 和 Azure 默认凭据链。请参阅 [Microsoft Foundry](/docs/zh-CN/microsoft-foundry)。需要 Claude Code v2.1.203 或更高版本 |
| `ANTHROPIC_FOUNDRY_BASE_URL` | Microsoft Foundry 资源的完整基础 URL（例如 `https://my-resource.services.ai.azure.com/anthropic`）。可替代 `ANTHROPIC_FOUNDRY_RESOURCE`（请参阅 [Microsoft Foundry](/docs/zh-CN/microsoft-foundry)） |
| `ANTHROPIC_FOUNDRY_RESOURCE` | Microsoft Foundry 资源名称（例如 `my-resource`）。Claude Code [会拒绝 URL 或主机名](/docs/zh-CN/errors#anthropic-foundry-resource-must-be-a-foundry-resource-name)。如果未设置 `ANTHROPIC_FOUNDRY_BASE_URL`，则为必需（请参阅 [Microsoft Foundry](/docs/zh-CN/microsoft-foundry)） |
| `ANTHROPIC_MODEL` | 要使用的模型设置的名称（请参阅[模型配置](/docs/zh-CN/model-config#environment-variables)） |
| `ANTHROPIC_ORGANIZATION_ID` | [Workload Identity Federation](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation) 的组织 ID。请将其与 `ANTHROPIC_FEDERATION_RULE_ID` 一起设置。请参阅[身份验证优先级](/docs/zh-CN/authentication#authentication-precedence) |
| `ANTHROPIC_PROFILE` | 用于身份验证的 Anthropic 配置文件名称，例如由 [`ant auth login`](https://platform.claude.com/docs/en/cli-sdks-libraries/cli/authentication) 或[在没有 API 密钥的情况下登录 Console 账户](/docs/zh-CN/authentication#sign-in-without-an-api-key)所创建的配置文件。请参阅[身份验证优先级](/docs/zh-CN/authentication#authentication-precedence) |
| `ANTHROPIC_SMALL_FAST_MODEL` | \[已弃用] [用于后台任务的 Haiku 级模型](/docs/zh-CN/costs)的名称 |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | 使用 Amazon Bedrock 或 Amazon Bedrock Mantle 时，覆盖 Haiku 级模型的 AWS 区域。在 Amazon Bedrock 上，仅当同时设置了 `ANTHROPIC_DEFAULT_HAIKU_MODEL` 或已弃用的 `ANTHROPIC_SMALL_FAST_MODEL` 时才会生效，因为否则 Amazon Bedrock 会在会话所在区域使用[默认 Sonnet 模型或主模型](/docs/zh-CN/amazon-bedrock#4-pin-model-versions)运行后台任务 |
| `ANTHROPIC_VERTEX_BASE_URL` | 覆盖 Google Cloud's Agent Platform 端点 URL。用于自定义 Google Cloud's Agent Platform 端点或通过 [LLM 网关](/docs/zh-CN/llm-gateway)路由时。请参阅 [Google Cloud's Agent Platform](/docs/zh-CN/google-vertex-ai) |
| `ANTHROPIC_VERTEX_PROJECT_ID` | Google Cloud's Agent Platform 请求所发往的 GCP 项目 ID。请参阅[配置 GCP 凭据](/docs/zh-CN/google-vertex-ai#3-configure-gcp-credentials) |
| `ANTHROPIC_WORKSPACE_ID` | [工作负载身份联合](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation)的工作区 ID。当您的联合规则的范围涵盖多个工作区时设置此项，以便令牌交换知道要以哪个工作区为目标 |
| `API_FORCE_IDLE_TIMEOUT` | 覆盖 5 分钟的响应体空闲超时，该超时会在没有字节到达时中止流式模型响应。设置为 `0` 可关闭该超时，例如当缓慢的[网关](/docs/zh-CN/llm-gateway)或本地模型在数据块之间暂停超过 5 分钟时；设置为 `1` 可对所有提供商保持启用。未设置时，该超时在直连 Anthropic API、[Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 以及设置了 `CLAUDE_ENABLE_BYTE_WATCHDOG_BEDROCK=1` 的 Amazon Bedrock 以外的提供商上生效。[流式看门狗](/docs/zh-CN/network-config#streaming-idle-watchdogs)独立于它运行，即使您在此处设置了 `0`，它们也会中止长时间的静默暂停 |
| `API_TIMEOUT_MS` | API 请求的超时时间，以毫秒为单位（默认值：600000，即 10 分钟；最大值：2147483647）。当请求在慢速网络上超时或通过代理路由时，请增大此值。超过最大值的值会使底层计时器溢出，导致请求立即失败 |
| `AWS_BEARER_TOKEN_BEDROCK` | 用于身份验证的 Amazon Bedrock API 密钥（请参阅 [Amazon Bedrock API 密钥](https://aws.amazon.com/blogs/machine-learning/accelerate-ai-development-with-amazon-bedrock-api-keys/)） |
| `BASH_DEFAULT_TIMEOUT_MS` | 前台 Bash 或 PowerShell 工具命令的默认超时时间，以毫秒为单位（默认值：120000，即 2 分钟）。超过 30 分钟的默认值还会成为无人值守会话中[后台命令的默认时间限制](/docs/zh-CN/tools-reference#time-limit-for-background-commands)。后台时间限制需要 Claude Code v2.1.285 或更高版本 |
| `BASH_MAX_OUTPUT_LENGTH` | Claude Code 读回到命令结果中的 bash 输出的最大字符数（默认值：30000；最大值：150000）。如果您设置了 [`bashOutputMaxChars`](/docs/zh-CN/settings-reference#bashoutputmaxchars) 设置，Claude Code 会忽略此变量。请参阅[输出限制](/docs/zh-CN/tools-reference#output-limits) |
| `BASH_MAX_TIMEOUT_MS` | 模型可以为前台 Bash 或 PowerShell 工具命令设置的最大超时时间，以毫秒为单位（默认值：600000，即 10 分钟）。有效上限为此值与 `BASH_DEFAULT_TIMEOUT_MS` 中的较大者。超过 2 小时的有效上限还会成为无人值守会话中[后台命令的最大时间限制](/docs/zh-CN/tools-reference#time-limit-for-background-commands)。后台时间限制需要 Claude Code v2.1.285 或更高版本 |
| `BETA_TRACING_ENDPOINT` | [详细 beta 追踪](/docs/zh-CN/monitoring-usage#traces-beta)的 OTLP/HTTP 端点：设置 `ENABLE_BETA_TRACING_DETAILED=1` 后，日志和追踪数据会发送到此处，而不是发送到已配置的导出器。请在您的 shell、用户设置或托管设置中设置它。在[项目设置和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中会被忽略 |
| `CCR_FORCE_BUNDLE` | 设置为 `1` 可强制 [`claude --cloud`](/docs/zh-CN/claude-code-on-the-web#send-local-repositories-without-github) 打包并上传您的本地仓库，而不是从其远程仓库克隆 |
| `CLAUDECODE` | 在 Claude Code 生成的子进程（Bash 和 PowerShell 工具、tmux 会话、[hook](/docs/zh-CN/hooks) 命令、[状态栏](/docs/zh-CN/statusline)命令、stdio [MCP 服务器](/docs/zh-CN/mcp)子进程）中设置为 `1`。IDE 扩展也会在其集成终端中设置此变量。用于检测脚本是否正在 Claude Code 生成的子进程中运行。要检查当前进程是否由工具调用或 hook 直接生成，而不是在 Claude Code 启动的 stdio MCP 服务器内部运行，请改用 `CLAUDE_CODE_CHILD_SESSION` |
| `CLAUDE_AFK_COUNTDOWN_MS` | 未回答的 [`AskUserQuestion`](/docs/zh-CN/tools-reference) 对话框在自动继续之前多少毫秒显示屏幕倒计时。默认值 `20000`（20 秒），上限为自动继续超时时间。除非启用了自动继续，否则不起作用；请参阅 [`askUserQuestionTimeout`](/docs/zh-CN/settings-reference#askuserquestiontimeout) 设置和 `CLAUDE_AFK_TIMEOUT_MS`。需要 Claude Code v2.1.198 或更高版本 |
| `CLAUDE_AFK_TIMEOUT_MS` | 未回答的 [`AskUserQuestion`](/docs/zh-CN/tools-reference) 对话框在空闲多少毫秒后无需您介入即自动继续。自动继续默认处于关闭状态；可通过 [`askUserQuestionTimeout`](/docs/zh-CN/settings-reference#askuserquestiontimeout) 设置选择启用。此变量是用于演示和自动化测试的覆盖项：设置后，它优先于该设置，即使该设置未设置或为 `never`，也会启用自动继续。设置 `0` 不会关闭超时，而是会立即关闭对话框。在 v2.1.198 和 v2.1.199 中，自动继续默认处于启用状态，超时时间为 `60000`（60 秒）。需要 Claude Code v2.1.198 或更高版本 |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | 设置为 `1` 可禁用所有内置[子代理](/docs/zh-CN/sub-agents)类型，例如 Explore 和 Plan。仅适用于非交互模式（`-p` 标志）。适用于希望从空白状态开始的 SDK 用户。这还会移除 `general-purpose`，即 Agent 工具调用省略 `subagent_type` 时 Claude Code 运行的子代理。此类调用随后会失败，并显示 [`subagent_type is required`](/docs/zh-CN/errors#subagent-type-is-required) |
| `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` | 设置为 `1` 可跳过 SDK 创建的 MCP 服务器中工具名称的 `mcp__<server>__` 前缀。工具使用其原始名称。仅用于 SDK |
| `CLAUDE_ASYNC_AGENT_STALL_TIMEOUT_MS` | 子代理的停滞超时时间，以毫秒为单位。默认值 `600000`（10 分钟）；如果您在流式看门狗启用时调高 `CLAUDE_STREAM_IDLE_TIMEOUT_MS`，默认值也会随之提高，如[处理缓慢或停滞的 API 响应](/docs/zh-CN/agent-sdk/typescript#handle-slow-or-stalled-api-responses)所述。计时器会在每个流式进度事件时重置；如果在该时间窗口内没有收到任何进度，Claude Code 会中止该子代理并向父级报告停滞 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 设置触发自动压缩时自动压缩窗口的百分比（1-100）。使用较低的值（如 `50`）可更早压缩；此变量无法提高阈值，因此高于默认百分比的值会被忽略。它仅适用于[在达到模型上下文限制之前压缩](/docs/zh-CN/model-config#context-window-and-auto-compaction)的会话。同时适用于主对话和子代理 |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | 设置为 `1` 可强制启用长时间运行的 Agent 任务的自动后台运行。启用后，子代理在运行约两分钟后会被移至后台。在 Claude Code v2.1.212 或更高版本中，还会在非交互模式下启用[长时间 MCP 工具调用的自动后台运行](/docs/zh-CN/mcp#automatic-backgrounding-of-long-tool-calls) |
| `CLAUDE_AX_PREPARK_MS` | 在[屏幕阅读器模式](/docs/zh-CN/accessibility)下，Claude Code 在写入新行或已更改的行之前等待的毫秒数。默认值 `0`，因此 Claude Code 不会等待。在 v2.1.287 之前，默认值为 `50`。Claude Code 将等待时间上限设为 `5000`。需要 Claude Code v2.1.233 或更高版本 |
| `CLAUDE_AX_SCREEN_READER` | 设置为 `1` 可渲染适合屏幕阅读器的输出：不带装饰性边框或动画的纯文本。设置为 `0` 可强制关闭屏幕阅读器模式，即使 [`axScreenReader`](/docs/zh-CN/settings-reference#axscreenreader) 为 `true`。[`--ax-screen-reader`](/docs/zh-CN/cli-reference#cli-flags) 标志优先。需要 Claude Code v2.1.181 或更高版本 |
| `CLAUDE_AX_STARTUP_QUIET_MS` | 在[屏幕阅读器模式](/docs/zh-CN/accessibility)下，Claude Code 在启动确认行之后暂缓首次界面渲染的毫秒数，以便您的屏幕阅读器能在新输出打断之前完整朗读该行。默认值 `3000`。设置 `0` 可立即渲染。Claude Code 将暂缓时间上限设为 `600000`（10 分钟）。您的第一次按键会提前结束暂缓。需要 Claude Code v2.1.217 或更高版本 |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | 在主会话中，每条 Bash 或 PowerShell 命令执行后返回原始工作目录 |
| `CLAUDE_BYTE_STREAM_IDLE_TIMEOUT_MS` | 字节级流式空闲看门狗的超时时间，以毫秒为单位；设置后，对于该看门狗，它优先于 `CLAUDE_STREAM_IDLE_TIMEOUT_MS`，且不会改变事件级看门狗。Claude Code 将此变量限制在 10 秒到 30 分钟之间。需要 Claude Code v2.1.210 或更高版本 |
| `CLAUDE_CLIENT_PRESENCE_FILE` | 某个文件的路径，外部工具（例如锁屏监听器）会在您解锁屏幕时创建该文件，并在您锁屏时删除它。该文件存在期间，Claude Code 会跳过 [Remote Control 移动推送通知](/docs/zh-CN/remote-control#mobile-push-notifications)，这样您在主动使用计算机时就不会收到推送。当该文件不存在或无法读取时，通知会照常发送。Claude Code 在每个触发推送的事件发生时检查一次该文件，而不是轮询它。需要 Claude Code v2.1.181 或更高版本 |
| `CLAUDE_CODE_ACCESSIBILITY` | 设置为 `1` 可保持原生终端光标可见，并禁用反色文本光标指示器。使 macOS Zoom 等屏幕放大器能够跟踪光标位置 |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | 设置为 `1` 可从通过 `--add-dir` 指定的目录加载记忆文件。加载 `CLAUDE.md`、`.claude/CLAUDE.md`、`.claude/rules/*.md` 和 `CLAUDE.local.md`。默认情况下，附加目录不会加载记忆文件 |
| `CLAUDE_CODE_ALT_SCREEN_FULL_REPAINT` | 设置为 `1` 可在[全屏渲染](/docs/zh-CN/fullscreen)中每一帧都重绘整个屏幕，而不是发送增量更新。如果全屏模式显示过时或错位的文本片段，请使用此设置。Claude Code 会在 Windows 上为后台会话和 [Agent 视图](/docs/zh-CN/agent-view)自动启用此功能 |
| `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` | 设置为 `1` 可在每个请求中发送 [effort](/docs/zh-CN/model-config#adjust-effort-level) 参数，即使 Claude Code 未将该模型 ID 识别为支持 effort。在通过 [LLM 网关](/docs/zh-CN/llm-gateway)或以自定义标识符提供模型的第三方提供商进行路由时使用此变量。在 API 层面拒绝 effort 参数的模型（包括 Claude 3 模型、Sonnet 4.0 和 4.5、Opus 4.0 和 4.1 以及 Haiku 4.5）仍会被排除在外，以免请求失败 |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | 刷新凭据的时间间隔，以毫秒为单位（使用 [`apiKeyHelper`](/docs/zh-CN/settings-reference#apikeyhelper) 时） |
| `CLAUDE_CODE_ARTIFACT_AUTO_OPEN` | 设置为 `0` 可阻止 Claude Code 在发布新的 [Artifact](/docs/zh-CN/artifacts#create-an-artifact) 时自动打开浏览器 |
| `CLAUDE_CODE_ARTIFACT_COMMENTS` | 设置为 `0` 可阻止 Claude 读取和回复 [Artifact 上的评论](/docs/zh-CN/artifacts#collect-comments-on-an-artifact)。当 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 已[关闭 Artifact](/docs/zh-CN/artifacts#availability) 时不起作用。需要 Claude Code v2.1.221 或更高版本 |
| `CLAUDE_CODE_ARTIFACT_COMMENTS_AUTOREACT` | 设置为 `0` 可阻止 Claude [自行回复发送给它的评论](/docs/zh-CN/artifacts#let-claude-reply-to-comments-on-its-own)。需要 Claude Code v2.1.228 或更高版本 |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | 设置为 `0` 可从系统提示词开头省略[归属块](/docs/zh-CN/llm-gateway-protocol#system-prompt-attribution-block)，该块包含客户端版本和提示词指纹。无论如何设置，直连 Anthropic API 时的缓存都不受影响。在某些直连配置中，即使您设置了 `0`，Claude Code 仍会在[自动模式](/docs/zh-CN/permission-modes#eliminate-prompts-with-auto-mode)分类器请求中保留该块。请在[系统提示词归属块](/docs/zh-CN/llm-gateway-protocol#system-prompt-attribution-block)中查看此情况涵盖哪些连接和凭据。在 v2.1.181 之前，该块在自定义基础 URL 和 Microsoft Foundry 连接上包含一个每个请求各不相同的令牌，因此在这些版本上，当您的 LLM 网关基于请求体进行缓存或将请求转发给第三方提供商时，或者当您直接连接到 Microsoft Foundry 时，请将其设置为 `0` |
| `CLAUDE_CODE_AUTO_BACKGROUND_WORKER_CHECKIN_SECONDS` | 已在 v2.1.283 中移除。请改用 `CLAUDE_CODE_WORKER_CHECKIN_SCHEDULE` |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 以 token 为单位设置[自动压缩窗口](/docs/zh-CN/model-config#set-the-auto-compact-window)，范围为 `100000` 到 `1000000`。仅接受纯整数，例如 `500000`：像 `500k` 这样的值会被读取为 `500`，并被限制为 100K 的最小值。有效窗口还受模型上下文窗口的上限约束。优先于 `/autocompact` 命令、`--autocompact` 标志和 `autoCompactWindow` 设置。状态栏的 `used_percentage` 始终以模型的完整上下文窗口为基准进行衡量，因此一旦设置了此变量，该百分比就不再能指示何时会运行压缩 |
| `CLAUDE_CODE_AUTO_CONNECT_IDE` | 覆盖自动 [IDE 连接](/docs/zh-CN/vs-code)。默认情况下，在受支持 IDE 的集成终端中启动时，Claude Code 会自动连接。设置为 `false` 可阻止此行为。设置为 `true` 可在自动检测失败时（例如 tmux 遮蔽了父终端时）强制尝试连接。优先于 [`autoConnectIde`](/docs/zh-CN/settings-reference#autoconnectide) 全局配置设置 |
| `CLAUDE_CODE_AUTO_MODE_SERVER` | 控制 Claude Code 是否请求服务器[审查自动模式操作](/docs/zh-CN/permission-modes#server-side-classifier-review)。设置为 `0` 可改用 Claude Code 自己的分类器请求。在直连 Anthropic API 时，需要 v2.1.281 或更高版本。链接的章节列出了未设置该变量时哪些会话会请求服务器，以及从哪个版本开始。需要 Claude Code v2.1.271 或更高版本 |
| `CLAUDE_CODE_AWS_CHAIN_RESOLVE_TIMEOUT_MS` | Claude Code 等待 AWS 默认凭据提供程序链生成凭据的时间（以毫秒为单位），超时后请求会失败并显示 [`AWS default-chain credential resolve timed out`](/docs/zh-CN/errors#aws-default-chain-credential-resolve-timed-out)（默认值：`60000`）。当凭据链中的某个步骤确实需要更长时间时（例如通过 `aws-vault` 之类的包装器进行带 MFA 的基于浏览器的 SSO 登录），请调高此值。适用于 Claude Code 使用默认链进行签名的所有场景：[Amazon Bedrock](/docs/zh-CN/amazon-bedrock#credential-caching-and-resolution-timeout)、[Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 和 [Mantle 端点](/docs/zh-CN/amazon-bedrock#use-the-mantle-endpoint)。需要 Claude Code v2.1.207 或更高版本 |
| `CLAUDE_CODE_BASH_EDIT_DIFF` | 设置为 `0` 可关闭 [Bash 命令运行期间已更改文件的 diff](/docs/zh-CN/hooks#bash)，设置为 `1` 可在所有权限模式下记录该 diff。优先于 [`bashEditDiffEnabled`](/docs/zh-CN/settings-reference#basheditdiffenabled) 设置。需要 Claude Code v2.1.269 或更高版本 |
| `CLAUDE_CODE_BG_TASKS_REPORT_RUNNING` | 设置为 `0` 可让非交互会话在每个轮次结束时向其宿主报告空闲状态，即使后台工作仍在运行。默认情况下，当后台 Agent 或[工作流](/docs/zh-CN/workflows)运行等后台工作仍处于活动状态时，会话在轮次结束后仍会持续报告运行状态。这可以防止监视该状态的宿主（例如远程会话列表）在工作进行中宣布 Claude 正在等待您的输入。后台 shell 命令（例如开发服务器）不会保持运行状态。运行状态这一默认行为以及 `0` 选择退出需要 Claude Code v2.1.269 或更高版本；在更早的版本上，设置 `1` 可保持运行状态 |
| `CLAUDE_CODE_BRIDGE_SESSION_ID` | 当会话具有活动的 [Remote Control](/docs/zh-CN/remote-control) 连接时，在 Bash 工具和 [hook 命令](/docs/zh-CN/hooks)子进程中自动设置，并在连接结束时移除。该值是 `session_` 形式的会话 ID，与会话的 `claude.ai/code` URL 中出现的标识符相同，因此脚本可以链接回运行它的会话。需要 Claude Code v2.1.199 或更高版本。在[云端会话](/docs/zh-CN/claude-code-on-the-web)中，请改为读取 `CLAUDE_CODE_REMOTE_SESSION_ID` |
| `CLAUDE_CODE_BS_AS_CTRL_BACKSPACE` | 设置为 `0` 可让 Claude Code 将 `0x08` 字节（也写作 `^H`）读取为普通 Backspace，设置为 `1` 可将其读取为 Ctrl+Backspace。任一值都会替换平台默认行为。默认情况下，Claude Code 在 Windows 上将其读取为 Ctrl+Backspace（`TERM_PROGRAM` 为 `mintty` 或 `TERM` 为 `cygwin` 时除外），在 macOS 和 Linux 上将其读取为普通 Backspace。在 [Backspace 会删除整个单词](/docs/zh-CN/terminal-config#fix-backspace-deleting-a-whole-word-on-windows)的 Windows 终端中，请设置 `0` |
| `CLAUDE_CODE_CERT_STORE` | TLS 连接所用 CA 证书来源的逗号分隔列表。`bundled` 是 Claude Code 附带的 Mozilla CA 集。`system` 是操作系统信任存储，仅在具有 `tls.getCACertificates` 的运行时上读取：原生二进制文件，或通过 npm 安装时的 Node 22.15 或更高版本。请参阅 [CA 证书存储](/docs/zh-CN/network-config#ca-certificate-store)。默认值为 `bundled,system` |
| `CLAUDE_CODE_CHILD_SESSION` | 在 Claude Code 通过 Bash、PowerShell 和 Monitor 工具、[hook](/docs/zh-CN/hooks) 命令以及[状态栏](/docs/zh-CN/statusline)命令生成的子进程中设置为 `1`。不会为 stdio [MCP 服务器](/docs/zh-CN/mcp)子进程设置，因为它们是长期运行的，生命周期长于生成它们的会话。与 `CLAUDECODE` 不同，此变量仅由 Claude Code 本身在启动子进程时设置，而不会由 IDE 扩展设置，因此它能可靠地区分嵌套会话与在 IDE 集成终端中启动的顶层 `claude`。以这种方式启动的嵌套交互式 `claude` TUI 会被自动排除在 `--resume`、`--continue`、向上箭头历史记录和 `claude agents` 列表之外。非交互的 `claude -p` 会话仍会持久保存。设置 `CLAUDE_CODE_FORCE_SESSION_PERSISTENCE=1` 可覆盖此排除行为。需要 Claude Code v2.1.172 或更高版本 |
| `CLAUDE_CODE_CLIENT_CERT` | 用于 mTLS 身份验证的客户端证书文件路径 |
| `CLAUDE_CODE_CLIENT_KEY` | 用于 mTLS 身份验证的客户端私钥文件路径 |
| `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` | 加密的 CLAUDE\_CODE\_CLIENT\_KEY 的密码短语（可选） |
| `CLAUDE_CODE_CONNECT_TIMEOUT_MS` | 已在 v2.1.186 中移除，现在不起任何作用。以前用于为流式 API 请求的连接、TLS 和响应标头阶段单独设置超时时间。请使用 `API_TIMEOUT_MS` 设置每个请求的超时时间。关于流式请求的响应标头阶段，请参阅 `CLAUDE_STREAM_FIRST_BYTE_TIMEOUT_MS` |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | 覆盖调试日志文件路径。尽管名称如此，这是一个文件路径，而不是目录。需要通过 `--debug`、`/debug` 或 `DEBUG` 环境变量单独启用调试模式：仅设置此变量不会启用日志记录。[`--debug-file`](/docs/zh-CN/cli-reference#cli-flags) 标志可同时完成这两项。默认为 `~/.claude/debug/<session-id>.txt` |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | 写入调试日志文件的最低日志级别。取值：`verbose`、`debug`（默认）、`info`、`warn`、`error`。设置为 `verbose` 可包含大量诊断信息（例如完整的状态栏命令输出），或提高到 `error` 以减少干扰信息 |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | 设置为 `1` 可禁用 [1M 上下文窗口](/docs/zh-CN/model-config#extended-context)支持。设置后，1M 模型变体在模型选择器中不可用，并且对于使用原生 1M 窗口的模型（例如 [Sonnet 5.5](/docs/zh-CN/model-config#sonnet-5-5-and-sonnet-5-context-window) 和 Fable 模型），Claude Code 会将其上的会话限制在 200K 窗口；关于如何强制执行此限制，请参阅[扩展上下文](/docs/zh-CN/model-config#extended-context)。适用于有合规要求的企业环境。关于它在为无法识别的 `[1m]` 模型 ID 校正窗口方面的作用，请参阅[为网关或自定义模型 ID 校正窗口](/docs/zh-CN/model-config#correct-the-window-for-a-gateway-or-custom-model-id) |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | 设置为 `1` 可在 Opus 4.6 和 Sonnet 4.6 上禁用[自适应推理](/docs/zh-CN/model-config#adjust-effort-level)，并回退到由 `MAX_THINKING_TOKENS` 控制的固定思考预算。对 [Fable 模型](/docs/zh-CN/model-config#extended-thinking)、Sonnet 5 及更高版本或 Opus 4.7 及更高版本没有影响，这些模型始终使用自适应推理 |
| `CLAUDE_CODE_DISABLE_ADMIN_ENV_UNION` | 设置为 `1` 可阻止 Claude Code 跨管理员来源按键合并[托管设置](/docs/zh-CN/managed-settings#precedence-within-the-managed-tier)的 `env` 块，这样只会应用最高优先级来源的整个 `env` 块，与 v2.1.223 之前的行为相同。请在启动 Claude Code 的环境中设置它，因为 Claude Code 会忽略通过设置的 `env` 块下发的副本。需要 Claude Code v2.1.223 或更高版本 |
| `CLAUDE_CODE_DISABLE_ADVISOR_TOOL` | 设置为 `1` 可禁用 [advisor 工具](/docs/zh-CN/advisor)。`/advisor` 命令将不可用，任何已配置的 `advisorModel` 都会被忽略，`--advisor` 标志会被接受但不起作用，因此传递该标志的现有脚本可继续正常运行而不会报错 |
| `CLAUDE_CODE_DISABLE_AGENT_VIEW` | 设置为 `1` 可关闭[后台 Agent 和 Agent 视图](/docs/zh-CN/agent-view)：`claude agents`、`--bg`、`/background` 以及按需启动的监管进程。等同于 [`disableAgentView`](/docs/zh-CN/settings-reference#disableagentview) 设置 |
| `CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN` | 设置为 `1` 可禁用[全屏渲染](/docs/zh-CN/fullscreen)并使用经典的主屏幕渲染器。对话会保留在终端的原生回滚缓冲区中，因此 `Cmd+f` 和 tmux 复制模式可照常工作。优先于 `CLAUDE_CODE_NO_FLICKER` 和 [`tui`](/docs/zh-CN/settings-reference#tui) 设置。您也可以使用 `/tui default` 进行切换。不适用于从 [Agent 视图](/docs/zh-CN/agent-view)打开的后台会话，这些会话始终使用全屏渲染 |
| `CLAUDE_CODE_DISABLE_ARTIFACT` | 设置为 `1` 可关闭 [Artifact](/docs/zh-CN/artifacts) 工具，该工具会将会话输出作为私有网页发布到 claude.ai 上。设置后，任何设置文件都无法重新启用该工具。如要改为通过设置文件关闭该工具，请将 [`enableArtifact`](/docs/zh-CN/settings-reference#enableartifact) 设置为 `false`；已弃用的 [`disableArtifact`](/docs/zh-CN/settings-reference#disableartifact) 键也可将其关闭 |
| `CLAUDE_CODE_DISABLE_ATTACHMENTS` | 设置为 `1` 可禁用附件处理。使用 `@` 语法的文件提及会以纯文本形式发送，而不会展开为文件内容 |
| `CLAUDE_CODE_DISABLE_AUTH_REFRESH_LOCK` | 设置为 `1` 可让 Claude Code 进程自行运行其 [`gcpAuthRefresh`](/docs/zh-CN/settings-reference#gcpauthrefresh) 或 [`awsAuthRefresh`](/docs/zh-CN/settings-reference#awsauthrefresh) 命令，而不是在另一个进程运行该命令时等待。需要 Claude Code v2.1.286 或更高版本 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 设置为 `1` 可禁用[自动记忆](/docs/zh-CN/memory#auto-memory)。设置为 `0` 可强制启用自动记忆，即使 `--bare` 模式或 [`autoMemoryEnabled: false`](/docs/zh-CN/settings-reference#automemoryenabled) 原本会禁用它。禁用后，Claude 不会创建或加载自动记忆文件 |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 设置为 `1` 可禁用所有后台任务功能，包括 Bash 和子代理工具上的 `run_in_background` 参数、自动后台运行以及 Ctrl+B 快捷键 |
| `CLAUDE_CODE_DISABLE_BEDROCK_CONTENT_TYPE_DEFAULT` | 设置为 `1` 可阻止 Claude Code 将缺少 `Content-Type` 标头或该标头为空的 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock) 流式响应视为 Amazon Bedrock 的二进制事件流。默认情况下，Claude Code 会假定是网关从一个其他方面未经修改的响应中丢弃了该标头，因此它会解码响应体，流式传输得以继续工作。仅当网关还会将流重新以服务器发送事件的形式输出时才设置此项；此时 Claude Code 会将没有该标头的响应体作为服务器发送事件读取。需要 Claude Code v2.1.239 或更高版本 |
| `CLAUDE_CODE_DISABLE_BEDROCK_CONTENT_TYPE_GUARD` | 设置为 `1` 可跳过对 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock) 流式响应是否带有 `application/vnd.amazon.eventstream` content-type 的检查。没有此变量时，如果响应带有不同的 content-type，Claude Code 会使请求失败，并显示指明该类型的错误，这意味着[网关或代理正在转换响应](/docs/zh-CN/amazon-bedrock#streaming-errors-behind-a-gateway-or-proxy)。请将网关配置为原样转发 `Content-Type` 标头和响应体，而不是设置此变量。需要 Claude Code v2.1.208 或更高版本 |
| `CLAUDE_CODE_DISABLE_BG_EXIT_HANDOFF` | 设置为 `1` 后，当[监管进程](/docs/zh-CN/agent-view#the-supervisor-process)停止、重启或更新[后台会话](/docs/zh-CN/agent-view)的进程时，会停止该会话正在运行的后台 shell 命令、动态工作流，以及（从 v2.1.198 开始）后台子代理，而不是将它们移交给该会话的下一个进程。仅影响此移交：使用 `←` 或 [`/background`](/docs/zh-CN/agent-view#from-inside-a-session) 将会话转入后台时，仍会延续进行中的工作，而 `CLAUDE_DISABLE_ADOPT` 会同时关闭这两者。需要 Claude Code v2.1.196 或更高版本 |
| `CLAUDE_CODE_DISABLE_BG_SHELL_PRESSURE_REAP` | 设置为 `1` 可阻止 Claude Code 在内存压力下终止[后台 shell 命令](/docs/zh-CN/interactive-mode#background-bash-commands)。默认情况下，在 macOS 和 Linux 上，当操作系统报告严重内存压力，且会话已空闲 30 分钟、没有轮次或子代理在运行时，Claude Code 会终止后台 shell。Windows 没有内存压力信号，因此此变量在 Windows 上不起作用。需要 Claude Code v2.1.193 或更高版本 |
| `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS` | 设置为 `1` 可禁用 Claude Code 随附的 [skill](/docs/zh-CN/skills) 和工作流：随附 skill 和工作流会被完全移除，而 `/init` 等内置命令仍可输入，但对模型隐藏。`/doctor` 与内置命令一样仍可输入；请改用 `DISABLE_DOCTOR_COMMAND` 将其隐藏。来自插件、`.claude/skills/` 和 `.claude/commands/` 的 skill 不受影响。等同于 [`disableBundledSkills`](/docs/zh-CN/settings-reference#disablebundledskills) 设置 |
| `CLAUDE_CODE_DISABLE_CFC_PROMPT` | 设置为 `1` 可保留 [Claude in Chrome](/docs/zh-CN/chrome) 浏览器工具，同时省略系统提示词中的 Chrome 部分以及 `/claude-in-chrome` [随附 skill](/docs/zh-CN/skills#bundled-skills)。适用于嵌入 Claude Code 并提供自有浏览器指导的宿主。需要 Claude Code v2.1.257 或更高版本 |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | 设置为 `1` 可阻止将任何 CLAUDE.md 记忆文件加载到上下文中，包括用户、项目和自动记忆文件 |
| `CLAUDE_CODE_DISABLE_CRON` | 设置为 `1` 可禁用[定时任务](/docs/zh-CN/scheduled-tasks)。`/loop` skill 和 cron 工具将不可用，所有已安排的任务都会停止触发，包括在会话中途已在运行的任务 |
| `CLAUDE_CODE_DISABLE_DANGEROUS_RM_TIMEOUT` | 设置为 `1` 可关闭[关键路径删除](/docs/zh-CN/permission-modes#critical-paths)提示的时间限制。之后，在 `auto` 模式下，Claude Code 会将这些删除操作发送给分类器；在 `bypassPermissions` 模式下，提示会一直等待您的回答。请在启动 Claude Code 的环境中设置它，因为 Claude Code 会忽略通过设置的 `env` 块下发的副本。需要 Claude Code v2.1.281 或更高版本 |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | 设置为 `1` 可从 API 请求中去除预发布的 `anthropic-beta` 请求标头、与之配对的请求体字段，以及 `defer_loading` 和 `eager_input_streaming` 等 beta 工具架构字段。当代理网关因 `anthropic-beta` 标头报告 `Unexpected value(s)` 错误或报告 `Extra inputs are not permitted` 错误而拒绝请求时，请使用此变量。[禁用预发布功能](/docs/zh-CN/llm-gateway-protocol#disable-pre-release-capabilities)列出了此变量移除的内容（包括 [MCP 工具搜索](/docs/zh-CN/mcp#scale-with-mcp-tool-search)）以及 Claude Code 仍会继续发送的内容 |
| `CLAUDE_CODE_DISABLE_EXPLORE_PLAN_AGENTS` | 设置为 `1` 可禁用内置的 [Explore 和 Plan 子代理](/docs/zh-CN/sub-agents#built-in-subagents)。Claude 会改用其搜索工具或通用子代理进行探索，[计划模式](/docs/zh-CN/permission-modes#analyze-before-you-edit-with-plan-mode)会直接读取文件，而不是启动 Explore 和 Plan Agent。名为 `Explore` 或 `Plan` 的自定义子代理不受影响。要在 Agent SDK 或非交互模式中移除所有内置子代理类型，请改用 `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS`。需要 Claude Code v2.1.198 或更高版本 |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | 设置为 `1` 可禁用[快速模式](/docs/zh-CN/fast-mode) |
| `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY` | 设置为 `1` 可禁用“How is Claude doing?”会话质量调查。当设置了 `DISABLE_TELEMETRY`、`DO_NOT_TRACK` 或 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 时，调查也会被禁用，除非 `CLAUDE_CODE_ENABLE_FEEDBACK_SURVEY_FOR_OTEL` 重新选择启用。要设置抽样率而不是完全禁用，请使用 [`feedbackSurveyRate`](/docs/zh-CN/settings-reference#feedbacksurveyrate) 设置。请参阅[会话质量调查](/docs/zh-CN/data-usage#session-quality-surveys) |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | 设置为 `1` 可禁用文件[检查点功能](/docs/zh-CN/checkpointing)。`/rewind` 命令将无法恢复代码更改。覆盖 [`fileCheckpointingEnabled`](/docs/zh-CN/settings-reference#filecheckpointingenabled) 设置 |
| `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS` | 设置为 `1` 可从 Claude 的上下文中移除内置的提交和 PR 工作流说明以及 git 状态快照。在使用您自己的 git 工作流 skill 时很有用。设置后，优先于 [`includeGitInstructions`](/docs/zh-CN/settings-reference#includegitinstructions) 设置 |
| `CLAUDE_CODE_DISABLE_INLINE_SHELL_RM_PROMPT` | 设置为 `1` 可阻止 Claude Code 读取通过 `-c` 传递给 shell 的脚本（例如 `bash -c 'rm -rf ~'`）来检查[关键路径](/docs/zh-CN/permission-modes#removals-inside-nested-commands-and-inline-scripts)删除。Claude Code 仍会检查这些脚本中 shell 变量和位置参数形式的目标，其他关键路径检查也会继续运行。请在启动 Claude Code 的环境中设置它，因为 Claude Code 会忽略通过设置的 `env` 块下发的副本。需要 Claude Code v2.1.288 或更高版本 |
| `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` | 设置为 `1` 可阻止在 Anthropic API 上将 Opus 4.0 和 4.1 自动重新映射到当前 Opus 版本。在您有意固定使用较旧的模型时使用。该重新映射不会在 Amazon Bedrock、Google Cloud's Agent Platform 或 Microsoft Foundry 上运行 |
| `CLAUDE_CODE_DISABLE_MODEL_ACCESS_FALLBACK` | 设置为 `1` 后，当您的账户在会话中途失去对该会话模型的访问权限时，[Amazon Bedrock](/docs/zh-CN/amazon-bedrock#when-a-model-is-disabled-mid-session) 和 [Google Cloud's Agent Platform](/docs/zh-CN/google-vertex-ai#when-a-model-is-disabled-mid-session) 上的 Claude Code 不会切换到较旧的模型，被拒绝的请求会立即失败。您配置的[备用模型链](/docs/zh-CN/model-config#fallback-model-chains)仍会在发生该拒绝时切换，[启动时模型检查](/docs/zh-CN/amazon-bedrock#startup-model-checks)也仍会在启动时回退。需要 Claude Code v2.1.285 或更高版本 |
| `CLAUDE_CODE_DISABLE_MOUSE` | 设置为 `1` 可在[全屏渲染](/docs/zh-CN/fullscreen)中禁用鼠标跟踪。使用 `PgUp` 和 `PgDn` 的键盘滚动仍然有效。使用此设置可保留终端原生的选中即复制行为 |
| `CLAUDE_CODE_DISABLE_MOUSE_CLICKS` | 设置为 `1` 可在[全屏渲染](/docs/zh-CN/fullscreen)中禁用点击、拖动和悬停处理，同时保留鼠标滚轮滚动。当您希望滚轮滚动在 Claude Code 中正常工作，但不希望点击定位光标、展开工具输出或打开链接时，请使用此设置。同时设置时，`CLAUDE_CODE_DISABLE_MOUSE` 优先。需要 Claude Code v2.1.195 或更高版本 |
| `CLAUDE_CODE_DISABLE_MTLS_RELOAD_ON_STALE_CONNECTION` | 设置为 `1` 可阻止 Claude Code 在 API 请求因连接级错误（例如连接重置或 TLS 握手错误）而失败时重新读取 [mTLS 客户端证书和密钥](/docs/zh-CN/network-config#mtls-authentication)。禁用重新加载后，Claude Code 仅在下次应用设置时或下次启动时加载轮换后的文件。需要 Claude Code v2.1.232 或更高版本 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 设置为任何非空值（例如 `1`）可禁用非必要的网络流量：自动更新、遥测、错误报告、`/feedback` 命令、[Claude 起草的反馈](/docs/zh-CN/tools-reference#sendfeedback-tool-behavior)、发行说明、[PR 和 MR 状态徽章](/docs/zh-CN/interactive-mode#pr-review-status)检查，以及[快速模式](/docs/zh-CN/fast-mode#use-fast-mode-behind-proxies-and-llm-gateways)检查等可用性检查。它还会停止[插件 `command` 来源的后台运行](/docs/zh-CN/plugins/loading#when-a-command-source-re-runs)，这些是本地命令而不是网络流量，但由于它们可能触发依赖安装，因此也会被停止。**将其设置为 `0` 或 `false` 仍会禁用此类流量**，这与大多数开关变量不同；要重新允许此类流量，请取消设置该变量。还会禁用功能标志获取，这会使 [Remote Control](/docs/zh-CN/remote-control#requirements) 以及其他[需要获取功能标志的功能](#features-that-need-feature-flag-fetching)不可用。官方插件市场自动安装不在此范围内；请使用 `CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL` 将其禁用。不影响[网关模型发现](/docs/zh-CN/llm-gateway-connect#add-gateway-models-to-the-model-picker)，后者有其自己的选择启用方式 |
| `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` | 设置为 `1` 可在流式请求中途失败时禁用非流式回退。流式错误会改为传播到重试层。当代理或网关导致回退产生重复的工具执行时很有用 |
| `CLAUDE_CODE_DISABLE_NOTIFICATION_PRESENCE_CHECK` | 设置为 `1` 可让 `PushNotification` 工具在您正在终端中输入或终端处于焦点状态时仍然发送桌面通知。默认情况下，当该工具检测到最近的键盘活动或终端焦点时，会同时跳过桌面通知和[移动推送](/docs/zh-CN/remote-control#mobile-push-notifications)。此变量仅禁用该本地检查，因此服务器在检测到您处于活动状态时仍可抑制移动推送。需要 Claude Code v2.1.193 或更高版本 |
| `CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL` | 设置为 `1` 可禁用官方插件市场的自动注册。Claude Code 会在即将注册该市场时读取此变量，通常是在机器首次交互式启动期间。如果此时已设置该变量，Claude Code 会永久跳过注册。之后取消设置该变量不会撤销此跳过。您可以随时运行 `claude plugin marketplace add anthropics/claude-plugins-official` 来注册该市场 |
| `CLAUDE_CODE_DISABLE_PERMISSION_PROMPT_NOTIFY_HOOKS` | 设置为 `1` 后，在 Claude Code 将权限请求发送到 Agent SDK 的 `canUseTool` 回调的会话中（Claude Desktop 和 VS Code 扩展就是以这种方式托管 Claude Code 的），Claude Code 不会运行您[针对未回答权限请求的 `Notification` hook](/docs/zh-CN/hooks#notification)。在终端会话中不起作用。需要 Claude Code v2.1.233 或更高版本 |
| `CLAUDE_CODE_DISABLE_POLICY_SKILLS` | 设置为 `1` 可跳过从系统范围的托管 skill 目录加载 skill。适用于不应加载运维人员预置 skill 的容器或 CI 会话 |
| `CLAUDE_CODE_DISABLE_POWERSHELL_CMD_RM_DENY` | 设置为 `1` 可关闭 [PowerShell 工具](/docs/zh-CN/tools-reference#powershell-tool)的一项检查，该检查会拒绝在[系统路径](/docs/zh-CN/permission-modes#remove-item-in-powershell)（例如驱动器根目录或您的主目录）上使用 `cmd` 内置命令 `rd`、`rmdir`、`del` 和 `erase`。Claude Code 会忽略设置文件 `env` 块中的此变量。需要 Claude Code v2.1.283 或更高版本 |
| `CLAUDE_CODE_DISABLE_REFUSAL_FALLBACK` | 设置为 `1` 可关闭[安全分类器标记请求时的自动模型切换](/docs/zh-CN/model-config#automatic-model-fallback)，即 [`switchModelsOnFlag`](/docs/zh-CN/settings-reference#switchmodelsonflag) 设置所控制的行为 |
| `CLAUDE_CODE_DISABLE_STRUCTURED_OUTPUTS` | 设置为 `1` 可阻止 Claude Code 发送结构化输出的 `output_config.format` 字段以及与之配对的 `anthropic-beta` 值，适用于其上游会拒绝这些内容的 [LLM 网关](/docs/zh-CN/llm-gateway-protocol#feature-pass-through)。这会保留 [`CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`](/docs/zh-CN/llm-gateway-protocol#disable-pre-release-capabilities) 所关闭的其他预发布功能。需要 Claude Code v2.1.288 或更高版本 |
| `CLAUDE_CODE_DISABLE_SUBSTITUTION_RM_PROMPT` | 设置为 `1` 可关闭对目标完全是命令替换输出的递归 `rm`（例如 `rm -rf "$(pwd)"`）的[关键路径](/docs/zh-CN/permission-modes#critical-paths)检查。其他关键路径检查会继续运行。请在启动 Claude Code 的环境中设置它，因为 Claude Code 会忽略通过设置的 `env` 块下发的副本。需要 Claude Code v2.1.281 或更高版本 |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | 设置为 `1` 可禁用基于对话上下文的终端标题自动更新。这还会跳过用于[生成会话标题](/docs/zh-CN/sessions#name-your-sessions)的后台小型/快速模型请求 |
| `CLAUDE_CODE_DISABLE_THINKING` | 设置为 `1` 可从 API 请求中完全省略 `thinking` 参数。这是针对会拒绝该参数的代理和网关的兼容性选项。在默认进行思考的模型上，省略该参数意味着模型可能仍会思考。要在 Anthropic API 上明确禁用[扩展思考](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)，请改用 `MAX_THINKING_TOKENS=0`。这两个变量都无法在 Opus 5.5、Sonnet 5.5 或 Fable 模型上关闭思考，因为这些模型的思考无法关闭。在[第三方提供商](/docs/zh-CN/third-party-integrations)上，`MAX_THINKING_TOKENS=0` 同样会省略该参数，因此这两个变量在那里的行为相同 |
| `CLAUDE_CODE_DISABLE_UNKNOWN_MODEL_WINDOW_ENFORCEMENT` | 设置为 `1` 可在 Claude Code 无法识别模型 ID（例如 [LLM 网关](/docs/zh-CN/llm-gateway)别名）时跳过主动[自动压缩](/docs/zh-CN/costs#reduce-token-usage)。没有此变量时，Claude Code 会按照它为该 ID 假定的上下文窗口进行压缩。`CLAUDE_CODE_MAX_CONTEXT_TOKENS` 则可以校正所假定的窗口；关于每个变量的适用场景，请参阅[为网关或自定义模型 ID 校正窗口](/docs/zh-CN/model-config#correct-the-window-for-a-gateway-or-custom-model-id)。需要 Claude Code v2.1.223 或更高版本 |
| `CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL` | 设置为 `1` 可在[全屏渲染](/docs/zh-CN/fullscreen)中禁用虚拟滚动，并渲染会话记录中的每条消息。如果在全屏模式下滚动时，本应显示消息的位置出现空白区域，请使用此设置 |
| `CLAUDE_CODE_DISABLE_WEB_FETCH` | 设置为 `1` 可关闭 [WebFetch](/docs/zh-CN/tools-reference#webfetch-tool-behavior) 工具。[WebSearch](/docs/zh-CN/tools-reference#websearch-tool-behavior) 工具仍然可用。需要 Claude Code v2.1.285 或更高版本 |
| `CLAUDE_CODE_DISABLE_WINDOWS_SHELL_LAUNCHER` | 设置为 `1` 可在 Windows 上直接启动 [PowerShell 工具](/docs/zh-CN/tools-reference#powershell-tool)命令，而不是通过 `cmd.exe` 启动器。默认情况下，该启动器可让[在后台运行](/docs/zh-CN/tools-reference#background-commands)的 PowerShell 命令[延续到会话的下一个进程](/docs/zh-CN/agent-view#the-supervisor-process)，例如当您[将会话转入后台](/docs/zh-CN/agent-view#from-inside-a-session)时。如果设置了该变量，转入后台的 PowerShell 命令会在会话进程退出时停止。Bash 命令不受影响。需要 Claude Code v2.1.269 或更高版本 |
| `CLAUDE_CODE_DISABLE_WORKFLOWS` | 设置为 `1` 可禁用[工作流](/docs/zh-CN/workflows#turn-workflows-off)。等同于 [`disableWorkflows`](/docs/zh-CN/settings-reference#disableworkflows) 设置 |
| `CLAUDE_CODE_EFFORT_LEVEL` | 为受支持的模型设置 effort 级别。取值：`low`、`medium`、`high`、`xhigh`、`max`，或 `auto` 以使用模型默认值。可用级别取决于模型。优先于 `--effort`、`/effort` 以及 `modelSettings` 和 `effortLevel` 设置。[`maxEffortLevel`](/docs/zh-CN/settings-reference#maxeffortlevel) 上限仍然适用。请参阅[调整 effort 级别](/docs/zh-CN/model-config#adjust-effort-level) |
| `CLAUDE_CODE_EMIT_SESSION_STATE_EVENTS` | 设置为 `1` 可向消息流中添加携带会话状态的 [`session_state_changed`](/docs/zh-CN/agent-sdk/typescript#sdksessionstatechangedmessage) 消息。需要使用 [Agent SDK](/docs/zh-CN/agent-sdk/overview)，或同时使用 `--print`、`--output-format stream-json` 和 `--verbose` |
| `CLAUDE_CODE_ENABLE_AUTO_MODE` | 仅为与旧版本兼容而接受，不起任何作用。自动模式默认在所有提供商上可用，包括 Amazon Bedrock、Google Cloud's Agent Platform、Microsoft Foundry 以及已登录的 [Claude apps gateway](/docs/zh-CN/claude-apps-gateway) 会话。在 v2.1.158 至 v2.1.206 中，必须将此变量设置为 `1` 才能在这些提供商上使用[自动模式](/docs/zh-CN/permission-modes#eliminate-prompts-with-auto-mode) |
| `CLAUDE_CODE_ENABLE_AWAY_SUMMARY` | 覆盖[会话回顾](/docs/zh-CN/interactive-mode#session-recap)的可用性。设置为 `0` 可强制关闭回顾，无论 `/config` 开关如何设置。当 [`awaySummaryEnabled`](/docs/zh-CN/settings-reference#awaysummaryenabled) 为 `false` 时，设置为 `1` 可强制开启回顾。优先于该设置和 `/config` 开关 |
| `CLAUDE_CODE_ENABLE_BACKGROUND_PLUGIN_REFRESH` | 设置为 `1` 可在[非交互模式](/docs/zh-CN/headless)下后台安装完成后，在轮次边界刷新插件状态。默认关闭，因为刷新会在会话中途更改系统提示词，导致该轮次的[提示缓存](/docs/zh-CN/prompt-caching)失效 |
| `CLAUDE_CODE_ENABLE_FEEDBACK_SURVEY_FOR_OTEL` | 设置为 `1` 可在发往 Anthropic 的非必要流量被阻止时，将“How is Claude doing?”会话质量调查路由到您自己的 [OpenTelemetry 收集器](/docs/zh-CN/monitoring-usage)。调查评分仅作为 OTEL 事件发送到您配置的收集器。在此模式下，不会向 Anthropic 发送任何调查数据。在设置了 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`、`DISABLE_TELEMETRY` 或 `DO_NOT_TRACK` 时适用，否则不起作用。`CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY` 和组织产品反馈策略优先于此变量 |
| `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING` | 控制工具调用输入是否在 Claude 生成时从 API 流式传输。关闭此功能时，较大的工具输入（例如较长的文件写入）只有在 Claude 生成完毕后才会到达，看起来可能像是卡住了。在 Anthropic API 上默认启用。在 Amazon Bedrock 和 Google Cloud's Agent Platform 上，按模型在所部署的容器支持时启用。设置为 `0` 可选择退出。通过 `ANTHROPIC_BASE_URL`、`ANTHROPIC_VERTEX_BASE_URL` 或 `ANTHROPIC_BEDROCK_BASE_URL` 经由代理路由时，设置为 `1` 可强制启用。在 Microsoft Foundry 和[网关](/docs/zh-CN/llm-gateway)连接上默认关闭 |
| `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY` | 当 `ANTHROPIC_BASE_URL` 指向与 Anthropic 兼容的网关（如 LiteLLM、Kong 或内部代理）时，设置为 `1` 可从网关的 `/v1/models` 端点填充 `/model` 选择器。默认关闭，因为否则由共享 API 密钥支持的网关会向每个用户显示该密钥可访问的所有模型。发现的模型仍会按会话接收到的 [`availableModels`](/docs/zh-CN/settings-reference#availablemodels) 允许列表进行过滤；请通过 [MDM 或托管设置文件](/docs/zh-CN/managed-settings#delivery-mechanisms)下发该列表，因为[服务器托管下发在网关配置上不可用](/docs/zh-CN/server-managed-settings#platform-availability) |
| `CLAUDE_CODE_ENABLE_OPUS_4_7_FAST_MODE` | 在 v2.1.142 中移除，当时[快速模式](/docs/zh-CN/fast-mode)的默认模型从 Opus 4.6 改为 Opus 4.7 |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | 设置为 `false` 可关闭提示词建议，即出现在输入框中的灰色预测内容。优先于 [`promptSuggestionEnabled`](/docs/zh-CN/settings-reference#promptsuggestionenabled) 设置，`/config` 中的 **Prompt suggestions** 开关写入的就是该设置。当您的账户接近或达到用量限制时，Claude Code 也会[暂停建议](/docs/zh-CN/interactive-mode#when-claude-code-skips-suggestions)。设置为 `true` 可在达到限制之前保持建议开启。需要 Claude Code v2.1.238 或更高版本。请参阅[提示词建议](/docs/zh-CN/interactive-mode#prompt-suggestions) |
| `CLAUDE_CODE_ENABLE_TASKS` | 选择 Claude Code 在[具有任务跟踪工具的会话](/docs/zh-CN/tools-reference#task-tool-availability)中提供哪些任务跟踪工具。默认情况下，Claude Code 提供 Task 工具 `TaskCreate`、`TaskUpdate`、`TaskGet` 和 `TaskList`。设置为 `0` 可改用旧版 `TodoWrite` 工具。请参阅[任务列表](/docs/zh-CN/interactive-mode#task-list) |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | 设置为 `1` 可启用用于指标和日志记录的 OpenTelemetry 数据收集。配置 OTel 导出器之前必须设置。请在您的 shell、用户设置或托管设置中设置。在[项目和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中会被忽略。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `CLAUDE_CODE_ENABLE_TODO_TOOLS` | 设置为 `1` 可在所有模型上获得任务跟踪工具。如果不设置，Claude Code 默认仅在 [Task 工具可用性](/docs/zh-CN/tools-reference#task-tool-availability)下列出的模型上提供这些工具。`CLAUDE_CODE_ENABLE_TASKS` 仍然决定使用 Task 工具还是 `TodoWrite`。需要 Claude Code v2.1.233 或更高版本 |
| `CLAUDE_CODE_EXIT_AFTER_STOP_DELAY` | 查询循环变为空闲后、自动退出前等待的时间（毫秒）。适用于使用 SDK 模式的自动化工作流和脚本 |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | 设置为 `1` 可启用 [agent team](/docs/zh-CN/agent-teams)。agent team 为实验性功能，默认禁用 |
| `CLAUDE_CODE_EXTRA_BODY` | 要合并到每个 API 请求正文顶层的 JSON 对象。适用于传递 Claude Code 未直接公开的特定于提供商的参数。在 shell 中导出的值也适用于您通过 `claude agents` 或 `--bg` 派发的[后台会话](/docs/zh-CN/agent-view)。在 v2.1.206 之前，后台会话会忽略 shell 导出的值，而使用后台监督进程所继承的副本 |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | 覆盖文件读取的默认 token 限制。当您需要完整读取较大文件时很有用 |
| `CLAUDE_CODE_FORCE_SESSION_PERSISTENCE` | 设置为 `1` 可强制持久化会话记录、提示词历史记录和 `claude agents` 注册，即使此 `claude` 是从另一个 Claude Code 会话内部启动的。当继承的 `CLAUDE_CODE_CHILD_SESSION` 值（例如来自 `screen` 会话，或最初由 Claude Code 的 Bash 工具启动的后台启动器）导致真正的顶层会话被误判为嵌套会话时使用。从 v2.1.178 开始，Claude Code 会自动检测 tmux 的情况并忽略继承的标记，因此 tmux 不再需要此变量。在 v2.1.169 及更早版本中同样有效；在 v2.1.170 和 v2.1.171 中不起作用，因为这两个版本移除了它所覆盖的嵌套会话检测 |
| `CLAUDE_CODE_FORCE_STRIKETHROUGH` | 当您的终端支持删除线但未被自动检测到时（例如通过 SSH 连接且未转发 `TERM_PROGRAM`），设置为 `1` 可强制对 Claude 回复中的 `~~text~~` 进行删除线渲染。否则，未检测到的终端会显示字面的 `~~` 标记，而不是将文本渲染为删除线。需要 Claude Code v2.1.186 或更高版本 |
| `CLAUDE_CODE_FORCE_SYNC_OUTPUT` | 当您的终端支持 DEC 私有模式 2026 [同步输出](https://gist.github.com/christianparpart/d8a62cc1ab659194337d73e399004036)但未被自动检测到时，设置为 `1` 可强制启用该功能。适用于 Emacs `eat` 等实现了 BSU/ESU 但不响应能力探测的模拟器。在 tmux 下不起作用。与切换到[全屏渲染](/docs/zh-CN/fullscreen)的 `CLAUDE_CODE_NO_FLICKER` 不同，此变量不会更改渲染器 |
| `CLAUDE_CODE_FORK_SUBAGENT` | 控制 [fork 模式](/docs/zh-CN/sub-agents#turn-fork-mode-on-or-off)，该模式让 Claude 自行生成[分叉子代理](/docs/zh-CN/sub-agents#fork-the-current-conversation)，默认仅在交互式会话中开启。设置为 `1` 可在 `claude -p` 和 Agent SDK 中也开启该模式，设置为 `0` 可在所有类型的会话中关闭该模式。无论 fork 模式是否开启，您都可以运行 `/subtask`。交互式会话中的默认开启需要 Claude Code v2.1.232 或更高版本；在更早的版本中，请将该变量设置为 `1` 以开启 fork 模式 |
| `CLAUDE_CODE_FORWARD_SUBAGENT_TEXT` | 设置为 `1` 可在 `claude -p --output-format stream-json` 输出中发出[子代理](/docs/zh-CN/sub-agents)的文本和思考块，与 [`--forward-subagent-text`](/docs/zh-CN/cli-reference#cli-flags) 标志的行为相同。当某个 harness 调用 `claude` 且无法自行传递该标志时使用此变量。该标志在使用 stream-json 输出的非交互模式之外会报错退出，而此变量在这些情况下会被忽略，因此在进程范围内设置时，嵌套调用仍可正常工作。需要 Claude Code v2.1.211 或更高版本 |
| `CLAUDE_CODE_GATEWAY_HINT_HEADERS` | 设置为 `1` 可在自定义代理或第三方提供商（如 Amazon Bedrock 或 Claude Platform on AWS）上发送[网关提示标头](/docs/zh-CN/llm-gateway-protocol#gateway-hint-headers)，例如 `x-claude-code-request-class` 和 `x-claude-code-compaction`。设置为 `0` 可在所有连接上停止发送这些标头，包括直接连接到 Anthropic API 的情况，Claude Code 在该情况下默认会发送它们。需要 Claude Code v2.1.273 或更高版本 |
| `CLAUDE_CODE_GATEWAY_MODEL_DISCOVERY_TIMEOUT_MS` | 由 `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY` 开启的[网关模型发现](/docs/zh-CN/llm-gateway-protocol#model-discovery)请求的超时时间（毫秒）（默认值：`3000`）。当您的网关在启动时需要超过三秒才能响应 `/v1/models` 时，请调高此值。仅接受纯数字；`0`、负值和其他写法会保持默认值。需要 Claude Code v2.1.269 或更高版本 |
| `CLAUDE_CODE_GIT_BASH_PATH` | 仅限 Windows：Git Bash 可执行文件（`bash.exe`）的路径。当 Git Bash 已安装但不在您的 PATH 中时使用。如果该路径不存在，或文件名不是 `bash.exe`、`sh.exe`、`bash` 或 `sh`，Claude Code 会忽略该变量，像未设置时一样自动检测 Git Bash，并记录一条使用 `--debug` 时可见的警告。在 v2.1.219 之前，路径不存在时 Claude Code 会在启动时退出，并且会将任何现有文件用作 shell，而不检查它是否为 bash 或 sh。请参阅 [Windows 设置](/docs/zh-CN/setup#set-up-on-windows) |
| `CLAUDE_CODE_GLOB_HIDDEN` | 设置为 `false` 可在 Claude 调用 [Glob 工具](/docs/zh-CN/tools-reference#glob-tool-behavior)时从结果中排除点文件。默认包含。不影响 `@` 文件自动补全、`ls`、Grep 或 Read |
| `CLAUDE_CODE_GLOB_NO_IGNORE` | 设置为 `false` 可使 [Glob 工具](/docs/zh-CN/tools-reference#glob-tool-behavior)遵循 `.gitignore` 模式。默认情况下，Glob 返回所有匹配的文件，包括被 gitignore 忽略的文件。不影响 `@` 文件自动补全，它有自己的 [`respectGitignore` 设置](/docs/zh-CN/settings-reference#respectgitignore) |
| `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` | Glob 工具文件发现的超时时间（秒）。在大多数平台上默认为 20 秒，在 WSL 上默认为 60 秒 |
| `CLAUDE_CODE_GOAL_CHECKIN_MINUTES` | 后台工作可以让活动目标等待多少分钟，超过后 Claude Code 会[请 Claude 检查它](/docs/zh-CN/goal#background-work-defers-evaluation)。默认值 `30`。设置 `0` 可关闭检查。请以纯数字给出整分钟数，最大为 `10080`，即一周。Claude Code 会将任何其他值视为未设置并使用默认值。需要 Claude Code v2.1.234 或更高版本 |
| `CLAUDE_CODE_GZIP_REQUEST_BODIES` | 设置为 `0` 可关闭对发送到 `api.anthropic.com` 的 Claude API、遥测和 [Artifact](/docs/zh-CN/artifacts) 发布请求体的 gzip 压缩。默认情况下，Claude Code 会在直接连接时压缩大型请求体，而在您通过代理发送请求、配置客户端证书或设置 `NODE_EXTRA_CA_CERTS` 时跳过压缩。如果 Claude Code 无法检测到的 [TLS 检查代理](/docs/zh-CN/network-config#ca-certificate-store)错误处理压缩请求，请使用 `0` |
| `CLAUDE_CODE_HIDE_CWD` | 设置为 `1` 可在启动徽标中隐藏工作目录。适用于路径会暴露您操作系统用户名的屏幕共享或录屏场景 |
| `CLAUDE_CODE_IDE_HOST_OVERRIDE` | 覆盖用于连接 IDE 扩展的主机地址。默认情况下，Claude Code 会自动检测正确的地址，包括 WSL 到 Windows 的路由 |
| `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL` | 设置为 `1` 可跳过 IDE 扩展的自动安装。等同于将 [`autoInstallIdeExtension`](/docs/zh-CN/settings-reference#autoinstallideextension) 设置为 `false` |
| `CLAUDE_CODE_IDE_SKIP_VALID_CHECK` | 设置为 `1` 可在连接时跳过 IDE 锁文件条目的验证。当 IDE 正在运行但自动连接仍找不到它时使用 |
| `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` | 在 Agent 工具拒绝再生成子代理之前，一个会话中可以同时运行的[子代理](/docs/zh-CN/sub-agents#concurrent-subagent-limit)数量（默认值：20）。接受以纯数字表示的正整数；其他任何值都会被忽略，因此该变量可以调整上限，但不能禁用上限。需要 Claude Code v2.1.217 或更高版本 |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | 覆盖 Claude Code 为当前模型假定的上下文窗口大小。从 v2.1.193 开始，其应用方式取决于 Claude Code 如何解析模型 ID；请参阅[为网关或自定义模型 ID 更正窗口](/docs/zh-CN/model-config#correct-the-window-for-a-gateway-or-custom-model-id)。当通过 `ANTHROPIC_BASE_URL` 路由到某个模型，而该模型的上下文窗口与其名称对应的内置大小不匹配时，请使用此变量 |
| `CLAUDE_CODE_MAX_MCP_DESCRIPTION_LENGTH` | Claude Code 发送给模型的每个 MCP 工具描述和每个 MCP 服务器指令的最大长度（字符数）（默认值：2048）。Claude Code 会[截断更长的文本](/docs/zh-CN/mcp#for-mcp-server-authors)。接受以纯数字表示的正整数。其他任何值都会被忽略并应用默认值。需要 Claude Code v2.1.280 或更高版本 |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | 设置大多数请求的最大输出 token 数。默认值和上限因模型而异；请参阅[最大输出 token](https://platform.claude.com/docs/en/about-claude/models/overview#latest-models-comparison)。如果值超过模型的上限，Claude Code 会将其降至上限。对于 Claude Code 无法解析为已知模型的模型 ID，默认值为 32000，上限为 128000。增大此值会减少触发[自动压缩](/docs/zh-CN/costs#reduce-token-usage)之前可用的有效上下文窗口 |
| `CLAUDE_CODE_MAX_RETRIES` | 覆盖失败 API 请求的重试次数（默认值：10）。从 v2.1.186 开始上限为 15；从 v2.1.199 开始，`CLAUDE_CODE_RETRY_WATCHDOG` 会提高默认值并取消上限。对于需要在较长服务中断期间持续等待的无人值守会话，请改为设置 `CLAUDE_CODE_RETRY_WATCHDOG` |
| `CLAUDE_CODE_MAX_SUBAGENTS_PER_SESSION` | 在 v2.1.224 中移除，现在不起作用。此前用于限制 Claude 在一个会话中可通过 Agent 工具生成的[子代理](/docs/zh-CN/sub-agents)总数（默认值：200）；超过上限的生成会失败并显示 `Subagent spawn limit reached`。[并发子代理限制](/docs/zh-CN/sub-agents#concurrent-subagent-limit)和[深度限制](/docs/zh-CN/sub-agents#let-subagents-spawn-their-own-subagents)仍然适用 |
| `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` | 主对话之下允许的[子代理层数](/docs/zh-CN/sub-agents#let-subagents-spawn-their-own-subagents) （默认值：3）。在默认值下，子代理可以生成自己的子代理，而位于第三层的子代理不能再继续生成；设置 `1` 可关闭嵌套。在 v2.1.217 至 v2.1.218 中，默认值为 1，因此除非您提高限制，否则子代理无法生成自己的子代理；v2.1.219 将默认值提高到 3。接受以纯数字表示的正整数；其他任何值都会被忽略，因此该限制可以调整但不能移除。需要 Claude Code v2.1.217 或更高版本 |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | 可并行执行的只读工具和子代理的最大数量（默认值：10）。较高的值会提高并行度，但会消耗更多资源 |
| `CLAUDE_CODE_MAX_TURNS` | 在未传递显式限制时，限制 agentic 轮次的数量。等同于传递 [`--max-turns`](/docs/zh-CN/cli-reference#cli-flags)，两者同时设置时后者优先。非正整数的值会在启动时被拒绝并报错，而不是被视为无上限 |
| `CLAUDE_CODE_MAX_WEB_SEARCHES_PER_SESSION` | 一个会话可以发起的 [WebSearch](/docs/zh-CN/tools-reference#websearch-tool-behavior) 调用总数上限（默认值：200）。当 Claude 达到上限时，后续的 WebSearch 调用会返回一条通知，告知它使用已收集的信息继续。接受没有上界的正整数。其他任何值都会被忽略并应用默认值，因此该上限可以提高但不能关闭。需要 Claude Code v2.1.212 或更高版本 |
| `CLAUDE_CODE_MCP_ALLOWLIST_ENV` | 设置为 `1` 可让 stdio MCP 服务器仅以安全的基线环境加上服务器配置的 `env` 启动，而不是继承您的 shell 环境 |
| `CLAUDE_CODE_MCP_AUTO_BACKGROUND_MS` | 仍在运行的 MCP 工具调用[转为后台任务](/docs/zh-CN/mcp#automatic-backgrounding-of-long-tool-calls)之前经过的时间（毫秒）（默认值：120000，即 2 分钟）。设置为 `0` 可关闭自动转入后台。需要 Claude Code v2.1.212 或更高版本 |
| `CLAUDE_CODE_MCP_STARTUP_WAIT_MS` | [非交互](/docs/zh-CN/headless)会话的第一轮等待仍在连接中的 MCP 服务器的时长（毫秒），用于替代默认的[第一轮等待](/docs/zh-CN/agent-sdk/mcp#connection-timing)。设置后，该等待涵盖所有待连接的服务器。设置为 `0` 可跳过等待。无论此值如何，[`--permission-prompt-tool`](/docs/zh-CN/cli-reference#cli-flags) 服务器都保留其自身的 `MCP_TIMEOUT` 等待。需要 Claude Code v2.1.274 或更高版本 |
| `CLAUDE_CODE_MCP_TOOL_IDLE_TIMEOUT` | MCP 工具调用的空闲超时时间（毫秒）。当 stdio、HTTP、SSE、WebSocket 或 [claude.ai 连接器](/docs/zh-CN/mcp#use-mcp-servers-from-claude-ai) MCP 服务器在这么长时间内既没有发送响应也没有发送进度通知时，工具调用会报错中止，而不是等待整体的 `MCP_TOOL_TIMEOUT`。覆盖各传输方式的默认值：网络服务器为 300000（5 分钟），stdio 服务器为 1800000（30 分钟）。设置为 `0` 可禁用空闲检查。低于 1000 的值会被提高到一秒，且该值的上限为实际生效的 `MCP_TOOL_TIMEOUT`。`.mcp.json` 中至少为 1000 的单服务器 `timeout` 会将该服务器的空闲窗口提高到至少为 `timeout` 的值。不适用于 IDE 服务器或 SDK 进程内服务器。需要 Claude Code v2.1.187 或更高版本。在 v2.1.203 之前，stdio 服务器不受空闲超时限制 |
| `CLAUDE_CODE_MESSAGING_SOCKET` | 由 Claude Code 设置，而非由您设置：在绑定了[收件箱套接字](/docs/zh-CN/cross-session-messaging#the-sessions-inbox-socket)的会话中，Claude Code 会在绑定套接字时将该套接字的路径导出给 hook 和 Bash 命令。在启动时即开启消息功能的会话中，Claude Code 会在任何 hook 运行之前绑定套接字。计算机上的其他会话会将消息投递到此路径。每个会话导出自己的套接字，而不是从父会话继承的套接字，到达该套接字的消息会经过会话的[入站控制](/docs/zh-CN/cross-session-messaging#control-inbound-messages)。设置中的 `env` 块无法设置它。需要 Claude Code v2.1.224 或更高版本 |
| `CLAUDE_CODE_MESSAGING_TOKEN` | 由 Claude Code 设置，而非由您设置：在绑定了[收件箱套接字](/docs/zh-CN/cross-session-messaging#the-sessions-inbox-socket)的会话中，Claude Code 会将此会话专属令牌与 `CLAUDE_CODE_MESSAGING_SOCKET` 一起导出给 hook 和 Bash 命令。向套接字发送内容的脚本可以将 `{"type":"auth","token":"<token>"}` 作为第一行发送，以证明它属于该会话。在原生 Windows 上，Claude Code 要求必须发送此行，并会关闭任何未以有效的此行开头的连接。[自有子进程规则](/docs/zh-CN/cross-session-messaging#the-sessions-inbox-socket)说明了 Claude Code 何时会查验该令牌。每个会话导出自己的令牌，绝不会使用从父会话继承的令牌。设置中的 `env` 块无法设置它。需要 Claude Code v2.1.228 或更高版本 |
| `CLAUDE_CODE_NATIVE_CURSOR` | 设置为 `1` 可在输入插入点显示终端自身的光标，而不是绘制的方块。该光标遵循终端的闪烁、形状和焦点设置 |
| `CLAUDE_CODE_NEW_INIT` | 设置为 `1` 可使 `/init` 运行交互式设置流程。该流程会先询问要生成哪些文件（包括 CLAUDE.md、skill 和 hook），然后再探索代码库并写入这些文件。如果不设置此变量，`/init` 会自动生成 CLAUDE.md 而不进行询问 |
| `CLAUDE_CODE_NONBLOCKING_STDOUT` | 设置为 `1` 可通过第二个非阻塞文件描述符写入终端输出，这样停止读取的终端（例如暂停的 tmux 控制模式窗格或停滞的 SSH 连接）就不会在会话中途冻结 Claude Code。在 stdout 为终端时适用于 macOS、Linux 和 WSL。需要 Claude Code v2.1.261 或更高版本 |
| `CLAUDE_CODE_NONSTREAMING_TIMEOUT_RETRIES` | 限制 Claude Code 重新发送超时的[非流式请求](/docs/zh-CN/errors#streaming-response-ended-before-any-complete-data-was-received)的次数。设置为 `0` 时，请求在第一次超时时即失败。默认未设置，因此由 `CLAUDE_CODE_MAX_RETRIES` 限制这些重新发送。有关超时时间，请参阅[调整重试行为](/docs/zh-CN/errors#tune-retry-behavior)。需要 Claude Code v2.1.285 或更高版本 |
| `CLAUDE_CODE_NO_FLICKER` | 设置为 `1` 可启用[全屏渲染](/docs/zh-CN/fullscreen)，这是一项研究预览功能，可减少闪烁并在长对话中保持内存占用平稳。覆盖 [`tui`](/docs/zh-CN/settings-reference#tui) 设置；您也可以使用 `/tui fullscreen` 进行切换 |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | 用于 Claude.ai 身份验证的 OAuth 刷新令牌。设置后，`claude auth login` 会直接兑换此令牌，而不是打开浏览器。需要 `CLAUDE_CODE_OAUTH_SCOPES`。适用于在自动化环境中预配身份验证 |
| `CLAUDE_CODE_OAUTH_SCOPES` | 颁发刷新令牌时使用的 OAuth 作用域，以空格分隔，例如 `"user:profile user:inference user:sessions:claude_code"`。设置 `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` 时必需 |
| `CLAUDE_CODE_OAUTH_TOKEN` | 用于 claude.ai 身份验证的 OAuth 访问令牌。在 SDK 和自动化环境中可替代 `/login`。优先于存储在钥匙串中的凭据。可使用 [`claude setup-token`](/docs/zh-CN/authentication#generate-a-long-lived-token) 生成。除非您运行 [`/login`](/docs/zh-CN/authentication#authentication-precedence)，否则 Claude Code 会在整个会话中使用您设置的令牌。要替换过期的令牌，请生成新令牌并重新启动 |
| `CLAUDE_CODE_OPUS_4_6_FAST_MODE_OVERRIDE` | 在 v2.1.160 中移除，现在不起作用。此前用于将[快速模式](/docs/zh-CN/fast-mode)固定到 Claude Opus 4.6，而不是当前默认模型。Opus 4.6 不再支持快速模式 |
| `CLAUDE_CODE_OTEL_CONTENT_MAX_LENGTH` | 承载内容的 OpenTelemetry 属性（模型响应、工具内容、系统提示词、原始 API 正文）的最大长度，包括截断标记，以 UTF-16 代码单元计（默认值：61440，即 60 KB）。仅当您的遥测后端接受大于 64 KB 的属性值时才调高此值，或调低此值以减少遥测量。需要 Claude Code v2.1.214 或更高版本。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `CLAUDE_CODE_OTEL_DIAG_STDERR` | 设置为 `1` 可将 OpenTelemetry 导出器诊断错误写入 stderr。默认情况下，这些错误仅在使用 `--debug` 时显示，因此配置错误的导出器（例如 Prometheus 端口冲突）在其他情况下会静默失败。需要 Claude Code v2.1.179 或更高版本。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` | 刷新待处理 OpenTelemetry span 的超时时间（毫秒）（默认值：5000）。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` | 刷新动态 OpenTelemetry 标头的间隔（毫秒）（默认值：1740000 / 29 分钟）。请参阅[动态标头](/docs/zh-CN/monitoring-usage#dynamic-headers) |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | OpenTelemetry 导出器在关闭时完成工作的超时时间（毫秒）（默认值：2000）。如果退出时指标丢失，请调高此值。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `CLAUDE_CODE_PACKAGE_MANAGER_AUTO_UPDATE` | 设置为 `1` 可让 Claude Code 在有新版本可用时在后台运行包管理器的升级命令。适用于 Homebrew 和 WinGet 安装。其他包管理器仍只显示升级命令而不运行它。请参阅[自动更新](/docs/zh-CN/setup#auto-updates) |
| `CLAUDE_CODE_PERFORCE_MODE` | 设置为 `1` 可启用 Perforce 感知的写保护。设置后，如果目标文件缺少所有者写入位（Perforce 会清除已同步文件的该位，直到 `p4 edit` 将其打开），Edit、Write 和 NotebookEdit 会失败并给出 `p4 edit <file>` 提示。这可以防止 Claude Code 绕过 Perforce 变更跟踪 |
| `CLAUDE_CODE_PLUGIN_CACHE_DIR` | 覆盖插件根目录。尽管名称如此，它设置的是父目录，而不是缓存本身：市场和插件缓存位于此路径下的子目录中。默认为 `~/.claude/plugins` |
| `CLAUDE_CODE_PLUGIN_DIRS` | 为会话加载的插件目录，每个目录的加载方式与 [`--plugin-dir`](/docs/zh-CN/plugins/cli-reference#flags-that-load-a-plugin-for-one-session) 标志相同。在 Unix 上用 `:` 分隔多个路径，在 Windows 上用 `;` 分隔。每个路径请使用绝对路径或以 `~` 开头，因为 Claude Code 会跳过相对路径。需要 Claude Code v2.1.280 或更高版本。请参阅[为单个会话加载插件](/docs/zh-CN/plugins/create#load-a-directory-or-archive-for-one-session) |
| `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` | 克隆或刷新插件市场的超时时间（毫秒）（默认值：120000）。对于大型仓库或较慢的网络连接，请调高此值。请参阅 [Git clone timed out](/docs/zh-CN/plugins/troubleshooting#git-clone-timed-out-after-120s) |
| `CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE` | 设置为 `1` 可在市场刷新无法访问远程或无法向远程进行身份验证时跳过重新克隆尝试，并继续使用现有的市场检出。适用于离线或隔离网络环境，在这些环境中重新克隆也会以同样的方式失败。请参阅[市场更新在离线环境中失败](/docs/zh-CN/plugins/troubleshooting#marketplace-updates-keep-failing-offline) |
| `CLAUDE_CODE_PLUGIN_PREFER_HTTPS` | 设置为 `1` 可通过 HTTPS 而不是 SSH 克隆 GitHub `owner/repo` 简写来源。适用于插件安装和更新，以及 `/plugin marketplace add` 和 `update`。适用于 CI 运行器、容器或任何未为 `github.com` 配置 SSH 密钥的环境 |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | 一个或多个只读插件种子目录的路径，在 Unix 上以 `:` 分隔，在 Windows 上以 `;` 分隔。使用它可将预先填充的插件目录打包到容器镜像中。Claude Code 会在启动时从这些目录注册市场，并使用预先缓存的插件而无需重新克隆。请参阅[为容器预先填充插件](/docs/zh-CN/plugins/org#seed-containers-and-ci) |
| `CLAUDE_CODE_POWERSHELL_RESPECT_EXECUTION_POLICY` | 设置为 `1` 可阻止 Claude Code 在为工具调用、hook 和状态栏命令启动 PowerShell 时传递 `-ExecutionPolicy Bypass`，转而遵循计算机的有效执行策略。默认情况下，Claude Code 会在进程作用域绕过执行策略，以便 `.ps1` 脚本和模块导入能在默认为 Restricted 的 Windows 安装上正常工作。无论此设置如何，进程作用域的绕过都不会覆盖组策略 `MachinePolicy` 或 `UserPolicy` |
| `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS` | 在使用 `-p` 标志的[非交互模式](/docs/zh-CN/headless#background-tasks-at-exit)下，最后一轮之后空闲等待后台工作（例如子代理和工作流）的上限（毫秒）。每当 Claude 用一轮来处理后台结果时，空闲等待都会重新计时。默认值：`600000`，即 10 分钟。当空闲等待达到上限时，Claude Code 会停止等待剩余的后台任务并退出。设置为 `0` 可无限期等待。此上限独立于适用于普通后台 shell 的五秒宽限期。需要 Claude Code v2.1.182 或更高版本 |
| `CLAUDE_CODE_PROCESS_WRAPPER` | 通过以 argv 前缀形式给出的企业启动器（如 `/opt/corp/launcher`）来启动 Claude Code 从其自身二进制文件启动的进程，例如托管 [Agent 视图](/docs/zh-CN/agent-view)会话的后台服务。请在用户设置或[托管设置](/docs/zh-CN/managed-settings)的 `env` 块中设置，而不是作为 shell 导出，以便分离的后台服务能够继承它；项目设置和本地设置无法设置它。等同于 [`processWrapper` 设置](/docs/zh-CN/settings-reference#processwrapper)，该设置需要 Claude Code v2.1.210 或更高版本；两者同时设置时，此变量优先。VS Code 扩展通过其 `claudeProcessWrapper` 设置单独配置自己的启动器。在 Windows 上会被忽略。有关值格式、启动器涵盖的范围以及启动器必须满足的约定，请参阅[在企业启动器后运行 Claude Code](/docs/zh-CN/corporate-launcher)。需要 Claude Code v2.1.208 或更高版本 |
| `CLAUDE_CODE_PROJECT_DIR_NAME` | 与 `CLAUDE_CONFIG_DIR` 一起设置，用于选择 Claude Code 存储该会话的会话记录和自动记忆的 `projects/` 目录名称，以替代从工作目录路径派生的名称。例如，使用 `CLAUDE_CONFIG_DIR=/srv/tenant-a CLAUDE_CODE_PROJECT_DIR_NAME=work claude` 启动 Claude Code 会将它们存储在 `/srv/tenant-a/projects/work/` 下。未设置 `CLAUDE_CONFIG_DIR` 时，Claude Code 会忽略此变量，并且仅从您启动 `claude` 的环境中读取它，绝不会从[设置文件的 `env` 块](#in-settings-files)中读取。请参阅[自行命名项目目录](/docs/zh-CN/sessions#name-the-project-directory-yourself)。需要 Claude Code v2.1.234 或更高版本 |
| `CLAUDE_CODE_PROMPT_CACHE_TTL` | 设置为 `5m` 或 `1h`（Claude Code 仅接受这两个值），为主对话选择[提示缓存 TTL](/docs/zh-CN/prompt-caching#cache-lifetime)：包括您的交互式、`-p` 和 SDK 轮次，以及与它们内联运行的辅助程序。优先于 `promptCacheTtl` 设置和 `ENABLE_PROMPT_CACHING_1H`，而 `FORCE_PROMPT_CACHING_5M` 会覆盖它。API 对 1 小时缓存写入按更高费率计费。需要 Claude Code v2.1.242 或更高版本 |
| `CLAUDE_CODE_PROPAGATE_TRACEPARENT` | 当 `ANTHROPIC_BASE_URL` 指向自定义代理时，设置为 `1` 可传播 W3C 追踪上下文。传播范围包括模型请求和 HTTP MCP 请求上的 `traceparent` 标头，以及 Bash、PowerShell 和 hook 子进程的 `TRACEPARENT` 环境变量。默认情况下，仅在直接连接到 Anthropic API 时才启用传播。在 v2.1.152 中添加。请参阅[追踪（beta）](/docs/zh-CN/monitoring-usage#traces-beta) |
| `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST` | 由嵌入 Claude Code 并代其管理模型提供商路由的宿主平台设置。设置后，Claude Code 会忽略设置文件中的提供商选择、端点和身份验证变量，例如 `CLAUDE_CODE_USE_BEDROCK`、`ANTHROPIC_BASE_URL` 和 `ANTHROPIC_API_KEY`，因此用户设置无法覆盖宿主的路由。Claude Code 还会忽略[托管设置](/docs/zh-CN/managed-settings)中的模型选择键，例如 `model`、`fallbackModel` 和 `modelOverrides`，无论它们由哪个托管来源下发，从而使宿主的模型配置优先于过时的托管模型固定设置。Claude Code 还会忽略托管 `env` 块中的模型选择变量，例如 `ANTHROPIC_MODEL` 和 `ANTHROPIC_DEFAULT_*_MODEL` 系列；托管设置中的 [`availableModels`](/docs/zh-CN/model-config#restrict-model-selection) 允许列表仍然适用，除非宿主提供了自己的允许列表。Claude Code 还会跳过它在 Amazon Bedrock、Claude Platform on AWS、Google Cloud's Agent Platform 和 Microsoft Foundry 等第三方提供商上原本会应用的自动遥测退出，因此遥测遵循标准的 `DISABLE_TELEMETRY` 退出方式。请参阅[按 API 提供商划分的默认行为](/docs/zh-CN/data-usage#default-behaviors-by-api-provider) |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` | 设置为 `1` 可允许代理执行 DNS 解析，而不是由调用方执行。适用于应由代理处理主机名解析的环境，需手动选择启用 |
| `CLAUDE_CODE_REMOTE` | 当 Claude Code 作为[云端会话](/docs/zh-CN/claude-code-on-the-web)运行时自动设置为 `true`。可从 hook 或设置脚本中读取此变量，以检测您是否处于云端会话中 |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | 在[云端会话](/docs/zh-CN/claude-code-on-the-web)中自动设置为当前会话的 ID。读取此变量可构造指回会话记录的链接。请参阅[将输出链接回会话](/docs/zh-CN/cloud-environments#link-output-back-to-the-session) |
| `CLAUDE_CODE_RESTRICTED` | 设置为 `1` 可在受限模式下启动会话，等同于传递 [`--restricted`](/docs/zh-CN/cli-reference#cli-flags)。Claude Code 会忽略设置文件 `env` 块中的此变量。需要 Claude Code v2.1.248 或更高版本 |
| `CLAUDE_CODE_RESUME_INTERRUPTED_TURN` | 设置为 `1` 可在上一个会话于轮次中途结束时自动恢复。用于 SDK 模式，使模型无需 SDK 重新发送提示词即可继续。要关闭此功能，请取消设置该变量或将其设置为 `0`。关于 VS Code 聊天面板，请参阅[重新加载后继续对话](/docs/zh-CN/vs-code#continue-conversations-after-a-reload) |
| `CLAUDE_CODE_RESUME_INTERRUPTED_TURN_MAX_AGE_MS` | 对于在轮次中途结束的会话，若要在恢复时自动继续，其最后一条会话记录消息所允许的最大时长（毫秒）。当最后一条消息早于此界限时，Claude Code 会跳过 `CLAUDE_CODE_RESUME_INTERRUPTED_TURN` 自动恢复及其 `CLAUDE_CODE_RESUME_PROMPT` 继续消息，会话以空闲状态启动，由您显式继续。未设置或为 `0` 表示没有界限，但最后一个请求因 API 错误而失败的轮次只有在该错误发生不到六小时时才会恢复。正值会对所有轮次（包括这些轮次）设置界限；负值或非数字值会应用一小时的界限。长时间运行的 Agent 的启动脚本可以设置此变量，以免针对旧会话记录重新启动时重新运行过时的提示词。当 Claude Code 重新启动一个崩溃的、从交互式会话继承了对话的 [Agent 视图](/docs/zh-CN/agent-view)会话时，它会自行设置一小时的界限。需要 Claude Code v2.1.211 或更高版本 |
| `CLAUDE_CODE_RESUME_PROMPT` | 覆盖 Claude Code 发送给 Claude 的继续消息，该消息用于 `CLAUDE_CODE_RESUME_INTERRUPTED_TURN` 继续被中断的轮次（而非重新发送其提示词）时，或您使用 `-p` 恢复[延迟的工具调用](/docs/zh-CN/hooks#defer-a-tool-call-for-later)时。默认为 `Continue from where you left off.`。空字符串会使用默认值 |
| `CLAUDE_CODE_RETRY_WATCHDOG` | 适用于评估框架、CI 作业或远程工作程序等无人值守会话，设置为 `1`。无限期重试 `429` 和 `529` 容量错误，而不是在 `CLAUDE_CODE_MAX_RETRIES` 次尝试后失败。当标准速度请求收到报告支出限额或使用额度已耗尽的 `429` 时，Claude Code 会立即失败，即使该错误来自按计划重置的[网关支出上限](/docs/zh-CN/errors#spend-limit-reached)。在 v2.1.239 之前，看门狗会无限期重试这些错误。关于快速模式请求，请参阅[处理速率限制](/docs/zh-CN/fast-mode#handle-rate-limits)。看门狗在两次尝试之间最多退避 5 分钟，或者当响应带有速率限制重置时间时一直等到限制重置，因此达到用量限制的会话会等待剩余的时间窗口结束。在 v2.1.199 或更高版本中，它还会将其他瞬态错误（例如服务器错误、超时和连接断开）的默认重试次数提高到 300，约相当于三小时的退避，并在您显式设置 `CLAUDE_CODE_MAX_RETRIES` 时取消其 15 次的上限。需要 Claude Code v2.1.186 或更高版本 |
| `CLAUDE_CODE_SAFE_MODE` | 设置为 `1` 可以安全模式启动：CLAUDE.md、skill、插件、hook、MCP 服务器、自定义命令和 Agent、输出样式、工作流、自定义主题、自定义快捷键、状态栏和文件建议命令、LSP 服务器以及自动记忆都不会加载，用于对损坏的配置进行故障排除。托管设置策略仍然适用，包括策略配置的 hook、状态栏和文件建议命令；托管插件、托管 skill、托管 CLAUDE.md 以及策略配置的 MCP 服务器则不会加载。等同于传递 [`--safe-mode`](/docs/zh-CN/cli-reference#cli-flags)。直接生成的子进程会继承该变量 |
| `CLAUDE_CODE_SCRIPT_CAPS` | JSON 对象，用于在设置了 `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` 时限制每个会话中特定脚本可被调用的次数。键是与命令文本匹配的子字符串；值是整数形式的调用次数限制。例如，`{"deploy.sh": 2}` 允许 `deploy.sh` 最多被调用两次。匹配基于子字符串，因此像 `./scripts/deploy.sh $(evil)` 这样的 shell 扩展技巧仍会计入上限。通过 `xargs` 或 `find -exec` 进行的运行时扇出不会被检测到；这是一项纵深防御控制 |
| `CLAUDE_CODE_SCROLL_SPEED` | 设置[全屏渲染](/docs/zh-CN/fullscreen#mouse-wheel-scrolling)中的鼠标滚轮滚动倍数。接受最大为 20 的任意正值，包括小于 1 的小数值（例如 `0.5`），以便在已经放大滚轮事件的终端中减慢加速的触控板和滚轮滚动。如果您的终端每一格只发送一个滚轮事件且不进行放大，请设置为 `3` 以与 `vim` 保持一致。在 JetBrains IDE 终端中会被忽略，Claude Code 在该终端中使用自己的滚动处理 |
| `CLAUDE_CODE_SEND_FEEDBACK` | 设置为 `0` 可为会话关闭 [Claude 起草的反馈](/docs/zh-CN/tools-reference#sendfeedback-tool-behavior)。设置为 `1` 可在您的账户已有访问权限的情况下开启该功能；该变量本身无法授予访问权限，其他关闭反馈的开关（例如 `DISABLE_FEEDBACK_COMMAND` 和 [`feedbackDrafts`](/docs/zh-CN/settings-reference#feedbackdrafts) 设置的 `off` 值）仍然适用 |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | 覆盖 [SessionEnd](/docs/zh-CN/hooks#sessionend) hook 的时间预算（毫秒）。该值也是每个未设置自身 `timeout` 的 hook 的超时时间。适用于会话退出、`/clear` 以及通过交互式 `/resume` 切换会话。默认预算为 1.5 秒，会自动提高到设置文件中配置的最高单个 hook `timeout`，最多 60 秒。插件提供的 hook 上的超时时间不会提高预算 |
| `CLAUDE_CODE_SESSION_ID` | 在 Bash 和 PowerShell 工具子进程、[hook 命令](/docs/zh-CN/hooks)子进程以及 stdio [MCP 服务器](/docs/zh-CN/mcp)子进程中自动设置为当前会话 ID。对于 Bash、PowerShell 和 hook，它与 hook JSON 输入中的 `session_id` 字段一致，并会在 `/clear` 时更新。MCP 服务器子进程会保留其启动时的 ID。使用 `--resume <session-id>` 时，它会收到恢复后的 ID，与 hook 和 Bash 一致。使用 `--continue` 或不带显式 ID 的 `--resume` 时，它可能会收到初始启动时的 ID。用于将脚本和外部工具与启动它们的 Claude Code 会话关联起来 |
| `CLAUDE_CODE_SHELL` | 设置 Claude Code 运行 Bash 工具命令所用的 shell。接受 `bash` 或 `zsh` 二进制文件的路径，例如 `/opt/homebrew/bin/bash`。不支持 `fish` 等其他 shell。如果该值不是可用的 `bash` 或 `zsh` 路径，Claude Code 会忽略它并回退到自动检测。自动检测会在您的 `$SHELL` 指向 `bash` 或 `zsh` 时使用它，否则会在您的 `PATH` 和标准安装位置中先查找可用的 `zsh`，再查找 `bash`，并选择找到的第一个 |
| `CLAUDE_CODE_SHELL_PREFIX` | 包装 Claude Code 所生成 shell 命令的命令前缀，适用于：Bash 工具调用、[hook](/docs/zh-CN/hooks) 命令、[状态栏](/docs/zh-CN/statusline)命令以及 stdio [MCP 服务器](/docs/zh-CN/mcp)启动命令。PowerShell hook 和 exec 形式的 hook 运行时不带前缀。适用于日志记录或审计。设置为 `/path/to/logger.sh` 这样的裸可执行文件路径时，每条命令会以 `/path/to/logger.sh '<command>'` 的形式运行。包装器会在 `$1` 中以单个经过 shell 引用的参数接收命令行，因此包装器必须用 shell 重新求值 `$1`，例如 `exec bash -c "$1"`。将 `$1` 当作裸可执行文件路径会导致传递 `npx -y <package>` 等参数的 stdio MCP 服务器出错。对于 Bash 工具调用，`$1` 包含 Claude Code 组装的完整 shell 调用（包括环境设置），而不仅仅是 Claude 运行的命令 |
| `CLAUDE_CODE_SIMPLE` | 设置为 `1` 可使用最小系统提示词运行，并且仅提供 Bash、文件读取和文件编辑工具。来自 `--mcp-config` 的 MCP 工具仍然可用。禁用对 hook、skill、自定义命令、子代理、已安装插件、MCP 服务器、自动记忆和 CLAUDE.md 的自动发现。通过 `--add-dir` 传递的目录中的 skill 仍会加载。不会读取 OAuth 令牌和钥匙串凭据，因此 Anthropic 身份验证必须来自 `ANTHROPIC_API_KEY` 或 `--settings` 中的 `apiKeyHelper`。等同于传递 [`--bare`](/docs/zh-CN/headless#start-faster-with-bare-mode) |
| `CLAUDE_CODE_SIMPLE_SYSTEM_PROMPT` | 设置为 `1` 可在任何模型上使用更短的系统提示词和简化的工具描述。设置为 `0`、`false`、`no` 或 `off` 可选择退出，即使在实验或服务器配置原本会启用它的模型上也是如此。完整的工具集、hook、MCP 服务器和 CLAUDE.md 发现仍保持启用 |
| `CLAUDE_CODE_SKIP_ANTHROPIC_AWS_AUTH` | 为 [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 跳过客户端身份验证，适用于自行对请求进行签名的网关 |
| `CLAUDE_CODE_SKIP_AWS_CRED_CACHE` | 设置为 `1` 可关闭从 AWS 默认凭据提供程序链解析出的凭据的进程内缓存，使 Claude Code 在每次 API 请求时都解析该链。关闭缓存后，基于 SSO 的配置文件会在每次请求时向 IAM Identity Center 请求凭据。请参阅[凭据缓存和解析超时](/docs/zh-CN/amazon-bedrock#credential-caching-and-resolution-timeout)。需要 Claude Code v2.1.207 或更高版本 |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | 跳过 Amazon Bedrock 的 AWS 身份验证（例如，使用 LLM 网关时） |
| `CLAUDE_CODE_SKIP_FAST_MODE_NETWORK_ERRORS` | 设置为 `1` 可将失败的[快速模式](/docs/zh-CN/fast-mode#use-fast-mode-behind-proxies-and-llm-gateways)可用性检查视为可用，适用于阻止该检查直接请求 `api.anthropic.com` 的网络。Claude Code 仍会遵循“已被您的组织禁用”的响应 |
| `CLAUDE_CODE_SKIP_FAST_MODE_ORG_CHECK` | 设置为 `1` 可跳过客户端的[快速模式](/docs/zh-CN/fast-mode#use-fast-mode-behind-proxies-and-llm-gateways)可用性检查，适用于拦截该检查请求而不是拒绝它的代理。当您的组织禁用了快速模式时，API 仍会拒绝快速模式请求 |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | 为 Microsoft Foundry 跳过 Azure 身份验证，适用于会注入自己的 `Authorization` 标头的代理或网关。Claude Code 发送请求时不附带 Azure 凭据，并保留您提供的 `Authorization` 标头（例如通过 `ANTHROPIC_CUSTOM_HEADERS` 提供）。设置了 `ANTHROPIC_FOUNDRY_API_KEY` 或 `ANTHROPIC_FOUNDRY_AUTH_TOKEN` 时会被忽略。在 v2.1.203 之前，除非同时设置了 API 密钥，否则此变量会导致 Microsoft Foundry 客户端无法发送请求 |
| `CLAUDE_CODE_SKIP_MANTLE_AUTH` | 跳过 Amazon Bedrock Mantle 的 AWS 身份验证（例如，使用 LLM 网关时） |
| `CLAUDE_CODE_SKIP_MODEL_ACCESS_MEMORY` | [Amazon Bedrock](/docs/zh-CN/amazon-bedrock) 和 [Google Cloud's Agent Platform](/docs/zh-CN/google-vertex-ai) 上的[启动模型检查](/docs/zh-CN/amazon-bedrock#startup-model-checks)会在此计算机上记住它们发现您的账户无法调用哪些模型，最长保留一天。设置为 `1` 可关闭这一记忆功能。需要 Claude Code v2.1.285 或更高版本 |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | 设置为 `1` 可跳过将提示词历史记录和会话记录写入磁盘。设置此变量后启动的会话不会出现在 `--resume`、`--continue` 或上箭头历史记录中。适用于临时的脚本化会话 |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | 跳过 Google Cloud's Agent Platform 的 Google 身份验证（例如，使用 LLM 网关时） |
| `CLAUDE_CODE_STARTUP_FAILURE_RESULTS` | 设置为 `1` 可让使用 `--output-format stream-json` 启动的会话，在原本仅以 stderr 输出结束的启动失败情况下，写入一条[说明 Claude Code 拒绝启动原因的结果消息](/docs/zh-CN/agent-sdk/typescript#startup_failure_reason)。需要 Claude Code v2.1.274 或更高版本 |
| `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` | [Stop](/docs/zh-CN/hooks#stop) 或 [SubagentStop](/docs/zh-CN/hooks#subagentstop) hook 可连续阻止轮次结束的最大次数，超过后 Claude Code 会覆盖它并强制结束该轮次（默认值：8）。设置为 `0` 可禁用该上限。如果您的 hook 确实需要更多迭代才能完成，请调高此值 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | [子代理](/docs/zh-CN/sub-agents#choose-a-model)、[agent team](/docs/zh-CN/agent-teams#specify-teammates-and-models) 队友以及[工作流](/docs/zh-CN/workflows) Agent 在未通过其他方式分配模型时使用的默认模型。接受 `haiku` 等别名或完整模型名称。有两个来源优先于它：Claude 生成 Agent 时传递的模型，以及 Agent 定义中的 `model` 字段（包括 `inherit`）。要改变这一点，请设置 [`CLAUDE_CODE_SUBAGENT_MODEL_FORCE`](/docs/zh-CN/sub-agents#run-every-subagent-on-one-model)。完整顺序请参阅[选择模型](/docs/zh-CN/sub-agents#choose-a-model)。将其设置为 `inherit` 与不设置相同。在 v2.1.251 之前，此变量会同时覆盖每次调用的模型和定义中的 `model` 字段 |
| `CLAUDE_CODE_SUBAGENT_MODEL_FORCE` | 设置为 `1` 可将同一个模型强制应用于子代理、队友和工作流 Agent。[在同一模型上运行所有子代理](/docs/zh-CN/sub-agents#run-every-subagent-on-one-model)说明了具体是哪个模型。需要 Claude Code v2.1.257 或更高版本 |
| `CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL` | 设置为 `5m` 或 `1h`（Claude Code 仅接受这两个值），为主对话之外的请求选择[提示缓存 TTL](/docs/zh-CN/prompt-caching#cache-lifetime)，例如[子代理](/docs/zh-CN/sub-agents)、工作流和后台工作。优先于 `subagentPromptCacheTtl` 设置和 `ENABLE_PROMPT_CACHING_1H`，而 `FORCE_PROMPT_CACHING_5M` 会覆盖它。API 对 1 小时缓存写入按更高费率计费。需要 Claude Code v2.1.242 或更高版本 |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | 设置为 `1` 可从 Claude Code 启动的子进程（例如 Bash 命令、hook 和 stdio MCP 服务器）的环境中剥离凭据。清理会根据变量名或变量值识别凭据，并保留 GitHub 令牌和代理设置。请参阅[子进程环境清理会移除哪些内容](#what-the-subprocess-environment-scrub-removes)。配置了 `allowed_non_write_users` 时，`claude-code-action` 会自动设置此变量 |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` | 在非交互模式（`-p` 标志）下设置为 `1`，可在第一次查询之前等待插件安装完成。否则，插件会在后台安装，在第一轮中可能不可用。与 `CLAUDE_CODE_SYNC_PLUGIN_INSTALL_TIMEOUT_MS` 结合使用可限制等待时间 |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL_TIMEOUT_MS` | 同步插件安装的超时时间（毫秒）。超时后，Claude Code 会在没有插件的情况下继续并记录一条错误。没有默认值：不设置此变量时，同步安装会一直等待直到完成 |
| `CLAUDE_CODE_SYNC_SKILLS` | 在使用 `-p` 标志的非交互模式下设置为 `1`，可让 Claude Code 在该次运行中下载为您的 claude.ai 账户启用的 skill，并在运行第一次查询之前等待它们的列表，最长等待 `CLAUDE_CODE_SYNC_SKILLS_WAIT_TIMEOUT_MS`。下载本身会在后台完成，Claude 在调用某个 skill 时会等待该 skill 下载完成。需要 claude.ai 身份验证。使用 claude.ai 账户登录的终端会话无需此变量即可将这些 skill [下载](/docs/zh-CN/skills#where-synced-skills-load)到 `~/.claude/skills/synced/`，并大约每 10 分钟重新同步一次，因此仅当某次 `-p` 运行需要在第一次查询时就使用您最新的 skill 时才需设置此变量。在 v2.1.273 之前，终端会话仅在设置了此变量的 `-p` 运行中下载这些 skill。`synced` 文件夹名称[为此下载保留](/docs/zh-CN/skills#where-skills-live)。在 v2.1.227 之前，skill 会直接下载到 `~/.claude/skills/`。Claude Code 会对[下载的 skill 应用额外规则](/docs/zh-CN/skills#how-synced-skills-behave)，例如不在您的计算机上运行其 `!` 命令 |
| `CLAUDE_CODE_SYNC_SKILLS_INSTALL_TIMEOUT_MS` | 当基于 [Agent SDK](/docs/zh-CN/agent-sdk/typescript#query-object) 构建的应用重新加载 skill 时，在会话中途运行的 skill 重新同步的超时时间（毫秒）（默认值：30000）。超时后，重新加载会使用已到达的 skill 继续，其余下载在后台完成 |
| `CLAUDE_CODE_SYNC_SKILLS_WAIT_TIMEOUT_MS` | 设置了 `CLAUDE_CODE_SYNC_SKILLS` 时，第一次查询等待初始 skill 列表的超时时间（毫秒）（默认值：5000）。超时后，第一次查询会使用已到达的 skill 运行。无论哪种情况，下载都会在后台完成，Claude 在调用某个 skill 时会等待该 skill 下载完成 |
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT` | 设置为 `false` 可禁用 diff 输出中的语法高亮。当颜色干扰您的终端设置时很有用。要同时禁用代码块和文件预览中的高亮，请使用 [`syntaxHighlightingDisabled`](/docs/zh-CN/settings-reference#syntaxhighlightingdisabled) 设置 |
| `CLAUDE_CODE_TASK_LIST_ID` | 在会话之间共享任务列表。在[具有 Task 工具的会话](/docs/zh-CN/tools-reference#task-tool-availability)中，在多个 Claude Code 实例中设置相同的 ID，即可在共享任务列表上协作。请参阅[任务列表](/docs/zh-CN/interactive-mode#task-list) |
| `CLAUDE_CODE_TEAM_TEARDOWN_PARK_TIMEOUT_MS` | 以毫秒为单位，覆盖非交互式会话在退出时等待其 [agent team](/docs/zh-CN/agent-teams) 完成拆除的时长。接受 1000 至 60000；超出范围的值会被忽略并应用默认值 10000。需要 Claude Code v2.1.206 或更高版本 |
| `CLAUDE_CODE_TMPDIR` | 覆盖用于内部临时文件的临时目录。Claude Code 在 Unix 上会在此路径后追加 `/claude-{uid}/`，在 Windows 上追加 `/claude/`。默认值：macOS 上为 `/tmp`，Linux 和 Windows 上为 `os.tmpdir()`。在 macOS 和 Linux 上，当您的覆盖值是较长路径时，[沙箱隔离](/docs/zh-CN/sandboxing)的 Bash 子进程会收到系统默认目录下一个较短的备用 `$TMPDIR`，因为某些工具在临时路径过长时会失败。未使用沙箱的 Bash 命令会在您的 shell 设置了 `$TMPDIR` 时继承它。在原生 Windows 上，当您的 shell 未设置 `$TMPDIR` 时，引用 `$TMPDIR` 的 Bash 命令会收到您的覆盖值，如果您未设置覆盖值则收到 `%TEMP%`。Claude Code 自身的临时文件始终使用您的覆盖值。请在您的 shell、用户设置或托管设置中设置。在[项目和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中会被忽略 |
| `CLAUDE_CODE_TMUX_TRUECOLOR` | 设置为任意非空值（例如 `1`）可在 tmux 中允许 24 位真彩色输出。**设置为 `0` 或 `false` 仍会允许真彩色**，这与大多数开关变量不同；取消设置该变量可恢复 256 色限制。默认情况下，当设置了 `$TMUX` 时，Claude Code 会限制为 256 色，因为除非经过配置，否则 tmux 不会透传真彩色转义序列。请在将 `set -ga terminal-overrides ',*:Tc'` 添加到您的 `~/.tmux.conf` 之后设置此变量。其他 tmux 设置请参阅[终端配置](/docs/zh-CN/terminal-config) |
| `CLAUDE_CODE_TOOL_MEMORY_CGROUP_EXCLUDE` | 在 Linux 和 WSL 上，设置为以逗号分隔的进程类型列表，Claude Code 会将这些类型[排除在工具内存上限之外](/docs/zh-CN/tools-reference#memory-limit-on-linux-and-wsl)，例如 `mcp` 或 `lsp`。设置 `none` 可对所有类型施加上限，或设置 `all-new` 仅对 Bash、PowerShell 和 Monitor 工具命令施加上限。无论您列出什么，Claude Code 都会让 Bash、PowerShell 和 Monitor 工具命令受该上限约束。需要 Claude Code v2.1.246 或更高版本 |
| `CLAUDE_CODE_TOOL_MEMORY_LIMIT` | 在 Linux 和 WSL 上，设置为 `4G` 等大小，可[限制 Bash 和 PowerShell 工具命令可使用的内存](/docs/zh-CN/tools-reference#memory-limit-on-linux-and-wsl)，在 v2.1.246 或更高版本中还包括 Monitor 工具命令。请以纯数字写入大小，单独的数字表示字节数，也可带 `K`、`M`、`G` 或 `T` 后缀。设置 `0` 或 `off` 可关闭上限。一旦 Claude Code 启动的第一个进程已开启或关闭上限，更改后的值将在您下次启动 `claude` 时生效。需要 Claude Code v2.1.233 或更高版本 |
| `CLAUDE_CODE_TRANSCRIPT_LOCAL_GC` | 设置为 `1` 可限制长时间运行的 `-p` 或 Agent SDK 会话的[会话记录文件](/docs/zh-CN/sessions#where-transcripts-are-stored)的增长大小。每次压缩之后，一旦文件大于 5 MB，Claude Code 会移除该次压缩之前的历史记录。无论文件是否经过裁剪，恢复会话都会还原相同的对话。请在您启动 Claude Code 的环境中设置它，因为设置中的 `env` 块无法开启它。需要 Claude Code v2.1.287 或更高版本 |
| `CLAUDE_CODE_USER_DIALOG_TIMEOUT_MS` | Claude Code 取消其转发给远程客户端（例如 [Remote Control](/docs/zh-CN/remote-control) 或 SDK 宿主）的对话框，或取消[被搁置的跨会话消息](/docs/zh-CN/cross-session-messaging#control-inbound-messages)的批准对话框之前的截止时间（毫秒）；权限提示和 `AskUserQuestion` 问题使用各自的流程，不受其控制。在 Claude Code v2.1.236 或更高版本中，它还会限制可能在无人值守状态下运行的会话中于会话中途出现的 [Fable 使用额度同意提示](/docs/zh-CN/model-config#fable-and-usage-credits)。[控制入站消息](/docs/zh-CN/cross-session-messaging#control-inbound-messages)和[非交互式会话](/docs/zh-CN/cross-session-messaging#non-interactive-sessions)介绍了完整的被搁置消息过期规则，包括截止时间不适用的情况。覆盖 [`dialogExpiry`](/docs/zh-CN/settings-reference#dialogexpiry) 设置。`0` 或负值会禁用截止时间 |
| `CLAUDE_CODE_USE_ANTHROPIC_AWS` | 使用 [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) |
| `CLAUDE_CODE_USE_BEDROCK` | 使用 [Amazon Bedrock](/docs/zh-CN/amazon-bedrock) |
| `CLAUDE_CODE_USE_FOUNDRY` | 使用 [Microsoft Foundry](/docs/zh-CN/microsoft-foundry) |
| `CLAUDE_CODE_USE_MANTLE` | 使用 Amazon Bedrock [Mantle 端点](/docs/zh-CN/amazon-bedrock#use-the-mantle-endpoint) |
| `CLAUDE_CODE_USE_NATIVE_FILE_SEARCH` | 设置为 `1` 可使用 Node.js 文件 API 而不是 ripgrep 来发现自定义命令、子代理和输出样式。如果捆绑的 ripgrep 二进制文件在您的环境中不可用或被阻止，请设置此变量。不影响 Grep 或文件搜索工具 |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | 控制 PowerShell 工具。在未安装 Git Bash 的 Windows 上，该工具会自动启用；设置为 `0` 可禁用。在安装了 Git Bash 的 Windows 上，该工具对 claude.ai 和 Console 账户默认开启；设置为 `1` 可在 Amazon Bedrock、Google Cloud's Agent Platform 和 Microsoft Foundry 会话中启用它，设置为 `0` 可关闭它。在 Linux、macOS 和 WSL 上，设置为 `1` 可启用它，这要求您的 `PATH` 中有 `pwsh`。在 Windows 上启用后，Claude 可以原生运行 PowerShell 命令，而不必经由 Git Bash 路由。请参阅 [PowerShell 工具](/docs/zh-CN/tools-reference#powershell-tool) |
| `CLAUDE_CODE_USE_VERTEX` | 使用 [Google Cloud's Agent Platform](/docs/zh-CN/google-vertex-ai) |
| `CLAUDE_CODE_WEBFETCH_CACHE_TTL_MS` | 设置为 [WebFetch](/docs/zh-CN/tools-reference#webfetch-tool-behavior) 缓存每个已获取 URL 响应的毫秒数。默认值为 `900000`，即 15 分钟。仅接受纯数字；`0`、小数或任何其他写法都会保留默认值。Claude Code 每次启动时读取一次该值，因此在设置的 `env` 块中所做的更改会在您下次启动 `claude` 时生效。需要 Claude Code v2.1.233 或更高版本 |
| `CLAUDE_CODE_WEBFETCH_DEADLINE_MS` | [WebFetch](/docs/zh-CN/tools-reference#webfetch-tool-behavior) 等待页面下载完成的时间上限（以毫秒为单位），包括其跟随的任何重定向。到时仍未完成的下载会以截止时间错误失败。默认值为 `300000`，即五分钟。设置为 `0` 可取消该限制。仅接受纯数字；小数或任何其他写法都会保留默认值。需要 Claude Code v2.1.268 或更高版本 |
| `CLAUDE_CODE_WORKER_CHECKIN_SCHEDULE` | 当 `CLAUDE_AUTO_BACKGROUND_TASKS` 设置为 `1` 时，Claude Code 每次提醒 Claude 检查仍在运行的[后台子代理](/docs/zh-CN/sub-agents#run-subagents-in-foreground-or-background)之前等待的时长。接受一个或多个以逗号分隔的等待时间，以整秒为单位，范围从 `1` 到 `86400`，例如 `600` 或 `600,1800,3600`。每个值是距下一次提醒的等待时间，最后一个值会重复使用。仅接受纯数字；任何其他值或写法都视为未设置。未设置时不会发送提醒。需要 Claude Code v2.1.283 或更高版本 |
| `CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS` | 单次[工作流](/docs/zh-CN/workflows)运行同时执行的 Agent 数量，范围从 `1` 到 `256`。默认情况下，一次运行最多同时执行 16 个 Agent，当 Claude Code 可用的 CPU 较少时会更少；排队中的 `agent()` 调用会等待空闲槽位。每个正在运行的 Agent 的会话记录都保存在 Claude Code 的内存中，因此较高的值会增加内存使用量。仅接受纯数字；超出范围的值和其他写法会保留默认值。需要 Claude Code v2.1.269 或更高版本 |
| `CLAUDE_CODE_WORKFLOW_PREFIX_STAGGER_MS` | [工作流](/docs/zh-CN/workflows) Agent 在发送自己的第一个请求之前，等待具有相同前缀的同级 Agent 的第一个响应开始的时间上限（以毫秒为单位）。当扇出启动多个共享同一[提示缓存前缀](/docs/zh-CN/workflows#prompt-caching-in-a-fan-out)的 Agent 时，Claude Code 会让除第一个 Agent 之外的所有 Agent 最多等待这么长时间，以便其余 Agent 读取已缓存的前缀，而不是各自在未缓存的情况下处理它。默认值为 `5000`。设置为 `0` 可禁用等待。设置了 `DISABLE_PROMPT_CACHING` 时，Agent 从不等待。需要 Claude Code v2.1.229 或更高版本 |
| `CLAUDE_CONFIG_DIR` | 覆盖配置目录（默认：`~/.claude`）。所有设置、会话历史和插件都存储在此路径下。有关凭据，请参阅 [Claude Code 存储凭据的位置](/docs/zh-CN/authentication#credential-management)。适用于同时运行多个账户：例如 `alias claude-work='CLAUDE_CONFIG_DIR=~/.claude-work claude'`。在您的 shell、用户设置或托管设置中设置它。在[项目和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中会被忽略 |
| `CLAUDE_DISABLE_ADOPT` | 设置为 `1` 后，当您按 `←` 或使用 [`/background`](/docs/zh-CN/agent-view#from-inside-a-session) 将会话转入后台时，会停止正在进行的后台工作，而不是将其延续。Claude Code 会在转入后台之前请您确认，然后停止原本会被延续的任务。需要 Claude Code v2.1.195 或更高版本 |
| `CLAUDE_EFFORT` | 在 Bash 工具子进程和 hook 命令中自动设置为子进程启动时生效的 [effort 级别](/docs/zh-CN/model-config#adjust-effort-level)：`low`、`medium`、`high`、`xhigh` 或 `max`。与传递给 [hook](/docs/zh-CN/hooks) 的 `effort.level` 字段一致。仅在当前模型支持 effort 参数时设置 |
| `CLAUDE_ENABLE_BYTE_WATCHDOG` | 设置为 `1` 可强制启用字节级流式空闲看门狗，设置为 `0` 可强制禁用。`0` 还会在运行[首字节截止时间](/docs/zh-CN/network-config#streaming-idle-watchdogs)的连接上关闭该截止时间。未设置时，默认对直连 Anthropic API 和 [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 的连接启用看门狗，也对通过 `ANTHROPIC_BASE_URL` 或 `ANTHROPIC_AWS_BASE_URL` 访问的[网关](/docs/zh-CN/gateways)连接上的流式响应启用；在 v2.1.222 之前，它不会在这些网关连接上运行，因此即使保活 ping 仍在到达，事件级看门狗也可能在那里报告停滞。有关超时以及各计时器如何相互作用，请参阅[流式空闲看门狗](/docs/zh-CN/network-config#streaming-idle-watchdogs) |
| `CLAUDE_ENABLE_BYTE_WATCHDOG_BEDROCK` | 设置为 `1` 可在 Amazon Bedrock `vnd.amazon.eventstream` 响应上启用字节级流式空闲看门狗，这也会在 Bedrock 流式请求上启用[首字节截止时间](/docs/zh-CN/network-config#streaming-idle-watchdogs)。默认关闭。使用 `CLAUDE_STREAM_IDLE_TIMEOUT_MS` 配置超时时间 |
| `CLAUDE_ENABLE_STREAM_WATCHDOG` | 设置为 `0` 可强制禁用事件级流式空闲看门狗，设置为 `1` 可强制启用。未设置时，看门狗默认对所有提供商开启。在 v2.1.196 之前，未设置时的默认值在直连 Anthropic API 上由服务器控制，在其他提供商上为关闭。使用 `CLAUDE_STREAM_IDLE_TIMEOUT_MS` 配置超时时间；有关与其并行运行的其他停滞计时器，请参阅[流式空闲看门狗](/docs/zh-CN/network-config#streaming-idle-watchdogs) |
| `CLAUDE_ENV_FILE` | shell 脚本的路径，Claude Code 会在同一 shell 进程中于每个 Bash 命令之前运行其内容，因此文件中的导出对该命令可见。用于在命令之间保持 virtualenv 或 conda 的激活状态。也会由 [SessionStart](/docs/zh-CN/hooks#persist-environment-variables)、[Setup](/docs/zh-CN/hooks#setup)、[CwdChanged](/docs/zh-CN/hooks#cwdchanged) 和 [FileChanged](/docs/zh-CN/hooks#filechanged) hook 动态填充 |
| `CLAUDE_JOB_DIR` | 由 Claude Code 在每个[后台会话](/docs/zh-CN/agent-view)中设置为该会话的 `~/.claude/jobs/<id>` 目录。该会话运行的 shell 命令会继承它。请将临时文件写入 [`$CLAUDE_JOB_DIR/tmp`](/docs/zh-CN/agent-view#where-state-is-stored)。Claude 在该目录中的 `Write` 和 `Edit` 调用不会请求权限，并且该目录会在会话被删除时移除 |
| `CLAUDE_PID` | Claude Code 在其派生的子进程中将此变量设置为自身的进程 ID，这些子进程包括：Bash 和 PowerShell 工具命令以及 hook 命令。在 Linux 上，Bash 工具的 shell 集成使用它来拒绝会匹配 Claude Code 进程本身的 `pkill` 模式；请参阅[错误参考](/docs/zh-CN/errors#pkill-pattern-matches-the-claude-code-process)。可在您自己的脚本中读取它，以有意识地识别父 Claude Code 进程或向其发送信号。需要 Claude Code v2.1.214 或更高版本 |
| `CLAUDE_REMOTE_CONTROL_SESSION_NAME_PREFIX` | 未提供显式名称时，自动生成的 [Remote Control](/docs/zh-CN/remote-control) 会话名称的前缀。默认为您机器的主机名，生成类似 `myhost-graceful-unicorn` 的名称。`--remote-control-session-name-prefix` CLI 标志可为单次调用设置相同的值 |
| `CLAUDE_STREAM_FIRST_BYTE_TIMEOUT_MS` | 在运行[首字节截止时间](/docs/zh-CN/network-config#streaming-idle-watchdogs)的连接上，流式请求第一个响应字节的截止时间（以毫秒为单位）。有关 Claude Code 如何限制该值、它为大型请求体增加的额外时间，以及未设置时如何选择截止时间，请参阅 [No response from API](/docs/zh-CN/errors#no-response-from-api)。需要 Claude Code v2.1.242 或更高版本 |
| `CLAUDE_STREAM_IDLE_TIMEOUT_MS` | 事件级和字节级流式空闲看门狗关闭停滞连接之前的超时时间（以毫秒为单位）。当您显式设置此变量时，最小值为 `300000`（5 分钟）；较低的值会被静默提升至该最小值，以容纳扩展思考停顿和代理缓冲，并且字节级看门狗会将该值的上限设为 30 分钟。对于字节级看门狗，`CLAUDE_BYTE_STREAM_IDLE_TIMEOUT_MS` 优先于此变量。有关各看门狗在未设置时的默认值，请参阅[流式空闲看门狗](/docs/zh-CN/network-config#streaming-idle-watchdogs) |
| `CLAUDE_SUBAGENT_BG_SHELL_MAX_MS` | 已在 v2.1.260 中移除，现在不起任何作用。此前用于限制由[子代理](/docs/zh-CN/sub-agents)启动的[后台 shell 命令](/docs/zh-CN/interactive-mode#background-bash-commands)可运行的时长（以毫秒为单位），默认为 60 分钟。请参阅[后台命令生命周期规则](/docs/zh-CN/tools-reference#background-commands) |
| `DEBUG` | 设置为 `1` 可启用调试模式，等同于使用 [`--debug`](/docs/zh-CN/cli-reference#cli-flags) 启动。调试日志会写入 `~/.claude/debug/<session-id>.txt`，或写入 `CLAUDE_CODE_DEBUG_LOGS_DIR` 设置的路径。只有真值 `1`、`true`、`yes` 和 `on` 才会启用调试模式，因此为其他工具设置的命名空间模式（例如 `DEBUG=express:*`）不会触发它 |
| `DISABLE_AUTOUPDATER` | 设置为 `1` 可禁用自动后台更新。手动执行 `claude update` 仍然有效。使用 `DISABLE_UPDATES` 可同时阻止两者 |
| `DISABLE_AUTO_COMPACT` | 设置为 `1` 可禁用接近上下文限制时的自动压缩。手动 `/compact` 命令仍然可用。适用于希望明确控制何时进行压缩的情况。覆盖 [`autoCompactEnabled`](/docs/zh-CN/settings-reference#autocompactenabled) 设置 |
| `DISABLE_COMPACT` | 设置为 `1` 可禁用所有压缩：包括自动压缩和手动 `/compact` 命令 |
| `DISABLE_COST_WARNINGS` | 设置为 `1` 可禁用费用警告消息 |
| `DISABLE_DOCTOR_COMMAND` | 设置为 `1` 可隐藏 [`/doctor`](/docs/zh-CN/commands#all-commands) 安装检查 skill 及其 `/checkup` 别名。适用于不应让用户在会话中运行安装诊断的托管部署。不影响 `claude doctor` 终端命令。在 v2.1.205 之前，此变量隐藏的是 `/doctor` 诊断屏幕命令 |
| `DISABLE_ERROR_REPORTING` | 设置为任意非空值（例如 `1`）可选择退出错误报告。**设置为 `0` 或 `false` 仍会选择退出**，这与大多数开关变量不同；取消设置该变量可重新启用错误报告 |
| `DISABLE_EXTRA_USAGE_COMMAND` | 设置为 `1` 可隐藏 `/usage-credits` 命令，该命令允许用户购买超出速率限制的额外用量 |
| `DISABLE_FEEDBACK_COMMAND` | 设置为 `1` 可禁用 `/feedback` 命令和 [Claude 起草的反馈](/docs/zh-CN/tools-reference#sendfeedback-tool-behavior)。还会禁用通过同一途径提交报告的 `/bug` 和 `/share`；在 v2.1.212 之前，它们是 `/feedback` 的别名，因此该命令在所有名称下都会被禁用。也接受旧名称 `DISABLE_BUG_COMMAND` |
| `DISABLE_GROWTHBOOK` | 设置为 `1` 或 `true` 可禁用 GrowthBook 功能标志获取，并对每个标志使用代码中的默认值。这会使 [Remote Control](/docs/zh-CN/remote-control#requirements) 以及其他[需要获取功能标志的功能](#features-that-need-feature-flag-fetching)不可用。设置为 `0` 或 `false` 会保持获取开启。除非同时设置了 `DISABLE_TELEMETRY`，否则遥测事件日志记录保持开启 |
| `DISABLE_INSTALLATION_CHECKS` | 设置为 `1` 可禁用安装警告。仅在手动管理安装位置时使用，因为这可能会掩盖标准安装中的问题 |
| `DISABLE_INSTALL_GITHUB_APP_COMMAND` | 设置为 `1` 可隐藏 `/install-github-app` 命令。使用第三方提供商（Amazon Bedrock、Google Cloud's Agent Platform 或 Microsoft Foundry）时该命令已被隐藏 |
| `DISABLE_INTERLEAVED_THINKING` | 设置为 `1` 可阻止发送 interleaved-thinking beta 标头。适用于您的 LLM 网关或提供商不支持[交错思考](https://platform.claude.com/docs/en/build-with-claude/extended-thinking#interleaved-thinking)的情况 |
| `DISABLE_LOGIN_COMMAND` | 设置为 `1` 可隐藏 `/login` 命令。适用于通过 API 密钥或 `apiKeyHelper` 在外部处理身份验证的情况 |
| `DISABLE_LOGOUT_COMMAND` | 设置为 `1` 可隐藏 `/logout` 命令 |
| `DISABLE_PROMPT_CACHING` | 设置为 `1` 可为所有模型禁用[提示缓存](/docs/zh-CN/prompt-caching#disable-prompt-caching)（优先于按模型的设置） |
| `DISABLE_PROMPT_CACHING_FABLE` | 设置为 `1` 可为 Fable 模型禁用提示缓存 |
| `DISABLE_PROMPT_CACHING_HAIKU` | 设置为 `1` 可为[默认 Haiku 模型](/docs/zh-CN/prompt-caching#disable-prompt-caching)禁用提示缓存，无论其在何处运行 |
| `DISABLE_PROMPT_CACHING_OPUS` | 设置为 `1` 可为[默认 Opus 模型](/docs/zh-CN/prompt-caching#disable-prompt-caching)禁用提示缓存 |
| `DISABLE_PROMPT_CACHING_SONNET` | 设置为 `1` 可为[默认 Sonnet 模型](/docs/zh-CN/prompt-caching#disable-prompt-caching)禁用提示缓存 |
| `DISABLE_TELEMETRY` | 设置为任意非空值（例如 `1`）可选择退出遥测。**设置为 `0` 或 `false` 仍会选择退出**，这与大多数开关变量不同；取消设置该变量可重新启用遥测。遥测事件不包含代码、文件路径或 Bash 命令等用户数据。还会禁用[功能标志获取](#features-that-need-feature-flag-fetching)。请参阅[为您的组织关闭遥测](/docs/zh-CN/managed-settings#turn-telemetry-off-for-your-organization) |
| `DISABLE_UPDATES` | 设置为 `1` 可阻止所有更新，包括手动执行的 `claude update` 和 `claude install`。比 `DISABLE_AUTOUPDATER` 更严格。适用于通过您自己的渠道分发 Claude Code 且用户不应自行更新的情况 |
| `DISABLE_UPGRADE_COMMAND` | 设置为 `1` 可隐藏 `/upgrade` 命令 |
| `DO_NOT_TRACK` | 设置为 `1` 可选择退出遥测，效果与 `DISABLE_TELEMETRY` 相同，包括对[功能标志获取](#features-that-need-feature-flag-fetching)的影响。Claude Code 将此变量作为标准布尔值读取，因此 `0` 会保持遥测开启；Claude Code 遵循此变量，是因为它是许多开发者 CLI 都认可的跨工具约定 |
| `ENABLE_BETA_TRACING_DETAILED` | 设置为 `1`，并将 `BETA_TRACING_ENDPOINT` 设置为您的 OTLP/HTTP 收集器端点，即可开启[详细 beta 追踪](/docs/zh-CN/monitoring-usage#traces-beta)，这会添加包含内容的 span 属性和 `claude_code.hook` span。交互式 CLI 会话还要求您的组织已列入该 beta 的允许列表。这两个变量在[项目设置和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中都会被忽略 |
| `ENABLE_CLAUDEAI_MCP_SERVERS` | 设置为 `false` 可阻止 Claude Code 获取 [claude.ai MCP 服务器](/docs/zh-CN/mcp#use-mcp-servers-from-claude-ai)。对已登录用户默认启用。若要按项目或按组织禁用，请改为在设置中设置 [`disableClaudeAiConnectors`](/docs/zh-CN/settings-reference#disableclaudeaiconnectors) |
| `ENABLE_PROMPT_CACHING_1H` | 设置为 `1` 可请求 1 小时的[提示缓存 TTL](/docs/zh-CN/prompt-caching#cache-lifetime)，而不是默认的 5 分钟。适用于 API 密钥、[Amazon Bedrock](/docs/zh-CN/amazon-bedrock)、[Google Cloud's Agent Platform](/docs/zh-CN/google-vertex-ai)、[Microsoft Foundry](/docs/zh-CN/microsoft-foundry) 和 [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 用户。在所含用量范围内的订阅用户会在[主对话](/docs/zh-CN/prompt-caching#which-ttl-each-request-gets)上自动获得 1 小时 TTL。正在使用[使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans)的订阅用户可以设置此变量以保持 1 小时 TTL。1 小时缓存写入按更高费率计费。若要改为按请求类别选择 TTL，请使用 `CLAUDE_CODE_PROMPT_CACHE_TTL` 和 `CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL`，它们优先于此变量 |
| `ENABLE_PROMPT_CACHING_1H_BEDROCK` | 已弃用。请改用 `ENABLE_PROMPT_CACHING_1H` |
| `ENABLE_TOOL_SEARCH` | 控制 [MCP 工具搜索](/docs/zh-CN/mcp#scale-with-mcp-tool-search)。未设置时，Claude Code 默认延迟加载所有 MCP 工具。但在早于 Claude 4.5 代的 Google Cloud's Agent Platform 模型上、在托管于 Azure 的 Microsoft Foundry 部署上，以及当 `ANTHROPIC_BASE_URL` 指向非第一方主机时，仍会预先加载这些工具。`true` 始终延迟加载并发送 beta 标头，但上述 Agent Platform 模型和 Microsoft Foundry 部署除外；在不支持 `tool_reference` 的代理上，请求会失败。`auto` 在工具定义所占空间不超过上下文的 10% 时预先加载。`auto:N` 设置自定义阈值，例如 `auto:5` 表示 5%。`false` 预先加载所有工具。设置了 `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` 时，您自行设置的值会被忽略。在 v2.1.221 之前，除非您将此变量设置为 `true`，否则 Claude Code 会在 Google Cloud's Agent Platform 上为所有模型禁用工具搜索 |
| `FALLBACK_FOR_ALL_PRIMARY_MODELS` | 设置为任意非空值（例如 `1`），可使 Claude Code 在未配置备用模型时，对所有模型在遇到重复过载错误时停止重试。**设置为 `0` 或 `false` 仍会启用此行为**，这与大多数开关变量不同；取消设置该变量可恢复默认的重试行为。若不设置，当您使用 API 密钥或[第三方提供商](/docs/zh-CN/third-party-integrations)而非 Claude 订阅进行身份验证时，Claude Code 仅在其识别为 Opus、Fable 或 Mythos 的模型上以这种方式停止重试。在 Claude Code v2.1.160 或更高版本上，Claude Code 会在任何主模型遇到重复过载错误时切换到您配置的[备用模型链](/docs/zh-CN/model-config#fallback-model-chains)，因此此变量不影响切换到备用模型 |
| `FORCE_AUTOUPDATE_PLUGINS` | 设置为 `1` 可强制插件自动更新，即使主自动更新程序已通过 `DISABLE_AUTOUPDATER` 禁用 |
| `FORCE_HYPERLINK` | 设置为 `1` 可在您的终端支持但未被自动检测到时启用可点击的 OSC 8 超链接，设置为 `0` 可禁用超链接。未设置时，Claude Code 仅在检测到终端支持时启用超链接。Claude Code 将此值解析为数字而非布尔值，因此 `false`、`no` 或 `off` 等值会启用超链接而不是禁用。即使 Claude Code 无法检测到终端支持（例如通过 SSH 时），页脚中的 [PR 或合并请求徽章](/docs/zh-CN/interactive-mode#pr-review-status)也会渲染为超链接。设置为 `0` 可将该徽章渲染为纯文本 |
| `FORCE_PROMPT_CACHING_5M` | 设置为 `1` 可强制使用 5 分钟的提示缓存 TTL，即使原本会应用 1 小时 TTL。覆盖 `CLAUDE_CODE_PROMPT_CACHE_TTL`、`CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL`、`ENABLE_PROMPT_CACHING_1H` 以及 `promptCacheTtl` 和 `subagentPromptCacheTtl` 设置 |
| `HTTP_PROXY` | 为网络连接指定 HTTP 代理服务器 |
| `HTTPS_PROXY` | 为网络连接指定 HTTPS 代理服务器 |
| `IS_DEMO` | 设置为任意非空值（例如 `1`）可启用演示模式：在标题栏和 `/status` 输出中隐藏您的电子邮件和组织名称，并跳过新手引导。**设置为 `0` 或 `false` 仍会启用演示模式**，这与大多数开关变量不同；取消设置该变量可将其关闭。适用于直播或录制会话时 |
| `MAX_MCP_OUTPUT_TOKENS` | MCP 工具响应中允许的最大 token 数（默认：25000）。当输出超过 10,000 个 token 时，Claude Code 会显示警告。声明了 [`anthropic/maxResultSizeChars`](/docs/zh-CN/mcp#raise-the-limit-for-a-specific-tool) 的工具会改为对文本内容使用该字符限制，但这些工具返回的图像内容仍受此变量约束。对于没有该注解的工具，超过 50,000 个字符的成功文本结果无论此变量如何设置，都会被[保存到文件](/docs/zh-CN/mcp#mcp-output-limits-and-warnings) |
| `MAX_STRUCTURED_OUTPUT_RETRIES` | 在使用 `-p` 标志的非交互模式下，当模型响应未通过 [`--json-schema`](/docs/zh-CN/cli-reference#cli-flags) 验证时，Claude Code 允许的尝试次数；在达到该次数的失败尝试且没有有效输出后，运行失败。当[工作流](/docs/zh-CN/workflows)子代理的结构化输出未通过验证时，也适用相同的上限。默认为 5，即一次初始尝试加四次重试 |
| `MAX_THINKING_TOKENS` | [扩展思考](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)的固定 token 预算。Claude Code 将其上限设为比请求的最大输出 token 数少一个 token，且绝不低于 1,024。有关该限制的设置方式，请参阅 `CLAUDE_CODE_MAX_OUTPUT_TOKENS`。未设置且已启用思考时，支持[自适应推理](/docs/zh-CN/model-config#adjust-effort-level)的模型会自行选择思考深度，其他模型则使用该上限。设置为 `0` 可在 Anthropic API 上禁用思考，但 Opus 5.5、Sonnet 5.5 和 Fable 模型除外，这些模型无法关闭思考。在[第三方提供商](/docs/zh-CN/third-party-integrations)上，`0` 会改为省略 `thinking` 参数。在 Anthropic API 上关闭思考时，对于 Claude Code 已知[不接受该组合](/docs/zh-CN/errors#effort-isnt-available-with-thinking-turned-off)的模型（例如 Opus 5），Claude Code 会发送 effort `high` 而不是更高的级别。对于正值，Claude Code 在自适应推理模型上会忽略该数字本身，除非 `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` 关闭了自适应推理 |
| `MCP_CLIENT_SECRET` | 用于需要[预配置凭据](/docs/zh-CN/mcp#use-pre-configured-oauth-credentials)的 MCP 服务器的 OAuth 客户端密钥。使用 `--client-secret` 添加服务器时可避免交互式提示 |
| `MCP_CONNECTION_NONBLOCKING` | 控制启动时是否在第一次查询之前等待 MCP 服务器连接。MCP 启动默认为非阻塞：服务器在后台连接，其工具在连接完成后即可使用。设置为 `0` 可使 Claude Code 在第一次查询之前等待服务器连接。配置了 [`alwaysLoad: true`](/docs/zh-CN/mcp#exempt-a-server-from-deferral) 的服务器无论如何仍会使启动等待，除非其由[发现缓存](/docs/zh-CN/mcp#server-status-detail)提供，因为构建第一个提示词时其工具必须已经存在。在不带 `--input-format stream-json` 的非交互模式（`-p`）下，无论此变量如何设置，Claude Code 也会在第一个轮次之前等待仍处于挂起状态的服务器。当您显式传递 [`--mcp-config`](/docs/zh-CN/cli-reference#cli-flags) 时，等待的截止时间更长；有关已缓存服务器的例外情况，请参阅该标志的条目 |
| `MCP_CONNECT_TIMEOUT_MS` | 阻塞式 MCP 启动在生成工具列表快照之前等待连接批次的时长（以毫秒为单位，默认：5000）。适用于 `MCP_CONNECTION_NONBLOCKING=0` 时，或标记为 [`alwaysLoad: true`](/docs/zh-CN/mcp#exempt-a-server-from-deferral) 的服务器。截止时仍处于挂起状态的服务器会继续在后台连接。不同于 `MCP_TIMEOUT`，后者限制的是单个服务器的连接尝试 |
| `MCP_DISCOVERY_CACHE` | 开启或关闭 [MCP 发现缓存](/docs/zh-CN/mcp#server-status-detail)。开启缓存后，您之前使用过的远程 HTTP 或 SSE 服务器可以显示 [`cached` 状态](/docs/zh-CN/mcp#server-status-detail)，Claude Code 会在其第一次工具调用时连接该服务器，而不是在启动时连接。缓存默认关闭，除非逐步推出已为您的账户启用了它。设置为 `1` 可将其开启，设置为 `0` 可在逐步推出已启用它的情况下仍保持关闭。在 v2.1.238 之前，缓存默认开启。`cached` 状态需要 Claude Code v2.1.221 或更高版本 |
| `MCP_DISCOVERY_CACHE_MAX_STALE_S` | [发现缓存](/docs/zh-CN/mcp#server-status-detail)条目的最长存留时间（以秒为单位）（默认：14400，即 4 小时）。在某次启动时，如果条目的存留时间超过该值，Claude Code 会将其丢弃并在启动时连接服务器，与关闭缓存时的行为相同。Claude Code 将该值的上限设为 7 天。在 v2.1.238 之前，默认值为 86400，即 24 小时，且 Claude Code 不限制该值 |
| `MCP_DISCOVERY_CACHE_STRIKES` | 在某次启动时，如果[发现缓存](/docs/zh-CN/mcp#server-status-detail)条目的存留时间超过 `MCP_DISCOVERY_CACHE_TTL_S`，Claude Code 会在后台刷新它。此变量设置在 Claude Code 丢弃该条目并改为在下次启动时连接服务器之前，允许连续失败的刷新次数（默认：1）。如果您的网络连接偶尔中断，可以调高此值，以免一次刷新失败就丢弃该条目。需要 Claude Code v2.1.238 或更高版本 |
| `MCP_DISCOVERY_CACHE_TTL_S` | Claude Code 使用[发现缓存](/docs/zh-CN/mcp#server-status-detail)条目而不刷新它的秒数（默认：900）。在某次启动时，如果条目的存留时间超过该值，Claude Code 仍会使用它，但会在后台刷新。一旦条目的存留时间超过 `MCP_DISCOVERY_CACHE_MAX_STALE_S`，Claude Code 会改为将其丢弃。Claude Code 将该值的上限设为 `MCP_DISCOVERY_CACHE_MAX_STALE_S`，默认为 4 小时。在 v2.1.238 之前，Claude Code 不限制该值 |
| `MCP_OAUTH_CALLBACK_PORT` | OAuth 重定向回调的固定端口，在使用[预配置凭据](/docs/zh-CN/mcp#use-pre-configured-oauth-credentials)添加 MCP 服务器时，可作为 `--callback-port` 的替代方案 |
| `MCP_PROTOCOL_NEGOTIATION` | 仅在 [v2 MCP 客户端运行时](/docs/zh-CN/mcp#mcp-client-runtimes)上有效，控制 Claude Code 是否探测服务器对 MCP 协议修订版 2026-07-28 的支持。设置为 `auto` 可探测 HTTP、claude.ai 连接器和 stdio 服务器，设置为 `legacy` 则不探测任何服务器。未设置该变量时，Claude Code 会探测 [MCP 客户端运行时](/docs/zh-CN/mcp#mcp-client-runtimes)中所述的服务器。任何其他值都会被忽略，并在调试日志中写入警告。需要 Claude Code v2.1.221 或更高版本 |
| `MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE` | 启动期间并行连接的远程 MCP 服务器（HTTP/SSE）的最大数量（默认：20） |
| `MCP_SDK_GENERATION` | 固定此进程用于连接 MCP 服务器的 [MCP 客户端运行时](/docs/zh-CN/mcp#mcp-client-runtimes)：`v1`，基于 MCP TypeScript SDK 1.x 构建；或 `v2`，基于 [MCP TypeScript SDK 2.0](https://ts.sdk.modelcontextprotocol.io/v2/) 构建。未设置该变量时，Claude Code 从该部分所列的版本开始使用 v2。在 Claude Code v2.1.221 或更高版本上，v2 运行时会检查 MCP OAuth 服务器在其授权响应中返回的颁发者，如果不匹配，登录会失败，并显示以 `Issuer mismatch in authorization response` 开头的错误。v1 运行时不执行此检查。如果您设置了无法识别的值，Claude Code 会忽略它并向调试日志写入警告。Claude Code 每个进程只读取一次该值。需要 Claude Code v2.1.218 或更高版本 |
| `MCP_SERVER_CONNECTION_BATCH_SIZE` | 启动期间并行连接的本地 MCP 服务器（stdio）的最大数量（默认：3） |
| `MCP_TIMEOUT` | MCP 服务器启动的超时时间（以毫秒为单位，默认：30000，即 30 秒） |
| `MCP_TOOL_TIMEOUT` | MCP 工具执行的超时时间（以毫秒为单位，默认：100000000，约 28 小时）。对于 HTTP、SSE 或 claude.ai 连接器服务器，每个请求默认还会在 60 秒后超时；将此变量或单个服务器的 `timeout` 设置为高于 60000 的值，可提高该单请求限制。较低的值仍会缩短整体工具执行超时时间，但单请求限制保持为 60 秒。Stdio 和 WebSocket 服务器没有单请求计时器。`.mcp.json` 中单个服务器的 `timeout` 字段会为该服务器覆盖此值。单个服务器的 `timeout` 若至少为 1000，还会为该服务器的工具调用设置最小空闲窗口，因此 `CLAUDE_CODE_MCP_TOOL_IDLE_TIMEOUT` 绝不会更早中止这些调用；此下限需要 Claude Code v2.1.203 或更高版本。对于环境变量，低于 1000 的值会被提升为一秒；对于单个服务器的字段，低于 1000 的值会被忽略 |
| `NO_PROXY` | 直接发出请求、绕过代理的域名和 IP 列表 |
| `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` | 标准 OpenTelemetry SDK 对属性值长度的限制。Claude Code 会将携带内容的遥测属性上限设为此值与 `CLAUDE_CODE_OTEL_CONTENT_MAX_LENGTH` 中的较小者，以使截断标记保持在 SDK 限制之内。Claude Code 以相同方式读取 `OTEL_LOGRECORD_ATTRIBUTE_VALUE_LENGTH_LIMIT` 和 `OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT` 变体，且所设置的最小值适用于所有信号。需要 Claude Code v2.1.214 或更高版本。请参阅[监控](/docs/zh-CN/monitoring-usage#common-configuration-variables) |
| `OTEL_LOG_ASSISTANT_RESPONSES` | 设置为 `1` 可在 `assistant_response` OpenTelemetry 日志事件中包含模型的回复文本。未设置时，Claude Code 改用 `OTEL_LOG_USER_PROMPTS` 的值。设置为 `0` 可在设置了 `OTEL_LOG_USER_PROMPTS` 的情况下仍保持回复脱敏。在您的 shell、用户设置或托管设置中设置它。在[项目和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中会被忽略。需要 Claude Code v2.1.193 或更高版本。请参阅[监控](/docs/zh-CN/monitoring-usage#assistant-response-event) |
| `OTEL_LOG_MANAGED_SETTINGS` | 设置为 `1` 可将脱敏后的托管设置以及脱敏前设置的 SHA-256 摘要添加到 `managed_settings_resolved` OpenTelemetry 日志事件中。默认禁用。在您的 shell、用户设置或托管设置中设置它；项目或本地设置中的值不会将其开启。需要 Claude Code v2.1.274 或更高版本。请参阅[监控](/docs/zh-CN/monitoring-usage#managed-settings-resolved-event) |
| `OTEL_LOG_RAW_API_BODIES` | 将 Anthropic Messages API 请求和响应 JSON 作为 `api_request_body` / `api_response_body` 日志事件发出。设置为 `1` 可发出在内容限制处截断的内联正文，或设置为 `file:<dir>` 将未截断的正文写入磁盘，并改为发出 `body_ref` 路径。`CLAUDE_CODE_OTEL_CONTENT_MAX_LENGTH` 用于配置内容限制，默认为 60 KB。默认禁用；正文包含完整的对话历史。在您的 shell、用户设置或托管设置中设置它。在[项目和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中会被忽略。请参阅[监控](/docs/zh-CN/monitoring-usage#api-request-body-event) |
| `OTEL_LOG_TOOL_CONTENT` | 设置为 `1` 可在 `tool.output` OpenTelemetry span 事件中包含工具内容。span 属性在[各自的开关](/docs/zh-CN/monitoring-usage#new-context-gates)控制下携带工具内容。需要启用[追踪](/docs/zh-CN/monitoring-usage#traces-beta)。默认禁用以保护敏感数据。在您的 shell、用户设置或托管设置中设置它。在[项目和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中会被忽略，但该部分所述的关闭值除外。请参阅[监控](/docs/zh-CN/monitoring-usage#tool-output-span-event) |
| `OTEL_LOG_TOOL_DETAILS` | 设置为 `1` 可在 OpenTelemetry 指标、追踪和日志中包含工具输入参数；MCP 服务器名称；用户编写的工作流名称；工具失败时的原始错误字符串；`api_refusal` 事件上的拒绝 `category`；[费用和 token 指标](/docs/zh-CN/monitoring-usage#cost-counter)上真实的 Agent、skill、插件和 MCP 服务器名称；以及其他工具详细信息。默认禁用以保护 PII。在您的 shell、用户设置或托管设置中设置它。在[项目和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中会被忽略，但该部分所述的关闭值除外。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `OTEL_LOG_USER_PROMPTS` | 设置为 `1` 可在 OpenTelemetry 追踪和日志中包含用户提示词文本。默认禁用（提示词会被脱敏）。在您的 shell、用户设置或托管设置中设置它。在[项目和本地设置](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)中会被忽略，但该部分所述的关闭值除外。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | 设置为 `false` 可从指标属性中排除账户 UUID（默认：包含）。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `OTEL_METRICS_INCLUDE_ENTRYPOINT` | 设置为 `true` 可在指标属性中包含会话入口点（默认：排除）。在 v2.1.152 中添加。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `OTEL_METRICS_INCLUDE_REPOSITORY` | 设置为 `true` 可为 OpenTelemetry 指标和事件添加标识会话所在仓库的 `vcs.*` 属性（默认：排除）。需要 Claude Code v2.1.269 或更高版本。请参阅[仓库属性](/docs/zh-CN/monitoring-usage#repository-attributes) |
| `OTEL_METRICS_INCLUDE_RESOURCE_ATTRIBUTES` | 从 v2.1.161 起，Claude Code 会将 `OTEL_RESOURCE_ATTRIBUTES` 中的键附加到指标数据点标签上。设置为 `false` 可排除它们（默认：包含）。请参阅[监控](/docs/zh-CN/monitoring-usage#multi-team-organization-support) |
| `OTEL_METRICS_INCLUDE_SESSION_ID` | 设置为 `false` 可从指标属性中排除会话 ID（默认：包含）。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `OTEL_METRICS_INCLUDE_VERSION` | 设置为 `true` 可在指标属性中包含 Claude Code 版本（默认：排除）。请参阅[监控](/docs/zh-CN/monitoring-usage) |
| `SLASH_COMMAND_TOOL_CHAR_BUDGET` | 覆盖向 [Skill 工具](/docs/zh-CN/skills#control-who-invokes-a-skill)显示的 skill 元数据的字符预算。该预算按上下文窗口的 1% 动态缩放，备用值为 8,000 个字符。保留旧名称是为了向后兼容 |
| `TASK_MAX_OUTPUT_LENGTH` | 已在 v2.1.277 中移除，现在不起任何作用，其所控制大小的 `TaskOutput` 工具也一并移除。此前用于设置 `TaskOutput` 工具保留的[后台任务](/docs/zh-CN/tools-reference#background-commands)输出的最大字符数。Claude 现在改为使用 `Read` 读取后台任务的输出文件 |
| `USE_BUILTIN_RIPGREP` | 设置为 `0` 可使用系统安装的 `rg`，而不是 Claude Code 附带的 `rg` |
| `VERTEX_REGION_CLAUDE_3_5_HAIKU` | 使用 Google Cloud's Agent Platform 时覆盖 Claude 3.5 Haiku 的区域 |
| `VERTEX_REGION_CLAUDE_3_5_SONNET` | 使用 Google Cloud's Agent Platform 时覆盖 Claude 3.5 Sonnet 的区域 |
| `VERTEX_REGION_CLAUDE_3_7_SONNET` | 使用 Google Cloud's Agent Platform 时覆盖 Claude 3.7 Sonnet 的区域 |
| `VERTEX_REGION_CLAUDE_4_0_OPUS` | 使用 Google Cloud's Agent Platform 时覆盖 Claude 4.0 Opus 的区域 |
| `VERTEX_REGION_CLAUDE_4_0_SONNET` | 使用 Google Cloud's Agent Platform 时覆盖 Claude 4.0 Sonnet 的区域 |
| `VERTEX_REGION_CLAUDE_4_1_OPUS` | 使用 Google Cloud's Agent Platform 时覆盖 Claude 4.1 Opus 的区域 |
| `VERTEX_REGION_CLAUDE_4_5_OPUS` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Opus 4.5 的区域 |
| `VERTEX_REGION_CLAUDE_4_5_SONNET` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Sonnet 4.5 的区域 |
| `VERTEX_REGION_CLAUDE_4_6_OPUS` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Opus 4.6 的区域 |
| `VERTEX_REGION_CLAUDE_4_6_SONNET` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Sonnet 4.6 的区域 |
| `VERTEX_REGION_CLAUDE_4_7_OPUS` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Opus 4.7 的区域 |
| `VERTEX_REGION_CLAUDE_4_8_OPUS` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Opus 4.8 的区域 |
| `VERTEX_REGION_CLAUDE_5_5_OPUS` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Opus 5.5 的区域。在 v2.1.280 中添加 |
| `VERTEX_REGION_CLAUDE_5_5_SONNET` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Sonnet 5.5 的区域。在 v2.1.284 中添加 |
| `VERTEX_REGION_CLAUDE_5_OPUS` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Opus 5 的区域。在 v2.1.219 中添加 |
| `VERTEX_REGION_CLAUDE_5_SONNET` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Sonnet 5 的区域。在 v2.1.197 中添加 |
| `VERTEX_REGION_CLAUDE_FABLE_5` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Fable 5 的区域。在 v2.1.170 中添加 |
| `VERTEX_REGION_CLAUDE_FABLE_5_1` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Fable 5.1 的区域。在 v2.1.257 中添加 |
| `VERTEX_REGION_CLAUDE_HAIKU_4_5` | 使用 Google Cloud's Agent Platform 时覆盖 Claude Haiku 4.5 的区域 |

还支持标准 OpenTelemetry 导出器变量（`OTEL_METRICS_EXPORTER`、`OTEL_LOGS_EXPORTER`、`OTEL_EXPORTER_OTLP_ENDPOINT`、`OTEL_EXPORTER_OTLP_PROTOCOL`、`OTEL_EXPORTER_OTLP_HEADERS`、`OTEL_METRIC_EXPORT_INTERVAL`、`OTEL_RESOURCE_ATTRIBUTES` 以及特定于信号的变体）。有关配置详情，请参阅[监控](/docs/zh-CN/monitoring-usage)。

请在您的 shell、用户设置或托管设置中设置 `CLAUDE_CODE_ENABLE_TELEMETRY`，以及用于启用导出、选择导出目标或捕获内容的 OpenTelemetry 变量。Claude Code [会在项目和本地设置中忽略这些变量](/docs/zh-CN/settings-reference#variables-claude-code-ignores-in-env)，但该部分所述的关闭值除外。`OTEL_RESOURCE_ATTRIBUTES` 以及导出间隔、超时和压缩相关变量（例如 `OTEL_METRIC_EXPORT_INTERVAL`）在项目和本地设置中仍然有效。

<h2 id="what-the-subprocess-environment-scrub-removes">
  子进程环境清理会移除哪些内容
</h2>

当您将 [`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`](#variables) 设置为 `1` 时，Claude Code 会从其启动的子进程（例如 Bash 命令、hook 和 stdio MCP 服务器）的环境中移除凭据。这可以减少提示词注入攻击通过 shell 展开所能读取的内容。Claude Code 进程本身会保留这些凭据，用于其自身的 API 调用。

清理功能通过变量名或变量值的形态来识别凭据，因此应将其作为与精细的[权限规则](/docs/zh-CN/permissions)配合使用的一层防护，而不是唯一的控制手段。

下表展示了清理功能对示例变量的处理方式：

| 示例变量 | 清理功能的处理方式 |
| :- | :- |
| `ANTHROPIC_API_KEY`、`AWS_SECRET_ACCESS_KEY` | 移除 |
| `NPM_TOKEN`、`DB_PASSWORD` | 移除，因为其名称看起来像凭据 |
| 包含密码的 `DATABASE_URL` | 移除，因为其值看起来像凭据 |
| 包含密码的 `PIP_INDEX_URL` 或 `NPM_CONFIG_REGISTRY` | 保留 URL，但从中删去用户名和密码 |
| `CLAUDE_CONFIG_DIR` | 移除。需要 Claude Code v2.1.251 或更高版本 |
| `GITHUB_TOKEN`、`GH_TOKEN`、`GH_ENTERPRISE_TOKEN`、`GITHUB_ENTERPRISE_TOKEN` | 保留，以便 `gh` 和调用 GitHub API 的脚本能够继续正常工作 |
| `HTTP_PROXY`、`HTTPS_PROXY` | 保留，包括 [URL 中的用户名和密码](/docs/zh-CN/network-config#basic-authentication)。[沙箱](/docs/zh-CN/sandboxing#network-isolation)可以自行为沙箱化的命令设置这些变量 |
| `GIT_CONFIG_COUNT`、`GIT_CONFIG_KEY_<n>`、`GIT_CONFIG_VALUE_<n>` | 保留，无论其内容为何 |
| 变量名和值都不像凭据的密钥 | 保留 |

由于清理功能会保留 `GITHUB_TOKEN`，请为 GitHub Actions 作业授予其所需的最小 `permissions`。要从沙箱化的 Bash 命令中移除 GitHub 令牌，请在 [`sandbox.credentials`](/docs/zh-CN/sandboxing#protect-credentials) 下添加一个 `deny` 条目。

如果某个子进程需要用到被移除的变量之一，请不要设置清理功能。

在 Linux 上，清理功能还会在隔离的 PID 命名空间中运行 Bash 子进程，使其无法通过 `/proc` 读取主机进程的环境。这带来的一个副作用是，`ps`、`pgrep` 和 `kill` 无法看到主机进程，也无法向其发送信号。

<h2 id="features-that-need-feature-flag-fetching">
  需要获取功能标志的功能
</h2>

Claude Code 通过从 Anthropic 获取的功能标志来启用部分功能。在以下会话中，Claude Code 会跳过该获取操作：

* 设置了 `DISABLE_GROWTHBOOK`、`DISABLE_TELEMETRY`、`DO_NOT_TRACK` 或 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 的会话；[变量表](#variables)中每个变量所在的行说明了哪些值会关闭获取
* 使用[第三方提供商](/docs/zh-CN/third-party-integrations)（例如 Amazon Bedrock、Claude Platform on AWS、Google Cloud 的 Agent Platform 或 Microsoft Foundry）的会话，除非嵌入 Claude Code 的宿主平台设置了 `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST`
* [Claude apps 网关](/docs/zh-CN/claude-apps-gateway)会话

关闭获取后，您将无法：

* 运行 [`/auto-mode-setup`](/docs/zh-CN/auto-mode-config#generate-environment-entries) 来起草 `autoMode.environment` 条目
* 在设置了 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 或 `DISABLE_GROWTHBOOK` 的情况下使用 [Remote Control](/docs/zh-CN/remote-control)。关于 `DISABLE_TELEMETRY` 和 `DO_NOT_TRACK`，请参阅 [Remote Control 要求](/docs/zh-CN/remote-control#requirements)
* 在 [Remote Control](/docs/zh-CN/remote-control#requirements) 不可用时[向本机以外的会话发送消息](/docs/zh-CN/cross-session-messaging#message-sessions-on-other-machines)。关闭获取后，本机上会话之间的消息传递仍可正常工作
* 运行 [`claude import` 或 `/import` 命令](/docs/zh-CN/cli-reference#cli-commands)
* 运行 [`/skill-doctor`](/docs/zh-CN/skills#find-unused-skills) 或在 `/plugin` 的 **Stats** 选项卡中打开其报告
* 将您的 claude.ai 账户中启用的 [skill](/docs/zh-CN/skills#where-synced-skills-load) 和[插件](/docs/zh-CN/plugins/loading#synced-plugins)同步到终端会话中
* 使用 [advisor 工具](/docs/zh-CN/advisor#requirements)
* 阅读或回复 [Artifact 上的评论](/docs/zh-CN/artifacts#collect-comments-on-an-artifact)
* 让 Claude 读取[其他组织的公开 Artifact](/docs/zh-CN/artifacts#read-an-artifact-shared-with-you)
* 让 Claude Code 针对 [MCP 协议修订版 2026-07-28](/docs/zh-CN/mcp#mcp-client-runtimes) 探测 claude.ai 连接器服务器或 stdio 服务器，除非您设置了 `MCP_PROTOCOL_NEGOTIATION=auto`
* 在安装了 Git Bash 的 Windows 上，为 claude.ai 和 Console 账户默认获得 [PowerShell 工具](/docs/zh-CN/tools-reference#powershell-tool)；除非您设置了 `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`，否则 Claude Code 会通过 Git Bash 执行 shell 命令。在未安装 Git Bash 的 Windows 上，该工具保持启用
* 获得 [Claude 起草的反馈](/docs/zh-CN/tools-reference#sendfeedback-tool-behavior)，该功能由 Claude Code 通过获取的标志启用
* 让 Claude [将大段粘贴内容视为粘贴而非键入的文本](/docs/zh-CN/terminal-config#how-claude-treats-pasted-text)；`[Pasted text #N]` 占位符背后的内容将以无标记形式传给 Claude
* 让 Claude Code [排除其输入 schema 会被 API 拒绝的 MCP 工具](/docs/zh-CN/mcp#tools-with-invalid-input-schemas)；它仍会发送该 schema，包含该 schema 的请求会失败，并返回[按位置指明该工具的 400 错误](/docs/zh-CN/errors#tool-input-schema-is-invalid)

<h3 id="first-session-after-an-install-or-upgrade">
  安装或升级后的首个会话
</h3>

在您安装 Claude Code 后，或升级到新增某项功能的版本后，首个会话中可能缺少[受标志控制的功能](#features-that-need-feature-flag-fetching)。该会话启动时的[权限模式](/docs/zh-CN/permission-modes#which-mode-a-session-starts-in)也可能与之后的会话不同。Claude Code 在该会话期间获取标志后，会将其保存在本机上，因此您在该机器上的下一个会话将具备该功能，并以通常的起始权限模式启动。

全新安装后，在非交互式会话（例如 `claude -p`、Agent SDK 或 VS Code 扩展）中，Claude Code 可能会在[选择起始权限模式](/docs/zh-CN/permission-modes#which-mode-a-session-starts-in)之前获取到标志，但并不总是会等待这些标志。

在以下这类设置中，首个会话之后的会话同样会在没有新获取标志的情况下启动：

* **每次运行都是全新环境**：如果每次运行都在 CI 容器中启动，或在任何没有先前会话所保存标志的其他环境中启动，那么每次运行都是首个会话
* **使用网关令牌且没有 API 密钥**：如果您使用 `ANTHROPIC_AUTH_TOKEN` 进行身份验证且没有 API 密钥，并且 `ANTHROPIC_BASE_URL` 指向 Anthropic 以外的主机（例如 [LLM 网关](/docs/zh-CN/llm-gateway)），Claude Code 将没有可用于获取标志的凭据

要选择这些设置中会话启动时使用的权限模式，请参阅[以不同的权限模式启动](/docs/zh-CN/permission-modes#start-in-a-different-mode)。

<h2 id="see-also">
  另请参阅
</h2>

* [设置](/docs/zh-CN/settings)：所有 `settings.json` 配置，包括 `env` 键
* [CLI 参考](/docs/zh-CN/cli-reference)：启动时标志
* [网络配置](/docs/zh-CN/network-config)：代理和 TLS 设置
* [监控](/docs/zh-CN/monitoring-usage)：OpenTelemetry 配置
