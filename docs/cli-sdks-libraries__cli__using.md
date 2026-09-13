---
title: 使用 CLI
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/using
description: ant CLI 的命令结构、输出格式、GJSON 转换、请求体以及调试。
---

本页介绍 `ant` CLI 适用于所有端点的输入和输出机制。如需安装和身份验证，请参阅[快速入门](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/quickstart)。如需串联命令和对资源进行版本控制，请参阅 [CLI 脚本与自动化](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/scripting)。

## 命令结构

命令遵循 `resource action`（资源 操作）模式。嵌套资源使用冒号：

```text wrap
ant <resource>[:<subresource>] <action> [flags]
```

运行 `ant --help` 查看完整的资源列表，或在任意子命令后追加 `--help` 查看其标志。

处于 beta 阶段的资源（包括 agents、sessions、deployments 和 environments）位于 `beta:` 前缀下。此命名空间中的命令会自动为该资源发送相应的 `anthropic-beta` 请求头，因此您无需自行传递。仅在需要覆盖默认值时使用 `--beta <header>`（例如，选择使用不同的 schema 版本）。

```bash
ant models list
ant messages create --model claude-opus-5 --max-tokens 1024 ...
ant beta:agents retrieve --agent-id agent_01...
ant beta:sessions:events list --session-id session_01...
```

### 全局标志

| 标志                                   | 描述                                                                                                                                                                                                                                                                                                                                 |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--profile`                          | 本次调用使用的命名配置文件（等同于设置 `ANTHROPIC_PROFILE`）。请参阅[在工作区之间切换](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/authentication#switch-between-workspaces)。                                                                                                                                                                    |
| `--format`                           | 输出格式：`auto`、`json`、`jsonl`、`yaml`、`pretty`、`raw`、`explore`                                                                                                                                                                                                                                                                         |
| `--transform`                        | 使用 [GJSON 路径](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/using#transform-output-with-gjson)过滤或重塑响应                                                                                                                                                                                                              |
| `-r`、`--raw-output`                  | 打印字符串结果时不带外围引号，类似 `jq -r`                                                                                                                                                                                                                                                                                                          |
| `--base-url`                         | 覆盖 API 基础 URL                                                                                                                                                                                                                                                                                                                      |
| `--workspace-id`                     | 可选。作为 `anthropic-workspace-id` 请求头发送的工作区 ID（`wrkspc_...`），适用于可访问多个工作区的 API 密钥（等同于设置 `ANTHROPIC_WORKSPACE_ID`）。请参阅[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)。[Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 命令有自己的 `--workspace-id`，用于指定其所管理的工作区。 |
| `--debug`                            | 将完整的 HTTP 请求和响应打印到 stderr                                                                                                                                                                                                                                                                                                          |
| `--format-error`、`--transform-error` | 与 `--format` 和 `--transform` 相同，但应用于[错误响应](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/scripting#inspect-errors)                                                                                                                                                                                                 |

## 输出格式

`auto` 会美化打印 JSON，是创建或修改资源的命令的默认格式。列表和检索命令在写入终端时默认使用[交互式浏览器](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/using#interactive-explorer)，在通过管道传输时默认使用美化打印的 JSON。可使用 `--format` 覆盖任一默认值：

```bash
ant models retrieve --model-id claude-opus-5 --format yaml
```

```yaml Output
type: model
id: claude-opus-5
display_name: Claude Opus 5
created_at: "2026-07-24T00:00:00Z"
...
```

列表端点会自动分页。在默认格式下，每个条目会单独写出（`jsonl` 模式下每行一个紧凑的 JSON 对象，`yaml` 模式下为 YAML 文档流），可以顺畅地流式传输到 `head`、`grep` 和 `--transform` 过滤器中。

### 交互式浏览器

该浏览器是一个支持折叠和搜索的 TUI（终端用户界面），用于浏览大型响应。方向键可展开和折叠节点，`/` 用于搜索，`q` 用于退出。列表和检索命令在连接到终端时默认打开它。传递 `--format explore` 可显式打开：

```bash
ant models list --format explore
```

## 使用 GJSON 转换输出

使用 `--transform` 在打印前重塑响应。表达式为 [GJSON 路径](https://github.com/tidwall/gjson/blob/master/SYNTAX.md)。对于列表端点，转换会针对每个条目单独运行，而不是针对外层封装：

```bash
ant beta:agents list \
  --transform "{id,name,model}" \
  --format jsonl
```

```jsonl Output
{"id": "agent_011CYm1BLqPX...", "name": "Docs CLI Test Agent", "model": "claude-opus-5"}
{"id": "agent_011CYkVwfaEt...", "name": "Coffee Making Assistant", "model": "claude-opus-5"}
{"id": "agent_011CYixHhtUP...", "name": "Coding Assistant", "model": "claude-opus-5"}
```

### 提取标量

要将单个字段捕获为不带引号的字符串（例如，新创建资源的 ID），请将 `--transform` 与 `--raw-output` 搭配使用。结果打印时不带 JSON 引号，可直接赋值给 shell 变量：

```bash
AGENT_ID=$(ant beta:agents create \
  --name "My Agent" \
  --model '{id: claude-opus-5}' \
  --transform id --raw-output)

