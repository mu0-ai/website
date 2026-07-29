---
title: "Quantile Balancing: Treating MoE Routing as an Optimal Assignment Problem"
date: 2026-07-30T09:00:00+10:00
draft: false
math: true
summary: "An English summary of Su Jianlin's derivation of Quantile Balancing — posing MoE load balancing as a constrained assignment problem, and showing that biased Top-k routing falls out of its dual as a set of expert congestion prices."
tags: ["AI", "Mixture of Experts", "Optimization", "Research"]
---

*Note: this is an English summary and interpretation of Su Jianlin's Chinese-language blog post, **"MoE环游记：6、最优分配促均衡"** ("MoE Journey 6: Promoting Balance Through Optimal Assignment"). The original derivation and ideas belong to the author. I found the post interesting and wanted to make its central argument accessible to English-speaking readers.*

Load balancing is one of the central engineering problems in sparse Mixture-of-Experts (MoE) models.

An MoE router assigns each token to a small number of experts — typically the experts receiving the highest router scores. Without an explicit balancing mechanism, however, some experts can receive far more tokens than others. The popular experts become computational bottlenecks while other expert capacity is wasted.

The usual question is:

> How can we encourage balanced expert usage without damaging the model's primary training objective?

The original post proposes a different framing:

> Instead of penalising imbalance *after* routing, solve for the highest-scoring assignment that is balanced *by construction*.

This leads to **Quantile Balancing (QB)**: a loss-free routing method derived from a constrained assignment problem.

## 1. The standard approaches

Suppose token \(i\) receives a router score \(s_{ij}\) for expert \(j\). Each token must activate \(k\) out of \(n\) experts.

The simplest router selects:

\[
  \operatorname{TopK}_j(s_{ij})
\]

This locally maximises the score for every token, but it does not control the total number of tokens assigned to each expert.

### Auxiliary-loss balancing

A common solution adds an auxiliary loss that penalises uneven expert utilisation. Conceptually, this says:

> Route normally, measure the imbalance, and train the router to avoid it.

This works, but introduces two problems.

**The coefficient must be tuned.** If the auxiliary-loss weight is too small, balancing is ineffective; if it is too large, it interferes with the main language-model objective.

**The gradient is approximate.** The discrete routing operation requires an approximate gradient, commonly involving a straight-through estimator. The resulting gradient is not necessarily limited to correcting load balance and can have unintended effects on training.

### DeepSeek's loss-free balancing

DeepSeek's loss-free approach avoids placing the balancing objective inside the model loss. It introduces an expert-specific bias \(b_j\) used only during routing:

\[
  \operatorname{TopK}_j(s_{ij} + b_j)
\]

The *original* router score — not the biased score — is still used when weighting the selected experts. The bias therefore changes which experts are selected without directly entering the expert computation.

If an expert is overused, its bias is reduced; if it is underused, its bias is increased:

\[
  b_j \leftarrow b_j - \gamma \operatorname{sign}\!\left(F_j - \frac{1}{n}\right)
\]

where \(F_j\) is the expert's observed fraction of the load.

This is elegant and minimally invasive, but the update rate \(\gamma\) remains a hyperparameter. Its appropriate magnitude depends on the scale and distribution of the router scores. Layers with unusual score distributions — often early MoE layers — may consequently be difficult to balance with one global value.

## 2. Reframing routing as constrained optimisation

Assume a batch contains \(m\) tokens and there are \(n\) experts. Let

\[
  x_{ij} \in \{0, 1\}
\]

indicate whether token \(i\) is assigned to expert \(j\).

The desired assignment has three properties.

**Every token selects exactly \(k\) experts:**

\[
  \sum_j x_{ij} = k
\]

**Every expert receives exactly its equal share of the work:**

\[
  \sum_i x_{ij} = \frac{mk}{n}
\]

**Among all assignments satisfying those constraints, the total router score is maximised.** Together, this gives:

\[
  \begin{aligned}
    \max_{x_{ij} \in \{0,1\}} &\quad \sum_{i,j} x_{ij} s_{ij} \\
    \text{subject to} &\quad \sum_j x_{ij} = k, \\
                      &\quad \sum_i x_{ij} = \frac{mk}{n}
  \end{aligned}
\]

This formulation makes the trade-off explicit. The system is not merely trying to make every expert equally busy; it is finding the balanced allocation that preserves as much of the router's preference as possible.

