---
title: Admin API
url: https://platform.claude.com/docs/zh-CN/manage-claude/admin-api
description: 使用 Admin API 以编程方式管理组织成员、工作区、邀请和 API 密钥，可使用 Admin API 密钥、`org:admin` OAuth 令牌，或个人密钥或服务账户密钥进行身份验证。
---

<Tip>
  **Admin API 不适用于个人账户。** 要与团队成员协作并添加成员，请在 **Console → Settings → Organization** 中设置您的组织。
</Tip>

[Admin API](https://platform.claude.com/docs/zh-CN/api/admin) 让您能够以编程方式管理组织的成员、工作区、邀请和 API 密钥，而无需在 [Claude Console](https://platform.claude.com/) 中手动操作。

<Check>
  **Admin API 需要特殊访问权限**

  Admin API 接受三种凭证：

  * **Admin API 密钥**（以 `sk-ant-admin...` 开头），通过 `x-api-key` 标头发送。只有具有 admin 角色的组织成员才能配置此类密钥。请参阅[创建 Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)。
  * 具有 `org:admin` 作用域的 **OAuth bearer 令牌**，通过 `authorization: Bearer` 标头发送。只有具有 admin、owner 或 primary owner 角色的成员才能获取此类令牌。请参阅[获取 OAuth bearer 令牌](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#oauth-bearer-token)。
  * 未限定于特定工作区的**个人密钥**或**服务账户密钥**，通过 `x-api-key` 标头发送。该密钥拥有与所关联账户相同的权限。请参阅[密钥类型](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)。
</Check>

<Note>
  **Claude Enterprise：** Claude Enterprise（claude.ai）组织使用在 claude.ai 中创建的限定作用域 API 密钥调用 Admin API。在本页中，只有成员和邀请端点适用于它们。它们还可使用 Enterprise 专属端点：群组和自定义角色读取，以及[支出限额](https://platform.claude.com/docs/zh-CN/manage-claude/spend-limits-api)。请参阅[用户管理](https://platform.claude.com/docs/zh-CN/manage-claude/user-management)。
</Note>

<Note>
  **Claude Platform on AWS：** 在 Claude Platform on AWS 上，仅提供工作区端点（`/v1/organizations/workspaces` 上的创建、获取、列出、更新和归档）以及外部密钥端点（`/v1/organizations/external_keys` 上的注册、获取、列出、更新和删除，用于 [CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#claude-platform-on-aws)；没有验证端点，因为密钥在附加到工作区时会进行验证）。组织成员、工作区成员、邀请、API 密钥以及用量、成本和速率限制报告均不可用。请参阅 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)。
</Note>

## 身份验证

使用三种凭证中的任意一种进行身份验证。Admin API 密钥涵盖大多数端点。服务账户、联合身份颁发者和联合规则端点仅接受 `org:admin` OAuth 令牌。个人密钥或服务账户密钥与 Admin API 密钥一样，通过 `x-api-key` 标头发送。以下示例分别使用 OAuth 令牌和 Admin API 密钥调用[组织信息端点](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#accessing-organization-info)。

Python、TypeScript、C#、Go、Java、PHP 和 Ruby SDK 在 `client.beta.organization` 下提供 Admin API，`ant` CLI 则在 `ant beta:organization` 下提供。本页的示例使用默认客户端，它会从 `ANTHROPIC_API_KEY` 读取 Admin API 密钥，或从 `ANTHROPIC_AUTH_TOKEN` 读取 OAuth bearer 令牌。Python、TypeScript、C#、Go 和 Java 中的 SDK 列表方法返回一个按需获取更多页面的迭代器，因此 `limit` 设置的是页面大小，而非总数。PHP、Ruby 和 curl 示例返回单页结果。在 CLI 中，`--limit` 会限制成员、邀请、工作区、工作区成员和 API 密钥列表的结果数量。有关每个端点的参数和响应，请参阅 [Admin API 参考](https://platform.claude.com/docs/zh-CN/api/admin)。

### OAuth bearer 令牌

使用 [`ant` CLI](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/quickstart) 在具有 `org:admin` 作用域的专用配置文件下登录（请参阅[管理员访问](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/cli/authentication#admin-access)），然后导出 bearer 令牌。`--profile admin` 会将 `org:admin` 凭证存储在其自己的配置文件下，并将其设为 CLI 的活动配置文件。导出的变量适用于该 shell 中的每个 SDK 和 CLI 调用。请使用专门用于管理的 shell，完成后取消设置该变量，并使用 `ant profile activate default` 将 CLI 切换回去：

```bash CLI
ant auth login --profile admin --scope "org:admin"
export ANTHROPIC_AUTH_TOKEN=$(ant auth print-credentials --profile admin --access-token)
```

交互式令牌的有效期很短。如果请求开始返回 401，请重新运行 `export` 命令以刷新令牌。

SDK 和 `ant` CLI 会自动读取 `ANTHROPIC_AUTH_TOKEN`。请在同一 shell 中保持 `ANTHROPIC_API_KEY` 未设置，以便它们发送 bearer 令牌。自动化工作负载可跳过登录：它们通过工作负载身份联合进行身份验证，SDK 和 CLI 会根据联合环境变量执行令牌交换。请参阅[引导工作负载以管理 WIF](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api#bootstrap-a-workload-to-manage-wif)。

使用导出的令牌调用 Admin API：

<CodeGroup>
  ```bash cURL
  curl --fail-with-body -sS "https://api.anthropic.com/v1/organizations/me" \
    -H "authorization: Bearer $ANTHROPIC_AUTH_TOKEN" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization retrieve
  ```

  ```python Python
  client = anthropic.Anthropic()

  organization = client.beta.organization.retrieve()

  print(f"id: {organization.id}")
  print(f"name: {organization.name}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const organization = await client.beta.organization.retrieve();

  console.log(`id: ${organization.id}`);
  console.log(`name: ${organization.name}`);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var organization = await client.Beta.Organization.Retrieve();

  Console.WriteLine($"id: {organization.ID}");
  Console.WriteLine($"name: {organization.Name}");
  ```

  ```go Go
  client := anthropic.NewClient()

  organization, err := client.Beta.Organization.Get(context.Background())
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", organization.ID)
  fmt.Printf("name: %s\n", organization.Name)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var organization = client.beta().organization().retrieve();

  IO.println("id: " + organization.id());
  IO.println("name: " + organization.name());
  ```

  ```php PHP
  $client = new Client();

  $organization = $client->beta->organization->retrieve();

  echo "id: {$organization->id}\n";
  echo "name: {$organization->name}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  organization = client.beta.organization.retrieve

  puts "id: #{organization.id}"
  puts "name: #{organization.name}"
  ```
</CodeGroup>

`org:admin` 令牌授予对整个组织的访问权限，无论底层配置文件或[联合规则](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#federation-rules)绑定到哪个工作区。

对于 CI 和其他非交互式工作负载，请使用工作负载身份联合（Workload Identity Federation）签发令牌，而不是交互式登录。请参阅[使用 Admin API 管理 WIF](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api#workload-ci-and-automation)。

### Admin API 密钥

要为您的组织类型创建 Admin API 密钥，请参阅[创建 Admin API 密钥](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api-keys)。

<CodeGroup>
  ```bash cURL
  curl --fail-with-body -sS "https://api.anthropic.com/v1/organizations/me" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization retrieve
  ```

  ```python Python
  client = anthropic.Anthropic()

  organization = client.beta.organization.retrieve()

  print(f"id: {organization.id}")
  print(f"name: {organization.name}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const organization = await client.beta.organization.retrieve();

  console.log(`id: ${organization.id}`);
  console.log(`name: ${organization.name}`);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var organization = await client.Beta.Organization.Retrieve();

  Console.WriteLine($"id: {organization.ID}");
  Console.WriteLine($"name: {organization.Name}");
  ```

  ```go Go
  client := anthropic.NewClient()

  organization, err := client.Beta.Organization.Get(context.Background())
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", organization.ID)
  fmt.Printf("name: %s\n", organization.Name)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var organization = client.beta().organization().retrieve();

  IO.println("id: " + organization.id());
  IO.println("name: " + organization.name());
  ```

  ```php PHP
  $client = new Client();

  $organization = $client->beta->organization->retrieve();

  echo "id: {$organization->id}\n";
  echo "name: {$organization->name}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  organization = client.beta.organization.retrieve

  puts "id: #{organization.id}"
  puts "name: #{organization.name}"
  ```
</CodeGroup>

## Admin API 的工作原理

使用[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#authentication)中的任意凭证进行身份验证，然后管理以下资源：

* 组织成员及其角色
* 组织邀请
* 工作区及其成员
* API 密钥
* 服务账户、联合身份颁发者和联合规则（仅限 `org:admin` OAuth 令牌）

常见用途包括自动化入职和离职流程、管理工作区访问权限以及审计 API 密钥。

## 组织角色和权限

共有五种组织级角色。详情请参阅 [API Console 角色和权限](https://support.claude.com/en/articles/10186004-api-console-roles-and-permissions)。

| 角色                 | 权限                                                                        |
| ------------------ | ------------------------------------------------------------------------- |
| user               | 可以使用 playground                                                           |
| claude\_code\_user | 可以使用 playground 和 [Claude Code](https://code.claude.com/docs/en/overview) |
| developer          | 可以使用 playground 并管理 API 密钥                                                |
| billing            | 可以使用 playground 并管理账单详情                                                   |
| admin              | 可以执行上述所有操作，并可管理用户                                                         |

组织 owner 和 primary owner 拥有所有 admin 权限，并且还可以管理 admin。本页中所有对 admin 角色的引用同样适用于 owner 和 primary owner。

## 关键概念

### 组织成员

列出[组织成员](https://platform.claude.com/docs/zh-CN/api/admin-api/users/get-user)、更新其角色以及移除成员。

列出您组织的成员：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/users?limit=10" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:users list --limit 10
  ```

  ```python Python
  client = anthropic.Anthropic()

  users = client.beta.organization.users.list(limit=10)

  # 根据需要自动获取更多页面。
  for user in users:
      print(f"{user.id}: {user.email} ({user.role})")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const users = await client.beta.organization.users.list({ limit: 10 });

  for await (const user of users) {
    console.log(`${user.id}: ${user.email} (${user.role})`);
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var page = await client.Beta.Organization.Users.List(new() { Limit = 10 });

  await foreach (var user in page.Paginate())
  {
      Console.WriteLine($"{user.ID}: {user.Email} ({user.Role.Raw()})");
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  users := client.Beta.Organization.Users.ListAutoPaging(context.Background(), anthropic.BetaOrganizationUserListParams{
  	Limit: anthropic.Int(10),
  })

  for users.Next() {
  	user := users.Current()
  	fmt.Printf("%s: %s (%s)\n", user.ID, user.Email, user.Role)
  }
  if err := users.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.beta.organization.users.UserListParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = UserListParams.builder()
          .limit(10)
          .build();
      var users = client.beta().organization().users().list(params);

      for (var user : users.autoPager()) {
          IO.println(user.id() + ": " + user.email() + " (" + user.role().asString() + ")");
      }
  }
  ```

  ```php PHP
  $client = new Client();

  $users = $client->beta->organization->users->list(limit: 10);

  foreach ($users->getItems() as $user) {
      echo "{$user->id}: {$user->email} ({$user->role})\n";
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  users = client.beta.organization.users.list(limit: 10)

  users.data.each do |user|
    puts "#{user.id}: #{user.email} (#{user.role})"
  end
  ```
</CodeGroup>

更新成员的角色：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/users/user_01XyDMpzjS89pFZXqSFUBDr6" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{"role": "developer"}'
  ```

  ```bash CLI
  ant beta:organization:users update \
    --user-id user_01XyDMpzjS89pFZXqSFUBDr6 \
    --role developer
  ```

  ```python Python
  client = anthropic.Anthropic()

  user = client.beta.organization.users.update(
      "user_01XyDMpzjS89pFZXqSFUBDr6", role="developer"
  )

  print(f"id: {user.id}")
  print(f"role: {user.role}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const user = await client.beta.organization.users.update("user_01XyDMpzjS89pFZXqSFUBDr6", {
    role: "developer"
  });

  console.log(`id: ${user.id}`);
  console.log(`role: ${user.role}`);
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Organization.Users;
  // ...

  AnthropicClient client = new();

  var user = await client.Beta.Organization.Users.Update(
      "user_01XyDMpzjS89pFZXqSFUBDr6",
      new() { Role = Role.Developer }
  );

  Console.WriteLine($"id: {user.ID}");
  Console.WriteLine($"role: {user.Role.Raw()}");
  ```

  ```go Go
  client := anthropic.NewClient()

  user, err := client.Beta.Organization.Users.Update(
  	context.Background(),
  	"user_01XyDMpzjS89pFZXqSFUBDr6",
  	anthropic.BetaOrganizationUserUpdateParams{
  		Role: anthropic.BetaOrganizationUserUpdateParamsRoleDeveloper,
  	},
  )
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", user.ID)
  fmt.Printf("role: %s\n", user.Role)
  ```

  ```java Java
  import com.anthropic.models.beta.organization.users.UserUpdateParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = UserUpdateParams.builder()
          .role(UserUpdateParams.Role.DEVELOPER)
          .build();
      var user = client.beta().organization().users()
          .update("user_01XyDMpzjS89pFZXqSFUBDr6", params);

      IO.println("id: " + user.id());
      IO.println("role: " + user.role().asString());
  }
  ```

  ```php PHP
  use Anthropic\Beta\Organization\Users\UserUpdateParams\Role;
  // ...

  $client = new Client();

  $user = $client->beta->organization->users->update(
      userID: 'user_01XyDMpzjS89pFZXqSFUBDr6',
      role: Role::DEVELOPER,
  );

  echo "id: {$user->id}\n";
  echo "role: {$user->role}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  user_id = "user_01XyDMpzjS89pFZXqSFUBDr6"
  user = client.beta.organization.users.update(user_id, role: :developer)

  puts "id: #{user.id}"
  puts "role: #{user.role}"
  ```
</CodeGroup>

从组织中移除成员：

<CodeGroup>
  ```bash cURL
  curl -X DELETE "https://api.anthropic.com/v1/organizations/users/user_01XyDMpzjS89pFZXqSFUBDr6" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:users remove --user-id user_01XyDMpzjS89pFZXqSFUBDr6
  ```

  ```python Python
  client = anthropic.Anthropic()

  removed_user = client.beta.organization.users.remove("user_01XyDMpzjS89pFZXqSFUBDr6")

  print(f"id: {removed_user.id}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const removedUser = await client.beta.organization.users.remove(
    "user_01XyDMpzjS89pFZXqSFUBDr6"
  );

  console.log(`id: ${removedUser.id}`);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var removedUser = await client.Beta.Organization.Users.Remove("user_01XyDMpzjS89pFZXqSFUBDr6");

  Console.WriteLine($"id: {removedUser.ID}");
  ```

  ```go Go
  client := anthropic.NewClient()

  removedUser, err := client.Beta.Organization.Users.Remove(context.Background(), "user_01XyDMpzjS89pFZXqSFUBDr6")
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", removedUser.ID)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var removedUser = client.beta().organization().users()
      .remove("user_01XyDMpzjS89pFZXqSFUBDr6");

  IO.println("id: " + removedUser.id());
  ```

  ```php PHP
  $client = new Client();

  $removedUser = $client->beta->organization->users->remove(
      userID: 'user_01XyDMpzjS89pFZXqSFUBDr6',
  );

  echo "id: {$removedUser->id}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  user_id = "user_01XyDMpzjS89pFZXqSFUBDr6"
  removed_user = client.beta.organization.users.remove(user_id)

  puts "id: #{removed_user.id}"
  ```
</CodeGroup>

### 组织邀请

邀请用户加入您的组织并管理待处理的[邀请](https://platform.claude.com/docs/zh-CN/api/admin-api/invites/get-invite)。

邀请用户加入您的组织：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/invites" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "email": "user@example.com",
      "role": "developer"
    }'
  ```

  ```bash CLI
  ant beta:organization:invites create --email user@example.com --role developer
  ```

  ```python Python
  client = anthropic.Anthropic()

  invite = client.beta.organization.invites.create(
      email="user@example.com", role="developer"
  )

  print(f"id: {invite.id}")
  print(f"email: {invite.email}")
  print(f"status: {invite.status}")
  print(f"expires_at: {invite.expires_at}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const invite = await client.beta.organization.invites.create({
    email: "user@example.com",
    role: "developer"
  });

  console.log(`id: ${invite.id}`);
  console.log(`email: ${invite.email}`);
  console.log(`status: ${invite.status}`);
  console.log(`expires_at: ${invite.expires_at}`);
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Organization.Invites;
  // ...

  AnthropicClient client = new();

  var invite = await client.Beta.Organization.Invites.Create(new()
  {
      Email = "user@example.com",
      Role = Role.Developer
  });

  Console.WriteLine($"id: {invite.ID}");
  Console.WriteLine($"email: {invite.Email}");
  Console.WriteLine($"status: {invite.Status.Raw()}");
  Console.WriteLine($"expires_at: {invite.ExpiresAt:O}");
  ```

  ```go Go
  client := anthropic.NewClient()

  invite, err := client.Beta.Organization.Invites.New(context.Background(), anthropic.BetaOrganizationInviteNewParams{
  	Email: "user@example.com",
  	Role:  anthropic.BetaOrganizationInviteNewParamsRoleDeveloper,
  })
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", invite.ID)
  fmt.Printf("email: %s\n", invite.Email)
  fmt.Printf("status: %s\n", invite.Status)
  fmt.Printf("expires_at: %s\n", invite.ExpiresAt.Format(time.RFC3339))
  ```

  ```java Java
  import com.anthropic.models.beta.organization.invites.InviteCreateParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = InviteCreateParams.builder()
          .email("user@example.com")
          .role(InviteCreateParams.Role.DEVELOPER)
          .build();
      var invite = client.beta().organization().invites().create(params);

      IO.println("id: " + invite.id());
      IO.println("email: " + invite.email());
      IO.println("status: " + invite.status().asString());
      IO.println("expires_at: " + invite.expiresAt());
  }
  ```

  ```php PHP
  use Anthropic\Beta\Organization\Invites\InviteCreateParams\Role;
  // ...

  $client = new Client();

  $invite = $client->beta->organization->invites->create(
      email: 'user@example.com',
      role: Role::DEVELOPER,
  );

  echo "id: {$invite->id}\n";
  echo "email: {$invite->email}\n";
  echo "status: {$invite->status}\n";
  echo "expires_at: {$invite->expiresAt->format(DATE_ATOM)}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  invite = client.beta.organization.invites.create(email: "user@example.com", role: :developer)

  puts "id: #{invite.id}"
  puts "email: #{invite.email}"
  puts "status: #{invite.status}"
  puts "expires_at: #{invite.expires_at.iso8601}"
  ```
</CodeGroup>

列出待处理的邀请：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/invites?limit=10" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:invites list --limit 10
  ```

  ```python Python
  client = anthropic.Anthropic()

  invites = client.beta.organization.invites.list(limit=10)

  # 根据需要自动获取更多页面。
  for invite in invites:
      print(f"{invite.id}: {invite.email} ({invite.status})")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const invites = await client.beta.organization.invites.list({ limit: 10 });

  for await (const invite of invites) {
    console.log(`${invite.id}: ${invite.email} (${invite.status})`);
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var page = await client.Beta.Organization.Invites.List(new() { Limit = 10 });

  await foreach (var invite in page.Paginate())
  {
      Console.WriteLine($"{invite.ID}: {invite.Email} ({invite.Status.Raw()})");
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  invites := client.Beta.Organization.Invites.ListAutoPaging(context.Background(), anthropic.BetaOrganizationInviteListParams{
  	Limit: anthropic.Int(10),
  })

  for invites.Next() {
  	invite := invites.Current()
  	fmt.Printf("%s: %s (%s)\n", invite.ID, invite.Email, invite.Status)
  }
  if err := invites.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.beta.organization.invites.InviteListParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = InviteListParams.builder()
          .limit(10)
          .build();
      var invites = client.beta().organization().invites().list(params);

      for (var invite : invites.autoPager()) {
          IO.println(invite.id() + ": " + invite.email() + " (" + invite.status().asString() + ")");
      }
  }
  ```

  ```php PHP
  $client = new Client();

  $invites = $client->beta->organization->invites->list(limit: 10);

  foreach ($invites->getItems() as $invite) {
      echo "{$invite->id}: {$invite->email} ({$invite->status})\n";
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  invites = client.beta.organization.invites.list(limit: 10)

  invites.data.each do |invite|
    puts "#{invite.id}: #{invite.email} (#{invite.status})"
  end
  ```
</CodeGroup>

删除邀请：

<CodeGroup>
  ```bash cURL
  curl -X DELETE "https://api.anthropic.com/v1/organizations/invites/invite_015gWxHNr6h6TdRPZTmuCGnn" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:invites delete --invite-id invite_015gWxHNr6h6TdRPZTmuCGnn
  ```

  ```python Python
  client = anthropic.Anthropic()

  deleted_invite = client.beta.organization.invites.delete(
      "invite_015gWxHNr6h6TdRPZTmuCGnn"
  )

  print(f"id: {deleted_invite.id}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const deletedInvite = await client.beta.organization.invites.delete(
    "invite_015gWxHNr6h6TdRPZTmuCGnn"
  );

  console.log(`id: ${deletedInvite.id}`);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var deletedInvite = await client.Beta.Organization.Invites.Delete(
      "invite_015gWxHNr6h6TdRPZTmuCGnn"
  );

  Console.WriteLine($"id: {deletedInvite.ID}");
  ```

  ```go Go
  client := anthropic.NewClient()

  deletedInvite, err := client.Beta.Organization.Invites.Delete(context.Background(), "invite_015gWxHNr6h6TdRPZTmuCGnn")
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", deletedInvite.ID)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var deletedInvite = client.beta().organization().invites()
      .delete("invite_015gWxHNr6h6TdRPZTmuCGnn");

  IO.println("id: " + deletedInvite.id());
  ```

  ```php PHP
  $client = new Client();

  $deletedInvite = $client->beta->organization->invites->delete(
      inviteID: 'invite_015gWxHNr6h6TdRPZTmuCGnn',
  );

  echo "id: {$deletedInvite->id}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  invite_id = "invite_015gWxHNr6h6TdRPZTmuCGnn"
  deleted_invite = client.beta.organization.invites.delete(invite_id)

  puts "id: #{deleted_invite.id}"
  ```
</CodeGroup>

### 工作区

有关 Console 和 API 示例，请参阅[工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces)。

### 工作区成员

管理[用户对特定工作区的访问权限](https://platform.claude.com/docs/zh-CN/api/admin-api/workspace_members/get-workspace-member)：

向工作区添加成员：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/workspaces/wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ/members" \
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

列出工作区的成员：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/workspaces/wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ/members?limit=10" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:workspaces:members list \
    --workspace-id wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ \
    --limit 10
  ```

  ```python Python
  client = anthropic.Anthropic()

  members = client.beta.organization.workspaces.members.list(
      "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ", limit=10
  )

  # 根据需要自动获取更多页面。
  for member in members:
      print(f"{member.user_id}: {member.workspace_role}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const members = await client.beta.organization.workspaces.members.list(
    "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
    { limit: 10 }
  );

  for await (const member of members) {
    console.log(`${member.user_id}: ${member.workspace_role}`);
  }
  ```

  ```csharp C#
  AnthropicClient client = new();

  var page = await client.Beta.Organization.Workspaces.Members.List(
      "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
      new() { Limit = 10 }
  );

  await foreach (var member in page.Paginate())
  {
      Console.WriteLine($"{member.UserID}: {member.WorkspaceRole.Raw()}");
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  members := client.Beta.Organization.Workspaces.Members.ListAutoPaging(
  	context.Background(),
  	"wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ",
  	anthropic.BetaOrganizationWorkspaceMemberListParams{
  		Limit: anthropic.Int(10),
  	},
  )

  for members.Next() {
  	member := members.Current()
  	fmt.Printf("%s: %s\n", member.UserID, member.WorkspaceRole)
  }
  if err := members.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.beta.organization.workspaces.members.MemberListParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = MemberListParams.builder()
          .limit(10)
          .build();
      var members = client.beta().organization().workspaces().members()
          .list("wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ", params);

      for (var member : members.autoPager()) {
          IO.println(member.userId() + ": " + member.workspaceRole().asString());
      }
  }
  ```

  ```php PHP
  $client = new Client();

  $members = $client->beta->organization->workspaces->members->list(
      workspaceID: 'wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ',
      limit: 10,
  );

  foreach ($members->getItems() as $member) {
      echo "{$member->userID}: {$member->workspaceRole}\n";
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  workspace_id = "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  members = client.beta.organization.workspaces.members.list(workspace_id, limit: 10)

  members.data.each do |member|
    puts "#{member.user_id}: #{member.workspace_role}"
  end
  ```
</CodeGroup>

更新工作区成员的角色：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/workspaces/wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ/members/user_01XyDMpzjS89pFZXqSFUBDr6" \
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

从工作区中移除成员：

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
      "user_01XyDMpzjS89pFZXqSFUBDr6", workspace_id="wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
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

### API 密钥

监控和管理 [API 密钥](https://platform.claude.com/docs/zh-CN/api/admin/api_keys/list)。响应中的每个密钥都包含其 `expires_at` 时间戳（对于没有[过期时间](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-expiration)的密钥为 `null`）和 `principal`，即该密钥所代表的身份（请参阅[密钥类型](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)）。对于个人密钥，`principal` 为 `{"type": "user_actor", "user_id": "user_..."}`；对于服务账户密钥，为 `{"type": "service_account_actor", "service_account_id": "svac_..."}`；对于工作区密钥，为 `null`。每个密钥还有一个 `scope` 对象：对于绑定到单个工作区的密钥为 `{"type": "workspace", "workspace_id": "wrkspc_..."}`，对于可在账户有权访问的任何工作区中使用的密钥为 `{"type": "organization"}`。顶层 `workspace_id` 字段已弃用，对于绑定到默认工作区的密钥和没有工作区作用域的密钥均为 `null`；请使用 `scope` 来区分它们。使用默认工作区的 ID 按 `workspace_id` 筛选列表时，仅返回绑定到默认工作区的密钥；没有工作区作用域的密钥不会在任何 `workspace_id` 筛选条件下返回。

列出工作区中的活动 API 密钥：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/api_keys?limit=10&status=active&workspace_id=wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization:api-keys list \
    --limit 10 \
    --status active \
    --workspace-id wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ
  ```

  ```python Python
  client = anthropic.Anthropic()

  api_keys = client.beta.organization.api_keys.list(
      limit=10, status="active", workspace_id="wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  )

  # 根据需要自动获取更多页面。
  for api_key in api_keys:
      print(f"{api_key.id}: {api_key.name} ({api_key.status})")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const apiKeys = await client.beta.organization.apiKeys.list({
    limit: 10,
    status: "active",
    workspace_id: "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  });

  for await (const apiKey of apiKeys) {
    console.log(`${apiKey.id}: ${apiKey.name} (${apiKey.status})`);
  }
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Organization.ApiKeys;

  AnthropicClient client = new();

  var page = await client.Beta.Organization.ApiKeys.List(new()
  {
      Limit = 10,
      Status = ApiKeyListParamsStatus.Active,
      WorkspaceID = "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  });

  await foreach (var apiKey in page.Paginate())
  {
      Console.WriteLine($"{apiKey.ID}: {apiKey.Name} ({apiKey.Status.Raw()})");
  }
  ```

  ```go Go
  client := anthropic.NewClient()

  apiKeys := client.Beta.Organization.APIKeys.ListAutoPaging(context.Background(), anthropic.BetaOrganizationAPIKeyListParams{
  	Limit:       anthropic.Int(10),
  	Status:      anthropic.BetaOrganizationAPIKeyListParamsStatusActive,
  	WorkspaceID: anthropic.String("wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"),
  })

  for apiKeys.Next() {
  	apiKey := apiKeys.Current()
  	fmt.Printf("%s: %s (%s)\n", apiKey.ID, apiKey.Name, apiKey.Status)
  }
  if err := apiKeys.Err(); err != nil {
  	log.Fatal(err)
  }
  ```

  ```java Java
  import com.anthropic.models.beta.organization.apikeys.ApiKeyListParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = ApiKeyListParams.builder()
          .limit(10)
          .status(ApiKeyListParams.Status.ACTIVE)
          .workspaceId("wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ")
          .build();
      var apiKeys = client.beta().organization().apiKeys().list(params);

      for (var apiKey : apiKeys.autoPager()) {
          IO.println(apiKey.id() + ": " + apiKey.name() + " (" + apiKey.status().asString() + ")");
      }
  }
  ```

  ```php PHP
  use Anthropic\Beta\Organization\APIKeys\APIKeyListParams\Status;
  // ...

  $client = new Client();

  $apiKeys = $client->beta->organization->apiKeys->list(
      limit: 10,
      status: Status::ACTIVE,
      workspaceID: 'wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ',
  );

  foreach ($apiKeys->getItems() as $apiKey) {
      echo "{$apiKey->id}: {$apiKey->name} ({$apiKey->status})\n";
  }
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  api_keys = client.beta.organization.api_keys.list(
    limit: 10,
    status: :active,
    workspace_id: "wrkspc_01JwQvzr7rXLA5AGx3HKfFUJ"
  )

  api_keys.data.each do |api_key|
    puts "#{api_key.id}: #{api_key.name} (#{api_key.status})"
  end
  ```
</CodeGroup>

重命名或停用 API 密钥：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/api_keys/apikey_01Rj2N8SVvo6BePZj99NhmiT" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{
      "status": "inactive",
      "name": "New Key Name"
    }'
  ```

  ```bash CLI
  ant beta:organization:api-keys update \
    --api-key-id apikey_01Rj2N8SVvo6BePZj99NhmiT \
    --status inactive \
    --name "New Key Name"
  ```

  ```python Python
  client = anthropic.Anthropic()

  api_key = client.beta.organization.api_keys.update(
      "apikey_01Rj2N8SVvo6BePZj99NhmiT", status="inactive", name="New Key Name"
  )

  print(f"id: {api_key.id}")
  print(f"name: {api_key.name}")
  print(f"status: {api_key.status}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const apiKey = await client.beta.organization.apiKeys.update(
    "apikey_01Rj2N8SVvo6BePZj99NhmiT",
    {
      status: "inactive",
      name: "New Key Name"
    }
  );

  console.log(`id: ${apiKey.id}`);
  console.log(`name: ${apiKey.name}`);
  console.log(`status: ${apiKey.status}`);
  ```

  ```csharp C#
  using Anthropic.Models.Beta.Organization.ApiKeys;

  AnthropicClient client = new();

  var apiKey = await client.Beta.Organization.ApiKeys.Update(
      "apikey_01Rj2N8SVvo6BePZj99NhmiT",
      new()
      {
          Status = Status.Inactive,
          Name = "New Key Name"
      }
  );

  Console.WriteLine($"id: {apiKey.ID}");
  Console.WriteLine($"name: {apiKey.Name}");
  Console.WriteLine($"status: {apiKey.Status.Raw()}");
  ```

  ```go Go
  client := anthropic.NewClient()

  apiKey, err := client.Beta.Organization.APIKeys.Update(
  	context.Background(),
  	"apikey_01Rj2N8SVvo6BePZj99NhmiT",
  	anthropic.BetaOrganizationAPIKeyUpdateParams{
  		Status: anthropic.BetaOrganizationAPIKeyUpdateParamsStatusInactive,
  		Name:   anthropic.String("New Key Name"),
  	},
  )
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", apiKey.ID)
  fmt.Printf("name: %s\n", apiKey.Name)
  fmt.Printf("status: %s\n", apiKey.Status)
  ```

  ```java Java
  import com.anthropic.models.beta.organization.apikeys.ApiKeyUpdateParams;

  void main() {
      AnthropicClient client = AnthropicOkHttpClient.fromEnv();

      var params = ApiKeyUpdateParams.builder()
          .status(ApiKeyUpdateParams.Status.INACTIVE)
          .name("New Key Name")
          .build();
      var apiKey = client.beta().organization().apiKeys()
          .update("apikey_01Rj2N8SVvo6BePZj99NhmiT", params);

      IO.println("id: " + apiKey.id());
      IO.println("name: " + apiKey.name());
      IO.println("status: " + apiKey.status().asString());
  }
  ```

  ```php PHP
  use Anthropic\Beta\Organization\APIKeys\APIKeyUpdateParams\Status;
  // ...

  $client = new Client();

  $apiKey = $client->beta->organization->apiKeys->update(
      apiKeyID: 'apikey_01Rj2N8SVvo6BePZj99NhmiT',
      status: Status::INACTIVE,
      name: 'New Key Name',
  );

  echo "id: {$apiKey->id}\n";
  echo "name: {$apiKey->name}\n";
  echo "status: {$apiKey->status}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  api_key_id = "apikey_01Rj2N8SVvo6BePZj99NhmiT"
  api_key = client.beta.organization.api_keys.update(
    api_key_id,
    status: :inactive,
    name: "New Key Name"
  )

  puts "id: #{api_key.id}"
  puts "name: #{api_key.name}"
  puts "status: #{api_key.status}"
  ```
</CodeGroup>

### 服务账户

创建和管理服务账户（`svac_...`），即[服务账户密钥](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)和[工作负载身份联合](https://platform.claude.com/docs/zh-CN/manage-claude/workload-identity-federation)令牌所代表的非人类身份。这些端点与联合身份颁发者和联合规则端点一样，需要 `org:admin` OAuth 令牌。请参阅[使用 Admin API 管理 WIF](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api#service-accounts)。

### 联合身份颁发者

注册其令牌可为您的组织声明工作负载身份的 OIDC 身份提供商（`fdis_...`）。请参阅[使用 Admin API 管理 WIF](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api#federation-issuers)。

### 联合规则

管理将颁发者令牌映射到服务账户和作用域的规则（`fdrl_...`）。请参阅[使用 Admin API 管理 WIF](https://platform.claude.com/docs/zh-CN/manage-claude/wif-admin-api#federation-rules)。

## 访问组织信息

`/v1/organizations/me` 端点返回您的凭证所属的组织：

<CodeGroup>
  ```bash cURL
  curl "https://api.anthropic.com/v1/organizations/me" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01"
  ```

  ```bash CLI
  ant beta:organization retrieve
  ```

  ```python Python
  client = anthropic.Anthropic()

  organization = client.beta.organization.retrieve()

  print(f"id: {organization.id}")
  print(f"name: {organization.name}")
  ```

  ```typescript TypeScript
  const client = new Anthropic();

  const organization = await client.beta.organization.retrieve();

  console.log(`id: ${organization.id}`);
  console.log(`name: ${organization.name}`);
  ```

  ```csharp C#
  AnthropicClient client = new();

  var organization = await client.Beta.Organization.Retrieve();

  Console.WriteLine($"id: {organization.ID}");
  Console.WriteLine($"name: {organization.Name}");
  ```

  ```go Go
  client := anthropic.NewClient()

  organization, err := client.Beta.Organization.Get(context.Background())
  if err != nil {
  	log.Fatal(err)
  }

  fmt.Printf("id: %s\n", organization.ID)
  fmt.Printf("name: %s\n", organization.Name)
  ```

  ```java Java
  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

  var organization = client.beta().organization().retrieve();

  IO.println("id: " + organization.id());
  IO.println("name: " + organization.name());
  ```

  ```php PHP
  $client = new Client();

  $organization = $client->beta->organization->retrieve();

  echo "id: {$organization->id}\n";
  echo "name: {$organization->name}\n";
  ```

  ```ruby Ruby
  client = Anthropic::Client.new

  organization = client.beta.organization.retrieve

  puts "id: #{organization.id}"
  puts "name: #{organization.name}"
  ```
</CodeGroup>

```json
{
  "id": "12345678-1234-5678-1234-567812345678",
  "type": "organization",
  "name": "Organization Name"
}
```

有关参数详情和响应架构，请参阅[组织信息 API 参考](https://platform.claude.com/docs/zh-CN/api/admin-api/organization/get-me)。

## 用量和成本报告

使用[用量和成本 API](https://platform.claude.com/docs/zh-CN/manage-claude/usage-cost-api) 跟踪您组织的用量和成本。

## Claude Code 分析

使用 [Claude Code 分析 API](https://platform.claude.com/docs/zh-CN/manage-claude/claude-code-analytics-api) 监控开发者生产力和 Claude Code 采用情况。

## 速率限制

使用[速率限制 API](https://platform.claude.com/docs/zh-CN/manage-claude/rate-limits-api) 读取为您的组织及其工作区配置的速率限制。

## 合规 API

使用[合规 API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api) 检索您组织的审计和活动数据。Admin API 密钥只能读取活动源（Activity Feed）。如需完整访问权限，请参阅[设置合规 API](https://platform.claude.com/docs/zh-CN/manage-claude/compliance-api-access)。

## 最佳实践

* 为工作区和 API 密钥使用有意义的名称和描述
* 处理失败操作产生的错误
* 定期审计成员角色和权限
* 清理未使用的工作区和已过期的邀请
* 监控 API 密钥使用情况，审计每个密钥的 [`expires_at`](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-expiration)，并定期轮换密钥

## 常见问题

<AccordionGroup>
  <Accordion title="使用 Admin API 需要哪些权限？">
    Admin API 接受 Admin API 密钥（以 `sk-ant-admin` 开头）、具有 `org:admin` 作用域的 OAuth bearer 令牌，或未限定于特定工作区的个人密钥或服务账户密钥。只有具有 admin 角色的组织成员才能配置 Admin API 密钥，只有具有 admin、owner 或 primary owner 角色的成员才能获取 `org:admin` 令牌。个人密钥或服务账户密钥拥有与所关联账户相同的权限。请参阅[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api#authentication)。
  </Accordion>

  <Accordion title="我可以通过 Admin API 创建新的 API 密钥吗？">
    不可以。您需要在 Claude Console 中创建 API 密钥。Admin API 只能读取、重命名和更改现有密钥的状态。
  </Accordion>

  <Accordion title="移除用户时 API 密钥会怎样？">
    具体行为取决于[密钥类型](https://platform.claude.com/docs/zh-CN/manage-claude/authentication#key-types)。

    个人密钥在其用户被从组织中移除时停止工作。服务账户密钥在其服务账户被归档时停止工作，但即使创建它们的用户被移除，它们仍会继续工作。工作区 API 密钥会继续工作。在 [Claude Code 工作区](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#claude-code-workspace)中，每个密钥都绑定到创建它的成员，并在该成员被移除时停止工作。
  </Accordion>

  <Accordion title="可以通过 API 移除组织管理员吗？">
    不可以。API 无法移除具有 admin 角色的成员。
  </Accordion>

  <Accordion title="组织邀请的有效期是多久？">
    邀请在 21 天后过期。过期期限不可配置。
  </Accordion>
</AccordionGroup>

有关工作区的具体问题，请参阅[工作区常见问题](https://platform.claude.com/docs/zh-CN/manage-claude/workspaces#faq)。
