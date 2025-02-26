# Great Consensus Cleanup Revival

AntoineP | 2024-05-16 11:05:57 UTC | #1

I've been working on revisiting Matt Corallo's [Great Consensus Cleanup proposal](https://github.com/TheBlueMatt/bips/blob/7f9670b643b7c943a0cc6d2197d3eabe661050c2/bip-XXXX.mediawiki). I was interested in figuring:
1. How bad the bugs actually are;
2. How much the proposed fixes improve the worst case;
3. Whether we can do better now we've had 5 more years of experience;
4. Whether there is anything else which would be worth fixing.

**TL;DR**: i think it's bad. The worst case block validation time is concerning. I also believe it's more important to fix the timewarp vulnerability than people usually think. Finally i think we can include a fix to avoid performing BIP30 validation after block 1,983,702 as well as an additional limitation on the maximum size of legacy transactions would provide a desirable safety margin with regard to block validation time.

With this post i intend to kick off a discussion about the protocol bugs mitigated by the Great Consensus Cleanup proposal. I'd like to gather comments and opinions about each of the mitigations proposed, as well as potential suggestions for more fixes. I will go through each of the bugs one by one. The section about the block validation time was significantly redacted: i'm sharing the numbers publicly but the details of the strategies to come up with a bad block will only be shared with regular Bitcoin Core contributors and a few other Bitcoin protocol developers.

## Timewarp

The timewarp vulnerability exploits how difficulty adjustment periods don't overlap. Miners can take advantage of this by setting the timestamp of the last block in a retarget period as much in the future as possible (now + 2h) while holding back the timestamps of all the other blocks to artificially reduce the difficulty. See [this stackexchange answer](https://bitcoin.stackexchange.com/a/75834/101498) for a more detailed explanation.

### How bad is it?

It's interesting to consider both how it worsens the situation and what it enables concretely. After all, miners can always hold back the timestamps even without exploiting the timewarp vulnerability.  But without setting the timestamp of the first block of a period before the timestamp of the last block of the preceding period, taking advantage of this necessarily involves bumping the difficulty back up. On the other hand by setting the timestamp of the first block of a period below the one of the preceding period's last block an attacker can continuously take advantage of the low difficulty while continuing to reduce it.

In practice an attacker could fatally hurt the network within a bit over a month of starting the attack. By starting at time `t` and period `N`, the attacker can already halve the difficulty at the end of period `N+1`. Which allows him to mine period `N+2` in a single week, further reducing the difficulty by a 2.5x factor. Etc.. Within less than 40 days the attacker would bring the difficulty down to 1, letting him mine millions of blocks. Besides claiming all the remaining subsidy, this would destroy the security of any L2 protocol relying on timelocks as well as exacerbate DoS vectors (for instance if this is combined with a spam of the UTxO set).

This disregards the MTP rule which would anyways at best be a small annoyance to the attacker.  However this begs the question: although the attack requires a majority hashrate, is it a winning strategy for a minority hashrate to try to opportunistically exploit this? That is, if mining the last block of a retarget period set the timestamp as much in the future as possible and if mining any other block set the timestamp as far back as the MTP rule will let you. Technically it comes at no (measurable) cost to the miner, but it *might* give him (and other miners) a slight increase in block reward at the expense of future miners. Turns out the marginal gain by adopting this strategy is ridiculously small for any minority hashrate therefore we can reasonably expect miners to not try to opportunistically exploit timewarp.

### Should we really fix it?

Some have argued miners would not purposefully shoot themselves in the foot. In fact, it's not realistic to expect they would kill the chain. Instead they would likely settle on an equilibrium of increasing the frequency of blocks by X%. I'll let readers draw their own conclusion with regard to the political implications of miners being able to increase the available block space without a change in nodes' consensus rules. Let's point out however that a cartel of miners forming to, even if only slightly, decrease the difficulty by exploiting the timewarp vulnerability would put the network constantly on the brink of getting taken down in a few weeks.

Another common arguments is how it's less of a priority to fix because the attack would be obvious and take time. I'm fairly sceptical of any argument which involves users coordinating and deciding of the validity of a block at any given height independently of the current consensus rules.  Besides, a month isn't a lot of time to coordinate and change Bitcoin's consensus rules. Further, as mentioned previously it's not even clear there would be widespread consensus for doing so. Users like lower fees and miners like more subsidy. Given the current distribution of control over hashrate and the exponentially decreasing block subsidy, it's not unreasonable to think a cartel could form to exploit the timewarp vulnerability.

Finally, some have argued it's less of a priority because the attack requires a majority of the hashpower anyways. I believe it's severely lacking nuance. Exploiting the timewarp vulnerability significantly increases the harm which a 51% attacker can do. He can normally "only" temporarily censor transactions. With timewarp he can potentially ruin the network.

### Can we come up with a better fix?

Probably not. The fix is straightforward: make the retarget periods overlap. Matt's proposed change is the most simple and obvious way of achieving this: constrain the timestamp of the first block in a period compared to the timestamp of the last block of the preceding period.


## Worst case block validation time

It's well known maliciously crafted non-Segwit transactions can be pretty expensive to validate.  Large block validation times could give attacking miners an unfair advantage, hinder block propagation (and its uniformity) across the network or even have detrimental consequences on software relying on block availability. To this effect the Great Consensus Cleanup proposal includes a number of additional constraints on legacy Script usage.

### How bad is it?

It's bad. The worst block i could come up with takes around 3 minutes to validate with all 16 cores of my modern laptop's CPU and a hour and a half of a RPi4. For obvious reasons i've redacted here the details of such block, as well as the various approaches to create similarly expensive-to-validate blocks. I'll share them in a semi-private companion post to other protocol developers using the private working group feature of Delving. If you think you should be in this working group and i forgot to add you, let me know.

[*REDACTED #1*](https://delvingbitcoin.org/t/worst-block-validation-time-inquiry/711/3?u=antoinep#redacted-block-1-1)

### How much does the proposal improve the worst case? Can we come up with more effective mitigations?

The mitigation proposed in the Great Consensus Cleanup makes the block i came up with in the previous section invalid. The worst block under the new constraints takes 5 seconds to validate on my laptop. I believe we could further introduce a limitation on the size of legacy transactions to be on the safe side.

Some confiscation concerns were raised about the proposed mitigations. I believe those concerns are reasonable and could be addressed by only applying the new rules when checking the script for an output created after a certain block height.

[*REDACTED #2*](https://delvingbitcoin.org/t/worst-block-validation-time-inquiry/711/3?u=antoinep#redacted-block-2-1)


## Merkle tree attacks using 64 bytes transactions

There are two (known) remaining attacks with how the merkle root in Bitcoin blocks is computed. Both involve creating a catenation of two 32 bytes hashes which successfully deserializes as a Bitcoin transaction. One (probably the most famous) is about "going down" the merkle tree by tricking a light client into accepting as payment a transaction which was not in fact committed into a block: a 64 bytes transaction is committed to the block whose last 32 bytes correspond to the txid of a non-committed transaction paying the victim. The other one "goes up" the merkle tree, by tricking a node into treating a valid block as permanently invalid: find a row of tree nodes which all deserialize as (invalid) 64-bytes transactions. For more details see [this writeup](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2019-February/016697.html) by Suhas Daftuar.

The Great Consensus Cleanup proposes to make 64-bytes-or-less transactions invalid altogether, fixing both vulnerabilities.

### How bad is it?

The attack against light clients (or anything which may accept a merkle proof, like a sidechain) requires to bruteforce between 61 and ~75 bits, depending on the amount of bitcoins dedicated to the attack. This is expensive, and simple mitigations exist. For instance checking the depth of the tree, which gives you the number of transactions in the block, by asking for a merkle proof of the coinbase transaction).

That said, the attack [was estimated to cost around $1M](https://bitslog.com/2018/06/09/leaf-node-weakness-in-bitcoin-merkle-tree-design/) when "state-of-the-art Bitcoin [ASICS] reached 14 TH/s". Nowadays, [looks like](https://whatsminer-microbt.com/product/whatsminer-m63s/) top ASICs reach 400TH/s. In addition, this attack allows to fake an arbitrary number of confirmations for the transaction. And the cost to simply mine a single fake block to fool an SPV client, which doesn't, is now higher (80 bits).

The attack to fork Bitcoin nodes is mitigated in Bitcoin Core for invalid transactions by not caching contextless block checks (`CheckBlock()`). Creating valid transactions is not practical: the first one must be a coinbase transaction, which requires bruteforcing 224 bits.

64 bytes transactions, due to how the merkle root in blocks is computed, is a core weakness in Bitcoin.  Although both (known) attacks it enables can be mitigated it would be nice to avoid this footgun as well as being able to cache contextless block checks in Bitcoin Core.

### Can we come up with a better fix?

64 bytes transactions cannot be "secure", as in 64 bytes in a transaction is not enough to have an output whose locking script won't be anyone-can-spend or lock the coins forever. They have no known use and have been non-standard for half a decade. Given the vulnerabilities they introduce and their lack of use, it is more than reasonable to make them invalid.

However the BIP proposes to also make less-than-64-bytes transactions invalid. Although they are of no (or little) use, such transactions are not harmful. I believe considering a type of transaction useless is not sufficient motivation for making them invalid through a soft fork.

Making (exactly) 64 bytes long transactions invalid is also what AJ implemented in [his pull request to Bitcoin-inquisition](https://github.com/bitcoin-inquisition/bitcoin/pull/24).

## Wishlist

### BIP30 verification

BIP34 temporarily made it possible to avoid the relatively expensive BIP30 check on every block connection. Starting at block height 1,983,702 it won't be possible to rely solely on BIP34 anymore.  If a soft fork is proposed to cleanup long standing protocol bugs it would be nice if it made coinbase transactions unique once and for all.

A neat fix would be to simply requires the `nLockTime` field of the coinbase transaction to be set to the height of the block being created. However there is a more roundabout fix which is potentially easier for miners to deploy: make the witness commitment mandatory in all coinbase transactions (h/t Greg Sanders). I have yet to check if there is any violation of this but i'd be surprised if there is a pre-Segwit coinbase transaction with an output pushing exactly the witness commitment header followed by 32 `0x00` bytes.

### Your favorite bug!

At this stage i'm interested in gathering as many suggestions for cleanups as i can, to make sure that if such a soft fork were to be proposed, we would have carefully analyzed all the fixes we'd like to include.

Of course, suggestions need to be reasonable to be considered. For instance, "banning ordinals" is an uninteresting suggestion and i doubt many would engage with it. In addition, let's keep suggestions focused on long-standing uncontroversial bugs. For instance "a block which takes 30 minutes to validate is bad", "the broken merkle tree calculation is creating footguns" or "let's make coinbase transactions *actually* unique" seem fairly uncontroversial. On the other hand while reasonable arguments for "let's decrease the block size limit" can be put forth, it seems much more controversial to me.

For instance here is a couple things i wasn't convinced were worth proposing:
- Requiring standard SIGHASH type bytes for Segwit v0 too;
- Limiting the maximum size of scriptPubKey's to reduce the worst case UTxO set growth.

-------------------------

1440000bytes | 2024-03-24 23:52:07 UTC | #2

[quote="AntoineP, post:1, topic:710"]
It’s bad. The worst block i could come up with takes around 3 minutes to validate with all 16 cores of my modern laptop’s CPU and a hour and a half of a RPi4. For obvious reasons i’ve redacted here the details of such block, as well as the various approaches to create similarly expensive-to-validate blocks. I’ll share them in a semi-private companion post to other protocol developers using the private working group feature of Delving. If you think you should be in this working group and i forgot to add you, let me know.
[/quote]

- It is the worst of all the attacks described in this post and I am surprised no state sponsored miners have tried to solo mine such blocks.

- Example for one such blocks and approach is already public.

[quote="AntoineP, post:1, topic:710"]
Some confiscation concerns were raised about the proposed mitigations. I believe those concerns are reasonable and could be addressed by only applying the new rules when checking the script for an output created after a certain block height.
[/quote]

This makes sense. 

[quote="AntoineP, post:1, topic:710"]
At this stage i’m interested in gathering as many suggestions for cleanups as i can, to make sure that if such a soft fork were to be proposed, we would have carefully analyzed all the fixes we’d like to include.
[/quote]

I think SIGHASH_SINGLE bug reported in 2012 should also be fixed with other bugs: https://www.mail-archive.com/bitcoin-development@lists.sourceforge.net/msg01408.html

-------------------------

sjors | 2024-03-25 14:35:18 UTC | #3

Thanks for the write-up! Parts of the BIP itself could also benefit from a more clear description, to better describe what the consensus changes are.

I agree that the changes here should be kept both simple and as uncontroversial as possible, because otherwise it slows down the process exponentially.

One potential way to mitigate the harm of slow validation, without / before a soft fork, is for nodes to validate competing tips in parallel. The node keeps an eye on alternative headers while block validation is in progress. If a second header arrives at the same height, it would validate it in parallel. Whichever candidate is validated first is announced to the network. The slower side would be fully validated (`valid-fork` in the `getchaintips` RPC of Bitcoin Core), so it can be switched to very quickly if needed.

This is only a small deviation from the current logic of preferring the first _seen_ header. But implementation wise it may not be easy.

If regular node runners and pools run that, it strongly increases the probability for the attacker block to go stale. But only if other pools actually _try_ to produce a competing block. That happens automatically if they _don't_ SPV mine.

Pools that _do_ SPV mine (in the period between seeing a header and validating the full block) could use some heuristic to decide the most profitable strategy here, taking into account how much time _other_ pools likely need to validate the block.

But without a soft fork I'm worried that if an attack takes places, especially if people see it coming, there's an incentive for pool operators to hop on a phone call and coordinate what to do about it. Such phone calls are not healthy in the long run.

-------------------------

AntoineP | 2024-03-26 23:31:32 UTC | #4

To expand on the coinbase uniqueness fix. Technically, only coinbase transactions before height 227,931 may be duplicated. Among these, only those whose first element is a minimal `CScriptNum` push of a height > 227,931 can effectively be [0]. Therefore an argument can be made in favour of reducing the risk of for miners to a minimum by only requiring a witness commitment at the heights which were committed to in these early coinbase transactions.

I don't think it is worth the complexity. This would need a mechanism to keep track of those heights. And i think the risk of implementing this mechanism outweights the risk of a miner mining an invalid block 21 years from now because he didn't update his software. (21 years because this is not needed before after block height 1,983,702.)

Still, it's interesting to see what future blocks could have a duplicate coinbase. To this end i've scanned the 227,931 first blocks of the chain. I'm counting 189,023 coinbase transactions which *could* (this disregards [0]) be duplicated without violating BIP34. I count 10 before block 5,000,000 (about 100 years from now), 24 before block 10,000,000 and 80 before block 100,000,000.

Here is the 24 ones before block 10,000,000:

| Original block height | duplicable block height |
| ----------------------------| ------------------------------ |
| 147,396 | 8,624,845 |
| 149,732 | 8,631,390 |
| 149,813 | 8,631,629 |
| 149,838 | 8,631,693 |
| 150,283 | 8,632,995 |
| 151,491 | 8,636,474 |
| 152,374 | 8,638,716 |
| 152,599 | 8,639,231 |
| 153,662 | 8,641,929 |
| 164,384 | 1,983,702 |
| 169,895 | 3,708,179 |
| 170,307 | 3,709,183 |
| 171,896 | 3,712,990 |
| 172,069 | 3,713,413 |
| 172,357 | 3,714,082 |
| 172,428 | 3,714,265 |
| 174,151 | 5,208,854 |
| 176,684 | 490,897 |
| 177,628 | 9,558,101 |
| 183,669 | 3,761,471 |
| 196,988 | 4,275,806 |
| 201,577 | 5,327,833 |
| 206,039 | 7,299,941 |
| 206,354 | 7,299,941 |

For the curious, [here](https://gist.github.com/darosior/3a5ac0a8d935fa9d3e90310590ca6699) is the full list of all the potential future violations.

---

[0] And even then only those which weren't spent in a transaction which also indirectly spends a non-duplicable coinbase can in practice lead to a BIP30 violation.

-------------------------

benthecarman | 2024-03-28 03:21:12 UTC | #5

> make the witness commitment mandatory in all coinbase transactions

Wouldn't this not work for the occasional empty block?

-------------------------

ajtowns | 2024-03-28 06:04:45 UTC | #6

[quote="benthecarman, post:5, topic:710, full:true"]
Wouldn't this not work for the occasional empty block?
[/quote]

An empty block can only claim the subsidy as the reward, which won't equal the ~25 or ~50  BTC reward that all the potential duplicates claimed, so it wouldn't be a duplicate in any event, even if they were given an exemption.

But an exemption isn't necessary: empty blocks can include a witness commitment, it'll just be constant (as the coinbase tx's wtxid is replaced by `uint256{0}`, eg https://mempool.space/signet/tx/d62676348ef1dfa05f8c5380bcccb1dc8c1bea4877e50495d577fd4f4c2f09e5 (a block with only the coinbase, the coinbase has the usual `uint256{0}` "witness reserved value" as its witness data, it has the witness commitment of `aa21a9ed e2f61c3f71d1defd3fa999dfa36953755c690689799962b48bebd836974e8cf9` followed by the signet block signature. You can see other empty blocks have the same witness commitment, eg https://mempool.space/signet/tx/cdf494066c05d46aa0ba0b9c2eab872d3a343f22f8b7a011c24a05e0ebc2d84b

EDIT to add: all the coinbases from the list in comment 4 have an nLocktime of 0 and an nVersion of 1, so requiring future blocks to have some different value for one or the other would also result in uniqueness.

-------------------------

ajtowns | 2024-04-05 02:30:30 UTC | #7

[quote="AntoineP, post:1, topic:710"]
Limiting the maximum size of scriptPubKey’s to reduce the worst case UTxO set growth.
[/quote]

I don't think that does reduce the worst case UTXO set growth? Creating a utxo costs 8 bytes for the value, and X+1 or X+3 bytes for the script; but storing a utxo costs an additional 4 bytes beyond that to record the coinbase flag and block height. So given a choice between spending 64 vbytes creating two p2sh outputs or spending 64 vbytes to create one output with a 55 byte scriptPubKey, then the two outputs uses up more space in the utxo set. Then there's also whatever indexes the database uses to make lookups by txid fast.

If anything, the better reason to limit scriptPubKey sizes seems to be that (a) with p2sh-style approaches large scriptPubKeys are unnecessary, and (b) it moves validation cost to where validation actually occurs. That is, if you have a large/complex spending constraint, if you put that in a large scriptPubKey, you can then redeem many utxos that use that constraint in a single block: in that case the cost you pay is spread out over many blocks (where the scriptPubKey contributes to sigop counts and weight limit), but the cost incurred by validating nodes is localised to the spending block (when hashing and signature checking actually occurs).

I think limiting scriptPubKeys to 105 bytes would cover all the existing standard cases (105 bytes for k-of-3 bare multisig, ~80 bytes for op_return), and provide plenty of room for larger p2sh-style hashes (eg ~50 bytes for taproot-style commitments with a bls12-381 curve or ~66 bytes for a sha512 hash).

-------------------------

ajtowns | 2024-04-05 03:26:53 UTC | #8

[quote="1440000bytes, post:2, topic:710"]
I think SIGHASH_SINGLE bug reported in 2012 should also be fixed with other bugs: [[Bitcoin-development] Warning to rawtx creators: bug in SIGHASH_SINGLE ](https://www.mail-archive.com/bitcoin-development@lists.sourceforge.net/msg01408.html)
[/quote]

This insecure use of SIGHASH_SINGLE is useful for cheaply spending insecure low-value/dust utxos to miner fees -- the [idea](https://bitcointalk.org/index.php?topic=1118704.msg11864913#msg11864913) is that you combine many inputs with only one output, use SIGHASH_SINGLE for inputs after the first so that you get cheap hashing, and for the signature R-value you use G/2 which has a short encoding, allowing you to clean up more dust in the same number of bytes. Because the utxos are already insecure, revealing the utxo's private key by spending with a known R-value is fine, and because you don't care about revealing the private key, having a signature that can be reused for other utxos with the same pubkey is also fine.

To some extent this can also be used as a "release" key; if you script is `<Pn> CHECKSIGVERIFY <T> CHECKSIG` then the `T` can publish a single signature that can be reused by `P0`, `P1`, etc who have locked different utxos against similar scripts (the `Pn` signature would be an ordinary `SIGHASH_ALL` here). That's potentially more convenient than `NONE|ANYONECANPAY` as it doesn't commit to the utxo being spent; provided you don't mind having extra inputs so that there's no output corresponding to this utxo's input.

So personally I think it's fine to leave this as-is, and just recommend people use p2wpkh, p2wsh or p2tr instead to avoid this weird behaviour unless they actually want it.

-------------------------

kcalvinalvin | 2024-04-05 04:38:11 UTC | #9

I'm not quite sure how painful it's for miners now since I've not been keeping up but the nonce field could be bigger than 4 bytes. That'd ease up things for miners. 8 bytes should be enough right? Maybe someone's done the math for this.

-------------------------

sjors | 2024-04-05 09:18:36 UTC | #10

It would be, but that's a hard-fork - that'll have to wait for 2106 :slight_smile:

[BIP 320](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0320.mediawiki) makes some more header bits available for nonce-like grinding. I'm not sure how widely used it is, and IIUC Stratum v2 takes advantage of it. In any case that doesn't require a (soft) fork.

-------------------------

recent798 | 2024-04-05 10:23:22 UTC | #11

[quote="AntoineP, post:4, topic:710"]
For the curious, [here ](https://gist.github.com/darosior/3a5ac0a8d935fa9d3e90310590ca6699) is the full list of all the potential future violations.
[/quote]

How many of those coinbase outputs are spent along with another coinbase output that stems from a coinbase transaction that can not be duplicated? If all are spent in this fashion, then a future soft fork may not be needed? The expensive check could be skipped, if on a known-good headers chain, and otherwise enabled.

For reference: https://github.com/bitcoin/bitcoin/pull/12204/files

The code comment in the reference also mentions that a soft-fork may be preferred, which "*retroactively* prevents block [...] from creating a duplicate coinbase". So requiring a witness commitment does not seem appropriate, because it can not be enforced on past blocks. My understanding is that a soft-fork that can not be applied from genesis won't help to skip the expensive check in case of a massive (theoretical) reorg, so fallback code will be needed for this case. If fallback code is needed anyway, then it seems better to do it without a consensus deployment.

-------------------------

AntoineP | 2024-04-05 15:37:24 UTC | #12

[quote="ajtowns, post:7, topic:710"]
I don’t think that does reduce the worst case UTXO set growth?
[/quote]

Indeed. It was a late (and not though-through) addition to the examples of things i don't think meet the bar to be "fixed" as part of this effort. Thanks for the correction, this all makes sense.

[quote="ajtowns, post:7, topic:710"]
moves validation cost to where validation actually occurs
[/quote]

Yes. With regard to validation costs i think we should instead make sigops of prevouts with bare scripts count toward the budget of the block it's spent in. It'd make them be counted twice but:
- we can't "uncount" the ones at creation time without a hard fork;
- better to double count them than to not count them at all where it matters;
- it should not affect any real world usage.

If we are worried about the last point we could, as you suggested to me elsewhere, discount 3 sigops per prevout to account for the current standard usage. But i don't think it's necessary, the size limit is hit before the sigop limit for any non-pathological use of legacy (or any) script. (From some napkin maths the less pathological legacy transaction which would hit the sigop limit is a (>773 kB) transaction which spends at least 6667 1-of-3 bare multisigs to a p2sh output.)

[quote="ajtowns, post:8, topic:710"]
This insecure use of SIGHASH_SINGLE is useful
[/quote]

Yes, it also seems worthless to fix on its own. The author didn't take the time to suggest a "fix", but changing this behaviour (besides just disabling it) would be a hard fork. The obvious backward compatible alternative is to have a new script version application developers can opt in to use. Good news: it exists, it's Taproot. Furthermore i don't think anybody reported having an issue with this since it was first discussed more than a decade ago. In fact it is so much not worth fixing that even Segwit v0 [explicitly did not](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0143.mediawiki#cite_note-7).

[quote="recent798, post:11, topic:710"]
How many of those coinbase outputs are spent along with another coinbase output that stems from a coinbase transaction that can not be duplicated? If all are spent in this fashion, then a future soft fork may not be needed?
[/quote]

I did not check how each of these 189,023 transactions were spent. Indeed if we are really sure all of the BIP34 exceptions have been spent in this fashion, and buried under an amount of work deemed sufficient, then we could get rid of the BIP30 validation without mandating the block height in `nLockTime` (or the witness commitment).

Still i would argue:
1. This analysis is tedious, error-prone and difficult to peer-review;
2. Making absolutely sure txids are unique seems like a good property to have in Bitcoin. And making it so comes at very little cost as this rule would only take effect in 21 years (much more than the current lifetime of Bitcoin).

[quote="recent798, post:11, topic:710"]
The code comment in the reference also mentions that a soft-fork may be preferred, which “*retroactively* prevents block […] from creating a duplicate coinbase”.
[/quote]

A check that the coinbase transaction at block 490,897 can never be `bec6606bb10f77f5f0395cc86d0489fbef70c0615b177a326612d8bd7b84f84d` could be hardcoded in the client, but i think if there is a reorg this deep we have bigger problems anyways.

-------------------------

recent798 | 2024-04-05 16:17:19 UTC | #13

[quote="AntoineP, post:12, topic:710"]
A check that the coinbase transaction at block 490,897 can never be `bec6606bb10f77f5f0395cc86d0489fbef70c0615b177a326612d8bd7b84f84d` could be hardcoded in the client, but i think if there is a reorg this deep we have bigger problems anyways.
[/quote]

Fair enough. Though, a reorg at an early height doesn't have to be problematic. It could normally happen, intentionally or accidentally. For example, if you yourself, or someone else fed blocks via `submitblock` or directly without announcement over P2P. I know that the new PoW DoS protection in headers-first download likely protects against a low-work reorg during IBD over P2P, but I don't think it does over RPC.

-------------------------

sjors | 2024-04-05 17:34:57 UTC | #14

[quote="AntoineP, post:12, topic:710"]
Making absolutely sure txids are unique seems like a good property to have in Bitcoin. And making it so comes at very little cost as this rule would only take effect in 21 years (much more than the current lifetime of Bitcoin).
[/quote]

Ah, so I guess the rule could be: blocks (with more than one transaction) at or above 1,983,702 MUST have a witness commitment. That's plenty of time to make sure all mining software does that by default.

This is probably the only new soft-fork rule that miners could violate by accident. Not requiring a witness commitment for "empty" (only a coinbase) blocks makes it safer, because who knows what custom code is out there to SPV/spy-mine in the brief interval between when a new block is seen and when it's fully validated and the new template is ready. 

[quote="AntoineP, post:1, topic:710"]
I have yet to check if there is any violation of this but i’d be surprised if there is a pre-Segwit coinbase transaction with an output pushing exactly the witness commitment header followed by 32 `0x00` bytes.
[/quote]

There isn't, unless I missed it: https://bitcoin.stackexchange.com/questions/121069/when-was-the-first-op-return-output-in-a-coinbase

(actually that's not exactly what you asked, but at least there's no OP_RETURN use in the coinbase before BIP30 and BIP34 kicked in)

[quote="AntoineP, post:12, topic:710"]
A check that the coinbase transaction at block 490,897 can never be `bec6606bb10f77f5f0395cc86d0489fbef70c0615b177a326612d8bd7b84f84d` could be hardcoded in the client, but i think if there is a reorg this deep we have bigger problems anyways.
[/quote]

This is not necessary. As explained in `validation.cpp`:

```cpp
// Block 490,897 was, in fact, mined with a different coinbase than
// block 176,684, but it is important to note that even if it hadn't been or
// is remined on an alternate fork with a duplicate coinbase, we would still
// not run into a BIP30 violation.  This is because the coinbase for 176,684
// is spent in block 185,956 in transaction
```

So a re-org would have to go all the way back to `185,956`. That's buried below multiple checkpoints. And although we might [get rid of checkpoints](https://github.com/bitcoin/bitcoin/pull/25725) completely, many nodes would consider a reorg that deep a hard fork.

-------------------------

AntoineP | 2024-04-05 18:21:08 UTC | #15

[quote="sjors, post:14, topic:710"]
So a re-org would have to go all the way back to `185,956`.
[/quote]

Sure but what @recent798 was asking about is actually the latter part of this precise comment you pasted, which states we should consider adding a rule to retroactively prevent the two historical candidates (209,921 and 490,897) from having a duplicate coinbase at all.

-------------------------

ajtowns | 2024-04-08 13:27:39 UTC | #16

[quote="sjors, post:10, topic:710, full:true"]
It would be, but that's a hard-fork - that'll have to wait for 2106 :slight_smile:

[BIP 320](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0320.mediawiki) makes some more header bits available for nonce-like grinding. I'm not sure how widely used it is, and IIUC Stratum v2 takes advantage of it. In any case that doesn't require a (soft) fork.
[/quote]

Over the last 10k blocks (828299 to 838299), 9404 blocks have bip320 bits set (matching `/[23]...[02468ace]00[04]/` while 597 blocks don't (matching `/2000000[04]/`), so at least ~94% of hashrate has adopted BIP320 (as compared to reserving all bits for signalling per BIP9). That percentage could be an underestimate if ASICs are grinding through the all-zeroes case as well.

The 4 at the end in both cases is to catch the ~216 blocks that are still signalling for taproot for whatever reason, which I think are all [SBI Crypto](https://twitter.com/ajtowns/status/1777324742149554295).

Reproducer if anyone cares: `$ for a in $(seq 828299 838299); do bitcoin-cli getblockheader $(bitcoin-cli getblockhash $a) | jq -j '.height, " ", .versionHex, "\n"'; done | sed 's/ 2000000[04]/ bip9/;s/ [23]...[02468ace]00[04]/ bip320/' | cut -d\  -f2 | sort | uniq -c`

-------------------------

AntoineP | 2024-05-17 09:38:42 UTC | #17

[quote="AntoineP, post:1, topic:710"]
Within less than 40 days the attacker would bring the difficulty down to 1
[/quote]

To back up this claim i've run a bit more rigorous simulation than my previous estimate. If the attack were to start at [block `842688`](https://blockstream.info/block/000000000000000000026b90d09b5e4fba615eadfc4ce2a19f6a68c9c18d4a2e) (timestamp `1715252414`, difficulty `83148355189239`) it'd take about 39 days to bring down the difficulty down to 1 by exploiting the timewarp vulnerability.

Here is the Python script i've used. The resulting table of every block and its timestamp in this window is available in [this gist](https://gist.github.com/darosior/5a755ebdaefa7ae73be5507d2920914c).

```python
from datetime import datetime

# A real block on mainnet.
START_HEIGHT = 842688
START_TIMESTAMP = 1715252414
START_DIFF = 83148355189239

# The list of (height, timestamp) of each block. Will contain all the blocks during
# the attack, plus the blocks for the period preceding the attack.
blocks = []

# Push the honest period of blocks before the starting height.
for i in range(START_HEIGHT - 2016, START_HEIGHT):
    blocks.append((i, START_TIMESTAMP - (START_HEIGHT - i) * 10 * 60))

# Now the attack starts.
difficulty = START_DIFF
height = START_HEIGHT
periods = 0
while difficulty > 1:
    # Always set the timestamp of each block to the minimum allowed by the MTP rule.
    # We'll override it below for the last block in a period.
    median = sorted(ts for (h, ts) in blocks[-11:])[5]
    blocks.append((height, median + 1))

    # New period. First override the last block of the previous period (ie not the block
    # at the tail of the list which is the first of the new period, but the one before).
    # Then update the difficulty.
    if height > START_HEIGHT and height % 2016 == 0:
        # Estimate how long it took to mine the past 2016 blocks given the current diff.
        diff_reduction = START_DIFF / difficulty
        time_spent = 2016 * 10 * 60 / diff_reduction
        # For the first period we set the 2h in the future. For the next ones, we
        # just offset from the previous period's last block's timestamp.
        prev_last_ts = blocks[-2 - 2016][1]
        max_timestamp = prev_last_ts + time_spent
        if periods == 0:
            max_timestamp += 3_600 * 2
        blocks[-2] = (height, max_timestamp)

        # Adjust the difficulty
        red = (blocks[-2][1] - blocks[-2 - 2015][1]) / (2016 * 10 * 60)
        assert red <= 4
        difficulty /= red
        periods += 1
        print(f"End of period {periods}, reducing the diff by {red}.")

    height += 1

attack_duration = datetime.fromtimestamp(blocks[-2][1] - 3_600 * 2) - datetime.fromtimestamp(START_TIMESTAMP)
print(f"Took the difficulty down to 1 in {attack_duration} after {periods} periods.")

print(f"| height | timestamp |")
print(f"| ------ | --------- |")
for (h, ts) in blocks:
    print(f"| {h} | {datetime.fromtimestamp(ts)} |")
```

-------------------------

AntoineP | 2024-05-17 12:09:42 UTC | #18

[quote="recent798, post:11, topic:710"]
My understanding is that a soft-fork that can not be applied from genesis won’t help to skip the expensive check in case of a massive (theoretical) reorg, so fallback code will be needed for this case. If fallback code is needed anyway, then it seems better to do it without a consensus deployment.
[/quote]

Thinking back about this. I was under the impression, probably influenced by this comment, that we could somehow get rid entirely of BIP30 validation with 1) a soft fork to make coinbase transactions unique in the future and 2) a retroactive soft fork to prevent block 490,897 to have txid `bec6606bb10f77f5f0395cc86d0489fbef70c0615b177a326612d8bd7b84f84d`.

But as @recent798 points out, you still need to have BIP30 validation for older forks, as nothing prevents *in theory* the pre-BIP34-activation committed heights to be changed. Except maybe a convoluted retroactive soft fork under which the pre-BIP34-activation coinbase transactions currently in the main chain are all valid but hypothetical duplicable ones woud not be (i can't think of anything short of a checkpoint which would fulfill both requirements). This does not seem worth the complexity, at all.

So i don't think we'll be able to entirely strip the BIP30 logic. That's fine. I think we should just make coinbase txids after block 1'983'702 unique (either through `nLockTime` or another mechanism). And we keep BIP30 validation for old blocks and forks. 

Note: of course all this disregards checkpoints.

-------------------------

AntoineP | 2024-06-19 08:51:54 UTC | #19

As an additional motivation for making txids unique post BIP34, @kcalvinalvin mentioned to me Utreexo nodes would not be able to perform BIP30 validation.

-------------------------

MattCorallo | 2024-07-22 00:33:57 UTC | #20

[quote="AntoineP, post:1, topic:710"]
However there is a more roundabout fix which is potentially easier for miners to deploy: make the witness commitment mandatory in all coinbase transactions (h/t Greg Sanders). I have yet to check if there is any violation of this but i’d be surprised if there is a pre-Segwit coinbase transaction with an output pushing exactly the witness commitment header followed by 32 `0x00` bytes.
[/quote]

This makes the witness commitment useless for its intended purpose - a flexible (future) merkle root committing to additional commitments. The nLockTime field is a *much* neater way of doing this, and I don't really see why "miners have to slightly tweak their software" is a huge deal as long as they get a year or two of lead time to do it. We could even set this part of the fork to activate on some substantial delay - after the GCCR fork activates, the nLockTime requirement only turns on two years later or whatever.

-------------------------

AntoineP | 2024-07-22 12:38:35 UTC | #21

I agree, i think we should just use `nLockTime` and make the rule kick in at height 1,983,702 (in about 21 years).

-------------------------

ajtowns | 2024-07-23 09:01:14 UTC | #22

[quote="AntoineP, post:1, topic:710"]
i’d be surprised if there is a pre-Segwit coinbase transaction with an output pushing exactly the witness commitment header followed by 32 `0x00` bytes.
[/quote]

As mentioned [earlier](https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710/6?u=ajtowns), the witness commitment that appears in the coinbase for an empty block where the coinbase witness is all zeroes is `aa21a9ed` (commitment marker) followed by `e2f61c3f71d1defd3fa999dfa36953755c690689799962b48bebd836974e8cf9` (commitment).

[quote="MattCorallo, post:20, topic:710"]
This makes the witness commitment useless for its intended purpose
[/quote]

Requiring the presence of a commitment doesn't prevent it from taking any particular value; so it should remain equally useful. Changing the coinbase witness from all zeroes would also change the witness commitment for empty blocks from the constant above, of course.

[quote="MattCorallo, post:20, topic:710"]
a flexible (future) merkle root committing to additional commitments.
[/quote]

I don't think that's actually possible in a soft-fork friendly way: once you add one commitment widely deployed, adding a second commitment will either prevent those nodes from validating the commitment they could validate (making the second commitment a hard fork, or allowing miners to disable validation of the first commitment on a per-block basis by inventing their own additional commitments), or will require providing a full merkle-path to all but the most recent commitment, which effectively just means publishing all the leaf commitments in each block directly.

[quote="MattCorallo, post:20, topic:710"]
The nLockTime field is a *much* neater way of doing this,
[/quote]

I don't really think it's much different either way for bitcoind; but I expect many empty blocks are generated via manual scripts in a KISS-manner, in which case bumping the `nLockTime` to match the BIP34 height commitment is likely a lot easier than adding support for even an all-zero coinbase witness, plus its commitment. So, :+1:

[quote="MattCorallo, post:20, topic:710"]
after the GCCR fork activates, the nLockTime requirement only turns on two years later or whatever.
[/quote]

[quote="AntoineP, post:21, topic:710, full:true"]
I agree, i think we should just use `nLockTime` and make the rule kick in at height 1,983,702 (in about 21 years).
[/quote]

Having a new rule not have any effect until decades in the future seems like a good way to have people not implement it (at all, let alone correctly), and leave it to being a potentially big problem for a future generation, a la Y2K. For a simple change like this, a couple of years' notice seems much better than ~20 years to me. (For a larger change, perhaps up to five years could be reasonable, or perhaps some similar timespan that matched mining equipment's expected lifecycle).

-------------------------

MattCorallo | 2024-07-23 16:04:18 UTC | #23

[quote="ajtowns, post:22, topic:710"]
Requiring the presence of a commitment doesn’t prevent it from taking any particular value; so it should remain equally useful.
[/quote]

Huh? If we fix the coinbase witness value to 00000000height, then you cannot start shoving further commitments in it? We could go ahead and make it a merkle tree of commitments where the left-most commitment is required to be 000000height, of course, but that isn't what I saw proposed here.

[quote="ajtowns, post:22, topic:710"]
Changing the coinbase witness from all zeroes would also change the witness commitment for empty blocks from the constant above, of course.
[/quote]

Sure, I was referring specifically to the coinbase witness value, which AFAIU currently has no consensus rules (and is only 0s by default behavior of Bitcoin Core).

[quote="ajtowns, post:22, topic:710"]
I don’t think that’s actually possible in a soft-fork friendly way: once you add one commitment widely deployed, adding a second commitment will either prevent those nodes from validating the commitment they could validate (making the second commitment a hard fork, or allowing miners to disable validation of the first commitment on a per-block basis by inventing their own additional commitments), or will require providing a full merkle-path to all but the most recent commitment, which effectively just means publishing all the leaf commitments in each block directly.
[/quote]

Indeed, any time you add a new commitment you'll have to have new nodes give old nodes the contents of the commitments/merkle paths, but that isn't a soft-fork-incompatible change, rather requires only a P2P extension.

[quote="ajtowns, post:22, topic:710"]
Having a new rule not have any effect until decades in the future seems like a good way to have people not implement it (at all, let alone correctly), and leave it to being a potentially big problem for a future generation, a la Y2K. For a simple change like this, a couple of years’ notice seems much better than ~20 years to me. (For a larger change, perhaps up to five years could be reasonable, or perhaps some similar timespan that matched mining equipment’s expected lifecycle).
[/quote]

Strongly agree. Also because I'd like to actually write software that is able to more easily look at block heights without looking at stupid scripts in my lifetime :)

-------------------------

ajtowns | 2024-07-24 06:18:44 UTC | #24

[quote="MattCorallo, post:23, topic:710"]
Huh? If we fix the coinbase witness value to 00000000height, then you cannot start shoving further commitments in it? We could go ahead and make it a merkle tree of commitments where the left-most commitment is required to be 000000height, of course, but that isn’t what I saw proposed here.
[/quote]

The idea isn't to encode the height in the coinbase witness, just to require that the coinbase have some witness. That ensures there's an OP_RETURN output in every future coinbase that isn't present in any of the potentially duplicatable existing coinbases.

-------------------------

MattCorallo | 2024-08-07 01:12:12 UTC | #25

[quote="ajtowns, post:24, topic:710"]
The idea isn’t to encode the height in the coinbase witness, just to require that the coinbase have some witness. That ensures there’s an OP_RETURN output in every future coinbase that isn’t present in any of the potentially duplicatable existing coinbases.
[/quote]

Ah, fair enough. I'd still very much love to see the nLockTime set to the block height, it makes pulling the block height out of the coinbase transaction simpler, though its not a super critical difference.

-------------------------

harding | 2024-08-07 05:49:06 UTC | #26

[quote="MattCorallo, post:25, topic:710"]
I’d still very much love to see the nLockTime set to the block height, it makes pulling the block height out of the coinbase transaction simpler, though its not a super critical difference.
[/quote]

I'm not sure if Matt is talking about deserialization being simpler or that compact extraction using a SHA256 midstate would be simpler.  The former is pretty obvious---there's no need to deal with CScript (yay!)---but the later might not be clear, so I'll briefly describe it: with a consensus-enforced commitment to height in the locktime field, which is the last field in a serialized transaction, I can prove to you that a particular 80-byte header is for block 999999 by giving you the 32-byte sha256 midstate of the coinbase transaction up to the final 1 or 2 chunks, the missing chunks (64 bytes each), and a partial merkle tree for the coinbase transaction (448 bytes or less).  You take the midstate and SHA256 iterate over the remaining chunks that commit to the locktime to get the coinbase transaction's txid.  You insert that in the partial merkle tree and verify the tree connects to the merkle root in the block header.

By comparison, to prove that now with the BIP30 height commitment, I need to give you the entire coinbase transaction plus the partial merkle tree (or I need to use a fancier proof system).  A coinbase transaction can be up to almost 1 MB, so the worst case proof of header height now is ~1 MB compared to ~700 bytes with a consensus-enforced commitment in locktime.

I can't think of how that would be useful offhand, but it seems like a nice advantage of the locktime approach.

(Edited: @ajtowns reminded me that SHA256 chunks are 64 bytes, not 32 bytes.)

-------------------------

sjors | 2024-08-20 15:47:06 UTC | #27

[quote="MattCorallo, post:25, topic:710"]
I’d still very much love to see the nLockTime set to the block height, it makes pulling the block height out of the coinbase transaction simpler
[/quote]

Do you have a use case in mind where you need the block height but can't just count the headers? I suppose an air-gapped embedded hardware device could enforce a rule like: "this transaction must exist above a certain height, and I'll believe it if you give me X cumulative difficulty of surrounding headers".

IIUC @harding explains how such a proof would be more compact.

This does imply that we should enforce this rule as soon as the soft fork activates and not wait for block 1,983,702.

Perhaps it should activate a (few) year(s) later, if it turns out miners have to do more than just upgrade their node. At first glance I would think `getblocktemplate` can just take this new rule into account and everything should be fine. But who knows what custom software miners run to process the coinbase transaction.

Here's a branch (PR to self) that adds `-coinbaselocktime` and has our mining code set it:
https://github.com/Sjors/bitcoin/pull/60

If we end up going for this solution, we could encourage miners to run this well before any activation parameters are set, to figure out if there is a problem we're not aware of.

-------------------------

evoskuil | 2024-08-25 15:19:46 UTC | #28

I don't see any reflection here of the [discussion on bitcoin-dev regarding block hash malleability](https://groups.google.com/g/bitcoindev/c/CAfm7D5ppjo/m/MsOdTqYyCwAJ). As that discussion presently stands I see no justification for the proposed invalidation of 64 byte transactions. As discussed, there is a much simpler, more efficient, and equally effective resolution that requires no new consensus rule.

-------------------------

harding | 2024-08-26 14:06:49 UTC | #29

[quote="evoskuil, post:28, topic:710"]
I see no justification for the proposed invalidation of 64 byte transactions. As discussed, there is a much simpler, more efficient, and equally effective resolution that requires no new consensus rule.
[/quote]

In the referenced thread, you [wrote](https://mailing-list.bitcoindevs.xyz/bitcoindev/be78e733-6e9f-4f4e-8dc2-67b79ddbf677n@googlegroups.com/):

> The only possible benefit that I can see here is the possible very small bandwidth savings pertaining to SPV proofs. I would have a very hard time justifying adding any consensus rule to achieve only that result.

It's true that the attack against simplified verification can be prevented through proofs that are a maximum of about 400 bytes larger per block in the worse case, which is about a 70% increase in proof size[1].  That doesn't seem significant in network traffic when many lightweight clients might be using something like BIP157/158 compact block filters that send extra _megabytes_ of data even in the best case.  However, that extra 400 bytes per proof could be significant if merkle proofs are being validated in consensus protocols (e.g. after a script upgrade to Bitcoin).

[1] Base proof: 80 byte header + 448 byte partial merkle tree = 528 bytes.  Proof with coinbase tx, assuming the coinbase tx is in the left half of the tree and the tx to prove is in the right half of the tree: 80 byte header + 416 bytes partial merkle tree for coinbase tx + 416 bytes partial merkle tree for tx = 912 bytes.

-------------------------

evoskuil | 2024-08-26 14:30:57 UTC | #30

This minor wallet optimization was not even mentioned in the above rationale for the new rule. If the sole objective of the proposed rule was to save a small amount of bandwidth for SPV proofs, we would not be having this discussion. The bandwidth savings was a modest side effect benefit of fixing a perceived consensus-related security issue, which has been shown to be a suboptimal and unnecessary fix. It is not a trivial thing to add a new consensus rule, even with the assuption of a roll-up soft fork. This should not even be under consideration at this point.

-------------------------

harding | 2024-08-26 15:00:54 UTC | #31

[quote="evoskuil, post:30, topic:710"]
This minor wallet optimization was not even mentioned in the above rationale for the new rule. If the sole objective of the proposed rule was to save a small amount of bandwidth for SPV proofs, we would not be having this discussion.
[/quote]

The rationale for the new rule starts by describing the vulnerability that affects almost all deployed SPV software.  I myself had forgotten the mention in Daftuar's paper that checking the depth of the coinbase transaction could provide a trustless way for SPV clients to avoid the attack.  I'm not sure how many authors of SPV software are aware of that.

My point was that, as a solution, it still leaves something to be desired.  It increases the amount of code that needs to be written to verify an SPV proof (the client now needs a way to request a coinbase transaction even if it doesn't know its txid, and it needs verify the coinbase tx appears in the correct location in the merkle tree and that its initial bytes have the correct structure for a coinbase transaction).  I previously mentioned that it inflates proof sizes by about 70%, but I now think it's greater than that (I think verification of the initial bytes of the coinbase transaction are required, which means the entire coinbase transaction needs to be downloaded to calculate its txid, which can inflate proof size by up to almost 1 MB).

Fixing a vulnerability that affects widely used software, that simplifies the design of future software, and that reduces network (and potentially consensus) bandwidth seems useful to me.

-------------------------

evoskuil | 2024-08-26 17:33:29 UTC | #32

Let's be absolutely clear about this. The proposed invalidation of 64 byte transactions does not in any way represent a fix to a vulnerability. Valid blocks do not have malleable block hashes.

Nor does it simplify the design of mitigating the invalid block hash caching weakness. Just the opposite, as a mitigation it is far more complex and costly than the existing solution. And note that caching of invalid messages as a DoS optimization is itself counterproductive, as discussed in detail.

In terms of consensus, it adds a pointless and counterproductive rule. It will require validation checks that are not necessary. If you want to argue it as an SPV bandwidth optimization, have at it. But please do not perpetuate this error that it fixes something.

-------------------------

harding | 2024-08-26 18:38:50 UTC | #33

[quote="evoskuil, post:32, topic:710"]
do not perpetuate this error that it fixes something.
[/quote]

I recall you making similar proclamations during the  [milk sad](https://milksad.info/disclosure.html#libbitcoin-team-response-and-context) disclosure.  The intuitive way of using your API was not the secure way to use it.  Rather than change your API, you put a single piece of documentation about the insecurity on a page that many API users may never have read.  Many other pages of your documentation gave examples that used the API in an insecure way.  You believed this was acceptable.  Others disagreed.

We have a similar situation with Bitcoin's merkle trees.  They were intended to allow the generation of cryptographically secure transaction inclusion proofs with a single partial merkle branch.  Now Bitcoin protocol developers know that is insecure.  There's some limited propagation of that knowledge to downstream developers, but it remains an obscure problem with an abstruse solution.  We could content ourselves with the limited documentation we've written about it and claim anyone who later loses money due to this problem is the victim of incompetence---or we could carefully weigh the consensus change required to restore the security of the simple, intuitive, and efficient way of generating and verifying transaction inclusion proofs.

Restoring a simple protocol to its originally intended security is a fix, in my opinion.

-------------------------

evoskuil | 2024-08-26 18:49:58 UTC | #34

[quote="harding, post:33, topic:710"]
I recall you making similar proclamations during the [milk sad](https://milksad.info/disclosure.html#libbitcoin-team-response-and-context) disclosure.
[/quote]

If you think that making this reference improves your argument, it does not. But it does reflect on you.

[CVE-2023-39910](https://github.com/libbitcoin/libbitcoin-explorer/wiki/CVE-2023-39910)

[quote="harding, post:33, topic:710"]
We have a similar situation with Bitcoin’s merkle trees.
[/quote]

No, we do not. You are quite literally arguing in favor of a new and unnecessary CONSENSUS RULE because of "limited propagation of knowledge."

-------------------------

ajtowns | 2024-08-27 09:50:19 UTC | #35

[quote="harding, post:26, topic:710"]
with a consensus-enforced commitment to height in the locktime field, which is the last field in a serialized transaction, I can prove to you that a particular 80-byte header is for block 999999 by giving you the 32-byte sha256 midstate of the coinbase transaction up to the final 1 or 2 chunks, the missing chunks (64 bytes each), and a partial merkle tree for the coinbase transaction (448 bytes or less). You take the midstate and SHA256 iterate over the remaining chunks that commit to the locktime to get the coinbase transaction’s txid.
[/quote]

One thing to note here is that partially verifying merkle trees can have subtle risks; for example if your actual coinbase tx is 400 bytes serialized with an nLockTime of 900,000; it could be that the last four bytes of the txid of the second tx in the block has the value 0x88bf0d00 (901,000 in little endian), at which point the concatenation of the two txids looks something like a 64-byte transaction with an nLockTime of 901,000, and you could give a "valid" merkle proof that the height of the block is 1000 blocks higher than its true height. That problem goes away if the verifier is able to assume that the coinbase tx for a valid block is always greater than 64 bytes (true provided either five or more bytes of extranonce is used, a segwit commitment is included, or the block reward is not burnt) and verifies the provided midstate is not the sha256 initial state.

-------------------------

evoskuil | 2024-08-27 16:05:12 UTC | #36

[quote="evoskuil, post:32, topic:710"]
It will require validation checks that are not necessary. If you want to argue it as an SPV bandwidth optimization, have at it.
[/quote]

The previously-cited discussion on bitcoin-dev centered on the detection of a malleated block message, for the purpose of caching block invalidity. As justification for the proposal it was argued that the fork allows for (1) earlier and therefore more efficient detection/prevention of block hash malleation, (2) that this is important because of the supposed DoS benefit of caching invalid block hashes, (3) that this would allow a block hash to uniquely identify a block (presumably the block message payload), valid or otherwise.

It was shown in the discussion, and I believe agreed, that these arguments aren't valid. (1) Detection is presently possible without even parsing beyond the input point of the coinbase tx, whereas prevention via size constraint requires parsing every transaction in the block to determine sizes, (2) caching the hashes of block headers that are determined to represent invalid blocks serves no effective DoS protection purpose, instead opening the node to a disk fill attack (similar to the recent banning vulnerability) that must be mitigated, and (3) this objective is not achieved as duplicated tx hash malleation remains.

These are the aspects that pertain to a node/consensus. As I stated above, one can certainly argue that this fork can simplify/optimize SPV implementation. My point is to exclude the invalid arguments from consideration.

[quote="evoskuil, post:30, topic:710"]
This should not even be under consideration at this point.
[/quote]

I'll withdraw this statement, I got a little carried away. It's worthy of consideration as long as we are clear about the meaningful objectives. Once invalid objectives are discarded it may (or may not) be that other SPV optimizations become more attractive.

-------------------------

evoskuil | 2024-08-27 17:57:10 UTC | #37

[quote="harding, post:31, topic:710"]
I previously mentioned that it inflates proof sizes by about 70%, but I now think it’s greater than that (I think verification of the initial bytes of the coinbase transaction are required, which means the entire coinbase transaction needs to be downloaded to calculate its txid, which can inflate proof size by up to almost 1 MB).
[/quote]

According to my quick computations, the average coinbase size for all blocks up to the most recent halving is 256 bytes and starting from segwit activation is 260 bytes. The average Merkle tree depth is 8 and 11 respectively. This implies an average segwit-era download cost of `11*32 + 260 = 612` bytes to validate the Merkle proofs for *all txs* of a given block (.34 seconds on a 14,400 baud modem).

Given the speeds involved and these averages, basing such a decision on worst case seems unreasonable to me. The [largest coinbase in the above BTC history](https://blockstream.info/tx/f36222943ad100899acbd8300f943ee2c127babef879d8a3c0696c0d914e04ca) is 31,353 bytes, while the [largest in the segwit era](https://blockstream.info/tx/296fd33e4cb75e6746d3f80f31bef8cb19bf1952690b72b1cec9198e3967a937) is just 6,825 bytes.

[quote="harding, post:31, topic:710"]
the client now needs a way to request a coinbase transaction even if it doesn’t know its txid
[/quote]

Its block hash is a sufficient coinbase identifier.

> Another way to fix SPV wallets is to require, along with a Merkle-proof of the inclusion of a transaction E, a Merkle proof for the coinbase transaction. Because building a dual transaction-node for the coinbase transaction requires brute-forcing 225 bits, showing a valid coinbase and its corresponding Merkle inclusion proof is enough to discover the tree height. Both the E branch and the coinbase branch should have equal tree depths. -
[Leaf-Node weakness in Bitcoin Merkle Tree Design](https://bitslog.com/2018/06/09/leaf-node-weakness-in-bitcoin-merkle-tree-design/)

-------------------------

ariard | 2024-08-27 18:45:52 UTC | #38

> I recall you making similar proclamations during the milk sad disclosure. The intuitive way of using your API was not the secure way to use it. Rather than change your API, you put a single piece of documentation about the insecurit> y on a page that many API users may never have read. Many other pages of your documentation gave examples that used the API in an insecure way. You believed this was acceptable. Others disagreed.

Come on Dave... this is borderline sheer intellectual dishonesty. I think you’re worth better than that.

I had a [look](https://github.com/libbitcoin/libbitcoin-explorer/issues/728#issuecomment-2068392998) few months on the libibtcoin full report about the milk sad disclosure (and I read previously the milk.sad report from their original authors when it was made available). I could easily point out many places of the Bitcoin Core API, which are weak in terms of usage documentation, and I'm polite. Designing a secure API, both in terms of code and conveying reasonable information on usage to downstream users ain’t an easy task…

> We have a similar situation with Bitcoin’s merkle trees. They were intended to allow the generation of cryptographically secure transaction inclusion proofs with a single partial merkle branch. Now Bitcoin protocol developers know that is insecure. There’s some limited propagation of that knowledge to downstream developers, but it remains an obscure problem with an abstruse solution. We could content ourselves with the limited documentation we’ve written about it and claim anyone who later loses money due to this problem is the victim of incompetence—or we could carefully weigh the consensus change required to restore the security of the simple, intuitive, and efficient way of generating and verifying transaction inclusion proofs.

If the crux of the conversation is reestablishing simplified payment verification in a robust fashion as it is described in the whitepaper section 8, I think there is one aspect which is missed in the section by Satoshi, and that has been corroborated by deploying things like bip37. Namely increasing the DoS surface of full-nodes delivering such transaction inclusion proofs, as generating and indexing can be a non-null computing cost for a full-node, already made that observation in the past e.g [on the scalability issues of lightweight clients](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-May/017818.html).

In my view, making a consensus change to optimize SPV verification (e.g such as requesting coinbase transaction to be invalid) is still running short of coming first with a paradigm finding an equilibrium for the network economics (what is already the number of available Electrum / BIP157 public servers ? like a x1000 order of magnitude less than IP-distinct full-nodes ?). It can be still be deemed a valuable goal, though I believe we’re missing the wider picture.

-------------------------

evoskuil | 2024-08-27 23:22:45 UTC | #40

[quote="evoskuil, post:37, topic:710"]
This implies an average segwit-era download cost of `11*32 + 260 = 612` bytes to validate the Merkle proofs for *all txs* of a given block (.34 seconds on a 14,400 baud modem).
[/quote]

This can be reduced by 36 bytes, since a valid coinbase must always contain the null point: `0xffffffff0000000000000000000000000000000000000000000000000000000000000000`. As this can be assumed, the segwit-era average download cost to validate all Merkle proofs for a given block would actually be 576 bytes.

It's also worth pointing out that the associated storage cost is nominally one byte (actually 4 bits) per block, since the only required information is the proven Merkle tree depth.

> Another soft-forking solution is to require that a new field “depth” requiring 4 bits is embedded in the block version field. This new fields should specify the Merkle tree depth minus one. By using 4 bits, a a tree of depth 16, containing up to 65K transactions can be specified. Full nodes much check that the actual tree depth matches the value of “depth” given. SPV nodes can check that the “depth” field matches the Merkle branch length received. - Ibid.

-------------------------

AntoineP | 2024-09-03 16:06:05 UTC | #41

I see reasonable arguments on both sides. Thanks everyone for contributing those.

To evaluate whether to include it in a future Consensus Cleanup proposal, i'd like to weigh the remaining pros and cons of making 64 bytes transactions invalid through a soft fork. To do so i'll list the main pros and cons i can think of and/or were raised by others (notably Dave and Eric above). Then i'll try to make the best case for each position's arguments, and provide a rebuttal for some of the points presented when i can.

Please let me know if an argument (or rebuttal) is missing or could be better presented.

Here are the pros:
- Reduces the bandwidth of block inclusion proofs for transactions.
- Eliminates potential risks when implementing an application that verifies such a proof.

And the cons:
- Introduces a non-obvious transaction validity rule.
- Requires a soft fork.

Let's try to make the best case for each argument, starting with the pros.

This change would allow merkle proofs to be ~50% smaller in both the worst and average cases for a typical 200 bytes transaction [0]. In addition it would remove a large footgun threatening anyone implementing software which needs to verify transaction merkle proofs. This concern is growing since, as network hashrate increases, the price of a fake inclusion proof using this method is decreasing compared to the price of mining an invalid block. This is in addition to the fact this method allows to fake any number of confirmations, compared to a single one by producing an invalid block.  Finally, it is important to not only consider lite clients. Proofs of inclusion of a transaction in a block are useful in other, existing or potential, applications. Examples include sidechains, or future Script changes which would allow to check a merkle proof directly in the interpreter.

Here is a list of rebuttals to those specific points:
- Looking at the worst case cost is not a valid way of judging efficiency gains.
  - True. But the proof size reduction is (very) similar in the average case.
- Hypothetical future consensus changes should not be taken into account for the purpose of this standalone fix. Because it's hard to weigh the benefit this usecase brings since we don't know whether it would ever exist, and simply because the fix could then be bundled with a future soft fork which enables this usecase.
  - Sure. Other usecases than lite-client remain though, such as sidechains. It's also not hard to imagine a Bitcoin-related application could verify a block inclusion proof.

Now, the cons.

All changes to consensus rules bear a cost. The proof size reduction and proof verifier simplification does not meet the bar to be considered as a soft fork. The proof size reduction seems large in proportion, but it's small in absolute: a few hundred bytes. In addition you need only one per block, so the efficiency improvement decreases quickly as you query proofs for more than one transaction per block. The implementation simplification is also not a given. It removes one non-obvious check to be performed by inclusion proofs verifiers by another that needs to be performed by all validating nodes, which introduces [consensus "seam"](https://groups.google.com/g/bitcoindev/c/CAfm7D5ppjo/m/kKD1x3gJCgAJ).

And a list of rebuttals for those:
- Part of the fixed cost of the soft fork itself is compensated by bundling this fix with others. The cost inherent to changing the consensus rules should be weighed against the benefits brought by all the fixes together.
- It's true making 64 bytes transactions invalid introduces consensus "seam", but it's not clear how it's an issue. It's a simple and straightforward rule which adds virtually no cost for full nodes to check.
- It's true that "all transaction sizes that fit in a block are valid except for 64 bytes" is surprising as a rule. But we can't equate this with "actually merkle proofs for 64 bytes transactions are insecure and you need to be aware of this clever-yet-convoluted workaround of asking for a proof for the coinbase too". In addition, we can generally expect protocol developers implementing a full node to be more aware of intricacies in the protocol than would be an application developer.


Anything i'm missing?

----

[0] 200 bytes is a slight over-estimation. The median vsize of transactions using a moving average over the past 1000 days is about 189 vbytes according to [transactionfee.info](https://transactionfee.info/charts/transactions-sizes). Calculation assumes a 260 bytes coinbase transaction, not compressed. In the worst case the proof is `80 + 14*32 + 200 = 728` without the coinbase vs `80 + 2*14*32 + 200 + 260 = 1436` with. In the modern average case under the same assumption the proof is `80 + 11*32 + 200 = 1244` without the coinbase vs `80 + 2*11*32 + 200 + 260 = 632` with.

-------------------------

harding | 2024-09-03 17:21:52 UTC | #42

[quote="AntoineP, post:41, topic:710"]
This change would allow merkle proofs to be ~50% smaller in both the worst and average cases for a typical 200 bytes transaction [0]
[/quote]

As mentioned [[1](https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710/31?u=harding)], I think the worst case proof size now is actually ~1 MB.  For example, imagine we have the following block (P2P `block` serialization):

| Bytes | Description |
|--:|--|
| 80 | Header |
| 1 | Tx count |
| 999,826 | Coinbase tx |
| 93 | 1-in, 1-out P2TR (stripped size) |

A lite client performing whitepaper-style SPV can be tricked into accepting a fake transaction if the coinbase transaction was actually 64 bytes.  To avoid that, it needs to learn the contents of the entire coinbase transaction in order to derive its txid for verifying its depth in the merkle tree.  That means, even with optimizations, a proof size of about 1 MB.

Obviously, very large coinbase transactions will be rare given that they reduce miners' ability include fee-paying transactions, but I think it's worth noting in discussion and documentation that the worst case is ~1 MB.  It should still be possible to validate a worst-case merkle proof with coinbase in witness data (given other soft fork changes), but it would be ~2,000x more expensive than validating a merkle proof that didn't require a copy of the entire coinbase transaction.

[quote="AntoineP, post:41, topic:710"]
Anything i’m missing?
[/quote]

@evoskuil mentioned an alternative potential soft fork: a commitment to tree depth, which could be done for any depth possible with the current consensus rules[1] using only 4 bits.  He didn't suggest where the commitment could be stored, but I think it's clear that we're probably never going to use all BIP8/9 versionbits and miners currently seem satisfied with the 16 BIP320 version bits, meaning we could probably put the commitment it the block header version.  That wouldn't require any extra bandwidth for SPV.

I think the two cons of that approach are:

- We might want to give more (eventually all of the available) versionbits to miners in the future as hashrate increases; that way individual hardware doesn't need to create coinbase extranonces or use hacks like nTime rolling.
- It still makes SPV more complicated than described in the whitepaper, although only slightly more so.

[1] 4 bits can express a maximum depth of 16 and a tree of depth 16 can have up to 65,536 transactions.  However, the minimum possible transaction size is [60 bytes](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-May/017883.html) and a 999,919-byte block (excluding header and tx count) can only fit a maximum of 16,665 transactions of that size.

-------------------------

ajtowns | 2024-09-04 03:16:40 UTC | #43

[quote="harding, post:42, topic:710"]
A lite client performing whitepaper-style SPV can be tricked into accepting a fake transaction if the coinbase transaction was actually 64 bytes. To avoid that, it needs to learn the contents of the entire coinbase transaction in order to derive its txid for verifying its depth in the merkle tree.
[/quote]

I don't think that's strictly true? sha256 includes the size of its input when calculating the hash, so implicit in being provided a midstate is being provided the number of bytes that went into the midstate, in bitcoin that's `s` (the midstate), `buf` (the remaining bytes that will build the next chunk) and `bytes` (the number of bytes that went into both those things). So a preimage for the txid of a very large coinbase tx can still be reduced to as few as 104 bytes. That requires you have some way of differentiating a "valid" 64 byte coinbase tx from an internal node of the tx merkle tree, but it's easy to transmit the full coinbase tx in that case anyway.

[quote="AntoineP, post:41, topic:710"]
Reduces the bandwidth of block inclusion proofs for transactions.
[/quote]

Compared to the status quo, it improves the correctness of block inclusion proofs. Compared to including the coinbase in the proof, I think the reduction in complexity is more of an advantage tbh:

  * you need a merkle path, and also to check the witness-stripped tx isn't 64 bytes
  * you need a combined merkle path to both the coinbase and the tx; you need to check they're the same depth; you need to check the coinbase is valid; in order to minimise bandwidth you need to deal with sha256 midstates
  * you need to lookup the block's merkle tree depth via (TBD), (except if the block height is prior to the activation height, in which case you'll need to use this fixed table and hope there's no absurdly large reorg) then check the merkle path is exactly that long

Perhaps worth noting that these merkle proofs will only give you the witness-stripped tx anyway -- in order to verify the tx's witness data you need the full coinbase in order to obtain the segwit commitment (which can be essentially anywhere in the coinbase tx), and then you need the matching merkle path from that to your non-stripped tx. So for a lite node, checking the stripped size of the tx is straight-forward -- that's all the data it's likely to be able to verify anyway.

It occurs to me that most of the time, you're probably more interested in whether an output is (was) in the utxo set, rather than whether a tx is in a block. But I think the concerns here are irrelevant for things like utreexo that have some new merkle tree: they can use a prefix to distinguish tree contents from internal nodes.

-------------------------

harding | 2024-09-04 11:08:40 UTC | #44

[quote="ajtowns, post:43, topic:710"]
sha256 includes the size of its input when calculating the hash, so implicit in being provided a midstate is being provided the number of bytes that went into the midstate, in bitcoin that’s `s` (the midstate), `buf` (the remaining bytes that will build the next chunk) and `bytes` (the number of bytes that went into both those things). So a preimage for the txid of a very large coinbase tx can still be reduced to as few as 104 bytes.
[/quote]

It took me a while to figure out how knowing the number of bytes used to 
create a digest was an alternative to validating the structure of a 
coinbase transaction.  Do I accurately understand that you propose the 
following:

1. Prover provides the coinbase transaction midstate (including size)
for all but the final chunk of the first of the double SHA256 hashes,
plus the final chunk.

2. Prover provides a partial merkle branch for the coinbase transaction.

3. Verifier uses the midstate (including size) and the final chunk to
generate the coinbase txid.

4. Verifier checks all provided nodes in the partial merkle branch are
on the right side (i.e., that the branch is for the first transaction in 
a block, i.e. the coinbase transaction).

5. Verifier inserts its generated coinbase txid into the partial merkle
branch and verifies up the tree until it connects to the merkle root in 
the block header.

After the above process, the verifier can be certain of the following:

- The number of bytes used to generate the digest (txid).

  - If the number of bytes is not equal to 64, then the digested data
    must have been a transaction (or the block was invalid).

  - If the number of bytes is equal to 64, then the digested data may
    have been either a transaction or concatenated digests for an 
    internal merkle node.  The verifier can distinguish between these by
    checking whether the 64-byte chunk provided to it matches a coinbase
    transaction pattern.

- The verified transaction is a coinbase transaction given the inability 
  to use CVE-2017-12842 to forge coinbase transactions and the position
  of the transaction in the merkle tree (or the block was invalid).  
  
- The depth of the coinbase transaction, which is the same depth as all
  other transactions in the tree.

-------------------------

evoskuil | 2024-09-04 16:01:43 UTC | #45

[quote="ajtowns, post:43, topic:710"]
I think the reduction in complexity is more of an advantage tbh:
[/quote]

I agree, as the average absolute bandwidth is trivial, can be easily reduced by another 36 bytes, and can be amortized over an entire block of transactions.

[quote="AntoineP, post:41, topic:710"]
Looking at the worst case cost is not a valid way of judging efficiency gains.
[/quote]

And as the above implies, worst case should not be a consideration given that it is reasonably bounded.

[quote="ajtowns, post:43, topic:710"]
you need a combined merkle path to both the coinbase and the tx; you need to check they’re the same depth; you need to check the coinbase is valid; in order to minimise bandwidth you need to deal with sha256 midstates
[/quote]

Given that bandwidth is not a material issue, adding sha256 midstates is unnecessary complexity.

Also, by simply compacting out the excess 36 bytes as mentioned above there is no need to validate the coinbase. Reconstituting it with the presumed null point is sufficient  (e.g. defaulting a tx deserialization with no input point to a null input point). 

Consequently the complexity comparison becomes:

* Ensure that the stripped transaction is not 64 bytes, and ensure that its Merkle path is valid.

vs:

* Ensure that the (reconstituted) coinbase Merkle path is valid, and ensure the same valid Merkle depth for any tx in the block.

Validating the coinbase Merkle path is of course code reuse, so the new logic is checking the Merkle path size. This is the tradeoff in the SPV wallet. The tradeoff in the full node is mandatory checking of the stripped size of all transactions (and soft fork) vs. doing nothing (possibly "compressing" null points for SPV clients).

-------------------------

plebhash | 2024-09-05 23:18:30 UTC | #46

Cross referencing this SV2-related post here, as I brought up GCC as a point of relevance on that discussion.

https://delvingbitcoin.org/t/pplns-with-job-declaration/1099/3

-------------------------

AntoineP | 2024-11-04 21:06:02 UTC | #47

[quote="harding, post:42, topic:710"]
A lite client performing whitepaper-style SPV can be tricked into accepting a fake transaction if the coinbase transaction was actually 64 bytes.
[/quote]

Not important to your point but executing this attack using the coinbase transaction requires brute-forcing 40 more bits in the first stage, which makes it fairly impractical.

[quote="harding, post:42, topic:710"]
@evoskuil mentioned an alternative potential soft fork: a commitment to tree depth [...].

I think the two cons of that approach are:
[/quote]

To add to this list:
- This is a roundabout way of closing the fake inclusion attack vector instead of addressing the root issue that 64 bytes transaction are indistinguishable from inner nodes.
- This requires miners to update their software stack, whereas making 64 bytes transactions invalid doesn't require them to do anything (if they aren't running a non-standard Bitcoin Core).

[quote="ajtowns, post:43, topic:710"]
* you need a merkle path, and also to check the witness-stripped tx isn’t 64 bytes
[/quote]

[quote="evoskuil, post:45, topic:710"]
Consequently the complexity comparison becomes:

* Ensure that the stripped transaction is not 64 bytes, and ensure that its Merkle path is valid.

vs:

* Ensure that the (reconstituted) coinbase Merkle path is valid, and ensure the same valid Merkle depth for any tx in the block.
[/quote]

Even then for an application which checks the SPV proof of transactions for received payments this would never be an issue as they would only check transactions paying to a normal scriptpubkey in the first place, which would implicitly ensure the transaction is never 64 bytes.

-------------------------

AntoineP | 2024-11-05 14:54:27 UTC | #48

After re-reading through the arguments presented in this thread i'm leaning toward including the invalidity of 64 bytes transactions as part of the consensus cleanup proposal. I appreciate Eric's push back against some of the advertised benefits (my caching assumptions being incorrect and the bandwidth reduction in absolute terms being modest). Still i think the actual benefits, mostly in terms of reduced complexity for SPV applications, outweigh the cost.

-------------------------

AntoineP | 2024-11-08 15:52:47 UTC | #49

[quote="AntoineP, post:1, topic:710"]
A neat fix would be to simply requires the `nLockTime` field of the coinbase transaction to be set to the height of the block being created.
[/quote]

It recently occurred to me this would require mandating the coinbase input to have its `nSequence` field be `SEQUENCE_FINAL`, as otherwise the coinbase transaction would fail the `IsFinalTx()` check (the locktime value is the height at which a transaction is valid in the *next* block).

-------------------------

ajtowns | 2024-11-09 08:35:14 UTC | #50

Why wouldn't you have `nLockTime` be the height of the parent block (mod 500M) instead?

-------------------------

AntoineP | 2024-11-09 15:16:49 UTC | #51

I just figured the off-by-one would be unnecessarily confusing to anyone trying to use this. But i don't have a strong opinion, and we might favor the small confusion over restricting the coinbase's `nSequence`.

The height could also be encoded in the coinbase's `nVersion`, and if we think making the block height available in a simpler manner than parsing the scriptSig isn't a goal then i think just requiring the coinbase transaction `nVersion` be anything but `1` would be sufficient.

-------------------------

AntoineP | 2024-11-26 16:12:00 UTC | #52

I just caught up with the testnet4 discussions around fixing timewarp from back in May. Here is the TL;DR:
-  The [original testnet4 PR](https://github.com/bitcoin/bitcoin/pull/29775) included a timewarp fix with a 2h leeway (as opposed to the 10 minutes from Matt's original proposal). The rationale for this was that if a miner publishes the last block of a period 2 hours in the future, another miner working on the first block of the next period should be able to use its "wall clock" as this block's timestamp. I'm speculating but i think the intention behind this was to not make it possible for a single miner to push the time forward just because it happened to mine the last block of a period. But this is not necessary, the second block of the new period can just get back to use the current time.
- The leeway was reduced from 2h to 10min in PR [#30647](https://github.com/bitcoin/bitcoin/pull/30647) after discussion in BIPs PR [#1658](https://github.com/bitcoin/bips/pull/1658) on the basis that 2h leeway would unnecessarily leave (slightly) more room for inflation, since Bitcoin Core would never create a block template with an invalid timestamp in the first place since PR [#30681](https://github.com/bitcoin/bitcoin/pull/30681). A further motivation was to go with the more restrictive value on the test network to be able to learn about any unknown compatibility issue.

-------------------------

AntoineP | 2024-11-26 16:26:00 UTC | #53

I think the Consensus Cleanup should include a fix for the [Murch-Zawy attack](https://delvingbitcoin.org/t/zawy-s-alternating-timestamp-attack/1062#variant-on-zawys-attack-2), in the form of requiring the timestamp of the last block in a retarget period to never be lower than the timestamp of the first block in this same period. In other words, the time span between the first and last block of a retarget period should never be negative.

[We discussed it](https://delvingbitcoin.org/t/zawy-s-alternating-timestamp-attack/1062/3?u=antoinep) in this thread already back in August, but i figured i should mention it here as well.

-------------------------

ajtowns | 2024-11-27 00:13:24 UTC | #54

[quote="AntoineP, post:53, topic:710"]
in the form of requiring the timestamp of the last block in a retarget period to never be lower than the timestamp of the first block in this same period
[/quote]

Why this, rather than a rolling "past time limit" along the lines of [option 3 here](https://delvingbitcoin.org/t/zawy-s-alternating-timestamp-attack/1062/5?u=ajtowns)? (eg: as well as the monotonic median time past, we maintain a non-decreasing `earliest_time_past`, which is initially the genesis timestamp, and thereafter `max(earliest_time_past, block.nTimestamp - 2 hours)`, and add a new consensus rule `next_block.nTimestamp >= earliest_time_past` in addition to `next_block.nTimestamp > median_time_past`)

Having a rule that is applied consistently to every block seems somewhat better to me than a rule that only applies to every 2016th block to me. YMMV.

(Most of the discussion in that thread was about options 1 and 2, which replaced/simplified median time past, which I don't think is a practical option)

-------------------------

zawy | 2024-11-27 14:48:15 UTC | #55

TL;DR
A 2 hr limit on every block enables an easy attack on external applications that don't implement it correctly if the median of past timestamps (MPT ot MTP) is ever more than 2 hours in the past. Applying Murch's timestamp limit to every block appears better than doing it only on the 2016th block because it forces everyone to view it and code it as a change to the MPT consensus rule which will prevent mistakes and attacks if it's viewed as only a "timestamp" or "difficulty" change. This is because there may be script or code somewhere in the world that is focused on MPT and the developer thinks the change only affects timestamp or the difficulty.

I called the 2 hr limit on every block the "safest and easiest" option, but I need to retract that due to 2 hour being too small. It changes the MPT consensus rule that a lot of software depends on by overriding it if a timestamp >2 hr in the past occurs and if the MPT is even further in the past. It enables an easy malicious attack on all software that doesn't implement the rule change. A single miner only needs to use a timestamp >2 hrs in the past if the MTP happens to be more than that.

The general rule is that if you make a consensus rule change, make it as small as possible. This is what Murch's fix does (timestamps at 2016th block can't older than the 1st block), but there's a lack of "symmetry", "consistency", "simplicity", or "beauty" in not applying the limit to every block which might be a way of saying there's an unknown risk. It seems harder for me to reason that doing it on only 1 block is as safe as doing it on every block. Applying it to every block might be simpler (less risky) code. In testing, the change would occur 2016 times more often, possibly making problems easier to detect. If it's safe at the transition, it better be safe on every block. 

So the choice is between minimal consensus change (apply only on 2016th block) and adhering to "symmetry", "consistency", "consistency", or "beauty" as a way to prevent problems that are hard to detect or deduce. 

As observational evidence that lack of "symmetry" or "beauty" is risky in ways that are hard to detect (and therefore the same changes should be applied to every block) I think halvings and difficulty changes not being slowly changed on every block are possibly Satoshi's biggest mistakes. No alt coin has survived on-off mining when using bitcoin's difficulty algorithm which means it's dangerous to bitcoin itself if the fundamental security requirement of non-repurposable hashrate equipment is relaxed or not present, as it is in most alt coins. The halvings are harmful to miners' investment planning and have caused large price fluctuations that have been brutal to weak hodlers and bitcoin's reputation in the populace. In both cases Satoshi chose math or code simplicity over "symmetry" or "consistency". He should have felt some uneasiness about those decisions despite there not being an error in the calculations. It's widely known (at least intuitively) that batch processing or fluctuations towards a goal is less efficient than continuous, stable progress.

The two problems occured but there wasn't any error in Satoshi's math. The problem is at a higher level than his objective. I don't know what problem there could be in this case of changing once per 2016 blocks, but it probably isn't in the specific objective of preventing attacks, but "external" to it. For example, "external" code or applications may not have an awareness of something happening every 2016 blocks but depend only on the MPT, which is potentially going to be overridden. So from their point of view, not doing it every block might be a "larger" consensus change.

If people only look at the difficulty algorithm or timestamp code to deploy or see the effects of this consensus change, it allows  attacks on code that only need to know the MPT. This should be marketed as an MTP change, not a difficulty or timestamp change. Doing it every block seems better in the sense it forces everyone to realize it's an MTP rule change.

This argument doesn't mean "we might as well use a 2 hr limit" because it allows an easy attack on code that doesn't update. If some reason the proposed two week limit makes people uneasy, a 1 day limit would be safer than 2 hours.

-------------------------

AntoineP | 2024-11-27 14:50:19 UTC | #56

[quote="ajtowns, post:54, topic:710"]
Why this, rather than a rolling “past time limit” along the lines of [option 3 here ](https://delvingbitcoin.org/t/zawy-s-alternating-timestamp-attack/1062/5)?
[/quote]

Because it would be an unnecessarily bigger and more restrictive change. Restricting only the timestamp of those blocks that matter is less invasive and i find it elegant to simply check the timespan is not negative.

[quote="ajtowns, post:54, topic:710"]
Having a rule that is applied consistently to every block seems somewhat better to me than a rule that only applies to every 2016th block to me. YMMV.
[/quote]

Why does it seem somewhat better to you?

-------------------------

zawy | 2024-11-28 12:39:39 UTC | #57

Consensus primacy: MTP -> timestamps -> difficulty.

In deciding the consensus in any consensus mechanism, monotonicity has primacy over timestamps. In the case of PoW, timestamps have precedence over difficulty because they are how difficulty is determined which determines PoW. No adjustment to timespan (or target for that matter) that is restricted to the difficulty calculation should be made. It doesn't matter what difficulty algorithm you use, as long as it's a "fair" (blind) mathematical estimate of the hashrate based on prior difficulty(ies) and solvetime(s). Changing timespan in the algorithm pollutes the math. I have seen many exploits that result from that. For example, the 4 and 1/4 timespan limits allow exploits which I previously said should exist for this theoretical reason and apparently PW believed my reasoning enough to found the exploit. He may do the same here. 

So this consensus rule change should be made in the MTP calculation. When it's at that level, it makes more intuitive sense to do it every block.

Hard-to-see "gotcha" problems involve the MTP because it's a patch instead of strictly following monotonicity.

edit: I was having trouble figuring out what it means to say

MTP -> timestamps -> difficulty.

when I know that's the theoretical requirement and yet timestamps dictate MTP. The answer prevents the exploit: it means that the "timestamps" used by the difficulty algorithm to calculate timespan must be the MTP of the 1st and 2016th blocks. Even PW's attack on the 4x ad 1/4 timespan limits can be fixed without removing them completely if they're turned into limits on the MTP. That is, no timestamp can be more that 3/4 * 2016 * 600 seconds into the past from the current MTP or more than 4 * 2016 * 600 into the future. If there's a network partition for more than that limit the limit is used until real time catches up. Likewise for the past limit.

But either of these are too big of a change to implement. I don't know of any simple solution that would make me feel comfortable in saying "it probably can't be attack" except to make MTP=1 instead of 11. This forces true monotonicity. But as a minimum, Murch's limit on timespan better be enforced on the timestamp instead of timespan or there's probably an exploit.

How many times have people had to learn "don't use timestamps for time, use the MTP"? And here we are still trying to use them for the absolute core of our consensus. Non-monotonic timestamps for any consensus mechanism are not legitimate.

It's concerning that the "official time" for a lot of things in bitcoin like what scripts use is the MTP, but the consensus mechanism  is based on timestamps. The consensus mechanism isn't determining the value of the MTP.  All nodes place an upper limit on it, so it's how far in the past that a 51% attack, enabling an unlimited disconnect even if the difficulty can't be exploited for excess blocks. The MTP monotonicity isn't enforcing monotonicity on the difficulty algorithm if there's a >51% attack which is why we had to proposed a monotonic rule on the timespan.  Also, by limiting only the timespan, the consensus isn't attesting to the value of the 1st and 2016th timestamps. 

This gives two levers that miners can control without PoW. 

Monotonicity on each timestamp is the only way to assure there are no unknown attacks external to the difficulty algorithm. Assuming that's too drastic to implement (via MTP=1 instead of 11), I'd say no timestamp should be older than 1 second after the 2016th timestamp in the past. But this doesn't address the problem of the disconnect between the difficulty timestamps and the MTP time used everywhere else. The simplest fix for that if MTP=1 can't be used is to base the timespan on the MTP of the 1st and 2016th blocks. This wouldn't need a limit on the timestamps.

Lamport's 1978 paper tells us what must be done with timestamps in all consensus mechanisms to do it correctly.  We can propose patches but if the rules aren't followed, there's an attack. The only way to let the PoW prove the 1st and 2016th timestamps and MTP are correct is to force the timestamps to be monotonic. All nodes independently prevent miners from pushing time forward. Monotonicty prevents them from holding it back.

The other rules being broken show an attack isn't necessarily large. The other rules are that clock tick must be slower than propagation speed but a lot faster than the length of consensus rounds (1 block) and clock accuracy should be less than 1/2 clock ticks. Enforcing these rules would prevent the possibility of 1 or 2 block selfish mining attacks, is they start being a problem.

The non-repurposability requirement for good security forces the miners to act in the best interest of the value of the coin, but I believe there are instances where they won't if they're not required to such as maximizing fees. They can't increase the number of blocks with the proposed fix, but can their control of the other two levers somehow increase fees or deprecate  something they don't like? What are the possible profitable  consequences if miners as a group hold the MTP back 6 months and make it jump to the present in 1 block?  This can't be done without a super-majority if the timespan limit is moved out of the DA to a limit on every timestamp.

Murch's fix seems the easiest and safest in terms of preventing an attack and not breaking anything. But a monotonicity fix should be readily available in case of an emergency. There should be a BIP.

-------------------------

ajtowns | 2024-11-30 09:28:17 UTC | #58

[quote="AntoineP, post:56, topic:710"]
Why does it seem somewhat better to you?
[/quote]

Mostly that a field with a fairly free choice of values normally having different constraints apply to it 0.05% of the time seems a bit error prone, in that it would be easy to not take the 0.05% chance thing into account, and cost people money as a result... But I suppose the same complaint would apply to the original timewarp fix too, and these are really 0% chance things outside of very high cost attacks anyway.

-------------------------

zawy | 2024-12-01 11:24:27 UTC | #59

Opentimestamps isn't secure [edit: to the extent MPT time is used as the measure of time instead of the highest timestamp value seen]. This is due to bitcoin not adhering to a proven distributed consensus requirement. 

[edit: Here's a simple solution if attacks on Opentimestamps start occurring: don't use MPT as the official bitcoin time. Use the timestamp in the prior 11 blocks that has with the highest value. It can only be 2 hours at most into the future, and being in the future instead of past is consistent with a working timestamp.] 

Unlike other 51% attacks, this isn't prevented by bitcoin's security model which depends on miners' long-term profit motive. **MPT isn't suported by PoW** because 1) bitcoin doesn't imposed a "reverse" time limit on it, and 2) manipulating it doesn't necessarily harm (and may increase) miners' profit motive.  MPT isn't a secure source of decentralized time. It's a lie to say bitcoin provides a secure decentralized source of time.  MPT is a permissioned consensus mechanism. 

To falsify the above, one of the following statements must be shown to not be true, or a fault found in the reasoning. It's verbose to make it easier to falsify. 

In short, miners should be able to find a way to profit more from holding MPT time back (due to its affect on Opentimestamps) than it reduces the present value of their equipment.

definitions:

"position" = existence, possession, and/or transfer.

1. Opentimestamps agglomerates attestations crucial to the position of value.
2. The agglomerations greatly reduce the fees paid per value positioned. 
3. Some attestations crucially depend on MPT time. Height & a single timestamp are not always useful options.
4. A hashrate majority can hold back MPT time.
5. The hashrate majority seeks to maximize net present value of their hashrate.
6. A hashrate majority could profit from making the attestations appear earlier in MPT time than real time. 
7. The profit from 6 can exceed the harm it causes to the majority hashrate's interest in the current and future BTC value on exchanges.

The above is proof of the first sentence. The following is proof of the second.

8. Bitcoin doesn't use monotonic timestamps on every block.
9. Lamport's 1978 paper on clocks and timestamps applies to all concepts of distributed consensus and it proves monotonic timestamps must be used. 
10. Monotonicity on every timestamp prevents a hashrate majority from being able to hold MPT back significantly unless it's >95% of hashrate or it orphans the honest hashrate's blocks.
11. At least 5% of hashrate will be honest.
12. The degree to which the majority hashrate can profit from 6 depends on the degree to which it can hold MPT back significantly. 
13. The majority hashrate's interest in the current and future BTC value on exchanges decreases in value as the number of orphans of honest miners increases.
14. Conjecture: Due to 12 and 13, the profit from 6 will not exceed the losses mentioned in 7. 

Monotonic timestamps would enable MPT (and therefore Opentimestamps) to be supported by PoW.  The goal of Opentimestamps is to have PoW decentrally support the claim that the stamps occurred at a certain point in real time or earlier, not that they definitely occurred before a certain time when they didn't.

The attestations my also change the movement of value based on a minimal or maximum *difference* in real time which the hashrate majority could thwart (initiate or prevent) by making MTP change a month in a single block or by holding MPT back.

There are two consensus mechanism at work in bitcoin when you don't have monotonicity. One is PoW based on single timestamps that dictate difficulty, not directly subject to the MPT. The other is a median of the block winners (MPT) which is a permissioned mechanism.

-------------------------

zawy | 2024-12-02 11:10:00 UTC | #60

If silence means everyone agrees MTP isn't secure, isn't supported by PoW, and shouldn't be used or recommended for any purpose that makes those assumptions, here's an alternative that doesn't require a change to bitcoin. Instead of using MTP, use the highest timestamp out of the prior 11 blocks and the current timestamp (or more simply, the highest timestamp seen). This would be at most 2 hours in future which is consistent with timestamping (whereas being in the past isn't). This has PoW support and is enforcing monotonicity. The highest timestamp seen would have better PoW support if the DAA changed every block which would increase the cost of miners delaying timestamps during the 2 week adjustment period.

Miners should enforce monotonicity even if it's not a requirement.

It's not a big problem in this sense: if timestamps are observed to monotonically increase, the MTP and the timestamps are valid and have PoW support.  The degree to which they don't have PoW support is limited by how far out of sequence they get. If it's a problems, it can be seen and the highest timestamp seen can be used. The problem is if code, script, apps, or legal contracts depend on MTP or current timestamps as accurate.

-------------------------

sjors | 2024-12-09 05:56:29 UTC | #61

Discussing and/or improving OpenTimeStamps seems out of scope for this discussion, maybe open a new topic for it?

-------------------------

zawy | 2024-12-09 17:23:48 UTC | #62

That's true, but my wider point is that MTP isn't a valid PoW timestamp as some scripts may assume.  Peter Todd informed me that he was aware of MTP not being good for timestamps back in 2016 and Opentimestamps therefore doesn't depend on it.

-------------------------

sjors | 2024-12-17 06:35:00 UTC | #63

I opened a separate discussion about https://delvingbitcoin.org/t/timewarp-attack-600-second-grace-period/1326

-------------------------

AntoineP | 2025-01-09 18:44:45 UTC | #64

Hi, i'd like to update this thread with the outcome of the discussions in the [private one](https://delvingbitcoin.org/t/worst-block-validation-time-inquiry/711?u=antoinep) on the topic of improving the worst case validation time.

Shortly after posting the details of the worst block, i realized i could adapt it to be valid under the [mitigations originally proposed](https://github.com/TheBlueMatt/bips/blob/7f9670b643b7c943a0cc6d2197d3eabe661050c2/bip-XXXX.mediawiki#specification) in 2019. From there we reconsidered possible mitigations and their tradeoffs in terms of impact, confiscatory surface and complexity. Thanks to everyone who participated and in particular to Anthony Towns for his contributions, corrections and the helpful discussions.

After studying the various options i believe the best way forward is to introduce a 2'500 per-transaction limit on the number of legacy (both bare and P2SH) input sigops. This provides a 40x decrease in the worst case validation time with a straightforward and flexible rule minimizing the confiscatory surface. A further 7x decrease is possible by combining it with another rule, which is in my opinion not worth the additional confiscatory surface.

-------------------------

AntoineP | 2025-01-29 21:33:58 UTC | #65

Regarding the duplicate coinbase fix, i think we should go with mandating the height of the previous block be set in all coinbase transactions' `nLockTime` field. The feedback i got from all miners i reached out to was that either of the fix being discussed was fine. Over other approaches, this one has the added advantage of letting one retrieve the coinbase's block height without having to parse Script.

If anyone has an objection or a good reason to prefer an alternative approach please let me know.

-------------------------

AntoineP | 2025-01-30 23:31:58 UTC | #66

Sjors convinced me of using a 2 hours grace period for the timewarp fix. Let me summarize the arguments put forth in the thread he opened and what led me to this conclusion.

One first illustration of his concern is with large powerful machines. He argues a hypothetical 3PH machine would need to roll timestamps by 10s every second. This could cause an issue when mining the first block of a period, if the miner of the previous block set its timestamp as far in the future as possible, and if the pool does not feed a new template to the miner more frequently than once every minute and 6 seconds.

This was addressed by Matt Corallo and Anthony Towns. Matt points out he's unaware of any miner that rolls `nTime` more aggressively than once per second, and that the existence of [old pool software that hard rejects rolling past 600 seconds](https://github.com/btccom/btcpool-ABANDONED/blob/e7c536834fd6785af7d7d68ff29111ed81209cdf/src/bitcoin/StratumServerBitcoin.cc#L384) makes that very unlikely. Further, he points out that in SV2 miners can roll the extranonce. So this hypothetical powerful ASIC could just do that when exhausting the 600 seconds.  Finally, Anthony Towns points out that as long as `nTime` is not rolled by more than 600 seconds the attack scenario is moot as the attacker's block would also be too far in the future and have been rejected in the first place.

Sjors then raises another concern about pool software ignoring the time in the Bitcoin Core provided block template and instead using wall clock time. He argues that while such software is technically broken today because of the MTP rule, it's unlikely to cause issues because nobody pushes time forward such as a block using current time would have a too far back timestamp. On the other hand, a pool using such software would be directly vulnerable post 600s-grace-period timewarp fix activation to producing an invalid first block of a period if the miner which produced the previous block set its timestamp as far in the future as possible.

I disagree we should make inferior protocol decisions to accommodate hypothetical software being ran by pools that would already be incompatible with today's consensus rules. Further, it's not clear the timewarp fix is worsening things that much since such software is already vulnerable today to having a timestamp lower than 6 or more of its previous 11 blocks. Contrary to the timewarp fix this can happen on any block and does not even require an attack scenario (misconfigured clock). So really the 600 seconds grace period only introduces marginal additional risks to a blatantly broken software that most likely nobody is running on mainnet with actual money on the line.

In addition to those concerns Sjors reminded me that Fabian Jahr initially [implemented the timewarp fix on testnet4 with a 2 hours grace period](https://github.com/bitcoin/bitcoin/pull/29775#discussion_r1704428831), and that it would make sense to keep this property of being able to use current time no matter the value of the previous block's timestamp (which may be up to 2 hours in the future).

Finally Sjors pressed me to consider the downside of a 2 hours grace period as opposed to a 10 minutes one. It would increase the worst case block rate increase from ~0.1% to ~0.65%. We would also lose the aesthetically pleasing property that with a 10 minutes grace period the block rate under attack would be equivalent to that originally intended: [one block every 599.9997 seconds](https://delvingbitcoin.org/t/timewarp-attack-600-second-grace-period/1326/32?u=antoinep) (vs one block every 596.7211 seconds with a 2 hours grace period).

I concluded that despite the fairly weak arguments in favor of increasing the grace period, the cost of doing so did not prohibit erring on the safe side.

-------------------------

harding | 2025-02-05 13:39:47 UTC | #67

[quote="AntoineP, post:64, topic:710"]
i believe the best way forward is to introduce a 2’500 per-transaction limit on the number of legacy (both bare and P2SH) input sigops.
[/quote]

In this accounting, how many sigops will `OP_CHECKMULTISIG` (CMS) count for?  IIRC, current accounting for bare scripts (which is only applied to output scripts) counts each CMS as 20 sigops, but accounting for P2SH redeem scripts makes the sigops equal to CMS parameter for number of pubkeys to check (e.g. `2 <key> <key> <key> 3 OP_CMS` counts as 3 sigops).

Also, to be clear, am I correct assume the 2,500 input limit will apply to CHECK*SIG operations specified in a bare prevout script?  E.g., spending a P2PK output will count 1 sigop towards the transaction limit.

-------------------------

AntoineP | 2025-02-05 15:30:36 UTC | #68

[quote="harding, post:67, topic:710"]
In this accounting, how many sigops will `OP_CHECKMULTISIG` (CMS) count for? IIRC, current accounting for bare scripts (which is only applied to output scripts) counts each CMS as 20 sigops, but accounting for P2SH redeem scripts makes the sigops equal to CMS parameter for number of pubkeys to check (e.g. `2 <key> <key> <key> 3 OP_CMS` counts as 3 sigops).
[/quote]

I will be sharing the specs i've been drafting soon (just wanted to cleanup and better test my implementation before). The intention here is to account in the same way as for P2SH/Segwit. The number of sigops in a CMS is the number of keys if it's inferior or equal to 16, and 20 otherwise.

[quote="harding, post:67, topic:710"]
Also, to be clear, am I correct assume the 2,500 input limit will apply to CHECK*SIG operations specified in a bare prevout script? E.g., spending a P2PK output will count 1 sigop towards the transaction limit.
[/quote]
Yes. The previous scriptpubkey, scriptsig + redeem script if P2SH all count toward the limit.

-------------------------

ajtowns | 2025-02-06 02:19:58 UTC | #69

I think it would be helpful if we could have distinct names for the two limits here -- keep "sigop" for signatures that appear in the p2sh, p2wsh and the output scriptPubKey, and use something new for the new limit ("sigchecked"? "legacysig"?)

[quote="AntoineP, post:68, topic:710"]
The number of sigops in a CMS is the number of keys if it’s inferior or equal to 16, and 20 otherwise.
[/quote]

The condition is whether the script is `OP_1` .. `OP_16` immediately followed by CMS, versus anything else, not strictly the number of keys. So eg `1 1 ADD CMS` will be treated as 20 sigops, not 2, and `0 0 0 CHECKMULTISIG` also counts as 20 sigops, rather than 0. It also counts CHECK*SIG opcodes from unexecuted branches, so `IF <P> CHECKSIG ELSE <Q> CHECKSIG ENDIF` counts as 2 sigops while  `IF <P> ELSE <Q> ENDIF CHECKSIG` counts as 1.

-------------------------

sjors | 2025-02-06 08:38:10 UTC | #70

Also keep in mind that pre-SegWit sigops are multiplied by 4.

But if you introduce a new name then maybe also don't bother with that multiplication, since it _only_ applies to pre-SegWit spends.

-------------------------

Chris_Stewart_5 | 2025-02-09 14:04:49 UTC | #71

One thing to note of disallowing 64 byte transactions on the network is that this will not work well if we ever decide to move away from a 256 bit hash digest for our merkle tree structure.

For instance if we decide to move to a 512 bit digest, we would now disallow 128 byte transactions, which by my count ~300,000 occurring in the bitcoin blockchain. A 1024 bit digest would disallow 256 byte transactions, of which there are ~450,000 occurring in the bitcoin blockchain.

More generally with this approach it seems if have a `N` byte digest, we cannot allow `N*2` transaction size? 

Ig if such a drastic change is needed, maybe we propose just reworking the entire merkle tree structure.

-------------------------

sipa | 2025-02-09 18:29:57 UTC | #72

[quote="Chris_Stewart_5, post:71, topic:710"]
One thing to note of disallowing 64 byte transactions on the network is that this will not work well if we ever decide to move away from a 256 bit hash digest for our merkle tree structure.
[/quote]

The entire reason why 64-byte transactions are problematic is because Bitcoin's txid Merkle tree has a design flaw that doesn't distinguish between internal nodes and leaves. If anyone ever realistically considers changing the Merkle tree design, like changing the hash function, they should start from a sane design that's not broken in the first place.

-------------------------

Chris_Stewart_5 | 2025-02-26 17:50:27 UTC | #73

# **Characteristics of a 64-byte Transaction**

I recently proposed [a Bitcoin Improvement Proposal (BIP)](https://github.com/Christewart/bips/blob/3f68df0ca172e2523afa809eb5d572b5a8881393/bip-XXXX.mediawiki) to make 64-byte transactions **consensus-invalid** in Bitcoin. This document examines the characteristics of 64-byte transactions.

## **Background**

According to [Suhas Daftuar](https://github.com/Christewart/bips/blob/3f68df0ca172e2523afa809eb5d572b5a8881393/bip-XXXX/2-BitcoinMerkle.pdf), 64-byte transactions follow this format:

* version (4 bytes)
* vin size (1 byte)
* outpoint (36 bytes)
* length scriptSig (1 byte)
* scriptSig (0–4 bytes, depending on the scriptPubKey in this transaction)
* sequence (4 bytes)
* vout size (1 byte)
* value (8 bytes)
* length scriptPubKey (1 byte)
* scriptPubKey (0–4 bytes, depending on the scriptSig in this transaction)
* locktime (4 bytes)

## **64-Byte Pre-Segwit Transactions Cannot Contain a Digital Signature in the `scriptSig`**

Since the activation of [BIP66](https://github.com/bitcoin/bips/blob/cc81fde2731546dd9185590c72661d7d620f7919/bip-0066.mediawiki#der-encoding-reference) on the Bitcoin network, digital signatures must be at least 9 bytes long.

As a result, a 64-byte pre-segwit transaction cannot spend raw scripts that contain:

* `OP_CHECKSIG`
* `OP_CHECKSIGVERIFY`
* `OP_CHECKMULTISIG`
* `OP_CHECKMULTISIGVERIFY`

## **64-Byte Transactions Must Create an `ANYONECANSPEND` Output**

No known `scriptPubKey` is 4 bytes long while still being protected by a public key or a supported hash function in Bitcoin Script.

Therefore, every output created by a 64-byte transaction can be trivially claimed by miners. If the goal of the transaction is to create an output claimable by miners, this can be achieved with Bitcoin transactions either smaller or larger than 64 bytes.

## **Nonstandard Outputs**

As of block `00000000000000000001194ae6be942619bf61aa70822b9643d01c1a441bf2b7`, there are no non-standard, non-zero-value outputs that could be satisfied exclusively by a 64-byte transaction.

## **P2SH Outputs**

P2SH outputs place a redeem script in the `scriptSig`. We can allocate up to 3 bytes for the `redeemScript` when spending a P2SH output.

As of block `00000000000000000001194ae6be942619bf61aa70822b9643d01c1a441bf2b7`, there are no UTXOs in the blockchain with `redeemScripts` of 0–3 bytes.

## **SegWit Outputs**

[BIP141](https://github.com/bitcoin/bips/blob/cc81fde2731546dd9185590c72661d7d620f7919/bip-0141.mediawiki#user-content-Witness_program) fundamentally restructures Bitcoin transactions. It introduces a new data structure called a **witness** that can replace the `scriptSig` for SegWit programs. This data does not count toward the 64-byte transaction limit, meaning digital signatures can be included in 64-byte SegWit transactions.

### **Native SegWit v0 and v1 Programs**

64-byte transactions that spend native SegWit programs must have exactly 4-byte `scriptPubKeys`. This is because the inputs to their programs are put into the witness rather than the `scriptSig`.

As a side note, when running tests for this document I realized it is [impossible to broadcast 64-byte transactions, even with `-acceptnonstdtxn=1`](https://github.com/bitcoin/bitcoin/blob/e486597f9a57903600656fb5106858941885852f/src/validation.cpp#L798) via the RPC interface without custom-compiling `bitcoind`.

### **Wrapped SegWit Programs**

There are two types of wrapped SegWit programs:

* `p2sh(p2wpkh)`
* `p2sh(p2wsh)`

Both types of outputs require witness programs in the `scriptSig`, which are larger than 4 bytes. Therefore, these types of outputs cannot be spent by 64-byte transactions.

### **Future SegWit Versions**

As per [BIP141](https://github.com/bitcoin/bips/blob/cc81fde2731546dd9185590c72661d7d620f7919/bip-0141.mediawiki#witness-program), this is how a witness program is defined:

> A `scriptPubKey` (or `redeemScript` as defined in BIP16/P2SH) that consists of a 1-byte push opcode (one of `OP_0,OP_1,OP_2,...,OP_16`) followed by a direct data push between 2 and 40 bytes gets a new special meaning.

If a BIP disallowing 64-byte transactions is activated, we will no longer allow **1-input, 1-output** SegWit transactions paying to 2-byte witness programs.

Here is an example of a witness program that would no longer be possible in a **1-input, 1-output** transaction:

> `OP_2 0x02 0xXXXX`

I am not aware of any reason why this would be a problem, but I have not seen it documented anywhere.

-------------------------

