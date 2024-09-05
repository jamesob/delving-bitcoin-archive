# PPLNS with job declaration

Fi3 | 2024-08-28 14:21:15 UTC | #1

I'm sketching up an sv2 extension, that allow miners to verify pool's payouts when miners are allowed to select transaction hence mining jobs with different fees.

The extension is based on a system proposed [here](https://github.com/demand-open-source/pplns-with-job-declaration/blob/bd7258db08e843a5d3732bec225644eda6923e48/pplns-with-job-declaration.pdf) this system is build on top of PPLNS.

You can find the extension spec [here](https://github.com/demand-open-source/share-accounting-ext/blob/281c1cbc4f9a07b21a443753a525197dc5d8e18c/extension.md)

[Here](https://github.com/demand-open-source/share-accounting-ext/tree/master/src) instead I implement the extension messages using SRI libraries.

[Here](https://github.com/demand-open-source/demand-cli) I started implementing a miner translator proxy that use the extension.

Everything above is still work in progress and need reviews.

-------------------------

plebhash | 2024-09-05 19:59:10 UTC | #2

I'm Spiral grantee working as a contributor to StratumV2 Reference Implementation (SRI).

In a time where the Bitmain FPPS debt-slavery cartel is pushing Bitcoin towards mining centralization, this post is extremely relevant and I would love to see more engagement from mining players here.

The ideas presented on the paper titled [PPLNS with Job Declaration](https://github.com/demand-open-source/pplns-with-job-declaration/blob/5e3e7666f177e9a1e217e72da65a35b612505613/pplns-with-job-declaration.pdf) are academically sound and it's refreshing to see that [Demand Pool](https://www.dmnd.work/) has some talented minds proposing a tangible path out of the dark situation Bitcoin is currently under.

Remember that while SV2 allows for hashers to choose their own templates via Job Declaration (JD), the protocol itself is inherently agnostic to:
- share accounting
- reward distribution

So when a pool decides what kind of algorithms to employ to solve those two specific problems, their design space is limited to:
- hashers blindly trusting the pool is fair
- feeding back information to hashers to minimize trust

And if they choose the second path, they will inevitably need protocol extensions.

Now, SV2 decentralizes via JD! 

Which means hashers have the right to choose a template that could be economically suboptimal with regards to fee revenue... that could happen for different reasons:
- maybe the hasher's Template Providing (TP) node is suboptimally connected (remote location, poor internet, plebhashers) and the "mempool" they see is suboptimal.
- maybe they are ideologically driven and see some kinds of transactions as spam (which is a subjective term, albeit introduced by Satoshi and ingrained into protocol primitives).
- maybe they decide to filter out consensus-valid and high-fee txs that would hurt low-end nodes (see [GCC](https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710) for more).

So an economically rational pool needs a mechanism to still allow for low-revenue jobs, while rewarding them fairly with regards to more economically optimal jobs, which have to be rewarded more.

And in order for hashers to be willing to put some level of trust on the pool, they need to be fed back some info to give them reassurance about how they are being rewarded in proportion to other hashers. That is exactly what this new SV2 protocol extension proposes, and I'm happy it's built on top of PPLNS, rather than FPPS.

I'm looking forward to following [this Discussion on SRI repo](https://github.com/stratum-mining/sv2-spec/issues/95), where Braiins and Demand engineers are shaping up the implementation details for the first ever SV2 protocol extension.

-------------------------

plebhash | 2024-09-05 20:00:37 UTC | #3

As a pleb advocate, I'm particularly curious how the proposed SV2 extension would affect GCC txs (meaning they would hurt low-end nodes, even if they pay high fees and are included in the so called "Standard/Canonical/Platonic" mempool).

How should a SV2-JD-enabled Pool take that into account? I see two options:
- A. reject all jobs that include txs that fill this criteria (as a JD policy)
- B. impose economical penalties to jobs that include txs that fill this criteria (as a reward policy)

The proposed extension is not really relevant for option A, since the basic SV2 primitives already allow for that.

But option B does have some relevance, and I'm curious as to whether this issue is being taken into account.

-------------------------

