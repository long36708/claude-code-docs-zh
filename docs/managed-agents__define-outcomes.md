---
title: 定义结果
url: https://platform.claude.com/docs/zh-CN/managed-agents/define-outcomes
description: 告诉智能体'完成'是什么样子，并让它迭代直到达成目标。
featureMetadata:
  topic:
    title: Managed Agents
    url: https://platform.claude.com/docs/en/managed-agents/overview
  status: beta
  betaHeader: managed-agents-2026-04-01
---

"outcome"（结果）告诉会话最终结果应该是什么样子，以及如何衡量其质量。智能体朝着该目标工作，不断进行自我评估和迭代，直到达成结果。

当您定义结果时，"harness"（运行框架）会自动配置一个 *grader*（评分器），根据 "rubric"（评分标准）评估产出物。评分器使用单独的 "context window"（上下文窗口），以避免受到主智能体实现选择的影响。

评分器会返回一段说明，总结哪些标准通过或未通过，或确认产出物满足评分标准。该反馈会交回给智能体，用于下一次迭代。

## 创建评分标准

评分标准是一个描述各项标准评分方式的 markdown 文档。评分标准是必需的。

<Accordion title="编写有效评分标准的技巧">
  将评分标准组织为明确、可评分的标准，例如"CSV 包含一个带有数值的 price 列"，而不是"数据看起来不错"。评分器会独立地为每项标准评分，因此模糊的标准会导致评估结果不稳定。

  如果您手头没有评分标准，可以尝试向 Claude 提供一个已知优质的产出物示例，让它分析该内容好在哪里，然后将该分析转化为评分标准。这种折中方法通常比从零开始编写标准效果更好。
</Accordion>

评分标准示例：

```markdown
# DCF Model Rubric

## Revenue Projections
- Uses historical revenue data from the last 5 fiscal years
- Projects revenue for at least 5 years forward
- Growth rate assumptions are explicitly stated and reasonable

## Cost Structure
- COGS and operating expenses are modeled separately
- Margins are consistent with historical trends or deviations are justified

## Discount Rate
- WACC is calculated with stated assumptions for cost of equity and cost of debt
- Beta, risk-free rate, and equity risk premium are sourced or justified

## Terminal Value
- Uses either perpetuity growth or exit multiple method (stated which)
- Terminal growth rate does not exceed long-term GDP growth

## Output Quality
- All figures are in a single .xlsx file with clearly labeled sheets
- Key assumptions are on a separate "Assumptions" sheet
- Sensitivity analysis on WACC and terminal growth rate is included
```

