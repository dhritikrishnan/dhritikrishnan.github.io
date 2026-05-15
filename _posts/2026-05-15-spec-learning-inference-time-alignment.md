---
title: 'Spec Learning: Compiling Preference Pairs into Inference-Time Prompts'
date: 2026-05-15
permalink: /posts/2026/05/spec-learning-inference-time-alignment/
tags:
  - alignment
  - large language models
  - preference learning
  - prompt optimization
  - dpo
---

Most ways to steer a language model toward a desired behavior fall into one of two camps. You either write a prompt and iterate on it by inspecting outputs (manual prompt engineering), or you collect a large preference dataset and fine-tune the model with something like DPO (Direct Preference Optimization). Both have well-known costs. Prompt engineering is brittle, takes specialized skill, and usually leaves the user under-evaluating the output. Preference fine-tuning needs thousands of labeled pairs and a training pipeline, and it bakes the alignment signal into weights you cannot inspect.

There is a gap in the middle that has been largely unexplored: what happens if you ask the user for only the bare minimum (a brief instruction and a handful of preference judgments) and treat alignment as a *compilation* problem? Use an LLM to read those preferences, extract the principles that explain them, and assemble those principles into a natural-language system prompt. No weights move. The compiled prompt steers the base model at inference time.

This is what our paper, *Towards Spec Learning: Inference-Time Alignment from Preference Pairs* (under review at NeurIPS 2026), explores. Our central claim is that around 20 preference pairs already contain enough signal to compile a natural-language prompt that matches gradient-based preference tuning trained on a corpus fifty times larger, while leaving the inference model's weights unchanged.

## Three Regimes for Shaping LLM Behavior

The cleanest way to think about spec learning is to ask where the alignment signal lives.

![Three regimes: signal in user, signal in weights, signal in input](/images/blog/spec-learning/three-regimes.png)

**Manual prompt engineering** keeps the signal in the user. The user iterates the prompt and inspects responses; no preference data is captured anywhere. The literature on non-expert prompting is pretty discouraging about how often this loop actually converges.

**Preference fine-tuning** moves the signal into the weights. You collect roughly a thousand preference pairs, run DPO or one of its variants, and produce a specialized model. The result generalizes well but is opaque: you cannot ask the model *why* it now prefers one response over another, and you cannot port the behavior to a different base model without retraining.

**Spec learning** puts the signal in the input. About 20 preference pairs feed an offline compiler that produces a natural-language specification $s$. At inference, $s$ is composed with the user query and consumed by the same unchanged base model. The artifact is a paragraph of text. You can read it, edit it, and apply it to any model that accepts a system prompt.

The reason all three regimes coexist in practice is that they make different trade-offs across three axes: how much labeled data they need, how much compute they consume, and how transparent the resulting policy is. Spec learning's bet is that for many domains the underlying preference signal is concentrated enough that 20 pairs are enough to recover it, and that recovering it as text rather than as gradients is worth doing.

## The Pipeline

Let a preference pair be $(x, y^+, y^-)$, where $x$ is an instruction and $y^+, y^-$ are the chosen and rejected responses. Given a set of $N$ such pairs $\mathcal{D} = \{(x_i, y^+_i, y^-_i)\}_{i=1}^N$, compilation proceeds in four stages.

![Spec compilation pipeline: Propose, Compress, Validate, Synthesize](/images/blog/spec-learning/pipeline.png)

**Propose.** A proposer LLM $\mathcal{P}$ reads the pairs and emits candidate principles:

$$\{p_1, \ldots, p_M\} = \mathcal{P}(\mathcal{D})$$

Each $p_j$ is a natural-language rule explaining why $y^+$ is preferred over $y^-$. The proposer sees all pairs and tries to articulate what makes the chosen responses better. A typical principle might be: *"Provide specific food recommendations that align with the request and avoid non-food items"* or *"Use correct data types, avoid invalid method calls, and ensure proper casting of user input."*

**Compress.** The candidate principles are clustered by semantic similarity and deduplicated. Multiple proposer passes will surface near-equivalent rules in slightly different language; compression collapses these into canonical representatives.

**Validate.** Surviving principles are tested on held-out pairs using a swap-and-average scoring step. For each principle, the validator checks whether $y^+$ satisfies it more than $y^-$, then swaps response positions and averages to cancel position bias. Principles are ranked by a composite of two things: prevalence (how often they apply) and accuracy (how reliably they discriminate). Only the top $K$ principles survive into the next stage.

**Synthesize.** A synthesizer $\mathcal{S}$ assembles the ranked principles into a specification:

$$s = \mathcal{S}(p_1, \ldots, p_K)$$

