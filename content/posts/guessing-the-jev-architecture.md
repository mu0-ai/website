---
title: "Guessing the Jev Architecture"
date: 2026-09-22T09:00:00+10:00
draft: false
math: true
summary: "TypeSafe has released Jev, a 'System One' model that returns typed, calibrated decisions instead of text. The architecture is not public. Here is the one I would bet on — a bidirectional encoder over the state, plus a language-conditioned cross-attention head that scores every candidate answer in parallel."
tags: ["AI", "Transformers", "Architecture", "Research"]
---

*Note: this is speculation. TypeSafe has not disclosed how Jev is built. Everything below is inferred from the behaviour and the claims they have made publicly — parallel sampling, typed outputs, calibrated probabilities, and a training method they call RLCD. I could be wrong in the details and still be right about the shape; I could also just be wrong. Read it as a reconstruction exercise, not as documentation.*

---

The public description of Jev is short. It is a **System One model**: you hand it a state and a set of questions, and it returns typed decisions rather than prose. Each question is a `Choice` (pick one of a supplied option set), a `Score` (a position on ordered levels, which may fall *between* levels), or a `Noul` (a yes/no proposition returned as a probability). Every answer comes with a distribution, not just a label. The claimed properties are that it samples all outputs in one shot rather than autoregressively, that it cannot hallucinate a label outside the supplied set, and that its probabilities are calibrated — trained by something called Reinforcement Learning for Calibrated Decisions.

Those four claims are unusually constraining. Taken together they rule out most of the obvious implementations, and they point fairly directly at one family of architectures. Below is the one I would bet on: a **bidirectional Transformer state encoder plus a parallel, language-conditioned decision head**.

## The setup

Let the input state be a token sequence:

\[
x = (x_1,\dots,x_n)
\]

This may be a plain document, or a serialised object such as:

```text
customer_message: ...
account_history: ...
policy_excerpt: ...
```

For each requested decision \(j\), the caller provides a natural-language specification:

\[
q_j = \text{“Which department should handle this request?”}
\]

For a categorical decision, it also provides \(K_j\) candidate answers, each with a label and a description:

\[
c_{j,k} = (\text{label}_{j,k}, \text{description}_{j,k}),
\qquad k \in \{1,\dots,K_j\}.
\]

For example:

\[
c_{1,1} = (\text{billing},\ \text{“invoices, payments, refunds”})
\]

\[
c_{1,2} = (\text{technical},\ \text{“bugs, outages, system errors”}).
\]

The thing to notice is that the *classes arrive at runtime, as text*. Whatever the architecture is, it cannot have a fixed output neuron per class.

## 1. Encode the state once

Unlike a causal LLM, which uses a one-directional decoder, Jev is likely to use an encoder-style Transformer:

\[
H = \mathrm{Encoder}_\theta(x)
\]

where

\[
H = (h_1,\dots,h_n), \qquad h_i \in \mathbb{R}^d.
\]

Every input token attends to both its left and its right context:

\[
\mathrm{Attention}(Q,K,V) =
\mathrm{softmax}\!\left(\frac{QK^\top}{\sqrt{d_k}}\right)V.
\]

This is well suited to decision problems. The model is not trying to predict the next word; it is building a representation of the complete state. Causal masking exists to make next-token prediction well posed, and it costs you something — the representation of token \(i\) cannot see what comes after it. If you are never generating, you should not be paying that.

For efficiency, it almost certainly computes this state encoding **once per request**, regardless of whether the request carries 1 question or 50. This is where a large part of the cost advantage comes from: the expensive object is amortised across every decision asked about it.

## 2. Convert each possible decision into a query vector

Each question-plus-option pair is encoded into a query representation:

\[
z_{j,k} =
\mathrm{OptionEncoder}_\phi(q_j, c_{j,k})
\in \mathbb{R}^d.
\]

This could be a small Transformer over text such as:

```text
Question: Which department should handle this request?
Option: billing
Criteria: invoices, payments, refunds
```

or a shared language encoder plus a learned projection.

