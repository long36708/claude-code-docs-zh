> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Claude Approvals 故障排除

> 修复未出现或未通过的 Claude Approvals 检查、被 Approvals 拒绝的策略文件，以及未给出答案的分类器

[Claude Approvals](https://code.claude.com/docs/zh-CN/approvals.md) 是 GitHub Pull Request 上的一项检查，用于报告还有谁需要审批。当检查未出现或未通过、Approvals 拒绝策略文件，或分类器未给出答案时，请从这里开始。

如果 Pull Request 上 **Claude Approvals** 检查的标题未在此处涵盖，请参阅[查找您的审批状态以及后续操作](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#find-your-approval-status-and-what-to-do-next)。

## 未出现 Claude Approvals 检查

当 [Claude Approvals](https://code.claude.com/docs/zh-CN/approvals.md) 无法运行时，它可能什么都不会发布。

### 检查您的 Pull Request 和策略文件

先从您可以自行检查的原因入手：

| 原因 | 操作 |
| :- | :- |
| Pull Request 是草稿 | 将其标记为可供审查 |
| [策略文件](https://code.claude.com/docs/zh-CN/approvals-policy.md#where-to-put-a-policy-file)不在 Approvals 查找的位置 | 将其放在默认分支上仓库根目录的 `.claude/approvals.yml` 中，并包含 `version: 1` |
| 文件合并后没有任何新动作 | 向 Pull Request 推送一个提交 |
| 策略处于观察模式，且 Pull Request 的目标分支不是默认分支 | 预期不会出现检查，除非 Pull Request 更改了策略文件或分类器文件 |
| 策略已强制执行，且没有任何规则覆盖 Pull Request 更改的任何文件 | 请参阅[没有规则覆盖文件时会发生什么](https://code.claude.com/docs/zh-CN/approvals-policy.md#how-rules-combine-into-one-verdict) |

### Claude 留下了评论但没有 Approvals 检查

找到 Claude 评论中的措辞，然后执行其旁边的步骤：

| 评论内容 | 操作 |
| :- | :- |
| Approvals "did not run on this change" | 按照评论所述操作。其中一个原因是 Pull Request 将文件移入或移出策略规则所覆盖的路径。请分两个 Pull Request 完成该移动：一个以新名称添加文件，另一个删除旧文件 |
| "More than one Claude organization is connected to this GitHub account" | 请应运行 Approvals 的 Claude 组织的 Owner 在 [**Organization settings > Claude Ship**](https://claude.ai/admin-settings/claude-ship) 上点击仓库所属 GitHub 组织旁边的 **Connect**。然后推送一个提交 |

### 请 Owner 检查您的 Claude 组织

请 Owner 针对您的 Claude 组织逐项检查以下内容：

| 检查内容 | 位置 |
| :- | :- |
| 组织是否符合条件 | [可用性](https://code.claude.com/docs/zh-CN/approvals.md#availability) |
| [Claude GitHub App](https://github.com/apps/claude) 是否可以访问该仓库并已关联 | [**Organization settings > Git providers**](https://claude.ai/admin-settings/source-control) |
| Approvals 是否在此处被关闭 | [**Organization settings > Claude Ship**](https://claude.ai/admin-settings/claude-ship) 上的 **Approvals** 列和 **Approve** 行 |

### 检查 GitHub IP 允许列表

如果请求来自不在组织 IP 允许列表中的地址，GitHub 会拒绝 Approvals 的请求，这可能导致既没有检查也没有评论。

请询问仓库所属 GitHub 组织或企业的所有者是否按 IP 地址限制访问。如果是，请让该所有者按照 [GitHub 允许列表和防火墙](https://code.claude.com/docs/zh-CN/network-config.md#github-allow-lists-and-firewalls)中的说明，允许 Anthropic 的所有出站地址。这些地址获得允许后，向您的 Pull Request 推送一个提交。

## 解决审查线程后 Approvals 仍未通过

检查您策略的 [`findings`](https://code.claude.com/docs/zh-CN/approvals-policy#findings) 列表中是否列出了您。

如果您的解决操作不计入，请推送修复，或请列表中的某人解决该线程。

## Approvals 已通过但 GitHub 仍阻止合并

阅读 GitHub 合并框中指出的规则。如果它要求审查，说明 GitHub 的分支规则要求进行一项策略未要求的审查，因此请让作者以外的人在 GitHub 审查中批准。有关 GitHub 计入哪些审查，请参阅[必需审查和 CODEOWNERS](https://code.claude.com/docs/zh-CN/approvals-github-rules.md#required-reviews-and-codeowners)。

## 修复被 Approvals 拒绝的策略文件

在下表中找到 **Claude Approvals** 检查的标题：

| 标题 | 出错的副本 | 消息 | 操作 |
| :- | :- | :- | :- |
| **This change breaks a policy file** | Pull Request 自身的副本 | 检查摘要中的 **Policy files in this change** | 在此 Pull Request 中修复 |
| **Default branch policy file cannot be used**、**Default branch policy file must be version 1**、**Default branch policy file is too large** | 默认分支上的根策略文件 | 检查摘要 | 从此仓库的某个分支发起新的 Pull Request 进行修复。其 **Policy files in this change** 会显示新副本是否能加载。无论处于观察模式还是强制执行，该 Pull Request 的检查都会失败，与所有 Pull Request 的检查一样。如果该分支要求此检查，请让可以绕过分支规则的人将其合并 |
| **A policy file cannot be used** | 默认分支上某个目录的策略文件 | 检查摘要 | 按照修复根策略文件的方式进行修复 |

修复合并后，等待五分钟，然后在每个检查尚未更新的已打开 Pull Request 上评论 [`@claude approvals rerun`](https://code.claude.com/docs/zh-CN/approvals-pull-requests#comment-commands-for-approvals)。对于来自 fork 的 Pull Request，请[先阅读更改并添加最新提交的完整 SHA](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#pull-requests-from-forks-and-outside-contributors)。

### 策略文件拒绝消息

一条拒绝会使整个策略文件不可用，而警告不会导致任何失败。

在此处找到您的消息开头：

| 消息 | 修复 |
| :- | :- |
| "no key of this name is read at this place" | 对照[顶级键](https://code.claude.com/docs/zh-CN/approvals-policy.md#top-level-keys)和[规则键](https://code.claude.com/docs/zh-CN/approvals-policy.md#rule-keys)检查键的拼写和层级 |
| "this names no fact and no usable classifier" | 对照[事实](https://code.claude.com/docs/zh-CN/approvals-policy.md#if)检查名称。对于 `repo/<name>`，检查策略文件旁边或其上级目录中的 `.claude/classifiers/<name>/def.yml` 是否能加载 |
| "a condition has no NOT" | 改写为相反的运算符 |
| "a login goes in single quotes inside a condition" | 如果该词是登录名，请将其放在单引号中，例如 `"pr/opened_by = 'ana-acme'"`。对于机器人，请写 `bot:` 加上其名称，不带引号，例如 `"pr/opened_by = bot:dependabot"` |
| "no local has this name" | 在此文件或其上级的策略文件中定义该 local。登录名或团队名不要带 `@` |
| "this is not YAML" | 在消息指出的行附近查找未闭合的引号或括号，或制表符。请使用空格缩进 |

## 分类器未给出答案或其定义被拒绝

要查看每次[分类器](https://code.claude.com/docs/zh-CN/approvals-classifiers.md)运行的状态以及缺少答案的原因，请在 Claude 于 Pull Request 上的评论中打开 **Classifier results**：

* **某次运行达到了预算**：拆分 Pull Request，或调高低于其[上限](https://code.claude.com/docs/zh-CN/approvals-classifiers.md#budget)的 `time` 或 `cost`
* **某次运行需要读取的内容超出了单次运行的处理能力**，例如较大的 diff：请尝试使用较小的 Pull Request

然后在此处找到 **Claude Approvals** 检查的标题：

| 标题 | 操作 |
| :- | :- |
| **A classifier definition is not valid**，或策略文件引用了该分类器时的 **This change breaks a policy file** | 在此 Pull Request 中修复定义。拒绝消息位于 **Policy files in this change** 下 |
| **A classifier's definition cannot be used** | 在默认分支上修复定义。更改该定义的 Pull Request 会显示其被拒绝的原因 |
| **Cannot evaluate: classifier outside advisory** | 当策略文件位于子目录中时，某些自定义分类器[只能放在 `advisory` 下](https://code.claude.com/docs/zh-CN/approvals-policy#policy-files-in-subdirectories)。请将该分类器移到那里 |
| **repo/change\_risk reached its run limit**（显示您的分类器名称），或 **A classifier reached its run limit** | 向 diff 推送一项更改。或者评论 `@claude approvals rerun` 再运行一次，这可能会丢弃所有分类器的结果，包括已通过的结果 |

## 相关资源

* [在 Pull Request 上使用 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md)：检查标题和评论命令
* [设置 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md)：要求、入门策略和必需检查
* [Claude Approvals 策略文件参考](https://code.claude.com/docs/zh-CN/approvals-policy.md)：策略文件的键
