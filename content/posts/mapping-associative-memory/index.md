---
title: "Describing the Computation of an Associative Memory"
date: 2026-11-09T09:00:00+10:00
draft: false
math: true
summary: "Replacing a small neural network's calculation with explicit polynomial formulas that are certified to stay within 0.02 of its output for every real input — what that guarantee covers, and what it leaves open."
tags: ["AI", "Transformers", "Interpretability", "Research"]
---

Can we replace a neural network's calculation with an explicit formula and still know how far its answer can be wrong? In a small constructed example, we can. Two alternative formulas reproduce the neural network's complete two-number output with a checked error of at most **0.02 for every real two-number input**, including two specified changes inside the neural network.

The formulas keep a known direct contribution exactly and approximate the rest with polynomials, sums built from powers and products of input numbers. The result covers inputs between the fitted samples and arbitrarily far beyond them. Its scope is still narrow: the neural network has three untrained feed-forward blocks, no attention layer, and carries only two numbers between blocks. Each formula stores 840 coefficient numbers that set the size of its terms, compared with 54 listed scalar parameters in the neural network before shared constants. This establishes a controlled alternative description, not a compression gain or an account of a trained language model's knowledge.

To use the formula, we need to know which parts of the neural calculation it preserves and how its error was checked. That starts with being precise about what the neural network retrieves and what a description of that retrieval should contain.

## Associative retrieval gives us a computation to describe

An associative memory retrieves information by matching a query, rather than by looking up an exact address. A neural network represents that query and the information it retrieves as vectors: ordered lists of numbers. A vector's coordinates are the individual numbers in the list.

Attention provides a direct example. A query vector is compared with key vectors, which are the patterns used for matching. Each key has an associated value vector, the information that can be returned. Softmax converts the comparison scores into nonnegative weights that add to one. Attention scales each value by its weight and adds the results. These attention weights are calculated for the current query; they are different from the model parameters stored during training.

