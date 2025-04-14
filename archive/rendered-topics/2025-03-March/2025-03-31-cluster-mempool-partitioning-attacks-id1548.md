# Cluster mempool partitioning attacks

stefanwouldgo | 2025-03-31 14:57:05 UTC | #1

This continues the discussion from [How to linearize your cluster](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303) on how requiring linearizations might impact mempool partitioning attacks. I have started this new topic in order not to clutter that topic too much and because it seems to me that this is interesting/problematic independently of that discussion. 

@sipa brought up the following attack as a reason against requiring linearizations/chunkings when doing RBF with non trivial mempool dependencies (I am going to assume we would only require them in this case, in order to avoid a potential resource draining attack): an attacker might rush several versions of a group of txns depending on an existing cluster to different mempools. Each version of this attacker subcluster conflicts with the others, but none can replace the others. There are multiple possible reasons for this, chief among them that these versions have corresponding fee diagrams that are either equal or incomparable. The result is that this partitions the mempools in the network with mutually exclusive versions of the cluster. This attack is possible today, though current RBF rules don’t compare mempool diagrams, and I am going to argue that that’s a good thing.

Now IIUC, the argument against requiring chunkings is that as an honest network participant I will only learn one version of the cluster and when attaching a new tx to it, I can only deliver an optimized chunking for this version. However, this chunking might not do much for other versions, because these might not overlap much. And so, my new tx will not be relayed across partition boundaries because my chunking cannot compete with the attacker chunkings. 

Now recall that I only argue for requiring linearizations when replacing existing txs using RBF, because it seems to me that this is the only scenario where we are time limited during relay. Please let me know if that’s a false assumption!

