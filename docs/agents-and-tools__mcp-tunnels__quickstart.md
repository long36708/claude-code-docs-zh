---
title: MCP 隧道快速入门
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/quickstart
description: 使用本地 Docker Compose 部署将 Claude 连接到私有 MCP 服务器。
---

<Note>
  MCP 隧道目前处于研究预览阶段。[申请访问权限](https://claude.com/form/claude-managed-agents)以试用。
</Note>

本快速入门将带您从零开始，直到 Claude 通过隧道调用私有 MCP 服务器。它使用 Docker Compose 并采用[手动](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#credential-provisioning)凭证配置（credential provisioning），这是本地测试的最短路径。对于生产部署，请参阅[使用 Helm 部署](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-helm)或[使用 Docker Compose 部署](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose)。

## 您将构建的内容

一个由两个容器组成的[隧道栈](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)（tunnel stack，包括[代理](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)（proxy）和 [cloudflared](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)），以及一个与之并行运行的示例 MCP 服务器。当一切运行起来后，即使没有任何服务在公共端口上监听，Claude 也可以通过 `https://echo.<your-tunnel-domain>/mcp` 访问该示例服务器。

## 您需要准备的内容

* 在一台具有出站互联网访问权限的机器上安装 [Docker 和 Docker Compose](https://docs.docker.com/get-docker/)。
* 在 [Claude Console](https://platform.claude.com) 中拥有可以管理 MCP 隧道的角色。请参阅 [Console 指南的先决条件](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/console#prerequisites)。
* [OpenSSL](https://openssl-library.org/source/) 1.1.1 或更高版本。macOS 和大多数 Linux 发行版已预装；在 Windows 上需要单独安装（`openssl` 二进制文件必须位于您的 `PATH` 中）。

<Steps>
  <Step title="创建隧道">
    在 Claude Console 侧边栏中，前往 **Manage > MCP tunnels** 并点击 **New tunnel**。为其命名。保持 **Set up programmatic access** 为关闭状态；本快速入门使用手动凭证配置。

    创建完成后，打开该隧道。从 **Connection** 部分复制两个值：

    * **Domain**（形如 `abcd1234.tunnel.anthropic.com`）
    * **Token**（点击眼睛图标，然后复制）
  </Step>

  <Step title="设置部署目录">
    <Tabs>
      <Tab title="macOS / Linux">
        ```bash
        mkdir -p mcp-tunnel/{config,data}
        cd mcp-tunnel
        export TUNNEL_DOMAIN=YOUR_TUNNEL_DOMAIN_HERE   # from step 1
        export TUNNEL_TOKEN='eyJ...'            # from step 1
        ```
      </Tab>

      <Tab title="Windows (PowerShell)">
        ```powershell
        New-Item -ItemType Directory -Force -Path mcp-tunnel/config, mcp-tunnel/data | Out-Null
        Set-Location mcp-tunnel
        $env:TUNNEL_DOMAIN = "YOUR_TUNNEL_DOMAIN_HERE"   # from step 1
        $env:TUNNEL_TOKEN  = "eyJ..."             # from step 1
        ```
      </Tab>
    </Tabs>
  </Step>

  <Step title="生成 CA 和服务器证书">
    代理使用由您控制的 CA 签发的证书来终止[内层 TLS](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/concepts#components)（inner TLS）。生成这两者：

    <Tabs>
      <Tab title="macOS / Linux">
        ```bash
        openssl req -x509 -newkey rsa:2048 -nodes \
          -keyout data/ca.key -out data/ca.crt \
          -days 3650 -subj "/CN=mcp-tunnel-ca" \
          -addext "basicConstraints=critical,CA:TRUE" \
          -addext "keyUsage=critical,keyCertSign,cRLSign" \
          -addext "subjectKeyIdentifier=hash"

        cat > data/tls.ext <<EOF
        subjectAltName = DNS:${TUNNEL_DOMAIN},DNS:*.${TUNNEL_DOMAIN}
        authorityKeyIdentifier = keyid,issuer
        extendedKeyUsage = serverAuth
        EOF

        openssl req -newkey rsa:2048 -nodes \
          -keyout data/tls.key -out /tmp/server.csr \
          -subj "/CN=${TUNNEL_DOMAIN}"
        openssl x509 -req -in /tmp/server.csr \
          -CA data/ca.crt -CAkey data/ca.key -CAcreateserial \
          -out data/tls.crt -days 90 -extfile data/tls.ext

        chmod 644 data/tls.key
        ```
      </Tab>

      <Tab title="Windows (PowerShell)">
        ```powershell
        openssl req -x509 -newkey rsa:2048 -nodes `
          -keyout data/ca.key -out data/ca.crt `
          -days 3650 -subj "/CN=mcp-tunnel-ca" `
          -addext "basicConstraints=critical,CA:TRUE" `
          -addext "keyUsage=critical,keyCertSign,cRLSign" `
          -addext "subjectKeyIdentifier=hash"

        @"
        subjectAltName = DNS:$env:TUNNEL_DOMAIN,DNS:*.$env:TUNNEL_DOMAIN
        authorityKeyIdentifier = keyid,issuer
        extendedKeyUsage = serverAuth
        "@ | Set-Content -NoNewline -Encoding ascii -Path data/tls.ext

        openssl req -newkey rsa:2048 -nodes `
          -keyout data/tls.key -out data/server.csr `
          -subj "/CN=$env:TUNNEL_DOMAIN"
        openssl x509 -req -in data/server.csr `
          -CA data/ca.crt -CAkey data/ca.key -CAcreateserial `
          -out data/tls.crt -days 90 -extfile data/tls.ext
        ```
      </Tab>
    </Tabs>

    回到 Console，在隧道详情页面上，点击 **Add certificate** 并上传 `data/ca.crt`（或粘贴其内容）。隧道状态将变为 **Active**。
  </Step>

  <Step title="编写示例 MCP 服务器">
    <Tabs>
      <Tab title="macOS / Linux">
        ```bash
        cat > hello_server.py <<'EOF'
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
      </Tab>

      <Tab title="Windows (PowerShell)">
        ```powershell
        @'
        from mcp.server.fastmcp import FastMCP

        mcp = FastMCP("hello-server", host="0.0.0.0", port=9000)


        @mcp.tool()
        def hello(name: str = "world") -> str:
            """Say hello to someone."""
            return f"Hello, {name}!"


        if __name__ == "__main__":
            mcp.run(transport="streamable-http")
        '@ | Set-Content -NoNewline -Encoding ascii -Path hello_server.py
        ```
      </Tab>
    </Tabs>
  </Step>

  <Step title="编写代理配置和 compose 文件">
    <Tabs>
      <Tab title="macOS / Linux">
        ```bash
        cat > config/mcp-proxy.yaml <<EOF
        listen_addr: ":8080"
        tunnel_domain: ${TUNNEL_DOMAIN}
        tls:
          cert_file: /data/tls.crt
          key_file: /data/tls.key
        routes:
          echo: http://hello-mcp:9000
        EOF

        cat > docker-compose.yaml <<'EOF'
        services:
          mcp-proxy:
            image: us-docker.pkg.dev/anthropic-public-registry/images/mcp-proxy@sha256:efb27b299d627e4134815663cb8896641eeaee025d734c0f695582b4df38f013
            volumes:
              - ./config/mcp-proxy.yaml:/etc/mcp-gateway/config.yaml:ro
              - ./data:/data:ro
            restart: unless-stopped

          cloudflared:
            image: cloudflare/cloudflared@sha256:6b599ca3e974349ead3286d178da61d291961182ec3fe9c505e1dd02c8ac31b0
            command: tunnel --no-autoupdate run --url http://localhost:8080
            environment:
              - TUNNEL_TOKEN
            network_mode: "service:mcp-proxy"
            restart: unless-stopped

          hello-mcp:
            image: python:3.13-slim
            working_dir: /app
            volumes:
              - ./hello_server.py:/app/hello_server.py:ro
            command: sh -c "pip install --quiet mcp && python hello_server.py"
            restart: unless-stopped
        EOF
        ```
      </Tab>

      <Tab title="Windows (PowerShell)">
        ```powershell
        @"
        listen_addr: ":8080"
        tunnel_domain: $env:TUNNEL_DOMAIN
        tls:
          cert_file: /data/tls.crt
          key_file: /data/tls.key
        routes:
          echo: http://hello-mcp:9000
        "@ | Set-Content -NoNewline -Encoding ascii -Path config/mcp-proxy.yaml

        @'
        services:
          mcp-proxy:
            image: us-docker.pkg.dev/anthropic-public-registry/images/mcp-proxy@sha256:efb27b299d627e4134815663cb8896641eeaee025d734c0f695582b4df38f013
            volumes:
              - ./config/mcp-proxy.yaml:/etc/mcp-gateway/config.yaml:ro
              - ./data:/data:ro
            restart: unless-stopped

          cloudflared:
            image: cloudflare/cloudflared@sha256:6b599ca3e974349ead3286d178da61d291961182ec3fe9c505e1dd02c8ac31b0
            command: tunnel --no-autoupdate run --url http://localhost:8080
            environment:
              - TUNNEL_TOKEN
            network_mode: "service:mcp-proxy"
            restart: unless-stopped

          hello-mcp:
            image: python:3.13-slim
            working_dir: /app
            volumes:
              - ./hello_server.py:/app/hello_server.py:ro
            command: sh -c "pip install --quiet mcp && python hello_server.py"
            restart: unless-stopped
        '@ | Set-Content -NoNewline -Encoding ascii -Path docker-compose.yaml
        ```
      </Tab>
    </Tabs>
  </Step>

  <Step title="启动">
    <Tabs>
      <Tab title="macOS / Linux">
        ```bash
        docker compose up -d
        docker compose logs mcp-proxy | grep "route configured"
        docker compose logs cloudflared | grep "Registered tunnel connection"
        ```
      </Tab>

      <Tab title="Windows (PowerShell)">
        ```powershell
        docker compose up -d
        docker compose logs mcp-proxy | Select-String "route configured"
        docker compose logs cloudflared | Select-String "Registered tunnel connection"
        ```
      </Tab>
    </Tabs>

    您应该会看到一行针对 `echo` 的 `route configured` 日志，以及四行 `Registered tunnel connection` 日志。容器需要几秒钟才能启动；如果日志命令返回为空，请重新运行。
  </Step>

  <Step title="从 Claude 调用">
    在 Console 中，前往 **Managed Agents > Sessions** 并创建一个会话。在智能体选择器中选择 **Create new agent**，为智能体命名，并保留预填的模型。点击 **+ MCP Server**，选择您的隧道，将 **Subdomain** 设置为 `echo`，将 **Path** 设置为 `mcp`。然后提问：

    > Use the hello tool to greet tunnel.

    您应该会看到一次工具调用，随后是其结果。
  </Step>
</Steps>

## 后续步骤

隧道已完成端到端验证。要换用您自己的 MCP 服务器，请将其添加到 `docker-compose.yaml`（或在同一 Docker 网络上运行它），在 `config/mcp-proxy.yaml` 中为其添加一条路由，然后重启代理（`docker compose restart mcp-proxy`）。

对于生产部署：

<CardGroup cols={2}>
  <Card title="使用 Docker Compose 部署" icon="cube" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-compose">
    经过加固的单主机部署，可选择是否启用编程访问。
  </Card>

  <Card title="使用 Helm 部署" icon="stack" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/deploy-helm">
    具有自动凭证管理的 Kubernetes 部署。
  </Card>
</CardGroup>
