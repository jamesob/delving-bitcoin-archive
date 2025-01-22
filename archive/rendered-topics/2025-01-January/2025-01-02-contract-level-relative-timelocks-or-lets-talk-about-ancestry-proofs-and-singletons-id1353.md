# Contract-level Relative Timelocks (or, let's talk about ancestry proofs and singletons)

instagibbs | 2025-01-06 13:48:48 UTC | #1

Contract-level Relative Timelock (CLRT) UTXO for Eltoo
===

Eltoo constructs such as ln-symmetry suffer from an issue
where every time an update transaction is confirmed on the
blockchain, the relative timelock to settle the contract is reset. This causes
further funds lockup, and further extends HTLC expiry in the LN
use-case, potentially reducing network utility.

How can we have a "contract" level relative timelock? Embedding state inside the continuously refreshed utxos seems intractable due to Bitcoin's requirement for monotonic validity of transactions. An adversary can simply "under-report" the number of "elapsed" blocks during the contract's lifetime.
The proposed solution here is to dedicate a specific utxo that doesn't move until
the the challenge period is over, and only allowing the contract state output and relative timelock output to be spent concurrently.

LN-symmetry and extension?
===

To recap, ln-symmetry has this series of transactions that must appear to settle the contract:

`funding->update->settle`

For this extension we change it to include a slightly
different update transaction we will call "kickoff":

`funding->kickoff->update->settle`

Kickoff transactions have an additional CLRT output that
commits to a relative delay (ln-symmetry's `shared_delay`) for the eltoo challenge period
before the settlement transaction can be confirmed.

The output is dust-level and commits to being spent concurrently with an
eltoo state output. To do this, you need a recursive proof that links
back to an update transcation's state output. In other words, the recursive
proof needs to demonstrate that another input has an ancestry that includes
the kickoff transaction itself.

Update transactions pre-commit to both the state output(s) and the CLRT
output to enforce that if the state output is spent, so is the CLRT output.

This makes mutual spending of a state output and CLRT a requirement for settlement.

How could the CLRT ancestry proof work?
===
Make the problem simpler: Assume TXID stability
===
If we accept the case where we are just doing "onchain" eltoo with normal SIGHASH_DEFAULT,
this becomes a lot simpler due to txid stability of the eltoo chain. The CLRT output becomes
a connector output that is "re-attached" via consensus signatures of channel participants, and the script being a simple CSV of the `shared_delay`.

Obviously, requiring O(n) state on-chain is sub-optimal, but I think it's important to have a correct construction for demonstration purposes.

Another usage, at least on paper: [Used for sequencing transactions in John Law's constructions of channels](https://delvingbitcoin.org/t/contract-level-relative-timelocks/1353)

Without TXID stability
===
Once we venture into "real" eltoo where we are re-attaching prevouts,
transaction id stability goes out the window and we cannot rely on
regular signatures to authorize the connector output.

High-level handwave solution:
1) CLRT introspects "current" input's prevout. This will be matched in ancestry proof next.
2) Proof contains first submitted update transaction with matching kickoff prevout, computes
its txid, checks that second confirmed update transaction in proof spends that utxo, etc, until
proof connects with the settlement transaction itself in the state output being spent.
The proof checker MUST enforce the right outputs are being spent at each step: the state output
of the eltoo contract, rather than say an anchor output.

Problems:
1) Requires some sort of consensus change to support proof construction, leaving this as exercise to reader.
2) Kickoff transaction and additional utxo costs extra vbytes and adds lots of complexity.
3) Proofs are vbytes-expensive and allow counterparty to penalize honest partner by doing additional updates.
4) Crucially, the counter-party can make it consensus-invalid to actually spend the CLRT by inflating the proof
beyond consensus limits. Either need a mechanism for fixed number of updates and constant sized updates, or ZK magic to compress the proof to constant size?

If we get OP_ZKP maybe it becomes practical with O(1) enforcement of transaction ancestry?

Are there simpler solutions to this problem I'm missing?

