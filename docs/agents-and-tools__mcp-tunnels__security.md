---
title: MCP 隧道安全
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/security
description: MCP 隧道部署的加固指南、凭证轮换、入侵响应和拆除。
---

<Note>
  MCP 隧道目前处于研究预览阶段。[申请访问权限](https://claude.com/form/claude-managed-agents)以试用。
</Note>

隧道架构提供了强大的默认设置（仅出站连接、端到端加密和 IP 验证），但您的[隧道栈](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)的整体安全性还取决于您如何配置和运维它。本页介绍推荐的加固措施、入侵响应以及如何停用隧道。

## 最佳实践

* **在每个 MCP 服务器上要求 OAuth。** 按照 [MCP 授权规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)中的说明，将每个[上游 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)配置为要求 OAuth。OAuth 在隧道的传输层身份验证之上提供纵深防御，并在数据层实现用户级授权。
* **为您的组织启用 SSO。** 隧道、联合规则和服务账户在 Claude Console 中管理。SSO 会对能够更改这些设置的管理员强制执行您的身份提供商的会话控制。
* **限制 `upstream.allowed_ips`。** 使用能够覆盖您的 MCP 服务器的最小 CIDR 范围。这是[代理](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)的主要 SSRF 防御手段。
* **监控日志。** 对来自隧道栈的警告、错误和异常流量模式设置告警。
* **轮换凭证。** 定期轮换服务器证书和隧道令牌，如果怀疑遭到泄露，请立即轮换。
* **保持镜像更新。** 跟踪新的代理版本，并通过 SHA-256 摘要固定镜像。
* **限制网络可达范围。** 代理和 [cloudflared](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components) 应只能访问[网络要求](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview#network-requirements)中列出的目标地址。使用 NetworkPolicy（Kubernetes）或主机防火墙规则（Compose）。
* **限制 MCP 服务器范围。** 每个服务器应仅暴露其用途所需的工具和数据。
* **保护静态凭证。** 对私钥和隧道令牌应用您组织的密钥管理实践。

## 响应疑似入侵

如果您认为您的隧道令牌、TLS 密钥或代理主机已遭泄露：

<Steps>
  <Step title="停止隧道栈">
    <Tabs>
      <Tab title="Helm">
        ```bash
        helm uninstall mcp-tunnel -n mcp-tunnel
        ```
      </Tab>

      <Tab title="Docker Compose">
        ```bash
        docker compose down --timeout 0
        ```
      </Tab>
    </Tabs>
  </Step>

  <Step title="分离上游 MCP 服务器">
    从所有使用上游 MCP 服务器的 Managed Agent 会话中移除这些服务器，并停止在 Messages API 请求的 `mcp_servers` 块中传递它们的 URL。
  </Step>

  <Step title="归档隧道">
    归档会使隧道令牌失效并分离域名。在 Console 中，从 **MCP tunnels** 列表中[归档隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#archive-a-tunnel)。如需改为通过 API 归档，请参阅[归档隧道](https://platform.claude.com/docs/zh-CN/api/beta/tunnels/archive)。
  </Step>

  <Step title="联系 Anthropic">
    向 Anthropic 支持团队报告疑似泄露事件。
  </Step>

  <Step title="轮换下游凭证">
    重新配置一个新的隧道，并轮换受影响的 MCP 服务器签发的所有 OAuth 令牌。
  </Step>

  <Step title="在恢复服务前审查日志">
    在新隧道上线之前，检查疑似泄露时间段内的代理、cloudflared 和 MCP 服务器日志。
  </Step>
</Steps>

## 拆除隧道

按照以下步骤停用隧道并移除所有已存储的凭证。

<Steps>
  <Step title="停止隧道栈">
    <Tabs>
      <Tab title="Helm">
        ```bash
        helm uninstall mcp-tunnel -n mcp-tunnel
        ```
      </Tab>

      <Tab title="Docker Compose">
        ```bash
        docker compose down
        ```
      </Tab>
    </Tabs>
  </Step>

  <Step title="归档隧道">
    在 Console 中，从 **MCP tunnels** 列表中[归档隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#archive-a-tunnel)。
  </Step>

  <Step title="移除已存储的凭证">
    <Tabs>
      <Tab title="Helm">
        使用编程访问时，setup 组件创建了一个以 release 命名的 Secret。未使用编程访问时，您自己创建了 `mcp-tunnel-token` 和 `mcp-tunnel-cert`。删除适用的项：

        ```bash
        kubectl -n mcp-tunnel delete secret \
          mcp-tunnel mcp-tunnel-token mcp-tunnel-cert \
          --ignore-not-found
        ```
      </Tab>

      <Tab title="Docker Compose">
        私钥和证书位于 `data/` 中。隧道令牌位于 `data/tunnel-token`（编程流程）或您的 shell 环境中（手动流程）。`config/` 目录和 `docker-compose.yaml` 不包含任何密钥；如果您计划重新配置，请保留它们，否则也可以一并移除。

        ```bash
        sudo rm -rf data
        ```
      </Tab>
    </Tabs>
  </Step>
</Steps>
