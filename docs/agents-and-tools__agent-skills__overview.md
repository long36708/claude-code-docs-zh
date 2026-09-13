---
title: Agent Skills
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview
description: Agent Skills 是扩展 Claude 功能的模块化能力。每个 Skill 都打包了指令、元数据和可选资源（脚本、模板），Claude 会在相关时自动使用它们。
---

<Note>
  要了解"zero data retention"（零数据保留），即 ZDR 如何适用于此功能，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。
</Note>

## 为什么使用 Skills

Skills 是可复用的、基于文件系统的资源，为 Claude 提供特定领域的专业知识：工作流程、上下文和最佳实践，将通用智能体转变为专家。与提示（用于一次性任务的对话级指令）不同，Skills 按需加载，因此您无需在不同对话中重复相同的指导。

**主要优势：**

* **让 Claude 专业化：** 为特定领域的任务定制能力
* **减少重复：** 创建一次，自动使用
* **组合能力：** 组合多个 Skills 以完成复杂的多步骤任务

<Note>
  有关 Agent Skills 的架构和实际应用的更多信息，请参阅工程博客文章 [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)。
</Note>

## 使用 Skills

Anthropic 为常见的文档任务（PowerPoint、Excel、Word、PDF）提供预构建的 Agent Skills，您也可以创建自己的自定义 Skills。两者的工作方式相同：一旦某个 Skill 在您的环境中可用，Claude 就会在与您的请求相关时自动使用它。

