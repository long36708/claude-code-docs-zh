> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Claude Approvals 分类器

> 让 Claude Approvals 规则依据 Code Review 的回答或您定义的问题来决定。添加分类器、读取其回答，并限制一次运行的费用。

在 [Claude Approvals](https://code.claude.com/docs/zh-CN/approvals.md) 中，[策略文件](https://code.claude.com/docs/zh-CN/approvals-policy.md)中的规则决定谁必须批准 Pull Request。分类器让规则可以依据对变更的判断来决定，例如 Code Review 是否通过，或变更的风险有多高。例如，一条规则可以要求两个人批准，而当分类器将变更评为低风险时只需一人。

规则可以依据路径、事实和分类器来决定：

* **路径**：Pull Request 更改了哪些文件
* **[事实](https://code.claude.com/docs/zh-CN/approvals-policy.md#if)**：Pull Request 本身的详细信息，例如更改了多少行或由谁发起
* **分类器**：关于变更的某个问题的回答。当路径和事实无法表达时使用分类器

Claude Approvals 目前以[公开测试版](https://code.claude.com/docs/zh-CN/approvals.md#availability)形式向 Team 和 Enterprise 计划的组织提供。

## 分类器的工作原理

分类器有一个名称、一个问题以及它可以给出的回答。其回答通过以下步骤传达到规则：

1. **选择或定义分类器**：`claude/code_review` 和 `claude/diff` [随 Claude Approvals 提供](#classifiers-anthropic-provides)。自定义分类器是仓库中默认分支上的一个文件夹 `.claude/classifiers/<name>/`
2. **在规则中命名它**：在策略文件 `.claude/approvals.yml` 中，将分类器的名称写入规则的条件。对于位于文件夹 `change_risk` 中的自定义分类器，名称为 `repo/change_risk`，例如 `"repo/change_risk = low"`
3. **Approvals 在 Pull Request 上运行它**：例如，当一个 Pull Request 以待审查状态打开，并更改了该规则匹配的文件时。请参阅[分类器何时运行](#when-a-classifier-runs)。每次调用模型的运行都会从您组织的使用额度中[计费](#what-a-classifier-run-costs)
4. **规则使用该回答**：该回答帮助决定谁必须批准。它显示在 Claude 在 Pull Request 上的评论中的 **Classifier results** 下，除非由 [`reuse`](https://code.claude.com/docs/zh-CN/approvals-policy#reuse) 条件读取它

## 分类器定义示例

您在名为 `def.yml` 的文件中定义自定义分类器。该文件在 `prompt` 中包含给模型的问题，在 `outputs` 中包含模型可以选择的回答。

此示例定义了一个名为 `change_risk` 的分类器。其 `prompt` 要求模型评估变更的风险程度，其 `outputs` 允许模型回答 `low`、`medium` 或 `high`：

```yaml theme={null}
# .claude/classifiers/change_risk/def.yml
version: 1
description: "How risky is this change to ship?"
model: opus
prompt: |
  Rate how risky this change is to ship.
  high: it touches sign-in, billing or a data migration.
  medium: it changes behavior a user can see.
  low: anything else.
  When two ratings fit, pick the higher.
outputs:
  risk: { type: ordinal, values: [low, medium, high] }
```

示例中的每个键各司其职：

| 键 | 在此示例中 |
| :- | :- |
| `version` | 格式的版本，`1` |
| `description` | 一行说明分类器所问的内容 |
| `model` | 作答的 Claude 模型，此处为最新的 Opus 模型 |
| `prompt` | 问题。它说明每个评级的含义，以及两个评级都适用时选择哪一个 |
| `outputs` | 模型从中选择的回答。此分类器有一个输出 `risk`，其 `values` 从最低排到最高 |

定义还可以设置 `inputs`、`tools` 和 `budget`。请参阅[分类器定义文件键](#classifier-definition-file-keys)。

### 在规则中使用示例的评级

分类器以其文件夹命名。示例保存在 `.claude/classifiers/change_risk/def.yml`，因此您在策略文件的规则中写 `repo/change_risk`。

要使用该评级，请在规则的条件中将其与某个值进行比较。例如，当模型回答 `low` 时，`if: "repo/change_risk = low"` 为真。关于在该条件下要求更少人员的规则，请参阅[从规则读取分类器的回答](#read-a-classifier’s-answer-from-a-rule)。

## Anthropic 提供的分类器

Claude Approvals 附带预构建的分类器 `claude/code_review` 和 `claude/diff`。要使用其中之一，请在某条规则的条件中写入其名称。

### `claude/code_review`

`claude/code_review` 告诉规则 [Code Review](https://code.claude.com/docs/zh-CN/code-review.md) 是否通过。例如，当其回答为 `pass` 时，`if: "claude/code_review = pass"` 为真。

`claude/code_review` 给出以下回答之一：

| 回答 | 何时 |
| :- | :- |
| `pass` | 审查完成，且没有未关闭的 [Important 发现](https://code.claude.com/docs/zh-CN/code-review-pull-requests.md#severity-levels) |
| `fail` | 审查完成，且有未关闭的 Important 发现 |

您可以在任何条件中读取它，或将其列在 [`advisory`](https://code.claude.com/docs/zh-CN/approvals-policy#advisory) 下。

* **要读取其未关闭发现的严重程度**：写 `claude/code_review.severity`，其值为 `none`、`minimal` 或 `high`
* **要允许他人将 `fail` 变为 `pass`**：在策略的 [`findings`](https://code.claude.com/docs/zh-CN/approvals-policy#findings) 列表中列出这些人。他们通过关闭 Important 发现来实现

### `claude/diff`

`claude/diff` 告诉规则自某次批准或更早的回答以来，Pull Request 变化了多少。它回答 `trivial`、`same_intent` 或 `different`。

在 [`reuse`](https://code.claude.com/docs/zh-CN/approvals-policy#reuse) 下的条件中读取它，`reuse` 说明推送后哪些内容继续有效。

## 添加自定义分类器

使用自定义分类器时，您编写一个关于 Pull Request 的问题，由 Claude 模型回答。您在仓库中的文件里定义分类器，然后在规则中命名它：

1. **定义分类器**：在 `.claude/classifiers/` 下创建一个文件夹，名称使用小写字母、数字和 `_`。在其中保存一个 [`def.yml`](#classifier-definition-file-keys)，包含问题以及可供选择的回答。在规则中，文件夹 `change_risk` 写作 `repo/change_risk`。
2. **在 `advisory` 下命名它**：在根目录的 `.claude/approvals.yml` 中，选择一条具有 `verdict` 且其 `paths` 匹配您希望评判的文件的规则，并将分类器列在该规则的 [`advisory`](https://code.claude.com/docs/zh-CN/approvals-policy#advisory) 下。在 `advisory` 下，分类器会运行并[计费](#what-a-classifier-run-costs)，但不决定任何事情。
3. **合并这两个文件**：用一个 Pull Request 同时提交这两个文件，Claude 会在该 Pull Request 上发表评论。如果您编辑的规则处于强制执行状态，预计会出现一条内容为 "rules still hold on these files" 的警告，它不会阻止合并。当该评论的 **Policy files in this change** 部分未报告任何拒绝时，合并该 Pull Request。
4. **读取其回答**：打开一个更改这些文件的 Pull Request，并在 Claude 的评论中[查找回答](#when-a-classifier’s-answer-appears)。
5. **让规则依据它**：一旦您信任这些回答，就[从规则中读取它们](#read-a-classifier’s-answer-from-a-rule)。

### 分类器的回答何时出现

在合并分类器的定义以及命名它的规则之后，请等待五分钟。然后打开一个非草稿、且更改了该规则 `paths` 所匹配文件的 Pull Request。在以下位置之一查找分类器对应的行：

* **Claude 的评论**：在 **Classifier results** 下
* **Claude Approvals 检查的摘要**：在 **Advisory classifiers (not counted toward this verdict)** 下

起初，该行显示 "no result recorded yet"。一次运行最多可能需要 10 分钟；如果分类器的 `def.yml` 设置了更长的[时间限制](#budget)，则可能更久。在强制执行的规则下，运行结束时该行保持不变。

回答会在 Approvals 下一次评估该 Pull Request 时显示，例如有人批准它或请求更改时。届时该行显示类似 "`repo/change_risk`: reported `low` (advisory)" 的内容。

* **该行仍未显示结果**：等到运行的时间限制结束，然后请人批准该 Pull Request 或请求更改
* **您的分类器没有对应的行**：检查命名该分类器的规则是否有 `verdict`，以及其 `paths` 是否匹配您更改的文件

## 从规则读取分类器的回答

要使规则依据分类器的回答，请将分类器的名称写入条件：规则自身的 `if`，或某个 [`override`](https://code.claude.com/docs/zh-CN/approvals-policy#override) 的 `if`。当 `repo/change_risk` 在 `advisory` 下回答良好后，将其名称移入某个覆盖的 `if`。

将严格的 verdict 保留在规则本身上，因为[当运行未给出回答时](#what-a-rule-sees-when-a-classifier-run-gives-no-answer)，适用的是规则的 verdict。

此规则要求两个人，而当 `repo/change_risk` 回答 `low` 时只需一人：

```yaml theme={null}
# .claude/approvals.yml
version: 1
rules:
  - purpose: "Two people. One is enough when the risk is low."
    paths: ["**"]
    verdict: { claude: comments, seats: 2 }
    override:
      - { if: "repo/change_risk = low", verdict: { claude: comments, seats: 1 } }
```

### 在条件中读取分类器的方式

条件可以用以下形式读取分类器：

| 形式 | 示例 | 读取 |
| :- | :- | :- |
| 仅名称 | `"repo/change_risk <= medium"` | 唯一的输出 |
| `.output` | `"repo/api_review.breaking = yes"` | 按名称读取某个输出。当分类器的定义有多个输出时必须使用 |
| `.severity` | `"repo/api_review.severity >= high"` | 最严重的未关闭发现，取值为 `none`、`minimal`、`low`、`medium`、`high` 或 `critical`。需要在 `tools` 中包含 `findings` |
| `(input=value)` | `"claude/code_review(level=deep) = pass"` | 使用该输入的一次运行。Code Review 的 `level` 为 `quick`、`standard` 或 `deep` |
| `= error`、`= incomplete` | `"repo/change_risk = error"` | 运行在没有回答的情况下如何结束 |

## 分类器定义文件键

在 `def.yml` 中编写分类器的定义，文件最大可达 64 KiB。完整文件请参阅[分类器定义示例](#example-classifier-definition)。

您可以在 `def.yml` 中编写以下键：

| 键 | 是否必需 | 值 |
| :- | :- | :- |
| `version` | 是 | `1`，以裸数字形式 |
| `description` | 是 | 一行，最多 1,000 个字符 |
| `model` | 是 | 小写的模型系列，例如 `opus` 或 `sonnet`，或[模型概览](https://platform.claude.com/docs/en/about-claude/models/overview)中某个正式发布模型的准确 Claude API ID。该模型必须[支持自动模式](https://code.claude.com/docs/zh-CN/permission-modes.md#eliminate-prompts-with-auto-mode) |
| `prompt` | 此项或 `prompt.md` | 问题。对于较长的问题，请将 `prompt.md` 保存在 `def.yml` 旁边，最大 512 KiB |
| [`outputs`](#outputs) | 否 | 输出名称到其类型和值的映射 |
| [`inputs`](#inputs) | 否 | 规则可以设置的输入名称的映射 |
| [`tools`](#tools) | 否 | 一个列表 |
| [`budget`](#budget) | 否 | `{ time, cost }` |

如果您指定模型系列，分类器会使用该系列的最新模型，因此费用和回答可能在没有任何 Pull Request 的情况下发生变化。如果您指定准确的 ID，当该模型退役时，分类器将停止加载。您组织的允许模型列表适用于这些运行。

### `outputs`

为每个输出指定 `type`，名称和值使用小写字母、数字和 `_`。不要将输出命名为 `severity`，也不要使用 `true`、`false`、`error`、`incomplete` 或 `complete` 作为值。为 `"yes"` 和 `"no"` 加上引号。

| `type` | 还需要 | 条件中的[运算符](https://code.claude.com/docs/zh-CN/approvals-policy.md#if) |
| :- | :- | :- |
| `criteria` | `values`，一个词列表 | `=`、`!=` |
| `ordinal` | `values`，最低的在前 | 全部六个 |
| `numeric` | `min` 和 `max`，整数 | 全部六个 |

### `inputs`

输入是规则可以为一次运行设置的值。在 `def.yml` 的 `inputs` 下定义每个输入，然后在规则中于分类器名称后的括号内设置它，例如 `repo/change_risk(rigor=thorough)`。为每个输入指定 `default`，以及 `values` 或 `min` 和 `max`，例如 `rigor: { values: [quick, thorough], merge: right, default: quick }`。在提示词中说明每个值会改变什么。

输入的 `merge` 说明当多条规则在同一个 Pull Request 上要求不同的值时会发生什么：

| `merge` | 运行什么 |
| :- | :- |
| `left`、`right` | 一次运行，使用在 `values` 中排在更前或更后的那个词 |
| `min`、`max` | 一次运行，使用较小或较大的数字 |
| `list` | 一次运行，使用所要求的每个词 |
| 省略 | 为每个值单独进行一次计费运行 |

### `tools`

在 `tools` 下列出条目，让运行可以做更多事情：

| 条目 | 允许运行 | 限制 |
| :- | :- | :- |
| `findings` | 读取 Pull Request 审查线程的部分内容，将发现报告为新发现，并解决或重新打开其自己的发现 | 16 个未关闭线程 |
| `comment` | 发布一条评论 | 5,000 个字符 |
| `code` | 读取 diff 之外的文件 | 只读 |
| `labels` | 添加仓库中已有的标签，并移除由它添加的标签 | 8 个标签。不能是名称以 `approvals:` 或 `claude-` 开头的标签 |
| `reviewers` | 添加审查者，并移除由它添加的审查者 | 8 个审查者 |

在观察模式下以及在某些 Pull Request 上，Approvals 会限制运行的部分 `tools`：

* **在观察模式下**：运行仅保留其 `tools` 中的 `findings` 和 `code`。Approvals 可以在其自己的评论中列出这些发现，且不会打开新的审查线程。当您的策略文件，或命名该分类器并匹配某个已更改文件的每条规则，都声明 [`mode: observe`](https://code.claude.com/docs/zh-CN/approvals-policy#rule-keys) 时，此情况成立。
* **在某些 Pull Request 上**：运行不会添加评论、标签、审查线程或 GitHub 审查请求。例如来自 fork 的 Pull Request，以及作者缺少写入权限的 Pull Request

#### `labels`

当某个标签用于把控合并、部署或工作流时，不要授予 `labels`。运行添加的标签是 Pull Request 上的真实标签，任何读取标签的东西都可以据此采取行动。

#### `reviewers`

在提示词中按登录名说明要添加谁。当某条[适用](https://code.claude.com/docs/zh-CN/approvals-policy.md#how-rules-combine-into-one-verdict)的规则[读取分类器的回答](#read-a-classifier’s-answer-from-a-rule)时，除 [`seats`](https://code.claude.com/docs/zh-CN/approvals-policy#verdict) 之外，**Claude Approvals** 检查还可以等待他们的批准。请指定具有写入权限的人员。如果策略文件位于子目录中，请[仅在 `advisory` 下列出该分类器](https://code.claude.com/docs/zh-CN/approvals-policy#policy-files-in-subdirectories)。

要移除无法批准的审查者，如果提示词中点名了此人，请先合并对提示词的更改，并等待五分钟。然后在 Pull Request 上评论 [`@claude approvals rerun`](https://code.claude.com/docs/zh-CN/approvals-pull-requests#comment-commands-for-approvals)。

### `budget`

在 `budget` 下设置 `time`、`cost` 或两者，例如 `budget: { time: 300, cost: 5 }`。达到任一限制的运行会停止，此时条件读取为 `incomplete`。如果您要求的值超过最大值，运行将获得最大值，并且 Approvals 会向您发出警告。

| 键 | 单位 | 默认值 | 最大值 |
| :- | :- | :- | :- |
| `time` | 整秒 | `600` | `3600` |
| `cost` | 整美元 | `10` | `10` |

当运行的预估支出达到 `cost` 时，Approvals 会停止该运行，而实际账单可能更高。若低于 \$5，在大型 Pull Request 上的运行可能在作答之前就停止。

## 分类器运行的行为

规则的结果可能取决于其分类器何时运行、运行能读取什么，以及运行未给出回答时会发生什么。

### 分类器何时运行

只有当您在匹配的规则中命名 `repo/` 分类器时，它才会在 Pull Request 上运行：在 `if`、某个覆盖的 `if`、`reuse` 条件或 `advisory` 中。在[来自 fork 的 Pull Request](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#pull-requests-from-forks-and-outside-contributors) 上，在维护者为该提交担保之前，`repo/` 分类器不会启动。关于 `claude/code_review`，请参阅 [Approvals 何时启动审查](https://code.claude.com/docs/zh-CN/approvals-setup.md#what-approvals-costs)。

* **什么可以启动运行**：Pull Request 以待审查状态打开、被标记为待审查或被重新打开。更改 diff 的推送、基础分支的更改以及 [Claude Approvals 检查的重新运行](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#comment-commands-for-approvals)也可以启动运行。在[您的策略停止观察](https://code.claude.com/docs/zh-CN/approvals-setup.md#enforce-your-policy-and-require-claude-approvals)之后，Approvals 会自行评估一些打开的 Pull Request，这可能会启动运行。
* **运行哪个定义**：默认分支上的定义和提示词副本。Pull Request 自身的副本不会运行。该文件夹中的文件是[受保护文件](https://code.claude.com/docs/zh-CN/approvals-policy.md#protected-files)。
* **更改定义会使先前的回答失效**：对 `def.yml` 或 `prompt.md` 的编辑在进入默认分支后生效。打开的 Pull Request 会保留其检查，直到下一次评估，例如推送或重新运行。该评估会丢弃先前的回答，分类器可以再次运行。
* **在更改 diff 的推送之后**：如果 [`reuse`](https://code.claude.com/docs/zh-CN/approvals-policy#reuse) 保留了先前的回答，或者推送仅更改了其规则未命名该分类器的受覆盖文件，则先前的回答仍可有效
* **顺序**：使用默认的 `run: sequential` 时，某个覆盖的分类器通常会等待其上方的覆盖。使用 `run: parallel` 时，它们可以同时启动，这可能会对任何 verdict 都不需要的运行计费。

### 分类器运行可以读取和更改什么

`repo/` 分类器的运行是在 Anthropic 基础设施上的 Claude Code 云端会话。如果您的组织要求使用自托管计算，条件将读取为 `error`，以代替该运行的回答。

请检查会话可以读取什么以及可以更改什么：

* **它可以读取什么**：Pull Request 的标题、描述、标签、作者、已更改的文件、diff、批准和必需审查者，分类器先前的运行，您的策略文件，以及 `.claude/classifiers/` 下的文件。当 `tools` 中包含 `findings` 时，它会读取审查线程的部分内容
* **`code` 增加了什么**：Pull Request head 和默认分支上文件的只读副本，但不包括某些文件，例如 `CLAUDE.md`、`AGENTS.md`、`.claude/`、`.mcp.json` 以及名称以 `.pdf` 结尾的文件
* **它可以更改什么**：会话的 GitHub 访问权限是只读的。当 [`tools`](#tools) 列出了具有写入能力的条目时，运行会记录其请求，Approvals 可以在运行之后执行这些请求。

运行会读取 Pull Request 作者编写的文本，而这些文本可能试图操纵分类器的回答。在重要的路径上，请在不读取任何分类器的规则中[要求由人批准](https://code.claude.com/docs/zh-CN/approvals-github-rules.md#ask-for-a-person-who-isn’t-an-author)。

### 当分类器运行未给出回答时规则看到什么

当运行在没有回答的情况下结束时，条件通常读取为以下值之一：

| 条件读取为 | 何时 |
| :- | :- |
| `incomplete` | 运行已开始，但在没有回答的情况下结束 |
| `error` | 运行无法启动、被中断，或超过了检查的等待时间 |

在这两种情况下，需要该回答的覆盖会保留规则自身的 `verdict`。需要该回答的规则自身 `if` 会使 **Claude Approvals** 检查失败，标题为 **A classifier had no result**。

要选择其他 verdict，请在读取回答的覆盖之上，针对 `= error` 或 `= incomplete` 放置[一个覆盖](https://code.claude.com/docs/zh-CN/approvals-policy.md#override)。对于规则自身 `if` 中缺失的回答，覆盖无法选择其他 verdict。请将读取分类器的比较从规则的 `if` 移入某个覆盖。

## 分类器运行的费用

调用模型的运行会按 token 从您组织的使用额度中计费：

| 分类器 | 计费内容 |
| :- | :- |
| 自定义分类器 `repo/<name>` | 每次运行。包括观察模式下的运行、`advisory` 下的运行，以及未给出回答就结束的运行。发生错误后，Approvals 可以自行多次重新启动运行，每一次运行都会计费 |
| `claude/diff` | 每次调用模型的回答 |
| `claude/code_review` | 按 [Code Review](https://code.claude.com/docs/zh-CN/code-review-costs.md) 计费 |

### 限制自定义分类器的费用

您可以为单次运行设置上限、为每月设置上限、选择成本更低的模型，或减少运行次数：

* **为单次运行设置上限**：在分类器的 `def.yml` 中添加 `budget: { cost: 5 }`。当运行的预估支出达到该美元金额时，Approvals 会停止该运行，而实际账单可能更高。如果没有 [`budget`](#budget)，上限为 \$10
* **为每月设置上限**：在 [**Organization settings > Usage**](https://claude.ai/admin-settings/usage) 中设置[分类器运行的支出限制](https://code.claude.com/docs/zh-CN/approvals-setup.md#limit-what-a-policy-costs)，这需要 Owner 角色
* **选择成本更低的模型**：在 `def.yml` 中更改 `model`，例如从 `opus` 改为 `sonnet`
* **减少运行次数**：在一个 Pull Request 中完成对 `def.yml` 和 `prompt.md` 的编辑。每次合并编辑后，分类器都会在每个打开的 Pull Request 的下一次评估时再次运行

## 相关资源

* [Claude Approvals 策略文件参考](https://code.claude.com/docs/zh-CN/approvals-policy.md)：条件、覆盖、`reuse` 和 `findings`
* [设置 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md)：观察模式、强制执行和支出限制
* [Claude Approvals 故障排除](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md)：当分类器未给出回答时
* [数据使用](https://code.claude.com/docs/zh-CN/data-usage.md)：Anthropic 如何处理来自 Claude Code 的数据
