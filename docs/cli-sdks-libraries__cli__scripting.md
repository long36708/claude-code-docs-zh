---
title: CLI 脚本编写与自动化
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/scripting
description: 将 API 资源以 YAML 形式进行版本控制，在脚本中串联 ant CLI 命令，从 Claude Code 操作资源，并使用 CLI 凭据对 curl 调用进行身份验证。
---

本页介绍基于 `ant` CLI 构建的面向任务的工作流。有关底层标志和输出选项，请参阅[使用 CLI](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/using)。

## 对 API 资源进行版本控制

您可以使用 CLI 将 skills、agents、environments 或 deployments 等 API 资源作为 YAML 文件在您的代码仓库中进行版本控制，并使其与 Claude API 保持同步。

<Note>
  有关这些资源的更多信息，请参阅 [Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)。
</Note>

<Steps>
  <Step title="定义您的 agent">
    将 agent 定义写入 `summarizer.agent.yaml`：

    ```yaml summarizer.agent.yaml
    name: Summarizer
    model: claude-opus-5
    system: |
      You are a helpful assistant that writes concise summaries.
    tools:
      - type: agent_toolset_20260401
    ```
  </Step>

  <Step title="创建 agent">
    ```bash
    ant beta:agents create < summarizer.agent.yaml
    ```

    ```json Output
    {
      "id": "agent_011CYm1BLqPXpQRk5khsSXrs",
      "version": 1,
      "name": "Summarizer",
      "model": "claude-opus-5"
      /* ... */
    }
    ```

    记下响应中的 `id`。您将在后续步骤中将其传递给会话创建命令。

    <Tip>
      将 `summarizer.agent.yaml` 提交到您的代码仓库，并在 CI 流水线中使其与 API 保持同步。更新命令需要以标志形式提供 agent ID 和当前版本：

      ```bash CLI
      ant beta:agents update --agent-id agent_011CYm1BLqPXpQRk5khsSXrs --version 1 < summarizer.agent.yaml
      ```
    </Tip>
  </Step>

  <Step title="定义 environment">
    会话在 [environment](https://platform.claude.com/docs/zh-CN/api/cli/beta/environments)（环境）中运行，环境定义了会话执行所在的沙箱。将环境定义写入 `summarizer.environment.yaml`：

    ```yaml summarizer.environment.yaml
    name: summarizer-env
    config:
      type: cloud
      networking:
        type: unrestricted
    ```
  </Step>

  <Step title="创建 environment">
    ```bash
    ant beta:environments create < summarizer.environment.yaml
    ```

    ```json Output
    {
      "id": "env_01595EKxaaTTGwwY3kyXdtbs",
      "name": "summarizer-env"
      /* ... */
    }
    ```

    记下响应中的 `id`。您将在后续步骤中将其传递给会话创建命令。

    <Tip>
      将 `summarizer.environment.yaml` 提交到您的代码仓库，并在 CI 流水线中使其与 API 保持同步。更新命令需要以标志形式提供 environment ID：

      ```bash CLI
      ant beta:environments update --environment-id env_01595EKxaaTTGwwY3kyXdtbs < summarizer.environment.yaml
      ```
    </Tip>
  </Step>

  <Step title="启动会话">
    将前面输出中的 agent `id` 和 environment `id` 粘贴到会话创建命令中：

    ```bash
    ant beta:sessions create \
      --agent agent_011CYm1BLqPXpQRk5khsSXrs \
      --environment-id env_01595EKxaaTTGwwY3kyXdtbs \
      --title "Summarization task"
    ```

    ```json Output
    {
      "id": "session_01JZCh78XvmxJjiXVy3oSi7K",
      "status": "running"
      /* ... */
    }
    ```
  </Step>

  <Step title="发送用户消息">
    将前面输出中的会话 `id` 复制到 `--session-id` 中：

    ```bash
    ant beta:sessions:events send \
      --session-id session_01JZCh78XvmxJjiXVy3oSi7K \
      --event '{type: user.message, content: [{type: text, text: "Summarize the benefits of type safety in one sentence."}]}'
    ```
  </Step>

  <Step title="读取对话">
    `--transform` 会针对列出的每个事件运行，因此这会按顺序打印每条消息的文本。`--format auto` 会覆盖 list 命令在终端中默认打开的交互式浏览器：

    ```bash
    ant beta:sessions:events list \
      --session-id session_01JZCh78XvmxJjiXVy3oSi7K \
      --transform 'content.0.text' --format auto --raw-output
    ```

    ```text Output wrap
    Summarize the benefits of type safety in one sentence.
    Type safety catches errors at compile time rather than runtime, reducing bugs, improving code clarity, enabling better tooling support, and making codebases easier to maintain and refactor with confidence.
    ```

    <Tip>
      要在会话运行时进行观察，请使用 `ant beta:sessions:events stream --session-id session_01JZCh78XvmxJjiXVy3oSi7K`。事件会在到达时写入 stdout。
    </Tip>
  </Step>
</Steps>

## 脚本编写模式

CLI 的设计旨在与标准 shell 工具组合使用。

### 将 list 输出串联到第二个命令

在 list 端点上使用 `--transform id --raw-output` 会每行输出一个裸 ID，因此 `head` 和 `xargs` 等标准工具可以直接应用。捕获第一个结果，然后将其传递给后续命令：

```bash
FIRST_AGENT=$(ant beta:agents list --transform id --raw-output | head -1)

ant beta:agents:versions list \
  --agent-id "$FIRST_AGENT" \
  --transform "{version,created_at}" --format jsonl
```

### 检查错误

`--transform-error` 和 `--format-error` 标志对错误响应应用相同的过滤。`--raw-output` 不适用于错误，因此请使用 `--format-error yaml` 来获取不带引号的标量。仅提取错误消息：

```bash
ant beta:agents retrieve --agent-id bogus \
  --transform-error error.message --format-error yaml 2>&1
```

```text Output wrap
GET "https://api.anthropic.com/v1/agents/bogus?beta=true": 404 Not Found
Agent not found.
```

## 从 Claude Code 使用 CLI

[Claude Code](https://code.claude.com/docs/en/overview) 可以开箱即用地使用 `ant` CLI。在安装 CLI 并完成身份验证后，您可以让 Claude Code 直接操作您的 API 资源。例如：

* "列出我最近的 agent 会话，并总结哪些会话出错了。"
* "将 `./reports` 中的每个 PDF 上传到 Files API，并打印生成的 ID。"
* "拉取会话 `session_01...` 的事件，并告诉我 agent 在哪里卡住了。"

Claude Code 会调用 `ant`，解析结构化输出，并对结果进行推理（无需自定义集成代码）。

## 使用 CLI 凭据对 curl 请求进行身份验证

使用 `curl` 或其他 HTTP 客户端调用 API 的脚本可以使用 [`ant auth login`](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/quickstart#authentication) 存储的凭据，而不是静态 API 密钥。OAuth 访问令牌作为 bearer 令牌放在 `Authorization` 标头中；`x-api-key` 标头仅用于静态 API 密钥。

`ant auth print-credentials --access-token` 会打印活动配置文件的访问令牌，如果令牌已过期或即将过期，则会先刷新它：

```bash cURL
curl https://api.anthropic.com/v1/messages \
  -H "Authorization: Bearer $(ant auth print-credentials --access-token)" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-5",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "hi"}]
  }'
```

<Note>
  通过 CLI 登录进行操作时，请保持 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN` 未设置。对于 `ant` 命令，这两个变量中的任何一个都优先于登录凭据（请参阅[凭据优先级](https://platform.claude.com/docs/zh-CN/manage-claude/wif-reference#credential-precedence)），并可能在不知不觉中将命令路由到不同的组织或工作区。
</Note>

运行 [`ant auth status`](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/authentication#check-authentication-status) 以确认您登录的是哪个组织和工作区；当环境变量覆盖您的登录时，它会发出警告。
