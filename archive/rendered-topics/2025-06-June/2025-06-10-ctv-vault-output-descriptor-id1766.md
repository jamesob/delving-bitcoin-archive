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

