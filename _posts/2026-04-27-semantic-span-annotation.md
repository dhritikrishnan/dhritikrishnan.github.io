---
title: 'Span Annotation with LLMs: When Definitions Help and When They Hurt'
date: 2026-04-27
permalink: /posts/2026/04/semantic-span-annotation/
tags:
  - information extraction
  - large language models
  - named entity recognition
  - evaluation
---

Named entity recognition, semantic role labeling, argument mining, discourse analysis, PII detection. All of these are span annotation tasks at heart: find a contiguous stretch of text and assign it a label from some ontology. Yet they are studied as separate problems, with separate datasets, separate evaluation protocols, and separate community benchmarks. The result is that we know how well LLMs do on, say, CoNLL-2003 NER, but the picture for span annotation as a general capability is fragmented.

Our paper at SRW ACL 2026 is an attempt to put those threads back together. We unify five datasets spanning four domains into a single evaluation pipeline, then run eight frontier LLMs through three prompting configurations and look at where the wins and losses are.

The headline finding is simple. There are two regimes. On tasks with specialized label ontologies that probably did not appear in pretraining data, label definitions help dramatically (Claude Opus 4.6 jumps from 8.8% to 57.5% F1 on FiNER-139). On tasks with familiar pattern-based labels like PII, definitions consistently hurt across all models tested.

## Why the Field Is Siloed

The fragmentation has historical roots. Early NER systems used CRFs and BiLSTMs (Lafferty et al., 2001); early argument mining used SVMs with syntactic features (Moens et al., 2007); domain-specific systems used heuristics and expert knowledge. Each instantiation needed its own technical solution, so each developed its own benchmark community and its own way of measuring progress.

LLMs change the engineering picture but not the evaluation picture. The same model can be applied to all of these tasks by recasting them as text generation problems (Brown et al., 2020), but the field is still measured one task at a time. CrossNER (Liu et al., 2021) and UniversalNER (Zhou et al., 2024) take cross-domain swings but stay within NER specifically. Most existing work also varies the model architecture between settings, which makes it hard to ask the more basic question: holding the model fixed, what prompting strategy works where?

That is the question our pipeline is built to answer.

## The Benchmark

We unify five English-language datasets across four domains, chosen to span a range from pattern matching to ontology-heavy reasoning, and from short utterances to long documents.

| Subset | Task | Labels | Avg chars | Spans/doc |
|---|---|---|---|---|
| GoEmotions | Emotion detection | 28 | 68 | 1.2 |
| EBM-NLP | PICO element extraction | 17 | 1,580 | 17.8 |
| FiNER-139 | Numeric entity recognition | 139 | 239 | 0.3 |
| Synth PII Fin | PII detection | 29 | 1,322 | 6.8 |
| Nemotron-PII | PII/PHI detection | 55 | 981 | 8.4 |

The two extremes are worth flagging. **FiNER-139** (Loukas et al., 2022) has 139 XBRL entity types that depend on knowing what financial role a number plays. The labels are unlikely to be salient features of pretraining data. **Nemotron-PII** (Steier et al., 2025) has 55 PII/PHI categories that mostly correspond to familiar surface forms (names, addresses, account numbers, phone numbers). The labels are well-represented in pretraining.

Each source dataset is converted into a unified JSONL format with character-level offsets. Every annotated instance pairs a raw `doc_text` with a list of annotations identified by inclusive `start_idx` and exclusive `end_idx`. Conversion is automatic; no manual re-annotation is involved. We use strict Pydantic schema validation to verify field types, boundary validity (`start_idx < end_idx ≤ len(doc_text)`), and label membership.

## The Pipeline

![ssa_baseline pipeline architecture](/images/blog/ssa/pipeline.png)

The pipeline, called `ssa_baseline`, takes a document and a label list, runs them through one of three prompting configurations, and feeds the result to the LLM. The model produces an XML-tagged output. A custom parser then string-matches the tagged spans back against the original document to recover exact character boundaries, which are scored against gold annotations using span F1.

The XML approach matters more than it might look. A common alternative is to have the model return a JSON array of entity strings and then reverse-search them in the document. That fails when the same string appears multiple times with different semantic roles. By having the model reproduce the document verbatim with inline XML tags, we preserve the local context of every span and avoid ambiguous reverse lookups.

The core instruction is:

> Output the verbatim copy of the document (same characters, spacing, punctuation) enriched with XML tags as appropriate. Use EXACTLY the label names shown above as XML tag names. Do not output anything else (no explanation, preamble, or commentary). Tags cannot overlap but they can nest if appropriate.

