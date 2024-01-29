# Replace-By-Fee-Rate vs V3

oohrah | 2024-01-28 01:33:34 UTC | #1

How do these proposals compare?

https://petertodd.org/2024/one-shot-replace-by-fee-rate
https://github.com/bitcoin/bips/pull/1541

I'm developing things on top of Lightning so I'm not a low level guy.

-------------------------

instagibbs | 2024-01-29 15:04:37 UTC | #2

At a high level, each approach is attempting to answer the question:

How do we make sure incentive compatible transactions can be mined without introducing transaction relay that isn't "paid for"? This also known as the "free relay problem".

The "free relay problem" becomes a problem if people are able to make transactions that enter into everyone's mempools, causing validation and bandwidth costs to everyone on the network. but then are ejected from the mempool later, resulting in *no fees* being paid for the privilege. 

To mitigate this problem, the current Bitcoin Core software makes attempts to minimize it:

1) Replacement transactions have to pay for what they replace("total fee"), plus "incremental fee"(today 1 sat/vbyte) to pay for the new bytes being gossiped around the network.
2) mempool minfee: When the mempool gets to full and gets "trimmed" down to ~300MB by default, the "min fee" to enter the mempool is raised to the highest paying thing that was ejected, plus the 1 sat/vbyte incremental rate. This "raises the bar" for the next things to enter, and only in the worst case allows some temporary free relay which is paid for immediately after.
3) timeout: After 2 weeks, the mempool ejects transactions as they aren't expected to be mined. This is free relay, bound by long timescales.

"V3": Relay Less
---
V3 is one attempt to get around pinning issues in an opt-in manner. If a transaction can commit to opting into a new policy, we can be more restrictive in a way that perhaps wouldn't be generally acceptable to users.

In essence, it restricts topology to avoid free relay, while mitigating pinning by restricting total size of portions of transactions. It's a mitigation, not a complete fix, but also doesn't allow new free relay avenues. [Future mempool updates](https://delvingbitcoin.org/t/an-overview-of-the-cluster-mempool-proposal/393) could likely allow this policy to evolve into something more incentive compatible and more pin resistant.

"Replace-by-Fee-Rate": Relay More
---
In contrast, some have suggested that the free relay problem isn't much of a problem at all, or that it's an unsolved problem, and that since it cannot be stopped, we should just not worry about offering services that would use it.

This is what Peter's argument essentially boils down to: He claims miners can already induce some free relay by leaving fees on the table with transaction filters, so we should just bite the bullet and allow more, and let anyone do it.

I'll let people dive into the precise arguments themselves, but it's important to note a few things:
1) If free relay is a solvable problem(through say, smarter relay logic), should we really be offering a replacement policy we may revoke later?
2) Is it really qualitatively the same if miners leaving fees on the table intentionally can do something, versus anyone on the network?
3) It's also important to note that pre-cluster mempool, reasoning about any of this is very hard to do. Peter's first iteration of the idea was [broken](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2024-January/022316.html), allowing unlimited free relay. He claims he's fixed it by hot-patching the idea with additional RBF restrictions, but like usual, reasoning about current RBF rules is very difficult, and maybe impossible. I think energy would be better focused on getting RBF incentives right, before giving up the idea of free relay protection entirely.

-------------------------

