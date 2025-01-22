# [BROKEN] Multi-Party Eltoo with bounded settlement

ademan | 2025-01-08 00:15:37 UTC | #1

# Disclosure

Since there seem to be a few eyes on this now I thought I'd better disclose the issue I had hoped to silently patch before anyone really read this over.
Over the weekend I've spent some time on implementing this scheme and found several problems with my writeup (to be addressed).
*More importantly, I discovered that when penalties are used, a malicious party can withhold their state update signature but collect everyone else's so that they are the only one who can publish S+1, and honest parties are prevented from publishing state S because of the penalty*.
Thank you to everyone for the thoughtful feedback and further reading, I hope that I can plug this hole in the protocol and reply properly later this week.
It's possible this proposal may not survive the fix at all, unfortunately...
I probably published this prematurely.

# Introduction

Eltoo schemes have desirable properties such as simplicity, constant storage requirements, and compatibility with multi-party channels.
However, naive eltoo is vulnerable to an attack where a dishonest party submits old update transactions in an attempt to delay settlement of the channel state until HTLCs have expired.

This proposes a multi-party eltoo scheme that both allows penalizing dishonest behavior, and provides a bounded settlement time.
By giving parties only one chance to update the channel state on-chain, this scheme drastically reduces the ability of malicious parties to prevent honest settlement of the channel.

This proposal depends on a way to make signed vector commitments, and check vector committed items against a spending transaction.
I call this construct "multi-transaction-signature"s.
This proposal also depends on "floating transactions" (as used in other eltoo schemes).
Therefore, we will assume the LNHANCE soft fork, consisting of `OP_INTERNALKEY` (IK), `OP_CHECKTEMPLATEVERIFY` (CTV), `OP_CHECKSIGFROMSTACK` (CSFS), and `OP_PAIRCOMMIT` (PC).
In LNHANCE CTV+CSFS enables the "floating transactions" construct, and CTV+CSFS+PC enable this "multi-transaction-signature" construct.
I believe this scheme could be adapted to ANYPREVOUT(APO)+CSFS+PC however I'm admittedly less familiar with APO than I ought to be.

# Prior Art

This is likely completely non-novel, but unfortunately I'm unfamiliar with most of the literature.
My apologies to anyone whose work I almost certainly overlooked.

# Acknowledgements

* Moonsettler for review, tentative approval, and driving the current LNHANCE efforts and propsing an opcode (`OP_PAIRCOMMIT`) that enables this without all of the aspects of CAT that scare me.
* ReardenCode for review, comments, and being open to discussion on xitter which encouraged me greatly in getting back into Bitcoin dev.
* Everyone else who read this mess of a draft and offered comments or encouragement (or not-so-encouragement, you know who you are! lol)
* The all of the Bitcoin giants out there, I'm not standing on your shoulders but I'm probably stepping on your feet.

# Description

This scheme bounds the number of times an update transaction can be submitted for a multi-party channel by limiting each party to submitting a single update.
State updates occur similar to Decker, Russell, and Osuntokun eltoo by using `CHECKLOCKTIMEVERIFY`(CLTV) and an incrementing absolute timelock to ensure that update transactions can only be spent by either the appropriate settlement transaction or a newer update transaction.
To remove parties' ability to update more than once, every state update consists of a large set of update transactions arranged into generations.
Every generation allows a party to submit an update, but removes that party from the set of parties eligible to submit an update in the future.
Each unique update transaction has a unique set of parties that are still eligible to submit updates.
Each generation `1 <= K < N` has one transaction for every possible subset of length `N - K` of parties still eligible to submit updates.
These update transactions are divided into generations `1 < K < N` of `N choose K` transactions each.
Generation 1 prohibits 1 party from updating any further and has `N choose 1` transactions in it.
Generation 2 prohibits 2 parties from updating any further and has `N choose 2` transactions in it.
Generation N-1 prohibits all but 1 party from updating.
The total number of all of these update transactions is ~`2^N - 1 + N` which obviously grows very quickly.
After an update transaction is accepted either because the last party submitted their update transaction, or by a timelock expiring, a settlement transaction splitting up the channel balance can be submitted as in other eltoo schemes.
This settlement transaction can optionally penalize misbehaving parties by dividing their channel balance among the honest parties.

