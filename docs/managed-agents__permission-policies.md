---
title: 权限策略
url: https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies
description: 控制智能体工具和 MCP 工具何时执行。
featureMetadata:
  topic:
    title: Managed Agents
    url: https://platform.claude.com/docs/en/managed-agents/overview
  status: beta
  betaHeader: managed-agents-2026-04-01
---

"Permission policies"（权限策略）控制服务器执行的工具（预构建的智能体工具集和 MCP 工具集）是自动运行、等待您的批准，还是由服务器逐一评估每次调用。"Custom tools"（自定义工具）由您的应用程序执行并由您控制，因此不受权限策略约束。

## 权限策略类型

| 策略             | 行为                                                                                                                                                                                  |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `always_allow` | 工具自动执行，无需确认。                                                                                                                                                                        |
| `always_ask`   | 会话暂停，并在执行前等待您的批准。有关事件流程，请参阅[响应确认请求](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies#respond-to-confirmation-requests)。                                    |
| `auto`         | 服务器评估每次调用，然后运行该调用、拒绝该调用或暂停以等待您的批准。请参阅[使用 `auto` 让服务器评估每次调用](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies#let-the-server-evaluate-each-call-with-auto)。 |

每种工具集类型都有自己的默认值：智能体工具集默认为 `always_allow`，MCP 工具集默认为 `always_ask`。

权限策略控制已启用的工具何时运行。若要将某个工具从智能体中完全移除，请改为禁用它。请参阅[禁用特定工具](https://platform.claude.com/docs/zh-CN/managed-agents/tools#disabling-specific-tools)。

## 为工具集设置策略

您在创建智能体时于智能体的 `tools` 配置中设置权限策略，之后可以通过[更新智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#update-an-agent)来更改它们。正在运行的会话会保留其创建时的工具集配置。更新仅适用于之后创建的会话。

### 智能体工具集权限

创建智能体时，您可以使用 `default_config.permission_policy` 将策略应用于 `agent_toolset_20260401` 中的每个工具：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  agent=$(curl -fsSL https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "name": "Coding Assistant",
      "model": "claude-opus-5-5",
      "tools": [
        {
          "type": "agent_toolset_20260401",
          "default_config": {
            "permission_policy": {"type": "always_ask"}
          }
        }
      ]
    }')
  ```

  <CodeGroupItem>
    ```bash CLI
    ant apply agent.md
    ```

    <File filename="agent.md">
      ```markdown
      ---
      name: Coding Assistant
      model: claude-opus-5-5
      tools:
        - type: agent_toolset_20260401
          default_config:
            permission_policy:
              type: always_ask
      ---
      ```
    </File>
  </CodeGroupItem>

  ```python Python
  agent = client.beta.agents.create(
      name="Coding Assistant",
      model="claude-opus-5-5",
      tools=[
          {
              "type": "agent_toolset_20260401",
              "default_config": {
                  "permission_policy": {"type": "always_ask"},
              },
          },
      ],
  )
  ```

  ```typescript TypeScript
  const agent = await client.beta.agents.create({
    name: "Coding Assistant",
    model: "claude-opus-5-5",
    tools: [
      {
        type: "agent_toolset_20260401",
        default_config: {
          permission_policy: { type: "always_ask" }
        }
      }
    ]
  });
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Agents;

  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Coding Assistant",
      Model = BetaManagedAgentsModel.ClaudeOpus5_5,
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = BetaManagedAgentsAgentToolset20260401ParamsType.AgentToolset20260401,
              DefaultConfig = new()
              {
                  PermissionPolicy = new BetaManagedAgentsAlwaysAskPolicy { Type = "always_ask" },
              },
          },
      ],
  });
  ```

  ```go Go
  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "Coding Assistant",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: "claude-opus-5-5",
  	},
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  		OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  			Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  			DefaultConfig: anthropic.BetaManagedAgentsAgentToolsetDefaultConfigParams{
  				PermissionPolicy: anthropic.BetaManagedAgentsAgentToolsetDefaultConfigParamsPermissionPolicyUnion{
  					OfAlwaysAsk: &anthropic.BetaManagedAgentsAlwaysAskPolicyParam{
  						Type: anthropic.BetaManagedAgentsAlwaysAskPolicyTypeAlwaysAsk,
  					},
  				},
  			},
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }
  _ = agent
  ```

  ```java Java
  import com.anthropic.models.beta.agents.*;

  var agent = client.beta().agents().create(
      AgentCreateParams.builder()
          .name("Coding Assistant")
          .model(BetaManagedAgentsModel.CLAUDE_OPUS_5_5)
          .addTool(
              BetaManagedAgentsAgentToolset20260401Params.builder()
                  .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
                  .defaultConfig(
                      BetaManagedAgentsAgentToolsetDefaultConfigParams.builder()
                          .permissionPolicy(
                              BetaManagedAgentsAlwaysAskPolicy.builder()
                                  .type(BetaManagedAgentsAlwaysAskPolicy.Type.ALWAYS_ASK)
                                  .build()
                          )
                          .build()
                  )
                  .build()
          )
          .build()
  );
  ```

  ```php PHP
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolsetDefaultConfigParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsAlwaysAskPolicy;

  $agent = $client->beta->agents->create(
      name: 'Coding Assistant',
      model: 'claude-opus-5-5',
      tools: [
          BetaManagedAgentsAgentToolset20260401Params::with(
              type: 'agent_toolset_20260401',
              defaultConfig: BetaManagedAgentsAgentToolsetDefaultConfigParams::with(
                  permissionPolicy: BetaManagedAgentsAlwaysAskPolicy::with(type: 'always_ask'),
              ),
          ),
      ],
  );
  ```

  ```ruby Ruby
  agent = client.beta.agents.create(
    name: "Coding Assistant",
    model: "claude-opus-5-5",
    tools: [
      {
        type: "agent_toolset_20260401",
        default_config: {
          permission_policy: {type: "always_ask"}
        }
      }
    ]
  )
  ```
</CodeGroup>

`default_config` 是可选的。如果省略它，智能体工具集将以默认权限策略 `always_allow` 启用。

### MCP 工具集权限

MCP 工具集默认为 `always_ask`。这可确保添加到 MCP 服务器的新工具不会在未经批准的情况下在您的应用程序中执行。若要自动批准来自受信任 MCP 服务器的工具，请在 `mcp_toolset` 条目上设置 `default_config.permission_policy`。

`mcp_server_name` 必须与 `mcp_servers` 数组中某个服务器的 `name` 匹配。

此示例连接一个 GitHub MCP 服务器，并允许其工具无需确认即可运行：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  agent=$(curl -fsSL https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "name": "Dev Assistant",
      "model": "claude-opus-5-5",
      "mcp_servers": [
        {"type": "url", "name": "github", "url": "https://mcp.example.com/github"}
      ],
      "tools": [
        {"type": "agent_toolset_20260401"},
        {
          "type": "mcp_toolset",
          "mcp_server_name": "github",
          "default_config": {
            "permission_policy": {"type": "always_allow"}
          }
        }
      ]
    }')
  ```

  <CodeGroupItem>
    ```bash CLI
    ant apply agent.md
    ```

    <File filename="agent.md">
      ```markdown
      ---
      name: Dev Assistant
      model: claude-opus-5-5
      mcp_servers:
        - type: url
          name: github
          url: https://mcp.example.com/github
      tools:
        - type: agent_toolset_20260401
        - type: mcp_toolset
          mcp_server_name: github
          default_config:
            permission_policy:
              type: always_allow
      ---
      ```
    </File>
  </CodeGroupItem>

  ```python Python
  agent = client.beta.agents.create(
      name="Dev Assistant",
      model="claude-opus-5-5",
      mcp_servers=[
          {"type": "url", "name": "github", "url": "https://mcp.example.com/github"},
      ],
      tools=[
          {"type": "agent_toolset_20260401"},
          {
              "type": "mcp_toolset",
              "mcp_server_name": "github",
              "default_config": {
                  "permission_policy": {"type": "always_allow"},
              },
          },
      ],
  )
  ```

  ```typescript TypeScript
  const agent = await client.beta.agents.create({
    name: "Dev Assistant",
    model: "claude-opus-5-5",
    mcp_servers: [{ type: "url", name: "github", url: "https://mcp.example.com/github" }],
    tools: [
      { type: "agent_toolset_20260401" },
      {
        type: "mcp_toolset",
        mcp_server_name: "github",
        default_config: {
          permission_policy: { type: "always_allow" }
        }
      }
    ]
  });
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Agents;

  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Dev Assistant",
      Model = BetaManagedAgentsModel.ClaudeOpus5_5,
      McpServers =
      [
          new()
          {
              Type = BetaManagedAgentsUrlMcpServerParamsType.Url,
              Name = "github",
              Url = "https://mcp.example.com/github",
          },
      ],
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = BetaManagedAgentsAgentToolset20260401ParamsType.AgentToolset20260401,
          },
          new BetaManagedAgentsMcpToolsetParams
          {
              Type = BetaManagedAgentsMcpToolsetParamsType.McpToolset,
              McpServerName = "github",
              DefaultConfig = new()
              {
                  PermissionPolicy = new BetaManagedAgentsAlwaysAllowPolicy { Type = "always_allow" },
              },
          },
      ],
  });
  ```

  ```go Go
  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "Dev Assistant",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: "claude-opus-5-5",
  	},
  	MCPServers: []anthropic.BetaManagedAgentsURLMCPServerParams{{
  		Type: anthropic.BetaManagedAgentsURLMCPServerParamsTypeURL,
  		Name: "github",
  		URL:  "https://mcp.example.com/github",
  	}},
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{
  		{
  			OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  				Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  			},
  		},
  		{
  			OfMCPToolset: &anthropic.BetaManagedAgentsMCPToolsetParams{
  				Type:          anthropic.BetaManagedAgentsMCPToolsetParamsTypeMCPToolset,
  				MCPServerName: "github",
  				DefaultConfig: anthropic.BetaManagedAgentsMCPToolsetDefaultConfigParams{
  					PermissionPolicy: anthropic.BetaManagedAgentsMCPToolsetDefaultConfigParamsPermissionPolicyUnion{
  						OfAlwaysAllow: &anthropic.BetaManagedAgentsAlwaysAllowPolicyParam{
  							Type: anthropic.BetaManagedAgentsAlwaysAllowPolicyTypeAlwaysAllow,
  						},
  					},
  				},
  			},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  _ = agent
  ```

  ```java Java
  import com.anthropic.models.beta.agents.*;

  var agent = client.beta().agents().create(
      AgentCreateParams.builder()
          .name("Dev Assistant")
          .model(BetaManagedAgentsModel.CLAUDE_OPUS_5_5)
          .addMcpServer(
              BetaManagedAgentsUrlMcpServerParams.builder()
                  .type(BetaManagedAgentsUrlMcpServerParams.Type.URL)
                  .name("github")
                  .url("https://mcp.example.com/github")
                  .build()
          )
          .addTool(
              BetaManagedAgentsAgentToolset20260401Params.builder()
                  .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
                  .build()
          )
          .addTool(
              BetaManagedAgentsMcpToolsetParams.builder()
                  .type(BetaManagedAgentsMcpToolsetParams.Type.MCP_TOOLSET)
                  .mcpServerName("github")
                  .defaultConfig(
                      BetaManagedAgentsMcpToolsetDefaultConfigParams.builder()
                          .permissionPolicy(
                              BetaManagedAgentsAlwaysAllowPolicy.builder()
                                  .type(BetaManagedAgentsAlwaysAllowPolicy.Type.ALWAYS_ALLOW)
                                  .build()
                          )
                          .build()
                  )
                  .build()
          )
          .build()
  );
  ```

  ```php PHP
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;
  use Anthropic\Beta\Agents\BetaManagedAgentsAlwaysAllowPolicy;
  use Anthropic\Beta\Agents\BetaManagedAgentsMCPToolsetDefaultConfigParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsMCPToolsetParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsURLMCPServerParams;

  $agent = $client->beta->agents->create(
      name: 'Dev Assistant',
      model: 'claude-opus-5-5',
      mcpServers: [
          BetaManagedAgentsURLMCPServerParams::with(
              type: 'url',
              name: 'github',
              url: 'https://mcp.example.com/github',
          ),
      ],
      tools: [
          BetaManagedAgentsAgentToolset20260401Params::with(
              type: 'agent_toolset_20260401',
          ),
          BetaManagedAgentsMCPToolsetParams::with(
              type: 'mcp_toolset',
              mcpServerName: 'github',
              defaultConfig: BetaManagedAgentsMCPToolsetDefaultConfigParams::with(
                  permissionPolicy: BetaManagedAgentsAlwaysAllowPolicy::with(type: 'always_allow'),
              ),
          ),
      ],
  );
  ```

  ```ruby Ruby
  agent = client.beta.agents.create(
    name: "Dev Assistant",
    model: "claude-opus-5-5",
    mcp_servers: [
      {type: "url", name: "github", url: "https://mcp.example.com/github"}
    ],
    tools: [
      {type: "agent_toolset_20260401"},
      {
        type: "mcp_toolset",
        mcp_server_name: "github",
        default_config: {
          permission_policy: {type: "always_allow"}
        }
      }
    ]
  )
  ```
