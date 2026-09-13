---
title: 安全模型
url: https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes-security
description: 自托管沙箱环境的共同责任模型。
---

Anthropic 负责保护所有环境中的控制平面：会话和工作队列的完整性、多租户隔离以及智能体上下文最小化。当您自托管时，以下责任由您承担。

## 您负责的部分

* **沙箱镜像质量和运行时加固。** Anthropic 不会检查或验证您的沙箱镜像。请遵循最佳实践，例如移除不必要的 Linux capabilities（能力）、以非 root 用户运行，以及使用只读根文件系统。
* **网络出站控制。** 您的沙箱的网络访问由您的 VPC 和防火墙规则决定。如果没有出站限制，被攻陷的工具执行可以访问任意外部主机。请将出站流量限制为仅您的工具所需的端点。
* **服务密钥的存储和轮换。** 环境服务密钥（`ANTHROPIC_ENVIRONMENT_KEY`）授权轮询您环境的工作队列并将结果提交回会话。请将其存储在密钥管理器中，而不是环境文件或沙箱镜像中。如果您怀疑密钥已泄露，请立即轮换。
* **隔离不受信任的工作负载。** 环境服务密钥的作用范围限定于单个环境的工作队列。如果您在沙箱内运行不受信任的代码，请考虑为每个信任边界配置单独的工作区和环境。这样可以将每个密钥限制为单个用户的会话，而不是共享池。
* **每会话凭据。** 您的 worker 领取的每个工作项都可以携带一个每会话的 `secret`，SDK worker 会使用它来代替环境服务密钥。访问[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory)需要 `secret`：记忆存储端点会拒绝环境密钥（请参阅[使用记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)）。请仅将 `secret` 传入为该会话提供服务的沙箱，不要将其放入镜像和共享卷中，并且绝不要将其记录到日志中。
* **工具执行的影响范围。** 工具在您的沙箱内以您的进程所拥有的任何权限运行。请对进程用户应用最小权限原则，并仅挂载您的工具所需的目录。
* **日志保留和会话内容。** 对话内容和工具输出会经过您的 worker 并保留在您的环境中。您有责任根据您自己的策略保留、脱敏或删除这些数据。一旦会话内容交付，Anthropic 无法了解您的 worker 如何处理这些内容。
* **记忆存储内容。** [记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory)仍由 Anthropic 托管，包括其版本历史。当会话附加一个记忆存储时，worker 会在会话期间在您沙箱中的 `/mnt/memory/` 下保留一份工作副本，并将更改同步回去。worker 会在会话结束时删除该副本，但如果 worker 在未运行其清理流程的情况下退出，则会将其遗留下来。清理遗留副本、该路径上的权限以及共享文件系统的会话之间的隔离均由您负责。
* **只读记忆存储。** 以 `read_only` 访问权限附加的存储受到的保护是防止上传，而非防止本地修改。worker 的 `write` 和 `edit` 工具会拒绝在其目录下写入，该目录下的任何内容都不会同步回去，并且记忆存储端点会拒绝使用会话的 `secret` 对其进行的写入。沙箱中的其他进程仍然可以更改本地副本：智能体通过 `bash` 工具运行的命令，以及您从沙箱提供的[自定义工具](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#serve-custom-tools-from-your-sandbox)或 MCP 服务器，它们以 worker 的权限运行。该会话中后续的工具调用会读取已更改的副本，直到该记忆在存储中下一次发生变化。如果智能体必须无法更改此类存储的本地视图，请为该智能体禁用 `bash` 工具，并且不要为其提供任何会写入沙箱文件系统的自定义工具。

## Anthropic 无法为您做的事情

* **知道您的密钥已泄露。** Anthropic 可以检测异常的使用模式，但无法知道您的密钥已被泄露。如果您怀疑 `ANTHROPIC_ENVIRONMENT_KEY` 已泄露，请立即撤销它并生成替换密钥。撤销会在每次请求时进行验证，因此它会在 worker 的下一次调用时生效。
* **验证您的 worker 构建。** Anthropic 不会检查您的沙箱镜像或运行时。您镜像中的供应链攻陷无法从控制平面检测到。
* **隔离您沙箱内的工具。** Anthropic 的安全边界止于沙箱。在该边界内如何将各个工具执行相互隔离完全由您负责。
* **在您的环境中强制执行数据保留。** 一旦会话内容到达您的 worker，它就超出了 Anthropic 的数据生命周期控制范围。
