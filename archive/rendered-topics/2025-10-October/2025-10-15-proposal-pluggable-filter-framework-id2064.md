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

