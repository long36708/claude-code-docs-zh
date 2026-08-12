# Claude Code 文档镜像

[![最近更新](https://img.shields.io/github/last-commit/long36708/claude-code-docs/main.svg?label=docs%20updated)](https://github.com/long36708/claude-code-docs/commits/main)
[![平台](https://img.shields.io/badge/platform-macOS%20%7C%20Linux-blue)]()
[![测试版](https://img.shields.io/badge/status-early%20beta-orange)](https://github.com/long36708/claude-code-docs/issues)

本仓库是 https://docs.anthropic.com/en/docs/claude-code/ 上 Claude Code 文档文件的本地镜像，每 3 小时更新一次。

## ⚠️ 早期测试版说明

**这是一个早期测试版**。可能存在错误或异常行为。如果你遇到任何问题，请[提交 issue](https://github.com/long36708/claude-code-docs/issues) —— 你的反馈有助于改进这个工具！

## 🆕 版本 0.3.3 - 集成更新日志

**本版本新增内容：**
- 📋 **Claude Code 更新日志**：通过 `/docs changelog` 访问官方 Claude Code 发布说明
- 🍎 **完整的 macOS 兼容性**：修复了 Mac 用户的 shell 兼容性问题
- 🐧 **Linux 支持**：已在 Ubuntu、Debian 等发行版上测试
- 🔧 **改进的安装程序**：更好地处理更新和边界情况

要更新，请运行：
```bash
curl -fsSL https://raw.githubusercontent.com/long36708/claude-code-docs/main/install.sh | bash
```

## 项目初衷

- **更快的访问** - 从本地文件读取，而非从网络获取
- **自动更新** - 尝试与最新文档保持同步
- **追踪变更** - 查看文档随时间发生的变化
- **Claude Code 更新日志** - 快速访问官方发布说明和版本历史
- **更好的 Claude Code 集成** - 让 Claude 更有效地浏览文档

## 平台兼容性

- ✅ **macOS**：完全支持（已在 macOS 12+ 上测试）
- ✅ **Linux**：完全支持（Ubuntu、Debian、Fedora 等）
- ⏳ **Windows**：暂不支持 - [欢迎贡献](#贡献)！

### 前置条件

本工具需要安装以下内容：
- **git** - 用于克隆和更新仓库（通常已预装）
- **jq** - 用于自动更新钩子中的 JSON 处理（macOS 预装；Linux 用户可能需要 `apt install jq` 或 `yum install jq`）
- **curl** - 用于下载安装脚本（通常已预装）
- **Claude Code** - 当然啦 :)

## 安装

运行以下单条命令即可：

```bash
curl -fsSL https://raw.githubusercontent.com/long36708/claude-code-docs/main/install.sh | bash
```

该命令会执行以下操作：
1. 安装到 `~/.claude-code-docs`（或迁移已有安装）
2. 创建 `/docs` 斜杠命令，用于向工具传递参数并告诉它文档的位置
3. 设置一个 `PreToolUse` 的 `Read` 钩子，在从 `~/.claude-code-docs` 读取文档时自动执行 `git pull`

**注意**：该命令为 `/docs (user)` —— 它会在你的命令列表中显示，名称后带有 `"(user)"`，表示这是一个用户自定义命令。

## 使用

`/docs` 命令可即时访问文档，并支持可选的时效性检查。

### 默认模式：极速访问（不检查）
```bash
/docs hooks        # 立即读取 hooks 文档
/docs mcp          # 立即读取 MCP 文档
/docs memory       # 立即读取 memory 文档
```

你会看到：`📚 Reading from local docs (run /docs -t to check freshness)`（正在读取本地文档，运行 /docs -t 可检查时效性）

### 使用 -t 标志检查文档同步状态
```bash
/docs -t           # 显示与 GitHub 的同步状态
/docs -t hooks     # 检查同步状态，然后读取 hooks 文档
/docs -t mcp       # 检查同步状态，然后读取 MCP 文档
```

### 查看最新变更
```bash
/docs what's new   # 显示带有 diff 的最近文档变更
```

### 阅读 Claude Code 更新日志
```bash
/docs changelog    # 阅读官方 Claude Code 发布说明和版本历史
```

更新日志功能会直接从官方 Claude Code 仓库获取最新发布说明，向你展示每个版本的新增内容。

### 卸载
```bash
/docs uninstall    # 获取彻底移除 claude-code-docs 的命令
```

### 自定义命令名称

如果你希望使用其他命令名称（例如用 `/claude-docs` 代替 `/docs`），可以轻松自定义：

```bash
# 重命名命令文件
mv ~/.claude/commands/docs.md ~/.claude/commands/claude-docs.md

# 现在使用 /claude-docs 代替 /docs
/claude-docs hooks
/claude-docs mcp
```

你可以使用任何喜欢的名称：`/cdocs`、`/claude-code-docs` 等。命令文件名决定了斜杠命令的名称。

### 创意用法示例
```bash
# 自然语言查询效果很好
/docs what environment variables exist and how do I use them?
/docs explain the differences between hooks and MCP

# 检查最近变更
/docs -t what's new in the latest documentation?
/docs changelog    # 查看 Claude Code 发布说明

# 在所有文档中搜索
/docs find all mentions of authentication
/docs how do I customize Claude Code's behavior?
```

## 更新机制

文档会尝试保持最新：
- GitHub Actions 定期运行以获取新文档
- 当你使用 `/docs` 时，它会检查更新
- 有可用更新时会被拉取
- 此时你可能会看到 "🔄 Updating documentation..."（正在更新文档……）

注意：如果自动更新失败，你随时可以再次运行安装程序以获取最新版本。

## 从旧版本升级

无论你安装的是哪个版本，只需运行：

```bash
curl -fsSL https://raw.githubusercontent.com/long36708/claude-code-docs/main/install.sh | bash
```

安装程序会自动处理迁移和更新。

## 故障排除

### 命令未找到
如果 `/docs` 返回 "command not found"（命令未找到）：
1. 检查命令文件是否存在：`ls ~/.claude/commands/docs.md`
2. 重启 Claude Code 以重新加载命令
3. 重新运行安装脚本

### 文档未更新
如果文档似乎过时：
1. 运行 `/docs -t` 检查同步状态并强制更新
2. 手动更新：`cd ~/.claude-code-docs && git pull`
3. 检查 GitHub Actions 是否正在运行：[查看 Actions](https://github.com/long36708/claude-code-docs/actions)

### 安装错误
- **"git/jq/curl not found"**（找不到 git/jq/curl）：先安装缺失的工具
- **"Failed to clone repository"**（克隆仓库失败）：检查你的网络连接
- **"Failed to update settings.json"**（更新 settings.json 失败）：检查 `~/.claude/settings.json` 的文件权限

## 卸载

要彻底移除文档集成：

```bash
/docs uninstall
```

或运行：
```bash
~/.claude-code-docs/uninstall.sh
```

有关手动卸载说明，请参阅 [UNINSTALL.md](UNINSTALL.md)。

## 安全说明

- 安装程序会修改 `~/.claude/settings.json` 以添加自动更新钩子
- 该钩子仅在读取文档文件时运行 `git pull`
- 所有操作都限制在文档目录内
- 没有数据发送到外部 —— 一切都在本地
- **仓库信任**：安装程序通过 HTTPS 从 GitHub 克隆。为了额外的安全性，你可以：
  - Fork 仓库并从你自己的 fork 安装
  - 手动克隆并从本地目录运行安装程序
  - 在安装前审查所有代码

## 更新内容

### v0.3.3（最新）
- 新增 Claude Code 更新日志集成（`/docs changelog`）
- 修复了 macOS 用户的 shell 兼容性（zsh/bash）
- 改进了文档和错误消息
- 添加了平台兼容性徽章

### v0.3.2
- 修复了自动更新功能
- 改进了对本地仓库变更的处理
- 更新期间更好的错误恢复

## 贡献

**欢迎贡献！** 这是一个社区项目，我们非常需要你的帮助：

- 🪟 **Windows 支持**：想帮忙添加 Windows 兼容性吗？[Fork 仓库](https://github.com/long36708/claude-code-docs/fork)并提交 PR！
- 🐛 **错误报告**：发现某些功能无法正常工作？[提交 issue](https://github.com/long36708/claude-code-docs/issues)
- 💡 **功能请求**：有好点子？[发起讨论](https://github.com/long36708/claude-code-docs/issues)
- 📝 **文档**：帮助改进文档或添加示例

你也可以直接使用 Claude Code 来帮助构建功能 —— 只需 fork 仓库，让 Claude 协助你！

## 已知问题

由于这是一个早期测试版，你可能会遇到一些问题：
- 在某些网络配置下，自动更新可能会偶尔失败
- 某些文档链接可能无法正确解析

如果你发现这里未列出的任何问题，请[报告给我们](https://github.com/long36708/claude-code-docs/issues)！

## 许可证

文档内容归 Anthropic 所有。
本镜像工具为开源项目 —— 欢迎贡献！
