# How CTV+CSFS improves BitVM bridges

RobinLinus | 2025-04-10 02:30:33 UTC | #1

**TL;DR:** CTV fundamentally improves BitVM bridges by eliminating the need for a presigning committee and the existential honesty assumption for safety. Additionally, CSFS drastically improves capital efficiency and reduces setup overhead.

### Introduction
In BitVM, we use input-committing covenants to represent state. E.g., we express "the operator can withdraw funds from the bridge if no challenge occurred" with a specific output that challengers may burn to indicate that they believe the operator's withdrawal request was malicious, forcing the operator to go through the challenge process. In contrast, in the happy path, if that output "survives" the challenge period then the operator can use it to execute the withdrawal transaction.


The current BitVM bridge design relies on emulated covenants to guarantee safety of the deposits. This committee requires a 1-of-n trust assumption and introduces various undesirable complexities in practice. Up until recently we had assumed that CTV is not sufficient to replace the committee because CTV is designed to commit only to _outputs_ but not to _inputs_. However, it turned out that there is a trick that enables the functionality that we need.

### The scriptSig Trick
The key idea is to use the fact that CTV commits to the scriptSig of all inputs.
Say we want to express "inputA is spendable only together with inputB". 
1. Define inputB to be a (legacy) P2SH output. 
2. Presign a signature using sighash `SINGLE|NONE`, effectively signing only inputB. This signature commits to inputB and since P2SH is not SegWit the signature will be in inputB's scriptSig.
3. Define inputA to be a P2TR output and contain a CTV condition with a template hash that commits to the scriptSigs, including the signature for inputB. 

The result: **inputA** commits to the signature for **inputB**, which itself commits to **inputB**. So **inputA** becomes spendable only in conjunction with **inputB**.

### Details
These commitments are one-way: **inputA** commits to **inputB**, but not vice versa. Two-way commitment would ideally be desirable but is impossible due to hash cycles (even with `TXHASH` or introspection opcodes).

For our bridge it would be sufficient to constrain **inputB** to always be burned. But that too is unachievable with CTV—its template hash would have to commit to its own `scriptSig`, which already commits to **inputB**, and therefore its TXID—creating a hash cycle.

To work around this, we redesigned the transaction graph such that the contract remains secure even if **inputB** is spent independently of **inputA**. The exact mechanics of that redesign are beyond the scope of this post, as they require a deeper dive into the bridge contract architecture.


### Concrete Improvements

- **CTV enables unconditional safety of deposits.** Even if **all** operators are malicious, they cannot steal from the bridge. While we still rely on an **existential liveness assumption** to ensure withdrawals, the simplified setup allows us to scale the operator set significantly—potentially by an order of magnitude. This makes it far more likely that at least one operator remains live.

- **CSFS replaces Lamport signatures**, reducing transaction sizes by approximately **10x**. This dramatically lowers the capital cost required for bridge operations.

- **Non-hardened key derivation** becomes possible with CSFS, enabling anyone to compute the operator's public keys **non-interactively**. This greatly simplifies the peg-in process by reducing the amount of data operators need to provide.

- In the current design, **peg-ins still require an operator signature**, which allows for censorship. While this can be mitigated in the side system, our long-term goal is to **eliminate this interaction entirely** by modifying the bridge contract to support fully non-interactive peg-ins.

