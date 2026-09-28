---
title: 为 CMEK 配置 Azure Key Vault
url: https://platform.claude.com/docs/zh-CN/manage-claude/cmek-azure-key-vault
description: 使用 Azure Key Vault 为您的组织提供加密密钥。
---

```bash Configure with the /claude-api skill in Claude Code
claude "/claude-api help me configure a customer-managed encryption key with Azure Key Vault"
```

本指南将引导您将 Azure Key Vault 密钥配置为 Anthropic 组织的[客户管理加密密钥（customer-managed encryption key，CMEK）](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。

<Warning>
  启用 CMEK 是永久性的。如果您的 Key Vault 密钥被删除或禁用，Anthropic 将无法恢复使用该密钥加密的数据。在开始之前，请查看[警告和限制](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。
</Warning>

## 前提条件

* 一个已**启用 RBAC 授权**（`enableRbacAuthorization: true`）且**允许公共网络访问**的 Azure Key Vault。Anthropic 通过公共数据平面端点调用您的保管库；不支持私有端点。
* 保管库上已**启用清除保护**（`enablePurgeProtection: true`）。如果没有它，已删除的密钥可能在软删除保留窗口期间被永久清除，导致您的 CMEK 保护数据不可逆转地丢失。清除保护一旦启用便无法禁用。
* 在保管库中创建密钥以及在其上分配 RBAC 角色的权限。
* 在您的 Entra 租户中创建服务主体的权限（`Application Administrator`、`Cloud Application Administrator` 或等效的自定义角色）。
* 您组织的 Anthropic Admin API 密钥。
* 已安装并通过身份验证的 [`az` CLI](https://learn.microsoft.com/en-us/cli/azure/?view=azure-cli-latest)。
* 在保管库上配置了**诊断设置（Diagnostic Settings）**，以将 `AuditEvent` 日志类别路由到 Log Analytics、存储帐户或事件中心。Azure Key Vault 默认不发出数据平面审计日志（例如 `KeyWrap`、`KeyUnwrap` 和 `KeyGet`），因此如果没有此配置，您将无法获得 Anthropic 密钥操作的审计跟踪。

## Anthropic 应用信息

要让 Anthropic 使用您的加密密钥，您必须配置一个 Anthropic 多租户应用程序 ID 和显示名称。这些值为：

| 字段              | 值                                      |
| --------------- | -------------------------------------- |
| 多租户应用客户端 ID（美国） | `8635ae1a-3e5d-44e8-a4ed-e0f614466f87` |
| 应用显示名称          | `anthropic-cmek-client-us`             |

<Warning>
  仅使用此已发布的客户端 ID 和显示名称。切勿信任通过电子邮件、聊天或任何入门渠道提供的标识符。
</Warning>

## 加密密钥设置

<Steps>
  <Step title="同意 Anthropic 多租户应用程序">
    这会在您的 Entra 租户中为 Anthropic 的 CMEK 客户端应用程序创建一个服务主体。该应用程序不请求任何 Microsoft Graph 权限；它仅作为 Key Vault 数据平面访问的联合目标而存在。

    ```bash
    az ad sp create --id 8635ae1a-3e5d-44e8-a4ed-e0f614466f87
    ```

    从输出中，捕获 `id` 字段。这是服务主体在您租户中的对象 ID，您在分配 RBAC 角色时会用到它。

    ```json
    {
      "appId": "8635ae1a-3e5d-44e8-a4ed-e0f614466f87",
      "displayName": "anthropic-cmek-client-us",
      "id": "<sp-object-id>"
    }
    ```

    如果服务主体已存在于您的租户中（来自先前的尝试或其他集成），`az ad sp create` 会以"already exists"错误退出。请改为获取其对象 ID：

    ```bash
    az ad sp show --id 8635ae1a-3e5d-44e8-a4ed-e0f614466f87 --query id -o tsv
    ```

    此步骤没有门户等效操作。如果您本地未安装 Azure CLI，请从门户顶部导航栏打开 Cloud Shell。命令成功后，您可以在 **Microsoft Entra ID > 企业应用程序**中，通过清除默认的应用程序类型筛选器并搜索 `anthropic-cmek-client-us` 来找到服务主体的对象 ID。

    <Frame caption="在其 Entra 企业应用程序概述中找到服务主体的对象 ID（Object ID）。">
      ![anthropic-cmek-client-us 的 Microsoft Entra 企业应用程序概述，显示其应用程序 ID（Application ID）和对象 ID（Object ID）。](https://platform.claude.com/docs/images/cmek/azure-service-principal.png)
    </Frame>
  </Step>

  <Step title="在您的保管库中创建 RSA 密钥">
    Azure Key Vault 不支持对称密钥包装，因此密钥必须是 RSA（3072 位或更大），并在其允许的操作中包含 `wrapKey` 和 `unwrapKey`。

    `--tags` 选项添加组织标签 `anthropic-org-<ORGANIZATION_UUID>`，其值为 `true`，其中 `<ORGANIZATION_UUID>` 是您的 Anthropic 组织 ID（小写）。此标签是 Anthropic 验证密钥所必需的。

    <Note>
      **查找您的组织 ID：** 在 Claude Console 的 **Settings > Organization** 下复制 **Organization ID** 字段，或在 claude.ai 的 **Organization settings > Organization** 下复制，或从 [Organization Info](https://platform.claude.com/docs/zh-CN/api/beta/organization/retrieve) 端点读取 `id` 字段。使用裸 UUID，而不是带 `org_` 前缀的 ID。
    </Note>

    ```bash
    az keyvault key create \
      --vault-name <VAULT_NAME> \
      --name <KEY_NAME> \
      --kty RSA --size 3072 \
      --ops wrapKey unwrapKey \
      --tags anthropic-org-<ORGANIZATION_UUID>=true
    ```

    对于 HSM 支持的密钥，使用 `--kty RSA-HSM`（需要 Premium-SKU 保管库）。软件保护的 RSA 密钥对于此集成是可接受的。

    从门户中，打开您的 Key Vault，选择 **Keys**，然后选择 **Generate/Import**。将密钥类型设置为 RSA，大小设置为 3072 或更大。要将密钥限制为仅包装和解包，请打开密钥版本，滚动到 **Permitted operations**，并取消选中除 **Wrap Key** 和 **Unwrap Key** 之外的所有内容。

    在 **Create a key** 页面上，还要在 **Tags** 下添加组织标签。

    <Frame caption="创建一个大小为 3072 或更大的 RSA 密钥，并将标签 anthropic-org-<ORGANIZATION_UUID> 设置为 true。">
      ![Azure Key Vault 的 Create a key 页面，密钥类型为 RSA，密钥大小为 3072，anthropic-org 标签已设置为 true。](https://platform.claude.com/docs/images/cmek/azure-create-key-tag.png)
    </Frame>

    <Frame caption="将允许的操作限制为 Wrap Key 和 Unwrap Key。密钥版本显示组织标签。">
      ![Azure Key Vault 密钥版本，带有 1 个标签，Permitted operations 仅限于 Wrap Key 和 Unwrap Key。](https://platform.claude.com/docs/images/cmek/azure-permitted-operations-tag.png)
    </Frame>

    要在多个 Anthropic 组织之间共享一个密钥，请为每个组织添加一个这样的标签。一个密钥版本最多可以携带 15 个标签，包括您自己的标签。

    <Note>
      要将标签添加到您已有的密钥，请在门户中打开密钥的当前版本，选择 **Tags** 旁边的链接，添加标签，然后单击 **Save**。使用 Azure CLI，运行 `az keyvault key set-attributes --vault-name <VAULT_NAME> --name <KEY_NAME> --tags anthropic-org-<ORGANIZATION_UUID>=true`。其 `--tags` 选项会替换该版本的标签，因此还要将该版本已有的每个标签以 `name=value` 的形式放入 `--tags` 中。对于 Managed HSM 中的密钥，使用 `--hsm-name <HSM_NAME>` 而不是 `--vault-name`。
    </Note>
  </Step>

  <Step title="授予 Anthropic 服务主体访问您密钥的权限">
    将 `Key Vault Crypto User` 角色分配给第一步中的服务主体，范围限定为**单个密钥**而不是整个保管库。

    ```bash
    VAULT_ID=$(az keyvault show --name <your-vault-name> --query id -o tsv)

    az role assignment create \
      --role "Key Vault Crypto User" \
      --assignee-object-id <sp-object-id> \
      --assignee-principal-type ServicePrincipal \
      --scope "${VAULT_ID}/keys/<your-key-name>"
    ```

    内置的 `Key Vault Crypto User` 角色在其分配的范围内授予密钥加密操作（加密、解密、包装、解包、签名、验证）以及密钥读取权限。您在上一步中对密钥设置的 `--ops wrapKey unwrapKey` 限制进一步缩小了针对此密钥可以成功执行的操作，因此实际上 Anthropic 只能包装和解包。

    从门户中，打开**密钥**（而不是保管库），选择其 **Access control (IAM)** 选项卡，单击 **Add > Add role assignment**，选择 **Key Vault Crypto User**，并将其分配给 `anthropic-cmek-client-us` 服务主体。

    <Note>
      **专用保管库替代方案：** Microsoft 建议每个应用程序使用专用保管库，并在保管库范围分配角色。如果您预配一个仅包含此 Anthropic CMEK 密钥的保管库，您可以改为在保管库范围分配角色，效果是相同的。当密钥位于共享保管库中时，请将范围限定为单个密钥。
    </Note>

    <Frame caption="将 Key Vault Crypto User 分配给 Anthropic 服务主体，范围限定为密钥。">
      ![Key Vault IAM 角色分配，显示 anthropic-cmek-client-us 已被分配 Key Vault Crypto User 角色。](https://platform.claude.com/docs/images/cmek/azure-role-assignment.png)
    </Frame>
  </Step>

  <Step title="验证您的保管库配置">
    ```bash
    az keyvault show --name <your-vault-name> \
      --query "{rbac:properties.enableRbacAuthorization, purge:properties.enablePurgeProtection, pub:properties.publicNetworkAccess, net:properties.networkAcls.defaultAction, ipRules:properties.networkAcls.ipRules, uri:properties.vaultUri, tenantId:properties.tenantId}"
    ```

    确认：

    * `rbac` 为 `true`。
    * `purge` 为 `true`。如果它为 `false` 或 `null`，请在继续之前在保管库上启用清除保护。如果没有它，软删除的密钥可能在保留窗口期间被永久清除，使您的 CMEK 保护数据无法恢复。
    * `pub` 为 `"Enabled"`。如果它为 `"Disabled"`，Anthropic 无法通过其公共数据平面端点访问保管库，验证将失败。
    * `net` 为 `"Allow"`，或者，如果它为 `"Deny"`，则 `ipRules` 包含 Anthropic 的出口范围（请联系 Anthropic 获取当前列表）。
    * `uri` 是您注册密钥时使用的保管库 URI。
    * `tenantId` 是管理保管库的租户。注册密钥时将此值用作 `tenant_id`，而不是您当前活动订阅的租户（在跨租户设置中两者可能不同）。
  </Step>
</Steps>

## 向 Anthropic 注册密钥

注册密钥的方式取决于您使用的产品。

<Tabs>
  <Tab title="Claude Platform">
    您可以在 Claude Console 中或通过 Admin API 设置密钥，结果相同。

    <Tabs>
      <Tab title="Claude Console">
        <Steps>
          <Step title="向 Anthropic 注册密钥">
            在 Claude Console 中，打开 **Settings > Encryption keys** 并单击 **Add key**。输入显示名称，选择 **Azure Key Vault**，然后单击 **Continue**。填写 **Vault URI**、**Key name** 和 **Tenant ID**，然后单击 **Add**。

            密钥详细信息步骤会显示组织标签。在单击 **Add** 之前，按照[创建步骤](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-azure-key-vault#organization-tag)中的描述将其添加到密钥。
          </Step>

          <Step title="验证密钥">
            在 **Encryption keys** 页面上，单击密钥旁边的 **Verify**。检查通过时会显示 **Connected**。如果失败，会有一条消息给出原因。
          </Step>

          <Step title="将密钥附加到工作区">
            在 Claude Console 中，转到 [Manage > Security](https://platform.claude.com/settings/workspaces/default/security-compliance)，并在侧边栏顶部的工作区选择器中选择工作区。在 **Encryption key** 下，选择密钥，单击 **Save**，然后确认。附加密钥无法撤消。对于已经接收请求的工作区，密钥可能需要[最多一天才能生效](https://platform.claude.com/docs/zh-CN/manage-claude/cmek#how-it-works)。
          </Step>
        </Steps>
      </Tab>

      <Tab title="API">
        <Steps>
          <Step title="向 Anthropic 注册密钥">
            通过 Admin API 创建外部密钥配置。

            <CodeGroup>
              ```bash cURL
              curl -sS "https://api.anthropic.com/v1/organizations/external_keys" \
                -H "x-api-key: $ANTHROPIC_API_KEY" \
                -H "anthropic-version: 2023-06-01" \
                -H "content-type: application/json" \
                -d '{
                  "display_name": "<friendly-name>",
                  "geo": "us",
                  "provider_config": {
                    "type": "azure",
                    "vault_uri": "https://<your-vault-name>.vault.azure.net/",
                    "key_name": "<your-key-name>",
                    "tenant_id": "<your-tenant-id>"
                  }
                }'
              ```

              ```bash CLI
              ant beta:organization:external-keys create <<'YAML'
              display_name: "<friendly-name>"
              geo: us
              provider_config:
                type: azure
                vault_uri: "https://<your-vault-name>.vault.azure.net/"
                key_name: "<your-key-name>"
                tenant_id: "<your-tenant-id>"
              YAML
              ```

              ```python Python
              client = anthropic.Anthropic()

              external_key = client.beta.organization.external_keys.create(
                  display_name="<friendly-name>",
                  geo="us",
                  provider_config={
                      "type": "azure",
                      "vault_uri": "https://<your-vault-name>.vault.azure.net/",
                      "key_name": "<your-key-name>",
                      "tenant_id": "<your-tenant-id>",
                  },
              )

              print(f"id: {external_key.id}")
              print(f"display_name: {external_key.display_name}")
              ```

              ```typescript TypeScript
              const client = new Anthropic();

              const externalKey = await client.beta.organization.externalKeys.create({
                display_name: "<friendly-name>",
                geo: "us",
                provider_config: {
                  type: "azure",
                  vault_uri: "https://<your-vault-name>.vault.azure.net/",
                  key_name: "<your-key-name>",
                  tenant_id: "<your-tenant-id>"
                }
              });

              console.log(`id: ${externalKey.id}`);
              console.log(`display_name: ${externalKey.display_name}`);
              ```

              ```csharp C#
              using Anthropic.Models.Beta.Organization.ExternalKeys;

              AnthropicClient client = new();

              var externalKey = await client.Beta.Organization.ExternalKeys.Create(new()
              {
                  DisplayName = "<friendly-name>",
                  Geo = Geo.Us,
                  ProviderConfig = new BetaAzureExternalKeyConfigParam
                  {
                      VaultUri = "https://<your-vault-name>.vault.azure.net/",
                      KeyName = "<your-key-name>",
                      TenantID = "<your-tenant-id>"
                  }
              });

              Console.WriteLine($"id: {externalKey.ID}");
              Console.WriteLine($"display_name: {externalKey.DisplayName}");
              ```

              ```go Go
              client := anthropic.NewClient()

              externalKey, err := client.Beta.Organization.ExternalKeys.New(context.Background(), anthropic.BetaOrganizationExternalKeyNewParams{
              	DisplayName: anthropic.String("<friendly-name>"),
              	Geo:         anthropic.BetaOrganizationExternalKeyNewParamsGeoUs,
              	ProviderConfig: anthropic.BetaOrganizationExternalKeyNewParamsProviderConfigUnion{
              		OfAzure: &anthropic.BetaAzureExternalKeyConfigParam{
              			VaultURI: "https://<your-vault-name>.vault.azure.net/",
              			KeyName:  "<your-key-name>",
              			TenantID: "<your-tenant-id>",
              		},
              	},
              })
              if err != nil {
              	log.Fatal(err)
              }

              fmt.Printf("id: %s\n", externalKey.ID)
              fmt.Printf("display_name: %s\n", externalKey.DisplayName)
              ```

              ```java Java
              import com.anthropic.models.beta.organization.externalkeys.BetaAzureExternalKeyConfigParam;
              import com.anthropic.models.beta.organization.externalkeys.ExternalKeyCreateParams;

              void main() {
                  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

                  var params = ExternalKeyCreateParams.builder()
                      .displayName("<friendly-name>")
                      .geo(ExternalKeyCreateParams.Geo.US)
                      .providerConfig(BetaAzureExternalKeyConfigParam.builder()
                          .vaultUri("https://<your-vault-name>.vault.azure.net/")
                          .keyName("<your-key-name>")
                          .tenantId("<your-tenant-id>")
                          .build())
                      .build();
                  var externalKey = client.beta().organization().externalKeys().create(params);

                  IO.println("id: " + externalKey.id());
                  IO.println("display_name: " + externalKey.displayName().orElseThrow());
              }
              ```

              ```php PHP
              use Anthropic\Beta\Organization\ExternalKeys\ExternalKeyCreateParams\Geo;
              // ...

              $client = new Client();

              $externalKey = $client->beta->organization->externalKeys->create(
                  displayName: '<friendly-name>',
                  geo: Geo::US,
                  providerConfig: [
                      'type' => 'azure',
                      'vaultURI' => 'https://<your-vault-name>.vault.azure.net/',
                      'keyName' => '<your-key-name>',
                      'tenantID' => '<your-tenant-id>',
                  ],
              );

              echo "id: {$externalKey->id}\n";
              echo "display_name: {$externalKey->displayName}\n";
              ```

              ```ruby Ruby
              client = Anthropic::Client.new

              external_key = client.beta.organization.external_keys.create(
                display_name: "<friendly-name>",
                geo: :us,
                provider_config: {
                  type: :azure,
                  vault_uri: "https://<your-vault-name>.vault.azure.net/",
                  key_name: "<your-key-name>",
                  tenant_id: "<your-tenant-id>"
                }
              )

              puts "id: #{external_key.id}"
              puts "display_name: #{external_key.display_name}"
              ```
            </CodeGroup>

            响应包含外部密钥 ID：

            ```json
            {
              "type": "external_key",
              "id": "ekey_<id>",
              "display_name": "<friendly-name>"
            }
            ```
          </Step>

          <Step title="验证密钥">
            针对您的密钥触发一次加密和解密往返。这确认 Anthropic 可以向您的租户进行身份验证并执行包装和解包操作。

            <CodeGroup>
              ```bash cURL
              curl -sS -X POST "https://api.anthropic.com/v1/organizations/external_keys/ekey_<id>/validate" \
                -H "x-api-key: $ANTHROPIC_API_KEY" \
                -H "anthropic-version: 2023-06-01"
              ```

              ```bash CLI
              ant beta:organization:external-keys validate --external-key-id "ekey_<id>"
              ```

              ```python Python
              client = anthropic.Anthropic()

              validation = client.beta.organization.external_keys.validate("ekey_<id>")

              print(f"status: {validation.status}")
              print(f"error: {validation.error}")
              ```

              ```typescript TypeScript
              const client = new Anthropic();

              const validation = await client.beta.organization.externalKeys.validate("ekey_<id>");

              console.log(`status: ${validation.status}`);
              console.log(`error: ${validation.error}`);
              ```

              ```csharp C#
              AnthropicClient client = new();

              var validation = await client.Beta.Organization.ExternalKeys.Validate("ekey_<id>");

              Console.WriteLine($"status: {validation.Status.Raw()}");
              Console.WriteLine($"error: {validation.Error}");
              ```

              ```go Go
              client := anthropic.NewClient()

              validation, err := client.Beta.Organization.ExternalKeys.Validate(context.Background(), "ekey_<id>")
              if err != nil {
              	log.Fatal(err)
              }

              fmt.Printf("status: %s\n", validation.Status)
              fmt.Printf("error: %s\n", validation.Error)
              ```

              ```java Java
              AnthropicClient client = AnthropicOkHttpClient.fromEnv();

              var validation = client.beta().organization().externalKeys().validate("ekey_<id>");

              IO.println("status: " + validation.status().asString());
              IO.println("error: " + validation.error().orElse(""));
              ```

              ```php PHP
              $client = new Client();

              $validation = $client->beta->organization->externalKeys->validate(
                  externalKeyID: 'ekey_<id>',
              );

              echo "status: {$validation->status}\n";
              echo "error: {$validation->error}\n";
              ```

              ```ruby Ruby
              client = Anthropic::Client.new

              validation = client.beta.organization.external_keys.validate("ekey_<id>")

              puts "status: #{validation.status}"
              puts "error: #{validation.error}"
              ```
            </CodeGroup>

            成功的响应如下所示：

            ```json
            { "type": "external_key_validation", "status": "success", "error": null }
            ```

            如果验证失败，`error` 字段会描述问题。常见原因有：

            * **RBAC 传播延迟：** 角色分配可能需要几分钟才能生效。请等待并重试。
            * **网络 ACL 阻止 Anthropic：** 按照验证步骤中的描述确认公共网络访问和 `ipRules`。
            * **工作负载标识上的条件访问策略：** 如果您的租户有针对服务主体的条件访问策略，请排除 Anthropic 服务主体，或将 Anthropic 的出口范围添加到策略的命名位置。
          </Step>

          <Step title="将密钥附加到工作区">
            密钥验证通过后，在向该工作区发送任何请求之前将其附加到新工作区。对于已经接收请求的工作区，密钥可能需要[最多一天才能生效](https://platform.claude.com/docs/zh-CN/manage-claude/cmek#how-it-works)。

            <CodeGroup>
              ```bash cURL
              curl -sS -X POST "https://api.anthropic.com/v1/organizations/workspaces/<workspace-id>" \
                -H "x-api-key: $ANTHROPIC_API_KEY" \
                -H "anthropic-version: 2023-06-01" \
                -H "content-type: application/json" \
                -d '{
                  "external_key_id": "ekey_<id>"
                }'
              ```

              ```bash CLI
              ant beta:organization:workspaces update \
                --workspace-id "<workspace-id>" \
                --external-key-id "ekey_<id>"
              ```

              ```python Python
              client = anthropic.Anthropic()

              workspace = client.beta.organization.workspaces.update(
                  "<workspace-id>", external_key_id="ekey_<id>"
              )

              print(f"id: {workspace.id}")
              print(f"external_key_id: {workspace.external_key_id}")
              ```

              ```typescript TypeScript
              const client = new Anthropic();

              const workspace = await client.beta.organization.workspaces.update("<workspace-id>", {
                external_key_id: "ekey_<id>"
              });

              console.log(`id: ${workspace.id}`);
              console.log(`external_key_id: ${workspace.external_key_id}`);
              ```

              ```csharp C#
              AnthropicClient client = new();

              var workspace = await client.Beta.Organization.Workspaces.Update("<workspace-id>", new()
              {
                  ExternalKeyID = "ekey_<id>"
              });

              Console.WriteLine($"id: {workspace.ID}");
              Console.WriteLine($"external_key_id: {workspace.ExternalKeyID}");
              ```

              ```go Go
              client := anthropic.NewClient()

              workspace, err := client.Beta.Organization.Workspaces.Update(
              	context.Background(),
              	"<workspace-id>",
              	anthropic.BetaOrganizationWorkspaceUpdateParams{
              		ExternalKeyID: anthropic.String("ekey_<id>"),
              	},
              )
              if err != nil {
              	log.Fatal(err)
              }

              fmt.Printf("id: %s\n", workspace.ID)
              fmt.Printf("external_key_id: %s\n", workspace.ExternalKeyID)
              ```

              ```java Java
              import com.anthropic.models.beta.organization.workspaces.WorkspaceUpdateParams;

              void main() {
                  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

                  var params = WorkspaceUpdateParams.builder()
                      .externalKeyId("ekey_<id>")
                      .build();
                  var workspace = client.beta().organization().workspaces().update("<workspace-id>", params);

                  IO.println("id: " + workspace.id());
                  IO.println("external_key_id: " + workspace.externalKeyId().orElseThrow());
              }
              ```

              ```php PHP
              $client = new Client();

              $workspace = $client->beta->organization->workspaces->update(
                  workspaceID: '<workspace-id>',
                  externalKeyID: 'ekey_<id>',
              );

              echo "id: {$workspace->id}\n";
              echo "external_key_id: {$workspace->externalKeyID}\n";
              ```

              ```ruby Ruby
              client = Anthropic::Client.new

              workspace = client.beta.organization.workspaces.update(
                "<workspace-id>",
                external_key_id: "ekey_<id>"
              )

              puts "id: #{workspace.id}"
              puts "external_key_id: #{workspace.external_key_id}"
              ```
            </CodeGroup>
          </Step>
        </Steps>
      </Tab>
    </Tabs>
  </Tab>

  <Tab title="Claude Enterprise">
    在 [claude.ai > Organization settings > Data and privacy](https://claude.ai/admin-settings/data-privacy-controls) 中，打开 **Encryption keys**，然后单击 **Add key**。选择 **Azure**，输入验证步骤中的保管库 URI、密钥名称和租户 ID，然后单击 **Continue**。Anthropic 通过加密和解密往返来验证密钥。一旦显示为已验证，您的组织从那时起就受到 CMEK 保护。

    在 Claude Enterprise 上，CMEK 适用于整个组织，因此没有单独的工作区附加步骤，并且一个组织只能有一个密钥。
  </Tab>
</Tabs>

## Terraform

对于基础设施即代码部署，相同的步骤映射到 `azurerm` 和 `azuread` 提供程序。
