---
title: 缓存诊断
url: https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics
description: 通过比较连续请求并精确识别提示前缀发生分歧的位置，诊断意外的提示缓存未命中问题。
---

## Compatibility
- Status: Beta
- [Beta header](https://platform.claude.com/docs/en/api/beta-headers): `cache-diagnosis-2026-04-07`
- [ZDR](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): eligible (excludes [Covered Models](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#model-specific-data-retention-requirements))
- Platforms: Claude API (beta); not available on Claude Platform on AWS, Amazon Bedrock, Google Cloud, Microsoft Foundry

[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)（prompt caching）可以显著降低延迟和成本，但前提是您的提示开头与最近的某个请求逐字节完全相同。工具顺序的调整、插入到系统提示中的时间戳，或对较早消息的编辑，都可能悄无声息地使缓存失效。如果没有缓存诊断，唯一的信号就是 `usage.cache_read_input_tokens` 降为零，而没有任何关于发生了什么变化的提示。

缓存诊断（cache diagnostics）填补了这一空白。传入您上一个响应的 `id`，API 会比较这两个请求并告诉您它们在哪里发生了分歧（模型、系统提示、工具或消息历史），这样您就可以修复根本原因，而不是靠猜测。

## 缓存诊断的工作原理

当存在 beta 请求头时，API 会为每个请求存储一个轻量级的"fingerprint"（指纹），以响应 `id` 作为键。在您的下一个请求中，将该 `id` 作为 `diagnostics.previous_message_id` 传入。API 会为新请求重建指纹，将其与已存储的指纹进行比较，并在响应中附加一个 `diagnostics` 对象，描述第一个分歧点。

该比较针对的是请求结构，与缓存是否实际命中无关。有关如何将 `diagnostics` 结果与 `usage.cache_read_input_tokens` 结合使用，请参阅[结合 usage 解读诊断结果](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics#reading-diagnostics-alongside-usage)。

指纹仅包含哈希值和令牌数量估计值（绝不包含原始提示内容），保留时间有限，作用域限定在您的组织和工作区内，并且不会用于任何其他目的。

## 基本用法

在每一轮都发送 beta 请求头。在第一轮，传入 `"previous_message_id": null` 以选择启用该功能，此时没有可供比较的先前消息。在后续轮次中，传入上一个响应的 `id`。

<CodeGroup>
  ```bash cURL
  # 第 1 轮：建立缓存并选择启用诊断
  response=$(curl -sS --fail-with-body https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: cache-diagnosis-2026-04-07" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "cache_control": {"type": "ephemeral"},
      "system": "You are an AI assistant analyzing a large document. <document>...</document>",
      "messages": [{"role": "user", "content": "Summarize section 1."}],
      "diagnostics": {"previous_message_id": null}
    }')
  jq '{id, diagnostics}' <<< "$response"
  message_id=$(jq -r '.id' <<< "$response")

  # 第 2 轮：引用上一轮，以便 API 比较前缀
  curl -sS --fail-with-body https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: cache-diagnosis-2026-04-07" \
    -H "content-type: application/json" \
    -d @- <<EOF | jq '{id, diagnostics}'  # diagnostics: null means no divergence was found
  {
    "model": "claude-opus-5",
    "max_tokens": 1024,
    "cache_control": {"type": "ephemeral"},
    "system": "You are an AI assistant analyzing a large document. <document>...</document>",
    "messages": [
      {"role": "user", "content": "Summarize section 1."},
      {"role": "assistant", "content": "Section 1 covers..."},
      {"role": "user", "content": "Now summarize section 2."}
    ],
    "diagnostics": {"previous_message_id": "$message_id"}
  }
  EOF
  ```

  ```bash CLI
  # 第 1 轮
  turn1=$(ant beta:messages create \
    --beta cache-diagnosis-2026-04-07 \
    --transform '{id,usage,diagnostics}' <<'YAML'
  model: claude-opus-5
  max_tokens: 1024
  cache_control:
    type: ephemeral
  system: "You are an AI assistant analyzing a large document. <document>...</document>"
  messages:
    - role: user
      content: Summarize section 1.
  diagnostics:
    previous_message_id: null
  YAML
  )
  printf '%s\n' "$turn1"

  # 第 2 轮：将第 1 轮返回的 id 作为 previous_message_id 传入
  message_id=$(jq -r '.id' <<<"$turn1")
  ant beta:messages create \
    --beta cache-diagnosis-2026-04-07 \
    --transform '{id,usage,diagnostics}' <<YAML
  model: claude-opus-5
  max_tokens: 1024
  cache_control:
    type: ephemeral
  system: "You are an AI assistant analyzing a large document. <document>...</document>"
  messages:
    - role: user
      content: Summarize section 1.
    - role: assistant
      content: Section 1 covers...
    - role: user
      content: Now summarize section 2.
  diagnostics:
    previous_message_id: $message_id
  YAML
  ```

  ```python Python
  client = anthropic.Anthropic()

  SYSTEM = "You are an AI assistant analyzing a large document. <document>...</document>"

  # 第 1 轮：通过 previous_message_id=None 选择启用
  r1 = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      cache_control={"type": "ephemeral"},
      system=SYSTEM,
      messages=[{"role": "user", "content": "Summarize section 1."}],
      diagnostics={"previous_message_id": None},
      betas=["cache-diagnosis-2026-04-07"],
  )

  # 第 2 轮：引用上一个响应的 id
  r2 = client.beta.messages.create(
      model="claude-opus-5",
      max_tokens=1024,
      cache_control={"type": "ephemeral"},
      system=SYSTEM,
      messages=[
          {"role": "user", "content": "Summarize section 1."},
          {"role": "assistant", "content": r1.content},
          {"role": "user", "content": "Now summarize section 2."},
      ],
      diagnostics={"previous_message_id": r1.id},
      betas=["cache-diagnosis-2026-04-07"],
  )

  diagnostics = r2.diagnostics
  if diagnostics is None:
      print("No divergence detected.")
  elif diagnostics.cache_miss_reason is None:
      print("Comparison still pending.")
  else:
      print(f"cache_miss_reason: {diagnostics.cache_miss_reason.type}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const SYSTEM = "You are an AI assistant analyzing a large document. <document>...</document>";

  // 第 1 轮：通过 previous_message_id: null 选择启用
  const r1 = await client.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    cache_control: { type: "ephemeral" },
    system: SYSTEM,
    messages: [{ role: "user", content: "Summarize section 1." }],
    diagnostics: { previous_message_id: null },
    betas: ["cache-diagnosis-2026-04-07"]
  });

  // 第 2 轮：引用上一个响应的 id
  const r2 = await client.beta.messages.create({
    model: "claude-opus-5",
    max_tokens: 1024,
    cache_control: { type: "ephemeral" },
    system: SYSTEM,
    messages: [
      { role: "user", content: "Summarize section 1." },
      { role: "assistant", content: r1.content },
      { role: "user", content: "Now summarize section 2." }
    ],
    diagnostics: { previous_message_id: r1.id },
    betas: ["cache-diagnosis-2026-04-07"]
  });

  if (r2.diagnostics === null) {
    console.log("No divergence detected.");
  } else if (r2.diagnostics.cache_miss_reason === null) {
    console.log("Comparison still pending.");
  } else {
    console.log(`cache_miss_reason: ${r2.diagnostics.cache_miss_reason.type}`);
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var system = "You are an AI assistant analyzing a large document. <document>...</document>";

  var r1 = await client.Beta.Messages.Create(
      new()
      {
          Model = Messages::Model.ClaudeOpus5,
          MaxTokens = 1024,
          CacheControl = new(),
          System = system,
          Messages =
          [
              new() { Role = Role.User, Content = "Summarize section 1." },
          ],
          Diagnostics = new() { PreviousMessageID = null },
          Betas = [AnthropicBeta.CacheDiagnosis2026_04_07],
      }
  );

  var r2 = await client.Beta.Messages.Create(
      new()
      {
          Model = Messages::Model.ClaudeOpus5,
          MaxTokens = 1024,
          CacheControl = new(),
          System = system,
          Messages =
          [
              new() { Role = Role.User, Content = "Summarize section 1." },
              new()
              {
                  Role = Role.Assistant,
                  Content = r1.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
              },
              new() { Role = Role.User, Content = "Now summarize section 2." },
          ],
          Diagnostics = new() { PreviousMessageID = r1.ID },
          Betas = [AnthropicBeta.CacheDiagnosis2026_04_07],
      }
  );

  Console.WriteLine(r2.Diagnostics switch
  {
      null => "No divergence detected.",
      { CacheMissReason: null } => "Comparison still pending.",
      { CacheMissReason.Type: var type } => $"cache_miss_reason: {type.GetString()}",
  });
  ```

  ```go Go
  client := anthropic.NewClient()
  ctx := context.Background()

  system := []anthropic.BetaTextBlockParam{
  	{Text: "You are an AI assistant analyzing a large document. <document>...</document>"},
  }

  r1, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  	Model:        anthropic.ModelClaudeOpus5,
  	MaxTokens:    1024,
  	CacheControl: anthropic.BetaCacheControlEphemeralParam{},
  	System:       system,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Summarize section 1.")),
  	},
  	Diagnostics: anthropic.BetaDiagnosticsParam{
  		PreviousMessageID: param.Null[string](),
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaCacheDiagnosis2026_04_07},
  })
  if err != nil {
  	panic(err)
  }

  r2, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
  	Model:        anthropic.ModelClaudeOpus5,
  	MaxTokens:    1024,
  	CacheControl: anthropic.BetaCacheControlEphemeralParam{},
  	System:       system,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Summarize section 1.")),
  		r1.ToParam(),
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Now summarize section 2.")),
  	},
  	Diagnostics: anthropic.BetaDiagnosticsParam{
  		PreviousMessageID: anthropic.String(r1.ID),
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaCacheDiagnosis2026_04_07},
  })
  if err != nil {
  	panic(err)
  }

  switch {
  case !r2.JSON.Diagnostics.Valid():
  	fmt.Println("No divergence detected.")
  case !r2.Diagnostics.JSON.CacheMissReason.Valid():
  	fmt.Println("Comparison still pending.")
  default:
  	fmt.Printf("cache_miss_reason: %s\n", r2.Diagnostics.CacheMissReason.Type)
  }
  ```

  ```java Java
  var client = AnthropicOkHttpClient.fromEnv();

  var system = "You are an AI assistant analyzing a large document. <document>...</document>";

  var r1 = client.beta().messages().create(
      MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .cacheControl(BetaCacheControlEphemeral.builder().build())
          .system(system)
          .addUserMessage("Summarize section 1.")
          // 首轮传入 null 即可启用，此时没有先前的消息可供比较。
          .diagnostics(BetaDiagnosticsParam.builder().previousMessageId((String) null).build())
          .addBeta(AnthropicBeta.CACHE_DIAGNOSIS_2026_04_07)
          .build()
  );

  var r2 = client.beta().messages().create(
      MessageCreateParams.builder()
          .model(Model.CLAUDE_OPUS_5)
          .maxTokens(1024)
          .cacheControl(BetaCacheControlEphemeral.builder().build())
          .system(system)
          .addUserMessage("Summarize section 1.")
          .addMessage(r1)
          .addUserMessage("Now summarize section 2.")
          .diagnostics(BetaDiagnosticsParam.builder().previousMessageId(r1.id()).build())
          .addBeta(AnthropicBeta.CACHE_DIAGNOSIS_2026_04_07)
          .build()
  );

  if (r2.diagnostics().isEmpty()) {
      IO.println("No divergence detected.");
  } else if (r2.diagnostics().get().cacheMissReason().isEmpty()) {
      IO.println("Comparison still pending.");
  } else {
      var reason = r2.diagnostics().get().cacheMissReason().get();
      // CacheMissReason 未提供带类型的 .type() 访问器；请从原始 JSON 中读取。
      @SuppressWarnings("unchecked")
      var json = (Map<String, JsonValue>) reason._json().orElseThrow().asObject().orElseThrow();
      IO.println("cache_miss_reason: " + json.get("type").asStringOrThrow());
  }
  ```

  ```php PHP
  $client = new Client();

  $system = 'You are an AI assistant analyzing a large document. <document>...</document>';

  $r1 = $client->beta->messages->create(
      model: Model::CLAUDE_OPUS_5,
      maxTokens: 1024,
      cacheControl: new BetaCacheControlEphemeral,
      system: $system,
      messages: [
          ['role' => 'user', 'content' => 'Summarize section 1.'],
      ],
      diagnostics: (new BetaDiagnosticsParam)->withPreviousMessageID(null),
      betas: [AnthropicBeta::CACHE_DIAGNOSIS_2026_04_07],
  );

  $r2 = $client->beta->messages->create(
      model: Model::CLAUDE_OPUS_5,
      maxTokens: 1024,
      cacheControl: new BetaCacheControlEphemeral,
      system: $system,
      messages: [
          ['role' => 'user', 'content' => 'Summarize section 1.'],
          ['role' => 'assistant', 'content' => $r1->content],
          ['role' => 'user', 'content' => 'Now summarize section 2.'],
      ],
      diagnostics: (new BetaDiagnosticsParam)->withPreviousMessageID($r1->id),
      betas: [AnthropicBeta::CACHE_DIAGNOSIS_2026_04_07],
  );

  echo match (true) {
      $r2->diagnostics === null => "No divergence detected.\n",
      $r2->diagnostics->cacheMissReason === null => "Comparison still pending.\n",
      default => "cache_miss_reason: {$r2->diagnostics->cacheMissReason->type}\n",
  };
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  SYSTEM = "You are an AI assistant analyzing a large document. <document>...</document>"

  r1 = client.beta.messages.create(
    model: :"claude-opus-5",
    max_tokens: 1024,
    cache_control: {type: "ephemeral"},
    system_: SYSTEM,
    messages: [
      {role: "user", content: "Summarize section 1."}
    ],
    diagnostics: {previous_message_id: nil},
    betas: ["cache-diagnosis-2026-04-07"]
  )

  r2 = client.beta.messages.create(
    model: :"claude-opus-5",
    max_tokens: 1024,
    cache_control: {type: "ephemeral"},
    system_: SYSTEM,
    messages: [
      {role: "user", content: "Summarize section 1."},
      {role: "assistant", content: r1.content},
      {role: "user", content: "Now summarize section 2."}
    ],
    diagnostics: {previous_message_id: r1.id},
    betas: ["cache-diagnosis-2026-04-07"]
  )

  case r2.diagnostics
  in nil
    puts "No divergence detected."
  in {cache_miss_reason: nil}
    puts "Comparison still pending."
  in {cache_miss_reason: {type:}}
    puts "cache_miss_reason: #{type}"
  end
  ```
</CodeGroup>

## 流式传输

在流式传输（streaming）响应中，`diagnostics` 出现在 `message_start` 事件上。

<CodeGroup>
  ```bash cURL
  # 第 2 轮：流式传输响应。diagnostics 随 message_start 事件到达；
  # null 值表示未发现差异。
  curl -sS --fail-with-body https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: cache-diagnosis-2026-04-07" \
    -H "content-type: application/json" \
    -d @- <<EOF | jq -R 'select(startswith("data: ")) | ltrimstr("data: ") | fromjson | select(.type == "message_start") | .message.diagnostics'
  {
    "model": "claude-opus-5",
    "max_tokens": 1024,
    "stream": true,
    "cache_control": {"type": "ephemeral"},
    "system": "You are an AI assistant analyzing a large document. <document>...</document>",
    "messages": [
      {"role": "user", "content": "Summarize section 1."},
      {"role": "assistant", "content": "Section 1 covers..."},
      {"role": "user", "content": "Now summarize section 2."}
    ],
    "diagnostics": {"previous_message_id": "$message_id"}
  }
  EOF
  ```

  ```bash CLI
  # 第 2 轮：流式传输。使用 --stream 时，CLI 会将每个 SSE 事件作为一个 JSON 对象输出。
  # diagnostics 随 message_start 事件到达；用 jq 将其提取出来。
  ant beta:messages create \
    --beta cache-diagnosis-2026-04-07 \
    --stream --format jsonl <<YAML |
  model: claude-opus-5
  max_tokens: 1024
  cache_control:
    type: ephemeral
  system: "You are an AI assistant analyzing a large document. <document>...</document>"
  messages:
    - role: user
      content: Summarize section 1.
    - role: assistant
      content: Section 1 covers...
    - role: user
      content: Now summarize section 2.
  diagnostics:
    previous_message_id: $message_id
  YAML
    jq -c 'select(.type == "message_start") | .message | {id,usage,diagnostics}'
  ```

  ```python Python
  # 第 2 轮：流式传输，引用上一个响应的 id
  with client.beta.messages.stream(
      model="claude-opus-5",
      max_tokens=1024,
      cache_control={"type": "ephemeral"},
      system=SYSTEM,
      messages=[
          {"role": "user", "content": "Summarize section 1."},
          {"role": "assistant", "content": r1.content},
          {"role": "user", "content": "Now summarize section 2."},
      ],
      diagnostics={"previous_message_id": r1.id},
      betas=["cache-diagnosis-2026-04-07"],
  ) as stream:
      for text in stream.text_stream:
          print(text, end="", flush=True)
      print()
      r2 = stream.get_final_message()

  diagnostics = r2.diagnostics
  if diagnostics is None:
      print("No divergence detected.")
  elif diagnostics.cache_miss_reason is None:
      print("Comparison still pending.")
  else:
      print(f"cache_miss_reason: {diagnostics.cache_miss_reason.type}")
  ```

  ```typescript TypeScript
  const stream = client.beta.messages.stream({
    model: "claude-opus-5",
    max_tokens: 1024,
    cache_control: { type: "ephemeral" },
    system: SYSTEM,
    messages: [
      { role: "user", content: "Summarize section 1." },
      { role: "assistant", content: r1.content },
      { role: "user", content: "Now summarize section 2." }
    ],
    diagnostics: { previous_message_id: r1.id },
    betas: ["cache-diagnosis-2026-04-07"]
  });

  for await (const event of stream) {
    if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
      process.stdout.write(event.delta.text);
    }
  }
  process.stdout.write("\n");

  // diagnostics 在 message_start 时到达，并一直保留到最终消息
  const r2 = await stream.finalMessage();

  if (r2.diagnostics === null) {
    console.log("No divergence detected.");
  } else if (r2.diagnostics.cache_miss_reason === null) {
    console.log("Comparison still pending.");
  } else {
    console.log(`cache_miss_reason: ${r2.diagnostics.cache_miss_reason.type}`);
  }
  ```

  ```csharp C#
  // 第 2 轮：流式传输，引用上一个响应的 id
  BetaDiagnostics? diagnostics = null;

  var stream = client.Beta.Messages.CreateStreaming(
      new()
      {
          Model = Messages::Model.ClaudeOpus5,
          MaxTokens = 1024,
          CacheControl = new(),
          System = system,
          Messages =
          [
              new() { Role = Role.User, Content = "Summarize section 1." },
              new()
              {
                  Role = Role.Assistant,
                  Content = r1.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
              },
              new() { Role = Role.User, Content = "Now summarize section 2." },
          ],
          Diagnostics = new() { PreviousMessageID = r1.ID },
          Betas = [AnthropicBeta.CacheDiagnosis2026_04_07],
      }
  );

  await foreach (var streamEvent in stream)
  {
      if (streamEvent.TryPickStart(out var start))
      {
          // diagnostics 随 message_start 事件到达
          diagnostics = start.Message.Diagnostics;
      }
      else if (streamEvent.TryPickContentBlockDelta(out var delta) && delta.Delta.TryPickText(out var textDelta))
      {
          Console.Write(textDelta.Text);
      }
  }
  Console.WriteLine();

  Console.WriteLine(diagnostics switch
  {
      null => "No divergence detected.",
      { CacheMissReason: null } => "Comparison still pending.",
      { CacheMissReason.Type: var type } => $"cache_miss_reason: {type.GetString()}",
  });
  ```

  ```go Go
  // 第 2 轮：流式传输，引用上一个响应的 id
  stream := client.Beta.Messages.NewStreaming(ctx, anthropic.BetaMessageNewParams{
  	Model:        anthropic.ModelClaudeOpus5,
  	MaxTokens:    1024,
  	CacheControl: anthropic.BetaCacheControlEphemeralParam{},
  	System:       system,
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Summarize section 1.")),
  		r1.ToParam(),
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Now summarize section 2.")),
  	},
  	Diagnostics: anthropic.BetaDiagnosticsParam{
  		PreviousMessageID: anthropic.String(r1.ID),
  	},
  	Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaCacheDiagnosis2026_04_07},
  })
  defer stream.Close()

  // diagnostics 随 message_start 到达；Accumulate 将其带入 r2
  var r2 anthropic.BetaMessage
  for stream.Next() {
  	if err := r2.Accumulate(stream.Current()); err != nil {
  		panic(err)
  	}
  }
  if err := stream.Err(); err != nil {
  	panic(err)
  }

  switch {
  case !r2.JSON.Diagnostics.Valid():
  	fmt.Println("No divergence detected.")
  case !r2.Diagnostics.JSON.CacheMissReason.Valid():
  	fmt.Println("Comparison still pending.")
  default:
  	fmt.Printf("cache_miss_reason: %s\n", r2.Diagnostics.CacheMissReason.Type)
  }
  ```

  ```java Java
  // 第 2 轮：流式传输，引用上一个响应的 id
  var params = MessageCreateParams.builder()
      .model(Model.CLAUDE_OPUS_5)
      .maxTokens(1024)
      .cacheControl(BetaCacheControlEphemeral.builder().build())
      .system(system)
      .addUserMessage("Summarize section 1.")
      .addMessage(r1)
      .addUserMessage("Now summarize section 2.")
      .diagnostics(BetaDiagnosticsParam.builder().previousMessageId(r1.id()).build())
      .addBeta(AnthropicBeta.CACHE_DIAGNOSIS_2026_04_07)
      .build();

  var accumulator = BetaMessageAccumulator.create();
  try (var streamResponse = client.beta().messages().createStreaming(params)) {
      streamResponse.stream()
          .peek(accumulator::accumulate)
          .flatMap(event -> event.contentBlockDelta().stream())
          .flatMap(deltaEvent -> deltaEvent.delta().text().stream())
          .forEach(textDelta -> IO.print(textDelta.text()));
      IO.println("");
  }

  // diagnostics 在 message_start 时到达，并一直保留到累积的消息中
  var diagnostics = accumulator.message().diagnostics();
  if (diagnostics.isEmpty()) {
      IO.println("No divergence detected.");
  } else if (diagnostics.get().cacheMissReason().isEmpty()) {
      IO.println("Comparison still pending.");
  } else {
      var reason = diagnostics.get().cacheMissReason().get();
      // CacheMissReason 未公开类型化的 .type() 访问器；请从原始 JSON 中读取。
      @SuppressWarnings("unchecked")
      var json = (Map<String, JsonValue>) reason._json().orElseThrow().asObject().orElseThrow();
      IO.println("cache_miss_reason: " + json.get("type").asStringOrThrow());
  }
  ```

  ```php PHP
  // 第 2 轮：流式传输，引用上一个响应的 id
  $stream = $client->beta->messages->createStream(
      model: Model::CLAUDE_OPUS_5,
      maxTokens: 1024,
      cacheControl: new BetaCacheControlEphemeral,
      system: $system,
      messages: [
          ['role' => 'user', 'content' => 'Summarize section 1.'],
          ['role' => 'assistant', 'content' => $r1->content],
          ['role' => 'user', 'content' => 'Now summarize section 2.'],
      ],
      diagnostics: (new BetaDiagnosticsParam)->withPreviousMessageID($r1->id),
      betas: [AnthropicBeta::CACHE_DIAGNOSIS_2026_04_07],
  );

  $diagnostics = null;
  foreach ($stream as $event) {
      if ($event instanceof BetaRawMessageStartEvent) {
          // diagnostics 随 message_start 事件中嵌入的 BetaMessage 一起到达
          $diagnostics = $event->message->diagnostics;
      } elseif ($event instanceof BetaRawContentBlockDeltaEvent && $event->delta instanceof BetaTextDelta) {
          echo $event->delta->text;
      }
  }
  echo PHP_EOL;

  echo match (true) {
      $diagnostics === null => "No divergence detected.\n",
      $diagnostics->cacheMissReason === null => "Comparison still pending.\n",
      default => "cache_miss_reason: {$diagnostics->cacheMissReason->type}\n",
  };
  ```

  ```ruby Ruby
  # 第 2 轮：流式传输，引用上一个响应的 id
  stream = client.beta.messages.stream(
    model: :"claude-opus-5",
    max_tokens: 1024,
    cache_control: {type: "ephemeral"},
    system_: SYSTEM,
    messages: [
      {role: "user", content: "Summarize section 1."},
      {role: "assistant", content: r1.content},
      {role: "user", content: "Now summarize section 2."}
    ],
    diagnostics: {previous_message_id: r1.id},
    betas: ["cache-diagnosis-2026-04-07"]
  )

  stream.each do |event|
    print(event.text) if event.is_a?(Anthropic::Streaming::TextEvent)
  end
  puts

  # diagnostics 在 message_start 时到达，并保留在累积的消息上
  r2 = stream.accumulated_message

  case r2.diagnostics
  in nil
    puts "No divergence detected."
  in {cache_miss_reason: nil}
    puts "Comparison still pending."
  in {cache_miss_reason: {type:}}
    puts "cache_miss_reason: #{type}"
  end
  ```
</CodeGroup>

`message_start` 事件携带完整的 `diagnostics` 字段；有关可能的取值，请参阅[响应格式](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics#response-format)。

## 在对话循环中传递诊断信息

在多轮对话中，每一轮都将最新的响应 `id` 作为 `previous_message_id` 向前传递。第一次迭代传入 `null` 以选择启用；之后的每次迭代传入上一个响应的 `id`。

<Tabs>
  <Tab title="cURL">
    <Info>
      此工作流程不太适合用一次性的 shell 命令来表达。请参阅 SDK 选项卡了解循环模式；每轮的 HTTP 请求与[基本用法](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics#basic-usage)相同。
    </Info>
  </Tab>

  <Tab title="CLI">
    <Info>
      此工作流程不太适合用一次性的 shell 命令来表达。请参阅 SDK 选项卡了解循环模式；每轮的 CLI 调用与[基本用法](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics#basic-usage)相同。
    </Info>
  </Tab>

  <Tab title="Python">
    ```python
    client = anthropic.Anthropic()

    SYSTEM = "You are an AI assistant analyzing a large document. <document>...</document>"

    messages = []
    prev_id = None

    for i, user_message in enumerate(
        ["Summarize section 1.", "Now section 2.", "Now section 3."]
    ):
        messages.append({"role": "user", "content": user_message})

        r = client.beta.messages.create(
            model="claude-opus-5",
            max_tokens=1024,
            cache_control={"type": "ephemeral"},
            system=SYSTEM,
            messages=messages,
            diagnostics={"previous_message_id": prev_id},
            betas=["cache-diagnosis-2026-04-07"],
        )

        if r.diagnostics is not None and r.diagnostics.cache_miss_reason is not None:
            print(f"Turn {i + 1} cache_miss_reason: {r.diagnostics.cache_miss_reason.type}")

        messages.append({"role": "assistant", "content": r.content})
        prev_id = r.id
    ```
  </Tab>

  <Tab title="TypeScript">
    ```typescript
    const client = new Anthropic();

    const SYSTEM = "You are an AI assistant analyzing a large document. <document>...</document>";

    const prompts = ["Summarize section 1.", "Now section 2.", "Now section 3."];

    const messages: BetaMessageParam[] = [];
    let prevId: string | null = null;

    for (const [i, prompt] of prompts.entries()) {
      messages.push({ role: "user", content: prompt });

      const r: BetaMessage = await client.beta.messages.create({
        model: "claude-opus-5",
        max_tokens: 1024,
        cache_control: { type: "ephemeral" },
        system: SYSTEM,
        messages,
        diagnostics: { previous_message_id: prevId },
        betas: ["cache-diagnosis-2026-04-07"]
      });

      if (r.diagnostics?.cache_miss_reason) {
        console.log(`Turn ${i + 1} cache_miss_reason: ${r.diagnostics.cache_miss_reason.type}`);
      }

      messages.push({ role: "assistant", content: r.content });
      prevId = r.id;
    }
    ```
  </Tab>

  <Tab title="C#">
    ```csharp
    AnthropicClient client = new();

    var system = "You are an AI assistant analyzing a large document. <document>...</document>";

    List<BetaMessageParam> messages = [];
    string? prevId = null;
    string[] prompts = ["Summarize section 1.", "Now section 2.", "Now section 3."];

    for (int i = 0; i < prompts.Length; i++)
    {
        messages.Add(new() { Role = Role.User, Content = prompts[i] });

        var r = await client.Beta.Messages.Create(
            new()
            {
                Model = Messages::Model.ClaudeOpus5,
                MaxTokens = 1024,
                CacheControl = new(),
                System = system,
                Messages = messages,
                Diagnostics = new() { PreviousMessageID = prevId },
                Betas = [AnthropicBeta.CacheDiagnosis2026_04_07],
            }
        );

        if (r.Diagnostics?.CacheMissReason is { Type: var type })
        {
            Console.WriteLine($"Turn {i + 1} cache_miss_reason: {type.GetString()}");
        }

        messages.Add(
            new()
            {
                Role = Role.Assistant,
                Content = r.Content.Select(block => new BetaContentBlockParam(block.Json)).ToList(),
            }
        );
        prevId = r.ID;
    }
    ```
  </Tab>

  <Tab title="Go">
    ```go
    client := anthropic.NewClient()
    ctx := context.Background()

    system := []anthropic.BetaTextBlockParam{
    	{Text: "You are an AI assistant analyzing a large document. <document>...</document>"},
    }

    prompts := []string{"Summarize section 1.", "Now section 2.", "Now section 3."}

    var messages []anthropic.BetaMessageParam
    prevID := param.Null[string]()

    for turn, prompt := range prompts {
    	messages = append(messages, anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock(prompt)))

    	r, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
    		Model:        anthropic.ModelClaudeOpus5,
    		MaxTokens:    1024,
    		CacheControl: anthropic.BetaCacheControlEphemeralParam{},
    		System:       system,
    		Messages:     messages,
    		Diagnostics: anthropic.BetaDiagnosticsParam{
    			PreviousMessageID: prevID,
    		},
    		Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaCacheDiagnosis2026_04_07},
    	})
    	if err != nil {
    		panic(err)
    	}

    	if r.JSON.Diagnostics.Valid() && r.Diagnostics.JSON.CacheMissReason.Valid() {
    		fmt.Printf("Turn %d cache_miss_reason: %s\n", turn+1, r.Diagnostics.CacheMissReason.Type)
    	}

    	messages = append(messages, r.ToParam())
    	prevID = anthropic.String(r.ID)
    }
    ```
  </Tab>

  <Tab title="Java">
    ```java
    var client = AnthropicOkHttpClient.fromEnv();

    var system = "You are an AI assistant analyzing a large document. <document>...</document>";
    var prompts = List.of("Summarize section 1.", "Now section 2.", "Now section 3.");

    var messages = new ArrayList<BetaMessageParam>();
    String prevId = null;

    for (var turn = 0; turn < prompts.size(); turn++) {
        messages.add(
            BetaMessageParam.builder()
                .role(BetaMessageParam.Role.USER)
                .content(prompts.get(turn))
                .build()
        );

        var r = client.beta().messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_OPUS_5)
                .maxTokens(1024)
                .cacheControl(BetaCacheControlEphemeral.builder().build())
                .system(system)
                .messages(messages)
                .diagnostics(BetaDiagnosticsParam.builder().previousMessageId(prevId).build())
                .addBeta(AnthropicBeta.CACHE_DIAGNOSIS_2026_04_07)
                .build()
        );

        if (r.diagnostics().isPresent() && r.diagnostics().get().cacheMissReason().isPresent()) {
            var reason = r.diagnostics().get().cacheMissReason().get();
            // CacheMissReason 未提供类型化的 .type() 访问器；请从原始 JSON 中读取。
            @SuppressWarnings("unchecked")
            var json = (Map<String, JsonValue>) reason._json().orElseThrow().asObject().orElseThrow();
            IO.println("Turn " + (turn + 1) + " cache_miss_reason: " + json.get("type").asStringOrThrow());
        }

        messages.add(r.toParam());
        prevId = r.id();
    }
    ```
  </Tab>

  <Tab title="PHP">
    ```php
    $client = new Client();

    $system = 'You are an AI assistant analyzing a large document. <document>...</document>';

    $messages = [];
    $prevId = null;

    foreach (['Summarize section 1.', 'Now section 2.', 'Now section 3.'] as $i => $userMsg) {
        $turn = $i + 1;
        $messages[] = ['role' => 'user', 'content' => $userMsg];

        $r = $client->beta->messages->create(
            model: Model::CLAUDE_OPUS_5,
            maxTokens: 1024,
            cacheControl: new BetaCacheControlEphemeral,
            system: $system,
            messages: $messages,
            diagnostics: (new BetaDiagnosticsParam)->withPreviousMessageID($prevId),
            betas: [AnthropicBeta::CACHE_DIAGNOSIS_2026_04_07],
        );

        if ($r->diagnostics?->cacheMissReason !== null) {
            echo "Turn {$turn} cache_miss_reason: {$r->diagnostics->cacheMissReason->type}\n";
        }

        $messages[] = ['role' => 'assistant', 'content' => $r->content];
        $prevId = $r->id;
    }
    ```
  </Tab>

  <Tab title="Ruby">
    ```ruby
    client = Anthropic::Client.new

    SYSTEM = "You are an AI assistant analyzing a large document. <document>...</document>"

    messages = []
    prev_id = nil

    ["Summarize section 1.", "Now section 2.", "Now section 3."].each_with_index do |user_msg, i|
      messages << {role: "user", content: user_msg}

      r = client.beta.messages.create(
        model: :"claude-opus-5",
        max_tokens: 1024,
        cache_control: {type: "ephemeral"},
        system_: SYSTEM,
        messages: messages,
        diagnostics: {previous_message_id: prev_id},
        betas: ["cache-diagnosis-2026-04-07"]
      )

      if (reason = r.diagnostics&.cache_miss_reason)
        puts "Turn #{i + 1} cache_miss_reason: #{reason.type}"
      end

      messages << {role: "assistant", content: r.content}
      prev_id = r.id
    end
    ```
  </Tab>
</Tabs>

## 响应格式

响应 `Message` 上的 `diagnostics` 字段有四种可能的状态：

| 值                              | 含义                                                                                                               |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| 字段不存在                          | 请求未包含 `diagnostics`，或缺少 beta 请求头。                                                                                |
| `null`                         | 要么 `previous_message_id` 为 `null`（第一轮，没有可比较的内容），要么比较已运行且未发现分歧。                                                   |
| `{"cache_miss_reason": null}`  | 响应被序列化时比较仍在运行。当响应启动非常快时可能会发生这种情况。请将其视为无定论，并检查下一轮。                                                                |
| `{"cache_miss_reason": {...}}` | 附加了一个 `cache_miss_reason`。对于 `*_changed` 类型，它标识第一个分歧点；`previous_message_not_found` 和 `unavailable` 则是未产生比较结果的情况。 |

当 `cache_miss_reason` 非空时，它看起来像这样：

```json
{
  "id": "msg_01Xyz...",
  "type": "message",
  "role": "assistant",
  "content": [{ "type": "text", "text": "..." }],
  "usage": {
    "input_tokens": 42,
    "cache_read_input_tokens": 0,
    "cache_creation_input_tokens": 41850,
    "output_tokens": 210
  },
  "diagnostics": {
    "cache_miss_reason": {
      "type": "system_changed",
      "cache_missed_input_tokens": 41850
    }
  }
}
```

## 缓存未命中原因类型

`cache_miss_reason` 是一个基于 `type` 的可辨识联合类型（discriminated union）。响应仅报告最早的分歧，因此请先修复它；后面的分歧可能被它掩盖。

| 类型                           | 含义                                                                                                                                                                                                      | 需要更改的内容                                                                                                                                                               |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `model_changed`              | `model` 与上一个请求不同（例如，路由器、A/B 测试或回退机制选择了不同的模型）。缓存是按模型区分的。                                                                                                                                                 | 在一个已缓存的对话中保持模型不变。                                                                                                                                                     |
| `system_changed`             | `system` 参数不同。通常是时间戳、请求 ID 或其他每次请求都不同的值被插入到了系统提示中。                                                                                                                                                      | 使系统提示成为字节稳定的常量，并将动态数据移到缓存断点之后的第一条 `user` 消息中。                                                                                                                         |
| `tools_changed`              | `tools` 数组不同：在轮次之间添加、删除或重新排序了工具，或者工具的 `input_schema` JSON 被非确定性地序列化。                                                                                                                                    | 每一轮都以固定顺序发送相同的工具列表，并使用确定性序列化的 schema（例如，对键进行排序）。                                                                                                                      |
| `messages_changed`           | 模型、系统提示和工具全部匹配，但 `messages` 中较早的条目被修改、重新排序或删除，而不是仅追加。通常是对话历史被截断或编辑，或者助手轮次和 `tool_result` 块在重新发送时被以不同方式重新序列化。                                                                                            | 将历史视为仅追加；原样回传助手的 `content` 和工具结果。                                                                                                                                     |
| `previous_message_not_found` | 所提供的 `previous_message_id` 不存在已存储的指纹。这并不能证明您的请求发生了变化。通常是上一个请求未携带 beta 请求头、来自不同的工作区，或者自发送以来已过去太长时间。                                                                                                      | 每一轮都发送 beta 请求头，并使连续轮次在时间上保持接近。                                                                                                                                       |
| `unavailable`                | 此请求的诊断信息不可用。这包括以下情况：`model`、`system` 和 `tools` 匹配，但另一个影响提示的请求参数（`tool_choice`、`thinking`、`context_management`、`output_config`、`output_format` 或活动的 `anthropic-beta` 请求头集合）不同；以及分歧超出比较范围的超长对话。您的请求已正常处理。 | 在已缓存对话的整个生命周期内保持影响提示的请求参数不变。如果问题持续存在，请应用提示缓存页面上[常见问题排查](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#troubleshooting-common-issues)下的手动检查。 |

<Note>
  四种 `*_changed` 类型还携带一个 `cache_missed_input_tokens` 整数：这是对分歧点之后有多少输入令牌的估计，让您了解丢失了多少可缓存的前缀。它是在分词之前根据字节长度推导出来的，因此请将其视为量级指标而非计费数字。它可能与 `usage.input_tokens` 不同（偶尔会超过它）。
</Note>

## 结合 usage 解读诊断结果

`diagnostics` 回答的是"我的请求变了吗？"，而 `usage.cache_read_input_tokens` 回答的是"缓存命中了吗？"。将两者结合起来可以告诉您应该从哪里着手排查。

此矩阵适用于您传入了真实 `previous_message_id` 的轮次。在第一轮（`previous_message_id: null`），`diagnostics` 始终为 `null`，且 `cache_read_input_tokens` 通常为零，因为此时缓存正在被写入而非读取；无需排查。当 `cache_miss_reason` 为 `null`（比较仍在进行中；请检查下一轮）或其 `type` 为 `previous_message_not_found` 或 `unavailable`（未产生比较结果）时，此矩阵也不适用。

| 诊断结果                                 | 缓存读取令牌数 | 解读                                                                                                                                              |
| ------------------------------------ | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `null`                               | 高       | 按预期工作。您的前缀稳定且缓存命中。                                                                                                                              |
| `null`                               | 低或为零    | 您的请求匹配，但缓存条目已不再可用。请考虑缩短轮次之间的间隔，或使用 [1 小时缓存 TTL](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching#1-hour-cache-duration)。 |
| `cache_miss_reason` 为 `*_changed` 类型 | 低或为零    | 您的 bug。请求发生了变化；请修复 `type` 所指示的原因。                                                                                                               |
| `cache_miss_reason` 为 `*_changed` 类型 | 高       | 罕见。变化发生在提示的后部，但较早的 `cache_control` 断点仍然命中。值得修复，但影响较小。                                                                                           |

## 限制

* **Beta：** 在此功能处于 beta 阶段期间，字段名称和语义可能会发生变化。
* **仅限 Claude API：** 在 Amazon Bedrock 或 Google Cloud 上不可用。
* **有限的保留期：** 用于 `previous_message_id` 查找的指纹会在短时间后过期。请在时间间隔较近的请求之间运行诊断比较。
* **同一工作区：** 上一个请求必须在同一组织和工作区中运行。要进行检查，请比较两个响应上的 `anthropic-workspace-id` [响应头](https://platform.claude.com/docs/zh-CN/api/overview#response-headers)。
* **比较范围：** 对于唯一变化位于消息列表深处的超长对话，响应可能是 `unavailable` 而非精确位置。
* **尽力而为：** 诊断绝不会阻塞您的请求或使其失败。如果诊断信息不可用，响应会返回 `unavailable`；如果比较仍在运行，则返回 `cache_miss_reason: null`。

## 数据保留

缓存诊断符合 ZDR 资格（有条件）。Anthropic 不会为此功能存储您的提示或 Claude 输出的原始文本。

为每个请求存储的指纹仅由加密哈希值和令牌数量估计值组成，以响应 `id` 作为键，作用域限定在您的组织和工作区内。指纹会在短时间后过期，并且不会用于任何其他目的。

有关所有功能的 ZDR 资格，请参阅 [API 和数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention)。

## 另请参阅

* [提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)
* [令牌计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)
* [Beta 请求头](https://platform.claude.com/docs/zh-CN/api/beta-headers)
