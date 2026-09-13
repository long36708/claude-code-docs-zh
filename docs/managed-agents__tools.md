---
title: 工具
url: https://platform.claude.com/docs/zh-CN/managed-agents/tools
description: 配置您的智能体可用的工具。
---

Claude Managed Agents 提供了一组内置工具，Claude 可以在[会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)中自主使用这些工具。您可以通过在智能体配置中指定工具来控制哪些工具可用。

Claude Managed Agents 还支持自定义的、用户定义的工具。您的应用程序单独执行这些工具并将结果返回给 Claude，Claude 使用这些结果继续执行任务。若要为智能体提供来自 MCP 服务器的工具，请改用 [MCP 连接器](https://platform.claude.com/docs/zh-CN/managed-agents/mcp-connector)。

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

## 可用工具

智能体工具集包含以下工具。当您在智能体配置中包含该工具集时，所有工具默认启用。`configs` 数组中的每个条目通过其 `name` 标识，使用"名称"列中的值，并接受一个具有相同值的可选 `type` 字段。`web_search` 和 `web_fetch` 条目接受额外的设置；请参阅[限制网页搜索和网页抓取的域名](https://platform.claude.com/docs/zh-CN/managed-agents/tools#restrict-web-search-and-web-fetch-domains)。

| 工具         | 名称           | 描述                    |
| ---------- | ------------ | --------------------- |
| Bash       | `bash`       | 在 shell 会话中执行 bash 命令 |
| Read       | `read`       | 从沙箱文件系统读取文件           |
| Write      | `write`      | 向沙箱文件系统写入文件           |
| Edit       | `edit`       | 在文件中执行字符串替换           |
| Glob       | `glob`       | 使用 glob 模式进行快速文件模式匹配  |
| Grep       | `grep`       | 使用正则表达式模式进行文本搜索       |
| Web fetch  | `web_fetch`  | 从 URL 抓取内容            |
| Web search | `web_search` | 在网络上搜索信息              |

当工具输出超过 100,000 个字符（约 25,000 个令牌）时，它会自动写入[沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/environments)中的文件。模型会收到一个包含文件路径的截断预览，并可以从该路径读取完整内容。

## 配置工具集

创建智能体时，使用 `agent_toolset_20260401` 启用完整工具集。使用 `configs` 数组禁用特定工具或覆盖其设置。每个配置条目还可以设置一个 `permission_policy`，用于控制该工具的调用是自动批准还是需要确认。有关可用的策略类型，请参阅[权限策略](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies)。

`web_search` 和 `web_fetch` 的配置条目还接受域名过滤器和其他网页设置；请参阅[限制网页搜索和网页抓取的域名](https://platform.claude.com/docs/zh-CN/managed-agents/tools#restrict-web-search-and-web-fetch-domains)。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  agent=$(curl -fsSL https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d @- <<'EOF'
  {
    "name": "Coding Assistant",
    "model": "claude-opus-5",
    "tools": [
      {
        "type": "agent_toolset_20260401",
        "configs": [
          {"name": "web_fetch", "enabled": false}
        ]
      }
    ]
  }
  EOF
  )
  ```

  ```bash CLI
  ant beta:agents create <<'YAML'
  name: Coding Assistant
  model: claude-opus-5
  tools:
    - type: agent_toolset_20260401
      configs:
        - name: web_fetch
          enabled: false
  YAML
  ```

  ```python Python
  agent = client.beta.agents.create(
      name="Coding Assistant",
      model="claude-opus-5",
      tools=[
          {
              "type": "agent_toolset_20260401",
              "configs": [
                  {"name": "web_fetch", "enabled": False},
              ],
          },
      ],
  )
  ```

  ```typescript TypeScript
  const agent = await client.beta.agents.create({
    name: "Coding Assistant",
    model: "claude-opus-5",
    tools: [
      {
        type: "agent_toolset_20260401",
        configs: [{ name: "web_fetch", enabled: false }]
      }
    ]
  });
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Agents;

  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Coding Assistant",
      Model = new("claude-opus-5"),
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = "agent_toolset_20260401",
              Configs =
              [
                  new BetaManagedAgentsWebFetchToolConfigParams { Enabled = false },
              ],
          },
      ],
  });
  ```

  ```go Go
  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "Coding Assistant",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: "claude-opus-5",
  	},
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  		OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  			Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  			Configs: []anthropic.BetaManagedAgentsAgentToolConfigParamsUnion{{
  				OfWebFetch: &anthropic.BetaManagedAgentsWebFetchToolConfigParams{
  					Enabled: anthropic.Bool(false),
  				},
  			}},
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

  var agent = client.beta().agents().create(AgentCreateParams.builder()
      .name("Coding Assistant")
      .model(BetaManagedAgentsModel.CLAUDE_OPUS_5)
      .addTool(BetaManagedAgentsAgentToolset20260401Params.builder()
          .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
          .addConfig(BetaManagedAgentsWebFetchToolConfigParams.builder()
              .enabled(false)
              .build())
          .build())
      .build());
  ```

  ```php PHP
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;
  use Anthropic\Beta\Agents\BetaManagedAgentsWebFetchToolConfigParams;

  $agent = $client->beta->agents->create(
      name: 'Coding Assistant',
      model: 'claude-opus-5',
      tools: [
          BetaManagedAgentsAgentToolset20260401Params::with(
              type: 'agent_toolset_20260401',
              configs: [
                  BetaManagedAgentsWebFetchToolConfigParams::with(enabled: false),
              ],
          ),
      ],
  );
  ```

  ```ruby Ruby
  agent = client.beta.agents.create(
    name: "Coding Assistant",
    model: "claude-opus-5",
    tools: [
      {
        type: :agent_toolset_20260401,
        configs: [
          {name: :web_fetch, enabled: false}
        ]
      }
    ]
  )
  ```
</CodeGroup>

### 禁用特定工具

要禁用某个工具，请在智能体 `tools` 数组的工具集对象中，将该工具配置条目的 `enabled` 设置为 `false`（即 `enabled: false`）：

```json
{
  "type": "agent_toolset_20260401",
  "configs": [
    { "name": "web_fetch", "enabled": false },
    { "name": "web_search", "enabled": false }
  ]
}
```

### 仅启用特定工具

`default_config` 对象为工具集中的每个工具设置基线，而每个工具的 `configs` 条目会覆盖它。若要从全部关闭开始并仅启用您需要的工具，请将 `default_config.enabled` 设置为 `false`：

```json
{
  "type": "agent_toolset_20260401",
  "default_config": { "enabled": false },
  "configs": [
    { "name": "bash", "enabled": true },
    { "name": "read", "enabled": true },
    { "name": "write", "enabled": true }
  ]
}
```

### 限制网页搜索和网页抓取的域名

要控制智能体的网页工具可以访问哪些站点，请在工具集 `configs` 数组的 `web_search` 和 `web_fetch` 条目上设置 `allowed_domains`（工具只能访问这些主机）或 `blocked_domains`（工具永远无法访问这些主机）。每个工具都有自己的列表，因此 `web_search` 和 `web_fetch` 可以有不同的限制。列出的域名涵盖该主机及其所有子域名。在运行时，对其列表不允许的 URL 发起的 `web_fetch` 调用会向智能体返回一个错误结果（`agent.tool_result` 事件上的 `is_error: true`，其内容会指明错误代码 `url_not_allowed`），而 `web_search` 会省略其列表不允许的结果。

以下工具集将 `web_search` 限制为两个站点并对其结果进行本地化，同时为 `web_fetch` 屏蔽一个主机，并限制进入上下文的抓取内容量：

```json
{
  "type": "agent_toolset_20260401",
  "configs": [
    {
      "type": "web_search",
      "name": "web_search",
      "allowed_domains": ["docs.example.com", "arxiv.org"],
      "user_location": {
        "type": "approximate",
        "country": "US",
        "timezone": "America/Los_Angeles"
      }
    },
    {
      "type": "web_fetch",
      "name": "web_fetch",
      "blocked_domains": ["ads.example.com"],
      "max_content_tokens": 50000
    }
  ]
}
```

<Note>
  在 Python、TypeScript、Go、Java、C#、Ruby 和 PHP SDK 中，每个 `configs` 条目都按工具进行类型化：一个联合类型，每个内置工具对应一个成员，通过 `type` 进行区分。构造条目时 `type` 是可选的（服务器会从 `name` 推断它），而在响应中始终存在。这种类型化不会改变条目序列化后的 JSON，因此仅设置了 `name`、`enabled` 和 `permission_policy` 的条目的请求无论是否带有 `type` 都是有效的。在通过类型化值而非普通字典或哈希构造条目的 SDK（Go、Java、C# 和 PHP）中，`configs` 的元素类型就是该联合类型本身：请使用每个工具对应的成员类型来构建每个条目。
</Note>

以下请求使用此工具集创建一个智能体，并打印响应中的 `configs` 数组：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  agent=$(curl -fsSL https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d @- <<'EOF'
  {
    "name": "Research Agent",
    "model": "claude-opus-5",
    "tools": [
      {
        "type": "agent_toolset_20260401",
        "configs": [
          {
            "type": "web_search",
            "name": "web_search",
            "allowed_domains": ["docs.example.com", "arxiv.org"],
            "user_location": {
              "type": "approximate",
              "country": "US",
              "timezone": "America/Los_Angeles"
            }
          },
          {
            "type": "web_fetch",
            "name": "web_fetch",
            "blocked_domains": ["ads.example.com"],
            "max_content_tokens": 50000
          }
        ]
      }
    ]
  }
  EOF
  )
  jq '.tools[0].configs' <<< "$agent"
  ```

  ```bash CLI
  ant beta:agents create --transform tools.0.configs <<'YAML'
  name: Research Agent
  model: claude-opus-5
  tools:
    - type: agent_toolset_20260401
      configs:
        - type: web_search
          name: web_search
          allowed_domains: [docs.example.com, arxiv.org]
          user_location:
            type: approximate
            country: US
            timezone: America/Los_Angeles
        - type: web_fetch
          name: web_fetch
          blocked_domains: [ads.example.com]
          max_content_tokens: 50000
  YAML
  ```

  ```python Python
  client = Anthropic()

  agent = client.beta.agents.create(
      name="Research Agent",
      model="claude-opus-5",
      tools=[
          {
              "type": "agent_toolset_20260401",
              "configs": [
                  {
                      "name": "web_search",
                      "allowed_domains": ["docs.example.com", "arxiv.org"],
                      "user_location": {
                          "type": "approximate",
                          "country": "US",
                          "timezone": "America/Los_Angeles",
                      },
                  },
                  {
                      "name": "web_fetch",
                      "blocked_domains": ["ads.example.com"],
                      "max_content_tokens": 50_000,
                  },
              ],
          }
      ],
  )

  for tool in agent.tools:
      if tool.type == "agent_toolset_20260401":
          print(json.dumps([config.to_dict() for config in tool.configs], indent=2))
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const agent = await client.beta.agents.create({
    name: "Research Agent",
    model: "claude-opus-5",
    tools: [
      {
        type: "agent_toolset_20260401",
        configs: [
          {
            name: "web_search",
            allowed_domains: ["docs.example.com", "arxiv.org"],
            user_location: {
              type: "approximate",
              country: "US",
              timezone: "America/Los_Angeles"
            }
          },
          {
            name: "web_fetch",
            blocked_domains: ["ads.example.com"],
            max_content_tokens: 50_000
          }
        ]
      }
    ]
  });

  for (const tool of agent.tools) {
    if (tool.type === "agent_toolset_20260401") {
      console.log(JSON.stringify(tool.configs, null, 2));
    }
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Agents;

  AnthropicClient client = new();

  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Research Agent",
      Model = BetaManagedAgentsModel.ClaudeOpus5,
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = BetaManagedAgentsAgentToolset20260401ParamsType.AgentToolset20260401,
              Configs =
              [
                  new BetaManagedAgentsWebSearchToolConfigParams
                  {
                      AllowedDomains = ["docs.example.com", "arxiv.org"],
                      UserLocation = new()
                      {
                          Country = "US",
                          Timezone = "America/Los_Angeles",
                      },
                  },
                  new BetaManagedAgentsWebFetchToolConfigParams
                  {
                      BlockedDomains = ["ads.example.com"],
                      MaxContentTokens = 50_000,
                  },
              ],
          },
      ],
  });

  JsonSerializerOptions jsonOptions = new() { WriteIndented = true };
  foreach (var tool in agent.Tools)
  {
      if (tool.TryPickBetaManagedAgentsAgentToolset20260401(out var toolset))
      {
          Console.WriteLine(JsonSerializer.Serialize(toolset.Configs, jsonOptions));
      }
  }
  ```

  ```go Go
  client := anthropic.NewClient()
  ctx := context.Background()

  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "Research Agent",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: anthropic.BetaManagedAgentsModelClaudeOpus5,
  	},
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  		OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  			Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  			Configs: []anthropic.BetaManagedAgentsAgentToolConfigParamsUnion{
  				{OfWebSearch: &anthropic.BetaManagedAgentsWebSearchToolConfigParams{
  					AllowedDomains: []string{"docs.example.com", "arxiv.org"},
  					UserLocation: anthropic.BetaManagedAgentsUserLocationParam{
  						Country:  anthropic.String("US"),
  						Timezone: anthropic.String("America/Los_Angeles"),
  					},
  				}},
  				{OfWebFetch: &anthropic.BetaManagedAgentsWebFetchToolConfigParams{
  					BlockedDomains:   []string{"ads.example.com"},
  					MaxContentTokens: anthropic.Int(50000),
  				}},
  			},
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }

  for _, tool := range agent.Tools {
  	switch toolset := tool.AsAny().(type) {
  	case anthropic.BetaManagedAgentsAgentToolset20260401:
  		configs := make([]json.RawMessage, len(toolset.Configs))
  		for i, config := range toolset.Configs {
  			configs[i] = json.RawMessage(config.RawJSON())
  		}
  		output, err := json.MarshalIndent(configs, "", "  ")
  		if err != nil {
  			panic(err)
  		}
  		fmt.Println(string(output))
  	}
  }
  ```

  ```java Java
  import com.anthropic.models.beta.agents.AgentCreateParams;
  import com.anthropic.models.beta.agents.BetaManagedAgentsAgentToolset20260401Params;
  import com.anthropic.models.beta.agents.BetaManagedAgentsModel;
  import com.anthropic.models.beta.agents.BetaManagedAgentsUserLocation;
  import com.anthropic.models.beta.agents.BetaManagedAgentsWebFetchToolConfigParams;
  import com.anthropic.models.beta.agents.BetaManagedAgentsWebSearchToolConfigParams;

  void main() {
      var client = AnthropicOkHttpClient.fromEnv();

      var agent = client.beta().agents().create(AgentCreateParams.builder()
          .name("Research Agent")
          .model(BetaManagedAgentsModel.CLAUDE_OPUS_5)
          .addTool(BetaManagedAgentsAgentToolset20260401Params.builder()
              .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
              .addConfig(BetaManagedAgentsWebSearchToolConfigParams.builder()
                  .allowedDomains(List.of("docs.example.com", "arxiv.org"))
                  .userLocation(BetaManagedAgentsUserLocation.builder()
                      .country("US")
                      .timezone("America/Los_Angeles")
                      .build())
                  .build())
              .addConfig(BetaManagedAgentsWebFetchToolConfigParams.builder()
                  .blockedDomains(List.of("ads.example.com"))
                  .maxContentTokens(50_000)
                  .build())
              .build())
          .build());

      for (var tool : agent.tools()) {
          if (tool.isAgentToolset20260401()) {
              var configs = tool.asAgentToolset20260401().configs();
              IO.println(ObjectMappers.jsonMapper().valueToTree(configs));
          }
      }
  }
  ```

  ```php PHP
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401;
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;
  use Anthropic\Beta\Agents\BetaManagedAgentsUserLocation;
  use Anthropic\Beta\Agents\BetaManagedAgentsWebFetchToolConfigParams;
  use Anthropic\Beta\Agents\BetaManagedAgentsWebSearchToolConfigParams;
  // ...

  $client = new Client();

  $agent = $client->beta->agents->create(
      name: 'Research Agent',
      model: 'claude-opus-5',
      tools: [
          BetaManagedAgentsAgentToolset20260401Params::with(
              type: 'agent_toolset_20260401',
              configs: [
                  BetaManagedAgentsWebSearchToolConfigParams::with(
                      allowedDomains: ['docs.example.com', 'arxiv.org'],
                      userLocation: BetaManagedAgentsUserLocation::with(
                          country: 'US',
                          timezone: 'America/Los_Angeles',
                      ),
                  ),
                  BetaManagedAgentsWebFetchToolConfigParams::with(
                      blockedDomains: ['ads.example.com'],
                      maxContentTokens: 50_000,
                  ),
              ],
          ),
      ],
  );

  foreach ($agent->tools as $tool) {
      if ($tool instanceof BetaManagedAgentsAgentToolset20260401) {
          echo json_encode($tool->configs, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES), PHP_EOL;
      }
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  agent = client.beta.agents.create(
    name: "Research Agent",
    model: "claude-opus-5",
    tools: [
      {
        type: :agent_toolset_20260401,
        configs: [
          {
            name: :web_search,
            allowed_domains: ["docs.example.com", "arxiv.org"],
            user_location: {type: :approximate, country: "US", timezone: "America/Los_Angeles"}
          },
          {
            name: :web_fetch,
            blocked_domains: ["ads.example.com"],
            max_content_tokens: 50_000
          }
        ]
      }
    ]
  )

  case agent.tools.first
  in Anthropic::Models::Beta::BetaManagedAgentsAgentToolset20260401 => toolset
    puts JSON.pretty_generate(toolset.configs.map(&:to_h))
  end
  ```
