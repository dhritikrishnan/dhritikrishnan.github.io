---
title: 'Modeling Learner Heterogeneity for Zero-Shot MCQ Difficulty Prediction'
date: 2026-03-30
permalink: /posts/2026/03/mcq-difficulty-cognitive-profiling/
tags:
  - assessment
  - item response theory
  - large language models
  - psychometrics
---

Predicting the difficulty of an assessment item is a foundational problem in psychometrics. If we want to calibrate a test, build an adaptive learning system, or evaluate curriculum efficacy, we need to know how hard the questions actually are. 

Historically, this is solved via Item Response Theory (IRT), which works really well, provided you have thousands of empirical student responses to fit the model. But what happens when you write a brand new item? You hit a severe cold-start problem. You can't calibrate the item without deploying it, and you shouldn't deploy it without calibrating it. Kind of a chicken and egg problem.

Recently, the field has attempted to circumvent this by using Large Language Models (LLMs) to simulate student responses. Current SOTA methods sample a latent student ability score from a normal distribution, feed it to an LLM, and ask it to answer the question. 

There is a pretty big issue with this approach, which we can call the **Homogeneity Gap**. By modeling ability as a single scalar value, we assume that a "weak" student is just a noisier version of a "strong" student. But in reality, learner errors are highly structured. Two students with the exact same overall accuracy might fail for entirely different cognitive reasons. One might lack spatial reasoning, while the other fundamentally misunderstands fractions.

In our recent AIED 2026 paper, we explored what happens when we discard the scalar ability assumption. Instead of treating errors as random noise along a unidimensional curve, we explicitly model learner heterogeneity using data-driven cognitive profiling. 

Let's dive into how replacing a generic ability parameter with discrete, LLM-simulated behavioral personas changes the math of difficulty prediction.

### The 2PL-IRT Baseline

To ground this, let's look at the standard 2-Parameter Logistic (2PL) IRT model. The probability that a student $j$ answers an item $i$ correctly is a function of the student's latent ability $\theta_j$ and the item's latent characteristics:

$$P(X_{ij} = 1 \mid \theta_j, \beta_i, \alpha_i) = \frac{1}{1 + \exp(-\alpha_i (\theta_j - \beta_i))}$$

Here, $\beta_i$ is the item difficulty (the point on the ability scale where a student has a 50% chance of guessing correctly), and $\alpha_i$ is the discrimination parameter. 

Our goal is to predict $\beta_i$ for a *new* item, without ever observing the empirical response matrix $X_{ij}$. 

### Discovering Latent Cognitive Profiles

We have already established that continuous ability parameter $\theta_j$ fails to capture the nuances of *why* students fail. In order to derive the implicit behaviours, we partition the student population into discrete behavioral clusters. 

We applied Latent Class Analysis (LCA) over the correctness vectors of student interactions in the EEDI dataset. Optimizing for the Bayesian Information Criterion (BIC), we found that the population naturally factored into $K=5$ distinct latent classes. 

To figure out what these mathematical clusters actually represented cognitively, we computed a deviation score for each item and cluster. Intuitively, this measures how much a specific cluster over-performs or under-performs relative to the global average:

$$\delta_i^{(c)} = a_i^{(c)} - \frac{1}{K} \sum_{k=1}^K a_i^{(k)}$$

By isolating the items with the highest and lowest $\delta_i^{(c)}$ values for each cluster, we extracted the core "strengths" and "weaknesses" of that sub-population. We then prompted Claude Opus 4.5 to reason the behaviour. 

The results were surprisingly interpretable:

| Persona ($c$) | Core Cognitive Strength | Primary Vulnerability |
| :--- | :--- | :--- |
| **The Rule Memorizer** | Applies formulas accurately | Lacks number sense; magnitude misconceptions |
| **The Procedural Calculator** | Strong single-step arithmetic | Fails inverse and multi-step reasoning |
| **The Abstract Reasoner** | Logical and proportional reasoning | Weak basic arithmetic automaticity |
| **The Conceptual Reasoner** | Strong mathematical intuition ("why") | Breakdown in procedural fluency ("how") |
| **The Fraction Calculator** | Solves standard fraction equations | Fails proportional reasoning in real contexts |

