---
title: 云沙箱参考
url: https://platform.claude.com/docs/zh-CN/managed-agents/cloud-sandboxes-reference
description: 云沙箱中可用的预装软件包、数据库和实用工具。
---

云沙箱（cloud sandbox）作为隔离的 Linux 容器运行在 Anthropic 托管的基础设施上。它们预装了一套全面的编程语言、数据库和实用工具。智能体无需任何安装步骤即可立即使用这些工具。

这些规格适用于 `cloud` 环境。自托管沙箱运行在您的基础设施上，使用您的 worker 所提供的任何内容。

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

## 编程语言

| 语言      | 版本                    | 包管理器              |
| ------- | --------------------- | ----------------- |
| Python  | 3.10、3.11、3.12 和 3.13 | pip、uv、poetry     |
| Node.js | 20、21 和 22（默认）        | npm、yarn、pnpm、bun |
| Go      | 1.24（默认）和 1.25        | go modules        |
| Rust    | 稳定版工具链（rustup）        | cargo             |
| Java    | OpenJDK 21            | maven、gradle      |
| Ruby    | 3.1、3.2 和 3.3（默认）     | bundler、gem       |
| PHP     | 8.3                   | composer          |
| C/C++   | GCC 13 和 Clang        | make、cmake、ninja  |

常用的 Python 数据和文档库（包括 NumPy、pandas、Matplotlib、openpyxl、python-docx、python-pptx 和 pypdf）已为 `python3` 解释器安装。

## 数据库

| 数据库           | 描述                                  |
| ------------- | ----------------------------------- |
| PostgreSQL 16 | 已安装服务器和 `psql` 客户端。服务器默认不运行。        |
| Redis 7       | 已安装服务器和 `redis-cli`。服务器默认不运行。       |
| SQLite        | 可通过语言绑定使用，例如 Python 的 `sqlite3` 模块。 |

## 实用工具

### 系统工具

* `git` - 版本控制
* `curl`、`wget` - HTTP 客户端
* `jq`、`yq` - JSON 和 YAML 处理
* `tar`、`zip`、`unzip` - 归档工具
* `tmux` - 终端复用器

### 开发工具

* `make`、`cmake` - 构建系统
* `docker` - 容器管理（可用性有限）
* `ripgrep`（`rg`）- 快速文件搜索

### 文本处理

* `sed`、`awk`、`grep` - 流编辑器
* `vim`、`nano` - 文本编辑器
* `diff`、`patch` - 文件比较

### 文档和媒体处理

* `ffmpeg` - 音频和视频处理
* ImageMagick（`convert`、`identify`）- 图像处理
* `pandoc` - 文档转换
* LibreOffice（无头模式）- Office 文档转换
* Poppler 实用工具（`pdftotext`、`pdftoppm`）和 `qpdf` - PDF 处理
* `tesseract` - 光学字符识别（英语语言数据）
* TeX Live（`pdflatex`、`xelatex`、`latexmk`）- 排版

### 浏览器自动化

* Playwright（Python 和 Node.js）- 浏览器自动化库
* Chromium（`/opt/pw-browsers/chromium`）- Playwright 使用的浏览器，不在 `PATH` 中

沙箱将 `PLAYWRIGHT_BROWSERS_PATH` 设置为 `/opt/pw-browsers`，因此预装的 Playwright 软件包无需配置即可在该位置找到 Chromium。Python 软件包已为 `python3` 解释器安装。请使用预装的软件包，而不要安装其他版本的 Playwright，否则它会查找并不存在的浏览器构建版本。Firefox 和 WebKit 未安装。

## 沙箱规格

| 属性   | 值                                                                                                                                                        |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 操作系统 | Ubuntu 24.04 LTS                                                                                                                                         |
| 架构   | x86\_64 (amd64)                                                                                                                                          |
| 内存   | 最高 8 GB                                                                                                                                                  |
| 磁盘空间 | 最高 10 GB                                                                                                                                                 |
| 网络   | 通过 API 创建的环境默认使用 [`unrestricted` 网络](https://platform.claude.com/docs/zh-CN/managed-agents/environments#networking)；通过 Claude Studio 配置的沙箱默认使用 `limited` |
