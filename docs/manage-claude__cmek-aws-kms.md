---
title: 为 CMEK 配置 AWS KMS
url: https://platform.claude.com/docs/zh-CN/manage-claude/cmek-aws-kms
description: 使用 AWS KMS 为您的组织提供加密密钥。
---

```bash Configure with the /claude-api skill in Claude Code
claude "/claude-api help me configure a customer-managed encryption key with AWS KMS"
```

本指南将逐步介绍如何将 [AWS KMS](https://aws.amazon.com/kms/) 密钥配置为您的 Anthropic 组织的 ["customer-managed encryption key"（客户管理的加密密钥），即 CMEK](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。

<Warning>
  启用 CMEK 是永久性的。如果您的 KMS 密钥被删除或禁用，Anthropic 将无法恢复使用该密钥加密的数据。在开始之前，请查看[警告和限制](https://platform.claude.com/docs/zh-CN/manage-claude/cmek)。
</Warning>

## 前提条件

* 一个拥有创建 KMS 密钥和设置密钥策略权限（`kms:CreateKey` 和 `kms:PutKeyPolicy`）的 AWS 账户。
* 您组织的 Anthropic Admin API 密钥。
* 已安装并完成身份验证的 [AWS CLI](https://aws.amazon.com/cli/)。

## Anthropic 的 Amazon 资源名称（ARN）

要让 Anthropic 使用您的加密密钥，您必须为 Anthropic 的 IAM 角色提供一个可用于加密数据的 KMS 密钥。Anthropic CMEK 的 ARN 为：

```text wrap
arn:aws:iam::915198916910:role/anthropic-cmek-client-us
```

<Warning>
  请仅使用此处公布的 ARN。切勿信任通过电子邮件、聊天或任何入门引导渠道提供的标识符。
</Warning>

## 加密密钥设置

<Steps>
  <Step title="创建带有跨账户密钥策略的 KMS 密钥">
    密钥策略授予 Anthropic 的 IAM 角色跨账户访问权限。需要以下三条语句：

    1. **账户根管理员：** 标准的 KMS 模式。您的账户保留完全的管理控制权。
    2. **Anthropic 加密和解密：** `kms:Encrypt` 和 `kms:Decrypt` 操作，Anthropic 使用它们来加密和解密保护您工作区数据的数据密钥（"envelope encryption"，即信封加密）。
    3. **Anthropic 描述：** Anthropic 在启动时执行的元数据读取。之所以单独授予，是因为 `DescribeKey` 没有 `EncryptionContext` 参数，因此对此操作设置 `EncryptionContext` 条件将始终导致拒绝。

    ```bash
    export YOUR_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

    aws kms create-key \
      --region <region> \
      --description "Anthropic CMEK" \
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
            \"Sid\": \"AllowAnthropicCMEKCrypto\",
            \"Effect\": \"Allow\",
            \"Principal\": {\"AWS\": \"arn:aws:iam::915198916910:role/anthropic-cmek-client-us\"},
            \"Action\": [\"kms:Encrypt\", \"kms:Decrypt\"],
            \"Resource\": \"*\",
            \"Condition\": {
              \"StringEquals\": {
                \"kms:EncryptionContext:anthropic:compartment_uuid\": [
                  \"00000000-0000-0000-0000-000000000000\",
                  \"<compartment-uuid>\"
                ]
              }
            }
          },
          {
            \"Sid\": \"AllowAnthropicCMEKDescribe\",
            \"Effect\": \"Allow\",
            \"Principal\": {\"AWS\": \"arn:aws:iam::915198916910:role/anthropic-cmek-client-us\"},
            \"Action\": \"kms:DescribeKey\",
            \"Resource\": \"*\"
          }
        ]
      }"
    ```

    从输出中记录 `KeyMetadata.Arn`。在下一步向 Anthropic 注册密钥时需要用到它。

    `EncryptionContext` 条件是推荐的，但并非必需。Anthropic 始终会在加密上下文中包含您工作区的 compartment ID（隔区 ID），因此无论如何，密文都会以加密方式绑定到该隔区。添加该条件可在 IAM 层提供纵深防御。如果想先不使用该条件，请从 `AllowAnthropicCMEKCrypto` 语句中省略 `Condition` 块，之后再通过 `kms:PutKeyPolicy` 添加。

    <Note>
      **查找您的 compartment ID：** 查找 compartment ID 的位置在 Claude Platform 和 Claude Enterprise 之间有所不同。请参阅**向 Anthropic 注册密钥**下的 **Claude Platform** 和 **Claude Enterprise** 选项卡。
    </Note>

    您也可以从 AWS 控制台创建密钥。选择对称密钥，密钥用途为加密和解密，单区域密钥，密钥材料来源为 KMS。创建密钥向导会在其 **Review**（审核）步骤提交密钥策略：如果您在该处的密钥使用权限下添加 Anthropic 的账户 ID `915198916910`，生成的策略会向整个 Anthropic 账户授予更广泛的操作（例如 `kms:ReEncrypt*` 和 `kms:GenerateDataKey*`），且不带 `EncryptionContext` 条件，而验证仍会针对该策略成功通过。为避免留下权限过宽的密钥，请仅使用管理权限完成向导，然后打开密钥的 **Key policy**（密钥策略）选项卡，将 JSON 替换为前面所示的角色范围策略（即限定于 `anthropic-cmek-client-us` 角色并带有 `EncryptionContext` 条件的三条语句）。

    <Frame caption="Configure key（配置密钥）：symmetric（对称）、encrypt and decrypt（加密和解密）、single-region key（单区域密钥）。">
      ![AWS KMS 创建密钥向导的 Configure key 步骤，已选择 Symmetric 密钥类型、Encrypt and decrypt 密钥用途以及 Single-Region key。](https://platform.claude.com/docs/images/cmek/aws-configure-key.png)
    </Frame>

    <Frame caption="为密钥添加 alias（别名）和 description（描述）。">
      ![AWS KMS Add labels 步骤，别名为 anthropic-cmek，描述为 Anthropic CMEK。](https://platform.claude.com/docs/images/cmek/aws-add-labels.png)
    </Frame>

    <Frame caption="Define key administrative permissions（定义密钥管理权限，可选）。您的账户保留完全的管理控制权。">
      ![AWS KMS Define key administrative permissions 步骤，列出了可以管理该密钥的 IAM 角色。](https://platform.claude.com/docs/images/cmek/aws-admin-permissions.png)
    </Frame>

    <Frame caption="请勿在此处添加 Anthropic 的账户 ID。此向导步骤会生成权限过宽的策略。请将 usage permissions（使用权限）留空，并在创建后编辑 Key policy（密钥策略）JSON（参见前面的密钥策略）。">
      ![AWS KMS Define key usage permissions 步骤，在 Other AWS accounts 下输入了 Anthropic 的账户 ID。](https://platform.claude.com/docs/images/cmek/aws-usage-permissions.png)
    </Frame>
  </Step>
</Steps>

## 向 Anthropic 注册密钥

注册密钥的方式取决于您使用的产品。

<Tabs>
  <Tab title="Claude Platform">
    <Note>
      **查找您的 compartment ID：** 每个工作区都有一个 compartment ID，用于限定其 CMEK 数据的范围。您可以在 Claude Console 的 **Workspace > Security > Encryption keys**（**Compartment ID** 字段）中找到它，或读取 [Get Workspace](https://platform.claude.com/docs/zh-CN/api/admin-api/workspaces/get-workspace) 端点返回的 `compartment_id` 字段。将该值替换前面密钥策略中的 `<compartment-uuid>`。

      密钥验证始终发送全零的 compartment UUID（`00000000-0000-0000-0000-000000000000`）作为加密上下文，因为验证在密钥附加到任何工作区之前运行。实际流量会发送每个已附加工作区的 compartment ID。

      任何 `EncryptionContext` 条件都必须允许全零值以及该密钥所附加的每个工作区的 compartment ID。每当重新运行密钥设置时，验证也会再次运行，因此请永久保留全零条目。

      要将密钥附加到其他工作区，请在附加之前使用 `kms:PutKeyPolicy` 将该工作区的 compartment ID 添加到条件中。
    </Note>

    <Steps>
      <Step title="向 Anthropic 注册密钥">
        通过 Admin API 创建外部密钥配置。

        <Note>
          对于使用 [Claude Platform on AWS](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws) 的组织，外部密钥端点尚不可用。请改为在 Claude Console 中注册、验证和附加您的密钥。
        </Note>

        ```bash
        curl -sS https://api.anthropic.com/v1/organizations/external_keys \
          -H "x-api-key: <anthropic-admin-api-key>" \
          -H "anthropic-version: 2023-06-01" \
          -H "content-type: application/json" \
          -d '{
            "display_name": "<friendly-name>",
            "geo": "us",
            "provider_config": {
              "type": "aws",
              "kms_arn": "<key-arn-from-create-key-step>",
              "role_arn": "arn:aws:iam::915198916910:role/anthropic-cmek-client-us"
            }
          }'
        ```

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
        针对您的密钥触发一次加密和解密往返。

        ```bash
        curl -sS -X POST https://api.anthropic.com/v1/organizations/external_keys/ekey_<id>/validate \
          -H "x-api-key: <anthropic-admin-api-key>" \
          -H "anthropic-version: 2023-06-01" \
          -H "content-type: application/json" \
          -d '{}'
        ```

        成功的响应如下所示：

        ```json
        { "type": "external_key_validation", "status": "success", "error": null }
        ```

        如果验证失败，常见原因包括：

        * **加密上下文不匹配：** 当 `kms:EncryptionContext:anthropic:compartment_uuid` 条件仅允许 Anthropic 发送的两个值之一时，验证会失败而数据流量正常（或相反），并返回不透明的 `AccessDeniedException`。验证发送全零 UUID（`00000000-0000-0000-0000-000000000000`）；实际流量发送已附加工作区的 compartment ID。请确认条件中列出了这两个值。要完全排除该条件的影响，请暂时从 `AllowAnthropicCMEKCrypto` 语句中移除 `Condition` 块并重新验证。
        * **资源控制策略（RCP）：** 如果您的 AWS 组织有一个 RCP，在 `aws:PrincipalOrgID` 与您的组织不匹配时拒绝 KMS 操作，它会阻止 Anthropic 的跨账户角色。该 RCP 需要为此密钥或 Anthropic 的角色 ARN 设置例外。服务控制策略在此不适用，因为它们不会对通过基于资源的策略进行调用的外部主体进行评估。
        * **通过 IAM 而非密钥策略授予访问权限：** 跨账户 KMS 访问权限必须在密钥策略本身中授予，而不是通过您账户中的 IAM 策略。请使用 `aws kms get-key-policy --key-id <id> --policy-name default` 进行检查。
        * **区域不匹配：** 请确认密钥所在区域是 Anthropic 针对您配置的地理层级所运营的区域之一。
      </Step>

      <Step title="将密钥附加到工作区">
        密钥验证通过后，请在向新工作区发送任何请求之前将其附加到该工作区。对于已经在接收请求的工作区，密钥可能需要[最多一天才能生效](https://platform.claude.com/docs/zh-CN/manage-claude/cmek#how-it-works)。

        ```bash
        curl -sS -X POST https://api.anthropic.com/v1/organizations/workspaces/<workspace-id> \
          -H "x-api-key: <anthropic-admin-api-key>" \
          -H "anthropic-version: 2023-06-01" \
          -H "content-type: application/json" \
          -d '{
            "external_key_id": "ekey_<id>"
          }'
        ```
      </Step>
    </Steps>
  </Tab>

  <Tab title="Claude Enterprise">
    在 [claude.ai > Organization settings > Data and privacy](https://claude.ai/admin-settings/data-privacy-controls) 中，打开 **Encryption keys**，然后点击 **Add key**。选择 **AWS** 并点击 **Continue**，然后粘贴上一步中的 Key ARN 并点击 **Add**。Anthropic 会通过一次加密和解密往返来验证密钥。一旦显示为已验证，您的组织从那时起即受 CMEK 保护。

    此流程的密钥详情步骤会显示您组织的 **Compartment ID**，并带有复制按钮。将该值替换密钥策略中的 `<compartment-uuid>`（参见加密密钥设置下的创建 KMS 密钥步骤）；您可以在创建密钥之前打开该流程以复制此 ID。设置完成后，该 ID 仍会显示在 **Encryption keys** 下的密钥上。

    在 Claude Enterprise 上，CMEK 适用于整个组织，因此没有单独的工作区附加步骤，并且一个组织只能有一个密钥。
  </Tab>
</Tabs>

## Terraform

对于基础设施即代码部署，相同的步骤对应于 `aws` provider 中的 `aws_kms_key` 和 `aws_kms_alias` 资源。
