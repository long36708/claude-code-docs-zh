---
title: 自托管沙箱
url: https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes
description: 在自托管沙箱中运行 Claude Managed Agents 会话，将工具执行、文件和网络出口保留在您自己的基础设施中。
---

默认情况下，Managed Agents 在 [Anthropic 托管的云沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/cloud-sandboxes-reference)中执行工具和代码。"Self-hosted sandboxes"（自托管沙箱）将编排保留在 Anthropic 一侧，但将工具执行移至您控制的基础设施中，因此智能体的代码、文件系统和网络出口永远不会离开您的环境。

工具执行保留在您的主机上：智能体读写的文件系统、它生成的进程以及它可以访问的网络都在您的控制之下。工具输入和输出仍会流向 Anthropic 的控制平面（Claude 运行的地方），以便模型能够看到结果并决定下一步做什么。智能体的[技能](https://platform.claude.com/docs/zh-CN/managed-agents/skills)以及附加到会话的任何[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory)的内容由 Anthropic 存储，并在会话期间复制到您的沙箱中；智能体对记忆文件所做的更改会同步回存储。有关完整的数据流边界，请参阅[安全模型](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes-security)。

<Note>
  自托管沙箱支持 Managed Agents 中可用的所有 Claude 模型，包括 Claude Opus 4.8 和 Claude Opus 5。模型在[智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)上配置，而不是在环境上配置。
</Note>

## 与云环境的区别

|                 | 云环境                            | 自托管沙箱                               |
| --------------- | ------------------------------ | ----------------------------------- |
| 工具运行位置          | Anthropic 托管的沙箱                | 您的基础设施                              |
| 网络可达范围          | Anthropic 的出口控制                | 您的网络策略                              |
| 文件和 GitHub 仓库挂载 | 由 Anthropic 管理                 | 由您管理                                |
| 记忆存储            | 由 Anthropic 挂载到 `/mnt/memory/` | 下载到 `/mnt/memory/` 并由 SDK worker 同步 |
| 生命周期            | 由 Anthropic 管理                 | 由您管理                                |

当智能体需要处理不能离开您网络边界的数据、访问不可公开路由的内部服务，或在您组织自己的合规和审计控制下运行时，自托管是一个很好的选择。

有关零数据保留和 HIPAA BAA 资格，请参阅 [API 和数据保留](https://platform.claude.com/docs/zh-CN/manage-claude/api-and-data-retention#feature-eligibility)。

## 何时与 MCP 隧道结合使用

自托管控制的是*智能体代码在哪里执行*。[MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview)控制的是 *Anthropic 如何访问您网络中的 MCP 服务器*。两者相互独立：在 Anthropic 云沙箱中运行的会话仍然可以通过隧道访问私有 MCP 服务器，而自托管会话可以使用隧道化的或公开的 MCP 服务器。当您希望执行和工具访问都保留在您的边界内时，请同时使用两者。若要在不运行隧道的情况下为智能体提供来自您网络内部 MCP 服务器的工具，您也可以[将该服务器包装为自定义工具](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#wrap-an-mcp-server-as-custom-tools)，由您的 worker 提供服务。

## 环境 worker

<Tip>
  本指南介绍如何使用任何通用沙箱平台构建 worker。另有针对特定平台的指南，适用于 [AWS Lambda MicroVMs](https://docs.aws.amazon.com/lambda/latest/dg/microvms-integrations-claude-managed-agents.html)、[Blaxel](https://docs.blaxel.ai/Tutorials/Claude-Managed-Agents)、[Cloudflare](https://developers.cloudflare.com/sandbox/claude-managed-agents/)、[Daytona](https://www.daytona.io/docs/en/guides/claude/claude-managed-agents)、[E2B](https://e2b.dev/docs/agents/claude-managed-agents)、[Fly.io](https://docs.sprites.dev/integrations/claude-managed-agents/)、[GKE Agent Sandbox](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/main/ai-ml/anthropic-agent-sandbox)、[Modal](https://github.com/modal-labs/claude-managed-agents-modal-sandbox)、[Namespace](https://namespace.so/docs/integrations/claude)、[Superserve](https://docs.superserve.ai/integrations/managed-agents/claude-managed-agents) 和 [Vercel](https://vercel.com/kb/guide/run-claude-managed-agent-tools-with-vercel-sandbox)。
</Tip>

"Environment worker"（环境 worker）是您在自己的基础设施上运行的进程。它接收来自 Anthropic 的工具执行请求并在本地运行它们。`self_hosted` 环境充当一个工作队列：当一个[会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)被分配给它时，Anthropic 会将该会话作为工作项入队。您的 worker 从该队列中认领工作项，为每个工作项生成一个执行上下文，下载智能体的[技能](https://platform.claude.com/docs/zh-CN/managed-agents/skills)（可复用的、基于文件系统的资源，为智能体提供特定领域的专业知识），运行工具调用，并将结果回传。

工作项通过轮询环境的队列来认领：要么由持续轮询的**常驻 worker** 认领，要么由在 `session.status_run_started` 时被唤醒并开始轮询的 **webhook 触发的处理程序**认领。

CLI 和 SDK 都附带预构建的 worker。`ant` CLI 仅支持常驻模式；SDK 同时支持常驻模式和 webhook 触发模式。两者都可配置：有关 CLI 标志，请参阅参考文档中的[自托管 worker](https://platform.claude.com/docs/zh-CN/managed-agents/reference#self-hosted-worker)；有关 SDK 选项，请参阅本页的 [SDK 辅助工具](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#sdk-helpers)。若需要更多控制，请直接调用 [Environments Work 端点](https://platform.claude.com/docs/zh-CN/api/beta/environments/work)并实现您自己的 worker。

### 沙箱文件系统

* **`/workspace`：** 工具执行和技能下载的系统默认工作目录。CLI 的 `--workdir` 标志默认为当前目录；传入 `--workdir /workspace` 以匹配系统默认值。技能会下载到 `<workdir>/skills/<name>/`。如果您使用不同的工作目录，请更新智能体的系统提示，以便 Claude 能够找到技能文件。
* **输出：** 在自托管环境中，会话的系统提示省略了 Anthropic 托管沙箱上使用的 `/mnt/session/outputs` 指令，因此最终交付物会落在智能体在您的沙箱文件系统中写入它们的任何位置，通常在工作目录下。
* **`/mnt/memory/`：** 附加到会话的记忆存储由 SDK worker 在此处实体化，每个存储一个目录，位于该存储的 `mount_path`（例如 `/mnt/memory/user-preferences/`）。worker 在认领会话时创建这些目录，并在会话结束时删除它们；请参阅[使用记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)。

## 开始之前

您需要：

* **一个现有的智能体。** 如果您还没有，请先完成[快速入门](https://platform.claude.com/docs/zh-CN/managed-agents/quickstart)并记下其智能体 ID。
* **一台 Linux 主机**，且 `/bin/bash` 位于该确切路径。worker 的 bash 工具直接调用它，而不查询 `PATH`。TypeScript SDK 还要求 `PATH` 上有 `unzip` 和 `tar`，以及 Node.js 22 或更高版本；Python 和 Go SDK 使用其标准库进行归档解压，没有额外的二进制要求。
* **worker 主机上的 `ant` CLI 或 Anthropic SDK**（Python、TypeScript 或 Go）。
* **凭据：** 环境密钥（在后续步骤中于 Console 中生成）用于向其队列验证 worker 的身份；您的 Claude API 密钥用于从 worker 主机外部创建会话和读取队列统计信息。密钥生成仅限于 Console。已认领的工作项还携带一个每会话的 `secret`，worker 用它来挂载[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)；您无需生成它，但在每会话一个沙箱的模式中，您需要自行将其转发到沙箱中（请参阅[每个会话运行一个沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#run-one-sandbox-per-session)）。
* **对于记忆存储，一台已准备好的主机。** 如果此环境上的会话将附加记忆存储，请在启动 worker 之前在 worker 主机上准备好 `/mnt/memory`；请参阅[准备主机](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#prepare-the-host)。

<Note>
  在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上，worker 使用 AWS IAM（SigV4）或[在 AWS Console 中生成的 API 密钥](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#api-key-authentication)进行身份验证，而不是环境密钥。请将 [`AnthropicSelfHostedEnvironmentAccess`](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#managed-policies) 托管策略附加到您的 worker 运行所用的 IAM 主体。在 Claude Console 中生成的环境密钥不适用于 Claude Platform on AWS 端点。

  在 Claude Platform on AWS 上，记忆存储无法附加到自托管环境上的会话。
</Note>

<Steps>
  <Step title="创建自托管环境">
    在 [Console](https://platform.claude.com/workspaces/default/environments) 中：**Workspace > Environments > New > Self-hosted**

    或通过 API：

    <CodeGroup>
      ```bash cURL
      curl -sS --fail-with-body https://api.anthropic.com/v1/environments \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "anthropic-beta: managed-agents-2026-04-01" \
        -H "content-type: application/json" \
        -d '{
          "name": "self-hosted",
          "config": {"type": "self_hosted"}
        }'
      ```

      <MultiFileExample language="cli" label="CLI">
        ```bash CLI
        ant beta:environments create < environment.yaml
        ```

        <File filename="environment.yaml">
          ```yaml
          name: self-hosted
          config:
            type: self_hosted
          ```
        </File>
      </MultiFileExample>

      ```python Python
      client = anthropic.Anthropic()

      environment = client.beta.environments.create(
          name="self-hosted", config={"type": "self_hosted"}
      )
      print(environment.id)
      ```

      ```typescript TypeScript
      const client = new Anthropic();

      const environment = await client.beta.environments.create({
        name: "self-hosted",
        config: { type: "self_hosted" }
      });
      console.log(environment.id);
      ```

      ```csharp C#
      using Anthropic.Models.Beta.Environments;

      var client = new AnthropicClient();

      var environment = await client.Beta.Environments.Create(
          new EnvironmentCreateParams
          {
              Name = "self-hosted",
              Config = new BetaSelfHostedConfigParams(),
          }
      );
      Console.WriteLine(environment.ID);
      ```

      ```go Go
      client := anthropic.NewClient()

      environment, err := client.Beta.Environments.New(context.Background(), anthropic.BetaEnvironmentNewParams{
      	Name: "self-hosted",
      	Config: anthropic.BetaEnvironmentNewParamsConfigUnion{
      		OfSelfHosted: &anthropic.BetaSelfHostedConfigParams{},
      	},
      })
      if err != nil {
      	panic(err)
      }
      fmt.Println(environment.ID)
      ```

      ```java Java
      import com.anthropic.models.beta.environments.BetaSelfHostedConfigParams;
      import com.anthropic.models.beta.environments.EnvironmentCreateParams;

      void main() {
          var client = AnthropicOkHttpClient.fromEnv();

          var environment = client.beta().environments().create(
              EnvironmentCreateParams.builder()
                  .name("self-hosted")
                  .config(BetaSelfHostedConfigParams.builder().build())
                  .build()
          );
          IO.println(environment.id());
      }
      ```

      ```php PHP
      $client = new Anthropic\Client();

      $environment = $client->beta->environments->create(
          name: 'self-hosted',
          config: ['type' => 'self_hosted'],
      );
      echo $environment->id, PHP_EOL;
      ```

      ```ruby Ruby
      client = Anthropic::Client.new

      environment = client.beta.environments.create(
        name: "self-hosted",
        config: {type: :self_hosted}
      )
      puts environment.id
      ```
    </CodeGroup>
  </Step>

  <Step title="生成环境密钥">
    在 Console 中，打开该环境并点击 **Generate environment key**。无论您是通过 Console 还是 API 创建的环境，密钥生成都仅限于 Console。然后在 worker 主机上导出环境 ID 和密钥：

    ```bash
    export ANTHROPIC_ENVIRONMENT_KEY="sk-ant-oat01-..."
    export ANTHROPIC_ENVIRONMENT_ID="env_..."
    ```
  </Step>
</Steps>

<Note>
  技能可以包含智能体可能直接运行的可执行文件。CLI 和 SDK worker 在解压技能包时会保留其中记录的可执行权限。如果您手动实现技能下载，则需要自行负责设置可执行权限。
</Note>

## 运行 worker

选择**常驻模式**以获得最简单的设置：一个长期运行的进程持续轮询队列，只需要出站 HTTPS。选择 **webhook 触发模式**以避免运行空闲的轮询器；它需要一个 Anthropic 可以访问的 webhook 端点（有关端点设置和签名验证，请参阅 [Webhooks](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks)）。

<Tabs>
  <Tab title="常驻模式（ant CLI）">
    <Steps>
      <Step title="安装 ant CLI">
        在 worker 主机上运行此命令。

        <Tabs>
          <Tab title="curl（Linux/WSL）">
            对于 Linux 环境，直接下载发布的二进制文件。

            ```bash
            VERSION=1.27.0
            OS=$(uname -s | tr '[:upper:]' '[:lower:]')
            case $(uname -m) in
              x86_64) ARCH=amd64 ;;
              aarch64) ARCH=arm64 ;;
            esac
            curl -fsSL "https://github.com/anthropics/anthropic-cli/releases/download/v${VERSION}/ant_${VERSION}_${OS}_${ARCH}.tar.gz" \
              | sudo tar -xz -C /usr/local/bin ant
            ```

            您可以在 [GitHub 发布页面](https://github.com/anthropics/anthropic-cli/releases)上找到所有版本。
          </Tab>

          <Tab title="Homebrew（macOS）">
            ```bash
            brew install anthropics/tap/ant
            ```
          </Tab>
        </Tabs>
      </Step>

      <Step title="运行 worker">
        **进程内**

        `ant beta:worker poll` 认领分配给该环境的工作项，下载技能，在工作目录中执行工具调用，并将结果回传。它从环境变量中读取 `ANTHROPIC_ENVIRONMENT_KEY` 和 `ANTHROPIC_ENVIRONMENT_ID`。

        ```bash
        ant beta:worker poll --workdir "/workspace"
        ```

        worker 在收到 SIGTERM 或 SIGINT 时会干净地退出：它会取消任何进行中的工具调用，发布其错误结果，并在停止前释放工作项。

        **每个会话一个沙箱**

        如果您需要更强的隔离（全新的文件系统、资源限制或每会话的网络控制），请在各自独立的沙箱中运行每个会话。构建一个安装了 `ant` 并以 `ant beta:worker run` 作为入口点的镜像。基础镜像必须提供 `/bin/bash`；`curl` 仅在构建时使用。沙箱启动时，它从环境变量中读取会话详情，处理该会话，然后退出：

        ```text
        FROM your-base-image
        ARG ANT_VERSION=1.27.0
        ARG TARGETARCH
        RUN ARCH=$([ "$TARGETARCH" = "arm64" ] && echo arm64 || echo amd64) && \
            curl -fsSL "https://github.com/anthropics/anthropic-cli/releases/download/v${ANT_VERSION}/ant_${ANT_VERSION}_linux_${ARCH}.tar.gz" \
              | tar -xz -C /usr/local/bin ant
        WORKDIR /workspace
        VOLUME /workspace
        ENTRYPOINT ["ant", "beta:worker", "run"]
        ```

        然后编写一个生成脚本，将会话详情转发到一个全新的沙箱中。轮询器将 `ANTHROPIC_SESSION_ID`、`ANTHROPIC_WORK_ID`、`ANTHROPIC_ENVIRONMENT_ID` 和 `ANTHROPIC_ENVIRONMENT_KEY` 注入脚本的环境中，并将已认领的工作项以 JSON 形式写入脚本的标准输入，其中包括工作项的每会话 `secret`（当 Anthropic 签发了该 secret 时）。`ANTHROPIC_BASE_URL` 是可选的，仅当它在轮询器主机上已设置时才会传递；它会覆盖默认的 API 端点。在示例中，`/host/outputs` 是您选择的主机目录；它被绑定挂载到沙箱的工作目录（`/workspace`），以便您可以在沙箱退出后取回会话交付物。在自托管环境中，智能体将交付物写入工作目录下而不是 `/mnt/session/outputs`（请参阅[沙箱文件系统](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#sandbox-filesystem)），因此挂载工作目录即可捕获它们；该挂载还会包含下载的 `skills/` 目录树以及智能体创建的任何中间文件。

        ```bash
        #!/bin/bash
        # spawn.sh：每个已认领的工作项调用一次
        mkdir -p "/host/outputs/$ANTHROPIC_SESSION_ID"
        exec docker run --rm \
          -e ANTHROPIC_SESSION_ID -e ANTHROPIC_ENVIRONMENT_KEY \
          -e ANTHROPIC_WORK_ID -e ANTHROPIC_ENVIRONMENT_ID -e ANTHROPIC_BASE_URL \
          -v "/host/outputs/$ANTHROPIC_SESSION_ID":/workspace \
          your-image
        ```

        `ant beta:worker run` 入口点不会挂载[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)。如果此环境上的会话附加了记忆存储，请保留轮询器，但围绕 SDK worker 构建每会话镜像，并扩展生成脚本以将工作项的 `secret` 转发到沙箱中，如[每个会话运行一个沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#run-one-sandbox-per-session)中所示。

        启动指向该脚本的轮询器：

        ```bash
        ant beta:worker poll --on-work ./spawn.sh
        ```
      </Step>
    </Steps>
  </Tab>

  <Tab title="常驻模式（SDK）">
    <Steps>
      <Step title="运行 worker">
        `EnvironmentWorker` 认领分配给该环境的工作项，下载技能，在工作目录中执行工具调用，并将结果回传。使用您在[开始之前](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#before-you-begin)中生成的环境密钥进行身份验证。

        <CodeGroup exclude="shell">
          ```python Python
          import asyncio
          import contextlib
          import os
          import signal
          from anthropic import AsyncAnthropic
          from anthropic.lib.environments import EnvironmentWorker


          async def main() -> None:
              environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
              environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
              async with AsyncAnthropic(auth_token=environment_key) as client:
                  worker = EnvironmentWorker(
                      client,
                      environment_id=environment_id,
                      environment_key=environment_key,
                      workdir="/workspace",
                  )
                  task = asyncio.create_task(worker.run())
                  # 取消任务（而非终止进程）可让 worker 停止其
                  # 正在处理的工作项，并在退出前上传已更改的内存文件。
                  loop = asyncio.get_running_loop()
                  for signum in (signal.SIGINT, signal.SIGTERM):
                      loop.add_signal_handler(signum, task.cancel)
                  with contextlib.suppress(asyncio.CancelledError):
                      await task


          asyncio.run(main())
          ```

          ```typescript TypeScript
          import Anthropic from "@anthropic-ai/sdk";
          import { EnvironmentWorker } from "@anthropic-ai/sdk/helpers/beta/environments";

          const environmentKey = process.env.ANTHROPIC_ENVIRONMENT_KEY!;
          const environmentId = process.env.ANTHROPIC_ENVIRONMENT_ID!;
          const client = new Anthropic({ authToken: environmentKey });
          const controller = new AbortController();
          // 在任一信号上中止可让 worker 上传已更改的内存文件并移除其
          // 存储目录，然后进程退出。
          process.once("SIGINT", () => controller.abort());
          process.once("SIGTERM", () => controller.abort());

          await new EnvironmentWorker({
            client,
            environmentId,
            environmentKey,
            workdir: "/workspace",
            signal: controller.signal
          }).run();
          ```

          ```csharp C#
          // EnvironmentWorker 目前在 C# SDK 中不可用。请参阅 Always-on (ant CLI) 选项卡。
          ```

          ```go Go
          package main

          import (
          	"context"
          	"log"
          	"os"
          	"os/signal"
          	"syscall"

          	"github.com/anthropics/anthropic-sdk-go"
          	"github.com/anthropics/anthropic-sdk-go/lib/environments"
          	"github.com/anthropics/anthropic-sdk-go/option"
          )

          func main() {
          	environmentKey := os.Getenv("ANTHROPIC_ENVIRONMENT_KEY")
          	environmentID := os.Getenv("ANTHROPIC_ENVIRONMENT_ID")

          	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
          	defer stop()

          	client := anthropic.NewClient(option.WithAuthToken(environmentKey))

          	worker := environments.NewEnvironmentWorker(client, environments.EnvironmentWorkerOptions{
          		EnvironmentID:  environmentID,
          		EnvironmentKey: environmentKey,
          		Workdir:        "/workspace",
          	})
          	if err := worker.Run(ctx); err != nil {
          		log.Fatalf("worker: %v", err)
          	}
          }

          ```

          ```java Java
          // Java SDK 目前不支持 EnvironmentWorker。请参阅 Always-on (ant CLI) 选项卡。
          ```

          ```php PHP
          // PHP SDK 目前不提供 EnvironmentWorker。请参阅 Always-on (ant CLI) 选项卡。
          ```

          ```ruby Ruby
          # Ruby SDK 目前不支持 EnvironmentWorker。请参阅 Always-on (ant CLI) 选项卡。
          ```
        </CodeGroup>
      </Step>
    </Steps>
  </Tab>

  <Tab title="Webhook 触发模式（SDK）">
    <Steps>
      <Step title="订阅会话 webhook">
        在 [Console](https://platform.claude.com/settings/workspaces/default/webhooks) 中，定义一个监听 `session.status_run_started` 事件的 webhook 端点。详情请参阅 [Webhooks](https://platform.claude.com/docs/zh-CN/managed-agents/webhooks)。
      </Step>

      <Step title="导出 webhook 签名密钥">
        除了[开始之前](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#before-you-begin)中的环境 ID 和密钥之外，还需在处理程序主机上导出 webhook 签名密钥，以便处理程序可以验证传入的负载。Python 处理程序中的签名验证需要 webhooks 扩展：`pip install "anthropic[webhooks]"`。

        ```bash
        export ANTHROPIC_WEBHOOK_SIGNING_KEY="whsec_..."
        ```
      </Step>

      <Step title="实现 webhook 处理程序">
        `EnvironmentWorker` 认领工作项，下载技能，在工作目录中执行工具调用，将结果回传，然后退出。在 `session.status_run_started` 触发时调用它。

        当您像此处理程序一样自行将已认领的工作项交给 `handle_item()` 时，请将工作项的 `secret` 作为 `work_secret`（TypeScript 中为 `workSecret`，Go 中为 `WorkSecret`）一并传递，以便会话可以挂载附加到它的任何[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)。像这样的处理程序在一台主机上的一个进程中运行每个已认领的项，因此附加同一记忆存储的两个会话不能同时通过它运行（请参阅[准备主机](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#prepare-the-host)）；如果您的会话共享存储，请改为启动[每个会话一个沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#run-one-sandbox-per-session)。

        <CodeGroup exclude="shell">
          ```python Python
          import asyncio
          import os
          import anthropic
          import standardwebhooks  # installed by the anthropic[webhooks] extra

          environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
          environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
          client = anthropic.AsyncAnthropic(
              auth_token=environment_key,
          )
          # 由 shutdown() 取消，以便正在进行的工作项能够上传已更改的内存文件，
          # 并在进程退出前移除其存储目录。
          inflight: set[asyncio.Task[None]] = set()


          # 请在宿主的关闭钩子中 await 此方法，例如 ASGI lifespan 关闭阶段（FastAPI lifespan 中
          # `yield` 之后的代码），uvicorn 会在收到 SIGTERM 时运行它。uvicorn 会让未完成的请求
          # 在该钩子运行前结束，因此请设置 --timeout-graceful-shutdown 以限制等待时间。
          async def shutdown() -> None:
              for task in inflight:
                  task.cancel()
              await asyncio.gather(*inflight, return_exceptions=True)


          async def handle(raw: bytes, headers: dict[str, str]) -> tuple[dict[str, str], int]:
              try:
                  event = client.beta.webhooks.unwrap(raw.decode(), headers=headers)
              except standardwebhooks.WebhookVerificationError:
                  return {"error": "signature verification failed"}, 401
              if event.data.type != "session.status_run_started":
                  return {"status": "ignored"}, 200
              task = asyncio.create_task(run_queued_work())
              inflight.add(task)
              task.add_done_callback(inflight.discard)
              try:
                  # 已屏蔽（shield）：投递被丢弃或超时不得取消该工作项；由 shutdown() 负责取消。
                  await asyncio.shield(task)
              except asyncio.CancelledError:
                  return {"status": "shutting down"}, 503
              return {"status": "ok"}, 200


          async def run_queued_work() -> None:
              async for work in client.beta.environments.work.poller(
                  environment_id=environment_id,
                  environment_key=environment_key,
                  block_ms=None,
                  reclaim_older_than_ms=2000,
                  drain=True,
                  auto_stop=False,
              ):
                  await client.beta.environments.work.worker(workdir="/workspace").handle_item(
                      work_id=work.id,
                      environment_id=environment_id,
                      session_id=work.data.id,
                      environment_key=environment_key,
                      # 每会话密钥使 worker 能够挂载该会话的内存存储。
                      work_secret=work.secret,
                  )
          ```

          ```typescript TypeScript
          import Anthropic from "@anthropic-ai/sdk";

          const environmentKey = process.env.ANTHROPIC_ENVIRONMENT_KEY!;
          const environmentId = process.env.ANTHROPIC_ENVIRONMENT_ID!;
          const client = new Anthropic({
            authToken: environmentKey
          });
          // 在宿主的 SIGTERM/SIGINT 处理程序中调用 shutdown.abort()，同时关闭服务器，
          // 然后在退出前等待进行中的 handle() 调用完成：abort 可让正在运行的工作项
          // 先上传已更改的内存文件并移除其存储目录。
          export const shutdown = new AbortController();

          export async function handle(req: Request): Promise<Response> {
            // 切勿确认其工作不会在此处运行的投递；返回 503 会让发送方重试。
            if (shutdown.signal.aborted) {
              return Response.json({ status: "shutting down" }, { status: 503 });
            }
            const body = await req.text();
            let event;
            try {
              event = client.beta.webhooks.unwrap(body, { headers: Object.fromEntries(req.headers) });
            } catch {
              return new Response("signature verification failed", { status: 401 });
            }
            if (event.data.type !== "session.status_run_started") {
              return Response.json({ status: "ignored" });
            }

            for await (const work of client.beta.environments.work.poller({
              environmentId,
              environmentKey,
              blockMs: null,
              reclaimOlderThanMs: 2000,
              drain: true,
              autoStop: false,
              signal: shutdown.signal
            })) {
              await client.beta.environments.work.worker({ workdir: "/workspace" }).handleItem({
                workId: work.id,
                environmentId,
                sessionId: work.data.id,
                environmentKey,
                // 每会话 secret 正是让 worker 能够挂载该会话内存存储的凭据。
                workSecret: work.secret ?? undefined,
                signal: shutdown.signal
              });
            }
            // 轮询器和 handleItem 在 abort 时会静默返回，因此被中断的排空操作会落到这里。
            if (shutdown.signal.aborted) {
              return Response.json({ status: "shutting down" }, { status: 503 });
            }
            return Response.json({ status: "ok" });
          }
          ```

          ```csharp C#
          // EnvironmentWorker 目前在 C# SDK 中不可用。
          // 如需直接处理工作项，请参阅 Environments Work 端点。
          ```

          ```go Go
          package main

          import (
          	"context"
          	"encoding/json"
          	"errors"
          	"io"
          	"log/slog"
          	"net/http"
          	"os"
          	"os/signal"
          	"syscall"

          	"github.com/anthropics/anthropic-sdk-go"
          	"github.com/anthropics/anthropic-sdk-go/lib/environments"
          	"github.com/anthropics/anthropic-sdk-go/option"
          	"github.com/anthropics/anthropic-sdk-go/packages/param"
          )

          var (
          	environmentKey = os.Getenv("ANTHROPIC_ENVIRONMENT_KEY")
          	environmentID  = os.Getenv("ANTHROPIC_ENVIRONMENT_ID")
          	client         = anthropic.NewClient(
          		option.WithAuthToken(environmentKey),
          		option.WithWebhookKey(os.Getenv("ANTHROPIC_WEBHOOK_SIGNING_KEY")),
          	)
          	worker = environments.NewEnvironmentWorker(client, environments.EnvironmentWorkerOptions{
          		Workdir: "/workspace",
          	})
          	// 在 SIGINT 或 SIGTERM 时取消（在 main 中设置），以便进行中的工作项能够
          	// 在退出前上传已更改的内存文件并移除其存储目录。
          	shutdown context.Context
          )

          func handle(w http.ResponseWriter, r *http.Request) {
          	body, err := io.ReadAll(r.Body)
          	if err != nil {
          		http.Error(w, "bad request", http.StatusBadRequest)
          		return
          	}
          	event, err := client.Beta.Webhooks.Unwrap(body, r.Header)
          	if err != nil {
          		http.Error(w, "signature verification failed", http.StatusUnauthorized)
          		return
          	}
          	if event.Data.Type != "session.status_run_started" {
          		json.NewEncoder(w).Encode(map[string]string{"status": "ignored"})
          		return
          	}

          	// Go SDK 不提供 RunOne 便捷方法：使用 WorkPoller 排空待处理项，
          	// 并使用 HandleItem 逐个运行。
          	// 与 r.Context() 分离：会话的存活时间可能超过 webhook 投递超时。
          	// 进程级的关闭上下文仍会在 SIGTERM 时干净地结束该工作项。
          	ctx := shutdown
          	poller := environments.NewWorkPoller(ctx, client, environments.WorkPollerOptions{
          		EnvironmentID:      environmentID,
          		EnvironmentKey:     environmentKey,
          		BlockMs:            param.Null[int64](),
          		ReclaimOlderThanMs: param.NewOpt[int64](2000),
          		Drain:              true,
          		AutoStop:           param.NewOpt(false),
          	})
          	defer poller.Close()
          	for poller.Next() {
          		item := poller.Current()
          		if err := worker.HandleItem(ctx, environments.HandleItemOptions{
          			WorkID:         item.ID,
          			EnvironmentID:  item.EnvironmentID,
          			SessionID:      item.Data.ID,
          			EnvironmentKey: environmentKey,
          			// 每会话密钥使 worker 能够挂载该会话的内存存储。
          			WorkSecret: item.Secret,
          		}); err != nil {
          			slog.Error("handle work item", "work_id", item.ID, "err", err)
          			http.Error(w, "internal error", http.StatusInternalServerError)
          			return
          		}
          	}
          	if err := poller.Err(); err != nil {
          		slog.Error("poll work queue", "err", err)
          		http.Error(w, "internal error", http.StatusInternalServerError)
          		return
          	}
          	json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
          }

          func main() {
          	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
          	defer stop()
          	shutdown = ctx

          	server := &http.Server{Addr: ":8080"}
          	http.HandleFunc("POST /webhook", handle)
          	go func() {
          		if err := server.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
          			slog.Error("http server", "err", err)
          			os.Exit(1)
          		}
          	}()
          	// 收到信号时，停止接受投递，并且仅在进行中的
          	// 处理程序（及其工作项的内存清理）完成后才返回。
          	<-ctx.Done()
          	if err := server.Shutdown(context.Background()); err != nil {
          		slog.Error("http shutdown", "err", err)
          	}
          }

          ```

          ```java Java
          // Java SDK 目前不提供 EnvironmentWorker。
          // 如需直接处理工作项，请参阅 Environments Work 端点。
          ```

          ```php PHP
          // PHP SDK 目前暂不提供 EnvironmentWorker。
          // 如需直接处理工作项，请参阅 Environments Work 端点。
          ```

          ```ruby Ruby
          # Ruby SDK 目前不提供 EnvironmentWorker。
          # 如需直接处理工作项，请参阅 Environments Work 端点。
          ```
        </CodeGroup>
      </Step>
    </Steps>
  </Tab>
</Tabs>

### SDK 辅助工具

SDK 提供三个不同控制级别的辅助工具。`EnvironmentWorker` 涵盖大多数用例；当您需要启动自己的每会话进程或针对已认领的会话运行工具时，请使用更底层的辅助工具。

* **`EnvironmentWorker`：** 开箱即用的 worker。端到端处理轮询、设置和执行。

  * `.run()`：无限期运行，在会话到达时接手处理。
  * `.handle_item()`：处理单个已认领的工作项并退出。显式传递工作、会话和环境标识符，或让它读取 `ant beta:worker poll --on-work` 为其生成的进程设置的 `ANTHROPIC_*` 变量。若要让会话挂载其[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)，还需将工作项的 `secret` 作为 `work_secret`（TypeScript 中为 `workSecret`，Go 中为 `WorkSecret`）传递，或设置 `ANTHROPIC_WORK_SECRET`；`ant beta:worker poll --on-work` 不会设置该变量，因此请从它写入您脚本标准输入的工作项 JSON 中读取 secret，如[每个会话运行一个沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#run-one-sandbox-per-session)中所示。
  * `memory_sync_interval`（TypeScript 中为 `memorySyncIntervalMs`，Go 中为 `MemorySyncInterval`）和 `memory_sync_deletions`（`memorySyncDeletions`、`MemorySyncDeletions`）：会话运行期间附加的记忆存储与服务器协调的频率，以及智能体在本地删除的文件是否也从存储中删除。有关单位、默认值以及如何禁用记忆支持，请参阅[配置同步](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#configure-sync)。

* **`work.poller()`：** 代您轮询工作队列，并将每个已认领的会话交给您。当您想决定每个会话发生什么时使用它，例如启动沙箱而不是在进程内运行工具。

  * `drain`：是否在队列为空时停止轮询，而不是等待新工作。
  * `block_ms`：在返回之前等待工作到达的时长，以毫秒为单位。必须在 1 到 999 之间（每次轮询的等待时间；辅助工具会自动重新轮询）。传入 `null`（Python 中为 `None`，Go 中为 `param.Null[int64]()`）进行非阻塞检查；省略该参数则使用默认的 999 毫秒长轮询。
  * `reclaim_older_than_ms`：重新认领已被认领但在此毫秒数内从未被确认的工作项。
  * `auto_stop`（TypeScript 中为 `autoStop`，Go 中为 `AutoStop`）：是否在您的循环体处理完每个工作项后为其发布停止信号。只要运行工作项的任何组件自行发布停止信号，就应将其关闭：`handle_item()` 会这样做，因此当您像本页的 webhook 处理程序那样将已认领的项交给 `handle_item()` 时，请将其设置为 false；您启动的、拥有停止调用的沙箱也是如此。

* **`client.beta.sessions.events.tool_runner()`：** 给定会话 ID 和工具列表，为单个会话运行工具调用。当您已经认领了工作且只需要执行层时使用。

当您想启动自己的每会话进程时，请直接使用工作轮询器，例如为每个已认领的会话启动一个沙箱：

<CodeGroup>
  ```bash cURL
  # work poller 是一个 SDK 辅助工具（Python、TypeScript、Go），而非原始
  # 端点。在 shell 中，请改用 `ant beta:worker poll --on-work`；
  # 请参阅 Always-on (ant CLI) 选项卡。
  ```

  ```bash CLI
  # work poller 是一个 SDK 辅助工具（Python、TypeScript、Go），而不是原始
  # 端点。在 shell 中，请改用 `ant beta:worker poll --on-work`；
  # 请参阅 Always-on (ant CLI) 选项卡。
  ```

  ```python Python
  import asyncio
  import os

  from anthropic import AsyncAnthropic
  from anthropic.types.beta.environments import BetaSelfHostedWork

  SANDBOX_ENV = (
      "ANTHROPIC_ENVIRONMENT_ID",
      "ANTHROPIC_ENVIRONMENT_KEY",
      "ANTHROPIC_WORK_ID",
      "ANTHROPIC_SESSION_ID",
      "ANTHROPIC_WORK_SECRET",
      "ANTHROPIC_BASE_URL",  # forwarded only when set on this host
  )


  async def launch_container(work: BetaSelfHostedWork) -> None:
      print(f"claimed session {work.data.id}")
      # 请将 `docker run` 替换为您自己的沙箱启动器。转发环境
      # 密钥（切勿转发您的 API 密钥）以及工作项的每会话密钥：内部的 worker
      # 需要该密钥来挂载会话的内存存储。
      env = os.environ | {
          "ANTHROPIC_WORK_ID": work.id,
          "ANTHROPIC_SESSION_ID": work.data.id,
          "ANTHROPIC_WORK_SECRET": work.secret or "",
      }
      forward = [arg for name in SANDBOX_ENV for arg in ("-e", name)]
      launcher = await asyncio.create_subprocess_exec(
          "docker", "run", "--rm", "--detach", *forward, "your-sdk-worker-image", env=env
      )
      await launcher.wait()


  async def main() -> None:
      environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
      environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
      async with AsyncAnthropic(auth_token=environment_key) as client:
          async for work in client.beta.environments.work.poller(
              environment_id=environment_id,
              environment_key=environment_key,
              auto_stop=False,  # the launched sandbox owns the stop call
          ):
              await launch_container(work)


  asyncio.run(main())
  ```

  ```typescript TypeScript
  import { spawn } from "node:child_process";
  import { once } from "node:events";
  import Anthropic from "@anthropic-ai/sdk";
  import { WorkPoller } from "@anthropic-ai/sdk/helpers/beta/environments";
  import type { BetaSelfHostedWork } from "@anthropic-ai/sdk/resources/beta/environments";

  const SANDBOX_ENV = [
    "ANTHROPIC_ENVIRONMENT_ID",
    "ANTHROPIC_ENVIRONMENT_KEY",
    "ANTHROPIC_WORK_ID",
    "ANTHROPIC_SESSION_ID",
    "ANTHROPIC_WORK_SECRET",
    "ANTHROPIC_BASE_URL" // forwarded only when set on this host
  ];

  const environmentKey = process.env.ANTHROPIC_ENVIRONMENT_KEY!;
  const environmentId = process.env.ANTHROPIC_ENVIRONMENT_ID!;
  const client = new Anthropic({ authToken: environmentKey });

  async function launchContainer(work: BetaSelfHostedWork): Promise<void> {
    console.log(`claimed session ${work.data.id}`);
    // 请将 `docker run` 替换为您自己的沙箱启动器。转发环境
    // 密钥（切勿转发您的 API 密钥）以及工作项的每会话密钥：
    // 内部的 worker 需要该密钥来挂载会话的内存存储。
    const env = {
      ...process.env,
      ANTHROPIC_WORK_ID: work.id,
      ANTHROPIC_SESSION_ID: work.data.id,
      ANTHROPIC_WORK_SECRET: work.secret ?? ""
    };
    const forward = SANDBOX_ENV.flatMap((name) => ["-e", name]);
    const launcher = spawn(
      "docker",
      ["run", "--rm", "--detach", ...forward, "your-sdk-worker-image"],
      { env, stdio: "inherit" }
    );
    await once(launcher, "close");
  }

  const poller = new WorkPoller({
    client,
    environmentId,
    environmentKey,
    autoStop: false // the launched sandbox owns the stop call
  });

  for await (const work of poller) {
    await launchContainer(work);
  }
  ```

  ```csharp C#
  // C# SDK 目前尚未提供工作轮询辅助工具。
  // 如需直接领取工作，请参阅 Environments Work 端点。
  ```

  ```go Go
  package main

  import (
  	"context"
  	"fmt"
  	"log"
  	"os"
  	"os/exec"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/lib/environments"
  	"github.com/anthropics/anthropic-sdk-go/option"
  	"github.com/anthropics/anthropic-sdk-go/packages/param"
  )

  var sandboxEnv = []string{
  	"ANTHROPIC_ENVIRONMENT_ID",
  	"ANTHROPIC_ENVIRONMENT_KEY",
  	"ANTHROPIC_WORK_ID",
  	"ANTHROPIC_SESSION_ID",
  	"ANTHROPIC_WORK_SECRET",
  	"ANTHROPIC_BASE_URL", // forwarded only when set on this host
  }

  func launchContainer(ctx context.Context, work *anthropic.BetaSelfHostedWork) error {
  	fmt.Printf("claimed session %s\n", work.Data.ID)
  	// 请将 `docker run` 替换为您自己的沙箱启动器。转发环境
  	// 密钥（绝不是您的 API 密钥）以及工作项的每会话密钥：内部的
  	// worker 需要该密钥来挂载会话的内存存储。
  	args := []string{"run", "--rm", "--detach"}
  	for _, name := range sandboxEnv {
  		args = append(args, "-e", name)
  	}
  	launcher := exec.CommandContext(ctx, "docker", append(args, "your-sdk-worker-image")...)
  	launcher.Env = append(os.Environ(),
  		"ANTHROPIC_WORK_ID="+work.ID,
  		"ANTHROPIC_SESSION_ID="+work.Data.ID,
  		"ANTHROPIC_WORK_SECRET="+work.Secret,
  	)
  	launcher.Stdout, launcher.Stderr = os.Stdout, os.Stderr
  	return launcher.Run()
  }

  func main() {
  	environmentID := os.Getenv("ANTHROPIC_ENVIRONMENT_ID")
  	environmentKey := os.Getenv("ANTHROPIC_ENVIRONMENT_KEY")

  	client := anthropic.NewClient(option.WithAuthToken(environmentKey))

  	ctx := context.Background()

  	poller := environments.NewWorkPoller(ctx, client, environments.WorkPollerOptions{
  		EnvironmentID:  environmentID,
  		EnvironmentKey: environmentKey,
  		AutoStop:       param.NewOpt(false), // the launched sandbox owns the stop call
  	})
  	defer poller.Close()

  	for work, err := range poller.All() {
  		if err != nil {
  			log.Fatal(err)
  		}
  		if err := launchContainer(ctx, work); err != nil {
  			log.Fatal(err)
  		}
  	}
  }
  ```

  ```java Java
  // Java SDK 目前尚未提供工作轮询辅助工具。
  // 如需直接领取工作，请参阅 Environments Work 端点。
  ```

  ```php PHP
  // PHP SDK 目前尚未提供工作轮询辅助函数。
  // 如需直接领取工作，请参阅 Environments Work 端点。
  ```

  ```ruby Ruby
  # Ruby SDK 目前尚未提供工作轮询辅助方法。
  # 如需直接领取工作，请参阅 Environments Work 端点。
  ```
</CodeGroup>

无论由什么启动沙箱，都必须将已认领工作项的 `secret` 连同会话、工作和环境标识符一起转发到沙箱中（例如作为 `ANTHROPIC_WORK_SECRET`），以便内部的 worker 可以挂载会话的[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)；请参阅[每个会话运行一个沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#run-one-sandbox-per-session)。

**`AgentToolContext`** 是工具调用的执行上下文。它定义工作目录和路径策略，并可以下载会话的技能。文件工具（`read`、`write`、`edit`、`glob`、`grep`）被限制在工作目录以及 `allowed_roots`（TypeScript 中为 `allowedRoots`，Go 中为 `AllowedRoots`）中列出的任何目录内，并且 `write` 和 `edit` 还会拒绝 `read_only_roots`（`readOnlyRoots`、`ReadOnlyRoots`）下的路径。`EnvironmentWorker` 会自行将会话的记忆存储目录添加到这些列表中。该限制仅是文件工具的护栏，而不是沙箱；它不约束 `bash`。**`beta_agent_toolset_20260401(env)`** 接受一个 `AgentToolContext` 并返回标准工具实现（`bash`、`read`、`write`、`edit`、`glob`、`grep`）。

**使用 `EnvironmentWorker`：** 两者都自动管理。传入一个 `tools` 工厂以自定义工具列表：

<CodeGroup exclude="shell">
  ```python Python
  EnvironmentWorker(client, ..., tools=lambda env: [beta_bash_tool(env), my_custom_tool])
  ```

  ```typescript TypeScript
  new EnvironmentWorker({
    client,
    environmentId,
    environmentKey,
    tools: (ctx) => [betaBashTool(ctx), myCustomTool]
  });
  ```

  ```csharp C#
  // EnvironmentWorker 目前在 C# SDK 中不可用。
  // 如需直接响应自定义工具调用，请参阅会话事件流。
  ```

  ```go Go
  worker := environments.NewEnvironmentWorker(client, environments.EnvironmentWorkerOptions{
  	EnvironmentID:  environmentID,
  	EnvironmentKey: environmentKey,
  	ToolsFunc: func(env *agenttoolset.AgentToolContext) []anthropic.BetaTool {
  		return []anthropic.BetaTool{agenttoolset.BetaBashTool(env), myCustomTool}
  	},
  })
  ```

  ```java Java
  // Java SDK 目前不提供 EnvironmentWorker。
  // 如需直接响应自定义工具调用，请参阅会话事件流。
  ```

  ```php PHP
  // PHP SDK 目前尚不支持 EnvironmentWorker。
  // 如需直接响应自定义工具调用，请参阅会话事件流。
  ```

  ```ruby Ruby
  # Ruby SDK 目前暂不提供 EnvironmentWorker。
  # 如需直接响应自定义工具调用，请参阅会话事件流。
  ```
</CodeGroup>

**使用 `work.poller()` 和 `tool_runner()`：** 将工具列表作为 `tools` 传递给 `client.beta.sessions.events.tool_runner()`。若要构建该列表，请自行设置 `AgentToolContext` 并调用 `beta_agent_toolset_20260401(env)`：

<CodeGroup exclude="shell">
  ```python Python
  from anthropic.lib.tools.agent_toolset import (
      AgentToolContext,
      beta_agent_toolset_20260401,
  )

  async with AgentToolContext(
      workdir="/workspace", client=client, session_id=work.data.id
  ) as env:
      # skills 已下载到 /workspace/skills/<name>/
      tools = beta_agent_toolset_20260401(env)
  ```

  ```typescript TypeScript
  import {
    setupSkills,
    betaAgentToolset20260401
  } from "@anthropic-ai/sdk/tools/agent-toolset/node";

  const ctx = { workdir: "/workspace", client, sessionId: work.data.id };
  await setupSkills(ctx);
  const tools = betaAgentToolset20260401(ctx);
  ```

  ```csharp C#
  // AgentToolContext 目前在 C# SDK 中不可用。
  ```

  ```go Go
  env := &agenttoolset.AgentToolContext{Workdir: "/workspace"}
  if err := env.SetupSkills(ctx, client, work.Data.ID); err != nil {
  	panic(err)
  }
  // skills 已下载到 /workspace/skills/<name>/
  tools := agenttoolset.BetaAgentToolset20260401(env)
  ```

  ```java Java
  // AgentToolContext 目前在 Java SDK 中不可用。
  ```

  ```php PHP
  // PHP SDK 目前不支持 AgentToolContext。
  ```

  ```ruby Ruby
  # Ruby SDK 目前不支持 AgentToolContext。
  ```
</CodeGroup>

### 验证 worker 已连接

在另一个 shell 中，将 `ANTHROPIC_API_KEY` 设置为您的 Claude API 密钥（而非环境密钥），确认 `workers_polling` 至少为 1：

```bash
ant beta:environments:work stats --environment-id "$ANTHROPIC_ENVIRONMENT_ID"
```

如果 `workers_polling` 保持为 0，则 worker 未能访问队列：请确认 worker 主机上已设置 `ANTHROPIC_ENVIRONMENT_KEY` 和 `ANTHROPIC_ENVIRONMENT_ID`。有关完整的统计响应和其他语言示例，请参阅[读取队列深度](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#read-queue-depth)。

## 启动会话

worker 运行后，创建一个以该环境为目标的会话。将 `AGENT_ID` 设置为您在[开始之前](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#before-you-begin)中记下的智能体 ID。会话进入环境的工作队列并在那里等待，直到有 worker 认领它；如果没有 worker 连接，会话将保持排队状态而不是失败。

Anthropic 不会将文件或 GitHub 仓库挂载到自托管沙箱中。若要使会话特定的文件可用，请在会话的 `metadata` 字段中传递文件引用（例如 S3 路径或提交 SHA）。已认领的工作项不携带会话的元数据，但携带会话 ID：您的生成脚本或 `--on-work` 处理程序检索会话（`GET /v1/sessions/{session_id}`）以读取 `metadata` 字段，然后在工具执行开始之前将文件暂存到工作目录中。

<CodeGroup>
  ```bash cURL
  curl -sS --fail-with-body https://api.anthropic.com/v1/sessions \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d @- <<EOF
  {
    "agent": "$AGENT_ID",
    "environment_id": "$ANTHROPIC_ENVIRONMENT_ID",
    "metadata": {"input_file": "s3://my-bucket/data.csv"}
  }
  EOF
  ```

  ```bash CLI
  ant beta:sessions create \
    --agent "$AGENT_ID" \
    --environment-id "$ANTHROPIC_ENVIRONMENT_ID" \
    --metadata '{"input_file": "s3://my-bucket/data.csv"}'
  ```

  ```python Python
  session = client.beta.sessions.create(
      agent=agent.id,
      environment_id=environment.id,
      metadata={"input_file": "s3://my-bucket/data.csv"},
  )
  ```

  ```typescript TypeScript
  const session = await client.beta.sessions.create({
    agent: agent.id,
    environment_id: environment.id,
    metadata: { input_file: "s3://my-bucket/data.csv" }
  });
  ```

  ```csharp C#
  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = agent.ID,
      EnvironmentID = environment.ID,
      Metadata = new Dictionary<string, string> { ["input_file"] = "s3://my-bucket/data.csv" },
  });
  ```

  ```go Go
  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent:         anthropic.BetaSessionNewParamsAgentUnion{OfString: anthropic.String(agent.ID)},
  	EnvironmentID: environment.ID,
  	Metadata: map[string]string{
  		"input_file": "s3://my-bucket/data.csv",
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var session = client.beta().sessions().create(SessionCreateParams.builder()
      .agent(agent.id())
      .environmentId(environment.id())
      .metadata(SessionCreateParams.Metadata.builder()
          .putAdditionalProperty("input_file", JsonValue.from("s3://my-bucket/data.csv"))
          .build())
      .build());
  ```

  ```php PHP
  $session = $client->beta->sessions->create(
      agent: $agent->id,
      environmentID: $environment->id,
      metadata: ['input_file' => 's3://my-bucket/data.csv'],
  );
  ```

  ```ruby Ruby
  session = client.beta.sessions.create(
    agent: agent.id,
    environment_id: environment.id,
    metadata: {input_file: "s3://my-bucket/data.csv"}
  )
  ```
</CodeGroup>

<Note>
  自托管沙箱仅支持 `memory_store` 资源；请参阅[使用记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)。自托管环境上包含 `file` 或 `github_repository` 资源的会话会被拒绝并返回 400 错误：

  ```text wrap
  Environment env_... is a self-hosted environment. `resources` are not supported with self-hosted environments.
  ```

  以自托管环境为目标的[部署](https://platform.claude.com/docs/zh-CN/managed-agents/scheduled-deployments)遵循相同的规则。
</Note>

有关 CLI 标志的完整列表，请参阅参考文档中的[自托管 worker](https://platform.claude.com/docs/zh-CN/managed-agents/reference#self-hosted-worker)；有关 SDK 辅助工具选项，请参阅 [SDK 辅助工具](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#sdk-helpers)。

## 使用记忆存储

自托管环境上的会话附加[记忆存储](https://platform.claude.com/docs/zh-CN/managed-agents/memory)的方式与云环境上的会话完全相同：在创建会话时将它们列在 `resources` 中，如[将记忆存储附加到会话](https://platform.claude.com/docs/zh-CN/managed-agents/memory#attach-a-memory-store-to-a-session)中所示。一个会话最多接受 8 个记忆存储。在自托管环境中，由 SDK worker 而非 Anthropic 的基础设施为智能体实体化每个存储，因此那里的记忆存储需要来自 Python、TypeScript 或 Go SDK 的 `EnvironmentWorker`（或其 `handle_item()` 方法）。

`ant` CLI worker（`ant beta:worker poll` 和 `ant beta:worker run`）不会挂载记忆存储。若要将 CLI 轮询器与记忆存储结合使用，请按照[每个会话运行一个沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#run-one-sandbox-per-session)中的描述，在每会话沙箱内运行 SDK worker。

在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上，记忆存储无法附加到自托管环境上的会话。

### worker 如何处理记忆

当 worker 认领一个其会话附加了记忆存储的工作项时，它会：

1. 将每个附加的存储下载到 worker 主机上的 `mount_path`，使用工作项的每会话 `secret` 进行身份验证。`mount_path` 与云会话使用的 `/mnt/memory/` 下的目录相同（例如，名为 "User Preferences" 的存储对应 `/mnt/memory/user-preferences/`），并且会话的系统提示会向智能体描述它。
2. 将这些目录添加到文件工具的允许根目录中，并将以 `access: "read_only"` 附加的存储的目录添加到其只读根目录中，以便智能体使用与在工作目录中相同的 `read`、`write`、`edit`、`glob` 和 `grep` 工具处理记忆。
3. 在工具调用后协调本地和远程更改，每个同步间隔（默认 15 秒）最多一次：存储中已更改的记忆会写入磁盘，智能体更改的文件会上传到存储。
4. 在会话结束时运行最终同步，在最多 30 秒内刷新任何仍待处理的上传，然后删除它创建的目录。在会话运行期间被取消的 worker 会跳过最终同步，但仍会在退出前上传已更改的文件并删除目录。

Anthropic 一侧的记忆存储仍然是事实来源。[记忆版本](https://platform.claude.com/docs/zh-CN/managed-agents/memory#audit-memory-changes)、脱敏以及在 Console 中查看或编辑记忆的工作方式与云会话相同，并且智能体的记忆读写会作为普通工具事件出现在[事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)中。由于每个 worker 按间隔同步，在一个会话中写入的更改只有在两个会话都同步之后才会对另一个正在运行的会话可见，在默认间隔下通常远低于一分钟；云沙箱上的会话几乎可以立即看到彼此的更改。

每个存储目录包含一个名为 `.anthropic-memory-store` 的标记文件，它将该目录与其存储关联起来。请保留它：worker 不会同步标记文件缺失或被更改的目录。

### 准备主机

自托管沙箱上的记忆存储需要 worker 主机（[开始之前](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#before-you-begin)中的 Linux 主机）上有 POSIX 文件系统；不支持 Windows 主机，因为 worker 在打开记忆文件时需要 `O_NOFOLLOW`。建议使用区分大小写的文件系统，以便仅大小写不同的记忆路径不会冲突。

在启动 worker 之前，创建父目录并使其对 worker 运行所用的用户可写：

```bash
sudo mkdir -p /mnt/memory && sudo chown "$USER" /mnt/memory
```

不要自行创建每个存储的目录。worker 在会话启动时创建每个存储的 `mount_path` 目录（例如 `/mnt/memory/user-preferences`），如果该路径上已存在某些内容则拒绝启动该会话的工作，并在会话结束时删除该目录。由此产生两条操作规则：

* **当会话附加同一存储时，每个文件系统运行一个会话。** 两个会话不能同时在一台主机上挂载同一存储，因为两者需要相同的路径。按照[每个会话运行一个沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#run-one-sandbox-per-session)中的描述为每个会话提供自己的沙箱即可满足此规则。
* **优雅地停止 worker。** 当您在会话运行期间停止 worker 时，`EnvironmentWorker` 仅在被取消而非被强制终止的情况下才会上传会话已更改的记忆文件并删除其存储目录：被强制终止的进程不会运行任何清理，并且 worker 本身不会安装信号处理程序。请在运行它的进程中将 SIGTERM 和 SIGINT 连接到取消操作：在 TypeScript 中中止您传递给 worker 的 `signal`，在 Go 中取消 context，在 Python 中取消运行 `run()` 或 `handle_item()` 的任务。当您的 worker 就是该进程时（如本页的独立 worker 那样），从信号处理程序中执行此操作；当 worker 在 webhook 处理程序内运行时，从您服务器自己的关闭钩子中执行此操作，此时它不得接管服务器的信号。然后使用 SIGTERM 停止 worker，并在任何强制终止之前给它们至少 30 秒的退出时间，因为最终上传可能需要那么长时间。如果 worker 在其清理运行之前被强制终止，请在下一个附加该存储的会话之前删除 `/mnt/memory/` 下遗留的存储目录；其中任何尚未同步的编辑都会丢失。

### 每个会话运行一个沙箱

[运行 worker](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#run-a-worker) 中的"每个会话一个沙箱"模式为每个会话提供一个全新的文件系统，这正是当多个会话挂载同一个存储时 [准备主机](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#prepare-the-host) 所要求的。请继续在主机上使用 `ant beta:worker poll --on-work`（或 SDK 的工作轮询器）作为轮询器。

那里展示的 `ant beta:worker run` 入口点不会挂载记忆存储，因此请改为围绕 SDK worker 构建每会话镜像：其入口点构造 `EnvironmentWorker` 并调用 `handle_item()`（TypeScript 中为 `handleItem`，Go 中为 `HandleItem`），该方法从 `ANTHROPIC_*` 变量中读取会话、工作和环境标识符，并从 `ANTHROPIC_WORK_SECRET` 中读取工作项的每会话 `secret`。您也可以将该 secret 显式地作为 `work_secret`（TypeScript 中为 `workSecret`，Go 中为 `WorkSecret`）传入。

<CodeGroup exclude="shell">
  ```python Python
  import asyncio
  import contextlib
  import os
  import signal
  from anthropic import AsyncAnthropic
  from anthropic.lib.environments import EnvironmentWorker


  async def main() -> None:
      async with AsyncAnthropic(auth_token=os.environ["ANTHROPIC_ENVIRONMENT_KEY"]) as client:
          worker = EnvironmentWorker(client, workdir="/workspace")
          # 不带参数时，handle_item() 会读取 spawn 脚本转发的 ANTHROPIC_* 变量，
          # 包括 ANTHROPIC_WORK_SECRET。
          task = asyncio.create_task(worker.handle_item())
          # 在容器停止时取消任务，可让工作进程在退出前上传
          # 已更改的内存文件并移除存储目录。
          loop = asyncio.get_running_loop()
          for signum in (signal.SIGINT, signal.SIGTERM):
              loop.add_signal_handler(signum, task.cancel)
          with contextlib.suppress(asyncio.CancelledError):
              await task


  asyncio.run(main())
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";
  import { EnvironmentWorker } from "@anthropic-ai/sdk/helpers/beta/environments";

  const client = new Anthropic({ authToken: process.env.ANTHROPIC_ENVIRONMENT_KEY });
  const controller = new AbortController();
  // 在容器停止时中止，可让 worker 在退出前上传已更改的内存
  // 文件并移除存储目录。
  process.once("SIGTERM", () => controller.abort());
  process.once("SIGINT", () => controller.abort());

  // 不带参数时，handleItem() 会读取 spawn 脚本转发的 ANTHROPIC_* 变量，
  // 包括 ANTHROPIC_WORK_SECRET。
  await new EnvironmentWorker({
    client,
    workdir: "/workspace",
    signal: controller.signal
  }).handleItem();
  ```

  ```csharp C#
  // EnvironmentWorker 目前在 C# SDK 中不可用。
  ```

  ```go Go
  package main

  import (
  	"context"
  	"log"
  	"os"
  	"os/signal"
  	"syscall"

  	"github.com/anthropics/anthropic-sdk-go"
  	"github.com/anthropics/anthropic-sdk-go/lib/environments"
  	"github.com/anthropics/anthropic-sdk-go/option"
  )

  func main() {
  	// 在容器停止时取消 context，可让 worker 在退出前上传
  	// 已更改的内存文件并移除存储目录。
  	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
  	defer stop()

  	client := anthropic.NewClient(option.WithAuthToken(os.Getenv("ANTHROPIC_ENVIRONMENT_KEY")))
  	worker := environments.NewEnvironmentWorker(client, environments.EnvironmentWorkerOptions{
  		Workdir: "/workspace",
  	})
  	// 使用零值选项时，HandleItem 会读取 spawn 脚本转发的 ANTHROPIC_* 变量，
  	// 包括 ANTHROPIC_WORK_SECRET。
  	if err := worker.HandleItem(ctx, environments.HandleItemOptions{}); err != nil {
  		log.Fatalf("worker: %v", err)
  	}
  }

  ```

  ```java Java
  // Java SDK 目前暂不提供 EnvironmentWorker。
  ```

  ```php PHP
  // PHP SDK 目前暂不支持 EnvironmentWorker。
  ```

  ```ruby Ruby
  # Ruby SDK 目前暂不支持 EnvironmentWorker。
  ```
</CodeGroup>

`ant beta:worker poll --on-work` 不会为其派生的脚本设置 `ANTHROPIC_WORK_SECRET`，因此派生脚本从其标准输入上的工作项 JSON 中读取该 secret，并将其传入沙箱：

```bash
#!/bin/bash
# spawn.sh：每个已领取的工作项调用一次
# 已领取的工作项以 JSON 形式通过 stdin 传入。其 secret 是
# 内存存储端点所需的每会话凭据。
ANTHROPIC_WORK_SECRET="$(jq -r '.secret // empty')"
export ANTHROPIC_WORK_SECRET
mkdir -p "/host/outputs/$ANTHROPIC_SESSION_ID"
exec docker run --rm \
  -e ANTHROPIC_SESSION_ID -e ANTHROPIC_ENVIRONMENT_KEY \
  -e ANTHROPIC_WORK_ID -e ANTHROPIC_ENVIRONMENT_ID -e ANTHROPIC_BASE_URL \
  -e ANTHROPIC_WORK_SECRET \
  -v "/host/outputs/$ANTHROPIC_SESSION_ID":/workspace \
  your-sdk-worker-image
```

如果您改用 SDK 的工作轮询器来领取工作，请以同样的方式将每个已领取项的 `secret` 传入您启动的沙箱。只将其传入为该会话提供服务的沙箱，并且绝不要将其记录到日志中。

沙箱镜像还需要一个可写的 `/mnt/memory`（请参阅 [准备主机](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#prepare-the-host)）。由于每个沙箱只服务一个会话并在之后被丢弃，因此没有遗留目录需要清理，记忆目录也不需要绑定挂载到主机：worker 会在沙箱退出之前将其内容上传到存储。如果您在会话结束之前停止容器，请发送一个入口点会将其转换为取消操作的信号（请参阅 [准备主机](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#prepare-the-host)），而不是直接杀死它，以便上传仍能运行。同时也要给容器留出完成上传的时间：Docker 默认在停止信号发出 10 秒后跟进 SIGKILL，因此请通过 `docker run` 上的 `--stop-timeout` 或您的编排器的终止宽限期，将该限制提高到至少"准备主机"所要求的 30 秒。

### 配置同步

两个 `EnvironmentWorker` 选项控制记忆行为：

* **`memory_sync_interval`**（Python 中以秒为单位；TypeScript 中为 `memorySyncIntervalMs`，以毫秒为单位；Go 中为 `MemorySyncInterval`，是一个 duration）：会话运行期间已挂载的存储与服务器进行协调的频率。默认为 15 秒；最小值为 5 秒。更短的间隔会缩小另一个会话看到过时记忆的时间窗口，代价是更多的记忆存储请求。Python 中的 `None`、TypeScript 中的 `null` 或 Go 中的负 duration 会完全禁用记忆支持：worker 既不下载也不同步存储，而挂载了记忆存储的会话会在没有它们的情况下运行，即使其系统提示仍然描述了它们，因此请仅在其会话不挂载任何记忆存储的 worker 上禁用记忆支持。在启用记忆支持时，对于挂载了存储的会话，如果到达的工作项没有每会话 `secret`，则该工作项会失败，而不是在没有记忆的情况下运行（请参阅 [排查记忆挂载问题](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#troubleshoot-memory-mounts)）。
* **`memory_sync_deletions`**（TypeScript 中为 `memorySyncDeletions`，Go 中为 `MemorySyncDeletions`）：智能体在本地删除的文件是否也从存储中删除。在 Python 和 TypeScript 中，该值为 `"enabled"`（默认值）、`"log_only"` 或 `"disabled"` 之一；在 Go 中为常量 `environments.MemorySyncDeletionsEnabled`（零值）、`environments.MemorySyncDeletionsLogOnly` 或 `environments.MemorySyncDeletionsDisabled` 之一。启用时，一旦后续同步确认文件仍然不存在，worker 就会从存储中删除该记忆；在仅记录模式下，它运行相同的检查，但只记录它本会删除的内容，这让您可以在信任启用模式之前观察您的 worker 会删除什么；禁用时，它从不从存储中删除。上传和下载不受此设置影响。

请在您构造 worker 的位置设置这些选项，无论是通过 `EnvironmentWorker` 构造函数，还是在 Python 和 TypeScript 中通过 webhook 处理程序所使用的 `client.beta.environments.work.worker()` 工厂。

例如，要每 10 秒同步一次，并且只记录 worker 本会执行的删除操作：

<CodeGroup exclude="shell">
  ```python Python
  worker = EnvironmentWorker(
      client,
      environment_id=environment_id,
      environment_key=environment_key,
      workdir="/workspace",
      memory_sync_interval=10,  # seconds
      memory_sync_deletions="log_only",
  )
  ```

  ```typescript TypeScript
  const worker = new EnvironmentWorker({
    client,
    environmentId,
    environmentKey,
    workdir: "/workspace",
    memorySyncIntervalMs: 10_000,
    memorySyncDeletions: "log_only"
  });
  ```

  ```csharp C#
  // EnvironmentWorker 目前在 C# SDK 中不可用。
  ```

  ```go Go
  worker := environments.NewEnvironmentWorker(client, environments.EnvironmentWorkerOptions{
  	EnvironmentID:       environmentID,
  	EnvironmentKey:      environmentKey,
  	Workdir:             "/workspace",
  	MemorySyncInterval:  10 * time.Second,
  	MemorySyncDeletions: environments.MemorySyncDeletionsLogOnly,
  })
  ```

  ```java Java
  // Java SDK 目前暂不支持 EnvironmentWorker。
  ```

  ```php PHP
  // PHP SDK 目前暂不支持 EnvironmentWorker。
  ```

  ```ruby Ruby
  # Ruby SDK 目前不支持 EnvironmentWorker。
  ```
</CodeGroup>

### 只读存储与冲突

对于以 `access: "read_only"` 挂载的存储，`write` 和 `edit` 工具会拒绝更改其目录内的文件，并且 worker 从不上传其中的任何内容。通过 `bash`，或通过您从沙箱提供的自定义工具或 MCP 服务器所做的更改不会在本地被阻止：它们永远不会同步到存储，并且对该记忆的下一次远程更改会覆盖它们。如果您需要本地副本本身在会话期间保持不变，请为该智能体禁用 `bash` 工具，并且不要为其提供任何会写入沙箱文件系统的自定义工具；不要以只读方式挂载存储路径，因为 worker 本身必须创建该目录并将下载的记忆写入其中。

冲突以存储为准进行解决。当智能体更改了一个记忆文件，而该文件自会话上次同步以来在存储中也发生了更改时，worker 会在下一次同步时保留存储的版本，用它覆盖本地文件，并记录一条警告；`write` 和 `edit` 工具本身会成功，不会有错误传达给智能体。如果智能体的更改仍然适用，它可以在同步后重新读取该文件并再次进行更改。

### 排查记忆挂载问题

worker 会记录挂载和后台同步失败，而不是将其报告给会话；只有只读拒绝会以工具错误的形式传达给智能体（请参阅 [只读存储与冲突](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#read-only-stores-and-conflicts)）。如果在 worker 领取会话时无法挂载记忆存储，worker 会使该工作项失败：会话不会发出错误事件，并保持空闲状态。

| 症状                                                                                                            | 原因                                                                        | 修复方法                                                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| worker 日志包含 `the work item carried no sessions token`（在 Go 中为 `ErrSessionMemoryNoToken` 错误），并且工作项失败。          | 工作项的每会话 `secret` 未到达 worker：您的组织未启用自托管沙箱上的记忆存储，或者您的派生脚本未将该 secret 转发到沙箱中。 | 在"每个会话一个沙箱"模式中，按照 [每个会话运行一个沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#run-one-sandbox-per-session) 中所示将 `ANTHROPIC_WORK_SECRET` 转发到沙箱中。如果 worker 在同一进程中轮询并运行会话但仍记录此信息，请联系支持团队。 |
| worker 日志包含 `something already exists at the memory store's path`。                                            | 上一个会话遗留的目录，通常是其 worker 在拆除运行之前被杀死的会话。                                     | 删除日志行中指明的遗留目录。其中尚未同步的编辑将丢失。                                                                                                                                                                                         |
| worker 日志包含 `cannot create the memory store's folder` 和 `the worker host must make this mount path writable`。 | worker 运行所用的用户无法在 `/mnt/memory` 下创建目录。                                    | 创建 `/mnt/memory` 并将其 `chown` 给该用户；请参阅 [准备主机](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#prepare-the-host)。                                                                         |
| 在 worker 领取会话后不久，会话处于 `idle` 状态，停止原因为 `requires_action`，且没有错误事件。                                              | 由于前述原因之一，worker 无法挂载记忆存储，因此使工作项失败。                                        | 在主机上修复原因，然后发送一个 [`user.interrupt`](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#integrating-events) 事件：会话的工作会再次排队，下一个领取它的 worker 会重试挂载。                                               |

## 从您的沙箱提供自定义工具

[自定义工具](https://platform.claude.com/docs/zh-CN/managed-agents/tools#custom-tools)是由您自己的代码执行的工具：智能体发出一个 `agent.custom_tool_use` 事件，并等待匹配的 `user.custom_tool_result`。worker 可以充当该代码，并且由于它在您的沙箱内运行，该工具可以访问您为沙箱配置的内部服务、凭据和网络出口，仅此而已。环境密钥授权发布自定义工具结果，因此您的 Claude API 密钥不会出现在 worker 主机上。

<Note>
  提供自定义工具需要 SDK worker：`ant` CLI worker 无法注册自定义工具实现。在"每个会话一个沙箱"模式中，请在沙箱内运行 `EnvironmentWorker` 并使用 `handle_item()`（TypeScript 中为 `handleItem`，Go 中为 `HandleItem`）来代替 `ant beta:worker run`。
</Note>

<Steps>
  <Step title="在智能体上声明工具">
    在智能体的 `tools` 中添加一个 `custom` 条目，其 `name` 与您的 worker 注册的工具相匹配。有关完整的声明结构，请参阅 [自定义工具](https://platform.claude.com/docs/zh-CN/managed-agents/tools#custom-tools)。

    ```json
    {
      "type": "custom",
      "name": "get_order_status",
      "description": "Look up an order in the internal fulfillment system by order ID.",
      "input_schema": {
        "type": "object",
        "properties": {
          "order_id": { "type": "string", "description": "The order ID" }
        },
        "required": ["order_id"]
      }
    }
    ```
  </Step>

  <Step title="向 worker 注册实现">
    通过 worker 的 `tools` 工厂（请参阅 [SDK 辅助工具](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#sdk-helpers)）传入该工具，与内置工具集一起：

    <CodeGroup exclude="shell">
      ```python Python
      import asyncio
      import os
      from anthropic import AsyncAnthropic, beta_async_tool
      from anthropic.lib.environments import EnvironmentWorker
      from anthropic.lib.tools.agent_toolset import beta_agent_toolset_20260401


      @beta_async_tool
      async def get_order_status(order_id: str) -> str:
          """Look up an order in the internal fulfillment system by order ID."""
          # 在 worker 主机上运行：可调用沙箱能够访问的任何内容。
          return f"Order {order_id}: shipped"


      async def main() -> None:
          environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
          environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
          async with AsyncAnthropic(auth_token=environment_key) as client:
              await EnvironmentWorker(
                  client,
                  environment_id=environment_id,
                  environment_key=environment_key,
                  workdir="/workspace",
                  tools=lambda env: [*beta_agent_toolset_20260401(env), get_order_status],
              ).run()


      asyncio.run(main())
      ```

      ```typescript TypeScript
      import Anthropic from "@anthropic-ai/sdk";
      import { EnvironmentWorker } from "@anthropic-ai/sdk/helpers/beta/environments";
      import { betaTool } from "@anthropic-ai/sdk/helpers/beta/json-schema";
      import { betaAgentToolset20260401 } from "@anthropic-ai/sdk/tools/agent-toolset/node";

      const getOrderStatus = betaTool({
        name: "get_order_status",
        description: "Look up an order in the internal fulfillment system by order ID.",
        inputSchema: {
          type: "object",
          properties: { order_id: { type: "string", description: "The order ID" } },
          required: ["order_id"]
        },
        // 在 worker 主机上运行：可调用沙箱能访问的任何内容。
        run: async ({ order_id }) => `Order ${order_id}: shipped`
      });

      const environmentKey = process.env.ANTHROPIC_ENVIRONMENT_KEY!;
      const environmentId = process.env.ANTHROPIC_ENVIRONMENT_ID!;
      const client = new Anthropic({ authToken: environmentKey });
      const controller = new AbortController();
      process.once("SIGTERM", () => controller.abort());

      await new EnvironmentWorker({
        client,
        environmentId,
        environmentKey,
        workdir: "/workspace",
        signal: controller.signal,
        tools: (ctx) => [...betaAgentToolset20260401(ctx), getOrderStatus]
      }).run();
      ```

      ```csharp C#
      // EnvironmentWorker 目前在 C# SDK 中不可用。
      // 如需直接响应自定义工具调用，请参阅会话事件流。
      ```

      ```go Go
      package main

      import (
      	"context"
      	"log"
      	"os"
      	"os/signal"
      	"syscall"

      	"github.com/anthropics/anthropic-sdk-go"
      	"github.com/anthropics/anthropic-sdk-go/lib/environments"
      	"github.com/anthropics/anthropic-sdk-go/option"
      	"github.com/anthropics/anthropic-sdk-go/toolrunner"
      	"github.com/anthropics/anthropic-sdk-go/tools/agenttoolset"
      )

      type orderStatusInput struct {
      	OrderID string `json:"order_id"`
      }

      func main() {
      	environmentKey := os.Getenv("ANTHROPIC_ENVIRONMENT_KEY")
      	environmentID := os.Getenv("ANTHROPIC_ENVIRONMENT_ID")

      	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
      	defer stop()

      	getOrderStatus := toolrunner.NewBetaTool(
      		"get_order_status",
      		"Look up an order in the internal fulfillment system by order ID.",
      		anthropic.BetaToolInputSchemaParam{
      			Properties: map[string]any{
      				"order_id": map[string]any{"type": "string", "description": "The order ID"},
      			},
      			Required: []string{"order_id"},
      		},
      		// 在工作进程主机上运行：可调用沙箱能够访问的任何内容。
      		func(ctx context.Context, input orderStatusInput) (anthropic.BetaToolResultBlockParamContentUnion, error) {
      			return anthropic.BetaToolResultBlockParamContentUnion{
      				OfText: &anthropic.BetaTextBlockParam{Text: "Order " + input.OrderID + ": shipped"},
      			}, nil
      		},
      	)

      	client := anthropic.NewClient(option.WithAuthToken(environmentKey))

      	worker := environments.NewEnvironmentWorker(client, environments.EnvironmentWorkerOptions{
      		EnvironmentID:  environmentID,
      		EnvironmentKey: environmentKey,
      		Workdir:        "/workspace",
      		ToolsFunc: func(env *agenttoolset.AgentToolContext) []anthropic.BetaTool {
      			return append(agenttoolset.BetaAgentToolset20260401(env), getOrderStatus)
      		},
      	})
      	if err := worker.Run(ctx); err != nil {
      		log.Fatalf("worker: %v", err)
      	}
      }

      ```

      ```java Java
      // EnvironmentWorker 目前在 Java SDK 中不可用。
      // 如需直接响应自定义工具调用，请参阅会话事件流。
      ```

      ```php PHP
      // PHP SDK 目前暂不提供 EnvironmentWorker。
      // 如需直接响应自定义工具调用，请参阅会话事件流。
      ```

      ```ruby Ruby
      # Ruby SDK 目前暂不提供 EnvironmentWorker。
      # 如需直接响应自定义工具调用，请参阅会话事件流。
      ```
    </CodeGroup>
  </Step>
</Steps>

worker 只响应向其注册的工具。在智能体上声明但未向任何 worker 或客户端注册的自定义工具会使会话以 `requires_action` 停止原因暂停，直到有东西发布其结果；有关事件流程，请参阅 [处理自定义工具调用](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming#handling-custom-tool-calls)。

### 将 MCP 服务器包装为自定义工具

[MCP 连接器](https://platform.claude.com/docs/zh-CN/managed-agents/mcp-connector)从 Anthropic 一侧连接到 MCP 服务器，因此服务器必须暴露一个 Anthropic 可以直接或通过 [MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview)访问的 HTTP 端点。要使用只有您的网络可以访问的服务器，请改为让 worker 充当 MCP 客户端，并将服务器的工具声明为自定义工具。MCP 服务器不需要来自您网络外部的入站连接；Anthropic 接收您在智能体上声明的工具定义、每次调用的输入以及您的 worker 回传的结果。在运行时，模型像调用任何其他自定义工具一样调用被包装的工具：

1. 智能体发出一个 `agent.custom_tool_use` 事件。
2. worker 在您的沙箱内，通过其打开的 MCP 会话将调用转发到您网络上的服务器。
3. worker 将服务器的响应作为 `user.custom_tool_result` 发布。

SDK 的 [客户端 MCP 辅助工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector#client-side-mcp-helpers)将服务器的工具转换为 worker 接受的可运行工具；请在 Anthropic SDK 之外安装一个 MCP SDK（`pip install "anthropic[mcp]" "mcp>=1.24"`、`npm install @modelcontextprotocol/sdk`、`go get github.com/modelcontextprotocol/go-sdk`）。示例在不进行身份验证的情况下连接；要发送凭据，请配置您交给 MCP 传输层的 HTTP 客户端或请求选项（Python 中为 `http_client`，TypeScript 中为 `requestInit`，Go 中为 `HTTPClient`）。

<Steps>
  <Step title="在智能体上声明服务器的工具">
    列出 MCP 服务器的工具，并将每一个声明为 `custom` 工具；MCP 的 `name`、`description` 和 `inputSchema` 一一对应到自定义工具的字段。如果服务器对其工具列表进行分页，请声明每一页；worker 必须列出相同的页面。

    <CodeGroup exclude="shell">
      ```python Python
      import asyncio
      from typing import Any, cast
      from anthropic import AsyncAnthropic
      from anthropic.types.beta import BetaManagedAgentsCustomToolParams
      from mcp import ClientSession, types
      # 需要 mcp >= 1.24，该版本将 streamablehttp_client 重命名为 streamable_http_client。
      from mcp.client.streamable_http import streamable_http_client

      MCP_SERVER_URL = "http://mcp.internal.example.com:8000/mcp"


      def to_custom_tool(tool: types.Tool) -> BetaManagedAgentsCustomToolParams:
          # MCP 字段与自定义工具声明一一对应。cast
          # 将 schema 字典原样传递给 SDK 的类型化参数。
          return {
              "type": "custom",
              "name": tool.name,
              "description": tool.description or tool.name,
              "input_schema": cast(Any, tool.inputSchema),
          }


      async def main() -> None:
          # 请在您创建代理的位置运行此代码，而不是在工作主机上：
          # 它使用您的 Claude API 密钥（ANTHROPIC_API_KEY）进行身份验证。
          async with (
              streamable_http_client(MCP_SERVER_URL) as (read, write, _),
              ClientSession(read, write) as mcp_session,
              AsyncAnthropic() as client,
          ):
              await mcp_session.initialize()
              listed = await mcp_session.list_tools()
              agent = await client.beta.agents.create(
                  name="Internal tools agent",
                  model="claude-opus-5",
                  tools=[
                      {"type": "agent_toolset_20260401"},
                      *[to_custom_tool(tool) for tool in listed.tools],
                  ],
              )
              print(agent.id)


      asyncio.run(main())
      ```

      ```typescript TypeScript
      import Anthropic from "@anthropic-ai/sdk";
      import { Client } from "@modelcontextprotocol/sdk/client/index.js";
      import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

      const MCP_SERVER_URL = "http://mcp.internal.example.com:8000/mcp";

      // 请在您创建代理的位置运行此代码，而不是在工作主机上：它
      // 使用您的 Claude API 密钥（ANTHROPIC_API_KEY）进行身份验证。
      const client = new Anthropic();

      const mcpClient = new Client({ name: "declare-agent-tools", version: "1.0.0" });
      await mcpClient.connect(new StreamableHTTPClientTransport(new URL(MCP_SERVER_URL)));
      const { tools } = await mcpClient.listTools();

      const agent = await client.beta.agents.create({
        name: "Internal tools agent",
        model: "claude-opus-5",
        tools: [
          { type: "agent_toolset_20260401" },
          // MCP 字段与自定义工具声明一一对应。
          ...tools.map((tool) => ({
            type: "custom" as const,
            name: tool.name,
            description: tool.description || tool.name,
            input_schema: tool.inputSchema
          }))
        ]
      });
      console.log(agent.id);

      await mcpClient.close();
      ```

      ```csharp C#
      // 请参阅 Python、TypeScript 和 Go 选项卡。使用 MCP 客户端列出服务器的工具后，
      // 在 C# 中声明自定义工具的方式与之相同。
      ```

      ```go Go
      package main

      import (
      	"context"
      	"encoding/json"
      	"fmt"
      	"log"

      	"github.com/anthropics/anthropic-sdk-go"
      	mcpsdk "github.com/modelcontextprotocol/go-sdk/mcp"
      )

      const mcpServerURL = "http://mcp.internal.example.com:8000/mcp"

      // toCustomTool 将一个 MCP 工具定义映射为一个自定义工具声明。
      // 字段一一对应：类型化参数承载 `properties` 和
      // `required`，服务器输出的其他所有 JSON Schema 关键字都放入
      // ExtraFields，以便声明的 schema 与服务器的 schema 保持一致。
      func toCustomTool(tool *mcpsdk.Tool) (anthropic.BetaAgentNewParamsToolUnion, error) {
      	raw, err := json.Marshal(tool.InputSchema)
      	if err != nil {
      		return anthropic.BetaAgentNewParamsToolUnion{}, err
      	}
      	var schema map[string]any
      	if err := json.Unmarshal(raw, &schema); err != nil {
      		return anthropic.BetaAgentNewParamsToolUnion{}, err
      	}

      	inputSchema := anthropic.BetaManagedAgentsCustomToolInputSchemaParam{ExtraFields: map[string]any{}}
      	for keyword, value := range schema {
      		switch keyword {
      		case "type":
      			// 该参数类型始终序列化为 "type": "object"。
      		case "properties":
      			properties, _ := value.(map[string]any)
      			inputSchema.Properties = properties
      		case "required":
      			entries, _ := value.([]any)
      			for _, entry := range entries {
      				if name, isString := entry.(string); isString {
      					inputSchema.Required = append(inputSchema.Required, name)
      				}
      			}
      		default:
      			inputSchema.ExtraFields[keyword] = value
      		}
      	}

      	description := tool.Description
      	if description == "" {
      		description = tool.Name
      	}
      	return anthropic.BetaAgentNewParamsToolUnion{
      		OfCustom: &anthropic.BetaManagedAgentsCustomToolParams{
      			Type:        anthropic.BetaManagedAgentsCustomToolParamsTypeCustom,
      			Name:        tool.Name,
      			Description: description,
      			InputSchema: inputSchema,
      		},
      	}, nil
      }

      func main() {
      	ctx := context.Background()

      	// 请在您创建智能体的位置运行此程序，而非在工作节点主机上：
      	// 它使用您的 Claude API 密钥（ANTHROPIC_API_KEY）进行身份验证。
      	client := anthropic.NewClient()

      	mcpClient := mcpsdk.NewClient(&mcpsdk.Implementation{Name: "declare-agent-tools", Version: "1.0.0"}, nil)
      	session, err := mcpClient.Connect(ctx, &mcpsdk.StreamableClientTransport{Endpoint: mcpServerURL}, nil)
      	if err != nil {
      		log.Fatalf("connect to MCP server: %v", err)
      	}
      	defer session.Close()

      	listed, err := session.ListTools(ctx, nil)
      	if err != nil {
      		log.Fatalf("list MCP tools: %v", err)
      	}

      	tools := []anthropic.BetaAgentNewParamsToolUnion{
      		{OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
      			Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
      		}},
      	}
      	for _, tool := range listed.Tools {
      		custom, err := toCustomTool(tool)
      		if err != nil {
      			log.Fatalf("convert MCP tool %s: %v", tool.Name, err)
      		}
      		tools = append(tools, custom)
      	}

      	agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
      		Name:  "Internal tools agent",
      		Model: anthropic.BetaManagedAgentsModelConfigParams{ID: anthropic.BetaManagedAgentsModelClaudeOpus5},
      		Tools: tools,
      	})
      	if err != nil {
      		log.Fatalf("create agent: %v", err)
      	}
      	fmt.Println(agent.ID)
      }

      ```

      ```java Java
      // 请参阅 Python、TypeScript 和 Go 选项卡。使用 MCP 客户端列出服务器的工具后，
      // 在 Java 中声明自定义工具的方式与之相同。
      ```

      ```php PHP
      // 请参阅 Python、TypeScript 和 Go 选项卡。一旦您使用 MCP 客户端列出服务器的工具，
      // 在 PHP 中声明自定义工具的方式与之相同。
      ```

      ```ruby Ruby
      # 请参阅 Python、TypeScript 和 Go 选项卡。使用 MCP 客户端列出服务器的工具后，
      # 在 Ruby 中声明自定义工具的方式与之相同。
      ```
    </CodeGroup>
  </Step>

  <Step title="从 worker 提供工具">
    在启动时连接到同一个 MCP 服务器，使用 MCP 辅助工具转换其工具，并将它们与内置工具集一起注册。在 worker 的整个生命周期内保持一个 MCP 会话打开。

    <CodeGroup exclude="shell">
      ```python Python
      import asyncio
      import os
      from datetime import timedelta
      from anthropic import AsyncAnthropic
      from anthropic.lib.environments import EnvironmentWorker
      from anthropic.lib.tools.agent_toolset import beta_agent_toolset_20260401
      from anthropic.lib.tools.mcp import async_mcp_tool
      from mcp import ClientSession
      # 需要 mcp >= 1.24，该版本将 streamablehttp_client 重命名为 streamable_http_client。
      from mcp.client.streamable_http import streamable_http_client

      MCP_SERVER_URL = "http://mcp.internal.example.com:8000/mcp"


      async def main() -> None:
          environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
          environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
          # 在启动时连接一次 MCP 服务器，并在工作进程的整个生命周期内
          # 保持会话打开。超时设置会将挂起的工具调用转为错误
          # 结果，而不是让调用一直停滞。
          async with (
              streamable_http_client(MCP_SERVER_URL) as (read, write, _),
              ClientSession(read, write, read_timeout_seconds=timedelta(seconds=60)) as mcp_session,
              AsyncAnthropic(auth_token=environment_key) as client,
          ):
              await mcp_session.initialize()
              listed = await mcp_session.list_tools()
              mcp_tools = [async_mcp_tool(tool, mcp_session) for tool in listed.tools]
              await EnvironmentWorker(
                  client,
                  environment_id=environment_id,
                  environment_key=environment_key,
                  workdir="/workspace",
                  tools=lambda env: [*beta_agent_toolset_20260401(env), *mcp_tools],
              ).run()


      asyncio.run(main())
      ```

      ```typescript TypeScript
      import Anthropic from "@anthropic-ai/sdk";
      import { EnvironmentWorker } from "@anthropic-ai/sdk/helpers/beta/environments";
      import {
        mcpTools,
        type MCPCallToolResultLike,
        type MCPClientLike
      } from "@anthropic-ai/sdk/helpers/beta/mcp";
      import { betaAgentToolset20260401 } from "@anthropic-ai/sdk/tools/agent-toolset/node";
      import { Client } from "@modelcontextprotocol/sdk/client/index.js";
      import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

      const MCP_SERVER_URL = "http://mcp.internal.example.com:8000/mcp";

      const environmentKey = process.env.ANTHROPIC_ENVIRONMENT_KEY!;
      const environmentId = process.env.ANTHROPIC_ENVIRONMENT_ID!;
      const client = new Anthropic({ authToken: environmentKey });
      const controller = new AbortController();
      process.once("SIGTERM", () => controller.abort());

      // 在启动时连接一次 MCP 服务器，并在工作进程的整个生命周期内
      // 保持连接打开。
      const mcpClient = new Client({ name: "sandbox-worker", version: "1.0.0" });
      await mcpClient.connect(new StreamableHTTPClientTransport(new URL(MCP_SERVER_URL)));
      const { tools } = await mcpClient.listTools();

      // MCP SDK 的 callTool 返回类型仍包含 mcpTools 不接受的旧版结果形状；
      // 需要收窄类型。待 MCPClientLike 放宽后删除此处理。
      const mcpClientForTools: MCPClientLike = {
        callTool: (params) => mcpClient.callTool(params) as Promise<MCPCallToolResultLike>
      };

      await new EnvironmentWorker({
        client,
        environmentId,
        environmentKey,
        workdir: "/workspace",
        signal: controller.signal,
        tools: (ctx) => [...betaAgentToolset20260401(ctx), ...mcpTools(tools, mcpClientForTools)]
      }).run();
      ```

      ```csharp C#
      // EnvironmentWorker 目前在 C# SDK 中不可用。
      ```

      ```go Go
      package main

      import (
      	"context"
      	"log"
      	"os"
      	"os/signal"
      	"syscall"

      	"github.com/anthropics/anthropic-sdk-go"
      	"github.com/anthropics/anthropic-sdk-go/lib/environments"
      	"github.com/anthropics/anthropic-sdk-go/mcp"
      	"github.com/anthropics/anthropic-sdk-go/option"
      	"github.com/anthropics/anthropic-sdk-go/tools/agenttoolset"
      	mcpsdk "github.com/modelcontextprotocol/go-sdk/mcp"
      )

      const mcpServerURL = "http://mcp.internal.example.com:8000/mcp"

      func main() {
      	environmentKey := os.Getenv("ANTHROPIC_ENVIRONMENT_KEY")
      	environmentID := os.Getenv("ANTHROPIC_ENVIRONMENT_ID")

      	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
      	defer stop()

      	client := anthropic.NewClient(option.WithAuthToken(environmentKey))

      	// 在启动时连接一次 MCP 服务器，并在工作进程的整个生命周期内
      	// 保持会话打开。
      	mcpClient := mcpsdk.NewClient(&mcpsdk.Implementation{Name: "sandbox-worker", Version: "1.0.0"}, nil)
      	session, err := mcpClient.Connect(ctx, &mcpsdk.StreamableClientTransport{Endpoint: mcpServerURL}, nil)
      	if err != nil {
      		log.Fatalf("connect to MCP server: %v", err)
      	}
      	defer session.Close()

      	listed, err := session.ListTools(ctx, nil)
      	if err != nil {
      		log.Fatalf("list MCP tools: %v", err)
      	}
      	mcpTools, err := mcp.NewBetaTools(listed.Tools, session)
      	if err != nil {
      		log.Fatalf("convert MCP tools: %v", err)
      	}

      	worker := environments.NewEnvironmentWorker(client, environments.EnvironmentWorkerOptions{
      		EnvironmentID:  environmentID,
      		EnvironmentKey: environmentKey,
      		Workdir:        "/workspace",
      		ToolsFunc: func(env *agenttoolset.AgentToolContext) []anthropic.BetaTool {
      			return append(agenttoolset.BetaAgentToolset20260401(env), mcpTools...)
      		},
      	})
      	if err := worker.Run(ctx); err != nil {
      		log.Fatalf("worker: %v", err)
      	}
      }

      ```

      ```java Java
      // Java SDK 目前暂不支持 EnvironmentWorker。
      ```

      ```php PHP
      // PHP SDK 目前不支持 EnvironmentWorker。
      ```

      ```ruby Ruby
      # Ruby SDK 目前不支持 EnvironmentWorker。
      ```
    </CodeGroup>
  </Step>
</Steps>

包装 MCP 服务器时请记住以下几点：

* **工具是声明的，而不是在运行时发现的。** worker 在启动时列出一次 MCP 服务器的工具，并且无法向正在运行的会话添加工具。当服务器的工具发生变化时，请在智能体上或通过 [更新智能体配置](https://platform.claude.com/docs/zh-CN/managed-agents/session-operations#updating-the-agent-configuration) 在空闲会话上重新声明它们，并重启 worker。
* **名称和描述必须符合 Managed Agents API。** 自定义工具名称在每个智能体内唯一，使用字母、数字、下划线和连字符（1–128 个字符）；需要非空描述；并且智能体的 `tools` 数组最多接受 128 个条目（每个被包装的工具是一个条目，内置工具集是另一个条目）。API 会拒绝重复使用工具名称、以内置智能体工具（如 `bash` 或 `read`）命名自定义工具，或使用保留的 `mcp__` 前缀的声明。MCP 辅助工具保留服务器的名称和描述，因此请在需要时重命名或裁剪。当两个服务器暴露相同的工具名称时，请自行以带前缀的名称定义包装器，并让它调用服务器的原始工具名称。
* **大多数 schema 原样通过。** API 接受 MCP 服务器通常发出的 JSON Schema 关键字，例如 `additionalProperties` 和 `title`。它拒绝自定义工具 `input_schema` 中任何位置的引用关键字（例如 `$ref`），因此请将 pydantic 等生成器分解到 `$defs` 中的 schema 内联。它还拒绝顶层的 `oneOf`、`anyOf` 和 `allOf`，以及字母、数字、下划线、点和连字符之外的属性名称（1–64 个字符）。
* **工具失败以错误工具结果的形式呈现。** 当 MCP 服务器报告工具错误时，worker 会发布一个模型可以做出反应的错误工具结果。没有工具结果等价物的 MCP 内容（例如音频块和资源链接）也会以错误的形式呈现。请在 MCP 客户端上设置超时，以获得更快、更清晰的失败，就像 Python worker 示例使用 `read_timeout_seconds` 所做的那样。如果没有超时，挂起的调用只有在 TypeScript MCP SDK 的默认请求超时触发时（大约一分钟）或 worker 自身的兜底机制触发时才会变成错误结果：Python 中大约两分半钟，Go 中为两分钟，此时 worker 会取消超过其 120 秒默认值的工具调用并发布错误结果。
* **包装您运营或信任的服务器。** 被包装工具的名称、描述和结果会像任何其他工具一样进入模型的上下文：这是不受信任的输入，可能影响智能体使用其他工具（包括 worker 主机上的 `bash`）所做的事情。只声明您打算让智能体使用的工具。
* **权限策略不适用于自定义工具。** [权限策略](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies#custom-tools)管理内置和 MCP 工具集；worker 会执行模型发出的每一个被包装工具调用，因此请将任何审批步骤放在您自己的工具代码中。

## 监控与运维

这些调用从您的监控或运维工具运行，使用您的 Claude API 密钥进行身份验证，以观察和管理 worker 集群。领取和保活循环在 worker 辅助工具内部处理，因此您不需要直接调用这些端点。

<Warning>
  这些端点接受您的组织 API 密钥或环境密钥。请从 worker 主机外部使用您的组织 API 密钥调用它们。在 worker 主机上设置 `ANTHROPIC_API_KEY` 会将组织范围的凭据暴露给智能体工具调用。
</Warning>

### 读取队列深度

`work.stats` 返回环境的队列状态：

* `depth` 是等待被领取的项数。请根据此值扩展您的 worker 集群或对积压发出告警。
* `pending` 是已被 worker 领取但尚未确认的项数。worker 辅助工具在处理每个项之前都会确认它，因此在正常运行中此值保持接近零；持续的非零值意味着某个 worker 在领取和确认之间停滞了。
* `oldest_queued_at` 是仍在队列中的最旧项的时间戳（等待被领取，或已领取但尚未确认），如果没有则为 `null`。
* `workers_polling` 是在过去 30 秒内进行过轮询的 worker 数量。请将其用于存活告警。

<CodeGroup>
  ```bash cURL
  curl -sS "https://api.anthropic.com/v1/environments/$ANTHROPIC_ENVIRONMENT_ID/work/stats" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:environments:work stats --environment-id "$ANTHROPIC_ENVIRONMENT_ID"
  ```

  ```python Python
  import os

  import anthropic

  client = anthropic.Anthropic()

  stats = client.beta.environments.work.stats(os.environ["ANTHROPIC_ENVIRONMENT_ID"])
  print(f"depth={stats.depth} pending={stats.pending}")
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";

  const client = new Anthropic();

  const stats = await client.beta.environments.work.stats(process.env.ANTHROPIC_ENVIRONMENT_ID!);

  console.log(`depth=${stats.depth} pending=${stats.pending}`);
  ```

  ```csharp C#
  using Anthropic;

  var client = new AnthropicClient();

  var environmentId = Environment.GetEnvironmentVariable("ANTHROPIC_ENVIRONMENT_ID")!;

  var stats = await client.Beta.Environments.Work.Stats(environmentId);

  Console.WriteLine($"depth={stats.Depth} pending={stats.Pending}");
  ```

  ```go Go
  package main

  import (
  	"context"
  	"fmt"
  	"os"

  	"github.com/anthropics/anthropic-sdk-go"
  )

  func main() {
  	client := anthropic.NewClient()
  	environmentID := os.Getenv("ANTHROPIC_ENVIRONMENT_ID")

  	stats, err := client.Beta.Environments.Work.Stats(
  		context.Background(),
  		environmentID,
  		anthropic.BetaEnvironmentWorkStatsParams{},
  	)
  	if err != nil {
  		panic(err)
  	}

  	fmt.Printf("depth=%d pending=%d\n", stats.Depth, stats.Pending)
  }
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.models.beta.environments.work.BetaSelfHostedWorkQueueStats;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      BetaSelfHostedWorkQueueStats stats = client.beta()
          .environments()
          .work()
          .stats(System.getenv("ANTHROPIC_ENVIRONMENT_ID"));

      IO.println("depth=" + stats.depth() + " pending=" + stats.pending());
  }
  ```

  ```php PHP
  <?php

  use Anthropic\Client;

  $client = new Client();

  $stats = $client->beta->environments->work->stats(getenv('ANTHROPIC_ENVIRONMENT_ID'));

  printf("depth=%d pending=%d\n", $stats->depth, $stats->pending);
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::Client.new

  stats = client.beta.environments.work.stats(ENV.fetch("ANTHROPIC_ENVIRONMENT_ID"))

  puts "depth=#{stats.depth} pending=#{stats.pending}"
  ```
</CodeGroup>

```text wrap
{
  "type": "work_queue_stats",
  "depth": 0,
  "pending": 0,
  "oldest_queued_at": null,
  "workers_polling": 0
}
```

### 优雅地停止会话

使用 `work.stop` 请求处理特定会话的 worker 将其关闭。默认情况下，工作项会转为 `stopping`：worker 在其下一次租约心跳时注意到这一点，取消会话正在进行的工具调用，并确认关闭，此时工作项变为 `stopped`。在请求体中传入 `force: true`（使用 CLI 时传入 `--force`）可立即将工作项标记为 `stopped`，而不是等待 worker 的确认。

由于这些调用从您的运维工具而不是 worker 主机运行，因此 `ANTHROPIC_WORK_ID` 不会自动设置。在运行以下示例之前，请将其设置为目标工作项的 ID。要查找工作项的 ID，请通过 [Environments Work 端点](https://platform.claude.com/docs/zh-CN/api/beta/environments/work)列出环境的工作项。

<CodeGroup>
  ```bash cURL
  curl -sS "https://api.anthropic.com/v1/environments/$ANTHROPIC_ENVIRONMENT_ID/work/$ANTHROPIC_WORK_ID/stop" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{}'
  ```

  ```bash CLI
  ant beta:environments:work stop \
    --environment-id "$ANTHROPIC_ENVIRONMENT_ID" \
    --work-id "$ANTHROPIC_WORK_ID"
  ```

  ```python Python
  import os

  import anthropic

  client = anthropic.Anthropic()

  work = client.beta.environments.work.stop(
      os.environ["ANTHROPIC_WORK_ID"],
      environment_id=os.environ["ANTHROPIC_ENVIRONMENT_ID"],
  )
  print(work.state)
  ```

  ```typescript TypeScript
  import Anthropic from "@anthropic-ai/sdk";

  const client = new Anthropic();

  const work = await client.beta.environments.work.stop(process.env.ANTHROPIC_WORK_ID!, {
    environment_id: process.env.ANTHROPIC_ENVIRONMENT_ID!
  });

  console.log(work.state);
  ```

  ```csharp C#
  using Anthropic;

  var client = new AnthropicClient();

  var work = await client.Beta.Environments.Work.Stop(
      Environment.GetEnvironmentVariable("ANTHROPIC_WORK_ID")!,
      new()
      {
          EnvironmentID = Environment.GetEnvironmentVariable("ANTHROPIC_ENVIRONMENT_ID")!
      }
  );

  Console.WriteLine(work.State);
  ```

  ```go Go
  package main

  import (
  	"context"
  	"fmt"
  	"os"

  	"github.com/anthropics/anthropic-sdk-go"
  )

  func main() {
  	client := anthropic.NewClient()

  	work, err := client.Beta.Environments.Work.Stop(
  		context.Background(),
  		os.Getenv("ANTHROPIC_WORK_ID"),
  		anthropic.BetaEnvironmentWorkStopParams{
  			EnvironmentID: os.Getenv("ANTHROPIC_ENVIRONMENT_ID"),
  		},
  	)
  	if err != nil {
  		panic(err)
  	}
  	fmt.Println(work.State)
  }
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.models.beta.environments.work.BetaSelfHostedWork;
  import com.anthropic.models.beta.environments.work.BetaSelfHostedWorkStopRequest;
  import com.anthropic.models.beta.environments.work.WorkStopParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      BetaSelfHostedWork work = client.beta().environments().work().stop(
          WorkStopParams.builder()
              .environmentId(System.getenv("ANTHROPIC_ENVIRONMENT_ID"))
              .workId(System.getenv("ANTHROPIC_WORK_ID"))
              .betaSelfHostedWorkStopRequest(BetaSelfHostedWorkStopRequest.builder().build())
              .build()
      );

      IO.println(work.state());
  }
  ```

  ```php PHP
  <?php

  use Anthropic\Client;

  $client = new Client();

  $work = $client->beta->environments->work->stop(
      getenv('ANTHROPIC_WORK_ID'),
      environmentID: getenv('ANTHROPIC_ENVIRONMENT_ID'),
  );

  echo $work->state . "\n";
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::Client.new

  work = client.beta.environments.work.stop(
    ENV.fetch("ANTHROPIC_WORK_ID"),
    environment_id: ENV.fetch("ANTHROPIC_ENVIRONMENT_ID")
  )

  puts work.state
  ```
</CodeGroup>

## 后续步骤

<CardGroup cols={2}>
  <Card title="安全模型" icon="lock" href="https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes-security">
    自托管沙箱环境的共担责任模型。
  </Card>

  <Card title="启动会话" icon="settings" href="https://platform.claude.com/docs/zh-CN/managed-agents/sessions">
    创建一个会话来运行您的智能体并开始执行任务。
  </Card>

  <Card title="MCP 隧道" icon="bolt" href="https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview">
    将 Claude 安全地连接到在您的私有网络中运行的 MCP 服务器，而无需开放入站端口或将服务暴露到公共互联网。
  </Card>
</CardGroup>
