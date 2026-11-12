# Decentralized Coinbase Payout Protocols (DCPP): A Framework for Discussion

7d5x9 | 2026-11-11 03:41:14 UTC | #1

The mining-focused FOSS ecosystem finds itself in the midst of a renaissance. Besides the efforts behind Sv2, DATUM, and “[the open mining stack](https://www.256foundation.org/),” there are significant resources being pointed at projects that seek to reignite the spark that was once lit by the original P2Pool project.^[This includes P2Poolv2, Braidpool, and others.]

Of the [many reasons](https://github.com/p2poolv2/p2poolv2/wiki/P2Pool-vs-P2Poolv2) P2Pool failed, its inability to scale payouts remains one of the [most notable](https://bitcointalk.org/index.php?topic=18313.msg13057899#msg13057899).

As the community continues developing the protocols that support novel decentralized pool structures, it is important that we think deeply about how to scale coinbase-native payout protocols. This write-up is intended to stimulate discussion and research around this topic. We hope the community is excited to work on this problem collectively to come up with compelling solutions.

---

At a high level, DCPPs have three primary design constraints:

* Paying participating miners with distinct coinbase outputs does not scale.[^1]
* DCPPs represent an attack surface. Miners have ongoing operational overhead, so making payouts unreliable or uncompetitive can make a decentralized pool economically unviable. Actors who seek to disrupt a decentralized pool will attempt to manipulate it via payout settlement griefing (PSG).
* Bitcoin script is limited, so the payout logic has to be assembled from a small set of primitives.

To understand the nature of these constraints, let’s look at the most potent form of PSG: the Rich List Attack (RLA).

Say a pool can fit $N$ payout outputs in the coinbase and more than $N$ miners want to be paid. The pool needs a rule for who gets a direct coinbase payout (a slot). The naive rule is to pay the top $N$ contributors by work (shares) over some window. This is the “rich list.”

Any rule that hands out a fixed number of slots based on relative rank can be gamed given the following constraints:

* A decentralized pool has no effective “identity laundering” detection mechanism; i.e., a miner can split their hashrate across many pool identities.
* Slots are finite. Every slot one identity takes is a slot another miner loses.

A large miner can thus split into several identities that each still outrank the smallest legitimate miners. Those identities take extra slots and push the smallest miners out of the coinbase. This is the RLA.

What the larger miners gain in such a scenario depends on what happens to the excluded miners’ share.

In the extreme case, those rewards are redistributed to the miners large enough to secure a slot, extinguishing the losing miners’ rewards. That’s obviously bad.

At the other extreme, failing to obtain an output merely defers settlement. Because a coinbase payout must always be paid in full, this means carrying a credit forward as a pool liability, not carrying the underlying bitcoin forward. This obligation must be accounted for across blocks, and the pool must have fair settlement rules to ensure all economically viable liabilities are eventually paid.^[With the caveat that there are no guarantees, as a pool's hashrate can drop off at any time. ] To do this fairly requires a number of mechanisms and gets at the very fundamental problem that this writeup intends to address.

To further paint the risks associated with PSG and the RLA, it’s helpful to review some historically described DCPPs:

In 2017, Chris Belcher [proposed](https://bitcointalk.org/index.php?topic=2135429.0) a scheme where the coinbase pays out a single (or federated) payment hub. The hub maintains a payment channel with each miner. As blocks are found, the hub sends payments in proportion to their work. The coinbase output can be spent in three ways:

* Normally by a $2$-of-$2$ multisig between the hub and the miner who found the block;
* By the hub alone, but only by revealing the hash preimage that lets every miner settle their channel on-chain; or
* If the hub stalls or tries to hold the reward for ransom, by the block finder alone after a six-month `OP_CSV` timelock.

In the happy path, the hub pays the miners via their payment channels and then cooperates with the winning hasher to release the payment via the $2$-of-$2$ pathway.

More recently, Braidpool described a scheme it calls [Rolling Coinbase Aggregation](https://github.com/braidpool/braidpool/blob/8d3c5198593d691f50ae5238cdaee215f4213cda/docs/braidpool_spec.md#payout-update) (RCA). Each Braidpool coinbase pays a single pool-controlled output instead of paying miners directly. Once a coinbase matures, a separate transaction combines it with the previous RCA output into a new, larger RCA output, so the pool’s accumulated rewards always sit in one UTXO. At each update, the pool also signs a payout transaction, the Unspent Hasher Payment Output (UHPO), which spends the RCA output and pays every miner according to their share of work. The UHPO is not broadcast. Each RCA update spends the previous RCA output, which makes the old UHPO invalid. The pool broadcasts the current UHPO at the end of each 2,016-block difficulty window, or earlier if the pool fails.

Both transactions are signed by a $66\% + 1$ threshold (FROST/ROAST) of the unique miners who found the pool’s most recent blocks. The signers then delete their key shares so the funds can’t be spent any other way. The result is a small coinbase and payouts batched over about two weeks.

P2Poolv2 supposes a [market-driven](https://github.com/p2poolv2/p2poolv2/wiki/Payout-mechanism---Trading-Shares-For-Bitcoin) DCPP. In this case, market makers, or hashrate aggregators, purchase “sharechain” UTXOs from P2Poolv2 miners who wish for a payout prior to the pool finding a block and/or whose hashrate is insufficient for a direct coinbase payout. They do this via an atomic swap of Lightning and a sharechain UTXO. These market makers must aggregate enough sharechain UTXOs to earn their own slot in the coinbase.

While useful schemes, each on their own is vulnerable to some form of PSG.

Besides some of the [self-described problems](https://bitcointalk.org/index.php?topic=2135429.0#:~:text=most%20serious%20problem%20with%20this%20proposal) with Belcher’s scheme, the “rich list” could launder their hashrate through hundreds or thousands of identities, forcing the hub to open, and fund, a channel to each identity before being paid. To prevent griefing, they might require some upfront payment or decide to implement some minimum payout threshold. Sadly, neither option is free: the fee might make the scheme uncompetitive with existing pools, and worse yet, a payout threshold brings back the RLA.

In Braidpool, the “rich list” concern shifts from payout eligibility to control of the RCA output. Malicious miners with enough signing power can redirect payouts arbitrarily.

In P2Poolv2’s market-structure design, while a market maker can aggregate sharechain value, unless the total aggregate value of that market maker’s sharechain UTXO exceeds the value of the lowest-ranked payout admitted to the coinbase even under an active RLA, the attacker can still crowd it out by splitting its holdings into enough higher-valued identities to occupy every available payout slot.

One conclusion we can see from the above is that the attack surface for PSG is quite large. Attempting to mitigate one PSG vector often shifts the risk factor from one area to another. We need to layer on more protocol rules to manage the risk.

As a helpful exercise, let’s imagine a hypothetically perfect coinpool-style DCPP:

When constructing a block template, participants non-interactively derive a single coinbase output that commits to a tree of proportional payout claims reflecting the sharechain UTXO set. Every eligible claim is represented without competition for ranked payout slots or reliance on a payout lottery.

Once the block is mined, each participant holds an independently enforceable claim to their leaf. They can non-interactively transfer that claim, atomically swap it for on-chain bitcoin or another off-chain claim, or exit unilaterally.

In this ideal construction, each participant’s exit cost and delay are bounded and economically reasonable, with the resources required to create and settle additional leaves charged to those who create them. Splitting an entitlement across identities therefore confers no allocation advantage and cannot impose unpriced settlement costs on others. Passive holders face no expiry that forfeits their funds, no intermediary can redirect the shared balance, and liquidity providers can recover acquired claims without buying out every participant or waiting for an unrelated sweep deadline.

For miners who seek advanced payout arrangements, a decentralized hashrate market exists which enables miners to swap their unrealized payout claims for an on- or off-chain payment.

While we don’t know how to build such a coinpool even with the most advanced covenants, we can use this as a benchmark and chip away at the unrealistic properties until we have something useful.

Stepping back, it seems then that we really have two tools to work with: Bitcoin script and hashrate/liquidity markets. For the sake of conversation, let us assume that we have a robust set of liquidity providers who are willing to make a hashrate market with competitive pricing.

From the above benchmark, our first goal is to build a coinpool funded directly in the coinbase. To guarantee unilateral exit from a coinpool, participants must acquire presigned exit transactions. This poses an issue.

Today, miners utilize the coinbase to expand their nonce space. This means that the coinbase’s txid is not stable. Because all known layer 2 systems that depend on unilateral exits require knowledge of the txid at signing time, the payout tree can only be signed after the block is mined.^[This is a problem solved by floating / (re)bindable signature schemes, enabled by various encumbrances, such as the set described in BIP 448. ]

I believe this limitation imposes a fundamental design constraint on DCPPs: all constructions collapse into a locking script controlled by a single or threshold of keys (with possible fallback paths guarded by timelocks and hashlocks).

Given this constraint, it would seem that allowing a single entity to control the output at maturation is naive; a threshold should be elected to take on this task. This raises the following questions:

* Who holds the threshold and how are they selected?
* What role does the threshold play in effectuating payouts?
* How long does exposure to the threshold last?
* What happens when the threshold becomes unavailable or refuses to cooperate?

I can offer some example answers to these questions via two DCPP constructions using existing coinpool structures, Spark and Ark (slightly modified for coinbase funding and MM efficiency), under the P2Poolv2 sharechain model. Notably, coinbase funding does not appear to introduce new attack primitives once a valid tree has been issued and miners have received their recovery paths. What you will see, however, is that miners can still identity-launder and make small claims more costly to aggregate or settle.

Let’s start with a statechain scheme that supports leaf splitting, like Spark. We can assume the SSP (Spark Service Provider) is a professional outfit that specializes in operating coinpools and is “enshrined” into the P2Poolv2 protocol.^[We will enshrine them for sake of simplicity.] It can be forked out by participants at any time.

Imagine that the signers associated with the funding output are a party made up of an SSP and a set of market makers (MMs), collectively the Coinbase Payout Operators (CPOs). Perhaps the MMs are elected based on some timelocked on-chain bond. The signing policy is

$$
\mathrm{SSP}\land(2\text{-of-}3\ \mathrm{MMs}),
$$

combined into a single operator key $O$ (for example, using weighted FROST). You might imagine fallback paths that involve the winning miner and the SSP **or** the MMs.

For the sake of simplicity, let’s assume the CPOs are honest.^[If they aren't there are many obvious attacks on the system at maturation.]

When building the template, the coinbase pays an output to $O$. The sharechain determines each small miner’s allocation and public key $m_i$. Once a block is found, the threshold builds a split tree spending that specific coinbase outpoint and signs every internal transaction with $O$. While the miners do not need to be online at issuance, they do need to contact the SSP for their complete root-to-leaf recovery paths (these could be gossiped via P2Poolv2 or served at a specific endpoint).

Each leaf is a Taproot output with two spending paths:

* **Key path:** the combined key $m_i+O$, used for Spark-style cooperative transfers.
* **Script path:** $\langle m_i\rangle\ \mathsf{CHECKSIG}$ after a relative timelock. This is the miner’s initial refund.

In Spark, the user co-signs their initial refund transaction before depositing. Putting the refund in the leaf’s script means miners don’t have to be online when payouts are issued. Each miner only needs to receive its signed path from the root.

Transfers and leaf splits then work as they do normally in Spark. Miners would swap their leaves to a MM, which would aggregate leaves and sweep them on-chain.

Without going into every protocol detail, let’s review one PSG vector that surfaces from this design: the “poison pill.” It is a combination of “tree inflation” and “aggregation holdout.”

In the context of a Spark-like construction, it’s important to note the role of the MM: to buy and consolidate the leaves until they can sweep the output on-chain to recycle their liquidity.

A malicious miner can easily attack this:

**Shard:** The attacker launders their hashrate into $N$ identities, which get spread across various Spark subtrees.

The attacker sells $N-1$ of its identities in every subtree in which it has positions.

Because the MM now owns all or most of the other positions in the subtree, they want to sweep the bitcoin on-chain so they can recycle their liquidity. Because the miner is griefing the subtrees, the MM has to unroll the tree on-chain to get their bitcoin out.

Because a rich-list miner can poison every subtree off-chain for zero fee cost, an MM can never price small miners’ leaves based on cheap cooperative aggregation ($\sim 1\ \mathrm{vB/leaf}$). The MM must price every single leaf as a full unilateral branch unroll plus leaf sweep. While in the covenant-less model the CPOs can restructure the tree, let’s assume they would not take such an action as it would destroy any unilateral exits.

This price, whatever it ends up being, just gets passed on to the miners. The CPOs could treat this as a bond and refund the user when they claim their leaf (as long as it’s done in a timely fashion), or just take it as a fee. There are many other options as well, left to the reader’s imagination.

Let’s now assume an Ark-based alternative, using [Second’s](https://second.tech/docs/learn/vtxo#transaction-trees) transaction trees as a reference. A vector for identity laundering is still present, but Second’s ordinary [expiry-based liquidity-recovery model mitigates it](https://second.tech/docs/learn/rounds).

As before, the coinbase funds the initial payout tree. In this case, the ASP and some MMs control the funding output and cooperate to construct the initial tree. Again, the miners’ exit packages can be retrieved at an ASP endpoint or gossiped across the P2Poolv2 network.

Let’s also assume the ASP and MMs controlling the funding output are honest, miners can hold their VTXOs passively until expiry, and MMs are willing to provide swap liquidity.

Second’s self-signed rounds require every user to cosign their branch, which offline miners cannot do. Instead, we adopt hArk’s delegated model: $O$ pre-signs every branch transaction in the miners’ place and then deletes its keys, so miners trust that the tree was built honestly. Each leaf is a VTXO with key path $m_i+S$ and script path $\langle m_i\rangle\ \mathsf{CHECKSIG}$ after $\Delta$, and the ASP sweeps any remaining tree after $T_{\mathrm{exp}}$.

When a miner offboards, the ASP supplies new on-chain bitcoin and receives a forfeit over the old VTXO. The forfeit prevents the miner from later broadcasting its emergency path and successfully claiming the VTXO a second time.

Second’s normal liquidity model is therefore straightforward: the ASP deploys BTC immediately and waits until expiry to recover the backing of the old VTXOs. Second prices this according to the amount deployed, its remaining lifetime, and the ASP’s bitcoin-denominated cost of capital.

What this does is effectively price in the poison-pill attack as the ASP assumes it will not be able to recover the VTXO until expiry. This means that the DCPP has effectively two liquidity providers: the ASPs and the MMs. The MMs in some cases prepay via FPPS for sharechain UTXOs and price in their risk, and ASPs must charge interest to VTXO holders to cover their own liquidity costs for the expiry window.

One could try to improve on this model with early aggregate settlement. When the ASP is offboarding a VTXO, it can request a signature from the participant so it can sweep a branch once it has collected all signatures.^[his assumes some atomic swap scheme that produces a signature in exchange for an off-board.] This has the capacity to reduce the ASP’s working-capital requirements. Unfortunately, this is where the poison pill reappears.

The attacker again launders its hashrate into $N$ identities and spreads its claims throughout the tree. In every subtree it wishes to affect, it sells $N-1$ positions and retains one small, unspent VTXO. The other miners can still offboard normally. Unlike the Spark construction, there is no MM that must immediately unroll its acquired claims: the ASP can simply pay each exiting miner from its own treasury and collect the corresponding forfeits.

The ASP has nevertheless paid against a branch whose old value remains locked. The residual VTXO owner refuses to authorize the early $n$-of-$n$ branch spend. The ASP must therefore choose between carrying the payment until the ordinary expiry sweep or paying for an earlier on-chain recovery.

The attack is consequently less dramatic, but it is still a PSG vector as the ASP must price the worst-case duration of its advance and maintain enough capital to survive that duration. At the user level, Ark requires miners to be online to refresh their VTXOs in the event they do not cash out. Whether or not this is acceptable is left to the reader’s imagination.

---

So, let’s assume we can leverage an Ark-like construction if we have a sufficiently capable and liquid ASP. The in-coinpool RLA appears managed by charging fees on a per-leaf basis. Unfortunately, this does not resolve the underlying RLA at the output level. While Ark provides a way to spend and settle an already-existing coinpool output, it does not determine how that output enters the coinbase in the first place.

And even if we decided to bypass the coinpool construction altogether and just rely on MMs to aggregate sharechain UTXOs, we still have the output-level RLA to deal with.

I see two viable options at present:

* Probabilistic guarantees of an output position via a value-weighted random draw.^[Idea suggested by @MattCorallo. The design for this needs to be considered carefully, but I can't imagine any specific reason why it shouldn't work.]
* Enshrinement of a given (or set of) CPOs/MMs.

In the first scenario, we’d likely see a set or single MM become the principal aggregator and settlor of coinbase-backed claims. This is a natural consequence of there being non-determinism in the payouts. Larger aggregators can settle more reliably and potentially offer prices that smaller competitors cannot sustain.

Additionally, FPPS-style liquidity provision now has a second layer of settlement variance: the underwriter must finance not only intervals without blocks, but also intervals in which its claims are not selected despite blocks being found. This further entrenches large aggregators who are sufficiently capitalized.

Enshrinement takes the opposite approach. Rather than requiring the CPO to accumulate enough sharechain value to make inclusion likely, the pool’s template-construction rules reserve an output (or set of outputs) for the recognized CPO (or set of CPOs). Any sharechain value below a specific threshold must be tagged with a specific CPO.

On its face, enshrinement feels antithetical to a decentralized pool’s goals. The balanced perspective is that a centralized pool has many individual points of centralization. A decentralized pool construction that uses an enshrined CPO adds back a narrowly defined coordination role, rather than undermining the entire premise of the scheme. A misbehaving or defaulted CPO would be routed around in the same way one would be in the market-driven context. The list of enshrined CPOs would have to be hard-forked to be changed, but a set could be introduced so that there is some market choice.

I would also provide the observation that in both situations the network would likely terminate at a similar outcome: a single/small set of MMs that control the market.

Which of these is the right answer, I don’t know. But, if we are to step back one final time, I would like to once more look at the P2Poolv2 market-based structure layered onto a value-weighted draw. In this case, there is no CPO; there is no coinpool. Miners atomically swap their sharechain UTXOs for Lightning, and the MMs take on whatever risk is associated with non-deterministic settlement. The reverse product, where miners give up their shares now and are paid a proportional cut when blocks are found, is also possible, but cannot be made atomic and is thus custodial.

Perhaps this is the simplest structure that we can come up with, and we decide the value of a covenant-less coinbase-funded coinpool protocol is not worth the engineering effort at this time. Perhaps when rebindable signatures become available, the conversation will change. I think the most compelling advantage of a well-designed DCPP (not necessarily one described here) is that it can dramatically lower the fees that small miners pay, making P2Poolv2 more competitive even if it adds complexity. The fewer intermediaries introduced in the system (e.g., no MMs), the fewer mouths to feed.

I also think, as a final note to the reader, it's always important to remind ourselves of the threat model: who is the attacker we are most worried about? A resourced attacker could join the pool and execute a block-withholding attack. They don’t need to attack the coinbase. What kind of attacker would see value in participating honestly in the mining game but griefing the payouts? If we don’t have a good answer to this, perhaps we can just breathe a sigh of relief and lean into the simplest solution.

[^1]: * Firmware limits: Mining firmwares have known (and in some cases undocumented) limits on coinbase size.
    * Block space / opportunity cost: Each output consumes scarce block space. That space competes with fee-paying transactions.

-------------------------

LearnBitcoin.com | 2026-11-11 19:50:47 UTC | #2

Thanks for writing this up. Having names for PSG and the RLA helps by itself, since until now everyone has described these problems in their own words.

Some disclosure first. I'm windpath on BitcoinTalk, so the "most notable" link in your second paragraph is my post from November 2015. I ran a public 0% fee P2Pool node at p2pool.org in those years, and still hold the domain (more on that at the end).

Since your argument leans on what happened to P2Pool, I went back to the chain rather than trust my memory of those years. I pulled the coinbase transaction of every block that pays P2Pool's donation script. There are 2,545 of them, 1,024 from 2012 alone, and the pool still turns one up about once a year. Some of what's in there bears on your framework.

The N in your example was a few hundred. A typical P2Pool block paid about 200 addresses in 2012 and about 390 in 2014. The most it ever paid was 553, in block 236,962 on 19 May 2013, and that coinbase transaction was 19 kB. Only 42 blocks ever paid 500 or more.

The original code already had a rich list rule in it. It kept only the 4,000 largest payouts by amount, with a comment calling the limit "unlikely to ever be hit". It never was, so that rule was never tested by anyone, friendly or hostile.

What squeezed small miners was share difficulty, with the dust rule on top of it. If a miner's expected payout per block fell under 0.001 BTC, the software raised that miner's share difficulty until a share would clear the line. You can see it on the chain as a cliff. In July 2014, 1,749 of the month's 15,865 payouts were between 0.001 and 0.002 BTC, and not one was under 0.001. The miners below the line don't appear as small outputs. They don't appear at all, and that's the limit of this data: it shows who got paid, not who gave up.

Nobody was attacking us when that happened. I wrote in that 2015 post that "you can only split the 25BTC reward so many ways in a decentralized trust-less environment, and smaller miners will always get squeezed out", and all it took was growth. So I'd treat the RLA as where any ranked-slot rule ends up by default. A motivated attacker gets you there sooner than growth would.

There's one row per block here if anyone wants to check the numbers or reuse them: 
https://www.learnbitcoin.com/downloads/p2pool-coinbases.csv

The same 2015 post guessed at a way out: let miners accrue shares somewhere trustless and pay out in BTC once a threshold is reached, so the dust never touches the generation transaction. Eleven years later that's still roughly the shape of what you're describing, which says something about how hard the middle part is.

On your closing question, who would mine honestly and grief the payouts instead of withholding blocks. I think there's a plain answer: anyone whose business depends on hash power staying where it is today. An existing pool, or a big miner friendly to one, already has the hash power, so pointing some of it at a decentralized pool costs them close to nothing beyond slightly higher latency. Withholding blocks costs them a little of their own income and drags down the pool's luck in a way people eventually notice and talk about. Splitting into a few hundred accounts costs nothing and looks exactly like the pool gaining small miners. If the goal is for the new pool to stay slightly worse than the alternative, quietly and deniably, payout griefing is the better tool. So I don't think we get the sigh of relief.

I do think that points toward your simplest structure anyway, with two things I'd want pinned down about the value-weighted draw.

First, whether splitting really becomes pointless depends on how the draw treats a big claim. If one large claim can only ever occupy one slot, a large miner still gains slots by splitting into many claims, and the RLA comes back as a probability game: the small miners aren't locked out, they just get drawn less often. For splitting to buy nothing, a claim's chance at each slot has to track its value no matter how it's packaged.

Second, the seed. The coinbase outputs are fixed before the block is mined, so the randomness has to come from something earlier, either the previous Bitcoin block or the sharechain. Shares are cheap to grind compared with blocks, so a seed taken from sharechain state looks a lot easier to bias than one taken from the last block hash. And whichever it is, claims shouldn't be able to change shape after the seed is known.

If both of those hold, the worst a motivated incumbent can do is make small claims more expensive for market makers to settle. That gets priced into a spread. It's a much better failure than payouts being deferred forever or redirected.

On the terms: I've written plain-language glossary pages for DCPP, PSG and the RLA at learnbitcoin.com, credited to this post, so there's somewhere to point newcomers. They start here: https://www.learnbitcoin.com/glossary/decentralized-coinbase-payout-protocol . If I've got anything wrong in them, please let me know and I'll fix it.

On p2pool.org: I've kept it since the original project. If one of the successor projects gets far enough along that it has a real chance, I'll lend the domain to it. Until then it stays neutral. I'm happy to dig up what I remember from running a node if it's useful to anyone here.

-------------------------

7d5x9 | 2026-11-12 05:48:18 UTC | #3

> I’m windpath on BitcoinTalk, so the “most notable” link in your second paragraph is my post from November 2015.

Whoa, that is so cool this post found you (and I found your post)! Thanks for taking the time to reply.

> I pulled the coinbase transaction of every block that pays P2Pool’s donation script. There are 2,545 of them, 1,024 from 2012 alone, and the pool still turns one up about once a year.

It's really impressive how many payouts these blocks carried in the coinbase. Thank you for sharing the data, and very cool (and unexpected) that there are still blocks being found!!

> The N in your example was a few hundred. A typical P2Pool block paid about 200 addresses in 2012 and about 390 in 2014. The most it ever paid was 553, in block 236,962 on 19 May 2013, and that coinbase transaction was 19 kB. Only 42 blocks ever paid 500 or more.

I did not mean to imply the N was in the hundreds. The [footnote here](https://delvingbitcoin.org/t/decentralized-coinbase-payout-protocols-dcpp-a-framework-for-discussion/3060#footnote-9019-2) should have been more explicit, but I don't think there is an official source to link to here, other than third party implementation details (e.g. [DATUM](https://github.com/OCEAN-xyz/datum_gateway/blob/85a19fd48caed0d00e2d26045916d30f84b0668a/src/datum_stratum.c#L1675\(\))). We are probably looking at <=25 outputs total on Bitmain hardware.

> So I’d treat the RLA as where any ranked-slot rule ends up by default. A motivated attacker gets you there sooner than growth would.

Right, the symptom of an RLA and the pool hitting its growth ceiling produces the same outcome. To be fair, I wouldn't call honest growth an attack though, that's just a protocol scaling limit.

> An existing pool, or a big miner friendly to one, already has the hash power, so pointing some of it at a decentralized pool costs them close to nothing beyond slightly higher latency.

Agreed, this is definitely a decentralized pools most likely adversary.

> Withholding blocks costs them a little of their own income and drags down the pool’s luck in a way people eventually notice and talk about.

A properly executed block witholding attack can increase the revenue of the attacking pool. I think the question is: can the attacking miner earn more revenue (while reducing the competitiveness of the decentralized pool) via a block witholding strategy or a PSG attack? If there is no DCPP at all, I agree with you, PSG all the way-–they earn upfront and undermine the pool.

However, even a modestly architected DCPP will mitigate the PSG enough as to make it not an attractive attack vector. I may have been a bit too strong in how I worded that paragraph. I do think we need to resolve this, but I also think it's possible it doesn't need to raise the bar much to mostly mitigate the attack vector. I think with the right protocol design (and probably covenants), what is most attractive about advanced DCPPs is that they lower participation fees (thus making a decentralized pool more competitive).

> two things I’d want pinned down about the value-weighted draw.

Yes, I think if we find enshrinement to be unpalatable, we really want to start focusing on the weighting and selection process. There will be many footguns and we don't want to introduce an incentive to do the thing we are trying to mitigate.

> On the terms: I’ve written plain-language glossary pages for DCPP, PSG and the RLA at learnbitcoin.com, credited to this post, so there’s somewhere to point newcomers. They start here: Decentralized Coinbase Payout Protocol (DCPP) | LearnBitcoin . If I’ve got anything wrong in them, please let me know and I’ll fix it.

Thank you, this reads quite well!

> I’m happy to dig up what I remember from running a node if it’s useful to anyone here.

So great to have you around for these discussions, and thank you for stewarding the domain all these years. It's so easy for these things to fall into the wrong hands.

-------------------------

