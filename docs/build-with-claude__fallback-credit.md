---
title: 回退抵扣
url: https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit
description: 当您在另一个模型上重试被拒绝的请求时，避免重复支付提示缓存费用。
---

提示缓存是按模型划分的。当某个模型拒绝了一个请求，而您在另一个模型上重试时，已为第一个模型缓存的对话前缀必须从头写入新模型的缓存。缓存写入的费用高于缓存读取。"Fallback credit"（回退抵扣）消除了这部分额外费用。拒绝响应会携带一个抵扣令牌，您在重试时回传该令牌，重试的计费方式就如同该对话一直在新模型上进行一样。

只有当您自行构建重试逻辑时才需要阅读本页：通过原始 HTTP 或使用自定义重试逻辑。[服务端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)和 [SDK 中间件](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#client-side-fallback)会自动应用回退抵扣。如果您使用其中任一种，请跳过本页。

[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)介绍了如何检测拒绝以及如何选择回退方式。如果您对缓存读取和缓存写入这些术语不熟悉，[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)对其进行了解释。

## 基本流程

<Steps>
  <Step title="通过 beta 标头选择启用">
    发送可能被拒绝的请求时，附带 `anthropic-beta: fallback-credit-2026-07-01` 标头。`server-side-fallback-2026-07-01` 标头也会提供相同的字段，较早的 `fallback-credit-2026-06-01` 标头仍然被接受并提供相同的字段。
  </Step>

  <Step title="从拒绝响应中读取两个字段">
    发生拒绝时，`stop_details` 包含两个字段：

    * **`fallback_credit_token`：** 一个表示抵扣的不透明字符串。
    * **`fallback_has_prefill_claim`：** 一个布尔值，告诉您应使用哪种重试请求体形态。

    当该拒绝没有可用抵扣时，两者均为 `null`。
  </Step>

  <Step title="构建重试请求">
    从被拒绝的请求体开始。将 `model` 设置为回退模型，并将令牌作为顶层 `fallback_credit_token` 参数添加。根据下表选择请求体形态。
  </Step>

  <Step title="使用相同的标头发送重试">
    使用相同的 `fallback-credit-2026-07-01` beta 标头发送重试。重试需要该标头才能兑换令牌。
  </Step>
</Steps>

`fallback_has_prefill_claim` 字段告诉您重试是否可以接续被拒绝模型的部分输出，而不是从头开始：

| `fallback_has_prefill_claim` | 重试请求体                                                                                                   |
| ---------------------------- | ------------------------------------------------------------------------------------------------------- |
| `true`                       | 被拒绝的请求体保持不变，再追加一条 assistant 消息，其 `content` 回传被拒绝响应的 `content`。重试模型从被拒绝模型停止的位置继续生成响应，已完成的服务端工具调用不会被重新执行。 |
| `false`                      | 被拒绝的请求体，保持不变。                                                                                           |

## 示例

以下示例发出一个可能被拒绝的请求，并在针对 Claude Opus 4.8 的重试中兑换抵扣令牌。当某次重试尝试被拒绝时，该示例会沿着拒绝阶梯逐级降级：即[当重试被拒绝时](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit#when-a-retry-is-rejected)中介绍的一系列逐步简化的重试形态。

<CodeGroup>
  ```bash cURL
  # 初始请求（可能被拒绝）
  response=$(curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: fallback-credit-2026-07-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-fable-5",
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello, Claude"}]
    }')

  # 拒绝响应会在 stop_details 中携带一个一次性信用令牌
  token=$(jq -r '.stop_details.fallback_credit_token // empty' <<<"${response}")

  if [[ -n "${token}" ]]; then
    # 使用该信用令牌在备用模型上重试（请求体相同）
    response=$(curl --fail-with-body -sS https://api.anthropic.com/v1/messages \
      -H "x-api-key: $ANTHROPIC_API_KEY" \
      -H "anthropic-version: 2023-06-01" \
      -H "anthropic-beta: fallback-credit-2026-07-01" \
      -H "content-type: application/json" \
      -d "$(jq -n --arg token "${token}" '{
        model: "claude-opus-4-8",
        max_tokens: 1024,
        messages: [{"role": "user", "content": "Hello, Claude"}],
        fallback_credit_token: $token
      }')")
  fi

  # 完整的拒绝处理阶梯请参阅 SDK 示例。
  jq -c '{stop_reason, model}' <<<"${response}"
  ```

  ```bash CLI
  # 初始请求（可能被拒绝）
  response=$(ant beta:messages create \
    --model claude-fable-5 \
    --max-tokens 1024 \
    --message '{"role":"user","content":"Hello, Claude"}' \
    --beta fallback-credit-2026-07-01 \
    --format json)

  # 拒绝响应会在 stop_details 中携带一次性额度令牌
  token=$(jq -r '.stop_details.fallback_credit_token // empty' <<<"${response}")

  if [[ -n "${token}" ]]; then
    # 使用该额度令牌在备用模型上重试
    response=$(ant beta:messages create \
      --model claude-opus-4-8 \
      --max-tokens 1024 \
      --message '{"role":"user","content":"Hello, Claude"}' \
      --fallback-credit-token "${token}" \
      --beta fallback-credit-2026-07-01 \
      --format json)
  fi

  # 完整的拒绝处理流程请参阅 SDK 示例。
  jq -c '{stop_reason, model}' <<<"${response}"
  ```

  ```python Python
  client = Anthropic()

  request = {
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello, Claude"}],
  }


  def send(model: str, body: dict[str, object]) -> BetaMessage:
      return client.beta.messages.create(
          model=model, betas=["fallback-credit-2026-07-01"], **body
      )


  response = send("claude-fable-5", request)

  if (
      response.stop_reason == "refusal"
      and (details := response.stop_details)
      and (token := details.fallback_credit_token)
  ):
      exact_body = request | {"fallback_credit_token": token}
      # 优先使用 continuation 形式，除非 claim 为 False
      if details.fallback_has_prefill_claim is not False:
          echoed = [block.model_dump() for block in response.content]
          match echoed:
              case [*_, {"type": "text"} as final_block]:
                  final_block["text"] = final_block["text"].rstrip()
          attempt = exact_body | {
              "messages": [
                  *request["messages"],
                  {"role": "assistant", "content": echoed},
              ]
          }
      else:
          attempt = exact_body

      try:
          response = send("claude-opus-4-8", attempt)
      except BadRequestError as error:
          if "redemption temporarily unavailable" in error.message:
              raise  # Transient: retry with the token within its five-minute window
          try:
              # 回退到未更改的 body，仍携带令牌
              response = send("claude-opus-4-8", exact_body)
          except BadRequestError as retry_error:
              if "redemption temporarily unavailable" in retry_error.message:
                  raise  # Transient: retry with the token within its five-minute window
              # 令牌本身被拒绝：放弃该令牌并在不携带它的情况下重试。
              response = send("claude-opus-4-8", request)

  print(json.dumps({"stop_reason": response.stop_reason, "model": response.model}))
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const request: Anthropic.Beta.MessageCreateParamsNonStreaming = {
    model: "claude-fable-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    betas: ["fallback-credit-2026-07-01"]
  };

  let response = await client.beta.messages.create(request);

  if (
    response.stop_reason === "refusal" &&
    response.stop_details?.type === "refusal" &&
    response.stop_details.fallback_credit_token
  ) {
    const { fallback_credit_token, fallback_has_prefill_claim } = response.stop_details;
    const fallbackModel = "claude-opus-4-8";

    const exactRetry: Anthropic.Beta.MessageCreateParamsNonStreaming = {
      ...request,
      model: fallbackModel,
      fallback_credit_token
    };

    // 优先采用最丰富的形态，每次被拒绝后逐级降级：先是 continuation
    // 形态（除非该声明为假），然后是仍携带令牌的未更改请求体，
    // 最后是放弃该令牌。
    let attempt = exactRetry;
    if (fallback_has_prefill_claim !== false) {
      const finalBlock = response.content.at(-1);
      const echoed: Anthropic.Beta.BetaContentBlockParam[] =
        finalBlock?.type === "text"
          ? [
              ...response.content.slice(0, -1),
              { ...finalBlock, text: finalBlock.text.trimEnd() }
            ]
          : response.content;
      attempt = {
        ...exactRetry,
        messages: [...request.messages, { role: "assistant", content: echoed }]
      };
    }

    try {
      response = await client.beta.messages.create(attempt);
    } catch (error) {
      // 仅在与形态相关的 400 错误时降级。"redemption temporarily
      // unavailable" 是暂时性错误：应在令牌的
      // 五分钟窗口内以相同方式重试。
      if (
        !(error instanceof Anthropic.BadRequestError) ||
        error.message.includes("redemption temporarily unavailable")
      ) {
        throw error;
      }
      try {
        response = await client.beta.messages.create(exactRetry);
      } catch (retryError) {
        if (
          !(retryError instanceof Anthropic.BadRequestError) ||
          retryError.message.includes("redemption temporarily unavailable")
        ) {
          throw retryError;
        }
        response = await client.beta.messages.create({ ...request, model: fallbackModel });
      }
    }
  }

  const { stop_reason, model } = response;
  console.log(JSON.stringify({ stop_reason, model }));
  ```

  ```csharp C#
  var client = new AnthropicClient();
  const string beta = "fallback-credit-2026-07-01";

  List<BetaMessageParam> requestMessages =
  [
      new() { Role = Role.User, Content = "Hello, Claude" },
  ];
  MessageCreateParams Request(string model) => new()
  {
      Model = model,
      MaxTokens = 1024,
      Messages = requestMessages,
      Betas = [beta],
  };
  var response = await client.Beta.Messages.Create(Request("claude-fable-5"));

  if (
      response.StopReason == BetaStopReason.Refusal
      && response.StopDetails is { FallbackCreditToken: string token } details
  )
  {
      var exactBody = Request("claude-opus-4-8") with { FallbackCreditToken = token };
      var attempt = exactBody;
      // 除非声明为假，否则优先采用延续形式
      if (details.FallbackHasPrefillClaim is not false)
      {
          var echoed = JsonArray.Create(response.RawData["content"])!;
          if (
              echoed is [.., JsonObject lastBlock]
              && lastBlock["type"]?.GetValue<string>() is "text"
              && lastBlock["text"]?.GetValue<string>() is string text
          )
          {
              lastBlock["text"] = text.TrimEnd();
          }
          attempt = exactBody with
          {
              Messages =
              [
                  .. requestMessages,
                  new()
                  {
                      Role = Role.Assistant,
                      Content = new BetaMessageParamContent(
                          JsonSerializer.SerializeToElement(echoed)
                      ),
                  },
              ],
          };
      }
      // 瞬时的“兑换暂时不可用”拒绝会从以下
      // 每个 catch 过滤器中传播出去：在令牌的五分钟窗口内携带该令牌重试。
      try
      {
          response = await client.Beta.Messages.Create(attempt);
      }
      catch (AnthropicBadRequestException e)
          when (!e.Message.Contains("redemption temporarily unavailable"))
      {
          try
          {
              // 回退到未更改的请求体，仍携带令牌
              response = await client.Beta.Messages.Create(exactBody);
          }
          catch (AnthropicBadRequestException retryError)
              when (!retryError.Message.Contains("redemption temporarily unavailable"))
          {
              // 令牌本身被拒绝：放弃该令牌并在不携带它的情况下重试。
              response = await client.Beta.Messages.Create(Request("claude-opus-4-8"));
          }
      }
  }

  Console.WriteLine(
      JsonSerializer.Serialize(
          new { stop_reason = response.StopReason?.Raw(), model = response.Model.Raw() }
      )
  );
  ```

  ```go Go
  ctx := context.Background()
  client := anthropic.NewClient()

  request := anthropic.BetaMessageNewParams{
  	MaxTokens: 1024,
  	Betas:     []anthropic.AnthropicBeta{anthropic.AnthropicBetaFallbackCredit2026_07_01},
  	Messages: []anthropic.BetaMessageParam{
  		anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Hello, Claude")),
  	},
  }

  send := func(model anthropic.Model, body anthropic.BetaMessageNewParams) (*anthropic.BetaMessage, error) {
  	body.Model = model
  	return client.Beta.Messages.New(ctx, body)
  }
  // 非瞬时的 400 表示此尝试形态或令牌被拒绝，
  // 应运行阶梯的下一级。"redemption temporarily
  // unavailable" 是瞬时的：将其呈现出来，并在令牌的
  // 五分钟窗口内使用该令牌重试。
  canFallBack := func(err error) bool {
  	apiErr, ok := errors.AsType[*anthropic.Error](err)
  	return ok && apiErr.StatusCode == 400 &&
  		!strings.Contains(apiErr.Error(), "redemption temporarily unavailable")
  }

  response, err := send(anthropic.ModelClaudeFable5, request)
  if err != nil {
  	log.Fatal(err)
  }

  if response.StopReason == anthropic.BetaStopReasonRefusal {
  	details := response.StopDetails
  	if token := details.FallbackCreditToken; token != "" {
  		exactBody := request
  		exactBody.FallbackCreditToken = anthropic.BetaMessageNewParamsFallbackCreditTokenUnion{
  			OfString: anthropic.String(token),
  		}
  		attempt := exactBody
  		// 除非声明为假，否则优先使用延续形态
  		if details.FallbackHasPrefillClaim || !details.JSON.FallbackHasPrefillClaim.Valid() {
  			echoed := response.ToParam()
  			if len(echoed.Content) > 0 {
  				if text := echoed.Content[len(echoed.Content)-1].OfText; text != nil {
  					text.Text = strings.TrimRightFunc(text.Text, unicode.IsSpace)
  				}
  			}
  			attempt.Messages = append(slices.Clone(request.Messages), echoed)
  		}
  		response, err = send(anthropic.ModelClaudeOpus4_8, attempt)
  		if err != nil && canFallBack(err) {
  			// 回退到未更改的请求体，仍携带令牌
  			response, err = send(anthropic.ModelClaudeOpus4_8, exactBody)
  			if err != nil && canFallBack(err) {
  				// 令牌本身被拒绝：放弃它并在不带令牌的情况下重试。
  				response, err = send(anthropic.ModelClaudeOpus4_8, request)
  			}
  		}
  		if err != nil {
  			log.Fatal(err)
  		}
  	}
  }

  summary, err := json.Marshal(struct {
  	StopReason anthropic.BetaStopReason `json:"stop_reason"`
  	Model      anthropic.Model          `json:"model"`
  }{response.StopReason, response.Model})
  if err != nil {
  	log.Fatal(err)
  }
  fmt.Println(string(summary))
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  MessageCreateParams.Builder request() {
      return MessageCreateParams.builder()
          .maxTokens(1024L)
          .addUserMessage("Hello, Claude")
          .addBeta(AnthropicBeta.FALLBACK_CREDIT_2026_07_01);
  }

  BetaMessage send(Model model, MessageCreateParams.Builder body) {
      return client.beta().messages().create(body.model(model).build());
  }

  void main() {
      BetaMessage response = send(Model.CLAUDE_FABLE_5, request());

      if (response.stopReason().map(BetaStopReason.REFUSAL::equals).orElse(false)
              && response.stopDetails().orElse(null) instanceof BetaRefusalStopDetails details
              && details.fallbackCreditToken().orElse(null) instanceof String creditToken) {
          MessageCreateParams.Builder attempt = request().fallbackCreditToken(creditToken);
          // 除非该声明为假，否则优先采用延续形式
          if (details.fallbackHasPrefillClaim().orElse(true)) {
              List<BetaContentBlockParam> echoed = new ArrayList<>(
                  response.content().stream().map(BetaContentBlock::toParam).toList());
              if (!echoed.isEmpty() && echoed.getLast().isText()) {
                  var lastText = echoed.removeLast().asText();
                  echoed.addLast(BetaContentBlockParam.ofText(
                      lastText.toBuilder().text(lastText.text().stripTrailing()).build()));
              }
              attempt.addAssistantMessageOfBetaContentBlockParams(echoed);
          }
          try {
              response = send(Model.CLAUDE_OPUS_4_8, attempt);
          } catch (BadRequestException badRequest) {
              // 瞬时错误：在令牌的五分钟有效窗口内携带令牌重试
              if (badRequest.getMessage().contains("redemption temporarily unavailable")) {
                  throw badRequest;
              }
              try {
                  // 回退到未更改的请求体，仍携带令牌
                  response = send(Model.CLAUDE_OPUS_4_8, request().fallbackCreditToken(creditToken));
              } catch (BadRequestException retryBadRequest) {
                  if (retryBadRequest.getMessage().contains("redemption temporarily unavailable")) {
                      throw retryBadRequest;
                  }
                  // 令牌本身被拒绝：放弃该令牌并在不携带它的情况下重试。
                  response = send(Model.CLAUDE_OPUS_4_8, request());
              }
          }
      }

      IO.println("""
          {"stop_reason": "%s", "model": "%s"}"""
          .formatted(response.stopReason().orElseThrow(), response.model()));
  }
  ```

  ```php PHP
  $client = new Client();
  $beta = 'fallback-credit-2026-07-01';
  $messages = [['role' => 'user', 'content' => 'Hello, Claude']];

  $send = fn (string $model, array $messages, ?string $token = null) => $client->beta->messages->create(
      maxTokens: 1024,
      messages: $messages,
      model: $model,
      fallbackCreditToken: $token,
      betas: [$beta],
  );
  $response = $send('claude-fable-5', $messages);

  $token = $response->stopReason === 'refusal'
      ? $response->stopDetails?->fallbackCreditToken
      : null;

  if ($token !== null) {
      $attemptMessages = $messages;
      // 除非声明为假，否则优先使用延续形式
      if ($response->stopDetails->fallbackHasPrefillClaim !== false) {
          $echoed = $response->content
              |> json_encode(...)
              |> (fn (string $json): array => json_decode($json, associative: true));
          $lastIndex = array_key_last($echoed);
          if ($lastIndex !== null && $echoed[$lastIndex]['type'] === 'text') {
              $echoed[$lastIndex]['text'] = rtrim($echoed[$lastIndex]['text']);
          }
          $attemptMessages[] = ['role' => 'assistant', 'content' => $echoed];
      }
      // 瞬时错误：在令牌的五分钟有效窗口内携带令牌重试
      $isTransientRedemption = fn (BadRequestException $error): bool =>
          str_contains($error->getMessage(), 'redemption temporarily unavailable');
      try {
          $response = $send('claude-opus-4-8', $attemptMessages, $token);
      } catch (BadRequestException $error) {
          if ($isTransientRedemption($error)) {
              throw $error;
          }
          try {
              // 回退到未更改的请求体，仍携带令牌
              $response = $send('claude-opus-4-8', $messages, $token);
          } catch (BadRequestException $retryError) {
              if ($isTransientRedemption($retryError)) {
                  throw $retryError;
              }
              // 令牌本身被拒绝：放弃该令牌并在不携带它的情况下重试。
              $response = $send('claude-opus-4-8', $messages);
          }
      }
  }

  echo json_encode(['stop_reason' => $response->stopReason, 'model' => $response->model]), PHP_EOL;
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  request = {
    max_tokens: 1024,
    messages: [{role: "user", content: "Hello, Claude"}]
  }

  send_message = ->(model, body) do
    client.beta.messages.create(model:, betas: ["fallback-credit-2026-07-01"], **body)
  end

  response = send_message.call("claude-fable-5", request)

  if response in {stop_reason: :refusal,
                  stop_details: {fallback_credit_token: String => credit_token} => details}
    exact_body = request.merge(fallback_credit_token: credit_token)

    # 除非声明为假，否则优先使用延续形式
    attempt = if details.fallback_has_prefill_claim != false
      echoed = response.content.map(&:to_h)
      if echoed.last in {type: :text, text: String => final_text}
        echoed[-1] = echoed.last.merge(text: final_text.rstrip)
      end
      exact_body.merge(
        messages: [*request[:messages], {role: "assistant", content: echoed}]
      )
    else
      exact_body
    end

    begin
      response = send_message.call("claude-opus-4-8", attempt)
    rescue Anthropic::Errors::BadRequestError => error
      # 瞬时错误：在令牌的五分钟有效期内携带令牌重试
      raise if error.message.include?("redemption temporarily unavailable")
      begin
        # 回退到未更改的请求体，仍携带令牌
        response = send_message.call("claude-opus-4-8", exact_body)
      rescue Anthropic::Errors::BadRequestError => error
        # 瞬时错误：在令牌的五分钟有效期内携带令牌重试
        raise if error.message.include?("redemption temporarily unavailable")
        # 令牌本身被拒绝：放弃该令牌并在不携带它的情况下重试。
        response = send_message.call("claude-opus-4-8", request)
      end
    end
  end

  puts JSON.generate({stop_reason: response.stop_reason, model: response.model})
  ```
</CodeGroup>

## 适用范围

回退抵扣在 Claude API、Amazon Bedrock、Claude Platform on AWS、Google Cloud 和 Microsoft Foundry 上处于 beta 阶段。[Message Batches](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing) 中的拒绝不会生成抵扣令牌，且兑换仅适用于直接的 Messages API 请求：在批处理请求中传递的令牌会被接受但会被忽略。

重试模型必须是被拒绝模型所允许的回退目标之一。对于 Claude Fable 5.1 和 Claude Fable 5，允许的目标是 Claude Opus 4.8（`claude-opus-4-8`）和 Claude Opus 5（`claude-opus-5`）。

<Accordion title="以编程方式查询允许的回退目标">
  在 Claude API 和 Claude Platform on AWS 上，当设置了 `server-side-fallback-2026-07-01` beta 标头时，目标列表会作为 `allowed_fallback_models` 发布在 [Models API](https://platform.claude.com/docs/zh-CN/api/models/list) 中每个模型的条目上。仅使用 `fallback-credit-*` 标头时，该列表尚不可见。它在 Amazon Bedrock、Google Cloud 或 Microsoft Foundry 上不对外公开。
</Accordion>

## 检查抵扣是否已生效

退款体现在重试的 `usage` 中。与同一请求在不带令牌时报告的数值相比，`cache_creation_input_tokens` 更低，而 `cache_read_input_tokens` 则高出相同的数量。变化为零意味着令牌已被接受，但没有需要重新定价的内容，例如因为重试模型的缓存已经是热的。

## 当重试被拒绝时

大多数重试在第一次尝试时即可兑换。如果未能兑换，API 会返回一个 400 错误，告诉您接下来该尝试什么。

<Steps>
  <Step title="接续被拒绝：重新发送未更改的请求体">
    如果追加了 assistant 消息的重试被 400 错误拒绝，请重新发送未更改的被拒绝请求体，仍然附带令牌。
  </Step>

  <Step title="令牌被拒绝：去掉令牌">
    如果未更改的请求体也被 400 错误拒绝，且错误消息中提到了 `fallback_credit_token`，请在不带令牌的情况下重试。抵扣将被放弃，但重试本身可以成功。
  </Step>
</Steps>

<Note>
  如果被拒绝的请求执行了服务端工具，不带令牌的重试会重新运行这些工具并重新计费。在这种情况下，请将 400 错误呈现给您的调用方，而不是降级到不带令牌的重试。
</Note>

<Accordion title="如果错误提示 'redemption temporarily unavailable'">
  这种拒绝是暂时性的，并非对您的重试形态的判定。请在令牌的五分钟有效期内，使用相同的令牌重试相同的请求。不要进入阶梯的下一步。
</Accordion>

## 参考

以下各节介绍边缘情况和完整的兑换规则。大多数集成不需要这些内容。

<Accordion title="必须与被拒绝请求匹配的字段">
  兑换时会将重试与被拒绝的请求进行比较。每个影响提示构成的字段都必须完全匹配。不影响提示构成的字段可以在重试时更改。

  | 规则       | 字段                                                                                                                                               |
  | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
  | 必须完全匹配   | `system`、`messages`、`tools`、`tool_choice`、`thinking` 和 `cache_control`，以及在您使用时的 `output_config`、`mcp_servers`、`context_management` 和 `container` |
  | 可以在重试时更改 | `model`、`max_tokens`、`stop_sequences`、`temperature`、`top_p`、`top_k`、`stream`、`metadata` 和 `service_tier`                                         |

  接续形态（`fallback_has_prefill_claim: true`）是 `messages` 匹配规则的唯一例外：它在 `messages` 末尾恰好添加一条 assistant 消息。

  不要在重试时从较早的轮次中剥离 `thinking` 或 `redacted_thinking` 块，即使不带令牌的普通重试通常会剥离它们。请求体必须与被拒绝的请求匹配，服务器会自行处理这些块。
</Accordion>

<Accordion title="Beta 标头也必须匹配">
  在重试时发送与被拒绝请求相同的 `anthropic-beta` 标头。某个 beta 标头出现在两个请求之一而未出现在另一个请求中，即使请求体完全相同，也可能导致匹配失败。由此产生的 400 错误携带与请求体差异相同的 `request body ... does not match` 消息，因此标头差异很容易被误读为请求体问题。特别是，不要根据请求所针对的模型来添加或删除 beta 标头。

  为了重试的需要，有两类标头不受匹配约束：

  * **`server-side-fallback-*`：** 重试必须去掉 `fallbacks` 参数，随之去掉此标头不会导致不匹配。
  * **`fallback-credit-*`：** 在两个请求上都保留此标头。重试需要它来兑换令牌。

  <Note>
    在默认包含 1M 令牌上下文窗口的模型上，例如 Claude Fable 5.1、Claude Fable 5、Claude Opus 5 和 Claude Opus 4.8，`context-1m-2025-08-07` beta 标头没有任何作用。为了保持两个请求完全相同，请在两个请求上都省略该标头，而不是在一个请求上发送而在另一个请求上不发送。
  </Note>
</Accordion>

<Accordion title="当 fallback_has_prefill_claim 缺失时">
  该字段仅在令牌也为 `null` 时才为 `null`，因此当您持有令牌时观察到的值永远不会是 `null`。在 Amazon Bedrock、Google Cloud 和 Microsoft Foundry 对该字段的支持逐步推出期间，它仍可能缺失（在类型化 SDK 中为 `None`）。在这种情况下，请将重试形态视为未知，而不是视为 `false`。先尝试追加 assistant 消息的形态，并依赖[当重试被拒绝时](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit#when-a-retry-is-rejected)中的拒绝处理逻辑，它会回退到未更改的请求体。
</Accordion>

<Accordion title="回传被拒绝响应的 content">
  当拒绝的令牌支持接续形态时，响应的 `content` 仅携带模型自身的输出，拒绝说明则通过 `stop_details.explanation` 传递。因此，您可以将 `content` 原样回传到追加的 assistant 消息中。

  发送前可能仍需要进行两项调整：

  * 如果您发送的最后一个块是 `text` 块，请去除其尾部空白。
  * 省略任何没有匹配 `tool_result` 的客户端 `tool_use` 块。

  如果回传的内容包含来自较早[服务端回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback#server-side-fallback)的 `fallback` 块，请将该块保留在其原来出现的确切位置。它在任何请求中都会被接受，无需 beta 标头。API 使用其位置来验证其周围的 thinking 块，因此如果该块被省略或移动，回传了该边界两侧 thinking 块的请求将被拒绝。
</Accordion>

<Accordion title="令牌的作用域与有效期">
  令牌只能由收到拒绝的组织和工作区兑换，包括在 Microsoft Foundry 上。在没有工作区的 Amazon Bedrock 和 Google Cloud 上，令牌则绑定到平台的调用方身份。

  令牌在拒绝发生五分钟后过期。过期后，请在不带令牌的情况下发送重试。令牌也是无状态的：服务器不存储任何与其相关的信息，也没有用于检查或撤销它的端点。
</Accordion>

<Accordion title="当令牌无法通过任一形态兑换时">
  当拒绝发生在请求内服务端工具已经执行之后时，令牌只能通过接续部分响应来兑换。正是这一限制防止了已完成的工具调用再次运行和再次计费。

  因此，当以下两个条件同时成立时，有一种组合会使令牌无法通过任一形态兑换：

  * 请求使用了 `output_config.format` 或强制工具使用的 `tool_choice`。其中任一项都会排除追加 assistant 消息的形态。
  * 拒绝发生在服务端工具执行之后。这排除了未更改的请求体。

  如果未更改请求体的重试被 400 错误拒绝，且错误提示令牌必须通过接续部分响应来兑换，请丢弃该令牌。不带令牌的重试可以成功，但它会重新运行已完成的服务端工具并重新计费。请将费用或错误呈现给您的调用方，而不是静默重试。
</Accordion>

## 后续步骤

<CardGroup>
  <Card title="拒绝与回退" icon="shield" href="https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback">
    检测拒绝，并在服务端回退、SDK 中间件和手动重试之间进行选择。
  </Card>

  <Card title="提示缓存" icon="bolt" href="https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching">
    缓存读取和缓存写入的计费方式。
  </Card>

  <Card title="停止原因与回退" icon="code" href="https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons">
    每个 `stop_reason` 值及其处理方式。
  </Card>

  <Card title="SDK 中间件" icon="settings" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/middleware">
    自动应用回退抵扣的 SDK 辅助工具。
  </Card>
</CardGroup>
