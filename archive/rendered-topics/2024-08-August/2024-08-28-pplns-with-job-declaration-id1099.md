# PPLNS with job declaration

Fi3 | 2024-08-28 14:21:15 UTC | #1

I'm sketching up an sv2 extension, that allow miners to verify pool's payouts when miners are allowed to select transaction hence mining jobs with different fees.

The extension is based on a system proposed [here](https://github.com/demand-open-source/pplns-with-job-declaration/blob/bd7258db08e843a5d3732bec225644eda6923e48/pplns-with-job-declaration.pdf) this system is build on top of PPLNS.

You can find the extension spec [here](https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/extension.md)

[Here](https://github.com/demand-open-source/share-accounting-ext/tree/master/src) instead I implement the extension messages using SRI libraries.

[Here](https://github.com/demand-open-source/demand-cli) I started implementing a miner translator proxy that use the extension.

Everything above is still work in progress and need reviews.

-------------------------

plebhash | 2024-09-06 12:33:28 UTC | #2

I'm a Spiral grantee working as a contributor to [StratumV2 Reference Implementation (SRI)](https://github.com/stratum-mining/stratum).

In a time where the Bitmain FPPS debt-slavery cartel is pushing Bitcoin towards dangerous levels of mining centralization, this post is extremely relevant and I would love to see more engagement from mining players here.

The ideas presented on the paper titled [PPLNS with Job Declaration](https://github.com/demand-open-source/pplns-with-job-declaration/blob/5e3e7666f177e9a1e217e72da65a35b612505613/pplns-with-job-declaration.pdf) are academically sound and it's refreshing to see that [Demand Pool](https://www.dmnd.work/) has some talented minds proposing a tangible path out of the dark situation Bitcoin is currently under.

Remember that while SV2 allows for hashers to choose their own templates via Job Declaration (JD), the protocol itself is inherently agnostic to:
- share accounting
- reward distribution

So when a pool decides what kind of algorithms to employ to solve those two specific problems, their design space is limited to:
- hashers blindly trusting the pool is fair
- feeding back information to hashers to minimize trust

And if they choose the second path, they will inevitably need protocol extensions.

Now, SV2 decentralizes via JD! 

Which means hashers have the right to be paid for hashing on templates that could be economically suboptimal with regards to fee revenue... that could happen for different reasons:
- maybe the hasher's Template Providing (TP) node is suboptimally connected (remote location, poor internet, plebhashers) and the "mempool" they see is suboptimal.
- maybe they are ideologically driven and see some categories of consensus-valid transactions as spam (which is a subjective term, albeit introduced by Satoshi and ingrained into protocol primitives).
- maybe they want to do [MEVil](https://bluematt.bitcoin.ninja/2024/04/16/stop-calling-it-mev/) and prioritize transactions under some specific meta-protocol.
- maybe they want to filter out consensus-valid transactions that would hurt low-end nodes (see [GCC](https://github.com/TheBlueMatt/bips/blob/7f9670b643b7c943a0cc6d2197d3eabe661050c2/bip-XXXX.mediawiki) for more).

So an economically rational pool needs a mechanism to still allow for jobs with low-revenue templates, while rewarding them fairly with regards to jobs with more economically optimal templates, which is work that deserves to be rewarded more.

And in order for hashers to be willing to put some level of trust on the pool, they need to be fed back some info to give them reassurance about how their templates are being rewarded fairly in proportion to other hashers. That is exactly what this new SV2 protocol extension proposes, and I'm happy it's built on top of PPLNS, rather than FPPS.

I'm looking forward to following [this Discussion on SRI repo](https://github.com/stratum-mining/sv2-spec/issues/95), where Braiins, Demand, and SRI engineers are shaping up the implementation details for the first ever SV2 protocol extension.

-------------------------

plebhash | 2024-09-06 11:24:19 UTC | #3

As a pleb advocate, I'm particularly curious how the proposed SV2 extension would affect transactions described under [GCC](https://github.com/TheBlueMatt/bips/blob/7f9670b643b7c943a0cc6d2197d3eabe661050c2/bip-XXXX.mediawiki), which would essentially penalize low-end nodes, even if they:
- pay high fees
- are consensus-valid
- are available in the so called "Standard/Canonical/Platonic" mempool

Let's call these transactions as **GCC vectors**.

How should a SV2-JD-enabled Pool take that into account? I see three options:
- A. reject all jobs that include GCC vectors in the proposed templates (as a JD policy)
- B. impose economical penalties to jobs that include GCC vectors in their template (as a reward policy)
- C. ignore GCC vectors

The proposed extension is not really relevant for option A, since the basic SV2 primitives already allow for that.

Option B does have some relevance here, and I'm curious as to whether this is being taken into account in the design of the proposed extension.

-------------------------

Fi3 | 2024-09-06 07:22:02 UTC | #4

We only look at total fee payed by the mined[^1] job, respect to the jobs in the same slice. So we will not penalize anything we can say that pool take an agnostic approach.

[^1]: job for which we received the share.

-------------------------

plebhash | 2024-09-06 12:30:42 UTC | #5

💔 from pleb noderunners.

But that is fully understandable, since it shouldn't be up to pools nor hashers to act as "benevolent guardians" of low-end nodes. Ideally, this is something to be addressed at the consensus level, not mining.

Hopefully @AntoineP will save the day for us with [GCC Revival](https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710)! 🙏

-------------------------

marathon-gary | 2024-09-06 14:28:25 UTC | #6

What prevents a pool from diluting shares within a slice/window?

I think this protocol makes sense but still suffers from the same issue that pools suffer from now, all of the validation data being used by miners is given out by the pool. 

I think there's a way forward with blinded signatures here that makes each proposal more robust. I'm not sure what that looks like yet but will post when I have someothing on the topic.

-------------------------

Fi3 | 2024-09-06 14:33:46 UTC | #7

the fact that every miner verify a random sample of shares and that shares can not be faked.

-------------------------

marathon-gary | 2024-09-06 14:38:21 UTC | #8

But miners can only validate using shares they know of? Where does this source of shares come from that miners are using as a random sampling?

-------------------------

Fi3 | 2024-09-06 14:40:53 UTC | #9

No miner can validate every share that the pool receive.

-------------------------

marathon-gary | 2024-09-06 14:51:39 UTC | #10

How do they get shares they didn't create?

-------------------------

Fi3 | 2024-09-06 14:58:25 UTC | #11

they can use this message https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/extension.md#getshares-client---server

This is not an sv2 message and it can be used only with pool that support the ext.

-------------------------

plebhash | 2024-09-06 15:03:21 UTC | #12

> How do they get shares they didn’t create?

See:
- [`Share` Data Type](https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/extension.md#share)
- [`GetShares` message](https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/extension.md#getshares-client---server) (Client -> Server)
- [`GetSharesSuccess` message](https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/extension.md#getsharessuccess-server---client) (Server -> Client)

-------------------------

Fi3 | 2024-09-06 15:07:52 UTC | #13

No Share is a custom datatype and is encoded like described in the extension spec.

Is implemented here: https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/src/data_types.rs#L174

-------------------------

marathon-gary | 2024-09-06 15:08:22 UTC | #14

The client gives the server a list of id's (how is this list of IDs determined?) of shares they want to validate, I understand this piece. There is just no guarantee that the pool is providing accurate information. This is the same as using an API to query for shares.

What stops a pool from providing inaccurate or misleading share data? or omitting shares when requested?

Edit: Inaccurate or misleading shares would fail the Merkle inclusion validation piece. but omission remains un answered.

-------------------------

plebhash | 2024-09-06 15:11:24 UTC | #15

> No Share is a custom datatype and is encoded like described in the extension spec.
>
> Is implemented here: https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/src/data_types.rs#L174

yeah I was making some confusion there, thanks for the clarification!

edited those questions out of my previous message to avoid further propagating my misunderstanding

-------------------------

Fi3 | 2024-09-06 15:09:43 UTC | #16

yep for inaccurate an misleading shares you have the merkle tree. Not clear what you mean by omission.

-------------------------

marathon-gary | 2024-09-06 15:16:25 UTC | #17

Nvm on the omission part, I misunderstood. 

Does the id of a share factor into the Merkle path for that share?

-------------------------

Fi3 | 2024-09-06 15:17:25 UTC | #18

this is not needed cause when you get the actual share you can verify that the is the share at position x in the slice, so is the one that you required

-------------------------

marathon-gary | 2024-09-06 15:22:17 UTC | #19

Given a window with one slice. Given that slice contains 10 shares, 0...10.

Client A requests shares 1,3,5,7. Pool provides 1,3,5,7

Client B requests shares 2,4,6,8. Pool provides 2,4,5,8, mislabeling share 5 as 6

How does Client B know they were deceived? Assuming share 6 and share 5 are not shares they themselves submitted.

-------------------------

Fi3 | 2024-09-06 15:27:29 UTC | #20

they can spot it cause each share come along the share's merkle path in the slice, hence the index.

-------------------------

Fi3 | 2024-09-06 15:34:51 UTC | #21

keep in mind that miners have shares mined by themself in the slice and they know the index of them: https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/extension.md#shareok-server---client

-------------------------

Fi3 | 2024-09-06 15:43:39 UTC | #22

we can add the index in the merkle root I think that would be better

-------------------------

lorbax | 2024-09-06 16:12:36 UTC | #23

A miner can solve this issue by asking the pool a bunch of consecutive shares, containing some produced by the miner itself. Since a miner knows the position in the slice of the shares he produced, becomes very difficult to perform the trick you said.
BTW I cannot see the incentive for doing so. I think the most likely scenario is when the pool is a miner and produces fake shares. Recall also that this issue affects also standard PPLNS and has nothing to do with job declaration.

-------------------------

marathon-gary | 2024-09-06 16:18:15 UTC | #24

>  think the most likely scenario is when the pool is a miner and produces fake shares. 

Exactly this. 

> Recall also that this issue affects also standard PPLNS and has nothing to do with job declaration.

You're right, the issue I describe is orthogonal to this proposal (which I like a lot!).

-------------------------

marathon-gary | 2024-09-06 16:21:39 UTC | #25

> we can add the index in the merkle root I think that would be better

You could use the share index to determine Merkle tree ordering for the slice, this would prevent the scenario I outlined above. IIUC, this would also give the client some idea as the what the Merkle path should look like for a given share.

-------------------------

marathon-gary | 2024-09-06 16:24:49 UTC | #26

> this would also give the client some idea as the what the Merkle path should look like for a given share.

If the Merkle tree ordering is pre-determined by what indeed it is in a slice, then you can omit that from the Share data struct.

-------------------------

Fi3 | 2024-09-06 16:29:52 UTC | #27

main issue is that you can not cache the hashes and for each window you need to recalculate everything.

-------------------------

marathon-gary | 2024-09-06 16:31:17 UTC | #28

[quote="Fi3, post:27, topic:1099"]
can not cache the hashes
[/quote]

for the Merkle tree?

-------------------------

Fi3 | 2024-09-06 16:38:03 UTC | #29

yep cause index depend on window, so slice root would be different based on the window on which belong. I have to look better into it, but it seems feasible.

-------------------------

plebhash | 2024-09-06 19:54:14 UTC | #30

@lorbax I have a question about the last formula of Section 3.3.

Can you demonstrate this? 

$\sum_{j=k_1}^{k_2} score_f(s_j) + ... + \sum_{j=k_{t-1}}^{k_t} score_f(s_j) = t$

From an intuitive perspective it makes sense, but I'm having trouble to visualize this clearly in my mind.

-------------------------

plebhash | 2024-09-06 22:06:12 UTC | #31

While I read the paper, I created some visuals to organize my thoughts.

If the information is accurate and the authors feel they are useful, they are free to use and modify them without attribution.

![PPLNS with Job Declaration-coinbase reward drawio (1)](upload://xbk5dybkdwKrDjBA2Y7QRMx7EgU.png)

![PPLNS with Job Declaration-slices drawio (1)](upload://rkUojT9Or7xsjJta4Vxj4Hfa3wI.png)

![PPLNS with Job Declaration-MMEF drawio (2)](upload://76zzXp5DJNvViw5xnBlKsxWteXz.png)

-------------------------

lorbax | 2024-09-09 11:29:59 UTC | #32

There is a typo. There should be $0=k_0<k_1<\dots < k_t = N$, where $N$ is the number of shares in the PPLNS window such that the $m$-slice contains the shares $s_{k_{m-1}+1}, \dots , s_{k_{m}}$. Recall that, by Remark 5 in Section 3.2, if $S$ is a slice,  we have that 
$ \sum_{s \in S} score_f(s) =1.$
Hence
$$
\sum_{j=k_0+1}^{k_1} score_f(s_j) + ... + \sum_{j=k_{t-1}+1}^{k_t} score_f(s_j) = 1+\dots +1  =t
$$

-------------------------

Fi3 | 2024-09-10 10:34:45 UTC | #33

I slightly changed the share definition https://github.com/demand-open-source/share-accounting-ext/blob/8cf48767de3a477ab2b11ada8fb3f05c19cce758/extension.md#share

1. Remove merkle_path from the share, the field is redundant cause we can derive it from the transactions in the job, later in the verification procedure.
2. Add share_index: the index of the share in the slice. This solve the issue raised above by @marathon-gary (there is no way for a miner the check if the share sent by the pool are the ones required)

-------------------------

marathon-gary | 2024-09-13 14:04:28 UTC | #34

Is there a minimum threshold for Merkle inclusion that a miner would want to challenge the pool for? mathematically or statistically speaking?

-------------------------

marathon-gary | 2024-09-13 14:23:55 UTC | #35

Is the delta for determining a new slice dynamic or static? I suppose that's left up to the pool or implementer of the payout schema, but am curious about that.

@plebhash I think I was wrong in our discussion about this.

-------------------------

Fi3 | 2024-09-13 14:37:49 UTC | #36

The pool can decide it, there no way to communicate it in the protocol, so is something static that pool and miners have to agree before.

-------------------------

Fi3 | 2024-09-13 14:38:31 UTC | #37

not sure, I guess that depends on the number of miners that use the pool

-------------------------

lorbax | 2024-09-19 09:07:34 UTC | #38

Hi!
Just to mention that a new version of the article is available. I would like to thank @plebhash and @marathon-gary, that reviewed the article and made a lot of useful suggestions/corrections
https://www.dmnd.work/pplns-with-job-declaration/pplns-with-job-declaration.pdf

-------------------------

Fi3 | 2024-09-19 09:46:43 UTC | #39

Update ext spec, add [PHash](https://github.com/demand-open-source/share-accounting-ext/blob/e25659ff92505913e1b8c87354751b46e198726d/extension.md#phash) data type, needed by the miner to verify the share's work


The [PHash](https://github.com/demand-open-source/share-accounting-ext/blob/e25659ff92505913e1b8c87354751b46e198726d/extension.md#phash) message includes the previous hash and the starting index of the slices that use it. This message is sent within the [GetWindowSuccess](https://github.com/demand-open-source/share-accounting-ext/blob/e25659ff92505913e1b8c87354751b46e198726d/extension.md#getwindowsuccess-server---client) response, enabling the miner to identify which slices correspond to which previous hash, and thereby determine the previous hash for each share. This information is essential for verifying the work of each share.

-------------------------

plebhash | 2024-09-19 14:55:27 UTC | #40

I made some adjustments to this graph.

Now the $\delta$ steps are explicitly clear on the MMEF axis.

![PPLNS with Job Declaration-MMEF.drawio|348x460](upload://dja6x7ARhpJTzSQm5y5NLIyiyjB.png)

-------------------------

sjors | 2024-12-09 05:25:00 UTC | #41

I'm a bit worried about miners proposing fake block templates with absurdly high fees, thereby enjoying a relatively large payout.

This seems worse than block withholding, because such a miner could run a way with ~100% of the block reward with ~0% of the PoW.

The obvious counter measure is for the pool (or separate Job Declarator server entity, JDS) to verify every template. But this a non-trivial task, since the JDS node mempool could be very different. It may need to replace transactions its mempool with that of the template to check that it doesn't contain a big fee transaction that's actually unspendable.

Absurdly high fees would always trigger a new slice, so you can at least prioritize its verification.

For coinbase-only templates the JDS doesn't know the transactions and can't verify anything.

It's also unclear how, in the random sample verification protocol, you would distinguish a malicious miner from a malicious pool making such templates for themselves (and 'accidentally' approving them).

*Perhaps a simpler solution* is to cap the fees for all slices to whatever fees were in the found block. The fake templates, if not caught, would then have their fees reduced to a level that at least in principle they could have found. Then it's not much worse than regular block witholding.

This also creates a disincentive to produce high value templates with 'secret' transactions. Template providers will want everyone else to have the good stuff in their mempool too.

-------------------------

lorbax | 2024-12-10 09:32:38 UTC | #42

> This seems worse than block withholding, because such a miner could run a way with ~100% of the block reward with ~0% of the PoW.

If a miner is providing ~0% of the PoW, then he would get ~0% of blocks subsidy and ~0% of the block fees, because the fee-based score is calculated on top of the difficulty based score. Suppose that a slice contains the shares from ther $k_1$-th to the $k_2$-th. Suppose also that the fee and the difficulty score of the $i$-th share are $f_i$ and $\bar d_i$. Then
$ score_f(i) = \frac{\bar d_i f_i}{\sum_{j=k_1}^{k_2}\bar d_j f_j}$. If $\bar d_i \approx 0$ then this $score_f(i) \approx 0$.

The easiest architecture is that each pool run its JDS. In this case, the JDS validates each share. In the SRI, the JDS runs a light mempool just for faster recognition of the transactions. If there is some unknown transaction during validating the shares, then it will be asked to the miner through "PovideMissingTransactions" message, as for Sv2 protocol.

> For coinbase-only templates the JDS doesn’t know the transactions and can’t verify anything.

I understand that "coinbase-only templates" is a share with just the coinbase transaction, so it is an "empty weak block". But then you refer at the transactions, so there is more than one. I do not understand, can you explain?

> *Perhaps a simpler solution* is to cap the fees for all slices to whatever fees were in the found block.

This reflects the state of the mempool when the blocks is found. If a share is produced in a moment in which the MMEF (mempool maximum extractible fees) is much less then the fee found in the block, then the share is devalued, even though the miner is very good at maximizing fees. This is not fair IMO.

> It’s also unclear how, in the random sample verification protocol, you would distinguish a malicious miner from a malicious pool making such templates for themselves (and ‘accidentally’ approving them).

I don't quite understand what you mean. Perhaps is worth pointing out that one of the main concerns is when the pool is also a miner, so there are some incentives for the pool to pay more for the fees it produced. Since a shares are public and those which are not valid can be spotted easily, one possible way the pool can cheat is by reordering shares by putting them in a slice with reference job that a a lower fee. But it is not trivial, because the job of the share must be compatible with the reference job, and fixing it involves timestamps or pointers, but this is outside the scope of the article.

-------------------------

lorbax | 2024-12-10 09:41:54 UTC | #43

If the JDS is an independent service, then we must take into account the fact that can service for different pools. In this setting, I think that each pool must communicate the JDS the coinbase outputs and each miner must communicate the pool that it is working for. Apart from this, seems to work exactly as it currently working in the SRI.

-------------------------

sjors | 2024-12-17 07:49:44 UTC | #44

[quote="lorbax, post:42, topic:1099"]
then this score_f(i) \approx 0scoref(i)≈0
[/quote]

Try the following numbers:
* fake fees: 1 million BTC
* shares: 0.0001% of the pool for the interval

So they could steal approximately 1 BTC.

[quote="lorbax, post:42, topic:1099"]
In this case, the JDS validates each share.
[/quote]

I assume you mean each proposed template (`DeclareMiningJob`), not each share?

>  If there is some unknown transaction during validating the shares, then it will be asked to the miner through “PovideMissingTransactions” message, as for Sv2 protocol.

This takes time and bandwidth. And in order to verify if the new transaction is actually valid the JDS has to insert it into its mempool. And in order to do that it may need to evict other conflicting transactions first. Doing this for every `DeclareMiningJob` may not scale very well.

[quote="lorbax, post:42, topic:1099"]
> For coinbase-only templates the JDS doesn’t know the transactions and can’t verify anything.

I understand that “coinbase-only templates” is a share with just the coinbase transaction, so it is an “empty weak block”. But then you refer at the transactions, so there is more than one. I do not understand, can you explain?
[/quote]

I'm referring to a recent proposal that allows a `DeclareMiningJob` message with just a merkle proof for the coinbase transaction, without revealing which transactions are in the block.

I can't find the link to the actual propososal, just a reference to in from the SRI call minutes: https://discord.com/channels/950687892169195530/1019861139573714994/1314469730781757500:

> `coinbase_only` Mode and `DeclareMiningJob`
> * **Optionality for `DeclareMiningJob` :** Its usage depends on the mode.
> * **Flag Renaming:** The flag `REQUIRES_ASYNC_JOB_MINING` in
> `AllocateMiningJobToken.Success` will be renamed to better reflect its purpose.
> * **Dropping Synchronous Case:** Only two modes will remain:
>  * **`coinbase_only JD` Mode:** No `DeclareMiningJob` is sent.
>  * **`template JD` Mode:** `DeclareMiningJob` is sent.

-------------------------

Fi3 | 2024-12-17 09:03:17 UTC | #45

[quote="sjors, post:44, topic:1099"]
Try the following numbers:

* fake fees: 1 million BTC
* shares: 0.0001% of the pool for the interval

So they could steal approximately 1 BTC.
[/quote]



I think there is a misunderstanding here. The pool is supposed to verify all the shares/jobs, the miners only a random subset. So this attack is not possible.

-------------------------

Fi3 | 2024-12-17 09:06:00 UTC | #46

[quote="sjors, post:44, topic:1099"]
This takes time and bandwidth. And in order to verify if the new transaction is actually valid the JDS has to insert it into its mempool. And in order to do that it may need to evict other conflicting transactions first. Doing this for every `DeclareMiningJob` may not scale very well.
[/quote]

not the most scalable solution but is the best trade off that we have been able to find, and a pool can support it.

-------------------------

Fi3 | 2024-12-17 09:07:19 UTC | #47

[quote="sjors, post:44, topic:1099"]
I’m referring to a recent proposal that allows a `DeclareMiningJob` message with just a merkle proof for the coinbase transaction, without revealing which transactions are in the block.

I can’t find the link to the actual propososal, just a reference to in from the SRI call minutes: [Discord](https://discord.com/channels/950687892169195530/1019861139573714994/1314469730781757500:)

> ``
[/quote]

this system do not work with coinbase only

-------------------------

sjors | 2024-12-17 11:43:26 UTC | #48

[quote="Fi3, post:45, topic:1099"]
I think there is a misunderstanding here. The pool is supposed to verify all the shares/jobs, the miners only a random subset. So this attack is not possible.
[/quote]

No, I understand that's the intention of your proposal. I just don't think verifying all jobs is realistic.

 SRI doesn't implement it yet, it only checks that all transactions are present. If the checks are incomplete there will always be some exploit. And if the checks are complete, it may be too easy to DoS the JDS.

-------------------------

Fi3 | 2024-12-17 12:41:02 UTC | #49

why you say that is easy to DoS the JDS? Do you have some numbers?

-------------------------

sjors | 2024-12-17 13:27:21 UTC | #50

You could send infinitely many block proposals to it that each take a long time validate. See https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710
The blocks can be invalid, so it's a free attack.

Fortunately there's at least two mitigations for this attack:

1. Don't validate anything until you have received some threshold worth of shares
2. Don't allow non-standard transactions in the template

Simply checking if all transactions are in the JDS mempool is not enough. There might be an unknown transaction. Those are fetched via an sv2 message, but the transactions that obtained this way could all be slow to validate. And in order to check them, e.g. to make sure they're not spending high value fake coins, you need to insert them into the JDS mempool. But maybe there's conflict in the pool, so you have to evict some other transactions.

The best way to fully check if the proposed block is valid is by having the node verify it just like any other block, but without checking the PoW. There's currently no RPC method to do this, e.g. `submitblock` requires proof-of-work. It may be useful to introduce such a method though.

-------------------------

Fi3 | 2024-12-17 13:53:13 UTC | #51

I'm assuming that you can do it faster. At demand we rebuilt the mempool. Another assumption is that user are authenticated (otherwise there are other ways to DoS the pool), so I can at least detect who is building unusual block. For unknown txs we check if the tx is spendable and do not have conflict with other txs in the proposed block this is enough.

-------------------------

Fi3 | 2024-12-17 14:09:53 UTC | #52

about solution (1) a pool can implement it outside of this proposal, for example the pool can send some work (so we are sure that work is valid) and activate the job declaration protocol and this extension only after that downstream sent some shares.

solution (2) is as well an implementation details, in this proposal we are not saying if pool can or can not deny some txs, and which should be denied.

-------------------------

Fi3 | 2024-12-17 13:59:40 UTC | #53

Personally I like more (1) then (2) since I think that (2) go against the goal of sv2

-------------------------

sjors | 2024-12-21 10:26:28 UTC | #54

[quote="sjors, post:50, topic:1099"]
The best way to fully check if the proposed block is valid is by having the node verify it just like any other block, but without checking the PoW. There’s currently no RPC method to do this, e.g. `submitblock` requires proof-of-work. It may be useful to introduce such a method though.
[/quote]

Here's a proof-of-concept implantation of a `checkblock` RPC for Bitcoin Core:

https://github.com/Sjors/bitcoin/pull/75

It can be used with block templates, which don't have any proof-of-work. But it also has `multiplier` argument that can be used to raise the target for weak blocks, inspired by @instagibbs work in https://delvingbitcoin.org/t/second-look-at-weak-blocks/805 (though a different use case).

-------------------------

Fi3 | 2025-01-09 11:16:34 UTC | #55

committing the share_index and also return it to the miner is a nightmare to implement right and efficiently. If the only reason to have it is to avoid the pool to lie about share indexes in slice (when challenged) we can remove it since the path is enough to know the index of a leaf and the pool already have to provide the path.

-------------------------

