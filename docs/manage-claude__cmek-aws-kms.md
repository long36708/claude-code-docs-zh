---
title: 为 CMEK 配置 AWS KMS
url: https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms
description: 使用 AWS KMS 为您的组织提供加密密钥。
---

```bash Configure with the /claude-api skill in Claude Code
claude "/claude-api help me configure a customer-managed encryption key with AWS KMS"
```

本指南将引导您将 [AWS KMS](https://aws.amazon.com/kms/) 密钥配置为 Anthropic 组织的[客户管理加密密钥（customer-managed encryption key，CMEK）](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。

<Warning>
  启用 CMEK 是永久性的。如果您的 KMS 密钥被删除或禁用，Anthropic 将无法恢复使用该密钥加密的数据。在开始之前，请查看[警告和限制](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。
</Warning>

<Note>
  **Claude Platform on AWS：** 在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上，您的密钥策略授予访问权限的是 AWS 服务主体，而不是 Anthropic 的 IAM 角色，没有单独的验证步骤，并且您在 Claude Console 中注册和附加密钥。请遵循本页上的 [在 Claude Platform on AWS 上设置 CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#claude-platform-on-aws)，而不是接下来几节中的步骤。
</Note>

## 前提条件

* 一个具有创建 KMS 密钥和设置密钥策略权限（`kms:CreateKey` 和 `kms:PutKeyPolicy`）的 AWS 账户。
* 您组织的 Anthropic Admin API 密钥。
* 已安装并通过身份验证的 [AWS CLI](https://aws.amazon.com/cli/)。

## Anthropic 的 Amazon 资源名称（ARN）

要让 Anthropic 使用您的加密密钥，您必须为 Anthropic 的 IAM 角色提供一个可用于加密数据的 KMS 密钥。Anthropic CMEK 的 ARN 为：

```text wrap
arn:aws:iam::915198916910:role/anthropic-cmek-client-us
```

<Warning>
  仅使用此已发布的 ARN。切勿信任通过电子邮件、聊天或任何入门渠道提供的标识符。
</Warning>

## 加密密钥设置

<Steps>
  <Step title="使用跨账户密钥策略创建 KMS 密钥">
    <Note>
      **Claude Platform on AWS：** 跳过此步骤。您的密钥策略授予访问权限的是 AWS 服务主体，并且没有组织条件。[在 Claude Platform on AWS 上设置 CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#claude-platform-on-aws) 提供了该策略。
    </Note>

    密钥策略授予 Anthropic 的 IAM 角色跨账户访问权限。需要三条语句：

    1. **账户根管理员：** 标准的 KMS 模式。您的账户保留完全的管理控制权。
    2. **Anthropic 加密和解密：** `kms:Encrypt` 和 `kms:Decrypt` 操作，Anthropic 使用它们来加密和解密保护您工作区数据的数据密钥（信封加密）。
    3. **Anthropic 描述：** Anthropic 在启动时执行的元数据读取。它被单独授予，因为 `DescribeKey` 没有 `EncryptionContext` 参数，因此对此操作的 `EncryptionContext` 条件将始终拒绝。

    要查找[您的 AWS 账户 ID](https://docs.aws.amazon.com/IAM/latest/UserGuide/console-account-id.html)，请运行 `aws sts get-caller-identity --query Account --output text`。

    在策略中，将 `<AWS_ACCOUNT_ID>` 替换为您的 AWS 账户 ID，将 `<ORGANIZATION_UUID>` 替换为您的组织 ID。`kms:EncryptionContext:anthropic:org_uuid` 上的 `StringEquals` 条件将密钥绑定到您的 Anthropic 组织，验证会拒绝没有该条件的密钥。要在多个 Anthropic 组织之间共享一个密钥，请在条件值中列出每个组织 ID。

    <Note>
      **查找您的组织 ID：** 在 Claude Console 中的 **Settings > Organization** 下复制 **Organization ID** 字段，或在 claude.ai 中的 **Organization settings > Organization** 下复制，或从 [Organization Info](https://platform.claude.com/docs/zh-CN/api/beta/organization/retrieve) 端点读取 `id` 字段。使用裸 UUID，而不是带 `org_` 前缀的 ID。
    </Note>

    将策略保存为 `key-policy.json`。要改为在 AWS Console 中创建密钥，请将策略粘贴到那里，如本步骤后面所述。

    ```json key-policy.json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "AccountRootAdmin",
          "Effect": "Allow",
          "Principal": {
            "AWS": "arn:aws:iam::<AWS_ACCOUNT_ID>:root"
          },
          "Action": "kms:*",
          "Resource": "*"
        },
        {
          "Sid": "AllowAnthropicCMEKCrypto",
          "Effect": "Allow",
          "Principal": {
            "AWS": "arn:aws:iam::915198916910:role/anthropic-cmek-client-us"
          },
          "Action": ["kms:Encrypt", "kms:Decrypt"],
          "Resource": "*",
          "Condition": {
            "StringEquals": {
              "kms:EncryptionContext:anthropic:org_uuid": ["<ORGANIZATION_UUID>"]
            }
          }
        },
        {
          "Sid": "AllowAnthropicCMEKDescribe",
          "Effect": "Allow",
          "Principal": {
            "AWS": "arn:aws:iam::915198916910:role/anthropic-cmek-client-us"
          },
          "Action": "kms:DescribeKey",
          "Resource": "*"
        }
      ]
    }
    ```

    <Note>
      **可选：** 要将密钥限制到您的部分工作区，请使用此 `AllowAnthropicCMEKCrypto` 语句代替前面策略 JSON 中的语句，为每个工作区提供一个分区 ID。在将密钥附加到工作区之前，先添加该工作区的分区 ID。对于新工作区，先在不带密钥的情况下创建它，添加其分区 ID，然后附加密钥。

      ```json
      {
        "Sid": "AllowAnthropicCMEKCrypto",
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::915198916910:role/anthropic-cmek-client-us"
        },
        "Action": ["kms:Encrypt", "kms:Decrypt"],
        "Resource": "*",
        "Condition": {
          "StringEquals": {
            "kms:EncryptionContext:anthropic:org_uuid": ["<ORGANIZATION_UUID>"]
          },
          "StringEqualsIfExists": {
            "kms:EncryptionContext:anthropic:compartment_uuid": ["<COMPARTMENT_UUID>"]
          }
        }
      }
      ```
    </Note>

    ```bash
    aws kms create-key \
      --region <REGION> \
      --description "Anthropic CMEK" \
      --key-usage ENCRYPT_DECRYPT \
      --policy file://key-policy.json
    ```

    从输出中捕获 `KeyMetadata.Arn`。在下一步注册密钥时您需要它。

    <Warning>
      如果密钥已为 CMEK 配置并保护现有数据，则除了前面策略中的三条语句外，您还必须添加一条允许 Anthropic 解密该数据的语句。在其条件中，列出该密钥当前或曾经附加到的每个工作区的分区 ID。

      ```json
      {
        "Sid": "AllowAnthropicCMEKDecryptExistingData",
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::915198916910:role/anthropic-cmek-client-us"
        },
        "Action": "kms:Decrypt",
        "Resource": "*",
        "Condition": {
          "StringEquals": {
            "kms:EncryptionContext:anthropic:compartment_uuid": ["<COMPARTMENT_UUID>"]
          }
        }
      }
      ```
    </Warning>

    当您验证密钥或将其附加到工作区时，Anthropic 会验证密钥。每次验证都会向 CloudTrail 添加四个访问被拒绝错误。这些是预期的。如果您需要将它们过滤掉，请对以下所有三个值进行过滤。仅第一个值是不够的，因为任何调用者都可以设置它：

    * `requestParameters.encryptionContext.associatedData`：`Y21lay12YWxpZGF0aW9u`
    * `userIdentity.accountId`：`915198916910`
    * `resources.ARN`：`arn:aws:kms:<REGION>:<AWS_ACCOUNT_ID>:key/<KEY_ID>`

    <Note>
      **查找您的分区 ID：** 请参阅 **Register the key with Anthropic** 下的 **Claude Platform** 选项卡。
    </Note>

    您也可以从 AWS Console 创建密钥。选择具有加密和解密密钥用途的对称密钥、单区域密钥和 KMS 密钥材料来源。创建密钥向导在其 **Review** 步骤提交密钥策略：如果您在那里的密钥用途权限下添加 Anthropic 的账户 ID `915198916910`，生成的策略会授予整个 Anthropic 账户更广泛的操作（例如 `kms:ReEncrypt*` 和 `kms:GenerateDataKey*`），且没有 `EncryptionContext` 条件，验证会拒绝它。为避免留下权限过大的密钥，请仅使用管理权限完成向导，然后打开密钥的 **Key policy** 选项卡，并将 JSON 替换为本步骤前面显示的 `key-policy.json` 策略。

    <Frame caption="配置密钥（Configure key）：对称、加密和解密、单区域密钥。">
      ![AWS KMS 创建密钥向导的配置密钥（Configure key）步骤，已选择对称密钥类型、加密和解密密钥用途以及单区域密钥。](https://platform.claude.com/docs/images/cmek/aws-configure-key.png)
    </Frame>

    <Frame caption="为密钥添加别名和描述。">
      ![AWS KMS 添加标签（Add labels）步骤，别名为 anthropic-cmek，描述为 Anthropic CMEK。](https://platform.claude.com/docs/images/cmek/aws-add-labels.png)
    </Frame>

    <Frame caption="定义密钥管理权限（可选）。您的账户保留完全的管理控制权。">
      ![AWS KMS 定义密钥管理权限（Define key administrative permissions）步骤，列出了可以管理该密钥的 IAM 角色。](https://platform.claude.com/docs/images/cmek/aws-admin-permissions.png)
    </Frame>

    <Frame caption="不要在此处添加 Anthropic 的账户 ID。此向导步骤会生成权限过大的策略。将用途权限留空，并在创建后编辑 Key policy JSON（请参阅前面的密钥策略）。">
      ![AWS KMS 定义密钥用途权限（Define key usage permissions）步骤，在其他 AWS 账户（Other AWS accounts）下输入了 Anthropic 的账户 ID。](https://platform.claude.com/docs/images/cmek/aws-usage-permissions.png)
    </Frame>
  </Step>
</Steps>

## 向 Anthropic 注册密钥

您如何注册密钥取决于您使用的产品。

<Tabs>
  <Tab title="Claude Platform">
    <Note>
      **Claude Platform on AWS：** 主体、密钥策略和注册流程不同，并且没有单独的验证步骤。请遵循 [在 Claude Platform on AWS 上设置 CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#claude-platform-on-aws)，而不是此选项卡。
    </Note>

    <Note>
      **查找您的分区 ID：** 每个工作区都有一个分区 ID，用于限定其 CMEK 数据的范围。要在 Claude Console 中查找它，请转到 [Manage > Security](https://platform.claude.com/settings/workspaces/default/security-compliance)，并在侧边栏顶部的工作区选择器中选择工作区。该 ID 位于 **Encryption key** 下的 **Compartment ID** 字段中。您也可以读取 [Get Workspace](https://platform.claude.com/docs/zh-CN/api/beta/organization/workspaces/retrieve) 端点返回的 `compartment_id` 字段。
    </Note>

    您可以在 Claude Console 中或通过 Admin API 设置密钥，结果相同。

    <Tabs>
      <Tab title="Claude Console">
        <Steps>
          <Step title="向 Anthropic 注册密钥">
            在 Claude Console 中，打开 **Settings > Encryption keys** 并单击 **Add key**。输入显示名称，选择 **AWS KMS**，然后单击 **Continue**。将密钥 ARN 粘贴到 **KMS key ARN** 中，然后单击 **Add**。

            密钥详细信息步骤会显示您的组织 ID。在单击 **Add** 之前，将其添加到[密钥策略](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#key-policy)中。
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
                    "type": "aws",
                    "kms_arn": "<key-arn-from-create-key-step>"
                  }
                }'
              ```

              ```bash CLI
              ant beta:organization:external-keys create <<'YAML'
              display_name: "<friendly-name>"
              geo: us
              provider_config:
                type: aws
                kms_arn: "<key-arn-from-create-key-step>"
              YAML
              ```

              ```python Python
              client = anthropic.Anthropic()

              external_key = client.beta.organization.external_keys.create(
                  display_name="<friendly-name>",
                  geo="us",
                  provider_config={"type": "aws", "kms_arn": "<key-arn-from-create-key-step>"},
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
                  type: "aws",
                  kms_arn: "<key-arn-from-create-key-step>"
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
                  ProviderConfig = new BetaAwsExternalKeyConfig
                  {
                      KmsArn = "<key-arn-from-create-key-step>"
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
              		OfAWS: &anthropic.BetaAWSExternalKeyConfigParam{
              			KMSARN: "<key-arn-from-create-key-step>",
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
              import com.anthropic.models.beta.organization.externalkeys.BetaAwsExternalKeyConfig;
              import com.anthropic.models.beta.organization.externalkeys.ExternalKeyCreateParams;

              void main() {
                  AnthropicClient client = AnthropicOkHttpClient.fromEnv();

                  var params = ExternalKeyCreateParams.builder()
                      .displayName("<friendly-name>")
                      .geo(ExternalKeyCreateParams.Geo.US)
                      .providerConfig(BetaAwsExternalKeyConfig.builder()
                          .kmsArn("<key-arn-from-create-key-step>")
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
                      'type' => 'aws',
                      'kmsARN' => '<key-arn-from-create-key-step>',
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
                  type: :aws,
                  kms_arn: "<key-arn-from-create-key-step>"
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

            * **加密上下文不匹配：** 如果策略有 `kms:EncryptionContext:anthropic:compartment_uuid` 条件，请确保它列出了密钥附加到的每个工作区的分区 ID。验证会发送它检查的工作区的分区 ID。对于一个较旧的密钥，如果其分区语句仍允许 `kms:Encrypt`，则在它未附加时会使用全零值（`00000000-0000-0000-0000-000000000000`）进行验证，因此请在其列表中保留该值。
            * **资源控制策略（RCP）：** 如果您的 AWS 组织有一个 RCP，当 `aws:PrincipalOrgID` 与您的组织不匹配时拒绝 KMS 操作，它会阻止 Anthropic 的跨账户角色。RCP 需要为此密钥或 Anthropic 的角色 ARN 设置例外。服务控制策略在此不适用，因为它们不会对通过基于资源的策略调用的外部主体进行评估。
            * **通过 IAM 而非密钥策略授予访问权限：** 跨账户 KMS 访问必须在密钥策略本身中授予，而不是通过您账户中的 IAM 策略。使用 `aws kms get-key-policy --key-id <id> --policy-name default` 进行检查。
            * **区域不匹配：** 确认密钥的区域是 Anthropic 为您配置的地理层级所运营的区域之一。
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
    在 [claude.ai > Organization settings > Data and privacy](https://claude.ai/admin-settings/data-privacy-controls) 中，打开 **Encryption keys**，然后单击 **Add key**。选择 **AWS** 并单击 **Continue**，然后粘贴上一步中的密钥 ARN 并单击 **Add**。Anthropic 通过加密和解密往返来验证密钥。一旦显示为已验证，您的组织从那时起就受到 CMEK 保护。

    此流程的密钥详细信息步骤会显示您的 **Organization ID for the key policy** 以及一个复制按钮。将该值替换密钥策略中的 `<ORGANIZATION_UUID>`。您可以在创建密钥之前打开该流程以复制 ID。

    在 Claude Enterprise 上，CMEK 适用于整个组织，因此没有单独的工作区附加步骤，并且一个组织只能有一个密钥。
  </Tab>
</Tabs>

## 在 Claude Platform on AWS 上设置 CMEK

在 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 上，CMEK 仅使用 AWS KMS 密钥，并且设置在以下方面与前面几节不同：

* **主体：** 您的密钥策略授予访问权限的是 AWS 服务主体 `aws-external-anthropic.amazonaws.com`。不使用 Anthropic 的 IAM 角色和账户 ID，因此 [Anthropic 的 ARN](https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms#amazon-resource-name-arn-for-anthropic) 不适用。
* **密钥要求：** 密钥必须是具有加密和解密用途的对称 KMS 密钥、单区域，并且与您附加它的工作区位于同一 AWS 账户和区域。不支持跨账户密钥：密钥必须位于托管您组织的 AWS 账户中。注册密钥时会拒绝多区域密钥（以 `mrk-` 开头的密钥 ID）和别名 ARN；请使用密钥 ARN。
* **没有单独的验证步骤：** 除了注册时对密钥 ARN 的这些检查外，密钥在您将其附加到工作区时进行验证。附加调用会以该工作区的分区 ID 作为加密上下文，针对密钥执行一次加密/解密往返，因此密钥策略问题会在附加时而不是注册时显现。因此 `EncryptionContext` 条件不需要全零条目。
* **您在何处管理密钥：** 在 Claude Console 中注册和附加密钥，通过 AWS 以 Admin 角色登录。外部密钥端点在 Claude Platform on AWS 上也可用，通过 [IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#encryption-keys)授权；在那里，密钥由其 KMS 密钥 ARN 而不是 `ekey_` ID 标识。

<Warning>
  仅使用此已发布的服务主体名称。切勿信任通过电子邮件、聊天或任何入门渠道提供的标识符。
</Warning>

### 前提条件

* 托管您的 Claude Platform on AWS 组织的 AWS 账户，具有创建 KMS 密钥和设置密钥策略的权限（`kms:CreateKey` 和 `kms:PutKeyPolicy`）。
* Claude Platform on AWS 的 Claude Console 中的 **Admin** 角色。请参阅[使用 Claude Console](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws#using-the-claude-console)。
* 对于您用于登录 Claude Console 的 IAM 主体：除了 `aws-external-anthropic:AssumeConsole` 之外，还需要您在那里执行的操作的 [IAM 操作](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#encryption-keys)，因为 Encryption keys 页面和密钥附加会通过 AWS 网关。注册密钥是 `RegisterKey`（使用 `ListKeys` 和 `GetKey` 查看注册），附加密钥是 `UpdateWorkspace` 或 `CreateWorkspace`。外部密钥操作（以及 `CreateWorkspace`）是账户范围的，因此在 `Resource: "*"` 上授予它们；仅限于工作区 ARN 的策略不包括它们。
* 对于将密钥附加到工作区的 IAM 主体（您登录 Claude Console 所用的身份）：密钥上的 `kms:DescribeKey`、`kms:Encrypt` 和 `kms:Decrypt`。附加密钥时，除了服务主体的访问权限外，还会检查您主体对密钥的访问权限。
* 可选，用于 Claude Console 中的密钥选择器：您登录所用主体的 `kms:ListKeys` 和 `kms:DescribeKey`。没有它们，请改为粘贴密钥 ARN。

### 创建 KMS 密钥

密钥策略包含三条语句：您账户的根管理员语句；一条允许 Claude Platform on AWS 服务主体加密、解密和生成数据密钥的语句；以及一条单独用于 `kms:DescribeKey` 的语句。加密语句带有一个可选的 `EncryptionContext` 条件，用于将密钥绑定到您列出的工作区。`DescribeKey` 需要单独授予，因为它没有 `EncryptionContext` 参数，所以在该操作上设置 `EncryptionContext` 条件将始终导致拒绝。

如果您计划使用此处所示的可选 `EncryptionContext` 条件，请先创建工作区（不使用密钥），复制其分区 ID，并用它替换 `<compartment-uuid>`。要在 Claude Console 中找到该 ID，请转到 [Manage > Security](https://platform.claude.com/settings/workspaces/default/security-compliance)，并在侧边栏顶部的工作区选择器中选择该工作区。该 ID 位于 **Encryption key** 下的 **Compartment ID** 字段中。您也可以从 [Get Workspace](https://platform.claude.com/docs/zh-CN/api/beta/organization/workspaces/retrieve) 端点返回的 `compartment_id` 字段中读取它。如果您不打算使用该条件，请从该语句中删除 `Condition` 块。

```bash
export YOUR_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

aws kms create-key \
  --region <workspace-region> \
  --description "Anthropic CMEK (Claude Platform on AWS)" \
  --key-usage ENCRYPT_DECRYPT \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Sid\": \"AccountRootAdmin\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"AWS\": \"arn:aws:iam::${YOUR_ACCOUNT}:root\"},
        \"Action\": \"kms:*\",
        \"Resource\": \"*\"
      },
      {
        \"Sid\": \"AllowClaudePlatformOnAWSCrypto\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"Service\": \"aws-external-anthropic.amazonaws.com\"},
        \"Action\": [\"kms:Encrypt\", \"kms:Decrypt\", \"kms:GenerateDataKey\"],
        \"Resource\": \"*\",
        \"Condition\": {
          \"StringEquals\": {
            \"kms:EncryptionContext:anthropic:compartment_uuid\": [
              \"<compartment-uuid>\"
            ]
          }
        }
      },
      {
        \"Sid\": \"AllowClaudePlatformOnAWSDescribe\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"Service\": \"aws-external-anthropic.amazonaws.com\"},
        \"Action\": \"kms:DescribeKey\",
        \"Resource\": \"*\"
      }
    ]
  }"
```

从输出中捕获 `KeyMetadata.Arn`。在注册密钥时您需要它。

`EncryptionContext` 条件是可选的。为某个工作区发起的每次加密、解密和数据密钥调用（包括附加时的检查）都会将该工作区的分区 ID 作为 `anthropic:compartment_uuid` 携带，因此该条件需列出您要附加密钥的每个工作区的分区 ID，且不需要全零条目。添加该条件还会在 IAM 层将密钥绑定到您列出的工作区。由于分区 ID 只有在其工作区存在后才会存在，因此顺序为：创建工作区，将其分区 ID 放入条件中（在创建密钥时，或之后通过 `kms:PutKeyPolicy`），然后附加密钥。在将密钥附加到每个额外的工作区之前，请以相同方式添加该工作区的分区 ID。如果要在不使用该条件的情况下开始，请从 `AllowClaudePlatformOnAWSCrypto` 语句中删除 `Condition` 块；如果之后再添加该条件，请包含该密钥已附加到的每个工作区的分区 ID。

您可以使用 `aws:SourceArn` 条件进一步限制这两条服务主体语句。该服务在使用您的密钥进行的每次调用中，都会将[工作区的 ARN](https://platform.claude.com/docs/zh-CN/api/claude-platform-on-aws-iam-actions#service-details)（`arn:aws:aws-external-anthropic:<region>:<account-id>:workspace/<workspace-id>`）作为源 ARN 传递，因此 `"ArnLike": {"aws:SourceArn": "arn:aws:aws-external-anthropic:*:<account-id>:workspace/*"}` 会将授权限制为您自己 AWS 账户中的工作区，而完整工作区 ARN 的列表则会将其限制为这些工作区。此条件不是必需的；仅凭 `EncryptionContext` 条件就能将密钥绑定到您列出的工作区。

您也可以从 AWS Console 创建密钥：在工作区的区域中，选择具有加密和解密密钥用途的对称密钥、单区域密钥和 KMS 密钥材料来源。在创建密钥向导中将密钥用途权限留空，然后打开密钥的 **Key policy** 选项卡，并将 JSON 替换为此处显示的策略。

### 注册并附加密钥

<Steps>
  <Step title="注册密钥">
    在 Claude Console 中，打开 **Settings > Encryption keys** 并单击 **Add key**。输入显示名称，然后从密钥选择器中选择密钥，或选择 **Enter ARN manually** 并粘贴密钥 ARN，然后单击 **Add**。密钥必须位于托管您组织的 AWS 账户中；不支持跨账户密钥。选择器列出您账户中位于您组织某个区域的已启用、客户管理、对称、单区域密钥；对于选择器未列出的密钥，请输入 ARN。仅当您登录所用主体可以调用 `kms:ListKeys` 和 `kms:DescribeKey` 时，它才会列出密钥。
  </Step>

  <Step title="将密钥附加到工作区">
    请在向新工作区发送任何请求之前将密钥附加到该工作区。对于已经在接收请求的工作区，密钥可能需要[最多一天才能生效](https://platform.claude.com/docs/zh-CN/manage-claude/cmek#how-it-works)。在 Claude Console 中，转到 [Manage > Security](https://platform.claude.com/settings/workspaces/default/security-compliance)，并在侧边栏顶部的工作区选择器中选择该工作区。在 **Encryption key** 下，选择密钥，单击 **Save** 并确认。您也可以在 Claude Console 中创建工作区时选择密钥，但前提是您的密钥策略尚未指定特定工作区（没有 `EncryptionContext` 条件），因为工作区的分区 ID 是在创建时分配的。附加后，工作区的密钥将无法更改。

    这是密钥被验证的时刻：附加调用会检查您主体对密钥的访问权限，并以工作区的分区 ID 作为加密上下文针对密钥执行一次加密/解密往返，因此密钥策略或您主体权限的问题会在该调用上显现为错误。如果附加因 KMS 访问错误而失败，请检查以下内容：

    * 密钥策略命名了 `aws-external-anthropic.amazonaws.com` 服务主体，并授予 `kms:Encrypt`、`kms:Decrypt` 和 `kms:GenerateDataKey`，以及在一条没有 `EncryptionContext` 条件的单独语句中的 `kms:DescribeKey`。
    * 任何 `EncryptionContext` 条件都包含此工作区的分区 ID，并且您添加的任何 `aws:SourceArn` 条件都与此工作区的 ARN 匹配。
    * 密钥已启用、单区域，并且与工作区位于同一 AWS 账户和区域。
    * 您登录所用的主体在密钥上具有 `kms:DescribeKey`、`kms:Encrypt` 和 `kms:Decrypt`。
    * 您的 AWS 组织中没有服务控制策略或资源控制策略阻止服务主体或您的主体使用密钥。
    * 如果策略看起来正确但附加仍然失败，请在密钥所在账户的 CloudTrail 中查找被拒绝的 `kms:` 事件（其中会显示调用主体，对于加密调用还会显示加密上下文），然后使用 `kms:PutKeyPolicy` 更正条件并重试。
  </Step>
</Steps>

## Terraform

对于基础设施即代码部署，相同的步骤映射到带有 `aws_kms_key` 和 `aws_kms_alias` 资源的 `aws` 提供程序。
