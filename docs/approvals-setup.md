> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 设置 Claude Approvals

> 在 GitHub 仓库中启用 Claude Approvals：从观察模式开始，强制执行策略，将检查设为必需，限制其花费，以及将其关闭

[Claude Approvals](https://code.claude.com/docs/zh-CN/approvals.md) 是 GitHub Pull Request 上的一项检查，用于报告还有谁需要批准。要在仓库中启用它，请在仓库默认分支的根目录下提交一个位于 `.claude/approvals.yml` 的[策略文件](https://code.claude.com/docs/zh-CN/approvals-policy.md)。一旦该文件进入默认分支，Approvals 即可开始[计费](#what-approvals-costs)。

## 前提条件

设置并强制执行 Approvals 需要以下各项：

* 一个[符合条件](https://code.claude.com/docs/zh-CN/approvals.md#availability)的 Claude 组织，并具有[使用额度](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans)，用于支付策略所启动的模型运行费用
* [Claude GitHub App](https://github.com/apps/claude)，已安装在该仓库上并[关联到您的 Claude 组织](https://code.claude.com/docs/zh-CN/admin-setup.md#connected-github-accounts)
* 合并到默认分支的权限
* GitHub 上该仓库的管理员访问权限，以便将 **Claude Approvals** 检查设为必需。私有仓库还需要[付费 GitHub 计划](https://docs.github.com/en/get-started/learning-about-github/githubs-plans)
* 在强制执行本页中的任一策略之前，需要有两个具有写入权限的人。对于 `.claude/` 下的更改（包括策略文件），两项策略都要求有两个人参与

## 规划您的第一个策略

在[策略文件](https://code.claude.com/docs/zh-CN/approvals-policy.md)中，您可以设置谁需要批准 Pull Request。对于第一个策略，请决定 Claude 是否可以批准，然后复制与之匹配的入门策略：

* **[在 Code Review 通过时让 Claude 批准](#let-claude-approve-when-code-review-passes)**：通过 Claude 审查的 Pull Request 通常只需要其作者
* **[每项更改保留两个人](#keep-two-people-on-each-change)**：Pull Request 通常需要作者以外的一个人，且 Claude 只发表评论

两个策略都以观察模式开始。在观察模式下，**Claude Approvals** 检查会显示策略将作出的决定，并且该检查[通常视为通过](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#approvals-in-observe-mode)。

两个策略还都指定了 `claude/code_review`。请将它保留在与您希望审查的文件相匹配的规则中，以便在观察模式下以及强制执行后，[自动 Code Review 能够继续进行](https://code.claude.com/docs/zh-CN/approvals.md#keep-automatic-code-reviews)。

> [!TIP]
>
> 如需让 Claude 为您编写或更改策略文件，请在仓库中的 Claude Code 输入框中输入 [`/claude-approvals`](https://code.claude.com/docs/zh-CN/commands)，后跟您想要的内容。需要 Claude Code v2.1.331 或更高版本以及[功能标志获取](https://code.claude.com/docs/zh-CN/env-vars.md#features-that-need-feature-flag-fetching)。

### 在 Code Review 通过时让 Claude 批准

强制执行此入门策略后，通过 Claude 审查的 Pull Request 通常只需要其作者，`.claude/` 下的更改除外。其他 Pull Request 通常需要[作者以外](https://code.claude.com/docs/zh-CN/approvals-policy.md#whether-a-pull-request’s-author-counts)的一个人。

在此策略中，第一条规则要求一项更改由两个人批准。其 `override` 在 `claude/code_review` 读取为 `pass` 时将人数降为一人，并允许 Claude 批准。第二条规则要求 `.claude/` 下的更改由两个人批准：

```yaml theme={null}
# .claude/approvals.yml
version: 1
mode: observe
rules:
  - purpose: "A change takes two people. When Claude's review passes, one is enough and Claude approves."
    paths: ["**"]
    verdict: { claude: comments, seats: 2 }
    override:
      - { if: "claude/code_review = pass", verdict: { claude: approves, seats: 1 } }
  - purpose: "A change to the review rules takes two people."
    paths: ["**/.claude/**"]
    nested: inherit
    verdict: { claude: comments, seats: 2 }
```

此策略可在每次推送时启动一次计费的 Code Review，且观察模式也会计费。如果您的 [Code Review 设置](https://code.claude.com/docs/zh-CN/code-review-setup.md#set-when-reviews-run)要求的审查次数更少，该策略会在其基础上增加审查。

### 每项更改保留两个人

使用此入门策略可要求每项更改由两个人批准。强制执行该策略后，Pull Request 通常需要作者以外的一个人，且 Claude 只发表评论。

在此策略中，两条规则都要求两个人。第一条规则将 `claude/code_review` 列在 [`advisory`](https://code.claude.com/docs/zh-CN/approvals-policy#advisory) 下，因此审查遵循您的 Code Review 设置：

```yaml theme={null}
# .claude/approvals.yml
version: 1
mode: observe
rules:
  - purpose: "A change takes two people."
    paths: ["**"]
    verdict: { claude: comments, seats: 2 }
    advisory: [claude/code_review]
  - purpose: "A change to the review rules takes two people."
    paths: ["**/.claude/**"]
    nested: inherit
    verdict: { claude: comments, seats: 2 }
```

## 以观察模式启用 Approvals

在单独的 Pull Request 中添加策略文件，然后打开第二个 Pull Request 以查看 **Claude Approvals** 检查：

**1. 添加策略文件**

在单独的 Pull Request 中，从仓库内的某个分支，将策略添加到仓库根目录下的 `.claude/approvals.yml`。

**2. 合并 Pull Request**

当 Claude 评论称其"found the policy file in this change valid"时进行合并。如果文件无效，会出现一个标题为 **This change breaks a policy file** 的失败检查。请在此 Pull Request 中[修复它](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md#fix-a-policy-file-that-approvals-refuses)。

**3. 打开另一个 Pull Request**

打开一个目标为默认分支且不是草稿的 Pull Request。会出现一个标题以 **(observe mode)** 开头的 **Claude Approvals** 检查。

如果没有出现任何内容，请参阅[未出现 Claude Approvals 检查](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md#no-claude-approvals-check-appeared)。

## 强制执行策略并将 Claude Approvals 设为必需

当您在观察模式下看到的 **Claude Approvals** 检查标题与审查者的决定一致时，即可强制执行策略。从那时起，该检查可以阻止合并，并且在策略允许的情况下 [Claude 可以批准](https://code.claude.com/docs/zh-CN/approvals-github-rules.md#merges-without-human-review)。

### 决定要保护的文件和分支

在[将策略从观察模式切换到强制执行](#switch-from-observe-mode-to-enforcement)之前，请决定哪些文件需要人工批准、该检查管控哪些分支，以及何时更改 GitHub 要求的批准数：

* **文件**：决定哪些文件无论 Claude 发现什么都需要人工批准，然后复制[此规则](https://code.claude.com/docs/zh-CN/approvals-github-rules.md#ask-for-a-person-who-isn’t-an-author)并缩小其 `paths` 列表
* **分支**：强制执行的策略也涵盖目标为默认分支以外分支的 Pull Request，因此请列出 **Claude Approvals** 检查应管控其合并的每个分支。在这些分支上，不包含任何受涵盖文件的 Pull Request 不会获得检查，因此必需检查会一直等待
* **GitHub 要求的批准数**：仅在切换的最后一步之后，再[更改 GitHub 要求的批准数](https://code.claude.com/docs/zh-CN/approvals-github-rules.md#required-reviews-and-codeowners)

### 从观察模式切换到强制执行

策略文件的 [`mode`](https://code.claude.com/docs/zh-CN/approvals-policy#top-level-keys) 键为以下值之一：

* **`observe`**：**Claude Approvals** 检查显示策略将作出的决定，Claude 不批准，并且该检查[通常视为通过](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#approvals-in-observe-mode)
* **`enforce`**：该检查可以阻止合并，并且在策略允许的情况下 Claude 可以批准。这是默认值，因此没有 `mode` 行的策略文件会被强制执行

要强制执行策略，请按照以下步骤操作。在第一步之后，打开的 Pull Request 可能会保留一个 GitHub 视为通过的检查，直到您对其重新运行 Approvals。在第 2 步中对打开的 Pull Request 重新运行 Approvals 之前，请勿将它们放入合并队列。

**1. 删除 mode: observe 行**

删除 `mode: observe` 并合并该更改。

**2. 对打开的 Pull Request 重新运行 Approvals**

合并五分钟后，在每个检查标题以 **(observe mode)** 或 **Observe mode ended** 开头的 Pull Request 上评论 `@claude approvals rerun`。对于来自 fork 的 Pull Request，请[先阅读更改并添加最新提交的完整 SHA](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#pull-requests-from-forks-and-outside-contributors)。

**3. 在 GitHub 中将 Claude Approvals 设为必需**

对于默认分支以及您列出的每个分支，[要求状态检查通过](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#require-status-checks-before-merging)并添加 **Claude Approvals**。选择报告该检查的 Claude 应用作为其来源。

**4. 确认 Approvals 会阻止合并**

打开一个更改 `.claude/` 下某个文件的 Pull Request。检查完成后，标题显示为 **Waiting on human review from any person (1/2)**，并且 GitHub 的合并框将该检查列为必需。

Approvals 稍后会评估部分您未重新运行的 Pull Request，这可能会启动[计费运行](#what-approvals-costs)。

### 设置 GitHub 接受的 Claude Approvals 来源

将必需检查的来源设置为 Claude 应用。如果允许 **any source**，GitHub 会接受来自其他地方、名为 **Claude Approvals** 的检查。要通过 GitHub 的 API 设置来源，请将 Claude 应用的 ID `1236702` 放入规则集中的 `integration_id` 或分支保护规则中的 `app_id`。

## 控制 Approvals 的花费

Approvals 会针对策略启动的模型运行扣除您组织的使用额度，且观察模式也会计费。计费可以启动，并且[在使用额度关闭时仍会继续](https://code.claude.com/docs/zh-CN/claude-ship.md#how-each-claude-ship-agent-is-billed)。要阻止任何策略文件启动计费，Owner 可以为整个组织[关闭 Approvals](#turn-off-approvals)。

### Approvals 的费用

在 `if` 条件中包含 `claude/code_review` 的策略，可在 Pull Request 打开时以及每次推送时启动一次[计费的 Code Review](https://code.claude.com/docs/zh-CN/code-review-costs.md)。要停止这些审查，请将其从条件中移到 `advisory`。当 `claude/code_review` 仅位于 `advisory` 下且没有 [`level`](https://code.claude.com/docs/zh-CN/approvals-classifiers#read-a-classifier’s-answer-from-a-rule) 时，审查遵循您的 Code Review 设置。

规则还可以读取其他[分类器](https://code.claude.com/docs/zh-CN/approvals-classifiers.md)。对于 `claude/diff` 和自定义分类器，每次[调用模型的运行](https://code.claude.com/docs/zh-CN/approvals-classifiers.md#what-a-classifier-run-costs)都会计费。要限制自定义分类器的每次运行，请设置其 [`budget`](https://code.claude.com/docs/zh-CN/approvals-classifiers#budget) 键。

### 限制策略的费用

要为您的组织为策略所启动的 Code Review 和分类器运行支付的费用设置上限，请设置支出限额。可在 [**Organization settings > Usage**](https://claude.ai/admin-settings/usage) 设置以下任一限额，这需要 Owner 角色：

* **组织的使用额度支出限额**：涵盖 Approvals
* **Code Review 的支出限额**：涵盖策略的审查以及 Code Review 的其他审查。在 **Services** 下，打开 **Claude Code Review** 行的菜单并选择 **Edit spend limit**
* **分类器运行的支出限额**：在 **Services** 下，打开 Approvals 行的菜单并选择 **Edit spend limit**。该行出现时没有限额

在该页面上，**Services** 下的 **Claude Code Review** 行可能在没有限额的情况下开始：

* **该行在限额位置显示短横线**：当策略的第一次计费审查启动时，系统会为您设置月度限额。金额显示后即可更改
* **Code Review 在您的策略之前已启用**：在您设置限额之前，该行没有限额

如果由经销商向您的组织计费，请向经销商申请限额。

达到 Code Review 的限额时，需要 Claude 审查的检查可能无法再通过。请提高限额或等待[下一个计费周期](https://code.claude.com/docs/zh-CN/code-review-costs.md#limit-your-spend)。

## 关闭 Approvals

您可以为单个仓库、某个 GitHub 组织或整个 Claude 组织关闭 Approvals。

> [!WARNING]
>
> 仅在关闭 Approvals 之后，才可暂停或卸载 Claude GitHub App，或取消关联您的 GitHub 组织。如果先执行其中任何一项操作，必需的 **Claude Approvals** 检查可能会阻止合并。GitHub 还可能继续计入 Claude 在打开的 Pull Request 上留下的批准审查。

请按以下顺序操作：

1. **检查 GitHub 如何处理过时的批准**：在 GitHub 要求批准的地方，启用 [**Dismiss stale pull request approvals when new commits are pushed**](https://code.claude.com/docs/zh-CN/approvals-github-rules.md#required-reviews-and-codeowners)。否则，在新的推送之后，GitHub 可能会继续计入 Claude 的批准审查，直到您在最后一步中驳回该审查。
2. **不再将该检查设为必需**：在 GitHub 中，从每个分支保护规则和规则集的必需检查中移除 **Claude Approvals**。
3. **关闭 Approvals**：根据所需的作用域[选择方式](#ways-to-turn-off-approvals)。
4. **驳回 Claude 的审查**：在 Claude 已批准的打开的 Pull Request 上，在 GitHub 中驳回该审查。

若要在不关闭 Approvals 的情况下阻止该检查阻止合并，请先在默认分支以外的分支上不再将其设为必需。然后在每个策略文件中加入 `mode: observe`。此后该检查[通常不再阻止合并](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#approvals-in-observe-mode)，而分类器仍会运行并计费。

### 关闭 Approvals 的方式

要关闭 Approvals，请在下表中找到所需的作用域：

| 作用域 | 如何关闭 Approvals |
| :- | :- |
| 单个仓库，无需 Owner | 从默认分支删除所有策略文件 |
| 某个 GitHub 组织，或其 **Edit** 对话框显示 **Claude Approvals** 的仓库 | 具有 [Owner 角色](https://code.claude.com/docs/zh-CN/server-managed-settings.md#access-control)时，前往 [**Organization settings > Claude Ship**](https://claude.ai/admin-settings/claude-ship)，点击其所在行的 **Edit**，然后关闭 **Claude Approvals** |
| 整个 Claude 组织 | 具有 Owner 角色时，前往 [**Organization settings > Claude Ship**](https://claude.ai/admin-settings/claude-ship)，在 **Approve** 行中点击 **Block globally**。要重新启用，请在同一行中点击 **Allow policy files** |

为整个 Claude 组织关闭 Approvals 后，即使之后有策略文件合并，它仍保持关闭。如果您也要关闭 Code Review，请先关闭 Approvals，因为即使 Code Review 已关闭，策略仍可能启动计费的审查。

## 相关资源

* [Claude Approvals 策略文件参考](https://code.claude.com/docs/zh-CN/approvals-policy.md)：键和示例策略
* [Approvals 与您的 GitHub 规则](https://code.claude.com/docs/zh-CN/approvals-github-rules.md)：哪些内容可以在无人批准的情况下合并，以及谁可以更改策略
* [Claude Approvals 故障排除](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md)：未出现检查，或检查拒绝策略文件时
