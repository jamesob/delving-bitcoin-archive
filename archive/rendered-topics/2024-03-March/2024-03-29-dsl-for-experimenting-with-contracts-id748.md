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

