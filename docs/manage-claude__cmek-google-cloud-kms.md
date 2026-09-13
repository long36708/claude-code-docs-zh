---
title: 为 CMEK 配置 Google Cloud KMS
url: https://platform.claude.com/docs/zh-CN/manage-claude/cmek-google-cloud-kms
description: 使用 Google Cloud KMS 为您的组织提供加密密钥。
---

```bash Configure with the /claude-api skill in Claude Code
claude "/claude-api help me configure a customer-managed encryption key with Google Cloud KMS"
```

本指南将逐步介绍如何将 Google Cloud KMS 密钥配置为您的 Anthropic 组织的 ["customer-managed encryption key"（客户管理的加密密钥），即 CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。

<Warning>
  启用 CMEK 是永久性的。如果您的 KMS 密钥被删除或禁用，Anthropic 将无法恢复使用该密钥加密的数据。在开始之前，请查看[警告和限制](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。
</Warning>

## 前提条件

* 一个已启用结算功能的 Google Cloud 项目。
* 已启用 Cloud KMS API（`cloudkms.googleapis.com`）。
* 拥有创建 KMS 密钥环和密钥以及为其设置 IAM 策略的权限（`roles/cloudkms.admin` 或同等权限）。
* 您组织的 Anthropic Admin API 密钥。
* 已安装并完成身份验证的 [`gcloud` CLI](https://cloud.google.com/cli)。
* 已为项目启用 Cloud KMS **数据访问审核日志**（IAM 和管理 > 审核日志 > Cloud Key Management Service，勾选 `DATA_READ` 和 `DATA_WRITE`）。这些日志默认处于关闭状态；如果不启用，Anthropic 的加密和解密操作将不会在 Cloud Logging 中产生任何条目。

## Anthropic 服务账号电子邮件

要让 Anthropic 使用您的加密密钥，您必须向 Anthropic 的服务账号授予一个可用于加密数据的密钥。Anthropic CMEK 的服务账号电子邮件为：

```text wrap
anthropic-cmek-client-us@gcp-anthropic-cmek-clients.iam.gserviceaccount.com
```

<Warning>
  请仅使用此处公布的服务账号电子邮件。切勿信任通过电子邮件、聊天或任何入门引导渠道提供的标识符。
</Warning>

<Note>
  **网域限定共享：** 如果您的项目隶属于强制执行 `constraints/iam.allowedPolicyMemberDomains` 的 Google Cloud 组织，则以下 IAM 绑定会被拒绝，因为 Anthropic 服务账号不在您的组织内。您需要在项目级别对该约束设置例外，或者将 Anthropic 的 Cloud Identity 客户 ID（格式为 `C0xxxxxxxx`）添加到允许列表中。如有需要，请联系 Anthropic 获取客户 ID。
</Note>

## 加密密钥设置

<Steps>
  <Step title="创建或选择密钥环">
    如果您已有可复用的密钥环，请跳过此步骤。密钥环是区域性的。请选择与您正在配置的 Anthropic 地理区域相匹配的美国单区域位置，例如 `us-east5`。不支持 `us` 和 `global` 等多区域位置。

    ```bash
    gcloud kms keyrings create <your-keyring-name> \
      --project=<your-project-id> \
      --location=<region>
    ```
  </Step>

  <Step title="创建加密密钥">
    创建一个用途为 `ENCRYPT_DECRYPT` 的对称密钥。Anthropic 强烈建议使用 HSM 保护：Cloud KMS HSM 密钥已通过 FIPS 140-2 Level 3 验证，且与软件密钥相比成本差异很小。

    ```bash
    gcloud kms keys create <your-key-name> \
      --project=<your-project-id> \
      --location=<region> \
      --keyring=<your-keyring-name> \
      --purpose=encryption \
      --protection-level=hsm
    ```

    如需改用软件保护，请省略 `--protection-level=hsm`。本指南中的其他内容均无需更改。

    您也可以通过 Google Cloud Console 创建密钥。打开密钥环，点击 **Create key**，选择 **Generated key**，将用途和算法设置为对称加密和解密，并在保护级别下选择 **HSM**。

    <Frame caption="创建一个受 HSM 保护的对称加密/解密（Symmetric encrypt/decrypt）密钥。">
      ![Google Cloud KMS 创建密钥页面，保护级别为 HSM，用途为对称加密/解密。](https://platform.claude.com/docs/images/cmek/gcp-create-key.png)
    </Frame>
  </Step>

  <Step title="授予 Anthropic 的服务账号对密钥的访问权限">
    需要两个密钥级别的 IAM 绑定。两者的作用范围均限定于单个加密密钥，而非整个项目或整个密钥环。

    加密和解密权限，Anthropic 使用该权限来加密和解密保护您工作区数据的数据密钥（"envelope encryption"，即信封加密）：

    ```bash
    gcloud kms keys add-iam-policy-binding <your-key-name> \
      --project=<your-project-id> \
      --location=<region> \
      --keyring=<your-keyring-name> \
      --member="serviceAccount:anthropic-cmek-client-us@gcp-anthropic-cmek-clients.iam.gserviceaccount.com" \
      --role=roles/cloudkms.cryptoKeyEncrypterDecrypter
    ```

    查看者权限，用于 Anthropic 在启动时执行的元数据读取（`cryptoKeys.get`），以验证密钥的用途和算法：

    ```bash
    gcloud kms keys add-iam-policy-binding <your-key-name> \
      --project=<your-project-id> \
      --location=<region> \
      --keyring=<your-keyring-name> \
      --member="serviceAccount:anthropic-cmek-client-us@gcp-anthropic-cmek-clients.iam.gserviceaccount.com" \
      --role=roles/cloudkms.viewer
    ```

    在 Console 中，选择该密钥，打开 **Permissions** 面板，点击 **Grant access**，然后添加服务账号并同时授予 Cloud KMS CryptoKey Encrypter/Decrypter 和 Cloud KMS Viewer 角色。请确保您位于密钥的权限页面，而非密钥环或项目的权限页面，以便授权范围仅限于此密钥。

    <Frame caption="向 Anthropic 服务账号授予这两个角色（Cloud KMS CryptoKey Encrypter/Decrypter 和 Viewer），作用范围限定于该密钥。">
      ![Grant access 对话框，其中 Anthropic 服务账号被分配了 Cloud KMS CryptoKey Encrypter/Decrypter 和 Viewer 角色。](https://platform.claude.com/docs/images/cmek/gcp-grant-access.png)
    </Frame>
  </Step>

  <Step title="记录完整的密钥资源名称">
    您在注册密钥时需要将其传递给 Anthropic。格式为：

    ```text wrap
    projects/<your-project-id>/locations/<region>/keyRings/<your-keyring-name>/cryptoKeys/<your-key-name>
    ```

    使用以下命令获取：

    ```bash
    gcloud kms keys describe <your-key-name> \
      --project=<your-project-id> \
      --location=<region> \
      --keyring=<your-keyring-name> \
      --format="value(name)"
    ```

    在 Console 中，打开密钥的详情页面并点击 **Copy resource name**。

    <Frame caption="从操作菜单中复制密钥的完整资源名称（Copy resource name）。">
      ![Google Cloud 密钥环详情页面，密钥操作菜单中的 Copy resource name 操作被高亮显示。](https://platform.claude.com/docs/images/cmek/gcp-copy-resource-name.png)
    </Frame>
  </Step>
</Steps>

## 向 Anthropic 注册密钥

注册密钥的方式取决于您使用的产品。

<Tabs>
  <Tab title="Claude Platform">
    <Steps>
      <Step title="向 Anthropic 注册密钥">
        通过 Admin API 创建外部密钥配置，使用"加密密钥设置"下"记录完整的密钥资源名称"步骤中获取的资源名称。

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

        响应中包含外部密钥 ID：

        ```json
        {
          "type": "external_key",
          "id": "ekey_<id>",
          "display_name": "<friendly-name>"
        }
        ```
      </Step>

      <Step title="验证密钥">
        针对您的密钥触发一次加密和解密往返操作。

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

        如果验证失败，常见原因包括：

        * **VPC Service Controls：** 如果您的项目中有服务边界保护 Cloud KMS，请将 Anthropic 添加到该边界的访问权限级别中（或将密钥所在项目排除在外），以便 Anthropic 能够访问该密钥。
        * **网域限定共享：** `constraints/iam.allowedPolicyMemberDomains` 组织策略可能会移除 Anthropic 服务账号的绑定（请参阅前文的说明）。请使用 `gcloud kms keys get-iam-policy <your-key-name> --project=<your-project-id> --location=<region> --keyring=<your-keyring-name>` 确认绑定是否存在。
        * **密钥版本已禁用或已销毁：** 请确认密钥的主版本已启用，且未被禁用、未计划销毁或未被销毁。
      </Step>

      <Step title="将密钥附加到工作区">
        密钥验证通过后，请在向新工作区发送任何请求之前将其附加到该工作区。对于已在接收请求的工作区，密钥可能需要[最多一天才能生效](https://platform.claude.com/docs/zh-CN/manage-claude/cmek#how-it-works)。

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

  <Tab title="Claude Enterprise">
    在 [claude.ai > Organization settings > Data and privacy](https://claude.ai/admin-settings/data-privacy-controls) 中，打开 **Encryption keys**，然后点击 **Add key**。选择 **Google Cloud**，粘贴上一步中获取的完整密钥资源名称，然后点击 **Continue**。Anthropic 会通过一次加密和解密往返操作来验证密钥。一旦显示为已验证，您的组织从此刻起即受 CMEK 保护。

    在 Claude Enterprise 上，CMEK 适用于整个组织，因此没有单独的工作区附加步骤，并且一个组织只能拥有一个密钥。
  </Tab>
</Tabs>

## Terraform

对于基础设施即代码（infrastructure-as-code）部署，相同的步骤对应于 `google` provider 中的 `google_kms_key_ring`、`google_kms_crypto_key` 和 `google_kms_crypto_key_iam_member` 资源。
