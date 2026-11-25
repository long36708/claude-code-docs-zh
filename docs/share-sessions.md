> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 共享云端会话或 Remote Control 会话

> 了解共享的 Claude Code 会话的接收者可以查看哪些内容，选择谁可以打开链接，以及如何停止共享。

您可以通过链接共享[云端会话](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md)或 [Remote Control](https://code.claude.com/docs/zh-CN/remote-control.md) 会话，以便其他人阅读对话内容。

> [!NOTE]
>
> 要共享 claude.ai 聊天，请参阅[共享和取消共享聊天](https://support.claude.com/en/articles/10593882-share-and-unshare-chats)。

## 接收者可以看到的内容

当您共享云端会话或 Remote Control 会话时，接收者可以看到截至目前的对话，包括 Claude 的工具调用及其结果，其中包含以下内容：

* **消息**：您的提示词、您添加的图片、您附加的文件名称，以及 Claude 的回复和思考
* **代码和命令**：Claude 读取、写入或编辑的文件内容，以及您或 Claude 运行的命令及其输出
* **凭据**：出现在消息、文件或命令输出中的密钥或令牌
* **其他仓库**：Claude 的 GitHub 搜索结果，其中可能包含接收者无法打开的私有仓库
* **连接器结果**：[连接器](https://code.claude.com/docs/zh-CN/mcp.md#use-mcp-servers-from-claude-ai)返回的内容，例如电子邮件、文档和 Slack 消息
* **网页内容**：Claude 读取的页面及其截取的屏幕截图
* **您的设置**：Claude 使用的连接器名称，以及它查找过的任何连接器、[skill](https://code.claude.com/docs/zh-CN/skills.md) 和[插件](https://code.claude.com/docs/zh-CN/plugins/overview.md)的名称和描述

您的显示名称会出现在共享的会话中。如果您使用 Max、Ultra 或 Pro 计划，可以[将其隐藏](#share-from-a-max-or-pro-account)。

接收者的视图不会实时更新。当他们重新加载页面或再次打开链接时，会看到您共享之后添加到会话中的消息，直到您[停止共享](#stop-sharing-a-session)为止。

### 接收者在 Remote Control 会话中可以看到的内容

当您共享 Remote Control 会话时，接收者还可以看到：

* Claude 在会话期间读取的您计算机上的文件内容，包括您启动会话所在文件夹之外的文件
* Claude 在会话期间通过 [Claude in Chrome 扩展程序](https://code.claude.com/docs/zh-CN/chrome.md)打开的页面的文本和屏幕截图，包括您已登录的网站

## 共享会话

请先[检查会话](#check-a-session-before-you-share)。然后选择会话顶部的 **Share**，根据您的计划选择 **Team** 或 **Public**，选择 **Copy link**，然后发送该链接。

### 共享前检查会话

阅读对话，展开工具调用，并查找[接收者可以看到的内容](#what-recipients-see)中列出的内容。如果对话中包含接收者不应看到的内容，请将会话保持为 **Private**。

服务器[在某些情况下会阻止共享](#unshareable-sessions)，但服务器允许您共享的会话仍可能包含敏感内容。

### 从 Enterprise 或 Team 账户共享

在 Enterprise 和 Team 计划中，您的 claude.ai 组织中的其他成员可能能够打开您共享的会话：

* **可见性选项**：**Private** 和 **Team**
* **仓库访问权限**：要打开您共享的云端会话，接收者需要通过其[已连接的 GitHub 账户](https://code.claude.com/docs/zh-CN/web-quickstart.md#connect-github)拥有该会话中每个私有仓库的访问权限。没有仓库访问权限的成员可能仍能打开 Remote Control 会话的链接。

### 从 Max、Ultra 或 Pro 账户共享

在 Max、Ultra 和 Pro 计划中，任何拥有该链接并已登录 claude.ai 的人都可能能够打开您共享的会话：

* **可见性选项**：**Private** 和 **Public**
* **仓库访问权限**：**Require repository access** 默认处于关闭状态。开启后，您共享的云端会话将仅限拥有该会话每个私有仓库访问权限的接收者打开。没有仓库访问权限的人可能仍能打开 Remote Control 会话的链接。

要开启 **Require repository access** 或关闭 **Show your name**，请前往 claude.ai 上的 [**Settings > Privacy**](https://claude.ai/settings/data-privacy-controls)，然后点击 **Shared Code sessions** 旁边的 **Manage**。

## 停止共享会话

要停止共享您已共享的会话，请将其可见性设置回 **Private**。请在您离开某个会话或组织之前执行此操作，因为在您执行以下操作后，链接仍然有效：

* 归档会话
* 关闭 Remote Control 或关闭终端
* 离开组织或被移出组织（针对您以 **Team** 方式共享的链接）

关于在您未将其设置为 **Private** 的情况下哪些情况会使链接失效，请参阅[无法再打开的链接](#links-that-stop-opening)。

## 共享问题排查

当您的组织已关闭共享功能，或服务器拒绝共享该会话时，您无法共享会话。链接在共享后也可能无法再打开。

### 您的组织已关闭共享功能

如果 **Share** 呈灰色并显示"Sharing is disabled by your organization."，请联系 [Owner](https://code.claude.com/docs/zh-CN/server-managed-settings.md#access-control) 在 [**Organization settings > Data and privacy**](https://claude.ai/admin-settings/data-privacy-controls) 中开启 **Share cloud sessions**。共享 Remote Control 会话同样需要开启此设置。

在某些合规配置下，即使开启了此设置，您也无法共享会话。

### 无法共享的会话

某些会话可能包含接收者不应看到的内容，因此您无法共享它们。如果共享失败并显示"This session can't be shared."之类的消息，重试也无济于事。

例如，在 Enterprise 和 Team 计划中，您无法共享使用了[上传代码](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md#send-local-repositories-without-github)的云端会话，因为无法检查谁有权查看该代码。要共享基于该代码的工作，请启动一个新会话，其仓库位于 github.com 上且并非上传而来。

### 无法再打开的链接

如果接收者无法打开链接，请检查以下原因：

* **可见性**：会话为 **Private**，或者接收者未登录 claude.ai，或者对于 **Team** 链接而言接收者不在您的组织中。请检查[谁可以打开该链接](#share-a-session)。
* **仓库访问权限**：该链接[要求仓库访问权限](#share-a-session)，而接收者的 GitHub 账户无法读取该仓库。请让他们[连接一个](https://code.claude.com/docs/zh-CN/web-quickstart.md#connect-github)具有读取权限的账户。
* **上传的代码或非 GitHub 代码**：该链接要求仓库访问权限，而代码是[上传的](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md#send-local-repositories-without-github)，或者位于 GitLab 或 Bitbucket 等 Git 托管平台上。请从仓库位于 github.com 上且并非上传而来的会话进行共享。
* **无效链接**：共享链接可能与浏览器中的地址不一致，并且当您停止共享后再次共享时，**Public** 链接会发生变化。请从 **Share** 复制当前链接并发送。
* **组织设置**：组织已[关闭 **Share cloud sessions**](#sharing-is-turned-off-for-your-organization)。请联系 Owner 重新开启。
* **已升级计划**：您共享了 **Public** 链接，随后将计划升级为 Team 或 Enterprise。该链接将遵循 [Enterprise 和 Team 账户](#share-from-an-enterprise-or-team-account)的规则。
* **已降级计划**：您共享了 **Public** 链接，随后转为 Free 计划。[升级](https://claude.ai/upgrade)到 Max、Ultra 或 Pro 计划后，同一链接即可打开，无需其他操作。

在以下情况下，您无法恢复链接：

* 会话已被删除
* 会话变为[无法共享的会话](#unshareable-sessions)
* 您共享了 **Public** 链接，随后将您的[个人账户迁移](https://support.claude.com/en/articles/9267400-move-your-personal-claude-account-to-a-team-or-enterprise-organization)到了现有组织中
* Anthropic 关闭了该链接，例如在滥用或版权审查之后。会话显示为 **Private**，再次共享时会失败并显示"Couldn't share this session. Try again."

## 相关资源

* [在云端使用 Claude Code](https://code.claude.com/docs/zh-CN/claude-code-on-the-web.md)：归档、删除和移动云端会话
* [Remote Control](https://code.claude.com/docs/zh-CN/remote-control.md)：在另一台设备上继续本地会话