This scheme as described so far, achieves both of the stated goals, bounding settlement time and optionally enabling punishment.
It does, however, require sharing `2^N` signatures *per party*.

Using `OP_PAIRCOMMIT` we can instead sign a commitment which authorizes many transactions, requiring only a merkle proof proving that a given CTV template has been signed, which indicates a valid state transition.
This compression from `2^N` signatures to one per party enormously reduces network communications required, and also makes it feasible to use musig to produce a single signature instead.
However, for the sake of reduced interactivity, this proposal will use one signature per party.

Outputs on each update transaction are Segwit v1, and have `P=N-K` tap leaves, one for every party still eligible to update.

The witness includes a merkle proof that the transaction has been signed for, as well as signatures on that merkle root from all parties (even ones no longer eligible to update), *except* for the updating party.
The updating party provides a regular `SIGHASH_ALL` signature which is checked with `CHECKSIGVERIFY`.
This ensures other parties cannot submit an update transaction on behalf of a different party.
Each tapscript follows a fixed path in the merkle tree, which binds it to transition to a specific transaction.
If parties could submit an arbitrary inclusion proof in their transaction, they could for instance fraudulently submit one that removes a different party from the eligible-to-update set.

To illustrate, here is what a generation 0 tap script (from the commitment TX) might look like for A to update.
Because party A is updating, the spending transaction must match the CTV template `TX_BCD_TEMPLATE` whose output is identical to this transaction, but without a tap leaf for A to submit another update.
This removes party A's ability to update further.
In addition to this one, generation 0 would have equivalent tap leaves with scripts corresponding to each of the other parties in the channel.

```
witness: <DSIG> <CSIG> <BSIG> <GEN2_MERKLE_ROOT> <ABD_ABC> <TX_ACD_TEMPLATE> <TX_BCD_TEMPLATE> <sig(A)>
tapscript:
<S+1> CHECKLOCKTIMEVERIFY DROP # <DSIG> <CSIG> <BSIG> <GEN2_MERKLE_ROOT> <ABD_ABC> <TX_ACD_TEMPLATE> <TX_BCD_TEMPLATE> <sig(A)>
<A>                            # <DSIG> <CSIG> <BSIG> <GEN2_MERKLE_ROOT> <ABD_ABC> <TX_ACD_TEMPLATE> <TX_BCD_TEMPLATE> <sig(A)> <A>
CHECKSIGVERIFY                 # <DSIG> <CSIG> <BSIG> <GEN2_MERKLE_ROOT> <ABD_ABC> <TX_ACD_TEMPLATE> <TX_BCD_TEMPLATE>
CHECKTEMPLATEVERIFY            # <DSIG> <CSIG> <BSIG> <GEN2_MERKLE_ROOT> <ABD_ABC> <TX_ACD_TEMPLATE> <TX_BCD_TEMPLATE>
SWAP                           # <DSIG> <CSIG> <BSIG> <GEN2_MERKLE_ROOT> <ABD_ABC> <TX_BCD_TEMPLATE> <TX_ACD_TEMPLATE>
PAIRCOMMIT                     # <DSIG> <CSIG> <BSIG> <GEN2_MERKLE_ROOT> <ABD_ABC> <BCD_ACD>
SWAP                           # <DSIG> <CSIG> <BSIG> <GEN2_MERKLE_ROOT> <BCD_ACD> <ABD_ABC>
PAIRCOMMIT                     # <DSIG> <CSIG> <BSIG> <GEN2_MERKLE_ROOT> <GEN1_ROOT>
SWAP                           # <DSIG> <CSIG> <BSIG> <GEN1_ROOT> <GEN2_MERKLE_ROOT>
PAIRCOMMIT                     # <DSIG> <CSIG> <BSIG> <GEN1_MERKLE_ROOT>
TUCK                           # <DSIG> <CSIG> <GEN1_MERKLE_ROOT> <BSIG> <GEN1_MERKLE_ROOT>
<B>                            # <DSIG> <CSIG> <GEN1_MERKLE_ROOT> <BSIG> <GEN1_MERKLE_ROOT> <B>
CHECKSIGFROMSTACK              # <DSIG> <CSIG> <GEN1_MERKLE_ROOT> <1|0>
VERIFY                         # <DSIG> <CSIG> <GEN1_MERKLE_ROOT>
TUCK                           # <DSIG> <GEN1_MERKLE_ROOT> <CSIG> <GEN1_MERKLE_ROOT>
<C>                            # <DSIG> <GEN1_MERKLE_ROOT> <CSIG> <GEN1_MERKLE_ROOT> <C>
CHECKSIGFROMSTACK              # <DSIG> <GEN1_MERKLE_ROOT> <1|0>
VERIFY                         # <DSIG> <GEN1_MERKLE_ROOT>
<D>                            # <DSIG> <GEN1_MERKLE_ROOT> <D>
CHECKSIGFROMSTACK              # <1|0>
```

