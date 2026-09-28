---
title: "Predicting Video in Representation Space"
date: 2026-09-28T09:00:00+10:00
draft: false
math: true
summary: "A working plan: encode each video frame with a frozen image transformer, compress it into a handful of continuous visual tokens, and train a causal transformer to predict the next frame's tokens — learning how scenes evolve without ever translating them into text or pixels."
tags: ["AI", "Transformers", "Video", "Research"]
---

*Note: this is a plan and a hypothesis, not a result. Nothing below has been run yet. It is written down so that the experiment has a clear shape before it starts, and so that it is obvious afterwards whether it worked.*

---

A lot of current video understanding goes through language at some point. Frames get captioned, captions get fed to a language model, and the language model reasons about what happens next. That works, but it throws away most of what is in the frame — whatever the caption does not mention is gone.

The idea here is to skip the text. Process a video into learned visual representations, then learn to predict how those representations change over time. The model never sees a word; it only sees vectors produced by an image encoder, and it is trained to guess the next ones.

The plan has four parts.

## 1. Encode each frame visually

For each frame \(I_t\), a pretrained image transformer produces a hidden state for each image patch:

\[
I_t \rightarrow H_t \in \mathbb{R}^{N \times d}
\]

Here \(N\) is the number of patches and \(d\) is the hidden width. These vectors are the image model's learned visual representations. They can encode edges, colours, shapes, and relationships between regions, without spelling any of it out as text.

A reasonable starting point is a model such as DINOv2. It has register tokens, which are tempting to use as a ready-made frame summary, and they are one possible feature. But there is no reason to assume that four registers are a *complete* summary of a frame — they were introduced to soak up attention artefacts, not to be a bottleneck representation. So the main input should be the image model's **patch hidden states**, and the registers can be tried as an extra signal on top.

## 2. Compress each frame into a small set of visual tokens

\(N\) patch vectors per frame is too many to feed a temporal model across a long clip. A learned projector or resampler reads the patch vectors and produces \(M\) vectors for that frame:

\[
H_t \rightarrow Z_t = (z_{t,1}, \ldots, z_{t,M})
\]

These are continuous vectors, not words or token IDs. Together they form a compact representation of the frame that the temporal model can process.

\(M=4\) is a natural first setting, but it should be treated as a **capacity choice to test**, not a theoretically special number. The obvious sweep is \(M \in \{4, 8, 16\}\), measuring how much of the encoder's visual information each setting retains. If 4 tokens already captures most of what matters for prediction, that is interesting in itself; if performance keeps climbing at 16, the bottleneck is doing real damage and the number needs to go up.

## 3. Predict the next frame's visual tokens

A causal temporal transformer processes the tokens from earlier frames, in order, and predicts the next frame's tokens:

\[
(Z_1, Z_2, \ldots, Z_t) \rightarrow \widehat{Z}_{t+1}
\]

Frame-order or timestamp information is added to each set of vectors so the model knows when they came from. Training compares the prediction \(\widehat{Z}_{t+1}\) with the actual representation \(Z_{t+1}\) produced by running the encoder and resampler on the real next frame.

This is **next-token prediction in spirit**, but the predicted "tokens" are vectors. There are two ways to train it:

- **Continuous.** Regress the vectors directly with a similarity or vector-prediction loss — cosine, L2, or something contrastive.
- **Discrete.** Quantize the vectors into codes and predict code IDs with a cross-entropy loss. This is closer to ordinary language-model next-token prediction, and it lets the model express a distribution over possible futures rather than averaging them.

The continuous version is simpler to start with. The discrete version is worth trying if the continuous predictions turn out blurry — a regression loss on an uncertain future tends to predict the mean of the possibilities, which may not look like any of them.

## 4. First experiment

Freeze the image encoder initially, then train the resampler and temporal transformer on video clips. Keeping the encoder frozen means the representation space is fixed, so the task cannot be made easier by quietly collapsing it.

The first comparison is against simple baselines, the most important being **copy-last-frame**: predict that the next frame's representation is the same as the current one. In most video, consecutive frames are very similar, so this baseline is strong and a model can look good on raw loss just by learning to approximate it.

If the model beats that baseline, the next question is whether it predicts *meaningful* changes — motion, objects entering or leaving, actions continuing — rather than just carrying forward the previous frame's features with a little noise. That means looking at the error specifically on frames where something changes, not averaged over the whole clip.

## The hypothesis

Stated plainly:

> A compact set of learned visual vectors from each frame can preserve enough of the image model's representation for a second transformer to learn how scenes evolve over time.

One important distinction: this predicts the next frame's **representation**, not the next frame's pixels. Nothing here generates an image. If the goal were a plausible-looking next frame, it would need an additional decoder from \(Z_{t+1}\) back to pixels. For the question being asked — can a model learn scene dynamics in a visual latent space, without passing through language — the representation is the thing that matters, and the pixels are optional.
