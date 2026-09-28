---
title: 云环境设置
url: https://platform.claude.com/docs/zh-CN/managed-agents/environments
description: 为您的会话自定义云沙箱。
featureMetadata:
  topic:
    title: Managed Agents
    url: https://platform.claude.com/docs/en/managed-agents/overview
  status: beta
  betaHeader: managed-agents-2026-04-01
---

环境（environment）定义了您的智能体运行所在的沙箱配置。您只需创建一次环境，然后在每次启动会话时引用其 ID。多个会话可以共享同一个环境，但每个会话都会获得自己独立隔离的沙箱（一个全新的 Linux 容器）。

本页介绍 `type: cloud` 类型的环境。若要在您自己的基础设施上运行沙箱，请参阅[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes)。

## 创建环境

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -fsS https://api.anthropic.com/v1/environments \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    --data @- <<'EOF'
  {
    "name": "python-dev",
    "config": {
      "type": "cloud",
      "networking": {"type": "unrestricted"}
    }
  }
  EOF
  ```

  <CodeGroupItem>
    ```bash CLI
    ant apply environment.yaml
    ```

    <File filename="environment.yaml">
      ```yaml
      # yaml-language-server: $schema=https://platform.claude.com/schemas/ant/beta/environment.json
      name: python-dev
      config:
        type: cloud
        networking:
          type: unrestricted
      ```
    </File>

    [`ant apply`](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/apply) 会根据 `environment.yaml` 创建环境，打印其 ID，并将其记录在 `claude-lock.json` 中。请提交 `claude-lock.json`，这样下一次运行 `ant apply` 时会更新此环境，而不是尝试再次创建它。
  </CodeGroupItem>

  ```python Python
  environment = client.beta.environments.create(
      name="python-dev",
      config={
          "type": "cloud",
          "networking": {"type": "unrestricted"},
      },
  )

  print(f"Environment ID: {environment.id}")
  ```

  ```typescript TypeScript
  const environment = await client.beta.environments.create({
    name: "python-dev",
    config: {
      type: "cloud",
      networking: { type: "unrestricted" },
    },
  });

  console.log(`Environment ID: ${environment.id}`);
  ```

  ```csharp C#
  var environment = await client.Beta.Environments.Create(new()
  {
      Name = "python-dev",
      Config = new BetaCloudConfigParams
      {
          Networking = new BetaUnrestrictedNetwork(),
      },
  });

  Console.WriteLine($"Environment ID: {environment.ID}");
  ```

  ```go Go
  environment, err := client.Beta.Environments.New(ctx, anthropic.BetaEnvironmentNewParams{
  	Name: "python-dev",
  	Config: anthropic.BetaEnvironmentNewParamsConfigUnion{
  		OfCloud: &anthropic.BetaCloudConfigParams{
  			Networking: anthropic.BetaCloudConfigParamsNetworkingUnion{
  				OfUnrestricted: &anthropic.BetaUnrestrictedNetworkParam{},
  			},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }

  fmt.Printf("Environment ID: %s\n", environment.ID)
  ```

  ```java Java
  var environment = client.beta().environments().create(EnvironmentCreateParams.builder()
      .name("python-dev")
      .config(BetaCloudConfigParams.builder()
          .networking(BetaUnrestrictedNetwork.builder().build())
          .build())
      .build());
  IO.println("Environment ID: " + environment.id());
  ```

  ```php PHP
  $environment = $client->beta->environments->create(
      name: 'python-dev',
      config: ['type' => 'cloud', 'networking' => ['type' => 'unrestricted']],
  );
  echo "Environment ID: {$environment->id}\n";
  ```

  ```ruby Ruby
  environment = client.beta.environments.create(
    name: "python-dev",
    config: {
      type: "cloud",
      networking: {type: "unrestricted"}
    }
  )

  puts "Environment ID: #{environment.id}"
  ```
</CodeGroup>

请使用唯一且具有描述性的 `name`，以便区分不同的环境。

## 在会话中使用环境

在[创建会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)时，以字符串形式传入环境 ID。

<CodeGroup>
  ```bash cURL
  curl -fsS https://api.anthropic.com/v1/sessions \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    --data @- <<EOF
  {
    "agent": "$AGENT_ID",
    "environment_id": "$ENVIRONMENT_ID"
  }
  EOF
  ```

  ```bash CLI
  ant beta:sessions create --agent "$AGENT_ID" --environment-id "$ENVIRONMENT_ID"
  ```

  ```python Python
  session = client.beta.sessions.create(
      agent=agent.id,
      environment_id=environment.id,
  )
  ```

  ```typescript TypeScript
  const session = await client.beta.sessions.create({
    agent: agent.id,
    environment_id: environment.id,
  });
  ```

  ```csharp C#
  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = agent.ID,
      EnvironmentID = environment.ID,
  });
  ```

  ```go Go
  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent: anthropic.BetaSessionNewParamsAgentUnion{
  		OfString: anthropic.String(agent.ID),
  	},
  	EnvironmentID: environment.ID,
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var session = client.beta().sessions().create(SessionCreateParams.builder()
      .agent(agent.id())
      .environmentId(environment.id())
      .build());
  ```

  ```php PHP
  $session = $client->beta->sessions->create(
      agent: $agent->id,
      environmentID: $environment->id,
  );
  ```

  ```ruby Ruby
  session = client.beta.sessions.create(
    agent: agent.id,
    environment_id: environment.id
  )
  ```
</CodeGroup>

## 配置选项

### 软件包

`packages` 字段会在智能体启动之前将软件包预安装到沙箱中。软件包由各自对应的包管理器安装，并在共享同一环境的会话之间进行缓存。当指定了多个包管理器时，它们按字母顺序运行（apt、cargo、gem、go、npm、pip）。您可以选择固定特定版本。未固定版本的软件包将安装最新版本。如果环境使用 `limited` [网络](https://platform.claude.com/docs/zh-CN/managed-agents/environments#networking)模式，还需将 `networking.allow_package_managers` 设置为 `true`；否则请求将被拒绝并返回 400 错误。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -fsS https://api.anthropic.com/v1/environments \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    --data @- <<'EOF'
  {
    "name": "data-analysis",
    "config": {
      "type": "cloud",
      "packages": {
        "pip": ["pandas", "numpy", "scikit-learn"],
        "npm": ["express"]
      },
      "networking": {"type": "unrestricted"}
    }
  }
  EOF
  ```

  <CodeGroupItem>
    ```bash CLI
    ant apply environment.yaml
    ```

    <File filename="environment.yaml">
      ```yaml
      # yaml-language-server: $schema=https://platform.claude.com/schemas/ant/beta/environment.json
      name: data-analysis
      config:
        type: cloud
        packages:
          pip:
            - pandas
            - numpy
            - scikit-learn
          npm:
            - express
        networking:
          type: unrestricted
      ```
    </File>
  </CodeGroupItem>

  ```python Python
  environment = client.beta.environments.create(
      name="data-analysis",
      config={
          "type": "cloud",
          "packages": {
              "pip": ["pandas", "numpy", "scikit-learn"],
              "npm": ["express"],
          },
          "networking": {"type": "unrestricted"},
      },
  )
  ```

  ```typescript TypeScript
  const environment = await client.beta.environments.create({
    name: "data-analysis",
    config: {
      type: "cloud",
      packages: {
        pip: ["pandas", "numpy", "scikit-learn"],
        npm: ["express"]
      },
      networking: { type: "unrestricted" }
    }
  });
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Environments;

  var environment = await client.Beta.Environments.Create(new()
  {
      Name = "data-analysis",
      Config = new BetaCloudConfigParams
      {
          Packages = new()
          {
              Pip = ["pandas", "numpy", "scikit-learn"],
              Npm = ["express"],
          },
          Networking = new BetaUnrestrictedNetwork(),
      },
  });
  ```

  ```go Go
  environment, err := client.Beta.Environments.New(ctx, anthropic.BetaEnvironmentNewParams{
  	Name: "data-analysis",
  	Config: anthropic.BetaEnvironmentNewParamsConfigUnion{
  		OfCloud: &anthropic.BetaCloudConfigParams{
  			Packages: anthropic.BetaPackagesParams{
  				Pip: []string{"pandas", "numpy", "scikit-learn"},
  				Npm: []string{"express"},
  			},
  			Networking: anthropic.BetaCloudConfigParamsNetworkingUnion{
  				OfUnrestricted: &anthropic.BetaUnrestrictedNetworkParam{},
  			},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  _ = environment
  ```

  ```java Java
  import com.anthropic.models.beta.environments.*;
  import java.util.List;

  var environment = client.beta().environments().create(EnvironmentCreateParams.builder()
      .name("data-analysis")
      .config(BetaCloudConfigParams.builder()
          .packages(BetaPackagesParams.builder()
              .pip(List.of("pandas", "numpy", "scikit-learn"))
              .npm(List.of("express"))
              .build())
          .networking(BetaUnrestrictedNetwork.builder().build())
          .build())
      .build());
  ```

  ```php PHP
  $environment = $client->beta->environments->create(
      name: 'data-analysis',
      config: [
          'type' => 'cloud',
          'packages' => [
              'pip' => ['pandas', 'numpy', 'scikit-learn'],
              'npm' => ['express'],
          ],
          'networking' => ['type' => 'unrestricted'],
      ],
  );
  ```

  ```ruby Ruby
  environment = client.beta.environments.create(
    name: "data-analysis",
    config: {
      type: "cloud",
      packages: {
        pip: %w[pandas numpy scikit-learn],
        npm: %w[express]
      },
      networking: {type: "unrestricted"}
    }
  )
  ```
</CodeGroup>

支持的包管理器：

| 字段      | 包管理器           | 示例                                          |
| ------- | -------------- | ------------------------------------------- |
| `apt`   | 系统软件包（apt-get） | `"graphviz"`                                |
| `cargo` | Rust（cargo）    | `"hyperfine@1.18.0"`                        |
| `gem`   | Ruby（gem）      | `"rails:7.1.0"`                             |
| `go`    | Go 模块          | `"golang.org/x/tools/cmd/goimports@latest"` |
| `npm`   | Node.js（npm）   | `"express@4.18.0"`                          |
| `pip`   | Python（pip）    | `"sqlalchemy==2.0.30"`                      |

### 网络

`networking` 字段控制沙箱的出站网络访问。它不会影响 `web_search` 或 `web_fetch` 工具，这些工具运行在 Anthropic 的服务器上；若要限制这些工具可访问的站点，请在智能体工具集中该工具的条目上设置 `allowed_domains` 或 `blocked_domains`。请参阅[限制网页搜索和网页抓取的域名](https://platform.claude.com/docs/zh-CN/managed-agents/tools#restrict-web-search-and-web-fetch-domains)。

| 模式             | 描述                                                                                                    |
| -------------- | ----------------------------------------------------------------------------------------------------- |
| `unrestricted` | 完全的出站网络访问，但受通用安全阻止列表限制。这是默认值。                                                                         |
| `limited`      | 将沙箱网络访问限制为 `allowed_hosts` 中的主机。将 `allow_package_managers` 和 `allow_mcp_servers` 设置为 `true` 可允许额外的访问。 |

以下示例创建一个使用 `limited` 网络模式的环境：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -fsS https://api.anthropic.com/v1/environments \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    -d '{
      "name": "api-access",
      "config": {
        "type": "cloud",
        "networking": {
          "type": "limited",
          "allowed_hosts": ["api.example.com"],
          "allow_mcp_servers": true,
          "allow_package_managers": true
        }
      }
    }'
  ```

  <CodeGroupItem>
    ```bash CLI
    ant apply environment.yaml
    ```

    <File filename="environment.yaml">
      ```yaml
      # yaml-language-server: $schema=https://platform.claude.com/schemas/ant/beta/environment.json
      name: api-access
      config:
        type: cloud
        networking:
          type: limited
          allowed_hosts:
            - api.example.com
          allow_mcp_servers: true
          allow_package_managers: true
      ```
    </File>
  </CodeGroupItem>

  ```python Python
  environment = client.beta.environments.create(
      name="api-access",
      config={
          "type": "cloud",
          "networking": {
              "type": "limited",
              "allowed_hosts": ["api.example.com"],
              "allow_mcp_servers": True,
              "allow_package_managers": True,
          },
      },
  )
  ```

  ```typescript TypeScript
  const environment = await client.beta.environments.create({
    name: "api-access",
    config: {
      type: "cloud",
      networking: {
        type: "limited",
        allowed_hosts: ["api.example.com"],
        allow_mcp_servers: true,
        allow_package_managers: true
      }
    }
  });
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Environments;

  var environment = await client.Beta.Environments.Create(new()
  {
      Name = "api-access",
      Config = new BetaCloudConfigParams
      {
          Networking = new BetaLimitedNetworkParams
          {
              AllowedHosts = ["api.example.com"],
              AllowMcpServers = true,
              AllowPackageManagers = true,
          },
      },
  });
  ```

  ```go Go
  environment, err := client.Beta.Environments.New(ctx, anthropic.BetaEnvironmentNewParams{
  	Name: "api-access",
  	Config: anthropic.BetaEnvironmentNewParamsConfigUnion{
  		OfCloud: &anthropic.BetaCloudConfigParams{
  			Networking: anthropic.BetaCloudConfigParamsNetworkingUnion{
  				OfLimited: &anthropic.BetaLimitedNetworkParams{
  					AllowedHosts:         []string{"api.example.com"},
  					AllowMCPServers:      anthropic.Bool(true),
  					AllowPackageManagers: anthropic.Bool(true),
  				},
  			},
  		},
  	},
  })
  if err != nil {
  	panic(err)
  }
  _ = environment
  ```

  ```java Java
  import com.anthropic.models.beta.environments.*;
  import java.util.List;

  var environment = client.beta().environments().create(EnvironmentCreateParams.builder()
      .name("api-access")
      .config(BetaCloudConfigParams.builder()
          .networking(BetaLimitedNetworkParams.builder()
              .allowedHosts(List.of("api.example.com"))
              .allowMcpServers(true)
              .allowPackageManagers(true)
              .build())
          .build())
      .build());
  ```

  ```php PHP
  $environment = $client->beta->environments->create(
      name: 'api-access',
      config: [
          'type' => 'cloud',
          'networking' => [
              'type' => 'limited',
              'allowed_hosts' => ['api.example.com'],
              'allow_mcp_servers' => true,
              'allow_package_managers' => true,
          ],
      ],
  );
  ```

  ```ruby Ruby
  environment = client.beta.environments.create(
    name: "api-access",
    config: {
      type: "cloud",
      networking: {
        type: "limited",
        allowed_hosts: %w[api.example.com],
        allow_mcp_servers: true,
        allow_package_managers: true
      }
    }
  )
  ```
</CodeGroup>

<Info>
  对于生产部署，请使用 `limited` 网络模式并配合明确的 `allowed_hosts` 列表。遵循最小权限原则，仅授予智能体所需的最低网络访问权限，并定期审核您允许的域名。
</Info>

使用 `limited` 网络模式时：

* `allowed_hosts` 指定沙箱可以访问的域名。请指定纯主机名或通配符模式（例如 `*.example.com`）。不要包含 URL 协议、端口或路径。
* `allow_mcp_servers` 允许对智能体上配置的 MCP 服务器端点进行出站访问，这些端点不必列在 `allowed_hosts` 数组中。默认为 `false`。
* `allow_package_managers` 允许对一组公共软件包注册表和代码托管站点进行出站访问，这些主机不必列在 `allowed_hosts` 数组中。有关列表，请参阅[包管理器主机](https://platform.claude.com/docs/zh-CN/managed-agents/environments#package-manager-hosts)。默认为 `false`。只要环境指定了 `packages`，就请将其设置为 `true`；否则请求将被拒绝并返回 400 错误，即使注册表主机已列在 `allowed_hosts` 中也是如此。

#### 包管理器主机

当 `allow_package_managers` 为 `true` 时，除 `allowed_hosts` 中的主机外，沙箱还可以访问以下主机。此列表由 Anthropic 维护，并可能发生变更。

| 生态系统        | 主机                                                                                                                                                                                  |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 代码托管        | `github.com`、`api.github.com`、`codeload.github.com`、`raw.githubusercontent.com`、`objects.githubusercontent.com`、`release-assets.githubusercontent.com`、`gitlab.com`、`bitbucket.org` |
| Node.js     | `registry.npmjs.org`、`registry.yarnpkg.com`、`nodejs.org`                                                                                                                            |
| Python      | `pypi.org`、`files.pythonhosted.org`                                                                                                                                                 |
| Rust        | `crates.io`、`index.crates.io`、`static.crates.io`、`static.rust-lang.org`                                                                                                             |
| Go          | `proxy.golang.org`、`sum.golang.org`                                                                                                                                                 |
| Java        | `repo1.maven.org`、`repo.maven.apache.org`、`services.gradle.org`、`plugins.gradle.org`、`plugins-artifacts.gradle.org`                                                                 |
| Ruby        | `rubygems.org`、`index.rubygems.org`                                                                                                                                                 |
| PHP         | `packagist.org`、`repo.packagist.org`                                                                                                                                                |
| Ubuntu（apt） | `archive.ubuntu.com`、`security.ubuntu.com`、`ppa.launchpad.net`                                                                                                                      |
| 容器          | `registry-1.docker.io`、`auth.docker.io`、`production.cloudflare.docker.com`、`download.docker.com`、`ghcr.io`                                                                          |

<Warning>
  网络访问是按主机授予的，而不是按操作授予的。沙箱可以使用命令提供的任何凭据向允许的主机发送任何请求，包括 `git push` 和发布软件包等上传操作。如果智能体处理不受信任的输入（仓库文件、获取的网页内容或第三方工具输出），一次成功的 "prompt injection"（提示注入）可能会利用允许的主机将文件复制到沙箱之外。为降低此风险，请将 `bash` 工具的[权限策略](https://platform.claude.com/docs/zh-CN/managed-agents/permission-policies)设置为 `always_ask` 或 `auto`。如果环境未指定 `packages`，您也可以将 `allow_package_managers` 保持为 `false`，并在 `allowed_hosts` 中仅列出您的智能体所需的主机。
</Warning>

## 环境生命周期

* 环境会一直保留，直到被显式归档或删除。
* 每个会话都会获得自己的沙箱实例，即使多个会话引用同一个环境也是如此。会话之间不共享文件系统状态。
* 环境不进行版本管理。如果您频繁更新环境，请自行记录变更，以便了解每个会话使用的是哪种配置。

## 管理环境

<CodeGroup>
  ```bash cURL
  # 列出环境
  curl -fsS https://api.anthropic.com/v1/environments \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"

  # 检索特定环境
  curl -fsS "https://api.anthropic.com/v1/environments/$ENVIRONMENT_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"

  # 归档环境（只读，现有会话继续运行）
  curl -fsS -X POST "https://api.anthropic.com/v1/environments/$ENVIRONMENT_ID/archive" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"

  # 删除环境（仅当没有会话引用它时）
  curl -fsS -X DELETE "https://api.anthropic.com/v1/environments/$ENVIRONMENT_ID" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01"
  ```

  ```bash CLI
  # 列出 environments
  ant beta:environments list

  # 获取指定的 environment
  ant beta:environments retrieve --environment-id "$ENVIRONMENT_ID"

  # 归档 environment（只读，现有会话继续运行）
  ant beta:environments archive --environment-id "$ENVIRONMENT_ID"

  # 删除 environment（仅当没有会话引用它时）
  ant beta:environments delete --environment-id "$ENVIRONMENT_ID"
  ```

  ```python Python
  # 列出环境
  environments = client.beta.environments.list()

  # 检索特定环境
  env = client.beta.environments.retrieve(environment.id)

  # 归档环境（只读，现有会话继续运行）
  client.beta.environments.archive(environment.id)

  # 删除环境（仅当没有会话引用它时）
  client.beta.environments.delete(environment.id)
  ```

  ```typescript TypeScript
  // 列出环境
  const environments = await client.beta.environments.list();

  // 检索特定环境
  const env = await client.beta.environments.retrieve(environment.id);

  // 归档环境（只读，现有会话继续运行）
  await client.beta.environments.archive(environment.id);

  // 删除环境（仅当没有会话引用它时）
  await client.beta.environments.delete(environment.id);
  ```

  ```csharp C#
  // 列出环境
  var environments = await client.Beta.Environments.List();

  // 检索特定环境
  var env = await client.Beta.Environments.Retrieve(environment.ID);

  // 归档环境（只读，现有会话继续运行）
  await client.Beta.Environments.Archive(environment.ID);

  // 删除环境（仅当没有会话引用它时）
  await client.Beta.Environments.Delete(environment.ID);
  ```

  ```go Go
  // 列出环境
  environments, err := client.Beta.Environments.List(ctx, anthropic.BetaEnvironmentListParams{})
  // ...

  // 检索特定环境
  env, err := client.Beta.Environments.Get(ctx, environment.ID, anthropic.BetaEnvironmentGetParams{})
  // ...

  // 归档环境（只读，现有会话继续运行）
  _, err = client.Beta.Environments.Archive(ctx, environment.ID, anthropic.BetaEnvironmentArchiveParams{})
  // ...

  // 删除环境（仅当没有会话引用它时）
  _, err = client.Beta.Environments.Delete(ctx, environment.ID, anthropic.BetaEnvironmentDeleteParams{})
  ```

  ```java Java
  // 列出环境
  var environments = client.beta().environments().list();
  // 检索特定环境
  var env = client.beta().environments().retrieve(environment.id());
  // 归档环境（只读，现有会话继续运行）
  client.beta().environments().archive(environment.id());
  // 删除环境（仅当没有会话引用它时）
  client.beta().environments().delete(environment.id());
  ```

  ```php PHP
  // 列出环境
  $environments = $client->beta->environments->list();
  // 检索特定环境
  $env = $client->beta->environments->retrieve($environment->id);
  // 归档环境（只读，现有会话继续运行）
  $client->beta->environments->archive($environment->id);
  // 删除环境（仅当没有会话引用它时）
  $client->beta->environments->delete($environment->id);
  ```

  ```ruby Ruby
  # 列出环境
  environments = client.beta.environments.list

  # 检索特定环境
  env = client.beta.environments.retrieve(environment.id)

  # 归档环境（只读，现有会话继续运行）
  client.beta.environments.archive(environment.id)

  # 删除环境（仅当没有会话引用它时）
  client.beta.environments.delete(environment.id)
  ```
</CodeGroup>

## 预安装的运行时

云沙箱开箱即包含常用的语言运行时、数据库和命令行工具。完整列表请参阅[云沙箱参考](https://platform.claude.com/docs/zh-CN/managed-agents/cloud-sandboxes-reference)。

## 后续步骤

<CardGroup cols={2}>
  <Card title="云沙箱参考" icon="book" href="https://platform.claude.com/docs/zh-CN/managed-agents/cloud-sandboxes-reference">
    云沙箱中可用的预安装软件包、数据库和实用工具。
  </Card>

  <Card title="启动会话" icon="play" href="https://platform.claude.com/docs/zh-CN/managed-agents/sessions">
    创建会话以运行您的智能体并开始执行任务。
  </Card>
</CardGroup>