## 3. From a global assignment to expert prices

At first glance, the formulation appears to require solving a large binary integer program. The key result of the post is that its solution can be represented much more simply.

After relaxing \(x_{ij}\) to the interval \([0,1]\), introducing Lagrange multipliers for the token and expert constraints, and applying minimax duality, the assignment rule becomes:

\[
  x_{ij}^{*} = 1 \quad\Longleftrightarrow\quad s_{ij} - \alpha_i - \beta_j > 0
\]

Here:

- \(\alpha_i\) is a **token-specific threshold**;
- \(\beta_j\) is an **expert-specific balancing price**.

Because every token must select exactly \(k\) experts, this is equivalent to:

\[
  \operatorname{TopK}_j(s_{ij} - \beta_j)
\]

The token threshold \(\alpha_i\) is needed while *solving* the optimisation problem, but it disappears from the final routing rule. Only the vector of expert biases \(\beta\) is required at inference time.

This is the most important conceptual result:

> The globally optimal balanced assignment can be implemented as ordinary per-token Top-k routing after subtracting an appropriate price from each expert.

An expert in high demand acquires a larger price and becomes harder to select. An underused expert receives a smaller — or potentially negative — price and becomes easier to select.

The analogy is congestion pricing: tokens retain their underlying expert preferences, but the effective price of scarce capacity changes.

## 4. Why quantiles appear

The remaining problem is finding the correct values of \(\alpha\) and \(\beta\).

For a fixed expert-price vector \(\beta\), the optimal token threshold \(\alpha_i\) lies between the \(k\)-th and \((k+1)\)-th largest elements of

\[
  s_{ij} - \beta_j
\]

The post chooses the \((k+1)\)-th largest value.

Similarly, for fixed token thresholds \(\alpha\), the optimal \(\beta_j\) is determined by the threshold that leaves exactly \(mk/n\) tokens assigned to expert \(j\).

Both operations are the same quantile viewed along different axes. Specifically, they correspond to the \(1 - k/n\) quantile. The resulting alternating algorithm is:

```
initialise β = 0

repeat T times:
    α = row-wise    (1 − k/n) quantile of (s − β)
    β = column-wise (1 − k/n) quantile of (s − α)

route each token using TopK(s − β)
```

This is why the method is called **Quantile Balancing**. Rather than nudging an expert bias by a manually chosen learning rate, QB *calculates* the threshold implied by the constrained assignment problem.

## 5. A practical training version

Solving the alternating problem to convergence on every training batch would be expensive, and could overfit the bias to that individual batch. The practical version instead carries \(\beta\) forward as state and performs one update per batch:

1. Route the current batch using the previous \(\beta\).
2. Calculate each token's Top-k threshold \(\alpha\).
3. Calculate a new expert bias \(\beta\) using a cross-token quantile.
4. Use the new \(\beta\) for the next batch.

This has several attractive properties:

- There is no auxiliary balancing loss.
- There is no balancing learning rate to tune.
- The update naturally adapts to the numerical range and distribution of the router scores.
- It can correct severe imbalance quickly.
- The deployed routing operation remains a simple biased Top-k.

The post reports that QB is particularly helpful for difficult or highly imbalanced layers, including the first layer of an all-MoE model. Where the simpler loss-free update already achieves good balance, QB may provide little additional benefit.

## 6. The causality and information-leakage trap

The order of operations matters.

The current batch must be routed using the **previous** value of \(\beta\). Only *after* routing should the current scores be used to update \(\beta\).

If the bias is first calculated from the current batch and then used to route that same batch, one token's routing decision can depend indirectly on the scores of other tokens in the batch. That creates a cross-sample information channel unavailable during independent inference.

Even if the leaked signal appears small, large models may learn to exploit subtle inconsistencies. The safer rule is:

> Route with old state; update state afterward.

At inference time, \(\beta\) is frozen. It is not continually recalculated from incoming requests.

## 7. The distributed-systems complication

The expensive part of QB is not the per-token Top-k calculation. An MoE model already performs that operation.

The difficult step is calculating an expert-wise quantile across *all* tokens in the global batch:

\[
  m = \text{global batch size} \times \text{sequence length}
\]

