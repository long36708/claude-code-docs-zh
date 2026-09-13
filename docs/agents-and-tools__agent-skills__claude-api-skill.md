---
title: Claude API 技能
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill
description: 一个开源的 Agent Skill，为 Claude 提供最新的 API 参考资料、SDK 文档，以及使用 Claude API 和 Claude Managed Agents 构建应用程序的最佳实践。
---

`claude-api` 技能是一个开源的 [Agent Skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview)（智能体技能），为 Claude 提供详细、最新的参考资料，用于在两个 Anthropic 平台上构建应用程序：

* **Messages API：** 用于单次请求、流式聊天、工具使用、批处理、提示缓存、结构化输出和自定义智能体循环的主要平台。
* **Claude Managed Agents（测试版）：** 一个由 Anthropic 托管的平台，用于服务器管理的有状态智能体，具备 Anthropic 托管的工具执行、持久化智能体配置和每会话沙箱。

它为 Messages API 和 Managed Agents 涵盖八种编程语言：Python、TypeScript、C#、Go、Java、PHP、Ruby 和 cURL。

该技能随 [Claude Code](https://code.claude.com/docs/en/overview) 捆绑提供，也可在开源的 [Anthropic 技能仓库](https://github.com/anthropics/skills)中获取，您可以将其安装在任何支持 Agent Skills 的环境中。

该技能使用 [progressive disclosure](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview#how-skills-work)（渐进式披露）来保持上下文高效：Claude 只加载与您项目的语言、平台（Messages API 或 Managed Agents）以及当前具体任务（工具使用、流式传输、批处理等）相关的文档，而不是一次性加载所有内容。

## 该技能提供的内容

触发后，该技能为 Claude 提供：

**针对 Messages API：**

* **特定语言的 SDK 文档：** 针对您项目语言的安装、快速入门、常见模式和错误处理
* **工具使用指南：** 函数调用的特定语言示例和[概念基础](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)，包括可用时的测试版工具运行器
* **流式传输模式：** 构建聊天 UI 和处理增量显示的实现细节
* **批处理：** 以 50% 成本进行离线批处理
* **提示缓存：** 前缀稳定性设计、断点放置和静默失效因素审计
* **模型迁移：** 迁移到较新 Claude 模型的分步指南（包括 [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-4-8-to-claude-opus-5) 和 [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide) 上的破坏性变更和行为变化）
* **当前模型信息：** 模型 ID、上下文窗口大小和定价
* **常见陷阱：** 关于在与 API 集成时避免常见错误的详细指南

**针对 Managed Agents（测试版）：**

* **入门流程：** 一个访谈驱动的演练，用于从零开始设置新的 Managed Agent，可通过 `/claude-api managed-agents-onboard` 子命令使用
* **特定语言的 Managed Agents 文档：** 针对 Python、TypeScript、C#、Go、Java、PHP、Ruby 和 cURL 的创建持久化智能体、启动会话、流式传输事件和处理工具确认
* **客户端模式：** 无损流重连、`processed_at` 排队/已处理门控、中断处理、文件挂载注意事项和凭证处理
* **部署限制：** Managed Agents 仅在 Claude API 和 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上可用（不在 Amazon Bedrock、Google Cloud 或 Microsoft Foundry 上提供）。该技能会将其他部署引导至 Messages API 和工具使用。

## 技能何时激活

该技能通过两种方式激活：

**自动激活**发生在以下情况：

* 您的代码导入了 Anthropic SDK（Python 为 `anthropic`，TypeScript/JavaScript 为 `@anthropic-ai/sdk`）
* 您请求 Claude 帮助使用 Claude API、Anthropic SDK 或 Managed Agents 构建、调试或优化某些内容
* 您在文件中添加、修改或调整 Claude 功能（提示缓存、自适应思考、压缩、工具使用、批处理、文件、引用、记忆）或模型引用

**手动调用**：在任何已安装该技能的环境中输入 `/claude-api`（可附带可选的子命令或文字描述）。

该技能不会为一般编程任务、ML/数据科学工作或导入其他 AI SDK（如 OpenAI）的代码激活。

## 支持的语言

该技能通过检查项目文件（例如，Python 的 `requirements.txt`、TypeScript 的 `tsconfig.json`、Go 的 `go.mod`）自动检测您项目的语言，并加载相应的文档。

| 语言         | Messages API SDK | 工具运行器  | Managed Agents |
| ---------- | ---------------- | ------ | -------------- |
| Python     | 是                | 是（测试版） | 是（测试版）         |
| TypeScript | 是                | 是（测试版） | 是（测试版）         |
| C#         | 是                | 是（测试版） | 是（测试版）         |
| Go         | 是                | 是（测试版） | 是（测试版）         |
| Java       | 是                | 是（测试版） | 是（测试版）         |
| PHP        | 是                | 是（测试版） | 是（测试版）         |
| Ruby       | 是                | 是（测试版） | 是（测试版）         |
| cURL       | 是                | 不适用    | 是（测试版）         |

如果您的项目使用多种语言，Claude 会询问适用哪一种。对于不支持的语言（Rust、Swift、C++），该技能提供 cURL/原始 HTTP 示例。

## 如何使用该技能

### 在 Claude Code 中（捆绑提供）

该技能随 [Claude Code](https://code.claude.com/docs/en/overview) 一起提供，无需安装。当您请求 Claude 帮助使用 Claude API 构建某些内容时，或者当您的项目已经导入了 Anthropic SDK 时，该技能会自动激活。

您也可以直接调用它：

```text wrap
/claude-api
```

有关捆绑技能在 Claude Code 中如何工作的更多信息，请参阅 [Claude Code 技能文档](https://code.claude.com/docs/en/skills#bundled-skills)。

### 从技能仓库安装

该技能的源代码可在 [Anthropic 技能仓库](https://github.com/anthropics/skills)中获取。您可以使用 `npx` 命令安装它：

```bash
npx skills add https://github.com/anthropics/skills --skill claude-api
```

或者将其作为 [Claude Code 插件](https://code.claude.com/docs/en/plugins)安装：

```text wrap
/plugin marketplace add anthropics/skills
/plugin install claude-api@anthropic-agent-skills
```

## 迁移到更新的 Claude 模型

Claude API 技能可以在整个代码库中执行 Claude 模型迁移。使用 `/claude-api migrate` 直接调用它：

```text wrap
/claude-api migrate this project to claude-opus-5
```

您也可以预先传入特定范围，以跳过范围确认问题：

```text wrap
/claude-api migrate everything under src/ to claude-opus-5
/claude-api migrate apps/api.py and apps/worker.py to claude-opus-5
```

当范围不明确时（例如，仅输入 `/claude-api migrate to claude-opus-5`），该技能会在编辑任何文件之前要求您在整个工作目录、特定子目录或明确的文件列表之间进行选择。这适用于 Messages API 和 Managed Agents 的调用方。

该技能处理：

* **模型 ID 替换**，包括所有支持语言中的类型化 SDK 常量（`Model.CLAUDE_OPUS_4_8` → `Model.CLAUDE_OPUS_5`），并在编辑前将每个文件分类为调用方、模型定义方或不透明字符串引用
* **云平台检测**，保留平台特定的模型 ID 格式（例如，Amazon Bedrock 上的 `anthropic.` 前缀），并跳过对合作伙伴运营平台上不可用功能的更改
* **破坏性参数变更**，例如为 Claude Opus 4.8 和 Claude Opus 4.7 移除 `temperature`、`top_p` 和 `top_k`，并将 `thinking: {type: "enabled", budget_tokens: N}` 转换为 `thinking: {type: "adaptive"}`
* **预填充替换**，在适用的情况下将助手消息预填充模式转换为[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)
* **Beta 请求头清理**，移除目标模型不需要的 beta 请求头（例如，`effort-2025-11-24`、`fine-grained-tool-streaming-2025-05-14`、`interleaved-thinking-2025-05-14`），并从 `client.beta.messages.create` 切换回 `client.messages.create`
* **努力程度校准**，为目标模型推荐 `output_config.effort` 起始值（例如，Claude Opus 5 上的默认值 `high`，以及 Claude Opus 4.8 和 Claude Opus 4.7 上编码和智能体用例的 `xhigh`）
* **提示行为调优**，标记在目标模型上可能表现不同的长度控制、工具触发、子智能体和指令遵循提示
* **静默默认值处理**，当在 Claude Opus 4.8 和 Claude Opus 4.7 上向用户展示推理时，重新启用思考摘要（`thinking.display: "summarized"`）
* **拒绝回退配置**，在读取响应内容之前添加 `stop_reason: "refusal"` 处理，并在目标为 Claude Fable 5.1、Claude Fable 5 或 Claude Opus 5 时设置[回退重试路径](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)（服务器端 `fallbacks` 参数，通常使用其 `"default"` 模式、SDK 拒绝回退中间件或回退额度重试），并更新针对早期预览形态编写的回退代码

在编辑过程中，该技能会内联解释每项更改及其动机。完成后，它会生成一份需要手动验证的项目清单（通常包括集成测试、长度控制提示调优以及成本/速率限制重新基准化）。

有关该技能应用的特定模型更改的完整列表，请参阅[从 Claude Opus 4.8 迁移到 Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-4-8-to-claude-opus-5) 和[迁移到 Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/migration-guide)。

## 设置 Managed Agent

要从零开始搭建新的 Managed Agent，请调用 `managed-agents-onboard` 子命令：

```text wrap
/claude-api managed-agents-onboard
```

该技能会运行一个访谈，引导您了解 Managed Agents 的思维模型（Agent 配置与 Session 的区别），生成智能体配置模板，配置环境和工具，设置会话循环，并为您的语言生成可运行的代码。该技能还涵盖强制性的 **Agent（一次）→ Session（每次运行）** 流程：`model`、`system` 和 `tools` 位于智能体上，而绝不位于会话上，并且智能体应只创建一次并通过 ID 引用。

Managed Agents 需要 `managed-agents-2026-04-01` beta 请求头，SDK 会为所有 `client.beta.agents.*`、`client.beta.environments.*`、`client.beta.sessions.*` 和 `client.beta.vaults.*` 调用自动设置该请求头。

## 使用示例

以下是该技能帮助 Claude 处理的任务示例：

**构建聊天应用程序：**

```text wrap
Build a streaming chat UI with the Claude API in TypeScript
```

**迁移现有项目：**

```text wrap
/claude-api migrate this codebase to claude-opus-5 and re-tune effort
```

**为新的 Managed Agent 进行入门设置：**

```text wrap
/claude-api managed-agents-onboard
```

在每种情况下，该技能都会加载相关的特定语言文档，并使用当前的 API 模式和最佳实践引导 Claude 完成实现。

## 后续步骤

<CardGroup cols={2}>
  <Card title="Agent Skills 概述" icon="graduation-cap" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview">
    了解 Agent Skills 的工作原理和渐进式披露模型
  </Card>

  <Card title="客户端 SDK" icon="code" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview">
    浏览所有支持语言的官方 Anthropic SDK
  </Card>

  <Card title="技能仓库" icon="github-logo" href="https://github.com/anthropics/skills">
    在 GitHub 上探索公开的 Anthropic 技能仓库
  </Card>
</CardGroup>