The important feature is that \(z_{j,k}\) is constructed from **user-supplied language**. This is what lets a single set of weights handle dynamic option sets. The model is not a classifier over a fixed label space; it is a matching function between a state and a described alternative.

## 3. Let every option read the shared state

Each candidate is then scored against the same encoded state:

\[
u_{j,k} =
\mathrm{CrossAttention}(z_{j,k}, H).
\]

Expanded:

\[
\alpha_{j,k,i} =
\mathrm{softmax}_i
\left(
\frac{
(W_Q z_{j,k})^\top(W_K h_i)
}{
\sqrt{d_k}
}
\right)
\]

\[
u_{j,k} =
\sum_{i=1}^{n}
\alpha_{j,k,i}\, W_V h_i.
\]

Intuitively, the **billing** candidate attends heavily to “charged twice” and “refund”; the **technical** candidate finds weaker evidence and says so.

A learned scoring head converts that readout to a scalar:

\[
\ell_{j,k} =
w^\top \sigma(W_u u_{j,k} + W_z z_{j,k}) + b.
\]

These logits are evaluated for all \(j,k\) at once:

\[
\{\ell_{j,k}\}_{j=1,\dots,J;\ k=1,\dots,K_j}.
\]

That is the whole content of “parallel sampling.” It is not that the model reasons in parallel in some exotic sense. It is that **there is no autoregressive output sequence to serialise**. The output is a tensor of decision scores computed in a fixed number of GPU operations, and adding a question adds a row, not a decoding loop.

## 4. Typed output heads

### Choice

For a multiple-choice question:

\[
p(y_j = k \mid x,q_j,C_j) =
\frac{\exp(\ell_{j,k}/T_j)}
{\sum_{k'=1}^{K_j}\exp(\ell_{j,k'}/T_j)}.
\]

The output is

\[
\hat y_j = \arg\max_k p(y_j=k)
\]

plus the full distribution, or at minimum its top probability.

```json
{
  "choice": "billing",
  "probabilities": {
    "billing": 0.94,
    "technical": 0.03,
    "sales": 0.01,
    "other": 0.02
  }
}
```

### Noul

For a binary proposition — “Does this customer request a refund?” — a single logit through a sigmoid:

\[
p(y_j = 1 \mid x,q_j) =
\sigma(\ell_j) =
\frac{1}{1+\exp(-\ell_j)}.
\]

```json
{
  "noul": 0.94
}
```

This also explains why a Noul has no separate confidence field, while a Choice does. The probability *is* the confidence. There is nothing else to report.

### Score

For an ordered \(R\)-level rubric, predict a distribution:

\[
p(y_j=r \mid x,q_j) =
\mathrm{softmax}(\ell_{j,0},\dots,\ell_{j,R})_r
\]

and return the expectation:

\[
\hat s_j =
\mathbb{E}[y_j] =
\sum_{r=0}^{R} r\; p(y_j=r).
\]

This is what makes a score fractional — 1.84 on a 0–2 urgency scale is not a level, it is the mean of a distribution over levels. A better implementation would use ordinal thresholds rather than a plain categorical softmax:

\[
p(y_j > r) = \sigma(a_j - \tau_r),
\]

which preserves the fact that the levels are ordered. A flat softmax over \(R+1\) classes throws that structure away and has to relearn it from data.

## 5. Calibration is part of the output contract

Raw softmax probabilities are generally overconfident. If calibration is a product guarantee rather than a happy accident, the training objective has to optimise the distribution itself, not just the argmax. That is the part of RLCD I would expect to be doing real work.

For a categorical answer with one-hot target \(y\), the usual proper scoring loss is cross-entropy:

\[
\mathcal{L}_{\mathrm{CE}} =
-\sum_k y_k \log p_k.
\]

A calibration-oriented objective could add a Brier term:

\[
\mathcal{L}_{\mathrm{Brier}} =
\sum_k(p_k-y_k)^2
\]

and/or a learned post-hoc temperature:

\[
p_k =
\mathrm{softmax}(\ell_k/T).
\]

So a plausible overall objective is

