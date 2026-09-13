---
title: Claude Managed Agents 概述
url: https://platform.claude.com/docs/zh-CN/managed-agents/overview
description: 预构建、可配置的智能体框架，运行在托管基础设施中。最适合长时间运行的任务和异步工作。
---

Anthropic 提供两种使用 Claude 进行构建的方式，每种方式适用于不同的使用场景：

|          | Messages API   | Claude Managed Agents    |
| -------- | -------------- | ------------------------ |
| **它是什么** | 直接的模型提示访问      | 预构建、可配置的智能体框架，运行在托管基础设施中 |
| **最适合**  | 自定义智能体循环和细粒度控制 | 长时间运行的任务和异步工作            |

Claude Managed Agents 提供了将 Claude 作为自主智能体运行所需的 harness（框架）和基础设施。您无需自行构建智能体循环、工具执行和运行时，即可获得一个完全托管的环境，Claude 可以在其中安全地读取文件、运行命令、浏览网页和运行代码。该框架支持内置的 prompt caching（提示缓存）、compaction（压缩）以及其他性能优化，以实现高质量、高效率的智能体输出。如果您希望通过直接访问模型来构建自己的智能体循环，请参阅[使用 Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages)。

<Note>
  Claude Managed Agents 也可在 Claude Platform on AWS 上使用，但在功能可用性和会话行为方面存在一些差异。请参阅 Claude Platform on AWS 指南中的 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#claude-managed-agents)。
</Note>

<CardGroup cols={3}>
  <Card title="快速入门" icon="play" href="https://platform.claude.com/docs/zh-CN/managed-agents/quickstart">
    创建您的第一个智能体会话
  </Card>

  <Card title="启动会话" icon="code-brackets" href="https://platform.claude.com/docs/zh-CN/managed-agents/sessions">
    创建会话并发送您的第一个事件
  </Card>

  <Card title="参考" icon="book" href="https://platform.claude.com/docs/zh-CN/managed-agents/reference">
    事件类型、速率限制、CLI 标志及其他查询表
  </Card>
</CardGroup>

## 核心概念

Claude Managed Agents 围绕四个概念构建：

| 概念                  | 描述                                           |
| ------------------- | -------------------------------------------- |
| **Agent（智能体）**      | 模型、系统提示、工具、MCP 服务器和技能                        |
| **Environment（环境）** | 会话运行位置的配置：Anthropic 托管的云沙箱，或在您自己的基础设施上自托管的沙箱 |
| **Session（会话）**     | 在环境中运行的智能体实例，执行特定任务并生成输出                     |
| **Events（事件）**      | 您的应用程序与智能体之间交换的消息（用户轮次、工具结果、状态更新）            |

## 工作原理

<Steps>
  <Step title="创建智能体">
    定义模型、系统提示、工具、MCP 服务器和技能。只需创建一次智能体，即可在各个会话中通过 ID 引用它。
  </Step>

  <Step title="创建环境">
    配置智能体的运行位置：云端沙箱，或在您自己的基础设施上运行的[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)。
  </Step>

  <Step title="启动会话">
    启动一个引用您的智能体和环境配置的会话。
  </Step>

  <Step title="发送事件并流式传输响应">
    以事件形式发送用户消息。Claude 会自主运行工具，并通过 "server-sent events"（服务器发送事件），即 SSE，以流式传输方式返回结果。事件历史记录会在服务器端持久化保存，并可完整获取。
  </Step>

  <Step title="引导或中断">
    在执行过程中发送额外的用户事件来引导智能体，或中断它以改变方向。
  </Step>
</Steps>

## 何时使用 Claude Managed Agents

Claude Managed Agents 最适合具有以下需求的工作负载：

* **长时间运行的执行：** 运行数分钟或数小时、涉及多次工具调用的任务
* **云基础设施：** 预装软件包并具备网络访问能力的安全沙箱
* **自托管执行：** 在您控制的基础设施上运行沙箱，以满足合规性或数据驻留要求
* **最少的基础设施：** 无需自行构建智能体循环、沙箱或工具执行层
* **有状态会话：** 跨多次交互持久保存的文件系统和对话历史
* **定时执行：** 通过[定时部署](https://platform.claude.com/docs/zh-CN/managed-agents/scheduled-deployments)按 cron 计划定期运行智能体

## 支持的工具

Claude Managed Agents 为 Claude 提供了一组内置工具：

* **Bash：** 在沙箱中运行 shell 命令
* **文件操作：** 在沙箱中读取、写入、编辑、glob 和 grep 文件
* **网页搜索和抓取：** 搜索网页并从 URL 获取内容，可选择限制为域名允许列表或阻止列表
* **MCP 服务器：** 连接到外部工具提供商

请参阅[工具](https://platform.claude.com/docs/zh-CN/managed-agents/tools)了解完整列表和配置选项。

## Beta 访问

<Note>
  Claude Managed Agents 目前处于 beta 阶段。所有 Managed Agents 端点都需要 `managed-agents-2026-04-01` beta 请求头。SDK 会自动设置该 beta 请求头。各版本之间可能会对行为进行调整以改进输出。
</Note>

要开始使用，您需要：

1. 一个 [Claude API 密钥](https://platform.claude.com/settings/keys)
2. 在所有请求中添加 `managed-agents-2026-04-01` beta 请求头
3. Claude Managed Agents 的访问权限（默认对所有 API 账户启用）

在 beta 阶段内，[MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview)和 [dreaming（梦境）](https://platform.claude.com/docs/zh-CN/managed-agents/dreams)处于更为有限的研究预览阶段。请[申请访问权限](https://claude.com/form/claude-managed-agents)以启用它们。

Claude Managed Agents 在设计上是有状态的：会话可长时间运行，暂停后可顺利恢复，并在服务器端存储对话历史、沙箱状态和输出。因此，Managed Agents 目前不符合[零数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#zero-data-retention-zdr-scope)或 HIPAA 商业伙伴协议（BAA）的覆盖条件。您保留对这些数据的控制权：您可以随时通过 API [删除会话](https://platform.claude.com/docs/zh-CN/managed-agents/session-operations#deleting-a-session)，并单独删除您上传的任何[文件](https://platform.claude.com/docs/zh-CN/build-with-claude/files#delete-a-file)。有关所有功能的适用资格，请参阅 [API 与数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)。

请参阅参考文档中的[速率限制](https://platform.claude.com/docs/zh-CN/managed-agents/reference#rate-limits)和[品牌指南](https://platform.claude.com/docs/zh-CN/managed-agents/reference#branding-guidelines)。
