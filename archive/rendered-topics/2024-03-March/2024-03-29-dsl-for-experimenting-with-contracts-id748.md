# DSL for experimenting with contracts

jungly | 2024-03-29 16:50:26 UTC | #1

I want to share a DSL I am building for describing and running bitcoin contracts.

It is **not** another effort to write Script in a higher level language. :sweat_smile:

Instead, the DSL is a tool for describing transactions, sending them to bitcoin node, and running assertions on the state of the system - using a declarative syntax.

The easiest way to describe it is by using an example

![image|355x500](upload://piNLDLRLnHgAhQ1bhLRKJiPLlZG.png)

Docs: https://opdup.com/bitcoin-dsl/index.html

Repo: https://github.com/pool2win/bitcoin-dsl

The key features of the DSL are to provide a declarative syntax for the following:

1. Describe transactions with a high level descriptive syntax 
2. Write locking using miniscript, descriptors and plain old Script
3. Write unlocking scripts with high level constructs again - the runtime automatically tracks witness programs for generating signatures
4. Execute multiple branches of contracts using system state transitions
5. Interact with bitcoin node
6. Assert system state

There is still some more things I want to build, for example, incorporate taproot constructions and making it easy for users to run experimental versions of bitcoin.

For the moment, I have example contracts for lightning (https://opdup.com/bitcoin-dsl/examples/lightning.html) and ARK (https://opdup.com/bitcoin-dsl/examples/ark.html) described using the DSL.

Being able to write the ARK and lightning contracts and execute them on regtest using a high level DSL has been fun for me. I hope others will find the DSL useful too.

Thanks,

Jungly

-------------------------

ajtowns | 2024-03-30 18:44:58 UTC | #2

Looking at the lightning examples, it seems like it would be helpful to have a `reorg_chain` command along the lines of "extend_chain" that combines `invalidateblock` and `generate` and potentially replaces some previously confirmed transactions.

You might consider differentiating keys and addresses -- eg

```raw
@alice_key = key :new 
@alice_addr = address :wpkh(alice)
@alice_to_self = transaction inputs: [
                              { tx: @alice_coinbase_tx, vout: 0, script_sig: '@alice_addr' }
                            ],
                            outputs: [
                              { to: '@alice_addr', amount: 49.99.sats }
                            ]
```

Seeing if you can rewrite [feature_block.py](https://github.com/bitcoin/bitcoin/blob/61de64df6790077857faba84796bb874b59c5d15/test/functional/feature_block.py) might be worth doing -- then the test case could perhaps easily be run on other node implementations, and perhaps it might be easier to add other tests, or easier to understand what the existing tests do if your DSL is actually a better way to describe these things?

Just some ideas, feel free to ignore!

-------------------------

jungly | 2024-03-30 21:52:17 UTC | #3

Thanks for the excellent suggestions.

The address suggestion is neat. It can help break away from the `sig:` prefixed construction. I'll have to see how we can translate that into plain Script usage, e.g. in situations where we do things like `script_sig: 'sig:@alice ""'`, etc.

Writing features.py with this DSL will be a neat way to validate the expressiveness of the DSL, so I'll try to do that. I do want to say that the goals of features.py and this DSL are slightly different. The goal here is to write contracts and system state transitions at a high level to facilitate clearer communication between developers and to quickly experiment with ideas.

`reorg_chain` is interesting idea too. I'll definitely incorporate it.  I have taken a slightly different approach until now and that is to [reset the system state to run a different set of transitions](https://opdup.com/bitcoin-dsl/overview/contract_branch_executions.html), but I can see some situations and developers will benefit from a `reorg_chain` approach.

-------------------------

RobinLinus | 2024-03-31 10:20:06 UTC | #4

Great work! 

We are planning to build something very similar for BitVM. Want to join forces? 
https://bitvm.org/treeplusplus

-------------------------

jungly | 2024-03-31 16:42:27 UTC | #5

Makes a lot of sense to join forces. The DSL has to eventually incorporate advances in contract definitions you guys are pushing forward.

As I see it, the DSL can definitely help with some of the goals mentioned in the "advanced scripting" and "graph of transactions" parts. The only thing I am not sure of and need to figure out is state-fullness.

On the tooling part, I am close to shipping a jupyter notebook to build, share and run DSL scripts. That might be useful to BitVM too. Attached is a sneak peak :wink: 

What's the best place to connect with people working on BitVM? Have you started work on defining constant expressions, templates and opcode composition?

![image|690x318](upload://nYm6o1wqrgpRM5OSXR5KpSjEZGz.png)

-------------------------

ajtowns | 2024-03-31 17:31:14 UTC | #6

[quote="jungly, post:3, topic:748"]
`reorg_chain` is interesting idea too. I’ll definitely incorporate it. I have taken a slightly different approach until now and that is to [reset the system state to run a different set of transitions](https://opdup.com/bitcoin-dsl/overview/contract_branch_executions.html), but I can see some situations and developers will benefit from a `reorg_chain` approach.
[/quote]

For me, the main difference I had in mind would be that if you reset the chain, you have to mine the funding transaction a second time, whereas if you do a reorg you can keep the block that included the funding tx, and be a little bit more sure that you don't accidentally change the funding tx as part of the reset. If you accidently change the coinbase txs when reorging you'll also change the funding tx (which presumably spends some coinbase), and the spends of that funding tx, all of which make for poor test cases if (as in lightning) your contract is meant to be able to be spent in multiple ways.

-------------------------

jungly | 2024-03-31 19:04:06 UTC | #7

That's a very good point, and I agree, we need the `reorg_chain` command.

I'll ping here once I have it shipped.

-------------------------

