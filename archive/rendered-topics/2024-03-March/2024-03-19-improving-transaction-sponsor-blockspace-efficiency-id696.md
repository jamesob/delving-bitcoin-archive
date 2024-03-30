# Improving transaction sponsor blockspace efficiency

harding | 2024-03-19 20:19:34 UTC | #1

Jeremy Rubin previously proposed [transaction fee sponsors][sponsors], a
soft fork that would allow a _sponsor transaction_ to only be valid if
it was included later in the same block as another transaction it explicitly
references.  Those explicit references are placed in the sponsor
transaction's output script.  For example, to sponsor txid
`0x1234...90ab`, the sponsor transaction would have a 33-byte output
script consisting of:

    0x62 0x01234567890abcdef01234567890abcdef01234567890abcdef01234567890ab

During a casual conversation a few months ago, Jeremy and I realized
that this could be done more efficiently.  The simplest and most
efficient form would only support sponsoring a single other transaction:

1. An input in the sponsor transaction has a signature message
   (signature hash) that commits to the txid to be sponsored.  This
   field is signed, preventing third parties from changing it, but it is
   not explicitly included in the transaction, saving vbytes.

2. The sponsor transaction must appear in a block immediately after the
   transaction it sponsors.  This allows the verifier of the sponsor
   transaction to infer the txid of the transaction it sponsors,
   allowing it to construct the signature message.

Depending on exactly how the above was added to Bitcoin, the overhead
for indicating the txid of the transaction to sponsor would be between
zero vbytes and a few vbytes, a reduction from 42 vbytes in Jeremy's original
proposal (8-vbyte output amount, size byte, 33-vbyte output
script).  It's also a reduction in space and overhead compared to
[ephemeral anchors][], which requires (1) a relationship between the
transactions, (2) the parent transaction to have an output of about 12
vbytes, (3) the child transaction to have an input of about 41 vbytes.

Of course, the sponsor transaction itself still takes up space, but if
someone was going to send a transaction anyway, then this allows that
transaction to sponsor another unrelated transaction with zero (or near
zero) block space overhead.

Jeremy's original proposal only had mempool policy consider a sponsor
transaction that sponsored a single other transaction.  He also only
allowed a transaction to be sponsored by a single sponsor transaction.
However, at the consensus level he proposed allowing a sponsor
transaction to sponsor multiple other transactions (all of which had to
be included in the same block for the sponsor to be valid) and for
multiple sponsor transactions to all sponsor one or more of the same
unrelated transactions.  Sponsors transactions could also sponsor other
sponsor transactions.  Additionally, sponsor transactions and the
transactions they sponsor are still subject to the regular consensus
transaction ordering constraint requiring ancestors appear before
descendants.

That means supporting an arbitrary number of sponsor transactions and
the transactions they sponsor is impractical with fixed orderings like
described above.  Instead, sponsor transactions would need to contain a
rewritable vector indicating where in a block to find the transactions
they sponsor.  For example:

1. Similar to before, an input in the sponsor transaction has a
   signature message that commits to a list of txids.

2. A malleable item on that input's witness stack indicates (1) the
   number of sponsored transactions and (2) the offset location of each
   sponsored transaction within a block.  For example, if the commitment
   message sponsored three transactions with txids
   {txid0,txid1,txid2}, the malleable witness item could be written to
   use `0x0002 0x0003 0x0000 0x0001` to indicate that there are three transactions (`0x0002`),
   that txid0 is located four transactions before the sponsor transaction (`0x0003`),
   that txid1 is located immediately before the sponsor transaction(`0x0000`), and
   that txid2 is located two transactions before the sponsor transaction (`0x0001`).
   Using fixed two-byte values is a simple way to support sponsoring up
   to 65k transactions at any position within a block (65k is about 4x
   the greatest number of tranactions possible in a single block without
   a hard fork).  Switching to fixed single-byte values halves the
   overhead and still allows a single sponsor transaction input to
   sponsor up to 256 unrelated transactions in any of the 256 preceding slots.  There may be more
   efficient encodings.

   For relayed transactions that are not yet included within a block,
   the malleable witness item could instead contain a list of 32-byte
   txids instead of two-byte offsets.  These would be extracted before
   the witness was evaluated to avoid them being misparsed or violating
   stack size limits.  Because the txids are conveyed directly here, the
   sponsor transaction's commitment hash can be validated without extra external lookups,
   allowing a mutated sponsor transaction to be dropped before we check
   the mempool for the transactions it sponsors.

In this variation with fixed two-byte offsets in witness data, the
amount of data that goes onchain is about 0.5 vbytes per sponsored
transaction.  That remains much more efficient than ephemeral anchors.

If someone is planning on creating a transaction anyway, they can
sponsor multiple other transactions with minimal overhead.  Even to
sponsor a minimal-sized 62-vbyte transaction, the sponsorship overhead
is significantly smaller than the amount that would be overpaid on
average using an exponential presigned RBF fee bumping strategy with 10%
increments.  E.g., 10% RBF increments implies overpaying on average by
about 5%, which is equivalent to paying for about 3 extra vbytes on
average on a 62-vbyte transaction.

