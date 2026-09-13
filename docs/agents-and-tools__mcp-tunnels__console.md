---
title: 在 Console 中管理隧道
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console
description: 在 Claude Console 中创建隧道、注册 CA 证书、获取隧道令牌，并将通过隧道连接的 MCP 服务器附加到智能体。
---

<Note>
  MCP 隧道目前处于研究预览阶段。[申请访问权限](https://claude.com/form/claude-managed-agents)以试用。
</Note>

本页介绍 MCP 隧道部署中 Console 一侧的内容：创建隧道、注册您的 CA 证书、获取隧道令牌，以及将[上游 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)附加到智能体。[使用 Helm 部署 MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-helm)和[使用 Docker Compose 部署 MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose)介绍了如何在您的网络内部运行[隧道栈](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)。

## 前提条件

* **一个或多个 MCP 服务器**，运行在您的私有网络中。隧道将流量路由到这些服务器；它并不托管它们。请参阅[远程 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/remote-mcp-servers)了解您可以部署的示例。

* **具有"管理隧道"权限的 Console 角色**，以便您可以创建和归档隧道、轮换令牌以及管理证书。组织管理员和所有者默认拥有该权限；自定义角色和按账户授予的权限也可以包含它。没有该权限的角色对 **MCP tunnels** 页面和隧道详情仅有只读访问权限。

* \*\*一种让您的栈向 Tunnels API 进行身份验证的方式。\*\*请选择其一：

  * \*\*[编程访问](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#credential-provisioning)（推荐）。\*\*在创建隧道期间设置 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)（工作负载身份联合），使您的栈能够从您的身份提供商签发短期 API 令牌、获取隧道令牌，并自动生成和注册 CA 证书。需要管理联合规则的权限、一个已注册的 OIDC 颁发者，以及一条具有 `workspace:manage_tunnels` 作用域的联合规则。
  * \*\*[手动](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#credential-provisioning)。\*\*跳过编程访问。创建隧道后，[获取隧道令牌](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#get-the-connection-details)，自行生成并[注册 CA 证书](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#add-a-ca-certificate)，然后将令牌和您的服务器证书作为机密提供给您的隧道栈。

## 创建隧道

<Steps>
  <Step title="打开 MCP tunnels 页面">
    在 Console 侧边栏中，前往 **Manage > MCP tunnels**。隧道的作用域为工作区；新隧道属于 Console 中当前选定的工作区，因此如果您希望将其创建在其他位置，请先切换工作区。
  </Step>

  <Step title="为隧道命名">
    点击 **New tunnel**，并在 **Create tunnel** 对话框中输入名称。名称为必填项，用于在列表、详情页面以及智能体 MCP 服务器选择器中标识该隧道。系统会自动分配一个形如 `abcd1234.tunnel.anthropic.com` 的域名。
  </Step>

  <Step title="可选：设置编程访问">
    如果您的角色可以管理联合规则，则会出现一个 **Set up programmatic access** 开关（默认关闭）。如果不能，Console 会在该位置显示一条通知，您的隧道栈将改用手动流程。无论哪种方式，创建流程的其余部分都相同。

    编程访问依赖于 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)；如果您不熟悉联合颁发者、规则和服务账户，请先阅读该页面。要打开该开关，您需要：

    1. **一个已注册的 OIDC 颁发者**，对应您的栈所出示令牌的身份提供商（例如 Kubernetes 集群、AWS IAM、Google Cloud 或 GitHub Actions）。如果您的组织还没有，请在 **Settings > Workload identity > Issuers** 下注册一个。
    2. \*\*一条具有 `workspace:manage_tunnels` 作用域的联合规则。\*\*打开开关后会显示一个 **Federation rule** 选择器。选择一条具有该作用域的现有规则，或点击 **Create federation rule** 以内联方式创建一条。
    3. \*\*将该规则的服务账户添加到此工作区。\*\*Tunnels API 根据服务账户的工作区成员资格进行授权。如果您在组织默认工作区以外的工作区中创建隧道，请在 **Settings > Workspaces** 下添加该服务账户，并在部署时传入工作区 ID（Helm 使用 `api.wif.workspaceId`，Compose 使用 `ANTHROPIC_WORKSPACE_ID`）。

    完全支持跳过此步骤；两份部署指南都有一个 **Without programmatic access** 选项卡。
  </Step>

  <Step title="创建隧道">
    点击 **Create tunnel**。Console 会配置该隧道并打开详情页面。
  </Step>

  <Step title="记录部署标识符">
    两种部署路径都需要：

    * **隧道 ID**（`tnl_...`），显示在隧道详情页面上。
    * **隧道域名**（`abcd1234.tunnel.anthropic.com`），显示在隧道详情页面上。用作代理的 `tunnel_domain`，并用于服务器证书的 SAN 中。

    您还需要什么取决于[凭据配置模式](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#credential-provisioning)：

    | 使用编程访问                                                                                           | 不使用编程访问                                                                                                                                                           |
    | ------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | 您所选规则的**联合规则 ID**（`fdrl_...`）。该规则是组织级别的，不存储在隧道上；可在 **Settings > Workload identity > Rules** 下找到。 | **隧道令牌**，通过详情页面上 **Token** 旁边的眼睛图标显示。请将其视为机密。请参阅[获取连接详情](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#get-the-connection-details)。 |
    | **组织 ID**（一个 UUID），显示在 **Settings > Organization** 下。                                            | 一个由您生成并[在隧道上注册](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#add-a-ca-certificate)的 **CA 证书**。                                     |

    使用编程访问时，您的栈通过 Tunnels API 获取隧道令牌，在本地生成 CA 和服务器证书（私钥永远不会离开您的环境），并且仅向 Anthropic 注册 CA 的公共证书。您仍然负责保护私钥的安全，并在服务器证书过期之前对其进行续期。
  </Step>
</Steps>

您的组织最多可以拥有 10 个活动隧道。创建隧道并不会建立任何连接；只有当您的栈使用隧道令牌拨入并且注册了 CA 证书后，连接才会建立。

## 获取连接详情

打开隧道。详情页面显示一个包含域名和令牌的 **Connection** 部分，以及一个 **Certificates** 部分。

| 字段         | 描述                                                                                       |
| ---------- | ---------------------------------------------------------------------------------------- |
| **Domain** | 复制分配的 `abcd1234.tunnel.anthropic.com` 值。您代理的路由是此域名的子域名。                                  |
| **Token**  | 点击眼睛图标（**Show token**）获取隧道令牌，然后使用复制图标将其复制到您隧道栈的机密存储中。点击 **Rotate token** 可使当前令牌失效并签发新令牌。 |

<Note>
  每次显示和轮换都会记录在您组织的 [Compliance API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 活动日志中。轮换不会切断已建立的 cloudflared 连接，因此您可以先轮换，再使用新值重新部署，并让旧连接自然排空。
</Note>

## 添加 CA 证书

Anthropic 根据您在隧道上注册的 CA 证书来验证到您[代理](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)的[内层 TLS](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)。没有活动证书的隧道无法接受连接，并且在注册证书之前不会出现在智能体 MCP 服务器选择器中。

<Steps>
  <Step title="找到 Certificates 部分">
    在隧道的详情页面上，滚动到 **Certificates** 部分并点击 **Add certificate**。
  </Step>

  <Step title="提供证书">
    点击 **Choose file** 选择一个 `.pem`、`.crt` 或 `.cer` 文件，将文件拖到文本区域，或直接粘贴 PEM 块。该模态框会拒绝私钥材料以及不是 `-----BEGIN CERTIFICATE-----` 块的内容。文件大小必须不超过 8 kB。
  </Step>

  <Step title="添加证书">
    点击 **Add certificate**。指纹和到期时间会出现在证书列表中，并且该部分标题上的槽位计数会递增。
  </Step>
</Steps>

一个隧道最多可持有两个活动证书，以便您可以在不停机的情况下进行轮换：在旧证书旁注册新证书，使用新密钥对重新部署您的代理，确认流量正常流动，然后点击旧证书所在行的 **Revoke**。已吊销的证书仍会在列表中显示，并带有 **Revoked** 标记。

## 部署隧道栈

隧道已存在于 Console 中，但在隧道栈于您的网络内部运行并使用隧道令牌拨入之前，不会有任何流量流动。请按照以下部署指南之一进行操作：

<CardGroup cols={2}>
  <Card title="使用 Docker Compose 部署" icon="cube" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose">
    在单台主机上运行隧道栈。涵盖编程访问和手动两种流程。
  </Card>

  <Card title="使用 Helm 部署" icon="stack" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-helm">
    在 Kubernetes 集群上运行隧道栈。涵盖编程访问和手动两种流程。
  </Card>
</CardGroup>

## 在智能体中使用隧道

一旦您的栈正在运行并配置了一个或多个 MCP 服务器，即可将上游 MCP 服务器附加到 Managed Agent 会话。如需改为从 Messages API 调用相同的服务器，请参阅[使用通过隧道连接的 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview#use-the-tunneled-mcp-servers)。

<Note>
  选择器仅显示至少拥有一个活动证书的隧道。在 **MCP tunnels** 列表中仍显示 **Needs certificate** 的隧道不会出现在下拉列表中；请先注册 CA 证书。选择器的作用域同样为工作区：它列出与会话位于同一工作区的隧道，而不列出其他工作区的隧道。
</Note>

<Steps>
  <Step title="打开 New session 模态框">
    前往 **Managed Agents > Sessions** 并点击 **New session**。
  </Step>

  <Step title="定义内联智能体">
    在智能体选择器中，选择 **Create new agent**，以便您可以直接编辑 MCP 服务器列表。
  </Step>

  <Step title="添加 MCP 服务器">
    点击 **+ MCP Server** 并打开下拉列表。在当前工作区中创建的隧道显示在列表顶部，位于公共连接器目录之上。选择位于您要访问的服务器前端的隧道。
  </Step>

  <Step title="提供路由信息">
    卡片显示两个可选字段：**Subdomain**（作为隧道域名的前缀）和 **Path**（附加在其后）。根据您代理的路由配置方式，填写其中一个或两个。**Resolves to** 行显示智能体所连接的完整 MCP 服务器 URL。
  </Step>
</Steps>

<Note>
  隧道承载流量；它不会向上游 MCP 服务器进行身份验证。请像对任何其他 MCP 服务器一样，在该 MCP 服务器上配置 OAuth 或 bearer 身份验证。
</Note>

## 归档隧道

归档会立即使隧道停止接受连接，并且是永久性的。

在 **MCP tunnels** 列表中，打开该隧道的行菜单并选择 **Archive**。当您按 **Archived** 或 **All** 筛选列表时，已归档的隧道仍然可见。

## 后续步骤

<CardGroup cols={2}>
  <Card title="使用 Helm 部署" icon="stack" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-helm">
    使用 Anthropic Helm chart 在 Kubernetes 集群上安装。
  </Card>

  <Card title="安全" icon="lock" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/security">
    加固指南、凭据轮换和入侵响应。
  </Card>
</CardGroup>
