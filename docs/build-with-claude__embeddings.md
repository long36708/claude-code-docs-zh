---
title: 嵌入
url: https://platform.claude.com/docs/zh-CN/build-with-claude/embeddings
description: 文本嵌入是文本的数值表示，可用于衡量语义相似度。本指南介绍嵌入及其应用，以及如何使用嵌入模型完成搜索、推荐和异常检测等任务。
---

## 实施嵌入之前

在选择 "embeddings"（嵌入）提供商时，您可以根据自己的需求和偏好考虑以下几个因素：

* 数据集规模与领域特异性：模型训练数据集的规模及其与您想要嵌入的领域的相关性。更大或更具领域特异性的数据通常会产生更好的领域内嵌入
* 推理性能：嵌入查找速度和端到端 "latency"（延迟）。对于大规模生产部署而言，这是一个尤为重要的考虑因素
* 定制化：在私有数据上继续训练的选项，或针对非常特定的领域对模型进行专门化。这可以提升在独特词汇上的性能

## 如何通过 Anthropic 获取嵌入

Anthropic 不提供自己的嵌入模型。Voyage AI 是一家拥有多种选项和能力、涵盖上述所有考虑因素的嵌入提供商。

Voyage AI 打造最先进的嵌入模型，并为金融和医疗等特定行业领域提供定制模型，或为个别客户提供专属的 "fine-tuned"（微调）模型。

本指南的其余部分针对 Voyage AI，但您应评估多家嵌入供应商，以找到最适合您特定用例的方案。

## 可用模型

Voyage 推荐使用以下文本嵌入模型：

**Voyage 4（最新一代）**

