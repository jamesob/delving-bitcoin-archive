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

