---
title: CLI 身份验证选项
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/authentication
description: 使用交互式登录、API 密钥、命名配置文件和 Workload Identity Federation 对 ant CLI 进行身份验证。
---

`ant` CLI 支持多种凭据来源。[快速入门](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/quickstart#authentication)介绍了单条命令的便捷路径（`ant auth login`）。本页完整介绍每一种选项。

## 交互式登录

`ant auth login` 让您无需创建或管理 API 密钥即可调用 API。它会针对 Claude Console 打开一个基于浏览器的 OAuth 流程，并将生成的凭据存储在 `$ANTHROPIC_CONFIG_DIR` 下（有关各操作系统的默认位置，请参阅[配置目录](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#configuration-directory)）。在远程主机上或任何没有本地浏览器的环境中，传入 `--no-browser` 以打印授权 URL，然后将返回的代码粘贴回终端。

```bash CLI
ant auth login

# 在没有浏览器的远程主机上：
ant auth login --no-browser

# 绑定到特定工作区并跳过浏览器选择器：
ant auth login --workspace-id wrkspc_01...

# 如果您通过 --profile 传入的命名配置文件不存在，
# 则会以该名称创建一个新的命名配置文件。
ant auth login --profile <profile-name>
```

在浏览器流程中，您先选择一个组织，然后选择一个 "workspace"（工作区），即[工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces)。颁发的令牌[作用域限定于该工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#api-keys-and-resource-scoping)，因此 CLI 只能看到属于该工作区的资源。传入 `--workspace-id` 可直接绑定并跳过选择器。若要在多个工作区中工作，请参阅[在工作区之间切换](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/authentication#switch-between-workspaces)。

交互式登录适用于在您自己的机器上进行本地开发和脚本编写。对于 CI、服务器和容器等非交互式工作负载，请改用 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)。

登录会将凭据写入 `credentials/<profile>.json`。某个配置文件的首次登录还会创建 `configs/<profile>.json` 并将其设为活动配置文件。要删除已存储的凭据，请运行 `ant auth logout`，或运行 `ant auth logout --all` 清除所有配置文件。

## 管理员访问

默认情况下，`ant auth login` 请求的是工作区作用域的令牌。要管理 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 页面中记录的资源，请在专用配置文件下请求 `org:admin` 作用域：

```bash CLI
ant auth login --profile admin --scope "org:admin"

# 打印用于 Authorization 标头的 bearer 令牌：
ant auth print-credentials --profile admin --access-token
```

`org:admin` 作用域仅授予具有 admin、owner 或 primary owner 角色的组织成员。颁发的令牌拥有组织范围的访问权限，配置文件上的任何工作区绑定都不会对其构成限制。请将管理员配置文件与您的日常配置文件分开，以确保常规命令永远不会以提升的权限运行。

## API 密钥

CLI 还会从 `ANTHROPIC_API_KEY` 环境变量读取您的 API 密钥。请从 [Claude Console](https://platform.claude.com/settings/keys) 获取密钥。

<Tabs>
  <Tab title="zsh">
    ```bash
    echo 'export ANTHROPIC_API_KEY=sk-ant-api03-...' >> ~/.zshrc
    source ~/.zshrc
    ```
  </Tab>

  <Tab title="bash">
    ```bash
    echo 'export ANTHROPIC_API_KEY=sk-ant-api03-...' >> ~/.bashrc
    source ~/.bashrc
    ```
  </Tab>

  <Tab title="Windows">
    ```powershell
    setx ANTHROPIC_API_KEY "sk-ant-api03-..."
    ```

    打开一个新终端以使更改生效。
  </Tab>
</Tabs>

要为单次调用覆盖密钥，请传入 `--api-key`。要指向不同的 API 主机，请设置 `ANTHROPIC_BASE_URL` 或传入 `--base-url`。

如果您使用的是作用域涵盖多个工作区的 API 密钥，例如[个人密钥或服务账户密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)，则必须[指定工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)来运行您的命令。为此，可以设置 `ANTHROPIC_WORKSPACE_ID` 环境变量（CLI 会自动读取），或使用 [`--workspace-id` 标志](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/using#global-flags)。该值必须是 `wrkspc_...` 形式的 ID；SDK 在 `ANTHROPIC_WORKSPACE_ID` 中为[联合令牌交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#environment-variables)所接受的字面值 `default` 在此处无效。

```bash CLI
ant messages create \
  --workspace-id wrkspc_01... \
  --model claude-opus-5 \
  --max-tokens 1024 \
  --message '{role: user, content: "Hello, Claude"}'
```

## 检查身份验证状态

`ant auth status` 会打印 CLI 所选择的凭据来源（API 密钥环境变量、OAuth 登录、联合身份或配置文件）、活动配置文件、活动令牌所绑定的工作区，以及配置目录路径。可用它来诊断某个工作负载为何选择了错误的凭据或工作区。

```bash CLI
ant auth status
```

```text
Active profile:  default
Config dir:      ~/.config/anthropic
Profile config:  ~/.config/anthropic/configs/default.json
Credentials:     ~/.config/anthropic/credentials/default.json

Credentials
  (active) * Profile (user_oauth) [via active_config]       sk-ant-oat01-EXA...
...

Workspace
  (active) * Workspace                                      wrkspc_01... (Engineering)
```

查看 `(active)` 行即可了解哪个凭据来源和工作区最终生效。该命令报告的是状态而非执行健康检查，因此不要针对其退出状态编写脚本。有关凭据来源的完整排序，请参阅[凭据优先级](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#credential-precedence)。

## 在工作区之间切换

交互式登录令牌绑定到单个工作区。要针对多个工作区使用 CLI，请在各自的命名配置文件下分别登录每个工作区，然后在它们之间切换：

```bash CLI
# 1. 创建配置文件（交互式；在浏览器中选择另一个工作区，
#    或传入 --workspace-id 以跳过选择器）：
# ant auth login --profile other-ws

# 2. 将其设为后续命令的默认配置文件：
ant profile activate other-ws

# 3. 或者仅为单条命令选择它，而不更改默认值：
ant --profile other-ws models list
ANTHROPIC_PROFILE=other-ws ant models list
```

运行 [`ant auth status`](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/authentication#check-authentication-status) 以确认哪个配置文件和工作区处于活动状态。

<Note>
  仅当未设置 API 密钥时才会查询配置文件。如果您的环境中存在 `ANTHROPIC_API_KEY`，它会覆盖所有配置文件，这些命令都将使用该密钥的工作区（或者，对于多工作区密钥，使用通过 `ANTHROPIC_WORKSPACE_ID` 或 `--workspace-id` 设置的工作区）。在切换配置文件之前请先取消设置该变量。
</Note>

## 管理配置文件

`ant profile` 子命令可直接检查和编辑配置文件状态：

```bash CLI
ant profile list
ant profile get --profile other-ws
ant profile set workspace_id wrkspc_01... --profile other-ws
```

`ant profile set` 可写入的键包括 `workspace_id`、`base_url`、`organization_id`、`scope`、`client_id` 和 `console_url`。设置 `workspace_id` 会在配置文件配置中记录目标工作区，但不会重新绑定已颁发的凭据；请在该配置文件下再次运行 `ant auth login`，为新工作区生成令牌。

有关配置文件的文件架构和 federation 块，请参阅[配置文件配置文件](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#profile-configuration-file)。有关 Workload Identity Federation，请参阅[身份验证概述](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)和 [WIF 参考](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference)。

## 后续步骤

<CardGroup cols={3}>
  <Card title="使用 CLI" icon="terminal" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/using">
    命令结构、输出格式、GJSON 转换和请求正文
  </Card>

  <Card title="CLI 脚本编写与自动化" icon="code" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/scripting">
    对 API 资源进行版本控制、脚本编写模式，以及在 Claude Code 中使用
  </Card>

  <Card title="Workload Identity Federation" icon="cloud" href="https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation">
    适用于 CI、服务器和容器的非交互式身份验证
  </Card>
</CardGroup>
