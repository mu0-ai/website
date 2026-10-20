---
title: "What If We Could Ask a Model for a Concept?"
date: 2026-10-20T09:00:00+10:00
draft: false
summary: "A proposal: extract one vector per concept from vision, language, and audio models, align the resulting spaces using shared concept cards, and test whether the alignment generalises to concepts it never saw — before asking whether models should be moved toward a shared space at all."
tags: ["AI", "Representations", "Research"]
---

Different models may not share an embedding space, but they may still represent many of the same things.

A vision model has some internal representation of a dog. A language model has some internal representation of a dog. A speech model may represent the sound of someone saying "dog." Those representations are not stored on matching axes. They may not even have the same dimensions. But perhaps they preserve some of the same relationships: dogs are more like wolves than tables; a puppy is a kind of dog; a dog chasing a ball connects an animal, an action, and an object.

That raises an intriguing possibility. Could we ask each model for a single vector representing a concept, collect enough of these vectors, and use them to build a common space?

## The extraction problem

Models already turn inputs into vectors, but they do not all offer the same kind of vector.

An image model may produce a vector for an entire image, alongside vectors for patches or regions. A language model produces a sequence of hidden states, one for each token. An audio model may produce a sequence of representations over time. To compare them, we need a rule that turns each model's response to a concept into one vector.

There are several possible rules.

**Use the model's existing global representation.** Vision encoders often produce a pooled image vector or a special classification-token vector. This is an appealing starting point because the model was designed to summarize an image. The equivalent is less obvious for a general-purpose language model.

**Read the hidden state at a designated position.** We could give the model a prompt with a marker, such as "The concept I want you to represent is: dog. [CONCEPT]", and extract the hidden state at the marker. This gives one vector without averaging every token. But the result may depend heavily on the prompt, the marker, and the layer we choose.

**Ask the model to generate a description, then pool its internal states.** For "dog," the model might produce "a domesticated animal often kept as a companion." We could take the hidden states associated with that generated description and combine them. This gives the model room to unpack the concept into useful detail, but generation introduces another choice: how do we combine the tokens, and how much does the answer vary from one generation to another?

**Learn a small pooling function.** Instead of taking a plain average or choosing one token, we could train a small module to read the model's token states and produce one concept vector. This may give better results, but it creates a harder question: what should that pooling module be trained to preserve?

There may not be one best answer. A simple average is a reasonable baseline, not a law of nature. The important requirement is that the extraction method be defined consistently enough to test.

## A concept should be more than a label

The word "dog" is an underspecified probe. A model's representation could change depending on whether the prompt evokes a pet, a biological category, a picture, a sound, or an action.

A better protocol might use a small *concept card*: a short definition, a few examples, and a few relationships. For example:

- **Concept:** dog
- **Definition:** a domesticated canine animal
- **Examples:** a puppy, a working dog, a family pet
- **Contrasts:** wolf, cat, table
- **Relations:** dogs are animals; dogs can be pets; dogs can chase balls

We could ask each model to represent the concept card as a whole, or extract separate vectors for the concept, examples, and relations. Keeping these probes consistent across models would let us measure whether their representations agree because they share structure, rather than because one prompt happened to suit one model better.

The same idea could extend beyond nouns. We could probe actions, attributes, comparisons, and events: *running*, *being larger than*, *a dog chasing a ball*. That matters because concepts are not just isolated points. Much of what a model knows may be expressed through relationships between things.

## From many vectors to a shared space

Suppose we collect vectors for thousands of the same concept cards from several models. We still cannot average the raw vectors. The dimensions are different, and the axes may have unrelated meanings. "Dimension 42" in one model is not automatically comparable to "dimension 42" in another.

Instead, we would first need to align the spaces. One option is to use a set of shared concept cards as anchors: find a mapping that places corresponding concepts near one another while preserving each model's internal relationships as much as possible. Another is to compare the geometry of the spaces—their neighborhoods and distances—and align them even when exact correspondences are unavailable.

Once aligned, we could ask whether the models converge on a common structure. Do "dog," "wolf," and "cat" form a similar neighborhood? Do the directions associated with *animal*, *pet*, or *chasing* agree? Does a shared representation make it easier to transfer information between image, text, and audio models?

Only after that would averaging make sense. We could average aligned vectors for the same concept to create a consensus representation. Or we could define a shared target space and train models to map their internal representations into it.

## Alignment is not automatically improvement

There is a large leap between "these models can be placed in a common coordinate system" and "moving them toward that space makes them better."

A shared space could help models exchange information. It might make a text model's concept of "dog" easier to connect to an image model's representations of dogs. It might also provide a useful interface between models that were trained separately.

But forcing every model toward the same representation could erase useful differences. A medical model may distinguish concepts that a general image model lumps together. An audio model may encode timing and rhythm that have no equivalent in a static image. If we optimize only for agreement, we could make the models more similar while making each one less useful.

So the goal should probably not be to make every model identical. It should be to identify which structure they share, preserve what is specific to each model, and create a reliable path between them.

## A testable first experiment

A practical first study could compare several extraction methods on a fixed collection of concept cards:

1. Use a vision encoder, a language model, and an audio model.
2. Extract vectors using each model's native global representation, a designated hidden-state position, and a generated-description method.
3. Repeat prompts with paraphrases and multiple generations to measure how stable each vector is.
4. Align the resulting spaces using shared concept cards.
5. Test whether the alignment transfers to held-out concepts, relations, and examples.

The critical result would not be that the training concepts can be lined up. It would be that the alignment predicts useful relationships for concepts the alignment never saw.

If it does, we may have a way to build a shared representational layer across models: not a perfect universal map of every concept, but a common structure that lets independently trained models meet, compare what they know, and perhaps learn from one another.
