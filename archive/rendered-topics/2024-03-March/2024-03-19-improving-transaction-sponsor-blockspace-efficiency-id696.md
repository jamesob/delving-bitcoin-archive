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

