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

