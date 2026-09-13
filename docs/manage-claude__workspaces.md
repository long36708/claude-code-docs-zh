---
title: 工作区
url: https://platform.claude.com/docs/zh-CN/manage-claude/workspaces
description: 使用工作区组织 API 密钥、管理团队访问权限并控制成本。
---

"Workspaces"（工作区）提供了一种在组织内组织 API 使用情况的方式。使用工作区可以将不同的项目、环境或团队分隔开来，同时保持集中的计费和管理。

## 工作区的工作原理

每个组织都有一个 **Default Workspace**（默认工作区），它无法被重命名、归档或删除。当您创建其他工作区时，可以为每个工作区分配成员、服务账户、API 密钥和资源限制。

主要特征：

* **工作区标识符**使用 `wrkspc_` 前缀（例如 `wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ`）
* 默认情况下每个组织**最多 100 个工作区**（已归档的工作区不计入）；如果您需要更多，请联系您的账户团队
* **Default Workspace** 与其他任何工作区一样拥有 `wrkspc_` ID（在 [`anthropic-workspace-id` 响应头](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#identify-the-workspace-behind-an-api-response)中返回，并被 [Get Workspace](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/retrieve) 接受），但它不会出现在 [List Workspaces](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/list) 的结果中，并且 API 密钥、使用报告和成本报告中其 `workspace_id` 显示为 `null`，全工作区 API 密钥也是如此（API 密钥的 `scope` 字段可以区分它们；对于绑定到 Default Workspace 的密钥，该字段携带真实 ID）
* **API 密钥**可以限定到单个工作区。在这种情况下，它们只能访问该工作区内的资源。某些 API 密钥可以被授予跨多个工作区的权限，并通过提供[工作区 ID 请求头](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#select-a-workspace)来访问该工作区内的资源

### Claude Code 工作区

当您组织中的成员首次使用其 Claude Console 账户登录 [Claude Code](https://code.claude.com/docs/en/overview) 时，Anthropic 会自动在组织中创建一个 **Claude Code** 工作区，并将该成员添加到其中。之后每一位登录 Claude Code 的成员都会以同样的方式被添加。

Claude Code 工作区将 Claude Code 流量与您的其他 API 工作负载分隔开来：

* Claude Code 在登录时会在此工作区中为每个用户生成一个 API 密钥。您无法从 Console 手动在其中创建密钥。
* 与工作区密钥不同，如果 Claude Code 密钥的所有者被从工作区或组织中移除，该密钥将停止工作。
* Claude Code 的使用量单独进行速率限制，管理员可以在[设置 > 工作区](https://platform.claude.com/settings/workspaces)下限制其占用组织限额的份额。
* 它是唯一支持按用户设置每月支出限额的工作区。

<Warning>
  归档 Claude Code 工作区会禁用整个组织通过 Console 计费登录 Claude Code 的功能。
</Warning>

## 工作区角色和权限

成员可以在每个工作区中拥有不同的角色，从而实现细粒度的访问控制。

| 角色                          | 权限                                   |
| --------------------------- | ------------------------------------ |
| Workspace User              | 仅可使用 playground                      |
| Workspace Limited Developer | 创建和管理 API 密钥，使用 API。无法访问会话追踪视图或下载文件。 |
| Workspace Developer         | 创建和管理 API 密钥，使用 API                  |
| Workspace Admin             | 完全控制工作区设置和成员                         |
| Workspace Billing           | 查看工作区计费信息（从组织计费角色继承）                 |

### 角色继承

* **组织管理员**自动获得所有工作区的 Workspace Admin 访问权限
* **组织计费成员**自动获得所有工作区的 Workspace Billing 访问权限
* **组织用户和开发者**必须被显式添加到每个工作区
* **服务账户**可从[设置 → 服务账户](https://platform.claude.com/settings/service-accounts)中的服务账户页面或从工作区的**服务账户**选项卡添加到工作区

<Note>
  Workspace Billing 角色无法手动分配。它是通过拥有组织计费角色继承而来的。
</Note>

## 管理工作区

<Note>
  只有组织管理员可以创建工作区。组织用户和开发者必须由管理员添加到工作区。
</Note>

### 使用 Console

在 [Claude Console](https://platform.claude.com/settings/workspaces) 中创建和管理工作区。

#### 创建工作区

<Steps>
  <Step title="打开工作区设置">
    在 Claude Console 中，前往**设置 > 工作区**。
  </Step>

  <Step title="创建工作区">
    点击**创建工作区**。
  </Step>

  <Step title="配置工作区">
    输入工作区名称并选择一种颜色以便于视觉识别。
  </Step>

  <Step title="创建该工作区">
    点击**创建**以完成。
  </Step>
</Steps>

<Tip>
  要在 Console 中切换工作区，请使用左上角的**工作区**选择器。
</Tip>

#### 编辑工作区详情

要修改工作区的名称或颜色：

1. 从列表中选择工作区。
2. 点击省略号菜单（**...**）并选择**编辑详情**。
3. 更新名称或颜色并保存更改。

<Note>
  Default Workspace 无法被重命名或删除。
</Note>

#### 向工作区添加成员

1. 导航到工作区的**成员**选项卡。
2. 点击**添加到工作区**。
3. 选择一位组织成员并为其分配[工作区角色](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#workspace-roles-and-permissions)。
4. 确认添加。

要移除成员，请点击其姓名旁边的垃圾桶图标。

<Note>
  组织管理员和计费成员在担任这些组织角色期间无法从工作区中移除。
</Note>

#### 设置工作区限制

每个工作区的设置将这些限制分布在两个选项卡中：

* **速率限制：**在**速率限制**选项卡上，按模型层级设置每分钟请求数、输入令牌数或输出令牌数的限制
* **支出限制：**在**支出限制**选项卡上，限制每月支出并配置在支出达到特定阈值时的提醒

#### 归档工作区

要归档工作区，请点击省略号菜单（**...**）并选择**归档**。归档操作：

* 保留历史数据以供报告
* 停用工作区并归档为其创建的每个 API 密钥
* 无法撤销

<Warning>
  归档工作区会在几秒钟内归档为该工作区创建的每个 API 密钥（它们在 Admin API 中仍以已归档状态列出），并且多工作区密钥将无法再在其中操作。此操作无法撤销。如果您归档 [Claude Code 工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#claude-code-workspace)，您组织的成员将无法再通过 Console 计费登录 Claude Code。
</Warning>

### 使用 Admin API

使用 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 以编程方式管理工作区。

<Note>
  Admin API 端点接受 [Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)、`org:admin` OAuth 令牌，或未限定到特定工作区的个人或服务账户密钥。工作区密钥在此处无效。请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#authentication)。
</Note>

以下 SDK 和 CLI 示例构造默认客户端，该客户端从 `ANTHROPIC_API_KEY` 环境变量读取 Admin API 密钥；SDK 在 `client.beta.organization.workspaces` 下公开这些端点。SDK 的列表方法会按需获取后续页面，因此 `limit` 设置的是页面大小；PHP、Ruby 和 curl 示例返回一页。

创建工作区：

<CodeGroup>
  ```bash cURL
  curl -X POST "https://api.anthropic.com/v1/organizations/workspaces" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{"name": "Production"}'
  ```

  ```bash CLI
  ant beta:organization:workspaces create --name Production
  ```

  ```python Python
  client = anthropic.Anthropic()

  workspace = client.beta.organization.workspaces.create(name="Production")

  print(f"id: {workspace.id}")
  print(f"name: {workspace.name}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const workspace = await client.beta.organization.workspaces.create({ name: "Production" });

  console.log(`id: ${workspace.id}`);
  console.log(`name: ${workspace.name}`);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var workspace = await client.Beta.Organization.Workspaces.Create(new()
  {
      Name = "Production"
  });

  Console.WriteLine($"id: {workspace.ID}");
  Console.WriteLine($"name: {workspace.Name}");
  ```

  ```go Go
  client := anthropic.NewClient()

  workspace, err := client.Beta.Organization.Workspaces.New(context.Background(), anthropic.BetaOrganizationWorkspaceNewParams{
  	Name: "Production",
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", workspace.ID)
  fmt.Printf("name: %s\n", workspace.Name)
  ```

  ```java Java
  import com.anthropic.models.beta.organization.workspaces.WorkspaceCreateParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = WorkspaceCreateParams.builder()
          .name("Production")
          .build();
      var workspace = client.beta().organization().workspaces().create(params);

      IO.println("id: " + workspace.id());
      IO.println("name: " + workspace.name());
  }
  ```

  ```php PHP
  $client = new Client();

  $workspace = $client->beta->organization->workspaces->create(
      name: 'Production',
  );

  echo "id: {$workspace->id}\n";
  echo "name: {$workspace->name}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  workspace = client.beta.organization.workspaces.create(name: "Production")

  puts "id: #{workspace.id}"
  puts "name: #{workspace.name}"
  ```
</CodeGroup>

列出工作区：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/workspaces?limit=10&include_archived=false" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:workspaces list --limit 10 --include-archived=false
  ```

  ```python Python
  client = anthropic.Anthropic()

  workspaces = client.beta.organization.workspaces.list(limit=10, include_archived=False)

  for workspace in workspaces:
      print(f"{workspace.id}: {workspace.name}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const workspaces = await client.beta.organization.workspaces.list({
    limit: 10,
    include_archived: false
  });

  for await (const workspace of workspaces) {
    console.log(`${workspace.id}: ${workspace.name}`);
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var workspaces = await client.Beta.Organization.Workspaces.List(new()
  {
      Limit = 10,
      IncludeArchived = false
  });

  await foreach (var workspace in workspaces.Paginate())
  {
      Console.WriteLine($"{workspace.ID}: {workspace.Name}");
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  workspaces := client.Beta.Organization.Workspaces.ListAutoPaging(context.Background(), anthropic.BetaOrganizationWorkspaceListParams{
  	Limit:           anthropic.Int(10),
  	IncludeArchived: anthropic.Bool(false),
  })

  for workspaces.Next() {
  	workspace := workspaces.Current()
  	fmt.Printf("%s: %s\n", workspace.ID, workspace.Name)
  }
  if err := workspaces.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.beta.organization.workspaces.WorkspaceListParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = WorkspaceListParams.builder()
          .limit(10)
          .includeArchived(false)
          .build();
      var workspaces = client.beta().organization().workspaces().list(params);

      for (var workspace : workspaces.autoPager()) {
          IO.println(workspace.id() + ": " + workspace.name());
      }
  }
  ```

  ```php PHP
  $client = new Client();

  $workspaces = $client->beta->organization->workspaces->list(
      limit: 10,
      includeArchived: false,
  );

  foreach ($workspaces->getItems() as $workspace) {
      echo "{$workspace->id}: {$workspace->name}\n";
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  workspaces = client.beta.organization.workspaces.list(limit: 10, include_archived: false)

  workspaces.data.each do |workspace|
    puts "#{workspace.id}: #{workspace.name}"
  end
  ```
</CodeGroup>

归档工作区：

<CodeGroup>
  ```bash cURL
  curl -X POST "https://api.anthropic.com/v1/organizations/workspaces/wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ/archive" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:workspaces archive --workspace-id wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ
  ```

  ```python Python
  client = anthropic.Anthropic()

  workspace = client.beta.organization.workspaces.archive(
      "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  )

  print(f"id: {workspace.id}")
  print(f"archived_at: {workspace.archived_at}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const workspace = await client.beta.organization.workspaces.archive(
    "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  );

  console.log(`id: ${workspace.id}`);
  console.log(`archived_at: ${workspace.archived_at}`);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var workspace = await client.Beta.Organization.Workspaces.Archive(
      "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  );

  Console.WriteLine($"id: {workspace.ID}");
  Console.WriteLine($"archived_at: {workspace.ArchivedAt:O}");
  ```

  ```go Go
  client := anthropic.NewClient()

  workspace, err := client.Beta.Organization.Workspaces.Archive(context.Background(), "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ")
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", workspace.ID)
  fmt.Printf("archived_at: %s\n", workspace.ArchivedAt)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var workspace = client.beta().organization().workspaces()
      .archive("wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ");

  IO.println("id: " + workspace.id());
  IO.println("archived_at: " + workspace.archivedAt().orElseThrow());
  ```

  ```php PHP
  $client = new Client();

  $workspace = $client->beta->organization->workspaces->archive(
      workspaceID: 'wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ',
  );

  echo "id: {$workspace->id}\n";
  echo "archived_at: {$workspace->archivedAt?->format(DATE_ATOM)}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  workspace_id = "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  workspace = client.beta.organization.workspaces.archive(workspace_id)

  puts "id: #{workspace.id}"
  puts "archived_at: #{workspace.archived_at}"
  ```
</CodeGroup>

有关完整的参数详情和响应模式，请参阅[工作区 API 参考](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/retrieve)。

### 管理工作区成员

向工作区添加成员：

<CodeGroup>
  ```bash cURL
  curl -X POST "https://api.anthropic.com/v1/organizations/workspaces/wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ/members" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "user_id": "user_01XyDMpzjS89pFZXqSFUBDr6",
      "workspace_role": "workspace_developer"
    }'
  ```

  ```bash CLI
  ant beta:organization:workspaces:members add \
    --workspace-id wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ \
    --user-id user_01XyDMpzjS89pFZXqSFUBDr6 \
    --workspace-role workspace_developer
  ```

  ```python Python
  client = anthropic.Anthropic()

  member = client.beta.organization.workspaces.members.add(
      "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
      user_id="user_01XyDMpzjS89pFZXqSFUBDr6",
      workspace_role="workspace_developer",
  )

  print(f"user_id: {member.user_id}")
  print(f"workspace_role: {member.workspace_role}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const member = await client.beta.organization.workspaces.members.add(
    "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
    {
      user_id: "user_01XyDMpzjS89pFZXqSFUBDr6",
      workspace_role: "workspace_developer"
    }
  );

  console.log(`user_id: ${member.user_id}`);
  console.log(`workspace_role: ${member.workspace_role}`);
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Organization.Workspaces;

  AnthropicClient client = new();

  var member = await client.Beta.Organization.Workspaces.Members.Add(
      "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
      new()
      {
          UserID = "user_01XyDMpzjS89pFZXqSFUBDr6",
          WorkspaceRole = BetaNoBillingWorkspaceRole.WorkspaceDeveloper
      }
  );

  Console.WriteLine($"user_id: {member.UserID}");
  Console.WriteLine($"workspace_role: {member.WorkspaceRole.Raw()}");
  ```

  ```go Go
  client := anthropic.NewClient()

  member, err := client.Beta.Organization.Workspaces.Members.Add(
  	context.Background(),
  	"wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
  	anthropic.BetaOrganizationWorkspaceMemberAddParams{
  		UserID:        "user_01XyDMpzjS89pFZXqSFUBDr6",
  		WorkspaceRole: anthropic.BetaNoBillingWorkspaceRoleWorkspaceDeveloper,
  	},
  )
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("user_id: %s\n", member.UserID)
  fmt.Printf("workspace_role: %s\n", member.WorkspaceRole)
  ```

  ```java Java
  import com.anthropic.models.beta.organization.workspaces.BetaNoBillingWorkspaceRole;
  import com.anthropic.models.beta.organization.workspaces.members.MemberAddParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = MemberAddParams.builder()
          .userId("user_01XyDMpzjS89pFZXqSFUBDr6")
          .workspaceRole(BetaNoBillingWorkspaceRole.WORKSPACE_DEVELOPER)
          .build();
      var member = client.beta().organization().workspaces().members()
          .add("wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ", params);

      IO.println("user_id: " + member.userId());
      IO.println("workspace_role: " + member.workspaceRole().asString());
  }
  ```

  ```php PHP
  use Anthropic\Beta\Organization\Workspaces\NoBillingWorkspaceRole;
  // ...

  $client = new Client();

  $member = $client->beta->organization->workspaces->members->add(
      workspaceID: 'wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ',
      userID: 'user_01XyDMpzjS89pFZXqSFUBDr6',
      workspaceRole: NoBillingWorkspaceRole::WORKSPACE_DEVELOPER,
  );

  echo "user_id: {$member->userID}\n";
  echo "workspace_role: {$member->workspaceRole}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  workspace_id = "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  member = client.beta.organization.workspaces.members.add(
    workspace_id,
    user_id: "user_01XyDMpzjS89pFZXqSFUBDr6",
    workspace_role: :workspace_developer
  )

  puts "user_id: #{member.user_id}"
  puts "workspace_role: #{member.workspace_role}"
  ```
</CodeGroup>

更新成员的角色：

<CodeGroup>
  ```bash cURL
  curl -X POST "https://api.anthropic.com/v1/organizations/workspaces/wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ/members/user_01XyDMpzjS89pFZXqSFUBDr6" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{"workspace_role": "workspace_admin"}'
  ```

  ```bash CLI
  ant beta:organization:workspaces:members update \
    --user-id user_01XyDMpzjS89pFZXqSFUBDr6 \
    --workspace-id wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ \
    --workspace-role workspace_admin
  ```

  ```python Python
  client = anthropic.Anthropic()

  member = client.beta.organization.workspaces.members.update(
      "user_01XyDMpzjS89pFZXqSFUBDr6",
      workspace_id="wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
      workspace_role="workspace_admin",
  )

  print(f"user_id: {member.user_id}")
  print(f"workspace_role: {member.workspace_role}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const member = await client.beta.organization.workspaces.members.update(
    "user_01XyDMpzjS89pFZXqSFUBDr6",
    {
      workspace_id: "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
      workspace_role: "workspace_admin"
    }
  );

  console.log(`user_id: ${member.user_id}`);
  console.log(`workspace_role: ${member.workspace_role}`);
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Organization.Workspaces;

  AnthropicClient client = new();

  var member = await client.Beta.Organization.Workspaces.Members.Update(
      "user_01XyDMpzjS89pFZXqSFUBDr6",
      new()
      {
          WorkspaceID = "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
          WorkspaceRole = BetaWorkspaceRole.WorkspaceAdmin
      }
  );

  Console.WriteLine($"user_id: {member.UserID}");
  Console.WriteLine($"workspace_role: {member.WorkspaceRole.Raw()}");
  ```

  ```go Go
  client := anthropic.NewClient()

  member, err := client.Beta.Organization.Workspaces.Members.Update(
  	context.Background(),
  	"user_01XyDMpzjS89pFZXqSFUBDr6",
  	anthropic.BetaOrganizationWorkspaceMemberUpdateParams{
  		WorkspaceID:   "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
  		WorkspaceRole: anthropic.BetaWorkspaceRoleWorkspaceAdmin,
  	},
  )
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("user_id: %s\n", member.UserID)
  fmt.Printf("workspace_role: %s\n", member.WorkspaceRole)
  ```

  ```java Java
  import com.anthropic.models.beta.organization.workspaces.BetaWorkspaceRole;
  import com.anthropic.models.beta.organization.workspaces.members.MemberUpdateParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = MemberUpdateParams.builder()
          .workspaceId("wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ")
          .workspaceRole(BetaWorkspaceRole.WORKSPACE_ADMIN)
          .build();
      var member = client.beta().organization().workspaces().members()
          .update("user_01XyDMpzjS89pFZXqSFUBDr6", params);

      IO.println("user_id: " + member.userId());
      IO.println("workspace_role: " + member.workspaceRole().asString());
  }
  ```

  ```php PHP
  use Anthropic\Beta\Organization\Workspaces\WorkspaceRole;
  // ...

  $client = new Client();

  $member = $client->beta->organization->workspaces->members->update(
      userID: 'user_01XyDMpzjS89pFZXqSFUBDr6',
      workspaceID: 'wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ',
      workspaceRole: WorkspaceRole::WORKSPACE_ADMIN,
  );

  echo "user_id: {$member->userID}\n";
  echo "workspace_role: {$member->workspaceRole}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  user_id = "user_01XyDMpzjS89pFZXqSFUBDr6"
  member = client.beta.organization.workspaces.members.update(
    user_id,
    workspace_id: "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
    workspace_role: :workspace_admin
  )

  puts "user_id: #{member.user_id}"
  puts "workspace_role: #{member.workspace_role}"
  ```
</CodeGroup>

从工作区移除成员：

<CodeGroup>
  ```bash cURL
  curl -X DELETE "https://api.anthropic.com/v1/organizations/workspaces/wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ/members/user_01XyDMpzjS89pFZXqSFUBDr6" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:workspaces:members remove \
    --user-id user_01XyDMpzjS89pFZXqSFUBDr6 \
    --workspace-id wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ
  ```

  ```python Python
  client = anthropic.Anthropic()

  removed_member = client.beta.organization.workspaces.members.remove(
      "user_01XyDMpzjS89pFZXqSFUBDr6",
      workspace_id="wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
  )

  print(f"user_id: {removed_member.user_id}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const removedMember = await client.beta.organization.workspaces.members.remove(
    "user_01XyDMpzjS89pFZXqSFUBDr6",
    { workspace_id: "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ" }
  );

  console.log(`user_id: ${removedMember.user_id}`);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var removedMember = await client.Beta.Organization.Workspaces.Members.Remove(
      "user_01XyDMpzjS89pFZXqSFUBDr6",
      new() { WorkspaceID = "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ" }
  );

  Console.WriteLine($"user_id: {removedMember.UserID}");
  ```

  ```go Go
  client := anthropic.NewClient()

  removedMember, err := client.Beta.Organization.Workspaces.Members.Remove(
  	context.Background(),
  	"user_01XyDMpzjS89pFZXqSFUBDr6",
  	anthropic.BetaOrganizationWorkspaceMemberRemoveParams{
  		WorkspaceID: "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
  	},
  )
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("user_id: %s\n", removedMember.UserID)
  ```

  ```java Java
  import com.anthropic.models.beta.organization.workspaces.members.MemberRemoveParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = MemberRemoveParams.builder()
          .workspaceId("wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ")
          .build();
      var removedMember = client.beta().organization().workspaces().members()
          .remove("user_01XyDMpzjS89pFZXqSFUBDr6", params);

      IO.println("user_id: " + removedMember.userId());
  }
  ```

  ```php PHP
  $client = new Client();

  $removedMember = $client->beta->organization->workspaces->members->remove(
      userID: 'user_01XyDMpzjS89pFZXqSFUBDr6',
      workspaceID: 'wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ',
  );

  echo "user_id: {$removedMember->userID}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  user_id = "user_01XyDMpzjS89pFZXqSFUBDr6"
  removed_member = client.beta.organization.workspaces.members.remove(
    user_id,
    workspace_id: "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  )

  puts "user_id: #{removed_member.user_id}"
  ```
</CodeGroup>

有关完整的参数详情，请参阅[工作区成员 API 参考](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/members/retrieve)。

## API 密钥和资源范围

每个请求都恰好在一个工作区中运行，并且只能访问该工作区内的资源。具体是哪个工作区取决于[密钥类型](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)：

* **工作区密钥**（没有所有者的旧版密钥）属于创建它的工作区，并始终在该工作区中运行。
* **个人密钥**或**服务账户密钥**以其用户或服务账户的身份操作。单工作区密钥始终在创建时选择的工作区中运行。多工作区密钥在每个请求的 `anthropic-workspace-id` 请求头所指定的工作区中运行。账户必须拥有该工作区的访问权限才能使用它。

限定到工作区的资源包括：

* 通过 [Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files) 创建的**文件**
* 通过 [Batch API](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing) 创建的**消息批次**
* 通过 [Skills API](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide) 创建的**技能**

某些资源的管理方式有所不同：

* \*\*[MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview)\*\*使用通过[工作负载身份联合](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)获取的 `workspace:manage_tunnels` OAuth 令牌进行管理，而不是 API 密钥。隧道在工作区中创建，Console 的 **MCP 隧道**列表和托管代理服务器选择器仅显示当前工作区中的隧道；10 个活动隧道的上限适用于整个组织。隧道管理需要具有隧道管理权限的角色；组织开发者可以查看但不能更改它们。
* **工作区**本身和**组织成员**通过 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 在组织级别进行管理，使用 Admin API 密钥、`org:admin` OAuth 令牌，或未限定到特定工作区的个人或服务账户密钥。

要查找您组织的工作区 ID，请调用 [List Workspaces](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/list) 端点，或在 [Claude Console](https://platform.claude.com/settings/workspaces) 中查找。

<Note>
  在 Claude API、[Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 和 [Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry) 上，[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)也按工作区隔离。在 Amazon Bedrock 和 Google Cloud 上，提示缓存按组织隔离。
</Note>

## 识别 API 响应背后的工作区

Claude API 响应在 `request-id` 和 `anthropic-organization-id` [响应头](https://platform.claude.com/docs/zh-CN/api/overview#response-headers)之外还包含一个 `anthropic-workspace-id` 响应头。其值是请求的 API 密钥或访问令牌所解析到的工作区的带 `wrkspc_` 前缀的 ID，包括该工作区是 Default Workspace 的情况。例如，一个成功的响应包含如下响应头：

```http
HTTP/1.1 200 OK
request-id: req_018EeWyXxfu5pfWkrYcMdjWG
anthropic-organization-id: 0d0e7a3b-52f1-4c7e-9a51-3f6f2f7c1b9e
anthropic-workspace-id: wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ
```

当凭据未解析到工作区时（例如在 Admin API 请求中），或当请求在身份验证完成之前失败时（例如 401 错误），该响应头不存在。

以下示例发送一个 Messages API 请求并打印响应头中的工作区 ID：

<CodeGroup>
  ```bash cURL
  # -D - 打印响应头；-o /dev/null 丢弃响应体
  curl -sS -D - -o /dev/null https://api.anthropic.com/v1/messages \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "model": "claude-opus-5",
      "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello, Claude"}]
    }' | grep -i '^anthropic-workspace-id'
  ```

  ```bash CLI
  # --debug 会将 HTTP 响应（包括 Anthropic-Workspace-Id
  # 标头）打印到 stderr；> /dev/null 会隐藏 stdout 上的 JSON 正文
  ant --debug messages create \
    --model claude-opus-5 \
    --max-tokens 1024 \
    --message '{role: user, content: "Hello, Claude"}' > /dev/null
  ```

  ```python Python
  client = anthropic.Anthropic()

  response = client.messages.with_raw_response.create(
      model="claude-opus-5",
      max_tokens=1024,
      messages=[{"role": "user", "content": "Hello, Claude"}],
  )
  workspace_id = response.headers.get("anthropic-workspace-id")
  print(f"Workspace ID: {workspace_id}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const { response } = await client.messages
    .create({
      model: "claude-opus-5",
      max_tokens: 1024,
      messages: [{ role: "user", content: "Hello, Claude" }]
    })
    .withResponse();
  console.log("Workspace ID:", response.headers.get("anthropic-workspace-id"));
  ```

  ```csharp C#
  AnthropicClient client = new();

  using var response = await client.WithRawResponse.Messages.Create(new()
  {
      Model = Model.ClaudeOpus5,
      MaxTokens = 1024,
      Messages = [new() { Role = Role.User, Content = "Hello, Claude" }]
  });
  var workspaceId = response.GetHeaderValues("anthropic-workspace-id").First();
  Console.WriteLine($"Workspace ID: {workspaceId}");
  ```

  ```go Go
  client := anthropic.NewClient()

  var response *http.Response
  _, err := client.Messages.New(
  	context.Background(),
  	anthropic.MessageNewParams{
  		Model:     anthropic.ModelClaudeOpus5,
  		MaxTokens: 1024,
  		Messages: []anthropic.MessageParam{
  			anthropic.NewUserMessage(anthropic.NewTextBlock("Hello, Claude")),
  		},
  	},
  	option.WithResponseInto(&response),
  )
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Println("Workspace ID:", response.Header.Get("anthropic-workspace-id"))
  ```

  ```java Java
  import com.anthropic.client.AnthropicClient;
  import com.anthropic.client.okhttp.AnthropicOkHttpClient;
  import com.anthropic.core.http.HttpResponseFor;
  import com.anthropic.models.messages.Message;
  import com.anthropic.models.messages.MessageCreateParams;
  import com.anthropic.models.messages.Model;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      HttpResponseFor<Message> response = client.messages().withRawResponse().create(
          MessageCreateParams.builder()
              .model(Model.CLAUDE_OPUS_5)
              .maxTokens(1024)
              .addUserMessage("Hello, Claude")
              .build()
      );

      String workspaceId = response.headers().values("anthropic-workspace-id").getFirst();
      IO.println("Workspace ID: " + workspaceId);
  }
  ```

  ```php PHP
  $client = new Client();

  $response = $client->messages->raw->create([
      'model' => Model::CLAUDE_OPUS_5,
      'maxTokens' => 1024,
      'messages' => [['role' => 'user', 'content' => 'Hello, Claude']],
  ]);
  echo 'Workspace ID: ' . $response->getHeaderLine('anthropic-workspace-id') . "\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  # 在每请求中间件中读取响应头，该中间件会在 SDK 解析之前
  # 接收到原始 HTTP 响应
  workspace_id = nil
  read_workspace_id = lambda do |request, call_next|
    response = call_next.call(request)
    # response.headers 中的键均为小写
    workspace_id = response.headers["anthropic-workspace-id"]
    response
  end

  client.messages.create(
    model: Anthropic::Model::CLAUDE_OPUS_5,
    max_tokens: 1024,
    messages: [{ role: "user", content: "Hello, Claude" }],
    request_options: { middleware: [read_workspace_id] }
  )
  puts "Workspace ID: #{workspace_id}"
  ```
</CodeGroup>

```text Output wrap
Workspace ID: wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ
```

同样的访问器也可以从其他 Claude API 端点读取该响应头，包括 [Claude Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview) API。例如，从[创建会话](https://platform.claude.com/docs/zh-CN/managed-agents/sessions)的响应中读取 `anthropic-workspace-id`，以记录该会话属于哪个工作区。

利用响应中的工作区 ID，您可以：

* 确认该请求计入了哪个工作区的使用量、成本和[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)
* 将其与[使用量和成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api) 报告中以及 [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api) 对象（例如 API 密钥）上的 `workspace_id` 字段进行匹配（两者对于 Default Workspace 都报告 `null`，API 密钥对于全工作区密钥也是如此；API 密钥的 `scope` 字段可以区分这两者，并且对于绑定到单个工作区的密钥，该字段携带该工作区的真实 ID）
* 通过使用 [Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)将其传递给 [Get Workspace](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/retrieve) 来检查它是否是您的 Default Workspace 的 ID：Default Workspace 会返回 `"name": "Default"`，即使 [List Workspaces](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/list) 省略了它
* 在 [Console](https://platform.claude.com/settings/workspaces) 中打开该工作区以查找该请求的资源，例如会话、文件、消息批次和技能

## 工作区限制

您可以为每个工作区设置自定义的支出和速率限制，以防止过度使用并确保公平的资源分配。

### 设置工作区限制

您可以将工作区限制设置为低于（但不能高于）您组织的限制：

* **支出限制：**限制工作区的每月支出。在 [Claude Console](https://platform.claude.com/settings/workspaces) 中工作区的**支出限制**设置选项卡上进行设置。
* **速率限制：**限制每分钟请求数、每分钟输入令牌数或每分钟输出令牌数。在 [Claude Console](https://platform.claude.com/settings/workspaces) 中工作区的**速率限制**设置选项卡上进行设置。

<Note>
  - 您无法对 Default Workspace 设置限制
  - 如果未设置，工作区限制与组织的限制一致
  - 组织范围的限制始终适用，即使各工作区限制加起来超过该值
</Note>

有关速率限制及其工作原理的详细信息，请参阅[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)。您还可以使用[速率限制 API](https://platform.claude.com/docs/zh-CN/manage-claude/rate-limits-api) 以编程方式读取您当前的组织和工作区速率限制。

## 使用量和成本跟踪

使用[使用量和成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api) 按工作区跟踪使用量和成本：

```bash cURL
curl "https://api.anthropic.com/v1/organizations/usage_report/messages?\
starting_at=2025-01-01T00:00:00Z&\
ending_at=2025-01-08T00:00:00Z&\
workspace_ids[]=wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ&\
group_by[]=workspace_id&\
bucket_width=1d" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY"
```

归属于 Default Workspace 的使用量和成本的 `workspace_id` 值为 `null`。

## 常见用例

### 环境分离

为开发、预发布和生产创建单独的工作区：

| 工作区 | 用途               |
| --- | ---------------- |
| 开发  | 使用较低速率限制进行测试和实验  |
| 预发布 | 使用类似生产的限制进行生产前测试 |
| 生产  | 具有完整速率限制和监控的实时流量 |

### 团队或部门隔离

将工作区分配给不同的团队，以便进行成本分摊和访问控制：

* 具有开发者访问权限的**工程团队**
* 拥有自己 API 密钥的**数据科学团队**
* 对客户工具具有有限访问权限的**支持团队**

### 基于项目的组织

为特定项目或产品创建工作区，以单独跟踪使用量和成本。

## 最佳实践

<Steps>
  <Step title="规划您的工作区结构">
    在创建工作区之前考虑如何组织它们。思考计费、访问控制和使用量跟踪方面的需求。
  </Step>

  <Step title="使用有意义的名称">
    清晰地命名工作区以表明其用途（例如"Production - Customer Chatbot"或"Dev - Internal Tools"）。
  </Step>

  <Step title="设置适当的限制">
    配置支出和速率限制，以防止意外成本并确保公平的资源分配。
  </Step>

  <Step title="定期审核访问权限">
    定期审查工作区成员资格，以确保只有适当的用户拥有访问权限。
  </Step>

  <Step title="监控使用量">
    使用[使用量和成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api) 跟踪工作区级别的消耗。
  </Step>
</Steps>

## 常见问题

<AccordionGroup>
  <Accordion title="什么是 Default Workspace？">
    每个组织都有一个"Default Workspace"，它无法被重命名、归档或删除。与每个工作区一样，它拥有一个 `wrkspc_` ID：API 在 [`anthropic-workspace-id` 响应头](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#identify-the-workspace-behind-an-api-response)中返回它，您可以将其传递给 [Get Workspace](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/retrieve) 和 [Update Workspace](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/update)。它没有自己的成员列表，因为对它的访问权限取决于每个成员的组织角色。它不会出现在 [List Workspaces](https://platform.claude.com/docs/zh-CN/api/admin/workspaces/list) 的结果中，并且属于它的 API 密钥、使用报告和成本报告的 `workspace_id` 显示为 `null`，全工作区 API 密钥也是如此；API 密钥的 `scope` 字段可以区分这两者，并且对于属于 Default Workspace 的密钥，该字段携带其真实 ID。
  </Accordion>

  <Accordion title="什么是 Claude Code 工作区？">
    当您组织中的成员首次使用其 Console 账户登录 Claude Code 时，Anthropic 会自动创建 Claude Code 工作区。它将 Claude Code 的 API 密钥、使用量和速率限制与您的其他工作负载隔离开来。详情请参阅 [Claude Code 工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#claude-code-workspace)。
  </Accordion>

  <Accordion title="工作区有数量限制吗？">
    有。默认情况下每个组织最多可以拥有 100 个工作区，已归档的工作区不计入此限制。如果您需要更多，请联系您的账户团队。
  </Accordion>

  <Accordion title="组织角色如何影响工作区访问权限？">
    组织管理员自动在所有工作区中获得 Workspace Admin 角色。组织计费成员自动获得 Workspace Billing 角色。组织用户和开发者必须被手动添加到每个工作区。
  </Accordion>

  <Accordion title="工作区中可以分配哪些角色？">
    组织用户和开发者可以被分配 Workspace Admin、Workspace Developer、Workspace Limited Developer 或 Workspace User 角色。Workspace Billing 角色无法手动分配；它是通过拥有组织 `billing` 角色继承而来的。
  </Accordion>

  <Accordion title="组织管理员或计费成员的工作区角色可以更改吗？">
    组织管理员和计费成员在担任这些组织角色期间，其工作区角色无法更改，也无法从工作区中移除（有一个例外：计费成员可以升级为 Workspace Admin 角色）。对于受此约束的其他所有人，请先更改其组织角色以更改其工作区访问权限。
  </Accordion>

  <Accordion title="当组织角色发生变化时，工作区访问权限会怎样？">
    如果组织管理员或计费成员被降级为用户或开发者，他们将失去对所有工作区的访问权限，但手动分配了角色的工作区除外。当用户被提升为管理员或计费角色时，他们会自动获得对所有工作区的访问权限。
  </Accordion>

  <Accordion title="当用户从工作区中移除时，API 密钥会怎样？">
    行为取决于[密钥类型](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)。

    个人或服务账户密钥在其用户或服务账户从工作区中移除后不久便会在该工作区中停止工作。即使创建服务账户密钥的用户被移除，该密钥仍会继续工作。工作区 API 密钥会继续工作。在 [Claude Code 工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#claude-code-workspace)中，每个密钥都绑定到创建它的成员，并在该成员被移除时停止工作。

    个人密钥在其用户从组织中移除时会被归档。如果该用户被重新邀请，他们需要创建新密钥；已归档的密钥不会恢复。
  </Accordion>
</AccordionGroup>

## 另请参阅

* [Admin API](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)
* [Admin API 参考](https://platform.claude.com/docs/zh-CN/api/admin)
* [速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits)
* [使用量和成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api)
