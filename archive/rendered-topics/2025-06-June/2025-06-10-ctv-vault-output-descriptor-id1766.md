# CTV vault output descriptor

sjors | 2025-06-10 12:12:07 UTC | #1

Given the [renewed effort](https://groups.google.com/g/bitcoindev/c/KJF6A55DPJ8/m/ZWhVgOm7AQAJ) to prioritise `OP_CTV` ([BIP 119](https://github.com/bitcoin/bips/blob/dbb9617e5f2c3e99d2d07f0b82dbb4ad861ad06e/bip-0119.mediawiki)) over other soft forks that can enable vaults (e.g. `OP_CCV`) and the [withdrawal of OP_VAULT](https://delvingbitcoin.org/t/withdrawing-op-vault-bip-345/1670), it's useful to revisit how one can build vaults with OP_CTV.

@jamesob designed such a vault a few years ago:

https://github.com/jamesob/simple-ctv-vault

> OP_CTV allows the vault strategy to be used without the need to maintain critical presigned transaction data for the lifetime of the vault, as in the case of earlier vault implementations. This approach is much simpler operationally, since all relevant data aside from key material can be regenerated algorithmically. This makes vaulting more practical at any scale.

It could be modernised to make use of ephemeral (dust) anchors (see [BIP 341](https://github.com/bitcoin/bips/blob/dbb9617e5f2c3e99d2d07f0b82dbb4ad861ad06e/bip-0431.mediawiki)), and to use taproot script paths rather than `OP_IF`.

But for any vault construction to be useful in wallets, it needs to either fit in the existing [BIP 380](https://github.com/bitcoin/bips/blob/dbb9617e5f2c3e99d2d07f0b82dbb4ad861ad06e/bip-0380.mediawiki) output descriptor paradigm or develop an alternative.

In the context of `OP_CHECKCONTRACTVERIFY` (`OP_CCV`, [BIP 443](https://github.com/bitcoin/bips/blob/dbb9617e5f2c3e99d2d07f0b82dbb4ad861ad06e/bip-0443.mediawiki)) @salvatoshi [wrote](https://github.com/bitcoin/bips/pull/1793#issuecomment-2749295131):

> My general take is that descriptors are the wrong tool for this purpose: a spend from UTXO X to UTXO Y where f(Y) = X needs to somehow encode the relation between X and Y as a *predicate* . While you could do that (every program is a predicate, and every program expressible in Script is a predicate that can be expressed in a tree structure like miniscript...) it quickly becomes unmanageable.

Perhaps this reasoning applies to `OP_CTV` as well. But given that it's less powerful, perhaps for a simple use case like vaults it can still work? It would be more likely to gain adoption given existing infrastructure.

Let's take a simple 2-of-2 multisig, where after an N block delay either party can spend the coins. 

This can already be done, e.g. with Alice (`A`) and Bob (`B`):

`tr(musig(A, B), {and_v(pk(A), older(N)),...})`

However this requires Alice and Bob to move their coins at least every `N` blocks. Higher values of $N$ means fewer such movements, but also a slower recovery time. Additionally [BIP 68](https://github.com/bitcoin/bips/blob/dbb9617e5f2c3e99d2d07f0b82dbb4ad861ad06e/bip-0068.mediawiki#compatibility) limits relative time and height locks to about a year, so coins have to be rotated at least that often.

This is where a (CTV) vault can come in handy.

The key path remains `musig(A, B)`, but the script paths allow either party, e.g. Alice, to move coins into a vault. After N blocks Alice can move the coins using a pre-determined key under her control (e.g. just `A`).

If Alice believes her key was compromised, or if Bob didn't actually lose his key, either of them can immediately and without a signature send the coins back. Back *where* though?

With CCV the vault could be recursive, but afaik not with CTV. That's ok because we wouldn't want to go in circles anyway. Instead the fallback could be the design we started out with: `tr(musig(A, B), {and_v(pk(A), older(N)),...})`. That gives both parties N blocks time to sort things out between them, after which either side can move the coins. Including the delay of the vault itself, they have `2 * N` blocks time.

So what would this look like as descriptor? How about:

`tr(musig(A, B),{and_v(pk(A),ctv(musig(A, B), {unvault_cold, unvault_hot}), ...})`

Where:
- `unvault_hot` is `and_v(older(N), pk(A))`
- `unvault_cold` is `ctv(musig(A, B), {and_v(pk(A), older(N)),...})`

Unvaulting back to cold can be done without any signature, hence the nested `ctv()`, which is good when Alice and/or Bob are racing an attacker. But if they have some extra time, they could double-spend the unsigned unvault using the `musig(A,B)` keypath and send coins straight to where they want it (even back to the original vault).

The `ctv` fragment has the same syntax as the `tr()` descriptor and would have very limited functionality here. It's just the key path and list of script paths that the committed transaction must send to. Perhaps it should be called `vault()`.

Assuming the above makes any sense, it'd love to see someone implement it...

-------------------------

1440000bytes | 2025-06-10 14:51:28 UTC | #2

[quote="sjors, post:1, topic:1766"]
It could be modernised to make use of ephemeral (dust) anchors (see [BIP 341](https://github.com/bitcoin/bips/blob/dbb9617e5f2c3e99d2d07f0b82dbb4ad861ad06e/bip-0431.mediawiki)), and to use taproot script paths rather than `OP_IF`.

But for any vault construction to be useful in wallets, it needs to either fit in the existing [BIP 380](https://github.com/bitcoin/bips/blob/dbb9617e5f2c3e99d2d07f0b82dbb4ad861ad06e/bip-0380.mediawiki) output descriptor paradigm or develop an alternative.
[/quote]

https://min.sc/v0.3/#github=examples/ctv-vault.minsc

Cc: @shesek

-------------------------

sanket1729 | 2025-06-12 01:11:15 UTC | #3

FTR, I am slowly warming to @salvatoshi's view that descriptors might be the wrong language to play with this. But I'm still exploring the idea as mentioned in the post. Descriptors **must** convey **all** the information necessary to spend the output.

> [quote="sjors, post:1, topic:1766"]
> `tr(musig(A, B), {and_v(pk(A), ctv(musig(A, B), {unvault_cold, unvault_hot}), ...)})`
> [/quote]

CTV spec specifies a bunch of other fields that are going to be hashed. For example, we need to decide what sequence value to set for the transaction, as well as the complete serialization of all inputs and outputs. If we don't store all of this information, then we don't satisfy the property that descriptors must contain all the information required to spend from it.

One naive attempt could be to encode the entire transaction as hex in the descriptor. But we still want the flexibility to express BIP32 keys in there. Maybe if we had a new `ctv_tx` fragment in the descriptor language that looks like:

```
ctv_tx(version, nlocktime, inputs_hash, [out_desc1, out_desc2, ...])
```

where each `out_desc_i` is a BIP380 descriptor in itself, that would be a complete specification. This is clearly clunky, and maybe with some work we can simplify the fragment for common use cases. But in any such simplification, we must preserve the property that the descriptor contains all information required to spend the output. (minus the private key of course). 

[quote="sjors, post:1, topic:1766"]
Assuming the above makes any sense, it’d love to see someone implement it…
[/quote]

Happy to try to do this once I get some spare time

-------------------------

ajtowns | 2025-06-12 02:28:54 UTC | #4

[quote="sanket1729, post:3, topic:1766"]
where each `out_desc_i` is a BIP380 descriptor in itself,
[/quote]

I think you'd need to include amounts for each output, in addition to a descriptor that can produce the scriptPubKey? You'd probably want some other expression that can convert `total_in_amount_pct(50)` to 50% of the sum of all the input amounts or similar.

-------------------------