In ordinary self-attention, each allowed token position supplies one key–value pair. The number of positions is therefore different from the number of coordinates in a vector. Holding those keys and values fixed lets us ask how the returned vector changes with the query. Changing the context changes the memory being queried. The connection to associative memory is mathematical. A modern Hopfield memory retrieves stored patterns by matching a query, and its update uses the same softmax key-matching weights ([Ramsauer and colleagues, 2021](https://arxiv.org/abs/2008.02217)). This connects the retrieval calculations, without making the entire transformer equivalent to a single memory update.

The feed-forward part of a transformer, usually called a multilayer perceptron or MLP, has a related interpretation. In a standard two-matrix MLP, the first matrix compares the input with stored vectors. An activation function transforms each comparison into a scaling number. The second matrix supplies output vectors, which are scaled by those numbers and added together. The comparison vectors act as keys and the output vectors as values ([Geva and colleagues, 2021](https://aclanthology.org/2021.emnlp-main.446/)).

The matching calculations differ in important ways. MLP scaling numbers need not add to one, and some activation functions allow negative values. GELU does; ReLU does not. A gated MLP computes two sets of numbers, multiplies matching entries, and uses those products to scale its output vectors. None of these operations requires a neuron to store one fact or a vector to represent an easily named concept.

Successive retrievals also interact. In a sequential transformer block, attention changes the current hidden state, the vector carried through the neural network, and the MLP reads that updated state. Later blocks receive the changes made by earlier ones. A useful description must therefore retain how one retrieval changes the input to the next. The experiment here isolates that problem on the MLP side.

## The target is a complete input to output relationship

The neural network's weights are fixed stored parameters. Its activations are numbers calculated while processing an input, including the coordinates of its hidden state. We hold the weights fixed and vary the input. The object we want to describe is the rule that pairs each input vector with its output vector, called a mathematical map.

Preserving the pairing matters because a plot of output points alone can hide the computation. Two rules might produce the same line segment of outputs, while one moves along it once and the other moves back and forth repeatedly as the input changes. The visible segment is identical, but the rules respond differently to the input. Its shape cannot tell us which rule the neural network implements.

An explicit formula keeps those pairings available for evaluation. For a simple illustration, let \(x\) and \(y\) be two input numbers, let \(F(x, y)\) be the output vector, and let each coefficient \(a\) be a fixed vector with the same number of coordinates as that output. A small polynomial description could take this form:

\[
F(x, y) \approx a_0 + a_1 x + a_2 y + a_3 xy + a_4 x^2 + a_5 y^2
\]

Multiplying a coefficient vector by \(x\) scales every coordinate in that vector. Adding the terms predicts the whole output. The \(xy\) term describes an interaction: changing \(x\) has a different effect depending on \(y\). This is an illustration of the representation, not the fitted formula used below. A Fourier expansion would use sine and cosine patterns instead; this experiment uses polynomial patterns. In either case, the coefficients describe regularities in a calculation. Whether those regularities make it smaller, faster or easier to understand remains a question to test.

## A small neural network makes the target explicit

We constructed a neural network with three gated MLP blocks. Its input, hidden state between blocks, and final output each contain two numbers. There is no attention layer and no final projection selecting or concealing some output components. All weights are rational numbers chosen before fitting the alternative formulas.

We also allow two controlled changes to the calculation. After the first block, an amount from \(-1/16\) to \(+1/16\) can be added to the first hidden-state coordinate. After the second block, a separate amount in the same range can be added to that coordinate. These are interventions on activations, not edits to the stored weights.

The formula must therefore handle four varying numbers: the two input coordinates and the two internal additions. An early addition can change the input to a later MLP and hence change its retrieved update. Simply adding the intervention amounts to an otherwise unchanged output would miss that interaction.

## Normalization lets the description cover arbitrarily large inputs

A formula can agree with every sampled input and still fail elsewhere. Covering the whole input plane requires more than a denser test set. Here, the neural network's normalization gives us a way to make that problem finite.

Before each MLP, root mean square or RMS normalization divides the current vector by the square root of the average of its squared coordinates, with a positive constant added inside the square root. The normalized vector has bounded length. As a result, each MLP's update remains bounded even when the original input grows arbitrarily large.

The neural network also has residual connections, which carry the current state forward and add each MLP update to it. This lets us separate the final output into a known direct contribution and a bounded remainder. The direct contribution is the original input plus the two specified additions. The remainder is the sum of the three MLP updates, which we call the nonlinear correction. We approximate that correction and then add the direct contribution back exactly. This also prevents a large carried-through input from hiding a poor approximation to the smaller correction.

There is a problem with fitting that bounded correction directly in the original coordinates: a nonconstant polynomial on an unrestricted input plane is unbounded. We therefore change coordinates first. Imagine squeezing an infinite sheet into the inside of a disk without merging any two finite points. Every finite input retains its own location inside the disk. Inputs moving farther and farther along one direction approach a corresponding point on the boundary circle.

For this normalized neural network, the correction has a well-defined limit along each direction. We can therefore describe it on the disk's interior and boundary. Large inputs are still represented; the boundary records their limiting behavior. This step depends on the calculation. An arbitrary unnormalized attention head can retain distinctions at large inputs that a direction-only boundary would lose, so the same transformation cannot simply be assumed to work there.

We fitted two representations on these bounded coordinates. Each stores 840 scalar coefficients across its two output coordinates and is linear in either internal addition when the other is held fixed. That is a restriction on the approximation, not an assumption that the neural network itself responds linearly to the additions.

The second representation adds one structural constraint. As the input becomes extremely large, normalization makes a bounded internal addition's effect on later MLP updates disappear. The direct addition still passes through the residual connection, but its extra effect on the nonlinear correction vanishes. The second formula forces its addition-dependent correction terms to vanish at the disk boundary. The first does not impose that condition.

## A separate check turns the fit into a guarantee

For each representation, we evaluated the neural correction at fixed input locations and the four extreme combinations of internal additions. We then chose coefficients by least squares, minimizing the sum of squared prediction differences at those samples. Once fitted, each formula was frozen. Verification could accept it, reject it or remain inconclusive; it could not improve the fit by changing its coefficients.

Both frozen formulas passed the same bound: **the straight-line distance between the true correction and the predicted correction is at most 0.02**, for every real two-number input and every allowed pair of additions. Because both outputs receive the same exact direct contribution, the same mathematical bound applies to the complete final output.

Figure 1 shows what is being preserved. Its original input coordinates are labelled \(q_1\) and \(q_2\); the same inputs in bounded coordinates are \(s_1\) and \(s_2\). The last panel shows the two predicted correction coordinates, \(c_1\) and \(c_2\). Matching letters identify the same input throughout. For example, B stays the same query as we move between panels, even though its plotted location changes. Keeping that correspondence is what distinguishes a map of the calculation from an unlabelled cloud of outputs.

{{< figure src="input-correction-map.png" alt="Three panels. Left: two input paths in the original plane, a horizontal line through A, B, C and a diagonal through D, B, E. Middle: the same paths squeezed inside the unit disk. Right: the predicted nonlinear correction for each path, with small shaded disks of radius 0.02 at the labelled points." caption="**Figure 1.** Original inputs, their bounded coordinates and their predicted corrections. Both internal additions are zero in this illustration. The shaded circles show the certified 0.02 error radius at the marked inputs; the guarantee also covers all other inputs and every allowed pair of additions." >}}

The guarantee comes from checking regions, rather than treating the plotted points as evidence for everything between them. The checker covered the transformed input disk with rectangles and bounded the error over each entire region and the full continuous range of both additions. The first representation required 16,613 accepted rectangles. The second required 16,537. Other rectangles were discarded only when strictly outside the disk. No part of the required input or addition range remained unresolved.

Getting those bounds tight enough required changing how the same saved polynomial was evaluated. An earlier method bounded its terms separately and lost cancellations between them. In Bernstein form, a polynomial on a rectangle is written as a weighted average of fixed coefficient vectors, with nonnegative weights that add to one. Those vectors bound the polynomial without separately summing the largest possible size of every original term. Subdividing the rectangles tightens the bounds further. This changed the verification method, not the fitted formula, neural weights or error target.

The checker used lower and upper bounds with outward rounding, so the true result stayed inside the calculated interval. A fresh process reconstructed both partitions and recomputed every accepted bound, including bounds from earlier partial attempts. All passed. The code was independently reviewed, but the replay used the same checker. This is a computer-assisted result, not a proof checked by a formal proof assistant.

The 0.02 tolerance is an absolute distance in the output coordinates. It is not 98 percent accuracy or a language-understanding score. The guarantee concerns the specified real-valued calculation. Ordinary floating-point implementations need a separate rounding analysis, especially when adding a small correction to a very large direct contribution. Nor does the result establish economy: both formulas use 840 coefficients against the neural network's 54 listed scalar parameters before shared constants.

## The error bound determines which comparisons are useful

Suppose two predictions are a distance \(d\) apart. Each can be wrong by at most 0.02, so the true distance can differ from \(d\) by at most 0.04. A difference well above that uncertainty can be meaningful; a tiny apparent separation may not be. The tolerance is useful only in relation to the effect we want to distinguish.

The two certified formulas also show why matching values does not preserve every property. At the disk boundary, the true correction no longer depends on the bounded internal additions. The second formula preserves that exactly. The first retained a small, false addition effect in the boundary directions we checked, while still satisfying the output-error bound. If that limiting behavior matters, the two formulas are not interchangeable. The extra constraint did not establish a tighter guaranteed error, fewer coefficients or general superiority.

### Predicting sensitivity requires a different target

A derivative measures how quickly an output coordinate changes when an input coordinate changes a little. Small output errors alone do not guarantee accurate derivatives. Later exact checks found points that violated two proposed bounds on derivative error. Those proposed bounds failed; the output-value certificate remained valid.

To investigate whether fitting for sensitivity would help, we fitted two new formulas with both internal additions fixed at zero. Each used 210 coefficients and the same 1,257 training inputs. One targeted output values alone. The other also received derivative targets measured in the original input coordinates, so its slopes describe changes in the original query units rather than in the squeezed disk coordinates.

We compared them on a separate fixed set of 5,024 interior points and 120 boundary points. Adding derivative targets reduced the largest observed derivative error from 0.04736 to 0.03790, while increasing the largest observed output error from 0.00575 to 0.00901. Output errors were larger especially near the disk boundary. These are maxima over the specified test points, not bounds over all inputs.

There are four derivative entries because each of two output coordinates can change with each of two input coordinates. The reported derivative error combines them by taking the square root of the sum of their squared errors, called the Frobenius norm. It is not the largest error along a single input direction.

Including derivatives in a fitting objective is an established approach, also used in [Sobolev training](https://arxiv.org/abs/1706.04859). Here it exposed a trade-off, rather than producing one formula that was best for every purpose. Neither new fit has an all-input certificate. If sensitivity is the intended use, we need to fit for sensitivity and verify that property in declared input units.

{{< figure src="derivative-tradeoff.png" alt="Two bar charts. Output values: the values-only fit has largest error 0.00575, the fit with derivative targets 0.00901. Change rates: the values-only fit has largest error 0.04736, the fit with derivative targets 0.03790." caption="**Figure 2.** Fitting derivatives improved the measured slopes at the cost of larger output errors. These are two separate, uncertified fits with zero internal additions. Neither replaces the formulas with the all-input output guarantee." >}}

### Specified interventions reveal more than an unchanged output

We can still use the certified formulas to ask how two queries respond to the same internal additions. For each query, predict the nonlinear correction under a shared pair of additions. Then ask how far apart those two corrections can become anywhere in the allowed addition range.

Only four paired comparisons are needed for the approximate formulas. Holding either addition fixed, the predicted difference moves along a straight line as the other addition varies. Its distance from zero reaches a maximum at an endpoint. Applying the same argument to both additions places a maximum at one of the four extreme combinations. The output certificates then put the true largest difference within 0.04 of that predicted maximum.

This lets us distinguish queries that look similar with no addition but respond differently when the same internal contribution changes. The resulting distance describes those selected outputs and interventions. It does not, by itself, measure similarity of meaning.

## Identical outputs can still hide different retrieval mechanisms

Even a perfect output formula would not uniquely identify the memory's internal organization. Two different attention banks, collections of keys and values, can return the same vector for every query while assigning different internal attention weights.

Consider two positions with identical value vectors. Moving some attention weight from one position to the other, while keeping their combined weight unchanged, leaves their combined returned value unchanged. A deliberately constructed bank can exploit this: an extra query direction changes where attention is routed, but its effect cancels in the output.

Taking derivatives of that unchanged output cannot expose the hidden redistribution. To distinguish the organizations, we need another observation or a specified intervention, such as changing one position's value while holding its key fixed. The cancellation example makes the limit precise: a description of returned vectors tells us how the memory responds under the observations we chose, not everything about how it stores or routes information.

## A trained computation is the next test

The constructed experiment shows that an explicit formula can cover a genuine two-dimensional input family and bounded internal activation changes without relying only on sampled agreement. It also shows what the guarantee leaves open: storage and evaluation cost, derivative accuracy, internal organization and a connection to learned meaning.

The next experiment should use fixed trained MLP weights, a declared input plane, the complete output vector under study and specified internal additions. This remains proposed work. Before fitting, the intended use should determine the success criterion. Studying small intervention effects requires an error tolerance below those effects. Compression or faster evaluation requires the full storage and computation costs to beat the neural calculation. Understanding learned knowledge requires relating the observed structure to meaningful behavior, rather than naming it from the shape of a plot.

We now have a checked way to describe the returned values of a small constructed neural computation. The next question is whether the same discipline can produce a description of a trained computation that is useful enough to justify its cost.
