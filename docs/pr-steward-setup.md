> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 设置 PR Steward

> 为您的 Claude 组织启用 PR Steward：要求审查、添加仓库、设置支出限额，并选择它何时启动以及为哪些作者工作

要设置 [PR Steward](https://code.claude.com/docs/zh-CN/pr-steward.md)，请先为您的 Claude 组织启用它，然后为每个仓库启用它。您需要拥有 [Owner 角色](https://code.claude.com/docs/zh-CN/server-managed-settings.md#access-control)。

在某个仓库中启用后，PR Steward 会向该仓库中的 Pull Request 推送提交以处理审查反馈，并尝试修复失败的检查和解决合并冲突。它会自行启动或在有人请求时启动，[具体由您为每个仓库选择](#choose-when-pr-steward-starts-in-each-repository)。

## 前提条件

请确认您的组织[符合使用 PR Steward 的条件](https://code.claude.com/docs/zh-CN/pr-steward.md#availability)。

在您于 [claude.ai](https://claude.ai/admin-settings) 管理的 Claude 组织中，您需要：

* [Owner 角色](https://code.claude.com/docs/zh-CN/server-managed-settings.md#access-control)
* [使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans)，用于支付 PR Steward 的工作费用
* GitHub 连接器，需在 [**Organization settings > Connectors**](https://claude.ai/admin-settings/connectors) 中启用。启用后，您组织中的成员可以将其 [GitHub 账户连接](https://code.claude.com/docs/zh-CN/web-quickstart.md#connect-github)到 Claude

在 GitHub 上，您需要具备以下权限，或由具备这些权限的人员操作：

* GitHub 组织的 Owner，用于安装 [Claude GitHub App](https://github.com/apps/claude) 并[将其关联到您的 Claude 组织](https://code.claude.com/docs/zh-CN/admin-setup.md#connected-github-accounts)
* 仓库的管理员，用于添加分支规则。私有仓库还需要 [GitHub 付费计划](https://docs.github.com/en/get-started/learning-about-github/githubs-plans)
* 您连接到 Claude 的 GitHub 账户对该仓库拥有推送权限

## 使用分支规则要求审查

在启用 PR Steward 之前，请先在 GitHub 中添加以下规则。

**1. 要求一次审批**

在默认分支上添加一条规则集规则，要求通过 Pull Request 进行更改，且至少获得一次审批。

**2. 推送后要求重新审批**

在该规则中，设置在推送新提交时撤销过时的审批，或要求对最近一次推送进行审批。

**3. 检查规则集的绕过列表**

不要将 Claude GitHub App 加入规则集的绕过列表。在 GitHub Enterprise Server 上，这指的是[您的组织创建的应用](https://code.claude.com/docs/zh-CN/github-enterprise-server.md#admin-setup)。如果 PR Steward 在[自托管环境](#run-pr-steward-in-a-self-hosted-environment)中运行，也不要将运行器的 git 身份加入该列表。

**4. 保护其他分支**

为 PR Steward 不应推送的任何其他分支添加规则，并在您不希望出现强制推送的地方阻止强制推送。

> [!NOTE]
>
> 在 [Claude Approvals 策略允许 Claude 批准](https://code.claude.com/docs/zh-CN/approvals-github-rules.md#merges-without-human-review)的情况下，Claude 的审查可以计为第 1 步中的审批。来自[自托管环境](#run-pr-steward-in-a-self-hosted-environment)中会话的审批也可以计入。

## 启用 PR Steward 并添加仓库

从 [**Organization settings > Claude Ship**](https://claude.ai/admin-settings/claude-ship) 开始。

该页面上的 **Repositories** 表列出了每个仓库及其 **Steward** 设置：

![Claude Ship 页面上的 Repositories 表，包含一个 Add 按钮和三个仓库，其 Steward 设置分别为 Manual、Off 和 On。每一行都有一个 Edit 按钮。](https://mintcdn.com/claude-code/cMjxXVBUSLLh9iVu/images/pr-steward-setup-repositories.png?fit=max\&auto=format\&n=cMjxXVBUSLLh9iVu\&q=85\&s=a76bbd05275c5a96053cbbe7e6f1a547)

> [!WARNING]
>
> 添加仓库可能会在该仓库中启动 [Code Review](https://code.claude.com/docs/zh-CN/code-review-setup.md)，这会产生费用。
>
> * **尚未列出任何仓库**：会弹出一个对话框，询问 Claude 在您添加的仓库上执行什么操作，默认选中 **Code review**。如果您在对话框中选择 **Steward**，PR Steward 会在您设置支出限额之前就为组织和这些仓库启用。
> * **已列出其他仓库**：不会弹出对话框，新仓库默认设置为在每次推送时进行审查。
>
> 要更改此行为，请为该仓库[设置审查的运行时机](https://code.claude.com/docs/zh-CN/code-review-setup.md#set-when-reviews-run)或[关闭 Code Review](https://code.claude.com/docs/zh-CN/code-review-setup.md#turn-off-code-review-or-remove-a-repository)。

**1. 添加仓库**

如果该仓库已列在 **Repositories** 下，请跳过此步骤。否则，点击 **Add**，在出现提示时连接您的 GitHub 账户，然后选择该仓库。如果随后弹出对话框询问 Claude 在其上执行什么操作，请点击 **Not now**，这样仓库会保留在列表中。

**2. 设置 PR Steward 为哪些作者工作**

在 **Steward** 开关下方，使用 **Runs for** 设置 PR Steward 处理哪些人的 Pull Request。选择 **All eligible authors**，或者选择 **Listed users only** 以先在少数人中试点，并输入他们的 GitHub 用户名。

在 GitHub Enterprise Server 上，请选择 **All eligible authors**，因为在那里使用列表时 PR Steward 不会启动。

**3. 为组织启用 PR Steward**

点击 **Steward** 开关，然后点击 **Turn on**。

**4. 设置 PR Steward 的每月限额**

前往 [**Organization settings > Usage**](https://claude.ai/admin-settings/usage)。在 **Services** 下，在 **PR Steward** 行中设置限额。在您设置之前，PR Steward 没有自己的支出限额。

**5. 为仓库启用 PR Steward**

在 **Claude Ship** 页面上，点击该仓库旁边的 **Edit**。将 **Steward** 设置为 [**Manual (@claude steward)**](#choose-when-pr-steward-starts-in-each-repository)。点击 **Steward spend cap** 旁边的 **Change**，输入以美元计的上限。请参阅[上限的工作方式](#set-pr-steward’s-spend-limits)。

**6. 测试 PR Steward**

使用一个拥有写入权限且在 **Runs for** 覆盖范围内的账户打开一个 Pull Request。评论 `@claude steward`。随后会出现 `claude-pr-steward-watching` 标签和一条标题为 **Claude PR Steward** 的评论。

如果它们没有出现，请参阅 [PR Steward 故障排除](https://code.claude.com/docs/zh-CN/pr-steward-troubleshooting.md)。

## 将 PR Steward 推广到更多 Pull Request

当 PR Steward 的修复与您的审查者会做的一致时，将该仓库设置为 **On**。对于之后添加的每个仓库，请先在其中[要求审查](#require-reviews-with-branch-rules)。然后重复[启用 PR Steward 并添加仓库](#turn-on-pr-steward-and-add-repositories)中的以下步骤：添加仓库、为其启用 PR Steward，并测试 PR Steward。PR Steward 最多可在 256 个仓库中启用。

### 选择 PR Steward 在每个仓库中的启动时机

在 **Claude Ship** 页面上，点击某个仓库旁边的 **Edit**，然后打开 **Steward** 菜单：

![仓库 Edit 对话框中的 Steward 菜单，列出了 Off、On、On (including drafts) 和 Manual (@claude steward)，其中 Manual (@claude steward) 已被选中。](https://mintcdn.com/claude-code/cMjxXVBUSLLh9iVu/images/pr-steward-setup-edit-options-light.png?fit=max\&auto=format\&n=cMjxXVBUSLLh9iVu\&q=85\&s=6dd7aaf84aa00b6893b9e7533b1089d7)

选择您希望 PR Steward 何时启动：

| 选项 | PR Steward 的启动时机 |
| :- | :- |
| **Manual (@claude steward)** | 当拥有写入权限的人员评论 `@claude steward` 或添加 `claude-pr-steward-watching` 标签时 |
| **On** | 当一个[不会被 PR Steward 跳过](https://code.claude.com/docs/zh-CN/pr-steward.md#when-pr-steward-starts)的 Pull Request 被打开、重新打开或标记为可供审查时。此外，对于它此前尚未启动过的已打开 Pull Request，会在下一次推送时启动 |
| **On (including drafts)** | 与 **On** 相同，并且也会在草稿 Pull Request 上启动 |
| **Off** | PR Steward 不会在该仓库中启动 |

在 **Manual** 仓库中，评论或标签可以在[草稿](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#choose-what-pr-steward-works-on)上启动 PR Steward。对于 [PR Steward 会跳过](https://code.claude.com/docs/zh-CN/pr-steward.md#when-pr-steward-starts)的其他 Pull Request，它们不会启动 PR Steward。

对于选择了 [**Act on its own**](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#turn-off-or-limit-pr-steward-on-all-your-pull-requests) 的开发者，PR Steward 也会在该仓库中的其 Pull Request 上启动。

### 限制 PR Steward 为哪些作者工作

要跳过某位作者，请将其 GitHub 用户名添加到 **Steward** 开关下方的 **Excluded users** 中。

## 控制 PR Steward 的支出

[在关闭使用额度时计费仍会继续](https://code.claude.com/docs/zh-CN/claude-ship.md#how-each-claude-ship-agent-is-billed)。

### 查看 PR Steward 的支出

要查看某个仓库在 PR Steward 上的预估支出，请打开该仓库的 **Edit** 对话框。要查看您组织过去 30 天的预估总支出，请前往 [claude.ai/analytics/pr-steward](https://claude.ai/analytics/pr-steward)。

### 设置 PR Steward 的支出限额

在 [**Organization settings > Usage**](https://claude.ai/admin-settings/usage) 的 **Services** 下的 **PR Steward** 行中更改每月限额。该行在您首次启用 PR Steward 后出现。在仓库的 **Edit** 对话框中更改支出上限。当 PR Steward 达到某个限额时，请查看它在 Pull Request 上发布的评论：

* **其每月限额，或您组织的使用额度限额**：PR Steward 暂停。一旦使用额度可以再次支付，它会在下一次推送或评论时恢复工作
* **支出上限**：PR Steward 在该 Pull Request 上停止。在停止之前，支出可能会超过上限。拥有写入权限的人员可以再次启动它，并获得完整的上限。当 PR Steward 在某个 Pull Request 上工作超过大约一周时，它可以启动一个新的[云端会话](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md)，此时完整的上限会再次适用

当没有剩余可用的使用额度时，PR Steward 不会启动，**Claude Ship** 页面会显示 **Steward cannot start on pull requests**。请在 **Usage** 页面上添加使用额度或提高限额。

## 设置 PR Steward 的构建环境

当 PR Steward 需要更多条件来构建和测试您的代码时，请为其选择一个环境。

### 选择 PR Steward 运行的环境

要添加设置脚本、环境变量或不同的网络访问权限，请使用[组织共享环境](https://code.claude.com/docs/zh-CN/cloud-environments.md#organization-shared-environments)。在 **Claude Ship** 页面上，在 **Where it runs** 下选择一个 **Default environment**。要为某个仓库覆盖该设置，请在 **Edit** 对话框中点击 **Environment** 旁边的 **Change**。

如果两者都未设置，PR Steward 可以使用您组织的[默认环境](https://code.claude.com/docs/zh-CN/cloud-environments.md#the-default-environment)。如果也没有默认环境，它将在一个由 Anthropic 托管、没有设置脚本也没有环境变量的环境中运行。

### 在自托管环境中运行 PR Steward

对于 PR Steward 运行的环境，您可以选择[自托管环境](https://code.claude.com/docs/zh-CN/self-hosted-environments)。[审批和合并方面的限制](https://code.claude.com/docs/zh-CN/pr-steward-security.md#limits-on-approvals-and-merges)仅针对 Anthropic 托管的会话进行描述。在没有 [Anthropic 托管 git](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#use-the-anthropic-git-proxy) 的运行器上，PR Steward 使用运行器自己的 git 凭据进行推送。

> [!WARNING]
>
> 在自托管环境中，Claude 可以使用[会话能够读取的任何 GitHub 凭据](https://code.claude.com/docs/zh-CN/self-hosted-environments-deploy.md#harden-your-deployment)直接访问 GitHub。Anthropic 的服务无法拒绝以这种方式做出的审批。GitHub 可能会将其计入您要求的审批数量。

在自托管环境中，PR Steward 不会从运行器或运行器检出的 Pull Request 中加载 `CLAUDE.md` 或规则文件。它也不会加载[托管的 `CLAUDE.md`](https://code.claude.com/docs/zh-CN/memory#deploy-organization-wide-claude-md)。请将 PR Steward 应遵循的内容放在[仓库的默认分支上](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md#save-pr-steward-instructions-for-everyone-in-a-repository)。

### 通过 Claude Tag 捆绑包为 PR Steward 提供凭据

要为 PR Steward 提供凭据（例如私有包注册表的登录信息），请将其保存在 [Claude Tag 捆绑包](https://claude.com/docs/claude-tag/concepts/glossary#bundle)中，并为 PR Steward 启用该捆绑包。当 PR Steward 在 Anthropic 托管的环境中运行时，即在除[自托管环境](#run-pr-steward-in-a-self-hosted-environment)以外的任何环境中运行时，它都可以使用捆绑包。

要将凭据放入捆绑包，请[设置 Claude Tag](https://claude.com/docs/claude-tag/admins/setup-overview) 并[创建捆绑包](https://claude.com/docs/claude-tag/admins/add-connections#create-a-bundle)。

> [!WARNING]
>
> PR Steward 在无人值守的情况下工作，并且会[读取其他人编写的 Pull Request 文本](https://code.claude.com/docs/zh-CN/pr-steward-security.md#pr-steward’s-session-environment)，这些文本可能会影响它使用凭据的方式。请[限制每个凭据](https://claude.com/docs/claude-tag/admins/add-connections#limit-access-to-specific-resources)，使其仅限于构建和测试您的代码所需的主机和权限。

要为 PR Steward 启用该捆绑包：

**1. 打开捆绑包**

前往 [**Organization settings > Claude Tag**](https://claude.ai/admin-settings/claude-tag)，选择 **Bundles** 选项卡，然后打开该捆绑包。

**2. 为该捆绑包启用 PR Steward**

在 **Where it applies** 下，打开 **PR Steward** 行中的开关，然后点击 **Save changes**。

随后，该捆绑包将在您的整个组织中对 PR Steward 启用，并且该开关适用于捆绑包中的每个凭据。

PR Steward 会话中的安装或测试可能会失败，并返回以如下文本开头的 403 错误：

```text wrap theme={null}
denied by policy: this host's bundle of logins and rules isn't turned on for PR Steward
```

如果 PR Steward 需要该主机来构建或测试您的代码，请为包含该主机凭据的捆绑包打开 **PR Steward** 开关。即使该捆绑包已打开 **Everywhere**，也可能需要这样做，因为这两个开关是相互独立的。如果该捆绑包包含的内容超出 PR Steward 所需，请[创建一个](https://claude.com/docs/claude-tag/admins/add-connections#create-a-bundle)仅包含该凭据的捆绑包，并为该捆绑包启用 PR Steward。

## 关闭 PR Steward

关闭 PR Steward 后，新的启动会停止。对于已带有 `claude-pr-steward-watching` 标签的 Pull Request，它会在一段时间内继续工作。当您为某位作者关闭它时，这段时间可能长达约一周。

如果也要在这些 Pull Request 上停止 PR Steward，请先从此更改所涵盖的 Pull Request 中移除该标签。

然后在 **Claude Ship** 页面上关闭 PR Steward：

| 要为以下对象关闭 PR Steward | 操作 |
| :- | :- |
| 组织 | 点击 **Steward** 开关 |
| 某个仓库 | 点击该仓库旁边的 **Edit**，并将 **Steward** 设置为 **Off** |
| 某位作者 | 将其 GitHub 用户名添加到 **Excluded users** |

如果您先关闭了它，请在之后移除标签以停止 PR Steward，除非您的 GitHub 组织关联到多个 Claude 组织。

## 后续步骤

* [使用 PR Steward](https://code.claude.com/docs/zh-CN/pr-steward-pull-requests.md)
* [PR Steward 可以访问的内容](https://code.claude.com/docs/zh-CN/pr-steward-security.md)
