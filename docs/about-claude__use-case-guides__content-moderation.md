---
title: 内容审核
url: https://platform.claude.com/docs/zh-CN/about-claude/use-case-guides/content-moderation
description: 内容审核是在数字应用中维护安全、尊重和高效环境的关键环节。本指南讨论如何使用 Claude 对您的数字应用中的内容进行审核。
---

> 访问[内容审核 cookbook](https://platform.claude.com/cookbook/misc-building-moderation-filter)，查看使用 Claude 实现内容审核的示例。

<Tip>
  本指南侧重于审核您应用中的用户生成内容。如果您正在寻找有关审核与 Claude 交互的指导，请参阅

  [缓解越狱和提示注入](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks)

  。
</Tip>

## 使用 Claude 构建之前

### 决定是否使用 Claude 进行内容审核

以下是一些关键指标，表明您应该使用像 Claude 这样的 LLM（"large language model"，大型语言模型）而不是传统的机器学习或基于规则的方法来进行内容审核：

<AccordionGroup>
  <Accordion title="您希望实现经济高效且快速的部署">
    传统的机器学习方法需要大量的工程资源、机器学习专业知识和基础设施成本。人工审核系统的成本甚至更高。使用 Claude，您可以在更短的时间内以更低的成本建立并运行一个成熟的审核系统。
  </Accordion>

  <Accordion title="您既需要语义理解又需要快速决策">
    传统的机器学习方法，例如词袋模型或简单的模式匹配，往往难以理解内容的语气、意图和上下文。虽然人工审核系统擅长理解语义含义，但审核内容需要时间。Claude 将语义理解与快速给出审核决策的能力相结合，同时满足了这两种需求。
  </Accordion>

  <Accordion title="您需要一致的策略决策">
    通过利用其先进的推理能力，Claude 可以统一地解释和应用复杂的审核准则。这种一致性有助于确保所有内容得到公平对待，降低因审核决策不一致或带有偏见而损害用户信任的风险。
  </Accordion>

  <Accordion title="您的审核策略可能会随时间变化或演进">
    一旦建立了传统的机器学习方法，对其进行更改是一项费力且需要大量数据的工作。另一方面，随着您的产品或客户需求的演进，Claude 可以轻松适应审核策略的变更或新增，而无需对训练数据进行大量重新标注。
  </Accordion>

  <Accordion title="您需要为审核决策提供可解释的推理">
    如果您希望向用户或监管机构提供审核决策背后的清晰解释，Claude 可以生成详细且连贯的理由说明。这种透明度对于在内容审核实践中建立信任和确保问责至关重要。
  </Accordion>

  <Accordion title="您需要多语言支持而无需维护多个独立模型">
    传统的机器学习方法通常需要为每种支持的语言建立单独的模型或进行大量的翻译处理。人工审核则需要雇用精通每种支持语言的人员。Claude 的多语言能力使其能够对各种语言的工单进行分类，而无需单独的模型或大量的翻译处理，从而简化了面向全球客户群的审核工作。
  </Accordion>

  <Accordion title="您需要多模态支持">
    Claude 的多模态能力使其能够分析和解释文本和图像内容。这使其成为一个多功能工具，适用于需要同时评估不同媒体类型的环境中的全面内容审核。
  </Accordion>
</AccordionGroup>

<Note>
  所有 Claude 模型都经过训练，具有内置的安全行为。这可能导致 Claude 无论使用何种提示，都会对被视为特别危险的内容进行审核（符合

  [可接受使用政策](https://www.anthropic.com/legal/aup)

  ）。例如，一个希望允许用户发布露骨性内容的成人网站可能会发现，即使他们在提示中指定不审核露骨性内容，Claude 仍会将露骨内容标记为需要审核。建议在构建审核解决方案之前先查阅可接受使用政策（AUP）。
</Note>

### 生成待审核内容的示例

在开发内容审核解决方案之前，首先创建应被标记的内容示例和不应被标记的内容示例。确保包含边缘案例和具有挑战性的场景，这些场景可能是内容审核系统难以有效处理的。之后，审查您的示例以创建一个定义明确的审核类别列表。 例如，社交媒体平台生成的示例可能包括以下内容：

<CodeGroup exclude="shell">
  ```python Python
  client = anthropic.Anthropic()

  allowed_user_comments = [
      "This movie was great, I really enjoyed it. The main actor really killed it!",
      "I hate Mondays.",
      "It is a great time to invest in gold!",
  ]

  disallowed_user_comments = [
      "Delete this post now or you better hide. I am coming after you and your family.",
      "Stay away from the 5G cellphones!! They are using 5G to control you.",
      "Congratulations! You have won a $1,000 gift card. Click here to claim your prize!",
  ]

  # 用于测试内容审核的示例用户评论
  user_comments = allowed_user_comments + disallowed_user_comments

  # 内容审核中被视为不安全的类别
  unsafe_categories = [
      "Child Exploitation",
      "Conspiracy Theories",
      "Hate",
      "Indiscriminate Weapons",
      "Intellectual Property",
      "Non-Violent Crimes",
      "Privacy",
      "Self-Harm",
      "Sex Crimes",
      "Sexual Content",
      "Specialized Advice",
      "Violent Crimes",
  ]
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const allowedUserComments = [
    "This movie was great, I really enjoyed it. The main actor really killed it!",
    "I hate Mondays.",
    "It is a great time to invest in gold!"
  ];

  const disallowedUserComments = [
    "Delete this post now or you better hide. I am coming after you and your family.",
    "Stay away from the 5G cellphones!! They are using 5G to control you.",
    "Congratulations! You have won a $1,000 gift card. Click here to claim your prize!"
  ];

  // 用于测试内容审核的示例用户评论
  const userComments = [...allowedUserComments, ...disallowedUserComments];

  // 内容审核中被视为不安全的类别
  const unsafeCategories = [
    "Child Exploitation",
    "Conspiracy Theories",
    "Hate",
    "Indiscriminate Weapons",
    "Intellectual Property",
    "Non-Violent Crimes",
    "Privacy",
    "Self-Harm",
    "Sex Crimes",
    "Sexual Content",
    "Specialized Advice",
    "Violent Crimes"
  ];
  ```

  ```csharp C#
  var client = new AnthropicClient();

  string[] allowedUserComments =
  [
      "This movie was great, I really enjoyed it. The main actor really killed it!",
      "I hate Mondays.",
      "It is a great time to invest in gold!",
  ];

  string[] disallowedUserComments =
  [
      "Delete this post now or you better hide. I am coming after you and your family.",
      "Stay away from the 5G cellphones!! They are using 5G to control you.",
      "Congratulations! You have won a $1,000 gift card. Click here to claim your prize!",
  ];

  // 用于测试内容审核的示例用户评论
  string[] userComments = [.. allowedUserComments, .. disallowedUserComments];

  // 内容审核中被视为不安全的类别
  string[] unsafeCategories =
  [
      "Child Exploitation",
      "Conspiracy Theories",
      "Hate",
      "Indiscriminate Weapons",
      "Intellectual Property",
      "Non-Violent Crimes",
      "Privacy",
      "Self-Harm",
      "Sex Crimes",
      "Sexual Content",
      "Specialized Advice",
      "Violent Crimes",
  ];
  ```

  ```go Go
  var client = anthropic.NewClient()

  var allowedUserComments = []string{
  	"This movie was great, I really enjoyed it. The main actor really killed it!",
  	"I hate Mondays.",
  	"It is a great time to invest in gold!",
  }

  var disallowedUserComments = []string{
  	"Delete this post now or you better hide. I am coming after you and your family.",
  	"Stay away from the 5G cellphones!! They are using 5G to control you.",
  	"Congratulations! You have won a $1,000 gift card. Click here to claim your prize!",
  }

  // 用于测试内容审核的示例用户评论
  var userComments = slices.Concat(allowedUserComments, disallowedUserComments)

  // 内容审核中被视为不安全的类别
  var unsafeCategories = []string{
  	"Child Exploitation",
  	"Conspiracy Theories",
  	"Hate",
  	"Indiscriminate Weapons",
  	"Intellectual Property",
  	"Non-Violent Crimes",
  	"Privacy",
  	"Self-Harm",
  	"Sex Crimes",
  	"Sexual Content",
  	"Specialized Advice",
  	"Violent Crimes",
  }

  ```

  ```java Java
  final AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  final List<String> allowedUserComments = List.of(
          "This movie was great, I really enjoyed it. The main actor really killed it!",
          "I hate Mondays.",
          "It is a great time to invest in gold!");

  final List<String> disallowedUserComments = List.of(
          "Delete this post now or you better hide. I am coming after you and your family.",
          "Stay away from the 5G cellphones!! They are using 5G to control you.",
          "Congratulations! You have won a $1,000 gift card. Click here to claim your prize!");

  // 用于测试内容审核的示例用户评论
  final List<String> userComments =
          Stream.concat(allowedUserComments.stream(), disallowedUserComments.stream()).toList();

  // 内容审核中被视为不安全的类别
  final List<String> unsafeCategories = List.of(
          "Child Exploitation",
          "Conspiracy Theories",
          "Hate",
          "Indiscriminate Weapons",
          "Intellectual Property",
          "Non-Violent Crimes",
          "Privacy",
          "Self-Harm",
          "Sex Crimes",
          "Sexual Content",
          "Specialized Advice",
          "Violent Crimes");
  ```

  ```php PHP
  $client = new Client();

  $allowedUserComments = [
      'This movie was great, I really enjoyed it. The main actor really killed it!',
      'I hate Mondays.',
      'It is a great time to invest in gold!',
  ];

  $disallowedUserComments = [
      'Delete this post now or you better hide. I am coming after you and your family.',
      'Stay away from the 5G cellphones!! They are using 5G to control you.',
      'Congratulations! You have won a $1,000 gift card. Click here to claim your prize!',
  ];

  // 用于测试内容审核的示例用户评论
  $userComments = [...$allowedUserComments, ...$disallowedUserComments];

  // 内容审核中被视为不安全的类别
  $unsafeCategories = [
      'Child Exploitation',
      'Conspiracy Theories',
      'Hate',
      'Indiscriminate Weapons',
      'Intellectual Property',
      'Non-Violent Crimes',
      'Privacy',
      'Self-Harm',
      'Sex Crimes',
      'Sexual Content',
      'Specialized Advice',
      'Violent Crimes',
  ];
  ```

  ```ruby Ruby
  CLIENT = Anthropic::Client.new

  ALLOWED_USER_COMMENTS = [
    "This movie was great, I really enjoyed it. The main actor really killed it!",
    "I hate Mondays.",
    "It is a great time to invest in gold!"
  ]

  DISALLOWED_USER_COMMENTS = [
    "Delete this post now or you better hide. I am coming after you and your family.",
    "Stay away from the 5G cellphones!! They are using 5G to control you.",
    "Congratulations! You have won a $1,000 gift card. Click here to claim your prize!"
  ]

  # 用于测试内容审核的示例用户评论
  USER_COMMENTS = ALLOWED_USER_COMMENTS + DISALLOWED_USER_COMMENTS

  # 内容审核中被视为不安全的类别
  UNSAFE_CATEGORIES = [
    "Child Exploitation",
    "Conspiracy Theories",
    "Hate",
    "Indiscriminate Weapons",
    "Intellectual Property",
    "Non-Violent Crimes",
    "Privacy",
    "Self-Harm",
    "Sex Crimes",
    "Sexual Content",
    "Specialized Advice",
    "Violent Crimes"
  ]
  ```
</CodeGroup>

有效审核这些示例需要对语言有细致入微的理解。在评论 `This movie was great, I really enjoyed it. The main actor really killed it!` 中，内容审核系统需要识别出 "killed it" 是一种比喻，而不是实际暴力的表示。相反，尽管没有明确提及暴力，评论 `Delete this post now or you better hide. I am coming after you and your family.` 应该被内容审核系统标记。

不安全类别可以根据您的具体需求进行自定义。例如，如果您希望防止未成年人在您的网站上创建内容，您可以将 "Underage Posting"（未成年人发帖）添加到类别中。

***

## 如何使用 Claude 审核内容

### 选择合适的 Claude 模型

在选择模型时，考虑数据规模非常重要。如果成本是一个考量因素，像 Claude Haiku 4.5 这样的较小模型因其成本效益而成为极佳选择。以下是对一个每月接收十亿条帖子的社交媒体平台进行文本审核的成本估算：

* **内容规模**

  * 每月帖子数：10 亿
  * 每条帖子字符数：100
  * 总字符数：1000 亿

* **估算令牌数**

  * 输入令牌：286 亿（假设每 3.5 个字符对应 1 个令牌）
  * 被标记消息的百分比：3%
  * 每条被标记消息的输出令牌数：50
  * 总输出令牌数：15 亿

* **Claude Haiku 4.5 估算成本**

  * 输入令牌成本：28,600 MTok \* $1.00/MTok = $28,600 USD
  * 输出令牌成本：1,500 MTok \* $5.00/MTok = $7,500 USD
  * 每月成本：$28,600 + $7,500 = $36,100 USD

* **Claude Opus 5 估算成本**

  * 输入令牌成本：28,600 MTok \* $5.00/MTok = $143,000 USD
  * 输出令牌成本：1,500 MTok \* $25.00/MTok = $37,500 USD
  * 每月成本：$143,000 + $37,500 = $180,500 USD

* **Claude Opus 4.8 估算成本**

  * 输入令牌成本：28,600 MTok \* $5.00/MTok = $143,000 USD
  * 输出令牌成本：1,500 MTok \* $25.00/MTok = $37,500 USD
  * 每月成本：$143,000 + $37,500 = $180,500 USD

<Tip>
  实际成本可能与这些估算有所不同。这些估算基于

  [批量处理](https://platform.claude.com/docs/zh-CN/about-claude/use-case-guides/content-moderation#consider-batch-processing)

  部分中重点介绍的提示。通过从响应中移除 

  `explanation`

   字段，可以进一步减少输出令牌。
</Tip>

### 构建强大的提示

要使用 Claude 进行内容审核，Claude 必须理解您应用的审核要求。首先编写一个允许您定义审核需求的提示：

<CodeGroup exclude="shell">
  ```python Python
  def moderate_message(message, unsafe_categories):
      # 将不安全类别转换为字符串，每个类别占一行
      unsafe_category_str = "\n".join(unsafe_categories)

      # 构建发送给 Claude 的提示，包含消息和不安全类别
      assessment_prompt = f"""
      Determine whether the following message warrants moderation,
      based on the unsafe categories outlined below.

      Message:
      <message>{message}</message>

      Unsafe Categories:
      <categories>
      {unsafe_category_str}
      </categories>

      Respond with ONLY a JSON object, using the format below:
      {{
      "violation": <Boolean field denoting whether the message should be moderated>,
      "categories": [Comma-separated list of violated categories],
      "explanation": [Optional. Only include if there is a violation.]
      }}
  Do not include markdown formatting or code fences in your response."""

      # 向 Claude 发送内容审核请求
      response = client.messages.create(
          model="claude-haiku-4-5-20251001",  # Using the Haiku model for lower costs
          max_tokens=200,
          messages=[{"role": "user", "content": assessment_prompt}],
      )

      # 解析 Claude 返回的 JSON 响应
      text_block = next(block for block in response.content if block.type == "text")
      assessment = json.loads(text_block.text)

      # 从评估结果中提取违规状态
      contains_violation = assessment["violation"]

      # 如果存在违规，获取类别和解释；否则使用空的默认值
      violated_categories = assessment.get("categories", []) if contains_violation else []
      explanation = assessment.get("explanation") if contains_violation else None

      return contains_violation, violated_categories, explanation


  # 处理每条评论并打印结果
  for comment in user_comments:
      print(f"\nComment: {comment}")
      violation, violated_categories, explanation = moderate_message(
          comment, unsafe_categories
      )

      if violation:
          print(f"Violated Categories: {', '.join(violated_categories)}")
          print(f"Explanation: {explanation}")
      else:
          print("No issues detected.")
  ```

  ```typescript TypeScript
  // Claude 返回的 JSON 评估结果的结构
  interface ModerationAssessment {
    violation: boolean;
    categories?: string[];
    explanation?: string;
  }

  async function moderateMessage(
    message: string,
    unsafeCategories: string[]
  ): Promise<{ violation: boolean; violatedCategories: string[]; explanation?: string }> {
    // 将不安全类别转换为字符串，每个类别占一行
    const unsafeCategoryStr = unsafeCategories.join("\n");

    // 构建发送给 Claude 的提示，包含消息和不安全类别
    const assessmentPrompt = `
      Determine whether the following message warrants moderation,
      based on the unsafe categories outlined below.

      Message:
      <message>${message}</message>

      Unsafe Categories:
      <categories>
      ${unsafeCategoryStr}
      </categories>

      Respond with ONLY a JSON object, using the format below:
      {
      "violation": <Boolean field denoting whether the message should be moderated>,
      "categories": [Comma-separated list of violated categories],
      "explanation": [Optional. Only include if there is a violation.]
      }
  Do not include markdown formatting or code fences in your response.`;

    // 向 Claude 发送内容审核请求
    const response = await client.messages.create({
      model: "claude-haiku-4-5-20251001", // Using the Haiku model for lower costs
      max_tokens: 200,
      messages: [{ role: "user", content: assessmentPrompt }]
    });

    // 解析 Claude 返回的 JSON 响应
    const textBlock = response.content.find((block) => block.type === "text");
    if (!textBlock) {
      throw new Error("Expected a text block in the response");
    }
    const assessment: ModerationAssessment = JSON.parse(textBlock.text);

    // 从评估结果中提取违规状态
    const containsViolation = assessment.violation;

    // 如果存在违规，获取类别和说明；否则使用空的默认值
    const violatedCategories = containsViolation ? assessment.categories ?? [] : [];
    const explanation = containsViolation ? assessment.explanation : undefined;

    return { violation: containsViolation, violatedCategories, explanation };
  }

  // 处理每条评论并打印结果
  for (const comment of userComments) {
    console.log(`\nComment: ${comment}`);
    const { violation, violatedCategories, explanation } = await moderateMessage(
      comment,
      unsafeCategories
    );

    if (violation) {
      console.log(`Violated Categories: ${violatedCategories.join(", ")}`);
      console.log(`Explanation: ${explanation}`);
    } else {
      console.log("No issues detected.");
    }
  }
  ```

  ```csharp C#
  async Task<(bool ContainsViolation, List<string> ViolatedCategories, string? Explanation)> ModerateMessage(
      string message,
      IReadOnlyList<string> categories
  )
  {
      // 将不安全类别转换为字符串，每个类别占一行
      var unsafeCategoryText = string.Join("\n", categories);

      // 构建发送给 Claude 的提示，包含消息和不安全类别
      var assessmentPrompt = $$"""

      Determine whether the following message warrants moderation,
      based on the unsafe categories outlined below.

      Message:
      <message>{{message}}</message>

      Unsafe Categories:
      <categories>
      {{unsafeCategoryText}}
      </categories>

      Respond with ONLY a JSON object, using the format below:
      {
      "violation": <Boolean field denoting whether the message should be moderated>,
      "categories": [Comma-separated list of violated categories],
      "explanation": [Optional. Only include if there is a violation.]
      }
  Do not include markdown formatting or code fences in your response.
  """;

      // 向 Claude 发送内容审核请求
      var response = await client.Messages.Create(
          new()
          {
              Model = Model.ClaudeHaiku4_5_20251001, // Using the Haiku model for lower costs
              MaxTokens = 200,
              Messages = [new() { Role = Role.User, Content = assessmentPrompt }],
          }
      );

      // 将第一个内容块收窄为文本块，然后解析 Claude 的 JSON 响应
      if (!response.Content[0].TryPickText(out var textBlock))
      {
          throw new InvalidOperationException("Expected a text response from Claude.");
      }
      var assessment = JsonNode.Parse(textBlock.Text)!;

      // 从评估结果中提取违规状态
      var containsViolation = assessment["violation"]!.GetValue<bool>();

      // 如果存在违规，获取类别和解释；否则使用空的默认值
      List<string> violatedCategories = containsViolation
          ? assessment["categories"]?.AsArray().Select(category => category!.GetValue<string>()).ToList() ?? []
          : [];
      var explanation = containsViolation ? assessment["explanation"]?.GetValue<string>() : null;

      return (containsViolation, violatedCategories, explanation);
  }

  // 处理每条评论并打印结果
  foreach (var comment in userComments)
  {
      Console.WriteLine($"\nComment: {comment}");
      var (violation, violatedCategories, explanation) = await ModerateMessage(comment, unsafeCategories);

      if (violation)
      {
          Console.WriteLine($"Violated Categories: {string.Join(", ", violatedCategories)}");
          Console.WriteLine($"Explanation: {explanation}");
      }
      else
      {
          Console.WriteLine("No issues detected.");
      }
  }
  ```

  ```go Go
  func moderateMessage(message string, unsafeCategories []string) (bool, []string, string) {
  	// 将不安全类别转换为字符串，每个类别占一行
  	unsafeCategoryStr := strings.Join(unsafeCategories, "\n")

  	// 构建发送给 Claude 的提示，包含消息和不安全类别
  	assessmentPrompt := fmt.Sprintf(`
      Determine whether the following message warrants moderation,
      based on the unsafe categories outlined below.

      Message:
      <message>%s</message>

      Unsafe Categories:
      <categories>
      %s
      </categories>

      Respond with ONLY a JSON object, using the format below:
      {
      "violation": <Boolean field denoting whether the message should be moderated>,
      "categories": [Comma-separated list of violated categories],
      "explanation": [Optional. Only include if there is a violation.]
      }
  Do not include markdown formatting or code fences in your response.`, message, unsafeCategoryStr)

  	// 向 Claude 发送内容审核请求
  	response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeHaiku4_5_20251001, // Using the Haiku model for lower costs
  		MaxTokens: 200,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock(assessmentPrompt)),
  		},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	// 在读取文本之前，将第一个内容块收窄为文本块
  	textBlock, ok := response.Content[0].AsAny().(anthropic.TextBlock)
  	if !ok {
  		log.Fatalf("expected a text block, got %q", response.Content[0].Type)
  	}

  	// 解析 Claude 返回的 JSON 响应
  	var assessment struct {
  		Violation   bool     `json:"violation"`
  		Categories  []string `json:"categories"`
  		Explanation string   `json:"explanation"`
  	}
  	if err := json.Unmarshal([]byte(textBlock.Text), &assessment); err != nil {
  		log.Fatal(err)
  	}

  	// 如果存在违规，返回类别和解释；否则使用空的默认值
  	if !assessment.Violation {
  		return false, nil, ""
  	}
  	return true, assessment.Categories, assessment.Explanation
  }

  // moderateAllComments 处理每条评论并打印结果。
  func moderateAllComments() {
  	for _, comment := range userComments {
  		fmt.Printf("\nComment: %s\n", comment)
  		violation, violatedCategories, explanation := moderateMessage(comment, unsafeCategories)

  		if violation {
  			fmt.Printf("Violated Categories: %s\n", strings.Join(violatedCategories, ", "))
  			fmt.Printf("Explanation: %s\n", explanation)
  		} else {
  			fmt.Println("No issues detected.")
  		}
  	}
  }

  ```

  ```java Java
  record ModerationResult(boolean violation, List<String> violatedCategories, String explanation) {}

  ModerationResult moderateMessage(String message, List<String> unsafeCategories)
          throws JsonProcessingException {
      // 将不安全类别转换为字符串，每个类别占一行
      String unsafeCategoryStr = String.join("\n", unsafeCategories);

      // 构建发送给 Claude 的提示，包含消息和不安全类别
      String assessmentPrompt = """

              Determine whether the following message warrants moderation,
              based on the unsafe categories outlined below.

              Message:
              <message>%s</message>

              Unsafe Categories:
              <categories>
              %s
              </categories>

              Respond with ONLY a JSON object, using the format below:
              {
              "violation": <Boolean field denoting whether the message should be moderated>,
              "categories": [Comma-separated list of violated categories],
              "explanation": [Optional. Only include if there is a violation.]
              }
          Do not include markdown formatting or code fences in your response."""
              .formatted(message, unsafeCategoryStr);

      // 向 Claude 发送内容审核请求
      Message response = client.messages().create(MessageCreateParams.builder()
              .model(Model.CLAUDE_HAIKU_4_5_20251001) // Using the Haiku model for lower costs
              .maxTokens(200)
              .addUserMessage(assessmentPrompt)
              .build());

      // 解析 Claude 返回的 JSON 响应
      String assessmentJson = response.content().stream()
              .flatMap(contentBlock -> contentBlock.text().stream())
              .findFirst()
              .orElseThrow()
              .text();
      ObjectMapper mapper = new ObjectMapper();
      JsonNode assessment = mapper.readTree(assessmentJson);

      // 从评估结果中提取违规状态
      boolean containsViolation = assessment.required("violation").asBoolean();

      // 如果存在违规，获取类别和解释；否则使用空的默认值
      List<String> violatedCategories = containsViolation && assessment.has("categories")
              ? mapper.convertValue(assessment.get("categories"), new TypeReference<List<String>>() {})
              : List.of();
      String explanation = containsViolation && assessment.hasNonNull("explanation")
              ? assessment.get("explanation").asText()
              : null;

      return new ModerationResult(containsViolation, violatedCategories, explanation);
  }

  // 处理每条评论并打印结果
  void printModerationResults() throws JsonProcessingException {
      for (String comment : userComments) {
          IO.println("\nComment: " + comment);
          ModerationResult result = moderateMessage(comment, unsafeCategories);

          if (result.violation()) {
              IO.println("Violated Categories: " + String.join(", ", result.violatedCategories()));
              IO.println("Explanation: " + result.explanation());
          } else {
              IO.println("No issues detected.");
          }
      }
  }
  ```

  ```php PHP
  $moderateMessage = function (string $message, array $unsafeCategories) use ($client): array {
      // 将不安全类别转换为字符串，每个类别占一行
      $unsafeCategoryStr = implode("\n", $unsafeCategories);

      // 构建发送给 Claude 的提示，包含消息和不安全类别
      $assessmentPrompt = <<<PROMPT

          Determine whether the following message warrants moderation,
          based on the unsafe categories outlined below.

          Message:
          <message>{$message}</message>

          Unsafe Categories:
          <categories>
          {$unsafeCategoryStr}
          </categories>

          Respond with ONLY a JSON object, using the format below:
          {
          "violation": <Boolean field denoting whether the message should be moderated>,
          "categories": [Comma-separated list of violated categories],
          "explanation": [Optional. Only include if there is a violation.]
          }
      Do not include markdown formatting or code fences in your response.
      PROMPT;

      // 向 Claude 发送内容审核请求
      $response = $client->messages->create(
          model: 'claude-haiku-4-5-20251001', // Using the Haiku model for lower costs
          maxTokens: 200,
          messages: [['role' => 'user', 'content' => $assessmentPrompt]],
      );

      // 解析 Claude 返回的 JSON 响应。SDK 会将每个内容块解码
      // 为其具体类，因此需先找到 TextBlock 再读取文本。
      $textBlock = array_find($response->content, fn ($block) => $block instanceof TextBlock)
          ?? throw new RuntimeException('Expected a text block in the response.');
      $assessment = json_decode($textBlock->text, associative: true, flags: JSON_THROW_ON_ERROR);

      // 从评估结果中提取违规状态
      $containsViolation = $assessment['violation'];

      // 如果存在违规，获取类别和解释；否则使用空的默认值
      $violatedCategories = $containsViolation ? ($assessment['categories'] ?? []) : [];
      $explanation = $containsViolation ? ($assessment['explanation'] ?? null) : null;

      return [$containsViolation, $violatedCategories, $explanation];
  };

  // 处理每条评论并打印结果
  foreach ($userComments as $comment) {
      echo "\nComment: {$comment}\n";
      [$violation, $violatedCategories, $explanation] = $moderateMessage($comment, $unsafeCategories);

      if ($violation) {
          echo 'Violated Categories: ' . implode(', ', $violatedCategories) . "\n";
          echo "Explanation: {$explanation}\n";
      } else {
          echo "No issues detected.\n";
      }
  }
  ```

  ```ruby Ruby
  def moderate_message(message, unsafe_categories)
    # 将不安全类别转换为字符串，每个类别占一行
    unsafe_category_str = unsafe_categories.join("\n")

    # 构建发送给 Claude 的提示，包含消息和不安全类别
    assessment_prompt = <<~PROMPT.chomp

          Determine whether the following message warrants moderation,
          based on the unsafe categories outlined below.

          Message:
          <message>#{message}</message>

          Unsafe Categories:
          <categories>
          #{unsafe_category_str}
          </categories>

          Respond with ONLY a JSON object, using the format below:
          {
          "violation": <Boolean field denoting whether the message should be moderated>,
          "categories": [Comma-separated list of violated categories],
          "explanation": [Optional. Only include if there is a violation.]
          }
      Do not include markdown formatting or code fences in your response.
    PROMPT

    # 向 Claude 发送内容审核请求
    response = CLIENT.messages.create(
      model: "claude-haiku-4-5-20251001", # Using the Haiku model for lower costs
      max_tokens: 200,
      messages: [{role: :user, content: assessment_prompt}]
    )

    # 解析 Claude 返回的 JSON 响应
    text_block = response.content.find { it.type == :text }
    assessment = JSON.parse(text_block.text)

    # 从评估结果中提取违规状态
    contains_violation = assessment["violation"]

    # 如果存在违规，获取类别和解释；否则使用空的默认值
    violated_categories = contains_violation ? assessment.fetch("categories", []) : []
    explanation = contains_violation ? assessment["explanation"] : nil

    [contains_violation, violated_categories, explanation]
  end


  # 处理每条评论并打印结果
  USER_COMMENTS.each do |comment|
    puts "\nComment: #{comment}"
    violation, violated_categories, explanation = moderate_message(comment, UNSAFE_CATEGORIES)

    if violation
      puts "Violated Categories: #{violated_categories.join(", ")}"
      puts "Explanation: #{explanation}"
    else
      puts "No issues detected."
    end
  end
  ```
</CodeGroup>

在此示例中，`moderate_message` 函数包含一个评估提示，其中包括不安全内容类别和待评估的消息。该提示要求 Claude 根据您之前定义的不安全类别，评估该消息是否应被审核。

然后解析模型的评估结果以确定是否存在违规。如果存在违规，Claude 还会返回违规类别列表以及该消息为何不安全的解释。

### 评估您的提示

内容审核是一个分类问题。因此，您可以使用[分类 cookbook](https://platform.claude.com/cookbook/capabilities-classification-guide) 中概述的相同技术来确定内容审核系统的准确性。

另一个需要考虑的因素是，您可以不将内容审核视为二元分类问题，而是创建多个类别来表示不同的风险级别。创建多个风险级别可以让您调整审核的严格程度。例如，您可能希望自动阻止被视为高风险的用户查询，而对有许多中等风险查询的用户进行标记以供人工审查。

<CodeGroup exclude="shell">
  ```python Python
  def assess_risk_level(message, unsafe_categories):
      # 将不安全类别转换为字符串，每个类别占一行
      unsafe_category_str = "\n".join(unsafe_categories)

      # 构建发送给 Claude 的提示，包含消息、不安全类别和风险等级定义
      assessment_prompt = f"""
      Assess the risk level of the following message,
      based on the unsafe categories listed below.

  Message:
  <message>{message}</message>

  Unsafe Categories:
  <categories>
  {unsafe_category_str}
  </categories>

  Assign a risk level based on your confidence that the user's message should be moderated
  based on the defined unsafe categories:

  0 - No risk
  1 - Low risk
  2 - Medium risk
  3 - High risk

  Respond with ONLY a JSON object, using the format below:
  {{
    "risk_level": <Numerical field denoting the risk level>,
    "categories": [Comma-separated list of violated categories],
    "explanation": <Optional. Only include if risk level is greater than 0>
  }}
  Do not include markdown formatting or code fences in your response."""

      # 向 Claude 发送风险评估请求
      response = client.messages.create(
          model="claude-haiku-4-5-20251001",  # Using the Haiku model for lower costs
          max_tokens=200,
          messages=[{"role": "user", "content": assessment_prompt}],
      )

      # 解析 Claude 返回的 JSON 响应
      text_block = next(block for block in response.content if block.type == "text")
      assessment = json.loads(text_block.text)

      # 从评估结果中提取风险等级、违规类别和解释
      risk_level = assessment["risk_level"]
      violated_categories = assessment["categories"]
      explanation = assessment.get("explanation")

      return risk_level, violated_categories, explanation


  # 处理每条评论并打印结果
  for comment in user_comments:
      print(f"\nComment: {comment}")
      risk_level, violated_categories, explanation = assess_risk_level(
          comment, unsafe_categories
      )

      print(f"Risk Level: {risk_level}")
      if violated_categories:
          print(f"Violated Categories: {', '.join(violated_categories)}")
      if explanation:
          print(f"Explanation: {explanation}")
  ```

  ```typescript TypeScript
  // Claude 返回的 JSON 风险评估结果的结构
  interface RiskAssessment {
    risk_level: number;
    categories: string[];
    explanation?: string;
  }

  async function assessRiskLevel(
    message: string,
    unsafeCategories: string[]
  ): Promise<{ riskLevel: number; violatedCategories: string[]; explanation?: string }> {
    // 将不安全类别转换为字符串，每个类别占一行
    const unsafeCategoryStr = unsafeCategories.join("\n");

    // 构建发送给 Claude 的提示，包含消息、不安全类别和风险等级定义
    const assessmentPrompt = `
      Assess the risk level of the following message,
      based on the unsafe categories listed below.

  Message:
  <message>${message}</message>

  Unsafe Categories:
  <categories>
  ${unsafeCategoryStr}
  </categories>

  Assign a risk level based on your confidence that the user's message should be moderated
  based on the defined unsafe categories:

  0 - No risk
  1 - Low risk
  2 - Medium risk
  3 - High risk

  Respond with ONLY a JSON object, using the format below:
  {
    "risk_level": <Numerical field denoting the risk level>,
    "categories": [Comma-separated list of violated categories],
    "explanation": <Optional. Only include if risk level is greater than 0>
  }
  Do not include markdown formatting or code fences in your response.`;

    // 向 Claude 发送风险评估请求
    const response = await client.messages.create({
      model: "claude-haiku-4-5-20251001", // Using the Haiku model for lower costs
      max_tokens: 200,
      messages: [{ role: "user", content: assessmentPrompt }]
    });

    // 解析 Claude 返回的 JSON 响应
    const textBlock = response.content.find((block) => block.type === "text");
    if (!textBlock) {
      throw new Error("Expected a text block in the response");
    }
    const assessment: RiskAssessment = JSON.parse(textBlock.text);

    // 从评估结果中提取风险等级、违规类别和说明
    const { risk_level: riskLevel, categories: violatedCategories, explanation } = assessment;

    return { riskLevel, violatedCategories, explanation };
  }

  // 处理每条评论并打印结果
  for (const comment of userComments) {
    console.log(`\nComment: ${comment}`);
    const { riskLevel, violatedCategories, explanation } = await assessRiskLevel(
      comment,
      unsafeCategories
    );

    console.log(`Risk Level: ${riskLevel}`);
    if (violatedCategories.length > 0) {
      console.log(`Violated Categories: ${violatedCategories.join(", ")}`);
    }
    if (explanation) {
      console.log(`Explanation: ${explanation}`);
    }
  }
  ```

  ```csharp C#
  async Task<(int RiskLevel, List<string> ViolatedCategories, string? Explanation)> AssessRiskLevel(
      string message,
      IReadOnlyList<string> categories
  )
  {
      // 将不安全类别转换为字符串，每个类别占一行
      var unsafeCategoryText = string.Join("\n", categories);

      // 构建发送给 Claude 的提示，包含消息、不安全类别和风险等级定义
      var assessmentPrompt = $$"""

      Assess the risk level of the following message,
      based on the unsafe categories listed below.

  Message:
  <message>{{message}}</message>

  Unsafe Categories:
  <categories>
  {{unsafeCategoryText}}
  </categories>

  Assign a risk level based on your confidence that the user's message should be moderated
  based on the defined unsafe categories:

  0 - No risk
  1 - Low risk
  2 - Medium risk
  3 - High risk

  Respond with ONLY a JSON object, using the format below:
  {
    "risk_level": <Numerical field denoting the risk level>,
    "categories": [Comma-separated list of violated categories],
    "explanation": <Optional. Only include if risk level is greater than 0>
  }
  Do not include markdown formatting or code fences in your response.
  """;

      // 向 Claude 发送风险评估请求
      var response = await client.Messages.Create(
          new()
          {
              Model = Model.ClaudeHaiku4_5_20251001, // Using the Haiku model for lower costs
              MaxTokens = 200,
              Messages = [new() { Role = Role.User, Content = assessmentPrompt }],
          }
      );

      // 将第一个内容块收窄为文本块，然后解析 Claude 的 JSON 响应
      if (!response.Content[0].TryPickText(out var textBlock))
      {
          throw new InvalidOperationException("Expected a text response from Claude.");
      }
      var assessment = JsonNode.Parse(textBlock.Text)!;

      // 从评估结果中提取风险等级、违规类别和解释
      var riskLevel = assessment["risk_level"]!.GetValue<int>();
      var violatedCategories = assessment["categories"]!
          .AsArray()
          .Select(category => category!.GetValue<string>())
          .ToList();
      var explanation = assessment["explanation"]?.GetValue<string>();

      return (riskLevel, violatedCategories, explanation);
  }

  // 处理每条评论并打印结果
  foreach (var comment in userComments)
  {
      Console.WriteLine($"\nComment: {comment}");
      var (riskLevel, violatedCategories, explanation) = await AssessRiskLevel(comment, unsafeCategories);

      Console.WriteLine($"Risk Level: {riskLevel}");
      if (violatedCategories.Count > 0)
      {
          Console.WriteLine($"Violated Categories: {string.Join(", ", violatedCategories)}");
      }
      if (!string.IsNullOrEmpty(explanation))
      {
          Console.WriteLine($"Explanation: {explanation}");
      }
  }
  ```

  ```go Go
  func assessRiskLevel(message string, unsafeCategories []string) (int, []string, string) {
  	// 将不安全类别转换为字符串，每个类别占一行
  	unsafeCategoryStr := strings.Join(unsafeCategories, "\n")

  	// 构建发送给 Claude 的提示，包含消息、不安全类别和风险等级定义
  	assessmentPrompt := fmt.Sprintf(`
      Assess the risk level of the following message,
      based on the unsafe categories listed below.

  Message:
  <message>%s</message>

  Unsafe Categories:
  <categories>
  %s
  </categories>

  Assign a risk level based on your confidence that the user's message should be moderated
  based on the defined unsafe categories:

  0 - No risk
  1 - Low risk
  2 - Medium risk
  3 - High risk

  Respond with ONLY a JSON object, using the format below:
  {
    "risk_level": <Numerical field denoting the risk level>,
    "categories": [Comma-separated list of violated categories],
    "explanation": <Optional. Only include if risk level is greater than 0>
  }
  Do not include markdown formatting or code fences in your response.`, message, unsafeCategoryStr)

  	// 向 Claude 发送风险评估请求
  	response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeHaiku4_5_20251001, // Using the Haiku model for lower costs
  		MaxTokens: 200,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock(assessmentPrompt)),
  		},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	// 在读取文本之前，将第一个内容块收窄为文本块
  	textBlock, ok := response.Content[0].AsAny().(anthropic.TextBlock)
  	if !ok {
  		log.Fatalf("expected a text block, got %q", response.Content[0].Type)
  	}

  	// 解析 Claude 返回的 JSON 响应
  	var assessment struct {
  		RiskLevel   int      `json:"risk_level"`
  		Categories  []string `json:"categories"`
  		Explanation string   `json:"explanation"`
  	}
  	if err := json.Unmarshal([]byte(textBlock.Text), &assessment); err != nil {
  		log.Fatal(err)
  	}

  	// 返回评估得出的风险等级、违规类别和解释
  	return assessment.RiskLevel, assessment.Categories, assessment.Explanation
  }

  // assessAllRiskLevels 处理每条评论并打印结果。
  func assessAllRiskLevels() {
  	for _, comment := range userComments {
  		fmt.Printf("\nComment: %s\n", comment)
  		riskLevel, violatedCategories, explanation := assessRiskLevel(comment, unsafeCategories)

  		fmt.Printf("Risk Level: %d\n", riskLevel)
  		if len(violatedCategories) > 0 {
  			fmt.Printf("Violated Categories: %s\n", strings.Join(violatedCategories, ", "))
  		}
  		if explanation != "" {
  			fmt.Printf("Explanation: %s\n", explanation)
  		}
  	}
  }

  ```

  ```java Java
  record RiskAssessment(int riskLevel, List<String> violatedCategories, String explanation) {}

  RiskAssessment assessRiskLevel(String message, List<String> unsafeCategories)
          throws JsonProcessingException {
      // 将不安全类别转换为字符串，每个类别占一行
      String unsafeCategoryStr = String.join("\n", unsafeCategories);

      // 构建发送给 Claude 的提示，包含消息、不安全类别和风险等级定义
      String assessmentPrompt = """

              Assess the risk level of the following message,
              based on the unsafe categories listed below.

          Message:
          <message>%s</message>

          Unsafe Categories:
          <categories>
          %s
          </categories>

          Assign a risk level based on your confidence that the user's message should be moderated
          based on the defined unsafe categories:

          0 - No risk
          1 - Low risk
          2 - Medium risk
          3 - High risk

          Respond with ONLY a JSON object, using the format below:
          {
            "risk_level": <Numerical field denoting the risk level>,
            "categories": [Comma-separated list of violated categories],
            "explanation": <Optional. Only include if risk level is greater than 0>
          }
          Do not include markdown formatting or code fences in your response."""
              .formatted(message, unsafeCategoryStr);

      // 向 Claude 发送风险评估请求
      Message response = client.messages().create(MessageCreateParams.builder()
              .model(Model.CLAUDE_HAIKU_4_5_20251001) // Using the Haiku model for lower costs
              .maxTokens(200)
              .addUserMessage(assessmentPrompt)
              .build());

      // 解析 Claude 返回的 JSON 响应
      String assessmentJson = response.content().stream()
              .flatMap(contentBlock -> contentBlock.text().stream())
              .findFirst()
              .orElseThrow()
              .text();
      ObjectMapper mapper = new ObjectMapper();
      JsonNode assessment = mapper.readTree(assessmentJson);

      // 从评估结果中提取风险等级、违规类别和解释
      int riskLevel = assessment.required("risk_level").asInt();
      JsonNode categoriesNode = assessment.required("categories");
      List<String> violatedCategories = categoriesNode.isNull()
              ? List.of()
              : mapper.convertValue(categoriesNode, new TypeReference<List<String>>() {});
      String explanation = assessment.hasNonNull("explanation")
              ? assessment.get("explanation").asText()
              : null;

      return new RiskAssessment(riskLevel, violatedCategories, explanation);
  }

  // 处理每条评论并打印结果
  void printRiskLevels() throws JsonProcessingException {
      for (String comment : userComments) {
          IO.println("\nComment: " + comment);
          RiskAssessment assessment = assessRiskLevel(comment, unsafeCategories);

          IO.println("Risk Level: " + assessment.riskLevel());
          if (!assessment.violatedCategories().isEmpty()) {
              IO.println("Violated Categories: " + String.join(", ", assessment.violatedCategories()));
          }
          if (assessment.explanation() != null && !assessment.explanation().isEmpty()) {
              IO.println("Explanation: " + assessment.explanation());
          }
      }
  }
  ```

  ```php PHP
  $assessRiskLevel = function (string $message, array $unsafeCategories) use ($client): array {
      // 将不安全类别转换为字符串，每个类别占一行
      $unsafeCategoryStr = implode("\n", $unsafeCategories);

      // 构建发送给 Claude 的提示，包含消息、不安全类别和风险等级定义
      $assessmentPrompt = <<<PROMPT

          Assess the risk level of the following message,
          based on the unsafe categories listed below.

      Message:
      <message>{$message}</message>

      Unsafe Categories:
      <categories>
      {$unsafeCategoryStr}
      </categories>

      Assign a risk level based on your confidence that the user's message should be moderated
      based on the defined unsafe categories:

      0 - No risk
      1 - Low risk
      2 - Medium risk
      3 - High risk

      Respond with ONLY a JSON object, using the format below:
      {
        "risk_level": <Numerical field denoting the risk level>,
        "categories": [Comma-separated list of violated categories],
        "explanation": <Optional. Only include if risk level is greater than 0>
      }
      Do not include markdown formatting or code fences in your response.
      PROMPT;

      // 向 Claude 发送风险评估请求
      $response = $client->messages->create(
          model: 'claude-haiku-4-5-20251001', // Using the Haiku model for lower costs
          maxTokens: 200,
          messages: [['role' => 'user', 'content' => $assessmentPrompt]],
      );

      // 解析 Claude 返回的 JSON 响应。SDK 会将每个内容块解码
      // 为其具体类，因此需先找到 TextBlock 再读取文本。
      $textBlock = array_find($response->content, fn ($block) => $block instanceof TextBlock)
          ?? throw new RuntimeException('Expected a text block in the response.');
      $assessment = json_decode($textBlock->text, associative: true, flags: JSON_THROW_ON_ERROR);

      // 从评估结果中提取风险等级、违规类别和解释
      $riskLevel = $assessment['risk_level'];
      $violatedCategories = $assessment['categories'];
      $explanation = $assessment['explanation'] ?? null;

      return [$riskLevel, $violatedCategories, $explanation];
  };

  // 处理每条评论并打印结果
  foreach ($userComments as $comment) {
      echo "\nComment: {$comment}\n";
      [$riskLevel, $violatedCategories, $explanation] = $assessRiskLevel($comment, $unsafeCategories);

      echo "Risk Level: {$riskLevel}\n";
      if ($violatedCategories) {
          echo 'Violated Categories: ' . implode(', ', $violatedCategories) . "\n";
      }
      if ($explanation) {
          echo "Explanation: {$explanation}\n";
      }
  }
  ```

  ```ruby Ruby
  def assess_risk_level(message, unsafe_categories)
    # 将不安全类别转换为字符串，每个类别占一行
    unsafe_category_str = unsafe_categories.join("\n")

    # 构建发送给 Claude 的提示，包含消息、不安全类别和风险等级定义
    assessment_prompt = <<~PROMPT.chomp

          Assess the risk level of the following message,
          based on the unsafe categories listed below.

      Message:
      <message>#{message}</message>

      Unsafe Categories:
      <categories>
      #{unsafe_category_str}
      </categories>

      Assign a risk level based on your confidence that the user's message should be moderated
      based on the defined unsafe categories:

      0 - No risk
      1 - Low risk
      2 - Medium risk
      3 - High risk

      Respond with ONLY a JSON object, using the format below:
      {
        "risk_level": <Numerical field denoting the risk level>,
        "categories": [Comma-separated list of violated categories],
        "explanation": <Optional. Only include if risk level is greater than 0>
      }
      Do not include markdown formatting or code fences in your response.
    PROMPT

    # 向 Claude 发送风险评估请求
    response = CLIENT.messages.create(
      model: "claude-haiku-4-5-20251001", # Using the Haiku model for lower costs
      max_tokens: 200,
      messages: [{role: :user, content: assessment_prompt}]
    )

    # 解析 Claude 返回的 JSON 响应
    text_block = response.content.find { it.type == :text }
    assessment = JSON.parse(text_block.text)

    # 从评估结果中提取风险等级、违规类别和解释
    risk_level = assessment["risk_level"]
    violated_categories = assessment["categories"]
    explanation = assessment["explanation"]

    [risk_level, violated_categories, explanation]
  end


  # 处理每条评论并打印结果
  USER_COMMENTS.each do |comment|
    puts "\nComment: #{comment}"
    risk_level, violated_categories, explanation = assess_risk_level(comment, UNSAFE_CATEGORIES)

    puts "Risk Level: #{risk_level}"
    puts "Violated Categories: #{violated_categories.join(", ")}" if violated_categories&.any?
    puts "Explanation: #{explanation}" if explanation
  end
  ```
</CodeGroup>

此代码实现了一个 `assess_risk_level` 函数，该函数使用 Claude 评估消息的风险级别。该函数接受消息和不安全类别作为输入。

在函数内部，会为 Claude 生成一个提示，其中包括待评估的消息、不安全类别以及评估风险级别的具体说明。该提示指示 Claude 以 JSON 对象进行响应，其中包括风险级别、违规类别以及可选的解释。

这种方法通过分配风险级别实现了灵活的内容审核。它可以无缝集成到更大的系统中，根据评估的风险级别自动过滤内容或标记评论以供人工审查。例如，运行此代码时，评论 `Delete this post now or you better hide. I am coming after you and your family.` 因其危险的威胁而被识别为高风险。相反，评论 `Stay away from the 5G cellphones!! They are using 5G to control you.` 被归类为中等风险。

### 部署您的提示

一旦您对解决方案的质量有信心，就可以将其部署到生产环境了。以下是在生产环境中使用内容审核时应遵循的一些最佳实践：

1. **向用户提供清晰的反馈：** 当用户输入因内容审核而被阻止或响应被标记时，提供有用且具有建设性的反馈，帮助用户理解其消息为何被标记以及如何适当地重新表述。在前面的代码示例中，这是通过 Claude 响应中的 `explanation` 字段实现的。

2. **分析被审核的内容：** 跟踪审核系统标记的内容类型，以识别趋势和潜在的改进领域。

3. **持续评估和改进：** 使用精确率和召回率跟踪等指标定期评估内容审核系统的性能。利用这些数据迭代优化您的审核提示、关键词和评估标准。

***

## 提升性能

在复杂场景中，除了标准的[提示工程技术](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/overview)之外，考虑其他策略来提升性能可能会有所帮助。以下是一些高级策略：

### 定义主题并提供示例

除了在提示中列出不安全类别之外，还可以通过提供与每个类别相关的定义和短语来进一步改进。

<CodeGroup exclude="shell">
  ```python Python
  # 内容审核中被视为不安全的类别及其定义
  unsafe_category_definitions = {
      "Child Exploitation": "Content that depicts child nudity or that enables, encourages, excuses, or depicts the sexual abuse of children.",
      "Conspiracy Theories": """Content that promotes or endorses unfounded, false, or misleading theories about events, situations, or individuals,
          which can incite harm or undermine public trust in institutions or facts.""",
      "Hate": """Content that is hateful toward people on the basis of their protected characteristics
          (race, color, ethnicity, national origin, disability, religious affiliation, caste, sexual orientation, sex, gender identity and serious disease),
          or content that perpetuates negative stereotypes.""",
      "Indiscriminate Weapons": "Content that enables the creation of weapons of mass destruction.",
      "Intellectual Property": "Content that violates the intellectual property rights of third parties.",
      "Non-Violent Crimes": "Content that enables, encourages, or excuses the commission of non-violent crimes.",
      "Privacy": "Content that contains sensitive, personal information about private individuals.",
      "Self-Harm": "Content that enables, encourages, or excuses acts of intentional self-harm.",
      "Sex Crimes": "Content that enables, encourages, or excuses the commission of sex crimes.",
      "Sexual Content": "Content that contains erotic descriptions or explicit depictions of sex acts.",
      "Specialized Advice": """Content that contains financial, medical, or legal advice.
          Financial advice includes guidance on investments, stocks, bonds, or any financial planning.""",
      "Violent Crimes": "Content that enables, encourages, or excuses the commission of violent crimes.",
  }


  def moderate_message_with_definitions(message, unsafe_category_definitions):
      # 格式化不安全类别字符串，每个类别及其定义占一行
      unsafe_category_str = "\n".join(
          f"{category}: {definition}"
          for category, definition in unsafe_category_definitions.items()
      )

      # 构建发送给 Claude 的提示，包含消息和不安全类别
      assessment_prompt = f"""Determine whether the following message warrants moderation, based on the unsafe categories outlined below.

  Message:
  <message>{message}</message>

  Unsafe Categories and Their Definitions:
  <categories>
  {unsafe_category_str}
  </categories>

  It's important that you remember all unsafe categories and their definitions.

  Respond with ONLY a JSON object, using the format below:
  {{
    "violation": <Boolean field denoting whether the message should be moderated>,
    "categories": [Comma-separated list of violated categories],
    "explanation": [Optional. Only include if there is a violation.]
  }}
  Do not include markdown formatting or code fences in your response."""

      # 向 Claude 发送内容审核请求
      response = client.messages.create(
          model="claude-haiku-4-5-20251001",  # Using the Haiku model for lower costs
          max_tokens=200,
          messages=[{"role": "user", "content": assessment_prompt}],
      )

      # 解析 Claude 返回的 JSON 响应
      text_block = next(block for block in response.content if block.type == "text")
      assessment = json.loads(text_block.text)

      # 从评估结果中提取违规状态
      contains_violation = assessment["violation"]

      # 如果存在违规，获取类别和解释；否则使用空的默认值
      violated_categories = assessment.get("categories", []) if contains_violation else []
      explanation = assessment.get("explanation") if contains_violation else None

      return contains_violation, violated_categories, explanation


  # 处理每条评论并打印结果
  for comment in user_comments:
      print(f"\nComment: {comment}")
      violation, violated_categories, explanation = moderate_message_with_definitions(
          comment, unsafe_category_definitions
      )

      if violation:
          print(f"Violated Categories: {', '.join(violated_categories)}")
          print(f"Explanation: {explanation}")
      else:
          print("No issues detected.")
  ```

  ```typescript TypeScript
  // Claude 返回的 JSON 评估结果的结构
  interface DefinitionBasedAssessment {
    violation: boolean;
    categories?: string[];
    explanation?: string;
  }

  // 内容审核中被视为不安全的类别及其定义
  // （对象键保留插入顺序，因此类别按此顺序呈现）
  const unsafeCategoryDefinitions: Record<string, string> = {
    "Child Exploitation":
      "Content that depicts child nudity or that enables, encourages, excuses, or depicts the sexual abuse of children.",
    "Conspiracy Theories": `Content that promotes or endorses unfounded, false, or misleading theories about events, situations, or individuals,
          which can incite harm or undermine public trust in institutions or facts.`,
    "Hate": `Content that is hateful toward people on the basis of their protected characteristics
          (race, color, ethnicity, national origin, disability, religious affiliation, caste, sexual orientation, sex, gender identity and serious disease),
          or content that perpetuates negative stereotypes.`,
    "Indiscriminate Weapons":
      "Content that enables the creation of weapons of mass destruction.",
    "Intellectual Property":
      "Content that violates the intellectual property rights of third parties.",
    "Non-Violent Crimes":
      "Content that enables, encourages, or excuses the commission of non-violent crimes.",
    "Privacy":
      "Content that contains sensitive, personal information about private individuals.",
    "Self-Harm": "Content that enables, encourages, or excuses acts of intentional self-harm.",
    "Sex Crimes": "Content that enables, encourages, or excuses the commission of sex crimes.",
    "Sexual Content":
      "Content that contains erotic descriptions or explicit depictions of sex acts.",
    "Specialized Advice": `Content that contains financial, medical, or legal advice.
          Financial advice includes guidance on investments, stocks, bonds, or any financial planning.`,
    "Violent Crimes":
      "Content that enables, encourages, or excuses the commission of violent crimes."
  };

  async function moderateMessageWithDefinitions(
    message: string,
    unsafeCategoryDefinitions: Record<string, string>
  ): Promise<{ violation: boolean; violatedCategories: string[]; explanation?: string }> {
    // 格式化不安全类别字符串，每个类别及其定义占一行
    const unsafeCategoryStr = Object.entries(unsafeCategoryDefinitions)
      .map(([category, definition]) => `${category}: ${definition}`)
      .join("\n");

    // 构建发送给 Claude 的提示，包含消息和不安全类别
    const assessmentPrompt = `Determine whether the following message warrants moderation, based on the unsafe categories outlined below.

  Message:
  <message>${message}</message>

  Unsafe Categories and Their Definitions:
  <categories>
  ${unsafeCategoryStr}
  </categories>

  It's important that you remember all unsafe categories and their definitions.

  Respond with ONLY a JSON object, using the format below:
  {
    "violation": <Boolean field denoting whether the message should be moderated>,
    "categories": [Comma-separated list of violated categories],
    "explanation": [Optional. Only include if there is a violation.]
  }
  Do not include markdown formatting or code fences in your response.`;

    // 向 Claude 发送内容审核请求
    const response = await client.messages.create({
      model: "claude-haiku-4-5-20251001", // Using the Haiku model for lower costs
      max_tokens: 200,
      messages: [{ role: "user", content: assessmentPrompt }]
    });

    // 解析 Claude 返回的 JSON 响应
    const textBlock = response.content.find((block) => block.type === "text");
    if (!textBlock) {
      throw new Error("Expected a text block in the response");
    }
    const assessment: DefinitionBasedAssessment = JSON.parse(textBlock.text);

    // 从评估结果中提取违规状态
    const containsViolation = assessment.violation;

    // 如果存在违规，获取类别和说明；否则使用空的默认值
    const violatedCategories = containsViolation ? assessment.categories ?? [] : [];
    const explanation = containsViolation ? assessment.explanation : undefined;

    return { violation: containsViolation, violatedCategories, explanation };
  }

  // 处理每条评论并打印结果
  for (const comment of userComments) {
    console.log(`\nComment: ${comment}`);
    const { violation, violatedCategories, explanation } = await moderateMessageWithDefinitions(
      comment,
      unsafeCategoryDefinitions
    );

    if (violation) {
      console.log(`Violated Categories: ${violatedCategories.join(", ")}`);
      console.log(`Explanation: ${explanation}`);
    } else {
      console.log("No issues detected.");
    }
  }
  ```

  ```csharp C#
  // 内容审核中被视为不安全的类别及其定义。
  // 条目保持插入顺序，因此渲染后的提示会严格按照
  // 此顺序列出类别。
  (string Category, string Definition)[] unsafeCategoryDefinitions =
  [
      (
          "Child Exploitation",
          "Content that depicts child nudity or that enables, encourages, excuses, or depicts the sexual abuse of children."
      ),
      (
          "Conspiracy Theories",
          """
          Content that promotes or endorses unfounded, false, or misleading theories about events, situations, or individuals,
                  which can incite harm or undermine public trust in institutions or facts.
          """
      ),
      (
          "Hate",
          """
          Content that is hateful toward people on the basis of their protected characteristics
                  (race, color, ethnicity, national origin, disability, religious affiliation, caste, sexual orientation, sex, gender identity and serious disease),
                  or content that perpetuates negative stereotypes.
          """
      ),
      ("Indiscriminate Weapons", "Content that enables the creation of weapons of mass destruction."),
      ("Intellectual Property", "Content that violates the intellectual property rights of third parties."),
      ("Non-Violent Crimes", "Content that enables, encourages, or excuses the commission of non-violent crimes."),
      ("Privacy", "Content that contains sensitive, personal information about private individuals."),
      ("Self-Harm", "Content that enables, encourages, or excuses acts of intentional self-harm."),
      ("Sex Crimes", "Content that enables, encourages, or excuses the commission of sex crimes."),
      ("Sexual Content", "Content that contains erotic descriptions or explicit depictions of sex acts."),
      (
          "Specialized Advice",
          """
          Content that contains financial, medical, or legal advice.
                  Financial advice includes guidance on investments, stocks, bonds, or any financial planning.
          """
      ),
      ("Violent Crimes", "Content that enables, encourages, or excuses the commission of violent crimes."),
  ];


  async Task<(bool ContainsViolation, List<string> ViolatedCategories, string? Explanation)> ModerateMessageWithDefinitions(
      string message,
      IReadOnlyList<(string Category, string Definition)> categoryDefinitions
  )
  {
      // 格式化不安全类别字符串，每个类别及其定义占一行
      var unsafeCategoryText = string.Join(
          "\n",
          categoryDefinitions.Select(entry => $"{entry.Category}: {entry.Definition}")
      );

      // 构建发送给 Claude 的提示，包含消息和不安全类别
      var assessmentPrompt = $$"""
  Determine whether the following message warrants moderation, based on the unsafe categories outlined below.

  Message:
  <message>{{message}}</message>

  Unsafe Categories and Their Definitions:
  <categories>
  {{unsafeCategoryText}}
  </categories>

  It's important that you remember all unsafe categories and their definitions.

  Respond with ONLY a JSON object, using the format below:
  {
    "violation": <Boolean field denoting whether the message should be moderated>,
    "categories": [Comma-separated list of violated categories],
    "explanation": [Optional. Only include if there is a violation.]
  }
  Do not include markdown formatting or code fences in your response.
  """;

      // 向 Claude 发送内容审核请求
      var response = await client.Messages.Create(
          new()
          {
              Model = Model.ClaudeHaiku4_5_20251001, // Using the Haiku model for lower costs
              MaxTokens = 200,
              Messages = [new() { Role = Role.User, Content = assessmentPrompt }],
          }
      );

      // 将第一个内容块收窄为文本块，然后解析 Claude 的 JSON 响应
      if (!response.Content[0].TryPickText(out var textBlock))
      {
          throw new InvalidOperationException("Expected a text response from Claude.");
      }
      var assessment = JsonNode.Parse(textBlock.Text)!;

      // 从评估结果中提取违规状态
      var containsViolation = assessment["violation"]!.GetValue<bool>();

      // 如果存在违规，获取类别和解释；否则使用空的默认值
      List<string> violatedCategories = containsViolation
          ? assessment["categories"]?.AsArray().Select(category => category!.GetValue<string>()).ToList() ?? []
          : [];
      var explanation = containsViolation ? assessment["explanation"]?.GetValue<string>() : null;

      return (containsViolation, violatedCategories, explanation);
  }

  // 处理每条评论并打印结果
  foreach (var comment in userComments)
  {
      Console.WriteLine($"\nComment: {comment}");
      var (violation, violatedCategories, explanation) = await ModerateMessageWithDefinitions(
          comment,
          unsafeCategoryDefinitions
      );

      if (violation)
      {
          Console.WriteLine($"Violated Categories: {string.Join(", ", violatedCategories)}");
          Console.WriteLine($"Explanation: {explanation}");
      }
      else
      {
          Console.WriteLine("No issues detected.");
      }
  }
  ```

  ```go Go
  // 内容审核中被视为不安全的类别及其定义。
  // 使用类别/定义对的切片（而非 map）可保持渲染
  // 顺序稳定；Go 的 map 迭代顺序是随机的。
  type categoryDefinition struct {
  	category   string
  	definition string
  }

  var unsafeCategoryDefinitions = []categoryDefinition{
  	{"Child Exploitation", "Content that depicts child nudity or that enables, encourages, excuses, or depicts the sexual abuse of children."},
  	{"Conspiracy Theories", `Content that promotes or endorses unfounded, false, or misleading theories about events, situations, or individuals,
          which can incite harm or undermine public trust in institutions or facts.`},
  	{"Hate", `Content that is hateful toward people on the basis of their protected characteristics
          (race, color, ethnicity, national origin, disability, religious affiliation, caste, sexual orientation, sex, gender identity and serious disease),
          or content that perpetuates negative stereotypes.`},
  	{"Indiscriminate Weapons", "Content that enables the creation of weapons of mass destruction."},
  	{"Intellectual Property", "Content that violates the intellectual property rights of third parties."},
  	{"Non-Violent Crimes", "Content that enables, encourages, or excuses the commission of non-violent crimes."},
  	{"Privacy", "Content that contains sensitive, personal information about private individuals."},
  	{"Self-Harm", "Content that enables, encourages, or excuses acts of intentional self-harm."},
  	{"Sex Crimes", "Content that enables, encourages, or excuses the commission of sex crimes."},
  	{"Sexual Content", "Content that contains erotic descriptions or explicit depictions of sex acts."},
  	{"Specialized Advice", `Content that contains financial, medical, or legal advice.
          Financial advice includes guidance on investments, stocks, bonds, or any financial planning.`},
  	{"Violent Crimes", "Content that enables, encourages, or excuses the commission of violent crimes."},
  }

  func moderateMessageWithDefinitions(message string, unsafeCategoryDefinitions []categoryDefinition) (bool, []string, string) {
  	// 格式化不安全类别字符串，每个类别及其定义占一行
  	categoryLines := make([]string, len(unsafeCategoryDefinitions))
  	for i, entry := range unsafeCategoryDefinitions {
  		categoryLines[i] = fmt.Sprintf("%s: %s", entry.category, entry.definition)
  	}
  	unsafeCategoryStr := strings.Join(categoryLines, "\n")

  	// 构建发送给 Claude 的提示，包含消息和不安全类别
  	assessmentPrompt := fmt.Sprintf(`Determine whether the following message warrants moderation, based on the unsafe categories outlined below.

  Message:
  <message>%s</message>

  Unsafe Categories and Their Definitions:
  <categories>
  %s
  </categories>

  It's important that you remember all unsafe categories and their definitions.

  Respond with ONLY a JSON object, using the format below:
  {
    "violation": <Boolean field denoting whether the message should be moderated>,
    "categories": [Comma-separated list of violated categories],
    "explanation": [Optional. Only include if there is a violation.]
  }
  Do not include markdown formatting or code fences in your response.`, message, unsafeCategoryStr)

  	// 向 Claude 发送内容审核请求
  	response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeHaiku4_5_20251001, // Using the Haiku model for lower costs
  		MaxTokens: 200,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock(assessmentPrompt)),
  		},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	// 在读取文本之前，将第一个内容块收窄为文本块
  	textBlock, ok := response.Content[0].AsAny().(anthropic.TextBlock)
  	if !ok {
  		log.Fatalf("expected a text block, got %q", response.Content[0].Type)
  	}

  	// 解析 Claude 返回的 JSON 响应
  	var assessment struct {
  		Violation   bool     `json:"violation"`
  		Categories  []string `json:"categories"`
  		Explanation string   `json:"explanation"`
  	}
  	if err := json.Unmarshal([]byte(textBlock.Text), &assessment); err != nil {
  		log.Fatal(err)
  	}

  	// 如果存在违规，返回类别和解释；否则使用空的默认值
  	if !assessment.Violation {
  		return false, nil, ""
  	}
  	return true, assessment.Categories, assessment.Explanation
  }

  // moderateAllCommentsWithDefinitions 处理每条评论并打印结果。
  func moderateAllCommentsWithDefinitions() {
  	for _, comment := range userComments {
  		fmt.Printf("\nComment: %s\n", comment)
  		violation, violatedCategories, explanation := moderateMessageWithDefinitions(comment, unsafeCategoryDefinitions)

  		if violation {
  			fmt.Printf("Violated Categories: %s\n", strings.Join(violatedCategories, ", "))
  			fmt.Printf("Explanation: %s\n", explanation)
  		} else {
  			fmt.Println("No issues detected.")
  		}
  	}
  }

  ```

  ```java Java
  // 内容审核中被视为不安全的类别及其定义
  record CategoryDefinition(String category, String definition) {}

  final List<CategoryDefinition> unsafeCategoryDefinitions = List.of(
          new CategoryDefinition(
                  "Child Exploitation",
                  "Content that depicts child nudity or that enables, encourages, excuses, or depicts the sexual abuse of children."),
          new CategoryDefinition(
                  "Conspiracy Theories",
                  """
                  Content that promotes or endorses unfounded, false, or misleading theories about events, situations, or individuals,
                          which can incite harm or undermine public trust in institutions or facts."""),
          new CategoryDefinition(
                  "Hate",
                  """
                  Content that is hateful toward people on the basis of their protected characteristics
                          (race, color, ethnicity, national origin, disability, religious affiliation, caste, sexual orientation, sex, gender identity and serious disease),
                          or content that perpetuates negative stereotypes."""),
          new CategoryDefinition(
                  "Indiscriminate Weapons",
                  "Content that enables the creation of weapons of mass destruction."),
          new CategoryDefinition(
                  "Intellectual Property",
                  "Content that violates the intellectual property rights of third parties."),
          new CategoryDefinition(
                  "Non-Violent Crimes",
                  "Content that enables, encourages, or excuses the commission of non-violent crimes."),
          new CategoryDefinition(
                  "Privacy",
                  "Content that contains sensitive, personal information about private individuals."),
          new CategoryDefinition(
                  "Self-Harm",
                  "Content that enables, encourages, or excuses acts of intentional self-harm."),
          new CategoryDefinition(
                  "Sex Crimes",
                  "Content that enables, encourages, or excuses the commission of sex crimes."),
          new CategoryDefinition(
                  "Sexual Content",
                  "Content that contains erotic descriptions or explicit depictions of sex acts."),
          new CategoryDefinition(
                  "Specialized Advice",
                  """
                  Content that contains financial, medical, or legal advice.
                          Financial advice includes guidance on investments, stocks, bonds, or any financial planning."""),
          new CategoryDefinition(
                  "Violent Crimes",
                  "Content that enables, encourages, or excuses the commission of violent crimes."));

  record ModerationDecision(boolean violation, List<String> violatedCategories, String explanation) {}

  ModerationDecision moderateMessageWithDefinitions(
          String message, List<CategoryDefinition> unsafeCategoryDefinitions)
          throws JsonProcessingException {
      // 格式化不安全类别字符串，每个类别及其定义占一行
      String unsafeCategoryStr = unsafeCategoryDefinitions.stream()
              .map(categoryDefinition ->
                      categoryDefinition.category() + ": " + categoryDefinition.definition())
              .collect(Collectors.joining("\n"));

      // 构建发送给 Claude 的提示，包含消息和不安全类别
      String assessmentPrompt = """
          Determine whether the following message warrants moderation, based on the unsafe categories outlined below.

          Message:
          <message>%s</message>

          Unsafe Categories and Their Definitions:
          <categories>
          %s
          </categories>

          It's important that you remember all unsafe categories and their definitions.

          Respond with ONLY a JSON object, using the format below:
          {
            "violation": <Boolean field denoting whether the message should be moderated>,
            "categories": [Comma-separated list of violated categories],
            "explanation": [Optional. Only include if there is a violation.]
          }
          Do not include markdown formatting or code fences in your response."""
              .formatted(message, unsafeCategoryStr);

      // 向 Claude 发送内容审核请求
      Message response = client.messages().create(MessageCreateParams.builder()
              .model(Model.CLAUDE_HAIKU_4_5_20251001) // Using the Haiku model for lower costs
              .maxTokens(200)
              .addUserMessage(assessmentPrompt)
              .build());

      // 解析 Claude 返回的 JSON 响应
      String assessmentJson = response.content().stream()
              .flatMap(contentBlock -> contentBlock.text().stream())
              .findFirst()
              .orElseThrow()
              .text();
      ObjectMapper mapper = new ObjectMapper();
      JsonNode assessment = mapper.readTree(assessmentJson);

      // 从评估结果中提取违规状态
      boolean containsViolation = assessment.required("violation").asBoolean();

      // 如果存在违规，获取类别和解释；否则使用空的默认值
      List<String> violatedCategories = containsViolation && assessment.has("categories")
              ? mapper.convertValue(assessment.get("categories"), new TypeReference<List<String>>() {})
              : List.of();
      String explanation = containsViolation && assessment.hasNonNull("explanation")
              ? assessment.get("explanation").asText()
              : null;

      return new ModerationDecision(containsViolation, violatedCategories, explanation);
  }

  // 处理每条评论并打印结果
  void printModerationResultsWithDefinitions() throws JsonProcessingException {
      for (String comment : userComments) {
          IO.println("\nComment: " + comment);
          ModerationDecision result = moderateMessageWithDefinitions(comment, unsafeCategoryDefinitions);

          if (result.violation()) {
              IO.println("Violated Categories: " + String.join(", ", result.violatedCategories()));
              IO.println("Explanation: " + result.explanation());
          } else {
              IO.println("No issues detected.");
          }
      }
  }
  ```

  ```php PHP
  // 内容审核中被视为不安全的类别及其定义
  $unsafeCategoryDefinitions = [
      'Child Exploitation' => 'Content that depicts child nudity or that enables, encourages, excuses, or depicts the sexual abuse of children.',
      'Conspiracy Theories' => 'Content that promotes or endorses unfounded, false, or misleading theories about events, situations, or individuals,
          which can incite harm or undermine public trust in institutions or facts.',
      'Hate' => 'Content that is hateful toward people on the basis of their protected characteristics
          (race, color, ethnicity, national origin, disability, religious affiliation, caste, sexual orientation, sex, gender identity and serious disease),
          or content that perpetuates negative stereotypes.',
      'Indiscriminate Weapons' => 'Content that enables the creation of weapons of mass destruction.',
      'Intellectual Property' => 'Content that violates the intellectual property rights of third parties.',
      'Non-Violent Crimes' => 'Content that enables, encourages, or excuses the commission of non-violent crimes.',
      'Privacy' => 'Content that contains sensitive, personal information about private individuals.',
      'Self-Harm' => 'Content that enables, encourages, or excuses acts of intentional self-harm.',
      'Sex Crimes' => 'Content that enables, encourages, or excuses the commission of sex crimes.',
      'Sexual Content' => 'Content that contains erotic descriptions or explicit depictions of sex acts.',
      'Specialized Advice' => 'Content that contains financial, medical, or legal advice.
          Financial advice includes guidance on investments, stocks, bonds, or any financial planning.',
      'Violent Crimes' => 'Content that enables, encourages, or excuses the commission of violent crimes.',
  ];

  $moderateMessageWithDefinitions = function (string $message, array $unsafeCategoryDefinitions) use ($client): array {
      // 格式化不安全类别字符串，每个类别及其定义占一行
      $categoryLines = [];
      foreach ($unsafeCategoryDefinitions as $category => $definition) {
          $categoryLines[] = "{$category}: {$definition}";
      }
      $unsafeCategoryStr = implode("\n", $categoryLines);

      // 构建发送给 Claude 的提示，包含消息和不安全类别
      $assessmentPrompt = <<<PROMPT
      Determine whether the following message warrants moderation, based on the unsafe categories outlined below.

      Message:
      <message>{$message}</message>

      Unsafe Categories and Their Definitions:
      <categories>
      {$unsafeCategoryStr}
      </categories>

      It's important that you remember all unsafe categories and their definitions.

      Respond with ONLY a JSON object, using the format below:
      {
        "violation": <Boolean field denoting whether the message should be moderated>,
        "categories": [Comma-separated list of violated categories],
        "explanation": [Optional. Only include if there is a violation.]
      }
      Do not include markdown formatting or code fences in your response.
      PROMPT;

      // 向 Claude 发送内容审核请求
      $response = $client->messages->create(
          model: 'claude-haiku-4-5-20251001', // Using the Haiku model for lower costs
          maxTokens: 200,
          messages: [['role' => 'user', 'content' => $assessmentPrompt]],
      );

      // 解析 Claude 返回的 JSON 响应。SDK 会将每个内容块解码
      // 为其具体类，因此需先找到 TextBlock 再读取文本。
      $textBlock = array_find($response->content, fn ($block) => $block instanceof TextBlock)
          ?? throw new RuntimeException('Expected a text block in the response.');
      $assessment = json_decode($textBlock->text, associative: true, flags: JSON_THROW_ON_ERROR);

      // 从评估结果中提取违规状态
      $containsViolation = $assessment['violation'];

      // 如果存在违规，获取类别和解释；否则使用空的默认值
      $violatedCategories = $containsViolation ? ($assessment['categories'] ?? []) : [];
      $explanation = $containsViolation ? ($assessment['explanation'] ?? null) : null;

      return [$containsViolation, $violatedCategories, $explanation];
  };

  // 处理每条评论并打印结果
  foreach ($userComments as $comment) {
      echo "\nComment: {$comment}\n";
      [$violation, $violatedCategories, $explanation] = $moderateMessageWithDefinitions($comment, $unsafeCategoryDefinitions);

      if ($violation) {
          echo 'Violated Categories: ' . implode(', ', $violatedCategories) . "\n";
          echo "Explanation: {$explanation}\n";
      } else {
          echo "No issues detected.\n";
      }
  }
  ```

  ```ruby Ruby
  # 内容审核中被视为不安全的类别及其定义
  UNSAFE_CATEGORY_DEFINITIONS = {
    "Child Exploitation" => "Content that depicts child nudity or that enables, encourages, excuses, or depicts the sexual abuse of children.",
    "Conspiracy Theories" => "Content that promotes or endorses unfounded, false, or misleading theories about events, situations, or individuals,
          which can incite harm or undermine public trust in institutions or facts.",
    "Hate" => "Content that is hateful toward people on the basis of their protected characteristics
          (race, color, ethnicity, national origin, disability, religious affiliation, caste, sexual orientation, sex, gender identity and serious disease),
          or content that perpetuates negative stereotypes.",
    "Indiscriminate Weapons" => "Content that enables the creation of weapons of mass destruction.",
    "Intellectual Property" => "Content that violates the intellectual property rights of third parties.",
    "Non-Violent Crimes" => "Content that enables, encourages, or excuses the commission of non-violent crimes.",
    "Privacy" => "Content that contains sensitive, personal information about private individuals.",
    "Self-Harm" => "Content that enables, encourages, or excuses acts of intentional self-harm.",
    "Sex Crimes" => "Content that enables, encourages, or excuses the commission of sex crimes.",
    "Sexual Content" => "Content that contains erotic descriptions or explicit depictions of sex acts.",
    "Specialized Advice" => "Content that contains financial, medical, or legal advice.
          Financial advice includes guidance on investments, stocks, bonds, or any financial planning.",
    "Violent Crimes" => "Content that enables, encourages, or excuses the commission of violent crimes."
  }


  def moderate_message_with_definitions(message, unsafe_category_definitions)
    # 格式化不安全类别字符串，每个类别及其定义占一行
    unsafe_category_str = unsafe_category_definitions
      .map { |category, definition| "#{category}: #{definition}" }
      .join("\n")

    # 构建发送给 Claude 的提示，包含消息和不安全类别
    assessment_prompt = <<~PROMPT.chomp
      Determine whether the following message warrants moderation, based on the unsafe categories outlined below.

      Message:
      <message>#{message}</message>

      Unsafe Categories and Their Definitions:
      <categories>
      #{unsafe_category_str}
      </categories>

      It's important that you remember all unsafe categories and their definitions.

      Respond with ONLY a JSON object, using the format below:
      {
        "violation": <Boolean field denoting whether the message should be moderated>,
        "categories": [Comma-separated list of violated categories],
        "explanation": [Optional. Only include if there is a violation.]
      }
      Do not include markdown formatting or code fences in your response.
    PROMPT

    # 向 Claude 发送内容审核请求
    response = CLIENT.messages.create(
      model: "claude-haiku-4-5-20251001", # Using the Haiku model for lower costs
      max_tokens: 200,
      messages: [{role: :user, content: assessment_prompt}]
    )

    # 解析 Claude 返回的 JSON 响应
    text_block = response.content.find { it.type == :text }
    assessment = JSON.parse(text_block.text)

    # 从评估结果中提取违规状态
    contains_violation = assessment["violation"]

    # 如果存在违规，获取类别和解释；否则使用空的默认值
    violated_categories = contains_violation ? assessment.fetch("categories", []) : []
    explanation = contains_violation ? assessment["explanation"] : nil

    [contains_violation, violated_categories, explanation]
  end


  # 处理每条评论并打印结果
  USER_COMMENTS.each do |comment|
    puts "\nComment: #{comment}"
    violation, violated_categories, explanation = moderate_message_with_definitions(comment, UNSAFE_CATEGORY_DEFINITIONS)

    if violation
      puts "Violated Categories: #{violated_categories.join(", ")}"
      puts "Explanation: #{explanation}"
    else
      puts "No issues detected."
    end
  end
  ```
</CodeGroup>

`moderate_message_with_definitions` 函数在之前的 `moderate_message` 函数基础上进行了扩展，允许每个不安全类别与详细定义配对。在代码中，这是通过将原函数中的 `unsafe_categories` 集合替换为 `unsafe_category_definitions` 映射来实现的。该映射将每个不安全类别与其对应的定义配对。类别名称及其定义都包含在提示中。

值得注意的是，`Specialized Advice` 类别的定义现在明确指定了应禁止的金融建议类型。因此，之前通过了 `moderate_message` 评估的评论 `It is a great time to invest in gold!` 现在会触发违规。

### 考虑批量处理

在不需要实时审核的情况下，为了降低成本，可以考虑批量审核消息。在提示的上下文中包含多条消息，并要求 Claude 评估哪些消息应被审核。

<CodeGroup exclude="shell">
  ```python Python
  def batch_moderate_messages(messages, unsafe_categories):
      # 将不安全类别转换为字符串，每个类别占一行
      unsafe_category_str = "\n".join(unsafe_categories)

      # 格式化消息字符串，每条消息用类 XML 标签包裹并分配一个 ID
      messages_str = "\n".join(
          [f"<message id={idx}>{msg}</message>" for idx, msg in enumerate(messages)]
      )

      # 构建发送给 Claude 的提示，包含消息和不安全类别
      assessment_prompt = f"""Determine the messages to moderate, based on the unsafe categories outlined below.

  Messages:
  <messages>
  {messages_str}
  </messages>

  Unsafe Categories:
  <categories>
  {unsafe_category_str}
  </categories>

  Respond with ONLY a JSON object, using the format below:
  {{
    "violations": [
      {{
        "id": <message id>,
        "categories": [list of violated categories],
        "explanation": <Explanation of why there's a violation>
      }}
    ]
  }}

  Important Notes:
  - Remember to analyze every message for a violation.
  - Select any number of violations that reasonably apply.
  - Do not include markdown formatting or code fences in your response."""

      # 向 Claude 发送内容审核请求
      response = client.messages.create(
          model="claude-haiku-4-5-20251001",  # Using the Haiku model for lower costs
          max_tokens=2048,  # Increased max token count to handle batches
          messages=[{"role": "user", "content": assessment_prompt}],
      )

      # 解析 Claude 返回的 JSON 响应
      text_block = next(block for block in response.content if block.type == "text")
      assessment = json.loads(text_block.text)
      return assessment


  # 批量处理评论并获取响应
  response_obj = batch_moderate_messages(user_comments, unsafe_categories)

  # 打印每个检测到的违规的结果
  for violation in response_obj["violations"]:
      print(f"""Comment: {user_comments[violation["id"]]}
  Violated Categories: {", ".join(violation["categories"])}
  Explanation: {violation["explanation"]}
  """)
  ```

  ```typescript TypeScript
  // Claude 返回的 JSON 批量评估结果的结构
  interface BatchAssessment {
    violations: {
      id: number;
      categories: string[];
      explanation: string;
    }[];
  }

  async function batchModerateMessages(
    messages: string[],
    unsafeCategories: string[]
  ): Promise<BatchAssessment> {
    // 将不安全类别转换为字符串，每个类别占一行
    const unsafeCategoryStr = unsafeCategories.join("\n");

    // 格式化消息字符串，每条消息用类 XML 标签包裹并分配一个 ID
    const messagesStr = messages
      .map((msg, idx) => `<message id=${idx}>${msg}</message>`)
      .join("\n");

    // 构建发送给 Claude 的提示，包含消息和不安全类别
    const assessmentPrompt = `Determine the messages to moderate, based on the unsafe categories outlined below.

  Messages:
  <messages>
  ${messagesStr}
  </messages>

  Unsafe Categories:
  <categories>
  ${unsafeCategoryStr}
  </categories>

  Respond with ONLY a JSON object, using the format below:
  {
    "violations": [
      {
        "id": <message id>,
        "categories": [list of violated categories],
        "explanation": <Explanation of why there's a violation>
      }
    ]
  }

  Important Notes:
  - Remember to analyze every message for a violation.
  - Select any number of violations that reasonably apply.
  - Do not include markdown formatting or code fences in your response.`;

    // 向 Claude 发送内容审核请求
    const response = await client.messages.create({
      model: "claude-haiku-4-5-20251001", // Using the Haiku model for lower costs
      max_tokens: 2048, // Increased max token count to handle batches
      messages: [{ role: "user", content: assessmentPrompt }]
    });

    // 解析 Claude 返回的 JSON 响应
    const textBlock = response.content.find((block) => block.type === "text");
    if (!textBlock) {
      throw new Error("Expected a text block in the response");
    }
    const assessment: BatchAssessment = JSON.parse(textBlock.text);
    return assessment;
  }

  // 批量处理评论并获取响应
  const batchAssessment = await batchModerateMessages(userComments, unsafeCategories);

  // 打印每个检测到的违规的结果
  for (const violation of batchAssessment.violations) {
    console.log(`Comment: ${userComments[violation.id]}
  Violated Categories: ${violation.categories.join(", ")}
  Explanation: ${violation.explanation}
  `);
  }
  ```

  ```csharp C#
  async Task<JsonNode> BatchModerateMessages(IReadOnlyList<string> messages, IReadOnlyList<string> categories)
  {
      // 将不安全类别转换为字符串，每个类别占一行
      var unsafeCategoryText = string.Join("\n", categories);

      // 格式化消息字符串，将每条消息包裹在类 XML 标签中并赋予 ID
      var messagesText = string.Join(
          "\n",
          messages.Select((message, index) => $"<message id={index}>{message}</message>")
      );

      // 构建发送给 Claude 的提示，包含消息和不安全类别
      var assessmentPrompt = $$"""
  Determine the messages to moderate, based on the unsafe categories outlined below.

  Messages:
  <messages>
  {{messagesText}}
  </messages>

  Unsafe Categories:
  <categories>
  {{unsafeCategoryText}}
  </categories>

  Respond with ONLY a JSON object, using the format below:
  {
    "violations": [
      {
        "id": <message id>,
        "categories": [list of violated categories],
        "explanation": <Explanation of why there's a violation>
      }
    ]
  }

  Important Notes:
  - Remember to analyze every message for a violation.
  - Select any number of violations that reasonably apply.
  - Do not include markdown formatting or code fences in your response.
  """;

      // 向 Claude 发送内容审核请求
      var response = await client.Messages.Create(
          new()
          {
              Model = Model.ClaudeHaiku4_5_20251001, // Using the Haiku model for lower costs
              MaxTokens = 2048, // Increased max token count to handle batches
              Messages = [new() { Role = Role.User, Content = assessmentPrompt }],
          }
      );

      // 将第一个内容块收窄为文本块，然后解析 Claude 的 JSON 响应
      if (!response.Content[0].TryPickText(out var textBlock))
      {
          throw new InvalidOperationException("Expected a text response from Claude.");
      }
      return JsonNode.Parse(textBlock.Text)!;
  }

  // 批量处理评论并获取响应
  var moderationResults = await BatchModerateMessages(userComments, unsafeCategories);

  // 打印每个检测到的违规的结果
  foreach (var violation in moderationResults["violations"]!.AsArray())
  {
      var flaggedComment = userComments[violation!["id"]!.GetValue<int>()];
      var violatedCategories = string.Join(
          ", ",
          violation["categories"]!.AsArray().Select(category => category!.GetValue<string>())
      );
      var explanation = violation["explanation"]!.GetValue<string>();

      Console.WriteLine($"""
          Comment: {flaggedComment}
          Violated Categories: {violatedCategories}
          Explanation: {explanation}

          """);
  }
  ```

  ```go Go
  // batchViolation 是 Claude 返回的 "violations" 数组中的一项：包含违规
  // 消息的索引、其违反的类别以及原因。
  type batchViolation struct {
  	ID          int      `json:"id"`
  	Categories  []string `json:"categories"`
  	Explanation string   `json:"explanation"`
  }

  func batchModerateMessages(messages []string, unsafeCategories []string) []batchViolation {
  	// 将不安全类别转换为字符串，每个类别占一行
  	unsafeCategoryStr := strings.Join(unsafeCategories, "\n")

  	// 格式化消息字符串，每条消息用类 XML 标签包裹并赋予 ID
  	messageLines := make([]string, len(messages))
  	for i, message := range messages {
  		messageLines[i] = fmt.Sprintf("<message id=%d>%s</message>", i, message)
  	}
  	messagesStr := strings.Join(messageLines, "\n")

  	// 构建发送给 Claude 的提示，包含消息和不安全类别
  	assessmentPrompt := fmt.Sprintf(`Determine the messages to moderate, based on the unsafe categories outlined below.

  Messages:
  <messages>
  %s
  </messages>

  Unsafe Categories:
  <categories>
  %s
  </categories>

  Respond with ONLY a JSON object, using the format below:
  {
    "violations": [
      {
        "id": <message id>,
        "categories": [list of violated categories],
        "explanation": <Explanation of why there's a violation>
      }
    ]
  }

  Important Notes:
  - Remember to analyze every message for a violation.
  - Select any number of violations that reasonably apply.
  - Do not include markdown formatting or code fences in your response.`, messagesStr, unsafeCategoryStr)

  	// 向 Claude 发送内容审核请求
  	response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeHaiku4_5_20251001, // Using the Haiku model for lower costs
  		MaxTokens: 2048,                                   // Increased max token count to handle batches
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock(assessmentPrompt)),
  		},
  	})
  	if err != nil {
  		log.Fatal(err)
  	}

  	// 在读取文本之前，将第一个内容块收窄为文本块
  	textBlock, ok := response.Content[0].AsAny().(anthropic.TextBlock)
  	if !ok {
  		log.Fatalf("expected a text block, got %q", response.Content[0].Type)
  	}

  	// 解析 Claude 返回的 JSON 响应
  	var assessment struct {
  		Violations []batchViolation `json:"violations"`
  	}
  	if err := json.Unmarshal([]byte(textBlock.Text), &assessment); err != nil {
  		log.Fatal(err)
  	}
  	return assessment.Violations
  }

  // moderateAllCommentsAsBatch 在单个请求中审核整批评论，
  // 并打印每个检测到的违规的结果。
  func moderateAllCommentsAsBatch() {
  	// 处理这批评论并获取响应
  	violations := batchModerateMessages(userComments, unsafeCategories)

  	// 打印每个检测到的违规的结果
  	for _, violation := range violations {
  		fmt.Printf(`Comment: %s
  Violated Categories: %s
  Explanation: %s

  `, userComments[violation.ID], strings.Join(violation.Categories, ", "), violation.Explanation)
  	}
  }

  ```

  ```java Java
  JsonNode batchModerateMessages(List<String> messages, List<String> unsafeCategories)
          throws JsonProcessingException {
      // 将不安全类别转换为字符串，每个类别占一行
      String unsafeCategoryStr = String.join("\n", unsafeCategories);

      // 格式化消息字符串，每条消息用类 XML 标签包裹并分配一个 ID
      String messagesStr = IntStream.range(0, messages.size())
              .mapToObj(idx -> "<message id=%d>%s</message>".formatted(idx, messages.get(idx)))
              .collect(Collectors.joining("\n"));

      // 构建发送给 Claude 的提示，包含消息和不安全类别
      String assessmentPrompt = """
          Determine the messages to moderate, based on the unsafe categories outlined below.

          Messages:
          <messages>
          %s
          </messages>

          Unsafe Categories:
          <categories>
          %s
          </categories>

          Respond with ONLY a JSON object, using the format below:
          {
            "violations": [
              {
                "id": <message id>,
                "categories": [list of violated categories],
                "explanation": <Explanation of why there's a violation>
              }
            ]
          }

          Important Notes:
          - Remember to analyze every message for a violation.
          - Select any number of violations that reasonably apply.
          - Do not include markdown formatting or code fences in your response."""
              .formatted(messagesStr, unsafeCategoryStr);

      // 向 Claude 发送内容审核请求
      Message response = client.messages().create(MessageCreateParams.builder()
              .model(Model.CLAUDE_HAIKU_4_5_20251001) // Using the Haiku model for lower costs
              .maxTokens(2048) // Increased max token count to handle batches
              .addUserMessage(assessmentPrompt)
              .build());

      // 解析 Claude 返回的 JSON 响应
      String assessmentJson = response.content().stream()
              .flatMap(contentBlock -> contentBlock.text().stream())
              .findFirst()
              .orElseThrow()
              .text();
      return new ObjectMapper().readTree(assessmentJson);
  }

  // 批量处理评论，并打印每个检测到的违规结果
  void printBatchViolations() throws JsonProcessingException {
      JsonNode response = batchModerateMessages(userComments, unsafeCategories);

      ObjectMapper mapper = new ObjectMapper();
      for (JsonNode violation : response.required("violations")) {
          List<String> violatedCategories =
                  mapper.convertValue(violation.required("categories"), new TypeReference<List<String>>() {});
          IO.println("""
                  Comment: %s
                  Violated Categories: %s
                  Explanation: %s
                  """.formatted(
                          userComments.get(violation.required("id").asInt()),
                          String.join(", ", violatedCategories),
                          violation.required("explanation").asText()));
      }
  }
  ```

  ```php PHP
  $batchModerateMessages = function (array $messages, array $unsafeCategories) use ($client): array {
      // 将不安全类别转换为字符串，每个类别占一行
      $unsafeCategoryStr = implode("\n", $unsafeCategories);

      // 格式化消息字符串，每条消息用类 XML 标签包裹并分配 ID
      $messageLines = [];
      foreach ($messages as $idx => $msg) {
          $messageLines[] = "<message id={$idx}>{$msg}</message>";
      }
      $messagesStr = implode("\n", $messageLines);

      // 构建发送给 Claude 的提示，包含消息和不安全类别
      $assessmentPrompt = <<<PROMPT
      Determine the messages to moderate, based on the unsafe categories outlined below.

      Messages:
      <messages>
      {$messagesStr}
      </messages>

      Unsafe Categories:
      <categories>
      {$unsafeCategoryStr}
      </categories>

      Respond with ONLY a JSON object, using the format below:
      {
        "violations": [
          {
            "id": <message id>,
            "categories": [list of violated categories],
            "explanation": <Explanation of why there's a violation>
          }
        ]
      }

      Important Notes:
      - Remember to analyze every message for a violation.
      - Select any number of violations that reasonably apply.
      - Do not include markdown formatting or code fences in your response.
      PROMPT;

      // 向 Claude 发送内容审核请求
      $response = $client->messages->create(
          model: 'claude-haiku-4-5-20251001', // Using the Haiku model for lower costs
          maxTokens: 2048, // Increased max token count to handle batches
          messages: [['role' => 'user', 'content' => $assessmentPrompt]],
      );

      // 解析 Claude 返回的 JSON 响应。SDK 会将每个内容块解码
      // 为其具体类，因此需先找到 TextBlock 再读取文本。
      $textBlock = array_find($response->content, fn ($block) => $block instanceof TextBlock)
          ?? throw new RuntimeException('Expected a text block in the response.');

      return json_decode($textBlock->text, associative: true, flags: JSON_THROW_ON_ERROR);
  };

  // 批量处理评论并获取响应
  $responseObj = $batchModerateMessages($userComments, $unsafeCategories);

  // 打印每个检测到的违规的结果
  foreach ($responseObj['violations'] as $violation) {
      echo "Comment: {$userComments[$violation['id']]}\n";
      echo 'Violated Categories: ' . implode(', ', $violation['categories']) . "\n";
      echo "Explanation: {$violation['explanation']}\n\n";
  }
  ```

  ```ruby Ruby
  def batch_moderate_messages(messages, unsafe_categories)
    # 将不安全类别转换为字符串，每个类别占一行
    unsafe_category_str = unsafe_categories.join("\n")

    # 格式化消息字符串，每条消息用类 XML 标签包裹并分配一个 ID
    messages_str = messages
      .map.with_index { |message, index| "<message id=#{index}>#{message}</message>" }
      .join("\n")

    # 构建发送给 Claude 的提示，包含消息和不安全类别
    assessment_prompt = <<~PROMPT.chomp
      Determine the messages to moderate, based on the unsafe categories outlined below.

      Messages:
      <messages>
      #{messages_str}
      </messages>

      Unsafe Categories:
      <categories>
      #{unsafe_category_str}
      </categories>

      Respond with ONLY a JSON object, using the format below:
      {
        "violations": [
          {
            "id": <message id>,
            "categories": [list of violated categories],
            "explanation": <Explanation of why there's a violation>
          }
        ]
      }

      Important Notes:
      - Remember to analyze every message for a violation.
      - Select any number of violations that reasonably apply.
      - Do not include markdown formatting or code fences in your response.
    PROMPT

    # 向 Claude 发送内容审核请求
    response = CLIENT.messages.create(
      model: "claude-haiku-4-5-20251001", # Using the Haiku model for lower costs
      max_tokens: 2048, # Increased max token count to handle batches
      messages: [{role: :user, content: assessment_prompt}]
    )

    # 解析 Claude 返回的 JSON 响应
    text_block = response.content.find { it.type == :text }
    JSON.parse(text_block.text)
  end


  # 批量处理评论并获取响应
  response_obj = batch_moderate_messages(USER_COMMENTS, UNSAFE_CATEGORIES)

  # 打印每个检测到的违规的结果
  response_obj["violations"].each do |violation|
    puts <<~RESULT
      Comment: #{USER_COMMENTS[violation["id"]]}
      Violated Categories: #{violation["categories"].join(", ")}
      Explanation: #{violation["explanation"]}

    RESULT
  end
  ```
</CodeGroup>

在此示例中，`batch_moderate_messages` 函数通过单次 Claude API 调用处理整批消息的审核。 在函数内部，会创建一个提示，其中包括待评估的消息列表和不安全内容类别。该提示指示 Claude 返回一个 JSON 对象，列出所有包含违规内容的消息。响应中的每条消息都通过其 `id` 进行标识，该 `id` 对应于消息在批次中的位置。 请记住，为您的特定需求找到最佳批次大小可能需要一些实验。虽然较大的批次大小可以降低成本，但也可能导致质量略有下降。此外，您可能需要增加 Claude API 调用中的 `max_tokens` 参数以容纳更长的响应。有关所选模型可输出的最大令牌数的详细信息，请参阅[模型比较表](https://platform.claude.com/docs/zh-CN/models/overview#latest-models-comparison)。

<CardGroup cols={2}>
  <Card title="内容审核 cookbook" icon="link" href="https://platform.claude.com/cookbook/misc-building-moderation-filter">
    查看一个完整实现的基于代码的示例，了解如何使用 Claude 进行内容审核。
  </Card>

  <Card title="缓解越狱" icon="link" href="https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks">
    探索用于审核与 Claude 交互的防护技术。
  </Card>
</CardGroup>
