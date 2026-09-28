---
title: CLI、SDK 和库
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview
description: 用于基于 Claude API 进行构建的官方工具：ant CLI、七种语言的客户端 SDK，以及面向特定框架的库。
---

Anthropic 提供三类官方工具，用于基于 Claude API 进行构建：

* **CLI：** `ant` 命令行工具，用于 shell 脚本编写和交互式使用。
* **客户端 SDK：** 适用于 Python、TypeScript、C#、Go、Java、PHP 和 Ruby 的通用 Messages API 客户端。每个 SDK 都提供符合语言习惯的接口、类型安全，以及对 "streaming"（流式传输）、重试和错误处理的内置支持。
* **库和集成：** 通过其他框架的 API 接口（而非直接通过 Messages API）暴露 Claude 的软件包和兼容层。

<Info>
  有关完整的 API 规范，请参阅 [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)。
</Info>

## CLI

<CardGroup cols={3}>
  <Card title="ant CLI" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/quickstart">
    Shell 脚本编写、类型化标志、响应转换
  </Card>
</CardGroup>

## 客户端 SDK

<CardGroup cols={3}>
  <Card title="Python" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/python">
    同步和异步客户端、Pydantic 模型
  </Card>

  <Card title="TypeScript" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/typescript">
    支持 Node.js、Deno、Bun 和浏览器
  </Card>

  <Card title="C#" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/csharp">
    .NET Standard 2.0+、IChatClient 集成
  </Card>

  <Card title="Go" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/go">
    基于 Context 的取消、函数式选项
  </Card>

  <Card title="Java" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/java">
    构建器模式、CompletableFuture 异步
  </Card>

  <Card title="PHP" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/php">
    值对象、构建器模式
  </Card>

  <Card title="Ruby" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/ruby">
    Sorbet 类型、流式传输辅助工具
  </Card>
</CardGroup>

## 库和集成

库和集成通过其他框架的 API 接口暴露 Claude。它们不是通用的 Messages API 客户端。

<CardGroup cols={3}>
  <Card title="Apple Foundation Models" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/apple-foundation-models">
    适用于 Apple `LanguageModelSession` API 的 Swift 软件包
  </Card>

  <Card title="OpenAI SDK 兼容性" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/openai-sdk">
    通过 OpenAI SDK 接口使用 Claude
  </Card>
</CardGroup>

## 正在构建智能体或使用 Claude Code？

CLI、客户端 SDK 和库用于您自行调用 Claude API：由您发送每个请求并处理每个响应。Claude Code、Claude Agent SDK 和 Claude Managed Agents 则在更高层级上工作，提供智能体循环、工具执行和运行时。

<CardGroup cols={3}>
  <Card title="Claude Code" href="https://code.claude.com/docs/zh-CN/overview">
    用于将编码任务委托给 Claude 的智能体式编码工具
  </Card>

  <Card title="Claude Agent SDK" href="https://code.claude.com/docs/zh-CN/agent-sdk/overview">
    构建在您自行运营的进程中运行的智能体
  </Card>

  <Card title="Claude Managed Agents" href="https://platform.claude.com/docs/zh-CN/managed-agents/overview">
    在 Anthropic 的托管基础设施中运行智能体
  </Card>
</CardGroup>
