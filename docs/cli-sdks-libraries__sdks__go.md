---
title: Go SDK
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/go
description: 安装并配置 Anthropic Go SDK，支持基于 context 的取消和函数式选项
---

Anthropic Go 库为使用 Go 编写的应用程序提供了便捷访问 Claude API 的方式。

<Info>
  有关带代码示例的 API 功能文档，请参阅 [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)。本页面涵盖 Go 特有的 SDK 功能和配置。
</Info>

## 安装

```go
import (
	"github.com/anthropics/anthropic-sdk-go" // imported as anthropic
)
```

使用 `go get` 安装：

```bash
go get github.com/anthropics/anthropic-sdk-go
```

## 要求

此库需要 Go 1.24+。

## 用法

```go
package main

import (
	"context"
	"fmt"

	"github.com/anthropics/anthropic-sdk-go"
	"github.com/anthropics/anthropic-sdk-go/option"
)

func main() {
	client := anthropic.NewClient(
		option.WithAPIKey("my-anthropic-api-key"), // defaults to os.LookupEnv("ANTHROPIC_API_KEY")
	)
	message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
		MaxTokens: 1024,
		Messages: []anthropic.MessageParam{
			anthropic.NewUserMessage(anthropic.NewTextBlock("What is a quaternion?")),
		},
		Model: anthropic.ModelClaudeOpus5,
	})
	if err != nil {
		panic(err.Error())
	}
	for _, block := range message.Content {
		if textBlock, ok := block.AsAny().(anthropic.TextBlock); ok {
			fmt.Println(textBlock.Text)
		}
	}
}
```

有关包括 Workload Identity Federation（工作负载身份联合）在内的身份验证选项，请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)。如果您的 API 密钥是可访问多个工作区的[个人密钥或服务账户密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)，请在 `anthropic-workspace-id` 请求头中设置工作区 ID；[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)展示了此 SDK 的按请求设置选项。

