---
title: 远程 MCP 服务器
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/remote-mcp-servers
description: 通过 MCP 连接器 API 将 Claude 连接到第三方远程 MCP 服务器。浏览示例服务器并查看连接步骤。
---

多家公司已部署了远程 MCP 服务器，开发者可以通过 Anthropic MCP 连接器 API 连接到这些服务器。这些服务器通过 MCP 协议提供对各种服务和工具的远程访问，从而扩展了开发者和最终用户可用的功能。

<Note>
  下面列出的远程 MCP 服务器是专为与 Claude API 配合使用而设计的第三方服务。这些服务器并非由 Anthropic 拥有、运营或认可。用户应仅连接到自己信任的远程 MCP 服务器，并应在连接之前查看每个服务器的安全实践和条款。
</Note>

## 连接到远程 MCP 服务器

要连接到远程 MCP 服务器：

1. 查看您想要使用的特定服务器的文档。
2. 确保您拥有必要的身份验证凭据。
3. 按照各公司提供的特定于服务器的连接说明进行操作。

有关在 Claude API 中使用远程 MCP 服务器的更多信息，请参阅 [MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)。

<Note>
  连接后，远程 MCP 工具遵循与任何其他工具相同的触发行为。请参阅 [Claude 何时使用 MCP 工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#when-claude-uses-mcp-tools)。
</Note>

## 远程 MCP 服务器示例

<MCPServersTable platform="mcpConnector" />

<Note>
  **想要了解更多？** [在 GitHub 上查找数百个更多的 MCP 服务器](https://github.com/modelcontextprotocol/servers)。
</Note>
