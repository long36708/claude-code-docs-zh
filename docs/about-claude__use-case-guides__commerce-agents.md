---
title: 商务智能体
url: https://platform.claude.com/docs/zh-CN/about-claude/use-case-guides/commerce-agents
description: 使用 Claude for commerce 在 Claude 上构建购物智能体和商家智能体。Claude for commerce 是一个开源蓝图，提供基于 Messages API、Claude Agent SDK 和 Claude Managed Agents 的可运行实现。
---

本指南介绍如何在 Claude 上构建 "commerce agent"（商务智能体）。其中包括两类智能体：一是 "shopping agent"（购物智能体），供客户在您的应用内使用；二是 "merchant agent"（商家智能体），供店铺运营人员使用，这些人员可以是企业自己的运营团队，也可以是其平台上的卖家。本指南通过 Claude for commerce 来实现这一点。Claude for commerce 是一个开源蓝图，包含以下内容：每种智能体分别基于 [Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages)、[Claude Agent SDK](https://code.claude.com/docs/zh-CN/agent-sdk/overview) 和 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 的可运行实现；零售、旅游、电信和娱乐行业的可运行示例；以及一个 Claude Code 插件，可针对您自己的系统搭建相同设计的脚手架。

代码、设置说明和安全文档位于 [GitHub 上的 Claude for commerce 代码仓库](https://github.com/anthropics/commerce-agents)中。如需了解这些智能体的构建方式及其设计原因，请阅读工程博文[高效商务智能体剖析指南](https://claude.com/blog/the-anatomy-of-effective-commerce-agents)。博文涵盖以下主题：单智能体加 "skills"（技能）的设计、将 UI 组件作为工具、由 "harness"（运行框架）强制执行的安全机制、提示缓存、记忆以及 "evals"（评估）。

<Note>
  该蓝图是供您 fork 和改造的参考实现，而非受支持的产品或托管服务。
</Note>

## 购物智能体

购物智能体运行在您的应用内，通过单一后端接口访问您的系统。您需要基于自己的商品目录、购物车、偏好、订单、政策和履约服务来实现该接口。该接口上没有任何方法会下单或转移资金。在对话中，该智能体可以：

* 搜索商品目录，比较候选商品，并将客户描述的需求转化为候选清单和推荐。
* 围绕某个目标（例如一次旅行、一场活动或一个房间）规划一组相互搭配的商品，并使其符合预算。
* 以在对话中渲染的 UI 组件形式展示商品、比较结果、方案、订单状态和购物车。
* 填充购物车并移交给您的结账流程。
* 依据您自己的订单和政策系统，回答有关订单、配送、退货和政策的问题。
* 记住客户要求它记住的内容，并在之后的会话中加以应用。

五项技能按需加载，分别涵盖搜索与发现、购买调研、围绕目标的规划、客户服务，以及记忆与个性化。智能体在大多数轮次中都需要遵循的规则（例如 "grounding"（事实依据）、购物车与结账语义以及呈现方式）则放在系统提示中。

智能体只依据对话中的工具结果来陈述商品、价格、库存情况和店铺条款；写入购物车时，只接受在该会话中由商品目录工具或订单工具返回的商品 ID。结账时会暂存一份摘要，由客户在您的应用中确认。如果您使用托管结账，您的后端会返回其 URL，由宿主应用直接渲染，而不经过模型。

## 商家智能体

商家智能体为店铺运营人员提供支持。它通过自己的后端接口访问您的分析、商品目录、库存、定价和营销活动系统，并且可以：

* 解释业务表现：某项指标为何变化、由哪个细分群体驱动，以及与可比时期相比的进度。一个可选的分析委托代理会在时间和数据量预算内运行只读查询。
* 呈现每日摘要，列出需要关注的事项，包括低库存、滞销商品和订单异常。
* 根据运营人员提供的材料改进商品详情内容并修正商品目录数据。
* 在店铺的防护规则范围内推荐价格调整和促销活动，并提供利润率预览。
* 起草营销活动，包括受众、投放位置和预算。

它的五项技能涵盖业绩洞察、库存与运营、商品目录与商品详情、定价与促销，以及营销活动。

商家智能体提出的每一项写入操作，无论是商品详情更新、价格调整、库存操作、促销活动还是营销活动，都是一项暂存变更，带有服务器生成的 ID，运营人员会以预览卡片的形式看到它。最大调价幅度、促销力度、补货数量、营销活动预算和受保护字段等防护规则，会在变更暂存时检查一次，并在应用时再次检查。只有在有人于对话之外批准后，变更才会生效：在 Messages API 路径上是商家门户中的一个按钮，在 Agent SDK 控制台中是一个确认提示，在 Claude Managed Agents 上则是应用工具上的[始终询问权限策略](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies)。在聊天中输入的批准不会批准任何内容。

## 行业示例

每个示例都包含一个基于虚构数据的客户店面和一个商家门户。

| 行业 | 店面                        | 商家门户                                 |
| -- | ------------------------- | ------------------------------------ |
| 零售 | 基于内置组件的搜索、比较、方案、购物车、结账和记忆 | 每日摘要、暂存的补货和商品详情修正，以及基于 SQL 视图的分析委托代理 |
| 旅游 | 与日期绑定的库存和行程组件             | 入住率日历和按日期窗口的价格调整                     |
| 电信 | 账户上下文、套餐矩阵和由服务器生成的费用披露    | 套餐组合、注明受影响线路的价格调整，以及受保护的监管费用         |
| 娱乐 | 限时保留、候补名单、转让、场馆地图和全包费用披露  | 活动销售进度、能真正增加可售容量的保留释放，以及保留费用的价格调整    |

## 运行环境

Messages API 和 Agent SDK 运行时可以在 Claude API、[Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)、[Google Cloud's Agent Platform](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 或 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上运行，也可以通过您自己的网关运行。Claude Managed Agents 路径在 Claude API 上运行。代码仓库中的部署指南说明了每条路径在何处选择平台，以及各自需要哪种模型 ID 格式。

## 使用 Claude Code 插件构建您自己的智能体

该蓝图附带一个 [Claude Code](https://code.claude.com/docs/zh-CN/overview) 插件。该插件以克隆下来的代码仓库作为参考，针对您自己的系统构建智能体。它的四个命令覆盖了从零开始到完成测试的智能体的完整路径：

| 命令                         | 作用                                        |
| -------------------------- | ----------------------------------------- |
| `/scaffold-commerce-agent` | 询问您的技术栈，复述规划方案，并基于参考包搭建购物智能体、商家智能体或两者的脚手架 |
| `/add-commerce-flow`       | 为现有智能体添加一个流程：复制其技能，接入其调用的工具，并编写其首批评估用例    |
| `/author-commerce-evals`   | 构建评估套件，包括运行器、基于您自己商品目录的首批用例，以及用于 CI 的回放关卡 |
| `/review-commerce-agent`   | 梳理您已在运行的智能体，将其与参考实现进行比较，并转换您选定的部分         |

该插件还包含六项技能，分别涉及架构、提示缓存、将 UI 作为工具、信任与安全、评估以及商家运营。每当 Claude Code 对话与这些技能匹配时，它们就会加载。安装说明见代码仓库的 README。

如果您想手动改造参考实现，请基于您的服务实现购物或商家后端接口，使用配置标志关闭您没有的系统，使其工具和相关提示内容不再出现，并设置您的品牌名称、助手名称和语气风格。代码仓库中的后端指南详细介绍了身份与凭据、有序流程、结账移交以及带选项的商品。试点项目可以只实现搜索和商品详情，其余部分使用桩实现。

## 开始使用

<CardGroup cols={2}>
  <Card title="GitHub 上的 Claude for commerce" icon="github-logo" href="https://github.com/anthropics/commerce-agents">
    克隆该蓝图，并在本地运行两个智能体。
  </Card>

  <Card title="高效商务智能体剖析指南" icon="book" href="https://claude.com/blog/the-anatomy-of-effective-commerce-agents">
    了解这些智能体的构建方式及其设计原因。
  </Card>
</CardGroup>
