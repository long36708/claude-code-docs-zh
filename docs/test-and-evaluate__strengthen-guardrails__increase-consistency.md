---
title: 提高输出一致性
url: https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/increase-consistency
description: 通过指定精确格式、预填充响应、使用示例进行约束以及基于检索来锚定答案，使 Claude 的输出更加一致。
---

<Tip>
  **如需保证 JSON 模式一致性**

  如果您需要 Claude 始终输出符合特定模式的有效 JSON，请使用[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)，而不是下面的提示工程技术。结构化输出可保证模式合规性，并且专为此用例而设计。

  当您需要一般性的输出一致性，或需要超出严格 JSON 模式的灵活性时，下面的技术会很有用。
</Tip>

以下是使 Claude 的响应更加一致的方法：

## 指定所需的输出格式

使用 JSON、XML 或自定义模板精确定义您所需的输出格式，以便 Claude 遵循您要求的每个输出格式元素。

<Accordion title="示例：标准化客户反馈">
  ```text User wrap
  You're a Customer Insights AI. Analyze this feedback and output in JSON format with keys: "sentiment" (positive/negative/neutral), "key_issues" (list), and "action_items" (list of dicts with "team" and "task").

  "I've been a loyal user for 3 years, but the recent UI update is a disaster. Finding basic features is now a scavenger hunt. Plus, the new 'premium' pricing is outrageous. I'm considering switching unless this is fixed ASAP."
  ```

  ```json Assistant
  {
    "sentiment": "negative",
    "key_issues": [
      "Poor UI/UX in recent update",
      "Difficulty finding basic features",
      "Perceived high pricing (new 'premium' tier)"
    ],
    "action_items": [
      {
        "team": "Product",
        "task": "Conduct usability testing and iterate on UI based on findings"
      },
      {
        "team": "UX",
        "task": "Create a feature discovery guide or onboarding flow"
      },
      {
        "team": "Marketing",
        "task": "Review pricing strategy and communicate value proposition"
      },
      {
        "team": "Customer Success",
        "task": "Reach out to user, offer concessions to prevent churn"
      }
    ]
  }
  ```
</Accordion>

## 预填充 Claude 的响应