<AccordionGroup>
  <Accordion title="对话">
    ```go
    messages := []anthropic.MessageParam{
    	anthropic.NewUserMessage(anthropic.NewTextBlock("What is my first name?")),
    }

    message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
    	Model:     anthropic.ModelClaudeOpus5,
    	Messages:  messages,
    	MaxTokens: 1024,
    })
    if err != nil {
    	panic(err)
    }

    fmt.Printf("%+v\n", message.Content)

    messages = append(messages, message.ToParam())
    messages = append(messages, anthropic.NewUserMessage(
    	anthropic.NewTextBlock("My full name is John Doe"),
    ))

    message, err = client.Messages.New(context.TODO(), anthropic.MessageNewParams{
    	Model:     anthropic.ModelClaudeOpus5,
    	Messages:  messages,
    	MaxTokens: 1024,
    })
    if err != nil {
    	panic(err)
    }

    fmt.Printf("%+v\n", message.Content)
    ```
  </Accordion>

  <Accordion title="系统提示">
    ```go
    message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
    	Model:     anthropic.ModelClaudeOpus5,
    	MaxTokens: 1024,
    	System: []anthropic.TextBlockParam{
    		{Text: "Be very serious at all times."},
    	},
    	Messages: messages,
    })
    if err != nil {
    	panic(err)
    }
    fmt.Printf("%+v\n", message.Content)
    ```
  </Accordion>

  <Accordion title="流式传输">
    ```go
    content := "What is a quaternion?"

    stream := client.Messages.NewStreaming(context.TODO(), anthropic.MessageNewParams{
    	Model:     anthropic.ModelClaudeOpus5,
    	MaxTokens: 1024,
    	Messages: []anthropic.MessageParam{
    		anthropic.NewUserMessage(anthropic.NewTextBlock(content)),
    	},
    })

    message := anthropic.Message{}
    for stream.Next() {
    	event := stream.Current()
    	err := message.Accumulate(event)
    	if err != nil {
    		panic(err)
    	}

    	switch eventVariant := event.AsAny().(type) {
    	case anthropic.ContentBlockDeltaEvent:
    		switch deltaVariant := eventVariant.Delta.AsAny().(type) {
    		case anthropic.TextDelta:
    			print(deltaVariant.Text)
    		}

    	}
    }

    if stream.Err() != nil {
    	panic(stream.Err())
    }
    ```
  </Accordion>

  <Accordion title="工具调用">
    ```go
    messages := []anthropic.MessageParam{
    	anthropic.NewUserMessage(anthropic.NewTextBlock(content)),
    }

    toolParams := []anthropic.ToolParam{
    	{
    		Name:        "get_coordinates",
    		Description: anthropic.String("Accepts a place as an address, then returns the latitude and longitude coordinates."),
    		InputSchema: GetCoordinatesInputSchema,
    	},
    }
    tools := make([]anthropic.ToolUnionParam, len(toolParams))
    for i, toolParam := range toolParams {
    	tools[i] = anthropic.ToolUnionParam{OfTool: &toolParam}
    }

    for {
    	message, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
    		Model:     anthropic.ModelClaudeOpus5,
    		MaxTokens: 1024,
    		Messages:  messages,
    		Tools:     tools,
    	})

    	if err != nil {
    		panic(err)
    	}

    	print(color("[assistant]: "))
    	for _, block := range message.Content {
    		switch block := block.AsAny().(type) {
    		case anthropic.TextBlock:
    			println(block.Text)
    			println()
    		case anthropic.ToolUseBlock:
    			inputJSON, _ := json.Marshal(block.Input)
    			println(block.Name + ": " + string(inputJSON))
    			println()
    		}
    	}

    	messages = append(messages, message.ToParam())
    	toolResults := []anthropic.ContentBlockParamUnion{}

    	for _, block := range message.Content {
    		switch variant := block.AsAny().(type) {
    		case anthropic.ToolUseBlock:
    			print(color("[user (" + block.Name + ")]: "))

    			var response interface{}
    			switch block.Name {
    			case "get_coordinates":
    				var input struct {
    					Location string `json:"location"`
    				}

    				err := json.Unmarshal([]byte(variant.JSON.Input.Raw()), &input)
    				if err != nil {
    					panic(err)
    				}

    				response = GetCoordinates(input.Location)
    			}

    			b, err := json.Marshal(response)
    			if err != nil {
    				panic(err)
    			}

    			println(string(b))

    			toolResults = append(toolResults, anthropic.NewToolResultBlock(block.ID, string(b), false))
    		}

    	}
    	if len(toolResults) == 0 {
    		break
    	}
    	messages = append(messages, anthropic.NewUserMessage(toolResults...))
    }
    ```
  </Accordion>
</AccordionGroup>

## 请求字段

