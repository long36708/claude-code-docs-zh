---
title: 为 CMEK 配置 Google Cloud KMS
url: https://platform.claude.com/docs/zh-CN/manage-claude/cmek-google-cloud-kms
description: 使用 Google Cloud KMS 为您的组织提供加密密钥。
---

```bash Configure with the /claude-api skill in Claude Code
claude "/claude-api help me configure a customer-managed encryption key with Google Cloud KMS"
```

本指南将引导您将 Google Cloud KMS 密钥配置为您的 Anthropic 组织的[客户管理加密密钥（customer-managed encryption key，CMEK）](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。

<Warning>
  启用 CMEK 是永久性的。如果您的 KMS 密钥被删除或禁用，Anthropic 无法恢复使用该密钥加密的数据。在开始之前，请查看[警告和限制](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。
</Warning>

## 前提条件

* 一个已启用计费的 Google Cloud 项目。
* 已启用 Cloud KMS API（`cloudkms.googleapis.com`）。
* 创建 KMS 密钥环和密钥以及对其设置 IAM 策略的权限（`roles/cloudkms.admin` 或等效权限）。
* 您组织的 Anthropic Admin API 密钥。
* 已安装并通过身份验证的 [`gcloud` CLI](https://cloud.google.com/cli)。
* 为项目启用 Cloud KMS **数据访问审计日志**（IAM & Admin > Audit Logs > Cloud Key Management Service，启用 `DATA_READ` 和 `DATA_WRITE`）。这些默认是关闭的；如果没有它们，Anthropic 的加密和解密操作不会在 Cloud Logging 中产生任何条目。

## Anthropic 服务账户电子邮件

要让 Anthropic 使用您的加密密钥，您必须为 Anthropic 的服务账户提供一个可用于加密数据的密钥。Anthropic CMEK 的服务账户电子邮件为：

```text wrap
anthropic-cmek-client-us@gcp-anthropic-cmek-clients.iam.gserviceaccount.com
```

<Warning>
  仅使用此已发布的服务账户电子邮件。切勿信任通过电子邮件、聊天或任何入门渠道提供的标识符。
</Warning>

<Note>
  **域限制共享：** 如果您的项目位于强制执行 `constraints/iam.allowedPolicyMemberDomains` 的 Google Cloud 组织下，则以下 IAM 绑定会被拒绝，因为 Anthropic 服务账户位于您的组织之外。您需要对该约束进行项目级别的豁免，或者将 Anthropic 的 Cloud Identity 客户 ID（格式为 `C0xxxxxxxx`）添加到允许列表中。如有需要，请联系 Anthropic 获取客户 ID。
</Note>

## 加密密钥设置

<Steps>
  <Step title="创建或选择密钥环">
    如果您已有可重用的密钥环，请跳过此步骤。密钥环是区域性的。选择一个与您正在配置的 Anthropic 地理位置匹配的单区域美国位置，例如 `us-east5`。不支持 `us` 和 `global` 等多区域位置。

    ```bash
    gcloud kms keyrings create <your-keyring-name> \
      --project=<your-project-id> \
      --location=<region>
    ```
  </Step>

  <Step title="创建加密密钥">
    创建一个具有 `ENCRYPT_DECRYPT` 用途的对称密钥。Anthropic 强烈建议使用 HSM 保护：Cloud KMS HSM 密钥已通过 FIPS 140-2 Level 3 验证，并且相对于软件密钥的成本差异很小。

    `--labels` 选项添加组织标签 `anthropic-org-<ORGANIZATION_UUID>`，其值为 `true`，其中 `<ORGANIZATION_UUID>` 是您的 Anthropic 组织 ID（小写）。Anthropic 需要此标签来验证密钥。

    <Note>
      **查找您的组织 ID：** 在 Claude Console 的 **Settings > Organization** 下复制 **Organization ID** 字段，或在 claude.ai 的 **Organization settings > Organization** 下复制，或从 [Organization Info](https://platform.claude.com/docs/zh-CN/api/beta/organization/retrieve) 端点读取 `id` 字段。使用裸 UUID，而不是带 `org_` 前缀的 ID。
    </Note>

    ```bash
    gcloud kms keys create <KEY_NAME> \
      --project=<PROJECT_ID> \
      --location=<REGION> \
      --keyring=<KEYRING_NAME> \
      --purpose=encryption \
      --protection-level=hsm \
      --labels=anthropic-org-<ORGANIZATION_UUID>=true
    ```

    如果改用软件保护，请省略 `--protection-level=hsm`。本指南中的其他内容不变。

    您也可以从 Google Cloud Console 创建密钥。打开密钥环，点击 **Create key**，选择 **Generated key**，将用途和算法设置为对称加密和解密，并在保护级别下选择 **HSM**。

    <Frame caption="创建一个带有组织标签的 HSM 保护对称加密/解密密钥。">
      ![Google Cloud KMS 的 Create key 页面，已选择 HSM 保护和对称加密/解密，anthropic-org 标签已设置为 true。](https://platform.claude.com/docs/images/cmek/gcp-create-key-label.png)
    </Frame>

    要在多个 Anthropic 组织之间共享一个密钥，请为每个组织添加一个这样的标签。一个密钥最多可以携带 64 个标签，包括您自己的标签。

    <Note>
      要将标签添加到尚未拥有它的密钥，请运行 `gcloud kms keys update <KEY_NAME> --project=<PROJECT_ID> --location=<REGION> --keyring=<KEYRING_NAME> --update-labels=anthropic-org-<ORGANIZATION_UUID>=true`。它会将标签与密钥已有的任何标签合并。
    </Note>
  </Step>

  <Step title="授予 Anthropic 服务账户对密钥的访问权限">
    需要两个密钥级别的 IAM 绑定。两者都限定于单个加密密钥，而不是项目范围或密钥环范围。

    加密和解密，Anthropic 用它来加密和解密保护您工作区数据的数据密钥（信封加密）：

    ```bash
    gcloud kms keys add-iam-policy-binding <your-key-name> \
      --project=<your-project-id> \
      --location=<region> \
      --keyring=<your-keyring-name> \
      --member="serviceAccount:anthropic-cmek-client-us@gcp-anthropic-cmek-clients.iam.gserviceaccount.com" \
      --role=roles/cloudkms.cryptoKeyEncrypterDecrypter
    ```

    查看者，用于 Anthropic 在启动时执行的元数据读取（`cryptoKeys.get`），以验证密钥的用途和算法：

    ```bash
    gcloud kms keys add-iam-policy-binding <your-key-name> \
      --project=<your-project-id> \
      --location=<region> \
      --keyring=<your-keyring-name> \
      --member="serviceAccount:anthropic-cmek-client-us@gcp-anthropic-cmek-clients.iam.gserviceaccount.com" \
      --role=roles/cloudkms.viewer
    ```

    在 Console 中，选择密钥，打开 **Permissions** 面板，点击 **Grant access**，并为服务账户添加 Cloud KMS CryptoKey Encrypter/Decrypter 和 Cloud KMS Viewer 两个角色。确保您位于密钥的权限页面，而不是密钥环或项目，以便授权仅限定于此密钥。

    <Frame caption="为 Anthropic 服务账户授予两个角色，限定于该密钥。">
      ![Grant access 对话框，其中为 Anthropic 服务账户分配了 Cloud KMS CryptoKey Encrypter/Decrypter 和 Viewer 角色。](https://platform.claude.com/docs/images/cmek/gcp-grant-access.png)
    </Frame>
  </Step>

  <Step title="记下完整的密钥资源名称">
    当您注册密钥时，需要将其传递给 Anthropic。格式为：

    ```text wrap
    projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>
    ```

    使用以下命令检索它：

    ```bash
    gcloud kms keys describe <your-key-name> \
      --project=<your-project-id> \
      --location=<region> \
      --keyring=<your-keyring-name> \
      --format="value(name)"
    ```

    在 Console 中，打开密钥的详情页面并点击 **Copy resource name**。

    <Frame caption="从操作菜单复制密钥的完整资源名称。">
      ![Google Cloud 密钥环详情页面，密钥的操作菜单中突出显示了 Copy resource name 操作。](https://platform.claude.com/docs/images/cmek/gcp-copy-resource-name.png)
    </Frame>
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
            在 Claude Console 中，打开 **Settings > Encryption keys** 并点击 **Add key**。输入显示名称，选择 **Google Cloud KMS**，然后点击 **Continue**。将完整的密钥资源名称粘贴到 **Key resource name** 中，然后点击 **Add**。

            密钥详情步骤会显示组织标签。在点击 **Add** 之前，按照[创建步骤](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-google-cloud-kms#organization-label)所述将其添加到密钥。
          </Step>

          <Step title="验证密钥">
            在 **Encryption keys** 页面上，点击密钥旁边的 **Verify**。检查通过时会显示 **Connected**。如果失败，会有一条消息给出原因。
          </Step>

          <Step title="将密钥附加到工作区">
            在 Claude Console 中，转到 [Manage > Security](https://platform.claude.com/settings/workspaces/default/security-compliance)，并在侧边栏顶部的工作区选择器中选择工作区。在 **Encryption key** 下，选择密钥，点击 **Save**，然后确认。附加密钥无法撤销。对于已经接收请求的工作区，密钥可能需要[最多一天才能生效](https://platform.claude.com/docs/zh-CN/manage-claude/cmek#how-it-works)。
          </Step>
        </Steps>
      </Tab>

      <Tab title="API">
        <Steps>
          <Step title="向 Anthropic 注册密钥">
            通过 Admin API 创建外部密钥配置，使用加密密钥设置下"记下完整的密钥资源名称"步骤中的资源名称。

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
                    "type": "gcp",
                    "key_name": "projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>"
                  }
                }'
              ```

              ```bash CLI
              ant beta:organization:external-keys create <<'YAML'
              display_name: "<friendly-name>"
              geo: us
              provider_config:
                type: gcp
                key_name: "projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>"
              YAML
              ```

              ```python Python
              client = anthropic.Anthropic()

              external_key = client.beta.organization.external_keys.create(
                  display_name="<friendly-name>",
                  geo="us",
                  provider_config={
                      "type": "gcp",
                      "key_name": "projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>",
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
                  type: "gcp",
                  key_name:
                    "projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>"
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
                  ProviderConfig = new BetaGcpExternalKeyConfig
                  {
                      KeyName = "projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>"
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
              		OfGCP: &anthropic.BetaGCPExternalKeyConfigParam{
              			KeyName: "projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>",
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
              import com.anthropic.models.beta.organization.externalkeys.ExternalKeyCreateParams;

              void main() {
                  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

                  var params = ExternalKeyCreateParams.builder()
                      .displayName("<friendly-name>")
                      .geo(ExternalKeyCreateParams.Geo.US)
                      .gcpProviderConfig("projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>")
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
                      'type' => 'gcp',
                      'keyName' => 'projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>',
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
                  type: :gcp,
                  key_name: "projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>"
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
            针对您的密钥触发一次加密和解密往返。

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

              external_key_id = "ekey_<id>"
              validation = client.beta.organization.external_keys.validate(external_key_id)

              puts "status: #{validation.status}"
              puts "error: #{validation.error}"
              ```
            </CodeGroup>

            成功的响应如下所示：

            ```json
            { "type": "external_key_validation", "status": "success", "error": null }
            ```

            如果验证失败，常见原因有：

            * **VPC Service Controls：** 如果服务边界保护了您项目中的 Cloud KMS，请将 Anthropic 添加到边界上的访问级别（或排除密钥的项目），以便 Anthropic 可以访问密钥。
            * **域限制共享：** `constraints/iam.allowedPolicyMemberDomains` 组织策略可能会剥离 Anthropic 服务账户绑定（参见前面的说明）。使用 `gcloud kms keys get-iam-policy <your-key-name> --project=<your-project-id> --location=<region> --keyring=<your-keyring-name>` 确认绑定存在。
            * **已禁用或已销毁的密钥版本：** 确认密钥的主版本已启用，而不是已禁用、计划销毁或已销毁。
          </Step>

          <Step title="将密钥附加到工作区">
            密钥验证通过后，在向该工作区发送任何请求之前，将其附加到新工作区。对于已经接收请求的工作区，密钥可能需要[最多一天才能生效](https://platform.claude.com/docs/zh-CN/manage-claude/cmek#how-it-works)。

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

              workspace_id = "<workspace-id>"
              workspace = client.beta.organization.workspaces.update(
                workspace_id,
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
    在 [claude.ai > Organization settings > Data and privacy](https://claude.ai/admin-settings/data-privacy-controls) 中，打开 **Encryption keys**，然后点击 **Add key**。选择 **Google Cloud**，粘贴上一步中的完整密钥资源名称，然后点击 **Continue**。Anthropic 通过加密和解密往返来验证密钥。一旦显示为已验证，您的组织从那时起就受到 CMEK 保护。

    在 Claude Enterprise 上，CMEK 适用于整个组织，因此没有单独的工作区附加步骤，并且一个组织只能有一个密钥。
  </Tab>
</Tabs>

## Terraform

对于基础设施即代码部署，相同的步骤映射到 `google` 提供程序的 `google_kms_key_ring`、`google_kms_crypto_key` 和 `google_kms_crypto_key_iam_member` 资源。
