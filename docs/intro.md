---
title: Claude 简介
url: https://platform.claude.com/docs/zh-CN/intro
description: Claude 是由 Anthropic 构建的高性能、值得信赖且智能的 AI 平台。Claude 擅长处理涉及语言、推理、分析、编码等方面的任务。
---

<Note>
  想要与 Claude 聊天？请访问 [claude.ai](https://claude.ai)。
</Note>

Anthropic 提供两种使用 Claude 进行构建的方式，每种方式适用于不同的使用场景：

|          | Messages API   | Claude Managed Agents    |
| -------- | -------------- | ------------------------ |
| **它是什么** | 直接的模型提示访问      | 预构建、可配置的智能体框架，运行在托管基础设施中 |
| **最适合**  | 自定义智能体循环和细粒度控制 | 长时间运行的任务和异步工作            |

要详细了解每一项，请参阅[使用 Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages) 和 [Claude Managed Agents 概述](https://platform.claude.com/docs/zh-CN/managed-agents/overview)。

## 探索最新一代 Claude 模型

如果您不确定该使用哪个模型，对于大多数工作负载，建议从 [Claude Opus 5.5](https://platform.claude.com/docs/zh-CN/models/opus-5-5/overview) 开始。对于要求较高的推理任务和长周期的 "agentic"（智能体）工作，或者当您在较高 effort 级别下对 Claude Opus 5.5 进行的 "evals"（评估）仍未达到要求时，请使用 [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview)。所有当前模型均支持文本和图像输入、文本输出、多语言能力、"vision"（视觉）以及 "tool use"（工具使用）。每个模型的页面都列出了该模型可用的平台。

* [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/overview) (`claude-fable-5-1`) — New — *For demanding reasoning and long-horizon agentic work* — Most capable · Research · Multi-day tasks
* [Claude Opus 5.5](https://platform.claude.com/docs/zh-CN/models/opus-5-5/overview) (`claude-opus-5-5`) — New — *For long-running agentic coding and knowledge work* — Complex projects · Agents · Coding
* [Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/overview) (`claude-sonnet-5`) — *The best combination of speed and intelligence* — Everyday tasks · Writing · Cost-efficient
* [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/haiku-4-5/overview) (`claude-haiku-4-5`) — *The fastest model with near-frontier intelligence* — Fastest · Lowest cost · High volume

[比较模型](https://platform.claude.com/docs/zh-CN/models/overview)

***

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
