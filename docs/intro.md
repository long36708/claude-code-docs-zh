---
title: Claude 简介
url: https://platform.claude.com/docs/zh-CN/intro
description: Claude 是由 Anthropic 构建的高性能、值得信赖且智能的 AI 平台。Claude 擅长处理涉及语言、推理、分析、编码等方面的任务。
---

<Tip>
  最新一代 Claude 模型：

  **Claude Fable 5.1** - 适用于高要求的推理和长周期智能体工作。阅读 [Claude Fable 5.1 公告](https://www.anthropic.com/claude-fable-and-mythos-5-1)。

  **Claude Mythos 5.1** - 通过 [Project Glasswing](https://anthropic.com/glasswing) 以邀请方式提供 Claude Fable 5.1 的能力。

  **Claude Opus 5** - 适用于复杂的智能体编码和企业工作。阅读 [Claude Opus 5 公告](https://www.anthropic.com/news/claude-opus-5)。

  **Claude Sonnet 5** - 规模化的前沿智能，专为编码、智能体和企业工作流而构建。阅读 [Claude Sonnet 5 公告](https://www.anthropic.com/news/claude-sonnet-5)。

  **Claude Haiku 4.5** - 速度最快的模型，具备接近前沿的智能。阅读 [Claude Haiku 4.5 公告](https://www.anthropic.com/news/claude-haiku-4-5)。
</Tip>

<Note>
  想要与 Claude 聊天？请访问 [claude.ai](https://claude.ai)。
</Note>

Anthropic 提供两种使用 Claude 进行构建的方式，每种方式适用于不同的使用场景：

|          | Messages API   | Claude Managed Agents    |
| -------- | -------------- | ------------------------ |
| **它是什么** | 直接的模型提示访问      | 预构建、可配置的智能体框架，运行在托管基础设施中 |
| **最适合**  | 自定义智能体循环和细粒度控制 | 长时间运行的任务和异步工作            |

要详细了解每一项，请参阅[使用 Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 和 [Claude Managed Agents 概述](https://platform.claude.com/docs/zh-CN/managed-agents/overview)。

## 新开发者的推荐路径

按照以下步骤，从零开始构建一个可运行的 Claude 集成。

<Steps>
  <Step title="发起您的第一次 API 调用">
    设置您的环境，安装 SDK，并向 Claude 发送您的第一条消息。

    [前往快速入门](https://platform.claude.com/docs/zh-CN/get-started)
  </Step>

  <Step title="保护您的凭证">
    在创建 API 密钥时设置过期时间。不要将密钥放入源代码管理、客户端代码和提示中。检查您的工作负载是否可以使用 Workload Identity Federation（工作负载身份联合）来代替静态密钥。

    [阅读身份验证指南](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)
  </Step>

  <Step title="了解 Messages API">
    学习核心的请求和响应结构，包括多轮对话、"system prompt"（系统提示）和停止原因。

    [阅读 Messages API 指南](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages)
  </Step>

  <Step title="选择合适的模型">
    按能力和成本比较 Claude 模型，为您的用例挑选最合适的模型。

    [查看模型概述](https://platform.claude.com/docs/zh-CN/models/overview)
  </Step>

  <Step title="探索功能和工具">
    了解 Claude 的能力："extended thinking"（扩展思考）、网络搜索、文件处理、结构化输出等。

    [浏览功能概述](https://platform.claude.com/docs/zh-CN/build-with-claude/overview)
  </Step>
</Steps>

***

## 使用 Claude 进行开发

Anthropic 提供开发者工具，帮助您使用 Claude 构建和扩展应用程序。

<CardGroup cols={3}>
  <Card title="开发者控制台" icon="computer" href="https://platform.claude.com/">
    在浏览器中通过 playground 探索和了解 API。
  </Card>

  <Card title="API 参考" icon="code" href="https://platform.claude.com/docs/zh-CN/api/overview">
    探索完整的 Claude API 和客户端 SDK 文档。
  </Card>

  <Card title="Claude Cookbook" icon="chef-hat" href="https://platform.claude.com/cookbook">
    通过涵盖 PDF、嵌入等内容的交互式 Jupyter 笔记本进行学习。
  </Card>
</CardGroup>

***

## 关键能力

Claude 可以协助完成许多涉及文本、代码和图像的任务。

<CardGroup cols={2}>
  <Card title="文本和代码生成" icon="text-aa" href="https://platform.claude.com/docs/zh-CN/build-with-claude/overview">
    总结文本、回答问题、提取数据、翻译文本，以及解释和生成代码。
  </Card>

  <Card title="视觉" icon="image" href="https://platform.claude.com/docs/zh-CN/build-with-claude/vision">
    处理和分析视觉输入，并根据图像生成文本和代码。
  </Card>
</CardGroup>

***

## 支持

<CardGroup cols={2}>
  <Card title="帮助中心" icon="help" href="https://support.claude.com/en/">
    查找有关账户和账单的常见问题解答。
  </Card>

  <Card title="服务状态" icon="chart" href="https://status.claude.com">
    查看 Anthropic 服务的状态。
  </Card>
</CardGroup>
