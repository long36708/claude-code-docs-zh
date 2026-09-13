---
title: Claude Managed Agents 快速入门
url: https://platform.claude.com/docs/zh-CN/managed-agents/quickstart
description: 创建您的第一个自主智能体。
---

本指南将引导您完成创建智能体、设置环境、启动会话以及流式传输智能体响应的全过程。

<Tip>
  **更喜欢交互式引导？** 在最新版本的 [Claude Code](https://claude.com/product/claude-code) 中运行 `/claude-api managed-agents-onboard`，即可获得引导式设置和交互式问答。
</Tip>

## 核心概念

| 概念                  | 描述                                           |
| ------------------- | -------------------------------------------- |
| **Agent（智能体）**      | 模型、系统提示、工具、MCP 服务器和技能                        |
| **Environment（环境）** | 会话运行位置的配置：Anthropic 托管的云沙箱，或在您自己的基础设施上自托管的沙箱 |
| **Session（会话）**     | 在环境中运行的智能体实例，执行特定任务并生成输出                     |
| **Events（事件）**      | 您的应用程序与智能体之间交换的消息（用户轮次、工具结果、状态更新）            |

## 前提条件

* 一个 [Claude Console 账户](https://platform.claude.com)
* 一个 [API 密钥](https://platform.claude.com/settings/keys)

## 安装 CLI

<Tabs>
  <Tab title="Homebrew (macOS)">
    ```bash
    brew install anthropics/tap/ant
    ```
  </Tab>

  <Tab title="curl (Linux/WSL)">
    对于 Linux 环境，请直接下载发布的二进制文件。

    ```bash
    VERSION=1.27.0
    OS=$(uname -s | tr '[:upper:]' '[:lower:]')
    case $(uname -m) in
      x86_64) ARCH=amd64 ;;
      aarch64) ARCH=arm64 ;;
    esac
    curl -fsSL "https://github.com/anthropics/anthropic-cli/releases/download/v${VERSION}/ant_${VERSION}_${OS}_${ARCH}.tar.gz" \
      | sudo tar -xz -C /usr/local/bin ant
    ```

    您可以在 [GitHub 发布页面](https://github.com/anthropics/anthropic-cli/releases)上找到所有发布版本。
  </Tab>

  <Tab title="Go">
    您也可以使用 `go install` 从源代码安装 CLI。需要 Go 1.25 或更高版本。

    ```bash
    go install github.com/anthropics/anthropic-cli/cmd/ant@latest
    ```

    二进制文件会被放置在 `$(go env GOPATH)/bin` 中。如果该目录尚未加入您的 `PATH`，请将其添加进去：

    ```bash
    export PATH="$PATH:$(go env GOPATH)/bin"
    ```
  </Tab>
</Tabs>

检查安装：

```bash
ant --version
```

## 安装 SDK

<Tabs>
  <Tab title="Python">
    ```bash
    pip install anthropic
    ```
  </Tab>

  <Tab title="TypeScript">
    ```bash
    npm install @anthropic-ai/sdk
    ```
  </Tab>

  <Tab title="Java">
    ```groovy Gradle
    implementation("com.anthropic:anthropic-java:2.58.0")
    ```
  </Tab>

  <Tab title="Go">
    ```bash
    go get github.com/anthropics/anthropic-sdk-go
    ```
  </Tab>

  <Tab title="C#">
    ```bash
    dotnet add package Anthropic
    ```
  </Tab>

  <Tab title="Ruby">
    ```bash
    bundle add anthropic
    ```
  </Tab>

  <Tab title="PHP">
    ```bash
    composer require "anthropic-ai/sdk" "guzzlehttp/guzzle:^7"
    ```
  </Tab>
</Tabs>

将您的 API 密钥设置为环境变量：

```bash
export ANTHROPIC_API_KEY="your-api-key-here"
```

## 创建您的第一个会话

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

<Steps>
  <Step title="创建智能体">
    创建一个智能体，用于定义模型、"system prompt"（系统提示）和可用工具。

    <CodeGroup defaultLanguage="CLI">
      ```bash cURL
      set -euo pipefail

      agent=$(
        curl -sS --fail-with-body https://api.anthropic.com/v1/agents \
          -H "x-api-key: $ANTHROPIC_API_KEY" \
          -H "anthropic-version: 2023-06-01" \
          -H "anthropic-beta: managed-agents-2026-04-01" \
          -H "content-type: application/json" \
          -d @- <<'EOF'
      {
        "name": "Coding Assistant",
        "model": "claude-opus-5",
        "system": "You are a helpful coding assistant. Write clean, well-documented code.",
        "tools": [
          {"type": "agent_toolset_20260401"}
        ]
      }
      EOF
      )

      AGENT_ID=$(jq -er '.id' <<<"$agent")
      AGENT_VERSION=$(jq -er '.version' <<<"$agent")

      echo "Agent ID: $AGENT_ID, version: $AGENT_VERSION"
      ```

      <MultiFileExample language="cli" label="CLI">
        ```bash CLI
        AGENT_ID=$(ant beta:agents create --transform id --raw-output < coding-assistant.agent.yaml)

        echo "Agent ID: $AGENT_ID"
        ```

        <File filename="coding-assistant.agent.yaml">
          ```yaml
          name: Coding Assistant
          model:
            id: claude-opus-5
          system: You are a helpful coding assistant. Write clean, well-documented code.
          tools:
            - type: agent_toolset_20260401
          ```
        </File>
      </MultiFileExample>

      ```python Python
      from anthropic import Anthropic

      client = Anthropic()

      agent = client.beta.agents.create(
          name="Coding Assistant",
          model="claude-opus-5",
          system="You are a helpful coding assistant. Write clean, well-documented code.",
          tools=[
              {"type": "agent_toolset_20260401"},
          ],
      )

      print(f"Agent ID: {agent.id}, version: {agent.version}")
      ```

      ```typescript TypeScript
      import Anthropic from "@anthropic-ai/sdk";

      const client = new Anthropic();

      const agent = await client.beta.agents.create({
        name: "Coding Assistant",
        model: "claude-opus-5",
        system: "You are a helpful coding assistant. Write clean, well-documented code.",
        tools: [
          { type: "agent_toolset_20260401" },
        ],
      });

      console.log(`Agent ID: ${agent.id}, version: ${agent.version}`);
      ```

      ```csharp C#
      using Anthropic;
      using Anthropic.Models.Beta.Agents;
      using Anthropic.Models.Beta.Environments;
      using Anthropic.Models.Beta.Sessions;
      using Anthropic.Models.Beta.Sessions.Events;

      var client = new AnthropicClient();

      var agent = await client.Beta.Agents.Create(new()
      {
          Name = "Coding Assistant",
          Model = BetaManagedAgentsModel.ClaudeOpus5,
          System = "You are a helpful coding assistant. Write clean, well-documented code.",
          Tools =
          [
              new BetaManagedAgentsAgentToolset20260401Params
              {
                  Type = "agent_toolset_20260401",
              },
          ],
      });

      Console.WriteLine($"Agent ID: {agent.ID}, version: {agent.Version}");
      ```

      ```go Go
      package main

      import (
      	"context"
      	"fmt"

      	"github.com/anthropics/anthropic-sdk-go"
      )

      func main() {
      	client := anthropic.NewClient()
      	ctx := context.Background()

      	agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
      		Name: "Coding Assistant",
      		Model: anthropic.BetaManagedAgentsModelConfigParams{
      			ID: anthropic.BetaManagedAgentsModelClaudeOpus5,
      		},
      		System: anthropic.String("You are a helpful coding assistant. Write clean, well-documented code."),
      		Tools: []anthropic.BetaAgentNewParamsToolUnion{{
      			OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
      				Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
      			},
      		}},
      	})
      	if err != nil {
      		panic(err)
      	}

      	fmt.Printf("Agent ID: %s, version: %d\n", agent.ID, agent.Version)
      ```

      ```java Java
      import com.anthropic.client.okhttp.AnthropicOkHttpClient;
      import com.anthropic.models.beta.agents.AgentCreateParams;
      import com.anthropic.models.beta.agents.BetaManagedAgentsAgentToolset20260401Params;
      import com.anthropic.models.beta.agents.BetaManagedAgentsModel;
      import com.anthropic.models.beta.environments.BetaCloudConfigParams;
      import com.anthropic.models.beta.environments.BetaUnrestrictedNetwork;
      import com.anthropic.models.beta.environments.EnvironmentCreateParams;
      import com.anthropic.models.beta.sessions.SessionCreateParams;
      import com.anthropic.models.beta.sessions.events.BetaManagedAgentsStreamSessionEvents;
      import com.anthropic.models.beta.sessions.events.BetaManagedAgentsUserMessageEventParams;
      import com.anthropic.models.beta.sessions.events.EventSendParams;

      void main() {
          var client = AnthropicOkHttpClient.fromEnv();

          var agent = client.beta().agents().create(AgentCreateParams.builder()
              .name("Coding Assistant")
              .model(BetaManagedAgentsModel.CLAUDE_OPUS_5)
              .system("You are a helpful coding assistant. Write clean, well-documented code.")
              .addTool(BetaManagedAgentsAgentToolset20260401Params.builder()
                  .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
                  .build())
              .build());

          IO.println("Agent ID: " + agent.id() + ", version: " + agent.version());
      ```

      ```php PHP
      use Anthropic\Client;

      $client = new Client();

      $agent = $client->beta->agents->create(
          name: 'Coding Assistant',
          model: 'claude-opus-5',
          system: 'You are a helpful coding assistant. Write clean, well-documented code.',
          tools: [
              ['type' => 'agent_toolset_20260401'],
          ],
      );

      echo "Agent ID: {$agent->id}, version: {$agent->version}\n";
      ```

      ```ruby Ruby
      require "anthropic"

      client = Anthropic::Client.new

      agent = client.beta.agents.create(
        name: "Coding Assistant",
        model: "claude-opus-5",
        system_: "You are a helpful coding assistant. Write clean, well-documented code.",
        tools: [{type: "agent_toolset_20260401"}]
      )

      puts "Agent ID: #{agent.id}, version: #{agent.version}"
      ```
    </CodeGroup>

    `agent_toolset_20260401` 工具类型会启用全套预构建的智能体工具（bash、文件操作、网页搜索等）。请参阅[工具](https://platform.claude.com/docs/zh-CN/managed-agents/tools)了解完整列表及每个工具的配置选项。

    保存返回的 `agent.id`。您将在创建的每个会话中引用它。
  </Step>

  <Step title="创建环境">
    环境定义了智能体运行所在的沙箱。

    <CodeGroup defaultLanguage="CLI">
      ```bash cURL
      environment=$(
        curl -sS --fail-with-body https://api.anthropic.com/v1/environments \
          -H "x-api-key: $ANTHROPIC_API_KEY" \
          -H "anthropic-version: 2023-06-01" \
          -H "anthropic-beta: managed-agents-2026-04-01" \
          -H "content-type: application/json" \
          -d @- <<'EOF'
      {
        "name": "quickstart-env",
        "config": {
          "type": "cloud",
          "networking": {"type": "unrestricted"}
        }
      }
      EOF
      )

      ENVIRONMENT_ID=$(jq -er '.id' <<<"$environment")

      echo "Environment ID: $ENVIRONMENT_ID"
      ```

      <MultiFileExample language="cli" label="CLI">
        ```bash CLI
        ENVIRONMENT_ID=$(ant beta:environments create --transform id --raw-output < quickstart.environment.yaml)

        echo "Environment ID: $ENVIRONMENT_ID"
        ```

        <File filename="quickstart.environment.yaml">
          ```yaml
          name: quickstart-env
          config:
            type: cloud
            networking:
              type: unrestricted
          ```
        </File>
      </MultiFileExample>

      ```python Python
      environment = client.beta.environments.create(
          name="quickstart-env",
          config={
              "type": "cloud",
              "networking": {"type": "unrestricted"},
          },
      )

      print(f"Environment ID: {environment.id}")
      ```

      ```typescript TypeScript
      const environment = await client.beta.environments.create({
        name: "quickstart-env",
        config: {
          type: "cloud",
          networking: { type: "unrestricted" },
        },
      });

      console.log(`Environment ID: ${environment.id}`);
      ```

      ```csharp C#
      var environment = await client.Beta.Environments.Create(new()
      {
          Name = "quickstart-env",
          Config = new BetaCloudConfigParams { Networking = new BetaUnrestrictedNetwork() },
      });

      Console.WriteLine($"Environment ID: {environment.ID}");
      ```

      ```go Go
      environment, err := client.Beta.Environments.New(ctx, anthropic.BetaEnvironmentNewParams{
      	Name: "quickstart-env",
      	Config: anthropic.BetaEnvironmentNewParamsConfigUnion{
      		OfCloud: &anthropic.BetaCloudConfigParams{
      			Networking: anthropic.BetaCloudConfigParamsNetworkingUnion{
      				OfUnrestricted: &anthropic.BetaUnrestrictedNetworkParam{},
      			},
      		},
      	},
      })
      if err != nil {
      	panic(err)
      }

      fmt.Printf("Environment ID: %s\n", environment.ID)
      ```

      ```java Java
      var environment = client.beta().environments().create(EnvironmentCreateParams.builder()
          .name("quickstart-env")
          .config(BetaCloudConfigParams.builder()
              .networking(BetaUnrestrictedNetwork.builder().build())
              .build())
          .build());

      IO.println("Environment ID: " + environment.id());
      ```

      ```php PHP
      $environment = $client->beta->environments->create(
          name: 'quickstart-env',
          config: ['type' => 'cloud', 'networking' => ['type' => 'unrestricted']],
      );

      echo "Environment ID: {$environment->id}\n";
      ```

      ```ruby Ruby
      environment = client.beta.environments.create(
        name: "quickstart-env",
        config: {type: "cloud", networking: {type: "unrestricted"}}
      )

      puts "Environment ID: #{environment.id}"
      ```
    </CodeGroup>

    保存返回的 `environment.id`。您将在创建的每个会话中引用它。

    <Tip>
      如需在您自己的基础设施上而非云端沙箱中运行沙箱，请参阅

      [自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)

      。
    </Tip>
  </Step>

  <Step title="启动会话">
    创建一个引用您的智能体和环境的会话。

    <CodeGroup>
      ```bash cURL
      session=$(
        curl -sS --fail-with-body https://api.anthropic.com/v1/sessions \
          -H "x-api-key: $ANTHROPIC_API_KEY" \
          -H "anthropic-version: 2023-06-01" \
          -H "anthropic-beta: managed-agents-2026-04-01" \
          -H "content-type: application/json" \
          -d @- <<EOF
      {
        "agent": "$AGENT_ID",
        "environment_id": "$ENVIRONMENT_ID",
        "title": "Quickstart session"
      }
      EOF
      )

      SESSION_ID=$(jq -er '.id' <<<"$session")

      echo "Session ID: $SESSION_ID"
      ```

      ```bash CLI
      SESSION_ID=$(ant beta:sessions create \
        --agent "$AGENT_ID" \
        --environment-id "$ENVIRONMENT_ID" \
        --title "Quickstart session" \
        --transform id --raw-output)

      echo "Session ID: $SESSION_ID"
      ```

      ```python Python
      session = client.beta.sessions.create(
          agent=agent.id,
          environment_id=environment.id,
          title="Quickstart session",
      )

      print(f"Session ID: {session.id}")
      ```

      ```typescript TypeScript
      const session = await client.beta.sessions.create({
        agent: agent.id,
        environment_id: environment.id,
        title: "Quickstart session",
      });

      console.log(`Session ID: ${session.id}`);
      ```

      ```csharp C#
      var session = await client.Beta.Sessions.Create(new()
      {
          Agent = agent.ID,
          EnvironmentID = environment.ID,
          Title = "Quickstart session",
      });

      Console.WriteLine($"Session ID: {session.ID}");
      ```

      ```go Go
      session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
      	Agent:         anthropic.BetaSessionNewParamsAgentUnion{OfString: anthropic.String(agent.ID)},
      	EnvironmentID: environment.ID,
      	Title:         anthropic.String("Quickstart session"),
      })
      if err != nil {
      	panic(err)
      }

      fmt.Printf("Session ID: %s\n", session.ID)
      ```

      ```java Java
      var session = client.beta().sessions().create(SessionCreateParams.builder()
          .agent(agent.id())
          .environmentId(environment.id())
          .title("Quickstart session")
          .build());

      IO.println("Session ID: " + session.id());
      ```

      ```php PHP
      $session = $client->beta->sessions->create(
          agent: $agent->id,
          environmentID: $environment->id,
          title: 'Quickstart session',
      );

      echo "Session ID: {$session->id}\n";
      ```

      ```ruby Ruby
      session = client.beta.sessions.create(
        agent: agent.id,
        environment_id: environment.id,
        title: "Quickstart session"
      )

      puts "Session ID: #{session.id}"
      ```
    </CodeGroup>
  </Step>

  <Step title="发送消息并流式传输响应">
    打开一个流，发送一个用户事件，然后在事件到达时进行处理：

    <CodeGroup>
      ```bash cURL
      # 此工作流不适合转换为一次性的 shell 命令。
      # 请改用此代码组中的某个 SDK 示例。
      ```

      ```bash CLI
      # 此工作流不适合用一次性 shell 命令来实现。
      # 请改用本代码组中的某个 SDK 示例。
      ```

      ```python Python
      with client.beta.sessions.events.stream(session.id) as stream:
          # 在流打开后发送用户消息
          client.beta.sessions.events.send(
              session.id,
              events=[
                  {
                      "type": "user.message",
                      "content": [
                          {
                              "type": "text",
                              "text": "Create a Python script that generates the first 20 Fibonacci numbers and saves them to fibonacci.txt",
                          },
                      ],
                  },
              ],
          )

          # 处理流式传输事件
          for event in stream:
              match event.type:
                  case "agent.message":
                      for block in event.content:
                          if block.type == "text":
                              print(block.text, end="")
                  case "agent.tool_use":
                      print(f"\n[Using tool: {event.name}]")
                  case "session.status_idle":
                      print("\n\nAgent finished.")
                      break
      ```

      ```typescript TypeScript
      const stream = await client.beta.sessions.events.stream(session.id);

      // 在流打开后发送用户消息
      await client.beta.sessions.events.send(session.id, {
        events: [
          {
            type: "user.message",
            content: [
              {
                type: "text",
                text: "Create a Python script that generates the first 20 Fibonacci numbers and saves them to fibonacci.txt",
              },
            ],
          },
        ],
      });

      // 处理流式传输事件
      for await (const event of stream) {
        if (event.type === "agent.message") {
          for (const block of event.content) {
            if (block.type === "text") {
              process.stdout.write(block.text);
            }
          }
        } else if (event.type === "agent.tool_use") {
          console.log(`\n[Using tool: ${event.name}]`);
        } else if (event.type === "session.status_idle") {
          console.log("\n\nAgent finished.");
          break;
        }
      }
      ```

      ```csharp C#
      var stream = client.Beta.Sessions.Events.StreamStreaming(session.ID);

      // 在流打开后发送用户消息
      await client.Beta.Sessions.Events.Send(session.ID, new()
      {
          Events =
          [
              new BetaManagedAgentsUserMessageEventParams
              {
                  Type = "user.message",
                  Content =
                  [
                      new BetaManagedAgentsTextBlock
                      {
                          Type = "text",
                          Text = "Create a Python script that generates the first 20 Fibonacci numbers and saves them to fibonacci.txt",
                      },
                  ],
              },
          ],
      });

      // 处理流式传输事件
      await foreach (var ev in stream)
      {
          if (ev.Value is BetaManagedAgentsAgentMessageEvent message)
          {
              foreach (var block in message.Content)
              {
                  if (block.Value is BetaManagedAgentsTextBlock textBlock)
                  {
                      Console.Write(textBlock.Text);
                  }
              }
          }
          else if (ev.Value is BetaManagedAgentsAgentToolUseEvent toolUse)
          {
              Console.WriteLine($"\n[Using tool: {toolUse.Name}]");
          }
          else if (ev.Value is BetaManagedAgentsSessionStatusIdleEvent)
          {
              Console.WriteLine("\n\nAgent finished.");
              break;
          }
      }
      ```

      ```go Go
      	stream := client.Beta.Sessions.Events.StreamEvents(ctx, session.ID, anthropic.BetaSessionEventStreamParams{})
      	defer stream.Close()

      	// 在流打开后发送用户消息
      	_, err = client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
      		Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
      			OfUserMessage: &anthropic.BetaManagedAgentsUserMessageEventParams{
      				Type: anthropic.BetaManagedAgentsUserMessageEventParamsTypeUserMessage,
      				Content: []anthropic.BetaManagedAgentsUserMessageEventParamsContentUnion{{
      					OfText: &anthropic.BetaManagedAgentsTextBlockParam{
      						Type: anthropic.BetaManagedAgentsTextBlockTypeText,
      						Text: "Create a Python script that generates the first 20 Fibonacci numbers and saves them to fibonacci.txt",
      					},
      				}},
      			},
      		}},
      	})
      	if err != nil {
      		panic(err)
      	}

      	// 处理流式传输事件
      loop:
      	for stream.Next() {
      		switch event := stream.Current().AsAny().(type) {
      		case anthropic.BetaManagedAgentsAgentMessageEvent:
      			for _, block := range event.Content {
      				if block.Type == "text" {
      					fmt.Print(block.Text)
      				}
      			}
      		case anthropic.BetaManagedAgentsAgentToolUseEvent:
      			fmt.Printf("\n[Using tool: %s]\n", event.Name)
      		case anthropic.BetaManagedAgentsSessionStatusIdleEvent:
      			fmt.Print("\n\nAgent finished.\n")
      			break loop
      		}
      	}
      	if err := stream.Err(); err != nil {
      		panic(err)
      	}
      ```

      ```java Java
      try (var stream = client.beta().sessions().events().streamStreaming(session.id())) {
          // 在流打开后发送用户消息
          client.beta().sessions().events().send(session.id(), EventSendParams.builder()
              .addEvent(BetaManagedAgentsUserMessageEventParams.builder()
                  .type(BetaManagedAgentsUserMessageEventParams.Type.USER_MESSAGE)
                  .addTextContent("Create a Python script that generates the first 20 Fibonacci numbers and saves them to fibonacci.txt")
                  .build())
              .build());

          // 处理流式传输事件
          for (var event : (Iterable<BetaManagedAgentsStreamSessionEvents>) stream.stream()::iterator) {
              if (event.isAgentMessage()) {
                  event.asAgentMessage().content().forEach(block -> block.text().ifPresent(textBlock -> IO.print(textBlock.text())));
              } else if (event.isAgentToolUse()) {
                  IO.println("\n[Using tool: " + event.asAgentToolUse().name() + "]");
              } else if (event.isSessionStatusIdle()) {
                  IO.println("\n\nAgent finished.");
                  break;
              }
          }
      }
      ```

      ```php PHP
      $stream = $client->beta->sessions->events->streamStream($session->id);

      // 在流打开后发送用户消息
      $client->beta->sessions->events->send(
          $session->id,
          events: [
              [
                  'type' => 'user.message',
                  'content' => [
                      ['type' => 'text', 'text' => 'Create a Python script that generates the first 20 Fibonacci numbers and saves them to fibonacci.txt'],
                  ],
              ],
          ],
      );

      // 处理流式传输事件
      foreach ($stream as $event) {
          match ($event->type) {
              'agent.message' => array_walk(
                  $event->content,
                  static fn ($block) => $block->type === 'text' ? print($block->text) : null,
              ),
              'agent.tool_use' => print("\n[Using tool: {$event->name}]\n"),
              'session.status_idle' => print("\n\nAgent finished.\n"),
              default => null,
          };
          if ($event->type === 'session.status_idle') {
              break;
          }
      }
      ```

      ```ruby Ruby
      stream = client.beta.sessions.events.stream_events(session.id)

      # 在流打开后发送用户消息
      client.beta.sessions.events.send_(
        session.id,
        events: [{
          type: "user.message",
          content: [{type: "text", text: "Create a Python script that generates the first 20 Fibonacci numbers and saves them to fibonacci.txt"}]
        }]
      )

      # 处理流式传输事件
      stream.each do |event|
        case event.type
        in :"agent.message"
          event.content.each { print it.text if it.type == :text }
        in :"agent.tool_use"
          puts "\n[Using tool: #{event.name}]"
        in :"session.status_idle"
          puts "\n\nAgent finished."
          break
        else
          # 忽略其他事件类型
        end
      end
      ```
    </CodeGroup>

    智能体会编写一个 Python 脚本，在沙箱中运行它，并验证输出文件已创建。您的输出将类似于以下内容：

    ```text wrap
    I'll create a Python script that generates the first 20 Fibonacci numbers and saves them to a file.
    [Using tool: write]
    [Using tool: bash]
    The script ran successfully. Let me verify the output file.
    [Using tool: bash]
    fibonacci.txt contains the first 20 Fibonacci numbers (0 through 4181).

    Agent finished.
    ```
  </Step>
</Steps>

## 发生了什么

当您发送用户事件时，Claude Managed Agents 会：

1. **配置沙箱：** 您的环境配置决定了沙箱的构建方式。
2. **运行智能体循环：** Claude 根据您的消息决定使用哪些工具。
3. **运行工具：** 文件写入、bash 命令和其他工具调用都在沙箱内运行。
4. **流式传输事件：** 在智能体工作时，您会收到实时更新。
5. **进入空闲状态：** 当智能体没有更多事情要做时，会发出 `session.status_idle` 事件。

## 构建完整应用

以下每个快速入门都将 Claude Managed Agents 与一个流行的聊天框架相结合，构建出一个完整、可运行的应用程序。在每个示例中，框架负责渲染聊天界面，而托管会话在服务器端运行智能体循环：会话保存对话记录、在沙箱中运行工具，并流式传输由前端渲染的事件。

<CardGroup cols={3}>
  <Card title="Chat SDK" icon="github-logo" href="https://github.com/anthropics/claude-quickstarts/tree/main/managed-agents/chat-sdk">
    一个使用 Vercel 的 Chat SDK 构建的浏览器聊天中的研究分析师。每个对话都是一个持久会话，在流式传输回复的同时，实时信息流会显示工具调用。更换 Chat SDK 适配器即可将同一处理程序迁移到 Slack、Teams、Discord 或 WhatsApp。
  </Card>

  <Card title="assistant-ui" icon="github-logo" href="https://github.com/anthropics/claude-quickstarts/tree/main/managed-agents/assistant-ui">
    一个使用 assistant-ui 原语构建的聊天中的电子表格分析师。会话即线程列表，一个 reducer 将会话事件日志转换为消息和工具卡片，每条 bash 命令在运行前都会渲染一个内联的允许/拒绝确认门。
  </Card>

  <Card title="CopilotKit (AG-UI)" icon="github-logo" href="https://github.com/anthropics/claude-quickstarts/tree/main/managed-agents/copilot-kit-ag-ui">
    一个 CopilotKit 聊天中的个人理财助手。适用于 Claude Managed Agents 的 AG-UI 适配器将每个聊天线程映射到一个托管会话，并逐令牌流式传输回复，自定义工具则在对话中内联渲染交互式图表。
  </Card>
</CardGroup>

## 后续步骤

<CardGroup cols={2}>
  <Card title="定义您的智能体" icon="brain" href="https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup">
    创建可复用、带版本控制的智能体配置
  </Card>

  <Card title="配置环境" icon="settings" href="https://platform.claude.com/docs/zh-CN/managed-agents/environments">
    自定义网络和沙箱设置
  </Card>

  <Card title="智能体工具" icon="tool" href="https://platform.claude.com/docs/zh-CN/managed-agents/tools">
    为您的智能体启用特定工具
  </Card>

  <Card title="会话事件流" icon="lightning" href="https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming">
    处理事件并在执行过程中引导智能体
  </Card>

  <Card title="定时部署" icon="arrows-clockwise" href="https://platform.claude.com/docs/zh-CN/managed-agents/scheduled-deployments">
    按周期性 cron 计划运行您的智能体
  </Card>

  <Card title="知识维基快速入门" icon="github-logo" href="https://github.com/anthropics/claude-quickstarts/tree/main/managed-agents/knowledge-wiki">
    将文档语料库一次性提炼为知识维基，然后以极低的成本从中回答重复性问题
  </Card>
</CardGroup>