printf '%s\n' "$AGENT_ID"
```

```text Output wrap
agent_011CYm1BLqPXpQRk5khsSXrs
```

<Note>
  `--raw-output` 与 `--format raw` 不同。`--raw-output` 会去除字符串结果的 JSON 引号，类似 `jq -r`。`--format raw` 会打印响应体的原始 JSON 字节而不自动分页；在列表端点上，它会将 `--transform` 应用于分页封装，而不是应用于每个条目。
</Note>

## 传递请求体

合适的输入机制取决于数据的形态：对标量字段和简短的结构化值使用**标志**，对嵌套或多行请求体通过管道传入 **stdin** 文档，并使用 **`@file` 引用**将文件内容拉入任意字符串或二进制字段。

### 标志

标量字段直接映射到标志。结构化字段接受宽松的类 YAML 语法（键不加引号，字符串引号可选）或严格的 JSON：

```bash
ant beta:sessions create \
  --agent '{type: agent, id: agent_011CYm1BLqPXpQRk5khsSXrs, version: 1}' \
  --environment-id env_01595EKxaaTTGwwY3kyXdtbs \
  --title "CLI docs test session"
```

可重复的标志用于构建数组。每个 `--tool` 或 `--event` 追加一个元素：

```bash
ant beta:agents create \
  --name "Research Agent" \
  --model '{id: claude-opus-5}' \
  --tool '{type: agent_toolset_20260401}' \
  --tool '{type: custom, name: search_docs, input_schema: {type: object, properties: {query: {type: string}}}}'
```

### Stdin

通过管道将 JSON 或 YAML 文档传入 stdin 以提供完整的请求体。来自 stdin 的字段会与标志合并，标志优先。此处 `version` 是先前 `retrieve` 返回的乐观锁令牌，`$AGENT_ID` 则按照[提取标量](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/using#extract-a-scalar)中的方式捕获：

```bash
echo '{"description": "Updated test agent.", "version": 1}' | \
  ant beta:agents update --agent-id "$AGENT_ID"
```

Heredoc 的工作方式相同，且便于编写多行 YAML。为分隔符加引号（如 `<<'YAML'`）可禁用请求体内的变量展开。

```bash
ant beta:agents create <<'YAML'
name: Research Agent
model: claude-opus-5
system: |
  You are a research assistant. Cite sources for every claim.
tools:
  - type: agent_toolset_20260401
YAML
```

### 文件引用

接受文件路径的标志（例如上传命令中的 `--file`）可直接接受裸路径：

```bash
ant files upload --file ./report.pdf
```

要将文件内容内联到字符串值字段中，请在路径前加上 `@` 前缀：

```bash
ant beta:agents create \
  --name "Researcher" --model '{id: claude-opus-5}' \
  --system @./prompts/researcher.txt
```

在结构化标志值内部，请用引号包裹路径。要向 Messages API 发送 PDF：

```bash
ant messages create \
  --model claude-opus-5 \
  --max-tokens 1024 \
  --message '{role: user, content: [
    {type: document, source: {type: base64, media_type: application/pdf, data: "@./scan.pdf"}},
    {type: text, text: "Extract the text from this scanned document."}
  ]}' \
  --transform 'content.#(type=="text").text' --raw-output
```

CLI 会检测文件类型，并自动将二进制文件编码为 base64。要强制使用特定编码，纯文本使用 `@file://`，base64 使用 `@data://`。使用反斜杠转义字面量开头的 `@`（`\@username`）。

## 调试

在任意命令中添加 `--debug`，即可将确切的 HTTP 请求和响应（请求头和请求体）打印到 stderr。API 密钥会被隐去。

```bash
ant --debug beta:agents list
```

```text Output wrap
GET /v1/agents?beta=true HTTP/1.1
Host: api.anthropic.com
Anthropic-Beta: managed-agents-2026-04-01
Anthropic-Version: 2023-06-01
X-Api-Key: <REDACTED>
...
```

## 可用资源

CLI 公开的每个 API 资源都记录在 [API 参考](https://platform.claude.com/docs/zh-CN/api/cli/messages/create)中。如需本地列表，请运行 `ant --help`，并在任意子命令后追加 `--help` 查看其标志和参数。

## 后续步骤

<CardGroup cols={3}>
  <Card title="CLI 脚本与自动化" icon="code" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/scripting">
    对 API 资源进行版本控制、脚本模式，以及在 Claude Code 中使用
  </Card>

  <Card title="API 参考" icon="book" href="https://platform.claude.com/docs/zh-CN/api/cli/messages/create">
    端点特定的参数、请求字段和响应 schema
  </Card>

  <Card title="CLI 身份验证选项" icon="lock" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/authentication">
    API 密钥、无头主机、多个工作区和命名配置文件
  </Card>
</CardGroup>