The synthesizer matters more than it sounds. We compare two variants. **JANUS** produces a rich, persona-style paragraph framing the principles as a coherent behavioral policy. **BULLETS** produces a compact numbered rule list from the same principles. JANUS consistently wins, which we read as evidence that *prompt register* (how the rules are framed) matters as much as which rules are picked.

The whole compilation runs offline. The output artifact is a single paragraph of text, typically 150 to 300 words, that prepends every query at inference time.

## Design Decisions

The pipeline has four moving parts: the number of pairs $N$, the selection strategy $\sigma$ that picks pairs from the corpus, the synthesizer $\mathcal{S}$, and the proposer $\mathcal{P}$. We ran ablations on each.

### Selection: Random Is Hard to Beat

To allow selection strategies beyond random sampling, we annotate each response with a scalar quality score from the DEITA quality scorer, giving us $(x, y^+, y^-, q^+, q^-)$ for each pair. From these we define five strategies: `RANDOM`, `HIGH_QUALITY` (top by $q^+$), `LOW_QUALITY` (bottom by $q^+$), `LARGE_GAP` (top by $|q^+ - q^-|$), and `SMALL_GAP` (bottom by $|q^+ - q^-|$).

Crossed with the two synthesizers, this gives ten configurations. Random sampling paired with JANUS achieves the highest macro win rate (0.698), beating every curated strategy. Curated selection in fact *hurts* on some datasets: `HIGH_QUALITY` and `LOW_QUALITY` filtering both degrade performance on PsyCoPref and Code-Security. The intuition we land on is that filtering by scalar quality narrows the variety of preference signal the proposer sees, which makes the induced principles less robust.

We adopt `RANDOM` + `JANUS` as the default.

### N-Sweep: Twenty Pairs Is the Sweet Spot

Sweeping $N \in \{10, 15, 20, 30, 40, 50\}$ with the random/JANUS defaults gives the following macro win rates: 0.639, 0.630, 0.650, 0.581, 0.623, 0.623. The peak sits at $N = 20$. More pairs do not reliably help. Even $N = 10$ reaches 0.664.

This is consistent with the assumption that the preference signal in well-defined domains is low-dimensional: there is not much for a larger dataset to find that twenty random pairs do not already contain. The proposer saturates fast.

### Proposer Scaling: A 31B Model Is Enough

We tried three proposers of increasing scale: Gemma 4 31B, DeepSeek V4 Flash (284B), and Kimi K2.6 (1T). Macro spec-vs-DPO win rates: 0.69, 0.69, 0.71. Every proposer beats DPO on all seven datasets. The largest proposer extracts marginally better principles, but the smallest is already enough to dominate.

This matters for cost. Compilation needs only inference calls (no gradient computation), and a 31B-parameter proposer makes the pipeline cheap.

## Calibrating the Judge

