# OP_EXPIRE: Mitigating replacing cycling attacks

mpch | 2024-11-27 11:36:53 UTC | #1

Hello all :wave:,

I'm currently working towards prototypes and implementations to fix replacement cycling attacks.

# **Why is this important?**

As the current state of the lightning network, malicious actors have a capacity to bribe miners on channel resolutions, consequently, have a capacity to steal funds from peers.

# **What are Replacement Cycling attacks?**

Replacement cycling attack is a method that makes use of transaction input replacements and HTLCs scripts conditions, to make forcefully stuck HTLCs not be confirmed until CLTV delta from the HTLC on a previous (malicious) hop timeout. After the timeout, as the attacker has the preimage, spend the funds from the stuck payment, stealing the payment that went through the HTLC.

In other words, as Peter Todd points out in the mailing list [[link]](https://gnusha.org/pi/bitcoindev/ZTMWrJ6DjxtslJBn@petertodd.org/) , the problem here is that after the HTLC-timeout path becomes spendable, the HTLC-preimage path remains spendable.

We need to find solutions to this issue.

# **Proposed and implemented mitigations**

Antonie Riard in his paper proposed five mitigations related to mempool monitoring and how the HTLC-timeout transaction is rebroadcasted [[link]](https://gnusha.org/pi/bitcoindev/CALZpt+GdyfDotdhrrVkjTALg5DbxJyiS8ruO2S7Ggmi9Ra5B9g@mail.gmail.com/): Local Mempool and transaction relay traffic monitoring, miner mempool monitoring, defensive fee broadcasting, aggressive broadcasting and per hop packet delay bumping. All of them, attempt to mitigate the issue doing either:

* Trying to win the race
* Trying to make the race not worthwhile.

Some of them, local-mempool and aggressive rebroadcasting, were implemented by the different node implementations to fix this issue.

# **Why we don't want races in Lightning Protocol**

Every time there is a dispute or want to *end a relationship*, nodes go on-chain. Both cases depend on what the contract between the parties says. I think anyone can see a problem with the "whoever comes first, was right" type of contract. Incentivises cheating. Also makes it so whoever has more capital (therefore, better machine or connection) will probably end up winning.

better contracts == better infrastructure

**OP_EXPIRE enters the scene.**

My goal is to build a prototype demonstrating the usefulness of this opcode to the lightning protocol, through a proposed patch to bitcoin-core and the required updates to the lightning spec and CLN, to demonstrate how the softfork proposal is useful for resolving HTLC timeouts off chain.

OP_EXPIRE, AKA, OP_ANTICHECKLOCKTIMEVERIFY lets us build better contracts so node implementations don’t need to go on-chain to be sure that funds are safe.

As the name says, it lets a branch or the script expire after certain block or delta blocks. This means that the HTLC-preimage branch would expire after a delta, previously negotiated, amount of blocks, making it an urgent thing. Now after a certain amount of block an HTLC is not returned, the channel wouldn’t need to be closed, implementation would need to make the HTLC-preimage become not spendable to return the error back to the network.

If someone has a problem, it wouldn't mean chaos on-chain given automatic unilateral closures.

If you see your peer down and you want to really want to close the channel, you would manually (or programmatically) take action and not be part of the protocol.

We can see the changes on the script here (extracted from Elle Mouton blog [[link]](https://ellemouton.com/posts/htlc-deep-dive/)):

**Current offering HTLC**
```
# To remote node with revocation key
OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
OP_IF
    OP_CHECKSIG
OP_ELSE
    <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
    OP_NOTIF
     # To local node via HTLC-timeout transaction (timelocked). OP_DROP 2                           OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
    OP_ELSE
           # To remote node with preimage
           <expire block height> OP_EXPIRE OP_DROP OP_HASH160
           <RIPEMD160(payment_hash)> OP_EQUALVERIFY OP_CHECKSIG
    OP_ENDIF
OP_ENDIF
```


**New offering HTLC**

When going to remote or pre image, after specified blockheight the script would fail, leaving only the first two branches as possible way to unlock the funds.

```
# To remote node with revocation key
OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
OP_IF
    OP_CHECKSIG
OP_ELSE
    <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
    OP_NOTIF
     # To local node via HTLC-timeout transaction (timelocked). OP_DROP 2                           OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
    OP_ELSE
           # To remote node with preimage that expires on expire block height.                                    <expire block height> OP_EXPIRE OP_DROP OP_HASH160
           <RIPEMD160(payment_hash)> OP_EQUALVERIFY OP_CHECKSIG
    OP_ENDIF
OP_ENDIF
```

**How could it be implemented?**

Peter Todd suggested a few ways of implementing this solution [[link]](https://gnusha.org/pi/bitcoindev/ZTMWrJ6DjxtslJBn@petertodd.org/) [[link]](https://www.youtube.com/watch?v=HXnNRmhCpNM):

1. Coinbase Bit: modify the nVersion to enable transactions to have tx outputs to be spendable like the coinbase transaction output.
2. Delta encoding expiration: encodes the a delta expiration in the nVersion that applied to the nLockTime, acts like a new factor “AbsoluteExpireHeight” to be checked as the OP_CLTV.
3. Taproot Annex: use the annex as nExpiryHeight that would work as the same of the “AbsoluteExpireHeight”.

I took the liberty to implement the second one.

All transactions using this op_code would need to use nLockTime greater than 0 and do not use sequence_final.

We take the first 16 bytes of the nVersion field of the transaction (now nDeltaExpireHeight), this would signal the maximum delta of all OP_EXPIRE in scripts inside the transaction from the nLockTime field.

So the actual script takes the first element of the stack and compares it to the AbsoluteExpireHeight (nDeltaExpireHeight + nLockTime).

The script would fail then in the following cases:

1. The stack is empty
2. The top item on the stack is less or equal than 0
3. The lock time type of the stack item is not block height
4. The lock time type of the nLockTime is not height
5. The top stack item is greater than AbsoluteExpireHeight
6. the nSequence field of the txin is sequence_final

[**You can see a draft implementation in bitcoin core here**](https://github.com/bitcoin/bitcoin/commit/7b3cc52e55e7f942bed8afe73ee33ac652b1c3ce)

[Branch with test from Antoine Riard and modified version of this HTLC
](https://github.com/a-mpch/bitcoin/commits/2024-11-op-expire/)

## Summary

Having OP_EXPIRE as a possibility in Bitcoin Script enable us:
- To solve Lightning open problem of replacement cycling attacks
- Have a less automated unilateral closure of channels, less footprint in base chain
- Better scripts for other projects: Auction bids, Vaults, Time-limited delegation (rough list from Peter Todd's presentation)
- We don't introduce a that different script as it is more-or-less a negation of OP_CLTV

Possible cons:
- We need would need to find consensus on what's the nExpireHeight field would be. Version? Annex? That's a long conversation to have. For now Version and have a delta works best. 

*Side notes:*

*Next steps are doing a CLN draft implementation making use of this HTLC script.*

*This is a extract of a [blog post](https://world.hey.com/mpch/mitigating-replacement-cycling-attacks-with-some-magic-part-i-4a76ad45) so this post would be concise,  If you need more details, graphical example of the problem or explanation of possible mitigations, you can checkout the blog itself.*

-------------------------

instagibbs | 2024-11-27 16:38:24 UTC | #2

OP_EXPIRE adds brand new behavior to Bitcoin that you can not emulate otherwise, so I think you're going to have a very tough time pushing it forward.

If it's specifically for replacement cycling, I'd suggest efforts are better spent understanding the problem first. We may have much lighter-weight solutions that don't require consensus changes.

AFAIK there is no working replacement cycling scenario for warnet, so I'd start there? I've been hoping someone would do that since the issue was first covertly disclosed but no one seems to bother?

-------------------------

