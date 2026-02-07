# A max-plus perspective on package relay and block assembly

GaloisField2718 | 2026-02-06 22:57:26 UTC | #1

# A max-plus perspective on package relay and block assembly

## Motivation

Package relay, CPFP, ancestor limits, and block assembly are often discussed separately. In practice, they interact tightly, and their combined behavior can be hard to reason about.

Common observations include:

* sharp inclusion thresholds for packages,
* blocks that appear to leave some weight unused,
* small fee changes that suddenly make a transaction viable.

This post proposes a simple conceptual framework to reason about these effects. It does **not** suggest any protocol or policy change.

*Related work includes mempool-based fee estimation techniques, which derive practical estimates from block templates. The perspective in this post abstracts beyond specific heuristics to clarify underlying economic incentives and marginal trade-offs.*

---

## What miners are effectively optimizing

From a miner’s perspective, block assembly consists of selecting transactions:

* subject to a block weight limit and dependency constraints,
* in order to maximize total fees.

Abstractly, this is an optimization problem over a DAG with additive rewards (fees) and additive costs (weight). In practice, miners rely on greedy heuristics to approximate this optimization, but the underlying objective remains the same.

---

## Packages as incremental decisions

Block construction is incremental. Miners repeatedly decide whether adding a transaction—or a transaction together with its unconfirmed ancestors—is worthwhile.

This makes it natural to reason in terms of *packages*, evaluated by their **marginal contribution**:

* additional fees gained,
* additional weight consumed.

Dependencies can make marginal contributions non-linear at the transaction level, which is why packages—not individual transactions—are the natural unit of comparison.

---

## The implicit trade-off and the shadow price of weight

Each incremental addition trades off fees against weight. The block weight limit induces an implicit *shadow price* for weight: the **minimum fee (in sats) a miner requires to consume one additional unit of block weight at that moment**.

This price:

* is not configured explicitly,
* is not fixed,
* emerges from competition between transactions in the mempool.

Concretely, the shadow price corresponds to the effective *feerate cutoff* at block assembly time. It is **revealed**, not computed:

* by the lowest-feerate transaction or package that still gets included,
* and the highest-feerate transaction or package that is excluded.

---

## Why leftover space does not imply non-optimal behavior

An optimal block is not necessarily a full block. If no remaining transaction or package offers a positive marginal benefit at the current shadow price of weight, then including nothing further is optimal—even if some weight remains unused.

In practice today, most blocks are near the weight limit, and leftover space is often explained by mempool conditions or template construction details. The point here is not about frequency, but about interpretation.

This marginal cost can implicitly reflect factors such as validation or propagation overhead, without needing to model them explicitly. The relevant question is therefore not:

> “Is there space left in the block?”

but:

> “Is there any remaining transaction whose marginal fee exceeds the current shadow price?”

Observing leftover block space alone is not sufficient to infer non-optimal miner behavior.

---

## Threshold effects and abrupt behavior

This optimization problem is linear but constrained, which makes its solutions **piecewise linear**.

As a result:

* small fee changes can cross thresholds,
* entire packages may suddenly become viable or non-viable.

These effects are structural consequences of the optimization problem, not necessarily strategic choices.

---

## Relation to max-plus algebra and tropical geometry

The underlying computation relies on:

* maximization (choosing the best option),
* addition (accumulating fees and weight).

This corresponds to well-known frameworks such as max-plus algebra and dynamic programming on DAGs. In mathematics, this perspective is often referred to as *tropical (max-plus) geometry*. No advanced mathematics is required here; the term is used only as a descriptive lens.

---

## Conclusion

Block assembly and package relay already follow a coherent optimization logic. Making this logic explicit helps clarify observed behavior and avoid misinterpretation, without requiring any changes to Bitcoin itself.

-------------------------

gmaxwell | 2026-02-07 17:52:55 UTC | #2

[quote="GaloisField2718, post:1, topic:2235"]
It is **revealed**, not computed

[/quote]

All that is revealed here is your use of AI to waste people’s time with a low value and outdated post.

The reason blocks are not essentially completely full is usually a product of the flawed design of the GBT interface resulting in createnewblock not having any idea how big the coinbase transaction will be and just making a conservative guess.  This is a long known flaw that has been preserved in the protocol due to abuse of abusive and anti-competitive conduct by Luke-jr basically making it unattractive for anyone to touch GBT.

Of course, they may not be a transaction to fit in the space– but that’s usually not the case.  The slope of fee rate in the feerate ordered mempool around the block limit is usually pretty flat, and so there is usually no problem with packing the block very close to completely full.

-------------------------

GaloisField2718 | 2026-02-07 20:53:40 UTC | #3

Thanks for the clarification.

I agree that in practice today, most of the small amount of leftover space we
observe is explained by GBT/coinbase sizing and template construction details.
That’s a well-known limitation of the current interface, and not something the
post was meant to dispute.

The point I was trying to make is a narrower one: leftover space, *by itself*,
is not sufficient evidence of non-optimal miner behavior, because block
assembly is driven by marginal trade-offs. That statement is logical rather
than empirical, and remains true regardless of the specific implementation
details that dominate in practice today.

If that distinction wasn’t clear, that’s on me.

-------------------------

