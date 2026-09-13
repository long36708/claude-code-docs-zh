---
title: MCP 隧道故障排查
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/troubleshooting
description: 诊断隧道栈中的连接、TLS、IP 验证和 OAuth 路由问题。
---

<Note>
  MCP 隧道目前处于研究预览阶段。[申请访问权限](https://claude.com/form/claude-managed-agents)以试用。
</Note>

通过隧道的请求可能在三个层级之一失败；请按顺序诊断：到[隧道边缘（tunnel edge）](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)的出站连接、从 Anthropic 到您的[代理（proxy）](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)的[内层 TLS（inner TLS）](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)，然后是通往[上游 MCP 服务器（upstream MCP server）](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)的路由和 IP 验证。

## 快速参考

| 症状                                                                                                                                                         | 原因                                                 | 修复方法                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| 隧道未出现在智能体的 **+ MCP Server** 选择器中                                                                                                                           | 选择器仅列出会话所在工作区中至少拥有一个有效证书的隧道。                       | 注册一个 CA 证书，或在创建该隧道的工作区中打开会话。                                                                                                                          |
| 调用方看到 HTTP 500；[cloudflared](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components) 日志显示 `No ingress rules were defined` | cloudflared 没有本地目标。                                | 为 cloudflared 服务添加 `--url http://localhost:8080` 和 `network_mode: "service:mcp-proxy"`。                                                               |
| 代理日志显示 `no route for host`                                                                                                                                 | `tunnel_domain` 与分配的域名不匹配，或编辑了 `config.yaml` 但未重启。 | 将 `tunnel_domain` 设置为隧道详情页上显示的确切域名，然后重启代理（`docker compose restart mcp-proxy`）。                                                                        |
| 代理日志显示 `IP validation failed: <ip> is not a private address`                                                                                               | 上游 MCP 服务器解析到 RFC1918 范围之外。                        | 请参阅[上游 IP 验证](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/troubleshooting#upstream-ip-validation)。                            |
| 代理退出并显示 `cannot unmarshal !!seq into map[string]string`                                                                                                    | `routes` 是一个 YAML 列表。                              | 使用 `routes: { name: http://host:port }`。                                                                                                              |
| 代理退出并显示 `open /data/tls.key: permission denied`                                                                                                            | 密钥权限为 `0600`；代理容器以非 root 身份运行。                     | `chmod 644 data/tls.key`。                                                                                                                             |
| `curl https://<proxy>:8080` 失败并显示 `wrong version number`                                                                                                   | 这是预期行为；监听器是明文 WebSocket。TLS 在 WS 流内部进行。            | 请改为通过 [Managed Agent 或 Messages API](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview#use-the-tunneled-mcp-servers) 进行验证。 |

以下各节涵盖需要不止一行修复的故障。

## OAuth 在源 IP 允许列表后失败

当您的授权服务器的源 IP 允许列表阻止 Anthropic 的后端访问 `/token`、`/register` 和发现端点时，OAuth 流程会失败。如果您不想将 Anthropic 的出口地址范围加入允许列表，可以将后端到后端的 OAuth 调用通过隧道路由，同时将面向浏览器的 `/authorize` 端点保留在您现有的公共主机名上。

<Steps>
  <Step title="为授权服务器添加代理路由">
    ```yaml
    routes:
      mcp: http://your-mcp-server:8080
      auth: http://your-auth-server:8080
    ```

    编辑 `routes` 后重启代理（`docker compose restart mcp-proxy`，或 `helm upgrade`）。
  </Step>

  <Step title="提供拆分端点的发现元数据">
    您的授权服务器的 `/.well-known/oauth-authorization-server` 响应应将 `authorization_endpoint` 指向您现有的已加入允许列表的主机名，并将其他所有端点指向隧道：

    ```json
    {
      "issuer": "https://auth.<tunnel-domain>",
      "authorization_endpoint": "https://<your-allowlisted-host>/authorize",
      "token_endpoint": "https://auth.<tunnel-domain>/token",
      "registration_endpoint": "https://auth.<tunnel-domain>/register",
      "code_challenge_methods_supported": ["S256"]
    }
    ```
  </Step>

  <Step title="将 MCP 服务器指向隧道颁发者">
    您的 MCP 服务器的 `/.well-known/oauth-protected-resource` 响应应将隧道主机名引用为其授权服务器：

    ```json
    {
      "resource": "https://mcp.<tunnel-domain>",
      "authorization_servers": ["https://auth.<tunnel-domain>"]
    }
    ```
  </Step>
</Steps>

采用此配置后，用户的浏览器会访问您现有主机名上的 `/authorize`（您的允许列表已经允许），而 Anthropic 的后端则通过隧道访问 `/token`、`/register` 和发现文档。

## 设置组件身份验证失败

[设置组件（setup component）](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)（Helm Job 或 Compose `setup` 服务）通过您的联合规则交换 OIDC JWT 来向 Tunnels API 进行身份验证。当交换失败时，请参阅 Workload Identity Federation 参考中的[排查失败的交换](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#troubleshoot-a-failed-exchange)；失败模式（subject、audience、issuer、JWKS、有效期）是相同的。

隧道特有的原因：

* chart 的默认 audience 是 `api.anthropic.com`（不带协议前缀）。如果您规则的 audience 是 `https://api.anthropic.com`，请将 `api.wif.audience` 设置为与之匹配。
* 交换成功后 Tunnels API 返回 `403`，意味着规则的作用域不包含 `workspace:manage_tunnels`，或者规则的服务账户不是隧道所在工作区的成员。请设置作用域并将服务账户添加到工作区。

在 Helm 上，设置组件作为 pre-install hook Job 运行。失败时，该 Job 会被保留以供检查（`kubectl logs job/mcp-tunnel-setup -n mcp-tunnel`）。Helm 不管理 hook 资源，因此在重试之前请将其删除：

```bash
helm uninstall mcp-tunnel -n mcp-tunnel
kubectl -n mcp-tunnel delete job mcp-tunnel-setup
```

## 隧道无法连接

首先检查 cloudflared 日志。常见原因：

* `TUNNEL_TOKEN` 缺失、已过期或复制有误。
* 防火墙阻止了到隧道边缘的 7844 端口出站 TCP/UDP 流量。

cloudflared 还可能记录有关 UDP 接收缓冲区大小的警告；这是 QUIC 调优提示，而非错误。

## 证书错误

当 Anthropic 在内层 TLS 期间拒绝代理的证书时，代理会记录 `tls handshake failed`。请验证：

* 服务器证书尚未过期。
* 证书的 Subject Alternative Name 与 `*.<tunnel-domain>` 匹配。
* 签名 CA 已针对此隧道在 Anthropic 注册。

有关完整的验证规则，请参阅[证书要求](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/reference#certificate-requirements)。

## 上游 IP 验证

为了防范 SSRF，代理默认只连接 RFC1918 私有地址范围（`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`）内的地址。代理到上游的连接仅支持 IPv4。（[网络要求](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview#network-requirements)中的 cloudflared 到边缘的出口范围是另一个跳段。）

如果代理记录 `IP validation failed: <ip> is not a private address`，说明上游主机名解析到了该范围之外。在 Kubernetes 上，某些托管发行版会在 RFC1918 之外分配 Service CIDR；如果 `kubectl get svc kubernetes -n default -o jsonpath='{.spec.clusterIP}'` 返回的地址在私有范围之外，请查找您集群的 Service CIDR 并将其添加进去。

如果该地址是合法的，请将覆盖它的最窄 CIDR 添加到 `upstream.allowed_ips`。设置 `allowed_ips` 会**替换** RFC1918 默认值而不是扩展它，因此请包含您其他上游 MCP 服务器使用的私有范围：

```yaml config/mcp-proxy.yaml
upstream:
  allowed_ips:
    - 10.0.0.0/8
    - 172.16.0.0/12
    - 192.168.0.0/16
    - 127.0.0.0/8       # loopback, for local testing only
```

<Warning>
  在本地测试之外请避免使用 `0.0.0.0/0`；它会完全禁用 SSRF 防护。
</Warning>
