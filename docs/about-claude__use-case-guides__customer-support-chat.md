---
title: 客户支持智能体
url: https://platform.claude.com/docs/zh-CN/about-claude/use-case-guides/customer-support-chat
description: 使用 Claude 构建一个客户支持聊天机器人，能够回答产品问题、保持话题聚焦，并通过工具使用生成报价。
---

## 前提条件

要按照本指南操作，您需要：

* 一个 Claude API 密钥（设置为 `ANTHROPIC_API_KEY` 环境变量）
* Python 3.10 或更高版本

安装所需的软件包：

```bash
pip install anthropic streamlit python-dotenv
```

## 使用 Claude 构建之前

### 决定是否将 Claude 用于支持聊天

以下是一些关键指标，表明您应该采用像 Claude 这样的"large language model"（大型语言模型），即 LLM，来自动化客户支持流程的部分环节：

<AccordionGroup>
  <Accordion title="大量重复性查询">
    Claude 擅长高效处理大量相似的问题，从而让人工客服腾出精力处理更复杂的问题。
  </Accordion>

  <Accordion title="需要快速综合信息">
    Claude 可以快速从庞大的知识库中检索、处理和整合信息，而人工客服可能需要时间进行研究或查阅多个来源。
  </Accordion>

  <Accordion title="需要 24/7 全天候可用">
    Claude 可以不知疲倦地提供全天候支持，而为人工客服安排持续覆盖的排班可能成本高昂且充满挑战。
  </Accordion>

  <Accordion title="高峰期快速扩展">
    Claude 可以应对查询量的突然增加，而无需招聘和培训额外的员工。
  </Accordion>

  <Accordion title="一致的品牌声音">
    您可以指示 Claude 始终如一地体现您品牌的语气和价值观，而人工客服的沟通风格可能各不相同。
  </Accordion>
</AccordionGroup>

选择 Claude 而非其他 LLM 的一些考虑因素：

* 您优先考虑自然、细腻的对话：Claude 精湛的语言理解能力可以实现更自然、更具上下文感知的对话，比与其他 LLM 的聊天更像人类交流。
* 您经常收到复杂且开放式的查询：Claude 可以处理广泛的话题和询问，而不会生成千篇一律的回复，也无需针对用户话语的各种排列组合进行大量编程。
* 您需要可扩展的多语言支持：Claude 的多语言能力使其能够以 200 多种语言进行对话，而无需为每种支持的语言配备单独的聊天机器人或进行大量翻译流程。

### 定义您理想的聊天交互

勾勒出一个理想的客户交互，以定义您期望客户如何以及何时与 Claude 交互。这一概要将有助于确定您解决方案的技术要求。

以下是汽车保险客户支持的一个聊天交互示例：

* **客户：** 发起支持聊天体验
  * **Claude：** 热情地问候客户并开启对话

* **客户：** 询问其新电动汽车的保险事宜
  * **Claude：** 提供有关电动汽车保险覆盖范围的相关信息

* **客户：** 询问与电动汽车保险独特需求相关的问题
  * **Claude：** 以准确且信息丰富的答案作出回应，并提供来源链接

* **客户：** 询问与保险或汽车无关的离题问题
  * **Claude：** 澄清其不讨论无关话题，并将用户引导回汽车保险

* **客户：** 表示对保险报价感兴趣

  * **Claude：** 提出一系列问题以确定合适的报价，并根据客户的回答进行调整
  * **Claude：** 发送使用报价生成 API 工具的请求，并附上从用户处收集的必要信息
  * **Claude：** 接收来自 API 工具使用的响应信息，将信息综合成自然的回复，并向用户呈现所提供的报价

* **客户：** 提出后续问题

  * **Claude：** 根据需要回答后续问题
  * **Claude：** 引导客户进入保险流程的下一步并结束对话

<Tip>
  在您为自己的用例编写的真实示例中，您可能会发现写出此交互中的实际措辞很有用，这样您也可以了解您希望 Claude 具备的理想语气、回复长度和详细程度。
</Tip>

### 将交互分解为独立的任务

客户支持聊天是多种不同任务的集合，从问题解答到信息检索再到对请求采取行动，全部包含在单次客户交互中。在开始构建之前，请将您理想的客户交互分解为您希望 Claude 能够执行的每一项任务。这可以确保您能够针对每项任务对 Claude 进行提示和评估，并让您充分了解在编写测试用例时需要考虑的交互范围。

<Tip>
  客户有时会发现，将其可视化为一个交互流程图很有帮助，该流程图展示了根据用户请求可能出现的对话转折点。
</Tip>

以下是与示例保险交互相关的关键任务：

1. 问候和一般指导

   * 热情地问候客户并开启对话
   * 提供有关公司和交互的一般信息

