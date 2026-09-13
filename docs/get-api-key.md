---
title: 获取您的 Claude API 密钥
url: https://platform.claude.com/docs/zh-CN/get-api-key
description: 在 Claude Console 中查找、创建和管理用于 Claude API 的 API 密钥。
---

用于 Claude API 的 API 密钥（也称为 Anthropic API 密钥）存放在 Claude Console 中。要查看您现有的密钥或创建新密钥，请前往 [Settings → API keys](https://platform.claude.com/settings/keys)。

## 选择密钥类型

创建密钥时，您需要选择其类型，这决定了该密钥可以做什么、在哪里有效以及何时停止工作。**个人密钥**（personal key）以您的身份行事，如果您离开组织，它将停止工作。**服务账户密钥**（service account key）代表一个[服务账户](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation#service-accounts)，可供 CI 流水线、生产服务或智能体等工作负载使用。请将个人密钥用于您自己的开发，将服务账户密钥用于任何共享用途。

您还可以创建**工作区密钥**（workspace key），这是一种没有所有者的旧版密钥：它属于您创建它时所在的工作区，并在其创建者离开后继续有效。建议优先使用个人密钥或服务账户密钥，因为当其关联账户从组织中移除时，这些密钥会自动停止工作。

## 创建 API 密钥

<Steps>
  <Step title="登录 Claude Console">
    前往 [platform.claude.com](https://platform.claude.com/) 并登录，如果您还没有账户，请创建一个。
  </Step>

  <Step title="打开 API 密钥页面">
    前往 [Settings → API keys](https://platform.claude.com/settings/keys)。
  </Step>

  <Step title="创建密钥">
    点击 **Create key**，为密钥命名，选择[过期时间](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-expiration)，并将 **Linked account** 设置为您自己或某个服务账户。您还可以选择一个[工作区](https://platform.claude.com/settings/workspaces)来限定该密钥的作用范围。
  </Step>

  <Step title="复制并保存密钥">
    Console 仅在创建时显示一次完整密钥（以 `sk-ant-` 开头）。请复制它并将其保存在安全的地方，例如密钥管理器中。如果您丢失了密钥，将无法在 Console 中再次查看它。请改为创建一个新密钥。
  </Step>
</Steps>

如果 API 密钥页面上的 **Create key** 按钮处于禁用状态，可能是您的角色不允许您在此处创建密钥。请联系组织管理员更改您的角色，或为您的工作负载创建一个服务账户密钥。

## 使用您的 API 密钥

将密钥设置为环境变量：

```bash
export ANTHROPIC_API_KEY="sk-ant-api03-..."
```

[客户端 SDK](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview) 会自动读取 `ANTHROPIC_API_KEY`。直接发送的 HTTP 请求需在 `x-api-key` 请求头中携带密钥。如果您的 API 密钥可在多个工作区中使用，您还必须在每个 Claude API 请求中发送 `anthropic-workspace-id` 请求头，如[选择工作区](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)中所示。关于 Admin API，请参阅 [API 密钥与 Admin API](https://platform.claude.com/docs/zh-CN/get-api-key#api-keys-and-the-admin-api)。

要发出您的第一个请求，请按照[快速入门](https://platform.claude.com/docs/zh-CN/get-started)操作，并参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)了解完整信息，包括通过 Workload Identity Federation（工作负载身份联合）获取的短期凭证。

## API 密钥与 Admin API

[Admin API](https://platform.claude.com/docs/zh-CN/api/admin) 包含用于以编程方式管理您组织的 API 密钥的端点，例如[检索 API 密钥](https://platform.claude.com/docs/zh-CN/api/admin/api_keys/retrieve)和[列出 API 密钥](https://platform.claude.com/docs/zh-CN/api/admin/api_keys/list)。这些端点面向需要自动化密钥管理的组织管理员。它们接受 [Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)、具有 `org:admin` 作用域的 OAuth 令牌，或未限定到特定工作区的个人密钥或服务账户密钥；工作区密钥在此处无效。它们从不返回密钥的机密值，只返回部分脱敏的提示信息。

<Note>
  Admin API 无法恢复丢失的密钥，也无法为您提供用于调用 Claude API 的密钥。要获取可用的 API 密钥，请在 Claude Console 的 [Settings → API keys](https://platform.claude.com/settings/keys) 中创建一个。
</Note>