For a large training run, \(m\) may reach millions of tokens spread across data-parallel, expert-parallel and gradient-accumulation boundaries. Computing an exact global quantile can therefore require substantial communication and temporary state.

The post discusses practical approximations:

- divide the token population into manageable sub-batches, calculate a bias estimate for each, and average the resulting estimates;
- or use histogram/bin statistics to approximate the global quantile with little communication.

This reveals the main trade-off of QB: it replaces a cheap but hyperparameter-sensitive control update with a more principled statistical calculation that is harder to implement globally.

## 8. Connection to loss-free SignSGD

The post also derives a cheaper approximation that helps explain DeepSeek's loss-free method.

Given fixed \(\alpha_i\), the dual objective for expert \(j\) has gradient

\[
  \frac{mk}{n} - \sum_i \mathbf{1}\!\left(s_{ij} - \alpha_i - \beta_j > 0\right)
\]

The second term is effectively the number of tokens currently selecting that expert. The gradient is therefore

\[
  \text{target load} - \text{observed load}
\]

Applying SignSGD gives

\[
  \beta_j \leftarrow \beta_j - \gamma \operatorname{sign}\!\left(\text{target load} - \text{observed load}\right)
\]

Up to sign conventions for whether the bias is added or subtracted, this recovers the loss-free balancing rule. Under ordinary conditions where there are no ties around the Top-k boundary, the post argues that this gradient approximation is equivalent to the existing loss-free update.

This gives a useful relationship:

- **Loss-free SignSGD** performs a cheap iterative correction toward balanced routing.
- **Quantile Balancing** attempts to jump directly to the coordinate-wise optimum implied by the same constrained problem.

QB is therefore not just another unrelated balancing trick. It supplies an optimisation-based *interpretation* of loss-free bias routing, and replaces its hand-tuned step size with a quantile solution.

## 9. Relation to earlier assignment-based routing

The article traces the optimal-assignment perspective back to BASE Layers, and develops QB as a modification of a Binary Integer Programming–based balancing method.

One important distinction is the choice of constraint. QB requires every token to select *exactly* \(k\) experts and every expert to receive *exactly* its allocated capacity. The compared BIP formulation uses upper bounds rather than equalities.

Those inequalities force the associated dual variables to remain non-negative, so in the resulting update the thresholds are clipped at zero. According to the post's experiments, this clipping slows balancing and can suppress overloaded experts without adequately rescuing severely underused ones. QB uses equality constraints, removes the non-negativity restriction, and permits negative prices when needed.

## 10. A compact mental model

The whole approach can be understood as a market:

| Market concept | MoE counterpart |
|---|---|
| Bids | Router scores \(s_{ij}\) |
| Supply | Equal capacity \(mk/n\) per expert |
| Price | Expert bias \(\beta_j\) |
| Purchase decision | \(\operatorname{TopK}_j(s_{ij} - \beta_j)\) |
| Price discovery | Alternating quantile updates |

Quantile Balancing adjusts prices until demand matches capacity. The important part is that the prices do not *replace* the router's preferences. They move the selection boundary just enough to satisfy the global capacity constraints.

## Takeaways

The article's central contribution is not merely another expert-bias update. It reframes MoE balancing as a constrained allocation problem, and shows why biased Top-k routing emerges naturally from its dual.

The main conclusions:

- Perfectly balanced routing can be posed as maximising total router score subject to exact token and expert capacity constraints.
- The solution takes the simple form \(\operatorname{TopK}(s - \beta)\), where \(\beta\) acts as an expert congestion price.
- Alternating quantile calculations provide a hyperparameter-free way to estimate those prices.
- Existing loss-free SignSGD balancing can be interpreted as a cheaper gradient approximation to the same underlying optimisation.
- The principal practical challenge is calculating global token quantiles efficiently across a distributed training system.
- Training should route with the previous bias and update afterward, to preserve causality and training–inference consistency.

The result is a satisfying connection between constrained optimisation, load balancing, and the very practical biased Top-k routers used in modern sparse MoE systems.

---

*Original source: Su Jianlin, "MoE环游记：6、最优分配促均衡" ("MoE Journey 6: Promoting Balance Through Optimal Assignment"), [Scientific Spaces](https://kexue.fm/), 22 February 2026. This post is only an English technical summary of that Chinese-language article. For the complete derivation, experimental discussion, implementation notes and references to related work, please read the original.*