anthropic 库对请求字段使用 Go 1.24+ `encoding/json` 版本中的 [`omitzero`](https://tip.golang.org/doc/go1.24#encodingjsonpkgencodingjson) 语义。

必需的基本类型字段（例如 `int64` 或 `string`）带有标签 `` `json:"...,required"` ``。这些字段始终会被序列化，即使是它们的零值。

可选的基本类型被包装在 `param.Opt[T]` 中。这些字段可以使用提供的构造函数进行设置，例如 `anthropic.String(string)` 或 `anthropic.Int(int64)`。

任何 `param.Opt[T]`、map、slice、struct 或字符串枚举都使用标签 `` `json:"...,omitzero"` ``。其零值被视为已省略。

`param.IsOmitted(any)` 函数可以确认任何 `omitzero` 字段是否存在。

```go
p := anthropic.ExampleParams{
	ID:   "id_xxx",                // required property
	Name: anthropic.String("..."), // optional property

	Point: anthropic.Point{
		X: 0,                // required field will serialize as 0
		Y: anthropic.Int(1), // optional field will serialize as 1
		// ... 省略的非必填字段将不会被序列化
	},

	Origin: anthropic.Origin{}, // the zero value of [Origin] is considered omitted
}
```

要发送 `null` 而不是 `param.Opt[T]`，请使用 `param.Null[T]()`。 要发送 `null` 而不是结构体 `T`，请使用 `param.NullStruct[T]()`。

```go
p.Name = param.Null[string]()       // 'null' instead of string
p.Point = param.NullStruct[Point]() // 'null' instead of struct

param.IsNull(p.Name)  // true
param.IsNull(p.Point) // true
```

请求结构体包含一个 `.SetExtraFields(map[string]any)` 方法，可以在请求体中发送不符合规范的字段。额外字段会覆盖任何具有匹配键的结构体字段。

<Warning>
  出于安全原因，请仅对可信数据使用 `SetExtraFields`。
</Warning>

要发送自定义值而不是结构体，请使用泛型函数 `param.Override`（例如 `param.Override[anthropic.FooParams](12)`）。

```go
// 如果 API 指定了某种类型，
// 但您想发送其他内容，请使用 [SetExtraFields]：
p.SetExtraFields(map[string]any{
	"x": 0.01, // send "x" as a float instead of int
})

// 发送数字而非对象
custom := param.Override[anthropic.FooParams](12)
```

### 请求联合类型

联合类型（union）表示为一个结构体，其每个变体对应一个以 "Of" 为前缀的字段，只有一个字段可以为非零值。非零字段将被序列化。

联合类型的子属性可以通过联合结构体上的方法访问。如果存在，这些方法会返回指向底层数据的可变指针。

```go
// 只能有一个字段为非零值，请使用 param.IsOmitted() 检查字段是否已设置
type AnimalUnionParam struct {
	OfCat *Cat `json:",omitzero,inline"`
	OfDog *Dog `json:",omitzero,inline"`
}

animal := AnimalUnionParam{
	OfCat: &Cat{
		Name: "Whiskers",
		Owner: PersonParam{
			Address: AddressParam{Street: "3333 Coyote Hill Rd", ZipCode: 0},
		},
	},
}

// 修改字段
if address := animal.GetOwner().GetAddress(); address != nil {
	address.ZipCode = 94304
}
```

### 反序列化参数

<Note>
  `param.SetJSON` 需要 SDK v1.20.0 或更高版本。
</Note>

Param 类型（以 `Param` 结尾的类型，例如 `MessageNewParams` 或 `ToolUnionParam`）仅为传出请求而设计。它们可以正确地编组为 JSON，但不完全支持往返反序列化。如果您将原始 JSON 解组到 param 结构体中，即使底层 JSON 有效，像 `OfBashTool20250124` 这样的类型化联合字段也将为 nil。

如果您需要从原始 JSON 重建参数（例如，来自数据库、中间件或先前的请求），请调用 `UnmarshalJSON` 填充非联合字段，然后使用 `param.SetJSON` 附加原始字节以便正确地重新序列化：

```go
// 序列化 params（例如用于存储或转发）
b, err := json.Marshal(original)
if err != nil {
	panic(err)
}

// 之后，从存储的 JSON 重建 params
var params anthropic.MessageNewParams
if err := params.UnmarshalJSON(b); err != nil {
	panic(err)
}
param.SetJSON(b, &params)

// params.Model 及其他标量字段由 UnmarshalJSON 填充。
// params.Tools[0].OfBashTool20250124 为 nil（联合类型的限制），
// 但原始 JSON 会被保留。当 params 再次被序列化
// 用于 API 调用时，tools 会正确序列化。
b2, _ := json.Marshal(params)
fmt.Println(string(b) == string(b2)) // true
```

对于此用例，`param.SetJSON`（自 v1.20.0 起可用）优于更通用的 `param.Override[T](any)`，因为它不需要写出类型参数，并且使往返意图更加明确。

## 响应对象

响应结构体中的所有字段都是普通值类型（不是指针或包装器）。 响应结构体还包含一个特殊的 `JSON` 字段，其中包含有关每个属性的元数据。

```go
type Animal struct {
	Name   string `json:"name,nullable"`
	Owners int    `json:"owners"`
	Age    int    `json:"age"`
	JSON   struct {
		Name        respjson.Field
		Owners      respjson.Field
		Age         respjson.Field
		ExtraFields map[string]respjson.Field
	} `json:"-"`
}
```

要处理可选数据，请在 JSON 字段上使用 `.Valid()` 方法。 当字段存在、非 `null` 且已成功解组时，`.Valid()` 返回 true。

如果 `.Valid()` 为 false，则相应字段将为其零值。

```go
raw := `{"owners": 1, "name": null}`

var res Animal
json.Unmarshal([]byte(raw), &res)

// 访问常规字段

res.Owners // 1
res.Name   // ""
res.Age    // 0

// 可选字段检查

res.JSON.Owners.Valid() // true
res.JSON.Name.Valid()   // false
res.JSON.Age.Valid()    // false

// 原始 JSON 值

res.JSON.Owners.Raw()                  // "1"
res.JSON.Name.Raw() == "null"          // true
res.JSON.Name.Raw() == respjson.Null   // true
res.JSON.Age.Raw() == ""               // true
res.JSON.Age.Raw() == respjson.Omitted // true
```

这些 `.JSON` 结构体还包含一个 `ExtraFields` map，其中包含 json 响应中未在结构体中指定的任何属性。这对于 SDK 中尚未提供的 API 功能可能很有用。

```go
body := res.JSON.ExtraFields["my_unexpected_field"].Raw()
```

### 响应联合类型

在响应中，联合类型由一个扁平化的结构体表示，其中包含每个对象变体的所有可能字段。 要将其转换为某个变体，请使用 `.AsFooVariant()` 方法，或者使用 `.AsAny()` 方法（如果存在）。

如果响应值联合类型包含基本类型值，则基本类型字段将与属性并列，但以 `Of` 为前缀，并带有标签 `json:"...,inline"`。

```go
type AnimalUnion struct {
	// 来自变体 [Dog]、[Cat]
	Owner Person `json:"owner"`
	// 来自变体 [Dog]
	DogBreed string `json:"dog_breed"`
	// 来自变体 [Cat]
	CatBreed string `json:"cat_breed"`
	// ...

	JSON struct {
		Owner respjson.Field
		// ...
	} `json:"-"`
}

// 如果是 animal 变体
if animal.Owner.Address.ZipCode == "" {
	panic("missing zip code")
}

// 根据变体进行 switch 分支
switch variant := animal.AsAny().(type) {
case Dog:
case Cat:
default:
	panic("unexpected type")
}
```

## 错误处理

当 API 返回非成功状态码时，SDK 会返回类型为 `*anthropic.Error` 的错误。其中包含请求的 `StatusCode`、`*http.Request` 和 `*http.Response` 值，以及错误体的 JSON（与 SDK 中的其他响应对象非常相似）。该错误还包含来自响应头的 `RequestID`，这在与 Anthropic 支持团队排查问题时非常有用。

要处理错误，请使用 `errors.As` 模式：

```go
_, err := client.Messages.New(context.TODO(), anthropic.MessageNewParams{
	MaxTokens: 1024,
	Messages: []anthropic.MessageParam{{
		Content: []anthropic.ContentBlockParamUnion{{
			OfText: &anthropic.TextBlockParam{
				Text: "What is a quaternion?",
			},
		}},
		Role: anthropic.MessageParamRoleUser,
	}},
	Model: anthropic.ModelClaudeOpus5,
})
if err != nil {
	var apierr *anthropic.Error
	if errors.As(err, &apierr) {
		println("Request ID:", apierr.RequestID)
		println(string(apierr.DumpRequest(true)))  // Prints the serialized HTTP request
		println(string(apierr.DumpResponse(true))) // Prints the serialized HTTP response
	}
	panic(err.Error()) // POST "/v1/messages": 400 Bad Request (Request-ID: req_xxx) { ... }
}
```

当发生其他错误时，它们会以未包装的形式返回；例如，如果 HTTP 传输失败，您可能会收到包装了 `*net.OpError` 的 `*url.Error`。

## 重试

某些错误默认会自动重试 2 次，并采用短暂的指数退避。 SDK 默认会重试所有连接错误、408 Request Timeout、409 Conflict、429 Rate Limit 以及 >=500 的内部错误。

您可以使用 `WithMaxRetries` 选项来配置或禁用此行为：

```go
// 为所有请求配置默认值:
client := anthropic.NewClient(
	option.WithMaxRetries(0), // default is 2
)

// 按请求覆盖:
// ...
	client.Messages.New(
		context.TODO(),
		anthropic.MessageNewParams{
			MaxTokens: 1024,
			Messages: []anthropic.MessageParam{{
				Content: []anthropic.ContentBlockParamUnion{{
					OfText: &anthropic.TextBlockParam{
						Text: "What is a quaternion?",
					},
				}},
				Role: anthropic.MessageParamRoleUser,
			}},
			Model: anthropic.ModelClaudeOpus5,
		},
		option.WithMaxRetries(5),
	)
```

## 超时

非流式传输的 Messages 请求默认在 10 分钟后超时；其他请求没有默认超时。请使用 context 为请求生命周期配置超时。

请注意，如果请求被[重试](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/go#retries)，context 超时不会重新开始计时。 要设置每次重试的超时，请使用 `option.WithRequestTimeout()`。

```go
// 此处设置请求的超时时间，包括所有重试。
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
defer cancel()
// ...
	client.Messages.New(
		ctx,
		anthropic.MessageNewParams{
			MaxTokens: 1024,
			Messages: []anthropic.MessageParam{{
				Content: []anthropic.ContentBlockParamUnion{{
					OfText: &anthropic.TextBlockParam{
						Text: "What is a quaternion?",
					},
				}},
				Role: anthropic.MessageParamRoleUser,
			}},
			Model: anthropic.ModelClaudeOpus5,
		},
		// 此处设置每次重试的超时时间
		option.WithRequestTimeout(20*time.Second),
	)
```

## 长请求

<Warning>
  对于运行时间较长的请求，请考虑使用流式传输 Messages API。
</Warning>

避免在不使用流式传输的情况下设置较大的 `MaxTokens` 值，因为某些网络可能会在一段时间后断开空闲连接，这可能导致请求失败或[超时](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/go#timeouts)而未收到来自 Anthropic 的响应。

如果预计非流式传输请求的时长超过大约 10 分钟，此 SDK 也会返回错误。 调用 `.Messages.NewStreaming()` 或[设置自定义超时](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/sdks/go#timeouts)可禁用此错误。

## 文件上传

与 multipart 请求中的文件上传相对应的请求参数类型为 `io.Reader`。`io.Reader` 的内容默认会作为 multipart 表单部分发送，文件名为 "anonymous\_file"，content-type 为 "application/octet-stream"，因此推荐的做法是使用 `anthropic.File(reader io.Reader, filename string, contentType string)` 辅助函数指定自定义 content-type，该函数会用适当的文件名和内容类型包装任何 `io.Reader`。

```go
// 来自文件系统的文件
file, err := os.Open("/path/to/file.json")
anthropic.FileUploadParams{
	File: anthropic.File(file, "custom-name.json", "application/json"),
}

// 来自字符串的文件
anthropic.FileUploadParams{
	File: anthropic.File(strings.NewReader("my file contents"), "custom-name.json", "application/json"),
}
```

文件名和 content-type 也可以通过在 `io.Reader` 的运行时类型上实现 `Name() string` 或 `ContentType() string` 来自定义。请注意，`os.File` 实现了 `Name() string`，因此由 `os.Open` 返回的文件将以磁盘上的文件名发送。

## 分页

此库为使用分页列表端点提供了一些便利功能。

您可以使用 `.ListAutoPaging()` 方法遍历所有页面中的项目：

```go
iter := client.Messages.Batches.ListAutoPaging(context.TODO(), anthropic.MessageBatchListParams{
	Limit: anthropic.Int(20),
})
// 根据需要自动获取更多页面。
for iter.Next() {
	messageBatch := iter.Current()
	fmt.Println(messageBatch.ID)
}
if err := iter.Err(); err != nil {
	panic(err.Error())
}
```

或者，您可以使用简单的 `.List()` 方法获取单个页面，并接收一个带有额外辅助方法（如 `.GetNextPage()`）的标准响应对象：

```go
page, err := client.Messages.Batches.List(context.TODO(), anthropic.MessageBatchListParams{
	Limit: anthropic.Int(20),
})
for page != nil {
	for _, batch := range page.Data {
		fmt.Println(batch.ID)
	}
	page, err = page.GetNextPage()
}
if err != nil {
	panic(err.Error())
}
```

## RequestOptions

此库使用函数式选项模式。`option` 包中定义的函数返回一个 `RequestOption`，它是一个修改 `RequestConfig` 的闭包。这些选项可以提供给客户端，也可以在单个请求中提供。例如：

```go
client := anthropic.NewClient(
	// 为客户端发出的每个请求添加一个标头
	option.WithHeader("X-Some-Header", "custom_header_info"),
)

client.Messages.New(context.TODO(), // ...,
	// 覆盖该标头
	option.WithHeader("X-Some-Header", "some_other_custom_header_info"),
	// 使用 sjson 语法向请求体添加一个未公开的字段
	option.WithJSONSet("some.json.path", map[string]string{"my": "object"}),
)
```

请求选项 `option.WithDebugLog(nil)` 在调试时可能会有帮助。

请参阅[请求选项的完整列表](https://pkg.go.dev/github.com/anthropics/anthropic-sdk-go/option)。

## HTTP 客户端自定义

有关请求中间件（`option.WithMiddleware`）和替换默认 `http.Client`（`option.WithHTTPClient`）的信息，请参阅 [SDK 中间件](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/middleware)。

## 平台集成

<Note>
  有关带代码示例的详细平台设置指南，请参阅：

  * [Amazon Bedrock](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)
  * [Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)
  * [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)
  * [Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)
</Note>

Go SDK 支持以下平台：

* **Agent Platform：** `import "github.com/anthropics/anthropic-sdk-go/vertex"`。使用 `vertex.WithGoogleAuth(ctx, region, projectID)` 或 `vertex.WithCredentials(ctx, region, projectID, creds)`。
* **Bedrock：** `import "github.com/anthropics/anthropic-sdk-go/bedrock"`。对于 Messages-API Bedrock 端点（通过 SSE 进行流式传输），请使用 `bedrock.NewMantleClient`；或者使用 `bedrock.WithLoadDefaultConfig(ctx)` / `bedrock.WithConfig(cfg)`（`bedrock-runtime` 路径）。导入 `bedrock` 包会在 SDK 的流式传输层中全局注册一个 `application/vnd.amazon.eventstream` 解码器（通过包的 `init()`）。无论您使用 `bedrock-runtime` 的 `WithConfig`/`WithLoadDefaultConfig` 路径还是 `NewMantleClient`，这都适用。
* **Claude Platform on AWS：** `import anthropicaws "github.com/anthropics/anthropic-sdk-go/aws"`。使用 `anthropicaws.NewClient(ctx, cfg)` 并传入一个 `anthropicaws.ClientConfig` 值来构造客户端；在配置上设置 `WorkspaceID`，或设置 `ANTHROPIC_AWS_WORKSPACE_ID` 环境变量。当同时导入 `github.com/aws/aws-sdk-go-v2/aws` 时，`anthropicaws` 导入别名可避免名称冲突。目前处于 beta 阶段。
* **Foundry：** Go SDK 目前不支持。有关支持的 SDK，请参阅 [Claude in Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)。

新项目请使用 `bedrock.NewMantleClient`；`bedrock.WithLoadDefaultConfig`/`WithConfig` 仍保留给使用 Bedrock `InvokeModel` API 的现有应用程序。

## 高级用法

### 访问原始响应数据（例如响应头）

您可以使用 `option.WithResponseInto()` 请求选项访问原始 HTTP 响应数据。当您需要检查响应头、状态码或其他详细信息时，这非常有用。

```go
// 创建一个变量来存储 HTTP 响应
var response *http.Response
message, err := client.Messages.New(
	context.TODO(),
	anthropic.MessageNewParams{
		MaxTokens: 1024,
		Messages: []anthropic.MessageParam{{
			Content: []anthropic.ContentBlockParamUnion{{
				OfText: &anthropic.TextBlockParam{
					Text: "What is a quaternion?",
				},
			}},
			Role: anthropic.MessageParamRoleUser,
		}},
		Model: anthropic.ModelClaudeOpus5,
	},
	option.WithResponseInto(&response),
)
if err != nil {
	// 处理错误
}
fmt.Printf("%+v\n", message.Content)

fmt.Printf("Status Code: %d\n", response.StatusCode)
fmt.Printf("Headers: %+#v\n", response.Header)
```

### 发起自定义/未文档化的请求

此库经过类型化，以便于访问已文档化的 API。如果您需要访问未文档化的端点、参数或响应属性，仍然可以使用此库。

#### 未文档化的端点

要向未文档化的端点发起请求，您可以使用 `client.Get`、`client.Post` 和其他 HTTP 动词。 发起这些请求时，客户端上的 `RequestOptions`（例如重试）将会生效。

```go
var (
	// params 可以是 io.Reader、[]byte、可通过 encoding/json 序列化的对象，
	// 或本库中定义的 "...Params" 结构体。
	params map[string]any

	// result 可以是 []byte、*http.Response、可通过 encoding/json 反序列化的对象，
	// 或本库中定义的模型。
	result *http.Response
)
err := client.Post(context.Background(), "/unspecified", params, &result)
if err != nil {
	// ...
}
```

#### 未文档化的请求参数

要使用未文档化的参数发起请求，您可以使用 `option.WithQuerySet()` 或 `option.WithJSONSet()` 方法。

```go
params := FooNewParams{
	ID: "id_xxxx",
	Data: FooNewParamsData{
		FirstName: anthropic.String("John"),
	},
}
client.Foo.New(context.Background(), params, option.WithJSONSet("data.last_name", "Doe"))
```

#### 未文档化的响应属性

要访问未文档化的响应属性，您可以使用 `result.JSON.RawJSON()` 以字符串形式访问响应的原始 JSON，或者使用 `result.JSON.Foo.Raw()` 获取结果中特定字段的原始 JSON。

响应结构体中不存在的任何字段都会被保存，并可通过 `result.JSON.ExtraFields` 访问，它是一个 `map[string]respjson.Field`。

## 语义化版本控制

此包总体上遵循 [SemVer](https://semver.org/spec/v2.0.0.html) 约定，但某些向后不兼容的更改可能会作为次要版本发布：

1. 对库内部的更改，这些内部在技术上是公开的，但并非为外部使用而设计或记录。
2. 预计在实践中不会影响绝大多数用户的更改。

我们非常重视向后兼容性，以确保您可以获得顺畅的升级体验。

欢迎您提供反馈；如有问题、bug 或建议，请提交 [issue](https://github.com/anthropics/anthropic-sdk-go/issues)。

## 其他资源

* [GitHub 仓库](https://github.com/anthropics/anthropic-sdk-go)
* [Go 包文档](https://pkg.go.dev/github.com/anthropics/anthropic-sdk-go)
* [API 参考](https://platform.claude.com/docs/zh-CN/api/overview)
* [流式传输 Messages](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)
