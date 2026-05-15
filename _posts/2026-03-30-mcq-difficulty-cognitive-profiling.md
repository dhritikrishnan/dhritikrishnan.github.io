---
title: 'Modeling Learner Heterogeneity for MCQ Difficulty Prediction'
date: 2026-03-30
permalink: /posts/2026/03/mcq-difficulty-cognitive-profiling/
tags:
  - assessment
  - item response theory
  - large language models
  - psychometrics
---

Predicting how hard a question is matters for almost everything you might do with an assessment: calibrating items, building adaptive tests, deciding what to teach next. The standard approach is item response theory (IRT), which fits per-item difficulty parameters from a large pool of real student responses. That works well in the steady state but breaks for new items, since you need pretesting data before you can deploy them. This is the cold-start problem, and it is the practical reason people have started using language models as stand-ins for students.

Recent work tries to skip pretesting by predicting difficulty directly from question text, learned representations, or LLM-simulated student responses. The simulation approach is the most interesting: if a language model can convincingly play the role of a typical student, you can estimate how often each option gets selected without running real pretests. Feng et al. (AIED 2025) sampled student ability levels from a standard normal distribution and used those samples to drive per-option selection likelihoods. SMART (Scarlatos et al., EMNLP 2025) aligned simulated students to IRT ability parameters via direct preference optimization.

These methods share an assumption that does not match how learners actually fail. They treat student ability as one-dimensional and treat errors as varying mostly in *frequency*: a weaker student gets more questions wrong than a stronger one. In reality, students with similar overall accuracy still make systematically different *kinds* of mistakes. A learner who has memorized the rules but lacks number sense fails very differently from one who reasons proportionally but stumbles on fraction arithmetic. If you collapse these into a scalar ability parameter, you lose exactly the structure that determines why an item is hard.

Our paper at AIED 2026 starts from this gap. The hypothesis is that student errors are structured around discrete cognitive profiles, and that conditioning simulated students on these profiles rather than on a scalar ability gives better difficulty estimates.

## The Pipeline

![Pipeline diagram](/images/blog/mcq/pipeline.png)

Four stages, plus a ground-truth estimation step:

0. Fit a 2PL-IRT model on real student responses from the EEDI dataset (Wang et al., 2020). This gives the difficulty parameter β for each item, which is what we predict.
1. Cluster students into latent classes via latent class analysis (LCA) on their response data. Each class becomes a *persona*.
2. For each item and each persona, prompt an LLM to predict the per-option response distribution that students of that persona would produce.
3. Aggregate the per-persona probabilities into a feature vector.
4. Predict β with ridge regression.

Steps 1 and 2 are where the heterogeneity assumption enters. The rest is standard.

## Finding the Personas

We use EEDI tasks 3 and 4: 948 mathematics MCQs, 4,918 students, 1,382,728 total interaction records. For psychometric profiling we keep questions with at least 50 responses and students with at least 10 attempts, then fit an LCA model with a binary measurement model over correctness vectors.

How many clusters? We sweep $k$ from 2 to 10 and track BIC and AIC. BIC reaches a global minimum at $k = 5$; AIC keeps dropping but with diminishing returns. We take the BIC minimum.

![Model selection: BIC and AIC across k](/images/blog/mcq/bic-aic.png)

The five clusters are not just statistical artifacts. To interpret them we compute, for each cluster $c$ and question $i$, a deviation score showing how much that cluster outperformed or underperformed the average:

$$\delta_i^{(c)} = a_i^{(c)} - \frac{1}{K} \sum_{k=1}^K a_i^{(k)}$$

For each cluster we pick its top 5 strengths (largest positive deviation) and top 5 weaknesses (largest negative). We hand those items, together with their topic labels and accuracy figures, to Claude Opus 4.5 and ask it to write a short description capturing the cognitive gap that separates the cluster's strengths from its weaknesses.

| Persona | Core strength | Core weakness |
|---|---|---|
| Rule Memorizer | Applies formulas accurately | Lacks number sense; magnitude misconceptions |
| Procedural Calculator | Strong single-step arithmetic | Fails inverse and multi-step reasoning |
| Abstract Reasoner | Logical and proportional reasoning | Weak basic arithmetic automaticity |
| Conceptual Reasoner | Strong "why" intuition | Procedural fluency on the "how" |
| Fraction Calculator | Standard fraction equations | Proportional reasoning in real contexts |

The Conceptual Reasoner is a useful case to think about. They can reason about why dividing by a fraction increases a number, but cannot actually execute $2/9 \div 3/4$. That is the kind of "right logic, wrong answer" error a scalar ability model has no language for.

![Persona spotlight: the Conceptual Reasoner](/images/blog/mcq/persona-spotlight.png)

## Simulating Responses

With the personas in hand: for each item, what would each persona do? We feed Claude 3.7 Sonnet the question image (EEDI questions are stored as images, so we use the vision capability directly) together with the persona's name and description, and ask the model to estimate the probability of selecting each of the four answer options *as that student type*. The instruction matters. We are not asking the model to solve the problem optimally. We are asking it to predict what a learner with that specific cognitive profile would select.

For each item this produces a $5 \times 4$ matrix of option probabilities, one row per persona.

