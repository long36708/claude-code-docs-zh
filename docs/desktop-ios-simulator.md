> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 在模拟器中测试 iOS 应用

> Claude Code Desktop 在 Claude 构建、运行或检查应用时，会在 iOS Simulator 窗格中打开您的应用。

> [!NOTE]
>
> iOS Simulator 窗格在 macOS 上的 Claude Code Desktop 中处于公开测试版。它在 Pro、Max、Ultra、Team 和 Enterprise 计划中可用，但在启用了 HIPAA 配置的 Enterprise 组织中不可用。如果您使用[第三方提供商](https://code.claude.com/docs/zh-CN/third-party-integrations.md)，则无法使用此功能。

iOS Simulator 窗格在 Claude Code Desktop 中的对话旁边显示你的应用在 Apple 的 iOS Simulator 中运行。当 Claude 在模拟器中构建、安装、启动或检查你的应用时，该窗格会自动打开并实时流式传输设备屏幕。使用它来观看 Claude 运行和测试你的应用，或者在 Claude 继续工作时自己点击浏览应用。

模拟器窗格直接驱动模拟器，因此它不需要[计算机使用](https://code.claude.com/docs/zh-CN/desktop.md#let-claude-use-your-computer)，也不会接管你的屏幕或隐藏其他窗口。从 CLI 中，Claude 通过[计算机使用](https://code.claude.com/docs/zh-CN/computer-use.md#test-a-simulator-flow)访问 iOS Simulator，它以与你使用鼠标相同的方式控制屏幕上的模拟器。

## 要求

模拟器窗格使用 Apple 的模拟器工具，桌面应用不包含这些工具。在开始会话之前，请确保您有：

* Claude Desktop v1.24012.0 或更高版本
* 一台 Mac，因为 Apple 的 iOS Simulator 仅在 macOS 上运行
* [Xcode](https://developer.apple.com/xcode/)，其中安装了 iOS 平台，它提供模拟器设备。如果 Xcode 还没有列出任何模拟器，请参阅[模拟器窗格显示未找到模拟器](#the-simulator-pane-says-no-simulators-were-found)
  * 推荐使用 Xcode 26.x 或 Xcode 27，该窗格可与两者一起使用。如果您安装了多个 Xcode，请参阅[选择模拟器窗格使用的 Xcode](#choose-which-xcode-the-simulator-pane-uses)
  * 要在 iPhone Duo 上进行测试，您需要 Xcode 27.1 或更高版本。有关其他所需条件，请参阅[在 iPhone Duo 上测试您的应用](#test-your-app-on-an-iphone-duo)。

> [!NOTE]
>
> 在本页上，"设备"指的是模拟的 iPhone 或 iPad，是您在 Xcode 中的**Window → Devices and Simulators** 下管理的相同模拟器设备之一，而不是物理硬件。

模拟器窗格仅在本地会话中可用。在[云](https://code.claude.com/docs/zh-CN/desktop.md#run-long-running-tasks-in-the-cloud)和 [SSH](https://code.claude.com/docs/zh-CN/desktop.md#ssh-sessions) 会话中，Claude 在无法访问您 Mac 上模拟器的机器上运行。

### 选择模拟器窗格使用的 Xcode

如果您安装了多个 Xcode，例如 Xcode 26.x 与 Xcode 27 并行安装，该窗格使用 `xcode-select` 指向的那一个。要使用不同的安装，请按其路径选择它。

例如，如果您想要的 Xcode 安装为 `/Applications/Xcode-26.4.app`，请在终端中运行以下命令：

```bash theme={null}
sudo xcode-select -s /Applications/Xcode-26.4.app
```

然后退出并重新打开 Claude Desktop，以便该窗格使用新选择的 Xcode。

要检查选择了哪个安装，请运行 `xcode-select -p`。它会打印所选安装的 Developer 文件夹，例如 `/Applications/Xcode-26.4.app/Contents/Developer`。

## 在模拟器中运行你的应用

你不需要命令或设置来打开模拟器窗格。当 Claude 在模拟器中运行你的应用时，它会打开该窗格。

**1. 打开你的 iOS 项目**

在 Claude Code Desktop 中，打开**Code** 选项卡并使用你的应用项目作为[项目文件夹](https://code.claude.com/docs/zh-CN/desktop.md#start-a-session)启动会话。任何为 iOS Simulator 构建应用的项目都可以使用。

**2. 要求 Claude 运行或测试应用**

围绕运行或验证应用来表述任务。例如：

```text theme={null}
构建应用并在模拟器中运行它以检查入门流程。
```

**3. 在模拟器窗格中观看应用**

当应用在模拟器中启动时，iOS Simulator 窗格会在对话旁边打开。Claude 第一次使用设备时，桌面应用会要求你允许它；请参阅[授予 Claude 对设备的访问权限](#grant-claude-access-to-a-device)。Claude 安装应用、点击浏览它，并读取屏幕以验证自己的更改，同时你观看。

当您的请求是关于查看应用时，例如"新屏幕看起来对吗？"，Claude 会在开始工作之前启动模拟器。在 Claude 修复错误或更改屏幕后，请要求它验证更改。

模拟器窗格显示应用实际启动的任何设备。要在特定设备上测试，在你的请求中命名它，例如"在 iPhone SE 模拟器上运行它"，Claude 在构建和启动时会针对该设备。

Claude 启动的设备也会出现在 Apple 的 Simulator 应用中，或在 Xcode 27 的 Device Hub 中。Claude 可以在你已经启动的设备上安装应用。

### 自行打开模拟器窗格

一旦会话连接了模拟器或 Claude Code 检测到 iOS 应用相关工作，请在会话的标题栏或[侧边面板](https://code.claude.com/docs/zh-CN/desktop.md#arrange-your-workspace)中选择 **iOS Simulator**。

如果窗格还没有显示设备，请单击 **Attach simulator**，或从它旁边的设备菜单中选择特定设备；选择已关闭的设备会启动它。如果缺少 Xcode 或其模拟器，窗格会改为显示设置步骤，并在您完成每个步骤时将其勾选。

## 自己控制模拟器

模拟器窗格是交互式的，不仅仅是查看器。在 Claude 工作时或任务之间，你可以：

* 通过在设备屏幕上单击和拖动来点击和滑动
* 使用与 Apple 的 Simulator 应用相同的快捷键按下硬件按钮：**Cmd+Shift+H** 表示主屏幕，**Cmd+L** 表示锁定，**Cmd+Up Arrow** 和 **Cmd+Down Arrow** 表示音量
* 使用旋转按钮或 **Cmd+Right Arrow** 将设备顺时针旋转四分之一圈
* 从设备菜单中切换窗格显示的设备，该菜单列出每个模拟器的操作系统版本以及它是否已启动
* 使用 **Cmd+S** 保存屏幕截图或使用 **Cmd+R** 保存屏幕录制，使用窗格的捕获按钮或快捷键；文件保存到你的桌面
* 通过单击**Detach simulator** 停止流式传输设备而不关闭它，这会将窗格返回到其**Attach simulator** 状态

如果窗格显示 **Display** 菜单，可以使用它来调整来自模拟器的视频流。如果窗格对您的 Mac 造成压力，请降低**Frame rate** 或**Resolution**。这两项设置改变窗格显示设备的方式，而不是应用运行的方式。

你和 Claude 驱动同一设备，因此你的点击会改变 Claude 看到的应用状态。要让 Claude 检查特定屏幕，通过点击导航到它，然后提出要求。当 Claude 驱动设备时，窗格在屏幕上方显示**Claude is using this device** 徽章；在徽章清除之前暂停点击，以便结果反映应用而不是你的输入。

### 最小化或关闭模拟器窗格

您可以最小化或关闭模拟器窗格，无论哪种方式，Claude 仍然可以使用该设备。

* 如果最小化窗格，设备会以较小尺寸保持在视图中。要最小化窗格或再次展开它，请参阅[最小化窗格](https://code.claude.com/docs/zh-CN/desktop.md#minimize-a-pane)。
* 如果关闭全尺寸或已最小化的窗格，设备仍保持连接并继续运行。要再次显示设备，请[重新打开窗格](#open-the-simulator-pane-yourself)。

## 会话如何管理设备

如果提示词中未指定模拟器，Claude 会使用一个已在运行的模拟器，即使它是由另一个[并行会话](https://code.claude.com/docs/zh-CN/desktop.md#work-in-parallel-with-sessions)打开的。要为每个会话分配各自的模拟器，请在每个会话的提示词中指定不同的模拟器。

Claude Code Desktop 会在模拟器不再使用时关闭它启动的模拟器：当你退出应用时、当你归档会话时，或在你从窗格中分离设备后 10 分钟。你在 Claude Code Desktop 外启动的设备，例如在 Apple 的 Simulator 应用或 Device Hub 中启动的设备，永远不会自动关闭。要立即关闭附加的设备，请使用窗格中的关闭按钮。

## 授予 Claude 对设备的访问权限

Claude 在控制设备之前会征求您的同意，而构建或启动应用、在设备上打开 URL 则遵循您会话的权限模式。要完全移除 Claude 的访问权限，请[关闭模拟器访问](#turn-off-simulator-access)。

### 第一次允许设备

Claude 第一次使用模拟器时，桌面应用会要求你允许它。同意涵盖控制该设备和对其进行屏幕截图，你每个设备给予一次而不是每个会话一次。Claude 对设备的屏幕截图被发送到 Anthropic 并根据你的正常对话保留设置保留，因此不要在 Claude 使用的设备上登录真实账户。

在您允许设备后，Claude 对其执行的操作（例如点击、输入和拍摄屏幕截图）无需提示即可运行，但启动应用需要按照您的[权限模式](#actions-that-follow-your-permission-mode)的要求进行批准。这些操作具有与您在窗格中单击相同的信任级别，并且仅作用于模拟设备，因此窗格不需要计算机使用所需的 macOS 辅助功能和屏幕录制权限。

如果您拒绝 Claude 的请求，设备不会连接，Desktop 也不会启动它。要自己使用该设备：

1. 在窗格中单击 **Attach simulator**。
2. 在权限屏幕上单击 **Continue**。
3. 在随后出现的对话框中单击 **Cancel**。

设备会连接并可用于您自己的点击，而 Claude 的访问保持关闭。之后，要授予 Claude 访问权限，请在窗格中单击 **Let Claude use it**。

### 遵循你的权限模式的操作

即使在您允许设备之后，以下三个操作仍遵循您会话的[权限模式](https://code.claude.com/docs/zh-CN/permissions.md#permission-modes)：

* 在设备上打开 URL，例如测试深层链接或在设备的 Safari 中加载页面，因为 URL 可以将数据从设备中携带出去。
* 构建应用，因为 `xcodebuild` 在你的 Mac 上运行你的项目的构建脚本。检查已在进行的构建不会提示。
* 安装和启动应用，因为 Claude 可以运行它指定的任何应用，而不仅仅是您正在构建的应用。

### 关闭模拟器访问

你可以在桌面应用的设置中关闭 Claude 的模拟器访问。组织有两种方式为所有人关闭它：

* `disableMobileSimulatorTools` [托管设置](https://code.claude.com/docs/zh-CN/desktop.md#managed-settings)阻止 Claude 的模拟器工具。模拟器窗格仍可用于你自己的点击，该设置无法从应用内覆盖。
* `requireCoworkFullVmSandbox` 策略密钥，它在隔离的虚拟机内而不是在你的 Mac 上运行 Claude 的工具，禁用模拟器窗格和 Claude 的模拟器工具，因此当它被设置时窗格无法连接设备。

Claude 会告诉你何时应用任一情况。

## 在 iPhone Duo 上测试您的应用

您可以在模拟器窗格中使用 iPhone Duo 模拟器测试应用，并将设备折叠为其各个姿态：合上、半开和展开。要在该设备上测试，您需要：

* Claude Desktop v4.29.0 或更高版本，该版本比模拟器窗格本身所需的 v1.24012.0 更新。要进行更新，请打开 **Claude > Check for Updates…**，然后重启应用。
* Xcode 27.1 或更高版本及其 iOS 模拟器运行时，其中包含 iPhone Duo 模拟器。对于 Xcode 27.1，该运行时为 iOS 27.1。如果缺少该运行时，请[从 Xcode 的 Components 设置中下载](https://developer.apple.com/documentation/xcode/downloading-and-installing-additional-xcode-components)。如果您安装了多个 Xcode，请[选择 Xcode 27.1 或更高版本](#choose-which-xcode-the-simulator-pane-uses)。

要在窗格中打开 iPhone Duo，请在请求中指明该设备，或[自行将窗格切换到该设备](#control-the-simulator-yourself)。如果设备菜单中尚未列出 iPhone Duo，请选择设备列表末尾的 **Add simulator…**，然后选择 iPhone Duo 设备类型。

例如，要让 Claude 在每种姿态下检查应用的某个屏幕：

```text theme={null}
Run the app on the iPhone Duo simulator and check the settings screen closed, half open, and open.
```

### 折叠 iPhone Duo

* **自行折叠**：点击旋转按钮旁边的三个姿态按钮之一。将指针悬停在按钮上可查看其名称：**Closed**、**Half open** 或 **Open**。如需介于这些姿态之间的角度，请将指针移到姿态按钮上，然后拖动其上方出现的滑块。
* **让 Claude 折叠**：请求三种姿态之一，或请求 1 到 179 度之间的半开铰链角度。

### 导出 Xcode Agent skill 的提示

在 iPhone Duo 上进行布局检查之前，Claude 可能会询问是否将 Xcode 的 Agent skill 导出到您的[个人 skill 文件夹](https://code.claude.com/docs/zh-CN/skills.md#where-skills-live)。这些 skill 随 Xcode 提供，用于指导编码 Agent；一旦它们位于该文件夹中，Claude Code 也会在您的其他项目中加载它们。其中包括一个用于让应用布局适配 iPhone Duo 的 skill。

## 限制

Claude 仅驱动模拟设备，无法控制物理 iPhone 或 iPad。要在其上测试，从 Xcode 自己在其上运行应用，然后描述你看到的内容或将屏幕截图附加到对话中供 Claude 使用。

## 故障排除

### 当 Claude 运行应用时模拟器窗格不打开

Claude 可能没有识别出你想要运行或测试应用，或者模拟器工具可能缺失。检查以下内容：

* 明确说明目标，例如"在 iOS Simulator 中运行应用并点击浏览注册流程"。
* 确认 Xcode 和 iOS 模拟器已安装，并且你的 Xcode 版本符合[要求](#requirements)。
* 如果你的组织管理 Claude Code，[模拟器工具可能被策略禁用](#turn-off-simulator-access)。
* 如果您在启用了 HIPAA 配置的 Enterprise 组织中，或者您使用的是[第三方提供商](https://code.claude.com/docs/zh-CN/third-party-integrations.md)，模拟器窗格对您不可用。
* 模拟器窗格需要 Claude Desktop v1.24012.0 或更高版本。打开 **Claude > Check for Updates…**，然后重启应用。

### 模拟器窗格显示未找到模拟器

Xcode 已安装但没有 iOS 模拟器可列出。模拟器窗格显示要遵循的设置步骤，并在每个步骤完成时检查它们。要手动安装缺失的部分，从 Xcode 的设置中下载 iOS 模拟器运行时，或运行 `xcodebuild -downloadPlatform iOS`。

## 另请参阅

* [Desktop 中的计算机使用](https://code.claude.com/docs/zh-CN/desktop.md#let-claude-use-your-computer)：没有专用窗格的应用的屏幕控制
* [CLI 中的计算机使用](https://code.claude.com/docs/zh-CN/computer-use.md)：CLI 如何访问 iOS Simulator
* [在独立的 worktree 中运行会话](https://code.claude.com/docs/zh-CN/desktop.md#run-a-session-in-its-own-worktree)：会话如何隔离更改
* [开始使用 Claude Code Desktop](https://code.claude.com/docs/zh-CN/desktop-quickstart.md)
