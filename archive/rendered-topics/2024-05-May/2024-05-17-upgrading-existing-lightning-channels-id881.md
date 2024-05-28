# Upgrading Existing Lightning Channels

carla | 2024-05-17 17:04:07 UTC | #1

I’ve been trying to wrap my head around the various ways we could go about upgrading lightning’s existing channels, specifically in the context of upgrading to use V3 channels to improve our resilience against pinning attacks. I hope that this is a decent overview of what’s out there - and if not, three cheers for cunningham’s law.

V3 transactions are [on their way to being standard](https://github.com/bitcoin/bitcoin/pull/29496) and now have [sibling eviction](https://github.com/bitcoin/bitcoin/pull/29306), which means that within the next year or so (™️) we’ll be able to get some great benefits by upgrading to a subset of the commitment changes outlined [here](https://delvingbitcoin.org/t/lightning-transactions-with-v3-and-ephemeral-anchors/418), specifically:

* V3 commitment transactions
* Single, shared key anchor
* Removing CSV-1 delay from outputs

To do this, we’ll need a way to update the commitment type for the existing channels in the network, which is what the remainder of this post will discuss.

#### Upgrading LN Channels


There are various components of lightning channels that we may want to update (as outlined in [#1117](https://github.com/lightning/bolts/pull/1117/files#diff-5152eb82a6ba0426cccaf2929b3731109ff96210379322e008f20974253ba179R527)):

* Parameter update: we would like to update the constraints exchanged in channel [open](https://github.com/lightning/bolts/blob/5f8fea8dc3c8c612167dd9645c4a21fe9de2f147/02-peer-protocol.md#the-open_channel-message)/[accept](https://github.com/lightning/bolts/blob/5f8fea8dc3c8c612167dd9645c4a21fe9de2f147/02-peer-protocol.md#the-accept_channel-message).
* Commitment update: we would like to change the format of our commitment transaction, to upgrade our channel_type.
* Funding output update: we would like spending our funding output, to change our channel capacity or output type.

We already have a few commitment type updates that we know we’ll want:

* Non-anchor/anchor -> Zero fee anchor channels
* Anchor channels -> Simple taproot channels
* Simple taproot channels -> PTLC channels

There are a few approaches that we could take to achieve these different update types, and some overlap between what can be achieved by each proposal - reductively summarized in this table (please DYOReading):

||Parameter Update|Commitment Update|Funding Output Update|Status|
| --- | --- | --- | --- | --- |
|Dynamic Commitments [#1117](https://github.com/lightning/bolts/pull/1117)|✓|✓|Change output type|Specified and recently updated|
|Splice to Upgrade (idea)|x| ✓|✓|Idea, dependent on [#1160](https://github.com/lightning/bolts/pull/1160) (currently in interop), not specified (afaik)|
|Upgrade on Re-Establish [#868](https://github.com/lightning/bolts/pull/868/files)|x|✓|x|Specified, no recent review/updates|

##### What about Taproot Channels?

Notable in the dynamic commitments proposal is the desire to update the network to simple taproot channels (STCs) to enjoy cost and privacy benefits, and eventually upgrade the network to PTLCs. This upgrade doesn’t cleanly fit into the above categorization because, as currently defined, an update to taproot channels requires updates to both our funding output and commitment. There is an in between state we could consider where we only update our commitment, but that seems like it’ll get ugly to implement.

The table that follows attempts to draw some comparisons of the various ways that we could go about upgrading our channel_type to STC. It also includes two “base” cases - V0 channels with no upgrade (ie, today’s channels) and “native” taproot channels (that don’t need any upgrading) to help contextualize these values.

| Upgrade type          | Upgrade Cost       | Coop Close Cost [*1*] | Channel Identity | New Features (eg PTLCs) | On Chain Privacy [*4*] |
|-----------------------|--------------------|------------------------|------------------|--------------------------|-----------------------|
| [base-case] <br> V0 No Upgrade | n/a: base case | Spend P2WSH <br> <br> Inputs (2-of-2 P2WSH): <br> 96 vbytes <br><br> Outputs (P2WPKH x2): <br> 62 vbytes <br><br> = 158 vbytes | n/a: no update | n/a: base case | funding_output identifiable as 2-of-2 on close |
| Dynamic Commitments   | Kickoff transaction <br> <br> Input (2-of-2 P2WSH): <br> 96 vbytes <br><br> Output (P2TR x3): <br> 129 vbytes <br><br> = 225 vbytes | Spend Kickoff transaction <br> <br> Inputs (1 P2TR): <br> 57.5 vbytes <br><br> Outputs (P2TR x2): <br> 86 vbytes <br><br> Claim Kickoff anchor [*3*]: <br> + 57.5 vbytes for input <br><br> = 201 vbytes | Maintained until “kickoff” broadcast, which is likely only on channel close so full channel lifetime | Suggestion in proposal to use a bit in channel_update to advertise. | funding_output identifiable as 2-of-2 on kickoff tx <br><br> 1-in/3-out kickoff fingerprintable |
| Splice to Upgrade     | Splice transaction <br> <br> Input (2-of-2 P2WSH) [*2*]: <br> 96 vbytes <br><br> Output (P2TR x1): <br> 43 vbytes <br><br> = 139 vbytes | Spend Splice Transaction <br> <br> Inputs (1 P2TR): <br> 57.5 vbytes <br><br> Outputs (P2TR x2): <br> 86 vbytes <br><br> = 143.5 vbytes | Maintained until splice confirmation + 12 blocks, then re-announced (new channel). <br><br> Blocked on taproot gossip. | Channel is re-announced, will be able to announce new features (with gossip v1.75). | funding_output identifiable as 2-of-2 on splice tx |
| Update on Re-Establish <br><br> *(or other update mechanism with no on-chain update) | n/a: no on-chain update | Spend P2WSH <br> <br> Inputs (2-of-2 P2WSH): <br> 96 vbytes <br><br> Outputs (P2WPKH x2): <br> 62 vbytes <br><br> = 158 vbytes | Maintained | No re-announcement in specification, would need gossip change to announce new features. | funding_output identifiable as 2-of-2 on close |
| [base case] <br> Native Taproot Channel | n/a: base case | Spend P2TR funding <br> <br> Inputs (1 P2TR): <br> 57.5 vbytes <br><br> Outputs (2 P2TR): <br> 86 vbytes <br><br> = 143.5 vbytes | n/a: no update | n/a: base case | P2TR 1-in-2-out is our anonymity set |



Takeaways that I arrive at:

* The privacy and cost benefits of upgrading legacy channels to taproot are dubious IMO, so the benefits are:
  * Forward progress of moving towards PTLCs (which still need specification)
  * Actually using taproot channels (more production experience, more good)
* We’ll likely get taproot gossip (1.75) before we get near specifying PTLCs, so there isn’t any particular rush to upgrade to STCs.
* We’re going to have to introduce a way to advertise new features (like PTLCs) for upgraded channels regardless of the approach we take.

#### Eating the Elephant

Some [discussion](https://github.com/lightning/bolts/pull/863#issuecomment-1965119165) on these various proposals has already taken place on the splicing PR, but I wanted to open this up to start a thread here so that we have a better place to discuss multiple proposals rather than spamming up one PR. I agree with @t-bast assessment [here](https://github.com/lightning/bolts/pull/863#issuecomment-2051246665) that it seems most reasonable to have two different upgrade paths: one for when we need to spend our funding output, and one where we just upgrade the channel without on-chain updates.

I think that it makes sense to start work on a parameter + commitment upgrade via dynamic commitments, independently of how we choose to go about upgrading to taproot channels. This gives us the ability to upgrade to `option_zero_fee_htlc_tx` anchor channels, and provides a commitment format upgrade mechanism that can be used to get us to V3 channels (once specified).

#### Appendix for Table

Assumptions and notes for comparison table:

[1] both parties have non-dust balances for cooperative close

[2] splice to upgrade will be “empty” (add no inputs), fixed to make this most comparable to kickoff tx

[3] assuming that the kickoff tx’s anchor isn’t spendable in the close transaction, but that we do have batching available so we only need to account for the extra input

[4] only considering on-chain heuristics when looking at privacy heuristics (not considering gossip identifying UTXOs as lightning channels, as that’s the same for everything)

\* note that an update without a funding output spend could also be achieved with an update to the dynamic commitments proposal.

-------------------------

t-bast | 2024-05-21 08:46:22 UTC | #2

Thanks for summarizing those, that looks good to me! I agree with your conclusion as well, and look forward to start working on an upgrade to v3 transactions :slight_smile:

-------------------------

williamsthe59th | 2024-05-23 04:49:35 UTC | #3

I think there is still an elephant in the room.

Upgrading channels to a new type e.g with support for a more network-level restrictive policy like v3 could provoke non-propagating transactions if deployed channels parameters (e.g each per-channel's max_accepted_htlcs) are too wide.

-------------------------

t-bast | 2024-05-23 08:45:23 UTC | #4

Why is that an elephant in the room? It's just an implementation detail and is not at all hard to handle? The upgrade to v3 will simply have additional constraints around the `max_accepted_htlcs` parameter.

-------------------------

ProofOfKeags | 2024-05-24 01:41:22 UTC | #5

Excellent analysis. I think that the uncontroversial common ground between the differing perspectives here is that the implementation complexity of anything that doesn't involve funding output conversion is very manageable. This is already reasonably well-specified in the Dynamic Commitments proposal, although I suspect there is room for improvement on the Commitment Transaction Re-Render process and as I finish the implementation for LND I'll likely add some details to that section.

A couple of things I'd like to note.

First, the Dynamic Commitments proposal, as specified today, permits channel upgrades without a funding output spend if the channel types share the same funding output script. If you believe it doesn't read that way, I encourage you to leave commentary on the proposal about what part of that is confusing so I can clarify it.

Second, there are reasons to upgrade to STCs from existing channel types, but lifetime fee cost savings, and privacy are probably not the reasons to do so, as you said. The main reason to do it would be if you wanted to preserve channel identity while also adding Taproot Assets to the channel state. This is admittedly a use case that is niche at the moment and I do not expect there to be a large appetite for this specific use case right now.

Finally, the question of whether we need STCs to get to PTLCs is definitely a discussion we should have but it comes down to the appetite for maintaining another hybrid channel type whose inputs are p2wsh and whose htlc outputs are p2tr *in addition to* the channel type where both inputs and outputs are p2tr. Given that a full implementation of PTLCs requires implementing the lion's share of STCs anyway do we really want to have Anchors, Anchors+PTLCs, STCs, *AND* STCs+PTLCs? I imagine the answer here is no, but I could be wrong and other implementations should weigh in here.

-------------------------

carla | 2024-05-24 15:10:07 UTC | #6

>  implementation complexity of anything that doesn’t involve funding output conversion is very manageable

:+1: 

> First, the Dynamic Commitments proposal, as specified today, permits channel upgrades without a funding output spend if the channel types share the same funding output script. If you believe it doesn’t read that way, I encourage you to leave commentary on the proposal about what part of that is confusing so I can clarify it.

Not at all, the spec is nice and clear! Just in the specific case of STC/that comparison table, it doesn't upgrade without funding output changes.

> Finally, the question of whether we need STCs to get to PTLCs is definitely a discussion we should have but it comes down to the appetite for maintaining another hybrid channel type

Def. Given that most implementations have splicing almost done (:tm:) and PTLCs are a ways out, I would imagine that they'd pursue splice-to-upgrade (with a fee-cognizant heuristic or manual upgrade to decide when to go to chain) after taproot gossip is out. Likewise interested to hear how everyone is going to approach this. 

Assuming we upgrade our funding output _somehow_ (via splicing or dyn commits), we can then just upgrade to PTLCs/V3 using dyn commitments to change our commitment format.

-------------------------

ProofOfKeags | 2024-05-24 20:33:44 UTC | #7

I guess the question I have is whether or not people want to see DynComms broken up into two proposals: the part with the funding output conversion, and the part without. I think we are amenable to breaking up the text into separate documents or per your suggestion maybe just different PRs into the same document if they are sharing a negotiation process.

If breaking it up means we can get better prioritization from Eclair/CLN/LDK for the ability to change the state-space constraints and the commitment rendering function, we are happy to do so. Being able to reuse DynComms for TRUC commitments seems like a very natural evolution to me.

CC @t-bast @rustyrussell @MattCorallo

-------------------------

ajtowns | 2024-05-27 00:44:35 UTC | #8

[quote="ProofOfKeags, post:5, topic:881"]
Finally, the question of whether we need STCs to get to PTLCs is definitely a discussion we should have but it comes down to the appetite for maintaining another hybrid channel type
[/quote]

Isn't the straightforward answer to imagine three upgrade options:

 1. p2wsh + HTLC -> taproot + PTLC (bump funding tx)
 2. taproot + HTLC -> taproot + PTLC
 3. p2wsh + HTLC -> p2wsh + PTLC

Everyone might support (1), while (2) might only be something only LND supports if PTLCs get standardised before taproot channels are more widely implemented, and (3) might only get implemented much later, if it turns out there's huge demand for PTLCs and fees are high that people would prefer to fund dev effort to avoid bumping funding txs?

Could represent that with just three feature bits: does your node support taproot+HTLC channels, taproot+PTLC channels, and/or p2wsh+PTLC channels?

-------------------------

t-bast | 2024-05-27 08:15:40 UTC | #9

> I guess the question I have is whether or not people want to see DynComms broken up into two proposals: the part with the funding output conversion, and the part without.

Yes, I'd love to see that. As I stated in other posts, I believe we should use splicing for upgrades that require spending the current funding output (since splicing exists and does spend the funding output, it would be adding unnecessary complexity to have yet another protocol that achieves almost the same thing).

But for all other upgrades that don't spend the funding output, we definitely need a different protocol, and dynamic commitments seem to be a good fit :+1:

-------------------------

williamsthe59th | 2024-05-27 22:46:47 UTC | #10

[quote="t-bast, post:4, topic:881, full:true"]
Why is that an elephant in the room? It’s just an implementation detail and is not at all hard to handle? The upgrade to v3 will simply have additional constraints around the `max_accepted_htlcs` parameter.
[/quote]

I believe the conservative order of deployment to be: (a) deploy the dyn comm thing (b) restrict `max_accepted_htlcs` on all channels to fit V3 via the dyn comm thing (c) update to limited V3 package size all channels txn. And other specs shenanigans like max number of anchor outputs or scriptpubkey types changing commitment size.

-------------------------

