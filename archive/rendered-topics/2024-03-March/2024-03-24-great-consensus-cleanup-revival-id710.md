# Great Consensus Cleanup Revival

AntoineP | 2024-03-24 19:53:27 UTC | #1

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

It's interesting to consider both how it worsens the situation and what it enables concretely. After all, miners can always hold back the timestamps even without exploiting the timewarp vulnerability.  But without setting the timestamp of the first block of a period before the timestamp of the last block of the preceding period, taking advantage of this necessarily involves bumping the difficulty back up. Therefore such an attack wouldn't achieve much: at best it could allow to mine 3 retarget intervals in 5 weeks instead of 6. On the other hand by setting the timestamp of the first block of a period below the one of the preceding period's last block an attacker can continuously take advantage of the low difficulty while continuing to reduce it.

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

AntoineP | 2024-04-05 15:32:17 UTC | #12

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

Yes, it also seems worthless to fix on its own. The author didn't take the time to suggest a "fix", but changing this behaviour (besides just disabling it) would be a hard fork. The obvious backward compatible alternative is to have a new script version application developers can opt in to use. Good news: it exists, it's Taproot. Furthermore i don't think anybody reported having an issue with this since it was first discussed more than a decade ago. In fact it is so much not worth fixing that even Segwit v0 [explicitly did not](https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki#cite_note-7).

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