</CodeGroup>

在 Claude Console 中，可在智能体表单的 **Built-in tools** 卡片的 `web_search` 和 `web_fetch` 行中设置允许或屏蔽的域名；在智能体配置的 **Raw** 视图中设置 `max_content_tokens` 和 `user_location`。

除了 `enabled` 和 `permission_policy` 之外，网页工具条目还接受以下设置：

| 设置                   | 适用于                      | 描述                                                                                                                                                        |
| -------------------- | ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `allowed_domains`    | `web_search`、`web_fetch` | 工具唯一可以访问的主机。不能与同一条目上的 `blocked_domains` 组合使用。                                                                                                             |
| `blocked_domains`    | `web_search`、`web_fetch` | 工具无法访问的主机。                                                                                                                                                |
| `max_content_tokens` | `web_fetch`              | 限制包含在上下文中的抓取页面内容量。必须为正整数。请参阅[内容限制](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool#content-limits)。                       |
| `user_location`      | `web_search`             | 对搜索结果进行本地化。一个与 Messages API [`user_location`](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool#localization) 参数具有相同字段的对象。 |

<Note>
  环境的 [`networking`](https://platform.claude.com/docs/zh-CN/managed-agents/environments#networking) 设置控制沙箱自身的出站流量。它们不影响 `web_search` 或 `web_fetch`，无论环境是云端沙箱还是自托管沙箱，这两个工具都在 Anthropic 的服务器上运行。每个工具的 `allowed_domains` 和 `blocked_domains` 列表是限制这些工具可访问范围的方式。
</Note>

<Note>
  Claude Console 中组织级别的网页搜索和网页抓取设置适用于 Messages API，不适用于 Managed Agents 会话。要限制智能体的网页工具，请改为在其工具集上配置 `allowed_domains` 或 `blocked_domains`。
</Note>

#### 域名列表规则

* 在一个条目上设置 `allowed_domains` 或 `blocked_domains` 之一，不能同时设置两者。同时设置两者的条目会被拒绝。
* 每个列表包含 1 到 64 个域名，每个域名 1 到 255 个字符。空列表会被拒绝：若不施加任何限制，请省略该字段或发送 `null`。
* 每个域名是一个可注册域名或其子域名，以纯主机名形式书写：ASCII 字母、数字、连字符、下划线和点，不含协议、端口、凭据、通配符或空白字符，不含以连字符开头或结尾的标签，并且除本列表后文所述的可选 `web_search` 路径后缀外不含路径。请使用 `example.com`，而不是 `https://example.com`、`example.com:443` 或 `*.example.com`。主机名比较时不区分大小写，并且会忽略单个尾随的 `/`。
* 列出的域名匹配该主机及其子域名：`example.com` 涵盖 `docs.example.com`，但 `docs.example.com` 不涵盖 `example.com` 或 `api.example.com`。前导的 `www.` 与其他子域名一样是子域名，因此 `www.example.com` 不涵盖 `example.com`；请列出裸域名以同时涵盖两者。
* 不接受任何形式的 IP 地址，无论是 IPv4、IPv6、带方括号的形式，还是诸如 `127.1` 之类的数字简写。请改为列出站点的域名。
* 裸顶级域名或注册后缀（如 `com`、`co.uk` 或 `gov.uk`）会被拒绝，单标签名称（如 `intranet`）也会被拒绝。请列出完整域名，如 `example.co.uk`。
* `localhost` 以及以 `.localhost`、`.local`、`.internal`、`.localdomain` 或 `.invalid` 结尾的主机会被拒绝。
* 对于国际化域名，请使用 `xn--`（Punycode）形式；包含非 ASCII 字符的域名会被拒绝。
* `web_fetch` 域名不能包含路径：请使用 `example.com`，而不是 `example.com/*`。`web_search` 域名可以带有路径后缀，如 `example.com/blog`，其中路径不能包含空格、`?`、`#` 或 `$ , | ^ !` 中的任何字符。对于 `web_search` 也建议优先使用纯主机名，因为搜索提供商将路径后缀作为 URL 模式而非严格的主机规则进行匹配。
* 列表中的重复域名会被拒绝。`www.example.com` 和 `example.com` 被视为不同的域名；有关各自涵盖的范围，请参阅前面的匹配规则。

#### 设置何时被验证

当您[创建智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#create-an-agent)或[更新智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup#update-an-agent)时，以及当您创建或更新提供了 `tools` 的会话时，格式和限制违规会以 400 `invalid_request_error` 被拒绝。例如，同时设置两个列表的条目的消息包含 `Only one of allowed_domains or blocked_domains may be set.`，空列表的消息包含 `allowed_domains: Empty list of domains is ambiguous. Provide at least one domain or null.`。违反格式规则的域名的消息会指明其所在列表和从零开始的位置，例如 `allowed_domains.0: IP addresses are not supported; provide a plain hostname like "example.com"`。

同样的请求还会拒绝三个依赖于搜索和抓取提供商的设置：`allowed_domains` 中 Anthropic 的爬虫不被允许访问的域名、搜索提供商不支持的 `user_location.country`（消息以 `user_location.country: not a country the search provider supports` 结尾），以及不是有效 IANA 名称的 `user_location.timezone`。会话在首次初始化工具时会再次检查配置；如果先前被接受的设置在那时不再有效，会话会发出一个 [`session.error`](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming) 事件并返回 `idle` 状态而不重试。请通过[更新会话的工具](https://platform.claude.com/docs/zh-CN/managed-agents/session-operations#updating-the-agent-configuration)来修复该设置，同时也更新智能体以便新会话以修正后的配置启动，然后发送新的 `user.message` 以继续。

#### 多智能体会话、结果和会话中途更新

在[多智能体会话](https://platform.claude.com/docs/zh-CN/managed-agents/multiagent-orchestration)中，适用于某个线程的每个域名列表都会同时生效：协调者名册中的智能体受其自身的 `allowed_domains` 和 `blocked_domains`、调用它的任何智能体的列表以及协调者当前列表的约束。

* 允许列表合并为所有列表共同涵盖的域名，屏蔽列表则相加，因此名册智能体可以缩小工具可访问的范围，但永远不能扩大。例如，设置了 `blocked_domains` 的名册智能体会保留协调者的 `allowed_domains` 并在其中屏蔽这些主机，而设置了自己的 `allowed_domains` 的名册智能体只能访问其列表和协调者列表共同涵盖的主机。
* 如果合并后的允许列表没有共同的域名，该工具对该智能体仍然可用，但每次调用都会失败并返回 `url_not_allowed` 错误，说明没有允许的域名，并且工具描述会将此告知模型。请将每个名册智能体的允许列表保持在协调者的允许列表之内以避免这种情况。
* `max_content_tokens` 和 `user_location` 不会合并：线程使用其自身工具配置中的值（如果已设置），否则使用调用它的智能体的值，否则使用协调者当前配置中的值。
* `{"type": "self"}` 名册条目没有自己的网页设置，并遵循协调者的当前设置。
* [结果驱动会话](https://platform.claude.com/docs/zh-CN/managed-agents/define-outcomes)中的评分器在没有 `web_search` 和 `web_fetch` 的情况下运行，无论这些设置如何。
* 您可以通过[更新其工具](https://platform.claude.com/docs/zh-CN/managed-agents/session-operations#updating-the-agent-configuration)来更改空闲会话上的列表。新列表适用于会话的剩余部分；在多智能体会话中，每个线程从其下一轮开始应用它们，而名册智能体自身的列表保持为会话创建时其智能体定义所设置的状态。

#### 与 Messages API 工具的差异

这些设置使用与 Messages API 服务器工具上的[域名过滤](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/server-tools#domain-filtering)相同的 `allowed_domains` 和 `blocked_domains` 词汇，在 Managed Agents 上有以下差异：

* 每个列表上限为 64 个域名。
* 为 `web_fetch` 列出的域名不能包含路径。
* 域名必须为 ASCII：对于国际化域名，请使用 `xn--`（Punycode）形式。Messages API 接受 Unicode 条目，尽管它不建议这样做。
* `max_uses`、`citations` 和 `cache_control` 在工具集上不可用。

## 自定义工具

除了内置工具之外，您还可以定义自定义工具。自定义工具类似于 Messages API 中的[用户定义的客户端工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/how-tool-use-works#user-defined-tools-client-executed)。

每个自定义工具定义一个契约：您指定哪些操作可用以及它们返回什么，Claude 决定何时以及如何调用它们。模型从不自行执行任何操作。它发出一个结构化请求，您的代码运行该操作，结果流回对话中。有关如何在会话期间接收自定义工具调用并返回结果，请参阅[会话事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#handling-custom-tool-calls)。

如果您的会话在自托管沙箱中运行，环境工作进程可以[从您的沙箱提供自定义工具](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#serve-custom-tools-from-your-sandbox)，包括在您的网络内部封装 MCP 服务器的工具。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  agent=$(curl -fsSL https://api.anthropic.com/v1/agents \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d @- <<'EOF'
  {
    "name": "Weather Agent",
    "model": "claude-opus-5",
    "tools": [
      {
        "type": "agent_toolset_20260401"
      },
      {
        "type": "custom",
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
          "type": "object",
          "properties": {
            "location": {"type": "string", "description": "City name"}
          },
          "required": ["location"]
        }
      }
    ]
  }
  EOF
  )
  ```

  <MultiFileExample language="cli" label="CLI">
    ```bash CLI
    ant beta:agents create < agent.yaml
    ```

    <File filename="agent.yaml">
      ```yaml
      name: Weather Agent
      model: claude-opus-5
      tools:
        - type: agent_toolset_20260401
        - type: custom
          name: get_weather
          description: Get current weather for a location
          input_schema:
            type: object
            properties:
              location:
                type: string
                description: City name
            required:
              - location
      ```
    </File>
  </MultiFileExample>

  ```python Python
  agent = client.beta.agents.create(
      name="Weather Agent",
      model="claude-opus-5",
      tools=[
          {
              "type": "agent_toolset_20260401",
          },
          {
              "type": "custom",
              "name": "get_weather",
              "description": "Get current weather for a location",
              "input_schema": {
                  "type": "object",
                  "properties": {
                      "location": {"type": "string", "description": "City name"},
                  },
                  "required": ["location"],
              },
          },
      ],
  )
  ```

  ```typescript TypeScript
  const agent = await client.beta.agents.create({
    name: "Weather Agent",
    model: "claude-opus-5",
    tools: [
      { type: "agent_toolset_20260401" },
      {
        type: "custom",
        name: "get_weather",
        description: "Get current weather for a location",
        input_schema: {
          type: "object",
          properties: { location: { type: "string", description: "City name" } },
          required: ["location"]
        }
      }
    ]
  });
  ```

  ```csharp C#
  using System.Text.Json;
  using Anthropic.Models.Beta.Agents;

  var agent = await client.Beta.Agents.Create(new()
  {
      Name = "Weather Agent",
      Model = new("claude-opus-5"),
      Tools =
      [
          new BetaManagedAgentsAgentToolset20260401Params
          {
              Type = "agent_toolset_20260401",
          },
          new BetaManagedAgentsCustomToolParams
          {
              Type = "custom",
              Name = "get_weather",
              Description = "Get current weather for a location",
              InputSchema = new()
              {
                  Properties = new Dictionary<string, JsonElement>
                  {
                      ["location"] = JsonSerializer.SerializeToElement(
                          new { type = "string", description = "City name" }
                      ),
                  },
                  Required = ["location"],
              },
          },
      ],
  });
  ```

  ```go Go
  agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
  	Name: "Weather Agent",
  	Model: anthropic.BetaManagedAgentsModelConfigParams{
  		ID: "claude-opus-5",
  	},
  	Tools: []anthropic.BetaAgentNewParamsToolUnion{{
  		OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
  			Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
  		},
  	}, {
  		OfCustom: &anthropic.BetaManagedAgentsCustomToolParams{
  			Type:        anthropic.BetaManagedAgentsCustomToolParamsTypeCustom,
  			Name:        "get_weather",
  			Description: "Get current weather for a location",
  			InputSchema: anthropic.BetaManagedAgentsCustomToolInputSchemaParam{
  				Properties: map[string]any{
  					"location": map[string]any{
  						"type":        "string",
  						"description": "City name",
  					},
  				},
  				Required: []string{"location"},
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
  import java.util.Map;

  var agent = client.beta().agents().create(AgentCreateParams.builder()
      .name("Weather Agent")
      .model(BetaManagedAgentsModel.CLAUDE_OPUS_5)
      .addTool(BetaManagedAgentsAgentToolset20260401Params.builder()
          .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
          .build())
      .addTool(BetaManagedAgentsCustomToolParams.builder()
          .type(BetaManagedAgentsCustomToolParams.Type.CUSTOM)
          .name("get_weather")
          .description("Get current weather for a location")
          .inputSchema(BetaManagedAgentsCustomToolInputSchema.builder()
              .properties(BetaManagedAgentsCustomToolInputSchema.Properties.builder()
                  .putAdditionalProperty("location", JsonValue.from(Map.of(
                      "type", "string",
                      "description", "City name")))
                  .build())
              .addRequired("location")
              .build())
          .build())
      .build());
  ```

  ```php PHP
  use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;
  use Anthropic\Beta\Agents\BetaManagedAgentsCustomToolInputSchema;
  use Anthropic\Beta\Agents\BetaManagedAgentsCustomToolParams;

  $agent = $client->beta->agents->create(
      name: 'Weather Agent',
      model: 'claude-opus-5',
      tools: [
          BetaManagedAgentsAgentToolset20260401Params::with(
              type: 'agent_toolset_20260401',
          ),
          BetaManagedAgentsCustomToolParams::with(
              type: 'custom',
              name: 'get_weather',
              description: 'Get current weather for a location',
              inputSchema: BetaManagedAgentsCustomToolInputSchema::with(
                  properties: ['location' => ['type' => 'string', 'description' => 'City name']],
                  required: ['location'],
              ),
          ),
      ],
  );
  ```

  ```ruby Ruby
  agent = client.beta.agents.create(
    name: "Weather Agent",
    model: "claude-opus-5",
    tools: [
      {type: :agent_toolset_20260401},
      {
        type: :custom,
        name: "get_weather",
        description: "Get current weather for a location",
        input_schema: {
          type: :object,
          properties: {location: {type: "string", description: "City name"}},
          required: ["location"]
        }
      }
    ]
  )
  ```
</CodeGroup>

在智能体上定义自定义工具后，智能体会在会话期间调用它们。

### 自定义工具定义的最佳实践

* **提供极其详细的描述。** 这是迄今为止影响工具性能的最重要因素。您的描述应解释工具的作用以及何时使用它（以及何时不使用）。解释每个参数的含义以及它如何影响工具的行为。指出任何重要的注意事项或限制。您能为 Claude 提供的关于工具的上下文越多，它就越能准确判断何时以及如何使用它们。每个工具描述的目标是三到四句话，如果工具复杂则更多。
* **将相关操作合并为更少的工具。** 与其为每个动作创建单独的工具（`create_pr`、`review_pr`、`merge_pr`），不如将它们组合为一个带有 `action` 参数的工具。更少、功能更强的工具可以减少选择歧义，并使 Claude 更容易浏览您的工具集合。
* **在工具名称中使用有意义的命名空间。** 当您的工具跨越多个服务或资源时，请以资源作为名称前缀（例如 `db_query` 或 `storage_read`）。随着您的工具库增长，这可以使工具选择毫无歧义。
* **设计工具响应以仅返回高信号信息。** 返回语义化、稳定的标识符（例如 slug 或 UUID），而不是不透明的内部引用，并且仅包含 Claude 确定下一步所需的字段。臃肿的响应会浪费上下文，并使 Claude 更难提取重要内容。

## 后续步骤

<CardGroup cols={2}>
  <Card title="MCP 连接器" icon="link" href="https://platform.claude.com/docs/zh-CN/managed-agents/mcp-connector">
    将 MCP 服务器连接到您的智能体，以访问外部工具和数据源。
  </Card>

  <Card title="权限策略" icon="lock" href="https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies">
    控制智能体和 MCP 工具何时执行。
  </Card>

  <Card title="会话事件流" icon="lightning" href="https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming">
    发送事件、流式传输响应，并在执行中途中断或重定向您的会话。
  </Card>
</CardGroup>