您可以在 `user.define_outcome` 上以内联文本形式传递评分标准（请参阅[创建带有结果的会话](https://platform.claude.com/docs/zh-CN/managed-agents/define-outcomes#create-a-session-with-an-outcome)），也可以通过 Files API 上传，以便在多个会话中复用。

<CodeGroup>
  ```bash cURL
  curl -fsSL https://api.anthropic.com/v1/files \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -F file=@/tmp/rubric.md
  ```

  ```bash CLI
  ant files upload --file /tmp/rubric.md
  ```

  ```python Python
  import time
  from pathlib import Path

  from anthropic import Anthropic

  client = Anthropic()

  RUBRIC = """# DCF Model Rubric

  ## Revenue Projections
  - Uses historical revenue data from the last 5 fiscal years
  - Projects revenue for at least 5 years forward

  ## Output Quality
  - All figures are in a single .xlsx file with clearly labeled sheets
  """
  Path("/tmp/rubric.md").write_text(RUBRIC)

  rubric = client.files.upload(file=Path("/tmp/rubric.md"))
  print(f"Uploaded rubric: {rubric.id}")
  ```

  ```typescript TypeScript
  import { writeFile, readFile } from "node:fs/promises";

  import Anthropic from "@anthropic-ai/sdk";
  import { toFile } from "@anthropic-ai/sdk";

  const client = new Anthropic();

  const RUBRIC = `# DCF Model Rubric

  ## Revenue Projections
  - Uses historical revenue data from the last 5 fiscal years
  - Projects revenue for at least 5 years forward

  ## Output Quality
  - All figures are in a single .xlsx file with clearly labeled sheets
  `;
  await writeFile("/tmp/rubric.md", RUBRIC);

  const rubric = await client.files.upload({
    file: await toFile(readFile("/tmp/rubric.md"), "/tmp/rubric.md"),
  });
  console.log(`Uploaded rubric: ${rubric.id}`);
  ```

  ```csharp C#
  using Anthropic;
  using Anthropic.Models.Beta.Agents;
  using Anthropic.Models.Beta.Environments;
  using Anthropic.Models.Beta.Sessions;
  using Anthropic.Models.Beta.Sessions.Events;
  using Anthropic.Models.Files;

  var client = new AnthropicClient();

  const string Rubric = """
      # DCF Model Rubric

      ## Revenue Projections
      - Uses historical revenue data from the last 5 fiscal years
      - Projects revenue for at least 5 years forward

      ## Output Quality
      - All figures are in a single .xlsx file with clearly labeled sheets
      """;
  await File.WriteAllTextAsync("/tmp/rubric.md", Rubric);

  var rubric = await client.Files.Upload(new()
  {
      File = File.OpenRead("/tmp/rubric.md"),
  });
  Console.WriteLine($"Uploaded rubric: {rubric.ID}");
  ```

  ```go Go
  package main

  import (
  	"context"
  	"fmt"
  	"io"
  	"os"
  	"time"

  	"github.com/anthropics/anthropic-sdk-go"
  )

  const rubric = `# DCF Model Rubric

  ## Revenue Projections
  - Uses historical revenue data from the last 5 fiscal years
  - Projects revenue for at least 5 years forward

  ## Output Quality
  - All figures are in a single .xlsx file with clearly labeled sheets
  `

  func main() {
  	ctx := context.Background()
  	client := anthropic.NewClient()

  	if err := os.WriteFile("/tmp/rubric.md", []byte(rubric), 0o644); err != nil {
  		panic(err)
  	}

  	f, err := os.Open("/tmp/rubric.md")
  	if err != nil {
  		panic(err)
  	}

  	uploaded, err := client.Files.Upload(ctx, anthropic.FileUploadParams{
  		File: anthropic.File(f, "rubric.md", "text/markdown"),
  	})
  	if err != nil {
  		panic(err)
  	}
  	fmt.Printf("Uploaded rubric: %s\n", uploaded.ID)
  ```

  ```java Java
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.core.http.HttpResponse;
  import com.anthropic.models.beta.AnthropicBeta;
  import com.anthropic.models.beta.agents.AgentCreateParams;
  import com.anthropic.models.beta.agents.BetaManagedAgentsAgentToolset20260401Params;
  import com.anthropic.models.beta.agents.BetaManagedAgentsModel;
  import com.anthropic.models.beta.environments.BetaCloudConfigParams;
  import com.anthropic.models.beta.environments.EnvironmentCreateParams;
  import com.anthropic.models.beta.files.FileListParams;
  import com.anthropic.models.beta.sessions.SessionCreateParams;
  import com.anthropic.models.beta.sessions.events.BetaManagedAgentsTextRubricParams;
  import com.anthropic.models.beta.sessions.events.BetaManagedAgentsUserDefineOutcomeEventParams;
  import com.anthropic.models.beta.sessions.events.BetaManagedAgentsUserInterruptEventParams;
  import com.anthropic.models.beta.sessions.events.EventSendParams;
  import com.anthropic.models.files.FileUploadParams;

  import java.io.InputStream;
  import java.nio.file.Files;
  import java.nio.file.Path;
  import java.nio.file.StandardCopyOption;

  void main() throws Exception {
      var client = AnthropicOkHttpClient.fromEnv();

      var RUBRIC = """
          # DCF Model Rubric

          ## Revenue Projections
          - Uses historical revenue data from the last 5 fiscal years
          - Projects revenue for at least 5 years forward

          ## Output Quality
          - All figures are in a single .xlsx file with clearly labeled sheets
          """;
      Files.writeString(Path.of("/tmp/rubric.md"), RUBRIC);

      var rubric = client.files().upload(
          FileUploadParams.builder()
              .file(Path.of("/tmp/rubric.md"))
              .build());
      IO.println("Uploaded rubric: " + rubric.id());
  ```

  ```php PHP
  use Anthropic\Client;
  use Anthropic\Core\FileParam;

  $client = new Client();

  $rubricText = <<<'MD'
  # DCF Model Rubric

  ## Revenue Projections
  - Uses historical revenue data from the last 5 fiscal years
  - Projects revenue for at least 5 years forward

  ## Output Quality
  - All figures are in a single .xlsx file with clearly labeled sheets
  MD;
  file_put_contents('/tmp/rubric.md', $rubricText);

  $rubric = $client->files->upload(
      file: FileParam::fromResource(fopen('/tmp/rubric.md', 'r'), contentType: 'text/markdown'),
  );
  echo "Uploaded rubric: {$rubric->id}\n";
  ```

  ```ruby Ruby
  require "anthropic"
  require "pathname"

  client = Anthropic::Client.new

  RUBRIC = <<~MD
    # DCF Model Rubric

    ## Revenue Projections
    - Uses historical revenue data from the last 5 fiscal years
    - Projects revenue for at least 5 years forward

    ## Output Quality
    - All figures are in a single .xlsx file with clearly labeled sheets
  MD
  File.write("/tmp/rubric.md", RUBRIC)

  rubric = client.files.upload(file: Pathname.new("/tmp/rubric.md"))
  puts "Uploaded rubric: #{rubric.id}"
  ```
</CodeGroup>

## 创建带有结果的会话

以下示例为现有的[智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)和[环境](https://platform.claude.com/docs/zh-CN/managed-agents/environments)（两者均单独创建）创建一个[会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)，然后发送一个 `user.define_outcome` 事件。智能体会立即开始工作，无需额外发送用户消息事件。

<CodeGroup>
  ```bash cURL
  # 创建会话
  session=$(curl -fsSL https://api.anthropic.com/v1/sessions \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    --json @- <<EOF
  {
    "agent": "$AGENT_ID",
    "environment_id": "$ENVIRONMENT_ID",
    "title": "Financial analysis on Costco"
  }
  EOF
  )
  SESSION_ID=$(jq -r '.id' <<<"$session")

  # 定义结果 — 代理在收到后即开始工作
  curl -fsSL "https://api.anthropic.com/v1/sessions/$SESSION_ID/events" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    --json @- >/dev/null <<EOF
  {
    "events": [
      {
        "type": "user.define_outcome",
        "description": "Build a DCF model for Costco in .xlsx",
        "rubric": {"type": "text", "content": "# DCF Model Rubric\n..."},
        "max_iterations": 5
      }
    ]
  }
  EOF
  # 或："rubric": {"type": "file", "file_id": "$RUBRIC_ID"}
  # "max_iterations" 为可选项；默认 3，最大 20
  ```

  ```bash CLI
  # 创建会话
  SESSION_ID=$(ant beta:sessions create \
    --agent "$AGENT_ID" \
    --environment-id "$ENVIRONMENT_ID" \
    --title "Financial analysis on Costco" \
    --transform id --raw-output)

  # 定义结果 — 代理在收到后即开始工作
  ant beta:sessions:events send --session-id "$SESSION_ID" <<YAML
  events:
    - type: user.define_outcome
      description: Build a DCF model for Costco in .xlsx
      rubric: {type: file, file_id: $RUBRIC_ID}
      # or: rubric: {type: text, content: "..."}
      max_iterations: 5  # optional; default 3, max 20
  YAML
  ```

  ```python Python
  # 创建会话
  session = client.beta.sessions.create(
      agent=agent.id,
      environment_id=environment.id,
      title="Financial analysis on Costco",
  )

  # 定义结果 — 代理收到后即开始工作
  client.beta.sessions.events.send(
      session_id=session.id,
      events=[
          {
              "type": "user.define_outcome",
              "description": "Build a DCF model for Costco in .xlsx",
              "rubric": {"type": "text", "content": RUBRIC},
              # 或："rubric": {"type": "file", "file_id": rubric.id},
              "max_iterations": 5,  # optional; default 3, max 20
          }
      ],
  )
  ```

  ```typescript TypeScript
  // 创建会话
  const session = await client.beta.sessions.create({
    agent: agent.id,
    environment_id: environment.id,
    title: "Financial analysis on Costco",
  });

  // 定义结果 — 代理收到后即开始工作
  await client.beta.sessions.events.send(session.id, {
    events: [
      {
        type: "user.define_outcome",
        description: "Build a DCF model for Costco in .xlsx",
        rubric: { type: "text", content: RUBRIC },
        // 或：rubric: { type: "file", file_id: rubric.id },
        max_iterations: 5, // optional; default 3, max 20
      },
    ],
  });
  ```

  ```csharp C#
  // 创建会话
  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = agent.ID,
      EnvironmentID = environment.ID,
      Title = "Financial analysis on Costco",
  });

  // 定义结果 — 代理收到后即开始工作
  await client.Beta.Sessions.Events.Send(session.ID, new()
  {
      Events =
      [
          new BetaManagedAgentsUserDefineOutcomeEventParams
          {
              Type = BetaManagedAgentsUserDefineOutcomeEventParamsType.UserDefineOutcome,
              Description = "Build a DCF model for Costco in .xlsx",
              Rubric = new BetaManagedAgentsTextRubricParams
              {
                  Type = BetaManagedAgentsTextRubricParamsType.Text,
                  Content = Rubric,
              },
              // 或：Rubric = new BetaManagedAgentsFileRubricParams
              //     { Type = BetaManagedAgentsFileRubricParamsType.File, FileID = rubric.ID },
              MaxIterations = 5, // optional; default 3, max 20
          },
      ],
  });
  ```

  ```go Go
  // 创建会话
  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent: anthropic.BetaSessionNewParamsAgentUnion{
  		OfString: anthropic.String(agent.ID),
  	},
  	EnvironmentID: environment.ID,
  	Title:         anthropic.String("Financial analysis on Costco"),
  })
  if err != nil {
  	panic(err)
  }

  // 定义结果 — 代理收到后即开始工作
  _, err = client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
  	Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
  		OfUserDefineOutcome: &anthropic.BetaManagedAgentsUserDefineOutcomeEventParams{
  			Type:        anthropic.BetaManagedAgentsUserDefineOutcomeEventParamsTypeUserDefineOutcome,
  			Description: "Build a DCF model for Costco in .xlsx",
  			Rubric: anthropic.BetaManagedAgentsUserDefineOutcomeEventParamsRubricUnion{
  				OfText: &anthropic.BetaManagedAgentsTextRubricParams{
  					Type:    anthropic.BetaManagedAgentsTextRubricParamsTypeText,
  					Content: rubric,
  				},
  			},
  			// 或：OfFile: &anthropic.BetaManagedAgentsFileRubricParams{
  			//     Type: anthropic.BetaManagedAgentsFileRubricParamsTypeFile, FileID: uploaded.ID},
  			MaxIterations: anthropic.Int(5), // optional; default 3, max 20
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  // 创建会话
  var session = client.beta().sessions().create(
      SessionCreateParams.builder()
          .agent(agent.id())
          .environmentId(environment.id())
          .title("Financial analysis on Costco")
          .build());

  // 定义结果 — 代理收到后即开始工作
  client.beta().sessions().events().send(
      session.id(),
      EventSendParams.builder()
          .addEvent(BetaManagedAgentsUserDefineOutcomeEventParams.builder()
              .type(BetaManagedAgentsUserDefineOutcomeEventParams.Type.USER_DEFINE_OUTCOME)
              .description("Build a DCF model for Costco in .xlsx")
              .rubric(BetaManagedAgentsTextRubricParams.builder()
                  .type(BetaManagedAgentsTextRubricParams.Type.TEXT)
                  .content(RUBRIC)
                  .build())
              // 或：.rubric(BetaManagedAgentsFileRubricParams.builder()
              //     .type(BetaManagedAgentsFileRubricParams.Type.FILE).fileId(rubric.id()).build())
              .maxIterations(5) // optional; default 3, max 20
              .build())
          .build());
  ```

  ```php PHP
  // 创建会话
  $session = $client->beta->sessions->create(
      agent: $agent->id,
      environmentID: $environment->id,
      title: 'Financial analysis on Costco',
  );

  // 定义结果 — 代理收到后即开始工作
  $client->beta->sessions->events->send(
      $session->id,
      events: [
          [
              'type' => 'user.define_outcome',
              'description' => 'Build a DCF model for Costco in .xlsx',
              'rubric' => ['type' => 'text', 'content' => $rubricText],
              // 或：'rubric' => ['type' => 'file', 'file_id' => $rubric->id],
              'max_iterations' => 5, // optional; default 3, max 20
          ],
      ],
  );
  ```

  ```ruby Ruby
  # 创建会话
  session = client.beta.sessions.create(
    agent: agent.id,
    environment_id: environment.id,
    title: "Financial analysis on Costco"
  )

  # 定义结果 — 代理收到后即开始工作
  client.beta.sessions.events.send_(
    session.id,
    events: [
      {
        type: "user.define_outcome",
        description: "Build a DCF model for Costco in .xlsx",
        rubric: {type: "text", content: RUBRIC},
        # 或：rubric: {type: "file", file_id: rubric.id},
        max_iterations: 5 # optional; default 3, max 20
      }
    ]
  )
  ```
</CodeGroup>

<Note>
  您也可以直接在创建请求中定义结果：在 [`initial_events`](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#seed-the-session-with-initial-events) 中传递单个 `user.define_outcome` 事件，即可通过一次调用创建会话并开始朝着结果工作。
</Note>

## 结果事件

面向结果的会话的进度会通过事件[流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)呈现。

* `agent.*` 事件（例如消息和工具使用）显示朝着结果推进的进度。
* `span.outcome_evaluation_*` 事件仅在面向结果的会话中发出，显示迭代循环的次数以及评分器的反馈过程。
* 您也可以向面向结果的会话发送 `user.message` [事件](https://platform.claude.com/docs/zh-CN/managed-agents/reference#event-types)，以在智能体工作过程中对其进行引导，但这不是必需的：智能体会自行朝着结果工作，不断迭代，直到成功或用完迭代次数。
* `user.interrupt` 事件会暂停当前结果的工作，并将 `span.outcome_evaluation_end.result` 标记为 `interrupted`，以便您启动新的结果。
* 在最终的结果评估之后，会话可以作为对话式会话继续进行，也可以启动新的结果。会话会保留先前结果的历史记录。

### 定义结果用户事件

<Note>
  一次仅支持一个结果，但您可以按顺序串联多个结果。为此，请在上一个结果的终止 `span.outcome_evaluation_end` 事件之后发送新的 `user.define_outcome` 事件。
</Note>

这是您为启动结果而发送的事件。收到后会回显该事件，其中包含 `processed_at` 时间戳和 `outcome_id`。

```json
{
  "type": "user.define_outcome",
  "description": "Build a DCF model for Costco in .xlsx",
  "rubric": { "type": "file", "file_id": "file_01..." },
  "max_iterations": 5
}
```

### 结果评估开始

当评分器针对一个迭代循环开始评估时发出。`iteration` 字段是一个从 0 开始的修订计数器：`0` 表示第一次评估，`1` 表示第一次修订后的重新评估，依此类推。

```json
{
  "type": "span.outcome_evaluation_start",
  "id": "sevt_01def...",
  "outcome_id": "outc_01a...",
  "iteration": 0,
  "processed_at": "2026-03-25T14:01:45Z"
}
```

### 结果评估进行中

评分器运行期间发出的心跳事件。评分器的内部推理是不透明的：您只能看到它正在工作，而看不到它在思考什么。

```json
{
  "type": "span.outcome_evaluation_ongoing",
  "id": "sevt_01ghi...",
  "outcome_id": "outc_01a...",
  "iteration": 0,
  "processed_at": "2026-03-25T14:02:10Z"
}
```

### 结果评估结束

在一个结果评估周期结束时发出：即评分器完成对一次迭代的评估之后，或在结果处于活动状态时会话被中断时。`result` 字段指示接下来会发生什么。

| 结果                       | 下一步                                                                                                       |
| ------------------------ | --------------------------------------------------------------------------------------------------------- |
| `satisfied`              | 会话转换为 `idle`。                                                                                             |
| `needs_revision`         | 智能体开始新的迭代周期。                                                                                              |
| `max_iterations_reached` | 在会话转换为 `idle` 之前，会进行最后一轮确认。不再运行任何评估。                                                                      |
| `failed`                 | 会话转换为 `idle`。当评分标准不适用于交付物时返回，例如描述与评分标准相互矛盾时。                                                              |
| `interrupted`            | 当结果处于活动状态时会话被中断时发出，即使评估尚未开始。如果在中断之前没有触发 `outcome_evaluation_start`，则 `outcome_evaluation_start_id` 为空字符串。 |

```json
{
  "type": "span.outcome_evaluation_end",
  "id": "sevt_01jkl...",
  "outcome_evaluation_start_id": "sevt_01def...",
  "outcome_id": "outc_01a...",
  "result": "satisfied",
  "explanation": "All 12 criteria met: revenue projections use 5 years of historical data, WACC assumptions are stated, sensitivity table is included...",
  "iteration": 0,
  "usage": {
    "input_tokens": 2400,
    "output_tokens": 350,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 1800
  },
  "processed_at": "2026-03-25T14:03:00Z"
}
```

## 检查结果状态

您可以在[事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)上监听 `span.outcome_evaluation_end`，也可以轮询 `GET /v1/sessions/{session_id}` 并读取 `outcome_evaluations[].result`。在评估完成之前，`result` 会报告 `pending`、`running` 或 `evaluating`：

<CodeGroup>
  ```bash cURL
  curl -fsSL "https://api.anthropic.com/v1/sessions/$SESSION_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"
  ```

  ```bash CLI
  ant beta:sessions retrieve --session-id "$SESSION_ID"
  ```

  ```python Python
  session = client.beta.sessions.retrieve(session.id)

  for outcome in session.outcome_evaluations:
      print(f"{outcome.outcome_id}: {outcome.result}")
      # outc_01a...: satisfied
  ```

  ```typescript TypeScript
  const retrieved = await client.beta.sessions.retrieve(session.id);

  for (const outcome of retrieved.outcome_evaluations) {
    console.log(`${outcome.outcome_id}: ${outcome.result}`);
    // outc_01a...: satisfied
  }
  ```

  ```csharp C#
  session = await client.Beta.Sessions.Retrieve(session.ID);

  foreach (var outcome in session.OutcomeEvaluations)
  {
      Console.WriteLine($"{outcome.OutcomeID}: {outcome.Result}");
      // outc_01a...: satisfied
  }
  ```

  ```go Go
  session, err = client.Beta.Sessions.Get(ctx, session.ID, anthropic.BetaSessionGetParams{})
  if err != nil {
  	panic(err)
  }

  for _, outcome := range session.OutcomeEvaluations {
  	fmt.Printf("%s: %s\n", outcome.OutcomeID, outcome.Result)
  	// outc_01a...: satisfied
  }
  ```

  ```java Java
  var retrieved = client.beta().sessions().retrieve(session.id());

  for (var outcome : retrieved.outcomeEvaluations()) {
      IO.println(outcome.outcomeId() + ": " + outcome.result());
      // outc_01a...: satisfied
  }
  ```

  ```php PHP
  $session = $client->beta->sessions->retrieve($session->id);

  foreach ($session->outcomeEvaluations as $outcome) {
      echo "{$outcome->outcomeID}: {$outcome->result}\n";
      // outc_01a...: satisfied
  }
  ```

  ```ruby Ruby
  session = client.beta.sessions.retrieve(session.id)

  session.outcome_evaluations.each do
    puts "#{it.outcome_id}: #{it.result}"
    # outc_01a...: satisfied
  end
  ```
</CodeGroup>

## 获取交付物

智能体会将输出文件写入沙箱内的 `/mnt/session/outputs/`。要获取这些文件，请通过 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 列出文件，并将会话 ID 作为 `scope_id`，然后按 ID 下载。按 `scope_id` 筛选需要在列表请求中包含 `managed-agents-2026-04-01` beta 标头，因此 SDK 和 CLI 示例通过 `beta` 命名空间发起该调用，并显式传递该标头。文件会在智能体写入完成后不久出现在列表中，有时会在会话进入空闲状态几秒钟之后才出现。如果您期望的文件尚未列出，请稍等片刻后再次列出；一旦文件出现在列表中，就表示其上传已完成。

<CodeGroup>
  ```bash cURL
  # 列出此会话生成的文件
  # scope_id 过滤需要 managed-agents 测试版
  curl -fsSL "https://api.anthropic.com/v1/files?scope_id=$SESSION_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"

  # 下载文件
  FILE_ID=$(curl -fsSL "https://api.anthropic.com/v1/files?scope_id=$SESSION_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" | jq -r '.data[0].id // empty')
  if [[ -n $FILE_ID ]]; then
    curl -fsSL "https://api.anthropic.com/v1/files/$FILE_ID/content" \
      -H "x-api-key: $ANTHROPIC_API_KEY" \
      -H "anthropic-version: 2023-06-01" \
      -H "anthropic-beta: managed-agents-2026-04-01" \
      -o /tmp/output.txt
  fi
  ```

  ```bash CLI
  # 列出此会话生成的文件
  # scope_id 过滤要求在文件请求中启用 managed-agents beta
  ant beta:files list --scope-id "$SESSION_ID" --beta managed-agents-2026-04-01

  # 下载文件
  FILE_ID=$(ant beta:files list --scope-id "$SESSION_ID" \
    --beta managed-agents-2026-04-01 \
    --transform 'data[0].id' --raw-output)
  if [[ -n $FILE_ID ]]; then
    ant files download --file-id "$FILE_ID" --output /tmp/output.txt
  fi
  ```

  ```python Python
  # 列出此会话生成的文件
  # scope_id 过滤要求在文件请求中启用 managed-agents beta
  files = client.beta.files.list(scope_id=session.id, betas=["managed-agents-2026-04-01"])
  for file in files:
      print(file.id, file.filename)

  # 下载文件
  if files.data:
      content = client.files.download(files.data[0].id)
      content.write_to_file("/tmp/output.txt")
  ```

  ```typescript TypeScript
  // 列出此会话生成的文件
  // scope_id 过滤要求在文件请求中启用 managed-agents beta
  const files = await client.beta.files.list({
    scope_id: session.id,
    betas: ["managed-agents-2026-04-01"],
  });
  for (const file of files.data) {
    console.log(file.id, file.filename);
  }

  // 下载文件
  if (files.data.length > 0) {
    const content = await client.files.download(files.data[0].id);
    await writeFile("/tmp/output.txt", new Uint8Array(await content.arrayBuffer()));
  }
  ```

  ```csharp C#
  // 列出此会话生成的文件
  // （scope_id 过滤要求在 files 请求上启用 managed-agents beta）
  var files = await client.Beta.Files.List(new()
  {
      ScopeID = session.ID,
      Betas = ["managed-agents-2026-04-01"],
  });
  foreach (var file in files.Items)
  {
      Console.WriteLine($"{file.ID} {file.Filename}");
  }

  // 下载文件
  if (files.Items.Count > 0)
  {
      using var download = await client.Files.Download(files.Items[0].ID);
      await using var output = File.Create("/tmp/output.txt");
      await (await download.ReadAsStream()).CopyToAsync(output);
  }
  ```

  ```go Go
  // 列出此会话生成的文件
  // （scope_id 过滤需要在文件请求中启用 managed-agents beta）
  files, err := client.Beta.Files.List(ctx, anthropic.BetaFileListParams{
  	ScopeID: anthropic.String(session.ID),
  	Betas:   []anthropic.AnthropicBeta{anthropic.AnthropicBetaManagedAgents2026_04_01},
  })
  if err != nil {
  	panic(err)
  }
  for _, file := range files.Data {
  	fmt.Println(file.ID, file.Filename)
  }

  // 下载文件
  if len(files.Data) > 0 {
  	resp, err := client.Files.Download(ctx, files.Data[0].ID, anthropic.FileDownloadParams{})
  	if err != nil {
  		panic(err)
  	}
  	defer resp.Body.Close()
  	out, err := os.Create("/tmp/output.txt")
  	if err != nil {
  		panic(err)
  	}
  	defer out.Close()
  	if _, err := io.Copy(out, resp.Body); err != nil {
  		panic(err)
  	}
  }
  ```

  ```java Java
  // 列出此会话生成的文件
  // （scope_id 过滤要求在文件请求中启用 managed-agents beta）
  var files = client.beta().files().list(
      FileListParams.builder()
          .scopeId(session.id())
          .addBeta(AnthropicBeta.MANAGED_AGENTS_2026_04_01)
          .build());
  for (var file : files.data()) {
      IO.println(file.id() + " " + file.filename());
  }

  // 下载文件
  if (!files.data().isEmpty()) {
      try (HttpResponse response = client.files().download(files.data().getFirst().id())) {
          try (InputStream body = response.body()) {
              Files.copy(body, Path.of("/tmp/output.txt"), StandardCopyOption.REPLACE_EXISTING);
          }
      }
  }
  ```

  ```php PHP
  // 列出此会话生成的文件
  // scope_id 过滤要求在文件请求中启用 managed-agents beta
  $files = $client->beta->files->list(scopeID: $session->id, betas: ['managed-agents-2026-04-01']);
  foreach ($files->getItems() as $file) {
      echo "{$file->id} {$file->filename}\n";
  }

  // 下载文件
  if (count($files->getItems()) > 0) {
      $content = $client->files->download($files->getItems()[0]->id);
      file_put_contents('/tmp/output.txt', $content);
  }
  ```

  ```ruby Ruby
  # 列出此会话生成的文件
  # scope_id 过滤要求在文件请求中启用 managed-agents beta
  files = client.beta.files.list(scope_id: session.id, betas: ["managed-agents-2026-04-01"])
  files.data.each { |file| puts "#{file.id} #{file.filename}" }

  # 下载文件
  if (first = files.data.first)
    content = client.files.download(first.id)
    File.binwrite("/tmp/output.txt", content.read)
  end
  ```
</CodeGroup>

## 后续步骤

<CardGroup cols={3}>
  <Card title="使用保管库进行身份验证" icon="fingerprint" href="https://platform.claude.com/docs/zh-CN/managed-agents/vaults">
    在创建会话时注册每个用户的凭据。
  </Card>

  <Card title="会话事件流" icon="lightning" href="https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming">
    发送事件、流式传输响应，并在执行过程中中断或重定向您的会话。
  </Card>

  <Card title="添加文件" icon="file" href="https://platform.claude.com/docs/zh-CN/managed-agents/files">
    上传文件并将其挂载到您的沙箱中，以供读取和处理。
  </Card>
</CardGroup>
