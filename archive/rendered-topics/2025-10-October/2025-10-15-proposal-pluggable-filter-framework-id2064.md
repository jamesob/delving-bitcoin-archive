# Proposal: Pluggable Filter Framework

garland | 2025-10-15 03:49:15 UTC | #1

First off, this is my first post. Hello to anyone who reads this. I’m a software engineer, and I’ve been a Bitcoiner for a long time, but I’m just now starting to study so that I can being contributing code and building in this space. Please critique.

**Problem:** Shipping hard-coded filters with on/off toggles centralizes policy. It opens the door for overreach & pressure to capitulate to the demands of nation states. It’s also a cat and mouse game.

**Solution:** Instead of baking in specific filters, ship a pluggable filter framework so that node operators can author and run their own policies. Node operators could then share effective filters organically.

**Constraints:**

* **Local policy modules only.** Operators can add custom filters. Possibly as strings that are added to the config file or as flags.

* **Minimal, deterministic API.** Pure function over tx/mempool context.

* **Safe execution.** Sandbox, strict time & memory limits so a bad filter can’t wedge a node.

* **Observability.** Local logs & metrics to tune false positives & false negatives.

* **Defaults:** Ship the framework **disabled**. No active content filters by default.

**Benefits:**

* Pushes policy to node operators, improving decentralization.

* Upstream is no longer a single point of failure.

* Lets communities iterate faster without waiting on node releases.

**Trade-offs:**

* Still a cat-and-mouse game. This does not solve abuse - it decentralizes the response.

* Operational overhead lands on node operators.

* Risk of policy fragmentation.

-------------------------

cguida | 2025-10-15 04:48:45 UTC | #2

Prior art:

https://groups.google.com/g/bitcoindev/c/o3JZhiOa2PQ/m/Pvg6ZQvSAQAJ

https://github.com/bitcoinknots/bitcoin/pull/119

-------------------------

garland | 2025-10-15 04:58:34 UTC | #3

Thanks! I’ll review these in the morning. I just read Greg’s wall of text and I’m now exhausted.

-------------------------

AdamISZ | 2025-10-15 11:28:14 UTC | #4

[quote="garland, post:3, topic:2064"]
Thanks! I’ll review these in the morning. I just read Greg’s wall of text and I’m now exhausted.

[/quote]

His first response there is all you need to read - the purpose of the mempool *is* to model what’s going to be mined. If you understand that, it has great relevance to your proposal; for example, when you say

[quote="garland, post:1, topic:2064"]
Pushes policy to node operators, improving decentralization.

[/quote]

… surprisingly it’s the opposite that’s true: decentralization is improved by *not* pushing policy to node operators; if nodes are in sync on the candidates for mining, propagation and block reconstruction is faster (and fee estimation is better, side effect), which makes Bitcoin’s decentralization better. Mining would be more centralized if everyone was blocks-only, and that’s still true if everyone’s using different policies for what they see as candidates for mining.

(Not that I’m saying you shouldn’t have control over your node’s policy, of course, you do - I’m saying that it isn’t helping the network; it might help you in some way or other, e.g. if your node has specific constraints on resources perhaps)

-------------------------

1440000bytes | 2025-10-15 14:38:22 UTC | #5

[quote="AdamISZ, post:4, topic:2064"]
His first response there is all you need to read - the purpose of the mempool *is* to model what’s going to be mined.
[/quote]

I don't think this is true for several policies used in bitcoin core. Example: `dustrelayfee`

```C++
/** Min feerate for defining dust.
 * Changing the dust limit changes which transactions are
 * standard and should be done with care and ideally rarely. It makes sense to
 * only increase the dust limit after prior releases were already not creating
 * outputs below the new threshold */
static constexpr unsigned int DUST_RELAY_TX_FEE{3000};
```
https://github.com/bitcoin/bitcoin/blob/40e7d4cd0d7f1d922b92b0c640d3d89eef059411/src/policy/policy.h#L64

* [e0abd83b847f2fbece4150a8e9c0829e077cf3f67f7b99853f1bd44099aa4bcb](https://mempool.space/tx/e0abd83b847f2fbece4150a8e9c0829e077cf3f67f7b99853f1bd44099aa4bcb)
* [682a4ac064cf5e4d92b1a89c5cfecb03cc1fdd75c1ed3958f9ac8dd745b8fd63](https://mempool.space/tx/682a4ac064cf5e4d92b1a89c5cfecb03cc1fdd75c1ed3958f9ac8dd745b8fd63)
* [62cbfded2c860c99ce24c6b862ca24cbcfbce32e506f989b76e5955435ba67c1](https://mempool.space/tx/62cbfded2c860c99ce24c6b862ca24cbcfbce32e506f989b76e5955435ba67c1)
* [390768ff00c51f5d76b4a4919aac2120c5a718ec5a4041ba580212fc2656bfa3](https://mempool.space/tx/390768ff00c51f5d76b4a4919aac2120c5a718ec5a4041ba580212fc2656bfa3)
* [bf22ec38d253e9d0cfca4abbb9cfa21322a51d61c3319dc89441a75cd8896d62](https://mempool.space/tx/bf22ec38d253e9d0cfca4abbb9cfa21322a51d61c3319dc89441a75cd8896d62)
* [d90b6662c62240d87fd5cbda97f8267e877ab02c75080746fc8514bcad9f99f7](https://mempool.space/tx/d90b6662c62240d87fd5cbda97f8267e877ab02c75080746fc8514bcad9f99f7)
* [9ee13a3f212ee310b0814420ee1ecfbcc9fc776a57bfe97e3cfb9d3d67baae9d](https://mempool.space/tx/9ee13a3f212ee310b0814420ee1ecfbcc9fc776a57bfe97e3cfb9d3d67baae9d)
* [0ce363883009ca1739e12d3f4481711b531d4ec82e5236314658a45adce023cb](https://mempool.space/tx/0ce363883009ca1739e12d3f4481711b531d4ec82e5236314658a45adce023cb)
* [64a333457107ef4812da605cd193c30334d35cd8e3d793c038f10c0227343c4b](https://mempool.space/tx/64a333457107ef4812da605cd193c30334d35cd8e3d793c038f10c0227343c4b)
* [bcd831a3db1e11d5b73d4b08b99c291589d239d01aedb8f2ca28d7610c7ce83c](https://mempool.space/tx/bcd831a3db1e11d5b73d4b08b99c291589d239d01aedb8f2ca28d7610c7ce83c)
* [c709b28927a3f0093adfb90c441e99aea1aeafc90c7ac6fbf579fcf990b74e12](https://mempool.space/tx/c709b28927a3f0093adfb90c441e99aea1aeafc90c7ac6fbf579fcf990b74e12)
* [12f67059eda9f7ace601d4ecd805841341ec3f6f2aa5e7094eae738f9896caf0](https://mempool.space/tx/12f67059eda9f7ace601d4ecd805841341ec3f6f2aa5e7094eae738f9896caf0)

-------------------------

AdamISZ | 2025-10-15 15:55:05 UTC | #6

[quote="1440000bytes, post:5, topic:2064"]
I don’t think this is true for several policies used in bitcoin core.

[/quote]

You are talking about the purpose of certain policies. That doesn’t contradict what I’m saying about the point of the mempool. Those are two different things.

Still you’re right to point out that simply “you might want to change things because you’re resource constrained” is overly simplistic. The various anti-DOS measures aren’t *quite* that trivial, but still I don’t think any of that changes the fundamental point I was making about decentralization and the purpose of mempools.

-------------------------

