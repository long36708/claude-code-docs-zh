> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 修改系统提示词

> 在 `claude_code` 预设和自定义系统提示词之间进行选择，并通过 CLAUDE.md、输出样式、追加或完全自定义提示词来自定义行为。

系统提示词定义了 Claude 的行为、能力和响应风格。从用于 CLI 或 IDE 类编码工具的 `claude_code` 预设开始，其中人类观察并指导工作。为具有不同界面、身份或权限模型的代理编写自己的提示词。

<h2 id="how-system-prompts-work">
  系统提示词的工作原理
</h2>

系统提示词是初始指令集，它塑造了 Claude 在整个对话中的行为方式。Agent SDK 有三个起点：

* **最小默认值**：当你在 TypeScript 中不设置 `systemPrompt` 或在 Python 中不设置 `system_prompt` 时，SDK 使用最小提示词，涵盖工具调用但省略了 `claude_code` 预设的其余内容，包括其安全和安全指令以及关于工作目录和环境的上下文。这与 `claude -p` 不同，后者默认使用 Claude Code 系统提示词。如果你从 CLI 迁移并想要匹配的行为，请设置 `claude_code` 预设。
* **`claude_code` 预设**：Claude Code CLI 使用的系统提示词，包含工具使用说明、安全和安全指令，以及关于工作目录和环境的上下文。在 TypeScript 中设置 `systemPrompt: { type: "preset", preset: "claude_code" }`，或在 Python 中设置 `system_prompt={"type": "preset", "preset": "claude_code"}`，可选择使用 `append` 在末尾添加你自己的指令。
* **自定义字符串**：你自己编写的提示词。SDK 仅发送你提供的内容。

<h3 id="decide-on-a-starting-point">
  决定起点
</h3>

决定因素是你的代理与 Claude Code 的相似程度：一个在存储库中运行的编码代理，有人类观看流式输出并指导工作。你的产品离这个越远，你就越想编写自己的提示词。

| 你正在构建                                                | 使用                         | 你获得的内容                                      |
| :--------------------------------------------------- | :------------------------- | :------------------------------------------ |
| 一个 CLI 或类似 IDE 的编码工具，其中人类观看和指导，Claude Code 的默认值是你想要的 | `claude_code` 预设           | Claude Code 提示词，包括工具指导、安全规则和环境上下文           |
| 相同类型的工具，加上产品特定的规则，如编码标准、输出格式或域上下文                    | `claude_code` 预设加 `append` | 上述所有内容，加上你的指令添加在预设之后。没有任何内容被删除，所以这是风险最低的自定义 |
| 具有不同表面、身份或权限模型的代理，或非编码代理                             | 自定义提示词字符串                  | 仅你编写的内容。你负责替换你的代理仍然需要的工具指导和安全指令             |
| 一个薄工具调用循环，没有代理角色，你在用户提示词中提供所有行为                      | 无 `systemPrompt` 选项        | 最小默认值：工具调用支持，仅此而已                           |

"不同于 Claude Code" 通常意味着以下之一：

* **不同的表面**：输出不是由触发它的人在终端中读取的。聊天 UI、结构化输出消费者和非编码自动化各自需要一个与其输出呈现和审查方式相匹配的提示词。无人值守的编码自动化，如修复 lint 错误或审查差异的 CI 作业，仍然适合预设，因为工作本身就是预设为之编写的。
* **不同的身份**：代理不应该将自己呈现为 Claude Code。支持机器人、数据分析助手或任何特定领域的代理需要自己的名称、范围和角色。
* **不同的权限模型**：代理自主运行，无需人类批准每一步，或在一组狭窄的资源上运行。Claude Code 的提示词假设人类在循环中，可以访问完整的工具集。
* **非编码任务**：Claude Code 提示词的大部分是编码指导。对于研究、内容或运营代理，该指导与你实际需要的指令竞争。

