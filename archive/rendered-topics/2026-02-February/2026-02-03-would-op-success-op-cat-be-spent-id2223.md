# Would OP_SUCCESS (OP_CAT) be spent?

Nuh | 2026-02-04 07:59:57 UTC | #1

I was wondering if miners would actually steal a utxo locked with OP_SUCCESS or would they honor the intent behind it …

So I locked some funds to the unspendable key `50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0` with this one script: `30202d20444f204e4f54205350454e4420574954484f5554204f505f43415420284249502d3334372920535550504f52547e4a4920535550504f5254204f505f43415420284249502d33343729202d20444f204e4f54205350454e4420574954484f5554204f505f43415420284249502d3334372920535550504f525487`
`<" - DO NOT SPEND WITHOUT OP_CAT (BIP-347) SUPPORT"> OP_CAT <“I SUPPORT OP_CAT (BIP-347) - DO NOT SPEND WITHOUT OP_CAT (BIP-347) SUPPORT”> OP_EQUAL`

The utxo is confirmed at `26af7ad831751354e600d3c4b373b02b6fafe1277adaec9d4a79a28de77a427b:0`

I tried to spend it with Mara’s Slipstream service, but even they seem to respect that OP_SUCCESSx should not be spent until a soft fork.

I am writing here to try make this a bit more public, and see if any miner would spend it, and if they did, would they at least pretend to follow the OP_CAT rules by providing a valid witness.