It appears to me that if the attacker constructs his txns in such a way that the different versions have incomparable diagrams, then independently of requiring linearizations, the rule that RBF requires strictly improving the diagram means that the attacker can keep me from relaying any RBF tx in this cluster, because I don’t even get to know all the versions I would have to improve upon. This is possible because for diagrams, we have only a partial order instead of the strict ordering with scalar fees. I’m referring to the RBF rules proposed here: [https://github.com/bitcoin/bitcoin/blob/20e42a4a7423d7bc62326cdb6405bc6d5900953e/doc/policy/mempool-replacements.md](https://github.com/bitcoin/bitcoin/blob/20e42a4a7423d7bc62326cdb6405bc6d5900953e/doc/policy/mempool-replacements.md)

Is this still the current proposal for RBF in a cluster mempool world? If it is, this seems like a devastating attack to me. If it is outdated I apologize for wasting everyone's time and would be grateful for a pointer to the current one.

-------------------------

instagibbs | 2025-03-31 19:47:44 UTC | #2

First, thanks for splitting this out, because indeed it was getting a like sidetracked.

[quote="stefanwouldgo, post:1, topic:1548"]
Now IIUC, the argument against requiring chunkings
[/quote]

There are other reasons too, such as bandwidth wasting. In probably 99.9% of the time the hints won't be required, so deciding when and what to send is an open question.

[quote="stefanwouldgo, post:1, topic:1548"]
Now recall that I only argue for requiring linearizations when replacing existing txs using RBF, because it seems to me that this is the only scenario where we are time limited during relay.
[/quote]

I don't believe that's correct; a better linearization could cause it to avoid eviction, or cause it to be mined faster. Perhaps the issues aren't as severe as with RBF, but they still exist.

[quote="stefanwouldgo, post:1, topic:1548"]
the attacker can keep me from relaying any RBF tx in this cluster, because I don’t even get to know all the versions I would have to improve upon
[/quote]

In general we cannot assume a "wallet" knows anything about mempool contents, and that a specific mempool is remotely consistent with another mempool. 

You don't have to "know" all the possible variants, you just have to relay a package with linearization information which ends up dominating in the diagram check.

The trick is finding a fix that isn't "send entire clusters redundantly" and is still useful for the <<0.1% of the time it's actually needed.

-------------------------

sipa | 2025-04-01 01:34:34 UTC | #3

[quote="stefanwouldgo, post:1, topic:1548"]
it seems to me that this is the only scenario where we are time limited during relay. Please let me know if that’s a false assumption!
[/quote]

There is another one: after processing a block, we need to relinearize all clusters affected by it (by having includes mempool transactions or by having included transactions that conflict with it). And in that setting, we obviously can't require linearization information for the clusters - we must accept valid blocks. It's even more complicated when there is a reorg involved, as the re-added transactions (those that were in disconnected blocks, and are being moved back to mempool) may cause us to temporarily violate cluster count/size limits. Fixing that is, due to the amount of transactions involved, even more of a best-effort thing compared to transaction relay.
[quote="stefanwouldgo, post:1, topic:1548"]
the rule that RBF requires strictly improving the diagram means that the attacker can keep me from relaying any RBF tx in this cluster, because I don’t even get to know all the versions I would have to improve upon.
[/quote]

There are many reasons why transaction relay on the network (today, but also post cluster-mempool) is non-confluent. Even just hearing about the exact same transactions, but in a different order, may result in a different set of actually accepted ones. This is due to several reasons:
* **DoS protection** not all policy rules boil down to incentive compatibility; there are other ones involving resource limits (e.g. cluster size/count), and ones to prevent free relay (generally, we expect all RBF transactions to pay some marginal fee in addition to normal requirements, which pay for the relay of the transactions they evicted).
* **Dealing with conflicts** While we have nice theoretical results about the linearization of a single set of transactions having a global and object optimum, this does not hold once replacements are introduced. Once you're considering an optimal choice/order of transaction among a set with dependencies *and* conflicts, it is not a well defined problem what the better one is even, let alone having an algorithm for picking the best one.

However, our general thinking is that this isn't really a problem, because people in practice don't reason about replacement fees. Even today, with the relatively simple (compared to post cluster mempool RBF) BIP125 rules, people just fire and forget: if a transaction doesn't look like it's confirming, retry with a higher fee, repeat. With cluster mempool the process would become a lot more opaque, but for more consistent.

My concern with requiring linearization for relay is however something in addition to problems with general non-confluence of relay. At least with the proposed cluster mempool RBF rules (and nodes individually computing near-optimal cluster linearizations for it), the transaction *will* relay and confirm once its feerate makes it incentive compatible to do. If senders are additionally required to come up with good linearizations themselves, for whatever clusters it may attach to in peers, you're requiring wallets/node to not only decide for themselves, but also for others.

> Is this still the current proposal for RBF in a cluster mempool world?

It's not, but I don't think the change invalidates anything you said here. Here is a writeup for what post-cluster mempool RBF would become (it focuses on package RBF, which is even more complicated, but just imagine the transactions are submitted one by one now): https://delvingbitcoin.org/t/post-clustermempool-package-rbf-per-chunk-processing/190

-------------------------

sipa | 2025-04-14 12:04:34 UTC | #4

Somewhat related, but if we are considering P2P extensions to relay linearization information for clusters (something I'm skeptical about as pointed out above, but ignoring that), there may be another option worth considering.

In the https://delvingbitcoin.org/t/spanning-forest-cluster-linearization/1419 algorithm, if one knows which dependencies are "active" (which is just $m$ bits, or $n \log_2(2n+1)$ bits with a more advanced encoding), it is possible to decide in $\mathcal{O}(n^2)$ time whether it represents an optimal state. In contrast, deciding whether an existing linearization is optimal is I believe equivalent to computing a single min-cut, so $\mathcal{O}(n^2 \sqrt{m})$ in practice, $\mathcal{O}(nm)$ in theory, in case the existing linearization consists of a single chunk.

A downside is that I don't know how to "combine" these states. For linearizations we have the $\mathcal{O}(n^2)$ [merging algorithm](https://delvingbitcoin.org/t/merging-incomparable-linearizations/209) that can take two incomparable suboptimal linearizations, and compute a new linearization that is better than both. In a P2P setting this means it is possible to incorporate good linearization information even if there are mismatches between peers (without guarantees of optimality, of course). I don't know anything similar for the spanning-forest state.

All of this may end up being moot if we can effectively linearize everything optimally in practice, of course...

-------------------------

stefanwouldgo | 2025-04-14 15:23:02 UTC | #5

[quote="sipa, post:3, topic:1548"]
If senders are additionally required to come up with good linearizations themselves, for whatever clusters it may attach to in peers, you’re requiring wallets/node to not only decide for themselves, but also for others.
[/quote]

I'm sorry, I don't understand what you mean here. What are wallets required to decide for others?

-------------------------

