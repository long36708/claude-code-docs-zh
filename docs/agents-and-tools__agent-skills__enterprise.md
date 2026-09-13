---
title: 面向企业的 Skills
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/enterprise
description: 在企业规模部署 Agent Skills 的治理、安全审查、评估和组织指南。
---

本指南面向需要在整个组织范围内治理 Agent Skills 的企业管理员和架构师。它涵盖了如何大规模审查、评估、部署和管理 Skills。有关编写指南，请参阅[最佳实践](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices)。有关架构详情，请参阅 [Skills 概述](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview)。

## 安全审查与审核

在企业中部署 Skills 需要回答两个不同的问题：

1. **Skills 总体上是否安全？** 请参阅概述中的[安全注意事项](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview#security-considerations)部分，了解平台级安全详情。
2. **我如何审核某个特定的 Skill？** 请使用以下风险评估和审查清单。

### 风险等级评估

在批准部署之前，请根据以下风险指标评估每个 Skill：

| 风险指标      | 需要关注的内容                                   | 关注级别                      |
| --------- | ----------------------------------------- | ------------------------- |
| 代码执行      | Skill 目录中的脚本（`*.py`、`*.sh`、`*.js`）        | 高：脚本以完整的环境访问权限运行          |
| 指令操纵      | 要求忽略安全规则、向用户隐藏操作或有条件地改变 Claude 行为的指令      | 高：可能绕过安全控制                |
| MCP 服务器引用 | 引用 MCP 工具的指令（`ServerName:tool_name`）      | 高：将访问范围扩展到 Skill 本身之外     |
| 网络访问模式    | URL、API 端点、`fetch`、`curl` 或 `requests` 调用 | 高：潜在的数据外泄途径               |
| 硬编码凭据     | Skill 文件或脚本中的 API 密钥、令牌或密码                | 高：机密信息暴露在 Git 历史记录和上下文窗口中 |
| 文件系统访问范围  | Skill 目录之外的路径、宽泛的 glob 模式、路径遍历（`../`）     | 中：可能访问非预期的数据              |
| 工具调用      | 指示 Claude 使用 bash、文件操作或其他工具的指令            | 中：审查所执行的操作                |

### 审查清单

在部署任何来自第三方或内部贡献者的 Skill 之前，请完成以下步骤：

1. **阅读 Skill 目录中的所有内容。** 审查 SKILL.md、所有被引用的 markdown 文件，以及任何捆绑的脚本或资源。
2. **验证脚本行为与其声明的用途一致。** 在沙盒环境中运行脚本，并确认输出与 Skill 的描述相符。
3. **检查是否存在对抗性指令。** 查找那些要求 Claude 忽略安全规则、向用户隐藏操作、通过响应外泄数据或根据特定输入改变行为的指令。
4. **检查是否存在外部 URL 获取或网络调用。** 在脚本和指令中搜索网络访问模式（`http`、`requests.get`、`urllib`、`curl`、`fetch`）。
5. **验证没有硬编码凭据。** 检查 Skill 文件中是否存在 API 密钥、令牌或密码。凭据应使用环境变量或安全的凭据存储，绝不应出现在 Skill 内容中。
6. **识别 Skill 指示 Claude 调用的工具和命令。** 列出所有 bash 命令、文件操作和工具引用。当一个 Skill 同时使用文件读取和网络工具时，请考虑其组合风险。
7. **确认重定向目标。** 如果 Skill 引用了外部 URL，请验证它们指向预期的域名。
8. **验证没有数据外泄模式。** 查找那些读取敏感数据然后将其写入、发送或编码以便向外部传输的指令，包括通过 Claude 的对话响应进行传输。

<Warning>
  切勿在未经全面审计的情况下部署来自不受信任来源的 Skills。恶意 Skill 可以指示 Claude 执行任意代码、访问敏感文件或向外部传输数据。请以与在生产系统上安装软件同等的严格程度对待 Skill 的安装。
</Warning>

### Skill 内容扫描

Claude Enterprise 组织可以为 claude.ai 和 Claude Cowork 中的自定义 Skills 开启自动安全扫描。在 [claude.ai > Organization settings > Skills](https://claude.ai/admin-settings/skills) 中开启 **Skill and plugin security scanning**（Skill 和插件安全扫描）后，成员随后在 claude.ai 或 Cowork 中上传或编辑的 Skills 将被扫描，以检测恶意行为的迹象，例如隐藏的代码执行、将您的数据发送到外部服务，或篡改 Claude 安全防护措施的指令。未通过扫描或扫描尚未完成的 Skill 将被阻止使用。带有警告通过扫描的 Skill 仍可使用，但会显示警示通知。如果您的组织可以使用扫描功能，请将其开启。它是对[审查清单](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/enterprise#review-checklist)的补充，而非替代。

扫描不涵盖 Claude API。您通过 Skills API（`/v1/skills`）上传的 Skills（包括从 Claude Console 上传的）不会被扫描，因此对于 API 部署，请依赖审查清单和[版本固定](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/enterprise#versioning-strategy)。扫描也不适用于您开启该功能时组织中已存在的 Skills，也不适用于具有某些数据处理配置的组织，例如客户管理的加密密钥（CMEK）、零数据保留（ZDR）或 HIPAA 就绪配置。有关设置步骤、排除项和结果类型，请参阅 Claude 帮助中心中的 [Get started with skill and plugin scanning](https://support.claude.com/en/articles/15927065-get-started-with-skill-and-plugin-scanning)。

## 部署前评估 Skills

如果 Skills 触发不正确、与其他 Skills 冲突或提供的指令质量不佳，可能会降低智能体的性能。请要求在任何生产部署之前进行评估。

### 评估内容

在部署任何 Skill 之前，为以下维度建立审批关卡：

| 维度    | 衡量内容                           | 失败示例                             |
| ----- | ------------------------------ | -------------------------------- |
| 触发准确性 | Skill 是否针对正确的查询激活，并对无关查询保持不激活？ | 每次提到电子表格时 Skill 都会触发，即使用户只是想讨论数据 |
| 隔离行为  | Skill 单独运行时是否正常工作？             | Skill 引用了其目录中不存在的文件              |
| 共存性   | 添加此 Skill 是否会降低其他 Skills 的表现？  | 新 Skill 的描述过于宽泛，抢占了现有 Skills 的触发 |
| 指令遵循  | Claude 是否准确遵循 Skill 的指令？       | Claude 跳过验证步骤或使用了错误的库            |
| 输出质量  | Skill 是否产生正确、有用的结果？            | 生成的报告存在格式错误或数据缺失                 |

### 评估要求

要求 Skill 作者为每个 Skill 提交包含 3–5 个代表性查询的评估套件，涵盖 Skill 应该触发、不应触发以及模糊边界情况。要求在您的组织所使用的各个模型（Haiku、Sonnet、Opus）上进行测试，因为 Skill 的有效性因模型而异。

有关构建评估的详细指南，请参阅最佳实践中的[评估与迭代](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices#evaluation-and-iteration)。有关通用评估方法，请参阅[开发测试用例](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests)。

### 使用评估进行生命周期决策

评估结果会提示何时采取行动：

* **触发准确性下降：** 更新 Skill 的描述或指令
* **共存冲突：** 合并重叠的 Skills 或收窄描述
* **输出质量持续偏低：** 重写指令或添加验证步骤
* **多次更新后仍持续失败：** 弃用该 Skill

## Skill 生命周期管理

<Steps>
  <Step title="规划">
    识别重复性高、容易出错或需要专业知识的工作流。将这些工作流映射到组织角色，并确定哪些适合作为 Skills 的候选。
  </Step>

  <Step title="创建与审查">
    确保 Skill 作者遵循[最佳实践](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices)。要求使用[审查清单](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/enterprise#review-checklist)进行安全审查。要求在批准前提供评估套件。建立职责分离：Skill 作者不应担任自己的审查者。
  </Step>

  <Step title="测试">
    要求进行隔离评估（Skill 单独运行）以及与现有 Skills 一起的评估（共存测试）。在批准投入生产之前，验证触发准确性、输出质量，并确认您的活跃 Skill 集合中没有出现回归。
  </Step>

  <Step title="部署">
    通过 Skills API 上传以实现工作区范围的访问。有关上传和版本管理，请参阅[通过 API 使用 Skills](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide)。在您的内部注册表中记录该 Skill 的用途、负责人和版本。
  </Step>

  <Step title="监控">
    跟踪使用模式并收集用户反馈。随着工作流和模型的演进，定期重新运行评估以检测漂移或回归。目前 Skills API 不提供使用分析功能。请实现应用级日志记录，以跟踪请求中包含了哪些 Skills。
  </Step>

  <Step title="迭代或弃用">
    要求在推广新版本之前通过完整的评估套件。当工作流发生变化或评估分数下降时更新 Skills。当评估持续失败或工作流被淘汰时弃用 Skills。
  </Step>
</Steps>

## 大规模组织 Skills

### 召回限制

作为一般准则，请限制同时加载的 Skills 数量，以保持可靠的召回准确性。每个 Skill 的元数据（名称和描述）都会在系统提示中争夺注意力。如果活跃的 Skills 过多，Claude 可能无法选择正确的 Skill，或完全遗漏相关的 Skill。在添加 Skills 时，使用您的评估套件来衡量召回准确性，并在性能下降时停止添加。

请注意，API 请求每次最多支持 20 个 Skills（请参阅[通过 API 使用 Skills](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide)）。如果某个角色需要的 Skills 数量超过单个请求所支持的数量，请考虑将范围较窄的 Skills 合并为范围更广的 Skills，或根据任务类型将请求路由到不同的 Skill 集合。

### 从具体开始，之后再合并

鼓励团队从范围较窄、针对特定工作流的 Skills 开始，而不是宽泛的多用途 Skills。随着组织内模式的显现，将相关的 Skills 合并为基于角色的捆绑包。

<Tip>
  使用评估来决定何时合并。只有当合并后的 Skill 的评估确认其性能与所替代的各个单独 Skills 相当时，才将范围较窄的 Skills 合并为一个范围更广的 Skill。
</Tip>

**演进示例：**

* 开始：`formatting-sales-reports`、`querying-pipeline-data`、`updating-crm-records`
* 合并：`sales-operations`（当评估确认性能相当时）

### 命名与编目

在整个组织中使用一致的命名约定。最佳实践中的[命名约定](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices#naming-conventions)部分提供了格式指南。

为每个 Skill 维护一个内部注册表，包含：

* **用途：** Skill 支持的工作流
* **负责人：** 负责维护的团队或个人
* **版本：** 当前部署的版本
* **依赖项：** 所需的 MCP 服务器、软件包或外部服务
* **评估状态：** 最近一次评估的日期和结果

### 基于角色的捆绑包

按组织角色对 Skills 进行分组，使每个用户的活跃 Skill 集合保持聚焦：

* **销售团队：** CRM 操作、销售管道报告、提案生成
* **工程团队：** 代码审查、部署工作流、事件响应
* **财务团队：** 报告生成、数据验证、审计准备

每个基于角色的捆绑包应仅包含与该角色日常工作流相关的 Skills。

## 分发与版本控制

### 源代码控制

将 Skill 目录存储在 Git 中，以便进行历史跟踪、通过拉取请求进行代码审查以及实现回滚能力。每个 Skill 目录（包含 SKILL.md 和任何捆绑文件）都可以自然地映射到一个由 Git 跟踪的文件夹。

### 基于 API 的分发

Skills API 提供工作区范围的分发。通过 API 上传的 Skills 可供所有工作区成员使用。有关上传、版本控制和管理端点，请参阅[通过 API 使用 Skills](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide)。

### 版本控制策略

* **生产环境：** 将 Skills 固定到特定版本。如果您省略 `version`，请求将使用最新版本，因此工作区中任何人上传的新版本都会立即改变生产智能体所运行的内容。在推广新版本之前运行完整的评估套件。将每次更新视为需要完整安全审查的新部署。
* **开发与测试：** 使用最新版本在推广到生产环境之前验证更改。
* **回滚计划：** 保留上一个版本作为后备。如果新版本在生产环境中未通过评估，请立即回退到最后一个已知良好的版本。
* **完整性验证：** 计算已审查 Skills 的校验和，并在部署时进行验证。在您的 Skill 仓库中使用签名提交以确保来源可信。

### 跨平台注意事项

<Warning>
  自定义 Skills 不会跨平台同步。上传到 API 的 Skills 在 claude.ai 或 Claude Code 中不可用，反之亦然。每个平台都需要单独上传和管理。
</Warning>

将 Git 中的 Skill 源文件作为唯一可信来源进行维护。如果您的组织在多个平台上部署 Skills，请实现您自己的同步流程以保持一致。有关完整详情，请参阅[跨平台可用性](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview#cross-surface-availability)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="Agent Skills 概述" icon="book-open" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview">
    架构和平台详情
  </Card>

  <Card title="最佳实践" icon="lightbulb" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices">
    面向 Skill 创建者的编写指南
  </Card>

  <Card title="通过 API 使用 Skills" icon="code" href="https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide">
    以编程方式上传和管理 Skills
  </Card>
</CardGroup>
