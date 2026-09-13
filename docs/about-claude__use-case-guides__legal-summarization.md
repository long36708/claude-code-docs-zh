---
title: 法律文件摘要
url: https://platform.claude.com/docs/zh-CN/about-claude/use-case-guides/legal-summarization
description: 本指南将介绍如何利用 Claude 先进的自然语言处理能力高效地对法律文件进行摘要，提取关键信息并加快法律研究。借助 Claude，您可以简化合同审查、诉讼准备和监管工作，节省时间并确保法律流程的准确性。
---

> 访问[摘要 cookbook](https://platform.claude.com/cookbook/capabilities-summarization-guide)，查看使用 Claude 实现法律文件摘要的示例。

## 使用 Claude 构建之前

### 决定是否使用 Claude 进行法律文件摘要

以下是一些关键指标，表明您应该使用 Claude 这样的"large language model"（大型语言模型），即 LLM 来对法律文件进行摘要：

<AccordionGroup>
  <Accordion title="您希望高效且经济地审查大量文件">
    大规模文件审查如果手动完成，可能既耗时又昂贵。Claude 可以快速处理和摘要大量法律文件，显著减少与文件审查相关的时间和成本。这一能力对于尽职调查、合同分析或诉讼证据开示等效率至关重要的任务尤其有价值。
  </Accordion>

  <Accordion title="您需要自动提取关键元数据">
    Claude 可以高效地从法律文件中提取并分类重要的元数据，例如相关当事方、日期、合同条款或特定条文。这种自动提取有助于组织信息，使大型文件集更易于搜索、分析和管理。它对于合同管理、合规检查或创建可搜索的法律信息数据库尤其有用。
  </Accordion>

  <Accordion title="您希望生成清晰、简洁且标准化的摘要">
    Claude 可以生成遵循预定格式的结构化摘要，使法律专业人士更容易快速掌握各类文件的要点。这些标准化摘要可以提高可读性，便于文件之间的比较，并增强整体理解，尤其是在处理复杂的法律语言或技术术语时。
  </Accordion>

  <Accordion title="您的摘要需要精确的引用">
    在创建法律摘要时，适当的出处说明和引用对于确保可信度和符合法律标准至关重要。可以通过提示让 Claude 为所有引用的法律要点提供准确的引用，使法律专业人士更容易审查和核实摘要信息。
  </Accordion>

  <Accordion title="您希望简化并加快法律研究流程">
    Claude 可以通过快速分析大量判例法、法规和法律评论来协助法律研究。它可以识别相关先例、提取关键法律原则并摘要复杂的法律论证。这一能力可以显著加快研究过程，使法律专业人士能够专注于更高层次的分析和策略制定。
  </Accordion>
</AccordionGroup>

### 确定您希望摘要提取的细节

对于任何给定的文件，都不存在唯一正确的摘要。如果没有明确的指引，Claude 可能难以确定应包含哪些细节。为了获得最佳结果，请确定您希望在摘要中包含的具体信息。

例如，在对转租协议进行摘要时，您可能希望提取以下要点：

```python
details_to_extract = [
    "Parties involved (sublessor, sublessee, and original lessor)",
    "Property details (address, description, and permitted use)",
    "Term and rent (start date, end date, monthly rent, and security deposit)",
    "Responsibilities (utilities, maintenance, and repairs)",
    "Consent and notices (landlord's consent, and notice requirements)",
    "Special provisions (furniture, parking, and subletting restrictions)",
]
```

### 建立成功标准

评估摘要质量是一项众所周知的艰巨任务。与许多其他自然语言处理任务不同，摘要的评估通常缺乏明确、客观的指标。这一过程可能非常主观，不同的读者看重摘要的不同方面。以下是您在评估 Claude 法律文件摘要表现时可能需要考虑的标准。

<AccordionGroup>
  <Accordion title="事实正确性">
    摘要应准确呈现文件中的事实、法律概念和要点。
  </Accordion>

  <Accordion title="法律精确性">
    术语以及对法规、判例法或规章的引用必须正确并符合法律标准。
  </Accordion>

  <Accordion title="简洁性">
    摘要应将法律文件浓缩为其核心要点，同时不丢失重要细节。
  </Accordion>

  <Accordion title="一致性">
    如果对多份文件进行摘要，LLM 应对每份摘要保持一致的结构和方法。
  </Accordion>

  <Accordion title="可读性">
    文本应清晰易懂。如果受众不是法律专家，摘要不应包含可能使受众困惑的法律术语。
  </Accordion>

  <Accordion title="偏见与公正性">
    摘要应对法律论证和立场进行无偏见且公正的描述。
  </Accordion>
</AccordionGroup>

请参阅[建立成功标准](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests)指南了解更多信息。

***

## 如何使用 Claude 对法律文件进行摘要

### 选择合适的 Claude 模型

在对法律文件进行摘要时，模型准确性极为重要。对于此类需要高准确性的用例，Claude Opus 5 是一个极佳的选择。如果您的文件规模和数量很大，以至于成本开始成为一个问题，您也可以尝试使用较小的模型，例如 Claude Haiku 4.5。

为帮助估算这些成本，以下是使用 Opus 和 Haiku 模型对 1,000 份转租协议进行摘要的成本比较：

* **内容规模**

  * 协议数量：1,000
  * 每份协议的字符数：300,000
  * 总字符数：3 亿

* **估算令牌数**

  * 输入令牌：8600 万（假设每 3.5 个字符对应 1 个令牌）
  * 每份摘要的输出令牌：350
  * 总输出令牌：350,000

* **Claude Opus 5 估算成本**

  * 输入令牌成本：86 MTok \* $5.00/MTok = $430.00 USD
  * 输出令牌成本：0.35 MTok \* $25.00/MTok = $8.75 USD
  * 总成本：$430.00 + $8.75 = $438.75 USD

* **Claude Opus 4.8 估算成本**

  * 输入令牌成本：86 MTok \* $5.00/MTok = $430.00 USD
  * 输出令牌成本：0.35 MTok \* $25.00/MTok = $8.75 USD
  * 总成本：$430.00 + $8.75 = $438.75 USD

* **Claude Haiku 4.5 估算成本**

  * 输入令牌成本：86 MTok \* $1.00/MTok = $86.00 USD
  * 输出令牌成本：0.35 MTok \* $5.00/MTok = $1.75 USD
  * 总成本：$86.00 + $1.75 = $87.75 USD

<Tip>
  实际成本可能与这些估算有所不同。这些估算基于

  [构建强大的提示](https://platform.claude.com/docs/zh-CN/about-claude/use-case-guides/legal-summarization#build-a-strong-prompt)

  部分中重点介绍的示例。
</Tip>

### 将文件转换为 Claude 可以处理的格式

在开始对文件进行摘要之前，您需要准备数据。这包括从 PDF 中提取文本、清理文本，并确保其可供 Claude 处理。

以下是在示例 PDF 上演示此过程：

```python
from io import BytesIO
import re

import pypdf
import requests


def get_llm_text(pdf_file):
    reader = pypdf.PdfReader(pdf_file)
    text = "\n".join([page.extract_text() for page in reader.pages])

    # 移除页码
    text = re.sub(r"\n\s*\d+\s*\n", "\n", text)

    # 移除多余的空白字符
    text = re.sub(r"\s+", " ", text)

    return text


# 根据 GitHub 仓库构建完整的 URL
url = "https://raw.githubusercontent.com/anthropics/claude-cookbooks/main/capabilities/summarization/data/Sample Sublease Agreement.pdf"
url = url.replace(" ", "%20")

# 将 PDF 文件下载到内存中
response = requests.get(url)

# 从内存中加载 PDF
pdf_file = BytesIO(response.content)

document_text = get_llm_text(pdf_file)
print(document_text[:50000])
```

在此示例中，您首先下载[摘要 cookbook](https://platform.claude.com/cookbook/capabilities-summarization-guide) 中使用的示例转租协议 PDF。该协议来源于 [sec.gov 网站](https://www.sec.gov/Archives/edgar/data/1045425/000119312507044370/dex1032.htm)上公开提供的转租协议。

该示例使用 pypdf 库提取 PDF 的内容并将其转换为文本。然后通过删除页码和多余空白来清理文本数据。

### 构建强大的提示

Claude 可以适应各种摘要风格。您可以更改提示的细节，引导 Claude 更详细或更简略、包含更多或更少的技术术语，或对当前上下文提供更高层次或更低层次的摘要。

以下示例展示了如何创建一个提示，以确保在分析转租协议时生成的摘要遵循一致的结构：

```python Python
# 初始化 Anthropic 客户端
client = anthropic.Anthropic()


def summarize_document(
    text, details_to_extract, model="claude-opus-5", max_tokens=1000
):
    # 格式化待提取的详细信息，以便放入提示的上下文中
    details_to_extract_str = "\n".join(details_to_extract)

    # 提示模型对转租协议进行总结
    prompt = f"""Summarize the following sublease agreement. Focus on these key aspects:

    {details_to_extract_str}

    Provide the summary in bullet points nested within the XML header for each section. For example:

    <parties involved>
    - Sublessor: [Name]
    // Add more details as needed
    </parties involved>

    If any information is not explicitly stated in the document, note it as "Not specified". Do not preamble.

    Sublease agreement text:
    {text}
    """

    response = client.messages.create(
        model=model,
        max_tokens=max_tokens,
        system="You are a legal analyst specializing in real estate law, known for highly accurate and detailed summaries of sublease agreements.",
        messages=[
            {"role": "user", "content": prompt},
        ],
    )

    return next(block.text for block in response.content if block.type == "text")


sublease_summary = summarize_document(document_text, details_to_extract)
print(sublease_summary)
```

此代码实现了一个 `summarize_document` 函数，该函数使用 Claude 对转租协议的内容进行摘要。该函数接受一个文本字符串和一个待提取细节列表作为输入。在此示例中，代码使用前面代码片段中定义的 `document_text` 和 `details_to_extract` 变量调用该函数。

在函数内部，会为 Claude 生成一个提示，其中包括待摘要的文件、待提取的细节以及摘要文件的具体指令。该提示指示 Claude 以嵌套在 XML 标头中的形式返回每个待提取细节的摘要。

由于代码将摘要的每个部分输出在标签内，因此每个部分都可以在后处理步骤中轻松解析出来。这种方法可以生成可根据您的用例进行调整的结构化摘要，使每份摘要都遵循相同的模式。

### 评估您的提示

提示通常需要经过测试和优化才能投入生产使用。要确定您的解决方案是否就绪，请使用结合定量和定性方法的系统化流程来评估摘要的质量。基于您定义的成功标准创建[强有力的实证评估](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests#build-evaluations)，可以让您优化提示。以下是您可能希望纳入实证评估的一些指标：

<AccordionGroup>
  <Accordion title="ROUGE 分数">
    该指标衡量生成的摘要与专家创建的参考摘要之间的重叠程度。该指标主要关注召回率，适用于评估内容覆盖度。
  </Accordion>

  <Accordion title="BLEU 分数">
    虽然最初是为机器翻译开发的，但该指标可以适用于摘要任务。BLEU 分数衡量生成的摘要与参考摘要之间 n-gram 匹配的精确率。分数越高，表明生成的摘要包含与参考摘要相似的短语和术语。
  </Accordion>

  <Accordion title="上下文嵌入相似度">
    该指标涉及为生成的摘要和参考摘要创建向量表示（嵌入）。然后计算这些嵌入之间的相似度，通常使用余弦相似度。相似度分数越高，表明生成的摘要捕捉到了参考摘要的语义含义和上下文，即使确切措辞有所不同。
  </Accordion>

  <Accordion title="基于 LLM 的评分">
    该方法涉及使用 Claude 等 LLM 根据评分标准评估生成摘要的质量。评分标准可以根据您的具体需求进行定制，评估准确性、完整性和连贯性等关键因素。有关实施指导，请参阅

    [基于 LLM 评分的技巧](https://platform.claude.com/docs/zh-CN/test-and-evaluate/develop-tests#tips-for-llm-based-grading)

    。
  </Accordion>

  <Accordion title="人工评估">
    除了创建参考摘要外，法律专家还可以评估生成摘要的质量。虽然大规模进行这项工作既昂贵又耗时，但在部署到生产环境之前，通常会对少量摘要进行此类评估作为验证检查。
  </Accordion>
</AccordionGroup>

### 部署您的提示

以下是在将解决方案部署到生产环境时需要牢记的一些其他注意事项。

1. **确保无责任风险：** 了解摘要中的错误可能带来的法律影响，这些错误可能导致您的组织或客户承担法律责任。提供免责声明或法律声明，说明摘要由 AI 生成，应由法律专业人士审查。

2. **处理多种文件类型：** 本指南讨论了如何从 PDF 中提取文本。在现实世界中，文件可能有多种格式（例如 PDF、Word 文档和文本文件）。请确保您的数据提取管道能够转换您预期接收的所有文件格式。

3. **并行化对 Claude 的 API 调用：** 包含大量令牌的长文件可能需要长达一分钟的时间才能让 Claude 生成摘要。对于大型文件集合，您可能希望并行向 Claude 发送 API 调用，以便在合理的时间范围内完成摘要。请参阅 Anthropic 的[速率限制](https://platform.claude.com/docs/zh-CN/api/rate-limits#rate-limits)以确定可以并行执行的最大 API 调用数量。

***

## 提升性能

在复杂场景中，除了标准的[提示工程技术](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-engineering/overview)之外，考虑其他策略来提升性能可能会有所帮助。以下是一些高级策略：

### 执行元摘要以对长文件进行摘要

法律文件摘要通常涉及一次处理长文件或许多相关文件，以至于超出 Claude 的"context window"（上下文窗口）。您可以使用一种称为"meta-summarization"（元摘要）的分块方法来处理此用例。该技术涉及将文件分解为更小、更易管理的块，然后分别处理每个块。之后，您可以将每个块的摘要组合起来，创建整个文件的元摘要。

以下是如何执行元摘要的示例：

```python Python
# 初始化 Anthropic 客户端
client = anthropic.Anthropic()


def chunk_text(text, chunk_size=20000):
    return [text[i : i + chunk_size] for i in range(0, len(text), chunk_size)]


def summarize_long_document(
    text, details_to_extract, model="claude-opus-5", max_tokens=1000
):
    # 格式化要提取的详细信息，以便放入提示的上下文中
    details_to_extract_str = "\n".join(details_to_extract)

    # 遍历各个分块并逐一进行总结
    chunk_summaries = [
        summarize_document(
            chunk, details_to_extract, model=model, max_tokens=max_tokens
        )
        for chunk in chunk_text(text)
    ]

    final_summary_prompt = f"""

    You are looking at the chunked summaries of multiple documents that are all related.
    Combine the following summaries of the document from different truthful sources into a coherent overall summary:

    <chunked_summaries>
    {"".join(chunk_summaries)}
    </chunked_summaries>

    Focus on these key aspects:
    {details_to_extract_str}

    Provide the summary in bullet points nested within the XML header for each section. For example:

    <parties involved>
    - Sublessor: [Name]
    // Add more details as needed
    </parties involved>

    If any information is not explicitly stated in the document, note it as "Not specified". Do not preamble.
    """

    response = client.messages.create(
        model=model,
        max_tokens=max_tokens,
        system="You are a legal expert that summarizes notes on one document.",
        messages=[
            {"role": "user", "content": final_summary_prompt},
        ],
    )

    return next(block.text for block in response.content if block.type == "text")


long_summary = summarize_long_document(document_text, details_to_extract)
print(long_summary)
```

`summarize_long_document` 函数在前面的 `summarize_document` 函数基础上构建，将文件拆分为更小的块并分别对每个块进行摘要。

代码通过对原始文件中每 20,000 个字符的块应用 `summarize_document` 函数来实现这一点。然后将各个摘要组合起来，并根据这些块摘要创建最终摘要。

请注意，对于示例 PDF 而言，`summarize_long_document` 函数并非严格必需，因为整个文件可以容纳在 Claude 的上下文窗口内。然而，对于超出 Claude 上下文窗口的文件，或在将多个相关文件一起摘要时，它就变得至关重要。无论如何，这种元摘要技术通常能在最终摘要中捕捉到早期单一摘要方法所遗漏的其他重要细节。

### 使用摘要索引文件探索大型文件集合

使用 LLM 搜索文件集合通常涉及"retrieval-augmented generation"（检索增强生成），即 RAG。然而，在涉及大型文件或精确信息检索至关重要的场景中，基本的 RAG 方法可能不够。摘要索引文件是一种高级 RAG 方法，它提供了一种更高效的文件检索排序方式，所使用的上下文比传统 RAG 方法更少。在这种方法中，您首先使用 Claude 为语料库中的每份文件生成简洁的摘要，然后使用 Claude 对每份摘要与所提问题的相关性进行排序。有关此方法的更多详细信息（包括基于代码的示例），请查看[摘要 cookbook](https://platform.claude.com/cookbook/capabilities-summarization-guide) 中的摘要索引文件部分。

### 微调 Claude 以从您的数据集中学习

另一种提升 Claude 生成摘要能力的高级技术是"fine-tuning"（微调）。微调涉及在专门符合您法律摘要需求的自定义数据集上训练 Claude，确保 Claude 适应您的用例。以下是如何执行微调的概述：

1. **识别错误：** 首先收集 Claude 摘要表现不佳的实例——这可能包括遗漏关键法律细节、误解上下文或使用不恰当的法律术语。

2. **整理数据集：** 一旦识别出这些问题，就将这些有问题的示例汇编成数据集。该数据集应包括原始法律文件以及您更正后的摘要，确保 Claude 学习到期望的行为。

3. **执行微调：** 微调涉及在您整理的数据集上重新训练模型，以调整其权重和参数。这种重新训练有助于 Claude 更好地适应您法律领域的特定要求，提升其按照您的标准对文件进行摘要的能力。

4. **迭代改进：** 微调不是一次性的过程。随着 Claude 继续生成摘要，您可以迭代地添加其表现不佳的新示例，进一步完善其能力。随着时间的推移，这种持续的反馈循环将产生一个高度专门化于您法律摘要任务的模型。

<Tip>
  微调目前仅通过 Amazon Bedrock 提供。更多详细信息请参阅 

  [AWS 发布博客](https://aws.amazon.com/blogs/machine-learning/fine-tune-anthropics-claude-3-haiku-in-amazon-bedrock-to-boost-model-accuracy-and-quality/)

  。
</Tip>

<CardGroup cols={2}>
  <Card title="摘要 cookbook" icon="link" href="https://platform.claude.com/cookbook/capabilities-summarization-guide">
    查看如何使用 Claude 对合同进行摘要的完整实现代码示例。
  </Card>

  <Card title="引用 cookbook" icon="link" href="https://platform.claude.com/cookbook/misc-using-citations">
    探索引用 cookbook 示例，获取有关如何确保信息准确性和可解释性的指导。
  </Card>
</CardGroup>
