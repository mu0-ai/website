---
title: "Transformer Blocks as Two Hopfield-Style Memory Systems"
date: 2026-07-03
draft: false
math: true
summary: "How attention mixes tokens via a modern Hopfield retrieval over context, while the MLP mixes channels via a Hopfield-like lookup into learned, persistent memories."
tags: ["AI", "Transformers", "Research"]
---

The usual slogan is: **attention** mixes **tokens**, and the **MLP** mixes **channels**.

The Hopfield interpretation sharpens this into:

> **Attention performs associative retrieval over token memories.**
> **The MLP performs associative retrieval over learned parameter memories.**

## 1. Hidden states as a token-by-channel matrix

A transformer layer carries a matrix:

\[
  H =
  \begin{bmatrix}
  h_1 \\
  h_2 \\
  \vdots \\
  h_n
  \end{bmatrix}
  \in \mathbb{R}^{n \times d}
\]

Each row \(h_i\) is one token state. Each column is a channel or feature dimension.

```
            channels / features
          --------------------->

Token 1      h11 h12 h13 ...
Token 2      h21 h22 h23 ...
Token 3      h31 h32 h33 ...
...
^
|
tokens
```

## 2. Attention as a modern Hopfield retrieval over tokens

For token \(i\), attention forms a query:

\[
  q_i = W_Q h_i
\]

Every token \(j\) produces a key and value:

\[
  k_j = W_K h_j,
  \qquad
  v_j = W_V h_j
\]

Token \(i\) compares itself against all token memories:

\[
  s_{ij} = q_i^\top k_j
\]

A modern Hopfield update uses a softmax over similarities:

\[
  a_{ij} = \frac{\exp(\beta q_i^\top k_j)}{\sum_{\ell=1}^{n}\exp(\beta q_i^\top k_\ell)}
\]

Then it retrieves the weighted value sum:

\[
  z_i = \sum_{j=1}^{n} a_{ij} v_j
\]

**Interpretation.** Attention is a Hopfield-style associative lookup where the memory bank is the current sequence. The memories are not permanent; they are generated dynamically from the prompt.

Step by step, the flow is: start from the current sequence, turn every token into a key and a value, then — for each token's query — compare it against every token's key. A softmax turns those comparisons into a competition, and the token reads back a weighted sum of the values. The result is an updated representation for that token, assembled from the tokens it found most relevant.

## 3. Why this is token mixing

The update for token \(i\) is made from values \(v_j\) belonging to other tokens:

\[
  z_i = a_{i1}v_1 + a_{i2}v_2 + \cdots + a_{in}v_n
\]

So token \(i\)'s new state is a mixture of other token states. That is the mathematical reason attention is called **token mixing**.

## 4. The MLP as a Hopfield-like lookup over learned memories

After attention, each token has an enriched hidden state \(x\). The MLP processes each token separately:

\[
  y = V \, \sigma(Kx)
\]

In the key-value memory interpretation:

- \(K\) contains learned **keys**.
- \(V\) contains learned **values**.
- The activation vector says which learned memories match the current token state.

### Compare against stored memory keys

\[
  m_i = \operatorname{ReLU}(k_i^\top x)
\]

**Each key asks:** does this token representation match the pattern I store?

### Retrieve stored memory values

\[
  y = \sum_{i=1}^{d_m} m_i v_i
\]

Each active memory contributes its value vector back into the hidden state.

The flow mirrors attention, but the memory bank is fixed. Take a single token's state, compare it against the learned keys, and let the ReLU gates decide which memories fire. Each memory that fires contributes its learned value, those values are added together, and the sum is written back into the token's channels.

## 5. How this relates to a true Hopfield update

The MLP is Hopfield-like, but not exactly a standard modern Hopfield network because it uses independent ReLU gates rather than normalized softmax competition.

| Component | Attention / Modern Hopfield | MLP / Feed-forward memory |
|---|---|---|
| Memory source | Current sequence tokens | Learned weights from training |
| Keys | \(k_j = W_K h_j\) | Rows of \(K\), written \(k_i\) |
| Values | \(v_j = W_V h_j\) | Rows/columns of \(V\), written \(v_i\) |
| Similarity | \(q_i^\top k_j\) | \(x^\top k_i\) |
| Activation rule | \(\operatorname{softmax}\) | \(\operatorname{ReLU}\) |
| Retrieval | \(\sum_j a_{ij}v_j\) | \(\sum_i m_i v_i\) |
| What it mixes | Tokens | Channels inside one token |

## 6. What if the MLP used softmax?

If the MLP replaced ReLU with softmax, it would look even more directly like a modern Hopfield update:

\[
  a_i = \frac{\exp(\beta k_i^\top x)}{\sum_j \exp(\beta k_j^\top x)}
\]

\[
  y = \sum_i a_i v_i
\]

That would make the learned memories compete. The normal MLP is less competitive: many memories can fire independently and add together.

**Softmax / Hopfield** — competitive retrieval. Memories fight for probability mass, and often one or a few memories dominate.

**ReLU / MLP** — additive retrieval. Any matching memory can fire, and many memories can contribute simultaneously.

## 7. The full transformer block as two memory systems

Reading a block top to bottom: the token hidden states enter attention, which does a Hopfield retrieval over the *temporary* token memories and hands back context-enriched states. Those flow into the MLP, which does a Hopfield-like retrieval over the *persistent* learned memories and hands back memory-enriched states. A residual update folds the result back in, and the whole thing repeats in the next layer.

So the transformer alternates between two kinds of memory:

- **Temporary memory:** the prompt/context, accessed through attention.
- **Persistent memory:** the model weights, accessed through the MLP.

## 8. Why the order matters

Attention first gathers evidence from the sequence. Then the MLP interprets the enriched token state and matches it against stored learned patterns.

For example, before attention the token *capital* might not know which country is being discussed. Attention can retrieve context from *France*. Then the MLP can match the enriched state against a learned memory corresponding to:

\[
  \text{capital of France} \longrightarrow \text{Paris}
\]

Put simply, attention asks *"what context from this prompt matters?"* and the MLP then asks *"given that contextualized state, which learned memories match?"* Layer after layer, temporary context and persistent knowledge take turns refining each other.

## Takeaway

The phrase **attention mixes tokens, MLP mixes channels** describes the axes of computation. The Hopfield view describes the *function* of that computation:

- **Attention:** associative retrieval from the current context.
- **MLP:** associative retrieval from learned memories stored in the weights.

Together, a transformer layer is a repeated dialogue between working memory and long-term memory.

*Note: the MLP is described as Hopfield-like because the structure is key/value retrieval, but its ReLU gating is not the same as the softmax-normalized modern Hopfield update.*
