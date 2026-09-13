---
title: WIF 参考
url: https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference
description: Workload Identity Federation 的环境变量、验证规则、配置文件配置以及错误参考。
---

本页汇总了 [Workload Identity Federation](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)（工作负载身份联合）的配置界面、验证约束和错误映射。有关设置演练，请参阅[提供商指南](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#identity-providers)。

## 令牌交换请求

`POST /v1/oauth/token` 接受使用 [RFC 7523](https://www.rfc-editor.org/rfc/rfc7523) `jwt-bearer` 授权类型的 JSON 请求体。SDK 会根据[环境变量](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#environment-variables)为您构建此请求；每个提供商指南中的 cURL 示例展示了原始请求体。

| 字段                   | 是否必需 | 描述                                                                                                   |
| -------------------- | ---- | ---------------------------------------------------------------------------------------------------- |
| `grant_type`         | 是    | 始终为 `urn:ietf:params:oauth:grant-type:jwt-bearer`。                                                   |
| `assertion`          | 是    | 由您的身份提供商签发的 OIDC JWT。                                                                                |
| `federation_rule_id` | 是    | 要评估的联合规则的带标签 ID（`fdrl_...`）。                                                                         |
| `organization_id`    | 是    | 您的 Anthropic 组织的 UUID。                                                                               |
| `service_account_id` | 是    | 目标服务账户的带标签 ID（`svac_...`）。                                                                           |
| `workspace_id`       | 有条件  | 要将所签发令牌限定到的工作区的带标签 ID（`wrkspc_...`），或字面值 `default`（表示组织的默认工作区）。当规则为多个工作区启用时必需。省略时，服务器会选择该规则唯一启用的工作区。 |

## 令牌交换响应

`POST /v1/oauth/token` 返回标准的 OAuth 2.0 令牌响应（[RFC 6749 §5.1](https://www.rfc-editor.org/rfc/rfc6749#section-5.1)）：

| 字段             | 类型      | 描述                                                                                 |
| -------------- | ------- | ---------------------------------------------------------------------------------- |
| `access_token` | string  | 短期有效的 Anthropic 令牌，前缀为 `sk-ant-oat01-...`。以 `Authorization: Bearer <token>` 的形式传递。 |
| `token_type`   | string  | 始终为 `Bearer`。                                                                      |
| `expires_in`   | integer | 令牌到期前的秒数。                                                                          |
| `scope`        | string  | 匹配规则所授予的 OAuth 作用域。                                                                |

## 环境变量

SDK 读取这些变量，以便在无需构造函数参数的情况下执行联合令牌交换。

| 变量                              | 是否必需                          | 描述                                                                                                                            | 示例                                     |
| ------------------------------- | ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| `ANTHROPIC_FEDERATION_RULE_ID`  | 是                             | 要评估的联合规则的带标签 ID。                                                                                                              | `fdrl_...`                             |
| `ANTHROPIC_ORGANIZATION_ID`     | 是                             | 您的 Anthropic 组织的 UUID。可在 Claude Console 的 **Settings > Organization** 下找到。                                                    | `00000000-0000-0000-0000-000000000000` |
| `ANTHROPIC_IDENTITY_TOKEN_FILE` | `_TOKEN_FILE` 或 `_TOKEN` 二者之一 | 由您的"identity provider"（身份提供商），即 IdP 签发的 JWT 的文件系统路径。SDK 在每次交换时都会重新读取此文件，以便磁盘上轮换的投射令牌始终是最新的。                                   | `/var/run/secrets/anthropic.com/token` |
| `ANTHROPIC_IDENTITY_TOKEN`      | `_TOKEN_FILE` 或 `_TOKEN` 二者之一 | 以字符串形式提供的字面 JWT。当您的平台以环境变量而非文件的形式注入令牌时使用。                                                                                     | `eyJhbGciOiJSUzI1NiIs...`              |
| `ANTHROPIC_SERVICE_ACCOUNT_ID`  | 是                             | 所签发访问令牌所代表的目标 Anthropic 服务账户的带标签 ID。                                                                                          | `svac_...`                             |
| `ANTHROPIC_WORKSPACE_ID`        | 有条件                           | 要将所签发令牌限定到的工作区的带标签 ID，或字面值 `default`。当联合规则为多个工作区启用时必需；当规则绑定到单个工作区时可选。所签发的令牌在交换时即被限定到此工作区，因此切换工作区需要进行新的交换。                     | `wrkspc_...`                           |
| `ANTHROPIC_PROFILE`             | 否                             | 要加载的[配置文件](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#profile-configuration-file)的名称。优先级高于本表中的联合环境变量。 | `staging-profile`                      |

仅当 `ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_SERVICE_ACCOUNT_ID` 以及 `ANTHROPIC_IDENTITY_TOKEN_FILE` 或 `ANTHROPIC_IDENTITY_TOKEN` 之一全部设置时，直接环境变量联合路径才会激活。`ANTHROPIC_WORKSPACE_ID` 会一并读取，但不影响激活条件。

<Warning>
  设置为空字符串的变量仍会占据其在凭据优先级链中的位置。如果导出了 `ANTHROPIC_API_KEY=""`，SDK 会选择使用空密钥的 API 密钥路径，而不会回退到联合。请取消设置未使用的凭据变量，而不是将其置空。
</Warning>

### 凭据优先级

SDK 按以下顺序解析凭据。第一个产生凭据的来源胜出。

| 顺序 | 来源                                              | 说明                                                                                                                                 |
| -- | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| 1  | 构造函数参数（`api_key=`、`auth_token=`、`credentials=`） | 始终覆盖其他所有来源。                                                                                                                        |
| 2  | `ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN`    | 完全遮蔽联合。从 API 密钥迁移时请取消设置这些变量。                                                                                                       |
| 3  | `ANTHROPIC_PROFILE`                             | 加载 `<config_dir>/configs/<name>.json`。指定名称的配置文件缺失是一个错误，而不会回退。                                                                      |
| 4  | 联合环境变量                                          | `ANTHROPIC_FEDERATION_RULE_ID` + `ANTHROPIC_ORGANIZATION_ID` + `ANTHROPIC_SERVICE_ACCOUNT_ID` + `ANTHROPIC_IDENTITY_TOKEN[_FILE]`。 |
| 5  | 活动配置文件                                          | 从 `<config_dir>/active_config` 解析，回退到名为 `default` 的配置文件。                                                                           |

加载配置文件时，环境变量会填充配置文件省略的任何字段，但绝不会覆盖配置文件显式设置的字段。例如，仅当活动配置文件未设置 `workspace_id` 时，`ANTHROPIC_WORKSPACE_ID` 才会填充该字段。

## 配置文件配置

"Profile"（配置文件）是 SDK 和 `ant` CLI 都会读取的命名配置文件。配置文件让您可以将联合参数随容器镜像一起发布，或在不更改代码的情况下在不同环境之间切换。

### 配置目录

SDK 按以下顺序定位配置目录：

1. `$ANTHROPIC_CONFIG_DIR`
2. Linux 和 macOS 上的 `~/.config/anthropic`
3. Windows 上的 `%APPDATA%\Anthropic`

### 活动配置文件

活动配置文件名称按以下顺序解析：

1. `$ANTHROPIC_PROFILE`
2. `<config_dir>/active_config` 的内容（由 `ant profile activate <name>` 写入的单行文件）
3. 字面名称 `default`

Claude Code 和 Claude Agent SDK 遵循相同的解析顺序，因此在此处配置的联合配置文件也可以为这些工具进行身份验证，无需额外设置。

### 文件布局

| 路径                                        | 内容                                                                          | 敏感性                     |
| ----------------------------------------- | --------------------------------------------------------------------------- | ----------------------- |
| `<config_dir>/configs/<profile>.json`     | `version`、`authentication` 块、`organization_id`、`workspace_id` 和 `base_url`。 | 非机密。可以安全地提交或烘焙到镜像中。     |
| `<config_dir>/credentials/<profile>.json` | `version`、缓存的 `access_token`、`expires_at`，以及（对于交互式登录）`refresh_token`。       | 机密。由 SDK 以 `0600` 模式写入。 |

配置文件和凭据文件都带有一个顶层字符串 `version` 字段，格式为 `major.minor`（当前为 `"1.0"`）。SDK 会自动写入此字段，以便未来版本能够检测并迁移旧格式；手动编写配置时可省略它，SDK 会将该文件视为当前版本。

### 联合配置文件示例

```json configs/production.json
{
  "version": "1.0",
  "authentication": {
    "type": "oidc_federation",
    "federation_rule_id": "fdrl_...",
    "service_account_id": "svac_...",
    "identity_token": {
      "source": "file",
      "path": "/var/run/secrets/anthropic.com/token"
    }
  },
  "organization_id": "00000000-0000-0000-0000-000000000000",
  "workspace_id": "wrkspc_...",
  "base_url": "https://api.anthropic.com"
}
```

如果省略 `authentication.identity_token`，SDK 会回退到环境中的 `ANTHROPIC_IDENTITY_TOKEN_FILE` 或 `ANTHROPIC_IDENTITY_TOKEN`。

## OAuth 作用域

您在联合规则上设置的 `oauth_scope` 决定了所签发的访问令牌可以调用哪些 Claude API 端点。

| 作用域                        | 授予访问权限                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workspace:developer`      | 规则所在工作区中的所有非管理类 Claude API 端点：[Messages](https://platform.claude.com/docs/zh-CN/api/messages)（包括流式传输和令牌计数）、[Models](https://platform.claude.com/docs/zh-CN/api/models-list)、[Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) 及其会话、[Files](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 和 [Skills](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide)。这与同一工作区中的工作区 API 密钥所拥有的访问权限一致。 |
| `workspace:inference`      | 规则所在工作区中的推理端点：[Messages](https://platform.claude.com/docs/zh-CN/api/messages)（包括流式传输和令牌计数）、[Models](https://platform.claude.com/docs/zh-CN/api/models-list) 以及 [OpenAI 兼容聊天端点](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/openai-sdk)。适用于只需调用 Claude 而从不需要管理 Files、Skills 或其他资源的工作负载。                                                                                                                                             |
| `workspace:manage_tunnels` | [MCP 隧道 API](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/reference#tunnels-api)：创建、列出和获取隧道，注册和归档 CA 证书，显示和轮换隧道令牌，以及归档隧道。当您从 Console 的创建隧道模态窗口创建规则时，该窗口会锁定此作用域。                                                                                                                                                                                                                                                                     |
| `org:admin`                | 对 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 的完全访问权限（组织成员、邀请、工作区、API 密钥等）。OAuth `org:admin` 令牌只能创建或修改作用域为 `workspace:developer` 或 `workspace:inference` 的规则，并且不能更新支撑任何其他作用域规则的签发者；请参阅[约束](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api#permissions-and-constraints)。                                                                                                                                |

对令牌作用域之外的端点发出的请求会返回 HTTP 403。目前不提供更细粒度的作用域（按资源，或读与写）。

### 权限边界

联合规则的 `oauth_scope` 是一个上限：所签发的令牌永远不能超过它。目标服务账户的 `organization_role`（`developer` 或 `admin`）决定了哪些作用域可被授予，因此授予 `org:admin` 的规则必须以 `organization_role=admin` 的服务账户为目标。有效权限是规则作用域与服务账户角色的交集。

| 规则 `oauth_scope`      | 服务账户 `organization_role` | 有效权限                                                                                                                                                                |
| --------------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workspace:developer` | `admin`                  | 仅限规则所在工作区中的 Claude API 访问权限。作用域将令牌限制在角色之下。                                                                                                                          |
| `org:admin`           | `admin`                  | 完全的 Admin API 访问权限（组织成员、邀请、工作区、API 密钥等），但不包括 OAuth 调用方的例外项；请参阅[约束](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api#permissions-and-constraints)。 |

## 验证规则

当您创建或更新签发者和规则时，以及在交换时验证传入的 JWT 时，Anthropic 会强制执行这些约束。

有关完整的参数详情和响应模式，请参阅[服务账户 API 参考](https://platform.claude.com/docs/zh-CN/api/admin/service_accounts)、[联合签发者 API 参考](https://platform.claude.com/docs/zh-CN/api/admin/federation_issuers)和[联合规则 API 参考](https://platform.claude.com/docs/zh-CN/api/admin/federation_rules)。

### 资源字段

| 字段                          | 约束                                                                                                                                                                                            |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 签发者、规则和服务账户的 `name`         | 必须匹配 `^[a-z0-9-]+$`，长度为 1 到 255 个字符。                                                                                                                                                          |
| `workspace_id`              | 创建时必需，除非 `applies_to_all_workspaces` 为 true。其配额、计费和速率限制适用于在此规则下签发的令牌的工作区（`wrkspc_...`）。必须是同一组织中的工作区，并且目标服务账户必须是该工作区的成员。                                                                       |
| `applies_to_all_workspaces` | 布尔值。设置为 `true` 可在组织中的每个工作区启用该规则，而不是指定一个工作区；创建时必须提供此字段或 `workspace_id` 之一。                                                                                                                     |
| `token_lifetime_seconds`    | 介于 `60` 和 `86400` 之间的整数（1 分钟到 24 小时）。默认为 `3600`。超出此范围的值会在请求时被拒绝。请参阅[令牌生命周期与刷新](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#token-lifetime-and-refresh)。 |

### URL 字段

`issuer_url`、`jwks.discovery_base` 和 `jwks.url` 字段会经过验证：

| 约束 | 详情                                                  |
| -- | --------------------------------------------------- |
| 协议 | 必须为 `https`。                                        |
| 端口 | 必须为 `443`（显式或默认）。                                   |
| 主机 | 必须是您的 OIDC 提供商的公共 DNS 主机名。必须解析为公共 IP 地址；不接受 IP 字面量。 |

URL 验证失败会返回 `400 invalid_request_error`，错误消息以字段名作为前缀（例如 `issuer_url: url must use https scheme`）。

<Note>
  URL 约束仅适用于 Anthropic 会访问的 URL。在 `explicit_url` 和 `inline` JWKS 模式下，以及在设置了 `jwks.discovery_base` 的 `discovery` 模式下，`issuer_url` 仅作为字符串与 JWT `iss` 声明进行比较，从不会被获取，因此它可以引用内部主机名或非标准端口。
</Note>

### JWT 验证

| 约束     | 详情                                                                                                                                                                                             |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 最大大小   | `assertion` JWT 最大为 16 KiB。                                                                                                                                                                    |
| 签名算法   | 仅接受非对称算法（RSA 和 ECDSA 系列：ES256、ES384、ES512、RS256、RS384、RS512、PS256、PS384、PS512）。HMAC（`HS256`、`HS384`、`HS512`）和 `none` 会被拒绝。                                                                     |
| 密钥 ID  | JWT 头部必须携带与签发者 JWKS 中某个密钥匹配的 `kid`。没有 `kid` 的令牌会被拒绝。                                                                                                                                           |
| 必需声明   | `sub` 必须存在。`iat` 必须存在且不在未来。`exp` 必须存在且在未来。                                                                                                                                                     |
| 单次使用   | 携带 `jti` 声明的断言对每个签发者只能交换一次：使用相同 `jti` 重复交换会被视为重放而拒绝。签发者的 `check_jti` 字段（默认启用）控制此检查；没有 `jti` 声明的断言不受此限制。请参阅[联合签发者 API 参考](https://platform.claude.com/docs/zh-CN/api/admin/federation_issuers)。 |
| 最大生命周期 | 令牌的生命周期（`exp` 减去 `iat`）不得超过签发者配置的最大值（默认为 1 小时，可在 Claude Console 中为每个签发者配置）。                                                                                                                    |
| 时钟偏差   | 对 `exp`、`nbf` 和 `iat` 应用 30 秒的宽限。                                                                                                                                                              |

## 规则匹配语义

联合规则的 `match` 块决定是否接受传入的 JWT。所有已填充的字段以 AND 语义进行评估：JWT 必须满足每个已填充的匹配器。`subject_prefix`、`claims` 或 `condition` 中至少必须设置一个；仅包含 `audience`（或完全没有匹配器）的 `match` 块会被拒绝。这可以防止出现会接受来自某个签发者的所有令牌的规则。

| 匹配器              | 类型                   | 语义                                                                             |
| ---------------- | -------------------- | ------------------------------------------------------------------------------ |
| `subject_prefix` | string               | 与 JWT `sub` 声明精确匹配。末尾的 `*` 使其成为前缀匹配（`sub` 值必须以 `*` 之前的字符开头）。区分大小写。             |
| `audience`       | string               | JWT `aud` 声明必须包含此精确字符串。当 `aud` 是数组时，任何精确匹配的元素都满足检查。                            |
| `claims`         | map\<string, string> | 每个键是一个顶层声明名称，每个值是所需的精确字符串值。对于嵌套、数值、布尔或复杂声明（如列表和映射），请改用带有 CEL 表达式的 `condition`。 |
| `condition`      | string (CEL)         | 必须求值为 `true` 的 [CEL](https://cel.dev/) 表达式。                                    |

### CEL 求值环境

`condition` 表达式可以访问单个变量：

| 变量       | 类型  | 内容                            |
| -------- | --- | ----------------------------- |
| `claims` | map | 完整的已解码 JWT 声明集。嵌套对象可作为嵌套映射访问。 |

示例：

```text wrap
claims.sub.startsWith("repo:acme-corp/") && claims.ref in ["refs/heads/main", "refs/heads/release"]
```

<Warning>
  CEL 条件是安全边界。对比预期更多的输入求值为 `true` 的表达式会授予比预期更广泛的访问权限。当静态匹配器能够表达您的约束时，请优先使用它们。
</Warning>

## 错误

### 令牌交换错误

`POST /v1/oauth/token` 以标准的 [API 错误格式](https://platform.claude.com/docs/zh-CN/api/errors)返回错误。SDK 将交换失败包装在类型化的 `FederationExchangeError`（或对应语言的等价类型）中，该类型公开 HTTP 状态、响应体和 `request_id`。

| 状态  | 错误                      | 原因                                                                        | 解决方法                                                                                                                                                                                                          |
| --- | ----------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 400 | `invalid_request_error` | `federation_rule_id` 格式错误或缺少必需的请求字段。                                      | 验证 `fdrl_` ID，并确认请求体包含所有必需字段。                                                                                                                                                                                 |
| 400 | `invalid_request_error` | `workspace_id` 存在，但不是格式正确的 `wrkspc_...` ID 或字面值 `default`。                | 修正 `workspace_id` 值；响应消息会指明预期格式。                                                                                                                                                                              |
| 401 | `authentication_error`  | JWT `iss` 声明与注册的 `issuer_url` 不完全相等。                                      | 逐字节比较，包括末尾斜杠和协议：`jq -rR 'split(".")[1] \| gsub("-";"+") \| gsub("_";"/") \| @base64d \| fromjson \| .iss' <<< "$JWT"`。                                                                                        |
| 401 | `authentication_error`  | JWKS 获取失败、JWKS 已过期，或 JWT 使用了不在 JWKS 中的密钥签名。                               | 对于 `inline` 模式，使用轮换后的密钥更新签发者。对于 `discovery` 和 `explicit_url`，确认 JWKS 端点可通过端口 443 访问；如果签发者最近轮换了签名密钥，请参阅[密钥轮换与缓存](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#key-rotation-and-caching)。 |
| 401 | `authentication_error`  | JWT `exp` 声明已过期（超出 30 秒偏差窗口）。                                             | 确认您的身份提供商正在投射新令牌，并且 SDK 正在重新读取令牌文件。                                                                                                                                                                           |
| 401 | `authentication_error`  | JWT 已通过验证，但其声明不满足规则的 `match` 块。                                           | 解码 JWT 并将每个声明与规则进行比较。`subject_prefix` 区分大小写。`audience` 要求元素精确匹配。                                                                                                                                              |
| 401 | `authentication_error`  | `federation_rule_id` 不存在、已归档，或 JWT 未获得其授权（为防止枚举而合并）。                      | 在 Claude Console 中确认规则 ID，并确认规则未被归档。                                                                                                                                                                          |
| 401 | `authentication_error`  | 联合规则为多个工作区启用，而请求省略了 `workspace_id`。身份验证历史条目显示原因为 `workspace_id_required`。 | 将 `ANTHROPIC_WORKSPACE_ID`（或原始请求中的 `workspace_id` 请求体字段）设置为您希望令牌限定到的 `wrkspc_...` ID。请参阅[令牌交换请求](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#token-exchange-request)。                  |

无论哪项检查失败，每次断言拒绝都会返回相同的不透明 `401` `authentication_error`，并带有固定消息 `Authentication failed`；可区分的错误会让调用方得以探测规则配置。拒绝原因记录在[身份验证历史](https://platform.claude.com/settings/workload-identity-federation?tab=history)中该次尝试的条目上，例如当 `sub` 声明未通过规则的 `subject_prefix` 时为 `match_subject_prefix`，或当规则跨越多个工作区而请求未指定任何工作区时为 `workspace_id_required`。在规则所属组织得到确认之前被拒绝的请求（上述 `400 invalid_request_error` 系列）不会留下历史条目；其响应消息会直接指明问题。没有匹配历史条目的 `401` 通常意味着 `federation_rule_id` 本身未被识别。

### 常见的 SDK 端故障

| 症状                            | 原因                                                                                                                                             | 解决方法                                                                                                                |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| SDK 报告"no credentials"而不是进行交换 | `ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_SERVICE_ACCOUNT_ID` 或 `ANTHROPIC_IDENTITY_TOKEN[_FILE]` 之一未设置，且没有活动配置文件。 | 设置全部四个变量，或配置一个配置文件。                                                                                                 |
| SDK 使用 API 密钥进行身份验证而不是联合      | 设置了 `ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN` 并赢得了优先级。                                                                                      | 取消设置密钥或令牌变量。                                                                                                        |
| 首次请求时出现 `FileNotFoundError`   | `ANTHROPIC_IDENTITY_TOKEN_FILE` 中的路径不存在。SDK 在交换时才延迟打开该文件。                                                                                      | 确认投射令牌卷已挂载且路径匹配。                                                                                                    |
| 令牌交换成功但 Claude API 请求返回 403   | 所签发令牌的作用域未授予对该端点的访问权限。                                                                                                                         | 对照 [OAuth 作用域](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#oauth-scopes)检查规则的 `oauth_scope`。 |
| 身份验证因空凭据而失败                   | 某个凭据环境变量已导出但设置为空字符串。空值仍会赢得其优先级位置。                                                                                                              | 使用 `unset VAR` 而不是 `VAR=""` 来取消设置变量。                                                                                |

## 排查失败的交换

`401` `authentication_error` 响应是有意设计为不透明的，其消息始终为 `Authentication failed`；拒绝原因记录在身份验证历史中，而不在响应中。

<Tip>
  请从 Claude Console 中的[身份验证历史页面](https://platform.claude.com/settings/workload-identity-federation?tab=history)开始。最近的交换尝试会显示所评估的签发者和规则、所检查的 JWT 声明以及哪个验证步骤失败，这通常可以省去以下检查。
</Tip>

一种常见的不透明故障是重放的断言：携带 `jti` 声明的断言[只能交换一次](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#jwt-verification)，因此重新发送相同 JWT 的工作负载（重试循环，或重新读取未轮换令牌的刷新）会在第二次交换时被拒绝。身份验证历史页面会以原因 `jti_reused` 显示这些尝试；解决方法是为每次交换签发新的断言。

如果您仍需要从 JWT 本身进行调试，请按顺序完成以下检查：

<Steps>
  <Step title="解码 JWT">
    解码您发送的断言，以便将每个声明与您的签发者和规则配置进行比较：

    ```bash cURL
    jq -rR 'split(".")[1] | gsub("-";"+") | gsub("_";"/") | @base64d | fromjson' <<< "$JWT"
    ```
  </Step>

  <Step title="检查 iss 是否与签发者匹配">
    解码后的 `iss` 声明必须与注册的 `issuer_url` 逐字节相等，包括协议、端口和任何末尾斜杠。单个字符不匹配即会导致验证失败。
  </Step>

  <Step title="检查 aud 是否与规则匹配">
    解码后的 `aud` 声明必须以精确匹配的方式包含规则的 `audience` 值。当 `aud` 是数组时，必须有一个元素精确匹配。
  </Step>

  <Step title="检查 sub 和每个 claims 条目">
    将 `sub` 与规则的 `subject_prefix` 进行比较（区分大小写；末尾的 `*` 表示前缀匹配，其他情况为精确匹配）。将规则 `claims` 映射中的每个键与同名的顶层声明进行比较。
  </Step>

  <Step title="检查 exp、nbf 和 iat">
    `exp` 必须在未来，`nbf`/`iat` 必须在过去，且在 30 秒偏差窗口内。如果工作负载主机的时钟发生漂移，原本有效的令牌也会被拒绝。
  </Step>

  <Step title="检查 JWKS 可达性">
    对于 `discovery` 模式，通过端口 443 上的公共 HTTPS 获取 `<jwks.discovery_base or issuer_url>/.well-known/openid-configuration`，并确认 `jwks_uri` 可解析。对于 `explicit_url`，直接获取 JWKS URL。对于 `inline`，确认自您注册密钥以来签发者的签名密钥未发生轮换。

    如果签发者轮换了签名密钥并立即开始使用它签名，在 Anthropic 的 JWKS 缓存刷新期间，交换可能会失败长达一分钟。请参阅[密钥轮换与缓存](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#key-rotation-and-caching)。
  </Step>
</Steps>

## JWKS 来源模式

当您注册联合签发者时，`jwks` 字段控制 Anthropic 如何获取用于验证来自该签发者的 JWT 签名的公钥。它是一个以 `type` 为键的可辨识联合：

| `jwks.type`     | `jwks` 结构                                                                                                    | 行为                                                                                                              | 适用场景                                                                                     |
| --------------- | ------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `discovery`（默认） | `{ "type": "discovery", "discovery_base": "https://..." }`（`discovery_base` 可选；当发现 URL 与 `issuer_url` 不同时设置） | Anthropic 获取 `<discovery_base or issuer_url>/.well-known/openid-configuration`，从发现文档中读取 `jwks_uri`，并从那里获取 JWKS。 | 您的 IdP 在公共互联网上提供标准的 OIDC 发现文档。大多数托管提供商（EKS、GKE、Cloud Run、GitHub Actions、Entra ID）都支持此方式。 |
| `explicit_url`  | `{ "type": "explicit_url", "url": "https://..." }`                                                           | Anthropic 直接从 `url` 获取 JWKS。`issuer_url` 仅用于与 JWT `iss` 声明进行字符串比较，从不会被访问。                                       | 您的 IdP 不提供发现文档，或者发现仅限内部但 JWKS 可公开访问。                                                     |
| `inline`        | `{ "type": "inline", "keys": [...] }`                                                                        | 您以内联方式提供 JWK 对象数组（JWKS 文档中的 `keys` 数组，而非包装对象）。Anthropic 不发出任何出站请求。`issuer_url` 仅用于 `iss` 比较。                    | 气隙隔离环境、签发者 URL 为集群内部地址的自管理 Kubernetes 集群，或您希望显式控制密钥轮换的情况。                                |

可辨识联合在结构上使配套字段互斥。`discovery` 和 `explicit_url` 还都接受一个可选的 `ca_cert_pem` 字符串，用于从私有 CA 提供 TLS 的签发者。

### 密钥轮换与缓存

在 `discovery` 和 `explicit_url` 模式下，Anthropic 会缓存获取到的 JWKS。如果您的身份提供商发布了新的签名密钥并立即开始使用它签名令牌，则在缓存刷新期间，提交这些令牌的交换可能会因签名错误而失败长达 1 分钟。

为避免此窗口期，请在您的身份提供商开始使用新签名密钥签名令牌之前至少 15 分钟将其发布到 JWKS 中，并将被取代的密钥保留在 JWKS 中，直到其签名的令牌过期。托管身份提供商通常会自行遵循此规范。如果您运营自己的签发者（自管理 Kubernetes 集群、SPIRE OIDC 发现提供商，或配置了轮换周期的 Okta 自定义授权服务器），请确认您的轮换策略会在首次使用之前发布新密钥。

<Warning>
  在 `inline` 模式下没有自动密钥刷新。当您的身份提供商轮换其签名密钥时，您必须使用新的 JWKS 更新签发者配置，否则所有令牌交换都将无法通过签名验证。
</Warning>