[比较表](#compare-the-four-approaches)显示了每种自定义方法保留的内容。

<h2 id="customize-agent-behavior">
  自定义代理行为
</h2>

`append` 和自定义提示字符串各自直接改变系统提示，输出样式改变 Claude Code 为每个响应给 Claude 的指令。CLAUDE.md 采用不同的路径：SDK 读取它并将其内容注入到对话中作为项目上下文，因此它与你选择的任何系统提示一起塑造行为。[Skills](/docs/zh-CN/agent-sdk/skills)、[hooks](/docs/zh-CN/agent-sdk/hooks) 和 [permissions](/docs/zh-CN/agent-sdk/permissions) 也在系统提示之外塑造行为，并在各自的页面上介绍。

<h3 id="claude-md-files-for-project-level-instructions">
  用于项目级指令的 CLAUDE.md 文件
</h3>

CLAUDE.md 文件为 Claude 提供持久的项目上下文和指令。SDK 将其内容注入到对话中并保持系统提示不变，因此它们与任何系统提示配置一起工作。关于在 CLAUDE.md 中放什么、放在哪里以及如何编写有效的指令，请参阅 [When to add to CLAUDE.md](/docs/zh-CN/memory#when-to-add-to-claude-md) 和 [How Claude remembers your project](/docs/zh-CN/memory) 的其余部分。本节涵盖 SDK 特定的内容：CLAUDE.md 如何加载。

SDK 在匹配的设置源启用时读取 CLAUDE.md：`'project'` 从工作目录加载 `CLAUDE.md` 或 `.claude/CLAUDE.md`，`'user'` 加载 `~/.claude/CLAUDE.md`。默认 `query()` 选项启用两个源，因此 CLAUDE.md 自动加载。如果你在 TypeScript 中显式设置 `settingSources` 或在 Python 中设置 `setting_sources`，请包含你需要的源。CLAUDE.md 加载由设置源控制，而不是由 `claude_code` 预设控制。

<h4 id="load-claude-md-with-the-sdk">
  使用 SDK 加载 CLAUDE.md
</h4>

要加载 CLAUDE.md，请设置 `settingSources` 以包含你保存 CLAUDE.md 的级别。下面的示例加载项目级 CLAUDE.md 以及 `claude_code` 预设，因此 Claude 既有编码代理提示，也有你的项目约定：

<CodeGroup>
  ```typescript TypeScript theme={null}
  import { query } from "@anthropic-ai/claude-agent-sdk";

  const messages = [];

  for await (const message of query({
    prompt: "Add a new React component for user profiles",
    options: {
      systemPrompt: {
        type: "preset",
        preset: "claude_code" // Use Claude Code's system prompt
      },
      settingSources: ["project"] // Loads CLAUDE.md from project
    }
  })) {
    messages.push(message);
  }

  // Now Claude has access to your project guidelines from CLAUDE.md
  ```

  ```python Python theme={null}
  import asyncio

  from claude_agent_sdk import query, ClaudeAgentOptions

  messages = []


  async def main():
      async for message in query(
          prompt="Add a new React component for user profiles",
          options=ClaudeAgentOptions(
              system_prompt={
                  "type": "preset",
                  "preset": "claude_code",  # Use Claude Code's system prompt
              },
              setting_sources=["project"],  # Loads CLAUDE.md from project
          ),
      ):
          messages.append(message)


  asyncio.run(main())

  # Now Claude has access to your project guidelines from CLAUDE.md
  ```
</CodeGroup>

当你运行任一示例时，SDK 在 Claude 工作时流式传输消息：系统初始化消息、助手消息、携带工具结果的用户消息，以及包含会话结果的最终结果消息。

CLAUDE.md 在项目中的所有会话中持久存在，通过 git 与你的团队共享，并自动发现而无需代码更改。如果你传递空的 `settingSources` 数组，它不会被加载。

<h3 id="output-styles-for-persistent-configurations">
  用于持久配置的输出样式
</h3>

输出样式是改变 Claude 的角色、语气和输出格式的已保存指令集。它们存储为 markdown 文件，可以在会话和项目中重复使用。

<h4 id="create-an-output-style">
  创建输出样式
</h4>

输出样式是一个 markdown 文件，包含用于元数据的 [frontmatter](/docs/zh-CN/output-styles#frontmatter)，后跟提示内容。将其保存到 `~/.claude/output-styles/` 以获得在每个项目中可用的用户级样式，或保存到你的存储库中的 `.claude/output-styles/` 以获得可以提交并与你的团队共享的项目级样式。

自定义输出样式会排除 `claude_code` 预设的软件工程指令，并使用你自己的。要保留它们并在其上分层你的指令，请在 frontmatter 中设置 `keep-coding-instructions: true`。这些指令仅在 Claude Code 的完整系统提示中，因此该设置在较短系统提示的会话中无效，你可以使用 [`CLAUDE_CODE_SIMPLE_SYSTEM_PROMPT`](/docs/zh-CN/env-vars#variables) 固定打开或关闭。当你的代理仍在进行软件工程工作时保留它们。当你完全替换角色时排除它们。

下面的示例定义了一个代码审查角色，它保留编码指令，因为审查代码仍然受益于 Claude Code 的安全和代码质量指导。将其保存为 `~/.claude/output-styles/code-reviewer.md` 以使其在项目中可用：

```markdown ~/.claude/output-styles/code-reviewer.md theme={null}
---
name: Code Reviewer
description: Thorough code review assistant
keep-coding-instructions: true
---

You are an expert code reviewer.

For every code submission:
1. Check for bugs and security issues
2. Evaluate performance
3. Suggest improvements
4. Rate code quality (1-10)
```

<h4 id="activate-an-output-style">
  激活输出样式
</h4>

创建后，通过以下方式激活输出样式：

* **CLI**：运行 `/output-style <style>`，例如 `/output-style concise`，或运行 `/config` 并选择一个。`/output-style` 命令需要 Claude Code v2.1.269 或更高版本。
* **设置**：在 `.claude/settings.local.json` 中设置 `outputStyle`
* **TypeScript SDK**：在传递给 `query()` 的内联 `settings` 对象中设置 `outputStyle`，或指向设置它的设置文件。`outputStyle` 不是顶级 `Options` 字段：

  ```typescript theme={null}
  const options = { settings: { outputStyle: "Explanatory" } };
  ```

在 Python SDK 中，通过 `settings` 选项设置 `outputStyle`，该选项接受 JSON 字符串（如 `'{"outputStyle": "Explanatory"}'`）或设置它的设置文件的路径。

**SDK 用户注意：** 当你包含 `settingSources: ['user']` 或 `settingSources: ['project']`（TypeScript）/ `setting_sources=["user"]` 或 `setting_sources=["project"]`（Python）在你的选项中时，输出样式会被加载。

<h3 id="append-to-the-claude_code-preset">
  追加到 `claude_code` 预设
</h3>

你可以使用带有 `append` 属性的 Claude Code 预设来添加你的自定义指令，同时保留所有内置功能。

<CodeGroup>
  ```typescript TypeScript theme={null}
  import { query } from "@anthropic-ai/claude-agent-sdk";

  const messages = [];

  for await (const message of query({
    prompt: "Help me write a Python function to calculate fibonacci numbers",
    options: {
      systemPrompt: {
        type: "preset",
        preset: "claude_code",
        append: "Always include detailed docstrings and type hints in Python code."
      }
    }
  })) {
    messages.push(message);
    if (message.type === "assistant") {
      console.log(message.message.content);
    }
  }
  ```

  ```python Python theme={null}
  import asyncio

  from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage

  messages = []


  async def main():
      async for message in query(
          prompt="Help me write a Python function to calculate fibonacci numbers",
          options=ClaudeAgentOptions(
              system_prompt={
                  "type": "preset",
                  "preset": "claude_code",
                  "append": "Always include detailed docstrings and type hints in Python code.",
              }
          ),
      ):
          messages.append(message)
          if isinstance(message, AssistantMessage):
              print(message.content)


  asyncio.run(main())
  ```
</CodeGroup>

<h4 id="improve-prompt-caching-across-users-and-machines">
  改进跨用户和机器的提示缓存
</h4>

默认情况下，两个使用相同 `claude_code` 预设和 `append` 文本的会话，如果从不同的工作目录运行，仍然无法共享提示缓存条目。这是因为预设在你的 `append` 文本之前在系统提示中嵌入了每个会话的上下文：工作目录、它是否是 git 存储库、平台、活跃的 shell、OS 版本和自动内存路径。该上下文中的任何差异都会产生不同的系统提示和缓存未命中。CLAUDE.md 内容不影响系统提示缓存，因为 SDK 将其注入到对话中，而不是系统提示。

要使系统提示在会话中相同，请在 TypeScript 中设置 `excludeDynamicSections: true` 或在 Python 中设置 `"exclude_dynamic_sections": True`。每个会话的上下文移动到第一条用户消息中，仅在系统提示中保留静态预设和你的 `append` 文本，因此相同的配置在用户和机器之间共享缓存条目。

<Note>
  `excludeDynamicSections` 需要 `@anthropic-ai/claude-agent-sdk` v0.2.98 或更高版本，或 Python 的 `claude-agent-sdk` v0.1.58 或更高版本。仅在预设对象形式上设置它。当你传递自定义提示而不是预设时，SDK 会忽略它；要在 TypeScript SDK 中保持自定义提示的指令缓存，请参阅 [Cache the static part of a custom prompt](#cache-the-static-part-of-a-custom-prompt)。
</Note>

以下示例将共享的 `append` 块与 `excludeDynamicSections` 配对，以便从不同目录运行的代理队列可以重复使用相同的缓存系统提示：

<CodeGroup>
  ```typescript TypeScript theme={null}
  import { query } from "@anthropic-ai/claude-agent-sdk";

  for await (const message of query({
    prompt: "Triage the open issues in this repo",
    options: {
      systemPrompt: {
        type: "preset",
        preset: "claude_code",
        append: "You operate Acme's internal triage workflow. Label issues by component and severity.",
        excludeDynamicSections: true
      }
    }
  })) {
    // ...
  }
  ```

  ```python Python theme={null}
  import asyncio

  from claude_agent_sdk import query, ClaudeAgentOptions


  async def main():
      async for message in query(
          prompt="Triage the open issues in this repo",
          options=ClaudeAgentOptions(
              system_prompt={
                  "type": "preset",
                  "preset": "claude_code",
                  "append": "You operate Acme's internal triage workflow. Label issues by component and severity.",
                  "exclude_dynamic_sections": True,
              },
          ),
      ):
          ...


  asyncio.run(main())
  ```
</CodeGroup>

**权衡：** 工作目录、git 存储库标志、平台、活跃的 shell、OS 版本和自动内存路径仍然到达 Claude，但作为第一条用户消息的一部分，而不是系统提示。用户消息中的指令比系统提示中的相同文本的权重略低，因此 Claude 在推理当前目录或自动内存路径时可能会更少依赖它们。当跨会话缓存重复使用比最大化权威环境上下文更重要时，启用此选项。

对于非交互式 CLI 模式中的等效标志，请参阅 [`--exclude-dynamic-system-prompt-sections`](/docs/zh-CN/cli-reference)。

<h3 id="custom-system-prompts">
  自定义系统提示
</h3>

你可以提供自定义字符串作为 `systemPrompt` 以完全替换默认值为你自己的指令。

<CodeGroup>
  ```typescript TypeScript theme={null}
  import { query } from "@anthropic-ai/claude-agent-sdk";

  const customPrompt = `You are a Python coding specialist.
  Follow these guidelines:
  - Write clean, well-documented code
  - Use type hints for all functions
  - Include comprehensive docstrings
  - Prefer functional programming patterns when appropriate
  - Always explain your code choices`;

  const messages = [];

  for await (const message of query({
    prompt: "Create a data processing pipeline",
    options: {
      systemPrompt: customPrompt
    }
  })) {
    messages.push(message);
    if (message.type === "assistant") {
      console.log(message.message.content);
    }
  }
  ```

  ```python Python theme={null}
  import asyncio

  from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage

  custom_prompt = """You are a Python coding specialist.
  Follow these guidelines:
  - Write clean, well-documented code
  - Use type hints for all functions
  - Include comprehensive docstrings
  - Prefer functional programming patterns when appropriate
  - Always explain your code choices"""

  messages = []


  async def main():
      async for message in query(
          prompt="Create a data processing pipeline",
          options=ClaudeAgentOptions(system_prompt=custom_prompt),
      ):
          messages.append(message)
          if isinstance(message, AssistantMessage):
              print(message.content)


  asyncio.run(main())
  ```
</CodeGroup>

在 Python 中，使用 `system_prompt={"type": "file", "path": "..."}` 从文件加载大型自定义提示，而不是将其作为字符串传递。Python SDK 将字符串提示作为一个命令行参数传递给 CLI 子进程，因此超过 OS 参数长度限制的提示在任何 API 请求发送之前在进程生成时失败。在 Linux 上，错误是 `Argument list too long`。请参阅 [`SystemPromptFile`](/docs/zh-CN/agent-sdk/python#systempromptfile) 了解平台阈值和 Windows 行为。

<h4 id="cache-the-static-part-of-a-custom-prompt">
  缓存自定义提示的静态部分
</h4>

在 TypeScript SDK 中，你可以将自定义提示作为字符串数组而不是一个字符串传递，在静态部分和其余部分之间使用 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记。当你的提示结合在每个请求上相同的指令与每个请求变化的上下文（如代理处理的客户或工单）时，使用此方法。当你将两部分作为一个字符串传递时，对每个请求部分的更改会改变整个系统提示，因此静态指令也会错过缓存。数组形式在 Python SDK 中不可用；[`ClaudeAgentOptions`](/docs/zh-CN/agent-sdk/python#claudeagentoptions) 列出 `system_prompt` 接受的形式。

<Note>
  Claude Code 仅在直接调用 Claude API 或在 [Claude Platform on AWS](/docs/zh-CN/claude-platform-on-aws) 上运行时拆分提示。在所有其他配置中，例如 Amazon Bedrock、Google Cloud 的 Agent Platform、Microsoft Foundry 或 [LLM gateway](/docs/zh-CN/llm-gateway-connect)，它将整个提示作为一个块发送，与传递一个字符串相同。每当你设置 [`CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1`](/docs/zh-CN/llm-gateway-protocol#disable-pre-release-capabilities) 时也会发生这种情况。
</Note>

要拆分提示，从 `@anthropic-ai/claude-agent-sdk` 导入 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 并将其作为自己的数组元素在两部分之间传递。SDK 将标记之前的字符串作为一个文本块发送，将标记之后的字符串作为第二个块发送，每个都有自己的缓存断点。在下面的示例中，支持代理从文件加载其分类指令，并在每个请求上接收有关一个工单的详细信息，因此指令保持缓存，而工单详细信息改变：

```typescript TypeScript theme={null}
import { readFile } from "node:fs/promises";
import { query, SYSTEM_PROMPT_DYNAMIC_BOUNDARY } from "@anthropic-ai/claude-agent-sdk";

// Identical on every request
const instructions = await readFile("triage-instructions.md", "utf8");
// Different on every request
const ticketContext = "Customer plan: Enterprise. Other open tickets from this customer: 3.";

for await (const message of query({
  prompt: "Triage ticket 4821",
  options: {
    systemPrompt: [instructions, SYSTEM_PROMPT_DYNAMIC_BOUNDARY, ticketContext]
  }
})) {
  // ...
}
```

[Track cache tokens](/docs/zh-CN/agent-sdk/cost-tracking#track-cache-tokens) 描述了每个结果消息上的 `cache_creation_input_tokens` 和 `cache_read_input_tokens` 字段。

SDK 从数组中组装块如下：

* SDK 将标记每一侧的字符串与它们之间的空行连接，并删除标记本身，因此标记文本不会到达 Claude。
* 如果你多次包含标记，第一个是拆分，SDK 删除其他的。
* 如果你省略标记，SDK 将所有字符串连接到一个块中，与传递一个字符串相同。

使用 CLI 的 [`--system-prompt` 或 `--system-prompt-file` 标志](/docs/zh-CN/cli-reference#system-prompt-flags)，提示是一个字符串，因此没有数组来携带标记。在静态和每个请求部分之间包含仅包含 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 的行。Claude Code 在第一个这样的行处拆分提示为相同的两个块并删除该行。需要 Claude Code v2.1.275 或更高版本。

在 SDK 中，更喜欢数组形式，它不带标记行地携带边界。

<h3 id="change-the-prompt-of-an-existing-session">
  更改现有会话的提示
</h3>

默认情况下，如果你在使用 `resume` 或 `continue` 返回会话时传递不同的 `append` 或自定义提示，Claude 在下一轮不会看到它。Claude Code 在会话的第一个请求上记录系统提示，并重复使用该记录直到会话被压缩。新文本在该压缩之后或在新会话中生效。

<h4 id="update-claude’s-instructions-mid-session">
  在会话中更新 Claude 的指令
</h4>

如果你在系统提示中放置的指令需要在会话运行时改变，例如因为你的用户将代理切换到只读模式或在你的应用中编辑其配置，请在对话中发送新指令，而不是改变 `systemPrompt`：

* **在你的下一条消息中**：在你发送的下一条用户消息中包含新指令。
* **从 hook 中**：从 `UserPromptSubmit` 或 `PostToolUse` [hook 回调](/docs/zh-CN/agent-sdk/hooks#outputs) 返回 [`additionalContext`](/docs/zh-CN/hooks#add-context-for-claude)，写成事实陈述，如"工作区现在是只读的"。SDK 在 hook 触发的点将文本插入到对话中，因此记录的提示保持不变。

<h4 id="turn-recording-off-while-you-iterate-on-wording">
  在迭代措辞时关闭记录
</h4>

当你迭代提示措辞并希望每个编辑到达你恢复的会话时，在系统提示的对象形式上设置 `snapshot` 为 false。Claude Code 然后在每个请求上重建提示。该字段在 TypeScript 中的 [`systemPrompt`](/docs/zh-CN/agent-sdk/typescript#options) 的预设和自定义形式上可用，在 Python 中的 [`system_prompt`](/docs/zh-CN/agent-sdk/python#systempromptpreset) 上可用，并需要 `@anthropic-ai/claude-agent-sdk` v0.3.257 或更高版本，或 `claude-agent-sdk` v0.2.153 或更高版本。

在生产中保持记录打开。关闭记录时，恢复的会话上的不同 `append` 或自定义提示在下一轮到达 Claude，该请求无法重复使用会话的 [prompt cache](/docs/zh-CN/prompt-caching#how-the-cache-is-organized)。在 API 强制 [preserved thinking](https://platform.claude.com/docs/en/build-with-claude/preserved-thinking) 的地方，Claude 也会失去其早期轮次的思考。

在 [cloud sessions](/docs/zh-CN/cloud-environments) 之外，如果你通过 `extraArgs` 传递 `--bare` 或设置 `CLAUDE_CODE_SIMPLE=1` 在 [bare mode](/docs/zh-CN/headless#start-faster-with-bare-mode) 中启动 Claude Code，记录保持关闭，除非你设置 `snapshot: true`。

默认记录 `append` 或自定义提示需要 Claude Code v2.1.265 或更高版本，TypeScript Agent SDK 从 v0.3.265 捆绑，Python Agent SDK 从 v0.2.153 捆绑。在 Claude Code v2.1.268 之前，不 [fetch feature flags](/docs/zh-CN/env-vars#features-that-need-feature-flag-fetching) 的会话，包括 Amazon Bedrock、Google Cloud 的 Agent Platform 和 Microsoft Foundry 上的会话，在每个请求上重建提示，`snapshot` 无效。

<h2 id="compare-the-four-approaches">
  比较四种方法
</h2>

这四种自定义方法在存储位置、共享方式以及从 `claude_code` 预设保留的内容方面有所不同。

| 功能        | CLAUDE.md | 输出样式     | 带有追加的 `systemPrompt` | 自定义 `systemPrompt` |
| --------- | --------- | -------- | -------------------- | ------------------ |
| **持久性**   | 每个项目文件    | 保存为文件    | 仅会话                  | 仅会话                |
| **可重用性**  | 每个项目      | 跨项目      | 代码重复                 | 代码重复               |
| **管理**    | 在文件系统上    | CLI + 文件 | 在代码中                 | 在代码中               |
| **默认工具**  | 保留        | 保留       | 保留                   | 丢失（除非包含）           |
| **内置安全**  | 维护        | 维护       | 维护                   | 必须添加               |
| **环境上下文** | 自动        | 自动       | 自动                   | 必须提供               |
| **自定义级别** | 仅添加       | 替换或扩展默认  | 仅添加                  | 完全控制               |
| **版本控制**  | 与项目一起     | 是        | 与代码一起                | 与代码一起              |
| **范围**    | 项目特定      | 用户或项目    | 代码会话                 | 代码会话               |

"带有追加"是指在 TypeScript 中使用 `systemPrompt: { type: "preset", preset: "claude_code", append: "..." }`，或在 Python 中使用 `system_prompt={"type": "preset", "preset": "claude_code", "append": "..."}`。CLAUDE.md 不会改变系统提示本身：SDK 将其内容作为项目上下文注入到对话中。

<h2 id="combine-approaches">
  组合方法
</h2>

这些方法可以组合使用。持久化的输出样式或 CLAUDE.md 设置长期行为，而 `append` 在不触及保存配置的情况下在顶部分层会话特定的指令。

<h3 id="combine-an-output-style-with-session-specific-additions">
  将输出样式与会话特定的添加内容组合
</h3>

下面的示例假设已经激活了代码审查员输出样式。`append` 块在角色的基础上分层会话特定的关注领域，因此单个审查会话可以优先考虑 OAuth 和令牌存储，而无需更改保存的输出样式：

<CodeGroup>
  ```typescript TypeScript theme={null}
  import { query } from "@anthropic-ai/claude-agent-sdk";

  // Assuming "Code Reviewer" output style is active (via /config or settings)
  // Add session-specific focus areas
  const messages = [];

  for await (const message of query({
    prompt: "Review this authentication module",
    options: {
      systemPrompt: {
        type: "preset",
        preset: "claude_code",
        append: `
          For this review, prioritize:
          - OAuth 2.0 compliance
          - Token storage security
          - Session management
        `
      }
    }
  })) {
    messages.push(message);
  }
  ```

  ```python Python theme={null}
  import asyncio

  from claude_agent_sdk import query, ClaudeAgentOptions

  # Assuming "Code Reviewer" output style is active (via /config or settings)
  # Add session-specific focus areas
  messages = []


  async def main():
      async for message in query(
          prompt="Review this authentication module",
          options=ClaudeAgentOptions(
              system_prompt={
                  "type": "preset",
                  "preset": "claude_code",
                  "append": """
                  For this review, prioritize:
                  - OAuth 2.0 compliance
                  - Token storage security
                  - Session management
                  """,
              }
          ),
      ):
          messages.append(message)


  asyncio.run(main())
  ```
</CodeGroup>

<h2 id="see-also">
  另请参阅
</h2>

* [输出样式](/docs/zh-CN/output-styles)：为 CLI 创建、管理和共享输出样式，包括文件格式和存储位置
* [Claude 如何记住您的项目](/docs/zh-CN/memory)：CLAUDE.md 中应放入的内容、放置位置以及如何编写有效的项目说明
* [TypeScript SDK 参考](/docs/zh-CN/agent-sdk/typescript)：完整的 `Options` 类型，包括 `systemPrompt`、`settingSources` 和 `settings`
* [Python SDK 参考](/docs/zh-CN/agent-sdk/python)：完整的 `ClaudeAgentOptions` 类型，包括 `system_prompt` 和 `setting_sources`
* [Settings](/docs/zh-CN/settings)：`settings.json` 参考，包括输出样式和其他配置的存储位置