2. 产品信息

   * 提供有关电动汽车保险覆盖范围的信息
     <Note>
       这将要求 Claude 在其上下文中拥有必要的信息，并且可能意味着需要 

       [RAG 集成](https://platform.claude.com/cookbook/capabilities-retrieval-augmented-generation-guide)

       。
     </Note>
   * 回答与电动汽车保险独特需求相关的问题
   * 回答有关报价或保险详情的后续问题
   * 在适当时提供来源链接

3. 对话管理

   * 保持话题聚焦（汽车保险）
   * 将离题问题重新引导回相关主题

4. 报价生成

   * 提出适当的问题以确定报价资格
   * 根据客户的回答调整问题
   * 将收集到的信息提交给报价生成 API
   * 向客户呈现所提供的报价

### 建立成功标准

与您的支持团队合作，[定义成功标准并编写详细的评估](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests)，其中包含可衡量的基准和目标。

以下是可用于评估 Claude 执行所定义任务的成功程度的标准和基准：

<AccordionGroup>
  <Accordion title="查询理解准确率">
    该指标评估 Claude 在各种话题上理解客户询问的准确程度。通过审查对话样本并评估 Claude 是否正确理解了客户意图、关键的后续步骤、成功解决的标准等来衡量这一点。目标是达到 95% 或更高的理解准确率。
  </Accordion>

  <Accordion title="回复相关性">
    这评估 Claude 的回复在多大程度上解决了客户的具体问题或疑虑。评估一组对话并对每个回复的相关性进行评分（使用基于 LLM 的评分以实现规模化）。目标是达到 90% 或以上的相关性得分。
  </Accordion>

  <Accordion title="回复准确性">
    根据在上下文中提供给 Claude 的信息，评估向用户提供的一般公司和产品信息的正确性。目标是在这些介绍性信息中达到 100% 的准确率。
  </Accordion>

  <Accordion title="引用提供相关性">
    跟踪所提供链接或来源的频率和相关性。目标是在 80% 的可能受益于额外信息的交互中提供相关来源。
  </Accordion>

  <Accordion title="话题遵循度">
    衡量 Claude 保持话题聚焦的程度，例如示例实现中的汽车保险话题。目标是 95% 的回复与汽车保险或客户的具体查询直接相关。
  </Accordion>

  <Accordion title="内容生成有效性">
    衡量 Claude 在判断何时生成信息性内容方面的成功程度，以及该内容的相关程度。例如，在此实现中，您将判断 Claude 对何时生成报价的理解程度以及该报价的准确程度。目标是 100% 的准确率，因为这是成功客户交互的关键信息。
  </Accordion>

  <Accordion title="升级效率">
    这衡量 Claude 识别查询何时需要人工干预并适当升级的能力。跟踪正确升级的对话与本应升级但未升级的对话的百分比。目标是达到 95% 或更高的升级准确率。
  </Accordion>
</AccordionGroup>

以下是可用于评估采用 Claude 进行支持的业务影响的标准和基准：

<AccordionGroup>
  <Accordion title="情绪维持">
    这评估 Claude 在整个对话过程中维持或改善客户情绪的能力。使用情绪分析工具衡量每次对话开始和结束时的情绪。目标是在 90% 的交互中维持或改善情绪。
  </Accordion>

  <Accordion title="分流率">
    聊天机器人在无需人工干预的情况下成功处理的客户询问的百分比。通常目标是 70-80% 的分流率，具体取决于询问的复杂程度。
  </Accordion>

  <Accordion title="客户满意度得分">
    衡量客户对其聊天机器人交互的满意程度。通常通过交互后调查完成。目标是 CSAT 得分达到 5 分中的 4 分或更高。
  </Accordion>

  <Accordion title="平均处理时间">
    聊天机器人解决一个询问所需的平均时间。这因问题的复杂程度而差异很大，但一般而言，目标是比人工客服更低的 AHT。
  </Accordion>
</AccordionGroup>

## 如何将 Claude 实现为客户服务智能体

### 选择合适的 Claude 模型

模型的选择取决于成本、准确性和响应时间之间的权衡。

对于客户支持聊天，Claude Opus 5 非常适合在智能、"latency"（延迟）和成本之间取得平衡，包括需要在漫长的多步骤对话中进行深度推理的最复杂支持场景。然而，对于包含多个提示（包括 RAG、工具使用或长上下文提示）的对话流程，Claude Haiku 4.5 可能更适合用于优化延迟。

### 构建强大的提示

将 Claude 用于客户支持需要 Claude 拥有足够的指导和上下文以作出适当回应，同时具备足够的灵活性来处理各种各样的客户询问。

首先编写强大提示的各个要素，从"system prompt"（系统提示）开始。创建一个名为 `config.py` 的文件，并将以下每个代码块添加到其中：

```python
IDENTITY = """You are Eva, a friendly and knowledgeable AI assistant for Acme Insurance
Company. Your role is to warmly welcome customers and provide information on
Acme's insurance offerings, which include car insurance and electric car
insurance. You can also help customers get quotes for their insurance needs."""
```

<Tip>
  虽然您可能倾向于将所有信息放入系统提示中，以此将指令与用户对话分开，但实际上 Claude 在将大部分提示内容写入第一个 

  `User`

   轮次时效果最佳（唯一的例外是角色提示）。请在

  [通过系统提示赋予 Claude 角色](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#give-claude-a-role)

  中阅读更多内容。
</Tip>

最好将复杂的提示分解为多个小节，并一次编写一个部分。对于每项任务，通过遵循逐步流程来定义 Claude 出色完成任务所需的提示部分，您可能会取得更大的成功。对于这个汽车保险客户支持示例，您将从"问候和一般指导"任务开始，逐一编写提示的所有部分。这也使调试提示变得更容易，因为您可以更快地调整整体提示的各个部分。

```python
STATIC_GREETINGS_AND_GENERAL = """
<static_context>
Acme Auto Insurance: Your Trusted Companion on the Road

About:
At Acme Insurance, we understand that your vehicle is more than just a mode of transportation—it's your ticket to life's adventures.
Since 1985, we've been crafting auto insurance policies that give drivers the confidence to explore, commute, and travel with peace of mind.
Whether you're navigating city streets or embarking on cross-country road trips, Acme is there to protect you and your vehicle.
Our innovative auto insurance policies are designed to adapt to your unique needs, covering everything from fender benders to major collisions.
With Acme's award-winning customer service and swift claim resolution, you can focus on the joy of driving while we handle the rest.
We're not just an insurance provider—we're your co-pilot in life's journeys.
Choose Acme Auto Insurance and experience the assurance that comes with superior coverage and genuine care. Because at Acme, we don't just
insure your car—we fuel your adventures on the open road.

Note: We also offer specialized coverage for electric vehicles, ensuring that drivers of all car types can benefit from our protection.

Acme Insurance offers the following products:
- Car insurance
- Electric car insurance
- Two-wheeler insurance

Business hours: Monday-Friday, 9 AM - 5 PM EST
Customer service number: 1-800-123-4567
</static_context>
"""
```

然后对您的汽车保险和电动汽车保险信息执行相同的操作。

```python
STATIC_CAR_INSURANCE = """
<static_context>
Car Insurance Coverage:
Acme's car insurance policies typically cover:
1. Liability coverage: Pays for bodily injury and property damage you cause to others.
2. Collision coverage: Pays for damage to your car in an accident.
3. Comprehensive coverage: Pays for damage to your car from non-collision incidents.
4. Medical payments coverage: Pays for medical expenses after an accident.
5. Uninsured/underinsured motorist coverage: Protects you if you're hit by a driver with insufficient insurance.

Optional coverages include:
- Rental car reimbursement
- Roadside assistance
- New car replacement
</static_context>
"""

STATIC_ELECTRIC_CAR_INSURANCE = """
<static_context>
Electric Car Insurance:
Our specialized electric car insurance goes beyond traditional auto coverage,
offering tailored protection for your high-tech investment. In addition to standard coverages, we offer:
- Safeguarding expensive battery systems
- Covering home charging equipment
- Adapting to the rapidly evolving world of electric vehicles

Whether you're driving a Rivian, Nissan Leaf, Chevrolet Bolt,
or any other electric vehicle, our knowledgeable agents are ready to craft a policy
that gives you the confidence to enjoy every electron-powered mile.
</static_context>
"""
```

现在您已经有了静态内容，请添加至少 4-5 个"良好"交互示例来引导 Claude 的回复。这些示例应能代表您理想的客户交互，并且可以包含护栏和工具调用等元素。

```python
EXAMPLES = """
Here are a few examples of how you can interact with customers:

<example 1>
H: Hi, do you offer commercial insurance for small businesses?

A: Ah! Unfortunately, we don't offer commercial insurance at this time.
However, we do provide a range of personal insurance products including car insurance and electric car insurance.
Would you like to know more about these offerings?
</example 1>

<example 2>
H: Hi, what are your business hours?

A: Our business hours are Monday to Friday, 9 AM to 5 PM EST.
If you have any questions or need assistance, feel free to reach out during those times.
How else can I assist you today?
</example 2>

<example 3>
H: I'm shopping for car insurance. What does Acme offer?

A: Great choice! Acme's car insurance policies typically include:
- Liability coverage: Protects you if you're at fault in an accident
- Collision coverage: Pays for damage to your car in an accident
- Comprehensive coverage: Covers non-collision incidents like theft or vandalism
- Medical payments coverage: Pays for medical expenses after an accident
- Uninsured/underinsured motorist coverage: Protects you from drivers with insufficient insurance

We also offer optional coverages like rental car reimbursement, roadside assistance, and new car replacement.
Would you like more details on any of these coverages, or do you have specific questions about insuring your vehicle?
</example 3>

<example 4>
H: I'd like to hear about your electric car insurance.

A: Certainly! Our electric car insurance is specially designed to meet the unique needs of electric vehicle owners.
In addition to all the standard coverages we offer for regular cars, our electric car policies include:

- Battery coverage: Protection against damage or degradation of your vehicle's expensive battery system
- Tech package: Covers repairs or replacement of advanced electronics and onboard computer systems
- Charging station coverage: Protects both home and portable charging equipment
- Cybersecurity protection: Coverage against hacking or digital threats
- Specialized roadside assistance: Includes flatbed towing to protect the vehicle's undercarriage

We cover a wide range of electric vehicles, including popular models from Tesla, Rivian, Nissan, Chevrolet, and more.
Is there a specific electric vehicle you're looking to insure, or would you like more details on any of these coverages?
</example 4>

<example 5>
H: I'd like to get a quote for my car insurance.

A: Certainly! I'd be happy to help you get a quote for your car insurance.
To provide you with an accurate quote, I'll need to collect some information about your vehicle and the primary driver.
Let's start with the basics:

1. What is the make and model of your vehicle?
2. What year was it manufactured?
3. Approximately how many miles have you driven?
4. What is the age of the primary driver?

Once you provide this information, I'll use our quoting tool to generate a personalized insurance quote for you.
</example 5>
"""
```

您还需要包含任何重要的指令，概述 Claude 应如何与客户交互的注意事项和禁忌。 这可能源自品牌护栏或支持政策。

```python
ADDITIONAL_GUARDRAILS = """Please adhere to the following guardrails:
1. Only provide information about insurance types listed in our offerings.
2. If asked about an insurance type we don't offer, politely state
that we don't provide that service.
3. Do not speculate about future product offerings or company plans.
4. Don't make promises or enter into agreements it's not authorized to make.
You only provide information and guidance.
5. Do not mention any competitor's products or services.
"""
```

现在将所有这些部分组合成一个字符串，用作您的提示。

```python
TASK_SPECIFIC_INSTRUCTIONS = " ".join(
    [
        STATIC_GREETINGS_AND_GENERAL,
        STATIC_CAR_INSURANCE,
        STATIC_ELECTRIC_CAR_INSURANCE,
        EXAMPLES,
        ADDITIONAL_GUARDRAILS,
    ]
)
```

### 通过工具使用添加动态和智能体能力

Claude 能够使用客户端"tool use"（工具使用）功能动态地采取行动和检索信息。首先列出提示应使用的任何外部工具或 API。

对于此示例，从一个用于计算报价的工具开始。

<Tip>
  提醒一下，此工具不会执行实际计算，它只会向应用程序发出信号，表明应使用某个工具以及所指定的参数。
</Tip>

将模型名称、工具定义和一个存根实现添加到 `config.py`：

```python
import time

MODEL = "claude-opus-5"

TOOLS = [
    {
        "name": "get_quote",
        "description": "Calculate the insurance quote based on user input. Returned value is per month premium.",
        "input_schema": {
            "type": "object",
            "properties": {
                "make": {"type": "string", "description": "The make of the vehicle."},
                "model": {"type": "string", "description": "The model of the vehicle."},
                "year": {
                    "type": "integer",
                    "description": "The year the vehicle was manufactured.",
                },
                "mileage": {
                    "type": "integer",
                    "description": "The mileage on the vehicle.",
                },
                "driver_age": {
                    "type": "integer",
                    "description": "The age of the primary driver.",
                },
            },
            "required": ["make", "model", "year", "mileage", "driver_age"],
        },
    }
]


def get_quote(make, model, year, mileage, driver_age):
    """Returns the premium per month in USD"""
    # 您可以调用 http 端点或数据库来获取报价。
    # 此处我们模拟 1 秒延迟，并返回固定报价 100。
    time.sleep(1)
    return 100
```

### 部署您的提示

如果不将提示部署到测试生产环境中并[运行评估](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests)，就很难知道您的提示效果如何。使用该提示、Anthropic SDK 和用于用户界面的 Streamlit 构建一个小型应用程序。

在名为 `chatbot.py` 的文件（或您所用语言中的等效模块）中，设置 ChatBot 类，该类将封装与 Anthropic SDK 的交互。

该类应有两个主要方法：一个调用 API 生成消息，另一个处理每个传入的用户输入。

<CodeGroup exclude="shell">
  ```python Python
  # 在您的 chatbot.py 中，从上面编写的 config.py 导入以下内容：
  # from config import IDENTITY, TOOLS, MODEL, get_quote
  from anthropic import Anthropic
  from dotenv import load_dotenv

  load_dotenv()


  class ChatBot:
      def __init__(self, session_state):
          self.anthropic = Anthropic()
          self.session_state = session_state

      def generate_message(
          self,
          messages,
          max_tokens,
      ):
          try:
              response = self.anthropic.messages.create(
                  model=MODEL,
                  system=IDENTITY,
                  max_tokens=max_tokens,
                  messages=messages,
                  tools=TOOLS,
              )
              return response
          except Exception as e:
              return {"error": str(e)}

      def process_user_input(self, user_input):
          self.session_state.messages.append({"role": "user", "content": user_input})

          response_message = self.generate_message(
              messages=self.session_state.messages,
              max_tokens=2048,
          )

          if "error" in response_message:
              return f"An error occurred: {response_message['error']}"

          if response_message.content[-1].type == "tool_use":
              tool_use = response_message.content[-1]
              func_name = tool_use.name
              func_params = tool_use.input
              tool_use_id = tool_use.id

              result = self.handle_tool_use(func_name, func_params)
              self.session_state.messages.append(
                  {"role": "assistant", "content": response_message.content}
              )
              self.session_state.messages.append(
                  {
                      "role": "user",
                      "content": [
                          {
                              "type": "tool_result",
                              "tool_use_id": tool_use_id,
                              "content": f"{result}",
                          }
                      ],
                  }
              )

              follow_up_response = self.generate_message(
                  messages=self.session_state.messages,
                  max_tokens=2048,
              )

              if "error" in follow_up_response:
                  return f"An error occurred: {follow_up_response['error']}"

              response_text = next(
                  (block.text for block in follow_up_response.content if block.type == "text"),
                  None,
              )
              if response_text is None:
                  raise Exception("An error occurred: Unexpected response type")
              self.session_state.messages.append(
                  {"role": "assistant", "content": response_text}
              )
              return response_text

          text_block = next(
              (block for block in response_message.content if block.type == "text"), None
          )
          if text_block is not None:
              response_text = text_block.text
              self.session_state.messages.append(
                  {"role": "assistant", "content": response_text}
              )
              return response_text

          raise Exception("An error occurred: Unexpected response type")

      def handle_tool_use(self, func_name, func_params):
          if func_name == "get_quote":
              premium = get_quote(**func_params)
              return f"Quote generated: ${premium:.2f} per month"

          raise Exception("An unexpected tool was used")
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";

  class ChatBot {
    // IDENTITY、MODEL、TOOLS 和 getQuote 对应本指南前面
    // 定义的 config.py 值（以 Python 展示）。
    readonly anthropic = new Anthropic();
    readonly messages: Anthropic.MessageParam[] = [];

    async generateMessage(
      messages: Anthropic.MessageParam[],
      maxTokens: number
    ): Promise<Anthropic.Message> {
      return this.anthropic.messages.create({
        model: MODEL,
        system: IDENTITY,
        max_tokens: maxTokens,
        messages,
        tools: TOOLS
      });
    }

    async processUserInput(userInput: string): Promise<string> {
      this.messages.push({ role: "user", content: userInput });

      const responseMessage = await this.generateMessage(this.messages, 2048);

      const lastBlock = responseMessage.content.at(-1);
      if (lastBlock?.type === "tool_use") {
        const toolResult = this.handleToolUse(lastBlock.name, lastBlock.input);

        this.messages.push({ role: "assistant", content: responseMessage.content });
        this.messages.push({
          role: "user",
          content: [{ type: "tool_result", tool_use_id: lastBlock.id, content: toolResult }]
        });

        const followUpResponse = await this.generateMessage(this.messages, 2048);

        const followUpBlock = followUpResponse.content.find(
          (block): block is Anthropic.TextBlock => block.type === "text"
        );
        if (!followUpBlock) {
          throw new Error("An error occurred: Unexpected response type");
        }
        this.messages.push({ role: "assistant", content: followUpBlock.text });
        return followUpBlock.text;
      }

      const firstBlock = responseMessage.content.find(
        (block): block is Anthropic.TextBlock => block.type === "text"
      );
      if (firstBlock) {
        this.messages.push({ role: "assistant", content: firstBlock.text });
        return firstBlock.text;
      }

      throw new Error("An error occurred: Unexpected response type");
    }

    handleToolUse(toolName: string, toolInput: unknown): string {
      if (toolName === "get_quote") {
        // SDK 将 tool_use.input 类型定义为 unknown；此处将其收窄为 get_quote 模式。
        if (
          toolInput === null ||
          typeof toolInput !== "object" ||
          !("make" in toolInput) || typeof toolInput.make !== "string" ||
          !("model" in toolInput) || typeof toolInput.model !== "string" ||
          !("year" in toolInput) || typeof toolInput.year !== "number" ||
          !("mileage" in toolInput) || typeof toolInput.mileage !== "number" ||
          !("driver_age" in toolInput) || typeof toolInput.driver_age !== "number"
        ) {
          throw new Error("An error occurred: Unexpected tool input");
        }
        const { make, model: vehicleModel, year, mileage, driver_age: driverAge } = toolInput;

        const premium = getQuote(make, vehicleModel, year, mileage, driverAge);
        return `Quote generated: $${premium.toFixed(2)} per month`;
      }

      throw new Error("An unexpected tool was used");
    }
  }
  ```

  ```csharp C#
  using System.Text.Json;
  using Anthropic;
  using Anthropic.Models.Messages;

  // Config.Model、Config.Identity、Config.Tools 和 Config.GetQuote 对应
  // 本指南前面定义的 config.py 值（以 Python 展示）。
  public class ChatBot
  {
      private readonly AnthropicClient _anthropic = new();

      public List<MessageParam> Messages { get; } = [];

      public async Task<Message> GenerateMessage(List<MessageParam> messages, long maxTokens) =>
          await _anthropic.Messages.Create(
              new MessageCreateParams
              {
                  Model = Config.Model,
                  System = Config.Identity,
                  MaxTokens = maxTokens,
                  Messages = messages,
                  Tools = Config.Tools,
              }
          );

      public async Task<string> ProcessUserInput(string userInput)
      {
          Messages.Add(new() { Role = Role.User, Content = userInput });

          var responseMessage = await GenerateMessage(Messages, maxTokens: 2048);

          if (responseMessage.Content[^1].TryPickToolUse(out var toolUse))
          {
              var toolResult = HandleToolUse(toolUse.Name, toolUse.Input);

              Messages.Add(new()
              {
                  Role = Role.Assistant,
                  Content = responseMessage.Content
                      .Select(contentBlock => new ContentBlockParam(contentBlock.Json))
                      .ToList(),
              });
              Messages.Add(new()
              {
                  Role = Role.User,
                  Content = new List<ContentBlockParam>
                  {
                      new ToolResultBlockParam { ToolUseID = toolUse.ID, Content = toolResult },
                  },
              });

              var followUpResponse = await GenerateMessage(Messages, maxTokens: 2048);

              foreach (var block in followUpResponse.Content)
              {
                  if (block.TryPickText(out var followUpText))
                  {
                      Messages.Add(new() { Role = Role.Assistant, Content = followUpText.Text });
                      return followUpText.Text;
                  }
              }

              throw new InvalidOperationException("An error occurred: Unexpected response type");
          }

          foreach (var block in responseMessage.Content)
          {
              if (block.TryPickText(out var textBlock))
              {
                  Messages.Add(new() { Role = Role.Assistant, Content = textBlock.Text });
                  return textBlock.Text;
              }
          }

          throw new InvalidOperationException("An error occurred: Unexpected response type");
      }

      public string HandleToolUse(string funcName, IReadOnlyDictionary<string, JsonElement> funcParams)
      {
          if (funcName == "get_quote")
          {
              var premium = Config.GetQuote(
                  funcParams["make"].GetString()!,
                  funcParams["model"].GetString()!,
                  funcParams["year"].GetInt64(),
                  funcParams["mileage"].GetInt64(),
                  funcParams["driver_age"].GetInt64()
              );
              return $"Quote generated: ${premium:F2} per month";
          }

          throw new ArgumentException("An unexpected tool was used");
      }
  }
  ```

  ```go Go
  import (
  	"context"
  	"encoding/json"
  	"fmt"

  	"github.com/anthropics/anthropic-sdk-go"
  )

  // ChatBot 封装了 Anthropic 客户端和对话历史。它所使用的
  // identity、model、tools 和 getQuote 值对应本指南前文
  // config.py 中的定义（以 Python 展示）。
  type ChatBot struct {
  	client   anthropic.Client
  	messages []anthropic.MessageParam
  }

  func NewChatBot() *ChatBot {
  	return &ChatBot{client: anthropic.NewClient()}
  }

  func (bot *ChatBot) GenerateMessage(ctx context.Context, messages []anthropic.MessageParam, maxTokens int64) (*anthropic.Message, error) {
  	return bot.client.Messages.New(ctx, anthropic.MessageNewParams{
  		Model:     model,
  		System:    []anthropic.TextBlockParam{{Text: identity}},
  		MaxTokens: maxTokens,
  		Messages:  messages,
  		Tools:     tools,
  	})
  }

  func (bot *ChatBot) ProcessUserInput(ctx context.Context, userInput string) (string, error) {
  	bot.messages = append(bot.messages, anthropic.NewUserMessage(anthropic.NewTextBlock(userInput)))

  	response, err := bot.GenerateMessage(ctx, bot.messages, 2048)
  	if err != nil {
  		return "", err
  	}

  	lastBlock := response.Content[len(response.Content)-1]
  	if toolUse, ok := lastBlock.AsAny().(anthropic.ToolUseBlock); ok {
  		result, err := bot.HandleToolUse(toolUse.Name, toolUse.Input)
  		if err != nil {
  			return "", err
  		}

  		bot.messages = append(bot.messages,
  			response.ToParam(),
  			anthropic.NewUserMessage(anthropic.NewToolResultBlock(toolUse.ID, result, false)),
  		)

  		followUp, err := bot.GenerateMessage(ctx, bot.messages, 2048)
  		if err != nil {
  			return "", err
  		}

  		for _, block := range followUp.Content {
  			if textBlock, ok := block.AsAny().(anthropic.TextBlock); ok {
  				bot.messages = append(bot.messages, anthropic.NewAssistantMessage(anthropic.NewTextBlock(textBlock.Text)))
  				return textBlock.Text, nil
  			}
  		}
  		return "", fmt.Errorf("an error occurred: unexpected response type")
  	}

  	for _, block := range response.Content {
  		if textBlock, ok := block.AsAny().(anthropic.TextBlock); ok {
  			bot.messages = append(bot.messages, anthropic.NewAssistantMessage(anthropic.NewTextBlock(textBlock.Text)))
  			return textBlock.Text, nil
  		}
  	}

  	return "", fmt.Errorf("an error occurred: unexpected response type")
  }

  func (bot *ChatBot) HandleToolUse(toolName string, toolInput json.RawMessage) (string, error) {
  	if toolName != "get_quote" {
  		return "", fmt.Errorf("an unexpected tool was used: %s", toolName)
  	}

  	var input struct {
  		Make      string `json:"make"`
  		Model     string `json:"model"`
  		Year      int    `json:"year"`
  		Mileage   int    `json:"mileage"`
  		DriverAge int    `json:"driver_age"`
  	}
  	if err := json.Unmarshal(toolInput, &input); err != nil {
  		return "", err
  	}
  	premium := getQuote(input.Make, input.Model, input.Year, input.Mileage, input.DriverAge)
  	return fmt.Sprintf("Quote generated: $%.2f per month", premium), nil
  }

  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.core.JsonValue;
  import com.anthropic.models.messages.ContentBlock;
  import com.anthropic.models.messages.ContentBlockParam;
  import com.anthropic.models.messages.Message;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.MessageParam;
  import com.anthropic.models.messages.ToolResultBlockParam;
  import com.anthropic.models.messages.ToolUseBlock;

  // IDENTITY、MODEL、TOOLS 和 getQuote 对应本指南前文
  // 定义的 config.py 值（以 Python 展示）。
  class ChatBot {
      final AnthropicClient anthropic;
      final List<MessageParam> messages;

      ChatBot() {
          // 从 ANTHROPIC_API_KEY 环境变量读取 API 密钥
          this.anthropic = AnthropicOkHttpClient.fromEnv();
          this.messages = new ArrayList<>();
      }

      Message generateMessage(List<MessageParam> messages, long maxTokens) {
          return anthropic.messages().create(MessageCreateParams.builder()
                  .model(MODEL)
                  .system(IDENTITY)
                  .maxTokens(maxTokens)
                  .messages(messages)
                  .tools(TOOLS)
                  .build());
      }

      String processUserInput(String userInput) {
          messages.add(MessageParam.builder()
                  .role(MessageParam.Role.USER)
                  .content(userInput)
                  .build());

          Message responseMessage = generateMessage(messages, 2048);

          List<ContentBlock> content = responseMessage.content();
          ContentBlock lastBlock = content.getLast();
          if (lastBlock.isToolUse()) {
              ToolUseBlock toolUse = lastBlock.asToolUse();
              Map<String, JsonValue> toolInput =
                      (Map<String, JsonValue>) toolUse._input().asObject().orElseThrow();
              String result = handleToolUse(toolUse.name(), toolInput);

              messages.add(MessageParam.builder()
                      .role(MessageParam.Role.ASSISTANT)
                      .contentOfBlockParams(content.stream().map(ContentBlock::toParam).toList())
                      .build());
              messages.add(MessageParam.builder()
                      .role(MessageParam.Role.USER)
                      .contentOfBlockParams(List.of(ContentBlockParam.ofToolResult(
                              ToolResultBlockParam.builder()
                                      .toolUseId(toolUse.id())
                                      .content(result)
                                      .build())))
                      .build());

              Message followUpResponse = generateMessage(messages, 2048);

              ContentBlock followUpBlock = followUpResponse.content().stream()
                      .filter(ContentBlock::isText)
                      .findFirst()
                      .orElseThrow(() -> new IllegalStateException("An error occurred: Unexpected response type"));
              String responseText = followUpBlock.asText().text();
              messages.add(MessageParam.builder()
                      .role(MessageParam.Role.ASSISTANT)
                      .content(responseText)
                      .build());
              return responseText;
          } else {
              ContentBlock textBlock = content.stream()
                      .filter(ContentBlock::isText)
                      .findFirst()
                      .orElseThrow(() -> new IllegalStateException("An error occurred: Unexpected response type"));
              String responseText = textBlock.asText().text();
              messages.add(MessageParam.builder()
                      .role(MessageParam.Role.ASSISTANT)
                      .content(responseText)
                      .build());
              return responseText;
          }
      }

      String handleToolUse(String funcName, Map<String, JsonValue> funcParams) {
          return switch (funcName) {
              case "get_quote" -> {
                  double premium = getQuote(
                          funcParams.get("make").asStringOrThrow(),
                          funcParams.get("model").asStringOrThrow(),
                          ((Number) funcParams.get("year").asNumber().orElseThrow()).longValue(),
                          ((Number) funcParams.get("mileage").asNumber().orElseThrow()).longValue(),
                          ((Number) funcParams.get("driver_age").asNumber().orElseThrow()).longValue());
                  yield "Quote generated: $%.2f per month".formatted(premium);
              }
              default -> throw new IllegalArgumentException("An unexpected tool was used");
          };
      }
  }
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Messages\Message;
  use Anthropic\Messages\MessageParam;
  use Anthropic\Messages\TextBlock;
  use Anthropic\Messages\ToolResultBlockParam;
  use Anthropic\Messages\ToolUseBlock;

  class ChatBot
  {
      // MODEL、IDENTITY、TOOLS 和 get_quote() 对应本指南前文
      // 定义的 config.py 值（以 Python 展示）。

      /** @var list<MessageParam> */
      public private(set) array $messages = [];

      public function __construct(
          private readonly Client $anthropic = new Client(),
      ) {}

      /**
       * @param list<MessageParam> $messages
       */
      public function generateMessage(array $messages, int $maxTokens): Message
      {
          return $this->anthropic->messages->create(
              model: MODEL,
              system: IDENTITY,
              maxTokens: $maxTokens,
              messages: $messages,
              tools: TOOLS,
          );
      }

      public function processUserInput(string $userInput): string
      {
          $this->messages[] = MessageParam::with(role: 'user', content: $userInput);

          $responseMessage = $this->generateMessage($this->messages, maxTokens: 2048);

          $content = $responseMessage->content;
          $lastBlock = array_last($content);

          if ($lastBlock instanceof ToolUseBlock) {
              $toolResult = $this->handleToolUse($lastBlock->name, $lastBlock->input);

              $this->messages[] = MessageParam::with(role: 'assistant', content: $content);
              $this->messages[] = MessageParam::with(
                  role: 'user',
                  content: [
                      ToolResultBlockParam::with(toolUseID: $lastBlock->id, content: $toolResult),
                  ],
              );

              $followUpResponse = $this->generateMessage($this->messages, maxTokens: 2048);

              $firstBlock = array_find(
                  $followUpResponse->content,
                  static fn ($block): bool => $block instanceof TextBlock,
              );
              if (!$firstBlock instanceof TextBlock) {
                  throw new RuntimeException('An error occurred: Unexpected response type');
              }

              $this->messages[] = MessageParam::with(role: 'assistant', content: $firstBlock->text);

              return $firstBlock->text;
          }

          $firstBlock = array_find($content, static fn ($block): bool => $block instanceof TextBlock);
          if ($firstBlock instanceof TextBlock) {
              $this->messages[] = MessageParam::with(role: 'assistant', content: $firstBlock->text);

              return $firstBlock->text;
          }

          throw new RuntimeException('An error occurred: Unexpected response type');
      }

      /**
       * @param array<string, mixed> $funcParams
       */
      private function handleToolUse(string $funcName, array $funcParams): string
      {
          if ($funcName === 'get_quote') {
              $premium = get_quote(...$funcParams);

              return sprintf('Quote generated: $%.2f per month', $premium);
          }

          throw new RuntimeException('An unexpected tool was used');
      }
  }
  ```

  ```ruby Ruby
  # IDENTITY、MODEL、TOOLS 和 get_quote 对应本指南前文
  # 定义的 config.py 值（以 Python 展示）。
  require "anthropic"

  class ChatBot
    attr_reader :messages

    def initialize
      @anthropic = Anthropic::Client.new
      @messages = []
    end

    def generate_message(messages, max_tokens)
      @anthropic.messages.create(
        model: MODEL,
        system_: IDENTITY,
        max_tokens:,
        messages:,
        tools: TOOLS
      )
    end

    def process_user_input(user_input)
      @messages << {role: "user", content: user_input}

      response_message = generate_message(@messages, 2048)

      case response_message.content
      in [*, Anthropic::ToolUseBlock => tool_use]
        result = handle_tool_use(tool_use.name, tool_use.input)
        @messages << {role: "assistant", content: response_message.content}
        @messages << {
          role: "user",
          content: [{type: "tool_result", tool_use_id: tool_use.id, content: result}]
        }

        follow_up_response = generate_message(@messages, 2048)

        case follow_up_response.content
        in [*, Anthropic::TextBlock => text_block, *]
          @messages << {role: "assistant", content: text_block.text}
          text_block.text
        else
          raise "An error occurred: Unexpected response type"
        end
      in [*, Anthropic::TextBlock => text_block, *]
        @messages << {role: "assistant", content: text_block.text}
        text_block.text
      else
        raise "An error occurred: Unexpected response type"
      end
    end

    def handle_tool_use(tool_name, tool_input)
      raise "An unexpected tool was used" unless tool_name == "get_quote"

      premium = get_quote(**tool_input)
      format("Quote generated: $%.2f per month", premium)
    end
  end
  ```
</CodeGroup>

### 构建您的用户界面

使用 main 方法通过 Streamlit 测试部署此代码。这个 `main()` 函数设置了一个基于 Streamlit 的聊天界面。Streamlit 是一个 Python 框架，因此本演练的这一部分仅以 Python 展示；上面的 ChatBot 类是您可以移植到任何语言的部分。

在名为 `app.py` 的文件中执行此操作

```python
import streamlit as st
from chatbot import ChatBot
from config import TASK_SPECIFIC_INSTRUCTIONS


def main():
    st.title("Chat with Eva, Acme Insurance Company's Assistant🤖")

    if "messages" not in st.session_state:
        st.session_state.messages = [
            {"role": "user", "content": TASK_SPECIFIC_INSTRUCTIONS},
            {"role": "assistant", "content": "Understood"},
        ]

    chatbot = ChatBot(st.session_state)

    # 显示用户和助手消息，跳过前两条
    for message in st.session_state.messages[2:]:
        # 忽略工具使用块
        if isinstance(message["content"], str):
            with st.chat_message(message["role"]):
                st.markdown(message["content"])

    if user_msg := st.chat_input("Type your message here..."):
        st.chat_message("user").markdown(user_msg)

        with st.chat_message("assistant"):
            with st.spinner("Eva is thinking..."):
                response_placeholder = st.empty()
                full_response = chatbot.process_user_input(user_msg)
                response_placeholder.markdown(full_response)


if __name__ == "__main__":
    main()
```

使用以下命令运行程序：

```bash
streamlit run app.py
```

### 评估您的提示

提示通常需要测试和优化才能达到生产就绪状态。要确定您的解决方案是否就绪，请使用结合定量和定性方法的系统化流程来评估聊天机器人的性能。基于您定义的成功标准创建[强有力的实证评估](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests#build-evaluations)，将使您能够优化您的提示。

### 提升性能

在复杂场景中，除了标准的[提示工程技术](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/overview)和[护栏实施策略](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)之外，考虑其他策略来提升性能可能会有所帮助。以下是一些常见场景：

#### 使用 RAG 降低长上下文延迟

在处理大量静态和动态上下文时，将所有信息包含在提示中可能会导致高成本、较慢的响应时间以及触及"context window"（上下文窗口）限制。在这种场景下，实施"Retrieval Augmented Generation"（检索增强生成），即 RAG 技术可以提升性能和效率。

通过使用[像 Voyage 这样的嵌入模型](https://platform.claude.com/docs/zh-CN/build-with-claude/embeddings)将信息转换为向量表示，您可以创建一个更具可扩展性和响应性的系统。这种方法允许根据当前查询动态检索相关信息，而不是在每个提示中包含所有可能的上下文。

事实证明，在具有大量上下文需求的系统中，为支持用例实施 RAG 可以提高准确性、缩短响应时间并降低 API 成本。请参阅 [RAG 示例](https://platform.claude.com/cookbook/capabilities-retrieval-augmented-generation-guide)了解完整的实例。

#### 通过工具使用集成实时数据

在处理需要实时信息的查询（例如账户余额或保单详情）时，基于嵌入的 RAG 方法是不够的。相反，工具使用可以增强您的聊天机器人提供准确、实时回复的能力。例如，您可以通过工具使用来查找客户信息、检索订单详情以及代表客户取消订单。

这种方法在[工具使用：客户服务智能体示例](https://platform.claude.com/cookbook/tool-use-customer-service-agent)中有所概述，它让您能够将实时数据集成到 Claude 的回复中，并提供更加个性化和高效的客户体验。

#### 加强输入和输出护栏

在部署聊天机器人时，尤其是在客户服务场景中，防范与滥用、超出范围的查询和不当回复相关的风险非常重要。虽然 Claude 本身对此类场景具有韧性，但以下是加强聊天机器人护栏的额外步骤：

* [减少幻觉](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)：实施事实核查机制和[引用](https://platform.claude.com/cookbook/misc-using-citations)，使回复以所提供的信息为依据。
* 交叉核对信息：验证智能体的回复是否与您公司的政策和已知事实一致。
* 避免合同承诺：确保智能体不会作出其无权作出的承诺或签订协议。
* [缓解越狱](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks)：使用无害性筛查和输入验证等方法，防止用户利用模型漏洞以生成不当内容。
* 避免提及竞争对手：实施竞争对手提及过滤器，以保持品牌聚焦，不提及任何竞争对手的产品或服务。
* [提高输出一致性](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/increase-consistency)：防止 Claude 改变风格或偏离角色，即使在漫长、复杂的交互中也是如此。
* 移除个人身份信息（PII）：除非明确要求并获得授权，否则从回复中剔除任何 PII。

#### 通过流式传输缩短感知响应时间

在处理可能较长的回复时，实施"streaming"（流式传输）可以提高用户参与度和满意度。在这种场景下，用户会逐步收到答案，而不是等待整个回复生成完毕。

以下是实施流式传输的方法：

1. 使用 [Anthropic 流式传输 API](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming) 来支持流式响应。
2. 设置您的前端以处理传入的文本块。
3. 在每个文本块到达时显示它，模拟实时打字。
4. 实施一种保存完整回复的机制，允许用户在离开后返回时查看。

在某些情况下，流式传输使得可以使用基础延迟较高的更高级模型，因为逐步显示减轻了较长处理时间的影响。

#### 扩展您的聊天机器人

随着聊天机器人复杂性的增长，您的应用程序架构可以随之演进。在向架构添加更多层之前，请考虑以下不那么繁复的选项：

* 确保您充分利用了您的提示，并通过提示工程进行优化。使用[提示工程指南](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/overview)编写最有效的提示。
* 向提示添加额外的[工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)（可以包括[提示链](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#chain-complex-prompts)），看看是否能实现所需的功能。

如果您的聊天机器人处理的任务极其多样，您可能需要考虑添加一个[单独的意图分类器](https://platform.claude.com/cookbook/capabilities-classification-guide)来路由初始客户查询。对于现有应用程序，这将涉及创建一个决策树，将客户查询通过分类器路由到专门的对话（具有各自的工具集和系统提示）。请注意，此方法需要额外调用一次 Claude，这可能会增加延迟。

### 将 Claude 集成到您的支持工作流程中

虽然这些示例侧重于可在 Streamlit 环境中调用的 Python 函数，但部署 Claude 用于实时支持聊天机器人需要一个 API 服务。

以下是您可以采取的方法：

1. 创建 API 封装器：围绕您的分类函数开发一个简单的 API 封装器。例如，您可以使用 Flask API 或 Fast API 将您的代码封装为 HTTP 服务。您的 HTTP 服务可以接受用户输入并完整返回 Assistant 回复。因此，您的服务可以具有以下特性：

   * 服务器发送事件（SSE）：SSE 允许将响应从服务器实时流式传输到客户端。这在使用 LLM 时提供了流畅的交互体验。
   * 缓存：实施缓存可以缩短响应时间并减少不必要的 API 调用。
   * 上下文保留：在用户离开后返回时保持上下文，对于对话的连续性非常重要。

2. 构建 Web 界面：实现一个用户友好的 Web UI，用于与由 Claude 驱动的智能体进行交互。

## 后续步骤

<CardGroup cols={2}>
  <Card title="工具使用" icon="wrench" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview">
    让 Claude 访问您的 API，以便它能够代表客户采取行动。
  </Card>

  <Card title="开发测试" icon="check" href="https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests">
    构建评估，以根据您定义的成功标准衡量您的支持智能体。
  </Card>

  <Card title="流式传输" icon="bolt" href="https://platform.claude.com/docs/zh-CN/build-with-claude/streaming">
    流式传输回复，让客户在答案生成时即可看到。
  </Card>

  <Card title="提示工程" icon="lightbulb" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/overview">
    完善您的系统提示和示例，以获得更好的任务表现。
  </Card>
</CardGroup>
