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

Luckylee | 2024-04-02 08:42:43 UTC | #8

We currently use Rust macros at https://github.com/BitVM/rust-bitcoin-script/tree/script_macro to work with Rust Bitcoin's Script.
New scripts and opcodes can be composed and there is some syntactic sugar like loops. Looks like this:
```
fn script_from_func() -> ScriptBuf {
    return script! { OP_ADD };
}

let script = script! {
    for i in 0..3 {
        for k in 0..(3 as u32) {
        OP_ADD
        if k == i {
            OP_SUB
        } else {
            script_from_func
        }
        { i * 42 }
        { k + 42}
        }
    }
    OP_ADD
};
```

-------------------------

jungly | 2024-04-02 10:56:33 UTC | #9

That's really cool.

I wonder if we can make such template using a declarative syntax. It depends on what the above example really wants to do, but maybe something like:

```
"ADD 3.times('ADD ADD (counter from: 0, step: 42) (counter from: 42, step: 1)' ADD"
```

In the above, the `times` generator tracks `i` and `j` and allows the developer to focus on what they want to do, without having to write imperative code.

Could such a declarative syntax make it easier to write scripts for BitVM?

I need to look at the constructs used in BitVM to be able to make a meaningful suggestion. Posting the above just to show an alternative example.

-------------------------

harding | 2024-04-07 04:26:02 UTC | #10

[quote="jungly, post:1, topic:748"]
I want to share a DSL I am building for describing and running bitcoin contracts
[/quote]

This is very cool, thank you for working on it!  I noticed on your personal website that you've also worked on using TLA+ ~~PLA+~~ to verify the completeness of Bitcoin contract protocols (hat tip to @dgpv who has also done [similar work](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-May/017866.html)).   I'm wondering if there's any planned overlap, for example an automated way to extract information from a comprehensive DSL to use with TLA+ ~~PLA+~~?

[Edited to correct name based on @dgpv reply.]

-------------------------

dgpv | 2024-04-06 20:43:12 UTC | #11

> PLA+

It's TLA+ (for "Temporal Logic of Actions")

-------------------------

jungly | 2024-04-09 09:08:16 UTC | #12

Hi David,

Thanks for looking into my other work.

TLA+ is a different beast from this DSL. The goals of the two systems are pretty different with TLA+ being just so much more powerful - but slightly harder to pick up.

### TLA+

With TLA+ you define a system state and possible transitions. TLA+ can then run a "model checker" that goes off and explores all possible reachable states. Once all the possible states have been generated, it asserts that certain properties that we define remain valid in all reachable states. If in any state, our properties are not satisfied, it can produce a trace showing which state transitions lead to that bad state. As you can see, this really helps debug concurrent protocols - just like Bitcoin and LN. I really think specing LN network communication protocols in TLA+ will help a lot. Anyway, I digress.

### Bitcoin DSL 
In contrast, the DSL requires developers to write all state transitions themselves and script their execution - this is similar to writing unit/functional tests for any code. The plus side of the DSL is that it runs your code on regtest, so you really know your scripts will result in real world system transitions. Which I find really helpful personally. I also found it very helpful to learn about various contract constructions - the DSL could be a nice teaching aid.

### Status of my TLA+ work

