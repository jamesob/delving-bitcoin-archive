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

