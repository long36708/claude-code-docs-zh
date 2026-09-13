---
title: 提示最佳实践
url: https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices
description: 面向 Claude 最新模型的提示工程技术综合指南，涵盖清晰性、示例、XML 结构化、思考以及智能体系统。
---

本文是针对当前 Claude 模型进行提示工程的参考文档，涵盖 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Opus 5、Claude Opus 4.8、Claude Opus 4.7、Claude Opus 4.6、Claude Sonnet 5、Claude Sonnet 4.6 和 Claude Haiku 4.5。本页分为三个部分：

* 首先是\*\*[特定模型指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#model-specific-guidance)\*\*：单个模型在哪些方面表现不同，以及您需要在提示中做出哪些更改。
* 其后是**适用于所有当前模型的技术**：通用原则、输出与格式、工具使用、思考以及智能体系统。
* 最后是**迁移注意事项**，适用于从早期版本迁移过来的提示。

<Tip>
  有关模型能力的概述，请参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)。有关 Claude Fable 5.1 的能力和 API 变更，请参阅 [Claude Fable 5.1 新特性](https://platform.claude.com/docs/zh-CN/models/fable-5-1/whats-new-fable-5-1)。有关 Claude Fable 5 的能力和 API 变更，请参阅 [Claude Fable 5 与 Claude Mythos 5 介绍](https://platform.claude.com/docs/zh-CN/models/fable-5/introducing-claude-fable-5-and-claude-mythos-5)。有关 Claude Sonnet 5 新特性的详细信息，请参阅 [Claude Sonnet 5 新特性](https://platform.claude.com/docs/zh-CN/models/sonnet-5/whats-new-sonnet-5)。有关 Claude Opus 5 新特性的详细信息，请参阅 [Claude Opus 5 新特性](https://platform.claude.com/docs/zh-CN/models/opus-5/whats-new-opus-5)。有关迁移指导，请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。
</Tip>

## 特定模型指南

以下每个模型都有各自的提示页面。请先阅读您所用模型的页面，然后再阅读后续的技术内容。

| 模型                                   | 指南                                                                                                                              | 不同之处                                                                                            |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Claude Fable 5.1 和 Claude Mythos 5.1 | [Claude Fable 5.1 提示指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1) | 与 Claude Fable 5 的差异：effort 级别、完成长任务、面向用户的进度更新、原样回传思考块、智能体循环中的工具调用批处理、低 effort 下的搜索触发、格式以及写作密度。 |
| Claude Fable 5 和 Claude Mythos 5     | [Claude Fable 5 提示指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5)     | 与 Claude Opus 4.8 的差异：effort 级别、指令遵循、长时间运行中的进度声明、记忆系统，以及 `reasoning_extraction` 拒绝类别。           |
| Claude Sonnet 5                      | [Claude Sonnet 5 提示指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-sonnet-5)   | 与 Claude Sonnet 4.6 的差异：响应长度、effort 与思考深度校准、工具使用触发、字面指令遵循，以及设计和前端默认行为。                          |
| Claude Opus 5                        | [Claude Opus 5 提示指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5)       | 与先前 Opus 模型的差异：响应长度与冗长程度、面向用户的进度更新、书面交付物长度、任务范围与过度验证、子智能体控制，以及自我纠正。                             |
| Claude Opus 4.8                      | [Claude Opus 4.8 提示指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-4-8)   | 响应长度、effort 与思考深度校准、工具使用触发、字面指令遵循、子智能体控制，以及设计和前端默认行为。                                           |

## 通用原则

本节及后续各节中的技术适用于当前的 Claude 模型，包括 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5 和 Claude Mythos 5。如果某项技术指明了特定模型，请将其视为在该模型上测得的结果，并在应用于其他模型之前，先用您自己的评估重新验证。

### 清晰直接

Claude 对清晰、明确的指令响应良好。具体说明您期望的输出有助于提升结果。如果您想要"超出预期"的表现，请明确提出要求，而不是依赖模型从模糊的提示中推断。

请把 Claude 想象成一位才华出众但刚入职的新员工，他对您的规范和工作流程缺乏了解。您对需求的解释越精确，结果就越好。

\*\*黄金法则：\*\*把您的提示拿给一位对该任务几乎没有背景了解的同事，请他们照着执行。如果他们会感到困惑，Claude 也会。

* 具体说明期望的输出格式和约束条件。
* 当步骤的顺序或完整性很重要时，使用编号列表或项目符号将指令写成按顺序排列的步骤。

<Accordion title="示例：创建分析仪表盘" defaultOpen>
  **效果较差：**

  ```text wrap
  Create an analytics dashboard
  ```

  **效果更好：**

  ```text wrap
  Create an analytics dashboard. Include as many relevant features and interactions as possible. Go beyond the basics to create a fully-featured implementation.
  ```
</Accordion>

### 添加上下文以提升表现

提供指令背后的上下文或动机，例如向 Claude 解释为什么某种行为很重要，可以帮助 Claude 更好地理解您的目标并给出更有针对性的响应。

<Accordion title="示例：格式偏好" defaultOpen>
  **效果较差：**

  ```text wrap
  NEVER use ellipses
  ```

  **效果更好：**

  ```text wrap
  Your response will be read aloud by a text-to-speech engine, so never use ellipses since the text-to-speech engine will not know how to pronounce them.
  ```
</Accordion>

Claude 足够聪明，能够从解释中进行泛化。

### 有效使用示例

示例是引导 Claude 输出格式、语气和结构最可靠的方法之一。几个精心设计的示例（称为 "few-shot" 或 "multishot prompting"，即少样本或多样本提示）可以提高准确性和一致性。

添加示例时，请确保它们：

* \*\*相关：\*\*紧密贴合您的实际用例。
* \*\*多样：\*\*覆盖边界情况，并具有足够的差异性，以免 Claude 学到非预期的模式。
* \*\*结构化：\*\*将示例包裹在 `<example>` 标签中（多个示例放在 `<examples>` 标签中），以便 Claude 将其与指令区分开来。

<Tip>
  包含 3–5 个示例可获得最佳效果。您也可以请 Claude 评估您的示例的相关性和多样性，或基于您的初始示例集生成更多示例。
</Tip>

### 使用 XML 标签构建提示结构

XML 标签有助于 Claude 无歧义地解析复杂提示，尤其是当您的提示混合了指令、上下文、示例和可变输入时。将每种类型的内容包裹在各自的标签中（例如 `<instructions>`、`<context>`、`<input>`）可以减少误解。

最佳实践：

* 在您的各个提示中使用一致且具有描述性的标签名称。
* 当内容具有天然的层级结构时嵌套标签（文档放在 `<documents>` 内，每个文档放在 `<document index="n">` 内）。

### 为 Claude 设定角色

在 system prompt（系统提示）中设定角色，可以使 Claude 的行为和语气聚焦于您的用例。即使只有一句话也会产生效果：

<CodeGroup>
  ```bash cURL
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "system": "You are a helpful coding assistant specializing in Python.",
      "messages": [
        {"role": "user", "content": "How do I sort a list of dictionaries by key?"}
      ]
    }'
  ```

  ```bash CLI
  ant messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --system "You are a helpful coding assistant specializing in Python." \
    --message '{role: user, content: "How do I sort a list of dictionaries by key?"}'
  ```

  ```python Python
  client = anthropic.Anthropic()

  message = client.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      system="You are a helpful coding assistant specializing in Python.",
      messages=[
          {"role": "user", "content": "How do I sort a list of dictionaries by key?"}
      ],
  )

  print(message.content)
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const message = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    system: "You are a helpful coding assistant specializing in Python.",
    messages: [{ role: "user", content: "How do I sort a list of dictionaries by key?" }]
  });

  console.log(message.content);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var parameters = new MessageCreateParams
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      System = "You are a helpful coding assistant specializing in Python.",
      Messages =
      [
          new() { Role = Role.User, Content = "How do I sort a list of dictionaries by key?" }
      ]
  };

  var message = await client.Messages.Create(parameters);
  Console.WriteLine(message);
  ```

  ```go Go
  client := anthropic.NewClient()

  message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus5,
  	MaxTokens: 1024,
  	System: []anthropic.TextBlockParam{
  		{Text: "You are a helpful coding assistant specializing in Python."},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("How do I sort a list of dictionaries by key?")),
  	},
  })
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(message.Content)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  MessageCreateParams params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(1024)
      .system("You are a helpful coding assistant specializing in Python.")
      .addUserMessage("How do I sort a list of dictionaries by key?")
      .build();

  Message message = client.messages().create(params);
  System.out.println(message.content());
  ```

  ```php PHP
  $client = new Client();

  $message = $client->messages->create(
      maxTokens: 1024,
      messages: [
          ['role' => 'user', 'content' => 'How do I sort a list of dictionaries by key?']
      ],
      model: 'claude-opus-5',
      system: 'You are a helpful coding assistant specializing in Python.',
  );

  echo json_encode($message->content, JSON_PRETTY_PRINT), PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  message = client.messages.create(
    model: "claude-opus-5",
    max_tokens: 1024,
    system: "You are a helpful coding assistant specializing in Python.",
    messages: [
      { role: "user", content: "How do I sort a list of dictionaries by key?" }
    ]
  )

  puts message.content
  ```
</CodeGroup>

### 长上下文提示

在处理大型文档或数据密集型输入（20k+ 令牌）时，请仔细构建提示结构以获得最佳结果：

* \*\*将长篇数据放在顶部：\*\*将长文档和输入放在提示的靠前位置，置于查询、指令和示例之上。这可以提升所有模型的表现。

  <Note>
    在测试中，将查询放在末尾可将响应质量提升最多 30%，尤其是在处理复杂的多文档输入时。
  </Note>

* \*\*使用 XML 标签组织文档内容和元数据：\*\*使用多个文档时，将每个文档包裹在 `<document>` 标签中，并使用 `<document_content>` 和 `<source>`（以及其他元数据）子标签以保持清晰。

  <Accordion title="多文档结构示例">
    ```xml
    <documents>
      <document index="1">
        <source>annual_report_2023.pdf</source>
        <document_content>
          {{ANNUAL_REPORT}}
        </document_content>
      </document>
      <document index="2">
        <source>competitor_analysis_q2.xlsx</source>
        <document_content>
          {{COMPETITOR_ANALYSIS}}
        </document_content>
      </document>
    </documents>

    Analyze the annual report and competitor analysis. Identify strategic advantages and recommend Q3 focus areas.
    ```
  </Accordion>

* \*\*以引文为依据作答：\*\*对于长文档任务，请让 Claude 在执行任务之前先引用文档中的相关部分。这有助于 Claude 聚焦于相关内容并忽略文档的其余部分。

  <Accordion title="引文提取示例">
    ```xml
    You are an AI physician's assistant. Your task is to help doctors diagnose possible patient illnesses.

    <documents>
      <document index="1">
        <source>patient_symptoms.txt</source>
        <document_content>
          {{PATIENT_SYMPTOMS}}
        </document_content>
      </document>
      <document index="2">
        <source>patient_records.txt</source>
        <document_content>
          {{PATIENT_RECORDS}}
        </document_content>
      </document>
      <document index="3">
        <source>patient01_appt_history.txt</source>
        <document_content>
          {{PATIENT01_APPOINTMENT_HISTORY}}
        </document_content>
      </document>
    </documents>

    Find quotes from the patient records and appointment history that are relevant to diagnosing the patient's reported symptoms. Place these in <quotes> tags. Then, based on these quotes, list all information that would help the doctor diagnose the patient's symptoms. Place your diagnostic information in <info> tags.
    ```
  </Accordion>

### 模型自我认知

如果您希望 Claude 在您的应用中正确标识自身或使用特定的 API 字符串：

```text Sample prompt for model identity wrap
The assistant is Claude, created by Anthropic. The current model is Claude Opus 5.
```

对于需要指定模型字符串的 LLM 驱动应用：

```text Sample prompt for model string wrap
When an LLM is needed, please default to Claude Opus 5 unless the user requests
otherwise. The exact model string for Claude Opus 5 is claude-opus-5.
```

## 输出与格式

### 沟通风格与冗长程度

与先前的模型相比，Claude 的最新模型具有更简洁、更自然的沟通风格：

* \*\*更直接、更有依据：\*\*提供基于事实的进度报告，而非自我夸耀式的更新
* \*\*更具对话感：\*\*略微更流畅、更口语化，更少机器感
* \*\*更不冗长：\*\*除非另有提示，否则可能为了效率而跳过详细总结

这意味着 Claude 可能会在工具调用后跳过文字总结，直接进入下一个操作。如果您希望更多地了解其推理过程：

```text Sample prompt wrap
After completing a task that involves tool use, provide a quick summary of the work you've done.
```

Claude Opus 5 在冗长程度上是个例外：其默认的面向用户响应比先前模型更长，而且提高或降低 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 并不能可靠地改变可见响应的长度。请改为明确提示要求简洁。示例指令请参阅 [Claude Opus 5 提示指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#response-length-and-verbosity)。Claude Fable 5.1 在智能体工作中则有相反的倾向：它在工具调用之间写出的面向用户更新更少。请明确要求输出进度文本，并删除任何要求其保持该文本简短的指令。请参阅[要求提供面向用户的进度更新](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#ask-for-user-facing-progress-updates)。

### 控制响应格式

有几种特别有效的方法可以引导输出格式：

1. **告诉 Claude 该做什么，而不是不该做什么**

   * 不要说："不要在响应中使用 markdown"
   * 尝试说："您的响应应由流畅连贯的散文段落组成。"

2. **使用 XML 格式指示符**

   * 尝试说："将响应中的散文部分写在 \<smoothly\_flowing\_prose\_paragraphs> 标签中。"

3. **使提示风格与期望输出相匹配**

   提示中使用的格式风格可能会影响 Claude 的响应风格。如果您在输出格式的可引导性方面仍然遇到问题，请尝试让提示风格尽可能贴近您期望的输出风格。例如，从提示中移除 markdown 可以减少输出中 markdown 的数量。

4. **针对特定格式偏好使用详细提示**

   如需对 markdown 和格式使用进行更多控制，请提供明确的指导：

````text Sample prompt to minimize markdown wrap
<avoid_excessive_markdown_and_bullet_points>
When writing reports, documents, technical explanations, analyses, or any long-form
content, write in clear, flowing prose using complete paragraphs and sentences. Use
standard paragraph breaks for organization and reserve markdown primarily for `inline
code`, code blocks (```...```), and simple headings (## and ###). Avoid using **bold**
and *italics*.

DO NOT use ordered lists (1. ...) or unordered lists (*) unless: a) you're presenting
truly discrete items where a list format is the best option, or b) the user explicitly
requests a list or ranking

Instead of listing items with bullets or numbers, incorporate them naturally into
sentences. This guidance applies especially to technical writing. Using prose instead of
excessive formatting will improve user satisfaction. NEVER output a series of overly
short bullet points.

Your goal is readable, flowing text that guides the reader naturally through ideas
rather than fragmenting information into isolated points.
</avoid_excessive_markdown_and_bullet_points>
````

Claude Fable 5.1 的格式化程度本就低于早期模型，因此在该模型上，像这样的指令块可能会抑制内容所需的结构。请将其删除，或替换为[聊天中的格式](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#formatting-in-chat)中更简短的规则。

### LaTeX 输出

Claude 的最新模型默认使用 LaTeX 来表示数学表达式、方程和技术说明。如果您更喜欢纯文本，请在提示中添加以下指令：

```text Sample prompt wrap
Format your response in plain text only. Do not use LaTeX, MathJax, or any markup
notation such as \( \), $, or \frac{}{}. Write all math expressions using standard text
characters (e.g., "/" for division, "*" for multiplication, and "^" for exponents).
```

### 文档创建

Claude 的最新模型在创建演示文稿、动画和可视化文档时具有很强的指令遵循能力，通常第一次尝试就能产出可用的结果。

要在文档创建中获得最佳效果：

```text Sample prompt wrap
Create a professional presentation on [topic]. Include thoughtful design elements,
visual hierarchy, and engaging animations where appropriate.
```

### 从预填充响应迁移

从 Claude 4.6 模型和 [Claude Mythos Preview](https://anthropic.com/glasswing) 开始，不再支持在最后一个 assistant 轮次上使用预填充响应（即提供部分 assistant 消息供 Claude 续写）。向这些模型发送带有预填充 assistant 消息的请求将返回 400 错误。模型智能和指令遵循能力已经进步到大多数预填充用例不再需要它的程度。早期模型继续支持预填充，在对话其他位置添加 assistant 消息不受影响。

以下是常见的预填充场景以及如何从中迁移：

<Accordion title="控制输出格式">
  预填充曾被用于强制特定的输出格式，如 JSON/YAML、分类以及类似的模式，即通过预填充将 Claude 约束到特定结构。

  **迁移方法：**[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)功能专为约束 Claude 的响应遵循给定 schema 而设计。请先尝试要求模型遵循您的输出结构，因为较新的模型在被告知时能够可靠地匹配复杂的 schema，尤其是在配合重试实现的情况下。对于分类任务，请使用带有包含有效标签的 enum 字段的工具，或使用结构化输出。
</Accordion>

<Accordion title="消除开场白">
  像 `Here is the requested summary:\n` 这样的预填充曾被用于跳过引导性文字。

  \*\*迁移方法：\*\*在系统提示中使用直接指令："直接回答，不要开场白。不要以'以下是……'、'根据……'等短语开头。"或者，指示模型在 XML 标签内输出、使用结构化输出，或使用工具调用。如果偶尔仍有开场白漏出，请在后处理中将其去除。
</Accordion>

<Accordion title="避免不当拒绝">
  预填充曾被用于绕开不必要的拒绝。

  \*\*迁移方法：\*\*Claude 现在在恰当拒绝方面表现好得多。在 `user` 消息中给出清晰的提示而不使用预填充应该就足够了。
</Accordion>

<Accordion title="续写">
  预填充曾被用于续写部分补全、恢复被中断的响应，或从上一次生成停止的地方继续。

  \*\*迁移方法：\*\*将续写移至 user 消息中，并包含被中断响应的末尾文本："您之前的响应被中断，结尾是 \`\[previous\_response]\`。请从中断处继续。"如果这是错误处理或不完整响应处理的一部分，且没有用户体验上的代价，请重试该请求。
</Accordion>

<Accordion title="上下文补充与角色一致性">
  预填充曾被用于定期确保上下文得到刷新或注入。

  \*\*迁移方法：\*\*对于非常长的对话，将之前通过预填充 assistant 消息实现的提醒注入到 user 轮次中。如果上下文补充是更复杂的智能体系统的一部分，请考虑通过工具进行补充（根据轮次数等启发式规则，暴露或鼓励使用包含上下文的工具），或在[上下文压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)期间进行补充。
</Accordion>

## 工具使用

### 工具用法

Claude 的最新模型经过精确指令遵循的训练，明确指示其使用特定工具会有所助益。如果您说"你能建议一些修改吗"，Claude 有时会提供建议而不是直接实施，即使进行修改可能正是您的本意。要了解如何定义工具以及排查工具触发问题，请参阅 [Claude 的 tool use（工具使用）](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。

要让 Claude 采取行动，请更加明确：

<Accordion title="示例：明确的指令" defaultOpen>
  **效果较差（Claude 只会给出建议）：**

  ```text wrap
  Can you suggest some changes to improve this function?
  ```

  **效果更好（Claude 会进行修改）：**

  ```text wrap
  Change this function to improve its performance.
  ```

  或者：

  ```text wrap
  Make these edits to the authentication flow.
  ```
</Accordion>

要让 Claude 默认更主动地采取行动，您可以将以下内容添加到系统提示中：

```text Sample prompt for proactive action wrap
<default_to_action>
By default, implement changes rather than only suggesting them. If the user's intent is
unclear, infer the most useful likely action and proceed, using tools to discover any
missing details instead of guessing. Try to infer the user's intent about whether a tool
call (e.g., file edit or read) is intended or not, and act accordingly.
</default_to_action>
```

另一方面，如果您希望模型默认更加谨慎、不那么急于直接进入实现，并且只在被要求时才采取行动，您可以使用如下提示来引导这种行为：

```text Sample prompt for conservative action wrap
<do_not_act_before_instructions>
Do not jump into implementation or change files unless clearly instructed to make
changes. When the user's intent is ambiguous, default to providing information, doing
research, and providing recommendations rather than taking action. Only proceed with
edits, modifications, or implementations when the user explicitly requests them.
</do_not_act_before_instructions>
```

Claude Opus 4.5 和 Claude Opus 4.6 对系统提示的响应也比先前模型更敏感。如果您的提示原本是为了减少工具或技能触发不足而设计的，这些模型现在可能会过度触发。解决方法是收敛任何激进的措辞。原本您可能会说"关键：您必须在……时使用此工具"，现在可以使用更平常的提示，如"在……时使用此工具"。

### 优化并行工具调用

Claude 的最新模型会并行运行相互独立的工具调用。这些模型会：

* 在研究过程中运行多个推测性搜索
* 一次读取多个文件以更快地构建上下文
* 并行运行 bash 命令（这甚至可能成为系统性能的瓶颈）

这种行为是可引导的。虽然模型在没有提示的情况下并行工具调用的成功率已经很高，但您可以将其提升至约 100% 或调整其激进程度：

```text Sample prompt for maximum parallel efficiency wrap
<use_parallel_tool_calls>
If you intend to call multiple tools and there are no dependencies between the tool
calls, make all of the independent tool calls in parallel. Prioritize calling tools
simultaneously whenever the actions can be done in parallel rather than sequentially.
For example, when reading 3 files, run 3 tool calls in parallel to read all 3 files into
context at the same time. Maximize use of parallel tool calls where possible to increase
speed and efficiency. However, if some tool calls depend on previous calls to inform
dependent values like the parameters, do NOT call these tools in parallel and instead
call them sequentially. Never use placeholders or guess missing parameters in tool
calls.
</use_parallel_tool_calls>
```

```text Sample prompt to reduce parallel execution wrap
Execute operations sequentially with brief pauses between each step to ensure stability.
```

在 Claude Fable 5.1 的长智能体循环中，请在每轮工具结果之后，将并行调用指令作为轮次范围的系统消息发送。请参阅[在智能体循环中批量处理独立的工具调用](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#batch-independent-tool-calls-in-agent-loops)。

## 思考与推理

### 过度思考与过度周全

Claude Opus 4.6 比先前模型进行更多的前期探索，尤其是在较高的 [`effort`](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 设置下。这些初始工作通常有助于优化最终结果，但模型可能会在未经提示的情况下收集大量上下文或同时追踪多条研究线索。如果您的提示之前鼓励模型更加周全，您应该针对 Claude Opus 4.6 调整该指导：

* \*\*用更有针对性的指令替换笼统的默认设置。\*\*不要说"默认使用 \[tool]"，而是添加类似"当 \[tool] 能增进您对问题的理解时使用它"的指导。
* \*\*移除过度提示。\*\*在先前模型中触发不足的工具现在很可能会恰当触发。像"如有疑问，请使用 \[tool]"这样的指令会导致过度触发。
* \*\*将 effort 作为后备手段。\*\*如果 Claude 仍然过于激进，请使用较低的 `effort` 设置。

在某些情况下，Claude Opus 4.6 可能会进行大量思考，这会增加思考令牌并拖慢响应。如果不希望出现这种行为，您可以添加明确的指令来约束其推理，或者降低 `effort` 设置以减少整体思考和令牌用量。

```text Sample prompt wrap
When you're deciding how to approach a problem, choose an approach and commit to it.
Avoid revisiting decisions unless you encounter new information that directly
contradicts your reasoning. If you're weighing two approaches, pick one and see it
through. You can always course-correct later if the chosen approach fails.
```

如果您需要对思考成本设置硬性上限，带 `budget_tokens` 上限的 extended thinking（扩展思考）在 Opus 4.6 和 Sonnet 4.6 上仍然可用，但已被弃用。在 Claude 4.7 及更高版本的模型上，设置 `budget_tokens` 会返回 400 错误。建议优先降低 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort) 设置，或配合[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)使用 `max_tokens` 作为硬性限制。

### 利用思考与交错思考能力

Claude 的最新模型提供思考能力，这对于涉及工具使用后反思或复杂多步推理的任务尤其有帮助。您可以引导其初始思考或交错思考以获得更好的结果。

Claude 4.6 及更高版本的模型以及 Claude Mythos Preview 使用[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)（`thinking: {type: "adaptive"}`），由 Claude 动态决定何时思考以及思考多少。在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5 和 Claude Mythos 5 上，思考始终开启，且自适应思考是唯一的模式。Claude 根据两个因素校准其思考：`effort` 参数和查询复杂度。更高的 effort 会引发更多思考，更复杂的查询也是如此。对于不需要思考的简单查询，模型会直接响应。在内部评估中，自适应思考始终比扩展思考带来更好的表现。请考虑迁移到自适应思考。

对于需要智能体行为的工作负载，如多步工具使用、复杂编码任务和长周期智能体循环，请使用自适应思考。较旧的模型使用带 `budget_tokens` 的手动[扩展思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)；有关每个模型接受哪种配置，请参阅[各模型配置表](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-troubleshooting#supported-models)。

您可以引导 Claude 的思考行为：

```text Example prompt wrap
After receiving tool results, carefully reflect on their quality and determine optimal
next steps before proceeding. Use your thinking to plan and iterate based on this new
information, and then take the best next action.
```

自适应思考的触发行为是可通过提示调整的。如果您发现模型思考的频率高于您的期望（这在系统提示庞大或复杂时可能发生），请添加指导加以引导：

```text Sample prompt wrap
Thinking adds latency and should only be used when it will meaningfully improve
answer quality - typically for problems that require multistep reasoning. When in
doubt, respond directly.
```

如果您正在从带 `budget_tokens` 的[扩展思考](https://platform.claude.com/docs/zh-CN/build-with-claude/extended-thinking)迁移，请替换您的思考配置并将预算控制移至 `effort`。以下示例展示了迁移前后的同一请求（有关可用级别和各模型的可用性，请参阅 [effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)）：

<CodeGroup>
  ```bash cURL
  # 之前：使用手动预算的扩展思考（旧模型）
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-sonnet-4-5-20250929",
      "max_tokens": 16000,
      "thinking": {"type": "enabled", "budget_tokens": 10000},
      "messages": [
        {"role": "user", "content": "..."}
      ]
    }'

  # 之后：使用 effort 的自适应思考
  curl https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -d '{
      "model": "claude-opus-4-8",
      "max_tokens": 16000,
      "thinking": {"type": "adaptive"},
      "output_config": {"effort": "high"},
      "messages": [
        {"role": "user", "content": "..."}
      ]
    }'
  ```

  ```bash CLI
  # 之前：使用手动预算的扩展思考（旧模型）
  ant messages create <<'YAML'
  model: claude-sonnet-4-5-20250929
  max_tokens: 16000
  thinking:
    type: enabled
    budget_tokens: 10000
  messages:
    - role: user
      content: "..."
  YAML

  # 之后：使用 effort 的自适应思考
  ant messages create <<'YAML'
  model: claude-opus-4-8
  max_tokens: 16000
  thinking:
    type: adaptive
  output_config:
    effort: high
  messages:
    - role: user
      content: "..."
  YAML
  ```

  ```python Python
  # 之前：使用手动预算的扩展思考（旧模型）
  client.messages.create(
      model="claude-sonnet-4-5-20250929",
      max_tokens=16000,
      thinking={"type": "enabled", "budget_tokens": 10000},
      messages=[{"role": "user", "content": "..."}],
  )

  # 之后：使用 effort 的自适应思考
  client.messages.create(
      model="claude-opus-4-8",
      max_tokens=16000,
      thinking={"type": "adaptive"},
      output_config={"effort": "high"},
      messages=[{"role": "user", "content": "..."}],
  )
  ```

  ```typescript TypeScript
  // 之前：使用手动预算的扩展思考（旧模型）
  await client.messages.create({
    model: "claude-sonnet-4-5-20250929",
    max_tokens: 16000,
    thinking: { type: "enabled", budget_tokens: 10000 },
    messages: [{ role: "user", content: "..." }]
  });

  // 之后：使用 effort 的自适应思考
  await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 16000,
    thinking: { type: "adaptive" },
    output_config: { effort: "high" },
    messages: [{ role: "user", content: "..." }]
  });
  ```

  ```csharp C#
  // 之前：使用手动预算的扩展思考（旧模型）
  await client.Messages.Create(new MessageCreateParams
  {
      Model = "claude-sonnet-4-5-20250929",
      MaxTokens = 16000,
      Thinking = new ThinkingConfigEnabled(budgetTokens: 10000),
      Messages = [new() { Role = Role.User, Content = "..." }]
  });

  // 之后：使用 effort 的自适应思考
  await client.Messages.Create(new MessageCreateParams
  {
      Model = Model.ClaudeOpus4_8,
      MaxTokens = 16000,
      Thinking = new ThinkingConfigAdaptive(),
      OutputConfig = new OutputConfig { Effort = Effort.High },
      Messages = [new() { Role = Role.User, Content = "..." }]
  });
  ```

  ```go Go
  // 之前：使用手动预算的扩展思考（旧模型）
  client.Messages.New(ctx, anthropic.MessageNewParams{
  	Model:     "claude-sonnet-4-5-20250929",
  	MaxTokens: 16000,
  	Thinking: anthropic.ThinkingConfigParamUnion{
  		OfEnabled: &anthropic.ThinkingConfigEnabledParam{BudgetTokens: 10000},
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("...")),
  	},
  })

  // 之后：使用 effort 的自适应思考
  client.Messages.New(ctx, anthropic.MessageNewParams{
  	Model:     anthropic.ModelClaudeOpus4_8,
  	MaxTokens: 16000,
  	Thinking: anthropic.ThinkingConfigParamUnion{
  		OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{},
  	},
  	OutputConfig: anthropic.OutputConfigParam{
  		Effort: anthropic.OutputConfigEffortHigh,
  	},
  	Messages: []anthropic.MessageParam{
  		anthropic.NewUserMessage(anthropic.NewTextBlock("...")),
  	},
  })
  ```

  ```java Java
  // 之前：使用手动预算的扩展思考（旧模型）
  client.messages().create(MessageCreateParams.builder()
      .model("claude-sonnet-4-5-20250929")
      .maxTokens(16000L)
      .thinking(ThinkingConfigEnabled.builder().budgetTokens(10000L).build())
      .addUserMessage("...")
      .build());

  // 之后：使用 effort 的自适应思考
  client.messages().create(MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_4_8)
      .maxTokens(16000L)
      .thinking(ThinkingConfigAdaptive.builder().build())
      .outputConfig(OutputConfig.builder()
          .effort(OutputConfig.Effort.HIGH)
          .build())
      .addUserMessage("...")
      .build());
  ```

  ```php PHP
  // 之前：使用手动预算的扩展思考（旧模型）
  $client->messages->create(
      model: 'claude-sonnet-4-5-20250929',
      maxTokens: 16000,
      thinking: ['type' => 'enabled', 'budget_tokens' => 10000],
      messages: [['role' => 'user', 'content' => '...']],
  );

  // 之后：使用 effort 的自适应思考
  $client->messages->create(
      model: 'claude-opus-4-8',
      maxTokens: 16000,
      thinking: ['type' => 'adaptive'],
      outputConfig: ['effort' => 'high'],
      messages: [['role' => 'user', 'content' => '...']],
  );
  ```

  ```ruby Ruby
  # 之前：使用手动预算的扩展思考（旧模型）
  client.messages.create(
    model: "claude-sonnet-4-5-20250929",
    max_tokens: 16000,
    thinking: { type: "enabled", budget_tokens: 10000 },
    messages: [{ role: "user", content: "..." }]
  )

  # 之后：使用 effort 的自适应思考
  client.messages.create(
    model: "claude-opus-4-8",
    max_tokens: 16000,
    thinking: { type: "adaptive" },
    output_config: { effort: "high" },
    messages: [{ role: "user", content: "..." }]
  )
  ```
</CodeGroup>

如果您没有使用扩展思考，则无需任何更改。在 Claude Opus 4.6 至 Claude Opus 4.8 以及 Claude Sonnet 4.6 上，省略 `thinking` 参数时思考处于关闭状态。在 Claude Opus 5 和 Claude Sonnet 5 上，省略 `thinking` 参数时思考默认开启。在 Claude Opus 5 上，您只能在 effort 为 `high` 或更低时禁用它。在 Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5 和 Claude Mythos 5 上，无论您是否设置 `thinking` 参数，思考始终开启。

* \*\*优先使用通用指令而非规定性步骤。\*\*像"深入思考"这样的提示通常比手写的分步计划产生更好的推理。Claude 的推理常常超出人类所能规定的范围。
* \*\*多样本示例可与思考配合使用。\*\*在少样本示例中使用 `<thinking>` 标签向 Claude 展示推理模式。它会将这种风格泛化到自己的扩展思考块中。
* \*\*将手动思维链（CoT）提示作为后备手段。\*\*当思考关闭时，您仍然可以通过要求 Claude 逐步思考问题来鼓励分步推理。使用 `<thinking>` 和 `<answer>` 等结构化标签将推理与最终输出清晰分开。在 Claude Opus 5 上，建议改为在较低的 effort 级别下保持思考开启：在思考禁用时，模型偶尔会将内部 XML 标签输出到可见输出中，因此在该模型上应用此模式之前，请先参阅[在思考禁用状态下运行](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#running-with-thinking-disabled)。
* \*\*让 Claude 自我检查。\*\*追加类似"在完成之前，请对照 \[测试标准] 验证您的答案"的内容。这能可靠地捕获错误，尤其是在编码和数学方面。Claude Opus 5 是例外：它无需明确指令就能很好地验证自己的工作，而从为早期模型调优的提示中沿用下来的验证指令可能导致过度验证，增加令牌和延迟。迁移到 Claude Opus 5 时，请删除这些指令而不是重写它们。请参阅[任务范围与过度验证](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#task-scope-and-over-verification)。

<Note>
  当扩展思考被禁用时，Claude Opus 4.5 对"think"一词及其变体特别敏感。在这些情况下，请考虑使用"consider"、"evaluate"或"reason through"等替代词。
</Note>

<Info>
  有关思考能力的更多信息，请参阅[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)和[引导思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking-steering-and-cost)。
</Info>

## 智能体系统

### 长周期推理与状态跟踪

Claude 的最新模型能够以强大的状态跟踪能力处理长周期推理任务。Claude 通过专注于增量进展来在长时间会话中保持方向感，每次稳步推进少数几件事，而不是试图一次完成所有事情。这种能力在跨越多个 context window（上下文窗口）或任务迭代时尤为突出，Claude 可以处理复杂任务、保存状态，然后在全新的上下文窗口中继续。

#### 上下文感知与多窗口工作流

Claude Sonnet 5、Claude Sonnet 4.6、Claude Sonnet 4.5 和 Claude Haiku 4.5 具备[上下文感知](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows#context-awareness)能力，使模型能够在整个对话过程中跟踪其剩余的上下文窗口（即其"令牌预算"）。这使 Claude 能够通过了解自己还有多少工作空间来更有效地执行任务和管理上下文。

**管理上下文限制：**

如果您在会压缩上下文或允许将上下文保存到外部文件的智能体框架（如 Claude Code）中使用 Claude，请考虑将此信息添加到提示中，以便 Claude 相应地行事。否则，Claude 有时可能会在接近上下文限制时自然地尝试收尾工作。以下是一个示例提示：

```text Sample prompt wrap
Your context window will be automatically compacted as it approaches its limit, allowing
you to continue working indefinitely from where you left off. Therefore, do not stop
tasks early due to token budget concerns. As you approach your token budget limit, save
your current progress and state to memory before the context window refreshes. Always be
as persistent and autonomous as possible and complete tasks fully, even if the end of
your budget is approaching. Never artificially stop any task early regardless of the
context remaining.
```

[记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)与上下文感知搭配使用，非常适合管理上下文切换。

#### 跨多个上下文窗口的工作流

对于跨越多个上下文窗口的任务：

1. \*\*为第一个上下文窗口使用不同的提示：\*\*使用第一个上下文窗口搭建框架（编写测试、创建设置脚本），然后使用后续的上下文窗口在待办列表上迭代。

2. \*\*让模型以结构化格式编写测试：\*\*要求 Claude 在开始工作前创建测试，并以结构化格式（例如 `tests.json`）跟踪它们。这有助于提升长期迭代能力。提醒 Claude 测试的重要性："删除或编辑测试是不可接受的，因为这可能导致功能缺失或存在缺陷。"

3. \*\*设置便利工具：\*\*鼓励 Claude 创建设置脚本（例如 `init.sh`）以优雅地启动服务器、运行测试套件和代码检查工具。这可以避免从全新上下文窗口继续时的重复工作。

4. \*\*全新开始与压缩的对比：\*\*当上下文窗口被清空时，请考虑从一个全新的上下文窗口开始，而不是使用压缩。Claude 的最新模型在从本地文件系统发现状态方面极为高效。在某些情况下，您可能希望利用这一点而非压缩。请明确规定它应如何开始：

   * "调用 pwd；您只能在此目录中读写文件。"
   * "查看 progress.txt、tests.json 和 git 日志。"
   * "在继续实现新功能之前，先手动运行一遍基础集成测试。"

5. \*\*提供验证工具：\*\*随着自主任务长度的增加，Claude 需要在没有持续人工反馈的情况下验证正确性。能让 Claude 验证 UI 工作的工具很有帮助，例如[计算机使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)、[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)或浏览器自动化 MCP 服务器。

6. \*\*鼓励充分利用上下文：\*\*提示 Claude 在继续之前高效地完成各个组件：

```text Sample prompt wrap
This is a very long task, so it may be beneficial to plan out your work clearly. It's
encouraged to spend your entire output context working on the task - just make sure you
don't run out of context with significant uncommitted work. Continue working
systematically until you have completed this task.
```

#### 状态管理最佳实践

* \*\*对状态数据使用结构化格式：\*\*在跟踪结构化信息（如测试结果或任务状态）时，使用 JSON 或其他结构化格式来帮助 Claude 理解 schema 要求。
* \*\*对进度笔记使用非结构化文本：\*\*自由格式的进度笔记非常适合跟踪总体进展和上下文。
* \*\*使用 git 进行状态跟踪：\*\*Git 提供了已完成工作的日志和可恢复的检查点。Claude 的最新模型在使用 git 跨多个会话跟踪状态方面表现尤为出色。
* \*\*强调增量进展：\*\*明确要求 Claude 跟踪其进度并专注于增量工作。

<Accordion title="示例：状态跟踪">
  ```json tests.json
  {
    "tests": [
      { "id": 1, "name": "authentication_flow", "status": "passing" },
      { "id": 2, "name": "user_management", "status": "failing" },
      { "id": 3, "name": "api_endpoints", "status": "not_started" }
    ],
    "total": 200,
    "passing": 150,
    "failing": 25,
    "not_started": 25
  }
  ```

  ```text wrap
  // Progress notes (progress.txt)
  Session 3 progress:
  - Fixed authentication token validation
  - Updated user model to handle edge cases
  - Next: investigate user_management test failures (test #2)
  - Note: Do not remove tests as this could lead to missing functionality
  ```
</Accordion>

### 平衡自主性与安全性

在没有指导的情况下，Claude Opus 4.6 可能会采取难以撤销或影响共享系统的操作，例如删除文件、强制推送或向外部服务发布内容。如果您希望 Claude Opus 4.6 在采取潜在风险操作之前进行确认，请在提示中添加指导：

```text Sample prompt wrap
Consider the reversibility and potential impact of your actions. You are encouraged to
take local, reversible actions like editing files or running tests, but for actions that
are hard to reverse, affect shared systems, or could be destructive, ask the user before
proceeding.

Examples of actions that warrant confirmation:
- Destructive operations: deleting files or branches, dropping database tables, rm -rf
- Hard to reverse operations: git push --force, git reset --hard, amending published commits
- Operations visible to others: pushing code, commenting on PRs/issues, sending
messages, modifying shared infrastructure

When encountering obstacles, do not use destructive actions as a shortcut. For example,
don't bypass safety checks (e.g. --no-verify) or discard unfamiliar files that may be
in-progress work.
```

### 研究与信息收集

Claude 的最新模型能够有效地从多个来源查找和综合信息。要获得最佳研究结果：

1. \*\*提供清晰的成功标准：\*\*定义什么样的答案算是成功回答了您的研究问题。

2. \*\*鼓励来源验证：\*\*要求 Claude 跨多个来源验证信息。

3. **对于复杂的研究任务，使用结构化方法：**

```text Sample prompt for complex research wrap
Search for this information in a structured way. As you gather data, develop several
competing hypotheses. Track your confidence levels in your progress notes to improve
calibration. Regularly self-critique your approach and plan. Update a hypothesis tree or
research notes file to persist information and provide transparency. Break down this
complex research task systematically.
```

这种结构化方法有助于 Claude 有条不紊地处理大型语料库，并迭代地审视其发现。

### 子智能体编排

Claude 的最新模型原生支持编排子智能体。这些模型能够识别何时将工作委派给专门的子智能体会对任务有益，并在无需明确指令的情况下主动这样做。

要利用这种行为：

1. \*\*确保子智能体工具定义良好：\*\*提供子智能体工具并在工具定义中加以描述。
2. \*\*让 Claude 自然地编排：\*\*Claude 会在没有明确指令的情况下恰当地进行委派。
3. \*\*注意过度使用：\*\*Claude Opus 4.6 对子智能体有强烈偏好，可能会在更简单、直接的方法就足够的情况下生成子智能体。例如，当直接调用 grep 更快且足够时，模型可能仍会为代码探索生成子智能体。Claude Opus 5 也比先前模型更容易委派给子智能体；有关指导和示例抑制提示，请参阅[控制子智能体生成](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5#controlling-subagent-spawning)。

如果您发现子智能体使用过度，请添加关于何时需要、何时不需要子智能体的明确指导：

```text Sample prompt for subagent usage wrap
Use subagents when tasks can run in parallel, require isolated context, or involve
independent workstreams that don't need to share state. For simple tasks, sequential
operations, single-file edits, or tasks where you need to maintain context across steps,
work directly rather than delegating.
```

### 链接复杂提示

借助自适应思考和子智能体编排，Claude 可以在内部处理大多数多步推理。当您需要检查中间输出或强制执行特定的流水线结构时，显式的提示链（将任务拆分为连续的 API 调用）仍然有用。

最常见的链式模式是\*\*自我纠正：\*\*生成草稿 → 让 Claude 对照标准审查 → 让 Claude 根据审查结果进行完善。每一步都是单独的 API 调用，因此您可以在任意节点记录、评估或分支。

### 减少智能体编码中的文件创建

Claude 的最新模型有时可能会出于测试和迭代目的创建新文件，尤其是在处理代码时。这种方法使 Claude 能够在保存最终输出之前将文件（尤其是 Python 脚本）用作"临时草稿本"。使用临时文件可以改善结果，尤其是在智能体编码用例中。

如果您希望尽量减少净新增文件的创建，可以指示 Claude 自行清理：

```text Sample prompt wrap
If you create any temporary new files, scripts, or helper files for iteration, clean up
these files by removing them at the end of the task.
```

### 过度积极

Claude Opus 4.5 和 Claude Opus 4.6 有过度工程化的倾向，表现为创建额外文件、添加不必要的抽象，或构建未被要求的灵活性。如果您看到这种不期望的行为，请添加具体指导以保持解决方案的精简。

例如：

```text Sample prompt to minimize overengineering wrap
Avoid over-engineering. Only make changes that are directly requested or clearly
necessary. Keep solutions simple and focused:

- Scope: Don't add features, refactor code, or make "improvements" beyond what was
asked. A bug fix doesn't need surrounding code cleaned up. A simple feature doesn't need
extra configurability.

- Documentation: Don't add docstrings, comments, or type annotations to code you didn't
change. Only add comments where the logic isn't self-evident.

- Defensive coding: Don't add error handling, fallbacks, or validation for scenarios
that can't happen. Trust internal code and framework guarantees. Only validate at system
boundaries (user input, external APIs).

- Abstractions: Don't create helpers, utilities, or abstractions for one-time
operations. Don't design for hypothetical future requirements. The right amount of
complexity is the minimum needed for the current task.
```

### 避免只关注通过测试和硬编码

Claude 有时可能过于专注于让测试通过，而牺牲了更通用的解决方案，或者可能使用辅助脚本等变通方法进行复杂重构，而不是直接使用标准工具。要防止这种行为并获得可泛化的解决方案：

```text Sample prompt wrap
Please write a high-quality, general-purpose solution using the standard tools
available. Do not create helper scripts or workarounds to accomplish the task more
efficiently. Implement a solution that works correctly for all valid inputs, not just
the test cases. Do not hard-code values or create solutions that only work for specific
test inputs. Instead, implement the actual logic that solves the problem generally.

Focus on understanding the problem requirements and implementing the correct algorithm.
Tests are there to verify correctness, not to define the solution. Provide a principled
implementation that follows best practices and software design principles.

If the task is unreasonable or infeasible, or if any of the tests are incorrect, please
inform me rather than working around them. The solution should be robust, maintainable,
and extendable.
```

### 最大限度减少智能体编码中的幻觉

Claude 的最新模型更不容易产生幻觉，并能基于代码给出更准确、更有依据、更智能的答案。要进一步鼓励这种行为并最大限度减少幻觉：

```text Sample prompt wrap
<investigate_before_answering>
Never speculate about code you have not opened. If the user references a specific file,
you MUST read the file before answering. Make sure to investigate and read relevant
files BEFORE answering questions about the codebase. Never make any claims about code
before investigating unless you are certain of the correct answer - give grounded and
hallucination-free answers.
</investigate_before_answering>
```

## 特定能力技巧

### 改进的视觉能力

与先前的 Claude 模型相比，Claude Opus 4.5 和 Claude Opus 4.6 具有改进的视觉能力。它们在图像处理和数据提取任务上表现更好，尤其是当上下文中存在多张图像时。这些改进也延伸到计算机使用场景，模型能够更可靠地解读屏幕截图和 UI 元素。您还可以通过将视频拆分为帧来使用这些模型分析视频。

一种已被证明能进一步提升表现的有效技术是为 Claude 提供裁剪工具或[智能体技能](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview)。测试表明，当 Claude 能够"放大"图像的相关区域时，图像评估结果会持续提升。Anthropic 已创建了一份[裁剪工具示例](https://platform.claude.com/cookbook/multimodal-crop-tool)。

### 前端设计

Claude Opus 4.5 和 Claude Opus 4.6 能够构建具有出色前端设计的复杂、真实世界的 Web 应用程序。然而，在没有指导的情况下，模型可能会默认采用通用模式，从而产生用户所称的"AI slop"（AI 粗制品）美学风格。要创建令人惊喜和愉悦的独特、富有创意的前端：

<Tip>
  有关改进前端设计的详细指南，请参阅关于[通过技能改进前端设计](https://www.claude.com/blog/improving-frontend-design-through-skills)的博客文章。
</Tip>

对于 API 之外的前端设计工作，[Claude Design](https://support.claude.com/en/articles/14604416-get-started-with-claude-design) 提供了画布和设计工具，Claude 可以在其中以交互方式生成和迭代设计。

以下是一个可用于鼓励更好前端设计的系统提示片段：

```text Sample prompt for frontend aesthetics wrap
<frontend_aesthetics>
You tend to converge toward generic, "on distribution" outputs. In frontend design, this
creates what users call the "AI slop" aesthetic. Avoid this: make creative, distinctive
frontends that surprise and delight.

Focus on:
- Typography: Choose fonts that are beautiful, unique, and interesting. Avoid generic
fonts like Arial and Inter; opt instead for distinctive choices that elevate the
frontend's aesthetics.
- Color & Theme: Commit to a cohesive aesthetic. Use CSS variables for consistency.
Dominant colors with sharp accents outperform timid, evenly-distributed palettes. Draw
from IDE themes and cultural aesthetics for inspiration.
- Motion: Use animations for effects and micro-interactions. Prioritize CSS-only
solutions for HTML. Use Motion library for React when available. Focus on high-impact
moments: one well-orchestrated page load with staggered reveals (animation-delay)
creates more delight than scattered micro-interactions.
- Backgrounds: Create atmosphere and depth rather than defaulting to solid colors. Layer
CSS gradients, use geometric patterns, or add contextual effects that match the overall
aesthetic.

Avoid generic AI-generated aesthetics:
- Overused font families (Inter, Roboto, Arial, system fonts)
- Clichéd color schemes (particularly purple gradients on white backgrounds)
- Predictable layouts and component patterns
- Cookie-cutter design that lacks context-specific character

Interpret creatively and make unexpected choices that feel genuinely designed for the
context. Vary between light and dark themes, different fonts, different aesthetics. You
still tend to converge on common choices (Space Grotesk, for example) across
generations. Avoid this: it is critical that you think outside the box!
</frontend_aesthetics>
```

您还可以参考[完整的技能定义](https://github.com/anthropics/claude-code/blob/main/plugins/frontend-design/skills/frontend-design/SKILL.md)。

## 迁移注意事项

从早期版本迁移到当前 Claude 模型时：

1. **明确说明期望的行为：** 考虑准确描述您希望在输出中看到的内容。

2. **使用修饰语来构建您的指令：** 添加鼓励 Claude 提高输出质量和细节的修饰语，有助于更好地塑造 Claude 的表现。例如，不要使用"创建一个分析仪表板"，而应使用"创建一个分析仪表板。包含尽可能多的相关功能和交互。超越基础功能，创建一个功能完备的实现。"

3. **明确请求特定功能：** 如需动画和交互元素，应明确提出请求。

4. **更新思考配置：** Claude 4.6 模型使用[自适应思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)（`thinking: {type: "adaptive"}`），而不是使用 `budget_tokens` 的手动思考。使用 [effort 参数](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)来控制思考深度。

5. **迁移弃用预填充响应：** 从 Claude 4.6 模型和 Claude Mythos Preview 开始，不再支持在最后一个 assistant 轮次中使用预填充响应。有关替代方案的详细指导，请参阅[迁移弃用预填充响应](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#migrating-away-from-prefilled-responses)。

6. **调整反懒惰提示：** 如果您之前的提示鼓励模型更加彻底或更积极地使用工具，请减少此类指导。Claude 4.6 模型更加主动，可能会对先前模型所需的指令过度触发。

7. **原样传回思考块并保持历史记录仅追加：** 按照 API 返回的原样追加每个 assistant 轮次，包括思考块。在 Claude Fable 5.1 上，[修改思考块之前的对话](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking#preserved-in-conversation)会导致错误，或者如果您选择启用该选项，则会导致该块被丢弃：在请求之间编辑较早的消息、重建 `system` 或 `tools`，或就地总结较早的轮次，都会使之后的每个思考块失效，因此请将这些更改移至对话中途的系统消息和服务器端上下文管理。请参阅[保持对话历史记录仅追加](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#keep-the-conversation-history-append-only)。

有关详细的迁移步骤，请参阅[迁移指南](https://platform.claude.com/docs/zh-CN/about-claude/models/migration-guide)。

### 从 Claude Sonnet 4.5 或更早版本迁移到 Claude Sonnet 5

请参阅迁移指南中的[从 Claude Sonnet 4.5 或更早版本迁移到 Claude Sonnet 5](https://platform.claude.com/docs/zh-CN/models/sonnet-5/migration-guide#migrating-from-sonnet-45)，其中涵盖了 effort 默认值的变更以及手动扩展思考（`budget_tokens`）的移除。

## 后续步骤

<CardGroup cols={2}>
  <Card title="Claude Fable 5.1 提示指南" icon="terminal" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5-1">
    Claude Fable 5.1 的行为差异和提示模式，涵盖 effort、任务完成、进度更新、思考块、工具调用批处理和写作风格。
  </Card>

  <Card title="Claude Fable 5 提示指南" icon="terminal" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-fable-5">
    Claude Fable 5 和 Claude Mythos 5 的行为差异和提示模式，涵盖 effort、指令遵循、长时间运行、记忆和脚手架变更。
  </Card>

  <Card title="Claude Sonnet 5 提示指南" icon="terminal" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-sonnet-5">
    Claude Sonnet 5 的行为差异和提示模式，涵盖 effort、自适应思考默认值、工具使用以及从 Claude Sonnet 4.6 迁移。
  </Card>

  <Card title="Claude Opus 5 提示指南" icon="terminal" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/prompting-claude-opus-5">
    Claude Opus 5 的行为差异和提示模式，涵盖响应详细程度、智能体叙述、任务范围界定、子智能体委派和自我纠正。
  </Card>

  <Card title="提示工程概述" icon="edit" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/overview">
    何时使用提示工程，以及在调整提示之前如何规划您的方法。
  </Card>
</CardGroup>