For these reasons, the BitVM Alliance strongly supports the [CTV + CSFS](https://delvingbitcoin.org/t/ctv-csfs-can-we-reach-consensus-on-a-first-step-towards-covenants/1509/11) proposal.

### Conclusion
This trick demonstrates that with a clever use of CTV and CSFS, we can dramatically simplify BitVM bridge construction while improving both safety and efficiency. By removing the need for a presigning committee and eliminating the existential honesty assumption, CTV enables bridges with unconditional safety guarantees. At the same time, CSFS reduces capital costs and operational complexity, paving the way for more scalable and decentralized bridges. While a few open challenges remain—such as making peg-ins fully non-interactive—this design represents a significant step forward in practical, trust-minimized Bitcoin interoperability.

-------------------------

Cyimon | 2025-04-10 14:28:23 UTC | #4

![image|689x411](upload://8BdEnPVfhewYeSZ8YaYIS99GlRD.png)
I tried to integrate OP_CTV in BitVM2 Bridge based on your idea. And this is the graph. Are there any mistakes?

-------------------------

ekrembal | 2025-04-10 16:09:12 UTC | #5

![bitvm_bridge_opctv|640x500](upload://nHkN9JnYCqgwQrk6JFN0zkvuZuz.jpeg)

Here is a transaction graph that supports efficient collateral usage for operators:  
- Operators provide signatures for the red arrows at the time of deposit.  
- Every kickoff must be finalized before the operator sends the **ready-to-reimburse transaction**.  
- A kickoff can get finalized with either **challenge timeout transaction** or **disprove timeout transaction**.
- Kickoffs can proceed in parallel, allowing many withdrawals to be processed with a single collateral.  
- If one of the kickoffs is disproven, all reimbursements stop.

Current Limitations:  
- A Bitcoin light client is needed to prove the inclusion of the payout transaction and the sidesystem's state.  (We are still working on this, but this also might add trust-minimized assumptions for safety)
- All operators' signatures need to be published somewhere for data availability, and Bitcoin is not ideal for this due to limited block space.

-------------------------

RobinLinus | 2025-04-11 17:37:20 UTC | #6

It seems I can’t edit my post anymore, but there’s an error:

[quote="RobinLinus, post:1, topic:1591"]
Presign a signature using sighash `SINGLE|NONE`
[/quote]

It should say:

[quote="RobinLinus, post:1, topic:1591"]
Presign a signature using sighash `ANYONECANPAY|NONE`
[/quote]

-------------------------

1440000bytes | 2025-04-15 00:00:02 UTC | #7

Added the link in https://en.bitcoin.it/wiki/Covenants_Uses

-------------------------

ajtowns | 2025-04-16 09:48:04 UTC | #8

[quote="RobinLinus, post:1, topic:1591"]
The key idea is to use the fact that CTV commits to the scriptSig of all inputs. Say we want to express “inputA is spendable only together with inputB”.

1. Define inputB to be a (legacy) P2SH output.
2. Presign a signature using sighash [`NONE|ANYONECANPAY`], effectively signing only inputB. This signature commits to inputB and since P2SH is not SegWit the signature will be in inputB’s scriptSig.
[/quote]

I don't think this works? As I understand it, with this setup you have something like:

```txt
utxo A:   10 BTC, p2tr: ... <H> CTV
utxo B:  500 sats, p2sh: <P> CHECKSIG
```

with the idea being that you spend utxo A via the CTV with:

```txt
input 1: utxo A, witness reveals CTV path and any other conditions, no scriptSig
input 2: utxo B, scriptSig = "<S;NONE|ANYONECANPAY>" "<P> CHECKSIG"
output: whatever
```

where `<H>` locks in where the output goes to, and input 2's scriptSig. But because H is only committing to the second input's scriptSig, then it's easy to construct another utxo that can be used instead, eg one with the (non-standard) scriptPubKey `OP_2DROP OP_TRUE`:

```txt
utxo C:  500 sats, scriptPubKey: OP_2DROP OP_TRUE

input 1: utxo A, no witness/scriptSig
input 2: utxo C, scriptSig = "<S;NONE|ANYONECANPAY>" "<P> CHECKSIG"
output: whatever
```   

That allows utxo A to be spent via the CTV path independently of whether utxo B has already been spent/burnt, which, as far as I can see, breaks the protocol you're trying to enforce.

(I'm assuming in a real example utxo B's spend condition is more complicated than `<P> CHECKSIG`, as otherwise the availability of a NONE|ANYONECANPAY signature means it can be spent immediately, so a griefer could fairly easily prevent the happy path from ever being taken)

[quote="RobinLinus, post:1, topic:1591"]
These commitments are one-way: **inputA** commits to **inputB**, but not vice versa. Two-way commitment would ideally be desirable but is impossible due to hash cycles (even with `TXHASH` or introspection opcodes).

For our bridge it would be sufficient to constrain **inputB** to always be burned. But that too is unachievable with CTV—its template hash would have to commit to its own `scriptSig`, which already commits to **inputB**, and therefore its TXID—creating a hash cycle.
[/quote]

I believe this sort of setup would actually be fairly simple to achieve with generic introspection opcodes: you first construct all your outputs simultaneously in a single transaction, eg:

```txt
output 1: 10 BTC, collateral
output 2: 500 sats, assertion output
output 3: 500 sats, challenge output
```

then you can setup "output 1" with a "happy path" script that checks that the second input has the same txid as this input, and spends the third output of that tx, and conversely for that output 3 has a happy path script where the first input has the same txid and spends the first output of that tx. Because a generic introspection opcode lets you just query an input's txid, that doesn't create any cryptographically intractable loops, and you can apply equivalent logic for each output.

In bllsh, you could write that as:

```
(all (= (tx 11) (tx '(11 . 1))) (= (+ (tx '(12 . 1))) 2))
```

where `(tx 11)` takes the current input's prevout's txid; `(tx '(11 . 1))` takes the prevout's txid for the input at position 1 (ie, the second input); `(tx '(12 . 1))` takes that prevout's index; and then it checks that the index is numerically 2, and the txid's are the same. Executable example in the [repo](https://github.com/ajtowns/bllsh/blob/5c981a57607452be8cb6dc4d83cb8bae7842cf99/examples/test-sibling-prevout).

If you can't construct all the inputs simultaneously, it gets a lot uglier, since you probably have to prove ancestry, which is [another thread entirely](https://delvingbitcoin.org/t/contract-level-relative-timelocks-or-lets-talk-about-ancestry-proofs-and-singletons/1353/1)...

-------------------------

instagibbs | 2025-04-16 10:58:29 UTC | #9

The scriptSig could include the `CHECKSIG` opcode directly, contra standardness rules. :grimacing:

In the original idea, a p2sh redeemscript is just pushes, so the spk could just be blank and it would pass, since cleanstack isn't consensus anyways.

-------------------------

ajtowns | 2025-04-16 11:19:31 UTC | #10

[quote="instagibbs, post:9, topic:1591"]
The scriptSig could include the `CHECKSIG` opcode directly, contra standardness rules. :grimacing:
[/quote]

You'd also be violating the `CONST_SCRIPTCODE` standardness check, I think, because you'd need `FindAndDelete` to actually delete the signature from the scriptcode (or you'd need to use OP_CODESEP to do the same thing manually). Would probably also mean you need two CHECKSIG ops, one in the scriptSig for CTV, and one in the scriptPubKey so that it's not anyone can spend. You'd also presumably need to repeat the public key, though could possibly at least reuse the signature.

Not doing it as p2sh would also make it cumbersome to have any more complicated logic in your spending condition.

-------------------------

JeremyRubin | 2025-04-16 13:49:57 UTC | #11

This is an interesting point; without some sort of "stack sentinel" that guarantees a specific script type, using CTV as a gadget for any P2SH type seems broken, as you can replace it with a legacy script that does something else. This is "confusing" because you cannot replace it with a p2sh script that does something else. I can make an effort to better document this issue...

wrt the legacy requirement, what you *can* do is as follows:

use a legacy input B which has 

    scriptSig: [other program stuff] <sig || ALL|ACP > Dup <pk> checksig
    script: <pk> checksig

you are free to include more inputs like this for more data.

What you can also do is employ a taproot-adapter to create B. That is a output X like:

    tr(NUMS, {[general program] <1-in-1-out w/ <pk> checksig> CTV)

You can then bind B as usual from A, and A's commitment to B's Outpoint via B's signature guarantees that the output X has executed. So if you were relying on X for knowing some other validation property has passed, you can move most of that work into the witness data space.

-------------------------

AntoineP | 2025-04-16 14:21:48 UTC | #12

[quote="JeremyRubin, post:11, topic:1591"]
use a legacy input B which has

```
scriptSig: [other program stuff] <sig || ALL|ACP > Dup <pk> checksig
script: <pk> checksig
```
[/quote]

For what it's worth i don't think you can reuse the signature like that. The message being signed by the `CHECKSIG` in the scriptSig is different from the message being signed by the `CHECKSIG` in the scriptPubKey. This is because [EvalScript is called](https://github.com/bitcoin/bitcoin/blob/cdc32994feadf3f15df3cfac5baae36b4b011462/src/script/interpreter.cpp#L1975-L1980) with the scriptSig for the former and with the scriptPubKey for the latter, which is later used as [the scriptCode in the `CHECKSIG` evaluation](https://github.com/bitcoin/bitcoin/blob/cdc32994feadf3f15df3cfac5baae36b4b011462/src/script/interpreter.cpp#L324-L338).

So, besides the fact that you are using a `CHECKSIG` in the scriptSig where i think you meant to use a `CHECKSIGVERIFY`, the script you propose would always either fail one of the two `CHECKSIG`s and therefore fail execution.

(Interestingly i think you might be able to not duplicate the signature by using some clever `CODESEPARATOR` hack so that the scriptCode is the same for both the scriptSig and the scriptPubKey.)

-------------------------

JeremyRubin | 2025-04-16 20:02:02 UTC | #13

yeah it does seem you'd need two independent signatures, unless @ajtowns had a trick in mind of codeseps.

-------------------------

JeremyRubin | 2025-04-21 20:23:13 UTC | #14

This example I created seems to deduplicate the signatures to work with OP_CODESEPARATOR


```python
#!/usr/bin/env python3
"""
Test‑bed for the script pair:

    scriptSig   = <sig> OP_DUP OP_CODESEPARATOR <pk> OP_CHECKSIG
    scriptPubKey= <pk> OP_CHECKSIG

The code builds a fake funding UTXO, spends it with the
non‑standard scriptSig above, and verifies consensus acceptance
(regtest only – policy rules would reject this on mainnet).
"""

from bitcointx.core import (
    COutPoint, CTxIn, CTxOut, CMutableTransaction, b2x
)
from bitcointx.core.script import (
    CScript, OP_DUP, OP_CODESEPARATOR, OP_CHECKSIGVERIFY, OP_TRUE
)
from bitcointx.core.key import CPubKey
from bitcointx.wallet import CBitcoinSecret
from bitcointx.core.scripteval import VerifyScript,MANDATORY_SCRIPT_VERIFY_FLAGS
from bitcointx.core.script import SignatureHash, SIGHASH_ALL


# --------------------------------------------------------------------------
# 1. Network & key‑pair
# --------------------------------------------------------------------------
seckey  = CBitcoinSecret.from_secret_bytes(b"\x01"*32)
pubkey  = CPubKey(seckey.pub)

# --------------------------------------------------------------------------
# 2. Fake funding tx : 1 BTC to "<pk> OP_CHECKSIG"
# --------------------------------------------------------------------------
funding_tx = CMutableTransaction([], [
    CTxOut(100_000_000, CScript([pubkey, OP_CHECKSIGVERIFY]))
])
funding_tx.GetHash()                                  # gives it a txid

# --------------------------------------------------------------------------
# 3. Spending tx skeleton
# --------------------------------------------------------------------------
vin  = CTxIn(COutPoint(funding_tx.GetTxid(), 0))
vout = CTxOut(90_000_000, CScript([OP_DUP]))         # arbitrary output
spend_tx = CMutableTransaction([vin], [vout])

# --------------------------------------------------------------------------
# 4. Signature – note the script *after* OP_CODESEPARATOR
# --------------------------------------------------------------------------
# For OP_CHECKSIG inside the scriptSig, only the bytes *after* the last
# OP_CODESEPARATOR – "[pubkey] OP_CHECKSIG" – are hashed.  That sequence
# is byte‑for‑byte identical to the funding output’s scriptPubKey, so a
# single SIGHASH_ALL signature works for both checks.

subscript = CScript([pubkey, OP_CHECKSIGVERIFY])           # bytes after CODESEP
sighash   = SignatureHash(subscript, spend_tx, 0, SIGHASH_ALL)

sig = seckey.sign(sighash) + bytes([SIGHASH_ALL])


print(len(sig), sig)
# --------------------------------------------------------------------------
# 5. Assemble full scriptSig
#    <sig> OP_DUP OP_CODESEPARATOR <pk> OP_CHECKSIG
# --------------------------------------------------------------------------
spend_tx.vin[0].scriptSig = CScript([
    OP_TRUE,
    sig,
    OP_DUP,
    OP_CODESEPARATOR,
    pubkey,
    OP_CHECKSIGVERIFY
])

spend_tx.GetHash()

# --------------------------------------------------------------------------
# 6. Consensus verification (policy flags kept minimal)
# --------------------------------------------------------------------------
VerifyScript(
    spend_tx.vin[0].scriptSig,
    CScript([pubkey, OP_CHECKSIGVERIFY]),
    spend_tx, 0,
    flags=MANDATORY_SCRIPT_VERIFY_FLAGS,
)

print("✓ script validated under consensus rules")
print("Funding txid :", b2x(funding_tx.GetTxid()))
print("Spending tx  :", b2x(spend_tx.serialize()))

```

output:

```
✓ script validated under consensus rules
Funding txid : a9114135fa063561c2f931374a411328225e4069f53bdbb57326ff919feef0d4
Spending tx  : 0200000001a9114135fa063561c2f931374a411328225e4069f53bdbb57326ff919feef0d4000000006e51473044022064c6b87eed1936f2c10ed05ef410f967aecea28acf1dbb58dcb5a643afb5c71f02202fbef2d81754f46a19583948d1c3fb38c25ea1ce5429affc4386c99ad2039e320176ab21031b84c5567b126440995d3ed5aaba0565d71e1834604819ff9c17f5e9d5dd078fadffffffff01804a5d0500000000017600000000
```


p.s., if you wanted the script to have a scriptSig that was a bit tighter, you could do something like this to restrict the scriptSig and scriptPubKey to have a single item on the stack that gets dup'd, and then checksigverify it. Less garbage data possible

```
POST_SEP = [
    OP_DEPTH,
    OP_1,
    OP_EQUALVERIFY,
    OP_DUP,
    pubkey,
    OP_CHECKSIGVERIFY]
```

-------------------------

JeremyRubin | 2025-04-20 21:20:33 UTC | #15

Overall this approach serves to allow A to be spent with a specific spend of B. However, B can still be spent elsewhere, and is still subject to third-party malleability (e.g. scriptSig NOP injection).

So while A can be sure it is spent with B or none-other, B could be caused to be spent spuriously as long as someone is willing to e.g. front the money to pay for whatever other outputs.

-------------------------

JeremyRubin | 2025-04-20 21:24:17 UTC | #16

Addenda:

this approach doesn't actually require OP_CODESEPARATOR at all it seems, as if you have the DUP contained in both scriptsig and scriptPubKey, following the signature, the FindAndDelete behavior will run more or less identically.

-------------------------

JeremyRubin | 2025-04-21 16:18:03 UTC | #17

Addenda:

to limit malleability, 197 OP_NOPs may be on the scriptSig following the signature.

The OP_DEPTH check prevents any other pushdatas. The NOPs also prevent any other codeseparator being placed.



```
POST_SEP =  [
    OP_DEPTH,
    OP_1,
    OP_EQUALVERIFY,
    OP_DUP,
    pubkey,
    OP_CHECKSIGVERIFY] + [OP_NOP]*197
```

B'd signature's scriptcode is comitted to. It also seems not possible to sneakily inject another copy of the signature itself somewhere, relying on FindAndDelete, since there is no way to drop it before the second stack size check (in the **scriptPubKey**) executes.

*edit: we also put the OP_NOP's at the end, so that the signature cannot be placed arbitrarily in the OP_NOP segment.*

~~I believe this prevents the issue that B could be caused to be spent spuriously as long as someone is willing to e.g. front the money to pay for whatever other outputs. It seems with this change, B can be sure that their scriptSig cannot be modified by a third party.~~

Edit: This prevents that *if* there is a checksig in the scriptSig, it must match this template. But it does not seem to require that the scriptSig must be a match to this template. E.g., you could drop all the other scriptSig logic entirely to just have the sig on the stack. This seems inherent.

-------------------------

niftynei | 2025-04-25 15:55:43 UTC | #19

“inputA is spendable only together with inputB”

Using signatures, a SIGHASH_GROUP* would trivially solve this. Current discussion seems to favor introspection vs sigs tho. 

*notably unimplemented/lacking a proposal so yes this isn't currently practicable.

-------------------------

instagibbs | 2025-04-28 13:51:17 UTC | #20

Capabilities offered in https://github.com/bitcoin/bips/pull/1500 which is a spiritual successor to GROUP in some ways would trivially do this, as you can selectively choose to commit to prevouts, or whatever you like.

-------------------------

JeremyRubin | 2025-04-28 15:47:16 UTC | #21

a sighash group actually does not trivially solve this, since a signature then re-entails the need for it to be constructed with interactive setup (Unless also using APO-semantics, or something else).

it's also not "trivial" in that A can commit to B, but B cannot commit to A, even with these other opcodes.

-------------------------

ajtowns | 2025-04-29 05:54:54 UTC | #22

[quote="JeremyRubin, post:21, topic:1591"]
it’s also not “trivial” in that A can commit to B, but B cannot commit to A, even with these other opcodes.
[/quote]

I think the TXHASH spec in bips [PR#1500](https://github.com/bitcoin/bips/pull/1500) can't directly do the same [bllsh trick mentioned above](https://delvingbitcoin.org/t/how-ctv-csfs-improves-bitvm-bridges/1591/8) because it only allows you to request the hash of a prevout, not the individual txid and out-index components that make that up. I think you could simulate it with CAT by requiring the user to provide the txid, the CAT'ing that with the expected vout, hashing, and comparing that with the TXHASH result, then doing the same for the other input and vout though.

-------------------------

RobinLinus | 2025-04-29 16:01:48 UTC | #23

[quote="RobinLinus, post:1, topic:1591"]
### The scriptSig Trick

The key idea is to use the fact that CTV commits to the scriptSig of all inputs. Say we want to express “inputA is spendable only together with inputB”.

1. Define inputB to be a (legacy) P2SH output.
2. Presign a signature using sighash `ANYONECANPAY|NONE`, effectively signing only inputB. This signature commits to inputB and since P2SH is not SegWit the signature will be in inputB’s scriptSig.
3. Define inputA to be a P2TR output and contain a CTV condition with a template hash that commits to the scriptSigs, including the signature for inputB.

The result: **inputA** commits to the signature for **inputB**, which itself commits to **inputB**. So **inputA** becomes spendable only in conjunction with **inputB**.
[/quote]


For completeness, we are encountering another issue with this construction:

* If `inputA` and `inputB` are created by the same transaction, we cannot apply our trick, as it would create a hash cycle.
* For the same reason, we cannot apply it to any `inputB'` that is a child of `inputB`.
* Similarly, we cannot apply it to any `inputA'` that is a child of `inputA` and connected via a chain of CTV hashes.

Furthermore, `inputB` should use sighash `ANYONECANPAY|ALL` to ensure it creates the correct output. However, it could still be spent "maliciously" without `inputA`.

-------------------------

JeremyRubin | 2025-04-29 18:10:29 UTC | #24

yeah that seems correct -- inputA and inputB must be separate txs, or you entail a hash cycle.

interestingly, for this case, you *can* do both sides with (a variant of txhash), by doing a constraint check that they both have the same txid and the correct vout...

e.g. 

input 0: GETTXIDOFINPUT(0) == GETTXIDOFINPUT(1) && GETCURRENTINDEX == 0
input 1: GETTXIDOFINPUT(0) == GETTXIDOFINPUT(1) && GETCURRENTINDEX == 1

where txid is known to have two outputs (or else do some other checks).

I'm not sure this is relevant, or even interesting, since there's marginal difference between two inputs that must be spent together from the same tx, and one input that contains the value for both(, or can be split in two after some delay, optionally).

-------------------------

Chris_Stewart_5 | 2025-06-25 19:39:15 UTC | #25

Hi everyone,

I took the liberty of implementing this idea [in the python test harness](https://github.com/Christewart/bitcoin/blob/c3431957a9d6dfcf68e00ceb3c5e02c3fdcdc6dc/test/functional/feature_bitvmctvcsfs_bridge.py) to familiarize myself with OP_CTV and make things a little more concerete in this thread. Hopefully this can help others that are learning about OP_CTV or be used to hack on other interesting OP_CTV projects. You will find direct links to the test cases embedded in the blog post

[quote="RobinLinus, post:1, topic:1591"]
The key idea is to use the fact that CTV commits to the scriptSig of all inputs. Say we want to express “inputA is spendable only together with inputB”.

1. Define inputB to be a (legacy) P2SH output.
2. Presign a signature using sighash `SINGLE|NONE`, effectively signing only inputB. This signature commits to inputB and since P2SH is not SegWit the signature will be in inputB’s scriptSig.
3. Define inputA to be a P2TR output and contain a CTV condition with a template hash that commits to the scriptSigs, including the signature for inputB.

The result: **inputA** commits to the signature for **inputB**, which itself commits to **inputB**. So **inputA** becomes spendable only in conjunction with **inputB**.
[/quote]

I first implemented the original idea by Robin Linus, you can find the test case [here](https://github.com/Christewart/bitcoin/blob/c3431957a9d6dfcf68e00ceb3c5e02c3fdcdc6dc/test/functional/feature_bitvmctvcsfs_bridge.py#L234)

[quote="ajtowns, post:8, topic:1591"]
I don’t think this works? As I understand it, with this setup you have something like:

```
utxo A:   10 BTC, p2tr: ... <H> CTV
utxo B:  500 sats, p2sh: <P> CHECKSIG
```

with the idea being that you spend utxo A via the CTV with:

```
input 1: utxo A, witness reveals CTV path and any other conditions, no scriptSig
input 2: utxo B, scriptSig = "<S;NONE|ANYONECANPAY>" "<P> CHECKSIG"
output: whatever
```

where `<H>` locks in where the output goes to, and input 2’s scriptSig. But because H is only committing to the second input’s scriptSig, then it’s easy to construct another utxo that can be used instead, eg one with the (non-standard) scriptPubKey `OP_2DROP OP_TRUE`:

```
utxo C:  500 sats, scriptPubKey: OP_2DROP OP_TRUE

input 1: utxo A, no witness/scriptSig
input 2: utxo C, scriptSig = "<S;NONE|ANYONECANPAY>" "<P> CHECKSIG"
output: whatever
```

That allows utxo A to be spent via the CTV path independently of whether utxo B has already been spent/burnt, which, as far as I can see, breaks the protocol you’re trying to enforce.
[/quote]

I implemented this logic in a test case [here](https://github.com/Christewart/bitcoin/blob/c3431957a9d6dfcf68e00ceb3c5e02c3fdcdc6dc/test/functional/feature_bitvmctvcsfs_bridge.py#L249). It seems AJ is correct about the limitations of Robin's original scheme.

[quote="instagibbs, post:9, topic:1591, full:true"]
The scriptSig could include the `CHECKSIG` opcode directly, contra standardness rules. :grimacing:

In the original idea, a p2sh redeemscript is just pushes, so the spk could just be blank and it would pass, since cleanstack isn’t consensus anyways.
[/quote]

Finally, I took a [crack at implementing this](https://github.com/Christewart/bitcoin/blob/c3431957a9d6dfcf68e00ceb3c5e02c3fdcdc6dc/test/functional/feature_bitvmctvcsfs_bridge.py#L288). This is a little more tricky than it appears. I believe this would theoretically work, but requires a lot of hacking around producing valid signatures for the scriptSig OP_CHECKSIG operation.

It is my understanding that the scriptcodes are different for the digital signature to satisfy the redeem script and the OP_CHECKSIG embedded in the scriptSignature. 

Here is where I believe they are defined
1. [Input scriptcode (i.e. the script signature including the redeem script?)](https://github.com/bitcoin/bitcoin/blob/ad654a4807cd584be9ffcd8640f628ab40cb5170/src/script/interpreter.cpp#L1975)
2. [The output scriptcode (i.e. the p2sh script)](https://github.com/bitcoin/bitcoin/blob/ad654a4807cd584be9ffcd8640f628ab40cb5170/src/script/interpreter.cpp#L1980).

I attempted to produce the 2 digital signatures. The p2sh output script was no problem of course as its a standardized script type. [Here](https://github.com/Christewart/bitcoin/blob/c3431957a9d6dfcf68e00ceb3c5e02c3fdcdc6dc/test/functional/feature_bitvmctvcsfs_bridge.py#L144) is where I attempted to produce the scriptSignature's OP_CHECKSIG operation. Unfortunately, the [`signrawtransactionwithkey`](https://github.com/bitcoin/bitcoin/blob/ad654a4807cd584be9ffcd8640f628ab40cb5170/src/rpc/rawtransaction.cpp#L710) RPC will not just give you the digital signature back that it produced (this would have saved me a lot of time :joy:). It attempts to place the digital signature in the transaction which only works for standardized transactions that the solver is aware of :/. 

Since this is a very nonstandard transaction, it doesn't give me back anything useful at all - just an the same transaction that you passed to the RPC. I do think this workaround will theoretically work, but the practical barriers in this code base are just too high to continue at this time.

Let me know if you see anything wrong with this analysis or have other use cases you would like to see prototyped with OP_{CTV,CSFS}

-------------------------

