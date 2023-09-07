# OP_VAULT fanfiction for rate-limited and collateralized unvaulting

instagibbs | 2023-08-22 20:42:30 UTC | #1

BIP345 is in roughly [final form](https://github.com/bitcoin/bips/pull/1421), but why not dream a little more?

Here's a small set of modifications I had proposed that would make the proposal slightly more flexible and interesting, at the cost of a few WU. James disagrees it's necessary, but others might find it interesting to think about. A slight cleanup of [OP_FORWARD_PARTIAL](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-March/021528.html) in my humble opinion.

Instead of `OP_VAULT` taking `<revault-amount>`, have it take `<unvault-amount>`. Drop `<revault-vout-idx>`. Instead of pushing `0x01` on the stack, have it push the minimally-encoded "residual" value of the input left over from `<unvault-amount>` onto the stack. This residual will be fed directly into the next opcode...

OP_VAULT args are then just:
```
<leaf-update-script-body>
<n-pushes>
[ n leaf-update script data items ... ]
<trigger-vout-idx>
```
 
Then create `OP_REVAULT` which takes:
```
<revault-amount>
<revault-vout-idx>
```
and returns `0x01` on success.

Typical usage looks like:
```
<revault-vout-idx> <trigger-vout-idx> <trigger-amt>
[ n leaf-update script data items ... ] <n-pushes>
<leaf-update-script-body> OP_VAULT OP_REVAULT
```

But now that these are split, we can do things like add partial collateral(by committing to additional VAULT opcodes):
```
<leaf-update-script-body> OP_VAULT OP_REVAULT
<duplicated args to make sure it's same trigger output spk....>
<collat-amt> OP_VAULT
```
by having multiple `OP_VAULT` opcodes with the explicit amounts.

Or per-utxo-rate-limited unvaults(by committing to the unvaulting amount):
```
<deposit-delay> OP_CSV
OP_DROP OP_DUP <max-val> OP_LEQ OP_VERIFY
<leaf-update-script-body> OP_VAULT OP_REVAULT
```
with some additional math to break up "denominations" in the actual use-case.

-------------------------

instagibbs | 2023-08-22 20:07:29 UTC | #2

Denominations mention I couldn't add to the OP: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-March/021530.html

-------------------------

jamesob | 2023-08-25 19:43:49 UTC | #3

I was watching Johan Halseth's [Surfin' Bitcoin 2023 talk](https://www.youtube.com/watch?v=5ORVoxZ1Wm0) today, and was hit with a use for collateralized vaults.

In his talk, Johan discusses how MATT might be used to create a trust-minimized two-way Taro assets peg (wrapped BTC, basically) to amortize off-chain payments activity.

To peg out from MATT-land, the user would present a claim on-chain accompanied by a proof that all the activity in MATT-land implies their ownership of the coins being claimed. Anyone watching the chain would then be able to contest this claim with a fraud proof; the claim would be collateralized, so that if anyone tries to cheat, the coins they stake to make the claim would be burned if their invalid claim is found out.

This strikes me as a very similar process to the vault workflow that motivated OP_VAULT in the first place, i.e.
- user makes some claim of ownership,
- if the claim is illegitimate, the "real" owner can present a "fraud proof" (i.e. recovery transaction) within some time period to claw back the funds, or
- if the claim is legit, time period elapses, and user claims funds.

So I wonder if OP_VAULT can be used to make this kind of two-way peg penalty enforced proof/claim process more efficient on-chain -- especially when it comes to batching multiple inputs from MATT-land, or some equivalent.

---

I mention this here because collateralization might be an important part of that process, so it may be something that we want to think more deeply about designing for.

An issue I have with the incantation you provide above,

> ```
> <leaf-update-script-body> OP_VAULT OP_REVAULT>
> <duplicated args to make sure it's same trigger output spk....>
> <collat-amt> OP_VAULT
> ```

is that we have to do a lot of annoying stack magic to duplicate the OP_VAULT arguments. 

I think a while back I had a counter proposal to introduce (yet another) OP_VAULT argument, `<collateral-amt>`, that would make collateral lockup a first-class usecase.

The problem with that is collateral specification isn't so simple - do you want to require x BTC per input being unvaulted, or for a single unvault operation? Or maybe you want to be able to specify a percentage of the amount that has to be locked for collateral? Adding a single `collateral-amt` arg makes these kind of things hard.

Perhaps an `<collateral-type> <collateral-amt> OP_VAULT_COLLATERAL` "decorator" opcode (that would precede the `OP_VAULT` evaluation) might be warranted?

-------------------------

instagibbs | 2023-08-25 20:29:03 UTC | #4

> annoying stack magic

This cases is mostly a bit of `OP_XDUP` overhead which just so happens to enable a feature. Something special-cased that *doesn't* need it will require interpreter-wide state to know where the value should be going which is what I was attempting to avoid in the first place with the replacement of `OP_FORWARD_PARTIAL`. I'm not sold that collateral should be a first-class citizen, just interesting that a potentially useful feature is possible with little additional spec complexity(I'd argue the complexity is a lateral move myself).

> The problem with that is collateral specification isn’t so simple - do you want to require x BTC per input being unvaulted, or for a single unvault operation? Or maybe you want to be able to specify a percentage of the amount that has to be locked for collateral?

Yeah this is where we're definitely at the limit of Bitcoin Script. We'd essentially be re-implementing `OP_DIV` et al just implicitly, maybe side-stepping `CScriptNum` arithmetic issues by doing it implicitly. Percentage-based rate-limiting of unvaults would also be great, and I could imagine it being used in many other contexts if it was a more general-purpose widget. 

Limits of Bitcoin Script, Simplicity right meow, you know the drill

-------------------------

instagibbs | 2023-09-05 14:15:58 UTC | #5

I wrote up these thoughts more explicitly in BIP PR form [here](https://github.com/jamesob/bips/pull/4)

edit: also added some last-minute MVP changes to demonstrate how %-based rate-limiting and collateral could work. Very small changes; a little hacky, just demonstrating what can be done

-------------------------

