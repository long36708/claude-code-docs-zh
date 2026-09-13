---
title: 使用智能体记忆
url: https://platform.claude.com/docs/zh-CN/managed-agents/memory
description: 使用记忆存储为您的智能体提供可跨会话持久保存的记忆。
---

默认情况下，每个 Managed Agents 会话都以全新的上下文开始。当会话结束时，智能体积累的任何状态都会消失。"Memory store"（记忆存储）让智能体能够跨会话携带信息：用户偏好、项目约定、先前的错误以及领域上下文。

<Note>
  Managed Agents API 请求需要 `managed-agents-2026-04-01` beta 头，但记忆存储（memory store）端点除外，这些端点改用 `agent-memory-2026-07-22`。SDK 会自动设置正确的 beta 头。请参阅 [Beta 头](https://platform.claude.com/docs/zh-CN/api/beta-headers#endpoint-specific-headers)。
</Note>

<Note>
  不要在记忆存储请求中将 `agent-memory-2026-07-22` 与 `managed-agents-2026-04-01` 组合使用：同时发送两者会返回 `400` 错误。如果您的代码显式设置了 beta 请求头，请在记忆存储调用中将 `managed-agents-2026-04-01` 替换为 `agent-memory-2026-07-22`，而不是添加第二个值。会话端点（包括将记忆存储附加到会话）仍然使用 `managed-agents-2026-04-01`。

  `GET /v1/memory_stores/{memory_store_id}/memories` 在任一请求头下的行为相同：结果以稳定的、由服务器定义的顺序返回，并且 `path_prefix` 和 `depth` 的应用方式相同。
</Note>

## 概述

**记忆存储**是一个工作区范围内、针对 Claude 优化的文本文档集合。当您将存储附加到会话时，它会作为一个目录挂载到会话的沙箱中。智能体使用与操作文件系统其余部分相同的文件工具来读写它，并且描述每个挂载的说明会自动添加到系统提示中，告诉智能体应该在哪里查找。这些交互需要[智能体工具集](https://platform.claude.com/docs/zh-CN/managed-agents/tools)；请确保在[创建智能体](https://platform.claude.com/docs/zh-CN/managed-agents/agent-setup)时启用它。在[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)上，该目录不是实时挂载。相反，SDK 的环境工作进程会在智能体的工具运行之前将每个附加的存储下载到您的沙箱中，并使该副本与存储保持同步。

存储中的每条**记忆**（memory）都通过路径寻址，可以直接通过 API 或 Claude Console 读取和编辑，从而支持调优、导入和导出。

对记忆的每次更改都会创建一个不可变的**记忆版本**（memory version），为智能体写入的所有内容提供审计跟踪和时间点恢复能力。

## 创建记忆存储

为存储指定 `name` 和 `description`。描述会传递给智能体，告诉它存储中包含什么内容。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  store=$(curl -s https://api.anthropic.com/v1/memory_stores \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" \
    -H "content-type: application/json" \
    -d '{"name": "User Preferences", "description": "Per-user preferences and project context."}')
  store_id=$(jq -r '.id' <<< "$store")
  echo "$store_id"  # memstore_01Hx...
  ```

  ```bash CLI
  store_id=$(ant beta:memory-stores create \
    --name "User Preferences" \
    --description "Per-user preferences and project context." \
    --transform id --raw-output)
  ```

  ```python Python
  store = client.beta.memory_stores.create(
      name="User Preferences",
      description="Per-user preferences and project context.",
  )
  print(store.id)  # memstore_01Hx...
  ```

  ```typescript TypeScript
  const store = await client.beta.memoryStores.create({
    name: "User Preferences",
    description: "Per-user preferences and project context."
  });
  console.log(store.id); // memstore_01Hx...
  ```

  ```csharp C#
  var store = await client.Beta.MemoryStores.Create(new()
  {
      Name = "User Preferences",
      Description = "Per-user preferences and project context.",
  });
  Console.WriteLine(store.ID);  // memstore_01Hx...
  ```

  ```go Go
  store, err := client.Beta.MemoryStores.New(ctx, anthropic.BetaMemoryStoreNewParams{
  	Name:        "User Preferences",
  	Description: anthropic.String("Per-user preferences and project context."),
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(store.ID) // memstore_01Hx...
  ```

  ```java Java
  var store = client.beta().memoryStores().create(
      MemoryStoreCreateParams.builder()
          .name("User Preferences")
          .description("Per-user preferences and project context.")
          .build()
  );
  IO.println(store.id());  // memstore_01Hx...
  ```

  ```php PHP
  use Anthropic\Client;

  $client = new Client();

  $store = $client->beta->memoryStores->create(
      name: 'User Preferences',
      description: 'Per-user preferences and project context.',
  );
  echo "{$store->id}\n"; // memstore_01Hx...
  ```

  ```ruby Ruby
  require "anthropic"

  client = Anthropic::Client.new

  store = client.beta.memory_stores.create(
    name: "User Preferences",
    description: "Per-user preferences and project context."
  )
  puts store.id # memstore_01Hx...
  ```
</CodeGroup>

记忆存储的 `id`（`memstore_...`）是您在将存储附加到会话时需要传递的值。

### 预填充内容（可选）

在任何智能体运行之前，为存储预加载参考资料：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s "https://api.anthropic.com/v1/memory_stores/$store_id/memories" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" \
    -H "content-type: application/json" \
    -d '{"path": "/formatting_standards.md", "content": "All reports use GAAP formatting. Dates are ISO-8601..."}' > /dev/null
  ```

  ```bash CLI
  ant beta:memory-stores:memories create \
    --memory-store-id "$store_id" \
    --path "/formatting_standards.md" \
    --content "All reports use GAAP formatting. Dates are ISO-8601..." \
    > /dev/null
  ```

  ```python Python
  client.beta.memory_stores.memories.create(
      store.id,
      path="/formatting_standards.md",
      content="All reports use GAAP formatting. Dates are ISO-8601...",
  )
  ```

  ```typescript TypeScript
  await client.beta.memoryStores.memories.create(store.id, {
    path: "/formatting_standards.md",
    content: "All reports use GAAP formatting. Dates are ISO-8601..."
  });
  ```

  ```csharp C#
  await client.Beta.MemoryStores.Memories.Create(store.ID, new()
  {
      Path = "/formatting_standards.md",
      Content = "All reports use GAAP formatting. Dates are ISO-8601...",
  });
  ```

  ```go Go
  _, err = client.Beta.MemoryStores.Memories.New(ctx, store.ID, anthropic.BetaMemoryStoreMemoryNewParams{
  	Path:    "/formatting_standards.md",
  	Content: anthropic.String("All reports use GAAP formatting. Dates are ISO-8601..."),
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().memoryStores().memories().create(
      store.id(),
      MemoryCreateParams.builder()
          .path("/formatting_standards.md")
          .content("All reports use GAAP formatting. Dates are ISO-8601...")
          .build()
  );
  ```

  ```php PHP
  $client->beta->memoryStores->memories->create(
      $store->id,
      path: '/formatting_standards.md',
      content: 'All reports use GAAP formatting. Dates are ISO-8601...',
  );
  ```

  ```ruby Ruby
  client.beta.memory_stores.memories.create(
    store.id,
    path: "/formatting_standards.md",
    content: "All reports use GAAP formatting. Dates are ISO-8601..."
  )
  ```
</CodeGroup>

<Tip>
  存储中的单条记忆上限为 100 kB（约 25k 令牌）。一个存储最多可容纳 10,000 条记忆。请将记忆组织为许多小而专注的文件，而不是少数几个大文件。
</Tip>

## 将记忆存储附加到会话

记忆存储在[创建会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions#creating-a-session)时通过会话的 `resources[]` 数组附加。与文件资源不同，记忆存储只能在会话创建时附加；不支持在运行中的会话中添加或移除记忆存储。对于云端和[自托管环境](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)上的会话，附加记忆存储的方式相同；自托管环境仅接受 `memory_store` 资源。

可以选择包含 `instructions`，为智能体应如何使用此存储提供会话特定的指导。它会与存储的 `name` 和 `description` 一起展示给智能体，上限为 4,096 个字符。

您还可以配置 `access`。它默认为 `read_write`（在以下示例中显式展示），但也支持 `read_only`。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s https://api.anthropic.com/v1/sessions \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: managed-agents-2026-04-01" \
    -H "content-type: application/json" \
    --data @- <<EOF
  {
    "agent": "$agent_id",
    "environment_id": "$environment_id",
    "resources": [
      {
        "type": "memory_store",
        "memory_store_id": "$store_id",
        "access": "read_write",
        "instructions": "User preferences and project context. Check before starting any task."
      }
    ]
  }
  EOF
  ```

  ```bash CLI
  ant beta:sessions create <<YAML
  agent: $agent_id
  environment_id: $environment_id
  resources:
    - type: memory_store
      memory_store_id: $store_id
      access: read_write
      instructions: User preferences and project context. Check before starting any task.
  YAML
  ```

  ```python Python
  session = client.beta.sessions.create(
      agent=agent.id,
      environment_id=environment.id,
      resources=[
          {
              "type": "memory_store",
              "memory_store_id": store.id,
              "access": "read_write",
              "instructions": "User preferences and project context. Check before starting any task.",
          }
      ],
  )
  ```

  ```typescript TypeScript
  const session = await client.beta.sessions.create({
    agent: agent.id,
    environment_id: environment.id,
    resources: [
      {
        type: "memory_store",
        memory_store_id: store.id,
        access: "read_write",
        instructions: "User preferences and project context. Check before starting any task."
      }
    ]
  });
  ```

  ```csharp C#
  var session = await client.Beta.Sessions.Create(new()
  {
      Agent = agent.ID,
      EnvironmentID = environment.ID,
      Resources =
      [
          new BetaManagedAgentsMemoryStoreResourceParam
          {
              Type = "memory_store",
              MemoryStoreID = store.ID,
              Access = "read_write",
              Instructions = "User preferences and project context. Check before starting any task.",
          },
      ],
  });
  ```

  ```go Go
  session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
  	Agent: anthropic.BetaSessionNewParamsAgentUnion{
  		OfString: anthropic.String(agent.ID),
  	},
  	EnvironmentID: environment.ID,
  	Resources: []anthropic.BetaSessionNewParamsResourceUnion{{
  		OfMemoryStore: &anthropic.BetaManagedAgentsMemoryStoreResourceParam{
  			Type:          anthropic.BetaManagedAgentsMemoryStoreResourceParamTypeMemoryStore,
  			MemoryStoreID: store.ID,
  			Access:        anthropic.BetaManagedAgentsMemoryStoreResourceParamAccessReadWrite,
  			Instructions:  anthropic.String("User preferences and project context. Check before starting any task."),
  		},
  	}},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var session = client.beta().sessions().create(
      SessionCreateParams.builder()
          .agent(agent.id())
          .environmentId(environment.id())
          .addResource(
              BetaManagedAgentsMemoryStoreResourceParam.builder()
                  .type(BetaManagedAgentsMemoryStoreResourceParam.Type.MEMORY_STORE)
                  .memoryStoreId(store.id())
                  .access(BetaManagedAgentsMemoryStoreResourceParam.Access.READ_WRITE)
                  .instructions("User preferences and project context. Check before starting any task.")
                  .build()
          )
          .build()
  );
  ```

  ```php PHP
  $session = $client->beta->sessions->create(
      agent: $agent->id,
      environmentID: $environment->id,
      resources: [
          [
              'type' => 'memory_store',
              'memory_store_id' => $store->id,
              'access' => 'read_write',
              'instructions' => 'User preferences and project context. Check before starting any task.',
          ],
      ],
  );
  ```

  ```ruby Ruby
  session = client.beta.sessions.create(
    agent: agent.id,
    environment_id: environment.id,
    resources: [
      {
        type: "memory_store",
        memory_store_id: store.id,
        access: "read_write",
        instructions: "User preferences and project context. Check before starting any task."
      }
    ]
  )
  ```
</CodeGroup>

<Warning>
  记忆存储默认以 `read_write` 访问权限附加。如果智能体处理不受信任的输入（用户提供的提示、抓取的网页内容或第三方工具输出），一次成功的提示注入可能会将恶意内容写入存储。之后的会话会将该内容作为受信任的记忆读取。对于参考资料、共享查询数据以及智能体不需要修改的任何存储，请使用 `read_only`。
</Warning>

每个会话最多支持 **8 个记忆存储**。当记忆的不同部分具有不同的所有者或访问规则时，可以附加多个存储。常见原因包括：

* \*\*共享参考资料：\*\*一个只读存储附加到多个会话（标准、约定、领域知识），与每个会话自己的读写存储分开。
* \*\*映射到您的产品结构：\*\*每个终端用户、每个团队或每个项目一个存储，同时共享单一的智能体配置。
* \*\*不同的生命周期：\*\*一个比任何单个会话存续时间更长的存储，或者您希望按其自身计划归档的存储。

### 智能体如何访问记忆

每个附加的存储都会作为 `/mnt/memory/` 下的一个目录挂载到会话的沙箱中。目录名称是存储的显示名称经过清理后得到的文件系统安全的 slug（转为小写；连续的非字母数字字符变为单个连字符），因此名为"Demo Memory"的存储会挂载在 `/mnt/memory/demo-memory/`。确切路径会在会话的记忆存储资源的 `mount_path` 字段中返回；请从那里读取，而不要自行构造。智能体使用标准的[智能体工具集](https://platform.claude.com/docs/zh-CN/managed-agents/tools)读写存储。挂载路径下的写入会持久化回存储，并在共享该存储的会话之间保持同步；对 `/mnt/memory/` 下任何其他路径的写入都会失败，因为沙箱以只读方式挂载该父目录。每个挂载的简短描述（显示名称、挂载路径、访问模式、存储的 `description` 以及任何 `instructions`）会自动添加到系统提示中。

`access` 在文件系统层面强制执行：`read_only` 挂载会拒绝写入，而对 `read_write` 挂载的写入会生成归属于该会话的[记忆版本](https://platform.claude.com/docs/zh-CN/managed-agents/memory#audit-memory-changes)。

<Note>
  在[自托管沙箱](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#use-memory-stores)上，每个存储的目录是由 SDK 工作进程管理的本地副本，而不是实时挂载。工作进程会在工具调用之后将每个副本与其存储进行协调，每个同步间隔（默认 15 秒）最多一次，并在会话结束时再进行一次。智能体的 `write` 和 `edit` 工具只更改本地副本；工作进程会在下一次同步时上传这些更改，因此在自托管沙箱上运行的另一个会话只有在两个工作进程都完成同步后才能看到更改。在那里，`/mnt/memory/` 下存储目录之外的路径不是临时空间：工作进程的文件工具拒绝写入这些路径，并且 shell 命令写入那里的任何内容永远不会同步到存储。

  对于 `read_only` 存储，工作进程的 `write` 和 `edit` 工具会拒绝该目录下的更改，并且工作进程永远不会从中上传任何内容。要了解工作进程如何解决写入冲突，以及 `bash` 工具在只读存储的本地副本中仍然可以更改什么，请参阅[只读存储与冲突](https://platform.claude.com/docs/zh-CN/managed-agents/self-hosted-sandboxes#read-only-stores-and-conflicts)。
</Note>

智能体的读写操作会在[事件流](https://platform.claude.com/docs/zh-CN/managed-agents/events-and-streaming)中显示为普通的 `agent.tool_use` 和 `agent.tool_result` 事件，对应于触及该挂载的任何工具。

## 查看和编辑记忆

记忆存储可以直接通过 API 管理。可用于构建审核工作流、纠正错误记忆，或在任何会话运行之前预填充存储。

### 列出记忆

列出存储中的记忆。结果以稳定的、由服务器定义的顺序返回。

* `path_prefix` 将列表范围限定到一个目录。它必须以 `/` 结尾，并匹配完整的路径段，因此 `path_prefix=/notes/` 会返回 `/notes/todo.md`，但不会返回 `/notes-archive/todo.md`。
* `depth` 控制列表在 `path_prefix` 之下深入的层级：省略它（或传递 `0`）以列出整个子树，或传递 `1` 以仅列出直接子项。其他值会返回 `400` 错误。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s "https://api.anthropic.com/v1/memory_stores/$store_id/memories?path_prefix=/" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" | jq -r '.data[] | "\(.type)  \(.path)"'
  ```

  ```bash CLI
  ant beta:memory-stores:memories list \
    --memory-store-id "$store_id" \
    --path-prefix "/"
  ```

  ```python Python
  page = client.beta.memory_stores.memories.list(
      store.id,
      path_prefix="/",
  )
  for item in page.data:
      print(item.type, item.path)
  ```

  ```typescript TypeScript
  const page = await client.beta.memoryStores.memories.list(store.id, {
    path_prefix: "/"
  });
  for (const item of page.data) {
    console.log(item.type, item.path);
  }
  ```

  ```csharp C#
  var page = await client.Beta.MemoryStores.Memories.List(store.ID, new()
  {
      PathPrefix = "/",
  });
  await foreach (var item in page.Paginate())
  {
      var line = item.Match(m => $"memory  {m.Path}", p => $"memory_prefix  {p.Path}");
      Console.WriteLine(line);
  }
  ```

  ```go Go
  page, err := client.Beta.MemoryStores.Memories.List(ctx, store.ID, anthropic.BetaMemoryStoreMemoryListParams{
  	PathPrefix: anthropic.String("/"),
  })
  if err != nil {
  	panic(err)
  }
  for _, item := range page.Data {
  	fmt.Println(item.Type, item.Path)
  }
  ```

  ```java Java
  var page = client.beta().memoryStores().memories().list(
      store.id(),
      MemoryListParams.builder()
          .pathPrefix("/")
          .build()
  );
  for (var item : page.data()) {
      item.memory().ifPresent(m -> IO.println("memory  " + m.path()));
      item.memoryPrefix().ifPresent(p -> IO.println("memory_prefix  " + p.path()));
  }
  ```

  ```php PHP
  $page = $client->beta->memoryStores->memories->list(
      $store->id,
      pathPrefix: '/',
  );
  foreach ($page->data as $item) {
      echo "{$item->type}  {$item->path}\n";
  }
  ```

  ```ruby Ruby
  page = client.beta.memory_stores.memories.list(
    store.id,
    path_prefix: "/"
  )
  page.data.each do |entry|
    puts "#{entry.type}  #{entry.path}"
  end
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[列出记忆参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/memories/list)。

### 读取记忆

获取单条记忆会返回完整内容。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s "https://api.anthropic.com/v1/memory_stores/$store_id/memories/$mem_id" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" | jq -r '.content'
  ```

  ```bash CLI
  ant beta:memory-stores:memories retrieve \
    --memory-store-id "$store_id" \
    --memory-id "$mem_id"
  ```

  ```python Python
  retrieved = client.beta.memory_stores.memories.retrieve(
      mem.id,
      memory_store_id=store.id,
  )
  print(retrieved.content)
  ```

  ```typescript TypeScript
  const retrieved = await client.beta.memoryStores.memories.retrieve(mem.id, {
    memory_store_id: store.id
  });
  console.log(retrieved.content);
  ```

  ```csharp C#
  var retrieved = await client.Beta.MemoryStores.Memories.Retrieve(mem.ID, new()
  {
      MemoryStoreID = store.ID,
  });
  Console.WriteLine(retrieved.Content);
  ```

  ```go Go
  retrieved, err := client.Beta.MemoryStores.Memories.Get(ctx, mem.ID, anthropic.BetaMemoryStoreMemoryGetParams{
  	MemoryStoreID: store.ID,
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(retrieved.Content)
  ```

  ```java Java
  var retrieved = client.beta().memoryStores().memories().retrieve(
      mem.id(),
      MemoryRetrieveParams.builder().memoryStoreId(store.id()).build()
  );
  IO.println(retrieved.content().orElseThrow());
  ```

  ```php PHP
  $retrieved = $client->beta->memoryStores->memories->retrieve($mem->id, memoryStoreID: $store->id);
  echo "{$retrieved->content}\n";
  ```

  ```ruby Ruby
  retrieved = client.beta.memory_stores.memories.retrieve(
    mem.id,
    memory_store_id: store.id
  )
  puts retrieved.content
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[检索记忆参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/memories/retrieve)。

### 创建记忆

`memories.create` 在给定的 `path` 处创建一条记忆。创建操作不会覆盖；要更改现有记忆，请使用 [`memories.update`](https://platform.claude.com/docs/zh-CN/managed-agents/memory#update-a-memory)。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  mem=$(curl -s "https://api.anthropic.com/v1/memory_stores/$store_id/memories" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" \
    -H "content-type: application/json" \
    -d '{"path": "/preferences/formatting.md", "content": "Always use tabs, not spaces."}')
  mem_id=$(jq -r '.id' <<< "$mem")
  mem_sha=$(jq -r '.content_sha256' <<< "$mem")
  ```

  ```bash CLI
  mem=$(ant beta:memory-stores:memories create \
    --memory-store-id "$store_id" \
    --path "/preferences/formatting.md" \
    --content "Always use tabs, not spaces." \
    --format json)
  mem_id=$(jq -r '.id' <<< "$mem")
  mem_sha=$(jq -r '.content_sha256' <<< "$mem")
  ```

  ```python Python
  mem = client.beta.memory_stores.memories.create(
      store.id,
      path="/preferences/formatting.md",
      content="Always use tabs, not spaces.",
  )
  ```

  ```typescript TypeScript
  const mem = await client.beta.memoryStores.memories.create(store.id, {
    path: "/preferences/formatting.md",
    content: "Always use tabs, not spaces."
  });
  ```

  ```csharp C#
  var mem = await client.Beta.MemoryStores.Memories.Create(store.ID, new()
  {
      Path = "/preferences/formatting.md",
      Content = "Always use tabs, not spaces.",
  });
  ```

  ```go Go
  mem, err := client.Beta.MemoryStores.Memories.New(ctx, store.ID, anthropic.BetaMemoryStoreMemoryNewParams{
  	Path:    "/preferences/formatting.md",
  	Content: anthropic.String("Always use tabs, not spaces."),
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  var mem = client.beta().memoryStores().memories().create(
      store.id(),
      MemoryCreateParams.builder()
          .path("/preferences/formatting.md")
          .content("Always use tabs, not spaces.")
          .build()
  );
  ```

  ```php PHP
  $mem = $client->beta->memoryStores->memories->create(
      $store->id,
      path: '/preferences/formatting.md',
      content: 'Always use tabs, not spaces.',
  );
  ```

  ```ruby Ruby
  mem = client.beta.memory_stores.memories.create(
    store.id,
    path: "/preferences/formatting.md",
    content: "Always use tabs, not spaces."
  )
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[创建记忆参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/memories/create)。

### 更新记忆

`memories.update` 通过 ID 修改现有记忆。您可以更改 `content`、`path`（重命名）或两者。以下示例将一条记忆重命名到归档路径：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s -X POST "https://api.anthropic.com/v1/memory_stores/$store_id/memories/$mem_id" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" \
    -H "content-type: application/json" \
    -d '{"path": "/archive/2026_q1_formatting.md"}' > /dev/null
  ```

  ```bash CLI
  ant beta:memory-stores:memories update \
    --memory-store-id "$store_id" \
    --memory-id "$mem_id" \
    --path "/archive/2026_q1_formatting.md" \
    > /dev/null
  ```

  ```python Python
  client.beta.memory_stores.memories.update(
      mem.id,
      memory_store_id=store.id,
      path="/archive/2026_q1_formatting.md",
  )
  ```

  ```typescript TypeScript
  await client.beta.memoryStores.memories.update(mem.id, {
    memory_store_id: store.id,
    path: "/archive/2026_q1_formatting.md"
  });
  ```

  ```csharp C#
  await client.Beta.MemoryStores.Memories.Update(mem.ID, new()
  {
      MemoryStoreID = store.ID,
      Path = "/archive/2026_q1_formatting.md",
  });
  ```

  ```go Go
  _, err = client.Beta.MemoryStores.Memories.Update(ctx, mem.ID, anthropic.BetaMemoryStoreMemoryUpdateParams{
  	MemoryStoreID: store.ID,
  	Path:          anthropic.String("/archive/2026_q1_formatting.md"),
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().memoryStores().memories().update(
      mem.id(),
      MemoryUpdateParams.builder()
          .memoryStoreId(store.id())
          .path("/archive/2026_q1_formatting.md")
          .build()
  );
  ```

  ```php PHP
  $client->beta->memoryStores->memories->update(
      $mem->id,
      memoryStoreID: $store->id,
      path: '/archive/2026_q1_formatting.md',
  );
  ```

  ```ruby Ruby
  client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id: store.id,
    path: "/archive/2026_q1_formatting.md"
  )
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[更新记忆参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/memories/update)。

#### 安全的内容编辑（乐观并发）

为避免覆盖并发写入，请传递 `content_sha256` 前置条件。只有当存储的内容哈希仍与您读取时的哈希匹配时，更新才会生效；如果不匹配，请重新读取记忆并针对最新状态重试。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s -X POST "https://api.anthropic.com/v1/memory_stores/$store_id/memories/$mem_id" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" \
    -H "content-type: application/json" \
    --data @- > /dev/null <<EOF
  {
    "content": "CORRECTED: Always use 2-space indentation.",
    "precondition": {"type": "content_sha256", "content_sha256": "$mem_sha"}
  }
  EOF
  ```

  ```bash CLI
  ant beta:memory-stores:memories update \
    --memory-store-id "$store_id" \
    --memory-id "$mem_id" \
    --content "CORRECTED: Always use 2-space indentation." \
    --precondition "{type: content_sha256, content_sha256: $mem_sha}" \
    > /dev/null
  ```

  ```python Python
  client.beta.memory_stores.memories.update(
      memory_id=mem.id,
      memory_store_id=store.id,
      content="CORRECTED: Always use 2-space indentation.",
      precondition={"type": "content_sha256", "content_sha256": mem.content_sha256},
  )
  ```

  ```typescript TypeScript
  await client.beta.memoryStores.memories.update(mem.id, {
    memory_store_id: store.id,
    content: "CORRECTED: Always use 2-space indentation.",
    precondition: { type: "content_sha256", content_sha256: mem.content_sha256 }
  });
  ```

  ```csharp C#
  await client.Beta.MemoryStores.Memories.Update(mem.ID, new()
  {
      MemoryStoreID = store.ID,
      Content = "CORRECTED: Always use 2-space indentation.",
      Precondition = new BetaManagedAgentsPrecondition
      {
          Type = "content_sha256",
          ContentSha256 = mem.ContentSha256,
      },
  });
  ```

  ```go Go
  _, err = client.Beta.MemoryStores.Memories.Update(ctx, mem.ID, anthropic.BetaMemoryStoreMemoryUpdateParams{
  	MemoryStoreID: store.ID,
  	Content:       anthropic.String("CORRECTED: Always use 2-space indentation."),
  	Precondition: anthropic.BetaManagedAgentsPreconditionParam{
  		Type:          anthropic.BetaManagedAgentsPreconditionTypeContentSha256,
  		ContentSha256: anthropic.String(mem.ContentSha256),
  	},
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().memoryStores().memories().update(
      mem.id(),
      MemoryUpdateParams.builder()
          .memoryStoreId(store.id())
          .content("CORRECTED: Always use 2-space indentation.")
          .precondition(
              BetaManagedAgentsPrecondition.builder()
                  .type(BetaManagedAgentsPrecondition.Type.CONTENT_SHA256)
                  .contentSha256(mem.contentSha256())
                  .build()
          )
          .build()
  );
  ```

  ```php PHP
  $client->beta->memoryStores->memories->update(
      $mem->id,
      memoryStoreID: $store->id,
      content: 'CORRECTED: Always use 2-space indentation.',
      precondition: ['type' => 'content_sha256', 'content_sha256' => $mem->contentSha256],
  );
  ```

  ```ruby Ruby
  client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id: store.id,
    content: "CORRECTED: Always use 2-space indentation.",
    precondition: {type: "content_sha256", content_sha256: mem.content_sha256}
  )
  ```
</CodeGroup>

### 删除记忆

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s -X DELETE "https://api.anthropic.com/v1/memory_stores/$store_id/memories/$mem_id" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" > /dev/null
  ```

  ```bash CLI
  ant beta:memory-stores:memories delete \
    --memory-store-id "$store_id" \
    --memory-id "$mem_id" \
    > /dev/null
  ```

  ```python Python
  client.beta.memory_stores.memories.delete(
      mem.id,
      memory_store_id=store.id,
  )
  ```

  ```typescript TypeScript
  await client.beta.memoryStores.memories.delete(mem.id, {
    memory_store_id: store.id
  });
  ```

  ```csharp C#
  await client.Beta.MemoryStores.Memories.Delete(mem.ID, new()
  {
      MemoryStoreID = store.ID,
  });
  ```

  ```go Go
  _, err = client.Beta.MemoryStores.Memories.Delete(ctx, mem.ID, anthropic.BetaMemoryStoreMemoryDeleteParams{
  	MemoryStoreID: store.ID,
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().memoryStores().memories().delete(
      mem.id(),
      MemoryDeleteParams.builder().memoryStoreId(store.id()).build()
  );
  ```

  ```php PHP
  $client->beta->memoryStores->memories->delete($mem->id, memoryStoreID: $store->id);
  ```

  ```ruby Ruby
  client.beta.memory_stores.memories.delete(
    mem.id,
    memory_store_id: store.id
  )
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[删除记忆参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/memories/delete)。

## 审计记忆更改

对记忆的每次变更都会创建一个不可变的**记忆版本**（`memver_...`）。使用版本端点可以审计谁在何时更改了什么、检查或恢复先前的快照，以及通过脱敏（redact）从历史记录中清除敏感内容。

版本属于存储（而非单条记忆），并且在记忆本身被删除时不会被删除，因此审计跟踪也涵盖已删除的记忆，但受下文所述的保留策略约束。版本在写入后保留 30 天；不过，活动记忆的最近版本无论存在多久都会始终保留，因此不常更改的记忆可能会保留超过 30 天的历史记录。实时的 `memories.retrieve` 调用始终返回最新版本；版本端点则为您提供保留的历史记录。

没有专门的恢复端点；要回滚，请检索您想要的版本，并使用 `memories.update` 将其 `content` 写回（如果父记忆已被删除，则使用 `memories.create`，前提是您想要的版本仍被保留）。

过去的记忆版本可能会在 30 天后被删除。要更长时间地保留记忆历史，请通过 API 导出版本。

### 列出版本

列出存储的版本历史，最新的在前。以下示例筛选出单条记忆的历史：

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  versions=$(curl -s "https://api.anthropic.com/v1/memory_stores/$store_id/memory_versions?memory_id=$mem_id" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22")
  jq -r '.data[] | "\(.id): \(.operation)"' <<< "$versions"
  version_id=$(jq -r '.data[1].id' <<< "$versions")
  ```

  ```bash CLI
  versions=$(ant beta:memory-stores:memory-versions list \
    --memory-store-id "$store_id" \
    --memory-id "$mem_id" \
    --format json)
  # `list --format json` 为每个条目输出一个 JSON 对象。
  jq -r '"\(.id): \(.operation)"' <<< "$versions"
  version_id=$(jq -rs '.[1].id' <<< "$versions")
  ```

  ```python Python
  versions = client.beta.memory_stores.memory_versions.list(
      store.id,
      memory_id=mem.id,
  )
  for version in versions:
      print(f"{version.id}: {version.operation}")

  version_id = versions.data[1].id
  ```

  ```typescript TypeScript
  const versions = await client.beta.memoryStores.memoryVersions.list(store.id, {
    memory_id: mem.id
  });
  for await (const v of versions) {
    console.log(`${v.id}: ${v.operation}`);
  }

  const versionId = versions.data[1].id;
  ```

  ```csharp C#
  var versions = await client.Beta.MemoryStores.MemoryVersions.List(store.ID, new()
  {
      MemoryID = mem.ID,
  });
  var versionIds = new List<string>();
  await foreach (var v in versions.Paginate())
  {
      Console.WriteLine($"{v.ID}: {v.Operation.Raw()}");
      versionIds.Add(v.ID);
  }

  var versionId = versionIds[1];
  ```

  ```go Go
  versions := client.Beta.MemoryStores.MemoryVersions.ListAutoPaging(ctx, store.ID, anthropic.BetaMemoryStoreMemoryVersionListParams{
  	MemoryID: anthropic.String(mem.ID),
  })
  for versions.Next() {
  	v := versions.Current()
  	fmt.Printf("%s: %s\n", v.ID, v.Operation)
  }
  if err := versions.Err(); err != nil {
  	panic(err)
  }

  vpage, err := client.Beta.MemoryStores.MemoryVersions.List(ctx, store.ID, anthropic.BetaMemoryStoreMemoryVersionListParams{
  	MemoryID: anthropic.String(mem.ID),
  })
  if err != nil {
  	panic(err)
  }
  versionID := vpage.Data[1].ID
  ```

  ```java Java
  var versions = client.beta().memoryStores().memoryVersions().list(
      store.id(),
      MemoryVersionListParams.builder().memoryId(mem.id()).build()
  );
  for (var v : versions.autoPager()) {
      IO.println(v.id() + ": " + v.operation());
  }

  var versionId = versions.data().get(1).id();
  ```

  ```php PHP
  $versions = $client->beta->memoryStores->memoryVersions->list(
      $store->id,
      memoryID: $mem->id,
  );
  foreach ($versions->pagingEachItem() as $v) {
      echo "{$v->id}: {$v->operation}\n";
  }

  $versionId = $versions->data[1]->id;
  ```

  ```ruby Ruby
  versions = client.beta.memory_stores.memory_versions.list(
    store.id,
    memory_id: mem.id
  )
  versions.auto_paging_each do |version|
    puts "#{version.id}: #{version.operation}"
  end

  version_id = versions.data[1].id
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[列出记忆版本参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/memory_versions/list)。

### 检索版本

获取单个版本会返回与列表响应相同的字段，外加完整的 `content` 正文。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s "https://api.anthropic.com/v1/memory_stores/$store_id/memory_versions/$version_id" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22"
  ```

  ```bash CLI
  ant beta:memory-stores:memory-versions retrieve \
    --memory-store-id "$store_id" \
    --memory-version-id "$version_id"
  ```

  ```python Python
  version = client.beta.memory_stores.memory_versions.retrieve(
      version_id,
      memory_store_id=store.id,
  )
  print(version.content)
  ```

  ```typescript TypeScript
  const version = await client.beta.memoryStores.memoryVersions.retrieve(versionId, {
    memory_store_id: store.id
  });
  console.log(version.content);
  ```

  ```csharp C#
  var version = await client.Beta.MemoryStores.MemoryVersions.Retrieve(versionId, new()
  {
      MemoryStoreID = store.ID,
  });
  Console.WriteLine(version.Content);
  ```

  ```go Go
  version, err := client.Beta.MemoryStores.MemoryVersions.Get(ctx, versionID, anthropic.BetaMemoryStoreMemoryVersionGetParams{
  	MemoryStoreID: store.ID,
  })
  if err != nil {
  	panic(err)
  }
  fmt.Println(version.Content)
  ```

  ```java Java
  var version = client.beta().memoryStores().memoryVersions().retrieve(
      versionId,
      MemoryVersionRetrieveParams.builder().memoryStoreId(store.id()).build()
  );
  IO.println(version.content().orElseThrow());
  ```

  ```php PHP
  $version = $client->beta->memoryStores->memoryVersions->retrieve(
      $versionId,
      memoryStoreID: $store->id,
  );
  echo "{$version->content}\n";
  ```

  ```ruby Ruby
  version = client.beta.memory_stores.memory_versions.retrieve(
    version_id,
    memory_store_id: store.id
  )
  puts version.content
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[检索记忆版本参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/memory_versions/retrieve)。

### 脱敏版本

脱敏会从历史版本中清除内容，同时保留审计跟踪（谁在何时做了什么）。可将其用于合规工作流，例如移除泄露的密钥、个人身份信息（PII）或处理用户删除请求。

作为活动记忆当前头部的版本无法被脱敏。请先写入一个新版本（或删除该记忆），然后再对旧版本进行脱敏。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s -X POST "https://api.anthropic.com/v1/memory_stores/$store_id/memory_versions/$version_id/redact" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" \
    -H "content-type: application/json" \
    -d '{}'
  ```

  ```bash CLI
  ant beta:memory-stores:memory-versions redact \
    --memory-store-id "$store_id" \
    --memory-version-id "$version_id"
  ```

  ```python Python
  client.beta.memory_stores.memory_versions.redact(
      version_id,
      memory_store_id=store.id,
  )
  ```

  ```typescript TypeScript
  await client.beta.memoryStores.memoryVersions.redact(versionId, {
    memory_store_id: store.id
  });
  ```

  ```csharp C#
  await client.Beta.MemoryStores.MemoryVersions.Redact(versionId, new()
  {
      MemoryStoreID = store.ID,
  });
  ```

  ```go Go
  _, err = client.Beta.MemoryStores.MemoryVersions.Redact(ctx, versionID, anthropic.BetaMemoryStoreMemoryVersionRedactParams{
  	MemoryStoreID: store.ID,
  })
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().memoryStores().memoryVersions().redact(
      versionId,
      MemoryVersionRedactParams.builder().memoryStoreId(store.id()).build()
  );
  ```

  ```php PHP
  $client->beta->memoryStores->memoryVersions->redact(
      $versionId,
      memoryStoreID: $store->id,
  );
  ```

  ```ruby Ruby
  client.beta.memory_stores.memory_versions.redact(
    version_id,
    memory_store_id: store.id
  )
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[脱敏记忆版本参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/memory_versions/redact)。

## 管理记忆存储

除了 [`create`](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/create) 之外，记忆存储还支持 [`retrieve`](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/retrieve)、[`update`](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/update)、[`list`](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/list)、[`archive`](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/archive) 和 [`delete`](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/delete)。

### 列出存储

列出工作区中的存储。默认排除已归档的存储；传递 `include_archived: true` 可将其包含在内。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s "https://api.anthropic.com/v1/memory_stores?include_archived=true" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" | jq '.data[] | {id, name, archived_at}'
  ```

  ```bash CLI
  ant beta:memory-stores list --include-archived
  ```

  ```python Python
  for memory_store in client.beta.memory_stores.list(include_archived=True):
      print(memory_store.id, memory_store.name, memory_store.archived_at)
  ```

  ```typescript TypeScript
  for await (const s of client.beta.memoryStores.list({ include_archived: true })) {
    console.log(s.id, s.name, s.archived_at);
  }
  ```

  ```csharp C#
  var stores = await client.Beta.MemoryStores.List(new() { IncludeArchived = true });
  await foreach (var s in stores.Paginate())
  {
      Console.WriteLine($"{s.ID} {s.Name} {s.ArchivedAt}");
  }
  ```

  ```go Go
  stores := client.Beta.MemoryStores.ListAutoPaging(ctx, anthropic.BetaMemoryStoreListParams{
  	IncludeArchived: anthropic.Bool(true),
  })
  for stores.Next() {
  	s := stores.Current()
  	fmt.Println(s.ID, s.Name, s.ArchivedAt)
  }
  if err := stores.Err(); err != nil {
  	panic(err)
  }
  ```

  ```java Java
  for (var s : client.beta().memoryStores().list(
      MemoryStoreListParams.builder().includeArchived(true).build()
  ).autoPager()) {
      IO.println(s.id() + " " + s.name() + " " + s.archivedAt());
  }
  ```

  ```php PHP
  foreach ($client->beta->memoryStores->list(includeArchived: true)->pagingEachItem() as $s) {
      // archivedAt 仅在已归档的存储上设置。
      $archivedAt = isset($s->archivedAt) ? $s->archivedAt->format(DATE_ATOM) : '';
      echo "{$s->id} {$s->name} {$archivedAt}\n";
  }
  ```

  ```ruby Ruby
  client.beta.memory_stores.list(include_archived: true).auto_paging_each do |memory_store|
    puts "#{memory_store.id} #{memory_store.name} #{memory_store.archived_at}"
  end
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[列出记忆存储参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/list)。

### 归档存储

归档会使存储变为只读，并阻止其被附加到新会话。归档是单向的；没有取消归档操作。

<CodeGroup defaultLanguage="CLI">
  ```bash cURL
  curl -s -X POST "https://api.anthropic.com/v1/memory_stores/$store_id/archive" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "anthropic-beta: agent-memory-2026-07-22" > /dev/null
  ```

  ```bash CLI
  ant beta:memory-stores archive --memory-store-id "$store_id"
  ```

  ```python Python
  client.beta.memory_stores.archive(store.id)
  ```

  ```typescript TypeScript
  await client.beta.memoryStores.archive(store.id);
  ```

  ```csharp C#
  await client.Beta.MemoryStores.Archive(store.ID);
  ```

  ```go Go
  _, err = client.Beta.MemoryStores.Archive(ctx, store.ID, anthropic.BetaMemoryStoreArchiveParams{})
  if err != nil {
  	panic(err)
  }
  ```

  ```java Java
  client.beta().memoryStores().archive(store.id());
  ```

  ```php PHP
  $client->beta->memoryStores->archive($store->id);
  ```

  ```ruby Ruby
  client.beta.memory_stores.archive(store.id)
  ```
</CodeGroup>

有关完整参数和响应模式，请参阅[归档记忆存储参考](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/archive)。

要永久移除存储及其所有记忆和版本，请使用 [`memory_stores.delete`](https://platform.claude.com/docs/zh-CN/api/beta/memory_stores/delete)。

## 记忆管理最佳实践

当存储达到 10,000 条记忆的上限时，对新记忆的写入会失败：包括直接的 `memories.create` 调用以及智能体对未映射路径的文件写入。现有记忆仍然可读可编辑。以下实践可帮助您远低于上限，并在达到上限时从容恢复。

* \*\*使用专注的存储。\*\*与其使用一个大型通用存储，不如使用更小的专用存储：每个用户一个、共享领域知识一个、项目特定上下文一个。每个存储都有自己的 10,000 条记忆上限，因此保持存储范围明确可以降低任何单个存储被填满的可能性。

* \*\*在存储填满之前进行精简或清理。\*\*使用 `memories.delete` 删除过时或冗余的记忆。您还可以运行[梦境会话](https://platform.claude.com/docs/zh-CN/managed-agents/dreams)，它会将碎片化的内容整合到一个单独的新输出存储中，而不是修改原始存储。将您的会话切换到该输出存储，然后归档或删除原始存储。

* \*\*在合适的时候附加新存储。\*\*如果某个存储已经超出其有用范围，请为新内容附加一个全新的存储，并以 `read_only` 访问权限附加原始存储。智能体可以从两者读取，同时只写入新存储。

* \*\*在适当的情况下限制写入权限。\*\*仅读取共享参考资料的会话不需要 `read_write`。将写入权限限定在实际添加新记忆的会话上，可以更容易地追踪增长的来源。
