> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# PR Steward 故障排除

> 修复 PR Steward 在 GitHub Pull Request 上未启动、已停止或已暂停的问题，并解决 Owner 在设置过程中看到的消息

当 [PR Steward](https://code.claude.com/docs/zh-CN/pr-steward.md) 在 Pull Request 上的行为不符合您的预期时，请先查找它发布的评论。如果 PR Steward 发布了评论，请[对照该评论](#pr-steward-posted-a-comment-that-says-why)。如果它没有在 Pull Request 上留下任何内容，请检查[它未启动的原因](#pr-steward-didn’t-start-on-a-pull-request)。

## PR Steward 未在 Pull Request 上启动

某些原因不会在 Pull Request 上留下任何内容，因此请逐一检查以下各节。

### PR Steward 未响应您的评论

如果您在 github.com 上评论了 `@claude steward`，但 PR Steward 没有启动，请查看您的评论：

| 您看到的情况 | 检查内容 |
| :- | :- |
| 没有 👀 回应 | 确认这是一条新评论而非编辑过的评论，并且您拥有写入权限 |
| 有 👀 回应，但之后没有出现 `claude-pr-steward-watching` 标签 | 确认该命令位于顶层评论的开头。然后检查 [PR Steward 未在 Pull Request 上启动](#pr-steward-didn’t-start-on-a-pull-request)下的其他各节 |

### PR Steward 不会启动的 Pull Request

检查您的 Pull Request 是否属于 PR Steward 会跳过的类型：

| 原因 | 处理方法 |
| :- | :- |
| 该 Pull Request 是草稿 | 将其标记为可供审查。如需在草稿上工作，请参阅[**Include my draft PRs**](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#choose-what-pr-steward-works-on) |
| 该 Pull Request 来自 fork、由 bot 创建，或其作者没有该仓库的写入权限 | PR Steward 不会在这些 Pull Request 上启动 |
| 描述中包含 `<!-- claude-pr-steward-disable -->` | 从描述中删除该标记，然后评论 `@claude steward` |
| 另一个 Claude 会话已在监视该 Pull Request，例如启用了 [auto-fix](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md#auto-fix-pull-requests) 的云端会话 | 在该会话中关闭 auto-fix，然后再次评论 `@claude steward` |

### 某项设置阻止 PR Steward 启动

某项设置可能会阻止 PR Steward 启动：

| 原因 | 处理方法 |
| :- | :- |
| 该仓库[设置为 **Manual**](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#choose-when-pr-steward-starts-in-each-repository)，或[您自己的选项](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#turn-off-or-limit-pr-steward-on-all-your-pull-requests)为 **Off** 或 **Only when tagged** | 评论 `@claude steward` |
| 该 Pull Request 属于某位团队成员，且其本人的选项为 **Off** | 请其评论 `@claude steward` |
| 已针对该仓库或作者的 GitHub 账户关闭 PR Steward | 请您的 Claude 组织的 [Owner](https://code.claude.com/docs/zh-CN/server-managed-settings.md#access-control) [添加该仓库](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#turn-on-pr-steward-and-add-repositories)，或更改 [PR Steward 为哪些作者工作](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#limit-which-authors-pr-steward-works-for) |

要确认最后一种原因，请让作者打开其 [PR Steward 设置页面](https://claude.ai/settings/claude-code#pr-steward)。当您的组织关闭了 PR Steward 时，该页面会显示一条横幅。当作者在该页面关联的 GitHub 账户被排除时，页面也会显示横幅。

### 短时间内打开了大量 Pull Request

如果 PR Steward 在短时间内于您组织中的大量 Pull Request 上启动，请评论 `@claude steward`。如果 `claude-pr-steward-watching` 标签仍未出现，请稍后再次评论。

## PR Steward 发布了说明原因的评论

在以下表格之一中查找 PR Steward 评论中的文字。

### PR Steward 无法启动时的评论

如果 PR Steward 发布了评论但未启动，请在此处查找该评论的文字：

| 评论内容 | 处理方法 |
| :- | :- |
| `PR Steward didn't start: only someone with write access` | 请拥有写入权限的人添加 `claude-pr-steward-watching` 标签或评论 `@claude steward` |
| `PR Steward hasn't started: your organization has no extra usage available` | 请 Owner 在 [**Organization settings > Usage**](https://claude.ai/admin-settings/usage) 中添加使用额度或提高支出限额。如果是您请求 PR Steward 启动的，那么再次请求时它就会启动。否则，它将在下一次推送时启动 |
| `PR Steward hasn't started: billing for PR Steward isn't set up` | 请 Owner 选择一个已启用 PR Steward 的仓库，并将其 [**Steward** 设置更改](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#choose-when-pr-steward-starts-in-each-repository)为 **Off** 以外的其他选项。之后可以立即改回 |
| `Claude PR Steward couldn't start, so it isn't watching this pull request` | 添加标签或评论 `@claude steward` 以重试 |
| `Claude PR Steward couldn't start on this pull request because the setup script` | 请能够编辑 [PR Steward 运行所在环境](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#choose-the-environment-pr-steward-runs-in)的人修复其设置脚本，然后再次启动 PR Steward |
| `this repository has PR Steward enabled in more than one Claude workspace` | 请各 Owner 仅在一个 Claude 组织中保留该仓库，并将其从其他组织中移除 |

### PR Steward 停止或暂停时的评论

如果 PR Steward 发布评论后停止或暂停，请在此处查找该评论的开头文字：

| 评论内容 | 含义 | 处理方法 |
| :- | :- | :- |
| `PR Steward is paused on this pull request` | 如评论所述，PR Steward 工作期间使用额度已用尽，或计费设置尚未完成 | 请 Owner 添加使用额度或提高支出限额，或按照评论中的说明完成计费设置。PR Steward 将在下一次推送或评论时继续工作 |
| `Claude PR Steward stopped watching this pull request: this session's spend reached the repository's PR Steward budget` | 本次会话已达到为该仓库设置的[支出上限](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#set-pr-steward’s-spend-limits) | 自行完成剩余工作，或再次添加 `claude-pr-steward-watching` 标签以启动新会话 |
| `Claude PR Steward stopped watching this pull request because your organization's PR Steward settings no longer cover it` | Owner 关闭了 PR Steward，或移除了该仓库或作者 | 在设置重新覆盖该 Pull Request 后，添加标签或评论 `@claude steward` |
| `Claude PR Steward stopped watching this pull request because for <N> days no person has commented or reviewed here` | 在评论给出的天数内，没有任何人发表评论或进行审查，并且除 PR Steward 外没有人推送过 | 添加标签或评论 `@claude steward` |

## Auto-fix 不可用

在 PR Steward 为您的 Pull Request 工作的仓库中，云端会话上的 **Auto-fix** 开关不可用。如果您请求对 PR Steward 正在监视的 Pull Request 进行 auto-fix，该请求将失败。在 claude.ai/code 上，显示的消息为 `PR Steward is already watching this PR.` 要在该仓库中使用 [auto-fix](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md#auto-fix-pull-requests)，请为您自己关闭 PR Steward：

* **之后的 Pull Request**：将[您自己的 PR Steward 选项](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#turn-off-or-limit-pr-steward-on-all-your-pull-requests)设置为 **Off**，这将覆盖您在所有仓库中的 Pull Request
* **PR Steward 已在监视的 Pull Request**：还需[在该处停止 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#start-or-stop-pr-steward-on-one-pull-request)

## 设置 PR Steward 时的消息

在您[设置 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-setup.md) 时，可能会在 [**Claude Ship**](https://claude.ai/admin-settings/claude-ship) 页面上看到以下消息：

| 您看到的情况 | 处理方法 |
| :- | :- |
| **The GitHub connector is not enabled for your organization.** | 在 [**Organization settings > Connectors**](https://claude.ai/admin-settings/connectors) 中启用该连接器 |
| 点击 **Add** 时，仓库列表中缺少某个仓库 | 前往 [**Organization settings > Git providers**](https://claude.ai/admin-settings/source-control)，在您的 GitHub 组织上安装 [Claude GitHub App](https://github.com/apps/claude)，并包含该仓库 |
| 一条消息提示您的 GitHub 账户“cannot push to that repository”，或添加该仓库“needs write access” | 关联一个可以推送的账户，或请一位可以推送的 Owner 来操作 |
| `PR Steward can't be enabled right now: billing setup is temporarily unavailable` | 几分钟后重试 |
| **PR Steward isn’t available for your organization.** | 检查[要求](https://code.claude.com/docs/zh-CN/pr-steward.md#availability)。如果 [**Runs for**](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#turn-on-pr-steward-and-add-repositories) 为 **Listed users only**，请添加一个用户名 |
| **PR Steward is on but not watching any repositories yet.** | [为仓库启用 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-setup.md#turn-on-pr-steward-and-add-repositories) |

## 相关资源

* [在 Pull Request 上使用 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md)
* [设置 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-setup.md)
* [PR Steward](https://code.claude.com/docs/zh-CN/pr-steward.md)