A concrete example from EBM-NLP (Nye et al., 2018) makes the failure modes visible. Given the input:

```
Randomized trial of intensive early intervention for children with pervasive
developmental disorder.
```

GPT-OSS-120B under the zero-shot baseline produces:

```
Randomized trial of intensive early intervention for children with
<P-Condition>pervasive developmental disorder</P-Condition>.
```

while the gold annotation is:

```
Randomized trial of <I-Educational>intensive early intervention</I-Educational>
for <P-Age>children</P-Age> with <P-Condition>pervasive developmental
disorder</P-Condition>.
```

The model correctly tags the medical condition but misses the age and intervention spans entirely. We will come back to this pattern.

## Three Prompting Configurations

Each model is evaluated under three configurations on the same 100 randomly sampled hold-out documents per dataset:

1. **Zero-Shot Baseline.** System instruction, label list, raw document text. No definitions, no examples.
2. **Definition-Augmented.** Same prompt plus a natural-language definition for every label. Definitions were generated by prompting an LLM with each label name and its domain context, then manually reviewed against the original dataset documentation.
3. **Few-Shot.** Same prompt as definition-augmented, plus three full document and gold-output exemplars showing correct span boundaries.

All inference uses greedy decoding ($T = 0.0$) for determinism. After inference we apply one piece of postprocessing: predicted spans with the same label separated by fewer than 10 characters are merged into a single span.

## Evaluation Metrics

A predicted span $p = (p_s, p_e, p_\ell)$ and a gold span $g = (g_s, g_e, g_\ell)$ carry start offset, end offset, and label. We score predictions with two match predicates.

**Exact Match.** All three fields must agree:

$$\text{match}_{\text{exact}}(p, g) \;=\; \mathbb{1}\!\left[\, p_s = g_s \;\land\; p_e = g_e \;\land\; p_\ell = g_\ell \,\right]$$

**Relaxed Match.** Label must agree; offsets must fall within a length-dependent tolerance $\tau$:

$$\text{match}_{\tau}(p, g) \;=\; \mathbb{1}\!\left[\, |p_s - g_s| \le \tau \;\land\; |p_e - g_e| \le \tau \;\land\; p_\ell = g_\ell \,\right]$$

where $L = g_e - g_s$ is the gold span length and

$$\tau \;=\; \min\!\left(6,\; \max\!\left(1,\; \left\lfloor \tfrac{L}{100} \right\rfloor + 1 \right)\right).$$

Short spans get a 1-character tolerance; long ones cap out at 6. From these per-pair predicates we compute standard span-level $F_1 = 2PR / (P + R)$ over the predicted and gold sets.

The gap between exact and relaxed F1 tells you whether failures come from missing spans or from imprecise boundaries. That distinction turns out to matter.

## Results

![Model-averaged Exact Match F1 across configurations](/images/blog/ssa/results-bar.png)

The macro-average across all five datasets, averaged over all eight models, looks clean:

| Configuration | Macro-avg Exact Match F1 |
|---|---|
| Zero-shot baseline | 23.1% |
| + Definitions | 32.2% |
| + 3-Shot | 33.5% |

That tells one story: more context helps. But the aggregate hides what is actually going on. Two specific contrasts from the full table:

- On **FiNER-139**, Claude Opus 4.6 goes from 8.8% F1 (zero-shot) to 57.5% F1 (definitions). GPT-5-Mini goes from 1.2% to 35.7%. Sonnet 4.6 starts at 0.0%. Definitions provide gains of tens of points across most of the eight models.
- On **Synth PII Fin**, every one of the eight models scores its best under the zero-shot baseline. Adding definitions reduces performance for all of them. Nemotron-PII numbers are roughly flat across configurations, with most models changing by less than 1 F1 point in either direction.

Same pipeline, same prompts, same models. Opposite effect of definitions.

## Two Regimes

The split tracks how familiar the label set is to the model.

**Ontology-Heavy Tasks.** FiNER-139 and EBM-NLP demand mapping spans to specialized labels (XBRL financial roles; PICO clinical elements). Zero-shot performance is near zero for most models on FiNER-139. Adding definitions yields large gains because the bottleneck is not span localization but label disambiguation: the model knows there is something interesting in the text but does not know which of 139 categories it belongs to. Defining the categories closes that gap. This echoes findings from GoLLIE (Sainz et al., 2024) on the importance of detailed annotation guidelines, and from Kim et al. (2024) on nested NER in specialized domains.