Appendix: Chia Version
===
h/t @ajtowns

With Chia's coinid, I think this gets pretty simple, but is still linear in update history:

reminder that `CoinID = SHA256(parent_coin_id || puzzle_hash || amount)`

Assuming the signature being used is functionally equivalent to `SIGHASH_ALL` and there is some sort of simplistic P2A like output for fees, that means there will be two types of outputs in each update: a contract output, and a static P2A-like puzzle we can filter for.

For witness data you are given a series of 32WU `puzzle_hash`es, none of which can be the P2A-equivalent. You take CLRT's parent coin id, and repeatedly hash it with the static amount and series of puzzle hashes. At the end, the coinids should match expected.

At 32WU per update, a standard relayable transaction could include a proof up to 12,500 levels, minus overhead.

-------------------------

instagibbs | 2025-01-02 19:30:47 UTC | #2

Turns out John Law employs two relative timelocks using separate utxos, with TXID stability for one of his later [payment channel constructions](https://github.com/JohnLaw2/ln-hierarchical-channels)

![image|690x306](upload://1JrKe9bX2FohPFnfWpgYGDhplTE.jpeg)

Figure 14 and 15 show the separate locks. I'm unclear as to the purpose of the tsdAB lock, but it in effect allows for two "lanes" of timeouts, one for revocation of commitment transaction  purposes, and one for HTLCs (revocation?). Once both relative timelocks are satisfied, the HTLC-payment transactions can be processed and neither can "reset" the either. The latter output is also a dust output, purely for control purposes.

This seems like a good "simple" example with txid stability.

Thanks for the tip @ajtowns

-------------------------

JeremyRubin | 2025-01-03 17:32:57 UTC | #3

MUON has an interesting interplay here... https://x.com/JeremyRubin/status/1782220444185116883

_Note that you still end up needing some sort of recursive proof inside R here. But MUON does the job of ensuring that no Tx Update can be broadcast without the right R being created for it, and makes Tx Update non-malleable._


![20250103_115444|481x500](upload://auLGMqnpFyQgW9kZLNUfygy2Ygs.jpeg)


Describing the above graph:


Tx Open:
- Inputs: ...
- Outputs: V_O <- N Sats

V_O.program <- tr(musig(A, B), {CTVHASH(kickoff) CTV})

Tx Kickoff:
- Inputs: V_O
- Outputs:
  - R[0] <- 0 sats / dust
  - V_K <- N Sats

V_K.program = tr(musig(A, B), {})

Updates are setup as follows:

Tx Update[i]:
- Sequence: 2 weeks
- Inputs V_K
- Outputs
    - MUON X_i <- 0 Sats
    - Alice <- k*N
    - Bob <- (1-k) *N



Tx Ratchet i:
 - nLockTime i
 - Input R[i]
 - Output
   - R[i+1]

R[i]'s program:

tr(NUMS_H, {
ratchet,
cospend
})

ratchet: <CTVHASH(Ratchet TX i+1)> CTV <musig(a,b)> CSFS [i] CLTV

cospend: 1 GETINPUT <COutpoint(MUON X_i)> EQUALVERIFY <CTVHash(Tx Exit)> CTV


Tx Exit:
  - nSequence: 1 day (min time between last update?)
  - Inputs: R[i] (via cospend path), MUON X_i
  - Outputs: OP_RETURN Update[i].details_to_reconstruct

muon X_i.program: tr(NUMS_H, {<CTVHash(Tx Exit)> CTV 0 GETINPUT  <R[i]> EQUALVERIFY })






------------


How signing works:


First you open the protocol to create V_O.

Then you create the updates off of V_K (go ahead and sign -- MUON X means a spend must exit).

You then create the ratchet update off of R, and exchange the sigs.


_Note that you still end up needing some sort of recursive proof inside R here. But MUON does the job of ensuring that no Tx Update can be broadcast without the right R being created for it, and makes Tx Update non-malleable._

-------------------------

cguida | 2025-01-03 22:02:29 UTC | #4

Why is the current situation untenable, where each party submits what it considers to be the newest update? What is the precise vulnerability the CLRT is trying to mitigate?

The only time I can see the delay taking longer than 2x the delay is if both parties publish a state they know is not the newest state, which would be kind of silly, wouldn't it?

What am I missing?

-------------------------

moonsettler | 2025-01-04 19:45:53 UTC | #5

Maybe I misunderstand the problem, but assume Mallory submits an old state, Alice submits the newest state in the mempool, but Mallory can outbid hers with any of the previous states. If you use a TXID dependent form of fee paying mechanism Alice has to re-sign each and every time this happens. Her rebindable signatures are still good ofc, just the transactions are ejected from the mempool.

CLRT does not really mitigate this afaik, in fact makes it more dangerous.

-------------------------

instagibbs | 2025-01-04 20:43:29 UTC | #6

[quote="cguida, post:4, topic:1353"]
What is the precise vulnerability the CLRT is trying to mitigate?
[/quote]

CLRT is mitigating the amount of time liquidity is locked up in protocols.

[quote="cguida, post:4, topic:1353"]
The only time I can see the delay taking longer than 2x the delay is if both parties publish a state they know is not the newest state, which would be kind of silly, wouldn’t it?
[/quote]

The 2 times `shared_delay` is when, f.e., ln-symmetry already has a `shared_delay` for the channel's settlement transaction via `nsequence`. Alice adds an incoming HTLC to an otherwise quiet channel at channel update `T-1`. Alice sends a signature for the newest state `T` to Bob. Bob simply doesn't respond with his signature. Alice needs to claim the incoming HTLC, so she goes to chain with her latest state, `T-1`. `shared_delay-1` blocks pass, and Bob then gets `T` mined. Another `shared_delay` must pass.

Both of these `shared_delay`s must be built into the HTLC expiry delta for safety.

CLRT output would drop this to a single `shared_delay` for ln-symmetry.

[quote="moonsettler, post:5, topic:1353"]
CLRT does not really mitigate this afaik, in fact makes it more dangerous.
[/quote]

It means you have to pick your `shared_delay` as something you deem secure, as its a liveliness security parameter. With or without CLRT if an adversary outbids you `shared_delay` times, your funds can be at risk for HTLC theft.

Also, it does not preclude someone developing a modification to eltoo that bounds the number of updates. This is meant to be a modular piece usable in any context desired. It's not practical yet anyways, but this is why I opened the thread!

-------------------------

ajtowns | 2025-01-05 02:52:14 UTC | #7

[quote="instagibbs, post:1, topic:1353"]
With Chia’s coinid, I think this gets pretty simple, but is still linear in update history:
[/quote]

The Chia coinid also allows a mind-bending approach whose proof size is independent of update history (which is used in their NFT schemes), called a [singleton](https://chialisp.com/singletons/).

I believe the way it works is you identify a coin X as holding the singleton S provided by saying:

   * parent_coin_id = A (extract from coin's coin id; ~40B)
   * parent's puzzle = B (extract from parent_coin_id; ~40B)
   * B = singleton_puzzle(singleton_id for S)  (same puzzle as self, so ~0B) 

Because the parent coin is already mined, you know the singleton_puzzle was satisfied, and the only way the singleton puzzle was satisfied is either recursively, or if the grantparent coin launched the singleton.

(I might have some details wrong there, but I'm pretty sure the gist is right)

-------------------------

salvatoshi | 2025-01-05 09:34:34 UTC | #8

Very interesting to see this concept distilled so cleanly!

@rijndael sketched what seems to me exactly the same idea in this thread describing a possible [token standard using CAT called CatNip](https://x.com/rot13maxi/status/1833667750469804315). Once you remove the 'token' parts, I think the singleton is precisely what's left - and indeed, constant-size proof of ancestry was the goal.

There, he used CAT in (at least) two ways:
- carrying state via extra outputs (using introspection with the Schnorr trick)
- introspecting the parent's Script based on its txid.

An opcode that allows a clean way of carrying state (like OP_CHECKCONTRACTERIFY) would avoid the Schnorr trick, but of course reading 'inside' a txid is trickier (perhaps not crazily so, though). There might be other ways of creating the singleton that do not require the txid.

The singleton primitive seems very interesting to me, as it would allow to generalize the concept of the 'connector outputs' (which are essentially a proof of ancestry at depth zero) used for example in Ark. I expect this will be very useful in trying to combine state-carrying UTXOs with Ark and other shared UTXO constructions.

-------------------------

ajtowns | 2025-01-06 06:55:17 UTC | #9

[quote="salvatoshi, post:8, topic:1353"]
reading ‘inside’ a txid is trickier (perhaps not crazily so, though).
[/quote]

Reading inside a txid is a lot more annoying, since you have to provide the entire (witness-stipped) tx, and correctly parse it in order to get the specific bit of data you want (either the prevout txid, or an output). Making the singleton be a 1-in/2-out tx where you use the first output holds the singleton, and the second output is a pay-to-anchor address whose spender can use the singleton might work though.

-------------------------

instagibbs | 2025-01-06 13:48:25 UTC | #10

[quote="salvatoshi, post:8, topic:1353"]
Once you remove the ‘token’ parts, I think the singleton is precisely what’s left - and indeed, constant-size proof of ancestry was the goal.
[/quote]

That's interesting! To make sure I'm understanding correctly: We essentially want an NFT (singleton) for the "contract" identity, because that allows us to make the ancestry proof recursive bounded to one introspection step.

f.e. in ln-symmetry this would mean update transactions would the same require introspection logic, in addition to the settlement transaction. Each update knows about the kickoff txid and makes assertions on the update step. You're adding additional overhead per *submitted* update, paid for by the *submitter*, rather than paid for by the honest participant at settlement time.

This seems to make it safe for relay/consensus as well, since you have a clear bound on the size of the witness-stripped update transactions.

Without a whole lot of thought that sounds right.

[quote="ajtowns, post:9, topic:1353"]
Making the singleton be a 1-in/2-out tx where you use the first output holds the singleton, and the second output is a pay-to-anchor address whose spender can use the singleton might work though.
[/quote]

I think this is the most sensible construction yes.

-------------------------

rijndael | 2025-01-06 14:40:49 UTC | #11

to share some learnings in this area: 

the schnorr trick with CAT lets you get an inputs scriptpubkey, prevout, and index onto the stack. You can also get the outputs onto the stack. if you enforce that "this" input's scriptpubkey matched an output scriptpubkey, then you can enforce that the spend is always happening back to the same taproot address. You can use this to build state machines where different states for your contract are different tapleafs that enforce the validity of a state transition (if you want all the gory details on this technique including the schnorr math, I gave a talk at Bitcoin++ that walks through it: https://t.co/tQJQoWepcK)

A technique I had played with in an [early vault prototype was to pass state from TX_n to TX_n+1](https://github.com/taproot-wizards/purrfect_vault?tab=readme-ov-file#complete-withdrawal) by:

- putting a little state commitment in an output of TX_n
- in TX_n+1, reconstruct the (as @ajtowns pointed out above) witness-stripped transaction on the stack, asserting the state commitment in the output from TX_n (either an OP_RETURN or a normal output if you're committing to a SPK)
- HASH256 the transaction on the stack to get the TXID, assert that it matches the prevout of the input
- you now have previously-committed state on the stack! 

the popular name for this technique now is the "state caboose" (you pull it along). there are lots of cool things you can do with that (like committing to a withdrawal destination in a vault) but as @salvatoshi pointed out, one really cool thing is you can do a constant-sized inductive proof of contract validity. Here's how it works:

Suppose you have a contract that is instantiated in TX FOO. let the TXID FOO be the contact instance ID. To spend a UTXO encumbered by FOO along, you have some set of validity rules for each step (for example, signature checks, amount validation logic, timelocks, whatever). Additionally, there is a "contract history" check that you do. There are two cases you have to check for contract history (implemented as different tapleafs):

- the parent TXID is FOO
- the parent transaction spent from the contract scriptpubkey to the contract scriptpubkey

if you have those two checks, then you always know that a UTXO in the contract is valid, because it came from either a valid state, or from the instance genesis. In the inductive case (coming from a valid state), you need to pass in a constant-ish amount of witness data (you can have different numbers of inputs and outputs, but its a bounded amount, so you in practice you write a script for the worst case), which means that you can actually use this in Script. The spender may need to do some extra work and state management, but the amount of data that hits the blockchain is bounded to a relatively small size.

An extension on this that we've done some experimenting with is that you can delegate your contract validity rules to another script. Here's the short version:

- in tx_n you make a commitment that says "in the next transaction, I want my UTXO to obey the rules of this other contract (scriptpubkey)". This commitment is made in the state caboose described earlier
- in tx_n+1, you do your normal contract history check (the inductive bit above) and then you check if the input at index 0 (or whatever) is the scriptpubkey of the contract you delegated to. if not, fail
- now your contract instance UTXO has to follow the rules of that other contract! this is super useful as either an upgrade/extension mechanism

we've been experimenting with this stuff assuming only OP_CAT, but other opcodes make things a lot cleaner or open up other interesting avenues. for example, CCV or some TAPTWEAK makes state carrying cleaner and removes some technical limitations on the size of transactions in these contracts. CSFS lets us more easily do authenticated delegation (you can delegate to an approved contract) which you can do with just CAT using some funky signing over the state caboose but its a lot easier to do with CSFS. more-specialized introspection opcodes would make the whole thing easier to reason about and would have smaller scripts.

-------------------------

reardencode | 2025-01-06 15:18:45 UTC | #12

I believe @ademan 's multi-party penalty(optional) rebindable channels also have a solution for this. Because his proposal limits the number of updates to the number of channel participants, each update is identifiable by its order, which means that the settlement delay can be reduced with each update.

Details fuzzy, but I think settlement can be made immediate on the last allowed update in an @ademan channel. This means that for 2-party @ademan channels HTLCs only need to tolerate 1 settlement delay.

-------------------------

cguida | 2025-01-13 19:35:32 UTC | #13

@moonsettler hmm whoa, I hadn't thought of that scenario...

@instagibbs is the scenario Moon mentioned applicable here?

-------------------------

instagibbs | 2025-01-13 19:38:04 UTC | #14

https://delvingbitcoin.org/t/contract-level-relative-timelocks-or-lets-talk-about-ancestry-proofs-and-singletons/1353/6?u=instagibbs

already replied

-------------------------

cguida | 2025-01-13 19:42:38 UTC | #15

@instagibbs Yes, I understand that `shared_delay` * 2 is the longest amount of time an HTLC could take to settle, so that must be built into the HTLC expiry.

This doesn't seem like a huge issue, unless there are ways for a malicious channel partner to continue broadcasting new update txs after the 2x delta has expired, right? We just need to make Symmetry HTLCs take longer to expire than Penalty ones, right? That is to say, there's no "attack" that CLRT is trying to mitigate, and CLRT is merely about making sure that capital on LN is used efficiently?

-------------------------

cguida | 2025-01-13 19:43:52 UTC | #16

@instagibbs you did not mention whether or not Moon's scenario is applicable. You just described an entirely different scenario.

-------------------------

ademan | 2025-01-14 14:38:30 UTC | #17

[quote="reardencode, post:12, topic:1353"]
I think settlement can be made immediate on the last allowed update in an @ademan channel
[/quote]

That's right. For simplicity the last update follows the same format as the previous ones right now, but I think you could even "inline" the settlement transaction into the last update. For N=2 this looks a lot like Daric actually (probably not coincidentally).

I think this strategy gives you a `(N - 1) * shared_delay` at best, since every party still needs a chance to update in the worst case.

-------------------------

