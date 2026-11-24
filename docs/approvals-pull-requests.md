> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 在 Pull Request 上使用 Claude Approvals

> 了解 Claude Approvals 检查标题的含义及后续操作，使用评论命令，并处理来自 Claude 和来自 fork 的 Pull Request

在启用了 [Claude Approvals](https://code.claude.com/docs/zh-CN/approvals.md) 的地方，它会向其评估的 Pull Request 添加一个名为 **Claude Approvals** 的检查。该检查依据仓库中的[策略文件](https://code.claude.com/docs/zh-CN/approvals-policy.md)，报告还有谁需要批准该 Pull Request。如果您的仓库将该检查设为必需，GitHub 可以在检查通过之前阻止合并。如果您的 Pull Request 没有该检查（例如因为它是草稿），请参阅[未出现 Claude Approvals 检查](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md#no-claude-approvals-check-appeared)。

## 让您的 Pull Request 通过 Claude Approvals

Approvals 会在推送或批准等事件发生后评估您的 Pull Request。它在 Pull Request 上的两个位置报告结果：

* **Claude Approvals 检查**：位于 **Conversation** 标签页底部的检查列表中。其名称旁边的文本称为标题，给出审批状态，例如 **Waiting on human review from any person (1/2)**
* **Claude 的评论**：位于对话中，说明得出该结果的原因。结果变化时，Approvals 会编辑同一条评论，因此它始终保留在首次出现的位置

要让检查通过：

1. **阅读审批状态**：检查的标题说明了您的 Pull Request 当前所处的状态，例如还有谁需要批准
2. **按照状态的要求操作**：[查找该标题](#find-your-approval-status-and-what-to-do-next)以了解需要做什么。例如，请具有写入权限的人批准、修复 Code Review 发布的问题，或等待审查完成
3. **再次阅读状态**：标题可能会在下一次评估时发生变化，例如有人批准或您推送时。在一次更改了 diff 的推送之后，[之前的批准可能不再计入](https://code.claude.com/docs/zh-CN/approvals-policy.md#earlier-approvals-after-a-new-push)
4. **合并**：检查通过后，按照仓库的 GitHub 规则允许的方式合并。在策略允许的情况下，Claude 也可以发布一条批准审查

## 查找您的审批状态及后续操作

**Claude Approvals** 检查在其标题（即检查列表中其名称旁边的文本）中给出 Pull Request 的审批状态。在下列表格中查找您的检查标题，以了解其含义及后续操作。

如果标题不在此处，请打开 **Claude Approvals** 检查的摘要，其中给出了原因和下一步操作。要找到策略文件 `.claude/approvals.yml` 的维护者，请查看默认分支上最后修改该文件的人。

### Approvals 等待人员操作

出现以下标题时，Approvals 正在等待某个人。在观察模式之外，GitHub 会将 **Claude Approvals** 检查列在待处理检查中，并标记为 **Action required**。

| 标题 | 操作 |
| :- | :- |
| **Waiting on human review from any person (1/2)** | 请具有写入权限的人批准。数字分别表示目前已计入的批准数和策略要求的批准数，并且作者通常已经[计为其中一个](https://code.claude.com/docs/zh-CN/approvals-policy.md#whether-a-pull-request’s-author-counts)。标题中 "any person" 的位置可能会改为某条规则要求的人员或团队名称，也可能不指明任何人 |
| **Changes requested** | 让请求更改的审查者批准，或让某人驳回其审查 |
| **Claude's approval still stands; dismiss it** | [驳回 Claude 的审查](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/dismissing-a-pull-request-review)，或请有权在该分支上驳回审查的人操作。GitHub 未允许 Approvals 驳回该审查 |

### Approvals 等待 Code Review 或分类器

出现以下标题时，Approvals 正在等待 [Code Review](https://code.claude.com/docs/zh-CN/code-review-pull-requests.md) 或[分类器](https://code.claude.com/docs/zh-CN/approvals-classifiers.md)的回复，或者未得到回复：

| 标题 | 操作 |
| :- | :- |
| **Waiting for Code Review**、**Waiting for repo/change\_risk**、**Waiting for classifiers** | 暂时无需操作 |
| **A classifier had no result**、**A classifier has not run**，或以某个名称代替 "A classifier"，如 **repo/change\_risk has not run** | [重新运行 Claude Approvals 检查](#comment-commands-for-approvals)。请参阅[运行未给出回复时的情况](https://code.claude.com/docs/zh-CN/approvals-classifiers.md#what-a-rule-sees-when-a-classifier-run-gives-no-answer) |

### 规则或策略文件阻止 Approvals 通过

出现以下标题时，表示有规则阻止该 Pull Request、没有规则适用于某个已更改的文件，或某个策略文件无法使用：

| 标题 | 操作 |
| :- | :- |
| **Held by Code Review's findings** | 修复 Code Review 在 Pull Request 上发布的问题，然后推送。如果策略的 [`findings`](https://code.claude.com/docs/zh-CN/approvals-policy#findings) 列表中包含您，您可以解决这些问题的讨论串。如果解决后 **Claude Approvals** 检查仍未通过，请推送修复 |
| **Held by** 加上引号中的规则用途、**Held by a policy rule** | 询问策略文件的维护者，检查摘要中的规则需要什么，然后推送修复 |
| **No rule applies to** 加上引号中的文件名、**No rule applies to a changed file** | 请策略文件的维护者添加一条不带 `if` 且覆盖该文件的规则。该规则合并后，重新运行检查 |
| **This change breaks a policy file**、**Default branch policy file cannot be used** | [修复策略文件](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md#fix-a-policy-file-that-approvals-refuses) |

在您推送修复后，来自较早提交的失败 Code Review 仍可能阻止该检查。如果没有开始新的审查，请发布一条仅包含 [`@claude review`](https://code.claude.com/docs/zh-CN/code-review-pull-requests#comment-commands) 的新评论，这需要写入权限。

### Approvals 已通过或未运行

出现以下标题时，Approvals 已通过或未评估该 Pull Request：

| 标题 | 操作 |
| :- | :- |
| **Passed; no approval from Claude** | 如果 GitHub 要求批准，请找人批准 |
| **Nothing ran, not blocking: no file is covered** | 无需操作。如果应有人审查该 Pull Request 更改的文件，请让[策略文件](https://code.claude.com/docs/zh-CN/approvals-policy.md)的维护者覆盖这些文件 |
| **Observe mode ended: not evaluated yet** | 推送一个提交或[重新运行 Claude Approvals 检查](#comment-commands-for-approvals)。如果检查通过，启用了自动合并或位于合并队列中的 Pull Request 可能会立即合并 |

在 **Passed; no approval from Claude** 状态下，Claude 在以下等情况中不会发布批准：

* 某条规则设置为 `claude: comments`
* 某个已更改的文件没有规则，或者是[符号链接、子模块指针或 `.gitattributes` 文件](https://code.claude.com/docs/zh-CN/approvals-policy#protected-files)
* Pull Request 由 `claude[bot]` 打开，除非已安装并关联 [Claude Approvals GitHub App](#pull-requests-that-claude-opens)

## 观察模式下的 Approvals

在观察模式下，**Claude Approvals** 检查以中性结果结束（GitHub 将其视为通过），其标题以 **(observe mode)** 开头。

在以下等情况中，检查结果不是中性的：

| 情况 | 检查的行为 |
| :- | :- |
| Pull Request 更改了策略文件或分类器文件 | 它可能会等待一个人（可以是作者）。如果更改破坏了其中某个文件，检查将失败 |
| 默认分支的策略文件无法使用 | 在您[修复策略文件](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md#fix-a-policy-file-that-approvals-refuses)之前，检查会失败 |
| 标题显示 Withheld | Approvals 暂不给出通过结果，例如因为另一个打开的 Pull Request 具有相同的最新提交。检查的摘要会说明如何解除 |
| Pull Request 的目标分支不是默认分支 | 不会出现检查，除非该 Pull Request 更改了策略文件或分类器文件 |

## Approvals 的评论命令

要使用命令，请将其作为新评论发布在 Pull Request 上。您需要具有仓库的写入权限。如果您的评论收到 😕 反应，表示该命令未被完整执行。

| 命令 | 作用 |
| :- | :- |
| `@claude approvals rerun` | 再次评估最新提交。分类器可能会再次运行并产生费用。请将其单独放在评论的第一行。在来自 fork 的 Pull Request 上，[请先阅读更改，然后添加最新提交的完整 SHA，并使其成为评论的唯一文本](#pull-requests-from-forks-and-outside-contributors) |
| `@claude approve` | 对于 **Claude Approvals** 检查，可以计为您对最新提交的批准。要指定某个提交，请添加其完整 SHA，来自 fork 的 Pull Request 必须这样做。请使其成为评论的唯一文本 |
| `@claude approvals resolve <ids>` | 如果策略的 [`findings`](https://code.claude.com/docs/zh-CN/approvals-policy#findings) 列表中包含您，可以关闭具有这些 ID 的问题。请将其单独放在第一行，并附上一到八个问题 ID，例如 `f-1a2b3c4d` |

Approvals 也可以响应来自 GitHub App 机器人账户的 `@claude approvals rerun`。该评论会启动一次评估，其中分类器可能会运行并产生费用，并且不会为 [fork 的提交](#pull-requests-from-forks-and-outside-contributors)作担保。

当检查摘要说明您[不计为一个批准](https://code.claude.com/docs/zh-CN/approvals-policy.md#whether-a-pull-request’s-author-counts)时，请在您自己的 Pull Request 上评论 `@claude approve`。要撤回该批准，请编辑或删除您的每条 `@claude approve` 评论。Approvals 不会将已编辑的评论计为批准，因此要再次批准，请发布一条新评论。

## 来自 Claude、fork 和外部贡献者的 Pull Request

由 `claude[bot]` 打开的 Pull Request 需要第二个 GitHub App，Claude 才能批准它。来自 fork 或来自没有写入权限的人的 Pull Request 可能需要维护者参与。

### Claude 打开的 Pull Request

GitHub 不允许打开 Pull Request 的账户批准该 Pull Request。如果没有第二个应用，Claude 不会批准由 `claude[bot]` 打开的 Pull Request，并且 **Claude Approvals** 检查可能会以标题 **Passed; no approval from Claude** 通过。

要让 Claude 批准这些 Pull Request，请添加第二个应用：

1. **安装**：请您的 GitHub 组织的所有者在该仓库上安装 [Claude Approvals GitHub App](https://github.com/apps/claude-approvals/installations/new)。
2. **关联**：[将该应用关联到您的 Claude 组织](https://claude.ai/settings/claude-approvals/github/finish)。这需要一个既是 GitHub 组织所有者、又是 Claude 组织 Owner 或 Primary Owner 的人来操作。如果没有人同时具备这两种身份，请联系 [Anthropic 支持](https://support.claude.com)。

然后，在由 `claude[bot]` 打开的未关闭 Pull Request 上[重新运行检查](#comment-commands-for-approvals)。请保持必需检查的来源不变，因为 [Claude GitHub App](https://github.com/apps/claude) 仍会报告该检查。

### 来自 fork 和外部贡献者的 Pull Request

Approvals 无法确认 fork 的提交是由具有写入权限的人推送的。在来自 fork 的 Pull Request 上，Approvals 可能会等待维护者（即具有写入权限的人）：

* **在维护者为某个提交作担保之前**：任何分类器的回复都不计入，包括 Code Review 的回复。如果某条规则依赖于分类器，**Claude Approvals** 检查的标题可能显示为 **No classifier result counts on this commit**。
* **为某个提交作担保**：在观察模式之外，如果您具有写入权限，请阅读更改。然后评论 `@claude approvals rerun`，后面加上最新提交的完整 SHA，并作为评论的唯一文本。之后分类器即可针对该提交运行。如果不带 SHA，该命令可能会覆盖您评论时的最新提交。
* **通过需要等待维护者**：在观察模式之外，如果有规则覆盖某个已更改的文件，检查还需要一位具有写入权限且不是该 Pull Request 打开者的人批准。这适用于来自 fork 的 Pull Request，也适用于由没有写入权限的人打开的 Pull Request。在此之前，Claude 不会批准
* **带有 `"!pr/authors"` 的规则**：在来自 fork 的 Pull Request 上，任何批准都不会计入该规则，并且一旦有人批准，检查的标题可能显示为 **Cannot tell who wrote this change**。检查的摘要会说明策略文件的维护者可以更改什么

## 相关资源

* [Claude Approvals](https://code.claude.com/docs/zh-CN/approvals.md)：什么是策略以及它会影响什么
* [Claude Approvals 故障排除](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md)：未出现检查或检查未通过时
* [Claude Approvals 策略文件参考](https://code.claude.com/docs/zh-CN/approvals-policy.md)：检查摘要中提到的键
