---
title: 减少幻觉
url: https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/reduce-hallucinations
description: 通过允许表达不确定性、以直接引用为依据生成回答，以及使用引文验证论断，最大限度地减少 Claude 输出中的幻觉。
---

即使是像 Claude 这样最先进的语言模型，有时也会生成与事实不符或与给定上下文不一致的文本。这种现象被称为"hallucination"（幻觉），可能会削弱您的 AI 驱动解决方案的可靠性。 本指南将探讨最大限度减少幻觉并确保 Claude 的输出准确可信的技巧。

## 基本的幻觉最小化策略

* **允许 Claude 说"我不知道"：** 明确允许 Claude 承认不确定性。这一简单的技巧可以大幅减少虚假信息。

<Accordion title="示例：分析一份并购报告">
  ```text User wrap
  As our M&A advisor, analyze this report on the potential acquisition of AcmeCo by ExampleCorp.

  <report>
  {{REPORT}}
  </report>

  Focus on financial projections, integration risks, and regulatory hurdles. If you're unsure about any aspect or if the report lacks necessary information, say "I don't have enough information to confidently assess this."
  ```
</Accordion>

* **使用直接引用作为事实依据：** 对于涉及长文档（>20k 令牌）的任务，请让 Claude 在执行任务之前先逐字提取引文。这会使其回答以实际文本为依据，从而减少幻觉。

<Accordion title="示例：审核一份数据隐私政策">
  ```text User wrap
  As our Data Protection Officer, review this updated privacy policy for GDPR and CCPA compliance.
  <policy>
  {{POLICY}}
  </policy>

  1. Extract exact quotes from the policy that are most relevant to GDPR and CCPA compliance. If you can't find relevant quotes, state "No relevant quotes found."

  2. Use the quotes to analyze the compliance of these policy sections, referencing the quotes by number. Only base your analysis on the extracted quotes.
  ```
</Accordion>

* **使用引文进行验证**：让 Claude 为其每一项论断引用原文和来源，使其回答可被审核。您还可以让 Claude 在生成回答后，通过查找支持性引文来验证每一项论断。如果找不到引文，它必须撤回该论断。

<Accordion title="示例：起草一份产品发布新闻稿">
  ```text User wrap
  Draft a press release for our new cybersecurity product, AcmeSecurity Pro, using only information from these product briefs and market reports.
  <documents>
  {{DOCUMENTS}}
  </documents>

  After drafting, review each claim in your press release. For each claim, find a direct quote from the documents that supports it. If you can't find a supporting quote for a claim, remove that claim from the press release and mark where it was removed with empty [] brackets.
  ```
</Accordion>

***

## 高级技巧

* **"Chain-of-thought"（思维链）验证**：要求 Claude 在给出最终答案之前逐步解释其推理过程。这可以揭示错误的逻辑或假设。

* **"Best-of-N"（N 选最优）验证**：使用同一提示多次运行 Claude 并比较输出结果。各输出之间的不一致可能表明存在幻觉。

* **迭代优化**：将 Claude 的输出用作后续提示的输入，要求它验证或扩展先前的陈述。这可以发现并纠正不一致之处。

* **外部知识限制**：明确指示 Claude 仅使用所提供文档中的信息，而不使用其通用知识。

<Note>
  请记住，虽然这些技巧能显著减少幻觉，但并不能完全消除幻觉。请务必验证关键信息，尤其是在涉及高风险决策时。
</Note>
