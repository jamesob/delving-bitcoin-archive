# Delving Simplicity Part Ⅴ: Programs and Addresses

roconnor-blockstream | 2025-11-17 20:32:26 UTC | #1

In [Part Ⅳ](https://delvingbitcoin.org/t/delving-simplicity-part-two-side-effects/2091) of this series, we discussed the side effects that Simplicity expressions can have. In particular, for Bitcoin and Liquid applications, Simplicity expressions can have a Reader effect, which provides read-only access to the transaction data and the Failure effect, which determines whether a transaction is successful or not.

The program for a UTXO is ultimately just a function that decides whether a given transaction is acceptable to redeem that particular UTXO’s funds. To this end, we define a *Simplicity program* as a Simplicity expression of type `𝟙 ⊢ 𝟙`. We rely on the Reader effect to capture the input (the transaction environment) and a Failure effect to capture the output (success or failure) of this program.

This means that Simplicity types are not used for the input and output of Simplicity programs, but rather Simplicity types are used to ensure the soundness of the internal composition of subexpressions within a Simplicity program.

# Commitment Merkle Root

We do not directly store an entire Simplicity program in a transaction’s output. Since the days of Bitcoin’s Pay-to-Script-Hash (P2SH), Bitcoin has stored only commitments to programs in transaction outputs. This, among other benefits, allows for a uniform addressing scheme for all programs, no matter their complexity.

For Simplicity programs, we use a *commitment Merkle root* (or *CMR)* for our commitment. Each Simplicity combinator has an associated tag, which is a SHA-256 hash of an identifying ASCII string

| <!-- --> | <!-- --> |
|---|---|
| $tag_{iden}$ | = SHA-256(`Simplicity␟Commitment␟iden`) |
| $tag_{comp}$ | = SHA-256(`Simplicity␟Commitment␟comp`) |
| $tag_{unit}$ | = SHA-256(`Simplicity␟Commitment␟unit`) |
| $tag_{injl}$ | = SHA-256(`Simplicity␟Commitment␟injl`) |
| $tag_{injr}$ | = SHA-256(`Simplicity␟Commitment␟injr`) |
| $tag_{case}$ | = SHA-256(`Simplicity␟Commitment␟case`) |
| $tag_{pair}$ | = SHA-256(`Simplicity␟Commitment␟pair`) |
| $tag_{take}$ | = SHA-256(`Simplicity␟Commitment␟take`) |
| $tag_{drop}$ | = SHA-256(`Simplicity␟Commitment␟drop`) |

where `␟` stands for ASCII code 31.

Simplicity expressions are recursively hashed to form a 256-bit CMR by computing tagged SHA-256 midstates for each combinator and its arguments.

| <!-- --> | <!-- --> |
|---|---|
| #ᶜ(`iden`) | = SHA-256-midstate $(tag_{iden} \parallel tag_{iden})$ |
| #ᶜ(`comp` $f$ $g$) | = SHA-256-midstate $(tag_{comp} \parallel tag_{comp} \parallel \#^c(f) \parallel \#^c(g))$ |
| #ᶜ(`unit`) | = SHA-256-midstate $(tag_{unit} \parallel tag_{unit})$ |
| #ᶜ(`injl` $f$) | = SHA-256-midstate $(tag_{injl} \parallel tag_{injl} \parallel 32\cdot\texttt{0x00} \parallel \#^c(f))$ |
| #ᶜ(`injr` $f$) | = SHA-256-midstate $(tag_{injr} \parallel tag_{injr} \parallel 32\cdot\texttt{0x00} \parallel \#^c(f))$ |
| #ᶜ(`case` $f$ $g$) | = SHA-256-midstate $(tag_{case} \parallel tag_{case} \parallel \#^c(f) \parallel \#^c(g))$ |
| #ᶜ(`pair` $f$ $g$) | = SHA-256-midstate $(tag_{pair} \parallel tag_{pair} \parallel \#^c(f) \parallel \#^c(g))$ |
| #ᶜ(`take` $f$) | = SHA-256-midstate $(tag_{take} \parallel tag_{take} \parallel 32\cdot\texttt{0x00} \parallel \#^c(f))$ |
| #ᶜ(`drop` $f$) | = SHA-256-midstate $(tag_{drop} \parallel tag_{drop} \parallel 32\cdot\texttt{0x00} \parallel \#^c(f))$ |

At redemption time, a Simplicity program with a matching CMR is revealed. We use a Merkle root for the CMR because we want to be able to prune unused branches from the revealed program.

Notice that the CMR does not commit to the types of Simplicity expressions. Simplicity will use type inference to reconstruct the minimal typing of the revealed Simplicity program.

We use SHA-256 midstates so that each expression requires at most one call to the SHA-256 compression function, assuming one precomputes the midstate up to the tags. In the case of one-argument constructors, we prefix the argument with 32 bytes of `0x00` padding because it allows for a small amount of extra precomputation if developers wish to implement it.

# Addresses

Addresses for Simplicity programs use [BIP-0341](https://github.com/bitcoin/bips/blob/bbaea3182b258a4eadc100279ec71c7a4a30482e/bip-0341.mediawiki)’s Taproot mechanism with the program’s CMR committed under a new TapLeaf version number. The Simplicity program’s CMR is hashed with the Simplicity leaf version byte to form Simplicity’s TapLeaf tagged hash.

For Liquid and Elements, this would be
<div align="center">

$hash_{TapLeaf/elements}(\texttt{0xbe} \parallel \texttt{0x20} \parallel CMR)$
</div>

where `0xbe` is Simplicity’s TapLeaf version and `0x20` is the length of a CMR. A tagged hash is defined in [BIP-0340](https://github.com/bitcoin/bips/blob/bbaea3182b258a4eadc100279ec71c7a4a30482e/bip-0340.mediawiki#description) as

<div align="center">

$hash_{str}(x) = \textit{SHA-256}(\textit{SHA-256}(str) \parallel \textit{SHA-256}(str) \parallel x)$.

</div>

From here, other TapBranches could be added containing other TapLeaves, which themselves can be either other Simplicity programs or even other Script programs.

Finally, the root of the TapTree is used in a TapTweak to tweak an internal public key and generate an output public key used to create a Segwit v1 address. If no key-spend path is desired, a NUMS point must be used for the internal public key. For more details, consult [BIP-0341](https://github.com/bitcoin/bips/blob/bbaea3182b258a4eadc100279ec71c7a4a30482e/bip-0341.mediawiki#constructing-and-spending-taproot-outputs).

## From Simplicity to Address

Let’s create an address for the simplest Simplicity program possible: `unit : 𝟙 ⊢ 𝟙`. This is a no-op program that always succeeds.

First, we compute the unit combinator’s tag:
<div align="center">

$tag_{unit}$= `0xd723083cff3c75e29f296707ecf2750338f100591c86e0c71717f807ff3cf69d`
</div>

Using this tag, we can compute the CMR for the `unit` expression.
| <!-- --> |
|---|
| #ᶜ(`unit`) |
| = |
| SHA-256-midstate $(tag_{unit} \parallel tag_{unit})$ |
| = |
| SHA-256-midstate(`0xd723083cff3c75e29f296707ecf2750338f100591c86e0c71717f807ff3cf69d`&#x2028;∥`0xd723083cff3c75e29f296707ecf2750338f100591c86e0c71717f807ff3cf69d`) |
| = |
| `0xc40a10263f7436b4160acbef1c36fba4be4d95df181a968afeab5eac247adff7` |

Next, we compute a TapLeaf tagged hash by prefixing the CMR with Simplicity’s TapLeaf version code and the CMR length.
| <!-- --> |
|---|
| $hash_{TapLeaf/elements}(\texttt{0xbe} \parallel \texttt{0x20} \parallel CMR)$ |
| = |
| $hash_{TapLeaf/elements}$(`0xbe20c40a10263f7436b4160acbef1c36fba4be4d95df181a968afeab5eac247adff7`) |
| = |
| `0x44cc38311ec7e5dfb7b573baf38449496ecd334eb5509cfed1b4fd30da8dd41c` |

In this example, our TapTree will only have this one leaf, so we won’t be introducing any TapBranches. Next, we need to use this hash to tweak some public key. Since we don’t want to enable a keyspend path in our example, we will use the NUMS point specified in [BIP-0341](https://github.com/bitcoin/bips/blob/bbaea3182b258a4eadc100279ec71c7a4a30482e/bip-0341.mediawiki#constructing-and-spending-taproot-outputs). In a real application, we would randomize this NUMS point as recommended by [BIP-0341](https://github.com/bitcoin/bips/blob/bbaea3182b258a4eadc100279ec71c7a4a30482e/bip-0341.mediawiki#constructing-and-spending-taproot-outputs).
<div align="center">

*internal_pk* = `0x50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0`
</div>

The next step is to compute the tweaked hash of this public key with our TapLeaf hash.
| <!-- --> |
|---|
| $hash_{TapTweak/elements}$(`0x50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0`&#x2028;∥ `0x44cc38311ec7e5dfb7b573baf38449496ecd334eb5509cfed1b4fd30da8dd41c`) |
| = |
| `0xb3bef172389b0937d7e5a8b15cfa41e776777f13f2f659cb06220a6ff0658285` |

Then we need to compute the Taproot output key by tweaking our internal public key by this value. This involves some elliptic curve calculations.
| <!-- --> |
|---|
| *output_pk* |
| = |
| lift_x(`0x50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0`) &#x2028;⊕ `0xb3bef172389b0937d7e5a8b15cfa41e776777f13f2f659cb06220a6ff0658285`·G |
| = |
| lift_x(`0x2cb0c20acd7340b4d4b65f6a60e2888d0d64e3267261f3b3cf7290e5af3f9e09`) |

Next, we need to convert this x-only output public key into [Bech32](https://github.com/bitcoin/bips/blob/66a41d32bf9c4dfecd80a81f1165dd3f59b5374b/bip-0173.mediawiki)’s alphabet:
<div align="center">

`9jcvyzkdwdqtf49kta4xpc5g35xkfcexwfsl8v70w2gwttelncys`
</div>

We prefix this string with a `p`, indicating this is a Segwit V1 address, and then we add a Bech32 prefix. For the Liquid testnet, this prefix is `tex1`. Then we append the [Bech32m](https://github.com/bitcoin/bips/blob/66a41d32bf9c4dfecd80a81f1165dd3f59b5374b/bip-0350.mediawiki) checksum. In this example, the checksum is `hxjk56`.

Finally, we have the address for our little Simplicity program:
<div align="center">

[`tex1p9jcvyzkdwdqtf49kta4xpc5g35xkfcexwfsl8v70w2gwttelncyshxjk56`](https://blockstream.info/liquidtestnet/address/tex1p9jcvyzkdwdqtf49kta4xpc5g35xkfcexwfsl8v70w2gwttelncyshxjk56)
</div>
Whew! That was a lot of work; however, much of this work is mandated by Taproot and is not Simplicity specific.

# Witness expressions

There is one more kind of input data that we have neglected so far: signature data and other witness data. This kind of input is separate from the transaction environment that the Reader effect can access. However, because Simplicity programs have no input, there appears to be no place to put signature data.

Our remedy is to add a new kind of Simplicity expression: the *witness expression*.

```none
      w : B
-----------------
witness w : A ⊢ B
```

This is the only combinator that takes an actual Simplicity value as an argument.

The semantics of the witness expression is that it ignores its input and just returns the value `w`, which can be of any Simplicity type.

`⟦witness w⟧(a) = w`

We know from the Simplicity completeness theorem that this function can already be expressed in Simplicity. In particular, in [Part Ⅱ](https://delvingbitcoin.org/t/delving-simplicity-part-combinator-completeness-of-simplicity/1935) of this series, we saw the `scribe` macro that constructs such expressions.

The purpose of the witness expression does not lie in its functional behavior, but rather in its CMR.

First, we define the witness commitment tag.

| <!-- --> | <!-- --> |
|---|---|
| $tag^c_{witness}$ | = SHA-256(`Simplicity␟Commitment␟witnesss`) |

Then, we define the CMR for witness expressions.
| <!-- --> | <!-- --> |
|---|---|
| #ᶜ(`witness` $w$) | = SHA-256-midstate $(tag^c_{witness} \parallel tag^c_{witness})$ |

Notice that the value `w` is **excluded** from the expression’s CMR. This means we can calculate an address for an expression before we know the value `w`. In other words, we get to set the value of `w` at redemption time. Whenever we need witness data, such as a digital signature, we can place it inside a witness node.

## Witness Values

It may seem like a limitation that witness expressions contain only values and not other Simplicity expressions more generally. However, programs for UTXO-based blockchains are executed only once. We don’t need to pass a whole Simplicity expression into a witness expression because the user could instead run that expression themselves and transcribe its output into the witness’s value to get the same result.

That said, later on we will introduce the `disconnect` combinator, and as we will see, it behaves much like a witness expression that takes an entire Simplicity expression as an argument.

An alternative design would be to take all witness data as an argument to the Simplicity program. We prefer to use witness expressions because of pruning. Pruning will be discussed later in this series, but the idea is that unexecuted branches of `case` expressions are not revealed on-chain. In particular, any witness expressions within those branches also get pruned away. Witness expressions also let us put witness values right where they are needed, rather than having to propagate those values down from the top-level program input.

# Conclusion

In this part, we defined a Simplicity program as a Simplicity expression from `𝟙` to `𝟙`. We defined the commitment Merkle root (or CMR) of Simplicity expressions and used it to form a commitment to a Simplicity program. We also introduced witness expressions, which contain witness data that is excluded from this commitment. We also saw a worked example of how to construct an address for a trivial Simplicity program.

In the next part, we will talk about jets, how their CMRs are constructed, and use some jets to construct a simple single-signature checking program.

-------------------------