</CodeGroup>

## 覆盖单个工具的策略

使用 `configs` 数组覆盖单个工具的默认值。智能体工具集的 `name` 值列于[可用工具](https://platform.claude.com/docs/zh-CN/managed-agents/tools#available-tools)中。此示例默认允许完整的智能体工具集，但要求在运行任何 bash 命令之前进行确认：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  tools='[
    {
      "type": "agent_toolset_20260401",
      "default_config": {
        "permission_policy": {"type": "always_allow"}
      },
      "configs": [
        {
          "name": "bash",
          "permission_policy": {"type": "always_ask"}
        }
      ]
    }
  ]'
  ```

  <CodeGroupItem>
    ```bash CLI
    ant apply agent.md
    ```

    <File filename="agent.md">
      ```markdown
      ---
      name: Coding Assistant
      model: claude-opus-5-5
      tools:
        - type: agent_toolset_20260401
          default_config:
            permission_policy:
              type: always_allow
          configs:
            - name: bash
              permission_policy:
                type: always_ask
      ---
      ```
    </File>
  </CodeGroupItem>

  ```python Python
  tools = [
      {
          "type": "agent_toolset_20260401",
          "default_config": {
              "permission_policy": {"type": "always_allow"},
          },
          "configs": [
              {
                  "name": "bash",
                  "permission_policy": {"type": "always_ask"},
              },
          ],
      },
  ]
  ```

  ```typescript TypeScript
  const tools = [
    {
      type: "agent_toolset_20260401",
      default_config: {
        permission_policy: { type: "always_allow" }
      },
      configs: [
        {
          name: "bash",
          permission_policy: { type: "always_ask" }
        }
      ]
    }
  ] satisfies Anthropic.Beta.AgentCreateParams["tools"];
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Agents;
  using Tool = Anthropic.Models.Beta.Agents.Tool;

  Tool[] tools =
  [
      new BetaManagedAgentsAgentToolset20260401Params
      {
          Type = BetaManagedAgentsAgentToolset20260401ParamsType.AgentToolset20260401,
          DefaultConfig = new()
          {
              PermissionPolicy = new BetaManagedAgentsAlwaysAllowPolicy { Type = "always_allow" },
          },
          Configs =
          [
              new BetaManagedAgentsBashToolConfigParams
              {
                  PermissionPolicy = new BetaManagedAgentsAlwaysAskPolicy { Type = "always_ask" },
              },
          ],
      },
  ];
  ```

  ```go Go
  tools := []anthropic.BetaAgentNewParamsToolUnion{{
  	OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  		Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  		DefaultConfig: anthropic.BetaManagedAgentsAgentToolsetDefaultConfigParams{
  			PermissionPolicy: anthropic.BetaManagedAgentsAgentToolsetDefaultConfigParamsPermissionPolicyUnion{
  				OfAlwaysAllow: &anthropic.BetaManagedAgentsAlwaysAllowPolicyParam{
  					Type: anthropic.BetaManagedAgentsAlwaysAllowPolicyTypeAlwaysAllow,
  				},
  			},
  		},
  		Configs: []anthropic.BetaManagedAgentsAgentToolConfigParamsUnion{{
  			OfBash: &anthropic.BetaManagedAgentsBashToolConfigParams{
  				PermissionPolicy: anthropic.BetaManagedAgentsBashToolConfigParamsPermissionPolicyUnion{
  					OfAlwaysAsk: &anthropic.BetaManagedAgentsAlwaysAskPolicyParam{
  						Type: anthropic.BetaManagedAgentsAlwaysAskPolicyTypeAlwaysAsk,
  					},
  				},
  			},
  		}},
  	},
  }}
  _ = tools
  ```

  ```java Java
  import com.anthropic.models.beta.agents.*;
  import java.util.List;

  var tools = List.of(
      AgentCreateParams.Tool.ofAgentToolset20260401(
          BetaManagedAgentsAgentToolset20260401Params.builder()
              .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
              .defaultConfig(
                  BetaManagedAgentsAgentToolsetDefaultConfigParams.builder()
                      .permissionPolicy(
                          BetaManagedAgentsAlwaysAllowPolicy.builder()
                              .type(BetaManagedAgentsAlwaysAllowPolicy.Type.ALWAYS_ALLOW)
                              .build()
                      )
                      .build()
              )
              .addConfig(
                  BetaManagedAgentsBashToolConfigParams.builder()
                      .permissionPolicy(
                          BetaManagedAgentsAlwaysAskPolicy.builder()
                              .type(BetaManagedAgentsAlwaysAskPolicy.Type.ALWAYS_ASK)
                              .build()
                      )
                      .build()
              )
              .build()
      )
  );
  ```

  ```php PHP
  use Anthropic\Beta\Agents\BetaManagedAgentsBashToolConfigParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolsetDefaultConfigParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsAlwaysAllowPolicy;
  use Anthropic\Beta\Agents\BetaManagedAgentsAlwaysAskPolicy;

  $tools = [
      BetaManagedAgentsAgentToolset20260401Params::with(
          type: 'agent_toolset_20260401',
          defaultConfig: BetaManagedAgentsAgentToolsetDefaultConfigParams::with(
              permissionPolicy: BetaManagedAgentsAlwaysAllowPolicy::with(type: 'always_allow'),
          ),
          configs: [
              BetaManagedAgentsBashToolConfigParams::with(
                  permissionPolicy: BetaManagedAgentsAlwaysAskPolicy::with(type: 'always_ask'),
              ),
          ],
      ),
  ];
  ```

  ```ruby Ruby
  tools = [
    {
      type: "agent_toolset_20260401",
      default_config: {
        permission_policy: {type: "always_allow"}
      },
      configs: [
        {
          name: "bash",
          permission_policy: {type: "always_ask"}
        }
      ]
    }
  ]
  ```
</CodeGroup>

在智能体创建请求中传递此 `tools` 配置（CLI 选项卡显示了完整命令）。MCP 工具集支持相同的按工具覆盖，其中 `name` 设置为 MCP 服务器报告的工具名称。请参阅[配置哪些 MCP 工具可用](https://platform.claude.com/docs/zh-CN/managed-agents/mcp-connector#configure-which-mcp-tools-are-available)。

## 使用 `auto` 让服务器评估每次调用

使用 `auto` 权限策略时，服务器会在每次调用运行之前对其进行评估。由于评估会考虑工具、调用的输入以及截至该时刻的会话内容，服务器可能会以不同方式处理对同一工具的两次调用。每次调用会产生以下三种结果之一：

* **调用运行。** 当服务器确定调用是安全的时，工具会像在 `always_allow` 下一样运行。
* **调用被拒绝。** 当服务器将调用评估为高风险时，工具不会运行。智能体会收到一个错误工具结果，其内容为 `Permission to use {tool_name} has been denied.`，并带有 `is_error: true`。会话继续运行，且您的客户端无法推翻该拒绝。
* **调用暂停以等待您的批准。** 当服务器无法做出判定时，会话会像在 `always_ask` 下一样暂停。请参阅[响应确认请求](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies#respond-to-confirmation-requests)。

要启用 `auto`，请将 `permission_policy` 设置为 `{"type": "auto"}`。它与其他策略放在相同的两个位置：工具集的 [`default_config`](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies#set-a-policy-for-a-toolset)（作用于整个工具集），或 [`configs` 条目](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies#override-an-individual-tool-policy)（作用于单个工具）。智能体工具集和 MCP 工具集都接受该策略。没有任何工具集默认使用 `auto`。

以下示例将 `auto` 设置为智能体工具集和 `github` MCP 工具集的默认值，并将 `bash` 覆盖为 `always_ask`：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  agent=$(curl -fsSL https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "name": "Ops Agent",
      "model": "claude-opus-5-5",
      "mcp_servers": [
        {"type": "url", "name": "github", "url": "https://mcp.example.com/github"}
      ],
      "tools": [
        {
          "type": "agent_toolset_20260401",
          "default_config": {
            "permission_policy": {"type": "auto"}
          },
          "configs": [
            {"name": "bash", "permission_policy": {"type": "always_ask"}}
          ]
        },
        {
          "type": "mcp_toolset",
          "mcp_server_name": "github",
          "default_config": {
            "permission_policy": {"type": "auto"}
          }
        }
      ]
    }')
  ```

  <CodeGroupItem>
    ```bash CLI
    ant apply agent.md
    ```

    <File filename="agent.md">
      ```markdown
      ---
      name: Ops Agent
      model: claude-opus-5-5
      mcp_servers:
        - type: url
          name: github
          url: https://mcp.example.com/github
      tools:
        - type: agent_toolset_20260401
          default_config:
            permission_policy:
              type: auto
          configs:
            - name: bash
              permission_policy:
                type: always_ask
        - type: mcp_toolset
          mcp_server_name: github
          default_config:
            permission_policy:
              type: auto
      ---
      ```
    </File>
  </CodeGroupItem>

  ```python Python
  agent = client.beta.agents.create(
      name="Ops Agent",
      model="claude-opus-5-5",
      mcp_servers=[
          {"type": "url", "name": "github", "url": "https://mcp.example.com/github"},
      ],
      tools=[
          {
              "type": "agent_toolset_20260401",
              "default_config": {
                  "permission_policy": {"type": "auto"},
              },
              "configs": [
                  {"name": "bash", "permission_policy": {"type": "always_ask"}},
              ],
          },
          {
              "type": "mcp_toolset",
              "mcp_server_name": "github",
              "default_config": {
                  "permission_policy": {"type": "auto"},
              },
          },
      ],
  )
  ```

  ```typescript TypeScript
  const agent = await client.beta.agents.create({
    name: "Ops Agent",
    model: "claude-opus-5-5",
    mcp_servers: [{ type: "url", name: "github", url: "https://mcp.example.com/github" }],
    tools: [
      {
        type: "agent_toolset_20260401",
        default_config: {
          permission_policy: { type: "auto" }
        },
        configs: [{ name: "bash", permission_policy: { type: "always_ask" } }]
      },
      {
        type: "mcp_toolset",
        mcp_server_name: "github",
        default_config: {
          permission_policy: { type: "auto" }
        }
      }
    ]
  });
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Agents;

  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Ops Agent",
      Model = BetaManagedAgentsModel.ClaudeOpus5_5,
      McpServers =
      [
          new()
          {
              Type = BetaManagedAgentsUrlMcpServerParamsType.Url,
              Name = "github",
              Url = "https://mcp.example.com/github",
          },
      ],
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = BetaManagedAgentsAgentToolset20260401ParamsType.AgentToolset20260401,
              DefaultConfig = new()
              {
                  PermissionPolicy = new BetaManagedAgentsAutoPolicy(),
              },
              Configs =
              [
                  new BetaManagedAgentsBashToolConfigParams
                  {
                      PermissionPolicy = new BetaManagedAgentsAlwaysAskPolicy { Type = "always_ask" },
                  },
              ],
          },
          new BetaManagedAgentsMcpToolsetParams
          {
              Type = BetaManagedAgentsMcpToolsetParamsType.McpToolset,
              McpServerName = "github",
              DefaultConfig = new()
              {
                  PermissionPolicy = new BetaManagedAgentsAutoPolicy(),
              },
          },
      ],
  });
  ```

  ```go Go
  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "Ops Agent",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: "claude-opus-5-5",
  	},
  	MCPServers: []anthropic.BetaManagedAgentsURLMCPServerParams{{
  		Type: anthropic.BetaManagedAgentsURLMCPServerParamsTypeURL,
  		Name: "github",
  		URL:  "https://mcp.example.com/github",
  	}},
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{
  		{
  			OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  				Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  				DefaultConfig: anthropic.BetaManagedAgentsAgentToolsetDefaultConfigParams{
  					PermissionPolicy: anthropic.BetaManagedAgentsAgentToolsetDefaultConfigParamsPermissionPolicyUnion{
  						OfAuto: &anthropic.BetaManagedAgentsAutoPolicyParam{},
  					},
  				},
  				Configs: []anthropic.BetaManagedAgentsAgentToolConfigParamsUnion{{
  					OfBash: &anthropic.BetaManagedAgentsBashToolConfigParams{
  						PermissionPolicy: anthropic.BetaManagedAgentsBashToolConfigParamsPermissionPolicyUnion{
  							OfAlwaysAsk: &anthropic.BetaManagedAgentsAlwaysAskPolicyParam{
  								Type: anthropic.BetaManagedAgentsAlwaysAskPolicyTypeAlwaysAsk,
  							},
  						},
  					},
  				}},
  			},
  		},
  		{
  			OfMCPToolset: &anthropic.BetaManagedAgentsMCPToolsetParams{
  				Type:          anthropic.BetaManagedAgentsMCPToolsetParamsTypeMCPToolset,
  				MCPServerName: "github",
  				DefaultConfig: anthropic.BetaManagedAgentsMCPToolsetDefaultConfigParams{
  					PermissionPolicy: anthropic.BetaManagedAgentsMCPToolsetDefaultConfigParamsPermissionPolicyUnion{
  						OfAuto: &anthropic.BetaManagedAgentsAutoPolicyParam{},
  					},
  				},
  			},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  _ = agent
  ```

  ```java Java
  import com.anthropic.models.beta.agents.*;

  var agent = client.beta().agents().create(
      AgentCreateParams.builder()
          .name("Ops Agent")
          .model(BetaManagedAgentsModel.CLAUDE_OPUS_5_5)
          .addMcpServer(
              BetaManagedAgentsUrlMcpServerParams.builder()
                  .type(BetaManagedAgentsUrlMcpServerParams.Type.URL)
                  .name("github")
                  .url("https://mcp.example.com/github")
                  .build()
          )
          .addTool(
              BetaManagedAgentsAgentToolset20260401Params.builder()
                  .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
                  .defaultConfig(
                      BetaManagedAgentsAgentToolsetDefaultConfigParams.builder()
                          .permissionPolicy(BetaManagedAgentsAutoPolicy.builder().build())
                          .build()
                  )
                  .addConfig(
                      BetaManagedAgentsBashToolConfigParams.builder()
                          .permissionPolicy(
                              BetaManagedAgentsAlwaysAskPolicy.builder()
                                  .type(BetaManagedAgentsAlwaysAskPolicy.Type.ALWAYS_ASK)
                                  .build()
                          )
                          .build()
                  )
                  .build()
          )
          .addTool(
              BetaManagedAgentsMcpToolsetParams.builder()
                  .type(BetaManagedAgentsMcpToolsetParams.Type.MCP_TOOLSET)
                  .mcpServerName("github")
                  .defaultConfig(
                      BetaManagedAgentsMcpToolsetDefaultConfigParams.builder()
                          .permissionPolicy(BetaManagedAgentsAutoPolicy.builder().build())
                          .build()
                  )
                  .build()
          )
          .build()
  );
  ```

  ```php PHP
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolsetDefaultConfigParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsAlwaysAskPolicy;
  use Anthropic\Beta\Agents\BetaManagedAgentsAutoPolicy;
  use Anthropic\Beta\Agents\BetaManagedAgentsBashToolConfigParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsMCPToolsetDefaultConfigParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsMCPToolsetParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsURLMCPServerParams;

  $agent = $client->beta->agents->create(
      name: 'Ops Agent',
      model: 'claude-opus-5-5',
      mcpServers: [
          BetaManagedAgentsURLMCPServerParams::with(
              type: 'url',
              name: 'github',
              url: 'https://mcp.example.com/github',
          ),
      ],
      tools: [
          BetaManagedAgentsAgentToolset20260401Params::with(
              type: 'agent_toolset_20260401',
              defaultConfig: BetaManagedAgentsAgentToolsetDefaultConfigParams::with(
                  permissionPolicy: BetaManagedAgentsAutoPolicy::with(),
              ),
              configs: [
                  BetaManagedAgentsBashToolConfigParams::with(
                      permissionPolicy: BetaManagedAgentsAlwaysAskPolicy::with(type: 'always_ask'),
                  ),
              ],
          ),
          BetaManagedAgentsMCPToolsetParams::with(
              type: 'mcp_toolset',
              mcpServerName: 'github',
              defaultConfig: BetaManagedAgentsMCPToolsetDefaultConfigParams::with(
                  permissionPolicy: BetaManagedAgentsAutoPolicy::with(),
              ),
          ),
      ],
  );
  ```

  ```ruby Ruby
  agent = client.beta.agents.create(
    name: "Ops Agent",
    model: "claude-opus-5-5",
    mcp_servers: [
      {type: "url", name: "github", url: "https://mcp.example.com/github"}
    ],
    tools: [
      {
        type: "agent_toolset_20260401",
        default_config: {
          permission_policy: {type: "auto"}
        },
        configs: [
          {name: "bash", permission_policy: {type: "always_ask"}}
        ]
      },
      {
        type: "mcp_toolset",
        mcp_server_name: "github",
        default_config: {
          permission_policy: {type: "auto"}
        }
      }
    ]
  )
  ```
