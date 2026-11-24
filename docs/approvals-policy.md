> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Claude Approvals 策略文件参考

> Claude Approvals 策略文件 `.claude/approvals.yml` 的参考：其键、规则如何组合、审批如何计数，以及一个示例策略。

策略文件 `.claude/approvals.yml` 告诉 [Claude Approvals](https://code.claude.com/docs/zh-CN/approvals.md) 谁必须审批 Pull Request，以及 Claude 何时可以审批它。您可以在此查阅其键、其规则如何组合，以及审批如何计数。

> [!TIP]
>
> 要设置 Approvals 并将 **Claude Approvals** 检查设为必需，请参阅[设置 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md)。要了解检查标题的含义，请参阅[在 Pull Request 上使用 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md)。对于被检查拒绝的文件，请参阅[修复被 Approvals 拒绝的策略文件](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md#fix-a-policy-file-that-approvals-refuses)。要定义 `repo/<name>` 分类器，请参阅 [Claude Approvals 分类器](https://code.claude.com/docs/zh-CN/approvals-classifiers.md)。

## 策略文件的工作方式

策略文件是一个规则列表。每条规则在 `paths` 中指明其所针对的文件，并说明更改这些文件的 Pull Request 需要什么：

* **`seats`**：需要多少人审批
* **`claude`**：Claude 是 `approves`、`comments` 还是 `blocks`
* **`override`**：另一个裁决，在其 `if` 为真时替换规则自身的裁决

以下策略文件是[设置 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md#let-claude-approve-when-code-review-passes) 所使用的起始文件。它要求一项更改由两个人审批，或者在 Claude 的审查通过时由一个人审批。在这两种情况下，Pull Request 的作者都可以算作其中一人：

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

该文件的每一行各负责一项工作：

| 行 | 作用 |
| :- | :- |
| [`mode: observe`](#top-level-keys) | **Claude Approvals** 检查显示规则将会作出的决定。删除此行即可强制执行规则 |
| `purpose` | 说明规则存在的原因 |
| [`paths: ["**"]`](#paths) | 第一条规则匹配所有文件 |
| [`verdict`](#verdict) | 需要两个人审批，且[作者可以算作其中一人](#whether-a-pull-request’s-author-counts)。Claude 发表评论，不进行审批 |
| [`override`](#override) | 当 [`claude/code_review`](https://code.claude.com/docs/zh-CN/approvals-classifiers#classifiers-anthropic-provides) 读数为 `pass` 时，一个人（可以是作者）即可，且 Claude 可以审批 |
| `paths: ["**/.claude/**"]` | 第二条规则匹配任何 `.claude/` 目录下的文件，其中包括策略文件 |
| [`nested: inherit`](#policy-files-in-subdirectories) | 在您强制执行策略后，使第二条规则在拥有自己策略文件的子目录中仍然有效 |

`.claude/` 下的文件同时匹配这两条规则。Approvals [采用所适用裁决中最严格的那个](#how-rules-combine-into-one-verdict)，因此在您强制执行策略后，对该文件的更改需要两个人，因为第二条规则没有 `override`。

## 策略文件的存放位置

将策略文件保存为仓库根目录下的 `.claude/approvals.yml`，并将其合并到默认分支。Approvals 从默认分支读取策略文件，因此文件合并后其规则即生效。

您也可以将文件命名为 `.claude/approvals.yaml`，在 `.yml` 文件不存在时，Approvals 会读取它。要为某个目录设置专属规则，请[在该目录中添加策略文件](#policy-files-in-subdirectories)。

### 策略文件的更改何时生效

当对策略文件的更改处于一个打开的 Pull Request 中时，Approvals 可以检查更改后的文件是否有错误，但不会应用其中的任何规则。该 Pull Request 由默认分支上的规则决定。

更改合并后，新的 Pull Request 会采用新规则，打开中的 Pull Request 则从其下一次评估起采用新规则。Approvals 可能还会在几分钟内读取之前的副本。要立即评估一个打开的 Pull Request，请评论 [`@claude approvals rerun`](https://code.claude.com/docs/zh-CN/approvals-pull-requests#comment-commands-for-approvals)。

### 子目录中的策略文件

要在根策略文件之外为某个目录设置专属规则，请在该目录内添加 `.claude/approvals.yml`，例如 `services/billing/.claude/approvals.yml`。

对于每个更改的文件，Approvals 从该文件所在目录开始向上查找。第一个覆盖该更改文件、且其中有带 `verdict` 的规则[适用](#how-rules-combine-into-one-verdict)的强制执行策略文件，将为该文件作出决定。更上层策略文件中的 `nested: inherit` 规则会被视为该决定策略文件的规则之一。

* **最近的策略文件作出决定**：它的要求可以低于根策略文件。在它之上，只有标记为 `nested: inherit` 的规则仍然有效。
* **`paths` 从该目录开始**：在 `services/billing/.claude/approvals.yml` 中，写 `api/**` 即可匹配 `services/billing/api/` 下的文件
* **`locals` 向下可达**：在一个策略文件中，您可以使用其上层策略文件的 locals
* **每个策略文件有自己的 `mode`**：当根策略文件处于观察模式时，在某目录的强制执行策略文件之下，根策略文件的任何规则都不生效，包括 `nested: inherit` 规则
* **某些自定义分类器只能放在 `advisory` 下**：即 [`tools`](https://code.claude.com/docs/zh-CN/approvals-classifiers#tools) 列出了 `reviewers` 或 `labels` 的分类器，或者具有不带 `merge` 的[输入](https://code.claude.com/docs/zh-CN/approvals-classifiers.md#inputs)的分类器。请在没有 `if` 的强制执行规则中列出它。如果某条规则读取其结果，则某目录策略文件之下的 Pull Request 上的检查可能显示 **Cannot evaluate: classifier outside advisory**

## 顶层键

Approvals 会拒绝包含未知键的文件。键以及 `observe` 和 `approves` 等固定词请使用小写。每个 purpose、模式、条件以及 `"!"` 项都要用双引号括起来。

请将文件保持为单个 YAML 文档，为每个 `!` 项加引号，并使用空格缩进，否则 Approvals 会拒绝该文件。锚点、别名和 `!!` 标签会引发警告。

在文件顶层编写以下键：

| 键 | 必需 | 值 | 默认值 | 说明 |
| :- | :- | :- | :- | :- |
| `version` | 是 | `1` | | schema 版本，写为不带引号的数字 |
| `rules` | 是 | [规则](#rule-keys)列表 | | 规则顺序无关紧要 |
| `mode` | 否 | `enforce`、`observe` | `enforce` | `observe` 显示规则将会作出的决定，并对文件中的每条规则生效。关于仍不为中性的检查，以及针对其他分支的 Pull Request，请参阅[观察模式下的 Approvals](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#approvals-in-observe-mode) |
| `nested` | 否 | `overwrite`、`inherit` | `overwrite` | 此文件中规则的默认值。请参阅[子目录中的策略文件](#policy-files-in-subdirectories) |
| `run` | 否 | `sequential`、`parallel` | `sequential` | 此文件中规则的默认值。请参阅[分类器何时运行](https://code.claude.com/docs/zh-CN/approvals-classifiers.md#when-a-classifier-runs) |
| `locals` | 否 | 名称到值的映射 | | 要复用的值。请参阅 [`locals` 和 `use`](#locals-and-use) |

## 规则键

为每条规则提供 `purpose`、`paths`，以及 `verdict`、`advisory`、`findings` 和 `reuse` 中的至少一个。如果这四个中规则只有 `reuse`，则 `reuse` 的值必须为 `false` 或指定一个分类器。没有 `verdict` 的规则不要求任何人，也不[覆盖](#how-rules-combine-into-one-verdict)任何文件。

您可以在规则上编写以下键：

| 键 | 类型 | 默认值 | 说明 |
| :- | :- | :- | :- |
| `purpose` | 一行文本 | 必需 | 规则存在的原因 |
| [`paths`](#paths) | 模式列表 | 必需 | 规则匹配的已更改文件 |
| [`verdict`](#verdict) | 映射 | | Claude 做什么以及谁必须审批 |
| [`if`](#if) | 条件 | `true` | 规则仅在条件为真时适用 |
| [`override`](#override) | 列表 | | 其他裁决，每个都附带一个条件 |
| [`advisory`](#advisory) | 分类器列表 | | 会运行但不作决定的分类器 |
| [`reuse`](#reuse) | 条件或映射 | 不保留任何内容 | 在更改 diff 的推送之后仍然有效的内容 |
| [`findings`](#findings) | 映射 | 无列表 | 谁可以关闭分类器的发现 |
| [`use`](#locals-and-use) | `"@name"` 列表 | | 其键会被复制到规则中的 locals |
| `mode` | `enforce`、`observe` | 文件的值 | 处于观察模式的规则既不[适用于也不覆盖](#how-rules-combine-into-one-verdict)文件 |
| `nested` | `overwrite`、`inherit` | 文件的值 | `inherit` 使规则在下层策略文件之下仍然有效 |
| `run` | `sequential`、`parallel` | 文件的值 | 规则自身的分类器是依次启动还是同时启动 |

### `paths`

将每个模式写为从包含 `.claude` 文件夹的目录开始的路径。被移动的文件在其旧路径和新路径上都会计入。

| 模式 | 匹配 |
| :- | :- |
| `*` | 单个文件夹内的任意字符 |
| `**` | 任意数量的文件夹，包括以点开头的文件夹。它必须是一个完整的段，如 `**/*.go` |
| `{a,b}`、`[ab]`、`?` | 备选项、集合中的一个字符，以及任意一个字符 |
| `infra/**` | 该文件夹中的所有文件。单独的 `infra` 不匹配其中任何文件 |
| `*.md` | 与策略文件的 `.claude` 文件夹同级的 Markdown 文件。若要匹配任意深度，请写 `**/*.md` |
| `"!docs/**"` | 仅从此规则中排除这些文件。规则至少需要一个不带 `!` 的模式 |

### `if`

将条件写为 `true`、`false`，或一个双引号字符串，其左侧为某个事实或分类器的名称，例如 `"pr/files_changed > 50"`。当您为规则设置 `if` 时，请在相同路径上保留另一条没有 `if` 的规则。当 `if` 为假时，带有它的规则不适用。如果有规则覆盖某个文件但没有规则适用，[Claude Approvals 检查会失败](#how-rules-combine-into-one-verdict)。

* **运算符**：`=`、`!=`、`<`、`<=`、`>`、`>=`
* **连接**：大写的 `AND` 和 `OR`。混合使用两者时请使用括号。
* **列表**：在 `pr/authors` 和 `pr/approved_by` 前写 `ANY` 或 `ALL`。当条件导向更宽松的裁决时，使用 `ALL`：若使用 `ANY`，一个列出的作者就会为其他所有人的提交放宽规则。当条件导向更严格的裁决时，使用 `ANY`：若使用 `ALL`，一个未列出的作者就会使其失效。
* **引号**：登录名或分支模式用单引号括起来，团队、`bot:` 名称或事实则不加引号，如 `"ALL pr/authors = ['ana-acme', acme/payments]"`
* **分类器**：`claude/code_review = pass`、`repo/change_risk <= medium`，以及[在规则中读取分类器的答案](https://code.claude.com/docs/zh-CN/approvals-classifiers.md#read-a-classifier’s-answer-from-a-rule)下的其他形式

事实来自 Pull Request 本身：

| 事实 | 示例 | 包含 |
| :- | :- | :- |
| `pr/files_changed`、`pr/files_added`、`pr/files_removed` | `"pr/files_changed > 50"` | 整个 Pull Request 中的计数 |
| `pr/diff_lines` | `"pr/diff_lines <= 20"` | 整个 Pull Request 中更改的行数 |
| `rule/files_changed`、`rule/files_added`、`rule/files_removed` | `"rule/files_changed = pr/files_changed"` | 与此规则的 `paths` 匹配的文件中的计数 |
| `pr/authors` | `"ALL pr/authors = bot:dependabot"` | 编写或推送了提交的账户，以及打开 Pull Request 的账户 |
| `pr/opened_by` | `"pr/opened_by = acme/release"` | 打开 Pull Request 的账户。当其他人向某个 bot 的 Pull Request 推送后，基于 `pr/opened_by = bot:dependabot` 的覆盖将不再降低规则的要求 |
| `pr/approved_by` | `"ANY pr/approved_by = acme/security"` | 具有写入权限且审批了当前 diff 的人。不包括作者和 bot。也支持 `= empty` 和 `!= empty` |
| `pr/base` | `"pr/base = 'release/**'"` | Pull Request 的目标分支 |

### `verdict`

`verdict` 说明 Claude 做什么以及谁必须审批。在规则的 `verdict` 中写 `claude`；除非 `claude` 为 `blocks`，否则还要写 `seats`。

| 字段 | 值 | 说明 |
| :- | :- | :- |
| `claude` | `approves`、`comments`、`blocks` | `approves`：如果[每条适用的规则都这样指定且每个更改的文件都被覆盖](#how-rules-combine-into-one-verdict)，则在 **Claude Approvals** 检查通过后，Claude 可以发布批准性审查。该字段不会启动任何审查，因此在规则不读取任何分类器的情况下，Claude 可以审批一个未经任何审查阅读过的更改。`comments`：Claude 不审批。`blocks`：无论谁审批，检查都会失败 |
| `seats` | `0` 到 `16`，或最多八个集合的列表 | 需要多少[人](#how-people’s-approvals-are-counted)审批。数字 `N` 是 `[{ count: N, from: humans }]` 的简写。与 `blocks` 同时出现时会被拒绝 |
| `max_age` | 一个整数加上 `s`、`m`、`h` 或 `d`，范围从 `1m` 到 `83d` | 一次评估的结果有效多长时间。超过此时间后，检查会失败，标题为 **Results are older than the policy allows** |

为 `seats` 中的每个集合提供一个 `count`（从 `1` 到 `16`）和一个 `from`（即一个[人员列表](#people-lists)）。每个集合都必须被填满，且一个人可以在多个集合中填充席位。

### 人员列表

人员列表用于指定账户。可将其写为 [`seats`](#verdict) 下某个集合的 `from`，或写在 [`findings`](#findings) 下，写成单个项或方括号中的列表。只有对仓库具有写入权限的账户才会计入。

| 项 | 含义 |
| :- | :- |
| `humans` | 所有人，包括作者。机器用户（即由脚本或 Agent 驱动的用户账户）算作人。Claude 的云端会话用于提交的 `claude` 用户账户则不算 |
| `bots` | 所有 bot 和 GitHub App，包括 Claude 的应用和 `github-actions` |
| `everyone` | `humans` 和 `bots` 的总和。单独书写，不加方括号 |
| `nobody` | 无人。单独书写，仅可用于 `findings` 下 |
| `ana-acme` | 一个登录名。在条件中，请用单引号括起来 |
| `acme/payments` | 仓库所属 GitHub 组织中的一个团队 |
| `bot:dependabot` | 一个 bot，不带 `[bot]` 后缀。`bot:claude` 表示 Claude 的应用 |
| `pr/authors`、`pr/opened_by` | 该事实所包含的账户 |
| `"!pr/authors"` | 排除账户。请加引号，并与一个不带 `!` 的项放在一起 |

如果某个集合的 `from` 接纳 Claude（例如通过 `bots`、`everyone` 或 `bot:claude`），且每条适用的规则都指定 `approves`，则 Approvals 会在 Claude 审批之前将该集合中的一个席位视为已填充。此时 **Claude Approvals** 检查可能在没有任何人审批的情况下通过。

### `override`

覆盖会在其自身的 `if` 为真时更改所属规则的裁决。Approvals 从上往下读取 `override` 列表，第一个为真的覆盖生效。

| 键 | 必需 | 说明 |
| :- | :- | :- |
| `if` | 是 | 一个[条件](#if) |
| `verdict` | 这三个中至少一个 | 它写入的每个字段都会替换规则的对应字段。`seats` 会被整体替换 |
| `reuse`、`findings` | 这三个中至少一个 | 替换规则的整个值 |
| `purpose`、`use` | 否 | 与规则上的相同。此处 `use` 可以复制 `verdict` 和 `findings` |

当覆盖的 `if` 读取某个分类器时，**Claude Approvals** 检查会等待该分类器运行：

* **运行仍在进行中**：检查保持进行中状态。当现有审批已满足规则仍可能选择的每个裁决时，检查可以提前完成，除非该分类器的 `tools` 列出了 [`reviewers`](https://code.claude.com/docs/zh-CN/approvals-classifiers#reviewers)。如果子目录中有策略文件，[请只在 `advisory` 下列出此类分类器](#policy-files-in-subdirectories)
* **运行结束但没有答案**：读取在该覆盖处停止，规则自身的 `verdict` 生效。因此，请将严格的裁决放在规则上。
* **您想捕获缺失的答案**：在读取答案的覆盖之上，放置一个基于 `= error` 或 `= incomplete` 的覆盖

在规则自身的 `if` 中，没有答案的比较会使检查失败，无论谁审批，除非条件的另一部分已能决定结果。

### `advisory`

在 `advisory` 下，列出分类器名称或调用，例如 `[claude/code_review, repo/change_risk]`。此处列出的 `repo/` 分类器[可以运行](https://code.claude.com/docs/zh-CN/approvals-classifiers.md#when-a-classifier-runs)，且每次运行都会计费。请在具有 `verdict` 的规则中列出它。

要保留 Code Review 的自动审查，请在具有 `verdict` 并匹配您希望被审查的文件的规则中，在此处列出 `claude/code_review`，或在 `if` 中读取它。

### `reuse`

`reuse` 说明在更改 diff 的推送之后，哪些先前的审批和分类器答案仍然有效，其条件通常读取 [`claude/diff`](https://code.claude.com/docs/zh-CN/approvals-classifiers#classifiers-anthropic-provides)。

当 `claude/diff` 的答案至多为 `same_intent` 且 Claude 对新提交的审查通过时，以下示例可以保留人员的审批：

```yaml theme={null}
# in .claude/approvals.yml, on a rule
    reuse:
      seats: "claude/diff <= same_intent AND claude/code_review = pass"
```

以下列形式之一编写 `reuse`：

| 形式 | 可以保留 |
| :- | :- |
| `seats: <condition>` | 人员的审批 |
| `<classifier>: <condition>` | 该分类器先前的答案 |
| `reuse: <condition>` | 两者：席位以及规则所读取的分类器 |
| `false` | 什么都不保留。在覆盖中，`unset` 会移除规则的值 |

在依赖 `reuse` 之前，请检查以下情况：

* **匹配文件的每条规则**：在每条规则上都编写 `reuse`
* **当 `claude/diff` 没有给出答案时**：读取它的条件不会保留任何内容
* **在驳回或强制推送之后**：该示例不会保留被 GitHub 驳回的审批，通常也不会保留强制推送之前给出的审批。`reuse` 可以在只读取 `claude/diff` 而不读取其他分类器的 `seats` 条件下保留它们，例如 `"claude/diff <= same_intent"`。
* **先前的作者也会计入**：在 `reuse` 保留 `seats` 的情况下，在较早提交时持有[作者席位](#whether-a-pull-request’s-author-counts)的人可以继续计入。其中两人可以在无人审批的情况下填满 `seats: 2`。为防止这种情况，请使用 `"!pr/authors"` 将作者从集合中排除。
* **推送未改动的文件**：在编写了 `reuse` 的策略文件中，审批可以在仅针对其他文件的推送之后继续有效。这些文件可以是没有规则覆盖的文件，或者其规则无法计入该人的文件。被 GitHub 驳回的审批通常不会继续有效

### `findings`

分类器可以将发现报告为审查线程，条件可以通过 `.severity` 读取其严重程度。使用 `findings` 指定谁可以关闭发现，方式是解决其线程或使用 [`@claude approvals resolve`](https://code.claude.com/docs/zh-CN/approvals-pull-requests#comment-commands-for-approvals)。

在规则的 `findings` 列出谁可以关闭发现之前，仅凭某人的解决操作不会降低 `.severity`。分类器关闭自己的发现不需要列表，例如 Code Review 在之后的审查发现问题已修复时就可以这样做。

以下示例允许 `acme/leads` 团队（不含 Pull Request 的作者）关闭 Code Review 的发现：

```yaml theme={null}
# in .claude/approvals.yml, on a rule with no if, beside verdict
    findings:
      resolution:
        claude/code_review: [acme/leads, "!pr/authors"]
```

在 `findings` 下，使用以下值之一编写 `resolution`：

| 值 | 含义 |
| :- | :- |
| 一个[人员列表](#people-lists) | 一个适用于所有分类器发现的列表。`resolution: humans` 允许具有写入权限的人关闭发现，包括作者 |
| 分类器名称到列表的映射 | 每个分类器一个列表 |
| 集合列表，每个为 `{ if, from }` | 每个 `if` 可以读取 `finding/severity`，如 `{ if: "finding/severity >= high", from: acme/leads }`。如果每个 `if` 都只读取 `finding/severity`，则第一个 `if` 为真的集合作出决定。以一个没有 `if` 的集合结尾 |
| `nobody` | 仅凭任何人的解决操作都不能关闭发现。分类器可以关闭自己的发现 |

要允许人员关闭发现，请将列表放在一条没有 `if` 且覆盖该发现所在文件的规则上。将该规则放在为该文件[作出决定的策略文件](#policy-files-in-subdirectories)中。要使仅凭解决操作即可计入，适用于该文件的每个列表都必须接纳执行解决操作的人。

失败的 `claude/code_review` 是针对整个 Pull Request 的一个答案。请检查每个更改的文件是否位于比较该答案的规则之下，如 `claude/code_review = pass`：

* **每个文件的列表都接纳执行解决操作的人**：一旦该审查的所有 Important 发现都被关闭，它就可以读数为 `pass`，无需新的审查
* **某个文件的列表不接纳执行解决操作的人，或该文件为 `nobody` 或没有列表**：该人的解决操作会使审查保持为 `fail`。请为该文件编写一个接纳该人的列表。

关闭发现不会更改任何代码，因此人们可以在未修复任何内容的情况下降低 `.severity` 并使失败的审查通过。请保持列表简短。

### `locals` 和 `use`

要复用某个值，请在策略文件顶部的 `locals` 下为其命名。然后在需要该值的位置写 `"@name"`，或使用 `use` 将整个映射复制到规则中。

在此示例中，`payments_owner` 指定了一个人员集合，`strict` 指定了一个完整的裁决，第一条规则通过 `use` 复制了它：

```yaml theme={null}
# .claude/approvals.yml
version: 1
locals:
  payments_owner: { count: 1, from: [acme/payments, "!pr/authors"] }
  strict:
    verdict: { claude: comments, seats: [{ count: 2, from: humans }, "@payments_owner"] }
rules:
  - purpose: "The payments API takes two people, one of them a payments owner."
    paths: ["services/payments/api/**"]
    use: ["@strict"]
  - purpose: "Everything else takes two people."
    paths: ["**"]
    verdict: { claude: comments, seats: 2 }
```

请检查 local 可以包含什么以及可以在哪里使用它：

* **local 包含的内容**：文本、数字、`true` 或 `false`、列表，或一个映射
* **`"@name"` 可以出现的位置**：作为整个 `paths`、人员列表、`seats`、`verdict` 或 `override`，以及作为 `paths`、人员列表或 `seats` 中的一项。在条件内部，它不需要单独加引号，如 `"pr/files_changed > @many"`。
* **`use` 复制的内容**：`verdict`、`override`、`if`、`paths`、`reuse`、`advisory` 和 `findings`。规则自身写入的内容优先：在 `verdict`、`findings` 和 `reuse` 映射内按键优先，对其他任何键则整体优先。

## 规则如何组合成一个裁决

Approvals 采用所适用裁决中最严格的那个：`blocks` 高于 `comments`，`comments` 高于 `approves`，且每条规则的席位都必须被填满。哪些裁决适用，取决于规则是否匹配、适用于或覆盖某个更改的文件：

* **匹配**：当规则的 `paths` 包含该文件且没有 `"!"` 项将其排除时，规则匹配该更改的文件
* **适用**：当规则匹配、其 [`mode`](#rule-keys) 为 `enforce`，且其 `if` 为真或不存在时，规则适用于该更改的文件
* **覆盖**：当规则匹配、其 `mode` 为 `enforce` 且具有 `verdict` 时，无论其 `if` 如何，规则都覆盖该更改的文件

Approvals 依次选出策略文件、规则，然后是裁决：

1. **策略文件**：对于每个更改的文件，选出覆盖该文件、且其中有带 `verdict` 的规则适用的[最近的强制执行策略文件](#policy-files-in-subdirectories)。更上层策略文件中的 `nested: inherit` 规则算作它的规则。
2. **规则**：适用于每个更改文件的规则。
3. **每条规则的裁决**：规则自身的裁决，或其第一个为真的 [`override`](#override)。
4. **Pull Request 的单一裁决**：所有更改文件中最严格的裁决。一个人可以在多条规则中填充席位，且最短的 `max_age` 生效。

在强制执行的策略下，以下情况会以其他方式结束：

| 情况 | 结果 |
| :- | :- |
| 没有规则覆盖某个更改的文件 | 不会要求任何人审查该文件。[受保护文件](#protected-files)仍可能需要一个人。Claude 不审批该 Pull Request |
| 没有规则覆盖任何更改的文件 | 通常不会出现检查。如果 GitHub 要求在默认分支上进行该检查，它通常以中性状态结束，标题为 **Nothing ran, not blocking: no file is covered**。GitHub 将其视为通过。如果没有根策略文件，未更改任何目录策略文件之下文件的 Pull Request 可能不会获得检查，因此必需的检查会一直等待 |
| 有规则覆盖某个更改的文件，但没有规则适用 | 检查失败，标题以 **No rule applies to** 开头 |
| 文件在规则覆盖的路径与不覆盖的路径之间移动 | 不会出现检查，Claude 会评论说明 Approvals 未运行。如果 GitHub 要求该检查，Pull Request 会一直等待 |

为避免这些情况，请使用没有 `if` 的根规则覆盖所有路径，就像[起始策略](#how-a-policy-file-works)的第一条规则那样。

## 人员的审批如何计数

当具有写入权限的账户在 GitHub 审查中或通过 [`@claude approve`](https://code.claude.com/docs/zh-CN/approvals-pull-requests#comment-commands-for-approvals) 评论给出审批时，该审批可以计入。它必须针对最新的提交，或针对[仍然有效](#earlier-approvals-after-a-new-push)的较早提交。

当具有写入权限的人在包含被覆盖文件的 Pull Request 上请求更改时，一旦没有其他因素阻挡，**Claude Approvals** 检查通常会显示 **Changes requested**。

### Pull Request 的作者是否计入

在由具有写入权限的人打开的 Pull Request 上，作者无需审批即可在接纳他们的集合中填充一个席位。除了[来自 fork 或外部贡献者的 Pull Request](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#pull-requests-from-forks-and-outside-contributors) 之外，作者一人即可满足 `seats: 1`。`seats: 2` 可以是作者加上另一个人。

* **谁持有作者席位**：在打开 Pull Request 时的提交上，是打开它的人。在之后的推送之后，通常是既编写又推送了最新提交的人。
* **何时无人持有**：例如，在推送了一个由一人编写、另一人推送的提交之后。**Claude Approvals** 检查的摘要会给出原因；如果 Pull Request 是您打开的，您随后可以评论 [`@claude approve`](https://code.claude.com/docs/zh-CN/approvals-pull-requests#comment-commands-for-approvals)。
* **排除作者**：写 `seats: [{ count: 1, from: [humans, "!pr/authors"] }]`。在由指定人员组成的集合中，指定的人数至少要比 `count` 多一个，否则作者自己的 Pull Request 可能会一直等待。在[来自 fork 的 Pull Request](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#pull-requests-from-forks-and-outside-contributors) 上，任何审批都不会计入排除了 `pr/authors` 的集合。
* **`"!pr/authors"` 的依据**：GitHub 认定为编写或推送了提交或打开了 Pull Request 的账户。它不读取 `Co-authored-by` 行，Agent 以另一个账户进行的提交也不会使某人成为作者。
* **Bot**：bot 仅在接纳它的集合中、在由它打开且最新提交由它推送的 Pull Request 上填充该席位
* **机器用户**：以 `claude` 之外的用户账户工作的 Agent 与人一样持有该席位，并且可以是[受保护文件](#protected-files)所需的那一个人。要将其排除，请使用 `"!"` 将其登录名从规则的集合中排除，如 `[humans, "!acme-ci"]`。对于受保护文件，请将该集合放在一条 `paths` 指定这些文件、并标记为 `nested: inherit` 的规则中。

### 新推送之后的先前审批

推送之后，先前的审批是否继续计入取决于推送更改了什么：

* **diff 的文本保持不变**：未被 GitHub 驳回的先前审批可以计入。
* **diff 的文本发生变化**：先前的审批通常不再计入。在策略编写了 [`reuse`](#reuse) 的情况下，部分审批可以继续计入

Approvals 在评估推送时，通常会[请求 GitHub 驳回](https://code.claude.com/docs/zh-CN/approvals-github-rules.md#required-reviews-and-codeowners) Claude 自己的审批，且只有在 **Claude Approvals** 检查再次通过时，Claude 才会审批新的提交。

## 受保护文件

受保护文件是 Approvals 对控制 Pull Request 审查方式的文件的称呼，例如策略文件本身。对此类文件的编辑可能削弱之后 Pull Request 的审查。例如，对策略文件的编辑可能降低它们所需的人数。

在有规则覆盖某个更改文件的情况下，Approvals 通常会对受保护文件的更改[要求一个人审批](#who-has-to-approve-a-change-to-a-protected-file)。受保护文件的列表是内置的：

* **策略和分类器文件**：所有策略文件，以及 `.claude/classifiers/` 下的所有内容
* **审查说明**：仓库根目录下的 `REVIEW.md`
* **Git 文件**：`.gitmodules`、`.gitattributes`、符号链接和子模块指针
* **代码所有者**：任何名为 `CODEOWNERS` 的文件

不要依赖其他文件受到保护。请为对您重要的文件编写规则，例如 `.github/`、`CLAUDE.md`、`AGENTS.md`、`.mcp.json` 以及 `.claude/` 的其余部分。

### 谁必须审批对受保护文件的更改

Approvals 要求的人可以是 Pull Request 的作者：

| Pull Request | 谁必须审批 |
| :- | :- |
| 有规则覆盖某个更改的文件 | 通常至少需要一个具有写入权限的人。该人可以是 Pull Request 的作者，且不是 bot |
| 没有规则覆盖任何更改的文件 | 可能需要一个人（可以是作者），也可能不需要任何人 |
| 在[观察模式](https://code.claude.com/docs/zh-CN/approvals-pull-requests.md#approvals-in-observe-mode)下更改了策略或分类器文件 | 可能需要一个人，可以是作者 |

### Claude 何时审批对受保护文件的更改

只有当有规则适用于受保护文件、且每条适用的规则都指定 `approves` 时，Claude 才会审批更改该文件的 Pull Request。基于 `"**"` 的规则可以计入。对于更改了 `.gitattributes` 文件、符号链接或子模块指针的 Pull Request，无论规则如何规定，Claude 都不会审批。

## 策略文件的大小和数量限制

请将每个文件保持在以下限制之内，否则 Approvals 会拒绝它：

| 项目 | 限制 |
| :- | :- |
| 单个策略文件 | 448 KiB |
| 仓库中的策略文件 | 1,024 个文件，总计 8 MiB。超过任一限制，Approvals 将不读取其中任何文件 |
| 一个文件中的规则 | 512 |
| 一个文件中的 locals | 512 |
| 一条规则中的模式，不计 `"!"` 项 | 256 |
| 覆盖 | 每条规则 16 个，每个文件 128 个 |
| 任何列表中的条目 | 1,024 |
| `purpose` | 1,000 个字符 |
| 一个条件 | 4,096 个字符，八层括号 |
| 数字形式的 `seats`，或集合的 `count` | 16 |
| 一个 `seats` 中的集合 | 8 |
| `resolution` 下的集合 | 16 |
| 一个列表中，或一个 `seats` 所有集合中的团队 | 10 |
| 一个 `use` 下的名称 | 8 |
| 一个文件指定的自定义分类器 | 16 |
| 一个文件请求的不同分类器运行 | 32 |

Pull Request 也有限制。超过这些限制时，GitHub 会将 **Claude Approvals** 检查列为 **Action required**：

* **更改的文件**：2,000 个或更多。检查的标题显示 **Change too large to evaluate**。请拆分该 Pull Request。
* **提交**：超过 250 个，且某个集合使用 `"!pr/authors"` 排除了作者。标题显示 **Too many commits to tell who wrote it**。请让作者将其压缩为 250 个或更少的提交并强制推送。

## 相关资源

* [设置 Claude Approvals](https://code.claude.com/docs/zh-CN/approvals-setup.md)：启用观察模式、强制执行策略，并将 **Claude Approvals** 检查设为必需
* [Claude Approvals 故障排除](https://code.claude.com/docs/zh-CN/approvals-troubleshooting.md)：修复被检查拒绝的策略文件
* [Claude Approvals 分类器](https://code.claude.com/docs/zh-CN/approvals-classifiers.md)：条件可以读取的分类器，以及如何定义您自己的分类器
