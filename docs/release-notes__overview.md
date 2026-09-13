---
title: Claude Platform 发布说明
url: https://platform.claude.com/docs/zh-CN/release-notes/overview
description: Claude Platform 的更新，包括 Claude API、客户端 SDK 和 Claude Console。
---

Claude Platform 发布说明列出了 Claude API、客户端 SDK 和 Claude Console 的变更，按时间由新到旧排列。

<Tip>
  有关 Claude Apps 的发布说明，请参阅 [Claude 帮助中心中的 Claude Apps 发布说明](https://support.claude.com/en/articles/12138966-release-notes)。

  有关 Claude Code 的更新，请参阅 `claude-code` 仓库中的[完整 CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)。
</Tip>

### 2026 年 9 月 1 日

* 我们发布了 **Claude Fable 5.1**（`claude-fable-5-1`），它是 Claude Fable 5 的继任者，适用于长时间运行的智能体编码、知识工作和研究；同时面向 Project Glasswing 参与者发布了 **Claude Mythos 5.1**（`claude-mythos-5-1`）。两个模型默认支持 [1M 令牌的 "context window"（上下文窗口）](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)、128k 最大输出令牌，以及始终开启的 ["adaptive thinking"（自适应思考）](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)，价格为每 MTok $10 / $50 美元，与 Claude Fable 5 相同，缓存读取价格降至每 MTok $0.25。Claude Fable 5.1 可在 Claude API、[Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)、[Claude on Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 和 [Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上使用。有关功能、API 变更和迁移指南，请参阅 [Claude Fable 5.1 新特性](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1)。
* Claude Fable 5.1 和 Claude Mythos 5.1 上的 "prompt caching"（提示缓存）读取费用为每百万令牌 $0.25 美元：是基础输入价格的 0.025 倍，而其他模型为 0.1 倍。缓存写入价格不变。请参阅[提示缓存定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#prompt-caching)。
* 在 Claude Fable 5.1 和 Claude Mythos 5.1 上，`tool_choice` 类型 `any` 和 `tool` 不受支持，会返回 400 错误。`auto` 和 `none` 保持不变。若要保证工具输入符合 schema，请使用[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)或[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)。
* Claude Fable 5.1 和 Claude Mythos 5.1 生成的思考块仅为生成它们的模型或更新的模型保留：更早的模型无法读取它们，并且当思考块被回放给更早的模型时，API 会将其丢弃。Claude Fable 5.1 接受来自 Claude Opus 5、Claude Fable 5、Claude Mythos 5 以及更早 Claude 模型的思考块。在 Claude Fable 5.1 上，API 还会[检查某个块之前的内容是否发生了变化](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)：对于 2026 年 8 月 31 日或之后创建的新账户，在 `system` 提示、`tools` 或更早的消息发生变化后回放思考块会返回 400 错误。使用 `thinking-binding-controls-2026-08-01` beta 请求头时，被丢弃的块会在 `input_transformations` 响应字段中报告，而 `thinking.block_binding.prefix_mismatch_behavior` 可在拒绝和丢弃历史已变更的块之间进行选择。请参阅[保留的思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-thinking)。
* 按消息设置 effort（努力程度）的变更功能已在 Claude API 上的 Claude Fable 5.1、Claude Mythos 5.1 和 Claude Opus 5 上进入 beta 阶段。在 `messages` 中添加一条带有 `output_config.effort` 的 `role: "system"` 消息，即可在保留提示缓存的同时更改后续轮次的 effort。请在请求中包含 `mid-conversation-output-config-2026-07-01` beta 请求头。请参阅[按消息设置 effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#change-effort-mid-conversation-beta)。
* [轮次作用域的系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#turn-scoped-system-messages)已进入 beta 阶段（`mid-conversation-system-clear-at-2026-08-21` 请求头）。在对话中途的 `role: "system"` 消息上设置 `clear_at: "next_user_message"`，该消息将仅在当前轮次渲染，之后保留在历史记录中且不产生令牌费用。每轮提醒不会累积，也不会使提示缓存或后续思考块失效。
* `thinking.display` 在 beta 阶段接受第三个值 `"updates"`（`thinking-display-updates-2026-08-18` 请求头）。推理内容以空的 `thinking` 字段返回，与 `"omitted"` 下相同；而 Claude Fable 5.1、Claude Mythos 5.1 和 Claude Fable 5 在工具调用之间写入的简短进度更新会以文本形式返回，每次工具调用之前最多一个 `thinking` 块。请参阅[工具调用之间的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#progress-updates)。
* Claude Fable 5.1 和 Claude Mythos 5.1 生成的文本带有 Anthropic 的文本水印；当您通过 Claude API 上的 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 检索 Claude 通过[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)生成的受支持图像和视频文件时，这些文件带有 C2PA 内容凭证。标记无需对您的请求或响应处理做任何更改。
* 与 Claude Fable 5 一样，两个模型都要求 30 天数据保留，除非获得 Anthropic 明确授权，否则不可在零数据保留下使用。请参阅[特定模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。

### 2026 年 8 月 27 日

* 在 Python SDK 1.2.0、TypeScript SDK 0.122.0、Go SDK 1.68.0、Java SDK 2.59.0、Ruby SDK 1.67.0 和 C# SDK 12.44.0 中，`client.beta.files` 和 `client.beta.skills` 不再发送 `files-api-2025-04-14` 和 `skills-2025-10-02` beta 请求头，并返回与 `client.files` 和 `client.skills` 相同的结构。随着这一变更，`client.beta.skills.delete()` 会删除一个 Skill 及其所有版本，并且 beta Messages 类型 `BetaSkill`（容器 Skill 引用）已重命名为 `BetaContainerSkill`。仍然发送 beta 请求头的请求将继续收到 beta 结构。请参阅[从 `files-api-2025-04-14` 迁移](https://platform.claude.com/docs/zh-CN/build-with-claude/files#migrate-from-files-api-2025-04-14)和[从 `skills-2025-10-02` 迁移](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide#migrate-from-skills-2025-10-02)。

- 您现在可以在 Claude Console 中创建**个人密钥**和**服务账户密钥**。它们以您本人或某个[服务账户](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#service-accounts)的身份运行，拥有相同的权限，并在关联账户从组织中移除时停止工作。这让组织管理员能够更轻松地跟踪每个账户的使用情况，并确保密钥使用合法。这些 "API key"（API 密钥）可以限定到特定工作区，也可以[在管理端点上以及该账户有权访问的任何工作区中使用](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)。工作区 API 密钥作为旧版选项仍受支持。更多信息请参阅 [API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#api-keys)。

### 2026 年 8 月 26 日

* [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 的会话端点已针对 Cowork 和 Claude Code 会话结束 beta 阶段。请参阅[检索会话记录](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions)。
* [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 的本地会话端点现在还会返回 Claude Science 会话（`product_surface` 值为 `claude_science`）以及 Excel、PowerPoint、Word 和 Outlook 中的 Claude for Microsoft 365 会话（`product_surface` 值以 `office_agents` 开头）的记录，面向 Claude Enterprise 组织提供 beta 版，使用您现有的 Compliance Access Key 和 `read:compliance_user_data` 作用域。请参阅[用户机器上的会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)。

- [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 现已在 `ant` CLI 以及 Python、TypeScript、C#、Go、Java、PHP 和 Ruby SDK 中通过 `client.beta.organization` 提供。它们涵盖组织信息、成员、邀请、工作区和工作区成员、API 密钥、速率限制、服务账户、工作负载身份联合颁发者和规则，以及客户管理的加密密钥。使用量和成本报告以及 Claude Enterprise 用户管理和分析端点仍仅支持 curl。CLI 和 SDK 从 `ANTHROPIC_API_KEY` 读取 Admin API 密钥，或从 `ANTHROPIC_AUTH_TOKEN` 读取 `org:admin` OAuth 令牌。

### 2026 年 8 月 20 日

* 我们发布了 **[Python SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/python) v1.0**。SDK 的 HTTP 层从 `httpx` 迁移到 [httpx2](https://httpx2.pydantic.dev)，这是一个持续维护、API 兼容的分支：请使用 `httpx2` 构建自定义的 `http_client`、`Timeout` 和传输对象（`DefaultHttpxClient` 辅助工具保持不变），如果您依赖于对 `httpx` 打补丁的追踪或模拟库，请在启动时调用 `httpx2.alias_httpx()`。v1.0 要求 Python 3.10 或更高版本，并移除了长期弃用的接口，包括旧版 Text Completions API、Messages 方法上的 `temperature`、`top_p` 和 `top_k` 参数，以及工具运行器的客户端 `compaction_control`。在异步客户端上，`.with_raw_response` 结果现在需要 `await response.parse()`，并且 `AnthropicBedrock` 在未配置 AWS 区域时现在会抛出错误，而不是默认使用 `us-east-1`。有关每项变更及前后对比代码片段，请参阅 [v1 迁移指南](https://github.com/anthropics/anthropic-sdk-python/blob/main/MIGRATION.md)。
* [计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)和[浏览器使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)工具集（`computer_toolset_20260801` 和 `browser_toolset_20260801`）现已在 [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 上面向 Claude Fable 5、Claude Mythos 5、Claude Opus 5、Claude Sonnet 5 和 Claude Opus 4.8 提供。请求使用与 Claude API 上相同的 `tools` 条目。

### 2026 年 8 月 19 日

* [计算机使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)已在 Claude API 上以 `computer_toolset_20260801` 工具集的形式结束 beta 阶段：无需 beta 请求头，支持批量操作（一轮中执行多个操作），默认启用 `zoom`，并可通过 `configs` 进行按成员配置。早期 beta 版本仍然可用。升级现有集成会改变请求结构和工具处理方式；请参阅[从 `computer_20251124` 迁移](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool#migrate-from-computer-20251124)。
* 我们发布了[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)（`browser_toolset_20260801`），这是一个用于驱动由您的应用程序托管的浏览器的客户端工具集。它在浏览器视口内工作，而非整个桌面，读取页面本身（其无障碍树、元素、表单和标签页），并在截图点击控制的基础上增加了元素引用、表单输入、标签页管理、下载报告和可选的文件上传功能。
* 两个工具集均在 Claude API 上面向 Claude Fable 5、Claude Mythos 5、Claude Opus 5、Claude Sonnet 5 和 Claude Opus 4.8 提供。
* [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 已在 Claude API 上结束 beta 阶段。对 `/v1/files` 端点的请求，以及引用已上传文件的 Messages API 请求，不再需要 `files-api-2025-04-14` beta 请求头。不带该请求头发送的请求使用当前的响应格式：[文件过期](https://platform.claude.com/docs/zh-CN/build-with-claude/files#file-expiration)（上传文件时设置 `expires_in_seconds`；文件对象报告 `expires_at`），以及在[列出文件](https://platform.claude.com/docs/zh-CN/build-with-claude/files#list-files)时使用 `page` 和 `next_page` [分页](https://platform.claude.com/docs/zh-CN/api/overview#pagination)加上 `ids[]` 过滤器。仍然发送 beta 请求头的 `/v1/files` 请求将继续工作并返回之前的响应格式。 要将现有集成从该请求头迁移出来，请参阅[从 `files-api-2025-04-14` 迁移](https://platform.claude.com/docs/zh-CN/build-with-claude/files#migrate-from-files-api-2025-04-14)。
* [Agent Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview) 和 Skills API（`/v1/skills`）已在 Claude API 上结束 beta 阶段。请求不再需要 `skills-2025-10-02` beta 请求头，包括通过 `container` 参数加载 Skills 的 Messages API 请求。仍然发送该请求头的请求将继续正常工作，不受影响。请参阅[通过 API 使用 Agent Skills](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide)。 要将现有集成从该请求头迁移出来，请参阅[从 `skills-2025-10-02` 迁移](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide#migrate-from-skills-2025-10-02)。
* 面向 **Claude Enterprise**（claude.ai）组织的 [Admin API](https://platform.claude.com/docs/zh-CN/api/admin) 用户管理端点（成员、邀请、群组和自定义角色）已结束 beta 阶段。群组和自定义角色请求不再需要 `anthropic-beta: ce-user-management-2026-07-13` 请求头；仍然发送该请求头的请求将被原样接受。请参阅[用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management)。
* 您现在可以限制 Claude Managed Agents 智能体的 `web_search` 和 `web_fetch` 工具可以访问哪些站点。在 `agent_toolset_20260401` 的 `configs` 数组中该工具的条目上设置 `allowed_domains` 或 `blocked_domains`；`web_fetch` 还接受 `max_content_tokens`，`web_search` 接受 `user_location`。每个 `configs` 条目通过其 `name` 标识，并通过可选的 `type` 指定类型，仅传递 `name`、`enabled` 和 `permission_policy` 的请求将继续工作；在类型化 SDK 中，`configs` 条目变为按工具区分的类型。请参阅[限制网页搜索和网页抓取域名](https://platform.claude.com/docs/zh-CN/managed-agents/tools#restrict-web-search-and-web-fetch-domains)。
* 在[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)中运行的 Claude Managed Agents 会话现在可以挂载[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory)。Python、TypeScript 和 Go SDK 工作进程会将每个挂载的存储下载到沙箱中的 `mount_path`，并将智能体的更改同步回存储。请参阅[使用记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)。
* Claude Console 中的会话查看器已重新设计，新增了时间线缩略图、按模型请求分组的记录，以及用于查看会话详情和成本、原始事件、按工具统计、已挂载资源和按线程活动的 Inspector 面板。请参阅 [Console 可观测性](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#console-observability)。

### 2026 年 8 月 18 日

* Workbench 现已成为 Claude Console 中的 [**playground**](https://platform.claude.com/playground)。Playground 支持所有 Messages API 参数，并包含演示代码执行和网页搜索等 API 功能的模板。它会显示每次运行的完整 SDK 请求和 API 响应，帮助您理解 API 并使用它进行构建。更多信息请参阅 [Claude 帮助中心](https://support.claude.com/en/articles/8606378-how-do-i-use-playground)，或在 [platform.claude.com/playground](https://platform.claude.com/playground) 上试用。

### 2026 年 8 月 11 日

* [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 现在会返回在您用户机器上运行的 Cowork 和 Claude Code 会话的记录，面向 Claude Enterprise 组织提供 beta 版。`GET /v1/compliance/apps/sessions/local` 列出您组织中的会话，`GET /v1/compliance/apps/sessions/local/{session_id}` 检索单个会话的元数据，`GET /v1/compliance/apps/sessions/local/{session_id}/messages` 返回其记录，全部使用您现有的 Compliance Access Key 和 `read:compliance_user_data` 作用域。请参阅[用户机器上的会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-local-sessions)。
* 我们为 Claude API 添加了 `anthropic-workspace-id` 响应头。它携带请求的 API 密钥或访问令牌所解析到的工作区的 `wrkspc_` 前缀 ID，包括您组织的默认工作区。请参阅[识别 API 响应背后的工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#identify-the-workspace-behind-an-api-response)。

### 2026 年 8 月 10 日

* **Claude Sonnet 5** 的推广定价（每 MTok $2 / $10）现已成为标准价格：原定于 2026 年 9 月 1 日上调至每 MTok $3 / $15 的计划将不会执行。请参阅[定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)。

### 2026 年 8 月 7 日

* 您现在可以为 Claude Managed Agents 会话设置预算：对会话支出的硬性上限，按公开标价计费。达到预算的会话会以 `budget_reached` 停止原因暂停，而不是启动新的模型请求；更改或移除预算会使其恢复。部署接受相同的预算，并将其应用于它们启动的每个会话。请参阅[会话预算](https://platform.claude.com/docs/zh-CN/managed-agents/budgets)。
* 您现在可以为 Claude Managed Agents 会话配置一个顾问：一个能力至少与智能体自身模型相当的模型，会话的主线程可以在轮次中途向其咨询战略指导。在智能体的多智能体名册中将其配置为 `{"type": "advisor"}` 条目，并指定要咨询的 `model`。请参阅[为会话配置顾问](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#give-the-session-an-advisor)。
* 您现在可以控制 Claude Managed Agents 智能体的模型推理在何处运行。在[创建智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#pin-the-inference-geo)时在 `model` 对象内设置 `inference_geo`，或[为单个会话覆盖该设置](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#pin-the-inference-geo-for-a-session)。有关可用地理区域和定价，请参阅[数据驻留](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)。
* Claude Managed Agents 会话现在可以[从 GitHub 仓库加载技能](https://platform.claude.com/docs/zh-CN/managed-agents/skills#load-skills-from-a-github-repository)。当会话[挂载仓库](https://platform.claude.com/docs/zh-CN/managed-agents/github)时，其根目录 `.claude/skills` 中的任何技能都会在会话启动时自动被发现，并在该会话中供智能体使用。

### 2026 年 8 月 5 日

* **推理钩子**现已面向 Claude Enterprise 组织提供 beta 版。将 Claude 指向您组织的 AI 安全服务器，claude.ai、Cowork 和 Claude Code 中每个受管控的提示都会在推理继续之前等待服务器的允许或拒绝裁决。请求经过签名，故障处理可配置，每次拒绝都会记录在合规[活动源](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)中。请参阅[推理钩子](https://platform.claude.com/docs/zh-CN/manage-claude/inference-hooks)。
* 我们已停用 Claude Opus 4.1 模型（`claude-opus-4-1-20250805`）。Claude API 上对该模型的所有请求现在都会返回错误。我们建议升级到 [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)。研究人员可以通过[外部研究人员访问计划](https://support.claude.com/en/articles/9125743-what-is-the-external-researcher-access-program)申请持续访问权限。

### 2026 年 8 月 3 日

* [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 现在会返回在 claude.ai 网页端或移动端启动的 Cowork 会话的记录，面向 Claude Enterprise 组织提供 beta 版。`GET /v1/compliance/apps/sessions/remote` 列出会话，`GET /v1/compliance/apps/sessions/remote/{session_id}/messages` 返回单个会话的记录，使用您现有的带有 `read:compliance_user_data` 作用域的 Compliance Access Key。请参阅[云端会话](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-sessions#retrieve-remote-sessions)。

### 2026 年 8 月 1 日

* [Dreams](https://platform.claude.com/docs/zh-CN/managed-agents/dreams)（研究预览版）现已支持 Claude Opus 5。请参阅[支持的模型](https://platform.claude.com/docs/zh-CN/managed-agents/dreams#limits)。

### 2026 年 7 月 24 日

* 我们发布了 **Claude Opus 5**（`claude-opus-5`），相比 Claude Opus 4.8 有跨越式的提升。Claude Opus 5 支持 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)（既是默认值也是最大值）、128k 最大输出令牌，以及默认开启的[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)，价格为每 MTok $5 / $25 美元，与 Claude Opus 4.8 定价相同。它可在 Claude API、[Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)、[Claude on Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai) 和 [Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上使用。有关新功能、行为变更和迁移指南，请参阅 [Claude Opus 5 新特性](https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5)；有关完整规格，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。
* 在 Claude Opus 5 上，仅允许在 effort 为 `high` 或更低时禁用思考：`thinking: {"type": "disabled"}` 搭配 effort `xhigh` 或 `max` 会返回 400 错误，这是相对于 Claude Opus 4.8 的破坏性变更。请参阅 [Claude Opus 5 新特性](https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5#behavior-changes)。
* [Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 是引导 Claude Opus 5 的主要控制手段：该模型支持完整的等级阶梯（`low`、`medium`、`high`、`xhigh`、`max`），其中 `max` 用于对能力要求极高的工作。
* 对话中途工具变更现已在 Claude Fable 5、Claude Mythos 5、Claude Opus 4.8 和 Claude Opus 5 上进入 beta 阶段：在对话轮次之间添加或移除工具，同时保留提示缓存。请在请求中包含 `mid-conversation-tool-changes-2026-07-01` beta 请求头。
* `fallbacks` 参数现在支持 `"default"` 模式，该模式按拒绝类别应用 Anthropic 推荐的回退模型。服务端回退处于 beta 阶段，`"default"` 模式需要 `server-side-fallback-2026-07-01` beta 请求头。请参阅[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)。
* 我们已移除 Claude Opus 4.7 的[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)。对 `claude-opus-4-7` 发送带有 `speed: "fast"` 的请求现在会返回错误；与 Claude Opus 4.6 不同，它们不会回退到标准速度。Claude Opus 4.7 本身仍可在标准速度下使用。要继续使用快速模式，请迁移到 [Claude Opus 5](https://platform.claude.com/docs/zh-CN/models/opus-5/migration-guide#migrating-from-claude-opus-47) 或 Claude Opus 4.8。更多信息请参阅[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#supported-models)。

### 2026 年 7 月 22 日

* 您现在可以在 Claude Managed Agents 智能体的模型配置上设置 `effort` 级别。在[创建智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#create-an-agent)时在 `model` 对象内传递 `effort`。有关每个级别的作用，请参阅 [Effort 级别](https://platform.claude.com/docs/zh-CN/build-with-claude/effort#effort-levels)。
* Claude Managed Agents 的 Webhook 现已涵盖环境和记忆存储的生命周期：四种 `environment.*` 事件类型和三种 `memory_store.*` 事件类型。您无需轮询即可响应环境和记忆存储的生命周期变化。请参阅[订阅 Webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks#supported-event-types) 中的环境事件和记忆存储事件标签页。
* 创建 Claude Managed Agents 会话时，您现在可以[使用初始事件为其播种](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#seed-the-session-with-initial-events)。在 `POST /v1/sessions` 上传递 `initial_events`，最多包含 50 个 `user.message` 和 `user.define_outcome` 事件。非空列表会在同一次调用中启动智能体循环，因此您无需单独发送事件请求来开始工作。
* 在[更新 Claude Managed Agents 智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#update-an-agent)时，`version` 字段现在是可选的。提供它可实现乐观并发控制（不匹配会返回 409 错误），或省略它以无条件应用更新。请参阅[更新语义](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#update-semantics)。
* Claude Managed Agents 会话线程事件流现已支持[事件增量](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#event-deltas)。`GET /v1/sessions/{session_id}/threads/{thread_id}/stream` 接受与会话级流相同的 `event_deltas[]` 查询参数，因此您可以在模型生成时预览子智能体的文本。一个连接仅预览它正在读取的线程。请参阅[预览会话线程事件](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#preview-session-thread-events)。

### 2026 年 7 月 17 日

* Claude Console 中的旧版 **Workbench**（[platform.claude.com/workbench](https://platform.claude.com/workbench)）即将停用，访问将于 2026 年 8 月 17 日结束。更新后的 [Workbench](https://platform.claude.com/playground) 不支持已保存的提示、变量和评估。您可以从横幅以及**组织设置**下导出任何想要保留的数据。更多信息请参阅 Claude 帮助中心中的[如何使用 Workbench？](https://support.claude.com/en/articles/8606378-how-do-i-use-the-workbench)。
* 用于生成、改进和模板化提示的实验性提示工具 API（`/v1/experimental/generate_prompt`、`/v1/experimental/improve_prompt` 和 `/v1/experimental/templatize_prompt`）将于 2026 年 8 月 17 日随 Workbench 一同停用。移除后，对这些端点的请求将返回错误。

### 2026 年 7 月 15 日

* [对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)已在 Claude Fable 5、Claude Mythos 5 和 Claude Opus 4.8 上提供，覆盖 Claude API、[Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 和 [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)。无需 beta 请求头。此条更正了之前的可用性说明。

### 2026 年 7 月 14 日

* 您现在可以使用 [Admin API](https://platform.claude.com/docs/zh-CN/api/admin) 管理您 **Claude Enterprise**（claude.ai）组织中的人员，面向所有 Claude Enterprise 组织提供 beta 版：列出成员并按电子邮件地址查找、更改成员角色、移除成员、发送和撤回邀请、管理群组及其成员资格，以及读取自定义角色。群组和自定义角色请求需要 `anthropic-beta: ce-user-management-2026-07-13` beta 请求头；成员和邀请请求无需 beta 请求头。具有 `read:org_audit` 作用域的 Admin API 密钥也可以调用所有用户管理 `GET` 端点。请参阅[用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management)。

### 2026 年 7 月 10 日

* [Dreams](https://platform.claude.com/docs/zh-CN/managed-agents/dreams)（研究预览版）现已支持 Claude Fable 5 和 Claude Sonnet 5。请参阅[支持的模型](https://platform.claude.com/docs/zh-CN/managed-agents/dreams#limits)。
* 我们扩展了 [Access Transparency](https://platform.claude.com/docs/zh-CN/manage-claude/access-transparency) 中关于 `cmek_preserve` 事件的文档，新增了过滤器示例、示例事件负载以及两个保留原因代码（`policy_violation_investigation`、`csae_report`）。文档现在还阐明，无论保留是由人工审核员还是自动化安全管道发起，都会写入保留事件。请参阅 [CMEK 内容保留](https://platform.claude.com/docs/zh-CN/manage-claude/access-transparency#cmek-content-preservation)。

### 2026 年 7 月 8 日

* 您现在可以在 [Claude Console](https://platform.claude.com/settings/keys) 中创建 API 密钥或 Admin API 密钥时设置过期时间。可选择预设值、自定义时长或**永不过期**。对于有效期至少为 7 天的密钥，Anthropic 会在过期前向创建者发送电子邮件。现有密钥不受影响。Admin API 在 [`expires_at`](https://platform.claude.com/docs/zh-CN/api/admin/api_keys/list) 字段中报告每个密钥的过期时间。请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-expiration)。

### 2026 年 7 月 2 日

* 我们添加了 `agent-memory-2026-07-22` beta 请求头，它改变了[列出记忆](https://platform.claude.com/docs/zh-CN/managed-agents/memory#list-memories)（`GET /v1/memory_stores/{memory_store_id}/memories`）的行为：结果以稳定的、服务端定义的顺序返回，`order_by` 和 `order` 参数被忽略；`depth` 仅接受 `0`、`1` 或省略（其他值返回 `400` 错误）；`path_prefix` 必须以 `/` 结尾，并匹配完整路径段而非子字符串。未使用该请求头签发的分页游标在使用该请求头时无效，因此在采用它时请从第一页重新开始。在记忆存储端点上，`agent-memory-2026-07-22` 取代 `managed-agents-2026-04-01`；同时发送两者会返回 `400` 错误。2026 年 7 月 22 日，`managed-agents-2026-04-01` 请求头将采用相同的列表行为。请参阅 [Beta 请求头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
* Python（0.116.0）、TypeScript（0.110.0）、Go（1.56.0）、Java（2.48.0）、Ruby（1.55.0）、PHP（0.36.0）、C#（12.35.0）和 CLI（1.16.0）SDK 现在在所有记忆存储调用上发送 `agent-memory-2026-07-22`，而非 `managed-agents-2026-04-01`。如果您的代码在记忆存储调用上显式传递 `betas`，请在那里将 `managed-agents-2026-04-01` 替换为 `agent-memory-2026-07-22`，而不是添加第二个值。

### 2026 年 7 月 1 日

* 我们已恢复对 Claude Fable 5 和 Claude Mythos 5 的访问。更多信息请参阅[我们的声明](https://www.anthropic.com/news/redeploying-fable-5)。

### 2026 年 6 月 30 日

* 我们发布了 **Claude Sonnet 5**（`claude-sonnet-5`），这是我们 Sonnet 模型系列的下一代产品，推广期定价为每 MTok $2 / $10（已于 2026 年 8 月 10 日成为标准价格）。Claude Sonnet 5 支持 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)、128k 最大输出令牌，以及与 Claude Sonnet 4.6 相同的工具和平台功能集，但 [Priority Tier](https://platform.claude.com/docs/zh-CN/api/service-tiers#supported-models) 除外，该功能在 Claude Sonnet 5 上不可用。迁移时有三项行为变更：[adaptive thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)（自适应思考）现在默认开启；手动扩展思考（`thinking: {type: "enabled", budget_tokens: N}`）已被移除并返回 400 错误（它在 Sonnet 4.6 上已被弃用）；将采样参数（`temperature`、`top_p`、`top_k`）设置为非默认值会返回 400 错误。Claude Sonnet 5 还使用了新的分词器，对于相同的文本会产生大约多 30% 的令牌。确切的增幅取决于内容和工作负载形态。有关详细信息和迁移指南，请参阅 [Claude Sonnet 5 的新功能](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5)。有关行为差异和特定于模型的提示模式，请参阅[为 Claude Sonnet 5 编写提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-sonnet-5)。
* Claude Managed Agents 会话事件流现在支持[事件增量](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#event-deltas)。通过在 `GET /v1/sessions/{session_id}/events/stream` 上使用 `event_deltas[]` 查询参数来选择启用。`event_start` 和 `event_delta` 事件会在完整的 `agent.message` 事件到达之前，预览智能体消息在生成过程中的文本。
* Claude Managed Agents 的[列出会话](https://platform.claude.com/docs/zh-CN/managed-agents/session-operations#listing-sessions)现在支持向后分页。`GET /v1/sessions` 会在 `next_page` 之外返回一个 `prev_page` 游标；将其作为 `page` 参数传递即可返回上一页。请参阅[分页](https://platform.claude.com/docs/zh-CN/api/overview#pagination)。
* 创建 Claude Managed Agents 会话时，您现在可以[为该会话覆盖智能体的配置](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#override-agent-configuration-for-a-session)。传递带有 `type: "agent_with_overrides"` 的 `agent`，即可为单个会话替换模型、系统提示、工具、MCP 服务器或技能。智能体本身保持不变。
* Claude Managed Agents 保管库现在支持在[环境变量凭据](https://platform.claude.com/docs/zh-CN/managed-agents/vaults#add-a-credential)（"环境变量"选项卡）上设置 `injection_location`。它控制凭据的值在出口处是被替换到智能体的出站请求头、请求体，还是两者皆有。
* Claude Managed Agents 的 Webhook 现在涵盖智能体、部署和部署运行的生命周期。您无需轮询即可对新发布的智能体版本、已暂停的部署或失败的计划运行做出响应。请参阅[订阅 Webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks#supported-event-types) 中的"智能体事件"、"部署事件"和"部署运行事件"选项卡。

### 2026 年 6 月 29 日

* 我们已移除 Claude Opus 4.6 的 [fast mode](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)（快速模式）。向 `claude-opus-4-6` 发送带有 `speed: "fast"` 的请求不再以快速速度或高级定价运行：它们以标准速度运行，按标准费率计费，并且不会返回错误。响应的 `usage.speed` 字段会报告所使用的速度。要继续使用快速模式，请迁移到 [Claude Opus 4.8](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。在[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#supported-models)中了解更多信息。

### 2026 年 6 月 26 日

* 我们提高了整个 Claude API 的[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)。Claude Sonnet 和 Claude Haiku 的速率限制现在在每个使用层级上都与 Claude Opus 一致，并且使用层级已合并为三个：Start、Build 和 Scale。大多数组织会升至更高的层级，没有任何组织获得比以前更低的限制，且无需采取任何操作。您可以在 [Claude Console](https://platform.claude.com/settings/limits) 中查看您的层级和当前限制。

### 2026 年 6 月 25 日

* 我们已弃用 Claude Opus 4.7 的[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)，并将于 2026 年 7 月 24 日移除。移除后，向 `claude-opus-4-7` 发送带有 `speed: "fast"` 的请求将返回错误。请迁移到 Claude Opus 4.8 的快速模式。在[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#supported-models)中了解更多信息。

### 2026 年 6 月 22 日

* **MCP 隧道**（研究预览版）：管理 API 已从 Admin API 上的 `/v1/organizations/tunnels` 迁移到 Claude API 上的 `/v1/tunnels`。新接口使用 `anthropic-beta: mcp-tunnels-2026-06-22` 请求头和 `workspace:manage_tunnels` WIF 作用域。旧接口在迁移窗口期内仍然可用。请参阅[隧道 API 参考](https://platform.claude.com/docs/zh-CN/api/beta/tunnels)。

### 2026 年 6 月 18 日

* Python、TypeScript、Go、Java、Ruby、PHP 和 C# SDK 现在支持 `code_execution_20260120`，这是[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)的一个版本，它增加了 REPL 状态持久化，并且是[程序化工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling)所需的最低版本。要采用它，请将工具的 `type` 设置为 `code_execution_20260120`；无需 beta 请求头。它可在 Claude Fable 5、Claude Mythos 5、Claude Opus 4.5 及更新版本以及 Claude Sonnet 4.5 及更新版本上使用；请参阅代码执行工具的[兼容性](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#compatibility)部分。

### 2026 年 6 月 15 日

* 我们已停用 Claude Sonnet 4 模型（`claude-sonnet-4-20250514`）和 Claude Opus 4 模型（`claude-opus-4-20250514`）。在 Claude API 上对这些模型的所有请求现在都将返回错误。我们建议分别升级到 [Claude Sonnet 4.6](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison) 和 [Claude Opus 4.8](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)。研究人员可以通过[外部研究人员访问计划](https://support.claude.com/en/articles/9125743-what-is-the-external-researcher-access-program)申请持续访问权限。

### 2026 年 6 月 11 日

* [代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)现在支持 `code_execution_20260521`，它在工具描述中披露了每个单元格 90 秒的执行时间限制，以便 Claude 能够为长时间运行的单元格做好预算。无需 beta 请求头。
* [网页搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)和[网页抓取工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)现在支持 `web_search_20260318` 和 `web_fetch_20260318`，新增了 `response_inclusion` 参数，可在智能体工作流中从 API 响应中删除已消费的结果块。无需 beta 请求头。

### 2026 年 6 月 10 日

* 用于列出[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)待处理工作的 `GET /v1/environments/{id}/work` 端点现已在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上可用。有关授权该端点的 `GetEnvironment` 操作，请参阅 [Claude Platform on AWS 的 IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions)。

### 2026 年 6 月 9 日

* 我们发布了 **Claude Fable 5**（`claude-fable-5`），这是我们能力最强的广泛发布模型，同时还为 Project Glasswing 参与者发布了 **Claude Mythos 5**（`claude-mythos-5`）。两个模型默认支持 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)、128k 最大输出令牌以及始终开启的[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)。有关功能、API 变更和可用性，请参阅 [Claude Fable 5 和 Claude Mythos 5 简介](https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5)。
* Claude Fable 5 和 Claude Mythos 5 使用随 Claude Opus 4.7 引入的分词器。与 Claude Opus 4.7 之前的模型相比，相同的文本会产生大约多 30% 的令牌。确切的增幅取决于内容和工作负载形态。使用带有 `model: "claude-fable-5"` 的[令牌计数 API](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting#token-counts-on-claude-fable-5) 来衡量您的提示在新分词器下的令牌数。
* Claude Fable 5 会在请求上以及响应生成期间运行安全分类器。当分类器拒绝某个请求时，Messages API 会返回 `stop_reason: "refusal"`。对于在生成任何输出之前被拒绝的请求，您不会被计费。一个可选启用的 `fallbacks` 参数（在 Claude API 和 Claude Platform on AWS 上处于 beta 阶段；Message Batches API 不支持）会在另一个模型上重新运行被拒绝的请求，并按回退模型的费率计费。请参阅[处理停止原因](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons)。
* 拒绝响应上的 [`stop_details.category`](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response) 字段现在在 Claude Fable 5 上包含 `"reasoning_extraction"`，当请求因 Anthropic 服务条款中关于逆向工程或复制模型输出的限制而被阻止时返回。现有的 `"cyber"` 和 `"bio"` 类别保持不变。无需 beta 请求头。
* 在 Claude Fable 5 和 Claude Mythos 5 上，[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)是唯一的思考模式：不支持 `thinking: {"type": "disabled"}`，也不支持手动扩展思考预算和助手预填充（两者都会返回 400 错误）。请参阅[从 Claude Mythos Preview 迁移到 Claude Mythos 5](https://platform.claude.com/docs/zh-CN/models/fable-5/migration-guide#migrating-from-claude-mythos-preview)。
* 在 Claude Fable 5 和 Claude Mythos 5 上，`thinking.display` 默认为 `"omitted"`，与 Claude Opus 4.8、Claude Opus 4.7 和 Claude Mythos Preview 相同；设置 `display: "summarized"` 可接收可读的思考摘要。原始思维链永远不会被返回；在同一模型上的多轮对话中，请原样传回思考块。请参阅 [Claude Fable 5 和 Claude Mythos 5 上的思考输出](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#thinking-output-on-claude-fable-5-and-claude-mythos-5)。
* Claude Fable 5 要求 30 天数据保留，在零数据保留下不可用。请参阅[特定于模型的数据保留要求](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#model-specific-data-retention-requirements)。
* Claude Managed Agents 现在支持[计划部署](https://platform.claude.com/docs/zh-CN/managed-agents/scheduled-deployments)，让您可以按 cron 计划运行会话，而无需管理自己的调度器。
* Claude Managed Agents 保管库现在支持[环境变量凭据](https://platform.claude.com/docs/zh-CN/managed-agents/vaults#add-a-credential)，因此您可以安全地将密钥注入智能体的沙箱，供 CLI、SDK 以及其他通过环境变量进行身份验证的服务使用。
* [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 的[活动源](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-activity-feed)（`GET /v1/compliance/activities`）现已在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上可用。有关授权该端点的 `ListComplianceActivities` 操作，请参阅 [Claude Platform on AWS 的 IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#compliance)。
* `session.thread_*` Webhook 事件现在包含一个 `session_thread_id` 字段，用于标识触发该事件的多智能体线程。
* 我们发布了一个处于 beta 阶段的 [Swift 包](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/apple-foundation-models)，它将 Claude 作为服务器端 `LanguageModel` 添加到 Apple 的 Foundation Models 框架中。在 iOS 27、macOS 27、visionOS 27 和 watchOS 27（beta）上，通过与 Apple 设备端模型相同的 `LanguageModelSession` API 调用 Claude。

### 2026 年 6 月 5 日

* 我们宣布弃用 Claude Opus 4.1 模型（`claude-opus-4-1-20250805`），计划于 2026 年 8 月 5 日在 Claude API 上停用。我们建议迁移到 [Claude Opus 4.8](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。

### 2026 年 6 月 2 日

* [顾问工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool)现在支持 `max_tokens` 参数，用于限制顾问模型每次调用的输出，从而为不需要完整长度顾问响应的工作负载降低延迟和输出令牌成本。在顾问工具定义上设置 `tools[].max_tokens`；请参阅[限制顾问输出](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool#capping-advisor-output)。
* 在 Claude API 上，当请求返回 `stop_reason: "refusal"` 且 Claude 未生成任何输出时，您不再为该请求计费。有关检测和处理拒绝的信息，请参阅[流式传输拒绝](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/handle-streaming-refusals)。

### 2026 年 5 月 29 日

* Claude Managed Agents 的 [Webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks)、[多智能体编排](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)和[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)现已在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上可用。有关新的 IAM 操作和 `AnthropicSelfHostedEnvironmentAccess` 托管策略，请参阅 [Claude Platform on AWS 的 IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions)。

### 2026 年 5 月 28 日

* 我们发布了 **Claude Opus 4.8**（claude-opus-4-8），这是我们能力最强的广泛发布模型。Claude Opus 4.8 在 Claude API、Amazon Bedrock、Google Cloud 和 Microsoft Foundry 上默认支持 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)、128k 最大输出令牌，以及与 Claude Opus 4.7 相同的工具和平台功能集。有关基准设置、功能和迁移指南，请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。
* 我们发布了[对话中途系统消息](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)。在 Claude Opus 4.8 上，您可以在 `messages` 数组中于用户轮次之后发送 `role: "system"` 消息（须遵守[放置规则](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages#limitations)），从而在长时间运行的会话中指令发生变化时保留提示缓存命中。无需 beta 请求头。
* 拒绝响应上的 [`stop_details`](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#refusal-response) 字段现已公开记录；它返回一个 `category`（`cyber`、`bio` 或 `null`）和一个人类可读的 `explanation`，以便您的应用程序可以将不同类别的拒绝路由到正确的下一步。无需 beta 请求头。
* 在 Claude Opus 4.8 上，[effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)在所有界面上默认为 `high`，包括 Claude Code 和 Messages API。
* 在 Claude Opus 4.8 上，[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)的最小可缓存提示长度为 1,024 个令牌，低于 Claude Opus 4.7。
* 启用[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)后，Claude Opus 4.8 仅在某个轮次需要时才触发推理，与相同 effort 级别下的 Claude Opus 4.7 相比，减少了浪费的思考令牌。
* Claude Opus 4.8 支持[高分辨率图像输入](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#high-resolution-image-support-on-claude-opus-4-7)（长边最多 2576 像素），与 Claude Opus 4.7 相同。
* [任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)现在支持 Claude Opus 4.8。
* [顾问工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool)现在支持 Claude Opus 4.8。
* [计算机使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)现在支持 Claude Opus 4.8。
* Claude Opus 4.8 的[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)仅在 Claude API 上作为研究预览版提供。
* 在 Claude Opus 4.8 上，将采样参数 `temperature`、`top_p` 或 `top_k` 设置为非默认值会返回 400 错误，与 Claude Opus 4.7 相同。有关详细信息，请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。
* 在 Claude Code 中，我们已将 Auto 模式扩展到更多用户，用于长时间运行的任务。请参阅 [Claude Code 文档](https://code.claude.com/docs)。
* 在 Claude Code 中，Max 计划用户现在在 Claude Opus 4.8 上默认使用[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)。请参阅 [Claude Code 文档](https://code.claude.com/docs)。
* 在 Claude Code 中，Workflows 作为研究预览版提供，让您可以定义和运行多步骤智能体计划。请参阅 [Claude Code 文档](https://code.claude.com/docs)。
* 我们已弃用 Claude Opus 4.6 的[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)，将在发布后约 30 天移除。请迁移到 Claude Opus 4.8 或 Claude Opus 4.7 的快速模式。在[快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode#supported-models)中了解更多信息。
* 有关本次发布中 claude.ai、Cowork、Claude for Microsoft 365 和其他 Claude 应用的更新，请参阅 [Claude 应用发布说明](https://support.claude.com/en/articles/12138966-release-notes)。

### 2026 年 5 月 27 日

* Messages API 响应现在包含 [`usage.output_tokens_details.thinking_tokens`](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking#budget-rules-and-tuning)，报告计费的输出令牌中有多少是扩展思考。流式传输时，该明细仅出现在最终的 `message_delta` 事件上。无需 beta 请求头。

### 2026 年 5 月 19 日

* [MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview)现已作为研究预览版提供，因此您可以连接到私有网络中的 MCP 服务器。
* 自托管沙箱现已可用于 Claude Managed Agents，作为在 Anthropic 基础设施中运行工具执行的替代方案。请参阅[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)。
* 使用 Claude Managed Agents，您现在可以更新与活动会话关联的智能体的 MCP 服务器和工具配置。
* 使用 Claude Managed Agents，来自 `agent_toolset` 和 MCP 工具的超过 100K 字符（约 25K 令牌）的大型输出现在会自动溢出到沙箱中的文件。模型会收到带有文件路径的截断预览，并可以从那里读取完整内容。

### 2026 年 5 月 18 日

* [网页搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)现在返回更丰富的 SEC 文件数据，使金融研究智能体、收益分析和尽职调查工作流更容易以带引用的一手来源为依据。

### 2026 年 5 月 13 日

* 我们已在公开 beta 中发布[缓存诊断](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics)。在 Messages 请求上传递 `diagnostics.previous_message_id`，API 会报告一个 `cache_miss_reason`，解释提示缓存前缀与上一轮次在何处发生分歧。请在您的请求中包含 `cache-diagnosis-2026-04-07` beta 请求头。

### 2026 年 5 月 12 日

* [快速模式](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)（研究预览版）现在支持 Claude Opus 4.7。设置 `speed: "fast"` 并配合 `model: "claude-opus-4-7"` 和 `fast-mode-2026-02-01` beta 请求头，即可以高级定价获得显著更快的输出令牌生成速度。定价、速率限制和访问权限与 Opus 4.6 快速模式相同；感兴趣的客户请加入[候补名单](https://claude.com/fast-mode)。

### 2026 年 5 月 11 日

* 我们发布了 **Claude Platform on AWS**，将 Claude API 带到可通过 AWS 访问的 Anthropic 托管基础设施上，并支持 AWS 计费和 IAM 身份验证。通过原生 AWS 端点访问完整的 Messages API、Files API、Message Batches API、Claude Managed Agents、Agent Skills、代码执行和工具使用。在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 中了解更多信息。

### 2026 年 5 月 6 日

* [多智能体编排](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)和[成果](https://platform.claude.com/docs/zh-CN/managed-agents/define-outcomes)现已在标准 `managed-agents-2026-04-01` beta 请求头下进入公开 beta。
* Claude Managed Agents 保管库凭据后台刷新现在支持 `mcp_oauth` 凭据。请参阅[使用保管库进行身份验证](https://platform.claude.com/docs/zh-CN/managed-agents/vaults)。
* 现在支持 Claude Managed Agents 的 Webhook。Webhook 事件类型包括会话和保管库生命周期事件。请参阅[订阅 Webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks)。
* Claude Managed Agents 现在支持更多筛选和排序选项。会话可以按状态筛选，事件可以按类型筛选。事件现在可以按创建时间筛选。
* Claude Managed Agents 的 [Dreams](https://platform.claude.com/docs/zh-CN/managed-agents/dreams) 现已作为研究预览版提供。一个 dream 会读取现有的记忆存储以及过去的会话记录，并生成一个重新组织的输出记忆存储，其中重复项被合并、过时条目被替换、新见解被呈现。Dream 端点受 `dreaming-2026-04-21` beta 请求头限制。[申请访问权限](https://claude.com/form/claude-managed-agents)以试用。

### 2026 年 5 月 4 日

* 我们发布了 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)（工作负载身份联合）。使用来自您自己的身份提供商（AWS IAM、Google Cloud、GitHub Actions、Kubernetes、Microsoft Entra ID、Okta、SPIFFE 等）的短期 OIDC 令牌向 Claude API 验证工作负载身份，而不是使用长期静态 API 密钥。在 Claude Console 中配置颁发者和联合规则，SDK 会自动处理令牌交换和刷新。请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)。

### 2026 年 4 月 30 日

* 我们已停用 Claude Sonnet 4.5 和 Claude Sonnet 4 的 1M 令牌上下文窗口 beta（`context-1m-2025-08-07`）。该 beta 请求头现在对这些模型没有任何效果，超过标准 200k 令牌上下文窗口的请求会返回错误。要使用 1M 上下文窗口，请迁移到 [Claude Sonnet 4.6](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison) 或 [Claude Opus 4.6](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)，在这些模型上它以标准定价提供，无需 beta 请求头。

### 2026 年 4 月 29 日

* 我们发布了 [Claude API 技能](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill)，这是一个开源的 [Agent Skill](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview)，为 Claude 提供跨 8 种语言在 Messages API 和 Claude Managed Agents 上进行构建的最新参考资料。该技能与 Claude Code 捆绑提供，并可在 [Anthropic 技能仓库](https://github.com/anthropics/skills/tree/main/skills/claude-api)中获取。

### 2026 年 4 月 24 日

* 我们发布了 [Rate Limits API](https://platform.claude.com/docs/zh-CN/manage-claude/rate-limits-api)，允许管理员以编程方式查询为其组织和工作区配置的速率限制。

### 2026 年 4 月 23 日

* Claude Managed Agents 的记忆功能现已在标准 `managed-agents-2026-04-01` 请求头下进入公开 beta。有关完整的集成指南，请参阅[使用智能体记忆](https://platform.claude.com/docs/zh-CN/managed-agents/memory)。

### 2026 年 4 月 20 日

* 我们已停用 Claude Haiku 3 模型（`claude-3-haiku-20240307`）。对该模型的所有请求现在都将返回错误。我们建议升级到 [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)。

### 2026 年 4 月 16 日

* 我们发布了 [Claude Opus 4.7](https://www.anthropic.com/news/claude-opus-4-7)，这是我们在复杂推理和智能体编码方面能力最强的广泛发布模型，定价与 Opus 4.6 相同，为每 MTok $5 / $25。有关能力改进、新功能和更新的分词器，请参阅 [Claude Opus 4.7 的新功能](https://platform.claude.com/docs/zh-CN/about-claude/models/whats-new-claude-4-7)。Opus 4.7 相对于 Opus 4.6 包含 API 破坏性变更；升级前请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。
* [Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 现已向所有 Amazon Bedrock 客户开放。Claude Opus 4.7 和 Claude Haiku 4.5 可通过位于 `/anthropic/v1/messages` 的 Messages API 端点从 Bedrock 控制台自助获取，覆盖 27 个 AWS 区域，提供全球和区域端点。
* 我们已在 Claude Opus 4.7 上以 beta 形式发布[任务预算](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)。为 Claude 提供一个针对完整智能体循环（思考、工具调用、工具结果和输出）的建议性令牌预算，模型会看到一个实时倒计时，并利用它来确定工作优先级，在预算消耗时优雅地完成。请在您的请求中包含 `task-budgets-2026-03-13` beta 请求头。
* Claude Opus 4.7 支持[高分辨率图像输入](https://platform.claude.com/docs/zh-CN/build-with-claude/vision#high-resolution-image-support-on-claude-opus-4-7)，将最大图像分辨率从长边 1568 像素提高到 2576 像素，以提升在计算机使用、屏幕截图理解和文档分析方面的性能。高分辨率支持是自动的，无需 beta 请求头；图像使用的图像令牌可能比之前的模型多出约 3 倍。
* 我们在 Claude Opus 4.7 上新增了 `xhigh` [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 级别。`xhigh` 介于 `high` 和 `max` 之间，针对令牌预算达数百万的长时间运行（超过 30 分钟）智能体和编码任务进行了调优。无需 beta 请求头。

### 2026 年 4 月 14 日

* 我们宣布弃用 Claude Sonnet 4 模型（`claude-sonnet-4-20250514`）和 Claude Opus 4 模型（`claude-opus-4-20250514`），计划于 2026 年 6 月 15 日在 Claude API 上停用。我们建议分别迁移到 [Claude Sonnet 4.6](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison) 和 [Claude Opus 4.8](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。

### 2026 年 4 月 9 日

* 我们已在公开 beta 中发布[顾问工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool)。将一个更快的执行器模型与一个更高智能的顾问模型配对，后者在生成过程中提供战略指导，从而使长周期智能体工作负载获得接近顾问模型单独运行的质量，而大部分令牌生成则以执行器模型的费率进行。请在您的请求中包含 beta 请求头 `advisor-tool-2026-03-01`。

### 2026 年 4 月 8 日

* 我们已在公开 beta 中发布 **Claude Managed Agents**，这是一个完全托管的智能体框架，用于将 Claude 作为自主智能体运行，具备安全沙箱、内置工具和服务器发送事件流式传输。通过 API 创建智能体、配置容器并运行会话。所有端点都需要 `managed-agents-2026-04-01` beta 请求头。在 [Claude Managed Agents 概述](https://platform.claude.com/docs/zh-CN/managed-agents/overview)中了解更多信息。
* 我们发布了 **`ant` CLI**，这是一个用于 Claude API 的命令行客户端，可实现与 Claude API 的更快交互、与 Claude Code 的原生集成，以及在 YAML 文件中对 API 资源进行版本管理。在 [CLI 快速入门](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/quickstart)中了解更多信息。

### 2026 年 4 月 7 日

* 我们宣布 [Claude Mythos Preview](https://anthropic.com/glasswing) 作为 [Project Glasswing](https://anthropic.com/glasswing) 的一部分，以受限研究预览版的形式提供，用于防御性网络安全工作。访问仅限受邀者。
* [Messages API](https://platform.claude.com/docs/zh-CN/api/messages) 现已作为研究预览版在 Amazon Bedrock 上提供。位于 `/anthropic/v1/messages` 的新 Claude in Amazon Bedrock 端点使用与第一方 Claude API 相同的请求结构，并在零运营商访问的 AWS 托管基础设施上运行。在 `us-east-1` 可用；请联系您的 Anthropic 客户经理申请访问权限。在 [Claude in Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock) 中了解更多信息。

### 2026 年 3 月 30 日

* 我们已将 Claude Opus 4.6 和 Sonnet 4.6 在 [Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#extended-output-beta) 上的 `max_tokens` 上限提高到 300k。包含 `output-300k-2026-03-24` beta 请求头，即可为长篇内容、结构化数据和大型代码生成任务生成更长的单轮输出。
* 我们将于 **2026 年 4 月 30 日**停用 Claude Sonnet 4.5 和 Claude Sonnet 4 的 1M 令牌上下文窗口 beta。在该日期之后，`context-1m-2025-08-07` beta 请求头将对这些模型没有任何效果，超过标准 200k 令牌上下文窗口的请求将返回错误。要继续使用 1M 上下文窗口，请迁移到 [Claude Sonnet 4.6](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison) 或 [Claude Opus 4.6](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)，它们以标准定价支持完整的 1M 令牌上下文窗口，无需 beta 请求头。

### 2026 年 3 月 18 日

* 我们已向 [Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 添加了模型能力字段。`GET /v1/models` 和 `GET /v1/models/{model_id}` 现在返回 `max_input_tokens`、`max_tokens` 和一个 `capabilities` 对象。查询 API 即可了解每个模型支持的功能。

### 2026 年 3 月 16 日

* 我们发布了扩展思考的 `display` 字段，让您可以从响应中省略思考内容以实现更快的流式传输。设置 `thinking.display: "omitted"` 即可接收 `thinking` 字段为空且保留 `signature` 的思考块，以实现多轮连续性。计费不变。在[控制思考显示](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#controlling-thinking-display)中了解更多信息。

### 2026 年 3 月 13 日

* [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)已在 Claude Opus 4.6 和 Sonnet 4.6 上结束 beta，以标准定价提供。对于这些模型，超过 200k 令牌的请求会自动生效，无需 beta 请求头。1M 令牌上下文窗口在 Claude Sonnet 4.5 和 Sonnet 4 上仍处于 beta 阶段。
* 我们已移除所有受支持模型的专用 1M 速率限制。您的标准账户限制现在适用于所有上下文长度。
* 使用 1M 令牌上下文窗口时，我们已将每个请求的媒体限制从 100 个提高到 600 个图像或 PDF 页面。

### 2026 年 2 月 19 日

* 我们为 Messages API 发布了**自动缓存**。在请求体中添加一个 `cache_control` 字段，系统会自动缓存最后一个可缓存块，并随着对话增长将缓存点向前移动。无需手动管理断点。可与现有的块级缓存控制配合使用，以实现细粒度优化。在 Claude API 和 Microsoft Foundry（预览版）上可用。在[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#automatic-caching)中了解更多信息。
* 我们已停用 Claude Sonnet 3.7 模型（`claude-3-7-sonnet-20250219`）和 Claude Haiku 3.5 模型（`claude-3-5-haiku-20241022`）。对 Claude Sonnet 3.7 的所有请求现在都将返回错误。在 Claude API 上对 Claude Haiku 3.5 的请求现在将返回错误；它在 Amazon Bedrock 和 Google Cloud 上仍然可用。我们建议分别升级到 [Claude Sonnet 4.6](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison) 和 [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)。研究人员可以通过[外部研究人员访问计划](https://support.claude.com/en/articles/9125743-what-is-the-external-researcher-access-program)申请持续访问权限。
* 我们宣布弃用 Claude Haiku 3 模型（`claude-3-haiku-20240307`），计划于 2026 年 4 月 20 日停用。我们建议迁移到 [Claude Haiku 4.5](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)。在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。

### 2026 年 2 月 17 日

* 我们发布了 [Claude Sonnet 4.6](https://www.anthropic.com/news/claude-sonnet-4-6)，这是我们最新的平衡型模型，兼具速度与智能，适用于日常任务。Sonnet 4.6 在消耗更少令牌的同时提供了更好的智能体搜索性能。Sonnet 4.6 支持[扩展思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)和 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)（beta）。有关详细信息，请参阅[模型与定价](https://platform.claude.com/docs/zh-CN/about-claude/models)。
* API [代码执行](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)现在**与网页搜索或网页抓取一起使用时免费**。沙箱化代码执行提升了模型能力和令牌效率。有关独立使用的信息，请参阅[定价详情](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool#usage-and-pricing)。
* [网页搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)和[程序化工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling)现已可用，无需 beta 请求头。网页搜索和网页抓取现在支持[动态筛选](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool#dynamic-filtering)，它使用代码执行在结果到达上下文窗口之前对其进行筛选，以获得更好的性能并降低令牌成本。
* [代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)、[网页抓取工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)、[工具搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)、[工具使用示例](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/define-tools#providing-tool-use-examples)和[记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)不再需要 beta 请求头。

### 2026 年 2 月 7 日

* 我们已在研究预览版中为 Opus 4.6 推出了 [fast mode](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)（快速模式），通过 `speed` 参数提供显著更快的输出令牌生成速度。快速模式的速度最高可达 2.5 倍，采用高级定价。感兴趣的客户请加入[等候名单](https://claude.com/fast-mode)。

### 2026 年 2 月 5 日

* 我们推出了 [Claude Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6)，这是我们面向复杂智能体任务和长周期工作的最智能模型。Opus 4.6 推荐使用 [adaptive thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)（自适应思考）（`thinking: {type: "adaptive"}`）；手动思考（`type: "enabled"` 搭配 `budget_tokens`）已被弃用。Opus 4.6 不支持预填充助手消息。请在 [Claude 4.6 新特性](https://platform.claude.com/docs/zh-CN/about-claude/models/whats-new-claude-4-6)中了解更多信息。
* [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)不再需要 beta 标头，现已支持 Claude Opus 4.6。在新模型上，effort 取代 `budget_tokens` 用于控制思考深度。
* 我们已推出 beta 版 [compaction API](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)（压缩 API），提供服务器端上下文摘要功能，实现实际上无限长的对话。可在 Opus 4.6 上使用。
* 我们引入了[数据驻留控制](https://platform.claude.com/docs/zh-CN/manage-claude/data-residency)，允许您通过 `inference_geo` 参数指定模型推理的运行位置。对于 2026 年 2 月 1 日之后发布的模型，仅限美国的推理以 1.1 倍定价提供。
* [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)现已在 beta 版中面向 Claude Opus 4.6 提供，此外还支持 Sonnet 4.5 和 Sonnet 4。[长上下文定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#long-context-pricing)适用于超过 200k 输入令牌的请求。
* [细粒度工具流式传输](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/fine-grained-tool-streaming)在任何模型或平台上都不再需要 beta 标头。

### 2026 年 1 月 29 日

* [Structured outputs](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)（结构化输出）已在 Claude API 上面向 Claude Sonnet 4.5、Claude Opus 4.5 和 Claude Haiku 4.5 结束 beta 阶段。此版本包括扩展的 schema 支持、改进的语法编译延迟，以及无需 beta 标头的简化集成路径。`output_format` 参数已移至 `output_config.format`。现有 beta 用户在过渡期内可继续使用 beta 标头。结构化输出在 Amazon Bedrock 和 Microsoft Foundry 上仍处于公开 beta 阶段。

### 2026 年 1 月 12 日

* `console.anthropic.com` 现在会重定向到 `platform.claude.com`。作为 Claude 品牌整合的一部分，Claude Console 已迁移至新地址。现有书签和链接将通过自动重定向继续有效。更多详情请参阅 [2025 年 9 月 16 日的公告](https://platform.claude.com/docs/zh-CN/release-notes/overview#september-16-2025)。

### 2026 年 1 月 5 日

* 我们已停用 Claude Opus 3 模型（`claude-3-opus-20240229`）。对该模型的所有请求现在都将返回错误。我们建议升级到 [Claude Opus 4.5](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)，它以三分之一的成本提供显著提升的智能水平。研究人员可通过[外部研究人员访问计划](https://support.claude.com/en/articles/9125743-what-is-the-external-researcher-access-program)申请在 API 上继续访问 Claude Opus 3。

### 2025 年 12 月 19 日

* 我们宣布弃用 Claude Haiku 3.5 模型。请在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。

### 2025 年 12 月 4 日

* [结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)现已支持 Claude Haiku 4.5。

### 2025 年 11 月 24 日

* 我们推出了 [Claude Opus 4.5](https://www.anthropic.com/news/claude-opus-4-5)，这是我们最智能的模型，兼具最强能力与实用性能。非常适合复杂的专业任务、专业软件工程和高级智能体。在视觉、编码和计算机使用方面实现了跨越式改进，且价格比以往的 Opus 模型更加亲民。请在[模型概览](https://platform.claude.com/docs/zh-CN/about-claude/models)中了解更多信息。
* 我们已推出公开 beta 版的[程序化工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling)，允许 Claude 在代码执行中调用工具，以减少多工具工作流中的延迟和令牌使用量。
* 我们已推出公开 beta 版的[工具搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)，使 Claude 能够从大型工具目录中按需动态发现和加载工具。
* 我们已为 Claude Opus 4.5 推出公开 beta 版的 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)，允许您通过在响应详尽程度与效率之间进行权衡来控制令牌使用量。
* 我们已在 Python 和 TypeScript SDK 中添加了[客户端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#client-side-compaction-sdk)，在使用 `tool_runner` 时通过摘要自动管理对话上下文。

### 2025 年 11 月 21 日

* 搜索结果内容块现已在 Amazon Bedrock 上提供，无需 beta 标头。请在[搜索结果](https://platform.claude.com/docs/zh-CN/build-with-claude/search-results)中了解更多信息。

### 2025 年 11 月 19 日

* 我们在 [platform.claude.com/docs](https://platform.claude.com/docs) 推出了**全新的文档平台**。我们的文档现在与 Claude Console 并列呈现，提供统一的开发者体验。之前位于 docs.claude.com 的文档站点将重定向到新地址。

### 2025 年 11 月 18 日

* 我们推出了 **Claude in Microsoft Foundry**，通过 Azure 计费和 OAuth 身份验证将 Claude 模型带给 Azure 客户。可访问完整的 Messages API，包括扩展思考、提示缓存（5 分钟和 1 小时）、PDF 支持、Files API、Agent Skills 和工具使用。请在 [Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 中了解更多信息。

### 2025 年 11 月 14 日

* 我们已推出公开 beta 版的[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)，为 Claude 的响应提供有保证的 schema 一致性。使用 JSON 输出获取结构化数据响应，或使用严格工具使用获取经过验证的工具输入。适用于 Claude Sonnet 4.5 和 Claude Opus 4.1。要启用此功能，请使用 beta 标头 `structured-outputs-2025-11-13`。

### 2025 年 10 月 28 日

* 我们宣布弃用 Claude Sonnet 3.7 模型。请在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。
* 我们已停用 Claude Sonnet 3.5 模型。对这些模型的所有请求现在都将返回错误。
* 我们通过思考块清除（`clear_thinking_20251015`）扩展了上下文编辑功能，实现思考块的自动管理。请在[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)中了解更多信息。

### 2025 年 10 月 16 日

* 我们推出了 [Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)（`skills-2025-10-02` beta），这是一种扩展 Claude 能力的新方式。Skills 是由指令、脚本和资源组成的有组织的文件夹，Claude 会动态加载它们以执行专门任务。初始版本包括：

  * **Anthropic 管理的 Skills**：用于处理 PowerPoint (.pptx)、Excel (.xlsx)、Word (.docx) 和 PDF 文件的预构建 Skills
  * **自定义 Skills**：通过 Skills API（`/v1/skills` 端点）上传您自己的 Skills，以封装领域专业知识和组织工作流
  * Skills 需要启用[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)
  * 请在 [Agent Skills](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview) 和 [API 参考](https://platform.claude.com/docs/zh-CN/api/skills/create)中了解更多信息

### 2025 年 10 月 15 日

* 我们推出了 [Claude Haiku 4.5](https://www.anthropic.com/news/claude-haiku-4-5)，这是我们速度最快、最智能的 Haiku 模型，具有接近前沿的性能。非常适合实时应用、高吞吐量处理以及需要强大推理能力的成本敏感型部署。请在[模型概览](https://platform.claude.com/docs/zh-CN/about-claude/models)中了解更多信息。

### 2025 年 9 月 29 日

* 我们推出了 [Claude Sonnet 4.5](https://www.anthropic.com/news/claude-sonnet-4-5)，这是我们面向复杂智能体和编码的最佳模型，在大多数任务中具有最高的智能水平。请在[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)中了解更多信息。
* 我们为 Amazon Bedrock 和 Vertex AI 引入了[全球端点定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing#cloud-platform-pricing)。Claude API（第一方）定价不受影响。
* 我们引入了新的停止原因 `model_context_window_exceeded`，允许您在不计算输入大小的情况下请求尽可能多的令牌。请在[处理停止原因](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons)中了解更多信息。
* 我们已推出 beta 版的记忆工具，使 Claude 能够跨对话存储和查阅信息。请在[记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)中了解更多信息。
* 我们已推出 beta 版的上下文编辑功能，提供自动管理对话上下文的策略。初始版本支持在接近令牌限制时清除较早的工具结果和调用。请在[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)中了解更多信息。

### 2025 年 9 月 17 日

* 我们已为 Python 和 TypeScript SDK 推出 beta 版的工具助手，通过类型安全的输入验证以及用于在对话中自动处理工具的工具运行器，简化工具的创建和执行。详情请参阅 [Python SDK](https://github.com/anthropics/anthropic-sdk-python/blob/main/tools.md) 和 [TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript/blob/main/helpers.md#tool-helpers) 的文档。

### 2025 年 9 月 16 日

* 我们已将开发者产品统一到 Claude 品牌下。您会在我们的平台和文档中看到更新后的命名和 URL，但**我们的开发者接口将保持不变**。以下是一些值得注意的变化：

  * Claude Console（[console.anthropic.com](https://console.anthropic.com)）→ Claude Console（[platform.claude.com](https://platform.claude.com)）。在 2026 年 1 月 12 日之前，控制台可通过这两个 URL 访问。在该日期之后，[console.anthropic.com](https://console.anthropic.com) 将自动重定向到 [platform.claude.com](https://platform.claude.com)。
  * Anthropic Docs（[docs.anthropic.com](https://docs.anthropic.com)）→ Claude Docs（[docs.claude.com](https://docs.claude.com)）
  * Anthropic 帮助中心（[support.anthropic.com](https://support.anthropic.com)）→ Claude 帮助中心（[support.claude.com](https://support.claude.com)）
  * API 端点、标头、环境变量和 SDK 保持不变。您现有的集成将继续正常工作，无需任何更改。

### 2025 年 9 月 10 日

* 我们已推出 beta 版的网页抓取工具，允许 Claude 从指定的网页和 PDF 文档中检索完整内容。请在[网页抓取工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)中了解更多信息。
* 我们推出了 [Claude Code Analytics API](https://platform.claude.com/docs/zh-CN/manage-claude/claude-code-analytics-api)，使组织能够以编程方式访问 Claude Code 的每日汇总使用指标，包括生产力指标、工具使用统计和成本数据。

### 2025 年 9 月 8 日

* 我们推出了 beta 版的 [C# SDK](https://github.com/anthropics/anthropic-sdk-csharp)。

### 2025 年 9 月 5 日

* 我们在 Console 的[用量](https://console.anthropic.com/settings/usage)页面推出了[速率限制图表](https://platform.claude.com/docs/zh-CN/api/rate-limits#monitoring-your-rate-limits-in-the-console)，允许您监控 API 速率限制使用情况和缓存率随时间的变化。

### 2025 年 9 月 3 日

* 我们已推出对客户端工具结果中可引用文档的支持。请在[处理工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/handle-tool-calls)中了解更多信息。

### 2025 年 9 月 2 日

* 我们已推出公开 beta 版的[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool) v2，以 Bash 命令执行和直接文件操作功能（包括用其他语言编写代码）取代了原先仅支持 Python 的工具。

### 2025 年 8 月 27 日

* 我们推出了 beta 版的 [PHP SDK](https://github.com/anthropics/anthropic-sdk-php)。

### 2025 年 8 月 26 日

* 我们提高了 Claude API 上 Claude Sonnet 4 的 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)的速率限制。
* 1M 令牌上下文窗口现已在 Vertex AI 上提供。更多信息请参阅 [Claude on Vertex AI](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)。

### 2025 年 8 月 19 日

* 请求 ID 现在除了包含在现有的 `request-id` 标头中之外，还直接包含在错误响应正文中。请在[错误](https://platform.claude.com/docs/zh-CN/api/errors#error-shapes)中了解更多信息。

### 2025 年 8 月 18 日

* 我们发布了 [Usage & Cost API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api)，允许管理员以编程方式监控其组织的使用情况和成本数据。
* 我们在 Admin API 中添加了一个用于检索组织信息的新端点。详情请参阅[组织信息 Admin API 参考](https://platform.claude.com/docs/zh-CN/api/admin-api/organization/get-me)。

### 2025 年 8 月 13 日

* 我们宣布弃用 Claude Sonnet 3.5 模型（`claude-3-5-sonnet-20240620` 和 `claude-3-5-sonnet-20241022`）。这些模型将于 2025 年 10 月 28 日停用。我们建议迁移到 Claude Sonnet 4.5（`claude-sonnet-4-5-20250929`）以获得更好的性能和能力。请在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。
* 提示缓存的 1 小时缓存时长不再需要 beta 标头。请在[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#1-hour-cache-duration)中了解更多信息。

### 2025 年 8 月 12 日

* 我们已在 Claude API 和 Amazon Bedrock 上为 Claude Sonnet 4 推出 [1M 令牌上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)的 beta 支持。

### 2025 年 8 月 11 日

* 由于 API 的加速限制，部分客户在 API 使用量急剧增加后可能会遇到 429（`rate_limit_error`）[错误](https://platform.claude.com/docs/zh-CN/api/errors)。此前，在类似情况下会出现 529（`overloaded_error`）错误。

### 2025 年 8 月 8 日

* 搜索结果内容块已在 Claude API 和 Vertex AI 上结束 beta 阶段。此功能为 RAG 应用提供带有正确来源归属的自然引用。不再需要 beta 标头 `search-results-2025-06-09`。请在[搜索结果](https://platform.claude.com/docs/zh-CN/build-with-claude/search-results)中了解更多信息。

### 2025 年 8 月 5 日

* 我们推出了 [Claude Opus 4.1](https://www.anthropic.com/news/claude-opus-4-1)，这是对 Claude Opus 4 的增量更新，具有增强的能力和性能改进。\* 请在[模型概览](https://platform.claude.com/docs/zh-CN/about-claude/models)中了解更多信息。

*\*Opus 4.1 不允许同时指定 `temperature` 和 `top_p` 参数。请仅使用其中一个。*

### 2025 年 7 月 28 日

* 我们发布了 `text_editor_20250728`，这是一个更新的文本编辑器工具，修复了之前版本的一些问题，并添加了可选的 `max_characters` 参数，允许您在查看大文件时控制截断长度。

### 2025 年 7 月 24 日

* 我们提高了 Claude API 上 Claude Opus 4 的[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)，为您提供更多容量来使用 Claude 进行构建和扩展。对于拥有[使用层级 1-4 速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits#rate-limits)的客户，这些更改会立即应用到您的账户——无需任何操作。

### 2025 年 7 月 21 日

* 我们已停用 Claude 2.0、Claude 2.1 和 Claude Sonnet 3 模型。对这些模型的所有请求现在都将返回错误。请在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。

### 2025 年 7 月 17 日

* 我们提高了 Claude API 上 Claude Sonnet 4 的[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)，为您提供更多容量来使用 Claude 进行构建和扩展。对于拥有[使用层级 1-4 速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits#rate-limits)的客户，这些更改会立即应用到您的账户——无需任何操作。

### 2025 年 7 月 3 日

* 我们已推出 beta 版的搜索结果内容块，为 RAG 应用提供自然引用。工具现在可以返回带有正确来源归属的搜索结果，Claude 会在其响应中自动引用这些来源——达到与网页搜索相当的引用质量。这消除了在自定义知识库应用中使用文档变通方案的需要。请在[搜索结果](https://platform.claude.com/docs/zh-CN/build-with-claude/search-results)中了解更多信息。要启用此功能，请使用 beta 标头 `search-results-2025-06-09`。

### 2025 年 6 月 30 日

* 我们宣布弃用 Claude Opus 3 模型。请在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。

### 2025 年 6 月 23 日

* 拥有 Developer 角色的 Console 用户现在可以访问[成本](https://console.anthropic.com/settings/cost)页面。此前，Developer 角色允许访问[用量](https://console.anthropic.com/settings/usage)页面，但不能访问成本页面。

### 2025 年 6 月 11 日

* 我们已推出公开 beta 版的[细粒度工具流式传输](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/fine-grained-tool-streaming)，该功能使 Claude 能够在不进行缓冲 / JSON 验证的情况下流式传输工具使用参数。要启用细粒度工具流式传输，请使用 [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers) `fine-grained-tool-streaming-2025-05-14`。

### 2025 年 5 月 22 日

* 我们推出了 [Claude Opus 4 和 Claude Sonnet 4](https://www.anthropic.com/news/claude-4)，这是我们具有扩展思考能力的最新模型。请在[模型概览](https://platform.claude.com/docs/zh-CN/about-claude/models)中了解更多信息。
* Claude 4 模型中[扩展思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)的默认行为是返回 Claude 完整思考过程的摘要，完整的思考内容经过加密后在 `thinking` 块输出的 `signature` 字段中返回。
* 我们已推出公开 beta 版的[交错思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#interleaved-thinking)，该功能使 Claude 能够在工具调用之间进行思考。要启用交错思考，请使用 [beta 标头](https://platform.claude.com/docs/zh-CN/api/beta-headers) `interleaved-thinking-2025-05-14`。
* 我们已推出公开 beta 版的 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)，使您能够上传文件并在 Messages API 和代码执行工具中引用它们。
* 我们已推出公开 beta 版的[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)，该工具使 Claude 能够在安全的沙盒环境中执行 Python 代码。
* 我们已推出公开 beta 版的 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)，该功能允许您直接从 Messages API 连接到远程 MCP 服务器。
* 为了提高回答质量并减少工具错误，我们已将 Messages API 中 `top_p` [核采样](https://en.wikipedia.org/wiki/Top-p_sampling)参数的默认值从 0.999 更改为 0.99，适用于所有模型。要还原此更改，请将 `top_p` 设置为 0.999。 此外，启用扩展思考时，您现在可以将 `top_p` 设置为 0.95 到 1 之间的值。
* 我们的 [Go SDK](https://github.com/anthropics/anthropic-sdk-go) 已从 beta 版过渡到首个稳定版本。
* 我们在 Console 的[用量](https://console.anthropic.com/settings/usage)页面中加入了分钟级和小时级粒度，并在用量页面上提供 429 错误率。

### 2025 年 5 月 21 日

* 我们的 [Ruby SDK](https://github.com/anthropics/anthropic-sdk-ruby) 已从 beta 版过渡到首个稳定版本。

### 2025 年 5 月 7 日

* 我们在 API 中推出了网页搜索工具，允许 Claude 访问来自网络的最新信息。请在[网页搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)中了解更多信息。

### 2025 年 5 月 1 日

* 缓存控制现在必须直接在 `tool_result` 和 `document.source` 的父级 `content` 块中指定。为了向后兼容，如果在 `tool_result.content` 或 `document.source.content` 的最后一个块上检测到缓存控制，它将自动应用到父块。在 `tool_result.content` 和 `document.source.content` 内任何其他块上的缓存控制将导致验证错误。

### 2025 年 4 月 9 日

* 我们推出了 beta 版的 [Ruby SDK](https://github.com/anthropics/anthropic-sdk-ruby)。

### 2025 年 3 月 31 日

* 我们的 [Java SDK](https://github.com/anthropics/anthropic-sdk-java) 已从 beta 版过渡到首个稳定版本。
* 我们已将 [Go SDK](https://github.com/anthropics/anthropic-sdk-go) 从 alpha 版升级到 beta 版。

### 2025 年 2 月 27 日

* 我们在 Messages API 中为图像和 PDF 添加了 URL 源块。您现在可以直接通过 URL 引用图像和 PDF，而无需对其进行 base64 编码。请在[视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)和 [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)中了解更多信息。
* 我们在 Messages API 的 `tool_choice` 参数中添加了对 `none` 选项的支持，可阻止 Claude 调用任何工具。此外，在包含 `tool_use` 和 `tool_result` 块时，您不再需要提供任何 `tools`。
* 我们推出了与 OpenAI 兼容的 API 端点，允许您只需在现有 OpenAI 集成中更改 API 密钥、基础 URL 和模型名称即可测试 Claude 模型。此兼容层支持核心的聊天补全功能。请在 [OpenAI SDK 兼容性](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/openai-sdk)中了解更多信息。

### 2025 年 2 月 24 日

* 我们推出了 [Claude Sonnet 3.7](https://www.anthropic.com/news/claude-3-7-sonnet)，这是我们迄今为止最智能的模型。Claude Sonnet 3.7 可以生成近乎即时的响应，也可以逐步展示其扩展思考过程。一个模型，两种思考方式。请在[模型概览](https://platform.claude.com/docs/zh-CN/about-claude/models)中了解所有 Claude 模型的更多信息。

* 我们为 Claude Haiku 3.5 添加了视觉支持，使该模型能够分析和理解图像。

* 我们发布了令牌高效的工具使用实现，提高了使用 Claude 工具时的整体性能。请在[使用 Claude 进行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)中了解更多信息。

* 我们已将 [Console](https://console.anthropic.com/workbench) 中新提示的默认温度从 0 更改为 1，以与 API 中的默认温度保持一致。现有已保存的提示不受影响。

* 我们发布了工具的更新版本，将文本编辑和 bash 工具与计算机使用系统提示解耦：

  * `bash_20250124`：功能与之前版本相同，但独立于计算机使用。不需要 beta 标头。
  * `text_editor_20250124`：功能与之前版本相同，但独立于计算机使用。不需要 beta 标头。
  * `computer_20250124`：更新的计算机使用工具，具有新的命令选项，包括 "hold\_key"、"left\_mouse\_down"、"left\_mouse\_up"、"scroll"、"triple\_click" 和 "wait"。此工具需要 "computer-use-2025-01-24" anthropic-beta 标头。 请在[使用 Claude 进行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)中了解更多信息。

### 2025 年 2 月 10 日

* 我们在所有 API 响应中添加了 `anthropic-organization-id` 响应标头。此标头提供与请求中所用 API 密钥关联的组织 ID。

### 2025 年 1 月 31 日

* 我们已将 [Java SDK](https://github.com/anthropics/anthropic-sdk-java) 从 alpha 版升级到 beta 版。

### 2025 年 1 月 23 日

* 我们在 API 中推出了引用功能，允许 Claude 为信息提供来源归属。请在[引用](https://platform.claude.com/docs/zh-CN/build-with-claude/citations)中了解更多信息。
* 我们在 Messages API 中添加了对纯文本文档和自定义内容文档的支持。

### 2025 年 1 月 21 日

* 我们宣布弃用 Claude 2、Claude 2.1 和 Claude Sonnet 3 模型。请在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。

### 2025 年 1 月 15 日

* 我们更新了[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)，使其更易于使用。现在，当您设置缓存断点时，我们会自动从您之前缓存的最长前缀中读取。
* 您现在可以在使用工具时预填 Claude 的回复内容。

### 2025 年 1 月 10 日

* 我们优化了对 [Message Batches API 中提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing#using-prompt-caching-with-message-batches)的支持，以提高缓存命中率。

### 2024 年 12 月 19 日

* 我们在 Message Batches API 中添加了对[删除端点](https://platform.claude.com/docs/zh-CN/api/deleting-message-batches)的支持。

### 2024 年 12 月 17 日

以下功能现已在 Claude API 中提供，无需 beta 标头：

* [Models API](https://platform.claude.com/docs/zh-CN/api/models/list)：查询可用模型、验证模型 ID，并将[模型别名](https://platform.claude.com/docs/zh-CN/models/overview)解析为其规范模型 ID。
* [Message Batches API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)：以标准 API 成本的 50% 异步处理大批量消息。
* [令牌计数 API](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)：在将消息发送给 Claude 之前计算其令牌数。
* [提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)：通过缓存和重用提示内容，将成本降低最多 90%，延迟降低最多 80%。
* [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)：处理 PDF 以分析文档中的文本和视觉内容。

我们还发布了新的官方 SDK：

* [Java SDK](https://github.com/anthropics/anthropic-sdk-java)（alpha）
* [Go SDK](https://github.com/anthropics/anthropic-sdk-go)（alpha）

### 2024 年 12 月 4 日

* 我们在[开发者控制台](https://console.anthropic.com)的[用量](https://console.anthropic.com/settings/usage)和[成本](https://console.anthropic.com/settings/cost)页面中添加了按 API 密钥分组的功能。
* 我们在[开发者控制台](https://console.anthropic.com)的 [API 密钥](https://console.anthropic.com/settings/keys)页面中添加了两个新列**上次使用时间**和**成本**，以及按任意列排序的功能。

### 2024 年 11 月 21 日

* 我们发布了 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)，允许用户以编程方式管理其组织的资源。

### 2024 年 11 月 20 日

* 我们更新了 Messages API 的速率限制。我们已将每分钟令牌数速率限制替换为新的每分钟输入令牌数和输出令牌数速率限制。请在[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)中了解更多信息。
* 我们在 [Workbench](https://console.anthropic.com/workbench) 中添加了对[工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)的支持。

### 2024 年 11 月 13 日

* 我们为所有 Claude Sonnet 3.5 模型添加了 PDF 支持。请在 [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)中了解更多信息。

### 2024 年 11 月 6 日

* 我们已停用 Claude 1 和 Instant 模型。请在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。

### 2024 年 11 月 4 日

* [Claude Haiku 3.5](https://www.anthropic.com/claude/haiku) 现已作为纯文本模型在 Claude API 上提供。

### 2024 年 11 月 1 日

* 我们添加了可与新版 Claude Sonnet 3.5 配合使用的 PDF 支持。请在 [PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)中了解更多信息。
* 我们还添加了令牌计数功能，允许您在将消息发送给 Claude 之前确定消息中的令牌总数。请在[令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)中了解更多信息。

### 2024 年 10 月 22 日

* 我们在 API 中添加了 Anthropic 定义的计算机使用工具，可与新版 Claude Sonnet 3.5 配合使用。请在[计算机使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)中了解更多信息。
* Claude Sonnet 3.5，我们迄今为止最智能的模型，刚刚获得升级，现已在 Claude API 上提供。请在 [Claude Sonnet 文档](https://www.anthropic.com/claude/sonnet)中了解更多信息。

### 2024 年 10 月 8 日

* Message Batches API 现已推出 beta 版。在 Claude API 中以低 50% 的成本异步处理大批量查询。请在[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)中了解更多信息。
* 我们放宽了 Messages API 中 `user`/`assistant` 轮次顺序的限制。连续的 `user`/`assistant` 消息将被合并为单条消息而不会报错，并且我们不再要求第一条输入消息必须是 `user` 消息。
* 我们已弃用 Build 和 Scale 计划，转而采用标准功能套件（以前称为 Build），以及可通过销售渠道获得的附加功能。请在我们的 [API 定价信息](https://claude.com/platform/api)中了解更多信息。

### 2024 年 10 月 3 日

* 我们在 API 中添加了禁用并行工具使用的功能。在 `tool_choice` 字段中设置 `disable_parallel_tool_use: true` 以确保 Claude 最多使用一个工具。请在[并行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/parallel-tool-use)中了解更多信息。

### 2024 年 9 月 10 日

* 我们在[开发者控制台](https://console.anthropic.com)中添加了工作区。工作区允许您设置自定义支出或速率限制、对 API 密钥进行分组、按项目跟踪使用情况，以及通过用户角色控制访问权限。请在我们的[博客文章](https://www.anthropic.com/news/workspaces)中了解更多信息。

### 2024 年 9 月 4 日

* 我们宣布弃用 Claude 1 模型。请在[模型弃用](https://platform.claude.com/docs/zh-CN/about-claude/model-deprecations)中了解更多信息。

### 2024 年 8 月 22 日

* 我们通过在 API 响应中返回 CORS 标头，添加了对在浏览器中使用 SDK 的支持。在 SDK 实例化时设置 `dangerouslyAllowBrowser: true` 以启用此功能。

### 2024 年 8 月 19 日

* Claude Sonnet 3.5 上的 8,192 令牌输出已结束 beta 阶段，不再需要 `max-tokens-3-5-sonnet-2024-07-15` 标头。

### 2024 年 8 月 14 日

* [提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)现已作为 beta 功能在 Claude API 中提供。缓存并重用提示，可将延迟降低最多 80%，成本降低最多 90%。

### 2024 年 7 月 15 日

* 使用新的 `anthropic-beta: max-tokens-3-5-sonnet-2024-07-15` 标头，从 Claude Sonnet 3.5 生成长度最多为 8,192 个令牌的输出。

### 2024 年 7 月 9 日

* 在[开发者控制台](https://console.anthropic.com)中使用 Claude 为您的提示自动生成测试用例。
* 在[开发者控制台](https://console.anthropic.com)新的输出比较模式中并排比较不同提示的输出。

### 2024 年 6 月 27 日

* 在[开发者控制台](https://console.anthropic.com)新的[用量](https://console.anthropic.com/settings/usage)和[成本](https://console.anthropic.com/settings/cost)选项卡中，按美元金额、令牌数和 API 密钥细分查看 API 使用情况和账单。
* 在[开发者控制台](https://console.anthropic.com)新的[速率限制](https://console.anthropic.com/settings/limits)选项卡中查看您当前的 API 速率限制。

### 2024 年 6 月 20 日

* [Claude Sonnet 3.5](https://www.anthropic.com/news/claude-3-5-sonnet)，我们迄今为止最智能的模型，现已在 Claude API、Amazon Bedrock 和 Vertex AI 上全面提供。

### 2024 年 5 月 30 日

* [工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)已在 Claude API、Amazon Bedrock 和 Vertex AI 上结束 beta 阶段，无需 beta 标头。

### 2024 年 5 月 10 日

* 我们的提示生成器工具现已在[开发者控制台](https://console.anthropic.com)中提供。提示生成器可以轻松引导 Claude 生成针对您特定任务量身定制的高质量提示。请在我们的[博客文章](https://www.anthropic.com/news/prompt-generator)中了解更多信息。