You can watch the balance and any withdrawals to this address [bc1p22kppn826lx9qtluelwqdvq4fuupnwmrtx060aanpgpdanmphddstsg5vc](https://mempool.space/address/bc1p22kppn826lx9qtluelwqdvq4fuupnwmrtx060aanpgpdanmphddstsg5vc) , or maybe send funds to it if you want to contribute to this experiment.

-------------------------

Nuh | 2026-02-03 19:41:53 UTC | #2

Transaction I tried to submit to Mara slipstream `010000000001017b427ae78da2794a9decda7a27e1af6f2bb073b3c4d300e654137531d87aaf260000000000ffffffff01123400000000000022512052ac10ccead7cc502ffccfdc06b0154f3819bb63599fa7f7b30a02decf61bb5b027e30202d20444f204e4f54205350454e4420574954484f5554204f505f43415420284249502d3334372920535550504f52547e4a4920535550504f5254204f505f43415420284249502d33343729202d20444f204e4f54205350454e4420574954484f5554204f505f43415420284249502d3334372920535550504f52548721c050929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac000000000`

-------------------------

AntoineP | 2026-02-03 19:46:36 UTC | #3

Incentivizing miners to disrespect upgrade hooks is, frankly, stupid. It appears that you would like to see BIP 347 get activated, but burning its upgrade hook is the surest way to hinder progress and create more work for its authors.

-------------------------

Nuh | 2026-02-03 19:54:40 UTC | #4

What if that utxo is never spent, doesn’t that tell us something interesting? 

Why would Mara censor OP_SUCCESS code, when they didn’t censor other non-consensus rules?

I apologize for any inconvenience to the authors, but my impression is that what is actually killing all these soft-forks is pure apathy, especially from miners.

-------------------------

AntoineP | 2026-02-03 20:27:46 UTC | #5

The point of upgrade hooks is to provide a risk-minimized way of rolling out upgrades on the network. If miners included such transactions, any soft fork that gives them meaning would make them trivially DoSable and the perfect proxy for an attacker to cheaply create deep reorgs.

Of course i don't think your 19,031 sats utxo alone will incentivize pools to start mining OP_SUCCESS spends. But i wanted to call out this practice early because default mining/relay policy is a brittle mechanism (as i hope the past few years have demonstrated) that makes it a lot safer (i.e. all other things equal likelier) to deploy soft forks.

Lack of interest from others could well be the reason BIP 347 is not activated, but that's not a reason to pile on top practical reasons for not activating it. Besides, i don't think apathy from miners is relevant here: a soft fork is a change of the consensus rules enforced by Bitcoin users, not a vote by miners.

-------------------------

Nuh | 2026-02-03 20:38:58 UTC | #6

Thank you for the feedback…

> any soft fork that gives them meaning would make them trivially DoSable and the perfect proxy for an attacker to cheaply create deep reorgs.

I don’t suspect you are wrong, I am just not sure I understand this attack. Either OP_SUCCESS is valid or it isn’t, and its behavior is only restricted after an objective signaling threshold, why would a reorg happen?

Especially when miners can preemptively censor OP_SUCCESS unless it follows restrictions, just like they currently seem to censor them entirely.

> Of course i don’t think your 19,031 sats utxo alone will incentivize pools to start mining OP_SUCCESS spends.

It doesn’t need to be pools though does it?

Also, I would very much like to figure out what is the threshold after which miners start caring.

I don’t mean to be disruptive, but I am not convinced that this is disruptive at all.

Thanks again for responding regardless.

-------------------------

AntoineP | 2026-02-03 21:12:39 UTC | #7

[quote="Nuh, post:6, topic:2223"]
why would a reorg happen?
[/quote]

If a miner includes OP_SUCCESS spends, they may include a transaction that is invalid according to a future soft fork they are not aware of. In this case they would create an invalid block, and other unupgraded miners may build on top even though they themselves do not include OP_SUCCESS spends in their blocks.

[quote="Nuh, post:6, topic:2223"]
Especially when miners can preemptively censor OP_SUCCESS unless it follows restrictions
[/quote]

So that would be miners essentially enforcing proposed soft forks by standardness. That's a different thing from just mining all valid transactions, and i think presents at least three types of challenges:
- scaling: which proposals should a mining pool implement? When should it stop enforcing each? How closely should it monitor progress on each (see next point)?
- DoS risks to the miners and the network: what if a proposal is updated? Any change in semantics can be exploited as per the above.
- incentive: once you open the pandora box, why would a miner collect fees only for transactions that respect the proposal? They are all consensus valid anyways and would not lead to its blocks being rejected.

[quote="Nuh, post:6, topic:2223"]
It doesn’t need to be pools though does it?
[/quote]

Right, any miner.

[quote="Nuh, post:6, topic:2223"]
Also, I would very much like to figure out what is the threshold after which miners start caring.
[/quote]

I'd rather not find out. :slight_smile:

[quote="Nuh, post:6, topic:2223"]
I don’t mean to be disruptive, but I am not convinced that this is disruptive at all.
[/quote]

I don't think your isolated instance is really disruptive. But we've seen in recent years that it's easy to kickstart a trend to disrespect widely-enforced standardness rules on the network. That has the potential of becoming disruptive very quickly. And as much as i didn't care too much about having to increase the size of standard `OP_RETURN`s or decreasing the minimum relay feerate, i would very much like that we be able to rely on upgrade hooks for at least a little longer.

-------------------------

Nuh | 2026-02-03 21:41:12 UTC | #8

[quote="AntoineP, post:7, topic:2223"]
scaling: which proposals should a mining pool implement? When should it stop enforcing each? How closely should it monitor progress on each (see next point)?

[/quote]

Seems to me a limitation in signaling mechanism more than anything else. It can be solved by changing the activation mechanism to require a more obvious signaling than flipping bits that are often flipped for grinding in PoW (I hear).

Wouldn’t an explicit signaling of BIP number + OP_SUCCESS code, make it suddenly clear to oblivious miners that there is an increasing signaling for limiting an OP_SUCCESS behavior? in which case miners (or their software) can choose to not mine any transaction using that opcode, and favor building on top of blocks that don’t have it.

[quote="AntoineP, post:7, topic:2223"]
* DoS risks to the miners and the network: what if a proposal is updated? Any change in semantics can be exploited as per the above.

[/quote]

Again the signaling seems the solution here, a commitment to an exact release / implementation / commit prefix can be sufficient.

[quote="AntoineP, post:7, topic:2223"]
incentive: once you open the pandora box, why would a miner collect fees only for transactions that respect the proposal? They are all consensus valid anyways and would not lead to its blocks being rejected.

[/quote]

If signaling is explicit enough, they are free to choose whether or not risk it.

In the absence of explicit signaling, I imagine miners already try avoid OP_SUCCESS altogether for the reasons you explained, and probably would favor building on top of blocks that aren’t including OP_SUCCESS.

I am just saying this is not that concerning, and even if it was, the solution isn’t “calling it out”, the solution is miners doing what the need to do for their own sake, and maybe we should do a better job at signaling soft-forks unambiguously. 

[quote="AntoineP, post:7, topic:2223"]
i would very much like that we be able to rely on upgrade hooks for at least a little longer.

[/quote]

I would very much like to see a soft-fork before I die :’D… all jokes aside, I am mostly ignorant, but maybe my suggestion about clearer signaling method is the way to achieve that safer upgrade hooks.

Because, as it stands, miners are totally checked out, signaling absolutely nothing, maybe you think that is because nothing is worth signaling for (in which case why do you care for upgrade hooks?), or maybe there is something else needs fixing about the process.

-------------------------

sipa | 2026-02-04 14:50:01 UTC | #9

Soft forks, or consensus changes in general, are not a popularity contest by miners. They are transitions of the entire ecosystem to a new set of rules, and then demanding that miners enforce those rules (or risk having their blocks being considered invalid by the ecosystem).

Signalling is a mechanism to get everyone to agree on the point in time **when** this enforcement will start, because things are better for everyone if this happens in a clean, coordinated, manner. But signalling isn't about determining **whether** a soft fork will happen - that is decided by Bitcoin users (of which miners are a part, but don't have a privileged position). If miners were to enforce consensus rules *without* ecosystem agreement, we'd have a real case of censorship.

