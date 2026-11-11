> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 自托管环境参考

> 自托管运行器和编排器的完整参考：CLI 标志、环境变量和 Prometheus 指标。

> [!NOTE]
>
> 自托管环境在 Team 和 Enterprise 计划上处于公开测试阶段；[所有者](https://code.claude.com/docs/zh-CN/cloud-environments.md#organization-shared-environments)通过在[**云环境**管理页面](https://claude.ai/admin-settings/cloud-environments)上打开**允许自托管环境**来启用它们。本页面是标志和指标参考；有关设置，请参阅[快速入门](https://code.claude.com/docs/zh-CN/self-hosted-environments-quickstart.md)，有关舰队配方，请参阅[部署到生产](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md)。

本页面是您在[自托管环境](https://code.claude.com/docs/zh-CN/self-hosted-environments)中运行的两个进程的参考：运行器，它在您的主机上执行 Claude Code [云会话](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md)，以及可选的自动扩展编排器，它在会话队列时启动运行器。每个都有自己的标志表。两者都在 Linux 或 macOS 主机上运行，默认值如 `/workspace` 和 `~/.claude` 假设。运行 `claude self-hosted-runner --help` 以获取已安装版本上的权威列表。

指标系列和一些 API 字段仍然使用 `pool` 来表示这些页面所称的环境；两个术语都指同一事物。环境 ID 是 `pool_id` 字段，形式为 `ccpool_...`：无论这些页面在哪里显示 `pool` 标识符，它都命名环境。CLI 标志和环境变量将其拼写为 `environment`，例如 `--environment-secret-file`；已弃用的 `pool` 拼写仍然有效，如[`--environment-secret-file` 行](#runner-cli-flags)所述。

## Runner CLI 标志

大多数标志都有对应的环境变量。两者同时设置时，标志优先。时长类标志在 CLI 中以分钟或秒为单位，但对应的环境变量始终以毫秒为单位（由 `_MS` 后缀表示），而“默认值”列显示的是标志的单位：`--exit-if-unused-min 10` 等同于 `SELF_HOSTED_RUNNER_IDLE_SHUTDOWN_MS=600000`，而像 `SELF_HOSTED_RUNNER_STARTUP_TIMEOUT_MS: "15"` 这样的 Helm 值表示 15 毫秒，而不是默认的 15 分钟。

| 标志 | 环境变量 | 默认值 | 描述 |
| :- | :- | :- | :- |
| `--api-url <url>` | 无 | `https://api.anthropic.com` | API 基础 URL。仅在测试时覆盖。 |
| `--base-dir <path>` | `SELF_HOSTED_RUNNER_BASE_DIR` | `/workspace`；在 Windows 上无默认值 | 用于仓库检出和每个会话工作目录的目录。runner 需要对此路径或其父目录具有写入权限。runner 在启动时创建该目录，若无法创建或写入该目录，则以 `cannot create or write to base directory` 退出。在 v2.1.225 之前，runner 在第一个会话启动时才创建该目录，因此不可用的路径会导致会话失败，而不是启动失败。在 Windows 上（Windows 不是受支持的 runner 主机）没有默认值：除非您传入该标志或设置该变量，否则 runner 会在启动时退出。在同一环境中的每个 runner 上使用相同的值。请参阅[在各 runner 之间保持基础目录和容量一致](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#keep-the-base-directory-and-capacity-identical-across-runners)。 |
| `--capacity <n>` | 无 | `1` | 此 runner 处理的最大并发会话数。所有会话都属于同一个已锁定的[所有者](https://code.claude.com/docs/zh-CN/self-hosted-environments#key-concepts)。在同一环境中的每个 runner 上使用相同的值；请参阅[在各 runner 之间保持基础目录和容量一致](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#keep-the-base-directory-and-capacity-identical-across-runners)。 |
| `--client-label <label>` | `SELF_HOSTED_RUNNER_CLIENT_LABEL` | 主机的主机名 | runner 注册时发送的标签。runner 还会将其作为 [`claude_code_self_hosted_runner_info`](#prometheus-metrics) 的 `client_label` 标签进行报告。需要 Claude Code v2.1.248 或更高版本。 |
| `--configure-git` | `SELF_HOSTED_RUNNER_CONFIGURE_GIT=1` | 关闭 | 在启动时写入全局 git 身份、启用 Anthropic 提交签名、打开 git push 协商，并安装用于追加 `Co-authored-by:` 尾注的提交钩子。push 协商需要 Claude Code v2.1.257 或更高版本。请参阅[配置 git](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#configure-git)。 |
| `--confine-repo-settings <mode>` | `SELF_HOSTED_RUNNER_CONFINE_REPO_SETTINGS` | `warn` | 设置防护机制的模式：当仓库中已提交的设置试图授予该会话自身工作区之外的写入或读取权限、设置环境变量，或覆盖运维人员的沙箱或 hook 策略（例如 `sandbox.enabled: false` 或 `disableAllHooks`）时，该防护机制会标记该会话。默认值 `warn` 会记录违规并仍然启动会话，`enforce` 会拒绝该会话，`off` 则禁用扫描。请参阅[加固您的部署](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#harden-your-deployment)。 |
| `--debug-token-dir <path>` | `SELF_HOSTED_RUNNER_DEBUG_TOKEN_DIR` | 未设置 | 将有效令牌写入磁盘以供检查。仅用于调试；请勿在生产环境中使用。 |
| `--defer-shutdown-max-min <n>` | `SELF_HOSTED_RUNNER_DEFER_SHUTDOWN_MAX_MS` | `0` | 收到第一个 `SIGTERM` 或 `SIGINT` 时，继续为已附加的会话提供服务而不是排空它们，然后在 N 分钟后释放仍附加的所有会话并退出。设置此项之前，请先调高主机的停止超时时间。请参阅[将排空推迟到第一个信号之后](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#defer-the-drain-past-the-first-signal)。`0` 表示禁用。需要 Claude Code v2.1.238 或更高版本。 |
| `--drain-grace-sec <n>` | `SELF_HOSTED_RUNNER_DRAIN_GRACE_MS` | `0` | 在 runner 收到关闭信号或到达其退役时间之前，控制 runner 在其活动会话结束后何时退出：`0` 表示立即退出而不再轮询更多工作；正值则使 runner 保持存活，并先在该秒数内重新轮询已锁定所有者的队列，代价是失去[加固部分](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#harden-your-deployment)中所述的每会话容器隔离。如果您使用 [`--defer-shutdown-max-min`](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy#defer-the-drain-past-the-first-signal) 推迟了第一个信号，则无论此处如何设置，runner 都会在不再持有任何会话时立即退出。 |
| `--drain-marker-file <path>` | `SELF_HOSTED_RUNNER_DRAIN_MARKER_FILE` | 未设置 | 标记文件，由您的主机在发送 `SIGTERM` 之前写入，以通告一次正常排空。如果排空开始时该文件存在，runner 会向 Anthropic 报告其退出原因为主机排空，而不是普通的关闭信号。排空过程本身（包括 `--drain-wait-sec` 的等待）与不使用该标志时相同。请指定本地文件系统上会话无法写入的路径。需要 Claude Code v2.1.271 或更高版本。 |
| `--drain-wait-sec <n>` | `SELF_HOSTED_RUNNER_DRAIN_WAIT_MS` | `0` | 排空开始后（除非您设置了 [`--defer-shutdown-max-min`](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy#defer-the-drain-past-the-first-signal)，否则排空在收到 `SIGTERM` 时开始），最多等待 N 秒，让每个会话正在进行的轮次和后台任务完成，然后再终止子进程。在此等待期间，runner 会将刚刚完成的后台任务视为仍在运行，直到读取其结果的后续轮次开始为止，最长不超过 [`SELF_HOSTED_RUNNER_BG_RESULT_GRACE_MS`](#environment-variable-only-settings) 窗口。[同时调高您的停止超时时间](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#let-running-turns-finish-before-a-restart)。 |
| `--environment-secret-file <path>` | `SELF_HOSTED_RUNNER_ENVIRONMENT_SECRET` | 必需 | 包含环境密钥的文件路径；对于由[编排器](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#on-demand-runners)生成的 runner，则为一次性工作单 JWT 的文件路径。`SELF_HOSTED_RUNNER_ENVIRONMENT_SECRET` 直接携带密钥值，而不是文件路径。旧的 `--pool-secret-file` 标志和 `SELF_HOSTED_RUNNER_POOL_SECRET` 变量仍然有效，并会向 stderr 打印弃用通知；早于 2.1.216 的预览计划 runner 构建版本只识别这些旧名称。 |
| `--exec-path <path>` | `SELF_HOSTED_RUNNER_EXEC_PATH` | 自身二进制文件 | 为每个会话生成的二进制文件或包装脚本。请参阅[包装脚本](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#wrapper-scripts)。 |
| `--exit-if-unused-min <n>` | `SELF_HOSTED_RUNNER_IDLE_SHUTDOWN_MS` | `0` | 在轮询 N 分钟且从未分配到任何工作后退出，用于自动扩缩器缩容。`0` 表示禁用。 |
| `--git-host-rewrite <from>=<to>` | 无 | 未设置 | 在克隆之前将 `https://<from>/...` 源 URL 改写为 `https://<to>/...`，用于拆分视图 DNS。可重复使用；仅限标志。 |
| `--git-ssh-rewrite <host>` | 无 | 未设置 | 在克隆之前将 `https://<host>/...` 源 URL 改写为 `git@<host>:...`，用于仅支持 SSH 的 git 主机。可重复使用；仅限标志。 |
| `--health-port <port>` | `SELF_HOSTED_RUNNER_HEALTH_PORT` | `8080` | `/healthz` 和 `/metrics` 监听器的端口。设置为 `0` 可禁用。 |
| `--hold-idle-file <path>` | `SELF_HOSTED_RUNNER_HOLD_IDLE_FILE` | 未设置 | 由您的主机持续更改的文件，用于在 runner 上保留空闲会话。请参阅[在进程运行期间保留空闲会话](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#keep-an-idle-session-while-its-processes-run)。需要 Claude Code v2.1.311 或更高版本。 |
| `--hooks-dir <path>` | `SELF_HOSTED_RUNNER_HOOKS_DIR` | 未设置 | 生命周期 hook 脚本所在的目录。请参阅[生命周期 hook](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#lifecycle-hooks)。 |
| `--host-config-snapshot <mode>` | `SELF_HOSTED_RUNNER_HOST_CONFIG_SNAPSHOT` | `disk` | runner 保存[主机配置目录](#environment-variable-only-settings)启动快照的位置，runner 以该快照为每个会话提供初始内容。`disk` 会将快照复制到 `--base-dir` 下由 runner 拥有的目录中，并在每次会话启动时根据内存中的摘要校验每个文件。如果副本中的某个文件已被修改，该会话会失败，并且 runner 会拒绝会话，直到您重启它。`memory` 将整个快照保存在堆上，上限为 64 MiB；超过上限时，会话将在没有主机配置的情况下启动，并显示相应通知。当 runner 无法写入磁盘快照时，它会记录该失败，并在本次运行中使用 `memory`。需要 Claude Code v2.1.271 或更高版本。 |
| `--kill-session-after-min <n>` | `SELF_HOSTED_RUNNER_MAX_LIFETIME_MS` | `0` | 将会话的挂钟时间限制为 N 分钟，作为针对卡住会话的安全限制。在 v2.1.260 或更高版本中，runner 会释放达到该限制的会话，以便其在用户发送下一条消息时恢复；只有当 [`SELF_HOSTED_RUNNER_MAX_LIFETIME_GRACE_MS`](#environment-variable-only-settings) 宽限窗口结束时该会话仍在 runner 上，才会将其终止。在 v2.1.260 之前，runner 会在达到限制时终止会话。有关详细信息以及如何选择取值，请参阅[某些会话不计为空闲](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#some-sessions-don’t-count-as-idle)。`0` 表示禁用。 |
| `--lock-to-account <id>` | `SELF_HOSTED_RUNNER_LOCK_TO_ACCOUNT` | 未设置 | 在启动时将 runner 预先锁定到特定账户，而不是在第一个会话时锁定。接受该环境所属组织中的电子邮件地址或 `user_...` ID。预先锁定的 runner 永远不会接收 Claude Tag 频道会话，因为这些会话没有账户。 |
| `--log-file <path>` | `SELF_HOSTED_RUNNER_LOG_FILE` | 未设置 | 除 stdout 和 stderr 外，还将 runner 日志镜像写入一个文件，该文件以 `0600` 权限创建。`self-hosted-runner doctor` 要在本地跟踪日志，必须设置此项。 |
| `--log-level <level>` | 无 | `info` | `info` 或 `debug` |
| `--max-hold-min <n>` | `SELF_HOSTED_RUNNER_MAX_HOLD_MS` | `1440` | 与 `--hold-idle-file` 配合使用时，被保留的会话在进入空闲 N 分钟后仍会被释放。取值小于或等于 `--release-idle-session-min` 时不会保留任何会话。`0` 表示关闭保留。需要 Claude Code v2.1.311 或更高版本。 |
| `--post-session-hook-timeout-sec <n>` | `SELF_HOSTED_RUNNER_POST_SESSION_HOOK_TIMEOUT_MS` | `60` | 每次会话结束时（包括 runner 关闭时）[`post-session` hook](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration#post-session) 的时间预算 |
| `--proxy-authorization-command <command>` | `SELF_HOSTED_RUNNER_PROXY_AUTHORIZATION_COMMAND` | 未设置 | runner 在每次连接您的出口代理时运行的 shell 命令，其去除首尾空白后的 stdout 用作 `Proxy-Authorization` 标头值。需要 `HTTPS_PROXY` 或 `HTTP_PROXY`，且不能与 `--proxy-authorization-file` 同时使用。请参阅[向出口代理进行身份验证](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#authenticate-to-an-egress-proxy)。需要 Claude Code v2.1.238 或更高版本。 |
| `--proxy-authorization-file <path>` | `SELF_HOSTED_RUNNER_PROXY_AUTHORIZATION_FILE` | 未设置 | runner 在每次连接您的出口代理时读取的文件，其去除首尾空白后的内容用作 `Proxy-Authorization` 标头值。对于由其他进程原地轮换的令牌，请使用此标志。要求与 `--proxy-authorization-command` 相同，且不能与其同时使用。请参阅[向出口代理进行身份验证](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#authenticate-to-an-egress-proxy)。需要 Claude Code v2.1.238 或更高版本。 |
| `--push-outcome-on-release` | `SELF_HOSTED_RUNNER_PUSH_OUTCOME_ON_RELEASE` | 关闭 | 在 runner 因排空或空闲释放等原因主动结束会话时，runner 会在删除工作区之前，将包含新提交的主会话分支推送到 `origin`。尽力而为；会为关闭预算增加 30 秒。需要 git 2.29 或更高版本才能从这些分支恢复。请将推送权限限制在 `claude/*` 引用；请参阅[恢复的会话会丢失未推送的工作](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#resumed-sessions-lose-unpushed-work)。`checkout` hook 检出的仓库不会被推送；请在 [`post-session` hook](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration#post-session) 中为其创建快照。 |
| `--release-idle-session-min <n>` | `SELF_HOSTED_RUNNER_SESSION_IDLE_MS` | `0` | 在轮次结束或会话等待用户操作后，若不活动达 N 分钟，则释放会话槽位。[某些会话不计为空闲](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#some-sessions-don’t-count-as-idle)，因此请搭配 `--kill-session-after-min` 使用。已完成的后台任务会使会话保持忙碌状态，最长不超过 [`SELF_HOSTED_RUNNER_BG_RESULT_GRACE_MS`](#environment-variable-only-settings) 窗口。要推迟释放，请设置 [`--hold-idle-file`](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy#keep-an-idle-session-while-its-processes-run)。`0` 表示禁用。 |
| `--remove-session-state [bool]` | `SELF_HOSTED_RUNNER_REMOVE_SESSION_STATE` | 关闭 | 当会话在此 runner 上结束时，无论结果如何，都删除该会话在 `<base-dir>/_sessions/` 下的每会话目录。[复用预热的检出](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#reuse-a-pre-warmed-checkout)介绍了这些目录保存的内容，以及它们保留时谁可以读取。删除是尽力而为的：如果 runner 在清理运行之前被终止或到达其排空截止时间，每会话目录将保留原位。启用该标志后，失败或中断的会话的调试日志不会保留在磁盘上。需要 Claude Code v2.1.268 或更高版本。 |
| `--retire-at <epoch-seconds>` | `SELF_HOSTED_RUNNER_RETIRE_AT` | 未设置 | 在以秒为单位的绝对 Unix 时间戳处让 runner 退役，适用于会在已知时间终止 runner 的基础设施；[Runner 生命周期](https://code.claude.com/docs/zh-CN/self-hosted-environments#runner-lifecycle)介绍了释放顺序以及如何确定余量。早于 2001 年或晚于 5138 年的值会被该标志拒绝，并被该环境变量忽略。 |
| `--server-auto-mode-lists <mode>` | `SELF_HOSTED_RUNNER_SERVER_AUTO_MODE_LISTS` | `no-allow` | 控制平面随会话发送的[自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode)分类器规则列表中，哪些可以作用于该会话：`all`、`no-allow` 或 `none`。有关每个值应用的内容，请参阅[自动模式规则列表](#auto-mode-rule-lists)。无效值会使 runner 在启动时停止。需要 Claude Code v2.1.295 或更高版本。 |
| `--session-stop-grace-sec <n>` | `SELF_HOSTED_RUNNER_SESSION_STOP_GRACE_MS` | `5` | 会话结束后，在强制终止 Claude 进程之前等待其正常退出的时长。如果子进程自身的 `SessionEnd` hook 需要更多时间，请调高此值。 |
| `--startup-timeout-min <n>` | `SELF_HOSTED_RUNNER_STARTUP_TIMEOUT_MS` | `15` | 在释放会话槽位之前，等待尚未初始化的子进程的时长。有关计时从何处开始以及哪些会话会继续运行，请参阅[会话启动超时](#session-startup-timeout)。`0` 表示禁用超时。 |
| `--trust-workspace [bool]` | `SELF_HOSTED_RUNNER_TRUST_WORKSPACE` | 开启 | 为每个会话的仓库路径预置持久化信任，以便遵循仓库中已提交的 `permissions.allow` 和 `additionalDirectories`。设置为 `false` 可丢弃仓库中已提交的权限授予，并改为在主机配置的 `settings.json` 中配置允许规则；无论哪种方式，仓库中已提交的 `sandbox.*` 设置仍然适用，这也是[仓库设置防护机制](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#harden-your-deployment)无论此标志如何都会扫描它们的原因。 |
| `--use-anthropic-git-proxy` | `CLAUDE_RUNNER_USE_GIT_PROXY=1` | 关闭 | 通过 [Anthropic git 代理](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#use-the-anthropic-git-proxy)克隆 github.com 上的仓库，而不是使用客户自管的 git 身份验证。需要 `--capacity 1` 和 git 2.32 或更高版本，否则 runner 会拒绝启动。还需要 Claude Code v2.1.267 或更高版本。此标志取代改写类标志。 |

大多数时长类标志都有上限，其选择是为了使每个超时都保持在运行时约 24.85 天的 32 位计时器上限之内。`--*-min` 标志的上限为 10080 分钟，即 7 天；`--drain-grace-sec` 为 604800 秒，同样是 7 天；`--drain-wait-sec` 为 86400 秒，即 24 小时。`--session-stop-grace-sec` 和 `--post-session-hook-timeout-sec` 没有上限。超出上限时，不同使用方式的行为有所不同：

* **标志**：启动失败并报错。
* **环境变量**：runner 将该值截断为计时器上限，而不是拒绝它。

### 会话启动超时

如果会话在 `/code-review` 等轮次中丢失了槽位，请将 runner 和子进程二进制文件更新到 v2.1.309 或更高版本。在 v2.1.299 至 v2.1.308 中，一旦启动超时已过且没有 `user` 或 `assistant` 消息，runner 就会释放会话，即使该会话正处于轮次中。

使用 `--startup-timeout-min <n>` 时，如果会话的子进程在克隆后 `<n>` 分钟内未完成初始化，runner 会释放该会话的槽位。在子进程初始化之前开始的轮次中，会话会保留其槽位，直到子进程的[活动通道](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#keep-stdin-and-file-descriptor-3-attached)上连续 `<n>` 分钟没有任何输出行为止。子进程初始化后，runner 会清除该计时，改由 `--release-idle-session-min` 控制释放。要限制子进程仍存活但已卡住的会话，请设置 `--kill-session-after-min`。

### 自动模式规则列表

`--server-auto-mode-lists` 让您决定来自 runner 外部的哪些[自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode)分类器规则可以作用于您的 runner 上的会话。Anthropic 的控制平面可以随会话发送规则列表，并请求 runner 应用它们。其中一些条目可能是您组织的管理员编写的规则。这些列表是 `environment`、`soft_deny` 和 `allow`：

* **`environment`**：其中的条目既可以让分类器允许更多操作，也可以让其允许更少操作。
* **`soft_deny`**：其中的条目会阻止某项操作，除非用户明确要求执行该操作，或适用某个 `allow` 例外。
* **`allow`**：`soft_deny` 条目的例外。

该标志的值决定 runner 应用哪些列表：

* **`no-allow`**：默认值。应用 `environment` 和 `soft_deny`，不应用 `allow`。`environment` 条目仍可能让分类器允许更多操作，因此默认值并不能排除所有放宽。
* **`all`**：应用全部三个列表。
* **`none`**：一个都不应用。若要排除来自这些列表的所有放宽，请选择 `none`。这也会去掉 `soft_deny` 的限制。

没有任何 runner 设置能让控制平面请求 runner 应用这些列表。当控制平面未发出请求时，无论您如何设置，会话都不会收到任何列表。要查看实际情况，请使用 `--log-level debug` 启动 runner。随后，runner 会为每个会话记录一行日志，其中包含 `the server asked this runner to apply`，或包含 `the server did not ask this runner to apply the auto mode lists it sends`。

## Orchestrator CLI 标志

`self-hosted-runner orchestrator` 子命令（生成[按需运行器](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#on-demand-runners)）接受 `--api-url`、`--environment-secret-file`、`--hooks-dir`、`--health-port` 和 `--log-level`，与运行器具有相同的默认值，以及运行器的标志具有的相同环境变量（除了 `--hooks-dir` 是必需的，必须包含 `spawn-runner` 钩子）。它还采用自己的标志：

| 标志 | 默认值 | 描述 |
| :- | :- | :- |
| `--hook-concurrency <n>` | `4` | 并行运行的 `spawn-runner` hook 的最大数量。同时限制每次轮询认领的生成请求数。编排器仅在其启动的每个 hook 都完成后才会检查新的生成请求，因此一个缓慢的 hook 会延迟该编排器的后续 hook。 |
| `--hook-timeout <sec>` | `60` | 在这么多秒后终止钩子的进程树。超时加其 5 秒杀死宽限期必须保持在 `--expected-spawn-seconds` 以下；编排器在启动时强制执行此操作。 |
| `--expected-spawn-seconds <sec>` | `120` | 从编排器收到生成请求到运行器注册的预期 p99 时间，包括在您的平台上等待容量的任何时间。服务器强制的范围为 10 到 3600。在每次轮询时作为服务器端租约发送；如果没有运行器在其过期前注册，会话将使用新的订单 ID 重新提供。所有副本必须共享此值。 |
| `--min-idle <n>` | `0` | 通过主动生成待命运行器来保持至少 N 个空闲会话槽。`0` 禁用预热。与运行器的 `--exit-if-unused-min` 配对，以便多余的待命运行器回收自己。 |
| `--shutdown-drain-sec <n>` | `0` | 收到 `SIGTERM` 时，停止轮询生成请求，并给正在运行的 `spawn-runner` hook 和任何尚未得到响应的轮询最多这么多秒来完成，之后停止剩余的部分。设为 `0` 时立即停止。最大值为 `3600`，编排器在启动时强制执行此限制。要选择合适的值，请参阅[确定 `--shutdown-drain-sec` 和编排器的停止超时时间](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration#size-shutdown-drain-sec-and-the-supervisors-stop-timeout-for-the-orchestrator)。需要 Claude Code v2.1.299 或更高版本。 |
| `--debug-dir <path>` | 未设置 | 将每个生成请求的工作订单和钩子 stderr 写入磁盘。仅用于调试；永远不要在生产中设置。 |

### SCM 连接器标志

SCM 连接器不可用，因此请将本部分中的标志保持未设置。如果您设置 `--scm-connector-host`，连接不会打开，编排器会继续重试。运行器仍然作为会话队列启动。

连接器是从编排器到 Anthropic 控制平面的常设 WebSocket 连接。它的设计目的是让托管的预会话流（例如存储库选择器和分支或 ref 解析器）能够到达仅从您的网络内部可路由的 GitHub Enterprise Server 主机。请参阅 GitHub Enterprise Server 页面上的[网络要求](https://code.claude.com/docs/zh-CN/github-enterprise-server.md#network-requirements)，了解这些流需要什么。

| 标志 | 默认值 | 描述 |
| :- | :- | :- |
| `--scm-connector-host <host[:port]>` | 未设置 | GitHub Enterprise Server 主机名以转发请求。端口默认为 `443`。 |
| `--scm-connector-id <n>` | 与 `--scm-connector-host` 一起需要 | 您的组织的 GitHub Enterprise Server 连接的数字 ID。 |
| `--scm-connector-provider <slug>` | `ghe` | 标识提供程序的路径段，匹配 `^[a-z0-9-]{1,32}$`。 |
| `--scm-connector-ca-file <path>` | 未设置 | 额外的 CA 包，PEM 格式，用于到 GitHub Enterprise Server 主机的 TLS 连接。 |
| `--scm-connector-host-rewrite <from>=<to_host:to_port>` | 未设置 | 仅用于端到端测试：重定向 TCP 连接，同时将 Host 标头和 TLS SNI 保持为 `--scm-connector-host`。 |

在每次连接尝试时，编排器发送其现有的环境密钥，并使用指数退避自动重试，上限为 30 秒加抖动。

## 仅环境变量设置

这些运行器设置仅从环境读取，涵盖大多数部署保留在默认值的行为：

| 环境变量 | 默认值 | 描述 |
| :- | :- | :- |
| `SELF_HOSTED_RUNNER_BG_RESULT_GRACE_MS` | `30000` | 运行器在后台任务完成后认为会话繁忙的时间，而读取结果的后续轮次尚未开始。[`--drain-wait-sec` 和 `--release-idle-session-min` 行](#runner-cli-flags)描述了保持在排空和空闲释放时的应用位置，[运行器生命周期](https://code.claude.com/docs/zh-CN/self-hosted-environments#runner-lifecycle)描述了它在 `--retire-at` 退休时的应用位置。`0` 或不可用的值回退到默认值，因此无法关闭保持。需要 Claude Code v2.1.228 或更高版本。 |
| `SELF_HOSTED_RUNNER_HOST_CONFIG_DIR` | `~/.claude` | 捕获到运行器启动快照中并播种到每个会话的 `CLAUDE_CONFIG_DIR` 的目录；磁盘上的更改在运行器重新启动后应用。设置变量也会移动运行器读取 `.claude.json` 的位置以进行 [MCP 播种](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#mcp-servers)，因此设置它（包括其自己的默认值）会重新定位该查找；指向空目录以完全禁用播种。 |
| `SELF_HOSTED_RUNNER_MAX_LIFETIME_GRACE_MS` | `900000` | 运行器在会话达到其 `--kill-session-after-min` 限制后等待的时间，以便运行中的轮次完成或释放完成，然后才终止会话 |
| `SELF_HOSTED_RUNNER_POST_TURN_SETTLE_MS` | `7000` | 运行器在轮次完成后计算会话繁忙的时间上限，用于 `--drain-wait-sec` 排空，而会话的进程向 Anthropic 报告轮次的结束。`0` 或不可用的值回退到默认值，因此无法关闭保持。需要 Claude Code v2.1.275 或更高版本。 |
| `SELF_HOSTED_RUNNER_SIGKILL_GRACE_MS` | `30000` | 运行器等待操作系统向陷入不可中断 I/O 的子进程传递 `SIGKILL` 的时间，然后自己退出。下限为 `--post-session-hook-timeout-sec` 加 15 秒，设置 `--push-outcome-on-release` 时再加 30 秒，因此有效最小值在默认值处为 75 秒。 |
| `CLAUDE_RUNNER_DISABLE_RENEWAL_WARNING` | 未设置 | 当为 `1` 时，运行器不会在会话中发布 `The server has refused to renew this session's access to the model`。每次被拒绝后，它会等到下一次计划的续订时再次请求。需要 Claude Code v2.1.319 或更高版本。 |
| `CLAUDE_RUNNER_FETCH_DEPTH` | `50` | 新克隆的 git 获取深度。设置正整数，或 `full` 或 `0` 以进行完整获取。对于工作区中已存在的克隆，请参阅[复用预热的检出](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#reuse-a-pre-warmed-checkout)。 |
| `CLAUDE_RUNNER_FETCH_SERVER_PROGRESS_CAP_MS` | `600000` | 在 git 服务器自身的进度数字持续上升期间（例如服务器为大型仓库准备 pack 时），每次尝试中 git 获取等待首批数据的最长时间（以毫秒为单位）。`0` 或 `off` 会关闭此等待：此类获取在两分钟内没有收到数据时将被中断。任何其他整数都会被限制在 `120000` 到 `1800000` 之间，即 2 到 30 分钟。需要 Claude Code v2.1.295 或更高版本。 |
| `CLAUDE_RUNNER_SKIP_GIT_VERIFY` | 未设置 | 当为 `1` 时，跳过 `checkout` 钩子运行后的 `.git` 存在检查。当您的钩子物化非 git 源时设置此项。 |
| `FORCE_AUTOUPDATE_PLUGINS` | 未设置 | 当为 `1` 时，让插件市场自动更新，即使二进制文件被固定 |
| `CLAUDE_CODE_DISABLE_ARTIFACT` | 未设置 | 当为 `1` 时，无论组织的管理员设置如何，都在会话中禁用 Artifact 工具，并删除 `*.frame.claudeusercontent.com` 出口要求 |

## 遥测

会话子进程向 Anthropic 发送操作遥测，除非您关闭它。不发送代码或存储库内容。在运行器进程上设置遥测变量；运行器在应用服务器提供的环境变量后重新声明它们，因此操作员的设置始终优先。

一个控制特定于自托管环境：`CLAUDE_CODE_BYOC_ENABLE_DATADOG=1` 选择加入 Datadog 操作指标，这在自托管环境中默认关闭。一般 Claude Code 遥测控制 `DISABLE_TELEMETRY`、`DO_NOT_TRACK`、`DISABLE_ERROR_REPORTING` 和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 适用于会话子进程，如[环境变量参考](https://code.claude.com/docs/zh-CN/env-vars.md)中所述。[`CLAUDE_CODE_DISABLE_FEATURE_FLAGS`](https://code.claude.com/docs/zh-CN/env-vars#variables) 相关但不同：设置 `CLAUDE_CODE_DISABLE_FEATURE_FLAGS=1` 禁用功能标志获取，遥测保持开启，除非也设置了 `DISABLE_TELEMETRY`。`CLAUDE_CODE_DISABLE_FEATURE_FLAGS` 需要 Claude Code v2.1.319 或更高版本。

`CLAUDE_CODE_ENABLE_TELEMETRY` 无关：它启用 OpenTelemetry 导出到您自己的收集器，如[监控](https://code.claude.com/docs/zh-CN/monitoring-usage.md)中所述，不控制 Anthropic 的分析。

## 健康端点

运行器在完成注册并结束启动检查后，会在配置的健康端口上提供 `GET /healthz`。从那时起，只要进程活跃，响应就是 `200 OK`，无论轮询循环处于什么状态，因此此端点上的 HTTP 探针仅检测死进程。JSON 正文描述当前状态：

```json theme={null}
{
  "status": "ok",
  "runner_id": "ccrunner_...",
  "active_sessions": 2,
  "last_poll_at": "2026-03-31T18:04:11.220Z",
  "last_poll_age_ms": 842
}
```

基于 `last_poll_age_ms` 设置警报，但不要在存活探针中使用它；存活探针应继续依据 HTTP 状态代码。该值是自上次成功轮询以来经过的时间，因此当运行器无法连接到 Anthropic 时它也会上升。在第一次轮询成功之前，`last_poll_at` 和 `last_poll_age_ms` 都是 `null`。要捕获从一开始就失败的轮询，还应针对 `claude_code_self_hosted_runner_poll_errors_total` 设置警报，该指标列在 [Prometheus 指标](#prometheus-metrics)下。

编排器自己的 `/healthz`（位于其健康端口上）始终返回 `200`。正文携带报告最近一次轮询是否成功的 `connected` 字段，加上 `queue_counts` 中的每个状态生成队列计数。要了解 `connected` 为何是 `false`，请读取顶层 `last_error` 字段，每次成功轮询都会将其重置为 `null`。使用 `connected` 而不是状态代码来控制就绪和警报。

当[SCM 连接器](#scm-connector-flags)被配置时，编排器的 `/healthz` 正文也携带 `scm_connector_connected` 和一个 `scm_connector` 对象，包含 `connected`、`last_connected_at`、`last_error`、`reconnects` 和 `requests_forwarded`。当未设置 `--scm-connector-host` 时，两个字段都是 `null`。

## Prometheus 指标

每个运行器都在与 `/healthz` 相同的端口上通过 `GET /metrics` 提供 Prometheus 指标。主要序列：

| 序列 | 说明 |
| :- | :- |
| `claude_code_self_hosted_runner_info{runner_id,version,client_label}` | 始终为 `1`；可用于机群清单和版本漂移检测 |
| `claude_code_self_hosted_runner_capacity` | 配置的 `--capacity` |
| `claude_code_self_hosted_runner_active_sessions` | 当前正在运行的会话 |
| `claude_code_self_hosted_runner_locked_account{email}` | 当运行器已锁定到某个用户，并且已签发携带 `act.email` 声明的会话令牌后出现。对于锁定到 Claude Tag Agent 的运行器，该序列不存在，因为其会话令牌不携带 `act.email`。标签值为账户电子邮件；如果您的指标存储可被广泛读取，请在抓取时删除该标签或对其进行哈希处理，例如使用 Prometheus `metric_relabel_configs`。 |
| `claude_code_self_hosted_runner_last_poll_age_seconds` | 距上次成功轮询的秒数。超过 60 时告警。 |
| `claude_code_self_hosted_runner_poll_errors_total{error_kind}` | 按类型累计的 PollWork 失败次数：`transport`、`timeout`、`5xx`、`429` 或 `4xx`。这五个序列从进程启动时即存在；在 `rate(...[5m]) > 0` 时告警。 |
| `claude_code_self_hosted_runner_sessions_started_total{client_platform}` | 运行器生命周期内生成的会话子进程数，每个会话来源一个序列，例如 `web_claude_ai`、`ios`、`android`、`desktop_app` 或 `claude_code_cli`，服务器未发送来源时为 `unknown`。Slack 会话根据创建它们的 Slack 集成不同，携带 `claude_in_slack` 或 `claude-in-slack`，因此请使用正则选择器同时匹配两者，例如 `{client_platform=~"claude[-_]in[-_]slack"}`。使用 `sum()` 获取整个机群的总数。 |
| `claude_code_self_hosted_runner_sessions_completed_total{client_platform}` | 正常结束的会话，标签方式相同。其范围比单纯的正常退出更广：哪些情况计入，请参阅[会话生命周期计数器语义](#session-lifecycle-counter-semantics)。 |
| `claude_code_self_hosted_runner_sessions_failed_total{client_platform}` | 以失败结束的会话，标签方式相同。同样需注意：请参阅[会话生命周期计数器语义](#session-lifecycle-counter-semantics)。 |
| `claude_code_self_hosted_runner_sessions_interrupted_total{client_platform}` | 运行器出于运维原因而非会话结果终止的会话，标签方式相同。请参阅[会话生命周期计数器语义](#session-lifecycle-counter-semantics)。 |
| `claude_code_self_hosted_runner_initializing_sessions` | 当前处于初始化阶段的会话，从分配开始直到子进程的 init 事件 |
| `claude_code_self_hosted_runner_session_init_duration_seconds` | 会话初始化时长的直方图 |
| `claude_code_self_hosted_runner_session_init_errors_total` | 在到达 init 之前失败的会话：checkout hook 失败、git 准备、令牌问题或 init 前子进程崩溃 |
| `claude_code_self_hosted_runner_session_start_hook_errors_total` | 报告错误结果的 `SessionStart` hook，每次失败的 hook 执行计一次 |
| `claude_code_self_hosted_runner_session_idle_seconds{session_id,client_platform}` | 每个会话的仪表，表示会话进入空闲状态后的秒数。可用于终止卡在未应答权限提示上的会话。 |

编排器在与其 `/healthz` 相同的端口上通过 `GET /metrics` 提供自己的序列：

| 序列 | 说明 |
| :- | :- |
| `claude_code_self_hosted_orchestrator_info{version,pool_id,orchestrator_uuid,hostname}` | 始终为 `1` |
| `claude_code_self_hosted_orchestrator_connected` | 最近一次轮询成功时为 `1`；任何轮询失败后降为 `0`，无论失败类型如何 |
| `claude_code_self_hosted_orchestrator_last_poll_age_seconds` | 距上次轮询尝试（无论成功或失败）的秒数，这与运行器的同名指标不同，后者衡量的是距上次成功的时间；请与 `connected` 搭配使用以捕获失败的轮询。编排器的轮询循环会等待 hook 执行，因此告警阈值应设为 `--hook-timeout` 加上一定余量（默认情况下约 90 秒），而不是固定的 60。 |
| `claude_code_self_hosted_orchestrator_poll_errors_total{error_kind}` | 按类型累计的 PollSpawnHints 失败次数：`transport`、`timeout`、`5xx`、`429` 或 `4xx`。这五个序列从进程启动时即存在；在 `rate(...[5m]) > 0` 时告警。 |
| `claude_code_self_hosted_orchestrator_queue_pending_sessions` | 当前可认领的生成请求 |
| `claude_code_self_hosted_orchestrator_queue_backing_off_sessions` | 在可重试的 hook 失败后处于重试退避中的生成请求 |
| `claude_code_self_hosted_orchestrator_queue_circuit_broken_sessions` | 被阻止生成的会话。每个会话会一直被阻止，直到用户向其发送新消息，或 Owner 从环境的 **Activity** 选项卡重试它。修复原因后，该计数可能仍保持在零以上。高于零时告警。 |
| `claude_code_self_hosted_orchestrator_pool_pending_sessions` | 该环境中等待运行器的会话总数。这是环境范围的聚合值，在每个编排器实例上都相同：跨实例时请使用 `MAX` 而不是 `SUM`。 |
| `claude_code_self_hosted_orchestrator_pool_active_sessions` | 当前分配给该环境中存活运行器的会话数。这是环境范围的聚合值，在每个编排器实例上都相同：跨实例时请使用 `MAX` 而不是 `SUM`。 |
| `claude_code_self_hosted_orchestrator_spawn_hooks_total{result}` | 累计的 `spawn-runner` hook 结果：`ok`、`retryable`、`non_retryable`。统计的是编排器的 hook 调用次数，而不是运行器生成的会话子进程数：它与 `sessions_started_total` 不可比较，因为大于 1 的容量、预热池以及为同一会话再次生成的运行器都会使两者产生偏差。 |
| `claude_code_self_hosted_orchestrator_spawn_hook_duration_seconds` | hook 时长的直方图 |
| `claude_code_self_hosted_orchestrator_warm_hints_dispatched_total` | 自进程启动以来分派的待机生成请求数 |
| `claude_code_self_hosted_orchestrator_session_queue_wait_seconds` | 每个会话在编排器认领其进行生成之前在队列中等待的秒数的直方图，根据控制平面随每个会话的生成请求发送的队列等待时间戳记录。用于 p50/p99 队列时间告警。预热生成不会被采样。 |
| `claude_code_self_hosted_orchestrator_clock_skew_seconds` | 本地减去服务器的时钟偏差；诊断用，测量后出现 |
| `claude_code_self_hosted_orchestrator_scm_connector_connected` | [SCM 连接器](#scm-connector-flags)的 WebSocket 打开时为 `1`；拨号或退避期间为 `0`。未设置 `--scm-connector-host` 时不存在。 |
| `claude_code_self_hosted_orchestrator_scm_connector_requests_forwarded_total` | 自进程启动以来代理到所配置 SCM 主机的累计 HTTP 请求数。未设置 `--scm-connector-host` 时不存在。 |

对于自动扩缩，请选择与您的扩缩方式相匹配的序列，并在其输入扩缩器之前对其进行门控：

* **基于队列深度扩缩**：将 `claude_code_self_hosted_orchestrator_pool_pending_sessions` 输入您的 HPA 或 KEDA 扩缩器，而不是 `queue_pending_sessions`。
* **基于容量扩缩**：根据运行器的 `active_sessions` 与 `capacity` 之比进行扩缩。
* **按 `connected` 门控**：按实例使用 `claude_code_self_hosted_orchestrator_connected == 1` 过滤查询，这样已断开连接的副本的陈旧值就不会输入扩缩器。

在轮询完全中断、所有副本都断开连接期间，门控查询不返回任何数据。HPA 在指标缺失时会保持当前副本数，但 KEDA 的 Prometheus 扩缩器在默认 `ignoreNullValues: "true"` 下会将空结果读取为零并缩容；请在 ScaledObject 上设置 `ignoreNullValues: "false"`，并可选择设置 `fallback` 副本下限。

以下 Prometheus Operator `PodMonitor` 涵盖这两个进程。它通过 `app.kubernetes.io/part-of: claude-code-self-hosted-runner` 标签以及 [Kubernetes 方案](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#kubernetes)所设置的命名 `health` 端口来选择 Pod；请调整命名空间以匹配您的部署：

```yaml theme={null}
# 适用于 Claude Code 自托管运行器 + 编排器的
# Prometheus Operator PodMonitor 示例。请调整命名空间和标签选择器以匹配
# 您的部署。运行器和编排器都在其 --health-port（默认 8080）
# 上提供 /metrics。
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: claude-code-self-hosted-runner
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - claude-runners
  selector:
    matchExpressions:
      # 匹配 Kubernetes 方案中的运行器 Deployment，以及您以相同方式
      # 添加标签并指定了命名 'health' containerPort 的
      # 按需运行器 Job 和编排器 Pod。
      - key: app.kubernetes.io/part-of
        operator: In
        values: [claude-code-self-hosted-runner]
  podMetricsEndpoints:
    - port: health
      path: /metrics
      interval: 30s
```

### 示例告警规则

这些示例告警规则只是一个起点；请根据您的机群规模调整阈值：

```yaml theme={null}
# 适用于 Claude Code 自托管运行器 + 编排器的
# Prometheus 告警规则示例。请根据您的机群规模和 SLO 调整阈值。
groups:
  - name: claude-code-self-hosted-runner
    rules:
      - alert: ClaudeRunnerPollStale
        expr: claude_code_self_hosted_runner_last_poll_age_seconds > 60
        for: 2m
        labels: {severity: warning}
        annotations:
          summary: "Runner {{ $labels.pod }} has not polled in >60s"
      - alert: ClaudeRunnerVersionDrift
        expr: count(count by (version) (claude_code_self_hosted_runner_info)) > 1
        for: 30m
        labels: {severity: info}
        annotations:
          summary: "Runners are running mixed versions"
      - alert: ClaudeRunnerInitErrorsHigh
        expr: sum by (namespace) (increase(claude_code_self_hosted_runner_session_init_errors_total[10m])) > 3
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "Sessions are failing before init (checkout hook / git / token / pre-init crash)"
      - alert: ClaudeRunnerPollErrors
        expr: sum by (pod) (rate(claude_code_self_hosted_runner_poll_errors_total[5m])) > 0
        for: 2m
        labels: {severity: warning}
        annotations:
          summary: "Runner {{ $labels.pod }}: PollWork failing ({{ $value | humanize }}/s over 5m)"
      - alert: ClaudeRunnerSessionStartHookErrors
        expr: sum by (namespace) (increase(claude_code_self_hosted_runner_session_start_hook_errors_total[10m])) > 3
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "SessionStart hooks are failing"

  - name: claude-code-self-hosted-orchestrator
    rules:
      - alert: ClaudeOrchestratorDisconnected
        expr: claude_code_self_hosted_orchestrator_connected == 0
        for: 2m
        labels: {severity: critical}
        annotations:
          summary: "Orchestrator {{ $labels.pod }} cannot reach the Anthropic control plane"
      - alert: ClaudeOrchestratorPollStale
        expr: claude_code_self_hosted_orchestrator_last_poll_age_seconds > 90
        for: 2m
        labels: {severity: warning}
        annotations:
          summary: "Orchestrator {{ $labels.pod }} has not polled in >90s (poll loop waits on hook execution)"
      - alert: ClaudeOrchestratorCircuitBroken
        expr: max by (namespace) (claude_code_self_hosted_orchestrator_queue_circuit_broken_sessions and on (instance) max_over_time(claude_code_self_hosted_orchestrator_connected[2m]) == 1) > 0
        for: 1m
        labels: {severity: critical}
        annotations:
          summary: "Sessions blocked from spawning: {{ $value }}. Read each one's error in the Activity tab, fix the cause, then select Retry"
      - alert: ClaudeOrchestratorPollErrors
        expr: sum by (pod) (rate(claude_code_self_hosted_orchestrator_poll_errors_total[5m])) > 0
        for: 2m
        labels: {severity: warning}
        annotations:
          summary: "Orchestrator {{ $labels.pod }}: PollSpawnHints failing ({{ $value | humanize }}/s over 5m)"
      - alert: ClaudeOrchestratorSpawnHookFailing
        expr: sum by (pod) (increase(claude_code_self_hosted_orchestrator_spawn_hooks_total{result!="ok"}[5m])) > 3
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "Orchestrator {{ $labels.pod }}: >3 spawn-runner hook failures in 5m"
```

### 针对运行器丢失和会话等待进行告警

若要在运行器丢失、会话等待或编排器停机时收到告警，请在[示例告警规则](#sample-alert-rules)之外加载以下 Prometheus 告警规则：

* **[固定机群](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#scale-the-fleet)**：`ClaudeRunnersBelowFloor`
* **[按需运行器](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#on-demand-runners)**：`ClaudeOrchestratorSessionsWaiting`、`ClaudeOrchestratorQueueWaitHigh` 和 `ClaudeOrchestratorAbsent`

#### 固定机群丢失运行器时告警

当已注册且被抓取的运行器数量低于下限并持续 10 分钟时，`ClaudeRunnersBelowFloor` 会触发：

```yaml theme={null}
groups:
  # 仅适用于固定机群。对于按需运行器，数量随需求变化，
  # 因此每当需求较低时 ClaudeRunnersBelowFloor 都会触发。
  - name: claude-code-self-hosted-fixed-fleet-capacity
    rules:
      - alert: ClaudeRunnersBelowFloor
        expr: (count(claude_code_self_hosted_runner_info{namespace="claude-runners"}) or vector(0)) < 2
        for: 10m
        labels: {severity: critical, namespace: claude-runners}
        annotations:
          summary: "Runners registered and scraped: {{ $value }}, below the floor. Check the runner workloads first, then the scrape targets if all are running."
```

* **下限**：将 `expr` 中的 `2` 设置为副本数的一半左右，向上取整。如果有三个副本，则保留 `2`，当只剩一个运行器或没有运行器时告警会触发。
* **命名空间**：替换匹配器和 `labels` 中的 `claude-runners`，并为每个其他命名空间复制该告警。如果您的目标没有 `namespace` 标签，请删除该匹配器和该标签，否则告警会读取到 0 并触发。
* **`ClaudeRunnerPollStale` 和 `ClaudeRunnerPollErrors`**：请保持从[示例告警规则](#sample-alert-rules)中加载它们，因为 `ClaudeRunnersBelowFloor` 仍会将无法轮询的运行器计入在内

#### 会话等待按需运行器时告警

当等待运行器的会话数保持在 0 以上并持续 10 分钟时，`ClaudeOrchestratorSessionsWaiting` 会触发。当过去 10 分钟内超过 1% 的队列等待超过 160 秒，并且该情况持续 5 分钟时，`ClaudeOrchestratorQueueWaitHigh` 会触发：

```yaml theme={null}
groups:
  # 仅适用于按需运行器。无需针对命名空间进行编辑。
  - name: claude-code-self-hosted-on-demand-capacity
    rules:
      - alert: ClaudeOrchestratorSessionsWaiting
        expr: |
          max by (namespace) (
            claude_code_self_hosted_orchestrator_pool_pending_sessions
            and on (instance)
            max_over_time(claude_code_self_hosted_orchestrator_connected[2m]) == 1
          ) > 0
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "Sessions waiting for a runner: {{ $value }}, above the threshold for 10 minutes. Check claude_code_self_hosted_orchestrator_spawn_hooks_total by result, then runner workloads that don't start."
      - alert: ClaudeOrchestratorQueueWaitHigh
        expr: |
          histogram_quantile(0.99, sum by (namespace, le) (
            rate(claude_code_self_hosted_orchestrator_session_queue_wait_seconds_bucket[10m])
          )) > 160
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "Over 1% of queue waits in the last 10 minutes exceeded 160 seconds. Check whether ClaudeOrchestratorDisconnected is firing, then claude_code_self_hosted_orchestrator_spawn_hooks_total by result and claude_code_self_hosted_orchestrator_spawn_hook_duration_seconds."
```

* **阈值**：如果始终有某个会话在等待，请将 `ClaudeOrchestratorSessionsWaiting` 中的 `> 0` 提高到您通常看到的数量以上
* **`ClaudeOrchestratorDisconnected`**：请保持从[示例告警规则](#sample-alert-rules)中加载它，因为在命名空间中最后一个编排器断开连接约 2 分钟后，`ClaudeOrchestratorSessionsWaiting` 就会停止触发

#### 编排器停机时告警

若要得知[编排器](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#on-demand-runners)本身已停机，请加载 `ClaudeOrchestratorAbsent`。当命名空间中没有任何编排器被抓取并持续 5 分钟时，它会触发：

```yaml theme={null}
groups:
  # 仅适用于按需运行器。请将 claude-runners 替换为编排器的命名空间。
  - name: claude-code-self-hosted-on-demand-orchestrator
    rules:
      - alert: ClaudeOrchestratorAbsent
        expr: absent(claude_code_self_hosted_orchestrator_connected{namespace="claude-runners"})
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "No orchestrator has been scraped for 5 minutes. If it's down, no on-demand runner starts, and sessions wait. Check the orchestrator workload first, then its scrape target if it's running."
```

* **命名空间**：将 `claude-runners` 替换为编排器的命名空间，并为每个运行编排器的其他命名空间复制该告警。如果您的目标没有 `namespace` 标签，请删除该匹配器，否则告警会在 5 分钟后触发并一直保持。
* **不抓取编排器的 Prometheus 服务器**：请不要将该告警放入它们的规则文件中，因为在这类服务器上它会在 5 分钟后触发并一直保持
* **不要依赖编排器的其他告警**：它们读取的是编排器自身提供的指标。在编排器停机后约 10 分钟内，`ClaudeOrchestratorSessionsWaiting`、`ClaudeOrchestratorQueueWaitHigh` 以及编排器的[示例告警规则](#sample-alert-rules)都会停止触发。

### 透传会话子进程指标

每个会话都在其自己的子进程中运行，并拥有自己的 OpenTelemetry 指标；当 `--capacity` 大于 1 时，运行器会改写这些子进程指标的暴露方式。在运行器主机上设置 `OTEL_METRICS_EXPORTER=prometheus`，并在会话的环境中设置 `CLAUDE_CODE_ENABLE_TELEMETRY=1`（例如通过您的[包装脚本](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#wrapper-scripts)或运行器自身的环境，会话会继承该环境），即可在运行器自身的 `/metrics` 端点上与运行器的序列一起重新暴露每个子进程的计数器和仪表类指标。运行器会改写子进程的导出器，使其通过 OTLP 推送到健康端口上仅限环回的接收器，为每个序列添加 `session_id` 和 `client_platform` 标签，并在会话结束时清除该会话的序列。直方图不会透传，名称会与运行器自身前缀冲突的子进程指标将被丢弃。

在默认的 `--capacity 1` 下，不会进行这种改写：会话的子进程照常在端口 9464 上绑定自己的 Prometheus 端点。

#### 将会话指标与运行器指标关联

若要查看空闲会话产生的成本，请在 `ccr_session_id` 上将运行器的 `claude_code_self_hosted_runner_session_idle_seconds` 与 Claude Code 的成本指标关联。运行器的指标不带该标签，因此请使用 `label_replace` 为其添加一个，如以下查询所示：

```text theme={null}
sum by (ccr_session_id) (claude_code_cost_usage_total)
and on (ccr_session_id)
label_replace(
  claude_code_self_hosted_runner_session_idle_seconds > 600,
  "ccr_session_id", "cse_$1", "session_id", "session_(.*)"
)
```

该查询为每个空闲超过十分钟的会话返回一个结果：其 Claude Code 进程迄今为止统计的以美元计的成本。若要关联其他 Claude Code 指标，请替换查询中的 `claude_code_cost_usage_total`。该查询适用于任何 `--capacity`。

* **`session_id`**：在端口 9464 上，Claude Code 指标中的该标签保存的是 Claude Code 自己的 [`session.id`](https://code.claude.com/docs/zh-CN/monitoring-usage#standard-attributes)，它不与任何运行器序列匹配。基于它进行关联不会返回任何结果，因此请基于 `ccr_session_id` 进行关联。
* **Claude Code 指标没有 `ccr_session_id`**：请检查 [`OTEL_METRICS_INCLUDE_SESSION_ID`](https://code.claude.com/docs/zh-CN/monitoring-usage#metrics-cardinality-control) 是否未设置为 `false`

### 会话生命周期计数器语义

`sessions_started_total`、`sessions_completed_total`、`sessions_failed_total` 和 `sessions_interrupted_total` 计数器根据每个会话的结束方式对其进行分类。每个生成的会话子进程都会在生成时递增 `sessions_started_total`，并在退出时恰好递增其他三个计数器中的一个，因此 `sessions_started_total` 减去其他三个计数器之和即等于当前正在运行的会话子进程数。

* `completed`：会话正常结束。这包括子进程以代码 `0` 自行退出、会话在子进程仍处于连接状态时被归档或删除，以及运行器正常交还槽位：在空闲超时、退役时间或 `--kill-session-after-min` 限制时释放会话；启动超时；或轮询循环在子进程退出前注意到服务器端取消分配。递增 `sessions_completed_total`。
* `failed`：子进程以非零代码自行退出，可能是崩溃，也可能是生成后的设置失败。递增 `sessions_failed_total`。
* `interrupted`：运行器出于既非会话成功也非运行器故障的运维原因终止了子进程，例如排空，或者在会话的 `--kill-session-after-min` 限制之后的 [`SELF_HOSTED_RUNNER_MAX_LIFETIME_GRACE_MS`](#environment-variable-only-settings) 宽限期结束时，终止仍在运行器上的会话。Kubernetes 滚动重启发送 `SIGTERM` 就是排空的一个例子。递增 `sessions_interrupted_total`。

在 v2.1.260 之前，运行器会终止每个达到其 `--kill-session-after-min` 限制的会话，并将其计入 `sessions_interrupted_total`。

[`post-session` hook](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration#post-session) 的 `CLAUDE_RUNNER_EXIT_REASON` 对正常交接的分类方式不同。该 hook 将以下情况报告为 `interrupted`，因为是运行器停止了子进程：释放、启动超时、服务器取消分配，以及轮询先注意到的归档或删除。这些计数器则将相同的事件记录为 `completed`，因为槽位已被正常交还。

如果您直接将 hook 回执与 `sessions_completed_total` 进行核对，会少计完成数。请使用 hook 获取针对每个会话的保证，使用计数器获取聚合速率。

在一次性环境中，即 `--capacity 1` 并使用默认的 `--drain-grace-sec 0`，每个运行器进程会在其唯一的会话结束后片刻退出。`sessions_completed_total`、`sessions_failed_total` 和 `sessions_interrupted_total` 仅在会话结束时（即该退出之前）递增，因此每 15 到 60 秒一次的 Prometheus 抓取很少能在运行器的序列消失之前捕获到该递增；这三个会话结束计数器就是本节其余部分所说的终结计数器。`sessions_started_total` 在生成时递增，并在会话的整个生命周期内保持可见，因此能可靠地出现，但在一次性环境中，它的含义更接近"当前正在运行的会话"，而不是累计计数。

请针对相应的目标使用此表中的序列，而不是终结计数器：

| 目标 | 使用 |
| :- | :- |
| 吞吐量 | `claude_code_self_hosted_orchestrator_spawn_hooks_total{result="ok"}`，这是长期运行的编排器上的计数器，每次 `spawn-runner` hook 成功时递增一次，在 `rate()` 下仍保持有意义。它统计的是 hook 调用次数而不是会话数，因此预热和为同一会话重复生成会使其与会话计数产生偏差。 |
| 利用率 | `sum(claude_code_self_hosted_runner_active_sessions)` 与 `sum(claude_code_self_hosted_runner_capacity)` 之比，两者都是仪表，无论运行器生命周期如何，在每次抓取时都有效 |
| 积压 | `claude_code_self_hosted_orchestrator_pool_pending_sessions` 用于队列深度，以及 `claude_code_self_hosted_orchestrator_queue_circuit_broken_sessions`，高于零时告警 |
| 失败 | `claude_code_self_hosted_runner_sessions_failed_total`，尽力而为：生成后的真实崩溃确实会使其递增，并且在 `--drain-grace-sec` 大于 `0`、运行器比其会话存活更久的情况下，`rate()` 是有意义的。一次性环境存在与其他终结计数器相同的抓取窗口问题，因此您看到的任何非零值都值得调查。生成之前的失败，例如 checkout hook 失败、git 准备或令牌问题，仅出现在 `session_init_errors_total` 中。 |

`orchestrator_*` 行仅存在于运行[按需编排器](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#on-demand-runners)的环境中。在运行器比其会话存活更久的固定机群上（`--drain-grace-sec` 大于 `0`），请使用 `sum(rate(claude_code_self_hosted_runner_sessions_started_total[5m]))` 衡量吞吐量；在一次性机群上，该序列存在与终结计数器相同的抓取窗口问题，因此请改为依赖排队会话计数。请在 [**Cloud environments** 管理页面](https://claude.ai/admin-settings/cloud-environments)上该环境的 **Activity** 选项卡中检查积压情况：运行器不导出队列深度序列。

如需按会话报告结果，请改用 [`post-session` hook](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration#post-session)：根据 [hook 自身的约定](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md#post-session)，除了运行器被突然终止（例如 VM 抢占）之外，它会在每个生成了子进程的会话结束时触发。

## 接下来

* [自托管环境](https://code.claude.com/docs/zh-CN/self-hosted-environments)：环境、运行器和会话模型；[快速入门](https://code.claude.com/docs/zh-CN/self-hosted-environments-quickstart.md)和[部署到生产](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md)包含设置和操作
* [自定义会话](https://code.claude.com/docs/zh-CN/self-hosted-environments-configuration.md)：包装脚本、生命周期钩子和按需运行器
* [验证会话身份](https://code.claude.com/docs/zh-CN/self-hosted-environments-identity.md)：会话令牌、其声明以及如何验证它
