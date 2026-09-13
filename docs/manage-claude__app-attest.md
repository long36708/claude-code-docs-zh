---
title: 适用于 iOS 和 macOS 应用的 App Attest
url: https://platform.claude.com/docs/zh-CN/manage-claude/app-attest
description: 使用 Apple 的 App Attest 服务，让您的 iOS 或 macOS 应用的正版安装实例无需内置 API 密钥或运行代理即可调用 Claude API。
---

App Attest 用于对直接从设备调用 Claude API 的 iOS 和 macOS 应用进行身份验证，其用量计入您的工作区账单。本页介绍 App Attest 的工作原理、如何在 Claude Console 中注册您的应用，以及如何撤销应用集成。

应用通过 [Claude for Foundation Models](https://github.com/anthropics/ClaudeForFoundationModels) Swift 包使用 App Attest，该包目前处于 beta 阶段：它需要 OS 27 beta 版本，并且 API 在 beta 期间可能会发生变化。有关 Swift 配置，请参阅 [Apple Foundation Models](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/apple-foundation-models#app-attest-production)。

## App Attest 的工作原理

您应用的每个安装实例都会使用 Apple 的 [App Attest](https://developer.apple.com/documentation/devicecheck/establishing-your-app-s-integrity) 服务来证明它是您所注册应用的真实、未经修改的构建版本。随后，Anthropic 会向该设备颁发一个短期有效的 "access token"（访问令牌），并将使用量计入您的工作区。应用中不附带任何 API 密钥，您也无需运营任何代理。

App Attest 身份验证仅在您的应用直接调用 Claude API 时可用。通过 Amazon Bedrock、Google Cloud 或 Microsoft Foundry 无法使用该功能。

当您的应用首次在某台设备上使用 Claude 时，应用会向 Anthropic 请求一个 challenge（质询），使用 Apple 的 `DCAppAttestService` 对设备进行证明，然后用经过验证的证明换取一个 access token（访问令牌）。Claude for Foundation Models 包会自动运行此流程，并在令牌过期时请求新令牌；您无需编写任何证明代码。

令牌的作用域限定于您的工作区，一小时后过期，并且仅授权 [Messages API](https://platform.claude.com/docs/zh-CN/api/messages/create) 调用。它们不携带任何终端用户身份：App Attest 识别的是您的应用，而不是使用它的人，因此请在您的应用中处理任何按用户区分的逻辑。

## 设置 App Attest

<Note>
  App Attest 需要实体设备。模拟器（Simulator）以及不具备 Secure Enclave 的硬件无法执行 App Attest。在模拟器中开发时，请改用 [API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#api-keys)进行身份验证。
</Note>

要设置 App Attest，您需要拥有 Apple Developer Team ID，并在您的组织中具有管理员、所有者或主要所有者角色。请配置您的 Xcode 项目，并在 [Claude Console](https://platform.claude.com/) 中注册您的应用：

1. 在 Xcode 中，在 **Signing & Capabilities** 下为您的应用目标添加 **App Attest** 功能。
2. 在 Claude Console 中您工作区的设置里，打开 **App integrations**。
3. 点击 **Create app integration**，然后输入名称、您的 Apple Developer Team ID 以及一个或多个 bundle ID（最多 32 个）。
4. 从该集成的 **Overview** 选项卡中复制客户端 ID（`clid_...`），并将其传递给您应用的 Claude 配置。

## 撤销应用集成

要停止已遭入侵或已停用的应用，请撤销其集成：在 Claude Console 中您工作区的设置里，打开 **App integrations**，选择该集成，点击 **Revoke**，然后确认。撤销集成会撤销其所有未过期的令牌，并且其已注册的设备将无法再请求新的令牌。撤销是永久性的，因此如需恢复访问，请创建一个新的应用集成。

## 后续步骤

<CardGroup cols={2}>
  <Card title="Apple Foundation Models" icon="code" href="https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/libraries/apple-foundation-models#app-attest-production">
    在 Claude for Foundation Models Swift 包中配置 App Attest
  </Card>

  <Card title="身份验证" icon="lock" href="https://platform.claude.com/docs/zh-CN/manage-claude/authentication">
    比较 API 密钥、Workload Identity Federation（工作负载身份联合）和 App Attest
  </Card>
</CardGroup>