I am currently [using TLA+ to specify the protocols used in Braidpool](https://blog.opdup.com/specification.html). It really helps capture error in our thinking before we write code. I am hoping this will help Braidpool be more robust to human error :slight_smile: 

I used [TLA+ to capture commitment and breach transactions protocol for LN](https://github.com/pool2win/bitcoin-contracts-tlaplus/blob/794ccf6e9dd9bdd323aa18d3a09648849a066411/LNContractsUsingBitcoinTransactions.tla) last year. I kind of lost focus as I was still working part time on all of this. My goal there was to first define a [model for Bitcoin using TLA+](https://github.com/pool2win/bitcoin-contracts-tlaplus/blob/794ccf6e9dd9bdd323aa18d3a09648849a066411/BitcoinTransactions.tla) and then use that for defining the state transitions resulting from LN contract executions.

I took a quick look at @dgpv's atomic swap contract. It is pretty serious effort. It will be nice to decompose such contracts into multiple modules.

1. Bitcoin blockchain - this captures the chain state and state transitions possible. It is kind of on the back burner atm. I'll get back to it soon.
2. A module of functions to help build contracts - this kind of explodes in complexity, but we can start small and write functions for the most common use cases.
3. Communication between parties - I believe this should be left to vanilla TLA+ or pluscal.

@dgpv - as you can see, I dream of a TLA+ modules library like the [TLA+ Community Modules](https://github.com/tlaplus/CommunityModules).

-kp

-------------------------

shesek | 2024-04-09 22:47:42 UTC | #13

[quote="jungly, post:9, topic:748"]
I wonder if we can make such template using a declarative syntax
[/quote]

Minsc has some functional constructs for looping that might be interesting to check out.

You can use `repeat($n, $script)` for cases where you need to repeat a script fragment exactly N times. For example to create a `rollFromAltStack` function (for statically known N):

```
fn rollFromAltStack($n) = `
  repeat($n, OP_FROMALTSTACK)
  repeat($n - 1, `OP_SWAP OP_TOALTSTACK`)
`;
script = `100 rollFromAltStack(5) OP_ADD`;
``` 
([playground](https://min.sc/next/#c=fn%20rollFromAltStack%28%24n%29%20%3D%20%60%0A%20%20repeat%28%24n%2C%20OP_FROMALTSTACK%29%0A%20%20repeat%28%24n%20-%201%2C%20%60OP_SWAP%20OP_TOALTSTACK%60%29%0A%60%3B%0A%0Ascript%20%3D%20%60100%20rollFromAltStack%285%29%20OP_ADD%60%3B%0A%0Ascript))

The fragment can also be constructed with a function if the index is needed, for example ``` `0 repeat(3, |$i| repeat(5, |$k| `$i OP_ADD {$k+10} OP_SUB`))` ```

There's also `unrollLoop($max, $condition, $body)` that runs a script fragment as long as a condition is met, up to `$max` times. For example, a simple countdown from a number on the stack (up to 50) down to 0:
```
unrollLoop(50, `OP_DUP 0 OP_GREATERTHANOREQUAL`, OP_1SUB)
```
([playground](https://min.sc/next/#c=unrollLoop%2850%2C%20%60OP_DUP%200%20OP_GREATERTHANOREQUAL%60%2C%20OP_1SUB%29), [more advanced example with liquid](https://min.sc/next/#gist=758c25489869d77d4ef624ea43f18c49). `unrollLoop` is itself [implemented](https://github.com/shesek/minsc/blob/a28412f2da02c9952881c36f01b1b5027c7520bd/src/stdlib/stdlib.minsc#L55) in Minsc as part of its stdlib.)

--

Minsc focuses entirely on defining the (Mini)Script-level spending conditions and doesn't deal with higher-level abstracts like Bitcoin DSL does, so perhaps they could somehow work together.

Note that the main https://min.sc website is outdated, the docs are lacking the (non-Miniscript) Script features and the playground runs an old version. A playground that matches the code on [github](https://github.com/shesek/minsc) is available at [https://min.sc/next/](https://min.sc/next/). I don't have updated docs but I did publish some [more advanced examples](https://twitter.com/shesek/status/1508586871819059202) using CTV and Liquid's introspection opcodes.

I should really get this cleaned up and released properly 😅  I haven't been working on this for some time but picking this back up has been on my mind.

-------------------------

sCrypt | 2024-04-11 22:14:30 UTC | #14

We developed [a TypeScript eDSL](https://medium.com/@scryptplatform/introduce-scrypt-a-layer-1-smart-contract-framework-for-btc-b8b39c125c1a) called sCrypt that compiles down to Bitcoin Script. This examples shows NAND gate commitment in BitVM.


```
import {
    assert, ByteString, hash160, method, prop, Ripemd160, SmartContract,
} from 'scrypt-ts-btc'

type HashPair = {
    hash0: Ripemd160
    hash1: Ripemd160
}

export class BitVM extends SmartContract {
    @prop()
    hashPairA: HashPair
    @prop()
    hashPairB: HashPair
    @prop()
    hashPairE: HashPair
    
    constructor(hashPairA: HashPair, hashPairB: HashPair, hashPairE: HashPair) {
        super(...arguments)
        this.hashPairA = hashPairA
        this.hashPairB = hashPairB
        this.hashPairE = hashPairE
    }
    
    @method()
    public openGateCommit(
        preimageA: ByteString,
        preimageB: ByteString,
        preimageE: ByteString
    ) {
        const bitA = DemoBitVM.bitCommit(this.hashPairA, preimageA)
        const bitB = DemoBitVM.bitCommit(this.hashPairB, preimageB)
        const bitE = DemoBitVM.bitCommit(this.hashPairE, preimageE)
        assert(DemoBitVM.nand(bitA, bitB) == bitE)
    }
    
    @method()
    static bitCommit(hashPair: HashPair, preimage: ByteString): boolean {
        const h = hash160(preimage)
        assert(h == hashPair.hash0 || h == hashPair.hash1)
        return h == hashPair.hash1
    }
    
    @method()
    static nand(A: boolean, B: boolean): boolean {
        return !(A && B)
    }
}
```

-------------------------

jungly | 2024-04-23 13:53:27 UTC | #15

Progress update, the DSL now supports taproot outputs - both to create outputs and spending them.

Other updates
1. New [release](https://github.com/pool2win/bitcoin-dsl/pkgs/container/bitcoin-dsl) is out with smaller docker image size and number of notebooks related bug fixes.
3. Support for [reorganising chain](https://opdup.com/bitcoin-dsl/dev/reference.html#reorganise-chain) to a given height, block hash or back up to a block to unconfirm a given transaction.
```
reorg_chain height: 95

reorg_chain blockhash: @blockhash

reorg_chain unconfirm_tx: @reorg_to_tx
```

2. Fixed broken links in the [documentation](https://opdup.com/bitcoin-dsl/index.html) pages.

# Taproot example

New [documentation section on taproot](https://opdup.com/bitcoin-dsl/examples/taproot.html) transaction discusses the choice of using object notation for specifying taproot outputs and script sigs.

Here's a quick example to show how to generate and spend taproot outputs on regtest.

## Create a Taproot Output
```ruby 
transaction inputs: [
              { tx: @coinbase_tx,
                    vout: 0,
                    script_sig: 'sig:wpkh(@alice)' }
                ],
             outputs: [
                  {
                    taproot: { internal_key: @bob,
                               leaves: ['pk(@carol)', 'pk(@alice)'] },
                    amount: 49.999.sats
                  }
                ]
broadcast @taproot_output_tx
```

## Spending Using a Script Path

```ruby 
transaction inputs: [
                  { tx: @taproot_output_tx,
                    vout: 0,
                    script_sig: { leaf_index: 0,
                                  sig: 'sig:@carol' },
                    sighash: :all }
                ],
             outputs: [
                  { descriptor: 'wpkh(@carol)',
                    amount: 49.998.sats }
                ]
broadcast @spend_taproot_output_tx
```

-------------------------

