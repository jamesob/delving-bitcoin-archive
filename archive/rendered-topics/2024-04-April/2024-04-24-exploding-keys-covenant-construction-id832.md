# Exploding Keys - Covenant construction

adiabat | 2024-04-24 03:12:24 UTC | #1

Exploding Keys

TLDR: Public keys that commit to a set of outputs.  If the key is spent to those outputs, no signature or witness is needed.

Motivation: Exploding keys gets you basically the same functionality as `op_ctv` but done in a somewhat clever way that saves a few bytes.  I came up with this a while ago but kept thinking there was a way to make this better / more powerful / really compelling.  Maybe that's not going to happen, and I want to write up all the things that would be really cool but don't work.  On it's own it's kind of fun though and maybe will give people some ideas, or could be a tool for bitcoin covenants.

For the example transactions, assume that public keys are used like in taproot outputs, rather than public key hashes.

The example tx has 1 input and 3 outputs.

```

Input 0:
txid:idx (points to prev PkScript KeyQ, 9 BTC)
no sigscript
no witness

Output 0:
KeyA, 2 BTC

Output 1:
KeyB, 3 BTC

Output 2:
KeyC, 4 BTC
```

Normally you'd look at this and say, well there's no witness, so no deal, fail the tx.  But with the exploding keys soft fork (guess you'd need to use a new segwit version or some other tag to keep the fork soft) you perform an additional check if the tx has no witnesses:

For each output, use the presented key as an "inner key" and commit in the amount of BTC taproot style (BIP 341).  Store the keys for the next step.

```
KeyA' = KeyA + hash(2, KeyA)*G
KeyB' = KeyB + hash(3, KeyB)*G
KeyC' = KeyC + hash(4, KeyC)*G
```
Next, take the key aggregation function in BIP 327, KeyAgg(), and feed it all the tweaked keys.  (I think you don't want to run KeySort() first and keep it in the order seen in the tx).
```
KeyW = KeyAgg(KeyA', KeyB', KeyC')
```
Then check if `KeyW == KeyQ` seen in the input.  If they're equal then the input passes the validation checks despite having an empty witness.  (You still need to check that the amounts work - in this example they do with 0 fee)

I think this works in that if you're given a `KeyQ`, there's no way to come up with a set of keys that aggregates to it other than the one initially used to construct `KeyQ` itself.  If there were, that would be a problem / collision for BIP 327 aggregation.  Similarly, given a tweaked key like `KeyA'`, there should be no way to come up with a different inner key and tweak value, because that would be a problem for BIP 341 tweaking.

To construct an exploding key, you go through the same sequence as verification: start with the keys and amounts you want to explode to, then tweak, then aggregate the tweaked keys.  The resulting key is an explodable key which you can send coins to.  What's nice is that private key holders A, B, and C can still interact to sign with the explodable key to send it somewhere else.  If they can't interact though, any party with the knowledge of the output set (which is not just A, B, and C, after all there is no signature needed) can explode the key into the 3 components.  This could be useful for a shared UTXO where multiple parties all claim a portion of the total coins - they can cooperate to sign or press the explode button to leave.

Just like `op_ctv`, this works recursively -- `KeyA` can itself be an exploding key.

So that's it.  It gives the basic functionality of `op_ctv`, while saving a few bytes of witness data.  On it's own it's maybe not all that compelling, but I wanted to put it out there as maybe it could be a useful primitive as part of a more complex covenant construction.

Don't have time today but I do want to write a bit in a day or two about some things that would be really cool if exploding keys could do them, but they can't and maybe nothing can :)

Thanks to sipa & real-or-random who I discussed this with a while ago and helped with feedback.  (Also I may have forgotten their feedback, rendering this construction insecure, in which case, that's on me not them :) )

-------------------------

ajtowns | 2024-04-30 02:28:23 UTC | #2

[quote="adiabat, post:1, topic:832"]
What’s nice is that private key holders A, B, and C can still interact to sign with the explodable key to send it somewhere else.
[/quote]

Without being able to enforce a timeout on the exploding path, I don't think having a key path is very interesting? In adversarial scenarios you can't use it, because your adversary can just broadcast the exploding version; in non-adversarial scenarios you wait until feerates are load and use the key path to spend directly to wherever you want and don't need an exploding tx in the middle?

I think you could extend this design to fix that, though, by making the commitment something like:

```raw
keyZ = H(nLockTime, nSequence, annex)*G
keyW = KeyAgg(keyA', keyB', keyC', keyZ)
```

with `nLockTime`, `nSequence` and `annex` pulled from the exploding spend. At that point, A,B,C have until the locktime/nseq expires to agree on and broadcast a key path spend, or after that time, they all just get their money back via the exploding tx. That would allow a one-shot payment pool, I think?

I don't see a way to tweak the maths to allow you to have both an exploding spend path and a script spend path -- I think in that case the exploding path would still need to reveal the script (hash), and at that point you might as well just use a script path in first place?

-------------------------

ZmnSCPxj | 2024-05-02 15:14:10 UTC | #3

Ah, I think I remember you mentioning this as similar to my `OP_EVICT` proposal before.

A way to allow both for a 'splody-path spend and a tapleaf path spend would be to require `MuSig(outputs)` to be the internal pubkey of the spent input, rather than the direct output.  Then you only reveal the topmost `TapRoot` hash without revealing any scripts in the 'splody-path spend.  You can even have an option of:

* Only 'splody-path spend: the actual pubkey is the MuSig(outputs), on 'splody-path spend, you have an empty witness.
  * 'splody-path spend: empty witness
  * consensual everyone-signs: a witness, 64 bytes signature
* 'splody-path spend OR taproot spend: the actual pubkey is `P + hash(P | tagged_hash("TapRoot", whatever))` and the internal pubkey `P` is the `MuSig(output)`
  * 'splody-path spend: a witness, 32 bytes tagged hash output
  * consensual everyone-signs: a witness, 64 bytes signature
  * tapleaf: witnesses: control block with path to tapleaf, tapleaf script, tapleaf script witness stack

-------------------------

