# P2share: how to turn any network (or testnet!) into a bitcoin miner

VzxPLnHqr | 2025-11-07 03:46:16 UTC | #1

Hi everyone,

I have been thinking about this topic for a while and wanted to share it here for discussion.

*TL;DR:* I wonder if there is some under-investigated territory where we can have efficient decentralized mining ~~pools~~ networks and use these networks as testing beds for new features. No mainchain changes required.

Feedback very welcome!

# p2share

A **p2share** network is a distributed bitcoin mining ~~pool~~ network with transferrable rewards (shares).

### **A mining pool which is actually a network of its own.**

Imagine a bitcoin mining ~~pool~~ network that operates very similarly to bitcoin itself:
* The network has its own consensus rules and "full nodes" which set and enforce the rules.
* The network rewards miners with "shares" when they create a valid block.
* Shares are transferrable among network participants (shareholders).
* Additional consensus rules, with different or experimental features, may also be in place.
     * For example, the network might use a mechanism like `sighash_anyprevout`, `op_cat`, or other covenant tools.
     * The network might have adjusted block size limits.
     * It may even support confidential transactions.
* **Most importantly, when the network mines a valid mainchain bitcoin block, the entire bitcoin reward is assigned to a *single, randomly selected shareholder*.**[^why_random_shareholder]  
     * Rather than distributing rewards to everyone (or using a centralized service), the coinbase output is constructed to pay out to the one eligible share.
     * This design keeps the bitcoin coinbase output compact, reducing transaction bloat and maximizing revenue from fee-paying bitcoin transactions.

### Example: a sharechain network

Let us call the above contemplated network a "sharechain" network. From the perspective of bitcoin mainchain, it appears **simply as a mining pool**. However, the sharechain enforces its own consensus rules, and one particularly unique rule concerns how bitcoin rewards are paid out.

Here is the process in more detail:

1. **The Network Operates like Bitcoin:**  
   Imagine the sharechain is built in close analogy to bitcoin:
   - It has a 10-minute block time.[^block_generation_rate]
   - It starts with 50 shares per block with a halving schedule like bitcoin's
   - The resulting deterministic “shrinking-pie” effect mimics bitcoin's supply schedule.[^shrinking_pie]

2. **Miners Construct Their Block According to Dual Rules:**  
   Mary, a sharechain miner, works on a valid bitcoin mainchain block template of her choosing. She selects the bitcoin transactions but must organize the bitcoin coinbase output under strict sharechain consensus rules:
   - **Key Rule:** The coinbase output must pay the public key corresponding to the owner of a specific share called `q`.

3. **Random Selection of the Reward Recipient:**  
   Suppose the sharechain is currently at block height `i` (i.e. all miners are searching for sharechain block `i+1`). Then, as part of constructing the bitcoin block template:  
   - A value `q` is determined using a function similar to:
     
     `q = sha256("q selector" || sharechain_blockheader_i) mod n`

     where $n$ is the total number of shares outstanding at block `i`.

   - **Clarification:**  
     This calculation employs a tagged hash that guarantees a **uniform, pseudorandom selection** among all shares. Every sharechain node will compute the same $q$ from the shared consensus state and agree on which share (and its owner) is designated to receive the bitcoin reward.[^exclude_immature_sharebase]
     
4. **Assigning the Bitcoin Reward:**  
   Mary looks up the designated share (at position $q$) in the sharechain's UTXO set:
   - Each share in the UTXO set is ordered canonically, ensuring consistency.
   - The bitcoin coinbase output is constructed to pay the public key associated with the selected share.
   - **Important Outcome:** Even if Mary mines the block and collects the share reward (the sharebase output), the bitcoin reward itself is not automatically hers. Instead, it goes to the owner of share $q$. The probability that Mary receives the bitcoin reward is precisely $v / n$ if she owns $v$ out of $n$ shares.

5. **Why is This Random Payout Mechanism OK?**  
   - **Fairness in Expectation:**  
     Although each block’s bitcoin reward is given entirely to one randomly selected shareholder, when considered over many blocks, every share is rewarded proportionally to its prevalence in the total supply. In expectation, it works much like a pro-rata distribution.
   - **Economical and Efficient:**  
     By assigning the payout to a single share, the bitcoin coinbase output remains small. This efficiency leaves more room for fee-paying transactions, maximizing revenue.
   - **Trustless and Decentralized:**  
     The random selection is completely deterministic and enforced by consensus rules. No centralized authority or federation is needed, preserving the trustless nature of both the sharechain network and bitcoin.
   - **Incentive Alignment:**  
     Miners are motivated to accumulate shares since owning a higher fraction improves their odds of winning the bitcoin reward, whether during active mining or while holding shares as a speculative asset.

### Some Observations

**Impact on the Miner:**  
If Mary holds all the shares, then the system essentially reverts to solo mining bitcoin. However, as soon as the share ownership is spread among various parties, there is a chance ($1 - v/n$ for Mary when holding $v$ of $n$ shares) that a randomly chosen share could belong to someone else. Over time, however, the expectation is for rewards to be distributed fairly in line with each miner’s share of the total.[^assign_to_self]

**Speculative and Functional Benefits:**  
Even if a miner ends up not winning a particular bitcoin reward, owning shares means that, over time, every share represents a chance at the reward. This quality may add an intrinsic or speculative value to shares, possibly fostering a secondary market—beneficial for atomic swaps or other network functionalities.

## Open Questions

* ### Is it incentive-compatible to mine on this chain?

In our approach the shares are issued in a fashion similar to bitcoin where earlier miners will obtain more shares per block than later miners. Maybe this tweak alone is enough to render the approach viable? or to render it non-viable? It is hard to know at this point in our analysis.

* ### Is a sharechain a sidechain?

A consequence of having a deterministic share supply is that the internal unit of account of the sharechain network will not be 1:1 with bitcoin. 

This lack of parity is upsetting to many bitcoiners who seek the holy grail of a trustless-2-way-peg and may be enough to label this approach as antithetical to bitcoin, label the sharechain network as an altcoin, or otherwise consisder it as an "attack" on bitcoin. However, these actions say little about whether it is incentive-compatible (i.e. rational) for a bitcoiner, or any sha256D-capable miner for that matter, to mine on the sharechain. 

Is a sha256D-capable miner better off simply mining mainchain bitcoin directly? Maybe? Maybe not?


* ### What happens if the features offered by a sharechain network network are incorporated into mainnet bitcoin?

This is an interesting question. If the feature is incorporated (or expected to be) incorporated into mainnet bitcoin, then it seems logical that there would be very little demand for the sharechain shares. Hence, the hash rate of the sharechain network would be small/negligible and the expected value (in sats) of the shares would be negligible.

On the other hand, if it is highly unlikely that the feature will ever be incorporated into bitcoin *and* there is demand for the feature, then it seems logical that there could also be significant demand for the sharechain shares. The expected value of the shares (in sats) in this case might be non-negligible.

Perhaps the most interesting case is one where the feature in question is not likely to be activated on mainnet until well after the sharechain network is expected to win a mainnet block (remember, the sharechain network is essentially a mainnet mining pool). In this manner the price (in sats) of the shares coupled with the sharechain network's current hashrate can serve somewhat as an informational proxy for the wider bitcoin ecosystem. It might help answer the question, "when will this feature be activated on mainnet?"

* ### Is it an altcoin?

Is what is described above an "altcoin"? If the definition of altcoin is a network which uses a different unit of account than bitcoin, then, yes sharechains may be considered altcoins as in this case the shares are acting as the unit of account within the sharechain network. 

However, it is important to remember that unlike forks like BCH, BSV, and really any other PoW network, which split thermodynamic work across incompatible chains and hence detract from the security of mainchain bitcoin, these sharechain networks are in fact, as a core component of their own consensus, valid bitcoin mining networks and are therefore contributing to, not detracting from, the security of mainchain bitcoin.

* ### Is it any threat to bitcoin mainnet?

Probably not, and it is not intended to be. Bitcoiners who want to use the sharechain network can trustlessly obtain shares by mining on the sharechain or purchase them via atomic swaps. For everyone else, the sharechain is simply a mining pool from the perspective of
bitcoin mainchain, and as such it contributes to the security of bitcoin.

* ### Will bitcoin mainnet miners want to 51% attack the sharechain networks?

Maybe? But they will lose regular mining revenue in the process. Eventually mainchain bitcoin miners stopped even caring about the BCH/BSV forks, and that makes sense because those forks are incompatible with mainchain bitcoin. BCH and BSV consensus rules do not add to the PoW security of mainchain bitcoin. Rather, they detract from it. 

These sharechain networks are different in that the network itself is actually trying to mine a mainchain bitcoin block. They lose both the mainchain bitcoin rewards *and* their sharechain network rewards if the bitcoin block is invalid. Therefore, these sharechain networks are in fact as bitcoin-friendly as they can be, and hopefully are less centralized than traditional mining pools. Everybody wins?

* ### Will bitcoin mainnet miners provide protection from unfriendly sha256D miners?

Maybe there is some game theory which would indicate that it is in the best interest of all bitcoin-friendly sha256D miners to "protect" these sharechain networks if they are ever under PoW attack.


## Conclusion 
**In summary:** p2share keeps bitcoin’s conservative security model intact but offers a fresh incentive mechanism that is fair, efficient, and aligned with the idea of a decentralized, innovative ecosystem.

---
(Thank you for reading this far! You can probably stop reading now, but if you are curious, here are more notes and background[^notes_background] and a thought experiment[^thought_experiment])

##### Footnotes
[^p2pool]: see https://en.bitcoin.it/wiki/P2Pool and various attempts at [rebooting p2pool](https://github.com/pool2win/p2pool-v2) as well as attempts to [scale miner share payouts using payment channels](https://bitcointalk.org/index.php?topic=2135429.0) rather than huge coinbase transactions.

[^shrinking_pie]: There are other ways to create a shrinking-pie effect, some of which even include a notion similar to tail emmission (thereby ensuring block subsidy will always be a non-zero number of shares), but lets skip that discussion for now. The point is that there is the effect of a finite and deterministic supply of shares, and there are multiple ways to achieve that effect. Here we simply chose one that we are all familiar with because it is nearly identical to bitcoin itself.

[^block_generation_rate]: Why such a long time between blocks? The parameters chosen for the example consensus rules above are just that: an example. However, by targeting a block generation rate which is similar, or longer, than bitcoin, we can reduce the sharechain orphan rate and keep our decentralized mining pool competitive. Stratum V2 can be used both within the sharechain network as well as at the interface between sharechain and mainchain. Let us not worry too much about these particular details right now. Also, from a UI/UX perspective a layer 2 network (e.g. lightning or similar) can be used to reduce the friction of longer block times.

