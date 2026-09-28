---
title: 提示工程概述
url: https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/overview
description: 了解何时提示工程是正确的解决方案，并查找 Claude 提示技巧和交互式教程。
---

## 提示工程之前

本指南假设您已具备：

1. 对您的用例成功标准的清晰定义
2. 一些根据这些标准进行实证测试的方法
3. 一份您想要改进的提示初稿

如果没有，请先花时间建立这些基础。请查看[定义成功标准并构建评估](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests)以获取提示和指导。

<CardGroup cols={2}>
  <Card title="提示生成器笔记本" icon="link" href="https://colab.research.google.com/github/anthropics/claude-cookbooks/blob/main/misc/metaprompt.ipynb">
    还没有提示初稿？使用 Claude Cookbook 中的 metaprompt（元提示）配方生成一份。
  </Card>

  <Card title="提示最佳实践" icon="link" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices">
    如需针对 Claude 最新模型的特定模型调优指导，请从这里开始。
  </Card>
</CardGroup>

***

## 何时进行提示工程

本指南侧重于可通过"prompt engineering"（提示工程）控制的成功标准。 并非每个成功标准或失败的评估都最适合通过提示工程来解决。例如，有时您可以通过选择不同的模型来更轻松地改善"latency"（延迟）和成本。

***

## 如何进行提示工程

所有提示技巧（从清晰表达和示例到 XML 结构化、角色提示、思考以及提示链）均在[提示最佳实践](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices)中有所介绍。那是持续更新的参考文档；请从那里开始。

如需了解 Claude 特定技巧之外的通用提示工程技艺，请参阅关于[提示工程最佳实践](https://claude.com/blog/best-practices-for-prompt-engineering)的博客文章。

***

## 提示工程教程

如果您是交互式学习者，您也可以从交互式教程开始！

<CardGroup cols={2}>
  <Card title="GitHub 提示教程" icon="link" href="https://github.com/anthropics/prompt-eng-interactive-tutorial">
    一个包含丰富示例的教程，涵盖文档中介绍的提示工程概念。
  </Card>

  <Card title="Google Sheets 提示教程" icon="link" href="https://docs.google.com/spreadsheets/d/19jzLgRruG9kjUQNKtCg1ZjdD6l6weA6qRXG5zLIAhC8">
    提示工程教程的轻量级版本，以交互式电子表格的形式呈现。
  </Card>
</CardGroup>