\[
\mathcal{L} =
\lambda_{\mathrm{task}}\mathcal{L}_{\mathrm{task}}
+
\lambda_{\mathrm{cal}}\mathcal{L}_{\mathrm{calibration}}
+
\lambda_{\mathrm{distill}}\mathcal{L}_{\mathrm{teacher}},
\]

where \(\mathcal{L}_{\mathrm{task}}\) rewards selecting correct decisions, \(\mathcal{L}_{\mathrm{calibration}}\) punishes unjustified confidence, and \(\mathcal{L}_{\mathrm{teacher}}\) distils probability distributions from stronger models or ensemble judges.

The synthetic-data story pushes strongly toward the last term. I would expect a great deal of the training set to look like

\[
(x, q, C) \longrightarrow p_{\mathrm{teacher}}(y \mid x,q,C),
\]

rather than hard labels alone. Soft targets are the natural supervision signal for ambiguous workflow decisions, where the honest answer is that “billing” is 0.65 and “other” is 0.30 — and where a one-hot label would actively teach the model to lie.

The reinforcement-learning framing then optimises the system-level outcome of a routing policy. High-confidence cases get automated; low-confidence cases cost an escalation:

\[
r =
\begin{cases}
+1 & \text{correct automatic action} \\
-\lambda & \text{incorrect automatic action} \\
-\gamma & \text{escalated case}.
\end{cases}
\]

With \(\lambda > \gamma\), the expected reward of acting is \(p - \lambda(1-p)\) against \(-\gamma\) for escalating, so the model should only act when \(p\) clears a threshold set by the cost ratio. Reporting an honest 0.6 is then strictly better than reporting a fake 0.95. That is the mechanism by which calibration stops being a metric and starts being the thing the model is optimised for.

## 6. Why this differs from an LLM in JSON mode

A normal LLM estimates

\[
p(w_1,\dots,w_m \mid x) =
\prod_{t=1}^{m}
p(w_t \mid w_{<t},x).
\]

Even when the required output is only

```json
{"route":"billing"}
```

it performs one sequential decoding step per output token, and it can still emit an invalid string. Constrained decoding fixes the syntax but not the cost, and it does not give you a distribution — it gives you a sampled path with a token probability attached to it, which is a different and much less useful object.

The Jev-like model instead estimates

\[
p(y_1,\dots,y_J \mid x, q_1,\dots,q_J,C_1,\dots,C_J),
\]

or, in the factorised form the architecture above actually implements,

\[
\prod_{j=1}^{J}
p(y_j \mid x,q_j,C_j).
\]

This is a fixed-width tensor computation:

\[
\text{state tokens} \times \text{question/option queries}
\longrightarrow
\text{typed probability tensors}.
\]

No decoding loop. No generated syntax. No parser. No possibility of a label outside \(C_j\). “Never hallucinates” is not a behavioural claim about a well-trained model; it is a statement about the support of the output distribution. The model can be *wrong*, but it cannot return something that is not an option.

Note that the factorisation also tells you where the limits are. Decisions within a request are conditionally independent given the state. If question 4 should depend on the answer to question 3, this architecture cannot express that in one call — you would have to chain requests, and the parallelism advantage goes away.

## The mental model

The most likely architecture is not:

> “an LLM that has been told to answer only with enums.”

It is closer to:

> “a language-conditioned cross-attention ranker, where the classes themselves are written in natural language and supplied at runtime.”

That is enough to produce most of the Jev effect. The genuinely hard parts are elsewhere: the quality of the state encoder, scaling the option-query mechanism to 255 choices without the candidates blurring into each other, generating synthetic decision data that covers the long tail of real workflows, and whether the calibration survives contact with a distribution the model was not trained on. The first three are engineering. The fourth is the one I would actually want to see measured.

If I am right, a few things should be true and are checkable from the outside: latency should be close to flat in the number of questions per request and roughly linear in state length; adding options should be much cheaper than adding questions; and two questions in one request should give exactly the same answers as the same two questions asked separately. If that last one fails, there is joint structure in the decision head that this write-up does not have.
