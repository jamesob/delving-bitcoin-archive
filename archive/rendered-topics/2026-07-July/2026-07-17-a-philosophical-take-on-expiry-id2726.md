# A philosophical take on "expiry"

josh | 2026-07-17 02:55:22 UTC | #1


> **TLDR:** Expiry is a contradictory state of being. It belongs to no one, for no one can be in a state of expiry. Expiry is defined in terms of a state of not-being, so it belongs outside being, and therefore outside "state."

## Motivation

This post is motivated by (absolute) [input-triggered expiry](https://delvingbitcoin.org/t/input-triggered-transaction-expiry/2667/18), my solution to the free relay problem in "expiring" transactions. The basic idea of the proposal is to use `nSequence` and `nLockTime` to condition the transaction on the input having $N$ confirmations by time $t$. This has one of three effects:
* If the state of the chain already reflects this condition, then the commitment merely reinforces the state and rewards miners on condition the state remains part of the chain's history.

* If the state could reflect this condition, then the transaction remains potentially valid.

* If the state could never reflect this condition under strict forward progress, then the transaction is unconfirmable with a high degree of certainty.

Notice that I avoided the use of the word "expiry" in the description, even though it enables [a similar capability](https://delvingbitcoin.org/t/expiring-htlcs-without-free-relay/2663). I believe this suggests that the proposal is more akin to a "state-contingent input" proposal, rather than an "expiry" proposal. This warrants a dedicated post to explain why.

## State requires Being

The protocol this forum is dedicated to is built on history, state, and being. History is the collection of prior states, which remain that way. Being is the act of being in the present, what I call the state-of-now.

If we want to describe the actors in the protocol, we need to start from a place of being and stay exclusively there. The Miner is a being. The Relayer is a being. The Block Explorer is a being. And of course, the User is a being.

Other beings exist in the protocol, but describing them all is beyond the scope of this post. The point is that we must describe the protocol in terms of being, and states of being.

> **Key idea:** We must describe the protocol in terms of being, and states of being.

## Expiry is a State of Not-Being

The problem with typical "expiry" proposals is that they require the transaction to explicitly claim a state of not-being. Presently, every state the transaction claims is a state of being:
* `nLockTime` *is* an absolute time in the past.
*  `nSequence` *is* a time in the past relative to the first confirmation of a coin.
* An outpoint *is* the location of a now-spent coin.
* An output *is* the location and spending conditions of an unspent coin.

With a traditional expiry proposal, like Peter Todd's [`OP_EXPIRE`](https://gnusha.org/pi/bitcoindev/ZTMWrJ6DjxtslJBn@petertodd.org/?__goaway_challenge=meta-refresh&__goaway_id=29e9646057638ed11b42056b3f8a01b1&__goaway_referer=https%3A%2F%2Fdelvingbitcoin.org%2F), we introduce a claim to a state we cannot physically see:
* The transaction expires *after* the expiry time.

Comparably, with expiry, we define ourselves, or in this case our claim, in terms of something we are not:
* We are *not* in this expired state.

Reframing our prior statements in terms of "We" for sake of comparison:
* We *see* an absolute time of at least `nLockTime`.
* We *see* elapsed time more than `nSequence` since the coin's first confirmation.
* We *see* a now-spent coin at this outpoint location.
* We *see* an unspent coin at this output location, under these spending conditions.

As you can see, describing a transaction as something we cannot see is a fundamental violation of the rule of being. That is the problem with traditional "expiry."

By comparison, with (absolute) input-triggered expiry, or what I will now call a state-contingent input, we have a clean claim to what we see:

* We *see* a now-spent coin at this outpoint location with these confirmations by time $t$.

> **Key idea:** 
> * Traditional "expiry" describes a transaction in terms of something we *cannot* see. 
> * A state-contingent input describes a transaction in terms of what we *could* see.

## Why should I care?

The traditional rationale for avoiding "expiry" is the "free relay" problem, defined as the possibility that nodes relay transaction that subsequently self-invalidate, never to be confirmed. If an attacker has a reasonable expectation of free relay, the attacker could spam the network, at little expected cost.

**This post aims to provide a philosophical basis for the same conclusion.**

When we think about the types of capabilities that are "safe" to enable, we must always think in terms of the state we can see:
* Is this a state that the user can see?
* Is this a state that the relayer can see?
* Is this a state that the miner can see?

If the answer is universally yes, then the state *may* be a part of history.

History, we say, has eyes. When we speak of History, we speak in reverse chronology, seeing what is written in the final version. So when we write history, we write what we think is part of history, in that final state.

Our goal is to write history that we know we can all see and that we also know will remain part of History. **That is how we build consensus.**

Therefore, it is *always* more desirable to reward miners for writing on top of recent history we know, rather than on top of history we lack (all else being equal).

In this context, there are three philosophical problems with "expiry":
* We commit to state we do not see.
* We relay state we do not see.
* We reward miners for history we do not see.

The problems of "expiry" flow from there.

-------------------------