> A custom figure would help here. A small heatmap showing the $5 \times 4$ probability matrix for one example item, with the correct option highlighted, would make the abstract "K × 4 matrix" concrete and let readers see how different personas concentrate probability mass on different distractors. Happy to put this together if you want to pull data from one item in the experiments.

## Predicting Difficulty

For each persona $c$ we extract the probability assigned to the correct option, $p^{(c)}_{i,\text{correct}}$. Across the five personas we compute the mean, variance, and range of these correct-option probabilities. The item's mathematical topic (Number, Algebra, Geometry and Measure) is one-hot encoded. Numeric features are standardized.

The predictor is ridge regression with cross-validated regularization strength $\alpha \in \{0.1, 1, 10, 100, 500\}$. We use five-fold cross-validation on the 900-item estimation set.

We deliberately kept the predictor simple. The point of the experiment is to test whether persona-conditioned features carry signal about difficulty, not to squeeze the last point of $R^2$ out of a more complex regressor.

## Results

| Method | MSE | $R^2$ |
|---|---|---|
| **Ours** | **0.274 ± 0.022** | **0.686 ± 0.012** |
| Two-Stage Likelihood (Feng et al.) | 0.367 ± 0.082 | 0.525 ± 0.101 |
| FTWR (Feng et al.) | 0.471 ± 0.149 | 0.390 ± 0.181 |
| FT (Feng et al.) | 0.522 ± 0.079 | 0.329 ± 0.048 |
| LR (Feng et al.) | 0.688 ± 0.028 | 0.084 ± 0.059 |

A 25% MSE reduction against the strongest baseline, and $R^2$ goes from 0.525 to 0.686. The variance across folds also drops, which suggests the persona features are stable predictors rather than noisy ones.

One caveat. Feng et al. use a different setup: 327 MCQs (excluding diagram items) with a 65/15/20 train/val/test split, while we use 900 MCQs with five-fold CV. IRT ground truth was estimated independently in both studies. We report this comparison faithfully but it is not a strictly controlled evaluation.

## What This Says

The framing we keep coming back to is that *item difficulty is not a property of the item alone*. It emerges from the interaction between an item and the learner population. A multi-step symbolic manipulation problem can be easy for a Rule Memorizer and hard for a Conceptual Reasoner even if those two clusters have similar overall accuracy under a one-dimensional IRT model. The persona-conditioned simulation makes this interaction visible to the predictor, and we think that is why the features work.

This fits with a broader pattern in the LLM-as-simulated-student literature. He-Yueya et al. (2024) noted that LLMs tend to produce expert-biased responses unless they are explicitly grounded. Feng et al. (2024) found the same in distractor generation: the models generate mathematically valid distractors that do not target the misconceptions real students hold. Both findings point at the same thing: LLM reasoning and student error patterns do not naturally align. Grounding simulation in empirically discovered personas is one way to close that gap.

A few downstream uses follow:

**Cold-start.** We never touch real responses for the target item itself, so difficulty can be estimated at authoring time. This is the use case that motivated the work.

**Diagnostic item design.** The per-persona response distributions are richer than a scalar difficulty estimate. An item where Conceptual Reasoners fail but Rule Memorizers succeed measures something quite different from one where both fail. That distinction is useful for instructors trying to verify that an item targets its intended concept.

**Targeted instruction.** If Rule Memorizers reliably fail a class of items, those items probably depend on conceptual understanding rather than procedural recall. Knowing which kind of student fails which kind of item is more actionable than knowing only that the item is hard.

Paper: *MCQ Difficulty Prediction via Modeling Learner Heterogeneity Using Data-Driven Cognitive Profiling*, AIED 2026.

## References

1. Wang, Z., Lamb, A., Saveliev, E., Cameron, P., Zaykov, Y., Hernández-Lobato, J.M., Turner, R.E., Baraniuk, R.G., Barton, C., Jones, S.P., Woodhead, S., Zhang, C. *Diagnostic Questions: The NeurIPS 2020 Education Challenge.* [arXiv:2007.12061](https://arxiv.org/abs/2007.12061) (2020).
2. Feng, W., Tran, P., Sireci, S., Lan, A.S. *Reasoning and Sampling-Augmented MCQ Difficulty Prediction via LLMs.* Artificial Intelligence in Education (AIED 2025), Springer LNCS, pp. 31–45.
3. Scarlatos, A., Fernandez, N., Ormerod, C., Lottridge, S., Lan, A. *SMART: Simulated Students Aligned with Item Response Theory for Question Difficulty Prediction.* Proceedings of EMNLP 2025.
4. He-Yueya, J., Ma, W.A., Gandhi, K., Domingue, B.W., Brunskill, E., Goodman, N.D. *Psychometric Alignment: Capturing Human Knowledge Distributions via Language Models.* [arXiv:2407.15645](https://arxiv.org/abs/2407.15645) (2024).
5. Feng, W., Lee, J., McNichols, H., Scarlatos, A., Smith, D., Woodhead, S., Ornelas, N., Lan, A. *Exploring Automated Distractor Generation for Math Multiple-Choice Questions via Large Language Models.* Findings of NAACL 2024, pp. 3067–3082.