<Note>
  Claude 4.6 及更高版本的模型以及 

  [Claude Mythos Preview](https://anthropic.com/glasswing)

   不支持预填充。请在支持的模型上使用

  [结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)

  ，或改用系统提示指令。
</Note>

使用您所需的格式预填充 `Assistant` 回合。此技巧可绕过 Claude 友好的开场白并强制执行您的结构。

<Accordion title="示例：每日销售报告">
  ```text User wrap
  You're an insightful Sales Intelligence AI. Generate today's sales report.

  Structure the report like this:

  <report>
      <summary>
          <metric name="total_revenue">$0.00</metric>
          <metric name="units_sold">0</metric>
          <metric name="avg_order_value">$0.00</metric>
      </summary>
      <top_products>
          <product>
              <name>Product Name</name>
              <revenue>$0.00</revenue>
              <units>0</units>
          </product>
          ...
      </top_products>
      <regional_performance>
          <region>
              <name>Region Name</name>
              <revenue>$0.00</revenue>
              <growth>0.0%</growth>
          </region>
          ...
      </regional_performance>
      <action_items>
          <item>Action item.</item>
          ...
      </action_items>
  </report>
  ```

  ```xml Assistant (prefill)
  <report>
      <summary>
          <metric name=
  ```

  ```xml Assistant
  "total_revenue">$842,567.00</metric>
          <metric name="units_sold">15,238</metric>
          <metric name="avg_order_value">$55.29</metric>
      </summary>
      <top_products>
          <product>
              <name>AcmeGuard Pro</name>
              <revenue>$325,890.00</revenue>
              <units>3,621</units>
          </product>
          <product>
              <name>AcmeVault Enterprise</name>
              <revenue>$254,400.00</revenue>
              <units>1,272</units>
          </product>
          <product>
              <name>AcmeBoost XL</name>
              <revenue>$147,690.00</revenue>
              <units>4,923</units>
          </product>
      </top_products>
      <regional_performance>
          <region>
              <name>APAC</name>
              <revenue>$258,960.00</revenue>
              <growth>12.8%</growth>
          </region>
          <region>
              <name>EMEA</name>
              <revenue>$325,478.00</revenue>
              <growth>3.2%</growth>
          </region>
          <region>
              <name>Americas</name>
              <revenue>$258,129.00</revenue>
              <growth>-2.1%</growth>
          </region>
      </regional_performance>
      <action_items>
          <item>Investigate Americas revenue drop; schedule regional sales team meeting.</item>
          <item>Accelerate CloudGuard Pro rollout in APAC to capitalize on growth.</item>
          <item>Review NetBoost XL pricing; high volume but lower revenue.</item>
      </action_items>
  </report>
  ```
</Accordion>

## 使用示例进行约束

提供您所需输出的示例。这比抽象的指令更有效。

<Accordion title="示例：生成一致的市场情报">
  ```text User wrap
  As a Market Intelligence AI, your task is to analyze data about our competitors. Here is our competitor data:
  <data>
  {{DATA}}
  </data>

  Output following this example format:

  <competitor>
    <name>Rival Inc</name>
    <overview>A 50-word summary.</overview>
    <swot>
      <strengths>- Bullet points</strengths>
      <weaknesses>- Bullet points</weaknesses>
      <opportunities>- Bullet points</opportunities>
      <threats>- Bullet points</threats>
    </swot>
    <strategy>A 30-word strategic response.</strategy>
  </competitor>

  Now, analyze AcmeGiant and AcmeDataCo using this format.
  ```

  ```text Assistant wrap
  <competitor>
    <name>AcmeGiant</name>
    <overview>Global tech leader with a vast ecosystem of consumer and enterprise products. Known for innovation, strong brand, and deep customer lock-in through integrated hardware, software, and services.</overview>
    <swot>
      <strengths>
        - Massive R&D budget fuels continuous innovation
        - Loyal customer base and powerful brand recognition
        - Synergistic product ecosystem increases switching costs
      </strengths>
      <weaknesses>
        - High prices may deter cost-sensitive customers
        - Slow to address enterprise-specific customization needs
        - Privacy concerns and regulatory scrutiny in multiple markets
      </weaknesses>
      <opportunities>
        - Growing demand for AI and ML capabilities
        - Expansion into emerging markets
        - Acquisitions to fill product or talent gaps
      </opportunities>
      <threats>
        - Rapid rise of cloud-native competitors
        - Antitrust lawsuits and regulatory actions
        - Talent poaching by startups and competitors
      </threats>
    </swot>
    <strategy>Focus on our agility and customization. Target TechGiant's slow enterprise response and privacy issues in our marketing.</strategy>
  </competitor>

  <competitor>
    <name>AcmeDataCo</name>
    <overview>Rising star in data analytics and visualization. Offers intuitive, self-service BI tools with a freemium model, appealing to startups and enterprise teams alike.</overview>
    <swot>
      <strengths>
        - User-friendly interface reduces need for specialized training
        - Freemium model lowers barrier to entry, drives viral growth
        - Strong product-market fit with rise of data-driven decisions
      </strengths>
      <weaknesses>
        - Limited advanced analytics compared to established BI tools
        - Scaling issues reported with very large datasets
        - Dependent on third-party cloud infrastructure (AWS)
      </weaknesses>
      <opportunities>
        - Expanding into predictive analytics and AI-driven insights
        - Forming strategic alliances with complementary tech vendors
        - Customizable dashboards for industry-specific solutions
      </opportunities>
      <threats>
        - Market saturation; many new entrants in self-service BI
        - Enterprise BI giants developing more user-friendly tools
        - Data privacy regulations increasing compliance complexity
      </threats>
    </swot>
    <strategy>Highlight our robust enterprise features and data governance. Target accounts struggling with DataCo's scale limitations.</strategy>
  </competitor>
  ```
</Accordion>

## 使用检索实现上下文一致性

对于需要一致上下文的任务（例如聊天机器人、知识库），使用检索将 Claude 的响应锚定在固定的信息集中。

<Accordion title="示例：增强 IT 支持的一致性">
  ```text User wrap
  You're our IT Support AI that draws on knowledge base data. Here are entries from your knowledge base:

  <kb>
    <entry>
      <id>1</id>
      <title>Reset Active Directory password</title>
      <content>1. Go to password.ourcompany.com
  2. Enter your username
  3. Click "Forgot Password"
  4. Follow email instructions</content>
    </entry>
    <entry>
      <id>2</id>
      <title>Connect to VPN</title>
      <content>1. Install GlobalProtect from software center
  2. Open GlobalProtect, enter "vpn.ourcompany.com"
  3. Use AD credentials to log in</content>
    </entry>
  </kb>

  When helping users, always check the knowledge base first. Respond in this format:

  <response>
    <kb_entry>Knowledge base entry used</kb_entry>
    <answer>Your response</answer>
  </response> Write some test questions for yourself and answer them using the knowledge base, just to make sure you understand how to use the knowledge base properly.
  ```

  ```text Assistant wrap
  Understood. I'll use the knowledge base for consistent responses. Let's test with a couple of scenarios.

  User: Hey, I can't log into my email. I think I forgot my password.

  <response>
    <kb_entry>1: Reset Active Directory password</kb_entry>
    <answer>I understand you're having trouble logging into your email due to a forgotten password. No worries, this is a common issue. To reset your Active Directory password, which is used for email access, follow these steps:

  1. Go to password.ourcompany.com
  2. Enter your username (same as your email address)
  3. Click on the "Forgot Password" link
  4. You'll receive an email with instructions. Follow them to reset your password.

  After resetting, you should be able to log into your email. Let me know if you need any further assistance.</answer>
  </response>

  User: I'm working from home today. How do I access internal resources?

  <response>
    <kb_entry>2: Connect to VPN</kb_entry>
    <answer>Great question! To access internal resources while working from home, you'll need to connect to our company VPN (Virtual Private Network). Here's how to set that up:

  1. First, install the GlobalProtect VPN client. You can find this in our software center.
  2. Once installed, open the GlobalProtect application.
  3. In the server field, enter "vpn.ourcompany.com".
  4. Use your Active Directory (AD) credentials to log in - the same username and password you use for your email.

  Once connected, you'll have secure access to all internal resources as if you were in the office. Let me know if you run into any issues during setup.</answer>
  </response>
  ```
</Accordion>

## 为复杂任务链接提示

将复杂任务分解为更小、更一致的子任务。每个子任务都能获得 Claude 的充分关注，从而减少大规模工作流中的不一致错误。

## 让 Claude 保持角色

对于基于角色的应用，保持一致的角色需要刻意的提示设计。

* **使用系统提示设置角色：** 使用[系统提示](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/claude-prompting-best-practices#give-claude-a-role)来定义 Claude 的角色和个性。这为一致的响应奠定了坚实的基础。
  <Tip>
    在设置角色时，提供有关个性、背景以及任何特定特征或癖好的详细信息。这有助于模型更好地模拟和泛化角色的特征。
  </Tip>
* **让 Claude 为可能的场景做好准备：** 在您的提示中提供常见场景和预期响应的列表。这可以"训练" Claude 在不脱离角色的情况下处理各种情况。

<Accordion title="示例：用于角色提示的企业聊天机器人">
  ```text System wrap
  You are AcmeBot, the enterprise-grade AI assistant for AcmeTechCo. Your role:
      - Analyze technical documents (TDDs, PRDs, RFCs)
      - Provide actionable insights for engineering, product, and ops teams
      - Maintain a professional, concise tone
  ```

  ```text User wrap
  Here is the user query for you to respond to:
  <user_query>
  {{USER_QUERY}}
  </user_query>

  Your rules for interaction are:
      - Always reference AcmeTechCo standards or industry best practices
      - If unsure, ask for clarification before proceeding
      - Never disclose confidential AcmeTechCo information.

  As AcmeBot, you should handle situations along these guidelines:
      - If asked about AcmeTechCo IP: "I cannot disclose TechCo's proprietary information."
      - If questioned on best practices: "Per ISO/IEC 25010, we prioritize..."
      - If unclear on a doc: "To ensure accuracy, please clarify section 3.2..."
  ```
</Accordion>