**Pattern-Based Tasks.** PII detection labels are common in pretraining data, and many entities follow recognizable surface forms. Zero-shot prompting already performs competitively. Adding definitions consistently hurts. Two explanations are compatible with the data. First, explicit definitions can introduce conflicting signals that override otherwise correct extractions. Second, the model may be doing surface pattern recall (possibly because similar data appeared in pretraining), and definitions push it off that recall track. The PII datasets used here are synthetic, which partly mitigates the second concern, but FiNER-139 and EBM-NLP have been publicly available for years and may have appeared in pretraining corpora.

One more observation about the *kind* of error. The mean gap between exact-match and relaxed-match F1, across all tasks and configurations, is under 3 percentage points. When a model identifies a span, it gets the boundaries right. The dominant failure mode is span omission, not boundary imprecision. Going back to the EBM-NLP example: the model did not pick wrong boundaries for *children* or *intensive early intervention*, it simply did not tag them.

## What This Means

A few takeaways:

**Prompting strategy should depend on label familiarity, not on task difficulty in the abstract.** "Add definitions" is not a general-purpose improvement. On tasks where the labels match pretraining patterns, definitions can degrade performance. On tasks where the labels are specialized vocabulary the model has never seen used as labels, definitions are the single most cost-effective intervention we tested.

**Omission, not imprecision, is the failure mode to target.** Future work on span extraction with LLMs should focus on getting models to *attempt* more spans (recall) rather than on tighter boundary prediction. Chain-of-thought prompting that explicitly scans for each label class, or retrieval-augmented prompting that surfaces label-relevant examples for each document, are natural candidates.

**Unified evaluation across domains is necessary to see these patterns.** The same model behaves in opposite ways on FiNER-139 and Nemotron-PII; that contrast is invisible if you only benchmark within a single domain. The fragmentation of span-extraction research has been a methodological liability.

Paper: *Semantic Span Annotation: An Exploratory Study of LLM Annotation*, SRW ACL 2026.

## References

1. Brown, T., Mann, B., Ryder, N., Subbiah, M., et al. *Language Models Are Few-Shot Learners.* NeurIPS 2020.
2. Demszky, D., Movshovitz-Attias, D., Ko, J., Cowen, A., Nemade, G., Ravi, S. *GoEmotions: A Dataset of Fine-Grained Emotions.* ACL 2020.
3. Kim, H., Kim, J., Kim, H. *Exploring Nested Named Entity Recognition with Large Language Models: Methods, Challenges, and Insights.* EMNLP 2024.
4. Lafferty, J.D., McCallum, A., Pereira, F.C.N. *Conditional Random Fields: Probabilistic Models for Segmenting and Labeling Sequence Data.* ICML 2001.
5. Liu, Z., Yan, X., Yu, T., Dai, W., Ji, Z., Madotto, A., Fung, P. *CrossNER: Evaluating Cross-Domain Named Entity Recognition.* AAAI 2021.
6. Loukas, L., Fergadiotis, M., Chalkidis, I., et al. *FiNER: Financial Numeric Entity Recognition for XBRL Tagging.* ACL 2022.
7. Moens, M.F., Boiy, E., Mochales Palau, R., Reed, C. *Automatic Detection of Arguments in Legal Texts.* ICAIL 2007.
8. Nye, B., Li, J.J., Patel, R., Yang, Y., Marshall, I., Nenkova, A., Wallace, B. *A Corpus with Multi-Level Annotations of Patients, Interventions and Outcomes to Support Language Processing for Medical Literature.* ACL 2018.
9. Sainz, O., García-Ferrero, I., Agerri, R., Lopez de Lacalle, O., Rigau, G., Agirre, E. *GoLLIE: Annotation Guidelines Improve Zero-Shot Information Extraction.* ICLR 2024.
10. Steier, A., Manoel, A., Haushalter, A., Van Segbroeck, M. *Nemotron-PII: Synthesized Data for Privacy-Preserving AI.* NVIDIA, 2025.
11. Tjong Kim Sang, E.F., De Meulder, F. *Introduction to the CoNLL-2003 Shared Task: Language-Independent Named Entity Recognition.* CoNLL 2003.
12. Watson, A., Meyer, Y., Van Segbroeck, M., et al. *Synthetic PII Finance Multilingual Dataset.* Gretel AI, 2024.
13. Zhou, W., Zhang, S., Gu, Y., Chen, M., Poon, H. *UniversalNER: Targeted Distillation from Large Language Models for Open Named Entity Recognition.* ICLR 2024.
