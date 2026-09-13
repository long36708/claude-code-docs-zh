---
title: 选择合适的模型
url: https://platform.claude.com/docs/zh-CN/about-claude/models/choosing-a-model
description: 选择 Claude 模型意味着在能力、速度和成本之间取得平衡。本指南涵盖了需要考虑的问题、两种选择起始模型的方法，以及如何测试您的选择。
---

## 确立关键标准

在选择 Claude 模型时，建议首先评估以下因素：

* **能力：** 您需要模型具备哪些特定功能或能力才能满足您的需求？
* **速度：** 在您的应用中，模型需要多快地响应？Claude Opus 5 和 Claude Opus 4.8 支持 [fast mode（快速模式）](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)（研究预览版），以高级定价提供最高 2.5 倍的输出速度。
* **成本：** 您在开发和生产使用方面的预算是多少？
* **Effort（努力程度）：** 多个 Claude 模型支持 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)，可在单个模型内以智能换取延迟和成本。调整 effort 通常是比切换模型更好的手段。在 Claude Fable 5.1 和 Claude Opus 5 上，请从默认值（`high`）开始，并根据您的评估结果向上或向下调整。在 Claude Opus 4.8 和 Claude Opus 4.7 上，介于 `high` 和 `max` 之间的 `xhigh` effort 级别是大多数编码和智能体用例的最佳设置。

***

## 选择最佳的起始模型

您可以使用两种通用方法来开始测试哪个 Claude 模型最适合您的需求。

### 选项 1：效率优先

对于许多应用而言，从更快、更具成本效益的模型（如 Claude Haiku 4.5）开始可能是最佳方法：

1. 使用 Claude Haiku 4.5 开始实现。
2. 全面测试您的用例。
3. 评估性能是否满足您的要求。
4. 仅在存在特定能力差距且确有必要时才升级。

这种方法可以实现快速迭代、降低开发成本，并且通常足以满足许多常见应用的需求。这种方法最适合：

* 初始原型设计和开发
* 对延迟要求严格的应用
* 对成本敏感的实现
* 高容量、简单直接的任务

### 选项 2：能力优先

对于智能和高级能力至关重要的复杂任务，您可能希望采用能力优先的方式：先使用最适合您任务的最强起点进行实现，然后再逐步优化到更高效的模型：

1. 使用 Claude Opus 5 进行实现。
2. 针对该模型[优化您的提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5)。
3. 评估性能是否满足您的要求。
4. 随着工作流程的进一步优化，考虑通过降低 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 或逐步降级模型来提高效率。
5. 如果您在 `xhigh` 或 `max` effort 下的评估在高要求推理或长周期智能体工作上仍然不达标，请转向 [Claude Fable 5.1](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1)。

这种方法最适合：

* 复杂推理任务
* 科学或数学应用
* 需要细致理解的任务
* 准确性优先于成本考量的应用
* 高级编码和高自主性智能体工作

**Claude Opus 5**（`claude-opus-5`）专为复杂的智能体编码和企业工作而构建，具备深度推理、长周期任务处理和测试时计算扩展能力。

**Claude Fable 5.1**（`claude-fable-5-1`）是 Anthropic 广泛发布的能力最强的模型。它在 Claude Fable 5 的基础上进行了扩展，以相同的输入和输出价格提供更强的长时间运行智能体编码、知识工作和研究能力，且缓存读取成本仅为四分之一。**Claude Mythos 5.1**（`claude-mythos-5-1`）仅向 [Project Glasswing](https://anthropic.com/glasswing) 参与者提供相同的能力。这两个模型都使用始终开启的 [adaptive thinking（自适应思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。详情请参阅 [Claude Fable 5.1 的新功能](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1)。

Claude Fable 5 和 Claude Mythos 5 也可供使用。详情请参阅 [Claude Fable 5 和 Claude Mythos 5 介绍](https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5)。有关上下文窗口、输出限制和价格，请参阅[模型比较表](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)。

## 模型选择矩阵

大多数工作负载从 Claude Opus 5 开始。

| 当您需要……                   | 考虑从以下模型开始……      | 示例用例                                         |
| ------------------------ | ---------------- | -------------------------------------------- |
| 最高的可用能力                  | Claude Fable 5.1 | 运行数小时的智能体会话、多步骤深度研究、一直推进到完成文档、电子表格或演示文稿的分析   |
| 复杂的智能体编码和企业工作            | Claude Opus 5    | 持续数小时的自主编码智能体、大规模重构、复杂系统工程、重度依赖视觉的工作流程、计算机使用 |
| 适用于日常编码、智能体和企业工作负载的速度与能力 | Claude Sonnet 5  | 代码生成、数据分析、内容创作、视觉理解、智能体工具使用                  |
| 最低的延迟和价格，并支持扩展思考         | Claude Haiku 4.5 | 实时应用、高容量智能处理、需要强大推理能力的成本敏感型部署、子智能体任务         |

***

## 决定是否升级或更换模型

要确定是否需要升级或更换模型，您应该：

1. [创建基准测试](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests)，专门针对您的用例——拥有一个良好的评估集是整个过程中最重要的一步。

2. 使用您的实际提示和数据进行测试。

3. 比较各模型在以下方面的性能：

   * 响应的准确性
   * 响应质量
   * 边缘情况的处理

4. 权衡性能与成本之间的取舍。

## 组合模型

多模型策略将低成本模型与前沿模型配对，使大多数令牌按较低费率计费。两种常见模式是：执行者将困难决策上报给顾问，以及编排者将批量工作委派给低成本的工作者。有关这两种策略、实测示例和实现选项，请参阅[针对成本和智能进行优化](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="模型比较图表" icon="settings" href="https://platform.claude.com/docs/zh-CN/models/overview">
    查看最新 Claude 模型的详细规格和定价
  </Card>

  <Card title="Claude Fable 5.1 的新功能" icon="sparkle" href="https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1">
    专为高要求推理和长周期智能体工作而构建
  </Card>

  <Card title="Claude Opus 5 的新功能" icon="sparkle" href="https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5">
    探索 Claude Opus 5 的改进之处
  </Card>

  <Card title="Claude Sonnet 5 的新功能" icon="sparkle" href="https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5">
    适用于兼顾速度与能力的日常工作负载
  </Card>

  <Card title="开始构建" icon="code" href="https://platform.claude.com/docs/zh-CN/get-started">
    开始您的第一次 API 调用
  </Card>
</CardGroup>
