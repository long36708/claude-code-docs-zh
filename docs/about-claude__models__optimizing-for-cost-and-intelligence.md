---
title: 优化成本与智能
url: https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence
description: 在 Claude Platform 上平衡成本与智能，附有提示缓存、effort、模型选择、预算以及多模型策略的实测结果。
---

当一个工作负载从原型走向生产时，成本就成为首要的设计约束。能力最强的模型在大规模使用时可能过于昂贵，而最便宜的模型在质量上可能达不到要求。要管理好成本，就需要理解每个成本杠杆如何影响输出质量，因为有些杠杆会以质量为代价，有些则不会。Claude Platform 让您可以直接控制这种权衡。您为每个请求选择模型、effort（努力程度）级别和架构，这使您几乎可以将工作负载放在成本—智能前沿上的任何位置。

成本与智能通常被描绘为一条前沿曲线，一方用来换取另一方。本页的第一组杠杆通过在不影响质量的情况下削减成本，使工作负载向该前沿靠近；只有第二组杠杆是沿着前沿移动：

![成本—智能前沿示意图：一个箭头在相同质量下削减支出，另一个箭头以质量换取成本](https://platform.claude.com/docs/images/cost-intel-frontier.png)

这些杠杆分为两类：

* **免费收益**在不影响质量的情况下削减支出："prompt caching"（提示缓存）、令牌精简、针对您正在运行的模型进行提示审计、对可以等待最多 24 小时的工作使用享有 50% 折扣的[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)，以及作为兜底的[工作区支出限额](https://platform.claude.com/docs/zh-CN/api/rate-limits#setting-lower-limits-for-workspaces)。
* **权衡**以成本换取智能：模型选择、effort、输出上限和任务预算，以及多模型架构。

每个杠杆都附有实测结果以及何时划算的规则。在 Anthropic 的测量中，提示缓存是遥遥领先的最大杠杆：在本指南的基准测试中，它将智能体循环的成本降低了 2.7 到 5.3 倍，并将一个小型分诊智能体的账单削减了 83%，加上输入精简后达到 88%。多模型杠杆的适用范围更窄；第二个模型在两种形态下划算：顾问（advisor）和编排器（orchestrator）。

## 从这里开始

将您的情况与某一行匹配。

| 您的情况                             | 这样做                                                                                                                                | 位置                                                                                                                                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 任何工作负载、任何模型                      | 开启提示缓存并精简不需要的令牌；两者都是免费的                                                                                                            | [缓存重复的上下文](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#cache-repeated-context) · [精简令牌](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens) |
| 轮次之间有人在等待                        | 一旦大约每 20 个轮次中有 1 个跟在 5 分钟到 1 小时之间的停顿之后，且很少有间隔超过 1 小时，就使用 1 小时缓存时长。在 Claude Fable 5.1 上，当停顿为几分钟时保持 5 分钟缓存处于热状态，当停顿接近 1 小时时购买 1 小时时长 | [选择缓存时长](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#pick-the-cache-duration)                                                                                                                                          |
| 成本太高；质量没问题                       | 在当前模型上向下扫描 effort                                                                                                                  | [调整 effort](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#tune-effort)                                                                                                                                                   |
| 您没有使用最新模型                        | 升级；当前模型能解决更多任务，每个已解决任务的成本从低约 40% 到高约 20% 不等                                                                                        | [升级模型](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#upgrade-the-model)                                                                                                                                                  |
| 您正在选择或切换模型                       | 按每个已完成任务的成本比较，而不是按每令牌                                                                                                              | [比较模型](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#compare-models-on-cost-per-task)                                                                                                                                    |
| 质量不够好                            | 如果您降低了 effort，请恢复它；否则尝试上一档模型的 `low` effort                                                                                         | [调整 effort](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#tune-effort) · [比较模型](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#compare-models-on-cost-per-task)         |
| 尝试以 `stop_reason: max_tokens` 结束 | 提高 `max_tokens`；在默认 effort 下测量的 14,000 个轮次中，64,000 覆盖了除 2 个之外的全部轮次，而 128,000 在每个已解决任务上不增加任何额外成本                                    | [设置预算](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#set-budgets-and-output-caps)                                                                                                                                        |
| 您可以检查输出（测试、验证器）                  | 以低 effort 运行所有内容，并以默认值（`high`）重新运行失败项；在所测量的编码基准上，通过率保持不变而成本约为一半                                                                    | [重新运行失败项](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#re-run-failures-at-higher-effort)                                                                                                                                |
| 智能体循环中有少数非常昂贵的运行                 | 设置任务预算（beta；查看支持表了解哪些模型支持）、Claude Managed Agents 会话预算和工作区支出限额                                                                      | [设置预算](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#set-budgets-and-output-caps)                                                                                                                                        |
| 低成本模型仅在困难决策上停滞                   | 添加一个前沿顾问。当其定价远高于执行者且确实被咨询时才划算，因此先单独以低 effort 为顾问的模型定价并测量咨询率                                                                        | [顾问策略](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#advisor-strategy-escalate-hard-decisions)                                                                                                                           |
| 工作超出一个上下文窗口                      | 将分区委派给更便宜的工作者                                                                                                                      | [编排器策略](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#orchestrator-strategy-delegate-bulk-work)                                                                                                                          |

这些结果是 Anthropic 内部的（[引用的基准](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)），具有方向性而非保证，因此请使用[四步方法](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#measure-on-your-own-workload)在您自己的工作负载上进行测量。

## 在不损失质量的情况下削减支出

提示缓存、令牌精简、批处理以及针对当前模型的提示审计都能降低您的支出而不降低输出质量。有两点需要注意：批处理以延迟换取折扣，而上下文编辑这一令牌精简杠杆在本节测量的运行中花费超过了它节省的。

### 缓存重复的上下文

#### 为什么缓存排在第一位

在使用任何其他杠杆之前先开启[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)，因为智能体任务的每个轮次都会重新发送整个不断增长的对话："system prompt"（系统提示）、工具定义以及之前的每个轮次。一个 40 轮的任务会将其第一轮发送 40 次，因此任务成本大致随轮次数的平方增长。缓存不会阻止重新发送，但每次重新发送的成本约为十分之一且处理更快：前缀按[缓存读取费率](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#pricing)计费，即输入价格的十分之一，每个轮次仅对新增内容支付 1.25 倍的缓存写入费率。

**良好状态是什么样的。** 在一整天的真实流量中，智能体循环从缓存中读取的输入中位数为 84%，而排名前 10% 的 harness（无论是否为编码类）读取 94% 或更多[17](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)。在任务深处，一个构建良好的循环对不到 1% 的输入支付全价。低于约 80% 时，请查找是什么破坏了缓存（参见[什么会破坏缓存](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#what-breaks-the-cache)）。

在 Anthropic 的实测运行中，缓存读取通常是任务成本中最大的单一组成部分，这使得缓存比大多数模型选择决策更有价值。Anthropic 对 DeepResearch Bench II[7](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 的运行分别在有缓存和无缓存的情况下进行了定价：

![哑铃图，DeepResearch Bench II：使用缓存后，Claude Fable 5.1 每任务从 $37.94 降至 $7.12，Claude Sonnet 5 从 $3.20 降至 $1.20](https://platform.claude.com/docs/images/cost-intel-caching.png)

缓存的默认生命周期为 5 分钟，而智能体循环的轮次间隔只有几秒，因此折扣适用于每个轮次的大多数令牌。缓存图表中的运行从缓存中读取了 79% 到 90% 的输入令牌。节省量随回合深度而变化，因为较短的循环重新读取的内容较少，但在所测量的每个模型和基准上，缓存始终是最大的单一杠杆。

#### 选择缓存时长

如果您的循环在轮次之间等待人的操作，请使用 [1 小时缓存时长](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#1-hour-cache-duration)。它的写入成本更高（输入价格的 2 倍而不是 1.25 倍）。任一时长下的未命中都会按写入价格而非读取价格对整个前缀计费，因此一旦每个会话中有几个轮次跟在 5 分钟到 1 小时之间的停顿之后，较长的时长就划算了。

要做出决定，请统计对话中连续请求之间的间隔：

* 大约每 20 个间隔中有超过 1 个落在 5 分钟到 1 小时之间，且超过 1 小时的间隔很少：使用 1 小时时长。
* 轮次间隔几秒到达：保持 5 分钟默认值。在没有任何停顿时，它在 Claude Sonnet 5 上比 1 小时设置便宜 15%，在 Claude Opus 5 上便宜 11%。
* 超过 1 小时的间隔很常见：保持默认值。超过 1 小时的间隔会使两种时长都过期，而 1 小时设置随后会以其更高的写入价格重新写入前缀，因此它在每个这样的间隔上都会亏损。在您超过 5 分钟的停顿中，如果约 60% 或更多也超过 1 小时，请保持默认值；只有当至少约 40% 的长停顿在 1 小时内结束时，1 小时时长才划算。

Anthropic 测量了[精简输入和上下文令牌](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens)中的分诊任务，在某些轮次之前插入停顿以模拟人的延迟[16](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)。在所测量的两个模型上，一旦大约每 30 个轮次中有 1 个跟在停顿之后，1 小时缓存就成为更便宜的设置，因此 1/20 规则留有余量，而且越过交叉点后差距迅速扩大，因为在 5 分钟设置下每个停顿后的轮次都会重新写入整个前缀。每个当前模型都使用相同的缓存写入倍数，除 Claude Fable 5.1 和 Claude Mythos 5.1 之外的每个模型都使用相同的读取价格，因此其他模型上的交叉点在相同范围内；Fable 5.1 是接下来要讨论的情况。在每个单元格中，准确率都保持在运行间噪声范围内。停顿后的轮次在 1 小时设置下保持了其热缓存延迟。下图绘制了 Claude Sonnet 5 上每会话成本与停顿轮次占比的关系：

![折线图：按停顿后轮次占比的每分诊会话成本；超过约每 30 个轮次 1 个后，1 小时缓存更便宜](https://platform.claude.com/docs/images/cost-intel-cache-ttl.png)

Anthropic 还测量了用于保持 5 分钟缓存处于热状态的额外请求。在 Claude Sonnet 5 和 Claude Opus 5 上，在任何停顿轮次占比下，它们相比 1 小时时长都没有可测量的节省，而在每个轮次前都有停顿时成本更高，因此请改用时长设置。

在 Claude Fable 5.1 上，最便宜的设置是另一种。它的[缓存读取](https://platform.claude.com/docs/zh-CN/about-claude/pricing#prompt-caching)成本为输入价格的 0.025 倍（每百万令牌 $0.25），而其缓存写入保持标准倍数，因此重新读取前缀的保活请求很便宜，而 1 小时时长的写入溢价是更大的账单。Anthropic 使用相同的三种设置在 Claude Fable 5.1 上测量了分诊任务[19](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)。只要停顿持续几分钟，保持 5 分钟缓存处于热状态的每会话成本就比 1 小时缓存低 13% 到 20%；只有在停顿接近 45 分钟时，1 小时缓存才胜出，每会话约 12 美分。在 Claude Fable 5.1 上，当人离开几分钟时保持 5 分钟缓存处于热状态，当停顿接近 1 小时时购买 1 小时时长：

![折线图：Claude Fable 5.1 和 Claude Sonnet 5 上按停顿轮次占比的实测每分诊会话成本；在 Fable 5.1 上保活始终低于 1 小时缓存，在 Sonnet 5 上一旦停顿常见，1 小时缓存胜出](https://platform.claude.com/docs/images/cost-intel-cache-keepalive.png)

要保持缓存处于热状态，请在上一个请求开始后的 4 分钟内再次发送上一个请求并将 `max_tokens` 设为 0，此后每 4 分钟发送一次，如果设置了 `stream` 则去掉它。从请求的开始计时，而不是从其响应的结束计时：[缓存的 5 分钟生命周期](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#how-prompt-caching-works)从写入或刷新该条目的请求开始时算起，因此响应生成所花费的时间也计入其中。这就是[预热请求](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#pre-warming-the-cache)：它刷新缓存的生命周期，不生成任何内容，并且只对缓存读取计费。不要更改前缀的任何一个字节，也不要使用 `max_tokens: 1`，那会无缘无故地采样一个令牌。请求的标头和正文都要重新发送：如果您的请求带有 `anthropic-beta` 标头（例如用于[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)），保活请求需要相同的标头，否则重放正文中受 beta 门控的字段会被拒绝。当请求设置了 `thinking.type: "enabled"`（Claude Fable 5.1 上默认的自适应思考没有问题）、结构化输出或强制工具选择时，`max_tokens: 0` 请求会被拒绝（[其限制](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#limitations)）；在这些工作负载上，请改为购买 1 小时时长。

<CodeGroup exclude="shell:CLI, python, typescript, csharp, go, java, php, ruby">
  ```bash cURL
  # 在上一个请求开始后的 4 分钟内（生成所花费的时间会计入
  # 缓存的生命周期），重新发送该请求并将 max_tokens 设为
  # 0，同时去掉 stream（max_tokens: 0 的请求无法进行流式传输）。发送与
  # 原始请求相同的标头，包括任何 anthropic-beta 标头。
  jq '.max_tokens = 0 | del(.stream)' last_request.json | \
    curl https://api.anthropic.com/v1/messages \
      -H "x-api-key: $ANTHROPIC_API_KEY" \
      -H "anthropic-version: 2023-06-01" \
      -H "content-type: application/json" \
      --data-binary @-
  ```
</CodeGroup>

#### 开启缓存

设置几乎不费功夫。[自动缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#automatic-caching)会为您放置断点；否则，Claude Code 附带的 [Claude API skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill) 可以通过一个提示为现有集成添加缓存。以下摘录展示了该 skill 将其添加到产生这些测量结果的 harness 中：

```text wrap
$ claude
> /claude-api add prompt caching to this integration

Done. Prompt caching is now wired into the harness. Two changes:

- build_system() now adds cache_control: {"type": "ephemeral"} to the
  system block. Since the API renders tools -> system -> messages, this
  one breakpoint caches the entire static prefix -- the full tool array
  plus the system prompt.
- apply_moving_cache_breakpoints() strips any stale markers, then marks
  the last content block of the two most recent user turns. The older
  marker is the read point matching the prefix the previous request
  cached; the newer one extends the cache for the next request.

That's 3 breakpoints total, under the limit of 4.
...
```

这些断点放置遵循[显式缓存断点](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#explicit-cache-breakpoints)中的标准模式。

#### 什么会破坏缓存

在任务期间有几种情况会破坏您的缓存。任何按请求变化的内容（例如时间戳或队列位置）如果放在稳定前缀之前，都会使每个请求变成一次完整的缓存写入：在[精简输入和上下文令牌](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens)中的分诊运行上，系统提示开头的一行 25 令牌的状态行使每次运行花费 $4.24 而不是 $0.59，比关闭缓存运行还贵。请将按请求变化的文本放在最新的用户轮次中。

缓存是对请求按顺序（工具，然后是系统提示，然后是消息）进行的字节精确前缀匹配，因此任何位置的更改都会使其后的所有内容失效。在请求之间更改 [`effort`](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 或思考配置会使缓存从该点起失效，在某些模型上还会使其前面的工具和系统提示失效；对系统提示的任何编辑都会使缓存从该点起失效；设置或更改输出格式会使整个对话的缓存失效；添加、删除或重新排序工具定义会使全部缓存失效。[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#what-invalidates-the-cache)页面列出了这些情况，输出格式除外，后者由[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs#prompt-modification-and-token-costs)涵盖。在最新的模型上，请使用[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)（即追加到 `messages` 的 `{"role": "system"}` 消息）来更改指令，而不是编辑顶层 `system` 字段：缓存的前缀保持完整。请查看该页面了解哪些模型支持它。在支持的模型上，[按消息更改 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta) 同样会保持缓存前缀完整。在 Claude Fable 5.1 和 Claude Mythos 5.1 上风险最高：一次破坏会以输入价格的 1.25 倍重新写入前缀，而不是以 0.025 倍读取，因此在 100,000 令牌的前缀上，一个被破坏的轮次花费 $1.25 而不是 $0.03，是读取的 50 倍，而在 Claude Opus 5 上是 12.5 倍（$0.63 而不是 $0.05）。

Anthropic 在分诊智能体的长会话上测量了这一点[18](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)。在会话中途进行的一次 effort 更改和一次工具添加分别重写了 39,000 和 60,000 个缓存令牌，这些会话每会话花费 $0.95。在压缩后的第一个请求上进行相同的两项更改花费 $0.75，而在触发压缩的请求上进行则花费 $0.92，因为压缩的摘要过程随后以缓存写入价格重新处理了 81,000 令牌的上下文：该摘要过程花费 $0.21，而当相同的更改晚一个请求进行时为 $0.04，每个分组中的准确率都在运行间噪声范围内：

![柱状图，每分诊会话成本：无更改 $0.81，会话中途更改 $0.95，在压缩请求上 $0.92，之后 $0.75](https://platform.claude.com/docs/images/cost-intel-compaction-timing.png)

中途更改[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)会使任何包含预算值的缓存前缀失效，因此请在第一个请求上设置一次。每次[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#context-editing-and-prompt-caching)都会使前缀从其清除点起失效，下一个请求需要付费重新缓存其后的所有内容，因此请分几个大批次清除，而不是许多小批次。在 Claude Fable 5.1 和 Claude Mythos 5.1 上，这些操作每令牌的成本是读取价格的 50 倍，因此在那里最为重要。请在自然断点处进行每一项会使缓存失效的更改，然后确认缓存读取没有下降；如果下降了，[缓存诊断](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics)会显示前缀在哪里发生了分歧。

### 精简输入和上下文令牌

大多数智能体请求携带着从不影响答案的令牌。精简它们很少会损失输出质量，尽管这里并非每个杠杆在测量时都节省了资金。有两个地方值得关注：

* **输入精简。** 网页抓取工具中的[动态过滤](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool#dynamic-filtering)将样板内容排除在抓取的页面之外，[图像调整大小](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#evaluate-image-size)使视觉输入尺寸合适，而[带延迟加载的工具搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)仅在需要时加载工具定义（本节稍后测量）。[程序化工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling)让 Claude 从代码中运行多个工具调用，从而只有过滤后的结果进入上下文；其文档报告在智能体搜索基准上输入令牌减少 24%，且得分更高。[管理工具上下文](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/manage-tool-context)比较了工具搜索、程序化工具调用、提示缓存和上下文编辑。
* **上下文生命周期。** [上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)清除过时的工具结果，而[自动压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)及其阈值可阻止长循环将其整个历史一直携带下去。

这些杠杆与缓存以及彼此之间相互作用，因此请按净效果来评判它们，并使用[缓存诊断](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics)确认您的缓存前缀在每次更改后仍然存在。Anthropic 在一个问题分诊智能体上测量了它们，该智能体处理来自一个公共仓库的 20 个带截图的真实 bug 报告，以及同一任务的一个令牌量为 2.6 倍的较长变体。在开启缓存的情况下，输入精简（图像调整大小和工具搜索）在短运行上进一步削减了 26%，在长运行上削减了 21%。

#### 延迟加载未使用的工具定义

附加到请求的每个工具定义在每个轮次都是输入，而几个 MCP 服务器加起来就有数百个。Anthropic 运行分诊智能体时使用了它自己的两个工具加上来自公共 MCP 服务器的真实工具定义目录，总计最多 502 个工具，要么全部加载，要么将额外的工具标记为 `defer_loading` 并置于[工具搜索](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)之后：

![折线图：全部工具加载时，运行成本从 $0.55 上升到 502 个工具时的 $1.02；使用工具搜索时保持在 $0.56](https://platform.claude.com/docs/images/cost-intel-tool-search.png)

在加载每个定义的情况下，随着目录增长，运行成本几乎翻倍，与每个请求上的 schema 令牌同步。使用工具搜索时，在每个目录规模下都保持平稳，在 502 个工具时便宜 45%。无论哪种方式，每个单元格中的准确率都是 20 个中的 15 到 18 个，而且模型从未调用错误的工具，因此在这个规模下，目录消耗的是金钱，而不是正确性。对于通过 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)接入的工具也是如此：在附加了一个公共 GitHub MCP 服务器的情况下，延迟加载其工具集（`default_config: {defer_loading: true}`）在相同准确率下将运行成本削减了 20%。

#### 将数据文件排除在提示之外

当模型需要对表格进行计算时，请使用 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 上传它，并让模型通过[代码执行](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)查询它，而不是将其粘贴进去。Anthropic 对一个 1,862 行的公共 CSV 提出了 25 个聚合问题[15](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)（求和、过滤计数、分组以及一个日期过滤），答案由 pandas 计算：

![散点图：上传文件并使用代码执行时，25 题中 25 题正确，花费 $0.40；粘贴到提示中时，25 题中 6 题正确，花费 $5.01](https://platform.claude.com/docs/images/cost-intel-data-files.png)

粘贴到提示中时，该表格在每个请求上约为 91,000 个输入令牌，Claude Sonnet 5 正确回答了 25 个问题中的 6 个。上传后使用代码执行，它回答了全部 25 个，而运行成本约为十二分之一。Claude Opus 5 表现出相同的模式。

#### 管理上下文生命周期

上下文杠杆只有在会话足够长、需要它们时才划算：

![按运行长度的柱状图：上下文编辑在短运行上增加 74%；压缩在长运行上节省 32%，修剪节省 39%](https://platform.claude.com/docs/images/cost-intel-hygiene.png)

在 20 个问题的运行上，它们没有节省任何费用，而上下文编辑多花了 74%。在长运行上，修剪节省了 39%，压缩节省了 32%，而上下文编辑没有任何变化。修剪是您自己编写的几行代码：在每个任务边界处，用一行摘录替换大型过时的工具结果。它缓存效果好，因为编辑位于对话的尾部，下一个任务反正会在那里添加新内容：边界后的第一个请求有 89% 的缓存读取，边界之间的请求有 81%。在整个运行范围内，修剪和上下文编辑的缓存效果大致相同。修剪更便宜，是因为上下文编辑会在任务中途重写修剪所删除的内容（约占差距的三分之二），并且因为它使上下文保持约一半大小（另外三分之一）。如果您使用上下文编辑，请[分几个大批次清除](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#context-editing-and-prompt-caching)。改编自 harness 的修剪代码：

```python
import re

PRUNED = "[pruned at issue boundary]"


def prune_task_boundary(messages, tool_name_by_id, threshold=2000):
    """Call once per task boundary. Replaces large, stale search results with a one-line extract."""
    for message in messages:
        if message["role"] != "user" or not isinstance(message["content"], list):
            continue
        for block in message["content"]:
            if not (isinstance(block, dict) and block.get("type") == "tool_result"):
                continue
            if tool_name_by_id.get(block.get("tool_use_id")) != "search_issues":
                continue
            result_text = block.get("content")
            if not isinstance(result_text, str) or len(result_text) <= threshold:
                continue
            if result_text.startswith(PRUNED):
                continue  # already pruned on an earlier boundary
            # 限制单行结果的长度，使提取内容保持简短
            first_line = result_text.split("\n", 1)[0].strip()[:200]
            refs = re.findall(r"#(\d+)", result_text)[:5]
            extract = f"{PRUNED} {first_line}"
            if refs:
                extract += " kept refs: " + " ".join("#" + r for r in refs)
            block["content"] = extract
```

### 批处理可以等待的工作

[Batch API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing) 对请求的每个令牌（包括缓存的令牌）提供 50% 折扣，代价是结果在 24 小时内的任何时间到达。将每个无人等待的请求通过批处理路由，其余的保留交互路径。对于无人值守的智能体工作，批处理是仅次于缓存的第二大免费杠杆：评估运行、回填以及计划作业，例如[令牌精简测量](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens)中问题分诊智能体的定期运行。它可以与本页上除交互性之外的所有内容结合使用，但不适用于 Claude Managed Agents 会话，后者在设计上就是交互式的（参见 [Claude Managed Agents 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#claude-managed-agents-pricing)）。

### 针对当前模型审计提示

每一代模型对提示的响应都不同，因此提示会积累为您不再使用的模型编写的文本。常见的情况是为弥补旧模型而添加的过于具体的指令："验证两次"、"尽最大可能彻底"、强制性的分步流程，或手工编写的推理草稿区。较新的模型会逐字遵循这些指令，产生额外的工具轮次和额外的书写，因此账单上升而准确率没有任何提升。针对您现在运行的模型审计提示，并在每次更换模型时再次审计，是一项免费收益。

审计只需一条命令。Claude Code 附带的 [Claude API skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill) 有一个 `prompt-audit` 命令，它读取项目的提示和请求代码，并报告哪些内容是为不同模型编写的。以下简短摘录展示了它针对包含这些模式的客服台提示和请求代码运行的情况：

```text wrap
$ claude
> /claude-api prompt-audit

10 findings. The three that matter most:

1. budget_tokens returns a 400 on Opus 5 (support_agent.py:10). This code
   does not run against its own target model. Everything else here is
   quality; this one is an outage.
2. The prompt contains two live contradictions. Line 15 says confirm before
   every refund, line 17 says process every eligible refund immediately.
   Line 19 asks for a complete recap *and* a three-sentence maximum.
3. The reasoning scaffold and the 6-step script fight the model rather than
   steer it. <scratchpad> + "reason step by step" is now a request
   parameter, not prose; the mandatory 6-step procedure plus "investigate
   fully even when the ticket looks simple" forces four tool calls on a
   "where's my package" ticket.
...
-After any refund or escalation, verify twice before submitting: re-fetch
-the order, re-check every figure in your reply against the fresh lookup,
-and review the reply a second time for errors.
+Before submitting a refund or an escalation, re-fetch the order and confirm
+every figure in your reply matches the fresh lookup.
```

该命令随后以 diff 形式提出其编辑（显示一个 hunk），并列出它有意保留不动的内容：退款窗口、语气要求和质量标准。您审查的是一个补丁，而不是重写。

效果是可测量的。在一项客服台评估[14](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)中，为 Claude Opus 4.8 编写的提示在 Claude Opus 5 上每张工单多花 36%，而准确率没有变化。对相同的提示运行审计后，Opus 5 既比未审计版本更便宜（便宜 14%），又更准确（97% 的工单，从 92% 上升，增益超出噪声范围）。在 Claude Sonnet 4.6 到 Claude Sonnet 5 的迁移中，审计在相同准确率下削减了 14%：

![散点图，客服台评估：旧提示在新模型上花费更多；审计后，它更便宜且同样准确](https://platform.claude.com/docs/images/cost-intel-prompt-audit.png)

两种过时文本的代价不同。新模型过于字面遵循的指令消耗金钱：删除"验证两次"将 Opus 5 的每工单成本削减了三分之一，删除"尽最大可能彻底"几乎同样多。不再适合模型的文本则消耗准确率：一个已退役的思考设置、相互矛盾的规则，以及与模型自身思考冲突的手工草稿区，在 Opus 5 上删除后各自恢复了 7 到 11 个百分点：

![按遗留模式的柱状图：过度遵循的指令消耗金钱；失效的设置和矛盾的规则消耗准确率](https://platform.claude.com/docs/images/cost-intel-prompt-audit-patterns.png)

相同的模式往往也出现在工具描述和 skill 中，它们同样值得审计。

## 以成本换取智能

这些杠杆决定单个模型在成本与智能之间的位置：模型选择、effort、以更高设置重新运行失败项，以及它在其中工作的预算和上限。从在当前模型上进行 effort 扫描开始（[调整 effort](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#tune-effort)）。按成本和能力从低到高，当前模型依次为 Claude Haiku 4.5、Claude Sonnet 5、Claude Opus 5 和 Claude Fable 5.1（前沿模型）；[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)有完整的阵容和价格。

### 按每任务成本比较模型

价格表是按每令牌编写的，而按每令牌计算，前沿模型看起来很贵：Claude Fable 5.1 的每令牌价格是 Claude Sonnet 5 的数倍。但您支付的是已完成的任务，因此请按每个已完成任务的成本比较模型。能力更强的模型用更少的工作完成任务：更少的轮次、更少的搜索、更少地重新读取自己的上下文，以及更少的回溯。每令牌的溢价常常被各方面工作量的减少所抵消。

Anthropic 在 SWE-bench Pro[3](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 子集上测量了这一点，按客户计费方式定价：

![散点图，SWE-bench Pro：Claude Fable 5.1 在 low effort 下比 Claude Sonnet 5 多解决 11 个百分点，每个已解决任务便宜 35%；Claude Opus 5 在 low effort 下更便宜](https://platform.claude.com/docs/images/cost-intel-cost-per-task.png)

Claude Fable 5.1 在 `low` effort 下以每个已解决任务 $0.54 解决了 88.6% 的任务，而 Claude Sonnet 5 在其默认值下以 $0.84 解决了 77.4%：多 11 个百分点，每个已解决任务便宜 35%，尽管每令牌价格高出五倍。不过它并不总是胜出。在同一子集上（两个模型都基本饱和，其分数与公开排行榜不可比），单独的 Claude Opus 5 在默认值下与单独的 Claude Fable 5.1 持平（91.7% 对 92.1%，在运行间噪声范围内），每个已解决任务便宜约 15%（$1.01 对 $1.19），而 Opus 5 在 `low` 下以 $0.25 解决了 84.0%。而在长研究循环上，前沿模型做的工作更多而不是更少：在 DeepResearch Bench II[7](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上，Fable 5.1 在 `low` 下得分比 Sonnet 5 高 10 个百分点（66% 对 56%），每任务成本约为四倍（$4.66 对 $1.20），因为它在更大的上下文上运行更长的研究循环。Claude Opus 5 在其默认值下按相同基准得分 71%，每任务 $6.71，高于 Fable 5.1 在其默认值下的表现（65%，$7.12），因此在研究上 Fable 5.1 也只有在 `low` 下才物有所值。

对于大多数智能体工作负载，从 Claude Fable 5.1 的 `low` effort 开始，并在它失手的地方提高 effort。按每令牌计算，它在未缓存输入上的成本是 Claude Opus 5 的两倍，但在缓存输入上只有一半（每百万 $0.25 对 $0.50），而在智能体循环中缓存输入是最大的一项。在[顾问策略](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#advisor-strategy-escalate-hard-decisions)中的编码基准上，Fable 5.1 在 `medium` 下与 Opus 5 在其默认值下持平，每次尝试的成本约为三分之一（$2.91 对 $8.50）。在 Chartography[13](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)（一个图表阅读基准）上，Fable 5.1 在 `low` 下得分 62.5，每张图表 $0.15，而 Opus 5 在 `low` 下得分 49，每张 $0.38。在 SWE-bench Pro 子集上，如前所述，Claude Opus 5 在其默认值下仍然是达到最高分的更便宜方式。在另一端，Claude Haiku 4.5 回答 GPQA Diamond[9](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 问题的每题成本约为 Opus 5 的十分之一，准确率为 63%，而 Opus 为 92%，并且在长编码任务上落后得更多。它适合具有可检查输出的高容量工作，而不是长智能体循环。

排名因工作负载而翻转，没有任何价格表能告诉您朝哪个方向。请在您自己的流量上按每个已完成任务的成本为每个候选模型定价，包括 Claude Opus 5 和降低 effort 的前沿模型。

为工作负载的尾部定价，而不是中位数：在您最难的十分之一任务上比较模型，而不是典型任务。在典型任务上，每个模型看起来都差不多，最便宜的看起来最好，但账单是由更便宜的模型失败的任务决定的，因为失败的任务仍然对其令牌计费，然后是重试，然后是失败在下游造成的任何代价。即使没有任何失败，尾部也是资金流向的地方。在一次 20 个问题的 WideSearch[1](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 运行中，两个问题承担了 43% 的支出：

![按成本排序的 20 个 WideSearch 问题柱状图：前两个承担 43% 的支出，最便宜的一半承担 10%](https://platform.claude.com/docs/images/cost-intel-tail.png)

[多模型策略](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#combine-models)的存在就是为了将前沿智能花在那个尾部上，而不为其余部分支付前沿费率。

### 升级模型

如果您落后一两个模型，最便宜的杠杆就是模型字符串。Anthropic 在 SWE-bench Pro[3](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 子集上通过相同的 harness 运行了最近的 Claude Opus、Claude Sonnet 和 Claude Fable 模型，每个都使用其出厂默认值并按标价定价，并在 Terminal-Bench 3[20](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上再次运行了 Opus 系列：

![两张每个已解决任务成本对已解决任务数的图表：在 SWE-bench Pro 上每个模型都解决了大多数任务，升级步幅很小；在 Terminal-Bench 3 上 Opus 阶梯从每个已解决任务 $183 降至 $63 再降至 $28](https://platform.claude.com/docs/images/cost-intel-upgrade-ladder.png)

Anthropic 对 Opus 系列各版本的每令牌定价相同，因此任何差异都来自每个模型每任务做了多少工作：按客户计费方式定价，Claude Opus 4.8 以每个已解决任务便宜 14% 的成本解决了与 Claude Opus 4.7 相同比例的任务，而 Claude Opus 5 随后以每个已解决任务贵 21% 的成本多解决了 12 个百分点的任务。Claude Opus 5 在 `low` effort 下在此基准上击败了 Opus 4.8 的默认值，每个已解决任务的成本约为其 30%，因此最便宜的升级是以较低设置使用新模型。Sonnet 5 的节省来自其较低的每令牌价格，这足以抵消它与 Sonnet 4.6 相比每任务使用的额外令牌：每个已解决任务便宜 15%，多 5 个百分点。前沿档位以同样的方式获益：Claude Fable 5.1 以每个已解决任务便宜 43% 的成本与 Claude Fable 5 的分数持平，其中大部分来自较低的缓存读取价格。这个方向并非必然：在 DeepResearch Bench II[7](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上，同样的升级在 `high` 下每任务贵 41%（在 `low` 下贵 79%），换来在每个分组中都干净的任务上多 2 到 3 个百分点（参考文献 7），因为新模型在那里每任务做的工作更多。输入和输出价格相同，缓存读取便宜 4 倍，因此在假设升级能省钱之前，请在您自己的工作负载上测量它。

在更难的工作上，差距会扩大。在 Terminal-Bench 3[20](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上，任务难到由通过率而非令牌决定账单，Claude Opus 4.7、Opus 4.8 和 Opus 5 每任务各花费 $8 到 $15，但分别解决了 7%、15% 和 41% 的任务，因此每个已解决任务的成本沿阶梯从 $183 降至 $63 再降至 $28。Claude Opus 5 在饱和的编码子集上相对 Opus 4.8 的 21% 溢价，在 Terminal-Bench 3 上变成了 56% 的节省，因为旧模型在那里大多失败：您的工作负载越能击败旧模型，升级在每个结果上节省的就越多。

按每个已解决任务的成本比较，而不是按每令牌：相同的文本在 Claude Opus 4.7 及更高版本上多消耗约 30% 的令牌，因此按每令牌比较会在构造上使较新的模型看起来更贵。

### 调整努力程度

"Effort"（努力程度）是针对您的任务调整模型的最直接方式。`effort` 参数控制模型进行多少思考、工具调用和自我验证，默认值（`high`）适合要求较高的任务。成本随所有这些活动而增长；而准确率只随您的任务真正需要的那部分而增长。在模型的能力上限之下，最高的努力程度级别是在为任务永远用不到的深度付费。

在研究和知识工作类基准测试上（WideSearch[1](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)、DeepWideSearch[6](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)、BrowseComp[4](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 和 GDPval[2](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)，均使用 Claude Fable 5），准确率相对于成本的曲线几乎是平的：`low` 以每任务成本降低三分之一到一半为代价放弃了 1 到 3 个点，`medium` 以默认值约 70% 到 87% 的成本达到了与默认值相同的准确率，而在这四项基准中的任何一项上，默认值相比 `medium` 都没有带来任何可测量的收益。在 DeepWideSearch 上，`low` 还以低 29% 的成本追平了一个使用 Claude Sonnet 5 工作器的编排器：降低努力程度胜过了架构变更。

较低的努力程度设置通常更快，当延迟是约束条件时这一点很重要。在这些运行中，`low` 在 DeepWideSearch 上每个问题耗时 4.5 分钟，而默认值为 7.9 分钟。在[语料库基准测试](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#orchestrator-strategy-delegate-bulk-work)上（其输入无法放入任何单个上下文窗口），Fable 5.1 在 `low`、`medium` 和 `high` 下每个回合分别耗时 15.2、17.5 和 19.9 小时。

长周期编码是努力程度真正能换来准确率的地方。在 SWE-bench Pro[3](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上，Claude Opus 5 在 `medium` 下以一半的成本放弃了约 2 个点，在 `low` 下以四分之一的成本放弃了约 8 个点：这是一个真实的权衡，而[以更高努力程度重新运行失败任务](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#re-run-failures-at-higher-effort)可以将其重新转化为节省。下图绘制了研究和知识工作类基准测试以及 SWE-bench Pro 上准确率相对于成本的关系：

![五项基准测试上按努力程度划分的准确率相对于成本的折线图：在四项研究任务上几乎持平，在 SWE-bench Pro 上陡峭](https://platform.claude.com/docs/images/cost-intel-effort-sweep.png)

由此得出两个结论。第一，在添加第二个模型之前，先为您自己的工作负载绘制这条曲线：在这些内部测量中，一个看起来比默认单模型更便宜的多模型配置，其成本高于同一模型在较低努力程度下的成本。第二，这条曲线是任何多模型策略都必须超越的单模型基线，因此[在您自己的工作负载上测量的第 2 步](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#measure-on-your-own-workload)会跨努力程度级别建立基线。

困难的工作并不自动需要高努力程度。在 DeepResearch Bench II[7](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上，Claude Fable 5.1 在 `low`、`medium` 和 `high` 下得分几乎相同，而每任务成本从 $4.66 上升到 $7.12，因此在这种情况下提高努力程度并不会明显提升输出质量；在每个实验组中都干净完成的 21 个任务上（参考文献 7），Claude Fable 5 在各努力程度下也是持平的，尽管图表的 33 任务基准（剔除了每个模型自身被中途截断的尝试）显示它在上升。请在您要发布的模型上测量这条曲线，而不是您上次测量的那个模型：

![DeepResearch Bench II 上评分标准得分相对于每任务成本的折线图：在 Claude Fable 5.1 上，更高的努力程度没有换来分数，只换来了成本](https://platform.claude.com/docs/images/cost-intel-effort-limit.png)

仅凭任务描述无法判断您的工作负载属于哪一类，因此请在您自己流量的样本上扫描两到三个努力程度级别，并从曲线上读出答案。在单独的会话中测试每个级别：在会话中途更改顶层努力程度会使缓存失效（参见[缓存重复的上下文](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#cache-repeated-context)）并扭曲比较结果。有关参数详情，请参阅[努力程度](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)。

### 以更高努力程度重新运行失败任务

当任务的结果可以检查时，努力程度曲线上最便宜的策略不是一个固定设置：以低设置运行每个任务，然后仅以更高设置重新运行失败的任务。

Anthropic 根据[调整努力程度](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#tune-effort)中 SWE-bench Pro[3](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 子集上的努力程度运行结果，逐任务计算了这一策略。使用 Claude Opus 5 在 `low` 下，16% 的任务失败；将这些任务以默认值重新运行后，约 93% 通过，每个任务约 $0.45，而全部以默认值运行则为 91.7% 通过、每个任务 $0.93：相同的通过率，成本减半，且已计入失败的廉价尝试。若改为从 `medium` 开始，则以约 $0.61 解决了约 94%。这一小幅提升大部分来自第二次尝试（以默认值重新运行默认值自身的失败任务得分大致相同，但花费更多），因此使用此策略是为了节省，而不是为了提升：

![图表，SWE-bench Pro：以 low 或 medium 运行并以默认值重新运行失败任务，在成本上胜过每一个固定努力程度设置](https://platform.claude.com/docs/images/cost-intel-escalation.png)

有两个适用条件。第一，您需要一个失败信号（此处为基准测试自身的测试）；一个会放过糟糕工作的检查器会让这些失败漏过去。第二，每个首轮失败都需要两次运行的挂钟时间，因此节省是以失败任务上的延迟为代价换来的。

### 设置预算和输出上限

大多数智能体任务运行都很便宜，但少数运行会在搜索、重复验证和过度测试上花费中位成本的许多倍。[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)针对的就是这个长尾。模型会看到整个任务的实时令牌倒计时并自我调节，削减低价值的搜索、跳过冗余验证，并及时收尾而不是陷入螺旋。

Anthropic 在 SWE-bench Pro[3](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上使用 Claude Fable 5.1 测量了随预算收紧时的通过率和每任务成本：

![SWE-bench Pro 上的折线图：随着任务预算收紧，pass@1 下降几个点，而每任务成本下降 44% 到 58%](https://platform.claude.com/docs/images/cost-intel-budget-pareto.png)

宽松的预算将每任务成本削减了 44%，代价是约 3 个点的通过率，处于运行间噪声的边缘；而允许的最紧预算将其削减了 58%，代价是 6 个点。预算在这里换来了效率，其通过率代价随预算收紧而增长。

三种控制手段承担三种不同的职责。任务预算能省钱，因为模型能看到它。`max_tokens` 是一个安全上限：降低它会削减每次尝试的成本，但不会降低每个已解决任务的成本。在 Claude Managed Agents 上，会话预算是两者背后的硬性美元止损。三者都要设置：一个任务预算、一个较高的 `max_tokens`，以及一个针对您永远不想在账单上看到的运行的会话上限，并以[工作区支出限额](https://platform.claude.com/docs/zh-CN/api/rate-limits#setting-lower-limits-for-workspaces)作为最后的兜底。

* **任务预算**在最新模型上处于 beta 阶段（beta 标头 `task-budgets-2026-03-13`）；请查看[支持表](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#feature-support)了解具体哪些模型。从接近您循环的第 90 百分位令牌用量开始，然后收紧（[选择预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets#choosing-a-budget)展示了如何收集该分布）。低于当前 20,000 令牌下限的预算会被拒绝，而非常紧的预算可能产生类似拒绝的行为。在第一个请求上一次性设置预算，因为任务中途更改会[使缓存失效](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#cache-repeated-context)。预算是建议性的，引导模型而非阻止模型，因此请在您的工作负载上验证其遵守情况。
* **`max_tokens`** 对单个响应设置上限，且对模型不可见，因此降低它不会让模型节约。需要这些空间的轮次会被丢弃但仍然计费。在一个内部代码仓库任务基准测试[12](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)上，16,384 令牌的上限在默认努力程度下终止了 Claude Opus 5 15% 的尝试和 Claude Fable 5.1 43% 的尝试，而 117 次被截断的 Fable 尝试中只有 9 次仍然通过。被截断的运行每次尝试花费更少，但换来的解决数量也成比例减少，因此每个已解决任务的成本与 64,000 时大致相同（$21 对 $22）。在 64,000 时，默认努力程度下约 14,000 个轮次中仍有 2 个被截断，而 Fable 5.1 解决了 58.5% 的任务而非 36.3%（在 SWE-bench Pro[3](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 子集的另一个切片上，如参考文献 12 所述，没有差异：两种上限下均为 100 个中的 94 个）。重试被截断的尝试很少有帮助：在相同上限下它们大多会再次失败，而在更高上限下您还要为浪费的尝试付费。对于智能体工作，将 `max_tokens` 设置为 64,000；当单次被截断的尝试代价高昂时，设置为最大值 128,000；在 128,000 时，Fable 5.1 以相同的每已解决任务成本解决了 60.0%。对这么大的响应使用[流式传输](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)，将 [`stop_reason: max_tokens`](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons#max-tokens) 视为失败，并通过模型可见的努力程度和任务预算来省钱。
* **Claude Managed Agents 上的会话预算**是硬性止损。[会话预算](https://platform.claude.com/docs/zh-CN/managed-agents/budgets)是按令牌、搜索和会话时间的标价对单个会话设置的美元上限。达到上限时，会话以 `stop_reason: budget_reached` 暂停；提高预算即可恢复。它由平台强制执行，适用于任何有标价的模型，包括任务预算尚不可用的模型，并可与建议性的任务预算结合使用。部署会将同一字段应用于每次运行。

要求更短的回答。在 Claude Sonnet 5 上，输出令牌的成本是输入令牌的五倍，而在智能体循环中，模型写出的每个令牌都会在之后的每一轮作为输入返回，因此您会为一个长回答一次又一次地付费。Anthropic 在三种最终回答指令下运行了分诊任务，每种运行三次，使用相同的模型和工具。原始指令要求两行：

```text wrap
4. Finish with exactly two lines:
LABEL: <one of: bug-confirmed, needs-more-info, duplicate-candidate, feature-request, upstream-issue, perf, ui-polish>
SUMMARY: <one or two sentences for the engineering team>
```

较短的变体要求一行：

```text wrap
4. Finish with exactly one line in this form:
DECISION | LABEL | REASON
where DECISION is one of: triage-now, needs-info, close-duplicate; LABEL is one of: bug-confirmed, needs-more-info, duplicate-candidate, feature-request, upstream-issue, perf, ui-polish; REASON is one clause under 15 words. Output nothing after that line.
```

较长的变体要求一份包含五个带标题小节的备忘录：问题摘要、证据、重复检查、建议标签和后续步骤。对于其中一个 issue（一个在跳过问题后永远不会发送的排队提示），前两种回答分别是：

```text wrap
LABEL: bug-confirmed
SUMMARY: When a user submits a new prompt instead of answering an agent's pending question, the question is cancelled/skipped but the new prompt remains stuck in "QUEUED" state indefinitely since it's waiting on a response to the now-cancelled question; the queued prompt should be processed immediately after cancellation.
```

```text wrap
triage-now | bug-confirmed | Clear repro steps show prompt queues indefinitely after cancelled question.
```

![柱状图：单行格式每次运行 $0.49，原始两行格式 $0.57，备忘录 $1.40，正确率均为 78% 到 85%](https://platform.claude.com/docs/images/cost-intel-output-format.png)

单行回答比两行原始格式少用了 39% 的输出令牌，每次运行成本低 14%。备忘录使用了六倍的输出令牌，成本是单行回答的 2.8 倍。三者对照黄金标签的得分都在彼此的运行间噪声范围内，因此这些格式在您支付的费用上的差异远大于它们在正确率上的差异。要求您会去读的回答，而不是看起来很周全的回答。

在较低的 `max_tokens` 上限下，两个模型每次尝试花费更少，但解决的任务也成比例减少，因此每个已解决任务的成本几乎不变：

![柱状图：在 16k 上限下，两个模型每次尝试花费更少，但每个已解决任务的花费与 64k 时大致相同，因为它们解决的任务更少](https://platform.claude.com/docs/images/cost-intel-max-tokens-saving.png)

几乎每一轮都在远低于任一上限的位置结束。罕见的长轮次正是更高上限所换来的：

![Opus 5 和 Fable 5.1 每轮输出的点图：中位数为几百个令牌，最长轮次为 33k 和 128k，对照各上限](https://platform.claude.com/docs/images/cost-intel-max-tokens-ladder.png)

## 组合模型

多模型架构适合任务复杂度变化足够大、以至于不同步骤最好由不同模型处理的工作负载。当您的流量混合了较小模型能可靠处理的常规工作与需要前沿能力的较难步骤时，拆分工作可以将前沿智能保留在重要之处，同时大多数令牌按较小模型的费率计费。当工作负载缺乏这种混合（因为其难度均匀，或者它是一条相互依赖的链）时，单个调优良好的模型通常是更好的选择。每个策略小节都给出了区分这两种情况的规则。

两种策略覆盖了大多数工作负载，它们的区别在于哪个模型持有主循环：

| 策略                    | 控制流              | 前沿模型的角色     | 适合                                   | 前沿成本随之增长的因素 |
| --------------------- | ---------------- | ----------- | ------------------------------------ | ----------- |
| **顾问（Advisor）**       | 较小模型运行循环，按需升级    | 被咨询以获取计划和纠正 | 局部困难的串行工作，例如编码智能体在少数真正决策之间的许多轮次      | 执行器卡住的频率    |
| **编排器（Orchestrator）** | 前沿模型运行循环，委派大批量工作 | 规划、分派和综合    | 可扇出到真正独立的文件、文档或案例的工作，尤其是超过一个上下文窗口的工作 | 各部分协调的难度    |

### 顾问策略：升级困难决策

在顾问策略中，一个成本较低的执行器（executor）模型运行智能体循环并执行大多数轮次。当它遇到需要更深判断的决策时，例如选择方法或从失败中恢复，它会调用一个智能更高的顾问（advisor）模型获取战略指导，然后继续。大多数令牌按执行器费率计费，只有偶尔的咨询按顾问费率计费。

要使用它，请将[顾问工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool)添加到您的请求中。这个 beta 功能在一个 `/v1/messages` 请求中于服务器端运行整个策略：执行器发出工具调用，Anthropic 运行顾问推理，执行器带着建议继续；您无需编写任何编排代码。在 Claude Managed Agents 上，通过在智能体的 `multiagent` 名册中添加一个 `advisor` 条目来[为会话提供顾问](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#give-the-session-an-advisor)；会话的主线程以同样的方式咨询它。Claude Code 也支持它；请参阅[使用顾问工具升级困难决策](https://code.claude.com/docs/zh-CN/advisor)。

![顾问策略示意图：一个执行器模型运行主循环，并按需调用 Claude Fable 5.1 顾问](https://platform.claude.com/docs/images/model-routing-advisor-strategy.png)

**决定回报的因素。** 顾问只能通过执行器的调用看到任务，因此有两件事决定它能帮多少忙。

第一是模型之间的差距。顾问只能交付执行器所缺乏的能力：在 GPQA Diamond[9](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上，Claude Haiku 4.5 执行器从 Claude Opus 5 顾问那里获益巨大，Claude Sonnet 5 执行器获益几个点，而前沿执行器几乎没有获益。

第二，也是脆弱的一点，是执行器是否真的会去问（咨询率，consult rate）。低努力程度下的执行器可能不再察觉自己卡住了：一个在默认努力程度下对大多数任务进行咨询的配对，在降低努力程度后可能跌至几乎不咨询，然后得分低于执行器单独运行。该比率也因任务而异：在 DeepSWE[10](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上，低努力程度的 Sonnet 5 执行器持续提问并获益 23 个点；在 SWE-bench Pro[3](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上，同一执行器停止了提问。当执行器确实提问时，它能弥补大部分差距。在下图中执行器持续提问的各配对中，顾问至少弥合了与更强模型之间一半的差距（编码配对直接胜过了更强模型），而您只在咨询时为更强模型付费，这正是成本案例得以成立的原因：

![六个顾问配对的柱状图，适用处以 Claude Fable 5.1 作为顾问：可用差距与实现的收益，标注了咨询率，收益与咨询率同步](https://platform.claude.com/docs/images/cost-intel-advisor-mechanism.png)

咨询率对提示有响应。仅凭工具的内置描述，执行器会调用不足，尤其是在编码工作上，因此[顾问工具文档](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool#prompting-for-coding-and-agent-tasks)给出了一个系统提示，要求在实质性工作之前调用一次、在完成之前调用一次，每个任务约两到三次调用。接下来测量的编码配对以该节奏运行，每个任务约两次咨询。该页面还涵盖了如何推动调用不足的执行器，以及如何在客户端限制调用次数以控制成本。因此请关注咨询率：为它编写提示、测量它，并在它崩塌时恢复执行器的努力程度。

**何时在成本上划算。** 当少数几次按顾问费率计费的简短咨询取代了在整个任务中运行顾问模型时，顾问就能省钱。当顾问模型的定价远高于执行器时效果最好，因此最具成本效益的配置是前沿顾问搭配中端执行器。即使在范围顶端，配对也能站得住脚，因为建议还能节省执行器令牌：被告知正确方法的执行器会探索更少的死胡同，这可以抵消咨询的费用。

在一个内部智能体编码基准测试[11](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)上，使用普通 API 智能体运行，Claude Opus 5 执行器搭配 Claude Fable 5.1 顾问是所测量的最准确配置，每次尝试 $7.69。它位于穿过每个模型自身努力程度设置的连线之上：以略少的花费比默认设置下单独的 Opus 5 高 3.5 个点（每任务五次尝试确实将这一差距与噪声区分开来），并以约多一半的花费比单独的顾问模型高约 2.5 个点：

![图表，编码基准测试：两个模型的努力程度曲线，Opus 5 加 Fable 5.1 顾问配对比单独的 Opus 5 高 3.5 个点，比单独的 Fable 5.1 高约 2.5 个点](https://platform.claude.com/docs/images/cost-intel-internal-coding-advisor.png)

早先通过 [Claude Code 的顾问模式](https://code.claude.com/docs/zh-CN/advisor)进行的测量产生了相同的排序。请将此结果视为一个需要在您的工作负载上测试的形态：顾问以大约执行器自身的价格换来几个点。更大的能力差距并不保证更划算。延迟成本就是咨询本身：在此基准测试上每个任务约两次额外的前沿模型调用，每次都在任务的关键路径上。

**何时单独使用更强模型是更好的一步。** 当工作负载的准确率对努力程度有响应时，在构建配对之前，先将其与顾问模型在降低设置下单独运行进行比较：顾问只在需要它的任务上付费，但一个在大多数任务上都触发的咨询比直接运行更强模型本身花费更多。在 Chartography[13](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 上，同一配对在运行间噪声范围内追平了 `medium` 下单独的 Claude Fable 5.1（65.0 对 67.5），每任务成本约为其 2.6 倍，因为几乎每个任务都咨询了顾问。先测量您自己的咨询率：如果执行器在大多数任务上都提问，您就是在整个工作负载上支付顾问费率，而直接运行顾问模型本身是达到相同分数的更便宜方式。

无论何种配对，先为顾问模型在低努力程度下单独运行定价；那是要超越的基线。在每次模型发布时重新检查，因为发布会同时改变能力差距和价格比。

**何时适合。** 顾问策略适合轮次大多是机械性的、但一个出色的计划很重要的工作负载：编码智能体、计算机使用和多步骤研究流水线。当每一轮都真正需要前沿能力、当没有什么可规划的（单轮问答），或者当您的执行器已经接近顾问的能力时，它就不太适合。

### 编排器策略：委派大批量工作

在编排器策略中，前沿模型持有循环。它分解任务，将子任务分派给成本较低的工作器（worker）模型，并合并它们的结果。编排器自身的对话记录保持简短，因为工作器吸收了令牌密集的探索，因此大多数令牌按工作器费率计费，而计划和综合仍来自前沿模型。

要构建一个编排器，请使用 Claude Managed Agents 中的[多智能体编排](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)：配置一个协调器智能体（即编排器）和一个工作器智能体名册，每个都有自己的模型。有关使用前沿协调器和 Claude Sonnet 5 工作器的完整可运行示例，请参阅 Claude Cookbook 配方[协调器模式：大模型用于规划，小模型用于执行](https://github.com/anthropics/claude-cookbooks/blob/main/managed_agents/CMA_plan_big_execute_small.ipynb)。

![编排器策略示意图：一个 Claude Fable 5.1 编排器将子任务扇出到三个 Claude Sonnet 5 工作器](https://platform.claude.com/docs/images/model-routing-orchestrator-strategy.png)

当工作器可以并行运行时，此模式可节省挂钟时间：在语料库基准测试[8](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)上，协调器以平台文档规定的 25 个并发工作器上限运行时，一个回合约耗时 2.3 小时，而单独运行则为 15 到 20 小时。它只在两种测量到的情形下省了钱。在单个模型可以独自处理的工作上，同一模型在较低努力程度下每次都更便宜。

**情形 1：针对常规工作成本长尾的保险。** 单独运行的前沿模型偶尔会在它通常能解决的常规问题上陷入螺旋。由于您无法提前知道会是哪些问题，少数这样的运行会主导账单。将常规工作交给成本较低的工作器的协调器可以封住这个长尾，因为任何螺旋现在都按工作器费率发生。

Anthropic 在 BrowseComp[4](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 的一个刻意简单的切片上测量了这一点（10 个单独模型能可靠解决的问题；50 次委派运行和 70 次单独运行）。一个 Claude Fable 5 协调器搭配一个 Claude Sonnet 5 工作器，平均成本约为单独 Claude Fable 5 的一半，在第 90 百分位约为三分之一（$12 对 $33），而单独模型最昂贵的单次运行花费 $84，且结果还是错的：

![点图，BrowseComp 常规切片：委派运行的平均成本约为单独 Claude Fable 5 的一半，在第 90 百分位为三分之一](https://platform.claude.com/docs/images/cost-intel-tail-insurance.png)

委派在常规的、通常可解决的那部分工作上划算，这与"工作器是用来处理难题的"这一直觉相反。在完整的、更难的 BrowseComp 集合上，经济性发生了逆转。如果您的流量在常规任务上有很长的成本长尾，这就是首先要测量的编排器情形。

**情形 2：大于一个上下文窗口的工作。** 单独模型必须串行处理如此大的输入，一次一个上下文窗口，并在每一遍都为重新读取自身状态付费。工作器各自读取自己的分区，并行且按工作器费率。仍能放入一个上下文窗口的阅读密集型工作是模型选择问题，而非委派问题：仅就阅读成本而言，只有当没有单个上下文能容纳工作时，编排器才会胜出。

Anthropic 为此情形构建了一个基准测试[8](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)：一个由 14 个公开 Python 包组成的 2160 万令牌语料库，其中植入了 130 个缺陷，大到任何上下文窗口都放不下。降低努力程度无济于事，因为账单就是语料库读取本身：Claude Fable 5.1 单独运行在三种努力程度设置下每回合成本为 $468 到 $552，只有其准确率发生了变化。协调器配置（一个 Claude Fable 5.1 主导者带领 25 个 Claude Sonnet 5 工作器）的成本约为这些设置的一半（低 47% 到 55%），得分比它们低 10 到 12 个点，每回合约 2.3 小时而非 15 到 20 小时，同时直接胜过了 Claude Sonnet 5 单独运行的基线：

![图表，语料库基准测试：协调器的成本约为任何努力程度下 Fable 5.1 单独运行的一半，比其最佳成绩低约 12 个点](https://platform.claude.com/docs/images/cost-intel-corpus-pareto.png)

令牌核算显示了阅读的规模：协调器配置每回合读取约 5.6 亿个缓存令牌，约为单独模型大约 3.65 亿的一倍半，几乎全部按 Claude Sonnet 5 的缓存读取费率计费，而总体成本仍约为一半。`high` 努力程度下的 Fable 5.1 仍保持峰值准确率，成本约为协调器配置的 2.2 倍，因此这里的委派换来了大部分准确率，而非全部。

**何时委派不划算。** 只有当有大批量工作可以交出去时，编排器才能换来东西：许多独立的部分，理想情况下多到一个上下文窗口放不下。当工作是一条相互依赖的链，或能放入单个上下文时，编排器要为计划、交接和合并付费，而单个模型免费获得这些。在每一个测量到的此类情形中，协调器的模型在较低努力程度下单独运行都胜出了。

边界在于任务难度，而非基准测试：在完整的、更难的 BrowseComp[4](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs) 集合上，单独的 Claude Fable 5 以低 22% 到 30% 的成本达到了协调器配置的准确率。独立的外部工作报告了相同的模式[5](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#refs)。如果工作是一条链、能放入一个上下文且没有长成本长尾，或者单个模型在较低努力程度下已经达到您的标准，就不要构建编排器。

### 在两种策略之间选择

大多数情形归结为一个问题：工作是拆分为独立的部分，还是通过一连串相互依赖的步骤得出一个答案？[策略表](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#combine-models)将这两种答案映射到两种策略。

如果您不确定，先什么都不要构建：

1. 先在您当前的模型上扫描努力程度。这是本页最便宜的实验，大多数工作负载到此为止。
2. 如果扫描显示存在差距，为更强模型在低努力程度下单独运行定价。那是顾问配对必须超越的数字，而本页上超越它的配对都是执行器确实进行了咨询的那些。

本页的多模型结果是对照同一模型在较低努力程度下以及对照下一级模型单独运行来评判的。那是要在您自己的工作负载上运行的比较，也是第一步是努力程度扫描的原因。

当您确实添加顾问时，它是一个工具定义，而非一次架构重构。

## 在您自己的工作负载上测量

本页的数字反映的是测量时的标价，并会随模型和价格变化而漂移。您的升级率、任务拆分的干净程度以及对话记录长度也会影响它们。方法保持不变：

1. 从生产日志中抽取一些任务，按真实流量加权，并为每个任务[编写结果检查](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests)：测试通过、工单关闭、行数正确。在分数旁记录每任务成本：按各自费率为每个响应 `usage` 中的五种计价令牌数定价（未缓存输入、按输入价格 1.25 倍和 2 倍计的 5 分钟和 1 小时缓存写入、缓存读取以及输出），并在任务的各请求间求和（[用量与成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api) 报告汇总值）。
2. 跨努力程度级别（而不仅是默认值）为各模型层级建立基线，并绘制分数相对于支出的图。多模型配置必须超越单模型的整条曲线。
3. 如果曲线显示存在努力程度无法弥合的差距，添加适合的多模型策略并重新运行测试套件。
4. 在切换之前，在一个流量切片上以影子模式运行胜出者，然后保持测试套件持续运行。

以下示例按 Claude Opus 5 的标价计算一个请求的第 1 步成本：

<CodeGroup>
  ```bash cURL
  # 每百万令牌价格取自定价页面；如需其他模型，请修改这三个值。
  INPUT_PER_MTOK=5.00 # Claude Opus 5
  CACHE_READ_PER_MTOK=0.50 # 0.1x the input price; 0.025x on Claude Fable 5.1 and Claude Mythos 5.1
  OUTPUT_PER_MTOK=25.00

  response=$(curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello, Claude"}]
    }')

  cost=$(jq -r --argjson in_price "$INPUT_PER_MTOK" --argjson read_price "$CACHE_READ_PER_MTOK" --argjson out_price "$OUTPUT_PER_MTOK" '
    .usage
    | (.input_tokens * $in_price
       + (.cache_creation.ephemeral_1h_input_tokens // 0) * $in_price * 2.00  # 1-hour cache write
       + (.cache_creation.ephemeral_5m_input_tokens // 0) * $in_price * 1.25  # 5-minute cache write
       + (.cache_read_input_tokens // 0) * $read_price                   # cache read
       + .output_tokens * $out_price) / 1e6
  ' <<<"$response")
  printf 'Request cost: $%.6f\n' "$cost"
  ```

  ```bash CLI
  # 每百万令牌价格取自定价页面；如需其他模型，请修改这三个值。
  INPUT_PER_MTOK=5.00 # Claude Opus 5
  CACHE_READ_PER_MTOK=0.50 # 0.1x the input price; 0.025x on Claude Fable 5.1 and Claude Mythos 5.1
  OUTPUT_PER_MTOK=25.00

  USAGE=$(ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello, Claude"}' \
    --transform usage)

  COST=$(jq -r --argjson in_price "$INPUT_PER_MTOK" --argjson read_price "$CACHE_READ_PER_MTOK" --argjson out_price "$OUTPUT_PER_MTOK" '
    (.input_tokens * $in_price
      + (.cache_creation.ephemeral_1h_input_tokens // 0) * $in_price * 2.00  # 1-hour cache write
      + (.cache_creation.ephemeral_5m_input_tokens // 0) * $in_price * 1.25  # 5-minute cache write
      + (.cache_read_input_tokens // 0) * $read_price                   # cache read
      + .output_tokens * $out_price) / 1e6
  ' <<<"$USAGE")
  printf 'Request cost: $%.6f\n' "$COST"
  ```

  ```python Python
  # 来自定价页面的每百万令牌价格；如需用于其他模型，请修改这三个值。
  INPUT_PER_MTOK = 5.00  # Claude Opus 5
  # 输入价格的 0.1 倍；在 Claude Fable 5.1 和 Claude Mythos 5.1 上为 0.025 倍
  CACHE_READ_PER_MTOK = 0.50
  OUTPUT_PER_MTOK = 25.00

  client = anthropic.Anthropic()
  response = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello, Claude"}],
  )
  usage = response.usage
  cache_writes = usage.cache_creation
  writes_1h = cache_writes.ephemeral_1h_input_tokens if cache_writes else 0
  writes_5m = cache_writes.ephemeral_5m_input_tokens if cache_writes else 0
  cost = (
      usage.input_tokens * INPUT_PER_MTOK
      # 1 小时缓存写入按输入价格的 2 倍计费，5 分钟按 1.25 倍；读取按缓存读取价格计费。
      + writes_1h * INPUT_PER_MTOK * 2.0
      + writes_5m * INPUT_PER_MTOK * 1.25
      + (usage.cache_read_input_tokens or 0) * CACHE_READ_PER_MTOK
      + usage.output_tokens * OUTPUT_PER_MTOK
  ) / 1_000_000
  print(f"Request cost: ${cost:.6f}")
  ```

  ```typescript TypeScript
  // 每百万令牌价格取自定价页面；如需其他模型，请修改这三个值。
  const INPUT_PER_MTOK = 5.0; // Claude Opus 5
  const CACHE_READ_PER_MTOK = 0.5; // 0.1x the input price; 0.025x on Claude Fable 5.1 and Claude Mythos 5.1
  const OUTPUT_PER_MTOK = 25.0;

  const client = new Anthropic();
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }]
  });
  const usage = response.usage;
  const cost =
    (usage.input_tokens * INPUT_PER_MTOK +
      (usage.cache_creation?.ephemeral_1h_input_tokens ?? 0) * INPUT_PER_MTOK * 2 + // 1-hour cache write
      (usage.cache_creation?.ephemeral_5m_input_tokens ?? 0) * INPUT_PER_MTOK * 1.25 + // 5-minute cache write
      (usage.cache_read_input_tokens ?? 0) * CACHE_READ_PER_MTOK + // cache read
      usage.output_tokens * OUTPUT_PER_MTOK) /
    1_000_000;
  console.log(`Request cost: $${cost.toFixed(6)}`);
  ```

  ```csharp C#
  // 每百万令牌价格取自定价页面；如需其他模型，请修改这三个值。
  const double InputPerMtok = 5.00; // Claude Opus 5
  const double CacheReadPerMtok = 0.50; // 0.1x the input price; 0.025x on Claude Fable 5.1 and Claude Mythos 5.1
  const double OutputPerMtok = 25.00;

  AnthropicClient client = new();
  var response = await client.Messages.Create(
      new MessageCreateParams
      {
          Model = Model.ClaudeOpus5,
          MaxTokens = 1024,
          Messages = [new() { Role = Role.User, Content = "Hello, Claude" }],
      }
  );
  var usage = response.Usage;
  double cost =
      (
          usage.InputTokens * InputPerMtok
          + (usage.CacheCreation?.Ephemeral1hInputTokens ?? 0) * InputPerMtok * 2.00 // 1-hour cache write
          + (usage.CacheCreation?.Ephemeral5mInputTokens ?? 0) * InputPerMtok * 1.25 // 5-minute cache write
          + (usage.CacheReadInputTokens ?? 0) * CacheReadPerMtok // cache read
          + usage.OutputTokens * OutputPerMtok
      ) / 1_000_000;
  Console.WriteLine($"Request cost: ${cost:F6}");
  ```

  ```go Go
  // 每百万令牌价格取自定价页面；如需其他模型，请修改这三个值。
  const (
  	inputPerMTok     = 5.00 // Claude Opus 5
  	cacheReadPerMTok = 0.50 // 0.1x the input price; 0.025x on Claude Fable 5.1 and Claude Mythos 5.1
  	outputPerMTok    = 25.00
  )

  // ...
  	client := anthropic.NewClient()

  	response, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeOpus5,
  		MaxTokens: 1024,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hello, Claude")),
  		},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	usage := response.Usage
  	cost := (float64(usage.InputTokens)*inputPerMTok +
  		float64(usage.CacheCreation.Ephemeral1hInputTokens)*inputPerMTok*2.00 + // 1-hour cache write
  		float64(usage.CacheCreation.Ephemeral5mInputTokens)*inputPerMTok*1.25 + // 5-minute cache write
  		float64(usage.CacheReadInputTokens)*cacheReadPerMTok + // cache read
  		float64(usage.OutputTokens)*outputPerMTok) / 1_000_000
  	fmt.Printf("Request cost: $%.6f\n", cost)
  ```

  ```java Java
  // 每百万令牌价格取自定价页面；如需其他模型，请修改这三个值。
  static final double INPUT_PER_MTOK = 5.00; // Claude Opus 5
  static final double CACHE_READ_PER_MTOK = 0.50; // 0.1x the input price; 0.025x on Claude Fable 5.1 and Claude Mythos 5.1
  static final double OUTPUT_PER_MTOK = 25.00;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      Message response = client.messages().create(MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .addUserMessage("Hello, Claude")
          .build());

      Usage usage = response.usage();
      long writes1h = usage.cacheCreation().map(CacheCreation::ephemeral1hInputTokens).orElse(0L);
      long writes5m = usage.cacheCreation().map(CacheCreation::ephemeral5mInputTokens).orElse(0L);
      double cost = (usage.inputTokens() * INPUT_PER_MTOK
          + writes1h * INPUT_PER_MTOK * 2.00 // 1-hour cache write
          + writes5m * INPUT_PER_MTOK * 1.25 // 5-minute cache write
          + usage.cacheReadInputTokens().orElse(0L) * CACHE_READ_PER_MTOK // cache read
          + usage.outputTokens() * OUTPUT_PER_MTOK) / 1_000_000;
      IO.println("Request cost: $%.6f".formatted(cost));
  }
  ```

  ```php PHP
  // 每百万令牌价格取自定价页面；如需用于其他模型，请修改这三个值。
  const INPUT_PER_MTOK = 5.00; // Claude Opus 5
  const CACHE_READ_PER_MTOK = 0.50; // 0.1x the input price; 0.025x on Claude Fable 5.1 and Claude Mythos 5.1
  const OUTPUT_PER_MTOK = 25.00;

  $client = new Client();
  $response = $client->messages->create(
      model: 'claude-opus-5',
      maxTokens: 1024,
      messages: [['role' => 'user', 'content' => 'Hello, Claude']],
  );
  $usage = $response->usage;
  $cost = (
      $usage->inputTokens * INPUT_PER_MTOK
      + ($usage->cacheCreation?->ephemeral1hInputTokens ?? 0) * INPUT_PER_MTOK * 2.00 // 1-hour cache write
      + ($usage->cacheCreation?->ephemeral5mInputTokens ?? 0) * INPUT_PER_MTOK * 1.25 // 5-minute cache write
      + ($usage->cacheReadInputTokens ?? 0) * CACHE_READ_PER_MTOK // cache read
      + $usage->outputTokens * OUTPUT_PER_MTOK
  ) / 1_000_000;
  printf("Request cost: \$%.6f\n", $cost);
  ```

  ```ruby Ruby
  # 每百万令牌价格取自定价页面；如需其他模型，请修改这三个值。
  INPUT_PER_MTOK = 5.00 # Claude Opus 5
  CACHE_READ_PER_MTOK = 0.50 # 0.1x the input price; 0.025x on Claude Fable 5.1 and Claude Mythos 5.1
  OUTPUT_PER_MTOK = 25.00

  client = Anthropic::Client.new
  response = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }]
  )
  usage = response.usage
  cost = (
    usage.input_tokens * INPUT_PER_MTOK +
    usage.cache_creation&.ephemeral_1h_input_tokens.to_i * INPUT_PER_MTOK * 2.00 + # 1-hour cache write
    usage.cache_creation&.ephemeral_5m_input_tokens.to_i * INPUT_PER_MTOK * 1.25 + # 5-minute cache write
    usage.cache_read_input_tokens.to_i * CACHE_READ_PER_MTOK + # cache read
    usage.output_tokens * OUTPUT_PER_MTOK
  ) / 1_000_000
  puts format("Request cost: $%.6f", cost)
  ```
</CodeGroup>

在智能体循环中，缓存读取项通常是五项中最大的；如果不是，请检查缓存是否已启用。当启用了[顾问工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool#usage-and-billing)或[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction#understanding-usage)时，部分令牌仅在 `usage.iterations` 中报告，而不在顶层总计中，因此请改为对 `usage.iterations` 求和，并按顾问模型的费率为 `advisor_message` 条目定价。

下表按尝试顺序列出了各项手段：

| 手段              | 这些运行中的节省                                                                                                                                                                                  | 质量代价                         | 延迟          | 位置                                                                                                                                                    |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| 提示缓存            | 智能体循环上成本降低 2.7 到 5.3 倍；分诊运行上降低 83%                                                                                                                                                        | 无                            | 更快          | [缓存重复的上下文](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#cache-repeated-context)                    |
| 1 小时缓存时长        | 一旦约每 20 轮中有 1 轮跟在 5 分钟到一小时之间的暂停之后、且很少有间隔超过一小时，就比 5 分钟默认值更便宜；Claude Fable 5.1 除外，在其上当暂停为几分钟时保持 5 分钟缓存温热更便宜，而当暂停接近一小时时 1 小时时长胜出；在没有暂停时，默认值在 Claude Sonnet 5 上便宜 15%，在 Claude Opus 5 上便宜 11% | 无                            | 暂停后保持温热     | [选择缓存时长](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#pick-the-cache-duration)                     |
| 输入精简            | 分诊运行上再降 5 个百分点                                                                                                                                                                            | 无                            | 中性          | [精简输入和上下文令牌](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens)           |
| 在任务边界修剪过时的工具结果  | 长分诊运行上 39%（压缩为 32%）；短循环上无                                                                                                                                                                 | 未测量到                         | 中性          | [精简输入和上下文令牌](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens)           |
| 工具搜索            | 附加 500 个工具定义时 45%；使用 GitHub MCP 服务器时 20%                                                                                                                                                  | 无                            | 中性          | [精简输入和上下文令牌](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens)           |
| 通过代码执行处理数据文件    | 25 个问题的数据任务上 92%                                                                                                                                                                          | 有提升，25 中 25 而非 25 中 6        | 更快          | [精简输入和上下文令牌](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens)           |
| Batch API       | 50%                                                                                                                                                                                       | 无                            | 24 小时内出结果   | [批处理可以等待的工作](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#batch-work-that-can-wait)                |
| 针对当前模型审查提示      | 两次测量的迁移上均为 14%                                                                                                                                                                            | 无；其中一次有提升                    | 更快（更少的工具轮次） | [针对当前模型审查提示](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#audit-prompts-against-the-current-model) |
| 升级模型            | Opus 4.8 到 Opus 5：每已解决任务多花 21% 换来多 12 个点（`low` 下的 Opus 5 以约 30% 的成本胜过 Opus 4.8）；Sonnet 4.6 到 Sonnet 5：每已解决任务少花 15%，多 5 个点；Fable 5 到 Fable 5.1：每已解决任务少花 43%，分数大致相同                         | 有提升                          | 中性          | [升级模型](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#upgrade-the-model)                             |
| 降低努力程度          | 知识工作：`medium` 13% 到 31%，`low` 三分之一到一半；长编码：`medium` 约一半，`low` 约四分之三                                                                                                                        | 知识工作上 1 到 3 个点，长编码上 2 到 8 个点 | 更快          | [调整努力程度](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#tune-effort)                                 |
| 重新运行失败任务        | 约一半，通过率相同                                                                                                                                                                                 | 无                            | 失败的任务上两次运行  | [以更高努力程度重新运行失败任务](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#re-run-failures-at-higher-effort)   |
| 任务预算            | 44% 到 58%                                                                                                                                                                                 | 3 到 6 个点                     | 更快          | [设置预算和输出上限](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#set-budgets-and-output-caps)              |
| 要求更短的回答         | 分诊运行上 39% 的输出令牌、14% 的成本                                                                                                                                                                   | 无                            | 更快          | [设置预算和输出上限](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#set-budgets-and-output-caps)              |
| 提高 `max_tokens` | 每已解决任务无节省，但解决的任务更多                                                                                                                                                                        | 内部集合上最多提升 22 个点；公开配对上无       | 中性          | [设置预算和输出上限](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#set-budgets-and-output-caps)              |
| 顾问              | 取决于能力差距和咨询率；编码配对比单独的 Opus 5 高 3.5 个点、比单独的 Fable 5.1 高约 2.5 个点，图表阅读配对以约 2.6 倍的价格追平了 `medium` 下单独的顾问模型                                                                                      | 小幅提升                         | 每任务约两次额外调用  | [顾问策略](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#advisor-strategy-escalate-hard-decisions)      |
| 编排器             | 相对前沿模型约一半，无论是超出一个上下文窗口还是在常规长尾上（后者在 Claude Fable 5 上测量）                                                                                                                                    | 比前沿模型低 10 到 12 个点            | 在大输入上快得多    | [编排器策略](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#orchestrator-strategy-delegate-bulk-work)     |

## 引用的基准测试

除非引用中另有说明，所有测量结果均为 Anthropic 内部运行这些基准测试所得。除非另有注明，成本均以各基准测试运行时有效的标价按美元计算；Claude Sonnet 5 的数据采用每百万输入令牌 2 美元、每百万输出令牌 10 美元的价格。标注为"notional USD"（名义美元）的图表按上述费率对每个请求的令牌数计价，而非报告实际账单。

1. **WideSearch：** Wong 等，"WideSearch: Benchmarking Agentic Broad Info-Seeking"，arXiv:2508.07999，2025。广泛的网络研究任务，按多行表格的完整性和准确性评分；200 道题，每种配置运行 3 次，运行于 2026 年 8 月 1 日至 2 日。成本集中度图表来自另一次 20 道题的运行，每题运行 3 次，运行于 2026 年 8 月 3 日至 4 日，成本依据逐请求计费记录计算。
2. **GDPval：** OpenAI，"GDPval: Evaluating AI Model Performance on Real-World Economically Valuable Tasks"，2025。知识工作交付物按任务评分标准评分；对已发布的黄金集进行 210 项任务的运行，每项任务尝试一次，运行于 2026 年 8 月 2 日。由 Claude 模型评分，因此绝对分数可能与已发表结果不同。
3. **SWE-bench Pro：** Scale AI，"SWE-Bench Pro: Can AI Agents Solve Long-Horizon Software Engineering Tasks?"，2025。为兼容 Anthropic 的评估框架而选取的 482 道题子集；分数与公开排行榜不可比。Claude Opus 5 在默认 effort 下取两次运行的平均值；降低 effort 的设置为单次运行；全部运行于 2026 年 8 月 4 日。升级数据逐任务取自这些运行：先用 `low`，再对其失败项使用默认设置，在各运行配对中解决率为 92.5% 至 93.6%，成本约 0.45 美元；先用 `medium`，解决率 93.8% 至 94.2%，成本约 0.61 美元；默认设置对其自身失败项重新运行，解决率 94.0%，成本 1.06 美元；全部使用默认设置，解决率 90.9% 至 92.5%，成本 0.93 美元。该子集上的成本按客户组织的计量方式计价：每个请求的先前提示按缓存读取计，其新令牌按 5 分钟缓存写入计，数据来自运行自身的用量记录，并与客户账本核对；评估组织自身的计量方式以 8,192 令牌为一页对缓存计费，得出的数字高出 1.4 至 1.8 倍。顾问图表上的 Claude Sonnet 5 执行器配对来自该子集上的同一测量系列：Sonnet 加 Opus 配对运行了两次（2026 年 8 月 7 日和 8 月 8 日，一次运行和一次精确复现），低 effort 配对运行一次（2026 年 8 月 8 日），Claude Sonnet 5 单独运行两次（77.4%，作为两个 Pro 行的基线）。"升级模型"中的 Claude Fable 5 数据点是默认 effort 下三次运行的平均值，运行于 2026 年 8 月 26 日，计价方式相同。Claude Fable 5.1 的任务预算数据为同一子集上默认 effort 下每个预算运行一次（35,000 令牌处运行两次），运行于 2026 年 8 月 26 日，并以同日一次无预算运行（92.1%，每任务 1.10 美元）作为基线；更早一组在 `low` effort 下运行于 2026 年 8 月 21 日，无预算得分 88.6%，每任务 0.48 美元。[比较模型](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#compare-models-on-cost-per-task)中的比较将该单次运行与同一子集上合并的两次 Claude Sonnet 5 运行配对；在 Fable 5.1 的默认 effort 下，这一对的结果方向相反，每个已解决任务的成本比 Sonnet 5 高 41%。升级阶梯为每个模型在其出厂默认设置下运行一次（Opus 5 和 Sonnet 5 各两次，Fable 5 数据点如上所述），Opus 和 Sonnet 的运行在同一周、同一框架和组织中完成。
4. **BrowseComp：** Wei 等，"BrowseComp: A Simple Yet Challenging Benchmark for Browsing Agents"，OpenAI，2025。effort 数据使用 500 道题的切分，每种设置运行一至三次，运行于 2026 年 8 月 3 日，默认数据点合并了 2026 年 7 月 26 日至 27 日的两次运行。成本保险图表使用 26 道题切片中 10 道可稳定解决的题目，50 次委派运行（2026 年 8 月 1 日至 2 日）和 70 次单独运行（50 次来自 2026 年 8 月 2 日至 3 日；20 次存档自 2026 年 7 月 12 日至 13 日及 8 月 1 日），期望值为每次运行 6.45 美元对比 11.99 美元；委派数据带有约 20% 的测量区间。
5. **智能体架构扩展：** Kim 等，"Towards a Science of Scaling Agent Systems"，arXiv:2512.08296，2025。独立的外部研究，仅引用其关于委派何时不划算这一发现的方向，不引用任何具体数字。
6. **DeepWideSearch：** "DeepWideSearch: Benchmarking Depth and Width in Agentic Information Seeking"，arXiv:2510.20168，2025。220 个问题涵盖 15 个领域，每个问题都将多行收集与多跳检索相结合；在该基准测试的固定行集上测量，每种配置运行 3 次，运行于 2026 年 8 月 2 日（单工作者团队数据点运行于 2026 年 7 月 26 日至 27 日）。
7. **DeepResearch Bench II：** Li 等，"DeepResearch Bench II: Diagnosing Deep Research Agents via Rubrics from Expert Report"，arXiv:2601.08536，2026。其涵盖 22 个领域的 132 项研究任务按专家推导的二元评分标准评分；在按所有主题分层的 50 项任务子集上测量，每项任务尝试一次，每种设置运行 3 次，在 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 上使用平台自带的网络搜索和抓取工具运行（2026 年 8 月 26 日至 27 日）；在没有任何配置拒绝的 33 项任务上评分，并剔除被生产安全分类器中途截断的尝试；成本为客户实际被计费的金额，即平台的请求费用加网络搜索费用。分数为每个模型在 33 项任务基础上剔除其自身被抢占任务后的平均值；在每个分组中均干净的 21 项任务上，Claude Fable 5.1 在每个 effort 级别上都比 Claude Fable 5 领先 2 至 3 分，且两个模型在各 effort 级别上表现持平。缓存图表将相同请求的每个输入令牌按未缓存费率重新计价。Claude Opus 4.6 按该基准测试的评分标准协议进行评判；原始基准使用不同的评判者，而 Anthropic 的评判者可能偏好自家风格。Claude Opus 5 在其默认 effort 下于 2026 年 8 月 28 日在相同平台和子集上运行了三次：原始 50 项任务得分 68.8%，33 项任务基础上得分 70.8%，21 项任务集上得分 71.1%，每任务 6.71 美元（无缓存时为 23.72 美元）；其所有尝试均未被安全分类器截断，所用的安全防护部署比其他模型运行时所用的更新。
8. **语料库缺陷扫描：** Anthropic 内部基准，针对超过一个上下文窗口的工作：一个来自 14 个公开 Python 包源码的 2160 万令牌语料库，植入 130 个缺陷并采用确定性评分；协议在运行前固定并经内部审查；每种配置运行三次。每种配置均在 Claude Managed Agents 上运行。图表中的团队配置是一次由 Claude Fable 5.1 协调器在平台内以其文档记载的 25 个并发 Claude Sonnet 5 工作者上限运行整个扫描的运行，运行于 2026 年 8 月 30 日；其三个回合在额外项审计后的 F1 得分为 0.764、0.825 和 0.791（原始值为 0.751、0.821 和 0.781），成本分别为 225 美元、234 美元和 283 美元。Claude Sonnet 5 单独配置运行于 2026 年 8 月 3 日至 4 日；Claude Fable 5.1 单独配置运行于 2026 年 8 月 24 日至 25 日，采用平台的发布时服务设置，每种 effort 设置三个种子，使用相同的语料库构建。沙箱镜像中带有部分语料库的已安装副本，Claude Fable 5.1 的最终汇总步骤在 9 个回合中有 7 个与之进行了比对；剔除这些新增项后重新评分，受影响种子的分数变动最多 3 分。绝对 F1 值特定于此语料库构建，不可跨基准测试比较；配置之间的比较是同类对比。
9. **GPQA Diamond：** Rein 等，"GPQA: A Graduate-Level Google-Proof Q\&A Benchmark"，2023。198 道题的 Diamond 子集，每种配置运行两次，运行于 2026 年 8 月 7 日，由模型对照参考答案评分，顾问令牌按请求计量。平台安全检查在 Sonnet 和 Opus 执行器上拒绝了两道生物学题目；排除它们后任何比较的变化均不超过一分。
10. **DeepSWE：** Datacurve，"DeepSWE: Measuring Frontier Coding Agents on Original, Long-Horizon Engineering Tasks"，arXiv:2607.07946，2026。该集合包含跨五种语言的 113 项原创任务，配有基于程序的验证器。配对各运行两次，运行于 2026 年 8 月 7 日，顾问令牌按请求计量，并使用客户端顾问循环而非顾问工具，计费方式相同。单模型 effort 扫描为单次运行，按令牌数计价，是一种考虑缓存的近似值。每任务成本为运行总额除以 113。
11. **内部智能体编码基准测试：** Anthropic 内部：370 项代码仓库任务，由各仓库自身的测试评分。API 数据在 128,000 令牌输出上限下测量，每种配置运行一次：Opus 5 单独在默认 effort 下运行于 2026 年 8 月 9 日至 10 日，Claude Fable 5.1 单独在五个显式设置的 effort 值下运行于 2026 年 8 月 20 日，配对运行于 2026 年 8 月 24 日至 25 日。每任务尝试次数：配对和 Opus 单独对照组为五次，单模型数据点为一次；配对平均每次尝试约进行两次顾问咨询；成本按每次尝试计。Claude Code 数据为 2026 年 7 月 8 日至 23 日对相同任务的运行，每种配置运行一次，成本为近似值。
12. **内部代码仓库任务基准测试（上限测量）：** 另一组 Anthropic 内部的约 130 项代码仓库任务，运行于 2026 年 8 月 8 日至 10 日（Claude Opus 5）和 2026 年 8 月 20 日（Claude Fable 5.1），使用普通 API 智能体循环，每项任务尝试一次。Claude Fable 5.1 的运行为每个上限 135 项任务，显式设置为默认 effort：16,384 令牌的数据为两次运行的平均值（两次均为 36.3%）；64,000 和 128,000 的数据为单次运行（58.5% 和 60.0%）。六道题在每次运行中都触发了安全拒绝，计为失败。Opus 5 的 16,384 令牌数据为两次运行的平均值，其 64,000 数据为单次运行（124 项任务计分）。SWE-bench Pro 上限数据为 Claude Fable 5.1 在默认 effort 下每个上限运行一次，运行于 2026 年 8 月 26 日，使用从引用 3 的 482 道题集合中分层抽取的 100 道题子集，与其分数不可比；两个上限在默认设置下得分相同。图表中的逐轮分布来自 Opus 在 64,000 下的运行和 Claude Fable 5.1 在 128,000 下的运行；没有任何 Opus 轮次达到其上限，有一个 Fable 5.1 轮次达到了 128,000（其 0.46% 的轮次超过 16,384）。
13. **Chartography：** Surge AI，"Chartography"，2026。完整发布的 100 道题集合，于 2026 年 8 月 8 日至 10 日使用 Anthropic 在 Claude Managed Agents 上的实现进行测量（标准云沙箱；顾问配置使用 Managed Agents 顾问）。由 Claude Sonnet 4.6 代替参考评判者评分，且基准测试带工具运行，因此分数可在此处各配置之间比较，但不可与已发布排行榜比较。每种配置运行两次并合并；运行间差异为 4 至 10 分。成本不含沙箱时间，后者增加不到 1%。Claude Fable 5.1 单独运行来自 2026 年 8 月 24 日，采用平台的发布时服务设置，每种设置运行两次；六次尝试触及 15 分钟会话上限并得 0 分，每次运行中有两张图表在安全拒绝后由 Claude Opus 5 作答。Claude Opus 5 低 effort 执行器搭配 Claude Fable 5.1 顾问于 2026 年 8 月 30 日在相同设置下运行两次（63.0 和 67.0，平均 65.0，每张图表 0.72 美元；每次运行中 88% 的任务咨询了顾问，其 219 条回复中有 4 条改由 Claude Opus 5 给出，每条均发生在生产安全过滤器阻止了顾问自身回复之后）。早期配对的咨询率比较来自 2026 年 8 月 10 日至 11 日在 Messages API 上使用容器工具集重新运行相同配置。
14. **支持台提示审计评估：** Anthropic 构建的 44 张支持工单集合，采用确定性评分，于 2026 年 8 月初运行并于 2026 年 8 月 8 日报告，使用六个系统提示，每个系统提示在同一干净提示上添加一种在为 Claude Opus 4.8 和 Claude Sonnet 4.6 编写的提示中常见的模式。每个图表数据点为三种情况之一（旧模型、新模型使用相同提示、新模型经审计后），在六个提示和 44 张工单上取平均。Opus 5 的准确率提升的 95% 置信区间为 3 至 8 分；Sonnet 的准确率差异在噪声范围内。
15. **数据文件问题集：** Anthropic 构建的 25 个聚合问题集合，针对一个公开酒类销售 CSV 的 1,862 行切片，基准真值由 pandas 计算并采用精确匹配评分，在 Claude Sonnet 5 和 Claude Opus 5 上运行，禁用思考（上下文内分组在默认设置下无法完成），4,000 令牌输出上限，不使用提示缓存，每种配置运行三次，运行于 2026 年 8 月 19 日。文件分组通过 Files API 上传 CSV 并使用 `code_execution_20260120` 工具。
16. **缓存时长测量：** 来自[精简输入和上下文令牌](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens)的 20 个 issue 分诊作业，于 2026 年 8 月 23 日在 Messages API 上使用相同框架在 Claude Sonnet 5 和 Claude Opus 5 上运行，Claude Opus 5 单元格的 `max_tokens` 提高至 4,096，并在随机选取的一部分轮次前插入暂停（在两个模型上对全部 20 个 issue 分别为无暂停、5%、10% 以及每轮 6 分钟，另在 Claude Sonnet 5 上每轮 2 分钟；在两个模型上对 5 个 issue 子集进行 20 分钟暂停；仅在 Claude Sonnet 5 上对 5 个 issue 子集进行 45 分钟暂停）。每个单元格运行三次，成本根据每个响应的 `usage` 字段在客户计费组织上按标价计算，准确率对照相同的黄金标签。两个模型上的交叉点均约为 3.3% 的轮次：即每个会话盈亏平衡比例的中位数，由成本模型根据该会话逐轮的上下文大小计算，覆盖分析中全部 45 个 Claude Sonnet 5 和 36 个 Claude Opus 5 的二十 issue 会话（每种暂停计划在完整作业上运行，三种缓存设置下各运行三次；5 个 issue 的单元格不在其中）。5% 单元格在 Claude Sonnet 5 上持平，因为该次抽样的暂停落在了较小的前缀上。本页的二十分之一规则高于测得的交叉点。Anthropic 仅将刷新 5 分钟缓存的保活请求作为对照进行了测量。它们最多与 1 小时设置持平，而在每轮前都有暂停时成本更高，因此不要在这两个模型上使用它们；在 Claude Fable 5.1 上算术结果相反（引用 19）。
17. **生产环境中的缓存读取占比：** 截至 2026 年 8 月 23 日的 14 天内汇总的第一方 Claude API 用量，仅限直接 API 产品，排除 Anthropic 内部组织，不识别任何组织。当某组织某日的请求携带工具定义和工具结果、其提示平均包含 9 次或更多先前工具调用、使用了缓存且至少发出 10 个此类请求时，该组织日计为智能体循环（API 没有对话标识符，因此以此代替对话长度）：106,487 个组织的 303,003 个组织日，缓存读取占全部输入令牌的中位数为 84.2%，上四分位数为 91.7%。用例标签（组织声明的用例，否则为其分类用例）覆盖这些组织日的 74% 及其令牌的 99%；编码组织提供 87% 的智能体输入令牌，读取中位数为 88.5%（25 次或更多先前工具调用时为 90.9%），上四分位数 93.4%，约 72% 的编码组织日达到 80% 或以上；支持、研究和数据智能体读取 84% 至 85%。组织日的最高十分位在编码上读取 95.9% 或以上，在支持、研究、数据和其他智能体上读取 94.2% 至 94.8%。25 次或更多先前工具调用时的请求级拆分来自一个六小时样本：编码 92% 读取、7% 写入、不到 1% 未缓存。未标注的组织大多规模较小，读取中位数为 11%。没有工具定义的组织日读取中位数为 34.6%。对同一时间窗口的一次独立查询重建了 10 个或更多请求的对话而非对组织日评分，得出中位数为 90.2%；差异在于范围而非数据。
18. **压缩时机测量：** 来自[精简输入和上下文令牌](https://platform.claude.com/docs/zh-CN/about-claude/models/optimizing-for-cost-and-intelligence#trim-input-and-context-tokens)的分诊智能体长变体，于 2026 年 8 月 24 日在 Claude Sonnet 5 上使用 5 分钟缓存运行，成本根据 usage 字段按标价计算，每个分组五个会话：一个全程默认 effort 的无变更分组（每会话 0.81 美元），以及两个从低 effort 开始并进行相同两项破坏缓存变更（切换到默认 effort 和新增一个工具）的分组，变更要么在会话中途的第 12 和第 17 个请求进行（0.95 美元），要么在首次压缩后的第一个请求上一并进行（0.75 美元）。第四个分组包含六个会话，运行于 2026 年 8 月 25 日，在触发首次压缩的请求上进行相同的两项变更（每会话 0.92 美元）：该请求的摘要过程将 81,000 令牌的上下文写入缓存而非读取，因此该过程成本为 0.21 美元，而边界分组中相同过程为 0.04 美元。会话在第 21 至 25 个请求时首次压缩（21 个会话中有 16 个在第 22 个请求），即提示超过 80,000 令牌压缩触发阈值之后，两个无变更会话在接近结束时进行了第二次压缩。边界分组总额低于无变更分组反映的是其变更前的低 effort 请求以及那些第二次压缩，而非缓存：两个分组的重写成本相差不到一美分。会话中途分组每会话支付了 0.23 美元的缓存重写费用；会话中途分组与边界分组之间的差异为 0.20 美元，95% 置信区间为 0.11 至 0.29 美元。一个会话中途会话在其模型于压缩后错误调用搜索工具并得到空结果后运行成本较低（0.82 美元）；它被计入在内，若不计入则该分组平均为 0.98 美元。8 月 24 日各分组的准确率平均为 20 个标签中的 14.2 个，8 月 25 日分组为 14.7 个；缓存读取占提示令牌的比例在无变更时为 91%，会话中途为 85%，边界处为 91%，在触发请求上变更时为 86%。
19. **Claude Fable 5.1 上的缓存时长测量：** 与引用 16 相同的 20 个 issue 分诊作业和框架，于 2026 年 8 月 23 日和 8 月 26 日在 Claude Fable 5.1 发布快照上按其发布价格运行（每百万令牌输入 10 美元、5 分钟写入 12.50 美元、1 小时写入 20 美元、缓存读取 0.25 美元、输出 50 美元），每种计划三种设置：5 分钟缓存、1 小时缓存，以及通过每空闲 4 分钟对未变更前缀发送一次 `max_tokens: 0` 请求来保温的 5 分钟缓存（8 月 23 日的运行使用 `max_tokens: 1` 进行 ping；8 月 26 日的每次 ping 都刷新了缓存且未计费任何输出）。计划：在全部 20 个 issue 上分别为无暂停、10% 的轮次以及每轮 6 分钟，并在 5 个 issue 子集上进行 45 分钟暂停；每个单元格运行三次，成本根据每个响应的 `usage` 字段按标价计算，准确率对照相同的黄金标签（20 个中 12 至 17 个精确标签）。8 月 26 日 5 分钟、1 小时和保活设置的每会话平均值：无暂停 2.42 美元、3.09 美元、2.29 美元；10% 暂停 4.50 美元、2.96 美元、2.36 美元；每轮暂停 22.89 美元、3.01 美元、2.62 美元；8 月 23 日的单元格在 6% 以内一致。45 分钟的数据（每个 5 issue 会话 1.68 美元、0.59 美元和 0.71 美元）来自 8 月 26 日的一次干净重新运行，此前一次缓存计费事故破坏了当天的首批单元格；8 月 23 日的运行给出 1.67 美元、0.58 美元和 0.70 美元。5 分钟与 1 小时设置之间的交叉点为 3.1% 的轮次，与引用 16 采用相同的度量。
20. **Terminal-Bench 3：** 该公开终端智能体基准测试的 74 项任务，在 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 上运行，使用两个自定义工具（一个 shell 和一个文件编辑器，由评估框架在每项任务自己的容器中运行）代替平台的内置工具，其余采用平台对外部账户的默认设置，每个模型在 `high` effort 下运行两次，2026 年 8 月 27 日至 28 日。分数为每个模型 148 次尝试的原始通过率；单次运行波动 5 至 11 分。成本为客户按标价将被计费的金额，根据运行的用量记录以 5 分钟缓存生命周期逐请求重新计价。Claude Opus 4.7 的 148 次尝试中有 11 次在其输出上限处结束。

## 后续步骤

<CardGroup cols={2}>
  <Card title="提示缓存" icon="database" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching">
    本页最大的免费收益：设置、生命周期和诊断。
  </Card>

  <Card title="Effort" icon="gauge" href="https://platform.claude.com/docs/zh-CN/build-with-claude/effort">
    在单个模型内以智能换取延迟和成本。
  </Card>

  <Card title="选择合适的模型" icon="settings" href="https://platform.claude.com/docs/zh-CN/about-claude/models/choosing-a-model">
    评估整个 Claude 模型家族的能力、速度和成本。
  </Card>

  <Card title="任务预算" icon="clock" href="https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets">
    为智能体循环提供一个可自我调节的令牌倒计时。
  </Card>

  <Card title="会话预算" icon="coins" href="https://platform.claude.com/docs/zh-CN/managed-agents/budgets">
    为 Managed Agents 会话设置硬性美元上限。
  </Card>

  <Card title="定价" icon="dollar-sign" href="https://platform.claude.com/docs/zh-CN/about-claude/pricing">
    查看每个 Claude 模型当前的每令牌定价。
  </Card>

  <Card title="Cookbook：Claude API 上的成本优化" icon="book" href="https://platform.claude.com/cookbook/cost-optimization-cost-optimization">
    在可运行的 notebook 中将这些手段逐一应用于一个可工作的智能体，并在每一步后查看每任务成本。
  </Card>

  <Card title="网络研讨会：在 Claude Platform 上构建" icon="play" href="https://www.anthropic.com/webinars/building-on-the-claude-platform-claude-fable-5-and-model-orchestration-patterns">
    观看顾问和编排器模式的演示讲解。
  </Card>
</CardGroup>
