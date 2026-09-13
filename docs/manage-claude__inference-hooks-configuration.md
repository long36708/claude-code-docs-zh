---
title: 配置 Inference hooks
url: https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration
description: 为您的 Claude Enterprise 组织允许 Inference hooks，连接您的 AI 安全服务器，并控制强制执行、故障处理和逐步发布。
---

<Note>
  Inference hooks（推理钩子）目前处于 beta 阶段，面向 Claude Enterprise 组织提供。配置它们需要 `organization:manage` 权限，内置的 Admin、Owner 和 Primary owner 角色拥有该权限，任何被授予该权限的自定义角色也同样拥有。
</Note>

Inference hooks 会将您组织中的提示发送到您选择的 AI 安全服务器，并在 Claude 处理每个请求之前将其挂起，等待允许或拒绝的裁决（verdict）。本页将逐步介绍如何开启该功能、连接您的服务器以及控制强制执行。要了解 Inference hooks 是什么以及何时使用它们，请参阅 [Inference hooks 概述](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks)。要构建 AI 安全服务器本身，请参阅[开发 Inference hooks 集成](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint)。

## 开始之前

您需要：

* claude.ai 中的 `organization:manage` 权限。内置的 **Admin**、**Owner** 和 **Primary owner** 角色拥有该权限，任何被授予该权限的自定义角色也同样拥有。
* 一个接受裁决请求的 AI 安全服务器 HTTPS 端点：位于可公开路由的主机上、使用 443 端口的 `https://` URL，且无需重定向即可访问。不支持反向隧道主机（ngrok 及类似的隧道服务）：Anthropic 的网络策略会阻止它们。请勿通过隧道进行测试；请将您的服务器托管在您控制的域名上。有关完整的[托管要求](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#receive-a-request)，以及如何构建服务器和验证签名请求，请参阅[开发 Inference hooks 集成](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint)。

## 设置 Inference hooks

共有三种强制执行状态：**关闭**（**Enforce verdicts** 处于关闭状态：永远不会联系您的 AI 安全服务器，也不会检查提示）、**影子**（**Enforce verdicts** 处于开启状态且 **Mode** 设置为 **Shadow mode**：您的 AI 安全服务器会接收提示并返回裁决，但不会阻止任何内容）以及**强制执行**（**Enforce verdicts** 处于开启状态且 **Mode** 设置为 **Allow the request** 或 **Block the request**：拒绝裁决会阻止请求）。以下步骤将一个新配置从关闭状态带到强制执行状态。

<Steps>
  <Step title="为您的组织允许 Inference hooks">
    前往 claude.ai > **Organization settings** > **Data and privacy**，找到 **Inference hooks** 部分。开启 **Allow for your organization**。

    开启此选项会解锁 Inference hooks 设置页面，并始终强制将 **Enforce verdicts** 置于关闭状态，因此允许该功能本身永远不会启动检查：即使是之前已开启强制执行的配置，在您于最后一步重新开启 **Enforce verdicts** 之前也会保持不检查状态。
  </Step>

  <Step title="打开 Inference hooks 设置页面">
    仍在 **Data and privacy** 中，打开 **Inference hooks** 部分以进入 Inference hooks 设置页面。它位于 Data and privacy 之下，而不是作为设置导航中的独立条目，因此其面包屑导航显示为 **Data and privacy / Inference hooks**。在您保存端点之前，该页面会警告提示尚未被检查，并且 **Enforce verdicts** 保持关闭状态并带有 **Requires endpoint** 徽章。
  </Step>

  <Step title="配置您的端点">
    点击 **Configure** 打开 **Configure endpoint** 对话框并填写：

    * **Endpoint URL：** 接收裁决请求的 `https://` URL。仅接受 `https://` URL。
    * **Custom request headers：** 最多 16 个静态标头，随每个裁决请求一起发送，以便您的 AI 安全服务器能够对调用方进行身份验证。标头值以加密方式存储，且永远不会再次显示；保存后仅显示标头名称。由于值是只写的，保存对标头的任何更改都需要重新输入每个值。更改端点 URL 会清除所有已存储的标头值，以确保您的凭据永远不会被发送到新的目的地；更改 URL 后请重新输入它们。标头名称必须使用标准 HTTP 令牌字符，使用 `-` 而非 `_`，并且不得与保留名称冲突（请求帧标头如 `Content-*` 和 `Host`、代理和 cookie 标头、客户端地址标头如 `X-Forwarded-*`、`webhook-*` 签名标头以及 `X-Anthropic-*` 前缀）。值必须是可打印的 ASCII 字符。

    该对话框仅涵盖这两个字段以及 **Test connection**；它不会询问故障处理，您将在第 6 步中选择故障处理。保存端点后，该按钮显示为 **Edit**。
  </Step>

  <Step title="测试连接">
    点击 **Test connection**。Claude 会向表单中当前的 URL 和标头（而非已保存的值）发送一个合成测试提示，因此在测试之前请重新输入任何已存储的标头值。成功时，结果会报告您的 AI 安全服务器对测试提示返回的是允许还是拒绝裁决，这可以在您开始强制执行之前暴露出"拒绝一切"的默认设置。

    常见的失败结果：

    | 结果       | 检查内容                                                       |
    | -------- | ---------------------------------------------------------- |
    | URL 被拒绝  | URL 未通过结构检查。请使用 443 端口上的 `https://` URL。                   |
    | 私有或内部 IP | 主机解析为私有或内部地址。请使用可公开路由的主机。                                  |
    | 超时       | AI 安全服务器未在超时时间内返回裁决。                                       |
    | 传输错误     | DNS 解析、TLS 握手或连接失败。                                        |
    | 非 200 状态 | AI 安全服务器以 200 以外的状态进行了响应。裁决必须以 HTTP 200 返回；重定向不会被跟随，并计为失败。 |
    | 无法解析的响应  | AI 安全服务器进行了响应，但响应体不是有效的裁决。                                 |
  </Step>

  <Step title="保存并存储您的签名密钥">
    保存端点配置。首次保存会生成您的 webhook 签名密钥并仅显示一次。请在关闭对话框之前复制并安全存储它：该密钥之后无法再检索，只能[轮换](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration#rotate-your-signing-secret)。

    您的 AI 安全服务器使用此密钥来验证其收到的每个请求上的签名。有关验证流程，请参阅[验证签名](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#verify-the-signature)。
  </Step>

  <Step title="选择故障处理和超时">
    在 **Failure handling** 下，设置 **Mode** 以选择当 AI 安全服务器无法访问或裁决超时时会发生什么：

    * **Block the request：** 当您的 AI 安全服务器无法提供裁决时停止推理（故障关闭）。
    * **Allow the request：** 让请求在未经检查的情况下继续发送到模型（故障开放）。

    下拉菜单的第三个选项 **Shadow mode** 是一种逐步发布工具，而非故障策略；请参阅[影子模式](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration#shadow-mode)。

    然后设置 **Prompt verdict timeout (ms)**：1 到 10,000 毫秒，默认为 5,000 毫秒。该预算涵盖整个交换过程，较慢的裁决会被视为服务器无法访问，因此请设置您的服务器能够可靠满足的最低值。

    此部分中的更改会在您进行更改时即时保存。首次保存时，默认值为 **Allow the request** 和 5,000 毫秒。
  </Step>

  <Step title="选择逐步发布百分比">
    在 **Rollout** 下，设置 **Requests inspected (%)**，以便在您启动 AI 安全服务器期间对一定百分比的请求运行检查。该值范围为 0 到 100：100 表示检查所有请求，0 表示关闭检查。

    每个请求针对其整个对话轮次只抽样一次，因此单个对话可能在不同轮次之间被部分检查。抽样百分比之外的请求会在未经检查的情况下继续进行，即使故障处理设置为 **Block the request** 也是如此。
  </Step>

  <Step title="开启 Enforce verdicts">
    若要先针对实时流量评估裁决而不阻止任何人，请在开启强制执行之前将 **Mode** 设置为 **Shadow mode**（第 6 步）；请参阅[影子模式](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-configuration#shadow-mode)。

    开启 **Enforce verdicts**，使 Claude 对每个受管控的提示都以您的 AI 安全服务器的裁决为准，然后在对话框中确认，该对话框会重述您的故障处理选择。请留出大约一分钟时间让更改到达每台 Anthropic 服务器；已在处理中的请求将按旧设置完成。关闭它会停止向您的 AI 安全服务器发送提示，同样在大约一分钟内生效；您的配置会被保留。
  </Step>
</Steps>

## 影子模式

Shadow mode（影子模式）针对实时流量运行您的钩子而不阻止任何内容。您的 AI 安全服务器会接收受管控的提示并返回裁决，与强制执行时完全相同，但不会阻止任何内容：每个请求都会继续发送到模型，即使您的服务器拒绝它或无法访问也是如此，并且最终用户不会看到任何内容。在开始强制执行之前，使用它来针对您组织的真实流量调整您的策略。

要使用影子模式，请在 **Failure handling** 下将 **Mode** 设置为 **Shadow mode**，然后开启 **Enforce verdicts**，以便提示流向您的 AI 安全服务器。当它处于活动状态时，设置页面会显示 **Shadow mode — not blocking** 徽章。要退出影子模式，请将 **Mode** 设置回 **Allow the request** 或 **Block the request**；一旦强制执行开启，裁决将再次被强制执行。

## 排除项

在 **Exclusions** 下，选择其成员不受 Inference hooks 覆盖的角色：他们的提示永远不会被发送到您的 AI 安全服务器。只有您的组织创建的自定义角色可以被排除；不提供内置角色。在角色选择器中选择它们（其占位符显示为 **Select roles to exclude**），并从角色管理页面（**Manage roles**）管理谁拥有每个角色；更改排除项需要身份管理权限。该列表默认为空，在没有排除任何角色的情况下，每个受管控的请求都会被检查。

排除适用于用户的交互式会话；通过机器凭据进行身份验证的流量始终会被检查。如果 Claude 无法解析请求者的角色成员身份，该请求会以可重试的错误故障关闭，而不是在未经检查的情况下继续进行。对排除列表的更改会记录在审计跟踪中。

## 自定义被阻止提示消息

在 **Custom blocked prompt message** 下，设置最多 500 个字符的自定义文本，当您的 AI 安全服务器拒绝请求时，该文本会附加到最终用户看到的错误之后（通常是联系谁或在哪里申请例外）。最终消息由您的 AI 安全服务器针对每个请求的 `deny_reason`（如果存在）、一个空行，然后是此文本组成。在未配置自定义文本的情况下，内置默认消息会引导用户联系其管理员；您也可以完全关闭附加消息，使用户只看到 `deny_reason`。

## 监控您的 AI 安全服务器

Inference hooks 设置页面的端点健康区域显示：

* **Endpoint status：** Healthy、Tripped、Not enforcing，或在保存端点之前显示 Not configured。
* **Failures per minute：** 过去两分钟内 webhook 失败次数的平均值。
* **Block rate：** 拒绝占您的 AI 安全服务器裁决的比例，在逐步发布百分比低于 100 时显示。
* **Circuit breaker tripped：** 断路器上次跳闸的时间（如果曾跳闸）。
* **Recent errors：** 每个条目被精简为时间戳、错误类型和一行原因。条目永远不会包含请求内容或您的端点 URL。

该面板是尽力而为的：如果 Anthropic 无法读取计数器，它会显示零失败和无错误，而不是显示其自身的错误，因此看起来健康的面板本身并不能证明您的 AI 安全服务器是健康的。**Failures per minute** 统计每一次失败，包括永远不会触发断路器的网络和 DNS 错误，因此它可能很高而 **Circuit breaker tripped** 仍为空。

## 断路器

可归因于您的 AI 安全服务器的持续 webhook 失败会触发 circuit breaker（断路器）跳闸，从而停止强制执行：不再联系您的服务器，并且您的 **Failure handling** 选择将应用于每个被检查的请求。如果选择了 **Block the request**，您组织中的用户将被阻止，直到断路器重置。当断路器跳闸时，管理员也会在 claude.ai 通知中心收到通知。

每次跳闸还会作为 `inference_hooks_circuit_breaker_tripped` 活动记录在您组织的[活动源](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)中，因此您的安全团队或供应商可以通过他们已在运行的监控（例如摄取该活动源的 SIEM）对跳闸发出警报。每次跳闸记录一个活动，而不是每个受影响的请求记录一个。记录需要为您的组织启用 Compliance API；请参阅[设置 Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。

要恢复，请修复服务器，然后重新开启 **Enforce verdicts** 以重置断路器。

断路器也可以自行重置。从跳闸后 10 分钟开始，Anthropic 会测试您的服务器是否已恢复：最多大约每分钟一次，从您组织的正常流量中取一个请求发送到您的服务器进行检查，并且无论您的服务器是否响应，该请求都会为其用户继续进行。如果您的服务器以有效的裁决（允许或拒绝）进行响应，断路器将重置并恢复强制执行。任何其他结果都是 webhook 失败：断路器保持跳闸状态并继续测试。

自动恢复仅在您的 Inference hooks 设置自跳闸以来未更改的情况下运行。如果您在跳闸后更改任何 Inference hooks 设置（包括轮换签名密钥），测试将停止，断路器不再自行重置；请在服务器修复后重新开启 **Enforce verdicts**。自动恢复仅适用于跳闸：如果您自己关闭了 **Enforce verdicts**，强制执行将保持关闭状态，直到您重新开启它。

## 轮换您的签名密钥

点击 **Request signing** 下的 **Rotate secret** 以替换您的签名密钥。轮换是立即切换：新密钥会生成并仅显示一次，旧密钥无法再检索，并且任何请求都不会同时使用两个密钥签名，因此没有可依赖的重叠期。

使用先前密钥签名的请求在轮换后仍可能短暂到达；[验证签名](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint#verify-the-signature)介绍了您的 AI 安全服务器应如何处理切换。

## 审计跟踪

Inference hooks 活动会记录在您组织的[活动源](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)中：配置更改、拒绝、断路器跳闸，以及根据您的故障处理设置在未经检查的情况下继续进行的请求。当断路器处于跳闸状态时，不会记录每个请求的 Inference hooks 活动；跳闸活动是活动源对该时间窗口的记录。拒绝记录带有标识符，可让您将每次拒绝与您自己系统中的匹配记录关联起来。

## 关闭 Inference hooks

关闭有两个级别：

* 在 Inference hooks 设置页面上关闭 **Enforce verdicts**：在大约一分钟内，您组织的提示将停止发送到您的 AI 安全服务器；已在处理中的请求将按旧设置完成。设置页面仍然可用，因此在您处理 AI 安全服务器时可使用此方式暂停强制执行。
* 在 **Data and privacy** 设置中关闭 **Allow for your organization**：提示不再被检查，并且 Inference hooks 设置将变为不可用，直到您重新开启它。无论哪种方式，您的端点配置、自定义标头和签名密钥都会被保留；重新开启它会强制将 **Enforce verdicts** 置于关闭状态并清除已跳闸的断路器，因此请在准备就绪时再次开启强制执行。

## 后续步骤

<CardGroup cols={2}>
  <Card title="开发 Inference hooks 集成" href="https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks-endpoint">
    构建 AI 安全服务器：请求和裁决模式、签名验证以及操作语义。
  </Card>

  <Card title="Inference hooks 概述" href="https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks">
    Inference hooks 是什么、裁决往返如何工作，以及哪些内容会被发送到您的 AI 安全服务器。
  </Card>
</CardGroup>
