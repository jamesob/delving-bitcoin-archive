# Would OP_SUCCESS (OP_CAT) be spent?

Nuh | 2026-02-03 19:40:20 UTC | #1

I was wondering if miners would actually steal a utxo locked with OP_SUCCESS or would they honor the intent behind it …

So I locked some funds to the unspendable key `50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0` with this one script: `30202d20444f204e4f54205350454e4420574954484f5554204f505f43415420284249502d3334372920535550504f52547e4a4920535550504f5254204f505f43415420284249502d33343729202d20444f204e4f54205350454e4420574954484f5554204f505f43415420284249502d3334372920535550504f525487`
`<" - DO NOT SPEND WITHOUT OP_CAT (BIP-347) SUPPORT"> OP_CAT <“I SUPPORT OP_CAT (BIP-347) - DO NOT SPEND WITHOUT OP_CAT (BIP-347) SUPPORT”> OP_EQUAL`

The utxo is confirmed at `26af7ad831751354e600d3c4b373b02b6fafe1277adaec9d4a79a28de77a427b:0`

I tried to spend it with Mara’s Slipstream service, but even they seem to respect that OP_SUCCESSx should not be spent until a soft fork.

I am writing here to try make this a bit more public, and see if any miner would spend it, and if they did, would they at least pretend to follow the OP_CAT rules by providing a valid witness.

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

AntoineP | 2026-02-03 20:07:58 UTC | #5

The point of upgrade hooks is to provide a risk-minimized way of rolling out upgrades on the network. If miners included such transactions, any soft fork that gives them meaning would make them trivially DoSable and a the perfect proxy for an attacker to create deep reorgs.

Lack of interest from others could well be the reason BIP 347 is not activated, but that's not a reason to pile on top practical reasons for not activating it. Besides, i don't think apathy from miners is relevant here: a soft fork is a change of the consensus rules enforced by Bitcoin users, not a vote by miners.

-------------------------