I admit I have not been following this discussion too closely, as I'm trying not to be involved in Bitcoin consensus changes anymore (I've had my fair share...), but as of right now, as far as I can tell, there aren't any proposed changes that have widespread agreement from the development community and user ecosystem at large. This is something that would translate to having full node software out there *and deployed* that is ready to trigger when activation thresholds are met. Absent those, soft fork signalling is just virtue signalling without effect.

It's certainly possible that apathy is a factor that is getting in the way of your favorite consensus change becoming reality. But it's not apathy by miners: they only come into play *after* the ecosystem has agreement that a change is to happen.

-------------------------

Nuh | 2026-02-04 17:36:54 UTC | #10

Thank you @sipa for your feedback.

It seems though that miners are currently censoring me using OP_SUCCESSx as defined by consensus, even though I didn’t notice any ecosystem agreement. I admit I might be ignorant, but from my point of view, I see much more explicit support for almost every soft-fork proposal, including BIP-110, than censoring OP_SUCCESS… the only privilege OP_SUCCESS censoring has, is being the standard behavior of Bitcoin Core.

The purpose of this exercise, is just verifying:
1. If miners are indeed motivated with extracting as much value as possible, or are they too scared to anger the “ecosystem”.
2. Are miners actively censoring valid, and well-paying transactions.
3. What happens when we “have a real case of censorship”, will anyone care? or is censorship (and the centralization necessary for it) is actually built-in in people’s assumptions, for things like “alignment” and filtering “bad content” out of the blockchain.

If we are taking that censorship for granted, but refuse to acknowledge it, I would like to know that.

-------------------------

Nuh | 2026-02-04 18:17:50 UTC | #11

@sipa I also want to ask another question… what is so wrong about *this type* of censorship?

To be clear, an oracle emulating an op_code or an entire different type of locking predicate, is censorship, not enforced by the entire network, but by a specific predetermined group of signers.

Now, what is wrong with current PoW majority, enforcing some “censorship” over a very specific pattern that is impossible for anyone to be using without opting-in that behavior?

Put another way, why is this ok to be enforced by oracles\[1\]:
`<INPUT> <PROGRAM> OP_2DROP <oracle_1_pk> OP_CHECKSIG`

But this is not ok to be enforced by miners:
`<INPUT> <hash(PROGRAM)> OP_NOP10 OP_2DROP OP_1`
where `<hash(PROGRAM)> OP_NOP10` is basically used as an ad-hoc (yet only opt-in) `OP_CHECKPROGRAMVERIFY`

Would that censorship be any cause of concern? And if not, who other than miners (persuaded by a minority of users interested in such use-case) are supposed to be involved in that “agreement”.

Now if that \_is\_ acceptable arrangement, if only risk for the users trusting miners (just as it is risky for users to trust oracles), does that mean the only problem with using `OP_SUCCESS126` is the scarcity of `OP_SUCCESSS` codes?