| 模型               | 上下文长度  | 嵌入维度                  | 描述                                                                                                               |
| ---------------- | ------ | --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `voyage-4-large` | 32,000 | 1024（默认）、256、512、2048 | 最佳的通用和多语言检索质量。详情请参阅 [Voyage 4 博客文章](https://blog.voyageai.com/2026/01/15/voyage-4/)。                             |
| `voyage-4`       | 32,000 | 1024（默认）、256、512、2048 | 针对通用和多语言检索质量进行了优化。在质量与效率之间取得平衡。详情请参阅 [Voyage 4 博客文章](https://blog.voyageai.com/2026/01/15/voyage-4/)。            |
| `voyage-4-lite`  | 32,000 | 1024（默认）、256、512、2048 | 针对延迟和成本进行了优化。详情请参阅 [Voyage 4 博客文章](https://blog.voyageai.com/2026/01/15/voyage-4/)。                              |
| `voyage-4-nano`  | 32,000 | 1024（默认）、256、512、2048 | 在 Hugging Face 上提供的开放权重模型（Apache 2.0 许可证）。详情请参阅 [Voyage 4 博客文章](https://blog.voyageai.com/2026/01/15/voyage-4/)。 |

**上一代**

| 模型                 | 上下文长度  | 嵌入维度                  | 描述                                                                                                                                                                                  |
| ------------------ | ------ | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `voyage-3-large`   | 32,000 | 1024（默认）、256、512、2048 | 最佳的通用和多语言检索质量。详情请参阅 [voyage-3-large 博客文章](https://blog.voyageai.com/2025/01/07/voyage-3-large/)。                                                                                    |
| `voyage-3.5`       | 32,000 | 1024（默认）、256、512、2048 | 针对通用和多语言检索质量进行了优化。详情请参阅 [voyage-3.5 博客文章](https://blog.voyageai.com/2025/05/20/voyage-3-5/)。                                                                                        |
| `voyage-3.5-lite`  | 32,000 | 1024（默认）、256、512、2048 | 针对延迟和成本进行了优化。详情请参阅 [voyage-3.5 博客文章](https://blog.voyageai.com/2025/05/20/voyage-3-5/)。                                                                                             |
| `voyage-code-3`    | 32,000 | 1024（默认）、256、512、2048 | 针对**代码**检索进行了优化。详情请参阅 [voyage-code-3 博客文章](https://blog.voyageai.com/2024/12/04/voyage-code-3/)。                                                                                    |
| `voyage-finance-2` | 32,000 | 1024                  | 针对**金融**检索和 RAG 进行了优化。详情请参阅 [voyage-finance-2 博客文章](https://blog.voyageai.com/2024/06/03/domain-specific-embeddings-finance-edition-voyage-finance-2/)。                             |
| `voyage-law-2`     | 16,000 | 1024                  | 针对**法律**和**长上下文**检索及 RAG 进行了优化。同时在所有领域的性能均有提升。详情请参阅 [voyage-law-2 博客文章](https://blog.voyageai.com/2024/04/15/domain-specific-embeddings-and-retrieval-legal-edition-voyage-law-2/)。 |

此外，Voyage 推荐以下多模态嵌入模型：

| 模型                      | 上下文长度  | 嵌入维度                  | 描述                                                                                                                                                      |
| ----------------------- | ------ | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `voyage-multimodal-3.5` | 32,000 | 1024（默认）、256、512、2048 | 功能丰富的多模态嵌入模型，可对交错排列的文本、图像和视频进行向量化。作为首个生产级视频嵌入模型，包含视频支持。详情请参阅 [voyage-multimodal-3.5 博客文章](https://blog.voyageai.com/2026/01/15/voyage-multimodal-3-5/)。 |
| `voyage-multimodal-3`   | 32,000 | 1024                  | 功能丰富的多模态嵌入模型，可对交错排列的文本和内容丰富的图像（例如 PDF 截图、幻灯片、表格、图表等）进行向量化。详情请参阅 [voyage-multimodal-3 博客文章](https://blog.voyageai.com/2024/11/12/voyage-multimodal-3/)。  |

以下上下文化分块嵌入模型可生成分块级向量，无需手动添加元数据即可捕获完整的文档上下文。请使用 `contextualized_embed()` 而非 `embed()` 调用这些模型：

| 模型                 | 上下文长度   | 嵌入维度                  | 描述                                                                                                                |
| ------------------ | ------- | --------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `voyage-context-4` | 120,000 | 1024（默认）、256、512、2048 | 针对通用和多语言检索质量进行了优化的上下文化分块嵌入。详情请参阅 [voyage-context-4 博客文章](https://blog.voyageai.com/2026/06/29/voyage-context-4/)。 |
| `voyage-context-3` | 120,000 | 1024（默认）、256、512、2048 | 针对通用和多语言检索质量进行了优化的上下文化分块嵌入。详情请参阅 [voyage-context-3 博客文章](https://blog.voyageai.com/2025/07/23/voyage-context-3/)。 |

Voyage AI 还提供 "rerankers"（重排序器），它接收一个查询和一组文档，并按与查询的相关性排序后返回这些文档。请使用 `rerank()` 调用这些模型：

| 模型                | 上下文长度  | 描述                                                                                         |
| ----------------- | ------ | ------------------------------------------------------------------------------------------ |
| `rerank-2.5`      | 32,000 | 准确率最高。推荐用于大多数应用。详情请参阅 [rerank-2.5 博客文章](https://blog.voyageai.com/2025/08/11/rerank-2-5/)。 |
| `rerank-2.5-lite` | 32,000 | 针对延迟和成本进行了优化。详情请参阅 [rerank-2.5 博客文章](https://blog.voyageai.com/2025/08/11/rerank-2-5/)。    |

需要帮助决定使用哪个文本嵌入模型？请查看 [Voyage AI 常见问题](https://docs.voyageai.com/docs/faq#what-embedding-models-are-available-and-which-one-should-i-use\&ref=anthropic)。

## Voyage AI 入门

要访问 Voyage 嵌入：

1. 在 Voyage AI 网站上注册。
2. 获取 API 密钥。
3. 为方便起见，将 API 密钥设置为环境变量：

```bash
export VOYAGE_API_KEY="<your secret key>"
```

您可以使用官方的 [`voyageai` Python 包](https://github.com/voyage-ai/voyageai-python)或 HTTP 请求来获取嵌入，如以下各节所述。

### Voyage Python 库

使用以下命令安装 `voyageai` 包：

```bash
pip install -U voyageai
```

然后，您可以创建一个客户端对象并开始使用它来嵌入您的文本：

```python
import voyageai

vo = voyageai.Client()
# 这将自动使用环境变量 VOYAGE_API_KEY。
# 或者，您也可以使用 vo = voyageai.Client(api_key="<your secret key>")

texts = ["Sample text 1", "Sample text 2"]

result = vo.embed(texts, model="voyage-4", input_type="document")
print(result.embeddings[0])
print(result.embeddings[1])
```

`result.embeddings` 是一个包含两个嵌入向量的列表，每个向量包含 1024 个浮点数。运行上述代码后，这两个嵌入将打印在屏幕上：

```text
[-0.013131560757756233, 0.019828535616397858, ...]   # embedding for "Sample text 1"
[-0.0069352793507277966, 0.020878976210951805, ...]  # embedding for "Sample text 2"
```

创建嵌入时，您可以为 `embed()` 函数指定一些其他参数。

有关 Voyage Python 包的更多信息，请参阅 [Voyage Python 包文档](https://docs.voyageai.com/docs/embeddings#python-api)。

### Voyage HTTP API

您也可以通过请求 Voyage HTTP API 来获取嵌入。例如，您可以在终端中通过 `curl` 命令发送 HTTP 请求：

```bash cURL
curl https://api.voyageai.com/v1/embeddings \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $VOYAGE_API_KEY" \
  -d '{
    "input": ["Sample text 1", "Sample text 2"],
    "model": "voyage-4"
  }'
```

您将得到的响应是一个包含嵌入和令牌用量的 JSON 对象：

```json
{
  "object": "list",
  "data": [
    {
      "embedding": [-0.013131560757756233, 0.019828535616397858 /* ... */],
      "index": 0
    },
    {
      "embedding": [-0.0069352793507277966, 0.020878976210951805 /* ... */],
      "index": 1
    }
  ],
  "model": "voyage-4",
  "usage": {
    "total_tokens": 10
  }
}
```

有关 Voyage HTTP API 的更多信息，请参阅 [Voyage HTTP API 文档](https://docs.voyageai.com/reference/embeddings-api)。

### AWS Marketplace

Voyage 嵌入可在 [AWS Marketplace](https://aws.amazon.com/marketplace/seller-profile?id=c9032c7b-70dd-459f-834f-c1e23cf3d092) 上获取。有关在 AWS 上访问 Voyage 的说明，请参阅 [Voyage AWS Marketplace 文档](https://docs.voyageai.com/docs/aws-marketplace-mongodb-voyage?ref=anthropic)。

## 快速入门示例

以下简短示例展示了如何使用嵌入。

假设您有一个包含六个文档的小型语料库可供检索

```python
documents = [
    "The Mediterranean diet emphasizes fish, olive oil, and vegetables, believed to reduce chronic diseases.",
    "Photosynthesis in plants converts light energy into glucose and produces essential oxygen.",
    "20th-century innovations, from radios to smartphones, centered on electronic advancements.",
    "Rivers provide water, irrigation, and habitat for aquatic species, vital for ecosystems.",
    "Apple's conference call to discuss fourth fiscal quarter results and business updates is scheduled for Thursday, November 2, 2023 at 2:00 p.m. PT / 5:00 p.m. ET.",
    "Shakespeare's works, like 'Hamlet' and 'A Midsummer Night's Dream,' endure in literature.",
]
```

首先，使用 Voyage 将每个文档转换为嵌入向量。

```python
import voyageai

vo = voyageai.Client()

# 嵌入文档
doc_embds = vo.embed(documents, model="voyage-4", input_type="document").embeddings
```

嵌入使您能够在向量空间中进行语义搜索/检索。给定一个示例查询，

```python
query = "When is Apple's conference call scheduled?"
```

接下来，将其转换为嵌入，并进行最近邻搜索，根据嵌入空间中的距离找到最相关的文档。

```python
import numpy as np

# 对查询进行嵌入
query_embd = vo.embed([query], model="voyage-4", input_type="query").embeddings[0]

# 计算相似度
# Voyage 嵌入已归一化为长度 1，因此点积
# 与余弦相似度是相同的。
similarities = np.dot(doc_embds, query_embd)

retrieved_id = np.argmax(similarities)
print(documents[retrieved_id])
```

请注意，`input_type="document"` 和 `input_type="query"` 分别用于嵌入文档和查询。更多规范可在 [Voyage Python 库](https://platform.claude.com/docs/zh-CN/build-with-claude/embeddings#voyage-python-library)中找到。

输出是第五个文档，它确实是与查询最相关的文档：

```text wrap
Apple's conference call to discuss fourth fiscal quarter results and business updates is scheduled for Thursday, November 2, 2023 at 2:00 p.m. PT / 5:00 p.m. ET.
```

如果您正在寻找一套关于如何使用嵌入进行 RAG（包括向量数据库）的详细方案，请查看 [RAG 方案](https://platform.claude.com/cookbook/third-party-pinecone-rag-using-pinecone)。

## 常见问题

<AccordionGroup>
  <Accordion title="为什么 Voyage 嵌入具有卓越的质量？">
    嵌入模型依赖强大的神经网络来捕获和压缩语义上下文，这与生成式模型类似。Voyage 经验丰富的 AI 研究团队对嵌入过程的每个组成部分都进行了优化，包括：

    * 模型架构
    * 数据收集
    * 损失函数
    * 优化器选择

    在 [Voyage AI 博客](https://blog.voyageai.com/)上了解更多关于 Voyage 技术方法的信息。
  </Accordion>

  <Accordion title="有哪些可用的嵌入模型，我应该使用哪一个？">
    对于通用嵌入，推荐的模型是：

    * `voyage-4-large`：质量最佳
    * `voyage-4-lite`：延迟和成本最低
    * `voyage-4`：性能均衡

    对于检索，请使用 `input_type` 参数指定文本是查询类型还是文档类型。

    领域专用模型：

    * 法律任务：`voyage-law-2`
    * 代码和编程文档：`voyage-code-3`
    * 金融相关任务：`voyage-finance-2`

    对于分块级和文档级检索：`voyage-context-4`
  </Accordion>

  <Accordion title="我应该使用哪种相似度函数？">
    您可以将 Voyage 嵌入与点积相似度、余弦相似度或欧几里得距离配合使用。有关嵌入相似度的说明，请参阅这篇[向量相似度指南](https://www.pinecone.io/learn/vector-similarity/)。

    Voyage AI 嵌入已归一化为长度 1，这意味着：

    * 余弦相似度等价于点积相似度，而后者的计算速度更快。
    * 余弦相似度和欧几里得距离会产生相同的排序结果。
  </Accordion>

  <Accordion title="字符、单词和令牌之间是什么关系？">
    请参阅 [Voyage 分词指南](https://docs.voyageai.com/docs/tokenization?ref=anthropic)。
  </Accordion>

  <Accordion title="何时以及如何使用 input_type 参数？">
    对于所有检索任务和用例（例如 RAG），请使用 `input_type` 参数指定输入文本是查询还是文档。不要省略 `input_type` 或设置 `input_type=None`。指定输入文本是查询还是文档可以为检索创建更好的稠密向量表示，从而带来更好的检索质量。

    使用 `input_type` 参数时，会在嵌入之前将特殊提示添加到输入文本的前面。具体而言：

    > 📘 **与 `input_type` 关联的提示**
    >
    > * 对于查询，提示为 "Represent the query for retrieving supporting documents: "。
    >
    > * 对于文档，提示为 "Represent the document for retrieval: "。
    >
    > * 示例
    >
    >   * 当 `input_type="query"` 时，像 "When is Apple's conference call scheduled?" 这样的查询将变为 "**Represent the query for retrieving supporting documents:** When is Apple's conference call scheduled?"
    >   * 当 `input_type="document"` 时，像 "Apple's conference call to discuss fourth fiscal quarter results and business updates is scheduled for Thursday, November 2, 2023 at 2:00 p.m. PT / 5:00 p.m. ET." 这样的查询将变为 "**Represent the document for retrieval:** Apple's conference call to discuss fourth fiscal quarter results and business updates is scheduled for Thursday, November 2, 2023 at 2:00 p.m. PT / 5:00 p.m. ET."

    `voyage-large-2-instruct` 顾名思义，经过训练可以响应添加到输入文本前面的附加指令。对于分类、聚类或其他 [MTEB](https://huggingface.co/mteb) 子任务，请使用 [voyage-large-2-instruct 指令](https://github.com/voyage-ai/voyage-large-2-instruct)。
  </Accordion>

  <Accordion title="有哪些可用的量化选项？">
    嵌入中的 "quantization"（量化）将高精度值（例如 32 位单精度浮点数）转换为较低精度的格式（例如 8 位整数或 1 位二进制值），分别将存储、内存和成本降低 4 倍和 32 倍。受支持的 Voyage 模型可通过 `output_dtype` 参数指定输出数据类型来启用量化：

    * `float`：每个返回的嵌入是一个由 32 位（4 字节）单精度浮点数组成的列表。这是默认值，可提供最高的精度/检索准确率。
    * `int8` 和 `uint8`：每个返回的嵌入是一个由 8 位（1 字节）整数组成的列表，取值范围分别为 -128 到 127 和 0 到 255。
    * `binary` 和 `ubinary`：每个返回的嵌入是一个由 8 位整数组成的列表，这些整数表示经过位打包、量化的单比特嵌入值：`binary` 对应 `int8`，`ubinary` 对应 `uint8`。返回的整数列表的长度是嵌入实际维度的 1/8。binary 类型使用偏移二进制方法，您可以在[嵌入常见问题](https://platform.claude.com/docs/zh-CN/build-with-claude/embeddings#faq)中了解更多信息。

    > **二进制量化示例**
    >
    > 考虑以下八个嵌入值：-0.03955078、0.006214142、-0.07446289、-0.039001465、0.0046463013、0.00030612946、-0.08496094 和 0.03994751。使用二进制量化时，小于或等于零的值将被量化为二进制零，正值将被量化为二进制一，从而得到以下二进制序列：0, 1, 0, 0, 1, 1, 0, 1。然后这八个比特被打包成一个 8 位整数 01001101（最左边的位为最高有效位）。
    >
    > * `ubinary`：该二进制序列被直接转换并表示为无符号整数（`uint8`）77。
    > * `binary`：该二进制序列被表示为有符号整数（`int8`）-51，使用偏移二进制方法计算得出（77 - 128 = -51）。
  </Accordion>

  <Accordion title="如何截断 Matryoshka 嵌入？">
    Matryoshka 学习可在单个向量中创建具有由粗到细表示的嵌入。支持多种输出维度的 Voyage 模型（例如 `voyage-code-3`）会生成此类 Matryoshka 嵌入。您可以通过保留前面的维度子集来截断这些向量。例如，以下 Python 代码演示了如何将 1024 维向量截断为 256 维：

    ```python
    import voyageai
    import numpy as np


    def embd_normalize(v: np.ndarray) -> np.ndarray:
        """
        Normalize the rows of a 2D numpy array to unit vectors by dividing each row by its Euclidean
        norm. Raises a ValueError if any row has a norm of zero to prevent division by zero.
        """
        row_norms = np.linalg.norm(v, axis=1, keepdims=True)
        if np.any(row_norms == 0):
            raise ValueError("Cannot normalize rows with a norm of zero.")
        return v / row_norms


    vo = voyageai.Client()

    # 生成 voyage-code-3 向量，默认为 1024 维浮点数
    embd = vo.embed(["Sample text 1", "Sample text 2"], model="voyage-code-3").embeddings

    # 设置较短的维度
    short_dim = 256

    # 将向量调整为较短维度并进行归一化
    resized_embd = embd_normalize(np.array(embd)[:, :short_dim]).tolist()
    ```
  </Accordion>
</AccordionGroup>

## 定价

请访问 Voyage 的[定价页面](https://docs.voyageai.com/docs/pricing?ref=anthropic)了解最新的定价详情。
