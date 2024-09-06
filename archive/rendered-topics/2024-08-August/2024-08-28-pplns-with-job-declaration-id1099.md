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

Fi3 | 2024-09-06 15:02:50 UTC | #13

No Share is a custom datatype and is encoded like described in the extension spec.

Is implemented here: https://github.com/demand-open-source/share-accounting-ext/blob/master/src/data_types.rs#L174

-------------------------

marathon-gary | 2024-09-06 15:03:55 UTC | #14

The client gives the server a list of id's (how id this list of IDs determined?) of shares they want to validate, I understand this piece. There is just no guarantee that the pool is providing accurate information. This is the same as using an API to query for shares.

What stops a pool from providing inaccurate or misleading share data? or omitting shares when requested?

-------------------------