Due to the high efficiency, I don't see any way for someone to
trustlessly pay a third party for sponsorship without prior setup and
the creation of a shared UTXO.  However, for anyone willing to depend on
trust and reputation, it's possible for any high frequency spenders to
offer a sponsorship service.  For example, Alice sends a transaction
every block; Bob wants txid_x to be sponsored; he trusts Alice and sends her an payment
over LN; she claims the payment, keeps part of it as profit, and uses
the rest to increase the feerate of her next transaction while adding a
sponsor dependency on txid_x.  Alice must check that sponsoring txid_x
won't slow down her transaction too much, she must be willing to
rewrite her transaction if a conflict of txid_x gets confirmed, and she
must also be aware of any mempool policies that could affect her, but
otherwise this seems a straightforward service to provide, so there
could develop a robust and reliable market for it.

[sponsors]: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-September/018168.html
[topic sponsors]: https://bitcoinops.org/en/topics/fee-sponsorship/
[ephemeral anchors]: https://bitcoinops.org/en/topics/ephemeral-anchors/

-------------------------

reardencode | 2024-03-19 19:22:22 UTC | #2

As I [posted](https://twitter.com/reardencode/status/1770094336434430157) on X, the biggest complication with sponsors is that the sponsor transaction can become invalid depending on innocent block reorderings, which would require it to be treated as a coinbase output and mature before it can be used in a normal transaction. I think this prevents sponsorship from being done as a part of regular transaction operations.

That said, sponsors (or other transactions that are provably re-bindable to either a current txo or one of its ancestors) could be allowed to spend before maturation, which means that a dedicated sponsor TXO could be spent repeatedly without maturing between spends, and only a subsequent spend that would not be re-bindable would have to wait for maturity.

-------------------------

harding | 2024-03-19 21:13:07 UTC | #3

[quote="reardencode, post:2, topic:696"]
As I [posted](https://twitter.com/reardencode/status/1770094336434430157) on X, the biggest complication with sponsors is that the sponsor transaction can become invalid depending on innocent block reorderings, which would require it to be treated as a coinbase output and mature before it can be used in a normal transaction.
[/quote]

Right, that was a [concern](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-September/018195.html) of @sdaftuar's in the original thread.  I agree that reorg safety has been a useful property these past 15 years, but I wonder if it will continue to be as useful in the future:

- Several of the soft fork proposals people are currently advocating for will allow scripts to verify SPV proofs, at which point it'll be possible for scripts to implement the functionality of transaction sponsorship, `OP_EXPIRE`, and several other features that are not reorg safe.
- No one can safely rely upon an input in a contract protocol unless it's been highly confirmed or comes from a chain of transactions they controlled that is rooted entirely in one or more highly confirmed transactions.  Reorg safety is a convenience feature; contract protocols must not rely on it for security.
- Frequent use of RBF fee bumping on the network can also lead to completely accidental invalidation during reorgs.

If we expect the near future to include more expressive scripting, more use of contract protocols, and more use of RBF, then I don't think we need to be as careful about protecting transaction chains against  reorgs as we have been in the past.

[quote="reardencode, post:2, topic:696"]
That said, sponsors (or other transactions that are provably re-bindable to either a current txo or one of its ancestors) could be allowed to spend before maturation
[/quote]

I think that works fine with the scheme described in my OP.  If we set a coinbase-like maturity limit on the output of any sponsor transaction, then any one of those outputs could be spent as the input to another sponsor transaction prior to maturity.

-------------------------

reardencode | 2024-03-19 21:31:38 UTC | #4

[quote="harding, post:3, topic:696"]
If we expect the near future to include more expressive scripting, more use of contract protocols, and more use of RBF, then I don’t think we need to be as careful about protecting transaction chains against reorgs as we have been in the past.
[/quote]

I love this perspective, and apparently have been delinquent in staying up to date on the topic!

Essentially what we're saying is that things like OP_EXPIRE or OP_SPONSOR that can make a confirmed outpoint invalid may have special treatment, or not, in terms of what users do with them depending on the protocols involved. For example, an OP_SPONSOR chain is a safe input to anything that uses TXO-rebindable locking, but not to an ATLC. Outputs from an spend restricted with OP_EXPIRE might require greater POW confirmation (just as historically transactions signaling RBF have).

-------------------------

ajtowns | 2024-03-20 02:18:55 UTC | #5

Some thoughts:

 * If you require the commitmed txids were sorted by txid before being committed to, you could index the txs in the block slightly more efficiently: for your example, `[2, 3, 0, 1]` could instead be expressed as `[2, 0, 0, 1]` which would indicate that you have three txs (2), one of which is immediately prior (skip 0 txs, txid1), another is immediately prior to that (skip 0 txs, txid2) and the final one is two txs prior to that (skip 1, txid0). Since you're getting smaller numbers (perhaps mostly 0), you should be able to do a pretty efficient bit-oriented encoding. Once you know you're commiting to [txid1, txid2, txid0], you just sort them, then hash the result to validate the commitment. OTOH, perhaps a larger tx that has constant size is better than a smaller one with variable size.

 * I was going to say that I figure this would warrant a new mempool structure for a set of dependent txs that need to be mined together, but I think perhaps cluster mempool's chunking might be sufficient for that? If you have `tx1` and `tx2`, then a sponsor `s2` of `tx2`, followed by a sponsor `s12` of both, followed by a much higher feerate sponsor `s1` of just `tx1`, then that would perhaps cluster as `[tx1 s1], [tx2 s2 s12]`, and the only special processing you would need to do is to evict `s12` from the mempool if `tx1` is mined first. Note that in this case `tx1` might be much earlier in the block than `s12`, so the commitment index might be large. Obviously you need to track the relationship between `s12` and `tx1, tx2` etc, so that you can update the indexes in `s12` when including it in a block, and remove it from the mempool if either `tx1, tx2` were included in a block or if either of them are removed from the mempool for any other reason.

 * Having sponsors and the tx they sponsor in the same chunk means the commitment index won't change depending on where in the block the tx goes; having it in a different chunk means it will change. Having the commitment index be unpredictable means the wtxid changes, which makes it difficult to cache the tx's validity (did the wtxid just change because of the commitment, or does it also have a different signature?), which presumably slows down block validation somewhat, which kind-of sucks. I guess we could introduce another txid type that skips committing to the sponsored txs' indexes, and cache validity based on that txid type.

 * For a miner who isn't aware of sponsoring behaviour; they would consider any sponsoring tx non-standard in the first place and not accept them into their mempool at all, which would mean this mostly isn't a big deal for reorg safety as far as miners are concerned. This is different from coinbase outputs in that you'd expect most reorgs to pick up the same sponsored tx, so spends of the sponsored tx would remain valid -- contrast this to coinbase outputs, where ~100% of reorgs will involve the reward/fees going to a different miner/pool, ensuring spends of the original output would become invalid ~100% of the time.

 * I think the incremental costs of different options of paying for a tx look like:
    * adding an input and change: 100vb
    * rbf'ing a tx that already has a change output: 0vb
    * cpfp'ing a tx via its change output: 110vb
    * adding an input and change via an ephemeral anchor: 160vb
    * off-chain payment to a sponsor who has to create a new sponsoring tx: 112vb
    * off-chain payment to a sponsor who can rbf an existing sponsoring tx with an additional 2B index: 0.5vb
    * off-chain payment to a miner: 0vb

-------------------------

sdaftuar | 2024-03-26 18:46:13 UTC | #6

[quote="harding, post:3, topic:696"]
* Several of the soft fork proposals people are currently advocating for will allow scripts to verify SPV proofs, at which point it’ll be possible for scripts to implement the functionality of transaction sponsorship, `OP_EXPIRE`, and several other features that are not reorg safe.
[/quote]

Could you elaborate on what the soft fork proposals are that would allow scripts to verify SPV proofs, which you mention here?

Regarding OP_EXPIRE, I think a significant drawback besides the reorg safety issue is the need to come up with a mempool design that allows for relaying of transactions which might become invalid at some point in the future.  I think there are obvious DoS issues which come up, which while may be solvable, would require significant engineering effort.

Relatedly, I think that transaction sponsorship also opens up a lot of engineering effort to get mempool policy to work properly, and that many of the underlying issues that it tries to solve require work at the policy layer that could just as readily be done without transaction sponsorship.  I tried to describe this perspective in my mailing list post that you already linked, so I won't repeat those points here.  However, one additional point I'll make is that I think if we ever were to go down this sponsorship road, the right way to do this would be for transactions to opt-into being sponsorable and thus affected by the consensus change.  If we were to let an arbitrary 3rd party on the network attach to the transaction graph of any relayed transaction, I think we'd open up all sorts of DoS concerns with mempool policy.

A simple example in the context of cluster mempool -- if a 3rd party can attach to any transaction, then that could be used to form large clusters out of the mempool that would prevent other parties from being able to chain their own children (due to cluster size limits being hit). Workarounds to this sort of thing may be possible, but it's a lot of work to think through all these issues, and conceptually, I think the right design for any proposed consensus change is to avoid negatively impacting existing use cases/usage patterns.

-------------------------

harding | 2024-03-26 21:43:43 UTC | #7

(Replying out of order)

[quote="sdaftuar, post:6, topic:696"]
Could you elaborate on what the soft fork proposals are that would allow scripts to verify SPV proofs, which you mention here?
[/quote]

I made the assumption (perhaps incorrectly?) that anything that enables `OP_CAT` or an equivalent allows verifying a merkle proof in conjunction with existing opcodes (namely `OP_SHA256`/`OP_HASH256`).  Similarly, those functions also allow verifying proof of work and header chains.  With OP_IF, I think you could be accommodate any consensus-compatible shape of the partial merkle branch.

If you want to implement something like "my transaction expires at height 1,234,567", you require a partial merkle branch that commits to both your transaction and a coinbase transaction with a BIP34 commitment of less than 1,234,567.

Proposals that enable `OP_CAT` or an equivalent include:

- [BIN24-1](https://github.com/bitcoin-inquisition/binana/blob/a1d1daab524007819aca70132e0dd97e0e8caf51/2024/BIN-2024-0001.md)
- [Simplicity](https://github.com/BlockstreamResearch/simplicity)
- [BTC Lisp](https://delvingbitcoin.org/t/btc-lisp-as-an-alternative-to-script/682)
- Elements-style [streaming SHA](https://github.com/ElementsProject/elements/blob/2d298f7e3f76bc6c19d9550af4fd1ef48cf0b2a6/doc/tapscript_opcodes.md#new-opcodes-for-additional-functionality) opcodes
- [MATT](https://merkle.fun/)

[quote="sdaftuar, post:6, topic:696"]
If we were to let an arbitrary 3rd party on the network attach to the transaction graph of any relayed transaction, I think we’d open up all sorts of DoS concerns with mempool policy.
[/quote]

Any time I create an output paying someone I don't trust, doesn't that have the same problem?  Similarly, any time I receive an output in a batched transaction that also pays other people---people I didn't choose to be in a relationship with---doesn't that also create the same problem?  I think it's already the case that typical users can be significantly affected by transaction graph dependencies that they have no control over.

I'm not opposed to an opt-in flag, but that has the downside of making blockchain analysis easier, so I'd love to avoid it if possible.  If it's possible to make cluster mempool safe for always-allowed sponsorship, I think that would also address the existing problems described above.

[quote="sdaftuar, post:6, topic:696"]
many of the underlying issues that [fee sponsors] tries to solve require work at the policy layer that could just as readily be done without transaction sponsorship
[/quote]

I don't understand this point.  Let me quickly quote what I believe is the entire text you previously wrote about this:

> the fee bumping improvement that this proposal aims at is
really coming from the policy change, rather than the consensus change. But
if policy changes are the direction we're going to solve these problems, we
could instead just propose new policy rules for the existing types of
transaction chaining that we have, rather than couple them to a new
transaction type.

It seems to me like the problem you're attempting to solve here is transaction pinning.  That's a problem created by policy, so it makes sense to me that we can solve it by policy alone.

But there's another problem which has come to my attention more recently, which is the high cost of exogenous fee bumping compared to paying miners out of band.  When the cost difference is too high, users might be incentivized to work with large miners directly, undermining Bitcoin's decentralization.

Peter Todd has wielded that argument as a weapon against v3 policy and ephemeral anchors, which I think is unfair: those policies improve the current state of using Bitcoin for a large number of users, so I think we should pursue them (and thank you for all of your work on them!).  But I find his fundamental argument worth thinking about.

That's where I think sponsors are interesting.  Whereas ephemeral anchors cost ~11 vbytes for the anchor output and ~41 vbytes for the spend, this improved sponsor proposal is only ~0.5 vbytes.  It's almost the same as the 0 vbytes used when paying a miner out of band or when using RBF under ideal circumstances.  It gives us all of the advantages of exogenous fees (for contract protocols and also for regular fee bumping) without it's major long-term downside of potentially worsening decentralization.

I don't think it's possible to achieve that efficiency benefit using policy only, at least not if we stick with a UTXO model.  Any policy we create that boosts the priority of transaction A using funds from transaction B must be rooted in the consensus rules, otherwise miners will claim the value from B without mining A.  But, if I'm missing something, I would love to learn more!

-------------------------

sdaftuar | 2024-03-26 23:36:27 UTC | #8

[quote="harding, post:7, topic:696"]
I made the assumption (perhaps incorrectly?) that anything that enables `OP_CAT` or an equivalent allows verifying a merkle proof in conjunction with existing opcodes (namely `OP_SHA256`/`OP_HASH256`). Similarly, those functions also allow verifying proof of work and header chains. With OP_IF, I think you could be accommodate any consensus-compatible shape of the partial merkle branch.
[/quote]

Unless I'm missing something, I think any scripts that could be deployed using just tools like this would in fact be reorg safe -- you'd need a way for a script to pull in data from the active headers chain itself in order to become invalid on a reorg.

(Thanks for linking to those examples -- from a quick look, I don't believe any of them propose op codes that would allow for inspecting anything outside of the transaction that is being validated, so as far as I can tell each of those proposals would be reorg safe.  Please let me know if I'm mistaken!)

[quote="harding, post:7, topic:696"]
Any time I create an output paying someone I don’t trust, doesn’t that have the same problem?
[/quote]

I believe there's a substantial difference between a single transaction being able to be griefed by someone else, and anyone on the network always being able to grief anyone else on the network at any time.

[quote="harding, post:7, topic:696"]
But there’s another problem which has come to my attention more recently, which is the high cost of exogenous fee bumping compared to paying miners out of band.
[/quote]

I think it may be helpful to adopt the terminology that @instagibbs has described in https://delvingbitcoin.org/t/taxonomy-of-transaction-fees-in-smart-contracts/512.

As @instagibbs points out there, exogenous fees can (in theory) be used to pay for a transaction with or without CPFP; similarly, endogenous fees can also in theory arise with or without CPFP.

If we wanted to be true blockchain-space-minimalists, we would be arguing that all protocols move towards an endogenous-single-tx model only.  I don't work on layer 2 protocols myself and so don't consider myself well-versed in what is possible and what is not, but my inclination is to think that this is probably overly restrictive.  Of course, if people come up with protocols that work within that guideline, then that's of course great.

I think what the transaction sponsor model is trying to do is make CPFP cheaper/more efficient in situations where a protocol is unable to bring endogenous fees to a transaction, compared with using anyone-can-spend outputs.  However, in order to be workable, Jeremy had proposed a number of policy rules on top of the consensus change, which were needed in order to make sponsors useful as a CPFP/RBF alternative. From his [mailing list post](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-September/018179.html):

> In this BIP, we only care to ensure a subset of behavior sufficient to replace CPFP and RBF for fee
> bumping.
> 
> Thus we restrict the mempool policy such that:
> 
> 1. No Transaction with a Sponsor Vector may have any child spends; and
> 1. No Transaction with a Sponsor Vector may have any unconfirmed parents; and
> 1. The Sponsor Vector must have exactly 1 entry; and
> 1. The Sponsor Vector's entry must be present in the mempool; and
> 1. Every Transaction may have exactly 1 sponsor in the mempool; except
> 1. Transactions with a Sponsor Vector may not be sponsored.
> 
> 
> The mempool treats ancestors and descendants limits as follows:
> 
> 1. Sponsors are counted as children transactions for descendants; but
> 1. Sponsoring transactions are exempted from any limits saturated at the time of submission.
> 
> This ensures that within a given package, every child transaction may have a sponsor, but that the
> mempool prefers to not accept new true children while there are parents that can be cleared.
> 
> To prevent garbage sponsors, we also require that:
> 
> 1. The Sponsor's feerate must be greater than the Sponsored's ancestor fee rate
> 
> We allow one Sponsor to replace another subject to normal replacement policies, they are treated as
> conflicts.

Looking at this now, this seems extremely similar to the rule set for v3 (TRUC!) children[^size] -- an opt-in policy to make RBF work better, while still allowing for CPFP.  In Jeremy's original post, he suggested that:

[^size]: If I'm not mistaken, Jeremy's original proposal didn't seem to include a bound on the size of a sponsor transaction, something which is part of the TRUC/v3 proposal -- without that, RBF pinning is still possible in a situation where someone creates a large sponsor transaction that is low feerate (but still above the feerate of the parent), making it difficult to replace with another higher feerate sponsor transaction.

> What is wanted is a minimal mechanism that allows arbitrary unconnected third parties to attach
fees to an arbitrary transaction. The set of rules given tightly bounds how much extra work the
mempool might have to do to account for the new sponsors in the worst case, while providing a "it
always works" API for end users that is not subject to traditional issues around pinning.

What I argue instead is that the *policy changes*, not the *consensus changes* are what allow us to avoid the "traditional issues around pinning" -- and the v3/TRUC proposal is an example of this.

So given that the policy changes are what is helpful here, the question becomes: is it worth changing the consensus model in order to save ~52 vbytes for protocols that rely on exogenous-fee-CPFP in order to fund transaction fees?

Exogenous-fee-CPFP is already still less efficient than paying a miner out of band to mine a given transaction, so if the concern is the miner-centralization-issue, then we should really abandon such protocols in favor of endogenous-fee-single-tx schemes.  Given that we're willing to tolerate some level of blockchain-inefficiency to make such protocols work, my personal view is that tradeoff to save those ~52vbytes is not worth it, and at a minimum, I'd suggest that the community first deploy exogenous-fee-CPFP protocols using existing methodologies (like anyone-can-spend outputs) to demonstrate utility and adoption before we seriously consider changing the consensus model.  

And even if somehow such protocols gained enough adoption that shaving a few bytes to make them more efficient catches on, I'd still argue that we should not opt existing transactions into this behavior, because I believe there will always be unintended policy side effects from allowing arbitrary third parties to attach to the transaction graph, and I think good soft-fork design should compel us to avoid causing any potential harm to existing use cases/users when rolling out new features.

-------------------------

harding | 2024-03-27 01:13:31 UTC | #9

[quote="sdaftuar, post:8, topic:696"]
you’d need a way for a script to pull in data from the active headers chain itself in order to become invalid on a reorg.
[/quote]

Yeah, I guess there is a cyclic dependency issue in my previous example given the need for a coinbase transaction to commit to the block's wtxid merkle root, but you need to known the txid of the coinbase transaction before you can construct the witness for a transaction that verifies a merkle proof for a block containing itself.

I think you could implement sponsors themselves with just `OP_CAT` as long as both the sponsor transaction and the transaction it sponsors are in a merkle sub-tree that doesn't include the coinbase transaction.

> I believe there’s a substantial difference between a single transaction being able to be griefed by someone else, and anyone on the network always being able to grief anyone else on the network at any time.

I agree, and I don't claim a false equivalency.  However, I think it's an existing problem that would be nice to solve, especially if the solution also makes very low cost sponsorship more appealing.  I'll try to spend some time better understanding how cluster mempool works so that I can better understand your concerns.

[quote="sdaftuar, post:8, topic:696"]
Exogenous-fee-CPFP is already still less efficient than paying a miner out of band to mine a given transaction, so if the concern is the miner-centralization-issue, then we should really abandon such protocols in favor of endogenous-fee-single-tx schemes.
[/quote]

I think this is my point: single-transaction sponsors can now be exactly as efficient as paying a miner OOB (zero overhead).  For a sponsor transaction with multiple dependencies, it's 0.5 vbytes per dependency.  That's less than 0.5% overhead to sponsor a 1-input, 1-output P2TR.

We now have an exogenous fee-paying mechanism that uses almost the same amount of space as the best endogenous mechanisms.  (And, in many cases, sponsors effectively uses less space!)  I don't think that's something we should rush to soft fork in, but it brings me a lot of comfort when thinking about increased use of exogenous fees.  If we as users ever do become concerned about excessive OOB fee paying, we have a drop-in solution.  Additionally, if more and more of the network shifts to the use of protocols that depend on exogenous fee paying, we'll have a scaling improvement ready to go.

I think the efficient multiple-use form of sponsors described earlier in this thread really requires a new witness version, which will automatically provide some level of opt-in given how few wallets/services seem to have implemented automatic bech32(m) forward compatibility.  I have some other ideas for things we could do for (nearly) free at the same time, which I plan to open a separate thread about after I've spent at least a week collecting some data about use of reorg safe transaction chains on the existing network.  But, in general, I currently think allowing sponsorship is something we should probably add the next time we plan to create a new witness version for general use---not something I think we should rush to add next week.

-------------------------

ajtowns | 2024-03-27 02:08:03 UTC | #10

[quote="sdaftuar, post:8, topic:696"]
Unless I’m missing something, I think any scripts that could be deployed using just tools like this would in fact be reorg safe – you’d need a way for a script to pull in data from the active headers chain itself in order to become invalid on a reorg.
[/quote]

I have a concept for per-input timelocks and reorg safety that would trigger that, for what it's worth: namely "any input can present an annex entry that commits to `height || bytes`; if there is not a prior block at height `height` whose hash ends with `bytes`, the tx is invalid". Presuming the script could access that annex entry, it could check that `bytes` is 32B long, then check for a 48B+32B value that hashes to it, then use the 32B value as the tx merkle root.

The "normal" use of that annex entry would be either as a per-input timelock (0-length bytes), so that presigned spends that use variable timelocks could be combined into a single tx (HTLC spends?), to prevent signet/testnet signatures being replayed on mainnet (commit to the last byte of block 1), to fork coins in a hardfork scenario (BCH's block 478559 ends in `ec`, BTC's ends in `48`), or (perhaps) to invalidate a tx should a reorg occur (you refund 0.5 BTC due to a mistaken payment, but the refund doesn't actually spend the payment txo, perhaps because that was already spent for some other reason, so you commit to the last 4 bytes of a subsequent block's hash effectively increasing the PoW required to replace it by 4B times). I say "perhaps" in the last case, because I could at least imagine this having an "implies a timelock of height+100 blocks" restriction which would make that usecase not very useful.

-------------------------

instagibbs | 2024-03-27 15:32:38 UTC | #11

[quote="harding, post:9, topic:696"]
we’ll have a scaling improvement ready to go.
[/quote]

A little push-back here: this precludes batched CPFP, which is somewhere in-between single-tx exo fees and single CPFP exo fees, depending on how many "well I was just making txs already" cases are in play.

I suspect if we really wanted to squeeze every byte(I'm unconvinced, but this thread is for people who are interested in doing so :) ), you'd want to support both this type of sponsor and the more general original one, with the associated complexities involved.

(or we figure out a softfork for robust single tx exo fees, with stacking ala @rustyrussell dream world!)

-------------------------

JeremyRubin | 2024-03-27 16:36:12 UTC | #12

Any protocol that is ever to be at all secure on Bitcoin has to be "progress friendly", that is, if there is a set of currently minable transactions $S$, the protocol works as expected if any $s \in S$ is mined, and any such $s$ being mined is a good outcome.

The idea that protocols could be "unhappy" if $s_0$ is mined instead of $s_1$ is weak.

Suhas' point about pinning is rendered more or less moot if you, as a point of policy, only allow sponsors if they put a cluster into the next block (or maybe a few blocks out). All users will be happy in that case because -- irrespective of if they were pinned for one block -- their protocol has made progress as some tx has been mined. This isn't really a problem for sponsors, because theres no point in sponsoring far ahead of when it is likely to be included.

For example, LN-Symmetry can be progress friendly because if the ratchet state is at 12321, and the miner chooses to mine state 112 instead of 12321 because of a sponsor, then 12321 can still be broadcast later at no additional cost to the participant wishing to finalize that state. Each update also expands the timeout before final claim, so there is always enough time even with a long sequence of bad updates.

Should there be any protocol which is attaching any value to preferring $s_1$ to $s_0$, and expecting to be able to have broadcast both to the network and have $s_1$ be guaranteed to be mined, is not secure and there is no reason to even attempt to support it beyond miner rationality. Sponsors to the rescure -- $s_1$ or $s_0$ are only preferred if they pay more directly or via a sponsor package.

To make this even more abundantly clear, a malicious 3rd party is only willing to sponsor $s_0$ over $s_1$ when there is some profit extractable by including $s_0$ instead of $s_1$. If there _is_ such a profit to be extracted, then we should expect (with or without sponsors) a miner to either figure out how to extract that profit themselves, or to accept an out-of-band bribe to include $s_0$ over $s_1$. Worse still, if out-of-band bribes are popular, you'd only do it to "trustworthy" miners, leading to a profit edge for centralization.

To really take it home, even with a normal use case of RBF, the preferences are backwards. Suppose a set $S$ where all $s$ make the same payments, less fees taken from a change output, and fees are strictly increasing. In general, a user prefers if $s_0$ is mined, as it pays lower fee, but only issues $s_1$ because $s_0$ didn't get mined. If $s_0$ were to be instead bumped by an out-of-band payment, or by a sponsor, then that user would be happier than $s_1$ was mined. Even if you do rolling batches from an exchange, where every $s_{i+1}$ pays more people than in $s_i$ (ignoring the minrelayfee increments meaning that $s_i$ is better, a reason this technique isn't popular) the batcher would be perfectly happy with having someone else pay for inclusion of an earlier state, since ultimately it means a fee savings to them, even if they have to reissue another transaction later (e.g., out of the change to guarantee common input).

The only case where you're unhappy with $s_0$ over $s_1$ is when you are making bad assumptions about how Bitcoin works.

(N.B. $S$ is the available transactions, so excluding future timelocked txs. Protocols like Lightning rely on an assumptions of execution speed)

-------------------------

sdaftuar | 2024-03-28 23:45:26 UTC | #13

[quote="harding, post:9, topic:696"]
I think you could implement sponsors themselves with just `OP_CAT` as long as both the sponsor transaction and the transaction it sponsors are in a merkle sub-tree that doesn’t include the coinbase transaction.
[/quote]

Not sure if we're talking past each other, but I just want to emphasize that there is a difference between implementing Jeremy's transaction sponsors proposal as he originally described, and implementing a script-verification-form of sponsors using OP_CAT to prove inclusion of transactions in some merkle root. Doing it in script, without a way to pull in the merkle root or headers chain from the actual active blockchain, will always be different from Jeremy's original proposal, because in the event of a reorg **the script will still be valid**, even if the merkle root that is referenced in a script is not in the active chain.

Another way of putting it is that the script-verification form of sponsors is equivalent to saying that a transaction must appear in some merkle root that has some high proof of work on it, not that the transaction must appear in the same actual block in the active chain.

Because the script form of this is just an approximation of transaction sponsors, it is incorrect to say (as far as I can tell at least) that these proposed script extensions would already give us a change to the reorg-safety consensus model that sponsors would introduce. 

[quote="harding, post:9, topic:696"]
I think this is my point: single-transaction sponsors can now be exactly as efficient as paying a miner OOB (zero overhead). For a sponsor transaction with multiple dependencies, it’s 0.5 vbytes per dependency. That’s less than 0.5% overhead to sponsor a 1-input, 1-output P2TR.
[/quote]

I don't think this is true?  Paying a miner out of band can be ~zero overhead, eg if a miner is paid using a lightning payment.  But using CPFP introduces an on-chain cost corresponding to the size of the CPFP transaction.  An average transaction right now is, say, 300 - 500 vbytes?  That means that we're talking about saving 10-15% of the cost of a CPFP by using an efficient sponsors implementation, versus something like Ephemeral Anchors.  

[quote="JeremyRubin, post:12, topic:696"]
The idea that protocols could be “unhappy” if $s_0$ is mined instead of $s_1$ is weak.
[/quote]

It took me a while to understand that you were responding to my post here!  Eventually I realized that you may have interpreted my comments about griefing as being a narrower statement about a 3rd party interfering in the network's choice over which one of a pair of spends is eventually confirmed.

Overall, I agree that it'd be great for protocols to be robust to any version of their transactions confirming, like lightning tries to be as I understand it.  (This obviously doesn't hold for some very basic "protocols", like a user double-spending some recipient, and there is a competition between the original spend and the replacement, but I don't think this is interesting enough to debate.)

However, there are other side effects to allowing 3rd parties to attach to your transaction than just pinning-type issues, or determining which version of various conflicting transactions gets mined.  One example is that you could use up all the mempool policy limits reserved for transaction chains via the sponsoring transaction, preventing a user from (say) spending their own unconfirmed outputs until the parent confirms.

I imagine this can also interfere with their own attempts to fee bump via CPFP? I'm imagining a situation where a transaction with its sponsor might confirm in the second block, but with some other high feerate child could confirm in the first block without the sponsor.  A miner might choose to hold on to the parent for the second block rather than confirm it without the sponsor in the first, to avoid invalidating the sponsor and having that drop out of the mempool?  Maybe there will end up being good answers to incentives questions like this, but I think we clearly haven't done the work yet to sort out all these incentive questions yet, and so forcing all transactions into a regime where they can be bumped by third parties seems like a stretch.

[quote="JeremyRubin, post:12, topic:696"]
Suhas’ point about pinning is rendered more or less moot if you, as a point of policy, only allow sponsors if they put a cluster into the next block (or maybe a few blocks out).
[/quote]

I agree that this is a potential mitigation, but just wanted to flag that as some of us have been thinking this through in related contexts (https://delvingbitcoin.org/t/v3-and-some-possible-futures/523/4), there can be issues that still come up if (say) a transaction package moves to near the top block, and then the child is replaced away and the parent is left without the high feerate child/sponsor.  If a large, low-feerate variant of the parent is what is left, then this might be pinning a higher feerate variant of the transaction.  

Again, while there ultimately may be good answers to these questions as we do more work, I think for now it would be too soon to say that we're sure we have a policy framework that is so robust and beneficial that everyone should be opted in to it.

-------------------------

harding | 2024-03-29 22:33:16 UTC | #14

[quote="sdaftuar, post:13, topic:696"]
in the event of a reorg **the script will still be valid**, even if the
merkle root that is referenced in a script is not in the active chain.
[/quote]

Argh, yes, you're correct, of course.  I really didn't think that one
through.  Thank you for putting up with me!

[quote="sdaftuar, post:13, topic:696"]
That means that we’re talking about saving 10-15% of the cost of a CPFP
by using an efficient sponsors implementation, versus something like
Ephemeral Anchors.
[/quote]

It was pointed out to me elsewhere that I'm not being as clear as I
should be about absolute costs versus marginal costs.

I agree with you about the absolute cost.

However, let's imagine any regular transaction could sponsor at least
one other transaction with no prior setup.  Let's also assume there will
continue to be people and organizations who broadcast at least one
transaction every hour or so, including someone named Alice.  This
allows Bob to pay Alice to make her next transaction a sponsor
transaction and the marginal increase in her transaction size over the
transaction she was planning to create anyway is ~0.5 vbytes.

By comparison, Bob could include an ephemeral anchor output in his
transaction, which would also allow Bob to pay third-party Alice (via an
off chain mechanism) to CPFP his transaction.  Alice would need to
include an extra ~40 vbytes in her transaction and would need pay for
Bob's transaction containing an extra ~11 vbytes (51 vbytes total).

Whether using anchors or sponsors, I'm not aware of any way for Bob to
pay Alice trustlessly offchain (without prior setup that adds
significant additional costs and complexity).  Bob will be paying Alice
with no guarantee of result; she could take his money and not sponsor
his transaction.  I'm pretty sure that's also the case with paying a
large miner OOB, so the individual user risks of all approaches for
paying for third party fee bumping seem roughly equivalent to me.

With both anchors and sponsors, Bob can of course trustlessly fee bump
his own transaction.  However, Bob can only fee bump his own transaction
if he has an unencumbered UTXO to spend.  If we start to see adoption of
pooling technologies (e.g. joinpools, channel factories, and timeout
trees), delayed spending protocols (like vaults), and a general increase
in typical feerates, it seems likely to me that there will be many users
who use trustless or trust-minimized offchain protocols but don't have
convenient access to an unencumbered UTXO.  For those users, obtaining
an unencumbered UTXO on short notice might require an additional
transaction onchain (or even more than one to do trustlessly, as things
like submarine swaps require two onchain transactions to settle).

To compare between Bob's different options at some point in the
future, let's assume that he knows he will later have a transaction
whose base size is 500 vbytes, that the typical fee for a transaction that
size is $1,000 USD (in circa 2024 dollars), and that he also has a
reliable offchain mechanism for paying third parties.

- $1,000 (500 vbytes) is his overall cost using endogenous fees and
  transmitting on the P2P protocol.

- $1,000 (500 vbytes) is his overall cost using OOB payment to miners.
  There may be a delay to confirmation and there's a risk a miner will
  take his money without mining his transaction.  He'll probably have to
  pay a slight premium over this so that the miner profits.

- $1,001 (500 + 0.5 vbytes) is his overall cost using sponsors via the
  P2P protocol with a third party who was already going to create a
  transaction.  There's a risk the third party will take his money
  without sponsoring his transaction.  He'll probably have to pay a
  slight premium over this so the sponsor profits.

- $1,022 (511 + 0 vbytes) is his overall cost of adding an anchor output but
  later deciding to OOB pay miners.  There may be a confirmation delay,
  he risks miners taking his money without providing confirmation, and
  there will likely be a slight premium.  This also leaves an output in
  the UTXO set indefinitely.

- $1,102 (511 + 40 vbytes) is his overall cost using ephemeral anchors
  via the P2P protocol with a third party who was already going to
  create a transaction.  There's a risk the third party will take his
  money without sponsoring his transaction and he'll probably have to
  pay a slight premium so that they profit.

- $1,322 (511 + 150 vbytes) is his overall cost using an ephemeral
  anchor via the P2P and fee bumping it himself, assuming he already
  had a UTXO ready to spend.

- $1,722 (511 + 200 + 150 vbytes) is a rough estimate of his overall
  cost using an ephemeral anchor via the P2P protocol plus trustlessly
  obtaining a UTXO to fee bump the anchor himself.

I don't think we can rule out a future where feerates are the equivalent
of $2/vbyte, and I think there would be quite a few people in that
potential future who would be willing to save $100 or more per
transaction by paying miners OOB.  Even if fees were $0.2/vbyte, a level
we've seen, I think there are a significant number of people who would
be willing to pay miners OOB to save $10 per transaction, especially if
there is a service (as has been proposed) to make paying OOB easy.

It adds significant overhead and complexity to contract protocols to use
incremental presigned RBF fee bumps for a wide range of potential fees,
so they're probably always going to depend at least partly on some sort
of exogenous fee mechanism.  Paying OOB may look unexciting given
that it only saves 10% over ephemeral anchors in the best case, but the
frequency at which I receive store coupons that offer around 10%
indicates that many people find that amount of savings to be highly
motivating.  Sponsors gives us the chance to offer that same 10%
discount that large miners can offer without the risk to mining
decentralization.  I think that makes it worth pursuing.

Thanks to the discussion on this thread, I realize there are several
challenges to sponsors that I need to do a better job of addressing,
particularly reorg safety, avoiding the creation of new pinning vectors,
and ensuring sponsors otherwise interact well with mempool structures
and algorithms.  I'm investigating those now.

-------------------------

