---
title: "The Transformer Memory Architecture"
date: 2026-07-13
draft: false
summary: "Viewing transformers as long-term memory, working memory, and associative retrieval — and how Anthropic's Workspace and J-Lens results fit into a memory-centric picture of the residual stream."
tags: ["AI", "Transformers", "Interpretability", "Research"]
---

Anthropic's recent work on the **Workspace** and **J-Lens** proposes that large language models contain a shared semantic workspace — a subset of the residual stream where concepts are represented in a form that can be read and written by many different parts of the network.

This is an intriguing result, but I believe it also suggests a broader architectural interpretation of transformers. Rather than viewing transformers as alternating **attention** and **MLP** layers, we can instead view them as a cognitive architecture built around three distinct components:

- **Long-term memory**
- **Working memory**
- **Associative retrieval**

Viewed through this lens, the transformer begins to resemble a surprisingly familiar information-processing system.

## Two different memory systems

One of the interesting observations about transformers is that both major computational blocks appear to behave like associative memories, but over completely different data sources. (I explored the mechanics of this in [a previous post on transformer blocks as Hopfield-style memory systems]({{< relref "transformer-hopfield-memory.md" >}}).)

### Attention: contextual memory

Attention retrieves information from the current context. Every previous token in the sequence acts as a temporary memory. The current token forms a query, and attention retrieves whichever contextual memories are most relevant.

These memories are ephemeral. They exist only for the lifetime of the current conversation. In cognitive terms, this resembles **episodic memory** — information that is available because it was recently experienced.

### MLPs: learned memory

The MLP serves a very different purpose. Its memories are not stored in the context, but in its parameters. Training gradually writes associations into these weights. During inference, the current hidden state again acts as a query, retrieving whichever learned patterns best match the current situation.

Unlike attention, these memories persist across every conversation. They are the model's accumulated knowledge. This closely resembles **semantic long-term memory**.

## Everything meets in the residual stream

Neither attention nor the MLP directly produces the model's thoughts. Instead, each contributes information to the residual stream.

The residual stream is therefore not simply an activation vector. It is the continually evolving computational state of the model. At any point it contains a mixture of many different kinds of information:

- retrieved contextual memories,
- retrieved learned memories,
- grammatical structure,
- positional information,
- routing signals,
- intermediate computations,
- partially formed plans,
- semantic concepts.

Most of these are internal computational artifacts rather than ideas that could be expressed directly in language.

## The workspace hypothesis

Anthropic's Workspace paper argues that embedded within this much larger residual state is a special representational space. Rather than every layer inventing its own representation of concepts like *Australia*, *gravity*, or *engine*, the model appears to maintain a shared semantic format that many different circuits understand. They call this the **workspace**.

Concepts written into this workspace become globally accessible. Different attention heads can read them. Different MLPs can manipulate them. Later layers can reason over them.

In other words, the workspace functions as a communication protocol shared across the network.

## The J-Lens does not find memories

The J-Lens is often described as discovering the model's concepts. I think a more precise interpretation is that it discovers the coordinate system of the workspace.

It estimates which directions in an intermediate residual stream survive the remaining computation and ultimately influence the model's outputs. These are not necessarily all of the computations occurring inside the transformer. Rather, they are the computations that have been promoted into a shared semantic representation.

The J-Lens therefore reveals the part of the residual stream that has become globally available for reasoning.

## A different way to think about transformers

Putting these ideas together suggests a different description of transformer computation.

Attention retrieves relevant information from contextual memory. MLPs retrieve relevant information from learned memory. Both write their retrieved information into a shared workspace contained within the residual stream. Subsequent layers then operate over this updated workspace before performing another round of retrieval.

Instead of alternating between "attention layers" and "feed-forward layers," the model repeatedly performs a simple cognitive cycle:

- retrieve information,
- update working memory,
- retrieve again,
- continue reasoning.

This perspective shifts the emphasis away from the individual components and toward the flow of information between them.

## The residual stream as working memory

Perhaps the most interesting implication is that the residual stream begins to resemble **working memory**. Not because it stores information permanently — it clearly does not — but because it is the location where active information is assembled, manipulated, and shared between otherwise independent computational modules.

Attention contributes information. MLPs contribute information. Later attention layers consume that information. Later MLPs consume that information. The residual stream becomes a shared blackboard upon which the model performs its reasoning.

Anthropic's workspace may simply be the semantic subset of this larger computational state — the part that has been translated into a representation understood across the network.

## Toward a memory-centric architecture

This perspective suggests a remarkably clean decomposition of transformer computation:

- **Attention** functions as associative retrieval over contextual memory.
- **MLPs** function as associative retrieval over learned long-term memory.
- **The residual stream** functions as working memory.
- **The workspace** provides the common semantic representation that allows independently retrieved information to interact.

Reasoning then emerges naturally through repeated cycles of retrieval and workspace updates.

Viewed this way, transformers are not fundamentally alternating stacks of attention and feed-forward layers. They are memory systems. Each block retrieves knowledge from multiple sources, integrates it into a shared working memory, and hands that updated state to the next round of retrieval.

The remarkable capabilities of modern language models may arise not from any single component, but from the continual interaction between long-term memory, contextual memory, and a shared semantic workspace.

As interpretability research continues to mature, this memory-centric view may provide a useful conceptual framework for understanding not only why transformers work, but also how entirely new architectures might be designed around the same underlying principles.
