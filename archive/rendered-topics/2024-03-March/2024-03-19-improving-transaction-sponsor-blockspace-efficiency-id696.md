# Improving transaction sponsor blockspace efficiency

harding | 2024-03-19 18:58:52 UTC | #1

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

[sponsors]: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-September/018195.html
[topic sponsors]: https://bitcoinops.org/en/topics/fee-sponsorship/
[ephemeral anchors]: https://bitcoinops.org/en/topics/ephemeral-anchors/

-------------------------

reardencode | 2024-03-19 19:22:22 UTC | #2

As I [posted](https://twitter.com/reardencode/status/1770094336434430157) on X, the biggest complication with sponsors is that the sponsor transaction can become invalid depending on innocent block reorderings, which would require it to be treated as a coinbase output and mature before it can be used in a normal transaction. I think this prevents sponsorship from being done as a part of regular transaction operations.

That said, sponsors (or other transactions that are provably re-bindable to either a current txo or one of its ancestors) could be allowed to spend before maturation, which means that a dedicated sponsor TXO could be spent repeatedly without maturing between spends, and only a subsequent spend that would not be re-bindable would have to wait for maturity.

-------------------------

