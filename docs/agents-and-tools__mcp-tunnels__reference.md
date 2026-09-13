---
title: MCP 隧道参考
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/reference
description: 代理配置字段、Tunnels REST API、证书要求以及设置组件。
---

<Note>
  MCP 隧道目前处于研究预览阶段。[申请访问权限](https://claude.com/form/claude-managed-agents)以试用。
</Note>

## 代理配置

[代理](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)从 `/etc/mcp-gateway/config.yaml`（Compose）或渲染后的 ConfigMap（Helm，由 `gateway.config.*` 填充）读取其配置。

| 字段                                | 描述                                                                                                                            | 默认值                   |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| `listen_addr`                     | 要监听的地址和端口。                                                                                                                    | 必填                    |
| `log_level`                       | 日志详细级别：`debug`、`info`、`warn` 或 `error`。                                                                                       | `info`                |
| `shutdown_timeout`                | 优雅关闭期间等待进行中请求的时长。                                                                                                             | `30s`                 |
| `tunnel_domain`                   | 分配给隧道的基础域名。设置后，路由查找会从传入的主机名中去除此后缀，因此 `routes` 的键可以是裸子域名（`wiki`）。为空时，`routes` 的键必须是精确的完整主机名。                                   | 当 `routes` 的键为裸子域名时必填 |
| `tls.cert_file`                   | 服务器 TLS 证书的路径。                                                                                                                | 必填                    |
| `tls.key_file`                    | 服务器 TLS 私钥的路径。                                                                                                                | 必填                    |
| `routes`                          | 子域名或完整主机名到上游 URL 的映射。请参阅[路由匹配](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/reference#route-matching)。 | 必填                    |
| `upstream.allowed_ips`            | 允许代理连接的 IPv4 CIDR 范围或单个地址。与 `disable_ip_validation` 互斥。                                                                       | RFC1918 私有地址范围        |
| `upstream.disable_ip_validation`  | 完全禁用上游 IP 验证。与 `allowed_ips` 互斥。                                                                                              | `false`               |
| `upstream.tls.ca_file`            | 用于验证上游 TLS 的 CA 证书包。                                                                                                          | 无                     |
| `upstream.tls.include_system_cas` | 对上游 TLS 同时信任系统 CA 证书包。                                                                                                        | `false`               |

对于 `https://` 上游路由，请至少设置 `upstream.tls.ca_file` 或 `upstream.tls.include_system_cas` 之一；否则代理将没有用于验证上游证书的信任锚。

### 路由匹配

`routes` 是一个扁平的字符串映射（`map[string]string`），而不是列表。代理首先按精确匹配查找传入的主机名，然后去除 `tunnel_domain` 后缀并匹配剩余的子域名。匹配仅考虑主机名；请求路径和查询字符串会原样转发到[上游 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)。

每个上游值必须严格为 `scheme://host:port` 格式。端口是必需的。包含路径会在配置加载时被拒绝，并报错 `invalid upstream (must be scheme://host:port)`。

## Tunnels API

Tunnels REST API 位于 `/v1/tunnels`，支持创建、列出和归档隧道，注册 CA 证书，以及显示或轮换隧道令牌。请参阅 [Tunnels API 参考](https://platform.claude.com/docs/zh-CN/api/beta/tunnels/list)了解所有端点、请求和响应模式以及示例。

<Note>
  之前位于 `/v1/organizations/tunnels` 的 Admin API 接口（beta 标头 `mcp-tunnels-2026-05-19`，作用域 `org:manage_tunnels`）在迁移窗口期内 继续可用，并仍在 [Admin API 参考](https://platform.claude.com/docs/zh-CN/api/admin/mcp_tunnels)中 附带弃用通知进行记录。要进行迁移，请将路径更新为 `/v1/tunnels`，将 beta 标头更新为 `mcp-tunnels-2026-06-22`，并将您的 WIF 令牌作用域更新为 `workspace:manage_tunnels`。
</Note>

<Warning>
  所有 MCP 隧道端点都需要通过 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)（工作负载身份联合）获取的、具有 `workspace:manage_tunnels` 作用域的 bearer 令牌。不接受 Admin API 密钥。
</Warning>

每个请求必需的标头：

| 标头                  | 值                               |
| ------------------- | ------------------------------- |
| `Authorization`     | `Bearer <token>`（经 WIF 交换得到的令牌） |
| `anthropic-version` | `2023-06-01`                    |
| `anthropic-beta`    | `mcp-tunnels-2026-06-22`        |

## 证书要求

[设置组件](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)会自动生成符合要求的证书。以下要求仅适用于您通过自己的 PKI 签发证书的情况。

### CA 证书

使用 `POST /v1/tunnels/{tunnel_id}/certificates` 上传。一个隧道同时最多可持有两个有效的 CA 证书，从而支持零停机轮换。

* PEM 编码，单个证书，最大 8 kB。
* 存在 `BasicConstraints` 扩展且 `CA:TRUE`，并标记为关键（critical）。
* 存在 `SubjectKeyIdentifier` 扩展。
* `KeyUsage` 包含 `keyCertSign`。
* 处于有效期内。
* RSA 2048 位或更大，或 ECDSA P-256 或更大，并使用 SHA-256 或更强的签名。

### 服务器证书

由代理在[内层 TLS](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components) 期间出示。

* 由已注册的 CA 直接签名（无中间证书）。
* 存在 `AuthorityKeyIdentifier` 扩展，且与 CA 的 `SubjectKeyIdentifier` 匹配。
* Subject Alternative Name（主题备用名称）包含与 `<route>.<tunnel-domain>` 匹配的 DNS 名称。通配符 `*.<tunnel-domain>` 可覆盖所有路由。
* 如果存在 `ExtendedKeyUsage` 扩展，则其包含 `serverAuth`。
* 处于有效期内。
* RSA 2048 位或更大，或 ECDSA P-256 或更大，并使用 SHA-256 或更强的签名。

设置组件会生成一个有效期为五年的 ECDSA P-256 CA，以及一个带通配符 SAN、有效期为 90 天的 RSA 4096 位服务器证书。

## 设置组件

设置组件以 `setup` 二进制文件的形式随 `mcp-proxy` 镜像一同提供。使用 `docker compose run --rm setup <subcommand>`（Compose）运行它，或依赖 chart 的钩子和 CronJob（Helm）。

### `setup init`

附加到现有隧道（或在未提供隧道 ID 时创建一个），然后生成 CA 和服务器证书，注册 CA，获取隧道令牌，并将所有输出写入目标位置。

| 标志                | 描述                                                                         | 默认值                                                     |
| ----------------- | -------------------------------------------------------------------------- | ------------------------------------------------------- |
| `--api-url`       | Claude API 基础 URL。也可从 `API_URL` 读取。                                        | 必填                                                      |
| `--tunnel-id`     | 要附加到的隧道 ID（`tnl_...`）。也可从 `TUNNEL_ID` 读取。省略时会创建新隧道；重新运行时会复用输出中已存储的隧道 ID。   | 无（创建隧道）                                                 |
| `--output`        | 输出目标：`dir:/path` 或 `k8s-secret:NAME`。Helm chart 传入 `k8s-secret:<release>`。 | `k8s-secret:mcp-tunnel`（在 Kubernetes pod 中运行时自动检测；否则必填） |
| `--cert-duration` | 服务器证书有效期。                                                                  | `2160h`（90 天）                                           |
| `--token-version` | 变更检测字符串。新值会在重新运行时触发令牌轮换。Helm chart 和 Compose 示例均传入 `1` 作为初始值。              | 无                                                       |

该命令通过 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation) 进行身份验证。它读取 `ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_WORKSPACE_ID`（可选），以及 `ANTHROPIC_IDENTITY_TOKEN_FILE` 或 `ANTHROPIC_IDENTITY_TOKEN` 中的恰好一个。请参阅 [WIF 参考](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference)了解这些变量的当前语义；设置组件会从联合规则推导出服务账户，因此不需要单独提供 `ANTHROPIC_SERVICE_ACCOUNT_ID`。

### `setup renew-cert`

签发一个由已存储的 CA 签名的新服务器证书。不进行任何 API 调用。

| 标志                | 描述                                                                         | 默认值                                                     |
| ----------------- | -------------------------------------------------------------------------- | ------------------------------------------------------- |
| `--output`        | 输出目标：`dir:/path` 或 `k8s-secret:NAME`。Helm chart 传入 `k8s-secret:<release>`。 | `k8s-secret:mcp-tunnel`（在 Kubernetes pod 中运行时自动检测；否则必填） |
| `--cert-duration` | 新证书有效期。                                                                    | `2160h`（90 天）                                           |
| `--renew-before`  | 如果现有证书的剩余有效期超过此时长，则跳过续期。                                                   | `0`（始终续期）                                               |

设置 `--renew-before=720h` 会使该命令在剩余有效期超过 30 天时不执行任何操作，因此可以安全地按固定计划运行。