Are miners then allowed to signal their intent to censor transactions (and blocks) that doesn’t respect a specific program’s hash? Or is that considered an attack on Bitcoin, because a broader “ecosystem agreement” hasn’t been reached? Of course other miners are affected, and will be required to react, but so would they in the case of an OP_SUCCESS soft fork. The rest of the ecosystem need not concern itself, no?

Thanks again for engaging with this.

\[1\] A more detailed example of how oracle-based emulation can be done that makes more sense than the script above https://github.com/Nuhvi/sake

-------------------------

1440000bytes | 2026-02-09 12:31:58 UTC | #12

[quote="Nuh, post:4, topic:2223"]
I apologize for any inconvenience to the authors, but my impression is that what is actually killing all these soft-forks is pure apathy, especially from miners.
[/quote]

Most users look for miners signaling in a soft fork, even though it is said that miners only signal readiness and that it’s not a vote.

You can look at this donut chart to understand who should be contacted:

![image|690x366](upload://dkRnxfsH3wZF8aVDJAP9bLx1bZJ.png)

Note: These mining pools (foundry, antpool, f2pool) also influence default relay policy in core.

<details>
<summary>Archive</summary>

https://archive.is/cldLr

</details>

-------------------------

murch | 2026-02-13 18:32:12 UTC | #13

What you are doing equates to building up a bounty for miners to start disrespecting upgrade hooks.

[quote="Nuh, post:4, topic:2223"]
What if that utxo is never spent, doesn’t that tell us something interesting?

[/quote]

It’s a gentlemen’s agreement that miners respect upgrade hooks. It’s absurd to expect that they’d honor the intent of an OP_SUCCESS redefinition they probably have never even heard about. You will not be able to use the opcode in your intended manner, because miners will not follow your hypothetical rules. They will ignore the OP_SUCCESS-laden scripts until there is enough money for someone to go to the trouble of starting to take it instead.

If your goal is to see more soft forks, you are self-sabotaging: bribing miners into disrespect upgrade hooks will make it messier and therefore harder to deploy soft forks.

Please find a more positive approach to push for protocol improvements than torching common goods.

-------------------------

Nuh | 2026-02-13 19:15:29 UTC | #14

Thanks for your elaboration. 

Saying that the upgrade hooks are a common good worth caring about assumes that upgrades are actually feasible, I see no evidence on that whatsoever.

Moreover, I believe that the threshold everyone seems to demand “ecosystem agreement” *otherwise miners are considered attackers* is clearly impossible, and it doesn’t apply to any previous soft fork, except because Bitcoin had a core of influential developers that were acting not-unlike Ethereum Foudnation or Rootstock Labs etc.., To their extreme credit, these developers have recognized that and stepped aside, even before the stakeholders outgrew their influence.

If one agrees that we only have between 0-1 soft-forks in our future, and the only way to get that is to dare everyone to spend a utxo wrong and risk a fork, then OP_CAT might be the best option for that dare, because I don’t think GSR will be here soon.

Still, I won’t put any more funds in that address, because I might be wrong, but I hope others try this aggressive approach, or we can just be happy with Rootstock-like merge mining and ugly BitVM bridges, because we won’t have anything else ever, if we wait for the mythical ecosystem to mythically reach an agreement.

-------------------------

murch | 2026-02-13 19:42:33 UTC | #15

I am confident that a sufficiently attractive soft fork can be deployed. The absence of such a proposal at this time doesn’t prove a negative.

-------------------------

Nuh | 2026-02-13 20:17:27 UTC | #16

A sufficiently attractive proposal to whom? because there is nothing I personally want more than a trustless bridge to escape this nightmare into a metaprotocol that is turing complete already, and enjoy things like vaults, shielded CSV, and bridges to scalable L2s. All of that is demonstrably possible with OP_CAT, so that is sufficiently attractive to me.

If I don’t matter, who does? how many do? If you don’t have an answer to that, then by default the answer is we will never have anything sufficiently attractive.

Also, objectively, the only proof that something is sufficiently attractive, is if their proponents are willing to exert pressure on miners to “censor” invalid transactions, so any action no matter how aggressive or insulting to the sensibility of some, would be more productive than spending years arguing with people with no skin in the game who can just say “NACK” or “Shitcoiner” and be taken seriously as a veto.

That being said, I wouldn’t push this any further, because the internet is full of weirdos as Gloria just realized, and it is way easier to just trust multisig federations.

-------------------------