[^assign_to_self]: Mary could, of course, try to game the system and find a nonce for her block (block `i+1`) which would cause the sharechain network to "choose" her as the assignee of the bitcoin reward for the next block (block `i+2`). This may be a problem in the beginning when the sharechain network difficulty is extremely low. However, as soon as the sharechain difficulty finds equilibrium this attack becomes expensive in that it will require significant extra work to pull off. That work is probably better spent simply mining according to consensus which often will mean assigning a shareholder other than onesself as the would-be-winner of the bitcoin block reward. Afterall, for every valid sharechain block Mary mines, she *does* get to assign herself the shares!

[^exclude_immature_sharebase]: Technically the selection of `q` from the utxo set should probably exclude sharechain coinbase utxos which have not yet matured, but let us skip that detail here for now.

[^notes_background]: Basically there are two problems that we are trying to solve here. The first is that it is extremely frustrating when otherwise good ideas (and people who want to work on them) are basically relegated to "go do it on an altcoin." So, somewhat like drivechain (but without needing a soft fork), it would be nice to provide a mechanism where those people can still at least hang around the bitcoin ecosystem rather than go enrich a competitor.
    - The other thing is that, in the opinion of this author (currently at least), if there is any argument to believe about the "energy waste" of PoW, it is actually to do with the plethora of PoW algorithms. PoW not applied to bitcoin is the extra waste. PoW applied to bitcoin is not waste. Therefore, it would be better if all PoW networks were merge-mined with bitcoin (and from the perspective of bitcoin appear simply as a mining pool) as then at least all PoW networks are adding to, rather than detracting from, bitcoin's security.
    - In other words, any sha256D mining which is not applied to bitcoin (or any PoW mining at all for that matter -- from a thermodynamic perspective) is wasted/misallocated use of energy. The idea here is an effort to curtail that, by giving people/networks a better way to do it -- **just turn your sh*tcoin network into a bitcoin mining pool and call it good.**
    - Imagine if ethereum started as, and continued today (and didn't switch to PoS), just a bitcoin mining pool. It could still have its own unit of account (eth) which is what we are calling "shares" here, and of course there could have still been a bunch of sh*tcoinery on it, but at least if it were a sharechain, it would be adding hashrate to bitcoin which is good.
    - Of course it is important that mainchain bitcoin itself stay extremely conservative. That is the beauty of the type of thing we are discussing here. Bitcoin itself can stay as-is and does not need to know or care about any of the things we are talking about here. **From the perspective of Bitcoin, valid hash power is valid no matter its origin** (whether a centralized mining pool, this p2share idea, solo mining, ...).

[^why_random_shareholder]: Why pay out to a randomly selected shareholder rather than pro-rata to all shareholders?
    - One reason why P2Pool[^p2pool], a decentralized bitcoin mining pool, is no longer used for mainchain bitcoin mining is because its coinbase transactions were much larger than trusted-third-party mining pools. As such, there was less room for fee-paying transactions, thereby decreasing expected pool revenue.

    - The coinbase output for P2Pool was large because each eligible shareholder was paid out in the coinbase whenever the pool successfully mined a block. Here we take a different approach and instead pay out a single shareholder selected uniform pseudorandomly. This keeps the bitcoin mainchain coinbase transaction extremely small, thereby maximizing the ability to capture revenue (bitcoin transaction fees).

    - Paying out to a randomly selected shareholder is also a way to avoid requiring a federation of some sort (static or dynamic) to take custody of the btc rewards and distribute them. **Paying out the entire btc reward to the owner of a randomly selected share is equivalent, in expectation, to paying out pro-rata.** It also as an interesting effect of endowing some sort of future value to a share, thereby (possibly) helping to establish a lower bound on a market price (in sats) per share. Such a market price might be useful for parties wishing to complete atomic swaps betweeen sharechain and mainchain bitcoin.

[^thought_experiment]: Technical details aside, the bigger issue that we are trying to resolve right now is whether the economics of p2share even make sense. Is it ever rational for anybody to even play by p2share rules when mining bitcoin, versus just using the established mining pools or solo mining bitcoin.
    - The primary, and possibly novel, "tweak" in the design here is is the idea of a blockchain (the p2share sharechain in this case) assigning its bitcoin reward to a random shareholder. We can take a step back, and think about what that might have looked like within bitcoin itself. 

    - For example, imagine if, at genesis in 2009 the rules for bitcoin mainchain were actually: miners assign all rewards to the owner of a pseudorandomly selected satoshi. 

    - It is weird to think about for sure. If bitcoin operated that way, it seems to change the economics quite a bit at first glance, but it is hard to be certain that the change is long-lived or even adverse. Maybe it is, but maybe it is not.

    - But let us think about it ... let us rewind to 2010-2013 or so. Imagine you wanted to mine back then, but owned not a single satoshi at the time. With the hypothetical rules we are contemplating, you could still mine of course -- it is just that there would be a 0% chance that you are assigned the reward (this is because you don't own any sats, so you are not represented on the ledger when the random sat is chosen).

    - But this may not be enough to actually kill the network. Rather, the behavior around mining would have just changed. For example, in that weird world I just described, it would behoove you to buy or otherwise obtain a bunch of sats before you start mining so as to increase your chances of becoming that lucky shareholder.

    - Also, we are setting aside the bootstrapping problem of: if nobody owns sats and the ledger is empty, then how does a prospective miner even buy sats -- here let us just assume that satoshi hard-coded that he was the owner of the first 50btc -- then by the rule proposed here, he would also be the assignee of the next block reward with 100% possibility, and this would remain true until he sold/transferred some sats to somebody else, and so on. This may seem like the ol' "pre-mine" trick in disguise, and it sort of is, but not really. Only the first block reward need be hardcoded, the rest is still determined via consensus. Over time that first block reward becomes a de-minimus proportion of the total bitcoin outstanding.

-------------------------

garlonicon | 2025-11-07 09:30:26 UTC | #2

> A consequence of having a deterministic share supply is that the internal unit of account of the sharechain network will not be 1:1 with bitcoin.

It could be, if there is a need to do so. It is all about proportions: in Lightning Network, there are separate units called “millisatoshis”, that can exist only there, but still, every satoshi has 1:1000 coverage. If in this network, there will be different “millisatoshis”, and they will represent some different, internal splitting of amounts between all participants, then it could still work, and be covered with real coins. Because after all, it is all about presigned transactions, in that way or another, which will settle all of that on-chain.

> Is what is described above an “altcoin”?

As long as it is compatible with Bitcoin, and you work on the same chain, then it is not an altcoin. If the implementation will not make it an altcoin, then it will be just yet another usage of Bitcoin.

> Will bitcoin mainnet miners want to 51% attack the sharechain networks?

It can be prevented, if mining a sharechain block will be as hard, as mining a Bitcoin block (or harder). It is possible to make a network, where attacking the sharechain would be comparable to attacking Lightning Network, and for example confirming old state of the channel on-chain. Also, penalty transactions, and other things, which LN invented, can be used as an additional protection, if needed.

> Why such a long time between blocks?

Well, 10 minutes per block is not “long”. For some use cases, even longer block times could be desired, for example for DNS records. Also, if someone wants for example 30 seconds per block, then things like P2Pool can be built on top of such system, so it is not a problem, if the main implementation will require 10 minutes or more per block.

-------------------------

VzxPLnHqr | 2025-11-07 20:18:38 UTC | #3

Thanks for reading and for your reply. 

> A consequence of having a deterministic share supply is that the internal unit of account of the sharechain network will not be 1:1 with bitcoin.
>> It could be, if there is a need to do so. It is all about proportions: in Lightning Network, there are separate units called “millisatoshis”, that can exist only there, but still, every satoshi has 1:1000 coverage. If in this network, there will be different “millisatoshis”, and they will represent some different, internal splitting of amounts between all participants, then it could still work, and be covered with real coins. Because after all, it is all about presigned transactions, in that way or another, which will settle all of that on-chain.

I think there may be a misunderstanding. Nothing in the p2share design, as currently presented, has anything to do with presigned transactions. That being said, I think you may be suggesting something different?

### favorite p2share idea: privacy as a mining pool (mimblewimble anyone?)

A p2share network which I think would be very useful for bitcoiners would be one which implements a privacy layer. In such a design, the sharechain would have mimblewimble-like semantics. Then privacy could easily be achieved, without leaving the bitcoin ecosystem, via atomic swaps between mainchain btc and the sharechain.

The problem I am currently unable to solve (which is why I post it here in case anyone here knows how):

**In a MW-like sharechain network, how can we pseudorandom-uniformly select a shareholder's public key?**

Selection of the public key of a random shareholder (weighted by number of shares) is easy in a UTXO system like bitcoin's where amounts are publicly associated with UTXOs. This was demonstrated in the above p2share writeup. However, recovering this criteria for a sharechain with MW-like semantics, without stripping MW of its privacy benefits, is non-trivial.

**EDIT: for MW-like semantics, I believe the answer to this is "no."** However we could perhaps imagine a sharechain with consensus rules such that each transaction output is required to be of equal amount, and coinbases are required to also have multiple outputs, each of equal amount. Then **all sharechain transactions, from a blockchain analysis standpoint, will appear as sharejoins, and this might satisfy the privacy criteria.**

-------------------------

ZmnSCPxj | 2025-11-09 06:07:07 UTC | #4

[quote="VzxPLnHqr, post:1, topic:2093"]
A value `q` is determined using a function similar to:

`q = sha256("q selector" || sharechain_blockheader_i) mod n`

where $n$ is the total number of shares outstanding at block `i`.
[/quote]

Note that the probability of getting the reward on the Bitcoin blockchain is proportional to the number of shares I own on the share network.  In particular, we should note that it is quite possible for a grinding attack to be performed, where I ensure that, when I am mining sharechain blockheader i + 1, I make sure that sharechain blockheader i + 2 will be be paid to me (and so forth in future mines).  If I secretly own much of the shares *and* more than 50% of the hashrate on the share-network, I can induce minor miners who join the share-network to effectively contribute all mining rewards to me, effectively taking over their hashrate.  This leads to a bootstrapping problem: why would anyone actually want to mine on the share-network I started, when it is entirely possible that I secretly own most of the shares and most of the hashrate?  And if there is not enough *other* people who want to join this new share-network, then there is no way to actually move from "the starting operator is in a position to steal all hashrate of all new joining miners" to "the shares are distributed enough that it is a non-issue" because nobody *else* will rationally mine on this.

In my opinion, the central error of proof-of-stake is the assumption that you cannot grind  the random selection of the winning staker, and this scheme evokes the same error; the above hash you show ***can*** be ground, especially as `n` will start out small, to always go to some target participant; this of course converts proof-of-stake to proof-of-work-with-extra-steps, which is undesirable. The more complex things become, the more it is possible that there is some hidden optimization that can massively disrupt the mining industry (the way ASICBOOST was disruptive and was implicated in the delay of SegWit activation).

That is, this scheme might end up just being solo-mining-with-extra-steps.

-------------------------

cmp_ancp | 2025-11-09 14:44:47 UTC | #5

I really found out this idea of a “pool sidechain” very elegant, though don’t know exactly how it would work. That is, the idea of a blockchain is itself a way to coordinate multiple nodes in a descentralized manner, and thinking in that way, a descentralized pool could be based on it.

Maybe we could have miners that are mining BTC blocks, but these blocks are, at the same time, commiting to a sidechain. Maybe the commiting part could be at the nonce, or in the coinbase. Actually, the miner could commit to a Merkle tree of multiple values.

Even if a miner fails on finding a block, maybe he could share weak blocks to the sidechain, prooving that he is indeed spending power. These sidechain commits could also serve as a proof that his templates send the coinbase to the pool.

The sidechain could have faster blocks, and could also be a lattice, I don’t know. Miners don’t need to sync to the entire sidechain, once the commits would be available on the BTC chain.

Just gathering a bunch of ideas, don’t really know how these could glue together, but at least I think someome smarter than me could have an insight lmao. Also, if this were to be glued together, there must be a way to limit bandwidth usage or unecessary computation. The system overall cannot be a burden to the miner, to the point of making mining with this pool disadvantageous.

-------------------------

ZmnSCPxj | 2025-11-09 15:51:03 UTC | #6

The concept is a revival of the old "P2Pool" design, which has a separate "sharechain".  By my memory (it may be inaccurate) the coinbase committed to multiple low-difficulty shares, i.e. actual full blocks whose block difficulty was lower than the difficulty of Bitcoin, but higher than the P2Pool share difficulty.  The innovation of ***this*** OP is to randomly select a lucky shareholder, instead of having one output per shareholder as in the original P2Pool.

The problem with ***this*** OP is that the whole point of a pool is precisely to manage luck: instead of some enormous windfall going to some miner, a bunch of cooperating miners pool their hashpower and divide the large block reward amongst themselves via some policy (some measure of their contributed hashpower).  So it does not quite provide the desired properties needed for a true pool.

P2Pool could not scale up because each hasher who contributed in the last N shares to the pool had to have an output on the coinbase, which was untenable.

In a non-P2Pool pool, typically the issue is that there is a central coordinator.  The central coordinator aggregates the multiple tiny shares that each hasher provides, and pays out to them as larger aggregated payments.  The problem is that the central coordinator can discredit the share that a hasher provides if the central coordinator wants to censor some transaction that the hasher submits in their work --- i.e. the new Stratum is insufficient as the central coordinator can always deny paying for the work submitted by a hasher after it discovers that the hasher has included censored transactions (and the coordinator still gets the coinbase rewards anyway).

What we ***actually*** need is a Lightning-like construction to aggregate the multiple small payments that a P2Pool-like pool builds, but we cannot use HTLCs; instead, we need a SCRIPT with comparison of large 256-bit numbers (difficulty comparison), ***and*** some way to ensure that the committed transactions in a block (which would need to be some kind of ZK-proof, or else the pool coordinator can inspect the block of the hasher before paying out --- we need the requirement that the pool coordinator pays out only if the hasher includes valid transactions, but cannot learn the transactions until after it has paid out, similar to how HTLC preimages work in Lightning: once the hasher has given out the transactions to the coordinator, the coordinator cannot "take back" the amount, in much the same way that once the preimage of an HTLC has been published, the HTLC-offeror cannot "take back" the HTLC).  Unfortunately, we cannot compare 256-bit numbers (`OP_GREATERTHAN` et al only work on 32-bit numbers) and in any case, the validity check cannot be performed (though it may be possible to generate a BitVM for it? Not sure: the big issue with BitVM is that in practice its logic is the inverse of what we want, i.e. it is a "get paid if invalid, until a timeout" system, not a "get paid if valid, until a timeout" system).

Probably the above would require changes to Bitcoin.  Unfortunately, large custodians already have much of the economic majority in Bitcoin --- and centralized non-P2Pool pool coordinators ***are*** custodians, full stop, because they hold the money that hashers have rightfully earned, which allows them undue power over the hashers --- and there is no incentive for custodians to allow non-custodial uses that would erode their use (which explains why there are astroturfed disinformation and distraction campaigns that prioritize other things --- particularly censorship, the most important power that custodians get --- over noncustodial-supporting changes like `OP_CTV`).

-------------------------

VzxPLnHqr | 2025-11-09 21:11:28 UTC | #7

Thank you for reading and for your replies. I was actually hesitant to tag this thread as mining, because, to me at least this is less about mining and more about the overarching economic and philosophical consequences (if any). You seem to have gleaned that too, so thank you!

> The concept is a revival of the old “P2Pool” design, which has a separate “sharechain”. By my memory (it may be inaccurate) the coinbase committed to multiple low-difficulty shares, i.e. actual full blocks whose block difficulty was lower than the difficulty of Bitcoin, but higher than the P2Pool share difficulty. 

I think your memory is correct here with regard to how P2Pool functioned. The concept presented here of of paying out to a random shareholder to save mainchain blockspace and thereby maximize fee revenue might have bought P2Pool some more time, but it is probably more nuanced than that ...

> The innovation of ***this*** OP is to randomly select a lucky shareholder, instead of having one output per shareholder as in the original P2Pool.

I think there are a couple subtle things that are worth mentioning:

First, P2Pool did not (to my memory at least) have transferable "shares." Adding this feature into the mix, coupled with a fixed schedule of *fair*[^fair_issuance] share issuance, is what may provide for the sharechain network to be longer lived itself.

Second, if you have a longer lived sharechain network(s) which is symbiotic with bitcoin, then such a network would be a great testing ground for new features, uses, services, etc -- basically anything for which bitcoiners may have demand for, but for which mainnet bitcoin is not suitable. 

Whether the shares themselves are able to obtain and maintain any "value" greater than `0.00000.... sats/share`, is currently an unknown. 

Yet, it is important to remember that many of the arguments we may come up with for why there is no value here, are almost identical to what could have been said about bitcoin itself. Now, I am not suggesting that any of these sharechains, if they existed, would or even could eclipse bitcoin.

Rather, if such mining networks existed, and even a small fraction of overall bitcoin hashrate was directed through such networks (and rewarded commensurately via the p2share mechanism contemplated here), then the value of those shares relative to bitcoin and those shares relative to one another (assuming multiple networks) does have at least _some_ information signal which is relevant to the bitcoin ecosystem.

> If I secretly own much of the shares *and* more than 50% of the hashrate ... this of course converts proof-of-stake to proof-of-work-with-extra-steps, which is undesirable

This actually gets to the crux of the issue, and why I posted here in the first place. While I agree with you in spirit here, I am not sure exactly why it is undesirable, especially if we extend your definition of "proof-of-stake" here to include "any non-bitcoin-denominated investments" (including equity investments wall street companies), then why would we _not_ want to convert those things into our beloved bitcoin-compatible proof of work?

Of course, we can argue that all those types of investments/investors need to just eventually learn the hard way and ultimately obtain bitcoin directly somehow which, outside of theft or gifts, leaves only two options: mine it or buy it. This doesn't change that. 


> ... That is, this scheme might end up just being solo-mining-with-extra-steps.

This is a good thing. Well, the extra-steps is perhaps annoying. However, the argument I am trying (and quite likely failing!) to make here is that this is the feature. The fact that it, in expectation, the p2share model economically degrade down to solo mining (albeit with some extra steps) is exactly why this concept is interesting:

**p2share gives bitcoin hodlers a (very risky! but trustless!) mechanism to buy "equity" (shares) which represent future mining earnings, if any, of a given p2share bitcoin-compatible mining network.** 

So, at least in theory, rather than buy wallstreet shares of wallstreet companies which are purporting to help bitcoin, but really only help wallstreet, and the fiat mindset, this is a way to keep it all in our dysfunctionally resilient bitcoin ecosystem.

> What we ***actually*** need is a Lightning-like construction to aggregate the multiple small payments that a P2Pool-like pool builds, but we cannot use HTLCs; instead, we need a SCRIPT with comparison of large 256-bit numbers (difficulty comparison),

I too came to this conclusion back when I was exploring worklocks [^sigpow]. It is ironic that bitcoin is itself distributed via guarded PoW (i.e. proof of work is the first pass, but success is also guarded by other criteria such as timestamps, ...), yet it is not trivial to do try to do the same thing within bitcoin itself (namely distribute a UTXOs sats to unknown ultimate beneficiaries, but with a possibly different set of guarding criteria -- at least without requiring an upgrade to Bitcoin).

> Probably the above would require changes to Bitcoin. Unfortunately, large custodians already have much of the economic majority in Bitcoin ...

Precisely why something like p2share may be helpful/useful? No changes to Bitcoin necessary.

### on p2share incentives

If we set aside the bootstrapping problem for a moment. I am not sure how such a network is bootstrapped. Maybe it is luck, or benevolence, or something else. Incidentally, I believe nonce grinding would only be an issue while the sharechain network difficulty is low, and here we are explicitly skipping that phase of the contemplated sharechain's history and teleporting to a period where difficulty is high enough and competition for shares is tough enough that the extra grinding is not worth it. That may be too unrealistic a simplification, but let us go with it for now.

A first-pass analysis hunch says that:

1. _if_ participants value their bitcoin and Bitcoin's future _and_ have determined that p2share is one way to try to bolster that future (i.e. so they buy shares of various sharechain(s) with features they desire/use/agree with, and direct their mining power to such sharechain(s))
2. _then_ a sharechain miner will likely want to obtain/maintain a balance of shares which is commensurate with the her hashrate relative to the _sharechain_ -- in other words, the "extra steps" she takes are simply to protect herself from diluation.

In some ways, such a sharechain miner might be lauded for this behavior since:
   * she is actively securing the bitcoin network and protecting its future
   * she is doing it in a trustless and decentralized way but (presumably, by her choice of sharechain(s)) with a set of like-minded participants, thereby sending a signal to mainchain of their desired future features..
   * her behavior is similar to someone who tries to obtain mainchain bitcoin (via buying or mining) and continues to accumulate -- so it gracefully degrades to a behavior we generally want to encourage.

But this all very informal so far and **maybe the p2share concept is a dead end?**

[^fair_issuance]: The reference escapes me, but I think there is a paper somewhere which proves that, at least for certain reasonable notions of fairness, issuance via proof-of-work is the optimal issuance mechanism.

[^sigpow]: https://github.com/VzxPLnHqr/sig-pow (note: there are some flaws in the code/construtcion, it was just an exploration)

-------------------------

ZmnSCPxj | 2025-11-09 23:15:13 UTC | #8

> then such a network would be a great testing ground for new features, uses, services, etc – basically anything for which bitcoiners may have demand for, but for which mainnet bitcoin is not suitable.

IMO if a feature cannot get into mainnet Bitcoin anyway, it is largely pointless to fantasize about it or "test" it.  First we have to demonstrate that Bitcoin has ***not*** ossified yet; then there is non-zero value in having a testing ground (and more specifically, *another* testing ground).

> Yet, it is important to remember that many of the arguments we may come up with for why there is no value here, are almost identical to what could have been said about bitcoin itself

The big advantage of Bitcoin was that it existed prior to any Bitcoin existing, and for much of its early days, people involved in it were doing FAFO on what was basically valueless tokens.  Nowadays you have to somehow displace the position of Bitcoin.

It appears to be non-obvious that when the whitepaper says "longest chain of proof-of-work wins" it cuts across even incompatible rules and so-called "altcoins"; there can only be really one Bitcoin, at least in the local space (dunno about spaces distant enough that even speed-of-light transmission has massive latency).

With that said --- it would theoretically be possible that miners of Bitcoin can follow additional rules that they impose on themselves and some certain other participants in some sub-network, but which they do not impose on participants outside of the sub-network (but which are still participants in the existing public Bitcoin mining network).  This is in principle something like a "softer" version of a softfork ("softerfork" tightening of rules on self and sub-network of the entire network, but not on the network as a whole, compared to "softfork" tightening of rules on the entire network), and such a "softerfork" would probably be able to implement what you have in mind (i.e. Bitcoin blocks of miners participating in this p2share sub-network commit to an extra non-Bitcoin block that transacts a different currency, the "share", that has different semantics).

-------------------------

jungly | 2025-11-10 05:29:55 UTC | #9

Interesting ideas here. What we are doing at P2Poolv2 is less revolutionary. Dropping a line here for completion.

1. More or less replicate the p2pool design.
2. There are three big changes though:
   1. use weak compact blocks as shares to avoid the transaction broadcasting that p2pool had to do in the original implementation.
   2. Instead of a linear chain use uncle blocks to scale the pool to larger number of miners.
   3. Add bitcoin Script, coinbase and transaction system to sharechain. This allows coinbase of constant size to support a larger number of users. 20 large miners get paid in the coinbase, the other smaller ones trade their shares with the large miners or market makers using atomic swap between sharechain and bitcoin. The market makers or the larger miners buy the smaller miner’s share at a discount, and the market pricing takes care of the discount amount.

We have published some gists as the design evolved, but we’ll soon share a proper write up once the code has reached MVP. A lot of decisions slightly change when we actually write the implementation. Meanwhile you can follow the development here: https://github.com/p2poolv2/p2poolv2/

-------------------------

VzxPLnHqr | 2025-11-10 21:26:38 UTC | #10

> It appears to be non-obvious that when the whitepaper says “longest chain of proof-of-work wins” it cuts across even incompatible rules and so-called “altcoins” ...

Yes, I agree this is non-obvious to many people. It is also why I want to be careful to point out that in the p2share model described here, the work applied to a sharechain is, by definition, work applied to mainchain bitcoin.

> With that said — it would theoretically be possible that miners of Bitcoin can follow additional rules  ...

I believe what you just described here is isomorphic to the underlying concept of p2share. So, if we think about the abstract notion of a "softerfork" (I like the term!), then I currently am, hesitantly, positing that the "select a random shareholder" rule might be the closest concrete instantiation of that abstract concept which (bootstrapping problem aside) approaches incentive compatibility.

Though, I think we should also clarify that it need not be existing bitcoin miners who come together to launch a softerfork (using p2share semantics or otherwise). Even if the sharechain only comprises an extremely small portion of mainchain hashrate, a group of dedicated bitcoiners with minimal collective hashpower could launch one. 

For the same reason that it is non-trivial to answer question, "why would someone ever want to mine on sharechain rather than solo mine mainchain directly?," it is also difficult to answer, **"why would existing, or otherwise larger, miners want to attack the sharechain?"** 

If larger miners wanted to just troll sharechain and cause a bunch of reorgs, well, at least unlike proof-of-stake, sharechain is in fact still a proof-of-work chain, so the attack still has an irreversible external cost. Further, each hash applied to attack the sharechain (thereby abiding by the sharechain consensus) _could have in fact been applied to mainchain_.

Unfortunately there probably are trolls in the world who would spend resources just to regularly wreak havoc (reorg) the sharechain. However, stalwart shareholders and prudent sharechain design parameters might still weather such storms. The best part is that even in this circumstance the trolls/attackers are still securing mainchain bitcoin throughout each and every attack! 

Now, for larger miners who are not trolls but merely trying to maximize their profits in whatever units of measurements their hearts desire, even if such miners are not interested in the sharechain initially, they may become interested if, by some bizarre aligning of moons, there are ever market arbitrage opportunities between shares and mainchain btc large enough to justify the "extra steps" of capturing (and thereby closing) those opportunities.

Knowing the above behavior is looming, it seems the price per share (in sats of course!) is naturally bounded on the upside. As such we need not worry about  sharechain(s) ever "overtaking" bitcoin. Sharechains might gobble up some hash rate from the centralized pools, but that is not a bad thing.

It still begs the question of whether a "softerfork" sharechain with p2share-like semantics will ever have sustained price per share greater than `0.000000... sats/share`.

> What we are doing at P2Poolv2 is less revolutionary. ... This allows coinbase of constant size to support a larger number of users. 20 large miners get paid in the coinbase, the other smaller ones trade their shares with the large miners or market makers using atomic swap between sharechain and bitcoin. The market makers or the larger miners buy the smaller miner’s share at a discount, and the market pricing takes care of the discount amount.


@jungly Thanks for chiming in! In the OP I linked to your repo so that people might also be aware that there is at least some attempts to revitalize (some of) the original P2Pool concepts.

I have some questions about your most recent design iteration, ~~but I will ask them in a different thread so that we can keep this one a little bit less implementation-specific (for now at least!).~~ (edit: could not find an existing thread for your p2poolv2):

1. Is it still custodial in the sense of using a FROST federation or similar? Asking because one reason why I am interested in alternative designs is specifically for them to remain non-custodial, which is in fact possible, as demonstrated here with p2share.

2. What are the motivations for a market maker or large miner to buy the smaller shares? What is the opportunity cost to them if they do not buy the smaller shares?

3. Does this mean that the shares in your current design are long-lived such that a small miner can obtain shares and then come back in the far future and try to sell those shares to a larger miner? or do the shares in your current design somehow expire?

-------------------------

jungly | 2025-11-11 07:59:21 UTC | #11

[quote="VzxPLnHqr, post:10, topic:2093"]
could not find an existing thread for your p2poolv2

[/quote]

We haven’t discussed it here yet. I’ll start something once I have an MVP ready to show.

Thanks for looking through and raising very relevant questions.

[quote="VzxPLnHqr, post:10, topic:2093"]
Is it still custodial in the sense of using a FROST federation or similar? Asking because one reason why I am interested in alternative designs is specifically for them to remain non-custodial, which is in fact possible, as demonstrated here with p2share.

[/quote]

No. The idea is top 20 miners in any PPLNS window get rewards directly in the bitcoin coinbase. The other, smaller, miners have a coinbase on the sharechain that they sell to the larger miners / market makers using atomic swaps. To support such transactions, and any other future constructions using Ark/Spark/.\* we support Script based transactions in the sharechain.

[quote="VzxPLnHqr, post:10, topic:2093"]
What are the motivations for a market maker or large miner to buy the smaller shares? What is the opportunity cost to them if they do not buy the smaller shares?

[/quote]

There are a couple of motivations for the large miners.

1. They can buy the shares from smaller miners at a discount. The discount is determined by the market.
2. The other motivation is to stay in the top 20 and get it a payout directly from the coinbase, which has it’s benefits.

The risk they take on is that they buy some shares for a PPLNS window and p2pool doesn’t find a block for that PPLNS window.

[quote="VzxPLnHqr, post:10, topic:2093"]
Does this mean that the shares in your current design are long-lived such that a small miner can obtain shares and then come back in the far future and try to sell those shares to a larger miner? or do the shares in your current design somehow expire?

[/quote]

Yes, they expire. This is because a share is valid only for a PPLNS window. Once a share is past the PPLNS window it won’t be accounted for in the PPLNS reward payout by the pool.

This makes the cost clear for the market maker, they have to bet on the pool’s hashrate and decide the discount they want for specific shares, given the probability of finding a block in any given PPLNS window. This risk reduces with increase in the pool’s hashrate and also with increase in the size of the PPLNS window. The original p2pool used a PPLNS window of about three days. We are considering making this as large as a week. Not set in stone yet, need to run some analysis.

-------------------------

VzxPLnHqr | 2025-11-11 19:37:16 UTC | #12

Thanks for providing some more details regarding your current design parameters. Here are some not-very-formal thoughts. I hope they are helpful:

> The discount is determined by the market....

The total discount factor `d`, a number in the range [0,1], can be broken into two components `d = d_time*d_params`:
1. `d_time`: time-value-of-money considerations -- sats available now (via atomic swap) are worth more than sats which are (expected to be) available later. 

2. `d_params`: protocol-specific constraints/parameters such as PPLNS window size, share expiry, etc.

Ideally we want a market which will price at `d = 1.0` which basically would mean that large miners and market makers would pay essentially full price to scoop up the shares of the smaller shareholders. Since time is inescapable, it is likely that `d_time` will almost always be less than `1.0`. That being said, with a strong track record, good future prospects, and ample liquidity, `d_time` may approach `1.0`, and that would be great. That is what we want.

However, if we choose the parameters incorrectly (note: I am not saying you are choosing them incorrectly here, mostly just trying to brainstorm with you), we run the risk of the effect being that `d_params` is going to max out at some value less than 1.0.

You already indicated that you believe PPLNS window size and share expiry will adversely affect the discount. So, what happens if we look at it in limit (unbounded window, no expiry)?

In the limit, I think you basically end up recovering the p2share design, but with a tweak: you are paying out only to the top 20 shareholders, rather than (in the p2share design) paying out to a randomly selected single shareholder. 

Of course we could expand p2share to randomly select 20 shareholders. But what I am trying to point out is that it is precisely the random selection process which pushes towards incentive alignment. It implies a risk that, even if you are a large shareholder, you _might not be selected immediately_, yet mitigates that risk because you can have confidence that _even the smallest shareholders will be selected eventually_ (so long as there is hashrate[^future_hashrate]). Therefore, it naturally enforces a 1 share = 1 share mentality, regardless of when the shares were originally earned. And that is why profit maximizing p2share miners have at least some motivation, but not the obligation, to also buy up other shares, especially when they come across price dislocations.

I am concerned that in your current design, the large shareholders and would-be market-makers are somehow being assumed to have the obligation to buy shares from the smaller miners, but the obligation is (of course!) unenforceable.

Attempting to think it through: 
* say your top 20 miners represent `p%` of your hash rate, but `q%` of outstanding shares (in the current PPLNS window). We cannot directly measure `p` (which is why shares exist in the first place!), so we want `q` to, in expectation, equal `p`.
* as I currently understand it, your current design will allocate `100%` of the mainchain reward to those top 20 shareholders _irrespective_ of whether/how/if those shareholders bought up shares from the smaller players

Now imagine you are contemplating being a miner/market maker for this. The problem (as I see it) is that if, through your trading actions, you cannot become one of those top 20 before the window closes, then all your hashes/trades affecting that window will irrecoverable losses. This is a result of the chosen parameters, and therefore is a cause of `d_params` being less than 1.0. 

A more ideal design:
1. for the top 20 shareholders which represent `q%` of shares outstanding would only be allocated `q%` of the mainchain bitcoin reward.
2. those shareholder's shares would be destroyed (because they have now been redeemed) and the remaining `(1-q)%` of the mainchain bitcoin reward would be paid out to the bottom `(1-q)%` of shareholders using some sort of off-chain mechanism.

Alas, I cannot figure out how to do the above without mainchain covenants and without some sort of custodial mechanism, and for both of those reasons I currently consider it a dead end. However you can probably see where this is going now: **can we approximate the above ideal design without requiring any changes to mainchain bitcoin and staying non-custodial?**

It may not fully or faithfully approximate it, but I do wonder whether randomly selected shareholders, non-expiring shares, and an unbounded PPLNS window (which is just another name for "deterministic share issuance schedule, coupled with difficulty adjustment algorithm"), might get the closest to recovering the desired ideal, and hence maximizing the marketability of the shares while remaining non-custodial and no-mainchain-upgrade-required.

### speaking of design parameters

@ZmnSCPxj While writing this post, it occurred to me that the OP which delineates the overall design of p2share does not explicitly state that when a payout happens (i.e. when that lucky shareholder is allocated the entire mainchain bitcoin reward), that the shareholder's shares are destroyed. 

Rather, in the OP I presented it more similar to a distribution of profits rather than a share buyback. The buyback/redemption model is probably the better model. However, I remember now why I arrived at the distributional design. It was because I could not solve the following problem cleanly: 

To actually implement the buyback model, the sharechain would need to carry, as part of its state, the current "price per share." This is so sharechain nodes can calculate how many of the selected shareholder's shares to buyback (destroy). Even if this calculation is somehow possible in an objective oracle-free manner (a tough, maybe impossible challenge in it of itself!), it triggers some other issues:

1. Overflow - what if the selected shareholder does not have enough shares to "sell" back? Where does the excess btc reward go then? Since we want to remain non-custodial, one idea would be to randomly select a second shareholder and buy their shares back, and so on, until the reward has been exhausted. With each new selected shareholder the mainchain coinbase output set grows, so we start losing our precious blockspace and we end up back at the original P2Pool bloated coinbase problem.

2. Market depth - even if we could have the sharechain objectively converge on a price per share and use it for these buybacks, it should also characterize market depth, and so then now we have to somehow emulate an entire order book inside sharechain...

That is when I stopped and switched to the "just distribute the entire reward to the randomly selected shareholder" model. But I suppose there is an unexplored third model of, "choose one lucky shareholder and buy/destroy all the shares in that share output, and assign that shareholder the entire mainchain reward." Naturally large shareholders would then split their share holding across multiple outputs. This would not affect their probability of being selected, but it would minimize how many shares they sell if they are selected. This behavior likely puts downward pressure on the share price (assuming there is any such price at all!), not to mention unnecessarily bloats the sharechain utxo set. Nevertheless, maybe there is something here.

---
[^future_hashrate]: whether there will be future hashrate, and how much, is accounted for in the `d_time` discount factor. It is a statement or expectation about the future, and hence about the time value of money.

-------------------------

VzxPLnHqr | 2025-11-19 16:58:08 UTC | #13

## on incentive compatibility (updated share issuance mechanism)

On further reflection, rather than tie sharechain issuance to a shrinking-pie schedule like Bitcoin (as proposed in the original p2share example above), we might be able to **recover incentive compatibility by simply issuing a constant number of shares per unit work.**

Bitcoin's base "unit of work" is 2^32 sha256d attempts. Now, given that in 2025 we have sha256d available ASICS right out of the gate, we should probably raise sharechain's "unit of work" to something higher. The actual value is not important, but a reasonable choice could be found empirically.

With this modification, the sharechain ledger will represent all contributed work over time. It is a fair  accounting because the sharechain still has its own difficulty adjustment algorithm.

The "select a random shareholder" technique from the original design remains unchanged.

For concreteness, let the share `S` issuance per sharechain block be something like `S = c*D_sharechain`, for some constant `c` which is set at genesis. 

As hash rate of the sharechain grows, the issuance of shares grows too, so there is inflation from a share supply standpoint. However, there is still always a chance that one of your early shares (from back when hash rate was small) will be the randomly selected share.

Similarly, when the hash rate of the sharechain drops, the number of shares issued per block will drop accordingly. 

Now, let us imagine a scenario where the sharechain hash rate grows 100x (say this happens across multiple difficulty adjustments even), and then stays there for some time, only to eventually drop down to the orignal 1x. Because the number of shares being issued is proportional to difficulty, the number of shares which were issued in the "high difficulty" period will _vastly_ out number the number which are issued during the "low difficulty" period.

However, individual miners aren't being disproportionately rewarded; the system is just scaling issuance to match the collective effort. It is a pure representation[^representation] of the notion of proportional fairness.[^proportional]

This might in fact recover incentive compatibility such that it is "just as rational" to mine bitcoin indirectly by mining on a p2share chain as it would be to mine bitcoin directly (solo or via pool). 

As @ZmnSCPxj elegantly put it earlier, this is "mining bitcoin with extra steps," but if it degenerates to an acceptable notion of incentive compatibility (in expectation), while remaining trustless and decentralized, then perhaps the extra steps may be worth it for many?


---

[^representation]: speaking of representations: yes, this does mean that if the sharechain starts at difficulty 1 and grows exponentially over time, that we might need larger than 64-bits to represent sharechain amounts. But it is also not 2009 anymore, so that is probably not too much of an issue.

[^proportional]: admittedly, not everyone agrees that proportional fairness is the "correct" notion of fairness in all circumstances. However, in a trustless, decentralized system like bitcoin, or this p2share mechanism, we can argue that it is the correct notion of fairness.

-------------------------

jungly | 2025-11-19 07:44:17 UTC | #14

It seems like the end result will be the same as PPLNS. Can you point out how it will be different, I might be failing to see some subtlety.

-------------------------

VzxPLnHqr | 2025-11-19 17:01:50 UTC | #15

[quote="jungly, post:14, topic:2093"]
end result will be the same as PPLNS.
[/quote]

Good observation! I believe the answer is mostly yes, but it is subtle:

If the `N` in "pay per last N shares" includes all shares ever issued _and_ there is a difficulty adjustment algorithm happening regularly _within_ the window so that each share represents the same amount of work, then yes I believe it will be the same, in expectation, as PPLNS.

This is a good thing. PPLNS broadly, in my understanding, is currently the most common mining protocol in use, and given that there are real costs to mining, we can infer that the continued use of PPLNS is the most incentive-compatible.

Some important differences here though, which, in my opinion are features of this model:

1. Without affecting mainchain Bitcoin we can smuggle in almost any desired extra functionality on the p2share sharechain network(s). This is made possible because sharechain rewards (shares) are transferable and long-lived.

2. The sharechain networks are, after all, valid bitcoin miners. Consensus changes within a given sharechain network can manifest directly, or indirectly, as mainchain miner signalling. This may be helpful for mainchain upgrade coordination while remaining more true to the spirit and practice of trustless decentralization.

    In practice in times when these sharechains account for only a trivial portion of Bitcoin's hash rate, the signals will likely be ignored/discounted. However, if these sharechains proliferate such that eventually the collection of sharechain networks represent a non-trivial portion of Bitcoin's hash rate, these signals may be worthwhile to consider.

### Some outstanding challenges and notes ### 

Open questions about the p2share model as it is currently being contemplated:

1. What to do about spam on the sharechain?

    Given that shares are long-lived just like regular bitcoin, a sharechain may suffer and have to overcome similar challenges that Bitcoin has to overcome. 

    The good news is that, by design, these p2share sharechains can themselves serve as testing grounds for trying out anti-spam mechanisms which are acceptable to the sharechain nodes.

    A sharechain with a track record of reckless features requiring hard-forks (of the sharechain) to fix might appeal to some, but not others. Whichever is the case, we can expect it will be reflected in the price (in sats), if any, that people are willing to pay to atomically swap in/out of those shares.

    A sharechain with a track record of pushing Bitcoin forward in a consistent fashion may experience some level of price stability and market liquidity for its shares. In other words, a sharechain which is consistently a good steward to its shareholders and to Bitcoin may enjoy a price premium for its shares relative to other sharechains.

4. What about ephemeral, purely speculative, sharechains?
    
    Why not? It is still Bitcoin mining and therefore still has real costs. Maybe the speculative sharechain is trying to explore a cutting edge feature and has no other way to do it and wants to remain within the Bitcoin ecosystem? Or maybe the people doing it are crazy. I suppose we can here speculate ourselves that the shares of such sharechains will be nearly, if not entirely, worthless -- until when/if they are somehow not.

5. Sharechain mergers?

    Suppose a speculative or otherwise ephemeral sharechain ends up demonstrating a feature which may be valuable to other sharechains?

    So long as the units of work are compatible, then it feels like it may be possible for one sharechain to absorb another (i.e. for the absorbing chain to hard-fork and recognize the work/shares of the absorbed chain). 

   Perhaps as part of the contemplated merger, the developers of the feature(s) of the speculative sharechain provide/backport an implementation of the feature(s) to the absorbing chain _and_ the merger is contingent upon the activation of that feature.

   Feels like a logistical nightmare, but still might be doable. At least Bitcoin itself can usually remain blissfully unaware of these things, yet these innovations can happen within the Bitcoin ecosystem (via these sharechains) rather than elsewhere.

-------------------------

VzxPLnHqr | 2025-11-20 01:43:27 UTC | #16

After re-reading this thread, it is clear that it would be difficult for a newcomer to fully come up to speed with what is being discussed here. Additionally, some of the design parameters have been refined. Below is an attempt to distill things into a more intuitive picture which is consistent with my current thinking about it.

_[Disclaimer: I used an LLM to help create this summary, but spent a lot of time carefully reviewing and editing it before posting here.]_

## The Story So Far

P2Share is a decentralized mining framework that builds on the legacy of P2Pool, enabling Bitcoin miners and non-miners to form symbiotic sub-networks called sharechains. These sharechains operate as valid Bitcoin mining pools from the perspective of the mainnet, contributing hash power to Bitcoin's security while introducing their own internal unit of account -- ransferable "shares." Unlike traditional altcoins or forks that dilute Bitcoin's proof-of-work (PoW) by splitting thermodynamic effort, sharechains align fully with Bitcoin's consensus, ensuring **all work bolsters mainnet security without requiring any changes to Bitcoin itself.**

### Core Mechanism of Sharechains
At its heart, a sharechain is a separate blockchain that miners participate in to collectively mine Bitcoin blocks. Miners submit low-difficulty shares to the sharechain, which adjusts its own difficulty independently. 

The hash of the sharechain blockheader is inserted within mainchain block template. The template is constructed by the sharechain miner in accordance with both sharechain consensus and Bitcoin consensus. Sharechain consensus specifies what the miner is to place in the bitcoin block's coinbase output (more on this later). Sharechain consensus also requires that the bitcoin block be valid according to Bitcoin consensus in every way except for possibly not meeting the Bitcoin difficulty requirement.

Within the bitcoin block, a sharechain miner is free to choose and include whichever bitcoin transactions she wants, so long as they are valid according to Bitcoin consensus. Within the sharechain, in addition to assigning herself the share reward, she is also free to choose and include whichever sharechain transactions she wants, so long as those transactions are valid according to sharechain consensus.

When the sharechain's combined effort eventually produces a valid Bitcoin block, the Bitcoin reward (subsidy plus fees) is assigned not pro-rata to all participants, but _to a single pseudorandomly selected shareholder._ At sharechain height `i`, sharechain miners are working to find sharechain block `i+1`. The selection for which share is chosen is derived from the sharechain's most recently mined block, using a mechanism like `q(i+1) = sha256("q selector" || sharechain_blockheader_i) mod total_shares_outstanding`. In expectation, this is equivalent to proportional distribution, as the probability of selection scales with shares owned.

To minimize Bitcoin mainnet block space usage, which was a key limitation of original P2Pool design, the coinbase transaction outputs to just one address: that of the selected shareholder. This maximizes room for fee-paying transactions, enhancing overall revenue potential. Shares themselves are minted and distributed via the sharechain's PoW, and they can be traded trustlessly via atomic swaps with Bitcoin, potentially establishing a market price in satoshis.

### Incentive-Compatible Share Issuance
The design aims for incentive compatibility by issuing a constant number of shares per unit of work, rather than following a fixed halving schedule like Bitcoin. Specifically, share issuance per block is `S = c * D_sharechain`, where `c` is an emperically determined genesis constant and `D_sharechain` is the current sharechain difficulty. This ties issuance directly to contributed work, creating inflation in share supply as hash rate grows, but preserving proportional fairness across all historical work.

- **Why this recovers incentive compatibility**: Each share represents a consistent unit of PoW effort, adjusted for difficulty changes. Early shares retain value because the random selection draws from the entire ledger, giving them ongoing chances at Bitcoin rewards even as the network scales. In scenarios of hash rate fluctuations (e.g., a 100x increase followed by a drop), issuance scales accordingly, ensuring no disproportionate rewards; miners are compensated based on collective effort over time.

- **Alignment with existing models**: This mirrors Pay-Per-Last-N-Shares (PPLNS) in expectation, where `N` encompasses all shares ever issued, and difficulty adjustments normalize work per share. PPLNS is widely used in mining pools due to its proven rationality, balancing real costs like electricity with predictable payouts. Here, miners rationally participate if their share holdings match their hash rate contribution, protecting against dilution while mining Bitcoin indirectly.

- **Nodes set and enforce sharechain consensus**: Just as with Bitcoin, not all sharechain nodes need be mining nodes. Just as with Bitcoin, consensus is set and enforced by the nodes collectively, and only a subset of those nodes are mining nodes. _A fully validating sharechain node is also, by definition, a fully validating Bitcoin node._ Notably, however, because any sharechain node with a nonzero share balance has a chance to receive the Bitcoin reward, there is a modicum of incentive to run a sharechain node.

This setup degrades gracefully to solo mining "with extra steps," but the extras provide trustless decentralization: miners avoid custodial risks of centralized pools, and there is incentive to hold shares as "equity" in future earnings. This is somewhat akin to buying equity in a Bitcoin-aligned venture, but more transparent, borderless, and argubably just plain better -- but still risky! Shares are merely a probabilistic claim on the future Bitcoin earnings of the sharechain, if any. This is not very different from equity in a startup.

### Benefits for the Bitcoin Ecosystem
Sharechains enhance Bitcoin without threatening its conservative core. They serve as testing grounds for innovative features, such as new scripting, anti-spam measures, or L2 integrations that might not suit mainnet, yet information about which may be nonetheless helpful. Consensus changes within a sharechain can signal preferences and/or readiness to Bitcoin via miner commitments, fostering decentralized coordination. There is also information in relative sharechain prices. For instance, multiple sharechains could emerge with distinct rules, and their relative share prices (realized via atomic swaps) would reflect market demand, thereby providing valuable signals without taking any hash power away from Bitcoin.

Ephemeral or speculative sharechains are viable too, as they still contribute PoW to Bitcoin at real cost. Successful innovations could lead to mergers, where one chain absorbs another's shares and features via a hard fork, backporting value while keeping everything within the Bitcoin ecosystem. Spam is managed internally, with reckless chains likely facing devalued shares, rewarding prudent stewardship.

From a game-theoretic view, sharechains deter attacks: 51% attempts on a sharechain cost attackers mainnet revenue, and Bitcoin-friendly miners may defend them to preserve overall security. Grinding attacks (e.g., nonce manipulation to self-assign rewards) become uneconomical at equilibrium difficulty, as the extra work outweighs benefits.

In essence, P2Share offers a fair, efficient path to decentralize mining further, channel experimental energy back into Bitcoin, and convert potentially wasted PoW into additive security, **all while remaining fully compatible and incentive-aligned with Bitcoin.** This could empower Bitcoiners to innovate symbiotically, strengthening the network's resilience without compromise.

-------------------------

VzxPLnHqr | 2025-11-25 01:46:55 UTC | #17

It may have already been clear to others, but the whole idea can be reasonably generalized. Though, at this point what is being discussed may be more appropriate for the #philosophy or #economics forums? 

Regardless, I tried to work up a concise but somewhat plausible example to demonstrate:

### Share Issuance as a Tool for Bitcoin Alignment

#### Example: Hard Money Coffee

Different choices of the issuance function `S(D_sharechain)` which may even depend on other inputs so long as they are "consensus observable" from the perspective of a sharechain node, can be experimented with on a per-sharechain or per-project basis, without impacting Bitcoin consensus. Such endeavors need only **publicly commit to a simple policy such as: all net profits will be used to either (a) mine-then-burn or (b) buy-then-burn shares of that sharechain.**

This policy is then enforced indirectly via market mechanisms around the sharechain ecosystem, which is a subset of the Bitcoin ecosystem.

As a concrete example, imagine a global decentralized coffee phenomenon, **Hard Money Coffee** ("HMC"), launching its own sharechain. At genesis, a large block of shares which are, initially, ineligible for the "select a random share" mechanism, could be created and assigned to a multi-sig “operations” address controlled by founders and/or stewards. These “operations shares” would function much like startup equity: they could be pledged as collateral to obtain working capital (ideally in sats of course, but also potentially as machinery, leases, or other contributions), while ongoing PoW mining creates a separate stream of newly issued shares according to the chosen `S(D_sharechain)`. The ongoing mining of the sharechain as well as atomic swaps to/from Bitcoin provide a real time and globally accessible information signal about the HMC phenomenon.

Founders or stewards of HMC might still want, for liability or operational reasons, to create one or more traditional legal entities—e.g., **HMC Inc., a Delaware corporation**. They can capitalize such an entity with sharechain shares (so that HMC Inc. can sign contracts, pledge collateral, lease property, hire employees, etc.), while remaining bound by a public commitment such as: “HMC Inc. will use its net profits to mine/buy-then-burn HMC sharechain shares.” The legal entity then looks and acts conventional to courts and counterparties, but economically it is just a steward that channels value back into the sharechain according to this commitment. From the perspective of the sharechain protocol, however, HMC Inc. is just one node among many: it controls some share addresses and may contribute hash rate, but it has no special consensus privileges.

HMC Inc. never custodies anyone else’s sharechain shares and does not maintain any off-chain register of “anonymous” beneficial owners; ownership and transfers are tracked natively on the sharechain itself. Because the HMC sharechain is a p2share sharechain, even the occasional Bitcoin block rewards it earns are automatically and non-custodially distributed on-chain to the selected share, never touching HMC Inc.’s balance sheet, which is win‑win‑win for incentive alignment, operational efficiency, and keeping the necessary “backward compatibility” surface with legacy legal/compliance systems as small and simple as possible.

The presence of such entities does not break the model, because market pricing of sharechain shares continuously reflects the perceived probability that stewards will honor their commitments. If there is even a hint that HMC Inc.’s stewards are going rogue—for example, diverting profits away from mining or buying-and-burning shares—the market will start discounting sharechain shares. Since HMC Inc.’s balance sheet is largely composed of pre-mined, not-yet-eligible sharechain shares (plus whatever other assets it has accumulated), this discount directly erodes its apparent net worth and indirectly pressures its other assets, founders, and stewards, in addition to the reputational damage. The p2share mechanism thus gives maximal flexibility to interoperate with legacy legal structures, while preserving bitcoin-aligned incentives and providing clear, on-chain signals for detection and evaluation of risk and reward.

Crucially, those pre-mined operations shares can be made **ineligible** for the “select a random share” mechanism at first, via consensus rules that simply exclude them from the eligible set. A later, consensus-enforced upgrade could then make some or all of them eligible, but only once Hard Money Coffee has reached a level of profitability where it is actually following through on its public commitment to mine/buy-then-burn shares out of business profits. It is natural to expect markets to price this path-dependence: so long as the enterprise’s success and honesty are uncertain, operations shares should trade at a steep discount; as evidence mounts that the venture is profitable and is indeed burning shares as promised, the discount should narrow, with the eligibility upgrade itself likely treated as a major priced-in milestone.

Over time, if the HMC sharechain becomes widely held, heavily used, and HMC Inc. (and others) are reliably buying and burning shares out of real‑world profits, the protocol need not keep issuing large amounts of new shares. In fact, `S(D_sharechain)` can be designed from genesis to taper issuance toward zero as the chain matures, without relying on any discretionary off‑chain judgment about HMC Inc.’s fortunes. In such a steady state, miners are compensated primarily by fees, while buy‑and‑burn activity supports the share price and thus the value of those fees. HMC Inc. remains, from the protocol’s perspective, just one sharechain node among many that happens to hold shares and perform work; it has no special authority over issuance rules beyond what is mechanically encoded in consensus.

-------------------------

cmp_ancp | 2025-11-25 03:31:51 UTC | #18

I want to point some problems I think we need to solve in order to the system to stand up.

* How much should we decrease the share of the winner? The winner (i.e., the selected owner that receives the fees on BTC onchain) should have some of its shares burnt, otherwise, they would just gather shares forever. Should we burn it all? Or is there a calculatable proportion? Some actor may find out sometimes that they have a lot of shares, and if they try to mine more sharecoin blocks, they may be spending more than it is valuable from a BTC block revenue (assuming they burn all accumulated shares). In that situation, it is prefereable to they to stop mining and wait untill some block is mined in their name. Maybe a solution would be to sell their shares in order to remain in a sustainable position.
* How frequent should be the sharecoin block issuance? And how should its difficulty vary? If the difficulty is always too high, small miners would never receive fair shares to their wasted energy. In that case, we should make the blocks more frequent. However, if blocks are too frequent, we may face propagation issues and flooding in the network, making not worthy to participate.
* How many txs should we be able to put on a sharecoin block? And how complex would be the scripts (if it was based on scripts at all)? This touches in the previous point, the propagation issues and cost to sustain it.

Being optimistic, at least new miners shouldn’t necessarily sync to all chain history. BTC mainchain is already prooved by work, and sharecoin state is dependent on BTC state. We could commit to actual state in found blocks, and new miners would need to sync to the last checkpoint, behind that, they could trust the previous state prooved on mainchain.

Throwing away part of syncing and storage problems, the sharecoin chain could grow in a faster pace, everything from one or two checkpoints behind could be prunned. Maybe the pace itself could vary from epochs, depending on number of miners and number of shares issued.

We should tune all of that in order to maintain some goals. Should we be small miner friendly? The ideia is to have a single sharecoin pool, or multiple ones, focused on different miner sizes, each one with its own difficulty? I think we need to be attractive to big miners in order to make a change in centralization, but small miner friendness is also important to decentralization.

-------------------------

VzxPLnHqr | 2025-11-25 05:08:28 UTC | #19

Thanks for your reply.

[quote="cmp_ancp, post:18, topic:2093"]
How much should we decrease the share of the winner? The winner (i.e., the selected owner that receives the fees on BTC onchain) should have some of its shares burnt
[/quote]

I alluded to this problem in a [prior post](https://delvingbitcoin.org/t/p2share-how-to-turn-any-network-or-testnet-into-a-bitcoin-miner/2093/12). There are some technical challenges with trying to do what (I think) you are suggesting, but I also do not think it is necessary. Yes, the same share on the sharechain might, in the long run, happen to be the winner of multiple bitcoin block rewards. That is fine. The market can price that in.

[quote="cmp_ancp, post:18, topic:2093"]
How frequent should be the sharecoin block issuance? And how should its difficulty vary?
...
How many txs should we be able to put on a sharecoin block?
[/quote]

The p2share design is extremely general in the sense that all of these parameters can of course be tuned. Additionally, depending on the underlying goal/purpose of the specific sharechain, even the issuance schedule and eligibility-for-random-selection of the shares can be tuned (see my prior post about the hypothetical "Hard Money Coffee" sharechain).

Naturally a sharechain can itself have its own lightning-like channels. As such, atomic swaps between between Bitcoin and a sharechain would be instant since both sides could use lightning.

[quote="cmp_ancp, post:18, topic:2093"]
Should we be small miner friendly?
[/quote]

Like Bitcoin itself, I think it probably makes the most sense to keep the operational costs of running a fully validating sharechain node (which is also, by definition, a Bitcoin node) minimal. 


[quote="cmp_ancp, post:18, topic:2093"]
The ideia is to have a single sharecoin pool, or multiple
[/quote]

Multiple, competing on everything from feature sets (per the original post) to more exotic endeavors (per the "Hard Money Coffee" example).

-------------------------

cmp_ancp | 2025-11-25 18:15:12 UTC | #20

[quote="VzxPLnHqr, post:19, topic:2093"]
I alluded to this problem in a [prior post](https://delvingbitcoin.org/t/p2share-how-to-turn-any-network-or-testnet-into-a-bitcoin-miner/2093/12). There are some technical challenges with trying to do what (I think) you are suggesting, but I also do not think it is necessary. Yes, the same share on the sharechain might, in the long run, happen to be the winner of multiple bitcoin block rewards. That is fine. The market can price that in.

[/quote]

I’ve read the linked post, and I got confused by what you are saying. The post aludes to burning all the shares of an account (and it has an interesting case, pointing out to the possibility of dividing shares in different accounts in order to remain profitable), but now, you say about the same share winning multiple times. Maybe it is a name confusion, what is a share? Is it different from an account? Is the share the “coin” or the “account”?

In any case, I am really pro bruning the selected account shares. That’s because, if large miners doesn’t get your shares burnt, we could have a centralization of shares to fewer hands, and those miners would win all the time. If the winner is always burnt, the small miner that isn’t selected could, in theory, on the long run, accumulate shares untill it is a valid candidate to be selected.

Inflation is also somewhat of a problem. If, in the long run, sharecoins are always devaluated, shares of small miners could be outpriced and never paid. Sure, if the small miner, on the long run, is contributing little to the total hash as a whole, it is fair that the reward would shrink until the next found block.

I should also point to the possibility of large miners pulverizing their shares on multiple accounts. That would totally break the incentives, once the miner doesn’t diminish its probability on being chosen, while they guarantee to burn little to no share. Maybe we can make an account only elegible to selection after some threshold, and miners with less shares would profit by selling them to the elegible ones. Or maybe UTXOs could only be consolidated, never broken down, or frequently refreshed? It’s concerning that, even in that situation, large miners could generate multiple small sized shares and never consolidate them. Maybe the probability on being chosen could grow exponentially with share size rather than linear, so \[p(s1+s2) > p(s1) + p(s2)\], setting a penalty to pulverization.

Another cool feature that could be managed by sharecoin is coordination to onchain txs. Imagine some ark style payout tree, instead of needing a coordinator to manage the signatures, the sidechain itself could be a place to cordination. Multiple parties could commit asynchronously to signatures to construct such thing, using the faster pace the sidechain grows and the ephemerity of the state (nodes doesn’t need to store all history), maybe signature commitments could have some expire time for the coordination to be complete? In that way, nodes could get rid of them after some time.

PS: re-read the linked post, and now understood that you were proposing different paying models lmao. Sorry, I’m working today, early in the morning, my mind isn’t fully focused yet (and english isn’t my native language). But I remain with my point: if shares are never burnt, then we can have a centralization and smaller miners could never get paid. Also, i think the exponential probability based on share quantity is a clever way out, we could scale down the exponentiability in order to not get an explosion. Maybe something like \[x log(x)\] can suffice, we only need the proposition \[p(s1+s2) > p(s1) + p(s2)\] to be true.

-------------------------

VzxPLnHqr | 2025-11-25 19:14:25 UTC | #21

Part of the confusion here is my fault. The original post presented a less generalized version of p2share but where the share issuance was more similar to Bitcoin's (shrinking-pie model). Then, later in the thread I presented a more refined version which, I believe, is equivalent to PPLNS in expectation. Then, to confuse things more, I presented the most general version where the "parameters" (shape of the share issuance schedule, block time, difficulty adjustment algorithm, etc) can all be fine tuned.

Now, to get on the same page with regard to nomenclature, when I refer to "shares" I am here referring to the sharechain's unit of account. You can imagine a sharechain as its own network which operates similar to bitcoin, and just like bitcoin has output amounts (sats) in its utxos, a sharechain has output amounts (shares) in its utxos. 

Due to the "select a random shareholder" mechanism of p2share, _unlike_ bitcoin itself, these shares possess an equity-like property that we might associate with more traditional for-profit enterprises. That is why I am calling them shares. Though, of course, it is important to remember that a sharechain is not itself any sort of "legal structure" in the traditional sense. In this manner it _is_ like Bitcoin.

You seem to be imagining a sharechain shareholder having an "account" more similar to how some networks use an account-based methodology for tracking amounts rather than a utxo methodology. While it might be possible or even preferable to use that sort of mechanism within the sharechain, that is a technical detail that, in my opinion, does not really matter right now in our discussion.

[quote="cmp_ancp, post:20, topic:2093"]
In any case, I am really pro bruning the selected account shares.
[/quote]

I am not sure if this is where you are getting confused or not, but we are not necessarily selecting an "account." We are randomly selecting a specific share. That specific share will (in a utxo model) be located in a certain utxo which will have a certain public key associated with it. That is all we, as sharechain nodes, know. Conveniently, that is also all we need to know.

Sure, we could "burn" the entire utxo which contains that share, but that suffers from a number of problems, some of which you have pointed out. A miner can easily minimize her number of burned shares by pulverizing her shares across many outputs.

Set all the technical concerns aside for a moment and assume that the sharechain has solved a Bitcoin block and the reward (subsidy + fees, in sats) is `R`. Let's assume we _could_ randomly select and "burn" `R` sats worth of shares, in exchange for distributing the `R` bitcoin to those shareholders. This is nearly equivalent to what happens in a share buyback in the traditional equity markets. 

However, mathematically the buyback method is equivalent, in expectation, to the much more simple, and also conveniently tractable/implementable: _"select a single share at random and distribute the entire reward `R` to the owner of that share."_

A major advantage of the above simplification is that we need only select a single share at random. Whereas with what you want to do, we would need to select a set of shares and the set of shares selected is necessarily a function of `R` and some sort of price signal getting smuggled into the sharechain (otherwise we would not know how many shares to "burn"). This is an unnecessary complication, and may not even be possible, yet the simple model of selecting and distributing the reward to a single share _achieves the same result,_ and we can let the exogenous market just price things accordingly.

[quote="cmp_ancp, post:20, topic:2093"]
Inflation is also somewhat of a problem. If, in the long run, sharecoins are always devaluated,
[/quote]

Again, this depends on the chosen parameters around share issuance. In the linear model where each share issued is always tied to a constant amount of work, then even though there is an ever-growing supply of shares, the difficulty adjustment algorithm takes care of this problem for us. Say difficulty grows by 100x on the sharechain, then sure there will be 100x more shares issued. And yes, in such a scenario, the new "high difficulty" shareholders would have a much higher liklihood of being randomly selected for the bitcoin reward. However, they also paid (in work) for those shares. Similarly, the "low difficulty" (earlier) shareholders _still_ have a non-zero chance of being selected. This is why, in expectation, everything works out just fine. In the linear issuance model, even though supply of shares tends to infinity (note: it will, of course never actually get there), everyone is fairly accounted for. This is the beauty of a difficulty adjustment algorithm tied to thermodynamic work. 

[quote="cmp_ancp, post:20, topic:2093"]
Another cool feature that could be managed by sharecoin is coordination to onchain txs. Imagine some ark style payout tree
[/quote]

Sure, there are a lot of interesting things which might be tried by a sharechain, all without affecting mainchain bitcoin. I am less focused on the specifics here though and right now just want to explore the general p2share framework to ensure that it is sound and properly Bitcoin-aligned.

Markets, especially of the open, permission-less, and unhampered kind, are very good at solving these sorts of problems. Sharechains with features or issuance models which are reckless will not do well in the atomic swap market. Nobody will want to part ways with their precious sats for those garbage shares, and those sharechains will lose hashrate (or never even achieve it in the first place) because of it. Sharechains with solid, but differentiated, feature sets and fair issuance have a much better chance of success.

-------------------------

cmp_ancp | 2025-11-25 22:46:20 UTC | #22

Hi

I want to point out what I judge to be the key ideas I presented on my last post, because I think it deserves more debate over the final architecture.

[quote="VzxPLnHqr, post:21, topic:2093"]
You seem to be imagining a sharechain shareholder having an “account” more similar to how some networks use an account-based methodology for tracking amounts rather than a utxo methodology. While it might be possible or even preferable to use that sort of mechanism within the sharechain, that is a technical detail that, in my opinion, does not really matter right now in our discussion.

[/quote]

Yes, I think it doesn’t matter *now*, at this point of discussion (even though I think that an account model would be easier to save a checkpoint at mainchain). When I spoke about account, you could read “UTXO” as you like (I was indeed thinking on an UTXO model).

[quote="VzxPLnHqr, post:21, topic:2093"]
Sure, we could “burn” the entire utxo which contains that share, but that suffers from a number of problems, some of which you have pointed out. A miner can easily minimize her number of burned shares by pulverizing her shares across many outputs.

[/quote]

I really think we cannot runaway from burning the entire UTXO. I’ll explain:

I pointed out to inflation and small miner friendliness, but there is another aspect: how much time the miner has contributed to the pool. If we don’t burn, the old miners would have a gigantic amount of shares, at a point that new miners couldn’t even compete. The conclusion would be a pool for little, where new miners couldn’t enter, and even the old miners would eventually leave because of the total pool size plateau.

Also, that system wouldn’t be fair, because miners would be double paid. When a miner is selected to receive the revenue, he’s getting paid by its work, but if he doesn’t lose shares, the amount of work spent one time would be valued multiple times.

[quote="VzxPLnHqr, post:21, topic:2093"]
Whereas with what you want to do, we would need to select a set of shares and the set of shares selected is necessarily a function of `R` and some sort of price signal getting smuggled into the sharechain (otherwise we would not know how many shares to “burn”)

[/quote]

I think I didn’t express myself very well. In an UTXO model, I think the selected UTXO would be the one that is burnt entirely, in an account based, the account would be burnt. That way, we don’t need to insert shares-to-BTC exchange rates, it’s all or nothing.

BUT, the main point, and waht I think is the final solution to all incentives problems: the above linear probability function.

Imagine an exponential probability function based on amount of shares. In that way, doubling the amount of shares would more than double the probability of being selected. That is, \[p(s1+s2) > p(s1) + p(s2)\]

[quote="VzxPLnHqr, post:21, topic:2093"]
A miner can easily minimize her number of burned shares by pulverizing her shares across many outputs.

[/quote]

In an an above linear probability system, that would be severely penalized, that’s because it is far more desirable to have a single big UTXO instead of many small UTXOs. Big miners would severely undervaluate their energy spents by pulverizing shares.

[quote="VzxPLnHqr, post:21, topic:2093"]
Sharechains with features or issuance models which are reckless will not do well in the atomic swap market. Nobody will want to part ways with their precious sats for those garbage shares

[/quote]

That’s another incentive gain from the above linear solution: shares have different values in the hands of different actors. The shares of the small miners are much more valuable on the hands of bigger miners, because at their point on the probability curve, the marginal increment of shares have more impact. That’s the solution for small miners, big miners and entities are incentivized to buy shares.

We could think that this would lead to centralization, and we need for sure run some simulations on that. But, looking by another angle, knowing that the selected share (either in an UTXO or an account) would be burnt, even the big miners go down to position 0. In theory, some long running small miner could eventually have an enormous amount of shares and be selected. That’s because the concurrent miners would eventually go back to 0.

[quote="VzxPLnHqr, post:21, topic:2093"]
Sure, there are a lot of interesting things which might be tried by a sharechain, all without affecting mainchain bitcoin. I am less focused on the specifics here though and right now just want to explore the general p2share framework to ensure that it is sound and properly Bitcoin-aligned.

[/quote]

I just pointed out the cordination because that could be another mean of getting paid. Miners could cordinate inside the pool on order to “join their shares” and distribute it onchain by a cordinated payout tree. The sidechain would be a cordination point by itself, as a data availability medium, and dealing with miners, they would certanly be more frequently online.

-------------------------

VzxPLnHqr | 2025-11-25 23:45:57 UTC | #23

[quote="cmp_ancp, post:22, topic:2093"]
If we don’t burn, the old miners would have a gigantic amount of shares, at a point that new miners couldn’t even compete.
[/quote]

This is not correct, at least not in a linear "will work-for-shares" issuance model. It it only correct if you assume that shares are issued in a some other manner which gives early sharechain miners an "advantage" with regard to required work (expected # hashes) per share they receive. That is what Bitcoin itself did, and that is fine for Bitcoin (without it, we could never even contemplate the p2share model!). However, part of the confusion is that the original post _did_ present such an issuance model where shares were issued non-linearly in the shrinking-pie Bitcoin-like way. 

However, the more refined linear model I proposed, coupled with a suitable difficulty adjustment algorithm, explicitly fixes that objection. 

[quote="cmp_ancp, post:22, topic:2093"]
he’s getting paid by its work, but if he doesn’t lose shares, the amount of work spent one time would be valued multiple times.
[/quote]

The “paid multiple times for the same work” concern doesn’t apply in the linear issuance model.

Every new unit of work adds new shares and proportionally dilutes *all* existing shares. Because selection is random over all shares, every share, regardless of when it was created, has the same expected value. There is no persistent extra advantage from “early” work; it’s continuously diluted by later work.

To describe behavior cleanly, fold difficulty into expected value per hash:

- Let `EV_sc` be the expected sats per unit hash on the sharechain (this already accounts for sharechain difficulty and share price).
- Let `EV_mc` be the expected sats per unit hash on mainchain.
- Let `P*` be the share price at which `EV_sc = EV_mc`.

Then miners simply arbitrage:

**1. Mining decision (where to point hash):**

| Condition                    | Mining action      |
|-----------------------------|--------------------|
| `EV_sc > EV_mc` | Mine on sharechain |
| `EV_sc < EV_mc` | Mine on mainchain  |
| `EV_sc ≈ EV_mc` | Indifferent        |

**2. Trading decision (what to do with shares):**

| Market share price          | Trading action                          |
|----------------------------|------------------------------------------|
| `Price > P*` (“too high”)  | Sell shares (including newly mined)     |
| `Price < P*` (“too low”)   | Buy shares (if you want more exposure)  |
| `Price ≈ P*` (“just right”)| Indifferent; no strong trade implied    |

So all shares have the same expected value, and the “right” behavior is just: point hash where `EV` per hash is higher, and trade shares when their market price deviates from the fair `P*`.

-------------------------

coinjoinkillua | 2025-12-26 22:11:17 UTC | #24

Have you taken a look at this yet?

https://github.com/braidpool/braidpool/blob/8d3c5198593d691f50ae5238cdaee215f4213cda/docs/overview.md

-------------------------

VzxPLnHqr | 2025-12-28 05:09:37 UTC | #25

Thanks for your reply. 

[quote="coinjoinkillua, post:24, topic:2093"]
Have you taken a look at this [braidpool] yet?
[/quote]

It has been a while since I looked at Braidpool. What they mean by shares in their documentation is subtly different than what I am exploring in this thread (and in a more economic-focused fashion over in [this thread](https://delvingbitcoin.org/t/bmax-pricing-sats-now-vs-sats-later-via-a-mining-sharechain-no-l1-changes-no-custodians-no-oracles/2165).

From Braidpool's readme:
> Custody of accumulated coinbase rewards and fees is performed by a large multi-sig among miners who have recently mined blocks using the [FROST Schnorr signature algorithm](https://glossary.blockstream.com/frost/). Consensus rules on the network ensure that only a payout properly paying all miners can be signed and no individual miner or small group of colluding miners can steal the rewards.

I have not yet fully been able to grasp how they intend to have the above claim be true. So one of the focuses of my reserach is to not introduce such a signing aspect and instead use game theory and random share selection to keep it entirely non-custodial.

-------------------------

