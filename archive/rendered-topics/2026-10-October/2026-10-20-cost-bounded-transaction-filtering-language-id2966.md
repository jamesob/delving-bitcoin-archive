# Cost-Bounded Transaction Filtering Language

bkeroack | 2026-10-20 22:33:01 UTC | #1

I designed a transaction filtering policy DSL and implemented it in satd, a full Bitcoin node implementation in Rust. This allows node runners to decide the exact transaction shapes they want their nodes to relay or include in block templates while not affecting consensus correctness.

The language is cost-bounded and intentionally not Turing-complete. Each rule either allows or quarantines a matching transaction, optionally preventing relay or inclusion in template construction. The rules are defined in a text file referenced in the node config and reloaded on SIGHUP. If the node detects that the quarantines might match important protocol transactions such as Lightning channel closures, it either refuses to load the filter (default) or optionally only warns if you use `allowdangerousfilters=1`.

We quarantine rather than reject so the policy is visible and tunable. In addition, it allows the node to process compact blocks more efficiently than if the transactions are discarded. If a transaction is relayed referencing a parent that matches our policy using CPFP, a naive rejection would result in the parent not being found. Instead, we quarantine (with a fixed, separate size budget) so parents and children are always associated correctly. If a parent matches and is quarantined, all children of that transaction are also quarantined.

Here is an example policy that quarantines inscriptions and large OP_RETURN values (probably the most common filtering targets):

```
quarantine big-inscription when
    any input (in.leaf_script.contains_ops(script(OP_FALSE OP_IF push(0x6f7264)))
               and in.witness_size > 1kb)

quarantine big-opreturn when
    sum outputs (out.op_return_size) > 80
```

Net effect if transaction filtering is used? On the network, not too much which is good. Consensus-correct transactions still find their way into most mempools and blocks, as long as some node somewhere (or some out-of-band mining process) agrees to it. So why implement filtering? To allow operator sovereignty and take pressure off node maintainers to choose the "right" defaults. Let users decide what their nodes do and we're likely to see fewer controversies over minor policy disagreements. It's also just the better way to design software, in my opinion. The downside is, if popular, "viral" copy-pasted rulesets could have negative network effects moving forward. I tried to keep the language as simple and understandable as possible to enable operators to write rules by hand. Complex, baroque languages tend towards wholesale copying without any understanding, which we want to avoid.

Open questions: Is it worth exploring more ways to introspect into transactions (decoding)? Should we support more general purpose regexp (within safety bounds)? Are there other danger gates, aside from LN transactions, that should be implemented?

Links:

https://github.com/epochbtc/satd

https://epochbtc.github.io/satd/policy.html

-------------------------

bkeroack | 2026-10-22 14:07:25 UTC | #2

Here's an example screenshot from satd's TUI showing quarantine status

![image|690x169](upload://beOHIoZbd7zJJchHJAzJ7GNuncc.jpeg)

-------------------------

