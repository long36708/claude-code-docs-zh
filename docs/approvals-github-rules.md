> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Claude Approvals 与您的 GitHub 规则

> 了解 Claude Approvals 检查如何与必需审查、CODEOWNERS 和合并队列协同工作、哪些内容可以在无人参与的情况下合并，以及谁可以更改策略

[Claude Approvals](https://code.claude.com/docs/zh-CN/approvals.md) 会为其评估的 GitHub Pull Request 添加一项检查。该检查依据[策略](https://code.claude.com/docs/zh-CN/approvals-policy.md)（即您仓库中的一个文件）报告谁必须批准某个 Pull Request。只有当您在 GitHub 中将该检查设为必需检查时，它才会阻止合并，而能否合并由您的分支规则决定。

如果您负责管理某个仓库或审查其安全性，请在[强制执行策略](https://code.claude.com/docs/zh-CN/approvals-setup.md#enforce-your-policy-and-require-claude-approvals)之前，检查您的策略与 GitHub 设置如何协同工作：

* **[无人工审查的合并](#merges-without-human-review)**：哪些设置会让 Pull Request 在除其作者外无人审查的情况下合并，以及一条要求非作者人员参与的规则
* **[必需审查与 CODEOWNERS](#required-reviews-and-codeowners)**：GitHub 自身的审查规则如何计算 Claude 的批准审查，以及应如何设置
* **[自动合并与合并队列](#auto-merge-and-merge-queues)**：哪些内容可以在无人操作的情况下合并
* **[谁可以更改或放宽策略](#who-can-change-or-loosen-a-policy)**：谁可以编辑策略文件、取消对该检查的要求，或引导分类器的答案

## 无人工审查的合并

启用 Approvals 后，Pull Request 可能会在除其作者外无人查看的情况下合并；如果 Pull Request 由 bot 编写，甚至可能在完全无人查看的情况下合并。如果 GitHub 不要求 **Claude Approvals** 检查，就可能发生这种情况。如果该检查在除作者外无人参与的情况下通过，也可能发生这种情况，例如在以下情形中：

* `seats: 0`
* `seats: 1`，且作者可以填补该席位
* 允许 bot 的 [`from` 列表](https://code.claude.com/docs/zh-CN/approvals-policy#people-lists)
* 机器用户，即由脚本或 Agent 驱动的用户账户，它[可以被算作一个人](https://code.claude.com/docs/zh-CN/approvals-policy.md#whether-a-pull-request’s-author-counts)
* 在推送后保留席位的 [`reuse`](https://code.claude.com/docs/zh-CN/approvals-policy#reuse)
* 没有任何规则覆盖的文件，GitHub 会将其中立结果视为通过
* 在规则的 `if`、覆盖的 `if` 或 `reuse` 中读取[分类器](https://code.claude.com/docs/zh-CN/approvals-classifiers.md)答案的条件。该答案是 [Pull Request 的文本可以引导的](#classifier-answers-that-a-pull-request-can-steer)

直接推送到默认分支不会触发任何检查。请在 GitHub 中为默认分支要求使用 Pull Request。

Claude 的[批准审查](https://code.claude.com/docs/zh-CN/approvals.md#find-claude-approvals-on-a-pull-request)可以满足要求一次批准的 GitHub 分支规则。

### 要求一名非作者人员

若要让 **Claude Approvals** 检查在每条路径上都要求一名非作者人员，请将此文件中的规则添加到您[强制执行](https://code.claude.com/docs/zh-CN/approvals-setup.md#enforce-your-policy-and-require-claude-approvals)的根[策略文件](https://code.claude.com/docs/zh-CN/approvals-policy.md)中：

```yaml theme={null}
# .claude/approvals.yml
version: 1
rules:
  - purpose: "A change takes one person who isn't an author."
    paths: ["**"]
    nested: inherit
    verdict:
      claude: comments
      seats: [{ count: 1, from: [humans, "!pr/authors"] }]
    advisory: [claude/code_review]
```

该规则的每个部分各司其职：

| 规则的部分 | 作用 |
| :- | :- |
| 无 `if` 或 `override` | 任何分类器的答案都不会改变该规则 |
| `claude: comments` | Approvals 不会发布 Claude 的批准 |
| `nested: inherit` | 该规则同样适用于[子目录中的策略文件](https://code.claude.com/docs/zh-CN/approvals-policy.md#policy-files-in-subdirectories)之下 |
| `humans` | [机器用户](https://code.claude.com/docs/zh-CN/approvals-policy.md#whether-a-pull-request’s-author-counts)（即由脚本或 Agent 驱动的用户账户）可以填补此席位。若要将某个机器用户排除在外，请添加其登录名，例如 `[humans, "!pr/authors", "!acme-ci"]` |
| `"!pr/authors"` | 排除 Pull Request 的作者。对于[来自 fork 的 Pull Request](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#pull-requests-from-forks-and-outside-contributors)，任何批准都不计入此席位 |
| `advisory` | [保留自动 Code Review](https://code.claude.com/docs/zh-CN/approvals.md#keep-automatic-code-reviews) |

对于需要更多人员的路径，请复制该规则，缩小 `paths` 范围，并提高 `count`。

## 必需审查与 CODEOWNERS

请检查每项 GitHub 设置如何与您的策略协同工作，以及应如何设置：

| GitHub 设置 | 与 Approvals 的协同方式 | 操作建议 |
| :- | :- | :- |
| 必需批准 | GitHub 可以将 Claude 的批准审查计为一次批准。GitHub 不会计入填补席位的作者，也不会计入 [`@claude approve`](https://code.claude.com/docs/zh-CN/approvals-pull-requests#comment-commands-for-approvals) 评论 | 在要求 **Claude Approvals** 检查的地方，在 GitHub 中要求的批准数不要超过一次，并在 `seats` 中要求更多人员。否则，Pull Request 可能在 **Claude Approvals** 检查通过后仍处于阻止状态 |
| CODEOWNERS | 分支规则可以要求 Code Owner 的审查 | 若要在策略中指定所有者，请在其路径的 `seats` 中使用[人员列表](https://code.claude.com/docs/zh-CN/approvals-policy.md#people-lists) |
| 过期批准 | Claude 在某个提交上发布其批准审查，GitHub 可能会继续计入该审查，直到它被撤销。Approvals 通常会在评估下一次推送时请求 GitHub 撤销该审查，而 GitHub 可能会[拒绝](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#approvals-waits-for-people) | 在 GitHub 要求批准的地方，启用 **Dismiss stale pull request approvals when new commits are pushed** 或 **Require approval of the most recent reviewable push** |

## 自动合并与合并队列

如果您将 GitHub 自动合并或合并队列与不要求人员批准的策略一起使用，更改可能会在无人操作的情况下合并。若要让该检查在每条路径上都要求一名非作者人员，请[添加此规则](#ask-for-a-person-who-isn’t-an-author)，并要求 **Claude Approvals** 检查。

Approvals 不会评估合并队列创建的提交：

* **队列的提交**：Approvals 通常会将其标记为已通过，标题为 **Stamped for merge queue**，但仅在[观察模式](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#approvals-in-observe-mode)下评估过的 Pull Request 除外
* **进入队列**：由 GitHub 自身的规则决定，通常在 Pull Request 的必需检查通过之后。Pull Request 也可能在检查仍在运行时进入队列。如果此后检查未通过，请将该 Pull Request 从队列中移除
* **仅被观察过的 Pull Request**：开始强制执行策略后，请在将每个打开的 Pull Request 加入队列之前，对其[重新运行检查](https://code.claude.com/docs/zh-CN/approvals-setup.md#enforce-your-policy-and-require-claude-approvals)。在其最新提交依据该策略通过之前，它在队列中的提交通常不会得到任何检查；在要求该检查的地方，它可能会在队列中等待
* **应用的权限**：请让您 GitHub 组织的所有者接受 Claude GitHub App 的 [**Merge queues** 权限](https://code.claude.com/docs/zh-CN/github-actions.md#github-app-permissions)。若没有该权限，队列的提交不会得到任何检查，因此在 GitHub 要求该检查的地方，队列条目会一直等待

## 谁可以更改或放宽策略

策略是您仓库中的一个文件，因此由您的 GitHub 权限决定谁可以更改它。

| 人员 | 可执行的操作 |
| :- | :- |
| 任何可以合并到默认分支的人 | 更改策略文件。策略文件是[受保护文件](https://code.claude.com/docs/zh-CN/approvals-policy.md#protected-files)：更改它的 Pull Request 可能需要一名具有写入权限的人员，而此人可以是该 Pull Request 的作者。[机器用户](https://code.claude.com/docs/zh-CN/approvals-policy.md#whether-a-pull-request’s-author-counts)被算作一个人 |
| GitHub 仓库管理员 | 在仓库自身的分支规则中取消对 **Claude Approvals** 检查的要求 |
| 任何可以绕过所有要求该检查的规则的人 | 绕过该检查进行合并 |
| Claude 组织中的 [Owner 角色](https://code.claude.com/docs/zh-CN/server-managed-settings.md#access-control) | 为某个 GitHub 组织、其 **Edit** 对话框中显示 **Claude Approvals** 的仓库或整个 Claude 组织[关闭 Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md#turn-off-approvals) |

### Pull Request 可以引导的分类器答案

Approvals 会应用默认分支中的策略文件和您的分类器定义。Pull Request 的[作者文本](https://code.claude.com/docs/zh-CN/approvals-classifiers.md#what-a-classifier-run-can-read-and-change)可以引导分类器的答案。

当规则的 `if`、覆盖的 `if` 或 `reuse` 条件读取某个答案时，被引导的答案可能会降低该 Pull Request 的要求。若要在每条路径上都要求一名非作者人员，请[添加此规则](#ask-for-a-person-who-isn’t-an-author)。

## 相关资源

* [设置 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md)：强制执行策略并要求 **Claude Approvals** 检查
* [Claude Approvals 策略文件参考](https://code.claude.com/docs/zh-CN/approvals-policy.md)：席位、人员列表和受保护文件
* [Claude Approvals 分类器](https://code.claude.com/docs/zh-CN/approvals-classifiers.md)：分类器运行可以读取和更改的内容