All of this depends on a judge LLM that can reliably score one response against another. LLM-as-a-judge is a known-imperfect paradigm: judges show family-aligned reward bias (they over-credit responses written in their own family's style), position bias (they favor the first or last response shown), and self-inconsistency across runs.

We control these in three ways.

**Family disjointness.** The primary judge (GLM-5.1) is from a different model family than every proposer and every policy model in the evaluation. This rules out the simplest form of reward bias.

**Three-pass position rotation.** Each pairwise comparison is decided by a three-pass forced-binary verdict with positions rotated across passes. The majority across passes is the recorded winner. Unparseable verdicts are dropped.

**Gold-recovery calibration.** Before adopting a judge for the main evaluation, we ran it on gold-labeled held-out preference pairs from three datasets that span our task range: Math-DPO ($n=20$), Truthy-DPO ($n=20$), Stack-Exchange ($n=40$). GLM-5.1 hit 0.70, 0.60, and 0.65 respectively, all clearly above chance. A closed-source frontier alternative we evaluated reached 0.80 on Truthy-DPO and 0.65 on Stack-Exchange but collapsed to 0.50 on Math-DPO, indistinguishable from chance. We disqualified it from the primary slot.

The judge protocol is not a side detail. If it leaks bias into the headline numbers, every claim downstream is suspect.

## Do the Principles Actually Discriminate?

Before looking at the spec-vs-DPO comparison, it is worth asking a more basic question: do the compiled principles actually encode the preference signal, or are they just plausible-sounding rules that the judge happens to like?

We test this directly. For each dataset, we prompt the judge with the compiled principles $\{p_1, \ldots, p_K\}$ and ask it to count how many each response in a held-out pair satisfies. Let $c(y)$ denote this count. The response with higher $c$ is predicted as preferred, and we compare against the same judge without access to the principles.

The result: guideline grounding improves average accuracy across five datasets from 0.773 to 0.806 (+3.3 points), with the largest gain on Code-Security (+21.7 points), a dataset where surface-level textual cues are least informative. On every dataset, the average count satisfied by chosen responses exceeds the average count for rejected responses, $\bar{c}(y^+, s) > \bar{c}(y^-, s)$. This means the principles are not just stylistic ornamentation. They consistently fire more on preferred outputs even when the underlying classification problem is hard.

So the specs are not merely effective prompts. They are written embodiments of the preference signal that produced them, and they discriminate between preferred and dispreferred outputs on their own.

## Headline Results

The main comparison fixes the configuration to $(\mathcal{D}_{20}, \sigma = \texttt{RANDOM}, \mathcal{S} = \texttt{JANUS}, \mathcal{P} = \text{best per dataset})$. The DPO baseline is trained on $N = 1{,}000$ pairs from the same training partition with a rank-32 LoRA adapter, AdamW at learning rate $5 \times 10^{-6}$, and three epochs (full hyperparameters in the paper). We draw 100 held-out instructions per dataset and compare three arms: baseline (the unmodified base model), spec (the base model with $s$ as a system prompt), and DPO (the fine-tuned variant).

| Dataset | $\mathcal{P}$ | spec vs. ctl | spec vs. DPO | DPO vs. ctl |
|---|---|---|---|---|
| Stack-Exchange | Gemma 4 31B | 0.66 | **0.83** | 0.71 |
| Code-Pref | Kimi K2.6 | 0.71 | **0.82** | 0.67 |
| Truthy-DPO | DeepSeek V4 Flash | 0.64 | **0.80** | 0.61 |
| Math-DPO | Kimi K2.6 | 0.78 | **0.75** | 0.79 |
| Code-Security | Kimi K2.6 | 0.73 | **0.73** | 0.68 |
| PsyCoPref | DeepSeek V4 Flash | 0.84 | **0.71** | 0.81 |
| HH-Helpful | Gemma 4 31B | 0.55 | 0.58† | 0.70 |
| **Macro mean** | | **0.70** | **0.75** | **0.71** |

The compiled spec beats DPO on all seven datasets, with a macro mean win rate of 0.75. The spec arm consumed 50× fewer preference pairs (20 vs. 1,000) and required no weight updates. Length-controlled win rates shift each cell by at most 6 points and never flip an arm's win or loss, so length is not the driver. Wilson 95% confidence intervals on five of the six wide-sample (n=100) cells sit strictly above 0.5. HH-Helpful is the exception (interval includes 0.5, marked †), and that exception turns out to be the most informative cell in the table.

## When It Breaks

The margin varies systematically with how well-defined the underlying task is.

Datasets with a tight, domain-specific preference signal yield the strongest results. Stack-Exchange (0.83), Code-Pref (0.82), Truthy-DPO (0.80), and Math-DPO (0.75) all have preferences that compress into a recognizable set of behavioral rules. *"Provide a direct answer instead of an anecdotal example."* *"Use correct data types."* *"Acknowledge ignorance instead of confabulating."* A paragraph of text can hold these.

As the task definition broadens, the advantage narrows. Code-Security (0.73) and PsyCoPref (0.71) sit in the middle. And HH-Helpful (0.58, CI includes 0.5) yields the smallest margin. HH-Helpful is Anthropic's helpful-only subset of HH-RLHF, where preferences span arbitrary user requests with no single governing task. We swept $N \in \{10, 15, 20, 30, 40, 50\}$ on HH-Helpful and the spec arm never exceeds 0.52 against the base model for any $N$. The ceiling is structural, not a budget issue.

DPO does not have this problem. It hits 0.70 on HH-Helpful against the baseline because gradient-based methods do not need to articulate what they have learned. They can distribute the preference signal across thousands of weights in ways that no single paragraph can capture.

This is the cleanest contrast in the paper, and it sharpens what spec learning is and is not. The method assumes three things:

1. The preference signal in the target domain admits a compact set of explicit principles.
2. The proposer and judge LLMs are strong enough to find and verify those principles from limited data.
3. The inference model can act on detailed natural-language instructions.

Assumption (1) is the one that breaks on HH-Helpful. The other two hold across the board in our setup.

## What This Says

A few takeaways.

**Preference compression is real.** The fact that 20 random pairs beat 1,000 pairs of DPO on six of seven datasets is not a marginal finding. It says the alignment signal in those domains is low-dimensional in a precise sense: it fits in a paragraph. This is not obvious a priori. The default assumption in the field has been that more data is always better.

**Where the signal lands matters.** Compiling preferences into text rather than into weights gives you a portable, inspectable, editable artifact. You can take a spec compiled with a Gemma 4 proposer and apply it to a Qwen 2.5 policy without retraining. You can read a spec, find a clause you disagree with, and delete it. Neither is possible with a DPO checkpoint.

**Synthesizer register matters as much as principle selection.** Two synthesizers seeded with the same ranked principles produce very different downstream performance. JANUS, which frames the principles as a coherent persona, consistently outperforms BULLETS, which lists them as numbered rules. The principles are the same; only the prosody changes. Whatever the policy LLM is doing when it reads a system prompt, it is paying attention to register, not just content.

**The failure mode is informative.** Spec learning's failure on HH-Helpful is exactly what you would predict if the method assumed compressibility, and it shows up as a cap at 0.52 against baseline that no amount of additional data can move. This is a cleaner separation between method and method-doesn't-apply than most alignment papers manage to demonstrate.

There are real costs we glossed over. Compiled specs prepend every query, so per-call token cost scales with traffic in a way DPO weights do not. Specs are static, which means distribution shift requires recompiling rather than retraining. Plain-text guardrails are more vulnerable to prompt injection than weight-level constraints, and the low compute overhead of compilation lowers the barrier for adversaries to produce malicious specs too. None of these defeat the method, but they shape where it is appropriate to deploy.

We recommend spec learning for narrow domains where the preference signal compresses into a few natural-language principles, deployed with a human in the loop: a brief instruction and a small set of judgments are compiled once into a specification, which then conditions the base model at inference. For broad, heterogeneous preference surfaces like HH-Helpful, gradient-based methods still win.

Paper: *Towards Spec Learning: Inference-Time Alignment from Preference Pairs*. Under review at NeurIPS 2026.

## References

1. Rafailov, R., Sharma, A., Mitchell, E., Ermon, S., Manning, C.D., Finn, C. *Direct Preference Optimization: Your Language Model Is Secretly a Reward Model.* NeurIPS 2023.
2. Findeis, A., Kaufmann, T., Hüllermeier, E., Albanie, S., Mullins, R. *Inverse Constitutional AI: Compressing Preferences into Principles.* [arXiv:2406.06560](https://arxiv.org/abs/2406.06560), 2025.
3. Petridis, S., Wedin, B.D., Wexler, J., Pushkarna, M., Donsbach, A., Goyal, N., Cai, C.J., Terry, M. *ConstitutionMaker: Interactively Critiquing Large Language Models by Converting Feedback into Principles.* IUI 2024.
4. Zamfirescu-Pereira, J.D., Wong, R.Y., Hartmann, B., Yang, Q. *Why Johnny Can't Prompt: How Non-AI Experts Try (and Fail) to Design LLM Prompts.* CHI 2023.
5. Lee, S., Park, S.H., Kim, S., Seo, M. *Aligning to Thousands of Preferences via System Messages.* NeurIPS 2024.
6. Liu, W., Zeng, W., He, K., Jiang, Y., He, J. *What Makes Good Data for Alignment? A Comprehensive Study of Automatic Data Selection in Instruction Tuning.* 2024.
7. Bai, Y., Jones, A., Ndousse, K., Askell, A., et al. *Training a Helpful and Harmless Assistant with Reinforcement Learning from Human Feedback.* [arXiv:2204.05862](https://arxiv.org/abs/2204.05862), 2022.
8. Panickssery, A., Bowman, S.R., Feng, S. *LLM Evaluators Recognize and Favor Their Own Generations.* NeurIPS 2024.
9. Wang, P., Li, L., Chen, L., Cai, Z., Zhu, D., et al. *Large Language Models Are Not Fair Evaluators.* ACL 2024.
10. Dubois, Y., Galambosi, B., Liang, P., Hashimoto, T.B. *Length-Controlled AlpacaEval: A Simple Way to Debias Automatic Evaluators.* 2024.
11. Ethayarajh, K., Xu, W., Muennighoff, N., Jurafsky, D., Kiela, D. *KTO: Model Alignment as Prospect Theoretic Optimization.* [arXiv:2402.01306](https://arxiv.org/abs/2402.01306), 2024.
12. Meng, Y., Xia, M., Chen, D. *SimPO: Simple Preference Optimization with a Reference-Free Reward.* NeurIPS 2024.
13. Hu, E.J., Shen, Y., Wallis, P., Allen-Zhu, Z., Li, Y., Wang, S., Wang, L., Chen, W. *LoRA: Low-Rank Adaptation of Large Language Models.* ICLR 2022.
14. Zheng, L., Chiang, W.-L., Sheng, Y., Zhuang, S., et al. *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena.* [arXiv:2306.05685](https://arxiv.org/abs/2306.05685), 2023.