Notice the non-linear dynamics here. "The Conceptual Reasoner" understands the abstract logic of *why* dividing by a fraction increases a number, but completely fails at the procedural execution of $2/9 \div 3/4$. A scalar $\theta_j$ model simply averages these traits out, losing the exact signal that makes the question difficult in the first place.

### Profile-Conditioned Simulation

Now we have $K=5$ distinct cognitive profiles. The next step is to simulate how each persona would approach a new item $i$.

We feed the item (as an image) and the persona description $\pi_c$ into a multi-modal LLM. Here, we specifically instruct the LLM to *not* solve the problem but rather adopt the persona $\pi_c$ and output a probability distribution over the answer options $\{A, B, C, D\}$ reflecting that specific student's likely behavior.

This gives us a profile-conditioned distribution vector for each persona:

$$p_i^{(c)} = [p_{iA}^{(c)}, p_{iB}^{(c)}, p_{iC}^{(c)}, p_{iD}^{(c)}], \quad \sum p = 1$$

When aggregated across all $K$ personas, we get a rich $K \times 4$ probability matrix for every single item. Instead of a binary "correct/incorrect" simulation, we now have a topographic map of how different misconceptions gravitate toward different distractors.

### Predicting Difficulty

To map this $K \times 4$ matrix back to our target continuous variable $\beta_i$, we extract the probability mass assigned to the correct option, $p_{i, \text{correct}}^{(c)}$. We compute the mean, variance, and range across our 5 personas, append a one-hot encoding of the math topic, and pass these features into a Ridge Regressor:

$$\hat{\beta} = \arg\min_{\beta} \left( \| Y - X\beta \|_2^2 + \alpha \| \beta \|_2^2 \right)$$

We deliberately used a simple linear model here. If the predictions are accurate, it proves that the signal lives in the persona-conditioned features.

### Empirical Results

We benchmarked this pipeline against a suite of baselines established by Feng et al. (AIED 2025), which utilized standard normal ability sampling for their LLM simulations.

| Method | MSE $\downarrow$ | $R^2 \uparrow$ |
| :--- | :--- | :--- |
| **Ours (Persona-Conditioned)** | **0.274 ± 0.022** | **0.686 ± 0.012** |
| Two-Stage Likelihood (Feng et al.) | 0.367 ± 0.082 | 0.525 ± 0.101 |
| FTWR (Feng et al.) | 0.471 ± 0.149 | 0.390 ± 0.181 |
| FT (Feng et al.) | 0.522 ± 0.079 | 0.329 ± 0.048 |

By explicitly modeling learner heterogeneity, we observed a ~25% reduction in Mean Squared Error (MSE) compared to the strongest baseline, and pushed the explained variance ($R^2$) from 0.525 up to 0.686. The low variance across our cross-validation folds further implies that these persona representations are highly stable.


### Why This Matters: The Relational Nature of Difficulty

Our experiments support the core hypothesis: **Item difficulty is not an intrinsic property of text.** Difficulty is relational. It emerges with the interaction between question's requirements and a learner's specific cognitive profiles. A multi-step algebra problem might be easy for a *Rule Memorizer* but difficult for an *Abstract Reasoner*, even if IRT grades them at the exact same overall ability level. 

By forcing LLMs to inhabit these distinct cognitive topologies, we narrow the gap between how AI reasons and how humans actually fail. Looking forward, this doesn't just solve the zero-shot cold-start problem. It opens up a new avenue for **diagnostic item design**, allowing test creators to simulate exactly *which* sub-populations will fall for *which* distractors.

---
*Reference: Krishnan, D., & Savelka, J. (2026). MCQ Difficulty Prediction via Modeling Learner Heterogeneity Using Data-Driven Cognitive Profiling. In Artificial Intelligence in Education (AIED 2026).*