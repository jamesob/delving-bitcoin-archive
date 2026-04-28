# Optimistic payout in BitVM implementations like Citrea?

AdamISZ | 2026-04-28 18:27:14 UTC | #1

I should preface all this by saying I am pretty enthusiastic about this technology in general; I'm asking, here, whether one specific detail is wrong.

Been doing some investigation of [Citrea](https://github.com/chainwayxyz/clementine/tree/main) as part of generally researching the L2 designs clustered around BitVM. This one is particularly interesting for the obvious reason that it is already [live](https://citrea.xyz/batch-explorer?page=1&limit=10) on mainnet!

It seems to have fleshed out basically all the parts of the BitVM2, "BitVM core" designs. But in reading the [Clementine whitepaper](https://eprint.iacr.org/2025/776) (Clementine is the business end of what they're building; the "Bitcoin bridge" part), I was struck by Section 8. They write:

> In this section, we discuss some features and optimizations of Clementine. Optimistic Payout. The protocol we described above guarantees that any peg out is completed even if all Signers are offline and all but one are malicious. However, if all Signers are honest and online, they have some time (in Clementine, it is ≃ 1 hour) to sign an issue a user’s peg out by posting an OptimisticPayout transaction. This transaction resembles the Payout transaction, with only two differences:(i) it spends the output of the MoveToVault transaction, so that the funds given to the user do not come from the Operator, and (ii) there is no OP RETURN output. If no OptimisticPayout transaction appears on-chain within some time, the peg out request is picked up by the Operator and the Clementine continue as described in Section 5. To enable the optimistic payout, Signers must not erase their keys, making the protocol secure against a non-adaptive adversary.

This made me take notice; wait, so you have a complex BitVM style exit proof with a claim/challenge protocol etc., to avoid trust - but you can also get out with a vanilla transaction signed by all the signing committee [1] (they're called "verifiers" in the codebase)?

From one angle such a facility makes sense: you are trusting 1 of n honesty of the signing committee in setup, so 1 of n honesty should also guarantee that the same signing committee doesn't sign malicious exits. But that's bad reasoning from almost every other angle:

* BitVM-and-friends' use of a signing committee is strictly the covenant emulation part: co-signing at setup only, and key deletion immediately after. If we had covenant functionality in Bitcoin, obviously even this step wouldn't be used, as it's an unpleasant extra "feature".
* If the same signers (even with different keys, as here) are *also* trusted to co-sign optimistic exits live, you might think "no big deal, I don't have to trust them, I'll use the trustless exit, even if it's slower"
* But no! The property a user wants is not only "I can get out with a valid request" but also "no one else can get out invalidly"! If others can get signatures on exits to arbitrary destinations, they can steal all the funds on the L2
* Even if technically you can restrict it so that exits have to go to user-defined destinations at setup (arguably, doesn't make sense, operators are exiting), that still wouldn't fix it; if a user currently holding 0 cBTC can exit with 10 cBTC, you get an unbacked L2 and a bank run scenario via the operators. Details depend on technicals, but the general principle is kinda clear enough.

[Audits](https://github.com/chainwayxyz/clementine/tree/main/audits) on the codebase don't cover this type of thing; they only address bugs in implementation. Indeed, [this doc](https://cantina.xyz/competitions/ce181972-2b40-4047-8ee9-89ec43527686) seems to specify the scope of the audit, and explicitly says:

> * **Withdrawal behavior:**
>  * If all Verifiers are live, they will send an **optimistic payout transaction** for a withdrawal.
>  * Every transaction signed during the deposit phase must be sufficient to disprove any malicious activity.

If I'm reading that right (I may well not be) it seems to imply an argument "if the verifiers=signers sign malicious optimistic withdrawals, it'll be a provable fraud and that's our security model". I don't think that's at all good enough for this kind of system. Not only do you have a threat of N of N malicious verifiers colluding, you also have a hacking threat of quasi-live signing keys, and most importantly a regulatory pressure threat because that quorum is essentially a custodian - we can clearly see how bad that can be in e.g. Liquid where the only exits allowed are whitelisted ones.

There's a decent chance I've just misunderstood something but I did look at the code (with Claude's help) and it seems that what I'm describing is actually how it works.

And to end: what I don't get is, why can't the system exist without this big security hole? Is the concern that funds might get permanently stuck and that's not acceptable? Or that exits might be too slow and that's not acceptable? Genuinely curious, because I don't think such a system is viable with such a security hole.

I also haven't gotten as far as looking at various other rollup-ish designs that I know exist based on BitVM protocols, so I don't know if all this is 100% exclusive to Citrea/Clementine, or not.

[1] In the [codebase](https://github.com/chainwayxyz/clementine/blob/05712dde61a80f6b3115f5273f6e72434cd61e6c/core/src/verifier.rs#L1610-L1632) there appears to be an extra, but optional, signature required from the Aggregator, but I don't think that changes much of what I'm saying here, so I set it aside.

-------------------------

