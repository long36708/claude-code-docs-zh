---
title: 使用 Docker Compose 部署 MCP 隧道
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose
description: 使用 Docker Compose 在虚拟机上安装 MCP 隧道栈。
---

<Note>
  MCP 隧道目前处于研究预览阶段。[申请访问权限](https://claude.com/form/claude-managed-agents)以试用。
</Note>

本指南将[隧道栈](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)作为加固容器部署在单台主机上。相同的配置可以复制到多台主机上以实现高可用性。

## 开始之前

您需要：

* **一个隧道。** 使用编程访问时，如果您不提供隧道 ID，[setup 组件](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)会为您创建一个；若要改为附加到现有隧道，请[在 Console 中创建它](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#create-a-tunnel)并记录隧道 ID（`tnl_...`）。手动配置始终从 Console 创建的隧道开始。

* **主机向 Tunnels API 进行身份验证的方式。**

  * **编程访问（推荐）。** 创建隧道时开启 **Set up programmatic access**（或者，如果您让 setup 组件创建隧道，则直接在 **Settings > Workload identity** 下创建联合规则），以便 setup 组件可以通过 "Workload Identity Federation"（工作负载身份联合）进行身份验证。记录联合规则 ID（`fdrl_...`）和您的组织 ID。
  * **手动。** 跳过编程访问。您将[从 Console 获取隧道令牌](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#get-the-connection-details)，自行生成 CA 和服务器证书，并[在 Console 中注册 CA](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#add-a-ca-certificate)。

* **一台安装了 Docker 和 Docker Compose 的主机。** 手动流程还需要 `openssl`（1.1.1 或更高版本）。

* **出站网络连接**，从主机到 `api.anthropic.com`（443 TCP）以及[隧道边缘](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)（7844 TCP 和 UDP）。请参阅完整的[网络要求](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview#network-requirements)。

* **一个或多个 MCP 服务器**，正在运行且可从主机通过您将在 `routes` 下配置的地址访问。如果您还没有，请[使用示例服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose#optional-use-a-sample-mcp-server)。

## 可选：使用示例 MCP 服务器

如果您没有可用于测试的 MCP 服务器，请使用这个最小化的服务器：

```bash
mkdir -p mcp-tunnel
cat > mcp-tunnel/hello_server.py <<'EOF'
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("hello-server", host="0.0.0.0", port=9000)


@mcp.tool()
def hello(name: str = "world") -> str:
    """Say hello to someone."""
    return f"Hello, {name}!"


if __name__ == "__main__":
    mcp.run(transport="streamable-http")
EOF
```

以下安装步骤会 `cd` 进入 `mcp-tunnel/`，并注明在何处添加相应的服务和路由。

## 安装

本指南提供一种使用 Docker Compose 的参考方法。您有责任对其进行调整，以满足您组织的安全要求。

<Tabs>
  <Tab title="使用编程访问">
    此路径要求主机具有 OIDC 身份提供商（例如云虚拟机元数据服务器或 SPIFFE）。如果没有，请改用**不使用编程访问**选项卡。

    setup 组件使用 Workload Identity Federation 获取隧道令牌、生成 CA 和服务器证书，并向 Anthropic 注册 CA。

    <Steps>
      <Step title="准备部署目录">
        ```bash
        mkdir -p mcp-tunnel/{config,data}
        cd mcp-tunnel
        sudo chown 65532:65532 data
        ```

        容器以非 root UID `65532` 运行，需要对 `data/` 具有写入权限。
      </Step>

      <Step title="编写 docker-compose.yaml">
        该 compose 文件通过 SHA-256 摘要固定镜像，以非 root 身份和只读文件系统运行每个容器，丢弃所有 Linux capabilities，并禁用权限提升。

        ```bash
        cat > docker-compose.yaml <<'EOF'
        services:
          setup:
            image: us-docker.pkg.dev/anthropic-public-registry/images/mcp-proxy@sha256:efb27b299d627e4134815663cb8896641eeaee025d734c0f695582b4df38f013
            entrypoint: ["/setup"]
            command:
              - init
              - --api-url=https://api.anthropic.com
              - --output=dir:/data
              - --token-version=1
            environment:
              - TUNNEL_ID
              - ANTHROPIC_FEDERATION_RULE_ID
              - ANTHROPIC_ORGANIZATION_ID
              - ANTHROPIC_WORKSPACE_ID
              - ANTHROPIC_IDENTITY_TOKEN
            volumes:
              - ./data:/data
            user: "65532:65532"
            read_only: true
            security_opt:
              - no-new-privileges:true
            cap_drop:
              - ALL
            profiles: ["setup"]

          cloudflared:
            image: cloudflare/cloudflared@sha256:6b599ca3e974349ead3286d178da61d291961182ec3fe9c505e1dd02c8ac31b0
            command: tunnel --no-autoupdate run --url http://localhost:8080
            environment:
              - TUNNEL_TOKEN
            # 共享代理的 netns，以便通过 localhost:8080 访问它。
            network_mode: "service:mcp-proxy"
            restart: unless-stopped
            user: "65532:65532"
            read_only: true
            security_opt:
              - no-new-privileges:true
            cap_drop:
              - ALL
            stop_grace_period: 30s
            logging:
              options:
                max-size: "10m"
                max-file: "3"

          mcp-proxy:
            image: us-docker.pkg.dev/anthropic-public-registry/images/mcp-proxy@sha256:efb27b299d627e4134815663cb8896641eeaee025d734c0f695582b4df38f013
            volumes:
              - ./config/mcp-proxy.yaml:/etc/mcp-gateway/config.yaml:ro
              - ./data:/data:ro
            restart: unless-stopped
            user: "65532:65532"
            read_only: true
            security_opt:
              - no-new-privileges:true
            cap_drop:
              - ALL
            stop_grace_period: 30s
            logging:
              options:
                max-size: "10m"
                max-file: "3"
        EOF
        ```

        如果您使用的是[示例 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose#optional-use-a-sample-mcp-server)，请将其作为服务追加：

        ```bash
        cat >> docker-compose.yaml <<'EOF'

          hello-mcp:
            image: python:3.13-slim
            working_dir: /app
            volumes:
              - ./hello_server.py:/app/hello_server.py:ro
            command: sh -c "pip install --quiet mcp && python hello_server.py"
            restart: unless-stopped
        EOF
        ```
      </Step>

      <Step title="配置隧道">
        设置标识符。不设置 `TUNNEL_ID` 可让 setup 组件创建隧道；设置它则附加到来自 [Console](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#create-a-tunnel) 的现有隧道：

        ```bash
        # export TUNNEL_ID=tnl_...   # 设置此项以连接到现有隧道
        export ANTHROPIC_FEDERATION_RULE_ID=fdrl_...
        export ANTHROPIC_ORGANIZATION_ID=00000000-0000-0000-0000-000000000000
        ```

        如果您的联合规则的作用域是组织默认工作区以外的工作区，还需设置 `ANTHROPIC_WORKSPACE_ID=wrkspc_...`；否则 setup 组件使用默认工作区。自动创建的隧道会创建在该工作区中。

        将 `ANTHROPIC_IDENTITY_TOKEN` 设置为来自此主机身份提供商的 OIDC JWT。请按照[适用于您提供商的 WIF 指南](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#identity-providers)注册颁发者、设置规则的 subject 并签发令牌；规则的 audience 必须与您签发时请求的 audience 匹配。

        运行 setup 组件：

        ```bash
        docker compose run --rm setup
        ```

        `setup init` 对 `data/` 是幂等的：重新运行它会复用已存储在其中的隧道 ID 和 CA，绝不会创建第二个隧道。仅当 `data/` 为空或 `TUNNEL_ID` 已更改时，才会生成并注册新的 CA；在这种情况下，适用两个活动证书的上限，因此如果两个槽位都已占用，请先在 Console 中吊销一个。

        如果出错，请参阅 [Setup 组件身份验证失败](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/troubleshooting#setup-component-authentication-failures)。

        获取您的隧道域名并将其导出以供后续步骤使用：

        ```bash
        export TUNNEL_DOMAIN=$(sudo cat data/tunnel-domain)
        echo "$TUNNEL_DOMAIN"
        ```

        <Note>
          Workload Identity Federation 令牌是短期的（默认 1 小时）并会自动过期；setup 完成后无需吊销任何内容。
        </Note>
      </Step>

      <Step title="编写代理配置">
        `tunnel_domain` 是**必需的**：[代理](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)使用它从传入的主机名中剥离域名后缀，然后在 `routes` 中查找子域名。`routes` 是从子域名到上游 URL 的扁平映射，而不是列表。

        ```bash
        cat > config/mcp-proxy.yaml <<EOF
        listen_addr: ":8080"
        log_level: info
        shutdown_timeout: 30s
        tunnel_domain: ${TUNNEL_DOMAIN}
        tls:
          cert_file: /data/tls.crt
          key_file: /data/tls.key
        routes:
          echo: http://hello-mcp:9000
        EOF
        ```

        `echo:` 路由指向[示例 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose#optional-use-a-sample-mcp-server)；请将其替换为（或添加）您自己的路由。有关所有可用字段，请参阅[代理配置](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/reference#proxy-configuration)参考。
      </Step>

      <Step title="启动部署">
        ```bash
        export TUNNEL_TOKEN=$(sudo cat data/tunnel-token)
        docker compose up -d
        ```
      </Step>
    </Steps>
  </Tab>

  <Tab title="不使用编程访问">
    如果您没有开启 **Set up programmatic access**，或者用于本地开发和测试，请使用此流程。此流程没有 `setup` 服务。

    <Steps>
      <Step title="从 Console 获取隧道令牌和域名">
        在隧道详情页面上，复制 **Domain**（其形式为 `abcd1234.tunnel.anthropic.com`），然后点击 **Token** 旁边的眼睛图标获取隧道令牌，并使用复制图标将其复制。

        将两者设置为 shell 变量，以供本指南其余部分使用：

        ```bash
        export TUNNEL_DOMAIN=YOUR_TUNNEL_DOMAIN_HERE
        export TUNNEL_TOKEN='eyJ...'
        ```
      </Step>

      <Step title="搭建目录并生成证书">
        ```bash
        mkdir -p mcp-tunnel/{data,config}
        cd mcp-tunnel
        ```

        代理通过明文 WebSocket 监听 `:8080`；[内层 TLS](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components) 握手使用这些证书在该 WebSocket 流**内部**进行。Anthropic 根据您在 Console 中注册的 CA 验证内层握手。根据[证书要求](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/reference#certificate-requirements)，服务器证书的 "Subject Alternative Name"（主体备用名称），即 SAN，必须包含 `*.<tunnel-domain>`。

        ```bash
        # 自签名 CA。显式指定扩展，使其满足证书要求，
        # 不受发行版 openssl.cnf 默认值的影响。
        openssl req -x509 -newkey rsa:2048 -nodes \
          -keyout data/ca.key -out data/ca.crt \
          -days 3650 -subj "/CN=mcp-tunnel-ca" \
          -addext "basicConstraints=critical,CA:TRUE" \
          -addext "keyUsage=critical,keyCertSign,cRLSign" \
          -addext "subjectKeyIdentifier=hash"

        # 服务器证书的扩展文件。使用 -extfile（而非
        # 仅 OpenSSL 3.0+ 支持的 -copy_extensions）可确保在
        # OpenSSL 1.1.x 上正常工作。
        cat > data/tls.ext <<EOF
        subjectAltName = DNS:${TUNNEL_DOMAIN},DNS:*.${TUNNEL_DOMAIN}
        authorityKeyIdentifier = keyid,issuer
        extendedKeyUsage = serverAuth
        EOF

        # 由 CA 签名的服务器证书
        openssl req -newkey rsa:2048 -nodes \
          -keyout data/tls.key -out /tmp/server.csr \
          -subj "/CN=${TUNNEL_DOMAIN}"
        openssl x509 -req -in /tmp/server.csr \
          -CA data/ca.crt -CAkey data/ca.key -CAcreateserial \
          -out data/tls.crt -days 90 \
          -extfile data/tls.ext

        # 允许非 root 代理容器（UID 65532）从绑定挂载中
        # 读取密钥。若无全局可读位，容器将无法打开
        # 宿主机所有的文件。
        chmod 644 data/tls.key
        ```
      </Step>

      <Step title="在 Console 中注册 CA 证书">
        在隧道详情页面上，滚动到 **Certificates** 部分并点击 **Add certificate**。使用 **Choose file** 直接上传 `data/ca.crt`（该对话框接受 `.pem`、`.crt` 和 `.cer`），或粘贴其内容：

        ```bash
        cat data/ca.crt
        ```

        注册证书后，隧道的状态会变为 **Active**。请参阅[添加 CA 证书](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#add-a-ca-certificate)。
      </Step>

      <Step title="编写代理配置">
        `tunnel_domain` 是**必需的**：代理使用它从传入的主机名中剥离域名后缀，然后在 `routes` 中查找子域名。`routes` 是从子域名到上游 URL 的扁平映射，而不是列表。

        ```bash
        cat > config/mcp-proxy.yaml <<EOF
        listen_addr: ":8080"
        log_level: info
        tunnel_domain: ${TUNNEL_DOMAIN}
        tls:
          cert_file: /data/tls.crt
          key_file: /data/tls.key
        routes:
          echo: http://hello-mcp:9000
        EOF
        ```

        `echo:` 路由指向[示例 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose#optional-use-a-sample-mcp-server)；请将其替换为（或添加）您自己的路由。有关所有可用字段，请参阅[代理配置](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/reference#proxy-configuration)参考。
      </Step>

      <Step title="编写 docker-compose.yaml">
        `network_mode: "service:mcp-proxy"` 设置将 [cloudflared](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components) 置于代理的网络命名空间中，以便 cloudflared 容器内的 `localhost:8080` 能够访问代理。`--url http://localhost:8080` 标志为 cloudflared 指定其转发目标；如果没有该标志，cloudflared 对传入请求没有路由，会返回 503。

        ```bash
        cat > docker-compose.yaml <<'EOF'
        services:
          cloudflared:
            image: cloudflare/cloudflared@sha256:6b599ca3e974349ead3286d178da61d291961182ec3fe9c505e1dd02c8ac31b0
            # --url 为必需项：手动流程中不会推送任何 ingress 规则，
            # 因此若缺少它，cloudflared 会对每个请求返回 503。
            command: tunnel --no-autoupdate run --url http://localhost:8080
            environment:
              - TUNNEL_TOKEN
            # 共享代理的 netns，使 localhost:8080 能够访问到它。
            network_mode: "service:mcp-proxy"
            restart: unless-stopped
            user: "65532:65532"
            read_only: true
            security_opt:
              - no-new-privileges:true
            cap_drop:
              - ALL
            stop_grace_period: 30s
            logging:
              options:
                max-size: "10m"
                max-file: "3"

          mcp-proxy:
            image: us-docker.pkg.dev/anthropic-public-registry/images/mcp-proxy@sha256:efb27b299d627e4134815663cb8896641eeaee025d734c0f695582b4df38f013
            volumes:
              - ./config/mcp-proxy.yaml:/etc/mcp-gateway/config.yaml:ro
              - ./data:/data:ro
            restart: unless-stopped
            user: "65532:65532"
            read_only: true
            security_opt:
              - no-new-privileges:true
            cap_drop:
              - ALL
            stop_grace_period: 30s
            logging:
              options:
                max-size: "10m"
                max-file: "3"
        EOF
        ```

        如果您使用的是[示例 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose#optional-use-a-sample-mcp-server)，请将其作为服务追加：

        ```bash
        cat >> docker-compose.yaml <<'EOF'

          hello-mcp:
            image: python:3.13-slim
            working_dir: /app
            volumes:
              - ./hello_server.py:/app/hello_server.py:ro
            command: sh -c "pip install --quiet mcp && python hello_server.py"
            restart: unless-stopped
        EOF
        ```
      </Step>

      <Step title="启动部署">
        ```bash
        docker compose up -d
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

compose 文件从主机环境读取 `TUNNEL_TOKEN` 且没有默认值，因此在每个新的 shell 中以及重启后都必须重新执行 export。

对于多虚拟机部署，请将 `mcp-tunnel/` 目录复制到每台主机，设置 `TUNNEL_TOKEN`，然后运行 `docker compose up -d`。在编程流程中，`TUNNEL_TOKEN` 为 `$(sudo cat data/tunnel-token)`；在手动流程中，它是您从 Console 复制的值。相同的隧道令牌和证书可在所有副本中使用。

## 验证部署

通过从 Anthropic 一侧调用[上游 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)进行端到端验证：请参阅[使用隧道化的 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview#use-the-tunneled-mcp-servers)。使用[示例 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose#optional-use-a-sample-mcp-server)时，路由后的 URL 为 `https://echo.<your-tunnel-domain>/mcp`。如果验证失败，请参阅[故障排除](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/troubleshooting)。

## 升级

请在 `mcp-tunnel/` 部署目录内运行本节中的命令。

### 轮换隧道令牌

使用编程访问时，递增 `setup` 服务命令中的 `--token-version`，设置 Workload Identity Federation 标识符，签发新的 OIDC JWT，然后重新运行 setup 组件：

```bash
# 编辑 docker-compose.yaml：递增 setup 服务的
# --token-version 参数中的整数（例如，从 --token-version=1 改为
# --token-version=2）。当该值未发生变化时，setup 二进制文件
# 会拒绝轮换。

# export TUNNEL_ID=tnl_...   # 仅当您在安装时设置过它时才设置
export ANTHROPIC_FEDERATION_RULE_ID=fdrl_...
export ANTHROPIC_ORGANIZATION_ID=00000000-0000-0000-0000-000000000000
# export ANTHROPIC_WORKSPACE_ID=wrkspc_...   # 如果您的规则是工作区范围的
# 按照适用于您环境的 WIF 提供商指南重新签发 ANTHROPIC_IDENTITY_TOKEN
# （自安装以来它应已过期）。
export ANTHROPIC_IDENTITY_TOKEN=...

docker compose run --rm setup

export TUNNEL_TOKEN=$(sudo cat data/tunnel-token)
docker compose up -d cloudflared
```

`--token-version` 参数在 `docker-compose.yaml` 中编辑，而不是在命令行上传递，这样新值会在 setup 组件的后续运行中持久保留。setup 组件使用 Workload Identity Federation 进行身份验证；没有需要吊销的 API 令牌。

不使用编程访问时，在 Console 的隧道详情页面上点击 **Rotate token**，然后更新每台主机上的 `TUNNEL_TOKEN` 环境变量并重启 cloudflared（`docker compose up -d cloudflared`）。

<Warning>
  点击 **Rotate token** 会立即使当前令牌失效。从那一刻起，到在每台主机上更新 `TUNNEL_TOKEN` 并重启 cloudflared 之前，任何 cloudflared 发生重启（崩溃、主机重启）的主机都无法重新连接。轮换后请及时更新每台主机。
</Warning>

### 证书续期

您有责任监控到期时间，并在服务器证书到期前进行续期。

使用编程访问时：

```bash
docker compose run --rm setup renew-cert --output=dir:/data
```

CLI 参数会替换 `setup` 服务的 `command`（即 `init` 参数），但保留其 `entrypoint`，因此这会运行 `/setup renew-cert --output=dir:/data`。

<Tip>
  传递 `--renew-before=720h` 可使该命令在剩余有效期超过 30 天时成为空操作。这使其可以安全地按固定计划运行。
</Tip>

不使用编程访问时，使用您现有的 CA 签署新的服务器证书（在 Console 中注册的 CA 不会改变）并替换 `data/tls.crt`。如果您是在新的 shell 中运行，请先设置 `TUNNEL_DOMAIN`。

```bash
export TUNNEL_DOMAIN=YOUR_TUNNEL_DOMAIN_HERE
openssl req -new -key data/tls.key -out /tmp/server.csr \
  -subj "/CN=${TUNNEL_DOMAIN}"
openssl x509 -req -in /tmp/server.csr \
  -CA data/ca.crt -CAkey data/ca.key -CAcreateserial \
  -out data/tls.crt -days 90 \
  -extfile data/tls.ext
```

在任一流程中，代理都会轮询 `tls.cert_file` 并自动重新加载，因此无需重启。

## 后续步骤

<CardGroup cols={2}>
  <Card title="使用隧道化的 MCP 服务器" icon="link" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview#use-the-tunneled-mcp-servers">
    将上游 MCP 服务器附加到 Managed Agent 或 Messages API。
  </Card>

  <Card title="安全" icon="lock" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/security">
    加固指南、凭证轮换和入侵响应。
  </Card>

  <Card title="故障排除" icon="wrench" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/troubleshooting">
    诊断连接、TLS 和路由问题。
  </Card>
</CardGroup>
