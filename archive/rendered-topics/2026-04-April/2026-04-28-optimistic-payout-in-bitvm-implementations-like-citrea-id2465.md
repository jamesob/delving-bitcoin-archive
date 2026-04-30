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

ekrembal | 2026-04-28 21:07:41 UTC | #2

Hi,

Lead dev of Clementine here, and thank you for your interest.

"No one else can get out invalidly" holds because signers verify the batch proof containing the withdrawal to be finalized on Bitcoin before signing any optimistic withdrawal.

The main problem with doing a key-deletion covenant with BitVM bridges is that signers would need to have an authorization key with which they authorize their new key for each deposit. This creates the same problem: if all of the N-of-N authorization keys are compromised, an attacker can do fake deposits and drain the bridge.

In other words, a similar attack is still possible even if signers delete their keys after presigning. But by not deleting keys, the optimistic withdrawals give great UX without changing any trust assumptions.

-------------------------

AdamISZ | 2026-04-28 22:09:25 UTC | #3

[quote="ekrembal, post:2, topic:2465"]
The main problem with doing a key-deletion covenant with BitVM bridges is that signers would need to have an authorization key with which they authorize their new key for each deposit. This creates the same problem: if all of the N-of-N authorization keys are compromised, an attacker can do fake deposits and drain the bridge.
[/quote]

Interesting. So the risk I see (pretty obvious from my post but I'm sure you already thought about it!) is the issue of continuous custody: if the signing keys are always available (according to honest protocol-following), attackers of any sort that can compromise all N can get the money.

I see two scenarios: a large signing committee for each setup with minimum trust placed on any entity involved (a la toxic wasted trusted setup, though of course that's the extreme); e.g. 1 of 100 with at least quasi-open involvement. It's understood that however you slice it if all 100 are rogue you are screwed. Doing this repeatedly is of course not trivial.

Other extreme is what you have: relatively small set that essentially don't even claim to give up custody as a group: the biggest difference is you aren't deleting the relevant signing keys, even if you're honest.

Your point that each new deposit (each new setup) would need authorization: that seems to depend on your notion of what the verifier/signer role is. Considering the first of the above two alternatives, keys could be created afresh for each new setup and I as a user am not trusting that it's the "right" set of signers, I'm trusting that at least 1 of them isn't an attacker [1]. *If* my assumption is valid, then I don't suffer a continuous risk from external attackers. In the ideal extreme, I myself as a user could take part in that signing setup when and if I wanted to. With your other model of fixed entities and continuously available keys, I do suffer from that risk, even if my assumption is valid.

> the optimistic withdrawals give great UX

That's the thing, it depends on how you look at UX. In assessing the system from my point of view as a potential user, what I see there is actually a negative for my UX: I want a system that is not exposed to external attackers or regulatory pressure (notice: even though N of N is a high bar for a hack-attacker, that's less strong against state-level control vectors). This feature makes one thing easier (withdrawal) but exactly that ability makes me much more concerned about the possibility of systemic failure, making me much less enthusiastic about using it (whereas a somewhat delayed exit wouldn't really bother me).

[1] Thinking about it, you could sign over that new key with your real-world identity, at the time of that setup, as a way of defending against the sybil problem, in the most extreme version of "anyone can take part", not that I'm arguing for one very specific way of doing it.

-------------------------

ekrembal | 2026-04-29 23:09:30 UTC | #4

One thing to note: when a user makes a deposit, and later does a withdrawal, the UTXO used for that withdrawal is not the deposit UTXO the user originally created. Withdrawals are paid out like a  FIFO, the i-th withdrawal is paid from the i-th deposit.

That's why all deposits need to be presigned with the same fixed set of signers, in order to preserve uniform trust assumptions across the bridge. A user withdrawing today might be paid from a deposit made a year ago, so the signers backing that older deposit are effectively the signers backing the user's withdrawal too. This is why solutions like "trust whoever is live during your deposit out of a set of 100 signers" don't actually help. (I am not sure if this is the first scenario you mentioned. Pls correct me if not.)

We did initially consider designs with a fixed signer set that also allowed anyone to participate in the signing ceremony as a volunteer. But this turns out to be non-trivial and doesn't actually improve trust assumptions: griefing a MuSig2 signing ceremony is easy, and there's no way for the rollup to verify that all volunteer signers were actually included. So the attack I mentioned earlier is still possible.

-------------------------