**预构建 Agent Skills** 可在 claude.ai、Claude API、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 和 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上使用。在 Microsoft Foundry 上，Agent Skills 需要 [Hosted on Anthropic 部署](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry#additional-features-not-supported-when-hosted-on-azure)。完整列表请参阅[可用的 Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview#available-skills)。

**自定义 Skills** 让您可以打包领域专业知识和组织知识。它们可在 Claude 的各个产品中使用：在 Claude Code 中创建、通过 Claude API 上传，或在 claude.ai 设置中添加。在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 和 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上，通过 Skills API 上传自定义 Skills。

<Note>
  **开始使用：**

  * 对于预构建 Agent Skills：请参阅[快速入门教程](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/quickstart)，开始在 API 中使用 PowerPoint、Excel、Word 和 PDF Skills
  * 对于自定义 Skills：请参阅 [Agent Skills Cookbook](https://platform.claude.com/cookbook/skills-notebooks-01-skills-introduction)，了解如何创建您自己的 Skills
</Note>

## Skills 的工作原理

Skills 利用 Claude 的虚拟机环境来提供仅靠提示无法实现的能力。Claude 在具有文件系统访问权限的虚拟机中运行，这使得 Skills 可以作为包含指令、可执行代码和参考资料的目录存在，其组织方式就像您为新团队成员编写的入职指南。

这种基于文件系统的架构实现了 **"progressive disclosure"（渐进式披露）：** Claude 根据需要分阶段加载信息，而不是预先消耗上下文。

Skills 可以包含三种类型的内容，每种在不同的时间加载：

### 第 1 级：元数据（始终加载）

Skill 的 YAML frontmatter 提供发现信息：

```yaml
---
name: pdf-processing
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---
```

Claude 在启动时加载这些元数据，并将其包含在 "system prompt"（系统提示）中。`description` 是 Claude 在判断是否触发该 Skill 时用来与您的请求进行匹配的内容，因此它必须同时说明该 Skill 的功能以及何时使用它。这种轻量级方法意味着您可以安装许多 Skills 而不会产生上下文开销：在 Skill 被触发之前，只有其名称和描述占用上下文。

### 第 2 级：指令（触发时加载）

SKILL.md 的主体包含程序性知识：工作流程、最佳实践和指导：

````markdown
# PDF Processing

## Quick start

Use pdfplumber to extract text from PDFs:

```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

For advanced form filling, see [FORMS.md](FORMS.md).
````

当您的请求与某个 Skill 的描述匹配时，Claude 会使用 bash 从文件系统读取 SKILL.md。只有在此时，这些内容才会进入 "context window"（上下文窗口）。

### 第 3 级：资源和代码（按需加载）

Skills 可以捆绑额外的材料：

* `pdf-processing/`

  * `SKILL.md`（主要指令）
  * `FORMS.md`（表单填写指南）
  * `REFERENCE.md`（详细的 API 参考）
  * `scripts/`
    * `fill_form.py`（实用脚本）

**指令：** 包含专门指导和工作流程的额外 markdown 文件（FORMS.md、REFERENCE.md）

**代码：** Claude 通过 bash 运行的可执行脚本（fill\_form.py、validate.py），提供确定性操作而无需将其代码加载到上下文中

**资源：** 参考资料，例如数据库模式、API 文档、模板或示例

Claude 仅在这些文件被引用时才访问它们。文件系统模型意味着每种内容类型各有所长：指令用于灵活指导，代码用于可靠性，资源用于事实查询。

| 级别              | 加载时机       | 令牌成本               | 内容                                             |
| --------------- | ---------- | ------------------ | ---------------------------------------------- |
| **第 1 级：元数据**   | 始终（启动时）    | 每个 Skill 约 100 个令牌 | YAML frontmatter 中的 `name` 和 `description`     |
| **第 2 级：指令**    | Skill 被触发时 | 少于 5k 个令牌          | 包含指令和指导的 SKILL.md 主体                           |
| **第 3 级及以上：资源** | 按需         | 访问前为零              | 捆绑的文件。参考文件在被读取时加载到上下文中。脚本通过 bash 运行，只有其输出进入上下文 |

渐进式披露确保在任何给定时间只有相关内容占用上下文窗口。

### Skills 架构

Skills 在代码执行环境中运行，Claude 在其中拥有文件系统访问权限、bash 命令和代码执行能力。Skills 作为虚拟机上的目录存在，Claude 使用与您在自己计算机上浏览文件时相同的 bash 命令与它们交互。

![Agent Skills Architecture（Agent Skills 架构）- 展示 Skills 如何与智能体的配置和虚拟机集成](https://platform.claude.com/docs/images/agent-skills-architecture.png)

**Claude 如何访问 Skill 内容：**

当某个 Skill 被触发时，Claude 使用 bash 从文件系统读取 SKILL.md，将其指令带入上下文窗口。如果这些指令引用了其他文件（例如 FORMS.md 或数据库模式），Claude 也会使用额外的 bash 命令读取这些文件。当指令提到可执行脚本时，Claude 通过 bash 运行它们并仅接收输出（脚本代码本身永远不会进入上下文）。

**这种架构带来的能力：**

* **按需文件访问：** Claude 只读取每个任务所需的文件。一个 Skill 可以包含数十个参考文件，但如果您的任务只需要销售数据模式，Claude 就只加载那一个文件。其余文件保留在文件系统上，不消耗任何令牌。
* **高效的脚本执行：** 当 Claude 运行 `validate_form.py` 时，脚本的代码永远不会加载到上下文窗口中。只有其输出（例如 "Validation passed" 或特定的错误消息）消耗令牌，这使得脚本比让 Claude 即时生成等效代码高效得多。
* **捆绑内容没有实际限制：** 文件在被访问之前不消耗上下文，因此 Skills 可以包含全面的 API 文档、大型数据集或大量示例。未使用的捆绑内容不会产生上下文开销。

### 示例：加载 PDF 处理 Skill

以下是 Claude 如何加载和使用前面示例中的自定义 `pdf-processing` Skill（而非预构建的 `pdf` Skill）：

1. **启动：** 系统提示包含：`pdf-processing - Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.`
2. **用户请求：** "提取这个 PDF 中的文本并进行总结"
3. **Claude 调用：** `bash: cat pdf-processing/SKILL.md` → 指令加载到上下文中
4. **Claude 判断：** 不需要填写表单，因此不读取 FORMS.md
5. **Claude 执行：** 使用 SKILL.md 中的指令完成任务

![Skills 加载到 context window（上下文窗口）- 展示 Skill 元数据和内容的渐进式加载](https://platform.claude.com/docs/images/agent-skills-context-window.png)

## Skills 在哪里可用

Skills 可在 Claude 的各个智能体产品中使用：

<Note>
  在以下所有章节中，Claude Platform on AWS 和 Microsoft Foundry 继承与 Claude API 相同的 Skills 行为。
</Note>

### Claude API

Claude API 同时支持预构建 Agent Skills 和自定义 Skills。两者的工作方式完全相同：在 `container` 参数中指定相关的 `skill_id`，并配合[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)使用。

**前提条件：** 通过 API 使用 Skills 需要[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)，Skills 在其容器中运行。

通过引用 `skill_id`（`pptx`、`xlsx`、`docx` 或 `pdf`）使用预构建 Agent Skills，或通过 Skills API（`/v1/skills` 端点）创建并上传您自己的 Skills。自定义 Skills 在整个工作区范围内共享：所有工作区成员都可以访问它们。

API 上的 Skills 在沙盒容器中运行，没有网络访问权限，也不能在运行时安装软件包。详情请参阅[限制和约束](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview#limitations-and-constraints)。

要了解更多信息，请参阅[通过 API 使用 Agent Skills](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide)。

### Claude Code

[Claude Code](https://code.claude.com/docs/en/overview) 支持自定义 Skills。预构建的文档 Skills（PowerPoint、Excel、Word、PDF）在 Claude Code 中不可用，但开源的 [Claude API skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill) 随其一同捆绑提供。请参阅 Claude Code 附带的[内置命令和 Skills](https://code.claude.com/docs/en/commands) 完整列表。

**自定义 Skills：** 将 Skills 创建为包含 SKILL.md 文件的目录。Claude 会自动发现并使用它们。

Claude Code 中的自定义 Skills 基于文件系统，不需要通过 API 上传：将它们放在 `~/.claude/skills/`（个人）或 `.claude/skills/`（项目）中。

要了解更多信息，请参阅[在 Claude Code 中使用 Skills](https://code.claude.com/docs/en/skills)。

### claude.ai

[claude.ai](https://claude.ai) 同时支持预构建 Agent Skills 和自定义 Skills。

**预构建 Agent Skills：** 这些 Skills 在您创建文档时处于激活状态。Claude 无需任何设置即可使用它们。

**自定义 Skills：** 通过"设置 > 功能"以 zip 文件形式上传您自己的 Skills。在[启用代码执行](https://support.claude.com/en/articles/12111783-create-and-edit-files-with-claude)的 Pro、Max、Team 和 Enterprise 套餐中可用。自定义 Skills 属于每个用户个人。它们不会在整个组织范围内共享，也无法由管理员集中管理。

要了解有关在 claude.ai 中使用 Skills 的更多信息，请参阅 Claude 帮助中心的以下资源：

* [什么是 Skills？](https://support.claude.com/en/articles/12512176-what-are-skills)
* [在 Claude 中使用 Skills](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
* [如何创建自定义 Skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)
* [使用 Skills 教会 Claude 您的工作方式](https://support.claude.com/en/articles/12580051-teach-claude-your-way-of-working-using-skills)

## Skill 结构

每个 Skill 都需要一个带有 YAML frontmatter 的 `SKILL.md` 文件：

```markdown
---
name: your-skill-name
description: Brief description of what this Skill does and when to use it
---

# Your Skill Name

## Instructions
[Clear, step-by-step guidance for Claude to follow]

## Examples
[Concrete examples of using this Skill]
```

**必填字段：** `name` 和 `description`

**字段要求：**

`name`：

* 最多 64 个字符
* 只能包含小写字母、数字和连字符
* 不能包含 XML 标签
* 不能包含保留字："anthropic"、"claude"

`description`：

* 不能为空
* 最多 1024 个字符
* 不能包含 XML 标签

`description` 必须同时包含该 Skill 的功能以及 Claude 应何时使用它。完整的编写指导请参阅 [Skill 编写最佳实践](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices)。

## 安全注意事项

仅使用来自可信来源的 Skills：您自己创建的或从 Anthropic 获得的。Skills 通过指令和代码赋予 Claude 新的能力，这也意味着恶意 Skill 可能会引导 Claude 以与该 Skill 声明用途不符的方式调用工具或执行代码。

<Warning>
  如果您必须使用来自不可信或未知来源的 Skill，请格外谨慎，并在使用前对其进行彻底审查。根据 Claude 在执行该 Skill 时拥有的访问权限，恶意 Skills 可能导致数据外泄、未经授权的系统访问或其他安全风险。
</Warning>

**主要安全注意事项：**

* **彻底审查：** 检查 Skill 中捆绑的所有文件：SKILL.md、脚本、图像和其他资源。留意异常模式，例如意外的网络调用、文件访问模式或与 Skill 声明用途不符的操作
* **外部来源存在风险：** 从外部 URL 获取数据的 Skills 风险尤其高，因为获取的内容可能包含恶意指令。即使是可信的 Skills，如果其外部依赖随时间发生变化，也可能被攻破
* **工具滥用：** 恶意 Skills 可能以有害的方式调用工具（文件操作、bash 命令、代码执行）
* **数据暴露：** 能够访问敏感数据的 Skills 可能被设计为向外部系统泄露信息
* **像安装软件一样对待：** 在将 Skills 集成到能够访问敏感数据或关键操作的生产系统时要特别小心

有关组织规模的治理、审核和部署指导，请参阅[面向企业的 Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/enterprise)。Claude Enterprise 组织还可以为在 claude.ai 和 Claude Cowork 中上传的自定义 Skills 开启 [Skill 内容扫描](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/enterprise#skill-content-scanning)。扫描不涵盖通过 Skills API 或 Claude Console 上传的 Skills。

## 可用的 Skills

### 预构建 Agent Skills

以下预构建 Agent Skills 可立即使用：

* **PowerPoint (pptx)：** 创建演示文稿、编辑幻灯片、分析演示文稿内容
* **Excel (xlsx)：** 创建电子表格、分析数据、生成带图表的报告
* **Word (docx)：** 创建文档、编辑内容、格式化文本
* **PDF (pdf)：** 生成格式化的 PDF 文档和报告

这些 Skills 可在 Claude API、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)、[Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 和 claude.ai 上使用。请参阅[快速入门教程](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/quickstart)，开始在 API 中使用它们。

### 开源 Skills

Anthropic 还在 [skills 代码仓库](https://github.com/anthropics/skills)中发布开源 Skills：

* **[Claude API skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill)：** 为 Claude 提供最新的 API 参考资料、SDK 文档以及八种编程语言的最佳实践。随 Claude Code 捆绑提供，也可从 skills 代码仓库安装。

### 自定义 Skills 示例

有关自定义 Skills 的完整示例，请参阅 [Skills cookbook](https://platform.claude.com/cookbook/skills-notebooks-01-skills-introduction)。

## 数据保留

Agent Skills 不在 ZDR 安排的涵盖范围内。Skill 定义和执行数据按照 Anthropic 的标准数据保留政策进行保留。

有关所有功能的 ZDR 资格，请参阅 [API 和数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。

有关 Skills API 操作的审计日志，请参阅"通过 API 使用 Agent Skills"中的[审计日志](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide#audit-logging)。

## 限制和约束

在以下小节中，Claude Platform on AWS 和 Microsoft Foundry 遵循与 Claude API 相同的限制。

### 跨平台可用性

**自定义 Skills 不会跨平台同步**。上传到一个平台的 Skills 不会自动在其他平台上可用：

* 上传到 claude.ai 的 Skills 必须单独上传到 API
* 通过 API 上传的 Skills 在 claude.ai 上不可用
* Claude Code Skills 基于文件系统，与 claude.ai 和 API 均相互独立

请为您想要使用 Skills 的每个平台分别管理和上传 Skills。

### 共享范围

Skills 根据您使用它们的位置具有不同的共享模型：

* **claude.ai：** 仅限个人用户。每个团队成员必须单独上传。
* **Claude API：** 工作区范围。所有工作区成员都可以访问已上传的 Skills。
* **Claude Code：** 个人（`~/.claude/skills/`）或基于项目（`.claude/skills/`）。也可以通过 Claude Code 插件共享。

claude.ai 不支持自定义 Skills 的集中管理员管理或组织范围分发。

### 运行时环境约束

您的 Skill 可用的确切运行时环境取决于您使用它的产品平台。

* **claude.ai：**
  * **网络访问权限不一：** 根据用户/管理员设置，Skills 可能拥有完全、部分或无网络访问权限。更多详情请参阅[创建和编辑文件](https://support.claude.com/en/articles/12111783-create-and-edit-files-with-claude#h_6b7e833898)支持文章。

* **Claude API：**

  * **无网络访问：** Skills 无法进行外部 API 调用或访问互联网。
  * **无运行时软件包安装：** 只有预安装的软件包可用。您无法在执行期间安装新软件包。
  * **仅限预配置的依赖项：** 请查看[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)文档以获取可用软件包列表。

* **Claude Code：**

  * **完全网络访问：** Skills 拥有与用户计算机上任何其他程序相同的网络访问权限。
  * **不建议全局安装软件包：** Skills 应仅在本地安装软件包，以避免干扰用户的计算机。

请规划您的 Skills 以在这些约束内工作。

## 后续步骤

<CardGroup cols={2}>
  <Card title="在 API 中开始使用 Agent Skills" icon="graduation-cap" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/quickstart">
    了解如何在 10 分钟内使用 Agent Skills 通过 Claude API 创建文档。
  </Card>

  <Card title="通过 API 使用 Agent Skills" icon="code" href="https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide">
    了解如何使用 Agent Skills 通过 API 扩展 Claude 的能力。
  </Card>

  <Card title="在 Claude Code 中使用 Skills" icon="terminal" href="https://code.claude.com/docs/en/skills">
    在 Claude Code 中创建和管理自定义 Skills。
  </Card>

  <Card title="Skill 编写最佳实践" icon="lightbulb" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices">
    了解如何编写 Claude 能够发现并成功使用的高效 Skills。
  </Card>
</CardGroup>
