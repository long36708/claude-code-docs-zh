> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 在自托管环境中自定义会话

> 使用包装脚本在自托管环境会话中自定义每个会话的凭据、生命周期 hook 和按需运行程序生成。

<Note>
  自托管环境在 Team 和 Enterprise 计划中处于公开测试阶段；[Owner](/docs/zh-CN/cloud-environments#organization-shared-environments) 通过在 [**Cloud environments** 管理页面](https://claude.ai/admin-settings/cloud-environments) 上打开 **Allow self-hosted environments** 来启用它们。本页面假设您已有一个正常运行的运行程序；有关设置，请参阅[快速入门](/docs/zh-CN/self-hosted-environments-quickstart)，有关集群部署方案，请参阅[部署到生产环境](/docs/zh-CN/self-hosted-environments-deploy)。
</Note>

[自托管环境](/docs/zh-CN/self-hosted-environments)在您自己的基础设施上运行 Claude Code [云端会话](/docs/zh-CN/claude-code-on-the-web)，由您部署的运行程序进程执行。在没有配置的情况下，该运行程序克隆会话的仓库，生成 Claude Code，然后进行清理。本页面适用于操作运行程序的平台工程师：它涵盖了当这些默认值不适用时的扩展点，从每个会话的凭据配置到完全替换检出。包装脚本和 hook 作为运行程序主机上的可执行文件运行，该主机是 Linux 或 macOS，本页面上的示例假设使用 POSIX shell。

本页面上的一些 hook 环境变量仍然使用 `pool`，例如 `CLAUDE_RUNNER_POOL_ID`；CLI 标志和环境变量名称使用 `environment`，例如 `--environment-secret-file`。

<h2 id="wrapper-scripts">
  包装脚本
</h2>

当每个会话需要运行器无法自行完成的设置时，使用包装脚本：为会话创建者配置作用域的短期凭据、导出特定于环境的密钥、准备语言工具链或围绕子进程应用资源限制。运行器每个会话启动一次您的包装脚本，而不是 Claude Code 二进制文件。通过 `exec` 进入 `$CLAUDE_RUNNER_CLAUDE_BIN`（运行器自己的二进制文件）来结束包装脚本，以便信号和退出码正确传播。

启动运行器时，使用 `--exec-path` 或 `SELF_HOSTED_RUNNER_EXEC_PATH` 指向包装脚本：

```bash theme={null}
claude self-hosted-runner --environment-secret-file /etc/claude/environment-secret --exec-path /etc/claude/session-wrapper.sh
```

运行器在包装脚本的环境中设置以下内容：

| 变量 | 描述 |
| :- | :- |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | 会话 JWT，前缀为 `sk-ant-cc-`。其 `act` 声明标识会话创建者，并在创建会话的使用入口记录了创建者电子邮件时包含该电子邮件。该值是生成时的令牌；刷新通过子进程的 stdin 到达，因此包装脚本只看到初始值。请参阅 [Verify session identity](/docs/zh-CN/self-hosted-environments-identity)。 |
| `CCR_SESSION_ACCOUNT_EMAIL` | 会话创建者的电子邮件，由运行器从令牌的 `act.email` 声明中预先提取，无需签名验证。适合用于标记，例如提交 trailer。当电子邮件控制凭据发放时，验证令牌并从中读取声明；请参阅 [Provision credentials scoped to the session creator](#provision-credentials-scoped-to-the-session-creator)。当令牌不包含创建者电子邮件时未设置。视为个人可识别信息。 |
| `CLAUDE_RUNNER_CLIENT_PLATFORM` | 创建会话的客户端使用入口，例如 `web_claude_ai`、`desktop_app`、`ios`、`claude_code_cli` 或 `scheduled_trigger`。Anthropic 在会话创建时记录该值一次，因此包装脚本和每个生命周期 hook 都看到相同的值。仅将其用于采用分析和标记，不用作授权信号。当会话没有记录或识别的使用入口时未设置，因此在 `set -u` 下将其引用为 `${CLAUDE_RUNNER_CLIENT_PLATFORM:-}`。需要 Claude Code v2.1.229 或更高版本。 |
| `CLAUDE_RUNNER_CLAUDE_BIN` | 运行器自己的 Claude Code 二进制文件的绝对路径。使用 `exec "$CLAUDE_RUNNER_CLAUDE_BIN" "$@"` 结束您的包装脚本，以移交到固定的二进制文件，而无需硬编码安装路径。 |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | 会话 ID，采用标记的 `cse_...` 形式。这与[生命周期 hook](#lifecycle-hooks)以 `session_...` 形式在 `CLAUDE_RUNNER_SESSION_ID` 中看到的是同一个会话；UUID 变量在两者之间匹配，将 `cse_` 前缀替换为 `session_` 会产生会话 URL 中显示的 ID。 |
| `CLAUDE_CODE_REMOTE_SESSION_UUID` | 相同的会话 ID，采用规范 UUID 形式，供以 UUID 作为键的系统使用。 |
| `CLAUDE_SESSION_INGRESS_TOKEN_FILE` | 绝对路径，指向保存当前会话 JWT 的按会话文件，在令牌刷新时保持最新。Shell 子进程在下载用户添加到会话的附件时从中读取其 `Authorization` 标头。`exec` 自动保留该变量；重建子进程环境的包装脚本必须携带该变量，否则附件下载会无声地停止工作。 |
| `CLAUDE_CONFIG_DIR` | 按会话 Claude 配置目录，在会话启动时从运行器在启动时捕获的运行器主机配置快照中写入；请参阅 [Permissions and tool approval](#permissions-and-tool-approval)。此处的写入仅限于此会话。除非您使用 [`--remove-session-state`](/docs/zh-CN/self-hosted-environments-reference#runner-cli-flags) 启动运行器，否则会话结束后该目录仍会保留在 `<base-dir>/_sessions/` 下；请参阅 [Reuse a pre-warmed checkout](/docs/zh-CN/self-hosted-environments-deploy#reuse-a-pre-warmed-checkout)。 |
| `ANTHROPIC_BASE_URL` | 子进程将使用的 API 基础 URL，由控制平面按会话交付，通常为 `https://api.anthropic.com`。不要覆盖它：会话的推理凭据是 Anthropic 颁发的 OAuth 令牌，其他提供者不接受，因此自托管环境中的推理无法路由到其他地方。 |
| `CLAUDE_CODE_OAUTH_TOKEN` | 子进程用于模型推理的短期 OAuth 访问令牌，作用域仅限于模型推理和文件上传，生命周期约为 30 分钟。运行器在过期前重新生成它，并通过子进程的 stdin 交付轮换，因此不 [keep stdin attached](#keep-stdin-and-file-descriptor-3-attached) 的包装脚本只看到初始值。不要依赖您的组织 IP 允许列表来限制此令牌的使用：将其视为持有者凭据，如果泄露，大约 30 分钟内仍可使用，不要记录它、写入磁盘或在会话容器外转发它。 |

包装脚本还继承子进程的其余托管环境，包括任何服务器提供的环境变量。`exec` 自动传播所有内容；如果您的包装脚本以其他方式生成子进程，请转发完整环境。

<h3 id="keep-stdin-and-file-descriptor-3-attached">
  保持 stdin 和文件描述符 3 的连接
</h3>

子进程的 stdin 是运行器的控制通道。令牌轮换和会话结束信号在其上到达。运行器还在文件描述符 3 上打开一个管道，并从中读取子进程的活动信号以驱动空闲和启动超时。普通的 `exec "$CLAUDE_RUNNER_CLAUDE_BIN" "$@"` 自动保留两者。

如果您的包装脚本使用裸 `&` 在后台运行子进程，它会切断子进程的 stdin：会话看起来健康，直到初始 OAuth 令牌的大约 30 分钟生命周期过期，然后每个 API 调用都失败，出现 `401 authentication_error`。如果您的包装脚本必须在后台运行子进程，例如保持拆卸陷阱活跃，请在文件描述符 4 或更高编号上保存 stdin 并显式重新连接它：

```bash theme={null}
exec 4<&0
"$CLAUDE_RUNNER_CLAUDE_BIN" "$@" <&4 4<&- &
CHILD=$!
trap 'teardown' EXIT
wait "$CHILD"
```

不要在包装脚本中关闭或重用文件描述符 3。重定向子进程的 stdout 和 stderr 是可以的。

<h3 id="pass-the-system-prompt-flags-through">
  透传系统提示词标志
</h3>

Anthropic 控制平面为会话发送的系统提示词和追加系统提示词以文件路径的形式（而不是内联文本）到达您的包装脚本。运行器将每个提示词写入会话配置目录 `CLAUDE_CONFIG_DIR` 中的文件，并在您的包装脚本接收的参数中传递其路径，形式为 [`--system-prompt-file <path>` 或 `--append-system-prompt-file <path>`](/docs/zh-CN/cli-reference#system-prompt-flags)。

Claude Code v2.1.281 或更高版本上的运行器以文件形式传递提示词。在 v2.1.281 之前，运行器以 `--system-prompt <text>` 和 `--append-system-prompt <text>` 的形式传递它们。

在您的包装脚本或 [`command` hook](#command) 中，按如下方式处理这些标志：

* **透传它们**：使用 `exec "$CLAUDE_RUNNER_CLAUDE_BIN" "$@"` 结束包装脚本，这会将文件标志与其他所有参数一起转发。不要丢弃或改写它们。如果会话丢失了某个提示词文件标志，它将在没有控制平面为其发送的指令的情况下运行。
* **在 v2.1.281 或更高版本的运行器上，您追加的文件标志会替换服务器的标志，而不会叠加**：每个提示词文件标志只接受单个值，Claude Code 保留最后一次出现的值，因此如果您在 `"$@"` 之后追加 `--append-system-prompt-file <path>`，您文件的内容将替换服务器追加的指令。要在服务器指令之上添加指令，请将其放入运行器镜像的 `CLAUDE.md` 中，运行器会将其[植入每个会话的用户级配置](#how-each-session’s-config-is-assembled)。

<h3 id="provision-credentials-scoped-to-the-session-creator">
  配置作用域限定为会话创建者的凭据
</h3>

使用 `decode-token` 子命令从会话 JWT 读取声明。它从参数、`CLAUDE_CODE_SESSION_ACCESS_TOKEN` 或 stdin 读取令牌，按该顺序；请参阅 [Verify the token inside the session](/docs/zh-CN/self-hosted-environments-identity#verify-the-token-inside-the-session) 了解它检查的内容。下面的示例解码创建者身份，将其交换为短期 AWS 凭据，并 exec 进入 Claude Code：

```bash theme={null}
#!/bin/bash
# Key on the stable Anthropic user ID and require a human creator.
CREATOR_SUB=$("$CLAUDE_RUNNER_CLAUDE_BIN" self-hosted-runner decode-token \
  | jq -re '.act.sub // "" | select(startswith("user:"))') \
  || { echo "decode-token: verification failed or no human creator" >&2; exit 1; }

creds=$(your-sts-helper assume-role --subject "$CREATOR_SUB") \
  || { echo "credential exchange failed" >&2; exit 1; }
eval "$creds"

exec "$CLAUDE_RUNNER_CLAUDE_BIN" "$@"
```

在提取的声明控制身份验证决策时，使用 `jq -re` 而不是 `jq -r`，以便缺失的声明以非零状态退出，而不是将字面字符串 `null` 传递给下游。由组织服务身份（例如机器人和 Agent 会话）创建的会话携带 `agent:` 主题而不是 `user:`，因此此示例拒绝它们；如果您的环境为这些会话提供服务，请明确决定包装脚本是否为它们回退到默认凭据，而不是退出。当您的凭据交换改为需要电子邮件时，读取 `.act.email` 并处理其缺失：令牌仅在创建会话的使用入口记录了电子邮件时才携带它，[CLI 分派的会话](/docs/zh-CN/self-hosted-environments-testing#run-the-test-loop) 可能缺少它。有关完整的声明参考和来自运行器外部服务的验证，请参阅 [Verify session identity](/docs/zh-CN/self-hosted-environments-identity)。

<h2 id="lifecycle-hooks">
  生命周期钩子
</h2>

生命周期钩子用您自己的脚本替换运行器按会话管道的阶段。使用 `--hooks-dir <path>` 或 `SELF_HOSTED_RUNNER_HOOKS_DIR` 将运行器指向钩子目录。运行器查找具有众所周知名称的可执行文件；任何不存在的钩子都会回退到内置行为，因此您只需编写需要的钩子。钩子以运行器自己的权限运行，会话子进程共享该 UID，因此请以只读方式挂载钩子目录，或将其烘焙到镜像中，以便会话代码无法修改它；请参阅 [hardening section](/docs/zh-CN/self-hosted-environments-deploy#harden-your-deployment)。

这些钩子不同于 [Claude Code hooks](/docs/zh-CN/hooks)，后者在会话内运行；生命周期钩子在运行器上运行，围绕会话。

<h3 id="checkout">
  checkout
</h3>

每个仓库运行一次，代替运行器的内置克隆和获取。使用此 hook 从读通镜像克隆、从存档为工作树设置种子或应用按会话的 git 身份验证。运行器设置以下变量，并且可能设置表中未列出的其他 `CLAUDE_RUNNER_` 变量：

| 变量 | 描述 |
| :- | :- |
| `CLAUDE_RUNNER_REPO_URL` | 要克隆的存储库 URL，在应用任何 `--git-host-rewrite` 和 `--git-ssh-rewrite` 之后 |
| `CLAUDE_RUNNER_REPO_REF` | 要检出的修订版本：分支、标签或提交 SHA，如会话请求的那样。空表示存储库的默认分支。 |
| `CLAUDE_RUNNER_CHECKOUT_PATH` | 必须留下工作树的绝对路径 |
| `CLAUDE_RUNNER_SESSION_ID` | 会话 ID，采用标记的 `session_...` 形式，用于日志记录和关联 |
| `CLAUDE_RUNNER_SESSION_UUID` | 相同的会话 ID，采用规范 UUID 形式 |
| `CLAUDE_RUNNER_API_BASE_URL` | Anthropic API 基础 URL，用于会话范围的调用 |
| `CLAUDE_RUNNER_CLIENT_PLATFORM` | 创建会话的客户端表面，例如 `web_claude_ai`、`desktop_app` 或 `ios`。当会话没有记录或识别的表面时未设置。 |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | 会话访问令牌，用于会话范围的 API 调用 |
| `GIT_CONFIG_COUNT`、`GIT_CONFIG_KEY_n`、`GIT_CONFIG_VALUE_n` | 运行器为您的 hook 所运行的 git 固定的 Git 设置。[生命周期 hook 中的 Git 配置](#git-configuration-inside-lifecycle-hooks)对其进行了说明。需要 Claude Code v2.1.280 或更高版本。 |

脚本必须在 `CLAUDE_RUNNER_CHECKOUT_PATH` 处留下一个工作树，检出到请求的修订版本。分离的 HEAD 是可以的；运行器在其上创建会话的工作分支。运行器之后验证路径包含 `.git`；如果您的钩子具体化非 git 源（例如 Perforce 或解包的 tarball），请在运行器的环境中设置 `CLAUDE_RUNNER_SKIP_GIT_VERIFY=1` 以跳过该检查。基于 Git 的流程（例如工作分支创建和推送结果）需要 git 检出，因此使用 [`post-session` 钩子](#post-session) 从非 git 树导出结果。

运行器不会将 git 凭证传递给钩子。相反，从会话的身份生成按会话克隆凭证：使用标准 JWT 库针对 `CLAUDE_RUNNER_API_BASE_URL` 下的 JWKS 端点验证 `CLAUDE_CODE_SESSION_ACCESS_TOKEN`，如 [Verify the token from your service](/docs/zh-CN/self-hosted-environments-identity#verify-the-token-from-your-service) 中所述，然后让您的凭证服务为令牌的 `act` 声明中的身份发放短期克隆凭证。`CLAUDE_RUNNER_CLAUDE_BIN` 未在 checkout-hook 环境中设置，因此 `decode-token` 子命令在此处不可用。回退到主机已有的任何 git 身份验证（例如 SSH 代理、凭证助手或 `.netrc`）也是一个选项。

当钩子以非零状态退出，或以 0 退出但没有留下可用的检出时，运行器的行为取决于存储库：

* **会话推送结果的存储库**：运行器失败会话，在非零退出时将脚本的 stderr 尾部呈现给用户。
* **会话仅从中读取的存储库**，例如添加到运行会话的存储库：运行器记录带有失败详情的 `[runner:warn]` 行，向会话发布 `Skipped` 步骤，删除钩子在检出路径处留下的任何内容，并继续处理其余存储库。当运行器无法立即删除路径时，它会在会话结束时重试删除。如果跳过使会话完全没有存储库，运行器仍然会失败会话。

在 v2.1.228 之前，运行器对任何存储库的钩子失败都会失败会话，因此钩子无法提供的只读存储库在会话恢复到的每个新运行器上再次失败会话。

运行器在会话结束后删除检出路径。

<h3 id="post-session">
  post-session
</h3>

每个会话运行一次，在 Claude Code 子进程退出后和运行器拆卸工作区之前。此钩子是保存未提交工作的唯一机会：在 `--capacity` 高于 1 时，运行器在钩子返回后立即删除按会话工作树，在 `--capacity 1` 时重用的 [canonical clone](/docs/zh-CN/self-hosted-environments-deploy#reuse-a-pre-warmed-checkout) 在下一个会话启动时硬重置，因此未提交的跟踪更改在两条路径上都不会存活。典型用途是推送未提交更改的快照分支、存档日志或向您自己的系统发出会话结束事件。

钩子在每个会话结束时触发，其中生成了子进程，无论原因如何；下面的 `CLAUDE_RUNNER_EXIT_REASON` 值枚举了这些情况。当运行器突然终止时（例如 VM 抢占或断电）它无法触发；如果您需要针对突然终止的保证，请改为使用 Claude Code `PostToolUse` 钩子从会话内定期快照。运行器设置：

| 变量 | 描述 |
| :- | :- |
| `CLAUDE_RUNNER_SESSION_ID` | 会话 ID，采用标记的 `session_...` 形式 |
| `CLAUDE_RUNNER_SESSION_UUID` | 相同的会话 ID，采用规范 UUID 形式 |
| `CLAUDE_RUNNER_EXIT_REASON` | 会话如何结束；请参阅表下方的值 |
| `CLAUDE_RUNNER_WORKSPACE_PATHS` | 会话工作树的冒号分隔绝对路径。对于零存储库会话为空。 |
| `CLAUDE_RUNNER_DEBUG_LOG_PATH` | 会话的调试日志的路径，在钩子运行时仍在磁盘上 |
| `CLAUDE_RUNNER_API_BASE_URL` | Anthropic API 基础 URL，用于会话范围的调用 |
| `CLAUDE_RUNNER_CLIENT_PLATFORM` | 创建会话的客户端表面，例如 `web_claude_ai`、`desktop_app` 或 `ios`。当会话没有记录或识别的表面时未设置。需要 Claude Code v2.1.229 或更高版本。 |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | 会话访问令牌，用于会话范围的 API 调用 |
| `GIT_CONFIG_COUNT`、`GIT_CONFIG_KEY_n`、`GIT_CONFIG_VALUE_n` | 运行器为您的 hook 所运行的 git 固定的 Git 设置。[生命周期 hook 中的 Git 配置](#git-configuration-inside-lifecycle-hooks)对其进行了说明。需要 Claude Code v2.1.280 或更高版本。 |

`CLAUDE_RUNNER_EXIT_REASON` 采用四个值之一：

* `completed`：会话干净地结束。Claude Code 进程正常退出，或会话在仍在运行时被存档或删除。
* `failed`：Claude Code 进程崩溃，或在启动后设置失败。
* `interrupted`：运行器停止了会话。它释放了会话以释放插槽、会话在启动时超时、服务器将会话移出此运行器、运行器正在排空，或会话超过了其 [`--kill-session-after-min`](/docs/zh-CN/self-hosted-environments-reference#runner-cli-flags) 限制。
* `abandoned`：为另一个运行器声称的会话保留。钩子目前在这种情况下不触发。

[session lifecycle counters](/docs/zh-CN/self-hosted-environments-reference#session-lifecycle-counter-semantics) 将释放、启动超时和服务器移动计为 `completed` 而不是 `interrupted`，因为运行器干净地交还了插槽。如果您将钩子收据与计数器进行比较，请预期这种差异。

钩子的退出状态永远不会影响会话结果；失败被记录并忽略。运行器在每个会话结束（包括运行器关闭）时等待最多 `--post-session-hook-timeout-sec`（默认 60 秒）。此示例将未提交的工作保存到救援分支：

```bash theme={null}
#!/usr/bin/env bash
set -u
IFS=':'
# -c overrides beat repo-local settings, blocking session-written fsmonitor,
# hook-path, and gpg-program config from executing code with the hook's
# privileges. -c commit.gpgsign=false also leaves these rescue commits
# unsigned under --configure-git.
# Repo-local credential.helper and pushurl still apply, and on a runner
# before v2.1.280 so does core.sshCommand; if the hook holds credentials
# the session didn't, see the note below the script.
g() { git -c core.fsmonitor=false -c core.hooksPath=/dev/null \
        -c commit.gpgsign=false "$@"; }
for ws in $CLAUDE_RUNNER_WORKSPACE_PATHS; do
  cd "$ws" 2>/dev/null || continue
  [ -z "$(g status --porcelain 2>/dev/null)" ] && continue
  g add -A
  g commit -q -m "runner snapshot: $CLAUDE_RUNNER_SESSION_ID ($CLAUDE_RUNNER_EXIT_REASON)" || continue
  g push -q origin "HEAD:refs/heads/rescue/$CLAUDE_RUNNER_SESSION_ID" || true
done
```

hook 使用运行器主机上其自身环境中可用的任何 git 凭据进行推送。在[镜像中不含凭据的部署方式](/docs/zh-CN/self-hosted-environments-deploy#configure-git)下，包括内置克隆通过 Anthropic git 代理进行时，都没有可用凭据，因此请在推送前于 hook 内生成短期推送凭据：将 hook 在 `CLAUDE_CODE_SESSION_ACCESS_TOKEN` 中收到的会话令牌与您自己的令牌服务进行交换，并按照[验证会话身份](/docs/zh-CN/self-hosted-environments-identity)中的说明对其进行验证。当 hook 持有会话没有的凭据时，请将 `origin` 替换为操作员提供的 URL，并传递 `-c credential.helper=` 加上您自己的助手。[生命周期 hook 中的 Git 配置](#git-configuration-inside-lifecycle-hooks)说明了会话写入的配置仍可能影响哪些内容。

<h4 id="hook-timing-when-the-runner-releases-a-session">
  运行器释放会话时的钩子时序
</h4>

已释放的会话可以在另一个运行器上恢复。在 v2.1.236 或更高版本的运行器上，会话在释放时所做的事情决定了它是否可以在此钩子完成前在另一个运行器上恢复：

* **在轮次后空闲，或在启动时超时**：运行器停止子进程并运行此钩子至完成。只有这样它才会释放会话。在钩子运行时发送的用户消息无法在钩子完成前在另一个运行器上恢复会话。
* **等待用户回答提示，例如权限提示**：运行器首先释放会话，然后运行此钩子。在钩子运行时发送的用户消息可以在钩子完成前在另一个运行器上恢复会话。

这适用于运行器释放会话的任何时候：在空闲超时、在 [`--retire-at`](/docs/zh-CN/self-hosted-environments-reference#runner-cli-flags) 时间，以及 在 v2.1.260 或更高版本的运行器上，在会话的 [`--kill-session-after-min`](/docs/zh-CN/self-hosted-environments-reference#runner-cli-flags) 限制。其轮次已结束且仅持有后台任务的会话在此处计为空闲。在 v2.1.236 之前，运行器在两种情况下都首先释放会话，然后运行此钩子。

在 `SIGTERM` 排空期间，运行器持有会话租约直到钩子完成；请参阅 [Shutdown timing](/docs/zh-CN/self-hosted-environments-deploy#shutdown-timing)。

<h3 id="git-configuration-inside-lifecycle-hooks">
  生命周期 hook 中的 Git 配置
</h3>

`checkout` 和 `post-session` hook 运行时，其环境中包含会话的访问令牌，而它们运行的 git 会读取会话可以写入的配置文件，例如 `~/.gitconfig` 和检出目录中的 `.git/config`。在任一 hook 运行之前，运行器会在 hook 的环境中以 `GIT_CONFIG_COUNT`/`GIT_CONFIG_KEY_n`/`GIT_CONFIG_VALUE_n` 对和 git 环境变量的形式设置 git 设置，包括以下各项。Git 将这些设置的优先级排在所有配置文件之上，并且它们仅适用于您的 hook 运行的 git，而不适用于会话自己的 git。启动时，运行器会打印一行 `[runner:git] lifecycle hooks:`，显示当前生效的钩子路径、允许的协议、gpg 程序和签名模式。需要 Claude Code v2.1.280 或更高版本。

* **Git 钩子**：除非您提供值，否则 `core.hooksPath` 为 `/dev/null`，因此 git 会跳过仓库 `.git/hooks` 中的钩子以及 `~/.gitconfig` 指定的任何钩子目录。要提供一个值，请在运行器的环境中将 `core.hooksPath` 导出为 `GIT_CONFIG_KEY_n`/`GIT_CONFIG_VALUE_n` 对。运行器还会从系统 git 配置中读取 `core.hooksPath`，并且仅当运行器的用户无法写入该文件、其指定的目录或其中的钩子文件时才使用它。当运行器忽略某个值时，启动时会有一行 `[runner:warn]` 指明该值及原因。
* **文件系统监视器**：`core.fsmonitor` 为空，因此 hook 中的 git 不会运行配置文件中指定的监视程序。
* **远程协议**：`GIT_ALLOW_PROTOCOL` 为 `https:http:ssh`。使用本地路径、`file://` URL 或 `git://` URL 的克隆、获取或推送会失败，并报错 `fatal: transport 'file' not allowed` 或 `fatal: transport 'git' not allowed`。
* **SSH 命令和凭据提示**：hook 中的 git 会忽略配置文件中的 `core.sshCommand` 和 `core.askPass`。要使用您自己的 SSH 命令，请在运行器的环境中设置 `GIT_SSH_COMMAND`。要使用凭据提示程序，请在其中设置 `GIT_ASKPASS`。会话会继承运行器的环境，因此这两个变量也会影响会话自己的 git。请勿在其中任何一个中放入凭据。
* **gpg 程序**：`gpg.program`、`gpg.openpgp.program`、`gpg.x509.program` 和 `gpg.ssh.program` 是运行器设置的路径，绝不会取自配置文件中的值。
* **提交签名**：使用 [`--configure-git`](/docs/zh-CN/self-hosted-environments-deploy#let-the-runner-configure-git) 时，您从 hook 中进行的提交会以会话身份签名。不使用该标志时，`commit.gpgsign` 和 `tag.gpgsign` 为 `false`。

要更改其中某项设置，请使用运行器的环境或在 hook 内使用 `git -c` 选项：

* **配置对**：您在运行器环境中导出的 `GIT_CONFIG_KEY_n`/`GIT_CONFIG_VALUE_n` 对会替换运行器为同一键设置的值。请从 `0` 开始为您的配置对编号，并将 `GIT_CONFIG_COUNT` 设置为配置对的数量。当计数所声明的最后一个配置对缺失时，运行器会忽略您的所有配置对，并在启动时记录一行 `[runner:warn]`。
* **Git 环境变量**：运行器会保留您在其环境中设置的 `GIT_ALLOW_PROTOCOL`、`GIT_SSH_COMMAND` 和 `GIT_ASKPASS`。
* **`git -c` 选项**：hook 内的 `git -c` 选项会覆盖 `GIT_CONFIG_KEY_n` 对，无论是运行器的还是您的。它不会更改 `GIT_ALLOW_PROTOCOL`、`GIT_SSH_COMMAND` 或 `GIT_ASKPASS`，git 会先于任何配置读取这些变量。

hook 中的 git 仍会从每个配置文件（包括会话可以写入的配置文件）中读取运行器未设置的所有设置，例如凭据助手、`url.*.insteadOf` 重写和过滤器驱动程序。这些文件之一中指定的凭据助手或过滤器驱动程序会以您的 hook 的权限作为程序运行，并且这些文件中的配置仍可能改变您的 hook 推送的目标位置，包括推送到您在命令行上传递的 URL。

在 v2.1.280 之前，运行器不设置这些设置中的任何一项，并且在 `--configure-git` 下，从 hook 中进行的提交会失败，除非 hook 传递了 `-c commit.gpgsign=false`。

<h3 id="command">
  command
</h3>

每个会话在检出后运行一次，代替内置子进程生成。钩子接收与 [wrapper script](#wrapper-scripts) 相同的环境，应该以相同的方式 `exec` 进入 `"$CLAUDE_RUNNER_CLAUDE_BIN"`。使用 `command` 钩子将所有自定义保留在一个钩子目录中；当包装脚本在其他地方时使用 `--exec-path`。如果也设置了 `--exec-path`，标志优先，`command` 钩子被忽略。

始终 `exec` 运行器自己的二进制文件，而不是 PATH 解析的 `claude`；否则您会破坏 [version pinning](/docs/zh-CN/self-hosted-environments-deploy#pin-the-version)。

<h2 id="on-demand-runners">
  按需运行器
</h2>

您可以为每个会话启动一个运行器，而不是运行固定的队列。编排器是一个单独的、无状态的子命令，它轮询 Anthropic 以获取生成请求（每个没有可用运行器的排队会话一个），并为每个请求运行您的 `spawn-runner` hook。您的 hook 向您的平台提交工作负载：Kubernetes Job、EC2 实例、Nomad dispatch。

按需运行器改进了凭据卫生。在固定队列上，环境密钥存在于每个运行器主机上，这是运行用户会话的同一主机。使用编排器，环境密钥仅保留在编排器主机上，该主机从不运行用户代码；每个生成的运行器接收一个单次使用的工作单，恰好注册一个运行器，然后过期。

要启动编排器，请传递环境密钥和包含可执行 `spawn-runner` 脚本的 hook 目录：

```bash theme={null}
claude self-hosted-runner orchestrator \
  --environment-secret-file /etc/claude/environment-secret \
  --hooks-dir /etc/claude/hooks
```

编排器在轮询之间保持无状态，因此您可以针对同一环境运行两个或多个副本以实现可用性。每个生成请求由服务器端的恰好一个副本声称。所有副本必须使用相同的 `--expected-spawn-seconds` 值；请参阅 [hook 约定](#the-spawn-runner-hook)。

<h3 id="the-spawn-runner-hook">
  spawn-runner hook
</h3>

编排器为每个生成请求运行一次 `${hooks-dir}/spawn-runner`。hook 必须异步提交工作，不等待运行器启动，并在 `--hook-timeout`（默认 60 秒）内返回。hook 接收：

| 变量 | 描述 |
| :- | :- |
| `CLAUDE_RUNNER_WORK_ORDER_FILE` | 包含新运行器注册所用的已签名工作单 JWT 的临时文件的路径。hook 退出后删除。不要记录文件的内容。 |
| `CLAUDE_RUNNER_ORDER_ID` | 不透明的幂等性密钥，每个生成请求唯一，对 Kubernetes 资源名称安全。仅将订单 ID 用作您的配置器的去重密钥。 |
| `CLAUDE_RUNNER_SESSION_ID` | 此请求所针对的会话。该会话的每次重新请求都会重复此值，因此请将其用于日志记录和路由，而不要用作去重密钥。对于预热请求为空，预热请求在设置 [`--min-idle`](/docs/zh-CN/self-hosted-environments-reference#orchestrator-cli-flags) 时于任何特定会话之前启动待命运行器，因此不要假设变量已设置。 |
| `CLAUDE_RUNNER_SESSION_UUID` | 相同的会话 ID，采用规范 UUID 形式。对于预热请求为空。 |
| `CLAUDE_RUNNER_ATTEMPT` | 此会话已有多少个生成请求。对于预热请求为 `0`。 |
| `CLAUDE_RUNNER_ORDER_SERVER_TIME` | 来自轮询响应的 HTTP `Date` 标头的服务器时间。当 hook 验证工作单 JWT 的 `exp` 时，与此值进行比较而不是本地时钟，以容忍时钟偏差。当网关省略标头时为空。 |
| `CLAUDE_RUNNER_POOL_ID` | 新运行器应加入的环境的 ID，采用 `ccpool_...` 形式 |
| `CLAUDE_RUNNER_ACCOUNT_ID` | 排队会话的帐户的标记 ID，用于按帐户路由、配额或退款。当不可用时为空，对于 Claude Tag 频道会话始终为空，这些会话没有帐户排队。 |
| `CLAUDE_RUNNER_ACCOUNT_EMAIL` | 排队会话的帐户的电子邮件。当不可用时为空。将电子邮件视为个人可识别信息，不要记录它。 |
| `CLAUDE_RUNNER_PRIMARY_REPO_URL` | 会话的第一个 git 源的 URL，用于路由到已预热该仓库的运行器。当会话没有 git 源时为空。 |
| `CLAUDE_RUNNER_PRIMARY_REPO_REVISION` | 会话的第一个 git 源的修订版本：分支、SHA 或标签。当未指定时为空。 |
| `CLAUDE_RUNNER_REPO_SOURCES` | 所有会话的 git 源的 `{url, revision}` 的 JSON 数组，用于根据辅助仓库进行路由的 hook。当没有源时为空。 |
| `CLAUDE_RUNNER_CORRELATION_ID` | 在会话创建时提供的关联 ID，回显以便 hook 可以将此工作单映射到创建会话的请求。当会话没有时为空。 |
| `CLAUDE_RUNNER_CLIENT_PLATFORM` | 创建会话的客户端使用入口，例如 `web_claude_ai`、`desktop_app`、`ios` 或 `scheduled_trigger`，用于采用分析。当会话没有记录或识别的使用入口时未设置，对于预热请求也未设置；使用 `[ -n "${CLAUDE_RUNNER_CLIENT_PLATFORM:-}" ]` 检查它，这在 `set -u` 下保持安全。 |

生成的运行器使用工作单代替环境密钥进行注册：

* **使用工作单启动它**：将 [`--environment-secret-file`](/docs/zh-CN/self-hosted-environments-reference#runner-cli-flags) 指向包含工作单 JWT 的文件，或将 `SELF_HOSTED_RUNNER_ENVIRONMENT_SECRET` 设置为 JWT 值。
* **在 hook 退出前复制 JWT**：编排器在 hook 退出后删除工作单文件，因此将 JWT 复制到您提交的工作负载中，例如生成的 Job 上的 Kubernetes Secret，而不是传递文件路径。
* **在生成的运行器上使用 `--capacity 1`**：会话绑定的工作单恰好注册一个绑定到该会话的运行器，因此更高的容量添加永远不会接收工作的插槽，运行器在启动时记录警告。
* **预热工作单注册未绑定**：待命运行器未绑定到会话，并像固定队列运行器一样声称排队的工作。

约定有四个与配置器无关的规则：

1. **在 `CLAUDE_RUNNER_ORDER_ID` 上保持幂等。** 相同请求的重新交付必须最多生成一个运行器。从订单 ID 派生确定性资源名称，让您的平台拒绝重复。不要改为以 `CLAUDE_RUNNER_SESSION_ID` 作为键。会话的每次重新请求都携带相同的会话 ID 和新的订单 ID，因此按会话 ID 命名或去重的工作负载只会创建一次，之后该会话再也不会创建。
2. **不要重试工作负载。** 一个订单 ID 意味着最多创建一个工作负载。如果运行器从不注册，Anthropic 在 `--expected-spawn-seconds` 后使用新订单 ID 重新请求。
3. **使用退出码约定。** 退出 0 表示已提交。退出 1 表示可重试失败；会话退避并被重新提供。退出 2 或更高表示不可重试；会话被阻止再次生成，直到 [Owner](/docs/zh-CN/cloud-environments#organization-shared-environments) 在环境的 **Activity** 标签中选择 **Retry**。在非零退出时，hook 的 stderr 尾部出现在那里作为失败原因，因此将可操作的错误写入 stderr，永远不要写密钥。对于预热请求，没有会话失败：编排器仅在本地记录非零退出，服务器在租约后重新请求生成。
4. **将 `--expected-spawn-seconds` 设置为至少您的 p99 启动时间。** 这是服务器端租约。所有编排器副本必须使用相同的值。

hook 写入 stdout 或 stderr 的所有内容都出现在编排器的日志中，凭据会自动脱敏。如果会话保持排队，检查编排器的 `/healthz` 正文以获取队列计数，然后在 [**Cloud environments** 管理页面](https://claude.ai/admin-settings/cloud-environments) 上打开您的环境的 **Activity** 标签：在那里展开失败的会话以获取其生成错误，并选择 **Retry** 以重新请求它。

如果会话保持排队，且 **Activity** 标签中没有生成错误，可能意味着 hook 以会话 ID 作为键。要确认这一点，请检查您的平台是否存在该会话第一次生成请求对应的工作负载，而重新请求却没有对应的工作负载。如果是这样，请改为以 `CLAUDE_RUNNER_ORDER_ID` 作为工作负载的键。

<h2 id="mcp-servers">
  MCP 服务器
</h2>

要在每个会话中提供 [MCP servers](/docs/zh-CN/mcp)，请在镜像构建时使用与桌面安装上使用的相同 `claude mcp add` 命令添加它们。如果您的运行器是裸进程而不是容器，请在主机上以运行器的用户身份运行相同的命令，然后重启运行器：它在启动时读取主机配置一次。`--scope user` 标志是必需的；默认本地作用域写入运行器不播种的按目录密钥下。例如，在您的 Dockerfile 中：

```dockerfile theme={null}
RUN claude mcp add --scope user sidecar -- /usr/local/bin/mcp-sidecar
RUN claude mcp add --scope user --transport http internal http://mcp-gateway.svc.cluster.local:8080
```

运行器在启动时快照主机的配置一次。快照从主机的 `.claude.json` 捕获 `mcpServers` 密钥，该密钥位于 `~/.claude/` 旁边而不是内部，运行器仅将该密钥播种到每个会话的隔离配置中；帐户状态和项目历史被删除。要确认服务器到达会话，请在环境上启动会话并要求 Claude 列出其 MCP 工具；运行器还为任何捕获的条目记录启动警告，其 `type` 它不识别并删除条目，因此您可以看到为什么该服务器从会话中丢失。当设置 `SELF_HOSTED_RUNNER_HOST_CONFIG_DIR` 时，运行器从该目录读取 `.claude.json` 而不是，因此将变量指向空目录也禁用 MCP 播种。

Claude Code 还从其他源加载 MCP 服务器：

* 企业作用域的 [managed MCP file](/docs/zh-CN/managed-mcp) 在其标准系统路径：Linux 运行器主机上的 `/etc/claude-code/managed-mcp.json`，macOS 主机上的 `/Library/Application Support/ClaudeCode/managed-mcp.json`。将其用于锁定的队列，其中只有管理员列出的服务器可能加载。有关优先级规则，请参阅 [exclusive control with managed-mcp.json](/docs/zh-CN/managed-mcp#exclusive-control-with-managed-mcp-json)。当此文件在运行器主机上时，Claude Code 跳过 Anthropic 的控制平面交付给会话的 MCP 服务器（包括 claude.ai 连接器），并在会话子进程的 stderr 上的警告中列出它们的名称，运行器在 `debug` 日志级别记录该警告。在 v2.1.229 之前，这些会话在启动时以 `You cannot dynamically configure MCP servers when an enterprise MCP config is present` 退出。
* 运行器主机上 [managed settings](/docs/zh-CN/managed-settings) 中的 [`managedMcpServers`](/docs/zh-CN/settings-reference#managedmcpservers) 密钥：提供 HTTP 和 SSE 服务器而不获得独占控制，因此来自其他源的服务器仍然加载。需要 Claude Code v2.1.259 或更高版本。
* `<repo>/.mcp.json`：项目作用域。将文件提交到仓库；其服务器在云端会话中自动批准。

当为您的组织启用连接器交付时，Anthropic 的控制平面将您在 claude.ai 上配置的连接器通过服务器提供的 MCP 配置交付给交互式创建的会话，通过 `api.anthropic.com` 路由。以编程方式创建的会话（例如 [CLI dispatches](/docs/zh-CN/self-hosted-environments-testing#run-the-test-loop)）不接收连接器交付；请改为通过本节列出的任何其他源为它们提供 MCP 服务器。子进程的 OAuth 令牌不携带直接获取连接器的作用域，因此子进程不尝试该获取本身；交付是服务器驱动的。

`settings.json` 不携带 MCP 服务器定义，设置 schema 中没有顶级 `mcpServers` 字段。在托管设置中，请改用 [`managedMcpServers`](/docs/zh-CN/settings-reference#managedmcpservers) 密钥提供服务器。

会话继承运行器的环境，因此在那里设置 [`ENABLE_TOOL_SEARCH`](/docs/zh-CN/mcp#scale-with-mcp-tool-search) 以控制运行器生成的每个会话的 MCP 工具搜索；MCP 页面涵盖了这些值。

<h3 id="turn-off-built-in-session-tools">
  关闭内置会话工具
</h3>

Anthropic 的控制平面会将其自己的 MCP 服务器（名为 Claude Code Remote）附加到云端会话。Claude 使用该服务器的工具来安排 [Routine](/docs/zh-CN/routines)、启动和引导其他云端会话、附加更多仓库，以及跟踪 Pull Request 活动。

要关闭整个服务器，请在设置中添加一条[服务器级拒绝规则](/docs/zh-CN/permissions#mcp)。控制平面会根据会话的创建方式，以三个名称之一注册该服务器。Claude Code 会精确匹配规则中的名称（包括大小写），因此请按如下所示为每个名称各写一条规则：

```json theme={null}
{
  "permissions": {
    "deny": [
      "mcp__Claude_Code_Remote",
      "mcp__claude-code-remote",
      "mcp__bf7c680d-5fdc-5ef4-b4a0-abadb619bf0a"
    ]
  }
}
```

指定整个服务器的规则也会覆盖该服务器之后新增的工具。要关闭某一个工具而保留其余工具，请在每条规则后追加两个下划线和工具名称，例如 `mcp__Claude_Code_Remote__add_repo`。如果要完全阻止该服务器连接，而不只是移除其工具，请改为将这三个名称（不带 `mcp__` 前缀）作为 `serverName` 条目添加到 [`deniedMcpServers`](/docs/zh-CN/managed-mcp#policy-based-control-with-allowlists-and-denylists) 下。

将这些规则放在[服务器托管设置](/docs/zh-CN/server-managed-settings)中，即可在不更改运行器的情况下作用于每个会话；也可以放在运行器上的 `~/.claude/settings.json` 中。[权限和工具批准](#permissions-and-tool-approval)说明了运行器上的设置如何到达会话。

要确认规则已生效，请在该环境上启动一个会话，并要求 Claude 列出其 MCP 工具。Claude Code 会从 Claude 的上下文中移除被拒绝的工具，因此这些被拒绝的工具不会出现在其回答中。

<h2 id="prompt-sessions-to-push-their-work">
  提示会话推送其工作
</h2>

Anthropic 托管的会话运行 [`Stop` hook](/docs/zh-CN/hooks#stop)，Claude Code 钩子在 Claude 完成响应时运行，提示 Claude 提交并推送其工作。运行器不安装一个。没有它，以未提交更改结束的会话仅在运行器的磁盘上留下该工作，claude.ai/code 中的 **Create PR** 按钮保持不活跃，直到分支存在于远程。

下面的参考实现有两部分。将设置块合并到运行器主机上的 `~/.claude/settings.json` 中，运行器将其播种到每个会话中，并将脚本保存为运行器主机上的 `~/.claude/hooks/stop-hook-nudge.sh` 并使其可执行：

```json theme={null}
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "timeout": 10,
            "command": "\"$CLAUDE_CONFIG_DIR/hooks/stop-hook-nudge.sh\""
          }
        ]
      }
    ]
  }
}
```

```sh theme={null}
#!/bin/sh
# Stop-hook reference implementation for self-hosted runners.
#
# Nudges Claude once per turn if the project directory has uncommitted
# changes OR unpushed commits, so work isn't lost when an idle session
# is released and so the "Create PR" button on claude.ai/code lights up.
#
# Runner-level (no repo changes): drop this file at ~/.claude/hooks/ on
# the runner host and merge the accompanying Stop-hook settings block
# into ~/.claude/settings.json — the runner seeds both into every session.
# Repo-level alternative: commit to <repo>/.claude/hooks/ and change the
# settings.json command path to $CLAUDE_PROJECT_DIR/.claude/hooks/.
#
# stdin: hook JSON payload (see https://code.claude.com/docs/en/hooks)
# stdout: {"decision":"block","reason":"..."} to nudge, or nothing to allow stop.

# Re-entry guard: the harness sets stop_hook_active=true when re-invoking
# the Stop hook after a block. Bail so we only nudge once per turn. The
# harness emits compact JSON (no space after the colon), which this
# pattern relies on; use jq if you need a whitespace-tolerant check.
in=$(cat)
case "$in" in *'"stop_hook_active":true'*) exit 0 ;; esac

d="$CLAUDE_PROJECT_DIR"

# Not a git repo → nothing to nudge.
git -C "$d" rev-parse --git-dir >/dev/null 2>&1 || exit 0

# No remote → "push to the remote" is unsatisfiable; bail.
[ -z "$(git -C "$d" remote 2>/dev/null)" ] && exit 0

# Uncommitted changes (staged, unstaged, or untracked). Exclude .claude/
# entirely — operator-seeded settings and CLI-written runtime state
# (scheduler lock, worktrees, routine state) live there and neither is
# "uncommitted work" the model needs to push.
s=$(git -C "$d" status --porcelain -- . ':(exclude).claude/' 2>/dev/null)
if [ -n "$s" ]; then
  printf '{"decision":"block","reason":"There are uncommitted changes in the repository. Please commit and push these changes to the remote branch."}'
  exit 0
fi

# Unpushed commits. Count commits on HEAD not reachable from any
# remote-tracking ref or FETCH_HEAD. This works uniformly for:
#   - init+fetch checkouts (runner default: only FETCH_HEAD exists)
#   - clone-based checkouts (origin/* exist)
#   - the runner default: the child starts on the session's outcome
#     branch, which the runner creates after checkout
#   - detached HEAD, when a custom setup skips that branch creation
# With no reference point at all (never fetched), stay silent rather
# than false-positive on a read-only turn.
base=""
git -C "$d" rev-parse --verify -q FETCH_HEAD >/dev/null && base="FETCH_HEAD"
if [ -z "$base" ] && [ -z "$(git -C "$d" for-each-ref --count=1 refs/remotes/origin 2>/dev/null)" ]; then
  exit 0
fi
# shellcheck disable=SC2086  # $base is either "" or "FETCH_HEAD", intentional word-split
unpushed=$(git -C "$d" rev-list HEAD --not $base --remotes=origin --count 2>/dev/null) || unpushed=0
if [ "$unpushed" -gt 0 ]; then
  branch=$(git -C "$d" symbolic-ref --short -q HEAD)
  if [ -n "$branch" ]; then
    # $branch is attacker-influenced — git-check-ref-format(1) allows `"`
    # in ref names. `\` is forbidden (rule 10) but escaped anyway as cheap
    # defense-in-depth.
    # Escape JSON metacharacters before interpolating into the hand-built
    # payload so a branch like x","continue":false can't inject keys into
    # the hook-output JSON the harness parses. $unpushed is safe — the
    # -gt guard above rejects anything that isn't a plain integer.
    branch_esc=$(printf '%s' "$branch" | sed 's/\\/\\\\/g; s/"/\\"/g')
    printf '{"decision":"block","reason":"There are %s unpushed commit(s) on branch '\''%s'\''. Please push these changes to the remote repository."}' "$unpushed" "$branch_esc"
  else
    printf '{"decision":"block","reason":"There are %s unpushed commit(s) on a detached HEAD. Please create a branch and push it to the remote repository."}' "$unpushed"
  fi
  exit 0
fi

exit 0
```

钩子在会话结束前提示 Claude 提交并推送，当目录不是 git 存储库或没有远程时保持沉默。

<h2 id="permissions-and-tool-approval">
  权限和工具批准
</h2>

自托管会话没有连接的终端，因此未回答的权限提示会使当前轮次停滞，直到用户在 UI 中响应。Anthropic 的控制平面随工作负载一起发送每个会话的工具列表和权限规则；默认配置预批准常规工具调用（包括 `Bash`），并且云端会话[无论处于何种模式都会预批准文件编辑](/docs/zh-CN/permission-modes#switch-permission-modes)。任何未被预批准的调用都会通过会话 UI 进行提示。

<Note>
  仅在会话容器运行时启用了[默认拒绝网络出口](/docs/zh-CN/self-hosted-environments-deploy#default-deny-egress)并落实了[加固部分](/docs/zh-CN/self-hosted-environments-deploy#harden-your-deployment)中其余措施的环境上固定自动模式。在默认预批准工具集和自动模式下，常规工具调用（包括 `Bash` 网络请求）都会在无人参与的情况下运行，因此网络边界才是限制这些调用可访问范围的关键。
</Note>

要无论控制平面发送什么都将提示保持在最低限度，请从您的包装脚本或 [`command` hook](#command) 固定[自动模式](/docs/zh-CN/permission-modes#eliminate-prompts-with-auto-mode)。自动模式让会话无需常规权限提示即可运行：单独的分类器模型在操作运行前对其进行审查，并阻止它拒绝的操作，而显式的询问规则仍会强制提示；权限模式页面介绍了分类器检查的内容。运行器在调用包装脚本前追加服务器计算的标志，对于单值标志（如 `--permission-mode`），解析器以最后一次出现的值为准，因此您在 `"$@"` 之后追加的标志会覆盖服务器发送的值：

```bash theme={null}
#!/bin/bash
exec "$CLAUDE_RUNNER_CLAUDE_BIN" "$@" --permission-mode auto
```

要改为预批准特定工具，请追加 `--allowed-tools` 和您的规则，例如 `--allowed-tools "Bash(bazel *) Bash(yarn *) mcp__internal__*"`。列表标志（如 `--allowed-tools` 和 `--disallowed-tools`）会在多次出现时累积而不是覆盖，因此您的规则会叠加在控制平面发送的任何规则之上。要缩小范围，请追加 `--disallowed-tools`，即使其他规则允许某些工具，它也会拒绝这些工具。

<h3 id="how-each-session’s-config-is-assembled">
  每个会话的配置如何组装
</h3>

运行器为每个会话提供自己的配置目录，该目录以运行器在启动时一次性捕获的主机 `~/.claude/` 快照为初始内容：您的运行器镜像中的 `settings.json`、`CLAUDE.md`、hook、Agent、命令和 skill 会作为用户级基线应用于每个会话。如果您更改正在运行的主机上的配置，更改仅在您重启运行器后生效。

设置 `SELF_HOSTED_RUNNER_HOST_CONFIG_DIR` 可从其他路径获取初始内容，或将其指向空目录以禁用初始内容填充。

仓库中提交的 `.claude/settings.json` 会作为项目设置叠加在其上。会话还会从运行器镜像中的标准系统路径读取 [`managed-settings.json`](/docs/zh-CN/settings#where-settings-live)。其中的键是否与[服务器托管设置](/docs/zh-CN/server-managed-settings)一起应用，取决于 [Claude Code 如何合并托管来源](/docs/zh-CN/managed-settings#how-claude-code-combines-managed-sources)：默认情况下，当您的组织下发了任何服务器托管的键时，会话会忽略运行器镜像中的该文件，但 [Claude Code 从每个管理员来源读取的键](/docs/zh-CN/managed-settings#keys-read-from-every-admin-source)除外，例如 `env` 块、沙箱锁定、沙箱二进制路径和 `forceRemoteSettingsRefresh`。请参阅[设置优先级](/docs/zh-CN/settings#settings-precedence)。

当 Anthropic 的控制平面为会话提供 [Claude Code hook](/docs/zh-CN/hooks) 时，运行器会将它们与您自己的配置并行安装，而不是覆盖您的配置。需要 Claude Code v2.1.229 或更高版本。

* **安装位置**：运行器将提供的每个 hook 脚本写入会话配置目录中保留的 `hooks/.ccr-launcher/` 子目录，并在一个单独的设置文件中注册这些脚本，该文件通过 `--settings` 传递给会话，从而使初始填充的 `settings.json` 以及您位于 `hooks/<name>` 的脚本保持不变。运行器会为每个会话重新创建该保留子目录，并且不会将主机上 `~/.claude/hooks/.ccr-launcher/` 中的内容填充到会话中。
* **编写者**：控制平面使用其自身部署中的固定常量填充这些脚本，绝不使用按会话或第三方的输入。
* **仍然适用的管控**：通过 `--settings` 下发的 hook 会进入普通的合并 hook 配置，而不是托管层，因此您的托管设置仍然适用。`disableAllHooks` 会禁用它们，并且它们不属于 [`allowManagedHooksOnly`](/docs/zh-CN/settings-reference#allowmanagedhooksonly) 保持加载的类别。

除 [Claude Tag](https://claude.com/docs/claude-tag/overview) 会话外，自托管环境中的会话默认关闭[自动记忆](/docs/zh-CN/memory#auto-memory)。对于需要跨会话保留的指令，请使用运行器镜像或仓库中的 `CLAUDE.md`。

运行器对主机 `~/.claude/` 的快照不包含 `projects/` 目录。自动记忆的默认存储位置就在该目录下。如果您将记忆文件放在那里，运行器不会将它们填充到会话中，它们也不会启用自动记忆。

<h3 id="repository-committed-permission-rules">
  仓库中提交的权限规则
</h3>

不要在仓库中提交的 `permissions.allow` 中放置不带限定的 `"Edit"`、`"Write"` 或 `"NotebookEdit"` 条目。不带限定的文件工具规则会匹配该工具而不论路径如何，从而授予在主机上任意位置写入的权限，而不仅限于工作区，因此运行器的写入范围限制守卫会标记该会话；使用 [`--confine-repo-settings enforce`](/docs/zh-CN/self-hosted-environments-reference#runner-cli-flags) 时，它会拒绝生成该会话，而不是记录日志后继续。请参阅[加固部分](/docs/zh-CN/self-hosted-environments-deploy#harden-your-deployment)。

仓库根本不需要文件工具规则：云端会话[无论处于何种模式都会预批准文件编辑](/docs/zh-CN/permission-modes#switch-permission-modes)。如果您确实要提交规则，请将其限定到工作区，例如 `"Edit(/**)"`；单个前导斜杠相对于项目根目录，即会话的工作区。不带限定的文件工具规则可以放在操作员的主机级 `settings.json` 中，因为该文件不是在仓库中提交的。

`defaultMode` 为 `auto` 的设置仅在来自镜像范围或用户级设置文件时才会生效，因此检出的仓库无法为自己授予自动模式。有关云端会话接受哪些模式以及完整的规则语法，请参阅[权限模式](/docs/zh-CN/permission-modes)。

<h2 id="what’s-next">
  接下来
</h2>

* [Reference](/docs/zh-CN/self-hosted-environments-reference)：每个 CLI 标志、环境变量和指标
* [Verify session identity](/docs/zh-CN/self-hosted-environments-identity)：从运行器外部的服务验证会话令牌