</CodeGroup>

您在 `user.message` 事件中发布的内容会被视为您的意图，这可能会使服务器允许原本会拒绝的调用。服务器不会从工具结果、获取的网页、MCP 服务器的响应或[会话线程](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration#tool-permissions-and-custom-tools)之间的消息中读取意图。服务器会评估这些内容，但不会从中接受指令。无论由谁发起请求，服务器都会将某些调用评估为高风险。如果您在 `user.message` 事件中转发不受信任的最终用户输入，服务器也会将该输入视为您的意图，从而可能使某个调用获得允许。对于您不希望该最终用户在未经审查的情况下运行的工具，请配置 `always_ask`。

<Warning>
  `auto` 不是人工检查点。如果服务器确定某个调用是安全的，该调用会在任何人看到之前运行，并且其影响可能无法撤销。如果某个工具的调用必须在运行前经过人工审查，请为该工具配置 `always_ask`。
</Warning>

## 查看每次调用的评估方式

在任何权限策略下，每个 `agent.tool_use` 和 `agent.mcp_tool_use` 事件都带有 `evaluated_permission`，即该调用权限检查的结果：`"allow"`、`"ask"` 或 `"deny"`。大多数事件还带有一个 `evaluation` 对象，其 `type` 指明产生该结果的策略。在 `auto` 下，该对象还会记录服务器的判定，并在结果为 `ask` 或 `deny` 时附带 `reason_code`。

例如，当 `bash` 处于 `auto` 下且服务器将某个调用评估为高风险时，被拒绝的调用在事件流中显示如下：

```json
{
  "type": "agent.tool_use",
  "id": "sevt_01pqr...",
  "name": "bash",
  "input": {
    "command": "rm -rf /workspace/reports"
  },
  "evaluated_permission": "deny",
  "evaluation": {
    "type": "auto",
    "evaluated_permission": {
      "type": "deny",
      "reason_code": "high_risk"
    }
  },
  "processed_at": "2026-03-25T14:05:12Z"
}
```

`evaluation` 对象采用下表中的某种形式。

| `evaluation`                                                                                | 顶层 `evaluated_permission` | 含义                                   |
| ------------------------------------------------------------------------------------------- | ------------------------- | ------------------------------------ |
| `{"type": "always_allow"}`                                                                  | `"allow"`                 | 解析后的策略为 `always_allow`，因此调用已运行。      |
| `{"type": "always_ask"}`                                                                    | `"ask"`                   | 解析后的策略为 `always_ask`，因此调用已暂停以等待您的批准。 |
| `{"type": "auto", "evaluated_permission": {"type": "allow"}}`                               | `"allow"`                 | 在 `auto` 下，服务器确定该调用是安全的，并已运行该调用。     |
| `{"type": "auto", "evaluated_permission": {"type": "ask", "reason_code": "indeterminate"}}` | `"ask"`                   | 在 `auto` 下，服务器未能做出判定，因此调用已暂停以等待您的批准。 |
| `{"type": "auto", "evaluated_permission": {"type": "deny", "reason_code": "high_risk"}}`    | `"deny"`                  | 在 `auto` 下，服务器将该调用评估为高风险并拒绝了它。       |

当 `evaluation.type` 为 `"auto"` 时，其嵌套的 `evaluated_permission.type` 与事件的顶层 `evaluated_permission` 相同，因此您可以从任一字段读取结果。`reason_code` 是供您的客户端进行分支判断并保存在审计记录中的值，而不是用于向最终用户显示的文本。

在两种情况下不存在 `evaluation`。当智能体指定了一个在会话中未启用的工具时，服务器会在不评估策略的情况下拒绝该调用：该事件带有 `evaluated_permission: "deny"`，但没有 `evaluation`。在引入 `evaluation` 之前记录的事件也不包含该字段：当 `evaluated_permission` 为 `"allow"` 时，请将其视为 `always_allow`；当其为 `"ask"` 时，请将其视为 `always_ask`。

请确保您的客户端能够容忍无法识别的 `evaluation.type` 或 `reason_code`。`agent.custom_tool_use` 事件不带这两个字段，因为权限策略不约束[自定义工具](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies#custom-tools)。

## 响应确认请求

在 `always_ask` 策略下，或在 `auto` 下服务器未能做出判定时，工具调用的评估结果为 `ask`。发生这种情况时：

1. 会话发出 `agent.tool_use` 或 `agent.mcp_tool_use` 事件。
2. 会话暂停并发出 `session.status_idle` 事件，其 `stop_reason.type` 为 `requires_action`。阻塞事件的 ID 位于 `stop_reason.event_ids` 数组中。会话会无限期等待响应。
3. 为每个阻塞事件发送一个 `user.tool_confirmation` 事件，并在 `tool_use_id` 参数中传递事件 ID。将 `result` 设置为 `"allow"` 或 `"deny"`。使用 `deny_message` 解释拒绝原因。您可以在单个 `events` 请求中发送多个确认。
4. 一旦所有阻塞事件都得到解决，会话将转换回 `running` 状态。被允许的工具会执行。被拒绝的工具不会运行，智能体会收到一个说明调用已被拒绝的工具结果，其中包含您的 `deny_message`。

如果您为 `evaluated_permission` 不是 `ask` 的事件发送 `user.tool_confirmation`，API 会以 400 错误拒绝该请求。这包括服务器在 `auto` 下拒绝的调用：您的客户端无法推翻这些拒绝。

如需以交互方式进行响应，请使用 `ant beta:sessions connect`，它会显示正在等待的调用，并在您允许或拒绝时发送此事件。请参阅[从终端连接到 Managed Agents 会话](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/sessions-connect#follow-and-steer-the-session)。

在以下示例中，工具使用事件 ID 来自 `session.status_idle` 事件的 `stop_reason.event_ids` 数组。请在[会话事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#integrating-events)指南中了解有关接收事件的更多信息，或[订阅 webhook](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks) 以便在会话暂停等待输入时收到通知。

<CodeGroup>
  ```bash cURL
  # 允许该工具执行
  curl -fsSL "https://api.anthropic.com/v1/sessions/$SESSION_ID/events" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "events": [
        {
          "type": "user.tool_confirmation",
          "tool_use_id": "'$AGENT_TOOL_USE_EVENT_ID'",
          "result": "allow"
        }
      ]
    }'

  # 或拒绝执行并附上说明
  curl -fsSL "https://api.anthropic.com/v1/sessions/$SESSION_ID/events" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "events": [
        {
          "type": "user.tool_confirmation",
          "tool_use_id": "'$MCP_TOOL_USE_EVENT_ID'",
          "result": "deny",
          "deny_message": "Don'\''t create issues in the production project. Use the staging project."
        }
      ]
    }'
  ```

  ```bash CLI
  # 允许工具执行
  ant beta:sessions:events send \
    --session-id "$SESSION_ID" \
    --event "{type: user.tool_confirmation, tool_use_id: $AGENT_TOOL_USE_EVENT_ID, result: allow}"

  # 或拒绝执行并给出说明
  ant beta:sessions:events send \
    --session-id "$SESSION_ID" \
    --event "{type: user.tool_confirmation, tool_use_id: $MCP_TOOL_USE_EVENT_ID, result: deny,
      deny_message: Don't create issues in the production project. Use the staging project.}"
  ```

  ```python Python
  # 允许工具执行
  client.beta.sessions.events.send(
      session.id,
      events=[
          {
              "type": "user.tool_confirmation",
              "tool_use_id": agent_tool_use_event.id,
              "result": "allow",
          },
      ],
  )

  # 或拒绝执行并给出说明
  client.beta.sessions.events.send(
      session.id,
      events=[
          {
              "type": "user.tool_confirmation",
              "tool_use_id": mcp_tool_use_event.id,
              "result": "deny",
              "deny_message": "Don't create issues in the production project. Use the staging project.",
          },
      ],
  )
  ```

  ```typescript TypeScript
  // 允许工具执行
  await client.beta.sessions.events.send(session.id, {
    events: [
      {
        type: "user.tool_confirmation",
        tool_use_id: agent_tool_use_event.id,
        result: "allow"
      }
    ]
  });

  // 或拒绝执行并给出说明
  await client.beta.sessions.events.send(session.id, {
    events: [
      {
        type: "user.tool_confirmation",
        tool_use_id: mcp_tool_use_event.id,
        result: "deny",
        deny_message: "Don't create issues in the production project. Use the staging project."
      }
    ]
  });
  ```

  ```csharp C#
  // 允许工具执行
  await client.Beta.Sessions.Events.Send(session.ID, new()
  {
      Events =
      [
          new BetaManagedAgentsUserToolConfirmationEventParams
          {
              Type = BetaManagedAgentsUserToolConfirmationEventParamsType.UserToolConfirmation,
              ToolUseID = agentToolUseEvent.ID,
              Result = "allow",
          },
      ],
  });

  // 或拒绝执行并给出说明
  await client.Beta.Sessions.Events.Send(session.ID, new()
  {
      Events =
      [
          new BetaManagedAgentsUserToolConfirmationEventParams
          {
              Type = BetaManagedAgentsUserToolConfirmationEventParamsType.UserToolConfirmation,
              ToolUseID = mcpToolUseEvent.ID,
              Result = "deny",
              DenyMessage = "Don't create issues in the production project. Use the staging project.",
          },
      ],
  });
  ```

  ```go Go
  // 允许该工具执行
  _, err = client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
  	Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
  		OfUserToolConfirmation: &anthropic.BetaManagedAgentsUserToolConfirmationEventParams{
  			Type:      anthropic.BetaManagedAgentsUserToolConfirmationEventParamsTypeUserToolConfirmation,
  			ToolUseID: agentToolUseEvent.ID,
  			Result:    anthropic.BetaManagedAgentsUserToolConfirmationEventParamsResultAllow,
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }

  // 或拒绝执行并给出说明
  _, err = client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
  	Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
  		OfUserToolConfirmation: &anthropic.BetaManagedAgentsUserToolConfirmationEventParams{
  			Type:        anthropic.BetaManagedAgentsUserToolConfirmationEventParamsTypeUserToolConfirmation,
  			ToolUseID:   mcpToolUseEvent.ID,
  			Result:      anthropic.BetaManagedAgentsUserToolConfirmationEventParamsResultDeny,
  			DenyMessage: anthropic.String("Don't create issues in the production project. Use the staging project."),
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  // 允许该工具执行
  client.beta().sessions().events().send(
      session.id(),
      EventSendParams.builder()
          .addEvent(
              BetaManagedAgentsUserToolConfirmationEventParams.builder()
                  .type(BetaManagedAgentsUserToolConfirmationEventParams.Type.USER_TOOL_CONFIRMATION)
                  .toolUseId(agentToolUseEvent.id())
                  .result(BetaManagedAgentsUserToolConfirmationEventParams.Result.ALLOW)
                  .build()
          )
          .build()
  );

  // 或拒绝执行并给出说明
  client.beta().sessions().events().send(
      session.id(),
      EventSendParams.builder()
          .addEvent(
              BetaManagedAgentsUserToolConfirmationEventParams.builder()
                  .type(BetaManagedAgentsUserToolConfirmationEventParams.Type.USER_TOOL_CONFIRMATION)
                  .toolUseId(mcpToolUseEvent.id())
                  .result(BetaManagedAgentsUserToolConfirmationEventParams.Result.DENY)
                  .denyMessage("Don't create issues in the production project. Use the staging project.")
                  .build()
          )
          .build()
  );
  ```

  ```php PHP
  use Anthropic\Beta\Sessions\Events\ManagedAgentsUserToolConfirmationEventParams;

  // 允许该工具执行
  $client->beta->sessions->events->send(
      $session->id,
      events: [
          ManagedAgentsUserToolConfirmationEventParams::with(
              type: 'user.tool_confirmation',
              toolUseID: $agentToolUseEvent->id,
              result: 'allow',
          ),
      ],
  );

  // 或拒绝执行并给出说明
  $client->beta->sessions->events->send(
      $session->id,
      events: [
          ManagedAgentsUserToolConfirmationEventParams::with(
              type: 'user.tool_confirmation',
              toolUseID: $mcpToolUseEvent->id,
              result: 'deny',
              denyMessage: "Don't create issues in the production project. Use the staging project.",
          ),
      ],
  );
  ```

  ```ruby Ruby
  # 允许工具执行
  client.beta.sessions.events.send_(
    session.id,
    events: [
      {
        type: "user.tool_confirmation",
        tool_use_id: agent_tool_use_event.id,
        result: "allow"
      }
    ]
  )

  # 或拒绝执行并给出说明
  client.beta.sessions.events.send_(
    session.id,
    events: [
      {
        type: "user.tool_confirmation",
        tool_use_id: mcp_tool_use_event.id,
        result: "deny",
        deny_message: "Don't create issues in the production project. Use the staging project."
      }
    ]
  )
  ```
</CodeGroup>

## 自定义工具

权限策略不适用于自定义工具。当智能体调用自定义工具时，您的应用程序会收到 `agent.custom_tool_use` 事件，并负责在发回 `user.custom_tool_result` 之前决定是否执行它。有关完整流程，请参阅[会话事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#handling-custom-tool-calls)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="技能" icon="books" href="https://platform.claude.com/docs/zh-CN/managed-agents/skills">
    为您的智能体附加可复用的、基于文件系统的专业知识，以支持特定领域的工作流程。
  </Card>

  <Card title="会话事件流" icon="lightning" href="https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming">
    发送事件、流式传输响应，并在执行过程中中断或重定向您的会话。
  </Card>
</CardGroup>
