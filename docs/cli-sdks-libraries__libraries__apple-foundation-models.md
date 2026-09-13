---
title: Apple Foundation Models
url: https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/apple-foundation-models
description: 通过 Foundation Models 框架和 Claude for Foundation Models Swift 包，在 Apple 平台上使用 Claude。
---

[Claude for Foundation Models](https://github.com/anthropics/ClaudeForFoundationModels) 是一个 Swift 包，它使 Claude 能够作为服务器端语言模型在 Apple 的 [Foundation Models](https://developer.apple.com/documentation/foundationmodels) 框架中使用。该包使 Claude 遵循框架的 `LanguageModel` 协议，因此您可以使用与 Apple 设备端模型相同的 `LanguageModelSession` API 来驱动它：`respond(to:)`、"streaming"（流式传输）、引导式生成和工具调用的工作方式完全相同。

请求直接从您的应用发送到 Claude API；Apple 不在请求路径中，也不会看到提示或响应。使用量按[标准 API 定价](https://platform.claude.com/docs/zh-CN/about-claude/pricing)计入您的 Anthropic 账户，因此您的组织需要有可用的信用余额或有效的计费方式。您的应用决定何时使用 Claude、何时使用 Apple 的设备端模型：将您想要的模型传递给每个会话即可。

<Note>
  **Beta。** 此包面向 OS 27 beta 版中引入的 Foundation Models 服务器端语言模型 API。在 beta 期间，API 可能会发生变化。
</Note>

<Info>
  Claude for Foundation Models **不是**通用的 Messages API 客户端。它的公开接口是 Foundation Models 提供者协议遵循，以及与之相关的配置类型（`ClaudeLanguageModel`、`ClaudeModel`、`AuthMode`、`ClaudeServerTool`）。如需在其他语言中直接访问 Messages API，请参阅[客户端 SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview#client-sdks)。
</Info>

## 要求

* iOS 27、macOS 27、visionOS 27 或 watchOS 27（均为 beta 版）：这些操作系统版本的 Foundation Models 框架支持服务器端语言模型
* Xcode 27（beta 版）
* 用于开发的 Claude "API key"（API 密钥），可从 [Claude Console](https://platform.claude.com/) 获取。生产环境选项请参阅[身份验证](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/apple-foundation-models#authentication)。

## 安装包

将该包添加到您的 `Package.swift`：

```swift
dependencies: [
  .package(url: "https://github.com/anthropics/ClaudeForFoundationModels.git", from: "0.1.0")
]
```

或在 Xcode 中：**File** > **Add Package Dependencies…**，然后输入仓库 URL。

然后将 `ClaudeForFoundationModels` 添加到您的 target 依赖项中，并与 `FoundationModels` 一起导入：

```swift
import FoundationModels
import ClaudeForFoundationModels
```

## 快速开始

`ClaudeLanguageModel` 是入口点。将其传递给 `LanguageModelSession`，然后像使用任何 Foundation Models 提供者一样使用该会话：

```swift
import FoundationModels
import ClaudeForFoundationModels

let model = ClaudeLanguageModel(
  name: .sonnet5,
  auth: .apiKey(ProcessInfo.processInfo.environment["ANTHROPIC_API_KEY"] ?? "")
)

let session = LanguageModelSession(model: model)
let response = try await session.respond(to: "Plan a 4-day trip to Buenos Aires.")
print(response.content)
```

初始化器还接受 `baseURL`（默认为 `https://api.anthropic.com`）、`timeout` 和 `serverTools`（请参阅[服务器端工具](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/apple-foundation-models#server-side-tools)）。

如需完整的可运行程序，仓库中包含 [`Examples/ClaudeExample`](https://github.com/anthropics/ClaudeForFoundationModels/tree/main/Examples/ClaudeExample)，这是一个可运行的命令行 target，可将一轮聊天以流式传输方式输出到终端，并带有 `--search` 标志，用于为该轮对话启用服务器端网页搜索。运行它需要 macOS 27 主机。

## 选择模型

模型标识符是 `ClaudeModel` 的值。使用编译内置的常量，或者为尚未编译内置的 ID 构造一个具有显式能力声明的值（请参阅[能力](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/apple-foundation-models#capabilities)）：

```swift
ClaudeLanguageModel(name: .opus5, auth: auth)
```

常量与 API 模型 ID 一一对应（`.opus5` 即 `claude-opus-5`），并携带每个模型的能力信息。新模型会在包的新版本中以新常量的形式发布；请在 Xcode 中查看 `ClaudeModel` 以获取当前列表，并参阅[模型概览](https://platform.claude.com/docs/zh-CN/models/overview)来比较模型。

### 能力

每个 `ClaudeModel` 都声明了它所接受的内容：采样参数、努力级别、自适应思考、结构化输出和图像输入。该包据此决定发送哪些请求字段，因为发送模型拒绝的字段会导致硬错误。常量携带了正确的能力信息。对于未编译内置的 ID，请声明该模型接受的内容（有意不提供任何进行猜测的简写方式）：

```swift
let model = ClaudeModel(
  id: "claude-experimental-x",
  capabilities: .init(samplingParams: false, effortLevels: [.low, .high])
)
ClaudeLanguageModel(name: model, auth: auth)
```

### 努力级别

使用 `fixedEffort:` 为每个请求固定一个 Claude [努力级别](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)。它优先于框架的每请求推理提示。框架的命名推理级别最高为 high；若要为单个请求请求更高的努力级别，请传递一个以 Claude 努力级别命名的自定义推理级别（`.custom("xhigh")` 或 `.custom("max")`），它会直接映射。当未发送努力级别时，API 默认为 `high`：

```swift
ClaudeLanguageModel(name: .opus5, auth: auth, fixedEffort: .xhigh)
```

该级别必须是模型所接受的级别。每个 `ClaudeModel` 都声明了其模型接受五个级别（`low`、`medium`、`high`、`xhigh`、`max`）中的哪些（如果有的话）：有些模型完全不接受努力级别。

### 何时使用 Claude，何时使用设备端模型

Apple 的设备端模型速度快、私密且可离线使用，但其规模适合轻量级任务。当您需要更大的上下文、前沿推理能力或服务器端工具（如网页搜索和代码执行）时，请升级到 Claude。由于两者使用相同的 `LanguageModelSession` API，您只需替换 `model:` 参数即可切换。

## 身份验证

使用 `auth:` 参数设置凭据。使用 `.appAttest` 可在无后端的情况下发布，使用 `.proxied` 可通过您自己的后端路由请求，使用 `.apiKey` 可在开发期间快速迭代。

### App Attest

您应用的每个安装实例都会使用 Apple 的 [App Attest](https://developer.apple.com/documentation/devicecheck/establishing-your-app-s-integrity) 服务来证明它是您所注册应用的真实、未经修改的构建版本。随后，Anthropic 会向该设备颁发一个短期有效的 "access token"（访问令牌），并将使用量计入您的工作区。应用中不附带任何 API 密钥，您也无需运营任何代理。

App Attest 身份验证仅在您的应用直接调用 Claude API 时可用。通过 Amazon Bedrock、Google Cloud 或 Microsoft Foundry 无法使用该功能。

若要在不运行后端的情况下发布，请使用 `.appAttest`：

```swift
ClaudeLanguageModel(
  name: .sonnet5,
  auth: .appAttest(clientID: "clid_...")
)
```

<Note>
  App Attest 需要物理设备。模拟器以及没有 Secure Enclave 的硬件无法执行 App Attest。在模拟器中迭代时使用 `.apiKey`，在设备上运行时使用 `.appAttest`。
</Note>

要设置 App Attest，您需要拥有 Apple Developer Team ID，并在您的组织中具有管理员、所有者或主要所有者角色。请配置您的 Xcode 项目，并在 [Claude Console](https://platform.claude.com/) 中注册您的应用：

1. 在 Xcode 中，在 **Signing & Capabilities** 下为您的应用目标添加 **App Attest** 功能。
2. 在 Claude Console 中您工作区的设置里，打开 **App integrations**。
3. 点击 **Create app integration**，然后输入名称、您的 Apple Developer Team ID 以及一个或多个 bundle ID（最多 32 个）。
4. 从该集成的 **Overview** 选项卡中复制客户端 ID（`clid_...`），并将其传递给您应用的 Claude 配置。

当您的应用首次在某台设备上使用 Claude 时，应用会向 Anthropic 请求一个 challenge（质询），使用 Apple 的 `DCAppAttestService` 对设备进行证明，然后用经过验证的证明换取一个 access token（访问令牌）。Claude for Foundation Models 包会自动运行此流程，并在令牌过期时请求新令牌；您无需编写任何证明代码。

令牌的作用域限定于您的工作区，一小时后过期，并且仅授权 [Messages API](https://platform.claude.com/docs/zh-CN/api/messages/create) 调用。它们不携带任何终端用户身份：App Attest 识别的是您的应用，而不是使用它的人，因此请在您的应用中处理任何按用户区分的逻辑。

要停止已遭入侵或已停用的应用，请撤销其集成：在 Claude Console 中您工作区的设置里，打开 **App integrations**，选择该集成，点击 **Revoke**，然后确认。撤销集成会撤销其所有未过期的令牌，并且其已注册的设备将无法再请求新的令牌。撤销是永久性的，因此如需恢复访问，请创建一个新的应用集成。

### 代理（生产环境）

对于生产环境，请使用 `.proxied` 通过您自己的后端路由请求。位于 `baseURL` 的中继会在服务器端添加 Claude API 凭据，因此应用中不包含任何密钥。您提供的 `headers` 会随每个请求发送，以便您的代理可以对调用方进行授权。如果不需要任何请求头，请传递 `[:]`：

```swift
ClaudeLanguageModel(
  name: .sonnet5,
  auth: .proxied(headers: ["X-App-Token": "..."]),
  baseURL: URL(string: "https://api.yourapp.com/claude")!
)
```

您的代理接收标准的 [Messages API](https://platform.claude.com/docs/zh-CN/api/messages/create) 请求，附加 `x-api-key` 请求头，然后将其转发到 `https://api.anthropic.com`。

### API 密钥（开发环境）

开发期间直接传递 API 密钥：

```swift
ClaudeLanguageModel(name: .sonnet5, auth: .apiKey("YOUR_API_KEY"))
```

<Warning>
  打包到应用中的密钥可以从发布的二进制文件中提取出来，任何提取到它的人都可以发起计入您账户的请求。`.apiKey` 仅用于开发，发布前请切换到 App Attest 或代理。
</Warning>

## 流式传输

`streamResponse(to:)` 以增量方式返回响应。每个元素都是截至目前响应的累积快照，而不是增量差异：

```swift
let stream = session.streamResponse(to: "Summarize today's top science stories.")
for try await partial in stream {
  print(partial.content)
}
```

## 结构化输出

使用 `@Generable` 标注一个类型，并通过 `generating:` 请求它。模型通过[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)返回该类型的值：

```swift
@Generable
struct Trip {
  @Guide(description: "Destination city") var destination: String
  @Guide(description: "Length in days") var days: Int
}

let response = try await session.respond(to: "Plan a trip to Tokyo.", generating: Trip.self)
print(response.content.destination)
```

结构化输出要求所选模型的能力包含该功能（所有编译内置的常量都包含）。如果所选模型不支持，该包会抛出 `LanguageModelError.unsupportedGenerationGuide`，而不是静默降级。

## 工具使用

### 客户端工具

框架的 `tools:` 数组无需更改即可使用。让您的类型遵循 `Tool`，将它们传递给 `LanguageModelSession`，当 Claude 调用它们时，框架会在设备上执行它们。请参阅[使用 Claude 进行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)。

```swift
let session = LanguageModelSession(model: model, tools: [FindRestaurantsTool()])
```

### 服务器端工具

[服务器工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/server-tools)（网页搜索、网页抓取和代码执行）在 Anthropic 的基础设施上于单次往返内运行，框架无需在设备上调用任何内容。使用 `serverTools:` 为每个模型配置它们：

```swift
let model = ClaudeLanguageModel(
  name: .sonnet5,
  auth: auth,
  serverTools: [
    .webSearch(maxUses: 5),
    .codeExecution,
  ]
)
```

`.webSearch` 和 `.webFetch` 接受可选的 `allowedDomains`、`blockedDomains` 和 `maxUses`。服务器工具活动会以 `ClaudeServerToolSegment` 自定义片段的形式出现在对话记录中。

<Note>
  `serverTools` 配置在 `ClaudeLanguageModel` 上而不是 `LanguageModelSession` 上，因为会话类型属于 Apple。若要为每个对话使用不同的服务器工具集，请构造多个 `ClaudeLanguageModel` 实例。
</Note>

## 图像

能力中包含图像输入的模型会声明框架的视觉能力。通过框架的标准会话 API 传递图像内容；该包会将其转换为 Claude API 的图像格式。图像要求请参阅[视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)。

## 错误处理

该包会在适用的情况下将 Claude API 错误映射到 Apple 的 `LanguageModelError` 枚举值：上下文窗口溢出表现为 `.contextSizeExceeded`，HTTP 429 表现为 `.rateLimited`，超过配置超时时间的请求表现为 `.timeout`。没有框架对应项的提供者错误表现为 `ClaudeError`。通过模式匹配来驱动产品流程：

```swift
do {
  let response = try await session.respond(to: prompt)
  print(response.content)
} catch ClaudeError.missingCredential {
  // 提示输入 API 密钥。
} catch let error as LanguageModelError {
  // 框架形态的错误（速率限制、防护栏、上下文长度、解码）。
} catch {
  // 传输错误。
}
```

一种常见模式是捕获 `.rateLimited`，并在该轮对话中回退到 `SystemLanguageModel`、将请求排队，或显示重试选项。

## 功能支持

该包提供 Foundation Models 提供者协议能够表达的 Messages API 能力。在 Apple 协议中没有对应表示的功能无法通过它使用，包括：

* "Prompt caching"（提示缓存）控制（该包会自动应用提示缓存；缓存 TTL 和断点位置不可配置）
* 停止序列
* 批处理
* Files API
* 令牌计数
* Beta 请求头

## 其他资源

| 参考资料                                                                                             | 涵盖内容                                                              |
| ------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------- |
| [Apple Foundation Models 文档](https://developer.apple.com/documentation/foundationmodels)         | `LanguageModelSession`、`@Generable`、`Transcript`、`Tool` 以及框架的其余接口 |
| [GitHub 上的 `ClaudeForFoundationModels`](https://github.com/anthropics/ClaudeForFoundationModels) | 源代码、可运行示例和问题跟踪器                                                   |
| [Claude API 参考](https://platform.claude.com/docs/zh-CN/api/overview)                             | 底层 Messages API                                                   |

该包采用 Apache 2.0 许可证。欢迎通过 GitHub issues 提交错误报告。beta 期间不接受外部拉取请求。