The templates are arranged into a proof tree structured like this:

```
                                                       GEN1_MERKLE_ROOT
                           GEN1_ROOT                                          GEN2_MERKLE_ROOT
            BCD_ACD                         ABD_ABC                     GEN2_ROOT             GEN3_MERKLE_ROOT
TX_BCD_TEMPLATE TX_ACD_TEMPLATE TX_ABD_TEMPLATE TX_ABC_TEMPLATE             |
                                                                            |
                                                   /---------------------------------------------------------\
                                              AB_AC_AD_BC                                                 BD_CD_DUMMY
                                   AB_AC                         AD_BC                         BD_CD                    DUMMY
                       TX_AB_TEMPLATE TX_AC_TEMPLATE TX_AD_TEMPLATE TX_BC_TEMPLATE TX_BD_TEMPLATE TX_CD_TEMPLATE

```

Every party signs the merkle root `GEN1_MERKLE_ROOT`, and that set of N signatures is sufficient for a state update.

The presence of SWAP or NOP opcodes commits to the path in the merkle tree, and prevents invalid state transitions.

Party B's script would instead have the sequence `NOP PAIRCOMMIT SWAP PAIRCOMMIT SWAP PAIRCOMMIT` since the `TX_ACD_TEMPLATE` which excludes B from further updates, is at index 1.
Party C's script would have the sequence `SWAP PAIRCOMMIT NOP PAIRCOMMIT SWAP PAIRCOMMIT` since it is at index 2.
You can see how the index of the desired template in the ordered list of transaction templates corresponds to the SWAP and NOP opcodes as a little-endian representation of bits, with SWAP representing a 0 and a NOP representing a 1.
Later generations are encoded in the lower branches of the tree.
NOPs could be eliminated making some scripts shorter than others.
I think this would have pretty negligible effects on incentives, but it's also simpler for me to think about with them in.

# Without PAIRCOMMIT

This scheme works without PAIRCOMMIT but requires every party to share a signature for every valid state transition.
~`2^N` signatures, per party, need to be shared to make this scheme work without PAIRCOMMIT.

Transitions must be individually authorized, otherwise a blanket signature authorizing a transition to state S would permit any update transaction in `R<S` to transition into any update transaction in state `S`.
Consider a case where parties A,B, and C are eligible to update.
With blanket signatures from B, and C to state S, A could sign a transaction that transitions to a state where A and B are able to update, thereby stealing C's update opportunity and preserving their own.
This is why each valid state transition must be authorized individually.

# Practicality

