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

