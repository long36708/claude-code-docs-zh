---
title: 压缩概述
url: https://platform.claude.com/docs/zh-CN/build-with-claude/compaction
description: 了解压缩的作用、按需压缩与按令牌阈值压缩的区别，以及哪个页面涵盖您的任务。
---

"Compaction"（压缩）会用 Claude 在服务器上编写的摘要替换对话中较早的轮次，因此您无需自己编写摘要代码。它能让长对话或智能体任务保持在 "context window"（上下文窗口）之内，并让活动上下文保持精简，因为响应质量会随着对话变长而下降。

如果您想跳过本概述，请从[按需压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand)开始，为您的应用程序添加压缩功能。虽然按需压缩和[按令牌阈值压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold)都处于 beta 阶段，但按需压缩涵盖的常见用例更多。如果您想按规则清除旧的工具结果或旧的思考块，而不是对其进行摘要，请参阅[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)。

## 选择压缩方式

只要按需压缩可用，就请使用它。

|                                                                                                              | [按需压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand)                                                                   | [按令牌阈值压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold)                                                                                                                      | 您自己的摘要器（[在客户端进行压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#custom-compaction-on-the-client)） |
| ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **由谁决定何时压缩**                                                                                                 | 您，通过发送请求                                                                                                                                                | API，在输入令牌达到您设置的触发值时                                                                                                                                                                                           | 您                                                                                                                                |
| **您需要编写的代码**                                                                                                 | 一个[压缩循环](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#compact-in-a-loop)，用于请求摘要并将其替换进去                                 | 在普通请求中添加一个参数                                                                                                                                                                                                  | 摘要调用、其提示以及历史记录重写                                                                                                                 |
| **之后您需要发回的内容**                                                                                               | 将返回的块放在 `messages` 的最前面，替换它所摘要的消息                                                                                                                       | 照常追加响应。API 会丢弃该块之前的内容                                                                                                                                                                                         | 您的摘要，作为您自己的一条消息                                                                                                                  |
| **最近的轮次保持原文不变**                                                                                              | 是：[保留最近轮次的压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-keep-recent-turns)                                                    | 是，通过[压缩后暂停](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold#pausing-after-compaction)并重新插入这些轮次                                                                                  | 是                                                                                                                                |
| **在后台运行**                                                                                                    | 是：[后台压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-background)                                                                | 否：它在达到阈值的请求内部运行                                                                                                                                                                                               | 是，在您自己的代码中                                                                                                                       |
| **保留的轮次保留其思考内容**，适用于支持[保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking)的模型 | 是，需满足[压缩与保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks)中的条件                                                 | 否：在 Claude Fable 5.1 和 Claude Opus 5.5 上，请移除或丢弃您重新插入的轮次中的思考内容（请参阅[阈值压缩示例](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold#examples)）                                            | 否：这些思考内容[无法通过检查](https://platform.claude.com/docs/zh-CN/build-with-claude/preserved-thinking#keep-tail-compaction)               |
| **Beta 标头、参数和平台**                                                                                            | `compact-2026-09-04` 和顶层 `compaction` 参数。平台：请参阅[按需压缩兼容性列表](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#compatibility) | 请参阅[阈值压缩兼容性列表](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold#compatibility)和[基本用法](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold#basic-usage) | 无。它在您的代码中运行                                                                                                                      |
| **适用场景**                                                                                                     | 您的应用程序需要控制何时进行压缩、无法在编写摘要时暂停，或者必须保留最近的轮次及其思考内容                                                                                                           | 您希望 API 在普通请求中管理上下文                                                                                                                                                                                           | 您已经在运行自己的摘要器，并且它会用摘要替换整个历史记录                                                                                                     |

## 按需压缩选项

[按需压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand)展示了一个在您的应用程序等待期间对整个对话进行摘要的循环。此列表中的前两个页面会改变该循环的运行方式，您可以将它们结合使用。如果您会发回思考块并使用了其中任一方式，则第三个页面适用。

* [保留最近轮次的压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-keep-recent-turns)：当最后几个轮次必须原文传递给 Claude 时使用。摘要仅涵盖较早的轮次。
* [后台压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-background)：当对话无法为生成摘要而暂停时使用。请求在工作继续进行的同时运行，块到达后您再将其替换进去。
* [压缩与保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks)：如果您在支持保留思考的模型上发回思考块，并且保留最近的轮次或在后台进行压缩，请阅读此页面。否则请跳过。
* [编写您自己的总结提示](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#write-your-own-summarization-prompt)：当默认摘要遗漏了后续轮次所需的内容时使用。您的提示会替换默认提示。
* [再次压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#compact-again)：当一个已经以压缩块开头的对话再次变长时使用。

以下主题位于其他页面：

* [请求摘要](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#request-a-summary)位于按需压缩页面。
* [从摘要继续](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#continue-from-the-summary)位于按需压缩页面。使您保留的轮次中的思考内容保持有效的条件位于[压缩与保留思考](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-thinking-blocks)页面。
* 未返回摘要时：请参阅按需压缩页面上的[处理缺失的摘要或错误](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#when-no-summary-comes-back)。
* 它如何与 API 的其他部分配合：请参阅按需压缩页面上的[限制以及与其他功能的交互](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#how-it-fits-with-the-rest-of-the-api)。
* 了解用量：对于按需压缩，请参阅[统计压缩用量](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#understanding-usage)；对于阈值压缩，请参阅[了解用量](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold#understanding-usage)。
* 兼容性：每种压缩方式都有自己的 beta 标头，因此每个页面都有自己的列表。请参阅[按需压缩兼容性列表](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-on-demand#compatibility)或[阈值压缩兼容性列表](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold#compatibility)。

阈值压缩有其专门的页面：[按令牌阈值压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction-threshold)。