Each party is required to recompute the entire multi-transaction-signature commitment every channel state update, and this involves computing ~2^N transactions and building a merkle tree committing to all of them.
This means that there are ~`(N - 1) * 2^(N + 1) - 1` SHA256 operations required for a penalty scheme. (Take this math with a grain of salt, I haven't proved it out)
A non-penalty scheme is around twice as fast because there is only one settlement transaction for all updates in a state, rather than one settlement transaction for every update in a state, this takes the complexity to ~`(N - 1) * 2^N - 1`.
For a penalty scheme this results in ~18431 SHA256 `operations for N=10 parties.
The amount of memory required should be logarithmic or better over N so memory should not be a limiting factor.
With a very plausible single core performance of ~1MHash/s on my aging mid-range desktop computer, that means calculating the commitment merkle root will optimistically take ~18ms.
State recomputation can be significantly parallelized, so with 12 threads that's a best case scenario of 1.5ms.
This is all very optimistic and assumes the other computations in state recalculation take negligible time.
The practical limit for this scheme is probably in the 10-20 party range because the exponential growth more than doubles for every party added.
Using these very optimistic assumptions, 17 parties, a penalty scheme would take ~743ms to recalculate.
I unfortunately don't know how many parties other schemes consider practical, but this scheme is fairly limited due to the exponential size of the number of potential update transactions that need to be computed.

# Conclusion

I present a multi-party eltoo scheme with desirable functional properties but exponential computational complexity using one known technique and one likely known technique.
What I'm calling a "multi-transaction-signature" permits a single signature to commit to multiple possible transactions representing state transitions.
I suspect it's a trivial variant of vector commitments too uninteresting to have a name.
This scheme combines these "multi-transaction-signature"s with existing eltoo "floating transactions" to enable a communication-efficient multiparty eltoo channel with or without penalty, and with bounded settlement time.
Parties are given one chance to update the on-chain state, and may (optionally) be penalized for publishing old state, which should make attacks economically unattractive.
The amount of computation required of parties is exponential on the number of parties N, however I believe that N~=10 is practical in this scheme, and also nearing the practical limit of multi-party channels anyway, due to the liveness requirements of participants, and network latency.
I present this scheme mostly to solicit feedback, is this interesting enough to deserve a better writeup? this is obviously very incompletel.
Is it interesting enough to warrant an implementation?
Maybe it is only rehashing previous work, if please direct me to it so I can familiarize myself.
Maybe, (my hope) is it inspires one of you wizards to invent a better or related scheme.

# Further Work

- Can this be safely integrated with a watchtower?
- Truncating the number of generations at some number `M < N` could potentially drastically reduce the number of transactions to calculate and hash, as long as there are less malicious users than M it should remain secure.
- Can this scheme be adapted to tolerate offline users?
  Each generation of update transaction already partitions the parties into two sets, it *might* be possible to use something like this to split and merge a multi party pool if parties also sign for a set of merge transactions?
  *Maybe?*. It's worth exploring but there's a lot of pitfalls, there might not be a (good) way to do it.
- Since state update communications are reduced to a single signature per party, is it practical to use musig instead, to reduce the witness size of uncooperative closes?
  That would require two rounds of communication, however it could still be practical.
- Is this interesting enough to implement? Maybe just to benchmark state update performance to gauge practical limits.

Copied from here: https://gist.github.com/Ademan/4a14614fa850511d63a5b2a9b5104cb7

I've started on a proof of concept implementation here, it is *really* rough and I'm only sharing it so other people can get benchmark numbers on their machines, there's probably many bugs: https://github.com/Ademan/multi-party-eltoo-with-bounded-settlement

As could be expected from an exponentially growing complexity, this protocol maxes out fairly quickly, I'm using a 20ms cutoff per an anonymous suggestion, however I think 10-12 parties should be pretty possible on similar hardware. Everything about the current version is very naive and could be improved, probably not enough to hit 16 parties though.

On my 16 thread Ryzen 7 5700U (mobile processor) here's a graph of time taken to recalculate state vs number of parties. It *is* already using multiple cores via rayon, but only in one place and in a very naive way.

![2025-01-04-154303_1064x616_scrot|690x399](upload://AuChp0IrpWLzhmzw0B5D9i2fBft.png)

Aside from general comments I'd like to know how many parties we're really shooting for in a multi-party channel. Is this a dead end, or somewhat interesting?

-------------------------

instagibbs | 2025-01-06 18:07:39 UTC | #2

[quote="ademan, post:1, topic:1364"]
By giving parties only one chance to update the channel state on-chain, this scheme drastically reduces the ability of malicious parties to prevent honest settlement of the channel.
[/quote]

I think this is a tradeoff when we cannot employ trustless watchtowers without adding them directly to the signing quorum.

Ideally we could have N counterparties (who are consensus quorum for state updates), and then have `M > N` slots for state submission. Obviously each additional N costs both in vbytes and in relative delay requirements, so I don't think there are any "right" answers here.

-------------------------

ariard | 2025-01-07 01:07:20 UTC | #3

About the idea of “punisheable Eltoo” in the context of multi-party setting, this has been already explored a bit in the past by Lloyd Fournier and also myself. It’s not exactly the same concept than you’re proposing if I’m understanding what you’re trying to achieve, though here few links if this can be of interest for your research:
- “[Using Per-Update Credential to enable Eltoo-Penalty](https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2019-July/002064.txt)”
- “[Witness Asymmetric Channel](https://github.com/LLFourn/witness-asymmetric-channel)”

There is also this post too on the challenges of multi-party construction, whatever the security model, in a pure trustless design (no coordinator, no quorum) though interesting for your current research.

- “[Conjectures on solving the high interactivity issue in payment pools and channel factories](https://gnusha.org/pi/bitcoindev/CALZpt+GNoRdbtfeBtpnitiAZ4jGwSsRpSRXOyX7mzrhwewmBcw@mail.gmail.com/)”

For the question about “how many parties we’re really shooting for in a multi-party channel”. For the CoinPool paper, we went with 1000-sized pool to get 7.9 billion of human beings each with its own pool balance. See the assumptions we were making on the bitcoin blockchain (no consensus change in validation ressources — all other things equals apart covenants) in this post.

- “[A Dive into CoinPool: Bitcoin Balances for Billions](https://gnusha.org/pi/bitcoindev/CALZpt+E+eKKtOXd-8A6oThw-1i5cJ4h12TuG8tnWswqbfAnRdA@mail.gmail.com/)”.

From a brief read of your research, the hard limitation I can see is not actually the one chance to update the channel state on-chain (though how do you assign blame correctly among N non-trusted counterparties in an efficient fashion is an interesting problem ?), but rather the exponential growth in the vector commitment with the number of counterparties participating in the off-chain construction.

-------------------------

cdecker | 2025-01-07 09:18:59 UTC | #4

I might be missing the point here, but how can an attacker delay the finalization of the channel? For any update the attacker may drop we have a response that is not encumbered by a timelock or CSV, thus all the transactions can end up in a single block. I always thought of it as this: attacker publishes an old update, the victim takes its latest update, creates two versions: a) bound to the funding and b) bound to the update the attacker broadcast.

We have several outcomes:

 1) the attacker succeeds in confirming its old update, in which case we have already submitted the response, and it might have been confirmed along with the first, i.e., the attacker just lost some funds, but did not succeed in claiming or delaying anything, since our own transactions were just as valid before as after the attacker published them.
 2. Our latest update is confirmed, the attacker doesn't lose anything, but he also doesn't gain anything.

This game can be iterated (unless the attacker can fill the block on its own he doesn't have an advantage) and we just create and broadcast a matching response to anything the attacker does.

A variant of this that I could see is if a miner and the attacker collude to attack silently, not giving the victim time to response, forcing them into the next block, but unless the attacker has a majority and the timeouts are chosen correctly, the attacker cannot censor indefinitely. It's also worth noting that in this case no system using timelocks or CSV can be safe anyway.

-------------------------

ajtowns | 2025-01-07 11:00:47 UTC | #5

[quote="cdecker, post:4, topic:1364"]
I might be missing the point here, but how can an attacker delay the finalization of the channel?
[/quote]

If it's easy for the attacker to send transactions to almost all miners without you seeing them, then they can publish state K, then broadcast state K+1, K+2, K+3 at each new block height at a higher feerate than your proposed update to state N. If you saw their tx prior to it being mined, you could just update your state N tx to spend K+i+1 instead of K+i, but only if you see their tx. Blocking you from seeing their txs in the mempool is probably not too difficult if they can directly connect to your bitcoin node.

Hard to do if you have a working watchtower arrangement, even one that's merely independently watching the mempool and sending you unconfirmed txs it thinks you might be interested in.

-------------------------

cdecker | 2025-01-07 11:57:23 UTC | #6

Makes sense, thanks AJ for the explanation.

I wonder however if it is worthwhile to even consider cases in which an attacker can keep things published but hidden from a specific victim, as that is quite the rabbithole, and I'm not sure it is that easy to pull off.

As you mention yourself the threat of a watchtower should be enough to keep the chances of succeeding to a minumum. Notice that if the hidden TX is discovered before being mined by the victim, then they can queue a reaction for the same block, forcing the attacker to add more funds to retry with a new commitment, hence why I talk about game theoretic threats despite there not being a direct penalty for attempting to cheat.

The added complexity for penalizing and/or forcing a sequence may outweigh the benefit.

-------------------------

instagibbs | 2025-01-07 14:20:15 UTC | #7

Remember the attacker is paying for each of these stalls, similar to a replacement cycling attack, but bound to one bid per block if you have no mempool knowledge.

I've always found this attack pretty weak personally since the attack can be driven to negative EV pretty easily, at relatively minor cost to the defender, even being mempool-blind

-------------------------

